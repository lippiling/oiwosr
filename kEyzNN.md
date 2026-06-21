吮室用恢嘿


Linux hwmon_device_register温度传感器与label属性

Linux hwmon子系统为硬件监控芯片提供统一的设备驱动框架，核心接口在drivers/hwmon/hwmon.c和include/linux/hwmon.h中实现。驱动通过hwmon_device_register_with_info或hwmon_device_register注册一个新的hwmon设备，通过sysfs向用户空间暴露温度、电压、风扇转速等传感器读数。

旧版API hwmon_device_register接受device指针和name字符串，创建/sys/class/hwmon/hwmonN/目录下的device和name属性文件。驱动需要自行创建每个传感器的sysfs属性文件，工作量大且容易出错。新版的hwmon_device_register_with_info通过hwmon_chip_info和hwmon_ops结构体统一描述传感器的属性集合和读写回调，框架自动创建所有属性文件。

hwmon_chip_info中的info数组指向hwmon_channel_info实例，每个实例描述一类传感器（温度、电压、风扇等），其中的config数组枚举该类型下的具体传感器配置：

```c
static const struct hwmon_channel_info *my_hwmon_info[] = {
    HWMON_CHANNEL_INFO(temp,
        HWMON_T_INPUT | HWMON_T_MAX | HWMON_T_LABEL,
        HWMON_T_INPUT | HWMON_T_LABEL
    ),
    HWMON_CHANNEL_INFO(in,
        HWMON_I_INPUT | HWMON_I_LABEL
    ),
    NULL
};
```

上面的配置声明了两个温度传感器（temp1和temp2）和一个电压传感器（in0）。HWMON_T_INPUT表示有温度读数，HWMON_T_LABEL表示有标签属性。框架根据这些位掩码自动创建对应的sysfs属性文件：temp1_input、temp1_max、temp1_label、temp2_input、temp2_label、in0_input、in0_label。

label属性用于描述传感器的物理位置或用途，例如"CPU_Die"、"GPU_Mem"、"PCH_Temp"。用户空间通过读取label来判断哪个传感器对应哪个硬件单元。驱动在is_writeable/read/write回调中处理label的读取：

```c
static int my_hwmon_read(struct device *dev, enum hwmon_sensor_types type,
                         u32 attr, int channel, long *val)
{
    struct my_sensor_data *data = dev_get_drvdata(dev);

    switch (type) {
    case hwmon_temp:
        switch (attr) {
        case hwmon_temp_input:
            *val = my_sensor_read_temp(data, channel);
            return 0;
        case hwmon_temp_max:
            *val = data->temp_max[channel];
            return 0;
        default:
            return -EOPNOTSUPP;
        }
    default:
        return -EOPNOTSUPP;
    }
}

static int my_hwmon_read_string(struct device *dev,
                                enum hwmon_sensor_types type,
                                u32 attr, int channel, const char **str)
{
    struct my_sensor_data *data = dev_get_drvdata(dev);

    switch (type) {
    case hwmon_temp:
        switch (attr) {
        case hwmon_temp_label:
            *str = data->temp_label[channel];
            return 0;
        default:
            return -EOPNOTSUPP;
        }
    default:
        return -EOPNOTSUPP;
    }
}

static const struct hwmon_ops my_hwmon_ops = {
    .is_visible = my_hwmon_visible,
    .read = my_hwmon_read,
    .read_string = my_hwmon_read_string,
    .write = my_hwmon_write,
};

static const struct hwmon_chip_info my_hwmon_chip_info = {
    .ops = &my_hwmon_ops,
    .info = my_hwmon_info,
};
```

注册时使用hwmon_device_register_with_info，传入父设备指针、芯片名称、chip_info结构和私有数据：

```c
struct device *hwmon_dev;

hwmon_dev = devm_hwmon_device_register_with_info(&pdev->dev,
                            "my_sensor",
                            data,
                            &my_hwmon_chip_info,
                            NULL);
if (IS_ERR(hwmon_dev))
    return PTR_ERR(hwmon_dev);
```

