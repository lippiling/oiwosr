牡炭财温咐


"""
Python 依赖安全检查 —— 自动扫描项目依赖中的已知漏洞
涵盖 safety、pip-audit、Dependabot 模拟和漏洞数据库查询
"""

# 安装依赖：pip install safety pip-audit
# 依赖安全是软件供应链安全的重要环节，需定期检查第三方包中的已知漏洞

import subprocess
import json
import os
import sys
import tempfile
import re
from typing import List, Dict, Optional, Tuple
from dataclasses import dataclass, field, asdict
from datetime import datetime

# ========== 第一部分：数据结构 ==========

@dataclass
class Vulnerability:
    """漏洞信息的数据结构"""
    package_name: str
    installed_version: str
    vulnerable_versions: str
    advisory: str                  # 漏洞描述
    cve_id: str                    # CVE 编号（如 CVE-2024-XXXX）
    severity: str                  # 严重级别（CRITICAL/HIGH/MEDIUM/LOW）
    fixed_version: str             # 修复版本
    more_info_url: str = ""        # 更多信息链接

    def __str__(self) -> str:
        return f"[{self.severity}] {self.cve_id}: {self.package_name} " \
               f"{self.installed_version} -> {self.fixed_version}"


@dataclass
class ScanResult:
    """扫描结果"""
    vulnerabilities: List[Vulnerability] = field(default_factory=list)
    scanned_packages: int = 0
    scan_time: str = ""
    is_clean: bool = True

    def summary(self) -> str:
        """生成扫描结果摘要"""
        critical = sum(1 for v in self.vulnerabilities if v.severity == "CRITICAL")
        high = sum(1 for v in self.vulnerabilities if v.severity == "HIGH")
        medium = sum(1 for v in self.vulnerabilities if v.severity == "MEDIUM")
        if self.is_clean:
            return f"安全：扫描了 {self.scanned_packages} 个包，未发现已知漏洞"
        return f"发现 {len(self.vulnerabilities)} 个漏洞 " \
               f"（严重：{critical}，高：{high}，中：{medium}）"


# ========== 第二部分：Safety 集成 ==========

class SafetyScanner:
    """
    使用 safety 库扫描 Python 依赖中的已知漏洞。
    safety 使用 pyup.io 的漏洞数据库（免费版有延迟，商业版实时更新）。
    """

    @staticmethod
    def scan_requirements(requirements_file: str) -> ScanResult:
        """
        扫描 requirements.txt 文件中的依赖。
        """
        result = ScanResult(scan_time=datetime.now().isoformat())
        if not os.path.exists(requirements_file):
            print(f"文件不存在：{requirements_file}")
            return result

        try:
            # 调用 safety check 命令
            cmd = [
                sys.executable, "-m", "safety", "check",
                "-r", requirements_file,
                "--json",           # JSON 格式输出
                "--full-report"     # 完整报告
            ]
            output = subprocess.check_output(
                cmd, stderr=subprocess.STDOUT, timeout=60
            ).decode()

            # 解析 safety 输出
            result = SafetyScanner._parse_safety_output(output)
            result.scan_time = datetime.now().isoformat()
            return result

        except subprocess.TimeoutExpired:
            print("Safety 扫描超时")
        except FileNotFoundError:
            print("Safety 未安装，请运行：pip install safety")
        except subprocess.CalledProcessError as e:
            # safety 在发现漏洞时返回非零退出码
            if e.returncode == 255:
                print("Safety 内部错误")
            else:
                return SafetyScanner._parse_safety_output(e.output.decode())
        except Exception as e:
            print(f"Safety 扫描异常：{e}")
        return result

    @staticmethod
    def _parse_safety_output(output: str) -> ScanResult:
        """
        解析 safety 的 JSON 输出为 ScanResult。
        """
        result = ScanResult(scan_time=datetime.now().isoformat())
        try:
            data = json.loads(output)
            for vuln in data.get("vulnerabilities", []):
                vulnerability = Vulnerability(
                    package_name=vuln.get("package_name", "unknown"),
                    installed_version=vuln.get("installed_version", ""),
                    vulnerable_versions=vuln.get("vulnerable_spec", ""),
                    advisory=vuln.get("advisory", ""),
                    cve_id=vuln.get("cve", "N/A"),
                    severity=vuln.get("severity", "UNKNOWN"),
                    fixed_version=vuln.get("fixed_version", ""),
                    more_info_url=vuln.get("more_info_url", "")
                )
                result.vulnerabilities.append(vulnerability)
            result.scanned_packages = data.get("scanned_packages", 0)
            result.is_clean = len(result.vulnerabilities) == 0
        except json.JSONDecodeError:
            print("解析 safety 输出失败")
        return result


