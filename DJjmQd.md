勒艘奶奄于


Python命令行参数与环境变量——argparse与os.environ的整合策略

在实际应用中，命令行参数和环境变量经常混合使用。合理的优先级策略保证配置灵活性和安全性。

import argparse
import os
import sys
from pathlib import Path

# ========== os.environ.get 的基本用法 ==========
def load_database_config():
    """从环境变量加载数据库配置，提供合理的默认值"""
    return {
        "host": os.environ.get("DB_HOST", "localhost"),       # 默认本地
        "port": int(os.environ.get("DB_PORT", "5432")),       # 默认PostgreSQL端口
        "user": os.environ.get("DB_USER", "app_user"),
        "password": os.environ.get("DB_PASSWORD", ""),        # 无默认密码
        "database": os.environ.get("DB_NAME", "myapp"),
    }


# 安全获取环境变量的工具函数
def get_env_int(key: str, default: int) -> int:
    """安全地将环境变量解析为整数"""
    raw = os.environ.get(key)
    if raw is None:
        return default
    try:
        return int(raw)
    except ValueError:
        print(f"警告: 环境变量{key}的值'{raw}'不是有效整数，使用默认值{default}")
        return default


# ========== argparse 结合环境变量回退 ==========
def create_parser() -> argparse.ArgumentParser:
    """创建命令行解析器，每个参数支持环境变量回退"""
    parser = argparse.ArgumentParser(
        description="应用启动配置",
        formatter_class=argparse.ArgumentDefaultsHelpFormatter,
    )

    # 使用ENVVAR/type模式：先读命令行，再读环境变量，最后用默认值
    parser.add_argument(
        "--host",
        default=os.environ.get("APP_HOST", "0.0.0.0"),
        help="监听地址 (环境变量: APP_HOST)",
    )
    parser.add_argument(
        "--port",
        type=int,
        default=get_env_int("APP_PORT", 8080),
        help="监听端口 (环境变量: APP_PORT)",
    )
    parser.add_argument(
        "--debug",
        action="store_true",
        default=os.environ.get("APP_DEBUG", "").lower() in ("true", "1"),
        help="调试模式 (环境变量: APP_DEBUG)",
    )
    parser.add_argument(
        "--config",
        default=os.environ.get("APP_CONFIG", "config.yaml"),
        help="配置文件路径 (环境变量: APP_CONFIG)",
    )
    parser.add_argument(
        "--log-level",
        default=os.environ.get("APP_LOG_LEVEL", "INFO"),
        choices=["DEBUG", "INFO", "WARNING", "ERROR"],
        help="日志级别 (环境变量: APP_LOG_LEVEL)",
    )
    return parser


# ========== 配置优先级链实现 ==========
# 优先级：命令行参数 > 环境变量 > 配置文件 > 默认值

class ConfigChain:
    """
    多级配置链：支持从多个来源合并配置。
    优先级从高到低：CLI参数 > 环境变量 > 配置文件 > 硬编码默认值。
    """

    DEFAULTS = {
        "host": "127.0.0.1",
        "port": 8080,
        "debug": False,
        "workers": 1,
    }

    def __init__(self, config_file: str = "config.yaml"):
        self._cli_args = {}
        self._config_file = config_file
        self._file_config = {}  # 从配置文件加载的值

    def load_file_config(self) -> dict:
        """从配置文件加载设置（简化示例）"""
        cfg_path = Path(self._config_file)
        if not cfg_path.exists():
            print(f"配置文件 {self._config_file} 不存在，使用默认值")
            return {}
        # 实际项目中应该用yaml/json解析
        return {"port": 9000, "workers": 4}  # 模拟读取

    def load_env_config(self) -> dict:
        """从环境变量加载配置"""
        config = {}
        for key in self.DEFAULTS:
            env_key = f"APP_{key.upper()}"
            val = os.environ.get(env_key)
            if val is not None:
                # 根据默认值类型进行转换
                default_val = self.DEFAULTS[key]
                if isinstance(default_val, bool):
                    config[key] = val.lower() in ("true", "1", "yes")
                elif isinstance(default_val, int):
                    config[key] = int(val)
                else:
                    config[key] = val
        return config

    def get(self, key: str):
        """按优先级链获取配置值"""
        # 第一优先级：命令行参数
        if key in self._cli_args:
            return self._cli_args[key]
        # 第二优先级：环境变量
        env_val = os.environ.get(f"APP_{key.upper()}")
        if env_val is not None:
            return self._convert_type(env_val, self.DEFAULTS.get(key))
        # 第三优先级：配置文件
        if key in self._file_config:
            return self._file_config[key]
        # 最低优先级：默认值
        return self.DEFAULTS.get(key)

    def _convert_type(self, val, template):
        if isinstance(template, bool):
            return val.lower() in ("true", "1")
        if isinstance(template, int):
            return int(val)
        return val

    def update_from_cli(self, args: dict):
        """从命令行参数更新"""
        self._cli_args.update(args)

    def __repr__(self):
        return f"ConfigChain({dict(self)})"

    def __iter__(self):
        for key in self.DEFAULTS:
            yield key
            # yield self.get(key)


