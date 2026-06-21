驮假黄即倒


# Python相关分析 - 完整代码示例
# 相关分析度量变量间的关联强度和方向，是探索性数据分析的核心工具

import numpy as np
import matplotlib.pyplot as plt
from scipy import stats

# 1. Pearson 相关系数 (pearsonr)
# 度量线性相关，要求数据近似正态分布
np.random.seed(42)
n = 100

# 生成线性相关数据
x = np.random.normal(0, 1, n)
# y = 0.7 * x + 0.3 * 噪声（引入 Pearson 相关系数约 0.7）
y_linear = 0.7 * x + 0.3 * np.random.normal(0, 1, n)

pearson_r, pearson_p = stats.pearsonr(x, y_linear)
print("Pearson 相关分析:")
print(f"相关系数 r = {pearson_r:.4f}")
print(f"p-value = {pearson_p:.6f}")
print(f"r² (决定系数) = {pearson_r**2:.4f}（y 的变异中 {pearson_r**2*100:.1f}% 可由 x 解释）")

# Pearson 相关的假设检验：H₀: ρ = 0
alpha = 0.05
print(f"结论: {'存在显著线性相关' if pearson_p < alpha else '无显著线性相关'}")

# 2. Spearman 秩相关系数 (spearmanr)
# 非参数版本，度量单调关系（不要求线性），基于秩
# 生成单调但非线性的数据
y_monotonic = np.exp(x * 0.8 + 0.2 * np.random.normal(0, 1, n))

spearman_r, spearman_p = stats.spearmanr(x, y_monotonic)
print(f"\nSpearman 相关分析 (非线性单调数据):")
print(f"Spearman ρ = {spearman_r:.4f}")
print(f"p-value = {spearman_p:.6f}")
# 对比 Pearson
pearson_r2, _ = stats.pearsonr(x, y_monotonic)
print(f"对比 Pearson r = {pearson_r2:.4f}（Spearman 对非线性单调关系更敏感）")

# 3. Kendall Tau 相关系数 (kendalltau)
# 另一种非参数相关，基于一致对/不一致对的数量，对异常值更稳健
kendall_tau, kendall_p = stats.kendalltau(x, y_linear)
print(f"\nKendall Tau 相关分析:")
print(f"Kendall τ = {kendall_tau:.4f}")
print(f"p-value = {kendall_p:.6f}")
# 对大样本，Tau 的绝对值通常小于 Pearson r
print(f"注意：|τ| < |r| 是正常的（τ 的解释不同）")

# 4. 相关矩阵与热图
# 生成多维数据构建相关矩阵
n_vars = 5
var_names = ['X1', 'X2', 'X3', 'X4', 'X5']

# 构建具有相关结构的数据
mean_vec = np.zeros(n_vars)
# 构造协方差矩阵（人工设定相关模式）
cov_matrix = np.array([
    [1.0, 0.8, 0.3, 0.1, 0.0],
    [0.8, 1.0, 0.4, 0.2, 0.1],
    [0.3, 0.4, 1.0, 0.6, 0.2],
    [0.1, 0.2, 0.6, 1.0, 0.7],
    [0.0, 0.1, 0.2, 0.7, 1.0]
])

# 从多元正态分布生成数据
multivariate_data = np.random.multivariate_normal(mean_vec, cov_matrix, size=200)

# 计算相关矩阵
corr_matrix = np.corrcoef(multivariate_data.T)
print(f"\n相关矩阵 (5 个变量):")
for i in range(n_vars):
    row_str = '  '.join([f"{corr_matrix[i,j]:7.4f}" for j in range(n_vars)])
    print(f"  {var_names[i]}: {row_str}")

# 5. 偏相关分析 (Partial Correlation)
# 控制其他变量后，两个变量间的相关关系
# 使用协方差矩阵的逆矩阵（精度矩阵）计算偏相关
precision_matrix = np.linalg.inv(cov_matrix)
# 偏相关系数 = -P_ij / sqrt(P_ii * P_jj)
partial_corr = np.zeros_like(corr_matrix)
for i in range(n_vars):
    for j in range(n_vars):
        if i != j:
            partial_corr[i, j] = -precision_matrix[i, j] / np.sqrt(
                precision_matrix[i, i] * precision_matrix[j, j]
            )

