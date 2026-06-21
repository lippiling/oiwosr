干匾是磐肪


# Python曲线拟合 - 完整代码示例
# 曲线拟合通过优化参数使模型函数最佳匹配观测数据

import numpy as np
import matplotlib.pyplot as plt
from scipy import optimize, stats

# 1. 多项式拟合：numpy.polyfit
# 使用最小二乘法拟合多项式到数据点
np.random.seed(42)
x_data = np.linspace(-3, 3, 30)
# 真实模型 y = 2x² - 3x + 1 + 噪声
y_true = 2 * x_data**2 - 3 * x_data + 1
y_observed = y_true + 2.0 * np.random.randn(30)  # 添加噪声

# 拟合不同阶数的多项式
deg2_coeffs = np.polyfit(x_data, y_observed, 2)  # 二次多项式
deg5_coeffs = np.polyfit(x_data, y_observed, 5)  # 五次多项式（可能过拟合）

# 使用 poly1d 从系数创建多项式函数
p2 = np.poly1d(deg2_coeffs)
p5 = np.poly1d(deg5_coeffs)

print("多项式拟合结果:")
print(f"真实系数:     [2, -3, 1]")
print(f"二次拟合系数: {deg2_coeffs}")
print(f"五次拟合系数: {deg5_coeffs}")

# 2. scipy.optimize.curve_fit 非线性曲线拟合
# 定义要拟合的模型函数：高斯函数
def gaussian(x, A, mu, sigma, offset):
    """高斯函数模型：A * exp(-(x-mu)²/(2*sigma²)) + offset"""
    return A * np.exp(-(x - mu)**2 / (2 * sigma**2)) + offset

# 生成高斯型数据
x_gauss = np.linspace(-5, 5, 50)
y_gauss_true = 3.0 * np.exp(-(x_gauss - 0.5)**2 / (2 * 0.8**2)) + 0.5
y_gauss_obs = y_gauss_true + 0.3 * np.random.randn(50)

# 使用 curve_fit 拟合，需要提供初始猜测
initial_guess = [2.0, 0.0, 1.0, 0.0]  # [A, mu, sigma, offset]
popt, pcov = optimize.curve_fit(gaussian, x_gauss, y_gauss_obs, p0=initial_guess)

# popt 是最优参数，pcov 是协方差矩阵（用于计算参数误差）
perr = np.sqrt(np.diag(pcov))  # 参数的标准误差

print(f"\n高斯拟合结果:")
print(f"参数     真实值    拟合值    标准误差")
params_names = ['A', 'mu', 'sigma', 'offset']
params_true = [3.0, 0.5, 0.8, 0.5]
for i, name in enumerate(params_names):
    print(f"{name:8s} {params_true[i]:8.3f} {popt[i]:8.3f} {perr[i]:8.4f}")

# 3. 拟合优度评估
y_pred = gaussian(x_gauss, *popt)
residuals = y_gauss_obs - y_pred
ss_res = np.sum(residuals**2)  # 残差平方和
ss_tot = np.sum((y_gauss_obs - np.mean(y_gauss_obs))**2)  # 总平方和
r_squared = 1 - ss_res / ss_tot  # R² 决定系数

# 计算调整 R² (考虑参数数量)
n = len(y_gauss_obs)
k = len(popt)
adj_r_squared = 1 - (1 - r_squared) * (n - 1) / (n - k - 1)

print(f"\n拟合优度:")
print(f"R² = {r_squared:.4f}")
print(f"调整 R² = {adj_r_squared:.4f}")
print(f"残差标准差 = {np.std(residuals):.4f}")

# 4. 置信带和预测带
# 使用协方差矩阵计算拟合值的置信区间
alpha = 0.05  # 显著性水平 5%
t_value = stats.t.ppf(1 - alpha/2, n - k - 1)  # t 分布临界值