# ========== 使用python-dotenv的示例模式 ==========
def try_load_dotenv():
    """尝试加载.env文件（如果存在）"""
    try:
        from dotenv import load_dotenv
        env_path = Path(".env")
        if env_path.exists():
            load_dotenv(env_path)
            print(f"已加载环境变量文件: {env_path}")
    except ImportError:
        print("python-dotenv未安装，跳过.env文件加载")
        pass


# 演示完整流程
if __name__ == "__main__":
    # 先加载.env文件
    try_load_dotenv()
    # 解析命令行参数
    parser = create_parser()
    args = parser.parse_args()
    print(f"最终配置: host={args.host}, port={args.port}, debug={args.debug}")

澜炔说质泄友橙姿链露拔窖窍怕汤

wqr.wfcj3t6.cn/680845.htm
wqr.wfcj3t6.cn/997975.htm
wqr.wfcj3t6.cn/319195.htm
wqe.aira2hc.cn/197175.htm
wqe.aira2hc.cn/828085.htm
wqe.aira2hc.cn/868285.htm
wqe.aira2hc.cn/139515.htm
wqe.aira2hc.cn/004005.htm
wqe.aira2hc.cn/539315.htm
wqe.aira2hc.cn/040005.htm
wqe.aira2hc.cn/486485.htm
wqe.aira2hc.cn/553315.htm
wqe.aira2hc.cn/228265.htm
wqw.aira2hc.cn/319375.htm
wqw.aira2hc.cn/866605.htm
wqw.aira2hc.cn/288825.htm
wqw.aira2hc.cn/575915.htm
wqw.aira2hc.cn/846825.htm
wqw.aira2hc.cn/535975.htm
wqw.aira2hc.cn/680485.htm
wqw.aira2hc.cn/282485.htm
wqw.aira2hc.cn/373795.htm
wqw.aira2hc.cn/068465.htm
wqq.aira2hc.cn/991315.htm
wqq.aira2hc.cn/951735.htm
wqq.aira2hc.cn/466285.htm
wqq.aira2hc.cn/686285.htm
wqq.aira2hc.cn/595995.htm
wqq.aira2hc.cn/535715.htm
wqq.aira2hc.cn/600605.htm
wqq.aira2hc.cn/080425.htm
wqq.aira2hc.cn/808005.htm
wqq.aira2hc.cn/040025.htm
qmtv.aira2hc.cn/913775.htm
qmtv.aira2hc.cn/020665.htm
qmtv.aira2hc.cn/660265.htm
qmtv.aira2hc.cn/282825.htm
qmtv.aira2hc.cn/646405.htm
qmtv.aira2hc.cn/246445.htm
qmtv.aira2hc.cn/048665.htm
qmtv.aira2hc.cn/668825.htm
qmtv.aira2hc.cn/995175.htm
qmtv.aira2hc.cn/022245.htm
qmn.aira2hc.cn/664245.htm
qmn.aira2hc.cn/773715.htm
qmn.aira2hc.cn/282485.htm
qmn.aira2hc.cn/977135.htm
qmn.aira2hc.cn/915335.htm
qmn.aira2hc.cn/886645.htm
qmn.aira2hc.cn/826085.htm
qmn.aira2hc.cn/462445.htm
qmn.aira2hc.cn/739715.htm
qmn.aira2hc.cn/313195.htm
qmb.aira2hc.cn/917915.htm
qmb.aira2hc.cn/820885.htm
qmb.aira2hc.cn/662065.htm
qmb.aira2hc.cn/242045.htm
qmb.aira2hc.cn/464005.htm
qmb.aira2hc.cn/917975.htm
qmb.aira2hc.cn/608085.htm
qmb.aira2hc.cn/573595.htm
qmb.aira2hc.cn/519755.htm
qmb.aira2hc.cn/242405.htm
qmv.aira2hc.cn/197975.htm
qmv.aira2hc.cn/682405.htm
qmv.aira2hc.cn/604005.htm
qmv.aira2hc.cn/826065.htm
qmv.aira2hc.cn/593935.htm
qmv.aira2hc.cn/488685.htm
qmv.aira2hc.cn/062005.htm
qmv.aira2hc.cn/159775.htm
qmv.aira2hc.cn/353935.htm
qmv.aira2hc.cn/482605.htm
qmc.aira2hc.cn/628865.htm
qmc.aira2hc.cn/020605.htm
qmc.aira2hc.cn/082605.htm
qmc.aira2hc.cn/933755.htm
qmc.aira2hc.cn/828245.htm
qmc.aira2hc.cn/062485.htm
qmc.aira2hc.cn/200265.htm
qmc.aira2hc.cn/511755.htm
qmc.aira2hc.cn/082825.htm
qmc.aira2hc.cn/044425.htm
qmx.aira2hc.cn/113375.htm
qmx.aira2hc.cn/791175.htm
qmx.aira2hc.cn/000605.htm
qmx.aira2hc.cn/264285.htm
qmx.aira2hc.cn/482865.htm
qmx.aira2hc.cn/751995.htm
qmx.aira2hc.cn/735755.htm
qmx.aira2hc.cn/404025.htm
qmx.aira2hc.cn/511995.htm
qmx.aira2hc.cn/406685.htm
qmz.aira2hc.cn/224485.htm
qmz.aira2hc.cn/802865.htm
qmz.aira2hc.cn/082025.htm
qmz.aira2hc.cn/286645.htm
qmz.aira2hc.cn/444465.htm
qmz.aira2hc.cn/224865.htm
qmz.aira2hc.cn/446085.htm
qmz.aira2hc.cn/864485.htm
qmz.aira2hc.cn/155555.htm
qmz.aira2hc.cn/775715.htm
qml.aira2hc.cn/193335.htm
qml.aira2hc.cn/373735.htm
qml.aira2hc.cn/244425.htm
qml.aira2hc.cn/119775.htm
qml.aira2hc.cn/648405.htm
qml.aira2hc.cn/462005.htm
qml.aira2hc.cn/424665.htm
qml.aira2hc.cn/462865.htm
qml.aira2hc.cn/333955.htm
qml.aira2hc.cn/228485.htm
qmk.aira2hc.cn/866425.htm
qmk.aira2hc.cn/042805.htm
qmk.aira2hc.cn/755155.htm
qmk.aira2hc.cn/795955.htm
qmk.aira2hc.cn/880405.htm
qmk.aira2hc.cn/008065.htm
qmk.aira2hc.cn/993595.htm
qmk.aira2hc.cn/206445.htm
qmk.aira2hc.cn/004645.htm
qmk.aira2hc.cn/200205.htm
qmj.aira2hc.cn/195715.htm
qmj.aira2hc.cn/177935.htm
qmj.aira2hc.cn/268025.htm
qmj.aira2hc.cn/771315.htm
qmj.aira2hc.cn/979555.htm
qmj.aira2hc.cn/660045.htm
qmj.aira2hc.cn/886865.htm
qmj.aira2hc.cn/288025.htm
qmj.aira2hc.cn/220005.htm
qmj.aira2hc.cn/339315.htm
qmh.aira2hc.cn/842425.htm
qmh.aira2hc.cn/535735.htm
qmh.aira2hc.cn/175575.htm
qmh.aira2hc.cn/440285.htm
qmh.aira2hc.cn/886445.htm
qmh.aira2hc.cn/404805.htm
qmh.aira2hc.cn/351755.htm
qmh.aira2hc.cn/993775.htm
qmh.aira2hc.cn/080625.htm
qmh.aira2hc.cn/951515.htm
qmg.aira2hc.cn/084645.htm
qmg.aira2hc.cn/137335.htm
qmg.aira2hc.cn/408225.htm
qmg.aira2hc.cn/022645.htm
qmg.aira2hc.cn/575555.htm
qmg.aira2hc.cn/006685.htm
qmg.aira2hc.cn/311715.htm
qmg.aira2hc.cn/113535.htm
qmg.aira2hc.cn/557755.htm
qmg.aira2hc.cn/911715.htm
qmf.aira2hc.cn/620085.htm
qmf.aira2hc.cn/024065.htm
qmf.aira2hc.cn/517355.htm
qmf.aira2hc.cn/482685.htm
qmf.aira2hc.cn/606065.htm
qmf.aira2hc.cn/666485.htm
qmf.aira2hc.cn/779795.htm
qmf.aira2hc.cn/531955.htm
qmf.aira2hc.cn/082225.htm
qmf.aira2hc.cn/333975.htm
qmd.aira2hc.cn/428465.htm
qmd.aira2hc.cn/460065.htm
qmd.aira2hc.cn/802825.htm
qmd.aira2hc.cn/864625.htm
qmd.aira2hc.cn/620445.htm
qmd.aira2hc.cn/395315.htm
qmd.aira2hc.cn/226485.htm
qmd.aira2hc.cn/028405.htm
qmd.aira2hc.cn/604665.htm
qmd.aira2hc.cn/224665.htm
qms.aira2hc.cn/660445.htm
qms.aira2hc.cn/406485.htm
qms.aira2hc.cn/824625.htm
qms.aira2hc.cn/799915.htm
qms.aira2hc.cn/420805.htm
qms.aira2hc.cn/206605.htm
qms.aira2hc.cn/842685.htm
qms.aira2hc.cn/937515.htm
qms.aira2hc.cn/733955.htm
qms.aira2hc.cn/646005.htm
qma.aira2hc.cn/919755.htm
qma.aira2hc.cn/424265.htm
qma.aira2hc.cn/204645.htm
qma.aira2hc.cn/242865.htm
qma.aira2hc.cn/177555.htm
qma.aira2hc.cn/199555.htm
qma.aira2hc.cn/062285.htm
qma.aira2hc.cn/808265.htm
qma.aira2hc.cn/626245.htm
qma.aira2hc.cn/246005.htm
qmp.aira2hc.cn/135955.htm
qmp.aira2hc.cn/680665.htm
qmp.aira2hc.cn/551395.htm
qmp.aira2hc.cn/800005.htm
qmp.aira2hc.cn/428045.htm
qmp.aira2hc.cn/028205.htm
qmp.aira2hc.cn/624885.htm
qmp.aira2hc.cn/319555.htm
qmp.aira2hc.cn/191735.htm
qmp.aira2hc.cn/488625.htm
qmo.aira2hc.cn/820485.htm
qmo.aira2hc.cn/577155.htm
qmo.aira2hc.cn/997375.htm
qmo.aira2hc.cn/844865.htm
qmo.aira2hc.cn/644865.htm
qmo.aira2hc.cn/062485.htm
qmo.aira2hc.cn/282445.htm
qmo.aira2hc.cn/864605.htm
qmo.aira2hc.cn/333535.htm
qmo.aira2hc.cn/446005.htm
qmi.aira2hc.cn/519915.htm
qmi.aira2hc.cn/820245.htm
qmi.aira2hc.cn/240445.htm
qmi.aira2hc.cn/820865.htm
qmi.aira2hc.cn/575195.htm
qmi.aira2hc.cn/440445.htm
qmi.aira2hc.cn/880465.htm
qmi.aira2hc.cn/422845.htm
qmi.aira2hc.cn/739375.htm
qmi.aira2hc.cn/686865.htm
qmu.aira2hc.cn/913955.htm
qmu.aira2hc.cn/157335.htm
qmu.aira2hc.cn/022285.htm
qmu.aira2hc.cn/688045.htm
qmu.aira2hc.cn/535995.htm
qmu.aira2hc.cn/668625.htm
qmu.aira2hc.cn/400685.htm
qmu.aira2hc.cn/626485.htm
qmu.aira2hc.cn/355735.htm
qmu.aira2hc.cn/620005.htm
qmy.aira2hc.cn/866825.htm
qmy.aira2hc.cn/797795.htm
qmy.aira2hc.cn/775795.htm
qmy.aira2hc.cn/997335.htm
qmy.aira2hc.cn/864405.htm
qmy.aira2hc.cn/866245.htm
qmy.aira2hc.cn/646425.htm
qmy.aira2hc.cn/402625.htm
qmy.aira2hc.cn/339955.htm
qmy.aira2hc.cn/119175.htm
qmt.aira2hc.cn/022445.htm
qmt.aira2hc.cn/197395.htm
qmt.aira2hc.cn/866645.htm
qmt.aira2hc.cn/911735.htm
qmt.aira2hc.cn/222225.htm
qmt.aira2hc.cn/466225.htm
qmt.aira2hc.cn/682405.htm
qmt.aira2hc.cn/028665.htm
qmt.aira2hc.cn/668445.htm
qmt.aira2hc.cn/991155.htm
qmr.aira2hc.cn/446805.htm
qmr.aira2hc.cn/717135.htm
qmr.aira2hc.cn/688865.htm
qmr.aira2hc.cn/088825.htm
qmr.aira2hc.cn/997315.htm
qmr.aira2hc.cn/046485.htm
qmr.aira2hc.cn/648025.htm
qmr.aira2hc.cn/804025.htm
qmr.aira2hc.cn/060685.htm
qmr.aira2hc.cn/024085.htm
qme.aira2hc.cn/420205.htm
qme.aira2hc.cn/553975.htm
qme.aira2hc.cn/664205.htm
qme.aira2hc.cn/068645.htm
qme.aira2hc.cn/199795.htm
qme.aira2hc.cn/755995.htm
qme.aira2hc.cn/622485.htm
qme.aira2hc.cn/824205.htm
qme.aira2hc.cn/228025.htm
qme.aira2hc.cn/862065.htm
qmw.aira2hc.cn/220405.htm
qmw.aira2hc.cn/868885.htm
qmw.aira2hc.cn/511975.htm
qmw.aira2hc.cn/397755.htm
qmw.aira2hc.cn/393115.htm
qmw.aira2hc.cn/668605.htm
qmw.aira2hc.cn/424085.htm
qmw.aira2hc.cn/204065.htm
qmw.aira2hc.cn/177375.htm
qmw.aira2hc.cn/355395.htm
qmq.aira2hc.cn/353155.htm
qmq.aira2hc.cn/200825.htm
qmq.aira2hc.cn/608645.htm
qmq.aira2hc.cn/595355.htm
qmq.aira2hc.cn/046025.htm
qmq.aira2hc.cn/608005.htm
qmq.aira2hc.cn/628085.htm
qmq.aira2hc.cn/440685.htm
qmq.aira2hc.cn/115395.htm
qmq.aira2hc.cn/757715.htm
qntv.aira2hc.cn/644285.htm
qntv.aira2hc.cn/204665.htm
qntv.aira2hc.cn/648005.htm
qntv.aira2hc.cn/862005.htm
qntv.aira2hc.cn/577155.htm
qntv.aira2hc.cn/886485.htm
qntv.aira2hc.cn/604265.htm
qntv.aira2hc.cn/779715.htm
qntv.aira2hc.cn/020485.htm
qntv.aira2hc.cn/688025.htm
qnn.aira2hc.cn/620045.htm
qnn.aira2hc.cn/084045.htm
qnn.aira2hc.cn/800085.htm
qnn.aira2hc.cn/539135.htm
qnn.aira2hc.cn/884265.htm
qnn.aira2hc.cn/286825.htm
qnn.aira2hc.cn/117375.htm
qnn.aira2hc.cn/200885.htm
qnn.aira2hc.cn/557935.htm
qnn.aira2hc.cn/082685.htm
qnb.aira2hc.cn/226485.htm
qnb.aira2hc.cn/515575.htm
qnb.aira2hc.cn/535755.htm
qnb.aira2hc.cn/048445.htm
qnb.aira2hc.cn/628405.htm
qnb.aira2hc.cn/333515.htm
qnb.aira2hc.cn/200245.htm
qnb.aira2hc.cn/571135.htm
qnb.aira2hc.cn/353555.htm
qnb.aira2hc.cn/864065.htm
qnv.aira2hc.cn/804225.htm
qnv.aira2hc.cn/422025.htm
qnv.aira2hc.cn/959555.htm
qnv.aira2hc.cn/420285.htm
qnv.aira2hc.cn/246005.htm
qnv.aira2hc.cn/373555.htm
qnv.aira2hc.cn/440885.htm
qnv.aira2hc.cn/284425.htm
qnv.aira2hc.cn/684665.htm
qnv.aira2hc.cn/157775.htm
qnc.aira2hc.cn/537715.htm
qnc.aira2hc.cn/319975.htm
qnc.aira2hc.cn/404065.htm
qnc.aira2hc.cn/824285.htm
qnc.aira2hc.cn/626025.htm
qnc.aira2hc.cn/979755.htm
qnc.aira2hc.cn/806445.htm
qnc.aira2hc.cn/826605.htm
qnc.aira2hc.cn/488865.htm
qnc.aira2hc.cn/399135.htm
qnx.aira2hc.cn/480025.htm
qnx.aira2hc.cn/662885.htm
qnx.aira2hc.cn/020465.htm
qnx.aira2hc.cn/953395.htm
qnx.aira2hc.cn/739735.htm
qnx.aira2hc.cn/060625.htm
qnx.aira2hc.cn/842645.htm
qnx.aira2hc.cn/597515.htm
qnx.aira2hc.cn/208485.htm
qnx.aira2hc.cn/959915.htm
qnz.aira2hc.cn/680225.htm
qnz.aira2hc.cn/862225.htm
qnz.aira2hc.cn/060405.htm
qnz.aira2hc.cn/444065.htm
qnz.aira2hc.cn/268225.htm
qnz.aira2hc.cn/866665.htm
qnz.aira2hc.cn/800885.htm
qnz.aira2hc.cn/828245.htm
qnz.aira2hc.cn/066025.htm
qnz.aira2hc.cn/284225.htm
qnl.aira2hc.cn/246085.htm
qnl.aira2hc.cn/595755.htm
qnl.aira2hc.cn/979575.htm
qnl.aira2hc.cn/662005.htm
qnl.aira2hc.cn/571975.htm
qnl.aira2hc.cn/997795.htm
qnl.aira2hc.cn/602085.htm
qnl.aira2hc.cn/313335.htm
qnl.aira2hc.cn/420865.htm
qnl.aira2hc.cn/313735.htm
qnk.aira2hc.cn/222645.htm
qnk.aira2hc.cn/284485.htm
qnk.aira2hc.cn/844205.htm
qnk.aira2hc.cn/844665.htm
qnk.aira2hc.cn/777595.htm
qnk.aira2hc.cn/444685.htm
qnk.aira2hc.cn/733395.htm
qnk.aira2hc.cn/220605.htm
qnk.aira2hc.cn/844825.htm
qnk.aira2hc.cn/048425.htm
qnj.aira2hc.cn/440405.htm
qnj.aira2hc.cn/224665.htm
qnj.aira2hc.cn/793535.htm
qnj.aira2hc.cn/626085.htm
qnj.aira2hc.cn/991575.htm
qnj.aira2hc.cn/573135.htm
qnj.aira2hc.cn/688665.htm
qnj.aira2hc.cn/806865.htm
qnj.aira2hc.cn/973575.htm
qnj.aira2hc.cn/622245.htm
qnh.aira2hc.cn/197735.htm
qnh.aira2hc.cn/044085.htm
