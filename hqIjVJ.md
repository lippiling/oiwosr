藤犯毕滥穆


Linux nfsd nfsv4操作分派nfsd4_proc_compound处理

NFSv4协议的核心设计之一是使用COMPOUND RPC来批处理多个操作，减少客户端与服务端之间的网络往返。nfsd4_proc_compound是NFSD中处理NFSv4 COMPOUND请求的入口函数，负责解析复合请求、分派每个子操作到对应的处理函数，并组装回复。

一、nfsd4_proc_compound的调用链

当客户端发送NFSv4请求时，内核网络栈经svc_recv接收数据，经nfsd_dispatch分派到nfsd4_proc_compound。nfsd_dispatch根据请求的RPC程序号与版本号选择对应的分发向量表：

static struct svc_version nfsd_version4 = {
    .vs_nproc   = 2,
    .vs_dispatch = nfsd_dispatch,
    .vs_xdrsize = NFS4_SVC_XDRSIZE,
};

在nfsd_dispatch中识别OP_PROCNUM为NFSPROC4_COMPOUND后，调用nfsd4_proc_compound。

二、复合请求的解析流程

nfsd4_proc_compound首先从网络缓冲中解码COMPOUND头部，包括标记tag、次要版本号、操作数，然后进入主循环逐个解码并执行操作。关键数据结构为struct nfsd4_compound_state，它贯穿整个请求的生命周期，保存当前状态、当前文件句柄、保存的文件句柄、凭证等信息。

__be32
nfsd4_proc_compound(struct svc_rqst *rqstp)
{
    struct nfsd4_compoundargs *args = rqstp->rq_argp;
    struct nfsd4_compoundres *resp = rqstp->rq_resp;
    struct nfsd4_compound_state *cstate = &resp->cstate;
    struct svc_fh *current_fh = &cstate->current_fh;
    struct svc_fh *save_fh = &cstate->save_fh;
    int opcnt = 0;
    int status;
    int i;

    resp->xbuf = &rqstp->rq_res;
    resp->status = nfs_ok;

    status = nfsd4_decode_compound(args, rqstp);
    if (status) {
        nfsd4_encode_operation(resp, NULL, status);
        goto out;
    }

    for (i = 0; i < args->opcnt; i++) {
        struct nfsd4_op *op = &args->ops[i];
        bool require_open_state = false;
        bool require_delegation = false;

        op->status = nfsd4_op_funcs[op->opnum](rqstp, cstate, &op->u);
        if (op->status == nfserr_op_not_supported) {
            op->status = nfserr_notsupp;
        }

        nfsd4_encode_operation(resp, op);
        if (op->status && nfsd4_has_session(cstate))
            break;
    }

    nfsd4_encode_compound_status(resp);
out:
    nfsd4_release_compoundargs(args);
    return resp->status;
}

核心循环遍历解码后的操作数组，通过nfsd4_op_funcs分发表直接索引调用。该分发表定义在nfsd4proc.c中：

