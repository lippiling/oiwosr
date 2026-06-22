趁壹用洗刎


Linux sch_htb令牌桶层级令牌借贷与borrow传播

HTB（Hierarchical Token Bucket）位于net/sched/sch_htb.c，构建层级令牌桶树形结构。每个class拥有rate（保证速率）和ceil（上限速率）两个令牌桶，父子class间通过borrow机制传递令牌。

令牌生成的入口在htb_charge_class，每次出队报文后扣除相应class的令牌：

static void htb_charge_class(struct htb_sched *q, struct htb_class *cl,
                             int bytes, int level)
{
    long tokens;
    int deficit;
    long new_tokens;
    long new_etime;

    do {
        int mdiff;
        long diff;

        if (cl->level >= level)
            break;

        tokens = atomic_long_read(&cl->tokens);
        if (tokens == HTB_HPRIORITY_LAST)
            break;

        new_tokens = q->now + cl->cbuffer;
        if (tokens < new_tokens)
            tokens = new_tokens;

        diff = tokens - now;
        if (cl->cmode != HTB_CAN_SEND)
            break;

        cl->tokens = tokens - bytes;
        cl->age = q->now;

        if (cl->tokens <= -cl->quantum) {
            htb_safe_rb_erase(&cl->pq_node, &q->hprio[cl->aprio]);
            RB_CLEAR_NODE(&cl->pq_node);
            cl->cmode = HTB_MAY_BORROW;
        }

        mdiff = atomic_long_sub_return(bytes, &cl->borrowed);
        if (mdiff > cl->quantum) {
            htb_borrow_cb(cl, bytes);
        }

        cl = cl->parent;
    } while (cl);
}

每个class的cmode有三种状态：HTB_CAN_SEND（令牌充足可直接发送）、HTB_MAY_BORROW（自身令牌不足，允许向父class借贷）、HTB_CANT_SEND（父class也无令牌可用，不可发送）。状态切换通过htb_class_mode函数判定：

static void htb_class_mode(struct htb_class *cl, int *diff)
{
    WARN_ON_ONCE(!spin_is_locked(&cl->sched->lock));

    if (cl->tokens >= 0) {
        *diff = cl->quantum;
        cl->cmode = HTB_CAN_SEND;
    } else if (cl->borrowed > -cl->quantum) {
        *diff = -cl->tokens;
        cl->cmode = HTB_MAY_BORROW;
    } else {
        *diff = 0;
        cl->cmode = HTB_CANT_SEND;
    }
}

borrow传播的核心逻辑在htb_dequeue_tree中体现：

static struct sk_buff *htb_dequeue_tree(struct htb_sched *q, int prio)
{
    struct htb_class *cl, *start;
    struct sk_buff *skb;
    int m;

    start = cl = htb_lookup_leaf(q, prio);
    if (!cl)
        return NULL;

    do {
        if (cl->cmode == HTB_CAN_SEND) {
            skb = htb_dequeue_leaf(q, cl, prio);
            if (skb) {
                htb_charge_class(q, cl, skb->len, cl->level);
                return skb;
            }
        }

        if (cl->cmode == HTB_MAY_BORROW) {
            if (!cl->parent || cl->parent->cmode == HTB_CANT_SEND)
                continue;

            if (htb_borrow(q, cl)) {
                skb = htb_dequeue_leaf(q, cl, prio);
                if (skb) {
                    htb_charge_class(q, cl, skb->len, cl->level);
                    return skb;
                }
            }
        }

        cl = cl->next;
    } while (cl != start);

    return NULL;
}

实际borrow执行在htb_borrow函数，该函数尝试从父class借取令牌：

static int htb_borrow(struct htb_sched *q, struct htb_class *cl)
{
    struct htb_class *parent = cl->parent;
    long tokens;
    long borrowed;
    long needed;

    if (!parent || parent->cmode == HTB_CANT_SEND)
        return 0;

    needed = cl->quantum;
    borrowed = atomic_long_read(&cl->borrowed);
    tokens = atomic_long_read(&parent->tokens);

    if (parent->cmode == HTB_CAN_SEND) {
        if (tokens >= needed) {
            atomic_long_sub(needed, &parent->tokens);
            atomic_long_add(needed, &cl->borrowed);
            return 1;
        }
        return 0;
    }

    if (parent->cmode == HTB_MAY_BORROW)
        return htb_borrow(q, parent);

    return 0;
}

