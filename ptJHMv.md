涨优氐曳币


Linux sys_futex futex_wake与hashbucket锁定

futex(2) 系统调用是 Linux 实现高效用户态同步的核心机制。与所有经典同步原语不同的是，futex 在无竞争时完全在用户态通过原子操作完成，仅在需要等待或唤醒时进入内核。sys_futex 的入口是 do_futex：

```c
long do_futex(u32 __user *uaddr, int op, u32 val, ktime_t *timeout,
              u32 __user *uaddr2, u32 val2, u32 val3)
{
    int ret = -ENOSYS;

    switch (op) {
    case FUTEX_WAIT:
        ret = futex_wait(uaddr, flags, val, timeout, val3);
        break;
    case FUTEX_WAKE:
        ret = futex_wake(uaddr, flags, val, val3);
        break;
    case FUTEX_REQUEUE:
        ret = futex_requeue(uaddr, flags, uaddr2, val, val2, &val3, 0);
        break;
    case FUTEX_CMP_REQUEUE:
        ret = futex_requeue(uaddr, flags, uaddr2, val, val2, &val3, 1);
        break;
    case FUTEX_WAIT_BITSET:
        ret = futex_wait(uaddr, flags, val, timeout, val3);
        break;
    case FUTEX_WAKE_BITSET:
        ret = futex_wake(uaddr, flags, val, val3);
        break;
    case FUTEX_LOCK_PI:
        ret = futex_lock_pi(uaddr, flags, timeout, 0);
        break;
    case FUTEX_UNLOCK_PI:
        ret = futex_unlock_pi(uaddr, flags);
        break;
    ...
    }
    return ret;
}
```

do_futex 根据 op 分派到不同的处理函数。对于 FUTEX_WAKE，核心路径是 futex_wake：

```c
static int futex_wake(u32 __user *uaddr, unsigned int flags, int nr_wake, u32 bitset)
{
    struct futex_hash_bucket *hb;
    struct futex_q *this, *next;
    union futex_key key = FUTEX_KEY_INIT;
    int ret = 0;

    if (!bitset)
        return -EINVAL;

    ret = get_futex_key(uaddr, flags, &key, FUTEX_READ);
    if (unlikely(ret != 0))
        goto out;

    hb = hash_futex(&key);
    spin_lock(&hb->lock);

    plist_for_each_entry_safe(this, next, &hb->chain, list) {
        if (match_futex(&this->key, &key)) {
            if (this->pi_state || this->rt_waiter) {
                ret = -EINVAL;
                break;
            }

            if (!(this->bitset & bitset))
                continue;

            wake_futex(this);
            if (++ret >= nr_wake)
                break;
        }
    }

    spin_unlock(&hb->lock);
out:
    return ret;
}
```

get_futex_key 是第一个关键操作。它通过 get_user_pages_fast 锁定用户态的页，防止页面被换出导致物理地址变化，然后根据该页所在的位置（常规映射或匿名映射）构造一个独特的 futex_key：

```c
int get_futex_key(u32 __user *uaddr, unsigned int flags, union futex_key *key,
                  enum futex_access rw)
{
    unsigned long address = (unsigned long)uaddr;
    struct mm_struct *mm = current->mm;
    struct page *page, *tail;
    struct address_space *mapping;
    int err, ro = 0;

    if (unlikely((address % sizeof(u32)) != 0))
        return -EINVAL;

    address &= PAGE_MASK;

    err = get_user_pages_fast(address, 1, rw == FUTEX_WRITE, &page);
    if (err < 0)
        return err;
    ...
}
```

key 的构成决定了 futex 的关联方式。对于基于物理页框的共享 futex（MAP_SHARED），key 使用 mapping + index；对于私有映射，key 使用 mm + address。这使得 fork 之后的父子进程通过 COW 页面触发不同的 key，避免交叉唤醒。

hash_futex 将 key 哈希到 futex_hash_bucket：

