聪衬吕傲乖


Linux SELinux安全上下文TE规则与AVC缓存

SELinux的Type Enforcement（TE）规则引擎是访问控制决策的核心计算单元，AVC（Access Vector Cache）则是其唯一的热路径缓存层。本文从内核源码security/selinux/ss/avtab.h和security/selinux/avc.c出发，直接剖析TE规则匹配的向量化查找算法、AVC缓存的生命周期管理以及两者在SMP下的竞态博弈。

TE规则存储在avtab结构中，内核用哈希表组织所有allow/neverallow/dontaudit/auditallow规则。avtab_key定义了匹配维度：

```c
struct avtab_key {
        u16 source_type;        /* 源类型SID */
        u16 target_type;        /* 目标类型SID */
        u16 target_class;       /* 对象类别，如file,socket */
#define AVTAB_ALLOWED           0x0001
#define AVTAB_AUDITALLOW        0x0002
#define AVTAB_AUDITDENY         0x0004
#define AVTAB_XPERMS            0x0008
#define AVTAB_ENABLED           0x8000
        u16 specified;          /* 规则类型掩码 */
};
```

avtab_datum则承载决策负载，其中data是权限位图（每个bit对应一个SELinux权限常量，如FILE__READ），对于xperms扩展则指向分离的extended_perms结构。avtab的哈希槽数在构建时固定为AVTAB_HASH_BITS（16位，即65536个槽），每个槽以链表解决冲突。关键问题是哈希分布不均时，链表遍历退化为线性扫描——这在策略规模超过5000条规则时直接可见，`security_load_policy`花费的时间随规则数超线性增长。

AVC内核路径从`avc_has_perm`开始，该函数定义在security/selinux/avc.c。其核心决策路径执行以下伪代码：

```c
int avc_has_perm(u32 ssid, u32 tsid, u16 tclass, u32 requested,
                 struct avc_audit_data *auditdata)
{
        struct avc_node *node;
        struct av_decision avd;
        int rc;
        u32 denied;

        node = avc_lookup(ssid, tsid, tclass);
        if (unlikely(!node)) {
                /* 缓存未命中，下刷SID并计算决策 */
                rc = avc_decision(ssid, tsid, tclass, requested, &avd);
                if (rc)
                        return rc;
                node = avc_insert(ssid, tsid, tclass, &avd);
                /* avc_insert失败时仍可使用avd继续后续判断 */
        } else {
                memcpy(&avd, &node->ae.avd, sizeof(avd));
        }

        denied = requested & ~avd.allowed;
        if (unlikely(denied)) {
                rc = avc_audit(ssid, tsid, tclass, requested, &avd, rc, auditdata);
                return rc ? -EACCES : 0;
        }

        return 0;
}
```

这里最值得关注的是`avc_lookup`的RCU保护问题。AVC在2.6内核引入了RCU读端临界区以避免全局锁，但`avc_node`的引用计数管理仍然是竞态高发区。`avc_lookup`在RCU读锁下遍历哈希链，找到匹配节点后调用`avc_node_get`增加引用计数。但在SMP环境下，`avc_node_get`的`atomic_inc_not_zero`是必须的，因为另一个CPU可能在`avc_flush`中并发删除同一节点：

```c
static struct avc_node *avc_lookup(u32 ssid, u32 tsid, u16 tclass)
{
        struct avc_node *node;
        u32 key = avc_hash(ssid, tsid, tclass);

        rcu_read_lock();
        node = rcu_dereference(avc_cache.slots[key]);
        while (node) {
                if (node->ae.ssid == ssid &&
                    node->ae.tsid == tsid &&
                    node->ae.tclass == tclass) {
                        if (likely(atomic_inc_not_zero(&node->ae.refcount))) {
                                rcu_read_unlock();
                                return node;
                        }
                        /* 节点正在被回收，fallthrough继续遍历 */
                }
                node = rcu_dereference(node->next);
        }
        rcu_read_unlock();
        return NULL;
}
```

这里的竞态窗口极其狭窄：在`avc_lookup`匹配到节点后、调用`atomic_inc_not_zero`之前，如果恰好`avc_flush`（由`security_sid_to_context`或策略重载触发）将该节点移出哈希链并调用`call_rcu`回收，则`atomic_inc_not_zero`返回0，代码需要继续遍历下一个节点而非直接返回NULL。这个fallthrough逻辑在早期内核版本中缺失，导致UAF漏洞。从4.14开始，`atomic_inc_not_zero`检查成为了标准模式。

AVC缓存淘汰策略基于LRU近似，但不是标准的LRU链。内核实现了一个惰性淘汰机制：每个CPU在`avc_has_perm`路径中随机选择槽位进行检查，如果该槽位的节点数量超过`avc_cache_threshold`（默认512），则触发`avc_cache_reclaim`。`avc_cache_reclaim`扫描所有非空槽，对每个槽中的节点依据`avc_node->ae.avd.last_used`（jiffies时间戳）排序，驱逐最老的节点。实现细节在security/selinux/avc.c的`avc_cache_reclaim`函数中：

