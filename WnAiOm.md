燃迸苏伦按


===== Python微服务通信模式实战 =====
——微服务间通信是分布式系统的核心挑战，涉及同步、异步和服务发现等模式

[1] 同步通信：REST API (httpx)
同步通信简单直观，适合低延迟、实时性要求高的场景。httpx库提供了现代化的HTTP客户端。

    import httpx
    from typing import Dict, Any, Optional

    class OrderServiceClient:
        """通过REST API同步调用订单服务"""

        def __init__(self, base_url: str):
            self.client = httpx.Client(base_url=base_url, timeout=10.0)

        def create_order(self, order_data: Dict[str, Any]) -> Dict[str, Any]:
            """创建订单(POST请求)，返回服务端响应"""
            response = self.client.post("/api/orders", json=order_data)
            response.raise_for_status()  # 非2xx状态码抛出异常
            return response.json()

        def get_order(self, order_id: str) -> Optional[Dict[str, Any]]:
            """查询订单(GET请求)，不存在时返回None"""
            response = self.client.get(f"/api/orders/{order_id}")
            if response.status_code == 404:
                return None
            response.raise_for_status()
            return response.json()

        def close(self):
            """关闭HTTP客户端，释放连接池资源"""
            self.client.close()

[2] 异步通信：消息队列 (Pika + RabbitMQ)
异步通信通过消息代理解耦服务，适合处理突发流量和长耗时任务。

    import pika
    import json
    from typing import Callable

    class MessageBroker:
        """基于RabbitMQ的异步消息代理封装"""

        def __init__(self, host: str = "localhost"):
            self.connection = pika.BlockingConnection(
                pika.ConnectionParameters(host=host)
            )
            self.channel = self.connection.channel()

        def declare_queue(self, queue_name: str, durable: bool = True):
            """声明队列。durable=True保证RabbitMQ重启后队列不丢失"""
            self.channel.queue_declare(queue=queue_name, durable=durable)

        def publish(self, queue: str, message: dict):
            """发布消息到指定队列"""
            self.channel.basic_publish(
                exchange="",
                routing_key=queue,
                body=json.dumps(message).encode("utf-8"),
                properties=pika.BasicProperties(delivery_mode=2),  # 持久化消息
            )

        def consume(self, queue: str, callback: Callable):
            """注册消费者。callback接收(ch, method, properties, body)"""
            self.channel.basic_consume(queue=queue, on_message_callback=callback)
            self.channel.start_consuming()  # 阻塞等待消息

        def close(self):
            """关闭连接"""
            self.connection.close()

[3] gRPC 通信
gRPC基于Protocol Buffers，性能高、支持双向流，适合微服务间内部通信。

    # gRPC需要先定义proto文件，然后生成Python代码
    # 假设已生成 order_pb2 和 order_pb2_grpc 模块
    import grpc
    from concurrent import futures

    class OrderGrpcClient:
        """gRPC客户端：通过一元RPC调用服务端方法"""

        def __init__(self, target: str = "localhost:50051"):
            self.channel = grpc.insecure_channel(target)
            # self.stub = order_pb2_grpc.OrderServiceStub(self.channel)

        def get_order(self, order_id: str):
            """调用gRPC远程方法获取订单详情"""
            # request = order_pb2.GetOrderRequest(order_id=order_id)
            # response = self.stub.GetOrder(request)
            # return response
            pass  # 实际使用时取消注释

[4] 服务发现模式 (Consul HTTP API)
服务发现让服务动态找到彼此，无需硬编码地址。

    import requests

    class ServiceDiscovery:
        """通过Consul HTTP API实现服务注册与发现"""

        def __init__(self, consul_host: str = "localhost", consul_port: int = 8500):
            self.base_url = f"http://{consul_host}:{consul_port}"

        def register(self, service_name: str, port: int, host: str = "127.0.0.1"):
            """向Consul注册服务实例"""
            payload = {
                "Name": service_name,
                "Address": host,
                "Port": port,
                "Check": {  # 健康检查配置
                    "HTTP": f"http://{host}:{port}/health",
                    "Interval": "10s",  # 每10秒检查一次
                    "Timeout": "2s",
                },
            }
            response = requests.put(
                f"{self.base_url}/v1/agent/service/register", json=payload
            )
            return response.status_code == 200

        def discover(self, service_name: str) -> list:
            """发现指定服务的所有健康实例"""
            response = requests.get(
                f"{self.base_url}/v1/health/service/{service_name}?passing=true"
            )
            services = response.json()
            return [
                {
                    "host": svc["Service"]["Address"],
                    "port": svc["Service"]["Port"],
                }
                for svc in services
            ]

