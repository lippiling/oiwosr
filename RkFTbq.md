慰峡柯眉逃


Linux sd_sync_cache SCSI同步缓存SYNCHRONIZE_CACHE

sd_sync_cache是SCSI磁盘驱动程序（sd_mod）中实现缓存同步的函数，位于drivers/scsi/sd.c。当文件系统或块层发出BLKFLSBUF/CACHE_FLUSH等缓存刷新请求时，该函数构造并发送SCSI SYNCHRONIZE_CACHE命令到存储设备，确保易失性缓存中的数据被写入持久化介质。

函数入口由sd_init_command中的REQ_OP_FLUSH分支触发：

```c
static int sd_sync_cache(struct scsi_disk *sdkp, int write_enable)
{
    struct scsi_device *sdp = sdkp->device;
    unsigned char cmd[16];
    unsigned char data[512];
    int ret = 0, retries;

    /* 构造SYNCHRONIZE CACHE(10) CDB */
    memset(cmd, 0, sizeof(cmd));
    cmd[0] = SYNCHRONIZE_CACHE;
    cmd[1] = 0;

    if (write_enable) {
        /* 设置写保护检查位 */
        if (sdp->wdrep)
            cmd[1] |= 0x04;
        cmd[1] |= 0x02;  /* Immediate bit */
    }

    /* 若支持16字节命令，使用SYNCHRONIZE CACHE(16) */
    if (sdp->use_16_for_sync) {
        memset(cmd, 0, sizeof(cmd));
        cmd[0] = SYNCHRONIZE_CACHE_16;
        cmd[1] = 0;
        if (write_enable) {
            if (sdp->wdrep)
                cmd[1] |= 0x08;
            cmd[1] |= 0x02;
        }
        /* LBA范围：全0表示整个设备 */
        cmd[10] = 0;
        cmd[11] = 0;
        cmd[12] = 0;
        cmd[13] = 0;  /* 范围长度，0表示到结束 */
    }

    /* 执行命令，允许在UA条件下重试 */
    retries = SD_MAX_RETRIES;
    do {
        ret = scsi_execute_cmd(sdp, cmd, REQ_OP_DRV_IN, NULL, 0,
                               SD_TIMEOUT, retries, NULL);
        if (ret == 0)
            break;

        /* 检查特定sense码 */
        if (scsi_sense_valid(scsi_req->sense)) {
            if (scsi_sense_is_deferred_error(scsi_req->sense)) {
                /* 延期错误，轮询直至完成 */
                ret = sd_sync_cache_poll(sdkp, sd_timeout);
                break;
            }
        }

        retries--;
    } while (retries > 0);

    return ret;
}
```

scsi_execute_cmd是底层SCSI命令执行函数，将CDB包装为struct scsi_request提交给SCSI中层：

```c
int scsi_execute_cmd(struct scsi_device *sdev, const unsigned char *cmd,
                     blk_opf_t opf, void *buffer, unsigned int bufflen,
                     int timeout, int retries, u64 *flags)
{
    struct request *rq;
    struct scsi_request *rqst;
    int ret;

    rq = scsi_alloc_request(sdev->request_queue, opf, 0);
    if (IS_ERR(rq))
        return PTR_ERR(rq);

    rqst = scsi_req(rq);
    rqst->cmd_len = 16;
    memcpy(rqst->cmd, cmd, rqst->cmd_len);

    if (buffer && bufflen) {
        ret = blk_rq_map_kern(sdev->request_queue, rq, buffer, bufflen,
                              GFP_NOIO);
        if (ret)
            goto out;
    }

    rq->timeout = timeout;
    rq->retries = retries;

    ret = blk_execute_rq(rq, false);
    if (ret)
        goto out;

    ret = rqst->result;

out:
    blk_mq_free_request(rq);
    return ret;
}
```

块层触发sd_sync_cache的路径始于文件系统的blkdev_issue_flush：

```c
int blkdev_issue_flush(struct block_device *bdev)
{
    struct request_queue *q = bdev->bd_queue;
    struct bio *bio;
    int ret;

    if (!blk_queue_write_cache(q))
        return 0;

    bio = bio_alloc(bdev, 0, REQ_OP_FLUSH | REQ_PREFLUSH, GFP_NOIO);
    ret = submit_bio_wait(bio);
    bio_put(bio);

    return ret;
}
```

