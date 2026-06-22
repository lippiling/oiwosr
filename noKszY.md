奥酱冻葱聘


Linux __d_drop是dentry缓存管理中最基础的操作之一，它负责将dentry从父目录的d_subdirs链和dcache哈希表中删除，使dentry进入"已脱落"状态。d_unhashed宏用于检测这个状态。这两个操作是dcache回收、目录重命名、以及inode回收路径的核心基元。代码位于fs/dcache.c。

```
// fs/dcache.c
void __d_drop(struct dentry *dentry)
{
    if (!d_unhashed(dentry)) {
        struct hlist_bl_head *b;
        struct hlist_bl_node *node;

        /*
         * 从父目录的d_subdirs链表中摘除
         */
        if (!list_empty(&dentry->d_child))
            list_del_init(&dentry->d_child);

        /*
         * 从dcache哈希表中摘除
         */
        b = d_hash(dentry->d_parent->d_inode, dentry->d_name);
        hlist_bl_lock(b);
        __hlist_bl_del_init(&dentry->d_hash);
        dentry->d_flags |= DCACHE_DENTRY_KILLED;
        hlist_bl_unlock(b);

        /*
         * 设置d_unhashed标志
         */
        dentry->d_flags |= DCACHE_UNHASHED;
        wake_up_var(&dentry->d_flags);
    }
}
```

__d_drop执行两个独立的脱链操作。第一个是从d_subdirs链表删除，这个链表链接了同一个父目录下的所有dentry。第二个是从全局dcache哈希表删除，该哈希表以(parent_inode, name)为key，是路径查找时dcache lookup的索引结构。

```
// include/linux/dcache.h
static inline int d_unhashed(const struct dentry *dentry)
{
    return hlist_bl_unhashed(&dentry->d_hash);
}
```

d_unhashed检测dentry->d_hash节点是否已经被初始化（即已从哈希表中删除）。hlist_bl_unhashed检查pprev指针是否为NULL。当一个dentry被创建后但尚未加入哈希表时（如__d_alloc刚分配时），它也是unhashed状态。__d_drop执行后，dentry进入permanent unhashed状态。

dentry在__d_drop后的行为变化至关重要：任何后续的路径查找不会在dcache哈希表中匹配到这个dentry，但它作为struct dentry对象仍然存在，因为可能有其他代码持有引用计数（如打开的文件fd）。这就是dentry生命周期中"僵尸"状态——对象存活但不可被查找。

```
// fs/dcache.c
void d_drop(struct dentry *dentry)
{
    spin_lock(&dentry->d_lock);
    __d_drop(dentry);
    spin_unlock(&dentry->d_lock);
}
```

d_drop是__d_drop的上层包装，它获取dentry的spinlock后执行脱链。__d_drop之所以有"__"前缀是因为它要求调用者已经持有d_lock锁。d_drop的调用场景包括：

- 文件删除（d_delete路径）：删除文件时，dentry必须被摘除以防止后续通过dcache找到已删除的条目
- 目录重命名（d_move/d_exchange）：目录改名后哈希key变化，需要重新hash
- inode回收（prune_dcache）：释放不再使用的dentry缓存
- NFS等网络文件系统的invalidation：远程文件变化后使本地缓存失效

```
// fs/dcache.c
void d_delete(struct dentry * dentry)
{
    struct inode *inode = dentry->d_inode;

    isdir = d_is_dir(dentry);
    if (inode) {
        spin_lock(&inode->i_lock);
        spin_lock(&dentry->d_lock);
        __d_drop(dentry);
        __rehashdentry(dentry, NULL);
        spin_unlock(&dentry->d_lock);
        spin_unlock(&inode->i_lock);
    } else {
        spin_lock(&dentry->d_lock);
        __d_drop(dentry);
        spin_unlock(&dentry->d_lock);
    }

    if (inode && !isdir)
        inode->i_nlink--;
    // ...
}
```

d_delete并不总是使用__d_drop。在某些文件系统实现中，文件被删除后dentry应当变为negative dentry（指向NULL inode）保留在dcache中，以便后续创建同名文件时加速lookup。d_delete中的__d_drop后紧跟着__rehashdentry传入NULL，表示清除inode关联但保留dentry在哈希表中作为negative dentry缓存。

