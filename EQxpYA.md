粘执堪冠兔


Linux内核模块加载机制与动态调试基础设施

内核模块加载从用户执行insmod触发系统调用init_module开始。sys_init_module读取ELF文件执行模块加载的完整流程：先通过load_module解析ELF结构、分配模块内存、处理依赖符号、调用module_init入口函数。模块文件被读入内核临时缓冲区后，load_module解析ELF节区头确定各节在模块内存中的布局。

模块内存布局通过module_alloc分配。module_alloc在x86上调用__vmalloc_node_range在MODULES_VADDR到MODULES_END区间（x86_64上位于0xffffffffa0000000到0xffffffffff000000）分配可执行虚拟内存。模块的代码段、数据段和rodata段在此区域内连续放置。ELF中的.text节映射到代码段，.rodata节映射到只读段，.data节映射到数据段。每个模块维护struct module中的module_layout记录这些段的基址和大小：

static int move_module(struct module *mod, struct load_info *info)
{
    // 分配各段layout
    mod->core_layout.size = ...
    mod->core_layout.base = module_alloc(mod->core_layout.size);
    // 逐个节区拷贝到layout
    for (i = 0; i < info->hdr->e_shnum; i++) {
        void *dest = module_region(mod, s, &mod->core_layout);
        memcpy(dest, s->sh_addr, s->sh_size);
    }
    // 设置页属性：代码段可执行，rodata只读
    set_memory_ro(layout->base, layout->size >> PAGE_SHIFT);
    set_memory_x(layout->base, layout->size >> PAGE_SHIFT);
}

模块加载的核心参数通过struct module的fields定义。name字段长度限制为MODULE_NAME_LEN（64字节）。version字段通过MODULE_VERSION宏导出。license字段通过MODULE_LICENSE宏约束可用符号范围。MODULE_LICENSE("GPL")是唯一能访问EXPORT_SYMBOL_GPL符号的声明。非GPL模块调用EXPORT_SYMBOL_GPL符号时kernel/module.c中的resolve_symbol检查license兼容性。

符号解析在模块加载阶段完成。模块ELF的.rela节区列出所有需要外部解析的符号。勤务器调用resolve_symbol遍历内核符号表（kallsyms）和已加载模块的符号表查找匹配符号。内核符号表通过struct kernel_symbol管理，kallsyms生成所有全局符号的地址映射。符号查找顺序是：先查core_kernel_symbols（vmlinux中的导出符号），再查已加载模块的符号表。查找到的符号地址写入模块GOT表：

resolve_symbol(wait_for_rcu, mod)
{
    // 查找 "__x64_sys_wait4" 的地址
    struct kernel_symbol *sym = find_symbol("__x64_sys_wait4");
    // 检查符号的license是否允许当前模块使用
    if (sym->license_gpl && mod->license != GPL)
        return -EPERM;
    // 写入符号值到模块的符号表项
    *(unsigned long *)loc = sym->value;
    // 增加导出模块的引用计数防止卸载
    try_module_get(sym->module);
}

符号解析失败（未定义符号）时load_module调用resolve_symbol返回-EINVAL，模块加载无法完成。模块对所有依赖模块增加引用计数：导出一个符号给其他模块使用时，该符号的模块module->refcnt不为零，阻止卸载。

引用计数的操作在try_module_get和module_put中完成。try_module_get原子增加mod->refcnt，如果mod->state已经是MODULE_STATE_GOING则返回失败。module_put减少refcnt，如果refcnt降为零且模块在卸载队列（MODULE_STATE_GOING）中则执行module_cleanup。refcnt保护模块在代码执行期间不被卸载：

static inline bool try_module_get(struct module *module)
{
    bool ret = true;
    if (module) {
        preempt_disable();
        if (likely(module_is_live(module)))
            atomic_inc(&module->refcnt);
        else
            ret = false;
        preempt_enable();
    }
    return ret;
}

模块初始化的入口在do_init_module中。load_module返回后do_init_module将模块状态设置为MODULE_STATE_COMING，调用complete_all通知MODULE_STATE_COMING的waiters。然后执行module->init函数指针，执行失败时调用module_put清理。成功后将模块状态设为MODULE_STATE_LIVE并添加到已加载模块链表：