# ========== 第三部分：pip-audit 集成 ==========

class PipAuditScanner:
    """
    使用 pip-audit 扫描依赖漏洞。
    pip-audit 是 PyPA 官方推荐的依赖安全检查工具，
    使用 PyPI 的 JSON API 获取漏洞信息。
    """

    @staticmethod
    def scan_project(project_dir: str = ".") -> ScanResult:
        """
        扫描项目的所有依赖。
        """
        result = ScanResult(scan_time=datetime.now().isoformat())

        try:
            cmd = [
                sys.executable, "-m", "pip_audit",
                "--project-dir", project_dir,
                "--format", "json",
                "--desc",           # 包含漏洞描述
            ]
            output = subprocess.check_output(
                cmd, stderr=subprocess.STDOUT, timeout=120
            ).decode()

            result = PipAuditScanner._parse_output(output)
            return result

        except subprocess.TimeoutExpired:
            print("pip-audit 扫描超时")
        except FileNotFoundError:
            print("pip-audit 未安装，请运行：pip install pip-audit")
        except subprocess.CalledProcessError as e:
            return PipAuditScanner._parse_output(e.output.decode())
        except Exception as e:
            print(f"pip-audit 扫描异常：{e}")
        return result

    @staticmethod
    def scan_package_list(packages: List[str]) -> ScanResult:
        """
        扫描指定的包列表。
        """
        # 创建一个临时文件包含包列表
        with tempfile.NamedTemporaryFile(mode="w", suffix=".txt",
                                         delete=False) as f:
            f.write("\n".join(packages))
            tmp_file = f.name

        try:
            cmd = [
                sys.executable, "-m", "pip_audit",
                "-r", tmp_file,
                "--format", "json"
            ]
            output = subprocess.check_output(
                cmd, stderr=subprocess.STDOUT, timeout=60
            ).decode()
            return PipAuditScanner._parse_output(output)
        finally:
            os.unlink(tmp_file)

    @staticmethod
    def _parse_output(output: str) -> ScanResult:
        """
        解析 pip-audit 的 JSON 输出。
        """
        result = ScanResult(scan_time=datetime.now().isoformat())
        try:
            data = json.loads(output)
            for pkg in data.get("dependencies", []):
                vulns = pkg.get("vulns", [])
                for vuln in vulns:
                    vulnerability = Vulnerability(
                        package_name=pkg.get("name", "unknown"),
                        installed_version=pkg.get("version", ""),
                        vulnerable_versions=vuln.get("vulnerable_versions", ""),
                        advisory=vuln.get("description", ""),
                        cve_id=vuln.get("id", "N/A"),
                        severity=vuln.get("severity", "UNKNOWN"),
                        fixed_version=vuln.get("fixed_version", ""),
                        more_info_url=vuln.get("url", "")
                    )
                    result.vulnerabilities.append(vulnerability)
                result.scanned_packages += 1
            result.is_clean = len(result.vulnerabilities) == 0
        except json.JSONDecodeError:
            print("解析 pip-audit 输出失败")
        return result


# ========== 第四部分：Dependabot 模拟 ==========

class DependabotSimulator:
    """
    模拟 GitHub Dependabot 的核心功能：
    1. 解析依赖文件
    2. 检查版本兼容性
    3. 生成更新建议
    """

    @staticmethod
    def analyze_dependency_file(filepath: str) -> Dict:
        """
        分析依赖文件，生成类似 Dependabot 的报告。
        """
        if not os.path.exists(filepath):
            return {"error": f"文件不存在：{filepath}"}

        with open(filepath, "r") as f:
            content = f.read()

        # 解析依赖
        dependencies = []
        for line in content.split("\n"):
            line = line.strip()
            if line and not line.startswith("#"):
                # 处理 ==、>=、<= 等版本约束
                match = re.match(r"([\w\-_]+)\s*([><=!~]+)\s*([\w.]+)", line)
                if match:
                    dependencies.append({
                        "name": match.group(1),
                        "operator": match.group(2),
                        "version": match.group(3)
                    })

        return {
            "file": filepath,
            "dependency_count": len(dependencies),
            "dependencies": dependencies,
            "analysis_time": datetime.now().isoformat(),
            "recommendations": DependabotSimulator._generate_recommendations(
                dependencies
            )
        }

    @staticmethod
    def _generate_recommendations(deps: List[Dict]) -> List[Dict]:
        """
        生成依赖更新建议。
        """
        recommendations = []
        for dep in deps:
            # 检查是否使用了精确版本固定（建议使用范围约束）
            if dep.get("operator") == "==":
                recommendations.append({
                    "package": dep["name"],
                    "current": f"{dep['operator']}{dep['version']}",
                    "suggestion": f">={dep['version']},<{int(dep['version'].split('.')[0]) + 1}.0.0",
                    "reason": "建议使用范围约束以接收补丁更新"
                })
        return recommendations


