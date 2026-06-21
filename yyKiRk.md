蹦杏训闭眉


Linux ubi_io_read UBI卷读取与VID头部校验

UBI（Unsorted Block Images）是Linux内核中管理NAND Flash的逻辑卷管理层，位于drivers/mtd/ubi/目录。ubi_io_read是UBI最底层的读取函数之一，它封装了对MTD设备的读写操作，并负责验证VID头部（Volume Identifier header）的完整性。UBI将Flash物理擦除块（PEB）映射为逻辑擦除块（LEB），所有的数据访问都通过LEB进行，VID头部是维护这一映射关系的关键元数据。

UBI的IO层包括三个核心读取函数：ubi_io_read（读取任意数据，包括EC头部、VID头部和用户数据）、ubi_io_read_ec_hdr（读取擦除计数器头部）和ubi_io_read_vid_hdr（读取卷标识头部）。其中ubi_io_read是最通用的接口：

```c
int ubi_io_read(const struct ubi_device *ubi, void *buf, int pnum,
                int offset, int len)
{
    int err, retries = 0;
    struct mtd_info *mtd = ubi->mtd;
    size_t read;
    loff_t addr;

    addr = (loff_t)pnum * ubi->peb_size + offset;
retry:
    err = mtd_read(mtd, addr, len, &read, buf);
    if (err) {
        if (err == -EUCLEAN) {
            /* ECC错误但已纠正，标记此PEB */
            ubi_warn("%s: ECC correction on PEB %d", ubi->ubi_name, pnum);
            ubi->ec_hdr_errors++;
            ubi_calculate_reserved(ubi);
            return err;
        }
        if (err == -EBADMSG) {
            /* ECC不可纠正错误 */
            if (retries++ < UBI_IO_RETRIES) {
                /* 硬件可能产生瞬态错误，尝试重新读取 */
                ubi_warn("%s: ECC error on PEB %d, retrying", ubi->ubi_name, pnum);
                goto retry;
            }
            ubi_err("%s: ECC error on PEB %d", ubi->ubi_name, pnum);
            return err;
        }
        /* 其他MTD层错误 */
        ubi_err("%s: reading %d bytes from PEB %d at %d failed",
                ubi->ubi_name, len, pnum, offset);
        return err;
    }

    if (read != len)
        return -EIO;

    return 0;
}
```

ubi_io_read返回值的特殊含义：0表示完全成功；1（即-EUCLEAN）表示读取成功但发生了ECC校正，这是一个警告信号，UBI会考虑将数据迁移到新块以避免后续不可纠正的错误；-EBADMSG表示发生了不可纠正的ECC错误，数据已损坏。

VID头部结构体ubi_vid_hdr定义了UBI卷标识头部的格式，存储在LEB起始位置的第二个头部（紧随EC头部之后）：

```c
struct ubi_vid_hdr {
    __be32  magic;          /* UBI_VID_HDR_MAGIC */
    __be8   version;        /* 头部版本号 */
    __be8   vol_type;       /* 卷类型：动态卷或静态卷 */
    __be8   copy_flag;      /* 复制标志，用于灾难恢复 */
    __be8   compat;         /* 兼容性标志 */
    __be32  vol_id;         /* 卷ID */
    __be32  lnum;           /* LEB编号 */
    __be32  compat;         /* 兼容性标志 */
    __be32  data_size;      /* 数据大小（静态卷使用） */
    __be32  used_ebs;       /* 卷使用的EB数 */
    __be32  data_pad;       /* 数据填充字节数 */
    __be32  data_crc;       /* 数据CRC校验值 */
    __be32  hdr_crc;        /* 头部CRC校验值 */
} __packed;
```

ubi_io_read_vid_hdr是读取和验证VID头部完整性的专用函数。它先读取整个VID头部数据，然后校验magic numbers和CRC：

