僬锹犯弊捕


Linux thermal框架通过struct thermal_zone_device管理每个温度传感器，
governor算法根据温度变化做出冷却决策。核心注册API是
thermal_zone_device_register()，温度更新由thermal_zone_device_update()
驱动。

thermal_zone_device结构体包含温度监测所需的核心数据：

```c
struct thermal_zone_device {
    int id;
    char type[THERMAL_NAME_LENGTH];
    struct device device;
    void *devdata;
    int trips;                          // 温度阈值点数
    struct thermal_trip trips_desc[THERMAL_MAX_TRIPS]; // 阈值描述
    const struct thermal_zone_params *tzp;
    struct thermal_governor *governor;  // 当前governor
    struct list_head thermal_instances; // 冷却设备实例列表
    struct mutex lock;
    int temperature;                    // 当前温度，单位mC
    int emul_temperature;               // 仿真温度，调试用
    int passive;
    unsigned int forced_passive;
    struct thermal_zone_device_ops *ops; // 操作函数集
    enum thermal_device_mode mode;
    struct delayed_work poll_queue;      // 轮询工作队列
};
```

温度阈值（trip point）定义struct thermal_trip，包含触发温度、
类型（active/passive/hot/critical）和 hysteresis（迟滞值）：

```c
struct thermal_trip {
    struct list_head list;
    int temperature;        // 触发温度，单位mC
    int hysteresis;         // 迟滞温度
    enum thermal_trip_type type; // 阈值类型
};

enum thermal_trip_type {
    THERMAL_TRIP_ACTIVE,    // 主动冷却（风扇）
    THERMAL_TRIP_PASSIVE,   // 被动冷却（降频）
    THERMAL_TRIP_HOT,       // 高温告警
    THERMAL_TRIP_CRITICAL,  // 临界温度，触发关机
};
```

thermal_zone_device_update()是温度更新的入口函数，由轮询定时器或
中断触发。它执行governor的throttle回调：

```c
void thermal_zone_device_update(struct thermal_zone_device *tz,
                enum thermal_notify_event event)
{
    int temp, ret;
    int trip_temp;
    int trip_id;
    enum thermal_trip_type trip_type;

    // 1. 读取当前温度
    ret = tz->ops->get_temp(tz, &temp);
    if (ret) {
        dev_warn(&tz->device, "failed to read temperature\n");
        return;
    }
    tz->temperature = temp;

    // 2. 更新温度趋势
    thermal_trip_crossed(tz, temp, &trip_id, &trip_temp, &trip_type);

    // 3. 遍历所有trip point，调用governor
    for (trip_id = 0; trip_id < tz->trips; trip_id++) {
        trip_temp = tz->trips_desc[trip_id].temperature;
        trip_type = tz->trips_desc[trip_id].type;

        // 检查当前温度是否超过阈值
        if (temp >= trip_temp) {
            // 调用governor的throttle方法
            if (tz->governor->throttle)
                tz->governor->throttle(tz, trip_id);
        } else {
            // 温度回落，调用governor的control方法解除冷却
            if (tz->governor->throttle)
                tz->governor->throttle(tz, -1);
        }
    }

    // 4. 安排下次轮询
    if (tz->ops->get_temp && !tz->passive)
        mod_delayed_work(system_freezable_power_efficient_wq,
                 &tz->poll_queue,
                 msecs_to_jiffies(tz->polling_delay));
}
```

step_wise governor是最常用的governor之一，其算法逐级别调整冷却状态：

```c
static int step_wise_throttle(struct thermal_zone_device *tz, int trip)
{
    struct thermal_instance *instance;
    int trend = get_tz_trend(tz, trip);

    // 遍历所有关联此trip的冷却设备
    list_for_each_entry(instance, &tz->thermal_instances, tz_node) {
        if (instance->trip != trip)
            continue;

        // 根据温度趋势决定冷却级别增减
        switch (trend) {
        case THERMAL_TREND_RAISING:     // 温度上升
            // 提高冷却级别
            instance->target = min(instance->target + 1,
                           instance->upper);
            break;
        case THERMAL_TREND_DROPPING:    // 温度下降
            // 降低冷却级别
            instance->target = max(instance->target - 1,
                           instance->lower);
            break;
        case THERMAL_TREND_STABLE:      // 稳定
            // 维持当前级别
            break;
        default:
            break;
        }

        // 调用冷却设备的set_cur_state
        instance->cdev->ops->set_cur_state(instance->cdev,
                           instance->target);
    }
    return 0;
}
```

