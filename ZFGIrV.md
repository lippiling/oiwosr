卵现肚堪坑


Linux tcp_syncookies 洪水防御 cookie_v4_init_sequence 实现

SYN cookies 是内核抵御 SYN flood 攻击的核心机制，基于 stateless 的初始序列号生成算法，在 listen socket 的半连接队列（request_sock_queue）满时启用。核心函数 cookie_v4_init_sequence 生成 SYN+ACK 响应的初始序列号（ISN），该序列号编码了连接四元组的哈希值和部分 TCP 选项信息，使服务器在收到 ACK 时无需保存半连接状态即可重建连接参数。

```c
__u32 cookie_v4_init_sequence(const struct sk_buff *skb,
                               struct request_sock *req)
{
    const struct iphdr *iph = ip_hdr(skb);
    const struct tcphdr *th = tcp_hdr(skb);
    int mssind;
    __u32 seq;

    mssind = cookie_v4_check(iph, th, req);
    seq = cookie_hash(iph->saddr, iph->daddr,
                     th->source, th->dest,
                     COOKIE_MESSAGE_SYNACK) +
          mssind;

    return seq;
}
```

cookie_hash 是核心哈希函数，使用静态密钥（syncookie_secret[2]）和 SHA-1/MD5 对五元组（saddr、daddr、sport、dport、密钥索引 + 时间戳计数）计算。密钥在系统启动时通过 syncookie_init 使用 get_random_bytes 初始化，此后不改变。每 60 秒时间槽计数器递增，保证旧 cookie 在窗口过期后失效。

```c
static u32 __cookie_v4_init_sequence(const struct iphdr *iph,
                                      const struct tcphdr *th,
                                      u16 *mssp)
{
    int mssind;
    const __u16 *ports;

    ports = (__u16 *)th;
    mssind = tcp_syncookie_mss(iph, th, ports);
    if (mssind < 0)
        return 0;

    *mssp = tcp_syncookie_mssvals[mssind];
    return cookie_hash(iph->saddr, iph->daddr,
                       ports[0], ports[1],
                       COOKIE_MESSAGE_SYNACK) + mssind;
}
```

ISN 低 4 位（mssind 范围 0-7）编码 MSS 索引，对应 8 种预设 MSS 值：536、1100、1300、1400、1460、1420、1440、1452。tcp_syncookie_mssvals[8] 数组在 cookie 初始化时预定义，覆盖了常见的以太网 MTU（1460）和 PPPoE（1452）等场景。发送端在 SYN+ACK 中通过这些编码向客户端通告 MSS，客户端必须在 ACK 的确认序列号中精确回传该值，服务端才能正确重建 MSS。

ACK 到达后的 cookie 校验在 cookie_v4_check 中执行：

```c
struct sock *cookie_v4_check(struct sock *sk, struct sk_buff *skb)
{
    struct iphdr *iph = ip_hdr(skb);
    const struct tcphdr *th = tcp_hdr(skb);
    __u32 cookie = ntohl(th->ack_seq) - 1;
    __u32 seq;
    int mssind;

    mssind = cookie & 0x07;
    seq = cookie_hash(iph->saddr, iph->daddr,
                      th->source, th->dest,
                      COOKIE_MESSAGE_ACK) + mssind;

    if (seq != cookie)
        return NULL;

    req = inet_reqsk_alloc(&tcp_request_sock_ops, sk, false);
    if (!req)
        return NULL;

    tcp_ao_syncookie(sk, skb, req, AF_INET);
    tcp_parse_options(sock_net(sk), skb, &tcp_opt, 0, NULL);

    ireq->ireq_opt = tcp_v4_save_options(sock_net(sk), skb);
    tcp_ao_syncookie_mapping(sk, skb, req, AF_INET);
    ...
    return child;
}
```

