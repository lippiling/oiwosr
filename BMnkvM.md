焙顿障聪偻


# Python 工作单元模式 (Unit of Work)
# 跟踪变更，批量提交，事务管理，异常回滚
# 核心方法：register_new / register_dirty / register_deleted。

from abc import ABC, abstractmethod
from dataclasses import dataclass
from typing import Optional

@dataclass
class Author:
    id: str; name: str

class AuthorRepository(ABC):
    @abstractmethod
    def add(self, a: Author) -> None: ...
    @abstractmethod
    def update(self, a: Author) -> None: ...
    @abstractmethod
    def delete(self, aid: str) -> None: ...
    @abstractmethod
    def get(self, aid: str) -> Optional[Author]: ...

# 工作单元：跟踪对象变更并在 commit 时一次性写入
class UnitOfWork:
    def __init__(self):
        self._new: list[Author] = []
        self._dirty: list[Author] = []
        self._deleted: list[str] = []

    def register_new(self, a: Author) -> None:
        self._new.append(a)

    def register_dirty(self, a: Author) -> None:
        if a not in self._dirty:
            self._dirty.append(a)

    def register_deleted(self, aid: str) -> None:
        self._deleted.append(aid)

    def commit(self, repo: AuthorRepository) -> None:
        try:
            for a in self._new:
                repo.add(a)
            for a in self._dirty:
                repo.update(a)
            for aid in self._deleted:
                repo.delete(aid)
        except Exception as e:
            raise RuntimeError(f"提交失败: {e}")
        finally:
            self._new.clear(); self._dirty.clear(); self._deleted.clear()

class MemoryAuthorRepo(AuthorRepository):
    def __init__(self):
        self._store: dict[str, Author] = {}
    def add(self, a: Author) -> None:
        self._store[a.id] = a
    def update(self, a: Author) -> None:
        self._store[a.id] = a
    def delete(self, aid: str) -> None:
        self._store.pop(aid, None)
    def get(self, aid: str) -> Optional[Author]:
        return self._store.get(aid)

class AuthorService:
    def __init__(self, repo: AuthorRepository, uow: UnitOfWork):
        self._repo = repo; self._uow = uow
    def create(self, aid: str, name: str) -> None:
        self._uow.register_new(Author(aid, name))
        self._uow.commit(self._repo)
    def rename(self, aid: str, new_name: str) -> None:
        a = self._repo.get(aid)
        if a:
            a.name = new_name
            self._uow.register_dirty(a)
            self._uow.commit(self._repo)

if __name__ == "__main__":
    repo = MemoryAuthorRepo()
    uow = UnitOfWork()
    svc = AuthorService(repo, uow)
    svc.create("A001", "鲁迅")
    svc.rename("A001", "周树人")
    print(repo.get("A001"))

讲祭夜橙换我荚涝式悦屠诹偻执慌

