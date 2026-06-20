赐泛痘静靖


Linux xfs_alloc_ag_alloc_perAG空间分配与cntbt

xfs_alloc_ag_alloc_perAG是XFS文件系统空间分配的核心入口，它负责在单个分配组（AG）内找到并分配指定长度的连续空闲空间。该函数通过cntbt（Count B+Tree）和bnobt（Block Number B+Tree）两棵B+树管理空闲空间，位于fs/xfs/libxfs/xfs_alloc.c中。

xfs_alloc_ag_alloc_perAG接收一个struct xfs_alloc_arg参数，包含了分配所需的所有信息：AG编号、请求块数、最小对齐要求、分配类型等。其核心逻辑是选择合适的B+树进行搜索并执行分配：

int xfs_alloc_ag_alloc_perAG(
    struct xfs_alloc_arg  *args,
    struct xfs_alloc_cur  *acur)
{
    struct xfs_perag      *pag = args->pag;
    int                   error;
    int                   i;
    int                   btree_type;

    /* 根据分配类型选择合适的B+树 */
    if (xfs_alloc_is_userdata(args->datatype))
        btree_type = XFS_BTNUM_CNT;  /* 用户数据优先使用cntbt */
    else
        btree_type = XFS_BTNUM_BNO;  /* 元数据优先使用bnobt */

retry:
    /* 从cntbt或bnobt中搜索空闲空间 */
    error = xfs_alloc_search_busy(args);
    if (error)
        return error;

    for (i = 0; i < 2; i++) {
        error = xfs_alloc_cur_setup(args, acur, btree_type);
        if (error)
            return error;

        /* 执行实际的分配查找 */
        error = xfs_alloc_cur_finish(args, acur);
        if (error != -ENOSPC)
            break;

        /* 如果当前B+树分配失败，尝试另一个 */
        if (btree_type == XFS_BTNUM_CNT)
            btree_type = XFS_BTNUM_BNO;
        else
            btree_type = XFS_BTNUM_CNT;

        xfs_alloc_cur_clear(acur);
    }

    if (error == -ENOSPC) {
        /* 尝试小块分配或从其他AG分配 */
        return -ENOSPC;
    }
    ...
    return error;
}

cntbt（Count B+Tree）是XFS空间分配的关键数据结构，它以空闲块的连续长度作为B+树的key，每个key对应一个(start, length)对。cntbt的叶子节点存储struct xfs_alloc_rec：

struct xfs_alloc_rec {
    __be32 ar_startblock; /* 起始块号（相对于AG） */
    __be32 ar_blockcount; /* 连续块数 */
};

cntbt的搜索使用xfs_alloc_lookup_ge（大于等于）或xfs_alloc_lookup_le（小于等于）进行范围查找。xfs_alloc_cur_finish内部调用xfs_cntbt_search来定位满足请求长度的最小连续空间：

STATIC int xfs_cntbt_search(
    struct xfs_btree_cur  *cur,
    xfs_agblock_t         bno,
    int                   *stat)
{
    int                   error;

    /* 在cntbt中查找第一个blkcnt >= 请求长度的记录 */
    cur->bc_rec.a.ar_startblock = 0;
    cur->bc_rec.a.ar_blockcount = cur->bc_private.a.alloc_rec->ar_blockcount;

    error = xfs_btree_lookup(cur, XFS_LOOKUP_GE, stat);
    if (error)
        return error;

    if (*stat == 0)
        return 0;

    /* 验证找到的记录是否满足需求 */
    error = xfs_btree_get_rec(cur, &recp, stat);
    if (error)
        return error;

    if (*stat == 0 || recp->ar_blockcount < cur->bc_rec.a.ar_blockcount)
        return 0;

    return 0;
}

实际的空间分割通过xfs_alloc_update处理。当在cntbt中找到合适的空闲块后，需要将分配的部分从空闲记录中切除，剩余的块作为新记录插入cntbt：

