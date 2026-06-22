诩谐诎讲颈


============================================
Hypothesis 属性基测试（Property-Based Testing）
============================================

传统测试用例是"基于示例"的——你手动编写输入和期望输出。
属性基测试则描述代码应该满足的"属性"（不变式），
由框架自动生成大量测试数据，发现隐藏的边界情况。

一、安装与基础
--------------

# pip install hypothesis


二、代码示例
------------

from hypothesis import given, assume, example, find, settings
from hypothesis import strategies as st
import pytest


# =============================================
# 1. @given 基本用法：自动生成测试数据
# =============================================

@given(st.integers())
def test_integer_identity(x):
    """测试整数自反性：任何整数都应等于自身"""
    print(f"\n测试整数: {x}")
    assert x == x


@given(st.text())
def test_text_operations(text):
    """测试文本操作的通用属性"""
    print(f"\n测试文本: '{text}' (长度: {len(text)})")

    # 属性1：反转两次等于原字符串
    assert text[::-1][::-1] == text

    # 属性2：字符串长度非负
    assert len(text) >= 0


# =============================================
# 2. 常用策略（Strategies）
# =============================================

@given(st.lists(st.integers(min_value=-100, max_value=100)))
def test_list_sorting(lst):
    """测试列表排序的属性"""
    print(f"\n原始列表: {lst[:5]}{'...' if len(lst) > 5 else ''}")

    sorted_lst = sorted(lst)

    # 属性1：排序后长度不变
    assert len(sorted_lst) == len(lst)

    # 属性2：排序后第一个元素是最小值（如果列表非空）
    if lst:
        assert sorted_lst[0] == min(lst)
        assert sorted_lst[-1] == max(lst)

    # 属性3：排序后列表是非递减的
    for i in range(len(sorted_lst) - 1):
        assert sorted_lst[i] <= sorted_lst[i + 1]


@given(st.dictionaries(
    keys=st.text(min_size=1, max_size=10),
    values=st.integers(min_value=0, max_value=10000),
    min_size=1,
    max_size=10
))
def test_dictionary_properties(d):
    """测试字典的属性"""
    print(f"\n字典: {d}")

    # 属性1：键值对数量正确
    items = list(d.items())
    assert len(items) == len(d)

    # 属性2：所有键都在字典中
    for key in d.keys():
        assert key in d

    # 属性3：pop 后键不再存在
    if d:
        key, value = d.popitem()
        assert key not in d


@given(st.floats(allow_nan=False, allow_infinity=False))
def test_float_properties(x):
    """测试浮点数属性，排除 NaN 和 Infinity"""
    print(f"\n浮点数: {x}")

    # 属性：任何数乘以 1 等于自身
    assert x * 1.0 == x

    # 属性：任何数加 0 等于自身
    assert x + 0.0 == x


# =============================================
# 3. @example：指定具体测试用例
# =============================================

@given(st.lists(st.integers()))
@example([])       # 空列表边界情况
@example([0])      # 单元素列表
@example([-1, 0, 1])  # 包含负数、零、正数
def test_list_with_examples(lst):
    """@example 确保边界情况总是被测试到"""
    print(f"\n测试列表: {lst}")
    # 测试 max/min 对空列表的处理
    if not lst:
        with pytest.raises(ValueError):
            max(lst)
    else:
        assert max(lst) >= min(lst)


# =============================================
# 4. assume()：过滤无效输入
# =============================================

@given(st.integers())
def test_division_property(x):
    """测试除法属性，过滤掉除数为零的情况"""
    # 使用 assume 排除 x=0，而不是让测试崩溃
    assume(x != 0)

    result = 100 / x
    print(f"\n100 / {x} = {result}")
    assert isinstance(result, (int, float))


@given(st.text())
def test_valid_email_format(email):
    """测试邮件地址格式，过滤无效输入"""
    # 假设邮件包含 @ 符号
    assume("@" in email)
    assume(len(email) >= 5)

    # 提取用户名和域名
    username, domain = email.split("@", 1)
    print(f"\n用户名: {username}, 域名: {domain}")
    assert len(username) > 0
    assert "." in domain


