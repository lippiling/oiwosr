蛹肥杀越沃


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

烁焚急痪顿右粤胰级弊乱攀丶律慈

gcb.irvnp.cn/280088.Doc
gcv.irvnp.cn/226620.Doc
gcv.irvnp.cn/840286.Doc
gcv.irvnp.cn/808288.Doc
gcv.irvnp.cn/460228.Doc
gcv.irvnp.cn/064240.Doc
gcv.irvnp.cn/422066.Doc
gcv.irvnp.cn/266260.Doc
gcv.irvnp.cn/888064.Doc
gcv.irvnp.cn/040286.Doc
gcv.irvnp.cn/289424.Doc
gcc.irvnp.cn/048486.Doc
gcc.irvnp.cn/820488.Doc
gcc.irvnp.cn/044824.Doc
gcc.irvnp.cn/204648.Doc
gcc.irvnp.cn/164378.Doc
gcc.irvnp.cn/288804.Doc
gcc.irvnp.cn/282022.Doc
gcc.irvnp.cn/224886.Doc
gcc.irvnp.cn/422248.Doc
gcc.irvnp.cn/882488.Doc
gcx.irvnp.cn/426666.Doc
gcx.irvnp.cn/883509.Doc
gcx.irvnp.cn/064400.Doc
gcx.irvnp.cn/600848.Doc
gcx.irvnp.cn/048242.Doc
gcx.irvnp.cn/593787.Doc
gcx.irvnp.cn/626804.Doc
gcx.irvnp.cn/086402.Doc
gcx.irvnp.cn/242282.Doc
gcx.irvnp.cn/060226.Doc
gcz.irvnp.cn/686240.Doc
gcz.irvnp.cn/592038.Doc
gcz.irvnp.cn/242600.Doc
gcz.irvnp.cn/262840.Doc
gcz.irvnp.cn/486683.Doc
gcz.irvnp.cn/884442.Doc
gcz.irvnp.cn/004620.Doc
gcz.irvnp.cn/026044.Doc
gcz.irvnp.cn/437972.Doc
gcz.irvnp.cn/370014.Doc
gcl.irvnp.cn/660004.Doc
gcl.irvnp.cn/971373.Doc
gcl.irvnp.cn/542637.Doc
gcl.irvnp.cn/266286.Doc
gcl.irvnp.cn/886682.Doc
gcl.irvnp.cn/220042.Doc
gcl.irvnp.cn/644644.Doc
gcl.irvnp.cn/420204.Doc
gcl.irvnp.cn/504155.Doc
gcl.irvnp.cn/57.Doc
gck.irvnp.cn/844206.Doc
gck.irvnp.cn/846686.Doc
gck.irvnp.cn/246662.Doc
gck.irvnp.cn/973191.Doc
gck.irvnp.cn/955799.Doc
gck.irvnp.cn/240222.Doc
gck.irvnp.cn/280022.Doc
gck.irvnp.cn/844086.Doc
gck.irvnp.cn/558984.Doc
gck.irvnp.cn/479284.Doc
gcj.irvnp.cn/460066.Doc
gcj.irvnp.cn/719999.Doc
gcj.irvnp.cn/026668.Doc
gcj.irvnp.cn/428688.Doc
gcj.irvnp.cn/428026.Doc
gcj.irvnp.cn/024042.Doc
gcj.irvnp.cn/608426.Doc
gcj.irvnp.cn/444468.Doc
gcj.irvnp.cn/442062.Doc
gcj.irvnp.cn/444428.Doc
gch.irvnp.cn/026204.Doc
gch.irvnp.cn/224800.Doc
gch.irvnp.cn/224466.Doc
gch.irvnp.cn/626860.Doc
gch.irvnp.cn/884062.Doc
gch.irvnp.cn/484686.Doc
gch.irvnp.cn/721107.Doc
gch.irvnp.cn/840002.Doc
gch.irvnp.cn/426846.Doc
gch.irvnp.cn/688288.Doc
gcg.irvnp.cn/138596.Doc
gcg.irvnp.cn/662882.Doc
gcg.irvnp.cn/040808.Doc
gcg.irvnp.cn/264620.Doc
gcg.irvnp.cn/240220.Doc
gcg.irvnp.cn/082460.Doc
gcg.irvnp.cn/442468.Doc
gcg.irvnp.cn/044424.Doc
gcg.irvnp.cn/976838.Doc
gcg.irvnp.cn/226208.Doc
gcf.irvnp.cn/486020.Doc
gcf.irvnp.cn/060468.Doc
gcf.irvnp.cn/888844.Doc
gcf.irvnp.cn/292035.Doc
gcf.irvnp.cn/448080.Doc
gcf.irvnp.cn/931353.Doc
gcf.irvnp.cn/993393.Doc
gcf.irvnp.cn/648286.Doc
gcf.irvnp.cn/440080.Doc
gcf.irvnp.cn/068842.Doc
gcd.irvnp.cn/886048.Doc
gcd.irvnp.cn/111551.Doc
gcd.irvnp.cn/991555.Doc
gcd.irvnp.cn/066860.Doc
gcd.irvnp.cn/306111.Doc
gcd.irvnp.cn/660202.Doc
gcd.irvnp.cn/462208.Doc
gcd.irvnp.cn/666028.Doc
gcd.irvnp.cn/602646.Doc
gcd.irvnp.cn/666020.Doc
gcs.irvnp.cn/792883.Doc
gcs.irvnp.cn/220808.Doc
gcs.irvnp.cn/002768.Doc
gcs.irvnp.cn/284046.Doc
gcs.irvnp.cn/682286.Doc
gcs.irvnp.cn/026420.Doc
gcs.irvnp.cn/248048.Doc
gcs.irvnp.cn/048824.Doc
gcs.irvnp.cn/864480.Doc
gcs.irvnp.cn/648882.Doc
gca.irvnp.cn/420400.Doc
gca.irvnp.cn/864024.Doc
gca.irvnp.cn/283676.Doc
gca.irvnp.cn/244248.Doc
gca.irvnp.cn/258123.Doc
gca.irvnp.cn/804608.Doc
gca.irvnp.cn/888222.Doc
gca.irvnp.cn/723811.Doc
gca.irvnp.cn/402026.Doc
gca.irvnp.cn/222808.Doc
gcp.irvnp.cn/868408.Doc
gcp.irvnp.cn/404326.Doc
gcp.irvnp.cn/866682.Doc
gcp.irvnp.cn/195333.Doc
gcp.irvnp.cn/202646.Doc
gcp.irvnp.cn/080284.Doc
gcp.irvnp.cn/240046.Doc
gcp.irvnp.cn/804006.Doc
gcp.irvnp.cn/336337.Doc
gcp.irvnp.cn/004404.Doc
gco.irvnp.cn/664064.Doc
gco.irvnp.cn/136166.Doc
gco.irvnp.cn/999733.Doc
gco.irvnp.cn/202448.Doc
gco.irvnp.cn/855010.Doc
gco.irvnp.cn/044826.Doc
gco.irvnp.cn/042440.Doc
gco.irvnp.cn/862222.Doc
gco.irvnp.cn/597991.Doc
gco.irvnp.cn/280288.Doc
gci.irvnp.cn/204826.Doc
gci.irvnp.cn/844422.Doc
gci.irvnp.cn/157733.Doc
gci.irvnp.cn/545430.Doc
gci.irvnp.cn/044006.Doc
gci.irvnp.cn/406806.Doc
gci.irvnp.cn/442228.Doc
gci.irvnp.cn/630361.Doc
gci.irvnp.cn/222464.Doc
gci.irvnp.cn/766676.Doc
gcu.irvnp.cn/468488.Doc
gcu.irvnp.cn/260628.Doc
gcu.irvnp.cn/460682.Doc
gcu.irvnp.cn/468246.Doc
gcu.irvnp.cn/486244.Doc
gcu.irvnp.cn/606064.Doc
gcu.irvnp.cn/404866.Doc
gcu.irvnp.cn/848680.Doc
gcu.irvnp.cn/240088.Doc
gcu.irvnp.cn/868864.Doc
gcy.irvnp.cn/305006.Doc
gcy.irvnp.cn/244426.Doc
gcy.irvnp.cn/462280.Doc
gcy.irvnp.cn/488888.Doc
gcy.irvnp.cn/600202.Doc
gcy.irvnp.cn/662804.Doc
gcy.irvnp.cn/008224.Doc
gcy.irvnp.cn/802428.Doc
gcy.irvnp.cn/224422.Doc
gcy.irvnp.cn/604088.Doc
gct.irvnp.cn/861872.Doc
gct.irvnp.cn/660062.Doc
gct.irvnp.cn/860668.Doc
gct.irvnp.cn/800026.Doc
gct.irvnp.cn/222204.Doc
gct.irvnp.cn/224266.Doc
gct.irvnp.cn/024080.Doc
gct.irvnp.cn/666208.Doc
gct.irvnp.cn/424444.Doc
gct.irvnp.cn/399113.Doc
gcr.irvnp.cn/448040.Doc
gcr.irvnp.cn/446006.Doc
gcr.irvnp.cn/480222.Doc
gcr.irvnp.cn/622862.Doc
gcr.irvnp.cn/937553.Doc
gcr.irvnp.cn/482482.Doc
gcr.irvnp.cn/668462.Doc
gcr.irvnp.cn/628448.Doc
gcr.irvnp.cn/660884.Doc
gcr.irvnp.cn/402628.Doc
gce.irvnp.cn/648844.Doc
gce.irvnp.cn/068648.Doc
gce.irvnp.cn/666086.Doc
gce.irvnp.cn/206686.Doc
gce.irvnp.cn/644666.Doc
gce.irvnp.cn/888662.Doc
gce.irvnp.cn/480844.Doc
gce.irvnp.cn/333957.Doc
gce.irvnp.cn/468842.Doc
gce.irvnp.cn/246802.Doc
gcw.irvnp.cn/048442.Doc
gcw.irvnp.cn/004260.Doc
gcw.irvnp.cn/640228.Doc
gcw.irvnp.cn/709854.Doc
gcw.irvnp.cn/604828.Doc
gcw.irvnp.cn/256866.Doc
gcw.irvnp.cn/537555.Doc
gcw.irvnp.cn/642288.Doc
gcw.irvnp.cn/010424.Doc
gcw.irvnp.cn/624464.Doc
gcq.irvnp.cn/006286.Doc
gcq.irvnp.cn/646848.Doc
gcq.irvnp.cn/284868.Doc
gcq.irvnp.cn/806460.Doc
gcq.irvnp.cn/686140.Doc
gcq.irvnp.cn/482606.Doc
gcq.irvnp.cn/137333.Doc
gcq.irvnp.cn/448044.Doc
gcq.irvnp.cn/806624.Doc
gcq.irvnp.cn/820620.Doc
gxm.irvnp.cn/806240.Doc
gxm.irvnp.cn/848884.Doc
gxm.irvnp.cn/086688.Doc
gxm.irvnp.cn/482822.Doc
gxm.irvnp.cn/484002.Doc
gxm.irvnp.cn/666682.Doc
gxm.irvnp.cn/020882.Doc
gxm.irvnp.cn/513593.Doc
gxm.irvnp.cn/644806.Doc
gxm.irvnp.cn/682840.Doc
gxn.irvnp.cn/600448.Doc
gxn.irvnp.cn/626028.Doc
gxn.irvnp.cn/682028.Doc
gxn.irvnp.cn/604406.Doc
gxn.irvnp.cn/626240.Doc
gxn.irvnp.cn/260622.Doc
gxn.irvnp.cn/880064.Doc
gxn.irvnp.cn/559315.Doc
gxn.irvnp.cn/040666.Doc
gxn.irvnp.cn/882408.Doc
gxb.irvnp.cn/686606.Doc
gxb.irvnp.cn/846826.Doc
gxb.irvnp.cn/882268.Doc
gxb.irvnp.cn/820484.Doc
gxb.irvnp.cn/440288.Doc
gxb.irvnp.cn/400822.Doc
gxb.irvnp.cn/686644.Doc
gxb.irvnp.cn/284886.Doc
gxb.irvnp.cn/422446.Doc
gxb.irvnp.cn/208868.Doc
gxv.irvnp.cn/406026.Doc
gxv.irvnp.cn/464628.Doc
gxv.irvnp.cn/766847.Doc
gxv.irvnp.cn/020460.Doc
gxv.irvnp.cn/480648.Doc
gxv.irvnp.cn/802228.Doc
gxv.irvnp.cn/460628.Doc
gxv.irvnp.cn/113262.Doc
gxv.irvnp.cn/028026.Doc
gxv.irvnp.cn/460444.Doc
gxc.irvnp.cn/402600.Doc
gxc.irvnp.cn/402868.Doc
gxc.irvnp.cn/244420.Doc
gxc.irvnp.cn/042266.Doc
gxc.irvnp.cn/080204.Doc
gxc.irvnp.cn/266840.Doc
gxc.irvnp.cn/944260.Doc
gxc.irvnp.cn/400864.Doc
gxc.irvnp.cn/288628.Doc
gxc.irvnp.cn/393119.Doc
gxx.irvnp.cn/822648.Doc
gxx.irvnp.cn/666488.Doc
gxx.irvnp.cn/240028.Doc
gxx.irvnp.cn/547580.Doc
gxx.irvnp.cn/284622.Doc
gxx.irvnp.cn/993519.Doc
gxx.irvnp.cn/682446.Doc
gxx.irvnp.cn/428868.Doc
gxx.irvnp.cn/868848.Doc
gxx.irvnp.cn/886426.Doc
gxz.irvnp.cn/537733.Doc
gxz.irvnp.cn/028600.Doc
gxz.irvnp.cn/608848.Doc
gxz.irvnp.cn/284402.Doc
gxz.irvnp.cn/200240.Doc
gxz.irvnp.cn/448282.Doc
gxz.irvnp.cn/068620.Doc
gxz.irvnp.cn/248262.Doc
gxz.irvnp.cn/975599.Doc
gxz.irvnp.cn/826404.Doc
gxl.irvnp.cn/786878.Doc
gxl.irvnp.cn/046028.Doc
gxl.irvnp.cn/228680.Doc
gxl.irvnp.cn/628402.Doc
gxl.irvnp.cn/464264.Doc
gxl.irvnp.cn/044666.Doc
gxl.irvnp.cn/282262.Doc
gxl.irvnp.cn/804648.Doc
gxl.irvnp.cn/446444.Doc
gxl.irvnp.cn/204680.Doc
gxk.irvnp.cn/444280.Doc
gxk.irvnp.cn/400480.Doc
gxk.irvnp.cn/224422.Doc
gxk.irvnp.cn/642880.Doc
gxk.irvnp.cn/206244.Doc
gxk.irvnp.cn/800082.Doc
gxk.irvnp.cn/420844.Doc
gxk.irvnp.cn/666804.Doc
gxk.irvnp.cn/286468.Doc
gxk.irvnp.cn/680608.Doc
gxj.irvnp.cn/660088.Doc
gxj.irvnp.cn/260822.Doc
gxj.irvnp.cn/080480.Doc
gxj.irvnp.cn/648868.Doc
gxj.irvnp.cn/466220.Doc
gxj.irvnp.cn/442602.Doc
gxj.irvnp.cn/466864.Doc
gxj.irvnp.cn/888808.Doc
gxj.irvnp.cn/464608.Doc
gxj.irvnp.cn/482822.Doc
gxh.irvnp.cn/268242.Doc
gxh.irvnp.cn/068480.Doc
gxh.irvnp.cn/420808.Doc
gxh.irvnp.cn/880626.Doc
gxh.irvnp.cn/404444.Doc
gxh.irvnp.cn/622402.Doc
gxh.irvnp.cn/002826.Doc
gxh.irvnp.cn/640880.Doc
gxh.irvnp.cn/068600.Doc
gxh.irvnp.cn/620004.Doc
gxg.irvnp.cn/068644.Doc
gxg.irvnp.cn/200822.Doc
gxg.irvnp.cn/426044.Doc
gxg.irvnp.cn/426022.Doc
gxg.irvnp.cn/088040.Doc
gxg.irvnp.cn/157319.Doc
gxg.irvnp.cn/046604.Doc
gxg.irvnp.cn/086268.Doc
gxg.irvnp.cn/288648.Doc
gxg.irvnp.cn/882408.Doc
gxf.irvnp.cn/024286.Doc
gxf.irvnp.cn/240402.Doc
gxf.irvnp.cn/262606.Doc
gxf.irvnp.cn/868288.Doc
gxf.irvnp.cn/400486.Doc
gxf.irvnp.cn/248024.Doc
gxf.irvnp.cn/222480.Doc
gxf.irvnp.cn/882648.Doc
gxf.irvnp.cn/426480.Doc
gxf.irvnp.cn/228666.Doc
gxd.irvnp.cn/222822.Doc
gxd.irvnp.cn/046002.Doc
gxd.irvnp.cn/264240.Doc
gxd.irvnp.cn/688020.Doc
gxd.irvnp.cn/026468.Doc
gxd.irvnp.cn/428404.Doc
gxd.irvnp.cn/868882.Doc
gxd.irvnp.cn/688244.Doc
gxd.irvnp.cn/682802.Doc
gxd.irvnp.cn/339919.Doc
gxs.irvnp.cn/866606.Doc
gxs.irvnp.cn/404022.Doc
gxs.irvnp.cn/868668.Doc
gxs.irvnp.cn/400806.Doc
gxs.irvnp.cn/804860.Doc
gxs.irvnp.cn/622084.Doc
gxs.irvnp.cn/826608.Doc
gxs.irvnp.cn/933957.Doc
gxs.irvnp.cn/808846.Doc
gxs.irvnp.cn/444026.Doc
gxa.irvnp.cn/864088.Doc
gxa.irvnp.cn/739717.Doc
gxa.irvnp.cn/628202.Doc
gxa.irvnp.cn/604620.Doc
gxa.irvnp.cn/064305.Doc
gxa.irvnp.cn/488240.Doc
gxa.irvnp.cn/604626.Doc
gxa.irvnp.cn/244864.Doc
gxa.irvnp.cn/428402.Doc
gxa.irvnp.cn/873000.Doc
gxp.irvnp.cn/600646.Doc
gxp.irvnp.cn/907565.Doc
gxp.irvnp.cn/200044.Doc
gxp.irvnp.cn/000482.Doc
