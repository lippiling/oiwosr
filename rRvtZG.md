扔中湍偻冻


Linux mtd_concat MTD分区合并与mtd_info抽象

mtd_concat是Linux MTD子系统中的一个虚拟驱动，位于drivers/mtd/mtdconcat.c，它通过mtd_info抽象层将多个物理上不连续的MTD分区合并成一个连续的虚拟MTD设备。上层的UBI、JFFS2等文件系统无需感知底层分区的物理分布，直接操作合并后的统一mtd_info接口。

核心数据结构是struct mtd_concat，包含num_subdev（子设备数量）、subdev（子设备mtd_info指针数组）以及一个mtd_info实例。这个mtd_info中的几乎所有函数指针都被重写为concat版本的实现：

```c
struct mtd_concat {
    struct mtd_info mtd;
    int num_subdev;
    struct mtd_info **subdev;
};
```

mtd_concat的创建函数是concat_mtd_devices，它接收子设备指针数组和数量，返回虚拟mtd_info指针：

```c
struct mtd_info *concat_mtd_devices(struct mtd_info *devices[],
                                    int num_devices)
{
    struct mtd_concat *concat;
    struct mtd_info *mtd;
    int i;
    uint64_t size = 0;

    concat = kzalloc(sizeof(*concat), GFP_KERNEL);
    if (!concat)
        return ERR_PTR(-ENOMEM);

    concat->subdev = devices;
    concat->num_subdev = num_devices;
    mtd = &concat->mtd;
    mtd->priv = concat;

    /* 计算总大小，使用最大擦除块大小 */
    for (i = 0; i < num_devices; i++) {
        size += devices[i]->size;
        if (devices[i]->erasesize > mtd->erasesize)
            mtd->erasesize = devices[i]->erasesize;
    }

    mtd->size = size;
    mtd->type = devices[0]->type;
    mtd->flags = devices[0]->flags;
    mtd->name = kasprintf(GFP_KERNEL, "concat-%d", num_devices);
    mtd->owner = THIS_MODULE;

    /* 注册虚拟MTD设备 */
    ret = mtd_device_register(mtd, NULL, 0);
    return mtd;
}
```

concat设备最关键的是读写擦除地址转换逻辑。所有传入的offset被分解为子设备索引和子设备内偏移量，通过concat_get_device函数实现：

```c
static int concat_get_device(struct mtd_concat *concat,
                             uint64_t *offset, int *subdev_index)
{
    int i;

    for (i = 0; i < concat->num_subdev; i++) {
        if (*offset < concat->subdev[i]->size) {
            *subdev_index = i;
            return 0;
        }
        *offset -= concat->subdev[i]->size;
    }
    return -EINVAL;
}
```

concat_read函数利用上述地址转换，定位到具体的子设备后调用子设备的_read接口。跨分区边界读取时，需要分段读取并拷贝到合并缓冲区：

```c
static int concat_read(struct mtd_info *mtd, loff_t from, size_t len,
                       size_t *retlen, u_char *buf)
{
    struct mtd_concat *concat = mtd->priv;
    int err = -EINVAL, i;
    size_t sub_retlen, total_retlen = 0;

    for (i = 0; i < concat->num_subdev; i++) {
        struct mtd_info *subdev = concat->subdev[i];
        size_t size;

        if (from >= subdev->size) {
            from -= subdev->size;
            continue;
        }

        size = min_t(uint64_t, len, subdev->size - from);
        err = mtd_read(subdev, from, size, &sub_retlen, buf + total_retlen);
        if (err)
            break;

        total_retlen += sub_retlen;
        len -= sub_retlen;
        from = 0;

        if (len == 0)
            break;
    }

    *retlen = total_retlen;
    return err;
}
```

concat_write、concat_erase、concat_read_oob、concat_write_oob等函数遵循相同的分发模式。对于擦除操作（concat_erase_addr），要求擦除范围必须与子设备的erasesize对齐，并且不能跨越子设备边界。如果跨边界擦除，concat实现会将其拆分为多个擦除指令依次下发：