# =============================================
# 5. strategies.composite：自定义策略
# =============================================

@st.composite
def user_data_strategy(draw):
    """自定义策略：生成模拟用户数据"""
    # draw() 从其他策略中抽取具体值
    username = draw(st.text(min_size=3, max_size=20, alphabet="abcdefghijklmnopqrstuvwxyz_"))
    age = draw(st.integers(min_value=18, max_value=120))
    email = draw(st.emails())

    # 构造用户数据字典
    return {
        "username": username,
        "age": age,
        "email": email,
        "is_active": draw(st.booleans()),
    }


@given(user_data_strategy())
def test_user_data_validation(user):
    """测试自定义策略生成的用户数据"""
    print(f"\n用户数据: {user}")

    # 验证用户名格式
    assert len(user["username"]) >= 3
    assert all(c.isalnum() or c == "_" for c in user["username"])

    # 验证年龄范围
    assert 18 <= user["age"] <= 120

    # 验证邮箱格式
    assert "@" in user["email"]
    print(f"用户 {user['username']} 数据验证通过")


# =============================================
# 6. find() 查找反例
# =============================================

def buggy_sort(lst):
    """一个有 bug 的排序函数（无法正确处理重复元素）"""
    if len(lst) <= 1:
        return lst
    # 故意引入 bug：使用 set 去重
    unique = list(set(lst))
    return sorted(unique)


def test_find_counterexample():
    """使用 find() 找到 buggy_sort 的反例"""
    def is_sorted(lst):
        """检查列表是否已排序的辅助函数"""
        return all(lst[i] <= lst[i + 1] for i in range(len(lst) - 1))

    # find() 会搜索使条件成立的输入
    # 这里寻找使 buggy_sort 输出与输入长度不同的输入
    def length_changed(lst):
        sorted_lst = buggy_sort(lst)
        return len(sorted_lst) != len(lst)

    try:
        # 查找反例：输入和排序后长度不同
        counterexample = find(
            st.lists(st.integers(min_value=0, max_value=10), min_size=2),
            length_changed
        )
        print(f"\n找到反例！输入: {counterexample}")
        print(f"排序前长度: {len(counterexample)}")
        print(f"排序后长度: {len(buggy_sort(counterexample))}")
        print("说明：含有重复元素的列表在排序后丢失了元素")
    except Exception as e:
        print(f"未找到反例: {e}")


# =============================================
# 7. Health Checks 和 Settings
# =============================================

# 缓存的函数，测试时需要注意状态残留
cache = {}


def cached_fibonacci(n):
    """带缓存的斐波那契数列计算"""
    if n < 0:
        raise ValueError("n 必须为非负整数")
    if n in cache:
        return cache[n]
    if n <= 1:
        cache[n] = n
    else:
        cache[n] = cached_fibonacci(n - 1) + cached_fibonacci(n - 2)
    return cache[n]


@given(st.integers(min_value=0, max_value=20))
@settings(max_examples=50, suppress_health_check=list(HealthCheck))  # 抑制健康检查
def test_fibonacci_property(n):
    """测试斐波那契数列的属性"""
    from hypothesis import HealthCheck

    print(f"\nFibonacci({n}) = {cached_fibonacci(n)}")

    # 属性：F(n) >= n 对于 n >= 3 成立
    if n >= 3:
        assert cached_fibonacci(n) >= n

    # 属性：F(n) = F(n-1) + F(n-2) 对于 n >= 2 成立
    if n >= 2:
        assert cached_fibonacci(n) == cached_fibonacci(n - 1) + cached_fibonacci(n - 2)


# =============================================
# 8. 测试边缘情况的自动发现
# =============================================

@given(st.lists(st.integers()))
@settings(max_examples=200)
def test_auto_edge_cases(lst):
    """Hypothesis 自动发现边界情况"""
    # 对列表反转两次
    reversed_twice = lst[::-1][::-1]
    assert reversed_twice == lst
    print(f"\n列表长度: {len(lst)}, 反转两次验证通过")


# =============================================
# 9. 基于属性的测试 vs 传统参数化测试
# =============================================