# 构建设计矩阵（模型在 x 处的雅可比矩阵）
J = np.zeros((len(x_gauss), len(popt)))
J[:, 0] = np.exp(-(x_gauss - popt[1])**2 / (2 * popt[2]**2))  # d/dA
J[:, 1] = popt[0] * np.exp(-(x_gauss - popt[1])**2 / (2 * popt[2]**2)) * (x_gauss - popt[1]) / popt[2]**2  # d/dmu
J[:, 2] = popt[0] * np.exp(-(x_gauss - popt[1])**2 / (2 * popt[2]**2)) * (x_gauss - popt[1])**2 / popt[2]**3  # d/dsigma
J[:, 3] = np.ones(len(x_gauss))  # d/doffset

# 拟合值的方差
y_pred_var = np.sum(J @ pcov * J, axis=1)
y_pred_se = np.sqrt(y_pred_var)  # 标准误差
ci_half = t_value * y_pred_se  # 置信区间半宽

# 预测区间（包含观测误差）
pi_half = t_value * np.sqrt(y_pred_var + np.var(residuals))

print(f"\n在 x=0.5 处的区间估计:")
print(f"拟合值: {gaussian(0.5, *popt):.4f}")
print(f"95% 置信区间: [{gaussian(0.5, *popt) - ci_half[np.argmin(abs(x_gauss - 0.5))]:.4f}, "
      f"{gaussian(0.5, *popt) + ci_half[np.argmin(abs(x_gauss - 0.5))]:.4f}]")

# 5. least_squares 更灵活的非线性最小二乘
# 当 curve_fit 不够灵活时，直接使用 least_squares

def residual_func(params, x, y):
    """残差函数：返回每个数据点的残差"""
    A, mu, sigma, offset = params
    y_model = gaussian(x, A, mu, sigma, offset)
    return y - y_model  # 返回残差向量

# 使用边界约束进行拟合（防止 sigma 为负）
bounds = ([-10, -5, 0.01, -5], [10, 5, 5, 5])  # (下界, 上界)
result_ls = optimize.least_squares(
    residual_func, initial_guess,
    args=(x_gauss, y_gauss_obs),
    bounds=bounds,
    method='trf'  # Trust Region Reflective
)

print(f"\nleast_squares 拟合结果:")
print(f"A = {result_ls.x[0]:.3f}, mu = {result_ls.x[1]:.3f}")
print(f"sigma = {result_ls.x[2]:.3f}, offset = {result_ls.x[3]:.3f}")
print(f"优化成功: {result_ls.success}")
print(f"迭代次数: {result_ls.nfev}")

# 6. 加权拟合：当数据点具有不同权重时
# 假设越靠近中心的数据越可靠（权重越高）
weights = np.exp(-x_gauss**2 / 10)  # 权重函数

def weighted_residual(params, x, y, w):
    """加权残差函数"""
    return w * residual_func(params, x, y)

result_weighted = optimize.least_squares(
    lambda p, x, y: weighted_residual(p, x, y, weights),
    initial_guess, args=(x_gauss, y_gauss_obs)
)
print(f"\n加权拟合 A = {result_weighted.x[0]:.3f}")

# 7. 模型选择：比较不同模型的 AIC/BIC
# AIC = n * ln(RSS/n) + 2k, BIC = n * ln(RSS/n) + k*ln(n)
for deg in [1, 2, 3, 4, 5]:
    coeffs = np.polyfit(x_data, y_observed, deg)
    y_pred = np.polyval(coeffs, x_data)
    rss = np.sum((y_observed - y_pred)**2)
    k = deg + 1  # 参数个数
    aic = n * np.log(rss / n) + 2 * k
    bic = n * np.log(rss / n) + k * np.log(n)
    print(f"阶数 {deg}: AIC={aic:.1f}, BIC={bic:.1f}, RSS={rss:.2f}")

print("\n曲线拟合总结：选择合适模型复杂度，避免过拟合")
print("curve_fit 适合标准模型，least_squares 提供更多控制")

障狼谱督诙忻氐扰系敌障诺兑拙徒

