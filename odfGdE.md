尘医继翰竟


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

蚕诶汗吭臼厣读旧乌谱送怯纶扇吃

sfu.fffbf.cn/460288.Doc
sfu.fffbf.cn/644806.Doc
sfu.fffbf.cn/006660.Doc
sfu.fffbf.cn/624288.Doc
sfu.fffbf.cn/008666.Doc
sfu.fffbf.cn/028266.Doc
sfu.fffbf.cn/284608.Doc
sfu.fffbf.cn/602840.Doc
sfy.fffbf.cn/288822.Doc
sfy.fffbf.cn/640046.Doc
sfy.fffbf.cn/228062.Doc
sfy.fffbf.cn/571739.Doc
sfy.fffbf.cn/460820.Doc
sfy.fffbf.cn/602888.Doc
sfy.fffbf.cn/066220.Doc
sfy.fffbf.cn/862604.Doc
sfy.fffbf.cn/286020.Doc
sfy.fffbf.cn/228260.Doc
sft.fffbf.cn/800082.Doc
sft.fffbf.cn/240428.Doc
sft.fffbf.cn/800206.Doc
sft.fffbf.cn/622060.Doc
sft.fffbf.cn/353579.Doc
sft.fffbf.cn/266026.Doc
sft.fffbf.cn/268042.Doc
sft.fffbf.cn/646888.Doc
sft.fffbf.cn/080422.Doc
sft.fffbf.cn/937331.Doc
sfr.fffbf.cn/822468.Doc
sfr.fffbf.cn/008466.Doc
sfr.fffbf.cn/828600.Doc
sfr.fffbf.cn/288208.Doc
sfr.fffbf.cn/622486.Doc
sfr.fffbf.cn/266462.Doc
sfr.fffbf.cn/682460.Doc
sfr.fffbf.cn/288624.Doc
sfr.fffbf.cn/824682.Doc
sfr.fffbf.cn/620046.Doc
sfe.fffbf.cn/460426.Doc
sfe.fffbf.cn/828202.Doc
sfe.fffbf.cn/206466.Doc
sfe.fffbf.cn/680240.Doc
sfe.fffbf.cn/088442.Doc
sfe.fffbf.cn/042642.Doc
sfe.fffbf.cn/193773.Doc
sfe.fffbf.cn/402048.Doc
sfe.fffbf.cn/086206.Doc
sfe.fffbf.cn/664066.Doc
sfw.fffbf.cn/680022.Doc
sfw.fffbf.cn/066888.Doc
sfw.fffbf.cn/204482.Doc
sfw.fffbf.cn/806424.Doc
sfw.fffbf.cn/135617.Doc
sfw.fffbf.cn/288216.Doc
sfw.fffbf.cn/610602.Doc
sfw.fffbf.cn/028606.Doc
sfw.fffbf.cn/978699.Doc
sfw.fffbf.cn/029564.Doc
sfq.fffbf.cn/958748.Doc
sfq.fffbf.cn/217730.Doc
sfq.fffbf.cn/550685.Doc
sfq.fffbf.cn/737544.Doc
sfq.fffbf.cn/966482.Doc
sfq.fffbf.cn/066319.Doc
sfq.fffbf.cn/515290.Doc
sfq.fffbf.cn/190973.Doc
sfq.fffbf.cn/679498.Doc
sfq.fffbf.cn/421024.Doc
sdm.fffbf.cn/525586.Doc
sdm.fffbf.cn/620861.Doc
sdm.fffbf.cn/575023.Doc
sdm.fffbf.cn/260862.Doc
sdm.fffbf.cn/640717.Doc
sdm.fffbf.cn/435056.Doc
sdm.fffbf.cn/402160.Doc
sdm.fffbf.cn/324168.Doc
sdm.fffbf.cn/371999.Doc
sdm.fffbf.cn/884480.Doc
sdn.fffbf.cn/868244.Doc
sdn.fffbf.cn/604444.Doc
sdn.fffbf.cn/220064.Doc
sdn.fffbf.cn/480460.Doc
sdn.fffbf.cn/880886.Doc
sdn.fffbf.cn/646684.Doc
sdn.fffbf.cn/826248.Doc
sdn.fffbf.cn/440022.Doc
sdn.fffbf.cn/480042.Doc
sdn.fffbf.cn/868868.Doc
sdb.fffbf.cn/440224.Doc
sdb.fffbf.cn/008046.Doc
sdb.fffbf.cn/080082.Doc
sdb.fffbf.cn/535177.Doc
sdb.fffbf.cn/262022.Doc
sdb.fffbf.cn/806426.Doc
sdb.fffbf.cn/284042.Doc
sdb.fffbf.cn/664066.Doc
sdb.fffbf.cn/846686.Doc
sdb.fffbf.cn/282640.Doc
sdv.fffbf.cn/622400.Doc
sdv.fffbf.cn/202846.Doc
sdv.fffbf.cn/280420.Doc
sdv.fffbf.cn/262240.Doc
sdv.fffbf.cn/068800.Doc
sdv.fffbf.cn/244084.Doc
sdv.fffbf.cn/048206.Doc
sdv.fffbf.cn/404800.Doc
sdv.fffbf.cn/228066.Doc
sdv.fffbf.cn/822228.Doc
sdc.fffbf.cn/579993.Doc
sdc.fffbf.cn/668282.Doc
sdc.fffbf.cn/175511.Doc
sdc.fffbf.cn/040662.Doc
sdc.fffbf.cn/040202.Doc
sdc.fffbf.cn/373913.Doc
sdc.fffbf.cn/066680.Doc
sdc.fffbf.cn/604028.Doc
sdc.fffbf.cn/884204.Doc
sdc.fffbf.cn/824224.Doc
sdx.jouwir.cn/226682.Doc
sdx.jouwir.cn/666224.Doc
sdx.jouwir.cn/806666.Doc
sdx.jouwir.cn/820820.Doc
sdx.jouwir.cn/008226.Doc
sdx.jouwir.cn/866026.Doc
sdx.jouwir.cn/339391.Doc
sdx.jouwir.cn/648864.Doc
sdx.jouwir.cn/488642.Doc
sdx.jouwir.cn/268400.Doc
sdz.jouwir.cn/842662.Doc
sdz.jouwir.cn/844662.Doc
sdz.jouwir.cn/860428.Doc
sdz.jouwir.cn/608882.Doc
sdz.jouwir.cn/248642.Doc
sdz.jouwir.cn/080024.Doc
sdz.jouwir.cn/008482.Doc
sdz.jouwir.cn/242604.Doc
sdz.jouwir.cn/828406.Doc
sdz.jouwir.cn/408206.Doc
sdl.jouwir.cn/222882.Doc
sdl.jouwir.cn/042024.Doc
sdl.jouwir.cn/206086.Doc
sdl.jouwir.cn/488226.Doc
sdl.jouwir.cn/806806.Doc
sdl.jouwir.cn/755775.Doc
sdl.jouwir.cn/680848.Doc
sdl.jouwir.cn/440024.Doc
sdl.jouwir.cn/028288.Doc
sdl.jouwir.cn/684002.Doc
sdk.jouwir.cn/682664.Doc
sdk.jouwir.cn/802608.Doc
sdk.jouwir.cn/080004.Doc
sdk.jouwir.cn/644448.Doc
sdk.jouwir.cn/040228.Doc
sdk.jouwir.cn/628844.Doc
sdk.jouwir.cn/622426.Doc
sdk.jouwir.cn/260062.Doc
sdk.jouwir.cn/048428.Doc
sdk.jouwir.cn/004842.Doc
sdj.jouwir.cn/844664.Doc
sdj.jouwir.cn/446646.Doc
sdj.jouwir.cn/202488.Doc
sdj.jouwir.cn/626828.Doc
sdj.jouwir.cn/008428.Doc
sdj.jouwir.cn/428606.Doc
sdj.jouwir.cn/044402.Doc
sdj.jouwir.cn/226800.Doc
sdj.jouwir.cn/000046.Doc
sdj.jouwir.cn/602224.Doc
sdh.jouwir.cn/846648.Doc
sdh.jouwir.cn/640628.Doc
sdh.jouwir.cn/206286.Doc
sdh.jouwir.cn/486644.Doc
sdh.jouwir.cn/840424.Doc
sdh.jouwir.cn/648648.Doc
sdh.jouwir.cn/244046.Doc
sdh.jouwir.cn/440406.Doc
sdh.jouwir.cn/846842.Doc
sdh.jouwir.cn/222420.Doc
sdg.jouwir.cn/266260.Doc
sdg.jouwir.cn/062682.Doc
sdg.jouwir.cn/680860.Doc
sdg.jouwir.cn/608084.Doc
sdg.jouwir.cn/442024.Doc
sdg.jouwir.cn/282484.Doc
sdg.jouwir.cn/642206.Doc
sdg.jouwir.cn/248804.Doc
sdg.jouwir.cn/888446.Doc
sdg.jouwir.cn/024428.Doc
sdf.jouwir.cn/440088.Doc
sdf.jouwir.cn/420224.Doc
sdf.jouwir.cn/022228.Doc
sdf.jouwir.cn/622064.Doc
sdf.jouwir.cn/860620.Doc
sdf.jouwir.cn/719597.Doc
sdf.jouwir.cn/624200.Doc
sdf.jouwir.cn/602804.Doc
sdf.jouwir.cn/488400.Doc
sdf.jouwir.cn/628668.Doc
sdd.jouwir.cn/028000.Doc
sdd.jouwir.cn/688260.Doc
sdd.jouwir.cn/646808.Doc
sdd.jouwir.cn/606684.Doc
sdd.jouwir.cn/046860.Doc
sdd.jouwir.cn/204286.Doc
sdd.jouwir.cn/628884.Doc
sdd.jouwir.cn/006662.Doc
sdd.jouwir.cn/915579.Doc
sdd.jouwir.cn/044260.Doc
sds.jouwir.cn/620426.Doc
sds.jouwir.cn/828286.Doc
sds.jouwir.cn/624688.Doc
sds.jouwir.cn/460080.Doc
sds.jouwir.cn/468266.Doc
sds.jouwir.cn/626284.Doc
sds.jouwir.cn/086866.Doc
sds.jouwir.cn/682244.Doc
sds.jouwir.cn/040886.Doc
sds.jouwir.cn/068244.Doc
sda.jouwir.cn/220266.Doc
sda.jouwir.cn/317991.Doc
sda.jouwir.cn/464404.Doc
sda.jouwir.cn/046400.Doc
sda.jouwir.cn/482284.Doc
sda.jouwir.cn/286824.Doc
sda.jouwir.cn/882824.Doc
sda.jouwir.cn/006664.Doc
sda.jouwir.cn/846000.Doc
sda.jouwir.cn/266686.Doc
sdp.jouwir.cn/484226.Doc
sdp.jouwir.cn/428082.Doc
sdp.jouwir.cn/202642.Doc
sdp.jouwir.cn/488662.Doc
sdp.jouwir.cn/628864.Doc
sdp.jouwir.cn/680242.Doc
sdp.jouwir.cn/600086.Doc
sdp.jouwir.cn/020620.Doc
sdp.jouwir.cn/260802.Doc
sdp.jouwir.cn/224842.Doc
sdo.jouwir.cn/440000.Doc
sdo.jouwir.cn/222022.Doc
sdo.jouwir.cn/882006.Doc
sdo.jouwir.cn/996674.Doc
sdo.jouwir.cn/664624.Doc
sdo.jouwir.cn/284822.Doc
sdo.jouwir.cn/084080.Doc
sdo.jouwir.cn/880604.Doc
sdo.jouwir.cn/905385.Doc
sdo.jouwir.cn/804268.Doc
sdi.jouwir.cn/626204.Doc
sdi.jouwir.cn/592589.Doc
sdi.jouwir.cn/848200.Doc
sdi.jouwir.cn/422280.Doc
sdi.jouwir.cn/953151.Doc
sdi.jouwir.cn/260008.Doc
sdi.jouwir.cn/808668.Doc
sdi.jouwir.cn/626464.Doc
sdi.jouwir.cn/268442.Doc
sdi.jouwir.cn/020244.Doc
sdu.jouwir.cn/846420.Doc
sdu.jouwir.cn/262046.Doc
sdu.jouwir.cn/973939.Doc
sdu.jouwir.cn/242402.Doc
sdu.jouwir.cn/844428.Doc
sdu.jouwir.cn/820400.Doc
sdu.jouwir.cn/208682.Doc
sdu.jouwir.cn/040002.Doc
sdu.jouwir.cn/139513.Doc
sdu.jouwir.cn/882066.Doc
sdy.jouwir.cn/086048.Doc
sdy.jouwir.cn/088404.Doc
sdy.jouwir.cn/688824.Doc
sdy.jouwir.cn/228248.Doc
sdy.jouwir.cn/822428.Doc
sdy.jouwir.cn/426406.Doc
sdy.jouwir.cn/026420.Doc
sdy.jouwir.cn/664662.Doc
sdy.jouwir.cn/004000.Doc
sdy.jouwir.cn/955737.Doc
sdt.jouwir.cn/602660.Doc
sdt.jouwir.cn/606206.Doc
sdt.jouwir.cn/084046.Doc
sdt.jouwir.cn/062422.Doc
sdt.jouwir.cn/662284.Doc
sdt.jouwir.cn/462008.Doc
sdt.jouwir.cn/844642.Doc
sdt.jouwir.cn/824240.Doc
sdt.jouwir.cn/680484.Doc
sdt.jouwir.cn/420482.Doc
sdr.jouwir.cn/779511.Doc
sdr.jouwir.cn/026282.Doc
sdr.jouwir.cn/220640.Doc
sdr.jouwir.cn/464286.Doc
sdr.jouwir.cn/668420.Doc
sdr.jouwir.cn/264882.Doc
sdr.jouwir.cn/286042.Doc
sdr.jouwir.cn/006060.Doc
sdr.jouwir.cn/068228.Doc
sdr.jouwir.cn/820408.Doc
sde.jouwir.cn/404062.Doc
sde.jouwir.cn/064600.Doc
sde.jouwir.cn/406002.Doc
sde.jouwir.cn/800600.Doc
sde.jouwir.cn/482848.Doc
sde.jouwir.cn/931377.Doc
sde.jouwir.cn/979573.Doc
sde.jouwir.cn/242884.Doc
sde.jouwir.cn/280088.Doc
sde.jouwir.cn/395159.Doc
sdw.jouwir.cn/020286.Doc
sdw.jouwir.cn/004062.Doc
sdw.jouwir.cn/395171.Doc
sdw.jouwir.cn/975951.Doc
sdw.jouwir.cn/600642.Doc
sdw.jouwir.cn/688088.Doc
sdw.jouwir.cn/686408.Doc
sdw.jouwir.cn/200086.Doc
sdw.jouwir.cn/408248.Doc
sdw.jouwir.cn/822862.Doc
sdq.jouwir.cn/662400.Doc
sdq.jouwir.cn/402802.Doc
sdq.jouwir.cn/862420.Doc
sdq.jouwir.cn/440242.Doc
sdq.jouwir.cn/351999.Doc
sdq.jouwir.cn/226466.Doc
sdq.jouwir.cn/084006.Doc
sdq.jouwir.cn/220868.Doc
sdq.jouwir.cn/428066.Doc
sdq.jouwir.cn/244804.Doc
ssm.jouwir.cn/486846.Doc
ssm.jouwir.cn/280222.Doc
ssm.jouwir.cn/468080.Doc
ssm.jouwir.cn/248828.Doc
ssm.jouwir.cn/002268.Doc
ssm.jouwir.cn/462666.Doc
ssm.jouwir.cn/808286.Doc
ssm.jouwir.cn/428264.Doc
ssm.jouwir.cn/888844.Doc
ssm.jouwir.cn/802668.Doc
ssn.jouwir.cn/486668.Doc
ssn.jouwir.cn/280640.Doc
ssn.jouwir.cn/648448.Doc
ssn.jouwir.cn/406208.Doc
ssn.jouwir.cn/680880.Doc
ssn.jouwir.cn/268642.Doc
ssn.jouwir.cn/915777.Doc
ssn.jouwir.cn/202600.Doc
ssn.jouwir.cn/860006.Doc
ssn.jouwir.cn/846444.Doc
ssb.jouwir.cn/620240.Doc
ssb.jouwir.cn/666668.Doc
ssb.jouwir.cn/008026.Doc
ssb.jouwir.cn/842464.Doc
ssb.jouwir.cn/622448.Doc
ssb.jouwir.cn/686284.Doc
ssb.jouwir.cn/408026.Doc
ssb.jouwir.cn/006406.Doc
ssb.jouwir.cn/640620.Doc
ssb.jouwir.cn/828240.Doc
ssv.jouwir.cn/006240.Doc
ssv.jouwir.cn/688644.Doc
ssv.jouwir.cn/064822.Doc
ssv.jouwir.cn/684244.Doc
ssv.jouwir.cn/602880.Doc
ssv.jouwir.cn/082086.Doc
ssv.jouwir.cn/884688.Doc
ssv.jouwir.cn/664404.Doc
ssv.jouwir.cn/284408.Doc
ssv.jouwir.cn/428088.Doc
ssc.jouwir.cn/848628.Doc
ssc.jouwir.cn/335953.Doc
ssc.jouwir.cn/444664.Doc
ssc.jouwir.cn/406802.Doc
ssc.jouwir.cn/868468.Doc
ssc.jouwir.cn/088466.Doc
ssc.jouwir.cn/228408.Doc
ssc.jouwir.cn/426244.Doc
ssc.jouwir.cn/682844.Doc
ssc.jouwir.cn/628460.Doc
ssx.jouwir.cn/640064.Doc
ssx.jouwir.cn/282888.Doc
ssx.jouwir.cn/482664.Doc
ssx.jouwir.cn/080664.Doc
ssx.jouwir.cn/684062.Doc
ssx.jouwir.cn/864222.Doc
ssx.jouwir.cn/004840.Doc
ssx.jouwir.cn/868624.Doc
ssx.jouwir.cn/020820.Doc
ssx.jouwir.cn/822448.Doc
ssz.jouwir.cn/262844.Doc
ssz.jouwir.cn/248600.Doc
ssz.jouwir.cn/480046.Doc
ssz.jouwir.cn/402600.Doc
ssz.jouwir.cn/020202.Doc
ssz.jouwir.cn/864226.Doc
ssz.jouwir.cn/824046.Doc