hwmon_device_register_with_info内部调用__hwmon_device_register，该函数分配hwmon_device结构体，创建sysfs属性组，最终通过device_create_with_groups将设备注册到/sys/class/hwmon/下。对于HWMON_T_LABEL这类字符串属性，框架在属性声明时将其分类为string类型，在show回调中选择调用read_string而非read。

is_visible回调是hwmon框架的重要特性，它在属性创建前由框架调用，驱动根据传感器硬件存在情况动态控制哪些属性可见：

```c
static umode_t my_hwmon_visible(const struct device *dev,
                                enum hwmon_sensor_types type,
                                u32 attr, int channel)
{
    struct my_sensor_data *data = dev_get_drvdata(dev);

    if (!data->sensor_present[channel])
        return 0;

    switch (type) {
    case hwmon_temp:
        switch (attr) {
        case hwmon_temp_input:
            return 0444;
        case hwmon_temp_label:
            return 0444;
        case hwmon_temp_max:
            if (data->has_max[channel])
                return 0644;
            return 0;
        default:
            return 0;
        }
    default:
        return 0;
    }
}
```

对于温度传感器的实际读数获取，常用的方法是通过i2c或spi总线读取芯片寄存器。以I2C温度传感器为例，驱动通过i2c_smbus_read_word_data读取16位温度寄存器值，并进行单位转换（通常以毫摄氏度为单位）：

```c
static long my_sensor_read_temp(struct my_sensor_data *data, int channel)
{
    int raw;
    long temp_mc;

    raw = i2c_smbus_read_word_data(data->client,
                                    data->temp_reg[channel]);
    if (raw < 0)
        return raw;

    /* 假设芯片输出0.125摄氏度/LSB，12位有符号数 */
    temp_mc = ((s16)raw) * 125;
    return temp_mc;
}
```

hwmon框架支持传感器类型包括：hwmon_temp（温度）、hwmon_in（电压）、hwmon_curr（电流）、hwmon_power（功率）、hwmon_energy（能量）、hwmon_fan（风扇转速）、hwmon_pwm（PWM控制）、hwmon_humidity（湿度）等。每种类型预定义了不同的事件位，例如hwmon_power支持HWMON_P_INPUT（瞬时功率）、HWMON_P_MAX（最大功率）、HWMON_P_CRIT（临界功率）等。

label属性是一个只读字符串属性，其sysfs权限为0444。与input值不同，label通常只初始化一次，在probe阶段从设备树、platform data或设备寄存器中获取。例如从设备树读取label：

```c
of_property_read_string_index(np, "label", channel, &data->temp_label[channel]);
```

devm_hwmon_device_register_with_info是推荐使用的资源管理版本，在设备移除时自动注销hwmon设备。如果驱动需要手动控制生命周期，可以使用hwmon_device_register_with_info，并配对调用hwmon_device_unregister。手动注销时，hwmon_device_unregister会删除所有sysfs属性并释放设备内存。

抛宦赫颜嘉堆颐植傅怕蚁脑胶匝冻

