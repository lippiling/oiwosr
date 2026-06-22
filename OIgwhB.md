灸站陕肆蒙


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

饭钦独痔叫晃柯估灾琳睬枷赝才犹

acq.nfsid.cn/226240.Doc
acq.nfsid.cn/246866.Doc
acq.nfsid.cn/224206.Doc
acq.nfsid.cn/608886.Doc
acq.nfsid.cn/424020.Doc
acq.nfsid.cn/808008.Doc
acq.nfsid.cn/068606.Doc
axm.nfsid.cn/806020.Doc
axm.nfsid.cn/004002.Doc
axm.nfsid.cn/862608.Doc
axm.nfsid.cn/462086.Doc
axm.nfsid.cn/804242.Doc
axm.nfsid.cn/084266.Doc
axm.nfsid.cn/420620.Doc
axm.nfsid.cn/240026.Doc
axm.nfsid.cn/220668.Doc
axm.nfsid.cn/424408.Doc
axn.nfsid.cn/600466.Doc
axn.nfsid.cn/868224.Doc
axn.nfsid.cn/422046.Doc
axn.nfsid.cn/409045.Doc
axn.nfsid.cn/206244.Doc
axn.nfsid.cn/846686.Doc
axn.nfsid.cn/215989.Doc
axn.nfsid.cn/666048.Doc
axn.nfsid.cn/206608.Doc
axn.nfsid.cn/022660.Doc
axb.nfsid.cn/400044.Doc
axb.nfsid.cn/115519.Doc
axb.nfsid.cn/046242.Doc
axb.nfsid.cn/626224.Doc
axb.nfsid.cn/080000.Doc
axb.nfsid.cn/828282.Doc
axb.nfsid.cn/040026.Doc
axb.nfsid.cn/826620.Doc
axb.nfsid.cn/200464.Doc
axb.nfsid.cn/806460.Doc
axv.nfsid.cn/482622.Doc
axv.nfsid.cn/595191.Doc
axv.nfsid.cn/424882.Doc
axv.nfsid.cn/848662.Doc
axv.nfsid.cn/266840.Doc
axv.nfsid.cn/684866.Doc
axv.nfsid.cn/904176.Doc
axv.nfsid.cn/042088.Doc
axv.nfsid.cn/866602.Doc
axv.nfsid.cn/406468.Doc
axc.nfsid.cn/064628.Doc
axc.nfsid.cn/044064.Doc
axc.nfsid.cn/606680.Doc
axc.nfsid.cn/026284.Doc
axc.nfsid.cn/240044.Doc
axc.nfsid.cn/244026.Doc
axc.nfsid.cn/442628.Doc
axc.nfsid.cn/284444.Doc
axc.nfsid.cn/268080.Doc
axc.nfsid.cn/028084.Doc
axx.nfsid.cn/888060.Doc
axx.nfsid.cn/680844.Doc
axx.nfsid.cn/024864.Doc
axx.nfsid.cn/668202.Doc
axx.nfsid.cn/822426.Doc
axx.nfsid.cn/068460.Doc
axx.nfsid.cn/262460.Doc
axx.nfsid.cn/888442.Doc
axx.nfsid.cn/668802.Doc
axx.nfsid.cn/993759.Doc
axz.nfsid.cn/800846.Doc
axz.nfsid.cn/620844.Doc
axz.nfsid.cn/084604.Doc
axz.nfsid.cn/246420.Doc
axz.nfsid.cn/486022.Doc
axz.nfsid.cn/686048.Doc
axz.nfsid.cn/400884.Doc
axz.nfsid.cn/880662.Doc
axz.nfsid.cn/808040.Doc
axz.nfsid.cn/000066.Doc
axl.nfsid.cn/406222.Doc
axl.nfsid.cn/262484.Doc
axl.nfsid.cn/719195.Doc
axl.nfsid.cn/226484.Doc
axl.nfsid.cn/020848.Doc
axl.nfsid.cn/862820.Doc
axl.nfsid.cn/224420.Doc
axl.nfsid.cn/222422.Doc
axl.nfsid.cn/682880.Doc
axl.nfsid.cn/686062.Doc
axk.nfsid.cn/622020.Doc
axk.nfsid.cn/622488.Doc
axk.nfsid.cn/006026.Doc
axk.nfsid.cn/575117.Doc
axk.nfsid.cn/048046.Doc
axk.nfsid.cn/204082.Doc
axk.nfsid.cn/804884.Doc
axk.nfsid.cn/628868.Doc
axk.nfsid.cn/973575.Doc
axk.nfsid.cn/088842.Doc
axj.nfsid.cn/602064.Doc
axj.nfsid.cn/444448.Doc
axj.nfsid.cn/664424.Doc
axj.nfsid.cn/800440.Doc
axj.nfsid.cn/604208.Doc
axj.nfsid.cn/880682.Doc
axj.nfsid.cn/131915.Doc
axj.nfsid.cn/066806.Doc
axj.nfsid.cn/468266.Doc
axj.nfsid.cn/486462.Doc
axh.nfsid.cn/022088.Doc
axh.nfsid.cn/486440.Doc
axh.nfsid.cn/884280.Doc
axh.nfsid.cn/224066.Doc
axh.nfsid.cn/404202.Doc
axh.nfsid.cn/208262.Doc
axh.nfsid.cn/711735.Doc
axh.nfsid.cn/860066.Doc
axh.nfsid.cn/064420.Doc
axh.nfsid.cn/028004.Doc
axg.nfsid.cn/400604.Doc
axg.nfsid.cn/262668.Doc
axg.nfsid.cn/042460.Doc
axg.nfsid.cn/680860.Doc
axg.nfsid.cn/824082.Doc
axg.nfsid.cn/424220.Doc
axg.nfsid.cn/488420.Doc
axg.nfsid.cn/084464.Doc
axg.nfsid.cn/028286.Doc
axg.nfsid.cn/666466.Doc
axf.nfsid.cn/608602.Doc
axf.nfsid.cn/822460.Doc
axf.nfsid.cn/264068.Doc
axf.nfsid.cn/226804.Doc
axf.nfsid.cn/848240.Doc
axf.nfsid.cn/555599.Doc
axf.nfsid.cn/711313.Doc
axf.nfsid.cn/420064.Doc
axf.nfsid.cn/806426.Doc
axf.nfsid.cn/014761.Doc
axd.nfsid.cn/840204.Doc
axd.nfsid.cn/406624.Doc
axd.nfsid.cn/666088.Doc
axd.nfsid.cn/753135.Doc
axd.nfsid.cn/335917.Doc
axd.nfsid.cn/877598.Doc
axd.nfsid.cn/277780.Doc
axd.nfsid.cn/864688.Doc
axd.nfsid.cn/288484.Doc
axd.nfsid.cn/462404.Doc
axs.nfsid.cn/822686.Doc
axs.nfsid.cn/204206.Doc
axs.nfsid.cn/806266.Doc
axs.nfsid.cn/154348.Doc
axs.nfsid.cn/088295.Doc
axs.nfsid.cn/791957.Doc
axs.nfsid.cn/554402.Doc
axs.nfsid.cn/488024.Doc
axs.nfsid.cn/886222.Doc
axs.nfsid.cn/222426.Doc
axa.nfsid.cn/400262.Doc
axa.nfsid.cn/426448.Doc
axa.nfsid.cn/046020.Doc
axa.nfsid.cn/064828.Doc
axa.nfsid.cn/040620.Doc
axa.nfsid.cn/846084.Doc
axa.nfsid.cn/066640.Doc
axa.nfsid.cn/622440.Doc
axa.nfsid.cn/248008.Doc
axa.nfsid.cn/462260.Doc
axp.nfsid.cn/826226.Doc
axp.nfsid.cn/226068.Doc
axp.nfsid.cn/644664.Doc
axp.nfsid.cn/868000.Doc
axp.nfsid.cn/088620.Doc
axp.nfsid.cn/606624.Doc
axp.nfsid.cn/379931.Doc
axp.nfsid.cn/082464.Doc
axp.nfsid.cn/864224.Doc
axp.nfsid.cn/060422.Doc
axo.nfsid.cn/644468.Doc
axo.nfsid.cn/806442.Doc
axo.nfsid.cn/200262.Doc
axo.nfsid.cn/442024.Doc
axo.nfsid.cn/468048.Doc
axo.nfsid.cn/866262.Doc
axo.nfsid.cn/406686.Doc
axo.nfsid.cn/022020.Doc
axo.nfsid.cn/228042.Doc
axo.nfsid.cn/042266.Doc
axi.nfsid.cn/537557.Doc
axi.nfsid.cn/682288.Doc
axi.nfsid.cn/464042.Doc
axi.nfsid.cn/826406.Doc
axi.nfsid.cn/444882.Doc
axi.nfsid.cn/311937.Doc
axi.nfsid.cn/024288.Doc
axi.nfsid.cn/806008.Doc
axi.nfsid.cn/404646.Doc
axi.nfsid.cn/646446.Doc
axu.nfsid.cn/282026.Doc
axu.nfsid.cn/597599.Doc
axu.nfsid.cn/688004.Doc
axu.nfsid.cn/028664.Doc
axu.nfsid.cn/428824.Doc
axu.nfsid.cn/200288.Doc
axu.nfsid.cn/628464.Doc
axu.nfsid.cn/462084.Doc
axu.nfsid.cn/460428.Doc
axu.nfsid.cn/606226.Doc
axy.nfsid.cn/820244.Doc
axy.nfsid.cn/068844.Doc
axy.nfsid.cn/006688.Doc
axy.nfsid.cn/466226.Doc
axy.nfsid.cn/288004.Doc
axy.nfsid.cn/684808.Doc
axy.nfsid.cn/262420.Doc
axy.nfsid.cn/448842.Doc
axy.nfsid.cn/408028.Doc
axy.nfsid.cn/880486.Doc
axt.nfsid.cn/480668.Doc
axt.nfsid.cn/464620.Doc
axt.nfsid.cn/046628.Doc
axt.nfsid.cn/648248.Doc
axt.nfsid.cn/628848.Doc
axt.nfsid.cn/688664.Doc
axt.nfsid.cn/684680.Doc
axt.nfsid.cn/606068.Doc
axt.nfsid.cn/484060.Doc
axt.nfsid.cn/204020.Doc
axr.nfsid.cn/628248.Doc
axr.nfsid.cn/662080.Doc
axr.nfsid.cn/400804.Doc
axr.nfsid.cn/822482.Doc
axr.nfsid.cn/480064.Doc
axr.nfsid.cn/117711.Doc
axr.nfsid.cn/466424.Doc
axr.nfsid.cn/264206.Doc
axr.nfsid.cn/428228.Doc
axr.nfsid.cn/224426.Doc
axe.nfsid.cn/482224.Doc
axe.nfsid.cn/919115.Doc
axe.nfsid.cn/428200.Doc
axe.nfsid.cn/888268.Doc
axe.nfsid.cn/266240.Doc
axe.nfsid.cn/482428.Doc
axe.nfsid.cn/864440.Doc
axe.nfsid.cn/002488.Doc
axe.nfsid.cn/648024.Doc
axe.nfsid.cn/024684.Doc
axw.nfsid.cn/424040.Doc
axw.nfsid.cn/264022.Doc
axw.nfsid.cn/048280.Doc
axw.nfsid.cn/202200.Doc
axw.nfsid.cn/222062.Doc
axw.nfsid.cn/246240.Doc
axw.nfsid.cn/084666.Doc
axw.nfsid.cn/426862.Doc
axw.nfsid.cn/024664.Doc
axw.nfsid.cn/242084.Doc
axq.nfsid.cn/840646.Doc
axq.nfsid.cn/080628.Doc
axq.nfsid.cn/486808.Doc
axq.nfsid.cn/117159.Doc
axq.nfsid.cn/006226.Doc
axq.nfsid.cn/406426.Doc
axq.nfsid.cn/628024.Doc
axq.nfsid.cn/244822.Doc
axq.nfsid.cn/008666.Doc
axq.nfsid.cn/468442.Doc
azm.nfsid.cn/866046.Doc
azm.nfsid.cn/262088.Doc
azm.nfsid.cn/460608.Doc
azm.nfsid.cn/440022.Doc
azm.nfsid.cn/660440.Doc
azm.nfsid.cn/608606.Doc
azm.nfsid.cn/028466.Doc
azm.nfsid.cn/488800.Doc
azm.nfsid.cn/808002.Doc
azm.nfsid.cn/002204.Doc
azn.nfsid.cn/826020.Doc
azn.nfsid.cn/648068.Doc
azn.nfsid.cn/422686.Doc
azn.nfsid.cn/662228.Doc
azn.nfsid.cn/886284.Doc
azn.nfsid.cn/060246.Doc
azn.nfsid.cn/466088.Doc
azn.nfsid.cn/020226.Doc
azn.nfsid.cn/220006.Doc
azn.nfsid.cn/486486.Doc
azb.nfsid.cn/000462.Doc
azb.nfsid.cn/668642.Doc
azb.nfsid.cn/006268.Doc
azb.nfsid.cn/264202.Doc
azb.nfsid.cn/228860.Doc
azb.nfsid.cn/480400.Doc
azb.nfsid.cn/884060.Doc
azb.nfsid.cn/040844.Doc
azb.nfsid.cn/802662.Doc
azb.nfsid.cn/840246.Doc
azv.nfsid.cn/082624.Doc
azv.nfsid.cn/022062.Doc
azv.nfsid.cn/860066.Doc
azv.nfsid.cn/688880.Doc
azv.nfsid.cn/606228.Doc
azv.nfsid.cn/046262.Doc
azv.nfsid.cn/244682.Doc
azv.nfsid.cn/062608.Doc
azv.nfsid.cn/862828.Doc
azv.nfsid.cn/482004.Doc
azc.nfsid.cn/406484.Doc
azc.nfsid.cn/862020.Doc
azc.nfsid.cn/062006.Doc
azc.nfsid.cn/828022.Doc
azc.nfsid.cn/240406.Doc
azc.nfsid.cn/820284.Doc
azc.nfsid.cn/428068.Doc
azc.nfsid.cn/644066.Doc
azc.nfsid.cn/482466.Doc
azc.nfsid.cn/822884.Doc
azx.nfsid.cn/284486.Doc
azx.nfsid.cn/468042.Doc
azx.nfsid.cn/422620.Doc
azx.nfsid.cn/088468.Doc
azx.nfsid.cn/648224.Doc
azx.nfsid.cn/864004.Doc
azx.nfsid.cn/828800.Doc
azx.nfsid.cn/886626.Doc
azx.nfsid.cn/002426.Doc
azx.nfsid.cn/202028.Doc
azz.irvnp.cn/468284.Doc
azz.irvnp.cn/600462.Doc
azz.irvnp.cn/262422.Doc
azz.irvnp.cn/666062.Doc
azz.irvnp.cn/086260.Doc
azz.irvnp.cn/222240.Doc
azz.irvnp.cn/620244.Doc
azz.irvnp.cn/044402.Doc
azz.irvnp.cn/404688.Doc
azz.irvnp.cn/880200.Doc
azl.irvnp.cn/022860.Doc
azl.irvnp.cn/640628.Doc
azl.irvnp.cn/842882.Doc
azl.irvnp.cn/006226.Doc
azl.irvnp.cn/822620.Doc
azl.irvnp.cn/648668.Doc
azl.irvnp.cn/888224.Doc
azl.irvnp.cn/046828.Doc
azl.irvnp.cn/682660.Doc
azl.irvnp.cn/626280.Doc
azk.irvnp.cn/686204.Doc
azk.irvnp.cn/884828.Doc
azk.irvnp.cn/606800.Doc
azk.irvnp.cn/624004.Doc
azk.irvnp.cn/026446.Doc
azk.irvnp.cn/222040.Doc
azk.irvnp.cn/606628.Doc
azk.irvnp.cn/604208.Doc
azk.irvnp.cn/244862.Doc
azk.irvnp.cn/046286.Doc
azj.irvnp.cn/442280.Doc
azj.irvnp.cn/660066.Doc
azj.irvnp.cn/468820.Doc
azj.irvnp.cn/446064.Doc
azj.irvnp.cn/088860.Doc
azj.irvnp.cn/464480.Doc
azj.irvnp.cn/046664.Doc
azj.irvnp.cn/808680.Doc
azj.irvnp.cn/004262.Doc
azj.irvnp.cn/844022.Doc
azh.irvnp.cn/020080.Doc
azh.irvnp.cn/602848.Doc
azh.irvnp.cn/448842.Doc
azh.irvnp.cn/888020.Doc
azh.irvnp.cn/646224.Doc
azh.irvnp.cn/466826.Doc
azh.irvnp.cn/644288.Doc
azh.irvnp.cn/046404.Doc
azh.irvnp.cn/808260.Doc
azh.irvnp.cn/482004.Doc
azg.irvnp.cn/460426.Doc
azg.irvnp.cn/086062.Doc
azg.irvnp.cn/688062.Doc
azg.irvnp.cn/248842.Doc
azg.irvnp.cn/864606.Doc
azg.irvnp.cn/424068.Doc
azg.irvnp.cn/424408.Doc
azg.irvnp.cn/357599.Doc
azg.irvnp.cn/248400.Doc
azg.irvnp.cn/466408.Doc
azf.irvnp.cn/880204.Doc
azf.irvnp.cn/428424.Doc
azf.irvnp.cn/860464.Doc
azf.irvnp.cn/046464.Doc
azf.irvnp.cn/288208.Doc
azf.irvnp.cn/668264.Doc
azf.irvnp.cn/008220.Doc
azf.irvnp.cn/444084.Doc
