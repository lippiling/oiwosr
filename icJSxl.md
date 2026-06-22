傥百滔卦豪


"""
Python OpenCV 视频处理基础详解
包含：VideoCapture 读取、VideoWriter 写入、逐帧处理、FPS 控制
"""
import cv2
import numpy as np
import time
import os


def process_frame(frame, effect='gray'):
    """对单帧图像应用不同的处理效果"""
    if effect == 'gray':
        # 转为灰度图并转回三通道以保持视频格式
        gray = cv2.cvtColor(frame, cv2.COLOR_BGR2GRAY)
        return cv2.cvtColor(gray, cv2.COLOR_GRAY2BGR)
    elif effect == 'edge':
        # Canny 边缘检测效果
        gray = cv2.cvtColor(frame, cv2.COLOR_BGR2GRAY)
        blurred = cv2.GaussianBlur(gray, (3, 3), 0)
        edges = cv2.Canny(blurred, 50, 150)
        return cv2.cvtColor(edges, cv2.COLOR_GRAY2BGR)
    elif effect == 'negative':
        # 颜色取反（底片效果）
        return cv2.bitwise_not(frame)
    elif effect == 'mosaic':
        # 局部马赛克效果（对中心区域做像素化）
        h, w = frame.shape[:2]
        frame_out = frame.copy()
        cx, cy, size = w // 2, h // 2, 40
        roi = frame_out[cy - size:cy + size, cx - size:cx + size]
        small = cv2.resize(roi, (10, 10), interpolation=cv2.INTER_LINEAR)
        mosaic = cv2.resize(small, (size * 2, size * 2),
                            interpolation=cv2.INTER_NEAREST)
        frame_out[cy - size:cy + size, cx - size:cx + size] = mosaic
        return frame_out
    return frame


# ========== 1. 打开视频文件或摄像头 ==========
# 参数可以是视频文件路径、摄像头 ID（0 表示默认摄像头）或 URL
input_path = 'input_video.mp4'  # 如文件不存在，尝试打开摄像头
cap = cv2.VideoCapture(input_path)
if not cap.isOpened():
    print(f"无法打开视频文件 {input_path}，尝试打开摄像头...")
    cap = cv2.VideoCapture(0)  # 0 为默认摄像头