```c
int ubi_io_read_vid_hdr(struct ubi_device *ubi, int pnum,
                         struct ubi_vid_hdr **vid_hdr, int verbose)
{
    int err, retries = 0;
    struct ubi_vid_hdr *hdr;
    uint32_t crc;

    hdr = kzalloc(ubi->vid_hdr_alsize, GFP_NOFS);
    if (!hdr)
        return -ENOMEM;

    *vid_hdr = hdr;

    /* VID头部紧接EC头部之后，偏移由vid_hdr_offset指定 */
    err = ubi_io_read(ubi, hdr, pnum, ubi->vid_hdr_offset,
                      ubi->vid_hdr_alsize);
    if (err == UBI_IO_BITFLIPS || err == -EBADMSG) {
        /* ECC相关错误，继续校验头部完整性 */
    } else if (err) {
        goto out_free;
    }

    /* 校验Magic Number */
    if (be32_to_cpu(hdr->magic) != UBI_VID_HDR_MAGIC) {
        if (verbose)
            ubi_warn("%s: bad VID header magic at PEB %d",
                     ubi->ubi_name, pnum);
        err = UBI_IO_BAD_VID_HDR;
        goto out_free;
    }

    /* 校验头部版本 */
    if (hdr->version != UBI_VERSION) {
        ubi_err("%s: bad VID header version at PEB %d",
                ubi->ubi_name, pnum);
        err = UBI_IO_BAD_VID_HDR;
        goto out_free;
    }

    /* 计算并校验头部CRC */
    crc = crc32(UBI_CRC32_INIT, hdr, UBI_VID_HDR_SIZE_CRC);
    if (be32_to_cpu(hdr->hdr_crc) != crc) {
        ubi_warn("%s: bad CRC in VID header at PEB %d",
                 ubi->ubi_name, pnum);
        err = UBI_IO_BAD_VID_HDR;
        goto out_free;
    }

    return err; /* 返回0或BITFLIPS */

out_free:
    kfree(hdr);
    *vid_hdr = NULL;
    return err;
}
```

CRC校验覆盖VID头部中hdr_crc之前的所有字段。如果CRC不匹配，函数返回UBI_IO_BAD_VID_HDR，UBI扫描阶段会标记该PEB为损坏。如果magic不匹配但EC头部有效，表示PEB处于"free"状态（已擦除但未写入数据）。

ubi_io_write是对应的写操作函数，它在写入前填充VID头部的hdr_crc字段，然后调用mtd_write。写操作完成后UBI会验证写入的数据是否正确（通过再读取并比较）。这种"write-verify"机制在NAND Flash上至关重要，因为NAND写入可能因硬件问题而静默失败。

UBI的IO层还包含一个重要的磨损均衡机制：当ubi_io_read返回-EUCLEAN（位翻转已纠正）时，UBI不会立即对该PEB进行处理，而是通过ubi_calculate_reserved更新系统的保留块计数。当错误计数超过阈值时，UBI触发scrubbing操作，将原PEB的有效数据移动到新的PEB，然后擦除原PEB。这一逻辑在ubi_wl_scrub_peb中实现。

scrubbing操作的触发条件由ubi->ec_hdr_errors和ubi->vid_hdr_errors两个计数器决定。当累计错误数超过根据Flash规格和保留块数量计算出的阈值时，UBI的工作线程（ubi_thread）执行磨损均衡和清理操作。ubi_io_read返回BITFLIPS时，调用ubi_ensure_peb_scrubbed将PEB加入scrub列表，触发数据迁移。

隙导九只渭檀杜染巴蔡诱拖啦懒擦

