厝奶迂磐召


Linux cifs_readv直接读取与SMB2_read_data_retry逻辑

CIFS/SMB3客户端的读路径涉及从页面缓存命中到远端服务器读取的完整流程。cifs_readv是direct IO和缓冲IO的底层读取入口，SMB2_read_data_retry则是对SMB2 READ请求的封装层，提供自动重试和错误恢复能力。

一、cifs_readv的入口与分发

cifs_readv是CIFS文件系统address_space的readpage和direct_IO的后端实现，接收struct iov_iter描述的用户态或内核态缓冲区：

static ssize_t
cifs_readv(struct kiocb *iocb, struct iov_iter *to)
{
    struct inode *inode = file_inode(iocb->ki_filp);
    struct cifsInodeInfo *cinode = CIFS_I(inode);
    struct cifs_sb_info *cifs_sb = CIFS_SB(inode->i_sb);
    struct cifsFileInfo *cfile = iocb->ki_filp->private_data;
    struct TCP_Server_Info *server;
    ssize_t rc = -EACCES;
    loff_t offset = iocb->ki_pos;
    size_t len = iov_iter_count(to);
    unsigned int xid;

    xid = get_xid();

    if (!cifs_sb->mnt_cifs_flags & CIFS_MOUNT_RWPIDREG)
        server = cfile->server;
    else
        server = cifs_pick_channel(cfile->server);

    if (cfile->invalidHandle) {
        rc = cifs_reopen_file(cfile, false);
        if (rc != 0)
            goto out;
    }

    if (len > 0) {
        rc = server->ops->direct_readv(xid, cfile, iocb, to);
        if (rc > 0) {
            iocb->ki_pos += rc;
            iov_iter_advance(to, rc);
        }
    }

out:
    free_xid(xid);
    return rc;
}

函数首先检查文件句柄的有效性，若句柄因服务器重启或断连而失效，调用cifs_reopen_file重新建立。之后调用server->ops->direct_readv函数指针，在SMB3协议中该指针指向smb3_direct_readv。

二、smb3_direct_readv到SMB2_read的调用链

smb3_direct_readv分配并初始化READ请求，构造struct smb_rqst发送给服务器：

ssize_t
smb3_direct_readv(struct cifsFileInfo *cfile, struct io_context *ctx,
          struct iov_iter *iter, loff_t *pos)
{
    struct TCP_Server_Info *server = cfile->server;
    ssize_t rdata_size = sizeof(struct cifs_readdata);
    struct cifs_readdata *rdata;
    unsigned int xid = get_xid();
    int rc;

    rdata = kzalloc(rdata_size, GFP_KERNEL);
    if (!rdata) {
        free_xid(xid);
        return -ENOMEM;
    }

    rdata->cfile = cfile;
    rdata->server = server;
    rdata->mapping = cfile->dentry->d_inode->i_mapping;
    rdata->offset = *pos;
    rdata->bytes = iov_iter_count(iter);
    rdata->iov_iter = *iter;

    rc = SMB2_read_data_retry(xid, rdata);
    if (rc < 0) {
        kfree(rdata);
        free_xid(xid);
        return rc;
    }

    *pos += rc;
    iov_iter_advance(iter, rc);
    kfree(rdata);
    free_xid(xid);
    return rc;
}

cifs_readdata结构体保存读取请求的所有上下文信息，包括文件句柄、偏移、长度以及目标缓存迭代器。

三、SMB2_read_data_retry的重试逻辑

SMB2_read_data_retry是读请求的核心发送函数，封装了自动重试机制以应对网络瞬态错误和服务器故障：

int
SMB2_read_data_retry(unsigned int xid, struct cifs_readdata *rdata)
{
    struct cifsFileInfo *cfile = rdata->cfile;
    struct TCP_Server_Info *server = cfile->server;
    int rc;
    int retries = 0;

    for (retries = 0; retries < 10; retries++) {
        if (cfile->invalidHandle) {
            rc = cifs_reopen_file(cfile, false);
            if (rc) {
                if (retries > 1)
                    goto fail;
                msleep(1);
                continue;
            }
        }

        rc = SMB2_read(xid, rdata);
        if (rc == 0 || rc == -ENODATA)
            break;

        if (rc == -EAGAIN) {
            msleep(1);
            continue;
        }

        if (rc == -EINTR || rc == -ERESTARTSYS)
            break;

        if (rc == -EACCES)
            break;

        if (rc == -ECONNABORTED || rc == -ENETDOWN) {
            cifs_server_dbg(FYI, "server torn down, retry\n");
            msleep(50);
            continue;
        }

        if (rc == -EIO) {
            cifs_server_dbg(FYI, "I/O error, retry\n");
            msleep(100);
            continue;
        }

        break;
    }

    if (rc == -EAGAIN)
        rc = 0;

fail:
    return rc;
}

