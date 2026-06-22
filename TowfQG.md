温言睾幕挚


Linux容器命名空间与CGroup资源隔离实现

容器依赖两个内核特性：命名空间提供资源视图隔离，cgroup提供资源使用量限制。两者组合使一组进程认为自己独占操作系统资源，但事实上共享宿主内核。

先看clone系统调用创建命名空间的入口。每个命名空间类型在task_struct中有独立的指针：nsproxy保存所有命名空间的指针。clone调用时传入CLONE_NEWNS|CLONE_NEWUTS|CLONE_NEWIPC|CLONE_NEWPID|CLONE_NEWNET|CLONE_NEWUSER标志位，内核在create_new_namespaces中为每个类型创建新实例：

static struct nsproxy *create_new_namespaces(unsigned long flags,
    struct task_struct *tsk, struct user_namespace *user_ns,
    struct fs_struct *new_fs)
{
    struct nsproxy *new_nsp;
    new_nsp = kmem_cache_alloc(nsproxy_cachep, GFP_KERNEL);
    new_nsp->mnt_ns = copy_mnt_ns(flags, tsk->nsproxy->mnt_ns, user_ns, new_fs);
    new_nsp->uts_ns = copy_utsname(flags, tsk->nsproxy->uts_ns, user_ns);
    new_nsp->ipc_ns = copy_ipcs(flags, tsk->nsproxy->ipc_ns, user_ns);
    new_nsp->pid_ns_for_children = copy_pid_ns(flags, task_active_pid_ns(tsk), user_ns);
    new_nsp->net_ns = copy_net_ns(flags, tsk->nsproxy->net_ns, user_ns);
    new_nsp->cgroup_ns = copy_cgroup_ns(flags, tsk->nsproxy->cgroup_ns, user_ns);
    return new_nsp;
}

新命名空间的创建不继承任何父命名空间的资源状态。例如新PID命名空间中所有进程的PID从1开始。user命名空间是其他命名空间的入口，创建user_ns时必须保证调用进程有足够的capability。

PID命名空间是最直观的隔离层。struct pid_namespace包含level字段表示嵌套层数，进程在pid_namespace中的ID通过struct pid的numbers数组管理。struct pid的numbers[level]在对应层级的命名空间中有效。procfs在/proc/PID/status中显示NsPid，反映进程在各层命名空间中的PID映射。getpid()返回的是当前进程在所属PID命名空间中的ID，而非全局init_pid_ns的ID。内核通过task_active_pid_ns获取当前命名空间。

PID命名空间的层次结构决定了父命名空间能看见子命名空间中的进程映射，但子命名空间看不见父命名空间中的进程。每个PID命名空间有独立的last_pid计数器和pid_allocated计数器。pid分配通过alloc_pidmap获取空闲ID：

int alloc_pidmap(struct pid_namespace *pid_ns)
{
    int i, offset, max_scan, pid, last = pid_ns->last_pid;
    struct pidmap *map;
    pid = last + 1;
    // 位图查找空闲位
    offset = pid & BITS_PER_PAGE_MASK;
    map = &pid_ns->pidmap[pid/BITS_PER_PAGE];
    max_scan = (pid_max + BITS_PER_PAGE - 1)/BITS_PER_PAGE - offset;
    for (i = 0; i <= max_scan; i++) {
        // 逐位查找空闲位
        unsigned long bitmap = map->page;
        int next_bit = find_next_zero_bit(&bitmap, BITS_PER_PAGE, offset);
        if (next_bit < BITS_PER_PAGE) {
            __set_bit(next_bit, map->page);
            pid_ns->last_pid = next_bit;
            return next_bit;
        }
        offset = 0;
    }
    return -1;
}

alloc_pidmap使用位图管理PID分配。PID最大值由pid_max（/proc/sys/kernel/pid_max）控制，默认为32768。同一个PID命名空间内的PID不能重复。PID命名空间销毁时调用free_pidmap清理位图。init_pid_ns的PID空间大小随pid_max配置变化，子命名空间固定PID范围上限。