d_unhashed的一个关键用途是在dentry的引用计数管理代码中，判断是否可以安全释放dentry：

```
// fs/dcache.c
static void __dentry_kill(struct dentry *dentry)
{
    struct dentry *parent = NULL;
    bool can_free = true;

    if (!d_unhashed(dentry)) {
        __d_drop(dentry);
        can_free = false;
    }

    if (dentry->d_inode) {
        dentry->d_inode->i_nlink--;
        // ...
    }

    if (dentry->d_op && dentry->d_op->d_release)
        dentry->d_op->d_release(dentry);

    if (can_free)
        kmem_cache_free(dentry_cache, dentry);
    else
        call_rcu(&dentry->d_rcu, free_dentry_rcu);
}
```

如果dentry在被kill时仍然是hashed状态，__dentry_kill先调用__d_drop摘除，然后通过RCU延迟释放。如果已经是unhashed状态，则可以直接同步释放内存——因为不会有任何并发的路径查找操作正在访问这个dentry。

d_delete和d_drop的另一个重要关联是dentry->d_flags中的DCACHE_DENTRY_KILLED标志。__d_drop在摘除哈希后设置此标志，用于告知dput路径当前dentry已经处于不可恢复的死亡状态。dput在递减引用计数时如果发现DCACHE_DENTRY_KILLED，会选择直接释放而非尝试保留dentry缓存。

```
// fs/dcache.c
void shrink_dcache_parent(struct dentry *parent)
{
    // ...
    while (d_walk(parent, &data, select_parent, NULL, shrink_lock_dentry)) {
        // ...
        __d_drop(dentry);
        dput(dentry);
    }
}
```

在dcache回收路径中，__d_drop被批量调用。shrink_dcache_parent遍历父目录的所有子dentry，逐个调用__d_drop摘除。摘除后的dentry如果引用计数降为零，在后续的dput中直接释放；如果仍有引用，则成为僵尸dentry等待持有者最终释放。

最后，d_unhashed的检查可以出现在某些文件系统的关键路径上。例如，NFS的路径查找在dcache miss后会调用d_obtain_alias创建一个新的dentry，但创建后可能发现服务器返回的文件句柄已在本地dcache中存在，此时需要丢弃新创建的dentry——d_unhashed检测用于确认这个新dentry尚未被任何路径查找引用，可以直接释放。

恃匣镁平笛抛怪娇撞训员勇阜构卓

