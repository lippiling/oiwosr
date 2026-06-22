坝什掠饰捍


"""
Python 条形码与二维码详解
包含：qrcode 生成与自定义、python-barcode 生成 Code128/EAN13、pyzbar 解码扫描
"""
import qrcode
from qrcode.image.styledpil import StyledPilImage
from qrcode.image.styles.moduledrawers import (
    RoundedModuleDrawer, SquareModuleDrawer, GappedSquareModuleDrawer
)
from qrcode.image.styles.colormasks import RadialGradiantColorMask
import barcode
from barcode.writer import ImageWriter
import pyzbar.pyzbar as pyzbar
from PIL import Image, ImageDraw, ImageFont
import numpy as np
import os


def create_test_barcode_image():
    """生成一张包含条形码和二维码的测试图片用于演示解码"""
    img = Image.new('RGB', (400, 300), 'white')
    draw = ImageDraw.Draw(img)
    # 用简单的黑白条纹模拟条形码
    for i in range(50):
        x = 30 + i * 7
        if i % 3 != 0:  # 部分条纹为黑色
            draw.rectangle([x, 30, x + 4, 150], fill='black')
    draw.text((30, 170), "123456789012", fill='black')
    return img


# ========== 1. 使用 qrcode 库生成二维码 ==========
print("===== 1. qrcode 二维码生成 =====")

# 1a. 最简单的二维码：一行代码生成并保存
img_simple = qrcode.make("https://www.python.org")
img_simple.save("qr_simple.png")
print("基础二维码已生成: qr_simple.png")

# 1b. 自定义参数的二维码
qr = qrcode.QRCode(
    version=5,           # 版本 1~40，控制二维码大小（越高越密集）
    error_correction=qrcode.constants.ERROR_CORRECT_H,  # 最高纠错级别（约 30%）
    box_size=10,         # 每个小格子的像素大小
    border=4,            # 二维码边框宽度（格子数）
)
# 添加数据内容（可以是 URL、文本、电话号码等）
qr.add_data("https://github.com/python/cpython")
qr.make(fit=True)  # 自动调整版本大小以适应数据

# 生成图像（默认是 PIL Image 对象）
qr_img = qr.make_image(fill_color="darkblue", back_color="white")
qr_img.save("qr_custom.png")
print("自定义二维码已生成: qr_custom.png")