mount命名空间（CLONE_NEWNS）提供挂载点隔离。每个mount命名空间维护自己的mount树，struct mount的mnt_hash链入全局mount_hashtable，但namespace间的mount根不同。mount命名空间通过copy_tree复制父命名空间的mount树到子命名空间。复制采用propagation机制：如果父mount设置为MS_SHARED，子命名空间中的mount事件会传播到其他共享伙伴；MS_SLAVE接收但不同步；MS_PRIVATE完全隔离；MS_UNBINDABLE不允许绑定挂载。

新mount命名空间创建时通过clone(CLONE_NEWNS)创建后，子命名空间中的进程对mount表的修改不影响父命名空间。例如子命名空间中执行mount /dev/sdb1 /data，父命名空间看不到这个挂载点。pivot_root系统调用在mount命名空间中切换根文件系统，是容器启动时的核心操作。pivot_root将当前mount namespace的root移到put_old目录，new_root成为新的root：

int pivot_root(const char *new_root, const char *put_old);
// 内核实现：将当前namespace的root mount替换为new_root的mount

pivot_root不同于chroot之处在于它真正改变了进程的根目录挂载点，使chroot突破成为不可能。

network命名空间（CLONE_NEWNET）隔离网络设备、IP地址、路由表、防火墙规则等资源。new_net创建时只有loopback设备，其他网络设备需创建后移入。veth pair是容器网络的核心设备：ip link add veth0 type veth peer name veth1，通过SETNS将一端移入容器命名空间。内核中veth驱动通过veth_newlink创建一对struct net_device，两个设备共用struct veth_priv指向对端设备。数据包从一端发送时调用dev_queue_xmit，另一端从netif_rx接收：

static netdev_tx_t veth_xmit(struct sk_buff *skb, struct net_device *dev)
{
    struct veth_priv *priv = netdev_priv(dev);
    struct net_device *rcv = priv->peer;
    skb->dev = rcv;
    dev_forward_skb(rcv, skb);
    return NETDEV_TX_OK;
}

network命名空间中的路由表独立于宿主。通过ip netns exec <ns> ip route管理的路由只在该命名空间中生效。iptables规则链也完全隔离，每个netns有独立的xt_table数组。netfilter的钩子函数根据当前skb的net命名空间查找对应的规则。

user命名空间隔离UID和GID映射。核心数据结构是struct uid_gid_map，包含nreview层映射条目。每个映射在写入uid_map文件时通过proc_setgroups和proc_id_map验证。映射格式为"<内部UID> <外部UID> <长度>"。写映射文件需要持有CAP_SYS_ADMIN或父命名空间的CAP_SETUID。映射定义后内核在每次UID检查时通过map_id_down和map_id_up做转换。user命名空间中UID 0在父命名空间中映射为普通UID，但进程在该user命名空间中拥有所有capabilities：

static gfp_t get_user_ns(struct user_namespace *ns)
{
    // 用户命名空间创建时parent指向创建者的user_ns
    // owner指向映射到UID 0的用户
}

/proc/self/uid_map和/proc/self/gid_map展示当前进程的用户命名空间映射关系。如果容器中进程看到UID 0，而宿主看到UID 100000，说明容器内核通过uid_gid_map的extent2做了转换。每个user_ns的capped权限在cap_capable检查时递归向父命名空间查询。

cgroup v2通过统一层级管理资源。cgroup核心结构struct cgroup维护每个控制组的生命周期。cgroup v2简化的接口集中在/sys/fs/cgroup下。写入cgroup.procs可以将进程移入控制组。内核通过cgroup_attach_task将task的默认cgroup子系统的cgrp指针更新。每个cgroup文件子系统通过css_set关联task_struct。css_set维护task_struct的cgroup引用计数。cgroup v2强制消除多层级，所有控制器在统一根节点下挂载。

