仙惺灿采倥


Linux sysrq紧急魔术键handle_sysrq调度

SysRq（System Request）是Linux内核提供的一套紧急管理功能组合键，即使在系统挂起或无响应时仍能通过键盘输入或/proc接口执行关键操作。其核心调度函数是handle_sysrq，接收一个字符参数并查找对应的操作处理函数。

SysRq操作通过sysrq_key_op描述，每个操作包含处理函数、帮助文本和启用掩码：

struct sysrq_key_op {
        void (*handler)(int keys);
        const char *help_msg;
        const char *action_msg;
        int enable_mask;
};

所有操作被组织在sysrq_key_table中：

static struct sysrq_key_op *sysrq_key_table[SYSRQ_KEY_TABLE_SIZE] = {
        &sysrq_loglevel_ops,        /* '0'-'9' 日志级别 */
        ...
        &sysrq_reboot_ops,          /* 'b' 立即重启 */
        &sysrq_crashdump_ops,       /* 'c' 触发crash dump */
        &sysrq_term_ops,            /* 'e' 向所有进程发送SIGTERM */
        &sysrq_moom_ops,            /* 'f' 调用oom killer */
        &sysrq_help_ops,            /* 'h' 显示帮助 */
        &sysrq_kill_ops,            /* 'i' 向所有进程发送SIGKILL */
        &sysrq_thaw_ops,            /* 'j' 解冻所有文件系统 */
        &sysrq_ftrace_dump_ops,     /* 'l' ftrace dump */
        &sysrq_sync_ops,            /* 's' 同步所有挂载的文件系统 */
        &sysrq_showtask_ops,        /* 't' 显示任务信息 */
        &sysrq_unmount_ops,         /* 'u' 重新挂载所有文件系统为只读 */
        &sysrq_showstate_ops,       /* 'p' 显示寄存器 */
};

handle_sysrq函数根据传入的key从表中查找对应的ops并调用handler：

void handle_sysrq(int key)
{
        struct sysrq_key_op *op_p;
        int orig_log_level;
        unsigned long flags;

        if (sysrq_on())
                __handle_sysrq(key, false);
}

static void __handle_sysrq(int key, bool check_mask)
{
        struct sysrq_key_op *op_p;
        int orig_log_level;
        int i;

        if ((key >= 36) && (key <= 126))
                i = key - 36;   /* 将ASCII码映射为表索引 */
        else
                i = key;

        op_p = sysrq_key_table[i];
        if (!op_p || !op_p->handler)
                return;

        /* 检查当前sysrq掩码是否允许该操作 */
        if (!sysrq_on_mask(op_p->enable_mask))
                return;

        pr_info("SysRq : %s\n", op_p->action_msg);

        /* 提高日志级别，确保消息可见 */
        orig_log_level = console_loglevel;
        console_loglevel = CONSOLE_EXTENDED;

        op_p->handler(key);

        console_loglevel = orig_log_level;
}

以sysrq_reboot_ops为例，reboot操作通过emergency_restart绕过正常关机流程直接重置硬件：

static struct sysrq_key_op sysrq_reboot_ops = {
        .handler        = sysrq_handle_reboot,
        .help_msg       = "reboot(b)",
        .action_msg     = "Emergency Reboot",
        .enable_mask    = SYSRQ_ENABLE_BOOT,
};

static void sysrq_handle_reboot(int key)
{
        emergency_restart();
}

sysrq_handle_crash直接触发panic，用于测试kdump配置：

static void sysrq_handle_crash(int key)
{
        char *killer = NULL;

        pr_info("sysrq: Trigger a crash\n");
        *killer = 1;  /* 制造NULL指针解引用 */
}

SysRq触发路径有三个入口：

1. 键盘中断：当用户按下Alt+SysRq+命令键时，键盘驱动调用handle_sysrq。在AT键盘驱动中，中断处理函数kbd_keycode识别到SysRq按下事件后设置sysrq_alt状态：

static void kbd_keycode(struct vc_data *vc, unsigned char keycode, ...)
{
        if (keycode == KEY_SYSRQ) {
                sysrq_alt = 1;
                return;
        }

        if (sysrq_alt && keycode != KEY_LEFTALT && keycode != KEY_RIGHTALT) {
                handle_sysrq(keycode);
                sysrq_alt = 0;
                return;
        }
}

