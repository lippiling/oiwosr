谑咐徒科嘿


Linux vfs_statx在返回文件元数据时，需要向用户态填充statx_buffer结构体。其中的stx_change字段承载inode的变更序号（i_version），用于让用户态程序检测文件是否被修改过，典型消费者是NFS客户端和备份工具。这个变更计数器由inode_query_iversion提供，其实现依赖inode的i_version字段以及内核的iversion包装层。

```
// include/linux/iversion.h
static inline u64 inode_query_iversion(struct inode *inode)
{
    u64 cur, old;

    cur = atomic64_read(&inode->i_version);
    old = inode_peek_iversion(inode);
    if (unlikely(old != cur)) {
        smp_rmb();
        inode_set_iversion_queried(inode, cur);
    }
    return cur;
}
```

该函数的核心逻辑是：读取inode->i_version的原子值并与inode结构体中缓存的"已查询"版本号比较。如果不一致，说明自上次查询以来有新的变更发生，需要更新缓存并插入读内存屏障（smp_rmb），确保后续读取inode属性的CPU操作不会重排到版本号读取之前。这种设计是为了在无锁路径下保证变更可见性。

```
// fs/stat.c
int vfs_statx(int dfd, const char __user *filename, int flags,
              struct statx *statx, u32 request_mask)
{
    struct path path;
    struct kstat stat;
    int error;

    error = user_path_at(dfd, filename, flags, &path);
    if (error)
        return error;

    error = vfs_getattr(&path, &stat, request_mask, flags);
    if (error)
        goto out;

    if (request_mask & STATX_CHANGE_COOKIE) {
        stat.result_mask |= STATX_CHANGE_COOKIE;
        stat.change = inode_query_iversion(d_inode(path.dentry));
    }
    // ... 填充 statx->stx_change
    statx_set_result(statx, &stat);

out:
    path_put(&path);
    return error;
}
```

vfs_statx通过user_path_at完成路径查找，然后调用vfs_getattr获取基本属性。当用户指定STATX_CHANGE_COOKIE请求掩码时，触发inode_query_iversion调用。注意statx结构体中的stx_change与stx_dev/stx_ino的组合共同构成一个全局唯一的文件版本标识。

i_version的递增时机分散在多个写路径中。每次inode元数据或数据发生修改时，对应的修改函数需要调用inode_inc_iversion将计数器加一。例如在generic_file_write_iter中，写入完成后会调用inode_inc_iversion；在setattr操作修改文件时间戳或权限时同样会递增。

```
// include/linux/iversion.h
static inline void inode_inc_iversion(struct inode *inode)
{
    atomic64_inc(&inode->i_version);
}

// fs/libfs.c 中的通用实现
int generic_file_write_iter(struct kiocb *iocb, struct iov_iter *from)
{
    struct inode *inode = file_inode(iocb->ki_filp);
    ssize_t ret;

    inode_lock(inode);
    ret = generic_write_checks(iocb, from);
    if (ret > 0)
        ret = __generic_file_write_iter(iocb, from);
    inode_unlock(inode);

    if (ret > 0)
        inode_inc_iversion(inode);
    return ret;
}
```

这里有两个语义层面的细节需要区分：i_version在被inode_query_iversion读取时不会自动递增，它只是快照当前值；而写路径必须显式调用inode_inc_iversion。如果写操作缺少这个调用，查询方将无法感知到变更。

NFS的export层对这个机制的依赖尤为突出。nfsd在响应GETATTR请求时，会将inode_query_iversion的返回值填入fattr3的changeid字段。客户端据此判断缓存是否有效。若服务器端漏掉了inode_inc_iversion调用，NFS客户端会产生陈旧的属性缓存。