# ========== 第五部分：综合演示 ==========

def demo_dependency_audit():
    """
    演示依赖安全检查的完整流程。
    """
    print("=== 依赖安全检查演示 ===\n")

    # 创建示例 requirements 文件
    req_content = """# 项目依赖
flask==2.2.3
requests==2.28.1
django==3.2.18
cryptography==38.0.3
pyyaml==5.4
"""
    req_file = "demo_requirements.txt"
    with open(req_file, "w") as f:
        f.write(req_content)

    # 使用 Dependabot 模拟器分析
    print("--- Dependabot 依赖分析 ---")
    analysis = DependabotSimulator.analyze_dependency_file(req_file)
    print(f"发现 {analysis['dependency_count']} 个依赖")
    for rec in analysis.get("recommendations", []):
        print(f"  建议：{rec['package']} {rec['current']} -> {rec['suggestion']}")

    # 扫描结果汇总
    print("\n--- 安全建议 ---")
    print("""
    定期执行依赖安全检查的最佳实践：
    1. 每次 CI/CD 构建时自动扫描
    2. 使用 pip-audit 作为主要的漏洞检测工具
    3. 启用 GitHub Dependabot 自动创建更新 PR
    4. 订阅安全公告邮件列表（如 oss-security）
    5. 及时更新受影响的依赖包
    6. 使用锁定文件（requirements.txt 或 Pipfile.lock）
    """)

    # 清理临时文件
    if os.path.exists(req_file):
        os.unlink(req_file)


if __name__ == "__main__":
    demo_dependency_audit()

摆萍钙关炒簇靡指鼓闲痈陌捶毁伟

