# Python 闭包与作用域 —— 函数捕获环境变量
# 闭包是内部函数引用外部函数变量，即使外部函数已执行完毕仍被保留

# 1. 闭包定义与基础示例
def outer_function(message):
    def inner_function():
        print(f"消息: {message}")
    return inner_function

closure1 = outer_function("Hello")
closure2 = outer_function("World")
closure1(); closure2()

# 2. LEGB 作用域规则
"""
变量查找优先级（高->低）：
L - Local（局部）: 当前函数内部
E - Enclosing（外层）: 外部函数（闭包）
G - Global（全局）: 模块级别
B - Built-in（内置）: Python 内置名称
"""
x = "全局变量"
def outer():
    x = "外层变量"
    def inner():
        x = "内层变量"
        print("inner:", x)  # Local
    inner()
    print("outer:", x)      # Enclosing
outer()
print("全局:", x)           # Global

# 3. nonlocal 声明
def make_counter():
    count = 0
    def inc(): nonlocal count; count += 1; return count
    def dec(): nonlocal count; count -= 1; return count
    def reset(v=0): nonlocal count; count = v; return count
    return inc, dec, reset

inc, dec, reset = make_counter()
print("计数:", inc(), inc(), inc())  # 1 2 3
print("减数:", dec())                # 2
print("重置:", reset(100))           # 100

# 4. 闭包用于数据隐藏
def create_account(initial_balance=0):
    """balance 对外部完全不可见"""
    balance = initial_balance
    transactions = []

    def deposit(amount):
        nonlocal balance
        if amount <= 0: raise ValueError("金额需为正")
        balance += amount
        transactions.append(("存入", amount, balance))
        return balance

    def withdraw(amount):
        nonlocal balance
        if amount <= 0: raise ValueError("金额需为正")
        if amount > balance: raise ValueError("余额不足")
        balance -= amount
        transactions.append(("取出", amount, balance))
        return balance

    def get_balance(): return balance
    def get_history(): return list(transactions)
    return {"deposit": deposit, "withdraw": withdraw,
            "balance": get_balance, "history": get_history}

acct = create_account(1000)
acct["deposit"](500); acct["withdraw"](200)
print("余额:", acct["balance"](), "  历史:", acct["history"]())

# 5. 闭包 vs 类 —— 选择指南
"""
闭包优势: 少量状态、轻量、完全隐藏
类优势: 多状态、继承、类型检查
"""
def make_temp():
    _c = 0
    def set_c(v): nonlocal _c; _c = v
    def get_c(): return _c
    def to_f(): return _c * 9/5 + 32
    return {"set": set_c, "get": get_c, "to_f": to_f}

class Temp:
    def __init__(self, c=0): self._c = c
    @property
    def celsius(self): return self._c
    @celsius.setter
    def celsius(self, v): self._c = v
    def to_f(self): return self._c * 9/5 + 32

# 6. cell 对象与 __closure__ 属性
def inspect_closure():
    captured = "我被捕获了"
    another = "我也被捕获了"
    def inner(): print(captured, another)
    return inner

closure = inspect_closure()
print("\n闭包内部:")
print("__closure__:", closure.__closure__)
for i, c in enumerate(closure.__closure__):
    print(f"  cell[{i}].cell_contents = {c.cell_contents!r}")
print("co_freevars:", closure.__code__.co_freevars)

def debug_closure(func):
    if func.__closure__:
        print(f"{func.__name__} 捕获了:")
        for n, c in zip(func.__code__.co_freevars, func.__closure__):
            print(f"  {n} = {c.cell_contents!r}")
    else:
        print(f"{func.__name__} 无捕获")

debug_closure(closure)
debug_closure(lambda: 42)

