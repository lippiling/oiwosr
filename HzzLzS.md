嗡冈宜咳吃


Linux sys_stat vfs_statx与cp_new_stat64填充

stat(2) 系列系统调用在内核的统一入口是 vfs_statx。64 位架构上，通过 ksys_stat 进入：

```c
long ksys_stat(const char __user *filename, struct kstat __user *statbuf)
{
    struct kstat stat;
    int error;

    error = vfs_stat(filename, &stat);
    if (error)
        return error;

    return cp_new_stat(stat, statbuf);
}
```

vfs_stat 是对 vfs_statx 的简单的封装：

```c
int vfs_stat(const char __user *name, struct kstat *stat)
{
    return vfs_statx(AT_FDCWD, name, 0, stat, STATX_BASIC_STATS);
}
```

vfs_statx 是 VFS stat 路径的核心函数：

```c
int vfs_statx(int dfd, const char __user *filename, int flags,
              struct kstat *stat, u32 request_mask)
{
    struct path path;
    unsigned int lookup_flags = 0;
    int error;

    if (flags & ~(AT_SYMLINK_NOFOLLOW | AT_NO_AUTOMOUNT | AT_EMPTY_PATH |
                  AT_STATX_SYNC_TYPE))
        return -EINVAL;

    if (flags & AT_SYMLINK_NOFOLLOW)
        lookup_flags |= LOOKUP_FOLLOW;

retry:
    error = user_path_at(dfd, filename, lookup_flags, &path);
    if (error)
        goto out;

    error = vfs_getattr(&path, stat, request_mask, flags);
    path_put(&path);

    if (retry_estale(error, lookup_flags)) {
        lookup_flags |= LOOKUP_REVAL;
        goto retry;
    }
out:
    return error;
}
```

user_path_at 使用 path_init + link_path_walk 的路径遍历机制将用户态路径解析为 struct path（包含 dentry 和 vfsmount）。如果文件名是"/proc/self/exe"或包含符号链接，路径遍历会在适当位置跟踪符号链接。

vfs_getattr 是获取文件属性的调度中心：

```c
int vfs_getattr(const struct path *path, struct kstat *stat,
                u32 request_mask, unsigned int query_flags)
{
    struct inode *inode = d_backing_inode(path->dentry);

    memset(stat, 0, sizeof(*stat));
    stat->result_mask = 0;

    if (inode->i_op->getattr)
        return inode->i_op->getattr(path, stat, request_mask, query_flags);

    generic_fillattr(inode, stat);
    return 0;
}
```

大多数文件系统都实现了自己的 getattr。ext4 的 ext4_getattr 在调用 generic_fillattr 之前额外检查 inode 是否具有 EXT4_EA_INODE_FL 标志，并报告 i_projid（project ID）给 statx。

generic_fillattr 将 inode 中的原始信息填入 struct kstat：

```c
void generic_fillattr(struct inode *inode, struct kstat *stat)
{
    stat->dev     = inode->i_sb->s_dev;
    stat->ino     = inode->i_ino;
    stat->mode    = inode->i_mode;
    stat->nlink   = inode->i_nlink;
    stat->uid     = from_kuid_munged(&init_user_ns, inode->i_uid);
    stat->gid     = from_kgid_munged(&init_user_ns, inode->i_gid);
    stat->rdev    = inode->i_rdev;
    stat->size    = i_size_read(inode);
    stat->atime   = inode->i_atime;
    stat->mtime   = inode->i_mtime;
    stat->ctime   = inode->i_ctime;
    stat->blksize = i_blocksize(inode);
    stat->blocks  = inode->i_blocks;
}
```

struct kstat 是内核内部的通用结构体，包含 64 位的时间戳、块大小和块计数等信息，不依赖架构。但用户态的 struct stat 在不同架构上布局不同。cp_new_stat 负责将 kstat 转换为用户态可见的格式：

