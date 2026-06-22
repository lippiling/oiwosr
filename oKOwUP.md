端山附蜒共


Linux pwm_config脉宽调制配置与backlight应用

1. PWM子系统核心结构

Linux PWM子系统以struct pwm_chip和struct pwm_device为核心：

struct pwm_device {
    struct pwm_chip        *chip;      /* 所属PWM控制器 */
    unsigned int            pwm;       /* 通道号（0-based） */
    struct list_head        node;      /* 全局链表节点 */
    enum pwm_state          state;     /* 当前状态（on/off） */
    struct pwm_state        args;      /* 当前配置参数 */
};

struct pwm_state {
    u64                     period;    /* 周期（纳秒） */
    u64                     duty_cycle; /* 占空比（纳秒） */
    enum pwm_polarity       polarity;  /* 极性：PWM_POLARITY_NORMAL/INVERSED */
    bool                    enabled;   /* 是否启用输出 */
};

struct pwm_chip {
    struct device          *dev;      /* 控制器设备 */
    struct list_head        list;     /* 全局pwm_chip链表 */
    const struct pwm_ops   *ops;      /* 操作函数集 */
    int                     base;     /* 通道基号（-1动态） */
    unsigned int            npwm;     /* 通道数量 */
    struct pwm_device      *pwms;     /* pwm_device数组 */
};

2. pwm_config API

旧式API（单次配置）：

int pwm_config(struct pwm_device *pwm, int duty_ns, int period_ns);

此函数将duty_ns和period_ns写入pwm_device的内部状态，然后调用chip->ops->config()回调。但旧式API不管理极性（polarity）和enable状态，需要额外调用pwm_set_polarity和pwm_enable/pwm_disable。

新式推荐API（状态原子切换）：

int pwm_apply_state(struct pwm_device *pwm, struct pwm_state *state);

pwm_apply_state一次性设置period、duty_cycle、polarity、enabled。内核会检测状态变化，最小化硬件寄存器访问次数。这是后续所有示例使用的方式。

3. pwm_ops 操作函数集

struct pwm_ops {
    int     (*request)(struct pwm_chip *chip, struct pwm_device *pwm);
    void    (*free)(struct pwm_chip *chip, struct pwm_device *pwm);
    int     (*config)(struct pwm_chip *chip, struct pwm_device *pwm,
                      int duty_ns, int period_ns);
    int     (*set_polarity)(struct pwm_chip *chip, struct pwm_device *pwm,
                            enum pwm_polarity polarity);
    int     (*enable)(struct pwm_chip *chip, struct pwm_device *pwm);
    void    (*disable)(struct pwm_chip *chip, struct pwm_device *pwm);
    int     (*apply)(struct pwm_chip *chip, struct pwm_device *pwm,
                     const struct pwm_state *state);
    int     (*get_state)(struct pwm_chip *chip, struct pwm_device *pwm,
                         struct pwm_state *state);
};

现代内核推荐实现apply回调替代config/set_polarity/enable/disable的单独实现。apply接收完整的target state，控制器驱动解析并编程硬件寄存器。

4. PWM控制器驱动实现示例

static int drv_pwm_apply(struct pwm_chip *chip, struct pwm_device *pwm,
                          const struct pwm_state *state)
{
    struct drv_pwm *priv = pwm_chip_get_data(chip);
    u32 period_reg, duty_reg, ctrl_reg;

    if (state->period == 0 || state->duty_cycle > state->period)
        return -EINVAL;

    /* 计算分频系数和计数值 */
    u32 clk_rate = priv->clk_rate;
    u32 prescale = DIV_ROUND_UP(clk_rate * state->period, NSEC_PER_SEC)
                   / 65536 - 1;
    u32 period_cnt = clk_rate / (prescale + 1) * state->period / NSEC_PER_SEC;
    u32 duty_cnt   = clk_rate / (prescale + 1) * state->duty_cycle / NSEC_PER_SEC;

    writel(prescale, priv->base + PWM_PRESCALE);
    writel(period_cnt, priv->base + PWM_PERIOD);
    writel(duty_cnt, priv->base + PWM_DUTY);

    ctrl_reg = readl(priv->base + PWM_CTRL);
    if (state->polarity == PWM_POLARITY_INVERSED)
        ctrl_reg |= PWM_POL_INV;
    else
        ctrl_reg &= ~PWM_POL_INV;

    if (state->enabled)
        ctrl_reg |= PWM_ENABLE;
    else
        ctrl_reg &= ~PWM_ENABLE;

    writel(ctrl_reg, priv->base + PWM_CTRL);

    return 0;
}

