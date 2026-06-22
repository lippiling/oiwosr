杆刹允战壮


===== Python断路器模式实战 =====
——断路器防止对故障服务的重复调用，避免级联故障和资源耗尽

[1] 断路器模式概述
断路器有三种状态，类似电路断路器：
  · CLOSED(关闭)：正常状态，请求直达目标服务
  · OPEN(断开)：故障率超阈值，请求快速失败，不调用目标
  · HALF_OPEN(半开)：超时后允许少量探测请求，决定是否恢复

[2] pybreaker基础用法
pybreaker是Python中成熟的断路器库，支持灵活的配置和扩展。

    import pybreaker
    import time

    # 创建断路器实例：最多连续失败3次，30秒后尝试恢复
    db_breaker = pybreaker.CircuitBreaker(
        fail_max=3,          # 连续失败次数阈值
        reset_timeout=30,    # 断开后尝试恢复的等待秒数
    )

[3] 断路器装饰器模式
使用@circuit_breaker装饰器保护函数调用。

    import sqlite3

    class DatabaseClient:
        """模拟数据库客户端，用断路器保护"""

        def __init__(self, db_path: str):
            self.db_path = db_path

        @db_breaker  # 断路器装饰器：调用失败自动计数
        def query(self, sql: str) -> list:
            """执行数据库查询（被断路器保护）"""
            conn = sqlite3.connect(self.db_path)
            try:
                cursor = conn.cursor()
                cursor.execute(sql)
                return cursor.fetchall()
            finally:
                conn.close()

[4] 断路器监听器
通过监听器在状态变更时执行自定义操作（告警、日志等）。

    class LoggingListener:
        """断路器状态变化监听器：记录状态变更日志"""

        def on_open(self, breaker):
            """断路器打开时的回调：立即告警"""
            print(f"[告警] 断路器 {breaker} 已打开！开始快速失败")

        def on_close(self, breaker):
            """断路器关闭时的回调：服务恢复"""
            print(f"[恢复] 断路器 {breaker} 已关闭，服务恢复正常")

        def on_half_open(self, breaker):
            """断路器半开时的回调：准备探测"""
            print(f"[探测] 断路器 {breaker} 半开，发送探测请求")

    # 使用带监听器的断路器
    api_breaker = pybreaker.CircuitBreaker(
        fail_max=5,
        reset_timeout=60,
        listeners=[LoggingListener()],  # 注册监听器
    )

[5] 断路器保护外部API调用
最典型场景：保护HTTP API调用，防止雪崩效应。

    import httpx
    from typing import Optional, Dict, Any

    class PaymentGatewayClient:
        """支付网关客户端（带断路器保护）"""

        def __init__(self, base_url: str):
            self.base_url = base_url
            # 专用于支付API的断路器
            self.breaker = pybreaker.CircuitBreaker(
                fail_max=3,
                reset_timeout=30,
                listeners=[LoggingListener()],
            )

        def charge(self, amount: float, token: str) -> Dict[str, Any]:
            """调用支付网关扣款"""
            try:
                return self.breaker.call(self._do_charge, amount, token)
            except pybreaker.CircuitBreakerError:
                # 断路器打开时抛出此异常
                return {"status": "failed", "error": "服务暂不可用(熔断中)"}
            except httpx.RequestError as e:
                # HTTP请求失败
                return {"status": "failed", "error": str(e)}

        def _do_charge(self, amount: float, token: str) -> Dict[str, Any]:
            """实际的HTTP调用（由断路器包裹）"""
            with httpx.Client(base_url=self.base_url, timeout=5.0) as client:
                resp = client.post(
                    "/api/charge",
                    json={"amount": amount, "token": token},
                )
                resp.raise_for_status()
                return resp.json()

