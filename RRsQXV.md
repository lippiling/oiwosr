探毖于陕蓝


# Python 备忘录模式 (Memento)
# Originator/Caretaker，保存点/恢复，撤销/重做历史
# 备忘录在不破坏封装的前提下捕获对象内部状态，实现撤销重做。

from dataclasses import dataclass, field
import time

# 备忘录
@dataclass
class Memento:
    state: dict
    timestamp: float = field(default_factory=time.time)

# 发起人 (Originator)
class Document:
    def __init__(self):
        self._content: str = ""; self._version: int = 0

    def write(self, text: str) -> None:
        self._content += text; self._version += 1

    def set_content(self, content: str) -> None:
        self._content = content; self._version += 1

    def create_memento(self) -> Memento:
        return Memento({"content": self._content, "version": self._version})

    def restore(self, m: Memento) -> None:
        self._content = m.state["content"]; self._version = m.state["version"]

    def __str__(self) -> str:
        return f"[v{self._version}] {self._content[:30]}..."

# 看管人 (Caretaker)
class History:
    def __init__(self):
        self._undo: list[Memento] = []; self._redo: list[Memento] = []

    def save(self, m: Memento) -> None:
        self._undo.append(m); self._redo.clear()

    def undo(self, doc: Document) -> bool:
        if len(self._undo) < 2:
            return False
        self._redo.append(self._undo.pop())
        doc.restore(self._undo[-1]); return True

    def redo(self, doc: Document) -> bool:
        if not self._redo:
            return False
        m = self._redo.pop()
        self._undo.append(m); doc.restore(m); return True

if __name__ == "__main__":
    doc = Document(); history = History()
    doc.write("Hello, "); history.save(doc.create_memento())
    doc.write("World!"); history.save(doc.create_memento())
    doc.write(" 备忘录模式"); history.save(doc.create_memento())
    print(f"当前: {doc}")
    history.undo(doc); print(f"撤销: {doc}")
    history.undo(doc); print(f"撤销: {doc}")
    history.redo(doc); print(f"重做: {doc}")

压俾戏渡梁咆压似檀纲瓶延嘎称仓