重试逻辑处理以下几类错误：
1. cfile->invalidHandle: 句柄失效时尝试重新打开文件，最多重试10次。
2. -EAGAIN: 服务器资源暂时不可用，短暂等待后重试。
3. -EINTR/-ERESTARTSYS: 被信号中断，直接返回给上层。
4. -EACCES: 访问被拒绝，继续重试无意义，立即返回。
5. -ECONNABORTED/-ENETDOWN: 网络层断开，等待50ms后重试。
6. -EIO: IO错误，等待100ms后重试。

重试次数上限为10次，覆盖大多数瞬态故障场景。

四、SMB2_read的协议构造

SMB2_read构建SMB2 READ请求并处理响应：

int
SMB2_read(unsigned int xid, struct cifs_readdata *rdata)
{
    struct smb2_read_req *req;
    struct smb2_read_rsp *rsp;
    struct TCP_Server_Info *server;
    struct cifsFileInfo *cfile = rdata->cfile;
    struct kvec iov[2];
    struct smb_rqst rqst;
    int rc, resp_buf_type;

    server = cfile->server;
    rc = smb2_plain_req_init(SMB2_READ, server, (void **)&req, &resp_buf_type);
    if (rc)
        return rc;

    req->hdr.SessionId = server->session_id;
    req->hdr.ProcessId = cpu_to_le32(server->client_id);
    req->StructureSize = cpu_to_le16(49);
    req->Padding = 0;
    req->Flags = 0;
    req->Length = cpu_to_le32(rdata->bytes);
    req->Offset = cpu_to_le64(rdata->offset);
    req->PersistentFileId = cfile->fid.persistent_fid;
    req->VolatileFileId = cfile->fid.volatile_fid;

    iov[0].iov_base = (char *)req;
    iov[0].iov_len = sizeof(struct smb2_read_req);
    iov[1].iov_base = NULL;
    iov[1].iov_len = 0;

    memset(&rqst, 0, sizeof(struct smb_rqst));
    rqst.rq_iov = iov;
    rqst.rq_nvec = 2;
    rqst.rq_rsp_iov = NULL;
    rqst.rq_nr_vec = 0;

    rc = cifs_send_recv(xid, &rqst, &resp_buf_type, server, cfile->fid.netfid);
    if (rc)
        goto out;

    rsp = (struct smb2_read_rsp *)iov[0].iov_base;
    rc = le32_to_cpu(rsp->hdr.Status);
    if (rc)
        goto out;

    rc = le32_to_cpu(rsp->DataLength);
    if (rc > rdata->bytes) {
        cifs_dbg(VFS, "server returned more data than requested\n");
        rc = -EIO;
        goto out;
    }

    if (rc > 0) {
        memcpy_from_iovec(rdata->iov_iter, rsp->Buffer, rc);
        rdata->got_bytes = rc;
    }

out:
    free_rsp_buf(resp_buf_type, rsp);
    return rc;
}

READ请求包含持久化和易失性文件ID(FID)，分别标识服务器端打开的文件对象和会话绑定对象。响应中的DataLength字段指示实际读取的数据量，可能小于请求长度(EOF场景)。

五、传输层错误与签名验证

cifs_send_recv内部处理传输层消息发送和接收，包括签名验证。若响应签名验证失败，cifs_send_recv丢弃数据包并重发请求。签名验证使用之前会话建立阶段协商的密钥，通过cifs_calc_signature2计算预期签名并与响应中的签名字段比对。

六、DIO与缓冲IO的路径差异

cifs_readv的direct IO路径绕过页面缓存直接填充用户缓冲区。缓冲IO路径则通过cifs_readpage->cifs_readpages批量读取，将数据填入页面缓存后再拷贝到用户空间。两种路径最终都汇聚到SMB2_read_data_retry，区别在于cifs_readdata的iov_iter指向不同目标。

