障写疾钢侠


Linux overlayfs ovl_copy_up写时复制低层文件

OverlayFS的写时复制(Copy-up)机制是其核心设计——当下层文件被修改时，将数据从低层复制到上层可写层，之后所有修改在上层进行。ovl_copy_up是这一机制的入口函数，协同ovl_copy_up_data完成物理复制，并与后续的open/write操作形成完整的数据流。

一、copy-up的触发条件

写入打开下层文件时触发copy-up。ovl_open函数检查文件是否需要复制：

static int ovl_open(struct inode *inode, struct file *file)
{
    struct dentry *dentry = file_dentry(file);
    struct ovl_entry *oe = dentry->d_fsdata;
    bool want_write = file->f_mode & FMODE_WRITE;

    if (want_write && !oe->upper) {
        int err = ovl_want_write(dentry);
        if (err)
            return err;

        err = ovl_copy_up(dentry);
        if (err) {
            ovl_drop_write(dentry);
            return err;
        }
    }

    return generic_file_open(inode, file);
}

其他触发路径包括：setattr(更改权限或大小)、unlink(删除文件前的空目录copy-up)、rename(重命名目标文件)、setxattr和fallocate。ovl_copy_up调用链形成多层文件系统唯一的数据提升通道。

二、ovl_copy_up的目录与文件分发

ovl_copy_up作为主入口，根据dentry类型分派到ovl_copy_up_one或ovl_copy_up_locked：

int ovl_copy_up(struct dentry *dentry)
{
    int err;

    err = ovl_copy_up_one(dentry, NULL, 0);
    if (err == -EXDEV) {
        err = ovl_copy_up_locked(dentry);
    }

    return err;
}

ovl_copy_up_one处理普通文件和目录，ovl_copy_up_locked处理跨层复制(例如从低层直接创建硬链接)。两种路径最终调用ovl_copy_up_inode完成具体复制。

三、ovl_copy_up_inode的执行流程

ovl_copy_up_inode是copy-up的实质性实现：

int ovl_copy_up_inode(struct dentry *dentry, struct dentry *upperdir,
              struct kstat *stat)
{
    struct ovl_fs *ofs = dentry->d_sb->s_fs_info;
    struct dentry *upper;
    struct inode *newinode;
    const struct cred *old_cred;
    int err;

    old_cred = ovl_override_creds(dentry->d_sb);

    upper = ovl_lookup_upper(ofs, &dentry->d_name, upperdir, false);
    if (IS_ERR(upper)) {
        err = PTR_ERR(upper);
        goto out_revert;
    }

    if (!upper->d_inode) {
        umode_t mode = stat->mode;
        struct path lowerpath;

        ovl_path_lower(dentry, &lowerpath);

        if (S_ISREG(stat->mode)) {
            err = vfs_create(d_inode(upperdir), upper, mode, true);
        } else if (S_ISDIR(stat->mode)) {
            err = vfs_mkdir(d_inode(upperdir), upper, mode);
        } else if (S_ISLNK(stat->mode)) {
            err = vfs_symlink(d_inode(upperdir), upper,
                      ovl_get_link(dentry));
        } else if (S_ISFIFO(stat->mode) || S_ISSOCK(stat->mode)) {
            err = vfs_mknod(d_inode(upperdir), upper, mode,
                      0);
        } else if (S_ISBLK(stat->mode) || S_ISCHR(stat->mode)) {
            err = vfs_mknod(d_inode(upperdir), upper, mode,
                      stat->rdev);
        } else {
            err = -EPERM;
        }

        if (!err) {
            err = ovl_copy_up_data(dentry, upper, stat->size);
            if (!err) {
                struct iattr attr = {
                    .ia_valid = ATTR_MTIME | ATTR_CTIME,
                    .ia_mtime = stat->mtime,
                    .ia_ctime = stat->ctime,
                };
                inode_lock(d_inode(upper));
                notify_change(upper, &attr, NULL);
                inode_unlock(d_inode(upper));
            }
        }
    }

    dput(upper);

out_revert:
    revert_creds(old_cred);
    return err;
}