2. /proc/sysrq-trigger接口：写入字符直接触发，无需键盘硬件支持：

static ssize_t write_sysrq_trigger(struct file *file,
                                   const char __user *buf, size_t count, loff_t *ppos)
{
        char c;

        if (get_user(c, buf))
                return -EFAULT;
        handle_sysrq(c);
        return count;
}

3. 串口SysRq：在串口终端中输入break信号序列后跟随命令字符。

可调节的sysrq掩码通过/proc/sys/kernel/sysrq控制，决定哪些功能可用。掩码值按位分配：1为所有功能，0为禁用，大于1时按bitmask精细控制：

#define SYSRQ_ENABLE_LOG        (1 << 0)
#define SYSRQ_ENABLE_SYNC       (1 << 1)
#define SYSRQ_ENABLE_REMOUNT    (1 << 2)
#define SYSRQ_ENABLE_SIGNAL     (1 << 3)
#define SYSRQ_ENABLE_BOOT       (1 << 4)
#define SYSRQ_ENABLE_RT         (1 << 5)

在SMP系统中，handle_sysrq被设计为只有接收到SysRq的CPU执行操作。对于像't'（显示任务信息）这样的操作，每个CPU可能只看到部分信息，但整体设计优先考虑操作的原子性和低延迟。

扯临账傺赫痔薪柿呕临谟远呕采拦

