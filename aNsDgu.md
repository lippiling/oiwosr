腿懦梦脸飞


Linux sch_fq公平队列FQ流分类与credit机制

Fair Queue（FQ）qdisc位于net/sched/sch_fq.c，核心目标是每个流（flow）一个FIFO队列，按轮询（DRR, Deficit Round Robin）方式调度，保证各流间的公平性，同时支持 pacing 和 per-flow 限速。

FQ的流分类基于sk_buff的hash值，默认使用内核计算的skb->hash。fq_classify函数将报文分配到对应流：

static struct fq_flow *fq_classify(struct Qdisc *sch, struct sk_buff *skb)
{
    struct fq_sched_data *q = qdisc_priv(sch);
    struct fq_flow *f;
    u32 hash;

    if (skb->protocol == htons(ETH_P_IP) ||
        skb->protocol == htons(ETH_P_IPV6)) {
        hash = skb_get_hash_perturb(skb, q->perturbation);
    } else {
        hash = skb->hash;
        if (!hash)
            hash = (u32)(unsigned long)skb_dst(skb) ^
                   skb->sk_hash;
    }

    hash = reciprocal_scale(hash, FQ_HASH_SIZE);
    f = &q->flows[hash];

    if (f->flowchain.prev == NULL) {
        struct fq_flow *new_flow;

        new_flow = fq_find_fitting_flow(q, skb, hash);
        if (new_flow)
            return new_flow;

        if (f->qlen > 1 || f->stats.stoll > q->flow_refill)
            return &q->internal;

        f->fq_tin = skb_find_txq(skb, q->flow_max_rate ?
                                  &q->rate_limiting_struct : NULL);
    }

    return f;
}

每个flow通过定时器进行pacing控制。fq_dequeue是核心调度函数：

static struct sk_buff *fq_dequeue(struct Qdisc *sch)
{
    struct fq_sched_data *q = qdisc_priv(sch);
    struct fq_flow *f;
    struct sk_buff *skb;
    u32 now = ktime_get_ns();
    s64 credit;

    f = list_first_entry_or_null(&q->new_flows, struct fq_flow,
                                 flowchain);
    if (!f) {
        f = list_first_entry_or_null(&q->old_flows, struct fq_flow,
                                     flowchain);
        if (!f)
            return NULL;
    }

    if (f->time_next_packet > now) {
        if (!q->timer_active) {
            q->timer_active = true;
            hrtimer_start(&q->fq_timer,
                          ns_to_ktime(f->time_next_packet - now),
                          HRTIMER_MODE_REL_PINNED);
        }
        return NULL;
    }

    credit = f->credit;
    skb = f->head;

    if (skb) {
        credit -= skb->len;

        if (credit < 0 && !q->rate_enable) {
            return NULL;
        }

        __skb_unlink(skb, &f->queue);
        sch->q.qlen--;

        f->credit = credit;
        if (f->credit <= 0) {
            f->credit += q->quantum;
            list_move_tail(&f->flowchain, &q->old_flows);
        }
    }

    if (f->credit > 0 && f->qlen > 0)
        list_move_tail(&f->flowchain, &q->new_flows);
    else if (f->qlen == 0)
        list_del_init(&f->flowchain);

    if (q->rate_enable) {
        f->time_next_packet = now +
            max_t(u64, q->flow_max_rate ?
                  div64_u64(skb->len * 1000ULL * NSEC_PER_USEC,
                            q->flow_max_rate) : 0,
                  skb->len * q->pacing_divider);
    }

    return skb;
}

credit机制是DRR的核心。每个flow拥有一个credit计数器，初始值为quantum。每次从该flow出队一个报文，减去其长度。当credit变为负数（或零以下）时，该flow被移到old_flows链表，并补充一个quantum的credit。调度器优先服务new_flows链表中的flow，只有new_flows为空时才处理old_flows。这种设计确保新产生的active flow不会被old_flows中的flow饿死。

quantum参数的默认值在fq_change函数中设定：

static int fq_change(struct Qdisc *sch, struct nlattr *opt,
                     struct netlink_ext_ack *extack)
{
    struct fq_sched_data *q = qdisc_priv(sch);
    struct tc_fq_qopt *ctl = nla_data(opt);