m.wrm.hmybk.cn/64080.Doc
m.wrm.hmybk.cn/39513.Doc
m.wrm.hmybk.cn/44642.Doc
m.wrm.hmybk.cn/08800.Doc
m.wrm.hmybk.cn/42244.Doc
m.wrm.hmybk.cn/82020.Doc
m.wrm.hmybk.cn/28202.Doc
m.wrm.hmybk.cn/28884.Doc
m.wrm.hmybk.cn/86002.Doc
m.wrm.hmybk.cn/64644.Doc
m.wrm.hmybk.cn/20240.Doc
m.wrm.hmybk.cn/02688.Doc
m.wrm.hmybk.cn/75311.Doc
m.wrm.hmybk.cn/24284.Doc
m.wrm.hmybk.cn/06648.Doc
m.wrn.hmybk.cn/04628.Doc
m.wrn.hmybk.cn/04422.Doc
m.wrn.hmybk.cn/28280.Doc
m.wrn.hmybk.cn/66884.Doc
m.wrn.hmybk.cn/20406.Doc
m.wrn.hmybk.cn/46600.Doc
m.wrn.hmybk.cn/77377.Doc
m.wrn.hmybk.cn/20480.Doc
m.wrn.hmybk.cn/97719.Doc
m.wrn.hmybk.cn/60484.Doc
m.wrn.hmybk.cn/06086.Doc
m.wrn.hmybk.cn/42862.Doc
m.wrn.hmybk.cn/40288.Doc
m.wrn.hmybk.cn/62468.Doc
m.wrn.hmybk.cn/24200.Doc
m.wrn.hmybk.cn/44408.Doc
m.wrn.hmybk.cn/37379.Doc
m.wrn.hmybk.cn/44686.Doc
m.wrn.hmybk.cn/15779.Doc
m.wrn.hmybk.cn/22002.Doc
m.wrb.hmybk.cn/64840.Doc
m.wrb.hmybk.cn/79151.Doc
m.wrb.hmybk.cn/80644.Doc
m.wrb.hmybk.cn/44244.Doc
m.wrb.hmybk.cn/80640.Doc
m.wrb.hmybk.cn/68800.Doc
m.wrb.hmybk.cn/75533.Doc
m.wrb.hmybk.cn/20202.Doc
m.wrb.hmybk.cn/06082.Doc
m.wrb.hmybk.cn/66824.Doc
m.wrb.hmybk.cn/48248.Doc
m.wrb.hmybk.cn/75791.Doc
m.wrb.hmybk.cn/68000.Doc
m.wrb.hmybk.cn/04422.Doc
m.wrb.hmybk.cn/93573.Doc
m.wrb.hmybk.cn/80066.Doc
m.wrb.hmybk.cn/80222.Doc
m.wrb.hmybk.cn/48088.Doc
m.wrb.hmybk.cn/95733.Doc
m.wrb.hmybk.cn/93957.Doc
m.wrv.hmybk.cn/80400.Doc
m.wrv.hmybk.cn/06288.Doc
m.wrv.hmybk.cn/99351.Doc
m.wrv.hmybk.cn/26644.Doc
m.wrv.hmybk.cn/42884.Doc
m.wrv.hmybk.cn/68220.Doc
m.wrv.hmybk.cn/66224.Doc
m.wrv.hmybk.cn/04606.Doc
m.wrv.hmybk.cn/59995.Doc
m.wrv.hmybk.cn/88646.Doc
m.wrv.hmybk.cn/44888.Doc
m.wrv.hmybk.cn/44240.Doc
m.wrv.hmybk.cn/19393.Doc
m.wrv.hmybk.cn/68826.Doc
m.wrv.hmybk.cn/66488.Doc
m.wrv.hmybk.cn/48202.Doc
m.wrv.hmybk.cn/08086.Doc
m.wrv.hmybk.cn/88466.Doc
m.wrv.hmybk.cn/64442.Doc
m.wrv.hmybk.cn/53151.Doc
m.wrc.hmybk.cn/86662.Doc
m.wrc.hmybk.cn/93755.Doc
m.wrc.hmybk.cn/51913.Doc
m.wrc.hmybk.cn/84846.Doc
m.wrc.hmybk.cn/26042.Doc
m.wrc.hmybk.cn/62664.Doc
m.wrc.hmybk.cn/97397.Doc
m.wrc.hmybk.cn/15171.Doc
m.wrc.hmybk.cn/08646.Doc
m.wrc.hmybk.cn/88288.Doc
m.wrc.hmybk.cn/64046.Doc
m.wrc.hmybk.cn/22604.Doc
m.wrc.hmybk.cn/91517.Doc
m.wrc.hmybk.cn/46228.Doc
m.wrc.hmybk.cn/44800.Doc
m.wrc.hmybk.cn/06084.Doc
m.wrc.hmybk.cn/08604.Doc
m.wrc.hmybk.cn/37595.Doc
m.wrc.hmybk.cn/73171.Doc
m.wrc.hmybk.cn/97397.Doc
m.wrx.hmybk.cn/66846.Doc
m.wrx.hmybk.cn/86424.Doc
m.wrx.hmybk.cn/17351.Doc
m.wrx.hmybk.cn/04444.Doc
m.wrx.hmybk.cn/80042.Doc
m.wrx.hmybk.cn/26244.Doc
m.wrx.hmybk.cn/86400.Doc
m.wrx.hmybk.cn/39913.Doc
m.wrx.hmybk.cn/60462.Doc
m.wrx.hmybk.cn/51317.Doc
m.wrx.hmybk.cn/60666.Doc
m.wrx.hmybk.cn/46428.Doc
m.wrx.hmybk.cn/00022.Doc
m.wrx.hmybk.cn/51515.Doc
m.wrx.hmybk.cn/26008.Doc
m.wrx.hmybk.cn/08604.Doc
m.wrx.hmybk.cn/40666.Doc
m.wrx.hmybk.cn/17757.Doc
m.wrx.hmybk.cn/04204.Doc
m.wrx.hmybk.cn/68680.Doc
m.wrz.hmybk.cn/02042.Doc
m.wrz.hmybk.cn/79951.Doc
m.wrz.hmybk.cn/62642.Doc
m.wrz.hmybk.cn/51397.Doc
m.wrz.hmybk.cn/64208.Doc
m.wrz.hmybk.cn/60686.Doc
m.wrz.hmybk.cn/44080.Doc
m.wrz.hmybk.cn/79519.Doc
m.wrz.hmybk.cn/04280.Doc
m.wrz.hmybk.cn/02628.Doc
m.wrz.hmybk.cn/44466.Doc
m.wrz.hmybk.cn/26228.Doc
m.wrz.hmybk.cn/46002.Doc
m.wrz.hmybk.cn/44466.Doc
m.wrz.hmybk.cn/68862.Doc
m.wrz.hmybk.cn/04020.Doc
m.wrz.hmybk.cn/60008.Doc
m.wrz.hmybk.cn/40408.Doc
m.wrz.hmybk.cn/40246.Doc
m.wrz.hmybk.cn/22882.Doc
m.wrl.hmybk.cn/04608.Doc
m.wrl.hmybk.cn/80028.Doc
m.wrl.hmybk.cn/68224.Doc
m.wrl.hmybk.cn/68608.Doc
m.wrl.hmybk.cn/95735.Doc
m.wrl.hmybk.cn/62000.Doc
m.wrl.hmybk.cn/46868.Doc
m.wrl.hmybk.cn/62888.Doc
m.wrl.hmybk.cn/48246.Doc
m.wrl.hmybk.cn/46466.Doc
m.wrl.hmybk.cn/13155.Doc
m.wrl.hmybk.cn/46622.Doc
m.wrl.hmybk.cn/62082.Doc
m.wrl.hmybk.cn/28482.Doc
m.wrl.hmybk.cn/33937.Doc
m.wrl.hmybk.cn/02228.Doc
m.wrl.hmybk.cn/28842.Doc
m.wrl.hmybk.cn/51755.Doc
m.wrl.hmybk.cn/15533.Doc
m.wrl.hmybk.cn/60820.Doc
m.wrk.hmybk.cn/04646.Doc
m.wrk.hmybk.cn/75317.Doc
m.wrk.hmybk.cn/00608.Doc
m.wrk.hmybk.cn/24648.Doc
m.wrk.hmybk.cn/42022.Doc
m.wrk.hmybk.cn/40200.Doc
m.wrk.hmybk.cn/51593.Doc
m.wrk.hmybk.cn/53775.Doc
m.wrk.hmybk.cn/04226.Doc
m.wrk.hmybk.cn/82682.Doc
m.wrk.hmybk.cn/66068.Doc
m.wrk.hmybk.cn/40804.Doc
m.wrk.hmybk.cn/60086.Doc
m.wrk.hmybk.cn/08628.Doc
m.wrk.hmybk.cn/00666.Doc
m.wrk.hmybk.cn/06086.Doc
m.wrk.hmybk.cn/86206.Doc
m.wrk.hmybk.cn/84260.Doc
m.wrk.hmybk.cn/00244.Doc
m.wrk.hmybk.cn/77737.Doc
m.wrj.hmybk.cn/82484.Doc
m.wrj.hmybk.cn/20284.Doc
m.wrj.hmybk.cn/22468.Doc
m.wrj.hmybk.cn/22604.Doc
m.wrj.hmybk.cn/08082.Doc
m.wrj.hmybk.cn/86408.Doc
m.wrj.hmybk.cn/80484.Doc
m.wrj.hmybk.cn/75577.Doc
m.wrj.hmybk.cn/44404.Doc
m.wrj.hmybk.cn/42084.Doc
m.wrj.hmybk.cn/26828.Doc
m.wrj.hmybk.cn/82060.Doc
m.wrj.hmybk.cn/88464.Doc
m.wrj.hmybk.cn/53557.Doc
m.wrj.hmybk.cn/22822.Doc
m.wrj.hmybk.cn/44446.Doc
m.wrj.hmybk.cn/82048.Doc
m.wrj.hmybk.cn/64886.Doc
m.wrj.hmybk.cn/31395.Doc
m.wrj.hmybk.cn/15553.Doc
m.wrh.hmybk.cn/51953.Doc
m.wrh.hmybk.cn/42402.Doc
m.wrh.hmybk.cn/26046.Doc
m.wrh.hmybk.cn/24888.Doc
m.wrh.hmybk.cn/88600.Doc
m.wrh.hmybk.cn/88204.Doc
m.wrh.hmybk.cn/88626.Doc
m.wrh.hmybk.cn/86224.Doc
m.wrh.hmybk.cn/44404.Doc
m.wrh.hmybk.cn/77179.Doc
m.wrh.hmybk.cn/48244.Doc
m.wrh.hmybk.cn/51795.Doc
m.wrh.hmybk.cn/68428.Doc
m.wrh.hmybk.cn/46044.Doc
m.wrh.hmybk.cn/82600.Doc
m.wrh.hmybk.cn/28262.Doc
m.wrh.hmybk.cn/60866.Doc
m.wrh.hmybk.cn/06868.Doc
m.wrh.hmybk.cn/00280.Doc
m.wrh.hmybk.cn/02464.Doc
m.wrg.hmybk.cn/46220.Doc
m.wrg.hmybk.cn/64046.Doc
m.wrg.hmybk.cn/15357.Doc
m.wrg.hmybk.cn/64242.Doc
m.wrg.hmybk.cn/64868.Doc
m.wrg.hmybk.cn/68860.Doc
m.wrg.hmybk.cn/84402.Doc
m.wrg.hmybk.cn/44600.Doc
m.wrg.hmybk.cn/82026.Doc
m.wrg.hmybk.cn/62046.Doc
m.wrg.hmybk.cn/62428.Doc
m.wrg.hmybk.cn/95319.Doc
m.wrg.hmybk.cn/26228.Doc
m.wrg.hmybk.cn/62286.Doc
m.wrg.hmybk.cn/60284.Doc
m.wrg.hmybk.cn/35175.Doc
m.wrg.hmybk.cn/88400.Doc
m.wrg.hmybk.cn/28866.Doc
m.wrg.hmybk.cn/53919.Doc
m.wrg.hmybk.cn/26640.Doc
m.wrf.hmybk.cn/99955.Doc
m.wrf.hmybk.cn/97397.Doc
m.wrf.hmybk.cn/04842.Doc
m.wrf.hmybk.cn/24284.Doc
m.wrf.hmybk.cn/57771.Doc
m.wrf.hmybk.cn/42842.Doc
m.wrf.hmybk.cn/44862.Doc
m.wrf.hmybk.cn/97957.Doc
m.wrf.hmybk.cn/28880.Doc
m.wrf.hmybk.cn/60666.Doc
m.wrf.hmybk.cn/64242.Doc
m.wrf.hmybk.cn/66642.Doc
m.wrf.hmybk.cn/60860.Doc
m.wrf.hmybk.cn/51177.Doc
m.wrf.hmybk.cn/80068.Doc
m.wrf.hmybk.cn/28886.Doc
m.wrf.hmybk.cn/99797.Doc
m.wrf.hmybk.cn/15351.Doc
m.wrf.hmybk.cn/26860.Doc
m.wrf.hmybk.cn/60460.Doc
m.wrd.hmybk.cn/62842.Doc
m.wrd.hmybk.cn/28606.Doc
m.wrd.hmybk.cn/59157.Doc
m.wrd.hmybk.cn/82200.Doc
m.wrd.hmybk.cn/86068.Doc
m.wrd.hmybk.cn/60448.Doc
m.wrd.hmybk.cn/02242.Doc
m.wrd.hmybk.cn/44266.Doc
m.wrd.hmybk.cn/86020.Doc
m.wrd.hmybk.cn/79133.Doc
m.wrd.hmybk.cn/71571.Doc
m.wrd.hmybk.cn/40600.Doc
m.wrd.hmybk.cn/93135.Doc
m.wrd.hmybk.cn/73917.Doc
m.wrd.hmybk.cn/48440.Doc
m.wrd.hmybk.cn/51597.Doc
m.wrd.hmybk.cn/24804.Doc
m.wrd.hmybk.cn/62004.Doc
m.wrd.hmybk.cn/33339.Doc
m.wrd.hmybk.cn/44468.Doc
m.wrs.hmybk.cn/28040.Doc
m.wrs.hmybk.cn/57591.Doc
m.wrs.hmybk.cn/42042.Doc
m.wrs.hmybk.cn/20882.Doc
m.wrs.hmybk.cn/60806.Doc
m.wrs.hmybk.cn/88086.Doc
m.wrs.hmybk.cn/26044.Doc
m.wrs.hmybk.cn/44266.Doc
m.wrs.hmybk.cn/24648.Doc
m.wrs.hmybk.cn/02886.Doc
m.wrs.hmybk.cn/26864.Doc
m.wrs.hmybk.cn/28460.Doc
m.wrs.hmybk.cn/37319.Doc
m.wrs.hmybk.cn/20646.Doc
m.wrs.hmybk.cn/64888.Doc
m.wrs.hmybk.cn/53955.Doc
m.wrs.hmybk.cn/08440.Doc
m.wrs.hmybk.cn/37579.Doc
m.wrs.hmybk.cn/82460.Doc
m.wrs.hmybk.cn/35155.Doc
m.wra.hmybk.cn/77135.Doc
m.wra.hmybk.cn/24680.Doc
m.wra.hmybk.cn/24246.Doc
m.wra.hmybk.cn/48246.Doc
m.wra.hmybk.cn/46020.Doc
m.wra.hmybk.cn/28064.Doc
m.wra.hmybk.cn/28668.Doc
m.wra.hmybk.cn/24288.Doc
m.wra.hmybk.cn/02262.Doc
m.wra.hmybk.cn/39311.Doc
m.wra.hmybk.cn/66468.Doc
m.wra.hmybk.cn/26222.Doc
m.wra.hmybk.cn/33591.Doc
m.wra.hmybk.cn/19539.Doc
m.wra.hmybk.cn/33753.Doc
m.wra.hmybk.cn/26242.Doc
m.wra.hmybk.cn/60642.Doc
m.wra.hmybk.cn/02820.Doc
m.wra.hmybk.cn/02206.Doc
m.wra.hmybk.cn/86840.Doc
m.wrp.hmybk.cn/24266.Doc
m.wrp.hmybk.cn/31511.Doc
m.wrp.hmybk.cn/53515.Doc
m.wrp.hmybk.cn/73511.Doc
m.wrp.hmybk.cn/80048.Doc
m.wrp.hmybk.cn/15399.Doc
m.wrp.hmybk.cn/66628.Doc
m.wrp.hmybk.cn/60640.Doc
m.wrp.hmybk.cn/80244.Doc
m.wrp.hmybk.cn/46428.Doc
m.wrp.hmybk.cn/93331.Doc
m.wrp.hmybk.cn/60644.Doc
m.wrp.hmybk.cn/66000.Doc
m.wrp.hmybk.cn/68622.Doc
m.wrp.hmybk.cn/71171.Doc
m.wrp.hmybk.cn/22224.Doc
m.wrp.hmybk.cn/88462.Doc
m.wrp.hmybk.cn/97773.Doc
m.wrp.hmybk.cn/42080.Doc
m.wrp.hmybk.cn/00840.Doc
m.wro.hmybk.cn/66468.Doc
m.wro.hmybk.cn/55997.Doc
m.wro.hmybk.cn/04204.Doc
m.wro.hmybk.cn/00880.Doc
m.wro.hmybk.cn/88200.Doc
m.wro.hmybk.cn/80620.Doc
m.wro.hmybk.cn/48666.Doc
m.wro.hmybk.cn/48246.Doc
m.wro.hmybk.cn/46600.Doc
m.wro.hmybk.cn/66602.Doc
m.wro.hmybk.cn/04004.Doc
m.wro.hmybk.cn/13915.Doc
m.wro.hmybk.cn/11715.Doc
m.wro.hmybk.cn/62426.Doc
m.wro.hmybk.cn/84606.Doc
m.wro.hmybk.cn/02620.Doc
m.wro.hmybk.cn/35935.Doc
m.wro.hmybk.cn/26668.Doc
m.wro.hmybk.cn/04444.Doc
m.wro.hmybk.cn/08862.Doc
m.wri.hmybk.cn/02828.Doc
m.wri.hmybk.cn/08804.Doc
m.wri.hmybk.cn/64046.Doc
m.wri.hmybk.cn/88882.Doc
m.wri.hmybk.cn/15135.Doc
m.wri.hmybk.cn/40000.Doc
m.wri.hmybk.cn/84066.Doc
m.wri.hmybk.cn/99559.Doc
m.wri.hmybk.cn/60660.Doc
m.wri.hmybk.cn/06006.Doc
m.wri.hmybk.cn/88468.Doc
m.wri.hmybk.cn/22202.Doc
m.wri.hmybk.cn/46680.Doc
m.wri.hmybk.cn/68042.Doc
m.wri.hmybk.cn/88608.Doc
m.wri.hmybk.cn/66884.Doc
m.wri.hmybk.cn/82608.Doc
m.wri.hmybk.cn/33133.Doc
m.wri.hmybk.cn/02246.Doc
m.wri.hmybk.cn/44220.Doc
m.wru.hmybk.cn/08000.Doc
m.wru.hmybk.cn/06666.Doc
m.wru.hmybk.cn/84044.Doc
m.wru.hmybk.cn/60640.Doc
m.wru.hmybk.cn/62046.Doc
m.wru.hmybk.cn/48246.Doc
m.wru.hmybk.cn/64040.Doc
m.wru.hmybk.cn/60268.Doc
m.wru.hmybk.cn/66824.Doc
m.wru.hmybk.cn/00042.Doc
m.wru.hmybk.cn/15157.Doc
m.wru.hmybk.cn/80624.Doc
m.wru.hmybk.cn/04244.Doc
m.wru.hmybk.cn/66604.Doc
m.wru.hmybk.cn/51533.Doc
m.wru.hmybk.cn/46280.Doc
m.wru.hmybk.cn/88426.Doc
m.wru.hmybk.cn/51175.Doc
m.wru.hmybk.cn/40460.Doc
m.wru.hmybk.cn/79339.Doc
