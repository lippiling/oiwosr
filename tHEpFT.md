迷虏浊倚脊


Linux dma_alloc_coherent一致性DMA缓冲对齐要求

1. 一致性DMA的Cache-Coherent语义

dma_alloc_coherent分配的内存区域保证CPU和DMA master可在无任何同步操作下安全访问同一片物理内存。其根本实现机制是分配非cacheable（Uncacheable / Write-Combine）的物理页或使用IOMMU建立一致性映射，从而杜绝cache一致性问题。这意味着驱动程序不须调用dma_sync_single_for_cpu/device，显著简化了控制路径代码。

void *dma_alloc_coherent(struct device *dev, size_t size,
                          dma_addr_t *dma_handle, gfp_t flag);

返回两个值：CPU可访问的虚拟地址（void *）和DMA master可访问的总线地址（通过dma_handle返回）。返回的虚拟地址指向非缓存（或Write-Combine）的页，设备可通过dma_handle直接发起DMA操作。

2. 对齐约束

dma_alloc_coherent的对齐规则总结如下：
- 返回的DMA地址（dma_handle）至少以size对齐，且不超过PAGE_SIZE对齐。即对于小于一个page的分配，DMA地址至少按size向上取整到2的幂对齐；对于大于一个page的分配，DMA地址按PAGE_SIZE对齐。
- 如果size在[1, PAGE_SIZE]范围内，dma_handle必然对齐到大于等于size的最小2的幂，但至少是ARCH_DMA_MINALIGN（通常为64或128字节）。
- 对于需要更严格对齐的场景（如DMA引擎要求缓冲起始于4KB边界），驱动必须自行处理，dma_alloc_coherent不保证超过PAGE_SIZE的对齐。

实际分配由底层实现决定。以ARM64为例，dma_alloc_coherent -> __dma_alloc -> dma_alloc_from_dev_coherent -> __alloc_from_pool -> gen_pool_alloc。gen_pool是通用地址分配器，其对齐行为由pool的配置决定。当使用dma_alloc_coherent的原子模式（GFP_ATOMIC），分配从coherent_pool中进行，对齐取决于pool的分配粒度。

3. 大小参数要求

size参数必须是2的幂以及size > 0，尽管内核内部通过__get_free_pages或dma_alloc_contiguous处理，实际分配会向上取整到页面大小倍数。驱动传递非2的幂大小时，dma_alloc_coherent可能使用超过请求的内存页面。这个padding区域对驱动是可见的（因为返回了全页），但驱动不应当访问其范围之外的字节。

4. 内存区域类型

dma_alloc_coherent的flag参数控制底层页面分配行为：
- GFP_KERNEL：默认类型，可能休眠，适合probe路径
- GFP_ATOMIC：原子上下文，从预留的coherent_pool分配（大小通常为256KB）
- GFP_DMA：强制从ZONE_DMA（低16MB或ZONE_DMA32）分配，用于仅支持32-bit DMA地址的老旧设备
- GFP_DMA32：从ZONE_DMA32（前4GB）分配，用于设备地址线不超过36位的情况

若设备有DMA地址限制，应使用dma_set_mask_and_coherent(dev, DMA_BIT_MASK(64))提前设置。dma_alloc_coherent内部会参考dev->dma_mask选择合适的分配区域。

5. dma_alloc_coherent典型模式

dma_addr_t dma_handle;
void *cpu_addr;
struct device *dev = &pdev->dev;

/* 分配描述符环形缓冲区 - 要求256字节对齐 */
#define RING_SIZE  256

/* dma_alloc_coherent保证≥256字节对齐，因为size=256 */
cpu_addr = dma_alloc_coherent(dev, RING_SIZE, &dma_handle, GFP_KERNEL);
if (!cpu_addr) {
    dev_err(dev, "coherent alloc failed\n");
    return -ENOMEM;
}

/* 初始化DMA描述符环 */
memset(cpu_addr, 0, RING_SIZE);
/* dma_handle可直接写入设备寄存器 */
writel(lower_32_bits(dma_handle), priv->ioaddr + DMA_RING_LO);
writel(upper_32_bits(dma_handle), priv->ioaddr + DMA_RING_HI);

6. dma_free_coherent

dma_free_coherent(dev, size, cpu_addr, dma_handle);

