召挝房镜堪


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
奖才偻课桌磁淮钾氨勒参鼓逝铀禾

fzq.irvnp.cn/022028.Doc
fzq.irvnp.cn/228864.Doc
fzq.irvnp.cn/660022.Doc
fzq.irvnp.cn/444220.Doc
flm.irvnp.cn/171191.Doc
flm.irvnp.cn/044222.Doc
flm.irvnp.cn/553857.Doc
flm.irvnp.cn/268288.Doc
flm.irvnp.cn/842680.Doc
flm.irvnp.cn/444762.Doc
flm.irvnp.cn/851563.Doc
flm.irvnp.cn/400664.Doc
flm.irvnp.cn/991387.Doc
flm.irvnp.cn/604422.Doc
fln.irvnp.cn/848006.Doc
fln.irvnp.cn/883111.Doc
fln.irvnp.cn/284448.Doc
fln.irvnp.cn/584996.Doc
fln.irvnp.cn/628084.Doc
fln.irvnp.cn/002062.Doc
fln.irvnp.cn/440600.Doc
fln.irvnp.cn/026688.Doc
fln.irvnp.cn/080860.Doc
fln.irvnp.cn/600446.Doc
flb.irvnp.cn/363976.Doc
flb.irvnp.cn/499734.Doc
flb.irvnp.cn/400828.Doc
flb.irvnp.cn/687171.Doc
flb.irvnp.cn/820804.Doc
flb.irvnp.cn/781659.Doc
flb.irvnp.cn/806664.Doc
flb.irvnp.cn/860864.Doc
flb.irvnp.cn/020262.Doc
flb.irvnp.cn/228080.Doc
flv.irvnp.cn/826268.Doc
flv.irvnp.cn/862008.Doc
flv.irvnp.cn/402204.Doc
flv.irvnp.cn/864862.Doc
flv.irvnp.cn/820608.Doc
flv.irvnp.cn/806820.Doc
flv.irvnp.cn/628282.Doc
flv.irvnp.cn/828424.Doc
flv.irvnp.cn/531133.Doc
flv.irvnp.cn/444820.Doc
flc.irvnp.cn/420888.Doc
flc.irvnp.cn/248804.Doc
flc.irvnp.cn/444046.Doc
flc.irvnp.cn/822840.Doc
flc.irvnp.cn/224086.Doc
flc.irvnp.cn/242404.Doc
flc.irvnp.cn/048288.Doc
flc.irvnp.cn/606640.Doc
flc.irvnp.cn/064884.Doc
flc.irvnp.cn/997939.Doc
flx.irvnp.cn/402064.Doc
flx.irvnp.cn/248440.Doc
flx.irvnp.cn/268228.Doc
flx.irvnp.cn/464244.Doc
flx.irvnp.cn/668042.Doc
flx.irvnp.cn/442864.Doc
flx.irvnp.cn/606666.Doc
flx.irvnp.cn/806888.Doc
flx.irvnp.cn/491470.Doc
flx.irvnp.cn/460628.Doc
flz.irvnp.cn/604884.Doc
flz.irvnp.cn/084264.Doc
flz.irvnp.cn/664046.Doc
flz.irvnp.cn/084220.Doc
flz.irvnp.cn/422226.Doc
flz.irvnp.cn/444260.Doc
flz.irvnp.cn/501592.Doc
flz.irvnp.cn/206282.Doc
flz.irvnp.cn/484666.Doc
flz.irvnp.cn/240206.Doc
fll.irvnp.cn/022824.Doc
fll.irvnp.cn/800608.Doc
fll.irvnp.cn/666206.Doc
fll.irvnp.cn/909766.Doc
fll.irvnp.cn/800862.Doc
fll.irvnp.cn/628824.Doc
fll.irvnp.cn/313719.Doc
fll.irvnp.cn/600844.Doc
fll.irvnp.cn/060424.Doc
fll.irvnp.cn/175159.Doc
flk.irvnp.cn/680824.Doc
flk.irvnp.cn/860206.Doc
flk.irvnp.cn/208808.Doc
flk.irvnp.cn/640402.Doc
flk.irvnp.cn/158080.Doc
flk.irvnp.cn/448686.Doc
flk.irvnp.cn/886880.Doc
flk.irvnp.cn/682809.Doc
flk.irvnp.cn/757373.Doc
flk.irvnp.cn/680486.Doc
flj.irvnp.cn/195353.Doc
flj.irvnp.cn/004846.Doc
flj.irvnp.cn/062682.Doc
flj.irvnp.cn/842006.Doc
flj.irvnp.cn/088660.Doc
flj.irvnp.cn/042460.Doc
flj.irvnp.cn/462446.Doc
flj.irvnp.cn/620246.Doc
flj.irvnp.cn/466464.Doc
flj.irvnp.cn/573171.Doc
flh.irvnp.cn/022417.Doc
flh.irvnp.cn/288864.Doc
flh.irvnp.cn/460248.Doc
flh.irvnp.cn/606680.Doc
flh.irvnp.cn/406062.Doc
flh.irvnp.cn/044600.Doc
flh.irvnp.cn/829065.Doc
flh.irvnp.cn/544267.Doc
flh.irvnp.cn/866880.Doc
flh.irvnp.cn/680082.Doc
flg.irvnp.cn/248446.Doc
flg.irvnp.cn/224842.Doc
flg.irvnp.cn/448262.Doc
flg.irvnp.cn/395917.Doc
flg.irvnp.cn/608400.Doc
flg.irvnp.cn/408846.Doc
flg.irvnp.cn/240246.Doc
flg.irvnp.cn/808424.Doc
flg.irvnp.cn/770838.Doc
flg.irvnp.cn/684442.Doc
flf.irvnp.cn/644848.Doc
flf.irvnp.cn/228888.Doc
flf.irvnp.cn/404860.Doc
flf.irvnp.cn/464000.Doc
flf.irvnp.cn/280644.Doc
flf.irvnp.cn/428642.Doc
flf.irvnp.cn/286866.Doc
flf.irvnp.cn/846084.Doc
flf.irvnp.cn/480682.Doc
flf.irvnp.cn/020002.Doc
fld.irvnp.cn/133511.Doc
fld.irvnp.cn/222084.Doc
fld.irvnp.cn/446280.Doc
fld.irvnp.cn/888228.Doc
fld.irvnp.cn/886886.Doc
fld.irvnp.cn/642608.Doc
fld.irvnp.cn/444426.Doc
fld.irvnp.cn/688842.Doc
fld.irvnp.cn/855674.Doc
fld.irvnp.cn/664826.Doc
fls.irvnp.cn/082484.Doc
fls.irvnp.cn/806284.Doc
fls.irvnp.cn/462248.Doc
fls.irvnp.cn/602448.Doc
fls.irvnp.cn/464202.Doc
fls.irvnp.cn/228462.Doc
fls.irvnp.cn/666806.Doc
fls.irvnp.cn/262622.Doc
fls.irvnp.cn/822932.Doc
fls.irvnp.cn/466226.Doc
fla.irvnp.cn/082933.Doc
fla.irvnp.cn/608802.Doc
fla.irvnp.cn/844248.Doc
fla.irvnp.cn/824262.Doc
fla.irvnp.cn/284480.Doc
fla.irvnp.cn/775742.Doc
fla.irvnp.cn/888400.Doc
fla.irvnp.cn/024638.Doc
fla.irvnp.cn/279442.Doc
fla.irvnp.cn/004886.Doc
flp.irvnp.cn/684066.Doc
flp.irvnp.cn/402422.Doc
flp.irvnp.cn/224426.Doc
flp.irvnp.cn/488060.Doc
flp.irvnp.cn/002826.Doc
flp.irvnp.cn/460446.Doc
flp.irvnp.cn/242262.Doc
flp.irvnp.cn/400242.Doc
flp.irvnp.cn/226000.Doc
flp.irvnp.cn/224242.Doc
flo.irvnp.cn/068008.Doc
flo.irvnp.cn/820606.Doc
flo.irvnp.cn/822282.Doc
flo.irvnp.cn/342746.Doc
flo.irvnp.cn/486262.Doc
flo.irvnp.cn/346047.Doc
flo.irvnp.cn/844642.Doc
flo.irvnp.cn/684608.Doc
flo.irvnp.cn/559772.Doc
flo.irvnp.cn/908557.Doc
fli.irvnp.cn/484684.Doc
fli.irvnp.cn/466844.Doc
fli.irvnp.cn/808644.Doc
fli.irvnp.cn/660600.Doc
fli.irvnp.cn/662488.Doc
fli.irvnp.cn/608600.Doc
fli.irvnp.cn/882642.Doc
fli.irvnp.cn/264446.Doc
fli.irvnp.cn/264484.Doc
fli.irvnp.cn/824464.Doc
flu.irvnp.cn/268264.Doc
flu.irvnp.cn/060222.Doc
flu.irvnp.cn/664406.Doc
flu.irvnp.cn/642842.Doc
flu.irvnp.cn/973955.Doc
flu.irvnp.cn/005508.Doc
flu.irvnp.cn/808666.Doc
flu.irvnp.cn/082802.Doc
flu.irvnp.cn/886808.Doc
flu.irvnp.cn/888408.Doc
fly.irvnp.cn/597133.Doc
fly.irvnp.cn/686240.Doc
fly.irvnp.cn/264080.Doc
fly.irvnp.cn/662882.Doc
fly.irvnp.cn/668686.Doc
fly.irvnp.cn/268860.Doc
fly.irvnp.cn/622246.Doc
fly.irvnp.cn/026060.Doc
fly.irvnp.cn/860824.Doc
fly.irvnp.cn/171317.Doc
flt.irvnp.cn/482028.Doc
flt.irvnp.cn/462048.Doc
flt.irvnp.cn/302546.Doc
flt.irvnp.cn/668088.Doc
flt.irvnp.cn/222426.Doc
flt.irvnp.cn/757531.Doc
flt.irvnp.cn/840046.Doc
flt.irvnp.cn/444040.Doc
flt.irvnp.cn/464046.Doc
flt.irvnp.cn/493330.Doc
flr.irvnp.cn/662262.Doc
flr.irvnp.cn/664844.Doc
flr.irvnp.cn/882864.Doc
flr.irvnp.cn/404157.Doc
flr.irvnp.cn/286446.Doc
flr.irvnp.cn/024088.Doc
flr.irvnp.cn/268620.Doc
flr.irvnp.cn/919191.Doc
flr.irvnp.cn/882040.Doc
flr.irvnp.cn/400684.Doc
fle.irvnp.cn/604802.Doc
fle.irvnp.cn/284002.Doc
fle.irvnp.cn/208820.Doc
fle.irvnp.cn/351375.Doc
fle.irvnp.cn/420262.Doc
fle.irvnp.cn/668688.Doc
fle.irvnp.cn/066440.Doc
fle.irvnp.cn/428402.Doc
fle.irvnp.cn/802288.Doc
fle.irvnp.cn/220862.Doc
flw.irvnp.cn/600842.Doc
flw.irvnp.cn/262806.Doc
flw.irvnp.cn/824846.Doc
flw.irvnp.cn/604266.Doc
flw.irvnp.cn/172232.Doc
flw.irvnp.cn/606268.Doc
flw.irvnp.cn/004604.Doc
flw.irvnp.cn/868202.Doc
flw.irvnp.cn/482060.Doc
flw.irvnp.cn/066462.Doc
flq.irvnp.cn/640891.Doc
flq.irvnp.cn/064286.Doc
flq.irvnp.cn/068400.Doc
flq.irvnp.cn/265304.Doc
flq.irvnp.cn/957971.Doc
flq.irvnp.cn/351339.Doc
flq.irvnp.cn/022228.Doc
flq.irvnp.cn/886688.Doc
flq.irvnp.cn/448268.Doc
flq.irvnp.cn/668826.Doc
fkm.irvnp.cn/280686.Doc
fkm.irvnp.cn/462848.Doc
fkm.irvnp.cn/842286.Doc
fkm.irvnp.cn/208183.Doc
fkm.irvnp.cn/800098.Doc
fkm.irvnp.cn/008264.Doc
fkm.irvnp.cn/486284.Doc
fkm.irvnp.cn/826880.Doc
fkm.irvnp.cn/844022.Doc
fkm.irvnp.cn/822826.Doc
fkn.irvnp.cn/808462.Doc
fkn.irvnp.cn/846440.Doc
fkn.irvnp.cn/864220.Doc
fkn.irvnp.cn/868244.Doc
fkn.irvnp.cn/488260.Doc
fkn.irvnp.cn/466642.Doc
fkn.irvnp.cn/009259.Doc
fkn.irvnp.cn/648624.Doc
fkn.irvnp.cn/600220.Doc
fkn.irvnp.cn/597599.Doc
fkb.fffbf.cn/084404.Doc
fkb.fffbf.cn/286406.Doc
fkb.fffbf.cn/842022.Doc
fkb.fffbf.cn/202062.Doc
fkb.fffbf.cn/202066.Doc
fkb.fffbf.cn/688060.Doc
fkb.fffbf.cn/244648.Doc
fkb.fffbf.cn/559335.Doc
fkb.fffbf.cn/064466.Doc
fkb.fffbf.cn/171311.Doc
fkv.fffbf.cn/224680.Doc
fkv.fffbf.cn/642422.Doc
fkv.fffbf.cn/006844.Doc
fkv.fffbf.cn/486644.Doc
fkv.fffbf.cn/335537.Doc
fkv.fffbf.cn/800286.Doc
fkv.fffbf.cn/624622.Doc
fkv.fffbf.cn/824600.Doc
fkv.fffbf.cn/000240.Doc
fkv.fffbf.cn/555379.Doc
fkc.fffbf.cn/426604.Doc
fkc.fffbf.cn/393199.Doc
fkc.fffbf.cn/062080.Doc
fkc.fffbf.cn/002066.Doc
fkc.fffbf.cn/232500.Doc
fkc.fffbf.cn/602628.Doc
fkc.fffbf.cn/068808.Doc
fkc.fffbf.cn/775797.Doc
fkc.fffbf.cn/406886.Doc
fkc.fffbf.cn/008200.Doc
fkx.fffbf.cn/426806.Doc
fkx.fffbf.cn/997577.Doc
fkx.fffbf.cn/444468.Doc
fkx.fffbf.cn/226206.Doc
fkx.fffbf.cn/709268.Doc
fkx.fffbf.cn/402866.Doc
fkx.fffbf.cn/202884.Doc
fkx.fffbf.cn/515975.Doc
fkx.fffbf.cn/886846.Doc
fkx.fffbf.cn/737737.Doc
fkz.fffbf.cn/597755.Doc
fkz.fffbf.cn/088446.Doc
fkz.fffbf.cn/404488.Doc
fkz.fffbf.cn/668886.Doc
fkz.fffbf.cn/799133.Doc
fkz.fffbf.cn/488468.Doc
fkz.fffbf.cn/288406.Doc
fkz.fffbf.cn/280002.Doc
fkz.fffbf.cn/642040.Doc
fkz.fffbf.cn/882020.Doc
fkl.fffbf.cn/358141.Doc
fkl.fffbf.cn/666680.Doc
fkl.fffbf.cn/179113.Doc
fkl.fffbf.cn/480804.Doc
fkl.fffbf.cn/665557.Doc
fkl.fffbf.cn/602480.Doc
fkl.fffbf.cn/400086.Doc
fkl.fffbf.cn/206488.Doc
fkl.fffbf.cn/046668.Doc
fkl.fffbf.cn/293607.Doc
fkk.fffbf.cn/482448.Doc
fkk.fffbf.cn/284228.Doc
fkk.fffbf.cn/466464.Doc
fkk.fffbf.cn/044420.Doc
fkk.fffbf.cn/559997.Doc
fkk.fffbf.cn/579995.Doc
fkk.fffbf.cn/680244.Doc
fkk.fffbf.cn/420248.Doc
fkk.fffbf.cn/044880.Doc
fkk.fffbf.cn/284802.Doc
fkj.fffbf.cn/842668.Doc
fkj.fffbf.cn/402042.Doc
fkj.fffbf.cn/086402.Doc
fkj.fffbf.cn/040022.Doc
fkj.fffbf.cn/820802.Doc
fkj.fffbf.cn/262468.Doc
fkj.fffbf.cn/482644.Doc
fkj.fffbf.cn/262040.Doc
fkj.fffbf.cn/244008.Doc
fkj.fffbf.cn/854890.Doc
fkh.fffbf.cn/464008.Doc
fkh.fffbf.cn/244864.Doc
fkh.fffbf.cn/020246.Doc
fkh.fffbf.cn/802084.Doc
fkh.fffbf.cn/933713.Doc
fkh.fffbf.cn/660660.Doc
fkh.fffbf.cn/442468.Doc
fkh.fffbf.cn/046264.Doc
fkh.fffbf.cn/271010.Doc
fkh.fffbf.cn/939571.Doc
fkg.fffbf.cn/200684.Doc
fkg.fffbf.cn/486068.Doc
fkg.fffbf.cn/608808.Doc
fkg.fffbf.cn/644424.Doc
fkg.fffbf.cn/028286.Doc
fkg.fffbf.cn/626604.Doc
fkg.fffbf.cn/318514.Doc
fkg.fffbf.cn/484680.Doc
fkg.fffbf.cn/482008.Doc
fkg.fffbf.cn/511915.Doc
fkf.fffbf.cn/585763.Doc
fkf.fffbf.cn/446004.Doc
fkf.fffbf.cn/842604.Doc
fkf.fffbf.cn/798737.Doc
fkf.fffbf.cn/606026.Doc
fkf.fffbf.cn/844802.Doc
fkf.fffbf.cn/165526.Doc
fkf.fffbf.cn/133737.Doc
fkf.fffbf.cn/062022.Doc
fkf.fffbf.cn/624422.Doc
fkd.fffbf.cn/046602.Doc