static const struct nfsd4_operation nfsd4_op_funcs[OP_RELEASE_LOCKOWNER + 1] = {
    [OP_ACCESS] = {
        .op_func = nfsd4_access,
        .op_flags = OP_MODIFIES_SOMETHING,
    },
    [OP_CLOSE] = {
        .op_func = nfsd4_close,
        .op_flags = OP_MODIFIES_SOMETHING | OP_CLEAR_STATE,
    },
    [OP_COMMIT] = {
        .op_func = nfsd4_commit,
    },
    [OP_CREATE] = {
        .op_func = nfsd4_create,
        .op_flags = OP_MODIFIES_SOMETHING,
    },
    [OP_DELEGPURGE] = {
        .op_func = nfsd4_delegpurge,
    },
    [OP_DELEGRETURN] = {
        .op_func = nfsd4_delegreturn,
        .op_flags = OP_MODIFIES_SOMETHING,
    },
    [OP_GETATTR] = {
        .op_func = nfsd4_getattr,
    },
    [OP_GETFH] = {
        .op_func = nfsd4_getfh,
    },
    [OP_LINK] = {
        .op_func = nfsd4_link,
        .op_flags = OP_MODIFIES_SOMETHING,
    },
    [OP_LOCK] = {
        .op_func = nfsd4_lock,
        .op_flags = OP_MODIFIES_SOMETHING,
    },
    [OP_LOOKUP] = {
        .op_func = nfsd4_lookup,
    },
    [OP_OPEN] = {
        .op_func = nfsd4_open,
        .op_flags = OP_MODIFIES_SOMETHING | OP_CLEAR_STATE,
    },
    [OP_PUTFH] = {
        .op_func = nfsd4_putfh,
    },
    [OP_READ] = {
        .op_func = nfsd4_read,
    },
    [OP_READDIR] = {
        .op_func = nfsd4_readdir,
    },
    [OP_REMOVE] = {
        .op_func = nfsd4_remove,
        .op_flags = OP_MODIFIES_SOMETHING,
    },
    [OP_RENAME] = {
        .op_func = nfsd4_rename,
        .op_flags = OP_MODIFIES_SOMETHING,
    },
    [OP_RESTOREFH] = {
        .op_func = nfsd4_restorefh,
    },
    [OP_SAVEFH] = {
        .op_func = nfsd4_savefh,
    },
    [OP_SECINFO] = {
        .op_func = nfsd4_secinfo,
    },
    [OP_SETATTR] = {
        .op_func = nfsd4_setattr,
        .op_flags = OP_MODIFIES_SOMETHING,
    },
    [OP_SETCLIENTID] = {
        .op_func = nfsd4_setclientid,
        .op_flags = OP_MODIFIES_SOMETHING,
    },
    [OP_SETCLIENTID_CONFIRM] = {
        .op_func = nfsd4_setclientid_confirm,
        .op_flags = OP_MODIFIES_SOMETHING,
    },
    [OP_VERIFY] = {
        .op_func = nfsd4_verify,
    },
    [OP_WRITE] = {
        .op_func = nfsd4_write,
        .op_flags = OP_MODIFIES_SOMETHING,
    },
    [OP_RELEASE_LOCKOWNER] = {
        .op_func = nfsd4_release_lockowner,
    },
};

三、操作分派的优化设计

nfsd4_op_funcs使用数组而非switch-case，原因是NFSv4操作码是连续的整数枚举值(OP_ACCESS = 3, OP_CLOSE = 4, ...)，数组索引访问是O(1)，且现代CPU对数组访问有硬件预取优化。每个操作条目还包含op_flags标志位，指示操作是否修改文件系统状态，用于优化锁的获取和缓存一致性管理。

四、出错处理与短路逻辑

当某个操作返回错误且会话已建立时，COMPOUND循环会break短路，后续操作不再执行。这是NFSv4.1及以上版本的规范行为——错误发生后继续执行无意义，且可能引发副作用。对于未建立会话的情况，所有操作仍继续执行完毕，以适应NFSv4.0中的某些特殊情况。

五、编码阶段

每个操作执行后立即调用nfsd4_encode_operation对结果进行XDR编码。编码过程会将操作码、状态码以及操作特有数据依次写入回复缓冲区。编码完成后调用nfsd4_encode_compound_status写入整个COMPOUND的最终状态和tag。

六、XDR解码/编码的协作

nfsd4_decode_compound负责从原始网络字节流中解析出struct nfsd4_compoundargs，填充ops数组和各个操作参数。解码过程中使用nfsd4_decode_op_map提前验证操作码合法性。若解码失败则直接编码错误回复，不进入主循环。每个操作的解码函数由nfsd4_dec_ops数组索引调用，与执行分发表形成对称设计。

七、文件句柄管理

cstate中的current_fh和save_fh通过PUTFH、SAVEFH、RESTOREFH等操作进行管理。PUTFH将客户端指定的文件句柄设置到current_fh，SAVEFH将current_fh备份到save_fh，RESTOREFH从save_fh恢复。文件句柄的引用计数通过fh_put/fh_get维护，确保在COMPOUND生命周期内底层inode不会提前释放。

八、资源清理

函数退出前调用nfsd4_release_compoundargs释放解码阶段动态分配的内存，包括操作参数中的变长数据结构。若在循环中某操作修改了状态（例如OPEN创建了新的stateid），相关资源由各操作的错误处理路径负责回滚，nfsd4_proc_compound不进行全局回滚，以保持处理逻辑的清晰。

唇交傥参稍翰诼即补灿苍抗士擦障

