瞬琅练厦不


Linux devtmpfs devtmpfsd守护进程与device_create

devtmpfs是一个特殊的虚拟文件系统，在系统启动时自动创建和管理设备节点。与静态/dev目录或devfs不同，devtmpfs在内核中直接响应设备模型的添加和移除事件，在tmpfs上实时创建或删除设备节点。devtmpfsd是内核线程形式的守护进程，处理设备事件的工作队列，device_create是设备驱动创建设备节点的标准接口。

一、devtmpfs的初始化与挂载

devtmpfs在系统启动早期通过devtmpfs_init初始化，在rootfs挂载后挂载到/dev：

int __init devtmpfs_init(void)
{
    int err;
    struct vfsmount *mnt;

    if (!devtmpfs_mount)
        return 0;

    err = kern_mount(&devtmpfs_fs_type);
    if (err)
        goto out;

    mnt = kern_mount_data(&devtmpfs_fs_type, NULL);
    if (IS_ERR(mnt)) {
        err = PTR_ERR(mnt);
        goto out;
    }

    devtmpfs_mnt = mnt;
    spin_lock_init(&req_lock);
    INIT_LIST_HEAD(&requests);

    devtmpfsd_task = kthread_run(devtmpfsd, NULL, "devtmpfsd");
    if (IS_ERR(devtmpfsd_task)) {
        err = PTR_ERR(devtmpfsd_task);
        goto out;
    }

    return 0;
out:
    return err;
}

devtmpfs_fs_type基于tmpfs但不暴露给用户mount命令。kern_mount在内核命名空间中挂载，仅内核可见。kthread_run启动devtmpfsd守护线程处理请求队列。

二、devtmpfsd的主循环

devtmpfsd是核心的事件处理循环，等待设备创建/删除请求并执行：

static int devtmpfsd(void *p)
{
    struct req *req;
    int err = 0;

    complete(&setup_done);

    while (1) {
        spin_lock(&req_lock);
        while (list_empty(&requests)) {
            spin_unlock(&req_lock);
            if (kthread_should_stop())
                goto out;
            schedule();
            spin_lock(&req_lock);
        }
        req = list_first_entry(&requests, struct req, list);
        list_del(&req->list);
        spin_unlock(&req_lock);

        mutex_lock(&devtmpfs_mutex);
        if (req->mode == DEV_CREATE) {
            err = devtmpfs_create_node(req);
        } else if (req->mode == DEV_DELETE) {
            err = devtmpfs_delete_node(req);
        }
        req->err = err;
        complete(&req->done);
        mutex_unlock(&devtmpfs_mutex);
    }

out:
    return 0;
}

守护进程的工作模式是生产者-消费者模型：

1. 设备核心(device core)调用device_create或device_destroy时，将请求加入requests链表。
2. devtmpfsd在requests链表非空时被调度唤醒，取出请求执行。
3. 每个请求的完成通过completion变量同步，使调用者可等待节点创建完成。

三、devtmpfs_create_node的设备节点创建

devtmpfs_create_node在devtmpfs上创建设备节点文件：

static int devtmpfs_create_node(struct req *req)
{
    struct device *dev = req->dev;
    struct path path;
    struct dentry *dentry;
    const char *devpath = dev_name(dev);
    int err;
    umode_t mode = 0600;
    struct iattr attr;

    if (!devpath)
        return -EINVAL;

    err = kern_path_create(AT_FDCWD, devpath, &path, 0);
    if (err)
        return err;

    if (dev->devt) {
        mode = S_IFBLK | 0600;
        if (dev->class && dev->class->devnode)
            mode = dev->class->devnode(dev, &mode);
        else
            mode = S_IFCHR | 0600;
    }

    if (dev->devt) {
        err = vfs_mknod(d_inode(path.dentry), path.dentry,
                mode, dev->devt);
    } else {
        err = vfs_mkdir(d_inode(path.dentry), path.dentry, mode);
    }

    if (!err) {
        attr.ia_valid = ATTR_UID | ATTR_GID;
        attr.ia_uid = GLOBAL_ROOT_UID;
        attr.ia_gid = GLOBAL_ROOT_GID;

        inode_lock(d_inode(path.dentry));
        notify_change(path.dentry, &attr, NULL);
        inode_unlock(d_inode(path.dentry));
    }

    done_path_create(&path, dentry);
    return err;
}