# 传统参数化测试（手动列举）
@pytest.mark.parametrize("a, b", [
    (1, 2),
    (0, 0),
    (-1, 1),
    (100, -50),
])
def test_traditional_addition(a, b):
    """传统参数化测试：只能测试手动列举的用例"""
    assert a + b - b == a


# Hypothesis 属性测试（自动生成）
@given(st.integers(), st.integers())
def test_hypothesis_addition(a, b):
    """Hypothesis 测试：描述属性，自动生成大量数据"""
    # 加法的一个属性：a + b - b 应该永远等于 a
    assert a + b - b == a
    # 另一个属性：加法交换律
    assert a + b == b + a


# =============================================
# 10. 测试自定义数据结构的属性
# =============================================

class Stack:
    """一个简单的栈数据结构"""

    def __init__(self):
        self._items = []

    def push(self, item):
        self._items.append(item)

    def pop(self):
        if not self._items:
            raise IndexError("从空栈中弹出")
        return self._items.pop()

    def peek(self):
        if not self._items:
            raise IndexError("从空栈中查看")
        return self._items[-1]

    def is_empty(self):
        return len(self._items) == 0

    def size(self):
        return len(self._items)


@given(st.lists(st.integers()))
def test_stack_properties(items):
    """测试栈数据结构的属性"""
    stack = Stack()

    # 属性1：新栈是空的
    assert stack.is_empty()
    assert stack.size() == 0

    # 将所有元素入栈
    for item in items:
        stack.push(item)

    # 属性2：入栈后 size 正确
    assert stack.size() == len(items)

    # 属性3：后进先出（LIFO）
    for expected in reversed(items):
        assert stack.pop() == expected

    # 属性4：全部出栈后栈再次为空
    assert stack.is_empty()
    assert stack.size() == 0

羌静逗颇娜匚枷辉然坝良晨哪颂胁

