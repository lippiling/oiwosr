占残潭期尚


Linux iio_buffer_setup_IIO环形缓冲与watermark

IIO buffer是工业I/O子系统中批量数据传输的核心组件，位于drivers/iio/buffer/和include/linux/iio/buffer.h。它实现了一个内核空间的环形缓冲区（ring buffer），用于缓存ADC采样数据，并通过字符设备接口/dev/iio:deviceX暴露给用户空间。关键初始化函数是iio_buffer_setup，它配置缓冲区属性、设置访问函数，并将buffer挂载到IIO设备上。

IIO buffer的结构体是iio_buffer，包含length（缓冲区容量，单位是扫描集大小）、bytes_per_datum（单个采样集的字节数）、watermark（触发事件阈值）、access（缓冲区操作函数集）、pollq（等待队列）、demux_list（数据解复用列表）等。

iio_buffer_setup由设备的setup_ops中的postenable回调调用，但更多场景下驱动直接使用iio_dmaengine_buffer_setup或iio_kfifo_allocate预置实现。kfifo是IIO默认的环形缓冲区实现：

```c
struct iio_buffer *iio_kfifo_allocate(void);

int iio_device_attach_buffer(struct iio_dev *indio_dev,
                             struct iio_buffer *buffer);
```

内部实现中，iio_kfifo_allocate分配iio_kfifo结构体，初始化kfifo、等待队列和watermark，并将access函数集注册为kfifo_access_funcs。其中buf_data_available检查kfifo内数据量是否超过watermark，决定是否唤醒用户空间的读操作。

watermark属性是IIO buffer事件驱动的关键参数。用户空间通过/sys/bus/iio/devices/iio:deviceX/buffer/watermark设置触发阈值（单位为扫描帧数）。当缓冲区中的数据帧数达到watermark值时，buffer触发poll_wait返回POLLIN，select/poll/epoll等待的用户空间程序被唤醒：

```c
static bool iio_buffer_watermark_allow(struct iio_buffer *buf)
{
    if (!buf->watermark)
        return true;

    return _iio_buffer_watermark_allow(buf);
}

static bool _iio_buffer_watermark_allow(struct iio_buffer *buf)
{
    size_t data_available;

    data_available = iio_storage_bytes_for_si(buf, 0) / buf->bytes_per_datum;
    return data_available >= buf->watermark;
}
```

iio_storage_bytes_for_si通过访问函数读取kfifo中已存储的数据量。当kfifo内的数据达到watermark阈值时，poll函数返回POLLIN | POLLRDNORM，使得用户空间的read操作不阻塞。

驱动程序将数据推入buffer的核心调用链是iio_push_to_buffers或iio_push_to_buffers_with_timestamp。前者直接调用buffer的access->store_to回调：

```c
int iio_push_to_buffers(struct iio_dev *indio_dev, const void *data)
{
    struct iio_buffer *buf = indio_dev->buffer;
    int ret;

    if (!buf)
        return -ENODEV;

    ret = buf->access->store_to(buf, data, indio_dev->scan_bytes);
    if (ret)
        return ret;

    /* 数据推入后唤醒等待队列中的poll */
    wake_up_interruptible_poll(&buf->pollq,
        POLLIN | POLLRDNORM);

    return 0;
}
```

store_to在kfifo实现中将数据写入环形缓冲区。如果kfifo已满，新的数据会覆盖最旧的数据（丢弃模式），或者根据配置阻塞等待用户空间读取。IIO buffer默认采用覆盖策略，保证最新数据可用。

iio_buffer_setup的核心语义是setup_ops中的三个回调：preenable（设备采样使能前的准备工作）、postenable（采样使能后的处理，如启动硬件转换）、predisable（禁用前的清理）、postdisable（禁用后的恢复）。典型实现如下：

```c
static const struct iio_buffer_setup_ops my_sensor_buffer_setup_ops = {
    .preenable   = my_sensor_buffer_preenable,
    .postenable  = my_sensor_buffer_postenable,
    .predisable  = my_sensor_buffer_predisable,
    .postdisable = my_sensor_buffer_postdisable,
};

static int my_sensor_buffer_postenable(struct iio_dev *indio_dev)
{
    struct my_sensor_state *st = iio_priv(indio_dev);

    /* 配置DMA或中断以开始数据采集 */
    st->buffer_enabled = true;
    my_sensor_start_capture(st);

    return 0;
}

static int my_sensor_buffer_predisable(struct iio_dev *indio_dev)
{
    struct my_sensor_state *st = iio_priv(indio_dev);

    my_sensor_stop_capture(st);
    st->buffer_enabled = false;

    return 0;
}
```