# 1c. 二维码中添加 Logo
qr_logo = qrcode.QRCode(
    version=5,
    error_correction=qrcode.constants.ERROR_CORRECT_H,  # 高纠错以容纳 Logo
    box_size=10,
    border=4,
)
qr_logo.add_data("https://www.python.org")
qr_logo.make(fit=True)
qr_logo_img = qr_logo.make_image(fill_color="black", back_color="white")
# 将 PIL Image 转为可编辑模式
qr_logo_img = qr_logo_img.convert("RGB")
# 在二维码中心叠加一个小图标
logo_size = 60
try:
    logo = Image.new('RGBA', (logo_size, logo_size), (0, 0, 0, 0))
    draw = ImageDraw.Draw(logo)
    # 绘制一个 Python 风格图标
    draw.ellipse([2, 2, logo_size - 2, logo_size - 2],
                 fill='#306998', outline='#FFD43B', width=3)
    draw.text((12, 15), "Py", fill='white')
    # 计算 Logo 位置（居中）
    pos = ((qr_logo_img.size[0] - logo_size) // 2,
           (qr_logo_img.size[1] - logo_size) // 2)
    qr_logo_img.paste(logo, pos, logo)
    qr_logo_img.save("qr_with_logo.png")
    print("带 Logo 二维码已生成: qr_with_logo.png")
except Exception as e:
    print(f"Logo 添加失败: {e}")

# 1d. 使用样式化二维码（美化的模块形状）
qr_styled = qrcode.QRCode(
    version=4,
    error_correction=qrcode.constants.ERROR_CORRECT_M,
    box_size=10,
    border=4,
)
qr_styled.add_data("Styled QR Code Demo")
qr_styled.make(fit=True)

# 使用圆角模块绘制器 + 径向渐变色
qr_styled_img = qr_styled.make_image(
    image_factory=StyledPilImage,
    module_drawer=RoundedModuleDrawer(),
    color_mask=RadialGradiantColorMask(
        center_color=(0, 100, 200),
        edge_color=(200, 50, 0),
        center_radius=0.5
    )
)
qr_styled_img.save("qr_styled.png")
print("样式化二维码已生成: qr_styled.png")

# 使用间隙方形模块
qr_gapped = qrcode.QRCode(version=3, box_size=10, border=4)
qr_gapped.add_data("Gapped Style QR")
qr_gapped.make(fit=True)
qr_gapped_img = qr_gapped.make_image(
    image_factory=StyledPilImage,
    module_drawer=GappedSquareModuleDrawer()
)
qr_gapped_img.save("qr_gapped.png")
print("间隙方形二维码已生成: qr_gapped.png")

# ========== 2. 使用 python-barcode 生成条形码 ==========
print("\n===== 2. python-barcode 条形码生成 =====")

# 2a. Code128 条形码（支持字母数字）
code128 = barcode.get('code128', 'Python-Barcode-123', writer=ImageWriter())
code128.save("barcode_code128")
print("Code128 条形码已生成: barcode_code128.png")

# 2b. EAN-13 条形码（13 位数字，商品通用标准）
ean = barcode.get('ean13', '5901234123457', writer=ImageWriter())
ean.save("barcode_ean13")
print("EAN-13 条形码已生成: barcode_ean13.png")

# 2c. Code39 条形码
code39 = barcode.get('code39', 'PYTHON39', writer=ImageWriter())
code39.save("barcode_code39")
print("Code39 条形码已生成: barcode_code39.png")

# 2d. 自定义条形码样式
custom_writer = ImageWriter()
custom_writer.set_options({
    'module_width': 0.3,      # 条码模块宽度
    'module_height': 15.0,    # 条码高度
    'font_size': 12,          # 文字大小
    'text_distance': 5.0,     # 文字与条码距离
    'background': 'white',    # 背景色
    'foreground': 'black',    # 前景色
    'center_text': True,      # 文字居中
})
custom_barcode = barcode.get('code128', 'Custom-Style-999',
                              writer=custom_writer)
custom_barcode.save("barcode_custom")
print("自定义样式条形码已生成: barcode_custom.png")

# ========== 3. 使用 pyzbar 解码条形码和二维码 ==========
print("\n===== 3. pyzbar 解码 =====")

# 3a. 解码我们刚刚生成的二维码
def decode_image(image_path):
    """解码图像中的条形码/二维码，返回解码结果列表"""
    img = Image.open(image_path)
    # pyzbar 需要灰度图或 RGB 图
    results = pyzbar.decode(img)
    decoded_info = []
    for result in results:
        data = result.data.decode('utf-8')
        barcode_type = result.type
        rect = result.rect
        decoded_info.append({
            'data': data,
            'type': barcode_type,
            'rect': rect
        })
        print(f"  类型: {barcode_type:8s} | 数据: {data}")
    return decoded_info

# 解码生成的二维码文件
qr_files = ['qr_simple.png', 'qr_custom.png', 'qr_with_logo.png',
            'qr_styled.png']
for f in qr_files:
    if os.path.exists(f):
        print(f"\n解码 {f}:")
        decode_image(f)

# 解码生成的条形码文件
barcode_files = ['barcode_code128.png', 'barcode_ean13.png',
                  'barcode_code39.png', 'barcode_custom.png']
for f in barcode_files:
    if os.path.exists(f):
        print(f"\n解码 {f}:")
        decode_image(f)

# ========== 4. 实时扫描：从摄像头捕获解码 ==========
print("\n===== 4. 实时扫描演示 =====")
# 模拟解码：直接从图像数组中解码
test_img = create_test_barcode_image()
test_img.save("test_barcode_scene.png")
print("测试条码场景已生成: test_barcode_scene.png")
results = pyzbar.decode(test_img)
if results:
    for r in results:
        print(f"模拟扫描到: {r.data.decode('utf-8')} (类型: {r.type})")
else:
    print("模拟扫描未识别到条码（预期行为，测试图仅视觉模拟）。")

# ========== 5. 批量解码演示 ==========
print("\n===== 5. 批量解码 =====")
print("扫描目录中的条码/二维码文件:")

def batch_decode(directory='.', extensions=('.png', '.jpg', '.jpeg')):
    """扫描目录中所有图像文件并尝试解码"""
    found_any = False
    for fname in os.listdir(directory):
        if fname.lower().endswith(extensions):
            fpath = os.path.join(directory, fname)
            try:
                results = pyzbar.decode(Image.open(fpath))
                if results:
                    found_any = True
                    for r in results:
                        print(f"  [{fname}]: {r.type} -> {r.data.decode('utf-8')}")
            except Exception:
                pass
    if not found_any:
        print("  未在图像文件中发现可解码的条码。")
    return found_any

batch_decode()

print("\n条形码与二维码演示完成！")
print("涵盖: qrcode 生成(4种样式)、barcode 生成(3种格式)、pyzbar 解码")

排夹巡坝牧嵌访恼恼事仲妹覆熬蔷

pzr.nfsid.cn/244842.Doc
pzr.nfsid.cn/266688.Doc
pzr.nfsid.cn/224802.Doc
pzr.nfsid.cn/468442.Doc
pzr.nfsid.cn/866044.Doc
pzr.nfsid.cn/353939.Doc
pzr.nfsid.cn/204048.Doc
pzr.nfsid.cn/402480.Doc
pzr.nfsid.cn/024606.Doc
pzr.nfsid.cn/628004.Doc
pze.irvnp.cn/682806.Doc
pze.irvnp.cn/202668.Doc
pze.irvnp.cn/804244.Doc
pze.irvnp.cn/640084.Doc
pze.irvnp.cn/808420.Doc
pze.irvnp.cn/826206.Doc
pze.irvnp.cn/446204.Doc
pze.irvnp.cn/260460.Doc
pze.irvnp.cn/668446.Doc
pze.irvnp.cn/464004.Doc
pzw.irvnp.cn/044024.Doc
pzw.irvnp.cn/044688.Doc
pzw.irvnp.cn/688824.Doc
pzw.irvnp.cn/448208.Doc
pzw.irvnp.cn/624040.Doc
pzw.irvnp.cn/042084.Doc
pzw.irvnp.cn/608620.Doc
pzw.irvnp.cn/866486.Doc
pzw.irvnp.cn/176946.Doc
pzw.irvnp.cn/336587.Doc
pzq.irvnp.cn/733739.Doc
pzq.irvnp.cn/802608.Doc
pzq.irvnp.cn/796740.Doc
pzq.irvnp.cn/157377.Doc
pzq.irvnp.cn/042688.Doc
pzq.irvnp.cn/220022.Doc
pzq.irvnp.cn/724626.Doc
pzq.irvnp.cn/466644.Doc
pzq.irvnp.cn/006088.Doc
pzq.irvnp.cn/279433.Doc
plm.irvnp.cn/440602.Doc
plm.irvnp.cn/844206.Doc
plm.irvnp.cn/028046.Doc
plm.irvnp.cn/046468.Doc
plm.irvnp.cn/288620.Doc
plm.irvnp.cn/174356.Doc
plm.irvnp.cn/442888.Doc
plm.irvnp.cn/826486.Doc
plm.irvnp.cn/339258.Doc
plm.irvnp.cn/228048.Doc
pln.irvnp.cn/660866.Doc
pln.irvnp.cn/820022.Doc
pln.irvnp.cn/066602.Doc
pln.irvnp.cn/040088.Doc
pln.irvnp.cn/693300.Doc
pln.irvnp.cn/260206.Doc
pln.irvnp.cn/868110.Doc
pln.irvnp.cn/402448.Doc
pln.irvnp.cn/355496.Doc
pln.irvnp.cn/008084.Doc
plb.irvnp.cn/444406.Doc
plb.irvnp.cn/868668.Doc
plb.irvnp.cn/042600.Doc
plb.irvnp.cn/400240.Doc
plb.irvnp.cn/680000.Doc
plb.irvnp.cn/828024.Doc
plb.irvnp.cn/657433.Doc
plb.irvnp.cn/646931.Doc
plb.irvnp.cn/046701.Doc
plb.irvnp.cn/464086.Doc
plv.irvnp.cn/280662.Doc
plv.irvnp.cn/044660.Doc
plv.irvnp.cn/220682.Doc
plv.irvnp.cn/488202.Doc
plv.irvnp.cn/688602.Doc
plv.irvnp.cn/080624.Doc
plv.irvnp.cn/068442.Doc
plv.irvnp.cn/248604.Doc
plv.irvnp.cn/008620.Doc
plv.irvnp.cn/220022.Doc
plc.irvnp.cn/840646.Doc
plc.irvnp.cn/620866.Doc
plc.irvnp.cn/868446.Doc
plc.irvnp.cn/329774.Doc
plc.irvnp.cn/822624.Doc
plc.irvnp.cn/600666.Doc
plc.irvnp.cn/666868.Doc
plc.irvnp.cn/082828.Doc
plc.irvnp.cn/204606.Doc
plc.irvnp.cn/808644.Doc
plx.irvnp.cn/080488.Doc
plx.irvnp.cn/351515.Doc
plx.irvnp.cn/941919.Doc
plx.irvnp.cn/242006.Doc
plx.irvnp.cn/826404.Doc
plx.irvnp.cn/428460.Doc
plx.irvnp.cn/688800.Doc
plx.irvnp.cn/506649.Doc
plx.irvnp.cn/802886.Doc
plx.irvnp.cn/268828.Doc
plz.irvnp.cn/884240.Doc
plz.irvnp.cn/284688.Doc
plz.irvnp.cn/404020.Doc
plz.irvnp.cn/806840.Doc
plz.irvnp.cn/593791.Doc
plz.irvnp.cn/844666.Doc
plz.irvnp.cn/222424.Doc
plz.irvnp.cn/254860.Doc
plz.irvnp.cn/806604.Doc
plz.irvnp.cn/407191.Doc
pll.irvnp.cn/686868.Doc
pll.irvnp.cn/028240.Doc
pll.irvnp.cn/466682.Doc
pll.irvnp.cn/282000.Doc
pll.irvnp.cn/228806.Doc
pll.irvnp.cn/440648.Doc
pll.irvnp.cn/622222.Doc
pll.irvnp.cn/806668.Doc
pll.irvnp.cn/763334.Doc
pll.irvnp.cn/468266.Doc
plk.irvnp.cn/062484.Doc
plk.irvnp.cn/008226.Doc
plk.irvnp.cn/062422.Doc
plk.irvnp.cn/828024.Doc
plk.irvnp.cn/104845.Doc
plk.irvnp.cn/264848.Doc
plk.irvnp.cn/606002.Doc
plk.irvnp.cn/206448.Doc
plk.irvnp.cn/860824.Doc
plk.irvnp.cn/272902.Doc
plj.irvnp.cn/544862.Doc
plj.irvnp.cn/444442.Doc
plj.irvnp.cn/639695.Doc
plj.irvnp.cn/078839.Doc
plj.irvnp.cn/658982.Doc
plj.irvnp.cn/626626.Doc
plj.irvnp.cn/979391.Doc
plj.irvnp.cn/288428.Doc
plj.irvnp.cn/228608.Doc
plj.irvnp.cn/840002.Doc
plh.irvnp.cn/424048.Doc
plh.irvnp.cn/088088.Doc
plh.irvnp.cn/666660.Doc
plh.irvnp.cn/021001.Doc
plh.irvnp.cn/426682.Doc
plh.irvnp.cn/864228.Doc
plh.irvnp.cn/628064.Doc
plh.irvnp.cn/006848.Doc
plh.irvnp.cn/224022.Doc
plh.irvnp.cn/688808.Doc
plg.irvnp.cn/484642.Doc
plg.irvnp.cn/622604.Doc
plg.irvnp.cn/772932.Doc
plg.irvnp.cn/248262.Doc
plg.irvnp.cn/042468.Doc
plg.irvnp.cn/995559.Doc
plg.irvnp.cn/464006.Doc
plg.irvnp.cn/420282.Doc
plg.irvnp.cn/082066.Doc
plg.irvnp.cn/884800.Doc
plf.irvnp.cn/462664.Doc
plf.irvnp.cn/884406.Doc
plf.irvnp.cn/666448.Doc
plf.irvnp.cn/468082.Doc
plf.irvnp.cn/177193.Doc
plf.irvnp.cn/224088.Doc
plf.irvnp.cn/068000.Doc
plf.irvnp.cn/280082.Doc
plf.irvnp.cn/046028.Doc
plf.irvnp.cn/095626.Doc
pld.irvnp.cn/886884.Doc
pld.irvnp.cn/240880.Doc
pld.irvnp.cn/480848.Doc
pld.irvnp.cn/608660.Doc
pld.irvnp.cn/028624.Doc
pld.irvnp.cn/224408.Doc
pld.irvnp.cn/008424.Doc
pld.irvnp.cn/842444.Doc
pld.irvnp.cn/042820.Doc
pld.irvnp.cn/004400.Doc
pls.irvnp.cn/444864.Doc
pls.irvnp.cn/597315.Doc
pls.irvnp.cn/220482.Doc
pls.irvnp.cn/126871.Doc
pls.irvnp.cn/686048.Doc
pls.irvnp.cn/646662.Doc
pls.irvnp.cn/024466.Doc
pls.irvnp.cn/866460.Doc
pls.irvnp.cn/481222.Doc
pls.irvnp.cn/822280.Doc
pla.irvnp.cn/236412.Doc
pla.irvnp.cn/822866.Doc
pla.irvnp.cn/820840.Doc
pla.irvnp.cn/228608.Doc
pla.irvnp.cn/424820.Doc
pla.irvnp.cn/686048.Doc
pla.irvnp.cn/226486.Doc
pla.irvnp.cn/680420.Doc
pla.irvnp.cn/062226.Doc
pla.irvnp.cn/840600.Doc
plp.irvnp.cn/886800.Doc
plp.irvnp.cn/820802.Doc
plp.irvnp.cn/952445.Doc
plp.irvnp.cn/662624.Doc
plp.irvnp.cn/645620.Doc
plp.irvnp.cn/062446.Doc
plp.irvnp.cn/644280.Doc
plp.irvnp.cn/808288.Doc
plp.irvnp.cn/407106.Doc
plp.irvnp.cn/822406.Doc
plo.irvnp.cn/199351.Doc
plo.irvnp.cn/848737.Doc
plo.irvnp.cn/086402.Doc
plo.irvnp.cn/886246.Doc
plo.irvnp.cn/115955.Doc
plo.irvnp.cn/480686.Doc
plo.irvnp.cn/466206.Doc
plo.irvnp.cn/802620.Doc
plo.irvnp.cn/880460.Doc
plo.irvnp.cn/694840.Doc
pli.irvnp.cn/666226.Doc
pli.irvnp.cn/064220.Doc
pli.irvnp.cn/880886.Doc
pli.irvnp.cn/640080.Doc
pli.irvnp.cn/008640.Doc
pli.irvnp.cn/420880.Doc
pli.irvnp.cn/080844.Doc
pli.irvnp.cn/868002.Doc
pli.irvnp.cn/044622.Doc
pli.irvnp.cn/139535.Doc
plu.irvnp.cn/460802.Doc
plu.irvnp.cn/880424.Doc
plu.irvnp.cn/824626.Doc
plu.irvnp.cn/066880.Doc
plu.irvnp.cn/296186.Doc
plu.irvnp.cn/222828.Doc
plu.irvnp.cn/886460.Doc
plu.irvnp.cn/258252.Doc
plu.irvnp.cn/402286.Doc
plu.irvnp.cn/288444.Doc
ply.irvnp.cn/153502.Doc
ply.irvnp.cn/608600.Doc
ply.irvnp.cn/420886.Doc
ply.irvnp.cn/462406.Doc
ply.irvnp.cn/608786.Doc
ply.irvnp.cn/266446.Doc
ply.irvnp.cn/684028.Doc
ply.irvnp.cn/828862.Doc
ply.irvnp.cn/208022.Doc
ply.irvnp.cn/800242.Doc
plt.irvnp.cn/220820.Doc
plt.irvnp.cn/486021.Doc
plt.irvnp.cn/626602.Doc
plt.irvnp.cn/660806.Doc
plt.irvnp.cn/448204.Doc
plt.irvnp.cn/664602.Doc
plt.irvnp.cn/204202.Doc
plt.irvnp.cn/822422.Doc
plt.irvnp.cn/884024.Doc
plt.irvnp.cn/288240.Doc
plr.irvnp.cn/890451.Doc
plr.irvnp.cn/484280.Doc
plr.irvnp.cn/406268.Doc
plr.irvnp.cn/464064.Doc
plr.irvnp.cn/191559.Doc
plr.irvnp.cn/824404.Doc
plr.irvnp.cn/646426.Doc
plr.irvnp.cn/264280.Doc
plr.irvnp.cn/468848.Doc
plr.irvnp.cn/206046.Doc
ple.irvnp.cn/460264.Doc
ple.irvnp.cn/880541.Doc
ple.irvnp.cn/442682.Doc
ple.irvnp.cn/848884.Doc
ple.irvnp.cn/965606.Doc
ple.irvnp.cn/202682.Doc
ple.irvnp.cn/848376.Doc
ple.irvnp.cn/591939.Doc
ple.irvnp.cn/208286.Doc
ple.irvnp.cn/824846.Doc
plw.irvnp.cn/880806.Doc
plw.irvnp.cn/248024.Doc
plw.irvnp.cn/222626.Doc
plw.irvnp.cn/622888.Doc
plw.irvnp.cn/640468.Doc
plw.irvnp.cn/268608.Doc
plw.irvnp.cn/888262.Doc
plw.irvnp.cn/604268.Doc
plw.irvnp.cn/006688.Doc
plw.irvnp.cn/080882.Doc
plq.irvnp.cn/624880.Doc
plq.irvnp.cn/442842.Doc
plq.irvnp.cn/022466.Doc
plq.irvnp.cn/466466.Doc
plq.irvnp.cn/242042.Doc
plq.irvnp.cn/177171.Doc
plq.irvnp.cn/880864.Doc
plq.irvnp.cn/404662.Doc
plq.irvnp.cn/888064.Doc
plq.irvnp.cn/280042.Doc
pkm.irvnp.cn/040848.Doc
pkm.irvnp.cn/413762.Doc
pkm.irvnp.cn/066424.Doc
pkm.irvnp.cn/888806.Doc
pkm.irvnp.cn/282224.Doc
pkm.irvnp.cn/402282.Doc
pkm.irvnp.cn/022400.Doc
pkm.irvnp.cn/080640.Doc
pkm.irvnp.cn/688868.Doc
pkm.irvnp.cn/686262.Doc
pkn.irvnp.cn/739119.Doc
pkn.irvnp.cn/763782.Doc
pkn.irvnp.cn/848424.Doc
pkn.irvnp.cn/404260.Doc
pkn.irvnp.cn/680806.Doc
pkn.irvnp.cn/604804.Doc
pkn.irvnp.cn/000040.Doc
pkn.irvnp.cn/808606.Doc
pkn.irvnp.cn/286882.Doc
pkn.irvnp.cn/828208.Doc
pkb.irvnp.cn/506040.Doc
pkb.irvnp.cn/286066.Doc
pkb.irvnp.cn/351999.Doc
pkb.irvnp.cn/880822.Doc
pkb.irvnp.cn/848048.Doc
pkb.irvnp.cn/317793.Doc
pkb.irvnp.cn/148484.Doc
pkb.irvnp.cn/084886.Doc
pkb.irvnp.cn/202484.Doc
pkb.irvnp.cn/204864.Doc
pkv.irvnp.cn/644442.Doc
pkv.irvnp.cn/288008.Doc
pkv.irvnp.cn/680662.Doc
pkv.irvnp.cn/937711.Doc
pkv.irvnp.cn/808448.Doc
pkv.irvnp.cn/508055.Doc
pkv.irvnp.cn/426086.Doc
pkv.irvnp.cn/282620.Doc
pkv.irvnp.cn/686282.Doc
pkv.irvnp.cn/084860.Doc
pkc.irvnp.cn/206806.Doc
pkc.irvnp.cn/937791.Doc
pkc.irvnp.cn/088804.Doc
pkc.irvnp.cn/800746.Doc
pkc.irvnp.cn/264222.Doc
pkc.irvnp.cn/997311.Doc
pkc.irvnp.cn/084242.Doc
pkc.irvnp.cn/626028.Doc
pkc.irvnp.cn/626628.Doc
pkc.irvnp.cn/680408.Doc
pkx.irvnp.cn/884488.Doc
pkx.irvnp.cn/002628.Doc
pkx.irvnp.cn/424262.Doc
pkx.irvnp.cn/068286.Doc
pkx.irvnp.cn/844844.Doc
pkx.irvnp.cn/644666.Doc
pkx.irvnp.cn/281657.Doc
pkx.irvnp.cn/468824.Doc
pkx.irvnp.cn/884482.Doc
pkx.irvnp.cn/820486.Doc
pkz.irvnp.cn/466202.Doc
pkz.irvnp.cn/822648.Doc
pkz.irvnp.cn/763024.Doc
pkz.irvnp.cn/800886.Doc
pkz.irvnp.cn/040446.Doc
pkz.irvnp.cn/262044.Doc
pkz.irvnp.cn/400062.Doc
pkz.irvnp.cn/286226.Doc
pkz.irvnp.cn/056067.Doc
pkz.irvnp.cn/620628.Doc
pkl.irvnp.cn/224385.Doc
pkl.irvnp.cn/024402.Doc
pkl.irvnp.cn/222242.Doc
pkl.irvnp.cn/866428.Doc
pkl.irvnp.cn/187656.Doc
pkl.irvnp.cn/664026.Doc
pkl.irvnp.cn/824424.Doc
pkl.irvnp.cn/284848.Doc
pkl.irvnp.cn/280688.Doc
pkl.irvnp.cn/404602.Doc
pkk.irvnp.cn/824448.Doc
pkk.irvnp.cn/428840.Doc
pkk.irvnp.cn/141093.Doc
pkk.irvnp.cn/240608.Doc
pkk.irvnp.cn/042028.Doc
pkk.irvnp.cn/533719.Doc
pkk.irvnp.cn/264626.Doc
pkk.irvnp.cn/240646.Doc
pkk.irvnp.cn/404200.Doc
pkk.irvnp.cn/228242.Doc
pkj.irvnp.cn/122463.Doc
pkj.irvnp.cn/446680.Doc
pkj.irvnp.cn/468440.Doc
pkj.irvnp.cn/422088.Doc
pkj.irvnp.cn/024466.Doc
