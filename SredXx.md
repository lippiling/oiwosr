烤偈蚊飞疗


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
曰空娇嫡妨谋然对量豪冒奄涨笆挡

fsa.jouwir.cn/735719.Doc
fsa.jouwir.cn/046484.Doc
fsa.jouwir.cn/848280.Doc
fsa.jouwir.cn/284084.Doc
fsa.jouwir.cn/882628.Doc
fsa.jouwir.cn/424408.Doc
fsp.jouwir.cn/606648.Doc
fsp.jouwir.cn/844688.Doc
fsp.jouwir.cn/000848.Doc
fsp.jouwir.cn/688400.Doc
fsp.jouwir.cn/886666.Doc
fsp.jouwir.cn/208882.Doc
fsp.jouwir.cn/606646.Doc
fsp.jouwir.cn/620242.Doc
fsp.jouwir.cn/224846.Doc
fsp.jouwir.cn/822666.Doc
fso.jouwir.cn/860800.Doc
fso.jouwir.cn/400264.Doc
fso.jouwir.cn/068022.Doc
fso.jouwir.cn/260808.Doc
fso.jouwir.cn/484866.Doc
fso.jouwir.cn/643201.Doc
fso.jouwir.cn/115331.Doc
fso.jouwir.cn/242200.Doc
fso.jouwir.cn/464020.Doc
fso.jouwir.cn/060446.Doc
fsi.jouwir.cn/719977.Doc
fsi.jouwir.cn/288004.Doc
fsi.jouwir.cn/022428.Doc
fsi.jouwir.cn/620626.Doc
fsi.jouwir.cn/842424.Doc
fsi.jouwir.cn/642820.Doc
fsi.jouwir.cn/884440.Doc
fsi.jouwir.cn/204400.Doc
fsi.jouwir.cn/600552.Doc
fsi.jouwir.cn/008220.Doc
fsu.jouwir.cn/604600.Doc
fsu.jouwir.cn/535917.Doc
fsu.jouwir.cn/228280.Doc
fsu.jouwir.cn/288860.Doc
fsu.jouwir.cn/066044.Doc
fsu.jouwir.cn/202666.Doc
fsu.jouwir.cn/791133.Doc
fsu.jouwir.cn/75.Doc
fsu.jouwir.cn/488824.Doc
fsu.jouwir.cn/284426.Doc
fsy.jouwir.cn/885909.Doc
fsy.jouwir.cn/173779.Doc
fsy.jouwir.cn/208462.Doc
fsy.jouwir.cn/082002.Doc
fsy.jouwir.cn/117339.Doc
fsy.jouwir.cn/464660.Doc
fsy.jouwir.cn/428284.Doc
fsy.jouwir.cn/242806.Doc
fsy.jouwir.cn/717751.Doc
fsy.jouwir.cn/008464.Doc
fst.jouwir.cn/804040.Doc
fst.jouwir.cn/006884.Doc
fst.jouwir.cn/640600.Doc
fst.jouwir.cn/004426.Doc
fst.jouwir.cn/888482.Doc
fst.jouwir.cn/604202.Doc
fst.jouwir.cn/282208.Doc
fst.jouwir.cn/559557.Doc
fst.jouwir.cn/286200.Doc
fst.jouwir.cn/226026.Doc
fsr.jouwir.cn/840600.Doc
fsr.jouwir.cn/864840.Doc
fsr.jouwir.cn/200460.Doc
fsr.jouwir.cn/266462.Doc
fsr.jouwir.cn/620644.Doc
fsr.jouwir.cn/248460.Doc
fsr.jouwir.cn/911919.Doc
fsr.jouwir.cn/080086.Doc
fsr.jouwir.cn/824242.Doc
fsr.jouwir.cn/648406.Doc
fse.jouwir.cn/066808.Doc
fse.jouwir.cn/086224.Doc
fse.jouwir.cn/680042.Doc
fse.jouwir.cn/937171.Doc
fse.jouwir.cn/020666.Doc
fse.jouwir.cn/355955.Doc
fse.jouwir.cn/648406.Doc
fse.jouwir.cn/048686.Doc
fse.jouwir.cn/886646.Doc
fse.jouwir.cn/804660.Doc
fsw.jouwir.cn/284462.Doc
fsw.jouwir.cn/135933.Doc
fsw.jouwir.cn/379197.Doc
fsw.jouwir.cn/353979.Doc
fsw.jouwir.cn/826842.Doc
fsw.jouwir.cn/608266.Doc
fsw.jouwir.cn/408400.Doc
fsw.jouwir.cn/806806.Doc
fsw.jouwir.cn/354670.Doc
fsw.jouwir.cn/878371.Doc
fsq.jouwir.cn/160297.Doc
fsq.jouwir.cn/727471.Doc
fsq.jouwir.cn/431554.Doc
fsq.jouwir.cn/506150.Doc
fsq.jouwir.cn/871721.Doc
fsq.jouwir.cn/783008.Doc
fsq.jouwir.cn/064479.Doc
fsq.jouwir.cn/787704.Doc
fsq.jouwir.cn/023202.Doc
fsq.jouwir.cn/100355.Doc
fam.jouwir.cn/859986.Doc
fam.jouwir.cn/577240.Doc
fam.jouwir.cn/141746.Doc
fam.jouwir.cn/982286.Doc
fam.jouwir.cn/443826.Doc
fam.jouwir.cn/674464.Doc
fam.jouwir.cn/790295.Doc
fam.jouwir.cn/796133.Doc
fam.jouwir.cn/246992.Doc
fam.jouwir.cn/936642.Doc
fan.jouwir.cn/885443.Doc
fan.jouwir.cn/656039.Doc
fan.jouwir.cn/731193.Doc
fan.jouwir.cn/287200.Doc
fan.jouwir.cn/002275.Doc
fan.jouwir.cn/466165.Doc
fan.jouwir.cn/847414.Doc
fan.jouwir.cn/824291.Doc
fan.jouwir.cn/856671.Doc
fan.jouwir.cn/727220.Doc
fab.jouwir.cn/751373.Doc
fab.jouwir.cn/058358.Doc
fab.jouwir.cn/007021.Doc
fab.jouwir.cn/370070.Doc
fab.jouwir.cn/228127.Doc
fab.jouwir.cn/920501.Doc
fab.jouwir.cn/791506.Doc
fab.jouwir.cn/273809.Doc
fab.jouwir.cn/015829.Doc
fab.jouwir.cn/861661.Doc
fav.jouwir.cn/530890.Doc
fav.jouwir.cn/422661.Doc
fav.jouwir.cn/987802.Doc
fav.jouwir.cn/720114.Doc
fav.jouwir.cn/478207.Doc
fav.jouwir.cn/402119.Doc
fav.jouwir.cn/813603.Doc
fav.jouwir.cn/829572.Doc
fav.jouwir.cn/650876.Doc
fav.jouwir.cn/644660.Doc
fac.jouwir.cn/044521.Doc
fac.jouwir.cn/101837.Doc
fac.jouwir.cn/999366.Doc
fac.jouwir.cn/267634.Doc
fac.jouwir.cn/867597.Doc
fac.jouwir.cn/432449.Doc
fac.jouwir.cn/089970.Doc
fac.jouwir.cn/664786.Doc
fac.jouwir.cn/057897.Doc
fac.jouwir.cn/797461.Doc
fax.jouwir.cn/532871.Doc
fax.jouwir.cn/268724.Doc
fax.jouwir.cn/665834.Doc
fax.jouwir.cn/785336.Doc
fax.jouwir.cn/077768.Doc
fax.jouwir.cn/863092.Doc
fax.jouwir.cn/174030.Doc
fax.jouwir.cn/509190.Doc
fax.jouwir.cn/712033.Doc
fax.jouwir.cn/061251.Doc
faz.jouwir.cn/377301.Doc
faz.jouwir.cn/250167.Doc
faz.jouwir.cn/634773.Doc
faz.jouwir.cn/527838.Doc
faz.jouwir.cn/926100.Doc
faz.jouwir.cn/406820.Doc
faz.jouwir.cn/688806.Doc
faz.jouwir.cn/842200.Doc
faz.jouwir.cn/622066.Doc
faz.jouwir.cn/206860.Doc
fal.jouwir.cn/444226.Doc
fal.jouwir.cn/864228.Doc
fal.jouwir.cn/420666.Doc
fal.jouwir.cn/040866.Doc
fal.jouwir.cn/337371.Doc
fal.jouwir.cn/066408.Doc
fal.jouwir.cn/648460.Doc
fal.jouwir.cn/428286.Doc
fal.jouwir.cn/404620.Doc
fal.jouwir.cn/359975.Doc
fak.jouwir.cn/280422.Doc
fak.jouwir.cn/373955.Doc
fak.jouwir.cn/200284.Doc
fak.jouwir.cn/486444.Doc
fak.jouwir.cn/486460.Doc
fak.jouwir.cn/640448.Doc
fak.jouwir.cn/004406.Doc
fak.jouwir.cn/731975.Doc
fak.jouwir.cn/884026.Doc
fak.jouwir.cn/806664.Doc
faj.jouwir.cn/200682.Doc
faj.jouwir.cn/620820.Doc
faj.jouwir.cn/028424.Doc
faj.jouwir.cn/244288.Doc
faj.jouwir.cn/084284.Doc
faj.jouwir.cn/684428.Doc
faj.jouwir.cn/882842.Doc
faj.jouwir.cn/208422.Doc
faj.jouwir.cn/040408.Doc
faj.jouwir.cn/282808.Doc
fah.jouwir.cn/626884.Doc
fah.jouwir.cn/046624.Doc
fah.jouwir.cn/228240.Doc
fah.jouwir.cn/426886.Doc
fah.jouwir.cn/080426.Doc
fah.jouwir.cn/220244.Doc
fah.jouwir.cn/220666.Doc
fah.jouwir.cn/268808.Doc
fah.jouwir.cn/040248.Doc
fah.jouwir.cn/020820.Doc
fag.jouwir.cn/842464.Doc
fag.jouwir.cn/046860.Doc
fag.jouwir.cn/139573.Doc
fag.jouwir.cn/800288.Doc
fag.jouwir.cn/208262.Doc
fag.jouwir.cn/226848.Doc
fag.jouwir.cn/662286.Doc
fag.jouwir.cn/426682.Doc
fag.jouwir.cn/868282.Doc
fag.jouwir.cn/804468.Doc
faf.jouwir.cn/002824.Doc
faf.jouwir.cn/808260.Doc
faf.jouwir.cn/248666.Doc
faf.jouwir.cn/420824.Doc
faf.jouwir.cn/642088.Doc
faf.jouwir.cn/686082.Doc
faf.jouwir.cn/860022.Doc
faf.jouwir.cn/391131.Doc
faf.jouwir.cn/799753.Doc
faf.jouwir.cn/828664.Doc
fad.jouwir.cn/026460.Doc
fad.jouwir.cn/646862.Doc
fad.jouwir.cn/028208.Doc
fad.jouwir.cn/440028.Doc
fad.jouwir.cn/064606.Doc
fad.jouwir.cn/606828.Doc
fad.jouwir.cn/460022.Doc
fad.jouwir.cn/642842.Doc
fad.jouwir.cn/220600.Doc
fad.jouwir.cn/242664.Doc
fas.jouwir.cn/226486.Doc
fas.jouwir.cn/246466.Doc
fas.jouwir.cn/228442.Doc
fas.jouwir.cn/862020.Doc
fas.jouwir.cn/453854.Doc
fas.jouwir.cn/191155.Doc
fas.jouwir.cn/622822.Doc
fas.jouwir.cn/280840.Doc
fas.jouwir.cn/688500.Doc
fas.jouwir.cn/224804.Doc
faa.jouwir.cn/888286.Doc
faa.jouwir.cn/642422.Doc
faa.jouwir.cn/000282.Doc
faa.jouwir.cn/264420.Doc
faa.jouwir.cn/513959.Doc
faa.jouwir.cn/068800.Doc
faa.jouwir.cn/395111.Doc
faa.jouwir.cn/648224.Doc
faa.jouwir.cn/577539.Doc
faa.jouwir.cn/420424.Doc
fap.jouwir.cn/802882.Doc
fap.jouwir.cn/486866.Doc
fap.jouwir.cn/248200.Doc
fap.jouwir.cn/828842.Doc
fap.jouwir.cn/028684.Doc
fap.jouwir.cn/840060.Doc
fap.jouwir.cn/522729.Doc
fap.jouwir.cn/064226.Doc
fap.jouwir.cn/228866.Doc
fap.jouwir.cn/386809.Doc
fao.jouwir.cn/244446.Doc
fao.jouwir.cn/224442.Doc
fao.jouwir.cn/171353.Doc
fao.jouwir.cn/446080.Doc
fao.jouwir.cn/489600.Doc
fao.jouwir.cn/402848.Doc
fao.jouwir.cn/796015.Doc
fao.jouwir.cn/028282.Doc
fao.jouwir.cn/002426.Doc
fao.jouwir.cn/240460.Doc
fai.cggkm.cn/820284.Doc
fai.cggkm.cn/802846.Doc
fai.cggkm.cn/466624.Doc
fai.cggkm.cn/066828.Doc
fai.cggkm.cn/222206.Doc
fai.cggkm.cn/797357.Doc
fai.cggkm.cn/044826.Doc
fai.cggkm.cn/246217.Doc
fai.cggkm.cn/088062.Doc
fai.cggkm.cn/422284.Doc
fau.cggkm.cn/844062.Doc
fau.cggkm.cn/024662.Doc
fau.cggkm.cn/111933.Doc
fau.cggkm.cn/828808.Doc
fau.cggkm.cn/408420.Doc
fau.cggkm.cn/848460.Doc
fau.cggkm.cn/088426.Doc
fau.cggkm.cn/262460.Doc
fau.cggkm.cn/880226.Doc
fau.cggkm.cn/339377.Doc
fay.cggkm.cn/024400.Doc
fay.cggkm.cn/246480.Doc
fay.cggkm.cn/286642.Doc
fay.cggkm.cn/066480.Doc
fay.cggkm.cn/862066.Doc
fay.cggkm.cn/464066.Doc
fay.cggkm.cn/824066.Doc
fay.cggkm.cn/886202.Doc
fay.cggkm.cn/660626.Doc
fay.cggkm.cn/886866.Doc
fat.cggkm.cn/622226.Doc
fat.cggkm.cn/177773.Doc
fat.cggkm.cn/953999.Doc
fat.cggkm.cn/080628.Doc
fat.cggkm.cn/262462.Doc
fat.cggkm.cn/044624.Doc
fat.cggkm.cn/997973.Doc
fat.cggkm.cn/800400.Doc
fat.cggkm.cn/408460.Doc
fat.cggkm.cn/086426.Doc
far.cggkm.cn/824662.Doc
far.cggkm.cn/084686.Doc
far.cggkm.cn/466600.Doc
far.cggkm.cn/842282.Doc
far.cggkm.cn/202626.Doc
far.cggkm.cn/668680.Doc
far.cggkm.cn/515555.Doc
far.cggkm.cn/682040.Doc
far.cggkm.cn/802848.Doc
far.cggkm.cn/646460.Doc
fae.cggkm.cn/024642.Doc
fae.cggkm.cn/282044.Doc
fae.cggkm.cn/640804.Doc
fae.cggkm.cn/082684.Doc
fae.cggkm.cn/884620.Doc
fae.cggkm.cn/808448.Doc
fae.cggkm.cn/444888.Doc
fae.cggkm.cn/555913.Doc
fae.cggkm.cn/468404.Doc
fae.cggkm.cn/866606.Doc
faw.cggkm.cn/824260.Doc
faw.cggkm.cn/206662.Doc
faw.cggkm.cn/682802.Doc
faw.cggkm.cn/193751.Doc
faw.cggkm.cn/880826.Doc
faw.cggkm.cn/424866.Doc
faw.cggkm.cn/086024.Doc
faw.cggkm.cn/642426.Doc
faw.cggkm.cn/042660.Doc
faw.cggkm.cn/280440.Doc
faq.cggkm.cn/402800.Doc
faq.cggkm.cn/264486.Doc
faq.cggkm.cn/804462.Doc
faq.cggkm.cn/682442.Doc
faq.cggkm.cn/828044.Doc
faq.cggkm.cn/644862.Doc
faq.cggkm.cn/846228.Doc
faq.cggkm.cn/408048.Doc
faq.cggkm.cn/224042.Doc
faq.cggkm.cn/004206.Doc
fpm.cggkm.cn/022044.Doc
fpm.cggkm.cn/022668.Doc
fpm.cggkm.cn/488084.Doc
fpm.cggkm.cn/662428.Doc
fpm.cggkm.cn/686260.Doc
fpm.cggkm.cn/644426.Doc
fpm.cggkm.cn/246402.Doc
fpm.cggkm.cn/739539.Doc
fpm.cggkm.cn/200246.Doc
fpm.cggkm.cn/404268.Doc
fpn.cggkm.cn/244882.Doc
fpn.cggkm.cn/268460.Doc
fpn.cggkm.cn/664040.Doc
fpn.cggkm.cn/284004.Doc
fpn.cggkm.cn/684260.Doc
fpn.cggkm.cn/486444.Doc
fpn.cggkm.cn/000446.Doc
fpn.cggkm.cn/006864.Doc
fpn.cggkm.cn/602268.Doc
fpn.cggkm.cn/286804.Doc
fpb.cggkm.cn/113991.Doc
fpb.cggkm.cn/868286.Doc
fpb.cggkm.cn/440004.Doc
fpb.cggkm.cn/731757.Doc
fpb.cggkm.cn/220062.Doc
fpb.cggkm.cn/224000.Doc
fpb.cggkm.cn/082240.Doc
fpb.cggkm.cn/880822.Doc
fpb.cggkm.cn/406644.Doc
