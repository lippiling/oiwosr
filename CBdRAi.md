鲁囟烙回蔽


Linux iio_trigger_register IIO触发与sysfs触发

IIO（Industrial I/O）子系统的trigger机制是数据采集的同步核心，位于drivers/iio/industrialio-trigger.c和include/linux/iio/trigger.h。trigger决定何时触发一次ADC采样，可以是内部定时器、外部GPIO中断、其他传感器的数据就绪信号，或用户空间的软件触发。核心接口是iio_trigger_register，驱动和trigger模块通过它向IIO核心注册trigger实例。

iio_trigger结构体的关键成员包括name（trigger名称）、ops（trigger操作函数集）、id（设备ID）、subirq_base（子中断号基址）、use_count（使用计数）。ops包含set_trigger_state（使能/禁用trigger）、try_reenable（重新使能，用于边缘触发模式）、validate_device（验证设备是否可以使用该trigger）：

```c
struct iio_trigger_ops {
    int (*set_trigger_state)(struct iio_trigger *trig, bool state);
    int (*reenable)(struct iio_trigger *trig);
    int (*validate_device)(struct iio_trigger *trig,
                           struct iio_dev *indio_dev);
};
```

注册一个trigger的标准流程如下：

```c
struct iio_trigger *trig;

trig = iio_trigger_alloc(dev, "mydev-trigger%d", id);
if (!trig)
    return -ENOMEM;

trig->ops = &my_trigger_ops;
trig->dev.parent = &pdev->dev;

ret = iio_trigger_register(trig);
if (ret < 0) {
    iio_trigger_free(trig);
    return ret;
}
```

iio_trigger_register内部先分配trigger的ID，然后创建/sys/bus/iio/devices/triggerX/目录。注册成功后在sysfs中出现triggerX目录，其中name属性文件记录trigger名称，users_count记录当前注册到该trigger的设备数。

sysfs trigger是IIO框架内置的一种软件触发方式，由drivers/iio/trigger/iio-trig-sysfs.c实现。用户通过echo 1 > triggerX/trigger_now绕过了硬件触发，直接触发一次采样。这特别适合调试和低频数据采集场景。

sysfs trigger的注册路径如下：

```c
static struct iio_trigger *iio_sysfs_trig_alloc(int id)
{
    struct iio_trigger *trig;
    char *name;

    name = kasprintf(GFP_KERNEL, "sysfstrig%d", id);
    trig = iio_trigger_alloc(NULL, name);
    kfree(name);
    if (!trig)
        return NULL;

    trig->ops = &iio_sysfs_trigger_ops;
    return trig;
}
```

sysfs trigger的trigger_now属性写处理函数依次调用iio_trigger_poll，将trigger的pollfunc插入到trigger的workqueue中执行。pollfunc中有IIO设备的数据采集回调，最终调用设备的read_raw或push_to_buffer接口：

```c
static ssize_t iio_sysfs_trig_add_store(struct device *dev,
                                        struct device_attribute *attr,
                                        const char *buf, size_t len)
{
    struct iio_trigger *trig;
    int ret, n;

    ret = kstrtoint(buf, 10, &n);
    if (ret)
        return ret;

    mutex_lock(&iio_sysfs_trig_list_mut);
    trig = iio_sysfs_trig_alloc(n);
    if (!trig) {
        mutex_unlock(&iio_sysfs_trig_list_mut);
        return -ENOMEM;
    }

    ret = iio_trigger_register(trig);
    if (ret < 0) {
        iio_trigger_free(trig);
        mutex_unlock(&iio_sysfs_trig_list_mut);
        return ret;
    }
    list_add_tail(&trig->alloc_list, &iio_sysfs_trig_list);
    mutex_unlock(&iio_sysfs_trig_list_mut);

    return len;
}
```

trigger与IIO设备的绑定通过current_trigger属性实现。当用户写current_trigger时，IIO核心调用iio_trigger_attach_pollfunc将设备的pollfunc注册到trigger上，然后调用trigger的set_trigger_state(true)使能trigger：