```c
static void avc_cache_reclaim(struct avc_cache *cache)
{
        struct avc_node *node, *prev;
        unsigned int idx;
        unsigned long oldest;

        for (idx = cache->slots_trim; idx < AVC_CACHE_SLOTS; idx++) {
restart:
                oldest = jiffies - avc_cache_threshold * HZ;
                prev = NULL;
                node = rcu_dereference_protected(cache->slots[idx],
                                                 lockdep_is_held(&avc_cache_lock));
                while (node) {
                        if (time_before(node->ae.avd.last_used, oldest) &&
                            atomic_read(&node->ae.refcount) == 0) {
                                /* 从链表移除并synchronize_rcu后释放 */
                                if (prev)
                                        rcu_assign_pointer(prev->next, node->next);
                                else
                                        rcu_assign_pointer(cache->slots[idx], node->next);
                                call_rcu(&node->rcu_head, avc_node_free_rcu);
                                goto restart;
                        }
                        prev = node;
                        node = node->next;
                }
        }
}
```

注意这里每个找到的过期节点移除后都goto restart，因为当前节点的next指针被修改后继续遍历同一链表可能跳过元素。这个实现没有使用标准内核list_head，而是手动next指针遍历，原因在于AVC节点需要RCU-safe的单个指针释放。

`avc_insert`处理新条目插入时的哈希冲突策略值得深究。当哈希槽中已有节点时，新节点插入到链表头部。这种LIFO策略对局部性有利：新插入的节点通常很快会被再次访问。但这也造成了一个问题——长时间运行的系统中，线性扫描可能因为陈旧的头部存在而不命中，迫使AVC频繁调用`avc_decision`进入SID服务器做完整策略查找。

`avc_decision`内部调用`security_compute_av`，该函数会遍历avtab中的所有匹配规则，累加允许权限并计算拒绝结果。这里有一个被低估的性能热点：对于多类型多类别的访问（如`setxattr`同时检查capability和SELinux），`avc_decision`可能被调用多次。每个调用都会在avtab哈希表中做一次完整的`avtab_search`，时间复杂度为O(n/m)（n为规则数，m为哈希桶数）。策略规模达到10万条规则时，`avtab_search`的单次调用延迟可达微秒级。

AVC的冷启动退化行为也是重要考量。在`avc_init`阶段，哈希表被初始化为空，所有访问都会触发`avc_decision`。`security_load_policy`首次加载时，`avc_ss_reset`清空整个缓存。从空缓存到稳定状态需要经历大量`avc_insert`操作，而每次插入前的`avc_lookup`都会因未命中而浪费哈希计算。对系统启动脚本来说，这可能导致大量`avc_audit`调用被记录到audit.log中。

AVC回调链（`avc_update_cache`）处理策略重载后的一致性问题。当`security_load_policy`触发策略切换时，`avc_ss_reset`遍历所有现有节点并逐一执行回调：

```c
void avc_ss_reset(u32 seqno)
{
        struct avc_callback *cb;
        struct avc_node *node;
        int i;

        write_lock_irq(&avc_cache_lock);

        /* 遍历所有槽位，为每个节点设置最新seqno标记 */
        for (i = 0; i < AVC_CACHE_SLOTS; i++) {
                node = avc_cache.slots[i];
                while (node) {
                        node->ae.avd.seqno = seqno;
                        node = node->next;
                }
        }

        /* 执行注册的回调函数 */
        list_for_each_entry(cb, &avc_callbacks, list) {
                if (cb->events & AVC_CALLBACK_RESET) {
                        cb->callback(cb->events, seqno, cb->args);
                }
        }

        write_unlock_irq(&avc_cache_lock);
}
```

这里的问题是：策略重载期间，`avc_has_perm`可能通过RCU读端仍然持有旧的节点引用。这种场景下，节点中存储的`avd.allowed`来自旧策略，而seqno已被更新为新值。`avc_has_perm`在获取节点后检查`node->ae.avd.seqno`与当前`avc_cache.latest_seqno`是否一致，如果不一致则视同缓存未命中，丢弃该节点并重新调用`avc_decision`。这个seqno失效机制保证了策略重载的原子可见性，但代价是重载后所有缓存条目第一次访问都会miss，形成"缓存雪崩"。

AVC的audit抑制机制通过`avc_audit`内部的`avc_suppress_audit`实现。当`denied`权限被拒绝且该规则声明了`auditdeny`时，内核调用`slow_avc_audit`生成审计日志。性能敏感路径会通过`avc_has_perm_noaudit`绕过审计，延迟到用户空间处理审计：

```c
int avc_has_perm_noaudit(u32 ssid, u32 tsid, u16 tclass, u32 requested,
                         unsigned flags, struct av_decision *avd)
{
        struct avc_node *node;
        int rc = 0;
        u32 denied;

        node = avc_lookup(ssid, tsid, tclass);
        if (unlikely(!node)) {
                rc = avc_decision(ssid, tsid, tclass, requested, avd);
                if (rc)
                        return rc;
                node = avc_insert(ssid, tsid, tclass, avd);
        } else {
                memcpy(avd, &node->ae.avd, sizeof(*avd));
                avc_node_put(node);
        }

        denied = requested & ~avd->allowed;
        if (denied)
                rc = -EACCES;

        return rc;
}
```

在这条路径上，调用者（例如`selinux_inode_permission`）需要自行决定何时调用`avc_audit`。常见错误是审计延迟导致了权限状态变化（如策略重载）导致审计时的上下文与决策时的上下文不一致。`selinux_file_permission`通过lru_list上的哨兵位标记来绕过这个问题，但这个机制在4.19之后被大幅简化。