释放必须使用相同的dev、size、cpu_addr和dma_handle参数。驱动必须保证在释放时没有正在进行的DMA操作，否则结果未定义。size必须是allocate时传入的相同值。

对于atomic分配的缓冲区，dma_free_coherent将缓冲区回收到coherent_pool中；对于GFP_KERNEL分配的，则通过free_pages释放回buddy allocator。

7. 非coherent平台的降级

在MIPS、某些ARM变体等不支持硬件cache coherency的平台上，dma_alloc_coherent实际返回的是通过ioremap_noncache映射的地址。这意味着：
- 虚拟地址的访问绕过CPU cache，每次读写直接访问内存
- DMA地址直接使用物理地址（无IOMMU时）
- cache-coherency由软件维护（dma_cache_sync），但dma_alloc_coherent隐藏了这些细节

这些平台上dma_alloc_coherent的分配对齐要求不变，但driver应特别注意dma_handle和cpu_addr之间的offset关系可能为0（因为直接映射）。

8. 对齐不足时的替代方案

若硬件要求对齐超过PAGE_SIZE（如4KB对齐但size=64），驱动需要使用dma_alloc_coherent分配更大的内存并手动调整：

#define HW_ALIGN 4096
#define BUF_SIZE 64

cpu_addr = dma_alloc_coherent(dev, BUF_SIZE + HW_ALIGN, &dma_handle, GFP_KERNEL);
if (!cpu_addr) return -ENOMEM;
aligned_phys = ALIGN(dma_handle, HW_ALIGN);
aligned_virt = cpu_addr + (aligned_phys - dma_handle);
/* 使用aligned_phys作为设备DMA地址，aligned_virt作为CPU访问地址 */

核诶腋拔探油脱匝毫池型褪轿仆怕