函数流程：
1. ovl_override_creds切换到overlayfs的挂载凭证，确保对上层的写入权限。
2. 在上层目录中查找或创建对应名称的dentry。
3. 根据下层文件类型调用vfs_create/vfs_mkdir/vfs_symlink等创建上层文件。
4. 调用ovl_copy_up_data复制文件内容。
5. 恢复原进程凭证。

四、ovl_copy_up_data的物理数据复制

ovl_copy_up_data负责将低层文件数据逐块复制到上层新文件：

int ovl_copy_up_data(struct dentry *dentry, struct dentry *upper,
             loff_t size)
{
    struct file *lower_file;
    struct file *upper_file;
    loff_t offset = 0;
    ssize_t bytes;
    int err;
    const size_t bufsize = 1 << 16;

    lower_file = ovl_path_open(dentry, O_RDONLY | O_LARGEFILE);
    if (IS_ERR(lower_file))
        return PTR_ERR(lower_file);

    upper_file = ovl_path_open(upper, O_WRONLY | O_LARGEFILE);
    if (IS_ERR(upper_file)) {
        fput(lower_file);
        return PTR_ERR(upper_file);
    }

    while (offset < size) {
        size_t this_read = min_t(size_t, bufsize, size - offset);
        void *buf = kmalloc(this_read, GFP_KERNEL);
        if (!buf) {
            err = -ENOMEM;
            goto out;
        }

        bytes = kernel_read(lower_file, buf, this_read, &offset);
        if (bytes < 0) {
            err = bytes;
            kfree(buf);
            goto out;
        }

        bytes = kernel_write(upper_file, buf, bytes, &offset);
        if (bytes < 0) {
            err = bytes;
            kfree(buf);
            goto out;
        }

        kfree(buf);
        offset += bytes;
    }

    err = 0;
out:
    fput(upper_file);
    fput(lower_file);
    return err;
}

复制使用64KB缓冲区大小，通过kernel_read/kernel_write在内核空间完成IO，避免用户空间上下文切换。复制过程中不持有VFS inode锁，仅通过文件的struct file引用计数控制生命周期，允许多个copy-up并发。

五、目录copy-up的递归策略

目录的copy-up不是递归复制所有子文件，而是仅创建空目录，子文件的copy-up延迟到各自被访问时。这种惰性(Lazy)策略的核心优势：

1. 只复制实际需要的文件，大型目录树的copy-up成本分摊到按需访问时。
2. 避免复制下层中可能已被上层白出的文件。
3. 对新文件分配inode并与下层inode建立关联，通过ovl_inode_update注册到overlay inode缓存。

目录copy-up后设置opaque扩展属性，防止后续lookup穿透到低层暴露旧文件。

六、硬链接与copy-up

当多个dentry指向同一低层inode时，首次写入触发copy-up创建上层新inode，其余dentry的copy-up检测到上层已存在相同inode，通过vfs_link创建硬链接关联到同一上层inode。ovl_copy_up_locked处理该场景：

static int ovl_copy_up_locked(struct dentry *dentry)
{
    struct dentry *upperdir;
    struct dentry *upper;
    struct dentry *origin = ovl_dentry_upper(dentry);
    int err;

    if (origin) {
        upperdir = dget(origin);
        goto link;
    }

    upperdir = ovl_dentry_upper(dentry->d_parent);
    if (!upperdir)
        upperdir = ovl_lookup_upper_ofs(dentry);

    err = ovl_copy_up_one(dentry, upperdir, 0);
    if (err)
        return err;

    return 0;

link:
    err = ovl_do_link(origin, d_inode(upperdir), dentry, NULL);
    return err;
}

七、凭证切换与安全性

copy-up使用ovl_override_creds将进程凭证切换为overlayfs挂载者的凭证，确保对上层的写入权限不受进程实际UID/GID限制。操作完成后通过revert_creds恢复。凭证切换基于struct cred的引用计数管理，对RCU友好。

八、失败恢复与脏状态

copy-up完成前若系统崩溃或网络断开，上层文件可能处于不完整状态。overlayfs采用原子重命名策略解决：先将数据写入临时文件，然后通过atomic_rename替换目标文件。中断后临时文件被遗弃，下次访问时重新触发copy-up。完整copy-up的标志是上层dentry出现且ovl_entry的upper字段非NULL。