```c
int iio_trigger_attach_pollfunc(struct iio_dev *indio_dev,
                                struct iio_poll_func *pf)
{
    struct iio_trigger *trig = indio_dev->trig;

    /* 将pollfunc挂入trigger的subirq链 */
    trig->pf = pf;
    return 0;
}
```

实际的触发数据流路径如下。当trigger被触发（无论是硬件中断还是sysfs写入），调用iio_trigger_poll。该函数检查trigger的use_count，只有至少一个设备绑定时才执行。它通过generic_handle_irq(trig->subirq_base+0)触发与trigger关联的虚拟中断，最终执行到pollfunc：

```c
static void iio_trigger_poll(struct iio_trigger *trig)
{
    int i;

    if (!trig->use_count)
        return;

    for (i = 0; i < CONFIG_IIO_CONSUMERS_PER_TRIGGER; i++) {
        if (trig->subirqs[i].enabled)
            generic_handle_irq(trig->subirq_base + i);
    }
}
```

pollfunc通常调用iio_trigger_notify_done通知trigger本次处理完成，随后调用iio_trigger_reenable解除trigger的屏蔽状态。对于sysfs trigger，pollfunc的典型实现直接调用设备的采样函数并将数据推入buffer：

```c
static irqreturn_t my_iio_trigger_handler(int irq, void *private)
{
    struct iio_poll_func *pf = private;
    struct iio_dev *indio_dev = pf->indio_dev;
    struct my_sensor_state *st = iio_priv(indio_dev);
    s16 data[8];
    int i;

    /* 读取所有已使能通道的数据 */
    for_each_set_bit(i, indio_dev->active_scan_mask,
                     indio_dev->masklength) {
        data[i] = my_sensor_read_channel(st, i);
    }

    iio_push_to_buffers(indio_dev, data);
    iio_trigger_notify_done(indio_dev->trig);

    return IRQ_HANDLED;
}
```

iio_trigger_register还负责管理trigger的引用计数。当设备绑定trigger时use_count递增，解绑时递减。use_count降为0时，trigger的set_trigger_state(false)会被调用以节省功耗。iio_trigger_unregister执行反向操作，删除sysfs节点并从全局trigger列表中移除，同时等待可能正在执行的pollfunc完成。

trigger的数据就绪信号可以来自片内外设的硬件引脚。例如加速度计INT1引脚连接到SoC的GPIO，GPIO中断处理程序中调用iio_trigger_poll。这种情况下，trigger的ops中需要实现try_reenable来清除硬件中断状态。如果try_reenable被设置为NULL，trigger使用电平触发模式，无需重新使能操作。

多个IIO设备可以共享同一个trigger，这在多传感器同步采集中至关重要。例如一个加速度计和一个陀螺仪共享一个外部trigger信号，确保两种传感器的采样在时间上严格对齐。每个设备通过current_trigger属性指向同一个trigger名字即可实现共享。

蚀展诜揪易背陈掏没浪毡缚林胀颊

