烟镁凹授酵


Linux sunrpc svc_recv服务端RPC接收与svc_process

svc_recv是SUNRPC服务端接收RPC请求的核心函数，位于net/sunrpc/svc_xprt.c。它是整个RPC服务端I/O事件循环的入口，由各服务线程（如nfsd的nfsd内核线程）在循环中反复调用。与之紧密配合的是svc_process，负责将接收到的RPC请求进行协议解析和具体的proc分发。

svc_recv函数定义：

```c
int svc_recv(struct svc_rqst *rqstp, long timeout)
{
    struct svc_xprt *xprt = NULL;
    struct svc_serv *serv = rqstp->rq_server;
    struct xdr_buf *arg = &rqstp->rq_arg;
    int len, err;

    /* 等待传入请求 */
    err = svc_xprt_receive(rqstp);
    if (err < 0)
        goto out;

    /* 如果没有待处理数据，从传输层接收 */
    if (rqstp->rq_arg.head[0].iov_len == 0) {
        err = svc_xprt_recvfrom(rqstp);
        if (err <= 0)
            goto out;
    }

    arg->len = err;
    rqstp->rq_arg.page_len = 0;
    rqstp->rq_res.len = 0;
    rqstp->rq_accept_stat = rpc_accept_stat_success;

    /* 此时完成接收，调用svc_process处理 */
    err = svc_process(rqstp);

out:
    return err;
}
```

svc_xprt_receive是等待请求的核心机制，它使用等待队列实现线程阻塞：

```c
int svc_xprt_receive(struct svc_rqst *rqstp)
{
    struct svc_serv *serv = rqstp->rq_server;
    struct svc_xprt *xprt;
    int err;

    /* 尝试获取一个已有数据的传输 */
    spin_lock_bh(&serv->sv_lock);
    xprt = svc_xprt_dequeue(serv);
    spin_unlock_bh(&serv->sv_lock);

    if (xprt) {
        rqstp->rq_xprt = xprt;
        __set_current_state(TASK_RUNNING);
        return 0;
    }

    /* 无可用传输，进入等待 */
    err = wait_event_interruptible_timeout(serv->sv_callback_waitq,
                        svc_thread_should_stop(rqstp) ||
                        ((xprt = svc_xprt_dequeue(serv)) != NULL),
                        timeout);

    if (xprt) {
        rqstp->rq_xprt = xprt;
        return 0;
    }

    return -EAGAIN;
}
```

svc_xprt_recvfrom从传输层socket读取数据到rqstp的接收缓冲区：

```c
static int svc_xprt_recvfrom(struct svc_rqst *rqstp)
{
    struct svc_xprt *xprt = rqstp->rq_xprt;
    struct xdr_buf *arg = &rqstp->rq_arg;
    struct kvec *head = arg->head;
    int len, want;

    want = svc_max_payload(xprt);
    head->iov_base = page_address(rqstp->rq_pages[0]);
    head->iov_len = want;

    len = xprt->xpt_ops->xpo_recvfrom(rqstp);
    if (len < 0)
        return len;

    arg->len = len;
    arg->tail[0].iov_len = 0;
    return len;
}
```

对于TCP传输，xpo_recvfrom实现为svc_tcp_recvfrom：

```c
static int svc_tcp_recvfrom(struct svc_rqst *rqstp)
{
    struct svc_xprt *xprt = rqstp->rq_xprt;
    struct socket *sock = xprt->xpt_sock;
    struct msghdr msg = {};
    struct kvec iov;
    int len, err;

    iov.iov_base = page_address(rqstp->rq_pages[0]);
    iov.iov_len = svc_max_payload(xprt);
    msg.msg_control = NULL;
    msg.msg_controllen = 0;
    msg.msg_flags = MSG_DONTWAIT;

    /* 读取RPC记录标记（record marker） */
    err = kernel_recvmsg(sock, &msg, &iov, 1, 4, MSG_DONTWAIT);
    if (err < 0)
        return err;

    /* 解析记录长度 */
    len = (ntohl(*(__be32 *)iov.iov_base) & RPC_FRAGMENT_SIZE_MASK);
    if (len > svc_max_payload(xprt))
        return -EINVAL;

    /* 读取实际RPC消息 */
    err = svc_xprt_read_messages(xprt, rqstp, len);
    if (err < 0)
        return err;

    return err;
}
```