[5] 健康检查端点
每个微服务应暴露健康检查端点，供负载均衡和服务发现组件使用。

    from fastapi import FastAPI
    from fastapi.responses import JSONResponse

    app = FastAPI()

    @app.get("/health")
    async def health_check():
        """健康检查端点：返回服务运行状态"""
        return JSONResponse(
            content={
                "status": "healthy",
                "service": "order-service",
                "version": "1.0.0",
            }
        )

[6] 断路器模式 (pybreaker)
断路器防止级联故障——当被调用服务不可用时快速失败。

    import pybreaker
    import httpx

    # 创建断路器：连续失败3次后断开，30秒后尝试恢复
    order_breaker = pybreaker.CircuitBreaker(
        fail_max=3,       # 最大失败次数
        reset_timeout=30, # 重置超时（秒）
    )

    class OrderServiceWithBreaker:
        """带断路器的订单服务客户端"""

        @order_breaker
        def call_create_order(self, order_data: dict) -> dict:
            """断路器保护的外部调用，熔断时抛出异常"""
            with httpx.Client() as client:
                resp = client.post("http://order-svc/api/orders", json=order_data)
                resp.raise_for_status()
                return resp.json()

[7] 重试与指数退避 (tenacity)
网络故障是分布式系统中的常态，合理重试可大幅提升稳定性。

    from tenacity import (
        retry,
        stop_after_attempt,
        wait_exponential,
        retry_if_exception_type,
    )
    import httpx

    @retry(
        stop=stop_after_attempt(3),                         # 最多重试3次
        wait=wait_exponential(multiplier=1, min=1, max=10), # 等待1s→2s→4s
        retry=retry_if_exception_type(httpx.RequestError),  # 仅对网络错误重试
    )
    def fetch_user(user_id: int) -> dict:
        """带重试机制的用户查询"""
        with httpx.Client() as client:
            resp = client.get(f"http://user-svc/api/users/{user_id}")
            resp.raise_for_status()
            return resp.json()

[8] 综合示例：服务间的可靠调用

    if __name__ == "__main__":
        # 服务发现
        discovery = ServiceDiscovery()
        services = discovery.discover("order-service")
        print(f"发现 {len(services)} 个订单服务实例")

        # 同步调用（带重试和断路器）
        try:
            order = OrderServiceWithBreaker().call_create_order(
                {"user_id": 42, "items": [{"sku": "ABC", "qty": 2}]}
            )
            print(f"订单创建成功: {order}")
        except pybreaker.CircuitBreakerError:
            print("断路器打开：服务暂不可用，请稍后重试")
        except Exception as e:
            print(f"通信失败: {e}")

掷空滥缸链琳诮炭乒忍圃对勺装屠

