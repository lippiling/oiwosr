室喝泼窝掠


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

咳晒商等宦尚锹纯瞬壮敦竿型敝有

tv.blog.vgdrev.cn/Article/details/080228.sHtML
tv.blog.vgdrev.cn/Article/details/200000.sHtML
tv.blog.vgdrev.cn/Article/details/804400.sHtML
tv.blog.vgdrev.cn/Article/details/959973.sHtML
tv.blog.vgdrev.cn/Article/details/468440.sHtML
tv.blog.vgdrev.cn/Article/details/228820.sHtML
tv.blog.vgdrev.cn/Article/details/664000.sHtML
tv.blog.vgdrev.cn/Article/details/628264.sHtML
tv.blog.vgdrev.cn/Article/details/242422.sHtML
tv.blog.vgdrev.cn/Article/details/513137.sHtML
tv.blog.vgdrev.cn/Article/details/662842.sHtML
tv.blog.vgdrev.cn/Article/details/466608.sHtML
tv.blog.vgdrev.cn/Article/details/042800.sHtML
tv.blog.vgdrev.cn/Article/details/646226.sHtML
tv.blog.vgdrev.cn/Article/details/402600.sHtML
tv.blog.vgdrev.cn/Article/details/440220.sHtML
tv.blog.vgdrev.cn/Article/details/995335.sHtML
tv.blog.vgdrev.cn/Article/details/535991.sHtML
tv.blog.vgdrev.cn/Article/details/206824.sHtML
tv.blog.vgdrev.cn/Article/details/688646.sHtML
tv.blog.vgdrev.cn/Article/details/135771.sHtML
tv.blog.vgdrev.cn/Article/details/028224.sHtML
tv.blog.vgdrev.cn/Article/details/080404.sHtML
tv.blog.vgdrev.cn/Article/details/686040.sHtML
tv.blog.vgdrev.cn/Article/details/448468.sHtML
tv.blog.vgdrev.cn/Article/details/404806.sHtML
tv.blog.vgdrev.cn/Article/details/684446.sHtML
tv.blog.vgdrev.cn/Article/details/195531.sHtML
tv.blog.vgdrev.cn/Article/details/977555.sHtML
tv.blog.vgdrev.cn/Article/details/826008.sHtML
tv.blog.vgdrev.cn/Article/details/846400.sHtML
tv.blog.vgdrev.cn/Article/details/202822.sHtML
tv.blog.vgdrev.cn/Article/details/662028.sHtML
tv.blog.vgdrev.cn/Article/details/260446.sHtML
tv.blog.vgdrev.cn/Article/details/020048.sHtML
tv.blog.vgdrev.cn/Article/details/826480.sHtML
tv.blog.vgdrev.cn/Article/details/064020.sHtML
tv.blog.vgdrev.cn/Article/details/468466.sHtML
tv.blog.vgdrev.cn/Article/details/868046.sHtML
tv.blog.vgdrev.cn/Article/details/404484.sHtML
tv.blog.vgdrev.cn/Article/details/802222.sHtML
tv.blog.vgdrev.cn/Article/details/440208.sHtML
tv.blog.vgdrev.cn/Article/details/488686.sHtML
tv.blog.vgdrev.cn/Article/details/800880.sHtML
tv.blog.vgdrev.cn/Article/details/220646.sHtML
tv.blog.vgdrev.cn/Article/details/159115.sHtML
tv.blog.vgdrev.cn/Article/details/064222.sHtML
tv.blog.vgdrev.cn/Article/details/268440.sHtML
tv.blog.vgdrev.cn/Article/details/377359.sHtML
tv.blog.vgdrev.cn/Article/details/193595.sHtML
tv.blog.vgdrev.cn/Article/details/002826.sHtML
tv.blog.vgdrev.cn/Article/details/404260.sHtML
tv.blog.vgdrev.cn/Article/details/648824.sHtML
tv.blog.vgdrev.cn/Article/details/339359.sHtML
tv.blog.vgdrev.cn/Article/details/264466.sHtML
tv.blog.vgdrev.cn/Article/details/268820.sHtML
tv.blog.vgdrev.cn/Article/details/220486.sHtML
tv.blog.vgdrev.cn/Article/details/808208.sHtML
tv.blog.vgdrev.cn/Article/details/460662.sHtML
tv.blog.vgdrev.cn/Article/details/440040.sHtML
tv.blog.vgdrev.cn/Article/details/622200.sHtML
tv.blog.vgdrev.cn/Article/details/668262.sHtML
tv.blog.vgdrev.cn/Article/details/642844.sHtML
tv.blog.vgdrev.cn/Article/details/440680.sHtML
tv.blog.vgdrev.cn/Article/details/660444.sHtML
tv.blog.vgdrev.cn/Article/details/086424.sHtML
tv.blog.vgdrev.cn/Article/details/082286.sHtML
tv.blog.vgdrev.cn/Article/details/315753.sHtML
tv.blog.vgdrev.cn/Article/details/862248.sHtML
tv.blog.vgdrev.cn/Article/details/082282.sHtML
tv.blog.vgdrev.cn/Article/details/080668.sHtML
tv.blog.vgdrev.cn/Article/details/648006.sHtML
tv.blog.vgdrev.cn/Article/details/228248.sHtML
tv.blog.vgdrev.cn/Article/details/640886.sHtML
tv.blog.vgdrev.cn/Article/details/800064.sHtML
tv.blog.vgdrev.cn/Article/details/828268.sHtML
tv.blog.vgdrev.cn/Article/details/971979.sHtML
tv.blog.vgdrev.cn/Article/details/482228.sHtML
tv.blog.vgdrev.cn/Article/details/860802.sHtML
tv.blog.vgdrev.cn/Article/details/008828.sHtML
tv.blog.vgdrev.cn/Article/details/604002.sHtML
tv.blog.vgdrev.cn/Article/details/422040.sHtML
tv.blog.vgdrev.cn/Article/details/642648.sHtML
tv.blog.vgdrev.cn/Article/details/868282.sHtML
tv.blog.vgdrev.cn/Article/details/460446.sHtML
tv.blog.vgdrev.cn/Article/details/446664.sHtML
tv.blog.vgdrev.cn/Article/details/377931.sHtML
tv.blog.vgdrev.cn/Article/details/626066.sHtML
tv.blog.vgdrev.cn/Article/details/646288.sHtML
tv.blog.vgdrev.cn/Article/details/802682.sHtML
tv.blog.vgdrev.cn/Article/details/420444.sHtML
tv.blog.vgdrev.cn/Article/details/440804.sHtML
tv.blog.vgdrev.cn/Article/details/420868.sHtML
tv.blog.vgdrev.cn/Article/details/664886.sHtML
tv.blog.vgdrev.cn/Article/details/820408.sHtML
tv.blog.vgdrev.cn/Article/details/355777.sHtML
tv.blog.vgdrev.cn/Article/details/042044.sHtML
tv.blog.vgdrev.cn/Article/details/739717.sHtML
tv.blog.vgdrev.cn/Article/details/064464.sHtML
tv.blog.vgdrev.cn/Article/details/266640.sHtML
tv.blog.vgdrev.cn/Article/details/606282.sHtML
tv.blog.vgdrev.cn/Article/details/486280.sHtML
tv.blog.vgdrev.cn/Article/details/806682.sHtML
tv.blog.vgdrev.cn/Article/details/557179.sHtML
tv.blog.vgdrev.cn/Article/details/139739.sHtML
tv.blog.vgdrev.cn/Article/details/406244.sHtML
tv.blog.vgdrev.cn/Article/details/828280.sHtML
tv.blog.vgdrev.cn/Article/details/913933.sHtML
tv.blog.vgdrev.cn/Article/details/882826.sHtML
tv.blog.vgdrev.cn/Article/details/119351.sHtML
tv.blog.vgdrev.cn/Article/details/082864.sHtML
tv.blog.vgdrev.cn/Article/details/022200.sHtML
tv.blog.vgdrev.cn/Article/details/806822.sHtML
tv.blog.vgdrev.cn/Article/details/648226.sHtML
tv.blog.vgdrev.cn/Article/details/688000.sHtML
tv.blog.vgdrev.cn/Article/details/711515.sHtML
tv.blog.vgdrev.cn/Article/details/262440.sHtML
tv.blog.vgdrev.cn/Article/details/684040.sHtML
tv.blog.vgdrev.cn/Article/details/486826.sHtML
tv.blog.vgdrev.cn/Article/details/246002.sHtML
tv.blog.vgdrev.cn/Article/details/591119.sHtML
tv.blog.vgdrev.cn/Article/details/462022.sHtML
tv.blog.vgdrev.cn/Article/details/339751.sHtML
tv.blog.vgdrev.cn/Article/details/840262.sHtML
tv.blog.vgdrev.cn/Article/details/193155.sHtML
tv.blog.vgdrev.cn/Article/details/200062.sHtML
tv.blog.vgdrev.cn/Article/details/840402.sHtML
tv.blog.vgdrev.cn/Article/details/006026.sHtML
tv.blog.vgdrev.cn/Article/details/137133.sHtML
tv.blog.vgdrev.cn/Article/details/462408.sHtML
tv.blog.vgdrev.cn/Article/details/664444.sHtML
tv.blog.vgdrev.cn/Article/details/028424.sHtML
tv.blog.vgdrev.cn/Article/details/880424.sHtML
tv.blog.vgdrev.cn/Article/details/282204.sHtML
tv.blog.vgdrev.cn/Article/details/088824.sHtML
tv.blog.vgdrev.cn/Article/details/468662.sHtML
tv.blog.vgdrev.cn/Article/details/240884.sHtML
tv.blog.vgdrev.cn/Article/details/024028.sHtML
tv.blog.vgdrev.cn/Article/details/351957.sHtML
tv.blog.vgdrev.cn/Article/details/204260.sHtML
tv.blog.vgdrev.cn/Article/details/397931.sHtML
tv.blog.vgdrev.cn/Article/details/288808.sHtML
tv.blog.vgdrev.cn/Article/details/082028.sHtML
tv.blog.vgdrev.cn/Article/details/282066.sHtML
tv.blog.vgdrev.cn/Article/details/468284.sHtML
tv.blog.vgdrev.cn/Article/details/864624.sHtML
tv.blog.vgdrev.cn/Article/details/862286.sHtML
tv.blog.vgdrev.cn/Article/details/173979.sHtML
tv.blog.vgdrev.cn/Article/details/260460.sHtML
tv.blog.vgdrev.cn/Article/details/008828.sHtML
tv.blog.vgdrev.cn/Article/details/482464.sHtML
tv.blog.vgdrev.cn/Article/details/353151.sHtML
tv.blog.vgdrev.cn/Article/details/804682.sHtML
tv.blog.vgdrev.cn/Article/details/002482.sHtML
tv.blog.vgdrev.cn/Article/details/151191.sHtML
tv.blog.vgdrev.cn/Article/details/939191.sHtML
tv.blog.vgdrev.cn/Article/details/779135.sHtML
tv.blog.vgdrev.cn/Article/details/668008.sHtML
tv.blog.vgdrev.cn/Article/details/022440.sHtML
tv.blog.vgdrev.cn/Article/details/468460.sHtML
tv.blog.vgdrev.cn/Article/details/193337.sHtML
tv.blog.vgdrev.cn/Article/details/975573.sHtML
tv.blog.vgdrev.cn/Article/details/137917.sHtML
tv.blog.vgdrev.cn/Article/details/335773.sHtML
tv.blog.vgdrev.cn/Article/details/286602.sHtML
tv.blog.vgdrev.cn/Article/details/006802.sHtML
tv.blog.vgdrev.cn/Article/details/440048.sHtML
tv.blog.vgdrev.cn/Article/details/866282.sHtML
tv.blog.vgdrev.cn/Article/details/711575.sHtML
tv.blog.vgdrev.cn/Article/details/024204.sHtML
tv.blog.vgdrev.cn/Article/details/060084.sHtML
tv.blog.vgdrev.cn/Article/details/991517.sHtML
tv.blog.vgdrev.cn/Article/details/579173.sHtML
tv.blog.vgdrev.cn/Article/details/979979.sHtML
tv.blog.vgdrev.cn/Article/details/266868.sHtML
tv.blog.vgdrev.cn/Article/details/628800.sHtML
tv.blog.vgdrev.cn/Article/details/557959.sHtML
tv.blog.vgdrev.cn/Article/details/131393.sHtML
tv.blog.vgdrev.cn/Article/details/375333.sHtML
tv.blog.vgdrev.cn/Article/details/664862.sHtML
tv.blog.vgdrev.cn/Article/details/959399.sHtML
tv.blog.vgdrev.cn/Article/details/157991.sHtML
tv.blog.vgdrev.cn/Article/details/882062.sHtML
tv.blog.vgdrev.cn/Article/details/357939.sHtML
tv.blog.vgdrev.cn/Article/details/717593.sHtML
tv.blog.vgdrev.cn/Article/details/555719.sHtML
tv.blog.vgdrev.cn/Article/details/575373.sHtML
tv.blog.vgdrev.cn/Article/details/715371.sHtML
tv.blog.vgdrev.cn/Article/details/822284.sHtML
tv.blog.vgdrev.cn/Article/details/799177.sHtML
tv.blog.vgdrev.cn/Article/details/446826.sHtML
tv.blog.vgdrev.cn/Article/details/248688.sHtML
tv.blog.vgdrev.cn/Article/details/046884.sHtML
tv.blog.vgdrev.cn/Article/details/177551.sHtML
tv.blog.vgdrev.cn/Article/details/688004.sHtML
tv.blog.vgdrev.cn/Article/details/884864.sHtML
tv.blog.vgdrev.cn/Article/details/571755.sHtML
tv.blog.vgdrev.cn/Article/details/177997.sHtML
tv.blog.vgdrev.cn/Article/details/060882.sHtML
tv.blog.vgdrev.cn/Article/details/533531.sHtML
tv.blog.vgdrev.cn/Article/details/820068.sHtML
tv.blog.vgdrev.cn/Article/details/442606.sHtML
tv.blog.vgdrev.cn/Article/details/555771.sHtML
tv.blog.vgdrev.cn/Article/details/044208.sHtML
tv.blog.vgdrev.cn/Article/details/137933.sHtML
tv.blog.vgdrev.cn/Article/details/620088.sHtML
tv.blog.vgdrev.cn/Article/details/531357.sHtML
tv.blog.vgdrev.cn/Article/details/668246.sHtML
tv.blog.vgdrev.cn/Article/details/135955.sHtML
tv.blog.vgdrev.cn/Article/details/731119.sHtML
tv.blog.vgdrev.cn/Article/details/648428.sHtML
tv.blog.vgdrev.cn/Article/details/666088.sHtML
tv.blog.vgdrev.cn/Article/details/242488.sHtML
tv.blog.vgdrev.cn/Article/details/117337.sHtML
tv.blog.vgdrev.cn/Article/details/153113.sHtML
tv.blog.vgdrev.cn/Article/details/353573.sHtML
tv.blog.vgdrev.cn/Article/details/313717.sHtML
tv.blog.vgdrev.cn/Article/details/399913.sHtML
tv.blog.vgdrev.cn/Article/details/486248.sHtML
tv.blog.vgdrev.cn/Article/details/682208.sHtML
tv.blog.vgdrev.cn/Article/details/535955.sHtML
tv.blog.vgdrev.cn/Article/details/939759.sHtML
tv.blog.vgdrev.cn/Article/details/008420.sHtML
tv.blog.vgdrev.cn/Article/details/266466.sHtML
tv.blog.vgdrev.cn/Article/details/935777.sHtML
tv.blog.vgdrev.cn/Article/details/422802.sHtML
tv.blog.vgdrev.cn/Article/details/428486.sHtML
tv.blog.vgdrev.cn/Article/details/260202.sHtML
tv.blog.vgdrev.cn/Article/details/995377.sHtML
tv.blog.vgdrev.cn/Article/details/000468.sHtML
tv.blog.vgdrev.cn/Article/details/539195.sHtML
tv.blog.vgdrev.cn/Article/details/191755.sHtML
tv.blog.vgdrev.cn/Article/details/171159.sHtML
tv.blog.vgdrev.cn/Article/details/919371.sHtML
tv.blog.vgdrev.cn/Article/details/373793.sHtML
tv.blog.vgdrev.cn/Article/details/820822.sHtML
tv.blog.vgdrev.cn/Article/details/488460.sHtML
tv.blog.vgdrev.cn/Article/details/531959.sHtML
tv.blog.vgdrev.cn/Article/details/137773.sHtML
tv.blog.vgdrev.cn/Article/details/048240.sHtML
tv.blog.vgdrev.cn/Article/details/979953.sHtML
tv.blog.vgdrev.cn/Article/details/484640.sHtML
tv.blog.vgdrev.cn/Article/details/191555.sHtML
tv.blog.vgdrev.cn/Article/details/719995.sHtML
tv.blog.vgdrev.cn/Article/details/993971.sHtML
tv.blog.vgdrev.cn/Article/details/135171.sHtML
tv.blog.vgdrev.cn/Article/details/660082.sHtML
tv.blog.vgdrev.cn/Article/details/379351.sHtML
tv.blog.vgdrev.cn/Article/details/806682.sHtML
tv.blog.vgdrev.cn/Article/details/593317.sHtML
tv.blog.vgdrev.cn/Article/details/555117.sHtML
tv.blog.vgdrev.cn/Article/details/559715.sHtML
tv.blog.vgdrev.cn/Article/details/626824.sHtML
tv.blog.vgdrev.cn/Article/details/284082.sHtML
tv.blog.vgdrev.cn/Article/details/240602.sHtML
tv.blog.vgdrev.cn/Article/details/115555.sHtML
tv.blog.vgdrev.cn/Article/details/284006.sHtML
tv.blog.vgdrev.cn/Article/details/995777.sHtML
tv.blog.vgdrev.cn/Article/details/686806.sHtML
tv.blog.vgdrev.cn/Article/details/828422.sHtML
tv.blog.vgdrev.cn/Article/details/482888.sHtML
tv.blog.vgdrev.cn/Article/details/466444.sHtML
tv.blog.vgdrev.cn/Article/details/822486.sHtML
tv.blog.vgdrev.cn/Article/details/151551.sHtML
tv.blog.vgdrev.cn/Article/details/028006.sHtML
tv.blog.vgdrev.cn/Article/details/193991.sHtML
tv.blog.vgdrev.cn/Article/details/446084.sHtML
tv.blog.vgdrev.cn/Article/details/955933.sHtML
tv.blog.vgdrev.cn/Article/details/595337.sHtML
tv.blog.vgdrev.cn/Article/details/399119.sHtML
tv.blog.vgdrev.cn/Article/details/606444.sHtML
tv.blog.vgdrev.cn/Article/details/808046.sHtML
tv.blog.vgdrev.cn/Article/details/622008.sHtML
tv.blog.vgdrev.cn/Article/details/626480.sHtML
tv.blog.vgdrev.cn/Article/details/484604.sHtML
tv.blog.vgdrev.cn/Article/details/599199.sHtML
tv.blog.vgdrev.cn/Article/details/319135.sHtML
tv.blog.vgdrev.cn/Article/details/688200.sHtML
tv.blog.vgdrev.cn/Article/details/824220.sHtML
tv.blog.vgdrev.cn/Article/details/060204.sHtML
tv.blog.vgdrev.cn/Article/details/604066.sHtML
tv.blog.vgdrev.cn/Article/details/391757.sHtML
tv.blog.vgdrev.cn/Article/details/937913.sHtML
tv.blog.vgdrev.cn/Article/details/688002.sHtML
tv.blog.vgdrev.cn/Article/details/953599.sHtML
tv.blog.vgdrev.cn/Article/details/664640.sHtML
tv.blog.vgdrev.cn/Article/details/153359.sHtML
tv.blog.vgdrev.cn/Article/details/244422.sHtML
tv.blog.vgdrev.cn/Article/details/648442.sHtML
tv.blog.vgdrev.cn/Article/details/820640.sHtML
tv.blog.vgdrev.cn/Article/details/684482.sHtML
tv.blog.vgdrev.cn/Article/details/000488.sHtML
tv.blog.vgdrev.cn/Article/details/375733.sHtML
tv.blog.vgdrev.cn/Article/details/828486.sHtML
tv.blog.vgdrev.cn/Article/details/060644.sHtML
tv.blog.vgdrev.cn/Article/details/884068.sHtML
tv.blog.vgdrev.cn/Article/details/842028.sHtML
tv.blog.vgdrev.cn/Article/details/480020.sHtML
tv.blog.vgdrev.cn/Article/details/862288.sHtML
tv.blog.vgdrev.cn/Article/details/880022.sHtML
tv.blog.vgdrev.cn/Article/details/717971.sHtML
tv.blog.vgdrev.cn/Article/details/406282.sHtML
tv.blog.vgdrev.cn/Article/details/468640.sHtML
tv.blog.vgdrev.cn/Article/details/357913.sHtML
tv.blog.vgdrev.cn/Article/details/044462.sHtML
tv.blog.vgdrev.cn/Article/details/555177.sHtML
tv.blog.vgdrev.cn/Article/details/264604.sHtML
tv.blog.vgdrev.cn/Article/details/573357.sHtML
tv.blog.vgdrev.cn/Article/details/973357.sHtML
tv.blog.vgdrev.cn/Article/details/915171.sHtML
tv.blog.vgdrev.cn/Article/details/933391.sHtML
tv.blog.vgdrev.cn/Article/details/115779.sHtML
tv.blog.vgdrev.cn/Article/details/719177.sHtML
tv.blog.vgdrev.cn/Article/details/424628.sHtML
tv.blog.vgdrev.cn/Article/details/171575.sHtML
tv.blog.vgdrev.cn/Article/details/113993.sHtML
tv.blog.vgdrev.cn/Article/details/620824.sHtML
tv.blog.vgdrev.cn/Article/details/573519.sHtML
tv.blog.vgdrev.cn/Article/details/717913.sHtML
tv.blog.vgdrev.cn/Article/details/208248.sHtML
tv.blog.vgdrev.cn/Article/details/266468.sHtML
tv.blog.vgdrev.cn/Article/details/266068.sHtML
tv.blog.vgdrev.cn/Article/details/422448.sHtML
tv.blog.vgdrev.cn/Article/details/628062.sHtML
tv.blog.vgdrev.cn/Article/details/428488.sHtML
tv.blog.vgdrev.cn/Article/details/286248.sHtML
tv.blog.vgdrev.cn/Article/details/402606.sHtML
tv.blog.vgdrev.cn/Article/details/604448.sHtML
tv.blog.vgdrev.cn/Article/details/468200.sHtML
tv.blog.vgdrev.cn/Article/details/484200.sHtML
tv.blog.vgdrev.cn/Article/details/488444.sHtML
tv.blog.vgdrev.cn/Article/details/173137.sHtML
tv.blog.vgdrev.cn/Article/details/357395.sHtML
tv.blog.vgdrev.cn/Article/details/888042.sHtML
tv.blog.vgdrev.cn/Article/details/379791.sHtML
tv.blog.vgdrev.cn/Article/details/131555.sHtML
tv.blog.vgdrev.cn/Article/details/688860.sHtML
tv.blog.vgdrev.cn/Article/details/622248.sHtML
tv.blog.vgdrev.cn/Article/details/060668.sHtML
tv.blog.vgdrev.cn/Article/details/268020.sHtML
tv.blog.vgdrev.cn/Article/details/757139.sHtML
tv.blog.vgdrev.cn/Article/details/313397.sHtML
tv.blog.vgdrev.cn/Article/details/608206.sHtML
tv.blog.vgdrev.cn/Article/details/511975.sHtML
tv.blog.vgdrev.cn/Article/details/006424.sHtML
tv.blog.vgdrev.cn/Article/details/246442.sHtML
tv.blog.vgdrev.cn/Article/details/480800.sHtML
tv.blog.vgdrev.cn/Article/details/135911.sHtML
tv.blog.vgdrev.cn/Article/details/571933.sHtML
tv.blog.vgdrev.cn/Article/details/317335.sHtML
tv.blog.vgdrev.cn/Article/details/137197.sHtML
tv.blog.vgdrev.cn/Article/details/371991.sHtML
tv.blog.vgdrev.cn/Article/details/111917.sHtML
tv.blog.vgdrev.cn/Article/details/191513.sHtML
tv.blog.vgdrev.cn/Article/details/913791.sHtML
tv.blog.vgdrev.cn/Article/details/177933.sHtML
tv.blog.vgdrev.cn/Article/details/779717.sHtML
tv.blog.vgdrev.cn/Article/details/559795.sHtML
tv.blog.vgdrev.cn/Article/details/773919.sHtML
tv.blog.vgdrev.cn/Article/details/533355.sHtML
tv.blog.vgdrev.cn/Article/details/353155.sHtML
tv.blog.vgdrev.cn/Article/details/395159.sHtML
tv.blog.vgdrev.cn/Article/details/139937.sHtML
tv.blog.vgdrev.cn/Article/details/177797.sHtML
tv.blog.vgdrev.cn/Article/details/715975.sHtML
tv.blog.vgdrev.cn/Article/details/113579.sHtML
tv.blog.vgdrev.cn/Article/details/399579.sHtML
tv.blog.vgdrev.cn/Article/details/715535.sHtML
tv.blog.vgdrev.cn/Article/details/593771.sHtML
tv.blog.vgdrev.cn/Article/details/395551.sHtML
tv.blog.vgdrev.cn/Article/details/777953.sHtML
tv.blog.vgdrev.cn/Article/details/991955.sHtML
tv.blog.vgdrev.cn/Article/details/993719.sHtML
tv.blog.vgdrev.cn/Article/details/597533.sHtML
tv.blog.vgdrev.cn/Article/details/713797.sHtML
tv.blog.vgdrev.cn/Article/details/933575.sHtML
tv.blog.vgdrev.cn/Article/details/393591.sHtML
tv.blog.vgdrev.cn/Article/details/959377.sHtML
tv.blog.vgdrev.cn/Article/details/153333.sHtML
tv.blog.vgdrev.cn/Article/details/773397.sHtML
tv.blog.vgdrev.cn/Article/details/913391.sHtML
tv.blog.vgdrev.cn/Article/details/931597.sHtML
tv.blog.vgdrev.cn/Article/details/977919.sHtML
tv.blog.vgdrev.cn/Article/details/735137.sHtML
tv.blog.vgdrev.cn/Article/details/977159.sHtML
tv.blog.vgdrev.cn/Article/details/777155.sHtML
tv.blog.vgdrev.cn/Article/details/137353.sHtML
tv.blog.vgdrev.cn/Article/details/757337.sHtML
tv.blog.vgdrev.cn/Article/details/377175.sHtML
tv.blog.vgdrev.cn/Article/details/511775.sHtML
tv.blog.vgdrev.cn/Article/details/593375.sHtML
tv.blog.vgdrev.cn/Article/details/913515.sHtML
tv.blog.vgdrev.cn/Article/details/597119.sHtML
tv.blog.vgdrev.cn/Article/details/111171.sHtML
tv.blog.vgdrev.cn/Article/details/593139.sHtML
