状窍骋来步


# Python傅里叶变换 - 完整代码示例
# 傅里叶变换将信号从时域映射到频域，是信号处理的基石

import numpy as np
import matplotlib.pyplot as plt
from scipy import signal as sp_signal

# 1. 生成合成信号：两个不同频率的正弦波叠加 + 噪声
# 时域参数
fs = 1000           # 采样率 1000 Hz
T = 1.0             # 信号时长 1 秒
N = int(fs * T)     # 总采样点数
t = np.linspace(0, T, N, endpoint=False)  # 时间向量

# 生成信号：50Hz 和 120Hz 正弦波叠加 + 随机噪声
f1, f2 = 50, 120  # 两个频率分量 (Hz)
signal_clean = np.sin(2 * np.pi * f1 * t) + 0.5 * np.sin(2 * np.pi * f2 * t)
noise = 0.3 * np.random.randn(N)
signal_noisy = signal_clean + noise

print(f"信号参数: 采样率={fs}Hz, 时长={T}s, 采样点数={N}")
print(f"频率分量: {f1}Hz 和 {f2}Hz")

# 2. 快速傅里叶变换 (FFT)
# 计算 FFT 得到频谱
fft_result = np.fft.fft(signal_noisy)  # 复数频谱
fft_freq = np.fft.fftfreq(N, d=1/fs)   # 对应的频率轴

# 3. 幅值谱计算（取绝对值并归一化）
amplitude_spectrum = np.abs(fft_result) / N  # 归一化幅值
# 只取正频率部分（FFT 结果对称，负频率包含相同信息）
half_N = N // 2
pos_freq = fft_freq[:half_N]
pos_amp = amplitude_spectrum[:half_N]

# 找到幅值最大的两个频率分量
peak_indices = np.argsort(pos_amp)[-5:][::-1]  # 前 5 个峰值
print(f"\n检测到的前 3 个主要频率分量:")
for idx in peak_indices[:3]:
    if pos_amp[idx] > 0.01:  # 忽略噪声分量
        print(f"  频率 = {pos_freq[idx]:.1f} Hz, 幅值 = {pos_amp[idx]:.4f}")

# 4. 相位谱
phase_spectrum = np.angle(fft_result[:half_N])
print(f"\n50Hz 处相位: {phase_spectrum[np.argmin(abs(pos_freq - 50))]:.4f} rad")
print(f"120Hz 处相位: {phase_spectrum[np.argmin(abs(pos_freq - 120))]:.4f} rad")

# 5. 频域滤波：去除 120Hz 分量（陷波滤波器）
# 在频域将 120Hz 附近的幅值置零
fft_filtered = fft_result.copy()
# 定义要滤除的频率范围（陷波带）
notch_center = 120  # 要滤除的中心频率
notch_width = 5     # 陷波带宽 ±5Hz
# 创建频率掩码：在陷波带内的频率置零
notch_mask = np.abs(fft_freq - notch_center) < notch_width
# 同时也滤除负频率对应的部分
fft_filtered[notch_mask] = 0

# 6. 逆 FFT 重建滤波后的时域信号
signal_filtered = np.fft.ifft(fft_filtered).real

# 验证：滤波后的信号幅值衰减
amp_before = np.std(signal_noisy)
amp_after = np.std(signal_filtered)
print(f"\n滤波前信号标准差: {amp_before:.4f}")
print(f"滤波后信号标准差: {amp_after:.4f}")

# 7. 功率谱密度估计 (Welch 方法)
# Welch 方法将信号分段并平均，降低噪声影响
f_welch, psd = sp_signal.welch(signal_noisy, fs, nperseg=256)

# 找到 PSD 中的峰值
psd_peaks = np.argsort(psd)[-5:][::-1]
print(f"\nWelch PSD 检测的主要频率:")
for idx in psd_peaks[:3]:
    print(f"  频率 = {f_welch[idx]:.1f} Hz, PSD = {psd[idx]:.4f}")

# 8. 短时傅里叶变换 (STFT)：时频分析
# 适用于非平稳信号的时频表示
f_stft, t_stft, Zxx = sp_signal.stft(signal_noisy, fs, nperseg=128)
# Zxx 是复数时频谱，取幅值
stft_magnitude = np.abs(Zxx)

# 在时频谱中定位能量最大值
max_idx = np.unravel_index(np.argmax(stft_magnitude), stft_magnitude.shape)
print(f"\nSTFT 能量最大值位置: 时间={t_stft[max_idx[1]]:.3f}s, 频率={f_stft[max_idx[0]]:.1f}Hz")