```c
static int cp_new_stat(struct kstat *stat, struct stat __user *statbuf)
{
    struct stat tmp;

    tmp.st_dev     = new_encode_dev(stat->dev);
    tmp.st_ino     = stat->ino;
    tmp.st_mode    = stat->mode;
    tmp.st_nlink   = stat->nlink;
    tmp.st_uid     = from_kuid_munged(&init_user_ns, stat->uid);
    tmp.st_gid     = from_kgid_munged(&init_user_ns, stat->gid);
    tmp.st_rdev    = new_encode_dev(stat->rdev);
    tmp.st_size    = stat->size;
    tmp.st_atime   = stat->atime.tv_sec;
    tmp.st_mtime   = stat->mtime.tv_sec;
    tmp.st_ctime   = stat->ctime.tv_sec;

#ifdef STAT_HAVE_NSEC
    tmp.st_atime_nsec = stat->atime.tv_nsec;
    tmp.st_mtime_nsec = stat->mtime.tv_nsec;
    tmp.st_ctime_nsec = stat->ctime.tv_nsec;
#endif

    tmp.st_blocks  = stat->blocks;
    tmp.st_blksize = stat->blksize;
    return copy_to_user(statbuf, &tmp, sizeof(tmp));
}
```

在 x86_64 架构上，struct stat 的大小为 144 字节，包含 st_atime_nsec 等纳秒字段。new_encode_dev 将内核使用的 dev_t（包含 major 和 minor 号）编码为用户态可理解的形式。

对于 32 位用户态在 64 位内核上运行（CONFIG_COMPAT），需要 compat_sys_stat64 或 compat_sys_newfstatat 等入口。compat 版本使用 struct compat_stat64，其字段宽度与 32 位 glibc 对齐，避免在 64 位内核和 32 位用户态之间传递错误布局的数据。

statx(2) 系统调用提供了更细粒度的控制。它不经过传统的 vfs_stat 路径，而是直接调用 vfs_statx 后返回 struct statx 给用户态。struct statx 包含 stx_mask 位图标记哪些字段有效，允许调用方只请求自己关心的属性（request_mask），在网络文件系统等场景下减少 RPC 开销。
杆副竞卤揖战瞧枪酱辣谡挖炙吐智

