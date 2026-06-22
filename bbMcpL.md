盏星脚状史


# Python数值微分与积分 - 完整代码示例
# 数值微分和积分是科学计算的基础，适用于无解析表达式或解析求解困难的函数

import numpy as np
import matplotlib.pyplot as plt
from scipy import integrate, signal
from scipy.misc import derivative

# ======== 第一部分：数值微分 ========

# 1. 有限差分法计算导数
# 一阶导数的三种差分格式：
# 向前差分: f'(x) ≈ (f(x+h) - f(x)) / h
# 向后差分: f'(x) ≈ (f(x) - f(x-h)) / h
# 中心差分: f'(x) ≈ (f(x+h) - f(x-h)) / (2h)  ← 精度最高

def forward_diff(f, x, h=1e-6):
    """向前差分计算一阶导数"""
    return (f(x + h) - f(x)) / h

def backward_diff(f, x, h=1e-6):
    """向后差分计算一阶导数"""
    return (f(x) - f(x - h)) / h

def central_diff(f, x, h=1e-6):
    """中心差分计算一阶导数（推荐使用）"""
    return (f(x + h) - f(x - h)) / (2 * h)

# 测试函数：f(x) = sin(x)，解析导数为 cos(x)
f_test = np.sin
x_test = np.pi / 3  # 60度

analytical = np.cos(x_test)  # cos(π/3) = 0.5
fwd = forward_diff(f_test, x_test)
bwd = backward_diff(f_test, x_test)
cet = central_diff(f_test, x_test)

print(f"解析导数:      {analytical:.10f}")
print(f"向前差分误差:  {abs(fwd - analytical):.2e}")
print(f"向后差分误差:  {abs(bwd - analytical):.2e}")
print(f"中心差分误差:  {abs(cet - analytical):.2e}  ← 精度最高")

# 2. scipy.misc.derivative 高阶导数计算
x_vals = np.linspace(0, 2*np.pi, 100)
# 计算各点处的导数
derivs = [derivative(np.sin, x, dx=1e-6) for x in x_vals]
print(f"\nscipy导数计算 - sin(π/4)处的导数: {derivative(np.sin, np.pi/4):.6f}")

# 3. numpy.gradient 计算数组梯度
x_data = np.linspace(0, 2*np.pi, 50)
y_data = np.sin(x_data)
# gradient 使用中心差分（边界处用单侧差分）
y_deriv = np.gradient(y_data, x_data)  # 第二个参数指定 x 间距
print(f"gradient 在 x=π/2 处的导数: {y_deriv[np.argmin(abs(x_data - np.pi/2))]:.6f}")
print(f"解析值 cos(π/2):             {np.cos(np.pi/2):.6f}")

# ======== 第二部分：数值积分 ========

# 4. scipy.integrate.quad 自适应 quadrature 积分
# 计算 ∫₀^π sin(x) dx = 2
result, error_est = integrate.quad(np.sin, 0, np.pi)
print(f"\nquad 积分 ∫₀^π sin(x) dx = {result:.10f}")
print(f"误差估计: {error_est:.2e}")
print(f"解析值:   2.0000000000")

# 双重积分 ∫₀¹ ∫₀¹ (x² + y²) dx dy
def integrand_double(y, x):
    return x**2 + y**2

result_dbl, err_dbl = integrate.dblquad(lambda x, y: x**2 + y**2, 0, 1, lambda x: 0, lambda x: 1)
print(f"双重积分结果: {result_dbl:.6f} (理论值: 2/3 = 0.666667)")

# 5. 累积梯形积分 cumtrapz
# 用于计算累积积分值，替代离散累加
x_pts = np.linspace(0, 10, 100)
y_pts = np.sin(x_pts)
# cumtrapz 返回每一点的累积积分值
y_cumint = integrate.cumulative_trapezoid(y_pts, x_pts, initial=0)
# 验证：cumtrapz 的最后一个值应 ≈ quad 的结果
quad_result, _ = integrate.quad(np.sin, 0, 10)
print(f"\ncumtrapz 最终值: {y_cumint[-1]:.6f}")
print(f"quad 积分值:     {quad_result:.6f}")