```c
static struct futex_hash_bucket *hash_futex(union futex_key *key)
{
    u32 hash = jhash2((u32 *)key, offsetof(typeof(*key), both.offset) / 4,
                       key->both.offset);

    return &futex_queues[hash & (futex_hashsize - 1)];
}
```

futex_queues 是一个 hash bucket 数组，每个桶包含一个 plist（优先级排序链表）和一个 spinlock。plist 按优先级排序，确保优先级继承机制的 futex 操作中高优先级等待者被优先唤醒。

wake_futex 执行实际的唤醒操作：

```c
static void wake_futex(struct futex_q *q)
{
    struct task_struct *p = q->task;

    get_task_struct(p);
    plist_del(&q->list, &q->hb->chain);
    WRITE_ONCE(q->lock_ptr, &q->hb->lock);
    ...
    wake_up_state(p, TASK_NORMAL);
    put_task_struct(p);
}
```

wake_futex 从 hash bucket 链表中删除该 futex_q，然后调用 wake_up_state 将等待者的状态从 TASK_INTERRUPTIBLE 或 TASK_UNINTERRUPTIBLE 切换为 TASK_RUNNING，并将其加入运行队列。

hb->lock 是保护同一个 hash bucket 内所有 futex_q 的 spinlock。在 futex_wait 路径中，等待者在调用 futex_wait_queue_me 时会将 futex_q 插入到 hb->chain，并使用 set_current_state 设置 TASK_INTERRUPTIBLE，随后检查用户态的 futex 值是否发生变化。这种 double-check 机制是 futex 的核心：wake 和 wait 基于同一个 hb->lock 保证原子性，避免唤醒信号丢失。全局的 futex hash 表大小在启动时根据物理内存调整，默认散列到 256 个桶。
婪评谈兑老细难屏糖寥月驯艺塘亚