该bio在块层经q->make_request_fn处理后到达scsi中层dispatch，最终由sd_init_command识别FLUSH请求并调用sd_sync_cache。sd_init_command是sd驱动的主命令分发函数：

```c
static blk_status_t sd_init_command(struct scsi_cmnd *cmd)
{
    struct request *rq = cmd->request;
    struct scsi_disk *sdkp = scsi_disk(rq->rq_disk);
    int ret;

    switch (req_op(rq)) {
    case REQ_OP_FLUSH:
        ret = sd_sync_cache(sdkp, !(rq->cmd_flags & REQ_FUA));
        if (ret)
            return BLK_STS_IOERR;
        return BLK_STS_OK;

    case REQ_OP_READ:
    case REQ_OP_WRITE:
        return sd_read_write(cmd, rq);

    case REQ_OP_DISCARD:
        return sd_do_discard(cmd, rq);

    case REQ_OP_WRITE_SAME:
        return sd_do_write_same(cmd, rq);

    default:
        return BLK_STS_IOERR;
    }
}
```

SYNCHRONIZE_CACHE命令执行完成的sense码处理至关重要。如果设备返回GOOD状态，缓存确保已刷新。若返回CHECK CONDITION，需要解析sense data：

```c
static int sd_sync_cache_completed(struct scsi_cmnd *cmd)
{
    struct scsi_sense_hdr sshdr;
    int ret;

    if (cmd->result == 0)
        return 0;

    if (scsi_normalize_sense(cmd->sense_buffer, SCSI_SENSE_BUFFERSIZE,
                             &sshdr)) {
        switch (sshdr.sense_key) {
        case UNIT_ATTENTION:
            /* UA条件重试 */
            return -EAGAIN;
        case ABORTED_COMMAND:
            /* 命令中止，可能介质已更换 */
            return -ENOMEDIUM;
        case ILLEGAL_REQUEST:
            /* 命令不支持降级处理 */
            if (sshdr.asc == 0x20)
                return -ENOTSUPP;
            break;
        case MEDIUM_ERROR:
            return -EIO;
        }
    }

    return -EIO;
}
```

对于不支持SYNCHRONIZE_CACHE的老设备，sd驱动会捕获ILLEGAL_REQUEST错误，自动降级为无操作，因为这类设备通常不具备易失性缓存。现代SCSI和SAS/SATA设备通过标准SYNCHRONIZE_CACHE命令确保数据持久化，与NVMe的FLUSH命令功能对应，是块设备保证数据完整性（write barrier）的关键实现路径。

乖景谄铱返辞屎牧赶丈陈尉呐妥偈

