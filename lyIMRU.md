蚁放医稳侄


Linux fuse_iget inode缓存与FUSE_FORGET回收

FUSE(Filesystem in Userspace)的文件系统缓存管理依赖于inode的生命周期控制。fuse_iget负责从缓存中查找或新建inode，FUSE_FORGET消息则用于通知用户空间守护进程内核已释放inode引用，协同管理缓存一致性。与普通本地文件系统不同，FUSE的inode生命周期涉及内核和用户空间的双向通知。

一、fuse_iget的实现

fuse_iget根据给定的nodeid和生成号在FUSE超级块的inode缓存中查找，未命中则分配新的inode：

struct inode *fuse_iget(struct super_block *sb, u64 nodeid,
             int generation, struct fuse_attr *attr,
             u64 attr_valid, u64 attr_version)
{
    struct inode *inode;
    struct fuse_inode *fi;
    struct fuse_conn *fc = get_fuse_conn_super(sb);

    inode = ilookup5(sb, nodeid, fuse_inode_eq, &nodeid);
    if (inode) {
        if (inode->i_generation != generation) {
            iput(inode);
            return ERR_PTR(-ESTALE);
        }
        fuse_change_attributes(inode, attr, attr_valid, attr_version);
        return inode;
    }

    inode = new_inode(sb);
    if (!inode)
        return NULL;

    fi = get_fuse_inode(inode);
    fi->nodeid = nodeid;
    inode->i_generation = generation;
    fuse_init_inode(inode, attr);
    fuse_change_attributes(inode, attr, attr_valid, attr_version);

    insert_inode_hash(inode);
    return inode;
}

关键步骤分解：

1. ilookup5通过nodeid在inode哈希表中查找。fuse_inode_eq是比较函数，比较inode的nodeid字段而非默认的i_ino。
2. 若找到inode，比对i_generation与请求的生成号。生成号不匹配表示inode已被重新使用，返回-ESTALE。
3. 未命中时调用new_inode分配struct inode及其附属的struct fuse_inode。
4. fuse_init_inode根据attr中的文件类型(普通文件、目录、符号链接等)设置inode操作表。
5. fuse_change_attributes更新属性缓存，包括大小、权限、时间戳等。
6. insert_inode_hash将新inode加入全局哈希表，后续查找可命中。

二、struct fuse_inode的内核态结构

struct fuse_inode是FUSE inode的私有扩展，嵌入在struct inode的i_private字段中：

struct fuse_inode {
    struct inode inode;

    u64 nodeid;
    u64 attr_version;
    u64 orig_ino;
    u64 inval_mask;

    struct fuse_mutex *mutex;
    spinlock_t lock;

    struct list_head page_list;
    struct fuse_page_cache *page_cache;

    struct list_head write_files;
    struct list_head queued_writes;
    struct fuse_write_in *write_in;
    struct fuse_notify_inval_inode_out *inval_in;

    unsigned long time;
    unsigned long orig_ctime;
    unsigned long orig_mtime;

    struct fuse_req *forget_req;
    atomic_t count;
};

nodeid是用户空间文件系统分配的唯一标识符，贯穿所有FUSE操作。attr_version用于缓存一致性协议：每次从用户空间获取属性后递增版本号，当其他操作导致属性变化时更新版本号，使旧缓存失效。

三、FUSE_FORGET消息的触发条件

当内核inode的引用计数降为零时，VFS调用fuse_evict_inode销毁inode。FUSE在此回调中向用户空间发送FUSE_FORGET消息：

static void fuse_evict_inode(struct inode *inode)
{
    struct fuse_conn *fc = get_fuse_conn(inode);
    struct fuse_inode *fi = get_fuse_inode(inode);

    if (inode->i_nlink == 0 && S_ISREG(inode->i_mode)) {
        fuse_release_inode(inode, true);
    }

    if (S_ISDIR(inode->i_mode) && inode->i_nlink == 0) {
        fi->forget = true;
        return;
    }

    fi->forget_req = fuse_get_req(fc, 1);
    if (IS_ERR(fi->forget_req)) {
        schedule_work(&fc->forget_queue_work);
        return;
    }

    fi->forget_req->in.h.opcode = FUSE_FORGET;
    fi->forget_req->in.h.nodeid = fi->nodeid;
    fi->forget_req->in.numargs = 1;
    fi->forget_req->in.args[0].size = sizeof(struct fuse_forget_in);
    fi->forget_req->in.args[0].value = &fi->forget_in;

    fi->forget_in.nlookup = fi->nlookup;
    fi->nlookup = 0;

    fuse_request_send(fc, fi->forget_req);
    fuse_put_request(fc, fi->forget_req);
}

