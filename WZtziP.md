园壳下倮毡


"""Python时区处理最佳实践——zoneinfo 与 DST 安全算术
------------------------------------------------------------------------------
Python 3.9+ zoneinfo 基于 IANA 时区数据库。核心原则：内部用 UTC 存储，
仅在输入输出时转换。aware 与 naive datetime 不能混合比较。
"""

from zoneinfo import ZoneInfo
from datetime import datetime, timedelta, timezone


# ========== 1. zoneinfo 基础 ==========

UTC = ZoneInfo("UTC")
SHANGHAI = ZoneInfo("Asia/Shanghai")
NEW_YORK = ZoneInfo("America/New_York")

# aware datetime：构造时传入 tzinfo
dt = datetime(2024, 6, 15, 10, 0, 0, tzinfo=SHANGHAI)
print(dt)                              # 2024-06-15 10:00:00+08:00

# naive datetime 附加时区
naive = datetime(2024, 6, 15, 10, 0, 0)
aware = naive.replace(tzinfo=SHANGHAI)


# ========== 2. UTC 统一存储模式 ==========

def store_event(name: str, local: datetime, tz: ZoneInfo) -> dict:
    """本地时间转 UTC 存储，保留时区元信息"""
    if local.tzinfo is None:
        local = local.replace(tzinfo=tz)
    return {
        "event": name,
        "utc": local.astimezone(UTC),      # 存储用 UTC
        "local": local,
        "tz": str(tz),
    }


# ========== 3. 跨时区转换 ==========

def convert(dt: datetime, to_tz: ZoneInfo) -> datetime:
    """安全转换到目标时区"""
    if dt.tzinfo is None:
        raise ValueError("需要 aware datetime")
    return dt.astimezone(to_tz)

# 北京上午 10:00 对应纽约几点？
bj = datetime(2024, 12, 1, 10, 0, 0, tzinfo=SHANGHAI)
ny = convert(bj, NEW_YORK)
print(f"北京：{bj}  纽约：{ny}")       # 2024-11-30 21:00:00-05:00


# ========== 4. DST 安全算术 ==========

def dst_add(dt: datetime, days: int) -> datetime:
    """转 UTC 运算后再转回，避免 DST 跳变影响"""
    if dt.tzinfo is None:
        raise ValueError("需要 aware datetime")
    orig_tz = dt.tzinfo
    utc = dt.astimezone(UTC) + timedelta(days=days)
    return utc.astimezone(orig_tz)

def demo_dst() -> None:
    """美东 2024-03-10 切夏令时，2:00 跳到 3:00"""
    us_east = ZoneInfo("America/New_York")
    before = datetime(2024, 3, 10, 1, 30, 0, tzinfo=us_east)
    after = dst_add(before, 1)
    print(f"前：{before}  后：{after}")   # 偏移 -05:00 -> -04:00


# ========== 5. aware vs naive 比较 ==========

def compare_demo():
    """aware 和 naive 不能直接比较"""
    a = datetime(2024, 1, 1, tzinfo=UTC)
    b = datetime(2024, 1, 1)
    try:
        a > b                           # TypeError！
    except TypeError as e:
        print(f"不可比较：{e}")
    # 统一成 aware 后再比较
    print(a > b.replace(tzinfo=UTC), datetime.now(timezone.utc))

def now_utc(): return datetime.now(timezone.utc)
def fmt(dt, tz): return dt.astimezone(tz).strftime("%Y-%m-%d %H:%M:%S")
def is_dst(dt):
    if dt.tzinfo is None: raise ValueError("需要 aware datetime")
    return bool(dt.dst())

卸哨下然乖阶狼苟泼枚灯干脊甭迅