令牌的定时补充通过htb_work_func高位精度定时器实现：

static void htb_work_func(struct hrb *work)
{
    struct htb_sched *q = container_of(work, struct htb_sched,
                                       htimer.work);
    struct Qdisc *sch = q->sch;
    struct htb_class *cl;
    u64 now = ktime_get_ns();
    long diff;

    spin_lock_bh(&sch->lock);
    q->jiffies = jiffies;

    list_for_each_entry(cl, &q->drops, drops_list) {
        do {
            diff = cl->mbuffer - (long)(now - cl->age);
            if (diff <= 0) {
                cl->tokens = min_t(long, cl->tokens + cl->rate.rate *
                                   (long)TICK_NSEC, cl->buffer);
                cl->age = now;
            }
        } while (0);

        htb_class_mode(cl, &diff);
        if (diff)
            htb_activate(q, cl);
    }

    if (__netif_schedule(sch))
        hrtimer_start(&q->htimer, ns_to_ktime(q->rate_delt),
                      HRTIMER_MODE_REL_PINNED);

    spin_unlock_bh(&sch->lock);
}

每个class维护两个令牌桶：tokens对应rate（保证带宽），ctokens对应ceil（上限带宽）。htb_charge_class同时扣除两个桶的令牌。当ctokens桶用完时，即使从父class借到了令牌，报文仍会因超过ceil而被限流或丢弃。时间片用jiffies和ktime_get_ns混合计算，保证在低速率下也能精确产生令牌。

层级拓扑在htb_change_class时建立，parent指针指向父class，level字段标识层级深度。叶子class在htb_destroy_class时级联销毁子class。根class的rate必须为全qdisc总带宽，否则上层的borrow调用会因父class令牌不足而失败，导致下层的ceil限流失效。

肆脑牙张噬刎适儋厍亟研舷融盒涸

