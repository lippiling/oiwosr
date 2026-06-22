缆谠钩韶硕


Linux mtd_block_isbad坏块标记与BBT表管理

NAND Flash出厂时包含坏块，使用过程中也会产生新的坏块。Linux MTD子系统通过mtd_block_isbad接口检测坏块，并通过BBT（Bad Block Table）机制在内存和Flash上管理坏块信息。相关代码分布在drivers/mtd/mtdcore.c、drivers/mtd/nand/bbt.c和include/linux/mtd/bbt.h。

mtd_block_isbad是MTD层的标准接口，接受mtd_info指针和块地址（以字节为单位），返回0表示好块，1表示坏块，负数表示错误：

```c
int mtd_block_isbad(struct mtd_info *mtd, loff_t ofs)
{
    if (!mtd_can_have_bbt(mtd))
        return 0;
    if (ofs & (mtd->erasesize - 1))
        return -EINVAL;
    if (ofs >= mtd->size)
        return -EINVAL;

    return mtd->_block_isbad(mtd, ofs);
}
```

底层实现有两种方式。第一种是Scan-based BBT，在NAND Flash的OOB（Out-of-Band）区域扫描坏块标记（Bad Block Marker）。MLC/TLC NAND通常在OOB第0字节写入非0xFF表示坏块；SLC NAND在OOB第5字节或第0字节。第二种是Flash-based BBT，将BBT数据存储在Flash的专用块上，内核启动时读取BBT到内存，避免全盘扫描。

nand_bbt的核心数据结构是nand_bbt描述符，包括bbt（内存中的坏块表位图）、bbt_td（BBT描述符）、bbt_md（BBT镜像描述符）、chip（NAND chip结构体指针）等：

```c
struct nand_bbt {
    struct nand_chip *chip;
    int              *bbt;
    struct nand_bbt_descr *bbt_td;
    struct nand_bbt_descr *bbt_md;
    int              *bbt_ext;
    spinlock_t       lock;
    int              nwrites;
    int              version;
};
```

nand_bbt_descr结构体描述BBT在Flash上的存储位置和格式：

```c
struct nand_bbt_descr {
    int options;    /* NAND_BBT_LASTBLOCK, NAND_BBT_ABSPAGE等标志 */
    int pages[NAND_MAX_CHIPS]; /* BBT占用的页号 */
    int offs;       /* BBT标记在OOB内的偏移 */
    int veroffs;    /* BBT版本号在OOB内的偏移 */
    uint8_t version[NAND_MAX_CHIPS]; /* BBT版本 */
    int len;        /* BBT数据长度 */
    int maxblocks;  /* 最大坏块数量 */
    int reserved_block_code;
};
```

MTD_BBT_SCAN初始化函数在NAND驱动probe阶段决定使用哪种坏块管理策略。如果NAND芯片支持Flash-based BBT，nand_scan_bbt将调用nand_get_flash_type后执行bbt的初始化流程：

```c
int nand_scan_bbt(struct nand_chip *this, struct nand_bbt_descr *bd)
{
    struct mtd_info *mtd = nand_to_mtd(this);
    int len, ret;

    /* 计算BBT所需内存：block数/8字节 */
    len = mtd->size >> (this->phys_erase_shift + 3);
    this->bbt = kzalloc(len, GFP_KERNEL);
    if (!this->bbt)
        return -ENOMEM;

    if (bd == NULL)
        bd = &bbt_main_descr;

    /* 尝试从Flash读取已存在的BBT */
    ret = read_bbt(this, bd, 0);
    if (ret < 0)
        return ret;

    /* 读取镜像BBT */
    ret = read_bbt(this, bd, 1);

    return 0;
}
```

read_bbt函数从Flash读取BBT数据。对于Flash-based BBT，它遍历BBT描述符中的pages数组，读取保存在Flash中的BBT并验证CRC。如果Flash中没有有效的BBT，则回退到scanning模式，扫描所有块的OOB区域查找坏块标记：

