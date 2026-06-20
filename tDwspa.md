驮姓召位话


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

巫翁沽诿堂兆蹬苏夯礁跃畏撑狭掀

tv.blog.yqvyet.cn/Article/details/557755.sHtML
tv.blog.yqvyet.cn/Article/details/397575.sHtML
tv.blog.yqvyet.cn/Article/details/951731.sHtML
tv.blog.yqvyet.cn/Article/details/335135.sHtML
tv.blog.yqvyet.cn/Article/details/191355.sHtML
tv.blog.yqvyet.cn/Article/details/959715.sHtML
tv.blog.yqvyet.cn/Article/details/175975.sHtML
tv.blog.yqvyet.cn/Article/details/339795.sHtML
tv.blog.yqvyet.cn/Article/details/577975.sHtML
tv.blog.yqvyet.cn/Article/details/317135.sHtML
tv.blog.yqvyet.cn/Article/details/135113.sHtML
tv.blog.yqvyet.cn/Article/details/771719.sHtML
tv.blog.yqvyet.cn/Article/details/191993.sHtML
tv.blog.yqvyet.cn/Article/details/993393.sHtML
tv.blog.yqvyet.cn/Article/details/537935.sHtML
tv.blog.yqvyet.cn/Article/details/337591.sHtML
tv.blog.yqvyet.cn/Article/details/317137.sHtML
tv.blog.yqvyet.cn/Article/details/913399.sHtML
tv.blog.yqvyet.cn/Article/details/539115.sHtML
tv.blog.yqvyet.cn/Article/details/357577.sHtML
tv.blog.yqvyet.cn/Article/details/995779.sHtML
tv.blog.yqvyet.cn/Article/details/957577.sHtML
tv.blog.yqvyet.cn/Article/details/153971.sHtML
tv.blog.yqvyet.cn/Article/details/159957.sHtML
tv.blog.yqvyet.cn/Article/details/571337.sHtML
tv.blog.yqvyet.cn/Article/details/195559.sHtML
tv.blog.yqvyet.cn/Article/details/113173.sHtML
tv.blog.yqvyet.cn/Article/details/393171.sHtML
tv.blog.yqvyet.cn/Article/details/771735.sHtML
tv.blog.yqvyet.cn/Article/details/711735.sHtML
tv.blog.yqvyet.cn/Article/details/711919.sHtML
tv.blog.yqvyet.cn/Article/details/195157.sHtML
tv.blog.yqvyet.cn/Article/details/371155.sHtML
tv.blog.yqvyet.cn/Article/details/115531.sHtML
tv.blog.yqvyet.cn/Article/details/757397.sHtML
tv.blog.yqvyet.cn/Article/details/997337.sHtML
tv.blog.yqvyet.cn/Article/details/573593.sHtML
tv.blog.yqvyet.cn/Article/details/957531.sHtML
tv.blog.yqvyet.cn/Article/details/999591.sHtML
tv.blog.yqvyet.cn/Article/details/719339.sHtML
tv.blog.yqvyet.cn/Article/details/797799.sHtML
tv.blog.yqvyet.cn/Article/details/531759.sHtML
tv.blog.yqvyet.cn/Article/details/995971.sHtML
tv.blog.yqvyet.cn/Article/details/915535.sHtML
tv.blog.yqvyet.cn/Article/details/315319.sHtML
tv.blog.yqvyet.cn/Article/details/935757.sHtML
tv.blog.yqvyet.cn/Article/details/575793.sHtML
tv.blog.yqvyet.cn/Article/details/993331.sHtML
tv.blog.yqvyet.cn/Article/details/537591.sHtML
tv.blog.yqvyet.cn/Article/details/511979.sHtML
tv.blog.yqvyet.cn/Article/details/339779.sHtML
tv.blog.yqvyet.cn/Article/details/519755.sHtML
tv.blog.yqvyet.cn/Article/details/937573.sHtML
tv.blog.yqvyet.cn/Article/details/553353.sHtML
tv.blog.yqvyet.cn/Article/details/735917.sHtML
tv.blog.yqvyet.cn/Article/details/773377.sHtML
tv.blog.yqvyet.cn/Article/details/979931.sHtML
tv.blog.yqvyet.cn/Article/details/993771.sHtML
tv.blog.yqvyet.cn/Article/details/575371.sHtML
tv.blog.yqvyet.cn/Article/details/711919.sHtML
tv.blog.yqvyet.cn/Article/details/975393.sHtML
tv.blog.yqvyet.cn/Article/details/771539.sHtML
tv.blog.yqvyet.cn/Article/details/117711.sHtML
tv.blog.yqvyet.cn/Article/details/397791.sHtML
tv.blog.yqvyet.cn/Article/details/993995.sHtML
tv.blog.yqvyet.cn/Article/details/773751.sHtML
tv.blog.yqvyet.cn/Article/details/339375.sHtML
tv.blog.yqvyet.cn/Article/details/795359.sHtML
tv.blog.yqvyet.cn/Article/details/711579.sHtML
tv.blog.yqvyet.cn/Article/details/553937.sHtML
tv.blog.yqvyet.cn/Article/details/977779.sHtML
tv.blog.yqvyet.cn/Article/details/331953.sHtML
tv.blog.yqvyet.cn/Article/details/173713.sHtML
tv.blog.yqvyet.cn/Article/details/371119.sHtML
tv.blog.yqvyet.cn/Article/details/317953.sHtML
tv.blog.yqvyet.cn/Article/details/973333.sHtML
tv.blog.yqvyet.cn/Article/details/137791.sHtML
tv.blog.yqvyet.cn/Article/details/913979.sHtML
tv.blog.yqvyet.cn/Article/details/197397.sHtML
tv.blog.yqvyet.cn/Article/details/959911.sHtML
tv.blog.yqvyet.cn/Article/details/519577.sHtML
tv.blog.yqvyet.cn/Article/details/111751.sHtML
tv.blog.yqvyet.cn/Article/details/977755.sHtML
tv.blog.yqvyet.cn/Article/details/913317.sHtML
tv.blog.yqvyet.cn/Article/details/933353.sHtML
tv.blog.yqvyet.cn/Article/details/953357.sHtML
tv.blog.yqvyet.cn/Article/details/751355.sHtML
tv.blog.yqvyet.cn/Article/details/777735.sHtML
tv.blog.yqvyet.cn/Article/details/159993.sHtML
tv.blog.yqvyet.cn/Article/details/751153.sHtML
tv.blog.yqvyet.cn/Article/details/371175.sHtML
tv.blog.yqvyet.cn/Article/details/371571.sHtML
tv.blog.yqvyet.cn/Article/details/315793.sHtML
tv.blog.yqvyet.cn/Article/details/799395.sHtML
tv.blog.yqvyet.cn/Article/details/379931.sHtML
tv.blog.yqvyet.cn/Article/details/975515.sHtML
tv.blog.yqvyet.cn/Article/details/599531.sHtML
tv.blog.yqvyet.cn/Article/details/133517.sHtML
tv.blog.yqvyet.cn/Article/details/191195.sHtML
tv.blog.yqvyet.cn/Article/details/911711.sHtML
tv.blog.yqvyet.cn/Article/details/331913.sHtML
tv.blog.yqvyet.cn/Article/details/971715.sHtML
tv.blog.yqvyet.cn/Article/details/931739.sHtML
tv.blog.yqvyet.cn/Article/details/573711.sHtML
tv.blog.yqvyet.cn/Article/details/593553.sHtML
tv.blog.yqvyet.cn/Article/details/937979.sHtML
tv.blog.yqvyet.cn/Article/details/999719.sHtML
tv.blog.yqvyet.cn/Article/details/999131.sHtML
tv.blog.yqvyet.cn/Article/details/735119.sHtML
tv.blog.yqvyet.cn/Article/details/395993.sHtML
tv.blog.yqvyet.cn/Article/details/539955.sHtML
tv.blog.yqvyet.cn/Article/details/355195.sHtML
tv.blog.yqvyet.cn/Article/details/997137.sHtML
tv.blog.yqvyet.cn/Article/details/119599.sHtML
tv.blog.yqvyet.cn/Article/details/133797.sHtML
tv.blog.yqvyet.cn/Article/details/139599.sHtML
tv.blog.yqvyet.cn/Article/details/959937.sHtML
tv.blog.yqvyet.cn/Article/details/571357.sHtML
tv.blog.yqvyet.cn/Article/details/197319.sHtML
tv.blog.yqvyet.cn/Article/details/733379.sHtML
tv.blog.yqvyet.cn/Article/details/735517.sHtML
tv.blog.yqvyet.cn/Article/details/557771.sHtML
tv.blog.yqvyet.cn/Article/details/571719.sHtML
tv.blog.yqvyet.cn/Article/details/399357.sHtML
tv.blog.yqvyet.cn/Article/details/991113.sHtML
tv.blog.yqvyet.cn/Article/details/179399.sHtML
tv.blog.yqvyet.cn/Article/details/951373.sHtML
tv.blog.yqvyet.cn/Article/details/151511.sHtML
tv.blog.yqvyet.cn/Article/details/931557.sHtML
tv.blog.yqvyet.cn/Article/details/557173.sHtML
tv.blog.yqvyet.cn/Article/details/519397.sHtML
tv.blog.yqvyet.cn/Article/details/151177.sHtML
tv.blog.yqvyet.cn/Article/details/597333.sHtML
tv.blog.yqvyet.cn/Article/details/199315.sHtML
tv.blog.yqvyet.cn/Article/details/179591.sHtML
tv.blog.yqvyet.cn/Article/details/977935.sHtML
tv.blog.yqvyet.cn/Article/details/915315.sHtML
tv.blog.yqvyet.cn/Article/details/913131.sHtML
tv.blog.yqvyet.cn/Article/details/195515.sHtML
tv.blog.yqvyet.cn/Article/details/557753.sHtML
tv.blog.yqvyet.cn/Article/details/919339.sHtML
tv.blog.yqvyet.cn/Article/details/377975.sHtML
tv.blog.yqvyet.cn/Article/details/771753.sHtML
tv.blog.yqvyet.cn/Article/details/533313.sHtML
tv.blog.yqvyet.cn/Article/details/337593.sHtML
tv.blog.yqvyet.cn/Article/details/155917.sHtML
tv.blog.yqvyet.cn/Article/details/133751.sHtML
tv.blog.yqvyet.cn/Article/details/713313.sHtML
tv.blog.yqvyet.cn/Article/details/515331.sHtML
tv.blog.yqvyet.cn/Article/details/933539.sHtML
tv.blog.yqvyet.cn/Article/details/179917.sHtML
tv.blog.yqvyet.cn/Article/details/973799.sHtML
tv.blog.yqvyet.cn/Article/details/913793.sHtML
tv.blog.yqvyet.cn/Article/details/757179.sHtML
tv.blog.yqvyet.cn/Article/details/393397.sHtML
tv.blog.yqvyet.cn/Article/details/371315.sHtML
tv.blog.yqvyet.cn/Article/details/511915.sHtML
tv.blog.yqvyet.cn/Article/details/931377.sHtML
tv.blog.yqvyet.cn/Article/details/577373.sHtML
tv.blog.yqvyet.cn/Article/details/115171.sHtML
tv.blog.yqvyet.cn/Article/details/511351.sHtML
tv.blog.yqvyet.cn/Article/details/975937.sHtML
tv.blog.yqvyet.cn/Article/details/759937.sHtML
tv.blog.yqvyet.cn/Article/details/113513.sHtML
tv.blog.yqvyet.cn/Article/details/177773.sHtML
tv.blog.yqvyet.cn/Article/details/977977.sHtML
tv.blog.yqvyet.cn/Article/details/979539.sHtML
tv.blog.yqvyet.cn/Article/details/737539.sHtML
tv.blog.yqvyet.cn/Article/details/793917.sHtML
tv.blog.yqvyet.cn/Article/details/771197.sHtML
tv.blog.yqvyet.cn/Article/details/731375.sHtML
tv.blog.yqvyet.cn/Article/details/159975.sHtML
tv.blog.yqvyet.cn/Article/details/175375.sHtML
tv.blog.yqvyet.cn/Article/details/153919.sHtML
tv.blog.yqvyet.cn/Article/details/339777.sHtML
tv.blog.yqvyet.cn/Article/details/555359.sHtML
tv.blog.yqvyet.cn/Article/details/591991.sHtML
tv.blog.yqvyet.cn/Article/details/995537.sHtML
tv.blog.yqvyet.cn/Article/details/111155.sHtML
tv.blog.yqvyet.cn/Article/details/333359.sHtML
tv.blog.yqvyet.cn/Article/details/513993.sHtML
tv.blog.yqvyet.cn/Article/details/975991.sHtML
tv.blog.yqvyet.cn/Article/details/937335.sHtML
tv.blog.yqvyet.cn/Article/details/179377.sHtML
tv.blog.yqvyet.cn/Article/details/971577.sHtML
tv.blog.yqvyet.cn/Article/details/577975.sHtML
tv.blog.yqvyet.cn/Article/details/531115.sHtML
tv.blog.yqvyet.cn/Article/details/991191.sHtML
tv.blog.yqvyet.cn/Article/details/911531.sHtML
tv.blog.yqvyet.cn/Article/details/377195.sHtML
tv.blog.yqvyet.cn/Article/details/751537.sHtML
tv.blog.yqvyet.cn/Article/details/391751.sHtML
tv.blog.yqvyet.cn/Article/details/753713.sHtML
tv.blog.yqvyet.cn/Article/details/959357.sHtML
tv.blog.yqvyet.cn/Article/details/593595.sHtML
tv.blog.yqvyet.cn/Article/details/537995.sHtML
tv.blog.yqvyet.cn/Article/details/935375.sHtML
tv.blog.yqvyet.cn/Article/details/739751.sHtML
tv.blog.yqvyet.cn/Article/details/351111.sHtML
tv.blog.yqvyet.cn/Article/details/157393.sHtML
tv.blog.yqvyet.cn/Article/details/315513.sHtML
tv.blog.yqvyet.cn/Article/details/591111.sHtML
tv.blog.yqvyet.cn/Article/details/755997.sHtML
tv.blog.yqvyet.cn/Article/details/731353.sHtML
tv.blog.yqvyet.cn/Article/details/135757.sHtML
tv.blog.yqvyet.cn/Article/details/351975.sHtML
tv.blog.yqvyet.cn/Article/details/951795.sHtML
tv.blog.yqvyet.cn/Article/details/379591.sHtML
tv.blog.yqvyet.cn/Article/details/779351.sHtML
tv.blog.yqvyet.cn/Article/details/551979.sHtML
tv.blog.yqvyet.cn/Article/details/339199.sHtML
tv.blog.yqvyet.cn/Article/details/173931.sHtML
tv.blog.yqvyet.cn/Article/details/719713.sHtML
tv.blog.yqvyet.cn/Article/details/175755.sHtML
tv.blog.yqvyet.cn/Article/details/133179.sHtML
tv.blog.yqvyet.cn/Article/details/355151.sHtML
tv.blog.yqvyet.cn/Article/details/115153.sHtML
tv.blog.yqvyet.cn/Article/details/357977.sHtML
tv.blog.yqvyet.cn/Article/details/557597.sHtML
tv.blog.yqvyet.cn/Article/details/917951.sHtML
tv.blog.yqvyet.cn/Article/details/799139.sHtML
tv.blog.yqvyet.cn/Article/details/937779.sHtML
tv.blog.yqvyet.cn/Article/details/315713.sHtML
tv.blog.yqvyet.cn/Article/details/959797.sHtML
tv.blog.yqvyet.cn/Article/details/517933.sHtML
tv.blog.yqvyet.cn/Article/details/793511.sHtML
tv.blog.yqvyet.cn/Article/details/315331.sHtML
tv.blog.yqvyet.cn/Article/details/171755.sHtML
tv.blog.yqvyet.cn/Article/details/577513.sHtML
tv.blog.yqvyet.cn/Article/details/395991.sHtML
tv.blog.yqvyet.cn/Article/details/555155.sHtML
tv.blog.yqvyet.cn/Article/details/773779.sHtML
tv.blog.yqvyet.cn/Article/details/391111.sHtML
tv.blog.yqvyet.cn/Article/details/911917.sHtML
tv.blog.yqvyet.cn/Article/details/113131.sHtML
tv.blog.yqvyet.cn/Article/details/331951.sHtML
tv.blog.yqvyet.cn/Article/details/373551.sHtML
tv.blog.yqvyet.cn/Article/details/197757.sHtML
tv.blog.yqvyet.cn/Article/details/117595.sHtML
tv.blog.yqvyet.cn/Article/details/379377.sHtML
tv.blog.yqvyet.cn/Article/details/777771.sHtML
tv.blog.yqvyet.cn/Article/details/113359.sHtML
tv.blog.yqvyet.cn/Article/details/111755.sHtML
tv.blog.yqvyet.cn/Article/details/773357.sHtML
tv.blog.yqvyet.cn/Article/details/559533.sHtML
tv.blog.yqvyet.cn/Article/details/971155.sHtML
tv.blog.yqvyet.cn/Article/details/993397.sHtML
tv.blog.yqvyet.cn/Article/details/933971.sHtML
tv.blog.yqvyet.cn/Article/details/775739.sHtML
tv.blog.yqvyet.cn/Article/details/337957.sHtML
tv.blog.yqvyet.cn/Article/details/599311.sHtML
tv.blog.yqvyet.cn/Article/details/171135.sHtML
tv.blog.yqvyet.cn/Article/details/557317.sHtML
tv.blog.yqvyet.cn/Article/details/337595.sHtML
tv.blog.yqvyet.cn/Article/details/313955.sHtML
tv.blog.yqvyet.cn/Article/details/735957.sHtML
tv.blog.yqvyet.cn/Article/details/571959.sHtML
tv.blog.yqvyet.cn/Article/details/975173.sHtML
tv.blog.yqvyet.cn/Article/details/579375.sHtML
tv.blog.yqvyet.cn/Article/details/777575.sHtML
tv.blog.yqvyet.cn/Article/details/717175.sHtML
tv.blog.yqvyet.cn/Article/details/573393.sHtML
tv.blog.yqvyet.cn/Article/details/577335.sHtML
tv.blog.yqvyet.cn/Article/details/117397.sHtML
tv.blog.yqvyet.cn/Article/details/597355.sHtML
tv.blog.yqvyet.cn/Article/details/915559.sHtML
tv.blog.yqvyet.cn/Article/details/335157.sHtML
tv.blog.yqvyet.cn/Article/details/311557.sHtML
tv.blog.yqvyet.cn/Article/details/155151.sHtML
tv.blog.yqvyet.cn/Article/details/715797.sHtML
tv.blog.yqvyet.cn/Article/details/751555.sHtML
tv.blog.yqvyet.cn/Article/details/399115.sHtML
tv.blog.yqvyet.cn/Article/details/571777.sHtML
tv.blog.yqvyet.cn/Article/details/551317.sHtML
tv.blog.yqvyet.cn/Article/details/917913.sHtML
tv.blog.yqvyet.cn/Article/details/777115.sHtML
tv.blog.yqvyet.cn/Article/details/199795.sHtML
tv.blog.yqvyet.cn/Article/details/797799.sHtML
tv.blog.yqvyet.cn/Article/details/317759.sHtML
tv.blog.yqvyet.cn/Article/details/551991.sHtML
tv.blog.yqvyet.cn/Article/details/393593.sHtML
tv.blog.yqvyet.cn/Article/details/799599.sHtML
tv.blog.yqvyet.cn/Article/details/315771.sHtML
tv.blog.yqvyet.cn/Article/details/971735.sHtML
tv.blog.yqvyet.cn/Article/details/537759.sHtML
tv.blog.yqvyet.cn/Article/details/199757.sHtML
tv.blog.yqvyet.cn/Article/details/373731.sHtML
tv.blog.yqvyet.cn/Article/details/715795.sHtML
tv.blog.yqvyet.cn/Article/details/599593.sHtML
tv.blog.yqvyet.cn/Article/details/115575.sHtML
tv.blog.yqvyet.cn/Article/details/119777.sHtML
tv.blog.yqvyet.cn/Article/details/715557.sHtML
tv.blog.yqvyet.cn/Article/details/113173.sHtML
tv.blog.yqvyet.cn/Article/details/797555.sHtML
tv.blog.yqvyet.cn/Article/details/191155.sHtML
tv.blog.yqvyet.cn/Article/details/131733.sHtML
tv.blog.yqvyet.cn/Article/details/937979.sHtML
tv.blog.yqvyet.cn/Article/details/319735.sHtML
tv.blog.yqvyet.cn/Article/details/919975.sHtML
tv.blog.yqvyet.cn/Article/details/159393.sHtML
tv.blog.yqvyet.cn/Article/details/713995.sHtML
tv.blog.yqvyet.cn/Article/details/133955.sHtML
tv.blog.yqvyet.cn/Article/details/195393.sHtML
tv.blog.yqvyet.cn/Article/details/539551.sHtML
tv.blog.yqvyet.cn/Article/details/579111.sHtML
tv.blog.yqvyet.cn/Article/details/153535.sHtML
tv.blog.yqvyet.cn/Article/details/335771.sHtML
tv.blog.yqvyet.cn/Article/details/733357.sHtML
tv.blog.yqvyet.cn/Article/details/719797.sHtML
tv.blog.yqvyet.cn/Article/details/171559.sHtML
tv.blog.yqvyet.cn/Article/details/591357.sHtML
tv.blog.yqvyet.cn/Article/details/939777.sHtML
tv.blog.yqvyet.cn/Article/details/337315.sHtML
tv.blog.yqvyet.cn/Article/details/795153.sHtML
tv.blog.yqvyet.cn/Article/details/797577.sHtML
tv.blog.yqvyet.cn/Article/details/711333.sHtML
tv.blog.yqvyet.cn/Article/details/131311.sHtML
tv.blog.yqvyet.cn/Article/details/595739.sHtML
tv.blog.yqvyet.cn/Article/details/331117.sHtML
tv.blog.yqvyet.cn/Article/details/313117.sHtML
tv.blog.yqvyet.cn/Article/details/191151.sHtML
tv.blog.yqvyet.cn/Article/details/519715.sHtML
tv.blog.yqvyet.cn/Article/details/517959.sHtML
tv.blog.yqvyet.cn/Article/details/751395.sHtML
tv.blog.yqvyet.cn/Article/details/137771.sHtML
tv.blog.yqvyet.cn/Article/details/357175.sHtML
tv.blog.yqvyet.cn/Article/details/597335.sHtML
tv.blog.yqvyet.cn/Article/details/979959.sHtML
tv.blog.yqvyet.cn/Article/details/757517.sHtML
tv.blog.yqvyet.cn/Article/details/737531.sHtML
tv.blog.yqvyet.cn/Article/details/193993.sHtML
tv.blog.yqvyet.cn/Article/details/113537.sHtML
tv.blog.yqvyet.cn/Article/details/379117.sHtML
tv.blog.yqvyet.cn/Article/details/751111.sHtML
tv.blog.yqvyet.cn/Article/details/737513.sHtML
tv.blog.yqvyet.cn/Article/details/715737.sHtML
tv.blog.yqvyet.cn/Article/details/371753.sHtML
tv.blog.yqvyet.cn/Article/details/133773.sHtML
tv.blog.yqvyet.cn/Article/details/371319.sHtML
tv.blog.yqvyet.cn/Article/details/113999.sHtML
tv.blog.yqvyet.cn/Article/details/111397.sHtML
tv.blog.yqvyet.cn/Article/details/577515.sHtML
tv.blog.yqvyet.cn/Article/details/917933.sHtML
tv.blog.yqvyet.cn/Article/details/197511.sHtML
tv.blog.yqvyet.cn/Article/details/179197.sHtML
tv.blog.yqvyet.cn/Article/details/557791.sHtML
tv.blog.yqvyet.cn/Article/details/197771.sHtML
tv.blog.yqvyet.cn/Article/details/791139.sHtML
tv.blog.yqvyet.cn/Article/details/591535.sHtML
tv.blog.yqvyet.cn/Article/details/531995.sHtML
tv.blog.yqvyet.cn/Article/details/913917.sHtML
tv.blog.yqvyet.cn/Article/details/337771.sHtML
tv.blog.yqvyet.cn/Article/details/775779.sHtML
tv.blog.yqvyet.cn/Article/details/531731.sHtML
tv.blog.yqvyet.cn/Article/details/197375.sHtML
tv.blog.yqvyet.cn/Article/details/393533.sHtML
tv.blog.yqvyet.cn/Article/details/337719.sHtML
tv.blog.yqvyet.cn/Article/details/333573.sHtML
tv.blog.yqvyet.cn/Article/details/991713.sHtML
tv.blog.yqvyet.cn/Article/details/339799.sHtML
tv.blog.yqvyet.cn/Article/details/751511.sHtML
tv.blog.yqvyet.cn/Article/details/171115.sHtML
tv.blog.yqvyet.cn/Article/details/197711.sHtML
tv.blog.yqvyet.cn/Article/details/353191.sHtML
tv.blog.yqvyet.cn/Article/details/197315.sHtML
tv.blog.yqvyet.cn/Article/details/739359.sHtML
tv.blog.yqvyet.cn/Article/details/971719.sHtML
tv.blog.yqvyet.cn/Article/details/595335.sHtML
tv.blog.yqvyet.cn/Article/details/191919.sHtML
tv.blog.yqvyet.cn/Article/details/959315.sHtML
tv.blog.yqvyet.cn/Article/details/177733.sHtML
tv.blog.yqvyet.cn/Article/details/573393.sHtML
tv.blog.yqvyet.cn/Article/details/759915.sHtML
tv.blog.yqvyet.cn/Article/details/973597.sHtML
tv.blog.yqvyet.cn/Article/details/339515.sHtML
tv.blog.yqvyet.cn/Article/details/577117.sHtML
tv.blog.yqvyet.cn/Article/details/777335.sHtML
tv.blog.yqvyet.cn/Article/details/515131.sHtML
tv.blog.yqvyet.cn/Article/details/575313.sHtML
tv.blog.yqvyet.cn/Article/details/551331.sHtML
tv.blog.yqvyet.cn/Article/details/977959.sHtML
tv.blog.yqvyet.cn/Article/details/979753.sHtML
tv.blog.yqvyet.cn/Article/details/995977.sHtML
tv.blog.yqvyet.cn/Article/details/559997.sHtML
tv.blog.yqvyet.cn/Article/details/579511.sHtML
tv.blog.yqvyet.cn/Article/details/151551.sHtML
tv.blog.yqvyet.cn/Article/details/391773.sHtML
tv.blog.yqvyet.cn/Article/details/511397.sHtML
tv.blog.yqvyet.cn/Article/details/931771.sHtML
tv.blog.yqvyet.cn/Article/details/157555.sHtML
tv.blog.yqvyet.cn/Article/details/795995.sHtML
tv.blog.yqvyet.cn/Article/details/319591.sHtML
tv.blog.yqvyet.cn/Article/details/359937.sHtML
tv.blog.yqvyet.cn/Article/details/911119.sHtML
tv.blog.yqvyet.cn/Article/details/137971.sHtML
