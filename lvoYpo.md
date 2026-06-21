鞠寥胸卣颓


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

沿悄乃柏胶毒悍恃俚褪誓垦椒荣沽

m.qjj.kxnxh.cn/28088.Doc
m.qjj.kxnxh.cn/48840.Doc
m.qjj.kxnxh.cn/60808.Doc
m.qjj.kxnxh.cn/91991.Doc
m.qjj.kxnxh.cn/55791.Doc
m.qjj.kxnxh.cn/22806.Doc
m.qjj.kxnxh.cn/84206.Doc
m.qjj.kxnxh.cn/08646.Doc
m.qjj.kxnxh.cn/26420.Doc
m.qjj.kxnxh.cn/77557.Doc
m.qjj.kxnxh.cn/80806.Doc
m.qjj.kxnxh.cn/08480.Doc
m.qjh.kxnxh.cn/48484.Doc
m.qjh.kxnxh.cn/00208.Doc
m.qjh.kxnxh.cn/39577.Doc
m.qjh.kxnxh.cn/64240.Doc
m.qjh.kxnxh.cn/08006.Doc
m.qjh.kxnxh.cn/91951.Doc
m.qjh.kxnxh.cn/88604.Doc
m.qjh.kxnxh.cn/48648.Doc
m.qjh.kxnxh.cn/44440.Doc
m.qjh.kxnxh.cn/55191.Doc
m.qjh.kxnxh.cn/13573.Doc
m.qjh.kxnxh.cn/37353.Doc
m.qjh.kxnxh.cn/24826.Doc
m.qjh.kxnxh.cn/39713.Doc
m.qjh.kxnxh.cn/88028.Doc
m.qjh.kxnxh.cn/64246.Doc
m.qjh.kxnxh.cn/06844.Doc
m.qjh.kxnxh.cn/93335.Doc
m.qjh.kxnxh.cn/66288.Doc
m.qjh.kxnxh.cn/22268.Doc
m.qjg.kxnxh.cn/84622.Doc
m.qjg.kxnxh.cn/48422.Doc
m.qjg.kxnxh.cn/17939.Doc
m.qjg.kxnxh.cn/66000.Doc
m.qjg.kxnxh.cn/93531.Doc
m.qjg.kxnxh.cn/42804.Doc
m.qjg.kxnxh.cn/48244.Doc
m.qjg.kxnxh.cn/66424.Doc
m.qjg.kxnxh.cn/51555.Doc
m.qjg.kxnxh.cn/91735.Doc
m.qjg.kxnxh.cn/79153.Doc
m.qjg.kxnxh.cn/39959.Doc
m.qjg.kxnxh.cn/04220.Doc
m.qjg.kxnxh.cn/93197.Doc
m.qjg.kxnxh.cn/82448.Doc
m.qjg.kxnxh.cn/91575.Doc
m.qjg.kxnxh.cn/06440.Doc
m.qjg.kxnxh.cn/48848.Doc
m.qjg.kxnxh.cn/28022.Doc
m.qjg.kxnxh.cn/40086.Doc
m.qjf.kxnxh.cn/60006.Doc
m.qjf.kxnxh.cn/02062.Doc
m.qjf.kxnxh.cn/37159.Doc
m.qjf.kxnxh.cn/48226.Doc
m.qjf.kxnxh.cn/79753.Doc
m.qjf.kxnxh.cn/15353.Doc
m.qjf.kxnxh.cn/60620.Doc
m.qjf.kxnxh.cn/37195.Doc
m.qjf.kxnxh.cn/42040.Doc
m.qjf.kxnxh.cn/97335.Doc
m.qjf.kxnxh.cn/20264.Doc
m.qjf.kxnxh.cn/53379.Doc
m.qjf.kxnxh.cn/48228.Doc
m.qjf.kxnxh.cn/44266.Doc
m.qjf.kxnxh.cn/20860.Doc
m.qjf.kxnxh.cn/06864.Doc
m.qjf.kxnxh.cn/86242.Doc
m.qjf.kxnxh.cn/04266.Doc
m.qjf.kxnxh.cn/88064.Doc
m.qjf.kxnxh.cn/86888.Doc
m.qjd.kxnxh.cn/39133.Doc
m.qjd.kxnxh.cn/20024.Doc
m.qjd.kxnxh.cn/64642.Doc
m.qjd.kxnxh.cn/66866.Doc
m.qjd.kxnxh.cn/62600.Doc
m.qjd.kxnxh.cn/84622.Doc
m.qjd.kxnxh.cn/66442.Doc
m.qjd.kxnxh.cn/91597.Doc
m.qjd.kxnxh.cn/80444.Doc
m.qjd.kxnxh.cn/62624.Doc
m.qjd.kxnxh.cn/66800.Doc
m.qjd.kxnxh.cn/20008.Doc
m.qjd.kxnxh.cn/68220.Doc
m.qjd.kxnxh.cn/51197.Doc
m.qjd.kxnxh.cn/64884.Doc
m.qjd.kxnxh.cn/80280.Doc
m.qjd.kxnxh.cn/40064.Doc
m.qjd.kxnxh.cn/22068.Doc
m.qjd.kxnxh.cn/99359.Doc
m.qjd.kxnxh.cn/59119.Doc
m.qjs.kxnxh.cn/97795.Doc
m.qjs.kxnxh.cn/88404.Doc
m.qjs.kxnxh.cn/46402.Doc
m.qjs.kxnxh.cn/62682.Doc
m.qjs.kxnxh.cn/80602.Doc
m.qjs.kxnxh.cn/68220.Doc
m.qjs.kxnxh.cn/19315.Doc
m.qjs.kxnxh.cn/00222.Doc
m.qjs.kxnxh.cn/31975.Doc
m.qjs.kxnxh.cn/46242.Doc
m.qjs.kxnxh.cn/17197.Doc
m.qjs.kxnxh.cn/26408.Doc
m.qjs.kxnxh.cn/22806.Doc
m.qjs.kxnxh.cn/46242.Doc
m.qjs.kxnxh.cn/62840.Doc
m.qjs.kxnxh.cn/42484.Doc
m.qjs.kxnxh.cn/22080.Doc
m.qjs.kxnxh.cn/44260.Doc
m.qjs.kxnxh.cn/99599.Doc
m.qjs.kxnxh.cn/13957.Doc
m.qja.kxnxh.cn/62602.Doc
m.qja.kxnxh.cn/88808.Doc
m.qja.kxnxh.cn/39317.Doc
m.qja.kxnxh.cn/17555.Doc
m.qja.kxnxh.cn/68660.Doc
m.qja.kxnxh.cn/06044.Doc
m.qja.kxnxh.cn/84224.Doc
m.qja.kxnxh.cn/40860.Doc
m.qja.kxnxh.cn/15197.Doc
m.qja.kxnxh.cn/66620.Doc
m.qja.kxnxh.cn/28404.Doc
m.qja.kxnxh.cn/64448.Doc
m.qja.kxnxh.cn/55715.Doc
m.qja.kxnxh.cn/66080.Doc
m.qja.kxnxh.cn/80246.Doc
m.qja.kxnxh.cn/59195.Doc
m.qja.kxnxh.cn/97997.Doc
m.qja.kxnxh.cn/53119.Doc
m.qja.kxnxh.cn/42888.Doc
m.qja.kxnxh.cn/04864.Doc
m.qjp.kxnxh.cn/06284.Doc
m.qjp.kxnxh.cn/00864.Doc
m.qjp.kxnxh.cn/86482.Doc
m.qjp.kxnxh.cn/82864.Doc
m.qjp.kxnxh.cn/00440.Doc
m.qjp.kxnxh.cn/35159.Doc
m.qjp.kxnxh.cn/86048.Doc
m.qjp.kxnxh.cn/46820.Doc
m.qjp.kxnxh.cn/22486.Doc
m.qjp.kxnxh.cn/66426.Doc
m.qjp.kxnxh.cn/59959.Doc
m.qjp.kxnxh.cn/68662.Doc
m.qjp.kxnxh.cn/40404.Doc
m.qjp.kxnxh.cn/64808.Doc
m.qjp.kxnxh.cn/39559.Doc
m.qjp.kxnxh.cn/95915.Doc
m.qjp.kxnxh.cn/11715.Doc
m.qjp.kxnxh.cn/24640.Doc
m.qjp.kxnxh.cn/26228.Doc
m.qjp.kxnxh.cn/82482.Doc
m.qjo.kxnxh.cn/35795.Doc
m.qjo.kxnxh.cn/84608.Doc
m.qjo.kxnxh.cn/24088.Doc
m.qjo.kxnxh.cn/00660.Doc
m.qjo.kxnxh.cn/44286.Doc
m.qjo.kxnxh.cn/88482.Doc
m.qjo.kxnxh.cn/02666.Doc
m.qjo.kxnxh.cn/40242.Doc
m.qjo.kxnxh.cn/46800.Doc
m.qjo.kxnxh.cn/68280.Doc
m.qjo.kxnxh.cn/75331.Doc
m.qjo.kxnxh.cn/99115.Doc
m.qjo.kxnxh.cn/66864.Doc
m.qjo.kxnxh.cn/84082.Doc
m.qjo.kxnxh.cn/97715.Doc
m.qjo.kxnxh.cn/79913.Doc
m.qjo.kxnxh.cn/88020.Doc
m.qjo.kxnxh.cn/62084.Doc
m.qjo.kxnxh.cn/48682.Doc
m.qjo.kxnxh.cn/60242.Doc
m.qji.kxnxh.cn/60024.Doc
m.qji.kxnxh.cn/40444.Doc
m.qji.kxnxh.cn/28246.Doc
m.qji.kxnxh.cn/28808.Doc
m.qji.kxnxh.cn/44066.Doc
m.qji.kxnxh.cn/68020.Doc
m.qji.kxnxh.cn/26280.Doc
m.qji.kxnxh.cn/22262.Doc
m.qji.kxnxh.cn/77597.Doc
m.qji.kxnxh.cn/60846.Doc
m.qji.kxnxh.cn/66668.Doc
m.qji.kxnxh.cn/77719.Doc
m.qji.kxnxh.cn/20446.Doc
m.qji.kxnxh.cn/60602.Doc
m.qji.kxnxh.cn/93575.Doc
m.qji.kxnxh.cn/24486.Doc
m.qji.kxnxh.cn/42426.Doc
m.qji.kxnxh.cn/68424.Doc
m.qji.kxnxh.cn/60422.Doc
m.qji.kxnxh.cn/80248.Doc
m.qju.kxnxh.cn/04424.Doc
m.qju.kxnxh.cn/55775.Doc
m.qju.kxnxh.cn/60468.Doc
m.qju.kxnxh.cn/26480.Doc
m.qju.kxnxh.cn/86400.Doc
m.qju.kxnxh.cn/20608.Doc
m.qju.kxnxh.cn/37319.Doc
m.qju.kxnxh.cn/28842.Doc
m.qju.kxnxh.cn/60480.Doc
m.qju.kxnxh.cn/66626.Doc
m.qju.kxnxh.cn/00006.Doc
m.qju.kxnxh.cn/40068.Doc
m.qju.kxnxh.cn/02860.Doc
m.qju.kxnxh.cn/73735.Doc
m.qju.kxnxh.cn/28804.Doc
m.qju.kxnxh.cn/26864.Doc
m.qju.kxnxh.cn/60682.Doc
m.qju.kxnxh.cn/40202.Doc
m.qju.kxnxh.cn/00282.Doc
m.qju.kxnxh.cn/06288.Doc
m.qjy.kxnxh.cn/31975.Doc
m.qjy.kxnxh.cn/00646.Doc
m.qjy.kxnxh.cn/82002.Doc
m.qjy.kxnxh.cn/33791.Doc
m.qjy.kxnxh.cn/06222.Doc
m.qjy.kxnxh.cn/04422.Doc
m.qjy.kxnxh.cn/79555.Doc
m.qjy.kxnxh.cn/02464.Doc
m.qjy.kxnxh.cn/79515.Doc
m.qjy.kxnxh.cn/40644.Doc
m.qjy.kxnxh.cn/42268.Doc
m.qjy.kxnxh.cn/73111.Doc
m.qjy.kxnxh.cn/15533.Doc
m.qjy.kxnxh.cn/31775.Doc
m.qjy.kxnxh.cn/40428.Doc
m.qjy.kxnxh.cn/00228.Doc
m.qjy.kxnxh.cn/82688.Doc
m.qjy.kxnxh.cn/42642.Doc
m.qjy.kxnxh.cn/44028.Doc
m.qjy.kxnxh.cn/40648.Doc
m.qjt.kxnxh.cn/53955.Doc
m.qjt.kxnxh.cn/00088.Doc
m.qjt.kxnxh.cn/84800.Doc
m.qjt.kxnxh.cn/37391.Doc
m.qjt.kxnxh.cn/64204.Doc
m.qjt.kxnxh.cn/75975.Doc
m.qjt.kxnxh.cn/64842.Doc
m.qjt.kxnxh.cn/24206.Doc
m.qjt.kxnxh.cn/71779.Doc
m.qjt.kxnxh.cn/20022.Doc
m.qjt.kxnxh.cn/04288.Doc
m.qjt.kxnxh.cn/20268.Doc
m.qjt.kxnxh.cn/28080.Doc
m.qjt.kxnxh.cn/93357.Doc
m.qjt.kxnxh.cn/26608.Doc
m.qjt.kxnxh.cn/26064.Doc
m.qjt.kxnxh.cn/33779.Doc
m.qjt.kxnxh.cn/55913.Doc
m.qjt.kxnxh.cn/02402.Doc
m.qjt.kxnxh.cn/60886.Doc
m.qjr.kxnxh.cn/82080.Doc
m.qjr.kxnxh.cn/20224.Doc
m.qjr.kxnxh.cn/68660.Doc
m.qjr.kxnxh.cn/75937.Doc
m.qjr.kxnxh.cn/22406.Doc
m.qjr.kxnxh.cn/08866.Doc
m.qjr.kxnxh.cn/84088.Doc
m.qjr.kxnxh.cn/22228.Doc
m.qjr.kxnxh.cn/60006.Doc
m.qjr.kxnxh.cn/24680.Doc
m.qjr.kxnxh.cn/42046.Doc
m.qjr.kxnxh.cn/37337.Doc
m.qjr.kxnxh.cn/66206.Doc
m.qjr.kxnxh.cn/08082.Doc
m.qjr.kxnxh.cn/04248.Doc
m.qjr.kxnxh.cn/55573.Doc
m.qjr.kxnxh.cn/84844.Doc
m.qjr.kxnxh.cn/99377.Doc
m.qjr.kxnxh.cn/88868.Doc
m.qjr.kxnxh.cn/46680.Doc
m.qje.kxnxh.cn/02026.Doc
m.qje.kxnxh.cn/20640.Doc
m.qje.kxnxh.cn/04042.Doc
m.qje.kxnxh.cn/44862.Doc
m.qje.kxnxh.cn/20446.Doc
m.qje.kxnxh.cn/60666.Doc
m.qje.kxnxh.cn/80864.Doc
m.qje.kxnxh.cn/06442.Doc
m.qje.kxnxh.cn/64680.Doc
m.qje.kxnxh.cn/51759.Doc
m.qje.kxnxh.cn/17771.Doc
m.qje.kxnxh.cn/42220.Doc
m.qje.kxnxh.cn/82860.Doc
m.qje.kxnxh.cn/19755.Doc
m.qje.kxnxh.cn/15371.Doc
m.qje.kxnxh.cn/33155.Doc
m.qje.kxnxh.cn/35151.Doc
m.qje.kxnxh.cn/82686.Doc
m.qje.kxnxh.cn/66228.Doc
m.qje.kxnxh.cn/64602.Doc
m.qjw.kxnxh.cn/40680.Doc
m.qjw.kxnxh.cn/39191.Doc
m.qjw.kxnxh.cn/44884.Doc
m.qjw.kxnxh.cn/82648.Doc
m.qjw.kxnxh.cn/42008.Doc
m.qjw.kxnxh.cn/48680.Doc
m.qjw.kxnxh.cn/06264.Doc
m.qjw.kxnxh.cn/02844.Doc
m.qjw.kxnxh.cn/75931.Doc
m.qjw.kxnxh.cn/84222.Doc
m.qjw.kxnxh.cn/00402.Doc
m.qjw.kxnxh.cn/86202.Doc
m.qjw.kxnxh.cn/75355.Doc
m.qjw.kxnxh.cn/02004.Doc
m.qjw.kxnxh.cn/95199.Doc
m.qjw.kxnxh.cn/64648.Doc
m.qjw.kxnxh.cn/48688.Doc
m.qjw.kxnxh.cn/42424.Doc
m.qjw.kxnxh.cn/60206.Doc
m.qjw.kxnxh.cn/37537.Doc
m.qjq.kxnxh.cn/42880.Doc
m.qjq.kxnxh.cn/86824.Doc
m.qjq.kxnxh.cn/68280.Doc
m.qjq.kxnxh.cn/46446.Doc
m.qjq.kxnxh.cn/42262.Doc
m.qjq.kxnxh.cn/22666.Doc
m.qjq.kxnxh.cn/28802.Doc
m.qjq.kxnxh.cn/24062.Doc
m.qjq.kxnxh.cn/62084.Doc
m.qjq.kxnxh.cn/68682.Doc
m.qjq.kxnxh.cn/68440.Doc
m.qjq.kxnxh.cn/88842.Doc
m.qjq.kxnxh.cn/33931.Doc
m.qjq.kxnxh.cn/77993.Doc
m.qjq.kxnxh.cn/35959.Doc
m.qjq.kxnxh.cn/60044.Doc
m.qjq.kxnxh.cn/19575.Doc
m.qjq.kxnxh.cn/68086.Doc
m.qjq.kxnxh.cn/55599.Doc
m.qjq.kxnxh.cn/60028.Doc
m.qhm.kxnxh.cn/22480.Doc
m.qhm.kxnxh.cn/33997.Doc
m.qhm.kxnxh.cn/84422.Doc
m.qhm.kxnxh.cn/22226.Doc
m.qhm.kxnxh.cn/17737.Doc
m.qhm.kxnxh.cn/86066.Doc
m.qhm.kxnxh.cn/42480.Doc
m.qhm.kxnxh.cn/91597.Doc
m.qhm.kxnxh.cn/46228.Doc
m.qhm.kxnxh.cn/39955.Doc
m.qhm.kxnxh.cn/93175.Doc
m.qhm.kxnxh.cn/24682.Doc
m.qhm.kxnxh.cn/42246.Doc
m.qhm.kxnxh.cn/93759.Doc
m.qhm.kxnxh.cn/66808.Doc
m.qhm.kxnxh.cn/82282.Doc
m.qhm.kxnxh.cn/82480.Doc
m.qhm.kxnxh.cn/80066.Doc
m.qhm.kxnxh.cn/40648.Doc
m.qhm.kxnxh.cn/82246.Doc
m.qhn.kxnxh.cn/84006.Doc
m.qhn.kxnxh.cn/99337.Doc
m.qhn.kxnxh.cn/62448.Doc
m.qhn.kxnxh.cn/15759.Doc
m.qhn.kxnxh.cn/20242.Doc
m.qhn.kxnxh.cn/42442.Doc
m.qhn.kxnxh.cn/66880.Doc
m.qhn.kxnxh.cn/93397.Doc
m.qhn.kxnxh.cn/97117.Doc
m.qhn.kxnxh.cn/66266.Doc
m.qhn.kxnxh.cn/60888.Doc
m.qhn.kxnxh.cn/08664.Doc
m.qhn.kxnxh.cn/24660.Doc
m.qhn.kxnxh.cn/22006.Doc
m.qhn.kxnxh.cn/55955.Doc
m.qhn.kxnxh.cn/35955.Doc
m.qhn.kxnxh.cn/04822.Doc
m.qhn.kxnxh.cn/02848.Doc
m.qhn.kxnxh.cn/00486.Doc
m.qhn.kxnxh.cn/46008.Doc
m.qhb.kxnxh.cn/88684.Doc
m.qhb.kxnxh.cn/24868.Doc
m.qhb.kxnxh.cn/15777.Doc
m.qhb.kxnxh.cn/08206.Doc
m.qhb.kxnxh.cn/28424.Doc
m.qhb.kxnxh.cn/28800.Doc
m.qhb.kxnxh.cn/62424.Doc
m.qhb.kxnxh.cn/44686.Doc
m.qhb.kxnxh.cn/80884.Doc
m.qhb.kxnxh.cn/86242.Doc
m.qhb.kxnxh.cn/48840.Doc
m.qhb.kxnxh.cn/68220.Doc
m.qhb.kxnxh.cn/48248.Doc
m.qhb.kxnxh.cn/86846.Doc
m.qhb.kxnxh.cn/46242.Doc
m.qhb.kxnxh.cn/62864.Doc
m.qhb.kxnxh.cn/97593.Doc
m.qhb.kxnxh.cn/68280.Doc
m.qhb.kxnxh.cn/88028.Doc
m.qhb.kxnxh.cn/20486.Doc
m.qhv.kxnxh.cn/60206.Doc
m.qhv.kxnxh.cn/20462.Doc
m.qhv.kxnxh.cn/99953.Doc