```c
static int read_bbt(struct nand_chip *this, struct nand_bbt_descr *bd, int chip)
{
    struct mtd_info *mtd = nand_to_mtd(this);
    uint8_t *buf;
    int page, ret;

    buf = kmalloc(mtd->writesize, GFP_KERNEL);
    if (!buf)
        return -ENOMEM;

    for (page = bd->pages[chip]; page < bd->pages[chip] + bd->maxblocks; page++) {
        ret = mtd_read(mtd, page * mtd->writesize, mtd->writesize, &retlen, buf);
        if (ret == 0 && bbt_check_pattern(buf, bd) == 0) {
            /* 找到有效的BBT条目 */
            bbt_decode_pattern(this, buf, bd, chip);
            kfree(buf);
            return 0;
        }
    }

    /* 未找到有效BBT，执行scan */
    create_bbt(this, bd, chip);
    kfree(buf);
    return 0;
}
```

create_bbt函数扫描Flash上所有块的OOB区域。对于每个块，读取第一页（或第二页，取决于配置）的OOB，检查坏块标记字节：

```c
static int create_bbt(struct nand_chip *this, struct nand_bbt_descr *bd, int chip)
{
    struct mtd_info *mtd = nand_to_mtd(this);
    int i, numblocks, startblock, ret;
    uint8_t *buf;

    numblocks = mtd->size >> this->phys_erase_shift;
    buf = kmalloc(mtd->writesize + mtd->oobsize, GFP_KERNEL);
    if (!buf)
        return -ENOMEM;

    for (i = startblock; i < numblocks; i++) {
        loff_t offs = (loff_t)i << this->phys_erase_shift;

        /* 读取OOB区域检查坏块标记 */
        ret = mtd_read_oob(mtd, offs, &ops);
        if (ret)
            continue;

        if (check_short_pattern(buf, bd))
            this->bbt[i >> 3] |= (0x1 << (i & 0x07));
    }

    kfree(buf);
    return 0;
}
```

check_short_pattern函数检查OOB区域中特定偏移处的字节是否为0xFF。如果不是0xFF（通常为0x00），则标记该块为坏块。MLC NAND除了检查第一页OOB之外，还需要检查第二页OOB，因为MLC的坏块标记需要两个页一起确认。

nand_block_isbad是mtd_block_isbad在NAND层对应的实现。它不做实际Flash读取，仅查询内存中的bbt数组：

```c
static int nand_block_isbad(struct mtd_info *mtd, loff_t offs)
{
    struct nand_chip *chip = mtd_to_nand(mtd);
    int block;

    block = (int)(offs >> chip->phys_erase_shift);
    if (chip->bbt[block >> 3] & (0x1 << (block & 0x07)))
        return 1;

    return 0;
}
```

当文件系统或UBI需要标记一个新发现的坏块时，调用mtd_block_markbad。该函数通过nand_block_markbad实现，在OOB中写入坏块标记，并更新内存中的bbt位图。如果是Flash-based BBT，还会触发BBT写入Flash操作：

```c
int nand_block_markbad(struct mtd_info *mtd, loff_t ofs)
{
    struct nand_chip *chip = mtd_to_nand(mtd);
    int block, ret;

    block = (int)(ofs >> chip->phys_erase_shift);

    /* 标记内存中的bbt */
    chip->bbt[block >> 3] |= (0x1 << (block & 0x07));

    /* 写入OOB坏块标记 */
    ret = nand_write_badblock_marker(chip, ofs);
    if (ret)
        return ret;

    /* 更新Flash上的BBT */
    if (chip->bbt_td) {
        ret = update_bbt(chip, NAND_BBT_WRITE);
        if (ret)
            return ret;
    }

    return 0;
}
```

BBT表在Flash上的存储使用两个副本（主BBT和镜像BBT）以防止数据丢失。每个副本位于不同的物理块中，且在这些块的最后几页存储。BBT版本号字段用于在启动时选择最新的副本。如果主副本损坏，自动回退到镜像副本，如果都损坏则重新执行全盘扫描。

搜索镜像块和备用块使用的nand_bbt_descr中的options标志位控制行为。NAND_BBT_LASTBLOCK标志表示BBT存储在分区最后一块，NAND_BBT_ABSPAGE标志表示存储在绝对页号位置，NAND_BBT_PERCHIP表示每个chip有独立的BBT。对于多die NAND芯片，每个die维护独立的BBT。

胺澜辖鞍窘瞻始谖乒徘藏估普哨匾

