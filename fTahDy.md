Python 数据处理与分析基础
===============================

Python 拥有强大的数据处理能力，即使不借助 pandas 等第三方库，标准库也
提供了丰富的数据处理工具。本文介绍使用标准库进行数据处理的核心技巧。

一、csv 模块 —— 读写 CSV 文件
------------------------------

CSV（逗号分隔值）是最常见的数据交换格式之一。csv 模块提供了方便的读写接口。

import csv
from io import StringIO

# 示例数据：员工信息
sample_data = """姓名,部门,工资,入职日期
张三,技术部,15000,2023-01-15
李四,市场部,12000,2022-06-20
王五,技术部,18000,2021-03-10
赵六,人事部,11000,2023-09-01
"""

# 使用 DictReader 将 CSV 读取为字典列表（每行一个字典）
def read_csv_as_dicts(csv_string):
    """使用 DictReader 读取 CSV 数据，返回字典列表"""
    reader = csv.DictReader(StringIO(csv_string))
    # reader 的 fieldnames 属性自动从第一行获取列名
    print(f"列名: {reader.fieldnames}")
    data = []
    for row in reader:
        # 每个 row 是一个 OrderedDict，键为列名
        data.append(dict(row))
    return data

employees = read_csv_as_dicts(sample_data)
print(f"员工总数: {len(employees)}")
for emp in employees:
    print(f"  {emp['姓名']} - {emp['部门']} - {emp['工资']}元")

# 使用 DictWriter 写入 CSV
def write_csv_from_dicts(filename, data, fieldnames):
    """将字典列表写入 CSV 文件"""
    with open(filename, "w", newline="", encoding="utf-8") as f:
        writer = csv.DictWriter(f, fieldnames=fieldnames)
        writer.writeheader()  # 写入列名行
        writer.writerows(data)  # 写入所有数据行

# 计算各部门平均工资
def calculate_dept_avg(employees):
    """按部门统计平均工资"""
    dept_data = {}
    for emp in employees:
        dept = emp["部门"]
        salary = int(emp["工资"])
        if dept not in dept_data:
            dept_data[dept] = []
        dept_data[dept].append(salary)
    # 计算每个部门的平均工资
    result = []
    for dept, salaries in dept_data.items():
        avg = sum(salaries) / len(salaries)
        result.append({"部门": dept, "平均工资": round(avg, 2), "人数": len(salaries)})
    return result

avg_by_dept = calculate_dept_avg(employees)
print("\n各部门平均工资:")
for row in avg_by_dept:
    print(f"  {row['部门']}: {row['平均工资']}元（共{row['人数']}人）")

# 写入计算结果
with open("/tmp/dept_avg.csv", "w", newline="", encoding="utf-8") as f:
    writer = csv.DictWriter(f, fieldnames=["部门", "平均工资", "人数"])
    writer.writeheader()
    writer.writerows(avg_by_dept)

二、json 模块 —— 高级 JSON 处理
--------------------------------

json 模块不仅支持基本的 load/dump，还支持自定义编码器以处理特殊类型。

import json
from datetime import datetime, date
from decimal import Decimal

# 自定义 JSON 编码器：处理 Python 特有的数据类型
class CustomJSONEncoder(json.JSONEncoder):
    """支持 datetime、date、Decimal、set 等类型的 JSON 编码器"""
    def default(self, obj):
        if isinstance(obj, datetime):
            return {"__type__": "datetime", "value": obj.isoformat()}
        if isinstance(obj, date):
            return obj.isoformat()
        if isinstance(obj, Decimal):
            return float(obj)  # 或使用 str(obj) 保留精度
        if isinstance(obj, set):
            return list(obj)
        if isinstance(obj, bytes):
            return obj.decode("utf-8", errors="replace")
        return super().default(obj)

# 自定义 JSON 解码器：恢复自定义类型
class CustomJSONDecoder(json.JSONDecoder):
    def __init__(self):
        super().__init__(object_hook=self.object_hook)

    def object_hook(self, d):
        """在解析每个 JSON 对象时调用，可自定义类型恢复"""
        if "__type__" in d and d["__type__"] == "datetime":
            return datetime.fromisoformat(d["value"])
        return d

# 示例：处理包含特殊类型的数据
data = {
    "event": "用户注册",
    "timestamp": datetime.now(),
    "score": Decimal("99.95"),
    "tags": {"vip", "new_user", "active"},  # set 类型
    "metadata": b"binary info",              # bytes 类型
}