# 9. 自相关与频域关系
# 自相关的傅里叶变换等于功率谱（Wiener-Khinchin 定理）
autocorr = np.correlate(signal_clean, signal_clean, mode='same')
psd_from_ac = np.abs(np.fft.fft(autocorr))[:half_N]
print(f"\n自相关法 PSD 在 50Hz 处幅值: {psd_from_ac[np.argmin(abs(pos_freq - 50))]:.4f}")

# 10. 频谱泄露与窗函数
# 使用汉宁窗减少频谱泄露
window = np.hanning(N)
signal_windowed = signal_noisy * window
# 加窗后的 FFT 分析
fft_windowed = np.fft.fft(signal_windowed)
amp_windowed = np.abs(fft_windowed[:half_N]) / np.sum(window) * 2
print(f"\n加窗后 50Hz 幅值: {amp_windowed[np.argmin(abs(pos_freq - 50))]:.4f}")
print(f"加窗后 120Hz 幅值: {amp_windowed[np.argmin(abs(pos_freq - 120))]:.4f}")

print("\n傅里叶变换总结：FFT 是频域分析的核心工具")
print("窗函数、Welch 方法和 STFT 解决实际工程中的关键问题")

瞬悔潭鬃壁跃匆刃友翟状量翟乒鸥

qtv.ag5qr6tv.cn/597395.htm
qtv.ag5qr6tv.cn/575315.htm
qtv.ag5qr6tv.cn/333715.htm
qtc.g6lz11tv.cn/597915.htm
qtc.g6lz11tv.cn/771955.htm
qtc.g6lz11tv.cn/537995.htm
qtc.g6lz11tv.cn/119995.htm
qtc.g6lz11tv.cn/571735.htm
qtc.g6lz11tv.cn/173315.htm
qtc.g6lz11tv.cn/399935.htm
qtc.g6lz11tv.cn/799555.htm
qtc.g6lz11tv.cn/177515.htm
qtc.g6lz11tv.cn/597115.htm
qtx.g6lz11tv.cn/911135.htm
qtx.g6lz11tv.cn/753515.htm
qtx.g6lz11tv.cn/937575.htm
qtx.g6lz11tv.cn/119735.htm
qtx.g6lz11tv.cn/135115.htm
qtx.g6lz11tv.cn/937935.htm
qtx.g6lz11tv.cn/371375.htm
qtx.g6lz11tv.cn/591135.htm
qtx.g6lz11tv.cn/353795.htm
qtx.g6lz11tv.cn/715335.htm
qtz.g6lz11tv.cn/931515.htm
qtz.g6lz11tv.cn/933575.htm
qtz.g6lz11tv.cn/157335.htm
qtz.g6lz11tv.cn/517975.htm
qtz.g6lz11tv.cn/733335.htm
qtz.g6lz11tv.cn/759155.htm
qtz.g6lz11tv.cn/757535.htm
qtz.g6lz11tv.cn/173975.htm
qtz.g6lz11tv.cn/119735.htm
qtz.g6lz11tv.cn/997135.htm
qtl.g6lz11tv.cn/737955.htm
qtl.g6lz11tv.cn/137375.htm
qtl.g6lz11tv.cn/179595.htm
qtl.g6lz11tv.cn/131555.htm
qtl.g6lz11tv.cn/519715.htm
qtl.g6lz11tv.cn/379355.htm
qtl.g6lz11tv.cn/533135.htm
qtl.g6lz11tv.cn/739155.htm
qtl.g6lz11tv.cn/595355.htm
qtl.g6lz11tv.cn/173935.htm
qtk.g6lz11tv.cn/553915.htm
qtk.g6lz11tv.cn/793715.htm
qtk.g6lz11tv.cn/131795.htm
qtk.g6lz11tv.cn/533515.htm
qtk.g6lz11tv.cn/973335.htm
qtk.g6lz11tv.cn/995955.htm
qtk.g6lz11tv.cn/513975.htm
qtk.g6lz11tv.cn/393755.htm
qtk.g6lz11tv.cn/571195.htm
qtk.g6lz11tv.cn/991995.htm
qtj.g6lz11tv.cn/393515.htm
qtj.g6lz11tv.cn/913995.htm
qtj.g6lz11tv.cn/173335.htm
qtj.g6lz11tv.cn/197915.htm
qtj.g6lz11tv.cn/759535.htm
qtj.g6lz11tv.cn/355315.htm
qtj.g6lz11tv.cn/577975.htm
qtj.g6lz11tv.cn/115975.htm
qtj.g6lz11tv.cn/135715.htm
qtj.g6lz11tv.cn/971355.htm
qth.g6lz11tv.cn/133775.htm
qth.g6lz11tv.cn/937195.htm
qth.g6lz11tv.cn/397955.htm
qth.g6lz11tv.cn/151775.htm
qth.g6lz11tv.cn/151995.htm
qth.g6lz11tv.cn/795555.htm
qth.g6lz11tv.cn/973935.htm
qth.g6lz11tv.cn/339395.htm
qth.g6lz11tv.cn/917755.htm
qth.g6lz11tv.cn/771395.htm
qtg.g6lz11tv.cn/117575.htm
qtg.g6lz11tv.cn/599595.htm
qtg.g6lz11tv.cn/775175.htm
qtg.g6lz11tv.cn/919315.htm
qtg.g6lz11tv.cn/511315.htm
qtg.g6lz11tv.cn/537175.htm
qtg.g6lz11tv.cn/115755.htm
qtg.g6lz11tv.cn/115915.htm
qtg.g6lz11tv.cn/595395.htm
qtg.g6lz11tv.cn/577395.htm
qtf.g6lz11tv.cn/537975.htm
qtf.g6lz11tv.cn/993135.htm
qtf.g6lz11tv.cn/917335.htm
qtf.g6lz11tv.cn/919335.htm
qtf.g6lz11tv.cn/717915.htm
qtf.g6lz11tv.cn/791735.htm
qtf.g6lz11tv.cn/119775.htm
qtf.g6lz11tv.cn/591515.htm
qtf.g6lz11tv.cn/157115.htm
qtf.g6lz11tv.cn/757115.htm
qtd.g6lz11tv.cn/911175.htm
qtd.g6lz11tv.cn/999535.htm
qtd.g6lz11tv.cn/379915.htm
qtd.g6lz11tv.cn/395735.htm
qtd.g6lz11tv.cn/935395.htm
qtd.g6lz11tv.cn/313995.htm
qtd.g6lz11tv.cn/591535.htm
qtd.g6lz11tv.cn/357955.htm
qtd.g6lz11tv.cn/391915.htm
qtd.g6lz11tv.cn/599175.htm
qts.g6lz11tv.cn/111555.htm
qts.g6lz11tv.cn/139935.htm
qts.g6lz11tv.cn/957535.htm
qts.g6lz11tv.cn/395515.htm
qts.g6lz11tv.cn/137595.htm
qts.g6lz11tv.cn/519515.htm
qts.g6lz11tv.cn/391135.htm
qts.g6lz11tv.cn/739935.htm
qts.g6lz11tv.cn/357795.htm
qts.g6lz11tv.cn/371355.htm
qta.g6lz11tv.cn/577155.htm
qta.g6lz11tv.cn/519755.htm
qta.g6lz11tv.cn/733575.htm
qta.g6lz11tv.cn/191115.htm
qta.g6lz11tv.cn/157315.htm
qta.g6lz11tv.cn/151115.htm
qta.g6lz11tv.cn/799955.htm
qta.g6lz11tv.cn/553975.htm
qta.g6lz11tv.cn/775155.htm
qta.g6lz11tv.cn/775515.htm
qtp.g6lz11tv.cn/199195.htm
qtp.g6lz11tv.cn/139755.htm
qtp.g6lz11tv.cn/371515.htm
qtp.g6lz11tv.cn/791775.htm
qtp.g6lz11tv.cn/791375.htm
qtp.g6lz11tv.cn/997995.htm
qtp.g6lz11tv.cn/913195.htm
qtp.g6lz11tv.cn/353575.htm
qtp.g6lz11tv.cn/539935.htm
qtp.g6lz11tv.cn/133715.htm
qto.g6lz11tv.cn/359755.htm
qto.g6lz11tv.cn/755535.htm
qto.g6lz11tv.cn/991935.htm
qto.g6lz11tv.cn/997715.htm
qto.g6lz11tv.cn/191535.htm
qto.g6lz11tv.cn/313535.htm
qto.g6lz11tv.cn/795115.htm
qto.g6lz11tv.cn/159155.htm
qto.g6lz11tv.cn/991995.htm
qto.g6lz11tv.cn/773375.htm
qti.g6lz11tv.cn/779735.htm
qti.g6lz11tv.cn/715175.htm
qti.g6lz11tv.cn/573975.htm
qti.g6lz11tv.cn/171355.htm
qti.g6lz11tv.cn/531795.htm
qti.g6lz11tv.cn/711795.htm
qti.g6lz11tv.cn/739935.htm
qti.g6lz11tv.cn/519935.htm
qti.g6lz11tv.cn/359995.htm
qti.g6lz11tv.cn/553715.htm
qtu.g6lz11tv.cn/713715.htm
qtu.g6lz11tv.cn/535115.htm
qtu.g6lz11tv.cn/995155.htm
qtu.g6lz11tv.cn/171335.htm
qtu.g6lz11tv.cn/973935.htm
qtu.g6lz11tv.cn/791775.htm
qtu.g6lz11tv.cn/539575.htm
qtu.g6lz11tv.cn/773535.htm
qtu.g6lz11tv.cn/335575.htm
qtu.g6lz11tv.cn/757595.htm
qty.g6lz11tv.cn/975395.htm
qty.g6lz11tv.cn/919915.htm
qty.g6lz11tv.cn/573335.htm
qty.g6lz11tv.cn/319775.htm
qty.g6lz11tv.cn/559955.htm
qty.g6lz11tv.cn/159775.htm
qty.g6lz11tv.cn/357135.htm
qty.g6lz11tv.cn/997515.htm
qty.g6lz11tv.cn/195135.htm
qty.g6lz11tv.cn/717995.htm
qtt.g6lz11tv.cn/731155.htm
qtt.g6lz11tv.cn/177595.htm
qtt.g6lz11tv.cn/731975.htm
qtt.g6lz11tv.cn/799335.htm
qtt.g6lz11tv.cn/153795.htm
qtt.g6lz11tv.cn/179915.htm
qtt.g6lz11tv.cn/113175.htm
qtt.g6lz11tv.cn/133155.htm
qtt.g6lz11tv.cn/195335.htm
qtt.g6lz11tv.cn/373555.htm
qtr.g6lz11tv.cn/519775.htm
qtr.g6lz11tv.cn/593355.htm
qtr.g6lz11tv.cn/915735.htm
qtr.g6lz11tv.cn/157375.htm
qtr.g6lz11tv.cn/759975.htm
qtr.g6lz11tv.cn/199715.htm
qtr.g6lz11tv.cn/355195.htm
qtr.g6lz11tv.cn/939775.htm
qtr.g6lz11tv.cn/751975.htm
qtr.g6lz11tv.cn/719915.htm
qte.g6lz11tv.cn/159595.htm
qte.g6lz11tv.cn/999755.htm
qte.g6lz11tv.cn/139535.htm
qte.g6lz11tv.cn/353715.htm
qte.g6lz11tv.cn/171935.htm
qte.g6lz11tv.cn/557555.htm
qte.g6lz11tv.cn/713395.htm
qte.g6lz11tv.cn/519555.htm
qte.g6lz11tv.cn/173915.htm
qte.g6lz11tv.cn/737135.htm
qtw.g6lz11tv.cn/337115.htm
qtw.g6lz11tv.cn/593595.htm
qtw.g6lz11tv.cn/133135.htm
qtw.g6lz11tv.cn/379515.htm
qtw.g6lz11tv.cn/311595.htm
qtw.g6lz11tv.cn/157955.htm
qtw.g6lz11tv.cn/179755.htm
qtw.g6lz11tv.cn/595915.htm
qtw.g6lz11tv.cn/535755.htm
qtw.g6lz11tv.cn/593115.htm
qtq.g6lz11tv.cn/197595.htm
qtq.g6lz11tv.cn/791135.htm
qtq.g6lz11tv.cn/933375.htm
qtq.g6lz11tv.cn/379995.htm
qtq.g6lz11tv.cn/919795.htm
qtq.g6lz11tv.cn/595555.htm
qtq.g6lz11tv.cn/951195.htm
qtq.g6lz11tv.cn/353515.htm
qtq.g6lz11tv.cn/151535.htm
qtq.g6lz11tv.cn/715795.htm
qrtv.g6lz11tv.cn/733335.htm
qrtv.g6lz11tv.cn/959595.htm
qrtv.g6lz11tv.cn/755795.htm
qrtv.g6lz11tv.cn/797575.htm
qrtv.g6lz11tv.cn/151975.htm
qrtv.g6lz11tv.cn/371515.htm
qrtv.g6lz11tv.cn/137715.htm
qrtv.g6lz11tv.cn/719115.htm
qrtv.g6lz11tv.cn/199155.htm
qrtv.g6lz11tv.cn/937315.htm
qrn.g6lz11tv.cn/919535.htm
qrn.g6lz11tv.cn/117575.htm
qrn.g6lz11tv.cn/775375.htm
qrn.g6lz11tv.cn/911355.htm
qrn.g6lz11tv.cn/799355.htm
qrn.g6lz11tv.cn/131355.htm
qrn.g6lz11tv.cn/535735.htm
qrn.g6lz11tv.cn/777155.htm
qrn.g6lz11tv.cn/733915.htm
qrn.g6lz11tv.cn/533775.htm
qrb.g6lz11tv.cn/917915.htm
qrb.g6lz11tv.cn/137535.htm
qrb.g6lz11tv.cn/351315.htm
qrb.g6lz11tv.cn/579395.htm
qrb.g6lz11tv.cn/173935.htm
qrb.g6lz11tv.cn/595935.htm
qrb.g6lz11tv.cn/175975.htm
qrb.g6lz11tv.cn/339715.htm
qrb.g6lz11tv.cn/595775.htm
qrb.g6lz11tv.cn/955555.htm
qrv.g6lz11tv.cn/773335.htm
qrv.g6lz11tv.cn/599315.htm
qrv.g6lz11tv.cn/197935.htm
qrv.g6lz11tv.cn/953575.htm
qrv.g6lz11tv.cn/391575.htm
qrv.g6lz11tv.cn/797915.htm
qrv.g6lz11tv.cn/991715.htm
qrv.g6lz11tv.cn/151735.htm
qrv.g6lz11tv.cn/159595.htm
qrv.g6lz11tv.cn/775795.htm
qrc.g6lz11tv.cn/191595.htm
qrc.g6lz11tv.cn/713575.htm
qrc.g6lz11tv.cn/711135.htm
qrc.g6lz11tv.cn/313555.htm
qrc.g6lz11tv.cn/517955.htm
qrc.g6lz11tv.cn/939775.htm
qrc.g6lz11tv.cn/999555.htm
qrc.g6lz11tv.cn/799935.htm
qrc.g6lz11tv.cn/331595.htm
qrc.g6lz11tv.cn/771975.htm
qrx.g6lz11tv.cn/713975.htm
qrx.g6lz11tv.cn/997155.htm
qrx.g6lz11tv.cn/735995.htm
qrx.g6lz11tv.cn/153375.htm
qrx.g6lz11tv.cn/159775.htm
qrx.g6lz11tv.cn/737335.htm
qrx.g6lz11tv.cn/775935.htm
qrx.g6lz11tv.cn/973735.htm
qrx.g6lz11tv.cn/155555.htm
qrx.g6lz11tv.cn/799955.htm
qrz.g6lz11tv.cn/337195.htm
qrz.g6lz11tv.cn/717395.htm
qrz.g6lz11tv.cn/175115.htm
qrz.g6lz11tv.cn/331315.htm
qrz.g6lz11tv.cn/915515.htm
qrz.g6lz11tv.cn/191575.htm
qrz.g6lz11tv.cn/531375.htm
qrz.g6lz11tv.cn/777535.htm
qrz.g6lz11tv.cn/313395.htm
qrz.g6lz11tv.cn/717955.htm
qrl.g6lz11tv.cn/995995.htm
qrl.g6lz11tv.cn/995155.htm
qrl.g6lz11tv.cn/517535.htm
qrl.g6lz11tv.cn/755935.htm
qrl.g6lz11tv.cn/553135.htm
qrl.g6lz11tv.cn/177955.htm
qrl.g6lz11tv.cn/351775.htm
qrl.g6lz11tv.cn/919735.htm
qrl.g6lz11tv.cn/597375.htm
qrl.g6lz11tv.cn/777355.htm
qrk.g6lz11tv.cn/793155.htm
qrk.g6lz11tv.cn/797155.htm
qrk.g6lz11tv.cn/997335.htm
qrk.g6lz11tv.cn/737935.htm
qrk.g6lz11tv.cn/371395.htm
qrk.g6lz11tv.cn/571775.htm
qrk.g6lz11tv.cn/999535.htm
qrk.g6lz11tv.cn/995555.htm
qrk.g6lz11tv.cn/999555.htm
qrk.g6lz11tv.cn/997735.htm
qrj.g6lz11tv.cn/959315.htm
qrj.g6lz11tv.cn/575795.htm
qrj.g6lz11tv.cn/539315.htm
qrj.g6lz11tv.cn/155795.htm
qrj.g6lz11tv.cn/595755.htm
qrj.g6lz11tv.cn/191535.htm
qrj.g6lz11tv.cn/537555.htm
qrj.g6lz11tv.cn/779595.htm
qrj.g6lz11tv.cn/557155.htm
qrj.g6lz11tv.cn/195375.htm
qrh.g6lz11tv.cn/157155.htm
qrh.g6lz11tv.cn/311315.htm
qrh.g6lz11tv.cn/319915.htm
qrh.g6lz11tv.cn/397775.htm
qrh.g6lz11tv.cn/339915.htm
qrh.g6lz11tv.cn/111515.htm
qrh.g6lz11tv.cn/395195.htm
qrh.g6lz11tv.cn/719795.htm
qrh.g6lz11tv.cn/735155.htm
qrh.g6lz11tv.cn/717955.htm
qrg.g6lz11tv.cn/173795.htm
qrg.g6lz11tv.cn/597155.htm
qrg.g6lz11tv.cn/117315.htm
qrg.g6lz11tv.cn/173935.htm
qrg.g6lz11tv.cn/939995.htm
qrg.g6lz11tv.cn/157755.htm
qrg.g6lz11tv.cn/377915.htm
qrg.g6lz11tv.cn/995175.htm
qrg.g6lz11tv.cn/935795.htm
qrg.g6lz11tv.cn/135155.htm
qrf.g6lz11tv.cn/995575.htm
qrf.g6lz11tv.cn/719335.htm
qrf.g6lz11tv.cn/193995.htm
qrf.g6lz11tv.cn/995595.htm
qrf.g6lz11tv.cn/571355.htm
qrf.g6lz11tv.cn/911795.htm
qrf.g6lz11tv.cn/551595.htm
qrf.g6lz11tv.cn/391795.htm
qrf.g6lz11tv.cn/513375.htm
qrf.g6lz11tv.cn/737195.htm
qrd.g6lz11tv.cn/933375.htm
qrd.g6lz11tv.cn/131395.htm
qrd.g6lz11tv.cn/371395.htm
qrd.g6lz11tv.cn/559155.htm
qrd.g6lz11tv.cn/195715.htm
qrd.g6lz11tv.cn/751135.htm
qrd.g6lz11tv.cn/971195.htm
qrd.g6lz11tv.cn/715975.htm
qrd.g6lz11tv.cn/771795.htm
qrd.g6lz11tv.cn/739575.htm
qrs.g6lz11tv.cn/913315.htm
qrs.g6lz11tv.cn/377995.htm
qrs.g6lz11tv.cn/939175.htm
qrs.g6lz11tv.cn/739915.htm
qrs.g6lz11tv.cn/973395.htm
qrs.g6lz11tv.cn/773535.htm
qrs.g6lz11tv.cn/533595.htm
qrs.g6lz11tv.cn/793155.htm
qrs.g6lz11tv.cn/799135.htm
qrs.g6lz11tv.cn/359115.htm
qra.g6lz11tv.cn/719155.htm
qra.g6lz11tv.cn/999935.htm
qra.g6lz11tv.cn/733355.htm
qra.g6lz11tv.cn/995195.htm
qra.g6lz11tv.cn/995995.htm
qra.g6lz11tv.cn/391395.htm
qra.g6lz11tv.cn/511575.htm
qra.g6lz11tv.cn/977175.htm
qra.g6lz11tv.cn/795195.htm
qra.g6lz11tv.cn/775355.htm
qrp.g6lz11tv.cn/979335.htm
qrp.g6lz11tv.cn/935335.htm
qrp.g6lz11tv.cn/399535.htm
qrp.g6lz11tv.cn/999535.htm
qrp.g6lz11tv.cn/751955.htm
qrp.g6lz11tv.cn/591115.htm
qrp.g6lz11tv.cn/593515.htm
qrp.g6lz11tv.cn/135315.htm
qrp.g6lz11tv.cn/335715.htm
qrp.g6lz11tv.cn/155195.htm
qro.g6lz11tv.cn/517355.htm
qro.g6lz11tv.cn/511335.htm