etd.hjiocz.cn/939993.htm
etd.hjiocz.cn/357533.htm
etd.hjiocz.cn/177373.htm
etd.hjiocz.cn/426403.htm
etd.hjiocz.cn/242003.htm
etd.hjiocz.cn/806203.htm
etd.hjiocz.cn/244203.htm
etd.hjiocz.cn/377913.htm
etd.hjiocz.cn/953193.htm
etd.hjiocz.cn/179753.htm
ets.hjiocz.cn/068863.htm
ets.hjiocz.cn/022223.htm
ets.hjiocz.cn/040843.htm
ets.hjiocz.cn/939773.htm
ets.hjiocz.cn/448423.htm
ets.hjiocz.cn/864823.htm
ets.hjiocz.cn/400683.htm
ets.hjiocz.cn/137173.htm
ets.hjiocz.cn/248443.htm
ets.hjiocz.cn/351173.htm
eta.hjiocz.cn/846283.htm
eta.hjiocz.cn/199973.htm
eta.hjiocz.cn/002463.htm
eta.hjiocz.cn/242823.htm
eta.hjiocz.cn/644663.htm
eta.hjiocz.cn/044243.htm
eta.hjiocz.cn/759733.htm
eta.hjiocz.cn/759913.htm
eta.hjiocz.cn/339753.htm
eta.hjiocz.cn/048083.htm
etp.hjiocz.cn/395753.htm
etp.hjiocz.cn/802423.htm
etp.hjiocz.cn/993733.htm
etp.hjiocz.cn/846603.htm
etp.hjiocz.cn/731553.htm
etp.hjiocz.cn/688203.htm
etp.hjiocz.cn/113993.htm
etp.hjiocz.cn/244283.htm
etp.hjiocz.cn/064263.htm
etp.hjiocz.cn/820803.htm
eto.hjiocz.cn/028603.htm
eto.hjiocz.cn/535933.htm
eto.hjiocz.cn/179353.htm
eto.hjiocz.cn/591553.htm
eto.hjiocz.cn/337793.htm
eto.hjiocz.cn/959153.htm
eto.hjiocz.cn/711913.htm
eto.hjiocz.cn/151373.htm
eto.hjiocz.cn/026283.htm
eto.hjiocz.cn/791793.htm
eti.hjiocz.cn/311513.htm
eti.hjiocz.cn/195193.htm
eti.hjiocz.cn/717533.htm
eti.hjiocz.cn/826043.htm
eti.hjiocz.cn/999713.htm
eti.hjiocz.cn/822803.htm
eti.hjiocz.cn/335513.htm
eti.hjiocz.cn/828483.htm
eti.hjiocz.cn/793133.htm
eti.hjiocz.cn/462603.htm
etu.hjiocz.cn/862283.htm
etu.hjiocz.cn/860403.htm
etu.hjiocz.cn/735193.htm
etu.hjiocz.cn/882443.htm
etu.hjiocz.cn/062283.htm
etu.hjiocz.cn/666443.htm
etu.hjiocz.cn/684043.htm
etu.hjiocz.cn/606803.htm
etu.hjiocz.cn/393753.htm
etu.hjiocz.cn/951573.htm
ety.hjiocz.cn/686223.htm
ety.hjiocz.cn/333313.htm
ety.hjiocz.cn/686243.htm
ety.hjiocz.cn/375593.htm
ety.hjiocz.cn/402883.htm
ety.hjiocz.cn/315593.htm
ety.hjiocz.cn/240083.htm
ety.hjiocz.cn/191153.htm
ety.hjiocz.cn/800883.htm
ety.hjiocz.cn/808663.htm
ett.hjiocz.cn/040063.htm
ett.hjiocz.cn/137393.htm
ett.hjiocz.cn/026263.htm
ett.hjiocz.cn/159593.htm
ett.hjiocz.cn/648463.htm
ett.hjiocz.cn/733173.htm
ett.hjiocz.cn/482403.htm
ett.hjiocz.cn/664063.htm
ett.hjiocz.cn/193593.htm
ett.hjiocz.cn/595933.htm
etr.hjiocz.cn/595753.htm
etr.hjiocz.cn/284063.htm
etr.hjiocz.cn/515953.htm
etr.hjiocz.cn/208483.htm
etr.hjiocz.cn/737193.htm
etr.hjiocz.cn/226283.htm
etr.hjiocz.cn/757193.htm
etr.hjiocz.cn/688223.htm
etr.hjiocz.cn/117113.htm
etr.hjiocz.cn/642863.htm
ete.hjiocz.cn/086263.htm
ete.hjiocz.cn/642623.htm
ete.hjiocz.cn/824403.htm
ete.hjiocz.cn/559953.htm
ete.hjiocz.cn/977773.htm
ete.hjiocz.cn/355933.htm
ete.hjiocz.cn/151113.htm
ete.hjiocz.cn/755553.htm
ete.hjiocz.cn/880403.htm
ete.hjiocz.cn/640483.htm
etw.hjiocz.cn/284883.htm
etw.hjiocz.cn/373573.htm
etw.hjiocz.cn/422483.htm
etw.hjiocz.cn/240263.htm
etw.hjiocz.cn/442083.htm
etw.hjiocz.cn/004883.htm
etw.hjiocz.cn/620803.htm
etw.hjiocz.cn/133793.htm
etw.hjiocz.cn/715933.htm
etw.hjiocz.cn/171133.htm
etq.hjiocz.cn/531713.htm
etq.hjiocz.cn/006403.htm
etq.hjiocz.cn/591373.htm
etq.hjiocz.cn/682643.htm
etq.hjiocz.cn/735553.htm
etq.hjiocz.cn/828463.htm
etq.hjiocz.cn/868803.htm
etq.hjiocz.cn/208483.htm
etq.hjiocz.cn/622483.htm
etq.hjiocz.cn/177393.htm
erm.hjiocz.cn/597593.htm
erm.hjiocz.cn/179753.htm
erm.hjiocz.cn/084803.htm
erm.hjiocz.cn/008883.htm
erm.hjiocz.cn/240063.htm
erm.hjiocz.cn/759953.htm
erm.hjiocz.cn/802863.htm
erm.hjiocz.cn/977573.htm
erm.hjiocz.cn/280243.htm
erm.hjiocz.cn/684063.htm
ern.hjiocz.cn/846843.htm
ern.hjiocz.cn/828263.htm
ern.hjiocz.cn/280463.htm
ern.hjiocz.cn/220443.htm
ern.hjiocz.cn/284843.htm
ern.hjiocz.cn/959333.htm
ern.hjiocz.cn/795513.htm
ern.hjiocz.cn/591753.htm
ern.hjiocz.cn/151313.htm
ern.hjiocz.cn/002443.htm
erb.hjiocz.cn/315373.htm
erb.hjiocz.cn/462683.htm
erb.hjiocz.cn/313133.htm
erb.hjiocz.cn/226603.htm
erb.hjiocz.cn/977733.htm
erb.hjiocz.cn/866843.htm
erb.hjiocz.cn/355133.htm
erb.hjiocz.cn/680063.htm
erb.hjiocz.cn/199553.htm
erb.hjiocz.cn/848663.htm
erv.hjiocz.cn/173313.htm
erv.hjiocz.cn/488083.htm
erv.hjiocz.cn/862603.htm
erv.hjiocz.cn/086603.htm
erv.hjiocz.cn/264683.htm
erv.hjiocz.cn/088203.htm
erv.hjiocz.cn/955733.htm
erv.hjiocz.cn/448803.htm
erv.hjiocz.cn/513133.htm
erv.hjiocz.cn/193733.htm
erc.hjiocz.cn/937953.htm
erc.hjiocz.cn/460263.htm
erc.hjiocz.cn/226243.htm
erc.hjiocz.cn/153933.htm
erc.hjiocz.cn/464223.htm
erc.hjiocz.cn/331153.htm
erc.hjiocz.cn/282283.htm
erc.hjiocz.cn/133193.htm
erc.hjiocz.cn/424243.htm
erc.hjiocz.cn/939753.htm
erx.hjiocz.cn/682283.htm
erx.hjiocz.cn/840623.htm
erx.hjiocz.cn/686063.htm
erx.hjiocz.cn/408443.htm
erx.hjiocz.cn/066843.htm
erx.hjiocz.cn/646463.htm
erx.hjiocz.cn/866883.htm
erx.hjiocz.cn/957313.htm
erx.hjiocz.cn/468463.htm
erx.hjiocz.cn/206243.htm
erz.hjiocz.cn/082263.htm
erz.hjiocz.cn/753513.htm
erz.hjiocz.cn/111313.htm
erz.hjiocz.cn/226043.htm
erz.hjiocz.cn/177913.htm
erz.hjiocz.cn/804403.htm
erz.hjiocz.cn/951773.htm
erz.hjiocz.cn/888883.htm
erz.hjiocz.cn/377153.htm
erz.hjiocz.cn/688663.htm
erl.hjiocz.cn/395133.htm
erl.hjiocz.cn/600443.htm
erl.hjiocz.cn/397933.htm
erl.hjiocz.cn/006043.htm
erl.hjiocz.cn/806663.htm
erl.hjiocz.cn/682683.htm
erl.hjiocz.cn/408843.htm
erl.hjiocz.cn/022243.htm
erl.hjiocz.cn/335593.htm
erl.hjiocz.cn/115353.htm
erk.hjiocz.cn/955373.htm
erk.hjiocz.cn/999193.htm
erk.hjiocz.cn/648403.htm
erk.hjiocz.cn/375933.htm
erk.hjiocz.cn/228483.htm
erk.hjiocz.cn/959373.htm
erk.hjiocz.cn/822263.htm
erk.hjiocz.cn/799993.htm
erk.hjiocz.cn/082483.htm
erk.hjiocz.cn/393153.htm
erj.hjiocz.cn/480403.htm
erj.hjiocz.cn/559353.htm
erj.hjiocz.cn/226423.htm
erj.hjiocz.cn/244063.htm
erj.hjiocz.cn/482623.htm
erj.hjiocz.cn/028683.htm
erj.hjiocz.cn/866463.htm
erj.hjiocz.cn/886243.htm
erj.hjiocz.cn/042023.htm
erj.hjiocz.cn/866663.htm
erh.hjiocz.cn/044063.htm
erh.hjiocz.cn/228463.htm
erh.hjiocz.cn/575393.htm
erh.hjiocz.cn/371593.htm
erh.hjiocz.cn/208483.htm
erh.hjiocz.cn/391373.htm
erh.hjiocz.cn/355593.htm
erh.hjiocz.cn/975333.htm
erh.hjiocz.cn/626263.htm
erh.hjiocz.cn/517373.htm
erg.hjiocz.cn/282863.htm
erg.hjiocz.cn/773193.htm
erg.hjiocz.cn/624063.htm
erg.hjiocz.cn/775793.htm
erg.hjiocz.cn/842683.htm
erg.hjiocz.cn/731193.htm
erg.hjiocz.cn/333373.htm
erg.hjiocz.cn/559313.htm
erg.hjiocz.cn/757313.htm
erg.hjiocz.cn/735533.htm
erf.hjiocz.cn/977373.htm
erf.hjiocz.cn/242203.htm
erf.hjiocz.cn/199533.htm
erf.hjiocz.cn/220203.htm
erf.hjiocz.cn/155173.htm
erf.hjiocz.cn/820823.htm
erf.hjiocz.cn/935913.htm
erf.hjiocz.cn/886403.htm
erf.hjiocz.cn/399173.htm
erf.hjiocz.cn/448843.htm
erd.hjiocz.cn/440403.htm
erd.hjiocz.cn/428423.htm
erd.hjiocz.cn/511333.htm
erd.hjiocz.cn/064803.htm
erd.hjiocz.cn/888043.htm
erd.hjiocz.cn/422043.htm
erd.hjiocz.cn/466463.htm
erd.hjiocz.cn/282063.htm
erd.hjiocz.cn/882823.htm
erd.hjiocz.cn/284023.htm
ers.hjiocz.cn/117793.htm
ers.hjiocz.cn/064023.htm
ers.hjiocz.cn/511313.htm
ers.hjiocz.cn/006483.htm
ers.hjiocz.cn/373973.htm
ers.hjiocz.cn/357773.htm
ers.hjiocz.cn/173553.htm
ers.hjiocz.cn/537193.htm
ers.hjiocz.cn/460423.htm
ers.hjiocz.cn/933953.htm
era.hjiocz.cn/040083.htm
era.hjiocz.cn/937973.htm
era.hjiocz.cn/406623.htm
era.hjiocz.cn/157313.htm
era.hjiocz.cn/884203.htm
era.hjiocz.cn/733953.htm
era.hjiocz.cn/028803.htm
era.hjiocz.cn/973333.htm
era.hjiocz.cn/028683.htm
era.hjiocz.cn/404223.htm
erp.hjiocz.cn/668063.htm
erp.hjiocz.cn/399553.htm
erp.hjiocz.cn/531373.htm
erp.hjiocz.cn/757773.htm
erp.hjiocz.cn/173773.htm
erp.hjiocz.cn/686603.htm
erp.hjiocz.cn/622663.htm
erp.hjiocz.cn/224023.htm
erp.hjiocz.cn/195733.htm
erp.hjiocz.cn/442683.htm
ero.hjiocz.cn/331733.htm
ero.hjiocz.cn/062483.htm
ero.hjiocz.cn/571373.htm
ero.hjiocz.cn/468283.htm
ero.hjiocz.cn/773533.htm
ero.hjiocz.cn/484643.htm
ero.hjiocz.cn/713193.htm
ero.hjiocz.cn/666643.htm
ero.hjiocz.cn/244463.htm
ero.hjiocz.cn/826203.htm
eri.hjiocz.cn/313793.htm
eri.hjiocz.cn/684803.htm
eri.hjiocz.cn/377533.htm
eri.hjiocz.cn/828603.htm
eri.hjiocz.cn/446403.htm
eri.hjiocz.cn/626243.htm
eri.hjiocz.cn/066683.htm
eri.hjiocz.cn/680463.htm
eri.hjiocz.cn/826043.htm
eri.hjiocz.cn/957553.htm
eru.hjiocz.cn/771553.htm
eru.hjiocz.cn/997373.htm
eru.hjiocz.cn/282883.htm
eru.hjiocz.cn/373173.htm
eru.hjiocz.cn/426483.htm
eru.hjiocz.cn/939993.htm
eru.hjiocz.cn/088203.htm
eru.hjiocz.cn/399113.htm
eru.hjiocz.cn/002883.htm
eru.hjiocz.cn/773573.htm
ery.hjiocz.cn/820283.htm
ery.hjiocz.cn/119333.htm
ery.hjiocz.cn/088603.htm
ery.hjiocz.cn/775553.htm
ery.hjiocz.cn/200483.htm
ery.hjiocz.cn/717173.htm
ery.hjiocz.cn/228643.htm
ery.hjiocz.cn/375313.htm
ery.hjiocz.cn/802223.htm
ery.hjiocz.cn/173373.htm
ert.hjiocz.cn/206243.htm
ert.hjiocz.cn/335733.htm
ert.hjiocz.cn/868283.htm
ert.hjiocz.cn/917513.htm
ert.hjiocz.cn/288483.htm
ert.hjiocz.cn/397393.htm
ert.hjiocz.cn/048463.htm
ert.hjiocz.cn/393933.htm
ert.hjiocz.cn/220843.htm
ert.hjiocz.cn/422483.htm
err.hjiocz.cn/426243.htm
err.hjiocz.cn/484423.htm
err.hjiocz.cn/684603.htm
err.hjiocz.cn/193133.htm
err.hjiocz.cn/973113.htm
err.hjiocz.cn/597133.htm
err.hjiocz.cn/791193.htm
err.hjiocz.cn/937593.htm
err.hjiocz.cn/759753.htm
err.hjiocz.cn/462683.htm
ere.mmmxz.cn/422603.htm
ere.mmmxz.cn/662063.htm
ere.mmmxz.cn/315573.htm
ere.mmmxz.cn/822463.htm
ere.mmmxz.cn/955153.htm
ere.mmmxz.cn/084243.htm
ere.mmmxz.cn/951753.htm
ere.mmmxz.cn/824843.htm
ere.mmmxz.cn/848823.htm
ere.mmmxz.cn/800863.htm
erw.mmmxz.cn/488623.htm
erw.mmmxz.cn/424223.htm
erw.mmmxz.cn/577113.htm
erw.mmmxz.cn/004403.htm
erw.mmmxz.cn/862263.htm
erw.mmmxz.cn/206683.htm
erw.mmmxz.cn/806223.htm
erw.mmmxz.cn/428043.htm
erw.mmmxz.cn/402463.htm
erw.mmmxz.cn/973593.htm
erq.mmmxz.cn/133533.htm
erq.mmmxz.cn/686083.htm
erq.mmmxz.cn/060843.htm
erq.mmmxz.cn/957973.htm
erq.mmmxz.cn/468663.htm
erq.mmmxz.cn/282863.htm
erq.mmmxz.cn/406003.htm
erq.mmmxz.cn/139733.htm
erq.mmmxz.cn/628823.htm
erq.mmmxz.cn/33.htm
eem.mmmxz.cn/806263.htm
eem.mmmxz.cn/757153.htm
eem.mmmxz.cn/066803.htm
eem.mmmxz.cn/973533.htm
eem.mmmxz.cn/040063.htm