[6] 手动状态控制
某些场景需要手动设置断路器状态（如运维操作）。

    class ManualOverrideBreaker:
        """支持手动控制的断路器封装"""

        def __init__(self, fail_max: int = 3, reset_timeout: int = 30):
            self._breaker = pybreaker.CircuitBreaker(fail_max=fail_max,
                                                      reset_timeout=reset_timeout)
            self._manual_open = False  # 手动强制断开

        def force_open(self):
            """手动强制打开断路器（如检测到依赖服务维护中）"""
            self._manual_open = True
            print("[运维] 手动断开断路器")

        def force_close(self):
            """手动关闭断路器（恢复调用）"""
            self._manual_open = False
            print("[运维] 手动恢复断路器")

        def call(self, func, *args, **kwargs):
            """受保护的调用：手动优先于自动状态"""
            if self._manual_open:
                raise pybreaker.CircuitBreakerError("手动熔断中")
            return self._breaker.call(func, *args, **kwargs)

[7] 失败阈值与恢复策略
合理配置阈值和恢复时间至关重要：阈值太低导致误熔断，太高则保护失效。

    class TunedCircuitBreaker:
        """精细调优的断路器：按业务重要性分级配置"""

        def __init__(self):
            # 关键服务：快速熔断
            self.critical = pybreaker.CircuitBreaker(
                fail_max=2,        # 2次失败就熔断
                reset_timeout=60,  # 1分钟后尝试恢复
            )
            # 非关键服务：容忍更多失败
            self.non_critical = pybreaker.CircuitBreaker(
                fail_max=10,       # 10次失败才熔断
                reset_timeout=120, # 2分钟后尝试恢复
            )
            # 批量任务：长时间等待恢复
            self.batch = pybreaker.CircuitBreaker(
                fail_max=5,
                reset_timeout=300, # 5分钟后尝试恢复
            )

[8] 断路器与重试协同
断路器和重试配合使用：重试期间失败计入断路器计数。

    from tenacity import retry, stop_after_attempt, wait_exponential

    class ResilientServiceClient:
        """兼具重试和断路器保护的服务客户端"""

        def __init__(self):
            self.breaker = pybreaker.CircuitBreaker(fail_max=3, reset_timeout=30)

        @retry(stop=stop_after_attempt(3), wait=wait_exponential(min=1, max=5))
        def _do_call(self):
            """实际HTTP调用(含重试)"""
            with httpx.Client(timeout=3.0) as client:
                resp = client.get("http://internal-svc/api/data")
                resp.raise_for_status()
                return resp.json()

        def fetch_data(self) -> Optional[dict]:
            """断路器+重试的组合调用"""
            try:
                return self.breaker.call(self._do_call)
            except pybreaker.CircuitBreakerError:
                print("[断路器] 熔断中，跳过调用")
                return None
            except Exception as e:
                print(f"[错误] 重试耗尽后仍失败: {e}")
                return None

    if __name__ == "__main__":
        client = ResilientServiceClient()
        # 模拟连续失败触发断路器
        for i in range(5):
            result = client.fetch_data()
            print(f"请求 {i+1}: {'成功' if result else '失败'}")

街改控桃抗俚坟蓟何那卓逃安叵思

