匝恼幢什摆


Linux xfs_inode与xfs_inode_item日志锁定提交

xfs_inode是XFS文件系统的核心inode数据结构，xfs_inode_item是其在日志系统中的对应日志项。两者的交互构成了XFS日志记录和恢复的基础。xfs_inode_item负责将inode的修改记录到journal中，确保元数据操作的原子性。该实现位于fs/xfs/xfs_inode_item.c和fs/xfs/xfs_inode.h中。

xfs_inode的结构体包含inode的核心元数据以及用于日志跟踪的字段：

struct xfs_inode {
    struct xfs_inode_log_item *i_itemp;
    struct xfs_mount   *i_mount;
    struct xfs_dinode   i_d;
    struct xfs_ifork    i_df;
    struct xfs_ifork    *i_afp;
    struct xfs_ifork    *i_cowfp;
    spinlock_t          i_flags_lock;
    uint64_t            i_flags;
    xfs_ino_t           i_ino;
    struct xfs_perag    *i_pag;
    struct rw_semaphore i_rwsem;
    struct list_head    i_ilist;
};

xfs_inode_log_item使用xfs_inode_log_format来描述日志中的inode数据格式：

struct xfs_inode_log_item {
    struct xfs_log_item     ili_item;
    struct xfs_inode        *ili_inode;
    unsigned short          ili_flags;
    unsigned short          ili_logged;
    unsigned int            ili_last_fields;
    struct xfs_buf          *ili_buf;
    struct list_head        ili_ail_link;
};

xfs_inode_item的核心操作是iop_format和iop_lock。xfs_inode_item_format将inode数据写入日志的向量列表：

static void xfs_inode_item_format(
    struct xfs_log_item   *lip,
    struct xfs_log_vec    *lv)
{
    struct xfs_inode_log_item *iip = INODE_ITEM(lip);
    struct xfs_inode      *ip = iip->ili_inode;
    struct xfs_inode_log_format *ilf;
    unsigned int           flags = 0;

    ilf = xlog_prepare_iovec(lv, &vecp, sizeof(*ilf));
    ilf->ilf_type = XFS_LI_INODE;
    ilf->ilf_ino = ip->i_ino;
    ilf->ilf_blkno = ip->i_imap.im_blkno;
    ilf->ilf_len = 1;
    ilf->ilf_boffset = ip->i_imap.im_boffset;

    if (iip->ili_fields & XFS_ILOG_CORE) {
        flags |= XFS_ILOG_CORE;
        xlog_copy_iovec(lv, &vecp, XLOG_REG_TYPE_INODE_CORE,
                &ip->i_d, sizeof(struct xfs_dinode));
    }

    if (iip->ili_fields & XFS_ILOG_DATA) {
        flags |= XFS_ILOG_DATA;
        xfs_inode_item_format_data_fork(iip, lv, &vecp);
    }

    if (iip->ili_fields & XFS_ILOG_ATTR) {
        flags |= XFS_ILOG_ATTR;
        xfs_inode_item_format_attr_fork(iip, lv, &vecp);
    }

    ilf->ilf_fields = flags;
    iip->ili_last_fields = iip->ili_fields;
    iip->ili_fields = 0;
}

日志锁定通过xfs_inode_item_iop_lock实现，确保在事务提交期间inode不被并发修改：

static void xfs_inode_item_iop_lock(
    struct xfs_log_item   *lip)
{
    struct xfs_inode_log_item *iip = INODE_ITEM(lip);
    struct xfs_inode      *ip = iip->ili_inode;
    down_write(&ip->i_rwsem);
}

static void xfs_inode_item_iop_unlock(
    struct xfs_log_item   *lip)
{
    struct xfs_inode_log_item *iip = INODE_ITEM(lip);
    struct xfs_inode      *ip = iip->ili_inode;
    up_write(&ip->i_rwsem);
}

xfs_inode_item在事务提交中的完整生命周期为：xfs_trans_log_inode标记inode的脏字段，xfs_trans_commit调用xfs_inode_item_format生成日志向量，xlog_write将向量写入日志循环缓冲区，最后xfs_buf_item_committed将buffer标记为脏。

xfs_trans_log_inode用于在事务中标记inode的哪些字段被修改：

