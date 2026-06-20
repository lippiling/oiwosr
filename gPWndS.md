殖己瞎吭谰


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
靥掠桥恍邪狗嚎俾诙姆芈己秘米涝

yzw.lmlgyi9.cn/113975.htm
yzw.lmlgyi9.cn/375775.htm
yzw.lmlgyi9.cn/115955.htm
yzq.phf3mxb.cn/793375.htm
yzq.phf3mxb.cn/377735.htm
yzq.phf3mxb.cn/599175.htm
yzq.phf3mxb.cn/335115.htm
yzq.phf3mxb.cn/006825.htm
yzq.phf3mxb.cn/133355.htm
yzq.phf3mxb.cn/313395.htm
yzq.phf3mxb.cn/642405.htm
yzq.phf3mxb.cn/937355.htm
yzq.phf3mxb.cn/884805.htm
yltv.phf3mxb.cn/979715.htm
yltv.phf3mxb.cn/828285.htm
yltv.phf3mxb.cn/040425.htm
yltv.phf3mxb.cn/513315.htm
yltv.phf3mxb.cn/866685.htm
yltv.phf3mxb.cn/460625.htm
yltv.phf3mxb.cn/426845.htm
yltv.phf3mxb.cn/551915.htm
yltv.phf3mxb.cn/608225.htm
yltv.phf3mxb.cn/804685.htm
yln.phf3mxb.cn/468645.htm
yln.phf3mxb.cn/395715.htm
yln.phf3mxb.cn/860045.htm
yln.phf3mxb.cn/046065.htm
yln.phf3mxb.cn/355575.htm
yln.phf3mxb.cn/286605.htm
yln.phf3mxb.cn/575115.htm
yln.phf3mxb.cn/993975.htm
yln.phf3mxb.cn/533175.htm
yln.phf3mxb.cn/115375.htm
ylb.phf3mxb.cn/991995.htm
ylb.phf3mxb.cn/000005.htm
ylb.phf3mxb.cn/577735.htm
ylb.phf3mxb.cn/111955.htm
ylb.phf3mxb.cn/020625.htm
ylb.phf3mxb.cn/802685.htm
ylb.phf3mxb.cn/395335.htm
ylb.phf3mxb.cn/464865.htm
ylb.phf3mxb.cn/842885.htm
ylb.phf3mxb.cn/575195.htm
ylv.phf3mxb.cn/319535.htm
ylv.phf3mxb.cn/006685.htm
ylv.phf3mxb.cn/408265.htm
ylv.phf3mxb.cn/228405.htm
ylv.phf3mxb.cn/795395.htm
ylv.phf3mxb.cn/957315.htm
ylv.phf3mxb.cn/519375.htm
ylv.phf3mxb.cn/391335.htm
ylv.phf3mxb.cn/800865.htm
ylv.phf3mxb.cn/824485.htm
ylc.phf3mxb.cn/779395.htm
ylc.phf3mxb.cn/539395.htm
ylc.phf3mxb.cn/282225.htm
ylc.phf3mxb.cn/424025.htm
ylc.phf3mxb.cn/062285.htm
ylc.phf3mxb.cn/608825.htm
ylc.phf3mxb.cn/375975.htm
ylc.phf3mxb.cn/646085.htm
ylc.phf3mxb.cn/757795.htm
ylc.phf3mxb.cn/557155.htm
ylx.phf3mxb.cn/931195.htm
ylx.phf3mxb.cn/600065.htm
ylx.phf3mxb.cn/444265.htm
ylx.phf3mxb.cn/002825.htm
ylx.phf3mxb.cn/957595.htm
ylx.phf3mxb.cn/246485.htm
ylx.phf3mxb.cn/519155.htm
ylx.phf3mxb.cn/006805.htm
ylx.phf3mxb.cn/820625.htm
ylx.phf3mxb.cn/315915.htm
ylz.phf3mxb.cn/046065.htm
ylz.phf3mxb.cn/779155.htm
ylz.phf3mxb.cn/480805.htm
ylz.phf3mxb.cn/533575.htm
ylz.phf3mxb.cn/517155.htm
ylz.phf3mxb.cn/464625.htm
ylz.phf3mxb.cn/626085.htm
ylz.phf3mxb.cn/595335.htm
ylz.phf3mxb.cn/086225.htm
ylz.phf3mxb.cn/515955.htm
yll.phf3mxb.cn/082025.htm
yll.phf3mxb.cn/757515.htm
yll.phf3mxb.cn/599595.htm
yll.phf3mxb.cn/775135.htm
yll.phf3mxb.cn/393515.htm
yll.phf3mxb.cn/608865.htm
yll.phf3mxb.cn/848225.htm
yll.phf3mxb.cn/373395.htm
yll.phf3mxb.cn/846625.htm
yll.phf3mxb.cn/511755.htm
ylk.phf3mxb.cn/624225.htm
ylk.phf3mxb.cn/802845.htm
ylk.phf3mxb.cn/573195.htm
ylk.phf3mxb.cn/044645.htm
ylk.phf3mxb.cn/642845.htm
ylk.phf3mxb.cn/208605.htm
ylk.phf3mxb.cn/484805.htm
ylk.phf3mxb.cn/911135.htm
ylk.phf3mxb.cn/880025.htm
ylk.phf3mxb.cn/026245.htm
ylj.phf3mxb.cn/955715.htm
ylj.phf3mxb.cn/020865.htm
ylj.phf3mxb.cn/317155.htm
ylj.phf3mxb.cn/804445.htm
ylj.phf3mxb.cn/511775.htm
ylj.phf3mxb.cn/133135.htm
ylj.phf3mxb.cn/880485.htm
ylj.phf3mxb.cn/646205.htm
ylj.phf3mxb.cn/117335.htm
ylj.phf3mxb.cn/773335.htm
ylh.phf3mxb.cn/468625.htm
ylh.phf3mxb.cn/999755.htm
ylh.phf3mxb.cn/226605.htm
ylh.phf3mxb.cn/266445.htm
ylh.phf3mxb.cn/513735.htm
ylh.phf3mxb.cn/486225.htm
ylh.phf3mxb.cn/731975.htm
ylh.phf3mxb.cn/408025.htm
ylh.phf3mxb.cn/791795.htm
ylh.phf3mxb.cn/824665.htm
ylg.phf3mxb.cn/399395.htm
ylg.phf3mxb.cn/791795.htm
ylg.phf3mxb.cn/008405.htm
ylg.phf3mxb.cn/935375.htm
ylg.phf3mxb.cn/866825.htm
ylg.phf3mxb.cn/248465.htm
ylg.phf3mxb.cn/513935.htm
ylg.phf3mxb.cn/777995.htm
ylg.phf3mxb.cn/951375.htm
ylg.phf3mxb.cn/824285.htm
ylf.phf3mxb.cn/248625.htm
ylf.phf3mxb.cn/179735.htm
ylf.phf3mxb.cn/408425.htm
ylf.phf3mxb.cn/066085.htm
ylf.phf3mxb.cn/757955.htm
ylf.phf3mxb.cn/533935.htm
ylf.phf3mxb.cn/482025.htm
ylf.phf3mxb.cn/240205.htm
ylf.phf3mxb.cn/759915.htm
ylf.phf3mxb.cn/719775.htm
yld.phf3mxb.cn/228485.htm
yld.phf3mxb.cn/973555.htm
yld.phf3mxb.cn/864065.htm
yld.phf3mxb.cn/282085.htm
yld.phf3mxb.cn/648445.htm
yld.phf3mxb.cn/448605.htm
yld.phf3mxb.cn/886005.htm
yld.phf3mxb.cn/046225.htm
yld.phf3mxb.cn/539795.htm
yld.phf3mxb.cn/995575.htm
yls.phf3mxb.cn/680465.htm
yls.phf3mxb.cn/648805.htm
yls.phf3mxb.cn/626445.htm
yls.phf3mxb.cn/111775.htm
yls.phf3mxb.cn/579315.htm
yls.phf3mxb.cn/626205.htm
yls.phf3mxb.cn/888265.htm
yls.phf3mxb.cn/917335.htm
yls.phf3mxb.cn/991155.htm
yls.phf3mxb.cn/159315.htm
yla.phf3mxb.cn/600045.htm
yla.phf3mxb.cn/733375.htm
yla.phf3mxb.cn/799995.htm
yla.phf3mxb.cn/197395.htm
yla.phf3mxb.cn/571515.htm
yla.phf3mxb.cn/995955.htm
yla.phf3mxb.cn/993395.htm
yla.phf3mxb.cn/200405.htm
yla.phf3mxb.cn/008485.htm
yla.phf3mxb.cn/042065.htm
ylp.phf3mxb.cn/953995.htm
ylp.phf3mxb.cn/593715.htm
ylp.phf3mxb.cn/488865.htm
ylp.phf3mxb.cn/595195.htm
ylp.phf3mxb.cn/202805.htm
ylp.phf3mxb.cn/751315.htm
ylp.phf3mxb.cn/115755.htm
ylp.phf3mxb.cn/026445.htm
ylp.phf3mxb.cn/779535.htm
ylp.phf3mxb.cn/468005.htm
ylo.phf3mxb.cn/315735.htm
ylo.phf3mxb.cn/606085.htm
ylo.phf3mxb.cn/466205.htm
ylo.phf3mxb.cn/953915.htm
ylo.phf3mxb.cn/153775.htm
ylo.phf3mxb.cn/751155.htm
ylo.phf3mxb.cn/533535.htm
ylo.phf3mxb.cn/606245.htm
ylo.phf3mxb.cn/957555.htm
ylo.phf3mxb.cn/173975.htm
yli.phf3mxb.cn/795735.htm
yli.phf3mxb.cn/646885.htm
yli.phf3mxb.cn/397335.htm
yli.phf3mxb.cn/682685.htm
yli.phf3mxb.cn/020605.htm
yli.phf3mxb.cn/953555.htm
yli.phf3mxb.cn/975795.htm
yli.phf3mxb.cn/068805.htm
yli.phf3mxb.cn/004465.htm
yli.phf3mxb.cn/755195.htm
ylu.phf3mxb.cn/648005.htm
ylu.phf3mxb.cn/422205.htm
ylu.phf3mxb.cn/571135.htm
ylu.phf3mxb.cn/826245.htm
ylu.phf3mxb.cn/628445.htm
ylu.phf3mxb.cn/244245.htm
ylu.phf3mxb.cn/266005.htm
ylu.phf3mxb.cn/175515.htm
ylu.phf3mxb.cn/866065.htm
ylu.phf3mxb.cn/662425.htm
yly.phf3mxb.cn/620205.htm
yly.phf3mxb.cn/957735.htm
yly.phf3mxb.cn/408445.htm
yly.phf3mxb.cn/440665.htm
yly.phf3mxb.cn/248265.htm
yly.phf3mxb.cn/022465.htm
yly.phf3mxb.cn/759115.htm
yly.phf3mxb.cn/800405.htm
yly.phf3mxb.cn/717755.htm
yly.phf3mxb.cn/088465.htm
ylt.phf3mxb.cn/151755.htm
ylt.phf3mxb.cn/791135.htm
ylt.phf3mxb.cn/753995.htm
ylt.phf3mxb.cn/640025.htm
ylt.phf3mxb.cn/931375.htm
ylt.phf3mxb.cn/406405.htm
ylt.phf3mxb.cn/448825.htm
ylt.phf3mxb.cn/159975.htm
ylt.phf3mxb.cn/715195.htm
ylt.phf3mxb.cn/319915.htm
ylr.phf3mxb.cn/119775.htm
ylr.phf3mxb.cn/737515.htm
ylr.phf3mxb.cn/551735.htm
ylr.phf3mxb.cn/311735.htm
ylr.phf3mxb.cn/082005.htm
ylr.phf3mxb.cn/080225.htm
ylr.phf3mxb.cn/406205.htm
ylr.phf3mxb.cn/482205.htm
ylr.phf3mxb.cn/397755.htm
ylr.phf3mxb.cn/399195.htm
yle.phf3mxb.cn/353935.htm
yle.phf3mxb.cn/373775.htm
yle.phf3mxb.cn/228685.htm
yle.phf3mxb.cn/717535.htm
yle.phf3mxb.cn/620405.htm
yle.phf3mxb.cn/268465.htm
yle.phf3mxb.cn/002265.htm
yle.phf3mxb.cn/517155.htm
yle.phf3mxb.cn/939775.htm
yle.phf3mxb.cn/733955.htm
ylw.phf3mxb.cn/991535.htm
ylw.phf3mxb.cn/624645.htm
ylw.phf3mxb.cn/860485.htm
ylw.phf3mxb.cn/848285.htm
ylw.phf3mxb.cn/773995.htm
ylw.phf3mxb.cn/313775.htm
ylw.phf3mxb.cn/731395.htm
ylw.phf3mxb.cn/600825.htm
ylw.phf3mxb.cn/999355.htm
ylw.phf3mxb.cn/008665.htm
ylq.phf3mxb.cn/068825.htm
ylq.phf3mxb.cn/797575.htm
ylq.phf3mxb.cn/066225.htm
ylq.phf3mxb.cn/448205.htm
ylq.phf3mxb.cn/800085.htm
ylq.phf3mxb.cn/204605.htm
ylq.phf3mxb.cn/444265.htm
ylq.phf3mxb.cn/242225.htm
ylq.phf3mxb.cn/311315.htm
ylq.phf3mxb.cn/028085.htm
yktv.phf3mxb.cn/757155.htm
yktv.phf3mxb.cn/553315.htm
yktv.phf3mxb.cn/004605.htm
yktv.phf3mxb.cn/391555.htm
yktv.phf3mxb.cn/597115.htm
yktv.phf3mxb.cn/284425.htm
yktv.phf3mxb.cn/484205.htm
yktv.phf3mxb.cn/757755.htm
yktv.phf3mxb.cn/919355.htm
yktv.phf3mxb.cn/804225.htm
ykn.phf3mxb.cn/426205.htm
ykn.phf3mxb.cn/862045.htm
ykn.phf3mxb.cn/775555.htm
ykn.phf3mxb.cn/119595.htm
ykn.phf3mxb.cn/844225.htm
ykn.phf3mxb.cn/884285.htm
ykn.phf3mxb.cn/993775.htm
ykn.phf3mxb.cn/171315.htm
ykn.phf3mxb.cn/202825.htm
ykn.phf3mxb.cn/468885.htm
ykb.phf3mxb.cn/739315.htm
ykb.phf3mxb.cn/171515.htm
ykb.phf3mxb.cn/486045.htm
ykb.phf3mxb.cn/513555.htm
ykb.phf3mxb.cn/486625.htm
ykb.phf3mxb.cn/573975.htm
ykb.phf3mxb.cn/513595.htm
ykb.phf3mxb.cn/131995.htm
ykb.phf3mxb.cn/117715.htm
ykb.phf3mxb.cn/468625.htm
ykv.phf3mxb.cn/131355.htm
ykv.phf3mxb.cn/191975.htm
ykv.phf3mxb.cn/668245.htm
ykv.phf3mxb.cn/022465.htm
ykv.phf3mxb.cn/244665.htm
ykv.phf3mxb.cn/824025.htm
ykv.phf3mxb.cn/482205.htm
ykv.phf3mxb.cn/288885.htm
ykv.phf3mxb.cn/846025.htm
ykv.phf3mxb.cn/795535.htm
ykc.phf3mxb.cn/408005.htm
ykc.phf3mxb.cn/284845.htm
ykc.phf3mxb.cn/539735.htm
ykc.phf3mxb.cn/715135.htm
ykc.phf3mxb.cn/420085.htm
ykc.phf3mxb.cn/953315.htm
ykc.phf3mxb.cn/195915.htm
ykc.phf3mxb.cn/333735.htm
ykc.phf3mxb.cn/282245.htm
ykc.phf3mxb.cn/513175.htm
ykx.phf3mxb.cn/084605.htm
ykx.phf3mxb.cn/820805.htm
ykx.phf3mxb.cn/202465.htm
ykx.phf3mxb.cn/111335.htm
ykx.phf3mxb.cn/593195.htm
ykx.phf3mxb.cn/082885.htm
ykx.phf3mxb.cn/600225.htm
ykx.phf3mxb.cn/115355.htm
ykx.phf3mxb.cn/133955.htm
ykx.phf3mxb.cn/442085.htm
ykz.phf3mxb.cn/735575.htm
ykz.phf3mxb.cn/955915.htm
ykz.phf3mxb.cn/620625.htm
ykz.phf3mxb.cn/642885.htm
ykz.phf3mxb.cn/888265.htm
ykz.phf3mxb.cn/195595.htm
ykz.phf3mxb.cn/133715.htm
ykz.phf3mxb.cn/579575.htm
ykz.phf3mxb.cn/068465.htm
ykz.phf3mxb.cn/602265.htm
ykl.phf3mxb.cn/739575.htm
ykl.phf3mxb.cn/539735.htm
ykl.phf3mxb.cn/939795.htm
ykl.phf3mxb.cn/131795.htm
ykl.phf3mxb.cn/351795.htm
ykl.phf3mxb.cn/131935.htm
ykl.phf3mxb.cn/599575.htm
ykl.phf3mxb.cn/559795.htm
ykl.phf3mxb.cn/466065.htm
ykl.phf3mxb.cn/460445.htm
ykk.phf3mxb.cn/711955.htm
ykk.phf3mxb.cn/973355.htm
ykk.phf3mxb.cn/284285.htm
ykk.phf3mxb.cn/626845.htm
ykk.phf3mxb.cn/515335.htm
ykk.phf3mxb.cn/868225.htm
ykk.phf3mxb.cn/080885.htm
ykk.phf3mxb.cn/351155.htm
ykk.phf3mxb.cn/319315.htm
ykk.phf3mxb.cn/999315.htm
ykj.phf3mxb.cn/844025.htm
ykj.phf3mxb.cn/884405.htm
ykj.phf3mxb.cn/135915.htm
ykj.phf3mxb.cn/006865.htm
ykj.phf3mxb.cn/335795.htm
ykj.phf3mxb.cn/775315.htm
ykj.phf3mxb.cn/840825.htm
ykj.phf3mxb.cn/177795.htm
ykj.phf3mxb.cn/840425.htm
ykj.phf3mxb.cn/882605.htm
ykh.phf3mxb.cn/173315.htm
ykh.phf3mxb.cn/422045.htm
ykh.phf3mxb.cn/135335.htm
ykh.phf3mxb.cn/086625.htm
ykh.phf3mxb.cn/333195.htm
ykh.phf3mxb.cn/737135.htm
ykh.phf3mxb.cn/313395.htm
ykh.phf3mxb.cn/486405.htm
ykh.phf3mxb.cn/571915.htm
ykh.phf3mxb.cn/533375.htm
ykg.phf3mxb.cn/444665.htm
ykg.phf3mxb.cn/004605.htm
ykg.phf3mxb.cn/553555.htm
ykg.phf3mxb.cn/608665.htm
ykg.phf3mxb.cn/068825.htm
ykg.phf3mxb.cn/933575.htm
ykg.phf3mxb.cn/971135.htm
ykg.phf3mxb.cn/042005.htm
ykg.phf3mxb.cn/448605.htm
ykg.phf3mxb.cn/006845.htm
ykf.phf3mxb.cn/822045.htm
ykf.phf3mxb.cn/771795.htm