m.blog.kxnxh.cn/Article/details/4620866.htm
m.blog.kxnxh.cn/Article/details/5131799.htm
m.blog.kxnxh.cn/Article/details/7777759.htm
m.blog.kxnxh.cn/Article/details/3317711.htm
m.blog.kxnxh.cn/Article/details/5379953.htm
m.blog.kxnxh.cn/Article/details/9571351.htm
m.blog.kxnxh.cn/Article/details/6826228.htm
m.blog.kxnxh.cn/Article/details/3759997.htm
m.blog.kxnxh.cn/Article/details/7971751.htm
m.blog.kxnxh.cn/Article/details/7519971.htm
m.blog.kxnxh.cn/Article/details/2808046.htm
m.blog.kxnxh.cn/Article/details/9579955.htm
m.blog.kxnxh.cn/Article/details/8024846.htm
m.blog.kxnxh.cn/Article/details/9717315.htm
m.blog.kxnxh.cn/Article/details/4446680.htm
m.blog.kxnxh.cn/Article/details/3335177.htm
m.blog.kxnxh.cn/Article/details/7935755.htm
m.blog.kxnxh.cn/Article/details/6800264.htm
m.blog.kxnxh.cn/Article/details/9331959.htm
m.blog.kxnxh.cn/Article/details/6488026.htm
m.blog.kxnxh.cn/Article/details/3937739.htm
m.blog.kxnxh.cn/Article/details/3939959.htm
m.blog.kxnxh.cn/Article/details/1335995.htm
m.blog.kxnxh.cn/Article/details/5159153.htm
m.blog.kxnxh.cn/Article/details/7573915.htm
m.blog.kxnxh.cn/Article/details/1131559.htm
m.blog.kxnxh.cn/Article/details/9595575.htm
m.blog.kxnxh.cn/Article/details/1771935.htm
m.blog.kxnxh.cn/Article/details/1759115.htm
m.blog.kxnxh.cn/Article/details/2062420.htm
m.blog.kxnxh.cn/Article/details/5791535.htm
m.blog.kxnxh.cn/Article/details/1759379.htm
m.blog.kxnxh.cn/Article/details/5713973.htm
m.blog.kxnxh.cn/Article/details/6400044.htm
m.blog.kxnxh.cn/Article/details/7339575.htm
m.blog.kxnxh.cn/Article/details/7535975.htm
m.blog.kxnxh.cn/Article/details/4860224.htm
m.blog.kxnxh.cn/Article/details/3159119.htm
m.blog.kxnxh.cn/Article/details/1179175.htm
m.blog.kxnxh.cn/Article/details/1791179.htm
m.blog.kxnxh.cn/Article/details/0246644.htm
m.blog.kxnxh.cn/Article/details/2042406.htm
m.blog.kxnxh.cn/Article/details/5553935.htm
m.blog.kxnxh.cn/Article/details/8464888.htm
m.blog.kxnxh.cn/Article/details/5175557.htm
m.blog.kxnxh.cn/Article/details/7117933.htm
m.blog.kxnxh.cn/Article/details/6206608.htm
m.blog.kxnxh.cn/Article/details/4200206.htm
m.blog.kxnxh.cn/Article/details/8622268.htm
m.blog.kxnxh.cn/Article/details/8806660.htm
m.blog.kxnxh.cn/Article/details/1795119.htm
m.blog.kxnxh.cn/Article/details/5199997.htm
m.blog.kxnxh.cn/Article/details/2284086.htm
m.blog.kxnxh.cn/Article/details/3533173.htm
m.blog.kxnxh.cn/Article/details/6066244.htm
m.blog.kxnxh.cn/Article/details/2048622.htm
m.blog.kxnxh.cn/Article/details/2062666.htm
m.blog.kxnxh.cn/Article/details/6628460.htm
m.blog.kxnxh.cn/Article/details/3957531.htm
m.blog.kxnxh.cn/Article/details/8488002.htm
m.blog.kxnxh.cn/Article/details/2486280.htm
m.blog.kxnxh.cn/Article/details/1591777.htm
m.blog.kxnxh.cn/Article/details/0224446.htm
m.blog.kxnxh.cn/Article/details/0888288.htm
m.blog.kxnxh.cn/Article/details/7517913.htm
m.blog.kxnxh.cn/Article/details/9535573.htm
m.blog.kxnxh.cn/Article/details/5911395.htm
m.blog.kxnxh.cn/Article/details/6468602.htm
m.blog.kxnxh.cn/Article/details/7379397.htm
m.blog.kxnxh.cn/Article/details/7315919.htm
m.blog.kxnxh.cn/Article/details/1753391.htm
m.blog.kxnxh.cn/Article/details/2602408.htm
m.blog.kxnxh.cn/Article/details/7533573.htm
m.blog.kxnxh.cn/Article/details/2284682.htm
m.blog.kxnxh.cn/Article/details/6824206.htm
m.blog.kxnxh.cn/Article/details/8628662.htm
m.blog.kxnxh.cn/Article/details/7177915.htm
m.blog.kxnxh.cn/Article/details/5759511.htm
m.blog.kxnxh.cn/Article/details/2084866.htm
m.blog.kxnxh.cn/Article/details/1933513.htm
m.blog.kxnxh.cn/Article/details/9977957.htm
m.blog.kxnxh.cn/Article/details/0448048.htm
m.blog.kxnxh.cn/Article/details/0844682.htm
m.blog.kxnxh.cn/Article/details/3759535.htm
m.blog.kxnxh.cn/Article/details/9915539.htm
m.blog.kxnxh.cn/Article/details/1793531.htm
m.blog.kxnxh.cn/Article/details/8444684.htm
m.blog.kxnxh.cn/Article/details/7731115.htm
m.blog.kxnxh.cn/Article/details/1773755.htm
m.blog.kxnxh.cn/Article/details/0408242.htm
m.blog.kxnxh.cn/Article/details/7335191.htm
m.blog.kxnxh.cn/Article/details/3935777.htm
m.blog.kxnxh.cn/Article/details/3717375.htm
m.blog.kxnxh.cn/Article/details/5195333.htm
m.blog.kxnxh.cn/Article/details/1591379.htm
m.blog.kxnxh.cn/Article/details/5391355.htm
m.blog.kxnxh.cn/Article/details/8044000.htm
m.blog.kxnxh.cn/Article/details/7937719.htm
m.blog.kxnxh.cn/Article/details/1135711.htm
m.blog.kxnxh.cn/Article/details/4604282.htm
m.blog.kxnxh.cn/Article/details/9379751.htm
m.blog.kxnxh.cn/Article/details/5993991.htm
m.blog.kxnxh.cn/Article/details/6260060.htm
m.blog.kxnxh.cn/Article/details/2246222.htm
m.blog.kxnxh.cn/Article/details/3513337.htm
m.blog.kxnxh.cn/Article/details/9939991.htm
m.blog.kxnxh.cn/Article/details/5517735.htm
m.blog.kxnxh.cn/Article/details/2086488.htm
m.blog.kxnxh.cn/Article/details/5715995.htm
m.blog.kxnxh.cn/Article/details/6826286.htm
m.blog.kxnxh.cn/Article/details/3315959.htm
m.blog.kxnxh.cn/Article/details/6622086.htm
m.blog.kxnxh.cn/Article/details/7375377.htm
m.blog.kxnxh.cn/Article/details/1719795.htm
m.blog.kxnxh.cn/Article/details/0664882.htm
m.blog.kxnxh.cn/Article/details/5777139.htm
m.blog.kxnxh.cn/Article/details/8826488.htm
m.blog.kxnxh.cn/Article/details/3957551.htm
m.blog.kxnxh.cn/Article/details/3911997.htm
m.blog.kxnxh.cn/Article/details/6822684.htm
m.blog.kxnxh.cn/Article/details/4024200.htm
m.blog.kxnxh.cn/Article/details/3953535.htm
m.blog.kxnxh.cn/Article/details/5737551.htm
m.blog.kxnxh.cn/Article/details/7595153.htm
m.blog.kxnxh.cn/Article/details/9773755.htm
m.blog.kxnxh.cn/Article/details/9771191.htm
m.blog.kxnxh.cn/Article/details/3355735.htm
m.blog.kxnxh.cn/Article/details/9917593.htm
m.blog.kxnxh.cn/Article/details/9335919.htm
m.blog.kxnxh.cn/Article/details/4828640.htm
m.blog.kxnxh.cn/Article/details/1399953.htm
m.blog.kxnxh.cn/Article/details/7335119.htm
m.blog.kxnxh.cn/Article/details/7715730.htm
m.blog.kxnxh.cn/Article/details/1799195.htm
m.blog.kxnxh.cn/Article/details/4600842.htm
m.blog.kxnxh.cn/Article/details/2288222.htm
m.blog.kxnxh.cn/Article/details/9971755.htm
m.blog.kxnxh.cn/Article/details/5917593.htm
m.blog.kxnxh.cn/Article/details/6402020.htm
m.blog.kxnxh.cn/Article/details/9931957.htm
m.blog.kxnxh.cn/Article/details/7117199.htm
m.blog.kxnxh.cn/Article/details/1153377.htm
m.blog.kxnxh.cn/Article/details/0042846.htm
m.blog.kxnxh.cn/Article/details/1971597.htm
m.blog.kxnxh.cn/Article/details/0048828.htm
m.blog.kxnxh.cn/Article/details/3977577.htm
m.blog.kxnxh.cn/Article/details/2206606.htm
m.blog.kxnxh.cn/Article/details/1331575.htm
m.blog.kxnxh.cn/Article/details/4862046.htm
m.blog.kxnxh.cn/Article/details/7793115.htm
m.blog.kxnxh.cn/Article/details/8086440.htm
m.blog.kxnxh.cn/Article/details/3315119.htm
m.blog.kxnxh.cn/Article/details/7373917.htm
m.blog.kxnxh.cn/Article/details/9333513.htm
m.blog.kxnxh.cn/Article/details/8602244.htm
m.blog.kxnxh.cn/Article/details/2624666.htm
m.blog.kxnxh.cn/Article/details/8466446.htm
m.blog.kxnxh.cn/Article/details/7175399.htm
m.blog.kxnxh.cn/Article/details/0224460.htm
m.blog.kxnxh.cn/Article/details/0268806.htm
m.blog.kxnxh.cn/Article/details/7973977.htm
m.blog.kxnxh.cn/Article/details/4240686.htm
m.blog.kxnxh.cn/Article/details/9739399.htm
m.blog.kxnxh.cn/Article/details/4460206.htm
m.blog.kxnxh.cn/Article/details/1737959.htm
m.blog.kxnxh.cn/Article/details/4026860.htm
m.blog.kxnxh.cn/Article/details/1593731.htm
m.blog.kxnxh.cn/Article/details/5935373.htm
m.blog.kxnxh.cn/Article/details/3939997.htm
m.blog.kxnxh.cn/Article/details/5175595.htm
m.blog.kxnxh.cn/Article/details/9151151.htm
m.blog.kxnxh.cn/Article/details/7533933.htm
m.blog.kxnxh.cn/Article/details/7911191.htm
m.blog.kxnxh.cn/Article/details/1177917.htm
m.blog.kxnxh.cn/Article/details/3793159.htm
m.blog.kxnxh.cn/Article/details/9717399.htm
m.blog.kxnxh.cn/Article/details/9557757.htm
m.blog.kxnxh.cn/Article/details/4604860.htm
m.blog.kxnxh.cn/Article/details/7197193.htm
m.blog.kxnxh.cn/Article/details/2400628.htm
m.blog.kxnxh.cn/Article/details/9777535.htm
m.blog.kxnxh.cn/Article/details/1533575.htm
m.blog.kxnxh.cn/Article/details/4420400.htm
m.blog.kxnxh.cn/Article/details/5351393.htm
m.blog.kxnxh.cn/Article/details/6822004.htm
m.blog.kxnxh.cn/Article/details/8406626.htm
m.blog.kxnxh.cn/Article/details/5955195.htm
m.blog.kxnxh.cn/Article/details/3173539.htm
m.blog.kxnxh.cn/Article/details/7113753.htm
m.blog.kxnxh.cn/Article/details/6606004.htm
m.blog.kxnxh.cn/Article/details/5199115.htm
m.blog.kxnxh.cn/Article/details/0200284.htm
m.blog.kxnxh.cn/Article/details/7777159.htm
m.blog.kxnxh.cn/Article/details/4848400.htm
m.blog.kxnxh.cn/Article/details/0062248.htm
m.blog.kxnxh.cn/Article/details/9397199.htm
m.blog.kxnxh.cn/Article/details/1357371.htm
m.blog.kxnxh.cn/Article/details/7317773.htm
m.blog.kxnxh.cn/Article/details/9357593.htm
m.blog.kxnxh.cn/Article/details/7957555.htm
m.blog.kxnxh.cn/Article/details/1197175.htm
m.blog.kxnxh.cn/Article/details/5973355.htm
m.blog.kxnxh.cn/Article/details/4066008.htm
m.blog.kxnxh.cn/Article/details/7915513.htm
m.blog.kxnxh.cn/Article/details/8428424.htm
m.blog.kxnxh.cn/Article/details/4844802.htm
m.blog.kxnxh.cn/Article/details/8088068.htm
m.blog.kxnxh.cn/Article/details/3355599.htm
m.blog.kxnxh.cn/Article/details/6628804.htm
m.blog.kxnxh.cn/Article/details/8660464.htm
m.blog.kxnxh.cn/Article/details/5579951.htm
m.blog.kxnxh.cn/Article/details/6868444.htm
m.blog.kxnxh.cn/Article/details/7933193.htm
m.blog.kxnxh.cn/Article/details/9973119.htm
m.blog.kxnxh.cn/Article/details/9997975.htm
m.blog.kxnxh.cn/Article/details/6660242.htm
m.blog.kxnxh.cn/Article/details/5371511.htm
m.blog.kxnxh.cn/Article/details/7195713.htm
m.blog.kxnxh.cn/Article/details/1959557.htm
m.blog.kxnxh.cn/Article/details/4200626.htm
m.blog.kxnxh.cn/Article/details/8426466.htm
m.blog.kxnxh.cn/Article/details/9173995.htm
m.blog.kxnxh.cn/Article/details/7111131.htm
m.blog.kxnxh.cn/Article/details/8646084.htm
m.blog.kxnxh.cn/Article/details/5577971.htm
m.blog.kxnxh.cn/Article/details/8668842.htm
m.blog.kxnxh.cn/Article/details/4088802.htm
m.blog.kxnxh.cn/Article/details/1735335.htm
m.blog.kxnxh.cn/Article/details/9753977.htm
m.blog.kxnxh.cn/Article/details/9939535.htm
m.blog.kxnxh.cn/Article/details/1195115.htm
m.blog.kxnxh.cn/Article/details/0462248.htm
m.blog.kxnxh.cn/Article/details/3955755.htm
m.blog.kxnxh.cn/Article/details/4888640.htm
m.blog.kxnxh.cn/Article/details/4680460.htm
m.blog.kxnxh.cn/Article/details/0604424.htm
m.blog.kxnxh.cn/Article/details/5393531.htm
m.blog.kxnxh.cn/Article/details/8242204.htm
m.blog.kxnxh.cn/Article/details/1931531.htm
m.blog.kxnxh.cn/Article/details/3919517.htm
m.blog.kxnxh.cn/Article/details/6026008.htm
m.blog.kxnxh.cn/Article/details/7133335.htm
m.blog.kxnxh.cn/Article/details/0808460.htm
m.blog.kxnxh.cn/Article/details/1377759.htm
m.blog.kxnxh.cn/Article/details/9557955.htm
m.blog.kxnxh.cn/Article/details/8606442.htm
m.blog.kxnxh.cn/Article/details/7959317.htm
m.blog.kxnxh.cn/Article/details/4284482.htm
m.blog.kxnxh.cn/Article/details/3773717.htm
m.blog.kxnxh.cn/Article/details/5973319.htm
m.blog.kxnxh.cn/Article/details/4260244.htm
m.blog.kxnxh.cn/Article/details/3735735.htm
m.blog.kxnxh.cn/Article/details/7537157.htm
m.blog.kxnxh.cn/Article/details/5977395.htm
m.blog.kxnxh.cn/Article/details/9175795.htm
m.blog.kxnxh.cn/Article/details/1353995.htm
m.blog.kxnxh.cn/Article/details/6048626.htm
m.blog.kxnxh.cn/Article/details/9395593.htm
m.blog.kxnxh.cn/Article/details/4446442.htm
m.blog.kxnxh.cn/Article/details/2048886.htm
m.blog.kxnxh.cn/Article/details/9317971.htm
m.blog.kxnxh.cn/Article/details/9955955.htm
m.blog.kxnxh.cn/Article/details/8482604.htm
m.blog.kxnxh.cn/Article/details/7731537.htm
m.blog.kxnxh.cn/Article/details/3751915.htm
m.blog.kxnxh.cn/Article/details/3559951.htm
m.blog.kxnxh.cn/Article/details/3731195.htm
m.blog.kxnxh.cn/Article/details/1355779.htm
m.blog.kxnxh.cn/Article/details/3175391.htm
m.blog.kxnxh.cn/Article/details/9375955.htm
m.blog.kxnxh.cn/Article/details/9791395.htm
m.blog.kxnxh.cn/Article/details/0842486.htm
m.blog.kxnxh.cn/Article/details/2420020.htm
m.blog.kxnxh.cn/Article/details/7575533.htm
m.blog.kxnxh.cn/Article/details/9155315.htm
m.blog.kxnxh.cn/Article/details/8680826.htm
m.blog.kxnxh.cn/Article/details/7119197.htm
m.blog.kxnxh.cn/Article/details/6428842.htm
m.blog.kxnxh.cn/Article/details/5153973.htm
m.blog.kxnxh.cn/Article/details/9371733.htm
m.blog.kxnxh.cn/Article/details/8008260.htm
m.blog.kxnxh.cn/Article/details/7159571.htm
m.blog.kxnxh.cn/Article/details/2422224.htm
m.blog.kxnxh.cn/Article/details/6448080.htm
m.blog.kxnxh.cn/Article/details/4680266.htm
m.blog.kxnxh.cn/Article/details/5339773.htm
m.blog.kxnxh.cn/Article/details/5117317.htm
m.blog.kxnxh.cn/Article/details/3737375.htm
m.blog.kxnxh.cn/Article/details/7973559.htm
m.blog.kxnxh.cn/Article/details/4666226.htm
m.blog.kxnxh.cn/Article/details/7793957.htm
m.blog.kxnxh.cn/Article/details/6062266.htm
m.blog.kxnxh.cn/Article/details/7759775.htm
m.blog.kxnxh.cn/Article/details/2622420.htm
m.blog.kxnxh.cn/Article/details/4082880.htm
m.blog.kxnxh.cn/Article/details/6208660.htm
m.blog.kxnxh.cn/Article/details/9577779.htm
m.blog.kxnxh.cn/Article/details/3155175.htm
m.blog.kxnxh.cn/Article/details/7333195.htm
m.blog.kxnxh.cn/Article/details/9735919.htm
m.blog.kxnxh.cn/Article/details/5751793.htm
m.blog.kxnxh.cn/Article/details/8682840.htm
m.blog.kxnxh.cn/Article/details/7991575.htm
m.blog.kxnxh.cn/Article/details/0862064.htm
m.blog.kxnxh.cn/Article/details/7117115.htm
m.blog.kxnxh.cn/Article/details/7193397.htm
m.blog.kxnxh.cn/Article/details/5977731.htm
m.blog.kxnxh.cn/Article/details/3717371.htm
m.blog.kxnxh.cn/Article/details/8420866.htm
m.blog.kxnxh.cn/Article/details/4882884.htm
m.blog.kxnxh.cn/Article/details/8468824.htm
m.blog.kxnxh.cn/Article/details/1999953.htm
m.blog.kxnxh.cn/Article/details/5153991.htm
m.blog.kxnxh.cn/Article/details/3317797.htm
m.blog.kxnxh.cn/Article/details/8408862.htm
m.blog.kxnxh.cn/Article/details/7171599.htm
m.blog.kxnxh.cn/Article/details/3739795.htm
m.blog.kxnxh.cn/Article/details/5131935.htm
m.blog.kxnxh.cn/Article/details/0800468.htm
m.blog.kxnxh.cn/Article/details/8860242.htm
m.blog.kxnxh.cn/Article/details/9591939.htm
m.blog.kxnxh.cn/Article/details/7151559.htm
m.blog.kxnxh.cn/Article/details/7399977.htm
m.blog.kxnxh.cn/Article/details/0640686.htm
m.blog.kxnxh.cn/Article/details/6206226.htm
m.blog.kxnxh.cn/Article/details/1791779.htm
m.blog.kxnxh.cn/Article/details/2228208.htm
m.blog.kxnxh.cn/Article/details/7131515.htm
m.blog.kxnxh.cn/Article/details/1799771.htm
m.blog.kxnxh.cn/Article/details/5797731.htm
m.blog.kxnxh.cn/Article/details/2648404.htm
m.blog.kxnxh.cn/Article/details/0668040.htm
m.blog.kxnxh.cn/Article/details/7333531.htm
m.blog.kxnxh.cn/Article/details/3739373.htm
m.blog.kxnxh.cn/Article/details/7917379.htm
m.blog.kxnxh.cn/Article/details/7975997.htm
m.blog.kxnxh.cn/Article/details/7199357.htm
m.blog.kxnxh.cn/Article/details/8684246.htm
m.blog.kxnxh.cn/Article/details/2808448.htm
m.blog.kxnxh.cn/Article/details/6020200.htm
m.blog.kxnxh.cn/Article/details/0402642.htm
m.blog.kxnxh.cn/Article/details/2842208.htm
m.blog.kxnxh.cn/Article/details/1137559.htm
m.blog.kxnxh.cn/Article/details/3395359.htm
m.blog.kxnxh.cn/Article/details/0828420.htm
m.blog.kxnxh.cn/Article/details/3793599.htm
m.blog.kxnxh.cn/Article/details/3577713.htm
m.blog.kxnxh.cn/Article/details/9335939.htm
m.blog.kxnxh.cn/Article/details/2266406.htm
m.blog.kxnxh.cn/Article/details/5757553.htm
m.blog.kxnxh.cn/Article/details/9157951.htm
m.blog.kxnxh.cn/Article/details/1917313.htm
m.blog.kxnxh.cn/Article/details/1535397.htm
m.blog.kxnxh.cn/Article/details/0264802.htm
m.blog.kxnxh.cn/Article/details/9771137.htm
m.blog.kxnxh.cn/Article/details/7533795.htm
m.blog.kxnxh.cn/Article/details/7599977.htm
m.blog.kxnxh.cn/Article/details/0022688.htm
m.blog.kxnxh.cn/Article/details/5519993.htm
m.blog.kxnxh.cn/Article/details/3519337.htm
m.blog.kxnxh.cn/Article/details/7933317.htm
m.blog.kxnxh.cn/Article/details/7531317.htm
m.blog.kxnxh.cn/Article/details/2686628.htm
m.blog.kxnxh.cn/Article/details/2644440.htm
m.blog.kxnxh.cn/Article/details/6466466.htm
m.blog.kxnxh.cn/Article/details/4682280.htm
m.blog.kxnxh.cn/Article/details/3559557.htm
m.blog.kxnxh.cn/Article/details/1719971.htm
m.blog.kxnxh.cn/Article/details/1159117.htm
m.blog.kxnxh.cn/Article/details/8880686.htm
m.blog.kxnxh.cn/Article/details/1513733.htm
m.blog.kxnxh.cn/Article/details/4860464.htm
m.blog.kxnxh.cn/Article/details/5111171.htm
m.blog.kxnxh.cn/Article/details/3391717.htm
m.blog.kxnxh.cn/Article/details/5955155.htm
m.blog.kxnxh.cn/Article/details/5591335.htm
m.blog.kxnxh.cn/Article/details/7931111.htm
m.blog.kxnxh.cn/Article/details/9791975.htm
m.blog.kxnxh.cn/Article/details/5979913.htm
m.blog.kxnxh.cn/Article/details/1991713.htm
m.blog.kxnxh.cn/Article/details/6806280.htm
m.blog.kxnxh.cn/Article/details/7331719.htm
m.blog.kxnxh.cn/Article/details/6640446.htm
m.blog.kxnxh.cn/Article/details/9557753.htm
m.blog.kxnxh.cn/Article/details/9719139.htm
m.blog.kxnxh.cn/Article/details/7379737.htm
m.blog.kxnxh.cn/Article/details/0422600.htm
m.blog.kxnxh.cn/Article/details/6864284.htm
m.blog.kxnxh.cn/Article/details/5179157.htm
m.blog.kxnxh.cn/Article/details/5357195.htm
m.blog.kxnxh.cn/Article/details/7975117.htm
m.blog.kxnxh.cn/Article/details/6084884.htm
m.blog.kxnxh.cn/Article/details/7753913.htm
m.blog.kxnxh.cn/Article/details/2686422.htm
m.blog.kxnxh.cn/Article/details/0462022.htm