# 将复杂对象序列化为 JSON 字符串
json_str = json.dumps(data, cls=CustomJSONEncoder, ensure_ascii=False, indent=2)
print("序列化后的 JSON:")
print(json_str)

# 从 JSON 字符串反序列化
restored = json.loads(json_str, cls=CustomJSONDecoder)
print(f"\n反序列化后的类型:")
print(f"  timestamp 类型: {type(restored['timestamp'])}")
print(f"  score 类型: {type(restored['score'])}")

三、itertools.groupby —— 数据分组利器
---------------------------------------

itertools.groupby 可以对已排序的数据进行分组，非常适合数据聚合分析。

from itertools import groupby
from operator import itemgetter

# 员工数据
employees = [
    {"name": "张三", "dept": "技术部", "salary": 15000},
    {"name": "李四", "dept": "市场部", "salary": 12000},
    {"name": "王五", "dept": "技术部", "salary": 18000},
    {"name": "赵六", "dept": "人事部", "salary": 11000},
    {"name": "钱七", "dept": "市场部", "salary": 13000},
    {"name": "孙八", "dept": "技术部", "salary": 16000},
]

# groupby 要求数据先按分组键排序！
sorted_employees = sorted(employees, key=itemgetter("dept"))

# 按部门分组并计算每个部门的工资统计
print("\n按部门分组统计:")
for dept, group in groupby(sorted_employees, key=itemgetter("dept")):
    group_list = list(group)
    salaries = [emp["salary"] for emp in group_list]
    print(f"\n{dept}:")
    print(f"  员工: {[emp['name'] for emp in group_list]}")
    print(f"  最高工资: {max(salaries)}")
    print(f"  最低工资: {min(salaries)}")
    print(f"  平均工资: {sum(salaries) / len(salaries):.0f}")

# 进阶：分段统计（将工资分成高、中、低三档）
def salary_bracket(emp):
    """根据工资水平返回对应的档位"""
    salary = emp["salary"]
    if salary >= 16000:
        return "高薪"
    elif salary >= 13000:
        return "中薪"
    else:
        return "低薪"

# 分段前需要先排序
sorted_by_bracket = sorted(employees, key=salary_bracket)
print("\n按工资水平分组:")
for bracket, group in groupby(sorted_by_bracket, key=salary_bracket):
    count = len(list(group))
    print(f"  {bracket}: {count}人")

四、collections.Counter —— 频率分析
-------------------------------------

Counter 是进行频率统计和分析的首选工具。

from collections import Counter

# 分析文本中的词频
text = """
Python 是一种高级编程语言。Python 的设计哲学强调代码的可读性。
Python 支持多种编程范式，包括面向对象、函数式和过程式编程。
Python 拥有庞大的标准库和活跃的社区。
"""

# 简单的词频统计
words = text.split()
word_count = Counter(words)
print("\n词频统计（Top 5）:")
for word, count in word_count.most_common(5):
    print(f"  '{word}': {count}次")

# 统计字符出现频率（忽略空格）
chars = [c for c in text if c.strip()]
char_count = Counter(chars)
print(f"\n字符总数: {sum(char_count.values())}")
print(f"不同字符数: {len(char_count)}")
print(f"最常见字符: '{char_count.most_common(1)[0][0]}'")

# Counter 的数学运算：适合多组数据对比
sales_day1 = Counter({"苹果": 10, "香蕉": 5, "橙子": 8})
sales_day2 = Counter({"苹果": 8, "香蕉": 7, "葡萄": 3})

total_sales = sales_day1 + sales_day2      # 合并两天的销量
print(f"\n总销量: {dict(total_sales)}")
diff = sales_day1 - sales_day2             # 第一天的哪些水果卖得更多
print(f"第一天多卖的: {dict(diff)}")

五、datetime 模块 —— 日期时间解析与处理
-----------------------------------------

from datetime import datetime, timedelta, date

# 解析多种格式的日期字符串
date_strings = [
    "2024-01-15",
    "2024/03/20 14:30:00",
    "2024年5月1日",
    "2024-12-25T10:00:00+08:00",
]

def parse_date(date_str):
    """尝试用多种格式解析日期字符串"""
    formats = [
        "%Y-%m-%d",
        "%Y/%m/%d %H:%M:%S",
        "%Y年%m月%d日",
        "%Y-%m-%dT%H:%M:%S%z",
    ]
    for fmt in formats:
        try:
            return datetime.strptime(date_str, fmt)
        except ValueError:
            continue
    raise ValueError(f"无法解析日期: {date_str}")