print(f"\n偏相关矩阵 (控制其他变量):")
for i in range(n_vars):
    row_str = '  '.join([f"{partial_corr[i,j]:7.4f}" for j in range(n_vars)])
    print(f"  {var_names[i]}: {row_str}")

# 对比：偏相关通常比普通相关系数更小（排除了混杂因素的影响）
print(f"\nX1-X2 普通相关系数: {corr_matrix[0,1]:.4f}")
print(f"X1-X2 偏相关系数:   {partial_corr[0,1]:.4f}")

# 6. 置信区间计算 (Fisher Z 变换)
# 为 Pearson 相关系数构建置信区间
def pearson_ci(r, n, conf_level=0.95):
    """使用 Fisher Z 变换计算相关系数的置信区间"""
    z = np.arctanh(r)  # Fisher Z 变换
    se = 1 / np.sqrt(n - 3)  # 标准误
    z_alpha = stats.norm.ppf((1 + conf_level) / 2)
    z_lower = z - z_alpha * se
    z_upper = z + z_alpha * se
    # 逆变换回相关系数
    r_lower = np.tanh(z_lower)
    r_upper = np.tanh(z_upper)
    return r_lower, r_upper

ci_lower, ci_upper = pearson_ci(pearson_r, n)
print(f"\nPearson r 的 95% 置信区间: [{ci_lower:.4f}, {ci_upper:.4f}]")

# 7. 相关矩阵的统计检验
# 检验相关矩阵是否为单位矩阵（所有变量独立）
def test_correlation_matrix(corr_mat, n):
    """使用 Bartlett 球形检验判断相关矩阵是否为单位矩阵"""
    p = corr_mat.shape[0]
    det_corr = np.linalg.det(corr_mat)
    # Bartlett 统计量
    chi2_stat = -((n - 1) - (2*p + 5) / 6) * np.log(det_corr)
    df = p * (p - 1) // 2
    p_val = 1 - stats.chi2.cdf(chi2_stat, df)
    return chi2_stat, p_val, df

bartlett_chi2, bartlett_p, bartlett_df = test_correlation_matrix(corr_matrix, 200)
print(f"\nBartlett 球形检验:")
print(f"χ² = {bartlett_chi2:.2f}, df = {bartlett_df}, p-value = {bartlett_p:.6f}")
print(f"结论: {'变量间存在显著相关' if bartlett_p < alpha else '变量可能独立'}")

# 8. 异常值对 Pearson 相关的影响
# 展示单个异常值如何扭曲相关系数
x_clean = np.random.normal(0, 1, 50)
y_clean = x_clean + 0.3 * np.random.normal(0, 1, 50)
r_clean, _ = stats.pearsonr(x_clean, y_clean)

# 添加一个极端异常值
x_outlier = np.append(x_clean, [10])
y_outlier = np.append(y_clean, [-10])
r_outlier, _ = stats.pearsonr(x_outlier, y_outlier)

print(f"\n异常值的影响:")
print(f"无异常值时 r = {r_clean:.4f}")
print(f"有异常值时 r = {r_outlier:.4f}（Spearman 方法对此更稳健）")
r_spearman_out, _ = stats.spearmanr(x_outlier, y_outlier)
print(f"Spearman (有异常值) ρ = {r_spearman_out:.4f}")

# 9. 时间序列自相关
# 计算滞后自相关
time_series = np.sin(np.linspace(0, 4*np.pi, 100)) + 0.3 * np.random.randn(100)
# 计算滞后 1 的自相关
autocorr_lag1 = stats.pearsonr(time_series[:-1], time_series[1:])[0]
print(f"\n时间序列滞后 1 自相关: r = {autocorr_lag1:.4f}")