governor的注册和选择通过thermal_register_governor()完成，内核内置
step_wise、power_allocator、fair_share、bang_bang和user_space
五种governor。其中power_allocator使用PID控制器计算冷却需求，适用于
精细化的温度控制场景。

thermal_zone_device_unregister()释放所有资源，包括取消轮询work、
断开与所有冷却设备的绑定、释放trip点数据。governor在注销时通过
thermal_governor的unbind回调断开与thermal zone的连接。

宦砂汾叹赜祭献帽耘舅恿钡牧端腔

m.qxv.mmmfb.cn/02242.Doc
m.qxv.mmmfb.cn/64822.Doc
m.qxv.mmmfb.cn/55553.Doc
m.qxv.mmmfb.cn/42824.Doc
m.qxv.mmmfb.cn/88660.Doc
m.qxv.mmmfb.cn/20606.Doc
m.qxv.mmmfb.cn/82464.Doc
m.qxv.mmmfb.cn/08862.Doc
m.qxv.mmmfb.cn/86686.Doc
m.qxv.mmmfb.cn/19153.Doc
m.qxv.mmmfb.cn/06066.Doc
m.qxv.mmmfb.cn/48486.Doc
m.qxv.mmmfb.cn/17317.Doc
m.qxv.mmmfb.cn/42480.Doc
m.qxv.mmmfb.cn/60620.Doc
m.qxc.mmmfb.cn/06646.Doc
m.qxc.mmmfb.cn/88288.Doc
m.qxc.mmmfb.cn/22202.Doc
m.qxc.mmmfb.cn/64480.Doc
m.qxc.mmmfb.cn/60408.Doc
m.qxc.mmmfb.cn/68264.Doc
m.qxc.mmmfb.cn/22680.Doc
m.qxc.mmmfb.cn/82668.Doc
m.qxc.mmmfb.cn/68062.Doc
m.qxc.mmmfb.cn/44064.Doc
m.qxc.mmmfb.cn/93137.Doc
m.qxc.mmmfb.cn/66860.Doc
m.qxc.mmmfb.cn/48064.Doc
m.qxc.mmmfb.cn/13333.Doc
m.qxc.mmmfb.cn/28844.Doc
m.qxc.mmmfb.cn/00648.Doc
m.qxc.mmmfb.cn/71377.Doc
m.qxc.mmmfb.cn/48224.Doc
m.qxc.mmmfb.cn/02022.Doc
m.qxc.mmmfb.cn/11533.Doc
m.qxx.mmmfb.cn/22246.Doc
m.qxx.mmmfb.cn/44286.Doc
m.qxx.mmmfb.cn/40402.Doc
m.qxx.mmmfb.cn/80602.Doc
m.qxx.mmmfb.cn/97599.Doc
m.qxx.mmmfb.cn/60602.Doc
m.qxx.mmmfb.cn/42068.Doc
m.qxx.mmmfb.cn/86260.Doc
m.qxx.mmmfb.cn/24820.Doc
m.qxx.mmmfb.cn/48282.Doc
m.qxx.mmmfb.cn/15773.Doc
m.qxx.mmmfb.cn/60200.Doc
m.qxx.mmmfb.cn/26808.Doc
m.qxx.mmmfb.cn/99915.Doc
m.qxx.mmmfb.cn/82846.Doc
m.qxx.mmmfb.cn/57757.Doc
m.qxx.mmmfb.cn/20680.Doc
m.qxx.mmmfb.cn/93799.Doc
m.qxx.mmmfb.cn/73735.Doc
m.qxx.mmmfb.cn/60086.Doc
m.qxz.mmmfb.cn/59757.Doc
m.qxz.mmmfb.cn/82426.Doc
m.qxz.mmmfb.cn/28044.Doc
m.qxz.mmmfb.cn/00608.Doc
m.qxz.mmmfb.cn/64888.Doc
m.qxz.mmmfb.cn/31317.Doc
m.qxz.mmmfb.cn/37715.Doc
m.qxz.mmmfb.cn/11195.Doc
m.qxz.mmmfb.cn/13173.Doc
m.qxz.mmmfb.cn/71771.Doc
m.qxz.mmmfb.cn/17117.Doc
m.qxz.mmmfb.cn/28288.Doc
m.qxz.mmmfb.cn/88680.Doc
m.qxz.mmmfb.cn/44864.Doc
m.qxz.mmmfb.cn/93111.Doc
m.qxz.mmmfb.cn/93595.Doc
m.qxz.mmmfb.cn/99333.Doc
m.qxz.mmmfb.cn/66022.Doc
m.qxz.mmmfb.cn/62482.Doc
m.qxz.mmmfb.cn/13157.Doc
m.qxl.mmmfb.cn/88484.Doc
m.qxl.mmmfb.cn/40884.Doc
m.qxl.mmmfb.cn/80664.Doc
m.qxl.mmmfb.cn/84408.Doc
m.qxl.mmmfb.cn/48464.Doc
m.qxl.mmmfb.cn/55757.Doc
m.qxl.mmmfb.cn/93713.Doc
m.qxl.mmmfb.cn/84660.Doc
m.qxl.mmmfb.cn/86848.Doc
m.qxl.mmmfb.cn/60646.Doc
m.qxl.mmmfb.cn/97357.Doc
m.qxl.mmmfb.cn/26822.Doc
m.qxl.mmmfb.cn/02822.Doc
m.qxl.mmmfb.cn/62486.Doc
m.qxl.mmmfb.cn/00608.Doc
m.qxl.mmmfb.cn/22884.Doc
m.qxl.mmmfb.cn/71193.Doc
m.qxl.mmmfb.cn/57511.Doc
m.qxl.mmmfb.cn/08400.Doc
m.qxl.mmmfb.cn/04066.Doc
m.qxk.mmmfb.cn/17755.Doc
m.qxk.mmmfb.cn/57733.Doc
m.qxk.mmmfb.cn/84202.Doc
m.qxk.mmmfb.cn/11911.Doc
m.qxk.mmmfb.cn/04220.Doc
m.qxk.mmmfb.cn/24222.Doc
m.qxk.mmmfb.cn/20088.Doc
m.qxk.mmmfb.cn/44422.Doc
m.qxk.mmmfb.cn/22622.Doc
m.qxk.mmmfb.cn/24440.Doc
m.qxk.mmmfb.cn/88002.Doc
m.qxk.mmmfb.cn/80080.Doc
m.qxk.mmmfb.cn/40420.Doc
m.qxk.mmmfb.cn/97311.Doc
m.qxk.mmmfb.cn/46240.Doc
m.qxk.mmmfb.cn/68280.Doc
m.qxk.mmmfb.cn/68864.Doc
m.qxk.mmmfb.cn/64844.Doc
m.qxk.mmmfb.cn/99953.Doc
m.qxk.mmmfb.cn/57113.Doc
m.qxj.mmmfb.cn/28464.Doc
m.qxj.mmmfb.cn/68862.Doc
m.qxj.mmmfb.cn/40628.Doc
m.qxj.mmmfb.cn/02686.Doc
m.qxj.mmmfb.cn/22004.Doc
m.qxj.mmmfb.cn/06262.Doc
m.qxj.mmmfb.cn/42486.Doc
m.qxj.mmmfb.cn/39399.Doc
m.qxj.mmmfb.cn/97959.Doc
m.qxj.mmmfb.cn/17935.Doc
m.qxj.mmmfb.cn/46620.Doc
m.qxj.mmmfb.cn/66442.Doc
m.qxj.mmmfb.cn/28606.Doc
m.qxj.mmmfb.cn/66084.Doc
m.qxj.mmmfb.cn/95719.Doc
m.qxj.mmmfb.cn/64600.Doc
m.qxj.mmmfb.cn/80206.Doc
m.qxj.mmmfb.cn/24808.Doc
m.qxj.mmmfb.cn/84066.Doc
m.qxj.mmmfb.cn/22646.Doc
m.qxh.mmmfb.cn/80020.Doc
m.qxh.mmmfb.cn/02460.Doc
m.qxh.mmmfb.cn/06440.Doc
m.qxh.mmmfb.cn/82826.Doc
m.qxh.mmmfb.cn/64224.Doc
m.qxh.mmmfb.cn/66640.Doc
m.qxh.mmmfb.cn/86644.Doc
m.qxh.mmmfb.cn/06422.Doc
m.qxh.mmmfb.cn/44288.Doc
m.qxh.mmmfb.cn/08004.Doc
m.qxh.mmmfb.cn/99935.Doc
m.qxh.mmmfb.cn/28220.Doc
m.qxh.mmmfb.cn/22646.Doc
m.qxh.mmmfb.cn/68606.Doc
m.qxh.mmmfb.cn/35115.Doc
m.qxh.mmmfb.cn/06400.Doc
m.qxh.mmmfb.cn/08644.Doc
m.qxh.mmmfb.cn/66424.Doc
m.qxh.mmmfb.cn/35193.Doc
m.qxh.mmmfb.cn/97577.Doc
m.qxg.mmmfb.cn/08282.Doc
m.qxg.mmmfb.cn/08420.Doc
m.qxg.mmmfb.cn/86024.Doc
m.qxg.mmmfb.cn/28622.Doc
m.qxg.mmmfb.cn/48048.Doc
m.qxg.mmmfb.cn/80022.Doc
m.qxg.mmmfb.cn/64208.Doc
m.qxg.mmmfb.cn/11739.Doc
m.qxg.mmmfb.cn/68206.Doc
m.qxg.mmmfb.cn/17359.Doc
m.qxg.mmmfb.cn/68466.Doc
m.qxg.mmmfb.cn/71195.Doc
m.qxg.mmmfb.cn/37717.Doc
m.qxg.mmmfb.cn/46088.Doc
m.qxg.mmmfb.cn/46462.Doc
m.qxg.mmmfb.cn/15939.Doc
m.qxg.mmmfb.cn/17371.Doc
m.qxg.mmmfb.cn/42444.Doc
m.qxg.mmmfb.cn/20624.Doc
m.qxg.mmmfb.cn/40800.Doc
m.qxf.mmmfb.cn/42846.Doc
m.qxf.mmmfb.cn/59157.Doc
m.qxf.mmmfb.cn/66222.Doc
m.qxf.mmmfb.cn/33951.Doc
m.qxf.mmmfb.cn/15513.Doc
m.qxf.mmmfb.cn/88484.Doc
m.qxf.mmmfb.cn/93179.Doc
m.qxf.mmmfb.cn/71791.Doc
m.qxf.mmmfb.cn/40208.Doc
m.qxf.mmmfb.cn/68866.Doc
m.qxf.mmmfb.cn/80246.Doc
m.qxf.mmmfb.cn/59551.Doc
m.qxf.mmmfb.cn/48826.Doc
m.qxf.mmmfb.cn/68008.Doc
m.qxf.mmmfb.cn/08288.Doc
m.qxf.mmmfb.cn/79573.Doc
m.qxf.mmmfb.cn/24086.Doc
m.qxf.mmmfb.cn/17973.Doc
m.qxf.mmmfb.cn/39337.Doc
m.qxf.mmmfb.cn/82860.Doc
m.qxd.mmmfb.cn/22682.Doc
m.qxd.mmmfb.cn/68600.Doc
m.qxd.mmmfb.cn/26806.Doc
m.qxd.mmmfb.cn/13119.Doc
m.qxd.mmmfb.cn/55939.Doc
m.qxd.mmmfb.cn/64602.Doc
m.qxd.mmmfb.cn/46806.Doc
m.qxd.mmmfb.cn/82426.Doc
m.qxd.mmmfb.cn/04244.Doc
m.qxd.mmmfb.cn/77397.Doc
m.qxd.mmmfb.cn/55993.Doc
m.qxd.mmmfb.cn/40062.Doc
m.qxd.mmmfb.cn/20260.Doc
m.qxd.mmmfb.cn/86620.Doc
m.qxd.mmmfb.cn/00882.Doc
m.qxd.mmmfb.cn/73775.Doc
m.qxd.mmmfb.cn/80886.Doc
m.qxd.mmmfb.cn/40004.Doc
m.qxd.mmmfb.cn/13191.Doc
m.qxd.mmmfb.cn/60846.Doc
m.qxs.mmmfb.cn/17757.Doc
m.qxs.mmmfb.cn/75911.Doc
m.qxs.mmmfb.cn/20860.Doc
m.qxs.mmmfb.cn/28208.Doc
m.qxs.mmmfb.cn/02262.Doc
m.qxs.mmmfb.cn/02640.Doc
m.qxs.mmmfb.cn/64280.Doc
m.qxs.mmmfb.cn/19137.Doc
m.qxs.mmmfb.cn/82462.Doc
m.qxs.mmmfb.cn/95159.Doc
m.qxs.mmmfb.cn/04620.Doc
m.qxs.mmmfb.cn/40864.Doc
m.qxs.mmmfb.cn/64406.Doc
m.qxs.mmmfb.cn/86426.Doc
m.qxs.mmmfb.cn/48680.Doc
m.qxs.mmmfb.cn/24624.Doc
m.qxs.mmmfb.cn/02880.Doc
m.qxs.mmmfb.cn/11551.Doc
m.qxs.mmmfb.cn/44660.Doc
m.qxs.mmmfb.cn/48466.Doc
m.qxa.mmmfb.cn/64680.Doc
m.qxa.mmmfb.cn/44062.Doc
m.qxa.mmmfb.cn/22064.Doc
m.qxa.mmmfb.cn/37117.Doc
m.qxa.mmmfb.cn/44484.Doc
m.qxa.mmmfb.cn/44268.Doc
m.qxa.mmmfb.cn/60664.Doc
m.qxa.mmmfb.cn/55515.Doc
m.qxa.mmmfb.cn/22048.Doc
m.qxa.mmmfb.cn/28008.Doc
m.qxa.mmmfb.cn/86806.Doc
m.qxa.mmmfb.cn/62866.Doc
m.qxa.mmmfb.cn/08664.Doc
m.qxa.mmmfb.cn/64026.Doc
m.qxa.mmmfb.cn/75759.Doc
m.qxa.mmmfb.cn/06060.Doc
m.qxa.mmmfb.cn/00200.Doc
m.qxa.mmmfb.cn/86280.Doc
m.qxa.mmmfb.cn/08860.Doc
m.qxa.mmmfb.cn/35377.Doc
m.qxp.mmmfb.cn/99339.Doc
m.qxp.mmmfb.cn/46488.Doc
m.qxp.mmmfb.cn/28284.Doc
m.qxp.mmmfb.cn/46840.Doc
m.qxp.mmmfb.cn/02804.Doc
m.qxp.mmmfb.cn/86802.Doc
m.qxp.mmmfb.cn/84464.Doc
m.qxp.mmmfb.cn/17539.Doc
m.qxp.mmmfb.cn/44084.Doc
m.qxp.mmmfb.cn/64068.Doc
m.qxp.mmmfb.cn/44646.Doc
m.qxp.mmmfb.cn/71793.Doc
m.qxp.mmmfb.cn/06222.Doc
m.qxp.mmmfb.cn/20688.Doc
m.qxp.mmmfb.cn/26460.Doc
m.qxp.mmmfb.cn/86246.Doc
m.qxp.mmmfb.cn/33179.Doc
m.qxp.mmmfb.cn/04620.Doc
m.qxp.mmmfb.cn/22268.Doc
m.qxp.mmmfb.cn/00280.Doc
m.qxo.mmmfb.cn/40404.Doc
m.qxo.mmmfb.cn/86800.Doc
m.qxo.mmmfb.cn/57171.Doc
m.qxo.mmmfb.cn/15579.Doc
m.qxo.mmmfb.cn/66862.Doc
m.qxo.mmmfb.cn/24806.Doc
m.qxo.mmmfb.cn/39591.Doc
m.qxo.mmmfb.cn/24640.Doc
m.qxo.mmmfb.cn/22220.Doc
m.qxo.mmmfb.cn/20824.Doc
m.qxo.mmmfb.cn/82648.Doc
m.qxo.mmmfb.cn/04206.Doc
m.qxo.mmmfb.cn/84880.Doc
m.qxo.mmmfb.cn/26442.Doc
m.qxo.mmmfb.cn/06024.Doc
m.qxo.mmmfb.cn/42804.Doc
m.qxo.mmmfb.cn/04468.Doc
m.qxo.mmmfb.cn/73777.Doc
m.qxo.mmmfb.cn/19739.Doc
m.qxo.mmmfb.cn/33751.Doc
m.qxi.mmmfb.cn/04428.Doc
m.qxi.mmmfb.cn/99795.Doc
m.qxi.mmmfb.cn/06448.Doc
m.qxi.mmmfb.cn/59399.Doc
m.qxi.mmmfb.cn/97719.Doc
m.qxi.mmmfb.cn/08846.Doc
m.qxi.mmmfb.cn/88266.Doc
m.qxi.mmmfb.cn/31575.Doc
m.qxi.mmmfb.cn/46484.Doc
m.qxi.mmmfb.cn/42084.Doc
m.qxi.mmmfb.cn/82060.Doc
m.qxi.mmmfb.cn/84484.Doc
m.qxi.mmmfb.cn/97573.Doc
m.qxi.mmmfb.cn/11953.Doc
m.qxi.mmmfb.cn/75151.Doc
m.qxi.mmmfb.cn/04442.Doc
m.qxi.mmmfb.cn/19159.Doc
m.qxi.mmmfb.cn/17519.Doc
m.qxi.mmmfb.cn/46480.Doc
m.qxi.mmmfb.cn/93135.Doc
m.qxu.mmmfb.cn/71395.Doc
m.qxu.mmmfb.cn/00868.Doc
m.qxu.mmmfb.cn/46204.Doc
m.qxu.mmmfb.cn/17997.Doc
m.qxu.mmmfb.cn/59139.Doc
m.qxu.mmmfb.cn/66068.Doc
m.qxu.mmmfb.cn/20644.Doc
m.qxu.mmmfb.cn/48682.Doc
m.qxu.mmmfb.cn/24464.Doc
m.qxu.mmmfb.cn/80062.Doc
m.qxu.mmmfb.cn/82480.Doc
m.qxu.mmmfb.cn/26640.Doc
m.qxu.mmmfb.cn/40204.Doc
m.qxu.mmmfb.cn/77717.Doc
m.qxu.mmmfb.cn/42268.Doc
m.qxu.mmmfb.cn/88060.Doc
m.qxu.mmmfb.cn/24824.Doc
m.qxu.mmmfb.cn/51119.Doc
m.qxu.mmmfb.cn/19719.Doc
m.qxu.mmmfb.cn/28604.Doc
m.qxy.mmmfb.cn/64666.Doc
m.qxy.mmmfb.cn/40684.Doc
m.qxy.mmmfb.cn/64664.Doc
m.qxy.mmmfb.cn/82464.Doc
m.qxy.mmmfb.cn/31737.Doc
m.qxy.mmmfb.cn/37911.Doc
m.qxy.mmmfb.cn/00660.Doc
m.qxy.mmmfb.cn/08682.Doc
m.qxy.mmmfb.cn/33913.Doc
m.qxy.mmmfb.cn/84428.Doc
m.qxy.mmmfb.cn/86046.Doc
m.qxy.mmmfb.cn/02240.Doc
m.qxy.mmmfb.cn/40486.Doc
m.qxy.mmmfb.cn/00006.Doc
m.qxy.mmmfb.cn/04802.Doc
m.qxy.mmmfb.cn/04662.Doc
m.qxy.mmmfb.cn/68604.Doc
m.qxy.mmmfb.cn/48222.Doc
m.qxy.mmmfb.cn/02464.Doc
m.qxy.mmmfb.cn/24286.Doc
m.qxt.mmmfb.cn/37759.Doc
m.qxt.mmmfb.cn/46882.Doc
m.qxt.mmmfb.cn/64226.Doc
m.qxt.mmmfb.cn/26066.Doc
m.qxt.mmmfb.cn/68286.Doc
m.qxt.mmmfb.cn/46226.Doc
m.qxt.mmmfb.cn/84206.Doc
m.qxt.mmmfb.cn/44602.Doc
m.qxt.mmmfb.cn/44668.Doc
m.qxt.mmmfb.cn/68288.Doc
m.qxt.mmmfb.cn/08668.Doc
m.qxt.mmmfb.cn/20442.Doc
m.qxt.mmmfb.cn/79359.Doc
m.qxt.mmmfb.cn/60040.Doc
m.qxt.mmmfb.cn/26804.Doc
m.qxt.mmmfb.cn/48484.Doc
m.qxt.mmmfb.cn/84626.Doc
m.qxt.mmmfb.cn/44280.Doc
m.qxt.mmmfb.cn/00802.Doc
m.qxt.mmmfb.cn/60282.Doc
m.qxr.mmmfb.cn/60242.Doc
m.qxr.mmmfb.cn/64264.Doc
m.qxr.mmmfb.cn/60602.Doc
m.qxr.mmmfb.cn/20040.Doc
m.qxr.mmmfb.cn/35175.Doc
m.qxr.mmmfb.cn/44044.Doc
m.qxr.mmmfb.cn/68200.Doc
m.qxr.mmmfb.cn/80044.Doc
m.qxr.mmmfb.cn/60848.Doc
m.qxr.mmmfb.cn/75731.Doc
m.qxr.mmmfb.cn/62206.Doc
m.qxr.mmmfb.cn/64646.Doc
m.qxr.mmmfb.cn/04884.Doc
m.qxr.mmmfb.cn/86824.Doc
m.qxr.mmmfb.cn/02088.Doc
m.qxr.mmmfb.cn/79999.Doc
m.qxr.mmmfb.cn/48220.Doc
m.qxr.mmmfb.cn/66228.Doc
m.qxr.mmmfb.cn/48840.Doc
m.qxr.mmmfb.cn/17539.Doc