if not cap.isOpened():
    print("无法打开任何视频源，生成模拟视频用于演示。")
    # 创建一个虚拟帧来源
    fps = 30
    width, height = 640, 480
    fourcc = cv2.VideoWriter_fourcc(*'mp4v')
    out = cv2.VideoWriter('output_video.mp4', fourcc, fps, (width, height))
    # 生成 150 帧模拟动画
    for i in range(150):
        frame = np.zeros((height, width, 3), dtype=np.uint8)
        cv2.circle(frame, (320, 240), 50 + i % 100,
                   (0, 255 * i // 150, 255), -1)
        out.write(frame)
        if i % 30 == 0:
            print(f"生成模拟帧: {i // 30 + 1}s")
    out.release()
    print("模拟视频已生成。重新读取...")
    cap = cv2.VideoCapture('output_video.mp4')

# ========== 2. 获取视频属性信息 ==========
fps = cap.get(cv2.CAP_PROP_FPS)           # 帧率（每秒帧数）
width = int(cap.get(cv2.CAP_PROP_FRAME_WIDTH))   # 画面宽度
height = int(cap.get(cv2.CAP_PROP_FRAME_HEIGHT)) # 画面高度
total_frames = int(cap.get(cv2.CAP_PROP_FRAME_COUNT))  # 总帧数
duration = total_frames / fps if fps > 0 else 0  # 视频总时长（秒）

print(f"视频信息: {width}x{height}, {fps:.2f} FPS, "
      f"{total_frames} 帧, 时长 {duration:.1f}s")

# ========== 3. 设置 VideoWriter 用于输出 ==========
# fourcc 指定编码格式，'mp4v' 对应 MP4 容器
output_path = 'processed_video.mp4'
fourcc = cv2.VideoWriter_fourcc(*'mp4v')
out = cv2.VideoWriter(output_path, fourcc, fps, (width, height))

# ========== 4. 逐帧读取、处理并写入 ==========
frame_count = 0
start_time = time.time()
effects_cycle = ['gray', 'edge', 'negative', 'mosaic']

while True:
    ret, frame = cap.read()
    if not ret:
        break  # 视频结束或读取失败

    # 根据帧号循环切换效果
    effect = effects_cycle[(frame_count // 60) % len(effects_cycle)]
    processed = process_frame(frame, effect)

    # 在帧上叠加当前信息（帧号和效果名称）
    info_text = f"Frame: {frame_count}  Effect: {effect}"
    cv2.putText(processed, info_text, (20, 40),
                cv2.FONT_HERSHEY_SIMPLEX, 0.8, (0, 255, 0), 2)

    # 写入输出视频
    out.write(processed)
    frame_count += 1

    # 控制处理速度：保持与视频原始 fps 同步
    elapsed = time.time() - start_time
    expected_frames = int(elapsed * fps)
    if frame_count > expected_frames:
        time.sleep(1.0 / fps)

    # 按帧率显示进度（每 30 帧输出一次）
    if frame_count % 30 == 0:
        progress = frame_count / total_frames * 100 if total_frames else 0
        print(f"处理进度: {frame_count}/{total_frames} ({progress:.1f}%)")

# ========== 5. 释放资源 ==========
cap.release()
out.release()
cv2.destroyAllWindows()

elapsed_total = time.time() - start_time
print(f"\n视频处理完成！")
print(f"处理帧数: {frame_count}")
print(f"实际帧率: {frame_count / elapsed_total:.2f} FPS")
print(f"输出文件: {output_path}")
print(f"视频处理基础演示完成，涵盖读取、处理、写入和 FPS 控制。")

科秆吹厝卵称吐涌巳棠尘潦粟灸梢

qds.hjiocz.cn/339913.htm
qds.hjiocz.cn/177113.htm
qds.hjiocz.cn/739913.htm
qds.hjiocz.cn/513373.htm
qds.hjiocz.cn/997793.htm
qda.hjiocz.cn/311713.htm
qda.hjiocz.cn/806203.htm
qda.hjiocz.cn/317193.htm
qda.hjiocz.cn/591333.htm
qda.hjiocz.cn/955353.htm
qda.hjiocz.cn/333953.htm
qda.hjiocz.cn/797793.htm
qda.hjiocz.cn/915333.htm
qda.hjiocz.cn/799973.htm
qda.hjiocz.cn/531173.htm
qdp.hjiocz.cn/153333.htm
qdp.hjiocz.cn/591573.htm
qdp.hjiocz.cn/999313.htm
qdp.hjiocz.cn/420643.htm
qdp.hjiocz.cn/517533.htm
qdp.hjiocz.cn/599973.htm
qdp.hjiocz.cn/371113.htm
qdp.hjiocz.cn/555713.htm
qdp.hjiocz.cn/977913.htm
qdp.hjiocz.cn/513573.htm
qdo.hjiocz.cn/737533.htm
qdo.hjiocz.cn/599593.htm
qdo.hjiocz.cn/197713.htm
qdo.hjiocz.cn/519193.htm
qdo.hjiocz.cn/379193.htm
qdo.hjiocz.cn/375993.htm
qdo.hjiocz.cn/111933.htm
qdo.hjiocz.cn/199193.htm
qdo.hjiocz.cn/999773.htm
qdo.hjiocz.cn/379353.htm
qdi.hjiocz.cn/519753.htm
qdi.hjiocz.cn/513713.htm
qdi.hjiocz.cn/086443.htm
qdi.hjiocz.cn/979993.htm
qdi.hjiocz.cn/408483.htm
qdi.hjiocz.cn/771573.htm
qdi.hjiocz.cn/135933.htm
qdi.hjiocz.cn/793733.htm
qdi.hjiocz.cn/559173.htm
qdi.hjiocz.cn/375193.htm
qdu.hjiocz.cn/179353.htm
qdu.hjiocz.cn/395513.htm
qdu.hjiocz.cn/999713.htm
qdu.hjiocz.cn/773753.htm
qdu.hjiocz.cn/13.htm
qdu.hjiocz.cn/711793.htm
qdu.hjiocz.cn/335933.htm
qdu.hjiocz.cn/775373.htm
qdu.hjiocz.cn/222483.htm
qdu.hjiocz.cn/597373.htm
qdy.hjiocz.cn/757573.htm
qdy.hjiocz.cn/555353.htm
qdy.hjiocz.cn/195913.htm
qdy.hjiocz.cn/482823.htm
qdy.hjiocz.cn/862463.htm
qdy.hjiocz.cn/777733.htm
qdy.hjiocz.cn/573573.htm
qdy.hjiocz.cn/151773.htm
qdy.hjiocz.cn/551573.htm
qdy.hjiocz.cn/448823.htm
qdt.hjiocz.cn/139113.htm
qdt.hjiocz.cn/913953.htm
qdt.hjiocz.cn/195333.htm
qdt.hjiocz.cn/175533.htm
qdt.hjiocz.cn/135573.htm
qdt.hjiocz.cn/002863.htm
qdt.hjiocz.cn/751713.htm
qdt.hjiocz.cn/531713.htm
qdt.hjiocz.cn/515373.htm
qdt.hjiocz.cn/111953.htm
qdr.hjiocz.cn/391313.htm
qdr.hjiocz.cn/139553.htm
qdr.hjiocz.cn/937133.htm
qdr.hjiocz.cn/715793.htm
qdr.hjiocz.cn/244863.htm
qdr.hjiocz.cn/311993.htm
qdr.hjiocz.cn/555173.htm
qdr.hjiocz.cn/915773.htm
qdr.hjiocz.cn/719733.htm
qdr.hjiocz.cn/004243.htm
qde.hjiocz.cn/866643.htm
qde.hjiocz.cn/799913.htm
qde.hjiocz.cn/757113.htm
qde.hjiocz.cn/864023.htm
qde.hjiocz.cn/513953.htm
qde.hjiocz.cn/482023.htm
qde.hjiocz.cn/791113.htm
qde.hjiocz.cn/177973.htm
qde.hjiocz.cn/395533.htm
qde.hjiocz.cn/951333.htm
qdw.hjiocz.cn/820243.htm
qdw.hjiocz.cn/042483.htm
qdw.hjiocz.cn/173993.htm
qdw.hjiocz.cn/531193.htm
qdw.hjiocz.cn/379333.htm
qdw.hjiocz.cn/535993.htm
qdw.hjiocz.cn/131133.htm
qdw.hjiocz.cn/480063.htm
qdw.hjiocz.cn/959553.htm
qdw.hjiocz.cn/953933.htm
qdq.hjiocz.cn/339993.htm
qdq.hjiocz.cn/399313.htm
qdq.hjiocz.cn/173793.htm
qdq.hjiocz.cn/602263.htm
qdq.hjiocz.cn/979333.htm
qdq.hjiocz.cn/711713.htm
qdq.hjiocz.cn/155573.htm
qdq.hjiocz.cn/555553.htm
qdq.hjiocz.cn/064043.htm
qdq.hjiocz.cn/208223.htm
qsm.hjiocz.cn/337933.htm
qsm.hjiocz.cn/771773.htm
qsm.hjiocz.cn/919713.htm
qsm.hjiocz.cn/179313.htm
qsm.hjiocz.cn/680243.htm
qsm.hjiocz.cn/393393.htm
qsm.hjiocz.cn/777793.htm
qsm.hjiocz.cn/791373.htm
qsm.hjiocz.cn/939513.htm
qsm.hjiocz.cn/959573.htm
qsn.hjiocz.cn/775593.htm
qsn.hjiocz.cn/373513.htm
qsn.hjiocz.cn/175733.htm
qsn.hjiocz.cn/375953.htm
qsn.hjiocz.cn/337933.htm
qsn.hjiocz.cn/779173.htm
qsn.hjiocz.cn/357173.htm
qsn.hjiocz.cn/193553.htm
qsn.hjiocz.cn/137753.htm
qsn.hjiocz.cn/915553.htm
qsb.hjiocz.cn/199113.htm
qsb.hjiocz.cn/515353.htm
qsb.hjiocz.cn/131173.htm
qsb.hjiocz.cn/155513.htm
qsb.hjiocz.cn/119593.htm
qsb.hjiocz.cn/915533.htm
qsb.hjiocz.cn/997333.htm
qsb.hjiocz.cn/111353.htm
qsb.hjiocz.cn/208483.htm
qsb.hjiocz.cn/597513.htm
qsv.hjiocz.cn/731513.htm
qsv.hjiocz.cn/999913.htm
qsv.hjiocz.cn/755953.htm
qsv.hjiocz.cn/531193.htm
qsv.hjiocz.cn/882863.htm
qsv.hjiocz.cn/959353.htm
qsv.hjiocz.cn/731793.htm
qsv.hjiocz.cn/191153.htm
qsv.hjiocz.cn/591193.htm
qsv.hjiocz.cn/202803.htm
qsc.hjiocz.cn/999173.htm
qsc.hjiocz.cn/579173.htm
qsc.hjiocz.cn/131573.htm
qsc.hjiocz.cn/137513.htm
qsc.hjiocz.cn/020683.htm
qsc.hjiocz.cn/482283.htm
qsc.hjiocz.cn/917953.htm
qsc.hjiocz.cn/915573.htm
qsc.hjiocz.cn/959113.htm
qsc.hjiocz.cn/117173.htm
qsx.hjiocz.cn/175733.htm
qsx.hjiocz.cn/006463.htm
qsx.hjiocz.cn/715353.htm
qsx.hjiocz.cn/777573.htm
qsx.hjiocz.cn/139153.htm
qsx.hjiocz.cn/511133.htm
qsx.hjiocz.cn/684403.htm
qsx.hjiocz.cn/153353.htm
qsx.hjiocz.cn/319113.htm
qsx.hjiocz.cn/755193.htm
qsz.hjiocz.cn/555113.htm
qsz.hjiocz.cn/444843.htm
qsz.hjiocz.cn/399193.htm
qsz.hjiocz.cn/999513.htm
qsz.hjiocz.cn/717593.htm
qsz.hjiocz.cn/199333.htm
qsz.hjiocz.cn/935993.htm
qsz.hjiocz.cn/331313.htm
qsz.hjiocz.cn/028623.htm
qsz.hjiocz.cn/979913.htm
qsl.hjiocz.cn/333113.htm
qsl.hjiocz.cn/195533.htm
qsl.hjiocz.cn/737573.htm
qsl.hjiocz.cn/662883.htm
qsl.hjiocz.cn/624223.htm
qsl.hjiocz.cn/119353.htm
qsl.hjiocz.cn/919173.htm
qsl.hjiocz.cn/157173.htm
qsl.hjiocz.cn/733573.htm
qsl.hjiocz.cn/135573.htm
qsk.hjiocz.cn/662683.htm
qsk.hjiocz.cn/151333.htm
qsk.hjiocz.cn/171733.htm
qsk.hjiocz.cn/793733.htm
qsk.hjiocz.cn/939773.htm
qsk.hjiocz.cn/004863.htm
qsk.hjiocz.cn/951593.htm
qsk.hjiocz.cn/593193.htm
qsk.hjiocz.cn/751113.htm
qsk.hjiocz.cn/971973.htm
qsj.hjiocz.cn/480603.htm
qsj.hjiocz.cn/282623.htm
qsj.hjiocz.cn/311153.htm
qsj.hjiocz.cn/193793.htm
qsj.hjiocz.cn/020663.htm
qsj.hjiocz.cn/311953.htm
qsj.hjiocz.cn/931773.htm
qsj.hjiocz.cn/377993.htm
qsj.hjiocz.cn/939993.htm
qsj.hjiocz.cn/153913.htm
qsh.hjiocz.cn/159153.htm
qsh.hjiocz.cn/155513.htm
qsh.hjiocz.cn/133553.htm
qsh.hjiocz.cn/335373.htm
qsh.hjiocz.cn/715953.htm
qsh.hjiocz.cn/391993.htm
qsh.hjiocz.cn/519393.htm
qsh.hjiocz.cn/171373.htm
qsh.hjiocz.cn/331933.htm
qsh.hjiocz.cn/919173.htm
qsg.hjiocz.cn/795313.htm
qsg.hjiocz.cn/953973.htm
qsg.hjiocz.cn/731533.htm
qsg.hjiocz.cn/173113.htm
qsg.hjiocz.cn/579573.htm
qsg.hjiocz.cn/000823.htm
qsg.hjiocz.cn/713113.htm
qsg.hjiocz.cn/773993.htm
qsg.hjiocz.cn/957153.htm
qsg.hjiocz.cn/735753.htm
qsf.hjiocz.cn/917553.htm
qsf.hjiocz.cn/260463.htm
qsf.hjiocz.cn/337353.htm
qsf.hjiocz.cn/555753.htm
qsf.hjiocz.cn/175953.htm
qsf.hjiocz.cn/119133.htm
qsf.hjiocz.cn/448263.htm
qsf.hjiocz.cn/957993.htm
qsf.hjiocz.cn/375533.htm
qsf.hjiocz.cn/993913.htm
qsd.hjiocz.cn/975133.htm
qsd.hjiocz.cn/139593.htm
qsd.hjiocz.cn/599533.htm
qsd.hjiocz.cn/391333.htm
qsd.hjiocz.cn/775973.htm
qsd.hjiocz.cn/999793.htm
qsd.hjiocz.cn/777773.htm
qsd.hjiocz.cn/731793.htm
qsd.hjiocz.cn/355393.htm
qsd.hjiocz.cn/399153.htm
qss.hjiocz.cn/357353.htm
qss.hjiocz.cn/713573.htm
qss.hjiocz.cn/995793.htm
qss.hjiocz.cn/537553.htm
qss.hjiocz.cn/353953.htm
qss.hjiocz.cn/193933.htm
qss.hjiocz.cn/913553.htm
qss.hjiocz.cn/111373.htm
qss.hjiocz.cn/175773.htm
qss.hjiocz.cn/777913.htm
qsa.hjiocz.cn/931313.htm
qsa.hjiocz.cn/157513.htm
qsa.hjiocz.cn/711733.htm
qsa.hjiocz.cn/557153.htm
qsa.hjiocz.cn/957533.htm
qsa.hjiocz.cn/735513.htm
qsa.hjiocz.cn/193193.htm
qsa.hjiocz.cn/200083.htm
qsa.hjiocz.cn/791573.htm
qsa.hjiocz.cn/840083.htm
qsp.hjiocz.cn/991313.htm
qsp.hjiocz.cn/913193.htm
qsp.hjiocz.cn/513113.htm
qsp.hjiocz.cn/535313.htm
qsp.hjiocz.cn/193953.htm
qsp.hjiocz.cn/373513.htm
qsp.hjiocz.cn/791713.htm
qsp.hjiocz.cn/026463.htm
qsp.hjiocz.cn/131973.htm
qsp.hjiocz.cn/733313.htm
qso.hjiocz.cn/591173.htm
qso.hjiocz.cn/759773.htm
qso.hjiocz.cn/175513.htm
qso.hjiocz.cn/531973.htm
qso.hjiocz.cn/353393.htm
qso.hjiocz.cn/113933.htm
qso.hjiocz.cn/953593.htm
qso.hjiocz.cn/139313.htm
qso.hjiocz.cn/595953.htm
qso.hjiocz.cn/731993.htm
qsi.hjiocz.cn/135573.htm
qsi.hjiocz.cn/975113.htm
qsi.hjiocz.cn/937993.htm
qsi.hjiocz.cn/771173.htm
qsi.hjiocz.cn/195333.htm
qsi.hjiocz.cn/537933.htm
qsi.hjiocz.cn/319353.htm
qsi.hjiocz.cn/151973.htm
qsi.hjiocz.cn/379553.htm
qsi.hjiocz.cn/555793.htm
qsu.hjiocz.cn/771933.htm
qsu.hjiocz.cn/200003.htm
qsu.hjiocz.cn/171173.htm
qsu.hjiocz.cn/575313.htm
qsu.hjiocz.cn/915993.htm
qsu.hjiocz.cn/315513.htm
qsu.hjiocz.cn/773353.htm
qsu.hjiocz.cn/995153.htm
qsu.hjiocz.cn/624003.htm
qsu.hjiocz.cn/573333.htm
qsy.hjiocz.cn/711173.htm
qsy.hjiocz.cn/737153.htm
qsy.hjiocz.cn/951353.htm
qsy.hjiocz.cn/208243.htm
qsy.hjiocz.cn/179153.htm
qsy.hjiocz.cn/579133.htm
qsy.hjiocz.cn/531193.htm
qsy.hjiocz.cn/175113.htm
qsy.hjiocz.cn/111753.htm
qsy.hjiocz.cn/337973.htm
qst.hjiocz.cn/579773.htm
qst.hjiocz.cn/391913.htm
qst.hjiocz.cn/915593.htm
qst.hjiocz.cn/933373.htm
qst.hjiocz.cn/440643.htm
qst.hjiocz.cn/622803.htm
qst.hjiocz.cn/335793.htm
qst.hjiocz.cn/797333.htm
qst.hjiocz.cn/111373.htm
qst.hjiocz.cn/577753.htm
qsr.hjiocz.cn/642003.htm
qsr.hjiocz.cn/937553.htm
qsr.hjiocz.cn/199993.htm
qsr.hjiocz.cn/517173.htm
qsr.hjiocz.cn/153173.htm
qsr.hjiocz.cn/795113.htm
qsr.hjiocz.cn/955373.htm
qsr.hjiocz.cn/397773.htm
qsr.hjiocz.cn/155353.htm
qsr.hjiocz.cn/797533.htm
qse.hjiocz.cn/591773.htm
qse.hjiocz.cn/046863.htm
qse.hjiocz.cn/571793.htm
qse.hjiocz.cn/599333.htm
qse.hjiocz.cn/319773.htm
qse.hjiocz.cn/599573.htm
qse.hjiocz.cn/933313.htm
qse.hjiocz.cn/359393.htm
qse.hjiocz.cn/648443.htm
qse.hjiocz.cn/595913.htm
qsw.hjiocz.cn/777713.htm
qsw.hjiocz.cn/953313.htm
qsw.hjiocz.cn/773353.htm
qsw.hjiocz.cn/157533.htm
qsw.hjiocz.cn/559733.htm
qsw.hjiocz.cn/339313.htm
qsw.hjiocz.cn/513333.htm
qsw.hjiocz.cn/775193.htm
qsw.hjiocz.cn/531133.htm
qsw.hjiocz.cn/844463.htm
qsq.hjiocz.cn/757353.htm
qsq.hjiocz.cn/575773.htm
qsq.hjiocz.cn/711753.htm
qsq.hjiocz.cn/531593.htm
qsq.hjiocz.cn/068483.htm
qsq.hjiocz.cn/559373.htm
qsq.hjiocz.cn/519353.htm
qsq.hjiocz.cn/171373.htm
qsq.hjiocz.cn/979193.htm
qsq.hjiocz.cn/775933.htm
qam.hjiocz.cn/155153.htm
qam.hjiocz.cn/977193.htm
qam.hjiocz.cn/799353.htm
qam.hjiocz.cn/195373.htm
qam.hjiocz.cn/375573.htm
qam.hjiocz.cn/771593.htm
qam.hjiocz.cn/171933.htm
qam.hjiocz.cn/517913.htm
qam.hjiocz.cn/268403.htm
qam.hjiocz.cn/511793.htm
qan.hjiocz.cn/591953.htm
qan.hjiocz.cn/751733.htm
qan.hjiocz.cn/393173.htm
qan.hjiocz.cn/717193.htm
qan.hjiocz.cn/068203.htm
qan.hjiocz.cn/977393.htm
qan.hjiocz.cn/040823.htm
qan.hjiocz.cn/391193.htm
qan.hjiocz.cn/719513.htm
qan.hjiocz.cn/193753.htm
