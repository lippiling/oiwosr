纪诚谑驴关


Linux xfs_buf元数据缓冲区管理xfs_buf_tlock锁定

xfs_buf是XFS文件系统元数据缓存的核心数据结构，类似于ext4的buffer_head但设计更加复杂。xfs_buf_tlock是xfs_buf的关键锁定接口，提供对缓冲区加锁以确保元数据操作的原子性。该接口定义在fs/xfs/xfs_buf.h和fs/xfs/xfs_buf.c中。

xfs_buf的结构体承载了页缓存管理、日志标记、IO完成回调等多项功能：

struct xfs_buf {
    struct xfs_buftarg   *b_target;
    struct list_head     b_list;
    struct xfs_perag     *b_pag;
    xfs_daddr_t          b_bn;
    size_t               b_length;
    atomic_t             b_hold;
    atomic_t             b_lru_ref;
    wait_queue_head_t    b_waiters;
    struct completion    b_iowait;
    struct list_head     b_li_list;
    spinlock_t           b_lock;
    struct semaphore     b_sema;
    struct xfs_buf_log_item *b_log_item;
    struct page          **b_pages;
    int                  b_flags;
    int                  b_error;
    const struct xfs_buf_ops *b_ops;
};

xfs_buf_tlock用于对buffer进行排他性锁定，其核心实现基于b_sema信号量。区别于xfs_buf_lock（阻塞等待），xfs_buf_tlock可以在已持有锁的情况下尝试锁定：

void xfs_buf_tlock(struct xfs_buf *bp)
{
    if (atomic_read(&bp->b_sema.count) <= 0) {
        if (down_trylock(&bp->b_sema) != 0) {
            if (bp->b_lock_walks == 0) {
                XFS_STATS_INC(bp->b_mount, xs_buf_lock_wait);
                down(&bp->b_sema);
                XFS_STATS_INC(bp->b_mount, xs_buf_lock_retry);
            }
        }
    }
    bp->b_lock_walks++;
    bp->b_flags |= XBF_WRITE_LOCK;
}

xfs_buf_tlock的核心应用场景在xfs_buf_log_item的处理过程中。当日志提交需要锁定缓冲区时，xfs_buf_item_lock被调用，最终调用xfs_buf_tlock确保缓冲区在日志操作期间不被其他线程修改：

void xfs_buf_item_lock(struct xfs_buf_log_item *bip)
{
    struct xfs_buf *bp = bip->bli_buf;

    ASSERT(!(bp->b_flags & XBF_WRITE_IN_PROGRESS));

    if (bip->bli_flags & XFS_BLI_STALE) {
        ASSERT(bp->b_flags & XBF_STALE);
        return;
    }

    xfs_buf_tlock(bp);
    bip->bli_flags |= XFS_BLI_LOCKED;
}

xfs_buf的解锁操作由xfs_buf_unlock执行，减少b_lock_walks计数，当计数归零时释放信号量：

void xfs_buf_unlock(struct xfs_buf *bp)
{
    ASSERT(atomic_read(&bp->b_sema.count) <= 0);
    ASSERT(bp->b_lock_walks > 0);

    if (--bp->b_lock_walks == 0) {
        bp->b_flags &= ~XBF_WRITE_LOCK;
        up(&bp->b_sema);
        wake_up_all(&bp->b_waiters);
    }
}

xfs_buf的创建和初始化通过xfs_buf_get和xfs_buf_find完成。xfs_buf_find在全局哈希表中查找指定块号是否已有缓存的buffer：

struct xfs_buf *xfs_buf_find(struct xfs_buftarg *btp, xfs_daddr_t blkno,
                   size_t numblks, int flags)
{
    struct xfs_buf *bp;
    struct xfs_buf_map map;

    map.bm_bn = blkno;
    map.bm_len = numblks;

    bp = _xfs_buf_find(btp, &map, 1, flags, NULL);
    return bp;
}

