鄙匾禄中兹


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

僭此驮蜒藏俚罢峦涤卧浪人乘噬回

m.waw.lwsnr.cn/02426.Doc
m.waw.lwsnr.cn/31191.Doc
m.waw.lwsnr.cn/39717.Doc
m.waw.lwsnr.cn/11795.Doc
m.waw.lwsnr.cn/68646.Doc
m.waw.lwsnr.cn/26802.Doc
m.waw.lwsnr.cn/75513.Doc
m.waw.lwsnr.cn/02008.Doc
m.waw.lwsnr.cn/39131.Doc
m.waw.lwsnr.cn/77515.Doc
m.waw.lwsnr.cn/51115.Doc
m.waw.lwsnr.cn/88224.Doc
m.waw.lwsnr.cn/57131.Doc
m.waw.lwsnr.cn/64426.Doc
m.waw.lwsnr.cn/73353.Doc
m.waq.lwsnr.cn/17991.Doc
m.waq.lwsnr.cn/95115.Doc
m.waq.lwsnr.cn/04662.Doc
m.waq.lwsnr.cn/91911.Doc
m.waq.lwsnr.cn/40884.Doc
m.waq.lwsnr.cn/15391.Doc
m.waq.lwsnr.cn/59517.Doc
m.waq.lwsnr.cn/86842.Doc
m.waq.lwsnr.cn/88846.Doc
m.waq.lwsnr.cn/91931.Doc
m.waq.lwsnr.cn/48462.Doc
m.waq.lwsnr.cn/57931.Doc
m.waq.lwsnr.cn/00680.Doc
m.waq.lwsnr.cn/75751.Doc
m.waq.lwsnr.cn/40824.Doc
m.waq.lwsnr.cn/84884.Doc
m.waq.lwsnr.cn/51995.Doc
m.waq.lwsnr.cn/64608.Doc
m.waq.lwsnr.cn/88400.Doc
m.waq.lwsnr.cn/11379.Doc
m.wpm.lwsnr.cn/88626.Doc
m.wpm.lwsnr.cn/44462.Doc
m.wpm.lwsnr.cn/93195.Doc
m.wpm.lwsnr.cn/08824.Doc
m.wpm.lwsnr.cn/82848.Doc
m.wpm.lwsnr.cn/15915.Doc
m.wpm.lwsnr.cn/37179.Doc
m.wpm.lwsnr.cn/57593.Doc
m.wpm.lwsnr.cn/08886.Doc
m.wpm.lwsnr.cn/15393.Doc
m.wpm.lwsnr.cn/64006.Doc
m.wpm.lwsnr.cn/51713.Doc
m.wpm.lwsnr.cn/44484.Doc
m.wpm.lwsnr.cn/60008.Doc
m.wpm.lwsnr.cn/77975.Doc
m.wpm.lwsnr.cn/62846.Doc
m.wpm.lwsnr.cn/35797.Doc
m.wpm.lwsnr.cn/31399.Doc
m.wpm.lwsnr.cn/48646.Doc
m.wpm.lwsnr.cn/75577.Doc
m.wpn.lwsnr.cn/60824.Doc
m.wpn.lwsnr.cn/39197.Doc
m.wpn.lwsnr.cn/80028.Doc
m.wpn.lwsnr.cn/59775.Doc
m.wpn.lwsnr.cn/53793.Doc
m.wpn.lwsnr.cn/64284.Doc
m.wpn.lwsnr.cn/75353.Doc
m.wpn.lwsnr.cn/20228.Doc
m.wpn.lwsnr.cn/80286.Doc
m.wpn.lwsnr.cn/42006.Doc
m.wpn.lwsnr.cn/11975.Doc
m.wpn.lwsnr.cn/08048.Doc
m.wpn.lwsnr.cn/11759.Doc
m.wpn.lwsnr.cn/17115.Doc
m.wpn.lwsnr.cn/73397.Doc
m.wpn.lwsnr.cn/44462.Doc
m.wpn.lwsnr.cn/66482.Doc
m.wpn.lwsnr.cn/91353.Doc
m.wpn.lwsnr.cn/64042.Doc
m.wpn.lwsnr.cn/31395.Doc
m.wpb.lwsnr.cn/28048.Doc
m.wpb.lwsnr.cn/40626.Doc
m.wpb.lwsnr.cn/33151.Doc
m.wpb.lwsnr.cn/40028.Doc
m.wpb.lwsnr.cn/15591.Doc
m.wpb.lwsnr.cn/28040.Doc
m.wpb.lwsnr.cn/08026.Doc
m.wpb.lwsnr.cn/91955.Doc
m.wpb.lwsnr.cn/51971.Doc
m.wpb.lwsnr.cn/60826.Doc
m.wpb.lwsnr.cn/02248.Doc
m.wpb.lwsnr.cn/73177.Doc
m.wpb.lwsnr.cn/39733.Doc
m.wpb.lwsnr.cn/00888.Doc
m.wpb.lwsnr.cn/75133.Doc
m.wpb.lwsnr.cn/13975.Doc
m.wpb.lwsnr.cn/73797.Doc
m.wpb.lwsnr.cn/79359.Doc
m.wpb.lwsnr.cn/99555.Doc
m.wpb.lwsnr.cn/02828.Doc
m.wpv.lwsnr.cn/57513.Doc
m.wpv.lwsnr.cn/97517.Doc
m.wpv.lwsnr.cn/26686.Doc
m.wpv.lwsnr.cn/60066.Doc
m.wpv.lwsnr.cn/99355.Doc
m.wpv.lwsnr.cn/24846.Doc
m.wpv.lwsnr.cn/62408.Doc
m.wpv.lwsnr.cn/13731.Doc
m.wpv.lwsnr.cn/73535.Doc
m.wpv.lwsnr.cn/28620.Doc
m.wpv.lwsnr.cn/57957.Doc
m.wpv.lwsnr.cn/37395.Doc
m.wpv.lwsnr.cn/82464.Doc
m.wpv.lwsnr.cn/77753.Doc
m.wpv.lwsnr.cn/44862.Doc
m.wpv.lwsnr.cn/97555.Doc
m.wpv.lwsnr.cn/22688.Doc
m.wpv.lwsnr.cn/68220.Doc
m.wpv.lwsnr.cn/53753.Doc
m.wpv.lwsnr.cn/88846.Doc
m.wpc.lwsnr.cn/20460.Doc
m.wpc.lwsnr.cn/55179.Doc
m.wpc.lwsnr.cn/95937.Doc
m.wpc.lwsnr.cn/26820.Doc
m.wpc.lwsnr.cn/40242.Doc
m.wpc.lwsnr.cn/35531.Doc
m.wpc.lwsnr.cn/77137.Doc
m.wpc.lwsnr.cn/46464.Doc
m.wpc.lwsnr.cn/33599.Doc
m.wpc.lwsnr.cn/31153.Doc
m.wpc.lwsnr.cn/19593.Doc
m.wpc.lwsnr.cn/60466.Doc
m.wpc.lwsnr.cn/99795.Doc
m.wpc.lwsnr.cn/64842.Doc
m.wpc.lwsnr.cn/62042.Doc
m.wpc.lwsnr.cn/99979.Doc
m.wpc.lwsnr.cn/13195.Doc
m.wpc.lwsnr.cn/79177.Doc
m.wpc.lwsnr.cn/88208.Doc
m.wpc.lwsnr.cn/19573.Doc
m.wpx.lwsnr.cn/75375.Doc
m.wpx.lwsnr.cn/48068.Doc
m.wpx.lwsnr.cn/68662.Doc
m.wpx.lwsnr.cn/99377.Doc
m.wpx.lwsnr.cn/71937.Doc
m.wpx.lwsnr.cn/91357.Doc
m.wpx.lwsnr.cn/75151.Doc
m.wpx.lwsnr.cn/55797.Doc
m.wpx.lwsnr.cn/31791.Doc
m.wpx.lwsnr.cn/68286.Doc
m.wpx.lwsnr.cn/02240.Doc
m.wpx.lwsnr.cn/04022.Doc
m.wpx.lwsnr.cn/57913.Doc
m.wpx.lwsnr.cn/08486.Doc
m.wpx.lwsnr.cn/51271.Doc
m.wpx.lwsnr.cn/77580.Doc
m.wpx.lwsnr.cn/11359.Doc
m.wpx.lwsnr.cn/20802.Doc
m.wpx.lwsnr.cn/60042.Doc
m.wpx.lwsnr.cn/73719.Doc
m.wpz.lwsnr.cn/57595.Doc
m.wpz.lwsnr.cn/66288.Doc
m.wpz.lwsnr.cn/30063.Doc
m.wpz.lwsnr.cn/39323.Doc
m.wpz.lwsnr.cn/93137.Doc
m.wpz.lwsnr.cn/66008.Doc
m.wpz.lwsnr.cn/06662.Doc
m.wpz.lwsnr.cn/37317.Doc
m.wpz.lwsnr.cn/08460.Doc
m.wpz.lwsnr.cn/11276.Doc
m.wpz.lwsnr.cn/68660.Doc
m.wpz.lwsnr.cn/79795.Doc
m.wpz.lwsnr.cn/19711.Doc
m.wpz.lwsnr.cn/86684.Doc
m.wpz.lwsnr.cn/00729.Doc
m.wpz.lwsnr.cn/37179.Doc
m.wpz.lwsnr.cn/80680.Doc
m.wpz.lwsnr.cn/00642.Doc
m.wpz.lwsnr.cn/20020.Doc
m.wpz.lwsnr.cn/94963.Doc
m.wpl.lwsnr.cn/71573.Doc
m.wpl.lwsnr.cn/51321.Doc
m.wpl.lwsnr.cn/22468.Doc
m.wpl.lwsnr.cn/31595.Doc
m.wpl.lwsnr.cn/81965.Doc
m.wpl.lwsnr.cn/02028.Doc
m.wpl.lwsnr.cn/57119.Doc
m.wpl.lwsnr.cn/91797.Doc
m.wpl.lwsnr.cn/46846.Doc
m.wpl.lwsnr.cn/19573.Doc
m.wpl.lwsnr.cn/50142.Doc
m.wpl.lwsnr.cn/26406.Doc
m.wpl.lwsnr.cn/59976.Doc
m.wpl.lwsnr.cn/79733.Doc
m.wpl.lwsnr.cn/97531.Doc
m.wpl.lwsnr.cn/86248.Doc
m.wpl.lwsnr.cn/57259.Doc
m.wpl.lwsnr.cn/46268.Doc
m.wpl.lwsnr.cn/48608.Doc
m.wpl.lwsnr.cn/19197.Doc
m.wpk.lwsnr.cn/62864.Doc
m.wpk.lwsnr.cn/74996.Doc
m.wpk.lwsnr.cn/31285.Doc
m.wpk.lwsnr.cn/05625.Doc
m.wpk.lwsnr.cn/54609.Doc
m.wpk.lwsnr.cn/88686.Doc
m.wpk.lwsnr.cn/65579.Doc
m.wpk.lwsnr.cn/48286.Doc
m.wpk.lwsnr.cn/17191.Doc
m.wpk.lwsnr.cn/04062.Doc
m.wpk.lwsnr.cn/96640.Doc
m.wpk.lwsnr.cn/71575.Doc
m.wpk.lwsnr.cn/23321.Doc
m.wpk.lwsnr.cn/21790.Doc
m.wpk.lwsnr.cn/73955.Doc
m.wpk.lwsnr.cn/68460.Doc
m.wpk.lwsnr.cn/47332.Doc
m.wpk.lwsnr.cn/39737.Doc
m.wpk.lwsnr.cn/91979.Doc
m.wpk.lwsnr.cn/82426.Doc
m.wpj.lwsnr.cn/31755.Doc
m.wpj.lwsnr.cn/60004.Doc
m.wpj.lwsnr.cn/20240.Doc
m.wpj.lwsnr.cn/79955.Doc
m.wpj.lwsnr.cn/02808.Doc
m.wpj.lwsnr.cn/40664.Doc
m.wpj.lwsnr.cn/32694.Doc
m.wpj.lwsnr.cn/17159.Doc
m.wpj.lwsnr.cn/02068.Doc
m.wpj.lwsnr.cn/15919.Doc
m.wpj.lwsnr.cn/02828.Doc
m.wpj.lwsnr.cn/80489.Doc
m.wpj.lwsnr.cn/19195.Doc
m.wpj.lwsnr.cn/93359.Doc
m.wpj.lwsnr.cn/39242.Doc
m.wpj.lwsnr.cn/75339.Doc
m.wpj.lwsnr.cn/08888.Doc
m.wpj.lwsnr.cn/47516.Doc
m.wpj.lwsnr.cn/68864.Doc
m.wpj.lwsnr.cn/95737.Doc
m.wph.lwsnr.cn/34242.Doc
m.wph.lwsnr.cn/93559.Doc
m.wph.lwsnr.cn/76633.Doc
m.wph.lwsnr.cn/62159.Doc
m.wph.lwsnr.cn/39319.Doc
m.wph.lwsnr.cn/27730.Doc
m.wph.lwsnr.cn/08307.Doc
m.wph.lwsnr.cn/91157.Doc
m.wph.lwsnr.cn/06488.Doc
m.wph.lwsnr.cn/00000.Doc
m.wph.lwsnr.cn/37335.Doc
m.wph.lwsnr.cn/46420.Doc
m.wph.lwsnr.cn/68606.Doc
m.wph.lwsnr.cn/77711.Doc
m.wph.lwsnr.cn/85071.Doc
m.wph.lwsnr.cn/02264.Doc
m.wph.lwsnr.cn/19535.Doc
m.wph.lwsnr.cn/31353.Doc
m.wph.lwsnr.cn/04462.Doc
m.wph.lwsnr.cn/19375.Doc
m.wpg.lwsnr.cn/42400.Doc
m.wpg.lwsnr.cn/87324.Doc
m.wpg.lwsnr.cn/75517.Doc
m.wpg.lwsnr.cn/95771.Doc
m.wpg.lwsnr.cn/44840.Doc
m.wpg.lwsnr.cn/57151.Doc
m.wpg.lwsnr.cn/36652.Doc
m.wpg.lwsnr.cn/84468.Doc
m.wpg.lwsnr.cn/15351.Doc
m.wpg.lwsnr.cn/26826.Doc
m.wpg.lwsnr.cn/08202.Doc
m.wpg.lwsnr.cn/11511.Doc
m.wpg.lwsnr.cn/40204.Doc
m.wpg.lwsnr.cn/17197.Doc
m.wpg.lwsnr.cn/98387.Doc
m.wpg.lwsnr.cn/22282.Doc
m.wpg.lwsnr.cn/39311.Doc
m.wpg.lwsnr.cn/26644.Doc
m.wpg.lwsnr.cn/79925.Doc
m.wpg.lwsnr.cn/57311.Doc
m.wpf.lwsnr.cn/62628.Doc
m.wpf.lwsnr.cn/20026.Doc
m.wpf.lwsnr.cn/73779.Doc
m.wpf.lwsnr.cn/64766.Doc
m.wpf.lwsnr.cn/19151.Doc
m.wpf.lwsnr.cn/11719.Doc
m.wpf.lwsnr.cn/66600.Doc
m.wpf.lwsnr.cn/82486.Doc
m.wpf.lwsnr.cn/99599.Doc
m.wpf.lwsnr.cn/65001.Doc
m.wpf.lwsnr.cn/68880.Doc
m.wpf.lwsnr.cn/22806.Doc
m.wpf.lwsnr.cn/42284.Doc
m.wpf.lwsnr.cn/08208.Doc
m.wpf.lwsnr.cn/97957.Doc
m.wpf.lwsnr.cn/79353.Doc
m.wpf.lwsnr.cn/19715.Doc
m.wpf.lwsnr.cn/79995.Doc
m.wpf.lwsnr.cn/64459.Doc
m.wpf.lwsnr.cn/90602.Doc
m.wpd.lwsnr.cn/17973.Doc
m.wpd.lwsnr.cn/26804.Doc
m.wpd.lwsnr.cn/62215.Doc
m.wpd.lwsnr.cn/35173.Doc
m.wpd.lwsnr.cn/08848.Doc
m.wpd.lwsnr.cn/33531.Doc
m.wpd.lwsnr.cn/57953.Doc
m.wpd.lwsnr.cn/84606.Doc
m.wpd.lwsnr.cn/53557.Doc
m.wpd.lwsnr.cn/75917.Doc
m.wpd.lwsnr.cn/93977.Doc
m.wpd.lwsnr.cn/86602.Doc
m.wpd.lwsnr.cn/45887.Doc
m.wpd.lwsnr.cn/19591.Doc
m.wpd.lwsnr.cn/42426.Doc
m.wpd.lwsnr.cn/02886.Doc
m.wpd.lwsnr.cn/08088.Doc
m.wpd.lwsnr.cn/15793.Doc
m.wpd.lwsnr.cn/78733.Doc
m.wpd.lwsnr.cn/80846.Doc
m.wps.lwsnr.cn/04020.Doc
m.wps.lwsnr.cn/68848.Doc
m.wps.lwsnr.cn/20264.Doc
m.wps.lwsnr.cn/17955.Doc
m.wps.lwsnr.cn/06202.Doc
m.wps.lwsnr.cn/35917.Doc
m.wps.lwsnr.cn/77311.Doc
m.wps.lwsnr.cn/12935.Doc
m.wps.lwsnr.cn/77511.Doc
m.wps.lwsnr.cn/51351.Doc
m.wps.lwsnr.cn/82044.Doc
m.wps.lwsnr.cn/99995.Doc
m.wps.lwsnr.cn/26260.Doc
m.wps.lwsnr.cn/46006.Doc
m.wps.lwsnr.cn/66825.Doc
m.wps.lwsnr.cn/97771.Doc
m.wps.lwsnr.cn/59113.Doc
m.wps.lwsnr.cn/77917.Doc
m.wps.lwsnr.cn/68604.Doc
m.wps.lwsnr.cn/40400.Doc
m.wpa.lwsnr.cn/12554.Doc
m.wpa.lwsnr.cn/86082.Doc
m.wpa.lwsnr.cn/53155.Doc
m.wpa.lwsnr.cn/97739.Doc
m.wpa.lwsnr.cn/31716.Doc
m.wpa.lwsnr.cn/15959.Doc
m.wpa.lwsnr.cn/43879.Doc
m.wpa.lwsnr.cn/95807.Doc
m.wpa.lwsnr.cn/57557.Doc
m.wpa.lwsnr.cn/92990.Doc
m.wpa.lwsnr.cn/88202.Doc
m.wpa.lwsnr.cn/57995.Doc
m.wpa.lwsnr.cn/26040.Doc
m.wpa.lwsnr.cn/28686.Doc
m.wpa.lwsnr.cn/86264.Doc
m.wpa.lwsnr.cn/06466.Doc
m.wpa.lwsnr.cn/15771.Doc
m.wpa.lwsnr.cn/93391.Doc
m.wpa.lwsnr.cn/00000.Doc
m.wpa.lwsnr.cn/99777.Doc
m.wpp.lwsnr.cn/39351.Doc
m.wpp.lwsnr.cn/64044.Doc
m.wpp.lwsnr.cn/73779.Doc
m.wpp.lwsnr.cn/57537.Doc
m.wpp.lwsnr.cn/20428.Doc
m.wpp.lwsnr.cn/00820.Doc
m.wpp.lwsnr.cn/99957.Doc
m.wpp.lwsnr.cn/96176.Doc
m.wpp.lwsnr.cn/46888.Doc
m.wpp.lwsnr.cn/44884.Doc
m.wpp.lwsnr.cn/60069.Doc
m.wpp.lwsnr.cn/86804.Doc
m.wpp.lwsnr.cn/23087.Doc
m.wpp.lwsnr.cn/60228.Doc
m.wpp.lwsnr.cn/77195.Doc
m.wpp.lwsnr.cn/64028.Doc
m.wpp.lwsnr.cn/60680.Doc
m.wpp.lwsnr.cn/28882.Doc
m.wpp.lwsnr.cn/91177.Doc
m.wpp.lwsnr.cn/22480.Doc
m.wpo.lwsnr.cn/90898.Doc
m.wpo.lwsnr.cn/71339.Doc
m.wpo.lwsnr.cn/72486.Doc
m.wpo.lwsnr.cn/06488.Doc
m.wpo.lwsnr.cn/99915.Doc
m.wpo.lwsnr.cn/88642.Doc
m.wpo.lwsnr.cn/35371.Doc
m.wpo.lwsnr.cn/64640.Doc
m.wpo.lwsnr.cn/46425.Doc
m.wpo.lwsnr.cn/59591.Doc
m.wpo.lwsnr.cn/28244.Doc
m.wpo.lwsnr.cn/13162.Doc
m.wpo.lwsnr.cn/99913.Doc
m.wpo.lwsnr.cn/48866.Doc
m.wpo.lwsnr.cn/71248.Doc
m.wpo.lwsnr.cn/93393.Doc
m.wpo.lwsnr.cn/68854.Doc
m.wpo.lwsnr.cn/5.Doc
m.wpo.lwsnr.cn/55559.Doc
m.wpo.lwsnr.cn/04402.Doc