static noinline int do_init_module(struct module *mod)
{
    // 设置状态为COMING，让依赖的子系统知道模块正在初始化
    mod->state = MODULE_STATE_COMING;
    // 调用模块的init函数
    ret = mod->init();
    if (ret < 0) {
        // 释放模块资源
        free_module(mod);
        return ret;
    }
    // 将模块加入全局链表
    list_add_rcu(&mod->list, &modules);
    mod->state = MODULE_STATE_LIVE;
    return 0;
}

模块初始化完成前模块已列入依赖系统的视图中。文件系统模块的init函数会调用register_filesystem，该函数将file_system_type加入全局file_systems链表。设备驱动模块的init调用pci_register_driver或platform_driver_register。当模块init函数在这些注册调用中失败时，需要cleanup已注册的子系统资源。

模块加载后通过/sys/module/<name>/parameters目录暴露模块参数。module_param宏定义模块参数后，内核在sysfs中创建对应的读写文件。参数类型包括byte、short、int、uint、long、ulong、charp、bool和invbool。参数值通过param_set_XXX和param_get_XXX回调与sysfs文件交互。module_param_cb允许自定义读写回调实现参数的侧效果：

module_param_cb(enable_interrupt, &interrupt_ops, &enable, 0644);
// 写入时触发中断使能/禁用的侧效果

模块参数值保存在模块的ELF数据段中，insmod传入参数时load_module解析参数字符串填充segment中的值。调用init函数前参数已准备完成。

模块依赖关系由modprobe自动处理。modprobe读取/lib/modules/$(uname -r)/modules.dep文件确定依赖链。当insmod一个模块时modprobe按拓扑顺序加载所有依赖模块。depmod在安装内核时扫描所有模块的符号导出和引用生成modules.dep。模块间循环依赖通过符号导出解决：如果模块A导出符号供B使用，A必须先加载。

模块卸载路径在delete_module系统调用中。sys_delete_module设置模块状态MODULE_STATE_GOING并调用module_put。等待refcnt降为零后执行module_exit回调。如果引用计数在超时后不为零返回-EBUSY。强制卸载（MODULE_FORCE_UNLOAD）绕过refcnt检查但不推荐。module_exit调用完成后释放模块的module_layout空间并清理符号表：

SYSCALL_DEFINE2(delete_module, const char __user *, name_user, unsigned int, flags)
{
    // 检查模块状态
    if (mod->state != MODULE_STATE_LIVE)
        return -EINVAL;
    // 如果可以强制卸载，跳过refcnt检查
    if (!(flags & O_NONBLOCK) && !try_module_get(mod))
        return -EWOULDBLOCK;
    // 设置Going状态并清理
    mod->state = MODULE_STATE_GOING;
    mod->exit();
    free_module(mod);
    return 0;
}

强行卸载正在被使用的模块导致内核崩溃。module exit函数中对硬件的操作可能导致系统不稳定，因为硬件可能被驱动正在使用（如正在传输DMA的数据）。

ftrace是内核的第一动态跟踪设施。ftrace通过编译时插入的mcount/fentry调用在函数入口和出口插入跟踪点。内核启动时ftrace_init遍历__ftrace_plt_entries填充ftrace_pages。函数跟踪通过set_ftrace_filter写入跟踪列表，ftrace_replace_code将跟踪点指令替换为跳转到ftrace_caller：

static int ftrace_make_call(struct dyn_ftrace *rec, unsigned long addr)
{
    unsigned char op[] = {
        0xe8, 0, 0, 0, 0  // call rel32
    };
    *(unsigned long *)&op[1] = addr - rec->ip - 5;
    // 调用text_poke动态修改代码段的指令
    text_poke(rec->ip, op, 5);
    return 0;
}

ftrace的函数跟踪器通过tracing_on和set_ftrace_filter控制。trace由ring buffer存储。用户通过trace_pipe读取实时流或trace文件读取历史记录。函数跟踪的每条记录包括进程PID、CPU、时间戳、函数名（从kallsyms解析）。ftrace自带的函数调用延迟统计在trace打印中以微秒精度显示。

kprobe是二进制级别的动态插桩工具。kprobe在指定地址写入断点指令（x86上是int3）。当CPU执行到断点时触发do_int3，arch_uprobe_page_fault将控制转到kprobe_handler。kprobe_handler保存陷阱上下文并执行用户注册的pre_handler回调。回调返回后单步执行原始指令（通过设置TF标志或模拟指令执行），完成后调用post_handler：

int register_kprobe(struct kprobe *kp)
{
    // 保存原始指令
    kp->opcode = *(kprobe_opcode_t *)kp->addr;
    // 写入int3指令（0xCC）
    arch_arm_kprobe(kp);
    // 将kprobe加入哈希表
    kprobe_add(kp);
}