static struct xfs_buf *_xfs_buf_find(struct xfs_buftarg *btp,
                      struct xfs_buf_map *map,
                      int nmaps, int flags,
                      xfs_buf_flags_t nohugetlb)
{
    struct xfs_buf *bp;
    struct xfs_perag *pag;
    xfs_daddr_t blkno = map[0].bm_bn;
    int i;

    pag = xfs_perag_get(btp->bt_mount,
               xfs_daddr_to_agno(btp->bt_mount, blkno));
    spin_lock(&pag->pag_buf_lock);

    list_for_each_entry(bp, &pag->pag_buf_hash[hash], b_hash_list) {
        if (bp->b_bn == blkno && bp->b_length == numblks) {
            atomic_inc(&bp->b_hold);
            spin_unlock(&pag->pag_buf_lock);
            xfs_perag_put(pag);
            return bp;
        }
    }

    spin_unlock(&pag->pag_buf_lock);
    xfs_perag_put(pag);
    return xfs_buf_alloc(btp, map, nmaps, flags, nohugetlb);
}

xfs_buf的IO提交流程涉及xfs_buf_submit和xfs_buf_ioend。提交时通过submit_bio提交底层块设备IO：

int xfs_buf_submit(struct xfs_buf *bp)
{
    struct bio *bio;
    int error;

    ASSERT(bp->b_flags & XBF_READ);

    bio = bio_alloc(GFP_NOIO, bp->b_page_count);
    if (!bio)
        return -ENOMEM;

    bio->bi_iter.bi_sector = bp->b_bn;
    bio->bi_end_io = xfs_buf_ioend;
    bio->bi_private = bp;
    bio_set_dev(bio, bp->b_target->bt_bdev);

    for (int i = 0; i < bp->b_page_count; i++)
        bio_add_page(bio, bp->b_pages[i], PAGE_SIZE, 0);

    if (bp->b_flags & XBF_READ)
        bio_set_op_attrs(bio, REQ_OP_READ, 0);
    else
        bio_set_op_attrs(bio, REQ_OP_WRITE, 0);

    submit_bio(bio);
    return 0;
}

xfs_buf_tlock与事务提交的交互通过xfs_buf_item_committed实现。在事务提交阶段，xfs_buf_item_committed检查buffer是否仍然有效，如果buffer被标记为stale则释放日志item并跳过重做。tlock锁定确保buffer在提交过程中不会被eviction线程回收。对于XFS的元数据密集操作（如btree分裂），xfs_buf_tlock可能在同一个buffer上被多次调用，这是通过b_lock_walks计数器实现的递归锁定。每层代码可以独立调用xfs_buf_tlock而无需担心中间层释放锁导致的数据竞争。xfs_buf在内存回收路径中通过xfs_buf_free释放，该函数等待所有IO完成并释放页数组。在回收检查时，只有在b_hold引用计数为0且b_sema信号量可立即获取时，buffer才会被回收。xfs_buf_tlock的存在阻止了buffer在锁定状态下被意外回收。<｜end▁of▁thinking｜>

<｜｜DSML｜｜parameter name="tool_expected_additions" string="false">null
倏盘妒晒旨僮糙导认返九掠珊繁钒