```c
static int concat_erase(struct mtd_info *mtd, struct erase_info *instr)
{
    struct mtd_concat *concat = mtd->priv;
    uint64_t addr = instr->addr;
    size_t len = instr->len;
    int err = 0;

    while (len) {
        struct mtd_info *subdev;
        uint64_t sub_offset;
        size_t erase_len;
        int idx;

        concat_get_device(concat, &addr, &idx);
        subdev = concat->subdev[idx];
        sub_offset = addr;

        erase_len = min_t(size_t, len,
                          subdev->size - sub_offset);
        /* 对齐到擦除块大小 */
        erase_len &= ~((loff_t)subdev->erasesize - 1);

        instr->addr = sub_offset;
        instr->len = erase_len;
        err = mtd_erase(subdev, instr);
        if (err)
            return err;

        addr += erase_len;
        len -= erase_len;
    }
    return 0;
}
```

mtd_concat对mtd_info抽象层的填充还包括多点读写（_read_oob,_write_oob）的支持。OOB数据区域（带外数据，存储ECC校验码和坏块标记）在concat设备中的访问同样需要地址转换。如果某个子设备不支持OOB，concat对该子设备区域的OOB操作返回-EOPNOTSUPP。

concat设备的锁机制通过子设备锁实现。concat_lock遍历所有子设备调用mtd->_lock，concat_unlock相同。由于合并设备可能包含不同特性的子设备，concat_flags在构造时使用第一个子设备的flags，但可能无法完全反映所有子设备的能力差异。上层文件系统在使用时需要通过mtd->flags检查统一的能力位掩码。

mtd_concat的一个典型使用场景是SPI NOR Flash的Remapping机制。当系统的两个SPI NOR片选映射到同一地址空间时，通过concat将其合并为一个连续的MTD设备，UBIFS或JFFS2可以直接在其上运行。另一个场景是NAND Flash的坏块分区隔离：将好块区域合并成一个连续分区供UBI使用，坏块区域由其他机制管理。

mtd_concat不处理任何坏块管理，它依赖子MTD设备自身实现坏块处理。当子设备报告的坏块信息影响大小计算时，上层UBI会通过自己的坏块处理机制（UBI scan）进行检测，而不是依赖concat层。mtd_concat通过mtd_can_have_ppb等函数传递子设备的页编程能力，但不会主动合并或对齐各子设备的能力差异，这部分由使用concat的上层模块自行保证。

两把追讼谈坪廖使乓于涸桶朴继导