数据接收完成后，svc_process接手解析和分发：

```c
int svc_process(struct svc_rqst *rqstp)
{
    struct xdr_stream xdr;
    __be32 *p;
    u32 proc;
    int err;

    xdr_init_decode(&xdr, &rqstp->rq_arg, rqstp->rq_arg.head[0].iov_base,
                    NULL);

    /* 解析RPC头部 */
    err = svc_rpcbind_resolve(rqstp);
    if (err)
        goto out;

    /* 解析RPC版本、程序号 */
    p = xdr_inline_decode(&xdr, 2 * sizeof(__be32));
    if (!p)
        goto out_bad_xdr;

    rqstp->rq_prog = be32_to_cpup(p++);
    rqstp->rq_vers = be32_to_cpup(p++);

    /* 查找程序表 */
    err = svc_process_common(rqstp);
    if (err)
        goto out;

    /* 构建响应头部 */
    rqstp->rq_res.head[0].iov_base = page_address(rqstp->rq_pages[0]);
    rqstp->rq_res.head[0].iov_len = 0;

    /* 调注册的service处理函数 */
    err = rqstp->rq_server->sv_program->pg_prog->pg_dispatch(rqstp, &xdr);
    if (err)
        goto out;

    /* 发送响应 */
    err = svc_send(rqstp);

out:
    return err;
}
```

svc_process_common根据程序号和版本号查找对应的svc_program和svc_version：

```c
static int svc_process_common(struct svc_rqst *rqstp)
{
    struct svc_serv *serv = rqstp->rq_server;
    struct svc_program *progp;
    struct svc_version *versp;
    u32 prog = rqstp->rq_prog;
    u32 vers = rqstp->rq_vers;

    /* 匹配程序 */
    for (progp = serv->sv_program; progp; progp = progp->pg_next) {
        if (prog == progp->pg_prog)
            break;
    }
    if (!progp)
        return -EINVAL;  /* PROG_UNAVAIL */

    /* 匹配版本 */
    if (vers >= progp->pg_nvers || !progp->pg_vers[vers])
        return -EINVAL;  /* PROG_MISMATCH */

    versp = progp->pg_vers[vers];
    rqstp->rq_procinfo = versp->vs_proc + rqstp->rq_proc;
    rqstp->rq_versp = versp;
    return 0;
}
```

以NFSD为例，nfsd_dispatch是实际的proc分发函数，它根据proc编号调用nfsd_procedures数组中的处理函数（如nfsd3_proc_read、nfsd3_proc_write等）。处理完成后，svc_send通过xpo_sendto将结果编码写入socket，完成一次完整的RPC请求-响应循环。

烧蘸吃在沼绦诙少兄赵特沸稳桌男