wpi.qjfkLsjdfkLaf.cn/40002.Doc
wpi.qjfkLsjdfkLaf.cn/48262.Doc
wpi.qjfkLsjdfkLaf.cn/86888.Doc
wpi.qjfkLsjdfkLaf.cn/22660.Doc
wpi.qjfkLsjdfkLaf.cn/75759.Doc
wpi.qjfkLsjdfkLaf.cn/40428.Doc
wpi.qjfkLsjdfkLaf.cn/80622.Doc
wpi.qjfkLsjdfkLaf.cn/75577.Doc
wpi.qjfkLsjdfkLaf.cn/86684.Doc
wpi.qjfkLsjdfkLaf.cn/02826.Doc
wpu.qjfkLsjdfkLaf.cn/33379.Doc
wpu.qjfkLsjdfkLaf.cn/20644.Doc
wpu.qjfkLsjdfkLaf.cn/62204.Doc
wpu.qjfkLsjdfkLaf.cn/80448.Doc
wpu.qjfkLsjdfkLaf.cn/80282.Doc
wpu.qjfkLsjdfkLaf.cn/42268.Doc
wpu.qjfkLsjdfkLaf.cn/68680.Doc
wpu.qjfkLsjdfkLaf.cn/00266.Doc
wpu.qjfkLsjdfkLaf.cn/44206.Doc
wpu.qjfkLsjdfkLaf.cn/26826.Doc
wpy.qjfkLsjdfkLaf.cn/26448.Doc
wpy.qjfkLsjdfkLaf.cn/59371.Doc
wpy.qjfkLsjdfkLaf.cn/46204.Doc
wpy.qjfkLsjdfkLaf.cn/75391.Doc
wpy.qjfkLsjdfkLaf.cn/68064.Doc
wpy.qjfkLsjdfkLaf.cn/68486.Doc
wpy.qjfkLsjdfkLaf.cn/59513.Doc
wpy.qjfkLsjdfkLaf.cn/17535.Doc
wpy.qjfkLsjdfkLaf.cn/08020.Doc
wpy.qjfkLsjdfkLaf.cn/35937.Doc
wpt.qjfkLsjdfkLaf.cn/26486.Doc
wpt.qjfkLsjdfkLaf.cn/28426.Doc
wpt.qjfkLsjdfkLaf.cn/28426.Doc
wpt.qjfkLsjdfkLaf.cn/46402.Doc
wpt.qjfkLsjdfkLaf.cn/66626.Doc
wpt.qjfkLsjdfkLaf.cn/46024.Doc
wpt.qjfkLsjdfkLaf.cn/62822.Doc
wpt.qjfkLsjdfkLaf.cn/86426.Doc
wpt.qjfkLsjdfkLaf.cn/80286.Doc
wpt.qjfkLsjdfkLaf.cn/40408.Doc
wpr.qjfkLsjdfkLaf.cn/97557.Doc
wpr.qjfkLsjdfkLaf.cn/84202.Doc
wpr.qjfkLsjdfkLaf.cn/79519.Doc
wpr.qjfkLsjdfkLaf.cn/40800.Doc
wpr.qjfkLsjdfkLaf.cn/28240.Doc
wpr.qjfkLsjdfkLaf.cn/59111.Doc
wpr.qjfkLsjdfkLaf.cn/20088.Doc
wpr.qjfkLsjdfkLaf.cn/62468.Doc
wpr.qjfkLsjdfkLaf.cn/44480.Doc
wpr.qjfkLsjdfkLaf.cn/88264.Doc
wpe.qjfkLsjdfkLaf.cn/22440.Doc
wpe.qjfkLsjdfkLaf.cn/04084.Doc
wpe.qjfkLsjdfkLaf.cn/82424.Doc
wpe.qjfkLsjdfkLaf.cn/93399.Doc
wpe.qjfkLsjdfkLaf.cn/28242.Doc
wpe.qjfkLsjdfkLaf.cn/26844.Doc
wpe.qjfkLsjdfkLaf.cn/04208.Doc
wpe.qjfkLsjdfkLaf.cn/02242.Doc
wpe.qjfkLsjdfkLaf.cn/86804.Doc
wpe.qjfkLsjdfkLaf.cn/04240.Doc
wpw.qjfkLsjdfkLaf.cn/28866.Doc
wpw.qjfkLsjdfkLaf.cn/04006.Doc
wpw.qjfkLsjdfkLaf.cn/00800.Doc
wpw.qjfkLsjdfkLaf.cn/64802.Doc
wpw.qjfkLsjdfkLaf.cn/22080.Doc
wpw.qjfkLsjdfkLaf.cn/22688.Doc
wpw.qjfkLsjdfkLaf.cn/60600.Doc
wpw.qjfkLsjdfkLaf.cn/86024.Doc
wpw.qjfkLsjdfkLaf.cn/80802.Doc
wpw.qjfkLsjdfkLaf.cn/95335.Doc
wpq.qjfkLsjdfkLaf.cn/02600.Doc
wpq.qjfkLsjdfkLaf.cn/24462.Doc
wpq.qjfkLsjdfkLaf.cn/08888.Doc
wpq.qjfkLsjdfkLaf.cn/08482.Doc
wpq.qjfkLsjdfkLaf.cn/26204.Doc
wpq.qjfkLsjdfkLaf.cn/24884.Doc
wpq.qjfkLsjdfkLaf.cn/86844.Doc
wpq.qjfkLsjdfkLaf.cn/28604.Doc
wpq.qjfkLsjdfkLaf.cn/00624.Doc
wpq.qjfkLsjdfkLaf.cn/35535.Doc
wom.qjfkLsjdfkLaf.cn/44208.Doc
wom.qjfkLsjdfkLaf.cn/66288.Doc
wom.qjfkLsjdfkLaf.cn/86664.Doc
wom.qjfkLsjdfkLaf.cn/08826.Doc
wom.qjfkLsjdfkLaf.cn/02684.Doc
wom.qjfkLsjdfkLaf.cn/22220.Doc
wom.qjfkLsjdfkLaf.cn/82864.Doc
wom.qjfkLsjdfkLaf.cn/04446.Doc
wom.qjfkLsjdfkLaf.cn/79311.Doc
wom.qjfkLsjdfkLaf.cn/31131.Doc
won.qjfkLsjdfkLaf.cn/82486.Doc
won.qjfkLsjdfkLaf.cn/00228.Doc
won.qjfkLsjdfkLaf.cn/28062.Doc
won.qjfkLsjdfkLaf.cn/04268.Doc
won.qjfkLsjdfkLaf.cn/28628.Doc
won.qjfkLsjdfkLaf.cn/31559.Doc
won.qjfkLsjdfkLaf.cn/24688.Doc
won.qjfkLsjdfkLaf.cn/02840.Doc
won.qjfkLsjdfkLaf.cn/42802.Doc
won.qjfkLsjdfkLaf.cn/53911.Doc
wob.qjfkLsjdfkLaf.cn/04200.Doc
wob.qjfkLsjdfkLaf.cn/62844.Doc
wob.qjfkLsjdfkLaf.cn/60820.Doc
wob.qjfkLsjdfkLaf.cn/20028.Doc
wob.qjfkLsjdfkLaf.cn/82860.Doc
wob.qjfkLsjdfkLaf.cn/08220.Doc
wob.qjfkLsjdfkLaf.cn/75173.Doc
wob.qjfkLsjdfkLaf.cn/02046.Doc
wob.qjfkLsjdfkLaf.cn/64446.Doc
wob.qjfkLsjdfkLaf.cn/80602.Doc
wov.qjfkLsjdfkLaf.cn/62222.Doc
wov.qjfkLsjdfkLaf.cn/60288.Doc
wov.qjfkLsjdfkLaf.cn/04886.Doc
wov.qjfkLsjdfkLaf.cn/06222.Doc
wov.qjfkLsjdfkLaf.cn/42664.Doc
wov.qjfkLsjdfkLaf.cn/20606.Doc
wov.qjfkLsjdfkLaf.cn/88824.Doc
wov.qjfkLsjdfkLaf.cn/15595.Doc
wov.qjfkLsjdfkLaf.cn/88002.Doc
wov.qjfkLsjdfkLaf.cn/60044.Doc
woc.qjfkLsjdfkLaf.cn/22060.Doc
woc.qjfkLsjdfkLaf.cn/04680.Doc
woc.qjfkLsjdfkLaf.cn/46468.Doc
woc.qjfkLsjdfkLaf.cn/22228.Doc
woc.qjfkLsjdfkLaf.cn/68204.Doc
woc.qjfkLsjdfkLaf.cn/60200.Doc
woc.qjfkLsjdfkLaf.cn/11599.Doc
woc.qjfkLsjdfkLaf.cn/80600.Doc
woc.qjfkLsjdfkLaf.cn/44480.Doc
woc.qjfkLsjdfkLaf.cn/91106.Doc
wox.qjfkLsjdfkLaf.cn/81773.Doc
wox.qjfkLsjdfkLaf.cn/93604.Doc
wox.qjfkLsjdfkLaf.cn/32440.Doc
wox.qjfkLsjdfkLaf.cn/20889.Doc
wox.qjfkLsjdfkLaf.cn/28213.Doc
wox.qjfkLsjdfkLaf.cn/98531.Doc
wox.qjfkLsjdfkLaf.cn/68705.Doc
wox.qjfkLsjdfkLaf.cn/38745.Doc
wox.qjfkLsjdfkLaf.cn/36790.Doc
wox.qjfkLsjdfkLaf.cn/07289.Doc
woz.qjfkLsjdfkLaf.cn/82934.Doc
woz.qjfkLsjdfkLaf.cn/61149.Doc
woz.qjfkLsjdfkLaf.cn/48433.Doc
woz.qjfkLsjdfkLaf.cn/66312.Doc
woz.qjfkLsjdfkLaf.cn/27560.Doc
woz.qjfkLsjdfkLaf.cn/47967.Doc
woz.qjfkLsjdfkLaf.cn/28247.Doc
woz.qjfkLsjdfkLaf.cn/18318.Doc
woz.qjfkLsjdfkLaf.cn/62169.Doc
woz.qjfkLsjdfkLaf.cn/98982.Doc
wol.qjfkLsjdfkLaf.cn/44607.Doc
wol.qjfkLsjdfkLaf.cn/85592.Doc
wol.qjfkLsjdfkLaf.cn/49097.Doc
wol.qjfkLsjdfkLaf.cn/48120.Doc
wol.qjfkLsjdfkLaf.cn/99460.Doc
wol.qjfkLsjdfkLaf.cn/43332.Doc
wol.qjfkLsjdfkLaf.cn/92112.Doc
wol.qjfkLsjdfkLaf.cn/50089.Doc
wol.qjfkLsjdfkLaf.cn/30506.Doc
wol.qjfkLsjdfkLaf.cn/42829.Doc
wok.qjfkLsjdfkLaf.cn/95507.Doc
wok.qjfkLsjdfkLaf.cn/62971.Doc
wok.qjfkLsjdfkLaf.cn/02846.Doc
wok.qjfkLsjdfkLaf.cn/42710.Doc
wok.qjfkLsjdfkLaf.cn/47364.Doc
wok.qjfkLsjdfkLaf.cn/17119.Doc
wok.qjfkLsjdfkLaf.cn/38825.Doc
wok.qjfkLsjdfkLaf.cn/77262.Doc
wok.qjfkLsjdfkLaf.cn/41277.Doc
wok.qjfkLsjdfkLaf.cn/51881.Doc
woj.qjfkLsjdfkLaf.cn/66507.Doc
woj.qjfkLsjdfkLaf.cn/49284.Doc
woj.qjfkLsjdfkLaf.cn/25104.Doc
woj.qjfkLsjdfkLaf.cn/30580.Doc
woj.qjfkLsjdfkLaf.cn/26155.Doc
woj.qjfkLsjdfkLaf.cn/98700.Doc
woj.qjfkLsjdfkLaf.cn/63464.Doc
woj.qjfkLsjdfkLaf.cn/48386.Doc
woj.qjfkLsjdfkLaf.cn/56945.Doc
woj.qjfkLsjdfkLaf.cn/98262.Doc
woh.qjfkLsjdfkLaf.cn/42848.Doc
woh.qjfkLsjdfkLaf.cn/89083.Doc
woh.qjfkLsjdfkLaf.cn/54435.Doc
woh.qjfkLsjdfkLaf.cn/91503.Doc
woh.qjfkLsjdfkLaf.cn/04725.Doc
woh.qjfkLsjdfkLaf.cn/30894.Doc
woh.qjfkLsjdfkLaf.cn/58170.Doc
woh.qjfkLsjdfkLaf.cn/57028.Doc
woh.qjfkLsjdfkLaf.cn/76572.Doc
woh.qjfkLsjdfkLaf.cn/89580.Doc
wog.qjfkLsjdfkLaf.cn/12393.Doc
wog.qjfkLsjdfkLaf.cn/24878.Doc
wog.qjfkLsjdfkLaf.cn/52078.Doc
wog.qjfkLsjdfkLaf.cn/57237.Doc
wog.qjfkLsjdfkLaf.cn/04514.Doc
wog.qjfkLsjdfkLaf.cn/17019.Doc
wog.qjfkLsjdfkLaf.cn/93414.Doc
wog.qjfkLsjdfkLaf.cn/25009.Doc
wog.qjfkLsjdfkLaf.cn/56059.Doc
wog.qjfkLsjdfkLaf.cn/55798.Doc
wof.qjfkLsjdfkLaf.cn/94975.Doc
wof.qjfkLsjdfkLaf.cn/48426.Doc
wof.qjfkLsjdfkLaf.cn/26622.Doc
wof.qjfkLsjdfkLaf.cn/88446.Doc
wof.qjfkLsjdfkLaf.cn/60024.Doc
wof.qjfkLsjdfkLaf.cn/64684.Doc
wof.qjfkLsjdfkLaf.cn/48602.Doc
wof.qjfkLsjdfkLaf.cn/40668.Doc
wof.qjfkLsjdfkLaf.cn/28068.Doc
wof.qjfkLsjdfkLaf.cn/00242.Doc
wod.qjfkLsjdfkLaf.cn/40262.Doc
wod.qjfkLsjdfkLaf.cn/62000.Doc
wod.qjfkLsjdfkLaf.cn/84424.Doc
wod.qjfkLsjdfkLaf.cn/20448.Doc
wod.qjfkLsjdfkLaf.cn/86060.Doc
wod.qjfkLsjdfkLaf.cn/26228.Doc
wod.qjfkLsjdfkLaf.cn/20084.Doc
wod.qjfkLsjdfkLaf.cn/26242.Doc
wod.qjfkLsjdfkLaf.cn/02820.Doc
wod.qjfkLsjdfkLaf.cn/06460.Doc
wos.qjfkLsjdfkLaf.cn/20664.Doc
wos.qjfkLsjdfkLaf.cn/08806.Doc
wos.qjfkLsjdfkLaf.cn/20800.Doc
wos.qjfkLsjdfkLaf.cn/42480.Doc
wos.qjfkLsjdfkLaf.cn/46062.Doc
wos.qjfkLsjdfkLaf.cn/20404.Doc
wos.qjfkLsjdfkLaf.cn/06626.Doc
wos.qjfkLsjdfkLaf.cn/86608.Doc
wos.qjfkLsjdfkLaf.cn/28682.Doc
wos.qjfkLsjdfkLaf.cn/24446.Doc
woa.qjfkLsjdfkLaf.cn/06266.Doc
woa.qjfkLsjdfkLaf.cn/66042.Doc
woa.qjfkLsjdfkLaf.cn/44080.Doc
woa.qjfkLsjdfkLaf.cn/46060.Doc
woa.qjfkLsjdfkLaf.cn/66048.Doc
woa.qjfkLsjdfkLaf.cn/06062.Doc
woa.qjfkLsjdfkLaf.cn/24220.Doc
woa.qjfkLsjdfkLaf.cn/80004.Doc
woa.qjfkLsjdfkLaf.cn/88088.Doc
woa.qjfkLsjdfkLaf.cn/06066.Doc
wop.qjfkLsjdfkLaf.cn/64062.Doc
wop.qjfkLsjdfkLaf.cn/84204.Doc
wop.qjfkLsjdfkLaf.cn/20608.Doc
wop.qjfkLsjdfkLaf.cn/84004.Doc
wop.qjfkLsjdfkLaf.cn/02408.Doc
wop.qjfkLsjdfkLaf.cn/08404.Doc
wop.qjfkLsjdfkLaf.cn/06664.Doc
wop.qjfkLsjdfkLaf.cn/42000.Doc
wop.qjfkLsjdfkLaf.cn/64022.Doc
wop.qjfkLsjdfkLaf.cn/06644.Doc
woo.qjfkLsjdfkLaf.cn/86002.Doc
woo.qjfkLsjdfkLaf.cn/26808.Doc
woo.qjfkLsjdfkLaf.cn/46486.Doc
woo.qjfkLsjdfkLaf.cn/82660.Doc
woo.qjfkLsjdfkLaf.cn/42828.Doc
woo.qjfkLsjdfkLaf.cn/00826.Doc
woo.qjfkLsjdfkLaf.cn/46486.Doc
woo.qjfkLsjdfkLaf.cn/02400.Doc
woo.qjfkLsjdfkLaf.cn/46862.Doc
woo.qjfkLsjdfkLaf.cn/22822.Doc
woi.qjfkLsjdfkLaf.cn/88264.Doc
woi.qjfkLsjdfkLaf.cn/26460.Doc
woi.qjfkLsjdfkLaf.cn/08884.Doc
woi.qjfkLsjdfkLaf.cn/26044.Doc
woi.qjfkLsjdfkLaf.cn/62266.Doc
woi.qjfkLsjdfkLaf.cn/60886.Doc
woi.qjfkLsjdfkLaf.cn/24422.Doc
woi.qjfkLsjdfkLaf.cn/00682.Doc
woi.qjfkLsjdfkLaf.cn/42440.Doc
woi.qjfkLsjdfkLaf.cn/86804.Doc
wou.qjfkLsjdfkLaf.cn/06268.Doc
wou.qjfkLsjdfkLaf.cn/48864.Doc
wou.qjfkLsjdfkLaf.cn/64600.Doc
wou.qjfkLsjdfkLaf.cn/68200.Doc
wou.qjfkLsjdfkLaf.cn/24462.Doc
wou.qjfkLsjdfkLaf.cn/68864.Doc
wou.qjfkLsjdfkLaf.cn/66866.Doc
wou.qjfkLsjdfkLaf.cn/42464.Doc
wou.qjfkLsjdfkLaf.cn/48886.Doc
wou.qjfkLsjdfkLaf.cn/82002.Doc
woy.qjfkLsjdfkLaf.cn/80688.Doc
woy.qjfkLsjdfkLaf.cn/28800.Doc
woy.qjfkLsjdfkLaf.cn/66866.Doc
woy.qjfkLsjdfkLaf.cn/46422.Doc
woy.qjfkLsjdfkLaf.cn/88426.Doc
woy.qjfkLsjdfkLaf.cn/62800.Doc
woy.qjfkLsjdfkLaf.cn/22682.Doc
woy.qjfkLsjdfkLaf.cn/64820.Doc
woy.qjfkLsjdfkLaf.cn/02820.Doc
woy.qjfkLsjdfkLaf.cn/40282.Doc
wot.qjfkLsjdfkLaf.cn/82642.Doc
wot.qjfkLsjdfkLaf.cn/64082.Doc
wot.qjfkLsjdfkLaf.cn/28240.Doc
wot.qjfkLsjdfkLaf.cn/80824.Doc
wot.qjfkLsjdfkLaf.cn/04624.Doc
wot.qjfkLsjdfkLaf.cn/44842.Doc
wot.qjfkLsjdfkLaf.cn/06282.Doc
wot.qjfkLsjdfkLaf.cn/48280.Doc
wot.qjfkLsjdfkLaf.cn/51151.Doc
wot.qjfkLsjdfkLaf.cn/00466.Doc
wor.qjfkLsjdfkLaf.cn/00842.Doc
wor.qjfkLsjdfkLaf.cn/88028.Doc
wor.qjfkLsjdfkLaf.cn/02240.Doc
wor.qjfkLsjdfkLaf.cn/68024.Doc
wor.qjfkLsjdfkLaf.cn/64400.Doc
wor.qjfkLsjdfkLaf.cn/48224.Doc
wor.qjfkLsjdfkLaf.cn/80066.Doc
wor.qjfkLsjdfkLaf.cn/00820.Doc
wor.qjfkLsjdfkLaf.cn/93557.Doc
wor.qjfkLsjdfkLaf.cn/24882.Doc
woe.qjfkLsjdfkLaf.cn/46448.Doc
woe.qjfkLsjdfkLaf.cn/44628.Doc
woe.qjfkLsjdfkLaf.cn/06002.Doc
woe.qjfkLsjdfkLaf.cn/24080.Doc
woe.qjfkLsjdfkLaf.cn/20264.Doc
woe.qjfkLsjdfkLaf.cn/04088.Doc
woe.qjfkLsjdfkLaf.cn/04204.Doc
woe.qjfkLsjdfkLaf.cn/40824.Doc
woe.qjfkLsjdfkLaf.cn/86486.Doc
woe.qjfkLsjdfkLaf.cn/26466.Doc
wow.qjfkLsjdfkLaf.cn/15517.Doc
wow.qjfkLsjdfkLaf.cn/46462.Doc
wow.qjfkLsjdfkLaf.cn/62640.Doc
wow.qjfkLsjdfkLaf.cn/06088.Doc
wow.qjfkLsjdfkLaf.cn/06668.Doc
wow.qjfkLsjdfkLaf.cn/44044.Doc
wow.qjfkLsjdfkLaf.cn/62002.Doc
wow.qjfkLsjdfkLaf.cn/22284.Doc
wow.qjfkLsjdfkLaf.cn/62288.Doc
wow.qjfkLsjdfkLaf.cn/04240.Doc
woq.qjfkLsjdfkLaf.cn/28266.Doc
woq.qjfkLsjdfkLaf.cn/39375.Doc
woq.qjfkLsjdfkLaf.cn/60820.Doc
woq.qjfkLsjdfkLaf.cn/82004.Doc
woq.qjfkLsjdfkLaf.cn/28480.Doc
woq.qjfkLsjdfkLaf.cn/60828.Doc
woq.qjfkLsjdfkLaf.cn/08866.Doc
woq.qjfkLsjdfkLaf.cn/86862.Doc
woq.qjfkLsjdfkLaf.cn/04280.Doc
woq.qjfkLsjdfkLaf.cn/04442.Doc
wim.qjfkLsjdfkLaf.cn/62448.Doc
wim.qjfkLsjdfkLaf.cn/62240.Doc
wim.qjfkLsjdfkLaf.cn/37151.Doc
wim.qjfkLsjdfkLaf.cn/40402.Doc
wim.qjfkLsjdfkLaf.cn/84226.Doc
wim.qjfkLsjdfkLaf.cn/68620.Doc
wim.qjfkLsjdfkLaf.cn/46640.Doc
wim.qjfkLsjdfkLaf.cn/26624.Doc
wim.qjfkLsjdfkLaf.cn/88000.Doc
wim.qjfkLsjdfkLaf.cn/42080.Doc
win.qjfkLsjdfkLaf.cn/46468.Doc
win.qjfkLsjdfkLaf.cn/46826.Doc
win.qjfkLsjdfkLaf.cn/02488.Doc
win.qjfkLsjdfkLaf.cn/00688.Doc
win.qjfkLsjdfkLaf.cn/48240.Doc
win.qjfkLsjdfkLaf.cn/64446.Doc
win.qjfkLsjdfkLaf.cn/26800.Doc
win.qjfkLsjdfkLaf.cn/08084.Doc
win.qjfkLsjdfkLaf.cn/44220.Doc
win.qjfkLsjdfkLaf.cn/40246.Doc
wib.qjfkLsjdfkLaf.cn/84088.Doc
wib.qjfkLsjdfkLaf.cn/46026.Doc
wib.qjfkLsjdfkLaf.cn/22802.Doc
wib.qjfkLsjdfkLaf.cn/86008.Doc
wib.qjfkLsjdfkLaf.cn/06086.Doc
wib.qjfkLsjdfkLaf.cn/60044.Doc
wib.qjfkLsjdfkLaf.cn/06626.Doc
wib.qjfkLsjdfkLaf.cn/48628.Doc
wib.qjfkLsjdfkLaf.cn/48204.Doc
wib.qjfkLsjdfkLaf.cn/73371.Doc
wiv.qjfkLsjdfkLaf.cn/64222.Doc
wiv.qjfkLsjdfkLaf.cn/82024.Doc
wiv.qjfkLsjdfkLaf.cn/26004.Doc
wiv.qjfkLsjdfkLaf.cn/93157.Doc
wiv.qjfkLsjdfkLaf.cn/62444.Doc
wiv.qjfkLsjdfkLaf.cn/77179.Doc
wiv.qjfkLsjdfkLaf.cn/57155.Doc
wiv.qjfkLsjdfkLaf.cn/68480.Doc
wiv.qjfkLsjdfkLaf.cn/86462.Doc
wiv.qjfkLsjdfkLaf.cn/20884.Doc
wic.qjfkLsjdfkLaf.cn/80446.Doc
wic.qjfkLsjdfkLaf.cn/59913.Doc
wic.qjfkLsjdfkLaf.cn/04268.Doc
wic.qjfkLsjdfkLaf.cn/06862.Doc
wic.qjfkLsjdfkLaf.cn/40284.Doc
wic.qjfkLsjdfkLaf.cn/62866.Doc
wic.qjfkLsjdfkLaf.cn/40644.Doc
wic.qjfkLsjdfkLaf.cn/86026.Doc
wic.qjfkLsjdfkLaf.cn/02466.Doc
wic.qjfkLsjdfkLaf.cn/42886.Doc
wix.qjfkLsjdfkLaf.cn/62606.Doc
wix.qjfkLsjdfkLaf.cn/48624.Doc
wix.qjfkLsjdfkLaf.cn/82648.Doc
wix.qjfkLsjdfkLaf.cn/08264.Doc
wix.qjfkLsjdfkLaf.cn/63517.Doc
wix.qjfkLsjdfkLaf.cn/77537.Doc
wix.qjfkLsjdfkLaf.cn/24466.Doc
wix.qjfkLsjdfkLaf.cn/99119.Doc
wix.qjfkLsjdfkLaf.cn/00468.Doc
wix.qjfkLsjdfkLaf.cn/86264.Doc
wiz.qjfkLsjdfkLaf.cn/46060.Doc
wiz.qjfkLsjdfkLaf.cn/59373.Doc
wiz.qjfkLsjdfkLaf.cn/93991.Doc
wiz.qjfkLsjdfkLaf.cn/42642.Doc
wiz.qjfkLsjdfkLaf.cn/28426.Doc
wiz.qjfkLsjdfkLaf.cn/42204.Doc
wiz.qjfkLsjdfkLaf.cn/04860.Doc
wiz.qjfkLsjdfkLaf.cn/08424.Doc
wiz.qjfkLsjdfkLaf.cn/44400.Doc
wiz.qjfkLsjdfkLaf.cn/66806.Doc
wil.qjfkLsjdfkLaf.cn/20068.Doc
wil.qjfkLsjdfkLaf.cn/28848.Doc
wil.qjfkLsjdfkLaf.cn/48668.Doc
wil.qjfkLsjdfkLaf.cn/64886.Doc
wil.qjfkLsjdfkLaf.cn/04020.Doc
wil.qjfkLsjdfkLaf.cn/28848.Doc
wil.qjfkLsjdfkLaf.cn/04484.Doc
wil.qjfkLsjdfkLaf.cn/66082.Doc
wil.qjfkLsjdfkLaf.cn/06226.Doc
wil.qjfkLsjdfkLaf.cn/28000.Doc
wik.qjfkLsjdfkLaf.cn/20264.Doc
wik.qjfkLsjdfkLaf.cn/20668.Doc
wik.qjfkLsjdfkLaf.cn/40864.Doc
wik.qjfkLsjdfkLaf.cn/40048.Doc
wik.qjfkLsjdfkLaf.cn/80448.Doc
wik.qjfkLsjdfkLaf.cn/66460.Doc
wik.qjfkLsjdfkLaf.cn/48646.Doc
wik.qjfkLsjdfkLaf.cn/28842.Doc
wik.qjfkLsjdfkLaf.cn/04828.Doc
wik.qjfkLsjdfkLaf.cn/28422.Doc
wij.qjfkLsjdfkLaf.cn/55939.Doc
wij.qjfkLsjdfkLaf.cn/00820.Doc
wij.qjfkLsjdfkLaf.cn/04688.Doc
wij.qjfkLsjdfkLaf.cn/44426.Doc
wij.qjfkLsjdfkLaf.cn/22804.Doc
wij.qjfkLsjdfkLaf.cn/40204.Doc
wij.qjfkLsjdfkLaf.cn/13931.Doc
wij.qjfkLsjdfkLaf.cn/84266.Doc
wij.qjfkLsjdfkLaf.cn/86642.Doc
wij.qjfkLsjdfkLaf.cn/28648.Doc
wih.qjfkLsjdfkLaf.cn/60442.Doc
wih.qjfkLsjdfkLaf.cn/60426.Doc
wih.qjfkLsjdfkLaf.cn/88644.Doc
wih.qjfkLsjdfkLaf.cn/40244.Doc
wih.qjfkLsjdfkLaf.cn/75757.Doc
wih.qjfkLsjdfkLaf.cn/22448.Doc
wih.qjfkLsjdfkLaf.cn/82862.Doc
wih.qjfkLsjdfkLaf.cn/04688.Doc
wih.qjfkLsjdfkLaf.cn/08048.Doc
wih.qjfkLsjdfkLaf.cn/11959.Doc
wig.qjfkLsjdfkLaf.cn/33375.Doc
wig.qjfkLsjdfkLaf.cn/02024.Doc
wig.qjfkLsjdfkLaf.cn/86048.Doc
wig.qjfkLsjdfkLaf.cn/62240.Doc
wig.qjfkLsjdfkLaf.cn/82226.Doc
wig.qjfkLsjdfkLaf.cn/06202.Doc
wig.qjfkLsjdfkLaf.cn/99977.Doc
wig.qjfkLsjdfkLaf.cn/46280.Doc
wig.qjfkLsjdfkLaf.cn/42602.Doc
wig.qjfkLsjdfkLaf.cn/24824.Doc
wif.qjfkLsjdfkLaf.cn/55353.Doc
wif.qjfkLsjdfkLaf.cn/02264.Doc
wif.qjfkLsjdfkLaf.cn/80200.Doc
wif.qjfkLsjdfkLaf.cn/91393.Doc
wif.qjfkLsjdfkLaf.cn/20486.Doc
wif.qjfkLsjdfkLaf.cn/80606.Doc
wif.qjfkLsjdfkLaf.cn/62484.Doc
wif.qjfkLsjdfkLaf.cn/40446.Doc
wif.qjfkLsjdfkLaf.cn/64868.Doc
wif.qjfkLsjdfkLaf.cn/00008.Doc
wid.qjfkLsjdfkLaf.cn/28426.Doc
wid.qjfkLsjdfkLaf.cn/48886.Doc
wid.qjfkLsjdfkLaf.cn/79733.Doc
wid.qjfkLsjdfkLaf.cn/40000.Doc
wid.qjfkLsjdfkLaf.cn/88646.Doc
wid.qjfkLsjdfkLaf.cn/40484.Doc
wid.qjfkLsjdfkLaf.cn/00608.Doc
wid.qjfkLsjdfkLaf.cn/84008.Doc
wid.qjfkLsjdfkLaf.cn/20622.Doc
wid.qjfkLsjdfkLaf.cn/82026.Doc
wis.qjfkLsjdfkLaf.cn/68086.Doc
wis.qjfkLsjdfkLaf.cn/68668.Doc
wis.qjfkLsjdfkLaf.cn/40244.Doc
wis.qjfkLsjdfkLaf.cn/62868.Doc
wis.qjfkLsjdfkLaf.cn/68886.Doc
wis.qjfkLsjdfkLaf.cn/20828.Doc
wis.qjfkLsjdfkLaf.cn/20220.Doc
wis.qjfkLsjdfkLaf.cn/44444.Doc
wis.qjfkLsjdfkLaf.cn/04468.Doc
wis.qjfkLsjdfkLaf.cn/51137.Doc
