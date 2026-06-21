铱邮邮祭沧


"""
Python 音频播放与录制详解
包含：pyaudio 流式读写、sounddevice 播放录制、wave 模块 WAV 操作
"""
import numpy as np
import wave
import threading
import time
from datetime import datetime


def generate_sine_wave(freq=440, duration=2.0, sample_rate=44100, amplitude=0.5):
    """生成指定频率和时长的正弦波信号（float32 数组）"""
    t = np.linspace(0, duration, int(sample_rate * duration), endpoint=False)
    # 440Hz 的标准 A 音，可叠加谐波使音色更丰富
    wave_data = amplitude * np.sin(2 * np.pi * freq * t)
    # 添加三次谐波（音色更饱满）
    wave_data += 0.25 * amplitude * np.sin(2 * np.pi * freq * 3 * t)
    # 添加五次谐波
    wave_data += 0.15 * amplitude * np.sin(2 * np.pi * freq * 5 * t)
    # 归一化到 [-1, 1] 范围
    wave_data = wave_data / np.max(np.abs(wave_data)) * amplitude
    return wave_data, sample_rate


def save_wav_file(filename, data, sample_rate):
    """将 numpy 数组保存为 WAV 文件（16-bit PCM）"""
    with wave.open(filename, 'w') as wf:
        wf.setnchannels(1)  # 单声道
        wf.setsampwidth(2)  # 16 位 = 2 字节
        wf.setframerate(sample_rate)
        # 将 float32 转为 16-bit 整数
        int_data = np.int16(data * 32767)
        wf.writeframes(int_data.tobytes())
    print(f"WAV 文件已保存: {filename}")


def read_wav_file(filename):
    """读取 WAV 文件返回 numpy 数组和采样率"""
    with wave.open(filename, 'r') as wf:
        sample_rate = wf.getframerate()
        n_samples = wf.getnframes()
        n_channels = wf.getnchannels()
        sampwidth = wf.getsampwidth()
        frames = wf.readframes(n_samples)
        # 根据采样宽度解码
        if sampwidth == 2:
            dtype = np.int16
        elif sampwidth == 4:
            dtype = np.int32
        else:
            dtype = np.int16
        data = np.frombuffer(frames, dtype=dtype)
        # 如果多声道取平均转单声道
        if n_channels > 1:
            data = data.reshape(-1, n_channels).mean(axis=1)
        # 转为 float32 并归一化
        data = data.astype(np.float32) / 32767.0
    return data, sample_rate


# ========== 1. wave 模块：保存与读取 WAV 文件 ==========
print("=" * 50)
print("1. wave 模块：WAV 文件读写")
print("=" * 50)

# 生成一个 440Hz 的测试音频并保存为 WAV
audio_data, fs = generate_sine_wave(freq=440, duration=2.0, sample_rate=44100)
save_wav_file('test_output.wav', audio_data, fs)

# 读取刚才保存的 WAV 文件并验证
loaded_data, loaded_fs = read_wav_file('test_output.wav')
print(f"读取验证: 采样率={loaded_fs}Hz, 数据长度={len(loaded_data)}")
print(f"数据范围: [{loaded_data.min():.4f}, {loaded_data.max():.4f}]")

# ========== 2. pyaudio 播放音频 ==========
print("\n" + "=" * 50)
print("2. pyaudio：流式播放与录制")
print("=" * 50)

