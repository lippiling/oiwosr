Python上下文管理器的原理与应用

一、上下文管理器协议

使用 with 语句确保资源在使用完毕后被正确释放，由 __enter__ 和 __exit__ 两个方法实现：

class DatabaseConnection:
    def __init__(self, host):
        self.host = host
    def __enter__(self):
        print(f"连接 {self.host}")
        self.conn = self._create_connection()
        return self.conn
    def __exit__(self, exc_type, exc_val, exc_tb):
        if exc_type:
            self.conn.rollback()
        else:
            self.conn.commit()
        self.conn.close()
        return False  # 不抑制异常


二、@contextmanager 装饰器

from contextlib import contextmanager
import time

@contextmanager
def timer(label):
    start = time.perf_counter()
    try:
        yield
    finally:
        elapsed = time.perf_counter() - start
        print(f"{label}: {elapsed:.4f}s")

with timer("查询"):
    time.sleep(0.5)


三、contextlib 工具

from contextlib import suppress, redirect_stdout, ExitStack
import io

# suppress：忽略指定异常
with suppress(FileNotFoundError):
    os.remove('temp.txt')

# redirect_stdout：捕获输出
buffer = io.StringIO()
with redirect_stdout(buffer):
    print("这段输出被捕获")

# ExitStack：动态管理多个上下文
with ExitStack() as stack:
    files = [stack.enter_context(open(f)) for f in ['a.txt', 'b.txt']]


四、嵌套上下文管理器

# Python 3.10+ 括号语法
with (
    open('input.txt') as infile,
    open('output.txt', 'w') as outfile,
    timer("处理"),
):
    outfile.write(infile.read())


五、异步上下文管理器

import aiohttp

class AsyncSession:
    async def __aenter__(self):
        self.session = aiohttp.ClientSession()
        return self
    async def __aexit__(self, *args):
        await self.session.close()

async with AsyncSession() as session:
    async with session.get('http://example.com') as resp:
        data = await resp.text()


六、实际应用

6.1 临时修改环境变量
6.2 数据库事务管理
6.3 性能分析器
6.4 文件锁

@contextmanager
def transaction(db):
    try:
        yield db
        db.commit()
    except:
        db.rollback()
        raise

总结：上下文管理器是Python资源管理的核心机制。发现"获取-使用-释放"模式时就应使用上下文管理器封装。

