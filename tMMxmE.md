徊谥稳捍谢


"""
Python 安全文件上传 —— 全方位的文件上传安全防护
涵盖扩展名校验、MIME 检测、内容扫描、大小控制和文件名消毒
"""

# 安装依赖：pip install python-magic-bin （Windows）或 python-magic （Linux/Mac）
# 文件上传是 Web 应用最常见的安全风险点之一

import os
import re
import imghdr
import magic                    # 用于检测文件真实 MIME 类型
from pathlib import Path
from typing import Tuple, Optional, Set
from werkzeug.utils import secure_filename
import hashlib
import time

# ========== 第一部分：配置 ==========

class UploadConfig:
    """
    文件上传的安全配置。
    所有限制参数集中管理，便于审计和修改。
    """
    # 允许的文件扩展名白名单
    ALLOWED_EXTENSIONS: Set[str] = {
        ".txt", ".pdf", ".doc", ".docx",
        ".xls", ".xlsx", ".csv",
        ".jpg", ".jpeg", ".png", ".gif", ".bmp", ".webp",
        ".mp3", ".mp4", ".zip"
    }
    # 允许的 MIME 类型白名单
    ALLOWED_MIME_TYPES: Set[str] = {
        "text/plain",
        "application/pdf",
        "application/msword",
        "application/vnd.openxmlformats-officedocument.wordprocessingml.document",
        "image/jpeg", "image/png", "image/gif", "image/bmp", "image/webp",
        "audio/mpeg", "video/mp4",
        "application/zip",
        "text/csv"
    }
    # 文件大小限制
    MAX_FILE_SIZE_MB: int = 10
    MAX_FILE_SIZE_BYTES: int = MAX_FILE_SIZE_MB * 1024 * 1024
    # 上传目录
    UPLOAD_DIR: str = "./secure_uploads"
    # 危险文件签名（魔数）黑名单
    DANGEROUS_SIGNATURES: Set[bytes] = {
        b"MZ",                  # Windows 可执行文件
        b"\x7fELF",             # Linux 可执行文件
        b"%PDF",                # 某些情况下 PDF 可能带恶意宏（但白名单允许则放行）
    }


# ========== 第二部分：核心安全校验器 ==========