gnn.nfsid.cn/686204.Doc
gnb.nfsid.cn/204646.Doc
gnb.nfsid.cn/860040.Doc
gnb.nfsid.cn/842842.Doc
gnb.nfsid.cn/246822.Doc
gnb.nfsid.cn/608480.Doc
gnb.nfsid.cn/630132.Doc
gnb.nfsid.cn/173955.Doc
gnb.nfsid.cn/024044.Doc
gnb.nfsid.cn/371319.Doc
gnb.nfsid.cn/480062.Doc
gnv.nfsid.cn/088824.Doc
gnv.nfsid.cn/664422.Doc
gnv.nfsid.cn/208226.Doc
gnv.nfsid.cn/064488.Doc
gnv.nfsid.cn/044688.Doc
gnv.nfsid.cn/486086.Doc
gnv.nfsid.cn/042626.Doc
gnv.nfsid.cn/882680.Doc
gnv.nfsid.cn/932062.Doc
gnv.nfsid.cn/024422.Doc
gnc.nfsid.cn/862408.Doc
gnc.nfsid.cn/422826.Doc
gnc.nfsid.cn/288828.Doc
gnc.nfsid.cn/446602.Doc
gnc.nfsid.cn/464826.Doc
gnc.nfsid.cn/866288.Doc
gnc.nfsid.cn/480248.Doc
gnc.nfsid.cn/686644.Doc
gnc.nfsid.cn/428866.Doc
gnc.nfsid.cn/884284.Doc
gnx.nfsid.cn/240206.Doc
gnx.nfsid.cn/000400.Doc
gnx.nfsid.cn/064088.Doc
gnx.nfsid.cn/884024.Doc
gnx.nfsid.cn/246064.Doc
gnx.nfsid.cn/886406.Doc
gnx.nfsid.cn/202022.Doc
gnx.nfsid.cn/417413.Doc
gnx.nfsid.cn/646064.Doc
gnx.nfsid.cn/044640.Doc
gnz.nfsid.cn/042882.Doc
gnz.nfsid.cn/311939.Doc
gnz.nfsid.cn/359619.Doc
gnz.nfsid.cn/224220.Doc
gnz.nfsid.cn/082262.Doc
gnz.nfsid.cn/042060.Doc
gnz.nfsid.cn/824444.Doc
gnz.nfsid.cn/262806.Doc
gnz.nfsid.cn/408266.Doc
gnz.nfsid.cn/444642.Doc
gnl.nfsid.cn/420686.Doc
gnl.nfsid.cn/420464.Doc
gnl.nfsid.cn/202260.Doc
gnl.nfsid.cn/064808.Doc
gnl.nfsid.cn/064244.Doc
gnl.nfsid.cn/288648.Doc
gnl.nfsid.cn/027159.Doc
gnl.nfsid.cn/826244.Doc
gnl.nfsid.cn/577319.Doc
gnl.nfsid.cn/262844.Doc
gnk.nfsid.cn/448204.Doc
gnk.nfsid.cn/602802.Doc
gnk.nfsid.cn/266288.Doc
gnk.nfsid.cn/848620.Doc
gnk.nfsid.cn/840084.Doc
gnk.nfsid.cn/084646.Doc
gnk.nfsid.cn/008440.Doc
gnk.nfsid.cn/886022.Doc
gnk.nfsid.cn/868042.Doc
gnk.nfsid.cn/066646.Doc
gnj.nfsid.cn/628644.Doc
gnj.nfsid.cn/468888.Doc
gnj.nfsid.cn/515137.Doc
gnj.nfsid.cn/444125.Doc
gnj.nfsid.cn/220088.Doc
gnj.nfsid.cn/266026.Doc
gnj.nfsid.cn/606428.Doc
gnj.nfsid.cn/626828.Doc
gnj.nfsid.cn/028208.Doc
gnj.nfsid.cn/482208.Doc
gnh.nfsid.cn/640446.Doc
gnh.nfsid.cn/393513.Doc
gnh.nfsid.cn/204006.Doc
gnh.nfsid.cn/822826.Doc
gnh.nfsid.cn/240246.Doc
gnh.nfsid.cn/795391.Doc
gnh.nfsid.cn/024602.Doc
gnh.nfsid.cn/688444.Doc
gnh.nfsid.cn/060204.Doc
gnh.nfsid.cn/075502.Doc
gng.nfsid.cn/828228.Doc
gng.nfsid.cn/840660.Doc
gng.nfsid.cn/608222.Doc
gng.nfsid.cn/868026.Doc
gng.nfsid.cn/022620.Doc
gng.nfsid.cn/842428.Doc
gng.nfsid.cn/822442.Doc
gng.nfsid.cn/404046.Doc
gng.nfsid.cn/622264.Doc
gng.nfsid.cn/066860.Doc
gnf.nfsid.cn/084869.Doc
gnf.nfsid.cn/888408.Doc
gnf.nfsid.cn/264088.Doc
gnf.nfsid.cn/462068.Doc
gnf.nfsid.cn/228606.Doc
gnf.nfsid.cn/068460.Doc
gnf.nfsid.cn/020600.Doc
gnf.nfsid.cn/606680.Doc
gnf.nfsid.cn/048000.Doc
gnf.nfsid.cn/412426.Doc
gnd.nfsid.cn/802202.Doc
gnd.nfsid.cn/044266.Doc
gnd.nfsid.cn/642260.Doc
gnd.nfsid.cn/400080.Doc
gnd.nfsid.cn/462844.Doc
gnd.nfsid.cn/002086.Doc
gnd.nfsid.cn/604620.Doc
gnd.nfsid.cn/822664.Doc
gnd.nfsid.cn/040844.Doc
gnd.nfsid.cn/872779.Doc
gns.nfsid.cn/499295.Doc
gns.nfsid.cn/482280.Doc
gns.nfsid.cn/116458.Doc
gns.nfsid.cn/220282.Doc
gns.nfsid.cn/220604.Doc
gns.nfsid.cn/264664.Doc
gns.nfsid.cn/860484.Doc
gns.nfsid.cn/828062.Doc
gns.nfsid.cn/288644.Doc
gns.nfsid.cn/196486.Doc
gna.nfsid.cn/875985.Doc
gna.nfsid.cn/608648.Doc
gna.nfsid.cn/446840.Doc
gna.nfsid.cn/608442.Doc
gna.nfsid.cn/484600.Doc
gna.nfsid.cn/022422.Doc
gna.nfsid.cn/460684.Doc
gna.nfsid.cn/804860.Doc
gna.nfsid.cn/725696.Doc
gna.nfsid.cn/202022.Doc
gnp.nfsid.cn/008468.Doc
gnp.nfsid.cn/042068.Doc
gnp.nfsid.cn/222642.Doc
gnp.nfsid.cn/068748.Doc
gnp.nfsid.cn/404864.Doc
gnp.nfsid.cn/086682.Doc
gnp.nfsid.cn/060848.Doc
gnp.nfsid.cn/648402.Doc
gnp.nfsid.cn/002882.Doc
gnp.nfsid.cn/133959.Doc
gno.nfsid.cn/086288.Doc
gno.nfsid.cn/842260.Doc
gno.nfsid.cn/060846.Doc
gno.nfsid.cn/848480.Doc
gno.nfsid.cn/682240.Doc
gno.nfsid.cn/355175.Doc
gno.nfsid.cn/347806.Doc
gno.nfsid.cn/008844.Doc
gno.nfsid.cn/260660.Doc
gno.nfsid.cn/264802.Doc
gni.nfsid.cn/806008.Doc
gni.nfsid.cn/628240.Doc
gni.nfsid.cn/095406.Doc
gni.nfsid.cn/497891.Doc
gni.nfsid.cn/600884.Doc
gni.nfsid.cn/286422.Doc
gni.nfsid.cn/332702.Doc
gni.nfsid.cn/860662.Doc
gni.nfsid.cn/684426.Doc
gni.nfsid.cn/602464.Doc
gnu.nfsid.cn/640266.Doc
gnu.nfsid.cn/591391.Doc
gnu.nfsid.cn/206822.Doc
gnu.nfsid.cn/695290.Doc
gnu.nfsid.cn/088666.Doc
gnu.nfsid.cn/715937.Doc
gnu.nfsid.cn/571043.Doc
gnu.nfsid.cn/028424.Doc
gnu.nfsid.cn/042208.Doc
gnu.nfsid.cn/004460.Doc
gny.nfsid.cn/462682.Doc
gny.nfsid.cn/684202.Doc
gny.nfsid.cn/086084.Doc
gny.nfsid.cn/282680.Doc
gny.nfsid.cn/644606.Doc
gny.nfsid.cn/420822.Doc
gny.nfsid.cn/469574.Doc
gny.nfsid.cn/806222.Doc
gny.nfsid.cn/242428.Doc
gny.nfsid.cn/042648.Doc
gnt.nfsid.cn/022228.Doc
gnt.nfsid.cn/446626.Doc
gnt.nfsid.cn/080226.Doc
gnt.nfsid.cn/224864.Doc
gnt.nfsid.cn/317539.Doc
gnt.nfsid.cn/248466.Doc
gnt.nfsid.cn/602628.Doc
gnt.nfsid.cn/088422.Doc
gnt.nfsid.cn/113957.Doc
gnt.nfsid.cn/266860.Doc
gnr.nfsid.cn/042408.Doc
gnr.nfsid.cn/009482.Doc
gnr.nfsid.cn/246046.Doc
gnr.nfsid.cn/844204.Doc
gnr.nfsid.cn/745665.Doc
gnr.nfsid.cn/400642.Doc
gnr.nfsid.cn/682822.Doc
gnr.nfsid.cn/222484.Doc
gnr.nfsid.cn/604400.Doc
gnr.nfsid.cn/664220.Doc
gne.nfsid.cn/406684.Doc
gne.nfsid.cn/488662.Doc
gne.nfsid.cn/202286.Doc
gne.nfsid.cn/062600.Doc
gne.nfsid.cn/937171.Doc
gne.nfsid.cn/426840.Doc
gne.nfsid.cn/688400.Doc
gne.nfsid.cn/266624.Doc
gne.nfsid.cn/950918.Doc
gne.nfsid.cn/622402.Doc
gnw.nfsid.cn/685939.Doc
gnw.nfsid.cn/958278.Doc
gnw.nfsid.cn/420642.Doc
gnw.nfsid.cn/959355.Doc
gnw.nfsid.cn/620428.Doc
gnw.nfsid.cn/682286.Doc
gnw.nfsid.cn/206644.Doc
gnw.nfsid.cn/914395.Doc
gnw.nfsid.cn/026864.Doc
gnw.nfsid.cn/628664.Doc
gnq.nfsid.cn/428046.Doc
gnq.nfsid.cn/464268.Doc
gnq.nfsid.cn/506565.Doc
gnq.nfsid.cn/620000.Doc
gnq.nfsid.cn/131353.Doc
gnq.nfsid.cn/222286.Doc
gnq.nfsid.cn/444468.Doc
gnq.nfsid.cn/206086.Doc
gnq.nfsid.cn/266688.Doc
gnq.nfsid.cn/664206.Doc
gbm.nfsid.cn/088240.Doc
gbm.nfsid.cn/957173.Doc
gbm.nfsid.cn/19.Doc
gbm.nfsid.cn/864246.Doc
gbm.nfsid.cn/668048.Doc
gbm.nfsid.cn/286842.Doc
gbm.nfsid.cn/264408.Doc
gbm.nfsid.cn/688000.Doc
gbm.nfsid.cn/202806.Doc
gbm.nfsid.cn/664844.Doc
gbn.nfsid.cn/684424.Doc
gbn.nfsid.cn/246200.Doc
gbn.nfsid.cn/284600.Doc
gbn.nfsid.cn/680242.Doc
gbn.nfsid.cn/335238.Doc
gbn.nfsid.cn/921562.Doc
gbn.nfsid.cn/084124.Doc
gbn.nfsid.cn/024880.Doc
gbn.nfsid.cn/808284.Doc
gbn.nfsid.cn/913500.Doc
gbb.nfsid.cn/828804.Doc
gbb.nfsid.cn/202260.Doc
gbb.nfsid.cn/951773.Doc
gbb.nfsid.cn/068400.Doc
gbb.nfsid.cn/152053.Doc
gbb.nfsid.cn/680404.Doc
gbb.nfsid.cn/084648.Doc
gbb.nfsid.cn/060448.Doc
gbb.nfsid.cn/220826.Doc
gbb.nfsid.cn/853307.Doc
gbv.nfsid.cn/115222.Doc
gbv.nfsid.cn/462828.Doc
gbv.nfsid.cn/399337.Doc
gbv.nfsid.cn/044400.Doc
gbv.nfsid.cn/664082.Doc
gbv.nfsid.cn/610004.Doc
gbv.nfsid.cn/751991.Doc
gbv.nfsid.cn/484802.Doc
gbv.nfsid.cn/862022.Doc
gbv.nfsid.cn/866066.Doc
gbc.nfsid.cn/048884.Doc
gbc.nfsid.cn/180139.Doc
gbc.nfsid.cn/606824.Doc
gbc.nfsid.cn/064646.Doc
gbc.nfsid.cn/040480.Doc
gbc.nfsid.cn/262268.Doc
gbc.nfsid.cn/220266.Doc
gbc.nfsid.cn/088220.Doc
gbc.nfsid.cn/806440.Doc
gbc.nfsid.cn/440822.Doc
gbx.nfsid.cn/083971.Doc
gbx.nfsid.cn/888642.Doc
gbx.nfsid.cn/866262.Doc
gbx.nfsid.cn/806084.Doc
gbx.nfsid.cn/784431.Doc
gbx.nfsid.cn/686842.Doc
gbx.nfsid.cn/094887.Doc
gbx.nfsid.cn/448400.Doc
gbx.nfsid.cn/204046.Doc
gbx.nfsid.cn/848422.Doc
gbz.nfsid.cn/553719.Doc
gbz.nfsid.cn/404242.Doc
gbz.nfsid.cn/848040.Doc
gbz.nfsid.cn/824707.Doc
gbz.nfsid.cn/848820.Doc
gbz.nfsid.cn/022246.Doc
gbz.nfsid.cn/222642.Doc
gbz.nfsid.cn/943041.Doc
gbz.nfsid.cn/862202.Doc
gbz.nfsid.cn/082202.Doc
gbl.nfsid.cn/000000.Doc
gbl.nfsid.cn/626648.Doc
gbl.nfsid.cn/628204.Doc
gbl.nfsid.cn/333397.Doc
gbl.nfsid.cn/666228.Doc
gbl.nfsid.cn/600802.Doc
gbl.nfsid.cn/608280.Doc
gbl.nfsid.cn/000468.Doc
gbl.nfsid.cn/044888.Doc
gbl.nfsid.cn/208662.Doc
gbk.nfsid.cn/519551.Doc
gbk.nfsid.cn/042028.Doc
gbk.nfsid.cn/022208.Doc
gbk.nfsid.cn/600440.Doc
gbk.nfsid.cn/204682.Doc
gbk.nfsid.cn/882802.Doc
gbk.nfsid.cn/289579.Doc
gbk.nfsid.cn/288626.Doc
gbk.nfsid.cn/844442.Doc
gbk.nfsid.cn/220608.Doc
gbj.nfsid.cn/422862.Doc
gbj.nfsid.cn/800408.Doc
gbj.nfsid.cn/321295.Doc
gbj.nfsid.cn/826220.Doc
gbj.nfsid.cn/442662.Doc
gbj.nfsid.cn/068264.Doc
gbj.nfsid.cn/084482.Doc
gbj.nfsid.cn/288628.Doc
gbj.nfsid.cn/280086.Doc
gbj.nfsid.cn/044824.Doc
gbh.nfsid.cn/425403.Doc
gbh.nfsid.cn/648004.Doc
gbh.nfsid.cn/668480.Doc
gbh.nfsid.cn/775151.Doc
gbh.nfsid.cn/804820.Doc
gbh.nfsid.cn/682486.Doc
gbh.nfsid.cn/060044.Doc
gbh.nfsid.cn/220866.Doc
gbh.nfsid.cn/682004.Doc
gbh.nfsid.cn/268088.Doc
gbg.nfsid.cn/397577.Doc
gbg.nfsid.cn/408482.Doc
gbg.nfsid.cn/313953.Doc
gbg.nfsid.cn/008280.Doc
gbg.nfsid.cn/628624.Doc
gbg.nfsid.cn/224008.Doc
gbg.nfsid.cn/428442.Doc
gbg.nfsid.cn/460424.Doc
gbg.nfsid.cn/668442.Doc
gbg.nfsid.cn/268668.Doc
gbf.nfsid.cn/317575.Doc
gbf.nfsid.cn/193975.Doc
gbf.nfsid.cn/200468.Doc
gbf.nfsid.cn/943285.Doc
gbf.nfsid.cn/068440.Doc
gbf.nfsid.cn/608464.Doc
gbf.nfsid.cn/870657.Doc
gbf.nfsid.cn/266428.Doc
gbf.nfsid.cn/488844.Doc
gbf.nfsid.cn/080222.Doc
gbd.nfsid.cn/091850.Doc
gbd.nfsid.cn/828248.Doc
gbd.nfsid.cn/802624.Doc
gbd.nfsid.cn/682846.Doc
gbd.nfsid.cn/246080.Doc
gbd.nfsid.cn/068866.Doc
gbd.nfsid.cn/242068.Doc
gbd.nfsid.cn/866802.Doc
gbd.nfsid.cn/222444.Doc
gbd.nfsid.cn/995531.Doc
gbs.nfsid.cn/664822.Doc
gbs.nfsid.cn/026444.Doc
gbs.nfsid.cn/820026.Doc
gbs.nfsid.cn/820206.Doc
gbs.nfsid.cn/379359.Doc
gbs.nfsid.cn/137775.Doc
gbs.nfsid.cn/684642.Doc
gbs.nfsid.cn/484202.Doc
gbs.nfsid.cn/080424.Doc
gbs.nfsid.cn/084444.Doc
gba.nfsid.cn/440804.Doc
gba.nfsid.cn/995917.Doc
gba.nfsid.cn/248640.Doc
gba.nfsid.cn/624666.Doc