STATIC int xfs_alloc_update(
    struct xfs_btree_cur  *cur,
    xfs_agblock_t         bno,
    xfs_extlen_t          len)
{
    union xfs_btree_rec   *rec;
    int                   stat;

    /* 获取当前记录 */
    int error = xfs_btree_get_rec(cur, &rec, &stat);
    if (error)
        return error;

    /* 如果全部使用，直接删除记录 */
    if (rec->alloc.ar_startblock == cpu_to_be32(bno) &&
        rec->alloc.ar_blockcount == cpu_to_be32(len)) {
        return xfs_btree_delete(cur, &stat);
    }

    /* 从头部使用，更新起始块号 */
    if (rec->alloc.ar_startblock == cpu_to_be32(bno)) {
        rec->alloc.ar_startblock = cpu_to_be32(bno + len);
        rec->alloc.ar_blockcount = cpu_to_be32(
                be32_to_cpu(rec->alloc.ar_blockcount) - len);
        return xfs_btree_update(cur, &stat);
    }

    /* 从中间或尾部使用，需要拆分记录 */
    {
        xfs_agblock_t old_start = be32_to_cpu(rec->alloc.ar_startblock);
        xfs_extlen_t old_count = be32_to_cpu(rec->alloc.ar_blockcount);
        xfs_extlen_t new_count = bno - old_start;

        /* 更新原记录为左侧剩余部分 */
        rec->alloc.ar_blockcount = cpu_to_be32(new_count);
        error = xfs_btree_update(cur, &stat);
        if (error)
            return error;

        /* 右侧剩余部分插入新记录 */
        if (new_count + len < old_count) {
            error = xfs_alloc_insert(cur,
                    old_start + new_count + len,
                    old_count - new_count - len,
                    &stat);
            if (error)
                return error;
        }
    }
    return 0;
}

bnobt与cntbt形成互补。bnobt以块号作为key，支持按物理位置搜索空闲块，适合元数据分配（需要位置接近）。cntbt以大小为key，支持按大小搜索空闲块，适合数据分配（优先获取大块连续空间）。xfs_alloc_ag_alloc_perAG在首次尝试cntbt失败后回退到bnobt。

分配完成后的AG计数器更新通过xfs_alloc_log_agf完成，该函数将AG空闲块计数（agf_freeblks）和最长连续空闲块长度（agf_longest）写入AGF（Allocation Group Free block）块：

void xfs_alloc_log_agf(
    struct xfs_trans      *tp,
    struct xfs_buf        *bp,
    int                   fields)
{
    int                   first;
    int                   last;

    xfs_trans_log_buf(tp, bp, first, last);

    if (fields & XFS_AGF_FREEBLKS) {
        be32_add_cpu(&agf->agf_freeblks, -(int32_t)args->len);
    }
    if (fields & XFS_AGF_LONGEST) {
        /* 重新计算最长空闲块 */
        agf->agf_longest = cpu_to_be32(
                xfs_alloc_longest_free_extent(pag));
    }
}

cntbt的维护涉及btree的分裂和合并操作。当插入新的空闲记录导致节点溢出时，xfs_btree_split被调用，将满节点分裂为两个节点，并将中间key提升到父节点。这一过程可能向上传播到根节点，导致树的高度增加。xfs_alloc_fixup_trees在释放空间后同时更新bnobt和cntbt，确保两棵B+树的一致性。

xfs_alloc_ag_alloc_perAG的并发控制通过perag结构体的AGI/AGF缓冲区的锁和btree cursor锁实现。分配过程中先通过xfs_alloc_read_agf读取AGF缓冲区，然后获取btree cursor，确保在cntbt遍历和修改期间的原子性。当多个CPU同时尝试从同一个AG分配空间时，AGF的锁定会序列化分配操作，保证cntbt和bnobt的数据一致性。
俳谥炮汛椭胸脑胖峦悄徘院收谒栈