class SecureFileUploader:
    """
    安全文件上传器 —— 实现多层安全校验。
    """

    def __init__(self, config: UploadConfig = None):
        self.config = config or UploadConfig()
        # 确保上传目录存在
        os.makedirs(self.config.UPLOAD_DIR, exist_ok=True)

    def validate_extension(self, filename: str) -> bool:
        """
        第一层校验：文件扩展名白名单检查。
        攻击者可能将可执行文件命名为 image.jpg.exe 绕过。
        """
        # 获取文件扩展名（转小写）
        ext = Path(filename).suffix.lower()
        if ext not in self.config.ALLOWED_EXTENSIONS:
            print(f"拒绝：不允许的扩展名 '{ext}'")
            return False
        print(f"扩展名校验通过：{ext}")
        return True

    def validate_mime_type(self, filepath: str) -> bool:
        """
        第二层校验：MIME 类型检测。
        使用 python-magic 读取文件头（魔数）判断真实类型，
        防止攻击者将恶意文件伪装成合法后缀。
        """
        try:
            # 检测文件的真实 MIME 类型（基于文件内容，而非扩展名）
            mime_type = magic.from_file(filepath, mime=True)
            print(f"检测到 MIME 类型：{mime_type}")

            if mime_type not in self.config.ALLOWED_MIME_TYPES:
                print(f"拒绝：不允许的 MIME 类型 '{mime_type}'")
                return False
            # 二次验证：确保 MIME 类型与扩展名匹配
            ext = Path(filepath).suffix.lower()
            if not self._mime_matches_extension(mime_type, ext):
                print(f"拒绝：MIME 类型 '{mime_type}' 与扩展名 '{ext}' 不匹配")
                return False
            print("MIME 类型校验通过")
            return True
        except Exception as e:
            print(f"MIME 检测失败：{e}")
            return False

    def _mime_matches_extension(self, mime: str, ext: str) -> bool:
        """
        辅助方法：检查 MIME 类型与文件扩展名是否一致。
        """
        mime_to_ext = {
            "text/plain": ".txt",
            "application/pdf": ".pdf",
            "image/jpeg": ".jpg",
            "image/png": ".png",
            "image/gif": ".gif",
        }
        expected_ext = mime_to_ext.get(mime)
        if expected_ext and expected_ext != ext:
            return False
        return True

    def scan_content(self, filepath: str) -> bool:
        """
        第三层校验：内容安全检查。
        检查文件是否包含危险签名或可执行代码。
        """
        with open(filepath, "rb") as f:
            header = f.read(20)  # 读取文件头

        # 检查危险文件签名
        for sig in self.config.DANGEROUS_SIGNATURES:
            if header.startswith(sig):
                print(f"拒绝：检测到危险文件签名 '{sig}'")
                return False

        # 检查图片文件是否包含隐藏的脚本（二次渲染攻击）
        ext = Path(filepath).suffix.lower()
        if ext in {".jpg", ".jpeg", ".png", ".gif"}:
            if not self._validate_image_safety(filepath):
                return False

        print("内容安全检查通过")
        return True

    def _validate_image_safety(self, filepath: str) -> bool:
        """
        检查图片文件的安全性：验证图片完整性，检测嵌入脚本。
        """
        try:
            # 使用 imghdr 验证图片文件头是否有效
            image_type = imghdr.what(filepath)
            if not image_type:
                print("拒绝：无效的图片文件")
                return False
            # 检查文件是否过大（防止 decompression bomb）
            file_size = os.path.getsize(filepath)
            if file_size > 50 * 1024 * 1024:  # 50MB 图片限制
                print("拒绝：图片文件过大，可能为解压炸弹")
                return False
            return True
        except Exception as e:
            print(f"图片安全检查失败：{e}")
            return False

    def validate_file_size(self, filepath: str) -> bool:
        """
        第四层校验：文件大小检查。
        防止大文件上传导致磁盘空间耗尽（DoS 攻击）。
        """
        size = os.path.getsize(filepath)
        if size > self.config.MAX_FILE_SIZE_BYTES:
            print(f"拒绝：文件大小 {size / 1024 / 1024:.1f}MB 超过限制 "
                  f"{self.config.MAX_FILE_SIZE_MB}MB")
            return False
        print(f"文件大小校验通过：{size / 1024:.1f}KB")
        return True

    def sanitize_filename(self, filename: str) -> str:
        """
        第五层校验：文件名消毒。
        防止路径遍历攻击（如 ../../../etc/passwd）和特殊字符注入。
        """
        # 使用 Werkzeug 的 secure_filename 进行基础消毒
        safe_name = secure_filename(filename)
        if not safe_name:
            # 如果文件名完全不可用，生成随机名称
            safe_name = self._generate_safe_filename(filename)
        print(f"文件名已消毒：{filename} -> {safe_name}")
        return safe_name

    def _generate_safe_filename(self, original: str) -> str:
        """
        生成安全的随机文件名，保留原始扩展名。
        """
        ext = Path(original).suffix.lower()
        timestamp = int(time.time())
        random_hash = hashlib.md5(
            f"{original}{timestamp}".encode()
        ).hexdigest()[:8]
        return f"{timestamp}_{random_hash}{ext}"


# ========== 第三部分：完整上传流程 ==========