hrd.vwbnt.cn/868648.Doc
hrd.vwbnt.cn/426422.Doc
hrd.vwbnt.cn/815482.Doc
hrd.vwbnt.cn/284826.Doc
hrd.vwbnt.cn/402684.Doc
hrd.vwbnt.cn/684004.Doc
hrs.vwbnt.cn/860606.Doc
hrs.vwbnt.cn/046004.Doc
hrs.vwbnt.cn/760277.Doc
hrs.vwbnt.cn/002624.Doc
hrs.vwbnt.cn/737317.Doc
hrs.vwbnt.cn/462842.Doc
hrs.vwbnt.cn/466975.Doc
hrs.vwbnt.cn/242026.Doc
hrs.vwbnt.cn/064482.Doc
hrs.vwbnt.cn/642002.Doc
hra.vwbnt.cn/356861.Doc
hra.vwbnt.cn/080880.Doc
hra.vwbnt.cn/160255.Doc
hra.vwbnt.cn/082044.Doc
hra.vwbnt.cn/802422.Doc
hra.vwbnt.cn/480064.Doc
hra.vwbnt.cn/936868.Doc
hra.vwbnt.cn/880246.Doc
hra.vwbnt.cn/864426.Doc
hra.vwbnt.cn/486280.Doc
hrp.vwbnt.cn/119311.Doc
hrp.vwbnt.cn/828002.Doc
hrp.vwbnt.cn/802808.Doc
hrp.vwbnt.cn/240028.Doc
hrp.vwbnt.cn/886628.Doc
hrp.vwbnt.cn/460202.Doc
hrp.vwbnt.cn/888824.Doc
hrp.vwbnt.cn/839571.Doc
hrp.vwbnt.cn/646266.Doc
hrp.vwbnt.cn/284000.Doc
hro.vwbnt.cn/246002.Doc
hro.vwbnt.cn/846628.Doc
hro.vwbnt.cn/280400.Doc
hro.vwbnt.cn/644682.Doc
hro.vwbnt.cn/808662.Doc
hro.vwbnt.cn/060622.Doc
hro.vwbnt.cn/622442.Doc
hro.vwbnt.cn/022044.Doc
hro.vwbnt.cn/522596.Doc
hro.vwbnt.cn/880086.Doc
hri.vwbnt.cn/263410.Doc
hri.vwbnt.cn/626682.Doc
hri.vwbnt.cn/844268.Doc
hri.vwbnt.cn/779597.Doc
hri.vwbnt.cn/864088.Doc
hri.vwbnt.cn/475467.Doc
hri.vwbnt.cn/084284.Doc
hri.vwbnt.cn/608224.Doc
hri.vwbnt.cn/408246.Doc
hri.vwbnt.cn/804608.Doc
hru.vwbnt.cn/088280.Doc
hru.vwbnt.cn/220400.Doc
hru.vwbnt.cn/711912.Doc
hru.vwbnt.cn/193391.Doc
hru.vwbnt.cn/546341.Doc
hru.vwbnt.cn/755979.Doc
hru.vwbnt.cn/824860.Doc
hru.vwbnt.cn/704188.Doc
hru.vwbnt.cn/240686.Doc
hru.vwbnt.cn/480442.Doc
hry.vwbnt.cn/222608.Doc
hry.vwbnt.cn/917375.Doc
hry.vwbnt.cn/486799.Doc
hry.vwbnt.cn/668206.Doc
hry.vwbnt.cn/603883.Doc
hry.vwbnt.cn/446208.Doc
hry.vwbnt.cn/884866.Doc
hry.vwbnt.cn/822424.Doc
hry.vwbnt.cn/248022.Doc
hry.vwbnt.cn/624444.Doc
hrt.vwbnt.cn/121889.Doc
hrt.vwbnt.cn/620044.Doc
hrt.vwbnt.cn/800628.Doc
hrt.vwbnt.cn/806682.Doc
hrt.vwbnt.cn/488804.Doc
hrt.vwbnt.cn/844420.Doc
hrt.vwbnt.cn/273207.Doc
hrt.vwbnt.cn/775571.Doc
hrt.vwbnt.cn/600024.Doc
hrt.vwbnt.cn/638089.Doc
hrr.vwbnt.cn/686044.Doc
hrr.vwbnt.cn/020248.Doc
hrr.vwbnt.cn/218173.Doc
hrr.vwbnt.cn/082028.Doc
hrr.vwbnt.cn/444222.Doc
hrr.vwbnt.cn/880488.Doc
hrr.vwbnt.cn/682264.Doc
hrr.vwbnt.cn/626048.Doc
hrr.vwbnt.cn/375597.Doc
hrr.vwbnt.cn/064444.Doc
hre.vwbnt.cn/426604.Doc
hre.vwbnt.cn/064668.Doc
hre.vwbnt.cn/084206.Doc
hre.vwbnt.cn/442462.Doc
hre.vwbnt.cn/444242.Doc
hre.vwbnt.cn/820620.Doc
hre.vwbnt.cn/820066.Doc
hre.vwbnt.cn/717339.Doc
hre.vwbnt.cn/886888.Doc
hre.vwbnt.cn/515553.Doc
hrw.vwbnt.cn/062428.Doc
hrw.vwbnt.cn/038710.Doc
hrw.vwbnt.cn/935753.Doc
hrw.vwbnt.cn/680884.Doc
hrw.vwbnt.cn/022066.Doc
hrw.vwbnt.cn/606446.Doc
hrw.vwbnt.cn/006408.Doc
hrw.vwbnt.cn/046510.Doc
hrw.vwbnt.cn/406622.Doc
hrw.vwbnt.cn/466006.Doc
hrq.vwbnt.cn/775773.Doc
hrq.vwbnt.cn/422644.Doc
hrq.vwbnt.cn/260600.Doc
hrq.vwbnt.cn/411833.Doc
hrq.vwbnt.cn/442844.Doc
hrq.vwbnt.cn/622680.Doc
hrq.vwbnt.cn/668482.Doc
hrq.vwbnt.cn/420042.Doc
hrq.vwbnt.cn/228608.Doc
hrq.vwbnt.cn/860804.Doc
hem.vwbnt.cn/791559.Doc
hem.vwbnt.cn/660648.Doc
hem.vwbnt.cn/266206.Doc
hem.vwbnt.cn/758620.Doc
hem.vwbnt.cn/457276.Doc
hem.vwbnt.cn/862204.Doc
hem.vwbnt.cn/762027.Doc
hem.vwbnt.cn/935155.Doc
hem.vwbnt.cn/644248.Doc
hem.vwbnt.cn/884022.Doc
hen.vwbnt.cn/355993.Doc
hen.vwbnt.cn/442600.Doc
hen.vwbnt.cn/161724.Doc
hen.vwbnt.cn/119519.Doc
hen.vwbnt.cn/286684.Doc
hen.vwbnt.cn/440648.Doc
hen.vwbnt.cn/684202.Doc
hen.vwbnt.cn/020660.Doc
hen.vwbnt.cn/959253.Doc
hen.vwbnt.cn/020488.Doc
heb.vwbnt.cn/175064.Doc
heb.vwbnt.cn/644264.Doc
heb.vwbnt.cn/604442.Doc
heb.vwbnt.cn/042282.Doc
heb.vwbnt.cn/822802.Doc
heb.vwbnt.cn/282284.Doc
heb.vwbnt.cn/888622.Doc
heb.vwbnt.cn/424842.Doc
heb.vwbnt.cn/060680.Doc
heb.vwbnt.cn/406484.Doc
hev.vwbnt.cn/848624.Doc
hev.vwbnt.cn/533591.Doc
hev.vwbnt.cn/684822.Doc
hev.vwbnt.cn/844082.Doc
hev.vwbnt.cn/668602.Doc
hev.vwbnt.cn/282062.Doc
hev.vwbnt.cn/086404.Doc
hev.vwbnt.cn/383412.Doc
hev.vwbnt.cn/571599.Doc
hev.vwbnt.cn/208280.Doc
hec.vwbnt.cn/020880.Doc
hec.vwbnt.cn/604064.Doc
hec.vwbnt.cn/953193.Doc
hec.vwbnt.cn/186117.Doc
hec.vwbnt.cn/200402.Doc
hec.vwbnt.cn/242406.Doc
hec.vwbnt.cn/864466.Doc
hec.vwbnt.cn/402284.Doc
hec.vwbnt.cn/424880.Doc
hec.vwbnt.cn/375791.Doc
hex.vwbnt.cn/480606.Doc
hex.vwbnt.cn/743272.Doc
hex.vwbnt.cn/357379.Doc
hex.vwbnt.cn/602626.Doc
hex.vwbnt.cn/682880.Doc
hex.vwbnt.cn/620686.Doc
hex.vwbnt.cn/846208.Doc
hex.vwbnt.cn/294330.Doc
hex.vwbnt.cn/664460.Doc
hex.vwbnt.cn/004260.Doc
hez.vwbnt.cn/644864.Doc
hez.vwbnt.cn/735415.Doc
hez.vwbnt.cn/266660.Doc
hez.vwbnt.cn/313719.Doc
hez.vwbnt.cn/284397.Doc
hez.vwbnt.cn/957111.Doc
hez.vwbnt.cn/620828.Doc
hez.vwbnt.cn/242222.Doc
hez.vwbnt.cn/119599.Doc
hez.vwbnt.cn/400284.Doc
hel.vwbnt.cn/202002.Doc
hel.vwbnt.cn/260080.Doc
hel.vwbnt.cn/226448.Doc
hel.vwbnt.cn/137559.Doc
hel.vwbnt.cn/002600.Doc
hel.vwbnt.cn/468684.Doc
hel.vwbnt.cn/644464.Doc
hel.vwbnt.cn/544494.Doc
hel.vwbnt.cn/004020.Doc
hel.vwbnt.cn/600260.Doc
hek.vwbnt.cn/848040.Doc
hek.vwbnt.cn/763429.Doc
hek.vwbnt.cn/995955.Doc
hek.vwbnt.cn/848046.Doc
hek.vwbnt.cn/424640.Doc
hek.vwbnt.cn/603444.Doc
hek.vwbnt.cn/452270.Doc
hek.vwbnt.cn/408800.Doc
hek.vwbnt.cn/882664.Doc
hek.vwbnt.cn/822686.Doc
hej.vwbnt.cn/028640.Doc
hej.vwbnt.cn/444800.Doc
hej.vwbnt.cn/026208.Doc
hej.vwbnt.cn/977537.Doc
hej.vwbnt.cn/086200.Doc
hej.vwbnt.cn/066606.Doc
hej.vwbnt.cn/862486.Doc
hej.vwbnt.cn/953197.Doc
hej.vwbnt.cn/484880.Doc
hej.vwbnt.cn/624806.Doc
heh.vwbnt.cn/739147.Doc
heh.vwbnt.cn/575797.Doc
heh.vwbnt.cn/464448.Doc
heh.vwbnt.cn/040660.Doc
heh.vwbnt.cn/062220.Doc
heh.vwbnt.cn/470024.Doc
heh.vwbnt.cn/026606.Doc
heh.vwbnt.cn/974504.Doc
heh.vwbnt.cn/646206.Doc
heh.vwbnt.cn/482202.Doc
heg.vwbnt.cn/620806.Doc
heg.vwbnt.cn/884848.Doc
heg.vwbnt.cn/826604.Doc
heg.vwbnt.cn/004006.Doc
heg.vwbnt.cn/668442.Doc
heg.vwbnt.cn/268228.Doc
heg.vwbnt.cn/265487.Doc
heg.vwbnt.cn/086026.Doc
heg.vwbnt.cn/498681.Doc
heg.vwbnt.cn/642048.Doc
hef.vwbnt.cn/042888.Doc
hef.vwbnt.cn/662202.Doc
hef.vwbnt.cn/600880.Doc
hef.vwbnt.cn/220040.Doc
hef.vwbnt.cn/042840.Doc
hef.vwbnt.cn/608620.Doc
hef.vwbnt.cn/628468.Doc
hef.vwbnt.cn/682486.Doc
hef.vwbnt.cn/228022.Doc
hef.vwbnt.cn/802820.Doc
hed.vwbnt.cn/202440.Doc
hed.vwbnt.cn/082600.Doc
hed.vwbnt.cn/777159.Doc
hed.vwbnt.cn/024240.Doc
hed.vwbnt.cn/402620.Doc
hed.vwbnt.cn/220628.Doc
hed.vwbnt.cn/462842.Doc
hed.vwbnt.cn/622026.Doc
hed.vwbnt.cn/686262.Doc
hed.vwbnt.cn/884668.Doc
hes.vwbnt.cn/882200.Doc
hes.vwbnt.cn/640240.Doc
hes.vwbnt.cn/408266.Doc
hes.vwbnt.cn/064442.Doc
hes.vwbnt.cn/530793.Doc
hes.vwbnt.cn/684000.Doc
hes.vwbnt.cn/086226.Doc
hes.vwbnt.cn/246400.Doc
hes.vwbnt.cn/460040.Doc
hes.vwbnt.cn/280028.Doc
hea.vwbnt.cn/171511.Doc
hea.vwbnt.cn/573135.Doc
hea.vwbnt.cn/719995.Doc
hea.vwbnt.cn/082424.Doc
hea.vwbnt.cn/222646.Doc
hea.vwbnt.cn/820200.Doc
hea.vwbnt.cn/048244.Doc
hea.vwbnt.cn/644428.Doc
hea.vwbnt.cn/939515.Doc
hea.vwbnt.cn/426244.Doc
hep.vwbnt.cn/860606.Doc
hep.vwbnt.cn/002262.Doc
hep.vwbnt.cn/840068.Doc
hep.vwbnt.cn/844086.Doc
hep.vwbnt.cn/444402.Doc
hep.vwbnt.cn/660886.Doc
hep.vwbnt.cn/751517.Doc
hep.vwbnt.cn/626240.Doc
hep.vwbnt.cn/684666.Doc
hep.vwbnt.cn/959155.Doc
heo.vwbnt.cn/825637.Doc
heo.vwbnt.cn/375933.Doc
heo.vwbnt.cn/735575.Doc
heo.vwbnt.cn/244842.Doc
heo.vwbnt.cn/006642.Doc
heo.vwbnt.cn/486088.Doc
heo.vwbnt.cn/244822.Doc
heo.vwbnt.cn/164474.Doc
heo.vwbnt.cn/177779.Doc
heo.vwbnt.cn/444808.Doc
hei.vwbnt.cn/082600.Doc
hei.vwbnt.cn/424600.Doc
hei.vwbnt.cn/600028.Doc
hei.vwbnt.cn/648602.Doc
hei.vwbnt.cn/246064.Doc
hei.vwbnt.cn/648428.Doc
hei.vwbnt.cn/806062.Doc
hei.vwbnt.cn/680268.Doc
hei.vwbnt.cn/359333.Doc
hei.vwbnt.cn/882404.Doc
heu.vwbnt.cn/422048.Doc
heu.vwbnt.cn/266044.Doc
heu.vwbnt.cn/420806.Doc
heu.vwbnt.cn/822242.Doc
heu.vwbnt.cn/022000.Doc
heu.vwbnt.cn/088602.Doc
heu.vwbnt.cn/408864.Doc
heu.vwbnt.cn/204282.Doc
heu.vwbnt.cn/731997.Doc
heu.vwbnt.cn/004626.Doc
hey.vwbnt.cn/600008.Doc
hey.vwbnt.cn/640206.Doc
hey.vwbnt.cn/884468.Doc
hey.vwbnt.cn/120737.Doc
hey.vwbnt.cn/226444.Doc
hey.vwbnt.cn/800668.Doc
hey.vwbnt.cn/266466.Doc
hey.vwbnt.cn/406640.Doc
hey.vwbnt.cn/008246.Doc
hey.vwbnt.cn/022006.Doc
het.vwbnt.cn/602628.Doc
het.vwbnt.cn/682208.Doc
het.vwbnt.cn/979133.Doc
het.vwbnt.cn/060402.Doc
het.vwbnt.cn/860046.Doc
het.vwbnt.cn/796548.Doc
het.vwbnt.cn/228824.Doc
het.vwbnt.cn/606040.Doc
het.vwbnt.cn/485980.Doc
het.vwbnt.cn/682420.Doc
her.vwbnt.cn/868046.Doc
her.vwbnt.cn/995151.Doc
her.vwbnt.cn/573971.Doc
her.vwbnt.cn/424808.Doc
her.vwbnt.cn/060488.Doc
her.vwbnt.cn/219826.Doc
her.vwbnt.cn/664286.Doc
her.vwbnt.cn/588655.Doc
her.vwbnt.cn/224860.Doc
her.vwbnt.cn/507859.Doc
hee.vwbnt.cn/824844.Doc
hee.vwbnt.cn/200648.Doc
hee.vwbnt.cn/602462.Doc
hee.vwbnt.cn/260040.Doc
hee.vwbnt.cn/264064.Doc
hee.vwbnt.cn/882846.Doc
hee.vwbnt.cn/992514.Doc
hee.vwbnt.cn/664226.Doc
hee.vwbnt.cn/822266.Doc
hee.vwbnt.cn/826064.Doc
hew.vwbnt.cn/240880.Doc
hew.vwbnt.cn/874837.Doc
hew.vwbnt.cn/086844.Doc
hew.vwbnt.cn/002688.Doc
hew.vwbnt.cn/620262.Doc
hew.vwbnt.cn/654975.Doc
hew.vwbnt.cn/444420.Doc
hew.vwbnt.cn/888684.Doc
hew.vwbnt.cn/515951.Doc
hew.vwbnt.cn/244860.Doc
heq.vwbnt.cn/224228.Doc
heq.vwbnt.cn/994650.Doc
heq.vwbnt.cn/804026.Doc
heq.vwbnt.cn/744281.Doc
heq.vwbnt.cn/808804.Doc
heq.vwbnt.cn/244620.Doc
heq.vwbnt.cn/404222.Doc
heq.vwbnt.cn/668804.Doc
heq.vwbnt.cn/644004.Doc
heq.vwbnt.cn/040446.Doc
hwm.vwbnt.cn/080462.Doc
hwm.vwbnt.cn/064802.Doc
hwm.vwbnt.cn/911197.Doc
hwm.vwbnt.cn/868620.Doc
hwm.vwbnt.cn/658282.Doc
hwm.vwbnt.cn/404264.Doc
hwm.vwbnt.cn/086620.Doc
hwm.vwbnt.cn/000222.Doc
hwm.vwbnt.cn/842620.Doc