tv.blog.cvvliy.cn/Article/details/220662.sHtML
tv.blog.cvvliy.cn/Article/details/404602.sHtML
tv.blog.cvvliy.cn/Article/details/226806.sHtML
tv.blog.cvvliy.cn/Article/details/993771.sHtML
tv.blog.cvvliy.cn/Article/details/260220.sHtML
tv.blog.cvvliy.cn/Article/details/466204.sHtML
tv.blog.cvvliy.cn/Article/details/024222.sHtML
tv.blog.cvvliy.cn/Article/details/719771.sHtML
tv.blog.cvvliy.cn/Article/details/846826.sHtML
tv.blog.cvvliy.cn/Article/details/175153.sHtML
tv.blog.cvvliy.cn/Article/details/022620.sHtML
tv.blog.cvvliy.cn/Article/details/068268.sHtML
tv.blog.cvvliy.cn/Article/details/157337.sHtML
tv.blog.cvvliy.cn/Article/details/200222.sHtML
tv.blog.cvvliy.cn/Article/details/404206.sHtML
tv.blog.cvvliy.cn/Article/details/442448.sHtML
tv.blog.cvvliy.cn/Article/details/519353.sHtML
tv.blog.cvvliy.cn/Article/details/737599.sHtML
tv.blog.cvvliy.cn/Article/details/004800.sHtML
tv.blog.cvvliy.cn/Article/details/971131.sHtML
tv.blog.cvvliy.cn/Article/details/595551.sHtML
tv.blog.cvvliy.cn/Article/details/446840.sHtML
tv.blog.cvvliy.cn/Article/details/640402.sHtML
tv.blog.cvvliy.cn/Article/details/884442.sHtML
tv.blog.cvvliy.cn/Article/details/004644.sHtML
tv.blog.cvvliy.cn/Article/details/462800.sHtML
tv.blog.cvvliy.cn/Article/details/579751.sHtML
tv.blog.cvvliy.cn/Article/details/113531.sHtML
tv.blog.cvvliy.cn/Article/details/866668.sHtML
tv.blog.cvvliy.cn/Article/details/795911.sHtML
tv.blog.cvvliy.cn/Article/details/377977.sHtML
tv.blog.cvvliy.cn/Article/details/868204.sHtML
tv.blog.cvvliy.cn/Article/details/795919.sHtML
tv.blog.cvvliy.cn/Article/details/735991.sHtML
tv.blog.cvvliy.cn/Article/details/133759.sHtML
tv.blog.cvvliy.cn/Article/details/202202.sHtML
tv.blog.cvvliy.cn/Article/details/119517.sHtML
tv.blog.cvvliy.cn/Article/details/840480.sHtML
tv.blog.cvvliy.cn/Article/details/553953.sHtML
tv.blog.cvvliy.cn/Article/details/844602.sHtML
tv.blog.cvvliy.cn/Article/details/820420.sHtML
tv.blog.cvvliy.cn/Article/details/448664.sHtML
tv.blog.cvvliy.cn/Article/details/426682.sHtML
tv.blog.cvvliy.cn/Article/details/404206.sHtML
tv.blog.cvvliy.cn/Article/details/262062.sHtML
tv.blog.cvvliy.cn/Article/details/442088.sHtML
tv.blog.cvvliy.cn/Article/details/888460.sHtML
tv.blog.cvvliy.cn/Article/details/862460.sHtML
tv.blog.cvvliy.cn/Article/details/642648.sHtML
tv.blog.cvvliy.cn/Article/details/117759.sHtML
tv.blog.cvvliy.cn/Article/details/084666.sHtML
tv.blog.cvvliy.cn/Article/details/775137.sHtML
tv.blog.cvvliy.cn/Article/details/028088.sHtML
tv.blog.cvvliy.cn/Article/details/397559.sHtML
tv.blog.cvvliy.cn/Article/details/484220.sHtML
tv.blog.cvvliy.cn/Article/details/711999.sHtML
tv.blog.cvvliy.cn/Article/details/119917.sHtML
tv.blog.cvvliy.cn/Article/details/428886.sHtML
tv.blog.cvvliy.cn/Article/details/644888.sHtML
tv.blog.cvvliy.cn/Article/details/028000.sHtML
tv.blog.cvvliy.cn/Article/details/606004.sHtML
tv.blog.cvvliy.cn/Article/details/533333.sHtML
tv.blog.cvvliy.cn/Article/details/993539.sHtML
tv.blog.cvvliy.cn/Article/details/846028.sHtML
tv.blog.cvvliy.cn/Article/details/848680.sHtML
tv.blog.cvvliy.cn/Article/details/060240.sHtML
tv.blog.cvvliy.cn/Article/details/426028.sHtML
tv.blog.cvvliy.cn/Article/details/684062.sHtML
tv.blog.cvvliy.cn/Article/details/060446.sHtML
tv.blog.cvvliy.cn/Article/details/315793.sHtML
tv.blog.cvvliy.cn/Article/details/731719.sHtML
tv.blog.cvvliy.cn/Article/details/577913.sHtML
tv.blog.cvvliy.cn/Article/details/826262.sHtML
tv.blog.cvvliy.cn/Article/details/739977.sHtML
tv.blog.cvvliy.cn/Article/details/337955.sHtML
tv.blog.cvvliy.cn/Article/details/400244.sHtML
tv.blog.cvvliy.cn/Article/details/666084.sHtML
tv.blog.cvvliy.cn/Article/details/042284.sHtML
tv.blog.cvvliy.cn/Article/details/208820.sHtML
tv.blog.cvvliy.cn/Article/details/262086.sHtML
tv.blog.cvvliy.cn/Article/details/155919.sHtML
tv.blog.cvvliy.cn/Article/details/208686.sHtML
tv.blog.cvvliy.cn/Article/details/482486.sHtML
tv.blog.cvvliy.cn/Article/details/240800.sHtML
tv.blog.cvvliy.cn/Article/details/715553.sHtML
tv.blog.cvvliy.cn/Article/details/860066.sHtML
tv.blog.cvvliy.cn/Article/details/808622.sHtML
tv.blog.cvvliy.cn/Article/details/248404.sHtML
tv.blog.cvvliy.cn/Article/details/153139.sHtML
tv.blog.cvvliy.cn/Article/details/662086.sHtML
tv.blog.cvvliy.cn/Article/details/422402.sHtML
tv.blog.cvvliy.cn/Article/details/284246.sHtML
tv.blog.cvvliy.cn/Article/details/800864.sHtML
tv.blog.cvvliy.cn/Article/details/480028.sHtML
tv.blog.cvvliy.cn/Article/details/804200.sHtML
tv.blog.cvvliy.cn/Article/details/406886.sHtML
tv.blog.cvvliy.cn/Article/details/848684.sHtML
tv.blog.cvvliy.cn/Article/details/953511.sHtML
tv.blog.cvvliy.cn/Article/details/204468.sHtML
tv.blog.cvvliy.cn/Article/details/537335.sHtML
tv.blog.cvvliy.cn/Article/details/824866.sHtML
tv.blog.cvvliy.cn/Article/details/226428.sHtML
tv.blog.cvvliy.cn/Article/details/775137.sHtML
tv.blog.cvvliy.cn/Article/details/862686.sHtML
tv.blog.cvvliy.cn/Article/details/284624.sHtML
tv.blog.cvvliy.cn/Article/details/224244.sHtML
tv.blog.cvvliy.cn/Article/details/735193.sHtML
tv.blog.cvvliy.cn/Article/details/008066.sHtML
tv.blog.cvvliy.cn/Article/details/626820.sHtML
tv.blog.cvvliy.cn/Article/details/024444.sHtML
tv.blog.cvvliy.cn/Article/details/042268.sHtML
tv.blog.cvvliy.cn/Article/details/648862.sHtML
tv.blog.cvvliy.cn/Article/details/191335.sHtML
tv.blog.cvvliy.cn/Article/details/248820.sHtML
tv.blog.cvvliy.cn/Article/details/400660.sHtML
tv.blog.cvvliy.cn/Article/details/688004.sHtML
tv.blog.cvvliy.cn/Article/details/082464.sHtML
tv.blog.cvvliy.cn/Article/details/282884.sHtML
tv.blog.cvvliy.cn/Article/details/440204.sHtML
tv.blog.cvvliy.cn/Article/details/006040.sHtML
tv.blog.cvvliy.cn/Article/details/775951.sHtML
tv.blog.cvvliy.cn/Article/details/020466.sHtML
tv.blog.cvvliy.cn/Article/details/937917.sHtML
tv.blog.cvvliy.cn/Article/details/539711.sHtML
tv.blog.cvvliy.cn/Article/details/644064.sHtML
tv.blog.cvvliy.cn/Article/details/937511.sHtML
tv.blog.cvvliy.cn/Article/details/088024.sHtML
tv.blog.cvvliy.cn/Article/details/280466.sHtML
tv.blog.cvvliy.cn/Article/details/480642.sHtML
tv.blog.cvvliy.cn/Article/details/286406.sHtML
tv.blog.cvvliy.cn/Article/details/131791.sHtML
tv.blog.cvvliy.cn/Article/details/971713.sHtML
tv.blog.cvvliy.cn/Article/details/753175.sHtML
tv.blog.cvvliy.cn/Article/details/428482.sHtML
tv.blog.cvvliy.cn/Article/details/020806.sHtML
tv.blog.cvvliy.cn/Article/details/022828.sHtML
tv.blog.cvvliy.cn/Article/details/440824.sHtML
tv.blog.cvvliy.cn/Article/details/046288.sHtML
tv.blog.cvvliy.cn/Article/details/044226.sHtML
tv.blog.cvvliy.cn/Article/details/840444.sHtML
tv.blog.cvvliy.cn/Article/details/624868.sHtML
tv.blog.cvvliy.cn/Article/details/753755.sHtML
tv.blog.cvvliy.cn/Article/details/935113.sHtML
tv.blog.cvvliy.cn/Article/details/628444.sHtML
tv.blog.cvvliy.cn/Article/details/262400.sHtML
tv.blog.cvvliy.cn/Article/details/531959.sHtML
tv.blog.cvvliy.cn/Article/details/179137.sHtML
tv.blog.cvvliy.cn/Article/details/371975.sHtML
tv.blog.cvvliy.cn/Article/details/262842.sHtML
tv.blog.cvvliy.cn/Article/details/151375.sHtML
tv.blog.cvvliy.cn/Article/details/084240.sHtML
tv.blog.cvvliy.cn/Article/details/759555.sHtML
tv.blog.cvvliy.cn/Article/details/737775.sHtML
tv.blog.cvvliy.cn/Article/details/240426.sHtML
tv.blog.cvvliy.cn/Article/details/773971.sHtML
tv.blog.cvvliy.cn/Article/details/266448.sHtML
tv.blog.cvvliy.cn/Article/details/319317.sHtML
tv.blog.cvvliy.cn/Article/details/137731.sHtML
tv.blog.cvvliy.cn/Article/details/119795.sHtML
tv.blog.cvvliy.cn/Article/details/591177.sHtML
tv.blog.cvvliy.cn/Article/details/559555.sHtML
tv.blog.cvvliy.cn/Article/details/046282.sHtML
tv.blog.cvvliy.cn/Article/details/004048.sHtML
tv.blog.cvvliy.cn/Article/details/046866.sHtML
tv.blog.cvvliy.cn/Article/details/139133.sHtML
tv.blog.cvvliy.cn/Article/details/282860.sHtML
tv.blog.cvvliy.cn/Article/details/739935.sHtML
tv.blog.cvvliy.cn/Article/details/355731.sHtML
tv.blog.cvvliy.cn/Article/details/820884.sHtML
tv.blog.cvvliy.cn/Article/details/317975.sHtML
tv.blog.cvvliy.cn/Article/details/139573.sHtML
tv.blog.cvvliy.cn/Article/details/200882.sHtML
tv.blog.cvvliy.cn/Article/details/048200.sHtML
tv.blog.cvvliy.cn/Article/details/008806.sHtML
tv.blog.cvvliy.cn/Article/details/646082.sHtML
tv.blog.cvvliy.cn/Article/details/448242.sHtML
tv.blog.cvvliy.cn/Article/details/604480.sHtML
tv.blog.cvvliy.cn/Article/details/804044.sHtML
tv.blog.cvvliy.cn/Article/details/404284.sHtML
tv.blog.cvvliy.cn/Article/details/888606.sHtML
tv.blog.cvvliy.cn/Article/details/084602.sHtML
tv.blog.cvvliy.cn/Article/details/575919.sHtML
tv.blog.cvvliy.cn/Article/details/139731.sHtML
tv.blog.cvvliy.cn/Article/details/868008.sHtML
tv.blog.cvvliy.cn/Article/details/640860.sHtML
tv.blog.cvvliy.cn/Article/details/517139.sHtML
tv.blog.cvvliy.cn/Article/details/800466.sHtML
tv.blog.cvvliy.cn/Article/details/171553.sHtML
tv.blog.cvvliy.cn/Article/details/979175.sHtML
tv.blog.cvvliy.cn/Article/details/284268.sHtML
tv.blog.cvvliy.cn/Article/details/577733.sHtML
tv.blog.cvvliy.cn/Article/details/286446.sHtML
tv.blog.cvvliy.cn/Article/details/773733.sHtML
tv.blog.cvvliy.cn/Article/details/222666.sHtML
tv.blog.cvvliy.cn/Article/details/593751.sHtML
tv.blog.cvvliy.cn/Article/details/062820.sHtML
tv.blog.cvvliy.cn/Article/details/884406.sHtML
tv.blog.cvvliy.cn/Article/details/422242.sHtML
tv.blog.cvvliy.cn/Article/details/551353.sHtML
tv.blog.cvvliy.cn/Article/details/664060.sHtML
tv.blog.cvvliy.cn/Article/details/151179.sHtML
tv.blog.cvvliy.cn/Article/details/155331.sHtML
tv.blog.cvvliy.cn/Article/details/482848.sHtML
tv.blog.cvvliy.cn/Article/details/571511.sHtML
tv.blog.cvvliy.cn/Article/details/488082.sHtML
tv.blog.cvvliy.cn/Article/details/248608.sHtML
tv.blog.cvvliy.cn/Article/details/480040.sHtML
tv.blog.cvvliy.cn/Article/details/557951.sHtML
tv.blog.cvvliy.cn/Article/details/199955.sHtML
tv.blog.cvvliy.cn/Article/details/593177.sHtML
tv.blog.cvvliy.cn/Article/details/044600.sHtML
tv.blog.cvvliy.cn/Article/details/402080.sHtML
tv.blog.cvvliy.cn/Article/details/806060.sHtML
tv.blog.cvvliy.cn/Article/details/662668.sHtML
tv.blog.cvvliy.cn/Article/details/800028.sHtML
tv.blog.cvvliy.cn/Article/details/000088.sHtML
tv.blog.cvvliy.cn/Article/details/353117.sHtML
tv.blog.cvvliy.cn/Article/details/399993.sHtML
tv.blog.cvvliy.cn/Article/details/466826.sHtML
tv.blog.cvvliy.cn/Article/details/204684.sHtML
tv.blog.cvvliy.cn/Article/details/886624.sHtML
tv.blog.cvvliy.cn/Article/details/317717.sHtML
tv.blog.cvvliy.cn/Article/details/404682.sHtML
tv.blog.cvvliy.cn/Article/details/602242.sHtML
tv.blog.cvvliy.cn/Article/details/797973.sHtML
tv.blog.cvvliy.cn/Article/details/888606.sHtML
tv.blog.cvvliy.cn/Article/details/062888.sHtML
tv.blog.cvvliy.cn/Article/details/606064.sHtML
tv.blog.cvvliy.cn/Article/details/179575.sHtML
tv.blog.cvvliy.cn/Article/details/559733.sHtML
tv.blog.cvvliy.cn/Article/details/646866.sHtML
tv.blog.cvvliy.cn/Article/details/933151.sHtML
tv.blog.cvvliy.cn/Article/details/155159.sHtML
tv.blog.cvvliy.cn/Article/details/886448.sHtML
tv.blog.cvvliy.cn/Article/details/575399.sHtML
tv.blog.cvvliy.cn/Article/details/268086.sHtML
tv.blog.cvvliy.cn/Article/details/977133.sHtML
tv.blog.cvvliy.cn/Article/details/482082.sHtML
tv.blog.cvvliy.cn/Article/details/844880.sHtML
tv.blog.cvvliy.cn/Article/details/088602.sHtML
tv.blog.cvvliy.cn/Article/details/042460.sHtML
tv.blog.cvvliy.cn/Article/details/484684.sHtML
tv.blog.cvvliy.cn/Article/details/353775.sHtML
tv.blog.cvvliy.cn/Article/details/971799.sHtML
tv.blog.cvvliy.cn/Article/details/333315.sHtML
tv.blog.cvvliy.cn/Article/details/242068.sHtML
tv.blog.cvvliy.cn/Article/details/860086.sHtML
tv.blog.cvvliy.cn/Article/details/044624.sHtML
tv.blog.cvvliy.cn/Article/details/264688.sHtML
tv.blog.cvvliy.cn/Article/details/755111.sHtML
tv.blog.cvvliy.cn/Article/details/535575.sHtML
tv.blog.cvvliy.cn/Article/details/915151.sHtML
tv.blog.cvvliy.cn/Article/details/888004.sHtML
tv.blog.cvvliy.cn/Article/details/642268.sHtML
tv.blog.cvvliy.cn/Article/details/842840.sHtML
tv.blog.cvvliy.cn/Article/details/399917.sHtML
tv.blog.cvvliy.cn/Article/details/848444.sHtML
tv.blog.cvvliy.cn/Article/details/260646.sHtML
tv.blog.cvvliy.cn/Article/details/339357.sHtML
tv.blog.cvvliy.cn/Article/details/206666.sHtML
tv.blog.cvvliy.cn/Article/details/759177.sHtML
tv.blog.cvvliy.cn/Article/details/737353.sHtML
tv.blog.cvvliy.cn/Article/details/224446.sHtML
tv.blog.cvvliy.cn/Article/details/113797.sHtML
tv.blog.cvvliy.cn/Article/details/042020.sHtML
tv.blog.cvvliy.cn/Article/details/371915.sHtML
tv.blog.cvvliy.cn/Article/details/951571.sHtML
tv.blog.cvvliy.cn/Article/details/797999.sHtML
tv.blog.cvvliy.cn/Article/details/373559.sHtML
tv.blog.cvvliy.cn/Article/details/175719.sHtML
tv.blog.cvvliy.cn/Article/details/139119.sHtML
tv.blog.cvvliy.cn/Article/details/828062.sHtML
tv.blog.cvvliy.cn/Article/details/913719.sHtML
tv.blog.cvvliy.cn/Article/details/195513.sHtML
tv.blog.cvvliy.cn/Article/details/175757.sHtML
tv.blog.cvvliy.cn/Article/details/119199.sHtML
tv.blog.cvvliy.cn/Article/details/511771.sHtML
tv.blog.cvvliy.cn/Article/details/822428.sHtML
tv.blog.cvvliy.cn/Article/details/240806.sHtML
tv.blog.cvvliy.cn/Article/details/573533.sHtML
tv.blog.cvvliy.cn/Article/details/226426.sHtML
tv.blog.cvvliy.cn/Article/details/591353.sHtML
tv.blog.cvvliy.cn/Article/details/006008.sHtML
tv.blog.cvvliy.cn/Article/details/646464.sHtML
tv.blog.cvvliy.cn/Article/details/060080.sHtML
tv.blog.cvvliy.cn/Article/details/595913.sHtML
tv.blog.cvvliy.cn/Article/details/339359.sHtML
tv.blog.cvvliy.cn/Article/details/406248.sHtML
tv.blog.cvvliy.cn/Article/details/533999.sHtML
tv.blog.cvvliy.cn/Article/details/082804.sHtML
tv.blog.cvvliy.cn/Article/details/997357.sHtML
tv.blog.cvvliy.cn/Article/details/226628.sHtML
tv.blog.cvvliy.cn/Article/details/640064.sHtML
tv.blog.cvvliy.cn/Article/details/284608.sHtML
tv.blog.cvvliy.cn/Article/details/040420.sHtML
tv.blog.cvvliy.cn/Article/details/717373.sHtML
tv.blog.cvvliy.cn/Article/details/200608.sHtML
tv.blog.cvvliy.cn/Article/details/400242.sHtML
tv.blog.cvvliy.cn/Article/details/408844.sHtML
tv.blog.cvvliy.cn/Article/details/797595.sHtML
tv.blog.cvvliy.cn/Article/details/824048.sHtML
tv.blog.cvvliy.cn/Article/details/402608.sHtML
tv.blog.cvvliy.cn/Article/details/840208.sHtML
tv.blog.cvvliy.cn/Article/details/137379.sHtML
tv.blog.cvvliy.cn/Article/details/515791.sHtML
tv.blog.cvvliy.cn/Article/details/991753.sHtML
tv.blog.cvvliy.cn/Article/details/200662.sHtML
tv.blog.cvvliy.cn/Article/details/519539.sHtML
tv.blog.cvvliy.cn/Article/details/206222.sHtML
tv.blog.cvvliy.cn/Article/details/913171.sHtML
tv.blog.cvvliy.cn/Article/details/642466.sHtML
tv.blog.cvvliy.cn/Article/details/395191.sHtML
tv.blog.cvvliy.cn/Article/details/919335.sHtML
tv.blog.cvvliy.cn/Article/details/937171.sHtML
tv.blog.cvvliy.cn/Article/details/933139.sHtML
tv.blog.cvvliy.cn/Article/details/339193.sHtML
tv.blog.cvvliy.cn/Article/details/040662.sHtML
tv.blog.cvvliy.cn/Article/details/024044.sHtML
tv.blog.cvvliy.cn/Article/details/351375.sHtML
tv.blog.cvvliy.cn/Article/details/802840.sHtML
tv.blog.cvvliy.cn/Article/details/260686.sHtML
tv.blog.cvvliy.cn/Article/details/660028.sHtML
tv.blog.cvvliy.cn/Article/details/971991.sHtML
tv.blog.cvvliy.cn/Article/details/311575.sHtML
tv.blog.cvvliy.cn/Article/details/802484.sHtML
tv.blog.cvvliy.cn/Article/details/351751.sHtML
tv.blog.cvvliy.cn/Article/details/424268.sHtML
tv.blog.cvvliy.cn/Article/details/088004.sHtML
tv.blog.cvvliy.cn/Article/details/266806.sHtML
tv.blog.cvvliy.cn/Article/details/515993.sHtML
tv.blog.cvvliy.cn/Article/details/666602.sHtML
tv.blog.cvvliy.cn/Article/details/111595.sHtML
tv.blog.cvvliy.cn/Article/details/206628.sHtML
tv.blog.cvvliy.cn/Article/details/533575.sHtML
tv.blog.cvvliy.cn/Article/details/175579.sHtML
tv.blog.cvvliy.cn/Article/details/191957.sHtML
tv.blog.cvvliy.cn/Article/details/911351.sHtML
tv.blog.cvvliy.cn/Article/details/313959.sHtML
tv.blog.cvvliy.cn/Article/details/533575.sHtML
tv.blog.cvvliy.cn/Article/details/088868.sHtML
tv.blog.cvvliy.cn/Article/details/046286.sHtML
tv.blog.cvvliy.cn/Article/details/468222.sHtML
tv.blog.cvvliy.cn/Article/details/220066.sHtML
tv.blog.cvvliy.cn/Article/details/915151.sHtML
tv.blog.cvvliy.cn/Article/details/460842.sHtML
tv.blog.cvvliy.cn/Article/details/088620.sHtML
tv.blog.cvvliy.cn/Article/details/848244.sHtML
tv.blog.cvvliy.cn/Article/details/628004.sHtML
tv.blog.cvvliy.cn/Article/details/006084.sHtML
tv.blog.cvvliy.cn/Article/details/664648.sHtML
tv.blog.cvvliy.cn/Article/details/080242.sHtML
tv.blog.cvvliy.cn/Article/details/682208.sHtML
tv.blog.cvvliy.cn/Article/details/591797.sHtML
tv.blog.cvvliy.cn/Article/details/260846.sHtML
tv.blog.cvvliy.cn/Article/details/442402.sHtML
tv.blog.cvvliy.cn/Article/details/046422.sHtML
tv.blog.cvvliy.cn/Article/details/862844.sHtML
tv.blog.cvvliy.cn/Article/details/442488.sHtML
tv.blog.cvvliy.cn/Article/details/399113.sHtML
tv.blog.cvvliy.cn/Article/details/440488.sHtML
tv.blog.cvvliy.cn/Article/details/422840.sHtML
tv.blog.cvvliy.cn/Article/details/715571.sHtML
tv.blog.cvvliy.cn/Article/details/468608.sHtML
tv.blog.cvvliy.cn/Article/details/626622.sHtML
tv.blog.cvvliy.cn/Article/details/606048.sHtML
tv.blog.cvvliy.cn/Article/details/773397.sHtML
tv.blog.cvvliy.cn/Article/details/933771.sHtML
tv.blog.cvvliy.cn/Article/details/379935.sHtML
tv.blog.cvvliy.cn/Article/details/737311.sHtML
tv.blog.cvvliy.cn/Article/details/575975.sHtML
tv.blog.cvvliy.cn/Article/details/864446.sHtML
tv.blog.cvvliy.cn/Article/details/751331.sHtML
tv.blog.cvvliy.cn/Article/details/644682.sHtML
tv.blog.cvvliy.cn/Article/details/157377.sHtML
tv.blog.cvvliy.cn/Article/details/622288.sHtML
tv.blog.cvvliy.cn/Article/details/959951.sHtML
tv.blog.cvvliy.cn/Article/details/242404.sHtML
tv.blog.cvvliy.cn/Article/details/806046.sHtML
tv.blog.cvvliy.cn/Article/details/024426.sHtML
tv.blog.cvvliy.cn/Article/details/862844.sHtML
tv.blog.cvvliy.cn/Article/details/751591.sHtML
tv.blog.cvvliy.cn/Article/details/448022.sHtML
tv.blog.cvvliy.cn/Article/details/264822.sHtML
tv.blog.cvvliy.cn/Article/details/006086.sHtML
tv.blog.cvvliy.cn/Article/details/755317.sHtML
tv.blog.cvvliy.cn/Article/details/979355.sHtML
tv.blog.cvvliy.cn/Article/details/511775.sHtML
tv.blog.cvvliy.cn/Article/details/119751.sHtML
tv.blog.cvvliy.cn/Article/details/795351.sHtML
tv.blog.cvvliy.cn/Article/details/682800.sHtML
tv.blog.cvvliy.cn/Article/details/842040.sHtML
tv.blog.cvvliy.cn/Article/details/464808.sHtML
tv.blog.cvvliy.cn/Article/details/759139.sHtML
tv.blog.cvvliy.cn/Article/details/517953.sHtML
tv.blog.cvvliy.cn/Article/details/575719.sHtML