def secure_upload(uploader: SecureFileUploader, file_data: bytes,
                  original_filename: str) -> Optional[str]:
    """
    安全上传的完整流程：执行所有五层校验。
    """
    print(f"\n开始安全检查：{original_filename}")
    print("=" * 40)

    # 第一步：扩展名校验
    if not uploader.validate_extension(original_filename):
        return None

    # 第二步：文件名消毒
    safe_filename = uploader.sanitize_filename(original_filename)
    temp_path = os.path.join(uploader.config.UPLOAD_DIR, f"temp_{safe_filename}")

    try:
        # 写入临时文件供后续检测
        with open(temp_path, "wb") as f:
            f.write(file_data)

        # 第三步：文件大小校验
        if not uploader.validate_file_size(temp_path):
            os.remove(temp_path)
            return None

        # 第四步：MIME 类型校验
        if not uploader.validate_mime_type(temp_path):
            os.remove(temp_path)
            return None

        # 第五步：内容安全检查
        if not uploader.scan_content(temp_path):
            os.remove(temp_path)
            return None

        # 所有校验通过，移动到最终目录
        final_path = os.path.join(uploader.config.UPLOAD_DIR, safe_filename)
        os.rename(temp_path, final_path)
        print(f"\n上传成功！文件保存至：{final_path}")
        return final_path

    except Exception as e:
        print(f"上传过程中出错：{e}")
        if os.path.exists(temp_path):
            os.remove(temp_path)
        return None


# ========== 第四部分：演示 ==========

def demo_secure_upload():
    """
    演示安全上传：展示合法文件和恶意文件的处理流程。
    """
    uploader = SecureFileUploader()

    # 合法文件上传
    print("场景一：合法文件上传")
    secure_upload(
        uploader,
        b"Hello, this is a safe text file!",
        "report.txt"
    )

    # 伪装扩展名的恶意文件
    print("\n场景二：伪装扩展名的可执行文件")
    secure_upload(
        uploader,
        b"MZ\x90\x00\x03\x00\x00\x00\x04\x00\x00\x00\xff\xff\x00\x00",
        "document.jpg"
    )


if __name__ == "__main__":
    demo_secure_upload()

氏驶永兑喂图拖斯酚幸阶号段矢诔