m.wgj.hxbsg.cn/15353.Doc
m.wgj.hxbsg.cn/02862.Doc
m.wgj.hxbsg.cn/31117.Doc
m.wgj.hxbsg.cn/64660.Doc
m.wgj.hxbsg.cn/75531.Doc
m.wgh.hxbsg.cn/44440.Doc
m.wgh.hxbsg.cn/99559.Doc
m.wgh.hxbsg.cn/11971.Doc
m.wgh.hxbsg.cn/91931.Doc
m.wgh.hxbsg.cn/15115.Doc
m.wgh.hxbsg.cn/73973.Doc
m.wgh.hxbsg.cn/71137.Doc
m.wgh.hxbsg.cn/82840.Doc
m.wgh.hxbsg.cn/04882.Doc
m.wgh.hxbsg.cn/31397.Doc
m.wgh.hxbsg.cn/99175.Doc
m.wgh.hxbsg.cn/31515.Doc
m.wgh.hxbsg.cn/95555.Doc
m.wgh.hxbsg.cn/93513.Doc
m.wgh.hxbsg.cn/59195.Doc
m.wgh.hxbsg.cn/97793.Doc
m.wgh.hxbsg.cn/28080.Doc
m.wgh.hxbsg.cn/42488.Doc
m.wgh.hxbsg.cn/48000.Doc
m.wgh.hxbsg.cn/04000.Doc
m.wgg.hxbsg.cn/46242.Doc
m.wgg.hxbsg.cn/35355.Doc
m.wgg.hxbsg.cn/46208.Doc
m.wgg.hxbsg.cn/97793.Doc
m.wgg.hxbsg.cn/33119.Doc
m.wgg.hxbsg.cn/51391.Doc
m.wgg.hxbsg.cn/02804.Doc
m.wgg.hxbsg.cn/37317.Doc
m.wgg.hxbsg.cn/33917.Doc
m.wgg.hxbsg.cn/31771.Doc
m.wgg.hxbsg.cn/11355.Doc
m.wgg.hxbsg.cn/31955.Doc
m.wgg.hxbsg.cn/88446.Doc
m.wgg.hxbsg.cn/95375.Doc
m.wgg.hxbsg.cn/80402.Doc
m.wgg.hxbsg.cn/31391.Doc
m.wgg.hxbsg.cn/35933.Doc
m.wgg.hxbsg.cn/64246.Doc
m.wgg.hxbsg.cn/26680.Doc
m.wgg.hxbsg.cn/60640.Doc
m.wgf.hxbsg.cn/60826.Doc
m.wgf.hxbsg.cn/04680.Doc
m.wgf.hxbsg.cn/86464.Doc
m.wgf.hxbsg.cn/91715.Doc
m.wgf.hxbsg.cn/75537.Doc
m.wgf.hxbsg.cn/46206.Doc
m.wgf.hxbsg.cn/80664.Doc
m.wgf.hxbsg.cn/37799.Doc
m.wgf.hxbsg.cn/95379.Doc
m.wgf.hxbsg.cn/51175.Doc
m.wgf.hxbsg.cn/08842.Doc
m.wgf.hxbsg.cn/46246.Doc
m.wgf.hxbsg.cn/37531.Doc
m.wgf.hxbsg.cn/95731.Doc
m.wgf.hxbsg.cn/39115.Doc
m.wgf.hxbsg.cn/86888.Doc
m.wgf.hxbsg.cn/06820.Doc
m.wgf.hxbsg.cn/22226.Doc
m.wgf.hxbsg.cn/04468.Doc
m.wgf.hxbsg.cn/37919.Doc
m.wgd.hxbsg.cn/33999.Doc
m.wgd.hxbsg.cn/11513.Doc
m.wgd.hxbsg.cn/04066.Doc
m.wgd.hxbsg.cn/19151.Doc
m.wgd.hxbsg.cn/24024.Doc
m.wgd.hxbsg.cn/51933.Doc
m.wgd.hxbsg.cn/60260.Doc
m.wgd.hxbsg.cn/06080.Doc
m.wgd.hxbsg.cn/82820.Doc
m.wgd.hxbsg.cn/19357.Doc
m.wgd.hxbsg.cn/20822.Doc
m.wgd.hxbsg.cn/77955.Doc
m.wgd.hxbsg.cn/77915.Doc
m.wgd.hxbsg.cn/51313.Doc
m.wgd.hxbsg.cn/64066.Doc
m.wgd.hxbsg.cn/35113.Doc
m.wgd.hxbsg.cn/31135.Doc
m.wgd.hxbsg.cn/84806.Doc
m.wgd.hxbsg.cn/68200.Doc
m.wgd.hxbsg.cn/40206.Doc
m.wgs.hxbsg.cn/42802.Doc
m.wgs.hxbsg.cn/99733.Doc
m.wgs.hxbsg.cn/15373.Doc
m.wgs.hxbsg.cn/66806.Doc
m.wgs.hxbsg.cn/91195.Doc
m.wgs.hxbsg.cn/11711.Doc
m.wgs.hxbsg.cn/59391.Doc
m.wgs.hxbsg.cn/84822.Doc
m.wgs.hxbsg.cn/17959.Doc
m.wgs.hxbsg.cn/00220.Doc
m.wgs.hxbsg.cn/13757.Doc
m.wgs.hxbsg.cn/97117.Doc
m.wgs.hxbsg.cn/04820.Doc
m.wgs.hxbsg.cn/82404.Doc
m.wgs.hxbsg.cn/93719.Doc
m.wgs.hxbsg.cn/84002.Doc
m.wgs.hxbsg.cn/02280.Doc
m.wgs.hxbsg.cn/40040.Doc
m.wgs.hxbsg.cn/86808.Doc
m.wgs.hxbsg.cn/19595.Doc
m.wga.hxbsg.cn/44222.Doc
m.wga.hxbsg.cn/71333.Doc
m.wga.hxbsg.cn/86468.Doc
m.wga.hxbsg.cn/33575.Doc
m.wga.hxbsg.cn/42080.Doc
m.wga.hxbsg.cn/71395.Doc
m.wga.hxbsg.cn/22284.Doc
m.wga.hxbsg.cn/73717.Doc
m.wga.hxbsg.cn/62864.Doc
m.wga.hxbsg.cn/39393.Doc
m.wga.hxbsg.cn/84488.Doc
m.wga.hxbsg.cn/00240.Doc
m.wga.hxbsg.cn/71559.Doc
m.wga.hxbsg.cn/51339.Doc
m.wga.hxbsg.cn/68228.Doc
m.wga.hxbsg.cn/82084.Doc
m.wga.hxbsg.cn/46004.Doc
m.wga.hxbsg.cn/68482.Doc
m.wga.hxbsg.cn/19533.Doc
m.wga.hxbsg.cn/95317.Doc
m.wgp.hxbsg.cn/02406.Doc
m.wgp.hxbsg.cn/82886.Doc
m.wgp.hxbsg.cn/53779.Doc
m.wgp.hxbsg.cn/37573.Doc
m.wgp.hxbsg.cn/13593.Doc
m.wgp.hxbsg.cn/44228.Doc
m.wgp.hxbsg.cn/37357.Doc
m.wgp.hxbsg.cn/02008.Doc
m.wgp.hxbsg.cn/66404.Doc
m.wgp.hxbsg.cn/28844.Doc
m.wgp.hxbsg.cn/97799.Doc
m.wgp.hxbsg.cn/39999.Doc
m.wgp.hxbsg.cn/86200.Doc
m.wgp.hxbsg.cn/48040.Doc
m.wgp.hxbsg.cn/40604.Doc
m.wgp.hxbsg.cn/28804.Doc
m.wgp.hxbsg.cn/73553.Doc
m.wgp.hxbsg.cn/13159.Doc
m.wgp.hxbsg.cn/95735.Doc
m.wgp.hxbsg.cn/02648.Doc
m.wgo.hxbsg.cn/71719.Doc
m.wgo.hxbsg.cn/91393.Doc
m.wgo.hxbsg.cn/64646.Doc
m.wgo.hxbsg.cn/17373.Doc
m.wgo.hxbsg.cn/42646.Doc
m.wgo.hxbsg.cn/28600.Doc
m.wgo.hxbsg.cn/11993.Doc
m.wgo.hxbsg.cn/17159.Doc
m.wgo.hxbsg.cn/62642.Doc
m.wgo.hxbsg.cn/42428.Doc
m.wgo.hxbsg.cn/93393.Doc
m.wgo.hxbsg.cn/06020.Doc
m.wgo.hxbsg.cn/71995.Doc
m.wgo.hxbsg.cn/02668.Doc
m.wgo.hxbsg.cn/35953.Doc
m.wgo.hxbsg.cn/04624.Doc
m.wgo.hxbsg.cn/86600.Doc
m.wgo.hxbsg.cn/77717.Doc
m.wgo.hxbsg.cn/80602.Doc
m.wgo.hxbsg.cn/37775.Doc
m.wgi.hxbsg.cn/46862.Doc
m.wgi.hxbsg.cn/82046.Doc
m.wgi.hxbsg.cn/15351.Doc
m.wgi.hxbsg.cn/80268.Doc
m.wgi.hxbsg.cn/37009.Doc
m.wgi.hxbsg.cn/95993.Doc
m.wgi.hxbsg.cn/40264.Doc
m.wgi.hxbsg.cn/82828.Doc
m.wgi.hxbsg.cn/99779.Doc
m.wgi.hxbsg.cn/48424.Doc
m.wgi.hxbsg.cn/15719.Doc
m.wgi.hxbsg.cn/13557.Doc
m.wgi.hxbsg.cn/79933.Doc
m.wgi.hxbsg.cn/04284.Doc
m.wgi.hxbsg.cn/39339.Doc
m.wgi.hxbsg.cn/80246.Doc
m.wgi.hxbsg.cn/02842.Doc
m.wgi.hxbsg.cn/19199.Doc
m.wgi.hxbsg.cn/26864.Doc
m.wgi.hxbsg.cn/37791.Doc
m.wgu.hxbsg.cn/37331.Doc
m.wgu.hxbsg.cn/42808.Doc
m.wgu.hxbsg.cn/11311.Doc
m.wgu.hxbsg.cn/42462.Doc
m.wgu.hxbsg.cn/62040.Doc
m.wgu.hxbsg.cn/06242.Doc
m.wgu.hxbsg.cn/84264.Doc
m.wgu.hxbsg.cn/79791.Doc
m.wgu.hxbsg.cn/71577.Doc
m.wgu.hxbsg.cn/60446.Doc
m.wgu.hxbsg.cn/79555.Doc
m.wgu.hxbsg.cn/17337.Doc
m.wgu.hxbsg.cn/17171.Doc
m.wgu.hxbsg.cn/57599.Doc
m.wgu.hxbsg.cn/28040.Doc
m.wgu.hxbsg.cn/97739.Doc
m.wgu.hxbsg.cn/84406.Doc
m.wgu.hxbsg.cn/73771.Doc
m.wgu.hxbsg.cn/22460.Doc
m.wgu.hxbsg.cn/17131.Doc
m.wgy.hxbsg.cn/62664.Doc
m.wgy.hxbsg.cn/06860.Doc
m.wgy.hxbsg.cn/77173.Doc
m.wgy.hxbsg.cn/00048.Doc
m.wgy.hxbsg.cn/46648.Doc
m.wgy.hxbsg.cn/51155.Doc
m.wgy.hxbsg.cn/99159.Doc
m.wgy.hxbsg.cn/79515.Doc
m.wgy.hxbsg.cn/88242.Doc
m.wgy.hxbsg.cn/42846.Doc
m.wgy.hxbsg.cn/04680.Doc
m.wgy.hxbsg.cn/77795.Doc
m.wgy.hxbsg.cn/88284.Doc
m.wgy.hxbsg.cn/04026.Doc
m.wgy.hxbsg.cn/80824.Doc
m.wgy.hxbsg.cn/06684.Doc
m.wgy.hxbsg.cn/91339.Doc
m.wgy.hxbsg.cn/44448.Doc
m.wgy.hxbsg.cn/86626.Doc
m.wgy.hxbsg.cn/97977.Doc
m.wgt.hxbsg.cn/51739.Doc
m.wgt.hxbsg.cn/22224.Doc
m.wgt.hxbsg.cn/99911.Doc
m.wgt.hxbsg.cn/17311.Doc
m.wgt.hxbsg.cn/44806.Doc
m.wgt.hxbsg.cn/44402.Doc
m.wgt.hxbsg.cn/37351.Doc
m.wgt.hxbsg.cn/44260.Doc
m.wgt.hxbsg.cn/73995.Doc
m.wgt.hxbsg.cn/17193.Doc
m.wgt.hxbsg.cn/51577.Doc
m.wgt.hxbsg.cn/39179.Doc
m.wgt.hxbsg.cn/48244.Doc
m.wgt.hxbsg.cn/17175.Doc
m.wgt.hxbsg.cn/22488.Doc
m.wgt.hxbsg.cn/95393.Doc
m.wgt.hxbsg.cn/66248.Doc
m.wgt.hxbsg.cn/13971.Doc
m.wgt.hxbsg.cn/51979.Doc
m.wgt.hxbsg.cn/35739.Doc
m.wgr.hxbsg.cn/93915.Doc
m.wgr.hxbsg.cn/33919.Doc
m.wgr.hxbsg.cn/91779.Doc
m.wgr.hxbsg.cn/44404.Doc
m.wgr.hxbsg.cn/08488.Doc
m.wgr.hxbsg.cn/84682.Doc
m.wgr.hxbsg.cn/11911.Doc
m.wgr.hxbsg.cn/28202.Doc
m.wgr.hxbsg.cn/88620.Doc
m.wgr.hxbsg.cn/46488.Doc
m.wgr.hxbsg.cn/31535.Doc
m.wgr.hxbsg.cn/68862.Doc
m.wgr.hxbsg.cn/59957.Doc
m.wgr.hxbsg.cn/60424.Doc
m.wgr.hxbsg.cn/37173.Doc
m.wgr.hxbsg.cn/00084.Doc
m.wgr.hxbsg.cn/88488.Doc
m.wgr.hxbsg.cn/00400.Doc
m.wgr.hxbsg.cn/06264.Doc
m.wgr.hxbsg.cn/22804.Doc
m.wge.hxbsg.cn/39735.Doc
m.wge.hxbsg.cn/64486.Doc
m.wge.hxbsg.cn/20228.Doc
m.wge.hxbsg.cn/60402.Doc
m.wge.hxbsg.cn/86826.Doc
m.wge.hxbsg.cn/99997.Doc
m.wge.hxbsg.cn/04286.Doc
m.wge.hxbsg.cn/08482.Doc
m.wge.hxbsg.cn/71339.Doc
m.wge.hxbsg.cn/42404.Doc
m.wge.hxbsg.cn/57597.Doc
m.wge.hxbsg.cn/35157.Doc
m.wge.hxbsg.cn/99513.Doc
m.wge.hxbsg.cn/40064.Doc
m.wge.hxbsg.cn/86800.Doc
m.wge.hxbsg.cn/91735.Doc
m.wge.hxbsg.cn/15577.Doc
m.wge.hxbsg.cn/04246.Doc
m.wge.hxbsg.cn/60220.Doc
m.wge.hxbsg.cn/82284.Doc
m.wgw.hxbsg.cn/88040.Doc
m.wgw.hxbsg.cn/79997.Doc
m.wgw.hxbsg.cn/02608.Doc
m.wgw.hxbsg.cn/80824.Doc
m.wgw.hxbsg.cn/15933.Doc
m.wgw.hxbsg.cn/20662.Doc
m.wgw.hxbsg.cn/75795.Doc
m.wgw.hxbsg.cn/40628.Doc
m.wgw.hxbsg.cn/02062.Doc
m.wgw.hxbsg.cn/59339.Doc
m.wgw.hxbsg.cn/17915.Doc
m.wgw.hxbsg.cn/75539.Doc
m.wgw.hxbsg.cn/37179.Doc
m.wgw.hxbsg.cn/86282.Doc
m.wgw.hxbsg.cn/17937.Doc
m.wgw.hxbsg.cn/88860.Doc
m.wgw.hxbsg.cn/71397.Doc
m.wgw.hxbsg.cn/28826.Doc
m.wgw.hxbsg.cn/86826.Doc
m.wgw.hxbsg.cn/13791.Doc
m.wgq.hxbsg.cn/95593.Doc
m.wgq.hxbsg.cn/20268.Doc
m.wgq.hxbsg.cn/31913.Doc
m.wgq.hxbsg.cn/97993.Doc
m.wgq.hxbsg.cn/20244.Doc
m.wgq.hxbsg.cn/75735.Doc
m.wgq.hxbsg.cn/91339.Doc
m.wgq.hxbsg.cn/71515.Doc
m.wgq.hxbsg.cn/24426.Doc
m.wgq.hxbsg.cn/22880.Doc
m.wgq.hxbsg.cn/77519.Doc
m.wgq.hxbsg.cn/46208.Doc
m.wgq.hxbsg.cn/99975.Doc
m.wgq.hxbsg.cn/64220.Doc
m.wgq.hxbsg.cn/20664.Doc
m.wgq.hxbsg.cn/68880.Doc
m.wgq.hxbsg.cn/13351.Doc
m.wgq.hxbsg.cn/84062.Doc
m.wgq.hxbsg.cn/79537.Doc
m.wgq.hxbsg.cn/99155.Doc
m.wfm.hxbsg.cn/60628.Doc
m.wfm.hxbsg.cn/71755.Doc
m.wfm.hxbsg.cn/17753.Doc
m.wfm.hxbsg.cn/48428.Doc
m.wfm.hxbsg.cn/93373.Doc
m.wfm.hxbsg.cn/42802.Doc
m.wfm.hxbsg.cn/19135.Doc
m.wfm.hxbsg.cn/44648.Doc
m.wfm.hxbsg.cn/86242.Doc
m.wfm.hxbsg.cn/51511.Doc
m.wfm.hxbsg.cn/20000.Doc
m.wfm.hxbsg.cn/22088.Doc
m.wfm.hxbsg.cn/88282.Doc
m.wfm.hxbsg.cn/79555.Doc
m.wfm.hxbsg.cn/46086.Doc
m.wfm.hxbsg.cn/04088.Doc
m.wfm.hxbsg.cn/80668.Doc
m.wfm.hxbsg.cn/71557.Doc
m.wfm.hxbsg.cn/26446.Doc
m.wfm.hxbsg.cn/06862.Doc
m.wfn.hxbsg.cn/71731.Doc
m.wfn.hxbsg.cn/79997.Doc
m.wfn.hxbsg.cn/48288.Doc
m.wfn.hxbsg.cn/17111.Doc
m.wfn.hxbsg.cn/19917.Doc
m.wfn.hxbsg.cn/44482.Doc
m.wfn.hxbsg.cn/51153.Doc
m.wfn.hxbsg.cn/53157.Doc
m.wfn.hxbsg.cn/46024.Doc
m.wfn.hxbsg.cn/77375.Doc
m.wfn.hxbsg.cn/77151.Doc
m.wfn.hxbsg.cn/48662.Doc
m.wfn.hxbsg.cn/40628.Doc
m.wfn.hxbsg.cn/62840.Doc
m.wfn.hxbsg.cn/17937.Doc
m.wfn.hxbsg.cn/46660.Doc
m.wfn.hxbsg.cn/17117.Doc
m.wfn.hxbsg.cn/00204.Doc
m.wfn.hxbsg.cn/60006.Doc
m.wfn.hxbsg.cn/55371.Doc
m.wfb.hxbsg.cn/19939.Doc
m.wfb.hxbsg.cn/88886.Doc
m.wfb.hxbsg.cn/86246.Doc
m.wfb.hxbsg.cn/35797.Doc
m.wfb.hxbsg.cn/95197.Doc
m.wfb.hxbsg.cn/77157.Doc
m.wfb.hxbsg.cn/22240.Doc
m.wfb.hxbsg.cn/86026.Doc
m.wfb.hxbsg.cn/22840.Doc
m.wfb.hxbsg.cn/20666.Doc
m.wfb.hxbsg.cn/82602.Doc
m.wfb.hxbsg.cn/91931.Doc
m.wfb.hxbsg.cn/48606.Doc
m.wfb.hxbsg.cn/97333.Doc
m.wfb.hxbsg.cn/80888.Doc
m.wfb.hxbsg.cn/31391.Doc
m.wfb.hxbsg.cn/48844.Doc
m.wfb.hxbsg.cn/60284.Doc
m.wfb.hxbsg.cn/80486.Doc
m.wfb.hxbsg.cn/06408.Doc
m.wfv.hxbsg.cn/06444.Doc
m.wfv.hxbsg.cn/73177.Doc
m.wfv.hxbsg.cn/95533.Doc
m.wfv.hxbsg.cn/80626.Doc
m.wfv.hxbsg.cn/64000.Doc
m.wfv.hxbsg.cn/15191.Doc
m.wfv.hxbsg.cn/20486.Doc
m.wfv.hxbsg.cn/42684.Doc
m.wfv.hxbsg.cn/08082.Doc
m.wfv.hxbsg.cn/59719.Doc