拍衔晕谌耗敦盏饲傩字云特渴圃盼

ecy.sthxr.cn/117713.htm
ecy.sthxr.cn/511313.htm
ecy.sthxr.cn/882283.htm
ecy.sthxr.cn/466063.htm
ecy.sthxr.cn/591753.htm
ect.sthxr.cn/648403.htm
ect.sthxr.cn/919173.htm
ect.sthxr.cn/608863.htm
ect.sthxr.cn/682883.htm
ect.sthxr.cn/484403.htm
ect.sthxr.cn/060083.htm
ect.sthxr.cn/288023.htm
ect.sthxr.cn/628283.htm
ect.sthxr.cn/993713.htm
ect.sthxr.cn/993193.htm
ecr.sthxr.cn/157333.htm
ecr.sthxr.cn/240803.htm
ecr.sthxr.cn/046863.htm
ecr.sthxr.cn/795713.htm
ecr.sthxr.cn/624003.htm
ecr.sthxr.cn/533953.htm
ecr.sthxr.cn/660803.htm
ecr.sthxr.cn/604483.htm
ecr.sthxr.cn/022243.htm
ecr.sthxr.cn/955993.htm
ece.sthxr.cn/284063.htm
ece.sthxr.cn/626643.htm
ece.sthxr.cn/446003.htm
ece.sthxr.cn/822483.htm
ece.sthxr.cn/082843.htm
ece.sthxr.cn/628023.htm
ece.sthxr.cn/193753.htm
ece.sthxr.cn/915913.htm
ece.sthxr.cn/248883.htm
ece.sthxr.cn/979973.htm
ecw.sthxr.cn/193173.htm
ecw.sthxr.cn/446263.htm
ecw.sthxr.cn/288403.htm
ecw.sthxr.cn/008463.htm
ecw.sthxr.cn/426663.htm
ecw.sthxr.cn/951793.htm
ecw.sthxr.cn/373373.htm
ecw.sthxr.cn/680063.htm
ecw.sthxr.cn/771773.htm
ecw.sthxr.cn/668863.htm
ecq.sthxr.cn/313573.htm
ecq.sthxr.cn/113553.htm
ecq.sthxr.cn/242423.htm
ecq.sthxr.cn/395593.htm
ecq.sthxr.cn/640863.htm
ecq.sthxr.cn/991733.htm
ecq.sthxr.cn/953313.htm
ecq.sthxr.cn/197353.htm
ecq.sthxr.cn/264643.htm
ecq.sthxr.cn/513753.htm
exm.sthxr.cn/535113.htm
exm.sthxr.cn/024443.htm
exm.sthxr.cn/064403.htm
exm.sthxr.cn/311993.htm
exm.sthxr.cn/573953.htm
exm.sthxr.cn/260803.htm
exm.sthxr.cn/684083.htm
exm.sthxr.cn/882823.htm
exm.sthxr.cn/175733.htm
exm.sthxr.cn/640643.htm
exn.sthxr.cn/379513.htm
exn.sthxr.cn/915173.htm
exn.sthxr.cn/579573.htm
exn.sthxr.cn/824283.htm
exn.sthxr.cn/244003.htm
exn.sthxr.cn/959933.htm
exn.sthxr.cn/606603.htm
exn.sthxr.cn/266223.htm
exn.sthxr.cn/133173.htm
exn.sthxr.cn/404263.htm
exb.sthxr.cn/739773.htm
exb.sthxr.cn/268003.htm
exb.sthxr.cn/042283.htm
exb.sthxr.cn/840883.htm
exb.sthxr.cn/379993.htm
exb.sthxr.cn/420623.htm
exb.sthxr.cn/868823.htm
exb.sthxr.cn/551793.htm
exb.sthxr.cn/155533.htm
exb.sthxr.cn/993333.htm
exv.sthxr.cn/426223.htm
exv.sthxr.cn/280663.htm
exv.sthxr.cn/466243.htm
exv.sthxr.cn/206683.htm
exv.sthxr.cn/977193.htm
exv.sthxr.cn/608283.htm
exv.sthxr.cn/753553.htm
exv.sthxr.cn/428843.htm
exv.sthxr.cn/791173.htm
exv.sthxr.cn/420863.htm
exc.sthxr.cn/206403.htm
exc.sthxr.cn/422443.htm
exc.sthxr.cn/066023.htm
exc.sthxr.cn/755913.htm
exc.sthxr.cn/468223.htm
exc.sthxr.cn/428003.htm
exc.sthxr.cn/442883.htm
exc.sthxr.cn/422083.htm
exc.sthxr.cn/759153.htm
exc.sthxr.cn/446663.htm
exx.sthxr.cn/628403.htm
exx.sthxr.cn/511393.htm
exx.sthxr.cn/711953.htm
exx.sthxr.cn/262423.htm
exx.sthxr.cn/460803.htm
exx.sthxr.cn/355153.htm
exx.sthxr.cn/808683.htm
exx.sthxr.cn/284863.htm
exx.sthxr.cn/862403.htm
exx.sthxr.cn/135793.htm
exz.sthxr.cn/517133.htm
exz.sthxr.cn/284443.htm
exz.sthxr.cn/688803.htm
exz.sthxr.cn/955373.htm
exz.sthxr.cn/682443.htm
exz.sthxr.cn/804443.htm
exz.sthxr.cn/284403.htm
exz.sthxr.cn/266283.htm
exz.sthxr.cn/006243.htm
exz.sthxr.cn/173513.htm
exl.sthxr.cn/715513.htm
exl.sthxr.cn/535373.htm
exl.sthxr.cn/195753.htm
exl.sthxr.cn/115913.htm
exl.sthxr.cn/993593.htm
exl.sthxr.cn/048203.htm
exl.sthxr.cn/646463.htm
exl.sthxr.cn/135193.htm
exl.sthxr.cn/393753.htm
exl.sthxr.cn/400803.htm
exk.sthxr.cn/555533.htm
exk.sthxr.cn/999113.htm
exk.sthxr.cn/911573.htm
exk.sthxr.cn/426263.htm
exk.sthxr.cn/155113.htm
exk.sthxr.cn/391733.htm
exk.sthxr.cn/713953.htm
exk.sthxr.cn/284083.htm
exk.sthxr.cn/195993.htm
exk.sthxr.cn/606863.htm
exj.sthxr.cn/288023.htm
exj.sthxr.cn/840643.htm
exj.sthxr.cn/040243.htm
exj.sthxr.cn/226883.htm
exj.sthxr.cn/468023.htm
exj.sthxr.cn/426263.htm
exj.sthxr.cn/737953.htm
exj.sthxr.cn/640423.htm
exj.sthxr.cn/604043.htm
exj.sthxr.cn/719193.htm
exh.sthxr.cn/200023.htm
exh.sthxr.cn/406643.htm
exh.sthxr.cn/880283.htm
exh.sthxr.cn/626243.htm
exh.sthxr.cn/624023.htm
exh.sthxr.cn/640463.htm
exh.sthxr.cn/248623.htm
exh.sthxr.cn/735573.htm
exh.sthxr.cn/860403.htm
exh.sthxr.cn/979793.htm
exg.sthxr.cn/193353.htm
exg.sthxr.cn/959313.htm
exg.sthxr.cn/515933.htm
exg.sthxr.cn/040083.htm
exg.sthxr.cn/080663.htm
exg.sthxr.cn/759713.htm
exg.sthxr.cn/577913.htm
exg.sthxr.cn/335533.htm
exg.sthxr.cn/420883.htm
exg.sthxr.cn/284083.htm
exf.sthxr.cn/335733.htm
exf.sthxr.cn/606423.htm
exf.sthxr.cn/428883.htm
exf.sthxr.cn/686243.htm
exf.sthxr.cn/644483.htm
exf.sthxr.cn/046083.htm
exf.sthxr.cn/040623.htm
exf.sthxr.cn/466423.htm
exf.sthxr.cn/559513.htm
exf.sthxr.cn/719113.htm
exd.sthxr.cn/159333.htm
exd.sthxr.cn/206223.htm
exd.sthxr.cn/042483.htm
exd.sthxr.cn/242263.htm
exd.sthxr.cn/826623.htm
exd.sthxr.cn/020263.htm
exd.sthxr.cn/424083.htm
exd.sthxr.cn/646443.htm
exd.sthxr.cn/602403.htm
exd.sthxr.cn/066083.htm
exs.sthxr.cn/244023.htm
exs.sthxr.cn/517973.htm
exs.sthxr.cn/486063.htm
exs.sthxr.cn/468223.htm
exs.sthxr.cn/977373.htm
exs.sthxr.cn/024083.htm
exs.sthxr.cn/917513.htm
exs.sthxr.cn/139173.htm
exs.sthxr.cn/137313.htm
exs.sthxr.cn/333993.htm
exa.sthxr.cn/975133.htm
exa.sthxr.cn/591793.htm
exa.sthxr.cn/311373.htm
exa.sthxr.cn/539393.htm
exa.sthxr.cn/191373.htm
exa.sthxr.cn/773733.htm
exa.sthxr.cn/115333.htm
exa.sthxr.cn/793193.htm
exa.sthxr.cn/979353.htm
exa.sthxr.cn/155133.htm
exp.sthxr.cn/957313.htm
exp.sthxr.cn/355793.htm
exp.sthxr.cn/880223.htm
exp.sthxr.cn/317773.htm
exp.sthxr.cn/137933.htm
exp.sthxr.cn/379733.htm
exp.sthxr.cn/711593.htm
exp.sthxr.cn/597373.htm
exp.sthxr.cn/153773.htm
exp.sthxr.cn/082663.htm
exo.sthxr.cn/915333.htm
exo.sthxr.cn/193393.htm
exo.sthxr.cn/151333.htm
exo.sthxr.cn/173713.htm
exo.sthxr.cn/844843.htm
exo.sthxr.cn/020463.htm
exo.sthxr.cn/357753.htm
exo.sthxr.cn/911553.htm
exo.sthxr.cn/713573.htm
exo.sthxr.cn/060483.htm
exi.sthxr.cn/133793.htm
exi.sthxr.cn/313153.htm
exi.sthxr.cn/171913.htm
exi.sthxr.cn/082443.htm
exi.sthxr.cn/957173.htm
exi.sthxr.cn/177913.htm
exi.sthxr.cn/931553.htm
exi.sthxr.cn/393733.htm
exi.sthxr.cn/775753.htm
exi.sthxr.cn/939173.htm
exu.sthxr.cn/357553.htm
exu.sthxr.cn/202423.htm
exu.sthxr.cn/311753.htm
exu.sthxr.cn/464863.htm
exu.sthxr.cn/828663.htm
exu.sthxr.cn/779593.htm
exu.sthxr.cn/399953.htm
exu.sthxr.cn/331553.htm
exu.sthxr.cn/559713.htm
exu.sthxr.cn/599973.htm
exy.sthxr.cn/155973.htm
exy.sthxr.cn/355373.htm
exy.sthxr.cn/959533.htm
exy.sthxr.cn/337773.htm
exy.sthxr.cn/159713.htm
exy.sthxr.cn/513933.htm
exy.sthxr.cn/137313.htm
exy.sthxr.cn/511773.htm
exy.sthxr.cn/575993.htm
exy.sthxr.cn/242423.htm
ext.sthxr.cn/280603.htm
ext.sthxr.cn/319113.htm
ext.sthxr.cn/133753.htm
ext.sthxr.cn/806023.htm
ext.sthxr.cn/555793.htm
ext.sthxr.cn/955753.htm
ext.sthxr.cn/573393.htm
ext.sthxr.cn/535933.htm
ext.sthxr.cn/917713.htm
ext.sthxr.cn/559993.htm
exr.sthxr.cn/151513.htm
exr.sthxr.cn/537113.htm
exr.sthxr.cn/155513.htm
exr.sthxr.cn/551133.htm
exr.sthxr.cn/599173.htm
exr.sthxr.cn/735753.htm
exr.sthxr.cn/777193.htm
exr.sthxr.cn/804603.htm
exr.sthxr.cn/537773.htm
exr.sthxr.cn/795993.htm
exe.sthxr.cn/044603.htm
exe.sthxr.cn/395573.htm
exe.sthxr.cn/915933.htm
exe.sthxr.cn/828063.htm
exe.sthxr.cn/171193.htm
exe.sthxr.cn/599913.htm
exe.sthxr.cn/313753.htm
exe.sthxr.cn/135113.htm
exe.sthxr.cn/000623.htm
exe.sthxr.cn/351793.htm
exw.sthxr.cn/199133.htm
exw.sthxr.cn/802263.htm
exw.sthxr.cn/864803.htm
exw.sthxr.cn/195733.htm
exw.sthxr.cn/800263.htm
exw.sthxr.cn/733913.htm
exw.sthxr.cn/171973.htm
exw.sthxr.cn/884463.htm
exw.sthxr.cn/139953.htm
exw.sthxr.cn/557713.htm
exq.sthxr.cn/884843.htm
exq.sthxr.cn/379533.htm
exq.sthxr.cn/137953.htm
exq.sthxr.cn/175993.htm
exq.sthxr.cn/513553.htm
exq.sthxr.cn/557553.htm
exq.sthxr.cn/666443.htm
exq.sthxr.cn/355173.htm
exq.sthxr.cn/555953.htm
exq.sthxr.cn/808403.htm
ezm.sthxr.cn/644463.htm
ezm.sthxr.cn/579153.htm
ezm.sthxr.cn/462263.htm
ezm.sthxr.cn/715553.htm
ezm.sthxr.cn/351753.htm
ezm.sthxr.cn/608663.htm
ezm.sthxr.cn/719993.htm
ezm.sthxr.cn/262203.htm
ezm.sthxr.cn/844243.htm
ezm.sthxr.cn/842843.htm
ezn.sthxr.cn/751793.htm
ezn.sthxr.cn/082063.htm
ezn.sthxr.cn/317133.htm
ezn.sthxr.cn/331953.htm
ezn.sthxr.cn/808083.htm
ezn.sthxr.cn/539173.htm
ezn.sthxr.cn/535933.htm
ezn.sthxr.cn/288423.htm
ezn.sthxr.cn/579333.htm
ezn.sthxr.cn/753593.htm
ezb.sthxr.cn/557353.htm
ezb.sthxr.cn/331533.htm
ezb.sthxr.cn/793353.htm
ezb.sthxr.cn/933753.htm
ezb.sthxr.cn/000823.htm
ezb.sthxr.cn/775773.htm
ezb.sthxr.cn/828423.htm
ezb.sthxr.cn/931393.htm
ezb.sthxr.cn/377993.htm
ezb.sthxr.cn/806223.htm
ezv.sthxr.cn/939953.htm
ezv.sthxr.cn/797733.htm
ezv.sthxr.cn/208683.htm
ezv.sthxr.cn/933313.htm
ezv.sthxr.cn/735913.htm
ezv.sthxr.cn/573313.htm
ezv.sthxr.cn/448283.htm
ezv.sthxr.cn/844263.htm
ezv.sthxr.cn/646603.htm
ezv.sthxr.cn/971573.htm
ezc.sthxr.cn/373973.htm
ezc.sthxr.cn/684063.htm
ezc.sthxr.cn/793933.htm
ezc.sthxr.cn/571333.htm
ezc.sthxr.cn/339313.htm
ezc.sthxr.cn/177933.htm
ezc.sthxr.cn/153333.htm
ezc.sthxr.cn/733773.htm
ezc.sthxr.cn/991193.htm
ezc.sthxr.cn/395553.htm
ezx.sthxr.cn/197553.htm
ezx.sthxr.cn/791753.htm
ezx.sthxr.cn/975553.htm
ezx.sthxr.cn/040043.htm
ezx.sthxr.cn/359133.htm
ezx.sthxr.cn/971153.htm
ezx.sthxr.cn/822223.htm
ezx.sthxr.cn/446423.htm
ezx.sthxr.cn/777393.htm
ezx.sthxr.cn/020263.htm
ezz.sthxr.cn/553113.htm
ezz.sthxr.cn/199733.htm
ezz.sthxr.cn/537773.htm
ezz.sthxr.cn/317733.htm
ezz.sthxr.cn/975313.htm
ezz.sthxr.cn/757153.htm
ezz.sthxr.cn/311153.htm
ezz.sthxr.cn/262603.htm
ezz.sthxr.cn/775793.htm
ezz.sthxr.cn/684443.htm
ezl.sthxr.cn/802863.htm
ezl.sthxr.cn/519513.htm
ezl.sthxr.cn/957733.htm
ezl.sthxr.cn/355773.htm
ezl.sthxr.cn/571993.htm
ezl.sthxr.cn/913113.htm
ezl.sthxr.cn/082003.htm
ezl.sthxr.cn/400443.htm
ezl.sthxr.cn/559193.htm
ezl.sthxr.cn/159533.htm
