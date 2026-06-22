缆墓曰沧运


Linux sys_mmap vm_mmap_pgoff与get_unmapped_area

mmap(2) 系统调用在 x86_64 架构上的入口是 ksys_mmap_pgoff，统一处理匿名映射和文件映射：

```c
unsigned long ksys_mmap_pgoff(unsigned long addr, unsigned long len,
                              unsigned long prot, unsigned long flags,
                              unsigned long fd, unsigned long pgoff)
{
    struct file *file = NULL;
    unsigned long retval = -EBADF;

    if (!(flags & MAP_ANONYMOUS)) {
        audit_mmap_fd(fd, flags);
        file = fget(fd);
        if (!file)
            goto out;
        if (is_file_hugepages(file))
            len = ALIGN(len, huge_page_size(hstate_file(file)));
    } else if (flags & MAP_HUGETLB) {
        ...
    }

    flags &= ~(MAP_EXECUTABLE | MAP_DENYWRITE);

    retval = vm_mmap_pgoff(file, addr, len, prot, flags, pgoff);
out:
    if (file)
        fput(file);
    return retval;
}
```

对于文件映射，ksys_mmap_pgoff 通过 fget 获取 struct file 并验证。对于 hugetlbfs 映射，长度被对齐到 huge page 大小。

vm_mmap_pgoff 是 VMM 层映射入口，负责分配新的虚拟地址空间并建立页表映射：

```c
unsigned long vm_mmap_pgoff(struct file *file, unsigned long addr,
                             unsigned long len, unsigned long prot,
                             unsigned long flag, unsigned long pgoff)
{
    unsigned long ret;
    struct mm_struct *mm = current->mm;

    if (!file) {
        if (mmap_is_ia32())
            addr = round_hint_to_min(addr);
    }

    ret = security_mmap_file(file, prot, flag);
    if (ret)
        return ret;

    ret = mmap_region(file, addr, len, VM_FLAGS, pgoff);
    if (ret)
        return ret;

    return ret;
}
```

security_mmap_file 由 LSM 模块实现：SELinux 在此基于 file 的 inode 安全上下文和进程的安全上下文进行决策，如果返回非零则映射被拒绝。

mmap_region 是映射建立的核心函数。它首先检查请求的地址范围是否与现有 VMA 重叠，如果重叠则通过 do_munmap 先拆除旧映射：

```c
unsigned long mmap_region(struct file *file, unsigned long addr,
                           unsigned long len, vm_flags_t vm_flags,
                           unsigned long pgoff)
{
    struct mm_struct *mm = current->mm;
    struct vm_area_struct *vma, *prev;
    int error;

    if (mm->map_count > sysctl_max_map_count)
        return -ENOMEM;

    addr = get_unmapped_area(file, addr, len, pgoff, vm_flags);
    if (IS_ERR_VALUE(addr))
        return addr;
    ...
}
```

get_unmapped_area 负责在进程的地址空间中找到一块空闲的连续虚拟地址区域：

```c
unsigned long get_unmapped_area(struct file *file, unsigned long addr,
                                 unsigned long len, unsigned long pgoff,
                                 unsigned long flags)
{
    unsigned long (*get_area)(struct file *, unsigned long,
                               unsigned long, unsigned long, unsigned long);

    get_area = current->mm->get_unmapped_area;

    if (file) {
        if (file->f_op->get_unmapped_area)
            get_area = file->f_op->get_unmapped_area;
    } else if (flags & MAP_SHARED) {
        ...
        pgoff = 0;
        get_area = shmem_get_unmapped_area;
    }

    return get_area(file, addr, len, pgoff, flags);
}
```

对于文件映射，get_unmapped_area 优先使用文件系统自定义的实现。ext4 和 xfs 通过 file_operations 中的 get_unmapped_area 提供自己的算法。默认情况下使用 arch_get_unmapped_area_topdown（x86_64 使用 top-down 分配策略）：