m.qms.cccnt.cn/00060.Doc
m.qms.cccnt.cn/93137.Doc
m.qms.cccnt.cn/40042.Doc
m.qms.cccnt.cn/82680.Doc
m.qms.cccnt.cn/82426.Doc
m.qma.cccnt.cn/59179.Doc
m.qma.cccnt.cn/44040.Doc
m.qma.cccnt.cn/62688.Doc
m.qma.cccnt.cn/20220.Doc
m.qma.cccnt.cn/66660.Doc
m.qma.cccnt.cn/48646.Doc
m.qma.cccnt.cn/28846.Doc
m.qma.cccnt.cn/64808.Doc
m.qma.cccnt.cn/66826.Doc
m.qma.cccnt.cn/66682.Doc
m.qma.cccnt.cn/99111.Doc
m.qma.cccnt.cn/00444.Doc
m.qma.cccnt.cn/84024.Doc
m.qma.cccnt.cn/39917.Doc
m.qma.cccnt.cn/08644.Doc
m.qma.cccnt.cn/68686.Doc
m.qma.cccnt.cn/02020.Doc
m.qma.cccnt.cn/42262.Doc
m.qma.cccnt.cn/84464.Doc
m.qma.cccnt.cn/40486.Doc
m.qmp.cccnt.cn/73915.Doc
m.qmp.cccnt.cn/02408.Doc
m.qmp.cccnt.cn/46268.Doc
m.qmp.cccnt.cn/66484.Doc
m.qmp.cccnt.cn/13115.Doc
m.qmp.cccnt.cn/46408.Doc
m.qmp.cccnt.cn/06480.Doc
m.qmp.cccnt.cn/46608.Doc
m.qmp.cccnt.cn/06644.Doc
m.qmp.cccnt.cn/66682.Doc
m.qmp.cccnt.cn/46440.Doc
m.qmp.cccnt.cn/08642.Doc
m.qmp.cccnt.cn/62022.Doc
m.qmp.cccnt.cn/24000.Doc
m.qmp.cccnt.cn/82662.Doc
m.qmp.cccnt.cn/22644.Doc
m.qmp.cccnt.cn/00666.Doc
m.qmp.cccnt.cn/77753.Doc
m.qmp.cccnt.cn/04442.Doc
m.qmp.cccnt.cn/46608.Doc
m.qmo.cccnt.cn/31797.Doc
m.qmo.cccnt.cn/91777.Doc
m.qmo.cccnt.cn/59593.Doc
m.qmo.cccnt.cn/35919.Doc
m.qmo.cccnt.cn/33151.Doc
m.qmo.cccnt.cn/62862.Doc
m.qmo.cccnt.cn/02446.Doc
m.qmo.cccnt.cn/00404.Doc
m.qmo.cccnt.cn/40444.Doc
m.qmo.cccnt.cn/24868.Doc
m.qmo.cccnt.cn/08842.Doc
m.qmo.cccnt.cn/02828.Doc
m.qmo.cccnt.cn/80282.Doc
m.qmo.cccnt.cn/02028.Doc
m.qmo.cccnt.cn/04484.Doc
m.qmo.cccnt.cn/68402.Doc
m.qmo.cccnt.cn/42842.Doc
m.qmo.cccnt.cn/28204.Doc
m.qmo.cccnt.cn/80066.Doc
m.qmo.cccnt.cn/24028.Doc
m.qmi.cccnt.cn/84460.Doc
m.qmi.cccnt.cn/04660.Doc
m.qmi.cccnt.cn/82424.Doc
m.qmi.cccnt.cn/24226.Doc
m.qmi.cccnt.cn/53911.Doc
m.qmi.cccnt.cn/40264.Doc
m.qmi.cccnt.cn/80204.Doc
m.qmi.cccnt.cn/80860.Doc
m.qmi.cccnt.cn/44228.Doc
m.qmi.cccnt.cn/22664.Doc
m.qmi.cccnt.cn/08868.Doc
m.qmi.cccnt.cn/02866.Doc
m.qmi.cccnt.cn/02668.Doc
m.qmi.cccnt.cn/80400.Doc
m.qmi.cccnt.cn/46408.Doc
m.qmi.cccnt.cn/88022.Doc
m.qmi.cccnt.cn/26240.Doc
m.qmi.cccnt.cn/42222.Doc
m.qmi.cccnt.cn/82824.Doc
m.qmi.cccnt.cn/79515.Doc
m.qmu.cccnt.cn/86488.Doc
m.qmu.cccnt.cn/28244.Doc
m.qmu.cccnt.cn/20468.Doc
m.qmu.cccnt.cn/55715.Doc
m.qmu.cccnt.cn/80006.Doc
m.qmu.cccnt.cn/44666.Doc
m.qmu.cccnt.cn/02682.Doc
m.qmu.cccnt.cn/62260.Doc
m.qmu.cccnt.cn/55339.Doc
m.qmu.cccnt.cn/06862.Doc
m.qmu.cccnt.cn/06084.Doc
m.qmu.cccnt.cn/08082.Doc
m.qmu.cccnt.cn/00806.Doc
m.qmu.cccnt.cn/37953.Doc
m.qmu.cccnt.cn/60628.Doc
m.qmu.cccnt.cn/86684.Doc
m.qmu.cccnt.cn/46448.Doc
m.qmu.cccnt.cn/88248.Doc
m.qmu.cccnt.cn/33791.Doc
m.qmu.cccnt.cn/46668.Doc
m.qmy.cccnt.cn/26804.Doc
m.qmy.cccnt.cn/20666.Doc
m.qmy.cccnt.cn/22224.Doc
m.qmy.cccnt.cn/06282.Doc
m.qmy.cccnt.cn/88046.Doc
m.qmy.cccnt.cn/84620.Doc
m.qmy.cccnt.cn/60264.Doc
m.qmy.cccnt.cn/42220.Doc
m.qmy.cccnt.cn/77519.Doc
m.qmy.cccnt.cn/60468.Doc
m.qmy.cccnt.cn/20866.Doc
m.qmy.cccnt.cn/59915.Doc
m.qmy.cccnt.cn/62240.Doc
m.qmy.cccnt.cn/77151.Doc
m.qmy.cccnt.cn/04080.Doc
m.qmy.cccnt.cn/55571.Doc
m.qmy.cccnt.cn/62888.Doc
m.qmy.cccnt.cn/17533.Doc
m.qmy.cccnt.cn/40284.Doc
m.qmy.cccnt.cn/31739.Doc
m.qmt.cccnt.cn/66460.Doc
m.qmt.cccnt.cn/33991.Doc
m.qmt.cccnt.cn/51711.Doc
m.qmt.cccnt.cn/06262.Doc
m.qmt.cccnt.cn/86402.Doc
m.qmt.cccnt.cn/39797.Doc
m.qmt.cccnt.cn/17539.Doc
m.qmt.cccnt.cn/35137.Doc
m.qmt.cccnt.cn/91779.Doc
m.qmt.cccnt.cn/77531.Doc
m.qmt.cccnt.cn/84264.Doc
m.qmt.cccnt.cn/42884.Doc
m.qmt.cccnt.cn/62040.Doc
m.qmt.cccnt.cn/86848.Doc
m.qmt.cccnt.cn/20684.Doc
m.qmt.cccnt.cn/02066.Doc
m.qmt.cccnt.cn/04006.Doc
m.qmt.cccnt.cn/82448.Doc
m.qmt.cccnt.cn/24040.Doc
m.qmt.cccnt.cn/60482.Doc
m.qmr.cccnt.cn/08022.Doc
m.qmr.cccnt.cn/51775.Doc
m.qmr.cccnt.cn/02248.Doc
m.qmr.cccnt.cn/84446.Doc
m.qmr.cccnt.cn/46800.Doc
m.qmr.cccnt.cn/91351.Doc
m.qmr.cccnt.cn/00262.Doc
m.qmr.cccnt.cn/28262.Doc
m.qmr.cccnt.cn/40426.Doc
m.qmr.cccnt.cn/46006.Doc
m.qmr.cccnt.cn/80622.Doc
m.qmr.cccnt.cn/00688.Doc
m.qmr.cccnt.cn/24604.Doc
m.qmr.cccnt.cn/04600.Doc
m.qmr.cccnt.cn/84008.Doc
m.qmr.cccnt.cn/24604.Doc
m.qmr.cccnt.cn/33913.Doc
m.qmr.cccnt.cn/40020.Doc
m.qmr.cccnt.cn/28622.Doc
m.qmr.cccnt.cn/75719.Doc
m.qme.cccnt.cn/80802.Doc
m.qme.cccnt.cn/75571.Doc
m.qme.cccnt.cn/00622.Doc
m.qme.cccnt.cn/06264.Doc
m.qme.cccnt.cn/46264.Doc
m.qme.cccnt.cn/51777.Doc
m.qme.cccnt.cn/66446.Doc
m.qme.cccnt.cn/26462.Doc
m.qme.cccnt.cn/84844.Doc
m.qme.cccnt.cn/44644.Doc
m.qme.cccnt.cn/68044.Doc
m.qme.cccnt.cn/53711.Doc
m.qme.cccnt.cn/44488.Doc
m.qme.cccnt.cn/88402.Doc
m.qme.cccnt.cn/48002.Doc
m.qme.cccnt.cn/82000.Doc
m.qme.cccnt.cn/48488.Doc
m.qme.cccnt.cn/66804.Doc
m.qme.cccnt.cn/79519.Doc
m.qme.cccnt.cn/71571.Doc
m.qmw.cccnt.cn/75137.Doc
m.qmw.cccnt.cn/26660.Doc
m.qmw.cccnt.cn/40628.Doc
m.qmw.cccnt.cn/31577.Doc
m.qmw.cccnt.cn/64468.Doc
m.qmw.cccnt.cn/82804.Doc
m.qmw.cccnt.cn/55971.Doc
m.qmw.cccnt.cn/00082.Doc
m.qmw.cccnt.cn/04826.Doc
m.qmw.cccnt.cn/42862.Doc
m.qmw.cccnt.cn/62600.Doc
m.qmw.cccnt.cn/84480.Doc
m.qmw.cccnt.cn/28060.Doc
m.qmw.cccnt.cn/22024.Doc
m.qmw.cccnt.cn/17357.Doc
m.qmw.cccnt.cn/60402.Doc
m.qmw.cccnt.cn/91711.Doc
m.qmw.cccnt.cn/95353.Doc
m.qmw.cccnt.cn/02088.Doc
m.qmw.cccnt.cn/40848.Doc
m.qmq.cccnt.cn/44880.Doc
m.qmq.cccnt.cn/84420.Doc
m.qmq.cccnt.cn/02846.Doc
m.qmq.cccnt.cn/80088.Doc
m.qmq.cccnt.cn/73319.Doc
m.qmq.cccnt.cn/00860.Doc
m.qmq.cccnt.cn/33917.Doc
m.qmq.cccnt.cn/22468.Doc
m.qmq.cccnt.cn/73119.Doc
m.qmq.cccnt.cn/22062.Doc
m.qmq.cccnt.cn/48662.Doc
m.qmq.cccnt.cn/08420.Doc
m.qmq.cccnt.cn/28688.Doc
m.qmq.cccnt.cn/19137.Doc
m.qmq.cccnt.cn/44006.Doc
m.qmq.cccnt.cn/48880.Doc
m.qmq.cccnt.cn/59373.Doc
m.qmq.cccnt.cn/86264.Doc
m.qmq.cccnt.cn/44042.Doc
m.qmq.cccnt.cn/42682.Doc
m.qnm.cccnt.cn/48606.Doc
m.qnm.cccnt.cn/68264.Doc
m.qnm.cccnt.cn/40842.Doc
m.qnm.cccnt.cn/22460.Doc
m.qnm.cccnt.cn/46882.Doc
m.qnm.cccnt.cn/22684.Doc
m.qnm.cccnt.cn/64284.Doc
m.qnm.cccnt.cn/95715.Doc
m.qnm.cccnt.cn/64220.Doc
m.qnm.cccnt.cn/40828.Doc
m.qnm.cccnt.cn/80400.Doc
m.qnm.cccnt.cn/20068.Doc
m.qnm.cccnt.cn/60448.Doc
m.qnm.cccnt.cn/39135.Doc
m.qnm.cccnt.cn/00848.Doc
m.qnm.cccnt.cn/31931.Doc
m.qnm.cccnt.cn/46088.Doc
m.qnm.cccnt.cn/00882.Doc
m.qnm.cccnt.cn/99393.Doc
m.qnm.cccnt.cn/62080.Doc
m.qnn.cccnt.cn/46024.Doc
m.qnn.cccnt.cn/46644.Doc
m.qnn.cccnt.cn/19355.Doc
m.qnn.cccnt.cn/02664.Doc
m.qnn.cccnt.cn/02080.Doc
m.qnn.cccnt.cn/28802.Doc
m.qnn.cccnt.cn/26644.Doc
m.qnn.cccnt.cn/24482.Doc
m.qnn.cccnt.cn/64620.Doc
m.qnn.cccnt.cn/46868.Doc
m.qnn.cccnt.cn/53799.Doc
m.qnn.cccnt.cn/08846.Doc
m.qnn.cccnt.cn/84460.Doc
m.qnn.cccnt.cn/40266.Doc
m.qnn.cccnt.cn/62424.Doc
m.qnn.cccnt.cn/40604.Doc
m.qnn.cccnt.cn/15137.Doc
m.qnn.cccnt.cn/24846.Doc
m.qnn.cccnt.cn/46824.Doc
m.qnn.cccnt.cn/44024.Doc
m.qnb.cccnt.cn/88642.Doc
m.qnb.cccnt.cn/28806.Doc
m.qnb.cccnt.cn/22824.Doc
m.qnb.cccnt.cn/13937.Doc
m.qnb.cccnt.cn/15371.Doc
m.qnb.cccnt.cn/88000.Doc
m.qnb.cccnt.cn/88668.Doc
m.qnb.cccnt.cn/13973.Doc
m.qnb.cccnt.cn/26206.Doc
m.qnb.cccnt.cn/64242.Doc
m.qnb.cccnt.cn/00824.Doc
m.qnb.cccnt.cn/84048.Doc
m.qnb.cccnt.cn/64864.Doc
m.qnb.cccnt.cn/66646.Doc
m.qnb.cccnt.cn/42822.Doc
m.qnb.cccnt.cn/82064.Doc
m.qnb.cccnt.cn/60440.Doc
m.qnb.cccnt.cn/95591.Doc
m.qnb.cccnt.cn/57139.Doc
m.qnb.cccnt.cn/22620.Doc
m.qnv.cccnt.cn/39515.Doc
m.qnv.cccnt.cn/80802.Doc
m.qnv.cccnt.cn/84680.Doc
m.qnv.cccnt.cn/22866.Doc
m.qnv.cccnt.cn/62664.Doc
m.qnv.cccnt.cn/46006.Doc
m.qnv.cccnt.cn/13337.Doc
m.qnv.cccnt.cn/66282.Doc
m.qnv.cccnt.cn/53775.Doc
m.qnv.cccnt.cn/60402.Doc
m.qnv.cccnt.cn/82000.Doc
m.qnv.cccnt.cn/04484.Doc
m.qnv.cccnt.cn/48068.Doc
m.qnv.cccnt.cn/00620.Doc
m.qnv.cccnt.cn/24468.Doc
m.qnv.cccnt.cn/44600.Doc
m.qnv.cccnt.cn/48420.Doc
m.qnv.cccnt.cn/00626.Doc
m.qnv.cccnt.cn/46462.Doc
m.qnv.cccnt.cn/48228.Doc
m.qnc.cccnt.cn/00048.Doc
m.qnc.cccnt.cn/99737.Doc
m.qnc.cccnt.cn/57937.Doc
m.qnc.cccnt.cn/35797.Doc
m.qnc.cccnt.cn/88040.Doc
m.qnc.cccnt.cn/39513.Doc
m.qnc.cccnt.cn/66426.Doc
m.qnc.cccnt.cn/22040.Doc
m.qnc.cccnt.cn/37975.Doc
m.qnc.cccnt.cn/37573.Doc
m.qnc.cccnt.cn/22462.Doc
m.qnc.cccnt.cn/91135.Doc
m.qnc.cccnt.cn/06408.Doc
m.qnc.cccnt.cn/08204.Doc
m.qnc.cccnt.cn/22684.Doc
m.qnc.cccnt.cn/88040.Doc
m.qnc.cccnt.cn/44846.Doc
m.qnc.cccnt.cn/68864.Doc
m.qnc.cccnt.cn/48646.Doc
m.qnc.cccnt.cn/22488.Doc
m.qnx.cccnt.cn/06620.Doc
m.qnx.cccnt.cn/64266.Doc
m.qnx.cccnt.cn/22844.Doc
m.qnx.cccnt.cn/00228.Doc
m.qnx.cccnt.cn/82484.Doc
m.qnx.cccnt.cn/00648.Doc
m.qnx.cccnt.cn/02808.Doc
m.qnx.cccnt.cn/20248.Doc
m.qnx.cccnt.cn/77519.Doc
m.qnx.cccnt.cn/31951.Doc
m.qnx.cccnt.cn/08028.Doc
m.qnx.cccnt.cn/44024.Doc
m.qnx.cccnt.cn/46462.Doc
m.qnx.cccnt.cn/62282.Doc
m.qnx.cccnt.cn/19597.Doc
m.qnx.cccnt.cn/22406.Doc
m.qnx.cccnt.cn/73755.Doc
m.qnx.cccnt.cn/39735.Doc
m.qnx.cccnt.cn/02026.Doc
m.qnx.cccnt.cn/68004.Doc
m.qnz.cccnt.cn/84644.Doc
m.qnz.cccnt.cn/60628.Doc
m.qnz.cccnt.cn/48848.Doc
m.qnz.cccnt.cn/24204.Doc
m.qnz.cccnt.cn/84240.Doc
m.qnz.cccnt.cn/26008.Doc
m.qnz.cccnt.cn/20084.Doc
m.qnz.cccnt.cn/93559.Doc
m.qnz.cccnt.cn/80804.Doc
m.qnz.cccnt.cn/82402.Doc
m.qnz.cccnt.cn/95197.Doc
m.qnz.cccnt.cn/51131.Doc
m.qnz.cccnt.cn/71993.Doc
m.qnz.cccnt.cn/02888.Doc
m.qnz.cccnt.cn/04422.Doc
m.qnz.cccnt.cn/40660.Doc
m.qnz.cccnt.cn/64668.Doc
m.qnz.cccnt.cn/15559.Doc
m.qnz.cccnt.cn/84648.Doc
m.qnz.cccnt.cn/46088.Doc
m.qnl.cccnt.cn/86004.Doc
m.qnl.cccnt.cn/44820.Doc
m.qnl.cccnt.cn/60480.Doc
m.qnl.cccnt.cn/13115.Doc
m.qnl.cccnt.cn/04046.Doc
m.qnl.cccnt.cn/46448.Doc
m.qnl.cccnt.cn/15399.Doc
m.qnl.cccnt.cn/37933.Doc
m.qnl.cccnt.cn/26224.Doc
m.qnl.cccnt.cn/35351.Doc
m.qnl.cccnt.cn/60428.Doc
m.qnl.cccnt.cn/55755.Doc
m.qnl.cccnt.cn/22466.Doc
m.qnl.cccnt.cn/08022.Doc
m.qnl.cccnt.cn/40042.Doc
m.qnl.cccnt.cn/28802.Doc
m.qnl.cccnt.cn/51971.Doc
m.qnl.cccnt.cn/80604.Doc
m.qnl.cccnt.cn/04282.Doc
m.qnl.cccnt.cn/84202.Doc
m.qnk.cccnt.cn/88660.Doc
m.qnk.cccnt.cn/75535.Doc
m.qnk.cccnt.cn/66864.Doc
m.qnk.cccnt.cn/60448.Doc
m.qnk.cccnt.cn/59375.Doc
m.qnk.cccnt.cn/26862.Doc
m.qnk.cccnt.cn/79717.Doc
m.qnk.cccnt.cn/08824.Doc
m.qnk.cccnt.cn/68028.Doc
m.qnk.cccnt.cn/64604.Doc