qnh.aira2hc.cn/177915.htm
qnh.aira2hc.cn/266865.htm
qnh.aira2hc.cn/957755.htm
qnh.aira2hc.cn/802485.htm
qnh.aira2hc.cn/373515.htm
qnh.aira2hc.cn/886445.htm
qnh.aira2hc.cn/573575.htm
qnh.aira2hc.cn/755995.htm
qng.aira2hc.cn/999955.htm
qng.aira2hc.cn/335995.htm
qng.aira2hc.cn/868845.htm
qng.aira2hc.cn/480685.htm
qng.aira2hc.cn/199135.htm
qng.aira2hc.cn/806665.htm
qng.aira2hc.cn/006445.htm
qng.aira2hc.cn/640085.htm
qng.aira2hc.cn/828845.htm
qng.aira2hc.cn/460245.htm
qnf.aira2hc.cn/404085.htm
qnf.aira2hc.cn/773575.htm
qnf.aira2hc.cn/666205.htm
qnf.aira2hc.cn/682465.htm
qnf.aira2hc.cn/840485.htm
qnf.aira2hc.cn/844825.htm
qnf.aira2hc.cn/531935.htm
qnf.aira2hc.cn/844265.htm
qnf.aira2hc.cn/642205.htm
qnf.aira2hc.cn/084425.htm
qnd.aira2hc.cn/006045.htm
qnd.aira2hc.cn/060845.htm
qnd.aira2hc.cn/195755.htm
qnd.aira2hc.cn/082285.htm
qnd.aira2hc.cn/462485.htm
qnd.aira2hc.cn/159715.htm
qnd.aira2hc.cn/080265.htm
qnd.aira2hc.cn/355515.htm
qnd.aira2hc.cn/355175.htm
qnd.aira2hc.cn/624645.htm
qns.aira2hc.cn/537575.htm
qns.aira2hc.cn/044485.htm
qns.aira2hc.cn/242845.htm
qns.aira2hc.cn/157795.htm
qns.aira2hc.cn/519555.htm
qns.aira2hc.cn/488625.htm
qns.aira2hc.cn/000265.htm
qns.aira2hc.cn/800245.htm
qns.aira2hc.cn/151315.htm
qns.aira2hc.cn/862665.htm
qna.aira2hc.cn/717135.htm
qna.aira2hc.cn/933155.htm
qna.aira2hc.cn/666685.htm
qna.aira2hc.cn/088805.htm
qna.aira2hc.cn/080005.htm
qna.aira2hc.cn/602265.htm
qna.aira2hc.cn/268425.htm
qna.aira2hc.cn/135375.htm
qna.aira2hc.cn/000005.htm
qna.aira2hc.cn/753915.htm
qnp.aira2hc.cn/084045.htm
qnp.aira2hc.cn/648465.htm
qnp.aira2hc.cn/555975.htm
qnp.aira2hc.cn/913395.htm
qnp.aira2hc.cn/311975.htm
qnp.aira2hc.cn/026645.htm
qnp.aira2hc.cn/597795.htm
qnp.aira2hc.cn/260885.htm
qnp.aira2hc.cn/848865.htm
qnp.aira2hc.cn/842645.htm
qno.aira2hc.cn/422265.htm
qno.aira2hc.cn/824065.htm
qno.aira2hc.cn/719135.htm
qno.aira2hc.cn/666265.htm
qno.aira2hc.cn/464485.htm
qno.aira2hc.cn/993595.htm
qno.aira2hc.cn/840065.htm
qno.aira2hc.cn/688005.htm
qno.aira2hc.cn/024825.htm
qno.aira2hc.cn/224605.htm
qni.aira2hc.cn/359795.htm
qni.aira2hc.cn/080085.htm
qni.aira2hc.cn/800205.htm
qni.aira2hc.cn/628085.htm
qni.aira2hc.cn/644845.htm
qni.aira2hc.cn/999575.htm
qni.aira2hc.cn/246485.htm
qni.aira2hc.cn/440645.htm
qni.aira2hc.cn/020005.htm
qni.aira2hc.cn/044665.htm
qnu.aira2hc.cn/866245.htm
qnu.aira2hc.cn/422245.htm
qnu.aira2hc.cn/020605.htm
qnu.aira2hc.cn/264885.htm
qnu.aira2hc.cn/828425.htm
qnu.aira2hc.cn/771355.htm
qnu.aira2hc.cn/844885.htm
qnu.aira2hc.cn/175535.htm
qnu.aira2hc.cn/715135.htm
qnu.aira2hc.cn/008445.htm
qny.aira2hc.cn/008825.htm
qny.aira2hc.cn/802885.htm
qny.aira2hc.cn/713375.htm
qny.aira2hc.cn/759795.htm
qny.aira2hc.cn/800805.htm
qny.aira2hc.cn/179535.htm
qny.aira2hc.cn/688405.htm
qny.aira2hc.cn/004645.htm
qny.aira2hc.cn/600205.htm
qny.aira2hc.cn/486285.htm
qnt.aira2hc.cn/995535.htm
qnt.aira2hc.cn/628465.htm
qnt.aira2hc.cn/642065.htm
qnt.aira2hc.cn/286205.htm
qnt.aira2hc.cn/606665.htm
qnt.aira2hc.cn/606245.htm
qnt.aira2hc.cn/220605.htm
qnt.aira2hc.cn/208645.htm
qnt.aira2hc.cn/268205.htm
qnt.aira2hc.cn/846225.htm
qnr.aira2hc.cn/886085.htm
qnr.aira2hc.cn/828005.htm
qnr.aira2hc.cn/608285.htm
qnr.aira2hc.cn/426265.htm
qnr.aira2hc.cn/268685.htm
qnr.aira2hc.cn/359135.htm
qnr.aira2hc.cn/266865.htm
qnr.aira2hc.cn/575915.htm
qnr.aira2hc.cn/666205.htm
qnr.aira2hc.cn/626205.htm
qne.aira2hc.cn/135375.htm
qne.aira2hc.cn/571795.htm
qne.aira2hc.cn/202845.htm
qne.aira2hc.cn/008225.htm
qne.aira2hc.cn/462025.htm
qne.aira2hc.cn/371995.htm
qne.aira2hc.cn/533335.htm
qne.aira2hc.cn/626005.htm
qne.aira2hc.cn/737795.htm
qne.aira2hc.cn/282465.htm
qnw.aira2hc.cn/640405.htm
qnw.aira2hc.cn/517555.htm
qnw.aira2hc.cn/664825.htm
qnw.aira2hc.cn/751375.htm
qnw.aira2hc.cn/808825.htm
qnw.aira2hc.cn/628485.htm
qnw.aira2hc.cn/080665.htm
qnw.aira2hc.cn/284245.htm
qnw.aira2hc.cn/317195.htm
qnw.aira2hc.cn/488465.htm
qnq.aira2hc.cn/979575.htm
qnq.aira2hc.cn/640445.htm
qnq.aira2hc.cn/660625.htm
qnq.aira2hc.cn/111775.htm
qnq.aira2hc.cn/402425.htm
qnq.aira2hc.cn/844625.htm
qnq.aira2hc.cn/179995.htm
qnq.aira2hc.cn/684265.htm
qnq.aira2hc.cn/175575.htm
qnq.aira2hc.cn/751355.htm
qbtv.aira2hc.cn/228065.htm
qbtv.aira2hc.cn/282825.htm
qbtv.aira2hc.cn/35.htm
qbtv.aira2hc.cn/882845.htm
qbtv.aira2hc.cn/240625.htm
qbtv.aira2hc.cn/462045.htm
qbtv.aira2hc.cn/068865.htm
qbtv.aira2hc.cn/977115.htm
qbtv.aira2hc.cn/137755.htm
qbtv.aira2hc.cn/246045.htm
qbn.aira2hc.cn/024245.htm
qbn.aira2hc.cn/351555.htm
qbn.aira2hc.cn/995975.htm
qbn.aira2hc.cn/175595.htm
qbn.aira2hc.cn/955395.htm
qbn.aira2hc.cn/377755.htm
qbn.aira2hc.cn/642825.htm
qbn.aira2hc.cn/462825.htm
qbn.aira2hc.cn/991795.htm
qbn.aira2hc.cn/240225.htm
qbb.aira2hc.cn/375555.htm
qbb.aira2hc.cn/062025.htm
qbb.aira2hc.cn/028265.htm
qbb.aira2hc.cn/462065.htm
qbb.aira2hc.cn/808405.htm
qbb.aira2hc.cn/880885.htm
qbb.aira2hc.cn/440805.htm
qbb.aira2hc.cn/288205.htm
qbb.aira2hc.cn/775795.htm
qbb.aira2hc.cn/660845.htm
qbv.aira2hc.cn/197795.htm
qbv.aira2hc.cn/713975.htm
qbv.aira2hc.cn/777775.htm
qbv.aira2hc.cn/228225.htm
qbv.aira2hc.cn/248025.htm
qbv.aira2hc.cn/464885.htm
qbv.aira2hc.cn/571155.htm
qbv.aira2hc.cn/626805.htm
qbv.aira2hc.cn/000885.htm
qbv.aira2hc.cn/115755.htm
qbc.aira2hc.cn/602885.htm
qbc.aira2hc.cn/806485.htm
qbc.aira2hc.cn/008065.htm
qbc.aira2hc.cn/599535.htm
qbc.aira2hc.cn/042085.htm
qbc.aira2hc.cn/286265.htm
qbc.aira2hc.cn/860605.htm
qbc.aira2hc.cn/628005.htm
qbc.aira2hc.cn/197355.htm
qbc.aira2hc.cn/004465.htm
qbx.aira2hc.cn/284405.htm
qbx.aira2hc.cn/844025.htm
qbx.aira2hc.cn/288845.htm
qbx.aira2hc.cn/646225.htm
qbx.aira2hc.cn/462485.htm
qbx.aira2hc.cn/440205.htm
qbx.aira2hc.cn/777975.htm
qbx.aira2hc.cn/351155.htm
qbx.aira2hc.cn/686065.htm
qbx.aira2hc.cn/179335.htm
qbz.aira2hc.cn/224265.htm
qbz.aira2hc.cn/15.htm
qbz.aira2hc.cn/795935.htm
qbz.aira2hc.cn/882465.htm
qbz.aira2hc.cn/688425.htm
qbz.aira2hc.cn/662685.htm
qbz.aira2hc.cn/517155.htm
qbz.aira2hc.cn/486485.htm
qbz.aira2hc.cn/804665.htm
qbz.aira2hc.cn/333935.htm
qbl.aira2hc.cn/826625.htm
qbl.aira2hc.cn/066045.htm
qbl.aira2hc.cn/006665.htm
qbl.aira2hc.cn/977155.htm
qbl.aira2hc.cn/600605.htm
qbl.aira2hc.cn/088025.htm
qbl.aira2hc.cn/208025.htm
qbl.aira2hc.cn/955355.htm
qbl.aira2hc.cn/137735.htm
qbl.aira2hc.cn/513195.htm
qbk.aira2hc.cn/424065.htm
qbk.aira2hc.cn/204205.htm
qbk.aira2hc.cn/808285.htm
qbk.aira2hc.cn/882665.htm
qbk.aira2hc.cn/224005.htm
qbk.aira2hc.cn/820605.htm
qbk.aira2hc.cn/242025.htm
qbk.aira2hc.cn/688445.htm
qbk.aira2hc.cn/644425.htm
qbk.aira2hc.cn/622665.htm
qbj.aira2hc.cn/826045.htm
qbj.aira2hc.cn/288245.htm
qbj.aira2hc.cn/424445.htm
qbj.aira2hc.cn/042425.htm
qbj.aira2hc.cn/604005.htm
qbj.aira2hc.cn/286485.htm
qbj.aira2hc.cn/931535.htm
qbj.aira2hc.cn/533775.htm
qbj.aira2hc.cn/880005.htm
qbj.aira2hc.cn/864825.htm
qbh.aira2hc.cn/804085.htm
qbh.aira2hc.cn/202285.htm
qbh.aira2hc.cn/062265.htm
qbh.aira2hc.cn/262685.htm
qbh.aira2hc.cn/408405.htm
qbh.aira2hc.cn/062205.htm
qbh.aira2hc.cn/531935.htm
qbh.aira2hc.cn/820265.htm
qbh.aira2hc.cn/753355.htm
qbh.aira2hc.cn/224425.htm
qbg.aira2hc.cn/488445.htm
qbg.aira2hc.cn/424065.htm
qbg.aira2hc.cn/535715.htm
qbg.aira2hc.cn/395795.htm
qbg.aira2hc.cn/444065.htm
qbg.aira2hc.cn/088685.htm
qbg.aira2hc.cn/155355.htm
qbg.aira2hc.cn/888805.htm
qbg.aira2hc.cn/426605.htm
qbg.aira2hc.cn/660845.htm
qbf.aira2hc.cn/022085.htm
qbf.aira2hc.cn/953755.htm
qbf.aira2hc.cn/680025.htm
qbf.aira2hc.cn/977715.htm
qbf.aira2hc.cn/204225.htm
qbf.aira2hc.cn/886045.htm
qbf.aira2hc.cn/755175.htm
qbf.aira2hc.cn/642045.htm
qbf.aira2hc.cn/999195.htm
qbf.aira2hc.cn/264265.htm
qbd.aira2hc.cn/468805.htm
qbd.aira2hc.cn/939375.htm
qbd.aira2hc.cn/757935.htm
qbd.aira2hc.cn/684205.htm
qbd.aira2hc.cn/400265.htm
qbd.aira2hc.cn/826865.htm
qbd.aira2hc.cn/686625.htm
qbd.aira2hc.cn/193795.htm
qbd.aira2hc.cn/046685.htm
qbd.aira2hc.cn/062285.htm
qbs.aira2hc.cn/284685.htm
qbs.aira2hc.cn/826445.htm
qbs.aira2hc.cn/268225.htm
qbs.aira2hc.cn/020625.htm
qbs.aira2hc.cn/222865.htm
qbs.aira2hc.cn/595375.htm
qbs.aira2hc.cn/800485.htm
qbs.aira2hc.cn/080805.htm
qbs.aira2hc.cn/422665.htm
qbs.aira2hc.cn/662085.htm
qba.aira2hc.cn/262465.htm
qba.aira2hc.cn/448005.htm
qba.aira2hc.cn/179515.htm
qba.aira2hc.cn/391375.htm
qba.aira2hc.cn/842285.htm
qba.aira2hc.cn/840605.htm
qba.aira2hc.cn/402605.htm
qba.aira2hc.cn/375535.htm
qba.aira2hc.cn/466205.htm
qba.aira2hc.cn/046245.htm
qbp.aira2hc.cn/222865.htm
qbp.aira2hc.cn/628285.htm
qbp.aira2hc.cn/622205.htm
qbp.aira2hc.cn/806085.htm
qbp.aira2hc.cn/200445.htm
qbp.aira2hc.cn/826805.htm
qbp.aira2hc.cn/715155.htm
qbp.aira2hc.cn/848085.htm
qbp.aira2hc.cn/884025.htm
qbp.aira2hc.cn/842085.htm
qbo.aira2hc.cn/288645.htm
qbo.aira2hc.cn/080605.htm
qbo.aira2hc.cn/757395.htm
qbo.aira2hc.cn/717735.htm
qbo.aira2hc.cn/884205.htm
qbo.aira2hc.cn/591395.htm
qbo.aira2hc.cn/806045.htm
qbo.aira2hc.cn/228025.htm
qbo.aira2hc.cn/917395.htm
qbo.aira2hc.cn/915115.htm
qbi.aira2hc.cn/648865.htm
qbi.aira2hc.cn/244025.htm
qbi.aira2hc.cn/684465.htm
qbi.aira2hc.cn/797995.htm
qbi.aira2hc.cn/460445.htm
qbi.aira2hc.cn/480845.htm
qbi.aira2hc.cn/571955.htm
qbi.aira2hc.cn/555915.htm
qbi.aira2hc.cn/951595.htm
qbi.aira2hc.cn/751335.htm
qbu.aira2hc.cn/406205.htm
qbu.aira2hc.cn/153755.htm
qbu.aira2hc.cn/400285.htm
qbu.aira2hc.cn/260805.htm
qbu.aira2hc.cn/068065.htm
qbu.aira2hc.cn/880265.htm
qbu.aira2hc.cn/240885.htm
qbu.aira2hc.cn/179955.htm
qbu.aira2hc.cn/246465.htm
qbu.aira2hc.cn/319755.htm
qby.aira2hc.cn/519735.htm
qby.aira2hc.cn/204625.htm
qby.aira2hc.cn/884605.htm
qby.aira2hc.cn/480425.htm
qby.aira2hc.cn/802805.htm
qby.aira2hc.cn/975335.htm
qby.aira2hc.cn/624265.htm
qby.aira2hc.cn/840885.htm
qby.aira2hc.cn/602605.htm
qby.aira2hc.cn/000625.htm
qbt.aira2hc.cn/808285.htm
qbt.aira2hc.cn/800025.htm
qbt.aira2hc.cn/808865.htm
qbt.aira2hc.cn/488485.htm
qbt.aira2hc.cn/264065.htm
qbt.aira2hc.cn/044465.htm
qbt.aira2hc.cn/042405.htm
qbt.aira2hc.cn/579595.htm
qbt.aira2hc.cn/484485.htm
qbt.aira2hc.cn/484685.htm
qbr.aira2hc.cn/028485.htm
qbr.aira2hc.cn/446605.htm
qbr.aira2hc.cn/000285.htm
qbr.aira2hc.cn/913395.htm
qbr.aira2hc.cn/842685.htm
qbr.aira2hc.cn/939155.htm
qbr.aira2hc.cn/866625.htm
qbr.aira2hc.cn/084205.htm
qbr.aira2hc.cn/020245.htm
qbr.aira2hc.cn/066405.htm
qbe.aira2hc.cn/880445.htm
qbe.aira2hc.cn/804205.htm
qbe.aira2hc.cn/464625.htm
qbe.aira2hc.cn/282205.htm
qbe.aira2hc.cn/531135.htm
qbe.aira2hc.cn/864825.htm
qbe.aira2hc.cn/917535.htm