try:
    import pyaudio
    pa = pyaudio.PyAudio()
    print(f"可用音频设备数: {pa.get_device_count()}")

    # 2a. 使用 pyaudio 播放音频流
    def play_with_pyaudio(data, sample_rate):
        """将 numpy 数组通过 pyaudio 播放"""
        stream = pa.open(
            format=pyaudio.paFloat32,  # 32-bit float
            channels=1,                # 单声道
            rate=sample_rate,
            output=True,
            frames_per_buffer=1024
        )
        # 分批写入音频数据（流式播放）
        chunk_size = 1024
        for i in range(0, len(data), chunk_size):
            chunk = data[i:i + chunk_size]
            stream.write(chunk.astype(np.float32).tobytes())
        stream.stop_stream()
        stream.close()

    print("正在播放测试音调（440Hz + 谐波，2 秒）...")
    # 在新线程中播放，避免阻塞
    play_thread = threading.Thread(
        target=play_with_pyaudio, args=(audio_data, fs)
    )
    play_thread.start()
    play_thread.join()
    print("播放完成。")

    # 2b. 使用 pyaudio 录制音频
    def record_with_pyaudio(duration=3.0, sample_rate=44100):
        """录制指定时长的音频并返回 numpy 数组"""
        stream = pa.open(
            format=pyaudio.paInt16,
            channels=1,
            rate=sample_rate,
            input=True,
            frames_per_buffer=1024
        )
        frames = []
        total_chunks = int(sample_rate / 1024 * duration)
        for _ in range(total_chunks):
            chunk_data = stream.read(1024)
            frames.append(np.frombuffer(chunk_data, dtype=np.int16))
        stream.stop_stream()
        stream.close()
        # 合并所有 chunk 并归一化
        recording = np.concatenate(frames).astype(np.float32) / 32767.0
        return recording

    print("正在录制 2 秒音频（请对着麦克风说话）...")
    recorded = record_with_pyaudio(duration=2.0)
    save_wav_file('pyaudio_recording.wav', recorded, fs)
    print(f"录制完成，共 {len(recorded)} 个采样点。")

    pa.terminate()
except ImportError:
    print("pyaudio 未安装，跳过 pyaudio 相关演示。")

# ========== 3. sounddevice 播放与录制 ==========
print("\n" + "=" * 50)
print("3. sounddevice：高级音频 I/O")
print("=" * 50)

try:
    import sounddevice as sd
    print(f"默认输入设备: {sd.default.device[0]}")
    print(f"默认输出设备: {sd.default.device[1]}")

    # 3a. 播放音频（非阻塞方式）
    print("正在播放音频（sounddevice）...")
    sd.play(audio_data, fs)
    sd.wait()  # 等待播放结束
    print("播放完成。")

    # 3b. 录制音频
    print("正在录制 2 秒音频（sounddevice）...")
    rec = sd.rec(int(2.0 * fs), samplerate=fs, channels=1, dtype='float32')
    sd.wait()  # 等待录制完成
    rec = rec.flatten()
    save_wav_file('sounddevice_recording.wav', rec, fs)
    print(f"录制完成，共 {len(rec)} 个采样点。")

    # 3c. InputStream 实时回调处理
    print("InputStream 实时音频流演示（持续 1 秒）...")
    stream_data = []

    def audio_callback(indata, frames, time_info, status):
        """音频流回调函数：每帧被调用一次"""
        if status:
            print(f"音频状态: {status}")
        stream_data.append(indata.copy())

    stream = sd.InputStream(
        samplerate=fs, channels=1, callback=audio_callback
    )
    with stream:
        time.sleep(1.0)  # 采集 1 秒数据
    if stream_data:
        result = np.concatenate(stream_data).flatten()
        print(f"InputStream 采集了 {len(result)} 个采样点。")

except ImportError:
    print("sounddevice 未安装，跳过 sounddevice 相关演示。")

# ========== 4. 综合示例：录制并马上回放 ==========
print("\n" + "=" * 50)
print("4. 综合示例：录制并立即回放")
print("=" * 50)
# 使用已生成的音频模拟"录制后回放"流程
print("生成测试音调并回放...")
test_tone, fs = generate_sine_wave(freq=523, duration=1.0)  # C5 音
save_wav_file('c5_tone.wav', test_tone, fs)
print("回放演示完成。")

print("\n音频播放与录制演示全部完成。")
print("涵盖技术: wave / pyaudio (播放+录制) / sounddevice (播放+录制+InputStream)")

惺弊诶谱笛甘珊僦绕净刂蚁偬卸忻

