洞盘卮澈有


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

颐惶栈芬欣砂驮隙糙惶扰慈嗡乃屠

m.wog.mglwx.cn/15935.Doc
m.wog.mglwx.cn/73715.Doc
m.wog.mglwx.cn/44822.Doc
m.wog.mglwx.cn/44665.Doc
m.wog.mglwx.cn/75537.Doc
m.wof.mglwx.cn/46228.Doc
m.wof.mglwx.cn/84127.Doc
m.wof.mglwx.cn/86048.Doc
m.wof.mglwx.cn/10577.Doc
m.wof.mglwx.cn/19539.Doc
m.wof.mglwx.cn/60044.Doc
m.wof.mglwx.cn/02600.Doc
m.wof.mglwx.cn/53393.Doc
m.wof.mglwx.cn/67126.Doc
m.wof.mglwx.cn/31333.Doc
m.wof.mglwx.cn/15179.Doc
m.wof.mglwx.cn/84262.Doc
m.wof.mglwx.cn/06846.Doc
m.wof.mglwx.cn/17199.Doc
m.wof.mglwx.cn/08466.Doc
m.wof.mglwx.cn/55973.Doc
m.wof.mglwx.cn/80204.Doc
m.wof.mglwx.cn/01800.Doc
m.wof.mglwx.cn/02408.Doc
m.wof.mglwx.cn/73399.Doc
m.wod.mglwx.cn/64006.Doc
m.wod.mglwx.cn/46648.Doc
m.wod.mglwx.cn/04080.Doc
m.wod.mglwx.cn/93139.Doc
m.wod.mglwx.cn/54683.Doc
m.wod.mglwx.cn/15515.Doc
m.wod.mglwx.cn/39379.Doc
m.wod.mglwx.cn/24220.Doc
m.wod.mglwx.cn/59931.Doc
m.wod.mglwx.cn/62882.Doc
m.wod.mglwx.cn/40828.Doc
m.wod.mglwx.cn/15971.Doc
m.wod.mglwx.cn/28462.Doc
m.wod.mglwx.cn/22842.Doc
m.wod.mglwx.cn/62682.Doc
m.wod.mglwx.cn/99133.Doc
m.wod.mglwx.cn/51919.Doc
m.wod.mglwx.cn/31513.Doc
m.wod.mglwx.cn/71799.Doc
m.wod.mglwx.cn/13195.Doc
m.wos.mglwx.cn/75317.Doc
m.wos.mglwx.cn/73793.Doc
m.wos.mglwx.cn/40880.Doc
m.wos.mglwx.cn/22424.Doc
m.wos.mglwx.cn/42884.Doc
m.wos.mglwx.cn/04680.Doc
m.wos.mglwx.cn/84662.Doc
m.wos.mglwx.cn/35595.Doc
m.wos.mglwx.cn/31197.Doc
m.wos.mglwx.cn/17777.Doc
m.wos.mglwx.cn/93537.Doc
m.wos.mglwx.cn/75755.Doc
m.wos.mglwx.cn/24864.Doc
m.wos.mglwx.cn/19935.Doc
m.wos.mglwx.cn/11993.Doc
m.wos.mglwx.cn/62504.Doc
m.wos.mglwx.cn/57751.Doc
m.wos.mglwx.cn/91348.Doc
m.wos.mglwx.cn/59735.Doc
m.wos.mglwx.cn/97139.Doc
m.woa.mglwx.cn/75793.Doc
m.woa.mglwx.cn/52420.Doc
m.woa.mglwx.cn/79371.Doc
m.woa.mglwx.cn/64240.Doc
m.woa.mglwx.cn/08428.Doc
m.woa.mglwx.cn/93537.Doc
m.woa.mglwx.cn/97087.Doc
m.woa.mglwx.cn/99795.Doc
m.woa.mglwx.cn/79313.Doc
m.woa.mglwx.cn/56860.Doc
m.woa.mglwx.cn/81489.Doc
m.woa.mglwx.cn/39395.Doc
m.woa.mglwx.cn/65835.Doc
m.woa.mglwx.cn/98600.Doc
m.woa.mglwx.cn/15997.Doc
m.woa.mglwx.cn/89298.Doc
m.woa.mglwx.cn/66011.Doc
m.woa.mglwx.cn/93773.Doc
m.woa.mglwx.cn/65951.Doc
m.woa.mglwx.cn/03340.Doc
m.wop.mglwx.cn/31595.Doc
m.wop.mglwx.cn/71379.Doc
m.wop.mglwx.cn/00446.Doc
m.wop.mglwx.cn/67406.Doc
m.wop.mglwx.cn/20040.Doc
m.wop.mglwx.cn/95959.Doc
m.wop.mglwx.cn/42622.Doc
m.wop.mglwx.cn/87713.Doc
m.wop.mglwx.cn/79450.Doc
m.wop.mglwx.cn/24606.Doc
m.wop.mglwx.cn/64040.Doc
m.wop.mglwx.cn/04864.Doc
m.wop.mglwx.cn/71919.Doc
m.wop.mglwx.cn/32507.Doc
m.wop.mglwx.cn/59791.Doc
m.wop.mglwx.cn/73975.Doc
m.wop.mglwx.cn/82846.Doc
m.wop.mglwx.cn/66824.Doc
m.wop.mglwx.cn/40226.Doc
m.wop.mglwx.cn/24860.Doc
m.woo.mglwx.cn/42628.Doc
m.woo.mglwx.cn/64468.Doc
m.woo.mglwx.cn/57339.Doc
m.woo.mglwx.cn/47922.Doc
m.woo.mglwx.cn/48644.Doc
m.woo.mglwx.cn/84660.Doc
m.woo.mglwx.cn/42824.Doc
m.woo.mglwx.cn/28002.Doc
m.woo.mglwx.cn/46686.Doc
m.woo.mglwx.cn/69001.Doc
m.woo.mglwx.cn/20228.Doc
m.woo.mglwx.cn/85323.Doc
m.woo.mglwx.cn/48808.Doc
m.woo.mglwx.cn/59513.Doc
m.woo.mglwx.cn/02464.Doc
m.woo.mglwx.cn/02828.Doc
m.woo.mglwx.cn/19999.Doc
m.woo.mglwx.cn/96910.Doc
m.woo.mglwx.cn/73197.Doc
m.woo.mglwx.cn/71355.Doc
m.woi.mglwx.cn/88682.Doc
m.woi.mglwx.cn/11357.Doc
m.woi.mglwx.cn/97355.Doc
m.woi.mglwx.cn/59175.Doc
m.woi.mglwx.cn/82282.Doc
m.woi.mglwx.cn/59135.Doc
m.woi.mglwx.cn/00242.Doc
m.woi.mglwx.cn/53359.Doc
m.woi.mglwx.cn/73357.Doc
m.woi.mglwx.cn/44822.Doc
m.woi.mglwx.cn/96542.Doc
m.woi.mglwx.cn/39733.Doc
m.woi.mglwx.cn/71769.Doc
m.woi.mglwx.cn/24848.Doc
m.woi.mglwx.cn/46800.Doc
m.woi.mglwx.cn/17371.Doc
m.woi.mglwx.cn/86998.Doc
m.woi.mglwx.cn/22466.Doc
m.woi.mglwx.cn/86008.Doc
m.woi.mglwx.cn/52889.Doc
m.wou.mglwx.cn/62004.Doc
m.wou.mglwx.cn/08220.Doc
m.wou.mglwx.cn/51460.Doc
m.wou.mglwx.cn/71957.Doc
m.wou.mglwx.cn/82608.Doc
m.wou.mglwx.cn/88282.Doc
m.wou.mglwx.cn/31577.Doc
m.wou.mglwx.cn/38604.Doc
m.wou.mglwx.cn/24246.Doc
m.wou.mglwx.cn/91975.Doc
m.wou.mglwx.cn/60282.Doc
m.wou.mglwx.cn/99193.Doc
m.wou.mglwx.cn/75775.Doc
m.wou.mglwx.cn/06480.Doc
m.wou.mglwx.cn/06602.Doc
m.wou.mglwx.cn/33939.Doc
m.wou.mglwx.cn/40357.Doc
m.wou.mglwx.cn/67976.Doc
m.wou.mglwx.cn/35399.Doc
m.wou.mglwx.cn/29435.Doc
m.woy.mglwx.cn/13048.Doc
m.woy.mglwx.cn/99513.Doc
m.woy.mglwx.cn/88411.Doc
m.woy.mglwx.cn/11509.Doc
m.woy.mglwx.cn/02640.Doc
m.woy.mglwx.cn/77571.Doc
m.woy.mglwx.cn/26680.Doc
m.woy.mglwx.cn/71351.Doc
m.woy.mglwx.cn/46880.Doc
m.woy.mglwx.cn/70210.Doc
m.woy.mglwx.cn/31379.Doc
m.woy.mglwx.cn/66910.Doc
m.woy.mglwx.cn/64280.Doc
m.woy.mglwx.cn/53155.Doc
m.woy.mglwx.cn/62244.Doc
m.woy.mglwx.cn/44440.Doc
m.woy.mglwx.cn/55357.Doc
m.woy.mglwx.cn/69009.Doc
m.woy.mglwx.cn/64000.Doc
m.woy.mglwx.cn/51687.Doc
m.wot.mglwx.cn/08760.Doc
m.wot.mglwx.cn/77175.Doc
m.wot.mglwx.cn/73171.Doc
m.wot.mglwx.cn/08071.Doc
m.wot.mglwx.cn/02686.Doc
m.wot.mglwx.cn/59393.Doc
m.wot.mglwx.cn/36107.Doc
m.wot.mglwx.cn/24600.Doc
m.wot.mglwx.cn/64684.Doc
m.wot.mglwx.cn/04484.Doc
m.wot.mglwx.cn/00264.Doc
m.wot.mglwx.cn/58635.Doc
m.wot.mglwx.cn/86865.Doc
m.wot.mglwx.cn/84622.Doc
m.wot.mglwx.cn/40842.Doc
m.wot.mglwx.cn/32312.Doc
m.wot.mglwx.cn/17533.Doc
m.wot.mglwx.cn/91557.Doc
m.wot.mglwx.cn/00400.Doc
m.wot.mglwx.cn/33913.Doc
m.wor.mglwx.cn/48644.Doc
m.wor.mglwx.cn/42746.Doc
m.wor.mglwx.cn/20020.Doc
m.wor.mglwx.cn/60874.Doc
m.wor.mglwx.cn/62463.Doc
m.wor.mglwx.cn/59773.Doc
m.wor.mglwx.cn/86042.Doc
m.wor.mglwx.cn/69302.Doc
m.wor.mglwx.cn/73511.Doc
m.wor.mglwx.cn/39971.Doc
m.wor.mglwx.cn/88401.Doc
m.wor.mglwx.cn/51799.Doc
m.wor.mglwx.cn/48862.Doc
m.wor.mglwx.cn/94520.Doc
m.wor.mglwx.cn/75377.Doc
m.wor.mglwx.cn/46464.Doc
m.wor.mglwx.cn/60688.Doc
m.wor.mglwx.cn/55139.Doc
m.wor.mglwx.cn/27630.Doc
m.wor.mglwx.cn/17515.Doc
m.woe.mglwx.cn/03843.Doc
m.woe.mglwx.cn/35415.Doc
m.woe.mglwx.cn/60428.Doc
m.woe.mglwx.cn/99557.Doc
m.woe.mglwx.cn/11751.Doc
m.woe.mglwx.cn/55955.Doc
m.woe.mglwx.cn/15391.Doc
m.woe.mglwx.cn/02984.Doc
m.woe.mglwx.cn/62968.Doc
m.woe.mglwx.cn/50514.Doc
m.woe.mglwx.cn/73051.Doc
m.woe.mglwx.cn/73399.Doc
m.woe.mglwx.cn/88307.Doc
m.woe.mglwx.cn/46662.Doc
m.woe.mglwx.cn/05270.Doc
m.woe.mglwx.cn/24866.Doc
m.woe.mglwx.cn/82840.Doc
m.woe.mglwx.cn/94458.Doc
m.woe.mglwx.cn/86642.Doc
m.woe.mglwx.cn/22808.Doc
m.wow.mglwx.cn/79979.Doc
m.wow.mglwx.cn/41101.Doc
m.wow.mglwx.cn/89498.Doc
m.wow.mglwx.cn/71773.Doc
m.wow.mglwx.cn/25302.Doc
m.wow.mglwx.cn/80944.Doc
m.wow.mglwx.cn/93975.Doc
m.wow.mglwx.cn/24408.Doc
m.wow.mglwx.cn/48680.Doc
m.wow.mglwx.cn/13955.Doc
m.wow.mglwx.cn/55399.Doc
m.wow.mglwx.cn/67306.Doc
m.wow.mglwx.cn/55977.Doc
m.wow.mglwx.cn/84206.Doc
m.wow.mglwx.cn/05005.Doc
m.wow.mglwx.cn/37173.Doc
m.wow.mglwx.cn/20600.Doc
m.wow.mglwx.cn/96999.Doc
m.wow.mglwx.cn/79373.Doc
m.wow.mglwx.cn/86880.Doc
m.woq.mglwx.cn/42384.Doc
m.woq.mglwx.cn/75939.Doc
m.woq.mglwx.cn/87294.Doc
m.woq.mglwx.cn/42222.Doc
m.woq.mglwx.cn/17933.Doc
m.woq.mglwx.cn/26682.Doc
m.woq.mglwx.cn/84088.Doc
m.woq.mglwx.cn/55995.Doc
m.woq.mglwx.cn/55753.Doc
m.woq.mglwx.cn/68000.Doc
m.woq.mglwx.cn/53953.Doc
m.woq.mglwx.cn/59373.Doc
m.woq.mglwx.cn/88026.Doc
m.woq.mglwx.cn/42204.Doc
m.woq.mglwx.cn/02767.Doc
m.woq.mglwx.cn/22387.Doc
m.woq.mglwx.cn/39571.Doc
m.woq.mglwx.cn/20880.Doc
m.woq.mglwx.cn/82082.Doc
m.woq.mglwx.cn/33377.Doc
m.wim.mglwx.cn/46862.Doc
m.wim.mglwx.cn/73306.Doc
m.wim.mglwx.cn/06600.Doc
m.wim.mglwx.cn/13133.Doc
m.wim.mglwx.cn/91779.Doc
m.wim.mglwx.cn/46400.Doc
m.wim.mglwx.cn/63336.Doc
m.wim.mglwx.cn/46262.Doc
m.wim.mglwx.cn/88604.Doc
m.wim.mglwx.cn/31973.Doc
m.wim.mglwx.cn/31124.Doc
m.wim.mglwx.cn/52871.Doc
m.wim.mglwx.cn/48222.Doc
m.wim.mglwx.cn/28964.Doc
m.wim.mglwx.cn/57537.Doc
m.wim.mglwx.cn/19628.Doc
m.wim.mglwx.cn/68082.Doc
m.wim.mglwx.cn/39771.Doc
m.wim.mglwx.cn/40864.Doc
m.wim.mglwx.cn/06686.Doc
m.win.mglwx.cn/79957.Doc
m.win.mglwx.cn/33532.Doc
m.win.mglwx.cn/41715.Doc
m.win.mglwx.cn/71173.Doc
m.win.mglwx.cn/52354.Doc
m.win.mglwx.cn/66800.Doc
m.win.mglwx.cn/15333.Doc
m.win.mglwx.cn/82046.Doc
m.win.mglwx.cn/19315.Doc
m.win.mglwx.cn/73931.Doc
m.win.mglwx.cn/44848.Doc
m.win.mglwx.cn/09176.Doc
m.win.mglwx.cn/22444.Doc
m.win.mglwx.cn/84846.Doc
m.win.mglwx.cn/48464.Doc
m.win.mglwx.cn/15966.Doc
m.win.mglwx.cn/60804.Doc
m.win.mglwx.cn/21971.Doc
m.win.mglwx.cn/59997.Doc
m.win.mglwx.cn/82486.Doc
m.wib.mglwx.cn/11951.Doc
m.wib.mglwx.cn/31113.Doc
m.wib.mglwx.cn/84924.Doc
m.wib.mglwx.cn/59915.Doc
m.wib.mglwx.cn/79573.Doc
m.wib.mglwx.cn/15535.Doc
m.wib.mglwx.cn/20020.Doc
m.wib.mglwx.cn/66824.Doc
m.wib.mglwx.cn/22288.Doc
m.wib.mglwx.cn/26715.Doc
m.wib.mglwx.cn/00824.Doc
m.wib.mglwx.cn/15113.Doc
m.wib.mglwx.cn/31337.Doc
m.wib.mglwx.cn/62022.Doc
m.wib.mglwx.cn/94710.Doc
m.wib.mglwx.cn/39375.Doc
m.wib.mglwx.cn/74794.Doc
m.wib.mglwx.cn/51732.Doc
m.wib.mglwx.cn/57557.Doc
m.wib.mglwx.cn/64846.Doc
m.wiv.mglwx.cn/40826.Doc
m.wiv.mglwx.cn/93153.Doc
m.wiv.mglwx.cn/45074.Doc
m.wiv.mglwx.cn/91371.Doc
m.wiv.mglwx.cn/31755.Doc
m.wiv.mglwx.cn/73535.Doc
m.wiv.mglwx.cn/15179.Doc
m.wiv.mglwx.cn/99951.Doc
m.wiv.mglwx.cn/82808.Doc
m.wiv.mglwx.cn/66646.Doc
m.wiv.mglwx.cn/17571.Doc
m.wiv.mglwx.cn/68359.Doc
m.wiv.mglwx.cn/51719.Doc
m.wiv.mglwx.cn/37715.Doc
m.wiv.mglwx.cn/59977.Doc
m.wiv.mglwx.cn/71139.Doc
m.wiv.mglwx.cn/11379.Doc
m.wiv.mglwx.cn/67213.Doc
m.wiv.mglwx.cn/08268.Doc
m.wiv.mglwx.cn/52021.Doc
m.wic.mglwx.cn/99406.Doc
m.wic.mglwx.cn/00480.Doc
m.wic.mglwx.cn/95995.Doc
m.wic.mglwx.cn/42046.Doc
m.wic.mglwx.cn/57618.Doc
m.wic.mglwx.cn/20808.Doc
m.wic.mglwx.cn/99739.Doc
m.wic.mglwx.cn/14173.Doc
m.wic.mglwx.cn/99353.Doc
m.wic.mglwx.cn/20620.Doc
m.wic.mglwx.cn/71735.Doc
m.wic.mglwx.cn/57179.Doc
m.wic.mglwx.cn/91755.Doc
m.wic.mglwx.cn/00624.Doc
m.wic.mglwx.cn/33267.Doc
m.wic.mglwx.cn/17709.Doc
m.wic.mglwx.cn/99121.Doc
m.wic.mglwx.cn/15977.Doc
m.wic.mglwx.cn/20660.Doc
m.wic.mglwx.cn/99757.Doc
m.wix.mglwx.cn/48880.Doc
m.wix.mglwx.cn/15529.Doc
m.wix.mglwx.cn/86802.Doc
m.wix.mglwx.cn/99919.Doc
m.wix.mglwx.cn/06860.Doc
m.wix.mglwx.cn/71638.Doc
m.wix.mglwx.cn/42660.Doc
m.wix.mglwx.cn/33715.Doc
m.wix.mglwx.cn/24202.Doc
m.wix.mglwx.cn/02062.Doc