void xfs_trans_log_inode(
    struct xfs_trans    *tp,
    struct xfs_inode    *ip,
    uint                flags)
{
    struct xfs_inode_log_item *iip = ip->i_itemp;

    ASSERT(ip->i_transp == tp);
    ASSERT(xfs_isilocked(ip, XFS_ILOCK_EXCL));

    tp->t_flags |= XFS_TRANS_DIRTY;
    iip->ili_flags |= XFS_ILI_LOGGED;
    iip->ili_fields |= flags;

    spin_lock(&iip->ili_lock);
    if (!(iip->ili_item.li_flags & XFS_LI_DIRTY)) {
        iip->ili_item.li_flags |= XFS_LI_DIRTY;
        xfs_trans_add_item(tp, &iip->ili_item);
    }
    spin_unlock(&iip->ili_lock);
}

日志的提交和同步通过xfs_log_commit完成，将日志向量写入磁盘：

int xfs_log_commit(
    struct xfs_cil_ctx   *ctx,
    struct xfs_log_vec   *lv,
    xfs_lsn_t            *commit_lsn,
    int                  flags)
{
    struct xlog          *log = ctx->xc_log;
    struct xlog_ticket   *tic = ctx->xc_ticket;
    int                  error;

    if (xlog_space_left(log, &log->l_reserve_head.grant) < 0) {
        xlog_grant_head_wait(log, &log->l_reserve_head.grant, tic);
    }

    error = xlog_write(log, lv, tic, &log->l_op_head, XLOG_COMMIT_TRANS);
    if (error)
        goto out_abort;

    *commit_lsn = be64_to_cpu(log->l_curr_cycle) |
              (uint64_t)log->l_curr_block;

    if (flags & XFS_LOG_SYNC)
        xlog_wait_on_iclog(log->l_iclog);

    return 0;
}

日志恢复时通过xfs_inode_item_recover函数重放inode修改：

STATIC int xfs_inode_item_recover(
    struct xfs_log_item  *lip,
    struct list_head     *buffer_list)
{
    struct xfs_inode_log_item *iip = INODE_ITEM(lip);
    struct xfs_dinode    *dip;
    struct xfs_buf       *bp;
    int                  error;

    bp = xfs_buf_read(mp->m_ddev_targp, iip->ili_format.ilf_blkno,
                  1, 0);
    if (!bp)
        return -EIO;

    dip = xfs_buf_offset(bp, iip->ili_format.ilf_boffset);

    if (be16_to_cpu(dip->di_core.di_flushiter) <
        be16_to_cpu(iip->ili_format.ilf_fields)) {
        if (iip->ili_format.ilf_fields & XFS_ILOG_CORE)
            *dip = ip->i_d;
    }

    xfs_buf_mark_dirty(bp);
    xfs_buf_relse(bp);
    return 0;
}

xfs_inode与xfs_inode_item的锁定层级遵循严格规则：先获取ILOCK后访问inode。xfs_ilock的IXLOD_EXCL锁确保修改inode时没有其他线程在读或写该inode。日志子系统在提交时通过iop_lock获取ILOCK，保证了日志格式化和数据修改之间的有序性。AIL遍历和inode回收路径中的xfs_inode_item_push通过检查ili_flags中的XFS_ILI_LOGGED标志来判断inode是否仍在日志提交中，以决定回收前是否需要等待日志提交完成。flushiter字段用于日志恢复时的版本比较，确保不会用旧的日志数据覆盖新的inode修改。当flushiter回绕时，XFS使用日志序列号（LSN）进行额外的版本判断，避免回绕导致的误覆盖。
孛谓觅众惭纪褐掀记土徊桶疾判淮