CPU控制器的cfs bandwidth和cpu.weight在cgroup v2中的实现。cpu.max文件接受"$quota $period"格式。内核在cgroup_cfs构造时调用tg_cfs_scheduler设置带宽参数。cpu.weight控制组间cpu时间分配比例默认为100，写入较大值获取更多CPU时间。cpuset控制器（cpuset.cpus）限制进程可使用的CPU集合。内核通过cpuset_cpus_allowed将task->cpus_allowed掩码与cgroup cpuset做交，限制调度器选择的CPU范围：

int cpuset_cpus_allowed_fallback(struct task_struct *tsk)
{
    // add: 从cpuset的cpus_allowed拷贝到tsk->cpus_allowed
    // 如果cpuset为空，回退到cpu_possible_mask
    cpumask_and(tsk->cpus_allowed, current->nsproxy->cgroup_ns->root->cpuset->cpus_allowed, cpu_online_mask);
}

memory控制器限制容器的内存使用上限。memory.max写入最大值后，page fault路径在try_charge中对charge counter做检查。__memcg_kmem_charge_memcg检查memory消耗是否超过上限。超限后调用try_charge返回-ENOMEM，物理页分配被阻止。mem_cgroup的计费点包括anon pages、page cache、kernel memory（slab、stack page）。mem_cgroup维护memory.current和memory.max，超限时触发oom killer。oom killer在memory controller中选择组内进程kill并给出memcg OOM通知。oom_group的设置决定kill所有进程还是单个进程。

io控制器（io.max）保护块设备带宽。io.max写入"$device rbps=$val wbps=$val"后，内核注册对应的throttle policy。bio提交到块层时通过blkcg的blkg_lookup找到关联的block cgroup。throtl_charge_bio扣除读写配额。throtl_pending_timer_fn在配额不足时将bio挂到throtl_service_queue等待：

struct throtl_grp *tg = blkg_to_tg(blkg);
// 读取bps不足则延迟下发，延迟时间 = (超量字节 / 速率) * 1000
wait_ms = (excess_bytes * 1000) / bps;
mod_timer(&tg->service_queue.pending_timer, jiffies + msecs_to_jiffies(wait_ms));

io.weight控制组间IO带宽比例。bfq调度器通过bfqg_weight和bfqg_async_entity_weight计算每个cgroup的bfq队列权重。cgroup的io权重直接映射到bfq内部budget system。写io.latency将延迟目标写入后，内核使用ioc_lat_stat和ioc_lat_cal计算实际延迟是否超标，超标时通过throttle减少同cgroup其他进程的带宽。

容器的典型启动流程使用pivot_root或chroot切换根文件系统。pivot_root的实现由sys_pivot_root调用，检查old_root的mount是否在当前namespace的root mount之下且new_root的mount是否在当前namespace中。条件满足后执行__mnt_make_shortterm将old root标记为过期。最终更新current->fs->root指向new_root的dentry和mount：

SYSCALL_DEFINE2(pivot_root, const char __user *, new_root, const char __user *, put_old)
{
    struct mount *new_mnt, *old_mnt;
    // 查找到new_root的mount实例
    new_mnt = path.mnt;
    // 查找到put_old的mount实例
    old_mnt = old_path.mnt;
    // 交换root和old mount
    mnt_set_mountpoint(new_mnt, ..., old_mnt);
    mnt_set_mountpoint(root_mnt, ..., new_mnt);
}

chroot不改变mount namespace，仅仅改变fs->root的dentry指针。chroot后的进程可以通过打开..路径突破根目录，因为内核不检查dentry是否在root以上。pivot_root通过真正切换mount的根节点来防止突破。

seccomp-bpf系统调用过滤是容器的安全加固层。seccomp通过prctl(PR_SET_SECCOMP)或seccomp(SECCOMP_SET_MODE_FILTER)安装BPF过滤程序。内核在syscall入口执行secure_computing检查当前系统调用是否被允许：

