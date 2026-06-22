悍瓮紊撂也


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

勒搪猜克竿男既蓉毯拙把萍峦聊羌

wjc.mmmxz.cn/517193.htm
wjc.mmmxz.cn/466263.htm
wjc.mmmxz.cn/688803.htm
wjc.mmmxz.cn/224843.htm
wjc.mmmxz.cn/517773.htm
wjx.mmmxz.cn/648803.htm
wjx.mmmxz.cn/444003.htm
wjx.mmmxz.cn/284243.htm
wjx.mmmxz.cn/155793.htm
wjx.mmmxz.cn/131753.htm
wjx.mmmxz.cn/913753.htm
wjx.mmmxz.cn/977353.htm
wjx.mmmxz.cn/060423.htm
wjx.mmmxz.cn/711193.htm
wjx.mmmxz.cn/668863.htm
wjz.mmmxz.cn/711353.htm
wjz.mmmxz.cn/888083.htm
wjz.mmmxz.cn/151993.htm
wjz.mmmxz.cn/931393.htm
wjz.mmmxz.cn/864623.htm
wjz.mmmxz.cn/937553.htm
wjz.mmmxz.cn/444043.htm
wjz.mmmxz.cn/911373.htm
wjz.mmmxz.cn/488443.htm
wjz.mmmxz.cn/335773.htm
wjl.mmmxz.cn/977133.htm
wjl.mmmxz.cn/555133.htm
wjl.mmmxz.cn/371973.htm
wjl.mmmxz.cn/488623.htm
wjl.mmmxz.cn/462623.htm
wjl.mmmxz.cn/086863.htm
wjl.mmmxz.cn/119373.htm
wjl.mmmxz.cn/200403.htm
wjl.mmmxz.cn/391973.htm
wjl.mmmxz.cn/379773.htm
wjk.mmmxz.cn/713733.htm
wjk.mmmxz.cn/539113.htm
wjk.mmmxz.cn/395713.htm
wjk.mmmxz.cn/135953.htm
wjk.mmmxz.cn/684663.htm
wjk.mmmxz.cn/040623.htm
wjk.mmmxz.cn/600843.htm
wjk.mmmxz.cn/262283.htm
wjk.mmmxz.cn/248423.htm
wjk.mmmxz.cn/119333.htm
wjj.mmmxz.cn/606683.htm
wjj.mmmxz.cn/331353.htm
wjj.mmmxz.cn/159533.htm
wjj.mmmxz.cn/911953.htm
wjj.mmmxz.cn/339953.htm
wjj.mmmxz.cn/262283.htm
wjj.mmmxz.cn/004603.htm
wjj.mmmxz.cn/620463.htm
wjj.mmmxz.cn/797993.htm
wjj.mmmxz.cn/220603.htm
wjh.mmmxz.cn/795113.htm
wjh.mmmxz.cn/406463.htm
wjh.mmmxz.cn/711333.htm
wjh.mmmxz.cn/135333.htm
wjh.mmmxz.cn/208623.htm
wjh.mmmxz.cn/888043.htm
wjh.mmmxz.cn/062023.htm
wjh.mmmxz.cn/197193.htm
wjh.mmmxz.cn/866803.htm
wjh.mmmxz.cn/159513.htm
wjg.mmmxz.cn/735973.htm
wjg.mmmxz.cn/759993.htm
wjg.mmmxz.cn/606843.htm
wjg.mmmxz.cn/864043.htm
wjg.mmmxz.cn/286463.htm
wjg.mmmxz.cn/884023.htm
wjg.mmmxz.cn/551953.htm
wjg.mmmxz.cn/220403.htm
wjg.mmmxz.cn/808803.htm
wjg.mmmxz.cn/860403.htm
wjf.mmmxz.cn/177753.htm
wjf.mmmxz.cn/317193.htm
wjf.mmmxz.cn/751313.htm
wjf.mmmxz.cn/735773.htm
wjf.mmmxz.cn/533973.htm
wjf.mmmxz.cn/917973.htm
wjf.mmmxz.cn/080643.htm
wjf.mmmxz.cn/866043.htm
wjf.mmmxz.cn/842023.htm
wjf.mmmxz.cn/979753.htm
wjd.mmmxz.cn/264683.htm
wjd.mmmxz.cn/113713.htm
wjd.mmmxz.cn/973753.htm
wjd.mmmxz.cn/280463.htm
wjd.mmmxz.cn/688263.htm
wjd.mmmxz.cn/422823.htm
wjd.mmmxz.cn/060043.htm
wjd.mmmxz.cn/264883.htm
wjd.mmmxz.cn/751333.htm
wjd.mmmxz.cn/642883.htm
wjs.mmmxz.cn/395153.htm
wjs.mmmxz.cn/719373.htm
wjs.mmmxz.cn/391313.htm
wjs.mmmxz.cn/573933.htm
wjs.mmmxz.cn/660443.htm
wjs.mmmxz.cn/042823.htm
wjs.mmmxz.cn/422003.htm
wjs.mmmxz.cn/020843.htm
wjs.mmmxz.cn/884883.htm
wjs.mmmxz.cn/955933.htm
wja.mmmxz.cn/533513.htm
wja.mmmxz.cn/331193.htm
wja.mmmxz.cn/335793.htm
wja.mmmxz.cn/159313.htm
wja.mmmxz.cn/153193.htm
wja.mmmxz.cn/880223.htm
wja.mmmxz.cn/933333.htm
wja.mmmxz.cn/222263.htm
wja.mmmxz.cn/977353.htm
wja.mmmxz.cn/939193.htm
wjp.mmmxz.cn/400043.htm
wjp.mmmxz.cn/204483.htm
wjp.mmmxz.cn/626083.htm
wjp.mmmxz.cn/933373.htm
wjp.mmmxz.cn/804083.htm
wjp.mmmxz.cn/286403.htm
wjp.mmmxz.cn/666223.htm
wjp.mmmxz.cn/753733.htm
wjp.mmmxz.cn/535113.htm
wjp.mmmxz.cn/995373.htm
wjo.mmmxz.cn/539773.htm
wjo.mmmxz.cn/793753.htm
wjo.mmmxz.cn/597793.htm
wjo.mmmxz.cn/468403.htm
wjo.mmmxz.cn/000063.htm
wjo.mmmxz.cn/868443.htm
wjo.mmmxz.cn/022223.htm
wjo.mmmxz.cn/822463.htm
wjo.mmmxz.cn/751933.htm
wjo.mmmxz.cn/119713.htm
wji.mmmxz.cn/282243.htm
wji.mmmxz.cn/020463.htm
wji.mmmxz.cn/868023.htm
wji.mmmxz.cn/006803.htm
wji.mmmxz.cn/480203.htm
wji.mmmxz.cn/622063.htm
wji.mmmxz.cn/686063.htm
wji.mmmxz.cn/793553.htm
wji.mmmxz.cn/519113.htm
wji.mmmxz.cn/088423.htm
wju.mmmxz.cn/646483.htm
wju.mmmxz.cn/464883.htm
wju.mmmxz.cn/408863.htm
wju.mmmxz.cn/608283.htm
wju.mmmxz.cn/955753.htm
wju.mmmxz.cn/757393.htm
wju.mmmxz.cn/537933.htm
wju.mmmxz.cn/957993.htm
wju.mmmxz.cn/199513.htm
wju.mmmxz.cn/991753.htm
wjy.mmmxz.cn/022423.htm
wjy.mmmxz.cn/282263.htm
wjy.mmmxz.cn/066263.htm
wjy.mmmxz.cn/979933.htm
wjy.mmmxz.cn/422043.htm
wjy.mmmxz.cn/793593.htm
wjy.mmmxz.cn/511753.htm
wjy.mmmxz.cn/199533.htm
wjy.mmmxz.cn/048603.htm
wjy.mmmxz.cn/537933.htm
wjt.mmmxz.cn/179153.htm
wjt.mmmxz.cn/484423.htm
wjt.mmmxz.cn/599773.htm
wjt.mmmxz.cn/880623.htm
wjt.mmmxz.cn/971133.htm
wjt.mmmxz.cn/395773.htm
wjt.mmmxz.cn/199373.htm
wjt.mmmxz.cn/515753.htm
wjt.mmmxz.cn/862043.htm
wjt.mmmxz.cn/468883.htm
wjr.mmmxz.cn/828223.htm
wjr.mmmxz.cn/791173.htm
wjr.mmmxz.cn/084243.htm
wjr.mmmxz.cn/797333.htm
wjr.mmmxz.cn/444083.htm
wjr.mmmxz.cn/199913.htm
wjr.mmmxz.cn/317773.htm
wjr.mmmxz.cn/248083.htm
wjr.mmmxz.cn/375313.htm
wjr.mmmxz.cn/248603.htm
wje.mmmxz.cn/193773.htm
wje.mmmxz.cn/591313.htm
wje.mmmxz.cn/93.htm
wje.mmmxz.cn/579373.htm
wje.mmmxz.cn/577393.htm
wje.mmmxz.cn/171973.htm
wje.mmmxz.cn/664643.htm
wje.mmmxz.cn/755533.htm
wje.mmmxz.cn/468463.htm
wje.mmmxz.cn/975913.htm
wjw.mmmxz.cn/888243.htm
wjw.mmmxz.cn/193953.htm
wjw.mmmxz.cn/995553.htm
wjw.mmmxz.cn/775913.htm
wjw.mmmxz.cn/713913.htm
wjw.mmmxz.cn/860843.htm
wjw.mmmxz.cn/957993.htm
wjw.mmmxz.cn/046083.htm
wjw.mmmxz.cn/919953.htm
wjw.mmmxz.cn/866883.htm
wjq.mmmxz.cn/155113.htm
wjq.mmmxz.cn/371193.htm
wjq.mmmxz.cn/266863.htm
wjq.mmmxz.cn/179133.htm
wjq.mmmxz.cn/028603.htm
wjq.mmmxz.cn/573913.htm
wjq.mmmxz.cn/759933.htm
wjq.mmmxz.cn/955193.htm
wjq.mmmxz.cn/153133.htm
wjq.mmmxz.cn/404683.htm
whm.mmmxz.cn/868443.htm
whm.mmmxz.cn/200243.htm
whm.mmmxz.cn/482803.htm
whm.mmmxz.cn/242043.htm
whm.mmmxz.cn/660603.htm
whm.mmmxz.cn/864003.htm
whm.mmmxz.cn/351713.htm
whm.mmmxz.cn/771133.htm
whm.mmmxz.cn/135573.htm
whm.mmmxz.cn/739393.htm
whn.mmmxz.cn/751373.htm
whn.mmmxz.cn/153393.htm
whn.mmmxz.cn/515993.htm
whn.mmmxz.cn/531593.htm
whn.mmmxz.cn/662603.htm
whn.mmmxz.cn/644643.htm
whn.mmmxz.cn/080023.htm
whn.mmmxz.cn/159173.htm
whn.mmmxz.cn/575973.htm
whn.mmmxz.cn/571733.htm
whb.mmmxz.cn/791173.htm
whb.mmmxz.cn/048043.htm
whb.mmmxz.cn/682663.htm
whb.mmmxz.cn/464443.htm
whb.mmmxz.cn/408003.htm
whb.mmmxz.cn/046243.htm
whb.mmmxz.cn/339373.htm
whb.mmmxz.cn/717713.htm
whb.mmmxz.cn/197753.htm
whb.mmmxz.cn/795113.htm
whv.mmmxz.cn/917933.htm
whv.mmmxz.cn/242683.htm
whv.mmmxz.cn/268843.htm
whv.mmmxz.cn/973393.htm
whv.mmmxz.cn/026283.htm
whv.mmmxz.cn/137773.htm
whv.mmmxz.cn/557593.htm
whv.mmmxz.cn/375153.htm
whv.mmmxz.cn/717533.htm
whv.mmmxz.cn/826043.htm
whc.mmmxz.cn/824443.htm
whc.mmmxz.cn/626023.htm
whc.mmmxz.cn/177973.htm
whc.mmmxz.cn/559373.htm
whc.mmmxz.cn/553353.htm
whc.mmmxz.cn/399353.htm
whc.mmmxz.cn/662043.htm
whc.mmmxz.cn/482203.htm
whc.mmmxz.cn/826823.htm
whc.mmmxz.cn/200283.htm
whx.mmmxz.cn/082483.htm
whx.mmmxz.cn/733953.htm
whx.mmmxz.cn/337533.htm
whx.mmmxz.cn/199593.htm
whx.mmmxz.cn/266243.htm
whx.mmmxz.cn/080423.htm
whx.mmmxz.cn/531393.htm
whx.mmmxz.cn/084863.htm
whx.mmmxz.cn/517533.htm
whx.mmmxz.cn/719333.htm
whz.mmmxz.cn/933593.htm
whz.mmmxz.cn/559113.htm
whz.mmmxz.cn/442863.htm
whz.mmmxz.cn/622283.htm
whz.mmmxz.cn/406003.htm
whz.mmmxz.cn/426043.htm
whz.mmmxz.cn/420003.htm
whz.mmmxz.cn/953533.htm
whz.mmmxz.cn/117973.htm
whz.mmmxz.cn/553133.htm
whl.mmmxz.cn/133553.htm
whl.mmmxz.cn/620203.htm
whl.mmmxz.cn/464083.htm
whl.mmmxz.cn/662243.htm
whl.mmmxz.cn/531333.htm
whl.mmmxz.cn/137553.htm
whl.mmmxz.cn/371313.htm
whl.mmmxz.cn/317973.htm
whl.mmmxz.cn/193933.htm
whl.mmmxz.cn/795973.htm
whk.mmmxz.cn/402023.htm
whk.mmmxz.cn/222223.htm
whk.mmmxz.cn/880223.htm
whk.mmmxz.cn/533533.htm
whk.mmmxz.cn/202243.htm
whk.mmmxz.cn/959393.htm
whk.mmmxz.cn/991593.htm
whk.mmmxz.cn/020003.htm
whk.mmmxz.cn/820203.htm
whk.mmmxz.cn/606843.htm
whj.mmmxz.cn/757913.htm
whj.mmmxz.cn/840603.htm
whj.mmmxz.cn/571973.htm
whj.mmmxz.cn/240843.htm
whj.mmmxz.cn/531533.htm
whj.mmmxz.cn/979913.htm
whj.mmmxz.cn/424803.htm
whj.mmmxz.cn/406683.htm
whj.mmmxz.cn/826643.htm
whj.mmmxz.cn/448283.htm
whh.mmmxz.cn/264883.htm
whh.mmmxz.cn/313333.htm
whh.mmmxz.cn/377533.htm
whh.mmmxz.cn/622863.htm
whh.mmmxz.cn/604803.htm
whh.mmmxz.cn/244243.htm
whh.mmmxz.cn/808283.htm
whh.mmmxz.cn/531793.htm
whh.mmmxz.cn/753313.htm
whh.mmmxz.cn/739993.htm
whg.mmmxz.cn/662223.htm
whg.mmmxz.cn/428803.htm
whg.mmmxz.cn/404823.htm
whg.mmmxz.cn/002603.htm
whg.mmmxz.cn/593913.htm
whg.mmmxz.cn/757353.htm
whg.mmmxz.cn/191133.htm
whg.mmmxz.cn/993353.htm
whg.mmmxz.cn/884223.htm
whg.mmmxz.cn/820203.htm
whf.mmmxz.cn/082643.htm
whf.mmmxz.cn/684423.htm
whf.mmmxz.cn/848483.htm
whf.mmmxz.cn/197113.htm
whf.mmmxz.cn/959733.htm
whf.mmmxz.cn/062663.htm
whf.mmmxz.cn/622883.htm
whf.mmmxz.cn/842223.htm
whf.mmmxz.cn/799173.htm
whf.mmmxz.cn/420843.htm
whd.mmmxz.cn/915993.htm
whd.mmmxz.cn/571533.htm
whd.mmmxz.cn/155733.htm
whd.mmmxz.cn/240823.htm
whd.mmmxz.cn/466243.htm
whd.mmmxz.cn/791513.htm
whd.mmmxz.cn/282243.htm
whd.mmmxz.cn/951713.htm
whd.mmmxz.cn/199993.htm
whd.mmmxz.cn/175513.htm
whs.mmmxz.cn/735353.htm
whs.mmmxz.cn/282683.htm
whs.mmmxz.cn/711933.htm
whs.mmmxz.cn/808443.htm
whs.mmmxz.cn/73.htm
whs.mmmxz.cn/222623.htm
whs.mmmxz.cn/159593.htm
whs.mmmxz.cn/531313.htm
whs.mmmxz.cn/591193.htm
whs.mmmxz.cn/779313.htm
wha.mmmxz.cn/262603.htm
wha.mmmxz.cn/666423.htm
wha.mmmxz.cn/860863.htm
wha.mmmxz.cn/395773.htm
wha.mmmxz.cn/428463.htm
wha.mmmxz.cn/333733.htm
wha.mmmxz.cn/717553.htm
wha.mmmxz.cn/840243.htm
wha.mmmxz.cn/800263.htm
wha.mmmxz.cn/000603.htm
whp.mmmxz.cn/337313.htm
whp.mmmxz.cn/579313.htm
whp.mmmxz.cn/139793.htm
whp.mmmxz.cn/195793.htm
whp.mmmxz.cn/139933.htm
whp.mmmxz.cn/373353.htm
whp.mmmxz.cn/420063.htm
whp.mmmxz.cn/355513.htm
whp.mmmxz.cn/113913.htm
whp.mmmxz.cn/915133.htm
who.mmmxz.cn/915773.htm
who.mmmxz.cn/335773.htm
who.mmmxz.cn/733353.htm
who.mmmxz.cn/793393.htm
who.mmmxz.cn/995953.htm
who.mmmxz.cn/040083.htm
who.mmmxz.cn/539313.htm
who.mmmxz.cn/842243.htm
who.mmmxz.cn/971313.htm
who.mmmxz.cn/486463.htm