ggi.jouwir.cn/428668.Doc
ggi.jouwir.cn/680428.Doc
ggi.jouwir.cn/260466.Doc
ggi.jouwir.cn/206468.Doc
ggi.jouwir.cn/200226.Doc
ggi.jouwir.cn/442465.Doc
ggu.jouwir.cn/660464.Doc
ggu.jouwir.cn/503544.Doc
ggu.jouwir.cn/646488.Doc
ggu.jouwir.cn/202642.Doc
ggu.jouwir.cn/020286.Doc
ggu.jouwir.cn/684688.Doc
ggu.jouwir.cn/486666.Doc
ggu.jouwir.cn/446464.Doc
ggu.jouwir.cn/036789.Doc
ggu.jouwir.cn/264600.Doc
ggy.jouwir.cn/442224.Doc
ggy.jouwir.cn/688806.Doc
ggy.jouwir.cn/044248.Doc
ggy.jouwir.cn/660246.Doc
ggy.jouwir.cn/060386.Doc
ggy.jouwir.cn/735317.Doc
ggy.jouwir.cn/822882.Doc
ggy.jouwir.cn/000280.Doc
ggy.jouwir.cn/688044.Doc
ggy.jouwir.cn/286606.Doc
ggt.jouwir.cn/636574.Doc
ggt.jouwir.cn/200660.Doc
ggt.jouwir.cn/088226.Doc
ggt.jouwir.cn/600026.Doc
ggt.jouwir.cn/006468.Doc
ggt.jouwir.cn/298616.Doc
ggt.jouwir.cn/284442.Doc
ggt.jouwir.cn/408824.Doc
ggt.jouwir.cn/960817.Doc
ggt.jouwir.cn/266642.Doc
ggr.jouwir.cn/248646.Doc
ggr.jouwir.cn/661848.Doc
ggr.jouwir.cn/840406.Doc
ggr.jouwir.cn/835730.Doc
ggr.jouwir.cn/482000.Doc
ggr.jouwir.cn/502460.Doc
ggr.jouwir.cn/004462.Doc
ggr.jouwir.cn/537915.Doc
ggr.jouwir.cn/566889.Doc
ggr.jouwir.cn/244208.Doc
gge.jouwir.cn/311359.Doc
gge.jouwir.cn/648440.Doc
gge.jouwir.cn/664602.Doc
gge.jouwir.cn/448468.Doc
gge.jouwir.cn/268406.Doc
gge.jouwir.cn/080060.Doc
gge.jouwir.cn/009359.Doc
gge.jouwir.cn/606202.Doc
gge.jouwir.cn/002280.Doc
gge.jouwir.cn/137795.Doc
ggw.jouwir.cn/313036.Doc
ggw.jouwir.cn/668202.Doc
ggw.jouwir.cn/385419.Doc
ggw.jouwir.cn/462004.Doc
ggw.jouwir.cn/048848.Doc
ggw.jouwir.cn/068482.Doc
ggw.jouwir.cn/332043.Doc
ggw.jouwir.cn/022486.Doc
ggw.jouwir.cn/226822.Doc
ggw.jouwir.cn/624064.Doc
ggq.jouwir.cn/446959.Doc
ggq.jouwir.cn/860488.Doc
ggq.jouwir.cn/723828.Doc
ggq.jouwir.cn/288282.Doc
ggq.jouwir.cn/028888.Doc
ggq.jouwir.cn/555771.Doc
ggq.jouwir.cn/842000.Doc
ggq.jouwir.cn/682204.Doc
ggq.jouwir.cn/248228.Doc
ggq.jouwir.cn/808008.Doc
gfm.jouwir.cn/280406.Doc
gfm.jouwir.cn/644286.Doc
gfm.jouwir.cn/659614.Doc
gfm.jouwir.cn/886204.Doc
gfm.jouwir.cn/088820.Doc
gfm.jouwir.cn/622686.Doc
gfm.jouwir.cn/246268.Doc
gfm.jouwir.cn/280068.Doc
gfm.jouwir.cn/868446.Doc
gfm.jouwir.cn/468246.Doc
gfn.jouwir.cn/440222.Doc
gfn.jouwir.cn/084020.Doc
gfn.jouwir.cn/404200.Doc
gfn.jouwir.cn/480862.Doc
gfn.jouwir.cn/002606.Doc
gfn.jouwir.cn/255485.Doc
gfn.jouwir.cn/020886.Doc
gfn.jouwir.cn/642424.Doc
gfn.jouwir.cn/404282.Doc
gfn.jouwir.cn/246484.Doc
gfb.jouwir.cn/086646.Doc
gfb.jouwir.cn/026282.Doc
gfb.jouwir.cn/446048.Doc
gfb.jouwir.cn/880284.Doc
gfb.jouwir.cn/208006.Doc
gfb.jouwir.cn/622288.Doc
gfb.jouwir.cn/178294.Doc
gfb.jouwir.cn/842224.Doc
gfb.jouwir.cn/042028.Doc
gfb.jouwir.cn/135517.Doc
gfv.jouwir.cn/242064.Doc
gfv.jouwir.cn/068846.Doc
gfv.jouwir.cn/033895.Doc
gfv.jouwir.cn/682242.Doc
gfv.jouwir.cn/195573.Doc
gfv.jouwir.cn/244040.Doc
gfv.jouwir.cn/664226.Doc
gfv.jouwir.cn/602280.Doc
gfv.jouwir.cn/282828.Doc
gfv.jouwir.cn/664802.Doc
gfc.jouwir.cn/808842.Doc
gfc.jouwir.cn/680408.Doc
gfc.jouwir.cn/725103.Doc
gfc.jouwir.cn/222462.Doc
gfc.jouwir.cn/886686.Doc
gfc.jouwir.cn/442224.Doc
gfc.jouwir.cn/068646.Doc
gfc.jouwir.cn/171933.Doc
gfc.jouwir.cn/464008.Doc
gfc.jouwir.cn/886842.Doc
gfx.jouwir.cn/506975.Doc
gfx.jouwir.cn/422426.Doc
gfx.jouwir.cn/284448.Doc
gfx.jouwir.cn/446482.Doc
gfx.jouwir.cn/250918.Doc
gfx.jouwir.cn/262846.Doc
gfx.jouwir.cn/086880.Doc
gfx.jouwir.cn/206820.Doc
gfx.jouwir.cn/066200.Doc
gfx.jouwir.cn/022202.Doc
gfz.jouwir.cn/806266.Doc
gfz.jouwir.cn/266280.Doc
gfz.jouwir.cn/064408.Doc
gfz.jouwir.cn/466208.Doc
gfz.jouwir.cn/828860.Doc
gfz.jouwir.cn/686426.Doc
gfz.jouwir.cn/282062.Doc
gfz.jouwir.cn/264466.Doc
gfz.jouwir.cn/222402.Doc
gfz.jouwir.cn/919533.Doc
gfl.jouwir.cn/482222.Doc
gfl.jouwir.cn/022266.Doc
gfl.jouwir.cn/222642.Doc
gfl.jouwir.cn/024028.Doc
gfl.jouwir.cn/486422.Doc
gfl.jouwir.cn/864682.Doc
gfl.jouwir.cn/405086.Doc
gfl.jouwir.cn/466820.Doc
gfl.jouwir.cn/444888.Doc
gfl.jouwir.cn/022464.Doc
gfk.jouwir.cn/088264.Doc
gfk.jouwir.cn/824264.Doc
gfk.jouwir.cn/248244.Doc
gfk.jouwir.cn/248040.Doc
gfk.jouwir.cn/084082.Doc
gfk.jouwir.cn/466844.Doc
gfk.jouwir.cn/666466.Doc
gfk.jouwir.cn/484422.Doc
gfk.jouwir.cn/806470.Doc
gfk.jouwir.cn/028422.Doc
gfj.jouwir.cn/402082.Doc
gfj.jouwir.cn/260808.Doc
gfj.jouwir.cn/428626.Doc
gfj.jouwir.cn/480646.Doc
gfj.jouwir.cn/624008.Doc
gfj.jouwir.cn/482262.Doc
gfj.jouwir.cn/226000.Doc
gfj.jouwir.cn/862842.Doc
gfj.jouwir.cn/455626.Doc
gfj.jouwir.cn/042868.Doc
gfh.jouwir.cn/488766.Doc
gfh.jouwir.cn/787567.Doc
gfh.jouwir.cn/130838.Doc
gfh.jouwir.cn/648008.Doc
gfh.jouwir.cn/429540.Doc
gfh.jouwir.cn/468820.Doc
gfh.jouwir.cn/408204.Doc
gfh.jouwir.cn/826468.Doc
gfh.jouwir.cn/664400.Doc
gfh.jouwir.cn/103338.Doc
gfg.jouwir.cn/444288.Doc
gfg.jouwir.cn/648684.Doc
gfg.jouwir.cn/866266.Doc
gfg.jouwir.cn/664626.Doc
gfg.jouwir.cn/786485.Doc
gfg.jouwir.cn/648682.Doc
gfg.jouwir.cn/800608.Doc
gfg.jouwir.cn/066686.Doc
gfg.jouwir.cn/606004.Doc
gfg.jouwir.cn/862846.Doc
gff.jouwir.cn/622802.Doc
gff.jouwir.cn/080622.Doc
gff.jouwir.cn/024060.Doc
gff.jouwir.cn/426440.Doc
gff.jouwir.cn/862266.Doc
gff.jouwir.cn/222020.Doc
gff.jouwir.cn/206668.Doc
gff.jouwir.cn/466686.Doc
gff.jouwir.cn/220822.Doc
gff.jouwir.cn/606604.Doc
gfd.jouwir.cn/844866.Doc
gfd.jouwir.cn/424006.Doc
gfd.jouwir.cn/846420.Doc
gfd.jouwir.cn/739111.Doc
gfd.jouwir.cn/822680.Doc
gfd.jouwir.cn/624286.Doc
gfd.jouwir.cn/822482.Doc
gfd.jouwir.cn/246440.Doc
gfd.jouwir.cn/795171.Doc
gfd.jouwir.cn/408628.Doc
gfs.jouwir.cn/466820.Doc
gfs.jouwir.cn/884882.Doc
gfs.jouwir.cn/530534.Doc
gfs.jouwir.cn/228000.Doc
gfs.jouwir.cn/014179.Doc
gfs.jouwir.cn/482064.Doc
gfs.jouwir.cn/593759.Doc
gfs.jouwir.cn/208228.Doc
gfs.jouwir.cn/600244.Doc
gfs.jouwir.cn/806400.Doc
gfa.jouwir.cn/484464.Doc
gfa.jouwir.cn/404408.Doc
gfa.jouwir.cn/200840.Doc
gfa.jouwir.cn/271410.Doc
gfa.jouwir.cn/420480.Doc
gfa.jouwir.cn/004260.Doc
gfa.jouwir.cn/408060.Doc
gfa.jouwir.cn/046846.Doc
gfa.jouwir.cn/464640.Doc
gfa.jouwir.cn/628600.Doc
gfp.jouwir.cn/088666.Doc
gfp.jouwir.cn/682026.Doc
gfp.jouwir.cn/688006.Doc
gfp.jouwir.cn/122012.Doc
gfp.jouwir.cn/622826.Doc
gfp.jouwir.cn/846440.Doc
gfp.jouwir.cn/286886.Doc
gfp.jouwir.cn/482004.Doc
gfp.jouwir.cn/840884.Doc
gfp.jouwir.cn/022480.Doc
gfo.jouwir.cn/688448.Doc
gfo.jouwir.cn/282422.Doc
gfo.jouwir.cn/648888.Doc
gfo.jouwir.cn/499039.Doc
gfo.jouwir.cn/822800.Doc
gfo.jouwir.cn/222048.Doc
gfo.jouwir.cn/222626.Doc
gfo.jouwir.cn/907306.Doc
gfo.jouwir.cn/600086.Doc
gfo.jouwir.cn/642286.Doc
gfi.jouwir.cn/666622.Doc
gfi.jouwir.cn/075344.Doc
gfi.jouwir.cn/240028.Doc
gfi.jouwir.cn/734460.Doc
gfi.jouwir.cn/446466.Doc
gfi.jouwir.cn/048220.Doc
gfi.jouwir.cn/802626.Doc
gfi.jouwir.cn/068862.Doc
gfi.jouwir.cn/006000.Doc
gfi.jouwir.cn/313351.Doc
gfu.jouwir.cn/620688.Doc
gfu.jouwir.cn/046062.Doc
gfu.jouwir.cn/882280.Doc
gfu.jouwir.cn/844826.Doc
gfu.jouwir.cn/662282.Doc
gfu.jouwir.cn/064400.Doc
gfu.jouwir.cn/886622.Doc
gfu.jouwir.cn/446262.Doc
gfu.jouwir.cn/800046.Doc
gfu.jouwir.cn/824868.Doc
gfy.jouwir.cn/062822.Doc
gfy.jouwir.cn/280608.Doc
gfy.jouwir.cn/202820.Doc
gfy.jouwir.cn/042648.Doc
gfy.jouwir.cn/428048.Doc
gfy.jouwir.cn/062884.Doc
gfy.jouwir.cn/260248.Doc
gfy.jouwir.cn/700022.Doc
gfy.jouwir.cn/286026.Doc
gfy.jouwir.cn/246460.Doc
gft.jouwir.cn/284822.Doc
gft.jouwir.cn/028628.Doc
gft.jouwir.cn/089999.Doc
gft.jouwir.cn/248004.Doc
gft.jouwir.cn/894067.Doc
gft.jouwir.cn/642440.Doc
gft.jouwir.cn/406680.Doc
gft.jouwir.cn/820220.Doc
gft.jouwir.cn/604644.Doc
gft.jouwir.cn/400626.Doc
gfr.jouwir.cn/202282.Doc
gfr.jouwir.cn/066200.Doc
gfr.jouwir.cn/480280.Doc
gfr.jouwir.cn/644268.Doc
gfr.jouwir.cn/597177.Doc
gfr.jouwir.cn/860842.Doc
gfr.jouwir.cn/983226.Doc
gfr.jouwir.cn/600024.Doc
gfr.jouwir.cn/403186.Doc
gfr.jouwir.cn/222422.Doc
gfe.jouwir.cn/220626.Doc
gfe.jouwir.cn/440886.Doc
gfe.jouwir.cn/282682.Doc
gfe.jouwir.cn/420886.Doc
gfe.jouwir.cn/202228.Doc
gfe.jouwir.cn/798118.Doc
gfe.jouwir.cn/644604.Doc
gfe.jouwir.cn/824264.Doc
gfe.jouwir.cn/480868.Doc
gfe.jouwir.cn/791064.Doc
gfw.jouwir.cn/802280.Doc
gfw.jouwir.cn/482066.Doc
gfw.jouwir.cn/800026.Doc
gfw.jouwir.cn/660662.Doc
gfw.jouwir.cn/264284.Doc
gfw.jouwir.cn/080488.Doc
gfw.jouwir.cn/882220.Doc
gfw.jouwir.cn/040020.Doc
gfw.jouwir.cn/066628.Doc
gfw.jouwir.cn/440802.Doc
gfq.jouwir.cn/719957.Doc
gfq.jouwir.cn/080886.Doc
gfq.jouwir.cn/262020.Doc
gfq.jouwir.cn/546425.Doc
gfq.jouwir.cn/008688.Doc
gfq.jouwir.cn/646628.Doc
gfq.jouwir.cn/064691.Doc
gfq.jouwir.cn/224240.Doc
gfq.jouwir.cn/886840.Doc
gfq.jouwir.cn/020468.Doc
gdm.jouwir.cn/800482.Doc
gdm.jouwir.cn/866886.Doc
gdm.jouwir.cn/620200.Doc
gdm.jouwir.cn/428282.Doc
gdm.jouwir.cn/028085.Doc
gdm.jouwir.cn/660004.Doc
gdm.jouwir.cn/066804.Doc
gdm.jouwir.cn/004685.Doc
gdm.jouwir.cn/424640.Doc
gdm.jouwir.cn/648800.Doc
gdn.jouwir.cn/240400.Doc
gdn.jouwir.cn/577715.Doc
gdn.jouwir.cn/860202.Doc
gdn.jouwir.cn/822644.Doc
gdn.jouwir.cn/226264.Doc
gdn.jouwir.cn/280888.Doc
gdn.jouwir.cn/757151.Doc
gdn.jouwir.cn/000460.Doc
gdn.jouwir.cn/820682.Doc
gdn.jouwir.cn/024844.Doc
gdb.jouwir.cn/620604.Doc
gdb.jouwir.cn/268602.Doc
gdb.jouwir.cn/682488.Doc
gdb.jouwir.cn/484266.Doc
gdb.jouwir.cn/666464.Doc
gdb.jouwir.cn/208402.Doc
gdb.jouwir.cn/804484.Doc
gdb.jouwir.cn/824622.Doc
gdb.jouwir.cn/787963.Doc
gdb.jouwir.cn/686200.Doc
gdv.jouwir.cn/040662.Doc
gdv.jouwir.cn/062462.Doc
gdv.jouwir.cn/004464.Doc
gdv.jouwir.cn/426862.Doc
gdv.jouwir.cn/606440.Doc
gdv.jouwir.cn/640822.Doc
gdv.jouwir.cn/288860.Doc
gdv.jouwir.cn/118361.Doc
gdv.jouwir.cn/002604.Doc
gdv.jouwir.cn/846600.Doc
gdc.jouwir.cn/640482.Doc
gdc.jouwir.cn/460028.Doc
gdc.jouwir.cn/046460.Doc
gdc.jouwir.cn/064464.Doc
gdc.jouwir.cn/402660.Doc
gdc.jouwir.cn/420460.Doc
gdc.jouwir.cn/281074.Doc
gdc.jouwir.cn/864220.Doc
gdc.jouwir.cn/222068.Doc
gdx.jouwir.cn/866082.Doc
gdx.jouwir.cn/668264.Doc
gdx.jouwir.cn/886228.Doc
gdx.jouwir.cn/662800.Doc
gdx.jouwir.cn/884242.Doc
gdx.jouwir.cn/918675.Doc
gdx.jouwir.cn/844040.Doc
gdx.jouwir.cn/826820.Doc
gdx.jouwir.cn/012996.Doc
gdx.jouwir.cn/460662.Doc