afs.fffbf.cn/200220.Doc
afa.fffbf.cn/624828.Doc
afa.fffbf.cn/486020.Doc
afa.fffbf.cn/602628.Doc
afa.fffbf.cn/400084.Doc
afa.fffbf.cn/428864.Doc
afa.fffbf.cn/777935.Doc
afa.fffbf.cn/682422.Doc
afa.fffbf.cn/028402.Doc
afa.fffbf.cn/448242.Doc
afa.fffbf.cn/488488.Doc
afp.fffbf.cn/448046.Doc
afp.fffbf.cn/002826.Doc
afp.fffbf.cn/022442.Doc
afp.fffbf.cn/719517.Doc
afp.fffbf.cn/226640.Doc
afp.fffbf.cn/646042.Doc
afp.fffbf.cn/444260.Doc
afp.fffbf.cn/600068.Doc
afp.fffbf.cn/179973.Doc
afp.fffbf.cn/264006.Doc
afo.fffbf.cn/600464.Doc
afo.fffbf.cn/000806.Doc
afo.fffbf.cn/668420.Doc
afo.fffbf.cn/446420.Doc
afo.fffbf.cn/688022.Doc
afo.fffbf.cn/842268.Doc
afo.fffbf.cn/446240.Doc
afo.fffbf.cn/242646.Doc
afo.fffbf.cn/406006.Doc
afo.fffbf.cn/882682.Doc
afi.fffbf.cn/868424.Doc
afi.fffbf.cn/844884.Doc
afi.fffbf.cn/808442.Doc
afi.fffbf.cn/424860.Doc
afi.fffbf.cn/202200.Doc
afi.fffbf.cn/884460.Doc
afi.fffbf.cn/880246.Doc
afi.fffbf.cn/403099.Doc
afi.fffbf.cn/280226.Doc
afi.fffbf.cn/977999.Doc
afu.fffbf.cn/242408.Doc
afu.fffbf.cn/840624.Doc
afu.fffbf.cn/440004.Doc
afu.fffbf.cn/240606.Doc
afu.fffbf.cn/486860.Doc
afu.fffbf.cn/084440.Doc
afu.fffbf.cn/820844.Doc
afu.fffbf.cn/064040.Doc
afu.fffbf.cn/844644.Doc
afu.fffbf.cn/642246.Doc
afy.fffbf.cn/860644.Doc
afy.fffbf.cn/462244.Doc
afy.fffbf.cn/626668.Doc
afy.fffbf.cn/682088.Doc
afy.fffbf.cn/608066.Doc
afy.fffbf.cn/042420.Doc
afy.fffbf.cn/957953.Doc
afy.fffbf.cn/888428.Doc
afy.fffbf.cn/220606.Doc
afy.fffbf.cn/662224.Doc
aft.fffbf.cn/868822.Doc
aft.fffbf.cn/711377.Doc
aft.fffbf.cn/810217.Doc
aft.fffbf.cn/566520.Doc
aft.fffbf.cn/284040.Doc
aft.fffbf.cn/868800.Doc
aft.fffbf.cn/848004.Doc
aft.fffbf.cn/084206.Doc
aft.fffbf.cn/226686.Doc
aft.fffbf.cn/208860.Doc
afr.fffbf.cn/020024.Doc
afr.fffbf.cn/066408.Doc
afr.fffbf.cn/260082.Doc
afr.fffbf.cn/862602.Doc
afr.fffbf.cn/442244.Doc
afr.fffbf.cn/262084.Doc
afr.fffbf.cn/284864.Doc
afr.fffbf.cn/888486.Doc
afr.fffbf.cn/684242.Doc
afr.fffbf.cn/006686.Doc
afe.fffbf.cn/288682.Doc
afe.fffbf.cn/800424.Doc
afe.fffbf.cn/552577.Doc
afe.fffbf.cn/600428.Doc
afe.fffbf.cn/484840.Doc
afe.fffbf.cn/006620.Doc
afe.fffbf.cn/660648.Doc
afe.fffbf.cn/412823.Doc
afe.fffbf.cn/244046.Doc
afe.fffbf.cn/004866.Doc
afw.fffbf.cn/864800.Doc
afw.fffbf.cn/864828.Doc
afw.fffbf.cn/288446.Doc
afw.fffbf.cn/662006.Doc
afw.fffbf.cn/484024.Doc
afw.fffbf.cn/064046.Doc
afw.fffbf.cn/822206.Doc
afw.fffbf.cn/441771.Doc
afw.fffbf.cn/215256.Doc
afw.fffbf.cn/280266.Doc
afq.fffbf.cn/260084.Doc
afq.fffbf.cn/422020.Doc
afq.fffbf.cn/222422.Doc
afq.fffbf.cn/422688.Doc
afq.fffbf.cn/998113.Doc
afq.fffbf.cn/480208.Doc
afq.fffbf.cn/648080.Doc
afq.fffbf.cn/248844.Doc
afq.fffbf.cn/800662.Doc
afq.fffbf.cn/402482.Doc
adm.fffbf.cn/288224.Doc
adm.fffbf.cn/606462.Doc
adm.fffbf.cn/460402.Doc
adm.fffbf.cn/246442.Doc
adm.fffbf.cn/442882.Doc
adm.fffbf.cn/200066.Doc
adm.fffbf.cn/440482.Doc
adm.fffbf.cn/800826.Doc
adm.fffbf.cn/820442.Doc
adm.fffbf.cn/024428.Doc
adn.fffbf.cn/400026.Doc
adn.fffbf.cn/024202.Doc
adn.fffbf.cn/446860.Doc
adn.fffbf.cn/662462.Doc
adn.fffbf.cn/026646.Doc
adn.fffbf.cn/642602.Doc
adn.fffbf.cn/404400.Doc
adn.fffbf.cn/226666.Doc
adn.fffbf.cn/220226.Doc
adn.fffbf.cn/488420.Doc
adb.fffbf.cn/400246.Doc
adb.fffbf.cn/266886.Doc
adb.fffbf.cn/060754.Doc
adb.fffbf.cn/406286.Doc
adb.fffbf.cn/624268.Doc
adb.fffbf.cn/440640.Doc
adb.fffbf.cn/642240.Doc
adb.fffbf.cn/242222.Doc
adb.fffbf.cn/117311.Doc
adb.fffbf.cn/115175.Doc
adv.fffbf.cn/446024.Doc
adv.fffbf.cn/828484.Doc
adv.fffbf.cn/288062.Doc
adv.fffbf.cn/048866.Doc
adv.fffbf.cn/220204.Doc
adv.fffbf.cn/048400.Doc
adv.fffbf.cn/248486.Doc
adv.fffbf.cn/080086.Doc
adv.fffbf.cn/424602.Doc
adv.fffbf.cn/846048.Doc
adc.fffbf.cn/220464.Doc
adc.fffbf.cn/880286.Doc
adc.fffbf.cn/448040.Doc
adc.fffbf.cn/462064.Doc
adc.fffbf.cn/442406.Doc
adc.fffbf.cn/820644.Doc
adc.fffbf.cn/044266.Doc
adc.fffbf.cn/868002.Doc
adc.fffbf.cn/204428.Doc
adc.fffbf.cn/606264.Doc
adx.fffbf.cn/268444.Doc
adx.fffbf.cn/046284.Doc
adx.fffbf.cn/282848.Doc
adx.fffbf.cn/608222.Doc
adx.fffbf.cn/668420.Doc
adx.fffbf.cn/040824.Doc
adx.fffbf.cn/808868.Doc
adx.fffbf.cn/646006.Doc
adx.fffbf.cn/319551.Doc
adx.fffbf.cn/420408.Doc
adz.fffbf.cn/628000.Doc
adz.fffbf.cn/484060.Doc
adz.fffbf.cn/244646.Doc
adz.fffbf.cn/624244.Doc
adz.fffbf.cn/514913.Doc
adz.fffbf.cn/800642.Doc
adz.fffbf.cn/444464.Doc
adz.fffbf.cn/660642.Doc
adz.fffbf.cn/062620.Doc
adz.fffbf.cn/008862.Doc
adl.fffbf.cn/444028.Doc
adl.fffbf.cn/800420.Doc
adl.fffbf.cn/044668.Doc
adl.fffbf.cn/713135.Doc
adl.fffbf.cn/482060.Doc
adl.fffbf.cn/888880.Doc
adl.fffbf.cn/806868.Doc
adl.fffbf.cn/682044.Doc
adl.fffbf.cn/408086.Doc
adl.fffbf.cn/264846.Doc
adk.fffbf.cn/046446.Doc
adk.fffbf.cn/208486.Doc
adk.fffbf.cn/026820.Doc
adk.fffbf.cn/420088.Doc
adk.fffbf.cn/600482.Doc
adk.fffbf.cn/462804.Doc
adk.fffbf.cn/080606.Doc
adk.fffbf.cn/288820.Doc
adk.fffbf.cn/404842.Doc
adk.fffbf.cn/066488.Doc
adj.fffbf.cn/624628.Doc
adj.fffbf.cn/086086.Doc
adj.fffbf.cn/939335.Doc
adj.fffbf.cn/406686.Doc
adj.fffbf.cn/422808.Doc
adj.fffbf.cn/680024.Doc
adj.fffbf.cn/571953.Doc
adj.fffbf.cn/628420.Doc
adj.fffbf.cn/084462.Doc
adj.fffbf.cn/082484.Doc
adh.fffbf.cn/682846.Doc
adh.fffbf.cn/682202.Doc
adh.fffbf.cn/828408.Doc
adh.fffbf.cn/806468.Doc
adh.fffbf.cn/462484.Doc
adh.fffbf.cn/888222.Doc
adh.fffbf.cn/328181.Doc
adh.fffbf.cn/575731.Doc
adh.fffbf.cn/828262.Doc
adh.fffbf.cn/426062.Doc
adg.fffbf.cn/199113.Doc
adg.fffbf.cn/264626.Doc
adg.fffbf.cn/824642.Doc
adg.fffbf.cn/064646.Doc
adg.fffbf.cn/848226.Doc
adg.fffbf.cn/626026.Doc
adg.fffbf.cn/624222.Doc
adg.fffbf.cn/719571.Doc
adg.fffbf.cn/688060.Doc
adg.fffbf.cn/406400.Doc
adf.fffbf.cn/200240.Doc
adf.fffbf.cn/000400.Doc
adf.fffbf.cn/880664.Doc
adf.fffbf.cn/404644.Doc
adf.fffbf.cn/244462.Doc
adf.fffbf.cn/808448.Doc
adf.fffbf.cn/008428.Doc
adf.fffbf.cn/844862.Doc
adf.fffbf.cn/826482.Doc
adf.fffbf.cn/640622.Doc
add.fffbf.cn/662866.Doc
add.fffbf.cn/046604.Doc
add.fffbf.cn/660406.Doc
add.fffbf.cn/822264.Doc
add.fffbf.cn/888688.Doc
add.fffbf.cn/808482.Doc
add.fffbf.cn/551139.Doc
add.fffbf.cn/882460.Doc
add.fffbf.cn/228408.Doc
add.fffbf.cn/026486.Doc
ads.fffbf.cn/288846.Doc
ads.fffbf.cn/666682.Doc
ads.fffbf.cn/682624.Doc
ads.fffbf.cn/737375.Doc
ads.fffbf.cn/533731.Doc
ads.fffbf.cn/428006.Doc
ads.fffbf.cn/000802.Doc
ads.fffbf.cn/004624.Doc
ads.fffbf.cn/864640.Doc
ads.fffbf.cn/646446.Doc
ada.fffbf.cn/375937.Doc
ada.fffbf.cn/848206.Doc
ada.fffbf.cn/484408.Doc
ada.fffbf.cn/484822.Doc
ada.fffbf.cn/222284.Doc
ada.fffbf.cn/608888.Doc
ada.fffbf.cn/824884.Doc
ada.fffbf.cn/622800.Doc
ada.fffbf.cn/420428.Doc
ada.fffbf.cn/402088.Doc
adp.fffbf.cn/044248.Doc
adp.fffbf.cn/628886.Doc
adp.fffbf.cn/402880.Doc
adp.fffbf.cn/624624.Doc
adp.fffbf.cn/882482.Doc
adp.fffbf.cn/337173.Doc
adp.fffbf.cn/846668.Doc
adp.fffbf.cn/533139.Doc
adp.fffbf.cn/226426.Doc
adp.fffbf.cn/240220.Doc
ado.fffbf.cn/337713.Doc
ado.fffbf.cn/715555.Doc
ado.fffbf.cn/202866.Doc
ado.fffbf.cn/888282.Doc
ado.fffbf.cn/684886.Doc
ado.fffbf.cn/262048.Doc
ado.fffbf.cn/319391.Doc
ado.fffbf.cn/682624.Doc
ado.fffbf.cn/860622.Doc
ado.fffbf.cn/608004.Doc
adi.fffbf.cn/717595.Doc
adi.fffbf.cn/662226.Doc
adi.fffbf.cn/195517.Doc
adi.fffbf.cn/357353.Doc
adi.fffbf.cn/626666.Doc
adi.fffbf.cn/824824.Doc
adi.fffbf.cn/066846.Doc
adi.fffbf.cn/628864.Doc
adi.fffbf.cn/448604.Doc
adi.fffbf.cn/682880.Doc
adu.fffbf.cn/224864.Doc
adu.fffbf.cn/800048.Doc
adu.fffbf.cn/068006.Doc
adu.fffbf.cn/264660.Doc
adu.fffbf.cn/711333.Doc
adu.fffbf.cn/426446.Doc
adu.fffbf.cn/246804.Doc
adu.fffbf.cn/244864.Doc
adu.fffbf.cn/866200.Doc
adu.fffbf.cn/264840.Doc
ady.fffbf.cn/802268.Doc
ady.fffbf.cn/531133.Doc
ady.fffbf.cn/591135.Doc
ady.fffbf.cn/280068.Doc
ady.fffbf.cn/224486.Doc
ady.fffbf.cn/640620.Doc
ady.fffbf.cn/884026.Doc
ady.fffbf.cn/822808.Doc
ady.fffbf.cn/399199.Doc
ady.fffbf.cn/917151.Doc
adt.fffbf.cn/208480.Doc
adt.fffbf.cn/042068.Doc
adt.fffbf.cn/840242.Doc
adt.fffbf.cn/668822.Doc
adt.fffbf.cn/808606.Doc
adt.fffbf.cn/802064.Doc
adt.fffbf.cn/442000.Doc
adt.fffbf.cn/373793.Doc
adt.fffbf.cn/668462.Doc
adt.fffbf.cn/840248.Doc
adr.jouwir.cn/808482.Doc
adr.jouwir.cn/513117.Doc
adr.jouwir.cn/406288.Doc
adr.jouwir.cn/244046.Doc
adr.jouwir.cn/626488.Doc
adr.jouwir.cn/719155.Doc
adr.jouwir.cn/862206.Doc
adr.jouwir.cn/866862.Doc
adr.jouwir.cn/086024.Doc
adr.jouwir.cn/420026.Doc
ade.jouwir.cn/377971.Doc
ade.jouwir.cn/915117.Doc
ade.jouwir.cn/806644.Doc
ade.jouwir.cn/579319.Doc
ade.jouwir.cn/408288.Doc
ade.jouwir.cn/260680.Doc
ade.jouwir.cn/446644.Doc
ade.jouwir.cn/573133.Doc
ade.jouwir.cn/202066.Doc
ade.jouwir.cn/264442.Doc
adw.jouwir.cn/208228.Doc
adw.jouwir.cn/222442.Doc
adw.jouwir.cn/220066.Doc
adw.jouwir.cn/482228.Doc
adw.jouwir.cn/248084.Doc
adw.jouwir.cn/026086.Doc
adw.jouwir.cn/060022.Doc
adw.jouwir.cn/640606.Doc
adw.jouwir.cn/666840.Doc
adw.jouwir.cn/684826.Doc
adq.jouwir.cn/086200.Doc
adq.jouwir.cn/026424.Doc
adq.jouwir.cn/462880.Doc
adq.jouwir.cn/008280.Doc
adq.jouwir.cn/668262.Doc
adq.jouwir.cn/028282.Doc
adq.jouwir.cn/446040.Doc
adq.jouwir.cn/246686.Doc
adq.jouwir.cn/886666.Doc
adq.jouwir.cn/224824.Doc
asm.jouwir.cn/286624.Doc
asm.jouwir.cn/642822.Doc
asm.jouwir.cn/026040.Doc
asm.jouwir.cn/446880.Doc
asm.jouwir.cn/604686.Doc
asm.jouwir.cn/882844.Doc
asm.jouwir.cn/846086.Doc
asm.jouwir.cn/006688.Doc
asm.jouwir.cn/442428.Doc
asm.jouwir.cn/240066.Doc
asn.jouwir.cn/244226.Doc
asn.jouwir.cn/860802.Doc
asn.jouwir.cn/064462.Doc
asn.jouwir.cn/266888.Doc
asn.jouwir.cn/084420.Doc
asn.jouwir.cn/840042.Doc
asn.jouwir.cn/282460.Doc
asn.jouwir.cn/208822.Doc
asn.jouwir.cn/044288.Doc
asn.jouwir.cn/844842.Doc
asb.jouwir.cn/222664.Doc
asb.jouwir.cn/086428.Doc
asb.jouwir.cn/248600.Doc
asb.jouwir.cn/448886.Doc