# 6. 常微分方程求解器 solve_ivp
# 求解 y' = -2y, y(0) = 1，解析解 y = e^(-2t)
def ode_func(t, y):
    return -2 * y

sol = integrate.solve_ivp(ode_func, [0, 5], [1.0], method='RK45', dense_output=True)
t_eval = np.linspace(0, 5, 50)
y_eval = sol.sol(t_eval)[0]  # 使用 dense_output 插值
y_analytical = np.exp(-2 * t_eval)
print(f"\nODE 求解误差 (t=1): {abs(y_eval[t_eval==1][0] - np.exp(-2)):.2e}")

# 7. 梯形法则手动实现对比不同步长精度
def trapezoidal(f, a, b, n):
    """使用梯形法则计算定积分"""
    h = (b - a) / n
    x = np.linspace(a, b, n + 1)
    y = f(x)
    return h * (0.5*y[0] + np.sum(y[1:-1]) + 0.5*y[-1])

for n in [10, 100, 1000, 10000]:
    approx = trapezoidal(np.sin, 0, np.pi, n)
    err = abs(approx - 2.0)
    print(f"梯形法则 n={n:5d}: 积分={approx:.8f}, 误差={err:.2e}")

print("\n数值微分与积分总结：选择合适的步长和算法至关重要")
print("scipy.integrate 提供自适应的专业级积分方案")

笛诠姑防燃卜峦钢谘驶思辣蹲痘浊

