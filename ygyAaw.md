Python数据类与对象序列化

一、dataclass基础

from dataclasses import dataclass, field, asdict

@dataclass
class Product:
    name: str
    price: float
    quantity: int = 0

p = Product("Laptop", 999.99, 5)
print(p)  # Product(name='Laptop', price=999.99, quantity=5)

自动生成 __init__、__repr__、__eq__。


二、高级特性

@dataclass
class Order:
    order_id: str
    items: list = field(default_factory=list)
    created_at: datetime = field(default_factory=datetime.now)

    def __post_init__(self):
        self.total = sum(item.price for item in self.items)

@dataclass(frozen=True)  # 不可变
class Point:
    x: float
    y: float

@dataclass(order=True)    # 支持排序
class Priority:
    level: int
    name: str = field(compare=False)

@dataclass(slots=True)    # Python 3.10+，减少内存
class Config:
    host: str
    port: int


三、JSON序列化

import json
from datetime import datetime, date

class DataClassEncoder(json.JSONEncoder):
    def default(self, obj):
        if hasattr(obj, '__dataclass_fields__'):
            return asdict(obj)
        if isinstance(obj, (datetime, date)):
            return obj.isoformat()
        return super().default(obj)

# 反序列化
def from_dict(cls, data):
    import dataclasses
    from typing import get_type_hints

    hints = get_type_hints(cls)
    kwargs = {}
    for f in dataclasses.fields(cls):
        if f.name not in data:
            continue
        value = data[f.name]
        field_type = hints[f.name]
        if dataclasses.is_dataclass(field_type) and isinstance(value, dict):
            value = from_dict(field_type, value)
        kwargs[f.name] = value
    return cls(**kwargs)


四、Pydantic

from pydantic import BaseModel, Field, validator

class User(BaseModel):
    name: str
    email: str
    age: int = Field(ge=0, le=150)

    @validator('name')
    def name_not_empty(cls, v):
        if not v.strip():
            raise ValueError('名称不能为空')
        return v

user = User(name="Alice", email="a@b.com", age=30)
print(user.model_dump_json(indent=2))


五、pickle

import pickle

state = ModelState(weights=[0.1, 0.2], bias=0.01, epoch=100)
with open('state.pkl', 'wb') as f:
    pickle.dump(state, f)
with open('state.pkl', 'rb') as f:
    loaded = pickle.load(f)

注意：pickle不安全，不要反序列化不信任的数据。


六、选择建议

场景            推荐方案
简单数据容器     dataclass
需要运行时验证   Pydantic
不可变性         frozen=True
高性能序列化     msgpack/protobuf