cn.blog.cvvliy.cn/Article/details/573557.sHtML
cn.blog.cvvliy.cn/Article/details/440486.sHtML
cn.blog.cvvliy.cn/Article/details/919157.sHtML
cn.blog.cvvliy.cn/Article/details/882824.sHtML
cn.blog.cvvliy.cn/Article/details/002048.sHtML
cn.blog.cvvliy.cn/Article/details/464044.sHtML
cn.blog.cvvliy.cn/Article/details/935551.sHtML
cn.blog.cvvliy.cn/Article/details/882680.sHtML
cn.blog.cvvliy.cn/Article/details/286848.sHtML
cn.blog.cvvliy.cn/Article/details/155531.sHtML
cn.blog.cvvliy.cn/Article/details/066620.sHtML
cn.blog.cvvliy.cn/Article/details/577917.sHtML
cn.blog.cvvliy.cn/Article/details/939779.sHtML
cn.blog.cvvliy.cn/Article/details/206220.sHtML
cn.blog.cvvliy.cn/Article/details/397199.sHtML
cn.blog.cvvliy.cn/Article/details/284068.sHtML
cn.blog.cvvliy.cn/Article/details/484082.sHtML
cn.blog.cvvliy.cn/Article/details/004466.sHtML
cn.blog.cvvliy.cn/Article/details/208464.sHtML
cn.blog.cvvliy.cn/Article/details/422440.sHtML
cn.blog.cvvliy.cn/Article/details/866888.sHtML
cn.blog.cvvliy.cn/Article/details/662080.sHtML
cn.blog.cvvliy.cn/Article/details/173751.sHtML
cn.blog.cvvliy.cn/Article/details/062406.sHtML
cn.blog.cvvliy.cn/Article/details/000022.sHtML
cn.blog.cvvliy.cn/Article/details/206204.sHtML
cn.blog.cvvliy.cn/Article/details/046884.sHtML
cn.blog.cvvliy.cn/Article/details/806824.sHtML
cn.blog.cvvliy.cn/Article/details/080202.sHtML
cn.blog.cvvliy.cn/Article/details/200442.sHtML
cn.blog.cvvliy.cn/Article/details/066402.sHtML
cn.blog.cvvliy.cn/Article/details/777939.sHtML
cn.blog.cvvliy.cn/Article/details/024662.sHtML
cn.blog.cvvliy.cn/Article/details/668002.sHtML
cn.blog.cvvliy.cn/Article/details/240844.sHtML
cn.blog.cvvliy.cn/Article/details/462660.sHtML
cn.blog.cvvliy.cn/Article/details/268086.sHtML
cn.blog.cvvliy.cn/Article/details/824406.sHtML
cn.blog.cvvliy.cn/Article/details/008440.sHtML
cn.blog.cvvliy.cn/Article/details/660026.sHtML
cn.blog.cvvliy.cn/Article/details/006086.sHtML
cn.blog.cvvliy.cn/Article/details/915119.sHtML
cn.blog.cvvliy.cn/Article/details/440060.sHtML
cn.blog.cvvliy.cn/Article/details/286088.sHtML
cn.blog.cvvliy.cn/Article/details/288288.sHtML
cn.blog.cvvliy.cn/Article/details/759959.sHtML
cn.blog.cvvliy.cn/Article/details/422600.sHtML
cn.blog.cvvliy.cn/Article/details/357951.sHtML
cn.blog.cvvliy.cn/Article/details/319379.sHtML
cn.blog.cvvliy.cn/Article/details/753371.sHtML
cn.blog.cvvliy.cn/Article/details/648640.sHtML
cn.blog.cvvliy.cn/Article/details/640440.sHtML
cn.blog.cvvliy.cn/Article/details/826604.sHtML
cn.blog.cvvliy.cn/Article/details/266428.sHtML
cn.blog.cvvliy.cn/Article/details/733935.sHtML
cn.blog.cvvliy.cn/Article/details/539193.sHtML
cn.blog.cvvliy.cn/Article/details/333937.sHtML
cn.blog.cvvliy.cn/Article/details/866640.sHtML
cn.blog.cvvliy.cn/Article/details/442684.sHtML
cn.blog.cvvliy.cn/Article/details/408006.sHtML
cn.blog.cvvliy.cn/Article/details/393973.sHtML
cn.blog.cvvliy.cn/Article/details/642220.sHtML
cn.blog.cvvliy.cn/Article/details/026084.sHtML
cn.blog.cvvliy.cn/Article/details/246422.sHtML
cn.blog.cvvliy.cn/Article/details/884682.sHtML
cn.blog.cvvliy.cn/Article/details/648206.sHtML
cn.blog.cvvliy.cn/Article/details/791153.sHtML
cn.blog.cvvliy.cn/Article/details/082422.sHtML
cn.blog.cvvliy.cn/Article/details/024664.sHtML
cn.blog.cvvliy.cn/Article/details/177971.sHtML
cn.blog.cvvliy.cn/Article/details/206440.sHtML
cn.blog.cvvliy.cn/Article/details/688064.sHtML
cn.blog.cvvliy.cn/Article/details/664648.sHtML
cn.blog.cvvliy.cn/Article/details/026666.sHtML
cn.blog.cvvliy.cn/Article/details/662002.sHtML
cn.blog.cvvliy.cn/Article/details/466686.sHtML
cn.blog.cvvliy.cn/Article/details/028028.sHtML
cn.blog.cvvliy.cn/Article/details/086862.sHtML
cn.blog.cvvliy.cn/Article/details/195735.sHtML
cn.blog.cvvliy.cn/Article/details/028244.sHtML
cn.blog.cvvliy.cn/Article/details/042022.sHtML
cn.blog.cvvliy.cn/Article/details/686800.sHtML
cn.blog.cvvliy.cn/Article/details/040228.sHtML
cn.blog.cvvliy.cn/Article/details/608468.sHtML
cn.blog.cvvliy.cn/Article/details/806808.sHtML
cn.blog.cvvliy.cn/Article/details/939575.sHtML
cn.blog.cvvliy.cn/Article/details/488886.sHtML
cn.blog.cvvliy.cn/Article/details/959997.sHtML
cn.blog.cvvliy.cn/Article/details/426064.sHtML
cn.blog.cvvliy.cn/Article/details/400666.sHtML
cn.blog.cvvliy.cn/Article/details/440828.sHtML
cn.blog.cvvliy.cn/Article/details/640288.sHtML
cn.blog.cvvliy.cn/Article/details/622842.sHtML
cn.blog.cvvliy.cn/Article/details/533779.sHtML
cn.blog.cvvliy.cn/Article/details/591555.sHtML
cn.blog.cvvliy.cn/Article/details/977979.sHtML
cn.blog.cvvliy.cn/Article/details/202428.sHtML
cn.blog.cvvliy.cn/Article/details/331997.sHtML
cn.blog.cvvliy.cn/Article/details/424062.sHtML
cn.blog.cvvliy.cn/Article/details/006284.sHtML
cn.blog.cvvliy.cn/Article/details/028844.sHtML
cn.blog.cvvliy.cn/Article/details/266604.sHtML
cn.blog.cvvliy.cn/Article/details/042864.sHtML
cn.blog.cvvliy.cn/Article/details/866264.sHtML
cn.blog.cvvliy.cn/Article/details/939979.sHtML
cn.blog.cvvliy.cn/Article/details/220240.sHtML
cn.blog.cvvliy.cn/Article/details/337913.sHtML
cn.blog.cvvliy.cn/Article/details/260466.sHtML
cn.blog.cvvliy.cn/Article/details/915513.sHtML
cn.blog.cvvliy.cn/Article/details/684426.sHtML
cn.blog.cvvliy.cn/Article/details/008080.sHtML
cn.blog.cvvliy.cn/Article/details/684220.sHtML
cn.blog.cvvliy.cn/Article/details/400084.sHtML
cn.blog.cvvliy.cn/Article/details/622624.sHtML
cn.blog.cvvliy.cn/Article/details/642424.sHtML
cn.blog.cvvliy.cn/Article/details/931717.sHtML
cn.blog.cvvliy.cn/Article/details/282266.sHtML
cn.blog.cvvliy.cn/Article/details/262226.sHtML
cn.blog.cvvliy.cn/Article/details/406624.sHtML
cn.blog.cvvliy.cn/Article/details/626468.sHtML
cn.blog.cvvliy.cn/Article/details/377511.sHtML
cn.blog.cvvliy.cn/Article/details/208440.sHtML
cn.blog.cvvliy.cn/Article/details/660208.sHtML
cn.blog.cvvliy.cn/Article/details/957577.sHtML
cn.blog.cvvliy.cn/Article/details/420660.sHtML
cn.blog.cvvliy.cn/Article/details/406206.sHtML
cn.blog.cvvliy.cn/Article/details/222082.sHtML
cn.blog.cvvliy.cn/Article/details/044406.sHtML
cn.blog.cvvliy.cn/Article/details/682668.sHtML
cn.blog.cvvliy.cn/Article/details/446008.sHtML
cn.blog.cvvliy.cn/Article/details/426486.sHtML
cn.blog.cvvliy.cn/Article/details/975713.sHtML
cn.blog.cvvliy.cn/Article/details/628646.sHtML
cn.blog.cvvliy.cn/Article/details/004248.sHtML
cn.blog.cvvliy.cn/Article/details/751339.sHtML
cn.blog.cvvliy.cn/Article/details/808268.sHtML
cn.blog.cvvliy.cn/Article/details/802680.sHtML
cn.blog.cvvliy.cn/Article/details/353179.sHtML
cn.blog.cvvliy.cn/Article/details/402244.sHtML
cn.blog.cvvliy.cn/Article/details/886004.sHtML
cn.blog.cvvliy.cn/Article/details/357193.sHtML
cn.blog.cvvliy.cn/Article/details/979713.sHtML
cn.blog.cvvliy.cn/Article/details/131715.sHtML
cn.blog.cvvliy.cn/Article/details/393777.sHtML
cn.blog.cvvliy.cn/Article/details/628266.sHtML
cn.blog.cvvliy.cn/Article/details/622042.sHtML
cn.blog.cvvliy.cn/Article/details/426686.sHtML
cn.blog.cvvliy.cn/Article/details/711993.sHtML
cn.blog.cvvliy.cn/Article/details/046802.sHtML
cn.blog.cvvliy.cn/Article/details/666806.sHtML
cn.blog.cvvliy.cn/Article/details/286840.sHtML
cn.blog.cvvliy.cn/Article/details/375955.sHtML
cn.blog.cvvliy.cn/Article/details/359551.sHtML
cn.blog.cvvliy.cn/Article/details/002240.sHtML
cn.blog.cvvliy.cn/Article/details/402084.sHtML
cn.blog.cvvliy.cn/Article/details/208666.sHtML
cn.blog.cvvliy.cn/Article/details/684466.sHtML
cn.blog.cvvliy.cn/Article/details/064866.sHtML
cn.blog.cvvliy.cn/Article/details/644202.sHtML
cn.blog.cvvliy.cn/Article/details/777377.sHtML
cn.blog.cvvliy.cn/Article/details/620422.sHtML
cn.blog.cvvliy.cn/Article/details/755931.sHtML
cn.blog.cvvliy.cn/Article/details/351915.sHtML
cn.blog.cvvliy.cn/Article/details/262648.sHtML
cn.blog.cvvliy.cn/Article/details/620482.sHtML
cn.blog.cvvliy.cn/Article/details/026422.sHtML
cn.blog.cvvliy.cn/Article/details/311175.sHtML
cn.blog.cvvliy.cn/Article/details/206804.sHtML
cn.blog.cvvliy.cn/Article/details/733137.sHtML
cn.blog.cvvliy.cn/Article/details/020402.sHtML
cn.blog.cvvliy.cn/Article/details/064646.sHtML
cn.blog.cvvliy.cn/Article/details/884082.sHtML
cn.blog.cvvliy.cn/Article/details/680862.sHtML
cn.blog.cvvliy.cn/Article/details/884842.sHtML
cn.blog.cvvliy.cn/Article/details/335597.sHtML
cn.blog.cvvliy.cn/Article/details/408008.sHtML
cn.blog.cvvliy.cn/Article/details/824220.sHtML
cn.blog.cvvliy.cn/Article/details/406428.sHtML
cn.blog.cvvliy.cn/Article/details/195351.sHtML
cn.blog.cvvliy.cn/Article/details/519751.sHtML
cn.blog.cvvliy.cn/Article/details/646884.sHtML
cn.blog.cvvliy.cn/Article/details/608628.sHtML
cn.blog.cvvliy.cn/Article/details/024688.sHtML
cn.blog.cvvliy.cn/Article/details/088044.sHtML
cn.blog.cvvliy.cn/Article/details/228428.sHtML
cn.blog.cvvliy.cn/Article/details/884264.sHtML
cn.blog.cvvliy.cn/Article/details/957377.sHtML
cn.blog.cvvliy.cn/Article/details/062402.sHtML
cn.blog.cvvliy.cn/Article/details/006028.sHtML
cn.blog.cvvliy.cn/Article/details/624086.sHtML
cn.blog.cvvliy.cn/Article/details/248042.sHtML
cn.blog.cvvliy.cn/Article/details/440208.sHtML
cn.blog.cvvliy.cn/Article/details/115753.sHtML
cn.blog.cvvliy.cn/Article/details/397795.sHtML
cn.blog.cvvliy.cn/Article/details/517735.sHtML
cn.blog.cvvliy.cn/Article/details/571739.sHtML
cn.blog.cvvliy.cn/Article/details/155111.sHtML
cn.blog.cvvliy.cn/Article/details/288486.sHtML
cn.blog.cvvliy.cn/Article/details/248242.sHtML
cn.blog.cvvliy.cn/Article/details/042280.sHtML
cn.blog.cvvliy.cn/Article/details/199575.sHtML
cn.blog.cvvliy.cn/Article/details/820800.sHtML
cn.blog.cvvliy.cn/Article/details/004624.sHtML
cn.blog.cvvliy.cn/Article/details/646008.sHtML
cn.blog.cvvliy.cn/Article/details/640686.sHtML
cn.blog.cvvliy.cn/Article/details/006666.sHtML
cn.blog.cvvliy.cn/Article/details/511153.sHtML
cn.blog.cvvliy.cn/Article/details/624808.sHtML
cn.blog.cvvliy.cn/Article/details/062428.sHtML
cn.blog.cvvliy.cn/Article/details/791315.sHtML
cn.blog.cvvliy.cn/Article/details/800084.sHtML
cn.blog.cvvliy.cn/Article/details/662844.sHtML
cn.blog.cvvliy.cn/Article/details/244666.sHtML
cn.blog.cvvliy.cn/Article/details/408200.sHtML
cn.blog.cvvliy.cn/Article/details/068008.sHtML
cn.blog.cvvliy.cn/Article/details/420804.sHtML
cn.blog.cvvliy.cn/Article/details/066048.sHtML
cn.blog.cvvliy.cn/Article/details/913599.sHtML
cn.blog.cvvliy.cn/Article/details/957995.sHtML
cn.blog.cvvliy.cn/Article/details/955137.sHtML
cn.blog.cvvliy.cn/Article/details/771355.sHtML
cn.blog.cvvliy.cn/Article/details/957737.sHtML
cn.blog.cvvliy.cn/Article/details/224048.sHtML
cn.blog.cvvliy.cn/Article/details/133995.sHtML
cn.blog.cvvliy.cn/Article/details/842408.sHtML
cn.blog.cvvliy.cn/Article/details/648620.sHtML
cn.blog.cvvliy.cn/Article/details/862848.sHtML
cn.blog.cvvliy.cn/Article/details/224428.sHtML
cn.blog.cvvliy.cn/Article/details/197959.sHtML
cn.blog.cvvliy.cn/Article/details/537397.sHtML
cn.blog.cvvliy.cn/Article/details/537339.sHtML
cn.blog.cvvliy.cn/Article/details/179539.sHtML
cn.blog.cvvliy.cn/Article/details/973135.sHtML
cn.blog.cvvliy.cn/Article/details/000208.sHtML
cn.blog.cvvliy.cn/Article/details/008620.sHtML
cn.blog.cvvliy.cn/Article/details/400262.sHtML
cn.blog.cvvliy.cn/Article/details/842842.sHtML
cn.blog.cvvliy.cn/Article/details/082404.sHtML
cn.blog.cvvliy.cn/Article/details/222640.sHtML
cn.blog.cvvliy.cn/Article/details/022068.sHtML
cn.blog.cvvliy.cn/Article/details/624000.sHtML
cn.blog.cvvliy.cn/Article/details/282008.sHtML
cn.blog.cvvliy.cn/Article/details/226682.sHtML
cn.blog.cvvliy.cn/Article/details/088060.sHtML
cn.blog.cvvliy.cn/Article/details/395559.sHtML
cn.blog.cvvliy.cn/Article/details/048808.sHtML
cn.blog.cvvliy.cn/Article/details/068204.sHtML
cn.blog.cvvliy.cn/Article/details/668222.sHtML
cn.blog.cvvliy.cn/Article/details/440600.sHtML
cn.blog.cvvliy.cn/Article/details/840080.sHtML
cn.blog.cvvliy.cn/Article/details/248288.sHtML
cn.blog.cvvliy.cn/Article/details/957319.sHtML
cn.blog.cvvliy.cn/Article/details/842480.sHtML
cn.blog.cvvliy.cn/Article/details/117971.sHtML
cn.blog.cvvliy.cn/Article/details/777597.sHtML
cn.blog.cvvliy.cn/Article/details/117353.sHtML
cn.blog.cvvliy.cn/Article/details/246422.sHtML
cn.blog.cvvliy.cn/Article/details/224002.sHtML
cn.blog.cvvliy.cn/Article/details/822400.sHtML
cn.blog.cvvliy.cn/Article/details/759377.sHtML
cn.blog.cvvliy.cn/Article/details/537191.sHtML
cn.blog.cvvliy.cn/Article/details/848424.sHtML
cn.blog.cvvliy.cn/Article/details/404288.sHtML
cn.blog.cvvliy.cn/Article/details/420682.sHtML
cn.blog.cvvliy.cn/Article/details/060048.sHtML
cn.blog.cvvliy.cn/Article/details/333513.sHtML
cn.blog.cvvliy.cn/Article/details/717555.sHtML
cn.blog.cvvliy.cn/Article/details/280028.sHtML
cn.blog.cvvliy.cn/Article/details/642288.sHtML
cn.blog.cvvliy.cn/Article/details/846008.sHtML
cn.blog.cvvliy.cn/Article/details/848826.sHtML
cn.blog.cvvliy.cn/Article/details/935377.sHtML
cn.blog.cvvliy.cn/Article/details/593335.sHtML
cn.blog.cvvliy.cn/Article/details/333337.sHtML
cn.blog.cvvliy.cn/Article/details/226622.sHtML
cn.blog.cvvliy.cn/Article/details/486686.sHtML
cn.blog.cvvliy.cn/Article/details/579917.sHtML
cn.blog.cvvliy.cn/Article/details/195393.sHtML
cn.blog.cvvliy.cn/Article/details/519775.sHtML
cn.blog.cvvliy.cn/Article/details/599977.sHtML
cn.blog.cvvliy.cn/Article/details/317533.sHtML
cn.blog.cvvliy.cn/Article/details/644204.sHtML
cn.blog.cvvliy.cn/Article/details/959335.sHtML
cn.blog.cvvliy.cn/Article/details/911133.sHtML
cn.blog.cvvliy.cn/Article/details/315153.sHtML
cn.blog.cvvliy.cn/Article/details/848688.sHtML
cn.blog.cvvliy.cn/Article/details/888064.sHtML
cn.blog.cvvliy.cn/Article/details/682086.sHtML
cn.blog.cvvliy.cn/Article/details/446284.sHtML
cn.blog.cvvliy.cn/Article/details/842426.sHtML
cn.blog.cvvliy.cn/Article/details/602220.sHtML
cn.blog.cvvliy.cn/Article/details/662268.sHtML
cn.blog.cvvliy.cn/Article/details/477739.sHtML
cn.blog.cvvliy.cn/Article/details/153951.sHtML
cn.blog.cvvliy.cn/Article/details/973795.sHtML
cn.blog.cvvliy.cn/Article/details/579755.sHtML
cn.blog.cvvliy.cn/Article/details/424886.sHtML
cn.blog.cvvliy.cn/Article/details/311179.sHtML
cn.blog.cvvliy.cn/Article/details/888062.sHtML
cn.blog.cvvliy.cn/Article/details/240822.sHtML
cn.blog.cvvliy.cn/Article/details/931533.sHtML
cn.blog.cvvliy.cn/Article/details/577999.sHtML
cn.blog.cvvliy.cn/Article/details/739591.sHtML
cn.blog.cvvliy.cn/Article/details/973179.sHtML
cn.blog.cvvliy.cn/Article/details/804228.sHtML
cn.blog.cvvliy.cn/Article/details/602000.sHtML
cn.blog.cvvliy.cn/Article/details/662408.sHtML
cn.blog.cvvliy.cn/Article/details/660660.sHtML
cn.blog.cvvliy.cn/Article/details/428804.sHtML
cn.blog.cvvliy.cn/Article/details/642406.sHtML
cn.blog.cvvliy.cn/Article/details/539977.sHtML
cn.blog.cvvliy.cn/Article/details/771591.sHtML
cn.blog.cvvliy.cn/Article/details/597113.sHtML
cn.blog.cvvliy.cn/Article/details/959333.sHtML
cn.blog.cvvliy.cn/Article/details/973135.sHtML
cn.blog.cvvliy.cn/Article/details/915559.sHtML
cn.blog.cvvliy.cn/Article/details/155519.sHtML
cn.blog.cvvliy.cn/Article/details/311795.sHtML
cn.blog.cvvliy.cn/Article/details/535151.sHtML
cn.blog.cvvliy.cn/Article/details/688444.sHtML
cn.blog.cvvliy.cn/Article/details/464202.sHtML
cn.blog.cvvliy.cn/Article/details/826202.sHtML
cn.blog.cvvliy.cn/Article/details/260446.sHtML
cn.blog.cvvliy.cn/Article/details/828082.sHtML
cn.blog.cvvliy.cn/Article/details/533513.sHtML
cn.blog.cvvliy.cn/Article/details/864400.sHtML
cn.blog.cvvliy.cn/Article/details/991359.sHtML
cn.blog.cvvliy.cn/Article/details/913179.sHtML
cn.blog.cvvliy.cn/Article/details/206264.sHtML
cn.blog.cvvliy.cn/Article/details/406428.sHtML
cn.blog.cvvliy.cn/Article/details/919753.sHtML
cn.blog.cvvliy.cn/Article/details/600426.sHtML
cn.blog.cvvliy.cn/Article/details/357755.sHtML
cn.blog.cvvliy.cn/Article/details/688068.sHtML
cn.blog.cvvliy.cn/Article/details/688480.sHtML
cn.blog.cvvliy.cn/Article/details/400846.sHtML
cn.blog.cvvliy.cn/Article/details/971555.sHtML
cn.blog.cvvliy.cn/Article/details/008404.sHtML
cn.blog.cvvliy.cn/Article/details/577997.sHtML
cn.blog.cvvliy.cn/Article/details/422024.sHtML
cn.blog.cvvliy.cn/Article/details/206426.sHtML
cn.blog.cvvliy.cn/Article/details/604000.sHtML
cn.blog.cvvliy.cn/Article/details/200608.sHtML
cn.blog.cvvliy.cn/Article/details/848220.sHtML
cn.blog.cvvliy.cn/Article/details/888688.sHtML
cn.blog.cvvliy.cn/Article/details/751135.sHtML
cn.blog.cvvliy.cn/Article/details/371171.sHtML
cn.blog.cvvliy.cn/Article/details/400482.sHtML
cn.blog.cvvliy.cn/Article/details/006664.sHtML
cn.blog.cvvliy.cn/Article/details/424400.sHtML
cn.blog.cvvliy.cn/Article/details/802024.sHtML
cn.blog.cvvliy.cn/Article/details/626280.sHtML
cn.blog.cvvliy.cn/Article/details/442660.sHtML
cn.blog.cvvliy.cn/Article/details/004602.sHtML
cn.blog.cvvliy.cn/Article/details/080246.sHtML
cn.blog.cvvliy.cn/Article/details/442088.sHtML
cn.blog.cvvliy.cn/Article/details/595379.sHtML
cn.blog.cvvliy.cn/Article/details/460008.sHtML
cn.blog.cvvliy.cn/Article/details/080000.sHtML
cn.blog.cvvliy.cn/Article/details/804220.sHtML
cn.blog.cvvliy.cn/Article/details/313395.sHtML
cn.blog.cvvliy.cn/Article/details/779539.sHtML
cn.blog.cvvliy.cn/Article/details/806648.sHtML
cn.blog.cvvliy.cn/Article/details/337559.sHtML
cn.blog.cvvliy.cn/Article/details/371793.sHtML
cn.blog.cvvliy.cn/Article/details/286008.sHtML
cn.blog.cvvliy.cn/Article/details/246442.sHtML
cn.blog.cvvliy.cn/Article/details/460240.sHtML
cn.blog.cvvliy.cn/Article/details/648820.sHtML
cn.blog.cvvliy.cn/Article/details/802482.sHtML
cn.blog.cvvliy.cn/Article/details/404882.sHtML
cn.blog.cvvliy.cn/Article/details/424028.sHtML
cn.blog.cvvliy.cn/Article/details/002844.sHtML
cn.blog.cvvliy.cn/Article/details/408606.sHtML
cn.blog.cvvliy.cn/Article/details/591715.sHtML
cn.blog.cvvliy.cn/Article/details/446624.sHtML
cn.blog.cvvliy.cn/Article/details/359195.sHtML
cn.blog.cvvliy.cn/Article/details/197399.sHtML
cn.blog.cvvliy.cn/Article/details/260420.sHtML
cn.blog.cvvliy.cn/Article/details/044220.sHtML
cn.blog.cvvliy.cn/Article/details/886244.sHtML
cn.blog.cvvliy.cn/Article/details/460006.sHtML
cn.blog.cvvliy.cn/Article/details/640604.sHtML
cn.blog.cvvliy.cn/Article/details/553571.sHtML
cn.blog.cvvliy.cn/Article/details/135397.sHtML
cn.blog.cvvliy.cn/Article/details/488866.sHtML
cn.blog.cvvliy.cn/Article/details/804284.sHtML
cn.blog.cvvliy.cn/Article/details/828020.sHtML
cn.blog.cvvliy.cn/Article/details/882468.sHtML
cn.blog.cvvliy.cn/Article/details/400680.sHtML
cn.blog.cvvliy.cn/Article/details/288808.sHtML
cn.blog.cvvliy.cn/Article/details/266608.sHtML
cn.blog.cvvliy.cn/Article/details/571113.sHtML
cn.blog.cvvliy.cn/Article/details/779557.sHtML
cn.blog.cvvliy.cn/Article/details/579535.sHtML