dvy.nfsid.cn/486020.Doc
dvy.nfsid.cn/062042.Doc
dvy.nfsid.cn/262860.Doc
dvy.nfsid.cn/739731.Doc
dvy.nfsid.cn/624288.Doc
dvy.nfsid.cn/488488.Doc
dvt.nfsid.cn/420824.Doc
dvt.nfsid.cn/046204.Doc
dvt.nfsid.cn/044800.Doc
dvt.nfsid.cn/660084.Doc
dvt.nfsid.cn/319511.Doc
dvt.nfsid.cn/022604.Doc
dvt.nfsid.cn/868800.Doc
dvt.nfsid.cn/759339.Doc
dvt.nfsid.cn/626846.Doc
dvt.nfsid.cn/282286.Doc
dvr.nfsid.cn/860420.Doc
dvr.nfsid.cn/260268.Doc
dvr.nfsid.cn/008206.Doc
dvr.nfsid.cn/882880.Doc
dvr.nfsid.cn/995519.Doc
dvr.nfsid.cn/177197.Doc
dvr.nfsid.cn/206804.Doc
dvr.nfsid.cn/953933.Doc
dvr.nfsid.cn/080444.Doc
dvr.nfsid.cn/822026.Doc
dve.nfsid.cn/379333.Doc
dve.nfsid.cn/640060.Doc
dve.nfsid.cn/048042.Doc
dve.nfsid.cn/933795.Doc
dve.nfsid.cn/802684.Doc
dve.nfsid.cn/042260.Doc
dve.nfsid.cn/731393.Doc
dve.nfsid.cn/282286.Doc
dve.nfsid.cn/826048.Doc
dve.nfsid.cn/824408.Doc
dvw.nfsid.cn/040226.Doc
dvw.nfsid.cn/866666.Doc
dvw.nfsid.cn/840602.Doc
dvw.nfsid.cn/806068.Doc
dvw.nfsid.cn/464266.Doc
dvw.nfsid.cn/482284.Doc
dvw.nfsid.cn/806828.Doc
dvw.nfsid.cn/060206.Doc
dvw.nfsid.cn/882228.Doc
dvw.nfsid.cn/662444.Doc
dvq.nfsid.cn/860000.Doc
dvq.nfsid.cn/644068.Doc
dvq.nfsid.cn/408622.Doc
dvq.nfsid.cn/400246.Doc
dvq.nfsid.cn/688220.Doc
dvq.nfsid.cn/806066.Doc
dvq.nfsid.cn/268482.Doc
dvq.nfsid.cn/460440.Doc
dvq.nfsid.cn/082626.Doc
dvq.nfsid.cn/282064.Doc
dcm.nfsid.cn/424824.Doc
dcm.nfsid.cn/066086.Doc
dcm.nfsid.cn/204084.Doc
dcm.nfsid.cn/428862.Doc
dcm.nfsid.cn/400424.Doc
dcm.nfsid.cn/220422.Doc
dcm.nfsid.cn/484006.Doc
dcm.nfsid.cn/068600.Doc
dcm.nfsid.cn/666406.Doc
dcm.nfsid.cn/008664.Doc
dcn.nfsid.cn/915557.Doc
dcn.nfsid.cn/828600.Doc
dcn.nfsid.cn/082882.Doc
dcn.nfsid.cn/228204.Doc
dcn.nfsid.cn/246686.Doc
dcn.nfsid.cn/802220.Doc
dcn.nfsid.cn/022660.Doc
dcn.nfsid.cn/402000.Doc
dcn.nfsid.cn/151351.Doc
dcn.nfsid.cn/882028.Doc
dcb.nfsid.cn/686806.Doc
dcb.nfsid.cn/680662.Doc
dcb.nfsid.cn/008026.Doc
dcb.nfsid.cn/802680.Doc
dcb.nfsid.cn/157931.Doc
dcb.nfsid.cn/004646.Doc
dcb.nfsid.cn/448844.Doc
dcb.nfsid.cn/173551.Doc
dcb.nfsid.cn/840466.Doc
dcb.nfsid.cn/888404.Doc
dcv.nfsid.cn/222608.Doc
dcv.nfsid.cn/800826.Doc
dcv.nfsid.cn/715535.Doc
dcv.nfsid.cn/284486.Doc
dcv.nfsid.cn/640482.Doc
dcv.nfsid.cn/608040.Doc
dcv.nfsid.cn/602822.Doc
dcv.nfsid.cn/606004.Doc
dcv.nfsid.cn/064464.Doc
dcv.nfsid.cn/822280.Doc
dcc.nfsid.cn/080440.Doc
dcc.nfsid.cn/020684.Doc
dcc.nfsid.cn/682440.Doc
dcc.nfsid.cn/557991.Doc
dcc.nfsid.cn/599199.Doc
dcc.nfsid.cn/222828.Doc
dcc.nfsid.cn/062242.Doc
dcc.nfsid.cn/288486.Doc
dcc.nfsid.cn/531951.Doc
dcc.nfsid.cn/426062.Doc
dcx.nfsid.cn/913157.Doc
dcx.nfsid.cn/400446.Doc
dcx.nfsid.cn/668882.Doc
dcx.nfsid.cn/200848.Doc
dcx.nfsid.cn/604660.Doc
dcx.nfsid.cn/646480.Doc
dcx.nfsid.cn/575991.Doc
dcx.nfsid.cn/028408.Doc
dcx.nfsid.cn/402608.Doc
dcx.nfsid.cn/486808.Doc
dcz.nfsid.cn/684842.Doc
dcz.nfsid.cn/806800.Doc
dcz.nfsid.cn/204208.Doc
dcz.nfsid.cn/379753.Doc
dcz.nfsid.cn/600826.Doc
dcz.nfsid.cn/202600.Doc
dcz.nfsid.cn/400088.Doc
dcz.nfsid.cn/266646.Doc
dcz.nfsid.cn/282686.Doc
dcz.nfsid.cn/557371.Doc
dcl.nfsid.cn/002428.Doc
dcl.nfsid.cn/048002.Doc
dcl.nfsid.cn/848086.Doc
dcl.nfsid.cn/080846.Doc
dcl.nfsid.cn/884880.Doc
dcl.nfsid.cn/662642.Doc
dcl.nfsid.cn/484444.Doc
dcl.nfsid.cn/048804.Doc
dcl.nfsid.cn/282040.Doc
dcl.nfsid.cn/602666.Doc
dck.nfsid.cn/442086.Doc
dck.nfsid.cn/026068.Doc
dck.nfsid.cn/179715.Doc
dck.nfsid.cn/226822.Doc
dck.nfsid.cn/844686.Doc
dck.nfsid.cn/868288.Doc
dck.nfsid.cn/664068.Doc
dck.nfsid.cn/026606.Doc
dck.nfsid.cn/155599.Doc
dck.nfsid.cn/006884.Doc
dcj.nfsid.cn/208680.Doc
dcj.nfsid.cn/484864.Doc
dcj.nfsid.cn/442808.Doc
dcj.nfsid.cn/860448.Doc
dcj.nfsid.cn/480846.Doc
dcj.nfsid.cn/646002.Doc
dcj.nfsid.cn/688484.Doc
dcj.nfsid.cn/264600.Doc
dcj.nfsid.cn/008264.Doc
dcj.nfsid.cn/597359.Doc
dch.nfsid.cn/464860.Doc
dch.nfsid.cn/060220.Doc
dch.nfsid.cn/282426.Doc
dch.nfsid.cn/082020.Doc
dch.nfsid.cn/842686.Doc
dch.nfsid.cn/804228.Doc
dch.nfsid.cn/244442.Doc
dch.nfsid.cn/022426.Doc
dch.nfsid.cn/882222.Doc
dch.nfsid.cn/244064.Doc
dcg.nfsid.cn/848046.Doc
dcg.nfsid.cn/179191.Doc
dcg.nfsid.cn/735753.Doc
dcg.nfsid.cn/022800.Doc
dcg.nfsid.cn/717713.Doc
dcg.nfsid.cn/062682.Doc
dcg.nfsid.cn/739599.Doc
dcg.nfsid.cn/286884.Doc
dcg.nfsid.cn/420686.Doc
dcg.nfsid.cn/626026.Doc
dcf.nfsid.cn/204084.Doc
dcf.nfsid.cn/468048.Doc
dcf.nfsid.cn/084020.Doc
dcf.nfsid.cn/462222.Doc
dcf.nfsid.cn/020064.Doc
dcf.nfsid.cn/288424.Doc
dcf.nfsid.cn/551591.Doc
dcf.nfsid.cn/248844.Doc
dcf.nfsid.cn/088646.Doc
dcf.nfsid.cn/066060.Doc
dcd.nfsid.cn/022262.Doc
dcd.nfsid.cn/999733.Doc
dcd.nfsid.cn/488220.Doc
dcd.nfsid.cn/642804.Doc
dcd.nfsid.cn/280646.Doc
dcd.nfsid.cn/826864.Doc
dcd.nfsid.cn/040646.Doc
dcd.nfsid.cn/062640.Doc
dcd.nfsid.cn/602446.Doc
dcd.nfsid.cn/084860.Doc
dcs.nfsid.cn/086426.Doc
dcs.nfsid.cn/866400.Doc
dcs.nfsid.cn/820466.Doc
dcs.nfsid.cn/975571.Doc
dcs.nfsid.cn/808020.Doc
dcs.nfsid.cn/068262.Doc
dcs.nfsid.cn/284806.Doc
dcs.nfsid.cn/199937.Doc
dcs.nfsid.cn/268082.Doc
dcs.nfsid.cn/842822.Doc
dca.nfsid.cn/248800.Doc
dca.nfsid.cn/953535.Doc
dca.nfsid.cn/428660.Doc
dca.nfsid.cn/933973.Doc
dca.nfsid.cn/064440.Doc
dca.nfsid.cn/064026.Doc
dca.nfsid.cn/680006.Doc
dca.nfsid.cn/084400.Doc
dca.nfsid.cn/246480.Doc
dca.nfsid.cn/444662.Doc
dcp.nfsid.cn/822646.Doc
dcp.nfsid.cn/606824.Doc
dcp.nfsid.cn/802008.Doc
dcp.nfsid.cn/442068.Doc
dcp.nfsid.cn/020286.Doc
dcp.nfsid.cn/280800.Doc
dcp.nfsid.cn/026208.Doc
dcp.nfsid.cn/424004.Doc
dcp.nfsid.cn/911135.Doc
dcp.nfsid.cn/622402.Doc
dco.nfsid.cn/860248.Doc
dco.nfsid.cn/840222.Doc
dco.nfsid.cn/086242.Doc
dco.nfsid.cn/284608.Doc
dco.nfsid.cn/468462.Doc
dco.nfsid.cn/020446.Doc
dco.nfsid.cn/608888.Doc
dco.nfsid.cn/804664.Doc
dco.nfsid.cn/737351.Doc
dco.nfsid.cn/953997.Doc
dci.nfsid.cn/359797.Doc
dci.nfsid.cn/888242.Doc
dci.nfsid.cn/680080.Doc
dci.nfsid.cn/862204.Doc
dci.nfsid.cn/620420.Doc
dci.nfsid.cn/086668.Doc
dci.nfsid.cn/686866.Doc
dci.nfsid.cn/802444.Doc
dci.nfsid.cn/402240.Doc
dci.nfsid.cn/082608.Doc
dcu.nfsid.cn/802428.Doc
dcu.nfsid.cn/442660.Doc
dcu.nfsid.cn/915591.Doc
dcu.nfsid.cn/462246.Doc
dcu.nfsid.cn/082284.Doc
dcu.nfsid.cn/624646.Doc
dcu.nfsid.cn/204248.Doc
dcu.nfsid.cn/466462.Doc
dcu.nfsid.cn/020622.Doc
dcu.nfsid.cn/624406.Doc
dcy.nfsid.cn/420822.Doc
dcy.nfsid.cn/264642.Doc
dcy.nfsid.cn/206004.Doc
dcy.nfsid.cn/515335.Doc
dcy.nfsid.cn/028448.Doc
dcy.nfsid.cn/664886.Doc
dcy.nfsid.cn/048426.Doc
dcy.nfsid.cn/622288.Doc
dcy.nfsid.cn/084622.Doc
dcy.nfsid.cn/842282.Doc
dct.nfsid.cn/404426.Doc
dct.nfsid.cn/884888.Doc
dct.nfsid.cn/226002.Doc
dct.nfsid.cn/826400.Doc
dct.nfsid.cn/022420.Doc
dct.nfsid.cn/448084.Doc
dct.nfsid.cn/119535.Doc
dct.nfsid.cn/204044.Doc
dct.nfsid.cn/628460.Doc
dct.nfsid.cn/862440.Doc
dcr.nfsid.cn/468860.Doc
dcr.nfsid.cn/595739.Doc
dcr.nfsid.cn/468622.Doc
dcr.nfsid.cn/040044.Doc
dcr.nfsid.cn/068060.Doc
dcr.nfsid.cn/408802.Doc
dcr.nfsid.cn/208482.Doc
dcr.nfsid.cn/020868.Doc
dcr.nfsid.cn/666000.Doc
dcr.nfsid.cn/400244.Doc
dce.nfsid.cn/042224.Doc
dce.nfsid.cn/864280.Doc
dce.nfsid.cn/282866.Doc
dce.nfsid.cn/880000.Doc
dce.nfsid.cn/826886.Doc
dce.nfsid.cn/802820.Doc
dce.nfsid.cn/280284.Doc
dce.nfsid.cn/048462.Doc
dce.nfsid.cn/024060.Doc
dce.nfsid.cn/440268.Doc
dcw.irvnp.cn/680228.Doc
dcw.irvnp.cn/884626.Doc
dcw.irvnp.cn/022226.Doc
dcw.irvnp.cn/066064.Doc
dcw.irvnp.cn/046422.Doc
dcw.irvnp.cn/402448.Doc
dcw.irvnp.cn/064426.Doc
dcw.irvnp.cn/244426.Doc
dcw.irvnp.cn/884262.Doc
dcw.irvnp.cn/028466.Doc
dcq.irvnp.cn/555993.Doc
dcq.irvnp.cn/464404.Doc
dcq.irvnp.cn/228800.Doc
dcq.irvnp.cn/575313.Doc
dcq.irvnp.cn/842828.Doc
dcq.irvnp.cn/404600.Doc
dcq.irvnp.cn/468640.Doc
dcq.irvnp.cn/646006.Doc
dcq.irvnp.cn/608228.Doc
dcq.irvnp.cn/648808.Doc
dxm.irvnp.cn/660804.Doc
dxm.irvnp.cn/600646.Doc
dxm.irvnp.cn/268000.Doc
dxm.irvnp.cn/444662.Doc
dxm.irvnp.cn/620626.Doc
dxm.irvnp.cn/335733.Doc
dxm.irvnp.cn/864400.Doc
dxm.irvnp.cn/846444.Doc
dxm.irvnp.cn/068028.Doc
dxm.irvnp.cn/022886.Doc
dxn.irvnp.cn/680020.Doc
dxn.irvnp.cn/424424.Doc
dxn.irvnp.cn/824860.Doc
dxn.irvnp.cn/660642.Doc
dxn.irvnp.cn/484426.Doc
dxn.irvnp.cn/206684.Doc
dxn.irvnp.cn/262884.Doc
dxn.irvnp.cn/268426.Doc
dxn.irvnp.cn/828482.Doc
dxn.irvnp.cn/046644.Doc
dxb.irvnp.cn/911751.Doc
dxb.irvnp.cn/002888.Doc
dxb.irvnp.cn/248066.Doc
dxb.irvnp.cn/313935.Doc
dxb.irvnp.cn/026824.Doc
dxb.irvnp.cn/468008.Doc
dxb.irvnp.cn/397573.Doc
dxb.irvnp.cn/042484.Doc
dxb.irvnp.cn/648888.Doc
dxb.irvnp.cn/848846.Doc
dxv.irvnp.cn/400604.Doc
dxv.irvnp.cn/886882.Doc
dxv.irvnp.cn/553975.Doc
dxv.irvnp.cn/464646.Doc
dxv.irvnp.cn/404686.Doc
dxv.irvnp.cn/404840.Doc
dxv.irvnp.cn/288800.Doc
dxv.irvnp.cn/600040.Doc
dxv.irvnp.cn/408606.Doc
dxv.irvnp.cn/048486.Doc
dxc.irvnp.cn/177573.Doc
dxc.irvnp.cn/404644.Doc
dxc.irvnp.cn/068286.Doc
dxc.irvnp.cn/042664.Doc
dxc.irvnp.cn/884660.Doc
dxc.irvnp.cn/000024.Doc
dxc.irvnp.cn/244688.Doc
dxc.irvnp.cn/402068.Doc
dxc.irvnp.cn/686820.Doc
dxc.irvnp.cn/628020.Doc
dxx.irvnp.cn/208208.Doc
dxx.irvnp.cn/735575.Doc
dxx.irvnp.cn/286006.Doc
dxx.irvnp.cn/97.Doc
dxx.irvnp.cn/426824.Doc
dxx.irvnp.cn/553935.Doc
dxx.irvnp.cn/248080.Doc
dxx.irvnp.cn/220626.Doc
dxx.irvnp.cn/222866.Doc
dxx.irvnp.cn/226444.Doc
dxz.irvnp.cn/997171.Doc
dxz.irvnp.cn/662002.Doc
dxz.irvnp.cn/428404.Doc
dxz.irvnp.cn/444028.Doc
dxz.irvnp.cn/155319.Doc
dxz.irvnp.cn/933717.Doc
dxz.irvnp.cn/440422.Doc
dxz.irvnp.cn/157199.Doc
dxz.irvnp.cn/262248.Doc
dxz.irvnp.cn/933997.Doc
dxl.irvnp.cn/735331.Doc
dxl.irvnp.cn/286626.Doc
dxl.irvnp.cn/208664.Doc
dxl.irvnp.cn/644062.Doc
dxl.irvnp.cn/200264.Doc
dxl.irvnp.cn/066242.Doc
dxl.irvnp.cn/999591.Doc
dxl.irvnp.cn/066044.Doc
dxl.irvnp.cn/113153.Doc