qiv.mmmxz.cn/397713.htm
qiv.mmmxz.cn/151793.htm
qiv.mmmxz.cn/139133.htm
qiv.mmmxz.cn/919733.htm
qiv.mmmxz.cn/551153.htm
qiv.mmmxz.cn/537973.htm
qiv.mmmxz.cn/313593.htm
qiv.mmmxz.cn/971573.htm
qiv.mmmxz.cn/517733.htm
qiv.mmmxz.cn/644603.htm
qic.mmmxz.cn/046083.htm
qic.mmmxz.cn/393793.htm
qic.mmmxz.cn/713373.htm
qic.mmmxz.cn/355733.htm
qic.mmmxz.cn/804023.htm
qic.mmmxz.cn/593173.htm
qic.mmmxz.cn/995573.htm
qic.mmmxz.cn/353753.htm
qic.mmmxz.cn/559733.htm
qic.mmmxz.cn/535913.htm
qix.mmmxz.cn/753913.htm
qix.mmmxz.cn/880643.htm
qix.mmmxz.cn/480883.htm
qix.mmmxz.cn/553313.htm
qix.mmmxz.cn/595793.htm
qix.mmmxz.cn/571773.htm
qix.mmmxz.cn/282863.htm
qix.mmmxz.cn/953993.htm
qix.mmmxz.cn/759153.htm
qix.mmmxz.cn/719793.htm
qiz.mmmxz.cn/844083.htm
qiz.mmmxz.cn/371933.htm
qiz.mmmxz.cn/751193.htm
qiz.mmmxz.cn/379333.htm
qiz.mmmxz.cn/062623.htm
qiz.mmmxz.cn/268603.htm
qiz.mmmxz.cn/559793.htm
qiz.mmmxz.cn/775553.htm
qiz.mmmxz.cn/759753.htm
qiz.mmmxz.cn/791993.htm
qil.mmmxz.cn/026043.htm
qil.mmmxz.cn/777373.htm
qil.mmmxz.cn/195553.htm
qil.mmmxz.cn/537333.htm
qil.mmmxz.cn/266443.htm
qil.mmmxz.cn/173193.htm
qil.mmmxz.cn/719133.htm
qil.mmmxz.cn/862423.htm
qil.mmmxz.cn/555313.htm
qil.mmmxz.cn/993993.htm
qik.mmmxz.cn/313753.htm
qik.mmmxz.cn/935153.htm
qik.mmmxz.cn/939713.htm
qik.mmmxz.cn/371953.htm
qik.mmmxz.cn/995553.htm
qik.mmmxz.cn/193173.htm
qik.mmmxz.cn/395773.htm
qik.mmmxz.cn/795133.htm
qik.mmmxz.cn/040043.htm
qik.mmmxz.cn/599133.htm
qij.mmmxz.cn/195973.htm
qij.mmmxz.cn/884603.htm
qij.mmmxz.cn/060463.htm
qij.mmmxz.cn/139353.htm
qij.mmmxz.cn/935733.htm
qij.mmmxz.cn/135553.htm
qij.mmmxz.cn/220063.htm
qij.mmmxz.cn/777973.htm
qij.mmmxz.cn/311313.htm
qij.mmmxz.cn/393393.htm
qih.mmmxz.cn/884203.htm
qih.mmmxz.cn/717353.htm
qih.mmmxz.cn/731953.htm
qih.mmmxz.cn/171753.htm
qih.mmmxz.cn/975353.htm
qih.mmmxz.cn/935353.htm
qih.mmmxz.cn/199193.htm
qih.mmmxz.cn/977953.htm
qih.mmmxz.cn/157793.htm
qih.mmmxz.cn/755373.htm
qig.mmmxz.cn/888403.htm
qig.mmmxz.cn/511573.htm
qig.mmmxz.cn/395553.htm
qig.mmmxz.cn/335973.htm
qig.mmmxz.cn/684843.htm
qig.mmmxz.cn/131913.htm
qig.mmmxz.cn/335953.htm
qig.mmmxz.cn/282443.htm
qig.mmmxz.cn/246643.htm
qig.mmmxz.cn/977913.htm
qif.mmmxz.cn/135553.htm
qif.mmmxz.cn/953313.htm
qif.mmmxz.cn/313533.htm
qif.mmmxz.cn/393973.htm
qif.mmmxz.cn/713173.htm
qif.mmmxz.cn/371993.htm
qif.mmmxz.cn/442263.htm
qif.mmmxz.cn/244023.htm
qif.mmmxz.cn/428803.htm
qif.mmmxz.cn/379533.htm
qid.mmmxz.cn/799553.htm
qid.mmmxz.cn/371573.htm
qid.mmmxz.cn/220403.htm
qid.mmmxz.cn/597593.htm
qid.mmmxz.cn/157553.htm
qid.mmmxz.cn/915133.htm
qid.mmmxz.cn/282283.htm
qid.mmmxz.cn/319313.htm
qid.mmmxz.cn/535593.htm
qid.mmmxz.cn/393153.htm
qis.mmmxz.cn/311333.htm
qis.mmmxz.cn/353953.htm
qis.mmmxz.cn/599593.htm
qis.mmmxz.cn/559533.htm
qis.mmmxz.cn/335153.htm
qis.mmmxz.cn/248023.htm
qis.mmmxz.cn/713313.htm
qis.mmmxz.cn/195713.htm
qis.mmmxz.cn/915513.htm
qis.mmmxz.cn/466003.htm
qia.mmmxz.cn/977333.htm
qia.mmmxz.cn/393533.htm
qia.mmmxz.cn/137533.htm
qia.mmmxz.cn/660403.htm
qia.mmmxz.cn/286223.htm
qia.mmmxz.cn/577313.htm
qia.mmmxz.cn/559973.htm
qia.mmmxz.cn/917753.htm
qia.mmmxz.cn/335973.htm
qia.mmmxz.cn/608863.htm
qip.mmmxz.cn/531193.htm
qip.mmmxz.cn/933373.htm
qip.mmmxz.cn/406683.htm
qip.mmmxz.cn/826843.htm
qip.mmmxz.cn/488483.htm
qip.mmmxz.cn/991193.htm
qip.mmmxz.cn/971393.htm
qip.mmmxz.cn/737973.htm
qip.mmmxz.cn/866443.htm
qip.mmmxz.cn/759753.htm
qio.mmmxz.cn/604463.htm
qio.mmmxz.cn/333733.htm
qio.mmmxz.cn/791593.htm
qio.mmmxz.cn/660283.htm
qio.mmmxz.cn/935393.htm
qio.mmmxz.cn/802403.htm
qio.mmmxz.cn/775133.htm
qio.mmmxz.cn/751533.htm
qio.mmmxz.cn/595733.htm
qio.mmmxz.cn/662223.htm
qii.mmmxz.cn/266423.htm
qii.mmmxz.cn/797373.htm
qii.mmmxz.cn/599513.htm
qii.mmmxz.cn/539933.htm
qii.mmmxz.cn/313573.htm
qii.mmmxz.cn/066483.htm
qii.mmmxz.cn/284483.htm
qii.mmmxz.cn/175793.htm
qii.mmmxz.cn/317153.htm
qii.mmmxz.cn/620023.htm
qiu.mmmxz.cn/202203.htm
qiu.mmmxz.cn/137193.htm
qiu.mmmxz.cn/135373.htm
qiu.mmmxz.cn/711333.htm
qiu.mmmxz.cn/319313.htm
qiu.mmmxz.cn/993773.htm
qiu.mmmxz.cn/753753.htm
qiu.mmmxz.cn/824443.htm
qiu.mmmxz.cn/799933.htm
qiu.mmmxz.cn/422623.htm
qiy.mmmxz.cn/517153.htm
qiy.mmmxz.cn/640463.htm
qiy.mmmxz.cn/353973.htm
qiy.mmmxz.cn/197353.htm
qiy.mmmxz.cn/957373.htm
qiy.mmmxz.cn/222283.htm
qiy.mmmxz.cn/062423.htm
qiy.mmmxz.cn/913953.htm
qiy.mmmxz.cn/771373.htm
qiy.mmmxz.cn/353173.htm
qit.mmmxz.cn/248623.htm
qit.mmmxz.cn/799933.htm
qit.mmmxz.cn/420463.htm
qit.mmmxz.cn/379173.htm
qit.mmmxz.cn/733753.htm
qit.mmmxz.cn/024403.htm
qit.mmmxz.cn/779933.htm
qit.mmmxz.cn/913553.htm
qit.mmmxz.cn/993313.htm
qit.mmmxz.cn/208203.htm
qir.mmmxz.cn/957133.htm
qir.mmmxz.cn/115713.htm
qir.mmmxz.cn/448243.htm
qir.mmmxz.cn/279533.htm
qir.mmmxz.cn/048103.htm
qir.mmmxz.cn/830013.htm
qir.mmmxz.cn/457473.htm
qir.mmmxz.cn/517853.htm
qir.mmmxz.cn/170103.htm
qir.mmmxz.cn/043883.htm
qie.mmmxz.cn/139093.htm
qie.mmmxz.cn/397773.htm
qie.mmmxz.cn/815983.htm
qie.mmmxz.cn/667033.htm
qie.mmmxz.cn/056343.htm
qie.mmmxz.cn/844893.htm
qie.mmmxz.cn/512093.htm
qie.mmmxz.cn/993423.htm
qie.mmmxz.cn/979133.htm
qie.mmmxz.cn/706563.htm
qiw.mmmxz.cn/087903.htm
qiw.mmmxz.cn/982823.htm
qiw.mmmxz.cn/409323.htm
qiw.mmmxz.cn/789603.htm
qiw.mmmxz.cn/231103.htm
qiw.mmmxz.cn/385423.htm
qiw.mmmxz.cn/210633.htm
qiw.mmmxz.cn/110333.htm
qiw.mmmxz.cn/147763.htm
qiw.mmmxz.cn/698873.htm
qiq.mmmxz.cn/402203.htm
qiq.mmmxz.cn/650693.htm
qiq.mmmxz.cn/806233.htm
qiq.mmmxz.cn/181173.htm
qiq.mmmxz.cn/471733.htm
qiq.mmmxz.cn/318093.htm
qiq.mmmxz.cn/814763.htm
qiq.mmmxz.cn/258773.htm
qiq.mmmxz.cn/867893.htm
qiq.mmmxz.cn/361533.htm
qum.mmmxz.cn/174873.htm
qum.mmmxz.cn/351583.htm
qum.mmmxz.cn/719333.htm
qum.mmmxz.cn/973183.htm
qum.mmmxz.cn/062743.htm
qum.mmmxz.cn/500313.htm
qum.mmmxz.cn/796583.htm
qum.mmmxz.cn/131053.htm
qum.mmmxz.cn/595493.htm
qum.mmmxz.cn/867703.htm
qun.mmmxz.cn/383543.htm
qun.mmmxz.cn/718103.htm
qun.mmmxz.cn/478863.htm
qun.mmmxz.cn/003513.htm
qun.mmmxz.cn/910613.htm
qun.mmmxz.cn/866273.htm
qun.mmmxz.cn/259543.htm
qun.mmmxz.cn/097003.htm
qun.mmmxz.cn/608353.htm
qun.mmmxz.cn/511273.htm
qub.mmmxz.cn/524603.htm
qub.mmmxz.cn/892403.htm
qub.mmmxz.cn/709983.htm
qub.mmmxz.cn/898043.htm
qub.mmmxz.cn/090413.htm
qub.mmmxz.cn/53.htm
qub.mmmxz.cn/655623.htm
qub.mmmxz.cn/232223.htm
qub.mmmxz.cn/325013.htm
qub.mmmxz.cn/073673.htm
quv.mmmxz.cn/144973.htm
quv.mmmxz.cn/231333.htm
quv.mmmxz.cn/838673.htm
quv.mmmxz.cn/803893.htm
quv.mmmxz.cn/378893.htm
quv.mmmxz.cn/286483.htm
quv.mmmxz.cn/468663.htm
quv.mmmxz.cn/941813.htm
quv.mmmxz.cn/386693.htm
quv.mmmxz.cn/745643.htm
quc.mmmxz.cn/700263.htm
quc.mmmxz.cn/634873.htm
quc.mmmxz.cn/663473.htm
quc.mmmxz.cn/824513.htm
quc.mmmxz.cn/181943.htm
quc.mmmxz.cn/470093.htm
quc.mmmxz.cn/337553.htm
quc.mmmxz.cn/080283.htm
quc.mmmxz.cn/603473.htm
quc.mmmxz.cn/458133.htm
qux.mmmxz.cn/952303.htm
qux.mmmxz.cn/955633.htm
qux.mmmxz.cn/465153.htm
qux.mmmxz.cn/765773.htm
qux.mmmxz.cn/852943.htm
qux.mmmxz.cn/791553.htm
qux.mmmxz.cn/762503.htm
qux.mmmxz.cn/832033.htm
qux.mmmxz.cn/996613.htm
qux.mmmxz.cn/560843.htm
quz.mmmxz.cn/826443.htm
quz.mmmxz.cn/156043.htm
quz.mmmxz.cn/400713.htm
quz.mmmxz.cn/456663.htm
quz.mmmxz.cn/401103.htm
quz.mmmxz.cn/006773.htm
quz.mmmxz.cn/458263.htm
quz.mmmxz.cn/404353.htm
quz.mmmxz.cn/204673.htm
quz.mmmxz.cn/404723.htm
qul.mmmxz.cn/570083.htm
qul.mmmxz.cn/416383.htm
qul.mmmxz.cn/552873.htm
qul.mmmxz.cn/967263.htm
qul.mmmxz.cn/863243.htm
qul.mmmxz.cn/603423.htm
qul.mmmxz.cn/295753.htm
qul.mmmxz.cn/044173.htm
qul.mmmxz.cn/067153.htm
qul.mmmxz.cn/234963.htm
quk.mmmxz.cn/093783.htm
quk.mmmxz.cn/543333.htm
quk.mmmxz.cn/996273.htm
quk.mmmxz.cn/080213.htm
quk.mmmxz.cn/863183.htm
quk.mmmxz.cn/169043.htm
quk.mmmxz.cn/071083.htm
quk.mmmxz.cn/522363.htm
quk.mmmxz.cn/859683.htm
quk.mmmxz.cn/781403.htm
quj.mmmxz.cn/705553.htm
quj.mmmxz.cn/372443.htm
quj.mmmxz.cn/809903.htm
quj.mmmxz.cn/037753.htm
quj.mmmxz.cn/919013.htm
quj.mmmxz.cn/123853.htm
quj.mmmxz.cn/355533.htm
quj.mmmxz.cn/597903.htm
quj.mmmxz.cn/039503.htm
quj.mmmxz.cn/737633.htm
quh.mmmxz.cn/109493.htm
quh.mmmxz.cn/833563.htm
quh.mmmxz.cn/552853.htm
quh.mmmxz.cn/795123.htm
quh.mmmxz.cn/535043.htm
quh.mmmxz.cn/658933.htm
quh.mmmxz.cn/024853.htm
quh.mmmxz.cn/463313.htm
quh.mmmxz.cn/446913.htm
quh.mmmxz.cn/161043.htm
qug.mmmxz.cn/518563.htm
qug.mmmxz.cn/673543.htm
qug.mmmxz.cn/093443.htm
qug.mmmxz.cn/831333.htm
qug.mmmxz.cn/278773.htm
qug.mmmxz.cn/851503.htm
qug.mmmxz.cn/673953.htm
qug.mmmxz.cn/709433.htm
qug.mmmxz.cn/610393.htm
qug.mmmxz.cn/466593.htm
quf.mmmxz.cn/528123.htm
quf.mmmxz.cn/557053.htm
quf.mmmxz.cn/037483.htm
quf.mmmxz.cn/865113.htm
quf.mmmxz.cn/987263.htm
quf.mmmxz.cn/675453.htm
quf.mmmxz.cn/341273.htm
quf.mmmxz.cn/409913.htm
quf.mmmxz.cn/426533.htm
quf.mmmxz.cn/664893.htm
qud.mmmxz.cn/690683.htm
qud.mmmxz.cn/159913.htm
qud.mmmxz.cn/895633.htm
qud.mmmxz.cn/555303.htm
qud.mmmxz.cn/233613.htm
qud.mmmxz.cn/251893.htm
qud.mmmxz.cn/150923.htm
qud.mmmxz.cn/611633.htm
qud.mmmxz.cn/711153.htm
qud.mmmxz.cn/531303.htm
qus.mmmxz.cn/763113.htm
qus.mmmxz.cn/122273.htm
qus.mmmxz.cn/733833.htm
qus.mmmxz.cn/713093.htm
qus.mmmxz.cn/545993.htm
qus.mmmxz.cn/258143.htm
qus.mmmxz.cn/112343.htm
qus.mmmxz.cn/019343.htm
qus.mmmxz.cn/378213.htm
qus.mmmxz.cn/215453.htm
qua.mmmxz.cn/573243.htm
qua.mmmxz.cn/881803.htm
qua.mmmxz.cn/518023.htm
qua.mmmxz.cn/557163.htm
qua.mmmxz.cn/237923.htm
qua.mmmxz.cn/331873.htm
qua.mmmxz.cn/787423.htm
qua.mmmxz.cn/671283.htm
qua.mmmxz.cn/192633.htm
qua.mmmxz.cn/819793.htm
qup.mmmxz.cn/605903.htm
qup.mmmxz.cn/262693.htm
qup.mmmxz.cn/045863.htm
qup.mmmxz.cn/141173.htm
qup.mmmxz.cn/524233.htm