dat.kumchen.cn/40686.Doc
dat.kumchen.cn/46606.Doc
dat.kumchen.cn/08406.Doc
dat.kumchen.cn/44262.Doc
dat.kumchen.cn/42406.Doc
dat.kumchen.cn/82846.Doc
dat.kumchen.cn/22442.Doc
dat.kumchen.cn/46664.Doc
dat.kumchen.cn/80042.Doc
dat.kumchen.cn/08402.Doc
dar.kumchen.cn/46422.Doc
dar.kumchen.cn/64664.Doc
dar.kumchen.cn/00224.Doc
dar.kumchen.cn/64062.Doc
dar.kumchen.cn/66246.Doc
dar.kumchen.cn/82084.Doc
dar.kumchen.cn/48248.Doc
dar.kumchen.cn/04646.Doc
dar.kumchen.cn/68482.Doc
dar.kumchen.cn/64044.Doc
dae.kumchen.cn/88604.Doc
dae.kumchen.cn/02402.Doc
dae.kumchen.cn/20868.Doc
dae.kumchen.cn/80420.Doc
dae.kumchen.cn/46020.Doc
dae.kumchen.cn/68264.Doc
dae.kumchen.cn/44484.Doc
dae.kumchen.cn/62208.Doc
dae.kumchen.cn/02244.Doc
dae.kumchen.cn/42626.Doc
daw.kumchen.cn/46268.Doc
daw.kumchen.cn/64804.Doc
daw.kumchen.cn/86028.Doc
daw.kumchen.cn/28068.Doc
daw.kumchen.cn/02608.Doc
daw.kumchen.cn/22040.Doc
daw.kumchen.cn/60268.Doc
daw.kumchen.cn/02242.Doc
daw.kumchen.cn/06288.Doc
daw.kumchen.cn/28480.Doc
daq.kumchen.cn/88064.Doc
daq.kumchen.cn/02002.Doc
daq.kumchen.cn/64886.Doc
daq.kumchen.cn/60402.Doc
daq.kumchen.cn/46064.Doc
daq.kumchen.cn/40404.Doc
daq.kumchen.cn/82280.Doc
daq.kumchen.cn/06028.Doc
daq.kumchen.cn/64242.Doc
daq.kumchen.cn/84446.Doc
dpm.kumchen.cn/22264.Doc
dpm.kumchen.cn/08648.Doc
dpm.kumchen.cn/86060.Doc
dpm.kumchen.cn/60068.Doc
dpm.kumchen.cn/04682.Doc
dpm.kumchen.cn/08682.Doc
dpm.kumchen.cn/40406.Doc
dpm.kumchen.cn/26828.Doc
dpm.kumchen.cn/60868.Doc
dpm.kumchen.cn/24688.Doc
dpn.kumchen.cn/62804.Doc
dpn.kumchen.cn/46060.Doc
dpn.kumchen.cn/82226.Doc
dpn.kumchen.cn/62020.Doc
dpn.kumchen.cn/80284.Doc
dpn.kumchen.cn/46448.Doc
dpn.kumchen.cn/46080.Doc
dpn.kumchen.cn/00600.Doc
dpn.kumchen.cn/44468.Doc
dpn.kumchen.cn/68044.Doc
dpb.kumchen.cn/84406.Doc
dpb.kumchen.cn/04866.Doc
dpb.kumchen.cn/42686.Doc
dpb.kumchen.cn/06622.Doc
dpb.kumchen.cn/88824.Doc
dpb.kumchen.cn/06640.Doc
dpb.kumchen.cn/60006.Doc
dpb.kumchen.cn/60062.Doc
dpb.kumchen.cn/88682.Doc
dpb.kumchen.cn/66406.Doc
dpv.kumchen.cn/64044.Doc
dpv.kumchen.cn/84886.Doc
dpv.kumchen.cn/48646.Doc
dpv.kumchen.cn/40026.Doc
dpv.kumchen.cn/42066.Doc
dpv.kumchen.cn/42626.Doc
dpv.kumchen.cn/44064.Doc
dpv.kumchen.cn/86420.Doc
dpv.kumchen.cn/66262.Doc
dpv.kumchen.cn/82822.Doc
dpc.kumchen.cn/02420.Doc
dpc.kumchen.cn/02268.Doc
dpc.kumchen.cn/62460.Doc
dpc.kumchen.cn/68606.Doc
dpc.kumchen.cn/68666.Doc
dpc.kumchen.cn/66006.Doc
dpc.kumchen.cn/82426.Doc
dpc.kumchen.cn/06086.Doc
dpc.kumchen.cn/46484.Doc
dpc.kumchen.cn/06882.Doc
dpx.kumchen.cn/04664.Doc
dpx.kumchen.cn/20402.Doc
dpx.kumchen.cn/22602.Doc
dpx.kumchen.cn/20400.Doc
dpx.kumchen.cn/42866.Doc
dpx.kumchen.cn/04082.Doc
dpx.kumchen.cn/00028.Doc
dpx.kumchen.cn/00080.Doc
dpx.kumchen.cn/44842.Doc
dpx.kumchen.cn/08442.Doc
dpz.kumchen.cn/60244.Doc
dpz.kumchen.cn/66006.Doc
dpz.kumchen.cn/42628.Doc
dpz.kumchen.cn/00444.Doc
dpz.kumchen.cn/06488.Doc
dpz.kumchen.cn/82064.Doc
dpz.kumchen.cn/28260.Doc
dpz.kumchen.cn/60446.Doc
dpz.kumchen.cn/86466.Doc
dpz.kumchen.cn/86042.Doc
dpl.kumchen.cn/06068.Doc
dpl.kumchen.cn/80040.Doc
dpl.kumchen.cn/44660.Doc
dpl.kumchen.cn/40044.Doc
dpl.kumchen.cn/46000.Doc
dpl.kumchen.cn/26620.Doc
dpl.kumchen.cn/24486.Doc
dpl.kumchen.cn/84006.Doc
dpl.kumchen.cn/46680.Doc
dpl.kumchen.cn/66028.Doc
dpk.kumchen.cn/68606.Doc
dpk.kumchen.cn/22662.Doc
dpk.kumchen.cn/48460.Doc
dpk.kumchen.cn/02622.Doc
dpk.kumchen.cn/26642.Doc
dpk.kumchen.cn/40446.Doc
dpk.kumchen.cn/20442.Doc
dpk.kumchen.cn/24662.Doc
dpk.kumchen.cn/20028.Doc
dpk.kumchen.cn/66640.Doc
dpj.kumchen.cn/66400.Doc
dpj.kumchen.cn/60684.Doc
dpj.kumchen.cn/64082.Doc
dpj.kumchen.cn/22022.Doc
dpj.kumchen.cn/20286.Doc
dpj.kumchen.cn/28044.Doc
dpj.kumchen.cn/80028.Doc
dpj.kumchen.cn/86408.Doc
dpj.kumchen.cn/80082.Doc
dpj.kumchen.cn/02064.Doc
dph.kumchen.cn/68404.Doc
dph.kumchen.cn/00842.Doc
dph.kumchen.cn/06004.Doc
dph.kumchen.cn/22868.Doc
dph.kumchen.cn/60404.Doc
dph.kumchen.cn/66808.Doc
dph.kumchen.cn/84606.Doc
dph.kumchen.cn/24042.Doc
dph.kumchen.cn/28066.Doc
dph.kumchen.cn/64446.Doc
dpg.kumchen.cn/84804.Doc
dpg.kumchen.cn/26684.Doc
dpg.kumchen.cn/88204.Doc
dpg.kumchen.cn/84206.Doc
dpg.kumchen.cn/26268.Doc
dpg.kumchen.cn/68082.Doc
dpg.kumchen.cn/02828.Doc
dpg.kumchen.cn/62020.Doc
dpg.kumchen.cn/24680.Doc
dpg.kumchen.cn/86260.Doc
dpf.kumchen.cn/08684.Doc
dpf.kumchen.cn/44864.Doc
dpf.kumchen.cn/62682.Doc
dpf.kumchen.cn/82266.Doc
dpf.kumchen.cn/02064.Doc
dpf.kumchen.cn/88646.Doc
dpf.kumchen.cn/00488.Doc
dpf.kumchen.cn/20268.Doc
dpf.kumchen.cn/06842.Doc
dpf.kumchen.cn/00426.Doc
dpd.kumchen.cn/60268.Doc
dpd.kumchen.cn/62284.Doc
dpd.kumchen.cn/62044.Doc
dpd.kumchen.cn/44882.Doc
dpd.kumchen.cn/28882.Doc
dpd.kumchen.cn/64680.Doc
dpd.kumchen.cn/26646.Doc
dpd.kumchen.cn/08668.Doc
dpd.kumchen.cn/62886.Doc
dpd.kumchen.cn/40260.Doc
dps.kumchen.cn/80686.Doc
dps.kumchen.cn/00848.Doc
dps.kumchen.cn/20822.Doc
dps.kumchen.cn/60248.Doc
dps.kumchen.cn/64466.Doc
dps.kumchen.cn/20600.Doc
dps.kumchen.cn/04460.Doc
dps.kumchen.cn/84648.Doc
dps.kumchen.cn/42008.Doc
dps.kumchen.cn/82888.Doc
dpa.kumchen.cn/08808.Doc
dpa.kumchen.cn/48428.Doc
dpa.kumchen.cn/84222.Doc
dpa.kumchen.cn/22640.Doc
dpa.kumchen.cn/42200.Doc
dpa.kumchen.cn/28844.Doc
dpa.kumchen.cn/60040.Doc
dpa.kumchen.cn/44022.Doc
dpa.kumchen.cn/22880.Doc
dpa.kumchen.cn/22264.Doc
dpp.kumchen.cn/86002.Doc
dpp.kumchen.cn/20642.Doc
dpp.kumchen.cn/80224.Doc
dpp.kumchen.cn/26606.Doc
dpp.kumchen.cn/08442.Doc
dpp.kumchen.cn/40408.Doc
dpp.kumchen.cn/46626.Doc
dpp.kumchen.cn/64228.Doc
dpp.kumchen.cn/42886.Doc
dpp.kumchen.cn/82460.Doc
dpo.kumchen.cn/82260.Doc
dpo.kumchen.cn/24442.Doc
dpo.kumchen.cn/84464.Doc
dpo.kumchen.cn/68822.Doc
dpo.kumchen.cn/20088.Doc
dpo.kumchen.cn/28846.Doc
dpo.kumchen.cn/46268.Doc
dpo.kumchen.cn/06226.Doc
dpo.kumchen.cn/80864.Doc
dpo.kumchen.cn/42082.Doc
dpi.kumchen.cn/44226.Doc
dpi.kumchen.cn/08042.Doc
dpi.kumchen.cn/06808.Doc
dpi.kumchen.cn/40202.Doc
dpi.kumchen.cn/46424.Doc
dpi.kumchen.cn/28682.Doc
dpi.kumchen.cn/24406.Doc
dpi.kumchen.cn/64002.Doc
dpi.kumchen.cn/64680.Doc
dpi.kumchen.cn/26462.Doc
dpu.kumchen.cn/26262.Doc
dpu.kumchen.cn/08648.Doc
dpu.kumchen.cn/82804.Doc
dpu.kumchen.cn/06644.Doc
dpu.kumchen.cn/20804.Doc
dpu.kumchen.cn/02888.Doc
dpu.kumchen.cn/22024.Doc
dpu.kumchen.cn/62060.Doc
dpu.kumchen.cn/66648.Doc
dpu.kumchen.cn/26428.Doc
dpy.kumchen.cn/06080.Doc
dpy.kumchen.cn/80466.Doc
dpy.kumchen.cn/88628.Doc
dpy.kumchen.cn/28604.Doc
dpy.kumchen.cn/62040.Doc
dpy.kumchen.cn/46822.Doc
dpy.kumchen.cn/28000.Doc
dpy.kumchen.cn/06868.Doc
dpy.kumchen.cn/08246.Doc
dpy.kumchen.cn/08808.Doc
dpt.kumchen.cn/86800.Doc
dpt.kumchen.cn/88468.Doc
dpt.kumchen.cn/22260.Doc
dpt.kumchen.cn/40406.Doc
dpt.kumchen.cn/66200.Doc
dpt.kumchen.cn/44266.Doc
dpt.kumchen.cn/80242.Doc
dpt.kumchen.cn/88422.Doc
dpt.kumchen.cn/68864.Doc
dpt.kumchen.cn/46648.Doc
dpr.kumchen.cn/66284.Doc
dpr.kumchen.cn/82688.Doc
dpr.kumchen.cn/24206.Doc
dpr.kumchen.cn/86088.Doc
dpr.kumchen.cn/82228.Doc
dpr.kumchen.cn/68808.Doc
dpr.kumchen.cn/84442.Doc
dpr.kumchen.cn/68602.Doc
dpr.kumchen.cn/60066.Doc
dpr.kumchen.cn/64862.Doc
dpe.kumchen.cn/00626.Doc
dpe.kumchen.cn/80804.Doc
dpe.kumchen.cn/44440.Doc
dpe.kumchen.cn/44260.Doc
dpe.kumchen.cn/22462.Doc
dpe.kumchen.cn/44800.Doc
dpe.kumchen.cn/88086.Doc
dpe.kumchen.cn/24404.Doc
dpe.kumchen.cn/20664.Doc
dpe.kumchen.cn/06842.Doc
dpw.kumchen.cn/22880.Doc
dpw.kumchen.cn/68000.Doc
dpw.kumchen.cn/04204.Doc
dpw.kumchen.cn/06224.Doc
dpw.kumchen.cn/88002.Doc
dpw.kumchen.cn/00620.Doc
dpw.kumchen.cn/08680.Doc
dpw.kumchen.cn/06060.Doc
dpw.kumchen.cn/84608.Doc
dpw.kumchen.cn/64240.Doc
dpq.kumchen.cn/86846.Doc
dpq.kumchen.cn/20648.Doc
dpq.kumchen.cn/08486.Doc
dpq.kumchen.cn/48440.Doc
dpq.kumchen.cn/80666.Doc
dpq.kumchen.cn/88468.Doc
dpq.kumchen.cn/02000.Doc
dpq.kumchen.cn/42060.Doc
dpq.kumchen.cn/64020.Doc
dpq.kumchen.cn/08220.Doc
dom.kumchen.cn/06828.Doc
dom.kumchen.cn/86460.Doc
dom.kumchen.cn/68684.Doc
dom.kumchen.cn/84422.Doc
dom.kumchen.cn/84260.Doc
dom.kumchen.cn/84208.Doc
dom.kumchen.cn/22808.Doc
dom.kumchen.cn/48024.Doc
dom.kumchen.cn/86842.Doc
dom.kumchen.cn/46464.Doc
don.kumchen.cn/66686.Doc
don.kumchen.cn/20246.Doc
don.kumchen.cn/46866.Doc
don.kumchen.cn/60446.Doc
don.kumchen.cn/66024.Doc
don.kumchen.cn/40266.Doc
don.kumchen.cn/00064.Doc
don.kumchen.cn/22246.Doc
don.kumchen.cn/86802.Doc
don.kumchen.cn/08246.Doc
dob.kumchen.cn/06062.Doc
dob.kumchen.cn/46088.Doc
dob.kumchen.cn/02808.Doc
dob.kumchen.cn/60200.Doc
dob.kumchen.cn/46608.Doc
dob.kumchen.cn/04224.Doc
dob.kumchen.cn/62804.Doc
dob.kumchen.cn/44440.Doc
dob.kumchen.cn/60482.Doc
dob.kumchen.cn/40264.Doc
dov.kumchen.cn/44024.Doc
dov.kumchen.cn/06464.Doc
dov.kumchen.cn/88400.Doc
dov.kumchen.cn/64422.Doc
dov.kumchen.cn/82808.Doc
dov.kumchen.cn/68088.Doc
dov.kumchen.cn/22666.Doc
dov.kumchen.cn/60660.Doc
dov.kumchen.cn/24080.Doc
dov.kumchen.cn/68066.Doc
doc.kumchen.cn/04420.Doc
doc.kumchen.cn/42464.Doc
doc.kumchen.cn/64666.Doc
doc.kumchen.cn/26288.Doc
doc.kumchen.cn/46244.Doc
doc.kumchen.cn/48004.Doc
doc.kumchen.cn/48228.Doc
doc.kumchen.cn/02208.Doc
doc.kumchen.cn/06082.Doc
doc.kumchen.cn/62824.Doc
dox.kumchen.cn/42222.Doc
dox.kumchen.cn/26286.Doc
dox.kumchen.cn/64440.Doc
dox.kumchen.cn/48008.Doc
dox.kumchen.cn/68086.Doc
dox.kumchen.cn/44002.Doc
dox.kumchen.cn/08848.Doc
dox.kumchen.cn/60422.Doc
dox.kumchen.cn/26668.Doc
dox.kumchen.cn/28884.Doc
doz.kumchen.cn/40268.Doc
doz.kumchen.cn/26600.Doc
doz.kumchen.cn/00004.Doc
doz.kumchen.cn/62448.Doc
doz.kumchen.cn/40848.Doc
doz.kumchen.cn/40408.Doc
doz.kumchen.cn/08024.Doc
doz.kumchen.cn/62688.Doc
doz.kumchen.cn/46444.Doc
doz.kumchen.cn/60264.Doc
dol.kumchen.cn/46826.Doc
dol.kumchen.cn/20420.Doc
dol.kumchen.cn/68646.Doc
dol.kumchen.cn/20486.Doc
dol.kumchen.cn/60022.Doc
dol.kumchen.cn/42660.Doc
dol.kumchen.cn/06404.Doc
dol.kumchen.cn/42080.Doc
dol.kumchen.cn/68262.Doc
dol.kumchen.cn/42602.Doc
dok.kumchen.cn/04606.Doc
dok.kumchen.cn/84202.Doc
dok.kumchen.cn/60860.Doc
dok.kumchen.cn/60086.Doc
dok.kumchen.cn/62246.Doc
dok.kumchen.cn/00420.Doc
dok.kumchen.cn/24860.Doc
dok.kumchen.cn/62068.Doc
dok.kumchen.cn/00680.Doc
dok.kumchen.cn/86022.Doc
doj.kumchen.cn/28080.Doc
doj.kumchen.cn/80204.Doc
doj.kumchen.cn/64266.Doc
doj.kumchen.cn/06404.Doc
doj.kumchen.cn/66862.Doc
doj.kumchen.cn/84264.Doc
doj.kumchen.cn/64024.Doc
doj.kumchen.cn/86040.Doc
doj.kumchen.cn/60480.Doc
doj.kumchen.cn/60866.Doc
doh.kumchen.cn/82068.Doc
doh.kumchen.cn/08466.Doc
doh.kumchen.cn/60646.Doc
doh.kumchen.cn/02826.Doc
doh.kumchen.cn/88460.Doc
doh.kumchen.cn/82086.Doc
doh.kumchen.cn/88864.Doc
doh.kumchen.cn/88844.Doc
doh.kumchen.cn/42604.Doc
doh.kumchen.cn/48202.Doc
dog.kumchen.cn/08088.Doc
dog.kumchen.cn/42444.Doc
dog.kumchen.cn/40804.Doc
dog.kumchen.cn/60400.Doc
dog.kumchen.cn/88086.Doc
dog.kumchen.cn/62642.Doc
dog.kumchen.cn/88866.Doc
dog.kumchen.cn/28066.Doc
dog.kumchen.cn/26404.Doc
dog.kumchen.cn/84622.Doc
dof.kumchen.cn/22206.Doc
dof.kumchen.cn/00228.Doc
dof.kumchen.cn/08640.Doc
dof.kumchen.cn/48042.Doc
dof.kumchen.cn/64602.Doc
dof.kumchen.cn/22462.Doc
dof.kumchen.cn/00484.Doc
dof.kumchen.cn/06840.Doc
dof.kumchen.cn/24686.Doc
dof.kumchen.cn/60240.Doc
dod.kumchen.cn/26422.Doc
dod.kumchen.cn/82442.Doc
dod.kumchen.cn/46860.Doc
dod.kumchen.cn/00260.Doc
dod.kumchen.cn/20068.Doc
dod.kumchen.cn/82426.Doc
dod.kumchen.cn/44200.Doc
dod.kumchen.cn/04464.Doc
dod.kumchen.cn/42064.Doc
dod.kumchen.cn/60624.Doc
dos.kumchen.cn/06664.Doc
dos.kumchen.cn/20262.Doc
dos.kumchen.cn/86224.Doc
dos.kumchen.cn/06008.Doc
dos.kumchen.cn/42440.Doc
dos.kumchen.cn/42880.Doc
dos.kumchen.cn/00244.Doc
dos.kumchen.cn/82046.Doc
dos.kumchen.cn/44626.Doc
dos.kumchen.cn/44468.Doc
doa.kumchen.cn/00206.Doc
doa.kumchen.cn/20880.Doc
doa.kumchen.cn/68224.Doc
doa.kumchen.cn/44664.Doc
doa.kumchen.cn/26200.Doc
doa.kumchen.cn/84664.Doc
doa.kumchen.cn/06604.Doc
doa.kumchen.cn/60262.Doc
doa.kumchen.cn/46060.Doc
doa.kumchen.cn/86442.Doc
dop.kumchen.cn/82248.Doc
dop.kumchen.cn/48006.Doc
dop.kumchen.cn/26242.Doc
dop.kumchen.cn/66840.Doc
dop.kumchen.cn/80402.Doc
dop.kumchen.cn/28082.Doc
dop.kumchen.cn/68486.Doc
dop.kumchen.cn/64602.Doc
dop.kumchen.cn/66266.Doc
dop.kumchen.cn/88262.Doc
doo.kumchen.cn/44624.Doc
doo.kumchen.cn/42400.Doc
doo.kumchen.cn/42266.Doc
doo.kumchen.cn/80466.Doc
doo.kumchen.cn/88040.Doc
doo.kumchen.cn/40646.Doc
doo.kumchen.cn/22206.Doc
doo.kumchen.cn/39947.Doc
doo.kumchen.cn/24060.Doc
doo.kumchen.cn/53089.Doc
