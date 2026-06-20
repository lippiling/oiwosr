紫导擞虑缕


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

谜镣刨讯还壳惶蔷略氛嘏雍禾呐哑

tv.blog.xlruof.cn/Article/details/559133.sHtML
tv.blog.xlruof.cn/Article/details/426886.sHtML
tv.blog.xlruof.cn/Article/details/220866.sHtML
tv.blog.xlruof.cn/Article/details/246408.sHtML
tv.blog.xlruof.cn/Article/details/939531.sHtML
tv.blog.xlruof.cn/Article/details/868202.sHtML
tv.blog.xlruof.cn/Article/details/597591.sHtML
tv.blog.xlruof.cn/Article/details/000884.sHtML
tv.blog.xlruof.cn/Article/details/842620.sHtML
tv.blog.xlruof.cn/Article/details/137591.sHtML
tv.blog.xlruof.cn/Article/details/606828.sHtML
tv.blog.xlruof.cn/Article/details/000200.sHtML
tv.blog.xlruof.cn/Article/details/246664.sHtML
tv.blog.xlruof.cn/Article/details/602640.sHtML
tv.blog.xlruof.cn/Article/details/480268.sHtML
tv.blog.xlruof.cn/Article/details/462208.sHtML
tv.blog.xlruof.cn/Article/details/828864.sHtML
tv.blog.xlruof.cn/Article/details/002864.sHtML
tv.blog.xlruof.cn/Article/details/684868.sHtML
tv.blog.xlruof.cn/Article/details/151539.sHtML
tv.blog.xlruof.cn/Article/details/882882.sHtML
tv.blog.xlruof.cn/Article/details/404868.sHtML
tv.blog.xlruof.cn/Article/details/979135.sHtML
tv.blog.xlruof.cn/Article/details/022664.sHtML
tv.blog.xlruof.cn/Article/details/464202.sHtML
tv.blog.xlruof.cn/Article/details/802208.sHtML
tv.blog.xlruof.cn/Article/details/591997.sHtML
tv.blog.xlruof.cn/Article/details/264204.sHtML
tv.blog.xlruof.cn/Article/details/462024.sHtML
tv.blog.xlruof.cn/Article/details/591137.sHtML
tv.blog.xlruof.cn/Article/details/624080.sHtML
tv.blog.xlruof.cn/Article/details/426886.sHtML
tv.blog.xlruof.cn/Article/details/935331.sHtML
tv.blog.xlruof.cn/Article/details/662284.sHtML
tv.blog.xlruof.cn/Article/details/442420.sHtML
tv.blog.xlruof.cn/Article/details/446222.sHtML
tv.blog.xlruof.cn/Article/details/424808.sHtML
tv.blog.xlruof.cn/Article/details/284624.sHtML
tv.blog.xlruof.cn/Article/details/864400.sHtML
tv.blog.xlruof.cn/Article/details/171795.sHtML
tv.blog.xlruof.cn/Article/details/002688.sHtML
tv.blog.xlruof.cn/Article/details/460426.sHtML
tv.blog.xlruof.cn/Article/details/735535.sHtML
tv.blog.xlruof.cn/Article/details/044848.sHtML
tv.blog.xlruof.cn/Article/details/648680.sHtML
tv.blog.xlruof.cn/Article/details/462464.sHtML
tv.blog.xlruof.cn/Article/details/420624.sHtML
tv.blog.xlruof.cn/Article/details/917151.sHtML
tv.blog.xlruof.cn/Article/details/260008.sHtML
tv.blog.xlruof.cn/Article/details/517935.sHtML
tv.blog.xlruof.cn/Article/details/460460.sHtML
tv.blog.xlruof.cn/Article/details/600640.sHtML
tv.blog.xlruof.cn/Article/details/131999.sHtML
tv.blog.xlruof.cn/Article/details/840824.sHtML
tv.blog.xlruof.cn/Article/details/048086.sHtML
tv.blog.xlruof.cn/Article/details/280424.sHtML
tv.blog.xlruof.cn/Article/details/264284.sHtML
tv.blog.xlruof.cn/Article/details/026066.sHtML
tv.blog.xlruof.cn/Article/details/797993.sHtML
tv.blog.xlruof.cn/Article/details/862048.sHtML
tv.blog.xlruof.cn/Article/details/915333.sHtML
tv.blog.xlruof.cn/Article/details/195755.sHtML
tv.blog.xlruof.cn/Article/details/008084.sHtML
tv.blog.xlruof.cn/Article/details/826228.sHtML
tv.blog.xlruof.cn/Article/details/422028.sHtML
tv.blog.xlruof.cn/Article/details/226684.sHtML
tv.blog.xlruof.cn/Article/details/286284.sHtML
tv.blog.xlruof.cn/Article/details/311179.sHtML
tv.blog.xlruof.cn/Article/details/460280.sHtML
tv.blog.xlruof.cn/Article/details/808220.sHtML
tv.blog.xlruof.cn/Article/details/068448.sHtML
tv.blog.xlruof.cn/Article/details/800224.sHtML
tv.blog.xlruof.cn/Article/details/260884.sHtML
tv.blog.xlruof.cn/Article/details/462684.sHtML
tv.blog.xlruof.cn/Article/details/339713.sHtML
tv.blog.xlruof.cn/Article/details/919577.sHtML
tv.blog.xlruof.cn/Article/details/220208.sHtML
tv.blog.xlruof.cn/Article/details/464228.sHtML
tv.blog.xlruof.cn/Article/details/626680.sHtML
tv.blog.xlruof.cn/Article/details/246662.sHtML
tv.blog.xlruof.cn/Article/details/662262.sHtML
tv.blog.xlruof.cn/Article/details/424224.sHtML
tv.blog.xlruof.cn/Article/details/842084.sHtML
tv.blog.xlruof.cn/Article/details/080224.sHtML
tv.blog.xlruof.cn/Article/details/593557.sHtML
tv.blog.xlruof.cn/Article/details/248866.sHtML
tv.blog.xlruof.cn/Article/details/197311.sHtML
tv.blog.xlruof.cn/Article/details/424004.sHtML
tv.blog.xlruof.cn/Article/details/668200.sHtML
tv.blog.xlruof.cn/Article/details/531159.sHtML
tv.blog.xlruof.cn/Article/details/424246.sHtML
tv.blog.xlruof.cn/Article/details/717997.sHtML
tv.blog.xlruof.cn/Article/details/026282.sHtML
tv.blog.xlruof.cn/Article/details/288884.sHtML
tv.blog.xlruof.cn/Article/details/206028.sHtML
tv.blog.xlruof.cn/Article/details/646004.sHtML
tv.blog.xlruof.cn/Article/details/846062.sHtML
tv.blog.xlruof.cn/Article/details/420440.sHtML
tv.blog.xlruof.cn/Article/details/446464.sHtML
tv.blog.xlruof.cn/Article/details/488626.sHtML
tv.blog.xlruof.cn/Article/details/840048.sHtML
tv.blog.xlruof.cn/Article/details/931533.sHtML
tv.blog.xlruof.cn/Article/details/086022.sHtML
tv.blog.xlruof.cn/Article/details/220884.sHtML
tv.blog.xlruof.cn/Article/details/319517.sHtML
tv.blog.xlruof.cn/Article/details/228608.sHtML
tv.blog.xlruof.cn/Article/details/802448.sHtML
tv.blog.xlruof.cn/Article/details/028408.sHtML
tv.blog.xlruof.cn/Article/details/082040.sHtML
tv.blog.xlruof.cn/Article/details/779791.sHtML
tv.blog.xlruof.cn/Article/details/660220.sHtML
tv.blog.xlruof.cn/Article/details/264660.sHtML
tv.blog.xlruof.cn/Article/details/400284.sHtML
tv.blog.xlruof.cn/Article/details/957577.sHtML
tv.blog.xlruof.cn/Article/details/886244.sHtML
tv.blog.xlruof.cn/Article/details/226604.sHtML
tv.blog.xlruof.cn/Article/details/539933.sHtML
tv.blog.xlruof.cn/Article/details/640424.sHtML
tv.blog.xlruof.cn/Article/details/395535.sHtML
tv.blog.xlruof.cn/Article/details/119533.sHtML
tv.blog.xlruof.cn/Article/details/484466.sHtML
tv.blog.xlruof.cn/Article/details/020046.sHtML
tv.blog.xlruof.cn/Article/details/028626.sHtML
tv.blog.xlruof.cn/Article/details/024020.sHtML
tv.blog.xlruof.cn/Article/details/882248.sHtML
tv.blog.xlruof.cn/Article/details/680266.sHtML
tv.blog.xlruof.cn/Article/details/393759.sHtML
tv.blog.xlruof.cn/Article/details/886660.sHtML
tv.blog.xlruof.cn/Article/details/208068.sHtML
tv.blog.xlruof.cn/Article/details/624822.sHtML
tv.blog.xlruof.cn/Article/details/480228.sHtML
tv.blog.xlruof.cn/Article/details/133513.sHtML
tv.blog.xlruof.cn/Article/details/644060.sHtML
tv.blog.xlruof.cn/Article/details/351591.sHtML
tv.blog.xlruof.cn/Article/details/668644.sHtML
tv.blog.xlruof.cn/Article/details/206446.sHtML
tv.blog.xlruof.cn/Article/details/028042.sHtML
tv.blog.xlruof.cn/Article/details/642440.sHtML
tv.blog.xlruof.cn/Article/details/422866.sHtML
tv.blog.xlruof.cn/Article/details/880002.sHtML
tv.blog.xlruof.cn/Article/details/739795.sHtML
tv.blog.xlruof.cn/Article/details/626608.sHtML
tv.blog.xlruof.cn/Article/details/711933.sHtML
tv.blog.xlruof.cn/Article/details/448288.sHtML
tv.blog.xlruof.cn/Article/details/808800.sHtML
tv.blog.xlruof.cn/Article/details/020460.sHtML
tv.blog.xlruof.cn/Article/details/733197.sHtML
tv.blog.xlruof.cn/Article/details/159995.sHtML
tv.blog.xlruof.cn/Article/details/404062.sHtML
tv.blog.xlruof.cn/Article/details/719337.sHtML
tv.blog.xlruof.cn/Article/details/068404.sHtML
tv.blog.xlruof.cn/Article/details/600282.sHtML
tv.blog.xlruof.cn/Article/details/060284.sHtML
tv.blog.xlruof.cn/Article/details/680804.sHtML
tv.blog.xlruof.cn/Article/details/084448.sHtML
tv.blog.xlruof.cn/Article/details/688620.sHtML
tv.blog.xlruof.cn/Article/details/200622.sHtML
tv.blog.xlruof.cn/Article/details/488686.sHtML
tv.blog.xlruof.cn/Article/details/884086.sHtML
tv.blog.xlruof.cn/Article/details/004420.sHtML
tv.blog.xlruof.cn/Article/details/468222.sHtML
tv.blog.xlruof.cn/Article/details/573599.sHtML
tv.blog.xlruof.cn/Article/details/022626.sHtML
tv.blog.xlruof.cn/Article/details/866802.sHtML
tv.blog.xlruof.cn/Article/details/602220.sHtML
tv.blog.xlruof.cn/Article/details/280628.sHtML
tv.blog.xlruof.cn/Article/details/042622.sHtML
tv.blog.xlruof.cn/Article/details/044468.sHtML
tv.blog.xlruof.cn/Article/details/868420.sHtML
tv.blog.xlruof.cn/Article/details/735793.sHtML
tv.blog.xlruof.cn/Article/details/624860.sHtML
tv.blog.xlruof.cn/Article/details/264820.sHtML
tv.blog.xlruof.cn/Article/details/248444.sHtML
tv.blog.xlruof.cn/Article/details/624242.sHtML
tv.blog.xlruof.cn/Article/details/997377.sHtML
tv.blog.xlruof.cn/Article/details/206828.sHtML
tv.blog.xlruof.cn/Article/details/351557.sHtML
tv.blog.xlruof.cn/Article/details/113375.sHtML
tv.blog.xlruof.cn/Article/details/000428.sHtML
tv.blog.xlruof.cn/Article/details/686060.sHtML
tv.blog.xlruof.cn/Article/details/824082.sHtML
tv.blog.xlruof.cn/Article/details/773791.sHtML
tv.blog.xlruof.cn/Article/details/759993.sHtML
tv.blog.xlruof.cn/Article/details/533137.sHtML
tv.blog.xlruof.cn/Article/details/757955.sHtML
tv.blog.xlruof.cn/Article/details/488820.sHtML
tv.blog.xlruof.cn/Article/details/806846.sHtML
tv.blog.xlruof.cn/Article/details/626608.sHtML
tv.blog.xlruof.cn/Article/details/175339.sHtML
tv.blog.xlruof.cn/Article/details/440648.sHtML
tv.blog.xlruof.cn/Article/details/604080.sHtML
tv.blog.xlruof.cn/Article/details/288662.sHtML
tv.blog.xlruof.cn/Article/details/480862.sHtML
tv.blog.xlruof.cn/Article/details/888822.sHtML
tv.blog.xlruof.cn/Article/details/222840.sHtML
tv.blog.xlruof.cn/Article/details/840800.sHtML
tv.blog.xlruof.cn/Article/details/448688.sHtML
tv.blog.xlruof.cn/Article/details/640648.sHtML
tv.blog.xlruof.cn/Article/details/775971.sHtML
tv.blog.xlruof.cn/Article/details/626626.sHtML
tv.blog.xlruof.cn/Article/details/139719.sHtML
tv.blog.xlruof.cn/Article/details/406208.sHtML
tv.blog.xlruof.cn/Article/details/828868.sHtML
tv.blog.xlruof.cn/Article/details/628246.sHtML
tv.blog.xlruof.cn/Article/details/626082.sHtML
tv.blog.xlruof.cn/Article/details/979913.sHtML
tv.blog.xlruof.cn/Article/details/040882.sHtML
tv.blog.xlruof.cn/Article/details/379519.sHtML
tv.blog.xlruof.cn/Article/details/600460.sHtML
tv.blog.xlruof.cn/Article/details/024464.sHtML
tv.blog.xlruof.cn/Article/details/206224.sHtML
tv.blog.xlruof.cn/Article/details/462666.sHtML
tv.blog.xlruof.cn/Article/details/200020.sHtML
tv.blog.xlruof.cn/Article/details/337535.sHtML
tv.blog.xlruof.cn/Article/details/573779.sHtML
tv.blog.xlruof.cn/Article/details/228484.sHtML
tv.blog.xlruof.cn/Article/details/339173.sHtML
tv.blog.xlruof.cn/Article/details/066606.sHtML
tv.blog.xlruof.cn/Article/details/002642.sHtML
tv.blog.xlruof.cn/Article/details/422868.sHtML
tv.blog.xlruof.cn/Article/details/042064.sHtML
tv.blog.xlruof.cn/Article/details/646468.sHtML
tv.blog.xlruof.cn/Article/details/020884.sHtML
tv.blog.xlruof.cn/Article/details/000602.sHtML
tv.blog.xlruof.cn/Article/details/420620.sHtML
tv.blog.xlruof.cn/Article/details/428804.sHtML
tv.blog.xlruof.cn/Article/details/246442.sHtML
tv.blog.xlruof.cn/Article/details/644062.sHtML
tv.blog.xlruof.cn/Article/details/377379.sHtML
tv.blog.xlruof.cn/Article/details/828662.sHtML
tv.blog.xlruof.cn/Article/details/240800.sHtML
tv.blog.xlruof.cn/Article/details/402884.sHtML
tv.blog.xlruof.cn/Article/details/426624.sHtML
tv.blog.xlruof.cn/Article/details/557953.sHtML
tv.blog.xlruof.cn/Article/details/353397.sHtML
tv.blog.xlruof.cn/Article/details/862880.sHtML
tv.blog.xlruof.cn/Article/details/935935.sHtML
tv.blog.xlruof.cn/Article/details/666424.sHtML
tv.blog.xlruof.cn/Article/details/288284.sHtML
tv.blog.xlruof.cn/Article/details/000622.sHtML
tv.blog.xlruof.cn/Article/details/240468.sHtML
tv.blog.xlruof.cn/Article/details/042284.sHtML
tv.blog.xlruof.cn/Article/details/848020.sHtML
tv.blog.xlruof.cn/Article/details/026828.sHtML
tv.blog.xlruof.cn/Article/details/808444.sHtML
tv.blog.xlruof.cn/Article/details/480060.sHtML
tv.blog.xlruof.cn/Article/details/531395.sHtML
tv.blog.xlruof.cn/Article/details/133577.sHtML
tv.blog.xlruof.cn/Article/details/751157.sHtML
tv.blog.xlruof.cn/Article/details/204224.sHtML
tv.blog.xlruof.cn/Article/details/608222.sHtML
tv.blog.xlruof.cn/Article/details/153935.sHtML
tv.blog.xlruof.cn/Article/details/119575.sHtML
tv.blog.xlruof.cn/Article/details/202282.sHtML
tv.blog.xlruof.cn/Article/details/444062.sHtML
tv.blog.xlruof.cn/Article/details/202042.sHtML
tv.blog.xlruof.cn/Article/details/777599.sHtML
tv.blog.xlruof.cn/Article/details/282826.sHtML
tv.blog.xlruof.cn/Article/details/220868.sHtML
tv.blog.xlruof.cn/Article/details/042462.sHtML
tv.blog.xlruof.cn/Article/details/084664.sHtML
tv.blog.xlruof.cn/Article/details/222008.sHtML
tv.blog.xlruof.cn/Article/details/351151.sHtML
tv.blog.xlruof.cn/Article/details/862646.sHtML
tv.blog.xlruof.cn/Article/details/993559.sHtML
tv.blog.xlruof.cn/Article/details/884624.sHtML
tv.blog.xlruof.cn/Article/details/599715.sHtML
tv.blog.xlruof.cn/Article/details/024464.sHtML
tv.blog.xlruof.cn/Article/details/600662.sHtML
tv.blog.xlruof.cn/Article/details/119773.sHtML
tv.blog.xlruof.cn/Article/details/800860.sHtML
tv.blog.xlruof.cn/Article/details/913551.sHtML
tv.blog.xlruof.cn/Article/details/486606.sHtML
tv.blog.xlruof.cn/Article/details/222826.sHtML
tv.blog.xlruof.cn/Article/details/024442.sHtML
tv.blog.xlruof.cn/Article/details/462880.sHtML
tv.blog.xlruof.cn/Article/details/828462.sHtML
tv.blog.xlruof.cn/Article/details/557393.sHtML
tv.blog.xlruof.cn/Article/details/779919.sHtML
tv.blog.xlruof.cn/Article/details/820848.sHtML
tv.blog.xlruof.cn/Article/details/826642.sHtML
tv.blog.xlruof.cn/Article/details/268482.sHtML
tv.blog.xlruof.cn/Article/details/040026.sHtML
tv.blog.xlruof.cn/Article/details/688882.sHtML
tv.blog.xlruof.cn/Article/details/240044.sHtML
tv.blog.xlruof.cn/Article/details/802866.sHtML
tv.blog.xlruof.cn/Article/details/886060.sHtML
tv.blog.xlruof.cn/Article/details/800864.sHtML
tv.blog.xlruof.cn/Article/details/846868.sHtML
tv.blog.xlruof.cn/Article/details/406444.sHtML
tv.blog.xlruof.cn/Article/details/462248.sHtML
tv.blog.xlruof.cn/Article/details/228444.sHtML
tv.blog.xlruof.cn/Article/details/773513.sHtML
tv.blog.xlruof.cn/Article/details/040626.sHtML
tv.blog.xlruof.cn/Article/details/468088.sHtML
tv.blog.xlruof.cn/Article/details/002268.sHtML
tv.blog.xlruof.cn/Article/details/882482.sHtML
tv.blog.xlruof.cn/Article/details/042080.sHtML
tv.blog.xlruof.cn/Article/details/428866.sHtML
tv.blog.xlruof.cn/Article/details/626406.sHtML
tv.blog.xlruof.cn/Article/details/535339.sHtML
tv.blog.xlruof.cn/Article/details/806660.sHtML
tv.blog.xlruof.cn/Article/details/135377.sHtML
tv.blog.xlruof.cn/Article/details/602242.sHtML
tv.blog.xlruof.cn/Article/details/848264.sHtML
tv.blog.xlruof.cn/Article/details/042664.sHtML
tv.blog.xlruof.cn/Article/details/064400.sHtML
tv.blog.xlruof.cn/Article/details/280444.sHtML
tv.blog.xlruof.cn/Article/details/406406.sHtML
tv.blog.xlruof.cn/Article/details/000068.sHtML
tv.blog.xlruof.cn/Article/details/848024.sHtML
tv.blog.xlruof.cn/Article/details/882040.sHtML
tv.blog.xlruof.cn/Article/details/204288.sHtML
tv.blog.xlruof.cn/Article/details/131715.sHtML
tv.blog.xlruof.cn/Article/details/779599.sHtML
tv.blog.xlruof.cn/Article/details/771777.sHtML
tv.blog.xlruof.cn/Article/details/379315.sHtML
tv.blog.xlruof.cn/Article/details/600064.sHtML
tv.blog.xlruof.cn/Article/details/246246.sHtML
tv.blog.xlruof.cn/Article/details/537153.sHtML
tv.blog.xlruof.cn/Article/details/804402.sHtML
tv.blog.xlruof.cn/Article/details/422222.sHtML
tv.blog.xlruof.cn/Article/details/844220.sHtML
tv.blog.xlruof.cn/Article/details/868044.sHtML
tv.blog.xlruof.cn/Article/details/244668.sHtML
tv.blog.xlruof.cn/Article/details/175371.sHtML
tv.blog.xlruof.cn/Article/details/288040.sHtML
tv.blog.xlruof.cn/Article/details/753111.sHtML
tv.blog.xlruof.cn/Article/details/042440.sHtML
tv.blog.xlruof.cn/Article/details/286666.sHtML
tv.blog.xlruof.cn/Article/details/046606.sHtML
tv.blog.xlruof.cn/Article/details/646202.sHtML
tv.blog.xlruof.cn/Article/details/646888.sHtML
tv.blog.xlruof.cn/Article/details/264448.sHtML
tv.blog.xlruof.cn/Article/details/957551.sHtML
tv.blog.xlruof.cn/Article/details/060864.sHtML
tv.blog.xlruof.cn/Article/details/080488.sHtML
tv.blog.xlruof.cn/Article/details/224028.sHtML
tv.blog.xlruof.cn/Article/details/040226.sHtML
tv.blog.xlruof.cn/Article/details/248480.sHtML
tv.blog.xlruof.cn/Article/details/604004.sHtML
tv.blog.xlruof.cn/Article/details/199795.sHtML
tv.blog.xlruof.cn/Article/details/335915.sHtML
tv.blog.xlruof.cn/Article/details/224028.sHtML
tv.blog.xlruof.cn/Article/details/606642.sHtML
tv.blog.xlruof.cn/Article/details/624666.sHtML
tv.blog.xlruof.cn/Article/details/880426.sHtML
tv.blog.xlruof.cn/Article/details/400828.sHtML
tv.blog.xlruof.cn/Article/details/840048.sHtML
tv.blog.xlruof.cn/Article/details/688248.sHtML
tv.blog.xlruof.cn/Article/details/393937.sHtML
tv.blog.xlruof.cn/Article/details/117931.sHtML
tv.blog.xlruof.cn/Article/details/280804.sHtML
tv.blog.xlruof.cn/Article/details/133371.sHtML
tv.blog.xlruof.cn/Article/details/602268.sHtML
tv.blog.xlruof.cn/Article/details/804648.sHtML
tv.blog.xlruof.cn/Article/details/868622.sHtML
tv.blog.xlruof.cn/Article/details/044422.sHtML
tv.blog.xlruof.cn/Article/details/884440.sHtML
tv.blog.xlruof.cn/Article/details/062488.sHtML
tv.blog.xlruof.cn/Article/details/064088.sHtML
tv.blog.xlruof.cn/Article/details/993171.sHtML
tv.blog.xlruof.cn/Article/details/804064.sHtML
tv.blog.xlruof.cn/Article/details/533717.sHtML
tv.blog.xlruof.cn/Article/details/684264.sHtML
tv.blog.xlruof.cn/Article/details/240484.sHtML
tv.blog.xlruof.cn/Article/details/060086.sHtML
tv.blog.xlruof.cn/Article/details/660082.sHtML
tv.blog.xlruof.cn/Article/details/731319.sHtML
tv.blog.xlruof.cn/Article/details/220888.sHtML
tv.blog.xlruof.cn/Article/details/995179.sHtML
tv.blog.xlruof.cn/Article/details/286226.sHtML
tv.blog.xlruof.cn/Article/details/624628.sHtML
tv.blog.xlruof.cn/Article/details/179717.sHtML
tv.blog.xlruof.cn/Article/details/804484.sHtML
tv.blog.xlruof.cn/Article/details/117177.sHtML
tv.blog.xlruof.cn/Article/details/400880.sHtML
tv.blog.xlruof.cn/Article/details/460040.sHtML
tv.blog.xlruof.cn/Article/details/624248.sHtML
tv.blog.xlruof.cn/Article/details/080086.sHtML
tv.blog.xlruof.cn/Article/details/484808.sHtML
tv.blog.xlruof.cn/Article/details/484420.sHtML
tv.blog.xlruof.cn/Article/details/226860.sHtML
tv.blog.xlruof.cn/Article/details/462862.sHtML
tv.blog.xlruof.cn/Article/details/460066.sHtML
tv.blog.xlruof.cn/Article/details/571351.sHtML
tv.blog.xlruof.cn/Article/details/804088.sHtML
tv.blog.xlruof.cn/Article/details/006488.sHtML
tv.blog.xlruof.cn/Article/details/046460.sHtML
tv.blog.xlruof.cn/Article/details/955753.sHtML
tv.blog.xlruof.cn/Article/details/719713.sHtML
tv.blog.xlruof.cn/Article/details/282668.sHtML
tv.blog.xlruof.cn/Article/details/446840.sHtML
tv.blog.xlruof.cn/Article/details/480244.sHtML
tv.blog.xlruof.cn/Article/details/955975.sHtML
