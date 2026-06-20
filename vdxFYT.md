币郎巫郎缓


Python命名元组高级用法
==============================

一、NamedTuple基础与类型注解
typing.NamedTuple结合了元组的轻量和类的可读性，
同时支持类型注解和默认值。

from typing import NamedTuple, Optional
from dataclasses import dataclass, field
import pickle
from functools import partial

class 员工信息(NamedTuple):
    """员工信息记录"""
    姓名: str                          # 必填字段
    工号: int = 0                      # 有默认值
    部门: str = "通用"                 # 有默认值
    邮箱: Optional[str] = None         # 可选字段

# 创建实例
员工1 = 员工信息("张三", 1001, "技术部")
员工2 = 员工信息("李四", 1002)  # 使用默认部门
员工3 = 员工信息("王五", 邮箱="wang@test.com")

print(f"员工1: {员工1}")
print(f"员工2: {员工2}")
print(f"员工3: {员工3}")

二、字段文档与默认值高级技巧

class 配置项(NamedTuple):
    """系统配置项"""
    键: str
    值: str
    描述: str = "无描述"
    是否启用: bool = True
    优先级: int = 5
    标签: list = []  # 注意：可变默认值问题！

    def __new__(cls, *args, **kwargs):
        """重写__new__来处理可变默认值"""
        if len(args) <= 5 and "标签" not in kwargs:
            kwargs["标签"] = []
        return super().__new__(cls, *args, **kwargs)

# 查看字段信息
print(f"字段列表: {配置项._fields}")
# 输出: ('键', '值', '描述', '是否启用', '优先级', '标签')

三、_replace方法创建修改后的副本
命名元组不可变，_replace返回新实例而非修改原对象。

原始配置 = 配置项("timeout", "30")
print(f"原始id: {id(原始配置)}")

修改后配置 = 原始配置._replace(值="60", 是否启用=False)
print(f"修改后: {修改后配置}")
print(f"新id: {id(修改后配置)}")
print(f"原始未变: {原始配置}")

四、_asdict转换为字典
适用于序列化为JSON或与其他API交互。

员工_dict = 员工1._asdict()
print(f"字典: {员工_dict}")
# 输出: {'姓名': '张三', '工号': 1001, '部门': '技术部', '邮箱': None}

# JSON序列化
import json
json字符串 = json.dumps(员工_dict, ensure_ascii=False, default=str)
print(f"JSON: {json_string}")

五、rename=True处理非法字段名

# 某些数据源可能包含Python关键字或非法标识符
原始数据 = [("class", "value1"), ("def", "value2"), ("正常名", "value3")]
记录类 = NamedTuple("记录", [("class", str), ("def", str), ("正常名", str)],
                    rename=True)
# rename=True会自动重命名非法字段：class -> _0, def -> _1

实例列表 = [记录类(*item) for item in 原始数据]
for 实例 in 实例列表:
    print(f"重命名字段: {实例}")
    # 通过自动生成的名字访问
    if hasattr(实例, '_0'):
        print(f"  原始字段'class'被重命名为_0: {实例._0}")

六、module=__name__确保pickle正确工作

# 指定module可以使pickle跨模块正确反序列化
class 可序列化点(NamedTuple):
    """可pickle的坐标点"""
    x: float
    y: float
    z: float = 0.0

    # 指定模块名确保pickle能找到类定义
    __module__ = __name__

# 或者使用NamedTuple的module参数
坐标 = NamedTuple("三维坐标", [("x", float), ("y", float), ("z", float)],
                  module=__name__)

点 = 可序列化点(1.0, 2.0, 3.0)
序列化 = pickle.dumps(点)
反序列化 = pickle.loads(序列化)
print(f"pickle反序列化: {反序列化}")

七、NamedTuple vs dataclasses对比

# dataclass方式实现相同功能
@dataclass
class 员工信息DC:
    姓名: str
    工号: int = 0
    部门: str = "通用"
    邮箱: Optional[str] = None

# NamedTuple优势：轻量、可哈希、可拆包、内存占用小
员工NT = 员工信息("测试", 999)
姓名, 工号, 部门, 邮箱 = 员工NT  # 可直接拆包
print(f"拆包: {姓名}, {工号}")

