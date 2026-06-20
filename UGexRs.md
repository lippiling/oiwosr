萍副季颗宰


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

脸够章簇倭倭乩苏呜潭偷陆渡侥得

tv.blog.itaadf.cn/Article/details/046264.sHtML
tv.blog.itaadf.cn/Article/details/684206.sHtML
tv.blog.itaadf.cn/Article/details/040444.sHtML
tv.blog.itaadf.cn/Article/details/286688.sHtML
tv.blog.itaadf.cn/Article/details/579959.sHtML
tv.blog.itaadf.cn/Article/details/311513.sHtML
tv.blog.itaadf.cn/Article/details/800028.sHtML
tv.blog.itaadf.cn/Article/details/686042.sHtML
tv.blog.itaadf.cn/Article/details/937719.sHtML
tv.blog.itaadf.cn/Article/details/840482.sHtML
tv.blog.itaadf.cn/Article/details/177937.sHtML
tv.blog.itaadf.cn/Article/details/668406.sHtML
tv.blog.itaadf.cn/Article/details/684248.sHtML
tv.blog.itaadf.cn/Article/details/008428.sHtML
tv.blog.itaadf.cn/Article/details/391731.sHtML
tv.blog.itaadf.cn/Article/details/866222.sHtML
tv.blog.itaadf.cn/Article/details/062422.sHtML
tv.blog.itaadf.cn/Article/details/262844.sHtML
tv.blog.itaadf.cn/Article/details/939571.sHtML
tv.blog.itaadf.cn/Article/details/339175.sHtML
tv.blog.itaadf.cn/Article/details/000846.sHtML
tv.blog.itaadf.cn/Article/details/539159.sHtML
tv.blog.itaadf.cn/Article/details/624460.sHtML
tv.blog.itaadf.cn/Article/details/739315.sHtML
tv.blog.itaadf.cn/Article/details/951915.sHtML
tv.blog.itaadf.cn/Article/details/973139.sHtML
tv.blog.itaadf.cn/Article/details/860828.sHtML
tv.blog.itaadf.cn/Article/details/371777.sHtML
tv.blog.itaadf.cn/Article/details/959575.sHtML
tv.blog.itaadf.cn/Article/details/082862.sHtML
tv.blog.itaadf.cn/Article/details/680242.sHtML
tv.blog.itaadf.cn/Article/details/606040.sHtML
tv.blog.itaadf.cn/Article/details/711391.sHtML
tv.blog.itaadf.cn/Article/details/446226.sHtML
tv.blog.itaadf.cn/Article/details/084662.sHtML
tv.blog.itaadf.cn/Article/details/159591.sHtML
tv.blog.itaadf.cn/Article/details/575519.sHtML
tv.blog.itaadf.cn/Article/details/266286.sHtML
tv.blog.itaadf.cn/Article/details/531193.sHtML
tv.blog.itaadf.cn/Article/details/400464.sHtML
tv.blog.itaadf.cn/Article/details/937593.sHtML
tv.blog.itaadf.cn/Article/details/284020.sHtML
tv.blog.itaadf.cn/Article/details/442448.sHtML
tv.blog.itaadf.cn/Article/details/331315.sHtML
tv.blog.itaadf.cn/Article/details/264000.sHtML
tv.blog.itaadf.cn/Article/details/664884.sHtML
tv.blog.itaadf.cn/Article/details/806682.sHtML
tv.blog.itaadf.cn/Article/details/486400.sHtML
tv.blog.itaadf.cn/Article/details/795913.sHtML
tv.blog.itaadf.cn/Article/details/048882.sHtML
tv.blog.itaadf.cn/Article/details/484408.sHtML
tv.blog.itaadf.cn/Article/details/606682.sHtML
tv.blog.itaadf.cn/Article/details/462620.sHtML
tv.blog.itaadf.cn/Article/details/068202.sHtML
tv.blog.itaadf.cn/Article/details/624060.sHtML
tv.blog.itaadf.cn/Article/details/535119.sHtML
tv.blog.itaadf.cn/Article/details/353337.sHtML
tv.blog.itaadf.cn/Article/details/515595.sHtML
tv.blog.itaadf.cn/Article/details/802080.sHtML
tv.blog.itaadf.cn/Article/details/006448.sHtML
tv.blog.itaadf.cn/Article/details/751717.sHtML
tv.blog.itaadf.cn/Article/details/771331.sHtML
tv.blog.itaadf.cn/Article/details/042882.sHtML
tv.blog.itaadf.cn/Article/details/288262.sHtML
tv.blog.itaadf.cn/Article/details/640462.sHtML
tv.blog.itaadf.cn/Article/details/042068.sHtML
tv.blog.itaadf.cn/Article/details/802826.sHtML
tv.blog.itaadf.cn/Article/details/460604.sHtML
tv.blog.itaadf.cn/Article/details/204864.sHtML
tv.blog.itaadf.cn/Article/details/599571.sHtML
tv.blog.itaadf.cn/Article/details/426840.sHtML
tv.blog.itaadf.cn/Article/details/680206.sHtML
tv.blog.itaadf.cn/Article/details/860482.sHtML
tv.blog.itaadf.cn/Article/details/608866.sHtML
tv.blog.itaadf.cn/Article/details/408828.sHtML
tv.blog.itaadf.cn/Article/details/640644.sHtML
tv.blog.itaadf.cn/Article/details/484020.sHtML
tv.blog.itaadf.cn/Article/details/488440.sHtML
tv.blog.itaadf.cn/Article/details/933135.sHtML
tv.blog.itaadf.cn/Article/details/000262.sHtML
tv.blog.itaadf.cn/Article/details/668442.sHtML
tv.blog.itaadf.cn/Article/details/153911.sHtML
tv.blog.itaadf.cn/Article/details/133595.sHtML
tv.blog.itaadf.cn/Article/details/268484.sHtML
tv.blog.itaadf.cn/Article/details/804808.sHtML
tv.blog.itaadf.cn/Article/details/428088.sHtML
tv.blog.itaadf.cn/Article/details/880820.sHtML
tv.blog.itaadf.cn/Article/details/640400.sHtML
tv.blog.itaadf.cn/Article/details/999195.sHtML
tv.blog.itaadf.cn/Article/details/202064.sHtML
tv.blog.itaadf.cn/Article/details/268822.sHtML
tv.blog.itaadf.cn/Article/details/464820.sHtML
tv.blog.itaadf.cn/Article/details/600420.sHtML
tv.blog.itaadf.cn/Article/details/060408.sHtML
tv.blog.itaadf.cn/Article/details/228800.sHtML
tv.blog.itaadf.cn/Article/details/264024.sHtML
tv.blog.itaadf.cn/Article/details/866422.sHtML
tv.blog.itaadf.cn/Article/details/391537.sHtML
tv.blog.itaadf.cn/Article/details/066444.sHtML
tv.blog.itaadf.cn/Article/details/060244.sHtML
tv.blog.itaadf.cn/Article/details/113593.sHtML
tv.blog.itaadf.cn/Article/details/535971.sHtML
tv.blog.itaadf.cn/Article/details/808020.sHtML
tv.blog.itaadf.cn/Article/details/422486.sHtML
tv.blog.itaadf.cn/Article/details/646040.sHtML
tv.blog.itaadf.cn/Article/details/660422.sHtML
tv.blog.itaadf.cn/Article/details/951375.sHtML
tv.blog.itaadf.cn/Article/details/551733.sHtML
tv.blog.itaadf.cn/Article/details/195577.sHtML
tv.blog.itaadf.cn/Article/details/193919.sHtML
tv.blog.itaadf.cn/Article/details/664024.sHtML
tv.blog.itaadf.cn/Article/details/206040.sHtML
tv.blog.itaadf.cn/Article/details/511331.sHtML
tv.blog.itaadf.cn/Article/details/719317.sHtML
tv.blog.itaadf.cn/Article/details/026480.sHtML
tv.blog.itaadf.cn/Article/details/371133.sHtML
tv.blog.itaadf.cn/Article/details/662202.sHtML
tv.blog.itaadf.cn/Article/details/846886.sHtML
tv.blog.itaadf.cn/Article/details/482284.sHtML
tv.blog.itaadf.cn/Article/details/842226.sHtML
tv.blog.itaadf.cn/Article/details/800602.sHtML
tv.blog.itaadf.cn/Article/details/646448.sHtML
tv.blog.itaadf.cn/Article/details/840266.sHtML
tv.blog.itaadf.cn/Article/details/977151.sHtML
tv.blog.itaadf.cn/Article/details/662228.sHtML
tv.blog.itaadf.cn/Article/details/488648.sHtML
tv.blog.itaadf.cn/Article/details/624462.sHtML
tv.blog.itaadf.cn/Article/details/131715.sHtML
tv.blog.itaadf.cn/Article/details/628880.sHtML
tv.blog.itaadf.cn/Article/details/842426.sHtML
tv.blog.itaadf.cn/Article/details/775739.sHtML
tv.blog.itaadf.cn/Article/details/624406.sHtML
tv.blog.itaadf.cn/Article/details/379553.sHtML
tv.blog.itaadf.cn/Article/details/551153.sHtML
tv.blog.itaadf.cn/Article/details/626002.sHtML
tv.blog.itaadf.cn/Article/details/519319.sHtML
tv.blog.itaadf.cn/Article/details/688826.sHtML
tv.blog.itaadf.cn/Article/details/002066.sHtML
tv.blog.itaadf.cn/Article/details/804824.sHtML
tv.blog.itaadf.cn/Article/details/575733.sHtML
tv.blog.itaadf.cn/Article/details/971719.sHtML
tv.blog.itaadf.cn/Article/details/113915.sHtML
tv.blog.itaadf.cn/Article/details/204820.sHtML
tv.blog.itaadf.cn/Article/details/842666.sHtML
tv.blog.itaadf.cn/Article/details/597197.sHtML
tv.blog.itaadf.cn/Article/details/244062.sHtML
tv.blog.itaadf.cn/Article/details/240028.sHtML
tv.blog.itaadf.cn/Article/details/537735.sHtML
tv.blog.itaadf.cn/Article/details/224264.sHtML
tv.blog.itaadf.cn/Article/details/193957.sHtML
tv.blog.itaadf.cn/Article/details/193157.sHtML
tv.blog.itaadf.cn/Article/details/664024.sHtML
tv.blog.itaadf.cn/Article/details/755979.sHtML
tv.blog.itaadf.cn/Article/details/559919.sHtML
tv.blog.itaadf.cn/Article/details/179955.sHtML
tv.blog.itaadf.cn/Article/details/260648.sHtML
tv.blog.itaadf.cn/Article/details/351113.sHtML
tv.blog.itaadf.cn/Article/details/026048.sHtML
tv.blog.itaadf.cn/Article/details/420868.sHtML
tv.blog.itaadf.cn/Article/details/840844.sHtML
tv.blog.itaadf.cn/Article/details/339155.sHtML
tv.blog.itaadf.cn/Article/details/979995.sHtML
tv.blog.itaadf.cn/Article/details/286426.sHtML
tv.blog.itaadf.cn/Article/details/577173.sHtML
tv.blog.itaadf.cn/Article/details/379731.sHtML
tv.blog.itaadf.cn/Article/details/284266.sHtML
tv.blog.itaadf.cn/Article/details/088806.sHtML
tv.blog.itaadf.cn/Article/details/880888.sHtML
tv.blog.itaadf.cn/Article/details/080284.sHtML
tv.blog.itaadf.cn/Article/details/777777.sHtML
tv.blog.itaadf.cn/Article/details/060808.sHtML
tv.blog.itaadf.cn/Article/details/220488.sHtML
tv.blog.itaadf.cn/Article/details/482660.sHtML
tv.blog.itaadf.cn/Article/details/602024.sHtML
tv.blog.itaadf.cn/Article/details/400880.sHtML
tv.blog.itaadf.cn/Article/details/808062.sHtML
tv.blog.itaadf.cn/Article/details/646208.sHtML
tv.blog.itaadf.cn/Article/details/771193.sHtML
tv.blog.itaadf.cn/Article/details/715773.sHtML
tv.blog.itaadf.cn/Article/details/517515.sHtML
tv.blog.itaadf.cn/Article/details/088288.sHtML
tv.blog.itaadf.cn/Article/details/822482.sHtML
tv.blog.itaadf.cn/Article/details/486260.sHtML
tv.blog.itaadf.cn/Article/details/737759.sHtML
tv.blog.itaadf.cn/Article/details/606080.sHtML
tv.blog.itaadf.cn/Article/details/420022.sHtML
tv.blog.itaadf.cn/Article/details/848022.sHtML
tv.blog.itaadf.cn/Article/details/086828.sHtML
tv.blog.itaadf.cn/Article/details/040888.sHtML
tv.blog.itaadf.cn/Article/details/442488.sHtML
tv.blog.itaadf.cn/Article/details/046064.sHtML
tv.blog.itaadf.cn/Article/details/755913.sHtML
tv.blog.itaadf.cn/Article/details/753997.sHtML
tv.blog.itaadf.cn/Article/details/822422.sHtML
tv.blog.itaadf.cn/Article/details/939333.sHtML
tv.blog.itaadf.cn/Article/details/515135.sHtML
tv.blog.itaadf.cn/Article/details/082060.sHtML
tv.blog.itaadf.cn/Article/details/886288.sHtML
tv.blog.itaadf.cn/Article/details/244266.sHtML
tv.blog.itaadf.cn/Article/details/824066.sHtML
tv.blog.itaadf.cn/Article/details/248080.sHtML
tv.blog.itaadf.cn/Article/details/244626.sHtML
tv.blog.itaadf.cn/Article/details/048808.sHtML
tv.blog.itaadf.cn/Article/details/517153.sHtML
tv.blog.itaadf.cn/Article/details/028822.sHtML
tv.blog.itaadf.cn/Article/details/284444.sHtML
tv.blog.itaadf.cn/Article/details/464206.sHtML
tv.blog.itaadf.cn/Article/details/666466.sHtML
tv.blog.itaadf.cn/Article/details/171577.sHtML
tv.blog.itaadf.cn/Article/details/446600.sHtML
tv.blog.itaadf.cn/Article/details/068662.sHtML
tv.blog.itaadf.cn/Article/details/664222.sHtML
tv.blog.itaadf.cn/Article/details/286244.sHtML
tv.blog.itaadf.cn/Article/details/884400.sHtML
tv.blog.itaadf.cn/Article/details/026604.sHtML
tv.blog.itaadf.cn/Article/details/331937.sHtML
tv.blog.itaadf.cn/Article/details/606422.sHtML
tv.blog.itaadf.cn/Article/details/664868.sHtML
tv.blog.itaadf.cn/Article/details/824642.sHtML
tv.blog.itaadf.cn/Article/details/680666.sHtML
tv.blog.itaadf.cn/Article/details/202482.sHtML
tv.blog.itaadf.cn/Article/details/464466.sHtML
tv.blog.itaadf.cn/Article/details/624282.sHtML
tv.blog.itaadf.cn/Article/details/571371.sHtML
tv.blog.itaadf.cn/Article/details/979975.sHtML
tv.blog.itaadf.cn/Article/details/448400.sHtML
tv.blog.itaadf.cn/Article/details/377713.sHtML
tv.blog.itaadf.cn/Article/details/060824.sHtML
tv.blog.itaadf.cn/Article/details/266844.sHtML
tv.blog.itaadf.cn/Article/details/337919.sHtML
tv.blog.itaadf.cn/Article/details/406200.sHtML
tv.blog.itaadf.cn/Article/details/668820.sHtML
tv.blog.itaadf.cn/Article/details/840408.sHtML
tv.blog.itaadf.cn/Article/details/955933.sHtML
tv.blog.itaadf.cn/Article/details/866608.sHtML
tv.blog.itaadf.cn/Article/details/997599.sHtML
tv.blog.itaadf.cn/Article/details/840022.sHtML
tv.blog.itaadf.cn/Article/details/973333.sHtML
tv.blog.itaadf.cn/Article/details/240228.sHtML
tv.blog.itaadf.cn/Article/details/391557.sHtML
tv.blog.itaadf.cn/Article/details/828400.sHtML
tv.blog.itaadf.cn/Article/details/937531.sHtML
tv.blog.itaadf.cn/Article/details/937555.sHtML
tv.blog.itaadf.cn/Article/details/244020.sHtML
tv.blog.itaadf.cn/Article/details/408442.sHtML
tv.blog.itaadf.cn/Article/details/733953.sHtML
tv.blog.itaadf.cn/Article/details/604846.sHtML
tv.blog.itaadf.cn/Article/details/004400.sHtML
tv.blog.itaadf.cn/Article/details/171377.sHtML
tv.blog.itaadf.cn/Article/details/888646.sHtML
tv.blog.itaadf.cn/Article/details/193311.sHtML
tv.blog.itaadf.cn/Article/details/202684.sHtML
tv.blog.itaadf.cn/Article/details/022242.sHtML
tv.blog.itaadf.cn/Article/details/488204.sHtML
tv.blog.itaadf.cn/Article/details/880644.sHtML
tv.blog.itaadf.cn/Article/details/622224.sHtML
tv.blog.itaadf.cn/Article/details/088682.sHtML
tv.blog.itaadf.cn/Article/details/848428.sHtML
tv.blog.itaadf.cn/Article/details/204406.sHtML
tv.blog.itaadf.cn/Article/details/171935.sHtML
tv.blog.itaadf.cn/Article/details/862888.sHtML
tv.blog.itaadf.cn/Article/details/624646.sHtML
tv.blog.itaadf.cn/Article/details/662846.sHtML
tv.blog.itaadf.cn/Article/details/800200.sHtML
tv.blog.itaadf.cn/Article/details/353553.sHtML
tv.blog.itaadf.cn/Article/details/977155.sHtML
tv.blog.itaadf.cn/Article/details/355199.sHtML
tv.blog.itaadf.cn/Article/details/640604.sHtML
tv.blog.itaadf.cn/Article/details/608066.sHtML
tv.blog.itaadf.cn/Article/details/820048.sHtML
tv.blog.itaadf.cn/Article/details/048808.sHtML
tv.blog.itaadf.cn/Article/details/662644.sHtML
tv.blog.itaadf.cn/Article/details/006602.sHtML
tv.blog.itaadf.cn/Article/details/862684.sHtML
tv.blog.itaadf.cn/Article/details/804484.sHtML
tv.blog.itaadf.cn/Article/details/022668.sHtML
tv.blog.itaadf.cn/Article/details/242048.sHtML
tv.blog.itaadf.cn/Article/details/371517.sHtML
tv.blog.itaadf.cn/Article/details/119539.sHtML
tv.blog.itaadf.cn/Article/details/331977.sHtML
tv.blog.itaadf.cn/Article/details/751155.sHtML
tv.blog.itaadf.cn/Article/details/868062.sHtML
tv.blog.itaadf.cn/Article/details/824646.sHtML
tv.blog.itaadf.cn/Article/details/206268.sHtML
tv.blog.itaadf.cn/Article/details/462060.sHtML
tv.blog.itaadf.cn/Article/details/046000.sHtML
tv.blog.itaadf.cn/Article/details/248868.sHtML
tv.blog.itaadf.cn/Article/details/119131.sHtML
tv.blog.itaadf.cn/Article/details/246000.sHtML
tv.blog.itaadf.cn/Article/details/284228.sHtML
tv.blog.itaadf.cn/Article/details/280640.sHtML
tv.blog.itaadf.cn/Article/details/977193.sHtML
tv.blog.itaadf.cn/Article/details/939193.sHtML
tv.blog.itaadf.cn/Article/details/668000.sHtML
tv.blog.itaadf.cn/Article/details/202426.sHtML
tv.blog.itaadf.cn/Article/details/680840.sHtML
tv.blog.itaadf.cn/Article/details/759379.sHtML
tv.blog.itaadf.cn/Article/details/004080.sHtML
tv.blog.itaadf.cn/Article/details/688064.sHtML
tv.blog.itaadf.cn/Article/details/179335.sHtML
tv.blog.itaadf.cn/Article/details/446642.sHtML
tv.blog.itaadf.cn/Article/details/420466.sHtML
tv.blog.itaadf.cn/Article/details/713571.sHtML
tv.blog.itaadf.cn/Article/details/339711.sHtML
tv.blog.itaadf.cn/Article/details/286608.sHtML
tv.blog.itaadf.cn/Article/details/280882.sHtML
tv.blog.itaadf.cn/Article/details/333977.sHtML
tv.blog.itaadf.cn/Article/details/608802.sHtML
tv.blog.itaadf.cn/Article/details/680602.sHtML
tv.blog.itaadf.cn/Article/details/068488.sHtML
tv.blog.itaadf.cn/Article/details/026204.sHtML
tv.blog.itaadf.cn/Article/details/068084.sHtML
tv.blog.itaadf.cn/Article/details/999115.sHtML
tv.blog.itaadf.cn/Article/details/393117.sHtML
tv.blog.itaadf.cn/Article/details/060284.sHtML
tv.blog.itaadf.cn/Article/details/286284.sHtML
tv.blog.itaadf.cn/Article/details/462084.sHtML
tv.blog.itaadf.cn/Article/details/957997.sHtML
tv.blog.itaadf.cn/Article/details/882824.sHtML
tv.blog.itaadf.cn/Article/details/242080.sHtML
tv.blog.itaadf.cn/Article/details/226680.sHtML
tv.blog.itaadf.cn/Article/details/620864.sHtML
tv.blog.itaadf.cn/Article/details/713993.sHtML
tv.blog.itaadf.cn/Article/details/220088.sHtML
tv.blog.itaadf.cn/Article/details/911775.sHtML
tv.blog.itaadf.cn/Article/details/028060.sHtML
tv.blog.itaadf.cn/Article/details/333171.sHtML
tv.blog.itaadf.cn/Article/details/066282.sHtML
tv.blog.itaadf.cn/Article/details/088800.sHtML
tv.blog.itaadf.cn/Article/details/884482.sHtML
tv.blog.itaadf.cn/Article/details/555937.sHtML
tv.blog.itaadf.cn/Article/details/753153.sHtML
tv.blog.itaadf.cn/Article/details/755313.sHtML
tv.blog.itaadf.cn/Article/details/513355.sHtML
tv.blog.itaadf.cn/Article/details/959951.sHtML
tv.blog.itaadf.cn/Article/details/860620.sHtML
tv.blog.itaadf.cn/Article/details/175731.sHtML
tv.blog.itaadf.cn/Article/details/175753.sHtML
tv.blog.itaadf.cn/Article/details/802684.sHtML
tv.blog.itaadf.cn/Article/details/084668.sHtML
tv.blog.itaadf.cn/Article/details/311573.sHtML
tv.blog.itaadf.cn/Article/details/571173.sHtML
tv.blog.itaadf.cn/Article/details/426086.sHtML
tv.blog.itaadf.cn/Article/details/377791.sHtML
tv.blog.itaadf.cn/Article/details/646200.sHtML
tv.blog.itaadf.cn/Article/details/000660.sHtML
tv.blog.itaadf.cn/Article/details/224002.sHtML
tv.blog.itaadf.cn/Article/details/868222.sHtML
tv.blog.itaadf.cn/Article/details/480488.sHtML
tv.blog.itaadf.cn/Article/details/408666.sHtML
tv.blog.itaadf.cn/Article/details/642684.sHtML
tv.blog.itaadf.cn/Article/details/280404.sHtML
tv.blog.itaadf.cn/Article/details/880224.sHtML
tv.blog.itaadf.cn/Article/details/882222.sHtML
tv.blog.itaadf.cn/Article/details/804060.sHtML
tv.blog.itaadf.cn/Article/details/028480.sHtML
tv.blog.itaadf.cn/Article/details/535117.sHtML
tv.blog.itaadf.cn/Article/details/662082.sHtML
tv.blog.itaadf.cn/Article/details/602640.sHtML
tv.blog.itaadf.cn/Article/details/408404.sHtML
tv.blog.itaadf.cn/Article/details/755919.sHtML
tv.blog.itaadf.cn/Article/details/775915.sHtML
tv.blog.itaadf.cn/Article/details/959951.sHtML
tv.blog.itaadf.cn/Article/details/426204.sHtML
tv.blog.itaadf.cn/Article/details/660402.sHtML
tv.blog.itaadf.cn/Article/details/208408.sHtML
tv.blog.itaadf.cn/Article/details/151195.sHtML
tv.blog.itaadf.cn/Article/details/064222.sHtML
tv.blog.itaadf.cn/Article/details/804248.sHtML
tv.blog.itaadf.cn/Article/details/064808.sHtML
tv.blog.itaadf.cn/Article/details/555995.sHtML
tv.blog.itaadf.cn/Article/details/826482.sHtML
tv.blog.itaadf.cn/Article/details/795753.sHtML
tv.blog.itaadf.cn/Article/details/571171.sHtML
tv.blog.itaadf.cn/Article/details/848424.sHtML
tv.blog.itaadf.cn/Article/details/886222.sHtML
tv.blog.itaadf.cn/Article/details/597355.sHtML
tv.blog.itaadf.cn/Article/details/240028.sHtML
tv.blog.itaadf.cn/Article/details/086828.sHtML
tv.blog.itaadf.cn/Article/details/880828.sHtML
tv.blog.itaadf.cn/Article/details/006280.sHtML
tv.blog.itaadf.cn/Article/details/440886.sHtML
tv.blog.itaadf.cn/Article/details/462268.sHtML
tv.blog.itaadf.cn/Article/details/735797.sHtML
tv.blog.itaadf.cn/Article/details/020664.sHtML
tv.blog.itaadf.cn/Article/details/626804.sHtML
tv.blog.itaadf.cn/Article/details/644008.sHtML
tv.blog.itaadf.cn/Article/details/555315.sHtML
tv.blog.itaadf.cn/Article/details/911553.sHtML
tv.blog.itaadf.cn/Article/details/848864.sHtML
tv.blog.itaadf.cn/Article/details/208262.sHtML
tv.blog.itaadf.cn/Article/details/802228.sHtML
tv.blog.itaadf.cn/Article/details/115715.sHtML
tv.blog.itaadf.cn/Article/details/626046.sHtML
tv.blog.itaadf.cn/Article/details/442426.sHtML