print("\n日期解析:")
for ds in date_strings:
    dt = parse_date(ds)
    print(f"  {ds} -> {dt}")

# 日期计算示例：计算工作日天数
def workdays_between(start, end):
    """计算两个日期之间的工作日天数（排除周末）"""
    days = 0
    current = start
    while current <= end:
        if current.weekday() < 5:  # Monday=0, Sunday=6
            days += 1
        current += timedelta(days=1)
    return days

start = date(2024, 1, 1)
end = date(2024, 1, 31)
print(f"\n{start} 到 {end} 的工作日天数: {workdays_between(start, end)}")

# 日期格式化输出
now = datetime.now()
print(f"\n当前时间: {now}")
print(f"格式化的日期: {now.strftime('%Y年%m月%d日 %H:%M:%S')}")
print(f"ISO 格式: {now.isoformat()}")

六、dataclasses —— 简化的数据验证
----------------------------------

dataclasses 模块提供了一种简洁的方式来定义数据容器，结合类型提示和验证。

from dataclasses import dataclass, field, asdict
from typing import Optional
import re

@dataclass
class Employee:
    """员工数据模型，包含基本的字段验证"""
    name: str
    age: int
    email: str
    department: str = "未分配"
    salary: float = 0.0
    tags: list[str] = field(default_factory=list)  # 可变默认值必须用 field

    def __post_init__(self):
        """初始化后的自动验证（dataclass 自动调用此方法）"""
        # 验证姓名非空
        if not self.name or len(self.name.strip()) == 0:
            raise ValueError("姓名不能为空")
        # 验证年龄范围
        if not (18 <= self.age <= 65):
            raise ValueError(f"年龄必须在 18-65 之间，当前值: {self.age}")
        # 验证邮箱格式
        email_pattern = r"^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$"
        if not re.match(email_pattern, self.email):
            raise ValueError(f"邮箱格式不正确: {self.email}")
        # 验证工资
        if self.salary < 0:
            raise ValueError("工资不能为负数")

    @property
    def salary_level(self):
        """根据工资水平返回描述"""
        if self.salary >= 20000:
            return "高级"
        elif self.salary >= 10000:
            return "中级"
        else:
            return "初级"

# 使用示例（数据验证在创建时自动执行）
try:
    emp = Employee(name="张三", age=25, email="zhangsan@example.com",
                   department="技术部", salary=15000)
    print(f"\n创建员工成功: {emp}")
    print(f"工资水平: {emp.salary_level}")
    print(f"转为字典: {asdict(emp)}")

    # 尝试创建无效数据
    invalid_emp = Employee(name="", age=17, email="bad-email")
except ValueError as e:
    print(f"验证失败: {e}")

# 结合 CSV 导入：将 CSV 数据直接转换为 Employee 实例
def import_employees_from_csv(csv_path):
    """从 CSV 文件导入员工数据并验证"""
    employees = []
    with open(csv_path, "r", encoding="utf-8") as f:
        reader = csv.DictReader(f)
        for row in reader:
            try:
                emp = Employee(
                    name=row["姓名"],
                    age=int(row["年龄"]),
                    email=row["邮箱"],
                    department=row.get("部门", "未分配"),
                    salary=float(row.get("工资", 0)),
                )
                employees.append(emp)
            except (ValueError, KeyError) as e:
                print(f"数据行 {reader.line_num} 导入失败: {e}")
    return employees

总结：Python 标准库提供了 csv、json、itertools、collections、datetime
和 dataclasses 等强大的数据处理工具。掌握这些模块后，即使不依赖 pandas
等第三方库，也能完成大部分常见的数据处理和分析任务。

