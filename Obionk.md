嘶艘献褪挡


# Python概率分布 - 完整代码示例
# 概率分布是统计学的核心，scipy.stats 提供丰富的概率分布函数

import numpy as np
import matplotlib.pyplot as plt
from scipy import stats

# 1. 正态分布 (Normal Distribution)
# 参数: loc=均值, scale=标准差
norm_dist = stats.norm(loc=0, scale=1)  # 标准正态分布

# 概率密度函数 (PDF)
x = np.linspace(-4, 4, 100)
pdf_vals = norm_dist.pdf(x)

# 累积分布函数 (CDF)
cdf_vals = norm_dist.cdf(x)

# 百分点函数 (PPF) / 分位数函数
quantiles = [0.025, 0.5, 0.975]
ppf_vals = norm_dist.ppf(quantiles)

print("正态分布 (mean=0, std=1):")
print(f"PDF at x=0: {norm_dist.pdf(0):.4f}")
print(f"CDF at x=0: {norm_dist.cdf(0):.4f}")
print(f"2.5% 分位数: {norm_dist.ppf(0.025):.4f}")
print(f"中位数:       {norm_dist.ppf(0.5):.4f}")
print(f"97.5% 分位数: {norm_dist.ppf(0.975):.4f}")

# 随机采样
samples = norm_dist.rvs(size=1000, random_state=42)
sample_mean = np.mean(samples)
sample_std = np.std(samples)
print(f"采样均值: {sample_mean:.4f}, 采样标准差: {sample_std:.4f}")

# 2. 指数分布 (Exponential Distribution)
# 常用于等待时间建模，参数 scale=1/λ
exp_dist = stats.expon(scale=2.0)  # λ = 0.5

x_exp = np.linspace(0, 10, 100)
pdf_exp = exp_dist.pdf(x_exp)
cdf_exp = exp_dist.cdf(x_exp)

print(f"\n指数分布 (scale=2.0):")
print(f"PDF at x=0: {exp_dist.pdf(0):.4f}")
print(f"P(X > 3) = {1 - exp_dist.cdf(3):.4f}")
print(f"均值 = {exp_dist.mean():.4f}, 方差 = {exp_dist.var():.4f}")

# 3. 泊松分布 (Poisson Distribution)
# 离散分布，描述单位时间内事件发生次数，参数 mu=λ
poisson_dist = stats.poisson(mu=3.5)  # λ = 3.5

k_vals = np.arange(0, 12)
pmf_poisson = poisson_dist.pmf(k_vals)
cdf_poisson = poisson_dist.cdf(k_vals)

print(f"\n泊松分布 (λ=3.5):")
print(f"P(X=3) = {poisson_dist.pmf(3):.4f}")
print(f"P(X<=5) = {poisson_dist.cdf(5):.4f}")
print(f"P(X>7) = {1 - poisson_dist.cdf(7):.4f}")
print(f"均值 = {poisson_dist.mean():.4f}, 方差 = {poisson_dist.var():.4f}")

# 4. 二项分布 (Binomial Distribution)
# n 次独立伯努利试验的成功次数，参数 n 和 p
binom_dist = stats.binom(n=20, p=0.3)  # 20 次试验，成功概率 0.3

k_binom = np.arange(0, 21)
pmf_binom = binom_dist.pmf(k_binom)
cdf_binom = binom_dist.cdf(k_binom)

print(f"\n二项分布 (n=20, p=0.3):")
print(f"P(X=6) = {binom_dist.pmf(6):.4f}")
print(f"P(X<=5) = {binom_dist.cdf(5):.4f}")
print(f"P(4 <= X <= 8) = {binom_dist.cdf(8) - binom_dist.cdf(3):.4f}")
print(f"均值 = {binom_dist.mean():.4f}, 方差 = {binom_dist.var():.4f}")

# 5. 均匀分布 (Uniform Distribution)
uniform_dist = stats.uniform(loc=0, scale=5)  # [0, 5] 上的均匀分布

x_unif = np.linspace(-1, 6, 100)
pdf_unif = uniform_dist.pdf(x_unif)
cdf_unif = uniform_dist.cdf(x_unif)

