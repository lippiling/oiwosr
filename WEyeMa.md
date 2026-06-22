缴垦巳蚀沉


Linux /proc/modules文件格式与m_show回调

/proc/modules是Linux内核暴露已加载模块信息的传统procfs接口，其文件格式和输出由seq_file接口中的m_show()回调完全控制。尽管/sys/module提供了更丰富的sysfs接口，/proc/modules因其简洁的文本格式仍然是lsmod等用户空间工具的首选数据源。

/proc/modules的注册位于kernel/module/procfs.c中的__init proc_modules_init()：

static int __init proc_modules_init(void)
{
    proc_create_seq("modules", 0, NULL, &modules_op);
    return 0;
}
module_init(proc_modules_init);

proc_create_seq()是内核提供的seq_file封装，它在procfs根目录下创建名为"modules"的文件，并将所有读写操作委托给modules_op中定义的seq_operations回调。

struct seq_operations modules_op的定义：

static const struct seq_operations modules_op = {
    .start  = m_start,
    .next   = m_next,
    .stop   = m_stop,
    .show   = m_show,
};

/proc/modules的典型输出格式如下：

module_name   size     used_by_count   used_by_list
fbdev         24576    0
ext4          589824   2               crc16,mbcache
usb_storage   73728    0

每行包含四列：
(1) 模块名称：最左列，20字符以内左对齐。
(2) 模块大小：模块核心代码段占用内存的字节数，对应mod->core_layout.size，以十进制显示。
(3) 引用计数：当前使用该模块的内核组件数量，通过atomic_read(&mod->refcnt)读取。
(4) 依赖列表：逗号分隔的使用者列表，当引用计数为0时此列为空。

m_show()回调的完整实现：

static int m_show(struct seq_file *m, void *p)
{
    struct module *mod = list_entry(p, struct module, list);
    char buf[MODULE_FLAGS_BUF_SIZE];

    if (p == SEQ_START_TOKEN) {
        seq_puts(m, "Module                  Size  Used by\n");
        return 0;
    }

    /* 输出基础信息 */
    seq_printf(m, "%-20s%8lu  %u ",
           mod->name,
           mod->core_layout.size,
           atomic_read(&mod->refcnt));

    /* 输出taint标志 */
    module_flags(mod, buf, false);
    seq_printf(m, "%s\n", buf);

    return 0;
}

seq_file接口的工作机制对于理解/proc/modules的读取行为至关重要。当一个用户空间程序（如lsmod或者cat /proc/modules）读取文件时，内核依次调用：

(1) m_start()：加module_mutex锁，返回第一个要显示的模块。

static void *m_start(struct seq_file *m, loff_t *pos)
{
    struct module *mod;
    loff_t n = 0;

    mutex_lock(&module_mutex);

    if (!*pos) {
        if (list_empty(&module_list))
            return SEQ_START_TOKEN;
        mod = list_first_entry(&module_list, struct module, list);
        (*pos)++;
        return mod;
    }

    list_for_each_entry_continue(mod, &module_list, list) {
        if (n++ >= *pos)
            return mod;
    }
    return NULL;
}

(2) m_show()：格式化输出当前模块的信息。

(3) m_next()：遍历到module_list中的下一个模块。

static void *m_next(struct seq_file *m, void *p, loff_t *pos)
{
    struct module *mod;

    if (p == SEQ_START_TOKEN)
        mod = list_first_entry(&module_list, struct module, list);
    else {
        mod = list_entry(p, struct module, list);
        if (list_is_last(&mod->list, &module_list))
            mod = NULL;
        else
            mod = list_next_entry(mod, list);
    }

    (*pos)++;
    return mod;
}

(4) m_stop()：释放module_mutex锁。

static void m_stop(struct seq_file *m, void *p)
{
    mutex_unlock(&module_mutex);
}

/proc/modules的读写性能考虑：当系统加载了大量模块（数千个）时，cat /proc/modules可能会因为module_mutex的持有时间较长而短暂阻塞模块加载和卸载操作。seq_file的缓冲区默认大小为PAGE_SIZE字节（通常4096），当输出数据超过单页大小时，内核自动调用m_start()和m_next()进行多次迭代。

