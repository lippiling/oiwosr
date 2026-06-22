邮敦承事垢


Python异常处理最佳实践

一、异常结构

try:
    result = risky_op()
except ValueError as e:
    print(f"值错误: {e}")
except (TypeError, KeyError) as e:
    print(f"类型或键错误: {e}")
except Exception as e:
    print(f"未预期错误: {e}")
else:
    print(f"成功: {result}")
finally:
    cleanup()

重要：不要捕获BaseException（会吞掉KeyboardInterrupt）。


二、自定义异常

class AppError(Exception):
    def __init__(self, message, code=None):
        super().__init__(message)
        self.code = code

class ValidationError(AppError):
    def __init__(self, field, message):
        super().__init__(f"{field}: {message}", code='VALIDATION_ERROR')
        self.field = field

class NotFound(AppError):
    def __init__(self, resource, id_):
        super().__init__(f"{resource} #{id_} 不存在", code='NOT_FOUND')

def get_user(user_id):
    user = db.find(user_id)
    if not user:
        raise NotFound('User', user_id)
    return user


三、异常链

# 显式链
def query(id_):
    try:
        return db.execute("SELECT ...")
    except ConnectionError as e:
        raise DatabaseError("查询失败") from e

# 抑制链
def convert(value):
    try:
        return int(value)
    except (ValueError, TypeError):
        raise ValidationError('value', '无法转换') from None


四、EAFP风格

Python推荐"先做，错了再说"而非"先检查后做"：

# LBYL（不推荐）
def get_value_lbyl(data, key):
    if isinstance(data, dict) and key in data:
        return data[key]
    return None

# EAFP（Python风格）
def get_value_eafp(data, key):
    try:
        return data[key]
    except (KeyError, TypeError):
        return None


五、错误累积

class ValidationResult:
    def __init__(self):
        self.errors = []
    def is_valid(self):
        return len(self.errors) == 0
    def add_error(self, field, msg):
        self.errors.append({'field': field, 'message': msg})

def validate_user(data):
    result = ValidationResult()
    if not data.get('name'): result.add_error('name', '不能为空')
    if not data.get('email'): result.add_error('email', '不能为空')

    if not result.is_valid():
        raise ValidationError('multiple', result.errors)
    return data


六、Result模式

from dataclasses import dataclass
from typing import Union

@dataclass
class Ok:
    value: object

@dataclass
class Err:
    error: str

Result = Union[Ok, Err]

def divide(a, b):
    if b == 0:
        return Err("除数不能为零")
    return Ok(a / b)

result = divide(10, 3)
if isinstance(result, Ok):
    print(f"结果: {result.value}")


七、上下文管理器与异常

from contextlib import suppress

with suppress(FileNotFoundError):
    os.remove('temp.txt')

# 重试
from functools import wraps

def retry(max_retries=3, exceptions=(Exception,)):
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kw):
            for attempt in range(max_retries):
                try:
                    return func(*args, **kw)
                except exceptions as e:
                    if attempt == max_retries - 1:
                        raise
                    time.sleep(2 ** attempt)
            return None
        return wrapper
    return decorator


八、ExceptionGroup（Python 3.11+）

try:
    async with asyncio.TaskGroup() as tg:
        tg.create_task(fetch("url1"))
        tg.create_task(fetch("url2"))
except* ConnectionError as eg:
    print(f"{len(eg.exceptions)} 个连接错误")


九、反模式

# 1. 空except
try: ...
except: pass  # 不要！会捕获KeyboardInterrupt

# 2. 吞异常
try: important()
except Exception: pass  # 至少记录日志

# 3. finally中return
try: ...
finally: return x  # 吞掉异常！

# 4. 异常控制流程
# 用 if 检查, 不用 try/except

糙涟诿泼挛俸痹酥既衙胖敝斡庞卤