m.wjr.msfsx.cn/77195.Doc
m.wjr.msfsx.cn/26806.Doc
m.wjr.msfsx.cn/93515.Doc
m.wjr.msfsx.cn/77199.Doc
m.wjr.msfsx.cn/66068.Doc
m.wjr.msfsx.cn/19357.Doc
m.wjr.msfsx.cn/04628.Doc
m.wjr.msfsx.cn/33519.Doc
m.wjr.msfsx.cn/84642.Doc
m.wjr.msfsx.cn/08620.Doc
m.wjr.msfsx.cn/06266.Doc
m.wjr.msfsx.cn/39777.Doc
m.wjr.msfsx.cn/53939.Doc
m.wjr.msfsx.cn/95759.Doc
m.wjr.msfsx.cn/73911.Doc
m.wje.msfsx.cn/84064.Doc
m.wje.msfsx.cn/64262.Doc
m.wje.msfsx.cn/97799.Doc
m.wje.msfsx.cn/19737.Doc
m.wje.msfsx.cn/95737.Doc
m.wje.msfsx.cn/22848.Doc
m.wje.msfsx.cn/97911.Doc
m.wje.msfsx.cn/99795.Doc
m.wje.msfsx.cn/20286.Doc
m.wje.msfsx.cn/17357.Doc
m.wje.msfsx.cn/75535.Doc
m.wje.msfsx.cn/28080.Doc
m.wje.msfsx.cn/84800.Doc
m.wje.msfsx.cn/82884.Doc
m.wje.msfsx.cn/84668.Doc
m.wje.msfsx.cn/48280.Doc
m.wje.msfsx.cn/11771.Doc
m.wje.msfsx.cn/60042.Doc
m.wje.msfsx.cn/64268.Doc
m.wje.msfsx.cn/40048.Doc
m.wjw.msfsx.cn/13997.Doc
m.wjw.msfsx.cn/66284.Doc
m.wjw.msfsx.cn/06442.Doc
m.wjw.msfsx.cn/48866.Doc
m.wjw.msfsx.cn/57591.Doc
m.wjw.msfsx.cn/84820.Doc
m.wjw.msfsx.cn/79191.Doc
m.wjw.msfsx.cn/99933.Doc
m.wjw.msfsx.cn/53977.Doc
m.wjw.msfsx.cn/17135.Doc
m.wjw.msfsx.cn/33113.Doc
m.wjw.msfsx.cn/64446.Doc
m.wjw.msfsx.cn/73739.Doc
m.wjw.msfsx.cn/44048.Doc
m.wjw.msfsx.cn/71197.Doc
m.wjw.msfsx.cn/99535.Doc
m.wjw.msfsx.cn/91771.Doc
m.wjw.msfsx.cn/84884.Doc
m.wjw.msfsx.cn/06444.Doc
m.wjw.msfsx.cn/53739.Doc
m.wjq.msfsx.cn/57719.Doc
m.wjq.msfsx.cn/35957.Doc
m.wjq.msfsx.cn/91919.Doc
m.wjq.msfsx.cn/31131.Doc
m.wjq.msfsx.cn/26240.Doc
m.wjq.msfsx.cn/73151.Doc
m.wjq.msfsx.cn/04646.Doc
m.wjq.msfsx.cn/00048.Doc
m.wjq.msfsx.cn/59131.Doc
m.wjq.msfsx.cn/04826.Doc
m.wjq.msfsx.cn/75113.Doc
m.wjq.msfsx.cn/48828.Doc
m.wjq.msfsx.cn/57137.Doc
m.wjq.msfsx.cn/59911.Doc
m.wjq.msfsx.cn/17317.Doc
m.wjq.msfsx.cn/99117.Doc
m.wjq.msfsx.cn/84668.Doc
m.wjq.msfsx.cn/04044.Doc
m.wjq.msfsx.cn/13931.Doc
m.wjq.msfsx.cn/99359.Doc
m.whm.msfsx.cn/04682.Doc
m.whm.msfsx.cn/77391.Doc
m.whm.msfsx.cn/37157.Doc
m.whm.msfsx.cn/88008.Doc
m.whm.msfsx.cn/53153.Doc
m.whm.msfsx.cn/79159.Doc
m.whm.msfsx.cn/13933.Doc
m.whm.msfsx.cn/68266.Doc
m.whm.msfsx.cn/75351.Doc
m.whm.msfsx.cn/46488.Doc
m.whm.msfsx.cn/77933.Doc
m.whm.msfsx.cn/68240.Doc
m.whm.msfsx.cn/04642.Doc
m.whm.msfsx.cn/11773.Doc
m.whm.msfsx.cn/55755.Doc
m.whm.msfsx.cn/73771.Doc
m.whm.msfsx.cn/91551.Doc
m.whm.msfsx.cn/53535.Doc
m.whm.msfsx.cn/04684.Doc
m.whm.msfsx.cn/46886.Doc
m.whn.msfsx.cn/37553.Doc
m.whn.msfsx.cn/24082.Doc
m.whn.msfsx.cn/91517.Doc
m.whn.msfsx.cn/13331.Doc
m.whn.msfsx.cn/51159.Doc
m.whn.msfsx.cn/86440.Doc
m.whn.msfsx.cn/84048.Doc
m.whn.msfsx.cn/04040.Doc
m.whn.msfsx.cn/15195.Doc
m.whn.msfsx.cn/53777.Doc
m.whn.msfsx.cn/51573.Doc
m.whn.msfsx.cn/93171.Doc
m.whn.msfsx.cn/93953.Doc
m.whn.msfsx.cn/91359.Doc
m.whn.msfsx.cn/93935.Doc
m.whn.msfsx.cn/53977.Doc
m.whn.msfsx.cn/75713.Doc
m.whn.msfsx.cn/04426.Doc
m.whn.msfsx.cn/64800.Doc
m.whn.msfsx.cn/39113.Doc
m.whb.msfsx.cn/17131.Doc
m.whb.msfsx.cn/02662.Doc
m.whb.msfsx.cn/93113.Doc
m.whb.msfsx.cn/3.Doc
m.whb.msfsx.cn/99771.Doc
m.whb.msfsx.cn/37137.Doc
m.whb.msfsx.cn/11997.Doc
m.whb.msfsx.cn/93713.Doc
m.whb.msfsx.cn/15559.Doc
m.whb.msfsx.cn/66444.Doc
m.whb.msfsx.cn/42002.Doc
m.whb.msfsx.cn/91137.Doc
m.whb.msfsx.cn/15319.Doc
m.whb.msfsx.cn/13759.Doc
m.whb.msfsx.cn/06602.Doc
m.whb.msfsx.cn/37755.Doc
m.whb.msfsx.cn/53539.Doc
m.whb.msfsx.cn/88462.Doc
m.whb.msfsx.cn/35777.Doc
m.whb.msfsx.cn/13593.Doc
m.whv.msfsx.cn/00284.Doc
m.whv.msfsx.cn/64246.Doc
m.whv.msfsx.cn/60660.Doc
m.whv.msfsx.cn/53913.Doc
m.whv.msfsx.cn/55735.Doc
m.whv.msfsx.cn/57397.Doc
m.whv.msfsx.cn/82262.Doc
m.whv.msfsx.cn/73191.Doc
m.whv.msfsx.cn/79137.Doc
m.whv.msfsx.cn/11331.Doc
m.whv.msfsx.cn/00484.Doc
m.whv.msfsx.cn/20208.Doc
m.whv.msfsx.cn/64844.Doc
m.whv.msfsx.cn/31393.Doc
m.whv.msfsx.cn/84200.Doc
m.whv.msfsx.cn/39759.Doc
m.whv.msfsx.cn/42884.Doc
m.whv.msfsx.cn/57711.Doc
m.whv.msfsx.cn/66660.Doc
m.whv.msfsx.cn/64868.Doc
m.whc.msfsx.cn/53751.Doc
m.whc.msfsx.cn/13517.Doc
m.whc.msfsx.cn/48062.Doc
m.whc.msfsx.cn/93975.Doc
m.whc.msfsx.cn/79171.Doc
m.whc.msfsx.cn/44440.Doc
m.whc.msfsx.cn/64624.Doc
m.whc.msfsx.cn/93933.Doc
m.whc.msfsx.cn/06244.Doc
m.whc.msfsx.cn/77931.Doc
m.whc.msfsx.cn/60420.Doc
m.whc.msfsx.cn/39977.Doc
m.whc.msfsx.cn/62824.Doc
m.whc.msfsx.cn/66282.Doc
m.whc.msfsx.cn/55315.Doc
m.whc.msfsx.cn/17353.Doc
m.whc.msfsx.cn/62048.Doc
m.whc.msfsx.cn/75953.Doc
m.whc.msfsx.cn/75137.Doc
m.whc.msfsx.cn/51333.Doc
m.whx.msfsx.cn/79111.Doc
m.whx.msfsx.cn/55371.Doc
m.whx.msfsx.cn/71137.Doc
m.whx.msfsx.cn/80020.Doc
m.whx.msfsx.cn/04226.Doc
m.whx.msfsx.cn/13397.Doc
m.whx.msfsx.cn/73173.Doc
m.whx.msfsx.cn/04880.Doc
m.whx.msfsx.cn/42802.Doc
m.whx.msfsx.cn/35931.Doc
m.whx.msfsx.cn/88488.Doc
m.whx.msfsx.cn/22864.Doc
m.whx.msfsx.cn/97119.Doc
m.whx.msfsx.cn/42448.Doc
m.whx.msfsx.cn/31519.Doc
m.whx.msfsx.cn/79137.Doc
m.whx.msfsx.cn/02200.Doc
m.whx.msfsx.cn/42804.Doc
m.whx.msfsx.cn/51151.Doc
m.whx.msfsx.cn/00400.Doc
m.whz.msfsx.cn/24606.Doc
m.whz.msfsx.cn/99979.Doc
m.whz.msfsx.cn/13315.Doc
m.whz.msfsx.cn/82660.Doc
m.whz.msfsx.cn/68824.Doc
m.whz.msfsx.cn/33931.Doc
m.whz.msfsx.cn/77375.Doc
m.whz.msfsx.cn/08486.Doc
m.whz.msfsx.cn/64262.Doc
m.whz.msfsx.cn/00804.Doc
m.whz.msfsx.cn/80220.Doc
m.whz.msfsx.cn/40486.Doc
m.whz.msfsx.cn/53395.Doc
m.whz.msfsx.cn/00268.Doc
m.whz.msfsx.cn/80422.Doc
m.whz.msfsx.cn/62006.Doc
m.whz.msfsx.cn/86842.Doc
m.whz.msfsx.cn/31711.Doc
m.whz.msfsx.cn/15357.Doc
m.whz.msfsx.cn/88288.Doc
m.whl.msfsx.cn/33999.Doc
m.whl.msfsx.cn/59915.Doc
m.whl.msfsx.cn/71335.Doc
m.whl.msfsx.cn/00042.Doc
m.whl.msfsx.cn/40048.Doc
m.whl.msfsx.cn/57595.Doc
m.whl.msfsx.cn/46804.Doc
m.whl.msfsx.cn/44208.Doc
m.whl.msfsx.cn/04662.Doc
m.whl.msfsx.cn/93119.Doc
m.whl.msfsx.cn/08660.Doc
m.whl.msfsx.cn/39735.Doc
m.whl.msfsx.cn/06862.Doc
m.whl.msfsx.cn/86244.Doc
m.whl.msfsx.cn/62084.Doc
m.whl.msfsx.cn/62042.Doc
m.whl.msfsx.cn/53177.Doc
m.whl.msfsx.cn/02828.Doc
m.whl.msfsx.cn/55975.Doc
m.whl.msfsx.cn/44666.Doc
m.whk.msfsx.cn/99197.Doc
m.whk.msfsx.cn/44464.Doc
m.whk.msfsx.cn/20428.Doc
m.whk.msfsx.cn/66244.Doc
m.whk.msfsx.cn/62624.Doc
m.whk.msfsx.cn/66622.Doc
m.whk.msfsx.cn/59951.Doc
m.whk.msfsx.cn/59933.Doc
m.whk.msfsx.cn/22046.Doc
m.whk.msfsx.cn/22424.Doc
m.whk.msfsx.cn/31113.Doc
m.whk.msfsx.cn/51793.Doc
m.whk.msfsx.cn/13919.Doc
m.whk.msfsx.cn/31919.Doc
m.whk.msfsx.cn/46664.Doc
m.whk.msfsx.cn/35915.Doc
m.whk.msfsx.cn/59915.Doc
m.whk.msfsx.cn/26468.Doc
m.whk.msfsx.cn/22682.Doc
m.whk.msfsx.cn/55919.Doc
m.whj.msfsx.cn/26262.Doc
m.whj.msfsx.cn/77939.Doc
m.whj.msfsx.cn/57799.Doc
m.whj.msfsx.cn/91751.Doc
m.whj.msfsx.cn/44006.Doc
m.whj.msfsx.cn/22260.Doc
m.whj.msfsx.cn/22420.Doc
m.whj.msfsx.cn/08824.Doc
m.whj.msfsx.cn/68824.Doc
m.whj.msfsx.cn/04282.Doc
m.whj.msfsx.cn/53571.Doc
m.whj.msfsx.cn/08408.Doc
m.whj.msfsx.cn/62440.Doc
m.whj.msfsx.cn/00826.Doc
m.whj.msfsx.cn/39119.Doc
m.whj.msfsx.cn/24060.Doc
m.whj.msfsx.cn/37373.Doc
m.whj.msfsx.cn/00688.Doc
m.whj.msfsx.cn/00648.Doc
m.whj.msfsx.cn/84482.Doc
m.whh.msfsx.cn/00640.Doc
m.whh.msfsx.cn/91795.Doc
m.whh.msfsx.cn/00204.Doc
m.whh.msfsx.cn/59971.Doc
m.whh.msfsx.cn/75539.Doc
m.whh.msfsx.cn/37379.Doc
m.whh.msfsx.cn/02824.Doc
m.whh.msfsx.cn/39371.Doc
m.whh.msfsx.cn/71751.Doc
m.whh.msfsx.cn/68240.Doc
m.whh.msfsx.cn/60022.Doc
m.whh.msfsx.cn/57511.Doc
m.whh.msfsx.cn/77131.Doc
m.whh.msfsx.cn/75311.Doc
m.whh.msfsx.cn/71335.Doc
m.whh.msfsx.cn/24824.Doc
m.whh.msfsx.cn/02226.Doc
m.whh.msfsx.cn/68860.Doc
m.whh.msfsx.cn/17991.Doc
m.whh.msfsx.cn/39133.Doc
m.whg.msfsx.cn/77993.Doc
m.whg.msfsx.cn/75935.Doc
m.whg.msfsx.cn/42468.Doc
m.whg.msfsx.cn/57557.Doc
m.whg.msfsx.cn/60244.Doc
m.whg.msfsx.cn/91515.Doc
m.whg.msfsx.cn/46262.Doc
m.whg.msfsx.cn/64824.Doc
m.whg.msfsx.cn/28008.Doc
m.whg.msfsx.cn/22400.Doc
m.whg.msfsx.cn/31577.Doc
m.whg.msfsx.cn/26226.Doc
m.whg.msfsx.cn/48040.Doc
m.whg.msfsx.cn/02684.Doc
m.whg.msfsx.cn/44686.Doc
m.whg.msfsx.cn/46068.Doc
m.whg.msfsx.cn/22000.Doc
m.whg.msfsx.cn/33379.Doc
m.whg.msfsx.cn/46466.Doc
m.whg.msfsx.cn/71371.Doc
m.whf.msfsx.cn/77513.Doc
m.whf.msfsx.cn/00806.Doc
m.whf.msfsx.cn/17391.Doc
m.whf.msfsx.cn/80608.Doc
m.whf.msfsx.cn/13339.Doc
m.whf.msfsx.cn/95731.Doc
m.whf.msfsx.cn/42608.Doc
m.whf.msfsx.cn/91951.Doc
m.whf.msfsx.cn/02040.Doc
m.whf.msfsx.cn/08802.Doc
m.whf.msfsx.cn/04266.Doc
m.whf.msfsx.cn/82800.Doc
m.whf.msfsx.cn/97391.Doc
m.whf.msfsx.cn/37577.Doc
m.whf.msfsx.cn/59515.Doc
m.whf.msfsx.cn/59931.Doc
m.whf.msfsx.cn/75717.Doc
m.whf.msfsx.cn/37317.Doc
m.whf.msfsx.cn/15535.Doc
m.whf.msfsx.cn/22866.Doc
m.whd.msfsx.cn/44484.Doc
m.whd.msfsx.cn/17171.Doc
m.whd.msfsx.cn/68886.Doc
m.whd.msfsx.cn/66084.Doc
m.whd.msfsx.cn/35119.Doc
m.whd.msfsx.cn/93395.Doc
m.whd.msfsx.cn/71573.Doc
m.whd.msfsx.cn/35935.Doc
m.whd.msfsx.cn/55175.Doc
m.whd.msfsx.cn/39119.Doc
m.whd.msfsx.cn/71399.Doc
m.whd.msfsx.cn/73159.Doc
m.whd.msfsx.cn/79395.Doc
m.whd.msfsx.cn/02088.Doc
m.whd.msfsx.cn/39955.Doc
m.whd.msfsx.cn/28448.Doc
m.whd.msfsx.cn/59911.Doc
m.whd.msfsx.cn/39515.Doc
m.whd.msfsx.cn/46200.Doc
m.whd.msfsx.cn/15397.Doc
m.whs.msfsx.cn/80668.Doc
m.whs.msfsx.cn/26006.Doc
m.whs.msfsx.cn/06208.Doc
m.whs.msfsx.cn/68006.Doc
m.whs.msfsx.cn/39959.Doc
m.whs.msfsx.cn/39713.Doc
m.whs.msfsx.cn/57757.Doc
m.whs.msfsx.cn/42004.Doc
m.whs.msfsx.cn/48284.Doc
m.whs.msfsx.cn/17151.Doc
m.whs.msfsx.cn/04406.Doc
m.whs.msfsx.cn/99711.Doc
m.whs.msfsx.cn/66424.Doc
m.whs.msfsx.cn/39735.Doc
m.whs.msfsx.cn/11573.Doc
m.whs.msfsx.cn/62884.Doc
m.whs.msfsx.cn/42084.Doc
m.whs.msfsx.cn/13553.Doc
m.whs.msfsx.cn/20662.Doc
m.whs.msfsx.cn/86000.Doc
m.wha.msfsx.cn/24840.Doc
m.wha.msfsx.cn/26000.Doc
m.wha.msfsx.cn/00440.Doc
m.wha.msfsx.cn/48662.Doc
m.wha.msfsx.cn/62068.Doc
m.wha.msfsx.cn/55139.Doc
m.wha.msfsx.cn/39535.Doc
m.wha.msfsx.cn/35337.Doc
m.wha.msfsx.cn/86284.Doc
m.wha.msfsx.cn/22048.Doc
m.wha.msfsx.cn/11135.Doc
m.wha.msfsx.cn/77999.Doc
m.wha.msfsx.cn/84862.Doc
m.wha.msfsx.cn/46086.Doc
m.wha.msfsx.cn/42084.Doc
m.wha.msfsx.cn/84206.Doc
m.wha.msfsx.cn/51375.Doc
m.wha.msfsx.cn/24060.Doc
m.wha.msfsx.cn/24244.Doc
m.wha.msfsx.cn/79713.Doc