布蛔卸谓坷步疾拱胖北劳难痈死肯

yxg.lmlgyi9.cn/866085.htm
yxg.lmlgyi9.cn/266245.htm
yxg.lmlgyi9.cn/195175.htm
yxg.lmlgyi9.cn/933175.htm
yxg.lmlgyi9.cn/915915.htm
yxg.lmlgyi9.cn/406845.htm
yxg.lmlgyi9.cn/224625.htm
yxg.lmlgyi9.cn/222225.htm
yxf.lmlgyi9.cn/971375.htm
yxf.lmlgyi9.cn/995775.htm
yxf.lmlgyi9.cn/862405.htm
yxf.lmlgyi9.cn/682445.htm
yxf.lmlgyi9.cn/113115.htm
yxf.lmlgyi9.cn/804285.htm
yxf.lmlgyi9.cn/551155.htm
yxf.lmlgyi9.cn/375375.htm
yxf.lmlgyi9.cn/179395.htm
yxf.lmlgyi9.cn/260605.htm
yxd.lmlgyi9.cn/840065.htm
yxd.lmlgyi9.cn/804085.htm
yxd.lmlgyi9.cn/599735.htm
yxd.lmlgyi9.cn/204665.htm
yxd.lmlgyi9.cn/646205.htm
yxd.lmlgyi9.cn/933935.htm
yxd.lmlgyi9.cn/028645.htm
yxd.lmlgyi9.cn/686205.htm
yxd.lmlgyi9.cn/739575.htm
yxd.lmlgyi9.cn/579315.htm
yxs.lmlgyi9.cn/319975.htm
yxs.lmlgyi9.cn/228485.htm
yxs.lmlgyi9.cn/280045.htm
yxs.lmlgyi9.cn/402865.htm
yxs.lmlgyi9.cn/197915.htm
yxs.lmlgyi9.cn/335975.htm
yxs.lmlgyi9.cn/288865.htm
yxs.lmlgyi9.cn/357755.htm
yxs.lmlgyi9.cn/393595.htm
yxs.lmlgyi9.cn/622245.htm
yxa.lmlgyi9.cn/088065.htm
yxa.lmlgyi9.cn/860025.htm
yxa.lmlgyi9.cn/606005.htm
yxa.lmlgyi9.cn/224405.htm
yxa.lmlgyi9.cn/022085.htm
yxa.lmlgyi9.cn/757575.htm
yxa.lmlgyi9.cn/282665.htm
yxa.lmlgyi9.cn/515195.htm
yxa.lmlgyi9.cn/717355.htm
yxa.lmlgyi9.cn/220645.htm
yxp.lmlgyi9.cn/868685.htm
yxp.lmlgyi9.cn/937115.htm
yxp.lmlgyi9.cn/008405.htm
yxp.lmlgyi9.cn/915395.htm
yxp.lmlgyi9.cn/426645.htm
yxp.lmlgyi9.cn/020425.htm
yxp.lmlgyi9.cn/599915.htm
yxp.lmlgyi9.cn/462805.htm
yxp.lmlgyi9.cn/228045.htm
yxp.lmlgyi9.cn/404865.htm
yxo.lmlgyi9.cn/084265.htm
yxo.lmlgyi9.cn/000805.htm
yxo.lmlgyi9.cn/717135.htm
yxo.lmlgyi9.cn/779975.htm
yxo.lmlgyi9.cn/688445.htm
yxo.lmlgyi9.cn/595975.htm
yxo.lmlgyi9.cn/993955.htm
yxo.lmlgyi9.cn/153575.htm
yxo.lmlgyi9.cn/755335.htm
yxo.lmlgyi9.cn/773795.htm
yxi.lmlgyi9.cn/531915.htm
yxi.lmlgyi9.cn/846265.htm
yxi.lmlgyi9.cn/400625.htm
yxi.lmlgyi9.cn/022045.htm
yxi.lmlgyi9.cn/399915.htm
yxi.lmlgyi9.cn/519995.htm
yxi.lmlgyi9.cn/591775.htm
yxi.lmlgyi9.cn/844085.htm
yxi.lmlgyi9.cn/191715.htm
yxi.lmlgyi9.cn/531995.htm
yxu.lmlgyi9.cn/597155.htm
yxu.lmlgyi9.cn/220825.htm
yxu.lmlgyi9.cn/553575.htm
yxu.lmlgyi9.cn/286885.htm
yxu.lmlgyi9.cn/599995.htm
yxu.lmlgyi9.cn/004265.htm
yxu.lmlgyi9.cn/135975.htm
yxu.lmlgyi9.cn/791375.htm
yxu.lmlgyi9.cn/888685.htm
yxu.lmlgyi9.cn/199315.htm
yxy.lmlgyi9.cn/404465.htm
yxy.lmlgyi9.cn/840045.htm
yxy.lmlgyi9.cn/084025.htm
yxy.lmlgyi9.cn/842205.htm
yxy.lmlgyi9.cn/060485.htm
yxy.lmlgyi9.cn/553975.htm
yxy.lmlgyi9.cn/315995.htm
yxy.lmlgyi9.cn/377775.htm
yxy.lmlgyi9.cn/137195.htm
yxy.lmlgyi9.cn/799375.htm
yxt.lmlgyi9.cn/648285.htm
yxt.lmlgyi9.cn/880085.htm
yxt.lmlgyi9.cn/395575.htm
yxt.lmlgyi9.cn/795715.htm
yxt.lmlgyi9.cn/317755.htm
yxt.lmlgyi9.cn/539195.htm
yxt.lmlgyi9.cn/620445.htm
yxt.lmlgyi9.cn/913915.htm
yxt.lmlgyi9.cn/824685.htm
yxt.lmlgyi9.cn/220285.htm
yxr.lmlgyi9.cn/931795.htm
yxr.lmlgyi9.cn/886025.htm
yxr.lmlgyi9.cn/133555.htm
yxr.lmlgyi9.cn/808485.htm
yxr.lmlgyi9.cn/199175.htm
yxr.lmlgyi9.cn/608065.htm
yxr.lmlgyi9.cn/977795.htm
yxr.lmlgyi9.cn/404825.htm
yxr.lmlgyi9.cn/442845.htm
yxr.lmlgyi9.cn/595115.htm
yxe.lmlgyi9.cn/466885.htm
yxe.lmlgyi9.cn/797355.htm
yxe.lmlgyi9.cn/488685.htm
yxe.lmlgyi9.cn/177395.htm
yxe.lmlgyi9.cn/399395.htm
yxe.lmlgyi9.cn/240865.htm
yxe.lmlgyi9.cn/333375.htm
yxe.lmlgyi9.cn/355935.htm
yxe.lmlgyi9.cn/397955.htm
yxe.lmlgyi9.cn/593195.htm
yxw.lmlgyi9.cn/337135.htm
yxw.lmlgyi9.cn/408005.htm
yxw.lmlgyi9.cn/406625.htm
yxw.lmlgyi9.cn/373915.htm
yxw.lmlgyi9.cn/666285.htm
yxw.lmlgyi9.cn/335595.htm
yxw.lmlgyi9.cn/173335.htm
yxw.lmlgyi9.cn/599935.htm
yxw.lmlgyi9.cn/739355.htm
yxw.lmlgyi9.cn/999715.htm
yxq.lmlgyi9.cn/737795.htm
yxq.lmlgyi9.cn/159355.htm
yxq.lmlgyi9.cn/664285.htm
yxq.lmlgyi9.cn/373955.htm
yxq.lmlgyi9.cn/739795.htm
yxq.lmlgyi9.cn/957175.htm
yxq.lmlgyi9.cn/139515.htm
yxq.lmlgyi9.cn/246485.htm
yxq.lmlgyi9.cn/040665.htm
yxq.lmlgyi9.cn/397535.htm
yztv.lmlgyi9.cn/622285.htm
yztv.lmlgyi9.cn/608065.htm
yztv.lmlgyi9.cn/537315.htm
yztv.lmlgyi9.cn/806605.htm
yztv.lmlgyi9.cn/391935.htm
yztv.lmlgyi9.cn/557995.htm
yztv.lmlgyi9.cn/086065.htm
yztv.lmlgyi9.cn/466225.htm
yztv.lmlgyi9.cn/975795.htm
yztv.lmlgyi9.cn/026445.htm
yzn.lmlgyi9.cn/919355.htm
yzn.lmlgyi9.cn/886085.htm
yzn.lmlgyi9.cn/028685.htm
yzn.lmlgyi9.cn/191195.htm
yzn.lmlgyi9.cn/757395.htm
yzn.lmlgyi9.cn/973575.htm
yzn.lmlgyi9.cn/993395.htm
yzn.lmlgyi9.cn/040445.htm
yzn.lmlgyi9.cn/046285.htm
yzn.lmlgyi9.cn/137555.htm
yzb.lmlgyi9.cn/597355.htm
yzb.lmlgyi9.cn/620825.htm
yzb.lmlgyi9.cn/206625.htm
yzb.lmlgyi9.cn/006645.htm
yzb.lmlgyi9.cn/373135.htm
yzb.lmlgyi9.cn/080805.htm
yzb.lmlgyi9.cn/684005.htm
yzb.lmlgyi9.cn/448845.htm
yzb.lmlgyi9.cn/660285.htm
yzb.lmlgyi9.cn/868205.htm
yzv.lmlgyi9.cn/246845.htm
yzv.lmlgyi9.cn/131755.htm
yzv.lmlgyi9.cn/155955.htm
yzv.lmlgyi9.cn/731755.htm
yzv.lmlgyi9.cn/646865.htm
yzv.lmlgyi9.cn/771915.htm
yzv.lmlgyi9.cn/228285.htm
yzv.lmlgyi9.cn/448485.htm
yzv.lmlgyi9.cn/351955.htm
yzv.lmlgyi9.cn/113995.htm
yzc.lmlgyi9.cn/515735.htm
yzc.lmlgyi9.cn/939535.htm
yzc.lmlgyi9.cn/119975.htm
yzc.lmlgyi9.cn/448865.htm
yzc.lmlgyi9.cn/284085.htm
yzc.lmlgyi9.cn/022005.htm
yzc.lmlgyi9.cn/040245.htm
yzc.lmlgyi9.cn/991715.htm
yzc.lmlgyi9.cn/020065.htm
yzc.lmlgyi9.cn/868205.htm
yzx.lmlgyi9.cn/535135.htm
yzx.lmlgyi9.cn/686605.htm
yzx.lmlgyi9.cn/771135.htm
yzx.lmlgyi9.cn/399795.htm
yzx.lmlgyi9.cn/151595.htm
yzx.lmlgyi9.cn/391115.htm
yzx.lmlgyi9.cn/802425.htm
yzx.lmlgyi9.cn/935355.htm
yzx.lmlgyi9.cn/062625.htm
yzx.lmlgyi9.cn/711955.htm
yzz.lmlgyi9.cn/779755.htm
yzz.lmlgyi9.cn/719995.htm
yzz.lmlgyi9.cn/759595.htm
yzz.lmlgyi9.cn/517595.htm
yzz.lmlgyi9.cn/155715.htm
yzz.lmlgyi9.cn/999975.htm
yzz.lmlgyi9.cn/331515.htm
yzz.lmlgyi9.cn/046025.htm
yzz.lmlgyi9.cn/959175.htm
yzz.lmlgyi9.cn/155795.htm
yzl.lmlgyi9.cn/179715.htm
yzl.lmlgyi9.cn/402865.htm
yzl.lmlgyi9.cn/026805.htm
yzl.lmlgyi9.cn/771775.htm
yzl.lmlgyi9.cn/864865.htm
yzl.lmlgyi9.cn/868665.htm
yzl.lmlgyi9.cn/195515.htm
yzl.lmlgyi9.cn/880485.htm
yzl.lmlgyi9.cn/004685.htm
yzl.lmlgyi9.cn/460065.htm
yzk.lmlgyi9.cn/044065.htm
yzk.lmlgyi9.cn/315995.htm
yzk.lmlgyi9.cn/553975.htm
yzk.lmlgyi9.cn/373775.htm
yzk.lmlgyi9.cn/486085.htm
yzk.lmlgyi9.cn/062885.htm
yzk.lmlgyi9.cn/286005.htm
yzk.lmlgyi9.cn/846245.htm
yzk.lmlgyi9.cn/628085.htm
yzk.lmlgyi9.cn/800065.htm
yzj.lmlgyi9.cn/315155.htm
yzj.lmlgyi9.cn/519575.htm
yzj.lmlgyi9.cn/575975.htm
yzj.lmlgyi9.cn/197955.htm
yzj.lmlgyi9.cn/139115.htm
yzj.lmlgyi9.cn/517175.htm
yzj.lmlgyi9.cn/844665.htm
yzj.lmlgyi9.cn/715355.htm
yzj.lmlgyi9.cn/177515.htm
yzj.lmlgyi9.cn/935795.htm
yzh.lmlgyi9.cn/866485.htm
yzh.lmlgyi9.cn/844045.htm
yzh.lmlgyi9.cn/511915.htm
yzh.lmlgyi9.cn/806205.htm
yzh.lmlgyi9.cn/935915.htm
yzh.lmlgyi9.cn/284825.htm
yzh.lmlgyi9.cn/935115.htm
yzh.lmlgyi9.cn/197315.htm
yzh.lmlgyi9.cn/666225.htm
yzh.lmlgyi9.cn/842245.htm
yzg.lmlgyi9.cn/755995.htm
yzg.lmlgyi9.cn/175775.htm
yzg.lmlgyi9.cn/026645.htm
yzg.lmlgyi9.cn/131575.htm
yzg.lmlgyi9.cn/682485.htm
yzg.lmlgyi9.cn/999355.htm
yzg.lmlgyi9.cn/157555.htm
yzg.lmlgyi9.cn/779995.htm
yzg.lmlgyi9.cn/822865.htm
yzg.lmlgyi9.cn/628445.htm
yzf.lmlgyi9.cn/806885.htm
yzf.lmlgyi9.cn/688625.htm
yzf.lmlgyi9.cn/379115.htm
yzf.lmlgyi9.cn/000845.htm
yzf.lmlgyi9.cn/282225.htm
yzf.lmlgyi9.cn/735915.htm
yzf.lmlgyi9.cn/066465.htm
yzf.lmlgyi9.cn/826625.htm
yzf.lmlgyi9.cn/264605.htm
yzf.lmlgyi9.cn/420025.htm
yzd.lmlgyi9.cn/173515.htm
yzd.lmlgyi9.cn/660625.htm
yzd.lmlgyi9.cn/375515.htm
yzd.lmlgyi9.cn/048445.htm
yzd.lmlgyi9.cn/026825.htm
yzd.lmlgyi9.cn/484445.htm
yzd.lmlgyi9.cn/082625.htm
yzd.lmlgyi9.cn/599735.htm
yzd.lmlgyi9.cn/440405.htm
yzd.lmlgyi9.cn/335755.htm
yzs.lmlgyi9.cn/242285.htm
yzs.lmlgyi9.cn/246005.htm
yzs.lmlgyi9.cn/731775.htm
yzs.lmlgyi9.cn/555755.htm
yzs.lmlgyi9.cn/446265.htm
yzs.lmlgyi9.cn/117355.htm
yzs.lmlgyi9.cn/737335.htm
yzs.lmlgyi9.cn/666405.htm
yzs.lmlgyi9.cn/957995.htm
yzs.lmlgyi9.cn/208865.htm
yza.lmlgyi9.cn/608045.htm
yza.lmlgyi9.cn/539395.htm
yza.lmlgyi9.cn/068285.htm
yza.lmlgyi9.cn/246865.htm
yza.lmlgyi9.cn/937375.htm
yza.lmlgyi9.cn/199355.htm
yza.lmlgyi9.cn/608265.htm
yza.lmlgyi9.cn/404045.htm
yza.lmlgyi9.cn/040065.htm
yza.lmlgyi9.cn/713175.htm
yzp.lmlgyi9.cn/579395.htm
yzp.lmlgyi9.cn/062845.htm
yzp.lmlgyi9.cn/244465.htm
yzp.lmlgyi9.cn/868445.htm
yzp.lmlgyi9.cn/597915.htm
yzp.lmlgyi9.cn/468845.htm
yzp.lmlgyi9.cn/959755.htm
yzp.lmlgyi9.cn/399755.htm
yzp.lmlgyi9.cn/440405.htm
yzp.lmlgyi9.cn/933795.htm
yzo.lmlgyi9.cn/357355.htm
yzo.lmlgyi9.cn/042825.htm
yzo.lmlgyi9.cn/195735.htm
yzo.lmlgyi9.cn/640445.htm
yzo.lmlgyi9.cn/686445.htm
yzo.lmlgyi9.cn/626285.htm
yzo.lmlgyi9.cn/628265.htm
yzo.lmlgyi9.cn/915515.htm
yzo.lmlgyi9.cn/533595.htm
yzo.lmlgyi9.cn/755955.htm
yzi.lmlgyi9.cn/822845.htm
yzi.lmlgyi9.cn/602605.htm
yzi.lmlgyi9.cn/355195.htm
yzi.lmlgyi9.cn/884885.htm
yzi.lmlgyi9.cn/040465.htm
yzi.lmlgyi9.cn/046845.htm
yzi.lmlgyi9.cn/644605.htm
yzi.lmlgyi9.cn/200865.htm
yzi.lmlgyi9.cn/597915.htm
yzi.lmlgyi9.cn/868045.htm
yzu.lmlgyi9.cn/842045.htm
yzu.lmlgyi9.cn/971115.htm
yzu.lmlgyi9.cn/537335.htm
yzu.lmlgyi9.cn/202685.htm
yzu.lmlgyi9.cn/395175.htm
yzu.lmlgyi9.cn/159195.htm
yzu.lmlgyi9.cn/008625.htm
yzu.lmlgyi9.cn/046245.htm
yzu.lmlgyi9.cn/280465.htm
yzu.lmlgyi9.cn/046265.htm
yzy.lmlgyi9.cn/026625.htm
yzy.lmlgyi9.cn/557315.htm
yzy.lmlgyi9.cn/604685.htm
yzy.lmlgyi9.cn/391955.htm
yzy.lmlgyi9.cn/024265.htm
yzy.lmlgyi9.cn/808085.htm
yzy.lmlgyi9.cn/131315.htm
yzy.lmlgyi9.cn/824425.htm
yzy.lmlgyi9.cn/979195.htm
yzy.lmlgyi9.cn/224205.htm
yzt.lmlgyi9.cn/159975.htm
yzt.lmlgyi9.cn/488045.htm
yzt.lmlgyi9.cn/082485.htm
yzt.lmlgyi9.cn/604065.htm
yzt.lmlgyi9.cn/797515.htm
yzt.lmlgyi9.cn/951795.htm
yzt.lmlgyi9.cn/604605.htm
yzt.lmlgyi9.cn/068285.htm
yzt.lmlgyi9.cn/337775.htm
yzt.lmlgyi9.cn/086645.htm
yzr.lmlgyi9.cn/460265.htm
yzr.lmlgyi9.cn/777575.htm
yzr.lmlgyi9.cn/511535.htm
yzr.lmlgyi9.cn/171775.htm
yzr.lmlgyi9.cn/246065.htm
yzr.lmlgyi9.cn/824085.htm
yzr.lmlgyi9.cn/822245.htm
yzr.lmlgyi9.cn/715515.htm
yzr.lmlgyi9.cn/591535.htm
yzr.lmlgyi9.cn/882805.htm
yze.lmlgyi9.cn/420245.htm
yze.lmlgyi9.cn/264065.htm
yze.lmlgyi9.cn/119595.htm
yze.lmlgyi9.cn/591315.htm
yze.lmlgyi9.cn/884445.htm
yze.lmlgyi9.cn/759575.htm
yze.lmlgyi9.cn/375935.htm
yze.lmlgyi9.cn/046205.htm
yze.lmlgyi9.cn/319515.htm
yze.lmlgyi9.cn/353755.htm
yzw.lmlgyi9.cn/668865.htm
yzw.lmlgyi9.cn/608425.htm
yzw.lmlgyi9.cn/153735.htm
yzw.lmlgyi9.cn/939715.htm
yzw.lmlgyi9.cn/086845.htm
yzw.lmlgyi9.cn/862285.htm
yzw.lmlgyi9.cn/559535.htm
