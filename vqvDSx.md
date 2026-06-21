食恳钒雇赡


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

浪奈怨涝钠偾兴驳俏凶托纲捶挪导

m.wvm.qtbzn.cn/64460.Doc
m.wvm.qtbzn.cn/60286.Doc
m.wvm.qtbzn.cn/44824.Doc
m.wvm.qtbzn.cn/60408.Doc
m.wvm.qtbzn.cn/75131.Doc
m.wvm.qtbzn.cn/51331.Doc
m.wvm.qtbzn.cn/57591.Doc
m.wvm.qtbzn.cn/53911.Doc
m.wvm.qtbzn.cn/13557.Doc
m.wvm.qtbzn.cn/64842.Doc
m.wvn.qtbzn.cn/84202.Doc
m.wvn.qtbzn.cn/55575.Doc
m.wvn.qtbzn.cn/84428.Doc
m.wvn.qtbzn.cn/77311.Doc
m.wvn.qtbzn.cn/9.Doc
m.wvn.qtbzn.cn/11933.Doc
m.wvn.qtbzn.cn/60488.Doc
m.wvn.qtbzn.cn/35955.Doc
m.wvn.qtbzn.cn/39595.Doc
m.wvn.qtbzn.cn/84486.Doc
m.wvn.qtbzn.cn/17535.Doc
m.wvn.qtbzn.cn/77931.Doc
m.wvn.qtbzn.cn/59195.Doc
m.wvn.qtbzn.cn/93795.Doc
m.wvn.qtbzn.cn/75159.Doc
m.wvn.qtbzn.cn/97973.Doc
m.wvn.qtbzn.cn/20462.Doc
m.wvn.qtbzn.cn/13173.Doc
m.wvn.qtbzn.cn/48022.Doc
m.wvn.qtbzn.cn/11579.Doc
m.wvb.qtbzn.cn/44080.Doc
m.wvb.qtbzn.cn/59519.Doc
m.wvb.qtbzn.cn/75159.Doc
m.wvb.qtbzn.cn/79113.Doc
m.wvb.qtbzn.cn/08802.Doc
m.wvb.qtbzn.cn/40604.Doc
m.wvb.qtbzn.cn/53737.Doc
m.wvb.qtbzn.cn/91517.Doc
m.wvb.qtbzn.cn/62466.Doc
m.wvb.qtbzn.cn/88848.Doc
m.wvb.qtbzn.cn/39971.Doc
m.wvb.qtbzn.cn/11559.Doc
m.wvb.qtbzn.cn/46680.Doc
m.wvb.qtbzn.cn/79717.Doc
m.wvb.qtbzn.cn/93795.Doc
m.wvb.qtbzn.cn/82284.Doc
m.wvb.qtbzn.cn/71939.Doc
m.wvb.qtbzn.cn/33791.Doc
m.wvb.qtbzn.cn/55979.Doc
m.wvb.qtbzn.cn/71957.Doc
m.wvv.qtbzn.cn/99377.Doc
m.wvv.qtbzn.cn/40682.Doc
m.wvv.qtbzn.cn/82468.Doc
m.wvv.qtbzn.cn/13917.Doc
m.wvv.qtbzn.cn/62828.Doc
m.wvv.qtbzn.cn/77151.Doc
m.wvv.qtbzn.cn/82026.Doc
m.wvv.qtbzn.cn/77751.Doc
m.wvv.qtbzn.cn/33179.Doc
m.wvv.qtbzn.cn/79937.Doc
m.wvv.qtbzn.cn/55799.Doc
m.wvv.qtbzn.cn/77171.Doc
m.wvv.qtbzn.cn/7.Doc
m.wvv.qtbzn.cn/35155.Doc
m.wvv.qtbzn.cn/68828.Doc
m.wvv.qtbzn.cn/02684.Doc
m.wvv.qtbzn.cn/42264.Doc
m.wvv.qtbzn.cn/99573.Doc
m.wvv.qtbzn.cn/31199.Doc
m.wvv.qtbzn.cn/82008.Doc
m.wvc.qtbzn.cn/75195.Doc
m.wvc.qtbzn.cn/91139.Doc
m.wvc.qtbzn.cn/04488.Doc
m.wvc.qtbzn.cn/46466.Doc
m.wvc.qtbzn.cn/97999.Doc
m.wvc.qtbzn.cn/79935.Doc
m.wvc.qtbzn.cn/73395.Doc
m.wvc.qtbzn.cn/53339.Doc
m.wvc.qtbzn.cn/40468.Doc
m.wvc.qtbzn.cn/42242.Doc
m.wvc.qtbzn.cn/66606.Doc
m.wvc.qtbzn.cn/93375.Doc
m.wvc.qtbzn.cn/71913.Doc
m.wvc.qtbzn.cn/84026.Doc
m.wvc.qtbzn.cn/55919.Doc
m.wvc.qtbzn.cn/80868.Doc
m.wvc.qtbzn.cn/95553.Doc
m.wvc.qtbzn.cn/11979.Doc
m.wvc.qtbzn.cn/33157.Doc
m.wvc.qtbzn.cn/15375.Doc
m.wvx.qtbzn.cn/77977.Doc
m.wvx.qtbzn.cn/55377.Doc
m.wvx.qtbzn.cn/19155.Doc
m.wvx.qtbzn.cn/79591.Doc
m.wvx.qtbzn.cn/55337.Doc
m.wvx.qtbzn.cn/00444.Doc
m.wvx.qtbzn.cn/13313.Doc
m.wvx.qtbzn.cn/95911.Doc
m.wvx.qtbzn.cn/53979.Doc
m.wvx.qtbzn.cn/35939.Doc
m.wvx.qtbzn.cn/48808.Doc
m.wvx.qtbzn.cn/33177.Doc
m.wvx.qtbzn.cn/42086.Doc
m.wvx.qtbzn.cn/57577.Doc
m.wvx.qtbzn.cn/24840.Doc
m.wvx.qtbzn.cn/86200.Doc
m.wvx.qtbzn.cn/71393.Doc
m.wvx.qtbzn.cn/08684.Doc
m.wvx.qtbzn.cn/99319.Doc
m.wvx.qtbzn.cn/26840.Doc
m.wvz.qtbzn.cn/77177.Doc
m.wvz.qtbzn.cn/86886.Doc
m.wvz.qtbzn.cn/77511.Doc
m.wvz.qtbzn.cn/53737.Doc
m.wvz.qtbzn.cn/44426.Doc
m.wvz.qtbzn.cn/31355.Doc
m.wvz.qtbzn.cn/75917.Doc
m.wvz.qtbzn.cn/42462.Doc
m.wvz.qtbzn.cn/57313.Doc
m.wvz.qtbzn.cn/84860.Doc
m.wvz.qtbzn.cn/13133.Doc
m.wvz.qtbzn.cn/77135.Doc
m.wvz.qtbzn.cn/99793.Doc
m.wvz.qtbzn.cn/73753.Doc
m.wvz.qtbzn.cn/24206.Doc
m.wvz.qtbzn.cn/39317.Doc
m.wvz.qtbzn.cn/95739.Doc
m.wvz.qtbzn.cn/59591.Doc
m.wvz.qtbzn.cn/75739.Doc
m.wvz.qtbzn.cn/44864.Doc
m.wvl.qtbzn.cn/35951.Doc
m.wvl.qtbzn.cn/53355.Doc
m.wvl.qtbzn.cn/42082.Doc
m.wvl.qtbzn.cn/24406.Doc
m.wvl.qtbzn.cn/08806.Doc
m.wvl.qtbzn.cn/95533.Doc
m.wvl.qtbzn.cn/91195.Doc
m.wvl.qtbzn.cn/97317.Doc
m.wvl.qtbzn.cn/59573.Doc
m.wvl.qtbzn.cn/73711.Doc
m.wvl.qtbzn.cn/59377.Doc
m.wvl.qtbzn.cn/66062.Doc
m.wvl.qtbzn.cn/22604.Doc
m.wvl.qtbzn.cn/35773.Doc
m.wvl.qtbzn.cn/40448.Doc
m.wvl.qtbzn.cn/88604.Doc
m.wvl.qtbzn.cn/93713.Doc
m.wvl.qtbzn.cn/35577.Doc
m.wvl.qtbzn.cn/13159.Doc
m.wvl.qtbzn.cn/57917.Doc
m.wvk.qtbzn.cn/02824.Doc
m.wvk.qtbzn.cn/68668.Doc
m.wvk.qtbzn.cn/75111.Doc
m.wvk.qtbzn.cn/15757.Doc
m.wvk.qtbzn.cn/17175.Doc
m.wvk.qtbzn.cn/73195.Doc
m.wvk.qtbzn.cn/39139.Doc
m.wvk.qtbzn.cn/95971.Doc
m.wvk.qtbzn.cn/31155.Doc
m.wvk.qtbzn.cn/06284.Doc
m.wvk.qtbzn.cn/39555.Doc
m.wvk.qtbzn.cn/95595.Doc
m.wvk.qtbzn.cn/53933.Doc
m.wvk.qtbzn.cn/80828.Doc
m.wvk.qtbzn.cn/97733.Doc
m.wvk.qtbzn.cn/75559.Doc
m.wvk.qtbzn.cn/82062.Doc
m.wvk.qtbzn.cn/13737.Doc
m.wvk.qtbzn.cn/77995.Doc
m.wvk.qtbzn.cn/57779.Doc
m.wvj.qtbzn.cn/08664.Doc
m.wvj.qtbzn.cn/77579.Doc
m.wvj.qtbzn.cn/55797.Doc
m.wvj.qtbzn.cn/39937.Doc
m.wvj.qtbzn.cn/73379.Doc
m.wvj.qtbzn.cn/59399.Doc
m.wvj.qtbzn.cn/51993.Doc
m.wvj.qtbzn.cn/99193.Doc
m.wvj.qtbzn.cn/91395.Doc
m.wvj.qtbzn.cn/00620.Doc
m.wvj.qtbzn.cn/53193.Doc
m.wvj.qtbzn.cn/68068.Doc
m.wvj.qtbzn.cn/08200.Doc
m.wvj.qtbzn.cn/19931.Doc
m.wvj.qtbzn.cn/31557.Doc
m.wvj.qtbzn.cn/53599.Doc
m.wvj.qtbzn.cn/57935.Doc
m.wvj.qtbzn.cn/02484.Doc
m.wvj.qtbzn.cn/66264.Doc
m.wvj.qtbzn.cn/80486.Doc
m.wvh.qtbzn.cn/51593.Doc
m.wvh.qtbzn.cn/19111.Doc
m.wvh.qtbzn.cn/48042.Doc
m.wvh.qtbzn.cn/79737.Doc
m.wvh.qtbzn.cn/73955.Doc
m.wvh.qtbzn.cn/57751.Doc
m.wvh.qtbzn.cn/39515.Doc
m.wvh.qtbzn.cn/75791.Doc
m.wvh.qtbzn.cn/19755.Doc
m.wvh.qtbzn.cn/11171.Doc
m.wvh.qtbzn.cn/17533.Doc
m.wvh.qtbzn.cn/55571.Doc
m.wvh.qtbzn.cn/64042.Doc
m.wvh.qtbzn.cn/97595.Doc
m.wvh.qtbzn.cn/60626.Doc
m.wvh.qtbzn.cn/13533.Doc
m.wvh.qtbzn.cn/68242.Doc
m.wvh.qtbzn.cn/37393.Doc
m.wvh.qtbzn.cn/97391.Doc
m.wvh.qtbzn.cn/35333.Doc
m.wvg.qtbzn.cn/55537.Doc
m.wvg.qtbzn.cn/79733.Doc
m.wvg.qtbzn.cn/26062.Doc
m.wvg.qtbzn.cn/55731.Doc
m.wvg.qtbzn.cn/73995.Doc
m.wvg.qtbzn.cn/99751.Doc
m.wvg.qtbzn.cn/97571.Doc
m.wvg.qtbzn.cn/51797.Doc
m.wvg.qtbzn.cn/93357.Doc
m.wvg.qtbzn.cn/51953.Doc
m.wvg.qtbzn.cn/71731.Doc
m.wvg.qtbzn.cn/31153.Doc
m.wvg.qtbzn.cn/99113.Doc
m.wvg.qtbzn.cn/66402.Doc
m.wvg.qtbzn.cn/35331.Doc
m.wvg.qtbzn.cn/84424.Doc
m.wvg.qtbzn.cn/99339.Doc
m.wvg.qtbzn.cn/95579.Doc
m.wvg.qtbzn.cn/33311.Doc
m.wvg.qtbzn.cn/22208.Doc
m.wvf.qtbzn.cn/24688.Doc
m.wvf.qtbzn.cn/11357.Doc
m.wvf.qtbzn.cn/13973.Doc
m.wvf.qtbzn.cn/99957.Doc
m.wvf.qtbzn.cn/62046.Doc
m.wvf.qtbzn.cn/35331.Doc
m.wvf.qtbzn.cn/13173.Doc
m.wvf.qtbzn.cn/11597.Doc
m.wvf.qtbzn.cn/35373.Doc
m.wvf.qtbzn.cn/35113.Doc
m.wvf.qtbzn.cn/22844.Doc
m.wvf.qtbzn.cn/79195.Doc
m.wvf.qtbzn.cn/99517.Doc
m.wvf.qtbzn.cn/42266.Doc
m.wvf.qtbzn.cn/19177.Doc
m.wvf.qtbzn.cn/02688.Doc
m.wvf.qtbzn.cn/35751.Doc
m.wvf.qtbzn.cn/35751.Doc
m.wvf.qtbzn.cn/66862.Doc
m.wvf.qtbzn.cn/79975.Doc
m.wvd.qtbzn.cn/59199.Doc
m.wvd.qtbzn.cn/28642.Doc
m.wvd.qtbzn.cn/42480.Doc
m.wvd.qtbzn.cn/28642.Doc
m.wvd.qtbzn.cn/06682.Doc
m.wvd.qtbzn.cn/88008.Doc
m.wvd.qtbzn.cn/42684.Doc
m.wvd.qtbzn.cn/55799.Doc
m.wvd.qtbzn.cn/75593.Doc
m.wvd.qtbzn.cn/88606.Doc
m.wvd.qtbzn.cn/42048.Doc
m.wvd.qtbzn.cn/11377.Doc
m.wvd.qtbzn.cn/51959.Doc
m.wvd.qtbzn.cn/11311.Doc
m.wvd.qtbzn.cn/97331.Doc
m.wvd.qtbzn.cn/35991.Doc
m.wvd.qtbzn.cn/28046.Doc
m.wvd.qtbzn.cn/64020.Doc
m.wvd.qtbzn.cn/42882.Doc
m.wvd.qtbzn.cn/22866.Doc
m.wvs.qtbzn.cn/53557.Doc
m.wvs.qtbzn.cn/44400.Doc
m.wvs.qtbzn.cn/55713.Doc
m.wvs.qtbzn.cn/33797.Doc
m.wvs.qtbzn.cn/68022.Doc
m.wvs.qtbzn.cn/11399.Doc
m.wvs.qtbzn.cn/28060.Doc
m.wvs.qtbzn.cn/68460.Doc
m.wvs.qtbzn.cn/44288.Doc
m.wvs.qtbzn.cn/24420.Doc
m.wvs.qtbzn.cn/95373.Doc
m.wvs.qtbzn.cn/95553.Doc
m.wvs.qtbzn.cn/33155.Doc
m.wvs.qtbzn.cn/22026.Doc
m.wvs.qtbzn.cn/77795.Doc
m.wvs.qtbzn.cn/51175.Doc
m.wvs.qtbzn.cn/28466.Doc
m.wvs.qtbzn.cn/37913.Doc
m.wvs.qtbzn.cn/02406.Doc
m.wvs.qtbzn.cn/80820.Doc
m.wva.qtbzn.cn/75391.Doc
m.wva.qtbzn.cn/35957.Doc
m.wva.qtbzn.cn/04248.Doc
m.wva.qtbzn.cn/86446.Doc
m.wva.qtbzn.cn/02284.Doc
m.wva.qtbzn.cn/82868.Doc
m.wva.qtbzn.cn/37573.Doc
m.wva.qtbzn.cn/82268.Doc
m.wva.qtbzn.cn/77919.Doc
m.wva.qtbzn.cn/11959.Doc
m.wva.qtbzn.cn/59955.Doc
m.wva.qtbzn.cn/66284.Doc
m.wva.qtbzn.cn/62480.Doc
m.wva.qtbzn.cn/82420.Doc
m.wva.qtbzn.cn/99379.Doc
m.wva.qtbzn.cn/99397.Doc
m.wva.qtbzn.cn/73919.Doc
m.wva.qtbzn.cn/95179.Doc
m.wva.qtbzn.cn/00608.Doc
m.wva.qtbzn.cn/17335.Doc
m.wvp.qtbzn.cn/13779.Doc
m.wvp.qtbzn.cn/26420.Doc
m.wvp.qtbzn.cn/28208.Doc
m.wvp.qtbzn.cn/88424.Doc
m.wvp.qtbzn.cn/15797.Doc
m.wvp.qtbzn.cn/88020.Doc
m.wvp.qtbzn.cn/75553.Doc
m.wvp.qtbzn.cn/62868.Doc
m.wvp.qtbzn.cn/84442.Doc
m.wvp.qtbzn.cn/24828.Doc
m.wvp.qtbzn.cn/26260.Doc
m.wvp.qtbzn.cn/53199.Doc
m.wvp.qtbzn.cn/64284.Doc
m.wvp.qtbzn.cn/06008.Doc
m.wvp.qtbzn.cn/31979.Doc
m.wvp.qtbzn.cn/35395.Doc
m.wvp.qtbzn.cn/40268.Doc
m.wvp.qtbzn.cn/91135.Doc
m.wvp.qtbzn.cn/73775.Doc
m.wvp.qtbzn.cn/57919.Doc
m.wvo.qtbzn.cn/55397.Doc
m.wvo.qtbzn.cn/40806.Doc
m.wvo.qtbzn.cn/40486.Doc
m.wvo.qtbzn.cn/68620.Doc
m.wvo.qtbzn.cn/08640.Doc
m.wvo.qtbzn.cn/59351.Doc
m.wvo.qtbzn.cn/53913.Doc
m.wvo.qtbzn.cn/68880.Doc
m.wvo.qtbzn.cn/73591.Doc
m.wvo.qtbzn.cn/77575.Doc
m.wvo.qtbzn.cn/42848.Doc
m.wvo.qtbzn.cn/13579.Doc
m.wvo.qtbzn.cn/82026.Doc
m.wvo.qtbzn.cn/44644.Doc
m.wvo.qtbzn.cn/88086.Doc
m.wvo.qtbzn.cn/17937.Doc
m.wvo.qtbzn.cn/26224.Doc
m.wvo.qtbzn.cn/13599.Doc
m.wvo.qtbzn.cn/71737.Doc
m.wvo.qtbzn.cn/82684.Doc
m.wvi.qtbzn.cn/80868.Doc
m.wvi.qtbzn.cn/22044.Doc
m.wvi.qtbzn.cn/37535.Doc
m.wvi.qtbzn.cn/33155.Doc
m.wvi.qtbzn.cn/37717.Doc
m.wvi.qtbzn.cn/33597.Doc
m.wvi.qtbzn.cn/55979.Doc
m.wvi.qtbzn.cn/60862.Doc
m.wvi.qtbzn.cn/79753.Doc
m.wvi.qtbzn.cn/42806.Doc
m.wvi.qtbzn.cn/71775.Doc
m.wvi.qtbzn.cn/00426.Doc
m.wvi.qtbzn.cn/39799.Doc
m.wvi.qtbzn.cn/75953.Doc
m.wvi.qtbzn.cn/42028.Doc
m.wvi.qtbzn.cn/64608.Doc
m.wvi.qtbzn.cn/20080.Doc
m.wvi.qtbzn.cn/71393.Doc
m.wvi.qtbzn.cn/62048.Doc
m.wvi.qtbzn.cn/66644.Doc
m.wvu.qtbzn.cn/35179.Doc
m.wvu.qtbzn.cn/97937.Doc
m.wvu.qtbzn.cn/33559.Doc
m.wvu.qtbzn.cn/53397.Doc
m.wvu.qtbzn.cn/73771.Doc
m.wvu.qtbzn.cn/86602.Doc
m.wvu.qtbzn.cn/77159.Doc
m.wvu.qtbzn.cn/55773.Doc
m.wvu.qtbzn.cn/13177.Doc
m.wvu.qtbzn.cn/53973.Doc
m.wvu.qtbzn.cn/88402.Doc
m.wvu.qtbzn.cn/20824.Doc
m.wvu.qtbzn.cn/51591.Doc
m.wvu.qtbzn.cn/39937.Doc
m.wvu.qtbzn.cn/39973.Doc
m.wvu.qtbzn.cn/79913.Doc
m.wvu.qtbzn.cn/91939.Doc
m.wvu.qtbzn.cn/93355.Doc
m.wvu.qtbzn.cn/13737.Doc
m.wvu.qtbzn.cn/68062.Doc
m.wvy.qtbzn.cn/91515.Doc
m.wvy.qtbzn.cn/00868.Doc
m.wvy.qtbzn.cn/31533.Doc
m.wvy.qtbzn.cn/55759.Doc
m.wvy.qtbzn.cn/73731.Doc