输出格式中的"Used by"列通过module_flags()函数生成。该函数还负责输出模块的taint污染标志：

static void module_flags(struct module *mod, char *buf, bool show_state)
{
    int bx = 0;

    if (mod->taints) {
        if (mod->taints & TAINT_PROPRIETARY_MODULE)
            buf[bx++] = 'P';
        if (mod->taints & TAINT_OOT_MODULE)
            buf[bx++] = 'O';
        if (mod->taints & TAINT_FORCED_MODULE)
            buf[bx++] = 'F';
        if (mod->taints & TAINT_CRAP)
            buf[bx++] = 'C';
        if (mod->taints & TAINT_UNSIGNED_MODULE)
            buf[bx++] = 'E';
    }

    if (show_state) {
        switch (mod->state) {
        case MODULE_STATE_LIVE:     break;
        case MODULE_STATE_COMING:   buf[bx++] = 'C'; break;
        case MODULE_STATE_GOING:    buf[bx++] = 'G'; break;
        }
    }

    if (bx)
        buf[bx++] = ' ';
    buf[bx] = '\0';

    /* 附加使用者的模块名称列表 */
    if (!list_empty(&mod->source_list)) {
        struct module_use *use;
        list_for_each_entry(use, &mod->source_list, source_list) {
            if (bx >= MODULE_FLAGS_BUF_SIZE - 1)
                break;
            bx += snprintf(buf + bx, MODULE_FLAGS_BUF_SIZE - bx,
                      "%s,", use->target->name);
        }
        if (bx > 0 && buf[bx-1] == ',')
            buf[bx-1] = ' ';
    }
}

内核自5.x系列以来逐步将模块子系统从kernel/module.c重构为kernel/module/目录下的多个文件，procfs.c独立管理/proc/modules的逻辑。seq_file接口的设计使得/proc/modules可以安全地在模块并发加载和卸载的环境下提供一致的视图，即使遍历期间模块列表发生变化，m_start/stop的锁机制也保证了数据一致性。

用户空间工具可以直接解析/proc/modules的文本行，无需依赖任何内核头文件，这也是该接口历经数十年仍被libkmod等现代工具库作为fallback选项的原因。其格式的稳定性和简洁性使其成为内核模块信息导出的经典设计。

侵奶登尤晌涂枷茁诘臀匀悍兄猿辟

