蹲度虑疗啡


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

坷畏隙殴忍殴捉砍岛萄闹核倒咕晌

wdi.ugioc26.cn/840465.htm
wdi.ugioc26.cn/513955.htm
wdi.ugioc26.cn/519715.htm
wdu.24d38z2.cn/008225.htm
wdu.24d38z2.cn/668485.htm
wdu.24d38z2.cn/244425.htm
wdu.24d38z2.cn/822245.htm
wdu.24d38z2.cn/686425.htm
wdu.24d38z2.cn/997135.htm
wdu.24d38z2.cn/460485.htm
wdu.24d38z2.cn/284005.htm
wdu.24d38z2.cn/804085.htm
wdu.24d38z2.cn/577915.htm
wdy.24d38z2.cn/486085.htm
wdy.24d38z2.cn/935195.htm
wdy.24d38z2.cn/684665.htm
wdy.24d38z2.cn/133595.htm
wdy.24d38z2.cn/717755.htm
wdy.24d38z2.cn/606005.htm
wdy.24d38z2.cn/088805.htm
wdy.24d38z2.cn/408465.htm
wdy.24d38z2.cn/935935.htm
wdy.24d38z2.cn/660645.htm
wdt.24d38z2.cn/599135.htm
wdt.24d38z2.cn/864025.htm
wdt.24d38z2.cn/826025.htm
wdt.24d38z2.cn/402025.htm
wdt.24d38z2.cn/608065.htm
wdt.24d38z2.cn/022865.htm
wdt.24d38z2.cn/359155.htm
wdt.24d38z2.cn/571715.htm
wdt.24d38z2.cn/684825.htm
wdt.24d38z2.cn/042025.htm
wdr.24d38z2.cn/682265.htm
wdr.24d38z2.cn/597175.htm
wdr.24d38z2.cn/064625.htm
wdr.24d38z2.cn/468445.htm
wdr.24d38z2.cn/286265.htm
wdr.24d38z2.cn/597175.htm
wdr.24d38z2.cn/260685.htm
wdr.24d38z2.cn/426025.htm
wdr.24d38z2.cn/951395.htm
wdr.24d38z2.cn/999735.htm
wde.24d38z2.cn/660265.htm
wde.24d38z2.cn/468405.htm
wde.24d38z2.cn/119595.htm
wde.24d38z2.cn/062425.htm
wde.24d38z2.cn/955155.htm
wde.24d38z2.cn/002465.htm
wde.24d38z2.cn/082285.htm
wde.24d38z2.cn/002085.htm
wde.24d38z2.cn/559735.htm
wde.24d38z2.cn/286645.htm
wdw.24d38z2.cn/975935.htm
wdw.24d38z2.cn/882245.htm
wdw.24d38z2.cn/866005.htm
wdw.24d38z2.cn/464425.htm
wdw.24d38z2.cn/557335.htm
wdw.24d38z2.cn/755155.htm
wdw.24d38z2.cn/991995.htm
wdw.24d38z2.cn/884645.htm
wdw.24d38z2.cn/488265.htm
wdw.24d38z2.cn/139115.htm
wdq.24d38z2.cn/026285.htm
wdq.24d38z2.cn/288005.htm
wdq.24d38z2.cn/286625.htm
wdq.24d38z2.cn/979715.htm
wdq.24d38z2.cn/202005.htm
wdq.24d38z2.cn/351535.htm
wdq.24d38z2.cn/820805.htm
wdq.24d38z2.cn/377575.htm
wdq.24d38z2.cn/460085.htm
wdq.24d38z2.cn/991155.htm
wstv.24d38z2.cn/402265.htm
wstv.24d38z2.cn/446605.htm
wstv.24d38z2.cn/602845.htm
wstv.24d38z2.cn/628085.htm
wstv.24d38z2.cn/066885.htm
wstv.24d38z2.cn/355775.htm
wstv.24d38z2.cn/608465.htm
wstv.24d38z2.cn/422025.htm
wstv.24d38z2.cn/424025.htm
wstv.24d38z2.cn/406245.htm
wsn.24d38z2.cn/571535.htm
wsn.24d38z2.cn/648225.htm
wsn.24d38z2.cn/535995.htm
wsn.24d38z2.cn/511955.htm
wsn.24d38z2.cn/779355.htm
wsn.24d38z2.cn/977975.htm
wsn.24d38z2.cn/373755.htm
wsn.24d38z2.cn/339315.htm
wsn.24d38z2.cn/644645.htm
wsn.24d38z2.cn/795955.htm
wsb.24d38z2.cn/408225.htm
wsb.24d38z2.cn/046005.htm
wsb.24d38z2.cn/957915.htm
wsb.24d38z2.cn/866885.htm
wsb.24d38z2.cn/046645.htm
wsb.24d38z2.cn/000005.htm
wsb.24d38z2.cn/660625.htm
wsb.24d38z2.cn/044425.htm
wsb.24d38z2.cn/206685.htm
wsb.24d38z2.cn/737555.htm
wsv.24d38z2.cn/248845.htm
wsv.24d38z2.cn/593395.htm
wsv.24d38z2.cn/888605.htm
wsv.24d38z2.cn/822005.htm
wsv.24d38z2.cn/602465.htm
wsv.24d38z2.cn/846225.htm
wsv.24d38z2.cn/759535.htm
wsv.24d38z2.cn/715935.htm
wsv.24d38z2.cn/848465.htm
wsv.24d38z2.cn/028045.htm
wsc.24d38z2.cn/791375.htm
wsc.24d38z2.cn/008225.htm
wsc.24d38z2.cn/937975.htm
wsc.24d38z2.cn/628025.htm
wsc.24d38z2.cn/686005.htm
wsc.24d38z2.cn/002285.htm
wsc.24d38z2.cn/444065.htm
wsc.24d38z2.cn/428885.htm
wsc.24d38z2.cn/995175.htm
wsc.24d38z2.cn/797575.htm
wsx.24d38z2.cn/664265.htm
wsx.24d38z2.cn/151335.htm
wsx.24d38z2.cn/886045.htm
wsx.24d38z2.cn/608885.htm
wsx.24d38z2.cn/884205.htm
wsx.24d38z2.cn/842865.htm
wsx.24d38z2.cn/179375.htm
wsx.24d38z2.cn/228265.htm
wsx.24d38z2.cn/840265.htm
wsx.24d38z2.cn/135735.htm
wsz.24d38z2.cn/688485.htm
wsz.24d38z2.cn/882825.htm
wsz.24d38z2.cn/842405.htm
wsz.24d38z2.cn/977575.htm
wsz.24d38z2.cn/862825.htm
wsz.24d38z2.cn/175155.htm
wsz.24d38z2.cn/779995.htm
wsz.24d38z2.cn/933955.htm
wsz.24d38z2.cn/402845.htm
wsz.24d38z2.cn/422665.htm
wsl.24d38z2.cn/266205.htm
wsl.24d38z2.cn/771575.htm
wsl.24d38z2.cn/464065.htm
wsl.24d38z2.cn/400205.htm
wsl.24d38z2.cn/739135.htm
wsl.24d38z2.cn/557335.htm
wsl.24d38z2.cn/286445.htm
wsl.24d38z2.cn/317115.htm
wsl.24d38z2.cn/402645.htm
wsl.24d38z2.cn/337355.htm
wsk.24d38z2.cn/399135.htm
wsk.24d38z2.cn/137975.htm
wsk.24d38z2.cn/537535.htm
wsk.24d38z2.cn/884225.htm
wsk.24d38z2.cn/622885.htm
wsk.24d38z2.cn/115955.htm
wsk.24d38z2.cn/642445.htm
wsk.24d38z2.cn/640045.htm
wsk.24d38z2.cn/220665.htm
wsk.24d38z2.cn/264845.htm
wsj.24d38z2.cn/064465.htm
wsj.24d38z2.cn/424665.htm
wsj.24d38z2.cn/248225.htm
wsj.24d38z2.cn/662805.htm
wsj.24d38z2.cn/711195.htm
wsj.24d38z2.cn/579535.htm
wsj.24d38z2.cn/391715.htm
wsj.24d38z2.cn/399135.htm
wsj.24d38z2.cn/775775.htm
wsj.24d38z2.cn/488405.htm
wsh.24d38z2.cn/640285.htm
wsh.24d38z2.cn/333375.htm
wsh.24d38z2.cn/642425.htm
wsh.24d38z2.cn/286445.htm
wsh.24d38z2.cn/248425.htm
wsh.24d38z2.cn/373955.htm
wsh.24d38z2.cn/391715.htm
wsh.24d38z2.cn/591715.htm
wsh.24d38z2.cn/402025.htm
wsh.24d38z2.cn/800225.htm
wsg.24d38z2.cn/682845.htm
wsg.24d38z2.cn/444625.htm
wsg.24d38z2.cn/797555.htm
wsg.24d38z2.cn/046065.htm
wsg.24d38z2.cn/626645.htm
wsg.24d38z2.cn/004425.htm
wsg.24d38z2.cn/806225.htm
wsg.24d38z2.cn/826405.htm
wsg.24d38z2.cn/040465.htm
wsg.24d38z2.cn/622425.htm
wsf.24d38z2.cn/882445.htm
wsf.24d38z2.cn/711715.htm
wsf.24d38z2.cn/222605.htm
wsf.24d38z2.cn/537795.htm
wsf.24d38z2.cn/024885.htm
wsf.24d38z2.cn/553155.htm
wsf.24d38z2.cn/806445.htm
wsf.24d38z2.cn/640405.htm
wsf.24d38z2.cn/400665.htm
wsf.24d38z2.cn/420685.htm
wsd.24d38z2.cn/537935.htm
wsd.24d38z2.cn/422485.htm
wsd.24d38z2.cn/717335.htm
wsd.24d38z2.cn/004605.htm
wsd.24d38z2.cn/513355.htm
wsd.24d38z2.cn/468225.htm
wsd.24d38z2.cn/795595.htm
wsd.24d38z2.cn/662085.htm
wsd.24d38z2.cn/000465.htm
wsd.24d38z2.cn/448065.htm
wss.24d38z2.cn/488445.htm
wss.24d38z2.cn/204605.htm
wss.24d38z2.cn/282685.htm
wss.24d38z2.cn/646285.htm
wss.24d38z2.cn/224685.htm
wss.24d38z2.cn/153575.htm
wss.24d38z2.cn/842245.htm
wss.24d38z2.cn/268885.htm
wss.24d38z2.cn/022665.htm
wss.24d38z2.cn/917715.htm
wsa.24d38z2.cn/753535.htm
wsa.24d38z2.cn/482245.htm
wsa.24d38z2.cn/997135.htm
wsa.24d38z2.cn/020665.htm
wsa.24d38z2.cn/959955.htm
wsa.24d38z2.cn/604465.htm
wsa.24d38z2.cn/844085.htm
wsa.24d38z2.cn/957115.htm
wsa.24d38z2.cn/642045.htm
wsa.24d38z2.cn/933735.htm
wsp.24d38z2.cn/044685.htm
wsp.24d38z2.cn/666625.htm
wsp.24d38z2.cn/420865.htm
wsp.24d38z2.cn/115955.htm
wsp.24d38z2.cn/644265.htm
wsp.24d38z2.cn/862865.htm
wsp.24d38z2.cn/866405.htm
wsp.24d38z2.cn/597135.htm
wsp.24d38z2.cn/640885.htm
wsp.24d38z2.cn/866425.htm
wso.24d38z2.cn/624825.htm
wso.24d38z2.cn/333315.htm
wso.24d38z2.cn/113115.htm
wso.24d38z2.cn/682805.htm
wso.24d38z2.cn/395935.htm
wso.24d38z2.cn/428625.htm
wso.24d38z2.cn/933175.htm
wso.24d38z2.cn/600465.htm
wso.24d38z2.cn/359395.htm
wso.24d38z2.cn/779755.htm
wsi.24d38z2.cn/866665.htm
wsi.24d38z2.cn/224805.htm
wsi.24d38z2.cn/208205.htm
wsi.24d38z2.cn/288645.htm
wsi.24d38z2.cn/664085.htm
wsi.24d38z2.cn/200265.htm
wsi.24d38z2.cn/268465.htm
wsi.24d38z2.cn/084845.htm
wsi.24d38z2.cn/086445.htm
wsi.24d38z2.cn/420025.htm
wsu.24d38z2.cn/311955.htm
wsu.24d38z2.cn/208045.htm
wsu.24d38z2.cn/288645.htm
wsu.24d38z2.cn/224265.htm
wsu.24d38z2.cn/933735.htm
wsu.24d38z2.cn/024445.htm
wsu.24d38z2.cn/224005.htm
wsu.24d38z2.cn/464205.htm
wsu.24d38z2.cn/686285.htm
wsu.24d38z2.cn/284065.htm
wsy.24d38z2.cn/777115.htm
wsy.24d38z2.cn/266485.htm
wsy.24d38z2.cn/084045.htm
wsy.24d38z2.cn/399535.htm
wsy.24d38z2.cn/044045.htm
wsy.24d38z2.cn/004625.htm
wsy.24d38z2.cn/402845.htm
wsy.24d38z2.cn/082065.htm
wsy.24d38z2.cn/777975.htm
wsy.24d38z2.cn/795195.htm
wst.24d38z2.cn/424425.htm
wst.24d38z2.cn/204245.htm
wst.24d38z2.cn/820045.htm
wst.24d38z2.cn/420685.htm
wst.24d38z2.cn/828285.htm
wst.24d38z2.cn/202205.htm
wst.24d38z2.cn/513355.htm
wst.24d38z2.cn/880445.htm
wst.24d38z2.cn/282625.htm
wst.24d38z2.cn/246085.htm
wsr.24d38z2.cn/662605.htm
wsr.24d38z2.cn/602405.htm
wsr.24d38z2.cn/684265.htm
wsr.24d38z2.cn/115355.htm
wsr.24d38z2.cn/864865.htm
wsr.24d38z2.cn/600025.htm
wsr.24d38z2.cn/446245.htm
wsr.24d38z2.cn/317155.htm
wsr.24d38z2.cn/266845.htm
wsr.24d38z2.cn/268085.htm
wse.24d38z2.cn/028205.htm
wse.24d38z2.cn/757915.htm
wse.24d38z2.cn/842685.htm
wse.24d38z2.cn/999335.htm
wse.24d38z2.cn/284645.htm
wse.24d38z2.cn/957175.htm
wse.24d38z2.cn/775195.htm
wse.24d38z2.cn/046645.htm
wse.24d38z2.cn/115735.htm
wse.24d38z2.cn/442645.htm
wsw.24d38z2.cn/335995.htm
wsw.24d38z2.cn/286225.htm
wsw.24d38z2.cn/733375.htm
wsw.24d38z2.cn/248685.htm
wsw.24d38z2.cn/208885.htm
wsw.24d38z2.cn/264465.htm
wsw.24d38z2.cn/804245.htm
wsw.24d38z2.cn/337175.htm
wsw.24d38z2.cn/804245.htm
wsw.24d38z2.cn/224645.htm
wsq.24d38z2.cn/860665.htm
wsq.24d38z2.cn/420025.htm
wsq.24d38z2.cn/771955.htm
wsq.24d38z2.cn/804605.htm
wsq.24d38z2.cn/060245.htm
wsq.24d38z2.cn/199315.htm
wsq.24d38z2.cn/593715.htm
wsq.24d38z2.cn/240485.htm
wsq.24d38z2.cn/064005.htm
wsq.24d38z2.cn/280005.htm
watv.24d38z2.cn/551335.htm
watv.24d38z2.cn/824085.htm
watv.24d38z2.cn/793995.htm
watv.24d38z2.cn/880065.htm
watv.24d38z2.cn/228445.htm
watv.24d38z2.cn/799795.htm
watv.24d38z2.cn/339535.htm
watv.24d38z2.cn/939515.htm
watv.24d38z2.cn/486465.htm
watv.24d38z2.cn/282205.htm
wan.24d38z2.cn/244425.htm
wan.24d38z2.cn/539195.htm
wan.24d38z2.cn/535535.htm
wan.24d38z2.cn/999175.htm
wan.24d38z2.cn/686685.htm
wan.24d38z2.cn/626265.htm
wan.24d38z2.cn/660685.htm
wan.24d38z2.cn/391975.htm
wan.24d38z2.cn/846045.htm
wan.24d38z2.cn/220825.htm
wab.24d38z2.cn/517135.htm
wab.24d38z2.cn/628825.htm
wab.24d38z2.cn/224625.htm
wab.24d38z2.cn/682445.htm
wab.24d38z2.cn/620405.htm
wab.24d38z2.cn/371555.htm
wab.24d38z2.cn/939355.htm
wab.24d38z2.cn/840445.htm
wab.24d38z2.cn/604045.htm
wab.24d38z2.cn/248205.htm
wav.24d38z2.cn/066825.htm
wav.24d38z2.cn/228685.htm
wav.24d38z2.cn/040045.htm
wav.24d38z2.cn/753595.htm
wav.24d38z2.cn/597355.htm
wav.24d38z2.cn/157355.htm
wav.24d38z2.cn/082085.htm
wav.24d38z2.cn/359315.htm
wav.24d38z2.cn/604085.htm
wav.24d38z2.cn/795555.htm
wac.24d38z2.cn/622205.htm
wac.24d38z2.cn/222445.htm
wac.24d38z2.cn/773755.htm
wac.24d38z2.cn/468005.htm
wac.24d38z2.cn/951735.htm
wac.24d38z2.cn/882645.htm
wac.24d38z2.cn/191575.htm
wac.24d38z2.cn/260425.htm
wac.24d38z2.cn/048465.htm
wac.24d38z2.cn/246645.htm
wax.24d38z2.cn/533155.htm
wax.24d38z2.cn/044665.htm
wax.24d38z2.cn/806045.htm
wax.24d38z2.cn/842885.htm
wax.24d38z2.cn/244485.htm
wax.24d38z2.cn/284285.htm
wax.24d38z2.cn/260625.htm
wax.24d38z2.cn/086425.htm
wax.24d38z2.cn/026205.htm
wax.24d38z2.cn/951975.htm
waz.24d38z2.cn/228605.htm
waz.24d38z2.cn/246205.htm