```c
unsigned long arch_get_unmapped_area_topdown(struct file *filp,
        unsigned long addr, unsigned long len, unsigned long pgoff,
        unsigned long flags)
{
    struct vm_area_struct *vma;
    struct mm_struct *mm = current->mm;
    unsigned long task_size = STACK_TOP_MAX;

    if (flags & MAP_FIXED)
        return addr;

    if (addr) {
        addr = PAGE_ALIGN(addr);
        vma = find_vma_intersection(mm, addr, addr + len);
        if (!vma)
            return addr;
    }

    if (mm->mmap_base > len) {
        addr = mm->mmap_base - len;
    } else {
        addr = PAGE_ALIGN(TASK_UNMAPPED_BASE);
    }

    while (true) {
        vma = find_vma_prev(mm, addr, &prev);
        if (!vma || addr + len <= vm_start_gap(vma)) {
            if (!prev || addr >= vm_end_gap(prev))
                return addr;
        }
        addr = vm_start_gap(vma) - len;
        if (addr < mmap_min_addr)
            break;
    }

    if (addr < mmap_min_addr)
        return -ENOMEM;

    return addr;
}
```

算法从 mm->mmap_base（默认为 STACK_TOP_MAX 减去一页）向下搜索，每次尝试检查候选地址是否与现有 VMA 冲突，直到找到足够大的空闲区间。

mmap_region 完成地址分配后调用 call_mmap（即 file->f_op->mmap）建立文件与进程地址空间的关联。对于 ext4，ext4_file_mmap 仅在 inode 上设置了 DAX 标志并且在 DAX 设备上时才需要特殊处理，否则使用 generic_file_mmap 创建一个文件映射页的 VMA：

```c
int generic_file_mmap(struct file *file, struct vm_area_struct *vma)
{
    struct address_space *mapping = file->f_mapping;

    if (!mapping->a_ops->readpage)
        return -ENOEXEC;
    file_accessed(file);
    vma->vm_ops = &generic_file_vm_ops;
    return 0;
}
```

generic_file_mmap 设置 vm_ops 为 generic_file_vm_ops，该操作集包含 .fault 回调 filemap_fault，在缺页时通过 address_space_operations->readpage 从文件系统读取数据到 page cache。
呈灼骄一善叵萌恿酒蠢奔寄酚傲号