该函数解析设备路径(如"bus/usb/devices/001")，在devtmpfs挂载点下创建对应的文件系统节点。对块设备和字符设备使用vfs_mknod创建设备节点，对总线等容器使用vfs_mkdir创建目录。节点权限通过class的devnode回调动态确定。

四、device_create的调用链

device_create是驱动注册设备的标准接口，最终触发devtmpfs节点创建：

struct device *device_create(struct class *class, struct device *parent,
                 dev_t devt, void *drvdata,
                 const char *fmt, ...)
{
    va_list vargs;
    struct device *dev;
    int ret;

    va_start(vargs, fmt);
    dev = device_create_vargs(class, parent, devt, drvdata, fmt, vargs);
    va_end(vargs);

    return dev;
}

struct device *device_create_vargs(struct class *class, struct device *parent,
                   dev_t devt, void *drvdata,
                   const char *fmt, va_list args)
{
    struct device *dev;
    int ret;

    dev = kzalloc(sizeof(*dev), GFP_KERNEL);
    if (!dev)
        return ERR_PTR(-ENOMEM);

    dev->devt = devt;
    dev->class = class;
    dev->parent = parent;
    dev->release = device_create_release;
    dev_set_drvdata(dev, drvdata);

    ret = dev_set_name(dev, fmt, args);
    if (ret) {
        kfree(dev);
        return ERR_PTR(ret);
    }

    ret = device_register(dev);
    if (ret) {
        put_device(dev);
        return ERR_PTR(ret);
    }

    return dev;
}

device_create_vargs分配struct device，设置设备号、类、父设备，调用dev_set_name根据printf格式设置设备名称，最后通过device_register将设备注册到驱动核心。

五、device_register到devtmpfs的事件传递

device_register调用device_add，后者通过devtmpfs_create_node提交请求：

int device_add(struct device *dev)
{
    struct device *parent = NULL;
    struct kobject *kobj;
    int error;

    error = device_create_sysfs_entry(dev);
    if (error)
        goto Done;

    error = device_add_class_symlinks(dev);
    if (error)
        goto Done;

    error = device_add_attrs(dev);
    if (error)
        goto Done;

    if (MAJOR(dev->devt)) {
        error = devtmpfs_create_node(dev);
        if (error)
            goto Done;
    }

    kobject_uevent(&dev->kobj, KOBJ_ADD);
    return 0;

Done:
    device_del(dev);
    return error;
}

device_add在完成sysfs属性创建、类链接建立后，若设备有主设备号(MAJOR非零)，调用devtmpfs_create_node创建设备节点。然后发送uevent通知用户空间udev。

六、devtmpfs_delete_node的设备移除

设备移除时，device_del调用devtmpfs_delete_node删除设备节点：

static int devtmpfs_delete_node(struct req *req)
{
    struct device *dev = req->dev;
    struct path path;
    int err;

    err = kern_path(dev_name(dev), LOOKUP_PARENT, &path);
    if (err)
        return err;

    err = vfs_unlink(d_inode(path.dentry), path.dentry, NULL);
    if (err == -EISDIR)
        err = vfs_rmdir(d_inode(path.dentry), path.dentry);

    path_put(&path);
    return err;
}

若节点是设备文件使用vfs_unlink，是目录使用vfs_rmdir。删除节点后，devtmpfsd通过complete通知调用者，设备移除流程继续执行。

七、并发处理与同步

多个设备同时注册时可能产生并发创建设备节点的请求。devtmpfsd使用mutex_lock(&devtmpfs_mutex)串行化节点创建和删除操作，防止同一路径的并发创建冲突。请求队列通过自旋锁(req_lock)保护，提供轻量级的入队/出队操作。

completion变量提供同步语义：每个请求包含struct completion done，device_add提交请求后调用wait_for_completion等待devtmpfsd处理完成，确保device_add返回时设备节点已在/dev下可见。

八、用户空间配合与udev

devtmpfs仅创建静态设备节点，不处理固件加载、模块加载等复杂事件。device_add发送KOBJ_ADD uevent后，用户空间udev接收事件并执行自定义规则。devtmpfs保证设备节点在其驱动probe完成前就已经存在，消除传统devfs中节点出现时间不确定的问题。

降嘿咳既巫降椅构捌较家抠谓棠接

