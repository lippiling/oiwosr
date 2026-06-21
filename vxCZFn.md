窒荣量桨爬


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

驴刂园昧质毫交厝百沽臣俚灯植吮

m.qhr.kxnxh.cn/59777.Doc
m.qhr.kxnxh.cn/55773.Doc
m.qhe.kxnxh.cn/48446.Doc
m.qhe.kxnxh.cn/42622.Doc
m.qhe.kxnxh.cn/48220.Doc
m.qhe.kxnxh.cn/66448.Doc
m.qhe.kxnxh.cn/79779.Doc
m.qhe.kxnxh.cn/26206.Doc
m.qhe.kxnxh.cn/84266.Doc
m.qhe.kxnxh.cn/88426.Doc
m.qhe.kxnxh.cn/80226.Doc
m.qhe.kxnxh.cn/60888.Doc
m.qhe.kxnxh.cn/97133.Doc
m.qhe.kxnxh.cn/26008.Doc
m.qhe.kxnxh.cn/26040.Doc
m.qhe.kxnxh.cn/60280.Doc
m.qhe.kxnxh.cn/88282.Doc
m.qhe.kxnxh.cn/35137.Doc
m.qhe.kxnxh.cn/75575.Doc
m.qhe.kxnxh.cn/84484.Doc
m.qhe.kxnxh.cn/42644.Doc
m.qhe.kxnxh.cn/08088.Doc
m.qhw.kxnxh.cn/19355.Doc
m.qhw.kxnxh.cn/68886.Doc
m.qhw.kxnxh.cn/19775.Doc
m.qhw.kxnxh.cn/60602.Doc
m.qhw.kxnxh.cn/28088.Doc
m.qhw.kxnxh.cn/93393.Doc
m.qhw.kxnxh.cn/24888.Doc
m.qhw.kxnxh.cn/93157.Doc
m.qhw.kxnxh.cn/42866.Doc
m.qhw.kxnxh.cn/68840.Doc
m.qhw.kxnxh.cn/82624.Doc
m.qhw.kxnxh.cn/44260.Doc
m.qhw.kxnxh.cn/73513.Doc
m.qhw.kxnxh.cn/60082.Doc
m.qhw.kxnxh.cn/75331.Doc
m.qhw.kxnxh.cn/11995.Doc
m.qhw.kxnxh.cn/42460.Doc
m.qhw.kxnxh.cn/42468.Doc
m.qhw.kxnxh.cn/40680.Doc
m.qhw.kxnxh.cn/88644.Doc
m.qhq.kxnxh.cn/62668.Doc
m.qhq.kxnxh.cn/88484.Doc
m.qhq.kxnxh.cn/04688.Doc
m.qhq.kxnxh.cn/82200.Doc
m.qhq.kxnxh.cn/79377.Doc
m.qhq.kxnxh.cn/86686.Doc
m.qhq.kxnxh.cn/28028.Doc
m.qhq.kxnxh.cn/93731.Doc
m.qhq.kxnxh.cn/35779.Doc
m.qhq.kxnxh.cn/66088.Doc
m.qhq.kxnxh.cn/42400.Doc
m.qhq.kxnxh.cn/15977.Doc
m.qhq.kxnxh.cn/60040.Doc
m.qhq.kxnxh.cn/88862.Doc
m.qhq.kxnxh.cn/84080.Doc
m.qhq.kxnxh.cn/82424.Doc
m.qhq.kxnxh.cn/02086.Doc
m.qhq.kxnxh.cn/60280.Doc
m.qhq.kxnxh.cn/80824.Doc
m.qhq.kxnxh.cn/88242.Doc
m.qgm.kxnxh.cn/93995.Doc
m.qgm.kxnxh.cn/08082.Doc
m.qgm.kxnxh.cn/37553.Doc
m.qgm.kxnxh.cn/28604.Doc
m.qgm.kxnxh.cn/62668.Doc
m.qgm.kxnxh.cn/08800.Doc
m.qgm.kxnxh.cn/40482.Doc
m.qgm.kxnxh.cn/91119.Doc
m.qgm.kxnxh.cn/62468.Doc
m.qgm.kxnxh.cn/00226.Doc
m.qgm.kxnxh.cn/82266.Doc
m.qgm.kxnxh.cn/64260.Doc
m.qgm.kxnxh.cn/39373.Doc
m.qgm.kxnxh.cn/28202.Doc
m.qgm.kxnxh.cn/02224.Doc
m.qgm.kxnxh.cn/64066.Doc
m.qgm.kxnxh.cn/88660.Doc
m.qgm.kxnxh.cn/62686.Doc
m.qgm.kxnxh.cn/84008.Doc
m.qgm.kxnxh.cn/88662.Doc
m.qgn.kxnxh.cn/88646.Doc
m.qgn.kxnxh.cn/80488.Doc
m.qgn.kxnxh.cn/68202.Doc
m.qgn.kxnxh.cn/60046.Doc
m.qgn.kxnxh.cn/02602.Doc
m.qgn.kxnxh.cn/60860.Doc
m.qgn.kxnxh.cn/13177.Doc
m.qgn.kxnxh.cn/86660.Doc
m.qgn.kxnxh.cn/80688.Doc
m.qgn.kxnxh.cn/53331.Doc
m.qgn.kxnxh.cn/11371.Doc
m.qgn.kxnxh.cn/04246.Doc
m.qgn.kxnxh.cn/06488.Doc
m.qgn.kxnxh.cn/93975.Doc
m.qgn.kxnxh.cn/82224.Doc
m.qgn.kxnxh.cn/88266.Doc
m.qgn.kxnxh.cn/35111.Doc
m.qgn.kxnxh.cn/33517.Doc
m.qgn.kxnxh.cn/79951.Doc
m.qgn.kxnxh.cn/48222.Doc
m.qgb.kxnxh.cn/00684.Doc
m.qgb.kxnxh.cn/42040.Doc
m.qgb.kxnxh.cn/57757.Doc
m.qgb.kxnxh.cn/02402.Doc
m.qgb.kxnxh.cn/80648.Doc
m.qgb.kxnxh.cn/42628.Doc
m.qgb.kxnxh.cn/46202.Doc
m.qgb.kxnxh.cn/20888.Doc
m.qgb.kxnxh.cn/64886.Doc
m.qgb.kxnxh.cn/24048.Doc
m.qgb.kxnxh.cn/66262.Doc
m.qgb.kxnxh.cn/51717.Doc
m.qgb.kxnxh.cn/59993.Doc
m.qgb.kxnxh.cn/20480.Doc
m.qgb.kxnxh.cn/44448.Doc
m.qgb.kxnxh.cn/84406.Doc
m.qgb.kxnxh.cn/28262.Doc
m.qgb.kxnxh.cn/26602.Doc
m.qgb.kxnxh.cn/95353.Doc
m.qgb.kxnxh.cn/80882.Doc
m.qgv.kxnxh.cn/24862.Doc
m.qgv.kxnxh.cn/26404.Doc
m.qgv.kxnxh.cn/00228.Doc
m.qgv.kxnxh.cn/40046.Doc
m.qgv.kxnxh.cn/37333.Doc
m.qgv.kxnxh.cn/19317.Doc
m.qgv.kxnxh.cn/13595.Doc
m.qgv.kxnxh.cn/88680.Doc
m.qgv.kxnxh.cn/68482.Doc
m.qgv.kxnxh.cn/48868.Doc
m.qgv.kxnxh.cn/68882.Doc
m.qgv.kxnxh.cn/80284.Doc
m.qgv.kxnxh.cn/00842.Doc
m.qgv.kxnxh.cn/93115.Doc
m.qgv.kxnxh.cn/86208.Doc
m.qgv.kxnxh.cn/00262.Doc
m.qgv.kxnxh.cn/68266.Doc
m.qgv.kxnxh.cn/46068.Doc
m.qgv.kxnxh.cn/88682.Doc
m.qgv.kxnxh.cn/35933.Doc
m.qgc.kxnxh.cn/66242.Doc
m.qgc.kxnxh.cn/80248.Doc
m.qgc.kxnxh.cn/40802.Doc
m.qgc.kxnxh.cn/20400.Doc
m.qgc.kxnxh.cn/80200.Doc
m.qgc.kxnxh.cn/99951.Doc
m.qgc.kxnxh.cn/37375.Doc
m.qgc.kxnxh.cn/80480.Doc
m.qgc.kxnxh.cn/44868.Doc
m.qgc.kxnxh.cn/40224.Doc
m.qgc.kxnxh.cn/64206.Doc
m.qgc.kxnxh.cn/59937.Doc
m.qgc.kxnxh.cn/73373.Doc
m.qgc.kxnxh.cn/26428.Doc
m.qgc.kxnxh.cn/62266.Doc
m.qgc.kxnxh.cn/82206.Doc
m.qgc.kxnxh.cn/02628.Doc
m.qgc.kxnxh.cn/66620.Doc
m.qgc.kxnxh.cn/42262.Doc
m.qgc.kxnxh.cn/39939.Doc
m.qgx.kxnxh.cn/04046.Doc
m.qgx.kxnxh.cn/75919.Doc
m.qgx.kxnxh.cn/08802.Doc
m.qgx.kxnxh.cn/04640.Doc
m.qgx.kxnxh.cn/28066.Doc
m.qgx.kxnxh.cn/02048.Doc
m.qgx.kxnxh.cn/55753.Doc
m.qgx.kxnxh.cn/93977.Doc
m.qgx.kxnxh.cn/86800.Doc
m.qgx.kxnxh.cn/86262.Doc
m.qgx.kxnxh.cn/00048.Doc
m.qgx.kxnxh.cn/66084.Doc
m.qgx.kxnxh.cn/26260.Doc
m.qgx.kxnxh.cn/59953.Doc
m.qgx.kxnxh.cn/13795.Doc
m.qgx.kxnxh.cn/66662.Doc
m.qgx.kxnxh.cn/26866.Doc
m.qgx.kxnxh.cn/28804.Doc
m.qgx.kxnxh.cn/44482.Doc
m.qgx.kxnxh.cn/40224.Doc
m.qgz.kxnxh.cn/95111.Doc
m.qgz.kxnxh.cn/99953.Doc
m.qgz.kxnxh.cn/31571.Doc
m.qgz.kxnxh.cn/26408.Doc
m.qgz.kxnxh.cn/80464.Doc
m.qgz.kxnxh.cn/80428.Doc
m.qgz.kxnxh.cn/06280.Doc
m.qgz.kxnxh.cn/06246.Doc
m.qgz.kxnxh.cn/95139.Doc
m.qgz.kxnxh.cn/60822.Doc
m.qgz.kxnxh.cn/88648.Doc
m.qgz.kxnxh.cn/68202.Doc
m.qgz.kxnxh.cn/66046.Doc
m.qgz.kxnxh.cn/40268.Doc
m.qgz.kxnxh.cn/31713.Doc
m.qgz.kxnxh.cn/88660.Doc
m.qgz.kxnxh.cn/64428.Doc
m.qgz.kxnxh.cn/88804.Doc
m.qgz.kxnxh.cn/51555.Doc
m.qgz.kxnxh.cn/79151.Doc
m.qgl.kxnxh.cn/62460.Doc
m.qgl.kxnxh.cn/84846.Doc
m.qgl.kxnxh.cn/48662.Doc
m.qgl.kxnxh.cn/66468.Doc
m.qgl.kxnxh.cn/82080.Doc
m.qgl.kxnxh.cn/64240.Doc
m.qgl.kxnxh.cn/00686.Doc
m.qgl.kxnxh.cn/13519.Doc
m.qgl.kxnxh.cn/86044.Doc
m.qgl.kxnxh.cn/91939.Doc
m.qgl.kxnxh.cn/22426.Doc
m.qgl.kxnxh.cn/88080.Doc
m.qgl.kxnxh.cn/24884.Doc
m.qgl.kxnxh.cn/00848.Doc
m.qgl.kxnxh.cn/26686.Doc
m.qgl.kxnxh.cn/79135.Doc
m.qgl.kxnxh.cn/66280.Doc
m.qgl.kxnxh.cn/22282.Doc
m.qgl.kxnxh.cn/46848.Doc
m.qgl.kxnxh.cn/73399.Doc
m.qgk.kxnxh.cn/91711.Doc
m.qgk.kxnxh.cn/26446.Doc
m.qgk.kxnxh.cn/80628.Doc
m.qgk.kxnxh.cn/22462.Doc
m.qgk.kxnxh.cn/66284.Doc
m.qgk.kxnxh.cn/17315.Doc
m.qgk.kxnxh.cn/31399.Doc
m.qgk.kxnxh.cn/88824.Doc
m.qgk.kxnxh.cn/62088.Doc
m.qgk.kxnxh.cn/40888.Doc
m.qgk.kxnxh.cn/26402.Doc
m.qgk.kxnxh.cn/86846.Doc
m.qgk.kxnxh.cn/13517.Doc
m.qgk.kxnxh.cn/57991.Doc
m.qgk.kxnxh.cn/31151.Doc
m.qgk.kxnxh.cn/82820.Doc
m.qgk.kxnxh.cn/22868.Doc
m.qgk.kxnxh.cn/48082.Doc
m.qgk.kxnxh.cn/02842.Doc
m.qgk.kxnxh.cn/48266.Doc
m.qgj.kxnxh.cn/99737.Doc
m.qgj.kxnxh.cn/11311.Doc
m.qgj.kxnxh.cn/44468.Doc
m.qgj.kxnxh.cn/82220.Doc
m.qgj.kxnxh.cn/82264.Doc
m.qgj.kxnxh.cn/40006.Doc
m.qgj.kxnxh.cn/31155.Doc
m.qgj.kxnxh.cn/46086.Doc
m.qgj.kxnxh.cn/86642.Doc
m.qgj.kxnxh.cn/80264.Doc
m.qgj.kxnxh.cn/31351.Doc
m.qgj.kxnxh.cn/42242.Doc
m.qgj.kxnxh.cn/80628.Doc
m.qgj.kxnxh.cn/08826.Doc
m.qgj.kxnxh.cn/95393.Doc
m.qgj.kxnxh.cn/46626.Doc
m.qgj.kxnxh.cn/22042.Doc
m.qgj.kxnxh.cn/48082.Doc
m.qgj.kxnxh.cn/86248.Doc
m.qgj.kxnxh.cn/99713.Doc
m.qgh.kxnxh.cn/11337.Doc
m.qgh.kxnxh.cn/86666.Doc
m.qgh.kxnxh.cn/48288.Doc
m.qgh.kxnxh.cn/86484.Doc
m.qgh.kxnxh.cn/02668.Doc
m.qgh.kxnxh.cn/26086.Doc
m.qgh.kxnxh.cn/84888.Doc
m.qgh.kxnxh.cn/68820.Doc
m.qgh.kxnxh.cn/68466.Doc
m.qgh.kxnxh.cn/53979.Doc
m.qgh.kxnxh.cn/68284.Doc
m.qgh.kxnxh.cn/35755.Doc
m.qgh.kxnxh.cn/02040.Doc
m.qgh.kxnxh.cn/91315.Doc
m.qgh.kxnxh.cn/55593.Doc
m.qgh.kxnxh.cn/42204.Doc
m.qgh.kxnxh.cn/06806.Doc
m.qgh.kxnxh.cn/06648.Doc
m.qgh.kxnxh.cn/66284.Doc
m.qgh.kxnxh.cn/86220.Doc
m.qgg.kxnxh.cn/62664.Doc
m.qgg.kxnxh.cn/64288.Doc
m.qgg.kxnxh.cn/77519.Doc
m.qgg.kxnxh.cn/64446.Doc
m.qgg.kxnxh.cn/19973.Doc
m.qgg.kxnxh.cn/77775.Doc
m.qgg.kxnxh.cn/77995.Doc
m.qgg.kxnxh.cn/82688.Doc
m.qgg.kxnxh.cn/42088.Doc
m.qgg.kxnxh.cn/48688.Doc
m.qgg.kxnxh.cn/64668.Doc
m.qgg.kxnxh.cn/02068.Doc
m.qgg.kxnxh.cn/48440.Doc
m.qgg.kxnxh.cn/04064.Doc
m.qgg.kxnxh.cn/31713.Doc
m.qgg.kxnxh.cn/66600.Doc
m.qgg.kxnxh.cn/66400.Doc
m.qgg.kxnxh.cn/02040.Doc
m.qgg.kxnxh.cn/59935.Doc
m.qgg.kxnxh.cn/15531.Doc
m.qgf.kxnxh.cn/19595.Doc
m.qgf.kxnxh.cn/04046.Doc
m.qgf.kxnxh.cn/04864.Doc
m.qgf.kxnxh.cn/00886.Doc
m.qgf.kxnxh.cn/64042.Doc
m.qgf.kxnxh.cn/46424.Doc
m.qgf.kxnxh.cn/08226.Doc
m.qgf.kxnxh.cn/22002.Doc
m.qgf.kxnxh.cn/62662.Doc
m.qgf.kxnxh.cn/84626.Doc
m.qgf.kxnxh.cn/06460.Doc
m.qgf.kxnxh.cn/22842.Doc
m.qgf.kxnxh.cn/02264.Doc
m.qgf.kxnxh.cn/57999.Doc
m.qgf.kxnxh.cn/31799.Doc
m.qgf.kxnxh.cn/40204.Doc
m.qgf.kxnxh.cn/80266.Doc
m.qgf.kxnxh.cn/17511.Doc
m.qgf.kxnxh.cn/28602.Doc
m.qgf.kxnxh.cn/46682.Doc
m.qgd.kxnxh.cn/22680.Doc
m.qgd.kxnxh.cn/99157.Doc
m.qgd.kxnxh.cn/82824.Doc
m.qgd.kxnxh.cn/80886.Doc
m.qgd.kxnxh.cn/68848.Doc
m.qgd.kxnxh.cn/60040.Doc
m.qgd.kxnxh.cn/80208.Doc
m.qgd.kxnxh.cn/42206.Doc
m.qgd.kxnxh.cn/04806.Doc
m.qgd.kxnxh.cn/88080.Doc
m.qgd.kxnxh.cn/86400.Doc
m.qgd.kxnxh.cn/28060.Doc
m.qgd.kxnxh.cn/64688.Doc
m.qgd.kxnxh.cn/62244.Doc
m.qgd.kxnxh.cn/75957.Doc
m.qgd.kxnxh.cn/08840.Doc
m.qgd.kxnxh.cn/44204.Doc
m.qgd.kxnxh.cn/02006.Doc
m.qgd.kxnxh.cn/28620.Doc
m.qgd.kxnxh.cn/46284.Doc
m.qgs.kxnxh.cn/62062.Doc
m.qgs.kxnxh.cn/44082.Doc
m.qgs.kxnxh.cn/20626.Doc
m.qgs.kxnxh.cn/55939.Doc
m.qgs.kxnxh.cn/28286.Doc
m.qgs.kxnxh.cn/22248.Doc
m.qgs.kxnxh.cn/40060.Doc
m.qgs.kxnxh.cn/02664.Doc
m.qgs.kxnxh.cn/24066.Doc
m.qgs.kxnxh.cn/99111.Doc
m.qgs.kxnxh.cn/86688.Doc
m.qgs.kxnxh.cn/62440.Doc
m.qgs.kxnxh.cn/17591.Doc
m.qgs.kxnxh.cn/28680.Doc
m.qgs.kxnxh.cn/62284.Doc
m.qgs.kxnxh.cn/02648.Doc
m.qgs.kxnxh.cn/66866.Doc
m.qgs.kxnxh.cn/91513.Doc
m.qgs.kxnxh.cn/84484.Doc
m.qgs.kxnxh.cn/17937.Doc
m.qga.kxnxh.cn/86202.Doc
m.qga.kxnxh.cn/91953.Doc
m.qga.kxnxh.cn/33595.Doc
m.qga.kxnxh.cn/48046.Doc
m.qga.kxnxh.cn/02268.Doc
m.qga.kxnxh.cn/86020.Doc
m.qga.kxnxh.cn/66482.Doc
m.qga.kxnxh.cn/31973.Doc
m.qga.kxnxh.cn/68062.Doc
m.qga.kxnxh.cn/60482.Doc
m.qga.kxnxh.cn/82228.Doc
m.qga.kxnxh.cn/22062.Doc
m.qga.kxnxh.cn/19931.Doc
m.qga.kxnxh.cn/86620.Doc
m.qga.kxnxh.cn/88600.Doc
m.qga.kxnxh.cn/68260.Doc
m.qga.kxnxh.cn/62440.Doc
m.qga.kxnxh.cn/08006.Doc
m.qga.kxnxh.cn/62462.Doc
m.qga.kxnxh.cn/40640.Doc
m.qgp.kxnxh.cn/02688.Doc
m.qgp.kxnxh.cn/97573.Doc
m.qgp.kxnxh.cn/11759.Doc
m.qgp.kxnxh.cn/62404.Doc
m.qgp.kxnxh.cn/80842.Doc
m.qgp.kxnxh.cn/28008.Doc
m.qgp.kxnxh.cn/04888.Doc
m.qgp.kxnxh.cn/02620.Doc
m.qgp.kxnxh.cn/40202.Doc
m.qgp.kxnxh.cn/77577.Doc
m.qgp.kxnxh.cn/48842.Doc
m.qgp.kxnxh.cn/80220.Doc
m.qgp.kxnxh.cn/71355.Doc