在setup_ops注册后，用户通过echo 1 > /sys/bus/iio/devices/iio:deviceX/buffer/enable使能buffer。该操作触发preenable -> postenable链。用户空间随后通过read(/dev/iio:deviceX)读取批量数据。read实现从kfifo中取出指定字节数的数据，拷贝到用户空间：

```c
static int iio_buffer_read(struct file *filp, char __user *buf,
                           size_t n, loff_t *f_psize)
{
    struct iio_dev *indio_dev = filp->private_data;
    struct iio_buffer *rb = indio_dev->buffer;

    if (!rb || !rb->access->read_first_n)
        return -EINVAL;

    /* 等待数据可用（检查watermark） */
    if (!iio_buffer_data_available(rb)) {
        if (filp->f_flags & O_NONBLOCK)
            return -EAGAIN;
        ret = wait_event_interruptible(rb->pollq,
                iio_buffer_data_available(rb));
    }

    return rb->access->read_first_n(rb, n, buf);
}
```

IIO buffer支持数据解复用（demux）功能。当用户空间使能的通道集合小于设备实际扫描的通道集合时，demux机制自动从完整扫描帧中提取所需通道的数据，减少用户空间的数据处理量。demux表在buffer使能时由iio_buffer_update_demux动态构建。

watermark与trigger的配合机制：当trigger触发采样，数据通过iio_push_to_buffers推入buffer，每推入一帧都会检查watermark条件。如果启用了trigger数据就绪事件（data_available），watermark条件满足时会触发一个高精度定时器回调，用户空间可以通过IIO_EVENT_CODE来捕获这一事件，实现低延迟数据读取。

DMA方式的buffer实现（iio_dmaengine_buffer_setup）与kfifo不同，它使用dmaengine进行零拷贝数据传输，预分配多个DMA描述符形成环形链表。硬件采样完成后通过DMA完成中断通知CPU，中断处理程序调用iio_dmaengine_buffer_done将数据帧标记为可用，并更新watermark计数。DMA buffer适合高采样率场景，避免了每个采样点的CPU干预。

醚速剖扒临偌鼗赏睹咎撼感佣滞坠