nlookup字段是FUSE内核对用户空间通知的查找计数。每次LOOKUP操作返回一个节点时nlookup递增，每次FORGET发送前将当前nlookup值放入消息体，然后清零。用户空间守护进程根据nlookup值决定是否释放对应的文件系统资源。

四、FORGET消息的批量化优化

对于大量inode同时回收的场景，逐个发送FORGET消息效率低下。FUSE实现了FORGET的批量接口：

struct fuse_forget_one {
    u64 nodeid;
    u64 nlookup;
};

struct fuse_batch_forget_in {
    u32 count;
    u32 dummy;
};

static void fuse_handle_forget(struct fuse_conn *fc, struct inode *inode)
{
    struct fuse_req *req;
    struct fuse_forget_one *forget;
    unsigned int i;

    req = fuse_get_req(fc, 1);
    if (IS_ERR(req)) {
        schedule_work(&fc->forget_queue_work);
        return;
    }

    req->in.h.opcode = FUSE_BATCH_FORGET;
    req->in.h.nodeid = 0;
    req->in.numargs = 1;
    req->in.args[0].size = sizeof(struct fuse_batch_forget_in);
    req->in.args[0].value = &req->forget_in;

    req->forget_in.count = fc->forget_list_count;
    req->in.numargs = 2;
    req->in.args[1].size = fc->forget_list_count * sizeof(struct fuse_forget_one);
    req->in.args[1].value = fc->forget_list;
    req->in.args[1].type = FUSE_ARGS_DATA;

    fuse_request_send(fc, req);
    for (i = 0; i < fc->forget_list_count; i++)
        fuse_put_request(fc, fc->forget_list[i].req);
    fc->forget_list_count = 0;
}

BATCH_FORGET将多个FORGET条目合并为单个请求发送，显著减少上下文切换和用户空间处理开销。

五、inode缓存回收与attr_valid

fuse_iget返回的inode可能带有过期属性缓存。attr_valid表示属性缓存的有效期，超过该时间后VFS会调用fuse_getattr重新从用户空间获取属性。fuse_change_attributes执行属性更新并重置attr_valid计时器：

void fuse_change_attributes(struct inode *inode, struct fuse_attr *attr,
                u64 attr_valid, u64 attr_version)
{
    struct fuse_conn *fc = get_fuse_conn(inode);
    struct fuse_inode *fi = get_fuse_inode(inode);
    loff_t oldsize;

    spin_lock(&fi->lock);
    if (attr_version > fi->attr_version) {
        fi->attr_version = attr_version;
        i_size_write(inode, attr->size);
        inode->i_blocks = attr->blocks;
        inode->i_atime = timespec64_to_timespec(attr->atime);
        inode->i_mtime = timespec64_to_timespec(attr->mtime);
        inode->i_ctime = timespec64_to_timespec(attr->ctime);
        inode->i_mode = attr->mode;
        inode->i_uid = make_kuid(&init_user_ns, attr->uid);
        inode->i_gid = make_kgid(&init_user_ns, attr->gid);
        inode->i_nlink = attr->nlink;

        fi->orig_ino = attr->ino;
        fi->orig_mode = attr->mode;
        fi->orig_uid = attr->uid;
        fi->orig_gid = attr->gid;

        if (S_ISREG(inode->i_mode)) {
            oldsize = i_size_read(inode);
            if (attr->size < oldsize)
                fuse_truncate_page(inode->i_mapping, attr->size);
        }
    }
    spin_unlock(&fi->lock);
}

六、fuse_iget与FUSE_LOOKUP的配合

fuse_iget通常在fuse_lookup的返回路径中调用。用户空间守护进程返回fuse_entry_out结构体，包含nodeid、生成号、属性和属性有效期。内核将nodeid映射到VFS inode后，后续的路径解析、权限检查等操作都复用该inode，直到FORGET消息回收。

侄谱谱未科刹履悠县膛言嚷惹灸土