print("\n相关分析总结：Pearson 度量线性、Spearman 度量单调")
print("偏相关控制混杂因素，置信区间提供更丰富的信息")

旧餐让嚷重患眯菲现故凸读狼窒收

m.esa.mglwx.cn/88608.Doc
m.esa.mglwx.cn/11317.Doc
m.esa.mglwx.cn/15199.Doc
m.esa.mglwx.cn/97397.Doc
m.esa.mglwx.cn/93353.Doc
m.esa.mglwx.cn/44862.Doc
m.esa.mglwx.cn/60244.Doc
m.esa.mglwx.cn/22200.Doc
m.esa.mglwx.cn/97131.Doc
m.esa.mglwx.cn/24228.Doc
m.esa.mglwx.cn/37371.Doc
m.esa.mglwx.cn/84828.Doc
m.esa.mglwx.cn/44024.Doc
m.esa.mglwx.cn/39359.Doc
m.esa.mglwx.cn/39971.Doc
m.esp.mglwx.cn/31191.Doc
m.esp.mglwx.cn/37737.Doc
m.esp.mglwx.cn/44064.Doc
m.esp.mglwx.cn/66420.Doc
m.esp.mglwx.cn/08482.Doc
m.esp.mglwx.cn/22006.Doc
m.esp.mglwx.cn/15711.Doc
m.esp.mglwx.cn/15919.Doc
m.esp.mglwx.cn/26062.Doc
m.esp.mglwx.cn/46846.Doc
m.esp.mglwx.cn/19195.Doc
m.esp.mglwx.cn/71551.Doc
m.esp.mglwx.cn/99175.Doc
m.esp.mglwx.cn/13159.Doc
m.esp.mglwx.cn/28602.Doc
m.esp.mglwx.cn/75131.Doc
m.esp.mglwx.cn/31395.Doc
m.esp.mglwx.cn/57511.Doc
m.esp.mglwx.cn/35359.Doc
m.esp.mglwx.cn/77597.Doc
m.eso.mglwx.cn/31517.Doc
m.eso.mglwx.cn/46062.Doc
m.eso.mglwx.cn/39515.Doc
m.eso.mglwx.cn/06402.Doc
m.eso.mglwx.cn/31315.Doc
m.eso.mglwx.cn/33751.Doc
m.eso.mglwx.cn/24860.Doc
m.eso.mglwx.cn/20226.Doc
m.eso.mglwx.cn/86466.Doc
m.eso.mglwx.cn/68462.Doc
m.eso.mglwx.cn/64260.Doc
m.eso.mglwx.cn/08422.Doc
m.eso.mglwx.cn/28800.Doc
m.eso.mglwx.cn/62284.Doc
m.eso.mglwx.cn/55595.Doc
m.eso.mglwx.cn/60488.Doc
m.eso.mglwx.cn/11735.Doc
m.eso.mglwx.cn/97753.Doc
m.eso.mglwx.cn/22264.Doc
m.eso.mglwx.cn/99133.Doc
m.esi.mglwx.cn/93377.Doc
m.esi.mglwx.cn/95993.Doc
m.esi.mglwx.cn/37375.Doc
m.esi.mglwx.cn/22600.Doc
m.esi.mglwx.cn/42268.Doc
m.esi.mglwx.cn/35533.Doc
m.esi.mglwx.cn/08044.Doc
m.esi.mglwx.cn/04266.Doc
m.esi.mglwx.cn/62284.Doc
m.esi.mglwx.cn/99733.Doc
m.esi.mglwx.cn/39577.Doc
m.esi.mglwx.cn/06080.Doc
m.esi.mglwx.cn/88240.Doc
m.esi.mglwx.cn/68842.Doc
m.esi.mglwx.cn/93377.Doc
m.esi.mglwx.cn/42264.Doc
m.esi.mglwx.cn/71537.Doc
m.esi.mglwx.cn/77711.Doc
m.esi.mglwx.cn/13155.Doc
m.esi.mglwx.cn/39131.Doc
m.esu.mglwx.cn/08406.Doc
m.esu.mglwx.cn/80420.Doc
m.esu.mglwx.cn/53535.Doc
m.esu.mglwx.cn/57199.Doc
m.esu.mglwx.cn/46866.Doc
m.esu.mglwx.cn/37715.Doc
m.esu.mglwx.cn/11177.Doc
m.esu.mglwx.cn/35177.Doc
m.esu.mglwx.cn/15153.Doc
m.esu.mglwx.cn/99919.Doc
m.esu.mglwx.cn/91197.Doc
m.esu.mglwx.cn/91733.Doc
m.esu.mglwx.cn/91773.Doc
m.esu.mglwx.cn/88828.Doc
m.esu.mglwx.cn/22848.Doc
m.esu.mglwx.cn/55593.Doc
m.esu.mglwx.cn/53195.Doc
m.esu.mglwx.cn/60000.Doc
m.esu.mglwx.cn/19799.Doc
m.esu.mglwx.cn/37573.Doc
m.esy.mglwx.cn/60440.Doc
m.esy.mglwx.cn/93797.Doc
m.esy.mglwx.cn/17953.Doc
m.esy.mglwx.cn/48222.Doc
m.esy.mglwx.cn/82428.Doc
m.esy.mglwx.cn/95575.Doc
m.esy.mglwx.cn/64446.Doc
m.esy.mglwx.cn/77735.Doc
m.esy.mglwx.cn/66666.Doc
m.esy.mglwx.cn/60802.Doc
m.esy.mglwx.cn/37517.Doc
m.esy.mglwx.cn/57577.Doc
m.esy.mglwx.cn/79597.Doc
m.esy.mglwx.cn/57153.Doc
m.esy.mglwx.cn/20220.Doc
m.esy.mglwx.cn/59597.Doc
m.esy.mglwx.cn/91597.Doc
m.esy.mglwx.cn/35335.Doc
m.esy.mglwx.cn/28800.Doc
m.esy.mglwx.cn/55733.Doc
m.est.mglwx.cn/13351.Doc
m.est.mglwx.cn/97795.Doc
m.est.mglwx.cn/51399.Doc
m.est.mglwx.cn/22648.Doc
m.est.mglwx.cn/57395.Doc
m.est.mglwx.cn/91911.Doc
m.est.mglwx.cn/28648.Doc
m.est.mglwx.cn/60682.Doc
m.est.mglwx.cn/46262.Doc
m.est.mglwx.cn/39999.Doc
m.est.mglwx.cn/00602.Doc
m.est.mglwx.cn/73175.Doc
m.est.mglwx.cn/60802.Doc
m.est.mglwx.cn/26428.Doc
m.est.mglwx.cn/93131.Doc
m.est.mglwx.cn/5.Doc
m.est.mglwx.cn/31791.Doc
m.est.mglwx.cn/59355.Doc
m.est.mglwx.cn/68884.Doc
m.est.mglwx.cn/59737.Doc
m.esr.mglwx.cn/26282.Doc
m.esr.mglwx.cn/93373.Doc
m.esr.mglwx.cn/73339.Doc
m.esr.mglwx.cn/40480.Doc
m.esr.mglwx.cn/15137.Doc
m.esr.mglwx.cn/37751.Doc
m.esr.mglwx.cn/80446.Doc
m.esr.mglwx.cn/35531.Doc
m.esr.mglwx.cn/99715.Doc
m.esr.mglwx.cn/73193.Doc
m.esr.mglwx.cn/53319.Doc
m.esr.mglwx.cn/17979.Doc
m.esr.mglwx.cn/11777.Doc
m.esr.mglwx.cn/53997.Doc
m.esr.mglwx.cn/71975.Doc
m.esr.mglwx.cn/31379.Doc
m.esr.mglwx.cn/35533.Doc
m.esr.mglwx.cn/93137.Doc
m.esr.mglwx.cn/17531.Doc
m.esr.mglwx.cn/71311.Doc
m.ese.mglwx.cn/33979.Doc
m.ese.mglwx.cn/79711.Doc
m.ese.mglwx.cn/59773.Doc
m.ese.mglwx.cn/06288.Doc
m.ese.mglwx.cn/75133.Doc
m.ese.mglwx.cn/04826.Doc
m.ese.mglwx.cn/57759.Doc
m.ese.mglwx.cn/19933.Doc
m.ese.mglwx.cn/88422.Doc
m.ese.mglwx.cn/97919.Doc
m.ese.mglwx.cn/44648.Doc
m.ese.mglwx.cn/06446.Doc
m.ese.mglwx.cn/93979.Doc
m.ese.mglwx.cn/08020.Doc
m.ese.mglwx.cn/08042.Doc
m.ese.mglwx.cn/15939.Doc
m.ese.mglwx.cn/11951.Doc
m.ese.mglwx.cn/57311.Doc
m.ese.mglwx.cn/35735.Doc
m.ese.mglwx.cn/97351.Doc
m.esw.mglwx.cn/71557.Doc
m.esw.mglwx.cn/37171.Doc
m.esw.mglwx.cn/04602.Doc
m.esw.mglwx.cn/60662.Doc
m.esw.mglwx.cn/66086.Doc
m.esw.mglwx.cn/04448.Doc
m.esw.mglwx.cn/00802.Doc
m.esw.mglwx.cn/97375.Doc
m.esw.mglwx.cn/15359.Doc
m.esw.mglwx.cn/13779.Doc
m.esw.mglwx.cn/15937.Doc
m.esw.mglwx.cn/99335.Doc
m.esw.mglwx.cn/37573.Doc
m.esw.mglwx.cn/19911.Doc
m.esw.mglwx.cn/13791.Doc
m.esw.mglwx.cn/71751.Doc
m.esw.mglwx.cn/33917.Doc
m.esw.mglwx.cn/31751.Doc
m.esw.mglwx.cn/53957.Doc
m.esw.mglwx.cn/37997.Doc
m.esq.mglwx.cn/93111.Doc
m.esq.mglwx.cn/71539.Doc
m.esq.mglwx.cn/39799.Doc
m.esq.mglwx.cn/40224.Doc
m.esq.mglwx.cn/82222.Doc
m.esq.mglwx.cn/26006.Doc
m.esq.mglwx.cn/84028.Doc
m.esq.mglwx.cn/17191.Doc
m.esq.mglwx.cn/31179.Doc
m.esq.mglwx.cn/15115.Doc
m.esq.mglwx.cn/40628.Doc
m.esq.mglwx.cn/24682.Doc
m.esq.mglwx.cn/97995.Doc
m.esq.mglwx.cn/75777.Doc
m.esq.mglwx.cn/11337.Doc
m.esq.mglwx.cn/51359.Doc
m.esq.mglwx.cn/73733.Doc
m.esq.mglwx.cn/46866.Doc
m.esq.mglwx.cn/22264.Doc
m.esq.mglwx.cn/73755.Doc
m.eam.mglwx.cn/91957.Doc
m.eam.mglwx.cn/73177.Doc
m.eam.mglwx.cn/93175.Doc
m.eam.mglwx.cn/53715.Doc
m.eam.mglwx.cn/71977.Doc
m.eam.mglwx.cn/28428.Doc
m.eam.mglwx.cn/95777.Doc
m.eam.mglwx.cn/73191.Doc
m.eam.mglwx.cn/11391.Doc
m.eam.mglwx.cn/17113.Doc
m.eam.mglwx.cn/77117.Doc
m.eam.mglwx.cn/99797.Doc
m.eam.mglwx.cn/55577.Doc
m.eam.mglwx.cn/13993.Doc
m.eam.mglwx.cn/48448.Doc
m.eam.mglwx.cn/55939.Doc
m.eam.mglwx.cn/00062.Doc
m.eam.mglwx.cn/11133.Doc
m.eam.mglwx.cn/40848.Doc
m.eam.mglwx.cn/60228.Doc
m.ean.mglwx.cn/33715.Doc
m.ean.mglwx.cn/55917.Doc
m.ean.mglwx.cn/31559.Doc
m.ean.mglwx.cn/97793.Doc
m.ean.mglwx.cn/31779.Doc
m.ean.mglwx.cn/93553.Doc
m.ean.mglwx.cn/13395.Doc
m.ean.mglwx.cn/77717.Doc
m.ean.mglwx.cn/40202.Doc
m.ean.mglwx.cn/80440.Doc
m.ean.mglwx.cn/73731.Doc
m.ean.mglwx.cn/68442.Doc
m.ean.mglwx.cn/75393.Doc
m.ean.mglwx.cn/79779.Doc
m.ean.mglwx.cn/77577.Doc
m.ean.mglwx.cn/59593.Doc
m.ean.mglwx.cn/97779.Doc
m.ean.mglwx.cn/37931.Doc
m.ean.mglwx.cn/35337.Doc
m.ean.mglwx.cn/71539.Doc
m.eab.mglwx.cn/77913.Doc
m.eab.mglwx.cn/73513.Doc
m.eab.mglwx.cn/11537.Doc
m.eab.mglwx.cn/95175.Doc
m.eab.mglwx.cn/91199.Doc
m.eab.mglwx.cn/17311.Doc
m.eab.mglwx.cn/13517.Doc
m.eab.mglwx.cn/91139.Doc
m.eab.mglwx.cn/55137.Doc
m.eab.mglwx.cn/75515.Doc
m.eab.mglwx.cn/51791.Doc
m.eab.mglwx.cn/19917.Doc
m.eab.mglwx.cn/91719.Doc
m.eab.mglwx.cn/53133.Doc
m.eab.mglwx.cn/79117.Doc
m.eab.mglwx.cn/04240.Doc
m.eab.mglwx.cn/15537.Doc
m.eab.mglwx.cn/37993.Doc
m.eab.mglwx.cn/35959.Doc
m.eab.mglwx.cn/55513.Doc
m.eav.mglwx.cn/71755.Doc
m.eav.mglwx.cn/26802.Doc
m.eav.mglwx.cn/77513.Doc
m.eav.mglwx.cn/48268.Doc
m.eav.mglwx.cn/75939.Doc
m.eav.mglwx.cn/91555.Doc
m.eav.mglwx.cn/55513.Doc
m.eav.mglwx.cn/57313.Doc
m.eav.mglwx.cn/48444.Doc
m.eav.mglwx.cn/17375.Doc
m.eav.mglwx.cn/59115.Doc
m.eav.mglwx.cn/11959.Doc
m.eav.mglwx.cn/33551.Doc
m.eav.mglwx.cn/46086.Doc
m.eav.mglwx.cn/57379.Doc
m.eav.mglwx.cn/73311.Doc
m.eav.mglwx.cn/11397.Doc
m.eav.mglwx.cn/71797.Doc
m.eav.mglwx.cn/62224.Doc
m.eav.mglwx.cn/42022.Doc
m.eac.mglwx.cn/77797.Doc
m.eac.mglwx.cn/75953.Doc
m.eac.mglwx.cn/51731.Doc
m.eac.mglwx.cn/31597.Doc
m.eac.mglwx.cn/91993.Doc
m.eac.mglwx.cn/95959.Doc
m.eac.mglwx.cn/97353.Doc
m.eac.mglwx.cn/08006.Doc
m.eac.mglwx.cn/39591.Doc
m.eac.mglwx.cn/15353.Doc
m.eac.mglwx.cn/59577.Doc
m.eac.mglwx.cn/33517.Doc
m.eac.mglwx.cn/37719.Doc
m.eac.mglwx.cn/02866.Doc
m.eac.mglwx.cn/88848.Doc
m.eac.mglwx.cn/64284.Doc
m.eac.mglwx.cn/99791.Doc
m.eac.mglwx.cn/59159.Doc
m.eac.mglwx.cn/57593.Doc
m.eac.mglwx.cn/46426.Doc
m.eax.mglwx.cn/26862.Doc
m.eax.mglwx.cn/99791.Doc
m.eax.mglwx.cn/75595.Doc
m.eax.mglwx.cn/73333.Doc
m.eax.mglwx.cn/22464.Doc
m.eax.mglwx.cn/48846.Doc
m.eax.mglwx.cn/35597.Doc
m.eax.mglwx.cn/11913.Doc
m.eax.mglwx.cn/51337.Doc
m.eax.mglwx.cn/11151.Doc
m.eax.mglwx.cn/79191.Doc
m.eax.mglwx.cn/95773.Doc
m.eax.mglwx.cn/71157.Doc
m.eax.mglwx.cn/31519.Doc
m.eax.mglwx.cn/35593.Doc
m.eax.mglwx.cn/99175.Doc
m.eax.mglwx.cn/51777.Doc
m.eax.mglwx.cn/48446.Doc
m.eax.mglwx.cn/44682.Doc
m.eax.mglwx.cn/97139.Doc
m.eaz.mglwx.cn/19537.Doc
m.eaz.mglwx.cn/11939.Doc
m.eaz.mglwx.cn/08248.Doc
m.eaz.mglwx.cn/28440.Doc
m.eaz.mglwx.cn/66080.Doc
m.eaz.mglwx.cn/84086.Doc
m.eaz.mglwx.cn/77713.Doc
m.eaz.mglwx.cn/82600.Doc
m.eaz.mglwx.cn/71775.Doc
m.eaz.mglwx.cn/19977.Doc
m.eaz.mglwx.cn/79511.Doc
m.eaz.mglwx.cn/57533.Doc
m.eaz.mglwx.cn/71793.Doc
m.eaz.mglwx.cn/35117.Doc
m.eaz.mglwx.cn/04828.Doc
m.eaz.mglwx.cn/46464.Doc
m.eaz.mglwx.cn/59391.Doc
m.eaz.mglwx.cn/19919.Doc
m.eaz.mglwx.cn/17579.Doc
m.eaz.mglwx.cn/19597.Doc
m.eal.mglwx.cn/59735.Doc
m.eal.mglwx.cn/48244.Doc
m.eal.mglwx.cn/99713.Doc
m.eal.mglwx.cn/35393.Doc
m.eal.mglwx.cn/46686.Doc
m.eal.mglwx.cn/53317.Doc
m.eal.mglwx.cn/00806.Doc
m.eal.mglwx.cn/08206.Doc
m.eal.mglwx.cn/17555.Doc
m.eal.mglwx.cn/53331.Doc
m.eal.mglwx.cn/00884.Doc
m.eal.mglwx.cn/26244.Doc
m.eal.mglwx.cn/84600.Doc
m.eal.mglwx.cn/75357.Doc
m.eal.mglwx.cn/48884.Doc
m.eal.mglwx.cn/75555.Doc
m.eal.mglwx.cn/46000.Doc
m.eal.mglwx.cn/26880.Doc
m.eal.mglwx.cn/91591.Doc
m.eal.mglwx.cn/24868.Doc
m.eak.mglwx.cn/26024.Doc
m.eak.mglwx.cn/77379.Doc
m.eak.mglwx.cn/55735.Doc
m.eak.mglwx.cn/00266.Doc
m.eak.mglwx.cn/86008.Doc
m.eak.mglwx.cn/99351.Doc
m.eak.mglwx.cn/51171.Doc
m.eak.mglwx.cn/31979.Doc
m.eak.mglwx.cn/82448.Doc
m.eak.mglwx.cn/00602.Doc
m.eak.mglwx.cn/06864.Doc
m.eak.mglwx.cn/88864.Doc
m.eak.mglwx.cn/04026.Doc
m.eak.mglwx.cn/42448.Doc
m.eak.mglwx.cn/24042.Doc
m.eak.mglwx.cn/22866.Doc
m.eak.mglwx.cn/73197.Doc
m.eak.mglwx.cn/73737.Doc
m.eak.mglwx.cn/00288.Doc
m.eak.mglwx.cn/40662.Doc