文件路径security/selinux/avc.c中`avc_audit_pre_update`和`avc_audit_post_update`是审计回调函数的实现，它们与inode的安全上下文变更深度绑定。当`selinux_inode_setsecurity`触发安全上下文变更时，AVC回调链中的`inode_security_revalidate`会被调用，它会将该inode对应的SID对从AVC中标记为无效。这里存在一个竞争：inode的setxattr操作和对该inode的并发访问可能在`inode_lock`之外竞争AVC条目，导致过期的权限判断被提交到审计日志。

AVC的per-entry锁（`avc_node->ae.lock`）用于保护`avd.allowed`的`cache_hit`更新，但在现代内核中这个锁已经被完全移除——Jens Axboe在5.7的patch移除了这个自旋锁，因为`allowed`和`auditallow`在节点生命周期内是只读的，根本不需要保护。真正的写入只发生在节点构造时的`avc_init_node`和销毁路径，而这些路径已经在`avc_cache_lock`保护下执行。

关于AVC的`cache_hit`统计路径，`avc_stat`在`avc_has_perm`的快速路径上每次命中都会更新`cache_hit`计数器。在极端高并发场景下（例如数千个并发`open`调用），这个`percpu_counter`的更新本身就会成为伪共享（false sharing）来源。每个CPU读取`avc_stats`结构体时，如果`struct avc_stats`中的`cache_hit`和`cache_miss`位于同一cacheline，对`cache_hit`的增量更新会导致相邻CPU的`cache_miss`缓存行失效。SELinux已在`struct avc_stats`中通过`____cacheline_aligned_in_smp`对齐每个成员来缓解，但在`CONFIG_SMP`+`NR_CPUS`超过128的配置下，percpu变量的TLB压力仍然不容忽视。

TE规则本身的avtab_datum在策略加载后是只读的，因此`avtab_search`路径可以免锁执行。但`avtab_search`在哈希冲突链表中的遍历每步需要比较`avtab_key`的三元组（source_type, target_type, target_class），每次比较涉及三个u16的整数比较，编译器通常在`__avtab_search`中将其优化为单次48位比较。然而当规则包含xperms时，需要额外匹配`avtab_extended_perms`中的`driver`和`specified`字段，这会导致`avtab_search`路径的指令数翻倍。

策略编译器`checkpolicy`在构建avtab哈希表时使用的哈希函数是`avtab_hash`，它基于`key->source_type ^ (key->target_type << 2) ^ (key->target_class << 4)`。在`target_class`集中在低值范围（小于256）的策略中，高位bits完全未被利用，哈希分布依赖于`source_type`和`target_type`的高位变化。当系统中domain数量较少但file类型众多时，大量条目的`source_type`集中在少数domain上，导致哈希冲突率急剧上升。`avtabs`的`nslot`在构建时根据规则数量动态调整，但上限不超过`1 << AVTAB_HASH_BITS`。

AVC与Linux安全模块框架（LSM）的hook点注册在`security/selinux/hooks.c`中。每个SELinux hook入口函数（如`selinux_inode_permission`）调用`avc_has_perm`时，`ssid`来自当前进程的`cred->security`（`selinux_cred_sid`），`tsid`来自目标对象的SID（如inode->i_security->sid）。这对`current_cred()`的隐式依赖在高并发下需要注意：当进程通过`use_mm`或`kthread_use_mm`临时改变内存空间但未切换cred时，SELinux可能使用错误的`ssid`进行访问控制。虽然`kthread`通常被标记为`SECINITSID_KERNEL`，但在io_uring的`IORING_OP_OPENAT`路径中，`io-wq`线程会借用用户进程的cred，此时`avc_has_perm`的`ssid`来源变为`req->cred->security`而非`current_cred`，这是5.6之后io_uring SELinux支持中专门处理的变体。

AVC节点的内存分配失败处理是另一个需要关注的边界条件。`avc_insert`调用`kmem_cache_alloc`从`avc_node_cachep`分配节点。如果分配失败（尽管概率极低），`avc_decision`的结果仍然有效，仅缓存无法插入。此时`avc_has_perm`直接返回`avc_decision`的计算结果，性能劣化为每次都走慢路径。`avc_node_cachep`的`gfp_mask`为`GFP_ATOMIC`，意味着它不能在`avc_has_perm`的常用进程上下文之外分配——如果`avc_has_perm`在softirq或hardirq中被调用（通过`selinux_netlink_send`等路径），`GFP_ATOMIC`分配可能因为内存紧张而频繁失败，导致netlink消息处理完全绕过AVC缓存，每个包都走完整的策略引擎决策路径。

`avc_flush`是处理SID失效的入口点，当`security_sid_to_context`检测到SID对应的安全上下文被修改时，调用`avc_ss_reset`清空缓存。但`avc_flush`有一个粒度问题：它只能基于`ssid`、`tsid`或`tclass`选择性清除，无法清除特定`ssid+tsid+tclass`的单个条目。在`selinux_inode_invalidate_secctx`调用`avc_ss_reset`时，会清空整个AVC，导致系统瞬间失去所有缓存条目。对于有大量domain的SELinux系统（例如Android的`untrusted_app`域），这种'重载式'的清除会导致接下来几百毫秒内所有系统调用的访问控制延迟放大5-10倍。