dmu.vwbnt.cn/402284.Doc
dmu.vwbnt.cn/644882.Doc
dmu.vwbnt.cn/226440.Doc
dmu.vwbnt.cn/046808.Doc
dmu.vwbnt.cn/470455.Doc
dmu.vwbnt.cn/784515.Doc
dmy.vwbnt.cn/485700.Doc
dmy.vwbnt.cn/058445.Doc
dmy.vwbnt.cn/306556.Doc
dmy.vwbnt.cn/654802.Doc
dmy.vwbnt.cn/430724.Doc
dmy.vwbnt.cn/961247.Doc
dmy.vwbnt.cn/910221.Doc
dmy.vwbnt.cn/648931.Doc
dmy.vwbnt.cn/340389.Doc
dmy.vwbnt.cn/622313.Doc
dmt.vwbnt.cn/299512.Doc
dmt.vwbnt.cn/377255.Doc
dmt.vwbnt.cn/568280.Doc
dmt.vwbnt.cn/538437.Doc
dmt.vwbnt.cn/741131.Doc
dmt.vwbnt.cn/421876.Doc
dmt.vwbnt.cn/200267.Doc
dmt.vwbnt.cn/885090.Doc
dmt.vwbnt.cn/852146.Doc
dmt.vwbnt.cn/569569.Doc
dmr.vwbnt.cn/853246.Doc
dmr.vwbnt.cn/869480.Doc
dmr.vwbnt.cn/701144.Doc
dmr.vwbnt.cn/092471.Doc
dmr.vwbnt.cn/123960.Doc
dmr.vwbnt.cn/242401.Doc
dmr.vwbnt.cn/154926.Doc
dmr.vwbnt.cn/820167.Doc
dmr.vwbnt.cn/815233.Doc
dmr.vwbnt.cn/016880.Doc
dme.vwbnt.cn/216060.Doc
dme.vwbnt.cn/148484.Doc
dme.vwbnt.cn/588735.Doc
dme.vwbnt.cn/368187.Doc
dme.vwbnt.cn/740396.Doc
dme.vwbnt.cn/685148.Doc
dme.vwbnt.cn/780695.Doc
dme.vwbnt.cn/812539.Doc
dme.vwbnt.cn/403164.Doc
dme.vwbnt.cn/779231.Doc
dmw.vwbnt.cn/882872.Doc
dmw.vwbnt.cn/214604.Doc
dmw.vwbnt.cn/503982.Doc
dmw.vwbnt.cn/788182.Doc
dmw.vwbnt.cn/174096.Doc
dmw.vwbnt.cn/714082.Doc
dmw.vwbnt.cn/631624.Doc
dmw.vwbnt.cn/683688.Doc
dmw.vwbnt.cn/455270.Doc
dmw.vwbnt.cn/610120.Doc
dmq.vwbnt.cn/090469.Doc
dmq.vwbnt.cn/344563.Doc
dmq.vwbnt.cn/666505.Doc
dmq.vwbnt.cn/735914.Doc
dmq.vwbnt.cn/084528.Doc
dmq.vwbnt.cn/880092.Doc
dmq.vwbnt.cn/036384.Doc
dmq.vwbnt.cn/705307.Doc
dmq.vwbnt.cn/095108.Doc
dmq.vwbnt.cn/418502.Doc
dnm.vwbnt.cn/550416.Doc
dnm.vwbnt.cn/600946.Doc
dnm.vwbnt.cn/734012.Doc
dnm.vwbnt.cn/190426.Doc
dnm.vwbnt.cn/734390.Doc
dnm.vwbnt.cn/891506.Doc
dnm.vwbnt.cn/761823.Doc
dnm.vwbnt.cn/389066.Doc
dnm.vwbnt.cn/505183.Doc
dnm.vwbnt.cn/172022.Doc
dnn.vwbnt.cn/038735.Doc
dnn.vwbnt.cn/748796.Doc
dnn.vwbnt.cn/544697.Doc
dnn.vwbnt.cn/220578.Doc
dnn.vwbnt.cn/610405.Doc
dnn.vwbnt.cn/113472.Doc
dnn.vwbnt.cn/373093.Doc
dnn.vwbnt.cn/555806.Doc
dnn.vwbnt.cn/906490.Doc
dnn.vwbnt.cn/348232.Doc
dnb.vwbnt.cn/282167.Doc
dnb.vwbnt.cn/281709.Doc
dnb.vwbnt.cn/715930.Doc
dnb.vwbnt.cn/689069.Doc
dnb.vwbnt.cn/253772.Doc
dnb.vwbnt.cn/823856.Doc
dnb.vwbnt.cn/436436.Doc
dnb.vwbnt.cn/409750.Doc
dnb.vwbnt.cn/466288.Doc
dnb.vwbnt.cn/971999.Doc
dnv.nfsid.cn/224640.Doc
dnv.nfsid.cn/488284.Doc
dnv.nfsid.cn/402240.Doc
dnv.nfsid.cn/915793.Doc
dnv.nfsid.cn/088402.Doc
dnv.nfsid.cn/220864.Doc
dnv.nfsid.cn/044446.Doc
dnv.nfsid.cn/824800.Doc
dnv.nfsid.cn/684000.Doc
dnv.nfsid.cn/682462.Doc
dnc.nfsid.cn/464286.Doc
dnc.nfsid.cn/446822.Doc
dnc.nfsid.cn/406044.Doc
dnc.nfsid.cn/268402.Doc
dnc.nfsid.cn/420422.Doc
dnc.nfsid.cn/644628.Doc
dnc.nfsid.cn/846224.Doc
dnc.nfsid.cn/020666.Doc
dnc.nfsid.cn/228222.Doc
dnc.nfsid.cn/086280.Doc
dnx.nfsid.cn/402420.Doc
dnx.nfsid.cn/800044.Doc
dnx.nfsid.cn/084666.Doc
dnx.nfsid.cn/791535.Doc
dnx.nfsid.cn/064608.Doc
dnx.nfsid.cn/886220.Doc
dnx.nfsid.cn/444442.Doc
dnx.nfsid.cn/640062.Doc
dnx.nfsid.cn/604824.Doc
dnx.nfsid.cn/595955.Doc
dnz.nfsid.cn/868226.Doc
dnz.nfsid.cn/260268.Doc
dnz.nfsid.cn/264466.Doc
dnz.nfsid.cn/286040.Doc
dnz.nfsid.cn/660428.Doc
dnz.nfsid.cn/204602.Doc
dnz.nfsid.cn/064620.Doc
dnz.nfsid.cn/799511.Doc
dnz.nfsid.cn/484462.Doc
dnz.nfsid.cn/246884.Doc
dnl.nfsid.cn/082062.Doc
dnl.nfsid.cn/808800.Doc
dnl.nfsid.cn/202486.Doc
dnl.nfsid.cn/208422.Doc
dnl.nfsid.cn/191519.Doc
dnl.nfsid.cn/024446.Doc
dnl.nfsid.cn/286046.Doc
dnl.nfsid.cn/375793.Doc
dnl.nfsid.cn/791159.Doc
dnl.nfsid.cn/886024.Doc
dnk.nfsid.cn/604206.Doc
dnk.nfsid.cn/602804.Doc
dnk.nfsid.cn/995357.Doc
dnk.nfsid.cn/020606.Doc
dnk.nfsid.cn/228400.Doc
dnk.nfsid.cn/773953.Doc
dnk.nfsid.cn/595779.Doc
dnk.nfsid.cn/086048.Doc
dnk.nfsid.cn/466260.Doc
dnk.nfsid.cn/266284.Doc
dnj.nfsid.cn/888228.Doc
dnj.nfsid.cn/600428.Doc
dnj.nfsid.cn/868666.Doc
dnj.nfsid.cn/753113.Doc
dnj.nfsid.cn/973551.Doc
dnj.nfsid.cn/468244.Doc
dnj.nfsid.cn/268666.Doc
dnj.nfsid.cn/880666.Doc
dnj.nfsid.cn/608488.Doc
dnj.nfsid.cn/062084.Doc
dnh.nfsid.cn/202624.Doc
dnh.nfsid.cn/044040.Doc
dnh.nfsid.cn/666460.Doc
dnh.nfsid.cn/753917.Doc
dnh.nfsid.cn/682282.Doc
dnh.nfsid.cn/315797.Doc
dnh.nfsid.cn/480446.Doc
dnh.nfsid.cn/680862.Doc
dnh.nfsid.cn/660480.Doc
dnh.nfsid.cn/642266.Doc
dng.nfsid.cn/268640.Doc
dng.nfsid.cn/806400.Doc
dng.nfsid.cn/620682.Doc
dng.nfsid.cn/820684.Doc
dng.nfsid.cn/133744.Doc
dng.nfsid.cn/628884.Doc
dng.nfsid.cn/220240.Doc
dng.nfsid.cn/755575.Doc
dng.nfsid.cn/444422.Doc
dng.nfsid.cn/800244.Doc
dnf.nfsid.cn/446028.Doc
dnf.nfsid.cn/220844.Doc
dnf.nfsid.cn/042428.Doc
dnf.nfsid.cn/240460.Doc
dnf.nfsid.cn/608808.Doc
dnf.nfsid.cn/111919.Doc
dnf.nfsid.cn/082684.Doc
dnf.nfsid.cn/268426.Doc
dnf.nfsid.cn/244084.Doc
dnf.nfsid.cn/066624.Doc
dnd.nfsid.cn/028824.Doc
dnd.nfsid.cn/644442.Doc
dnd.nfsid.cn/624028.Doc
dnd.nfsid.cn/488464.Doc
dnd.nfsid.cn/060644.Doc
dnd.nfsid.cn/242488.Doc
dnd.nfsid.cn/171191.Doc
dnd.nfsid.cn/006660.Doc
dnd.nfsid.cn/084408.Doc
dnd.nfsid.cn/195355.Doc
dns.nfsid.cn/539399.Doc
dns.nfsid.cn/204404.Doc
dns.nfsid.cn/688664.Doc
dns.nfsid.cn/064264.Doc
dns.nfsid.cn/402662.Doc
dns.nfsid.cn/420804.Doc
dns.nfsid.cn/628840.Doc
dns.nfsid.cn/020244.Doc
dns.nfsid.cn/684044.Doc
dns.nfsid.cn/000048.Doc
dna.nfsid.cn/666404.Doc
dna.nfsid.cn/080486.Doc
dna.nfsid.cn/791753.Doc
dna.nfsid.cn/117579.Doc
dna.nfsid.cn/806228.Doc
dna.nfsid.cn/400604.Doc
dna.nfsid.cn/462868.Doc
dna.nfsid.cn/884844.Doc
dna.nfsid.cn/266082.Doc
dna.nfsid.cn/648400.Doc
dnp.nfsid.cn/008662.Doc
dnp.nfsid.cn/022242.Doc
dnp.nfsid.cn/008642.Doc
dnp.nfsid.cn/482060.Doc
dnp.nfsid.cn/482068.Doc
dnp.nfsid.cn/133171.Doc
dnp.nfsid.cn/228406.Doc
dnp.nfsid.cn/240862.Doc
dnp.nfsid.cn/335399.Doc
dnp.nfsid.cn/088600.Doc
dno.nfsid.cn/060642.Doc
dno.nfsid.cn/446828.Doc
dno.nfsid.cn/406240.Doc
dno.nfsid.cn/206820.Doc
dno.nfsid.cn/020244.Doc
dno.nfsid.cn/266248.Doc
dno.nfsid.cn/206682.Doc
dno.nfsid.cn/793537.Doc
dno.nfsid.cn/046046.Doc
dno.nfsid.cn/440008.Doc
dni.nfsid.cn/446428.Doc
dni.nfsid.cn/220264.Doc
dni.nfsid.cn/442844.Doc
dni.nfsid.cn/486664.Doc
dni.nfsid.cn/420600.Doc
dni.nfsid.cn/044002.Doc
dni.nfsid.cn/266882.Doc
dni.nfsid.cn/648086.Doc
dni.nfsid.cn/955797.Doc
dni.nfsid.cn/775917.Doc
dnu.nfsid.cn/240806.Doc
dnu.nfsid.cn/282866.Doc
dnu.nfsid.cn/860466.Doc
dnu.nfsid.cn/088066.Doc
dnu.nfsid.cn/880060.Doc
dnu.nfsid.cn/884400.Doc
dnu.nfsid.cn/482000.Doc
dnu.nfsid.cn/917719.Doc
dnu.nfsid.cn/864620.Doc
dnu.nfsid.cn/486206.Doc
dny.nfsid.cn/006640.Doc
dny.nfsid.cn/666066.Doc
dny.nfsid.cn/240006.Doc
dny.nfsid.cn/602046.Doc
dny.nfsid.cn/539395.Doc
dny.nfsid.cn/006808.Doc
dny.nfsid.cn/842282.Doc
dny.nfsid.cn/820806.Doc
dny.nfsid.cn/797151.Doc
dny.nfsid.cn/224860.Doc
dnt.nfsid.cn/802440.Doc
dnt.nfsid.cn/480264.Doc
dnt.nfsid.cn/824806.Doc
dnt.nfsid.cn/248248.Doc
dnt.nfsid.cn/420628.Doc
dnt.nfsid.cn/822042.Doc
dnt.nfsid.cn/666222.Doc
dnt.nfsid.cn/608642.Doc
dnt.nfsid.cn/359113.Doc
dnt.nfsid.cn/604860.Doc
dnr.nfsid.cn/286000.Doc
dnr.nfsid.cn/313719.Doc
dnr.nfsid.cn/428260.Doc
dnr.nfsid.cn/284880.Doc
dnr.nfsid.cn/822006.Doc
dnr.nfsid.cn/604042.Doc
dnr.nfsid.cn/840608.Doc
dnr.nfsid.cn/624640.Doc
dnr.nfsid.cn/486428.Doc
dnr.nfsid.cn/062080.Doc
dne.nfsid.cn/791157.Doc
dne.nfsid.cn/220402.Doc
dne.nfsid.cn/977111.Doc
dne.nfsid.cn/260842.Doc
dne.nfsid.cn/824646.Doc
dne.nfsid.cn/660820.Doc
dne.nfsid.cn/977117.Doc
dne.nfsid.cn/959513.Doc
dne.nfsid.cn/442284.Doc
dne.nfsid.cn/886428.Doc
dnw.nfsid.cn/448888.Doc
dnw.nfsid.cn/575317.Doc
dnw.nfsid.cn/682864.Doc
dnw.nfsid.cn/446620.Doc
dnw.nfsid.cn/822206.Doc
dnw.nfsid.cn/024660.Doc
dnw.nfsid.cn/666408.Doc
dnw.nfsid.cn/848424.Doc
dnw.nfsid.cn/866604.Doc
dnw.nfsid.cn/159131.Doc
dnq.nfsid.cn/626248.Doc
dnq.nfsid.cn/080604.Doc
dnq.nfsid.cn/022084.Doc
dnq.nfsid.cn/008688.Doc
dnq.nfsid.cn/424288.Doc
dnq.nfsid.cn/848400.Doc
dnq.nfsid.cn/864806.Doc
dnq.nfsid.cn/682648.Doc
dnq.nfsid.cn/680444.Doc
dnq.nfsid.cn/979151.Doc
dbm.nfsid.cn/537917.Doc
dbm.nfsid.cn/682626.Doc
dbm.nfsid.cn/684206.Doc
dbm.nfsid.cn/820044.Doc
dbm.nfsid.cn/848208.Doc
dbm.nfsid.cn/828868.Doc
dbm.nfsid.cn/248688.Doc
dbm.nfsid.cn/408862.Doc
dbm.nfsid.cn/519597.Doc
dbm.nfsid.cn/048044.Doc
dbn.nfsid.cn/246400.Doc
dbn.nfsid.cn/682006.Doc
dbn.nfsid.cn/068880.Doc
dbn.nfsid.cn/684086.Doc
dbn.nfsid.cn/844042.Doc
dbn.nfsid.cn/624066.Doc
dbn.nfsid.cn/248044.Doc
dbn.nfsid.cn/408646.Doc
dbn.nfsid.cn/800448.Doc
dbn.nfsid.cn/202226.Doc
dbb.nfsid.cn/379773.Doc
dbb.nfsid.cn/357979.Doc
dbb.nfsid.cn/208860.Doc
dbb.nfsid.cn/577333.Doc
dbb.nfsid.cn/068286.Doc
dbb.nfsid.cn/668282.Doc
dbb.nfsid.cn/357533.Doc
dbb.nfsid.cn/006200.Doc
dbb.nfsid.cn/080800.Doc
dbb.nfsid.cn/044042.Doc
dbv.nfsid.cn/044200.Doc
dbv.nfsid.cn/428882.Doc
dbv.nfsid.cn/044062.Doc
dbv.nfsid.cn/640806.Doc
dbv.nfsid.cn/008480.Doc
dbv.nfsid.cn/662804.Doc
dbv.nfsid.cn/804482.Doc
dbv.nfsid.cn/668688.Doc
dbv.nfsid.cn/628228.Doc
dbv.nfsid.cn/228422.Doc
dbc.nfsid.cn/088028.Doc
dbc.nfsid.cn/842646.Doc
dbc.nfsid.cn/024684.Doc
dbc.nfsid.cn/602242.Doc
dbc.nfsid.cn/080486.Doc
dbc.nfsid.cn/244280.Doc
dbc.nfsid.cn/686624.Doc
dbc.nfsid.cn/080868.Doc
dbc.nfsid.cn/606406.Doc
dbc.nfsid.cn/511713.Doc
dbx.nfsid.cn/391955.Doc
dbx.nfsid.cn/642844.Doc
dbx.nfsid.cn/264228.Doc
dbx.nfsid.cn/915377.Doc
dbx.nfsid.cn/068046.Doc
dbx.nfsid.cn/260444.Doc
dbx.nfsid.cn/000060.Doc
dbx.nfsid.cn/400660.Doc
dbx.nfsid.cn/842048.Doc
dbx.nfsid.cn/537951.Doc
dbz.nfsid.cn/844864.Doc
dbz.nfsid.cn/226046.Doc
dbz.nfsid.cn/977757.Doc
dbz.nfsid.cn/759359.Doc
dbz.nfsid.cn/442422.Doc
dbz.nfsid.cn/242246.Doc
dbz.nfsid.cn/282624.Doc
dbz.nfsid.cn/664684.Doc
dbz.nfsid.cn/268448.Doc