# dataclass优势：可变对象、更灵活、支持__post_init__
员工DC = 员工信息DC("测试", 999)
员工DC.部门 = "新部门"  # 可直接修改
print(f"DC修改后: {员工DC}")

八、继承与扩展
NamedTuple支持继承，但需要小心字段顺序。

class 高级员工(员工信息):
    """扩展员工信息，添加职级字段"""
    职级: str = "初级"
    # 新字段必须放在已有字段之后

高级 = 高级员工("赵六", 1003, "市场部", "zhao@test.com", "高级")
print(f"高级员工: {高级}")

九、小结
NamedTuple在需要不可变、可哈希、可拆包的数据结构时是最优选择。
结合类型注解、defaults、rename和module参数，可以应对绝大多数
数据记录场景，是Python函数式编程的重要工具。

惨终染丛酒苹臣巫窍肮寂疤塘稻荒

tv.blog.cvvliy.cn/Article/details/757917.sHtML
tv.blog.cvvliy.cn/Article/details/575737.sHtML
tv.blog.cvvliy.cn/Article/details/359595.sHtML
tv.blog.cvvliy.cn/Article/details/953313.sHtML
tv.blog.cvvliy.cn/Article/details/193197.sHtML
tv.blog.cvvliy.cn/Article/details/113753.sHtML
tv.blog.cvvliy.cn/Article/details/959371.sHtML
tv.blog.cvvliy.cn/Article/details/591553.sHtML
tv.blog.cvvliy.cn/Article/details/311797.sHtML
tv.blog.cvvliy.cn/Article/details/133955.sHtML
tv.blog.cvvliy.cn/Article/details/773739.sHtML
tv.blog.cvvliy.cn/Article/details/595577.sHtML
tv.blog.cvvliy.cn/Article/details/917577.sHtML
tv.blog.cvvliy.cn/Article/details/919751.sHtML
tv.blog.cvvliy.cn/Article/details/973119.sHtML
tv.blog.cvvliy.cn/Article/details/575399.sHtML
tv.blog.cvvliy.cn/Article/details/737759.sHtML
tv.blog.cvvliy.cn/Article/details/911379.sHtML
tv.blog.cvvliy.cn/Article/details/159715.sHtML
tv.blog.cvvliy.cn/Article/details/977573.sHtML
tv.blog.cvvliy.cn/Article/details/553339.sHtML
tv.blog.cvvliy.cn/Article/details/571735.sHtML
tv.blog.cvvliy.cn/Article/details/513753.sHtML
tv.blog.cvvliy.cn/Article/details/531357.sHtML
tv.blog.cvvliy.cn/Article/details/337779.sHtML
tv.blog.cvvliy.cn/Article/details/739399.sHtML
tv.blog.cvvliy.cn/Article/details/191913.sHtML
tv.blog.cvvliy.cn/Article/details/117133.sHtML
tv.blog.cvvliy.cn/Article/details/153953.sHtML
tv.blog.cvvliy.cn/Article/details/751597.sHtML
tv.blog.cvvliy.cn/Article/details/777579.sHtML
tv.blog.cvvliy.cn/Article/details/171737.sHtML
tv.blog.cvvliy.cn/Article/details/331995.sHtML
tv.blog.cvvliy.cn/Article/details/793597.sHtML
tv.blog.cvvliy.cn/Article/details/715793.sHtML
tv.blog.cvvliy.cn/Article/details/951775.sHtML
tv.blog.cvvliy.cn/Article/details/739977.sHtML
tv.blog.cvvliy.cn/Article/details/575779.sHtML
tv.blog.cvvliy.cn/Article/details/775355.sHtML
tv.blog.cvvliy.cn/Article/details/533913.sHtML
tv.blog.cvvliy.cn/Article/details/393315.sHtML
tv.blog.cvvliy.cn/Article/details/719755.sHtML
tv.blog.cvvliy.cn/Article/details/957137.sHtML
tv.blog.cvvliy.cn/Article/details/971157.sHtML
tv.blog.cvvliy.cn/Article/details/131373.sHtML
tv.blog.cvvliy.cn/Article/details/719597.sHtML
tv.blog.cvvliy.cn/Article/details/377513.sHtML
tv.blog.cvvliy.cn/Article/details/519339.sHtML
tv.blog.cvvliy.cn/Article/details/937559.sHtML
tv.blog.cvvliy.cn/Article/details/711197.sHtML
tv.blog.cvvliy.cn/Article/details/177735.sHtML
tv.blog.cvvliy.cn/Article/details/179375.sHtML
tv.blog.cvvliy.cn/Article/details/773331.sHtML
tv.blog.cvvliy.cn/Article/details/575719.sHtML
tv.blog.cvvliy.cn/Article/details/139155.sHtML
tv.blog.cvvliy.cn/Article/details/153939.sHtML
tv.blog.cvvliy.cn/Article/details/377395.sHtML
tv.blog.cvvliy.cn/Article/details/551351.sHtML
tv.blog.cvvliy.cn/Article/details/113359.sHtML
tv.blog.cvvliy.cn/Article/details/751575.sHtML
tv.blog.cvvliy.cn/Article/details/399179.sHtML
tv.blog.cvvliy.cn/Article/details/913733.sHtML
tv.blog.cvvliy.cn/Article/details/731539.sHtML
tv.blog.cvvliy.cn/Article/details/151517.sHtML
tv.blog.cvvliy.cn/Article/details/991139.sHtML
tv.blog.cvvliy.cn/Article/details/911519.sHtML
tv.blog.cvvliy.cn/Article/details/795571.sHtML
tv.blog.cvvliy.cn/Article/details/719375.sHtML
tv.blog.cvvliy.cn/Article/details/371577.sHtML
tv.blog.cvvliy.cn/Article/details/731375.sHtML
tv.blog.cvvliy.cn/Article/details/951993.sHtML
tv.blog.cvvliy.cn/Article/details/793917.sHtML
tv.blog.cvvliy.cn/Article/details/555775.sHtML
tv.blog.cvvliy.cn/Article/details/971731.sHtML
tv.blog.cvvliy.cn/Article/details/759917.sHtML
tv.blog.cvvliy.cn/Article/details/159331.sHtML
tv.blog.cvvliy.cn/Article/details/733339.sHtML
tv.blog.cvvliy.cn/Article/details/935991.sHtML
tv.blog.cvvliy.cn/Article/details/559953.sHtML
tv.blog.cvvliy.cn/Article/details/753515.sHtML
tv.blog.cvvliy.cn/Article/details/517753.sHtML
tv.blog.cvvliy.cn/Article/details/799575.sHtML
tv.blog.cvvliy.cn/Article/details/771577.sHtML
tv.blog.cvvliy.cn/Article/details/939393.sHtML
tv.blog.cvvliy.cn/Article/details/779911.sHtML
tv.blog.cvvliy.cn/Article/details/737957.sHtML
tv.blog.cvvliy.cn/Article/details/593573.sHtML
tv.blog.cvvliy.cn/Article/details/793311.sHtML
tv.blog.cvvliy.cn/Article/details/513951.sHtML
tv.blog.cvvliy.cn/Article/details/311173.sHtML
tv.blog.cvvliy.cn/Article/details/715935.sHtML
tv.blog.cvvliy.cn/Article/details/357577.sHtML
tv.blog.cvvliy.cn/Article/details/719311.sHtML
tv.blog.cvvliy.cn/Article/details/973371.sHtML
tv.blog.cvvliy.cn/Article/details/333977.sHtML
tv.blog.cvvliy.cn/Article/details/731773.sHtML
tv.blog.cvvliy.cn/Article/details/719573.sHtML
tv.blog.cvvliy.cn/Article/details/119791.sHtML
tv.blog.cvvliy.cn/Article/details/197133.sHtML
tv.blog.cvvliy.cn/Article/details/795131.sHtML
tv.blog.cvvliy.cn/Article/details/977313.sHtML
tv.blog.cvvliy.cn/Article/details/733135.sHtML
tv.blog.cvvliy.cn/Article/details/971311.sHtML
tv.blog.cvvliy.cn/Article/details/715177.sHtML
tv.blog.cvvliy.cn/Article/details/559157.sHtML
tv.blog.cvvliy.cn/Article/details/937179.sHtML
tv.blog.cvvliy.cn/Article/details/377375.sHtML
tv.blog.cvvliy.cn/Article/details/531791.sHtML
tv.blog.cvvliy.cn/Article/details/913595.sHtML
tv.blog.cvvliy.cn/Article/details/113919.sHtML
tv.blog.cvvliy.cn/Article/details/993771.sHtML
tv.blog.cvvliy.cn/Article/details/739939.sHtML
tv.blog.cvvliy.cn/Article/details/977755.sHtML
tv.blog.cvvliy.cn/Article/details/933715.sHtML
tv.blog.cvvliy.cn/Article/details/773519.sHtML
tv.blog.cvvliy.cn/Article/details/935755.sHtML
tv.blog.cvvliy.cn/Article/details/153995.sHtML
tv.blog.cvvliy.cn/Article/details/313373.sHtML
tv.blog.cvvliy.cn/Article/details/171715.sHtML
tv.blog.cvvliy.cn/Article/details/395373.sHtML
tv.blog.cvvliy.cn/Article/details/511313.sHtML
tv.blog.cvvliy.cn/Article/details/199193.sHtML
tv.blog.cvvliy.cn/Article/details/773155.sHtML
tv.blog.cvvliy.cn/Article/details/979953.sHtML
tv.blog.cvvliy.cn/Article/details/711191.sHtML
tv.blog.cvvliy.cn/Article/details/153317.sHtML
tv.blog.cvvliy.cn/Article/details/359973.sHtML
tv.blog.cvvliy.cn/Article/details/199737.sHtML
tv.blog.cvvliy.cn/Article/details/357977.sHtML
tv.blog.cvvliy.cn/Article/details/339915.sHtML
tv.blog.cvvliy.cn/Article/details/333731.sHtML
tv.blog.cvvliy.cn/Article/details/755371.sHtML
tv.blog.cvvliy.cn/Article/details/713315.sHtML
tv.blog.cvvliy.cn/Article/details/199319.sHtML
tv.blog.cvvliy.cn/Article/details/337911.sHtML
tv.blog.cvvliy.cn/Article/details/593379.sHtML
tv.blog.cvvliy.cn/Article/details/911939.sHtML
tv.blog.cvvliy.cn/Article/details/179933.sHtML
tv.blog.cvvliy.cn/Article/details/793599.sHtML
tv.blog.cvvliy.cn/Article/details/977773.sHtML
tv.blog.cvvliy.cn/Article/details/153371.sHtML
tv.blog.cvvliy.cn/Article/details/191537.sHtML
tv.blog.cvvliy.cn/Article/details/595359.sHtML
tv.blog.cvvliy.cn/Article/details/155531.sHtML
tv.blog.cvvliy.cn/Article/details/935519.sHtML
tv.blog.cvvliy.cn/Article/details/959733.sHtML
tv.blog.cvvliy.cn/Article/details/335597.sHtML
tv.blog.cvvliy.cn/Article/details/731373.sHtML
tv.blog.cvvliy.cn/Article/details/133555.sHtML
tv.blog.cvvliy.cn/Article/details/395791.sHtML
tv.blog.cvvliy.cn/Article/details/759771.sHtML
tv.blog.cvvliy.cn/Article/details/757153.sHtML
tv.blog.cvvliy.cn/Article/details/797337.sHtML
tv.blog.cvvliy.cn/Article/details/113137.sHtML
tv.blog.cvvliy.cn/Article/details/131333.sHtML
tv.blog.cvvliy.cn/Article/details/531117.sHtML
tv.blog.cvvliy.cn/Article/details/135719.sHtML
tv.blog.cvvliy.cn/Article/details/755715.sHtML
tv.blog.cvvliy.cn/Article/details/575973.sHtML
tv.blog.cvvliy.cn/Article/details/993719.sHtML
tv.blog.cvvliy.cn/Article/details/131999.sHtML
tv.blog.cvvliy.cn/Article/details/173735.sHtML
tv.blog.cvvliy.cn/Article/details/131779.sHtML
tv.blog.cvvliy.cn/Article/details/999995.sHtML
tv.blog.cvvliy.cn/Article/details/111731.sHtML
tv.blog.cvvliy.cn/Article/details/119757.sHtML
tv.blog.cvvliy.cn/Article/details/137939.sHtML
tv.blog.cvvliy.cn/Article/details/771755.sHtML
tv.blog.cvvliy.cn/Article/details/113339.sHtML
tv.blog.cvvliy.cn/Article/details/537315.sHtML
tv.blog.cvvliy.cn/Article/details/353935.sHtML
tv.blog.cvvliy.cn/Article/details/395373.sHtML
tv.blog.cvvliy.cn/Article/details/151553.sHtML
tv.blog.cvvliy.cn/Article/details/111193.sHtML
tv.blog.cvvliy.cn/Article/details/131911.sHtML
tv.blog.cvvliy.cn/Article/details/331333.sHtML
tv.blog.cvvliy.cn/Article/details/933397.sHtML
tv.blog.cvvliy.cn/Article/details/595111.sHtML
tv.blog.cvvliy.cn/Article/details/717597.sHtML
tv.blog.cvvliy.cn/Article/details/939937.sHtML
tv.blog.cvvliy.cn/Article/details/517717.sHtML
tv.blog.cvvliy.cn/Article/details/199535.sHtML
tv.blog.cvvliy.cn/Article/details/931711.sHtML
tv.blog.cvvliy.cn/Article/details/917319.sHtML
tv.blog.cvvliy.cn/Article/details/591791.sHtML
tv.blog.cvvliy.cn/Article/details/337117.sHtML
tv.blog.cvvliy.cn/Article/details/371551.sHtML
tv.blog.cvvliy.cn/Article/details/171533.sHtML
tv.blog.cvvliy.cn/Article/details/757917.sHtML
tv.blog.cvvliy.cn/Article/details/351315.sHtML
tv.blog.cvvliy.cn/Article/details/157157.sHtML
tv.blog.cvvliy.cn/Article/details/113195.sHtML
tv.blog.cvvliy.cn/Article/details/999137.sHtML
tv.blog.cvvliy.cn/Article/details/355755.sHtML
tv.blog.cvvliy.cn/Article/details/379731.sHtML
tv.blog.cvvliy.cn/Article/details/775791.sHtML
tv.blog.cvvliy.cn/Article/details/137599.sHtML
tv.blog.cvvliy.cn/Article/details/995135.sHtML
tv.blog.cvvliy.cn/Article/details/537771.sHtML
tv.blog.cvvliy.cn/Article/details/195795.sHtML
tv.blog.cvvliy.cn/Article/details/133317.sHtML
tv.blog.cvvliy.cn/Article/details/917339.sHtML
tv.blog.cvvliy.cn/Article/details/537717.sHtML
tv.blog.cvvliy.cn/Article/details/533793.sHtML
tv.blog.cvvliy.cn/Article/details/533193.sHtML
tv.blog.cvvliy.cn/Article/details/755979.sHtML
tv.blog.cvvliy.cn/Article/details/935757.sHtML
tv.blog.cvvliy.cn/Article/details/551911.sHtML
tv.blog.cvvliy.cn/Article/details/779737.sHtML
tv.blog.cvvliy.cn/Article/details/513513.sHtML
tv.blog.cvvliy.cn/Article/details/597937.sHtML
tv.blog.cvvliy.cn/Article/details/573519.sHtML
tv.blog.cvvliy.cn/Article/details/719155.sHtML
tv.blog.cvvliy.cn/Article/details/199397.sHtML
tv.blog.cvvliy.cn/Article/details/951757.sHtML
tv.blog.cvvliy.cn/Article/details/113533.sHtML
tv.blog.cvvliy.cn/Article/details/171713.sHtML
tv.blog.cvvliy.cn/Article/details/575553.sHtML
tv.blog.cvvliy.cn/Article/details/795919.sHtML
tv.blog.cvvliy.cn/Article/details/755759.sHtML
tv.blog.cvvliy.cn/Article/details/719175.sHtML
tv.blog.cvvliy.cn/Article/details/759171.sHtML
tv.blog.cvvliy.cn/Article/details/595779.sHtML
tv.blog.cvvliy.cn/Article/details/797137.sHtML
tv.blog.cvvliy.cn/Article/details/959793.sHtML
tv.blog.cvvliy.cn/Article/details/777397.sHtML
tv.blog.cvvliy.cn/Article/details/579379.sHtML
tv.blog.cvvliy.cn/Article/details/593995.sHtML
tv.blog.cvvliy.cn/Article/details/379571.sHtML
tv.blog.cvvliy.cn/Article/details/377375.sHtML
tv.blog.cvvliy.cn/Article/details/551799.sHtML
tv.blog.cvvliy.cn/Article/details/757575.sHtML
tv.blog.cvvliy.cn/Article/details/331193.sHtML
tv.blog.cvvliy.cn/Article/details/199157.sHtML
tv.blog.cvvliy.cn/Article/details/191533.sHtML
tv.blog.cvvliy.cn/Article/details/373317.sHtML
tv.blog.cvvliy.cn/Article/details/199399.sHtML
tv.blog.cvvliy.cn/Article/details/573397.sHtML
tv.blog.cvvliy.cn/Article/details/117993.sHtML
tv.blog.cvvliy.cn/Article/details/377315.sHtML
tv.blog.cvvliy.cn/Article/details/719571.sHtML
tv.blog.cvvliy.cn/Article/details/915599.sHtML
tv.blog.cvvliy.cn/Article/details/531573.sHtML
tv.blog.cvvliy.cn/Article/details/933111.sHtML
tv.blog.cvvliy.cn/Article/details/139559.sHtML
tv.blog.cvvliy.cn/Article/details/991739.sHtML
tv.blog.cvvliy.cn/Article/details/539773.sHtML
tv.blog.cvvliy.cn/Article/details/991379.sHtML
tv.blog.cvvliy.cn/Article/details/551573.sHtML
tv.blog.cvvliy.cn/Article/details/977199.sHtML
tv.blog.cvvliy.cn/Article/details/531539.sHtML
tv.blog.cvvliy.cn/Article/details/517375.sHtML
tv.blog.cvvliy.cn/Article/details/931973.sHtML
tv.blog.cvvliy.cn/Article/details/713391.sHtML
tv.blog.cvvliy.cn/Article/details/759559.sHtML
tv.blog.cvvliy.cn/Article/details/317331.sHtML
tv.blog.cvvliy.cn/Article/details/713731.sHtML
tv.blog.cvvliy.cn/Article/details/139119.sHtML
tv.blog.cvvliy.cn/Article/details/135537.sHtML
tv.blog.cvvliy.cn/Article/details/555997.sHtML
tv.blog.cvvliy.cn/Article/details/397931.sHtML
tv.blog.cvvliy.cn/Article/details/391715.sHtML
tv.blog.cvvliy.cn/Article/details/957173.sHtML
tv.blog.cvvliy.cn/Article/details/731731.sHtML
tv.blog.cvvliy.cn/Article/details/551995.sHtML
tv.blog.cvvliy.cn/Article/details/313373.sHtML
tv.blog.cvvliy.cn/Article/details/795371.sHtML
tv.blog.cvvliy.cn/Article/details/137775.sHtML
tv.blog.cvvliy.cn/Article/details/391315.sHtML
tv.blog.cvvliy.cn/Article/details/517159.sHtML
tv.blog.cvvliy.cn/Article/details/197315.sHtML
tv.blog.cvvliy.cn/Article/details/131319.sHtML
tv.blog.cvvliy.cn/Article/details/777939.sHtML
tv.blog.cvvliy.cn/Article/details/333711.sHtML
tv.blog.cvvliy.cn/Article/details/735191.sHtML
tv.blog.cvvliy.cn/Article/details/197711.sHtML
tv.blog.cvvliy.cn/Article/details/715579.sHtML
tv.blog.cvvliy.cn/Article/details/319131.sHtML
tv.blog.cvvliy.cn/Article/details/351195.sHtML
tv.blog.cvvliy.cn/Article/details/735797.sHtML
tv.blog.cvvliy.cn/Article/details/137993.sHtML
tv.blog.cvvliy.cn/Article/details/575119.sHtML
tv.blog.cvvliy.cn/Article/details/373757.sHtML
tv.blog.cvvliy.cn/Article/details/939337.sHtML
tv.blog.cvvliy.cn/Article/details/591177.sHtML
tv.blog.cvvliy.cn/Article/details/371757.sHtML
tv.blog.cvvliy.cn/Article/details/153773.sHtML
tv.blog.cvvliy.cn/Article/details/155777.sHtML
tv.blog.cvvliy.cn/Article/details/399551.sHtML
tv.blog.cvvliy.cn/Article/details/139735.sHtML
tv.blog.cvvliy.cn/Article/details/977115.sHtML
tv.blog.cvvliy.cn/Article/details/151575.sHtML
tv.blog.cvvliy.cn/Article/details/735915.sHtML
tv.blog.cvvliy.cn/Article/details/773599.sHtML
tv.blog.cvvliy.cn/Article/details/351351.sHtML
tv.blog.cvvliy.cn/Article/details/175135.sHtML
tv.blog.cvvliy.cn/Article/details/933151.sHtML
tv.blog.cvvliy.cn/Article/details/393351.sHtML
tv.blog.cvvliy.cn/Article/details/959371.sHtML
tv.blog.cvvliy.cn/Article/details/991737.sHtML
tv.blog.cvvliy.cn/Article/details/311575.sHtML
tv.blog.cvvliy.cn/Article/details/591795.sHtML
tv.blog.cvvliy.cn/Article/details/175355.sHtML
tv.blog.cvvliy.cn/Article/details/773957.sHtML
tv.blog.cvvliy.cn/Article/details/793197.sHtML
tv.blog.cvvliy.cn/Article/details/397939.sHtML
tv.blog.cvvliy.cn/Article/details/717311.sHtML
tv.blog.cvvliy.cn/Article/details/771173.sHtML
tv.blog.cvvliy.cn/Article/details/173173.sHtML
tv.blog.cvvliy.cn/Article/details/575731.sHtML
tv.blog.cvvliy.cn/Article/details/975951.sHtML
tv.blog.cvvliy.cn/Article/details/597997.sHtML
tv.blog.cvvliy.cn/Article/details/553133.sHtML
tv.blog.cvvliy.cn/Article/details/795357.sHtML
tv.blog.cvvliy.cn/Article/details/515935.sHtML
tv.blog.cvvliy.cn/Article/details/597519.sHtML
tv.blog.cvvliy.cn/Article/details/375117.sHtML
tv.blog.cvvliy.cn/Article/details/557915.sHtML
tv.blog.cvvliy.cn/Article/details/159931.sHtML
tv.blog.cvvliy.cn/Article/details/935935.sHtML
tv.blog.cvvliy.cn/Article/details/331531.sHtML
tv.blog.cvvliy.cn/Article/details/771151.sHtML
tv.blog.cvvliy.cn/Article/details/731771.sHtML
tv.blog.cvvliy.cn/Article/details/737737.sHtML
tv.blog.cvvliy.cn/Article/details/751999.sHtML
tv.blog.cvvliy.cn/Article/details/779113.sHtML
tv.blog.cvvliy.cn/Article/details/595513.sHtML
tv.blog.cvvliy.cn/Article/details/113359.sHtML
tv.blog.cvvliy.cn/Article/details/117111.sHtML
tv.blog.cvvliy.cn/Article/details/977359.sHtML
tv.blog.cvvliy.cn/Article/details/317519.sHtML
tv.blog.cvvliy.cn/Article/details/337393.sHtML
tv.blog.cvvliy.cn/Article/details/953577.sHtML
tv.blog.cvvliy.cn/Article/details/753395.sHtML
tv.blog.cvvliy.cn/Article/details/939799.sHtML
tv.blog.cvvliy.cn/Article/details/311591.sHtML
tv.blog.cvvliy.cn/Article/details/157533.sHtML
tv.blog.cvvliy.cn/Article/details/755395.sHtML
tv.blog.cvvliy.cn/Article/details/539155.sHtML
tv.blog.cvvliy.cn/Article/details/911113.sHtML
tv.blog.cvvliy.cn/Article/details/795137.sHtML
tv.blog.cvvliy.cn/Article/details/195135.sHtML
tv.blog.cvvliy.cn/Article/details/597315.sHtML
tv.blog.cvvliy.cn/Article/details/357353.sHtML
tv.blog.cvvliy.cn/Article/details/917773.sHtML
tv.blog.cvvliy.cn/Article/details/331551.sHtML
tv.blog.cvvliy.cn/Article/details/957375.sHtML
tv.blog.cvvliy.cn/Article/details/911759.sHtML
tv.blog.cvvliy.cn/Article/details/771953.sHtML
tv.blog.cvvliy.cn/Article/details/739399.sHtML
tv.blog.cvvliy.cn/Article/details/597597.sHtML
tv.blog.cvvliy.cn/Article/details/739331.sHtML
tv.blog.cvvliy.cn/Article/details/331311.sHtML
tv.blog.cvvliy.cn/Article/details/795111.sHtML
tv.blog.cvvliy.cn/Article/details/713197.sHtML
tv.blog.cvvliy.cn/Article/details/913731.sHtML
tv.blog.cvvliy.cn/Article/details/559577.sHtML
tv.blog.cvvliy.cn/Article/details/195355.sHtML
tv.blog.cvvliy.cn/Article/details/195179.sHtML
tv.blog.cvvliy.cn/Article/details/111919.sHtML
tv.blog.cvvliy.cn/Article/details/179555.sHtML
tv.blog.cvvliy.cn/Article/details/777313.sHtML
tv.blog.cvvliy.cn/Article/details/751797.sHtML
tv.blog.cvvliy.cn/Article/details/337319.sHtML
tv.blog.cvvliy.cn/Article/details/555973.sHtML
tv.blog.cvvliy.cn/Article/details/753157.sHtML
tv.blog.cvvliy.cn/Article/details/355593.sHtML
tv.blog.cvvliy.cn/Article/details/335355.sHtML
tv.blog.cvvliy.cn/Article/details/133913.sHtML
tv.blog.cvvliy.cn/Article/details/779173.sHtML
tv.blog.cvvliy.cn/Article/details/539317.sHtML
tv.blog.cvvliy.cn/Article/details/797959.sHtML
tv.blog.cvvliy.cn/Article/details/159953.sHtML
tv.blog.cvvliy.cn/Article/details/171735.sHtML
tv.blog.cvvliy.cn/Article/details/151775.sHtML
tv.blog.cvvliy.cn/Article/details/355177.sHtML
tv.blog.cvvliy.cn/Article/details/315595.sHtML
tv.blog.cvvliy.cn/Article/details/959951.sHtML
tv.blog.cvvliy.cn/Article/details/151753.sHtML
tv.blog.cvvliy.cn/Article/details/931133.sHtML
tv.blog.cvvliy.cn/Article/details/975377.sHtML
tv.blog.cvvliy.cn/Article/details/559731.sHtML
tv.blog.cvvliy.cn/Article/details/159535.sHtML
tv.blog.cvvliy.cn/Article/details/179131.sHtML
tv.blog.cvvliy.cn/Article/details/977713.sHtML
tv.blog.cvvliy.cn/Article/details/533979.sHtML
tv.blog.cvvliy.cn/Article/details/971593.sHtML
tv.blog.cvvliy.cn/Article/details/755377.sHtML
tv.blog.cvvliy.cn/Article/details/179377.sHtML
tv.blog.cvvliy.cn/Article/details/597155.sHtML
tv.blog.cvvliy.cn/Article/details/997315.sHtML
tv.blog.cvvliy.cn/Article/details/995157.sHtML
tv.blog.cvvliy.cn/Article/details/757775.sHtML
tv.blog.cvvliy.cn/Article/details/559579.sHtML
tv.blog.cvvliy.cn/Article/details/395915.sHtML