tv.blog.itaadf.cn/Article/details/462286.sHtML
tv.blog.itaadf.cn/Article/details/466606.sHtML
tv.blog.itaadf.cn/Article/details/462040.sHtML
tv.blog.itaadf.cn/Article/details/937975.sHtML
tv.blog.itaadf.cn/Article/details/777319.sHtML
tv.blog.itaadf.cn/Article/details/731759.sHtML
tv.blog.itaadf.cn/Article/details/482644.sHtML
tv.blog.itaadf.cn/Article/details/020082.sHtML
tv.blog.itaadf.cn/Article/details/424826.sHtML
tv.blog.itaadf.cn/Article/details/062626.sHtML
tv.blog.itaadf.cn/Article/details/799133.sHtML
tv.blog.itaadf.cn/Article/details/042262.sHtML
tv.blog.itaadf.cn/Article/details/599535.sHtML
tv.blog.itaadf.cn/Article/details/206264.sHtML
tv.blog.itaadf.cn/Article/details/199717.sHtML
tv.blog.itaadf.cn/Article/details/468020.sHtML
tv.blog.itaadf.cn/Article/details/595131.sHtML
tv.blog.itaadf.cn/Article/details/995139.sHtML
tv.blog.itaadf.cn/Article/details/913571.sHtML
tv.blog.itaadf.cn/Article/details/195959.sHtML
tv.blog.itaadf.cn/Article/details/373955.sHtML
tv.blog.itaadf.cn/Article/details/002468.sHtML
tv.blog.itaadf.cn/Article/details/408864.sHtML
tv.blog.itaadf.cn/Article/details/064462.sHtML
tv.blog.itaadf.cn/Article/details/757999.sHtML
tv.blog.itaadf.cn/Article/details/046466.sHtML
tv.blog.itaadf.cn/Article/details/200200.sHtML
tv.blog.itaadf.cn/Article/details/688060.sHtML
tv.blog.itaadf.cn/Article/details/351375.sHtML
tv.blog.itaadf.cn/Article/details/866888.sHtML
tv.blog.itaadf.cn/Article/details/460028.sHtML
tv.blog.itaadf.cn/Article/details/624408.sHtML
tv.blog.itaadf.cn/Article/details/006286.sHtML
tv.blog.itaadf.cn/Article/details/088684.sHtML
tv.blog.itaadf.cn/Article/details/604484.sHtML
tv.blog.itaadf.cn/Article/details/862664.sHtML
tv.blog.itaadf.cn/Article/details/006820.sHtML
tv.blog.itaadf.cn/Article/details/422424.sHtML
tv.blog.itaadf.cn/Article/details/531597.sHtML
tv.blog.itaadf.cn/Article/details/193755.sHtML
tv.blog.itaadf.cn/Article/details/000842.sHtML
tv.blog.itaadf.cn/Article/details/779957.sHtML
tv.blog.itaadf.cn/Article/details/466848.sHtML
tv.blog.itaadf.cn/Article/details/268620.sHtML
tv.blog.itaadf.cn/Article/details/175359.sHtML
tv.blog.itaadf.cn/Article/details/684448.sHtML
tv.blog.itaadf.cn/Article/details/391517.sHtML
tv.blog.itaadf.cn/Article/details/208282.sHtML
tv.blog.itaadf.cn/Article/details/315537.sHtML
tv.blog.itaadf.cn/Article/details/684864.sHtML
tv.blog.itaadf.cn/Article/details/111559.sHtML
tv.blog.itaadf.cn/Article/details/133791.sHtML
tv.blog.itaadf.cn/Article/details/484404.sHtML
tv.blog.itaadf.cn/Article/details/806208.sHtML
tv.blog.itaadf.cn/Article/details/317935.sHtML
tv.blog.itaadf.cn/Article/details/331937.sHtML
tv.blog.itaadf.cn/Article/details/313153.sHtML
tv.blog.itaadf.cn/Article/details/739337.sHtML
tv.blog.itaadf.cn/Article/details/737513.sHtML
tv.blog.itaadf.cn/Article/details/022462.sHtML
tv.blog.itaadf.cn/Article/details/393139.sHtML
tv.blog.itaadf.cn/Article/details/133933.sHtML
tv.blog.itaadf.cn/Article/details/220808.sHtML
tv.blog.itaadf.cn/Article/details/882460.sHtML
tv.blog.itaadf.cn/Article/details/206664.sHtML
tv.blog.itaadf.cn/Article/details/402806.sHtML
tv.blog.itaadf.cn/Article/details/735199.sHtML
tv.blog.itaadf.cn/Article/details/660884.sHtML
tv.blog.itaadf.cn/Article/details/288044.sHtML
tv.blog.itaadf.cn/Article/details/331395.sHtML
tv.blog.itaadf.cn/Article/details/735937.sHtML
tv.blog.itaadf.cn/Article/details/628666.sHtML
tv.blog.itaadf.cn/Article/details/937971.sHtML
tv.blog.itaadf.cn/Article/details/066282.sHtML
tv.blog.itaadf.cn/Article/details/715377.sHtML
tv.blog.itaadf.cn/Article/details/955197.sHtML
tv.blog.itaadf.cn/Article/details/286460.sHtML
tv.blog.itaadf.cn/Article/details/688884.sHtML
tv.blog.itaadf.cn/Article/details/044402.sHtML
tv.blog.itaadf.cn/Article/details/113993.sHtML
tv.blog.itaadf.cn/Article/details/048260.sHtML
tv.blog.itaadf.cn/Article/details/468044.sHtML
tv.blog.itaadf.cn/Article/details/480648.sHtML
tv.blog.itaadf.cn/Article/details/480062.sHtML
tv.blog.itaadf.cn/Article/details/028222.sHtML
tv.blog.itaadf.cn/Article/details/264222.sHtML
tv.blog.itaadf.cn/Article/details/860668.sHtML
tv.blog.itaadf.cn/Article/details/179591.sHtML
tv.blog.itaadf.cn/Article/details/688440.sHtML
tv.blog.itaadf.cn/Article/details/842602.sHtML
tv.blog.itaadf.cn/Article/details/284228.sHtML
tv.blog.itaadf.cn/Article/details/331117.sHtML
tv.blog.itaadf.cn/Article/details/359935.sHtML
tv.blog.itaadf.cn/Article/details/973331.sHtML
tv.blog.itaadf.cn/Article/details/917791.sHtML
tv.blog.itaadf.cn/Article/details/793197.sHtML
tv.blog.itaadf.cn/Article/details/822266.sHtML
tv.blog.itaadf.cn/Article/details/244622.sHtML
tv.blog.itaadf.cn/Article/details/086484.sHtML
tv.blog.itaadf.cn/Article/details/628840.sHtML
tv.blog.itaadf.cn/Article/details/519191.sHtML
tv.blog.itaadf.cn/Article/details/222624.sHtML
tv.blog.itaadf.cn/Article/details/066624.sHtML
tv.blog.itaadf.cn/Article/details/406044.sHtML
tv.blog.itaadf.cn/Article/details/175155.sHtML
tv.blog.itaadf.cn/Article/details/351335.sHtML
tv.blog.itaadf.cn/Article/details/600222.sHtML
tv.blog.itaadf.cn/Article/details/664460.sHtML
tv.blog.itaadf.cn/Article/details/208422.sHtML
tv.blog.itaadf.cn/Article/details/862028.sHtML
tv.blog.itaadf.cn/Article/details/119915.sHtML
tv.blog.itaadf.cn/Article/details/593399.sHtML
tv.blog.itaadf.cn/Article/details/977335.sHtML
tv.blog.itaadf.cn/Article/details/686800.sHtML
tv.blog.itaadf.cn/Article/details/606660.sHtML
tv.blog.itaadf.cn/Article/details/664046.sHtML
tv.blog.itaadf.cn/Article/details/448000.sHtML
tv.blog.itaadf.cn/Article/details/733339.sHtML
tv.blog.itaadf.cn/Article/details/008420.sHtML
tv.blog.itaadf.cn/Article/details/804040.sHtML
tv.blog.itaadf.cn/Article/details/208268.sHtML
tv.blog.itaadf.cn/Article/details/755755.sHtML
tv.blog.itaadf.cn/Article/details/844282.sHtML
tv.blog.itaadf.cn/Article/details/802280.sHtML
tv.blog.itaadf.cn/Article/details/040420.sHtML
tv.blog.itaadf.cn/Article/details/951591.sHtML
tv.blog.itaadf.cn/Article/details/466848.sHtML
tv.blog.itaadf.cn/Article/details/482600.sHtML
tv.blog.itaadf.cn/Article/details/260082.sHtML
tv.blog.itaadf.cn/Article/details/824246.sHtML
tv.blog.itaadf.cn/Article/details/000642.sHtML
tv.blog.itaadf.cn/Article/details/973777.sHtML
tv.blog.itaadf.cn/Article/details/715379.sHtML
tv.blog.itaadf.cn/Article/details/242806.sHtML
tv.blog.itaadf.cn/Article/details/006686.sHtML
tv.blog.itaadf.cn/Article/details/539133.sHtML
tv.blog.itaadf.cn/Article/details/000204.sHtML
tv.blog.itaadf.cn/Article/details/222428.sHtML
tv.blog.itaadf.cn/Article/details/157757.sHtML
tv.blog.itaadf.cn/Article/details/286664.sHtML
tv.blog.itaadf.cn/Article/details/551575.sHtML
tv.blog.itaadf.cn/Article/details/864424.sHtML
tv.blog.itaadf.cn/Article/details/177179.sHtML
tv.blog.itaadf.cn/Article/details/379199.sHtML
tv.blog.itaadf.cn/Article/details/006642.sHtML
tv.blog.itaadf.cn/Article/details/339577.sHtML
tv.blog.itaadf.cn/Article/details/395591.sHtML
tv.blog.itaadf.cn/Article/details/139997.sHtML
tv.blog.itaadf.cn/Article/details/846224.sHtML
tv.blog.itaadf.cn/Article/details/462220.sHtML
tv.blog.itaadf.cn/Article/details/002808.sHtML
tv.blog.itaadf.cn/Article/details/804008.sHtML
tv.blog.itaadf.cn/Article/details/888646.sHtML
tv.blog.itaadf.cn/Article/details/311955.sHtML
tv.blog.itaadf.cn/Article/details/020860.sHtML
tv.blog.itaadf.cn/Article/details/171579.sHtML
tv.blog.itaadf.cn/Article/details/202402.sHtML
tv.blog.itaadf.cn/Article/details/040006.sHtML
tv.blog.itaadf.cn/Article/details/117595.sHtML
tv.blog.itaadf.cn/Article/details/040826.sHtML
tv.blog.itaadf.cn/Article/details/086020.sHtML
tv.blog.itaadf.cn/Article/details/804646.sHtML
tv.blog.itaadf.cn/Article/details/339155.sHtML
tv.blog.itaadf.cn/Article/details/406026.sHtML
tv.blog.itaadf.cn/Article/details/533117.sHtML
tv.blog.itaadf.cn/Article/details/539573.sHtML
tv.blog.itaadf.cn/Article/details/426208.sHtML
tv.blog.itaadf.cn/Article/details/048404.sHtML
tv.blog.itaadf.cn/Article/details/660800.sHtML
tv.blog.itaadf.cn/Article/details/684268.sHtML
tv.blog.itaadf.cn/Article/details/971573.sHtML
tv.blog.itaadf.cn/Article/details/159533.sHtML
tv.blog.itaadf.cn/Article/details/331715.sHtML
tv.blog.itaadf.cn/Article/details/179393.sHtML
tv.blog.itaadf.cn/Article/details/597959.sHtML
tv.blog.itaadf.cn/Article/details/408228.sHtML
tv.blog.itaadf.cn/Article/details/860044.sHtML
tv.blog.itaadf.cn/Article/details/919555.sHtML
tv.blog.itaadf.cn/Article/details/539393.sHtML
tv.blog.itaadf.cn/Article/details/806022.sHtML
tv.blog.itaadf.cn/Article/details/353515.sHtML
tv.blog.itaadf.cn/Article/details/919597.sHtML
tv.blog.itaadf.cn/Article/details/866628.sHtML
tv.blog.itaadf.cn/Article/details/557953.sHtML
tv.blog.itaadf.cn/Article/details/420682.sHtML
tv.blog.itaadf.cn/Article/details/931557.sHtML
tv.blog.itaadf.cn/Article/details/268206.sHtML
tv.blog.itaadf.cn/Article/details/571399.sHtML
tv.blog.itaadf.cn/Article/details/688064.sHtML
tv.blog.itaadf.cn/Article/details/795591.sHtML
tv.blog.itaadf.cn/Article/details/555317.sHtML
tv.blog.itaadf.cn/Article/details/040864.sHtML
tv.blog.itaadf.cn/Article/details/242888.sHtML
tv.blog.itaadf.cn/Article/details/002826.sHtML
tv.blog.itaadf.cn/Article/details/791957.sHtML
tv.blog.itaadf.cn/Article/details/628264.sHtML
tv.blog.itaadf.cn/Article/details/602460.sHtML
tv.blog.itaadf.cn/Article/details/113199.sHtML
tv.blog.itaadf.cn/Article/details/086064.sHtML
tv.blog.itaadf.cn/Article/details/666468.sHtML
tv.blog.itaadf.cn/Article/details/157375.sHtML
tv.blog.itaadf.cn/Article/details/579351.sHtML
tv.blog.itaadf.cn/Article/details/931595.sHtML
tv.blog.itaadf.cn/Article/details/684442.sHtML
tv.blog.itaadf.cn/Article/details/779555.sHtML
tv.blog.itaadf.cn/Article/details/399359.sHtML
tv.blog.itaadf.cn/Article/details/171551.sHtML
tv.blog.itaadf.cn/Article/details/620826.sHtML
tv.blog.itaadf.cn/Article/details/997173.sHtML
tv.blog.itaadf.cn/Article/details/862448.sHtML
tv.blog.itaadf.cn/Article/details/408804.sHtML
tv.blog.itaadf.cn/Article/details/204226.sHtML
tv.blog.itaadf.cn/Article/details/771591.sHtML
tv.blog.itaadf.cn/Article/details/848664.sHtML
tv.blog.itaadf.cn/Article/details/688660.sHtML
tv.blog.itaadf.cn/Article/details/355555.sHtML
tv.blog.itaadf.cn/Article/details/646008.sHtML
tv.blog.itaadf.cn/Article/details/286648.sHtML
tv.blog.itaadf.cn/Article/details/020608.sHtML
tv.blog.itaadf.cn/Article/details/066260.sHtML
tv.blog.itaadf.cn/Article/details/644682.sHtML
tv.blog.itaadf.cn/Article/details/608422.sHtML
tv.blog.itaadf.cn/Article/details/468248.sHtML
tv.blog.itaadf.cn/Article/details/848684.sHtML
tv.blog.itaadf.cn/Article/details/480822.sHtML
tv.blog.itaadf.cn/Article/details/488008.sHtML
tv.blog.itaadf.cn/Article/details/713735.sHtML
tv.blog.itaadf.cn/Article/details/575131.sHtML
tv.blog.itaadf.cn/Article/details/842086.sHtML
tv.blog.itaadf.cn/Article/details/288400.sHtML
tv.blog.itaadf.cn/Article/details/400822.sHtML
tv.blog.itaadf.cn/Article/details/119331.sHtML
tv.blog.itaadf.cn/Article/details/644886.sHtML
tv.blog.itaadf.cn/Article/details/226842.sHtML
tv.blog.itaadf.cn/Article/details/195591.sHtML
tv.blog.itaadf.cn/Article/details/113175.sHtML
tv.blog.itaadf.cn/Article/details/208604.sHtML
tv.blog.itaadf.cn/Article/details/531395.sHtML
tv.blog.itaadf.cn/Article/details/111139.sHtML
tv.blog.itaadf.cn/Article/details/315973.sHtML
tv.blog.itaadf.cn/Article/details/028826.sHtML
tv.blog.itaadf.cn/Article/details/484000.sHtML
tv.blog.itaadf.cn/Article/details/939113.sHtML
tv.blog.itaadf.cn/Article/details/066048.sHtML
tv.blog.itaadf.cn/Article/details/266206.sHtML
tv.blog.itaadf.cn/Article/details/442400.sHtML
tv.blog.itaadf.cn/Article/details/288020.sHtML
tv.blog.itaadf.cn/Article/details/646248.sHtML
tv.blog.itaadf.cn/Article/details/559133.sHtML
tv.blog.itaadf.cn/Article/details/806868.sHtML
tv.blog.itaadf.cn/Article/details/319717.sHtML
tv.blog.itaadf.cn/Article/details/939957.sHtML
tv.blog.itaadf.cn/Article/details/317979.sHtML
tv.blog.itaadf.cn/Article/details/319353.sHtML
tv.blog.itaadf.cn/Article/details/379197.sHtML
tv.blog.itaadf.cn/Article/details/339311.sHtML
tv.blog.itaadf.cn/Article/details/882062.sHtML
tv.blog.itaadf.cn/Article/details/246028.sHtML
tv.blog.itaadf.cn/Article/details/868084.sHtML
tv.blog.itaadf.cn/Article/details/711135.sHtML
tv.blog.itaadf.cn/Article/details/084664.sHtML
tv.blog.itaadf.cn/Article/details/993937.sHtML
tv.blog.itaadf.cn/Article/details/246240.sHtML
tv.blog.itaadf.cn/Article/details/660624.sHtML
tv.blog.itaadf.cn/Article/details/404668.sHtML
tv.blog.itaadf.cn/Article/details/997731.sHtML
tv.blog.itaadf.cn/Article/details/864288.sHtML
tv.blog.itaadf.cn/Article/details/195155.sHtML
tv.blog.itaadf.cn/Article/details/519137.sHtML
tv.blog.itaadf.cn/Article/details/799753.sHtML
tv.blog.itaadf.cn/Article/details/535373.sHtML
tv.blog.itaadf.cn/Article/details/395133.sHtML
tv.blog.itaadf.cn/Article/details/424222.sHtML
tv.blog.itaadf.cn/Article/details/608206.sHtML
tv.blog.itaadf.cn/Article/details/248222.sHtML
tv.blog.itaadf.cn/Article/details/824424.sHtML
tv.blog.itaadf.cn/Article/details/995513.sHtML
tv.blog.itaadf.cn/Article/details/048282.sHtML
tv.blog.itaadf.cn/Article/details/579999.sHtML
tv.blog.itaadf.cn/Article/details/533799.sHtML
tv.blog.itaadf.cn/Article/details/628460.sHtML
tv.blog.itaadf.cn/Article/details/399713.sHtML
tv.blog.itaadf.cn/Article/details/793159.sHtML
tv.blog.itaadf.cn/Article/details/393335.sHtML
tv.blog.itaadf.cn/Article/details/202204.sHtML
tv.blog.itaadf.cn/Article/details/399373.sHtML
tv.blog.itaadf.cn/Article/details/446668.sHtML
tv.blog.itaadf.cn/Article/details/604408.sHtML
tv.blog.itaadf.cn/Article/details/240868.sHtML
tv.blog.itaadf.cn/Article/details/333337.sHtML
tv.blog.itaadf.cn/Article/details/206448.sHtML
tv.blog.itaadf.cn/Article/details/533991.sHtML
tv.blog.itaadf.cn/Article/details/733917.sHtML
tv.blog.itaadf.cn/Article/details/517799.sHtML
tv.blog.itaadf.cn/Article/details/220626.sHtML
tv.blog.itaadf.cn/Article/details/571551.sHtML
tv.blog.itaadf.cn/Article/details/484622.sHtML
tv.blog.itaadf.cn/Article/details/117753.sHtML
tv.blog.itaadf.cn/Article/details/999519.sHtML
tv.blog.itaadf.cn/Article/details/688848.sHtML
tv.blog.itaadf.cn/Article/details/173155.sHtML
tv.blog.itaadf.cn/Article/details/826288.sHtML
tv.blog.itaadf.cn/Article/details/462482.sHtML
tv.blog.itaadf.cn/Article/details/828066.sHtML
tv.blog.itaadf.cn/Article/details/260824.sHtML
tv.blog.itaadf.cn/Article/details/191997.sHtML
tv.blog.itaadf.cn/Article/details/860840.sHtML
tv.blog.itaadf.cn/Article/details/646402.sHtML
tv.blog.itaadf.cn/Article/details/082680.sHtML
tv.blog.itaadf.cn/Article/details/622006.sHtML
tv.blog.itaadf.cn/Article/details/286806.sHtML
tv.blog.itaadf.cn/Article/details/648484.sHtML
tv.blog.itaadf.cn/Article/details/773799.sHtML
tv.blog.itaadf.cn/Article/details/044826.sHtML
tv.blog.itaadf.cn/Article/details/597719.sHtML
tv.blog.itaadf.cn/Article/details/519595.sHtML
tv.blog.itaadf.cn/Article/details/468404.sHtML
tv.blog.itaadf.cn/Article/details/373359.sHtML
tv.blog.itaadf.cn/Article/details/846804.sHtML
tv.blog.itaadf.cn/Article/details/864644.sHtML
tv.blog.itaadf.cn/Article/details/220820.sHtML
tv.blog.itaadf.cn/Article/details/408800.sHtML
tv.blog.itaadf.cn/Article/details/339357.sHtML
tv.blog.itaadf.cn/Article/details/979131.sHtML
tv.blog.itaadf.cn/Article/details/688682.sHtML
tv.blog.itaadf.cn/Article/details/739751.sHtML
tv.blog.itaadf.cn/Article/details/684620.sHtML
tv.blog.itaadf.cn/Article/details/086444.sHtML
tv.blog.itaadf.cn/Article/details/200880.sHtML
tv.blog.itaadf.cn/Article/details/953377.sHtML
tv.blog.itaadf.cn/Article/details/135991.sHtML
tv.blog.itaadf.cn/Article/details/426862.sHtML
tv.blog.itaadf.cn/Article/details/060480.sHtML
tv.blog.itaadf.cn/Article/details/715973.sHtML
tv.blog.itaadf.cn/Article/details/755373.sHtML
tv.blog.itaadf.cn/Article/details/737131.sHtML
tv.blog.itaadf.cn/Article/details/113759.sHtML
tv.blog.itaadf.cn/Article/details/777375.sHtML
tv.blog.itaadf.cn/Article/details/399793.sHtML
tv.blog.itaadf.cn/Article/details/931515.sHtML
tv.blog.itaadf.cn/Article/details/539351.sHtML
tv.blog.itaadf.cn/Article/details/775573.sHtML
tv.blog.itaadf.cn/Article/details/379939.sHtML
tv.blog.itaadf.cn/Article/details/484000.sHtML
tv.blog.itaadf.cn/Article/details/046442.sHtML
tv.blog.itaadf.cn/Article/details/204266.sHtML
tv.blog.itaadf.cn/Article/details/628462.sHtML
tv.blog.itaadf.cn/Article/details/953993.sHtML
tv.blog.itaadf.cn/Article/details/971999.sHtML
tv.blog.itaadf.cn/Article/details/682604.sHtML
tv.blog.itaadf.cn/Article/details/919331.sHtML
tv.blog.itaadf.cn/Article/details/286286.sHtML
tv.blog.itaadf.cn/Article/details/113337.sHtML
tv.blog.itaadf.cn/Article/details/935573.sHtML
tv.blog.itaadf.cn/Article/details/335993.sHtML
tv.blog.itaadf.cn/Article/details/551537.sHtML
tv.blog.itaadf.cn/Article/details/519919.sHtML
tv.blog.itaadf.cn/Article/details/757551.sHtML
tv.blog.itaadf.cn/Article/details/620426.sHtML
tv.blog.itaadf.cn/Article/details/242240.sHtML
tv.blog.itaadf.cn/Article/details/822246.sHtML
tv.blog.itaadf.cn/Article/details/220084.sHtML
tv.blog.itaadf.cn/Article/details/991733.sHtML
tv.blog.itaadf.cn/Article/details/717515.sHtML
tv.blog.itaadf.cn/Article/details/175311.sHtML
tv.blog.itaadf.cn/Article/details/266004.sHtML
tv.blog.itaadf.cn/Article/details/482268.sHtML
tv.blog.itaadf.cn/Article/details/268288.sHtML
tv.blog.itaadf.cn/Article/details/268240.sHtML
tv.blog.itaadf.cn/Article/details/119719.sHtML
tv.blog.itaadf.cn/Article/details/462422.sHtML
tv.blog.itaadf.cn/Article/details/084242.sHtML
tv.blog.itaadf.cn/Article/details/757133.sHtML
tv.blog.itaadf.cn/Article/details/088024.sHtML
tv.blog.itaadf.cn/Article/details/339395.sHtML
tv.blog.itaadf.cn/Article/details/602622.sHtML
tv.blog.itaadf.cn/Article/details/513915.sHtML
tv.blog.itaadf.cn/Article/details/242202.sHtML
tv.blog.itaadf.cn/Article/details/933913.sHtML
tv.blog.itaadf.cn/Article/details/222040.sHtML
tv.blog.itaadf.cn/Article/details/006084.sHtML
tv.blog.itaadf.cn/Article/details/379971.sHtML
tv.blog.itaadf.cn/Article/details/517139.sHtML
tv.blog.itaadf.cn/Article/details/846224.sHtML
tv.blog.itaadf.cn/Article/details/193139.sHtML
tv.blog.itaadf.cn/Article/details/008248.sHtML
tv.blog.itaadf.cn/Article/details/246684.sHtML
tv.blog.itaadf.cn/Article/details/804468.sHtML
tv.blog.itaadf.cn/Article/details/593177.sHtML
tv.blog.itaadf.cn/Article/details/115391.sHtML
tv.blog.itaadf.cn/Article/details/600826.sHtML
tv.blog.itaadf.cn/Article/details/442460.sHtML
tv.blog.itaadf.cn/Article/details/888866.sHtML
tv.blog.itaadf.cn/Article/details/608220.sHtML
tv.blog.itaadf.cn/Article/details/844020.sHtML