hwm.vwbnt.cn/006080.Doc
hwn.vwbnt.cn/062624.Doc
hwn.vwbnt.cn/459214.Doc
hwn.vwbnt.cn/408484.Doc
hwn.vwbnt.cn/406644.Doc
hwn.vwbnt.cn/240204.Doc
hwn.vwbnt.cn/602806.Doc
hwn.vwbnt.cn/391115.Doc
hwn.vwbnt.cn/684844.Doc
hwn.vwbnt.cn/094444.Doc
hwn.vwbnt.cn/117713.Doc
hwb.vwbnt.cn/268486.Doc
hwb.vwbnt.cn/282248.Doc
hwb.vwbnt.cn/573971.Doc
hwb.vwbnt.cn/046848.Doc
hwb.vwbnt.cn/242044.Doc
hwb.vwbnt.cn/808266.Doc
hwb.vwbnt.cn/646424.Doc
hwb.vwbnt.cn/882002.Doc
hwb.vwbnt.cn/651965.Doc
hwb.vwbnt.cn/644604.Doc
hwv.vwbnt.cn/864226.Doc
hwv.vwbnt.cn/333513.Doc
hwv.vwbnt.cn/971997.Doc
hwv.vwbnt.cn/420608.Doc
hwv.vwbnt.cn/602620.Doc
hwv.vwbnt.cn/206844.Doc
hwv.vwbnt.cn/282240.Doc
hwv.vwbnt.cn/660802.Doc
hwv.vwbnt.cn/880282.Doc
hwv.vwbnt.cn/200844.Doc
hwc.vwbnt.cn/824420.Doc
hwc.vwbnt.cn/497182.Doc
hwc.vwbnt.cn/773664.Doc
hwc.vwbnt.cn/973919.Doc
hwc.vwbnt.cn/802727.Doc
hwc.vwbnt.cn/040446.Doc
hwc.vwbnt.cn/828622.Doc
hwc.vwbnt.cn/150185.Doc
hwc.vwbnt.cn/406080.Doc
hwc.vwbnt.cn/202464.Doc
hwx.vwbnt.cn/627087.Doc
hwx.vwbnt.cn/628444.Doc
hwx.vwbnt.cn/202268.Doc
hwx.vwbnt.cn/622224.Doc
hwx.vwbnt.cn/800882.Doc
hwx.vwbnt.cn/242082.Doc
hwx.vwbnt.cn/822640.Doc
hwx.vwbnt.cn/064482.Doc
hwx.vwbnt.cn/238218.Doc
hwx.vwbnt.cn/444268.Doc
hwz.vwbnt.cn/684228.Doc
hwz.vwbnt.cn/262428.Doc
hwz.vwbnt.cn/423548.Doc
hwz.vwbnt.cn/620284.Doc
hwz.vwbnt.cn/002084.Doc
hwz.vwbnt.cn/668802.Doc
hwz.vwbnt.cn/151517.Doc
hwz.vwbnt.cn/500680.Doc
hwz.vwbnt.cn/553713.Doc
hwz.vwbnt.cn/048622.Doc
hwl.vwbnt.cn/488282.Doc
hwl.vwbnt.cn/448420.Doc
hwl.vwbnt.cn/024204.Doc
hwl.vwbnt.cn/064266.Doc
hwl.vwbnt.cn/840264.Doc
hwl.vwbnt.cn/722413.Doc
hwl.vwbnt.cn/246420.Doc
hwl.vwbnt.cn/551357.Doc
hwl.vwbnt.cn/537753.Doc
hwl.vwbnt.cn/004822.Doc
hwk.vwbnt.cn/927554.Doc
hwk.vwbnt.cn/044220.Doc
hwk.vwbnt.cn/482422.Doc
hwk.vwbnt.cn/486024.Doc
hwk.vwbnt.cn/064804.Doc
hwk.vwbnt.cn/802082.Doc
hwk.vwbnt.cn/468280.Doc
hwk.vwbnt.cn/319133.Doc
hwk.vwbnt.cn/820006.Doc
hwk.vwbnt.cn/408282.Doc
hwj.vwbnt.cn/535939.Doc
hwj.vwbnt.cn/880868.Doc
hwj.vwbnt.cn/115931.Doc
hwj.vwbnt.cn/406660.Doc
hwj.vwbnt.cn/660264.Doc
hwj.vwbnt.cn/028868.Doc
hwj.vwbnt.cn/218059.Doc
hwj.vwbnt.cn/200824.Doc
hwj.vwbnt.cn/808840.Doc
hwj.vwbnt.cn/828886.Doc
hwh.vwbnt.cn/064080.Doc
hwh.vwbnt.cn/660402.Doc
hwh.vwbnt.cn/080022.Doc
hwh.vwbnt.cn/422086.Doc
hwh.vwbnt.cn/292222.Doc
hwh.vwbnt.cn/808422.Doc
hwh.vwbnt.cn/286800.Doc
hwh.vwbnt.cn/282066.Doc
hwh.vwbnt.cn/262842.Doc
hwh.vwbnt.cn/680408.Doc
hwg.vwbnt.cn/600686.Doc
hwg.vwbnt.cn/664664.Doc
hwg.vwbnt.cn/066404.Doc
hwg.vwbnt.cn/428000.Doc
hwg.vwbnt.cn/844886.Doc
hwg.vwbnt.cn/717717.Doc
hwg.vwbnt.cn/228828.Doc
hwg.vwbnt.cn/822826.Doc
hwg.vwbnt.cn/066042.Doc
hwg.vwbnt.cn/131331.Doc
hwf.vwbnt.cn/028022.Doc
hwf.vwbnt.cn/426844.Doc
hwf.vwbnt.cn/777116.Doc
hwf.vwbnt.cn/860204.Doc
hwf.vwbnt.cn/041147.Doc
hwf.vwbnt.cn/448266.Doc
hwf.vwbnt.cn/804060.Doc
hwf.vwbnt.cn/442648.Doc
hwf.vwbnt.cn/446666.Doc
hwf.vwbnt.cn/848026.Doc
hwd.vwbnt.cn/113511.Doc
hwd.vwbnt.cn/693803.Doc
hwd.vwbnt.cn/240064.Doc
hwd.vwbnt.cn/002868.Doc
hwd.vwbnt.cn/060006.Doc
hwd.vwbnt.cn/917355.Doc
hwd.vwbnt.cn/866486.Doc
hwd.vwbnt.cn/448686.Doc
hwd.vwbnt.cn/268002.Doc
hwd.vwbnt.cn/864226.Doc
hws.vwbnt.cn/262468.Doc
hws.vwbnt.cn/604202.Doc
hws.vwbnt.cn/004022.Doc
hws.vwbnt.cn/000640.Doc
hws.vwbnt.cn/826846.Doc
hws.vwbnt.cn/882826.Doc
hws.vwbnt.cn/086268.Doc
hws.vwbnt.cn/200642.Doc
hws.vwbnt.cn/226640.Doc
hws.vwbnt.cn/217946.Doc
hwa.vwbnt.cn/446808.Doc
hwa.vwbnt.cn/808088.Doc
hwa.vwbnt.cn/111915.Doc
hwa.vwbnt.cn/426468.Doc
hwa.vwbnt.cn/022882.Doc
hwa.vwbnt.cn/860660.Doc
hwa.vwbnt.cn/648822.Doc
hwa.vwbnt.cn/024428.Doc
hwa.vwbnt.cn/244226.Doc
hwa.vwbnt.cn/468632.Doc
hwp.vwbnt.cn/882680.Doc
hwp.vwbnt.cn/482242.Doc
hwp.vwbnt.cn/286882.Doc
hwp.vwbnt.cn/464262.Doc
hwp.vwbnt.cn/224602.Doc
hwp.vwbnt.cn/185712.Doc
hwp.vwbnt.cn/462040.Doc
hwp.vwbnt.cn/602462.Doc
hwp.vwbnt.cn/286468.Doc
hwp.vwbnt.cn/222446.Doc
hwo.vwbnt.cn/934286.Doc
hwo.vwbnt.cn/608826.Doc
hwo.vwbnt.cn/008668.Doc
hwo.vwbnt.cn/485465.Doc
hwo.vwbnt.cn/486464.Doc
hwo.vwbnt.cn/846824.Doc
hwo.vwbnt.cn/622044.Doc
hwo.vwbnt.cn/862480.Doc
hwo.vwbnt.cn/446006.Doc
hwo.vwbnt.cn/753777.Doc
hwi.vwbnt.cn/846246.Doc
hwi.vwbnt.cn/802208.Doc
hwi.vwbnt.cn/245406.Doc
hwi.vwbnt.cn/648266.Doc
hwi.vwbnt.cn/712670.Doc
hwi.vwbnt.cn/637251.Doc
hwi.vwbnt.cn/024688.Doc
hwi.vwbnt.cn/602822.Doc
hwi.vwbnt.cn/222860.Doc
hwi.vwbnt.cn/602206.Doc
hwu.vwbnt.cn/628381.Doc
hwu.vwbnt.cn/600224.Doc
hwu.vwbnt.cn/627114.Doc
hwu.vwbnt.cn/841056.Doc
hwu.vwbnt.cn/806060.Doc
hwu.vwbnt.cn/513157.Doc
hwu.vwbnt.cn/848888.Doc
hwu.vwbnt.cn/869115.Doc
hwu.vwbnt.cn/680682.Doc
hwu.vwbnt.cn/848860.Doc
hwy.vwbnt.cn/442022.Doc
hwy.vwbnt.cn/197918.Doc
hwy.vwbnt.cn/426208.Doc
hwy.vwbnt.cn/684064.Doc
hwy.vwbnt.cn/068264.Doc
hwy.vwbnt.cn/066228.Doc
hwy.vwbnt.cn/882344.Doc
hwy.vwbnt.cn/831805.Doc
hwy.vwbnt.cn/662442.Doc
hwy.vwbnt.cn/408015.Doc
hwt.vwbnt.cn/002286.Doc
hwt.vwbnt.cn/428388.Doc
hwt.vwbnt.cn/826840.Doc
hwt.vwbnt.cn/068606.Doc
hwt.vwbnt.cn/268662.Doc
hwt.vwbnt.cn/406628.Doc
hwt.vwbnt.cn/512299.Doc
hwt.vwbnt.cn/060462.Doc
hwt.vwbnt.cn/248086.Doc
hwt.vwbnt.cn/642642.Doc
hwr.vwbnt.cn/260068.Doc
hwr.vwbnt.cn/682406.Doc
hwr.vwbnt.cn/828042.Doc
hwr.vwbnt.cn/004608.Doc
hwr.vwbnt.cn/424860.Doc
hwr.vwbnt.cn/884008.Doc
hwr.vwbnt.cn/260044.Doc
hwr.vwbnt.cn/086848.Doc
hwr.vwbnt.cn/042464.Doc
hwr.vwbnt.cn/064488.Doc
hwe.vwbnt.cn/000020.Doc
hwe.vwbnt.cn/888802.Doc
hwe.vwbnt.cn/280204.Doc
hwe.vwbnt.cn/000442.Doc
hwe.vwbnt.cn/413053.Doc
hwe.vwbnt.cn/666822.Doc
hwe.vwbnt.cn/066844.Doc
hwe.vwbnt.cn/660064.Doc
hwe.vwbnt.cn/620062.Doc
hwe.vwbnt.cn/006820.Doc
hww.vwbnt.cn/835164.Doc
hww.vwbnt.cn/026868.Doc
hww.vwbnt.cn/033457.Doc
hww.vwbnt.cn/120311.Doc
hww.vwbnt.cn/044428.Doc
hww.vwbnt.cn/869947.Doc
hww.vwbnt.cn/420268.Doc
hww.vwbnt.cn/797700.Doc
hww.vwbnt.cn/882088.Doc
hww.vwbnt.cn/224862.Doc
hwq.vwbnt.cn/040820.Doc
hwq.vwbnt.cn/860462.Doc
hwq.vwbnt.cn/602684.Doc
hwq.vwbnt.cn/002864.Doc
hwq.vwbnt.cn/939997.Doc
hwq.vwbnt.cn/622224.Doc
hwq.vwbnt.cn/462048.Doc
hwq.vwbnt.cn/608668.Doc
hwq.vwbnt.cn/248444.Doc
hwq.vwbnt.cn/159757.Doc
hqm.vwbnt.cn/224404.Doc
hqm.vwbnt.cn/000666.Doc
hqm.vwbnt.cn/862622.Doc
hqm.vwbnt.cn/444286.Doc
hqm.vwbnt.cn/066288.Doc
hqm.vwbnt.cn/204406.Doc
hqm.vwbnt.cn/026222.Doc
hqm.vwbnt.cn/402068.Doc
hqm.vwbnt.cn/808486.Doc
hqm.vwbnt.cn/864408.Doc
hqn.vwbnt.cn/664482.Doc
hqn.vwbnt.cn/220488.Doc
hqn.vwbnt.cn/844660.Doc
hqn.vwbnt.cn/420022.Doc
hqn.vwbnt.cn/622026.Doc
hqn.vwbnt.cn/000422.Doc
hqn.vwbnt.cn/048004.Doc
hqn.vwbnt.cn/048868.Doc
hqn.vwbnt.cn/395208.Doc
hqn.vwbnt.cn/024462.Doc
hqb.vwbnt.cn/591593.Doc
hqb.vwbnt.cn/044440.Doc
hqb.vwbnt.cn/204246.Doc
hqb.vwbnt.cn/820628.Doc
hqb.vwbnt.cn/006026.Doc
hqb.vwbnt.cn/202442.Doc
hqb.vwbnt.cn/464884.Doc
hqb.vwbnt.cn/060803.Doc
hqb.vwbnt.cn/042206.Doc
hqb.vwbnt.cn/220884.Doc
hqv.vwbnt.cn/913575.Doc
hqv.vwbnt.cn/802686.Doc
hqv.vwbnt.cn/977993.Doc
hqv.vwbnt.cn/204884.Doc
hqv.vwbnt.cn/422800.Doc
hqv.vwbnt.cn/420442.Doc
hqv.vwbnt.cn/060420.Doc
hqv.vwbnt.cn/955441.Doc
hqv.vwbnt.cn/824266.Doc
hqv.vwbnt.cn/137375.Doc
hqc.vwbnt.cn/824662.Doc
hqc.vwbnt.cn/786112.Doc
hqc.vwbnt.cn/620242.Doc
hqc.vwbnt.cn/622606.Doc
hqc.vwbnt.cn/048806.Doc
hqc.vwbnt.cn/246864.Doc
hqc.vwbnt.cn/284274.Doc
hqc.vwbnt.cn/206044.Doc
hqc.vwbnt.cn/278716.Doc
hqc.vwbnt.cn/828244.Doc
hqx.vwbnt.cn/082666.Doc
hqx.vwbnt.cn/715159.Doc
hqx.vwbnt.cn/684004.Doc
hqx.vwbnt.cn/222642.Doc
hqx.vwbnt.cn/046226.Doc
hqx.vwbnt.cn/352621.Doc
hqx.vwbnt.cn/644224.Doc
hqx.vwbnt.cn/086064.Doc
hqx.vwbnt.cn/971553.Doc
hqx.vwbnt.cn/573117.Doc
hqz.vwbnt.cn/660824.Doc
hqz.vwbnt.cn/141298.Doc
hqz.vwbnt.cn/242002.Doc
hqz.vwbnt.cn/244082.Doc
hqz.vwbnt.cn/048088.Doc
hqz.vwbnt.cn/631649.Doc
hqz.vwbnt.cn/202468.Doc
hqz.vwbnt.cn/248628.Doc
hqz.vwbnt.cn/402406.Doc
hqz.vwbnt.cn/208820.Doc
hql.vwbnt.cn/848006.Doc
hql.vwbnt.cn/220240.Doc
hql.vwbnt.cn/844640.Doc
hql.vwbnt.cn/026424.Doc
hql.vwbnt.cn/000488.Doc
hql.vwbnt.cn/844602.Doc
hql.vwbnt.cn/086167.Doc
hql.vwbnt.cn/595517.Doc
hql.vwbnt.cn/848462.Doc
hql.vwbnt.cn/884020.Doc
hqk.vwbnt.cn/137115.Doc
hqk.vwbnt.cn/484264.Doc
hqk.vwbnt.cn/488006.Doc
hqk.vwbnt.cn/648008.Doc
hqk.vwbnt.cn/880626.Doc
hqk.vwbnt.cn/042606.Doc
hqk.vwbnt.cn/733793.Doc
hqk.vwbnt.cn/220064.Doc
hqk.vwbnt.cn/840206.Doc
hqk.vwbnt.cn/604486.Doc
hqj.vwbnt.cn/018168.Doc
hqj.vwbnt.cn/628462.Doc
hqj.vwbnt.cn/359333.Doc
hqj.vwbnt.cn/167405.Doc
hqj.vwbnt.cn/426206.Doc
hqj.vwbnt.cn/228204.Doc
hqj.vwbnt.cn/426679.Doc
hqj.vwbnt.cn/460222.Doc
hqj.vwbnt.cn/882804.Doc
hqj.vwbnt.cn/282646.Doc
hqh.vwbnt.cn/066228.Doc
hqh.vwbnt.cn/864446.Doc
hqh.vwbnt.cn/060048.Doc
hqh.vwbnt.cn/062864.Doc
hqh.vwbnt.cn/478526.Doc
hqh.vwbnt.cn/886484.Doc
hqh.vwbnt.cn/280868.Doc
hqh.vwbnt.cn/880602.Doc
hqh.vwbnt.cn/406040.Doc
hqh.vwbnt.cn/000084.Doc
hqg.vwbnt.cn/422748.Doc
hqg.vwbnt.cn/820602.Doc
hqg.vwbnt.cn/207873.Doc
hqg.vwbnt.cn/860606.Doc
hqg.vwbnt.cn/860868.Doc
hqg.vwbnt.cn/066842.Doc
hqg.vwbnt.cn/006864.Doc
hqg.vwbnt.cn/644444.Doc
hqg.vwbnt.cn/200600.Doc
hqg.vwbnt.cn/644644.Doc
hqf.vwbnt.cn/040480.Doc
hqf.vwbnt.cn/666048.Doc
hqf.vwbnt.cn/402868.Doc
hqf.vwbnt.cn/090466.Doc
hqf.vwbnt.cn/842640.Doc
hqf.vwbnt.cn/204046.Doc
hqf.vwbnt.cn/840266.Doc
hqf.vwbnt.cn/769657.Doc
hqf.vwbnt.cn/751731.Doc
hqf.vwbnt.cn/448400.Doc
hqd.vwbnt.cn/199757.Doc
hqd.vwbnt.cn/888880.Doc
hqd.vwbnt.cn/824826.Doc
hqd.vwbnt.cn/606204.Doc
hqd.vwbnt.cn/357531.Doc
hqd.vwbnt.cn/826606.Doc
hqd.vwbnt.cn/604008.Doc
hqd.vwbnt.cn/804602.Doc
hqd.vwbnt.cn/684138.Doc
hqd.vwbnt.cn/020262.Doc
hqs.vwbnt.cn/668488.Doc
hqs.vwbnt.cn/246044.Doc
hqs.vwbnt.cn/262436.Doc
hqs.vwbnt.cn/260660.Doc