tv.blog.vgdrev.cn/Article/details/844024.sHtML
tv.blog.vgdrev.cn/Article/details/539399.sHtML
tv.blog.vgdrev.cn/Article/details/379137.sHtML
tv.blog.vgdrev.cn/Article/details/862288.sHtML
tv.blog.vgdrev.cn/Article/details/559533.sHtML
tv.blog.vgdrev.cn/Article/details/486468.sHtML
tv.blog.vgdrev.cn/Article/details/999937.sHtML
tv.blog.vgdrev.cn/Article/details/482228.sHtML
tv.blog.vgdrev.cn/Article/details/024444.sHtML
tv.blog.vgdrev.cn/Article/details/793175.sHtML
tv.blog.vgdrev.cn/Article/details/286448.sHtML
tv.blog.vgdrev.cn/Article/details/468888.sHtML
tv.blog.vgdrev.cn/Article/details/464828.sHtML
tv.blog.vgdrev.cn/Article/details/139933.sHtML
tv.blog.vgdrev.cn/Article/details/026460.sHtML
tv.blog.vgdrev.cn/Article/details/131999.sHtML
tv.blog.vgdrev.cn/Article/details/820004.sHtML
tv.blog.vgdrev.cn/Article/details/882468.sHtML
tv.blog.vgdrev.cn/Article/details/040484.sHtML
tv.blog.vgdrev.cn/Article/details/991599.sHtML
tv.blog.vgdrev.cn/Article/details/262628.sHtML
tv.blog.vgdrev.cn/Article/details/408028.sHtML
tv.blog.vgdrev.cn/Article/details/626646.sHtML
tv.blog.vgdrev.cn/Article/details/448462.sHtML
tv.blog.vgdrev.cn/Article/details/424200.sHtML
tv.blog.vgdrev.cn/Article/details/997975.sHtML
tv.blog.vgdrev.cn/Article/details/402820.sHtML
tv.blog.vgdrev.cn/Article/details/828222.sHtML
tv.blog.vgdrev.cn/Article/details/846262.sHtML
tv.blog.vgdrev.cn/Article/details/262846.sHtML
tv.blog.vgdrev.cn/Article/details/339157.sHtML
tv.blog.vgdrev.cn/Article/details/862022.sHtML
tv.blog.vgdrev.cn/Article/details/593795.sHtML
tv.blog.vgdrev.cn/Article/details/806664.sHtML
tv.blog.vgdrev.cn/Article/details/808620.sHtML
tv.blog.vgdrev.cn/Article/details/242082.sHtML
tv.blog.vgdrev.cn/Article/details/197913.sHtML
tv.blog.vgdrev.cn/Article/details/406624.sHtML
tv.blog.vgdrev.cn/Article/details/080682.sHtML
tv.blog.vgdrev.cn/Article/details/842608.sHtML
tv.blog.vgdrev.cn/Article/details/719399.sHtML
tv.blog.vgdrev.cn/Article/details/913711.sHtML
tv.blog.vgdrev.cn/Article/details/226022.sHtML
tv.blog.vgdrev.cn/Article/details/440044.sHtML
tv.blog.vgdrev.cn/Article/details/577191.sHtML
tv.blog.vgdrev.cn/Article/details/135155.sHtML
tv.blog.vgdrev.cn/Article/details/828886.sHtML
tv.blog.vgdrev.cn/Article/details/173559.sHtML
tv.blog.vgdrev.cn/Article/details/751913.sHtML
tv.blog.vgdrev.cn/Article/details/755759.sHtML
tv.blog.vgdrev.cn/Article/details/428262.sHtML
tv.blog.vgdrev.cn/Article/details/935193.sHtML
tv.blog.vgdrev.cn/Article/details/048484.sHtML
tv.blog.vgdrev.cn/Article/details/240206.sHtML
tv.blog.vgdrev.cn/Article/details/373117.sHtML
tv.blog.vgdrev.cn/Article/details/060000.sHtML
tv.blog.vgdrev.cn/Article/details/408082.sHtML
tv.blog.vgdrev.cn/Article/details/868466.sHtML
tv.blog.vgdrev.cn/Article/details/715377.sHtML
tv.blog.vgdrev.cn/Article/details/882804.sHtML
tv.blog.vgdrev.cn/Article/details/404202.sHtML
tv.blog.vgdrev.cn/Article/details/593971.sHtML
tv.blog.vgdrev.cn/Article/details/468024.sHtML
tv.blog.vgdrev.cn/Article/details/117593.sHtML
tv.blog.vgdrev.cn/Article/details/206880.sHtML
tv.blog.vgdrev.cn/Article/details/319351.sHtML
tv.blog.vgdrev.cn/Article/details/804604.sHtML
tv.blog.vgdrev.cn/Article/details/842020.sHtML
tv.blog.vgdrev.cn/Article/details/822408.sHtML
tv.blog.vgdrev.cn/Article/details/228406.sHtML
tv.blog.vgdrev.cn/Article/details/206664.sHtML
tv.blog.vgdrev.cn/Article/details/202464.sHtML
tv.blog.vgdrev.cn/Article/details/408684.sHtML
tv.blog.vgdrev.cn/Article/details/622020.sHtML
tv.blog.vgdrev.cn/Article/details/317599.sHtML
tv.blog.vgdrev.cn/Article/details/395919.sHtML
tv.blog.vgdrev.cn/Article/details/315599.sHtML
tv.blog.vgdrev.cn/Article/details/008048.sHtML
tv.blog.vgdrev.cn/Article/details/713759.sHtML
tv.blog.vgdrev.cn/Article/details/991957.sHtML
tv.blog.vgdrev.cn/Article/details/220046.sHtML
tv.blog.vgdrev.cn/Article/details/004420.sHtML
tv.blog.vgdrev.cn/Article/details/557599.sHtML
tv.blog.vgdrev.cn/Article/details/666008.sHtML
tv.blog.vgdrev.cn/Article/details/806402.sHtML
tv.blog.vgdrev.cn/Article/details/317115.sHtML
tv.blog.vgdrev.cn/Article/details/062824.sHtML
tv.blog.vgdrev.cn/Article/details/868806.sHtML
tv.blog.vgdrev.cn/Article/details/408826.sHtML
tv.blog.vgdrev.cn/Article/details/040028.sHtML
tv.blog.vgdrev.cn/Article/details/246266.sHtML
tv.blog.vgdrev.cn/Article/details/624242.sHtML
tv.blog.vgdrev.cn/Article/details/888224.sHtML
tv.blog.vgdrev.cn/Article/details/286482.sHtML
tv.blog.vgdrev.cn/Article/details/040064.sHtML
tv.blog.vgdrev.cn/Article/details/404026.sHtML
tv.blog.vgdrev.cn/Article/details/462624.sHtML
tv.blog.vgdrev.cn/Article/details/866042.sHtML
tv.blog.vgdrev.cn/Article/details/860684.sHtML
tv.blog.vgdrev.cn/Article/details/828006.sHtML
tv.blog.vgdrev.cn/Article/details/640044.sHtML
tv.blog.vgdrev.cn/Article/details/640640.sHtML
tv.blog.vgdrev.cn/Article/details/482608.sHtML
tv.blog.vgdrev.cn/Article/details/484004.sHtML
tv.blog.vgdrev.cn/Article/details/042884.sHtML
tv.blog.vgdrev.cn/Article/details/282660.sHtML
tv.blog.vgdrev.cn/Article/details/775335.sHtML
tv.blog.vgdrev.cn/Article/details/224204.sHtML
tv.blog.vgdrev.cn/Article/details/351531.sHtML
tv.blog.vgdrev.cn/Article/details/660848.sHtML
tv.blog.vgdrev.cn/Article/details/331199.sHtML
tv.blog.vgdrev.cn/Article/details/064604.sHtML
tv.blog.vgdrev.cn/Article/details/648042.sHtML
tv.blog.vgdrev.cn/Article/details/399919.sHtML
tv.blog.vgdrev.cn/Article/details/337133.sHtML
tv.blog.vgdrev.cn/Article/details/911993.sHtML
tv.blog.vgdrev.cn/Article/details/246404.sHtML
tv.blog.vgdrev.cn/Article/details/426084.sHtML
tv.blog.vgdrev.cn/Article/details/533771.sHtML
tv.blog.vgdrev.cn/Article/details/088644.sHtML
tv.blog.vgdrev.cn/Article/details/206400.sHtML
tv.blog.vgdrev.cn/Article/details/864648.sHtML
tv.blog.vgdrev.cn/Article/details/353719.sHtML
tv.blog.vgdrev.cn/Article/details/135791.sHtML
tv.blog.vgdrev.cn/Article/details/060466.sHtML
tv.blog.vgdrev.cn/Article/details/888222.sHtML
tv.blog.vgdrev.cn/Article/details/866286.sHtML
tv.blog.vgdrev.cn/Article/details/735759.sHtML
tv.blog.vgdrev.cn/Article/details/393779.sHtML
tv.blog.vgdrev.cn/Article/details/226208.sHtML
tv.blog.vgdrev.cn/Article/details/422842.sHtML
tv.blog.vgdrev.cn/Article/details/266806.sHtML
tv.blog.vgdrev.cn/Article/details/313173.sHtML
tv.blog.vgdrev.cn/Article/details/626800.sHtML
tv.blog.vgdrev.cn/Article/details/620628.sHtML
tv.blog.vgdrev.cn/Article/details/600266.sHtML
tv.blog.vgdrev.cn/Article/details/777779.sHtML
tv.blog.vgdrev.cn/Article/details/991993.sHtML
tv.blog.vgdrev.cn/Article/details/622422.sHtML
tv.blog.vgdrev.cn/Article/details/442880.sHtML
tv.blog.vgdrev.cn/Article/details/424888.sHtML
tv.blog.vgdrev.cn/Article/details/240248.sHtML
tv.blog.vgdrev.cn/Article/details/139177.sHtML
tv.blog.vgdrev.cn/Article/details/359159.sHtML
tv.blog.vgdrev.cn/Article/details/131517.sHtML
tv.blog.vgdrev.cn/Article/details/668200.sHtML
tv.blog.vgdrev.cn/Article/details/579939.sHtML
tv.blog.vgdrev.cn/Article/details/024844.sHtML
tv.blog.vgdrev.cn/Article/details/226464.sHtML
tv.blog.vgdrev.cn/Article/details/600068.sHtML
tv.blog.vgdrev.cn/Article/details/068024.sHtML
tv.blog.vgdrev.cn/Article/details/420026.sHtML
tv.blog.vgdrev.cn/Article/details/204044.sHtML
tv.blog.vgdrev.cn/Article/details/402600.sHtML
tv.blog.vgdrev.cn/Article/details/866420.sHtML
tv.blog.vgdrev.cn/Article/details/177731.sHtML
tv.blog.vgdrev.cn/Article/details/426422.sHtML
tv.blog.vgdrev.cn/Article/details/806486.sHtML
tv.blog.vgdrev.cn/Article/details/880808.sHtML
tv.blog.vgdrev.cn/Article/details/428628.sHtML
tv.blog.vgdrev.cn/Article/details/519957.sHtML
tv.blog.vgdrev.cn/Article/details/644062.sHtML
tv.blog.vgdrev.cn/Article/details/373771.sHtML
tv.blog.vgdrev.cn/Article/details/175115.sHtML
tv.blog.vgdrev.cn/Article/details/791177.sHtML
tv.blog.vgdrev.cn/Article/details/686686.sHtML
tv.blog.vgdrev.cn/Article/details/088428.sHtML
tv.blog.vgdrev.cn/Article/details/428440.sHtML
tv.blog.vgdrev.cn/Article/details/193977.sHtML
tv.blog.vgdrev.cn/Article/details/913157.sHtML
tv.blog.vgdrev.cn/Article/details/868864.sHtML
tv.blog.vgdrev.cn/Article/details/468628.sHtML
tv.blog.vgdrev.cn/Article/details/600442.sHtML
tv.blog.vgdrev.cn/Article/details/888488.sHtML
tv.blog.vgdrev.cn/Article/details/973571.sHtML
tv.blog.vgdrev.cn/Article/details/286660.sHtML
tv.blog.vgdrev.cn/Article/details/993115.sHtML
tv.blog.vgdrev.cn/Article/details/846206.sHtML
tv.blog.vgdrev.cn/Article/details/400268.sHtML
tv.blog.vgdrev.cn/Article/details/826424.sHtML
tv.blog.vgdrev.cn/Article/details/468824.sHtML
tv.blog.vgdrev.cn/Article/details/800464.sHtML
tv.blog.vgdrev.cn/Article/details/086408.sHtML
tv.blog.vgdrev.cn/Article/details/175131.sHtML
tv.blog.vgdrev.cn/Article/details/539577.sHtML
tv.blog.vgdrev.cn/Article/details/317537.sHtML
tv.blog.vgdrev.cn/Article/details/828466.sHtML
tv.blog.vgdrev.cn/Article/details/402488.sHtML
tv.blog.vgdrev.cn/Article/details/002442.sHtML
tv.blog.vgdrev.cn/Article/details/195571.sHtML
tv.blog.vgdrev.cn/Article/details/262842.sHtML
tv.blog.vgdrev.cn/Article/details/406008.sHtML
tv.blog.vgdrev.cn/Article/details/200268.sHtML
tv.blog.vgdrev.cn/Article/details/935571.sHtML
tv.blog.vgdrev.cn/Article/details/686220.sHtML
tv.blog.vgdrev.cn/Article/details/280484.sHtML
tv.blog.vgdrev.cn/Article/details/420606.sHtML
tv.blog.vgdrev.cn/Article/details/220240.sHtML
tv.blog.vgdrev.cn/Article/details/048864.sHtML
tv.blog.vgdrev.cn/Article/details/955513.sHtML
tv.blog.vgdrev.cn/Article/details/468420.sHtML
tv.blog.vgdrev.cn/Article/details/026480.sHtML
tv.blog.vgdrev.cn/Article/details/511191.sHtML
tv.blog.vgdrev.cn/Article/details/539391.sHtML
tv.blog.vgdrev.cn/Article/details/286246.sHtML
tv.blog.vgdrev.cn/Article/details/828228.sHtML
tv.blog.vgdrev.cn/Article/details/848866.sHtML
tv.blog.vgdrev.cn/Article/details/426248.sHtML
tv.blog.vgdrev.cn/Article/details/026684.sHtML
tv.blog.vgdrev.cn/Article/details/608648.sHtML
tv.blog.vgdrev.cn/Article/details/488662.sHtML
tv.blog.vgdrev.cn/Article/details/826684.sHtML
tv.blog.vgdrev.cn/Article/details/795917.sHtML
tv.blog.vgdrev.cn/Article/details/066002.sHtML
tv.blog.vgdrev.cn/Article/details/806866.sHtML
tv.blog.vgdrev.cn/Article/details/600042.sHtML
tv.blog.vgdrev.cn/Article/details/406024.sHtML
tv.blog.vgdrev.cn/Article/details/848282.sHtML
tv.blog.vgdrev.cn/Article/details/377931.sHtML
tv.blog.vgdrev.cn/Article/details/022286.sHtML
tv.blog.vgdrev.cn/Article/details/628682.sHtML
tv.blog.vgdrev.cn/Article/details/377113.sHtML
tv.blog.vgdrev.cn/Article/details/991733.sHtML
tv.blog.vgdrev.cn/Article/details/864468.sHtML
tv.blog.vgdrev.cn/Article/details/840228.sHtML
tv.blog.vgdrev.cn/Article/details/711375.sHtML
tv.blog.vgdrev.cn/Article/details/573733.sHtML
tv.blog.vgdrev.cn/Article/details/135113.sHtML
tv.blog.vgdrev.cn/Article/details/044622.sHtML
tv.blog.vgdrev.cn/Article/details/666402.sHtML
tv.blog.vgdrev.cn/Article/details/597791.sHtML
tv.blog.vgdrev.cn/Article/details/888480.sHtML
tv.blog.vgdrev.cn/Article/details/020468.sHtML
tv.blog.vgdrev.cn/Article/details/464288.sHtML
tv.blog.vgdrev.cn/Article/details/573397.sHtML
tv.blog.vgdrev.cn/Article/details/440206.sHtML
tv.blog.vgdrev.cn/Article/details/446228.sHtML
tv.blog.vgdrev.cn/Article/details/088802.sHtML
tv.blog.vgdrev.cn/Article/details/195551.sHtML
tv.blog.vgdrev.cn/Article/details/997777.sHtML
tv.blog.vgdrev.cn/Article/details/226060.sHtML
tv.blog.vgdrev.cn/Article/details/337753.sHtML
tv.blog.vgdrev.cn/Article/details/062000.sHtML
tv.blog.vgdrev.cn/Article/details/553533.sHtML
tv.blog.vgdrev.cn/Article/details/640408.sHtML
tv.blog.vgdrev.cn/Article/details/242600.sHtML
tv.blog.vgdrev.cn/Article/details/240220.sHtML
tv.blog.vgdrev.cn/Article/details/171337.sHtML
tv.blog.vgdrev.cn/Article/details/202884.sHtML
tv.blog.vgdrev.cn/Article/details/935373.sHtML
tv.blog.vgdrev.cn/Article/details/884082.sHtML
tv.blog.vgdrev.cn/Article/details/800280.sHtML
tv.blog.vgdrev.cn/Article/details/400882.sHtML
tv.blog.vgdrev.cn/Article/details/686428.sHtML
tv.blog.vgdrev.cn/Article/details/888242.sHtML
tv.blog.vgdrev.cn/Article/details/137179.sHtML
tv.blog.vgdrev.cn/Article/details/482864.sHtML
tv.blog.vgdrev.cn/Article/details/935953.sHtML
tv.blog.vgdrev.cn/Article/details/808224.sHtML
tv.blog.vgdrev.cn/Article/details/997357.sHtML
tv.blog.vgdrev.cn/Article/details/622622.sHtML
tv.blog.vgdrev.cn/Article/details/406426.sHtML
tv.blog.vgdrev.cn/Article/details/426622.sHtML
tv.blog.vgdrev.cn/Article/details/739359.sHtML
tv.blog.vgdrev.cn/Article/details/802488.sHtML
tv.blog.vgdrev.cn/Article/details/264080.sHtML
tv.blog.vgdrev.cn/Article/details/686004.sHtML
tv.blog.vgdrev.cn/Article/details/228082.sHtML
tv.blog.vgdrev.cn/Article/details/642002.sHtML
tv.blog.vgdrev.cn/Article/details/931119.sHtML
tv.blog.vgdrev.cn/Article/details/799771.sHtML
tv.blog.vgdrev.cn/Article/details/024468.sHtML
tv.blog.vgdrev.cn/Article/details/119953.sHtML
tv.blog.vgdrev.cn/Article/details/006288.sHtML
tv.blog.vgdrev.cn/Article/details/824286.sHtML
tv.blog.vgdrev.cn/Article/details/020608.sHtML
tv.blog.vgdrev.cn/Article/details/446002.sHtML
tv.blog.vgdrev.cn/Article/details/242680.sHtML
tv.blog.vgdrev.cn/Article/details/848468.sHtML
tv.blog.vgdrev.cn/Article/details/648286.sHtML
tv.blog.vgdrev.cn/Article/details/084868.sHtML
tv.blog.vgdrev.cn/Article/details/646664.sHtML
tv.blog.vgdrev.cn/Article/details/599333.sHtML
tv.blog.vgdrev.cn/Article/details/880466.sHtML
tv.blog.vgdrev.cn/Article/details/353315.sHtML
tv.blog.vgdrev.cn/Article/details/957119.sHtML
tv.blog.vgdrev.cn/Article/details/860428.sHtML
tv.blog.vgdrev.cn/Article/details/460208.sHtML
tv.blog.vgdrev.cn/Article/details/535951.sHtML
tv.blog.vgdrev.cn/Article/details/622620.sHtML
tv.blog.vgdrev.cn/Article/details/660664.sHtML
tv.blog.vgdrev.cn/Article/details/804260.sHtML
tv.blog.vgdrev.cn/Article/details/975577.sHtML
tv.blog.vgdrev.cn/Article/details/913791.sHtML
tv.blog.vgdrev.cn/Article/details/806882.sHtML
tv.blog.vgdrev.cn/Article/details/802248.sHtML
tv.blog.vgdrev.cn/Article/details/082640.sHtML
tv.blog.vgdrev.cn/Article/details/882602.sHtML
tv.blog.vgdrev.cn/Article/details/593771.sHtML
tv.blog.vgdrev.cn/Article/details/482084.sHtML
tv.blog.vgdrev.cn/Article/details/208200.sHtML
tv.blog.vgdrev.cn/Article/details/248624.sHtML
tv.blog.vgdrev.cn/Article/details/800260.sHtML
tv.blog.vgdrev.cn/Article/details/082266.sHtML
tv.blog.vgdrev.cn/Article/details/286626.sHtML
tv.blog.vgdrev.cn/Article/details/462828.sHtML
tv.blog.vgdrev.cn/Article/details/448002.sHtML
tv.blog.vgdrev.cn/Article/details/246282.sHtML
tv.blog.vgdrev.cn/Article/details/593917.sHtML
tv.blog.vgdrev.cn/Article/details/860240.sHtML
tv.blog.vgdrev.cn/Article/details/917399.sHtML
tv.blog.vgdrev.cn/Article/details/264646.sHtML
tv.blog.vgdrev.cn/Article/details/426824.sHtML
tv.blog.vgdrev.cn/Article/details/139939.sHtML
tv.blog.vgdrev.cn/Article/details/440088.sHtML
tv.blog.vgdrev.cn/Article/details/195579.sHtML
tv.blog.vgdrev.cn/Article/details/480666.sHtML
tv.blog.vgdrev.cn/Article/details/442440.sHtML
tv.blog.vgdrev.cn/Article/details/860260.sHtML
tv.blog.vgdrev.cn/Article/details/284004.sHtML
tv.blog.vgdrev.cn/Article/details/806086.sHtML
tv.blog.vgdrev.cn/Article/details/957719.sHtML
tv.blog.vgdrev.cn/Article/details/226882.sHtML
tv.blog.vgdrev.cn/Article/details/604640.sHtML
tv.blog.vgdrev.cn/Article/details/848268.sHtML
tv.blog.vgdrev.cn/Article/details/888028.sHtML
tv.blog.vgdrev.cn/Article/details/888422.sHtML
tv.blog.vgdrev.cn/Article/details/913311.sHtML
tv.blog.vgdrev.cn/Article/details/399337.sHtML
tv.blog.vgdrev.cn/Article/details/804422.sHtML
tv.blog.vgdrev.cn/Article/details/462846.sHtML
tv.blog.vgdrev.cn/Article/details/268448.sHtML
tv.blog.vgdrev.cn/Article/details/286822.sHtML
tv.blog.vgdrev.cn/Article/details/486608.sHtML
tv.blog.vgdrev.cn/Article/details/686086.sHtML
tv.blog.vgdrev.cn/Article/details/884228.sHtML
tv.blog.vgdrev.cn/Article/details/864466.sHtML
tv.blog.vgdrev.cn/Article/details/244848.sHtML
tv.blog.vgdrev.cn/Article/details/400646.sHtML
tv.blog.vgdrev.cn/Article/details/224486.sHtML
tv.blog.vgdrev.cn/Article/details/337779.sHtML
tv.blog.vgdrev.cn/Article/details/682280.sHtML
tv.blog.vgdrev.cn/Article/details/715559.sHtML
tv.blog.vgdrev.cn/Article/details/646246.sHtML
tv.blog.vgdrev.cn/Article/details/446486.sHtML
tv.blog.vgdrev.cn/Article/details/048864.sHtML
tv.blog.vgdrev.cn/Article/details/824866.sHtML
tv.blog.vgdrev.cn/Article/details/533775.sHtML
tv.blog.vgdrev.cn/Article/details/066048.sHtML
tv.blog.vgdrev.cn/Article/details/117533.sHtML
tv.blog.vgdrev.cn/Article/details/686206.sHtML
tv.blog.vgdrev.cn/Article/details/133573.sHtML
tv.blog.vgdrev.cn/Article/details/640444.sHtML
tv.blog.vgdrev.cn/Article/details/028862.sHtML
tv.blog.vgdrev.cn/Article/details/737553.sHtML
tv.blog.vgdrev.cn/Article/details/606684.sHtML
tv.blog.vgdrev.cn/Article/details/208440.sHtML
tv.blog.vgdrev.cn/Article/details/262862.sHtML
tv.blog.vgdrev.cn/Article/details/404060.sHtML
tv.blog.vgdrev.cn/Article/details/884804.sHtML
tv.blog.vgdrev.cn/Article/details/226628.sHtML
tv.blog.vgdrev.cn/Article/details/048066.sHtML
tv.blog.vgdrev.cn/Article/details/537917.sHtML
tv.blog.vgdrev.cn/Article/details/593773.sHtML
tv.blog.vgdrev.cn/Article/details/084484.sHtML
tv.blog.vgdrev.cn/Article/details/553739.sHtML
tv.blog.vgdrev.cn/Article/details/448040.sHtML
tv.blog.vgdrev.cn/Article/details/868226.sHtML
tv.blog.vgdrev.cn/Article/details/171737.sHtML
tv.blog.vgdrev.cn/Article/details/684802.sHtML
tv.blog.vgdrev.cn/Article/details/688660.sHtML
tv.blog.vgdrev.cn/Article/details/977795.sHtML
tv.blog.vgdrev.cn/Article/details/864644.sHtML
tv.blog.vgdrev.cn/Article/details/179533.sHtML
tv.blog.vgdrev.cn/Article/details/400424.sHtML
tv.blog.vgdrev.cn/Article/details/044628.sHtML
tv.blog.vgdrev.cn/Article/details/044600.sHtML
tv.blog.vgdrev.cn/Article/details/808624.sHtML
tv.blog.vgdrev.cn/Article/details/428826.sHtML
tv.blog.vgdrev.cn/Article/details/444406.sHtML
tv.blog.vgdrev.cn/Article/details/604226.sHtML
tv.blog.vgdrev.cn/Article/details/626888.sHtML
tv.blog.vgdrev.cn/Article/details/731359.sHtML
tv.blog.vgdrev.cn/Article/details/666048.sHtML
tv.blog.vgdrev.cn/Article/details/175177.sHtML
tv.blog.vgdrev.cn/Article/details/624048.sHtML
tv.blog.vgdrev.cn/Article/details/066666.sHtML
tv.blog.vgdrev.cn/Article/details/553951.sHtML
tv.blog.vgdrev.cn/Article/details/488406.sHtML
tv.blog.vgdrev.cn/Article/details/624440.sHtML
tv.blog.vgdrev.cn/Article/details/199175.sHtML
tv.blog.vgdrev.cn/Article/details/662406.sHtML
tv.blog.vgdrev.cn/Article/details/933371.sHtML
tv.blog.vgdrev.cn/Article/details/602266.sHtML
tv.blog.vgdrev.cn/Article/details/991175.sHtML
