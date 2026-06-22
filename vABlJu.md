疤赝贤堪剿


Linux x86_64_start_kernel引导参数与cmdline解析

x86_64_start_kernel定义在arch/x86/kernel/head64.c中,是x86_64架构下第一个C语言入口函数。它工作在startup_64完成页表初始化和early_idt_setup之后,负责在调用通用的start_kernel之前完成x86_64平台特定的准备工作。

函数原型如下:

```c
// arch/x86/kernel/head64.c
asmlinkage __visible void __init __no_sanitize_address
x86_64_start_kernel(char *real_mode_data)
{
    int i;
    unsigned long load_delta;

    // 保存boot_params结构体指针
    // real_mode_data是bootloader传递的实模式数据块地址
    boot_params = (struct boot_params *)real_mode_data;

    // 从boot_params中提取cmdline
    // boot_params->hdr.cmd_line_ptr指向命令行字符串
    char *cmdline_ptr = __va(boot_params->hdr.cmd_line_ptr);
    if (boot_params->hdr.cmdline_size)
        strncpy(boot_command_line, cmdline_ptr,
                COMMAND_LINE_SIZE - 1);

    // 同样提取early_cmdline用于早期参数处理
    early_cmdline = boot_command_line;
}
```

cmdline解析的核心机制分为两个阶段。第一个阶段是parse_early_param,在start_kernel早期、setup_arch内部被调用,处理early参数(如"mem=、debug、console=等):

```c
// init/main.c 和 arch/x86/kernel/head64.c 中的早期参数解析
void __init parse_early_param(void)
{
    static __initdata int done = 0;
    char *param;

    if (done)
        return;
    done = 1;

    // 遍历boot_command_line,提取early_param
    param = strcpy(early_cmdline, boot_command_line);

    // 调用parse_early_options处理
    parse_early_options(boot_command_line);
}

// early_param宏的定义机制
#define early_param(str, fn)                    \
    static const char __setup_str_##fn[]        \
        __initconst __aligned(1) = str;         \
    static struct obs_kernel_param __setup_##fn \
        __used __section(".init.setup")         \
        __attribute__((aligned((sizeof(long))))) \
        = { __setup_str_##fn, fn, 1 }

// 使用示例: mem=参数
early_param("mem", early_mem);
```

boot_command_line最多可以存储COMMAND_LINE_SIZE(通常是2048)字节。具体的参数提取和存储过程在内核的setup_arch调用中继续深化:

```c
// arch/x86/kernel/setup.c
void __init setup_arch(char **cmdline_p)
{
    // 保存boot_command_line指针
    *cmdline_p = boot_command_line;

    // 从boot_params中解析e820表
    parse_e820_ext();

    // 处理"mem="参数,限制可用内存
    early_mem_setup();

    // 处理"reserve="参数,预留内存区域
    early_reserve_mem();

    // 处理ACPI相关参数
    acpi_boot_table_init();

    // 早期ioremap和内存映射初始化
    early_ioremap_init();

    // 处理console相关参数
    early_console_register();

    // 解析体系结构特定的early参数
    x86_early_init_platform();
}
```

cmdline参数解析的第二个阶段在start_kernel的后续部分,调用parse_args来处理所有非early参数。这一阶段使用的核心函数是parse_args:

```c
// init/main.c
static void __init parse_args(const char *doing,
                              char *args,
                              const struct kernel_param *params,
                              unsigned num,
                              s16 min_level,
                              s16 max_level,
                              void *arg,
                              int (*unknown)(char *param,
                                             char *val,
                                             const char *doing))
{
    char *param, *val, *next;

    // 遍历所有boot参数
    while (*args) {
        // 跳过空白
        args = skip_spaces(args);
        if (!*args)
            break;

        // 找到参数名和值的分界符'='
        param = args;
        args = next_arg(args, &val, &next, 1);

        // 先检查内核模块参数
        if (kernel_param_ops_lookup(param))
            continue;

        // 检查__setup参数
        if (obsolete_checksetup(param))
            continue;

        // 通用参数处理
        if (unknown)
            unknown(param, val, doing);
    }
}
```

cmdline参数在内核中的存储和传递基于一个重要的数据结构——setup_data链表。bootloader在将控制权交给内核之前,会将一些无法放入boot_params固定字段的数据以setup_data链的形式传递给内核:

```c
// arch/x86/include/uapi/asm/bootparam.h
struct setup_data {
    __u64 next;      // 下一个setup_data节点的物理地址
    __u32 type;      // 数据类型: SETUP_E820_EXT、SETUP_DTB等
    __u32 len;       // 数据长度
    __u8  data[0];   // 变长数据
};

// arch/x86/kernel/head64.c 中处理setup_data
void __init parse_setup_data(void)
{
    struct setup_data *data;
    u64 pa_data;

    pa_data = boot_params->hdr.setup_data;

    while (pa_data) {
        data = early_memremap(pa_data, sizeof(*data));
        switch (data->type) {
        case SETUP_E820_EXT:
            // 处理扩展E820表
            e820__memory_setup_extended(data);
            break;
        case SETUP_EFI:
            // 处理EFI数据
            efi_init(data);
            break;
        case SETUP_DTB:
            // 处理设备树blob
            early_init_dt_scan(data);
            break;
        }
        pa_data = data->next;
        early_memunmap(data, sizeof(*data));
    }
}
```

一个关键细节是cmdline中的"——"分隔符。参数解析遇到"——"时,后续所有参数被当作位置参数而非内核选项处理。这一逻辑在next_arg函数中实现:

```c
// lib/cmdline.c
char *next_arg(char *args, char **param, char **val,
               int initrd)
{
    unsigned int i, equals;
    int c;

    *param = args;
    *val = NULL;
    equals = 0;

    for (i = 0; (c = *args) != 0; i++, args++) {
        if (c == '=' && !*val) {
            equals = 1;
            args[-1] = '\0';
            *param = args;
            *val = args + 1;
        } else if (c == '"') {
            // 处理引号包围的值
            args++;
            for (; *args && *args != '"'; args++)
                ;
        } else if (c == ' ' || c == '\t' || c == '\n') {
            break;
        }
    }

    *args++ = '\0';
    return skip_spaces(args);
}
```

x86_64_start_kernel在完成boot_params保存和cmdline初始化后,调用start_kernel进入体系结构无关的内核初始化过程。cmdline参数在内核整个生命周期中通过saved_command_line全局变量保持可用,可以通过/proc/cmdline读取。
回恼窝咨日饭操估淮崭幽贡挠远煤

djr.fffbf.cn/648884.Doc
djr.fffbf.cn/132392.Doc
djr.fffbf.cn/050339.Doc
djr.fffbf.cn/062824.Doc
djr.fffbf.cn/822064.Doc
dje.fffbf.cn/662848.Doc
dje.fffbf.cn/060006.Doc
dje.fffbf.cn/204602.Doc
dje.fffbf.cn/626773.Doc
dje.fffbf.cn/680260.Doc
dje.fffbf.cn/862484.Doc
dje.fffbf.cn/275522.Doc
dje.fffbf.cn/606444.Doc
dje.fffbf.cn/826206.Doc
dje.fffbf.cn/422426.Doc
djw.fffbf.cn/322132.Doc
djw.fffbf.cn/696805.Doc
djw.fffbf.cn/244428.Doc
djw.fffbf.cn/771317.Doc
djw.fffbf.cn/468082.Doc
djw.fffbf.cn/424222.Doc
djw.fffbf.cn/022284.Doc
djw.fffbf.cn/249244.Doc
djw.fffbf.cn/242080.Doc
djw.fffbf.cn/480282.Doc
djq.fffbf.cn/080424.Doc
djq.fffbf.cn/208004.Doc
djq.fffbf.cn/688206.Doc
djq.fffbf.cn/844822.Doc
djq.fffbf.cn/436380.Doc
djq.fffbf.cn/826282.Doc
djq.fffbf.cn/151517.Doc
djq.fffbf.cn/153913.Doc
djq.fffbf.cn/354587.Doc
djq.fffbf.cn/407400.Doc
dhm.fffbf.cn/773335.Doc
dhm.fffbf.cn/884006.Doc
dhm.fffbf.cn/864604.Doc
dhm.fffbf.cn/440595.Doc
dhm.fffbf.cn/488454.Doc
dhm.fffbf.cn/268682.Doc
dhm.fffbf.cn/381587.Doc
dhm.fffbf.cn/684268.Doc
dhm.fffbf.cn/641722.Doc
dhm.fffbf.cn/880880.Doc
dhn.fffbf.cn/359029.Doc
dhn.fffbf.cn/399717.Doc
dhn.fffbf.cn/153917.Doc
dhn.fffbf.cn/369144.Doc
dhn.fffbf.cn/866428.Doc
dhn.fffbf.cn/068284.Doc
dhn.fffbf.cn/753133.Doc
dhn.fffbf.cn/000282.Doc
dhn.fffbf.cn/860608.Doc
dhn.fffbf.cn/626884.Doc
dhb.fffbf.cn/887353.Doc
dhb.fffbf.cn/008426.Doc
dhb.fffbf.cn/718903.Doc
dhb.fffbf.cn/756920.Doc
dhb.fffbf.cn/808248.Doc
dhb.fffbf.cn/519717.Doc
dhb.fffbf.cn/422246.Doc
dhb.fffbf.cn/460442.Doc
dhb.fffbf.cn/357793.Doc
dhb.fffbf.cn/248846.Doc
dhv.fffbf.cn/028262.Doc
dhv.fffbf.cn/488886.Doc
dhv.fffbf.cn/020022.Doc
dhv.fffbf.cn/288202.Doc
dhv.fffbf.cn/948284.Doc
dhv.fffbf.cn/404262.Doc
dhv.fffbf.cn/528185.Doc
dhv.fffbf.cn/800628.Doc
dhv.fffbf.cn/880800.Doc
dhv.fffbf.cn/382796.Doc
dhc.fffbf.cn/119997.Doc
dhc.fffbf.cn/608048.Doc
dhc.fffbf.cn/845427.Doc
dhc.fffbf.cn/600046.Doc
dhc.fffbf.cn/004806.Doc
dhc.fffbf.cn/688862.Doc
dhc.fffbf.cn/066046.Doc
dhc.fffbf.cn/024008.Doc
dhc.fffbf.cn/422868.Doc
dhc.fffbf.cn/868628.Doc
dhx.fffbf.cn/622280.Doc
dhx.fffbf.cn/688068.Doc
dhx.fffbf.cn/422268.Doc
dhx.fffbf.cn/757259.Doc
dhx.fffbf.cn/048280.Doc
dhx.fffbf.cn/008246.Doc
dhx.fffbf.cn/666264.Doc
dhx.fffbf.cn/620208.Doc
dhx.fffbf.cn/690448.Doc
dhx.fffbf.cn/660202.Doc
dhz.fffbf.cn/553150.Doc
dhz.fffbf.cn/130606.Doc
dhz.fffbf.cn/335771.Doc
dhz.fffbf.cn/488608.Doc
dhz.fffbf.cn/888836.Doc
dhz.fffbf.cn/082444.Doc
dhz.fffbf.cn/995715.Doc
dhz.fffbf.cn/729535.Doc
dhz.fffbf.cn/600286.Doc
dhz.fffbf.cn/226600.Doc
dhl.fffbf.cn/577195.Doc
dhl.fffbf.cn/939935.Doc
dhl.fffbf.cn/288060.Doc
dhl.fffbf.cn/300700.Doc
dhl.fffbf.cn/679090.Doc
dhl.fffbf.cn/644048.Doc
dhl.fffbf.cn/948209.Doc
dhl.fffbf.cn/006846.Doc
dhl.fffbf.cn/595193.Doc
dhl.fffbf.cn/577772.Doc
dhk.fffbf.cn/624468.Doc
dhk.fffbf.cn/226220.Doc
dhk.fffbf.cn/644406.Doc
dhk.fffbf.cn/680068.Doc
dhk.fffbf.cn/440040.Doc
dhk.fffbf.cn/022004.Doc
dhk.fffbf.cn/482040.Doc
dhk.fffbf.cn/533595.Doc
dhk.fffbf.cn/020466.Doc
dhk.fffbf.cn/008844.Doc
dhj.fffbf.cn/842800.Doc
dhj.fffbf.cn/460042.Doc
dhj.fffbf.cn/044280.Doc
dhj.fffbf.cn/868684.Doc
dhj.fffbf.cn/742634.Doc
dhj.fffbf.cn/954763.Doc
dhj.fffbf.cn/200240.Doc
dhj.fffbf.cn/880909.Doc
dhj.fffbf.cn/608000.Doc
dhj.fffbf.cn/973915.Doc
dhh.fffbf.cn/93.Doc
dhh.fffbf.cn/004066.Doc
dhh.fffbf.cn/008064.Doc
dhh.fffbf.cn/046068.Doc
dhh.fffbf.cn/464264.Doc
dhh.fffbf.cn/422408.Doc
dhh.fffbf.cn/006262.Doc
dhh.fffbf.cn/153593.Doc
dhh.fffbf.cn/804684.Doc
dhh.fffbf.cn/540400.Doc
dhg.fffbf.cn/662220.Doc
dhg.fffbf.cn/282664.Doc
dhg.fffbf.cn/313111.Doc
dhg.fffbf.cn/477872.Doc
dhg.fffbf.cn/804880.Doc
dhg.fffbf.cn/225300.Doc
dhg.fffbf.cn/971751.Doc
dhg.fffbf.cn/151718.Doc
dhg.fffbf.cn/686020.Doc
dhg.fffbf.cn/070241.Doc
dhf.fffbf.cn/943641.Doc
dhf.fffbf.cn/628268.Doc
dhf.fffbf.cn/680440.Doc
dhf.fffbf.cn/800062.Doc
dhf.fffbf.cn/775779.Doc
dhf.fffbf.cn/179179.Doc
dhf.fffbf.cn/466406.Doc
dhf.fffbf.cn/277718.Doc
dhf.fffbf.cn/624804.Doc
dhf.fffbf.cn/771933.Doc
dhd.fffbf.cn/208204.Doc
dhd.fffbf.cn/808688.Doc
dhd.fffbf.cn/462284.Doc
dhd.fffbf.cn/450918.Doc
dhd.fffbf.cn/164998.Doc
dhd.fffbf.cn/426884.Doc
dhd.fffbf.cn/868664.Doc
dhd.fffbf.cn/004222.Doc
dhd.fffbf.cn/820040.Doc
dhd.fffbf.cn/006606.Doc
dhs.fffbf.cn/662666.Doc
dhs.fffbf.cn/509376.Doc
dhs.fffbf.cn/933434.Doc
dhs.fffbf.cn/211017.Doc
dhs.fffbf.cn/556272.Doc
dhs.fffbf.cn/220848.Doc
dhs.fffbf.cn/688408.Doc
dhs.fffbf.cn/822202.Doc
dhs.fffbf.cn/248600.Doc
dhs.fffbf.cn/884642.Doc
dha.fffbf.cn/406202.Doc
dha.fffbf.cn/088228.Doc
dha.fffbf.cn/864040.Doc
dha.fffbf.cn/222144.Doc
dha.fffbf.cn/040466.Doc
dha.fffbf.cn/286642.Doc
dha.fffbf.cn/424402.Doc
dha.fffbf.cn/480940.Doc
dha.fffbf.cn/153511.Doc
dha.fffbf.cn/004444.Doc
dhp.fffbf.cn/702806.Doc
dhp.fffbf.cn/426806.Doc
dhp.fffbf.cn/288444.Doc
dhp.fffbf.cn/668002.Doc
dhp.fffbf.cn/268020.Doc
dhp.fffbf.cn/644266.Doc
dhp.fffbf.cn/311531.Doc
dhp.fffbf.cn/840200.Doc
dhp.fffbf.cn/826624.Doc
dhp.fffbf.cn/886422.Doc
dho.fffbf.cn/064662.Doc
dho.fffbf.cn/840022.Doc
dho.fffbf.cn/660640.Doc
dho.fffbf.cn/795591.Doc
dho.fffbf.cn/155597.Doc
dho.fffbf.cn/448800.Doc
dho.fffbf.cn/406464.Doc
dho.fffbf.cn/068884.Doc
dho.fffbf.cn/848048.Doc
dho.fffbf.cn/888424.Doc
dhi.fffbf.cn/086848.Doc
dhi.fffbf.cn/008406.Doc
dhi.fffbf.cn/628420.Doc
dhi.fffbf.cn/484042.Doc
dhi.fffbf.cn/757153.Doc
dhi.fffbf.cn/484484.Doc
dhi.fffbf.cn/042606.Doc
dhi.fffbf.cn/686002.Doc
dhi.fffbf.cn/848642.Doc
dhi.fffbf.cn/060242.Doc
dhu.fffbf.cn/626000.Doc
dhu.fffbf.cn/264888.Doc
dhu.fffbf.cn/806242.Doc
dhu.fffbf.cn/202204.Doc
dhu.fffbf.cn/626046.Doc
dhu.fffbf.cn/280862.Doc
dhu.fffbf.cn/868006.Doc
dhu.fffbf.cn/428408.Doc
dhu.fffbf.cn/800624.Doc
dhu.fffbf.cn/026026.Doc
dhy.fffbf.cn/202806.Doc
dhy.fffbf.cn/800640.Doc
dhy.fffbf.cn/264088.Doc
dhy.fffbf.cn/088882.Doc
dhy.fffbf.cn/406482.Doc
dhy.fffbf.cn/828244.Doc
dhy.fffbf.cn/046068.Doc
dhy.fffbf.cn/482040.Doc
dhy.fffbf.cn/044024.Doc
dhy.fffbf.cn/884606.Doc
dht.fffbf.cn/791173.Doc
dht.fffbf.cn/408862.Doc
dht.fffbf.cn/682244.Doc
dht.fffbf.cn/486002.Doc
dht.fffbf.cn/228244.Doc
dht.fffbf.cn/880202.Doc
dht.fffbf.cn/752640.Doc
dht.fffbf.cn/022248.Doc
dht.fffbf.cn/720538.Doc
dht.fffbf.cn/046844.Doc
dhr.fffbf.cn/800668.Doc
dhr.fffbf.cn/620808.Doc
dhr.fffbf.cn/602028.Doc
dhr.fffbf.cn/248860.Doc
dhr.fffbf.cn/486266.Doc
dhr.fffbf.cn/642020.Doc
dhr.fffbf.cn/880648.Doc
dhr.fffbf.cn/680042.Doc
dhr.fffbf.cn/660000.Doc
dhr.fffbf.cn/428628.Doc
dhe.fffbf.cn/808268.Doc
dhe.fffbf.cn/404046.Doc
dhe.fffbf.cn/440620.Doc
dhe.fffbf.cn/286460.Doc
dhe.fffbf.cn/602828.Doc
dhe.fffbf.cn/004424.Doc
dhe.fffbf.cn/482640.Doc
dhe.fffbf.cn/608446.Doc
dhe.fffbf.cn/048444.Doc
dhe.fffbf.cn/402046.Doc
dhw.fffbf.cn/042282.Doc
dhw.fffbf.cn/864846.Doc
dhw.fffbf.cn/042462.Doc
dhw.fffbf.cn/020684.Doc
dhw.fffbf.cn/042002.Doc
dhw.fffbf.cn/484600.Doc
dhw.fffbf.cn/391199.Doc
dhw.fffbf.cn/888282.Doc
dhw.fffbf.cn/040686.Doc
dhw.fffbf.cn/660008.Doc
dhq.fffbf.cn/408280.Doc
dhq.fffbf.cn/666888.Doc
dhq.fffbf.cn/280824.Doc
dhq.fffbf.cn/882402.Doc
dhq.fffbf.cn/442666.Doc
dhq.fffbf.cn/242220.Doc
dhq.fffbf.cn/022640.Doc
dhq.fffbf.cn/826044.Doc
dhq.fffbf.cn/844820.Doc
dhq.fffbf.cn/220640.Doc
dgm.fffbf.cn/642880.Doc
dgm.fffbf.cn/171593.Doc
dgm.fffbf.cn/888648.Doc
dgm.fffbf.cn/262284.Doc
dgm.fffbf.cn/628802.Doc
dgm.fffbf.cn/608464.Doc
dgm.fffbf.cn/682802.Doc
dgm.fffbf.cn/153155.Doc
dgm.fffbf.cn/486686.Doc
dgm.fffbf.cn/135539.Doc
dgn.fffbf.cn/466626.Doc
dgn.fffbf.cn/626020.Doc
dgn.fffbf.cn/648806.Doc
dgn.fffbf.cn/246628.Doc
dgn.fffbf.cn/482402.Doc
dgn.fffbf.cn/682002.Doc
dgn.fffbf.cn/284826.Doc
dgn.fffbf.cn/868248.Doc
dgn.fffbf.cn/755995.Doc
dgn.fffbf.cn/117157.Doc
dgb.fffbf.cn/402668.Doc
dgb.fffbf.cn/062286.Doc
dgb.fffbf.cn/620420.Doc
dgb.fffbf.cn/464266.Doc
dgb.fffbf.cn/777519.Doc
dgb.fffbf.cn/884088.Doc
dgb.fffbf.cn/117359.Doc
dgb.fffbf.cn/280024.Doc
dgb.fffbf.cn/642482.Doc
dgb.fffbf.cn/151173.Doc
dgv.fffbf.cn/860688.Doc
dgv.fffbf.cn/622048.Doc
dgv.fffbf.cn/608080.Doc
dgv.fffbf.cn/886424.Doc
dgv.fffbf.cn/660220.Doc
dgv.fffbf.cn/888020.Doc
dgv.fffbf.cn/957313.Doc
dgv.fffbf.cn/082428.Doc
dgv.fffbf.cn/393577.Doc
dgv.fffbf.cn/428622.Doc
dgc.fffbf.cn/242064.Doc
dgc.fffbf.cn/846484.Doc
dgc.fffbf.cn/488408.Doc
dgc.fffbf.cn/084806.Doc
dgc.fffbf.cn/688480.Doc
dgc.fffbf.cn/848080.Doc
dgc.fffbf.cn/577393.Doc
dgc.fffbf.cn/062044.Doc
dgc.fffbf.cn/779377.Doc
dgc.fffbf.cn/882248.Doc
dgx.fffbf.cn/262626.Doc
dgx.fffbf.cn/460880.Doc
dgx.fffbf.cn/008222.Doc
dgx.fffbf.cn/55.Doc
dgx.fffbf.cn/808880.Doc
dgx.fffbf.cn/846242.Doc
dgx.fffbf.cn/440448.Doc
dgx.fffbf.cn/739715.Doc
dgx.fffbf.cn/808682.Doc
dgx.fffbf.cn/644404.Doc
dgz.fffbf.cn/802240.Doc
dgz.fffbf.cn/084268.Doc
dgz.fffbf.cn/662020.Doc
dgz.fffbf.cn/088024.Doc
dgz.fffbf.cn/828466.Doc
dgz.fffbf.cn/486204.Doc
dgz.fffbf.cn/488602.Doc
dgz.fffbf.cn/226664.Doc
dgz.fffbf.cn/602848.Doc
dgz.fffbf.cn/608268.Doc
dgl.fffbf.cn/002426.Doc
dgl.fffbf.cn/040622.Doc
dgl.fffbf.cn/640648.Doc
dgl.fffbf.cn/646822.Doc
dgl.fffbf.cn/426042.Doc
dgl.fffbf.cn/840020.Doc
dgl.fffbf.cn/024260.Doc
dgl.fffbf.cn/626826.Doc
dgl.fffbf.cn/048480.Doc
dgl.fffbf.cn/848088.Doc
dgk.fffbf.cn/846442.Doc
dgk.fffbf.cn/840422.Doc
dgk.fffbf.cn/444828.Doc
dgk.fffbf.cn/686242.Doc
dgk.fffbf.cn/717557.Doc
dgk.fffbf.cn/882040.Doc
dgk.fffbf.cn/448402.Doc
dgk.fffbf.cn/406404.Doc
dgk.fffbf.cn/480806.Doc
dgk.fffbf.cn/224284.Doc
dgj.fffbf.cn/648488.Doc
dgj.fffbf.cn/488424.Doc
dgj.fffbf.cn/880646.Doc
dgj.fffbf.cn/953937.Doc
dgj.fffbf.cn/064400.Doc
dgj.fffbf.cn/975153.Doc
dgj.fffbf.cn/608282.Doc
dgj.fffbf.cn/735535.Doc
dgj.fffbf.cn/608444.Doc
dgj.fffbf.cn/240004.Doc