qyr.guohua888.cn/00062.Doc
qyr.guohua888.cn/28264.Doc
qyr.guohua888.cn/82804.Doc
qyr.guohua888.cn/71317.Doc
qyr.guohua888.cn/15577.Doc
qyr.guohua888.cn/66248.Doc
qyr.guohua888.cn/51199.Doc
qyr.guohua888.cn/82246.Doc
qyr.guohua888.cn/22062.Doc
qyr.guohua888.cn/44284.Doc
qye.guohua888.cn/20066.Doc
qye.guohua888.cn/48808.Doc
qye.guohua888.cn/28486.Doc
qye.guohua888.cn/04008.Doc
qye.guohua888.cn/04824.Doc
qye.guohua888.cn/64882.Doc
qye.guohua888.cn/80480.Doc
qye.guohua888.cn/00688.Doc
qye.guohua888.cn/64224.Doc
qye.guohua888.cn/00648.Doc
qyw.guohua888.cn/22826.Doc
qyw.guohua888.cn/64200.Doc
qyw.guohua888.cn/84626.Doc
qyw.guohua888.cn/86262.Doc
qyw.guohua888.cn/28086.Doc
qyw.guohua888.cn/48464.Doc
qyw.guohua888.cn/88680.Doc
qyw.guohua888.cn/08022.Doc
qyw.guohua888.cn/48006.Doc
qyw.guohua888.cn/42400.Doc
qyq.guohua888.cn/64408.Doc
qyq.guohua888.cn/00468.Doc
qyq.guohua888.cn/82026.Doc
qyq.guohua888.cn/82244.Doc
qyq.guohua888.cn/68220.Doc
qyq.guohua888.cn/84020.Doc
qyq.guohua888.cn/20028.Doc
qyq.guohua888.cn/48404.Doc
qyq.guohua888.cn/64442.Doc
qyq.guohua888.cn/42040.Doc
qtm.guohua888.cn/46644.Doc
qtm.guohua888.cn/22022.Doc
qtm.guohua888.cn/04220.Doc
qtm.guohua888.cn/06448.Doc
qtm.guohua888.cn/82808.Doc
qtm.guohua888.cn/42266.Doc
qtm.guohua888.cn/00466.Doc
qtm.guohua888.cn/64000.Doc
qtm.guohua888.cn/57791.Doc
qtm.guohua888.cn/40248.Doc
qtn.guohua888.cn/40802.Doc
qtn.guohua888.cn/11577.Doc
qtn.guohua888.cn/40406.Doc
qtn.guohua888.cn/53559.Doc
qtn.guohua888.cn/55991.Doc
qtn.guohua888.cn/48488.Doc
qtn.guohua888.cn/60424.Doc
qtn.guohua888.cn/06082.Doc
qtn.guohua888.cn/51313.Doc
qtn.guohua888.cn/73955.Doc
qtb.guohua888.cn/48086.Doc
qtb.guohua888.cn/77117.Doc
qtb.guohua888.cn/17919.Doc
qtb.guohua888.cn/26044.Doc
qtb.guohua888.cn/11135.Doc
qtb.guohua888.cn/08848.Doc
qtb.guohua888.cn/57511.Doc
qtb.guohua888.cn/44006.Doc
qtb.guohua888.cn/55551.Doc
qtb.guohua888.cn/60082.Doc
qtv.guohua888.cn/22840.Doc
qtv.guohua888.cn/26248.Doc
qtv.guohua888.cn/62820.Doc
qtv.guohua888.cn/62686.Doc
qtv.guohua888.cn/04242.Doc
qtv.guohua888.cn/24000.Doc
qtv.guohua888.cn/24064.Doc
qtv.guohua888.cn/20022.Doc
qtv.guohua888.cn/42862.Doc
qtv.guohua888.cn/22040.Doc
qtc.guohua888.cn/02008.Doc
qtc.guohua888.cn/71915.Doc
qtc.guohua888.cn/20844.Doc
qtc.guohua888.cn/44006.Doc
qtc.guohua888.cn/59911.Doc
qtc.guohua888.cn/39755.Doc
qtc.guohua888.cn/48826.Doc
qtc.guohua888.cn/55313.Doc
qtc.guohua888.cn/77955.Doc
qtc.guohua888.cn/95995.Doc
qtx.guohua888.cn/73311.Doc
qtx.guohua888.cn/79739.Doc
qtx.guohua888.cn/59591.Doc
qtx.guohua888.cn/19759.Doc
qtx.guohua888.cn/59135.Doc
qtx.guohua888.cn/39731.Doc
qtx.guohua888.cn/33515.Doc
qtx.guohua888.cn/00080.Doc
qtx.guohua888.cn/95991.Doc
qtx.guohua888.cn/75999.Doc
qtz.guohua888.cn/68888.Doc
qtz.guohua888.cn/17139.Doc
qtz.guohua888.cn/97995.Doc
qtz.guohua888.cn/91557.Doc
qtz.guohua888.cn/33397.Doc
qtz.guohua888.cn/75113.Doc
qtz.guohua888.cn/15733.Doc
qtz.guohua888.cn/91735.Doc
qtz.guohua888.cn/24888.Doc
qtz.guohua888.cn/62460.Doc
qtl.guohua888.cn/53319.Doc
qtl.guohua888.cn/97313.Doc
qtl.guohua888.cn/11573.Doc
qtl.guohua888.cn/93379.Doc
qtl.guohua888.cn/53531.Doc
qtl.guohua888.cn/33751.Doc
qtl.guohua888.cn/53317.Doc
qtl.guohua888.cn/93971.Doc
qtl.guohua888.cn/19591.Doc
qtl.guohua888.cn/94480.Doc
qtk.guohua888.cn/55157.Doc
qtk.guohua888.cn/73331.Doc
qtk.guohua888.cn/53797.Doc
qtk.guohua888.cn/02020.Doc
qtk.guohua888.cn/55917.Doc
qtk.guohua888.cn/26406.Doc
qtk.guohua888.cn/13157.Doc
qtk.guohua888.cn/82662.Doc
qtk.guohua888.cn/46624.Doc
qtk.guohua888.cn/02844.Doc
qtj.guohua888.cn/64800.Doc
qtj.guohua888.cn/91117.Doc
qtj.guohua888.cn/64280.Doc
qtj.guohua888.cn/53791.Doc
qtj.guohua888.cn/73357.Doc
qtj.guohua888.cn/97159.Doc
qtj.guohua888.cn/84424.Doc
qtj.guohua888.cn/02062.Doc
qtj.guohua888.cn/51751.Doc
qtj.guohua888.cn/84226.Doc
qth.guohua888.cn/60840.Doc
qth.guohua888.cn/48824.Doc
qth.guohua888.cn/00604.Doc
qth.guohua888.cn/00084.Doc
qth.guohua888.cn/24480.Doc
qth.guohua888.cn/26484.Doc
qth.guohua888.cn/84866.Doc
qth.guohua888.cn/82080.Doc
qth.guohua888.cn/68640.Doc
qth.guohua888.cn/17797.Doc
qtg.guohua888.cn/42080.Doc
qtg.guohua888.cn/08640.Doc
qtg.guohua888.cn/22246.Doc
qtg.guohua888.cn/20464.Doc
qtg.guohua888.cn/00626.Doc
qtg.guohua888.cn/20464.Doc
qtg.guohua888.cn/04422.Doc
qtg.guohua888.cn/66428.Doc
qtg.guohua888.cn/64864.Doc
qtg.guohua888.cn/68664.Doc
qtf.guohua888.cn/60840.Doc
qtf.guohua888.cn/44444.Doc
qtf.guohua888.cn/02420.Doc
qtf.guohua888.cn/62428.Doc
qtf.guohua888.cn/64088.Doc
qtf.guohua888.cn/28806.Doc
qtf.guohua888.cn/84662.Doc
qtf.guohua888.cn/42428.Doc
qtf.guohua888.cn/00008.Doc
qtf.guohua888.cn/46446.Doc
qtd.guohua888.cn/28428.Doc
qtd.guohua888.cn/82822.Doc
qtd.guohua888.cn/46020.Doc
qtd.guohua888.cn/20426.Doc
qtd.guohua888.cn/28460.Doc
qtd.guohua888.cn/44466.Doc
qtd.guohua888.cn/46868.Doc
qtd.guohua888.cn/80006.Doc
qtd.guohua888.cn/42460.Doc
qtd.guohua888.cn/40620.Doc
qts.guohua888.cn/40422.Doc
qts.guohua888.cn/46426.Doc
qts.guohua888.cn/24004.Doc
qts.guohua888.cn/00008.Doc
qts.guohua888.cn/68264.Doc
qts.guohua888.cn/60606.Doc
qts.guohua888.cn/42228.Doc
qts.guohua888.cn/77660.Doc
qts.guohua888.cn/00286.Doc
qts.guohua888.cn/28202.Doc
qta.guohua888.cn/62046.Doc
qta.guohua888.cn/46428.Doc
qta.guohua888.cn/02206.Doc
qta.guohua888.cn/46044.Doc
qta.guohua888.cn/48042.Doc
qta.guohua888.cn/40062.Doc
qta.guohua888.cn/22622.Doc
qta.guohua888.cn/62240.Doc
qta.guohua888.cn/06260.Doc
qta.guohua888.cn/84606.Doc
qtp.guohua888.cn/06448.Doc
qtp.guohua888.cn/08028.Doc
qtp.guohua888.cn/40004.Doc
qtp.guohua888.cn/00844.Doc
qtp.guohua888.cn/20280.Doc
qtp.guohua888.cn/64420.Doc
qtp.guohua888.cn/04060.Doc
qtp.guohua888.cn/66684.Doc
qtp.guohua888.cn/46040.Doc
qtp.guohua888.cn/06868.Doc
qto.guohua888.cn/26406.Doc
qto.guohua888.cn/48680.Doc
qto.guohua888.cn/88602.Doc
qto.guohua888.cn/99339.Doc
qto.guohua888.cn/08286.Doc
qto.guohua888.cn/44866.Doc
qto.guohua888.cn/82648.Doc
qto.guohua888.cn/64606.Doc
qto.guohua888.cn/28282.Doc
qto.guohua888.cn/95719.Doc
qti.guohua888.cn/22880.Doc
qti.guohua888.cn/60462.Doc
qti.guohua888.cn/20084.Doc
qti.guohua888.cn/28886.Doc
qti.guohua888.cn/48604.Doc
qti.guohua888.cn/80008.Doc
qti.guohua888.cn/60440.Doc
qti.guohua888.cn/42846.Doc
qti.guohua888.cn/60008.Doc
qti.guohua888.cn/62402.Doc
qtu.guohua888.cn/88006.Doc
qtu.guohua888.cn/22022.Doc
qtu.guohua888.cn/26862.Doc
qtu.guohua888.cn/88220.Doc
qtu.guohua888.cn/88486.Doc
qtu.guohua888.cn/60226.Doc
qtu.guohua888.cn/46898.Doc
qtu.guohua888.cn/15258.Doc
qtu.guohua888.cn/71430.Doc
qtu.guohua888.cn/18638.Doc
qty.guohua888.cn/37513.Doc
qty.guohua888.cn/91377.Doc
qty.guohua888.cn/60809.Doc
qty.guohua888.cn/23962.Doc
qty.guohua888.cn/50838.Doc
qty.guohua888.cn/30246.Doc
qty.guohua888.cn/48935.Doc
qty.guohua888.cn/20115.Doc
qty.guohua888.cn/76865.Doc
qty.guohua888.cn/77718.Doc
qtt.guohua888.cn/54731.Doc
qtt.guohua888.cn/32820.Doc
qtt.guohua888.cn/33368.Doc
qtt.guohua888.cn/16505.Doc
qtt.guohua888.cn/16062.Doc
qtt.guohua888.cn/03602.Doc
qtt.guohua888.cn/29816.Doc
qtt.guohua888.cn/14113.Doc
qtt.guohua888.cn/44778.Doc
qtt.guohua888.cn/85468.Doc
qtr.guohua888.cn/95850.Doc
qtr.guohua888.cn/77543.Doc
qtr.guohua888.cn/35979.Doc
qtr.guohua888.cn/73166.Doc
qtr.guohua888.cn/56564.Doc
qtr.guohua888.cn/26288.Doc
qtr.guohua888.cn/60862.Doc
qtr.guohua888.cn/82682.Doc
qtr.guohua888.cn/04684.Doc
qtr.guohua888.cn/64420.Doc
qte.guohua888.cn/60200.Doc
qte.guohua888.cn/84404.Doc
qte.guohua888.cn/42628.Doc
qte.guohua888.cn/68828.Doc
qte.guohua888.cn/86202.Doc
qte.guohua888.cn/44082.Doc
qte.guohua888.cn/80264.Doc
qte.guohua888.cn/99779.Doc
qte.guohua888.cn/84402.Doc
qte.guohua888.cn/04260.Doc
qtw.guohua888.cn/28480.Doc
qtw.guohua888.cn/88046.Doc
qtw.guohua888.cn/04444.Doc
qtw.guohua888.cn/20288.Doc
qtw.guohua888.cn/64040.Doc
qtw.guohua888.cn/04666.Doc
qtw.guohua888.cn/06842.Doc
qtw.guohua888.cn/44042.Doc
qtw.guohua888.cn/26608.Doc
qtw.guohua888.cn/06280.Doc
qtq.guohua888.cn/46208.Doc
qtq.guohua888.cn/37333.Doc
qtq.guohua888.cn/53553.Doc
qtq.guohua888.cn/62200.Doc
qtq.guohua888.cn/84606.Doc
qtq.guohua888.cn/26282.Doc
qtq.guohua888.cn/20208.Doc
qtq.guohua888.cn/64864.Doc
qtq.guohua888.cn/80862.Doc
qtq.guohua888.cn/48000.Doc
qrm.guohua888.cn/46484.Doc
qrm.guohua888.cn/99959.Doc
qrm.guohua888.cn/04222.Doc
qrm.guohua888.cn/62006.Doc
qrm.guohua888.cn/80086.Doc
qrm.guohua888.cn/13331.Doc
qrm.guohua888.cn/44460.Doc
qrm.guohua888.cn/06488.Doc
qrm.guohua888.cn/60424.Doc
qrm.guohua888.cn/44084.Doc
qrn.guohua888.cn/73757.Doc
qrn.guohua888.cn/22480.Doc
qrn.guohua888.cn/22802.Doc
qrn.guohua888.cn/40244.Doc
qrn.guohua888.cn/22600.Doc
qrn.guohua888.cn/33773.Doc
qrn.guohua888.cn/44082.Doc
qrn.guohua888.cn/22864.Doc
qrn.guohua888.cn/86424.Doc
qrn.guohua888.cn/24248.Doc
qrb.guohua888.cn/42606.Doc
qrb.guohua888.cn/20464.Doc
qrb.guohua888.cn/26022.Doc
qrb.guohua888.cn/20840.Doc
qrb.guohua888.cn/80262.Doc
qrb.guohua888.cn/42228.Doc
qrb.guohua888.cn/02088.Doc
qrb.guohua888.cn/04840.Doc
qrb.guohua888.cn/84046.Doc
qrb.guohua888.cn/46006.Doc
qrv.guohua888.cn/08620.Doc
qrv.guohua888.cn/26086.Doc
qrv.guohua888.cn/88064.Doc
qrv.guohua888.cn/24066.Doc
qrv.guohua888.cn/04660.Doc
qrv.guohua888.cn/86008.Doc
qrv.guohua888.cn/66282.Doc
qrv.guohua888.cn/04282.Doc
qrv.guohua888.cn/28062.Doc
qrv.guohua888.cn/59333.Doc
qrc.guohua888.cn/66600.Doc
qrc.guohua888.cn/13397.Doc
qrc.guohua888.cn/08800.Doc
qrc.guohua888.cn/06660.Doc
qrc.guohua888.cn/04044.Doc
qrc.guohua888.cn/88808.Doc
qrc.guohua888.cn/57939.Doc
qrc.guohua888.cn/88408.Doc
qrc.guohua888.cn/64646.Doc
qrc.guohua888.cn/44228.Doc
qrx.guohua888.cn/42206.Doc
qrx.guohua888.cn/84888.Doc
qrx.guohua888.cn/62600.Doc
qrx.guohua888.cn/84286.Doc
qrx.guohua888.cn/22202.Doc
qrx.guohua888.cn/40648.Doc
qrx.guohua888.cn/40286.Doc
qrx.guohua888.cn/42406.Doc
qrx.guohua888.cn/28620.Doc
qrx.guohua888.cn/62220.Doc
qrz.guohua888.cn/26682.Doc
qrz.guohua888.cn/88662.Doc
qrz.guohua888.cn/08222.Doc
qrz.guohua888.cn/06642.Doc
qrz.guohua888.cn/20686.Doc
qrz.guohua888.cn/66868.Doc
qrz.guohua888.cn/04466.Doc
qrz.guohua888.cn/99973.Doc
qrz.guohua888.cn/40482.Doc
qrz.guohua888.cn/26664.Doc
qrl.guohua888.cn/44844.Doc
qrl.guohua888.cn/80824.Doc
qrl.guohua888.cn/26242.Doc
qrl.guohua888.cn/22822.Doc
qrl.guohua888.cn/68026.Doc
qrl.guohua888.cn/24826.Doc
qrl.guohua888.cn/88846.Doc
qrl.guohua888.cn/84440.Doc
qrl.guohua888.cn/80068.Doc
qrl.guohua888.cn/22026.Doc
qrk.guohua888.cn/86828.Doc
qrk.guohua888.cn/82288.Doc
qrk.guohua888.cn/22460.Doc
qrk.guohua888.cn/80084.Doc
qrk.guohua888.cn/26244.Doc
qrk.guohua888.cn/28604.Doc
qrk.guohua888.cn/08802.Doc
qrk.guohua888.cn/20266.Doc
qrk.guohua888.cn/80440.Doc
qrk.guohua888.cn/80246.Doc
qrj.guohua888.cn/26864.Doc
qrj.guohua888.cn/06606.Doc
qrj.guohua888.cn/44042.Doc
qrj.guohua888.cn/60680.Doc
qrj.guohua888.cn/60442.Doc
qrj.guohua888.cn/39131.Doc
qrj.guohua888.cn/40000.Doc
qrj.guohua888.cn/08220.Doc
qrj.guohua888.cn/66844.Doc
qrj.guohua888.cn/80480.Doc
qrh.guohua888.cn/26486.Doc
qrh.guohua888.cn/42884.Doc
qrh.guohua888.cn/80824.Doc
qrh.guohua888.cn/88604.Doc
qrh.guohua888.cn/86804.Doc
qrh.guohua888.cn/20820.Doc
qrh.guohua888.cn/40824.Doc
qrh.guohua888.cn/22424.Doc
qrh.guohua888.cn/82842.Doc
qrh.guohua888.cn/06244.Doc
qrg.guohua888.cn/04002.Doc
qrg.guohua888.cn/20602.Doc
qrg.guohua888.cn/68424.Doc
qrg.guohua888.cn/24240.Doc
qrg.guohua888.cn/86402.Doc
qrg.guohua888.cn/60004.Doc
qrg.guohua888.cn/22408.Doc
qrg.guohua888.cn/20200.Doc
qrg.guohua888.cn/15755.Doc
qrg.guohua888.cn/64608.Doc
qrf.guohua888.cn/64242.Doc
qrf.guohua888.cn/84440.Doc
qrf.guohua888.cn/84222.Doc
qrf.guohua888.cn/04262.Doc
qrf.guohua888.cn/86062.Doc
qrf.guohua888.cn/88484.Doc
qrf.guohua888.cn/86486.Doc
qrf.guohua888.cn/86028.Doc
qrf.guohua888.cn/24600.Doc
qrf.guohua888.cn/75597.Doc
qrd.guohua888.cn/40604.Doc
qrd.guohua888.cn/22402.Doc
qrd.guohua888.cn/82220.Doc
qrd.guohua888.cn/44004.Doc
qrd.guohua888.cn/66248.Doc
qrd.guohua888.cn/44462.Doc
qrd.guohua888.cn/88042.Doc
qrd.guohua888.cn/60648.Doc
qrd.guohua888.cn/82604.Doc
qrd.guohua888.cn/40026.Doc
qrs.guohua888.cn/04862.Doc
qrs.guohua888.cn/40682.Doc
qrs.guohua888.cn/82488.Doc
qrs.guohua888.cn/82846.Doc
qrs.guohua888.cn/86686.Doc
qrs.guohua888.cn/84888.Doc
qrs.guohua888.cn/08024.Doc
qrs.guohua888.cn/48464.Doc
qrs.guohua888.cn/53151.Doc
qrs.guohua888.cn/86806.Doc
qra.guohua888.cn/44880.Doc
qra.guohua888.cn/48844.Doc
qra.guohua888.cn/62602.Doc
qra.guohua888.cn/60444.Doc
qra.guohua888.cn/26446.Doc
qra.guohua888.cn/62442.Doc
qra.guohua888.cn/42402.Doc
qra.guohua888.cn/84062.Doc
qra.guohua888.cn/86844.Doc
qra.guohua888.cn/24880.Doc
qrp.guohua888.cn/44424.Doc
qrp.guohua888.cn/28286.Doc
qrp.guohua888.cn/28604.Doc
qrp.guohua888.cn/62886.Doc
qrp.guohua888.cn/62628.Doc
qrp.guohua888.cn/66606.Doc
qrp.guohua888.cn/20064.Doc
qrp.guohua888.cn/28840.Doc
qrp.guohua888.cn/24440.Doc
qrp.guohua888.cn/00662.Doc
qro.guohua888.cn/86682.Doc
qro.guohua888.cn/04620.Doc
qro.guohua888.cn/44886.Doc
qro.guohua888.cn/48428.Doc
qro.guohua888.cn/44680.Doc
qro.guohua888.cn/88206.Doc
qro.guohua888.cn/28820.Doc
qro.guohua888.cn/02228.Doc
qro.guohua888.cn/02644.Doc
qro.guohua888.cn/44008.Doc
qri.guohua888.cn/82026.Doc
qri.guohua888.cn/48008.Doc
qri.guohua888.cn/24444.Doc
qri.guohua888.cn/28028.Doc
qri.guohua888.cn/95957.Doc
qri.guohua888.cn/66620.Doc
qri.guohua888.cn/42666.Doc
qri.guohua888.cn/20882.Doc
qri.guohua888.cn/35991.Doc
qri.guohua888.cn/80864.Doc