qru.guohua888.cn/24862.Doc
qru.guohua888.cn/28422.Doc
qru.guohua888.cn/80820.Doc
qru.guohua888.cn/28866.Doc
qru.guohua888.cn/46068.Doc
qru.guohua888.cn/20662.Doc
qru.guohua888.cn/80826.Doc
qru.guohua888.cn/22822.Doc
qru.guohua888.cn/02662.Doc
qru.guohua888.cn/40480.Doc
qry.guohua888.cn/60604.Doc
qry.guohua888.cn/39535.Doc
qry.guohua888.cn/84044.Doc
qry.guohua888.cn/82442.Doc
qry.guohua888.cn/22480.Doc
qry.guohua888.cn/02888.Doc
qry.guohua888.cn/42882.Doc
qry.guohua888.cn/28448.Doc
qry.guohua888.cn/06646.Doc
qry.guohua888.cn/62062.Doc
qrt.guohua888.cn/24602.Doc
qrt.guohua888.cn/22202.Doc
qrt.guohua888.cn/08260.Doc
qrt.guohua888.cn/86240.Doc
qrt.guohua888.cn/68662.Doc
qrt.guohua888.cn/68402.Doc
qrt.guohua888.cn/55131.Doc
qrt.guohua888.cn/48664.Doc
qrt.guohua888.cn/26808.Doc
qrt.guohua888.cn/80840.Doc
qrr.guohua888.cn/42868.Doc
qrr.guohua888.cn/00604.Doc
qrr.guohua888.cn/80880.Doc
qrr.guohua888.cn/26484.Doc
qrr.guohua888.cn/68404.Doc
qrr.guohua888.cn/68048.Doc
qrr.guohua888.cn/48420.Doc
qrr.guohua888.cn/08864.Doc
qrr.guohua888.cn/02820.Doc
qrr.guohua888.cn/88868.Doc
qre.guohua888.cn/88064.Doc
qre.guohua888.cn/68660.Doc
qre.guohua888.cn/48482.Doc
qre.guohua888.cn/80668.Doc
qre.guohua888.cn/08662.Doc
qre.guohua888.cn/08888.Doc
qre.guohua888.cn/48448.Doc
qre.guohua888.cn/42622.Doc
qre.guohua888.cn/28488.Doc
qre.guohua888.cn/88666.Doc
qrw.guohua888.cn/44444.Doc
qrw.guohua888.cn/06264.Doc
qrw.guohua888.cn/20262.Doc
qrw.guohua888.cn/40422.Doc
qrw.guohua888.cn/44606.Doc
qrw.guohua888.cn/02484.Doc
qrw.guohua888.cn/26868.Doc
qrw.guohua888.cn/00888.Doc
qrw.guohua888.cn/22824.Doc
qrw.guohua888.cn/99995.Doc
qrq.guohua888.cn/40682.Doc
qrq.guohua888.cn/08646.Doc
qrq.guohua888.cn/26024.Doc
qrq.guohua888.cn/44886.Doc
qrq.guohua888.cn/22466.Doc
qrq.guohua888.cn/22604.Doc
qrq.guohua888.cn/20226.Doc
qrq.guohua888.cn/64668.Doc
qrq.guohua888.cn/60462.Doc
qrq.guohua888.cn/64820.Doc
qem.guohua888.cn/62026.Doc
qem.guohua888.cn/88040.Doc
qem.guohua888.cn/62884.Doc
qem.guohua888.cn/82204.Doc
qem.guohua888.cn/44062.Doc
qem.guohua888.cn/00822.Doc
qem.guohua888.cn/88860.Doc
qem.guohua888.cn/28260.Doc
qem.guohua888.cn/84488.Doc
qem.guohua888.cn/66840.Doc
qen.guohua888.cn/80484.Doc
qen.guohua888.cn/48248.Doc
qen.guohua888.cn/42424.Doc
qen.guohua888.cn/24682.Doc
qen.guohua888.cn/46002.Doc
qen.guohua888.cn/62288.Doc
qen.guohua888.cn/80848.Doc
qen.guohua888.cn/44242.Doc
qen.guohua888.cn/04206.Doc
qen.guohua888.cn/22022.Doc
qeb.guohua888.cn/82044.Doc
qeb.guohua888.cn/88840.Doc
qeb.guohua888.cn/02244.Doc
qeb.guohua888.cn/37135.Doc
qeb.guohua888.cn/40040.Doc
qeb.guohua888.cn/82004.Doc
qeb.guohua888.cn/62048.Doc
qeb.guohua888.cn/22624.Doc
qeb.guohua888.cn/00420.Doc
qeb.guohua888.cn/80408.Doc
qev.guohua888.cn/86642.Doc
qev.guohua888.cn/26642.Doc
qev.guohua888.cn/79795.Doc
qev.guohua888.cn/22204.Doc
qev.guohua888.cn/08206.Doc
qev.guohua888.cn/46846.Doc
qev.guohua888.cn/64666.Doc
qev.guohua888.cn/86880.Doc
qev.guohua888.cn/60000.Doc
qev.guohua888.cn/82642.Doc
qec.guohua888.cn/48222.Doc
qec.guohua888.cn/06064.Doc
qec.guohua888.cn/86402.Doc
qec.guohua888.cn/42684.Doc
qec.guohua888.cn/66004.Doc
qec.guohua888.cn/24460.Doc
qec.guohua888.cn/82806.Doc
qec.guohua888.cn/84642.Doc
qec.guohua888.cn/48662.Doc
qec.guohua888.cn/84426.Doc
qex.guohua888.cn/68026.Doc
qex.guohua888.cn/06606.Doc
qex.guohua888.cn/44648.Doc
qex.guohua888.cn/06686.Doc
qex.guohua888.cn/24682.Doc
qex.guohua888.cn/40246.Doc
qex.guohua888.cn/08086.Doc
qex.guohua888.cn/46868.Doc
qex.guohua888.cn/04620.Doc
qex.guohua888.cn/24806.Doc
qez.guohua888.cn/20424.Doc
qez.guohua888.cn/00082.Doc
qez.guohua888.cn/60608.Doc
qez.guohua888.cn/80808.Doc
qez.guohua888.cn/42004.Doc
qez.guohua888.cn/84448.Doc
qez.guohua888.cn/73999.Doc
qez.guohua888.cn/53511.Doc
qez.guohua888.cn/06488.Doc
qez.guohua888.cn/22406.Doc
qel.guohua888.cn/80080.Doc
qel.guohua888.cn/04868.Doc
qel.guohua888.cn/08044.Doc
qel.guohua888.cn/80066.Doc
qel.guohua888.cn/40484.Doc
qel.guohua888.cn/22464.Doc
qel.guohua888.cn/28466.Doc
qel.guohua888.cn/84024.Doc
qel.guohua888.cn/04888.Doc
qel.guohua888.cn/86844.Doc
qek.guohua888.cn/57193.Doc
qek.guohua888.cn/62420.Doc
qek.guohua888.cn/22260.Doc
qek.guohua888.cn/02246.Doc
qek.guohua888.cn/04822.Doc
qek.guohua888.cn/06640.Doc
qek.guohua888.cn/88844.Doc
qek.guohua888.cn/40628.Doc
qek.guohua888.cn/24068.Doc
qek.guohua888.cn/24286.Doc
qej.guohua888.cn/99179.Doc
qej.guohua888.cn/02688.Doc
qej.guohua888.cn/24468.Doc
qej.guohua888.cn/04840.Doc
qej.guohua888.cn/06828.Doc
qej.guohua888.cn/08024.Doc
qej.guohua888.cn/24848.Doc
qej.guohua888.cn/04440.Doc
qej.guohua888.cn/95331.Doc
qej.guohua888.cn/62628.Doc
qeh.guohua888.cn/00028.Doc
qeh.guohua888.cn/68642.Doc
qeh.guohua888.cn/40868.Doc
qeh.guohua888.cn/55111.Doc
qeh.guohua888.cn/20000.Doc
qeh.guohua888.cn/26022.Doc
qeh.guohua888.cn/06824.Doc
qeh.guohua888.cn/40822.Doc
qeh.guohua888.cn/66064.Doc
qeh.guohua888.cn/40882.Doc
qeg.guohua888.cn/26006.Doc
qeg.guohua888.cn/62244.Doc
qeg.guohua888.cn/64228.Doc
qeg.guohua888.cn/88242.Doc
qeg.guohua888.cn/64282.Doc
qeg.guohua888.cn/64044.Doc
qeg.guohua888.cn/20802.Doc
qeg.guohua888.cn/42440.Doc
qeg.guohua888.cn/64644.Doc
qeg.guohua888.cn/44604.Doc
qef.guohua888.cn/35151.Doc
qef.guohua888.cn/04668.Doc
qef.guohua888.cn/60666.Doc
qef.guohua888.cn/88222.Doc
qef.guohua888.cn/42864.Doc
qef.guohua888.cn/08224.Doc
qef.guohua888.cn/62680.Doc
qef.guohua888.cn/42406.Doc
qef.guohua888.cn/82622.Doc
qef.guohua888.cn/46620.Doc
qed.guohua888.cn/88044.Doc
qed.guohua888.cn/26466.Doc
qed.guohua888.cn/60068.Doc
qed.guohua888.cn/13935.Doc
qed.guohua888.cn/20626.Doc
qed.guohua888.cn/75775.Doc
qed.guohua888.cn/86424.Doc
qed.guohua888.cn/08646.Doc
qed.guohua888.cn/28660.Doc
qed.guohua888.cn/44448.Doc
qes.guohua888.cn/93131.Doc
qes.guohua888.cn/62200.Doc
qes.guohua888.cn/66426.Doc
qes.guohua888.cn/04446.Doc
qes.guohua888.cn/66288.Doc
qes.guohua888.cn/19591.Doc
qes.guohua888.cn/04204.Doc
qes.guohua888.cn/88680.Doc
qes.guohua888.cn/08264.Doc
qes.guohua888.cn/44220.Doc
qea.guohua888.cn/62008.Doc
qea.guohua888.cn/04040.Doc
qea.guohua888.cn/62068.Doc
qea.guohua888.cn/19935.Doc
qea.guohua888.cn/20480.Doc
qea.guohua888.cn/22484.Doc
qea.guohua888.cn/44280.Doc
qea.guohua888.cn/88840.Doc
qea.guohua888.cn/60866.Doc
qea.guohua888.cn/02242.Doc
qep.guohua888.cn/04404.Doc
qep.guohua888.cn/84282.Doc
qep.guohua888.cn/46866.Doc
qep.guohua888.cn/68042.Doc
qep.guohua888.cn/40002.Doc
qep.guohua888.cn/22248.Doc
qep.guohua888.cn/62240.Doc
qep.guohua888.cn/84848.Doc
qep.guohua888.cn/26062.Doc
qep.guohua888.cn/66042.Doc
qeo.guohua888.cn/02006.Doc
qeo.guohua888.cn/15753.Doc
qeo.guohua888.cn/62264.Doc
qeo.guohua888.cn/00246.Doc
qeo.guohua888.cn/99117.Doc
qeo.guohua888.cn/22440.Doc
qeo.guohua888.cn/86424.Doc
qeo.guohua888.cn/80206.Doc
qeo.guohua888.cn/08820.Doc
qeo.guohua888.cn/84002.Doc
qei.guohua888.cn/64668.Doc
qei.guohua888.cn/42044.Doc
qei.guohua888.cn/15593.Doc
qei.guohua888.cn/02868.Doc
qei.guohua888.cn/53371.Doc
qei.guohua888.cn/02400.Doc
qei.guohua888.cn/24202.Doc
qei.guohua888.cn/26264.Doc
qei.guohua888.cn/57935.Doc
qei.guohua888.cn/66226.Doc
qeu.guohua888.cn/44420.Doc
qeu.guohua888.cn/24868.Doc
qeu.guohua888.cn/24482.Doc
qeu.guohua888.cn/62286.Doc
qeu.guohua888.cn/40282.Doc
qeu.guohua888.cn/26822.Doc
qeu.guohua888.cn/22884.Doc
qeu.guohua888.cn/42080.Doc
qeu.guohua888.cn/86800.Doc
qeu.guohua888.cn/82220.Doc
qey.guohua888.cn/17551.Doc
qey.guohua888.cn/80620.Doc
qey.guohua888.cn/88240.Doc
qey.guohua888.cn/84400.Doc
qey.guohua888.cn/84040.Doc
qey.guohua888.cn/08840.Doc
qey.guohua888.cn/08660.Doc
qey.guohua888.cn/04480.Doc
qey.guohua888.cn/84682.Doc
qey.guohua888.cn/06424.Doc
qet.guohua888.cn/40224.Doc
qet.guohua888.cn/84666.Doc
qet.guohua888.cn/06884.Doc
qet.guohua888.cn/80666.Doc
qet.guohua888.cn/26860.Doc
qet.guohua888.cn/22082.Doc
qet.guohua888.cn/48600.Doc
qet.guohua888.cn/00640.Doc
qet.guohua888.cn/08246.Doc
qet.guohua888.cn/66226.Doc
qer.guohua888.cn/24082.Doc
qer.guohua888.cn/66200.Doc
qer.guohua888.cn/02222.Doc
qer.guohua888.cn/66846.Doc
qer.guohua888.cn/82808.Doc
qer.guohua888.cn/42046.Doc
qer.guohua888.cn/82066.Doc
qer.guohua888.cn/48842.Doc
qer.guohua888.cn/62628.Doc
qer.guohua888.cn/33573.Doc
qee.guohua888.cn/46802.Doc
qee.guohua888.cn/04606.Doc
qee.guohua888.cn/68866.Doc
qee.guohua888.cn/42260.Doc
qee.guohua888.cn/40648.Doc
qee.guohua888.cn/04808.Doc
qee.guohua888.cn/97735.Doc
qee.guohua888.cn/44842.Doc
qee.guohua888.cn/42040.Doc
qee.guohua888.cn/22600.Doc
qew.guohua888.cn/80422.Doc
qew.guohua888.cn/86066.Doc
qew.guohua888.cn/48004.Doc
qew.guohua888.cn/48068.Doc
qew.guohua888.cn/59353.Doc
qew.guohua888.cn/46642.Doc
qew.guohua888.cn/20480.Doc
qew.guohua888.cn/44802.Doc
qew.guohua888.cn/08220.Doc
qew.guohua888.cn/42248.Doc
qeq.guohua888.cn/48204.Doc
qeq.guohua888.cn/48488.Doc
qeq.guohua888.cn/62862.Doc
qeq.guohua888.cn/71795.Doc
qeq.guohua888.cn/04682.Doc
qeq.guohua888.cn/02664.Doc
qeq.guohua888.cn/84868.Doc
qeq.guohua888.cn/64406.Doc
qeq.guohua888.cn/93537.Doc
qeq.guohua888.cn/64806.Doc
qwm.guohua888.cn/68084.Doc
qwm.guohua888.cn/00262.Doc
qwm.guohua888.cn/80608.Doc
qwm.guohua888.cn/06448.Doc
qwm.guohua888.cn/20022.Doc
qwm.guohua888.cn/28886.Doc
qwm.guohua888.cn/17193.Doc
qwm.guohua888.cn/00828.Doc
qwm.guohua888.cn/28088.Doc
qwm.guohua888.cn/68846.Doc
qwn.guohua888.cn/60468.Doc
qwn.guohua888.cn/42884.Doc
qwn.guohua888.cn/26846.Doc
qwn.guohua888.cn/04428.Doc
qwn.guohua888.cn/44646.Doc
qwn.guohua888.cn/62446.Doc
qwn.guohua888.cn/55081.Doc
qwn.guohua888.cn/28664.Doc
qwn.guohua888.cn/42048.Doc
qwn.guohua888.cn/04868.Doc
qwb.guohua888.cn/62024.Doc
qwb.guohua888.cn/80800.Doc
qwb.guohua888.cn/00482.Doc
qwb.guohua888.cn/28660.Doc
qwb.guohua888.cn/02482.Doc
qwb.guohua888.cn/24028.Doc
qwb.guohua888.cn/17577.Doc
qwb.guohua888.cn/84822.Doc
qwb.guohua888.cn/66406.Doc
qwb.guohua888.cn/62268.Doc
qwv.guohua888.cn/24224.Doc
qwv.guohua888.cn/60640.Doc
qwv.guohua888.cn/48084.Doc
qwv.guohua888.cn/00240.Doc
qwv.guohua888.cn/46808.Doc
qwv.guohua888.cn/72488.Doc
qwv.guohua888.cn/64680.Doc
qwv.guohua888.cn/66284.Doc
qwv.guohua888.cn/19131.Doc
qwv.guohua888.cn/97135.Doc
qwc.guohua888.cn/80004.Doc
qwc.guohua888.cn/00805.Doc
qwc.guohua888.cn/82266.Doc
qwc.guohua888.cn/44062.Doc
qwc.guohua888.cn/06008.Doc
qwc.guohua888.cn/42806.Doc
qwc.guohua888.cn/68444.Doc
qwc.guohua888.cn/20886.Doc
qwc.guohua888.cn/80062.Doc
qwc.guohua888.cn/82686.Doc
fyo.guohua888.cn/75391.Doc
fyo.guohua888.cn/02048.Doc
fyo.guohua888.cn/60488.Doc
fyo.guohua888.cn/62020.Doc
fyo.guohua888.cn/08044.Doc
fyo.guohua888.cn/04888.Doc
fyo.guohua888.cn/42404.Doc
fyo.guohua888.cn/64008.Doc
fyo.guohua888.cn/28804.Doc
fyo.guohua888.cn/84482.Doc
fyi.guohua888.cn/00886.Doc
fyi.guohua888.cn/82008.Doc
fyi.guohua888.cn/22040.Doc
fyi.guohua888.cn/02426.Doc
fyi.guohua888.cn/00880.Doc
fyi.guohua888.cn/80826.Doc
fyi.guohua888.cn/24244.Doc
fyi.guohua888.cn/40062.Doc
fyi.guohua888.cn/86462.Doc
fyi.guohua888.cn/66220.Doc
fyu.guohua888.cn/62628.Doc
fyu.guohua888.cn/60428.Doc
fyu.guohua888.cn/88668.Doc
fyu.guohua888.cn/42602.Doc
fyu.guohua888.cn/64448.Doc
fyu.guohua888.cn/44202.Doc
fyu.guohua888.cn/62442.Doc
fyu.guohua888.cn/82244.Doc
fyu.guohua888.cn/60202.Doc
fyu.guohua888.cn/68244.Doc
fyy.guohua888.cn/64488.Doc
fyy.guohua888.cn/84604.Doc
fyy.guohua888.cn/26064.Doc
fyy.guohua888.cn/93795.Doc
fyy.guohua888.cn/26828.Doc
fyy.guohua888.cn/82800.Doc
fyy.guohua888.cn/37713.Doc
fyy.guohua888.cn/40822.Doc
fyy.guohua888.cn/42244.Doc
fyy.guohua888.cn/28640.Doc
fyt.guohua888.cn/80060.Doc
fyt.guohua888.cn/42488.Doc
fyt.guohua888.cn/82080.Doc
fyt.guohua888.cn/02260.Doc
fyt.guohua888.cn/68822.Doc
fyt.guohua888.cn/40228.Doc
fyt.guohua888.cn/02640.Doc
fyt.guohua888.cn/46262.Doc
fyt.guohua888.cn/31977.Doc
fyt.guohua888.cn/26426.Doc
fyr.guohua888.cn/42202.Doc
fyr.guohua888.cn/06202.Doc
fyr.guohua888.cn/11197.Doc
fyr.guohua888.cn/60060.Doc
fyr.guohua888.cn/08828.Doc
fyr.guohua888.cn/55751.Doc
fyr.guohua888.cn/80828.Doc
fyr.guohua888.cn/64688.Doc
fyr.guohua888.cn/44208.Doc
fyr.guohua888.cn/28333.Doc
fye.guohua888.cn/08800.Doc
fye.guohua888.cn/86848.Doc
fye.guohua888.cn/00408.Doc
fye.guohua888.cn/66868.Doc
fye.guohua888.cn/97513.Doc
fye.guohua888.cn/80286.Doc
fye.guohua888.cn/28646.Doc
fye.guohua888.cn/22448.Doc
fye.guohua888.cn/28466.Doc
fye.guohua888.cn/80004.Doc
fyw.guohua888.cn/04620.Doc
fyw.guohua888.cn/86624.Doc
fyw.guohua888.cn/62606.Doc
fyw.guohua888.cn/46400.Doc
fyw.guohua888.cn/80208.Doc
fyw.guohua888.cn/08622.Doc
fyw.guohua888.cn/46402.Doc
fyw.guohua888.cn/80028.Doc
fyw.guohua888.cn/20602.Doc
fyw.guohua888.cn/20888.Doc
fyq.guohua888.cn/84464.Doc
fyq.guohua888.cn/06008.Doc
fyq.guohua888.cn/26484.Doc
fyq.guohua888.cn/28660.Doc
fyq.guohua888.cn/51015.Doc
fyq.guohua888.cn/23414.Doc
fyq.guohua888.cn/62612.Doc
fyq.guohua888.cn/72121.Doc
fyq.guohua888.cn/90450.Doc
fyq.guohua888.cn/83876.Doc
ftm.guohua888.cn/91001.Doc
ftm.guohua888.cn/19366.Doc
ftm.guohua888.cn/28678.Doc
ftm.guohua888.cn/99138.Doc
ftm.guohua888.cn/60287.Doc
ftm.guohua888.cn/45488.Doc
ftm.guohua888.cn/30696.Doc
ftm.guohua888.cn/23600.Doc
ftm.guohua888.cn/33617.Doc
ftm.guohua888.cn/26088.Doc
ftn.guohua888.cn/28022.Doc
ftn.guohua888.cn/46482.Doc
ftn.guohua888.cn/28448.Doc
ftn.guohua888.cn/46886.Doc
ftn.guohua888.cn/64826.Doc
ftn.guohua888.cn/04860.Doc
ftn.guohua888.cn/22488.Doc
ftn.guohua888.cn/57173.Doc
ftn.guohua888.cn/60442.Doc
ftn.guohua888.cn/84480.Doc