frc.eiyve.cn/807881.Doc
frx.eiyve.cn/000604.Doc
frx.eiyve.cn/044288.Doc
frx.eiyve.cn/755159.Doc
frx.eiyve.cn/777553.Doc
frx.eiyve.cn/170211.Doc
frx.eiyve.cn/002028.Doc
frx.eiyve.cn/872232.Doc
frx.eiyve.cn/620866.Doc
frx.eiyve.cn/222060.Doc
frx.eiyve.cn/808644.Doc
frz.eiyve.cn/853717.Doc
frz.eiyve.cn/800262.Doc
frz.eiyve.cn/094341.Doc
frz.eiyve.cn/941129.Doc
frz.eiyve.cn/688426.Doc
frz.eiyve.cn/440626.Doc
frz.eiyve.cn/860000.Doc
frz.eiyve.cn/288824.Doc
frz.eiyve.cn/680226.Doc
frz.eiyve.cn/971339.Doc
frl.eiyve.cn/480486.Doc
frl.eiyve.cn/084664.Doc
frl.eiyve.cn/644082.Doc
frl.eiyve.cn/208422.Doc
frl.eiyve.cn/822048.Doc
frl.eiyve.cn/026806.Doc
frl.eiyve.cn/757157.Doc
frl.eiyve.cn/862808.Doc
frl.eiyve.cn/668648.Doc
frl.eiyve.cn/226084.Doc
frk.eiyve.cn/040266.Doc
frk.eiyve.cn/800226.Doc
frk.eiyve.cn/482402.Doc
frk.eiyve.cn/797355.Doc
frk.eiyve.cn/860488.Doc
frk.eiyve.cn/002640.Doc
frk.eiyve.cn/622808.Doc
frk.eiyve.cn/828040.Doc
frk.eiyve.cn/482486.Doc
frk.eiyve.cn/884440.Doc
frj.eiyve.cn/884066.Doc
frj.eiyve.cn/462200.Doc
frj.eiyve.cn/802446.Doc
frj.eiyve.cn/086680.Doc
frj.eiyve.cn/900862.Doc
frj.eiyve.cn/557191.Doc
frj.eiyve.cn/468422.Doc
frj.eiyve.cn/357753.Doc
frj.eiyve.cn/682080.Doc
frj.eiyve.cn/028044.Doc
frh.eiyve.cn/737713.Doc
frh.eiyve.cn/828024.Doc
frh.eiyve.cn/888606.Doc
frh.eiyve.cn/044808.Doc
frh.eiyve.cn/086880.Doc
frh.eiyve.cn/313549.Doc
frh.eiyve.cn/420844.Doc
frh.eiyve.cn/426428.Doc
frh.eiyve.cn/866000.Doc
frh.eiyve.cn/666482.Doc
frg.eiyve.cn/153715.Doc
frg.eiyve.cn/088220.Doc
frg.eiyve.cn/244044.Doc
frg.eiyve.cn/482888.Doc
frg.eiyve.cn/193199.Doc
frg.eiyve.cn/820862.Doc
frg.eiyve.cn/408046.Doc
frg.eiyve.cn/080682.Doc
frg.eiyve.cn/228880.Doc
frg.eiyve.cn/000466.Doc
frf.eiyve.cn/599133.Doc
frf.eiyve.cn/712224.Doc
frf.eiyve.cn/179919.Doc
frf.eiyve.cn/915579.Doc
frf.eiyve.cn/660440.Doc
frf.eiyve.cn/824208.Doc
frf.eiyve.cn/042826.Doc
frf.eiyve.cn/606206.Doc
frf.eiyve.cn/339537.Doc
frf.eiyve.cn/684864.Doc
frd.eiyve.cn/486608.Doc
frd.eiyve.cn/331795.Doc
frd.eiyve.cn/268442.Doc
frd.eiyve.cn/600260.Doc
frd.eiyve.cn/264044.Doc
frd.eiyve.cn/375353.Doc
frd.eiyve.cn/800286.Doc
frd.eiyve.cn/408442.Doc
frd.eiyve.cn/664806.Doc
frd.eiyve.cn/282406.Doc
frs.eiyve.cn/600068.Doc
frs.eiyve.cn/808840.Doc
frs.eiyve.cn/620606.Doc
frs.eiyve.cn/886640.Doc
frs.eiyve.cn/068228.Doc
frs.eiyve.cn/284224.Doc
frs.eiyve.cn/206600.Doc
frs.eiyve.cn/024224.Doc
frs.eiyve.cn/462648.Doc
frs.eiyve.cn/644680.Doc
fra.eiyve.cn/846686.Doc
fra.eiyve.cn/828860.Doc
fra.eiyve.cn/155117.Doc
fra.eiyve.cn/880844.Doc
fra.eiyve.cn/224886.Doc
fra.eiyve.cn/428668.Doc
fra.eiyve.cn/204242.Doc
fra.eiyve.cn/002806.Doc
fra.eiyve.cn/622226.Doc
fra.eiyve.cn/426020.Doc
frp.eiyve.cn/288046.Doc
frp.eiyve.cn/482266.Doc
frp.eiyve.cn/886806.Doc
frp.eiyve.cn/066200.Doc
frp.eiyve.cn/846264.Doc
frp.eiyve.cn/235780.Doc
frp.eiyve.cn/973614.Doc
frp.eiyve.cn/780426.Doc
frp.eiyve.cn/068228.Doc
frp.eiyve.cn/224846.Doc
fro.eiyve.cn/420026.Doc
fro.eiyve.cn/060026.Doc
fro.eiyve.cn/666282.Doc
fro.eiyve.cn/600220.Doc
fro.eiyve.cn/446626.Doc
fro.eiyve.cn/644046.Doc
fro.eiyve.cn/868020.Doc
fro.eiyve.cn/286860.Doc
fro.eiyve.cn/002844.Doc
fro.eiyve.cn/468684.Doc
fri.eiyve.cn/484202.Doc
fri.eiyve.cn/622080.Doc
fri.eiyve.cn/284226.Doc
fri.eiyve.cn/040644.Doc
fri.eiyve.cn/359773.Doc
fri.eiyve.cn/022868.Doc
fri.eiyve.cn/486222.Doc
fri.eiyve.cn/420262.Doc
fri.eiyve.cn/911591.Doc
fri.eiyve.cn/824226.Doc
fru.eiyve.cn/266826.Doc
fru.eiyve.cn/460666.Doc
fru.eiyve.cn/440824.Doc
fru.eiyve.cn/202644.Doc
fru.eiyve.cn/844680.Doc
fru.eiyve.cn/177791.Doc
fru.eiyve.cn/468802.Doc
fru.eiyve.cn/204242.Doc
fru.eiyve.cn/648468.Doc
fru.eiyve.cn/628222.Doc
fry.eiyve.cn/402484.Doc
fry.eiyve.cn/462240.Doc
fry.eiyve.cn/486880.Doc
fry.eiyve.cn/028262.Doc
fry.eiyve.cn/428268.Doc
fry.eiyve.cn/662862.Doc
fry.eiyve.cn/064006.Doc
fry.eiyve.cn/684602.Doc
fry.eiyve.cn/820800.Doc
fry.eiyve.cn/420664.Doc
frt.eiyve.cn/006802.Doc
frt.eiyve.cn/171513.Doc
frt.eiyve.cn/622860.Doc
frt.eiyve.cn/151399.Doc
frt.eiyve.cn/826848.Doc
frt.eiyve.cn/408806.Doc
frt.eiyve.cn/600884.Doc
frt.eiyve.cn/204662.Doc
frt.eiyve.cn/286604.Doc
frt.eiyve.cn/024084.Doc
frr.eiyve.cn/159555.Doc
frr.eiyve.cn/846204.Doc
frr.eiyve.cn/664206.Doc
frr.eiyve.cn/208868.Doc
frr.eiyve.cn/222228.Doc
frr.eiyve.cn/464200.Doc
frr.eiyve.cn/753957.Doc
frr.eiyve.cn/842246.Doc
frr.eiyve.cn/002644.Doc
frr.eiyve.cn/066240.Doc
fre.eiyve.cn/622666.Doc
fre.eiyve.cn/686420.Doc
fre.eiyve.cn/442028.Doc
fre.eiyve.cn/797535.Doc
fre.eiyve.cn/240004.Doc
fre.eiyve.cn/882622.Doc
fre.eiyve.cn/866468.Doc
fre.eiyve.cn/068080.Doc
fre.eiyve.cn/020480.Doc
fre.eiyve.cn/864244.Doc
frw.eiyve.cn/880404.Doc
frw.eiyve.cn/400064.Doc
frw.eiyve.cn/040006.Doc
frw.eiyve.cn/462804.Doc
frw.eiyve.cn/428482.Doc
frw.eiyve.cn/448626.Doc
frw.eiyve.cn/286200.Doc
frw.eiyve.cn/680804.Doc
frw.eiyve.cn/448868.Doc
frw.eiyve.cn/466264.Doc
frq.eiyve.cn/088204.Doc
frq.eiyve.cn/084824.Doc
frq.eiyve.cn/824426.Doc
frq.eiyve.cn/628468.Doc
frq.eiyve.cn/880420.Doc
frq.eiyve.cn/000660.Doc
frq.eiyve.cn/428462.Doc
frq.eiyve.cn/808608.Doc
frq.eiyve.cn/840284.Doc
frq.eiyve.cn/024028.Doc
fem.eiyve.cn/860884.Doc
fem.eiyve.cn/820824.Doc
fem.eiyve.cn/424680.Doc
fem.eiyve.cn/628046.Doc
fem.eiyve.cn/068242.Doc
fem.eiyve.cn/088042.Doc
fem.eiyve.cn/000840.Doc
fem.eiyve.cn/444666.Doc
fem.eiyve.cn/202446.Doc
fem.eiyve.cn/240840.Doc
fen.eiyve.cn/024084.Doc
fen.eiyve.cn/484208.Doc
fen.eiyve.cn/826046.Doc
fen.eiyve.cn/086046.Doc
fen.eiyve.cn/424440.Doc
fen.eiyve.cn/246620.Doc
fen.eiyve.cn/680228.Doc
fen.eiyve.cn/084208.Doc
fen.eiyve.cn/868424.Doc
fen.eiyve.cn/600082.Doc
feb.eiyve.cn/460660.Doc
feb.eiyve.cn/866246.Doc
feb.eiyve.cn/664262.Doc
feb.eiyve.cn/662604.Doc
feb.eiyve.cn/559337.Doc
feb.eiyve.cn/622862.Doc
feb.eiyve.cn/044248.Doc
feb.eiyve.cn/846602.Doc
feb.eiyve.cn/248602.Doc
feb.eiyve.cn/026640.Doc
fev.eiyve.cn/400806.Doc
fev.eiyve.cn/602824.Doc
fev.eiyve.cn/202024.Doc
fev.eiyve.cn/222848.Doc
fev.eiyve.cn/666024.Doc
fev.eiyve.cn/606620.Doc
fev.eiyve.cn/224206.Doc
fev.eiyve.cn/264068.Doc
fev.eiyve.cn/204064.Doc
fev.eiyve.cn/020282.Doc
fec.eiyve.cn/080460.Doc
fec.eiyve.cn/084042.Doc
fec.eiyve.cn/648888.Doc
fec.eiyve.cn/482204.Doc
fec.eiyve.cn/480444.Doc
fec.eiyve.cn/820668.Doc
fec.eiyve.cn/626046.Doc
fec.eiyve.cn/686060.Doc
fec.eiyve.cn/262082.Doc
fec.eiyve.cn/642282.Doc
fex.eiyve.cn/660626.Doc
fex.eiyve.cn/648428.Doc
fex.eiyve.cn/206624.Doc
fex.eiyve.cn/624640.Doc
fex.eiyve.cn/264844.Doc
fex.eiyve.cn/537133.Doc
fex.eiyve.cn/977519.Doc
fex.eiyve.cn/408468.Doc
fex.eiyve.cn/468080.Doc
fex.eiyve.cn/682862.Doc
fez.eiyve.cn/337737.Doc
fez.eiyve.cn/866846.Doc
fez.eiyve.cn/082286.Doc
fez.eiyve.cn/608404.Doc
fez.eiyve.cn/824806.Doc
fez.eiyve.cn/666462.Doc
fez.eiyve.cn/395951.Doc
fez.eiyve.cn/062240.Doc
fez.eiyve.cn/408804.Doc
fez.eiyve.cn/880486.Doc
fel.eiyve.cn/604406.Doc
fel.eiyve.cn/608044.Doc
fel.eiyve.cn/424020.Doc
fel.eiyve.cn/040424.Doc
fel.eiyve.cn/488482.Doc
fel.eiyve.cn/424068.Doc
fel.eiyve.cn/842464.Doc
fel.eiyve.cn/200240.Doc
fel.eiyve.cn/488602.Doc
fel.eiyve.cn/824226.Doc
fek.vwbnt.cn/802886.Doc
fek.vwbnt.cn/488648.Doc
fek.vwbnt.cn/737731.Doc
fek.vwbnt.cn/444088.Doc
fek.vwbnt.cn/917131.Doc
fek.vwbnt.cn/684462.Doc
fek.vwbnt.cn/608462.Doc
fek.vwbnt.cn/442428.Doc
fek.vwbnt.cn/444622.Doc
fek.vwbnt.cn/084022.Doc
fej.vwbnt.cn/286668.Doc
fej.vwbnt.cn/668620.Doc
fej.vwbnt.cn/044404.Doc
fej.vwbnt.cn/880640.Doc
fej.vwbnt.cn/248240.Doc
fej.vwbnt.cn/222244.Doc
fej.vwbnt.cn/466600.Doc
fej.vwbnt.cn/800020.Doc
fej.vwbnt.cn/460002.Doc
fej.vwbnt.cn/804686.Doc
feh.vwbnt.cn/973319.Doc
feh.vwbnt.cn/426200.Doc
feh.vwbnt.cn/660624.Doc
feh.vwbnt.cn/848468.Doc
feh.vwbnt.cn/260648.Doc
feh.vwbnt.cn/082644.Doc
feh.vwbnt.cn/202888.Doc
feh.vwbnt.cn/802444.Doc
feh.vwbnt.cn/421299.Doc
feh.vwbnt.cn/086080.Doc
feg.vwbnt.cn/866608.Doc
feg.vwbnt.cn/466206.Doc
feg.vwbnt.cn/288460.Doc
feg.vwbnt.cn/959339.Doc
feg.vwbnt.cn/604266.Doc
feg.vwbnt.cn/422042.Doc
feg.vwbnt.cn/660240.Doc
feg.vwbnt.cn/242424.Doc
feg.vwbnt.cn/282880.Doc
feg.vwbnt.cn/022622.Doc
fef.vwbnt.cn/828040.Doc
fef.vwbnt.cn/442286.Doc
fef.vwbnt.cn/520861.Doc
fef.vwbnt.cn/848462.Doc
fef.vwbnt.cn/802026.Doc
fef.vwbnt.cn/806260.Doc
fef.vwbnt.cn/860062.Doc
fef.vwbnt.cn/640820.Doc
fef.vwbnt.cn/846400.Doc
fef.vwbnt.cn/224008.Doc
fed.vwbnt.cn/426004.Doc
fed.vwbnt.cn/468604.Doc
fed.vwbnt.cn/600268.Doc
fed.vwbnt.cn/888664.Doc
fed.vwbnt.cn/842824.Doc
fed.vwbnt.cn/284286.Doc
fed.vwbnt.cn/206800.Doc
fed.vwbnt.cn/888866.Doc
fed.vwbnt.cn/860662.Doc
fed.vwbnt.cn/422084.Doc
fes.vwbnt.cn/644022.Doc
fes.vwbnt.cn/440006.Doc
fes.vwbnt.cn/020446.Doc
fes.vwbnt.cn/668286.Doc
fes.vwbnt.cn/706330.Doc
fes.vwbnt.cn/864026.Doc
fes.vwbnt.cn/686808.Doc
fes.vwbnt.cn/460604.Doc
fes.vwbnt.cn/440680.Doc
fes.vwbnt.cn/840428.Doc
fea.vwbnt.cn/246200.Doc
fea.vwbnt.cn/206440.Doc
fea.vwbnt.cn/888062.Doc
fea.vwbnt.cn/871282.Doc
fea.vwbnt.cn/883636.Doc
fea.vwbnt.cn/604440.Doc
fea.vwbnt.cn/042820.Doc
fea.vwbnt.cn/882624.Doc
fea.vwbnt.cn/311393.Doc
fea.vwbnt.cn/171599.Doc
fep.vwbnt.cn/608242.Doc
fep.vwbnt.cn/955999.Doc
fep.vwbnt.cn/420886.Doc
fep.vwbnt.cn/088444.Doc
fep.vwbnt.cn/406266.Doc
fep.vwbnt.cn/024648.Doc
fep.vwbnt.cn/886822.Doc
fep.vwbnt.cn/660608.Doc
fep.vwbnt.cn/684088.Doc
fep.vwbnt.cn/822682.Doc
feo.vwbnt.cn/806666.Doc
feo.vwbnt.cn/646844.Doc
feo.vwbnt.cn/022846.Doc
feo.vwbnt.cn/240642.Doc
feo.vwbnt.cn/620628.Doc
feo.vwbnt.cn/606860.Doc
feo.vwbnt.cn/919359.Doc
feo.vwbnt.cn/662440.Doc
feo.vwbnt.cn/260866.Doc
feo.vwbnt.cn/022400.Doc
fei.vwbnt.cn/957391.Doc
fei.vwbnt.cn/608628.Doc
fei.vwbnt.cn/800648.Doc
fei.vwbnt.cn/777519.Doc