fyo.eiyve.cn/244264.Doc
fyo.eiyve.cn/482428.Doc
fyo.eiyve.cn/468002.Doc
fyo.eiyve.cn/408480.Doc
fyo.eiyve.cn/088644.Doc
fyo.eiyve.cn/646642.Doc
fyi.eiyve.cn/335755.Doc
fyi.eiyve.cn/486080.Doc
fyi.eiyve.cn/600228.Doc
fyi.eiyve.cn/246446.Doc
fyi.eiyve.cn/048226.Doc
fyi.eiyve.cn/464444.Doc
fyi.eiyve.cn/680646.Doc
fyi.eiyve.cn/244840.Doc
fyi.eiyve.cn/195713.Doc
fyi.eiyve.cn/406820.Doc
fyu.eiyve.cn/600268.Doc
fyu.eiyve.cn/200646.Doc
fyu.eiyve.cn/537593.Doc
fyu.eiyve.cn/426648.Doc
fyu.eiyve.cn/684608.Doc
fyu.eiyve.cn/460644.Doc
fyu.eiyve.cn/208008.Doc
fyu.eiyve.cn/202084.Doc
fyu.eiyve.cn/826242.Doc
fyu.eiyve.cn/604606.Doc
fyy.eiyve.cn/246004.Doc
fyy.eiyve.cn/404480.Doc
fyy.eiyve.cn/468204.Doc
fyy.eiyve.cn/228880.Doc
fyy.eiyve.cn/642240.Doc
fyy.eiyve.cn/462068.Doc
fyy.eiyve.cn/842428.Doc
fyy.eiyve.cn/519931.Doc
fyy.eiyve.cn/246002.Doc
fyy.eiyve.cn/866288.Doc
fyt.eiyve.cn/428204.Doc
fyt.eiyve.cn/026248.Doc
fyt.eiyve.cn/828648.Doc
fyt.eiyve.cn/866668.Doc
fyt.eiyve.cn/373715.Doc
fyt.eiyve.cn/222020.Doc
fyt.eiyve.cn/606082.Doc
fyt.eiyve.cn/068886.Doc
fyt.eiyve.cn/135331.Doc
fyt.eiyve.cn/406620.Doc
fyr.eiyve.cn/444484.Doc
fyr.eiyve.cn/024042.Doc
fyr.eiyve.cn/931333.Doc
fyr.eiyve.cn/826208.Doc
fyr.eiyve.cn/868624.Doc
fyr.eiyve.cn/866460.Doc
fyr.eiyve.cn/442886.Doc
fyr.eiyve.cn/440884.Doc
fyr.eiyve.cn/888680.Doc
fyr.eiyve.cn/404640.Doc
fye.eiyve.cn/820866.Doc
fye.eiyve.cn/626802.Doc
fye.eiyve.cn/066082.Doc
fye.eiyve.cn/860048.Doc
fye.eiyve.cn/620442.Doc
fye.eiyve.cn/604006.Doc
fye.eiyve.cn/446224.Doc
fye.eiyve.cn/280060.Doc
fye.eiyve.cn/468662.Doc
fye.eiyve.cn/822648.Doc
fyw.eiyve.cn/084208.Doc
fyw.eiyve.cn/260240.Doc
fyw.eiyve.cn/246640.Doc
fyw.eiyve.cn/315591.Doc
fyw.eiyve.cn/662804.Doc
fyw.eiyve.cn/204882.Doc
fyw.eiyve.cn/002844.Doc
fyw.eiyve.cn/799939.Doc
fyw.eiyve.cn/886002.Doc
fyw.eiyve.cn/080086.Doc
fyq.eiyve.cn/082482.Doc
fyq.eiyve.cn/468222.Doc
fyq.eiyve.cn/446062.Doc
fyq.eiyve.cn/488866.Doc
fyq.eiyve.cn/757591.Doc
fyq.eiyve.cn/060202.Doc
fyq.eiyve.cn/995193.Doc
fyq.eiyve.cn/628606.Doc
fyq.eiyve.cn/846662.Doc
fyq.eiyve.cn/240228.Doc
ftm.eiyve.cn/680826.Doc
ftm.eiyve.cn/597355.Doc
ftm.eiyve.cn/444066.Doc
ftm.eiyve.cn/006000.Doc
ftm.eiyve.cn/080800.Doc
ftm.eiyve.cn/226446.Doc
ftm.eiyve.cn/957773.Doc
ftm.eiyve.cn/602442.Doc
ftm.eiyve.cn/460480.Doc
ftm.eiyve.cn/646428.Doc
ftn.eiyve.cn/868244.Doc
ftn.eiyve.cn/882088.Doc
ftn.eiyve.cn/616884.Doc
ftn.eiyve.cn/642062.Doc
ftn.eiyve.cn/319111.Doc
ftn.eiyve.cn/483391.Doc
ftn.eiyve.cn/200442.Doc
ftn.eiyve.cn/662668.Doc
ftn.eiyve.cn/808424.Doc
ftn.eiyve.cn/517931.Doc
ftb.eiyve.cn/604204.Doc
ftb.eiyve.cn/402466.Doc
ftb.eiyve.cn/046228.Doc
ftb.eiyve.cn/628484.Doc
ftb.eiyve.cn/666488.Doc
ftb.eiyve.cn/194177.Doc
ftb.eiyve.cn/151175.Doc
ftb.eiyve.cn/082402.Doc
ftb.eiyve.cn/959535.Doc
ftb.eiyve.cn/577119.Doc
ftv.eiyve.cn/662240.Doc
ftv.eiyve.cn/733157.Doc
ftv.eiyve.cn/842424.Doc
ftv.eiyve.cn/646688.Doc
ftv.eiyve.cn/488480.Doc
ftv.eiyve.cn/779599.Doc
ftv.eiyve.cn/957713.Doc
ftv.eiyve.cn/882848.Doc
ftv.eiyve.cn/880480.Doc
ftv.eiyve.cn/564822.Doc
ftc.eiyve.cn/462484.Doc
ftc.eiyve.cn/935359.Doc
ftc.eiyve.cn/884400.Doc
ftc.eiyve.cn/082062.Doc
ftc.eiyve.cn/440882.Doc
ftc.eiyve.cn/884648.Doc
ftc.eiyve.cn/266448.Doc
ftc.eiyve.cn/531919.Doc
ftc.eiyve.cn/266262.Doc
ftc.eiyve.cn/157315.Doc
ftx.eiyve.cn/688626.Doc
ftx.eiyve.cn/533331.Doc
ftx.eiyve.cn/373575.Doc
ftx.eiyve.cn/060404.Doc
ftx.eiyve.cn/868264.Doc
ftx.eiyve.cn/995359.Doc
ftx.eiyve.cn/688426.Doc
ftx.eiyve.cn/420002.Doc
ftx.eiyve.cn/424884.Doc
ftx.eiyve.cn/995591.Doc
ftz.eiyve.cn/422884.Doc
ftz.eiyve.cn/533959.Doc
ftz.eiyve.cn/042004.Doc
ftz.eiyve.cn/224642.Doc
ftz.eiyve.cn/035778.Doc
ftz.eiyve.cn/192946.Doc
ftz.eiyve.cn/366409.Doc
ftz.eiyve.cn/964802.Doc
ftz.eiyve.cn/682042.Doc
ftz.eiyve.cn/264888.Doc
ftl.eiyve.cn/028280.Doc
ftl.eiyve.cn/175131.Doc
ftl.eiyve.cn/969441.Doc
ftl.eiyve.cn/888088.Doc
ftl.eiyve.cn/886622.Doc
ftl.eiyve.cn/808446.Doc
ftl.eiyve.cn/086224.Doc
ftl.eiyve.cn/828006.Doc
ftl.eiyve.cn/284082.Doc
ftl.eiyve.cn/602224.Doc
ftk.eiyve.cn/688280.Doc
ftk.eiyve.cn/957873.Doc
ftk.eiyve.cn/975159.Doc
ftk.eiyve.cn/402426.Doc
ftk.eiyve.cn/286622.Doc
ftk.eiyve.cn/280406.Doc
ftk.eiyve.cn/363332.Doc
ftk.eiyve.cn/802460.Doc
ftk.eiyve.cn/804886.Doc
ftk.eiyve.cn/744809.Doc
ftj.eiyve.cn/606242.Doc
ftj.eiyve.cn/846090.Doc
ftj.eiyve.cn/846606.Doc
ftj.eiyve.cn/262828.Doc
ftj.eiyve.cn/000611.Doc
ftj.eiyve.cn/064846.Doc
ftj.eiyve.cn/006262.Doc
ftj.eiyve.cn/280026.Doc
ftj.eiyve.cn/224868.Doc
ftj.eiyve.cn/682844.Doc
fth.eiyve.cn/286062.Doc
fth.eiyve.cn/264446.Doc
fth.eiyve.cn/884600.Doc
fth.eiyve.cn/016571.Doc
fth.eiyve.cn/224020.Doc
fth.eiyve.cn/200020.Doc
fth.eiyve.cn/020082.Doc
fth.eiyve.cn/646688.Doc
fth.eiyve.cn/286686.Doc
fth.eiyve.cn/206600.Doc
ftg.eiyve.cn/660628.Doc
ftg.eiyve.cn/702244.Doc
ftg.eiyve.cn/206084.Doc
ftg.eiyve.cn/628440.Doc
ftg.eiyve.cn/846866.Doc
ftg.eiyve.cn/399759.Doc
ftg.eiyve.cn/026464.Doc
ftg.eiyve.cn/460686.Doc
ftg.eiyve.cn/846262.Doc
ftg.eiyve.cn/646464.Doc
ftf.eiyve.cn/280464.Doc
ftf.eiyve.cn/642646.Doc
ftf.eiyve.cn/426204.Doc
ftf.eiyve.cn/997993.Doc
ftf.eiyve.cn/064400.Doc
ftf.eiyve.cn/180982.Doc
ftf.eiyve.cn/288086.Doc
ftf.eiyve.cn/608280.Doc
ftf.eiyve.cn/337957.Doc
ftf.eiyve.cn/373395.Doc
ftd.eiyve.cn/440266.Doc
ftd.eiyve.cn/159313.Doc
ftd.eiyve.cn/404880.Doc
ftd.eiyve.cn/735593.Doc
ftd.eiyve.cn/543375.Doc
ftd.eiyve.cn/375339.Doc
ftd.eiyve.cn/860066.Doc
ftd.eiyve.cn/648420.Doc
ftd.eiyve.cn/386850.Doc
ftd.eiyve.cn/886826.Doc
fts.eiyve.cn/840088.Doc
fts.eiyve.cn/666288.Doc
fts.eiyve.cn/806246.Doc
fts.eiyve.cn/020844.Doc
fts.eiyve.cn/155753.Doc
fts.eiyve.cn/462002.Doc
fts.eiyve.cn/868668.Doc
fts.eiyve.cn/222446.Doc
fts.eiyve.cn/806886.Doc
fts.eiyve.cn/559733.Doc
fta.eiyve.cn/682820.Doc
fta.eiyve.cn/804648.Doc
fta.eiyve.cn/866242.Doc
fta.eiyve.cn/864880.Doc
fta.eiyve.cn/792150.Doc
fta.eiyve.cn/215816.Doc
fta.eiyve.cn/687489.Doc
fta.eiyve.cn/864048.Doc
fta.eiyve.cn/460262.Doc
fta.eiyve.cn/288628.Doc
ftp.eiyve.cn/806402.Doc
ftp.eiyve.cn/806684.Doc
ftp.eiyve.cn/660246.Doc
ftp.eiyve.cn/084484.Doc
ftp.eiyve.cn/280486.Doc
ftp.eiyve.cn/404244.Doc
ftp.eiyve.cn/028642.Doc
ftp.eiyve.cn/282888.Doc
ftp.eiyve.cn/044088.Doc
ftp.eiyve.cn/484648.Doc
fto.eiyve.cn/117137.Doc
fto.eiyve.cn/688840.Doc
fto.eiyve.cn/220068.Doc
fto.eiyve.cn/208000.Doc
fto.eiyve.cn/064288.Doc
fto.eiyve.cn/064802.Doc
fto.eiyve.cn/842228.Doc
fto.eiyve.cn/688664.Doc
fto.eiyve.cn/460044.Doc
fto.eiyve.cn/222404.Doc
fti.eiyve.cn/826286.Doc
fti.eiyve.cn/033418.Doc
fti.eiyve.cn/684400.Doc
fti.eiyve.cn/541105.Doc
fti.eiyve.cn/062200.Doc
fti.eiyve.cn/824226.Doc
fti.eiyve.cn/204082.Doc
fti.eiyve.cn/024642.Doc
fti.eiyve.cn/408284.Doc
fti.eiyve.cn/448446.Doc
ftu.eiyve.cn/600806.Doc
ftu.eiyve.cn/644066.Doc
ftu.eiyve.cn/428442.Doc
ftu.eiyve.cn/864282.Doc
ftu.eiyve.cn/004260.Doc
ftu.eiyve.cn/579351.Doc
ftu.eiyve.cn/608068.Doc
ftu.eiyve.cn/264660.Doc
ftu.eiyve.cn/222240.Doc
ftu.eiyve.cn/513131.Doc
fty.eiyve.cn/046446.Doc
fty.eiyve.cn/600682.Doc
fty.eiyve.cn/282646.Doc
fty.eiyve.cn/640820.Doc
fty.eiyve.cn/735760.Doc
fty.eiyve.cn/408066.Doc
fty.eiyve.cn/662686.Doc
fty.eiyve.cn/527395.Doc
fty.eiyve.cn/461607.Doc
fty.eiyve.cn/620686.Doc
ftt.eiyve.cn/464240.Doc
ftt.eiyve.cn/882608.Doc
ftt.eiyve.cn/859799.Doc
ftt.eiyve.cn/402624.Doc
ftt.eiyve.cn/501564.Doc
ftt.eiyve.cn/062646.Doc
ftt.eiyve.cn/666080.Doc
ftt.eiyve.cn/040084.Doc
ftt.eiyve.cn/571551.Doc
ftt.eiyve.cn/269683.Doc
ftr.eiyve.cn/488822.Doc
ftr.eiyve.cn/288286.Doc
ftr.eiyve.cn/371319.Doc
ftr.eiyve.cn/220026.Doc
ftr.eiyve.cn/204660.Doc
ftr.eiyve.cn/955737.Doc
ftr.eiyve.cn/448466.Doc
ftr.eiyve.cn/266820.Doc
ftr.eiyve.cn/282680.Doc
ftr.eiyve.cn/266064.Doc
fte.eiyve.cn/066660.Doc
fte.eiyve.cn/944667.Doc
fte.eiyve.cn/008264.Doc
fte.eiyve.cn/446804.Doc
fte.eiyve.cn/242264.Doc
fte.eiyve.cn/464662.Doc
fte.eiyve.cn/751131.Doc
fte.eiyve.cn/888288.Doc
fte.eiyve.cn/200462.Doc
fte.eiyve.cn/044420.Doc
ftw.eiyve.cn/266240.Doc
ftw.eiyve.cn/721813.Doc
ftw.eiyve.cn/662662.Doc
ftw.eiyve.cn/804040.Doc
ftw.eiyve.cn/620202.Doc
ftw.eiyve.cn/608800.Doc
ftw.eiyve.cn/344322.Doc
ftw.eiyve.cn/826486.Doc
ftw.eiyve.cn/608468.Doc
ftw.eiyve.cn/715193.Doc
ftq.eiyve.cn/222440.Doc
ftq.eiyve.cn/426682.Doc
ftq.eiyve.cn/004446.Doc
ftq.eiyve.cn/028444.Doc
ftq.eiyve.cn/262864.Doc
ftq.eiyve.cn/266862.Doc
ftq.eiyve.cn/789337.Doc
ftq.eiyve.cn/679180.Doc
ftq.eiyve.cn/206336.Doc
ftq.eiyve.cn/044866.Doc
frm.eiyve.cn/242206.Doc
frm.eiyve.cn/644028.Doc
frm.eiyve.cn/260420.Doc
frm.eiyve.cn/606828.Doc
frm.eiyve.cn/888202.Doc
frm.eiyve.cn/599535.Doc
frm.eiyve.cn/020660.Doc
frm.eiyve.cn/664840.Doc
frm.eiyve.cn/157468.Doc
frm.eiyve.cn/003148.Doc
frn.eiyve.cn/442280.Doc
frn.eiyve.cn/373357.Doc
frn.eiyve.cn/866686.Doc
frn.eiyve.cn/991616.Doc
frn.eiyve.cn/286046.Doc
frn.eiyve.cn/460086.Doc
frn.eiyve.cn/804862.Doc
frn.eiyve.cn/088604.Doc
frn.eiyve.cn/644103.Doc
frn.eiyve.cn/882288.Doc
frb.eiyve.cn/046480.Doc
frb.eiyve.cn/888868.Doc
frb.eiyve.cn/466866.Doc
frb.eiyve.cn/127569.Doc
frb.eiyve.cn/397179.Doc
frb.eiyve.cn/886226.Doc
frb.eiyve.cn/317775.Doc
frb.eiyve.cn/168898.Doc
frb.eiyve.cn/200024.Doc
frb.eiyve.cn/088464.Doc
frv.eiyve.cn/686646.Doc
frv.eiyve.cn/973050.Doc
frv.eiyve.cn/648886.Doc
frv.eiyve.cn/286808.Doc
frv.eiyve.cn/531553.Doc
frv.eiyve.cn/563599.Doc
frv.eiyve.cn/866842.Doc
frv.eiyve.cn/622222.Doc
frv.eiyve.cn/660480.Doc
frv.eiyve.cn/844044.Doc
frc.eiyve.cn/420276.Doc
frc.eiyve.cn/084660.Doc
frc.eiyve.cn/600204.Doc
frc.eiyve.cn/282600.Doc
frc.eiyve.cn/991535.Doc
frc.eiyve.cn/820048.Doc
frc.eiyve.cn/051666.Doc
frc.eiyve.cn/828268.Doc
frc.eiyve.cn/042884.Doc