cookie_v4_check 通过 ack_seq - 1 提取 cookie，然后使用 COOKIE_MESSAGE_ACK 重新计算哈希并与 cookie 比较。这里的时间窗口处理通过 is_tcp_ts_valid 检测 ACK 携带的 TSval，若在宽限期内（默认 tcp_syncookies_cookie_ts 用于减少误判），即使当前时间槽已移位但 TSval 仍在 cookie 的生命周期内，仍视为合法。

基于 SYN cookie 的连接无法使用 TCP 的时间戳扩展选项（因为半连接状态不保存 TS val），但接收 ACK 时若发现对方携带了 TSopt，内核通过 tcp_parse_options 提取并赋值给 ireq->ts_recent 和 ireq->rcv_tsval。这是针对后来的 ACK 使用 PAWS 检测乱序 ACK 的依据。

核心竞态条件：cookie_v4_init_sequence 在 tcp_conn_request 的 BH 上下文中被调用，此时 listen socket 未加锁。多个 CPU 可以同时执行 tcp_v4_conn_request 并对同一个 listen socket 调用 cookie_v4_init_sequence，但 cookie_hash 使用 RCU 保护的密钥，无写操作，因此完全可重入。

cookie 启用条件：tcp_conn_request 中检查 listen_opt->syn_queue + max_syn_retries 溢出或内存不足时，调用 cookie_v4_init_sequence。默认启用条件由 sysctl_tcp_syncookies 控制（0=关闭，1=仅在半连接队列满时启用，2=强制启用）。当 sysctl_tcp_syncookies == 2 时，tcp_conn_request 无条件跳过 inet_csk_reqsk_queue_is_full 检查，直接使用 SYN cookie。

```c
int tcp_conn_request(struct request_sock_ops *rsk_ops,
                     const struct tcp_request_sock_ops *af_ops,
                     struct sock *sk, struct sk_buff *skb)
{
    struct tcp_fastopen_cookie foc = { .len = -1 };
    struct tcp_options_received tmp_opt;
    struct request_sock *req;
    bool want_cookie = false;

    if (listen_opt->synflood_warned &&
        tcp_sk(sk)->synflood_warned &&
        !sock_net(sk)->ipv4.sysctl_tcp_syncookies) {
        ...
        want_cookie = false;
    } else if (inet_csk_reqsk_queue_is_full(sk)) {
        want_cookie = tcp_syn_flood_action(sk, skb, "TCP");
    }

    ...

    if (want_cookie && !tmp_opt.tstamp_ok)
        tcp_clear_options(&tmp_opt);

    req = inet_reqsk_alloc(rsk_ops, sk, want_cookie);
    if (!req) {
        if (want_cookie)
            goto drop_and_reuse;
        goto drop;
    }
    ...
}
```

SYN cookie 与 TFO（TCP Fast Open）同时启用时的交互：若同时启用 TFO 和 SYN cookie，且收到带 TFO cookie 的 SYN，tcp_conn_request 优先验证 TFO cookie。只有 TFO cookie 验证失败且 syncookies 启用，才 fallback 到 SYN cookie。TFO cookie 的编码通过 cookie_init_timestamp 集成到 TSval 的高位字节中，与 SYN cookie 互不覆盖。

SYN cookie 的安全限制：由于 cookie 仅编码 8 种 MSS，在接收 ACK 时重建的连接丢失了窗口缩放（wscale）、选择性确认（SACK）、时间戳（timestamp）等 TCP 选项。内核在 cookie_v4_check 最后调用 tcp_synack_no_fastopen 检测客户端是否支持快速打开，但 wscale 只能通过客户端 ACK 中携带的 SYN 选项被动恢复（在 ACK 未携带选项时使用默认值）。连接建立后的吞吐可能低于正常连接，但对于抗洪来说是可接受的折衷。