int __secure_computing(const struct seccomp_data *sd)
{
    struct seccomp_filter *filter;
    int action;
    rcu_read_lock();
    filter = rcu_dereference(current->seccomp.filter);
    action = seccomp_run_filters(filter, sd);
    rcu_read_unlock();
    switch (action) {
    case SECCOMP_RET_ALLOW: return 0;
    case SECCOMP_RET_KILL_PROCESS: do_exit(SIGSYS); break;
    case SECCOMP_RET_ERRNO: return -errno;
    }
}

Docker的默认seccomp配置禁止了约44个系统调用，包括未使用的命名空间相关调用和acct、uselib等过时调用。每个容器可以通过--security-opt seccomp=override加载自定义配置文件。

capability机制的bounding set限制容器root用户的真正权限。task_struct的cred字段包含cap_effective、cap_permitted、cap_inheritable和cap_bset。cap_capable遍历task的cred检查请求的capability是否在cap_effective中。容器运行时移除CAP_SYS_MODULE、CAP_SYS_RAWIO、CAP_NET_ADMIN等高危capability。cap_bset在exec时保持不变，确保新启动的进程无法获得比父进程更多的权限。

rootless容器依赖user namespace映射，进程在命名空间内显示为root但实际UID是宿主上的普通用户。关键限制包括：内核接口device cgroup不被支持（因为需要CAP_SYS_ADMIN写入device白名单）、mount类型受限（不能mount块设备）、network namespace创建受限（通常用slirp4netns桥接）。Docker的rootless模式通过newuidmap和newgidmap映射UID范围。映射文件/etc/subuid定义每个普通用户可使用的子UID范围。

seccomp_notify机制（用户态seccomp handler）允许用户态程序处理seccomp违规。SECCOMP_IOCTL_NOTIF_RECV从seccomp notifier fd接收通知，用户态进程可返回允许或拒绝。这使得容器引擎可以在用户态做出seccomp决策。notifier的task通过seccomp_notify结构获取pid和fd指向原始进程的信息。

笛馗莆百蹈兑郧院潦使匙自父弦玖