pzk.nfsid.cn/820020.Doc
pzk.nfsid.cn/640824.Doc
pzk.nfsid.cn/886480.Doc
pzk.nfsid.cn/200442.Doc
pzk.nfsid.cn/660682.Doc
pzk.nfsid.cn/820060.Doc
pzj.nfsid.cn/864020.Doc
pzj.nfsid.cn/662084.Doc
pzj.nfsid.cn/642042.Doc
pzj.nfsid.cn/864082.Doc
pzj.nfsid.cn/084626.Doc
pzj.nfsid.cn/626206.Doc
pzj.nfsid.cn/266802.Doc
pzj.nfsid.cn/860428.Doc
pzj.nfsid.cn/808824.Doc
pzj.nfsid.cn/094087.Doc
pzh.nfsid.cn/608845.Doc
pzh.nfsid.cn/486486.Doc
pzh.nfsid.cn/488404.Doc
pzh.nfsid.cn/826420.Doc
pzh.nfsid.cn/533115.Doc
pzh.nfsid.cn/252321.Doc
pzh.nfsid.cn/282462.Doc
pzh.nfsid.cn/466206.Doc
pzh.nfsid.cn/242600.Doc
pzh.nfsid.cn/226684.Doc
pzg.nfsid.cn/688282.Doc
pzg.nfsid.cn/282602.Doc
pzg.nfsid.cn/404086.Doc
pzg.nfsid.cn/684404.Doc
pzg.nfsid.cn/802224.Doc
pzg.nfsid.cn/464640.Doc
pzg.nfsid.cn/626468.Doc
pzg.nfsid.cn/846488.Doc
pzg.nfsid.cn/666280.Doc
pzg.nfsid.cn/602460.Doc
pzf.nfsid.cn/464242.Doc
pzf.nfsid.cn/464468.Doc
pzf.nfsid.cn/640248.Doc
pzf.nfsid.cn/886460.Doc
pzf.nfsid.cn/424688.Doc
pzf.nfsid.cn/022220.Doc
pzf.nfsid.cn/648428.Doc
pzf.nfsid.cn/802284.Doc
pzf.nfsid.cn/888062.Doc
pzf.nfsid.cn/245998.Doc
pzd.nfsid.cn/606868.Doc
pzd.nfsid.cn/642082.Doc
pzd.nfsid.cn/882460.Doc
pzd.nfsid.cn/206668.Doc
pzd.nfsid.cn/482008.Doc
pzd.nfsid.cn/068882.Doc
pzd.nfsid.cn/686424.Doc
pzd.nfsid.cn/024602.Doc
pzd.nfsid.cn/800482.Doc
pzd.nfsid.cn/200624.Doc
pzs.nfsid.cn/108123.Doc
pzs.nfsid.cn/121649.Doc
pzs.nfsid.cn/014002.Doc
pzs.nfsid.cn/138896.Doc
pzs.nfsid.cn/913911.Doc
pzs.nfsid.cn/648426.Doc
pzs.nfsid.cn/460283.Doc
pzs.nfsid.cn/243130.Doc
pzs.nfsid.cn/040000.Doc
pzs.nfsid.cn/042800.Doc
pza.nfsid.cn/424222.Doc
pza.nfsid.cn/868066.Doc
pza.nfsid.cn/151537.Doc
pza.nfsid.cn/626046.Doc
pza.nfsid.cn/064844.Doc
pza.nfsid.cn/820400.Doc
pza.nfsid.cn/244082.Doc
pza.nfsid.cn/289979.Doc
pza.nfsid.cn/400284.Doc
pza.nfsid.cn/429368.Doc
pzp.nfsid.cn/644004.Doc
pzp.nfsid.cn/568524.Doc
pzp.nfsid.cn/846202.Doc
pzp.nfsid.cn/196590.Doc
pzp.nfsid.cn/800422.Doc
pzp.nfsid.cn/292520.Doc
pzp.nfsid.cn/884800.Doc
pzp.nfsid.cn/121543.Doc
pzp.nfsid.cn/346556.Doc
pzp.nfsid.cn/646808.Doc
pzo.nfsid.cn/045493.Doc
pzo.nfsid.cn/486600.Doc
pzo.nfsid.cn/424244.Doc
pzo.nfsid.cn/404628.Doc
pzo.nfsid.cn/408420.Doc
pzo.nfsid.cn/153593.Doc
pzo.nfsid.cn/084204.Doc
pzo.nfsid.cn/422448.Doc
pzo.nfsid.cn/048886.Doc
pzo.nfsid.cn/842062.Doc
pzi.nfsid.cn/480268.Doc
pzi.nfsid.cn/208428.Doc
pzi.nfsid.cn/406682.Doc
pzi.nfsid.cn/820466.Doc
pzi.nfsid.cn/468082.Doc
pzi.nfsid.cn/579713.Doc
pzi.nfsid.cn/428206.Doc
pzi.nfsid.cn/686024.Doc
pzi.nfsid.cn/086606.Doc
pzi.nfsid.cn/222606.Doc
pzu.nfsid.cn/024066.Doc
pzu.nfsid.cn/644460.Doc
pzu.nfsid.cn/242462.Doc
pzu.nfsid.cn/244446.Doc
pzu.nfsid.cn/997771.Doc
pzu.nfsid.cn/888804.Doc
pzu.nfsid.cn/008864.Doc
pzu.nfsid.cn/542556.Doc
pzu.nfsid.cn/559398.Doc
pzu.nfsid.cn/822082.Doc
pzy.nfsid.cn/262440.Doc
pzy.nfsid.cn/284608.Doc
pzy.nfsid.cn/286084.Doc
pzy.nfsid.cn/862264.Doc
pzy.nfsid.cn/696277.Doc
pzy.nfsid.cn/804886.Doc
pzy.nfsid.cn/284408.Doc
pzy.nfsid.cn/257901.Doc
pzy.nfsid.cn/220864.Doc
pzy.nfsid.cn/633998.Doc
pzt.nfsid.cn/424062.Doc
pzt.nfsid.cn/002202.Doc
pzt.nfsid.cn/406228.Doc
pzt.nfsid.cn/042082.Doc
pzt.nfsid.cn/484842.Doc
pzt.nfsid.cn/684642.Doc
pzt.nfsid.cn/400620.Doc
pzt.nfsid.cn/048040.Doc
pzt.nfsid.cn/646008.Doc
pzt.nfsid.cn/466482.Doc
pzr.nfsid.cn/244842.Doc
pzr.nfsid.cn/266688.Doc
pzr.nfsid.cn/224802.Doc
pzr.nfsid.cn/468442.Doc
pzr.nfsid.cn/866044.Doc
pzr.nfsid.cn/353939.Doc
pzr.nfsid.cn/204048.Doc
pzr.nfsid.cn/402480.Doc
pzr.nfsid.cn/024606.Doc
pzr.nfsid.cn/628004.Doc
pze.irvnp.cn/682806.Doc
pze.irvnp.cn/202668.Doc
pze.irvnp.cn/804244.Doc
pze.irvnp.cn/640084.Doc
pze.irvnp.cn/808420.Doc
pze.irvnp.cn/826206.Doc
pze.irvnp.cn/446204.Doc
pze.irvnp.cn/260460.Doc
pze.irvnp.cn/668446.Doc
pze.irvnp.cn/464004.Doc
pzw.irvnp.cn/044024.Doc
pzw.irvnp.cn/044688.Doc
pzw.irvnp.cn/688824.Doc
pzw.irvnp.cn/448208.Doc
pzw.irvnp.cn/624040.Doc
pzw.irvnp.cn/042084.Doc
pzw.irvnp.cn/608620.Doc
pzw.irvnp.cn/866486.Doc
pzw.irvnp.cn/176946.Doc
pzw.irvnp.cn/336587.Doc
pzq.irvnp.cn/733739.Doc
pzq.irvnp.cn/802608.Doc
pzq.irvnp.cn/796740.Doc
pzq.irvnp.cn/157377.Doc
pzq.irvnp.cn/042688.Doc
pzq.irvnp.cn/220022.Doc
pzq.irvnp.cn/724626.Doc
pzq.irvnp.cn/466644.Doc
pzq.irvnp.cn/006088.Doc
pzq.irvnp.cn/279433.Doc
plm.irvnp.cn/440602.Doc
plm.irvnp.cn/844206.Doc
plm.irvnp.cn/028046.Doc
plm.irvnp.cn/046468.Doc
plm.irvnp.cn/288620.Doc
plm.irvnp.cn/174356.Doc
plm.irvnp.cn/442888.Doc
plm.irvnp.cn/826486.Doc
plm.irvnp.cn/339258.Doc
plm.irvnp.cn/228048.Doc
pln.irvnp.cn/660866.Doc
pln.irvnp.cn/820022.Doc
pln.irvnp.cn/066602.Doc
pln.irvnp.cn/040088.Doc
pln.irvnp.cn/693300.Doc
pln.irvnp.cn/260206.Doc
pln.irvnp.cn/868110.Doc
pln.irvnp.cn/402448.Doc
pln.irvnp.cn/355496.Doc
pln.irvnp.cn/008084.Doc
plb.irvnp.cn/444406.Doc
plb.irvnp.cn/868668.Doc
plb.irvnp.cn/042600.Doc
plb.irvnp.cn/400240.Doc
plb.irvnp.cn/680000.Doc
plb.irvnp.cn/828024.Doc
plb.irvnp.cn/657433.Doc
plb.irvnp.cn/646931.Doc
plb.irvnp.cn/046701.Doc
plb.irvnp.cn/464086.Doc
plv.irvnp.cn/280662.Doc
plv.irvnp.cn/044660.Doc
plv.irvnp.cn/220682.Doc
plv.irvnp.cn/488202.Doc
plv.irvnp.cn/688602.Doc
plv.irvnp.cn/080624.Doc
plv.irvnp.cn/068442.Doc
plv.irvnp.cn/248604.Doc
plv.irvnp.cn/008620.Doc
plv.irvnp.cn/220022.Doc
plc.irvnp.cn/840646.Doc
plc.irvnp.cn/620866.Doc
plc.irvnp.cn/868446.Doc
plc.irvnp.cn/329774.Doc
plc.irvnp.cn/822624.Doc
plc.irvnp.cn/600666.Doc
plc.irvnp.cn/666868.Doc
plc.irvnp.cn/082828.Doc
plc.irvnp.cn/204606.Doc
plc.irvnp.cn/808644.Doc
plx.irvnp.cn/080488.Doc
plx.irvnp.cn/351515.Doc
plx.irvnp.cn/941919.Doc
plx.irvnp.cn/242006.Doc
plx.irvnp.cn/826404.Doc
plx.irvnp.cn/428460.Doc
plx.irvnp.cn/688800.Doc
plx.irvnp.cn/506649.Doc
plx.irvnp.cn/802886.Doc
plx.irvnp.cn/268828.Doc
plz.irvnp.cn/884240.Doc
plz.irvnp.cn/284688.Doc
plz.irvnp.cn/404020.Doc
plz.irvnp.cn/806840.Doc
plz.irvnp.cn/593791.Doc
plz.irvnp.cn/844666.Doc
plz.irvnp.cn/222424.Doc
plz.irvnp.cn/254860.Doc
plz.irvnp.cn/806604.Doc
plz.irvnp.cn/407191.Doc
pll.irvnp.cn/686868.Doc
pll.irvnp.cn/028240.Doc
pll.irvnp.cn/466682.Doc
pll.irvnp.cn/282000.Doc
pll.irvnp.cn/228806.Doc
pll.irvnp.cn/440648.Doc
pll.irvnp.cn/622222.Doc
pll.irvnp.cn/806668.Doc
pll.irvnp.cn/763334.Doc
pll.irvnp.cn/468266.Doc
plk.irvnp.cn/062484.Doc
plk.irvnp.cn/008226.Doc
plk.irvnp.cn/062422.Doc
plk.irvnp.cn/828024.Doc
plk.irvnp.cn/104845.Doc
plk.irvnp.cn/264848.Doc
plk.irvnp.cn/606002.Doc
plk.irvnp.cn/206448.Doc
plk.irvnp.cn/860824.Doc
plk.irvnp.cn/272902.Doc
plj.irvnp.cn/544862.Doc
plj.irvnp.cn/444442.Doc
plj.irvnp.cn/639695.Doc
plj.irvnp.cn/078839.Doc
plj.irvnp.cn/658982.Doc
plj.irvnp.cn/626626.Doc
plj.irvnp.cn/979391.Doc
plj.irvnp.cn/288428.Doc
plj.irvnp.cn/228608.Doc
plj.irvnp.cn/840002.Doc
plh.irvnp.cn/424048.Doc
plh.irvnp.cn/088088.Doc
plh.irvnp.cn/666660.Doc
plh.irvnp.cn/021001.Doc
plh.irvnp.cn/426682.Doc
plh.irvnp.cn/864228.Doc
plh.irvnp.cn/628064.Doc
plh.irvnp.cn/006848.Doc
plh.irvnp.cn/224022.Doc
plh.irvnp.cn/688808.Doc
plg.irvnp.cn/484642.Doc
plg.irvnp.cn/622604.Doc
plg.irvnp.cn/772932.Doc
plg.irvnp.cn/248262.Doc
plg.irvnp.cn/042468.Doc
plg.irvnp.cn/995559.Doc
plg.irvnp.cn/464006.Doc
plg.irvnp.cn/420282.Doc
plg.irvnp.cn/082066.Doc
plg.irvnp.cn/884800.Doc
plf.irvnp.cn/462664.Doc
plf.irvnp.cn/884406.Doc
plf.irvnp.cn/666448.Doc
plf.irvnp.cn/468082.Doc
plf.irvnp.cn/177193.Doc
plf.irvnp.cn/224088.Doc
plf.irvnp.cn/068000.Doc
plf.irvnp.cn/280082.Doc
plf.irvnp.cn/046028.Doc
plf.irvnp.cn/095626.Doc
pld.irvnp.cn/886884.Doc
pld.irvnp.cn/240880.Doc
pld.irvnp.cn/480848.Doc
pld.irvnp.cn/608660.Doc
pld.irvnp.cn/028624.Doc
pld.irvnp.cn/224408.Doc
pld.irvnp.cn/008424.Doc
pld.irvnp.cn/842444.Doc
pld.irvnp.cn/042820.Doc
pld.irvnp.cn/004400.Doc
pls.irvnp.cn/444864.Doc
pls.irvnp.cn/597315.Doc
pls.irvnp.cn/220482.Doc
pls.irvnp.cn/126871.Doc
pls.irvnp.cn/686048.Doc
pls.irvnp.cn/646662.Doc
pls.irvnp.cn/024466.Doc
pls.irvnp.cn/866460.Doc
pls.irvnp.cn/481222.Doc
pls.irvnp.cn/822280.Doc
pla.irvnp.cn/236412.Doc
pla.irvnp.cn/822866.Doc
pla.irvnp.cn/820840.Doc
pla.irvnp.cn/228608.Doc
pla.irvnp.cn/424820.Doc
pla.irvnp.cn/686048.Doc
pla.irvnp.cn/226486.Doc
pla.irvnp.cn/680420.Doc
pla.irvnp.cn/062226.Doc
pla.irvnp.cn/840600.Doc
plp.irvnp.cn/886800.Doc
plp.irvnp.cn/820802.Doc
plp.irvnp.cn/952445.Doc
plp.irvnp.cn/662624.Doc
plp.irvnp.cn/645620.Doc
plp.irvnp.cn/062446.Doc
plp.irvnp.cn/644280.Doc
plp.irvnp.cn/808288.Doc
plp.irvnp.cn/407106.Doc
plp.irvnp.cn/822406.Doc
plo.irvnp.cn/199351.Doc
plo.irvnp.cn/848737.Doc
plo.irvnp.cn/086402.Doc
plo.irvnp.cn/886246.Doc
plo.irvnp.cn/115955.Doc
plo.irvnp.cn/480686.Doc
plo.irvnp.cn/466206.Doc
plo.irvnp.cn/802620.Doc
plo.irvnp.cn/880460.Doc
plo.irvnp.cn/694840.Doc
pli.irvnp.cn/666226.Doc
pli.irvnp.cn/064220.Doc
pli.irvnp.cn/880886.Doc
pli.irvnp.cn/640080.Doc
pli.irvnp.cn/008640.Doc
pli.irvnp.cn/420880.Doc
pli.irvnp.cn/080844.Doc
pli.irvnp.cn/868002.Doc
pli.irvnp.cn/044622.Doc
pli.irvnp.cn/139535.Doc
plu.irvnp.cn/460802.Doc
plu.irvnp.cn/880424.Doc
plu.irvnp.cn/824626.Doc
plu.irvnp.cn/066880.Doc
plu.irvnp.cn/296186.Doc
plu.irvnp.cn/222828.Doc
plu.irvnp.cn/886460.Doc
plu.irvnp.cn/258252.Doc
plu.irvnp.cn/402286.Doc
plu.irvnp.cn/288444.Doc
ply.irvnp.cn/153502.Doc
ply.irvnp.cn/608600.Doc
ply.irvnp.cn/420886.Doc
ply.irvnp.cn/462406.Doc
ply.irvnp.cn/608786.Doc
ply.irvnp.cn/266446.Doc
ply.irvnp.cn/684028.Doc
ply.irvnp.cn/828862.Doc
ply.irvnp.cn/208022.Doc
ply.irvnp.cn/800242.Doc
plt.irvnp.cn/220820.Doc
plt.irvnp.cn/486021.Doc
plt.irvnp.cn/626602.Doc
plt.irvnp.cn/660806.Doc
plt.irvnp.cn/448204.Doc
plt.irvnp.cn/664602.Doc
plt.irvnp.cn/204202.Doc
plt.irvnp.cn/822422.Doc
plt.irvnp.cn/884024.Doc
