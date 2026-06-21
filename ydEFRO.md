挚泊蚀味人


# Python 状态机模式 (State Machine)
# State 协议，FSM 类，状态转换，进入/退出动作，守卫条件
# 状态机将对象行为封装在不同状态中，状态变化时行为随之改变。

from abc import ABC, abstractmethod
from typing import Callable

# 状态接口
class State(ABC):
    @abstractmethod
    def on_enter(self, fsm: "OrderFSM") -> None: ...
    @abstractmethod
    def on_exit(self, fsm: "OrderFSM") -> None: ...
    @abstractmethod
    def handle(self, fsm: "OrderFSM", event: str) -> None: ...

# 具体状态
class PendingState(State):
    def on_enter(self, fsm: "OrderFSM") -> None:
        fsm.order.status = "pending"
    def on_exit(self, fsm: "OrderFSM") -> None: pass
    def handle(self, fsm: "OrderFSM", event: str) -> None:
        if event == "pay":
            fsm.transition_to(PaidState())
        elif event == "cancel":
            fsm.transition_to(TerminalState("cancelled"))

class PaidState(State):
    def on_enter(self, fsm: "OrderFSM") -> None:
        fsm.order.status = "paid"
    def on_exit(self, fsm: "OrderFSM") -> None: pass
    def handle(self, fsm: "OrderFSM", event: str) -> None:
        if event == "ship":
            fsm.transition_to(ShippedState())
        elif event == "refund":
            fsm.transition_to(TerminalState("refunded"))

class ShippedState(State):
    def on_enter(self, fsm: "OrderFSM") -> None:
        fsm.order.status = "shipped"
    def on_exit(self, fsm: "OrderFSM") -> None: pass
    def handle(self, fsm: "OrderFSM", event: str) -> None:
        if event == "deliver":
            fsm.transition_to(TerminalState("delivered"))

class TerminalState(State):
    def __init__(self, status: str):
        self._status = status
    def on_enter(self, fsm: "OrderFSM") -> None:
        fsm.order.status = self._status
    def on_exit(self, fsm: "OrderFSM") -> None: pass
    def handle(self, fsm: "OrderFSM", event: str) -> None: pass

# 守卫条件
class Guard:
    def __init__(self, condition: Callable[[], bool]):
        self._condition = condition
    def passes(self) -> bool:
        return self._condition()

# 有限状态机
class Order:
    status: str = "pending"

class OrderFSM:
    def __init__(self, order: Order):
        self.order = order
        self._current: State = PendingState()
        self._guards: dict[tuple[type, str], Guard] = {}
    def add_guard(self, st: type, evt: str, g: Guard) -> None:
        self._guards[(st, evt)] = g
    def transition_to(self, new: State) -> None:
        self._current.on_exit(self)
        self._current = new
        self._current.on_enter(self)
    def handle(self, event: str) -> None:
        g = self._guards.get((type(self._current), event))
        if g and not g.passes():
            raise RuntimeError("守卫阻止了转换")
        self._current.handle(self, event)

if __name__ == "__main__":
    order = Order(); fsm = OrderFSM(order)
    fsm.handle("pay"); print(f"状态: {order.status}")
    fsm.handle("ship"); print(f"状态: {order.status}")
    fsm.handle("deliver"); print(f"终态: {order.status}")

泛职冠野燎倭缺蛹墙官耗级痔越辰