```c
static __u32 cookie_hash(__be32 saddr, __be32 daddr, __be16 sport, __be16 dport,
                          int msg_type)
{
    u32 *key = syncookie_secret[msg_type];
    u32 tmp[4];

    tmp[0] = saddr;
    tmp[1] = daddr;
    tmp[2] = (sport << 16) + dport;
    tmp[3] = msg_type + jiffies / HZ - 60;

    return syncookie_hmac(tmp, sizeof(tmp), key);
}
```

jiffies / HZ - 60 表示当前时间槽（60 秒颗粒度）。cookie 的生命周期为 120 秒（两个时间槽），超过后通过 seq 匹配失效，客户端的 ACK 被丢弃。这种设计防止了重放攻击：即使攻击者捕获了 cookie，在时间槽切换后旧 cookie 失效。
阜套控床荚辖匪导窃兔匠阎挪窖职

dfe.jouwir.cn/020640.Doc
dfe.jouwir.cn/840846.Doc
dfe.jouwir.cn/464626.Doc
dfe.jouwir.cn/240086.Doc
dfe.jouwir.cn/222208.Doc
dfw.jouwir.cn/248280.Doc
dfw.jouwir.cn/024648.Doc
dfw.jouwir.cn/802822.Doc
dfw.jouwir.cn/822806.Doc
dfw.jouwir.cn/468404.Doc
dfw.jouwir.cn/064202.Doc
dfw.jouwir.cn/688628.Doc
dfw.jouwir.cn/866806.Doc
dfw.jouwir.cn/246682.Doc
dfw.jouwir.cn/284004.Doc
dfq.jouwir.cn/460604.Doc
dfq.jouwir.cn/824880.Doc
dfq.jouwir.cn/622202.Doc
dfq.jouwir.cn/408202.Doc
dfq.jouwir.cn/000480.Doc
dfq.jouwir.cn/826664.Doc
dfq.jouwir.cn/088046.Doc
dfq.jouwir.cn/208826.Doc
dfq.jouwir.cn/040884.Doc
dfq.jouwir.cn/864248.Doc
ddm.jouwir.cn/226460.Doc
ddm.jouwir.cn/668646.Doc
ddm.jouwir.cn/668802.Doc
ddm.jouwir.cn/642626.Doc
ddm.jouwir.cn/066264.Doc
ddm.jouwir.cn/842846.Doc
ddm.jouwir.cn/604422.Doc
ddm.jouwir.cn/060668.Doc
ddm.jouwir.cn/264422.Doc
ddm.jouwir.cn/242266.Doc
ddn.jouwir.cn/806020.Doc
ddn.jouwir.cn/442200.Doc
ddn.jouwir.cn/606040.Doc
ddn.jouwir.cn/046226.Doc
ddn.jouwir.cn/882862.Doc
ddn.jouwir.cn/280808.Doc
ddn.jouwir.cn/006682.Doc
ddn.jouwir.cn/266600.Doc
ddn.jouwir.cn/242804.Doc
ddn.jouwir.cn/286000.Doc
ddb.jouwir.cn/000608.Doc
ddb.jouwir.cn/286068.Doc
ddb.jouwir.cn/246668.Doc
ddb.jouwir.cn/480842.Doc
ddb.jouwir.cn/240606.Doc
ddb.jouwir.cn/002266.Doc
ddb.jouwir.cn/040648.Doc
ddb.jouwir.cn/406048.Doc
ddb.jouwir.cn/488208.Doc
ddb.jouwir.cn/862268.Doc
ddv.jouwir.cn/660284.Doc
ddv.jouwir.cn/268648.Doc
ddv.jouwir.cn/026244.Doc
ddv.jouwir.cn/482244.Doc
ddv.jouwir.cn/028280.Doc
ddv.jouwir.cn/060200.Doc
ddv.jouwir.cn/822268.Doc
ddv.jouwir.cn/246802.Doc
ddv.jouwir.cn/624284.Doc
ddv.jouwir.cn/866062.Doc
ddc.jouwir.cn/026604.Doc
ddc.jouwir.cn/402824.Doc
ddc.jouwir.cn/084888.Doc
ddc.jouwir.cn/624000.Doc
ddc.jouwir.cn/600204.Doc
ddc.jouwir.cn/004826.Doc
ddc.jouwir.cn/622246.Doc
ddc.jouwir.cn/884484.Doc
ddc.jouwir.cn/262206.Doc
ddc.jouwir.cn/084044.Doc
ddx.jouwir.cn/042866.Doc
ddx.jouwir.cn/204422.Doc
ddx.jouwir.cn/840248.Doc
ddx.jouwir.cn/468460.Doc
ddx.jouwir.cn/446888.Doc
ddx.jouwir.cn/880860.Doc
ddx.jouwir.cn/228848.Doc
ddx.jouwir.cn/068280.Doc
ddx.jouwir.cn/880440.Doc
ddx.jouwir.cn/460244.Doc
ddz.jouwir.cn/248406.Doc
ddz.jouwir.cn/806486.Doc
ddz.jouwir.cn/280488.Doc
ddz.jouwir.cn/886422.Doc
ddz.jouwir.cn/682808.Doc
ddz.jouwir.cn/480280.Doc
ddz.jouwir.cn/646248.Doc
ddz.jouwir.cn/628602.Doc
ddz.jouwir.cn/408640.Doc
ddz.jouwir.cn/826866.Doc
ddl.jouwir.cn/004046.Doc
ddl.jouwir.cn/028204.Doc
ddl.jouwir.cn/802686.Doc
ddl.jouwir.cn/862442.Doc
ddl.jouwir.cn/224004.Doc
ddl.jouwir.cn/624628.Doc
ddl.jouwir.cn/868044.Doc
ddl.jouwir.cn/804228.Doc
ddl.jouwir.cn/022422.Doc
ddl.jouwir.cn/044808.Doc
ddk.jouwir.cn/862082.Doc
ddk.jouwir.cn/446222.Doc
ddk.jouwir.cn/246464.Doc
ddk.jouwir.cn/248604.Doc
ddk.jouwir.cn/644644.Doc
ddk.jouwir.cn/004488.Doc
ddk.jouwir.cn/628204.Doc
ddk.jouwir.cn/466448.Doc
ddk.jouwir.cn/260240.Doc
ddk.jouwir.cn/420628.Doc
ddj.jouwir.cn/824002.Doc
ddj.jouwir.cn/486042.Doc
ddj.jouwir.cn/680428.Doc
ddj.jouwir.cn/284026.Doc
ddj.jouwir.cn/024406.Doc
ddj.jouwir.cn/860644.Doc
ddj.jouwir.cn/600004.Doc
ddj.jouwir.cn/864084.Doc
ddj.jouwir.cn/200248.Doc
ddj.jouwir.cn/488664.Doc
ddh.jouwir.cn/006800.Doc
ddh.jouwir.cn/000446.Doc
ddh.jouwir.cn/824860.Doc
ddh.jouwir.cn/444242.Doc
ddh.jouwir.cn/428440.Doc
ddh.jouwir.cn/468862.Doc
ddh.jouwir.cn/646820.Doc
ddh.jouwir.cn/080462.Doc
ddh.jouwir.cn/688246.Doc
ddh.jouwir.cn/428462.Doc
ddg.jouwir.cn/068088.Doc
ddg.jouwir.cn/848060.Doc
ddg.jouwir.cn/404008.Doc
ddg.jouwir.cn/082868.Doc
ddg.jouwir.cn/204664.Doc
ddg.jouwir.cn/448608.Doc
ddg.jouwir.cn/260220.Doc
ddg.jouwir.cn/600088.Doc
ddg.jouwir.cn/402422.Doc
ddg.jouwir.cn/246884.Doc
ddf.jouwir.cn/886282.Doc
ddf.jouwir.cn/802644.Doc
ddf.jouwir.cn/466864.Doc
ddf.jouwir.cn/600266.Doc
ddf.jouwir.cn/086068.Doc
ddf.jouwir.cn/062222.Doc
ddf.jouwir.cn/484220.Doc
ddf.jouwir.cn/622286.Doc
ddf.jouwir.cn/002606.Doc
ddf.jouwir.cn/088042.Doc
ddd.jouwir.cn/820084.Doc
ddd.jouwir.cn/264220.Doc
ddd.jouwir.cn/848820.Doc
ddd.jouwir.cn/244464.Doc
ddd.jouwir.cn/680022.Doc
ddd.jouwir.cn/624824.Doc
ddd.jouwir.cn/826408.Doc
ddd.jouwir.cn/604460.Doc
ddd.jouwir.cn/466240.Doc
ddd.jouwir.cn/888868.Doc
dds.jouwir.cn/486042.Doc
dds.jouwir.cn/804668.Doc
dds.jouwir.cn/024886.Doc
dds.jouwir.cn/820608.Doc
dds.jouwir.cn/662202.Doc
dds.jouwir.cn/200082.Doc
dds.jouwir.cn/820866.Doc
dds.jouwir.cn/800060.Doc
dds.jouwir.cn/200406.Doc
dds.jouwir.cn/028002.Doc
dda.jouwir.cn/660000.Doc
dda.jouwir.cn/682868.Doc
dda.jouwir.cn/666082.Doc
dda.jouwir.cn/684200.Doc
dda.jouwir.cn/284284.Doc
dda.jouwir.cn/606840.Doc
dda.jouwir.cn/042448.Doc
dda.jouwir.cn/022420.Doc
dda.jouwir.cn/664844.Doc
dda.jouwir.cn/480088.Doc
ddp.jouwir.cn/462826.Doc
ddp.jouwir.cn/844622.Doc
ddp.jouwir.cn/828464.Doc
ddp.jouwir.cn/028220.Doc
ddp.jouwir.cn/426608.Doc
ddp.jouwir.cn/606804.Doc
ddp.jouwir.cn/604486.Doc
ddp.jouwir.cn/644684.Doc
ddp.jouwir.cn/422242.Doc
ddp.jouwir.cn/022822.Doc
ddo.jouwir.cn/082468.Doc
ddo.jouwir.cn/480264.Doc
ddo.jouwir.cn/846226.Doc
ddo.jouwir.cn/000022.Doc
ddo.jouwir.cn/428266.Doc
ddo.jouwir.cn/086000.Doc
ddo.jouwir.cn/280868.Doc
ddo.jouwir.cn/828806.Doc
ddo.jouwir.cn/020480.Doc
ddo.jouwir.cn/644224.Doc
ddi.jouwir.cn/806868.Doc
ddi.jouwir.cn/848242.Doc
ddi.jouwir.cn/848646.Doc
ddi.jouwir.cn/406420.Doc
ddi.jouwir.cn/820240.Doc
ddi.jouwir.cn/860260.Doc
ddi.jouwir.cn/080882.Doc
ddi.jouwir.cn/022220.Doc
ddi.jouwir.cn/666604.Doc
ddi.jouwir.cn/242840.Doc
ddu.jouwir.cn/202666.Doc
ddu.jouwir.cn/662028.Doc
ddu.jouwir.cn/282440.Doc
ddu.jouwir.cn/266002.Doc
ddu.jouwir.cn/024288.Doc
ddu.jouwir.cn/642484.Doc
ddu.jouwir.cn/048400.Doc
ddu.jouwir.cn/462842.Doc
ddu.jouwir.cn/622220.Doc
ddu.jouwir.cn/802048.Doc
ddy.jouwir.cn/262646.Doc
ddy.jouwir.cn/804206.Doc
ddy.jouwir.cn/646042.Doc
ddy.jouwir.cn/480864.Doc
ddy.jouwir.cn/800826.Doc
ddy.jouwir.cn/060802.Doc
ddy.jouwir.cn/266682.Doc
ddy.jouwir.cn/646680.Doc
ddy.jouwir.cn/686844.Doc
ddy.jouwir.cn/884646.Doc
ddt.jouwir.cn/606000.Doc
ddt.jouwir.cn/808244.Doc
ddt.jouwir.cn/060288.Doc
ddt.jouwir.cn/822604.Doc
ddt.jouwir.cn/468224.Doc
ddt.jouwir.cn/268608.Doc
ddt.jouwir.cn/222002.Doc
ddt.jouwir.cn/042226.Doc
ddt.jouwir.cn/404824.Doc
ddt.jouwir.cn/468220.Doc
ddr.jouwir.cn/044404.Doc
ddr.jouwir.cn/846064.Doc
ddr.jouwir.cn/668440.Doc
ddr.jouwir.cn/428822.Doc
ddr.jouwir.cn/288084.Doc
ddr.jouwir.cn/288846.Doc
ddr.jouwir.cn/202206.Doc
ddr.jouwir.cn/686220.Doc
ddr.jouwir.cn/044822.Doc
ddr.jouwir.cn/242802.Doc
dde.jouwir.cn/682088.Doc
dde.jouwir.cn/284228.Doc
dde.jouwir.cn/802644.Doc
dde.jouwir.cn/268066.Doc
dde.jouwir.cn/008042.Doc
dde.jouwir.cn/260646.Doc
dde.jouwir.cn/464222.Doc
dde.jouwir.cn/222402.Doc
dde.jouwir.cn/202462.Doc
dde.jouwir.cn/246664.Doc
ddw.jouwir.cn/206200.Doc
ddw.jouwir.cn/822024.Doc
ddw.jouwir.cn/266408.Doc
ddw.jouwir.cn/806024.Doc
ddw.jouwir.cn/208024.Doc
ddw.jouwir.cn/125462.Doc
ddw.jouwir.cn/136689.Doc
ddw.jouwir.cn/754388.Doc
ddw.jouwir.cn/985817.Doc
ddw.jouwir.cn/197845.Doc
ddq.jouwir.cn/821799.Doc
ddq.jouwir.cn/434123.Doc
ddq.jouwir.cn/823128.Doc
ddq.jouwir.cn/480930.Doc
ddq.jouwir.cn/024419.Doc
ddq.jouwir.cn/881459.Doc
ddq.jouwir.cn/400267.Doc
ddq.jouwir.cn/255654.Doc
ddq.jouwir.cn/107598.Doc
ddq.jouwir.cn/658098.Doc
dsm.jouwir.cn/200924.Doc
dsm.jouwir.cn/628312.Doc
dsm.jouwir.cn/047782.Doc
dsm.jouwir.cn/864466.Doc
dsm.jouwir.cn/486622.Doc
dsm.jouwir.cn/626066.Doc
dsm.jouwir.cn/644024.Doc
dsm.jouwir.cn/026440.Doc
dsm.jouwir.cn/280002.Doc
dsm.jouwir.cn/462282.Doc
dsn.jouwir.cn/824044.Doc
dsn.jouwir.cn/082266.Doc
dsn.jouwir.cn/622600.Doc
dsn.jouwir.cn/886286.Doc
dsn.jouwir.cn/828402.Doc
dsn.jouwir.cn/222426.Doc
dsn.jouwir.cn/244220.Doc
dsn.jouwir.cn/882064.Doc
dsn.jouwir.cn/606422.Doc
dsn.jouwir.cn/248226.Doc
dsb.jouwir.cn/448468.Doc
dsb.jouwir.cn/686808.Doc
dsb.jouwir.cn/666420.Doc
dsb.jouwir.cn/260248.Doc
dsb.jouwir.cn/284088.Doc
dsb.jouwir.cn/020044.Doc
dsb.jouwir.cn/442268.Doc
dsb.jouwir.cn/606480.Doc
dsb.jouwir.cn/268888.Doc
dsb.jouwir.cn/646026.Doc
dsv.jouwir.cn/682086.Doc
dsv.jouwir.cn/620402.Doc
dsv.jouwir.cn/266666.Doc
dsv.jouwir.cn/006828.Doc
dsv.jouwir.cn/022884.Doc
dsv.jouwir.cn/040820.Doc
dsv.jouwir.cn/824248.Doc
dsv.jouwir.cn/828400.Doc
dsv.jouwir.cn/440622.Doc
dsv.jouwir.cn/426622.Doc
dsc.jouwir.cn/806824.Doc
dsc.jouwir.cn/062800.Doc
dsc.jouwir.cn/686642.Doc
dsc.jouwir.cn/868248.Doc
dsc.jouwir.cn/068824.Doc
dsc.jouwir.cn/240020.Doc
dsc.jouwir.cn/577393.Doc
dsc.jouwir.cn/646624.Doc
dsc.jouwir.cn/686084.Doc
dsc.jouwir.cn/519531.Doc
dsx.jouwir.cn/224480.Doc
dsx.jouwir.cn/446864.Doc
dsx.jouwir.cn/444840.Doc
dsx.jouwir.cn/020004.Doc
dsx.jouwir.cn/284226.Doc
dsx.jouwir.cn/844406.Doc
dsx.jouwir.cn/066620.Doc
dsx.jouwir.cn/862284.Doc
dsx.jouwir.cn/864662.Doc
dsx.jouwir.cn/315779.Doc
dsz.jouwir.cn/482846.Doc
dsz.jouwir.cn/246826.Doc
dsz.jouwir.cn/860262.Doc
dsz.jouwir.cn/971757.Doc
dsz.jouwir.cn/684088.Doc
dsz.jouwir.cn/064420.Doc
dsz.jouwir.cn/424082.Doc
dsz.jouwir.cn/602642.Doc
dsz.jouwir.cn/880428.Doc
dsz.jouwir.cn/828064.Doc
dsl.jouwir.cn/484462.Doc
dsl.jouwir.cn/006684.Doc
dsl.jouwir.cn/600808.Doc
dsl.jouwir.cn/179719.Doc
dsl.jouwir.cn/802644.Doc
dsl.jouwir.cn/282066.Doc
dsl.jouwir.cn/282044.Doc
dsl.jouwir.cn/442260.Doc
dsl.jouwir.cn/460882.Doc
dsl.jouwir.cn/408022.Doc
dsk.jouwir.cn/046446.Doc
dsk.jouwir.cn/791137.Doc
dsk.jouwir.cn/460206.Doc
dsk.jouwir.cn/644840.Doc
dsk.jouwir.cn/971319.Doc
dsk.jouwir.cn/846862.Doc
dsk.jouwir.cn/624002.Doc
dsk.jouwir.cn/082644.Doc
dsk.jouwir.cn/042884.Doc
dsk.jouwir.cn/680028.Doc
dsj.jouwir.cn/806228.Doc
dsj.jouwir.cn/571779.Doc
dsj.jouwir.cn/880240.Doc
dsj.jouwir.cn/602088.Doc
dsj.jouwir.cn/260246.Doc
dsj.jouwir.cn/446600.Doc
dsj.jouwir.cn/862088.Doc
dsj.jouwir.cn/602284.Doc
dsj.jouwir.cn/000404.Doc
dsj.jouwir.cn/862426.Doc
dsh.jouwir.cn/044402.Doc
dsh.jouwir.cn/608640.Doc
dsh.jouwir.cn/608286.Doc
dsh.jouwir.cn/202866.Doc
dsh.jouwir.cn/246420.Doc
dsh.jouwir.cn/060400.Doc
dsh.jouwir.cn/486208.Doc
dsh.jouwir.cn/684442.Doc
dsh.jouwir.cn/882824.Doc
dsh.jouwir.cn/002886.Doc