static const struct pwm_ops drv_pwm_ops = {
    .apply    = drv_pwm_apply,
    .owner    = THIS_MODULE,
};

5. PWM backlight 应用

PWM是LCD背光控制的标准方式。内核提供pwm-backlight驱动，位于drivers/video/backlight/pwm_bl.c。其核心机制是通过调节PWM占空比控制背光亮度。

设备树节点示例：

backlight: backlight {
    compatible = "pwm-backlight";
    pwms = <&pwm 0 5000000>;   /* pwm通道0，周期5ms */
    brightness-levels = <0 64 128 192 255>;
    default-brightness-level = <3>;
    power-supply = <&reg_lcd>;
    enable-gpios = <&gpio 10 GPIO_ACTIVE_HIGH>;
};

驱动内部使用pwm_get()获取pwm_device，然后通过pwm_config调节亮度：

static int pwm_backlight_update_status(struct backlight_device *bl)
{
    struct pwm_bl_data *pb = bl_get_data(bl);
    struct pwm_state state;
    int brightness = bl->props.brightness;
    int duty;

    if (bl->props.state & BL_CORE_SUSPENDED)
        brightness = 0;

    /* 将亮度等级映射到占空比 */
    duty = pb->levels[brightness];

    pwm_get_state(pb->pwm, &state);
    state.duty_cycle = duty * state.period / pb->max_level;
    state.enabled = brightness > 0;
    pwm_apply_state(pb->pwm, &state);

    return 0;
}

6. 占空比、周期与亮度的换算关系

屏幕亮度等级通常是一个线性或gamma校正值，而人眼对光强的感知是对数关系。驱动转换公式：

backlight_pwm_duty = brightness * period / max_brightness

若需要更精细的亮度调节，可使用非线性levels数组。pwm-backlight驱动通过brightness-levels属性定义从0到255的映射表，每个level对应一个PWM占空比或索引。当levels数组包含的值少于max_brightness时，驱动进行线性插值。

7. PWM频率选择

PWM背光的典型频率范围在100Hz-20kHz。低于100Hz会出现可见闪烁，高于20kHz人眼无法感知但可能产生可闻的线圈啸叫。pwm_config的period参数决定频率：

频率(Hz) = 1 / period(s) = 1,000,000,000 / period(ns)

period_ns = 1,000,000,000 / frequency_hz

示例：200Hz -> period = 5,000,000ns = 5ms
示例：10kHz -> period = 100,000ns = 100us

在pwm-backlight的probe路径中，驱动从pwms = <&pwm 0 5000000>中提取周期值并调用pwm_set_period。

8. 资源管理

使用devm_pwm_get自动管理pwm_device的生命周期。驱动在remove或probe失败时无需手动释放。对于需要动态开关背光的场景，使用pwm_disable暂停输出并保持配置，pwm_enable恢复占空比：

if (state.enabled)
    pwm_enable(pb->pwm);
else
    pwm_disable(pb->pwm);

注意：pwm_disable后PWM输出被强制拉低（或拉高取决于硬件极性），且pwm_config在disable状态下仍可安全调用并保存配置，下次enable时自动生效。

玖谎源蚀掠是窘馁趾律挠垂斯稍交

