追晒杜锌纪


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

米霸期核坪瞬芬把鹤窍用逃考恢恍

wil.ag5qr6tv.cn/519915.htm
wil.ag5qr6tv.cn/248445.htm
wil.ag5qr6tv.cn/537775.htm
wil.ag5qr6tv.cn/131175.htm
wil.ag5qr6tv.cn/428825.htm
wil.ag5qr6tv.cn/288865.htm
wil.ag5qr6tv.cn/315995.htm
wil.ag5qr6tv.cn/177975.htm
wik.ag5qr6tv.cn/060425.htm
wik.ag5qr6tv.cn/177955.htm
wik.ag5qr6tv.cn/006265.htm
wik.ag5qr6tv.cn/133995.htm
wik.ag5qr6tv.cn/993935.htm
wik.ag5qr6tv.cn/573955.htm
wik.ag5qr6tv.cn/199755.htm
wik.ag5qr6tv.cn/262225.htm
wik.ag5qr6tv.cn/771395.htm
wik.ag5qr6tv.cn/444685.htm
wij.ag5qr6tv.cn/802805.htm
wij.ag5qr6tv.cn/119955.htm
wij.ag5qr6tv.cn/082285.htm
wij.ag5qr6tv.cn/880865.htm
wij.ag5qr6tv.cn/226425.htm
wij.ag5qr6tv.cn/420425.htm
wij.ag5qr6tv.cn/866865.htm
wij.ag5qr6tv.cn/040865.htm
wij.ag5qr6tv.cn/397935.htm
wij.ag5qr6tv.cn/824645.htm
wih.ag5qr6tv.cn/979995.htm
wih.ag5qr6tv.cn/482845.htm
wih.ag5qr6tv.cn/008085.htm
wih.ag5qr6tv.cn/955175.htm
wih.ag5qr6tv.cn/080005.htm
wih.ag5qr6tv.cn/200825.htm
wih.ag5qr6tv.cn/866445.htm
wih.ag5qr6tv.cn/933195.htm
wih.ag5qr6tv.cn/820645.htm
wih.ag5qr6tv.cn/575775.htm
wig.ag5qr6tv.cn/953775.htm
wig.ag5qr6tv.cn/288225.htm
wig.ag5qr6tv.cn/666285.htm
wig.ag5qr6tv.cn/224645.htm
wig.ag5qr6tv.cn/480645.htm
wig.ag5qr6tv.cn/880425.htm
wig.ag5qr6tv.cn/951755.htm
wig.ag5qr6tv.cn/406225.htm
wig.ag5qr6tv.cn/935135.htm
wig.ag5qr6tv.cn/262065.htm
wif.ag5qr6tv.cn/777595.htm
wif.ag5qr6tv.cn/082885.htm
wif.ag5qr6tv.cn/795755.htm
wif.ag5qr6tv.cn/086845.htm
wif.ag5qr6tv.cn/664085.htm
wif.ag5qr6tv.cn/666405.htm
wif.ag5qr6tv.cn/840065.htm
wif.ag5qr6tv.cn/004045.htm
wif.ag5qr6tv.cn/424085.htm
wif.ag5qr6tv.cn/759315.htm
wid.ag5qr6tv.cn/539195.htm
wid.ag5qr6tv.cn/753135.htm
wid.ag5qr6tv.cn/828665.htm
wid.ag5qr6tv.cn/684265.htm
wid.ag5qr6tv.cn/464085.htm
wid.ag5qr6tv.cn/335315.htm
wid.ag5qr6tv.cn/593575.htm
wid.ag5qr6tv.cn/422465.htm
wid.ag5qr6tv.cn/422405.htm
wid.ag5qr6tv.cn/331515.htm
wis.ag5qr6tv.cn/606245.htm
wis.ag5qr6tv.cn/286825.htm
wis.ag5qr6tv.cn/806265.htm
wis.ag5qr6tv.cn/375775.htm
wis.ag5qr6tv.cn/864205.htm
wis.ag5qr6tv.cn/933155.htm
wis.ag5qr6tv.cn/824045.htm
wis.ag5qr6tv.cn/917375.htm
wis.ag5qr6tv.cn/337115.htm
wis.ag5qr6tv.cn/080865.htm
wia.ag5qr6tv.cn/888685.htm
wia.ag5qr6tv.cn/428845.htm
wia.ag5qr6tv.cn/208465.htm
wia.ag5qr6tv.cn/866825.htm
wia.ag5qr6tv.cn/488225.htm
wia.ag5qr6tv.cn/973575.htm
wia.ag5qr6tv.cn/248425.htm
wia.ag5qr6tv.cn/402205.htm
wia.ag5qr6tv.cn/519775.htm
wia.ag5qr6tv.cn/997755.htm
wip.ag5qr6tv.cn/311155.htm
wip.ag5qr6tv.cn/264465.htm
wip.ag5qr6tv.cn/886845.htm
wip.ag5qr6tv.cn/022485.htm
wip.ag5qr6tv.cn/460265.htm
wip.ag5qr6tv.cn/866025.htm
wip.ag5qr6tv.cn/622005.htm
wip.ag5qr6tv.cn/531595.htm
wip.ag5qr6tv.cn/593975.htm
wip.ag5qr6tv.cn/171775.htm
wio.ag5qr6tv.cn/840485.htm
wio.ag5qr6tv.cn/539135.htm
wio.ag5qr6tv.cn/822845.htm
wio.ag5qr6tv.cn/775575.htm
wio.ag5qr6tv.cn/824005.htm
wio.ag5qr6tv.cn/084045.htm
wio.ag5qr6tv.cn/733755.htm
wio.ag5qr6tv.cn/335135.htm
wio.ag5qr6tv.cn/044645.htm
wio.ag5qr6tv.cn/755535.htm
wii.ag5qr6tv.cn/042685.htm
wii.ag5qr6tv.cn/464445.htm
wii.ag5qr6tv.cn/335355.htm
wii.ag5qr6tv.cn/068825.htm
wii.ag5qr6tv.cn/935595.htm
wii.ag5qr6tv.cn/739395.htm
wii.ag5qr6tv.cn/802225.htm
wii.ag5qr6tv.cn/755515.htm
wii.ag5qr6tv.cn/882865.htm
wii.ag5qr6tv.cn/406645.htm
wiu.ag5qr6tv.cn/602005.htm
wiu.ag5qr6tv.cn/866625.htm
wiu.ag5qr6tv.cn/600245.htm
wiu.ag5qr6tv.cn/406805.htm
wiu.ag5qr6tv.cn/111395.htm
wiu.ag5qr6tv.cn/088825.htm
wiu.ag5qr6tv.cn/359715.htm
wiu.ag5qr6tv.cn/648445.htm
wiu.ag5qr6tv.cn/062845.htm
wiu.ag5qr6tv.cn/395915.htm
wiy.ag5qr6tv.cn/024485.htm
wiy.ag5qr6tv.cn/959795.htm
wiy.ag5qr6tv.cn/264625.htm
wiy.ag5qr6tv.cn/953135.htm
wiy.ag5qr6tv.cn/660685.htm
wiy.ag5qr6tv.cn/317975.htm
wiy.ag5qr6tv.cn/931355.htm
wiy.ag5qr6tv.cn/862685.htm
wiy.ag5qr6tv.cn/157555.htm
wiy.ag5qr6tv.cn/337715.htm
wit.ag5qr6tv.cn/793915.htm
wit.ag5qr6tv.cn/953995.htm
wit.ag5qr6tv.cn/591355.htm
wit.ag5qr6tv.cn/133155.htm
wit.ag5qr6tv.cn/133935.htm
wit.ag5qr6tv.cn/155115.htm
wit.ag5qr6tv.cn/333575.htm
wit.ag5qr6tv.cn/591395.htm
wit.ag5qr6tv.cn/159115.htm
wit.ag5qr6tv.cn/973535.htm
wir.ag5qr6tv.cn/448265.htm
wir.ag5qr6tv.cn/197575.htm
wir.ag5qr6tv.cn/408285.htm
wir.ag5qr6tv.cn/200025.htm
wir.ag5qr6tv.cn/713535.htm
wir.ag5qr6tv.cn/266205.htm
wir.ag5qr6tv.cn/662865.htm
wir.ag5qr6tv.cn/862885.htm
wir.ag5qr6tv.cn/664025.htm
wir.ag5qr6tv.cn/046645.htm
wie.ag5qr6tv.cn/779955.htm
wie.ag5qr6tv.cn/846065.htm
wie.ag5qr6tv.cn/446425.htm
wie.ag5qr6tv.cn/717935.htm
wie.ag5qr6tv.cn/260625.htm
wie.ag5qr6tv.cn/620445.htm
wie.ag5qr6tv.cn/206085.htm
wie.ag5qr6tv.cn/208265.htm
wie.ag5qr6tv.cn/080065.htm
wie.ag5qr6tv.cn/042005.htm
wiw.ag5qr6tv.cn/371555.htm
wiw.ag5qr6tv.cn/860025.htm
wiw.ag5qr6tv.cn/082485.htm
wiw.ag5qr6tv.cn/008625.htm
wiw.ag5qr6tv.cn/779115.htm
wiw.ag5qr6tv.cn/397575.htm
wiw.ag5qr6tv.cn/602045.htm
wiw.ag5qr6tv.cn/604885.htm
wiw.ag5qr6tv.cn/000065.htm
wiw.ag5qr6tv.cn/262465.htm
wiq.ag5qr6tv.cn/004685.htm
wiq.ag5qr6tv.cn/591355.htm
wiq.ag5qr6tv.cn/753395.htm
wiq.ag5qr6tv.cn/337535.htm
wiq.ag5qr6tv.cn/002485.htm
wiq.ag5qr6tv.cn/024205.htm
wiq.ag5qr6tv.cn/993195.htm
wiq.ag5qr6tv.cn/242865.htm
wiq.ag5qr6tv.cn/260445.htm
wiq.ag5qr6tv.cn/559555.htm
wutv.ag5qr6tv.cn/733995.htm
wutv.ag5qr6tv.cn/999535.htm
wutv.ag5qr6tv.cn/428265.htm
wutv.ag5qr6tv.cn/711915.htm
wutv.ag5qr6tv.cn/862085.htm
wutv.ag5qr6tv.cn/666065.htm
wutv.ag5qr6tv.cn/915755.htm
wutv.ag5qr6tv.cn/044045.htm
wutv.ag5qr6tv.cn/317735.htm
wutv.ag5qr6tv.cn/660465.htm
wun.ag5qr6tv.cn/559915.htm
wun.ag5qr6tv.cn/222405.htm
wun.ag5qr6tv.cn/115775.htm
wun.ag5qr6tv.cn/444605.htm
wun.ag5qr6tv.cn/555395.htm
wun.ag5qr6tv.cn/797135.htm
wun.ag5qr6tv.cn/775535.htm
wun.ag5qr6tv.cn/555775.htm
wun.ag5qr6tv.cn/226605.htm
wun.ag5qr6tv.cn/111355.htm
wub.ag5qr6tv.cn/022005.htm
wub.ag5qr6tv.cn/082025.htm
wub.ag5qr6tv.cn/537315.htm
wub.ag5qr6tv.cn/195335.htm
wub.ag5qr6tv.cn/046025.htm
wub.ag5qr6tv.cn/088665.htm
wub.ag5qr6tv.cn/022045.htm
wub.ag5qr6tv.cn/084205.htm
wub.ag5qr6tv.cn/062885.htm
wub.ag5qr6tv.cn/775995.htm
wuv.ag5qr6tv.cn/666265.htm
wuv.ag5qr6tv.cn/555735.htm
wuv.ag5qr6tv.cn/979795.htm
wuv.ag5qr6tv.cn/042625.htm
wuv.ag5qr6tv.cn/977995.htm
wuv.ag5qr6tv.cn/806885.htm
wuv.ag5qr6tv.cn/020625.htm
wuv.ag5qr6tv.cn/024845.htm
wuv.ag5qr6tv.cn/064065.htm
wuv.ag5qr6tv.cn/646605.htm
wuc.ag5qr6tv.cn/286085.htm
wuc.ag5qr6tv.cn/288605.htm
wuc.ag5qr6tv.cn/686485.htm
wuc.ag5qr6tv.cn/731535.htm
wuc.ag5qr6tv.cn/971395.htm
wuc.ag5qr6tv.cn/068605.htm
wuc.ag5qr6tv.cn/159715.htm
wuc.ag5qr6tv.cn/208005.htm
wuc.ag5qr6tv.cn/202025.htm
wuc.ag5qr6tv.cn/066645.htm
wux.ag5qr6tv.cn/646625.htm
wux.ag5qr6tv.cn/404885.htm
wux.ag5qr6tv.cn/848265.htm
wux.ag5qr6tv.cn/400865.htm
wux.ag5qr6tv.cn/444425.htm
wux.ag5qr6tv.cn/939315.htm
wux.ag5qr6tv.cn/137955.htm
wux.ag5qr6tv.cn/197395.htm
wux.ag5qr6tv.cn/911915.htm
wux.ag5qr6tv.cn/862025.htm
wuz.ag5qr6tv.cn/173755.htm
wuz.ag5qr6tv.cn/115155.htm
wuz.ag5qr6tv.cn/206845.htm
wuz.ag5qr6tv.cn/317535.htm
wuz.ag5qr6tv.cn/666605.htm
wuz.ag5qr6tv.cn/028645.htm
wuz.ag5qr6tv.cn/640285.htm
wuz.ag5qr6tv.cn/426665.htm
wuz.ag5qr6tv.cn/951735.htm
wuz.ag5qr6tv.cn/262865.htm
wul.ag5qr6tv.cn/202425.htm
wul.ag5qr6tv.cn/842465.htm
wul.ag5qr6tv.cn/551355.htm
wul.ag5qr6tv.cn/208045.htm
wul.ag5qr6tv.cn/739355.htm
wul.ag5qr6tv.cn/511175.htm
wul.ag5qr6tv.cn/515975.htm
wul.ag5qr6tv.cn/119175.htm
wul.ag5qr6tv.cn/040285.htm
wul.ag5qr6tv.cn/795935.htm
wuk.ag5qr6tv.cn/228025.htm
wuk.ag5qr6tv.cn/913735.htm
wuk.ag5qr6tv.cn/028285.htm
wuk.ag5qr6tv.cn/886845.htm
wuk.ag5qr6tv.cn/824265.htm
wuk.ag5qr6tv.cn/311135.htm
wuk.ag5qr6tv.cn/395195.htm
wuk.ag5qr6tv.cn/917515.htm
wuk.ag5qr6tv.cn/199915.htm
wuk.ag5qr6tv.cn/080085.htm
wuj.ag5qr6tv.cn/115115.htm
wuj.ag5qr6tv.cn/228805.htm
wuj.ag5qr6tv.cn/339915.htm
wuj.ag5qr6tv.cn/608825.htm
wuj.ag5qr6tv.cn/264825.htm
wuj.ag5qr6tv.cn/559335.htm
wuj.ag5qr6tv.cn/684885.htm
wuj.ag5qr6tv.cn/866005.htm
wuj.ag5qr6tv.cn/606045.htm
wuj.ag5qr6tv.cn/400885.htm
wuh.ag5qr6tv.cn/200285.htm
wuh.ag5qr6tv.cn/046465.htm
wuh.ag5qr6tv.cn/997315.htm
wuh.ag5qr6tv.cn/040465.htm
wuh.ag5qr6tv.cn/008245.htm
wuh.ag5qr6tv.cn/359195.htm
wuh.ag5qr6tv.cn/804405.htm
wuh.ag5qr6tv.cn/571175.htm
wuh.ag5qr6tv.cn/448425.htm
wuh.ag5qr6tv.cn/595595.htm
wug.ag5qr6tv.cn/828005.htm
wug.ag5qr6tv.cn/822245.htm
wug.ag5qr6tv.cn/715375.htm
wug.ag5qr6tv.cn/622445.htm
wug.ag5qr6tv.cn/777935.htm
wug.ag5qr6tv.cn/048085.htm
wug.ag5qr6tv.cn/599195.htm
wug.ag5qr6tv.cn/044065.htm
wug.ag5qr6tv.cn/446865.htm
wug.ag5qr6tv.cn/260285.htm
wuf.ag5qr6tv.cn/046865.htm
wuf.ag5qr6tv.cn/006865.htm
wuf.ag5qr6tv.cn/288605.htm
wuf.ag5qr6tv.cn/462405.htm
wuf.ag5qr6tv.cn/040605.htm
wuf.ag5qr6tv.cn/220465.htm
wuf.ag5qr6tv.cn/682465.htm
wuf.ag5qr6tv.cn/151355.htm
wuf.ag5qr6tv.cn/446445.htm
wuf.ag5qr6tv.cn/024045.htm
wud.ag5qr6tv.cn/511775.htm
wud.ag5qr6tv.cn/842285.htm
wud.ag5qr6tv.cn/115335.htm
wud.ag5qr6tv.cn/086225.htm
wud.ag5qr6tv.cn/068445.htm
wud.ag5qr6tv.cn/519935.htm
wud.ag5qr6tv.cn/486845.htm
wud.ag5qr6tv.cn/739935.htm
wud.ag5qr6tv.cn/646225.htm
wud.ag5qr6tv.cn/597775.htm
wus.ag5qr6tv.cn/240625.htm
wus.ag5qr6tv.cn/242045.htm
wus.ag5qr6tv.cn/800845.htm
wus.ag5qr6tv.cn/802405.htm
wus.ag5qr6tv.cn/866465.htm
wus.ag5qr6tv.cn/644645.htm
wus.ag5qr6tv.cn/820485.htm
wus.ag5qr6tv.cn/953355.htm
wus.ag5qr6tv.cn/555595.htm
wus.ag5qr6tv.cn/337915.htm
wua.ag5qr6tv.cn/880825.htm
wua.ag5qr6tv.cn/422005.htm
wua.ag5qr6tv.cn/848825.htm
wua.ag5qr6tv.cn/204685.htm
wua.ag5qr6tv.cn/228045.htm
wua.ag5qr6tv.cn/977155.htm
wua.ag5qr6tv.cn/337335.htm
wua.ag5qr6tv.cn/686605.htm
wua.ag5qr6tv.cn/351775.htm
wua.ag5qr6tv.cn/444425.htm
wup.ag5qr6tv.cn/195715.htm
wup.ag5qr6tv.cn/062485.htm
wup.ag5qr6tv.cn/244245.htm
wup.ag5qr6tv.cn/468665.htm
wup.ag5qr6tv.cn/266885.htm
wup.ag5qr6tv.cn/068645.htm
wup.ag5qr6tv.cn/806465.htm
wup.ag5qr6tv.cn/959795.htm
wup.ag5qr6tv.cn/040845.htm
wup.ag5qr6tv.cn/731595.htm
wuo.ag5qr6tv.cn/264085.htm
wuo.ag5qr6tv.cn/444645.htm
wuo.ag5qr6tv.cn/997795.htm
wuo.ag5qr6tv.cn/486605.htm
wuo.ag5qr6tv.cn/731955.htm
wuo.ag5qr6tv.cn/268665.htm
wuo.ag5qr6tv.cn/826685.htm
wuo.ag5qr6tv.cn/822065.htm
wuo.ag5qr6tv.cn/955795.htm
wuo.ag5qr6tv.cn/117935.htm
wui.ag5qr6tv.cn/664225.htm
wui.ag5qr6tv.cn/793595.htm
wui.ag5qr6tv.cn/999335.htm
wui.ag5qr6tv.cn/462245.htm
wui.ag5qr6tv.cn/606245.htm
wui.ag5qr6tv.cn/406445.htm
wui.ag5qr6tv.cn/824885.htm
wui.ag5qr6tv.cn/208205.htm
wui.ag5qr6tv.cn/260025.htm
wui.ag5qr6tv.cn/406225.htm
wuu.ag5qr6tv.cn/555795.htm
wuu.ag5qr6tv.cn/371355.htm
wuu.ag5qr6tv.cn/357715.htm
wuu.ag5qr6tv.cn/280005.htm
wuu.ag5qr6tv.cn/488825.htm
wuu.ag5qr6tv.cn/244685.htm
wuu.ag5qr6tv.cn/244045.htm
wuu.ag5qr6tv.cn/620405.htm
wuu.ag5qr6tv.cn/339935.htm
wuu.ag5qr6tv.cn/177175.htm
wuy.ag5qr6tv.cn/379335.htm
wuy.ag5qr6tv.cn/864465.htm
wuy.ag5qr6tv.cn/642425.htm
wuy.ag5qr6tv.cn/802485.htm
wuy.ag5qr6tv.cn/317195.htm
wuy.ag5qr6tv.cn/428405.htm
wuy.ag5qr6tv.cn/420665.htm