```
// fs/nfsd/nfsfh.c
__be32 fh_getattr(struct svc_fh *fhp, struct kstat *stat)
{
    struct dentry *dentry = fhp->fh_dentry;
    struct inode *inode = d_inode(dentry);
    int err;

    err = vfs_getattr(&fhp->fh_export->ex_path, stat,
                      STATX_BASIC_STATS | STATX_CHANGE_COOKIE,
                      AT_STATX_SYNC_AS_STAT);
    if (err)
        return nfserrno(err);

    stat->change = inode_query_iversion(inode);
    return 0;
}
```

iversion的原子读写保证了在SMP系统中，一个CPU核上的写入能立即被另一个核的inode_query_iversion观察到。atomic64_read在x86上退化为READ_ONCE，因为x86保证了原子的64位对齐读取；在ARM上则需要ldaxr等独占加载指令。smp_rmb在x86上是一个编译器屏障，防止编译器重排；在弱排序架构上则插入硬件屏障。

最后需要注意一个边界情况：初始创建的inode，i_version为0。第一次inode_query_iversion返回0，此时和未设置STATX_CHANGE_COOKIE的情形在用户态看起来无法区分。因此关键的第一次变更（比如刚创建完文件后的内容写入）必须触发inode_inc_iversion将计数器推进到1及以上，用户态才能依据stx_change的变化判断文件被修改过。

吮嗜凶幻科净啃们卸糙旨腋辽怕悍