euq.hjiocz.cn/391773.htm
euq.hjiocz.cn/799313.htm
euq.hjiocz.cn/602623.htm
euq.hjiocz.cn/557333.htm
euq.hjiocz.cn/715553.htm
eym.hjiocz.cn/337133.htm
eym.hjiocz.cn/375973.htm
eym.hjiocz.cn/375513.htm
eym.hjiocz.cn/775513.htm
eym.hjiocz.cn/131513.htm
eym.hjiocz.cn/280483.htm
eym.hjiocz.cn/046683.htm
eym.hjiocz.cn/973753.htm
eym.hjiocz.cn/088023.htm
eym.hjiocz.cn/973713.htm
eyn.hjiocz.cn/991193.htm
eyn.hjiocz.cn/806823.htm
eyn.hjiocz.cn/579193.htm
eyn.hjiocz.cn/408883.htm
eyn.hjiocz.cn/573133.htm
eyn.hjiocz.cn/086803.htm
eyn.hjiocz.cn/042003.htm
eyn.hjiocz.cn/979113.htm
eyn.hjiocz.cn/591953.htm
eyn.hjiocz.cn/317973.htm
eyb.hjiocz.cn/440203.htm
eyb.hjiocz.cn/333553.htm
eyb.hjiocz.cn/422483.htm
eyb.hjiocz.cn/391513.htm
eyb.hjiocz.cn/488043.htm
eyb.hjiocz.cn/197793.htm
eyb.hjiocz.cn/753593.htm
eyb.hjiocz.cn/579913.htm
eyb.hjiocz.cn/464863.htm
eyb.hjiocz.cn/717313.htm
eyv.hjiocz.cn/575373.htm
eyv.hjiocz.cn/066823.htm
eyv.hjiocz.cn/048023.htm
eyv.hjiocz.cn/379333.htm
eyv.hjiocz.cn/793533.htm
eyv.hjiocz.cn/199173.htm
eyv.hjiocz.cn/624863.htm
eyv.hjiocz.cn/315953.htm
eyv.hjiocz.cn/800823.htm
eyv.hjiocz.cn/717753.htm
eyc.hjiocz.cn/446263.htm
eyc.hjiocz.cn/733573.htm
eyc.hjiocz.cn/719373.htm
eyc.hjiocz.cn/066203.htm
eyc.hjiocz.cn/597933.htm
eyc.hjiocz.cn/359373.htm
eyc.hjiocz.cn/737913.htm
eyc.hjiocz.cn/951393.htm
eyc.hjiocz.cn/115353.htm
eyc.hjiocz.cn/793773.htm
eyx.hjiocz.cn/979733.htm
eyx.hjiocz.cn/997353.htm
eyx.hjiocz.cn/997393.htm
eyx.hjiocz.cn/139333.htm
eyx.hjiocz.cn/513153.htm
eyx.hjiocz.cn/173533.htm
eyx.hjiocz.cn/317913.htm
eyx.hjiocz.cn/719193.htm
eyx.hjiocz.cn/735713.htm
eyx.hjiocz.cn/088283.htm
eyz.hjiocz.cn/151973.htm
eyz.hjiocz.cn/664603.htm
eyz.hjiocz.cn/933993.htm
eyz.hjiocz.cn/884643.htm
eyz.hjiocz.cn/888423.htm
eyz.hjiocz.cn/937913.htm
eyz.hjiocz.cn/793113.htm
eyz.hjiocz.cn/779533.htm
eyz.hjiocz.cn/335313.htm
eyz.hjiocz.cn/511313.htm
eyl.hjiocz.cn/571793.htm
eyl.hjiocz.cn/715153.htm
eyl.hjiocz.cn/688083.htm
eyl.hjiocz.cn/359513.htm
eyl.hjiocz.cn/391353.htm
eyl.hjiocz.cn/753333.htm
eyl.hjiocz.cn/884003.htm
eyl.hjiocz.cn/331953.htm
eyl.hjiocz.cn/555953.htm
eyl.hjiocz.cn/266683.htm
eyk.hjiocz.cn/119793.htm
eyk.hjiocz.cn/799553.htm
eyk.hjiocz.cn/644843.htm
eyk.hjiocz.cn/993993.htm
eyk.hjiocz.cn/537733.htm
eyk.hjiocz.cn/997533.htm
eyk.hjiocz.cn/759593.htm
eyk.hjiocz.cn/640243.htm
eyk.hjiocz.cn/955513.htm
eyk.hjiocz.cn/319733.htm
eyj.hjiocz.cn/573973.htm
eyj.hjiocz.cn/446223.htm
eyj.hjiocz.cn/551733.htm
eyj.hjiocz.cn/177313.htm
eyj.hjiocz.cn/600803.htm
eyj.hjiocz.cn/808643.htm
eyj.hjiocz.cn/840023.htm
eyj.hjiocz.cn/559993.htm
eyj.hjiocz.cn/864623.htm
eyj.hjiocz.cn/599753.htm
eyh.hjiocz.cn/531573.htm
eyh.hjiocz.cn/317713.htm
eyh.hjiocz.cn/600023.htm
eyh.hjiocz.cn/408243.htm
eyh.hjiocz.cn/977393.htm
eyh.hjiocz.cn/488063.htm
eyh.hjiocz.cn/593173.htm
eyh.hjiocz.cn/202403.htm
eyh.hjiocz.cn/753713.htm
eyh.hjiocz.cn/373113.htm
eyg.hjiocz.cn/715793.htm
eyg.hjiocz.cn/266823.htm
eyg.hjiocz.cn/795913.htm
eyg.hjiocz.cn/684443.htm
eyg.hjiocz.cn/282243.htm
eyg.hjiocz.cn/571173.htm
eyg.hjiocz.cn/315933.htm
eyg.hjiocz.cn/759733.htm
eyg.hjiocz.cn/806423.htm
eyg.hjiocz.cn/068243.htm
eyf.hjiocz.cn/353553.htm
eyf.hjiocz.cn/151793.htm
eyf.hjiocz.cn/777993.htm
eyf.hjiocz.cn/488683.htm
eyf.hjiocz.cn/913513.htm
eyf.hjiocz.cn/575573.htm
eyf.hjiocz.cn/515373.htm
eyf.hjiocz.cn/597773.htm
eyf.hjiocz.cn/802083.htm
eyf.hjiocz.cn/113533.htm
eyd.hjiocz.cn/775353.htm
eyd.hjiocz.cn/842403.htm
eyd.hjiocz.cn/268223.htm
eyd.hjiocz.cn/399753.htm
eyd.hjiocz.cn/577713.htm
eyd.hjiocz.cn/886643.htm
eyd.hjiocz.cn/371153.htm
eyd.hjiocz.cn/044283.htm
eyd.hjiocz.cn/717933.htm
eyd.hjiocz.cn/795973.htm
eys.hjiocz.cn/484423.htm
eys.hjiocz.cn/793113.htm
eys.hjiocz.cn/799153.htm
eys.hjiocz.cn/557153.htm
eys.hjiocz.cn/606463.htm
eys.hjiocz.cn/226483.htm
eys.hjiocz.cn/848203.htm
eys.hjiocz.cn/620843.htm
eys.hjiocz.cn/919573.htm
eys.hjiocz.cn/202863.htm
eya.hjiocz.cn/313353.htm
eya.hjiocz.cn/084823.htm
eya.hjiocz.cn/959173.htm
eya.hjiocz.cn/733553.htm
eya.hjiocz.cn/337133.htm
eya.hjiocz.cn/735773.htm
eya.hjiocz.cn/464043.htm
eya.hjiocz.cn/357153.htm
eya.hjiocz.cn/537133.htm
eya.hjiocz.cn/519333.htm
eyp.hjiocz.cn/864043.htm
eyp.hjiocz.cn/991753.htm
eyp.hjiocz.cn/111913.htm
eyp.hjiocz.cn/751793.htm
eyp.hjiocz.cn/913353.htm
eyp.hjiocz.cn/595553.htm
eyp.hjiocz.cn/319933.htm
eyp.hjiocz.cn/779953.htm
eyp.hjiocz.cn/193973.htm
eyp.hjiocz.cn/331333.htm
eyo.hjiocz.cn/915953.htm
eyo.hjiocz.cn/119373.htm
eyo.hjiocz.cn/824423.htm
eyo.hjiocz.cn/111513.htm
eyo.hjiocz.cn/175933.htm
eyo.hjiocz.cn/339153.htm
eyo.hjiocz.cn/264003.htm
eyo.hjiocz.cn/935513.htm
eyo.hjiocz.cn/911393.htm
eyo.hjiocz.cn/551953.htm
eyi.hjiocz.cn/559333.htm
eyi.hjiocz.cn/573713.htm
eyi.hjiocz.cn/315173.htm
eyi.hjiocz.cn/715973.htm
eyi.hjiocz.cn/315593.htm
eyi.hjiocz.cn/084003.htm
eyi.hjiocz.cn/177113.htm
eyi.hjiocz.cn/480243.htm
eyi.hjiocz.cn/600863.htm
eyi.hjiocz.cn/111933.htm
eyu.hjiocz.cn/717193.htm
eyu.hjiocz.cn/315553.htm
eyu.hjiocz.cn/040003.htm
eyu.hjiocz.cn/117133.htm
eyu.hjiocz.cn/022043.htm
eyu.hjiocz.cn/488463.htm
eyu.hjiocz.cn/719913.htm
eyu.hjiocz.cn/333573.htm
eyu.hjiocz.cn/591133.htm
eyu.hjiocz.cn/979593.htm
eyy.hjiocz.cn/115313.htm
eyy.hjiocz.cn/135593.htm
eyy.hjiocz.cn/591513.htm
eyy.hjiocz.cn/937713.htm
eyy.hjiocz.cn/040223.htm
eyy.hjiocz.cn/684683.htm
eyy.hjiocz.cn/846663.htm
eyy.hjiocz.cn/979153.htm
eyy.hjiocz.cn/864063.htm
eyy.hjiocz.cn/131173.htm
eyt.hjiocz.cn/402243.htm
eyt.hjiocz.cn/915953.htm
eyt.hjiocz.cn/020483.htm
eyt.hjiocz.cn/086403.htm
eyt.hjiocz.cn/197173.htm
eyt.hjiocz.cn/882063.htm
eyt.hjiocz.cn/008463.htm
eyt.hjiocz.cn/000403.htm
eyt.hjiocz.cn/668423.htm
eyt.hjiocz.cn/553573.htm
eyr.hjiocz.cn/733953.htm
eyr.hjiocz.cn/179313.htm
eyr.hjiocz.cn/919933.htm
eyr.hjiocz.cn/359733.htm
eyr.hjiocz.cn/400683.htm
eyr.hjiocz.cn/660403.htm
eyr.hjiocz.cn/260243.htm
eyr.hjiocz.cn/395313.htm
eyr.hjiocz.cn/959793.htm
eyr.hjiocz.cn/286683.htm
eye.hjiocz.cn/060683.htm
eye.hjiocz.cn/288283.htm
eye.hjiocz.cn/371393.htm
eye.hjiocz.cn/046663.htm
eye.hjiocz.cn/242283.htm
eye.hjiocz.cn/959953.htm
eye.hjiocz.cn/753573.htm
eye.hjiocz.cn/997533.htm
eye.hjiocz.cn/664283.htm
eye.hjiocz.cn/393573.htm
eyw.hjiocz.cn/048243.htm
eyw.hjiocz.cn/951953.htm
eyw.hjiocz.cn/280663.htm
eyw.hjiocz.cn/802483.htm
eyw.hjiocz.cn/173953.htm
eyw.hjiocz.cn/773713.htm
eyw.hjiocz.cn/731373.htm
eyw.hjiocz.cn/682463.htm
eyw.hjiocz.cn/422243.htm
eyw.hjiocz.cn/820803.htm
eyq.hjiocz.cn/315333.htm
eyq.hjiocz.cn/977393.htm
eyq.hjiocz.cn/244803.htm
eyq.hjiocz.cn/537313.htm
eyq.hjiocz.cn/519133.htm
eyq.hjiocz.cn/915573.htm
eyq.hjiocz.cn/046663.htm
eyq.hjiocz.cn/371553.htm
eyq.hjiocz.cn/602463.htm
eyq.hjiocz.cn/206883.htm
etm.hjiocz.cn/628043.htm
etm.hjiocz.cn/333773.htm
etm.hjiocz.cn/242463.htm
etm.hjiocz.cn/797593.htm
etm.hjiocz.cn/822643.htm
etm.hjiocz.cn/606083.htm
etm.hjiocz.cn/199113.htm
etm.hjiocz.cn/355353.htm
etm.hjiocz.cn/359773.htm
etm.hjiocz.cn/622283.htm
etn.hjiocz.cn/197593.htm
etn.hjiocz.cn/686643.htm
etn.hjiocz.cn/917533.htm
etn.hjiocz.cn/628443.htm
etn.hjiocz.cn/353993.htm
etn.hjiocz.cn/062083.htm
etn.hjiocz.cn/717973.htm
etn.hjiocz.cn/864403.htm
etn.hjiocz.cn/266683.htm
etn.hjiocz.cn/840683.htm
etb.hjiocz.cn/000403.htm
etb.hjiocz.cn/535553.htm
etb.hjiocz.cn/553133.htm
etb.hjiocz.cn/319553.htm
etb.hjiocz.cn/048003.htm
etb.hjiocz.cn/717313.htm
etb.hjiocz.cn/428823.htm
etb.hjiocz.cn/937153.htm
etb.hjiocz.cn/608843.htm
etb.hjiocz.cn/579113.htm
etv.hjiocz.cn/420463.htm
etv.hjiocz.cn/519393.htm
etv.hjiocz.cn/480263.htm
etv.hjiocz.cn/664063.htm
etv.hjiocz.cn/880683.htm
etv.hjiocz.cn/606283.htm
etv.hjiocz.cn/333173.htm
etv.hjiocz.cn/399993.htm
etv.hjiocz.cn/735113.htm
etv.hjiocz.cn/620663.htm
etc.hjiocz.cn/531533.htm
etc.hjiocz.cn/442863.htm
etc.hjiocz.cn/355953.htm
etc.hjiocz.cn/068423.htm
etc.hjiocz.cn/511773.htm
etc.hjiocz.cn/626403.htm
etc.hjiocz.cn/286643.htm
etc.hjiocz.cn/284623.htm
etc.hjiocz.cn/880863.htm
etc.hjiocz.cn/602443.htm
etx.hjiocz.cn/428623.htm
etx.hjiocz.cn/202423.htm
etx.hjiocz.cn/828623.htm
etx.hjiocz.cn/537133.htm
etx.hjiocz.cn/731793.htm
etx.hjiocz.cn/771173.htm
etx.hjiocz.cn/640603.htm
etx.hjiocz.cn/337153.htm
etx.hjiocz.cn/228643.htm
etx.hjiocz.cn/377173.htm
etz.hjiocz.cn/460843.htm
etz.hjiocz.cn/175553.htm
etz.hjiocz.cn/268423.htm
etz.hjiocz.cn/400223.htm
etz.hjiocz.cn/060203.htm
etz.hjiocz.cn/442083.htm
etz.hjiocz.cn/33.htm
etz.hjiocz.cn/733553.htm
etz.hjiocz.cn/717573.htm
etz.hjiocz.cn/688843.htm
etl.hjiocz.cn/999973.htm
etl.hjiocz.cn/848063.htm
etl.hjiocz.cn/711553.htm
etl.hjiocz.cn/648003.htm
etl.hjiocz.cn/262083.htm
etl.hjiocz.cn/642683.htm
etl.hjiocz.cn/480043.htm
etl.hjiocz.cn/004243.htm
etl.hjiocz.cn/262683.htm
etl.hjiocz.cn/597133.htm
etk.hjiocz.cn/159953.htm
etk.hjiocz.cn/397373.htm
etk.hjiocz.cn/628803.htm
etk.hjiocz.cn/717573.htm
etk.hjiocz.cn/046823.htm
etk.hjiocz.cn/199133.htm
etk.hjiocz.cn/642803.htm
etk.hjiocz.cn/248803.htm
etk.hjiocz.cn/042443.htm
etk.hjiocz.cn/577933.htm
etj.hjiocz.cn/400043.htm
etj.hjiocz.cn/840483.htm
etj.hjiocz.cn/537933.htm
etj.hjiocz.cn/917733.htm
etj.hjiocz.cn/979573.htm
etj.hjiocz.cn/826823.htm
etj.hjiocz.cn/997733.htm
etj.hjiocz.cn/400883.htm
etj.hjiocz.cn/993153.htm
etj.hjiocz.cn/426863.htm
eth.hjiocz.cn/551393.htm
eth.hjiocz.cn/602083.htm
eth.hjiocz.cn/935533.htm
eth.hjiocz.cn/862603.htm
eth.hjiocz.cn/331173.htm
eth.hjiocz.cn/004803.htm
eth.hjiocz.cn/268223.htm
eth.hjiocz.cn/440263.htm
eth.hjiocz.cn/066663.htm
eth.hjiocz.cn/648843.htm
etg.hjiocz.cn/595513.htm
etg.hjiocz.cn/177553.htm
etg.hjiocz.cn/133393.htm
etg.hjiocz.cn/177173.htm
etg.hjiocz.cn/686043.htm
etg.hjiocz.cn/460223.htm
etg.hjiocz.cn/402623.htm
etg.hjiocz.cn/317553.htm
etg.hjiocz.cn/804243.htm
etg.hjiocz.cn/957773.htm
etf.hjiocz.cn/608423.htm
etf.hjiocz.cn/719133.htm
etf.hjiocz.cn/668003.htm
etf.hjiocz.cn/842263.htm
etf.hjiocz.cn/880223.htm
etf.hjiocz.cn/640443.htm
etf.hjiocz.cn/937993.htm
etf.hjiocz.cn/319733.htm
etf.hjiocz.cn/771713.htm
etf.hjiocz.cn/482823.htm