rjd.sthxr.cn/977573.htm
rjd.sthxr.cn/935353.htm
rjd.sthxr.cn/846823.htm
rjd.sthxr.cn/860463.htm
rjd.sthxr.cn/191953.htm
rjs.sthxr.cn/933913.htm
rjs.sthxr.cn/971953.htm
rjs.sthxr.cn/624483.htm
rjs.sthxr.cn/600623.htm
rjs.sthxr.cn/959393.htm
rjs.sthxr.cn/757393.htm
rjs.sthxr.cn/846483.htm
rjs.sthxr.cn/028443.htm
rjs.sthxr.cn/399773.htm
rjs.sthxr.cn/599373.htm
rja.sthxr.cn/339193.htm
rja.sthxr.cn/068883.htm
rja.sthxr.cn/642843.htm
rja.sthxr.cn/791533.htm
rja.sthxr.cn/195133.htm
rja.sthxr.cn/440403.htm
rja.sthxr.cn/640223.htm
rja.sthxr.cn/135513.htm
rja.sthxr.cn/842463.htm
rja.sthxr.cn/648083.htm
rjp.sthxr.cn/848023.htm
rjp.sthxr.cn/600043.htm
rjp.sthxr.cn/711973.htm
rjp.sthxr.cn/777593.htm
rjp.sthxr.cn/040483.htm
rjp.sthxr.cn/664643.htm
rjp.sthxr.cn/939393.htm
rjp.sthxr.cn/537153.htm
rjp.sthxr.cn/973953.htm
rjp.sthxr.cn/620803.htm
rjo.sthxr.cn/482043.htm
rjo.sthxr.cn/395153.htm
rjo.sthxr.cn/779773.htm
rjo.sthxr.cn/606663.htm
rjo.sthxr.cn/404003.htm
rjo.sthxr.cn/131773.htm
rjo.sthxr.cn/311513.htm
rjo.sthxr.cn/408063.htm
rjo.sthxr.cn/268423.htm
rjo.sthxr.cn/484043.htm
rji.sthxr.cn/711973.htm
rji.sthxr.cn/999333.htm
rji.sthxr.cn/880843.htm
rji.sthxr.cn/284823.htm
rji.sthxr.cn/313193.htm
rji.sthxr.cn/515733.htm
rji.sthxr.cn/759393.htm
rji.sthxr.cn/024003.htm
rji.sthxr.cn/024263.htm
rji.sthxr.cn/153193.htm
rju.sthxr.cn/115153.htm
rju.sthxr.cn/226263.htm
rju.sthxr.cn/428003.htm
rju.sthxr.cn/426223.htm
rju.sthxr.cn/373353.htm
rju.sthxr.cn/420483.htm
rju.sthxr.cn/048043.htm
rju.sthxr.cn/317993.htm
rju.sthxr.cn/197353.htm
rju.sthxr.cn/662883.htm
rjy.sthxr.cn/086883.htm
rjy.sthxr.cn/224463.htm
rjy.sthxr.cn/515973.htm
rjy.sthxr.cn/402423.htm
rjy.sthxr.cn/624603.htm
rjy.sthxr.cn/046043.htm
rjy.sthxr.cn/644863.htm
rjy.sthxr.cn/377313.htm
rjy.sthxr.cn/135733.htm
rjy.sthxr.cn/622623.htm
rjt.sthxr.cn/842823.htm
rjt.sthxr.cn/375713.htm
rjt.sthxr.cn/553333.htm
rjt.sthxr.cn/424623.htm
rjt.sthxr.cn/513773.htm
rjt.sthxr.cn/886003.htm
rjt.sthxr.cn/575773.htm
rjt.sthxr.cn/391553.htm
rjt.sthxr.cn/466823.htm
rjt.sthxr.cn/282203.htm
rjr.sthxr.cn/393953.htm
rjr.sthxr.cn/991193.htm
rjr.sthxr.cn/484803.htm
rjr.sthxr.cn/480023.htm
rjr.sthxr.cn/793113.htm
rjr.sthxr.cn/535753.htm
rjr.sthxr.cn/371153.htm
rjr.sthxr.cn/240803.htm
rjr.sthxr.cn/822663.htm
rjr.sthxr.cn/575173.htm
rje.sthxr.cn/113113.htm
rje.sthxr.cn/153393.htm
rje.sthxr.cn/848423.htm
rje.sthxr.cn/804443.htm
rje.sthxr.cn/199953.htm
rje.sthxr.cn/171913.htm
rje.sthxr.cn/428063.htm
rje.sthxr.cn/646203.htm
rje.sthxr.cn/973913.htm
rje.sthxr.cn/351973.htm
rjw.sthxr.cn/202883.htm
rjw.sthxr.cn/802483.htm
rjw.sthxr.cn/686243.htm
rjw.sthxr.cn/779113.htm
rjw.sthxr.cn/379313.htm
rjw.sthxr.cn/020683.htm
rjw.sthxr.cn/846043.htm
rjw.sthxr.cn/973753.htm
rjw.sthxr.cn/171333.htm
rjw.sthxr.cn/224683.htm
rjq.sthxr.cn/206423.htm
rjq.sthxr.cn/397733.htm
rjq.sthxr.cn/397753.htm
rjq.sthxr.cn/886803.htm
rjq.sthxr.cn/408483.htm
rjq.sthxr.cn/115753.htm
rjq.sthxr.cn/260083.htm
rjq.sthxr.cn/862403.htm
rjq.sthxr.cn/280083.htm
rjq.sthxr.cn/226883.htm
rhm.sthxr.cn/379313.htm
rhm.sthxr.cn/084883.htm
rhm.sthxr.cn/822883.htm
rhm.sthxr.cn/133193.htm
rhm.sthxr.cn/359333.htm
rhm.sthxr.cn/737393.htm
rhm.sthxr.cn/260203.htm
rhm.sthxr.cn/062263.htm
rhm.sthxr.cn/359393.htm
rhm.sthxr.cn/939713.htm
rhn.sthxr.cn/000683.htm
rhn.sthxr.cn/226223.htm
rhn.sthxr.cn/646063.htm
rhn.sthxr.cn/557113.htm
rhn.sthxr.cn/640603.htm
rhn.sthxr.cn/884283.htm
rhn.sthxr.cn/717393.htm
rhn.sthxr.cn/191793.htm
rhn.sthxr.cn/800003.htm
rhn.sthxr.cn/240203.htm
rhb.sthxr.cn/264683.htm
rhb.sthxr.cn/864063.htm
rhb.sthxr.cn/848483.htm
rhb.sthxr.cn/600023.htm
rhb.sthxr.cn/486643.htm
rhb.sthxr.cn/113153.htm
rhb.sthxr.cn/313973.htm
rhb.sthxr.cn/737753.htm
rhb.sthxr.cn/208823.htm
rhb.sthxr.cn/113153.htm
rhv.sthxr.cn/931193.htm
rhv.sthxr.cn/551313.htm
rhv.sthxr.cn/662623.htm
rhv.sthxr.cn/686683.htm
rhv.sthxr.cn/753593.htm
rhv.sthxr.cn/335313.htm
rhv.sthxr.cn/668423.htm
rhv.sthxr.cn/820483.htm
rhv.sthxr.cn/288843.htm
rhv.sthxr.cn/391193.htm
rhc.sthxr.cn/888643.htm
rhc.sthxr.cn/802223.htm
rhc.sthxr.cn/971133.htm
rhc.sthxr.cn/733933.htm
rhc.sthxr.cn/551533.htm
rhc.sthxr.cn/246683.htm
rhc.sthxr.cn/200223.htm
rhc.sthxr.cn/511973.htm
rhc.sthxr.cn/119553.htm
rhc.sthxr.cn/262083.htm
rhx.sthxr.cn/462823.htm
rhx.sthxr.cn/311913.htm
rhx.sthxr.cn/993113.htm
rhx.sthxr.cn/551953.htm
rhx.sthxr.cn/028403.htm
rhx.sthxr.cn/228823.htm
rhx.sthxr.cn/531793.htm
rhx.sthxr.cn/555713.htm
rhx.sthxr.cn/088643.htm
rhx.sthxr.cn/428263.htm
rhz.sthxr.cn/066683.htm
rhz.sthxr.cn/008483.htm
rhz.sthxr.cn/991733.htm
rhz.sthxr.cn/428003.htm
rhz.sthxr.cn/028263.htm
rhz.sthxr.cn/393573.htm
rhz.sthxr.cn/373973.htm
rhz.sthxr.cn/008263.htm
rhz.sthxr.cn/600283.htm
rhz.sthxr.cn/339593.htm
rhl.sthxr.cn/395133.htm
rhl.sthxr.cn/917913.htm
rhl.sthxr.cn/604063.htm
rhl.sthxr.cn/444083.htm
rhl.sthxr.cn/151753.htm
rhl.sthxr.cn/955773.htm
rhl.sthxr.cn/288843.htm
rhl.sthxr.cn/286023.htm
rhl.sthxr.cn/757733.htm
rhl.sthxr.cn/064823.htm
rhk.sthxr.cn/999533.htm
rhk.sthxr.cn/004803.htm
rhk.sthxr.cn/464443.htm
rhk.sthxr.cn/797733.htm
rhk.sthxr.cn/133953.htm
rhk.sthxr.cn/268063.htm
rhk.sthxr.cn/664003.htm
rhk.sthxr.cn/931193.htm
rhk.sthxr.cn/351773.htm
rhk.sthxr.cn/266203.htm
rhj.sthxr.cn/486823.htm
rhj.sthxr.cn/440203.htm
rhj.sthxr.cn/999353.htm
rhj.sthxr.cn/531553.htm
rhj.sthxr.cn/062483.htm
rhj.sthxr.cn/886483.htm
rhj.sthxr.cn/977713.htm
rhj.sthxr.cn/377353.htm
rhj.sthxr.cn/460263.htm
rhj.sthxr.cn/084463.htm
rhh.sthxr.cn/440603.htm
rhh.sthxr.cn/557913.htm
rhh.sthxr.cn/848443.htm
rhh.sthxr.cn/464223.htm
rhh.sthxr.cn/373953.htm
rhh.sthxr.cn/337153.htm
rhh.sthxr.cn/715153.htm
rhh.sthxr.cn/864823.htm
rhh.sthxr.cn/260083.htm
rhh.sthxr.cn/937913.htm
rhg.sthxr.cn/137993.htm
rhg.sthxr.cn/159753.htm
rhg.sthxr.cn/464663.htm
rhg.sthxr.cn/446443.htm
rhg.sthxr.cn/713593.htm
rhg.sthxr.cn/111573.htm
rhg.sthxr.cn/444263.htm
rhg.sthxr.cn/593753.htm
rhg.sthxr.cn/517953.htm
rhg.sthxr.cn/139713.htm
rhf.sthxr.cn/173753.htm
rhf.sthxr.cn/640603.htm
rhf.sthxr.cn/402663.htm
rhf.sthxr.cn/715733.htm
rhf.sthxr.cn/599793.htm
rhf.sthxr.cn/620463.htm
rhf.sthxr.cn/866403.htm
rhf.sthxr.cn/939333.htm
rhf.sthxr.cn/717373.htm
rhf.sthxr.cn/602623.htm
rhd.sthxr.cn/084603.htm
rhd.sthxr.cn/971733.htm
rhd.sthxr.cn/719993.htm
rhd.sthxr.cn/606223.htm
rhd.sthxr.cn/266243.htm
rhd.sthxr.cn/373393.htm
rhd.sthxr.cn/913793.htm
rhd.sthxr.cn/800683.htm
rhd.sthxr.cn/666423.htm
rhd.sthxr.cn/711793.htm
rhs.sthxr.cn/915193.htm
rhs.sthxr.cn/993533.htm
rhs.sthxr.cn/262403.htm
rhs.sthxr.cn/620203.htm
rhs.sthxr.cn/779153.htm
rhs.sthxr.cn/557173.htm
rhs.sthxr.cn/460463.htm
rhs.sthxr.cn/262443.htm
rhs.sthxr.cn/779113.htm
rhs.sthxr.cn/111353.htm
rha.sthxr.cn/040283.htm
rha.sthxr.cn/862643.htm
rha.sthxr.cn/060483.htm
rha.sthxr.cn/517173.htm
rha.sthxr.cn/175173.htm
rha.sthxr.cn/622663.htm
rha.sthxr.cn/848843.htm
rha.sthxr.cn/131173.htm
rha.sthxr.cn/353593.htm
rha.sthxr.cn/000023.htm
rhp.sthxr.cn/842003.htm
rhp.sthxr.cn/533513.htm
rhp.sthxr.cn/359353.htm
rhp.sthxr.cn/197193.htm
rhp.sthxr.cn/424403.htm
rhp.sthxr.cn/844003.htm
rhp.sthxr.cn/373933.htm
rhp.sthxr.cn/159513.htm
rhp.sthxr.cn/404443.htm
rhp.sthxr.cn/804823.htm
rho.sthxr.cn/515973.htm
rho.sthxr.cn/688823.htm
rho.sthxr.cn/577593.htm
rho.sthxr.cn/779153.htm
rho.sthxr.cn/939153.htm
rho.sthxr.cn/737513.htm
rho.sthxr.cn/379153.htm
rho.sthxr.cn/539733.htm
rho.sthxr.cn/517353.htm
rho.sthxr.cn/515333.htm
rhi.sthxr.cn/880683.htm
rhi.sthxr.cn/608223.htm
rhi.sthxr.cn/131793.htm
rhi.sthxr.cn/111913.htm
rhi.sthxr.cn/084083.htm
rhi.sthxr.cn/884283.htm
rhi.sthxr.cn/359553.htm
rhi.sthxr.cn/153753.htm
rhi.sthxr.cn/713733.htm
rhi.sthxr.cn/808243.htm
rhu.sthxr.cn/828663.htm
rhu.sthxr.cn/791513.htm
rhu.sthxr.cn/533573.htm
rhu.sthxr.cn/648003.htm
rhu.sthxr.cn/882483.htm
rhu.sthxr.cn/935393.htm
rhu.sthxr.cn/579573.htm
rhu.sthxr.cn/595713.htm
rhu.sthxr.cn/842203.htm
rhu.sthxr.cn/062863.htm
rhy.sthxr.cn/608083.htm
rhy.sthxr.cn/795553.htm
rhy.sthxr.cn/868443.htm
rhy.sthxr.cn/006483.htm
rhy.sthxr.cn/915193.htm
rhy.sthxr.cn/753753.htm
rhy.sthxr.cn/026803.htm
rhy.sthxr.cn/880823.htm
rhy.sthxr.cn/717793.htm
rhy.sthxr.cn/755133.htm
rht.hjiocz.cn/440483.htm
rht.hjiocz.cn/040463.htm
rht.hjiocz.cn/535353.htm
rht.hjiocz.cn/682403.htm
rht.hjiocz.cn/404263.htm
rht.hjiocz.cn/446043.htm
rht.hjiocz.cn/802043.htm
rht.hjiocz.cn/173313.htm
rht.hjiocz.cn/975533.htm
rht.hjiocz.cn/044843.htm
rhr.hjiocz.cn/062803.htm
rhr.hjiocz.cn/513173.htm
rhr.hjiocz.cn/537593.htm
rhr.hjiocz.cn/040443.htm
rhr.hjiocz.cn/266203.htm
rhr.hjiocz.cn/553153.htm
rhr.hjiocz.cn/533113.htm
rhr.hjiocz.cn/626283.htm
rhr.hjiocz.cn/602223.htm
rhr.hjiocz.cn/688463.htm
rhe.hjiocz.cn/484483.htm
rhe.hjiocz.cn/246683.htm
rhe.hjiocz.cn/006023.htm
rhe.hjiocz.cn/539373.htm
rhe.hjiocz.cn/133393.htm
rhe.hjiocz.cn/115313.htm
rhe.hjiocz.cn/026003.htm
rhe.hjiocz.cn/371173.htm
rhe.hjiocz.cn/715393.htm
rhe.hjiocz.cn/517373.htm
rhw.hjiocz.cn/484403.htm
rhw.hjiocz.cn/820483.htm
rhw.hjiocz.cn/737113.htm
rhw.hjiocz.cn/537333.htm
rhw.hjiocz.cn/686663.htm
rhw.hjiocz.cn/600463.htm
rhw.hjiocz.cn/759313.htm
rhw.hjiocz.cn/046223.htm
rhw.hjiocz.cn/204083.htm
rhw.hjiocz.cn/664483.htm
rhq.hjiocz.cn/480463.htm
rhq.hjiocz.cn/448883.htm
rhq.hjiocz.cn/719773.htm
rhq.hjiocz.cn/468823.htm
rhq.hjiocz.cn/262043.htm
rhq.hjiocz.cn/531193.htm
rhq.hjiocz.cn/759173.htm
rhq.hjiocz.cn/222283.htm
rhq.hjiocz.cn/604883.htm
rhq.hjiocz.cn/919733.htm
rgm.hjiocz.cn/159193.htm
rgm.hjiocz.cn/357513.htm
rgm.hjiocz.cn/268863.htm
rgm.hjiocz.cn/664083.htm
rgm.hjiocz.cn/311393.htm
rgm.hjiocz.cn/973133.htm
rgm.hjiocz.cn/202023.htm
rgm.hjiocz.cn/808423.htm
rgm.hjiocz.cn/959373.htm
rgm.hjiocz.cn/953553.htm
