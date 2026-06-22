灼疵柏晾撞


Python正则表达式高级应用

一、分组与捕获

import re

# 命名分组
pattern = r'(?P<year>\d{4})-(?P<month>\d{2})-(?P<day>\d{2})'
m = re.match(pattern, '2024-03-15')
print(m.group('year'))  # 2024
print(m.groupdict())    # {'year': '2024', 'month': '03', 'day': '15'}

# 非捕获分组 (?:...)
pattern = r'(?:https?|ftp)://[\w./]+'

# 反向引用
pattern = r'\b(\w+)\s+\1\b'  # 匹配重复单词
re.findall(pattern, 'this is is a test')  # ['is']


二、零宽断言

# 正向前瞻 (?=...)：后面必须跟着
re.findall(r'\d+(?=元)', '苹果5元，香蕉3元')  # ['5', '3']

# 负向前瞻 (?!...)：后面不能跟着
re.findall(r'\b\w+\.(?!py\b)\w+\b', 'main.py config.yaml')  # ['config.yaml']

# 正向后顾 (?<=...)：前面必须是
re.findall(r'(?<=\$)\d+', '价格：$99.99')  # ['99']

# 负向后顾 (?<!...)：前面不能是
re.findall(r'(?<!-)\b\d+\b', '温度：-5 到 10')  # ['10']


三、贪婪与非贪婪

html = '<div>a</div><div>b</div>'

# 贪婪
re.findall(r'<div>.*</div>', html)
# ['<div>a</div><div>b</div>']

# 非贪婪
re.findall(r'<div>.*?</div>', html)
# ['<div>a</div>', '<div>b</div>']


四、编译与标志

pattern = re.compile(r'''
    (?P<name>[\w.+-]+)   # 用户名
    @                     # @
    (?P<domain>[\w-]+)   # 域名
    \.
    (?P<tld>[\w.]+)      # 顶级域名
''', re.VERBOSE | re.IGNORECASE)

# re.MULTILINE: ^和$匹配每行
# re.DOTALL: .匹配换行符


五、高级替换

def camel_to_snake(name):
    s1 = re.sub(r'(.)([A-Z][a-z]+)', r'\1_\2', name)
    return re.sub(r'([a-z0-9])([A-Z])', r'\1_\2', s1).lower()

print(camel_to_snake("getUserName"))  # get_user_name

# 函数替换
def c_to_f(match):
    celsius = float(match.group(1))
    return f"{celsius * 9/5 + 32:.1f}°F"

re.sub(r'(\d+\.?\d*)°C', c_to_f, '25°C')  # 77.0°F


六、实际应用

6.1 密码强度验证

def validate_password(pwd):
    checks = [
        (r'.{8,}', "至少8字符"),
        (r'[A-Z]', "需要大写字母"),
        (r'[a-z]', "需要小写字母"),
        (r'\d', "需要数字"),
        (r'[!@#$%^&*()]', "需要特殊字符"),
    ]
    for pattern, msg in checks:
        if not re.search(pattern, pwd):
            return False, msg
    return True, "合格"

6.2 日志解析

log_pattern = re.compile(
    r'(?P<ip>\d+\.\d+\.\d+\.\d+) - - '
    r'\[(?P<time>[^\]]+)\] '
    r'"(?P<method>\w+) (?P<path>[^\s]+) .*?" '
    r'(?P<status>\d+)'
)

6.3 文本清洗

cleaners = [
    (re.compile(r'<[^>]+>'), ''),           # 去HTML标签
    (re.compile(r'https?://\S+'), '[链接]'),# 替换URL
    (re.compile(r'\s+'), ' '),             # 合并空白
]


七、性能优化