kprobe_handler的调用栈：do_int3->do_int3_user/kprobe_int3_handler->kprobe_handler。单步执行的恢复使用kprobe_resume_from_singlestep，当单步完成时继续原始指令流的执行。kprobe的pre_handler运行在中断上下文中，禁止可睡眠操作。post_handler在单步执行后的上下文中运行，仍不能获取mmap_sem等可睡眠锁。

jprobe基于kprobe实现但用于替代被探测函数的参数获取。jprobe被6.x内核标记为deprecated，建议直接使用kprobe读取参数寄存器。kretprobe通过设置kprobe在函数入口处拦截执行，在函数返回点使用trampoline记录返回值。kretprobe的maxactive设置最大同时跟踪的返回数，超出时entry handler被丢弃。

tracepoint是编译时插入的静态跟踪点，比kprobe稳定但需要内核开发者手动插入。tracepoint通过DECLARE_TRACE和DEFINE_TRACE宏定义，使用时展开为static_key控制的跳转。当tracepoint被关闭时，跳转指令跳过跟踪函数体零开销。当开启时通过rcu保护的跟踪函数列表分发事件：

#define DECLARE_TRACE(name, proto, args)                    \
    static inline void trace_##name(proto)                  \
    {                                                       \
        if (static_key_false(&__tracepoint_##name.key))     \
            __traceiter_##name(args);                       \
    }

tracepoint的注册通过register_trace_##name函数添加probe函数到__tracepoint_##name的funcs列表。probe函数调用次序取决于注册时间。tracepoint的批量操作可以使用tracepoint_synchronize_unregister等待所有追踪上下文退出。

BPF程序通过bpf系统调用加载到内核中执行。bpf_check执行校验器对BPF指令逐条分析。校验器使用bpf_checker的状态机模拟每条指令的执行路径，跟踪寄存器类型（BPF_REG_0是返回值、BPF_REG_6~9是纯量寄存器等）。模拟过程中追踪栈的slot类型并验证内存访问不超过栈大小。bpf_prog_select_func将BPF程序翻译为x86机器码，使用BPF_JIT_ALWAYS_ON默认启用JIT编译：

int bpf_check(struct bpf_prog **prog, union bpf_attr *attr)
{
    // 深度优先遍历指令序列
    // 对每条指令检查操作数类型匹配
    // 检查栈访问范围、指针运算合法性
    // 检查helper函数调用参数约束
    // 检测不可达代码和循环（早期内核不支持循环）
    return ret;
}

BPF helper函数（如bpf_get_current_pid_tgid和bpf_trace_printk）通过注册宏声明参数类型和返回值约束。bpf_prog->aux->attach_btf决定程序类型。perf_event类型的BPF程序通过perf_event_open附加到硬件事件。tracepoint类型BPF程序通过perf_event_open附加到tracepoint。XDP BPF程序通过netdev的xdp_prog附加。

debugfs是模块开发者自建调试接口的轻量方案。debugfs_create_file在debugfs中创建blob、u8、u16、u32、u64和bool类型的文件。debugfs的fops使用simple_attr进行的封装。用户层读写debugfs文件时触发模块实现的回调。debugfs相比sysfs更灵活但不提供ABI稳定性。模块卸载时debugfs_create_dir创建的目录和文件通过debugfs_remove递归清理。debugfs的默认挂载点是/sys/kernel/debug，由debugfs自动挂载。

printk的日志级别通过控制台loglevel和默认loglevel过滤。echo "8" > /proc/sys/kernel/printk设置控制台loglevel级别为8（KERN_DEBUG及以上都输出）。模块中使用pr_debug和pr_info的日志宏，编译时将DEBUG定义或开启dynamic debug之后才能看到输出。dynamic debug通过/sys/kernel/debug/dynamic_debug/control控制每个文件的pr_debug开关：

echo "file $filename +p" > /sys/kernel/debug/dynamic_debug/control

dynamic debug的实现为每个pr_debug调用点生成struct _ddebug记录在__dyndbg节区。写control文件时遍历__start___verbose到__stop___verbose之间的所有_ddebug条目，匹配filename后修改flags字段的控制位。

oops后的堆栈打印通过show_stack实现。show_stack遍历struct pt_regs中的栈帧和返回地址。模块的栈回溯需要kallsyms支持。kallsyms中模块符号通过module_kallsyms_lookup_name查找。新增或移除模块时kallsyms的符号表通过update_modules_kallsyms更新。dump_stack在调试期间打印当前栈的调用链。

内核模块开发使用modprobe的blacklist机制可以暂时禁用模块。echo "blacklist $module" > /etc/modprobe.d/blacklist.conf后initramfs重建时不会加载该模块。模块在加载顺序中的依赖由depmod计算并写入modules.dep。

玫咨叹吃槐呵葡陶量吐孤压犊纲铰

m.qhv.kxnxh.cn/06480.Doc
m.qhv.kxnxh.cn/66462.Doc
m.qhv.kxnxh.cn/64008.Doc
m.qhv.kxnxh.cn/80800.Doc
m.qhv.kxnxh.cn/35573.Doc
m.qhv.kxnxh.cn/64200.Doc
m.qhv.kxnxh.cn/95917.Doc
m.qhv.kxnxh.cn/68620.Doc
m.qhv.kxnxh.cn/80220.Doc
m.qhv.kxnxh.cn/02644.Doc
m.qhv.kxnxh.cn/46860.Doc
m.qhv.kxnxh.cn/57713.Doc
m.qhv.kxnxh.cn/00826.Doc
m.qhv.kxnxh.cn/62088.Doc
m.qhv.kxnxh.cn/22242.Doc
m.qhv.kxnxh.cn/82080.Doc
m.qhv.kxnxh.cn/00682.Doc
m.qhc.kxnxh.cn/80884.Doc
m.qhc.kxnxh.cn/82464.Doc
m.qhc.kxnxh.cn/64424.Doc
m.qhc.kxnxh.cn/24006.Doc
m.qhc.kxnxh.cn/24806.Doc
m.qhc.kxnxh.cn/57999.Doc
m.qhc.kxnxh.cn/62024.Doc
m.qhc.kxnxh.cn/71997.Doc
m.qhc.kxnxh.cn/42660.Doc
m.qhc.kxnxh.cn/02206.Doc
m.qhc.kxnxh.cn/37711.Doc
m.qhc.kxnxh.cn/95959.Doc
m.qhc.kxnxh.cn/68244.Doc
m.qhc.kxnxh.cn/04480.Doc
m.qhc.kxnxh.cn/26664.Doc
m.qhc.kxnxh.cn/84462.Doc
m.qhc.kxnxh.cn/88244.Doc
m.qhc.kxnxh.cn/04644.Doc
m.qhc.kxnxh.cn/22008.Doc
m.qhc.kxnxh.cn/40406.Doc
m.qhx.kxnxh.cn/99557.Doc
m.qhx.kxnxh.cn/64604.Doc
m.qhx.kxnxh.cn/26804.Doc
m.qhx.kxnxh.cn/48062.Doc
m.qhx.kxnxh.cn/24224.Doc
m.qhx.kxnxh.cn/48464.Doc
m.qhx.kxnxh.cn/24224.Doc
m.qhx.kxnxh.cn/77555.Doc
m.qhx.kxnxh.cn/80640.Doc
m.qhx.kxnxh.cn/84886.Doc
m.qhx.kxnxh.cn/08842.Doc
m.qhx.kxnxh.cn/80684.Doc
m.qhx.kxnxh.cn/66206.Doc
m.qhx.kxnxh.cn/37193.Doc
m.qhx.kxnxh.cn/44048.Doc
m.qhx.kxnxh.cn/26444.Doc
m.qhx.kxnxh.cn/99517.Doc
m.qhx.kxnxh.cn/35391.Doc
m.qhx.kxnxh.cn/95119.Doc
m.qhx.kxnxh.cn/51179.Doc
m.qhz.kxnxh.cn/88208.Doc
m.qhz.kxnxh.cn/64044.Doc
m.qhz.kxnxh.cn/91395.Doc
m.qhz.kxnxh.cn/40220.Doc
m.qhz.kxnxh.cn/84642.Doc
m.qhz.kxnxh.cn/22866.Doc
m.qhz.kxnxh.cn/19739.Doc
m.qhz.kxnxh.cn/22462.Doc
m.qhz.kxnxh.cn/22804.Doc
m.qhz.kxnxh.cn/62844.Doc
m.qhz.kxnxh.cn/40866.Doc
m.qhz.kxnxh.cn/26248.Doc
m.qhz.kxnxh.cn/80422.Doc
m.qhz.kxnxh.cn/46866.Doc
m.qhz.kxnxh.cn/80462.Doc
m.qhz.kxnxh.cn/40088.Doc
m.qhz.kxnxh.cn/42846.Doc
m.qhz.kxnxh.cn/08842.Doc
m.qhz.kxnxh.cn/53793.Doc
m.qhz.kxnxh.cn/55331.Doc
m.qhl.kxnxh.cn/48640.Doc
m.qhl.kxnxh.cn/82464.Doc
m.qhl.kxnxh.cn/46400.Doc
m.qhl.kxnxh.cn/84048.Doc
m.qhl.kxnxh.cn/20060.Doc
m.qhl.kxnxh.cn/06448.Doc
m.qhl.kxnxh.cn/04864.Doc
m.qhl.kxnxh.cn/93931.Doc
m.qhl.kxnxh.cn/04862.Doc
m.qhl.kxnxh.cn/46040.Doc
m.qhl.kxnxh.cn/22204.Doc
m.qhl.kxnxh.cn/86282.Doc
m.qhl.kxnxh.cn/31139.Doc
m.qhl.kxnxh.cn/17115.Doc
m.qhl.kxnxh.cn/26822.Doc
m.qhl.kxnxh.cn/84624.Doc
m.qhl.kxnxh.cn/64206.Doc
m.qhl.kxnxh.cn/82646.Doc
m.qhl.kxnxh.cn/53735.Doc
m.qhl.kxnxh.cn/42264.Doc
m.qhk.kxnxh.cn/62040.Doc
m.qhk.kxnxh.cn/28286.Doc
m.qhk.kxnxh.cn/62066.Doc
m.qhk.kxnxh.cn/26026.Doc
m.qhk.kxnxh.cn/93195.Doc
m.qhk.kxnxh.cn/26008.Doc
m.qhk.kxnxh.cn/39111.Doc
m.qhk.kxnxh.cn/95331.Doc
m.qhk.kxnxh.cn/40044.Doc
m.qhk.kxnxh.cn/15737.Doc
m.qhk.kxnxh.cn/00246.Doc
m.qhk.kxnxh.cn/60220.Doc
m.qhk.kxnxh.cn/86048.Doc
m.qhk.kxnxh.cn/44220.Doc
m.qhk.kxnxh.cn/80608.Doc
m.qhk.kxnxh.cn/04042.Doc
m.qhk.kxnxh.cn/02884.Doc
m.qhk.kxnxh.cn/08286.Doc
m.qhk.kxnxh.cn/57315.Doc
m.qhk.kxnxh.cn/22062.Doc
m.qhj.kxnxh.cn/28062.Doc
m.qhj.kxnxh.cn/20228.Doc
m.qhj.kxnxh.cn/99553.Doc
m.qhj.kxnxh.cn/66668.Doc
m.qhj.kxnxh.cn/68404.Doc
m.qhj.kxnxh.cn/48044.Doc
m.qhj.kxnxh.cn/04200.Doc
m.qhj.kxnxh.cn/20466.Doc
m.qhj.kxnxh.cn/46068.Doc
m.qhj.kxnxh.cn/44288.Doc
m.qhj.kxnxh.cn/66086.Doc
m.qhj.kxnxh.cn/06480.Doc
m.qhj.kxnxh.cn/66480.Doc
m.qhj.kxnxh.cn/91775.Doc
m.qhj.kxnxh.cn/19595.Doc
m.qhj.kxnxh.cn/62606.Doc
m.qhj.kxnxh.cn/08828.Doc
m.qhj.kxnxh.cn/06260.Doc
m.qhj.kxnxh.cn/86486.Doc
m.qhj.kxnxh.cn/64426.Doc
m.qhh.kxnxh.cn/86244.Doc
m.qhh.kxnxh.cn/60464.Doc
m.qhh.kxnxh.cn/00444.Doc
m.qhh.kxnxh.cn/55751.Doc
m.qhh.kxnxh.cn/62868.Doc
m.qhh.kxnxh.cn/22806.Doc
m.qhh.kxnxh.cn/59937.Doc
m.qhh.kxnxh.cn/42848.Doc
m.qhh.kxnxh.cn/44884.Doc
m.qhh.kxnxh.cn/53395.Doc
m.qhh.kxnxh.cn/66042.Doc
m.qhh.kxnxh.cn/84286.Doc
m.qhh.kxnxh.cn/06248.Doc
m.qhh.kxnxh.cn/48268.Doc
m.qhh.kxnxh.cn/19517.Doc
m.qhh.kxnxh.cn/84060.Doc
m.qhh.kxnxh.cn/28080.Doc
m.qhh.kxnxh.cn/08266.Doc
m.qhh.kxnxh.cn/04200.Doc
m.qhh.kxnxh.cn/20044.Doc
m.qhg.kxnxh.cn/42082.Doc
m.qhg.kxnxh.cn/53939.Doc
m.qhg.kxnxh.cn/40266.Doc
m.qhg.kxnxh.cn/86828.Doc
m.qhg.kxnxh.cn/62046.Doc
m.qhg.kxnxh.cn/00493.Doc
m.qhg.kxnxh.cn/79595.Doc
m.qhg.kxnxh.cn/55753.Doc
m.qhg.kxnxh.cn/80806.Doc
m.qhg.kxnxh.cn/68626.Doc
m.qhg.kxnxh.cn/17771.Doc
m.qhg.kxnxh.cn/37579.Doc
m.qhg.kxnxh.cn/55731.Doc
m.qhg.kxnxh.cn/00228.Doc
m.qhg.kxnxh.cn/20046.Doc
m.qhg.kxnxh.cn/37191.Doc
m.qhg.kxnxh.cn/86204.Doc
m.qhg.kxnxh.cn/06482.Doc
m.qhg.kxnxh.cn/02608.Doc
m.qhg.kxnxh.cn/04840.Doc
m.qhf.kxnxh.cn/26828.Doc
m.qhf.kxnxh.cn/46884.Doc
m.qhf.kxnxh.cn/62046.Doc
m.qhf.kxnxh.cn/82082.Doc
m.qhf.kxnxh.cn/28244.Doc
m.qhf.kxnxh.cn/19355.Doc
m.qhf.kxnxh.cn/79719.Doc
m.qhf.kxnxh.cn/60446.Doc
m.qhf.kxnxh.cn/00624.Doc
m.qhf.kxnxh.cn/44620.Doc
m.qhf.kxnxh.cn/20200.Doc
m.qhf.kxnxh.cn/39391.Doc
m.qhf.kxnxh.cn/42640.Doc
m.qhf.kxnxh.cn/46206.Doc
m.qhf.kxnxh.cn/22460.Doc
m.qhf.kxnxh.cn/26882.Doc
m.qhf.kxnxh.cn/95397.Doc
m.qhf.kxnxh.cn/88620.Doc
m.qhf.kxnxh.cn/15399.Doc
m.qhf.kxnxh.cn/68022.Doc
m.qhd.kxnxh.cn/22006.Doc
m.qhd.kxnxh.cn/33917.Doc
m.qhd.kxnxh.cn/88286.Doc
m.qhd.kxnxh.cn/42408.Doc
m.qhd.kxnxh.cn/82426.Doc
m.qhd.kxnxh.cn/86868.Doc
m.qhd.kxnxh.cn/40882.Doc
m.qhd.kxnxh.cn/40206.Doc
m.qhd.kxnxh.cn/40624.Doc
m.qhd.kxnxh.cn/68426.Doc
m.qhd.kxnxh.cn/00260.Doc
m.qhd.kxnxh.cn/86604.Doc
m.qhd.kxnxh.cn/44660.Doc
m.qhd.kxnxh.cn/46288.Doc
m.qhd.kxnxh.cn/44022.Doc
m.qhd.kxnxh.cn/62242.Doc
m.qhd.kxnxh.cn/44420.Doc
m.qhd.kxnxh.cn/86020.Doc
m.qhd.kxnxh.cn/68826.Doc
m.qhd.kxnxh.cn/84204.Doc
m.qhs.kxnxh.cn/64206.Doc
m.qhs.kxnxh.cn/44480.Doc
m.qhs.kxnxh.cn/44680.Doc
m.qhs.kxnxh.cn/40428.Doc
m.qhs.kxnxh.cn/99715.Doc
m.qhs.kxnxh.cn/08646.Doc
m.qhs.kxnxh.cn/15551.Doc
m.qhs.kxnxh.cn/04026.Doc
m.qhs.kxnxh.cn/24066.Doc
m.qhs.kxnxh.cn/91595.Doc
m.qhs.kxnxh.cn/44804.Doc
m.qhs.kxnxh.cn/84802.Doc
m.qhs.kxnxh.cn/86842.Doc
m.qhs.kxnxh.cn/42846.Doc
m.qhs.kxnxh.cn/86404.Doc
m.qhs.kxnxh.cn/46062.Doc
m.qhs.kxnxh.cn/93719.Doc
m.qhs.kxnxh.cn/19717.Doc
m.qhs.kxnxh.cn/00624.Doc
m.qhs.kxnxh.cn/62660.Doc
m.qha.kxnxh.cn/80266.Doc
m.qha.kxnxh.cn/24688.Doc
m.qha.kxnxh.cn/08086.Doc
m.qha.kxnxh.cn/66446.Doc
m.qha.kxnxh.cn/59593.Doc
m.qha.kxnxh.cn/82040.Doc
m.qha.kxnxh.cn/06024.Doc
m.qha.kxnxh.cn/20200.Doc
m.qha.kxnxh.cn/51511.Doc
m.qha.kxnxh.cn/06442.Doc
m.qha.kxnxh.cn/77579.Doc
m.qha.kxnxh.cn/91331.Doc
m.qha.kxnxh.cn/00008.Doc
m.qha.kxnxh.cn/08400.Doc
m.qha.kxnxh.cn/20040.Doc
m.qha.kxnxh.cn/46428.Doc
m.qha.kxnxh.cn/82622.Doc
m.qha.kxnxh.cn/80226.Doc
m.qha.kxnxh.cn/22800.Doc
m.qha.kxnxh.cn/66082.Doc
m.qhp.kxnxh.cn/26208.Doc
m.qhp.kxnxh.cn/93731.Doc
m.qhp.kxnxh.cn/28448.Doc
m.qhp.kxnxh.cn/93975.Doc
m.qhp.kxnxh.cn/68466.Doc
m.qhp.kxnxh.cn/28444.Doc
m.qhp.kxnxh.cn/84800.Doc
m.qhp.kxnxh.cn/22602.Doc
m.qhp.kxnxh.cn/62228.Doc
m.qhp.kxnxh.cn/24804.Doc
m.qhp.kxnxh.cn/28286.Doc
m.qhp.kxnxh.cn/26440.Doc
m.qhp.kxnxh.cn/20420.Doc
m.qhp.kxnxh.cn/44424.Doc
m.qhp.kxnxh.cn/79733.Doc
m.qhp.kxnxh.cn/93195.Doc
m.qhp.kxnxh.cn/15939.Doc
m.qhp.kxnxh.cn/42822.Doc
m.qhp.kxnxh.cn/20040.Doc
m.qhp.kxnxh.cn/64820.Doc
m.qho.kxnxh.cn/53375.Doc
m.qho.kxnxh.cn/82466.Doc
m.qho.kxnxh.cn/26828.Doc
m.qho.kxnxh.cn/73155.Doc
m.qho.kxnxh.cn/86686.Doc
m.qho.kxnxh.cn/93371.Doc
m.qho.kxnxh.cn/28826.Doc
m.qho.kxnxh.cn/39915.Doc
m.qho.kxnxh.cn/00680.Doc
m.qho.kxnxh.cn/24644.Doc
m.qho.kxnxh.cn/40622.Doc
m.qho.kxnxh.cn/66260.Doc
m.qho.kxnxh.cn/91537.Doc
m.qho.kxnxh.cn/99171.Doc
m.qho.kxnxh.cn/73911.Doc
m.qho.kxnxh.cn/79317.Doc
m.qho.kxnxh.cn/17311.Doc
m.qho.kxnxh.cn/46662.Doc
m.qho.kxnxh.cn/66260.Doc
m.qho.kxnxh.cn/55513.Doc
m.qhi.kxnxh.cn/06062.Doc
m.qhi.kxnxh.cn/19531.Doc
m.qhi.kxnxh.cn/62242.Doc
m.qhi.kxnxh.cn/04642.Doc
m.qhi.kxnxh.cn/80448.Doc
m.qhi.kxnxh.cn/46884.Doc
m.qhi.kxnxh.cn/28206.Doc
m.qhi.kxnxh.cn/46464.Doc
m.qhi.kxnxh.cn/37115.Doc
m.qhi.kxnxh.cn/82480.Doc
m.qhi.kxnxh.cn/55717.Doc
m.qhi.kxnxh.cn/53315.Doc
m.qhi.kxnxh.cn/59935.Doc
m.qhi.kxnxh.cn/02662.Doc
m.qhi.kxnxh.cn/51171.Doc
m.qhi.kxnxh.cn/66228.Doc
m.qhi.kxnxh.cn/40224.Doc
m.qhi.kxnxh.cn/46044.Doc
m.qhi.kxnxh.cn/42080.Doc
m.qhi.kxnxh.cn/62488.Doc
m.qhu.kxnxh.cn/82824.Doc
m.qhu.kxnxh.cn/00808.Doc
m.qhu.kxnxh.cn/68828.Doc
m.qhu.kxnxh.cn/97355.Doc
m.qhu.kxnxh.cn/77171.Doc
m.qhu.kxnxh.cn/00686.Doc
m.qhu.kxnxh.cn/88866.Doc
m.qhu.kxnxh.cn/77331.Doc
m.qhu.kxnxh.cn/91537.Doc
m.qhu.kxnxh.cn/64400.Doc
m.qhu.kxnxh.cn/46680.Doc
m.qhu.kxnxh.cn/66624.Doc
m.qhu.kxnxh.cn/64440.Doc
m.qhu.kxnxh.cn/20622.Doc
m.qhu.kxnxh.cn/82820.Doc
m.qhu.kxnxh.cn/62402.Doc
m.qhu.kxnxh.cn/33137.Doc
m.qhu.kxnxh.cn/66006.Doc
m.qhu.kxnxh.cn/53917.Doc
m.qhu.kxnxh.cn/66284.Doc
m.qhy.kxnxh.cn/42806.Doc
m.qhy.kxnxh.cn/80004.Doc
m.qhy.kxnxh.cn/66400.Doc
m.qhy.kxnxh.cn/60646.Doc
m.qhy.kxnxh.cn/88820.Doc
m.qhy.kxnxh.cn/11939.Doc
m.qhy.kxnxh.cn/00640.Doc
m.qhy.kxnxh.cn/66464.Doc
m.qhy.kxnxh.cn/22024.Doc
m.qhy.kxnxh.cn/95799.Doc
m.qhy.kxnxh.cn/48224.Doc
m.qhy.kxnxh.cn/99311.Doc
m.qhy.kxnxh.cn/51331.Doc
m.qhy.kxnxh.cn/73737.Doc
m.qhy.kxnxh.cn/11399.Doc
m.qhy.kxnxh.cn/22226.Doc
m.qhy.kxnxh.cn/00466.Doc
m.qhy.kxnxh.cn/99151.Doc
m.qhy.kxnxh.cn/00866.Doc
m.qhy.kxnxh.cn/28264.Doc
m.qht.kxnxh.cn/04884.Doc
m.qht.kxnxh.cn/62400.Doc
m.qht.kxnxh.cn/26208.Doc
m.qht.kxnxh.cn/60002.Doc
m.qht.kxnxh.cn/7.Doc
m.qht.kxnxh.cn/84860.Doc
m.qht.kxnxh.cn/80482.Doc
m.qht.kxnxh.cn/26064.Doc
m.qht.kxnxh.cn/44466.Doc
m.qht.kxnxh.cn/84606.Doc
m.qht.kxnxh.cn/88286.Doc
m.qht.kxnxh.cn/31315.Doc
m.qht.kxnxh.cn/15571.Doc
m.qht.kxnxh.cn/62426.Doc
m.qht.kxnxh.cn/88206.Doc
m.qht.kxnxh.cn/08686.Doc
m.qht.kxnxh.cn/79391.Doc
m.qht.kxnxh.cn/06628.Doc
m.qht.kxnxh.cn/68866.Doc
m.qht.kxnxh.cn/20406.Doc
m.qhr.kxnxh.cn/48268.Doc
m.qhr.kxnxh.cn/88466.Doc
m.qhr.kxnxh.cn/44044.Doc
m.qhr.kxnxh.cn/60226.Doc
m.qhr.kxnxh.cn/53537.Doc
m.qhr.kxnxh.cn/99535.Doc
m.qhr.kxnxh.cn/13957.Doc
m.qhr.kxnxh.cn/35371.Doc
m.qhr.kxnxh.cn/48606.Doc
m.qhr.kxnxh.cn/44642.Doc
m.qhr.kxnxh.cn/91517.Doc
m.qhr.kxnxh.cn/95197.Doc
m.qhr.kxnxh.cn/06082.Doc
m.qhr.kxnxh.cn/06220.Doc
m.qhr.kxnxh.cn/20440.Doc
m.qhr.kxnxh.cn/66446.Doc
m.qhr.kxnxh.cn/80200.Doc
m.qhr.kxnxh.cn/59179.Doc