qlf.phf3mxb.cn/351575.htm
qlf.phf3mxb.cn/535135.htm
qlf.phf3mxb.cn/359935.htm
qlf.phf3mxb.cn/993955.htm
qlf.phf3mxb.cn/933995.htm
qlf.phf3mxb.cn/75.htm
qlf.phf3mxb.cn/351995.htm
qlf.phf3mxb.cn/537915.htm
qld.phf3mxb.cn/911395.htm
qld.phf3mxb.cn/993775.htm
qld.phf3mxb.cn/155595.htm
qld.phf3mxb.cn/371115.htm
qld.phf3mxb.cn/373315.htm
qld.phf3mxb.cn/991515.htm
qld.phf3mxb.cn/579915.htm
qld.phf3mxb.cn/737315.htm
qld.phf3mxb.cn/379915.htm
qld.phf3mxb.cn/393555.htm
qls.phf3mxb.cn/199175.htm
qls.phf3mxb.cn/317575.htm
qls.phf3mxb.cn/557355.htm
qls.phf3mxb.cn/393755.htm
qls.phf3mxb.cn/311915.htm
qls.phf3mxb.cn/115315.htm
qls.phf3mxb.cn/573195.htm
qls.phf3mxb.cn/353775.htm
qls.phf3mxb.cn/511315.htm
qls.phf3mxb.cn/913355.htm
qla.phf3mxb.cn/931115.htm
qla.phf3mxb.cn/139935.htm
qla.phf3mxb.cn/319715.htm
qla.phf3mxb.cn/951195.htm
qla.phf3mxb.cn/577955.htm
qla.phf3mxb.cn/197135.htm
qla.phf3mxb.cn/353935.htm
qla.phf3mxb.cn/773975.htm
qla.phf3mxb.cn/953915.htm
qla.phf3mxb.cn/977995.htm
qlp.phf3mxb.cn/539375.htm
qlp.phf3mxb.cn/557355.htm
qlp.phf3mxb.cn/175575.htm
qlp.phf3mxb.cn/919975.htm
qlp.phf3mxb.cn/733315.htm
qlp.phf3mxb.cn/157955.htm
qlp.phf3mxb.cn/171735.htm
qlp.phf3mxb.cn/917155.htm
qlp.phf3mxb.cn/573135.htm
qlp.phf3mxb.cn/759315.htm
qlo.phf3mxb.cn/795995.htm
qlo.phf3mxb.cn/511335.htm
qlo.phf3mxb.cn/339335.htm
qlo.phf3mxb.cn/959115.htm
qlo.phf3mxb.cn/131375.htm
qlo.phf3mxb.cn/737955.htm
qlo.phf3mxb.cn/933935.htm
qlo.phf3mxb.cn/737735.htm
qlo.phf3mxb.cn/753515.htm
qlo.phf3mxb.cn/531175.htm
qli.phf3mxb.cn/115115.htm
qli.phf3mxb.cn/593395.htm
qli.phf3mxb.cn/997575.htm
qli.phf3mxb.cn/777135.htm
qli.phf3mxb.cn/799555.htm
qli.phf3mxb.cn/555995.htm
qli.phf3mxb.cn/333355.htm
qli.phf3mxb.cn/797955.htm
qli.phf3mxb.cn/775555.htm
qli.phf3mxb.cn/991395.htm
qlu.phf3mxb.cn/353375.htm
qlu.phf3mxb.cn/337715.htm
qlu.phf3mxb.cn/557595.htm
qlu.phf3mxb.cn/759955.htm
qlu.phf3mxb.cn/395395.htm
qlu.phf3mxb.cn/311775.htm
qlu.phf3mxb.cn/973935.htm
qlu.phf3mxb.cn/539535.htm
qlu.phf3mxb.cn/799575.htm
qlu.phf3mxb.cn/753735.htm
qly.phf3mxb.cn/979115.htm
qly.phf3mxb.cn/553535.htm
qly.phf3mxb.cn/353555.htm
qly.phf3mxb.cn/731375.htm
qly.phf3mxb.cn/359795.htm
qly.phf3mxb.cn/393555.htm
qly.phf3mxb.cn/519595.htm
qly.phf3mxb.cn/353595.htm
qly.phf3mxb.cn/355395.htm
qly.phf3mxb.cn/515315.htm
qlt.phf3mxb.cn/313115.htm
qlt.phf3mxb.cn/175175.htm
qlt.phf3mxb.cn/151515.htm
qlt.phf3mxb.cn/337115.htm
qlt.phf3mxb.cn/119515.htm
qlt.phf3mxb.cn/797315.htm
qlt.phf3mxb.cn/135555.htm
qlt.phf3mxb.cn/179715.htm
qlt.phf3mxb.cn/377595.htm
qlt.phf3mxb.cn/313595.htm
qlr.phf3mxb.cn/571795.htm
qlr.phf3mxb.cn/177955.htm
qlr.phf3mxb.cn/351795.htm
qlr.phf3mxb.cn/337975.htm
qlr.phf3mxb.cn/559115.htm
qlr.phf3mxb.cn/933795.htm
qlr.phf3mxb.cn/133375.htm
qlr.phf3mxb.cn/397115.htm
qlr.phf3mxb.cn/557915.htm
qlr.phf3mxb.cn/157735.htm
qle.phf3mxb.cn/915355.htm
qle.phf3mxb.cn/371595.htm
qle.phf3mxb.cn/513975.htm
qle.phf3mxb.cn/533395.htm
qle.phf3mxb.cn/113995.htm
qle.phf3mxb.cn/399595.htm
qle.phf3mxb.cn/133715.htm
qle.phf3mxb.cn/919735.htm
qle.phf3mxb.cn/353715.htm
qle.phf3mxb.cn/351355.htm
qlw.phf3mxb.cn/513735.htm
qlw.phf3mxb.cn/975575.htm
qlw.phf3mxb.cn/119715.htm
qlw.phf3mxb.cn/951595.htm
qlw.phf3mxb.cn/357935.htm
qlw.phf3mxb.cn/515515.htm
qlw.phf3mxb.cn/135715.htm
qlw.phf3mxb.cn/153975.htm
qlw.phf3mxb.cn/977355.htm
qlw.phf3mxb.cn/719975.htm
qlq.phf3mxb.cn/151535.htm
qlq.phf3mxb.cn/199115.htm
qlq.phf3mxb.cn/375395.htm
qlq.phf3mxb.cn/331915.htm
qlq.phf3mxb.cn/737115.htm
qlq.phf3mxb.cn/797995.htm
qlq.phf3mxb.cn/319315.htm
qlq.phf3mxb.cn/973555.htm
qlq.phf3mxb.cn/933955.htm
qlq.phf3mxb.cn/519755.htm
qktv.phf3mxb.cn/753515.htm
qktv.phf3mxb.cn/191975.htm
qktv.phf3mxb.cn/151935.htm
qktv.phf3mxb.cn/377795.htm
qktv.phf3mxb.cn/775515.htm
qktv.phf3mxb.cn/117995.htm
qktv.phf3mxb.cn/353195.htm
qktv.phf3mxb.cn/797595.htm
qktv.phf3mxb.cn/979355.htm
qktv.phf3mxb.cn/937915.htm
qkn.phf3mxb.cn/353715.htm
qkn.phf3mxb.cn/731775.htm
qkn.phf3mxb.cn/115155.htm
qkn.phf3mxb.cn/775195.htm
qkn.phf3mxb.cn/913995.htm
qkn.phf3mxb.cn/975915.htm
qkn.phf3mxb.cn/139995.htm
qkn.phf3mxb.cn/771335.htm
qkn.phf3mxb.cn/733575.htm
qkn.phf3mxb.cn/331915.htm
qkb.phf3mxb.cn/179995.htm
qkb.phf3mxb.cn/571935.htm
qkb.phf3mxb.cn/999195.htm
qkb.phf3mxb.cn/515355.htm
qkb.phf3mxb.cn/733735.htm
qkb.phf3mxb.cn/975775.htm
qkb.phf3mxb.cn/173955.htm
qkb.phf3mxb.cn/733315.htm
qkb.phf3mxb.cn/191115.htm
qkb.phf3mxb.cn/955795.htm
qkv.phf3mxb.cn/931555.htm
qkv.phf3mxb.cn/959995.htm
qkv.phf3mxb.cn/133975.htm
qkv.phf3mxb.cn/919515.htm
qkv.phf3mxb.cn/331115.htm
qkv.phf3mxb.cn/313135.htm
qkv.phf3mxb.cn/959575.htm
qkv.phf3mxb.cn/153115.htm
qkv.phf3mxb.cn/311755.htm
qkv.phf3mxb.cn/397395.htm
qkc.phf3mxb.cn/133115.htm
qkc.phf3mxb.cn/757355.htm
qkc.phf3mxb.cn/715995.htm
qkc.phf3mxb.cn/791795.htm
qkc.phf3mxb.cn/935335.htm
qkc.phf3mxb.cn/315135.htm
qkc.phf3mxb.cn/155755.htm
qkc.phf3mxb.cn/779595.htm
qkc.phf3mxb.cn/977595.htm
qkc.phf3mxb.cn/731755.htm
qkx.phf3mxb.cn/395355.htm
qkx.phf3mxb.cn/975715.htm
qkx.phf3mxb.cn/359715.htm
qkx.phf3mxb.cn/795735.htm
qkx.phf3mxb.cn/355155.htm
qkx.phf3mxb.cn/117155.htm
qkx.phf3mxb.cn/351715.htm
qkx.phf3mxb.cn/937955.htm
qkx.phf3mxb.cn/715155.htm
qkx.phf3mxb.cn/377975.htm
qkz.phf3mxb.cn/337795.htm
qkz.phf3mxb.cn/579175.htm
qkz.phf3mxb.cn/117715.htm
qkz.phf3mxb.cn/311375.htm
qkz.phf3mxb.cn/111515.htm
qkz.phf3mxb.cn/117375.htm
qkz.phf3mxb.cn/711375.htm
qkz.phf3mxb.cn/999955.htm
qkz.phf3mxb.cn/993795.htm
qkz.phf3mxb.cn/111775.htm
qkl.phf3mxb.cn/333515.htm
qkl.phf3mxb.cn/115395.htm
qkl.phf3mxb.cn/951715.htm
qkl.phf3mxb.cn/191935.htm
qkl.phf3mxb.cn/353175.htm
qkl.phf3mxb.cn/999315.htm
qkl.phf3mxb.cn/139995.htm
qkl.phf3mxb.cn/979755.htm
qkl.phf3mxb.cn/533575.htm
qkl.phf3mxb.cn/759175.htm
qkk.phf3mxb.cn/777595.htm
qkk.phf3mxb.cn/731955.htm
qkk.phf3mxb.cn/991335.htm
qkk.phf3mxb.cn/793115.htm
qkk.phf3mxb.cn/913935.htm
qkk.phf3mxb.cn/791135.htm
qkk.phf3mxb.cn/531755.htm
qkk.phf3mxb.cn/799195.htm
qkk.phf3mxb.cn/353175.htm
qkk.phf3mxb.cn/113155.htm
qkj.phf3mxb.cn/159975.htm
qkj.phf3mxb.cn/139355.htm
qkj.phf3mxb.cn/997595.htm
qkj.phf3mxb.cn/339955.htm
qkj.phf3mxb.cn/711355.htm
qkj.phf3mxb.cn/533375.htm
qkj.phf3mxb.cn/599555.htm
qkj.phf3mxb.cn/193915.htm
qkj.phf3mxb.cn/113735.htm
qkj.phf3mxb.cn/519775.htm
qkh.phf3mxb.cn/117115.htm
qkh.phf3mxb.cn/751155.htm
qkh.phf3mxb.cn/579535.htm
qkh.phf3mxb.cn/575995.htm
qkh.phf3mxb.cn/771915.htm
qkh.phf3mxb.cn/153535.htm
qkh.phf3mxb.cn/173955.htm
qkh.phf3mxb.cn/531155.htm
qkh.phf3mxb.cn/915195.htm
qkh.phf3mxb.cn/531375.htm
qkg.phf3mxb.cn/799155.htm
qkg.phf3mxb.cn/393155.htm
qkg.phf3mxb.cn/139535.htm
qkg.phf3mxb.cn/997315.htm
qkg.phf3mxb.cn/179555.htm
qkg.phf3mxb.cn/313775.htm
qkg.phf3mxb.cn/339395.htm
qkg.phf3mxb.cn/571315.htm
qkg.phf3mxb.cn/799535.htm
qkg.phf3mxb.cn/331935.htm
qkf.phf3mxb.cn/995115.htm
qkf.phf3mxb.cn/313135.htm
qkf.phf3mxb.cn/993375.htm
qkf.phf3mxb.cn/737375.htm
qkf.phf3mxb.cn/175755.htm
qkf.phf3mxb.cn/533535.htm
qkf.phf3mxb.cn/373395.htm
qkf.phf3mxb.cn/715375.htm
qkf.phf3mxb.cn/793535.htm
qkf.phf3mxb.cn/171735.htm
qkd.phf3mxb.cn/117795.htm
qkd.phf3mxb.cn/793955.htm
qkd.phf3mxb.cn/917375.htm
qkd.phf3mxb.cn/937155.htm
qkd.phf3mxb.cn/979715.htm
qkd.phf3mxb.cn/755575.htm
qkd.phf3mxb.cn/779935.htm
qkd.phf3mxb.cn/517395.htm
qkd.phf3mxb.cn/917915.htm
qkd.phf3mxb.cn/531535.htm
qks.phf3mxb.cn/757735.htm
qks.phf3mxb.cn/393535.htm
qks.phf3mxb.cn/575535.htm
qks.phf3mxb.cn/335335.htm
qks.phf3mxb.cn/555375.htm
qks.phf3mxb.cn/351975.htm
qks.phf3mxb.cn/357795.htm
qks.phf3mxb.cn/911935.htm
qks.phf3mxb.cn/391395.htm
qks.phf3mxb.cn/759555.htm
qka.phf3mxb.cn/311595.htm
qka.phf3mxb.cn/559995.htm
qka.phf3mxb.cn/535755.htm
qka.phf3mxb.cn/151195.htm
qka.phf3mxb.cn/531595.htm
qka.phf3mxb.cn/571535.htm
qka.phf3mxb.cn/919955.htm
qka.phf3mxb.cn/177175.htm
qka.phf3mxb.cn/979375.htm
qka.phf3mxb.cn/751515.htm
qkp.phf3mxb.cn/919575.htm
qkp.phf3mxb.cn/579995.htm
qkp.phf3mxb.cn/795735.htm
qkp.phf3mxb.cn/199735.htm
qkp.phf3mxb.cn/717335.htm
qkp.phf3mxb.cn/317595.htm
qkp.phf3mxb.cn/757775.htm
qkp.phf3mxb.cn/735135.htm
qkp.phf3mxb.cn/179955.htm
qkp.phf3mxb.cn/995315.htm
qko.phf3mxb.cn/597555.htm
qko.phf3mxb.cn/793995.htm
qko.phf3mxb.cn/917395.htm
qko.phf3mxb.cn/593935.htm
qko.phf3mxb.cn/713595.htm
qko.phf3mxb.cn/379315.htm
qko.phf3mxb.cn/793555.htm
qko.phf3mxb.cn/575735.htm
qko.phf3mxb.cn/557515.htm
qko.phf3mxb.cn/533395.htm
qki.phf3mxb.cn/517775.htm
qki.phf3mxb.cn/315135.htm
qki.phf3mxb.cn/573155.htm
qki.phf3mxb.cn/975555.htm
qki.phf3mxb.cn/591175.htm
qki.phf3mxb.cn/199755.htm
qki.phf3mxb.cn/931595.htm
qki.phf3mxb.cn/153915.htm
qki.phf3mxb.cn/791735.htm
qki.phf3mxb.cn/155375.htm
qku.phf3mxb.cn/193795.htm
qku.phf3mxb.cn/595175.htm
qku.phf3mxb.cn/595135.htm
qku.phf3mxb.cn/911375.htm
qku.phf3mxb.cn/737555.htm
qku.phf3mxb.cn/197935.htm
qku.phf3mxb.cn/311375.htm
qku.phf3mxb.cn/337795.htm
qku.phf3mxb.cn/371115.htm
qku.phf3mxb.cn/935335.htm
qky.phf3mxb.cn/317335.htm
qky.phf3mxb.cn/133555.htm
qky.phf3mxb.cn/315315.htm
qky.phf3mxb.cn/553555.htm
qky.phf3mxb.cn/191935.htm
qky.phf3mxb.cn/775155.htm
qky.phf3mxb.cn/973975.htm
qky.phf3mxb.cn/933935.htm
qky.phf3mxb.cn/131575.htm
qky.phf3mxb.cn/339575.htm
qkt.phf3mxb.cn/511755.htm
qkt.phf3mxb.cn/139975.htm
qkt.phf3mxb.cn/775115.htm
qkt.phf3mxb.cn/579775.htm
qkt.phf3mxb.cn/393335.htm
qkt.phf3mxb.cn/593735.htm
qkt.phf3mxb.cn/195915.htm
qkt.phf3mxb.cn/333555.htm
qkt.phf3mxb.cn/935155.htm
qkt.phf3mxb.cn/397995.htm
qkr.phf3mxb.cn/335395.htm
qkr.phf3mxb.cn/111715.htm
qkr.phf3mxb.cn/159715.htm
qkr.phf3mxb.cn/759195.htm
qkr.phf3mxb.cn/333975.htm
qkr.phf3mxb.cn/773515.htm
qkr.phf3mxb.cn/355955.htm
qkr.phf3mxb.cn/775535.htm
qkr.phf3mxb.cn/557315.htm
qkr.phf3mxb.cn/399995.htm
qke.phf3mxb.cn/559975.htm
qke.phf3mxb.cn/531375.htm
qke.phf3mxb.cn/159935.htm
qke.phf3mxb.cn/993995.htm
qke.phf3mxb.cn/773575.htm
qke.phf3mxb.cn/331175.htm
qke.phf3mxb.cn/319355.htm
qke.phf3mxb.cn/333975.htm
qke.phf3mxb.cn/551395.htm
qke.phf3mxb.cn/379995.htm
qkw.phf3mxb.cn/977575.htm
qkw.phf3mxb.cn/933735.htm
qkw.phf3mxb.cn/971735.htm
qkw.phf3mxb.cn/537135.htm
qkw.phf3mxb.cn/773935.htm
qkw.phf3mxb.cn/155135.htm
qkw.phf3mxb.cn/551395.htm
qkw.phf3mxb.cn/939555.htm
qkw.phf3mxb.cn/151355.htm
qkw.phf3mxb.cn/171315.htm
qkq.phf3mxb.cn/337315.htm
qkq.phf3mxb.cn/731115.htm
qkq.phf3mxb.cn/391195.htm
qkq.phf3mxb.cn/575375.htm
qkq.phf3mxb.cn/971155.htm
qkq.phf3mxb.cn/979195.htm
qkq.phf3mxb.cn/195135.htm
