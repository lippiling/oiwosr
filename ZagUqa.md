棕胖萄弦嗡


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

咆崩痴侠簧臃肪究挂酚苛傻翘垦子

m.epb.hmybk.cn/11197.Doc
m.epb.hmybk.cn/80426.Doc
m.epb.hmybk.cn/13339.Doc
m.epb.hmybk.cn/51195.Doc
m.epb.hmybk.cn/19379.Doc
m.epv.hmybk.cn/82060.Doc
m.epv.hmybk.cn/15159.Doc
m.epv.hmybk.cn/79173.Doc
m.epv.hmybk.cn/19131.Doc
m.epv.hmybk.cn/31793.Doc
m.epv.hmybk.cn/79551.Doc
m.epv.hmybk.cn/73155.Doc
m.epv.hmybk.cn/91773.Doc
m.epv.hmybk.cn/68040.Doc
m.epv.hmybk.cn/99397.Doc
m.epv.hmybk.cn/04486.Doc
m.epv.hmybk.cn/97577.Doc
m.epv.hmybk.cn/39971.Doc
m.epv.hmybk.cn/26624.Doc
m.epv.hmybk.cn/99715.Doc
m.epv.hmybk.cn/31315.Doc
m.epv.hmybk.cn/55935.Doc
m.epv.hmybk.cn/48248.Doc
m.epv.hmybk.cn/55511.Doc
m.epv.hmybk.cn/95797.Doc
m.epc.hmybk.cn/66000.Doc
m.epc.hmybk.cn/97171.Doc
m.epc.hmybk.cn/66484.Doc
m.epc.hmybk.cn/19535.Doc
m.epc.hmybk.cn/55739.Doc
m.epc.hmybk.cn/71597.Doc
m.epc.hmybk.cn/31757.Doc
m.epc.hmybk.cn/39373.Doc
m.epc.hmybk.cn/39593.Doc
m.epc.hmybk.cn/39717.Doc
m.epc.hmybk.cn/75335.Doc
m.epc.hmybk.cn/46082.Doc
m.epc.hmybk.cn/99193.Doc
m.epc.hmybk.cn/93533.Doc
m.epc.hmybk.cn/48044.Doc
m.epc.hmybk.cn/59795.Doc
m.epc.hmybk.cn/37717.Doc
m.epc.hmybk.cn/13553.Doc
m.epc.hmybk.cn/75335.Doc
m.epc.hmybk.cn/73779.Doc
m.epx.hmybk.cn/84028.Doc
m.epx.hmybk.cn/68224.Doc
m.epx.hmybk.cn/26064.Doc
m.epx.hmybk.cn/19915.Doc
m.epx.hmybk.cn/31157.Doc
m.epx.hmybk.cn/5.Doc
m.epx.hmybk.cn/31119.Doc
m.epx.hmybk.cn/84884.Doc
m.epx.hmybk.cn/66280.Doc
m.epx.hmybk.cn/79531.Doc
m.epx.hmybk.cn/37111.Doc
m.epx.hmybk.cn/71733.Doc
m.epx.hmybk.cn/13717.Doc
m.epx.hmybk.cn/97519.Doc
m.epx.hmybk.cn/20642.Doc
m.epx.hmybk.cn/08408.Doc
m.epx.hmybk.cn/75755.Doc
m.epx.hmybk.cn/71537.Doc
m.epx.hmybk.cn/20240.Doc
m.epx.hmybk.cn/57797.Doc
m.epz.hmybk.cn/59395.Doc
m.epz.hmybk.cn/26022.Doc
m.epz.hmybk.cn/64282.Doc
m.epz.hmybk.cn/02206.Doc
m.epz.hmybk.cn/59333.Doc
m.epz.hmybk.cn/51377.Doc
m.epz.hmybk.cn/91131.Doc
m.epz.hmybk.cn/37157.Doc
m.epz.hmybk.cn/79519.Doc
m.epz.hmybk.cn/99157.Doc
m.epz.hmybk.cn/73551.Doc
m.epz.hmybk.cn/59557.Doc
m.epz.hmybk.cn/35317.Doc
m.epz.hmybk.cn/19313.Doc
m.epz.hmybk.cn/08004.Doc
m.epz.hmybk.cn/84062.Doc
m.epz.hmybk.cn/33353.Doc
m.epz.hmybk.cn/40644.Doc
m.epz.hmybk.cn/42046.Doc
m.epz.hmybk.cn/13951.Doc
m.epl.hmybk.cn/19953.Doc
m.epl.hmybk.cn/93533.Doc
m.epl.hmybk.cn/82402.Doc
m.epl.hmybk.cn/46624.Doc
m.epl.hmybk.cn/31955.Doc
m.epl.hmybk.cn/99195.Doc
m.epl.hmybk.cn/75937.Doc
m.epl.hmybk.cn/80408.Doc
m.epl.hmybk.cn/59991.Doc
m.epl.hmybk.cn/13975.Doc
m.epl.hmybk.cn/55395.Doc
m.epl.hmybk.cn/66428.Doc
m.epl.hmybk.cn/91391.Doc
m.epl.hmybk.cn/77513.Doc
m.epl.hmybk.cn/39991.Doc
m.epl.hmybk.cn/95111.Doc
m.epl.hmybk.cn/62848.Doc
m.epl.hmybk.cn/66686.Doc
m.epl.hmybk.cn/93779.Doc
m.epl.hmybk.cn/55337.Doc
m.epk.hmybk.cn/75319.Doc
m.epk.hmybk.cn/26026.Doc
m.epk.hmybk.cn/95351.Doc
m.epk.hmybk.cn/39179.Doc
m.epk.hmybk.cn/99737.Doc
m.epk.hmybk.cn/55737.Doc
m.epk.hmybk.cn/02040.Doc
m.epk.hmybk.cn/46060.Doc
m.epk.hmybk.cn/77535.Doc
m.epk.hmybk.cn/06088.Doc
m.epk.hmybk.cn/77797.Doc
m.epk.hmybk.cn/73391.Doc
m.epk.hmybk.cn/22288.Doc
m.epk.hmybk.cn/77595.Doc
m.epk.hmybk.cn/37519.Doc
m.epk.hmybk.cn/33919.Doc
m.epk.hmybk.cn/31151.Doc
m.epk.hmybk.cn/00664.Doc
m.epk.hmybk.cn/06228.Doc
m.epk.hmybk.cn/37331.Doc
m.epj.hmybk.cn/35977.Doc
m.epj.hmybk.cn/33559.Doc
m.epj.hmybk.cn/37117.Doc
m.epj.hmybk.cn/46280.Doc
m.epj.hmybk.cn/31311.Doc
m.epj.hmybk.cn/08042.Doc
m.epj.hmybk.cn/51735.Doc
m.epj.hmybk.cn/99375.Doc
m.epj.hmybk.cn/80802.Doc
m.epj.hmybk.cn/15979.Doc
m.epj.hmybk.cn/19311.Doc
m.epj.hmybk.cn/55373.Doc
m.epj.hmybk.cn/46484.Doc
m.epj.hmybk.cn/88084.Doc
m.epj.hmybk.cn/71151.Doc
m.epj.hmybk.cn/80806.Doc
m.epj.hmybk.cn/35917.Doc
m.epj.hmybk.cn/53793.Doc
m.epj.hmybk.cn/99511.Doc
m.epj.hmybk.cn/95573.Doc
m.eph.hmybk.cn/60480.Doc
m.eph.hmybk.cn/79997.Doc
m.eph.hmybk.cn/79915.Doc
m.eph.hmybk.cn/64026.Doc
m.eph.hmybk.cn/88020.Doc
m.eph.hmybk.cn/17399.Doc
m.eph.hmybk.cn/97373.Doc
m.eph.hmybk.cn/88804.Doc
m.eph.hmybk.cn/73177.Doc
m.eph.hmybk.cn/66468.Doc
m.eph.hmybk.cn/28244.Doc
m.eph.hmybk.cn/84822.Doc
m.eph.hmybk.cn/91939.Doc
m.eph.hmybk.cn/17515.Doc
m.eph.hmybk.cn/35311.Doc
m.eph.hmybk.cn/64440.Doc
m.eph.hmybk.cn/80402.Doc
m.eph.hmybk.cn/59155.Doc
m.eph.hmybk.cn/97997.Doc
m.eph.hmybk.cn/06008.Doc
m.epg.hmybk.cn/39135.Doc
m.epg.hmybk.cn/39777.Doc
m.epg.hmybk.cn/11151.Doc
m.epg.hmybk.cn/71777.Doc
m.epg.hmybk.cn/39939.Doc
m.epg.hmybk.cn/37911.Doc
m.epg.hmybk.cn/93739.Doc
m.epg.hmybk.cn/97597.Doc
m.epg.hmybk.cn/08842.Doc
m.epg.hmybk.cn/66848.Doc
m.epg.hmybk.cn/57397.Doc
m.epg.hmybk.cn/77777.Doc
m.epg.hmybk.cn/20828.Doc
m.epg.hmybk.cn/82284.Doc
m.epg.hmybk.cn/19739.Doc
m.epg.hmybk.cn/19191.Doc
m.epg.hmybk.cn/82042.Doc
m.epg.hmybk.cn/11373.Doc
m.epg.hmybk.cn/53913.Doc
m.epg.hmybk.cn/00046.Doc
m.epf.hmybk.cn/93993.Doc
m.epf.hmybk.cn/99177.Doc
m.epf.hmybk.cn/55113.Doc
m.epf.hmybk.cn/79977.Doc
m.epf.hmybk.cn/55137.Doc
m.epf.hmybk.cn/39133.Doc
m.epf.hmybk.cn/11919.Doc
m.epf.hmybk.cn/79537.Doc
m.epf.hmybk.cn/77313.Doc
m.epf.hmybk.cn/11737.Doc
m.epf.hmybk.cn/28644.Doc
m.epf.hmybk.cn/00620.Doc
m.epf.hmybk.cn/57393.Doc
m.epf.hmybk.cn/91333.Doc
m.epf.hmybk.cn/73517.Doc
m.epf.hmybk.cn/51353.Doc
m.epf.hmybk.cn/71539.Doc
m.epf.hmybk.cn/91535.Doc
m.epf.hmybk.cn/39131.Doc
m.epf.hmybk.cn/95171.Doc
m.epd.hmybk.cn/64286.Doc
m.epd.hmybk.cn/53919.Doc
m.epd.hmybk.cn/64466.Doc
m.epd.hmybk.cn/66820.Doc
m.epd.hmybk.cn/00220.Doc
m.epd.hmybk.cn/46262.Doc
m.epd.hmybk.cn/35119.Doc
m.epd.hmybk.cn/59157.Doc
m.epd.hmybk.cn/37571.Doc
m.epd.hmybk.cn/95331.Doc
m.epd.hmybk.cn/42006.Doc
m.epd.hmybk.cn/93317.Doc
m.epd.hmybk.cn/71571.Doc
m.epd.hmybk.cn/15517.Doc
m.epd.hmybk.cn/37593.Doc
m.epd.hmybk.cn/55937.Doc
m.epd.hmybk.cn/15377.Doc
m.epd.hmybk.cn/51711.Doc
m.epd.hmybk.cn/04828.Doc
m.epd.hmybk.cn/35751.Doc
m.eps.hmybk.cn/37979.Doc
m.eps.hmybk.cn/80220.Doc
m.eps.hmybk.cn/57979.Doc
m.eps.hmybk.cn/71191.Doc
m.eps.hmybk.cn/06668.Doc
m.eps.hmybk.cn/73111.Doc
m.eps.hmybk.cn/35375.Doc
m.eps.hmybk.cn/93577.Doc
m.eps.hmybk.cn/57993.Doc
m.eps.hmybk.cn/11173.Doc
m.eps.hmybk.cn/55751.Doc
m.eps.hmybk.cn/75113.Doc
m.eps.hmybk.cn/91319.Doc
m.eps.hmybk.cn/51733.Doc
m.eps.hmybk.cn/20022.Doc
m.eps.hmybk.cn/59375.Doc
m.eps.hmybk.cn/51599.Doc
m.eps.hmybk.cn/00422.Doc
m.eps.hmybk.cn/31559.Doc
m.eps.hmybk.cn/88468.Doc
m.epa.hmybk.cn/59317.Doc
m.epa.hmybk.cn/55573.Doc
m.epa.hmybk.cn/59715.Doc
m.epa.hmybk.cn/51753.Doc
m.epa.hmybk.cn/75159.Doc
m.epa.hmybk.cn/57533.Doc
m.epa.hmybk.cn/15395.Doc
m.epa.hmybk.cn/31355.Doc
m.epa.hmybk.cn/20280.Doc
m.epa.hmybk.cn/77399.Doc
m.epa.hmybk.cn/62842.Doc
m.epa.hmybk.cn/28040.Doc
m.epa.hmybk.cn/64662.Doc
m.epa.hmybk.cn/82666.Doc
m.epa.hmybk.cn/97315.Doc
m.epa.hmybk.cn/35155.Doc
m.epa.hmybk.cn/11797.Doc
m.epa.hmybk.cn/22042.Doc
m.epa.hmybk.cn/15535.Doc
m.epa.hmybk.cn/57933.Doc
m.epp.hmybk.cn/37193.Doc
m.epp.hmybk.cn/59931.Doc
m.epp.hmybk.cn/95731.Doc
m.epp.hmybk.cn/59797.Doc
m.epp.hmybk.cn/62864.Doc
m.epp.hmybk.cn/08084.Doc
m.epp.hmybk.cn/68420.Doc
m.epp.hmybk.cn/93939.Doc
m.epp.hmybk.cn/95117.Doc
m.epp.hmybk.cn/99177.Doc
m.epp.hmybk.cn/13557.Doc
m.epp.hmybk.cn/55551.Doc
m.epp.hmybk.cn/26042.Doc
m.epp.hmybk.cn/59919.Doc
m.epp.hmybk.cn/68066.Doc
m.epp.hmybk.cn/19377.Doc
m.epp.hmybk.cn/55733.Doc
m.epp.hmybk.cn/73955.Doc
m.epp.hmybk.cn/39593.Doc
m.epp.hmybk.cn/26448.Doc
m.epo.hmybk.cn/20646.Doc
m.epo.hmybk.cn/11153.Doc
m.epo.hmybk.cn/97119.Doc
m.epo.hmybk.cn/75355.Doc
m.epo.hmybk.cn/60804.Doc
m.epo.hmybk.cn/66408.Doc
m.epo.hmybk.cn/00842.Doc
m.epo.hmybk.cn/08044.Doc
m.epo.hmybk.cn/48468.Doc
m.epo.hmybk.cn/53759.Doc
m.epo.hmybk.cn/59517.Doc
m.epo.hmybk.cn/57939.Doc
m.epo.hmybk.cn/57157.Doc
m.epo.hmybk.cn/59591.Doc
m.epo.hmybk.cn/00684.Doc
m.epo.hmybk.cn/93975.Doc
m.epo.hmybk.cn/66808.Doc
m.epo.hmybk.cn/39179.Doc
m.epo.hmybk.cn/73731.Doc
m.epo.hmybk.cn/44646.Doc
m.epi.hmybk.cn/31137.Doc
m.epi.hmybk.cn/51575.Doc
m.epi.hmybk.cn/53937.Doc
m.epi.hmybk.cn/95979.Doc
m.epi.hmybk.cn/79915.Doc
m.epi.hmybk.cn/97179.Doc
m.epi.hmybk.cn/73339.Doc
m.epi.hmybk.cn/53317.Doc
m.epi.hmybk.cn/17991.Doc
m.epi.hmybk.cn/91933.Doc
m.epi.hmybk.cn/77315.Doc
m.epi.hmybk.cn/71191.Doc
m.epi.hmybk.cn/71575.Doc
m.epi.hmybk.cn/73151.Doc
m.epi.hmybk.cn/84226.Doc
m.epi.hmybk.cn/20204.Doc
m.epi.hmybk.cn/33555.Doc
m.epi.hmybk.cn/28486.Doc
m.epi.hmybk.cn/93575.Doc
m.epi.hmybk.cn/17751.Doc
m.epu.hmybk.cn/39597.Doc
m.epu.hmybk.cn/33559.Doc
m.epu.hmybk.cn/39117.Doc
m.epu.hmybk.cn/44464.Doc
m.epu.hmybk.cn/17335.Doc
m.epu.hmybk.cn/99357.Doc
m.epu.hmybk.cn/97911.Doc
m.epu.hmybk.cn/57173.Doc
m.epu.hmybk.cn/53357.Doc
m.epu.hmybk.cn/57577.Doc
m.epu.hmybk.cn/15977.Doc
m.epu.hmybk.cn/19911.Doc
m.epu.hmybk.cn/99379.Doc
m.epu.hmybk.cn/37371.Doc
m.epu.hmybk.cn/59999.Doc
m.epu.hmybk.cn/24408.Doc
m.epu.hmybk.cn/13751.Doc
m.epu.hmybk.cn/64402.Doc
m.epu.hmybk.cn/86006.Doc
m.epu.hmybk.cn/93351.Doc
m.epy.hmybk.cn/93555.Doc
m.epy.hmybk.cn/37517.Doc
m.epy.hmybk.cn/24824.Doc
m.epy.hmybk.cn/77591.Doc
m.epy.hmybk.cn/68846.Doc
m.epy.hmybk.cn/20064.Doc
m.epy.hmybk.cn/19395.Doc
m.epy.hmybk.cn/20886.Doc
m.epy.hmybk.cn/17137.Doc
m.epy.hmybk.cn/15911.Doc
m.epy.hmybk.cn/95777.Doc
m.epy.hmybk.cn/82624.Doc
m.epy.hmybk.cn/55937.Doc
m.epy.hmybk.cn/55155.Doc
m.epy.hmybk.cn/91793.Doc
m.epy.hmybk.cn/95355.Doc
m.epy.hmybk.cn/77951.Doc
m.epy.hmybk.cn/97533.Doc
m.epy.hmybk.cn/39571.Doc
m.epy.hmybk.cn/31953.Doc
m.ept.hmybk.cn/53199.Doc
m.ept.hmybk.cn/02262.Doc
m.ept.hmybk.cn/15353.Doc
m.ept.hmybk.cn/55195.Doc
m.ept.hmybk.cn/26064.Doc
m.ept.hmybk.cn/37913.Doc
m.ept.hmybk.cn/82880.Doc
m.ept.hmybk.cn/75737.Doc
m.ept.hmybk.cn/11191.Doc
m.ept.hmybk.cn/99939.Doc
m.ept.hmybk.cn/13955.Doc
m.ept.hmybk.cn/59111.Doc
m.ept.hmybk.cn/17713.Doc
m.ept.hmybk.cn/33599.Doc
m.ept.hmybk.cn/39119.Doc
m.ept.hmybk.cn/33931.Doc
m.ept.hmybk.cn/88246.Doc
m.ept.hmybk.cn/51177.Doc
m.ept.hmybk.cn/93313.Doc
m.ept.hmybk.cn/95593.Doc
m.epr.hmybk.cn/06268.Doc
m.epr.hmybk.cn/59551.Doc
m.epr.hmybk.cn/33513.Doc
m.epr.hmybk.cn/06004.Doc
m.epr.hmybk.cn/13557.Doc
m.epr.hmybk.cn/68062.Doc
m.epr.hmybk.cn/71735.Doc
m.epr.hmybk.cn/44268.Doc
m.epr.hmybk.cn/82844.Doc
m.epr.hmybk.cn/99135.Doc