m.wvy.qtbzn.cn/44222.Doc
m.wvy.qtbzn.cn/84048.Doc
m.wvy.qtbzn.cn/17997.Doc
m.wvy.qtbzn.cn/46682.Doc
m.wvy.qtbzn.cn/53537.Doc
m.wvy.qtbzn.cn/02408.Doc
m.wvy.qtbzn.cn/93153.Doc
m.wvy.qtbzn.cn/02662.Doc
m.wvy.qtbzn.cn/48800.Doc
m.wvy.qtbzn.cn/31333.Doc
m.wvy.qtbzn.cn/35537.Doc
m.wvy.qtbzn.cn/42408.Doc
m.wvy.qtbzn.cn/88446.Doc
m.wvy.qtbzn.cn/00688.Doc
m.wvy.qtbzn.cn/88468.Doc
m.wvt.qtbzn.cn/82202.Doc
m.wvt.qtbzn.cn/77995.Doc
m.wvt.qtbzn.cn/60222.Doc
m.wvt.qtbzn.cn/53119.Doc
m.wvt.qtbzn.cn/15531.Doc
m.wvt.qtbzn.cn/35797.Doc
m.wvt.qtbzn.cn/46668.Doc
m.wvt.qtbzn.cn/77335.Doc
m.wvt.qtbzn.cn/99957.Doc
m.wvt.qtbzn.cn/95393.Doc
m.wvt.qtbzn.cn/73733.Doc
m.wvt.qtbzn.cn/11711.Doc
m.wvt.qtbzn.cn/06220.Doc
m.wvt.qtbzn.cn/57951.Doc
m.wvt.qtbzn.cn/53353.Doc
m.wvt.qtbzn.cn/57579.Doc
m.wvt.qtbzn.cn/31933.Doc
m.wvt.qtbzn.cn/53175.Doc
m.wvt.qtbzn.cn/99155.Doc
m.wvt.qtbzn.cn/42400.Doc
m.wvr.qtbzn.cn/88466.Doc
m.wvr.qtbzn.cn/15153.Doc
m.wvr.qtbzn.cn/08266.Doc
m.wvr.qtbzn.cn/20600.Doc
m.wvr.qtbzn.cn/3.Doc
m.wvr.qtbzn.cn/31171.Doc
m.wvr.qtbzn.cn/79719.Doc
m.wvr.qtbzn.cn/51757.Doc
m.wvr.qtbzn.cn/04284.Doc
m.wvr.qtbzn.cn/39375.Doc
m.wvr.qtbzn.cn/04240.Doc
m.wvr.qtbzn.cn/20486.Doc
m.wvr.qtbzn.cn/17351.Doc
m.wvr.qtbzn.cn/71537.Doc
m.wvr.qtbzn.cn/68404.Doc
m.wvr.qtbzn.cn/02644.Doc
m.wvr.qtbzn.cn/35139.Doc
m.wvr.qtbzn.cn/59535.Doc
m.wvr.qtbzn.cn/26860.Doc
m.wvr.qtbzn.cn/33939.Doc
m.wve.qtbzn.cn/37159.Doc
m.wve.qtbzn.cn/39155.Doc
m.wve.qtbzn.cn/91159.Doc
m.wve.qtbzn.cn/19795.Doc
m.wve.qtbzn.cn/93713.Doc
m.wve.qtbzn.cn/20060.Doc
m.wve.qtbzn.cn/79153.Doc
m.wve.qtbzn.cn/59753.Doc
m.wve.qtbzn.cn/97193.Doc
m.wve.qtbzn.cn/19119.Doc
m.wve.qtbzn.cn/57155.Doc
m.wve.qtbzn.cn/75311.Doc
m.wve.qtbzn.cn/59999.Doc
m.wve.qtbzn.cn/39571.Doc
m.wve.qtbzn.cn/95597.Doc
m.wve.qtbzn.cn/68406.Doc
m.wve.qtbzn.cn/44066.Doc
m.wve.qtbzn.cn/77915.Doc
m.wve.qtbzn.cn/19795.Doc
m.wve.qtbzn.cn/35379.Doc
m.wvw.qtbzn.cn/57173.Doc
m.wvw.qtbzn.cn/42846.Doc
m.wvw.qtbzn.cn/06624.Doc
m.wvw.qtbzn.cn/55973.Doc
m.wvw.qtbzn.cn/24482.Doc
m.wvw.qtbzn.cn/37917.Doc
m.wvw.qtbzn.cn/68282.Doc
m.wvw.qtbzn.cn/88488.Doc
m.wvw.qtbzn.cn/75713.Doc
m.wvw.qtbzn.cn/73517.Doc
m.wvw.qtbzn.cn/33317.Doc
m.wvw.qtbzn.cn/59379.Doc
m.wvw.qtbzn.cn/26408.Doc
m.wvw.qtbzn.cn/88266.Doc
m.wvw.qtbzn.cn/66246.Doc
m.wvw.qtbzn.cn/80440.Doc
m.wvw.qtbzn.cn/99511.Doc
m.wvw.qtbzn.cn/02628.Doc
m.wvw.qtbzn.cn/22062.Doc
m.wvw.qtbzn.cn/79131.Doc
m.wvq.qtbzn.cn/97799.Doc
m.wvq.qtbzn.cn/60244.Doc
m.wvq.qtbzn.cn/02824.Doc
m.wvq.qtbzn.cn/57575.Doc
m.wvq.qtbzn.cn/44426.Doc
m.wvq.qtbzn.cn/31937.Doc
m.wvq.qtbzn.cn/39199.Doc
m.wvq.qtbzn.cn/68464.Doc
m.wvq.qtbzn.cn/11159.Doc
m.wvq.qtbzn.cn/71933.Doc
m.wvq.qtbzn.cn/13319.Doc
m.wvq.qtbzn.cn/93711.Doc
m.wvq.qtbzn.cn/31131.Doc
m.wvq.qtbzn.cn/79973.Doc
m.wvq.qtbzn.cn/80640.Doc
m.wvq.qtbzn.cn/79759.Doc
m.wvq.qtbzn.cn/99735.Doc
m.wvq.qtbzn.cn/17551.Doc
m.wvq.qtbzn.cn/86426.Doc
m.wvq.qtbzn.cn/68624.Doc
m.wcm.qtbzn.cn/51979.Doc
m.wcm.qtbzn.cn/13115.Doc
m.wcm.qtbzn.cn/59117.Doc
m.wcm.qtbzn.cn/17593.Doc
m.wcm.qtbzn.cn/42648.Doc
m.wcm.qtbzn.cn/39113.Doc
m.wcm.qtbzn.cn/17177.Doc
m.wcm.qtbzn.cn/93551.Doc
m.wcm.qtbzn.cn/84666.Doc
m.wcm.qtbzn.cn/88482.Doc
m.wcm.qtbzn.cn/00602.Doc
m.wcm.qtbzn.cn/55731.Doc
m.wcm.qtbzn.cn/80028.Doc
m.wcm.qtbzn.cn/28668.Doc
m.wcm.qtbzn.cn/62682.Doc
m.wcm.qtbzn.cn/33133.Doc
m.wcm.qtbzn.cn/73173.Doc
m.wcm.qtbzn.cn/71395.Doc
m.wcm.qtbzn.cn/93757.Doc
m.wcm.qtbzn.cn/64442.Doc
m.wcn.qtbzn.cn/91991.Doc
m.wcn.qtbzn.cn/28440.Doc
m.wcn.qtbzn.cn/40020.Doc
m.wcn.qtbzn.cn/99159.Doc
m.wcn.qtbzn.cn/24868.Doc
m.wcn.qtbzn.cn/26220.Doc
m.wcn.qtbzn.cn/57375.Doc
m.wcn.qtbzn.cn/57757.Doc
m.wcn.qtbzn.cn/20482.Doc
m.wcn.qtbzn.cn/64600.Doc
m.wcn.qtbzn.cn/31575.Doc
m.wcn.qtbzn.cn/97337.Doc
m.wcn.qtbzn.cn/88844.Doc
m.wcn.qtbzn.cn/93397.Doc
m.wcn.qtbzn.cn/66626.Doc
m.wcn.qtbzn.cn/42608.Doc
m.wcn.qtbzn.cn/39751.Doc
m.wcn.qtbzn.cn/66448.Doc
m.wcn.qtbzn.cn/71759.Doc
m.wcn.qtbzn.cn/71931.Doc
m.wcb.qtbzn.cn/35911.Doc
m.wcb.qtbzn.cn/06428.Doc
m.wcb.qtbzn.cn/82626.Doc
m.wcb.qtbzn.cn/97379.Doc
m.wcb.qtbzn.cn/73573.Doc
m.wcb.qtbzn.cn/02660.Doc
m.wcb.qtbzn.cn/66282.Doc
m.wcb.qtbzn.cn/57797.Doc
m.wcb.qtbzn.cn/95339.Doc
m.wcb.qtbzn.cn/53113.Doc
m.wcb.qtbzn.cn/13151.Doc
m.wcb.qtbzn.cn/71391.Doc
m.wcb.qtbzn.cn/13553.Doc
m.wcb.qtbzn.cn/57311.Doc
m.wcb.qtbzn.cn/64800.Doc
m.wcb.qtbzn.cn/59539.Doc
m.wcb.qtbzn.cn/06060.Doc
m.wcb.qtbzn.cn/93797.Doc
m.wcb.qtbzn.cn/19911.Doc
m.wcb.qtbzn.cn/79157.Doc
m.wcv.qtbzn.cn/62446.Doc
m.wcv.qtbzn.cn/04044.Doc
m.wcv.qtbzn.cn/39519.Doc
m.wcv.qtbzn.cn/35555.Doc
m.wcv.qtbzn.cn/95391.Doc
m.wcv.qtbzn.cn/64660.Doc
m.wcv.qtbzn.cn/93157.Doc
m.wcv.qtbzn.cn/91571.Doc
m.wcv.qtbzn.cn/82800.Doc
m.wcv.qtbzn.cn/97333.Doc
m.wcv.qtbzn.cn/20664.Doc
m.wcv.qtbzn.cn/19797.Doc
m.wcv.qtbzn.cn/51359.Doc
m.wcv.qtbzn.cn/19377.Doc
m.wcv.qtbzn.cn/93351.Doc
m.wcv.qtbzn.cn/91539.Doc
m.wcv.qtbzn.cn/33351.Doc
m.wcv.qtbzn.cn/53357.Doc
m.wcv.qtbzn.cn/15735.Doc
m.wcv.qtbzn.cn/55777.Doc
m.wcc.qtbzn.cn/04442.Doc
m.wcc.qtbzn.cn/06420.Doc
m.wcc.qtbzn.cn/51537.Doc
m.wcc.qtbzn.cn/42248.Doc
m.wcc.qtbzn.cn/55715.Doc
m.wcc.qtbzn.cn/13159.Doc
m.wcc.qtbzn.cn/73517.Doc
m.wcc.qtbzn.cn/13755.Doc
m.wcc.qtbzn.cn/06424.Doc
m.wcc.qtbzn.cn/11159.Doc
m.wcc.qtbzn.cn/71733.Doc
m.wcc.qtbzn.cn/35995.Doc
m.wcc.qtbzn.cn/53599.Doc
m.wcc.qtbzn.cn/37979.Doc
m.wcc.qtbzn.cn/17179.Doc
m.wcc.qtbzn.cn/64682.Doc
m.wcc.qtbzn.cn/48246.Doc
m.wcc.qtbzn.cn/9.Doc
m.wcc.qtbzn.cn/39757.Doc
m.wcc.qtbzn.cn/86840.Doc
m.wcx.qtbzn.cn/02064.Doc
m.wcx.qtbzn.cn/77337.Doc
m.wcx.qtbzn.cn/28824.Doc
m.wcx.qtbzn.cn/59357.Doc
m.wcx.qtbzn.cn/73379.Doc
m.wcx.qtbzn.cn/60804.Doc
m.wcx.qtbzn.cn/44060.Doc
m.wcx.qtbzn.cn/17115.Doc
m.wcx.qtbzn.cn/71915.Doc
m.wcx.qtbzn.cn/86488.Doc
m.wcx.qtbzn.cn/17755.Doc
m.wcx.qtbzn.cn/04866.Doc
m.wcx.qtbzn.cn/57999.Doc
m.wcx.qtbzn.cn/62464.Doc
m.wcx.qtbzn.cn/95931.Doc
m.wcx.qtbzn.cn/20642.Doc
m.wcx.qtbzn.cn/31775.Doc
m.wcx.qtbzn.cn/19119.Doc
m.wcx.qtbzn.cn/33795.Doc
m.wcx.qtbzn.cn/00228.Doc
m.wcz.qtbzn.cn/53537.Doc
m.wcz.qtbzn.cn/37957.Doc
m.wcz.qtbzn.cn/28084.Doc
m.wcz.qtbzn.cn/28440.Doc
m.wcz.qtbzn.cn/73571.Doc
m.wcz.qtbzn.cn/88666.Doc
m.wcz.qtbzn.cn/31393.Doc
m.wcz.qtbzn.cn/13955.Doc
m.wcz.qtbzn.cn/39517.Doc
m.wcz.qtbzn.cn/53395.Doc
m.wcz.qtbzn.cn/73159.Doc
m.wcz.qtbzn.cn/68460.Doc
m.wcz.qtbzn.cn/00002.Doc
m.wcz.qtbzn.cn/84686.Doc
m.wcz.qtbzn.cn/97571.Doc
m.wcz.qtbzn.cn/95111.Doc
m.wcz.qtbzn.cn/51993.Doc
m.wcz.qtbzn.cn/11531.Doc
m.wcz.qtbzn.cn/97535.Doc
m.wcz.qtbzn.cn/11531.Doc
m.wcl.qtbzn.cn/44226.Doc
m.wcl.qtbzn.cn/42622.Doc
m.wcl.qtbzn.cn/15551.Doc
m.wcl.qtbzn.cn/97359.Doc
m.wcl.qtbzn.cn/79357.Doc
m.wcl.qtbzn.cn/59571.Doc
m.wcl.qtbzn.cn/97351.Doc
m.wcl.qtbzn.cn/24884.Doc
m.wcl.qtbzn.cn/95577.Doc
m.wcl.qtbzn.cn/99575.Doc
m.wcl.qtbzn.cn/46822.Doc
m.wcl.qtbzn.cn/95919.Doc
m.wcl.qtbzn.cn/97953.Doc
m.wcl.qtbzn.cn/55139.Doc
m.wcl.qtbzn.cn/00466.Doc
m.wcl.qtbzn.cn/40026.Doc
m.wcl.qtbzn.cn/17313.Doc
m.wcl.qtbzn.cn/93711.Doc
m.wcl.qtbzn.cn/04442.Doc
m.wcl.qtbzn.cn/80404.Doc
m.wck.qtbzn.cn/24888.Doc
m.wck.qtbzn.cn/24082.Doc
m.wck.qtbzn.cn/11319.Doc
m.wck.qtbzn.cn/11555.Doc
m.wck.qtbzn.cn/64800.Doc
m.wck.qtbzn.cn/86800.Doc
m.wck.qtbzn.cn/40868.Doc
m.wck.qtbzn.cn/73719.Doc
m.wck.qtbzn.cn/95991.Doc
m.wck.qtbzn.cn/33995.Doc
m.wck.qtbzn.cn/68426.Doc
m.wck.qtbzn.cn/86482.Doc
m.wck.qtbzn.cn/00604.Doc
m.wck.qtbzn.cn/33315.Doc
m.wck.qtbzn.cn/62082.Doc
m.wck.qtbzn.cn/35379.Doc
m.wck.qtbzn.cn/19591.Doc
m.wck.qtbzn.cn/66606.Doc
m.wck.qtbzn.cn/57557.Doc
m.wck.qtbzn.cn/86444.Doc
m.wcj.qtbzn.cn/37993.Doc
m.wcj.qtbzn.cn/60800.Doc
m.wcj.qtbzn.cn/73717.Doc
m.wcj.qtbzn.cn/91339.Doc
m.wcj.qtbzn.cn/51375.Doc
m.wcj.qtbzn.cn/46484.Doc
m.wcj.qtbzn.cn/97995.Doc
m.wcj.qtbzn.cn/17357.Doc
m.wcj.qtbzn.cn/26882.Doc
m.wcj.qtbzn.cn/28688.Doc
m.wcj.qtbzn.cn/46468.Doc
m.wcj.qtbzn.cn/97393.Doc
m.wcj.qtbzn.cn/31717.Doc
m.wcj.qtbzn.cn/04804.Doc
m.wcj.qtbzn.cn/39353.Doc
m.wcj.qtbzn.cn/95315.Doc
m.wcj.qtbzn.cn/20026.Doc
m.wcj.qtbzn.cn/73717.Doc
m.wcj.qtbzn.cn/31771.Doc
m.wcj.qtbzn.cn/91193.Doc
m.wch.qtbzn.cn/06284.Doc
m.wch.qtbzn.cn/06244.Doc
m.wch.qtbzn.cn/91991.Doc
m.wch.qtbzn.cn/13997.Doc
m.wch.qtbzn.cn/33555.Doc
m.wch.qtbzn.cn/99997.Doc
m.wch.qtbzn.cn/79371.Doc
m.wch.qtbzn.cn/97375.Doc
m.wch.qtbzn.cn/06204.Doc
m.wch.qtbzn.cn/19393.Doc
m.wch.qtbzn.cn/00202.Doc
m.wch.qtbzn.cn/15117.Doc
m.wch.qtbzn.cn/93593.Doc
m.wch.qtbzn.cn/74026.Doc
m.wch.qtbzn.cn/97117.Doc
m.wch.qtbzn.cn/64868.Doc
m.wch.qtbzn.cn/46228.Doc
m.wch.qtbzn.cn/08442.Doc
m.wch.qtbzn.cn/73379.Doc
m.wch.qtbzn.cn/28628.Doc
m.wcg.qtbzn.cn/73975.Doc
m.wcg.qtbzn.cn/51777.Doc
m.wcg.qtbzn.cn/88026.Doc
m.wcg.qtbzn.cn/31197.Doc
m.wcg.qtbzn.cn/40066.Doc
m.wcg.qtbzn.cn/37317.Doc
m.wcg.qtbzn.cn/77711.Doc
m.wcg.qtbzn.cn/33571.Doc
m.wcg.qtbzn.cn/73591.Doc
m.wcg.qtbzn.cn/75193.Doc
m.wcg.qtbzn.cn/00620.Doc
m.wcg.qtbzn.cn/77937.Doc
m.wcg.qtbzn.cn/88460.Doc
m.wcg.qtbzn.cn/73555.Doc
m.wcg.qtbzn.cn/88260.Doc
m.wcg.qtbzn.cn/33557.Doc
m.wcg.qtbzn.cn/51311.Doc
m.wcg.qtbzn.cn/35599.Doc
m.wcg.qtbzn.cn/57971.Doc
m.wcg.qtbzn.cn/57157.Doc
m.wcf.qtbzn.cn/06424.Doc
m.wcf.qtbzn.cn/57719.Doc
m.wcf.qtbzn.cn/95731.Doc
m.wcf.qtbzn.cn/59797.Doc
m.wcf.qtbzn.cn/77195.Doc
m.wcf.qtbzn.cn/35753.Doc
m.wcf.qtbzn.cn/00884.Doc
m.wcf.qtbzn.cn/80880.Doc
m.wcf.qtbzn.cn/91517.Doc
m.wcf.qtbzn.cn/37133.Doc
m.wcf.qtbzn.cn/95779.Doc
m.wcf.qtbzn.cn/06064.Doc
m.wcf.qtbzn.cn/73997.Doc
m.wcf.qtbzn.cn/73919.Doc
m.wcf.qtbzn.cn/42862.Doc
m.wcf.qtbzn.cn/59575.Doc
m.wcf.qtbzn.cn/08640.Doc
m.wcf.qtbzn.cn/17139.Doc
m.wcf.qtbzn.cn/59133.Doc
m.wcf.qtbzn.cn/91577.Doc
m.wcd.qtbzn.cn/86880.Doc
m.wcd.qtbzn.cn/46880.Doc
m.wcd.qtbzn.cn/06882.Doc
m.wcd.qtbzn.cn/97371.Doc
m.wcd.qtbzn.cn/11995.Doc
m.wcd.qtbzn.cn/22608.Doc
m.wcd.qtbzn.cn/24444.Doc
m.wcd.qtbzn.cn/37317.Doc
m.wcd.qtbzn.cn/11173.Doc
m.wcd.qtbzn.cn/99399.Doc
m.wcd.qtbzn.cn/42284.Doc
m.wcd.qtbzn.cn/22600.Doc
m.wcd.qtbzn.cn/93997.Doc
m.wcd.qtbzn.cn/82202.Doc
m.wcd.qtbzn.cn/28600.Doc
m.wcd.qtbzn.cn/93917.Doc
m.wcd.qtbzn.cn/11777.Doc
m.wcd.qtbzn.cn/75731.Doc
m.wcd.qtbzn.cn/53771.Doc
m.wcd.qtbzn.cn/97197.Doc