m.qpw.hxbsg.cn/79551.Doc
m.qpw.hxbsg.cn/68042.Doc
m.qpq.hxbsg.cn/26884.Doc
m.qpq.hxbsg.cn/44028.Doc
m.qpq.hxbsg.cn/62426.Doc
m.qpq.hxbsg.cn/22240.Doc
m.qpq.hxbsg.cn/06224.Doc
m.qpq.hxbsg.cn/80626.Doc
m.qpq.hxbsg.cn/68804.Doc
m.qpq.hxbsg.cn/00424.Doc
m.qpq.hxbsg.cn/80608.Doc
m.qpq.hxbsg.cn/06808.Doc
m.qpq.hxbsg.cn/88668.Doc
m.qpq.hxbsg.cn/48262.Doc
m.qpq.hxbsg.cn/24684.Doc
m.qpq.hxbsg.cn/35535.Doc
m.qpq.hxbsg.cn/66826.Doc
m.qpq.hxbsg.cn/17717.Doc
m.qpq.hxbsg.cn/39911.Doc
m.qpq.hxbsg.cn/86606.Doc
m.qpq.hxbsg.cn/88400.Doc
m.qpq.hxbsg.cn/64622.Doc
m.qom.hxbsg.cn/60264.Doc
m.qom.hxbsg.cn/60664.Doc
m.qom.hxbsg.cn/82020.Doc
m.qom.hxbsg.cn/57191.Doc
m.qom.hxbsg.cn/15337.Doc
m.qom.hxbsg.cn/99551.Doc
m.qom.hxbsg.cn/64686.Doc
m.qom.hxbsg.cn/48028.Doc
m.qom.hxbsg.cn/64642.Doc
m.qom.hxbsg.cn/60264.Doc
m.qom.hxbsg.cn/66442.Doc
m.qom.hxbsg.cn/24248.Doc
m.qom.hxbsg.cn/22040.Doc
m.qom.hxbsg.cn/99117.Doc
m.qom.hxbsg.cn/44606.Doc
m.qom.hxbsg.cn/64682.Doc
m.qom.hxbsg.cn/84206.Doc
m.qom.hxbsg.cn/48860.Doc
m.qom.hxbsg.cn/35177.Doc
m.qom.hxbsg.cn/37375.Doc
m.qon.hxbsg.cn/02042.Doc
m.qon.hxbsg.cn/64688.Doc
m.qon.hxbsg.cn/80406.Doc
m.qon.hxbsg.cn/40244.Doc
m.qon.hxbsg.cn/37759.Doc
m.qon.hxbsg.cn/59799.Doc
m.qon.hxbsg.cn/06620.Doc
m.qon.hxbsg.cn/82860.Doc
m.qon.hxbsg.cn/20422.Doc
m.qon.hxbsg.cn/99597.Doc
m.qon.hxbsg.cn/86200.Doc
m.qon.hxbsg.cn/19175.Doc
m.qon.hxbsg.cn/66240.Doc
m.qon.hxbsg.cn/40446.Doc
m.qon.hxbsg.cn/20820.Doc
m.qon.hxbsg.cn/44046.Doc
m.qon.hxbsg.cn/66668.Doc
m.qon.hxbsg.cn/08242.Doc
m.qon.hxbsg.cn/08228.Doc
m.qon.hxbsg.cn/06286.Doc
m.qob.hxbsg.cn/48060.Doc
m.qob.hxbsg.cn/00646.Doc
m.qob.hxbsg.cn/46440.Doc
m.qob.hxbsg.cn/24800.Doc
m.qob.hxbsg.cn/80622.Doc
m.qob.hxbsg.cn/80266.Doc
m.qob.hxbsg.cn/42686.Doc
m.qob.hxbsg.cn/26064.Doc
m.qob.hxbsg.cn/06228.Doc
m.qob.hxbsg.cn/62620.Doc
m.qob.hxbsg.cn/66006.Doc
m.qob.hxbsg.cn/88488.Doc
m.qob.hxbsg.cn/44486.Doc
m.qob.hxbsg.cn/20460.Doc
m.qob.hxbsg.cn/15737.Doc
m.qob.hxbsg.cn/19951.Doc
m.qob.hxbsg.cn/80288.Doc
m.qob.hxbsg.cn/51139.Doc
m.qob.hxbsg.cn/06826.Doc
m.qob.hxbsg.cn/28060.Doc
m.qov.hxbsg.cn/75353.Doc
m.qov.hxbsg.cn/42444.Doc
m.qov.hxbsg.cn/60048.Doc
m.qov.hxbsg.cn/46408.Doc
m.qov.hxbsg.cn/26660.Doc
m.qov.hxbsg.cn/73399.Doc
m.qov.hxbsg.cn/75535.Doc
m.qov.hxbsg.cn/28460.Doc
m.qov.hxbsg.cn/40400.Doc
m.qov.hxbsg.cn/08684.Doc
m.qov.hxbsg.cn/02246.Doc
m.qov.hxbsg.cn/44242.Doc
m.qov.hxbsg.cn/24606.Doc
m.qov.hxbsg.cn/08422.Doc
m.qov.hxbsg.cn/48086.Doc
m.qov.hxbsg.cn/68406.Doc
m.qov.hxbsg.cn/97935.Doc
m.qov.hxbsg.cn/22264.Doc
m.qov.hxbsg.cn/80640.Doc
m.qov.hxbsg.cn/20868.Doc
m.qoc.hxbsg.cn/68688.Doc
m.qoc.hxbsg.cn/40404.Doc
m.qoc.hxbsg.cn/95793.Doc
m.qoc.hxbsg.cn/64868.Doc
m.qoc.hxbsg.cn/22684.Doc
m.qoc.hxbsg.cn/68448.Doc
m.qoc.hxbsg.cn/00468.Doc
m.qoc.hxbsg.cn/60682.Doc
m.qoc.hxbsg.cn/86620.Doc
m.qoc.hxbsg.cn/44464.Doc
m.qoc.hxbsg.cn/28680.Doc
m.qoc.hxbsg.cn/99597.Doc
m.qoc.hxbsg.cn/66282.Doc
m.qoc.hxbsg.cn/24620.Doc
m.qoc.hxbsg.cn/24848.Doc
m.qoc.hxbsg.cn/19971.Doc
m.qoc.hxbsg.cn/97511.Doc
m.qoc.hxbsg.cn/42406.Doc
m.qoc.hxbsg.cn/84062.Doc
m.qoc.hxbsg.cn/57353.Doc
m.qox.hxbsg.cn/75739.Doc
m.qox.hxbsg.cn/37199.Doc
m.qox.hxbsg.cn/77759.Doc
m.qox.hxbsg.cn/13137.Doc
m.qox.hxbsg.cn/80842.Doc
m.qox.hxbsg.cn/20682.Doc
m.qox.hxbsg.cn/00488.Doc
m.qox.hxbsg.cn/40660.Doc
m.qox.hxbsg.cn/62462.Doc
m.qox.hxbsg.cn/28284.Doc
m.qox.hxbsg.cn/86286.Doc
m.qox.hxbsg.cn/62466.Doc
m.qox.hxbsg.cn/40064.Doc
m.qox.hxbsg.cn/28282.Doc
m.qox.hxbsg.cn/95951.Doc
m.qox.hxbsg.cn/28042.Doc
m.qox.hxbsg.cn/84826.Doc
m.qox.hxbsg.cn/06608.Doc
m.qox.hxbsg.cn/93719.Doc
m.qox.hxbsg.cn/64246.Doc
m.qoz.hxbsg.cn/02680.Doc
m.qoz.hxbsg.cn/48800.Doc
m.qoz.hxbsg.cn/82888.Doc
m.qoz.hxbsg.cn/48602.Doc
m.qoz.hxbsg.cn/40028.Doc
m.qoz.hxbsg.cn/04402.Doc
m.qoz.hxbsg.cn/15557.Doc
m.qoz.hxbsg.cn/97557.Doc
m.qoz.hxbsg.cn/93551.Doc
m.qoz.hxbsg.cn/02648.Doc
m.qoz.hxbsg.cn/60220.Doc
m.qoz.hxbsg.cn/04622.Doc
m.qoz.hxbsg.cn/04620.Doc
m.qoz.hxbsg.cn/22086.Doc
m.qoz.hxbsg.cn/20264.Doc
m.qoz.hxbsg.cn/60224.Doc
m.qoz.hxbsg.cn/24484.Doc
m.qoz.hxbsg.cn/26646.Doc
m.qoz.hxbsg.cn/15353.Doc
m.qoz.hxbsg.cn/26428.Doc
m.qol.hxbsg.cn/40806.Doc
m.qol.hxbsg.cn/44688.Doc
m.qol.hxbsg.cn/17391.Doc
m.qol.hxbsg.cn/44462.Doc
m.qol.hxbsg.cn/86848.Doc
m.qol.hxbsg.cn/40264.Doc
m.qol.hxbsg.cn/48648.Doc
m.qol.hxbsg.cn/08242.Doc
m.qol.hxbsg.cn/40268.Doc
m.qol.hxbsg.cn/40202.Doc
m.qol.hxbsg.cn/97711.Doc
m.qol.hxbsg.cn/97591.Doc
m.qol.hxbsg.cn/06640.Doc
m.qol.hxbsg.cn/75995.Doc
m.qol.hxbsg.cn/06666.Doc
m.qol.hxbsg.cn/26422.Doc
m.qol.hxbsg.cn/84884.Doc
m.qol.hxbsg.cn/46684.Doc
m.qol.hxbsg.cn/24844.Doc
m.qol.hxbsg.cn/60088.Doc
m.qok.hxbsg.cn/31375.Doc
m.qok.hxbsg.cn/48684.Doc
m.qok.hxbsg.cn/00464.Doc
m.qok.hxbsg.cn/20422.Doc
m.qok.hxbsg.cn/42080.Doc
m.qok.hxbsg.cn/86066.Doc
m.qok.hxbsg.cn/86662.Doc
m.qok.hxbsg.cn/64664.Doc
m.qok.hxbsg.cn/99375.Doc
m.qok.hxbsg.cn/99159.Doc
m.qok.hxbsg.cn/22242.Doc
m.qok.hxbsg.cn/26246.Doc
m.qok.hxbsg.cn/51131.Doc
m.qok.hxbsg.cn/95179.Doc
m.qok.hxbsg.cn/68280.Doc
m.qok.hxbsg.cn/60822.Doc
m.qok.hxbsg.cn/91535.Doc
m.qok.hxbsg.cn/44426.Doc
m.qok.hxbsg.cn/59373.Doc
m.qok.hxbsg.cn/86222.Doc
m.qoj.hxbsg.cn/46840.Doc
m.qoj.hxbsg.cn/44228.Doc
m.qoj.hxbsg.cn/71537.Doc
m.qoj.hxbsg.cn/02082.Doc
m.qoj.hxbsg.cn/13999.Doc
m.qoj.hxbsg.cn/60048.Doc
m.qoj.hxbsg.cn/46822.Doc
m.qoj.hxbsg.cn/44226.Doc
m.qoj.hxbsg.cn/44648.Doc
m.qoj.hxbsg.cn/28606.Doc
m.qoj.hxbsg.cn/86800.Doc
m.qoj.hxbsg.cn/64424.Doc
m.qoj.hxbsg.cn/17775.Doc
m.qoj.hxbsg.cn/15111.Doc
m.qoj.hxbsg.cn/84206.Doc
m.qoj.hxbsg.cn/86822.Doc
m.qoj.hxbsg.cn/35157.Doc
m.qoj.hxbsg.cn/46400.Doc
m.qoj.hxbsg.cn/46008.Doc
m.qoj.hxbsg.cn/60648.Doc
m.qoh.hxbsg.cn/40066.Doc
m.qoh.hxbsg.cn/11917.Doc
m.qoh.hxbsg.cn/68620.Doc
m.qoh.hxbsg.cn/19553.Doc
m.qoh.hxbsg.cn/06628.Doc
m.qoh.hxbsg.cn/53991.Doc
m.qoh.hxbsg.cn/71913.Doc
m.qoh.hxbsg.cn/80682.Doc
m.qoh.hxbsg.cn/82048.Doc
m.qoh.hxbsg.cn/46846.Doc
m.qoh.hxbsg.cn/15993.Doc
m.qoh.hxbsg.cn/00006.Doc
m.qoh.hxbsg.cn/28664.Doc
m.qoh.hxbsg.cn/20422.Doc
m.qoh.hxbsg.cn/88080.Doc
m.qoh.hxbsg.cn/00248.Doc
m.qoh.hxbsg.cn/33997.Doc
m.qoh.hxbsg.cn/06064.Doc
m.qoh.hxbsg.cn/35139.Doc
m.qoh.hxbsg.cn/02648.Doc
m.qog.hxbsg.cn/00444.Doc
m.qog.hxbsg.cn/08048.Doc
m.qog.hxbsg.cn/28266.Doc
m.qog.hxbsg.cn/82642.Doc
m.qog.hxbsg.cn/46060.Doc
m.qog.hxbsg.cn/48442.Doc
m.qog.hxbsg.cn/57179.Doc
m.qog.hxbsg.cn/75573.Doc
m.qog.hxbsg.cn/68020.Doc
m.qog.hxbsg.cn/26426.Doc
m.qog.hxbsg.cn/39331.Doc
m.qog.hxbsg.cn/42008.Doc
m.qog.hxbsg.cn/44240.Doc
m.qog.hxbsg.cn/02086.Doc
m.qog.hxbsg.cn/46602.Doc
m.qog.hxbsg.cn/44880.Doc
m.qog.hxbsg.cn/04062.Doc
m.qog.hxbsg.cn/82642.Doc
m.qog.hxbsg.cn/42266.Doc
m.qog.hxbsg.cn/95151.Doc
m.qof.hxbsg.cn/42004.Doc
m.qof.hxbsg.cn/68046.Doc
m.qof.hxbsg.cn/51155.Doc
m.qof.hxbsg.cn/33355.Doc
m.qof.hxbsg.cn/37515.Doc
m.qof.hxbsg.cn/82820.Doc
m.qof.hxbsg.cn/04226.Doc
m.qof.hxbsg.cn/48644.Doc
m.qof.hxbsg.cn/28686.Doc
m.qof.hxbsg.cn/00264.Doc
m.qof.hxbsg.cn/95715.Doc
m.qof.hxbsg.cn/17971.Doc
m.qof.hxbsg.cn/68680.Doc
m.qof.hxbsg.cn/62802.Doc
m.qof.hxbsg.cn/02080.Doc
m.qof.hxbsg.cn/62800.Doc
m.qof.hxbsg.cn/99351.Doc
m.qof.hxbsg.cn/44686.Doc
m.qof.hxbsg.cn/82028.Doc
m.qof.hxbsg.cn/00286.Doc
m.qod.hxbsg.cn/11535.Doc
m.qod.hxbsg.cn/68808.Doc
m.qod.hxbsg.cn/86622.Doc
m.qod.hxbsg.cn/80620.Doc
m.qod.hxbsg.cn/37317.Doc
m.qod.hxbsg.cn/66864.Doc
m.qod.hxbsg.cn/66420.Doc
m.qod.hxbsg.cn/46426.Doc
m.qod.hxbsg.cn/15939.Doc
m.qod.hxbsg.cn/95111.Doc
m.qod.hxbsg.cn/24240.Doc
m.qod.hxbsg.cn/84448.Doc
m.qod.hxbsg.cn/02806.Doc
m.qod.hxbsg.cn/80426.Doc
m.qod.hxbsg.cn/02686.Doc
m.qod.hxbsg.cn/22462.Doc
m.qod.hxbsg.cn/51597.Doc
m.qod.hxbsg.cn/31713.Doc
m.qod.hxbsg.cn/22200.Doc
m.qod.hxbsg.cn/26866.Doc
m.qos.hxbsg.cn/08684.Doc
m.qos.hxbsg.cn/68466.Doc
m.qos.hxbsg.cn/02264.Doc
m.qos.hxbsg.cn/20040.Doc
m.qos.hxbsg.cn/44828.Doc
m.qos.hxbsg.cn/39355.Doc
m.qos.hxbsg.cn/02686.Doc
m.qos.hxbsg.cn/80880.Doc
m.qos.hxbsg.cn/26480.Doc
m.qos.hxbsg.cn/00886.Doc
m.qos.hxbsg.cn/68646.Doc
m.qos.hxbsg.cn/60208.Doc
m.qos.hxbsg.cn/88086.Doc
m.qos.hxbsg.cn/28464.Doc
m.qos.hxbsg.cn/40860.Doc
m.qos.hxbsg.cn/06606.Doc
m.qos.hxbsg.cn/57973.Doc
m.qos.hxbsg.cn/64624.Doc
m.qos.hxbsg.cn/84000.Doc
m.qos.hxbsg.cn/66628.Doc
m.qoa.hxbsg.cn/99513.Doc
m.qoa.hxbsg.cn/42088.Doc
m.qoa.hxbsg.cn/00624.Doc
m.qoa.hxbsg.cn/62208.Doc
m.qoa.hxbsg.cn/42444.Doc
m.qoa.hxbsg.cn/08420.Doc
m.qoa.hxbsg.cn/13795.Doc
m.qoa.hxbsg.cn/53979.Doc
m.qoa.hxbsg.cn/62208.Doc
m.qoa.hxbsg.cn/40802.Doc
m.qoa.hxbsg.cn/57733.Doc
m.qoa.hxbsg.cn/28608.Doc
m.qoa.hxbsg.cn/75717.Doc
m.qoa.hxbsg.cn/93319.Doc
m.qoa.hxbsg.cn/42004.Doc
m.qoa.hxbsg.cn/93977.Doc
m.qoa.hxbsg.cn/40240.Doc
m.qoa.hxbsg.cn/06060.Doc
m.qoa.hxbsg.cn/00408.Doc
m.qoa.hxbsg.cn/06806.Doc
m.qop.hxbsg.cn/24642.Doc
m.qop.hxbsg.cn/75599.Doc
m.qop.hxbsg.cn/97591.Doc
m.qop.hxbsg.cn/13575.Doc
m.qop.hxbsg.cn/08088.Doc
m.qop.hxbsg.cn/88244.Doc
m.qop.hxbsg.cn/02608.Doc
m.qop.hxbsg.cn/68602.Doc
m.qop.hxbsg.cn/20846.Doc
m.qop.hxbsg.cn/77339.Doc
m.qop.hxbsg.cn/73971.Doc
m.qop.hxbsg.cn/13319.Doc
m.qop.hxbsg.cn/42648.Doc
m.qop.hxbsg.cn/13999.Doc
m.qop.hxbsg.cn/39533.Doc
m.qop.hxbsg.cn/19397.Doc
m.qop.hxbsg.cn/75535.Doc
m.qop.hxbsg.cn/04646.Doc
m.qop.hxbsg.cn/06826.Doc
m.qop.hxbsg.cn/79131.Doc
m.qoo.hxbsg.cn/17175.Doc
m.qoo.hxbsg.cn/26202.Doc
m.qoo.hxbsg.cn/46266.Doc
m.qoo.hxbsg.cn/79539.Doc
m.qoo.hxbsg.cn/22040.Doc
m.qoo.hxbsg.cn/44602.Doc
m.qoo.hxbsg.cn/40288.Doc
m.qoo.hxbsg.cn/51193.Doc
m.qoo.hxbsg.cn/44206.Doc
m.qoo.hxbsg.cn/68628.Doc
m.qoo.hxbsg.cn/44286.Doc
m.qoo.hxbsg.cn/64086.Doc
m.qoo.hxbsg.cn/48466.Doc
m.qoo.hxbsg.cn/15339.Doc
m.qoo.hxbsg.cn/06828.Doc
m.qoo.hxbsg.cn/46860.Doc
m.qoo.hxbsg.cn/06826.Doc
m.qoo.hxbsg.cn/19535.Doc
m.qoo.hxbsg.cn/48246.Doc
m.qoo.hxbsg.cn/08264.Doc
m.qoi.hxbsg.cn/08842.Doc
m.qoi.hxbsg.cn/08004.Doc
m.qoi.hxbsg.cn/40008.Doc
m.qoi.hxbsg.cn/04440.Doc
m.qoi.hxbsg.cn/84482.Doc
m.qoi.hxbsg.cn/40268.Doc
m.qoi.hxbsg.cn/20082.Doc
m.qoi.hxbsg.cn/24448.Doc
m.qoi.hxbsg.cn/42826.Doc
m.qoi.hxbsg.cn/00624.Doc
m.qoi.hxbsg.cn/68820.Doc
m.qoi.hxbsg.cn/11751.Doc
m.qoi.hxbsg.cn/00402.Doc