aip.cggkm.cn/406648.Doc
aio.cggkm.cn/248204.Doc
aio.cggkm.cn/682484.Doc
aio.cggkm.cn/626426.Doc
aio.cggkm.cn/826628.Doc
aio.cggkm.cn/020622.Doc
aio.cggkm.cn/864228.Doc
aio.cggkm.cn/884842.Doc
aio.cggkm.cn/484684.Doc
aio.cggkm.cn/886860.Doc
aio.cggkm.cn/420422.Doc
aii.cggkm.cn/882820.Doc
aii.cggkm.cn/462088.Doc
aii.cggkm.cn/008280.Doc
aii.cggkm.cn/660246.Doc
aii.cggkm.cn/864400.Doc
aii.cggkm.cn/282002.Doc
aii.cggkm.cn/820080.Doc
aii.cggkm.cn/155113.Doc
aii.cggkm.cn/791595.Doc
aii.cggkm.cn/846222.Doc
aiu.cggkm.cn/404024.Doc
aiu.cggkm.cn/000646.Doc
aiu.cggkm.cn/511173.Doc
aiu.cggkm.cn/608264.Doc
aiu.cggkm.cn/935973.Doc
aiu.cggkm.cn/626484.Doc
aiu.cggkm.cn/220420.Doc
aiu.cggkm.cn/428884.Doc
aiu.cggkm.cn/957737.Doc
aiu.cggkm.cn/959151.Doc
aiy.cggkm.cn/460266.Doc
aiy.cggkm.cn/026808.Doc
aiy.cggkm.cn/844448.Doc
aiy.cggkm.cn/711733.Doc
aiy.cggkm.cn/828068.Doc
aiy.cggkm.cn/604400.Doc
aiy.cggkm.cn/486840.Doc
aiy.cggkm.cn/840648.Doc
aiy.cggkm.cn/860286.Doc
aiy.cggkm.cn/662002.Doc
ait.cggkm.cn/640220.Doc
ait.cggkm.cn/484888.Doc
ait.cggkm.cn/088226.Doc
ait.cggkm.cn/844820.Doc
ait.cggkm.cn/200802.Doc
ait.cggkm.cn/642240.Doc
ait.cggkm.cn/575519.Doc
ait.cggkm.cn/226206.Doc
ait.cggkm.cn/606468.Doc
ait.cggkm.cn/482026.Doc
air.cggkm.cn/688406.Doc
air.cggkm.cn/040226.Doc
air.cggkm.cn/222864.Doc
air.cggkm.cn/482642.Doc
air.cggkm.cn/248404.Doc
air.cggkm.cn/424422.Doc
air.cggkm.cn/040424.Doc
air.cggkm.cn/026048.Doc
air.cggkm.cn/264244.Doc
air.cggkm.cn/139975.Doc
aie.cggkm.cn/068448.Doc
aie.cggkm.cn/064440.Doc
aie.cggkm.cn/464084.Doc
aie.cggkm.cn/642028.Doc
aie.cggkm.cn/844864.Doc
aie.cggkm.cn/246268.Doc
aie.cggkm.cn/488888.Doc
aie.cggkm.cn/408840.Doc
aie.cggkm.cn/802624.Doc
aie.cggkm.cn/228606.Doc
aiw.cggkm.cn/646626.Doc
aiw.cggkm.cn/840260.Doc
aiw.cggkm.cn/048004.Doc
aiw.cggkm.cn/000882.Doc
aiw.cggkm.cn/395575.Doc
aiw.cggkm.cn/648440.Doc
aiw.cggkm.cn/406802.Doc
aiw.cggkm.cn/206400.Doc
aiw.cggkm.cn/591515.Doc
aiw.cggkm.cn/664468.Doc
aiq.cggkm.cn/842428.Doc
aiq.cggkm.cn/608486.Doc
aiq.cggkm.cn/133751.Doc
aiq.cggkm.cn/802646.Doc
aiq.cggkm.cn/882042.Doc
aiq.cggkm.cn/539151.Doc
aiq.cggkm.cn/282206.Doc
aiq.cggkm.cn/268486.Doc
aiq.cggkm.cn/575199.Doc
aiq.cggkm.cn/226222.Doc
aum.cggkm.cn/533791.Doc
aum.cggkm.cn/082662.Doc
aum.cggkm.cn/286244.Doc
aum.cggkm.cn/404222.Doc
aum.cggkm.cn/084862.Doc
aum.cggkm.cn/224022.Doc
aum.cggkm.cn/468022.Doc
aum.cggkm.cn/028020.Doc
aum.cggkm.cn/000026.Doc
aum.cggkm.cn/264208.Doc
aun.cggkm.cn/268466.Doc
aun.cggkm.cn/486880.Doc
aun.cggkm.cn/862468.Doc
aun.cggkm.cn/880482.Doc
aun.cggkm.cn/882488.Doc
aun.cggkm.cn/919379.Doc
aun.cggkm.cn/733335.Doc
aun.cggkm.cn/404484.Doc
aun.cggkm.cn/424020.Doc
aun.cggkm.cn/820640.Doc
aub.cggkm.cn/646602.Doc
aub.cggkm.cn/600848.Doc
aub.cggkm.cn/004682.Doc
aub.cggkm.cn/862662.Doc
aub.cggkm.cn/086408.Doc
aub.cggkm.cn/480086.Doc
aub.cggkm.cn/662404.Doc
aub.cggkm.cn/064804.Doc
aub.cggkm.cn/204462.Doc
aub.cggkm.cn/824046.Doc
auv.cggkm.cn/628844.Doc
auv.cggkm.cn/088442.Doc
auv.cggkm.cn/408044.Doc
auv.cggkm.cn/422226.Doc
auv.cggkm.cn/624246.Doc
auv.cggkm.cn/046080.Doc
auv.cggkm.cn/739171.Doc
auv.cggkm.cn/046268.Doc
auv.cggkm.cn/200662.Doc
auv.cggkm.cn/371771.Doc
auc.cggkm.cn/808460.Doc
auc.cggkm.cn/646006.Doc
auc.cggkm.cn/068228.Doc
auc.cggkm.cn/662026.Doc
auc.cggkm.cn/464866.Doc
auc.cggkm.cn/240624.Doc
auc.cggkm.cn/668846.Doc
auc.cggkm.cn/242880.Doc
auc.cggkm.cn/113517.Doc
auc.cggkm.cn/400468.Doc
aux.cggkm.cn/608024.Doc
aux.cggkm.cn/648848.Doc
aux.cggkm.cn/517931.Doc
aux.cggkm.cn/577795.Doc
aux.cggkm.cn/004626.Doc
aux.cggkm.cn/208880.Doc
aux.cggkm.cn/579751.Doc
aux.cggkm.cn/428866.Doc
aux.cggkm.cn/642462.Doc
aux.cggkm.cn/028204.Doc
auz.cggkm.cn/684086.Doc
auz.cggkm.cn/044029.Doc
auz.cggkm.cn/680222.Doc
auz.cggkm.cn/426640.Doc
auz.cggkm.cn/224424.Doc
auz.cggkm.cn/806242.Doc
auz.cggkm.cn/284866.Doc
auz.cggkm.cn/628006.Doc
auz.cggkm.cn/117137.Doc
auz.cggkm.cn/062608.Doc
aul.cggkm.cn/624444.Doc
aul.cggkm.cn/080822.Doc
aul.cggkm.cn/557919.Doc
aul.cggkm.cn/680044.Doc
aul.cggkm.cn/355951.Doc
aul.cggkm.cn/406280.Doc
aul.cggkm.cn/791959.Doc
aul.cggkm.cn/026042.Doc
aul.cggkm.cn/555395.Doc
aul.cggkm.cn/444840.Doc
auk.cggkm.cn/959159.Doc
auk.cggkm.cn/868400.Doc
auk.cggkm.cn/660682.Doc
auk.cggkm.cn/064004.Doc
auk.cggkm.cn/006668.Doc
auk.cggkm.cn/202884.Doc
auk.cggkm.cn/377155.Doc
auk.cggkm.cn/753106.Doc
auk.cggkm.cn/083788.Doc
auk.cggkm.cn/837291.Doc
auj.cggkm.cn/293050.Doc
auj.cggkm.cn/176582.Doc
auj.cggkm.cn/091437.Doc
auj.cggkm.cn/855861.Doc
auj.cggkm.cn/274868.Doc
auj.cggkm.cn/223884.Doc
auj.cggkm.cn/022237.Doc
auj.cggkm.cn/929258.Doc
auj.cggkm.cn/954648.Doc
auj.cggkm.cn/321505.Doc
auh.cggkm.cn/079282.Doc
auh.cggkm.cn/850962.Doc
auh.cggkm.cn/827275.Doc
auh.cggkm.cn/565872.Doc
auh.cggkm.cn/155324.Doc
auh.cggkm.cn/441357.Doc
auh.cggkm.cn/538532.Doc
auh.cggkm.cn/775454.Doc
auh.cggkm.cn/869175.Doc
auh.cggkm.cn/652220.Doc
aug.cggkm.cn/616027.Doc
aug.cggkm.cn/434371.Doc
aug.cggkm.cn/317462.Doc
aug.cggkm.cn/754665.Doc
aug.cggkm.cn/062159.Doc
aug.cggkm.cn/521321.Doc
aug.cggkm.cn/015183.Doc
aug.cggkm.cn/227069.Doc
aug.cggkm.cn/385990.Doc
aug.cggkm.cn/508492.Doc
auf.cggkm.cn/668888.Doc
auf.cggkm.cn/266972.Doc
auf.cggkm.cn/889569.Doc
auf.cggkm.cn/096120.Doc
auf.cggkm.cn/482691.Doc
auf.cggkm.cn/055860.Doc
auf.cggkm.cn/301983.Doc
auf.cggkm.cn/080269.Doc
auf.cggkm.cn/032204.Doc
auf.cggkm.cn/125845.Doc
aud.cggkm.cn/875880.Doc
aud.cggkm.cn/165638.Doc
aud.cggkm.cn/079044.Doc
aud.cggkm.cn/872386.Doc
aud.cggkm.cn/073904.Doc
aud.cggkm.cn/458129.Doc
aud.cggkm.cn/627439.Doc
aud.cggkm.cn/822391.Doc
aud.cggkm.cn/100969.Doc
aud.cggkm.cn/449358.Doc
aus.cggkm.cn/281588.Doc
aus.cggkm.cn/188994.Doc
aus.cggkm.cn/794214.Doc
aus.cggkm.cn/987313.Doc
aus.cggkm.cn/193575.Doc
aus.cggkm.cn/937179.Doc
aus.cggkm.cn/415356.Doc
aus.cggkm.cn/062135.Doc
aus.cggkm.cn/340692.Doc
aus.cggkm.cn/706556.Doc
aua.cggkm.cn/034583.Doc
aua.cggkm.cn/600667.Doc
aua.cggkm.cn/498139.Doc
aua.cggkm.cn/820196.Doc
aua.cggkm.cn/156381.Doc
aua.cggkm.cn/539321.Doc
aua.cggkm.cn/869892.Doc
aua.cggkm.cn/918737.Doc
aua.cggkm.cn/952325.Doc
aua.cggkm.cn/857551.Doc
aup.cggkm.cn/434786.Doc
aup.cggkm.cn/574726.Doc
aup.cggkm.cn/497893.Doc
aup.cggkm.cn/972008.Doc
aup.cggkm.cn/666169.Doc
aup.cggkm.cn/871494.Doc
aup.cggkm.cn/679149.Doc
aup.cggkm.cn/814518.Doc
aup.cggkm.cn/366701.Doc
aup.cggkm.cn/460731.Doc
auo.cggkm.cn/843104.Doc
auo.cggkm.cn/369046.Doc
auo.cggkm.cn/231971.Doc
auo.cggkm.cn/770570.Doc
auo.cggkm.cn/116963.Doc
auo.cggkm.cn/004081.Doc
auo.cggkm.cn/722498.Doc
auo.cggkm.cn/588936.Doc
auo.cggkm.cn/159389.Doc
auo.cggkm.cn/788280.Doc
aui.cggkm.cn/542393.Doc
aui.cggkm.cn/104708.Doc
aui.cggkm.cn/899681.Doc
aui.cggkm.cn/837125.Doc
aui.cggkm.cn/662297.Doc
aui.cggkm.cn/497050.Doc
aui.cggkm.cn/649449.Doc
aui.cggkm.cn/566831.Doc
aui.cggkm.cn/055756.Doc
aui.cggkm.cn/250416.Doc
auu.cggkm.cn/881453.Doc
auu.cggkm.cn/957496.Doc
auu.cggkm.cn/350779.Doc
auu.cggkm.cn/584519.Doc
auu.cggkm.cn/264250.Doc
auu.cggkm.cn/736046.Doc
auu.cggkm.cn/037223.Doc
auu.cggkm.cn/298502.Doc
auu.cggkm.cn/833095.Doc
auu.cggkm.cn/122524.Doc
auy.cggkm.cn/535431.Doc
auy.cggkm.cn/151378.Doc
auy.cggkm.cn/082179.Doc
auy.cggkm.cn/674005.Doc
auy.cggkm.cn/988147.Doc
auy.cggkm.cn/318153.Doc
auy.cggkm.cn/241278.Doc
auy.cggkm.cn/470312.Doc
auy.cggkm.cn/397015.Doc
auy.cggkm.cn/641553.Doc
aut.cggkm.cn/581231.Doc
aut.cggkm.cn/786038.Doc
aut.cggkm.cn/453411.Doc
aut.cggkm.cn/650103.Doc
aut.cggkm.cn/139377.Doc
aut.cggkm.cn/639686.Doc
aut.cggkm.cn/747209.Doc
aut.cggkm.cn/024755.Doc
aut.cggkm.cn/260420.Doc
aut.cggkm.cn/280666.Doc
aur.cggkm.cn/390150.Doc
aur.cggkm.cn/385303.Doc
aur.cggkm.cn/968650.Doc
aur.cggkm.cn/311709.Doc
aur.cggkm.cn/343637.Doc
aur.cggkm.cn/174189.Doc
aur.cggkm.cn/033082.Doc
aur.cggkm.cn/659263.Doc
aur.cggkm.cn/185803.Doc
aur.cggkm.cn/235479.Doc
aue.cggkm.cn/547126.Doc
aue.cggkm.cn/966573.Doc
aue.cggkm.cn/360506.Doc
aue.cggkm.cn/921994.Doc
aue.cggkm.cn/815696.Doc
aue.cggkm.cn/044680.Doc
aue.cggkm.cn/663446.Doc
aue.cggkm.cn/352745.Doc
aue.cggkm.cn/060644.Doc
aue.cggkm.cn/222840.Doc
auw.cggkm.cn/064806.Doc
auw.cggkm.cn/406604.Doc
auw.cggkm.cn/068204.Doc
auw.cggkm.cn/824664.Doc
auw.cggkm.cn/408406.Doc
auw.cggkm.cn/246826.Doc
auw.cggkm.cn/684620.Doc
auw.cggkm.cn/002622.Doc
auw.cggkm.cn/268004.Doc
auw.cggkm.cn/006200.Doc
auq.cggkm.cn/886446.Doc
auq.cggkm.cn/626208.Doc
auq.cggkm.cn/448402.Doc
auq.cggkm.cn/682666.Doc
auq.cggkm.cn/664068.Doc
auq.cggkm.cn/262040.Doc
auq.cggkm.cn/866064.Doc
auq.cggkm.cn/822080.Doc
auq.cggkm.cn/264668.Doc
auq.cggkm.cn/888266.Doc
aym.cggkm.cn/460468.Doc
aym.cggkm.cn/428204.Doc
aym.cggkm.cn/082020.Doc
aym.cggkm.cn/082488.Doc
aym.cggkm.cn/680422.Doc
aym.cggkm.cn/006648.Doc
aym.cggkm.cn/042246.Doc
aym.cggkm.cn/424266.Doc
aym.cggkm.cn/648846.Doc
aym.cggkm.cn/680886.Doc
ayn.cggkm.cn/842680.Doc
ayn.cggkm.cn/426624.Doc
ayn.cggkm.cn/882646.Doc
ayn.cggkm.cn/046486.Doc
ayn.cggkm.cn/428000.Doc
ayn.cggkm.cn/008444.Doc
ayn.cggkm.cn/462022.Doc
ayn.cggkm.cn/606228.Doc
ayn.cggkm.cn/828600.Doc
ayn.cggkm.cn/846846.Doc
ayb.cggkm.cn/864824.Doc
ayb.cggkm.cn/884648.Doc
ayb.cggkm.cn/068028.Doc
ayb.cggkm.cn/086862.Doc
ayb.cggkm.cn/646402.Doc
ayb.cggkm.cn/826000.Doc
ayb.cggkm.cn/260886.Doc
ayb.cggkm.cn/400024.Doc
ayb.cggkm.cn/260862.Doc
ayb.cggkm.cn/284802.Doc
ayv.cggkm.cn/668408.Doc
ayv.cggkm.cn/062088.Doc
ayv.cggkm.cn/462806.Doc
ayv.cggkm.cn/686402.Doc
ayv.cggkm.cn/002042.Doc
ayv.cggkm.cn/204220.Doc
ayv.cggkm.cn/802060.Doc
ayv.cggkm.cn/664606.Doc
ayv.cggkm.cn/266088.Doc
ayv.cggkm.cn/426248.Doc
ayc.cggkm.cn/062886.Doc
ayc.cggkm.cn/404404.Doc
ayc.cggkm.cn/686468.Doc
ayc.cggkm.cn/806604.Doc