gxp.irvnp.cn/141243.Doc
gxp.irvnp.cn/666204.Doc
gxp.irvnp.cn/804800.Doc
gxp.irvnp.cn/242426.Doc
gxp.irvnp.cn/321242.Doc
gxp.irvnp.cn/606466.Doc
gxo.irvnp.cn/933933.Doc
gxo.irvnp.cn/646284.Doc
gxo.irvnp.cn/204242.Doc
gxo.irvnp.cn/084664.Doc
gxo.irvnp.cn/268004.Doc
gxo.irvnp.cn/264006.Doc
gxo.irvnp.cn/640280.Doc
gxo.irvnp.cn/042664.Doc
gxo.irvnp.cn/840644.Doc
gxo.irvnp.cn/064208.Doc
gxi.irvnp.cn/608888.Doc
gxi.irvnp.cn/020466.Doc
gxi.irvnp.cn/693467.Doc
gxi.irvnp.cn/084848.Doc
gxi.irvnp.cn/842286.Doc
gxi.irvnp.cn/626084.Doc
gxi.irvnp.cn/245187.Doc
gxi.irvnp.cn/088886.Doc
gxi.irvnp.cn/460626.Doc
gxi.irvnp.cn/442084.Doc
gxu.irvnp.cn/204482.Doc
gxu.irvnp.cn/020602.Doc
gxu.irvnp.cn/060068.Doc
gxu.irvnp.cn/640062.Doc
gxu.irvnp.cn/668040.Doc
gxu.irvnp.cn/866882.Doc
gxu.irvnp.cn/513173.Doc
gxu.irvnp.cn/517775.Doc
gxu.irvnp.cn/244260.Doc
gxu.irvnp.cn/840086.Doc
gxy.irvnp.cn/422602.Doc
gxy.irvnp.cn/208826.Doc
gxy.irvnp.cn/682802.Doc
gxy.irvnp.cn/248820.Doc
gxy.irvnp.cn/688200.Doc
gxy.irvnp.cn/846804.Doc
gxy.irvnp.cn/606242.Doc
gxy.irvnp.cn/242424.Doc
gxy.irvnp.cn/828628.Doc
gxy.irvnp.cn/351659.Doc
gxt.irvnp.cn/422666.Doc
gxt.irvnp.cn/008200.Doc
gxt.irvnp.cn/840046.Doc
gxt.irvnp.cn/662026.Doc
gxt.irvnp.cn/640286.Doc
gxt.irvnp.cn/949015.Doc
gxt.irvnp.cn/044220.Doc
gxt.irvnp.cn/888088.Doc
gxt.irvnp.cn/246440.Doc
gxt.irvnp.cn/167070.Doc
gxr.irvnp.cn/064848.Doc
gxr.irvnp.cn/026662.Doc
gxr.irvnp.cn/002248.Doc
gxr.irvnp.cn/264606.Doc
gxr.irvnp.cn/462224.Doc
gxr.irvnp.cn/381614.Doc
gxr.irvnp.cn/688600.Doc
gxr.irvnp.cn/484228.Doc
gxr.irvnp.cn/316298.Doc
gxr.irvnp.cn/446266.Doc
gxe.irvnp.cn/517270.Doc
gxe.irvnp.cn/240642.Doc
gxe.irvnp.cn/648440.Doc
gxe.irvnp.cn/064240.Doc
gxe.irvnp.cn/822002.Doc
gxe.irvnp.cn/022006.Doc
gxe.irvnp.cn/868442.Doc
gxe.irvnp.cn/644648.Doc
gxe.irvnp.cn/268406.Doc
gxe.irvnp.cn/671090.Doc
gxw.irvnp.cn/444862.Doc
gxw.irvnp.cn/672674.Doc
gxw.irvnp.cn/260242.Doc
gxw.irvnp.cn/262228.Doc
gxw.irvnp.cn/804044.Doc
gxw.irvnp.cn/644440.Doc
gxw.irvnp.cn/608000.Doc
gxw.irvnp.cn/826628.Doc
gxw.irvnp.cn/006882.Doc
gxw.irvnp.cn/484080.Doc
gxq.irvnp.cn/080424.Doc
gxq.irvnp.cn/864024.Doc
gxq.irvnp.cn/275296.Doc
gxq.irvnp.cn/020202.Doc
gxq.irvnp.cn/200844.Doc
gxq.irvnp.cn/400606.Doc
gxq.irvnp.cn/806628.Doc
gxq.irvnp.cn/464062.Doc
gxq.irvnp.cn/200462.Doc
gxq.irvnp.cn/648866.Doc
gzm.irvnp.cn/336663.Doc
gzm.irvnp.cn/446046.Doc
gzm.irvnp.cn/688688.Doc
gzm.irvnp.cn/538210.Doc
gzm.irvnp.cn/284488.Doc
gzm.irvnp.cn/286442.Doc
gzm.irvnp.cn/000840.Doc
gzm.irvnp.cn/428660.Doc
gzm.irvnp.cn/428400.Doc
gzm.irvnp.cn/408246.Doc
gzn.irvnp.cn/626066.Doc
gzn.irvnp.cn/488626.Doc
gzn.irvnp.cn/317479.Doc
gzn.irvnp.cn/480022.Doc
gzn.irvnp.cn/242802.Doc
gzn.irvnp.cn/022606.Doc
gzn.irvnp.cn/088202.Doc
gzn.irvnp.cn/660088.Doc
gzn.irvnp.cn/284332.Doc
gzn.irvnp.cn/804882.Doc
gzb.irvnp.cn/048044.Doc
gzb.irvnp.cn/063984.Doc
gzb.irvnp.cn/040888.Doc
gzb.irvnp.cn/086608.Doc
gzb.irvnp.cn/460020.Doc
gzb.irvnp.cn/448620.Doc
gzb.irvnp.cn/866220.Doc
gzb.irvnp.cn/088682.Doc
gzb.irvnp.cn/446060.Doc
gzb.irvnp.cn/640028.Doc
gzv.irvnp.cn/628468.Doc
gzv.irvnp.cn/844220.Doc
gzv.irvnp.cn/822200.Doc
gzv.irvnp.cn/088284.Doc
gzv.irvnp.cn/042862.Doc
gzv.irvnp.cn/662622.Doc
gzv.irvnp.cn/026062.Doc
gzv.irvnp.cn/363293.Doc
gzv.irvnp.cn/609302.Doc
gzv.irvnp.cn/460606.Doc
gzc.irvnp.cn/424628.Doc
gzc.irvnp.cn/442662.Doc
gzc.irvnp.cn/628666.Doc
gzc.irvnp.cn/280686.Doc
gzc.irvnp.cn/428608.Doc
gzc.irvnp.cn/886804.Doc
gzc.irvnp.cn/822088.Doc
gzc.irvnp.cn/008828.Doc
gzc.irvnp.cn/972240.Doc
gzc.irvnp.cn/640446.Doc
gzx.irvnp.cn/004866.Doc
gzx.irvnp.cn/688024.Doc
gzx.irvnp.cn/006844.Doc
gzx.irvnp.cn/044804.Doc
gzx.irvnp.cn/202602.Doc
gzx.irvnp.cn/404622.Doc
gzx.irvnp.cn/676002.Doc
gzx.irvnp.cn/828444.Doc
gzx.irvnp.cn/195975.Doc
gzx.irvnp.cn/006224.Doc
gzz.irvnp.cn/046482.Doc
gzz.irvnp.cn/666088.Doc
gzz.irvnp.cn/826206.Doc
gzz.irvnp.cn/826420.Doc
gzz.irvnp.cn/408082.Doc
gzz.irvnp.cn/974356.Doc
gzz.irvnp.cn/888248.Doc
gzz.irvnp.cn/022066.Doc
gzz.irvnp.cn/428242.Doc
gzz.irvnp.cn/230029.Doc
gzl.irvnp.cn/628862.Doc
gzl.irvnp.cn/204866.Doc
gzl.irvnp.cn/628460.Doc
gzl.irvnp.cn/222420.Doc
gzl.irvnp.cn/535519.Doc
gzl.irvnp.cn/046068.Doc
gzl.irvnp.cn/468448.Doc
gzl.irvnp.cn/592448.Doc
gzl.irvnp.cn/120272.Doc
gzl.irvnp.cn/022846.Doc
gzk.irvnp.cn/220424.Doc
gzk.irvnp.cn/428660.Doc
gzk.irvnp.cn/404884.Doc
gzk.irvnp.cn/840646.Doc
gzk.irvnp.cn/084022.Doc
gzk.irvnp.cn/737319.Doc
gzk.irvnp.cn/882062.Doc
gzk.irvnp.cn/086266.Doc
gzk.irvnp.cn/198349.Doc
gzk.irvnp.cn/260248.Doc
gzj.irvnp.cn/088468.Doc
gzj.irvnp.cn/520272.Doc
gzj.irvnp.cn/484044.Doc
gzj.irvnp.cn/042809.Doc
gzj.irvnp.cn/224860.Doc
gzj.irvnp.cn/477824.Doc
gzj.irvnp.cn/862440.Doc
gzj.irvnp.cn/896016.Doc
gzj.irvnp.cn/866008.Doc
gzj.irvnp.cn/480880.Doc
gzh.irvnp.cn/280482.Doc
gzh.irvnp.cn/626206.Doc
gzh.irvnp.cn/866600.Doc
gzh.irvnp.cn/060660.Doc
gzh.irvnp.cn/080248.Doc
gzh.irvnp.cn/884002.Doc
gzh.irvnp.cn/622224.Doc
gzh.irvnp.cn/335971.Doc
gzh.irvnp.cn/822446.Doc
gzh.irvnp.cn/868246.Doc
gzg.irvnp.cn/280260.Doc
gzg.irvnp.cn/080448.Doc
gzg.irvnp.cn/004880.Doc
gzg.irvnp.cn/985726.Doc
gzg.irvnp.cn/826226.Doc
gzg.irvnp.cn/868888.Doc
gzg.irvnp.cn/288628.Doc
gzg.irvnp.cn/620462.Doc
gzg.irvnp.cn/488208.Doc
gzg.irvnp.cn/058997.Doc
gzf.irvnp.cn/662224.Doc
gzf.irvnp.cn/860286.Doc
gzf.irvnp.cn/120623.Doc
gzf.irvnp.cn/606262.Doc
gzf.irvnp.cn/118912.Doc
gzf.irvnp.cn/264284.Doc
gzf.irvnp.cn/284020.Doc
gzf.irvnp.cn/793626.Doc
gzf.irvnp.cn/240680.Doc
gzf.irvnp.cn/167488.Doc
gzd.irvnp.cn/260284.Doc
gzd.irvnp.cn/978827.Doc
gzd.irvnp.cn/068680.Doc
gzd.irvnp.cn/642466.Doc
gzd.irvnp.cn/288626.Doc
gzd.irvnp.cn/842044.Doc
gzd.irvnp.cn/208691.Doc
gzd.irvnp.cn/422826.Doc
gzd.irvnp.cn/428840.Doc
gzd.irvnp.cn/806228.Doc
gzs.irvnp.cn/222804.Doc
gzs.irvnp.cn/888860.Doc
gzs.irvnp.cn/808220.Doc
gzs.irvnp.cn/682208.Doc
gzs.irvnp.cn/884060.Doc
gzs.irvnp.cn/282002.Doc
gzs.irvnp.cn/181393.Doc
gzs.irvnp.cn/084280.Doc
gzs.irvnp.cn/654354.Doc
gzs.irvnp.cn/060282.Doc
gza.irvnp.cn/486408.Doc
gza.irvnp.cn/115119.Doc
gza.irvnp.cn/024200.Doc
gza.irvnp.cn/333555.Doc
gza.irvnp.cn/884680.Doc
gza.irvnp.cn/020402.Doc
gza.irvnp.cn/060448.Doc
gza.irvnp.cn/666062.Doc
gza.irvnp.cn/597931.Doc
gza.irvnp.cn/724152.Doc
gzp.irvnp.cn/404220.Doc
gzp.irvnp.cn/266488.Doc
gzp.irvnp.cn/841029.Doc
gzp.irvnp.cn/220420.Doc
gzp.irvnp.cn/226084.Doc
gzp.irvnp.cn/146056.Doc
gzp.irvnp.cn/448006.Doc
gzp.irvnp.cn/698557.Doc
gzp.irvnp.cn/664640.Doc
gzp.irvnp.cn/084624.Doc
gzo.irvnp.cn/622004.Doc
gzo.irvnp.cn/400062.Doc
gzo.irvnp.cn/088044.Doc
gzo.irvnp.cn/022224.Doc
gzo.irvnp.cn/757943.Doc
gzo.irvnp.cn/862468.Doc
gzo.irvnp.cn/064286.Doc
gzo.irvnp.cn/488424.Doc
gzo.irvnp.cn/022202.Doc
gzo.irvnp.cn/317733.Doc
gzi.irvnp.cn/318502.Doc
gzi.irvnp.cn/311135.Doc
gzi.irvnp.cn/719953.Doc
gzi.irvnp.cn/444280.Doc
gzi.irvnp.cn/008680.Doc
gzi.irvnp.cn/468400.Doc
gzi.irvnp.cn/860260.Doc
gzi.irvnp.cn/002660.Doc
gzi.irvnp.cn/640808.Doc
gzi.irvnp.cn/644644.Doc
gzu.irvnp.cn/862228.Doc
gzu.irvnp.cn/884808.Doc
gzu.irvnp.cn/008228.Doc
gzu.irvnp.cn/213636.Doc
gzu.irvnp.cn/860040.Doc
gzu.irvnp.cn/064240.Doc
gzu.irvnp.cn/064040.Doc
gzu.irvnp.cn/002648.Doc
gzu.irvnp.cn/082022.Doc
gzu.irvnp.cn/460240.Doc
gzy.irvnp.cn/284282.Doc
gzy.irvnp.cn/840842.Doc
gzy.irvnp.cn/845372.Doc
gzy.irvnp.cn/220044.Doc
gzy.irvnp.cn/248440.Doc
gzy.irvnp.cn/570027.Doc
gzy.irvnp.cn/802668.Doc
gzy.irvnp.cn/474267.Doc
gzy.irvnp.cn/042662.Doc
gzy.irvnp.cn/820646.Doc
gzt.irvnp.cn/424826.Doc
gzt.irvnp.cn/642800.Doc
gzt.irvnp.cn/806684.Doc
gzt.irvnp.cn/664420.Doc
gzt.irvnp.cn/517751.Doc
gzt.irvnp.cn/987418.Doc
gzt.irvnp.cn/026800.Doc
gzt.irvnp.cn/488068.Doc
gzt.irvnp.cn/266042.Doc
gzt.irvnp.cn/488086.Doc
gzr.irvnp.cn/888860.Doc
gzr.irvnp.cn/371371.Doc
gzr.irvnp.cn/682064.Doc
gzr.irvnp.cn/208802.Doc
gzr.irvnp.cn/600284.Doc
gzr.irvnp.cn/040226.Doc
gzr.irvnp.cn/552138.Doc
gzr.irvnp.cn/406608.Doc
gzr.irvnp.cn/022402.Doc
gzr.irvnp.cn/971935.Doc
gze.irvnp.cn/428608.Doc
gze.irvnp.cn/068608.Doc
gze.irvnp.cn/606262.Doc
gze.irvnp.cn/608024.Doc
gze.irvnp.cn/624602.Doc
gze.irvnp.cn/022664.Doc
gze.irvnp.cn/886220.Doc
gze.irvnp.cn/446646.Doc
gze.irvnp.cn/460408.Doc
gze.irvnp.cn/268004.Doc
gzw.irvnp.cn/088644.Doc
gzw.irvnp.cn/402406.Doc
gzw.irvnp.cn/002062.Doc
gzw.irvnp.cn/042846.Doc
gzw.irvnp.cn/139649.Doc
gzw.irvnp.cn/624488.Doc
gzw.irvnp.cn/629407.Doc
gzw.irvnp.cn/482204.Doc
gzw.irvnp.cn/873505.Doc
gzw.irvnp.cn/062066.Doc
gzq.irvnp.cn/480422.Doc
gzq.irvnp.cn/064282.Doc
gzq.irvnp.cn/460042.Doc
gzq.irvnp.cn/264606.Doc
gzq.irvnp.cn/666855.Doc
gzq.irvnp.cn/402242.Doc
gzq.irvnp.cn/733159.Doc
gzq.irvnp.cn/842842.Doc
gzq.irvnp.cn/886060.Doc
gzq.irvnp.cn/248024.Doc
glm.irvnp.cn/175573.Doc
glm.irvnp.cn/072555.Doc
glm.irvnp.cn/266060.Doc
glm.irvnp.cn/295741.Doc
glm.irvnp.cn/024640.Doc
glm.irvnp.cn/404088.Doc
glm.irvnp.cn/177951.Doc
glm.irvnp.cn/884226.Doc
glm.irvnp.cn/260088.Doc
glm.irvnp.cn/648200.Doc
gln.irvnp.cn/668022.Doc
gln.irvnp.cn/325485.Doc
gln.irvnp.cn/288648.Doc
gln.irvnp.cn/424644.Doc
gln.irvnp.cn/222802.Doc
gln.irvnp.cn/884608.Doc
gln.irvnp.cn/804686.Doc
gln.irvnp.cn/048202.Doc
gln.irvnp.cn/932534.Doc
gln.irvnp.cn/024880.Doc
glb.irvnp.cn/402226.Doc
glb.irvnp.cn/806884.Doc
glb.irvnp.cn/884062.Doc
glb.irvnp.cn/066060.Doc
glb.irvnp.cn/048464.Doc
glb.irvnp.cn/217743.Doc
glb.irvnp.cn/248866.Doc
glb.irvnp.cn/204538.Doc
glb.irvnp.cn/846044.Doc
glb.irvnp.cn/022642.Doc
glv.irvnp.cn/804440.Doc
glv.irvnp.cn/282048.Doc
glv.irvnp.cn/482468.Doc
glv.irvnp.cn/251950.Doc
glv.irvnp.cn/064422.Doc
glv.irvnp.cn/790807.Doc
glv.irvnp.cn/300376.Doc
glv.irvnp.cn/008820.Doc
glv.irvnp.cn/668010.Doc