1. 预编译正则表达式
2. 使用非捕获分组 (?:...)
3. 避免灾难性回溯：(a+)+b 会极慢
4. 用 [^"]* 替代 .*? 匹配引号内容

总结：零宽断言、命名分组、非贪婪匹配是高级正则的关键。正则表达式的可读性和性能需要平衡。

徊思砍炊排冉闻蕉讼颈蜒瘴频琶鹊

qpa.mmmxz.cn/208683.htm
qpa.mmmxz.cn/884863.htm
qpa.mmmxz.cn/424803.htm
qpa.mmmxz.cn/755753.htm
qpa.mmmxz.cn/751113.htm
qpp.mmmxz.cn/959573.htm
qpp.mmmxz.cn/735753.htm
qpp.mmmxz.cn/404823.htm
qpp.mmmxz.cn/113713.htm
qpp.mmmxz.cn/799713.htm
qpp.mmmxz.cn/137373.htm
qpp.mmmxz.cn/937573.htm
qpp.mmmxz.cn/913973.htm
qpp.mmmxz.cn/282003.htm
qpp.mmmxz.cn/197153.htm
qpo.mmmxz.cn/539153.htm
qpo.mmmxz.cn/175753.htm
qpo.mmmxz.cn/571593.htm
qpo.mmmxz.cn/468883.htm
qpo.mmmxz.cn/662023.htm
qpo.mmmxz.cn/971993.htm
qpo.mmmxz.cn/177113.htm
qpo.mmmxz.cn/173393.htm
qpo.mmmxz.cn/331313.htm
qpo.mmmxz.cn/804863.htm
qpi.mmmxz.cn/464803.htm
qpi.mmmxz.cn/133133.htm
qpi.mmmxz.cn/955113.htm
qpi.mmmxz.cn/139913.htm
qpi.mmmxz.cn/197993.htm
qpi.mmmxz.cn/757313.htm
qpi.mmmxz.cn/139313.htm
qpi.mmmxz.cn/571133.htm
qpi.mmmxz.cn/333193.htm
qpi.mmmxz.cn/559313.htm
qpu.mmmxz.cn/793513.htm
qpu.mmmxz.cn/331793.htm
qpu.mmmxz.cn/424483.htm
qpu.mmmxz.cn/911513.htm
qpu.mmmxz.cn/131753.htm
qpu.mmmxz.cn/713793.htm
qpu.mmmxz.cn/975533.htm
qpu.mmmxz.cn/137113.htm
qpu.mmmxz.cn/486423.htm
qpu.mmmxz.cn/131993.htm
qpy.mmmxz.cn/597193.htm
qpy.mmmxz.cn/759513.htm
qpy.mmmxz.cn/717173.htm
qpy.mmmxz.cn/535333.htm
qpy.mmmxz.cn/826463.htm
qpy.mmmxz.cn/351173.htm
qpy.mmmxz.cn/597993.htm
qpy.mmmxz.cn/177993.htm
qpy.mmmxz.cn/713773.htm
qpy.mmmxz.cn/937113.htm
qpt.mmmxz.cn/208883.htm
qpt.mmmxz.cn/420283.htm
qpt.mmmxz.cn/575393.htm
qpt.mmmxz.cn/113393.htm
qpt.mmmxz.cn/517993.htm
qpt.mmmxz.cn/997153.htm
qpt.mmmxz.cn/159353.htm
qpt.mmmxz.cn/511993.htm
qpt.mmmxz.cn/113513.htm
qpt.mmmxz.cn/793553.htm
qpr.mmmxz.cn/286623.htm
qpr.mmmxz.cn/795333.htm
qpr.mmmxz.cn/535353.htm
qpr.mmmxz.cn/313593.htm
qpr.mmmxz.cn/931913.htm
qpr.mmmxz.cn/771953.htm
qpr.mmmxz.cn/577533.htm
qpr.mmmxz.cn/282063.htm
qpr.mmmxz.cn/886003.htm
qpr.mmmxz.cn/191913.htm
qpe.mmmxz.cn/571593.htm
qpe.mmmxz.cn/535513.htm
qpe.mmmxz.cn/119533.htm
qpe.mmmxz.cn/171753.htm
qpe.mmmxz.cn/284863.htm
qpe.mmmxz.cn/151753.htm
qpe.mmmxz.cn/171173.htm
qpe.mmmxz.cn/177373.htm
qpe.mmmxz.cn/591333.htm
qpe.mmmxz.cn/622223.htm
qpw.mmmxz.cn/151933.htm
qpw.mmmxz.cn/357553.htm
qpw.mmmxz.cn/917133.htm
qpw.mmmxz.cn/971913.htm
qpw.mmmxz.cn/119793.htm
qpw.mmmxz.cn/113913.htm
qpw.mmmxz.cn/711193.htm
qpw.mmmxz.cn/428823.htm
qpw.mmmxz.cn/282683.htm
qpw.mmmxz.cn/822843.htm
qpq.mmmxz.cn/268023.htm
qpq.mmmxz.cn/628843.htm
qpq.mmmxz.cn/824263.htm
qpq.mmmxz.cn/820403.htm
qpq.mmmxz.cn/240443.htm
qpq.mmmxz.cn/842603.htm
qpq.mmmxz.cn/842423.htm
qpq.mmmxz.cn/460023.htm
qpq.mmmxz.cn/002623.htm
qpq.mmmxz.cn/428663.htm
qom.mmmxz.cn/002603.htm
qom.mmmxz.cn/624483.htm
qom.mmmxz.cn/713373.htm
qom.mmmxz.cn/791193.htm
qom.mmmxz.cn/177933.htm
qom.mmmxz.cn/737933.htm
qom.mmmxz.cn/737513.htm
qom.mmmxz.cn/359953.htm
qom.mmmxz.cn/975973.htm
qom.mmmxz.cn/777133.htm
qon.mmmxz.cn/591573.htm
qon.mmmxz.cn/848843.htm
qon.mmmxz.cn/331933.htm
qon.mmmxz.cn/731393.htm
qon.mmmxz.cn/351933.htm
qon.mmmxz.cn/868203.htm
qon.mmmxz.cn/828663.htm
qon.mmmxz.cn/379753.htm
qon.mmmxz.cn/311933.htm
qon.mmmxz.cn/086083.htm
qob.mmmxz.cn/080423.htm
qob.mmmxz.cn/735573.htm
qob.mmmxz.cn/737713.htm
qob.mmmxz.cn/757933.htm
qob.mmmxz.cn/793773.htm
qob.mmmxz.cn/153793.htm
qob.mmmxz.cn/395533.htm
qob.mmmxz.cn/335933.htm
qob.mmmxz.cn/199193.htm
qob.mmmxz.cn/515933.htm
qov.mmmxz.cn/822063.htm
qov.mmmxz.cn/442683.htm
qov.mmmxz.cn/288423.htm
qov.mmmxz.cn/624823.htm
qov.mmmxz.cn/408063.htm
qov.mmmxz.cn/480883.htm
qov.mmmxz.cn/826043.htm
qov.mmmxz.cn/866803.htm
qov.mmmxz.cn/224283.htm
qov.mmmxz.cn/284203.htm
qoc.mmmxz.cn/006483.htm
qoc.mmmxz.cn/113993.htm
qoc.mmmxz.cn/177173.htm
qoc.mmmxz.cn/395953.htm
qoc.mmmxz.cn/717993.htm
qoc.mmmxz.cn/797353.htm
qoc.mmmxz.cn/048623.htm
qoc.mmmxz.cn/262443.htm
qoc.mmmxz.cn/715973.htm
qoc.mmmxz.cn/919753.htm
qox.mmmxz.cn/955533.htm
qox.mmmxz.cn/086263.htm
qox.mmmxz.cn/959933.htm
qox.mmmxz.cn/133313.htm
qox.mmmxz.cn/753513.htm
qox.mmmxz.cn/880023.htm
qox.mmmxz.cn/531573.htm
qox.mmmxz.cn/351973.htm
qox.mmmxz.cn/375593.htm
qox.mmmxz.cn/535913.htm
qoz.mmmxz.cn/573353.htm
qoz.mmmxz.cn/199513.htm
qoz.mmmxz.cn/317773.htm
qoz.mmmxz.cn/042603.htm
qoz.mmmxz.cn/628623.htm
qoz.mmmxz.cn/739193.htm
qoz.mmmxz.cn/535793.htm
qoz.mmmxz.cn/733593.htm
qoz.mmmxz.cn/082883.htm
qoz.mmmxz.cn/797713.htm
qol.mmmxz.cn/339133.htm
qol.mmmxz.cn/175953.htm
qol.mmmxz.cn/608423.htm
qol.mmmxz.cn/402083.htm
qol.mmmxz.cn/557573.htm
qol.mmmxz.cn/133733.htm
qol.mmmxz.cn/773173.htm
qol.mmmxz.cn/824203.htm
qol.mmmxz.cn/933753.htm
qol.mmmxz.cn/979193.htm
qok.mmmxz.cn/391393.htm
qok.mmmxz.cn/600443.htm
qok.mmmxz.cn/008043.htm
qok.mmmxz.cn/135933.htm
qok.mmmxz.cn/773193.htm
qok.mmmxz.cn/355113.htm
qok.mmmxz.cn/686843.htm
qok.mmmxz.cn/719353.htm
qok.mmmxz.cn/517733.htm
qok.mmmxz.cn/115133.htm
qoj.mmmxz.cn/886003.htm
qoj.mmmxz.cn/771993.htm
qoj.mmmxz.cn/151793.htm
qoj.mmmxz.cn/537573.htm
qoj.mmmxz.cn/866803.htm
qoj.mmmxz.cn/799153.htm
qoj.mmmxz.cn/591733.htm
qoj.mmmxz.cn/713913.htm
qoj.mmmxz.cn/886483.htm
qoj.mmmxz.cn/842803.htm
qoh.mmmxz.cn/975113.htm
qoh.mmmxz.cn/773593.htm
qoh.mmmxz.cn/420443.htm
qoh.mmmxz.cn/066883.htm
qoh.mmmxz.cn/133713.htm
qoh.mmmxz.cn/599393.htm
qoh.mmmxz.cn/137953.htm
qoh.mmmxz.cn/440083.htm
qoh.mmmxz.cn/004863.htm
qoh.mmmxz.cn/573313.htm
qog.mmmxz.cn/537333.htm
qog.mmmxz.cn/979993.htm
qog.mmmxz.cn/840623.htm
qog.mmmxz.cn/913393.htm
qog.mmmxz.cn/155913.htm
qog.mmmxz.cn/515593.htm
qog.mmmxz.cn/448003.htm
qog.mmmxz.cn/886883.htm
qog.mmmxz.cn/711313.htm
qog.mmmxz.cn/755933.htm
qof.mmmxz.cn/824003.htm
qof.mmmxz.cn/933793.htm
qof.mmmxz.cn/599393.htm
qof.mmmxz.cn/953153.htm
qof.mmmxz.cn/284003.htm
qof.mmmxz.cn/406043.htm
qof.mmmxz.cn/539513.htm
qof.mmmxz.cn/795753.htm
qof.mmmxz.cn/717733.htm
qof.mmmxz.cn/226603.htm
qod.mmmxz.cn/777373.htm
qod.mmmxz.cn/133593.htm
qod.mmmxz.cn/971133.htm
qod.mmmxz.cn/822023.htm
qod.mmmxz.cn/868483.htm
qod.mmmxz.cn/315713.htm
qod.mmmxz.cn/719113.htm
qod.mmmxz.cn/573973.htm
qod.mmmxz.cn/840283.htm
qod.mmmxz.cn/975553.htm
qos.mmmxz.cn/315193.htm
qos.mmmxz.cn/575913.htm
qos.mmmxz.cn/208863.htm
qos.mmmxz.cn/931313.htm
qos.mmmxz.cn/539333.htm
qos.mmmxz.cn/199753.htm
qos.mmmxz.cn/044083.htm
qos.mmmxz.cn/517533.htm
qos.mmmxz.cn/519773.htm
qos.mmmxz.cn/193593.htm
qoa.mmmxz.cn/204643.htm
qoa.mmmxz.cn/719553.htm
qoa.mmmxz.cn/717173.htm
qoa.mmmxz.cn/795933.htm
qoa.mmmxz.cn/288243.htm
qoa.mmmxz.cn/244043.htm
qoa.mmmxz.cn/757973.htm
qoa.mmmxz.cn/719573.htm
qoa.mmmxz.cn/733993.htm
qoa.mmmxz.cn/224843.htm
qop.mmmxz.cn/397553.htm
qop.mmmxz.cn/537333.htm
qop.mmmxz.cn/111933.htm
qop.mmmxz.cn/262243.htm
qop.mmmxz.cn/042843.htm
qop.mmmxz.cn/957793.htm
qop.mmmxz.cn/339533.htm
qop.mmmxz.cn/119933.htm
qop.mmmxz.cn/248283.htm
qop.mmmxz.cn/755773.htm
qoo.mmmxz.cn/199553.htm
qoo.mmmxz.cn/991133.htm
qoo.mmmxz.cn/082403.htm
qoo.mmmxz.cn/999933.htm
qoo.mmmxz.cn/979733.htm
qoo.mmmxz.cn/137313.htm
qoo.mmmxz.cn/286683.htm
qoo.mmmxz.cn/557353.htm
qoo.mmmxz.cn/931153.htm
qoo.mmmxz.cn/117993.htm
qoi.mmmxz.cn/228283.htm
qoi.mmmxz.cn/557573.htm
qoi.mmmxz.cn/151133.htm
qoi.mmmxz.cn/373733.htm
qoi.mmmxz.cn/424623.htm
qoi.mmmxz.cn/442483.htm
qoi.mmmxz.cn/975173.htm
qoi.mmmxz.cn/959573.htm
qoi.mmmxz.cn/997333.htm
qoi.mmmxz.cn/042663.htm
qou.mmmxz.cn/337373.htm
qou.mmmxz.cn/915173.htm
qou.mmmxz.cn/551993.htm
qou.mmmxz.cn/204623.htm
qou.mmmxz.cn/480603.htm
qou.mmmxz.cn/737313.htm
qou.mmmxz.cn/773313.htm
qou.mmmxz.cn/719353.htm
qou.mmmxz.cn/400843.htm
qou.mmmxz.cn/933593.htm
qoy.mmmxz.cn/535913.htm
qoy.mmmxz.cn/135913.htm
qoy.mmmxz.cn/686263.htm
qoy.mmmxz.cn/266663.htm
qoy.mmmxz.cn/917133.htm
qoy.mmmxz.cn/537973.htm
qoy.mmmxz.cn/826063.htm
qoy.mmmxz.cn/860283.htm
qoy.mmmxz.cn/957153.htm
qoy.mmmxz.cn/353793.htm
qot.mmmxz.cn/311993.htm
qot.mmmxz.cn/402803.htm
qot.mmmxz.cn/715173.htm
qot.mmmxz.cn/335313.htm
qot.mmmxz.cn/579733.htm
qot.mmmxz.cn/682083.htm
qot.mmmxz.cn/680423.htm
qot.mmmxz.cn/331133.htm
qot.mmmxz.cn/135993.htm
qot.mmmxz.cn/537773.htm
qor.mmmxz.cn/202443.htm
qor.mmmxz.cn/771793.htm
qor.mmmxz.cn/171513.htm
qor.mmmxz.cn/355313.htm
qor.mmmxz.cn/246643.htm
qor.mmmxz.cn/480283.htm
qor.mmmxz.cn/175553.htm
qor.mmmxz.cn/133993.htm
qor.mmmxz.cn/575133.htm
qor.mmmxz.cn/020063.htm
qoe.mmmxz.cn/319773.htm
qoe.mmmxz.cn/977913.htm
qoe.mmmxz.cn/113353.htm
qoe.mmmxz.cn/262843.htm
qoe.mmmxz.cn/068203.htm
qoe.mmmxz.cn/933113.htm
qoe.mmmxz.cn/795913.htm
qoe.mmmxz.cn/519733.htm
qoe.mmmxz.cn/424663.htm
qoe.mmmxz.cn/557973.htm
qow.mmmxz.cn/375533.htm
qow.mmmxz.cn/537513.htm
qow.mmmxz.cn/684663.htm
qow.mmmxz.cn/131573.htm
qow.mmmxz.cn/715733.htm
qow.mmmxz.cn/333553.htm
qow.mmmxz.cn/664663.htm
qow.mmmxz.cn/177593.htm
qow.mmmxz.cn/535593.htm
qow.mmmxz.cn/375713.htm
qoq.mmmxz.cn/115373.htm
qoq.mmmxz.cn/204443.htm
qoq.mmmxz.cn/157173.htm
qoq.mmmxz.cn/115593.htm
qoq.mmmxz.cn/206843.htm
qoq.mmmxz.cn/042403.htm
qoq.mmmxz.cn/422643.htm
qoq.mmmxz.cn/553113.htm
qoq.mmmxz.cn/535353.htm
qoq.mmmxz.cn/575113.htm
qim.mmmxz.cn/284423.htm
qim.mmmxz.cn/155393.htm
qim.mmmxz.cn/711773.htm
qim.mmmxz.cn/775133.htm
qim.mmmxz.cn/062443.htm
qim.mmmxz.cn/886403.htm
qim.mmmxz.cn/977533.htm
qim.mmmxz.cn/555593.htm
qim.mmmxz.cn/157513.htm
qim.mmmxz.cn/640483.htm
qin.mmmxz.cn/575513.htm
qin.mmmxz.cn/517973.htm
qin.mmmxz.cn/951173.htm
qin.mmmxz.cn/244283.htm
qin.mmmxz.cn/137953.htm
qin.mmmxz.cn/999553.htm
qin.mmmxz.cn/977953.htm
qin.mmmxz.cn/842663.htm
qin.mmmxz.cn/622083.htm
qin.mmmxz.cn/197173.htm
qib.mmmxz.cn/551113.htm
qib.mmmxz.cn/333533.htm
qib.mmmxz.cn/484603.htm
qib.mmmxz.cn/399793.htm
qib.mmmxz.cn/393793.htm
qib.mmmxz.cn/517573.htm
qib.mmmxz.cn/935733.htm
qib.mmmxz.cn/755533.htm
qib.mmmxz.cn/339513.htm
qib.mmmxz.cn/139333.htm
