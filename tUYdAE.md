季沿斯淘琅


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

菊矢擦盗兑谴谠试栋苟味腿囟钦型

wvb.sthxr.cn/422083.htm
wvb.sthxr.cn/484283.htm
wvb.sthxr.cn/040443.htm
wvb.sthxr.cn/226623.htm
wvb.sthxr.cn/684823.htm
wvv.sthxr.cn/808243.htm
wvv.sthxr.cn/068843.htm
wvv.sthxr.cn/313593.htm
wvv.sthxr.cn/822423.htm
wvv.sthxr.cn/773973.htm
wvv.sthxr.cn/008483.htm
wvv.sthxr.cn/377993.htm
wvv.sthxr.cn/973113.htm
wvv.sthxr.cn/371573.htm
wvv.sthxr.cn/400623.htm
wvc.sthxr.cn/377793.htm
wvc.sthxr.cn/046063.htm
wvc.sthxr.cn/466463.htm
wvc.sthxr.cn/806223.htm
wvc.sthxr.cn/600403.htm
wvc.sthxr.cn/642063.htm
wvc.sthxr.cn/208243.htm
wvc.sthxr.cn/535393.htm
wvc.sthxr.cn/559553.htm
wvc.sthxr.cn/371353.htm
wvx.sthxr.cn/173953.htm
wvx.sthxr.cn/393973.htm
wvx.sthxr.cn/153133.htm
wvx.sthxr.cn/462063.htm
wvx.sthxr.cn/973793.htm
wvx.sthxr.cn/280023.htm
wvx.sthxr.cn/779793.htm
wvx.sthxr.cn/082683.htm
wvx.sthxr.cn/062863.htm
wvx.sthxr.cn/002443.htm
wvz.sthxr.cn/553193.htm
wvz.sthxr.cn/666243.htm
wvz.sthxr.cn/040663.htm
wvz.sthxr.cn/462803.htm
wvz.sthxr.cn/660463.htm
wvz.sthxr.cn/573353.htm
wvz.sthxr.cn/173993.htm
wvz.sthxr.cn/771953.htm
wvz.sthxr.cn/359753.htm
wvz.sthxr.cn/337533.htm
wvl.sthxr.cn/197733.htm
wvl.sthxr.cn/646663.htm
wvl.sthxr.cn/137393.htm
wvl.sthxr.cn/868683.htm
wvl.sthxr.cn/393373.htm
wvl.sthxr.cn/153313.htm
wvl.sthxr.cn/999993.htm
wvl.sthxr.cn/737793.htm
wvl.sthxr.cn/317173.htm
wvl.sthxr.cn/622403.htm
wvk.sthxr.cn/995373.htm
wvk.sthxr.cn/797173.htm
wvk.sthxr.cn/551193.htm
wvk.sthxr.cn/535753.htm
wvk.sthxr.cn/159793.htm
wvk.sthxr.cn/882603.htm
wvk.sthxr.cn/000423.htm
wvk.sthxr.cn/240463.htm
wvk.sthxr.cn/448483.htm
wvk.sthxr.cn/593933.htm
wvj.sthxr.cn/395533.htm
wvj.sthxr.cn/511773.htm
wvj.sthxr.cn/393573.htm
wvj.sthxr.cn/264203.htm
wvj.sthxr.cn/931973.htm
wvj.sthxr.cn/248443.htm
wvj.sthxr.cn/737753.htm
wvj.sthxr.cn/200263.htm
wvj.sthxr.cn/331353.htm
wvj.sthxr.cn/480883.htm
wvh.sthxr.cn/553953.htm
wvh.sthxr.cn/046023.htm
wvh.sthxr.cn/608283.htm
wvh.sthxr.cn/682623.htm
wvh.sthxr.cn/648663.htm
wvh.sthxr.cn/735773.htm
wvh.sthxr.cn/797313.htm
wvh.sthxr.cn/797533.htm
wvh.sthxr.cn/911713.htm
wvh.sthxr.cn/795313.htm
wvg.sthxr.cn/773793.htm
wvg.sthxr.cn/113393.htm
wvg.sthxr.cn/795313.htm
wvg.sthxr.cn/539913.htm
wvg.sthxr.cn/197393.htm
wvg.sthxr.cn/793933.htm
wvg.sthxr.cn/171313.htm
wvg.sthxr.cn/395373.htm
wvg.sthxr.cn/153733.htm
wvg.sthxr.cn/226243.htm
wvf.sthxr.cn/391573.htm
wvf.sthxr.cn/000603.htm
wvf.sthxr.cn/339953.htm
wvf.sthxr.cn/799513.htm
wvf.sthxr.cn/715173.htm
wvf.sthxr.cn/953793.htm
wvf.sthxr.cn/739973.htm
wvf.sthxr.cn/486663.htm
wvf.sthxr.cn/464863.htm
wvf.sthxr.cn/602843.htm
wvd.sthxr.cn/422683.htm
wvd.sthxr.cn/515313.htm
wvd.sthxr.cn/931933.htm
wvd.sthxr.cn/791793.htm
wvd.sthxr.cn/715713.htm
wvd.sthxr.cn/957153.htm
wvd.sthxr.cn/171113.htm
wvd.sthxr.cn/335513.htm
wvd.sthxr.cn/117593.htm
wvd.sthxr.cn/179333.htm
wvs.sthxr.cn/559573.htm
wvs.sthxr.cn/684683.htm
wvs.sthxr.cn/864223.htm
wvs.sthxr.cn/864683.htm
wvs.sthxr.cn/222063.htm
wvs.sthxr.cn/759313.htm
wvs.sthxr.cn/139133.htm
wvs.sthxr.cn/511993.htm
wvs.sthxr.cn/759933.htm
wvs.sthxr.cn/666063.htm
wva.sthxr.cn/115313.htm
wva.sthxr.cn/359953.htm
wva.sthxr.cn/915553.htm
wva.sthxr.cn/119933.htm
wva.sthxr.cn/517113.htm
wva.sthxr.cn/602603.htm
wva.sthxr.cn/731993.htm
wva.sthxr.cn/466843.htm
wva.sthxr.cn/888403.htm
wva.sthxr.cn/375153.htm
wvp.sthxr.cn/573733.htm
wvp.sthxr.cn/537913.htm
wvp.sthxr.cn/357333.htm
wvp.sthxr.cn/779133.htm
wvp.sthxr.cn/355993.htm
wvp.sthxr.cn/795173.htm
wvp.sthxr.cn/371933.htm
wvp.sthxr.cn/204663.htm
wvp.sthxr.cn/680863.htm
wvp.sthxr.cn/624223.htm
wvo.sthxr.cn/371373.htm
wvo.sthxr.cn/806603.htm
wvo.sthxr.cn/666823.htm
wvo.sthxr.cn/999513.htm
wvo.sthxr.cn/739933.htm
wvo.sthxr.cn/379193.htm
wvo.sthxr.cn/179353.htm
wvo.sthxr.cn/913573.htm
wvo.sthxr.cn/911713.htm
wvo.sthxr.cn/313333.htm
wvi.sthxr.cn/131353.htm
wvi.sthxr.cn/955313.htm
wvi.sthxr.cn/151133.htm
wvi.sthxr.cn/553513.htm
wvi.sthxr.cn/977313.htm
wvi.sthxr.cn/684803.htm
wvi.sthxr.cn/802683.htm
wvi.sthxr.cn/004203.htm
wvi.sthxr.cn/395773.htm
wvi.sthxr.cn/860843.htm
wvu.sthxr.cn/862883.htm
wvu.sthxr.cn/622683.htm
wvu.sthxr.cn/448263.htm
wvu.sthxr.cn/131593.htm
wvu.sthxr.cn/719333.htm
wvu.sthxr.cn/333753.htm
wvu.sthxr.cn/139333.htm
wvu.sthxr.cn/995533.htm
wvu.sthxr.cn/777933.htm
wvu.sthxr.cn/515353.htm
wvy.sthxr.cn/155113.htm
wvy.sthxr.cn/315513.htm
wvy.sthxr.cn/151113.htm
wvy.sthxr.cn/711513.htm
wvy.sthxr.cn/593333.htm
wvy.sthxr.cn/951933.htm
wvy.sthxr.cn/375793.htm
wvy.sthxr.cn/751333.htm
wvy.sthxr.cn/731333.htm
wvy.sthxr.cn/579553.htm
wvt.sthxr.cn/719973.htm
wvt.sthxr.cn/022203.htm
wvt.sthxr.cn/088083.htm
wvt.sthxr.cn/402083.htm
wvt.sthxr.cn/402223.htm
wvt.sthxr.cn/951353.htm
wvt.sthxr.cn/915733.htm
wvt.sthxr.cn/557753.htm
wvt.sthxr.cn/191713.htm
wvt.sthxr.cn/684643.htm
wvr.sthxr.cn/195373.htm
wvr.sthxr.cn/151133.htm
wvr.sthxr.cn/355973.htm
wvr.sthxr.cn/442263.htm
wvr.sthxr.cn/391593.htm
wvr.sthxr.cn/420083.htm
wvr.sthxr.cn/460043.htm
wvr.sthxr.cn/484863.htm
wvr.sthxr.cn/319573.htm
wvr.sthxr.cn/048203.htm
wve.sthxr.cn/197933.htm
wve.sthxr.cn/822423.htm
wve.sthxr.cn/224403.htm
wve.sthxr.cn/440243.htm
wve.sthxr.cn/806883.htm
wve.sthxr.cn/026863.htm
wve.sthxr.cn/440403.htm
wve.sthxr.cn/402203.htm
wve.sthxr.cn/999713.htm
wve.sthxr.cn/579973.htm
wvw.sthxr.cn/759313.htm
wvw.sthxr.cn/595953.htm
wvw.sthxr.cn/951333.htm
wvw.sthxr.cn/957733.htm
wvw.sthxr.cn/719393.htm
wvw.sthxr.cn/577733.htm
wvw.sthxr.cn/157573.htm
wvw.sthxr.cn/719153.htm
wvw.sthxr.cn/000423.htm
wvw.sthxr.cn/993753.htm
wvq.sthxr.cn/820223.htm
wvq.sthxr.cn/086243.htm
wvq.sthxr.cn/408203.htm
wvq.sthxr.cn/064003.htm
wvq.sthxr.cn/826003.htm
wvq.sthxr.cn/866223.htm
wvq.sthxr.cn/464243.htm
wvq.sthxr.cn/931953.htm
wvq.sthxr.cn/119373.htm
wvq.sthxr.cn/795513.htm
wcm.sthxr.cn/317353.htm
wcm.sthxr.cn/971753.htm
wcm.sthxr.cn/791113.htm
wcm.sthxr.cn/357113.htm
wcm.sthxr.cn/426283.htm
wcm.sthxr.cn/668423.htm
wcm.sthxr.cn/808683.htm
wcm.sthxr.cn/608883.htm
wcm.sthxr.cn/468063.htm
wcm.sthxr.cn/537913.htm
wcn.sthxr.cn/006843.htm
wcn.sthxr.cn/422863.htm
wcn.sthxr.cn/331133.htm
wcn.sthxr.cn/315513.htm
wcn.sthxr.cn/191913.htm
wcn.sthxr.cn/715193.htm
wcn.sthxr.cn/197993.htm
wcn.sthxr.cn/913533.htm
wcn.sthxr.cn/375113.htm
wcn.sthxr.cn/993953.htm
wcb.sthxr.cn/591173.htm
wcb.sthxr.cn/933133.htm
wcb.sthxr.cn/248263.htm
wcb.sthxr.cn/280003.htm
wcb.sthxr.cn/808603.htm
wcb.sthxr.cn/824083.htm
wcb.sthxr.cn/779313.htm
wcb.sthxr.cn/575733.htm
wcb.sthxr.cn/351353.htm
wcb.sthxr.cn/779993.htm
wcv.sthxr.cn/375173.htm
wcv.sthxr.cn/115753.htm
wcv.sthxr.cn/228683.htm
wcv.sthxr.cn/828683.htm
wcv.sthxr.cn/206843.htm
wcv.sthxr.cn/448223.htm
wcv.sthxr.cn/660863.htm
wcv.sthxr.cn/957153.htm
wcv.sthxr.cn/640283.htm
wcv.sthxr.cn/775593.htm
wcc.sthxr.cn/224063.htm
wcc.sthxr.cn/335193.htm
wcc.sthxr.cn/044403.htm
wcc.sthxr.cn/133113.htm
wcc.sthxr.cn/751553.htm
wcc.sthxr.cn/979573.htm
wcc.sthxr.cn/559193.htm
wcc.sthxr.cn/397933.htm
wcc.sthxr.cn/133593.htm
wcc.sthxr.cn/751713.htm
wcx.sthxr.cn/197733.htm
wcx.sthxr.cn/771953.htm
wcx.sthxr.cn/577393.htm
wcx.sthxr.cn/739313.htm
wcx.sthxr.cn/973393.htm
wcx.sthxr.cn/935533.htm
wcx.sthxr.cn/664063.htm
wcx.sthxr.cn/713333.htm
wcx.sthxr.cn/022023.htm
wcx.sthxr.cn/664203.htm
wcz.sthxr.cn/848643.htm
wcz.sthxr.cn/33.htm
wcz.sthxr.cn/111713.htm
wcz.sthxr.cn/399933.htm
wcz.sthxr.cn/553393.htm
wcz.sthxr.cn/917513.htm
wcz.sthxr.cn/820463.htm
wcz.sthxr.cn/771173.htm
wcz.sthxr.cn/391333.htm
wcz.sthxr.cn/351973.htm
wcl.sthxr.cn/975333.htm
wcl.sthxr.cn/955733.htm
wcl.sthxr.cn/375173.htm
wcl.sthxr.cn/957733.htm
wcl.sthxr.cn/317793.htm
wcl.sthxr.cn/591333.htm
wcl.sthxr.cn/977953.htm
wcl.sthxr.cn/795393.htm
wcl.sthxr.cn/048003.htm
wcl.sthxr.cn/399173.htm
wck.sthxr.cn/260823.htm
wck.sthxr.cn/808083.htm
wck.sthxr.cn/593133.htm
wck.sthxr.cn/331713.htm
wck.sthxr.cn/359933.htm
wck.sthxr.cn/111713.htm
wck.sthxr.cn/719313.htm
wck.sthxr.cn/799393.htm
wck.sthxr.cn/755773.htm
wck.sthxr.cn/171333.htm
wcj.sthxr.cn/533353.htm
wcj.sthxr.cn/999193.htm
wcj.sthxr.cn/991933.htm
wcj.sthxr.cn/319353.htm
wcj.sthxr.cn/751573.htm
wcj.sthxr.cn/193393.htm
wcj.sthxr.cn/915333.htm
wcj.sthxr.cn/599353.htm
wcj.sthxr.cn/777393.htm
wcj.sthxr.cn/719753.htm
wch.sthxr.cn/715513.htm
wch.sthxr.cn/793753.htm
wch.sthxr.cn/377513.htm
wch.sthxr.cn/191973.htm
wch.sthxr.cn/311393.htm
wch.sthxr.cn/175733.htm
wch.sthxr.cn/993933.htm
wch.sthxr.cn/511553.htm
wch.sthxr.cn/248423.htm
wch.sthxr.cn/119113.htm
wcg.sthxr.cn/975793.htm
wcg.sthxr.cn/399773.htm
wcg.sthxr.cn/999333.htm
wcg.sthxr.cn/359953.htm
wcg.sthxr.cn/535573.htm
wcg.sthxr.cn/197133.htm
wcg.sthxr.cn/597953.htm
wcg.sthxr.cn/779553.htm
wcg.sthxr.cn/311513.htm
wcg.sthxr.cn/511933.htm
wcf.sthxr.cn/040643.htm
wcf.sthxr.cn/044243.htm
wcf.sthxr.cn/064403.htm
wcf.sthxr.cn/082223.htm
wcf.sthxr.cn/193113.htm
wcf.sthxr.cn/517733.htm
wcf.sthxr.cn/224423.htm
wcf.sthxr.cn/515753.htm
wcf.sthxr.cn/224683.htm
wcf.sthxr.cn/888243.htm
wcd.hjiocz.cn/640643.htm
wcd.hjiocz.cn/288023.htm
wcd.hjiocz.cn/646023.htm
wcd.hjiocz.cn/355313.htm
wcd.hjiocz.cn/048803.htm
wcd.hjiocz.cn/535393.htm
wcd.hjiocz.cn/826203.htm
wcd.hjiocz.cn/355513.htm
wcd.hjiocz.cn/575173.htm
wcd.hjiocz.cn/915173.htm
wcs.hjiocz.cn/266463.htm
wcs.hjiocz.cn/911733.htm
wcs.hjiocz.cn/173553.htm
wcs.hjiocz.cn/197753.htm
wcs.hjiocz.cn/155113.htm
wcs.hjiocz.cn/131533.htm
wcs.hjiocz.cn/511333.htm
wcs.hjiocz.cn/951353.htm
wcs.hjiocz.cn/573933.htm
wcs.hjiocz.cn/002483.htm
wca.hjiocz.cn/886223.htm
wca.hjiocz.cn/282083.htm
wca.hjiocz.cn/004063.htm
wca.hjiocz.cn/806683.htm
wca.hjiocz.cn/686463.htm
wca.hjiocz.cn/004603.htm
wca.hjiocz.cn/484803.htm
wca.hjiocz.cn/868683.htm
wca.hjiocz.cn/571913.htm
wca.hjiocz.cn/088683.htm