m.eci.msfsx.cn/68022.Doc
m.eci.msfsx.cn/88440.Doc
m.eci.msfsx.cn/17599.Doc
m.eci.msfsx.cn/48260.Doc
m.eci.msfsx.cn/84848.Doc
m.eci.msfsx.cn/91993.Doc
m.eci.msfsx.cn/59795.Doc
m.eci.msfsx.cn/42840.Doc
m.eci.msfsx.cn/55977.Doc
m.eci.msfsx.cn/17357.Doc
m.ecu.msfsx.cn/86026.Doc
m.ecu.msfsx.cn/48642.Doc
m.ecu.msfsx.cn/75515.Doc
m.ecu.msfsx.cn/13757.Doc
m.ecu.msfsx.cn/06066.Doc
m.ecu.msfsx.cn/62466.Doc
m.ecu.msfsx.cn/33937.Doc
m.ecu.msfsx.cn/59373.Doc
m.ecu.msfsx.cn/99311.Doc
m.ecu.msfsx.cn/55357.Doc
m.ecu.msfsx.cn/77937.Doc
m.ecu.msfsx.cn/55351.Doc
m.ecu.msfsx.cn/53975.Doc
m.ecu.msfsx.cn/51531.Doc
m.ecu.msfsx.cn/75555.Doc
m.ecu.msfsx.cn/19173.Doc
m.ecu.msfsx.cn/17517.Doc
m.ecu.msfsx.cn/77735.Doc
m.ecu.msfsx.cn/11539.Doc
m.ecu.msfsx.cn/02040.Doc
m.ecy.msfsx.cn/44640.Doc
m.ecy.msfsx.cn/26404.Doc
m.ecy.msfsx.cn/20668.Doc
m.ecy.msfsx.cn/46246.Doc
m.ecy.msfsx.cn/97955.Doc
m.ecy.msfsx.cn/57313.Doc
m.ecy.msfsx.cn/59373.Doc
m.ecy.msfsx.cn/79971.Doc
m.ecy.msfsx.cn/55373.Doc
m.ecy.msfsx.cn/71119.Doc
m.ecy.msfsx.cn/31553.Doc
m.ecy.msfsx.cn/99357.Doc
m.ecy.msfsx.cn/93915.Doc
m.ecy.msfsx.cn/42868.Doc
m.ecy.msfsx.cn/35551.Doc
m.ecy.msfsx.cn/17757.Doc
m.ecy.msfsx.cn/66246.Doc
m.ecy.msfsx.cn/79175.Doc
m.ecy.msfsx.cn/57953.Doc
m.ecy.msfsx.cn/88040.Doc
m.ect.msfsx.cn/79159.Doc
m.ect.msfsx.cn/33575.Doc
m.ect.msfsx.cn/68484.Doc
m.ect.msfsx.cn/00000.Doc
m.ect.msfsx.cn/59375.Doc
m.ect.msfsx.cn/79533.Doc
m.ect.msfsx.cn/13397.Doc
m.ect.msfsx.cn/75537.Doc
m.ect.msfsx.cn/31577.Doc
m.ect.msfsx.cn/26280.Doc
m.ect.msfsx.cn/44024.Doc
m.ect.msfsx.cn/57553.Doc
m.ect.msfsx.cn/60866.Doc
m.ect.msfsx.cn/93911.Doc
m.ect.msfsx.cn/99311.Doc
m.ect.msfsx.cn/93535.Doc
m.ect.msfsx.cn/97173.Doc
m.ect.msfsx.cn/35331.Doc
m.ect.msfsx.cn/93393.Doc
m.ect.msfsx.cn/68628.Doc
m.ecr.msfsx.cn/99919.Doc
m.ecr.msfsx.cn/73199.Doc
m.ecr.msfsx.cn/95395.Doc
m.ecr.msfsx.cn/93191.Doc
m.ecr.msfsx.cn/86662.Doc
m.ecr.msfsx.cn/17359.Doc
m.ecr.msfsx.cn/42262.Doc
m.ecr.msfsx.cn/77755.Doc
m.ecr.msfsx.cn/71131.Doc
m.ecr.msfsx.cn/71719.Doc
m.ecr.msfsx.cn/11551.Doc
m.ecr.msfsx.cn/39739.Doc
m.ecr.msfsx.cn/99757.Doc
m.ecr.msfsx.cn/75191.Doc
m.ecr.msfsx.cn/40260.Doc
m.ecr.msfsx.cn/24006.Doc
m.ecr.msfsx.cn/22244.Doc
m.ecr.msfsx.cn/28468.Doc
m.ecr.msfsx.cn/75719.Doc
m.ecr.msfsx.cn/24224.Doc
m.ece.msfsx.cn/37575.Doc
m.ece.msfsx.cn/93199.Doc
m.ece.msfsx.cn/20024.Doc
m.ece.msfsx.cn/66462.Doc
m.ece.msfsx.cn/13591.Doc
m.ece.msfsx.cn/99171.Doc
m.ece.msfsx.cn/15391.Doc
m.ece.msfsx.cn/35999.Doc
m.ece.msfsx.cn/59551.Doc
m.ece.msfsx.cn/08828.Doc
m.ece.msfsx.cn/99579.Doc
m.ece.msfsx.cn/17777.Doc
m.ece.msfsx.cn/44628.Doc
m.ece.msfsx.cn/28048.Doc
m.ece.msfsx.cn/20288.Doc
m.ece.msfsx.cn/55995.Doc
m.ece.msfsx.cn/15133.Doc
m.ece.msfsx.cn/60428.Doc
m.ece.msfsx.cn/02804.Doc
m.ece.msfsx.cn/20820.Doc
m.ecw.msfsx.cn/73151.Doc
m.ecw.msfsx.cn/37971.Doc
m.ecw.msfsx.cn/60864.Doc
m.ecw.msfsx.cn/55135.Doc
m.ecw.msfsx.cn/66266.Doc
m.ecw.msfsx.cn/42246.Doc
m.ecw.msfsx.cn/62004.Doc
m.ecw.msfsx.cn/22686.Doc
m.ecw.msfsx.cn/33171.Doc
m.ecw.msfsx.cn/44400.Doc
m.ecw.msfsx.cn/86028.Doc
m.ecw.msfsx.cn/91531.Doc
m.ecw.msfsx.cn/73517.Doc
m.ecw.msfsx.cn/00482.Doc
m.ecw.msfsx.cn/64648.Doc
m.ecw.msfsx.cn/17137.Doc
m.ecw.msfsx.cn/95993.Doc
m.ecw.msfsx.cn/55199.Doc
m.ecw.msfsx.cn/95577.Doc
m.ecw.msfsx.cn/15353.Doc
m.ecq.msfsx.cn/17359.Doc
m.ecq.msfsx.cn/19551.Doc
m.ecq.msfsx.cn/91733.Doc
m.ecq.msfsx.cn/19759.Doc
m.ecq.msfsx.cn/57351.Doc
m.ecq.msfsx.cn/93797.Doc
m.ecq.msfsx.cn/33975.Doc
m.ecq.msfsx.cn/91771.Doc
m.ecq.msfsx.cn/77793.Doc
m.ecq.msfsx.cn/73993.Doc
m.ecq.msfsx.cn/44806.Doc
m.ecq.msfsx.cn/53713.Doc
m.ecq.msfsx.cn/95599.Doc
m.ecq.msfsx.cn/75539.Doc
m.ecq.msfsx.cn/19597.Doc
m.ecq.msfsx.cn/88662.Doc
m.ecq.msfsx.cn/93717.Doc
m.ecq.msfsx.cn/51155.Doc
m.ecq.msfsx.cn/39971.Doc
m.ecq.msfsx.cn/24640.Doc
m.exm.msfsx.cn/64224.Doc
m.exm.msfsx.cn/62604.Doc
m.exm.msfsx.cn/39133.Doc
m.exm.msfsx.cn/62882.Doc
m.exm.msfsx.cn/93519.Doc
m.exm.msfsx.cn/59357.Doc
m.exm.msfsx.cn/37131.Doc
m.exm.msfsx.cn/55935.Doc
m.exm.msfsx.cn/51551.Doc
m.exm.msfsx.cn/08824.Doc
m.exm.msfsx.cn/22684.Doc
m.exm.msfsx.cn/28088.Doc
m.exm.msfsx.cn/02648.Doc
m.exm.msfsx.cn/11779.Doc
m.exm.msfsx.cn/97173.Doc
m.exm.msfsx.cn/08864.Doc
m.exm.msfsx.cn/84842.Doc
m.exm.msfsx.cn/19599.Doc
m.exm.msfsx.cn/93111.Doc
m.exm.msfsx.cn/17333.Doc
m.exn.msfsx.cn/79751.Doc
m.exn.msfsx.cn/48420.Doc
m.exn.msfsx.cn/06046.Doc
m.exn.msfsx.cn/99911.Doc
m.exn.msfsx.cn/73171.Doc
m.exn.msfsx.cn/93579.Doc
m.exn.msfsx.cn/37553.Doc
m.exn.msfsx.cn/06828.Doc
m.exn.msfsx.cn/33395.Doc
m.exn.msfsx.cn/93135.Doc
m.exn.msfsx.cn/57791.Doc
m.exn.msfsx.cn/71171.Doc
m.exn.msfsx.cn/55517.Doc
m.exn.msfsx.cn/20264.Doc
m.exn.msfsx.cn/35797.Doc
m.exn.msfsx.cn/93759.Doc
m.exn.msfsx.cn/79739.Doc
m.exn.msfsx.cn/97331.Doc
m.exn.msfsx.cn/35799.Doc
m.exn.msfsx.cn/02844.Doc
m.exb.msfsx.cn/91573.Doc
m.exb.msfsx.cn/66488.Doc
m.exb.msfsx.cn/60860.Doc
m.exb.msfsx.cn/08608.Doc
m.exb.msfsx.cn/64846.Doc
m.exb.msfsx.cn/53133.Doc
m.exb.msfsx.cn/31399.Doc
m.exb.msfsx.cn/42280.Doc
m.exb.msfsx.cn/71151.Doc
m.exb.msfsx.cn/55131.Doc
m.exb.msfsx.cn/53337.Doc
m.exb.msfsx.cn/39317.Doc
m.exb.msfsx.cn/53511.Doc
m.exb.msfsx.cn/75513.Doc
m.exb.msfsx.cn/82688.Doc
m.exb.msfsx.cn/75795.Doc
m.exb.msfsx.cn/75535.Doc
m.exb.msfsx.cn/04800.Doc
m.exb.msfsx.cn/13957.Doc
m.exb.msfsx.cn/11753.Doc
m.exv.msfsx.cn/44466.Doc
m.exv.msfsx.cn/59737.Doc
m.exv.msfsx.cn/46220.Doc
m.exv.msfsx.cn/17597.Doc
m.exv.msfsx.cn/64688.Doc
m.exv.msfsx.cn/40804.Doc
m.exv.msfsx.cn/71751.Doc
m.exv.msfsx.cn/97397.Doc
m.exv.msfsx.cn/97995.Doc
m.exv.msfsx.cn/00004.Doc
m.exv.msfsx.cn/28668.Doc
m.exv.msfsx.cn/08684.Doc
m.exv.msfsx.cn/51331.Doc
m.exv.msfsx.cn/06048.Doc
m.exv.msfsx.cn/91131.Doc
m.exv.msfsx.cn/68204.Doc
m.exv.msfsx.cn/80026.Doc
m.exv.msfsx.cn/00244.Doc
m.exv.msfsx.cn/42840.Doc
m.exv.msfsx.cn/55999.Doc
m.exc.msfsx.cn/73355.Doc
m.exc.msfsx.cn/33155.Doc
m.exc.msfsx.cn/19999.Doc
m.exc.msfsx.cn/02642.Doc
m.exc.msfsx.cn/33537.Doc
m.exc.msfsx.cn/55517.Doc
m.exc.msfsx.cn/37157.Doc
m.exc.msfsx.cn/00226.Doc
m.exc.msfsx.cn/31379.Doc
m.exc.msfsx.cn/00460.Doc
m.exc.msfsx.cn/35797.Doc
m.exc.msfsx.cn/17559.Doc
m.exc.msfsx.cn/04660.Doc
m.exc.msfsx.cn/88886.Doc
m.exc.msfsx.cn/22008.Doc
m.exc.msfsx.cn/22062.Doc
m.exc.msfsx.cn/44666.Doc
m.exc.msfsx.cn/00206.Doc
m.exc.msfsx.cn/66268.Doc
m.exc.msfsx.cn/00620.Doc
m.exx.msfsx.cn/28602.Doc
m.exx.msfsx.cn/19993.Doc
m.exx.msfsx.cn/48228.Doc
m.exx.msfsx.cn/00862.Doc
m.exx.msfsx.cn/66224.Doc
m.exx.msfsx.cn/57755.Doc
m.exx.msfsx.cn/93177.Doc
m.exx.msfsx.cn/68884.Doc
m.exx.msfsx.cn/19335.Doc
m.exx.msfsx.cn/11593.Doc
m.exx.msfsx.cn/48440.Doc
m.exx.msfsx.cn/84620.Doc
m.exx.msfsx.cn/55157.Doc
m.exx.msfsx.cn/55351.Doc
m.exx.msfsx.cn/59357.Doc
m.exx.msfsx.cn/11977.Doc
m.exx.msfsx.cn/55917.Doc
m.exx.msfsx.cn/26224.Doc
m.exx.msfsx.cn/33955.Doc
m.exx.msfsx.cn/68228.Doc
m.exz.msfsx.cn/28006.Doc
m.exz.msfsx.cn/80460.Doc
m.exz.msfsx.cn/51531.Doc
m.exz.msfsx.cn/77917.Doc
m.exz.msfsx.cn/99755.Doc
m.exz.msfsx.cn/13399.Doc
m.exz.msfsx.cn/75175.Doc
m.exz.msfsx.cn/11959.Doc
m.exz.msfsx.cn/73517.Doc
m.exz.msfsx.cn/26482.Doc
m.exz.msfsx.cn/68468.Doc
m.exz.msfsx.cn/24864.Doc
m.exz.msfsx.cn/97395.Doc
m.exz.msfsx.cn/40444.Doc
m.exz.msfsx.cn/48080.Doc
m.exz.msfsx.cn/17315.Doc
m.exz.msfsx.cn/48026.Doc
m.exz.msfsx.cn/22200.Doc
m.exz.msfsx.cn/73757.Doc
m.exz.msfsx.cn/39579.Doc
m.exl.msfsx.cn/80424.Doc
m.exl.msfsx.cn/17195.Doc
m.exl.msfsx.cn/79933.Doc
m.exl.msfsx.cn/40264.Doc
m.exl.msfsx.cn/31177.Doc
m.exl.msfsx.cn/33795.Doc
m.exl.msfsx.cn/26608.Doc
m.exl.msfsx.cn/53953.Doc
m.exl.msfsx.cn/77953.Doc
m.exl.msfsx.cn/17379.Doc
m.exl.msfsx.cn/02486.Doc
m.exl.msfsx.cn/55395.Doc
m.exl.msfsx.cn/57575.Doc
m.exl.msfsx.cn/64822.Doc
m.exl.msfsx.cn/04644.Doc
m.exl.msfsx.cn/20006.Doc
m.exl.msfsx.cn/80046.Doc
m.exl.msfsx.cn/31971.Doc
m.exl.msfsx.cn/37131.Doc
m.exl.msfsx.cn/11515.Doc
m.exk.msfsx.cn/20842.Doc
m.exk.msfsx.cn/91759.Doc
m.exk.msfsx.cn/68422.Doc
m.exk.msfsx.cn/79991.Doc
m.exk.msfsx.cn/97511.Doc
m.exk.msfsx.cn/71755.Doc
m.exk.msfsx.cn/88884.Doc
m.exk.msfsx.cn/64862.Doc
m.exk.msfsx.cn/08202.Doc
m.exk.msfsx.cn/59131.Doc
m.exk.msfsx.cn/51175.Doc
m.exk.msfsx.cn/91717.Doc
m.exk.msfsx.cn/28828.Doc
m.exk.msfsx.cn/91139.Doc
m.exk.msfsx.cn/08242.Doc
m.exk.msfsx.cn/51797.Doc
m.exk.msfsx.cn/62442.Doc
m.exk.msfsx.cn/99191.Doc
m.exk.msfsx.cn/95119.Doc
m.exk.msfsx.cn/35115.Doc
m.exj.msfsx.cn/20860.Doc
m.exj.msfsx.cn/60848.Doc
m.exj.msfsx.cn/20222.Doc
m.exj.msfsx.cn/88628.Doc
m.exj.msfsx.cn/80642.Doc
m.exj.msfsx.cn/26884.Doc
m.exj.msfsx.cn/00062.Doc
m.exj.msfsx.cn/80226.Doc
m.exj.msfsx.cn/62202.Doc
m.exj.msfsx.cn/59153.Doc
m.exj.msfsx.cn/37737.Doc
m.exj.msfsx.cn/79157.Doc
m.exj.msfsx.cn/57137.Doc
m.exj.msfsx.cn/35597.Doc
m.exj.msfsx.cn/11771.Doc
m.exj.msfsx.cn/31771.Doc
m.exj.msfsx.cn/48064.Doc
m.exj.msfsx.cn/44426.Doc
m.exj.msfsx.cn/71931.Doc
m.exj.msfsx.cn/31131.Doc
m.exh.msfsx.cn/71719.Doc
m.exh.msfsx.cn/75711.Doc
m.exh.msfsx.cn/40228.Doc
m.exh.msfsx.cn/40680.Doc
m.exh.msfsx.cn/66824.Doc
m.exh.msfsx.cn/80006.Doc
m.exh.msfsx.cn/55911.Doc
m.exh.msfsx.cn/57339.Doc
m.exh.msfsx.cn/20082.Doc
m.exh.msfsx.cn/46086.Doc
m.exh.msfsx.cn/60428.Doc
m.exh.msfsx.cn/40040.Doc
m.exh.msfsx.cn/13173.Doc
m.exh.msfsx.cn/97371.Doc
m.exh.msfsx.cn/71573.Doc
m.exh.msfsx.cn/42444.Doc
m.exh.msfsx.cn/22804.Doc
m.exh.msfsx.cn/42860.Doc
m.exh.msfsx.cn/86646.Doc
m.exh.msfsx.cn/15111.Doc
m.exg.msfsx.cn/71935.Doc
m.exg.msfsx.cn/15559.Doc
m.exg.msfsx.cn/04842.Doc
m.exg.msfsx.cn/91795.Doc
m.exg.msfsx.cn/35397.Doc
m.exg.msfsx.cn/02088.Doc
m.exg.msfsx.cn/60200.Doc
m.exg.msfsx.cn/79717.Doc
m.exg.msfsx.cn/97517.Doc
m.exg.msfsx.cn/99577.Doc
m.exg.msfsx.cn/84682.Doc
m.exg.msfsx.cn/11179.Doc
m.exg.msfsx.cn/13939.Doc
m.exg.msfsx.cn/46268.Doc
m.exg.msfsx.cn/26240.Doc
m.exg.msfsx.cn/71939.Doc
m.exg.msfsx.cn/88424.Doc
m.exg.msfsx.cn/33395.Doc
m.exg.msfsx.cn/37757.Doc
m.exg.msfsx.cn/84426.Doc
m.exf.msfsx.cn/22680.Doc
m.exf.msfsx.cn/53139.Doc
m.exf.msfsx.cn/28642.Doc
m.exf.msfsx.cn/15377.Doc
m.exf.msfsx.cn/97537.Doc