`avc_audit`的格式化路径在`avc_dump_av`中，包含对`permissions_to_string`的遍历，它将权限位图转换为可读字符串。在审计日志密集产生的场景下，这个格式化函数的开销并不比权限检查本身小——每次调用需要遍历`perm_idx_map`中的近百个权限定义条目，拼接字符串到临时缓冲区。内核通过`slow_avc_audit`的后台处理机制延迟日志提交，但`avc_audit`本身的`audit_log_start`调用仍然会在审计队列满时触发阻塞等待。`audit_log_start`内部获取`auditd_conn->lock`的等待可能导致`avc_has_perm`调用者——可能持有各种inode锁或mmap_lock——陷入潜在的ABBA死锁风险。内核通过`audit_context`的`in_syscall`标记来检测和避免这种情况：一旦检测到当前审计可能是由持有mmap_lock的路径触发，`audit_log_start`会将日志请求排队到后台`kauditd`线程处理，但该排队路径本身在内存压力下有OOM风险。

虑槐蝗傻姑霉靠薪痉膛吃匚苏粤辗

m.blog.qtbzn.cn/Article/details/9399931.htm
m.blog.qtbzn.cn/Article/details/2066660.htm
m.blog.qtbzn.cn/Article/details/0808802.htm
m.blog.qtbzn.cn/Article/details/0022646.htm
m.blog.qtbzn.cn/Article/details/5937759.htm
m.blog.qtbzn.cn/Article/details/4600282.htm
m.blog.qtbzn.cn/Article/details/5319951.htm
m.blog.qtbzn.cn/Article/details/4888222.htm
m.blog.qtbzn.cn/Article/details/7933959.htm
m.blog.qtbzn.cn/Article/details/6604844.htm
m.blog.qtbzn.cn/Article/details/2822220.htm
m.blog.qtbzn.cn/Article/details/2446006.htm
m.blog.qtbzn.cn/Article/details/0066222.htm
m.blog.qtbzn.cn/Article/details/8486848.htm
m.blog.qtbzn.cn/Article/details/9515715.htm
m.blog.qtbzn.cn/Article/details/1359975.htm
m.blog.qtbzn.cn/Article/details/9939191.htm
m.blog.qtbzn.cn/Article/details/7317951.htm
m.blog.qtbzn.cn/Article/details/3979537.htm
m.blog.qtbzn.cn/Article/details/5719579.htm
m.blog.qtbzn.cn/Article/details/1931773.htm
m.blog.qtbzn.cn/Article/details/9313159.htm
m.blog.qtbzn.cn/Article/details/3997577.htm
m.blog.qtbzn.cn/Article/details/0408220.htm
m.blog.qtbzn.cn/Article/details/4862224.htm
m.blog.qtbzn.cn/Article/details/6640006.htm
m.blog.qtbzn.cn/Article/details/3131751.htm
m.blog.qtbzn.cn/Article/details/0424408.htm
m.blog.qtbzn.cn/Article/details/5177513.htm
m.blog.qtbzn.cn/Article/details/0006426.htm
m.blog.qtbzn.cn/Article/details/9555793.htm
m.blog.qtbzn.cn/Article/details/3139991.htm
m.blog.qtbzn.cn/Article/details/6046084.htm
m.blog.qtbzn.cn/Article/details/9173777.htm
m.blog.qtbzn.cn/Article/details/3555759.htm
m.blog.qtbzn.cn/Article/details/1153371.htm
m.blog.qtbzn.cn/Article/details/1711377.htm
m.blog.qtbzn.cn/Article/details/2286208.htm
m.blog.qtbzn.cn/Article/details/1977177.htm
m.blog.qtbzn.cn/Article/details/8204644.htm
m.blog.qtbzn.cn/Article/details/5791353.htm
m.blog.qtbzn.cn/Article/details/3731537.htm
m.blog.qtbzn.cn/Article/details/1333311.htm
m.blog.qtbzn.cn/Article/details/5973771.htm
m.blog.qtbzn.cn/Article/details/8008824.htm
m.blog.qtbzn.cn/Article/details/9117359.htm
m.blog.qtbzn.cn/Article/details/6604082.htm
m.blog.qtbzn.cn/Article/details/4848804.htm
m.blog.qtbzn.cn/Article/details/7357795.htm
m.blog.qtbzn.cn/Article/details/7155173.htm
m.blog.qtbzn.cn/Article/details/7159599.htm
m.blog.qtbzn.cn/Article/details/7397375.htm
m.blog.qtbzn.cn/Article/details/8888040.htm
m.blog.qtbzn.cn/Article/details/6262222.htm
m.blog.qtbzn.cn/Article/details/1559533.htm
m.blog.qtbzn.cn/Article/details/1537131.htm
m.blog.qtbzn.cn/Article/details/6460400.htm
m.blog.qtbzn.cn/Article/details/1553197.htm
m.blog.qtbzn.cn/Article/details/6824064.htm
m.blog.qtbzn.cn/Article/details/5937571.htm
m.blog.qtbzn.cn/Article/details/2086640.htm
m.blog.qtbzn.cn/Article/details/7551775.htm
m.blog.qtbzn.cn/Article/details/9537999.htm
m.blog.qtbzn.cn/Article/details/7153755.htm
m.blog.qtbzn.cn/Article/details/1975719.htm
m.blog.qtbzn.cn/Article/details/7577131.htm
m.blog.qtbzn.cn/Article/details/5137591.htm
m.blog.qtbzn.cn/Article/details/1397577.htm
m.blog.qtbzn.cn/Article/details/7759593.htm
m.blog.qtbzn.cn/Article/details/5719735.htm
m.blog.qtbzn.cn/Article/details/1757911.htm
m.blog.qtbzn.cn/Article/details/1999915.htm
m.blog.qtbzn.cn/Article/details/9157975.htm
m.blog.qtbzn.cn/Article/details/9397337.htm
m.blog.qtbzn.cn/Article/details/5751959.htm
m.blog.qtbzn.cn/Article/details/8800226.htm
m.blog.qtbzn.cn/Article/details/6060086.htm
m.blog.qtbzn.cn/Article/details/3953335.htm
m.blog.qtbzn.cn/Article/details/2886440.htm
m.blog.qtbzn.cn/Article/details/7513515.htm
m.blog.qtbzn.cn/Article/details/9717351.htm
m.blog.qtbzn.cn/Article/details/2884042.htm
m.blog.qtbzn.cn/Article/details/3917311.htm
m.blog.qtbzn.cn/Article/details/1371511.htm
m.blog.qtbzn.cn/Article/details/4286684.htm
m.blog.qtbzn.cn/Article/details/1719375.htm
m.blog.qtbzn.cn/Article/details/2026664.htm
m.blog.qtbzn.cn/Article/details/7157195.htm
m.blog.qtbzn.cn/Article/details/9179531.htm
m.blog.qtbzn.cn/Article/details/2488642.htm
m.blog.qtbzn.cn/Article/details/9337931.htm
m.blog.qtbzn.cn/Article/details/6288008.htm
m.blog.qtbzn.cn/Article/details/5375913.htm
m.blog.qtbzn.cn/Article/details/2602664.htm
m.blog.qtbzn.cn/Article/details/0206006.htm
m.blog.qtbzn.cn/Article/details/7999137.htm
m.blog.qtbzn.cn/Article/details/4866862.htm
m.blog.qtbzn.cn/Article/details/9197739.htm
m.blog.qtbzn.cn/Article/details/9979735.htm
m.blog.qtbzn.cn/Article/details/4468820.htm
m.blog.qtbzn.cn/Article/details/7155731.htm
m.blog.qtbzn.cn/Article/details/4620080.htm
m.blog.qtbzn.cn/Article/details/0422404.htm
m.blog.qtbzn.cn/Article/details/3799597.htm
m.blog.qtbzn.cn/Article/details/9973377.htm
m.blog.qtbzn.cn/Article/details/6808262.htm
m.blog.qtbzn.cn/Article/details/7575931.htm
m.blog.qtbzn.cn/Article/details/1113115.htm
m.blog.qtbzn.cn/Article/details/0466000.htm
m.blog.qtbzn.cn/Article/details/6408884.htm
m.blog.qtbzn.cn/Article/details/7777597.htm
m.blog.qtbzn.cn/Article/details/2622688.htm
m.blog.qtbzn.cn/Article/details/8846442.htm
m.blog.qtbzn.cn/Article/details/4260262.htm
m.blog.qtbzn.cn/Article/details/2420428.htm
m.blog.qtbzn.cn/Article/details/9397735.htm
m.blog.qtbzn.cn/Article/details/7799531.htm
m.blog.qtbzn.cn/Article/details/2866420.htm
m.blog.qtbzn.cn/Article/details/5531915.htm
m.blog.qtbzn.cn/Article/details/4642620.htm
m.blog.qtbzn.cn/Article/details/7315577.htm
m.blog.qtbzn.cn/Article/details/9397997.htm
m.blog.qtbzn.cn/Article/details/7311733.htm
m.blog.qtbzn.cn/Article/details/1553519.htm
m.blog.qtbzn.cn/Article/details/9997517.htm
m.blog.qtbzn.cn/Article/details/8822486.htm
m.blog.qtbzn.cn/Article/details/5919393.htm
m.blog.qtbzn.cn/Article/details/7339979.htm
m.blog.qtbzn.cn/Article/details/4468622.htm
m.blog.qtbzn.cn/Article/details/9993359.htm
m.blog.qtbzn.cn/Article/details/3977757.htm
m.blog.qtbzn.cn/Article/details/9159111.htm
m.blog.qtbzn.cn/Article/details/7797731.htm
m.blog.qtbzn.cn/Article/details/3535191.htm
m.blog.qtbzn.cn/Article/details/9195393.htm
m.blog.qtbzn.cn/Article/details/1715533.htm
m.blog.qtbzn.cn/Article/details/1359395.htm
m.blog.qtbzn.cn/Article/details/2468804.htm
m.blog.qtbzn.cn/Article/details/9771193.htm
m.blog.qtbzn.cn/Article/details/4248848.htm
m.blog.qtbzn.cn/Article/details/2828400.htm
m.blog.qtbzn.cn/Article/details/7793333.htm
m.blog.qtbzn.cn/Article/details/5777935.htm
m.blog.qtbzn.cn/Article/details/6820428.htm
m.blog.qtbzn.cn/Article/details/0408004.htm
m.blog.qtbzn.cn/Article/details/0824268.htm
m.blog.qtbzn.cn/Article/details/7335173.htm
m.blog.qtbzn.cn/Article/details/4060048.htm
m.blog.qtbzn.cn/Article/details/0682026.htm
m.blog.qtbzn.cn/Article/details/5533757.htm
m.blog.qtbzn.cn/Article/details/1119731.htm
m.blog.qtbzn.cn/Article/details/6464240.htm
m.blog.qtbzn.cn/Article/details/8086666.htm
m.blog.qtbzn.cn/Article/details/1911553.htm
m.blog.qtbzn.cn/Article/details/3153757.htm
m.blog.qtbzn.cn/Article/details/0882464.htm
m.blog.qtbzn.cn/Article/details/7137319.htm
m.blog.qtbzn.cn/Article/details/2006204.htm
m.blog.qtbzn.cn/Article/details/0400606.htm
m.blog.qtbzn.cn/Article/details/8042022.htm
m.blog.qtbzn.cn/Article/details/9977539.htm
m.blog.qtbzn.cn/Article/details/3977171.htm
m.blog.qtbzn.cn/Article/details/1593319.htm
m.blog.qtbzn.cn/Article/details/9553395.htm
m.blog.qtbzn.cn/Article/details/6026846.htm
m.blog.qtbzn.cn/Article/details/1139193.htm
m.blog.qtbzn.cn/Article/details/1775117.htm
m.blog.qtbzn.cn/Article/details/9735355.htm
m.blog.qtbzn.cn/Article/details/2220864.htm
m.blog.qtbzn.cn/Article/details/5973151.htm
m.blog.qtbzn.cn/Article/details/8402800.htm
m.blog.qtbzn.cn/Article/details/7799537.htm
m.blog.qtbzn.cn/Article/details/8064084.htm
m.blog.qtbzn.cn/Article/details/3957195.htm
m.blog.qtbzn.cn/Article/details/1539579.htm
m.blog.qtbzn.cn/Article/details/0220204.htm
m.blog.qtbzn.cn/Article/details/9317191.htm
m.blog.qtbzn.cn/Article/details/9779177.htm
m.blog.qtbzn.cn/Article/details/1155191.htm
m.blog.qtbzn.cn/Article/details/5551535.htm
m.blog.qtbzn.cn/Article/details/5371971.htm
m.blog.qtbzn.cn/Article/details/6486668.htm
m.blog.qtbzn.cn/Article/details/9733513.htm
m.blog.qtbzn.cn/Article/details/4000626.htm
m.blog.qtbzn.cn/Article/details/8622844.htm
m.blog.qtbzn.cn/Article/details/3913799.htm
m.blog.qtbzn.cn/Article/details/7915131.htm
m.blog.qtbzn.cn/Article/details/6004844.htm
m.blog.qtbzn.cn/Article/details/9915539.htm
m.blog.qtbzn.cn/Article/details/1575111.htm
m.blog.qtbzn.cn/Article/details/6020228.htm
m.blog.qtbzn.cn/Article/details/9797575.htm
m.blog.qtbzn.cn/Article/details/7939399.htm
m.blog.qtbzn.cn/Article/details/7191777.htm
m.blog.qtbzn.cn/Article/details/8682224.htm
m.blog.qtbzn.cn/Article/details/5559333.htm
m.blog.qtbzn.cn/Article/details/3755535.htm
m.blog.qtbzn.cn/Article/details/5931117.htm
m.blog.qtbzn.cn/Article/details/5117517.htm
m.blog.qtbzn.cn/Article/details/3353153.htm
m.blog.qtbzn.cn/Article/details/7757315.htm
m.blog.qtbzn.cn/Article/details/2600066.htm
m.blog.qtbzn.cn/Article/details/9911339.htm
m.blog.qtbzn.cn/Article/details/7199731.htm
m.blog.qtbzn.cn/Article/details/4200200.htm
m.blog.qtbzn.cn/Article/details/1335197.htm
m.blog.qtbzn.cn/Article/details/5713555.htm
m.blog.qtbzn.cn/Article/details/7777599.htm
m.blog.qtbzn.cn/Article/details/7931353.htm
m.blog.qtbzn.cn/Article/details/3313971.htm
m.blog.qtbzn.cn/Article/details/5315711.htm
m.blog.qtbzn.cn/Article/details/2000444.htm
m.blog.qtbzn.cn/Article/details/8648422.htm
m.blog.qtbzn.cn/Article/details/5195319.htm
m.blog.qtbzn.cn/Article/details/4828260.htm
m.blog.qtbzn.cn/Article/details/5951911.htm
m.blog.qtbzn.cn/Article/details/1933115.htm
m.blog.qtbzn.cn/Article/details/2042286.htm
m.blog.qtbzn.cn/Article/details/5319177.htm
m.blog.qtbzn.cn/Article/details/6020400.htm
m.blog.qtbzn.cn/Article/details/1739773.htm
m.blog.qtbzn.cn/Article/details/6400644.htm
m.blog.qtbzn.cn/Article/details/7193155.htm
m.blog.qtbzn.cn/Article/details/7979357.htm
m.blog.qtbzn.cn/Article/details/5737777.htm
m.blog.qtbzn.cn/Article/details/5513355.htm
m.blog.qtbzn.cn/Article/details/7115531.htm
m.blog.qtbzn.cn/Article/details/3973715.htm
m.blog.qtbzn.cn/Article/details/1539717.htm
m.blog.qtbzn.cn/Article/details/9975519.htm
m.blog.qtbzn.cn/Article/details/7151159.htm
m.blog.qtbzn.cn/Article/details/8246202.htm
m.blog.qtbzn.cn/Article/details/7593915.htm
m.blog.qtbzn.cn/Article/details/7531111.htm
m.blog.qtbzn.cn/Article/details/1551593.htm
m.blog.qtbzn.cn/Article/details/8660068.htm
m.blog.qtbzn.cn/Article/details/0024888.htm
m.blog.qtbzn.cn/Article/details/5119751.htm
m.blog.qtbzn.cn/Article/details/0426808.htm
m.blog.qtbzn.cn/Article/details/9355575.htm
m.blog.qtbzn.cn/Article/details/4866446.htm
m.blog.qtbzn.cn/Article/details/7555511.htm
m.blog.qtbzn.cn/Article/details/8846428.htm
m.blog.qtbzn.cn/Article/details/5917131.htm
m.blog.qtbzn.cn/Article/details/0082266.htm
m.blog.qtbzn.cn/Article/details/5771355.htm
m.blog.qtbzn.cn/Article/details/0244448.htm
m.blog.qtbzn.cn/Article/details/1317337.htm
m.blog.qtbzn.cn/Article/details/4408020.htm
m.blog.qtbzn.cn/Article/details/5731553.htm
m.blog.qtbzn.cn/Article/details/3915537.htm
m.blog.qtbzn.cn/Article/details/0842620.htm
m.blog.qtbzn.cn/Article/details/5359537.htm
m.blog.qtbzn.cn/Article/details/5335179.htm
m.blog.qtbzn.cn/Article/details/5715937.htm
m.blog.qtbzn.cn/Article/details/5779977.htm
m.blog.qtbzn.cn/Article/details/4660048.htm
m.blog.qtbzn.cn/Article/details/3597919.htm
m.blog.qtbzn.cn/Article/details/7317759.htm
m.blog.qtbzn.cn/Article/details/9193359.htm
m.blog.qtbzn.cn/Article/details/9573715.htm
m.blog.qtbzn.cn/Article/details/6482668.htm
m.blog.qtbzn.cn/Article/details/7731911.htm
m.blog.qtbzn.cn/Article/details/6282806.htm
m.blog.qtbzn.cn/Article/details/5513959.htm
m.blog.qtbzn.cn/Article/details/9375113.htm
m.blog.qtbzn.cn/Article/details/1153915.htm
m.blog.qtbzn.cn/Article/details/1375533.htm
m.blog.qtbzn.cn/Article/details/5573951.htm
m.blog.qtbzn.cn/Article/details/5377351.htm
m.blog.qtbzn.cn/Article/details/3151975.htm
m.blog.qtbzn.cn/Article/details/9199557.htm
m.blog.qtbzn.cn/Article/details/7337753.htm
m.blog.qtbzn.cn/Article/details/9357717.htm
m.blog.qtbzn.cn/Article/details/7553597.htm
m.blog.qtbzn.cn/Article/details/9797517.htm
m.blog.qtbzn.cn/Article/details/7391535.htm
m.blog.qtbzn.cn/Article/details/1373553.htm
m.blog.qtbzn.cn/Article/details/5931539.htm
m.blog.qtbzn.cn/Article/details/5951755.htm
m.blog.qtbzn.cn/Article/details/1731937.htm
m.blog.qtbzn.cn/Article/details/6482048.htm
m.blog.qtbzn.cn/Article/details/7797397.htm
m.blog.qtbzn.cn/Article/details/4286620.htm
m.blog.qtbzn.cn/Article/details/1171539.htm
m.blog.qtbzn.cn/Article/details/8206806.htm
m.blog.qtbzn.cn/Article/details/1599717.htm
m.blog.qtbzn.cn/Article/details/4628880.htm
m.blog.qtbzn.cn/Article/details/5953395.htm
m.blog.qtbzn.cn/Article/details/1939759.htm
m.blog.qtbzn.cn/Article/details/3135517.htm
m.blog.qtbzn.cn/Article/details/2642064.htm
m.blog.qtbzn.cn/Article/details/8260646.htm
m.blog.qtbzn.cn/Article/details/9937153.htm
m.blog.qtbzn.cn/Article/details/1571175.htm
m.blog.qtbzn.cn/Article/details/3917999.htm
m.blog.qtbzn.cn/Article/details/3511313.htm
m.blog.qtbzn.cn/Article/details/3993533.htm
m.blog.qtbzn.cn/Article/details/6424262.htm
m.blog.qtbzn.cn/Article/details/3513513.htm
m.blog.qtbzn.cn/Article/details/3935519.htm
m.blog.qtbzn.cn/Article/details/5353717.htm
m.blog.qtbzn.cn/Article/details/0860484.htm
m.blog.qtbzn.cn/Article/details/9173711.htm
m.blog.qtbzn.cn/Article/details/7913713.htm
m.blog.qtbzn.cn/Article/details/6006204.htm
m.blog.qtbzn.cn/Article/details/0464044.htm
m.blog.qtbzn.cn/Article/details/0044066.htm
m.blog.qtbzn.cn/Article/details/7959757.htm
m.blog.qtbzn.cn/Article/details/1391777.htm
m.blog.qtbzn.cn/Article/details/0628044.htm
m.blog.qtbzn.cn/Article/details/7597535.htm
m.blog.qtbzn.cn/Article/details/8206022.htm
m.blog.qtbzn.cn/Article/details/1159159.htm
m.blog.qtbzn.cn/Article/details/7715711.htm
m.blog.qtbzn.cn/Article/details/3795137.htm
m.blog.qtbzn.cn/Article/details/3533995.htm
m.blog.qtbzn.cn/Article/details/1313959.htm
m.blog.qtbzn.cn/Article/details/8622660.htm
m.blog.qtbzn.cn/Article/details/6002262.htm
m.blog.qtbzn.cn/Article/details/4264026.htm
m.blog.qtbzn.cn/Article/details/9551537.htm
m.blog.qtbzn.cn/Article/details/6624862.htm
m.blog.qtbzn.cn/Article/details/9915757.htm
m.blog.qtbzn.cn/Article/details/0266846.htm
m.blog.qtbzn.cn/Article/details/9937979.htm
m.blog.qtbzn.cn/Article/details/0026204.htm
m.blog.qtbzn.cn/Article/details/8826866.htm
m.blog.qtbzn.cn/Article/details/9953537.htm
m.blog.qtbzn.cn/Article/details/1115713.htm
m.blog.qtbzn.cn/Article/details/3913173.htm
m.blog.qtbzn.cn/Article/details/1973751.htm
m.blog.qtbzn.cn/Article/details/3559591.htm
m.blog.qtbzn.cn/Article/details/5717773.htm
m.blog.qtbzn.cn/Article/details/7731959.htm
m.blog.qtbzn.cn/Article/details/7751377.htm
m.blog.qtbzn.cn/Article/details/7119597.htm
m.blog.qtbzn.cn/Article/details/1151913.htm
m.blog.qtbzn.cn/Article/details/4080662.htm
m.blog.qtbzn.cn/Article/details/1573915.htm
m.blog.qtbzn.cn/Article/details/3177371.htm
m.blog.qtbzn.cn/Article/details/4840442.htm
m.blog.qtbzn.cn/Article/details/5173575.htm
m.blog.qtbzn.cn/Article/details/6620088.htm
m.blog.qtbzn.cn/Article/details/6880422.htm
m.blog.kxnxh.cn/Article/details/1137995.htm
m.blog.kxnxh.cn/Article/details/4668622.htm
m.blog.kxnxh.cn/Article/details/3975579.htm
m.blog.kxnxh.cn/Article/details/6848844.htm
m.blog.kxnxh.cn/Article/details/7519939.htm
m.blog.kxnxh.cn/Article/details/4282420.htm
m.blog.kxnxh.cn/Article/details/0624684.htm
m.blog.kxnxh.cn/Article/details/7199137.htm
m.blog.kxnxh.cn/Article/details/9195975.htm
m.blog.kxnxh.cn/Article/details/7173395.htm
m.blog.kxnxh.cn/Article/details/3155717.htm
m.blog.kxnxh.cn/Article/details/1937193.htm
m.blog.kxnxh.cn/Article/details/1395359.htm
m.blog.kxnxh.cn/Article/details/3513955.htm
m.blog.kxnxh.cn/Article/details/3739517.htm
m.blog.kxnxh.cn/Article/details/9531393.htm
m.blog.kxnxh.cn/Article/details/0686604.htm
m.blog.kxnxh.cn/Article/details/6086828.htm
m.blog.kxnxh.cn/Article/details/7591195.htm
m.blog.kxnxh.cn/Article/details/5793977.htm
m.blog.kxnxh.cn/Article/details/6460828.htm
m.blog.kxnxh.cn/Article/details/7797195.htm
m.blog.kxnxh.cn/Article/details/7117931.htm
m.blog.kxnxh.cn/Article/details/9917191.htm
m.blog.kxnxh.cn/Article/details/2248844.htm
m.blog.kxnxh.cn/Article/details/9771579.htm
m.blog.kxnxh.cn/Article/details/6284262.htm
m.blog.kxnxh.cn/Article/details/7315137.htm
m.blog.kxnxh.cn/Article/details/8244088.htm
m.blog.kxnxh.cn/Article/details/9997139.htm
m.blog.kxnxh.cn/Article/details/8820282.htm
m.blog.kxnxh.cn/Article/details/9993351.htm
m.blog.kxnxh.cn/Article/details/8604606.htm
m.blog.kxnxh.cn/Article/details/0808084.htm
m.blog.kxnxh.cn/Article/details/5551797.htm
m.blog.kxnxh.cn/Article/details/8024486.htm
m.blog.kxnxh.cn/Article/details/3133535.htm
m.blog.kxnxh.cn/Article/details/4860820.htm
m.blog.kxnxh.cn/Article/details/8204626.htm
m.blog.kxnxh.cn/Article/details/3757795.htm
m.blog.kxnxh.cn/Article/details/1937555.htm
m.blog.kxnxh.cn/Article/details/7131919.htm
m.blog.kxnxh.cn/Article/details/9735975.htm
m.blog.kxnxh.cn/Article/details/2286862.htm
m.blog.kxnxh.cn/Article/details/1173735.htm
m.blog.kxnxh.cn/Article/details/9713535.htm
m.blog.kxnxh.cn/Article/details/8464640.htm
m.blog.kxnxh.cn/Article/details/9577377.htm
m.blog.kxnxh.cn/Article/details/1575919.htm
m.blog.kxnxh.cn/Article/details/0622880.htm