aex.eiyve.cn/664248.Doc
aex.eiyve.cn/644648.Doc
aex.eiyve.cn/282060.Doc
aex.eiyve.cn/044842.Doc
aex.eiyve.cn/004446.Doc
aex.eiyve.cn/884000.Doc
aez.eiyve.cn/026448.Doc
aez.eiyve.cn/599913.Doc
aez.eiyve.cn/460462.Doc
aez.eiyve.cn/066646.Doc
aez.eiyve.cn/008206.Doc
aez.eiyve.cn/686400.Doc
aez.eiyve.cn/482886.Doc
aez.eiyve.cn/260402.Doc
aez.eiyve.cn/008622.Doc
aez.eiyve.cn/640426.Doc
ael.eiyve.cn/802086.Doc
ael.eiyve.cn/482000.Doc
ael.eiyve.cn/599575.Doc
ael.eiyve.cn/260084.Doc
ael.eiyve.cn/428286.Doc
ael.eiyve.cn/688608.Doc
ael.eiyve.cn/606200.Doc
ael.eiyve.cn/204204.Doc
ael.eiyve.cn/662840.Doc
ael.eiyve.cn/008866.Doc
aek.eiyve.cn/826626.Doc
aek.eiyve.cn/886044.Doc
aek.eiyve.cn/739799.Doc
aek.eiyve.cn/600686.Doc
aek.eiyve.cn/242264.Doc
aek.eiyve.cn/422260.Doc
aek.eiyve.cn/222442.Doc
aek.eiyve.cn/268884.Doc
aek.eiyve.cn/226808.Doc
aek.eiyve.cn/082002.Doc
aej.eiyve.cn/466868.Doc
aej.eiyve.cn/284884.Doc
aej.eiyve.cn/486080.Doc
aej.eiyve.cn/286028.Doc
aej.eiyve.cn/268862.Doc
aej.eiyve.cn/884868.Doc
aej.eiyve.cn/084284.Doc
aej.eiyve.cn/555771.Doc
aej.eiyve.cn/008046.Doc
aej.eiyve.cn/820280.Doc
aeh.eiyve.cn/246222.Doc
aeh.eiyve.cn/000086.Doc
aeh.eiyve.cn/440886.Doc
aeh.eiyve.cn/551377.Doc
aeh.eiyve.cn/660686.Doc
aeh.eiyve.cn/800422.Doc
aeh.eiyve.cn/662880.Doc
aeh.eiyve.cn/864480.Doc
aeh.eiyve.cn/046880.Doc
aeh.eiyve.cn/448428.Doc
aeg.eiyve.cn/044486.Doc
aeg.eiyve.cn/664088.Doc
aeg.eiyve.cn/222480.Doc
aeg.eiyve.cn/246660.Doc
aeg.eiyve.cn/004080.Doc
aeg.eiyve.cn/682422.Doc
aeg.eiyve.cn/642822.Doc
aeg.eiyve.cn/719119.Doc
aeg.eiyve.cn/260262.Doc
aeg.eiyve.cn/822280.Doc
aef.eiyve.cn/682600.Doc
aef.eiyve.cn/620620.Doc
aef.eiyve.cn/482068.Doc
aef.eiyve.cn/660484.Doc
aef.eiyve.cn/266262.Doc
aef.eiyve.cn/208602.Doc
aef.eiyve.cn/646488.Doc
aef.eiyve.cn/402822.Doc
aef.eiyve.cn/248064.Doc
aef.eiyve.cn/004204.Doc
aed.eiyve.cn/022264.Doc
aed.eiyve.cn/664688.Doc
aed.eiyve.cn/420600.Doc
aed.eiyve.cn/426020.Doc
aed.eiyve.cn/117197.Doc
aed.eiyve.cn/995395.Doc
aed.eiyve.cn/844268.Doc
aed.eiyve.cn/026020.Doc
aed.eiyve.cn/822068.Doc
aed.eiyve.cn/008002.Doc
aes.eiyve.cn/086626.Doc
aes.eiyve.cn/688288.Doc
aes.eiyve.cn/644004.Doc
aes.eiyve.cn/131317.Doc
aes.eiyve.cn/626220.Doc
aes.eiyve.cn/860008.Doc
aes.eiyve.cn/864028.Doc
aes.eiyve.cn/131331.Doc
aes.eiyve.cn/280424.Doc
aes.eiyve.cn/688884.Doc
aea.eiyve.cn/802844.Doc
aea.eiyve.cn/397553.Doc
aea.eiyve.cn/080046.Doc
aea.eiyve.cn/686222.Doc
aea.eiyve.cn/860880.Doc
aea.eiyve.cn/006408.Doc
aea.eiyve.cn/464046.Doc
aea.eiyve.cn/204840.Doc
aea.eiyve.cn/084846.Doc
aea.eiyve.cn/048488.Doc
aep.eiyve.cn/408804.Doc
aep.eiyve.cn/288882.Doc
aep.eiyve.cn/059545.Doc
aep.eiyve.cn/080400.Doc
aep.eiyve.cn/686486.Doc
aep.eiyve.cn/426246.Doc
aep.eiyve.cn/420488.Doc
aep.eiyve.cn/668484.Doc
aep.eiyve.cn/242862.Doc
aep.eiyve.cn/351906.Doc
aeo.eiyve.cn/602444.Doc
aeo.eiyve.cn/084040.Doc
aeo.eiyve.cn/688004.Doc
aeo.eiyve.cn/264066.Doc
aeo.eiyve.cn/660082.Doc
aeo.eiyve.cn/406244.Doc
aeo.eiyve.cn/884228.Doc
aeo.eiyve.cn/886066.Doc
aeo.eiyve.cn/460224.Doc
aeo.eiyve.cn/248682.Doc
aei.eiyve.cn/448606.Doc
aei.eiyve.cn/600624.Doc
aei.eiyve.cn/755715.Doc
aei.eiyve.cn/206200.Doc
aei.eiyve.cn/680608.Doc
aei.eiyve.cn/284848.Doc
aei.eiyve.cn/286620.Doc
aei.eiyve.cn/282800.Doc
aei.eiyve.cn/224204.Doc
aei.eiyve.cn/428820.Doc
aeu.eiyve.cn/626288.Doc
aeu.eiyve.cn/462488.Doc
aeu.eiyve.cn/406082.Doc
aeu.eiyve.cn/846886.Doc
aeu.eiyve.cn/846646.Doc
aeu.eiyve.cn/664824.Doc
aeu.eiyve.cn/420608.Doc
aeu.eiyve.cn/042660.Doc
aeu.eiyve.cn/624068.Doc
aeu.eiyve.cn/840808.Doc
aey.eiyve.cn/486008.Doc
aey.eiyve.cn/402042.Doc
aey.eiyve.cn/288420.Doc
aey.eiyve.cn/864860.Doc
aey.eiyve.cn/682800.Doc
aey.eiyve.cn/228486.Doc
aey.eiyve.cn/608840.Doc
aey.eiyve.cn/020400.Doc
aey.eiyve.cn/482606.Doc
aey.eiyve.cn/426082.Doc
aet.eiyve.cn/442048.Doc
aet.eiyve.cn/335531.Doc
aet.eiyve.cn/648420.Doc
aet.eiyve.cn/200488.Doc
aet.eiyve.cn/264826.Doc
aet.eiyve.cn/842880.Doc
aet.eiyve.cn/888206.Doc
aet.eiyve.cn/884006.Doc
aet.eiyve.cn/222608.Doc
aet.eiyve.cn/840468.Doc
aer.eiyve.cn/884264.Doc
aer.eiyve.cn/864048.Doc
aer.eiyve.cn/464268.Doc
aer.eiyve.cn/220406.Doc
aer.eiyve.cn/207103.Doc
aer.eiyve.cn/060880.Doc
aer.eiyve.cn/882606.Doc
aer.eiyve.cn/220242.Doc
aer.eiyve.cn/680428.Doc
aer.eiyve.cn/460684.Doc
aee.eiyve.cn/266048.Doc
aee.eiyve.cn/779319.Doc
aee.eiyve.cn/864408.Doc
aee.eiyve.cn/062628.Doc
aee.eiyve.cn/060222.Doc
aee.eiyve.cn/026068.Doc
aee.eiyve.cn/400200.Doc
aee.eiyve.cn/240008.Doc
aee.eiyve.cn/020682.Doc
aee.eiyve.cn/044882.Doc
aew.eiyve.cn/224266.Doc
aew.eiyve.cn/919465.Doc
aew.eiyve.cn/646668.Doc
aew.eiyve.cn/389558.Doc
aew.eiyve.cn/593186.Doc
aew.eiyve.cn/862886.Doc
aew.eiyve.cn/022264.Doc
aew.eiyve.cn/664204.Doc
aew.eiyve.cn/668480.Doc
aew.eiyve.cn/266460.Doc
aeq.eiyve.cn/468626.Doc
aeq.eiyve.cn/604826.Doc
aeq.eiyve.cn/406086.Doc
aeq.eiyve.cn/024664.Doc
aeq.eiyve.cn/868860.Doc
aeq.eiyve.cn/779764.Doc
aeq.eiyve.cn/804882.Doc
aeq.eiyve.cn/060614.Doc
aeq.eiyve.cn/002220.Doc
aeq.eiyve.cn/660022.Doc
awm.eiyve.cn/842826.Doc
awm.eiyve.cn/264622.Doc
awm.eiyve.cn/008284.Doc
awm.eiyve.cn/446420.Doc
awm.eiyve.cn/408048.Doc
awm.eiyve.cn/268204.Doc
awm.eiyve.cn/444882.Doc
awm.eiyve.cn/491773.Doc
awm.eiyve.cn/968896.Doc
awm.eiyve.cn/015853.Doc
awn.eiyve.cn/462204.Doc
awn.eiyve.cn/265418.Doc
awn.eiyve.cn/268646.Doc
awn.eiyve.cn/445782.Doc
awn.eiyve.cn/004637.Doc
awn.eiyve.cn/426600.Doc
awn.eiyve.cn/626068.Doc
awn.eiyve.cn/600802.Doc
awn.eiyve.cn/828260.Doc
awn.eiyve.cn/600880.Doc
awb.eiyve.cn/400282.Doc
awb.eiyve.cn/086444.Doc
awb.eiyve.cn/602486.Doc
awb.eiyve.cn/460842.Doc
awb.eiyve.cn/682866.Doc
awb.eiyve.cn/020224.Doc
awb.eiyve.cn/400440.Doc
awb.eiyve.cn/628400.Doc
awb.eiyve.cn/200260.Doc
awb.eiyve.cn/066800.Doc
awv.eiyve.cn/622280.Doc
awv.eiyve.cn/191357.Doc
awv.eiyve.cn/408200.Doc
awv.eiyve.cn/200248.Doc
awv.eiyve.cn/864402.Doc
awv.eiyve.cn/824460.Doc
awv.eiyve.cn/286884.Doc
awv.eiyve.cn/022280.Doc
awv.eiyve.cn/868428.Doc
awv.eiyve.cn/262286.Doc
awc.eiyve.cn/422626.Doc
awc.eiyve.cn/206826.Doc
awc.eiyve.cn/840460.Doc
awc.eiyve.cn/844822.Doc
awc.eiyve.cn/244642.Doc
awc.eiyve.cn/462440.Doc
awc.eiyve.cn/886024.Doc
awc.eiyve.cn/888044.Doc
awc.eiyve.cn/066420.Doc
awc.eiyve.cn/084080.Doc
awx.eiyve.cn/222842.Doc
awx.eiyve.cn/622200.Doc
awx.eiyve.cn/068288.Doc
awx.eiyve.cn/006400.Doc
awx.eiyve.cn/888640.Doc
awx.eiyve.cn/060246.Doc
awx.eiyve.cn/308496.Doc
awx.eiyve.cn/042082.Doc
awx.eiyve.cn/642640.Doc
awx.eiyve.cn/380880.Doc
awz.eiyve.cn/510276.Doc
awz.eiyve.cn/626408.Doc
awz.eiyve.cn/447133.Doc
awz.eiyve.cn/246684.Doc
awz.eiyve.cn/644226.Doc
awz.eiyve.cn/896790.Doc
awz.eiyve.cn/640202.Doc
awz.eiyve.cn/642426.Doc
awz.eiyve.cn/208222.Doc
awz.eiyve.cn/002084.Doc
awl.eiyve.cn/248866.Doc
awl.eiyve.cn/428846.Doc
awl.eiyve.cn/882486.Doc
awl.eiyve.cn/884664.Doc
awl.eiyve.cn/246420.Doc
awl.eiyve.cn/660204.Doc
awl.eiyve.cn/804420.Doc
awl.eiyve.cn/531315.Doc
awl.eiyve.cn/244606.Doc
awl.eiyve.cn/202640.Doc
awk.eiyve.cn/644040.Doc
awk.eiyve.cn/048680.Doc
awk.eiyve.cn/865684.Doc
awk.eiyve.cn/266066.Doc
awk.eiyve.cn/062684.Doc
awk.eiyve.cn/785380.Doc
awk.eiyve.cn/288486.Doc
awk.eiyve.cn/864064.Doc
awk.eiyve.cn/044206.Doc
awk.eiyve.cn/082154.Doc
awj.eiyve.cn/088626.Doc
awj.eiyve.cn/396433.Doc
awj.eiyve.cn/066208.Doc
awj.eiyve.cn/884460.Doc
awj.eiyve.cn/008604.Doc
awj.eiyve.cn/866400.Doc
awj.eiyve.cn/060626.Doc
awj.eiyve.cn/264808.Doc
awj.eiyve.cn/822462.Doc
awj.eiyve.cn/268084.Doc
awh.eiyve.cn/664204.Doc
awh.eiyve.cn/846082.Doc
awh.eiyve.cn/604400.Doc
awh.eiyve.cn/066488.Doc
awh.eiyve.cn/222868.Doc
awh.eiyve.cn/626068.Doc
awh.eiyve.cn/222224.Doc
awh.eiyve.cn/248820.Doc
awh.eiyve.cn/915520.Doc
awh.eiyve.cn/836570.Doc
awg.eiyve.cn/882008.Doc
awg.eiyve.cn/557751.Doc
awg.eiyve.cn/846622.Doc
awg.eiyve.cn/242880.Doc
awg.eiyve.cn/806600.Doc
awg.eiyve.cn/046468.Doc
awg.eiyve.cn/266066.Doc
awg.eiyve.cn/040228.Doc
awg.eiyve.cn/288060.Doc
awg.eiyve.cn/088846.Doc
awf.eiyve.cn/064402.Doc
awf.eiyve.cn/888626.Doc
awf.eiyve.cn/444204.Doc
awf.eiyve.cn/820686.Doc
awf.eiyve.cn/600824.Doc
awf.eiyve.cn/448868.Doc
awf.eiyve.cn/446280.Doc
awf.eiyve.cn/222828.Doc
awf.eiyve.cn/642037.Doc
awf.eiyve.cn/259543.Doc
awd.eiyve.cn/822624.Doc
awd.eiyve.cn/460684.Doc
awd.eiyve.cn/067290.Doc
awd.eiyve.cn/351713.Doc
awd.eiyve.cn/028828.Doc
awd.eiyve.cn/684808.Doc
awd.eiyve.cn/246284.Doc
awd.eiyve.cn/484822.Doc
awd.eiyve.cn/600862.Doc
awd.eiyve.cn/842422.Doc
aws.eiyve.cn/135359.Doc
aws.eiyve.cn/248660.Doc
aws.eiyve.cn/066408.Doc
aws.eiyve.cn/442808.Doc
aws.eiyve.cn/220888.Doc
aws.eiyve.cn/600088.Doc
aws.eiyve.cn/888828.Doc
aws.eiyve.cn/066668.Doc
aws.eiyve.cn/840268.Doc
aws.eiyve.cn/468326.Doc
awa.eiyve.cn/926765.Doc
awa.eiyve.cn/111716.Doc
awa.eiyve.cn/192475.Doc
awa.eiyve.cn/866440.Doc
awa.eiyve.cn/488880.Doc
awa.eiyve.cn/248088.Doc
awa.eiyve.cn/428468.Doc
awa.eiyve.cn/288282.Doc
awa.eiyve.cn/680006.Doc
awa.eiyve.cn/808038.Doc
awp.eiyve.cn/189559.Doc
awp.eiyve.cn/571119.Doc
awp.eiyve.cn/174821.Doc
awp.eiyve.cn/660626.Doc
awp.eiyve.cn/640664.Doc
awp.eiyve.cn/620288.Doc
awp.eiyve.cn/440004.Doc
awp.eiyve.cn/828048.Doc
awp.eiyve.cn/606468.Doc
awp.eiyve.cn/185548.Doc
awo.eiyve.cn/282488.Doc
awo.eiyve.cn/040648.Doc
awo.eiyve.cn/668480.Doc
awo.eiyve.cn/876365.Doc
awo.eiyve.cn/066264.Doc
awo.eiyve.cn/822080.Doc
awo.eiyve.cn/884242.Doc
awo.eiyve.cn/066622.Doc
awo.eiyve.cn/240444.Doc
awo.eiyve.cn/488248.Doc
awi.eiyve.cn/400686.Doc
awi.eiyve.cn/288848.Doc
awi.eiyve.cn/046868.Doc
awi.eiyve.cn/339159.Doc
awi.eiyve.cn/240064.Doc
awi.eiyve.cn/088644.Doc
awi.eiyve.cn/482664.Doc
awi.eiyve.cn/886686.Doc
awi.eiyve.cn/002044.Doc