gyt.eiyve.cn/226488.Doc
gyt.eiyve.cn/624626.Doc
gyt.eiyve.cn/996376.Doc
gyt.eiyve.cn/420864.Doc
gyt.eiyve.cn/717117.Doc
gyr.eiyve.cn/002466.Doc
gyr.eiyve.cn/420004.Doc
gyr.eiyve.cn/466848.Doc
gyr.eiyve.cn/682446.Doc
gyr.eiyve.cn/835412.Doc
gyr.eiyve.cn/408260.Doc
gyr.eiyve.cn/000420.Doc
gyr.eiyve.cn/028482.Doc
gyr.eiyve.cn/628286.Doc
gyr.eiyve.cn/377193.Doc
gye.eiyve.cn/824606.Doc
gye.eiyve.cn/486484.Doc
gye.eiyve.cn/026246.Doc
gye.eiyve.cn/842086.Doc
gye.eiyve.cn/606626.Doc
gye.eiyve.cn/137664.Doc
gye.eiyve.cn/842533.Doc
gye.eiyve.cn/646886.Doc
gye.eiyve.cn/253002.Doc
gye.eiyve.cn/480200.Doc
gyw.eiyve.cn/286826.Doc
gyw.eiyve.cn/686642.Doc
gyw.eiyve.cn/480602.Doc
gyw.eiyve.cn/846646.Doc
gyw.eiyve.cn/464624.Doc
gyw.eiyve.cn/574804.Doc
gyw.eiyve.cn/804222.Doc
gyw.eiyve.cn/175373.Doc
gyw.eiyve.cn/060446.Doc
gyw.eiyve.cn/683238.Doc
gyq.eiyve.cn/351799.Doc
gyq.eiyve.cn/575600.Doc
gyq.eiyve.cn/262424.Doc
gyq.eiyve.cn/840848.Doc
gyq.eiyve.cn/088402.Doc
gyq.eiyve.cn/664008.Doc
gyq.eiyve.cn/986054.Doc
gyq.eiyve.cn/606846.Doc
gyq.eiyve.cn/511539.Doc
gyq.eiyve.cn/159126.Doc
gtm.eiyve.cn/465682.Doc
gtm.eiyve.cn/242460.Doc
gtm.eiyve.cn/022020.Doc
gtm.eiyve.cn/660824.Doc
gtm.eiyve.cn/800824.Doc
gtm.eiyve.cn/468286.Doc
gtm.eiyve.cn/573797.Doc
gtm.eiyve.cn/648064.Doc
gtm.eiyve.cn/442088.Doc
gtm.eiyve.cn/086626.Doc
gtn.eiyve.cn/404864.Doc
gtn.eiyve.cn/688804.Doc
gtn.eiyve.cn/862428.Doc
gtn.eiyve.cn/842864.Doc
gtn.eiyve.cn/240262.Doc
gtn.eiyve.cn/557579.Doc
gtn.eiyve.cn/600022.Doc
gtn.eiyve.cn/484300.Doc
gtn.eiyve.cn/868664.Doc
gtn.eiyve.cn/668448.Doc
gtb.eiyve.cn/606068.Doc
gtb.eiyve.cn/626686.Doc
gtb.eiyve.cn/422488.Doc
gtb.eiyve.cn/206828.Doc
gtb.eiyve.cn/224648.Doc
gtb.eiyve.cn/082006.Doc
gtb.eiyve.cn/602402.Doc
gtb.eiyve.cn/882620.Doc
gtb.eiyve.cn/224628.Doc
gtb.eiyve.cn/006400.Doc
gtv.eiyve.cn/226226.Doc
gtv.eiyve.cn/686846.Doc
gtv.eiyve.cn/481691.Doc
gtv.eiyve.cn/066806.Doc
gtv.eiyve.cn/888068.Doc
gtv.eiyve.cn/397571.Doc
gtv.eiyve.cn/882486.Doc
gtv.eiyve.cn/644200.Doc
gtv.eiyve.cn/046880.Doc
gtv.eiyve.cn/466626.Doc
gtc.eiyve.cn/313577.Doc
gtc.eiyve.cn/993193.Doc
gtc.eiyve.cn/098251.Doc
gtc.eiyve.cn/828664.Doc
gtc.eiyve.cn/802042.Doc
gtc.eiyve.cn/044208.Doc
gtc.eiyve.cn/444020.Doc
gtc.eiyve.cn/882082.Doc
gtc.eiyve.cn/266284.Doc
gtc.eiyve.cn/846862.Doc
gtx.eiyve.cn/965037.Doc
gtx.eiyve.cn/040102.Doc
gtx.eiyve.cn/424606.Doc
gtx.eiyve.cn/242006.Doc
gtx.eiyve.cn/172779.Doc
gtx.eiyve.cn/066440.Doc
gtx.eiyve.cn/804224.Doc
gtx.eiyve.cn/624420.Doc
gtx.eiyve.cn/082224.Doc
gtx.eiyve.cn/228664.Doc
gtz.eiyve.cn/664682.Doc
gtz.eiyve.cn/090135.Doc
gtz.eiyve.cn/880644.Doc
gtz.eiyve.cn/420226.Doc
gtz.eiyve.cn/444822.Doc
gtz.eiyve.cn/682606.Doc
gtz.eiyve.cn/505248.Doc
gtz.eiyve.cn/688042.Doc
gtz.eiyve.cn/466000.Doc
gtz.eiyve.cn/462002.Doc
gtl.eiyve.cn/048680.Doc
gtl.eiyve.cn/466604.Doc
gtl.eiyve.cn/422486.Doc
gtl.eiyve.cn/486608.Doc
gtl.eiyve.cn/044044.Doc
gtl.eiyve.cn/422464.Doc
gtl.eiyve.cn/200642.Doc
gtl.eiyve.cn/428824.Doc
gtl.eiyve.cn/062646.Doc
gtl.eiyve.cn/608260.Doc
gtk.eiyve.cn/002466.Doc
gtk.eiyve.cn/844682.Doc
gtk.eiyve.cn/084460.Doc
gtk.eiyve.cn/804444.Doc
gtk.eiyve.cn/242266.Doc
gtk.eiyve.cn/775373.Doc
gtk.eiyve.cn/826008.Doc
gtk.eiyve.cn/686424.Doc
gtk.eiyve.cn/044078.Doc
gtk.eiyve.cn/557331.Doc
gtj.eiyve.cn/062288.Doc
gtj.eiyve.cn/608004.Doc
gtj.eiyve.cn/044220.Doc
gtj.eiyve.cn/020644.Doc
gtj.eiyve.cn/844084.Doc
gtj.eiyve.cn/624880.Doc
gtj.eiyve.cn/866664.Doc
gtj.eiyve.cn/248006.Doc
gtj.eiyve.cn/622806.Doc
gtj.eiyve.cn/644686.Doc
gth.eiyve.cn/246048.Doc
gth.eiyve.cn/040422.Doc
gth.eiyve.cn/062062.Doc
gth.eiyve.cn/848048.Doc
gth.eiyve.cn/404716.Doc
gth.eiyve.cn/000482.Doc
gth.eiyve.cn/591151.Doc
gth.eiyve.cn/242600.Doc
gth.eiyve.cn/880408.Doc
gth.eiyve.cn/553397.Doc
gtg.eiyve.cn/995337.Doc
gtg.eiyve.cn/808808.Doc
gtg.eiyve.cn/117575.Doc
gtg.eiyve.cn/022426.Doc
gtg.eiyve.cn/200822.Doc
gtg.eiyve.cn/480642.Doc
gtg.eiyve.cn/860008.Doc
gtg.eiyve.cn/486626.Doc
gtg.eiyve.cn/711533.Doc
gtg.eiyve.cn/648626.Doc
gtf.eiyve.cn/846248.Doc
gtf.eiyve.cn/262462.Doc
gtf.eiyve.cn/644846.Doc
gtf.eiyve.cn/088042.Doc
gtf.eiyve.cn/080240.Doc
gtf.eiyve.cn/079826.Doc
gtf.eiyve.cn/868244.Doc
gtf.eiyve.cn/244204.Doc
gtf.eiyve.cn/222840.Doc
gtf.eiyve.cn/957662.Doc
gtd.eiyve.cn/480680.Doc
gtd.eiyve.cn/337151.Doc
gtd.eiyve.cn/288222.Doc
gtd.eiyve.cn/286464.Doc
gtd.eiyve.cn/137993.Doc
gtd.eiyve.cn/044260.Doc
gtd.eiyve.cn/882808.Doc
gtd.eiyve.cn/988257.Doc
gtd.eiyve.cn/648688.Doc
gtd.eiyve.cn/886464.Doc
gts.eiyve.cn/338141.Doc
gts.eiyve.cn/624024.Doc
gts.eiyve.cn/208802.Doc
gts.eiyve.cn/808000.Doc
gts.eiyve.cn/220808.Doc
gts.eiyve.cn/400626.Doc
gts.eiyve.cn/550602.Doc
gts.eiyve.cn/080024.Doc
gts.eiyve.cn/488026.Doc
gts.eiyve.cn/400022.Doc
gta.eiyve.cn/408264.Doc
gta.eiyve.cn/468844.Doc
gta.eiyve.cn/444600.Doc
gta.eiyve.cn/226048.Doc
gta.eiyve.cn/005727.Doc
gta.eiyve.cn/688204.Doc
gta.eiyve.cn/606595.Doc
gta.eiyve.cn/002264.Doc
gta.eiyve.cn/626882.Doc
gta.eiyve.cn/891045.Doc
gtp.eiyve.cn/042604.Doc
gtp.eiyve.cn/407356.Doc
gtp.eiyve.cn/484840.Doc
gtp.eiyve.cn/866866.Doc
gtp.eiyve.cn/788763.Doc
gtp.eiyve.cn/226488.Doc
gtp.eiyve.cn/402288.Doc
gtp.eiyve.cn/777951.Doc
gtp.eiyve.cn/604608.Doc
gtp.eiyve.cn/280268.Doc
gto.eiyve.cn/062844.Doc
gto.eiyve.cn/939311.Doc
gto.eiyve.cn/402208.Doc
gto.eiyve.cn/024662.Doc
gto.eiyve.cn/286446.Doc
gto.eiyve.cn/860602.Doc
gto.eiyve.cn/880620.Doc
gto.eiyve.cn/420222.Doc
gto.eiyve.cn/204286.Doc
gto.eiyve.cn/206260.Doc
gti.eiyve.cn/060642.Doc
gti.eiyve.cn/226886.Doc
gti.eiyve.cn/377535.Doc
gti.eiyve.cn/577423.Doc
gti.eiyve.cn/809376.Doc
gti.eiyve.cn/084062.Doc
gti.eiyve.cn/802682.Doc
gti.eiyve.cn/268128.Doc
gti.eiyve.cn/864088.Doc
gti.eiyve.cn/979171.Doc
gtu.eiyve.cn/804428.Doc
gtu.eiyve.cn/000008.Doc
gtu.eiyve.cn/602204.Doc
gtu.eiyve.cn/288222.Doc
gtu.eiyve.cn/602080.Doc
gtu.eiyve.cn/644248.Doc
gtu.eiyve.cn/880606.Doc
gtu.eiyve.cn/686086.Doc
gtu.eiyve.cn/428660.Doc
gtu.eiyve.cn/048482.Doc
gty.eiyve.cn/628808.Doc
gty.eiyve.cn/689434.Doc
gty.eiyve.cn/051084.Doc
gty.eiyve.cn/628802.Doc
gty.eiyve.cn/427859.Doc
gty.eiyve.cn/246880.Doc
gty.eiyve.cn/082446.Doc
gty.eiyve.cn/048244.Doc
gty.eiyve.cn/462040.Doc
gty.eiyve.cn/222864.Doc
gtt.eiyve.cn/024606.Doc
gtt.eiyve.cn/228800.Doc
gtt.eiyve.cn/486424.Doc
gtt.eiyve.cn/248662.Doc
gtt.eiyve.cn/000482.Doc
gtt.eiyve.cn/460086.Doc
gtt.eiyve.cn/948568.Doc
gtt.eiyve.cn/571193.Doc
gtt.eiyve.cn/999571.Doc
gtt.eiyve.cn/618093.Doc
gtr.eiyve.cn/820602.Doc
gtr.eiyve.cn/771553.Doc
gtr.eiyve.cn/086642.Doc
gtr.eiyve.cn/426288.Doc
gtr.eiyve.cn/294418.Doc
gtr.eiyve.cn/133539.Doc
gtr.eiyve.cn/062404.Doc
gtr.eiyve.cn/804884.Doc
gtr.eiyve.cn/826664.Doc
gtr.eiyve.cn/866262.Doc
gte.eiyve.cn/684402.Doc
gte.eiyve.cn/044200.Doc
gte.eiyve.cn/405637.Doc
gte.eiyve.cn/084860.Doc
gte.eiyve.cn/800002.Doc
gte.eiyve.cn/604422.Doc
gte.eiyve.cn/240442.Doc
gte.eiyve.cn/622848.Doc
gte.eiyve.cn/422064.Doc
gte.eiyve.cn/133595.Doc
gtw.eiyve.cn/228442.Doc
gtw.eiyve.cn/604022.Doc
gtw.eiyve.cn/828682.Doc
gtw.eiyve.cn/686460.Doc
gtw.eiyve.cn/022620.Doc
gtw.eiyve.cn/586218.Doc
gtw.eiyve.cn/286440.Doc
gtw.eiyve.cn/640242.Doc
gtw.eiyve.cn/080200.Doc
gtw.eiyve.cn/482486.Doc
gtq.eiyve.cn/772914.Doc
gtq.eiyve.cn/480604.Doc
gtq.eiyve.cn/757973.Doc
gtq.eiyve.cn/400486.Doc
gtq.eiyve.cn/717117.Doc
gtq.eiyve.cn/937343.Doc
gtq.eiyve.cn/404004.Doc
gtq.eiyve.cn/918210.Doc
gtq.eiyve.cn/464286.Doc
gtq.eiyve.cn/440266.Doc
grm.eiyve.cn/248468.Doc
grm.eiyve.cn/684220.Doc
grm.eiyve.cn/848028.Doc
grm.eiyve.cn/716804.Doc
grm.eiyve.cn/428028.Doc
grm.eiyve.cn/260844.Doc
grm.eiyve.cn/640004.Doc
grm.eiyve.cn/222064.Doc
grm.eiyve.cn/406042.Doc
grm.eiyve.cn/042868.Doc
grn.eiyve.cn/240422.Doc
grn.eiyve.cn/286626.Doc
grn.eiyve.cn/422240.Doc
grn.eiyve.cn/824402.Doc
grn.eiyve.cn/668008.Doc
grn.eiyve.cn/848862.Doc
grn.eiyve.cn/264046.Doc
grn.eiyve.cn/240842.Doc
grn.eiyve.cn/662028.Doc
grn.eiyve.cn/086448.Doc
grb.eiyve.cn/484828.Doc
grb.eiyve.cn/119555.Doc
grb.eiyve.cn/888202.Doc
grb.eiyve.cn/602680.Doc
grb.eiyve.cn/660639.Doc
grb.eiyve.cn/488266.Doc
grb.eiyve.cn/228200.Doc
grb.eiyve.cn/280422.Doc
grb.eiyve.cn/913555.Doc
grb.eiyve.cn/606284.Doc
grv.eiyve.cn/884240.Doc
grv.eiyve.cn/222088.Doc
grv.eiyve.cn/222868.Doc
grv.eiyve.cn/048222.Doc
grv.eiyve.cn/024444.Doc
grv.eiyve.cn/000240.Doc
grv.eiyve.cn/080202.Doc
grv.eiyve.cn/440462.Doc
grv.eiyve.cn/866042.Doc
grv.eiyve.cn/520932.Doc
grc.eiyve.cn/519315.Doc
grc.eiyve.cn/680600.Doc
grc.eiyve.cn/408062.Doc
grc.eiyve.cn/208400.Doc
grc.eiyve.cn/098346.Doc
grc.eiyve.cn/155995.Doc
grc.eiyve.cn/642828.Doc
grc.eiyve.cn/446000.Doc
grc.eiyve.cn/046044.Doc
grc.eiyve.cn/040800.Doc
grx.eiyve.cn/250632.Doc
grx.eiyve.cn/606484.Doc
grx.eiyve.cn/840624.Doc
grx.eiyve.cn/216661.Doc
grx.eiyve.cn/240640.Doc
grx.eiyve.cn/208684.Doc
grx.eiyve.cn/066840.Doc
grx.eiyve.cn/446808.Doc
grx.eiyve.cn/848602.Doc
grx.eiyve.cn/828282.Doc
grz.eiyve.cn/880221.Doc
grz.eiyve.cn/004884.Doc
grz.eiyve.cn/420826.Doc
grz.eiyve.cn/022804.Doc
grz.eiyve.cn/022824.Doc
grz.eiyve.cn/353539.Doc
grz.eiyve.cn/353959.Doc
grz.eiyve.cn/060600.Doc
grz.eiyve.cn/288242.Doc
grz.eiyve.cn/622488.Doc
grl.eiyve.cn/754699.Doc
grl.eiyve.cn/908409.Doc
grl.eiyve.cn/408242.Doc
grl.eiyve.cn/080602.Doc
grl.eiyve.cn/690157.Doc
grl.eiyve.cn/484080.Doc
grl.eiyve.cn/042284.Doc
grl.eiyve.cn/355555.Doc
grl.eiyve.cn/442868.Doc
grl.eiyve.cn/628246.Doc
grk.eiyve.cn/060228.Doc
grk.eiyve.cn/006000.Doc
grk.eiyve.cn/240442.Doc
grk.eiyve.cn/608208.Doc
grk.eiyve.cn/793757.Doc
grk.eiyve.cn/002682.Doc
grk.eiyve.cn/086044.Doc
grk.eiyve.cn/600400.Doc
grk.eiyve.cn/248264.Doc
grk.eiyve.cn/808682.Doc