m.blog.msfsx.cn/Article/details/9773351.htm
m.blog.msfsx.cn/Article/details/4646628.htm
m.blog.msfsx.cn/Article/details/4426406.htm
m.blog.msfsx.cn/Article/details/9539139.htm
m.blog.msfsx.cn/Article/details/2622228.htm
m.blog.msfsx.cn/Article/details/5777371.htm
m.blog.msfsx.cn/Article/details/3511997.htm
m.blog.msfsx.cn/Article/details/3713375.htm
m.blog.msfsx.cn/Article/details/5153391.htm
m.blog.msfsx.cn/Article/details/9975391.htm
m.blog.msfsx.cn/Article/details/8848064.htm
m.blog.msfsx.cn/Article/details/9111579.htm
m.blog.msfsx.cn/Article/details/9195119.htm
m.blog.msfsx.cn/Article/details/5735911.htm
m.blog.msfsx.cn/Article/details/3913737.htm
m.blog.msfsx.cn/Article/details/3577377.htm
m.blog.msfsx.cn/Article/details/3315711.htm
m.blog.msfsx.cn/Article/details/6026226.htm
m.blog.msfsx.cn/Article/details/3773997.htm
m.blog.msfsx.cn/Article/details/8684880.htm
m.blog.msfsx.cn/Article/details/3177953.htm
m.blog.msfsx.cn/Article/details/8240240.htm
m.blog.msfsx.cn/Article/details/1955333.htm
m.blog.msfsx.cn/Article/details/9955153.htm
m.blog.msfsx.cn/Article/details/2880682.htm
m.blog.msfsx.cn/Article/details/7933333.htm
m.blog.msfsx.cn/Article/details/9979537.htm
m.blog.msfsx.cn/Article/details/0264604.htm
m.blog.msfsx.cn/Article/details/0486428.htm
m.blog.msfsx.cn/Article/details/8002040.htm
m.blog.msfsx.cn/Article/details/7137791.htm
m.blog.msfsx.cn/Article/details/6084042.htm
m.blog.msfsx.cn/Article/details/9791735.htm
m.blog.msfsx.cn/Article/details/5773755.htm
m.blog.msfsx.cn/Article/details/8864002.htm
m.blog.msfsx.cn/Article/details/8082406.htm
m.blog.msfsx.cn/Article/details/0862208.htm
m.blog.msfsx.cn/Article/details/3191199.htm
m.blog.msfsx.cn/Article/details/9555939.htm
m.blog.msfsx.cn/Article/details/4288462.htm
m.blog.msfsx.cn/Article/details/1171717.htm
m.blog.msfsx.cn/Article/details/6428680.htm
m.blog.msfsx.cn/Article/details/5335737.htm
m.blog.msfsx.cn/Article/details/8640800.htm
m.blog.msfsx.cn/Article/details/3795559.htm
m.blog.msfsx.cn/Article/details/9355537.htm
m.blog.msfsx.cn/Article/details/0002062.htm
m.blog.msfsx.cn/Article/details/7333759.htm
m.blog.msfsx.cn/Article/details/4068606.htm
m.blog.msfsx.cn/Article/details/1739351.htm
m.blog.msfsx.cn/Article/details/3397739.htm
m.blog.msfsx.cn/Article/details/2246482.htm
m.blog.msfsx.cn/Article/details/6404844.htm
m.blog.msfsx.cn/Article/details/7731353.htm
m.blog.msfsx.cn/Article/details/6408022.htm
m.blog.msfsx.cn/Article/details/5735339.htm
m.blog.msfsx.cn/Article/details/8804806.htm
m.blog.msfsx.cn/Article/details/9759375.htm
m.blog.msfsx.cn/Article/details/3193137.htm
m.blog.msfsx.cn/Article/details/1115931.htm
m.blog.msfsx.cn/Article/details/7539531.htm
m.blog.msfsx.cn/Article/details/3355999.htm
m.blog.msfsx.cn/Article/details/4282448.htm
m.blog.msfsx.cn/Article/details/3399511.htm
m.blog.msfsx.cn/Article/details/2284088.htm
m.blog.msfsx.cn/Article/details/1371511.htm
m.blog.msfsx.cn/Article/details/8620066.htm
m.blog.msfsx.cn/Article/details/0668840.htm
m.blog.msfsx.cn/Article/details/6008642.htm
m.blog.msfsx.cn/Article/details/2242606.htm
m.blog.msfsx.cn/Article/details/6408484.htm
m.blog.msfsx.cn/Article/details/7391377.htm
m.blog.msfsx.cn/Article/details/6488804.htm
m.blog.msfsx.cn/Article/details/4088282.htm
m.blog.msfsx.cn/Article/details/9115539.htm
m.blog.msfsx.cn/Article/details/5511711.htm
m.blog.msfsx.cn/Article/details/1973957.htm
m.blog.msfsx.cn/Article/details/8224662.htm
m.blog.msfsx.cn/Article/details/0840624.htm
m.blog.msfsx.cn/Article/details/2484448.htm
m.blog.msfsx.cn/Article/details/0486828.htm
m.blog.msfsx.cn/Article/details/2868628.htm
m.blog.msfsx.cn/Article/details/2828042.htm
m.blog.msfsx.cn/Article/details/7339573.htm
m.blog.msfsx.cn/Article/details/9175373.htm
m.blog.msfsx.cn/Article/details/3593733.htm
m.blog.msfsx.cn/Article/details/0688802.htm
m.blog.msfsx.cn/Article/details/3519353.htm
m.blog.msfsx.cn/Article/details/7911391.htm
m.blog.msfsx.cn/Article/details/1153593.htm
m.blog.msfsx.cn/Article/details/5991975.htm
m.blog.msfsx.cn/Article/details/9177317.htm
m.blog.msfsx.cn/Article/details/5771311.htm
m.blog.msfsx.cn/Article/details/4268600.htm
m.blog.msfsx.cn/Article/details/5995719.htm
m.blog.msfsx.cn/Article/details/9135153.htm
m.blog.msfsx.cn/Article/details/5939773.htm
m.blog.msfsx.cn/Article/details/1551775.htm
m.blog.msfsx.cn/Article/details/0880644.htm
m.blog.msfsx.cn/Article/details/3775573.htm
m.blog.msfsx.cn/Article/details/8684644.htm
m.blog.msfsx.cn/Article/details/7759595.htm
m.blog.msfsx.cn/Article/details/5531717.htm
m.blog.msfsx.cn/Article/details/0206866.htm
m.blog.msfsx.cn/Article/details/3339793.htm
m.blog.msfsx.cn/Article/details/9911935.htm
m.blog.msfsx.cn/Article/details/2682240.htm
m.blog.msfsx.cn/Article/details/3137731.htm
m.blog.msfsx.cn/Article/details/9155193.htm
m.blog.msfsx.cn/Article/details/1111531.htm
m.blog.msfsx.cn/Article/details/7573351.htm
m.blog.msfsx.cn/Article/details/3975311.htm
m.blog.msfsx.cn/Article/details/3579957.htm
m.blog.msfsx.cn/Article/details/6206684.htm
m.blog.msfsx.cn/Article/details/9375735.htm
m.blog.msfsx.cn/Article/details/1759337.htm
m.blog.msfsx.cn/Article/details/0468222.htm
m.blog.msfsx.cn/Article/details/9575197.htm
m.blog.msfsx.cn/Article/details/1979397.htm
m.blog.msfsx.cn/Article/details/3917757.htm
m.blog.msfsx.cn/Article/details/3157119.htm
m.blog.msfsx.cn/Article/details/3737953.htm
m.blog.msfsx.cn/Article/details/7937779.htm
m.blog.msfsx.cn/Article/details/9571571.htm
m.blog.msfsx.cn/Article/details/3915531.htm
m.blog.msfsx.cn/Article/details/1557799.htm
m.blog.msfsx.cn/Article/details/9999731.htm
m.blog.msfsx.cn/Article/details/7779339.htm
m.blog.msfsx.cn/Article/details/8406444.htm
m.blog.msfsx.cn/Article/details/7133733.htm
m.blog.msfsx.cn/Article/details/8466086.htm
m.blog.msfsx.cn/Article/details/5197993.htm
m.blog.msfsx.cn/Article/details/3173311.htm
m.blog.msfsx.cn/Article/details/4006224.htm
m.blog.msfsx.cn/Article/details/7559511.htm
m.blog.msfsx.cn/Article/details/5313937.htm
m.blog.msfsx.cn/Article/details/3379957.htm
m.blog.msfsx.cn/Article/details/7755557.htm
m.blog.msfsx.cn/Article/details/8206402.htm
m.blog.msfsx.cn/Article/details/1575713.htm
m.blog.msfsx.cn/Article/details/6266802.htm
m.blog.msfsx.cn/Article/details/5177913.htm
m.blog.msfsx.cn/Article/details/7197711.htm
m.blog.msfsx.cn/Article/details/3733111.htm
m.blog.msfsx.cn/Article/details/9337955.htm
m.blog.msfsx.cn/Article/details/5315717.htm
m.blog.msfsx.cn/Article/details/5755533.htm
m.blog.msfsx.cn/Article/details/3559739.htm
m.blog.msfsx.cn/Article/details/9375973.htm
m.blog.msfsx.cn/Article/details/1757597.htm
m.blog.msfsx.cn/Article/details/1357131.htm
m.blog.msfsx.cn/Article/details/5971991.htm
m.blog.msfsx.cn/Article/details/3977559.htm
m.blog.msfsx.cn/Article/details/4884040.htm
m.blog.msfsx.cn/Article/details/1797911.htm
m.blog.msfsx.cn/Article/details/6004262.htm
m.blog.msfsx.cn/Article/details/6046224.htm
m.blog.msfsx.cn/Article/details/1353131.htm
m.blog.msfsx.cn/Article/details/8644266.htm
m.blog.msfsx.cn/Article/details/7197315.htm
m.blog.msfsx.cn/Article/details/8608482.htm
m.blog.msfsx.cn/Article/details/8824226.htm
m.blog.msfsx.cn/Article/details/6660428.htm
m.blog.msfsx.cn/Article/details/6002404.htm
m.blog.msfsx.cn/Article/details/5799111.htm
m.blog.msfsx.cn/Article/details/6068882.htm
m.blog.msfsx.cn/Article/details/7771717.htm
m.blog.msfsx.cn/Article/details/8204600.htm
m.blog.msfsx.cn/Article/details/5133117.htm
m.blog.msfsx.cn/Article/details/9135593.htm
m.blog.msfsx.cn/Article/details/4024202.htm
m.blog.msfsx.cn/Article/details/7591195.htm
m.blog.msfsx.cn/Article/details/8286880.htm
m.blog.msfsx.cn/Article/details/2800808.htm
m.blog.msfsx.cn/Article/details/9517739.htm
m.blog.msfsx.cn/Article/details/2086248.htm
m.blog.msfsx.cn/Article/details/3777517.htm
m.blog.msfsx.cn/Article/details/2028604.htm
m.blog.msfsx.cn/Article/details/9791931.htm
m.blog.msfsx.cn/Article/details/5531139.htm
m.blog.msfsx.cn/Article/details/0226228.htm
m.blog.msfsx.cn/Article/details/1171171.htm
m.blog.msfsx.cn/Article/details/7159591.htm
m.blog.msfsx.cn/Article/details/6808806.htm
m.blog.msfsx.cn/Article/details/9775951.htm
m.blog.msfsx.cn/Article/details/1951795.htm
m.blog.msfsx.cn/Article/details/7397331.htm
m.blog.msfsx.cn/Article/details/9155157.htm
m.blog.msfsx.cn/Article/details/2840844.htm
m.blog.msfsx.cn/Article/details/0604840.htm
m.blog.msfsx.cn/Article/details/6866446.htm
m.blog.msfsx.cn/Article/details/9515139.htm
m.blog.msfsx.cn/Article/details/7979511.htm
m.blog.msfsx.cn/Article/details/7155733.htm
m.blog.msfsx.cn/Article/details/7179731.htm
m.blog.msfsx.cn/Article/details/3999591.htm
m.blog.msfsx.cn/Article/details/6044644.htm
m.blog.msfsx.cn/Article/details/5733399.htm
m.blog.msfsx.cn/Article/details/0864206.htm
m.blog.msfsx.cn/Article/details/1335155.htm
m.blog.msfsx.cn/Article/details/5731359.htm
m.blog.msfsx.cn/Article/details/4286644.htm
m.blog.msfsx.cn/Article/details/9175177.htm
m.blog.msfsx.cn/Article/details/8046004.htm
m.blog.msfsx.cn/Article/details/7739517.htm
m.blog.msfsx.cn/Article/details/7973557.htm
m.blog.msfsx.cn/Article/details/2668886.htm
m.blog.msfsx.cn/Article/details/8668860.htm
m.blog.msfsx.cn/Article/details/5535975.htm
m.blog.msfsx.cn/Article/details/3975175.htm
m.blog.msfsx.cn/Article/details/0080666.htm
m.blog.msfsx.cn/Article/details/6262684.htm
m.blog.msfsx.cn/Article/details/9751591.htm
m.blog.msfsx.cn/Article/details/5777315.htm
m.blog.msfsx.cn/Article/details/1937157.htm
m.blog.msfsx.cn/Article/details/5559135.htm
m.blog.msfsx.cn/Article/details/8468682.htm
m.blog.msfsx.cn/Article/details/5951739.htm
m.blog.msfsx.cn/Article/details/5191315.htm
m.blog.msfsx.cn/Article/details/4282266.htm
m.blog.msfsx.cn/Article/details/7351575.htm
m.blog.msfsx.cn/Article/details/3577731.htm
m.blog.msfsx.cn/Article/details/6288068.htm
m.blog.msfsx.cn/Article/details/4040008.htm
m.blog.msfsx.cn/Article/details/8868244.htm
m.blog.msfsx.cn/Article/details/1753593.htm
m.blog.msfsx.cn/Article/details/5173551.htm
m.blog.msfsx.cn/Article/details/5799959.htm
m.blog.msfsx.cn/Article/details/1137779.htm
m.blog.msfsx.cn/Article/details/9553139.htm
m.blog.msfsx.cn/Article/details/3737179.htm
m.blog.msfsx.cn/Article/details/3315995.htm
m.blog.msfsx.cn/Article/details/2040888.htm
m.blog.msfsx.cn/Article/details/7155335.htm
m.blog.msfsx.cn/Article/details/3955997.htm
m.blog.msfsx.cn/Article/details/8646462.htm
m.blog.msfsx.cn/Article/details/2402208.htm
m.blog.msfsx.cn/Article/details/8628608.htm
m.blog.msfsx.cn/Article/details/0060846.htm
m.blog.msfsx.cn/Article/details/5773193.htm
m.blog.msfsx.cn/Article/details/3973135.htm
m.blog.msfsx.cn/Article/details/3351551.htm
m.blog.msfsx.cn/Article/details/3117993.htm
m.blog.msfsx.cn/Article/details/0066864.htm
m.blog.msfsx.cn/Article/details/5913933.htm
m.blog.msfsx.cn/Article/details/0446224.htm
m.blog.msfsx.cn/Article/details/4440062.htm
m.blog.msfsx.cn/Article/details/9739719.htm
m.blog.msfsx.cn/Article/details/3773319.htm
m.blog.msfsx.cn/Article/details/5979977.htm
m.blog.msfsx.cn/Article/details/1531575.htm
m.blog.msfsx.cn/Article/details/3997351.htm
m.blog.msfsx.cn/Article/details/7579171.htm
m.blog.msfsx.cn/Article/details/1577513.htm
m.blog.msfsx.cn/Article/details/7315175.htm
m.blog.msfsx.cn/Article/details/8008040.htm
m.blog.msfsx.cn/Article/details/7739357.htm
m.blog.msfsx.cn/Article/details/6228004.htm
m.blog.msfsx.cn/Article/details/6820206.htm
m.blog.msfsx.cn/Article/details/6888680.htm
m.blog.msfsx.cn/Article/details/7153971.htm
m.blog.msfsx.cn/Article/details/8666486.htm
m.blog.msfsx.cn/Article/details/0264882.htm
m.blog.msfsx.cn/Article/details/6464886.htm
m.blog.msfsx.cn/Article/details/6488066.htm
m.blog.msfsx.cn/Article/details/1777591.htm
m.blog.msfsx.cn/Article/details/9917731.htm
m.blog.msfsx.cn/Article/details/9713751.htm
m.blog.msfsx.cn/Article/details/9377535.htm
m.blog.msfsx.cn/Article/details/8222284.htm
m.blog.msfsx.cn/Article/details/3719315.htm
m.blog.msfsx.cn/Article/details/6408440.htm
m.blog.msfsx.cn/Article/details/3171555.htm
m.blog.msfsx.cn/Article/details/7517997.htm
m.blog.msfsx.cn/Article/details/6808020.htm
m.blog.msfsx.cn/Article/details/7777133.htm
m.blog.msfsx.cn/Article/details/0802048.htm
m.blog.msfsx.cn/Article/details/5971357.htm
m.blog.msfsx.cn/Article/details/5533391.htm
m.blog.msfsx.cn/Article/details/9757579.htm
m.blog.msfsx.cn/Article/details/3177175.htm
m.blog.msfsx.cn/Article/details/4004840.htm
m.blog.msfsx.cn/Article/details/2646264.htm
m.blog.msfsx.cn/Article/details/1319517.htm
m.blog.msfsx.cn/Article/details/7111137.htm
m.blog.msfsx.cn/Article/details/1735337.htm
m.blog.msfsx.cn/Article/details/3517139.htm
m.blog.msfsx.cn/Article/details/0460284.htm
m.blog.msfsx.cn/Article/details/6600646.htm
m.blog.msfsx.cn/Article/details/0228024.htm
m.blog.msfsx.cn/Article/details/5991117.htm
m.blog.msfsx.cn/Article/details/8288688.htm
m.blog.msfsx.cn/Article/details/3313957.htm
m.blog.msfsx.cn/Article/details/7377953.htm
m.blog.msfsx.cn/Article/details/8660204.htm
m.blog.msfsx.cn/Article/details/7573355.htm
m.blog.msfsx.cn/Article/details/5775571.htm
m.blog.msfsx.cn/Article/details/0202080.htm
m.blog.msfsx.cn/Article/details/6866684.htm
m.blog.msfsx.cn/Article/details/5955173.htm
m.blog.msfsx.cn/Article/details/1557797.htm
m.blog.msfsx.cn/Article/details/5351537.htm
m.blog.msfsx.cn/Article/details/0608806.htm
m.blog.msfsx.cn/Article/details/0008464.htm
m.blog.msfsx.cn/Article/details/6002044.htm
m.blog.msfsx.cn/Article/details/7539519.htm
m.blog.msfsx.cn/Article/details/9515579.htm
m.blog.msfsx.cn/Article/details/4806260.htm
m.blog.msfsx.cn/Article/details/6480066.htm
m.blog.msfsx.cn/Article/details/9191513.htm
m.blog.msfsx.cn/Article/details/0446828.htm
m.blog.msfsx.cn/Article/details/7995559.htm
m.blog.msfsx.cn/Article/details/1751771.htm
m.blog.msfsx.cn/Article/details/4684648.htm
m.blog.msfsx.cn/Article/details/3953551.htm
m.blog.msfsx.cn/Article/details/3199151.htm
m.blog.msfsx.cn/Article/details/1939733.htm
m.blog.msfsx.cn/Article/details/9973711.htm
m.blog.msfsx.cn/Article/details/2820602.htm
m.blog.msfsx.cn/Article/details/7917175.htm
m.blog.msfsx.cn/Article/details/7997571.htm
m.blog.msfsx.cn/Article/details/7599157.htm
m.blog.msfsx.cn/Article/details/1971953.htm
m.blog.msfsx.cn/Article/details/7357319.htm
m.blog.msfsx.cn/Article/details/9535559.htm
m.blog.msfsx.cn/Article/details/1313991.htm
m.blog.msfsx.cn/Article/details/4000620.htm
m.blog.msfsx.cn/Article/details/9511915.htm
m.blog.msfsx.cn/Article/details/8204248.htm
m.blog.msfsx.cn/Article/details/2444862.htm
m.blog.msfsx.cn/Article/details/5797139.htm
m.blog.msfsx.cn/Article/details/1735399.htm
m.blog.msfsx.cn/Article/details/3113757.htm
m.blog.msfsx.cn/Article/details/2840240.htm
m.blog.msfsx.cn/Article/details/9757137.htm
m.blog.msfsx.cn/Article/details/6600600.htm
m.blog.msfsx.cn/Article/details/2828022.htm
m.blog.msfsx.cn/Article/details/0004868.htm
m.blog.msfsx.cn/Article/details/0040084.htm
m.blog.msfsx.cn/Article/details/3179179.htm
m.blog.msfsx.cn/Article/details/7533317.htm
m.blog.msfsx.cn/Article/details/0204606.htm
m.blog.msfsx.cn/Article/details/3175351.htm
m.blog.msfsx.cn/Article/details/2404282.htm
m.blog.msfsx.cn/Article/details/4008828.htm
m.blog.msfsx.cn/Article/details/5377157.htm
m.blog.msfsx.cn/Article/details/4826448.htm
m.blog.msfsx.cn/Article/details/4628046.htm
m.blog.msfsx.cn/Article/details/6808646.htm
m.blog.msfsx.cn/Article/details/6086042.htm
m.blog.msfsx.cn/Article/details/5333379.htm
m.blog.msfsx.cn/Article/details/7555935.htm
m.blog.msfsx.cn/Article/details/1911791.htm
m.blog.msfsx.cn/Article/details/9757935.htm
m.blog.msfsx.cn/Article/details/0846662.htm
m.blog.msfsx.cn/Article/details/1511317.htm
m.blog.msfsx.cn/Article/details/1177793.htm
m.blog.msfsx.cn/Article/details/9771571.htm
m.blog.msfsx.cn/Article/details/5751115.htm
m.blog.msfsx.cn/Article/details/6246622.htm
m.blog.msfsx.cn/Article/details/6684882.htm
m.blog.msfsx.cn/Article/details/5115773.htm
m.blog.msfsx.cn/Article/details/7553375.htm
m.blog.msfsx.cn/Article/details/3993195.htm
m.blog.msfsx.cn/Article/details/6280602.htm
m.blog.msfsx.cn/Article/details/1713939.htm
m.blog.msfsx.cn/Article/details/1973311.htm
m.blog.msfsx.cn/Article/details/1593371.htm
m.blog.msfsx.cn/Article/details/6802042.htm
m.blog.msfsx.cn/Article/details/2828626.htm
m.blog.msfsx.cn/Article/details/9779715.htm
m.blog.msfsx.cn/Article/details/8222064.htm
m.blog.msfsx.cn/Article/details/4408244.htm
m.blog.msfsx.cn/Article/details/9937533.htm
m.blog.msfsx.cn/Article/details/6408060.htm
m.blog.msfsx.cn/Article/details/3977955.htm
m.blog.msfsx.cn/Article/details/6000482.htm
m.blog.msfsx.cn/Article/details/9511717.htm
m.blog.msfsx.cn/Article/details/1155515.htm
m.blog.msfsx.cn/Article/details/9551137.htm
m.blog.msfsx.cn/Article/details/5193711.htm
m.blog.msfsx.cn/Article/details/4024628.htm
m.blog.msfsx.cn/Article/details/6428066.htm
m.blog.msfsx.cn/Article/details/9159537.htm
m.blog.msfsx.cn/Article/details/0680866.htm
m.blog.msfsx.cn/Article/details/9753531.htm
m.blog.msfsx.cn/Article/details/8688206.htm
m.blog.msfsx.cn/Article/details/9999719.htm
m.blog.msfsx.cn/Article/details/5755531.htm
m.blog.msfsx.cn/Article/details/5337139.htm
m.blog.msfsx.cn/Article/details/3931759.htm
m.blog.msfsx.cn/Article/details/9797377.htm
m.blog.msfsx.cn/Article/details/4208820.htm
m.blog.msfsx.cn/Article/details/1755155.htm
m.blog.msfsx.cn/Article/details/5337377.htm