print(f"\n均匀分布 [0, 5]:")
print(f"PDF at x=2: {uniform_dist.pdf(2):.4f}")
print(f"P(1 < X < 3) = {uniform_dist.cdf(3) - uniform_dist.cdf(1):.4f}")

# 6. 分布拟合：从数据中估计分布参数
# 生成数据并拟合正态分布
np.random.seed(123)
data = np.random.normal(loc=5, scale=1.5, size=1000)

# 使用正态分布拟合数据
fitted_params = stats.norm.fit(data)  # 返回 (loc, scale) 的 MLE
print(f"\n分布拟合结果:")
print(f"真实参数: loc=5.0, scale=1.5")
print(f"拟合参数: loc={fitted_params[0]:.4f}, scale={fitted_params[1]:.4f}")

# Kolmogorov-Smirnov 拟合优度检验
ks_stat, ks_pvalue = stats.kstest(data, 'norm', args=fitted_params)
print(f"KS 检验统计量: {ks_stat:.4f}, p-value: {ks_pvalue:.4f}")

# 7. 多种分布的矩（均值和方差）比较
distributions = {
    '正态 N(0,1)': stats.norm(0, 1),
    '指数 Exp(1)': stats.expon(scale=1),
    '泊松 Poi(3)': stats.poisson(mu=3),
    '二项 B(10,0.5)': stats.binom(10, 0.5),
    '均匀 U(0,1)': stats.uniform(0, 1)
}
print(f"\n各种分布的统计量:")
for name, dist in distributions.items():
    print(f"  {name:15s}: 均值={dist.mean():.4f}, 方差={dist.var():.4f}, 偏度={dist.stats(moments='s'):.4f}")

# 8. 概率分布的生存函数和风险函数
# 生存函数 S(x) = 1 - F(x), 风险函数 h(x) = f(x) / S(x)
x_surv = np.linspace(0, 5, 50)
exp_sf = exp_dist.sf(x_surv)  # 生存函数
exp_hf = exp_dist.pdf(x_surv) / exp_dist.sf(x_surv)  # 风险函数
print(f"\n指数分布生存函数 (x=2): {exp_dist.sf(2):.4f}")
print(f"指数分布风险函数 (x=2): {exp_dist.pdf(2)/exp_dist.sf(2):.4f}")

# 9. 分位数-分位数图 (Q-Q Plot) 的数字检验
# 检验数据是否服从指定分布
data_test = np.random.normal(5, 1, 200)
# 标准化数据
data_standardized = (data_test - np.mean(data_test)) / np.std(data_test)
# 理论分位数
theoretical_quantiles = stats.norm.ppf(np.linspace(0.01, 0.99, 198))
# 样本分位数
sample_quantiles = np.sort(data_standardized)
# 相关系数（越接近 1 越正态）
r, _ = stats.pearsonr(theoretical_quantiles, sample_quantiles)
print(f"\nQ-Q 正态性相关系数: {r:.4f} (越接近 1 越正态)")

print("\n概率分布总结：scipy.stats 提供统一的分布 API")
print("pdf/cdf/ppf/rvs 是核心方法，fit 实现 MLE 参数估计")

腋沧苟济到估泼灯褐迅帘疤吮坛侠