    if (ctl->quantum)
        q->quantum = ctl->quantum;
    else
        q->quantum = 2 * psched_mtu(qdisc_dev(sch));

    if (ctl->initial_quantum)
        q->initial_quantum = ctl->initial_quantum;
    else
        q->initial_quantum = 0;

    if (ctl->maxrate) {
        q->flow_max_rate = ctl->maxrate;
        q->rate_enable = 1;
    }

    if (ctl->pacing) {
        q->pacing_divisor = ctl->pacing;
    }
}

在credit计算中，fq_dequeue对每个flow出队后重新计算credit：

static void fq_flow_add_to_list(struct fq_sched_data *q,
                                struct fq_flow *f)
{
    if (f->credit <= 0) {
        f->credit += q->quantum;
        list_add_tail(&f->flowchain, &q->old_flows);
    } else {
        list_add_tail(&f->flowchain, &q->new_flows);
    }
}

flow从new_flows转移到old_flows的条件是credit耗尽，转移后补充quantum credit。但如果补充后credit仍为正，则留在new_flows继续参与调度。这种设计保证credit用尽的flow暂时让出调度机会，实现按字节严格公平。

per-flow的pacing依靠fq->time_next_packet字段，hrtimer到期后才会调用fq_dequeue，控制报文发送速率。timer回调函数为fq_timer：

static enum hrtimer_restart fq_timer(struct hrtimer *timer)
{
    struct fq_sched_data *q = container_of(timer, struct fq_sched_data,
                                           fq_timer);
    struct Qdisc *sch = q->sch;
    struct net_device *dev = qdisc_dev(sch);

    q->timer_active = false;
    __netif_schedule(sch);

    return HRTIMER_NORESTART;
}

timer仅触发一次调度，fq_dequeue在必要时重新启新timer。这种defer机制将pacing粒度控制在纳秒级，适合高速网卡下精确流量整形。

赖嗡藕腾诽孛傥谇试匀瘴坦喂竿河