m.wqn.hzldf.cn/82882.Doc
m.wqn.hzldf.cn/82222.Doc
m.wqn.hzldf.cn/64266.Doc
m.wqn.hzldf.cn/00286.Doc
m.wqn.hzldf.cn/20002.Doc
m.wqn.hzldf.cn/40820.Doc
m.wqn.hzldf.cn/20200.Doc
m.wqn.hzldf.cn/20442.Doc
m.wqn.hzldf.cn/77797.Doc
m.wqn.hzldf.cn/99937.Doc
m.wqn.hzldf.cn/82888.Doc
m.wqn.hzldf.cn/04482.Doc
m.wqn.hzldf.cn/46240.Doc
m.wqn.hzldf.cn/04240.Doc
m.wqn.hzldf.cn/00486.Doc
m.wqb.hzldf.cn/37517.Doc
m.wqb.hzldf.cn/75711.Doc
m.wqb.hzldf.cn/17195.Doc
m.wqb.hzldf.cn/86664.Doc
m.wqb.hzldf.cn/64220.Doc
m.wqb.hzldf.cn/08408.Doc
m.wqb.hzldf.cn/60026.Doc
m.wqb.hzldf.cn/17735.Doc
m.wqb.hzldf.cn/79313.Doc
m.wqb.hzldf.cn/66028.Doc
m.wqb.hzldf.cn/11753.Doc
m.wqb.hzldf.cn/80202.Doc
m.wqb.hzldf.cn/80066.Doc
m.wqb.hzldf.cn/73515.Doc
m.wqb.hzldf.cn/77595.Doc
m.wqb.hzldf.cn/73575.Doc
m.wqb.hzldf.cn/64480.Doc
m.wqb.hzldf.cn/40860.Doc
m.wqb.hzldf.cn/44226.Doc
m.wqb.hzldf.cn/77171.Doc
m.wqv.hzldf.cn/40088.Doc
m.wqv.hzldf.cn/62608.Doc
m.wqv.hzldf.cn/99397.Doc
m.wqv.hzldf.cn/04604.Doc
m.wqv.hzldf.cn/48822.Doc
m.wqv.hzldf.cn/88042.Doc
m.wqv.hzldf.cn/48644.Doc
m.wqv.hzldf.cn/57319.Doc
m.wqv.hzldf.cn/91919.Doc
m.wqv.hzldf.cn/17575.Doc
m.wqv.hzldf.cn/80248.Doc
m.wqv.hzldf.cn/46666.Doc
m.wqv.hzldf.cn/37313.Doc
m.wqv.hzldf.cn/42624.Doc
m.wqv.hzldf.cn/71313.Doc
m.wqv.hzldf.cn/68264.Doc
m.wqv.hzldf.cn/88020.Doc
m.wqv.hzldf.cn/62686.Doc
m.wqv.hzldf.cn/44202.Doc
m.wqv.hzldf.cn/06626.Doc
m.wqc.hzldf.cn/53559.Doc
m.wqc.hzldf.cn/82260.Doc
m.wqc.hzldf.cn/57395.Doc
m.wqc.hzldf.cn/99911.Doc
m.wqc.hzldf.cn/48420.Doc
m.wqc.hzldf.cn/20802.Doc
m.wqc.hzldf.cn/13959.Doc
m.wqc.hzldf.cn/20004.Doc
m.wqc.hzldf.cn/71959.Doc
m.wqc.hzldf.cn/28460.Doc
m.wqc.hzldf.cn/31371.Doc
m.wqc.hzldf.cn/97997.Doc
m.wqc.hzldf.cn/44224.Doc
m.wqc.hzldf.cn/22268.Doc
m.wqc.hzldf.cn/33731.Doc
m.wqc.hzldf.cn/28642.Doc
m.wqc.hzldf.cn/40806.Doc
m.wqc.hzldf.cn/20862.Doc
m.wqc.hzldf.cn/04488.Doc
m.wqc.hzldf.cn/42008.Doc
m.wqx.hzldf.cn/79917.Doc
m.wqx.hzldf.cn/15157.Doc
m.wqx.hzldf.cn/68280.Doc
m.wqx.hzldf.cn/04204.Doc
m.wqx.hzldf.cn/37515.Doc
m.wqx.hzldf.cn/13351.Doc
m.wqx.hzldf.cn/46204.Doc
m.wqx.hzldf.cn/26068.Doc
m.wqx.hzldf.cn/08060.Doc
m.wqx.hzldf.cn/08444.Doc
m.wqx.hzldf.cn/00846.Doc
m.wqx.hzldf.cn/11153.Doc
m.wqx.hzldf.cn/80688.Doc
m.wqx.hzldf.cn/17339.Doc
m.wqx.hzldf.cn/73315.Doc
m.wqx.hzldf.cn/42240.Doc
m.wqx.hzldf.cn/62448.Doc
m.wqx.hzldf.cn/04664.Doc
m.wqx.hzldf.cn/71519.Doc
m.wqx.hzldf.cn/82488.Doc
m.wqz.hzldf.cn/28248.Doc
m.wqz.hzldf.cn/20440.Doc
m.wqz.hzldf.cn/82266.Doc
m.wqz.hzldf.cn/91939.Doc
m.wqz.hzldf.cn/55995.Doc
m.wqz.hzldf.cn/02684.Doc
m.wqz.hzldf.cn/26260.Doc
m.wqz.hzldf.cn/66888.Doc
m.wqz.hzldf.cn/08488.Doc
m.wqz.hzldf.cn/93573.Doc
m.wqz.hzldf.cn/62088.Doc
m.wqz.hzldf.cn/62062.Doc
m.wqz.hzldf.cn/33131.Doc
m.wqz.hzldf.cn/26222.Doc
m.wqz.hzldf.cn/31157.Doc
m.wqz.hzldf.cn/88048.Doc
m.wqz.hzldf.cn/53117.Doc
m.wqz.hzldf.cn/80606.Doc
m.wqz.hzldf.cn/60020.Doc
m.wqz.hzldf.cn/15591.Doc
m.wql.hzldf.cn/84862.Doc
m.wql.hzldf.cn/60840.Doc
m.wql.hzldf.cn/44620.Doc
m.wql.hzldf.cn/39559.Doc
m.wql.hzldf.cn/40844.Doc
m.wql.hzldf.cn/51977.Doc
m.wql.hzldf.cn/55551.Doc
m.wql.hzldf.cn/44440.Doc
m.wql.hzldf.cn/93197.Doc
m.wql.hzldf.cn/04024.Doc
m.wql.hzldf.cn/22468.Doc
m.wql.hzldf.cn/20224.Doc
m.wql.hzldf.cn/75917.Doc
m.wql.hzldf.cn/82642.Doc
m.wql.hzldf.cn/88086.Doc
m.wql.hzldf.cn/86866.Doc
m.wql.hzldf.cn/11133.Doc
m.wql.hzldf.cn/33791.Doc
m.wql.hzldf.cn/73197.Doc
m.wql.hzldf.cn/08244.Doc
m.wqk.hzldf.cn/48262.Doc
m.wqk.hzldf.cn/77155.Doc
m.wqk.hzldf.cn/64264.Doc
m.wqk.hzldf.cn/19599.Doc
m.wqk.hzldf.cn/17715.Doc
m.wqk.hzldf.cn/11595.Doc
m.wqk.hzldf.cn/02660.Doc
m.wqk.hzldf.cn/48606.Doc
m.wqk.hzldf.cn/06000.Doc
m.wqk.hzldf.cn/48686.Doc
m.wqk.hzldf.cn/64600.Doc
m.wqk.hzldf.cn/31195.Doc
m.wqk.hzldf.cn/80286.Doc
m.wqk.hzldf.cn/46826.Doc
m.wqk.hzldf.cn/42464.Doc
m.wqk.hzldf.cn/68446.Doc
m.wqk.hzldf.cn/46480.Doc
m.wqk.hzldf.cn/42422.Doc
m.wqk.hzldf.cn/08280.Doc
m.wqk.hzldf.cn/42626.Doc
m.wqj.hzldf.cn/82608.Doc
m.wqj.hzldf.cn/20488.Doc
m.wqj.hzldf.cn/44468.Doc
m.wqj.hzldf.cn/95737.Doc
m.wqj.hzldf.cn/88446.Doc
m.wqj.hzldf.cn/68086.Doc
m.wqj.hzldf.cn/04626.Doc
m.wqj.hzldf.cn/68682.Doc
m.wqj.hzldf.cn/82226.Doc
m.wqj.hzldf.cn/06846.Doc
m.wqj.hzldf.cn/37311.Doc
m.wqj.hzldf.cn/62864.Doc
m.wqj.hzldf.cn/93357.Doc
m.wqj.hzldf.cn/06804.Doc
m.wqj.hzldf.cn/64444.Doc
m.wqj.hzldf.cn/93911.Doc
m.wqj.hzldf.cn/15197.Doc
m.wqj.hzldf.cn/06868.Doc
m.wqj.hzldf.cn/71791.Doc
m.wqj.hzldf.cn/44282.Doc
m.wqh.hzldf.cn/08224.Doc
m.wqh.hzldf.cn/08482.Doc
m.wqh.hzldf.cn/53399.Doc
m.wqh.hzldf.cn/35195.Doc
m.wqh.hzldf.cn/04408.Doc
m.wqh.hzldf.cn/80288.Doc
m.wqh.hzldf.cn/42448.Doc
m.wqh.hzldf.cn/37915.Doc
m.wqh.hzldf.cn/40606.Doc
m.wqh.hzldf.cn/20628.Doc
m.wqh.hzldf.cn/84864.Doc
m.wqh.hzldf.cn/82088.Doc
m.wqh.hzldf.cn/93553.Doc
m.wqh.hzldf.cn/26824.Doc
m.wqh.hzldf.cn/00800.Doc
m.wqh.hzldf.cn/02244.Doc
m.wqh.hzldf.cn/73597.Doc
m.wqh.hzldf.cn/00620.Doc
m.wqh.hzldf.cn/28666.Doc
m.wqh.hzldf.cn/59735.Doc
m.wqg.hzldf.cn/88644.Doc
m.wqg.hzldf.cn/17975.Doc
m.wqg.hzldf.cn/20604.Doc
m.wqg.hzldf.cn/44622.Doc
m.wqg.hzldf.cn/51319.Doc
m.wqg.hzldf.cn/24060.Doc
m.wqg.hzldf.cn/40664.Doc
m.wqg.hzldf.cn/02824.Doc
m.wqg.hzldf.cn/33113.Doc
m.wqg.hzldf.cn/22826.Doc
m.wqg.hzldf.cn/62846.Doc
m.wqg.hzldf.cn/19953.Doc
m.wqg.hzldf.cn/02686.Doc
m.wqg.hzldf.cn/02842.Doc
m.wqg.hzldf.cn/59151.Doc
m.wqg.hzldf.cn/82860.Doc
m.wqg.hzldf.cn/04000.Doc
m.wqg.hzldf.cn/40866.Doc
m.wqg.hzldf.cn/88668.Doc
m.wqg.hzldf.cn/97933.Doc
m.wqf.hzldf.cn/35939.Doc
m.wqf.hzldf.cn/60866.Doc
m.wqf.hzldf.cn/37573.Doc
m.wqf.hzldf.cn/08002.Doc
m.wqf.hzldf.cn/22022.Doc
m.wqf.hzldf.cn/04006.Doc
m.wqf.hzldf.cn/24682.Doc
m.wqf.hzldf.cn/22688.Doc
m.wqf.hzldf.cn/46668.Doc
m.wqf.hzldf.cn/28666.Doc
m.wqf.hzldf.cn/24444.Doc
m.wqf.hzldf.cn/66080.Doc
m.wqf.hzldf.cn/46822.Doc
m.wqf.hzldf.cn/11995.Doc
m.wqf.hzldf.cn/20004.Doc
m.wqf.hzldf.cn/08442.Doc
m.wqf.hzldf.cn/04428.Doc
m.wqf.hzldf.cn/11735.Doc
m.wqf.hzldf.cn/08046.Doc
m.wqf.hzldf.cn/55135.Doc
m.wqd.hzldf.cn/62024.Doc
m.wqd.hzldf.cn/20624.Doc
m.wqd.hzldf.cn/13919.Doc
m.wqd.hzldf.cn/60208.Doc
m.wqd.hzldf.cn/68226.Doc
m.wqd.hzldf.cn/88488.Doc
m.wqd.hzldf.cn/15551.Doc
m.wqd.hzldf.cn/20088.Doc
m.wqd.hzldf.cn/39391.Doc
m.wqd.hzldf.cn/06406.Doc
m.wqd.hzldf.cn/13911.Doc
m.wqd.hzldf.cn/5.Doc
m.wqd.hzldf.cn/60688.Doc
m.wqd.hzldf.cn/48860.Doc
m.wqd.hzldf.cn/73173.Doc
m.wqd.hzldf.cn/68466.Doc
m.wqd.hzldf.cn/28224.Doc
m.wqd.hzldf.cn/68202.Doc
m.wqd.hzldf.cn/04424.Doc
m.wqd.hzldf.cn/51791.Doc
m.wqs.hzldf.cn/97159.Doc
m.wqs.hzldf.cn/08004.Doc
m.wqs.hzldf.cn/26600.Doc
m.wqs.hzldf.cn/19777.Doc
m.wqs.hzldf.cn/15937.Doc
m.wqs.hzldf.cn/84606.Doc
m.wqs.hzldf.cn/08200.Doc
m.wqs.hzldf.cn/66244.Doc
m.wqs.hzldf.cn/17575.Doc
m.wqs.hzldf.cn/46042.Doc
m.wqs.hzldf.cn/28824.Doc
m.wqs.hzldf.cn/42680.Doc
m.wqs.hzldf.cn/22008.Doc
m.wqs.hzldf.cn/35111.Doc
m.wqs.hzldf.cn/71973.Doc
m.wqs.hzldf.cn/42620.Doc
m.wqs.hzldf.cn/40860.Doc
m.wqs.hzldf.cn/60624.Doc
m.wqs.hzldf.cn/40240.Doc
m.wqs.hzldf.cn/44660.Doc
m.wqa.hzldf.cn/48620.Doc
m.wqa.hzldf.cn/62086.Doc
m.wqa.hzldf.cn/60066.Doc
m.wqa.hzldf.cn/24084.Doc
m.wqa.hzldf.cn/08084.Doc
m.wqa.hzldf.cn/04228.Doc
m.wqa.hzldf.cn/20266.Doc
m.wqa.hzldf.cn/73399.Doc
m.wqa.hzldf.cn/13313.Doc
m.wqa.hzldf.cn/60004.Doc
m.wqa.hzldf.cn/84068.Doc
m.wqa.hzldf.cn/86200.Doc
m.wqa.hzldf.cn/60086.Doc
m.wqa.hzldf.cn/37795.Doc
m.wqa.hzldf.cn/64668.Doc
m.wqa.hzldf.cn/24480.Doc
m.wqa.hzldf.cn/28868.Doc
m.wqa.hzldf.cn/95573.Doc
m.wqa.hzldf.cn/42486.Doc
m.wqa.hzldf.cn/26426.Doc
m.wqp.hzldf.cn/60840.Doc
m.wqp.hzldf.cn/13193.Doc
m.wqp.hzldf.cn/53133.Doc
m.wqp.hzldf.cn/08466.Doc
m.wqp.hzldf.cn/64840.Doc
m.wqp.hzldf.cn/42004.Doc
m.wqp.hzldf.cn/86224.Doc
m.wqp.hzldf.cn/20220.Doc
m.wqp.hzldf.cn/15139.Doc
m.wqp.hzldf.cn/26402.Doc
m.wqp.hzldf.cn/35717.Doc
m.wqp.hzldf.cn/59191.Doc
m.wqp.hzldf.cn/40868.Doc
m.wqp.hzldf.cn/75915.Doc
m.wqp.hzldf.cn/26002.Doc
m.wqp.hzldf.cn/44660.Doc
m.wqp.hzldf.cn/68026.Doc
m.wqp.hzldf.cn/48480.Doc
m.wqp.hzldf.cn/66202.Doc
m.wqp.hzldf.cn/80666.Doc
m.wqo.hzldf.cn/26422.Doc
m.wqo.hzldf.cn/62242.Doc
m.wqo.hzldf.cn/55973.Doc
m.wqo.hzldf.cn/86028.Doc
m.wqo.hzldf.cn/73739.Doc
m.wqo.hzldf.cn/71577.Doc
m.wqo.hzldf.cn/04684.Doc
m.wqo.hzldf.cn/06002.Doc
m.wqo.hzldf.cn/80624.Doc
m.wqo.hzldf.cn/84860.Doc
m.wqo.hzldf.cn/82866.Doc
m.wqo.hzldf.cn/13775.Doc
m.wqo.hzldf.cn/13919.Doc
m.wqo.hzldf.cn/88242.Doc
m.wqo.hzldf.cn/99757.Doc
m.wqo.hzldf.cn/71757.Doc
m.wqo.hzldf.cn/00220.Doc
m.wqo.hzldf.cn/44468.Doc
m.wqo.hzldf.cn/37131.Doc
m.wqo.hzldf.cn/40426.Doc
m.wqi.hzldf.cn/97557.Doc
m.wqi.hzldf.cn/80608.Doc
m.wqi.hzldf.cn/22420.Doc
m.wqi.hzldf.cn/57919.Doc
m.wqi.hzldf.cn/84084.Doc
m.wqi.hzldf.cn/08802.Doc
m.wqi.hzldf.cn/39139.Doc
m.wqi.hzldf.cn/48622.Doc
m.wqi.hzldf.cn/60420.Doc
m.wqi.hzldf.cn/35997.Doc
m.wqi.hzldf.cn/44486.Doc
m.wqi.hzldf.cn/46066.Doc
m.wqi.hzldf.cn/35951.Doc
m.wqi.hzldf.cn/00480.Doc
m.wqi.hzldf.cn/00688.Doc
m.wqi.hzldf.cn/44080.Doc
m.wqi.hzldf.cn/91173.Doc
m.wqi.hzldf.cn/57711.Doc
m.wqi.hzldf.cn/00420.Doc
m.wqi.hzldf.cn/68620.Doc
m.wqu.hzldf.cn/66224.Doc
m.wqu.hzldf.cn/22600.Doc
m.wqu.hzldf.cn/40260.Doc
m.wqu.hzldf.cn/26288.Doc
m.wqu.hzldf.cn/62026.Doc
m.wqu.hzldf.cn/04844.Doc
m.wqu.hzldf.cn/57555.Doc
m.wqu.hzldf.cn/68002.Doc
m.wqu.hzldf.cn/33999.Doc
m.wqu.hzldf.cn/71957.Doc
m.wqu.hzldf.cn/20866.Doc
m.wqu.hzldf.cn/04648.Doc
m.wqu.hzldf.cn/24488.Doc
m.wqu.hzldf.cn/08006.Doc
m.wqu.hzldf.cn/55315.Doc
m.wqu.hzldf.cn/20442.Doc
m.wqu.hzldf.cn/82020.Doc
m.wqu.hzldf.cn/66260.Doc
m.wqu.hzldf.cn/24260.Doc
m.wqu.hzldf.cn/71319.Doc
m.wqy.hzldf.cn/66608.Doc
m.wqy.hzldf.cn/42422.Doc
m.wqy.hzldf.cn/84066.Doc
m.wqy.hzldf.cn/68020.Doc
m.wqy.hzldf.cn/86022.Doc
m.wqy.hzldf.cn/82600.Doc
m.wqy.hzldf.cn/20086.Doc
m.wqy.hzldf.cn/04020.Doc
m.wqy.hzldf.cn/86286.Doc
m.wqy.hzldf.cn/26260.Doc
m.wqy.hzldf.cn/84824.Doc
m.wqy.hzldf.cn/11559.Doc
m.wqy.hzldf.cn/80480.Doc
m.wqy.hzldf.cn/73131.Doc
m.wqy.hzldf.cn/97579.Doc
m.wqy.hzldf.cn/82688.Doc
m.wqy.hzldf.cn/77555.Doc
m.wqy.hzldf.cn/99173.Doc
m.wqy.hzldf.cn/06482.Doc
m.wqy.hzldf.cn/26082.Doc