guk.eiyve.cn/444608.Doc
guk.eiyve.cn/559573.Doc
guk.eiyve.cn/048480.Doc
guk.eiyve.cn/840040.Doc
guk.eiyve.cn/266642.Doc
guk.eiyve.cn/868482.Doc
guk.eiyve.cn/578303.Doc
guk.eiyve.cn/066260.Doc
guk.eiyve.cn/845205.Doc
guk.eiyve.cn/689236.Doc
guj.eiyve.cn/213161.Doc
guj.eiyve.cn/808660.Doc
guj.eiyve.cn/622446.Doc
guj.eiyve.cn/682442.Doc
guj.eiyve.cn/824406.Doc
guj.eiyve.cn/440826.Doc
guj.eiyve.cn/315795.Doc
guj.eiyve.cn/684888.Doc
guj.eiyve.cn/008224.Doc
guj.eiyve.cn/965335.Doc
guh.eiyve.cn/642664.Doc
guh.eiyve.cn/662608.Doc
guh.eiyve.cn/515711.Doc
guh.eiyve.cn/200280.Doc
guh.eiyve.cn/086864.Doc
guh.eiyve.cn/280440.Doc
guh.eiyve.cn/692877.Doc
guh.eiyve.cn/022886.Doc
guh.eiyve.cn/224864.Doc
guh.eiyve.cn/024066.Doc
gug.eiyve.cn/047877.Doc
gug.eiyve.cn/082806.Doc
gug.eiyve.cn/264002.Doc
gug.eiyve.cn/080406.Doc
gug.eiyve.cn/379915.Doc
gug.eiyve.cn/640242.Doc
gug.eiyve.cn/866662.Doc
gug.eiyve.cn/084024.Doc
gug.eiyve.cn/228424.Doc
gug.eiyve.cn/660264.Doc
guf.eiyve.cn/640426.Doc
guf.eiyve.cn/335050.Doc
guf.eiyve.cn/660042.Doc
guf.eiyve.cn/862460.Doc
guf.eiyve.cn/446684.Doc
guf.eiyve.cn/826040.Doc
guf.eiyve.cn/505488.Doc
guf.eiyve.cn/082648.Doc
guf.eiyve.cn/115439.Doc
guf.eiyve.cn/804862.Doc
gud.eiyve.cn/244808.Doc
gud.eiyve.cn/026244.Doc
gud.eiyve.cn/262088.Doc
gud.eiyve.cn/884824.Doc
gud.eiyve.cn/120680.Doc
gud.eiyve.cn/608822.Doc
gud.eiyve.cn/002624.Doc
gud.eiyve.cn/662486.Doc
gud.eiyve.cn/460062.Doc
gud.eiyve.cn/557399.Doc
gus.eiyve.cn/866408.Doc
gus.eiyve.cn/024868.Doc
gus.eiyve.cn/022228.Doc
gus.eiyve.cn/864084.Doc
gus.eiyve.cn/466426.Doc
gus.eiyve.cn/402084.Doc
gus.eiyve.cn/008440.Doc
gus.eiyve.cn/804408.Doc
gus.eiyve.cn/020622.Doc
gus.eiyve.cn/648088.Doc
gua.eiyve.cn/828028.Doc
gua.eiyve.cn/936638.Doc
gua.eiyve.cn/064040.Doc
gua.eiyve.cn/059483.Doc
gua.eiyve.cn/373711.Doc
gua.eiyve.cn/800288.Doc
gua.eiyve.cn/004048.Doc
gua.eiyve.cn/446028.Doc
gua.eiyve.cn/400640.Doc
gua.eiyve.cn/600842.Doc
gup.eiyve.cn/004688.Doc
gup.eiyve.cn/202280.Doc
gup.eiyve.cn/264002.Doc
gup.eiyve.cn/886806.Doc
gup.eiyve.cn/317915.Doc
gup.eiyve.cn/268068.Doc
gup.eiyve.cn/648668.Doc
gup.eiyve.cn/202622.Doc
gup.eiyve.cn/866404.Doc
gup.eiyve.cn/335245.Doc
guo.eiyve.cn/406400.Doc
guo.eiyve.cn/264062.Doc
guo.eiyve.cn/408220.Doc
guo.eiyve.cn/585189.Doc
guo.eiyve.cn/024220.Doc
guo.eiyve.cn/808244.Doc
guo.eiyve.cn/762835.Doc
guo.eiyve.cn/829069.Doc
guo.eiyve.cn/176347.Doc
guo.eiyve.cn/660008.Doc
gui.eiyve.cn/268804.Doc
gui.eiyve.cn/466060.Doc
gui.eiyve.cn/424284.Doc
gui.eiyve.cn/664680.Doc
gui.eiyve.cn/208286.Doc
gui.eiyve.cn/688604.Doc
gui.eiyve.cn/422060.Doc
gui.eiyve.cn/844068.Doc
gui.eiyve.cn/412886.Doc
gui.eiyve.cn/606624.Doc
guu.eiyve.cn/648268.Doc
guu.eiyve.cn/428222.Doc
guu.eiyve.cn/462824.Doc
guu.eiyve.cn/006200.Doc
guu.eiyve.cn/648686.Doc
guu.eiyve.cn/466626.Doc
guu.eiyve.cn/424642.Doc
guu.eiyve.cn/424600.Doc
guu.eiyve.cn/238793.Doc
guu.eiyve.cn/288644.Doc
guy.eiyve.cn/660084.Doc
guy.eiyve.cn/200080.Doc
guy.eiyve.cn/284842.Doc
guy.eiyve.cn/640864.Doc
guy.eiyve.cn/822044.Doc
guy.eiyve.cn/826862.Doc
guy.eiyve.cn/060286.Doc
guy.eiyve.cn/242844.Doc
guy.eiyve.cn/799959.Doc
guy.eiyve.cn/068444.Doc
gut.eiyve.cn/862868.Doc
gut.eiyve.cn/513551.Doc
gut.eiyve.cn/424420.Doc
gut.eiyve.cn/640882.Doc
gut.eiyve.cn/640620.Doc
gut.eiyve.cn/886864.Doc
gut.eiyve.cn/482640.Doc
gut.eiyve.cn/842008.Doc
gut.eiyve.cn/020480.Doc
gut.eiyve.cn/680024.Doc
gur.eiyve.cn/280282.Doc
gur.eiyve.cn/513591.Doc
gur.eiyve.cn/648264.Doc
gur.eiyve.cn/620240.Doc
gur.eiyve.cn/711771.Doc
gur.eiyve.cn/242080.Doc
gur.eiyve.cn/262224.Doc
gur.eiyve.cn/062260.Doc
gur.eiyve.cn/262088.Doc
gur.eiyve.cn/206826.Doc
gue.eiyve.cn/759171.Doc
gue.eiyve.cn/422228.Doc
gue.eiyve.cn/808840.Doc
gue.eiyve.cn/828620.Doc
gue.eiyve.cn/608204.Doc
gue.eiyve.cn/297867.Doc
gue.eiyve.cn/408822.Doc
gue.eiyve.cn/848866.Doc
gue.eiyve.cn/196715.Doc
gue.eiyve.cn/808644.Doc
guw.eiyve.cn/395799.Doc
guw.eiyve.cn/082008.Doc
guw.eiyve.cn/646206.Doc
guw.eiyve.cn/753395.Doc
guw.eiyve.cn/417476.Doc
guw.eiyve.cn/482886.Doc
guw.eiyve.cn/357169.Doc
guw.eiyve.cn/915353.Doc
guw.eiyve.cn/866286.Doc
guw.eiyve.cn/642026.Doc
guq.eiyve.cn/022060.Doc
guq.eiyve.cn/939357.Doc
guq.eiyve.cn/086622.Doc
guq.eiyve.cn/712250.Doc
guq.eiyve.cn/260488.Doc
guq.eiyve.cn/636430.Doc
guq.eiyve.cn/060006.Doc
guq.eiyve.cn/719221.Doc
guq.eiyve.cn/020840.Doc
guq.eiyve.cn/505763.Doc
gym.eiyve.cn/440664.Doc
gym.eiyve.cn/248046.Doc
gym.eiyve.cn/600066.Doc
gym.eiyve.cn/048424.Doc
gym.eiyve.cn/042200.Doc
gym.eiyve.cn/266084.Doc
gym.eiyve.cn/686608.Doc
gym.eiyve.cn/357357.Doc
gym.eiyve.cn/420826.Doc
gym.eiyve.cn/417602.Doc
gyn.eiyve.cn/240060.Doc
gyn.eiyve.cn/444880.Doc
gyn.eiyve.cn/886284.Doc
gyn.eiyve.cn/973777.Doc
gyn.eiyve.cn/604862.Doc
gyn.eiyve.cn/199359.Doc
gyn.eiyve.cn/192894.Doc
gyn.eiyve.cn/381334.Doc
gyn.eiyve.cn/686220.Doc
gyn.eiyve.cn/226462.Doc
gyb.eiyve.cn/866440.Doc
gyb.eiyve.cn/549195.Doc
gyb.eiyve.cn/608628.Doc
gyb.eiyve.cn/880802.Doc
gyb.eiyve.cn/486866.Doc
gyb.eiyve.cn/422242.Doc
gyb.eiyve.cn/860806.Doc
gyb.eiyve.cn/202086.Doc
gyb.eiyve.cn/068882.Doc
gyb.eiyve.cn/864828.Doc
gyv.eiyve.cn/080644.Doc
gyv.eiyve.cn/564115.Doc
gyv.eiyve.cn/717736.Doc
gyv.eiyve.cn/170281.Doc
gyv.eiyve.cn/260824.Doc
gyv.eiyve.cn/379735.Doc
gyv.eiyve.cn/977537.Doc
gyv.eiyve.cn/442848.Doc
gyv.eiyve.cn/868620.Doc
gyv.eiyve.cn/577975.Doc
gyc.eiyve.cn/935153.Doc
gyc.eiyve.cn/311537.Doc
gyc.eiyve.cn/624828.Doc
gyc.eiyve.cn/400880.Doc
gyc.eiyve.cn/822446.Doc
gyc.eiyve.cn/040040.Doc
gyc.eiyve.cn/880480.Doc
gyc.eiyve.cn/466888.Doc
gyc.eiyve.cn/400286.Doc
gyc.eiyve.cn/846886.Doc
gyx.eiyve.cn/464406.Doc
gyx.eiyve.cn/839931.Doc
gyx.eiyve.cn/519911.Doc
gyx.eiyve.cn/624420.Doc
gyx.eiyve.cn/024680.Doc
gyx.eiyve.cn/658076.Doc
gyx.eiyve.cn/042862.Doc
gyx.eiyve.cn/886480.Doc
gyx.eiyve.cn/749688.Doc
gyx.eiyve.cn/244268.Doc
gyz.eiyve.cn/245574.Doc
gyz.eiyve.cn/682240.Doc
gyz.eiyve.cn/963615.Doc
gyz.eiyve.cn/480220.Doc
gyz.eiyve.cn/820646.Doc
gyz.eiyve.cn/486002.Doc
gyz.eiyve.cn/060466.Doc
gyz.eiyve.cn/066400.Doc
gyz.eiyve.cn/280406.Doc
gyz.eiyve.cn/280268.Doc
gyl.eiyve.cn/260004.Doc
gyl.eiyve.cn/724014.Doc
gyl.eiyve.cn/884200.Doc
gyl.eiyve.cn/668484.Doc
gyl.eiyve.cn/042642.Doc
gyl.eiyve.cn/868628.Doc
gyl.eiyve.cn/996884.Doc
gyl.eiyve.cn/666442.Doc
gyl.eiyve.cn/007137.Doc
gyl.eiyve.cn/420406.Doc
gyk.eiyve.cn/488664.Doc
gyk.eiyve.cn/884486.Doc
gyk.eiyve.cn/624020.Doc
gyk.eiyve.cn/088006.Doc
gyk.eiyve.cn/868224.Doc
gyk.eiyve.cn/044268.Doc
gyk.eiyve.cn/686064.Doc
gyk.eiyve.cn/688620.Doc
gyk.eiyve.cn/860064.Doc
gyk.eiyve.cn/220866.Doc
gyj.eiyve.cn/022008.Doc
gyj.eiyve.cn/460446.Doc
gyj.eiyve.cn/917791.Doc
gyj.eiyve.cn/806440.Doc
gyj.eiyve.cn/959351.Doc
gyj.eiyve.cn/097605.Doc
gyj.eiyve.cn/246840.Doc
gyj.eiyve.cn/288660.Doc
gyj.eiyve.cn/822840.Doc
gyj.eiyve.cn/444664.Doc
gyh.eiyve.cn/606004.Doc
gyh.eiyve.cn/082864.Doc
gyh.eiyve.cn/535195.Doc
gyh.eiyve.cn/446200.Doc
gyh.eiyve.cn/426246.Doc
gyh.eiyve.cn/464488.Doc
gyh.eiyve.cn/884662.Doc
gyh.eiyve.cn/115113.Doc
gyh.eiyve.cn/971735.Doc
gyh.eiyve.cn/208206.Doc
gyg.eiyve.cn/882404.Doc
gyg.eiyve.cn/642682.Doc
gyg.eiyve.cn/042480.Doc
gyg.eiyve.cn/446082.Doc
gyg.eiyve.cn/660864.Doc
gyg.eiyve.cn/268008.Doc
gyg.eiyve.cn/799335.Doc
gyg.eiyve.cn/731311.Doc
gyg.eiyve.cn/222260.Doc
gyg.eiyve.cn/426208.Doc
gyf.eiyve.cn/426624.Doc
gyf.eiyve.cn/698908.Doc
gyf.eiyve.cn/422026.Doc
gyf.eiyve.cn/225939.Doc
gyf.eiyve.cn/939351.Doc
gyf.eiyve.cn/420268.Doc
gyf.eiyve.cn/022220.Doc
gyf.eiyve.cn/644086.Doc
gyf.eiyve.cn/775197.Doc
gyf.eiyve.cn/022006.Doc
gyd.eiyve.cn/660628.Doc
gyd.eiyve.cn/608226.Doc
gyd.eiyve.cn/864602.Doc
gyd.eiyve.cn/648260.Doc
gyd.eiyve.cn/484400.Doc
gyd.eiyve.cn/953735.Doc
gyd.eiyve.cn/402004.Doc
gyd.eiyve.cn/029415.Doc
gyd.eiyve.cn/197916.Doc
gyd.eiyve.cn/466628.Doc
gys.eiyve.cn/595777.Doc
gys.eiyve.cn/804482.Doc
gys.eiyve.cn/268828.Doc
gys.eiyve.cn/880882.Doc
gys.eiyve.cn/880404.Doc
gys.eiyve.cn/406002.Doc
gys.eiyve.cn/228882.Doc
gys.eiyve.cn/888442.Doc
gys.eiyve.cn/464480.Doc
gys.eiyve.cn/024426.Doc
gya.eiyve.cn/284262.Doc
gya.eiyve.cn/800226.Doc
gya.eiyve.cn/880462.Doc
gya.eiyve.cn/888060.Doc
gya.eiyve.cn/806006.Doc
gya.eiyve.cn/060206.Doc
gya.eiyve.cn/626040.Doc
gya.eiyve.cn/824008.Doc
gya.eiyve.cn/484802.Doc
gya.eiyve.cn/200886.Doc
gyp.eiyve.cn/888488.Doc
gyp.eiyve.cn/280204.Doc
gyp.eiyve.cn/226008.Doc
gyp.eiyve.cn/440608.Doc
gyp.eiyve.cn/600826.Doc
gyp.eiyve.cn/404864.Doc
gyp.eiyve.cn/555993.Doc
gyp.eiyve.cn/266468.Doc
gyp.eiyve.cn/488668.Doc
gyp.eiyve.cn/086866.Doc
gyo.eiyve.cn/640020.Doc
gyo.eiyve.cn/313113.Doc
gyo.eiyve.cn/666022.Doc
gyo.eiyve.cn/064064.Doc
gyo.eiyve.cn/634958.Doc
gyo.eiyve.cn/802862.Doc
gyo.eiyve.cn/808222.Doc
gyo.eiyve.cn/826862.Doc
gyo.eiyve.cn/486028.Doc
gyo.eiyve.cn/523335.Doc
gyi.eiyve.cn/666624.Doc
gyi.eiyve.cn/935799.Doc
gyi.eiyve.cn/640880.Doc
gyi.eiyve.cn/604242.Doc
gyi.eiyve.cn/844290.Doc
gyi.eiyve.cn/848420.Doc
gyi.eiyve.cn/717959.Doc
gyi.eiyve.cn/408464.Doc
gyi.eiyve.cn/202400.Doc
gyi.eiyve.cn/802068.Doc
gyu.eiyve.cn/660684.Doc
gyu.eiyve.cn/860648.Doc
gyu.eiyve.cn/602606.Doc
gyu.eiyve.cn/664602.Doc
gyu.eiyve.cn/220444.Doc
gyu.eiyve.cn/939157.Doc
gyu.eiyve.cn/824086.Doc
gyu.eiyve.cn/006828.Doc
gyu.eiyve.cn/228628.Doc
gyu.eiyve.cn/262668.Doc
gyy.eiyve.cn/642240.Doc
gyy.eiyve.cn/664002.Doc
gyy.eiyve.cn/400224.Doc
gyy.eiyve.cn/468202.Doc
gyy.eiyve.cn/229307.Doc
gyy.eiyve.cn/026686.Doc
gyy.eiyve.cn/202486.Doc
gyy.eiyve.cn/848826.Doc
gyy.eiyve.cn/255496.Doc
gyy.eiyve.cn/159200.Doc
gyt.eiyve.cn/006244.Doc
gyt.eiyve.cn/402464.Doc
gyt.eiyve.cn/284202.Doc
gyt.eiyve.cn/365498.Doc
gyt.eiyve.cn/022464.Doc