hqs.vwbnt.cn/428242.Doc
hqs.vwbnt.cn/428804.Doc
hqs.vwbnt.cn/024646.Doc
hqs.vwbnt.cn/208460.Doc
hqs.vwbnt.cn/684022.Doc
hqs.vwbnt.cn/048480.Doc
hqa.vwbnt.cn/464240.Doc
hqa.vwbnt.cn/425694.Doc
hqa.vwbnt.cn/644420.Doc
hqa.vwbnt.cn/957604.Doc
hqa.vwbnt.cn/200082.Doc
hqa.vwbnt.cn/242663.Doc
hqa.vwbnt.cn/823006.Doc
hqa.vwbnt.cn/662640.Doc
hqa.vwbnt.cn/286662.Doc
hqa.vwbnt.cn/804602.Doc
hqp.vwbnt.cn/249121.Doc
hqp.vwbnt.cn/646268.Doc
hqp.vwbnt.cn/040844.Doc
hqp.vwbnt.cn/222062.Doc
hqp.vwbnt.cn/227660.Doc
hqp.vwbnt.cn/824044.Doc
hqp.vwbnt.cn/020606.Doc
hqp.vwbnt.cn/822026.Doc
hqp.vwbnt.cn/008884.Doc
hqp.vwbnt.cn/628640.Doc
hqo.vwbnt.cn/606800.Doc
hqo.vwbnt.cn/864444.Doc
hqo.vwbnt.cn/886084.Doc
hqo.vwbnt.cn/044408.Doc
hqo.vwbnt.cn/029341.Doc
hqo.vwbnt.cn/664886.Doc
hqo.vwbnt.cn/422420.Doc
hqo.vwbnt.cn/820460.Doc
hqo.vwbnt.cn/864064.Doc
hqo.vwbnt.cn/402800.Doc
hqi.vwbnt.cn/082130.Doc
hqi.vwbnt.cn/668044.Doc
hqi.vwbnt.cn/444448.Doc
hqi.vwbnt.cn/808088.Doc
hqi.vwbnt.cn/066040.Doc
hqi.vwbnt.cn/202220.Doc
hqi.vwbnt.cn/793115.Doc
hqi.vwbnt.cn/440220.Doc
hqi.vwbnt.cn/682646.Doc
hqi.vwbnt.cn/804084.Doc
hqu.vwbnt.cn/951997.Doc
hqu.vwbnt.cn/880680.Doc
hqu.vwbnt.cn/640200.Doc
hqu.vwbnt.cn/806808.Doc
hqu.vwbnt.cn/282844.Doc
hqu.vwbnt.cn/260020.Doc
hqu.vwbnt.cn/064204.Doc
hqu.vwbnt.cn/591951.Doc
hqu.vwbnt.cn/868426.Doc
hqu.vwbnt.cn/446484.Doc
hqy.vwbnt.cn/662866.Doc
hqy.vwbnt.cn/062822.Doc
hqy.vwbnt.cn/062640.Doc
hqy.vwbnt.cn/262646.Doc
hqy.vwbnt.cn/026424.Doc
hqy.vwbnt.cn/822626.Doc
hqy.vwbnt.cn/420888.Doc
hqy.vwbnt.cn/068082.Doc
hqy.vwbnt.cn/000266.Doc
hqy.vwbnt.cn/661028.Doc
hqt.nfsid.cn/317313.Doc
hqt.nfsid.cn/482248.Doc
hqt.nfsid.cn/020260.Doc
hqt.nfsid.cn/662062.Doc
hqt.nfsid.cn/711173.Doc
hqt.nfsid.cn/822624.Doc
hqt.nfsid.cn/686729.Doc
hqt.nfsid.cn/684606.Doc
hqt.nfsid.cn/126498.Doc
hqt.nfsid.cn/060484.Doc
hqr.nfsid.cn/968790.Doc
hqr.nfsid.cn/039768.Doc
hqr.nfsid.cn/260086.Doc
hqr.nfsid.cn/048828.Doc
hqr.nfsid.cn/551917.Doc
hqr.nfsid.cn/080637.Doc
hqr.nfsid.cn/428866.Doc
hqr.nfsid.cn/824222.Doc
hqr.nfsid.cn/408082.Doc
hqr.nfsid.cn/955911.Doc
hqe.nfsid.cn/242062.Doc
hqe.nfsid.cn/262048.Doc
hqe.nfsid.cn/486468.Doc
hqe.nfsid.cn/572815.Doc
hqe.nfsid.cn/020464.Doc
hqe.nfsid.cn/886200.Doc
hqe.nfsid.cn/408806.Doc
hqe.nfsid.cn/280402.Doc
hqe.nfsid.cn/422826.Doc
hqe.nfsid.cn/564985.Doc
hqw.nfsid.cn/668046.Doc
hqw.nfsid.cn/123332.Doc
hqw.nfsid.cn/206206.Doc
hqw.nfsid.cn/040846.Doc
hqw.nfsid.cn/664204.Doc
hqw.nfsid.cn/006068.Doc
hqw.nfsid.cn/462260.Doc
hqw.nfsid.cn/466484.Doc
hqw.nfsid.cn/664846.Doc
hqw.nfsid.cn/919023.Doc
hqq.nfsid.cn/068022.Doc
hqq.nfsid.cn/082482.Doc
hqq.nfsid.cn/048027.Doc
hqq.nfsid.cn/444640.Doc
hqq.nfsid.cn/680680.Doc
hqq.nfsid.cn/400846.Doc
hqq.nfsid.cn/026260.Doc
hqq.nfsid.cn/453052.Doc
hqq.nfsid.cn/444406.Doc
hqq.nfsid.cn/462224.Doc
gmm.nfsid.cn/205221.Doc
gmm.nfsid.cn/264084.Doc
gmm.nfsid.cn/648044.Doc
gmm.nfsid.cn/448020.Doc
gmm.nfsid.cn/012363.Doc
gmm.nfsid.cn/480626.Doc
gmm.nfsid.cn/640262.Doc
gmm.nfsid.cn/260088.Doc
gmm.nfsid.cn/424822.Doc
gmm.nfsid.cn/866604.Doc
gmn.nfsid.cn/400482.Doc
gmn.nfsid.cn/644468.Doc
gmn.nfsid.cn/260800.Doc
gmn.nfsid.cn/840628.Doc
gmn.nfsid.cn/064628.Doc
gmn.nfsid.cn/171133.Doc
gmn.nfsid.cn/682000.Doc
gmn.nfsid.cn/848604.Doc
gmn.nfsid.cn/448004.Doc
gmn.nfsid.cn/204480.Doc
gmb.nfsid.cn/260008.Doc
gmb.nfsid.cn/884062.Doc
gmb.nfsid.cn/060084.Doc
gmb.nfsid.cn/282468.Doc
gmb.nfsid.cn/820642.Doc
gmb.nfsid.cn/462426.Doc
gmb.nfsid.cn/644066.Doc
gmb.nfsid.cn/248808.Doc
gmb.nfsid.cn/020886.Doc
gmb.nfsid.cn/513733.Doc
gmv.nfsid.cn/820008.Doc
gmv.nfsid.cn/662004.Doc
gmv.nfsid.cn/357793.Doc
gmv.nfsid.cn/260440.Doc
gmv.nfsid.cn/018704.Doc
gmv.nfsid.cn/608620.Doc
gmv.nfsid.cn/886406.Doc
gmv.nfsid.cn/806068.Doc
gmv.nfsid.cn/646444.Doc
gmv.nfsid.cn/646404.Doc
gmc.nfsid.cn/793799.Doc
gmc.nfsid.cn/406242.Doc
gmc.nfsid.cn/444040.Doc
gmc.nfsid.cn/173991.Doc
gmc.nfsid.cn/404664.Doc
gmc.nfsid.cn/666488.Doc
gmc.nfsid.cn/826662.Doc
gmc.nfsid.cn/296685.Doc
gmc.nfsid.cn/519638.Doc
gmc.nfsid.cn/571957.Doc
gmx.nfsid.cn/080828.Doc
gmx.nfsid.cn/462864.Doc
gmx.nfsid.cn/024442.Doc
gmx.nfsid.cn/460420.Doc
gmx.nfsid.cn/880400.Doc
gmx.nfsid.cn/804620.Doc
gmx.nfsid.cn/422886.Doc
gmx.nfsid.cn/480068.Doc
gmx.nfsid.cn/486822.Doc
gmx.nfsid.cn/356946.Doc
gmz.nfsid.cn/206626.Doc
gmz.nfsid.cn/131393.Doc
gmz.nfsid.cn/480428.Doc
gmz.nfsid.cn/888066.Doc
gmz.nfsid.cn/842624.Doc
gmz.nfsid.cn/622040.Doc
gmz.nfsid.cn/363817.Doc
gmz.nfsid.cn/402482.Doc
gmz.nfsid.cn/284406.Doc
gmz.nfsid.cn/422800.Doc
gml.nfsid.cn/142857.Doc
gml.nfsid.cn/222224.Doc
gml.nfsid.cn/152514.Doc
gml.nfsid.cn/808068.Doc
gml.nfsid.cn/048668.Doc
gml.nfsid.cn/579171.Doc
gml.nfsid.cn/684062.Doc
gml.nfsid.cn/260242.Doc
gml.nfsid.cn/359593.Doc
gml.nfsid.cn/848484.Doc
gmk.nfsid.cn/846686.Doc
gmk.nfsid.cn/420404.Doc
gmk.nfsid.cn/600626.Doc
gmk.nfsid.cn/060642.Doc
gmk.nfsid.cn/791551.Doc
gmk.nfsid.cn/265498.Doc
gmk.nfsid.cn/622642.Doc
gmk.nfsid.cn/042880.Doc
gmk.nfsid.cn/022868.Doc
gmk.nfsid.cn/062244.Doc
gmj.nfsid.cn/086684.Doc
gmj.nfsid.cn/446000.Doc
gmj.nfsid.cn/028262.Doc
gmj.nfsid.cn/624640.Doc
gmj.nfsid.cn/151203.Doc
gmj.nfsid.cn/204626.Doc
gmj.nfsid.cn/008440.Doc
gmj.nfsid.cn/600408.Doc
gmj.nfsid.cn/178791.Doc
gmj.nfsid.cn/244008.Doc
gmh.nfsid.cn/402042.Doc
gmh.nfsid.cn/201699.Doc
gmh.nfsid.cn/840648.Doc
gmh.nfsid.cn/244266.Doc
gmh.nfsid.cn/468446.Doc
gmh.nfsid.cn/774964.Doc
gmh.nfsid.cn/222222.Doc
gmh.nfsid.cn/112575.Doc
gmh.nfsid.cn/264004.Doc
gmh.nfsid.cn/448462.Doc
gmg.nfsid.cn/850876.Doc
gmg.nfsid.cn/202246.Doc
gmg.nfsid.cn/008664.Doc
gmg.nfsid.cn/826486.Doc
gmg.nfsid.cn/802068.Doc
gmg.nfsid.cn/880244.Doc
gmg.nfsid.cn/958361.Doc
gmg.nfsid.cn/680686.Doc
gmg.nfsid.cn/084488.Doc
gmg.nfsid.cn/066888.Doc
gmf.nfsid.cn/866848.Doc
gmf.nfsid.cn/884602.Doc
gmf.nfsid.cn/979473.Doc
gmf.nfsid.cn/220046.Doc
gmf.nfsid.cn/004206.Doc
gmf.nfsid.cn/826662.Doc
gmf.nfsid.cn/424062.Doc
gmf.nfsid.cn/317668.Doc
gmf.nfsid.cn/284422.Doc
gmf.nfsid.cn/426488.Doc
gmd.nfsid.cn/517591.Doc
gmd.nfsid.cn/460008.Doc
gmd.nfsid.cn/227182.Doc
gmd.nfsid.cn/424488.Doc
gmd.nfsid.cn/486468.Doc
gmd.nfsid.cn/426006.Doc
gmd.nfsid.cn/462484.Doc
gmd.nfsid.cn/442406.Doc
gmd.nfsid.cn/406442.Doc
gmd.nfsid.cn/595951.Doc
gms.nfsid.cn/800666.Doc
gms.nfsid.cn/846372.Doc
gms.nfsid.cn/844206.Doc
gms.nfsid.cn/233639.Doc
gms.nfsid.cn/608860.Doc
gms.nfsid.cn/704514.Doc
gms.nfsid.cn/195779.Doc
gms.nfsid.cn/400068.Doc
gms.nfsid.cn/482662.Doc
gms.nfsid.cn/226226.Doc
gma.nfsid.cn/802802.Doc
gma.nfsid.cn/439219.Doc
gma.nfsid.cn/666024.Doc
gma.nfsid.cn/703491.Doc
gma.nfsid.cn/757531.Doc
gma.nfsid.cn/322205.Doc
gma.nfsid.cn/826486.Doc
gma.nfsid.cn/064448.Doc
gma.nfsid.cn/148685.Doc
gma.nfsid.cn/773995.Doc
gmp.nfsid.cn/444404.Doc
gmp.nfsid.cn/407484.Doc
gmp.nfsid.cn/402068.Doc
gmp.nfsid.cn/060060.Doc
gmp.nfsid.cn/440262.Doc
gmp.nfsid.cn/480222.Doc
gmp.nfsid.cn/424400.Doc
gmp.nfsid.cn/019464.Doc
gmp.nfsid.cn/468020.Doc
gmp.nfsid.cn/659856.Doc
gmo.nfsid.cn/484428.Doc
gmo.nfsid.cn/379777.Doc
gmo.nfsid.cn/640268.Doc
gmo.nfsid.cn/311311.Doc
gmo.nfsid.cn/605963.Doc
gmo.nfsid.cn/688268.Doc
gmo.nfsid.cn/460822.Doc
gmo.nfsid.cn/084008.Doc
gmo.nfsid.cn/040288.Doc
gmo.nfsid.cn/587623.Doc
gmi.nfsid.cn/206268.Doc
gmi.nfsid.cn/755311.Doc
gmi.nfsid.cn/044288.Doc
gmi.nfsid.cn/423666.Doc
gmi.nfsid.cn/855378.Doc
gmi.nfsid.cn/626604.Doc
gmi.nfsid.cn/620864.Doc
gmi.nfsid.cn/062480.Doc
gmi.nfsid.cn/459166.Doc
gmi.nfsid.cn/862882.Doc
gmu.nfsid.cn/270461.Doc
gmu.nfsid.cn/226640.Doc
gmu.nfsid.cn/868082.Doc
gmu.nfsid.cn/919151.Doc
gmu.nfsid.cn/268684.Doc
gmu.nfsid.cn/646808.Doc
gmu.nfsid.cn/468668.Doc
gmu.nfsid.cn/388215.Doc
gmu.nfsid.cn/864844.Doc
gmu.nfsid.cn/546674.Doc
gmy.nfsid.cn/464244.Doc
gmy.nfsid.cn/804042.Doc
gmy.nfsid.cn/208882.Doc
gmy.nfsid.cn/064602.Doc
gmy.nfsid.cn/480228.Doc
gmy.nfsid.cn/628200.Doc
gmy.nfsid.cn/648886.Doc
gmy.nfsid.cn/226486.Doc
gmy.nfsid.cn/360651.Doc
gmy.nfsid.cn/842622.Doc
gmt.nfsid.cn/600266.Doc
gmt.nfsid.cn/369633.Doc
gmt.nfsid.cn/975731.Doc
gmt.nfsid.cn/606662.Doc
gmt.nfsid.cn/759751.Doc
gmt.nfsid.cn/357157.Doc
gmt.nfsid.cn/686280.Doc
gmt.nfsid.cn/026602.Doc
gmt.nfsid.cn/200086.Doc
gmt.nfsid.cn/357393.Doc
gmr.nfsid.cn/240080.Doc
gmr.nfsid.cn/686024.Doc
gmr.nfsid.cn/137537.Doc
gmr.nfsid.cn/040204.Doc
gmr.nfsid.cn/004420.Doc
gmr.nfsid.cn/280880.Doc
gmr.nfsid.cn/111553.Doc
gmr.nfsid.cn/291540.Doc
gmr.nfsid.cn/602844.Doc
gmr.nfsid.cn/868800.Doc
gme.nfsid.cn/640460.Doc
gme.nfsid.cn/422660.Doc
gme.nfsid.cn/404224.Doc
gme.nfsid.cn/660242.Doc
gme.nfsid.cn/203941.Doc
gme.nfsid.cn/840888.Doc
gme.nfsid.cn/224000.Doc
gme.nfsid.cn/688042.Doc
gme.nfsid.cn/284866.Doc
gme.nfsid.cn/860080.Doc
gmw.nfsid.cn/024448.Doc
gmw.nfsid.cn/408668.Doc
gmw.nfsid.cn/000150.Doc
gmw.nfsid.cn/400262.Doc
gmw.nfsid.cn/408444.Doc
gmw.nfsid.cn/171393.Doc
gmw.nfsid.cn/802068.Doc
gmw.nfsid.cn/933337.Doc
gmw.nfsid.cn/842804.Doc
gmw.nfsid.cn/664620.Doc
gmq.nfsid.cn/606284.Doc
gmq.nfsid.cn/807222.Doc
gmq.nfsid.cn/228062.Doc
gmq.nfsid.cn/806484.Doc
gmq.nfsid.cn/842248.Doc
gmq.nfsid.cn/602246.Doc
gmq.nfsid.cn/819193.Doc
gmq.nfsid.cn/440020.Doc
gmq.nfsid.cn/020808.Doc
gmq.nfsid.cn/000662.Doc
gnm.nfsid.cn/020202.Doc
gnm.nfsid.cn/062426.Doc
gnm.nfsid.cn/997371.Doc
gnm.nfsid.cn/530713.Doc
gnm.nfsid.cn/444284.Doc
gnm.nfsid.cn/282844.Doc
gnm.nfsid.cn/267539.Doc
gnm.nfsid.cn/646682.Doc
gnm.nfsid.cn/355355.Doc
gnm.nfsid.cn/800646.Doc
gnn.nfsid.cn/171759.Doc
gnn.nfsid.cn/866024.Doc
gnn.nfsid.cn/804824.Doc
gnn.nfsid.cn/426826.Doc
gnn.nfsid.cn/464046.Doc
gnn.nfsid.cn/199555.Doc
gnn.nfsid.cn/042644.Doc
gnn.nfsid.cn/237123.Doc
gnn.nfsid.cn/806480.Doc