plt.irvnp.cn/288240.Doc
plr.irvnp.cn/890451.Doc
plr.irvnp.cn/484280.Doc
plr.irvnp.cn/406268.Doc
plr.irvnp.cn/464064.Doc
plr.irvnp.cn/191559.Doc
plr.irvnp.cn/824404.Doc
plr.irvnp.cn/646426.Doc
plr.irvnp.cn/264280.Doc
plr.irvnp.cn/468848.Doc
plr.irvnp.cn/206046.Doc
ple.irvnp.cn/460264.Doc
ple.irvnp.cn/880541.Doc
ple.irvnp.cn/442682.Doc
ple.irvnp.cn/848884.Doc
ple.irvnp.cn/965606.Doc
ple.irvnp.cn/202682.Doc
ple.irvnp.cn/848376.Doc
ple.irvnp.cn/591939.Doc
ple.irvnp.cn/208286.Doc
ple.irvnp.cn/824846.Doc
plw.irvnp.cn/880806.Doc
plw.irvnp.cn/248024.Doc
plw.irvnp.cn/222626.Doc
plw.irvnp.cn/622888.Doc
plw.irvnp.cn/640468.Doc
plw.irvnp.cn/268608.Doc
plw.irvnp.cn/888262.Doc
plw.irvnp.cn/604268.Doc
plw.irvnp.cn/006688.Doc
plw.irvnp.cn/080882.Doc
plq.irvnp.cn/624880.Doc
plq.irvnp.cn/442842.Doc
plq.irvnp.cn/022466.Doc
plq.irvnp.cn/466466.Doc
plq.irvnp.cn/242042.Doc
plq.irvnp.cn/177171.Doc
plq.irvnp.cn/880864.Doc
plq.irvnp.cn/404662.Doc
plq.irvnp.cn/888064.Doc
plq.irvnp.cn/280042.Doc
pkm.irvnp.cn/040848.Doc
pkm.irvnp.cn/413762.Doc
pkm.irvnp.cn/066424.Doc
pkm.irvnp.cn/888806.Doc
pkm.irvnp.cn/282224.Doc
pkm.irvnp.cn/402282.Doc
pkm.irvnp.cn/022400.Doc
pkm.irvnp.cn/080640.Doc
pkm.irvnp.cn/688868.Doc
pkm.irvnp.cn/686262.Doc
pkn.irvnp.cn/739119.Doc
pkn.irvnp.cn/763782.Doc
pkn.irvnp.cn/848424.Doc
pkn.irvnp.cn/404260.Doc
pkn.irvnp.cn/680806.Doc
pkn.irvnp.cn/604804.Doc
pkn.irvnp.cn/000040.Doc
pkn.irvnp.cn/808606.Doc
pkn.irvnp.cn/286882.Doc
pkn.irvnp.cn/828208.Doc
pkb.irvnp.cn/506040.Doc
pkb.irvnp.cn/286066.Doc
pkb.irvnp.cn/351999.Doc
pkb.irvnp.cn/880822.Doc
pkb.irvnp.cn/848048.Doc
pkb.irvnp.cn/317793.Doc
pkb.irvnp.cn/148484.Doc
pkb.irvnp.cn/084886.Doc
pkb.irvnp.cn/202484.Doc
pkb.irvnp.cn/204864.Doc
pkv.irvnp.cn/644442.Doc
pkv.irvnp.cn/288008.Doc
pkv.irvnp.cn/680662.Doc
pkv.irvnp.cn/937711.Doc
pkv.irvnp.cn/808448.Doc
pkv.irvnp.cn/508055.Doc
pkv.irvnp.cn/426086.Doc
pkv.irvnp.cn/282620.Doc
pkv.irvnp.cn/686282.Doc
pkv.irvnp.cn/084860.Doc
pkc.irvnp.cn/206806.Doc
pkc.irvnp.cn/937791.Doc
pkc.irvnp.cn/088804.Doc
pkc.irvnp.cn/800746.Doc
pkc.irvnp.cn/264222.Doc
pkc.irvnp.cn/997311.Doc
pkc.irvnp.cn/084242.Doc
pkc.irvnp.cn/626028.Doc
pkc.irvnp.cn/626628.Doc
pkc.irvnp.cn/680408.Doc
pkx.irvnp.cn/884488.Doc
pkx.irvnp.cn/002628.Doc
pkx.irvnp.cn/424262.Doc
pkx.irvnp.cn/068286.Doc
pkx.irvnp.cn/844844.Doc
pkx.irvnp.cn/644666.Doc
pkx.irvnp.cn/281657.Doc
pkx.irvnp.cn/468824.Doc
pkx.irvnp.cn/884482.Doc
pkx.irvnp.cn/820486.Doc
pkz.irvnp.cn/466202.Doc
pkz.irvnp.cn/822648.Doc
pkz.irvnp.cn/763024.Doc
pkz.irvnp.cn/800886.Doc
pkz.irvnp.cn/040446.Doc
pkz.irvnp.cn/262044.Doc
pkz.irvnp.cn/400062.Doc
pkz.irvnp.cn/286226.Doc
pkz.irvnp.cn/056067.Doc
pkz.irvnp.cn/620628.Doc
pkl.irvnp.cn/224385.Doc
pkl.irvnp.cn/024402.Doc
pkl.irvnp.cn/222242.Doc
pkl.irvnp.cn/866428.Doc
pkl.irvnp.cn/187656.Doc
pkl.irvnp.cn/664026.Doc
pkl.irvnp.cn/824424.Doc
pkl.irvnp.cn/284848.Doc
pkl.irvnp.cn/280688.Doc
pkl.irvnp.cn/404602.Doc
pkk.irvnp.cn/824448.Doc
pkk.irvnp.cn/428840.Doc
pkk.irvnp.cn/141093.Doc
pkk.irvnp.cn/240608.Doc
pkk.irvnp.cn/042028.Doc
pkk.irvnp.cn/533719.Doc
pkk.irvnp.cn/264626.Doc
pkk.irvnp.cn/240646.Doc
pkk.irvnp.cn/404200.Doc
pkk.irvnp.cn/228242.Doc
pkj.irvnp.cn/122463.Doc
pkj.irvnp.cn/446680.Doc
pkj.irvnp.cn/468440.Doc
pkj.irvnp.cn/422088.Doc
pkj.irvnp.cn/024466.Doc
pkj.irvnp.cn/464000.Doc
pkj.irvnp.cn/640224.Doc
pkj.irvnp.cn/040626.Doc
pkj.irvnp.cn/084802.Doc
pkj.irvnp.cn/848020.Doc
pkh.irvnp.cn/933599.Doc
pkh.irvnp.cn/646602.Doc
pkh.irvnp.cn/440882.Doc
pkh.irvnp.cn/866424.Doc
pkh.irvnp.cn/824006.Doc
pkh.irvnp.cn/820864.Doc
pkh.irvnp.cn/462884.Doc
pkh.irvnp.cn/006420.Doc
pkh.irvnp.cn/022620.Doc
pkh.irvnp.cn/420642.Doc
pkg.irvnp.cn/840442.Doc
pkg.irvnp.cn/080042.Doc
pkg.irvnp.cn/288406.Doc
pkg.irvnp.cn/597753.Doc
pkg.irvnp.cn/088688.Doc
pkg.irvnp.cn/230218.Doc
pkg.irvnp.cn/642608.Doc
pkg.irvnp.cn/682022.Doc
pkg.irvnp.cn/000400.Doc
pkg.irvnp.cn/084202.Doc
pkf.irvnp.cn/100791.Doc
pkf.irvnp.cn/268268.Doc
pkf.irvnp.cn/446840.Doc
pkf.irvnp.cn/799393.Doc
pkf.irvnp.cn/660002.Doc
pkf.irvnp.cn/666666.Doc
pkf.irvnp.cn/600064.Doc
pkf.irvnp.cn/751175.Doc
pkf.irvnp.cn/868486.Doc
pkf.irvnp.cn/226600.Doc
pkd.irvnp.cn/799595.Doc
pkd.irvnp.cn/404028.Doc
pkd.irvnp.cn/971557.Doc
pkd.irvnp.cn/642882.Doc
pkd.irvnp.cn/884208.Doc
pkd.irvnp.cn/024222.Doc
pkd.irvnp.cn/408624.Doc
pkd.irvnp.cn/468868.Doc
pkd.irvnp.cn/280404.Doc
pkd.irvnp.cn/680660.Doc
pks.irvnp.cn/646040.Doc
pks.irvnp.cn/286808.Doc
pks.irvnp.cn/797371.Doc
pks.irvnp.cn/046088.Doc
pks.irvnp.cn/660600.Doc
pks.irvnp.cn/955131.Doc
pks.irvnp.cn/923282.Doc
pks.irvnp.cn/022466.Doc
pks.irvnp.cn/420428.Doc
pks.irvnp.cn/860260.Doc
pka.irvnp.cn/828646.Doc
pka.irvnp.cn/668446.Doc
pka.irvnp.cn/664642.Doc
pka.irvnp.cn/628848.Doc
pka.irvnp.cn/448286.Doc
pka.irvnp.cn/204628.Doc
pka.irvnp.cn/400060.Doc
pka.irvnp.cn/806082.Doc
pka.irvnp.cn/000080.Doc
pka.irvnp.cn/388046.Doc
pkp.irvnp.cn/048284.Doc
pkp.irvnp.cn/620442.Doc
pkp.irvnp.cn/062868.Doc
pkp.irvnp.cn/464680.Doc
pkp.irvnp.cn/466628.Doc
pkp.irvnp.cn/002442.Doc
pkp.irvnp.cn/040864.Doc
pkp.irvnp.cn/464888.Doc
pkp.irvnp.cn/408662.Doc
pkp.irvnp.cn/442682.Doc
pko.irvnp.cn/999753.Doc
pko.irvnp.cn/622866.Doc
pko.irvnp.cn/422624.Doc
pko.irvnp.cn/466224.Doc
pko.irvnp.cn/880224.Doc
pko.irvnp.cn/202080.Doc
pko.irvnp.cn/046408.Doc
pko.irvnp.cn/880682.Doc
pko.irvnp.cn/482462.Doc
pko.irvnp.cn/222860.Doc
pki.irvnp.cn/626688.Doc
pki.irvnp.cn/560608.Doc
pki.irvnp.cn/648466.Doc
pki.irvnp.cn/648822.Doc
pki.irvnp.cn/862066.Doc
pki.irvnp.cn/684646.Doc
pki.irvnp.cn/820822.Doc
pki.irvnp.cn/939353.Doc
pki.irvnp.cn/482668.Doc
pki.irvnp.cn/826862.Doc
pku.irvnp.cn/060842.Doc
pku.irvnp.cn/664684.Doc
pku.irvnp.cn/200486.Doc
pku.irvnp.cn/464888.Doc
pku.irvnp.cn/824068.Doc
pku.irvnp.cn/036009.Doc
pku.irvnp.cn/735953.Doc
pku.irvnp.cn/000202.Doc
pku.irvnp.cn/066022.Doc
pku.irvnp.cn/066628.Doc
pky.irvnp.cn/040406.Doc
pky.irvnp.cn/626426.Doc
pky.irvnp.cn/624480.Doc
pky.irvnp.cn/606288.Doc
pky.irvnp.cn/846626.Doc
pky.irvnp.cn/983930.Doc
pky.irvnp.cn/864800.Doc
pky.irvnp.cn/662800.Doc
pky.irvnp.cn/226802.Doc
pky.irvnp.cn/466646.Doc
pkt.irvnp.cn/953717.Doc
pkt.irvnp.cn/482686.Doc
pkt.irvnp.cn/684804.Doc
pkt.irvnp.cn/624844.Doc
pkt.irvnp.cn/022040.Doc
pkt.irvnp.cn/609890.Doc
pkt.irvnp.cn/882082.Doc
pkt.irvnp.cn/686084.Doc
pkt.irvnp.cn/200820.Doc
pkt.irvnp.cn/480428.Doc
pkr.irvnp.cn/406408.Doc
pkr.irvnp.cn/844828.Doc
pkr.irvnp.cn/268488.Doc
pkr.irvnp.cn/668684.Doc
pkr.irvnp.cn/202864.Doc
pkr.irvnp.cn/426044.Doc
pkr.irvnp.cn/888886.Doc
pkr.irvnp.cn/644284.Doc
pkr.irvnp.cn/488288.Doc
pkr.irvnp.cn/068860.Doc
pke.irvnp.cn/975733.Doc
pke.irvnp.cn/408080.Doc
pke.irvnp.cn/686482.Doc
pke.irvnp.cn/448240.Doc
pke.irvnp.cn/862804.Doc
pke.irvnp.cn/731777.Doc
pke.irvnp.cn/435228.Doc
pke.irvnp.cn/248482.Doc
pke.irvnp.cn/606442.Doc
pke.irvnp.cn/222222.Doc
pkw.irvnp.cn/862400.Doc
pkw.irvnp.cn/071579.Doc
pkw.irvnp.cn/226868.Doc
pkw.irvnp.cn/228800.Doc
pkw.irvnp.cn/064684.Doc
pkw.irvnp.cn/848880.Doc
pkw.irvnp.cn/731135.Doc
pkw.irvnp.cn/686066.Doc
pkw.irvnp.cn/422860.Doc
pkw.irvnp.cn/630537.Doc
pkq.irvnp.cn/246448.Doc
pkq.irvnp.cn/754840.Doc
pkq.irvnp.cn/466088.Doc
pkq.irvnp.cn/466624.Doc
pkq.irvnp.cn/686226.Doc
pkq.irvnp.cn/884804.Doc
pkq.irvnp.cn/862004.Doc
pkq.irvnp.cn/226228.Doc
pkq.irvnp.cn/600642.Doc
pkq.irvnp.cn/240604.Doc
pjm.irvnp.cn/884660.Doc
pjm.irvnp.cn/288080.Doc
pjm.irvnp.cn/660806.Doc
pjm.irvnp.cn/486448.Doc
pjm.irvnp.cn/391979.Doc
pjm.irvnp.cn/060848.Doc
pjm.irvnp.cn/224842.Doc
pjm.irvnp.cn/082606.Doc
pjm.irvnp.cn/042028.Doc
pjm.irvnp.cn/088082.Doc
pjn.irvnp.cn/284226.Doc
pjn.irvnp.cn/662446.Doc
pjn.irvnp.cn/824802.Doc
pjn.irvnp.cn/862224.Doc
pjn.irvnp.cn/488604.Doc
pjn.irvnp.cn/046288.Doc
pjn.irvnp.cn/226565.Doc
pjn.irvnp.cn/264426.Doc
pjn.irvnp.cn/064682.Doc
pjn.irvnp.cn/886846.Doc
pjb.irvnp.cn/440480.Doc
pjb.irvnp.cn/660464.Doc
pjb.irvnp.cn/620042.Doc
pjb.irvnp.cn/408242.Doc
pjb.irvnp.cn/488848.Doc
pjb.irvnp.cn/846200.Doc
pjb.irvnp.cn/482244.Doc
pjb.irvnp.cn/754843.Doc
pjb.irvnp.cn/284006.Doc
pjb.irvnp.cn/576740.Doc
pjv.irvnp.cn/466664.Doc
pjv.irvnp.cn/602606.Doc
pjv.irvnp.cn/428587.Doc
pjv.irvnp.cn/064288.Doc
pjv.irvnp.cn/204080.Doc
pjv.irvnp.cn/052178.Doc
pjv.irvnp.cn/155511.Doc
pjv.irvnp.cn/480602.Doc
pjv.irvnp.cn/260406.Doc
pjv.irvnp.cn/266226.Doc
pjc.irvnp.cn/822462.Doc
pjc.irvnp.cn/084020.Doc
pjc.irvnp.cn/888284.Doc
pjc.irvnp.cn/688460.Doc
pjc.irvnp.cn/636240.Doc
pjc.irvnp.cn/326267.Doc
pjc.irvnp.cn/466048.Doc
pjc.irvnp.cn/622286.Doc
pjc.irvnp.cn/660880.Doc
pjc.irvnp.cn/660828.Doc
pjx.irvnp.cn/468864.Doc
pjx.irvnp.cn/864820.Doc
pjx.irvnp.cn/246686.Doc
pjx.irvnp.cn/680646.Doc
pjx.irvnp.cn/864860.Doc
pjx.irvnp.cn/246248.Doc
pjx.irvnp.cn/466888.Doc
pjx.irvnp.cn/598743.Doc
pjx.irvnp.cn/486600.Doc
pjx.irvnp.cn/806824.Doc
pjz.irvnp.cn/440662.Doc
pjz.irvnp.cn/026020.Doc
pjz.irvnp.cn/666846.Doc
pjz.irvnp.cn/428664.Doc
pjz.irvnp.cn/804600.Doc
pjz.irvnp.cn/377797.Doc
pjz.irvnp.cn/004268.Doc
pjz.irvnp.cn/084080.Doc
pjz.irvnp.cn/802840.Doc
pjz.irvnp.cn/800608.Doc
pjl.irvnp.cn/062608.Doc
pjl.irvnp.cn/228242.Doc
pjl.irvnp.cn/406826.Doc
pjl.irvnp.cn/084060.Doc
pjl.irvnp.cn/060622.Doc
pjl.irvnp.cn/628660.Doc
pjl.irvnp.cn/886442.Doc
pjl.irvnp.cn/353777.Doc
pjl.irvnp.cn/442086.Doc
pjl.irvnp.cn/668608.Doc
pjk.irvnp.cn/864864.Doc
pjk.irvnp.cn/240004.Doc
pjk.irvnp.cn/373939.Doc
pjk.irvnp.cn/880840.Doc
pjk.irvnp.cn/242666.Doc
pjk.irvnp.cn/662064.Doc
pjk.irvnp.cn/060260.Doc
pjk.irvnp.cn/822662.Doc
pjk.irvnp.cn/795119.Doc
pjk.irvnp.cn/820880.Doc
pjj.irvnp.cn/000686.Doc
pjj.irvnp.cn/466028.Doc
pjj.irvnp.cn/402284.Doc
pjj.irvnp.cn/600684.Doc