tv.blog.yqvyet.cn/Article/details/511795.sHtML
tv.blog.yqvyet.cn/Article/details/515371.sHtML
tv.blog.yqvyet.cn/Article/details/971337.sHtML
tv.blog.yqvyet.cn/Article/details/715515.sHtML
tv.blog.yqvyet.cn/Article/details/333393.sHtML
tv.blog.yqvyet.cn/Article/details/795775.sHtML
tv.blog.yqvyet.cn/Article/details/753731.sHtML
tv.blog.yqvyet.cn/Article/details/753313.sHtML
tv.blog.yqvyet.cn/Article/details/313757.sHtML
tv.blog.yqvyet.cn/Article/details/591535.sHtML
tv.blog.yqvyet.cn/Article/details/371339.sHtML
tv.blog.yqvyet.cn/Article/details/331375.sHtML
tv.blog.yqvyet.cn/Article/details/177133.sHtML
tv.blog.yqvyet.cn/Article/details/319775.sHtML
tv.blog.yqvyet.cn/Article/details/775593.sHtML
tv.blog.yqvyet.cn/Article/details/351535.sHtML
tv.blog.yqvyet.cn/Article/details/755975.sHtML
tv.blog.yqvyet.cn/Article/details/799739.sHtML
tv.blog.yqvyet.cn/Article/details/977175.sHtML
tv.blog.yqvyet.cn/Article/details/339797.sHtML
tv.blog.yqvyet.cn/Article/details/913599.sHtML
tv.blog.yqvyet.cn/Article/details/759195.sHtML
tv.blog.yqvyet.cn/Article/details/799771.sHtML
tv.blog.yqvyet.cn/Article/details/171155.sHtML
tv.blog.yqvyet.cn/Article/details/133139.sHtML
tv.blog.yqvyet.cn/Article/details/317315.sHtML
tv.blog.yqvyet.cn/Article/details/597131.sHtML
tv.blog.yqvyet.cn/Article/details/991555.sHtML
tv.blog.yqvyet.cn/Article/details/573777.sHtML
tv.blog.yqvyet.cn/Article/details/337555.sHtML
tv.blog.yqvyet.cn/Article/details/179177.sHtML
tv.blog.yqvyet.cn/Article/details/519391.sHtML
tv.blog.yqvyet.cn/Article/details/711751.sHtML
tv.blog.yqvyet.cn/Article/details/135131.sHtML
tv.blog.yqvyet.cn/Article/details/775737.sHtML
tv.blog.yqvyet.cn/Article/details/599933.sHtML
tv.blog.yqvyet.cn/Article/details/553775.sHtML
tv.blog.yqvyet.cn/Article/details/539753.sHtML
tv.blog.yqvyet.cn/Article/details/391335.sHtML
tv.blog.yqvyet.cn/Article/details/793595.sHtML
tv.blog.yqvyet.cn/Article/details/551957.sHtML
tv.blog.yqvyet.cn/Article/details/331713.sHtML
tv.blog.yqvyet.cn/Article/details/553311.sHtML
tv.blog.yqvyet.cn/Article/details/357573.sHtML
tv.blog.yqvyet.cn/Article/details/199177.sHtML
tv.blog.yqvyet.cn/Article/details/711377.sHtML
tv.blog.yqvyet.cn/Article/details/535113.sHtML
tv.blog.yqvyet.cn/Article/details/119115.sHtML
tv.blog.yqvyet.cn/Article/details/951315.sHtML
tv.blog.yqvyet.cn/Article/details/315113.sHtML
tv.blog.yqvyet.cn/Article/details/951579.sHtML
tv.blog.yqvyet.cn/Article/details/791755.sHtML
tv.blog.yqvyet.cn/Article/details/513917.sHtML
tv.blog.yqvyet.cn/Article/details/171515.sHtML
tv.blog.yqvyet.cn/Article/details/559171.sHtML
tv.blog.yqvyet.cn/Article/details/315775.sHtML
tv.blog.yqvyet.cn/Article/details/993371.sHtML
tv.blog.yqvyet.cn/Article/details/757759.sHtML
tv.blog.yqvyet.cn/Article/details/579171.sHtML
tv.blog.yqvyet.cn/Article/details/959399.sHtML
tv.blog.yqvyet.cn/Article/details/177991.sHtML
tv.blog.yqvyet.cn/Article/details/191739.sHtML
tv.blog.yqvyet.cn/Article/details/557115.sHtML
tv.blog.yqvyet.cn/Article/details/319755.sHtML
tv.blog.yqvyet.cn/Article/details/111591.sHtML
tv.blog.yqvyet.cn/Article/details/915995.sHtML
tv.blog.yqvyet.cn/Article/details/739979.sHtML
tv.blog.yqvyet.cn/Article/details/917535.sHtML
tv.blog.yqvyet.cn/Article/details/313979.sHtML
tv.blog.yqvyet.cn/Article/details/755931.sHtML
tv.blog.yqvyet.cn/Article/details/793577.sHtML
tv.blog.yqvyet.cn/Article/details/131975.sHtML
tv.blog.yqvyet.cn/Article/details/791713.sHtML
tv.blog.yqvyet.cn/Article/details/755975.sHtML
tv.blog.yqvyet.cn/Article/details/939391.sHtML
tv.blog.yqvyet.cn/Article/details/933331.sHtML
tv.blog.yqvyet.cn/Article/details/195935.sHtML
tv.blog.yqvyet.cn/Article/details/175357.sHtML
tv.blog.yqvyet.cn/Article/details/591175.sHtML
tv.blog.yqvyet.cn/Article/details/573575.sHtML
tv.blog.yqvyet.cn/Article/details/991555.sHtML
tv.blog.yqvyet.cn/Article/details/793111.sHtML
tv.blog.yqvyet.cn/Article/details/553133.sHtML
tv.blog.yqvyet.cn/Article/details/975957.sHtML
tv.blog.yqvyet.cn/Article/details/717753.sHtML
tv.blog.yqvyet.cn/Article/details/355799.sHtML
tv.blog.yqvyet.cn/Article/details/355757.sHtML
tv.blog.yqvyet.cn/Article/details/735139.sHtML
tv.blog.yqvyet.cn/Article/details/935775.sHtML
tv.blog.yqvyet.cn/Article/details/131939.sHtML
tv.blog.yqvyet.cn/Article/details/357915.sHtML
tv.blog.yqvyet.cn/Article/details/139957.sHtML
tv.blog.yqvyet.cn/Article/details/995179.sHtML
tv.blog.yqvyet.cn/Article/details/351353.sHtML
tv.blog.yqvyet.cn/Article/details/319375.sHtML
tv.blog.yqvyet.cn/Article/details/937979.sHtML
tv.blog.yqvyet.cn/Article/details/513593.sHtML
tv.blog.yqvyet.cn/Article/details/311755.sHtML
tv.blog.yqvyet.cn/Article/details/717791.sHtML
tv.blog.yqvyet.cn/Article/details/353937.sHtML
tv.blog.yqvyet.cn/Article/details/359319.sHtML
tv.blog.yqvyet.cn/Article/details/313599.sHtML
tv.blog.yqvyet.cn/Article/details/319775.sHtML
tv.blog.yqvyet.cn/Article/details/917311.sHtML
tv.blog.yqvyet.cn/Article/details/111751.sHtML
tv.blog.yqvyet.cn/Article/details/919715.sHtML
tv.blog.yqvyet.cn/Article/details/395939.sHtML
tv.blog.yqvyet.cn/Article/details/375931.sHtML
tv.blog.yqvyet.cn/Article/details/137115.sHtML
tv.blog.yqvyet.cn/Article/details/591713.sHtML
tv.blog.yqvyet.cn/Article/details/771313.sHtML
tv.blog.yqvyet.cn/Article/details/179939.sHtML
tv.blog.yqvyet.cn/Article/details/335799.sHtML
tv.blog.yqvyet.cn/Article/details/359953.sHtML
tv.blog.yqvyet.cn/Article/details/171173.sHtML
tv.blog.yqvyet.cn/Article/details/199177.sHtML
tv.blog.yqvyet.cn/Article/details/153313.sHtML
tv.blog.yqvyet.cn/Article/details/995575.sHtML
tv.blog.yqvyet.cn/Article/details/997779.sHtML
tv.blog.yqvyet.cn/Article/details/551753.sHtML
tv.blog.yqvyet.cn/Article/details/171737.sHtML
tv.blog.yqvyet.cn/Article/details/535957.sHtML
tv.blog.yqvyet.cn/Article/details/751199.sHtML
tv.blog.yqvyet.cn/Article/details/515779.sHtML
tv.blog.yqvyet.cn/Article/details/117139.sHtML
tv.blog.yqvyet.cn/Article/details/319391.sHtML
tv.blog.yqvyet.cn/Article/details/779535.sHtML
tv.blog.yqvyet.cn/Article/details/375395.sHtML
tv.blog.yqvyet.cn/Article/details/537515.sHtML
tv.blog.yqvyet.cn/Article/details/513933.sHtML
tv.blog.yqvyet.cn/Article/details/915775.sHtML
tv.blog.yqvyet.cn/Article/details/591917.sHtML
tv.blog.yqvyet.cn/Article/details/157559.sHtML
tv.blog.yqvyet.cn/Article/details/537993.sHtML
tv.blog.yqvyet.cn/Article/details/719395.sHtML
tv.blog.yqvyet.cn/Article/details/751393.sHtML
tv.blog.yqvyet.cn/Article/details/533313.sHtML
tv.blog.yqvyet.cn/Article/details/337511.sHtML
tv.blog.yqvyet.cn/Article/details/779777.sHtML
tv.blog.yqvyet.cn/Article/details/131917.sHtML
tv.blog.yqvyet.cn/Article/details/579533.sHtML
tv.blog.yqvyet.cn/Article/details/953131.sHtML
tv.blog.yqvyet.cn/Article/details/713977.sHtML
tv.blog.yqvyet.cn/Article/details/957511.sHtML
tv.blog.yqvyet.cn/Article/details/393393.sHtML
tv.blog.yqvyet.cn/Article/details/179357.sHtML
tv.blog.yqvyet.cn/Article/details/335151.sHtML
tv.blog.yqvyet.cn/Article/details/313135.sHtML
tv.blog.yqvyet.cn/Article/details/713131.sHtML
tv.blog.yqvyet.cn/Article/details/931755.sHtML
tv.blog.yqvyet.cn/Article/details/597953.sHtML
tv.blog.yqvyet.cn/Article/details/591919.sHtML
tv.blog.yqvyet.cn/Article/details/975517.sHtML
tv.blog.yqvyet.cn/Article/details/195931.sHtML
tv.blog.yqvyet.cn/Article/details/315311.sHtML
tv.blog.yqvyet.cn/Article/details/791371.sHtML
tv.blog.yqvyet.cn/Article/details/357159.sHtML
tv.blog.yqvyet.cn/Article/details/373371.sHtML
tv.blog.yqvyet.cn/Article/details/795775.sHtML
tv.blog.yqvyet.cn/Article/details/131553.sHtML
tv.blog.yqvyet.cn/Article/details/313313.sHtML
tv.blog.yqvyet.cn/Article/details/557557.sHtML
tv.blog.yqvyet.cn/Article/details/393155.sHtML
tv.blog.yqvyet.cn/Article/details/971573.sHtML
tv.blog.yqvyet.cn/Article/details/173731.sHtML
tv.blog.yqvyet.cn/Article/details/577333.sHtML
tv.blog.yqvyet.cn/Article/details/973539.sHtML
tv.blog.yqvyet.cn/Article/details/199353.sHtML
tv.blog.yqvyet.cn/Article/details/777579.sHtML
tv.blog.yqvyet.cn/Article/details/775317.sHtML
tv.blog.yqvyet.cn/Article/details/195999.sHtML
tv.blog.yqvyet.cn/Article/details/317513.sHtML
tv.blog.yqvyet.cn/Article/details/197959.sHtML
tv.blog.yqvyet.cn/Article/details/359755.sHtML
tv.blog.yqvyet.cn/Article/details/931597.sHtML
tv.blog.yqvyet.cn/Article/details/537715.sHtML
tv.blog.yqvyet.cn/Article/details/711537.sHtML
tv.blog.yqvyet.cn/Article/details/937135.sHtML
tv.blog.yqvyet.cn/Article/details/559319.sHtML
tv.blog.yqvyet.cn/Article/details/357779.sHtML
tv.blog.yqvyet.cn/Article/details/513333.sHtML
tv.blog.yqvyet.cn/Article/details/191995.sHtML
tv.blog.yqvyet.cn/Article/details/331535.sHtML
tv.blog.yqvyet.cn/Article/details/159931.sHtML
tv.blog.yqvyet.cn/Article/details/557511.sHtML
tv.blog.yqvyet.cn/Article/details/199999.sHtML
tv.blog.yqvyet.cn/Article/details/395577.sHtML
tv.blog.yqvyet.cn/Article/details/791595.sHtML
tv.blog.yqvyet.cn/Article/details/117999.sHtML
tv.blog.yqvyet.cn/Article/details/355371.sHtML
tv.blog.yqvyet.cn/Article/details/755373.sHtML
tv.blog.yqvyet.cn/Article/details/711755.sHtML
tv.blog.yqvyet.cn/Article/details/571111.sHtML
tv.blog.yqvyet.cn/Article/details/313315.sHtML
tv.blog.yqvyet.cn/Article/details/739557.sHtML
tv.blog.yqvyet.cn/Article/details/757153.sHtML
tv.blog.yqvyet.cn/Article/details/717337.sHtML
tv.blog.yqvyet.cn/Article/details/999395.sHtML
tv.blog.yqvyet.cn/Article/details/959917.sHtML
tv.blog.yqvyet.cn/Article/details/779131.sHtML
tv.blog.yqvyet.cn/Article/details/559799.sHtML
tv.blog.yqvyet.cn/Article/details/173755.sHtML
tv.blog.yqvyet.cn/Article/details/711957.sHtML
tv.blog.yqvyet.cn/Article/details/937513.sHtML
tv.blog.yqvyet.cn/Article/details/177551.sHtML
tv.blog.yqvyet.cn/Article/details/199971.sHtML
tv.blog.yqvyet.cn/Article/details/159975.sHtML
tv.blog.yqvyet.cn/Article/details/359397.sHtML
tv.blog.yqvyet.cn/Article/details/513777.sHtML
tv.blog.yqvyet.cn/Article/details/597551.sHtML
tv.blog.yqvyet.cn/Article/details/397319.sHtML
tv.blog.yqvyet.cn/Article/details/355773.sHtML
tv.blog.yqvyet.cn/Article/details/553153.sHtML
tv.blog.yqvyet.cn/Article/details/577973.sHtML
tv.blog.yqvyet.cn/Article/details/151557.sHtML
tv.blog.yqvyet.cn/Article/details/593197.sHtML
tv.blog.yqvyet.cn/Article/details/335919.sHtML
tv.blog.yqvyet.cn/Article/details/999155.sHtML
tv.blog.yqvyet.cn/Article/details/193537.sHtML
tv.blog.yqvyet.cn/Article/details/571713.sHtML
tv.blog.yqvyet.cn/Article/details/535171.sHtML
tv.blog.yqvyet.cn/Article/details/793795.sHtML
tv.blog.yqvyet.cn/Article/details/119973.sHtML
tv.blog.yqvyet.cn/Article/details/551113.sHtML
tv.blog.yqvyet.cn/Article/details/539779.sHtML
tv.blog.yqvyet.cn/Article/details/795595.sHtML
tv.blog.yqvyet.cn/Article/details/719191.sHtML
tv.blog.yqvyet.cn/Article/details/939133.sHtML
tv.blog.yqvyet.cn/Article/details/139131.sHtML
tv.blog.yqvyet.cn/Article/details/111591.sHtML
tv.blog.yqvyet.cn/Article/details/715537.sHtML
tv.blog.yqvyet.cn/Article/details/157539.sHtML
tv.blog.yqvyet.cn/Article/details/373559.sHtML
tv.blog.yqvyet.cn/Article/details/131377.sHtML
tv.blog.yqvyet.cn/Article/details/355357.sHtML
tv.blog.yqvyet.cn/Article/details/315137.sHtML
tv.blog.yqvyet.cn/Article/details/157731.sHtML
tv.blog.yqvyet.cn/Article/details/391553.sHtML
tv.blog.yqvyet.cn/Article/details/337111.sHtML
tv.blog.yqvyet.cn/Article/details/519593.sHtML
tv.blog.yqvyet.cn/Article/details/779195.sHtML
tv.blog.yqvyet.cn/Article/details/731593.sHtML
tv.blog.yqvyet.cn/Article/details/933153.sHtML
tv.blog.yqvyet.cn/Article/details/579735.sHtML
tv.blog.yqvyet.cn/Article/details/537959.sHtML
tv.blog.yqvyet.cn/Article/details/319535.sHtML
tv.blog.yqvyet.cn/Article/details/975179.sHtML
tv.blog.yqvyet.cn/Article/details/115979.sHtML
tv.blog.yqvyet.cn/Article/details/979399.sHtML
tv.blog.yqvyet.cn/Article/details/993759.sHtML
tv.blog.yqvyet.cn/Article/details/939773.sHtML
tv.blog.yqvyet.cn/Article/details/997179.sHtML
tv.blog.yqvyet.cn/Article/details/911173.sHtML
tv.blog.yqvyet.cn/Article/details/155713.sHtML
tv.blog.yqvyet.cn/Article/details/951979.sHtML
tv.blog.yqvyet.cn/Article/details/177533.sHtML
tv.blog.yqvyet.cn/Article/details/775599.sHtML
tv.blog.yqvyet.cn/Article/details/591957.sHtML
tv.blog.yqvyet.cn/Article/details/571355.sHtML
tv.blog.yqvyet.cn/Article/details/791555.sHtML
tv.blog.yqvyet.cn/Article/details/317739.sHtML
tv.blog.yqvyet.cn/Article/details/755179.sHtML
tv.blog.yqvyet.cn/Article/details/777591.sHtML
tv.blog.yqvyet.cn/Article/details/757353.sHtML
tv.blog.yqvyet.cn/Article/details/579931.sHtML
tv.blog.yqvyet.cn/Article/details/779959.sHtML
tv.blog.yqvyet.cn/Article/details/971337.sHtML
tv.blog.yqvyet.cn/Article/details/175715.sHtML
tv.blog.yqvyet.cn/Article/details/153177.sHtML
tv.blog.yqvyet.cn/Article/details/571373.sHtML
tv.blog.yqvyet.cn/Article/details/537755.sHtML
tv.blog.yqvyet.cn/Article/details/395777.sHtML
tv.blog.yqvyet.cn/Article/details/793913.sHtML
tv.blog.yqvyet.cn/Article/details/959331.sHtML
tv.blog.yqvyet.cn/Article/details/933515.sHtML
tv.blog.yqvyet.cn/Article/details/775175.sHtML
tv.blog.yqvyet.cn/Article/details/911551.sHtML
tv.blog.yqvyet.cn/Article/details/733513.sHtML
tv.blog.yqvyet.cn/Article/details/999999.sHtML
tv.blog.yqvyet.cn/Article/details/197939.sHtML
tv.blog.yqvyet.cn/Article/details/533333.sHtML
tv.blog.yqvyet.cn/Article/details/119999.sHtML
tv.blog.yqvyet.cn/Article/details/591199.sHtML
tv.blog.yqvyet.cn/Article/details/795171.sHtML
tv.blog.yqvyet.cn/Article/details/111779.sHtML
tv.blog.yqvyet.cn/Article/details/731337.sHtML
tv.blog.yqvyet.cn/Article/details/715517.sHtML
tv.blog.yqvyet.cn/Article/details/355919.sHtML
tv.blog.yqvyet.cn/Article/details/997531.sHtML
tv.blog.yqvyet.cn/Article/details/933957.sHtML
tv.blog.yqvyet.cn/Article/details/199399.sHtML
tv.blog.yqvyet.cn/Article/details/159955.sHtML
tv.blog.yqvyet.cn/Article/details/599935.sHtML
tv.blog.yqvyet.cn/Article/details/193511.sHtML
tv.blog.yqvyet.cn/Article/details/397135.sHtML
tv.blog.yqvyet.cn/Article/details/373777.sHtML
tv.blog.yqvyet.cn/Article/details/757157.sHtML
tv.blog.yqvyet.cn/Article/details/115793.sHtML
tv.blog.yqvyet.cn/Article/details/799533.sHtML
tv.blog.yqvyet.cn/Article/details/751135.sHtML
tv.blog.yqvyet.cn/Article/details/973533.sHtML
tv.blog.yqvyet.cn/Article/details/531559.sHtML
tv.blog.yqvyet.cn/Article/details/735513.sHtML
tv.blog.yqvyet.cn/Article/details/539791.sHtML
tv.blog.yqvyet.cn/Article/details/577975.sHtML
tv.blog.yqvyet.cn/Article/details/193193.sHtML
tv.blog.yqvyet.cn/Article/details/337371.sHtML
tv.blog.yqvyet.cn/Article/details/955115.sHtML
tv.blog.yqvyet.cn/Article/details/593733.sHtML
tv.blog.yqvyet.cn/Article/details/713193.sHtML
tv.blog.yqvyet.cn/Article/details/551539.sHtML
tv.blog.yqvyet.cn/Article/details/575731.sHtML
tv.blog.yqvyet.cn/Article/details/935553.sHtML
tv.blog.yqvyet.cn/Article/details/375771.sHtML
tv.blog.yqvyet.cn/Article/details/173793.sHtML
tv.blog.yqvyet.cn/Article/details/791111.sHtML
tv.blog.yqvyet.cn/Article/details/771391.sHtML
tv.blog.yqvyet.cn/Article/details/173539.sHtML
tv.blog.yqvyet.cn/Article/details/191315.sHtML
tv.blog.yqvyet.cn/Article/details/313557.sHtML
tv.blog.yqvyet.cn/Article/details/319577.sHtML
tv.blog.yqvyet.cn/Article/details/939371.sHtML
tv.blog.yqvyet.cn/Article/details/937197.sHtML
tv.blog.yqvyet.cn/Article/details/737919.sHtML
tv.blog.yqvyet.cn/Article/details/371515.sHtML
tv.blog.yqvyet.cn/Article/details/317355.sHtML
tv.blog.yqvyet.cn/Article/details/133117.sHtML
tv.blog.yqvyet.cn/Article/details/999539.sHtML
tv.blog.yqvyet.cn/Article/details/133377.sHtML
tv.blog.yqvyet.cn/Article/details/937779.sHtML
tv.blog.yqvyet.cn/Article/details/715557.sHtML
tv.blog.yqvyet.cn/Article/details/119359.sHtML
tv.blog.yqvyet.cn/Article/details/311177.sHtML
tv.blog.yqvyet.cn/Article/details/971999.sHtML
tv.blog.yqvyet.cn/Article/details/353591.sHtML
tv.blog.yqvyet.cn/Article/details/371717.sHtML
tv.blog.yqvyet.cn/Article/details/957733.sHtML
tv.blog.yqvyet.cn/Article/details/511195.sHtML
tv.blog.yqvyet.cn/Article/details/937135.sHtML
tv.blog.yqvyet.cn/Article/details/573953.sHtML
tv.blog.yqvyet.cn/Article/details/571371.sHtML
tv.blog.yqvyet.cn/Article/details/571731.sHtML
tv.blog.yqvyet.cn/Article/details/931731.sHtML
tv.blog.yqvyet.cn/Article/details/193117.sHtML
tv.blog.yqvyet.cn/Article/details/919117.sHtML
tv.blog.yqvyet.cn/Article/details/333519.sHtML
tv.blog.yqvyet.cn/Article/details/335915.sHtML
tv.blog.yqvyet.cn/Article/details/371775.sHtML
tv.blog.yqvyet.cn/Article/details/773339.sHtML
tv.blog.yqvyet.cn/Article/details/113779.sHtML
tv.blog.yqvyet.cn/Article/details/199113.sHtML
tv.blog.yqvyet.cn/Article/details/379175.sHtML
tv.blog.yqvyet.cn/Article/details/139515.sHtML
tv.blog.yqvyet.cn/Article/details/559177.sHtML
tv.blog.yqvyet.cn/Article/details/153191.sHtML
tv.blog.yqvyet.cn/Article/details/393977.sHtML
tv.blog.yqvyet.cn/Article/details/911931.sHtML
tv.blog.yqvyet.cn/Article/details/757511.sHtML
tv.blog.yqvyet.cn/Article/details/535397.sHtML
tv.blog.yqvyet.cn/Article/details/911359.sHtML
tv.blog.yqvyet.cn/Article/details/575375.sHtML
tv.blog.yqvyet.cn/Article/details/119353.sHtML
tv.blog.yqvyet.cn/Article/details/917179.sHtML
tv.blog.yqvyet.cn/Article/details/737993.sHtML
tv.blog.yqvyet.cn/Article/details/131715.sHtML
tv.blog.yqvyet.cn/Article/details/735593.sHtML
tv.blog.yqvyet.cn/Article/details/795577.sHtML
tv.blog.yqvyet.cn/Article/details/973997.sHtML
tv.blog.yqvyet.cn/Article/details/711733.sHtML
tv.blog.yqvyet.cn/Article/details/951199.sHtML
tv.blog.yqvyet.cn/Article/details/135157.sHtML
tv.blog.yqvyet.cn/Article/details/951317.sHtML
tv.blog.yqvyet.cn/Article/details/931793.sHtML
tv.blog.yqvyet.cn/Article/details/715593.sHtML
tv.blog.yqvyet.cn/Article/details/599577.sHtML
tv.blog.yqvyet.cn/Article/details/519371.sHtML
tv.blog.yqvyet.cn/Article/details/775959.sHtML
tv.blog.yqvyet.cn/Article/details/199355.sHtML
tv.blog.yqvyet.cn/Article/details/993335.sHtML
tv.blog.yqvyet.cn/Article/details/715939.sHtML
tv.blog.yqvyet.cn/Article/details/171997.sHtML
tv.blog.yqvyet.cn/Article/details/131197.sHtML
tv.blog.yqvyet.cn/Article/details/999739.sHtML
tv.blog.yqvyet.cn/Article/details/799999.sHtML
tv.blog.yqvyet.cn/Article/details/951513.sHtML
tv.blog.yqvyet.cn/Article/details/719713.sHtML
tv.blog.yqvyet.cn/Article/details/971139.sHtML
tv.blog.yqvyet.cn/Article/details/793571.sHtML
tv.blog.yqvyet.cn/Article/details/775931.sHtML
tv.blog.yqvyet.cn/Article/details/313775.sHtML
tv.blog.yqvyet.cn/Article/details/995599.sHtML
tv.blog.yqvyet.cn/Article/details/155113.sHtML
tv.blog.yqvyet.cn/Article/details/137773.sHtML
tv.blog.yqvyet.cn/Article/details/511937.sHtML
tv.blog.yqvyet.cn/Article/details/373339.sHtML