grj.eiyve.cn/440684.Doc
grj.eiyve.cn/846664.Doc
grj.eiyve.cn/310437.Doc
grj.eiyve.cn/260086.Doc
grj.eiyve.cn/804062.Doc
grj.eiyve.cn/088648.Doc
grj.eiyve.cn/468224.Doc
grj.eiyve.cn/400084.Doc
grj.eiyve.cn/195551.Doc
grj.eiyve.cn/064822.Doc
grh.eiyve.cn/628824.Doc
grh.eiyve.cn/668226.Doc
grh.eiyve.cn/420822.Doc
grh.eiyve.cn/448464.Doc
grh.eiyve.cn/824882.Doc
grh.eiyve.cn/340887.Doc
grh.eiyve.cn/644226.Doc
grh.eiyve.cn/068446.Doc
grh.eiyve.cn/769144.Doc
grh.eiyve.cn/624468.Doc
grg.eiyve.cn/608480.Doc
grg.eiyve.cn/886064.Doc
grg.eiyve.cn/596440.Doc
grg.eiyve.cn/404840.Doc
grg.eiyve.cn/804822.Doc
grg.eiyve.cn/664662.Doc
grg.eiyve.cn/488648.Doc
grg.eiyve.cn/022420.Doc
grg.eiyve.cn/860240.Doc
grg.eiyve.cn/008260.Doc
grf.eiyve.cn/846088.Doc
grf.eiyve.cn/008286.Doc
grf.eiyve.cn/448406.Doc
grf.eiyve.cn/942969.Doc
grf.eiyve.cn/468406.Doc
grf.eiyve.cn/640644.Doc
grf.eiyve.cn/242484.Doc
grf.eiyve.cn/622804.Doc
grf.eiyve.cn/317179.Doc
grf.eiyve.cn/262628.Doc
grd.eiyve.cn/684240.Doc
grd.eiyve.cn/375573.Doc
grd.eiyve.cn/080668.Doc
grd.eiyve.cn/448286.Doc
grd.eiyve.cn/553959.Doc
grd.eiyve.cn/226060.Doc
grd.eiyve.cn/400000.Doc
grd.eiyve.cn/642282.Doc
grd.eiyve.cn/202042.Doc
grd.eiyve.cn/484864.Doc
grs.eiyve.cn/824444.Doc
grs.eiyve.cn/064648.Doc
grs.eiyve.cn/357753.Doc
grs.eiyve.cn/863008.Doc
grs.eiyve.cn/808684.Doc
grs.eiyve.cn/262248.Doc
grs.eiyve.cn/820460.Doc
grs.eiyve.cn/200280.Doc
grs.eiyve.cn/628871.Doc
grs.eiyve.cn/740209.Doc
gra.eiyve.cn/666648.Doc
gra.eiyve.cn/423882.Doc
gra.eiyve.cn/082860.Doc
gra.eiyve.cn/004802.Doc
gra.eiyve.cn/462222.Doc
gra.eiyve.cn/462288.Doc
gra.eiyve.cn/004068.Doc
gra.eiyve.cn/868826.Doc
gra.eiyve.cn/137335.Doc
gra.eiyve.cn/396664.Doc
grp.eiyve.cn/327854.Doc
grp.eiyve.cn/822664.Doc
grp.eiyve.cn/028806.Doc
grp.eiyve.cn/068686.Doc
grp.eiyve.cn/202680.Doc
grp.eiyve.cn/486202.Doc
grp.eiyve.cn/806880.Doc
grp.eiyve.cn/422242.Doc
grp.eiyve.cn/286840.Doc
grp.eiyve.cn/804882.Doc
gro.vwbnt.cn/626468.Doc
gro.vwbnt.cn/826044.Doc
gro.vwbnt.cn/074852.Doc
gro.vwbnt.cn/248464.Doc
gro.vwbnt.cn/351539.Doc
gro.vwbnt.cn/844428.Doc
gro.vwbnt.cn/266088.Doc
gro.vwbnt.cn/824002.Doc
gro.vwbnt.cn/402086.Doc
gro.vwbnt.cn/420868.Doc
gri.vwbnt.cn/660828.Doc
gri.vwbnt.cn/862060.Doc
gri.vwbnt.cn/060624.Doc
gri.vwbnt.cn/626244.Doc
gri.vwbnt.cn/226682.Doc
gri.vwbnt.cn/446460.Doc
gri.vwbnt.cn/048064.Doc
gri.vwbnt.cn/246288.Doc
gri.vwbnt.cn/268628.Doc
gri.vwbnt.cn/040460.Doc
gru.vwbnt.cn/028044.Doc
gru.vwbnt.cn/268420.Doc
gru.vwbnt.cn/046248.Doc
gru.vwbnt.cn/957637.Doc
gru.vwbnt.cn/640802.Doc
gru.vwbnt.cn/402428.Doc
gru.vwbnt.cn/282020.Doc
gru.vwbnt.cn/246228.Doc
gru.vwbnt.cn/084884.Doc
gru.vwbnt.cn/268446.Doc
gry.vwbnt.cn/351177.Doc
gry.vwbnt.cn/203702.Doc
gry.vwbnt.cn/068608.Doc
gry.vwbnt.cn/606202.Doc
gry.vwbnt.cn/626626.Doc
gry.vwbnt.cn/624444.Doc
gry.vwbnt.cn/192777.Doc
gry.vwbnt.cn/880264.Doc
gry.vwbnt.cn/462400.Doc
gry.vwbnt.cn/840004.Doc
grt.vwbnt.cn/485325.Doc
grt.vwbnt.cn/228260.Doc
grt.vwbnt.cn/738257.Doc
grt.vwbnt.cn/644280.Doc
grt.vwbnt.cn/026084.Doc
grt.vwbnt.cn/488428.Doc
grt.vwbnt.cn/080182.Doc
grt.vwbnt.cn/826800.Doc
grt.vwbnt.cn/066848.Doc
grt.vwbnt.cn/026062.Doc
grr.vwbnt.cn/066246.Doc
grr.vwbnt.cn/288280.Doc
grr.vwbnt.cn/266040.Doc
grr.vwbnt.cn/620860.Doc
grr.vwbnt.cn/448880.Doc
grr.vwbnt.cn/967777.Doc
grr.vwbnt.cn/466262.Doc
grr.vwbnt.cn/460680.Doc
grr.vwbnt.cn/404141.Doc
grr.vwbnt.cn/772799.Doc
gre.vwbnt.cn/246880.Doc
gre.vwbnt.cn/466646.Doc
gre.vwbnt.cn/004608.Doc
gre.vwbnt.cn/662648.Doc
gre.vwbnt.cn/844644.Doc
gre.vwbnt.cn/569657.Doc
gre.vwbnt.cn/551788.Doc
gre.vwbnt.cn/693788.Doc
gre.vwbnt.cn/400846.Doc
gre.vwbnt.cn/822208.Doc
grw.vwbnt.cn/620282.Doc
grw.vwbnt.cn/604685.Doc
grw.vwbnt.cn/602866.Doc
grw.vwbnt.cn/860062.Doc
grw.vwbnt.cn/864800.Doc
grw.vwbnt.cn/486668.Doc
grw.vwbnt.cn/664268.Doc
grw.vwbnt.cn/083574.Doc
grw.vwbnt.cn/844066.Doc
grw.vwbnt.cn/446244.Doc
grq.vwbnt.cn/582999.Doc
grq.vwbnt.cn/204200.Doc
grq.vwbnt.cn/660666.Doc
grq.vwbnt.cn/466482.Doc
grq.vwbnt.cn/282244.Doc
grq.vwbnt.cn/646886.Doc
grq.vwbnt.cn/659048.Doc
grq.vwbnt.cn/973619.Doc
grq.vwbnt.cn/062446.Doc
grq.vwbnt.cn/602008.Doc
gem.vwbnt.cn/440613.Doc
gem.vwbnt.cn/242622.Doc
gem.vwbnt.cn/886224.Doc
gem.vwbnt.cn/842642.Doc
gem.vwbnt.cn/662420.Doc
gem.vwbnt.cn/840822.Doc
gem.vwbnt.cn/202646.Doc
gem.vwbnt.cn/552712.Doc
gem.vwbnt.cn/135416.Doc
gem.vwbnt.cn/642884.Doc
gen.vwbnt.cn/826688.Doc
gen.vwbnt.cn/220040.Doc
gen.vwbnt.cn/864266.Doc
gen.vwbnt.cn/864240.Doc
gen.vwbnt.cn/280224.Doc
gen.vwbnt.cn/659828.Doc
gen.vwbnt.cn/602282.Doc
gen.vwbnt.cn/844664.Doc
gen.vwbnt.cn/557311.Doc
gen.vwbnt.cn/646800.Doc
geb.vwbnt.cn/648600.Doc
geb.vwbnt.cn/008204.Doc
geb.vwbnt.cn/820600.Doc
geb.vwbnt.cn/955115.Doc
geb.vwbnt.cn/446484.Doc
geb.vwbnt.cn/264862.Doc
geb.vwbnt.cn/802567.Doc
geb.vwbnt.cn/222048.Doc
geb.vwbnt.cn/844404.Doc
geb.vwbnt.cn/468022.Doc
gev.vwbnt.cn/624008.Doc
gev.vwbnt.cn/260626.Doc
gev.vwbnt.cn/648240.Doc
gev.vwbnt.cn/228801.Doc
gev.vwbnt.cn/448000.Doc
gev.vwbnt.cn/882248.Doc
gev.vwbnt.cn/686486.Doc
gev.vwbnt.cn/220620.Doc
gev.vwbnt.cn/608088.Doc
gev.vwbnt.cn/884086.Doc
gec.vwbnt.cn/339175.Doc
gec.vwbnt.cn/006664.Doc
gec.vwbnt.cn/375515.Doc
gec.vwbnt.cn/466688.Doc
gec.vwbnt.cn/840820.Doc
gec.vwbnt.cn/066064.Doc
gec.vwbnt.cn/884420.Doc
gec.vwbnt.cn/282046.Doc
gec.vwbnt.cn/222448.Doc
gec.vwbnt.cn/958284.Doc
gex.vwbnt.cn/086402.Doc
gex.vwbnt.cn/008482.Doc
gex.vwbnt.cn/884882.Doc
gex.vwbnt.cn/840462.Doc
gex.vwbnt.cn/246428.Doc
gex.vwbnt.cn/793597.Doc
gex.vwbnt.cn/084066.Doc
gex.vwbnt.cn/448006.Doc
gex.vwbnt.cn/684688.Doc
gex.vwbnt.cn/496304.Doc
gez.vwbnt.cn/042088.Doc
gez.vwbnt.cn/062440.Doc
gez.vwbnt.cn/246808.Doc
gez.vwbnt.cn/022806.Doc
gez.vwbnt.cn/888462.Doc
gez.vwbnt.cn/500607.Doc
gez.vwbnt.cn/448064.Doc
gez.vwbnt.cn/311379.Doc
gez.vwbnt.cn/224604.Doc
gez.vwbnt.cn/828024.Doc
gel.vwbnt.cn/53.Doc
gel.vwbnt.cn/268686.Doc
gel.vwbnt.cn/222628.Doc
gel.vwbnt.cn/397732.Doc
gel.vwbnt.cn/399755.Doc
gel.vwbnt.cn/824488.Doc
gel.vwbnt.cn/791377.Doc
gel.vwbnt.cn/044444.Doc
gel.vwbnt.cn/400040.Doc
gel.vwbnt.cn/686402.Doc
gek.vwbnt.cn/800046.Doc
gek.vwbnt.cn/486000.Doc
gek.vwbnt.cn/226440.Doc
gek.vwbnt.cn/204622.Doc
gek.vwbnt.cn/284866.Doc
gek.vwbnt.cn/600802.Doc
gek.vwbnt.cn/743286.Doc
gek.vwbnt.cn/179379.Doc
gek.vwbnt.cn/646890.Doc
gek.vwbnt.cn/002682.Doc
gej.vwbnt.cn/824202.Doc
gej.vwbnt.cn/295265.Doc
gej.vwbnt.cn/200260.Doc
gej.vwbnt.cn/062068.Doc
gej.vwbnt.cn/648042.Doc
gej.vwbnt.cn/202642.Doc
gej.vwbnt.cn/879593.Doc
gej.vwbnt.cn/804048.Doc
gej.vwbnt.cn/428804.Doc
gej.vwbnt.cn/668880.Doc
geh.vwbnt.cn/730487.Doc
geh.vwbnt.cn/072109.Doc
geh.vwbnt.cn/824602.Doc
geh.vwbnt.cn/466620.Doc
geh.vwbnt.cn/666226.Doc
geh.vwbnt.cn/820402.Doc
geh.vwbnt.cn/466622.Doc
geh.vwbnt.cn/206062.Doc
geh.vwbnt.cn/208888.Doc
geh.vwbnt.cn/042486.Doc
geg.vwbnt.cn/262264.Doc
geg.vwbnt.cn/872873.Doc
geg.vwbnt.cn/860048.Doc
geg.vwbnt.cn/024282.Doc
geg.vwbnt.cn/882646.Doc
geg.vwbnt.cn/600608.Doc
geg.vwbnt.cn/553597.Doc
geg.vwbnt.cn/423377.Doc
geg.vwbnt.cn/028488.Doc
geg.vwbnt.cn/004824.Doc
gef.vwbnt.cn/282826.Doc
gef.vwbnt.cn/423074.Doc
gef.vwbnt.cn/337955.Doc
gef.vwbnt.cn/539999.Doc
gef.vwbnt.cn/640606.Doc
gef.vwbnt.cn/228844.Doc
gef.vwbnt.cn/620662.Doc
gef.vwbnt.cn/206088.Doc
gef.vwbnt.cn/860024.Doc
gef.vwbnt.cn/062228.Doc
ged.vwbnt.cn/824024.Doc
ged.vwbnt.cn/442868.Doc
ged.vwbnt.cn/080684.Doc
ged.vwbnt.cn/642806.Doc
ged.vwbnt.cn/042642.Doc
ged.vwbnt.cn/046282.Doc
ged.vwbnt.cn/668848.Doc
ged.vwbnt.cn/486046.Doc
ged.vwbnt.cn/860262.Doc
ged.vwbnt.cn/280222.Doc
ges.vwbnt.cn/642000.Doc
ges.vwbnt.cn/822602.Doc
ges.vwbnt.cn/606280.Doc
ges.vwbnt.cn/710002.Doc
ges.vwbnt.cn/759115.Doc
ges.vwbnt.cn/042646.Doc
ges.vwbnt.cn/868464.Doc
ges.vwbnt.cn/642408.Doc
ges.vwbnt.cn/226008.Doc
ges.vwbnt.cn/860480.Doc
gea.vwbnt.cn/648620.Doc
gea.vwbnt.cn/848648.Doc
gea.vwbnt.cn/246024.Doc
gea.vwbnt.cn/448420.Doc
gea.vwbnt.cn/579117.Doc
gea.vwbnt.cn/440480.Doc
gea.vwbnt.cn/248404.Doc
gea.vwbnt.cn/024648.Doc
gea.vwbnt.cn/480628.Doc
gea.vwbnt.cn/606808.Doc
gep.vwbnt.cn/331197.Doc
gep.vwbnt.cn/862947.Doc
gep.vwbnt.cn/977135.Doc
gep.vwbnt.cn/884402.Doc
gep.vwbnt.cn/068208.Doc
gep.vwbnt.cn/688606.Doc
gep.vwbnt.cn/486620.Doc
gep.vwbnt.cn/642264.Doc
gep.vwbnt.cn/628844.Doc
gep.vwbnt.cn/711537.Doc
geo.vwbnt.cn/400020.Doc
geo.vwbnt.cn/688626.Doc
geo.vwbnt.cn/882224.Doc
geo.vwbnt.cn/428462.Doc
geo.vwbnt.cn/062620.Doc
geo.vwbnt.cn/482282.Doc
geo.vwbnt.cn/606404.Doc
geo.vwbnt.cn/751931.Doc
geo.vwbnt.cn/644628.Doc
geo.vwbnt.cn/462604.Doc
gei.vwbnt.cn/240082.Doc
gei.vwbnt.cn/482206.Doc
gei.vwbnt.cn/406888.Doc
gei.vwbnt.cn/288088.Doc
gei.vwbnt.cn/680882.Doc
gei.vwbnt.cn/080420.Doc
gei.vwbnt.cn/915157.Doc
gei.vwbnt.cn/921500.Doc
gei.vwbnt.cn/264208.Doc
gei.vwbnt.cn/553931.Doc
geu.vwbnt.cn/064660.Doc
geu.vwbnt.cn/448608.Doc
geu.vwbnt.cn/000204.Doc
geu.vwbnt.cn/974400.Doc
geu.vwbnt.cn/662808.Doc
geu.vwbnt.cn/646442.Doc
geu.vwbnt.cn/808462.Doc
geu.vwbnt.cn/442480.Doc
geu.vwbnt.cn/860648.Doc
geu.vwbnt.cn/408466.Doc
gey.vwbnt.cn/440168.Doc
gey.vwbnt.cn/089534.Doc
gey.vwbnt.cn/464208.Doc
gey.vwbnt.cn/800606.Doc
gey.vwbnt.cn/860068.Doc
gey.vwbnt.cn/026082.Doc
gey.vwbnt.cn/408808.Doc
gey.vwbnt.cn/208064.Doc
gey.vwbnt.cn/846680.Doc
gey.vwbnt.cn/822024.Doc
get.vwbnt.cn/060244.Doc
get.vwbnt.cn/880666.Doc
get.vwbnt.cn/597193.Doc
get.vwbnt.cn/660084.Doc
get.vwbnt.cn/852105.Doc
get.vwbnt.cn/111995.Doc
get.vwbnt.cn/559599.Doc
get.vwbnt.cn/442082.Doc
get.vwbnt.cn/626280.Doc
get.vwbnt.cn/624082.Doc
ger.vwbnt.cn/488846.Doc
ger.vwbnt.cn/684848.Doc
ger.vwbnt.cn/066406.Doc
ger.vwbnt.cn/402440.Doc
ger.vwbnt.cn/378637.Doc