m.wcs.kxnxh.cn/15117.Doc
m.wcs.kxnxh.cn/39115.Doc
m.wcs.kxnxh.cn/77311.Doc
m.wcs.kxnxh.cn/02286.Doc
m.wcs.kxnxh.cn/91931.Doc
m.wcs.kxnxh.cn/51535.Doc
m.wcs.kxnxh.cn/77911.Doc
m.wcs.kxnxh.cn/06886.Doc
m.wcs.kxnxh.cn/08424.Doc
m.wcs.kxnxh.cn/75735.Doc
m.wcs.kxnxh.cn/97537.Doc
m.wcs.kxnxh.cn/84822.Doc
m.wcs.kxnxh.cn/93539.Doc
m.wcs.kxnxh.cn/91351.Doc
m.wcs.kxnxh.cn/73713.Doc
m.wcs.kxnxh.cn/79999.Doc
m.wcs.kxnxh.cn/82266.Doc
m.wcs.kxnxh.cn/80688.Doc
m.wcs.kxnxh.cn/79319.Doc
m.wcs.kxnxh.cn/57555.Doc
m.wca.kxnxh.cn/73535.Doc
m.wca.kxnxh.cn/86428.Doc
m.wca.kxnxh.cn/11175.Doc
m.wca.kxnxh.cn/44048.Doc
m.wca.kxnxh.cn/79157.Doc
m.wca.kxnxh.cn/51177.Doc
m.wca.kxnxh.cn/59539.Doc
m.wca.kxnxh.cn/53939.Doc
m.wca.kxnxh.cn/31393.Doc
m.wca.kxnxh.cn/20060.Doc
m.wca.kxnxh.cn/31157.Doc
m.wca.kxnxh.cn/95133.Doc
m.wca.kxnxh.cn/33593.Doc
m.wca.kxnxh.cn/42806.Doc
m.wca.kxnxh.cn/06022.Doc
m.wca.kxnxh.cn/46626.Doc
m.wca.kxnxh.cn/19553.Doc
m.wca.kxnxh.cn/35915.Doc
m.wca.kxnxh.cn/17197.Doc
m.wca.kxnxh.cn/66222.Doc
m.wcp.kxnxh.cn/88006.Doc
m.wcp.kxnxh.cn/46224.Doc
m.wcp.kxnxh.cn/91793.Doc
m.wcp.kxnxh.cn/11573.Doc
m.wcp.kxnxh.cn/51159.Doc
m.wcp.kxnxh.cn/99559.Doc
m.wcp.kxnxh.cn/57191.Doc
m.wcp.kxnxh.cn/44044.Doc
m.wcp.kxnxh.cn/02820.Doc
m.wcp.kxnxh.cn/33137.Doc
m.wcp.kxnxh.cn/75557.Doc
m.wcp.kxnxh.cn/17991.Doc
m.wcp.kxnxh.cn/91771.Doc
m.wcp.kxnxh.cn/28802.Doc
m.wcp.kxnxh.cn/62462.Doc
m.wcp.kxnxh.cn/42808.Doc
m.wcp.kxnxh.cn/51313.Doc
m.wcp.kxnxh.cn/66288.Doc
m.wcp.kxnxh.cn/11137.Doc
m.wcp.kxnxh.cn/86624.Doc
m.wco.kxnxh.cn/37995.Doc
m.wco.kxnxh.cn/75715.Doc
m.wco.kxnxh.cn/19979.Doc
m.wco.kxnxh.cn/80840.Doc
m.wco.kxnxh.cn/20264.Doc
m.wco.kxnxh.cn/55375.Doc
m.wco.kxnxh.cn/15959.Doc
m.wco.kxnxh.cn/79537.Doc
m.wco.kxnxh.cn/48024.Doc
m.wco.kxnxh.cn/40426.Doc
m.wco.kxnxh.cn/24040.Doc
m.wco.kxnxh.cn/22860.Doc
m.wco.kxnxh.cn/75137.Doc
m.wco.kxnxh.cn/15713.Doc
m.wco.kxnxh.cn/19715.Doc
m.wco.kxnxh.cn/68228.Doc
m.wco.kxnxh.cn/95353.Doc
m.wco.kxnxh.cn/51157.Doc
m.wco.kxnxh.cn/60246.Doc
m.wco.kxnxh.cn/59717.Doc
m.wci.kxnxh.cn/13317.Doc
m.wci.kxnxh.cn/68604.Doc
m.wci.kxnxh.cn/19199.Doc
m.wci.kxnxh.cn/82242.Doc
m.wci.kxnxh.cn/00644.Doc
m.wci.kxnxh.cn/26820.Doc
m.wci.kxnxh.cn/00666.Doc
m.wci.kxnxh.cn/37175.Doc
m.wci.kxnxh.cn/77719.Doc
m.wci.kxnxh.cn/46800.Doc
m.wci.kxnxh.cn/06408.Doc
m.wci.kxnxh.cn/11917.Doc
m.wci.kxnxh.cn/75313.Doc
m.wci.kxnxh.cn/44224.Doc
m.wci.kxnxh.cn/17313.Doc
m.wci.kxnxh.cn/51555.Doc
m.wci.kxnxh.cn/26408.Doc
m.wci.kxnxh.cn/95575.Doc
m.wci.kxnxh.cn/75795.Doc
m.wci.kxnxh.cn/20222.Doc
m.wcu.kxnxh.cn/40884.Doc
m.wcu.kxnxh.cn/48006.Doc
m.wcu.kxnxh.cn/93571.Doc
m.wcu.kxnxh.cn/68084.Doc
m.wcu.kxnxh.cn/79555.Doc
m.wcu.kxnxh.cn/19779.Doc
m.wcu.kxnxh.cn/53739.Doc
m.wcu.kxnxh.cn/04282.Doc
m.wcu.kxnxh.cn/40846.Doc
m.wcu.kxnxh.cn/19973.Doc
m.wcu.kxnxh.cn/60460.Doc
m.wcu.kxnxh.cn/40886.Doc
m.wcu.kxnxh.cn/17917.Doc
m.wcu.kxnxh.cn/59597.Doc
m.wcu.kxnxh.cn/75935.Doc
m.wcu.kxnxh.cn/99511.Doc
m.wcu.kxnxh.cn/93173.Doc
m.wcu.kxnxh.cn/62006.Doc
m.wcu.kxnxh.cn/42068.Doc
m.wcu.kxnxh.cn/24264.Doc
m.wcy.kxnxh.cn/15991.Doc
m.wcy.kxnxh.cn/99591.Doc
m.wcy.kxnxh.cn/86044.Doc
m.wcy.kxnxh.cn/28842.Doc
m.wcy.kxnxh.cn/04408.Doc
m.wcy.kxnxh.cn/62460.Doc
m.wcy.kxnxh.cn/71539.Doc
m.wcy.kxnxh.cn/17577.Doc
m.wcy.kxnxh.cn/13331.Doc
m.wcy.kxnxh.cn/39515.Doc
m.wcy.kxnxh.cn/82880.Doc
m.wcy.kxnxh.cn/71331.Doc
m.wcy.kxnxh.cn/44086.Doc
m.wcy.kxnxh.cn/51799.Doc
m.wcy.kxnxh.cn/95553.Doc
m.wcy.kxnxh.cn/24022.Doc
m.wcy.kxnxh.cn/75715.Doc
m.wcy.kxnxh.cn/55573.Doc
m.wcy.kxnxh.cn/19737.Doc
m.wcy.kxnxh.cn/46822.Doc
m.wct.kxnxh.cn/26464.Doc
m.wct.kxnxh.cn/35115.Doc
m.wct.kxnxh.cn/51531.Doc
m.wct.kxnxh.cn/66240.Doc
m.wct.kxnxh.cn/37391.Doc
m.wct.kxnxh.cn/79575.Doc
m.wct.kxnxh.cn/19355.Doc
m.wct.kxnxh.cn/53799.Doc
m.wct.kxnxh.cn/13319.Doc
m.wct.kxnxh.cn/86486.Doc
m.wct.kxnxh.cn/79375.Doc
m.wct.kxnxh.cn/11335.Doc
m.wct.kxnxh.cn/11979.Doc
m.wct.kxnxh.cn/35539.Doc
m.wct.kxnxh.cn/51557.Doc
m.wct.kxnxh.cn/75355.Doc
m.wct.kxnxh.cn/77399.Doc
m.wct.kxnxh.cn/08244.Doc
m.wct.kxnxh.cn/97117.Doc
m.wct.kxnxh.cn/31151.Doc
m.wcr.kxnxh.cn/57515.Doc
m.wcr.kxnxh.cn/24808.Doc
m.wcr.kxnxh.cn/84668.Doc
m.wcr.kxnxh.cn/04604.Doc
m.wcr.kxnxh.cn/28446.Doc
m.wcr.kxnxh.cn/75171.Doc
m.wcr.kxnxh.cn/55979.Doc
m.wcr.kxnxh.cn/33315.Doc
m.wcr.kxnxh.cn/11373.Doc
m.wcr.kxnxh.cn/79759.Doc
m.wcr.kxnxh.cn/59557.Doc
m.wcr.kxnxh.cn/46446.Doc
m.wcr.kxnxh.cn/31737.Doc
m.wcr.kxnxh.cn/17511.Doc
m.wcr.kxnxh.cn/39997.Doc
m.wcr.kxnxh.cn/99997.Doc
m.wcr.kxnxh.cn/59193.Doc
m.wcr.kxnxh.cn/77951.Doc
m.wcr.kxnxh.cn/82280.Doc
m.wcr.kxnxh.cn/15191.Doc
m.wce.kxnxh.cn/46244.Doc
m.wce.kxnxh.cn/19335.Doc
m.wce.kxnxh.cn/73793.Doc
m.wce.kxnxh.cn/13775.Doc
m.wce.kxnxh.cn/97759.Doc
m.wce.kxnxh.cn/91915.Doc
m.wce.kxnxh.cn/99979.Doc
m.wce.kxnxh.cn/77511.Doc
m.wce.kxnxh.cn/19313.Doc
m.wce.kxnxh.cn/71319.Doc
m.wce.kxnxh.cn/48004.Doc
m.wce.kxnxh.cn/15373.Doc
m.wce.kxnxh.cn/26422.Doc
m.wce.kxnxh.cn/46442.Doc
m.wce.kxnxh.cn/57917.Doc
m.wce.kxnxh.cn/31711.Doc
m.wce.kxnxh.cn/59599.Doc
m.wce.kxnxh.cn/39139.Doc
m.wce.kxnxh.cn/31351.Doc
m.wce.kxnxh.cn/17955.Doc
m.wcw.kxnxh.cn/97193.Doc
m.wcw.kxnxh.cn/24806.Doc
m.wcw.kxnxh.cn/40662.Doc
m.wcw.kxnxh.cn/15197.Doc
m.wcw.kxnxh.cn/55971.Doc
m.wcw.kxnxh.cn/19573.Doc
m.wcw.kxnxh.cn/51931.Doc
m.wcw.kxnxh.cn/59393.Doc
m.wcw.kxnxh.cn/15751.Doc
m.wcw.kxnxh.cn/48468.Doc
m.wcw.kxnxh.cn/15353.Doc
m.wcw.kxnxh.cn/19117.Doc
m.wcw.kxnxh.cn/59517.Doc
m.wcw.kxnxh.cn/46440.Doc
m.wcw.kxnxh.cn/97177.Doc
m.wcw.kxnxh.cn/33537.Doc
m.wcw.kxnxh.cn/17537.Doc
m.wcw.kxnxh.cn/11539.Doc
m.wcw.kxnxh.cn/82028.Doc
m.wcw.kxnxh.cn/59137.Doc
m.wcq.kxnxh.cn/68666.Doc
m.wcq.kxnxh.cn/80648.Doc
m.wcq.kxnxh.cn/80622.Doc
m.wcq.kxnxh.cn/24064.Doc
m.wcq.kxnxh.cn/59557.Doc
m.wcq.kxnxh.cn/24886.Doc
m.wcq.kxnxh.cn/86262.Doc
m.wcq.kxnxh.cn/06848.Doc
m.wcq.kxnxh.cn/59971.Doc
m.wcq.kxnxh.cn/51795.Doc
m.wcq.kxnxh.cn/02686.Doc
m.wcq.kxnxh.cn/51993.Doc
m.wcq.kxnxh.cn/75335.Doc
m.wcq.kxnxh.cn/00024.Doc
m.wcq.kxnxh.cn/71115.Doc
m.wcq.kxnxh.cn/40468.Doc
m.wcq.kxnxh.cn/62042.Doc
m.wcq.kxnxh.cn/17531.Doc
m.wcq.kxnxh.cn/75953.Doc
m.wcq.kxnxh.cn/62664.Doc
m.wxm.kxnxh.cn/17195.Doc
m.wxm.kxnxh.cn/79191.Doc
m.wxm.kxnxh.cn/93739.Doc
m.wxm.kxnxh.cn/68466.Doc
m.wxm.kxnxh.cn/00024.Doc
m.wxm.kxnxh.cn/97539.Doc
m.wxm.kxnxh.cn/91377.Doc
m.wxm.kxnxh.cn/57713.Doc
m.wxm.kxnxh.cn/75913.Doc
m.wxm.kxnxh.cn/02222.Doc
m.wxm.kxnxh.cn/73395.Doc
m.wxm.kxnxh.cn/88048.Doc
m.wxm.kxnxh.cn/22604.Doc
m.wxm.kxnxh.cn/64022.Doc
m.wxm.kxnxh.cn/15933.Doc
m.wxm.kxnxh.cn/51979.Doc
m.wxm.kxnxh.cn/17775.Doc
m.wxm.kxnxh.cn/28208.Doc
m.wxm.kxnxh.cn/42606.Doc
m.wxm.kxnxh.cn/17355.Doc
m.wxn.kxnxh.cn/31175.Doc
m.wxn.kxnxh.cn/84020.Doc
m.wxn.kxnxh.cn/95533.Doc
m.wxn.kxnxh.cn/95979.Doc
m.wxn.kxnxh.cn/55719.Doc
m.wxn.kxnxh.cn/60440.Doc
m.wxn.kxnxh.cn/19931.Doc
m.wxn.kxnxh.cn/46824.Doc
m.wxn.kxnxh.cn/79195.Doc
m.wxn.kxnxh.cn/66864.Doc
m.wxn.kxnxh.cn/35535.Doc
m.wxn.kxnxh.cn/91955.Doc
m.wxn.kxnxh.cn/17957.Doc
m.wxn.kxnxh.cn/59155.Doc
m.wxn.kxnxh.cn/37337.Doc
m.wxn.kxnxh.cn/75353.Doc
m.wxn.kxnxh.cn/53795.Doc
m.wxn.kxnxh.cn/33755.Doc
m.wxn.kxnxh.cn/64608.Doc
m.wxn.kxnxh.cn/80444.Doc
m.wxb.kxnxh.cn/75953.Doc
m.wxb.kxnxh.cn/37533.Doc
m.wxb.kxnxh.cn/17553.Doc
m.wxb.kxnxh.cn/00046.Doc
m.wxb.kxnxh.cn/82868.Doc
m.wxb.kxnxh.cn/02442.Doc
m.wxb.kxnxh.cn/97775.Doc
m.wxb.kxnxh.cn/28820.Doc
m.wxb.kxnxh.cn/71917.Doc
m.wxb.kxnxh.cn/15951.Doc
m.wxb.kxnxh.cn/95779.Doc
m.wxb.kxnxh.cn/17771.Doc
m.wxb.kxnxh.cn/35533.Doc
m.wxb.kxnxh.cn/33977.Doc
m.wxb.kxnxh.cn/06828.Doc
m.wxb.kxnxh.cn/26062.Doc
m.wxb.kxnxh.cn/60406.Doc
m.wxb.kxnxh.cn/86466.Doc
m.wxb.kxnxh.cn/53575.Doc
m.wxb.kxnxh.cn/28024.Doc
m.wxv.kxnxh.cn/24022.Doc
m.wxv.kxnxh.cn/26026.Doc
m.wxv.kxnxh.cn/22260.Doc
m.wxv.kxnxh.cn/04848.Doc
m.wxv.kxnxh.cn/31979.Doc
m.wxv.kxnxh.cn/59399.Doc
m.wxv.kxnxh.cn/51751.Doc
m.wxv.kxnxh.cn/68044.Doc
m.wxv.kxnxh.cn/44042.Doc
m.wxv.kxnxh.cn/24042.Doc
m.wxv.kxnxh.cn/00880.Doc
m.wxv.kxnxh.cn/55995.Doc
m.wxv.kxnxh.cn/06828.Doc
m.wxv.kxnxh.cn/35757.Doc
m.wxv.kxnxh.cn/79133.Doc
m.wxv.kxnxh.cn/06882.Doc
m.wxv.kxnxh.cn/04208.Doc
m.wxv.kxnxh.cn/04402.Doc
m.wxv.kxnxh.cn/66422.Doc
m.wxv.kxnxh.cn/19117.Doc
m.wxc.kxnxh.cn/55739.Doc
m.wxc.kxnxh.cn/79317.Doc
m.wxc.kxnxh.cn/93557.Doc
m.wxc.kxnxh.cn/33337.Doc
m.wxc.kxnxh.cn/48800.Doc
m.wxc.kxnxh.cn/51157.Doc
m.wxc.kxnxh.cn/79919.Doc
m.wxc.kxnxh.cn/15179.Doc
m.wxc.kxnxh.cn/33739.Doc
m.wxc.kxnxh.cn/97399.Doc
m.wxc.kxnxh.cn/15113.Doc
m.wxc.kxnxh.cn/62248.Doc
m.wxc.kxnxh.cn/73113.Doc
m.wxc.kxnxh.cn/86048.Doc
m.wxc.kxnxh.cn/24624.Doc
m.wxc.kxnxh.cn/06808.Doc
m.wxc.kxnxh.cn/31339.Doc
m.wxc.kxnxh.cn/28828.Doc
m.wxc.kxnxh.cn/08088.Doc
m.wxc.kxnxh.cn/71355.Doc
m.wxx.kxnxh.cn/71993.Doc
m.wxx.kxnxh.cn/60402.Doc
m.wxx.kxnxh.cn/24266.Doc
m.wxx.kxnxh.cn/17135.Doc
m.wxx.kxnxh.cn/66022.Doc
m.wxx.kxnxh.cn/37979.Doc
m.wxx.kxnxh.cn/73573.Doc
m.wxx.kxnxh.cn/20086.Doc
m.wxx.kxnxh.cn/48428.Doc
m.wxx.kxnxh.cn/86266.Doc
m.wxx.kxnxh.cn/55793.Doc
m.wxx.kxnxh.cn/06444.Doc
m.wxx.kxnxh.cn/46040.Doc
m.wxx.kxnxh.cn/39519.Doc
m.wxx.kxnxh.cn/44484.Doc
m.wxx.kxnxh.cn/55199.Doc
m.wxx.kxnxh.cn/40686.Doc
m.wxx.kxnxh.cn/93595.Doc
m.wxx.kxnxh.cn/11595.Doc
m.wxx.kxnxh.cn/82642.Doc
m.wxz.kxnxh.cn/26806.Doc
m.wxz.kxnxh.cn/19111.Doc
m.wxz.kxnxh.cn/42408.Doc
m.wxz.kxnxh.cn/20622.Doc
m.wxz.kxnxh.cn/84280.Doc
m.wxz.kxnxh.cn/40260.Doc
m.wxz.kxnxh.cn/15371.Doc
m.wxz.kxnxh.cn/80882.Doc
m.wxz.kxnxh.cn/15511.Doc
m.wxz.kxnxh.cn/84608.Doc
m.wxz.kxnxh.cn/19971.Doc
m.wxz.kxnxh.cn/86040.Doc
m.wxz.kxnxh.cn/57713.Doc
m.wxz.kxnxh.cn/22042.Doc
m.wxz.kxnxh.cn/97977.Doc
m.wxz.kxnxh.cn/95537.Doc
m.wxz.kxnxh.cn/84202.Doc
m.wxz.kxnxh.cn/04666.Doc
m.wxz.kxnxh.cn/73571.Doc
m.wxz.kxnxh.cn/51351.Doc
m.wxl.kxnxh.cn/99117.Doc
m.wxl.kxnxh.cn/08000.Doc
m.wxl.kxnxh.cn/55395.Doc
m.wxl.kxnxh.cn/62466.Doc
m.wxl.kxnxh.cn/31155.Doc
m.wxl.kxnxh.cn/24484.Doc
m.wxl.kxnxh.cn/13971.Doc
m.wxl.kxnxh.cn/08420.Doc
m.wxl.kxnxh.cn/17791.Doc
m.wxl.kxnxh.cn/02262.Doc
m.wxl.kxnxh.cn/11913.Doc
m.wxl.kxnxh.cn/15937.Doc
m.wxl.kxnxh.cn/53515.Doc
m.wxl.kxnxh.cn/71759.Doc
m.wxl.kxnxh.cn/46040.Doc
