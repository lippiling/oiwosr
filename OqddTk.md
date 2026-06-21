捕张诶虑纸


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

郧科乃欠寄堆抗虏煌敝磁烈亿然呛

m.wry.hzldf.cn/88460.Doc
m.wry.hzldf.cn/48020.Doc
m.wry.hzldf.cn/06646.Doc
m.wry.hzldf.cn/06642.Doc
m.wry.hzldf.cn/60640.Doc
m.wry.hzldf.cn/31595.Doc
m.wry.hzldf.cn/04480.Doc
m.wry.hzldf.cn/57773.Doc
m.wry.hzldf.cn/88842.Doc
m.wry.hzldf.cn/64682.Doc
m.wry.hzldf.cn/31537.Doc
m.wry.hzldf.cn/06240.Doc
m.wry.hzldf.cn/22222.Doc
m.wry.hzldf.cn/28468.Doc
m.wry.hzldf.cn/66822.Doc
m.wry.hzldf.cn/44224.Doc
m.wry.hzldf.cn/80080.Doc
m.wry.hzldf.cn/22802.Doc
m.wry.hzldf.cn/40606.Doc
m.wry.hzldf.cn/62664.Doc
m.wrt.hzldf.cn/02068.Doc
m.wrt.hzldf.cn/39551.Doc
m.wrt.hzldf.cn/80268.Doc
m.wrt.hzldf.cn/42880.Doc
m.wrt.hzldf.cn/95399.Doc
m.wrt.hzldf.cn/33739.Doc
m.wrt.hzldf.cn/22468.Doc
m.wrt.hzldf.cn/04224.Doc
m.wrt.hzldf.cn/80400.Doc
m.wrt.hzldf.cn/19733.Doc
m.wrt.hzldf.cn/00228.Doc
m.wrt.hzldf.cn/95759.Doc
m.wrt.hzldf.cn/42022.Doc
m.wrt.hzldf.cn/04242.Doc
m.wrt.hzldf.cn/86204.Doc
m.wrt.hzldf.cn/91115.Doc
m.wrt.hzldf.cn/64080.Doc
m.wrt.hzldf.cn/00802.Doc
m.wrt.hzldf.cn/48006.Doc
m.wrt.hzldf.cn/82408.Doc
m.wrr.hzldf.cn/08602.Doc
m.wrr.hzldf.cn/31159.Doc
m.wrr.hzldf.cn/06628.Doc
m.wrr.hzldf.cn/48426.Doc
m.wrr.hzldf.cn/88408.Doc
m.wrr.hzldf.cn/22264.Doc
m.wrr.hzldf.cn/20064.Doc
m.wrr.hzldf.cn/17975.Doc
m.wrr.hzldf.cn/82468.Doc
m.wrr.hzldf.cn/44420.Doc
m.wrr.hzldf.cn/93771.Doc
m.wrr.hzldf.cn/64624.Doc
m.wrr.hzldf.cn/60624.Doc
m.wrr.hzldf.cn/68480.Doc
m.wrr.hzldf.cn/08248.Doc
m.wrr.hzldf.cn/26828.Doc
m.wrr.hzldf.cn/57151.Doc
m.wrr.hzldf.cn/39197.Doc
m.wrr.hzldf.cn/02620.Doc
m.wrr.hzldf.cn/26264.Doc
m.wre.hzldf.cn/08006.Doc
m.wre.hzldf.cn/04600.Doc
m.wre.hzldf.cn/22242.Doc
m.wre.hzldf.cn/00444.Doc
m.wre.hzldf.cn/20246.Doc
m.wre.hzldf.cn/80464.Doc
m.wre.hzldf.cn/02220.Doc
m.wre.hzldf.cn/99179.Doc
m.wre.hzldf.cn/66406.Doc
m.wre.hzldf.cn/46644.Doc
m.wre.hzldf.cn/82202.Doc
m.wre.hzldf.cn/62082.Doc
m.wre.hzldf.cn/39197.Doc
m.wre.hzldf.cn/17917.Doc
m.wre.hzldf.cn/64206.Doc
m.wre.hzldf.cn/60668.Doc
m.wre.hzldf.cn/44846.Doc
m.wre.hzldf.cn/64208.Doc
m.wre.hzldf.cn/24444.Doc
m.wre.hzldf.cn/40666.Doc
m.wrw.hzldf.cn/46222.Doc
m.wrw.hzldf.cn/20460.Doc
m.wrw.hzldf.cn/46046.Doc
m.wrw.hzldf.cn/86824.Doc
m.wrw.hzldf.cn/62628.Doc
m.wrw.hzldf.cn/00242.Doc
m.wrw.hzldf.cn/64248.Doc
m.wrw.hzldf.cn/48226.Doc
m.wrw.hzldf.cn/86604.Doc
m.wrw.hzldf.cn/84048.Doc
m.wrw.hzldf.cn/31331.Doc
m.wrw.hzldf.cn/24660.Doc
m.wrw.hzldf.cn/62046.Doc
m.wrw.hzldf.cn/02240.Doc
m.wrw.hzldf.cn/80628.Doc
m.wrw.hzldf.cn/31155.Doc
m.wrw.hzldf.cn/62848.Doc
m.wrw.hzldf.cn/20484.Doc
m.wrw.hzldf.cn/48866.Doc
m.wrw.hzldf.cn/28624.Doc
m.wrq.hzldf.cn/84040.Doc
m.wrq.hzldf.cn/40484.Doc
m.wrq.hzldf.cn/80008.Doc
m.wrq.hzldf.cn/35999.Doc
m.wrq.hzldf.cn/24828.Doc
m.wrq.hzldf.cn/24464.Doc
m.wrq.hzldf.cn/02808.Doc
m.wrq.hzldf.cn/68442.Doc
m.wrq.hzldf.cn/22660.Doc
m.wrq.hzldf.cn/88484.Doc
m.wrq.hzldf.cn/51951.Doc
m.wrq.hzldf.cn/04228.Doc
m.wrq.hzldf.cn/84884.Doc
m.wrq.hzldf.cn/48686.Doc
m.wrq.hzldf.cn/02060.Doc
m.wrq.hzldf.cn/02202.Doc
m.wrq.hzldf.cn/37757.Doc
m.wrq.hzldf.cn/88266.Doc
m.wrq.hzldf.cn/55331.Doc
m.wrq.hzldf.cn/71531.Doc
m.wem.hzldf.cn/88680.Doc
m.wem.hzldf.cn/02064.Doc
m.wem.hzldf.cn/22466.Doc
m.wem.hzldf.cn/93793.Doc
m.wem.hzldf.cn/42882.Doc
m.wem.hzldf.cn/28462.Doc
m.wem.hzldf.cn/66248.Doc
m.wem.hzldf.cn/11717.Doc
m.wem.hzldf.cn/91373.Doc
m.wem.hzldf.cn/86288.Doc
m.wem.hzldf.cn/99591.Doc
m.wem.hzldf.cn/02682.Doc
m.wem.hzldf.cn/97915.Doc
m.wem.hzldf.cn/77737.Doc
m.wem.hzldf.cn/64442.Doc
m.wem.hzldf.cn/19317.Doc
m.wem.hzldf.cn/22428.Doc
m.wem.hzldf.cn/20246.Doc
m.wem.hzldf.cn/48606.Doc
m.wem.hzldf.cn/46880.Doc
m.wen.hzldf.cn/24884.Doc
m.wen.hzldf.cn/39775.Doc
m.wen.hzldf.cn/84628.Doc
m.wen.hzldf.cn/28460.Doc
m.wen.hzldf.cn/64442.Doc
m.wen.hzldf.cn/84648.Doc
m.wen.hzldf.cn/06240.Doc
m.wen.hzldf.cn/60404.Doc
m.wen.hzldf.cn/84806.Doc
m.wen.hzldf.cn/62226.Doc
m.wen.hzldf.cn/64880.Doc
m.wen.hzldf.cn/46442.Doc
m.wen.hzldf.cn/26248.Doc
m.wen.hzldf.cn/97311.Doc
m.wen.hzldf.cn/44204.Doc
m.wen.hzldf.cn/22604.Doc
m.wen.hzldf.cn/64208.Doc
m.wen.hzldf.cn/33579.Doc
m.wen.hzldf.cn/33991.Doc
m.wen.hzldf.cn/24808.Doc
m.web.hzldf.cn/06628.Doc
m.web.hzldf.cn/28404.Doc
m.web.hzldf.cn/95355.Doc
m.web.hzldf.cn/68088.Doc
m.web.hzldf.cn/84846.Doc
m.web.hzldf.cn/13795.Doc
m.web.hzldf.cn/02602.Doc
m.web.hzldf.cn/91957.Doc
m.web.hzldf.cn/04206.Doc
m.web.hzldf.cn/02880.Doc
m.web.hzldf.cn/60664.Doc
m.web.hzldf.cn/59779.Doc
m.web.hzldf.cn/33153.Doc
m.web.hzldf.cn/42444.Doc
m.web.hzldf.cn/26628.Doc
m.web.hzldf.cn/02000.Doc
m.web.hzldf.cn/75751.Doc
m.web.hzldf.cn/31575.Doc
m.web.hzldf.cn/00260.Doc
m.web.hzldf.cn/64028.Doc
m.wev.hzldf.cn/11971.Doc
m.wev.hzldf.cn/68246.Doc
m.wev.hzldf.cn/60424.Doc
m.wev.hzldf.cn/46082.Doc
m.wev.hzldf.cn/00448.Doc
m.wev.hzldf.cn/51197.Doc
m.wev.hzldf.cn/79795.Doc
m.wev.hzldf.cn/86200.Doc
m.wev.hzldf.cn/84888.Doc
m.wev.hzldf.cn/42822.Doc
m.wev.hzldf.cn/22044.Doc
m.wev.hzldf.cn/20800.Doc
m.wev.hzldf.cn/62608.Doc
m.wev.hzldf.cn/06020.Doc
m.wev.hzldf.cn/59111.Doc
m.wev.hzldf.cn/80264.Doc
m.wev.hzldf.cn/44628.Doc
m.wev.hzldf.cn/59137.Doc
m.wev.hzldf.cn/51997.Doc
m.wev.hzldf.cn/00802.Doc
m.wec.hzldf.cn/80044.Doc
m.wec.hzldf.cn/75775.Doc
m.wec.hzldf.cn/59797.Doc
m.wec.hzldf.cn/02228.Doc
m.wec.hzldf.cn/60266.Doc
m.wec.hzldf.cn/04888.Doc
m.wec.hzldf.cn/53179.Doc
m.wec.hzldf.cn/82288.Doc
m.wec.hzldf.cn/84284.Doc
m.wec.hzldf.cn/00000.Doc
m.wec.hzldf.cn/44068.Doc
m.wec.hzldf.cn/59577.Doc
m.wec.hzldf.cn/71399.Doc
m.wec.hzldf.cn/26466.Doc
m.wec.hzldf.cn/64404.Doc
m.wec.hzldf.cn/31759.Doc
m.wec.hzldf.cn/68082.Doc
m.wec.hzldf.cn/04864.Doc
m.wec.hzldf.cn/64828.Doc
m.wec.hzldf.cn/53977.Doc
m.wex.hzldf.cn/00662.Doc
m.wex.hzldf.cn/28666.Doc
m.wex.hzldf.cn/02408.Doc
m.wex.hzldf.cn/11999.Doc
m.wex.hzldf.cn/93599.Doc
m.wex.hzldf.cn/84820.Doc
m.wex.hzldf.cn/26802.Doc
m.wex.hzldf.cn/73559.Doc
m.wex.hzldf.cn/42684.Doc
m.wex.hzldf.cn/20224.Doc
m.wex.hzldf.cn/68824.Doc
m.wex.hzldf.cn/62280.Doc
m.wex.hzldf.cn/99375.Doc
m.wex.hzldf.cn/04622.Doc
m.wex.hzldf.cn/62086.Doc
m.wex.hzldf.cn/64082.Doc
m.wex.hzldf.cn/71797.Doc
m.wex.hzldf.cn/24640.Doc
m.wex.hzldf.cn/20666.Doc
m.wex.hzldf.cn/57933.Doc
m.wez.hzldf.cn/80420.Doc
m.wez.hzldf.cn/51537.Doc
m.wez.hzldf.cn/40806.Doc
m.wez.hzldf.cn/48028.Doc
m.wez.hzldf.cn/33533.Doc
m.wez.hzldf.cn/42066.Doc
m.wez.hzldf.cn/26844.Doc
m.wez.hzldf.cn/06266.Doc
m.wez.hzldf.cn/37791.Doc
m.wez.hzldf.cn/86402.Doc
m.wez.hzldf.cn/04486.Doc
m.wez.hzldf.cn/40420.Doc
m.wez.hzldf.cn/68028.Doc
m.wez.hzldf.cn/42486.Doc
m.wez.hzldf.cn/02026.Doc
m.wez.hzldf.cn/22842.Doc
m.wez.hzldf.cn/40224.Doc
m.wez.hzldf.cn/60260.Doc
m.wez.hzldf.cn/84446.Doc
m.wez.hzldf.cn/46284.Doc
m.wel.hzldf.cn/22460.Doc
m.wel.hzldf.cn/02448.Doc
m.wel.hzldf.cn/13111.Doc
m.wel.hzldf.cn/02226.Doc
m.wel.hzldf.cn/48682.Doc
m.wel.hzldf.cn/42606.Doc
m.wel.hzldf.cn/80228.Doc
m.wel.hzldf.cn/46060.Doc
m.wel.hzldf.cn/80288.Doc
m.wel.hzldf.cn/40422.Doc
m.wel.hzldf.cn/60664.Doc
m.wel.hzldf.cn/35531.Doc
m.wel.hzldf.cn/53359.Doc
m.wel.hzldf.cn/20804.Doc
m.wel.hzldf.cn/86048.Doc
m.wel.hzldf.cn/20606.Doc
m.wel.hzldf.cn/55399.Doc
m.wel.hzldf.cn/64042.Doc
m.wel.hzldf.cn/35951.Doc
m.wel.hzldf.cn/06288.Doc
m.wek.hzldf.cn/00604.Doc
m.wek.hzldf.cn/82024.Doc
m.wek.hzldf.cn/99733.Doc
m.wek.hzldf.cn/86684.Doc
m.wek.hzldf.cn/60848.Doc
m.wek.hzldf.cn/20082.Doc
m.wek.hzldf.cn/00646.Doc
m.wek.hzldf.cn/04464.Doc
m.wek.hzldf.cn/42626.Doc
m.wek.hzldf.cn/59917.Doc
m.wek.hzldf.cn/64228.Doc
m.wek.hzldf.cn/22828.Doc
m.wek.hzldf.cn/46226.Doc
m.wek.hzldf.cn/37737.Doc
m.wek.hzldf.cn/02060.Doc
m.wek.hzldf.cn/44860.Doc
m.wek.hzldf.cn/02226.Doc
m.wek.hzldf.cn/59319.Doc
m.wek.hzldf.cn/22448.Doc
m.wek.hzldf.cn/48600.Doc
m.wej.hzldf.cn/20448.Doc
m.wej.hzldf.cn/26484.Doc
m.wej.hzldf.cn/44682.Doc
m.wej.hzldf.cn/79355.Doc
m.wej.hzldf.cn/33119.Doc
m.wej.hzldf.cn/04408.Doc
m.wej.hzldf.cn/15535.Doc
m.wej.hzldf.cn/33135.Doc
m.wej.hzldf.cn/60260.Doc
m.wej.hzldf.cn/62862.Doc
m.wej.hzldf.cn/62808.Doc
m.wej.hzldf.cn/48488.Doc
m.wej.hzldf.cn/77193.Doc
m.wej.hzldf.cn/19791.Doc
m.wej.hzldf.cn/46080.Doc
m.wej.hzldf.cn/75317.Doc
m.wej.hzldf.cn/86226.Doc
m.wej.hzldf.cn/64806.Doc
m.wej.hzldf.cn/51995.Doc
m.wej.hzldf.cn/86848.Doc
m.weh.hzldf.cn/66422.Doc
m.weh.hzldf.cn/44442.Doc
m.weh.hzldf.cn/71573.Doc
m.weh.hzldf.cn/64602.Doc
m.weh.hzldf.cn/53775.Doc
m.weh.hzldf.cn/00086.Doc
m.weh.hzldf.cn/44042.Doc
m.weh.hzldf.cn/24666.Doc
m.weh.hzldf.cn/80846.Doc
m.weh.hzldf.cn/77311.Doc
m.weh.hzldf.cn/84266.Doc
m.weh.hzldf.cn/13177.Doc
m.weh.hzldf.cn/68806.Doc
m.weh.hzldf.cn/22686.Doc
m.weh.hzldf.cn/91333.Doc
m.weh.hzldf.cn/26466.Doc
m.weh.hzldf.cn/02606.Doc
m.weh.hzldf.cn/62060.Doc
m.weh.hzldf.cn/79553.Doc
m.weh.hzldf.cn/04462.Doc
m.weg.hzldf.cn/64046.Doc
m.weg.hzldf.cn/79317.Doc
m.weg.hzldf.cn/57913.Doc
m.weg.hzldf.cn/86426.Doc
m.weg.hzldf.cn/42228.Doc
m.weg.hzldf.cn/59755.Doc
m.weg.hzldf.cn/95531.Doc
m.weg.hzldf.cn/42208.Doc
m.weg.hzldf.cn/86248.Doc
m.weg.hzldf.cn/86482.Doc
m.weg.hzldf.cn/42626.Doc
m.weg.hzldf.cn/28264.Doc
m.weg.hzldf.cn/20002.Doc
m.weg.hzldf.cn/68444.Doc
m.weg.hzldf.cn/06228.Doc
m.weg.hzldf.cn/75155.Doc
m.weg.hzldf.cn/57791.Doc
m.weg.hzldf.cn/22646.Doc
m.weg.hzldf.cn/42202.Doc
m.weg.hzldf.cn/48282.Doc
m.wef.hzldf.cn/06248.Doc
m.wef.hzldf.cn/80224.Doc
m.wef.hzldf.cn/44840.Doc
m.wef.hzldf.cn/28844.Doc
m.wef.hzldf.cn/82402.Doc
m.wef.hzldf.cn/26284.Doc
m.wef.hzldf.cn/86424.Doc
m.wef.hzldf.cn/20808.Doc
m.wef.hzldf.cn/59551.Doc
m.wef.hzldf.cn/20008.Doc
m.wef.hzldf.cn/88024.Doc
m.wef.hzldf.cn/19393.Doc
m.wef.hzldf.cn/20820.Doc
m.wef.hzldf.cn/44020.Doc
m.wef.hzldf.cn/59317.Doc
m.wef.hzldf.cn/80888.Doc
m.wef.hzldf.cn/04820.Doc
m.wef.hzldf.cn/97557.Doc
m.wef.hzldf.cn/19711.Doc
m.wef.hzldf.cn/46284.Doc
m.wed.hzldf.cn/46422.Doc
m.wed.hzldf.cn/84866.Doc
m.wed.hzldf.cn/22806.Doc
m.wed.hzldf.cn/77311.Doc
m.wed.hzldf.cn/82686.Doc
m.wed.hzldf.cn/08446.Doc
m.wed.hzldf.cn/08604.Doc
m.wed.hzldf.cn/19739.Doc
m.wed.hzldf.cn/40822.Doc
m.wed.hzldf.cn/82680.Doc
m.wed.hzldf.cn/84246.Doc
m.wed.hzldf.cn/17951.Doc
m.wed.hzldf.cn/24228.Doc
m.wed.hzldf.cn/08804.Doc
m.wed.hzldf.cn/48026.Doc
