Python文件处理与IO操作

一、文件读写

with open('example.txt', 'r', encoding='utf-8') as f:
    content = f.read()        # 全部读取
    line = f.readline()       # 一行
    lines = f.readlines()     # 所有行

with open('output.txt', 'w', encoding='utf-8') as f:
    f.write("第一行\n")

# 追加
with open('log.txt', 'a') as f:
    f.write("新日志\n")

# 二进制
with open('image.png', 'rb') as f:
    data = f.read()

文件模式：r/w/a/x/b/+


二、高效处理大文件

def process_large(filepath):
    with open(filepath) as f:
        for line in f:  # 文件对象本身就是迭代器
            yield line.strip()

def count_lines(filepath):
    return sum(1 for _ in open(filepath, 'rb'))

def read_chunks(filepath, size=8192):
    with open(filepath, 'rb') as f:
        while chunk := f.read(size):  # walrus operator
            yield chunk


三、pathlib模块

from pathlib import Path

p = Path('/home/user/docs')
config = Path('project') / 'config' / 'settings.yaml'

print(p.name)      # 文件名
print(p.stem)      # 无扩展名
print(p.suffix)    # 扩展名
print(p.parent)    # 父目录
print(p.parts)     # 所有部分

# 读写
Path('file.txt').write_text("Hello", encoding='utf-8')
content = Path('file.txt').read_text(encoding='utf-8')

# 遍历
for item in Path('.').iterdir():
    if item.is_file():
        print(item.name)

# glob
python_files = list(Path('src').glob('**/*.py'))
configs = list(Path('.').glob('*.{yaml,yml,json}'))


四、临时文件

import tempfile

with tempfile.NamedTemporaryFile(mode='w', suffix='.txt') as f:
    f.write("临时数据")
    path = f.name

with tempfile.TemporaryDirectory() as tmpdir:
    Path(tmpdir, 'file.txt').write_text("内容")


五、CSV处理

import csv

# 写入
with open('data.csv', 'w', newline='', encoding='utf-8') as f:
    writer = csv.writer(f)
    writer.writerow(['姓名', '年龄'])
    writer.writerows([('张三', 30), ('李四', 25)])

# DictReader
with open('data.csv', 'r', encoding='utf-8') as f:
    for row in csv.DictReader(f):
        print(row['姓名'], row['年龄'])

# DictWriter
with open('out.csv', 'w', newline='') as f:
    writer = csv.DictWriter(f, fieldnames=['name', 'age'])
    writer.writeheader()
    writer.writerow({'name': 'Alice', 'age': 30})


六、JSON处理

import json

with open('data.json', 'w', encoding='utf-8') as f:
    json.dump({'name': '张三'}, f, ensure_ascii=False, indent=2)

with open('data.json', 'r') as f:
    data = json.load(f)

# 自定义编码
from datetime import datetime, date
class CustomEncoder(json.JSONEncoder):
    def default(self, obj):
        if isinstance(obj, (datetime, date)):
            return obj.isoformat()
        return super().default(obj)


七、文件压缩

import zipfile, tarfile, gzip

# ZIP
with zipfile.ZipFile('archive.zip', 'w', zipfile.ZIP_DEFLATED) as zf:
    zf.write('file1.txt')
    zf.writestr('gen.txt', '动态内容')

# TAR
with tarfile.open('archive.tar.gz', 'w:gz') as tar:
    tar.add('src/', arcname='source')

# GZIP
with gzip.open('data.txt.gz', 'wt') as f:
    f.write("压缩的内容")


八、文件监控

from pathlib import Path
import time

class Watcher:
    def __init__(self, path):
        self.path = Path(path)
        self._mtimes = {}

    def check(self):
        changes = []
        for f in self.path.glob('**/*.py'):
            mtime = f.stat().st_mtime
            if str(f) in self._mtimes and self._mtimes[str(f)] != mtime:
                changes.append(f)
            self._mtimes[str(f)] = mtime
        return changes

总结：使用pathlib替代os.path，大文件用迭代器处理，明确指定编码，with语句管理资源。

epk.xkzjnc.cn/44204.Doc
epk.xkzjnc.cn/22880.Doc
epk.xkzjnc.cn/80626.Doc
epk.xkzjnc.cn/06820.Doc
epk.xkzjnc.cn/40422.Doc
epk.xkzjnc.cn/64220.Doc
epk.xkzjnc.cn/08048.Doc
epk.xkzjnc.cn/88262.Doc
epk.xkzjnc.cn/46804.Doc
epk.xkzjnc.cn/00022.Doc
epj.xkzjnc.cn/40622.Doc
epj.xkzjnc.cn/44886.Doc
epj.xkzjnc.cn/80602.Doc
epj.xkzjnc.cn/84482.Doc
epj.xkzjnc.cn/40682.Doc
epj.xkzjnc.cn/80066.Doc
epj.xkzjnc.cn/84684.Doc
epj.xkzjnc.cn/40462.Doc
epj.xkzjnc.cn/48808.Doc
epj.xkzjnc.cn/42428.Doc
eph.xkzjnc.cn/88402.Doc
eph.xkzjnc.cn/24868.Doc
eph.xkzjnc.cn/86608.Doc
eph.xkzjnc.cn/60626.Doc
eph.xkzjnc.cn/04840.Doc
eph.xkzjnc.cn/04026.Doc
eph.xkzjnc.cn/06420.Doc
eph.xkzjnc.cn/00042.Doc
eph.xkzjnc.cn/42664.Doc
eph.xkzjnc.cn/00602.Doc
epg.xkzjnc.cn/22060.Doc
epg.xkzjnc.cn/08262.Doc
epg.xkzjnc.cn/84846.Doc
epg.xkzjnc.cn/44682.Doc
epg.xkzjnc.cn/68600.Doc
epg.xkzjnc.cn/60464.Doc
epg.xkzjnc.cn/86626.Doc
epg.xkzjnc.cn/42080.Doc
epg.xkzjnc.cn/82024.Doc
epg.xkzjnc.cn/66226.Doc
epf.xkzjnc.cn/66066.Doc
epf.xkzjnc.cn/84600.Doc
epf.xkzjnc.cn/80204.Doc
epf.xkzjnc.cn/60404.Doc
epf.xkzjnc.cn/82606.Doc
epf.xkzjnc.cn/42848.Doc
epf.xkzjnc.cn/04244.Doc
epf.xkzjnc.cn/04062.Doc
epf.xkzjnc.cn/48268.Doc
epf.xkzjnc.cn/26624.Doc
epd.xkzjnc.cn/86864.Doc
epd.xkzjnc.cn/40242.Doc
epd.xkzjnc.cn/40880.Doc
epd.xkzjnc.cn/82042.Doc
epd.xkzjnc.cn/86882.Doc
epd.xkzjnc.cn/08266.Doc
epd.xkzjnc.cn/80800.Doc
epd.xkzjnc.cn/60024.Doc
epd.xkzjnc.cn/48202.Doc
epd.xkzjnc.cn/88002.Doc
eps.xkzjnc.cn/60404.Doc
eps.xkzjnc.cn/64046.Doc
eps.xkzjnc.cn/84462.Doc
eps.xkzjnc.cn/06804.Doc
eps.xkzjnc.cn/64006.Doc
eps.xkzjnc.cn/00406.Doc
eps.xkzjnc.cn/46006.Doc
eps.xkzjnc.cn/26446.Doc
eps.xkzjnc.cn/44664.Doc
eps.xkzjnc.cn/66266.Doc
epa.xkzjnc.cn/06044.Doc
epa.xkzjnc.cn/04466.Doc
epa.xkzjnc.cn/88066.Doc
epa.xkzjnc.cn/60428.Doc
epa.xkzjnc.cn/46682.Doc
epa.xkzjnc.cn/06606.Doc
epa.xkzjnc.cn/64808.Doc
epa.xkzjnc.cn/26604.Doc
epa.xkzjnc.cn/24880.Doc
epa.xkzjnc.cn/08042.Doc
epp.xkzjnc.cn/06424.Doc
epp.xkzjnc.cn/44028.Doc
epp.xkzjnc.cn/86260.Doc
epp.xkzjnc.cn/02408.Doc
epp.xkzjnc.cn/00604.Doc
epp.xkzjnc.cn/08480.Doc
epp.xkzjnc.cn/82204.Doc
epp.xkzjnc.cn/26206.Doc
epp.xkzjnc.cn/02600.Doc
epp.xkzjnc.cn/02424.Doc
epo.xkzjnc.cn/44080.Doc
epo.xkzjnc.cn/00880.Doc
epo.xkzjnc.cn/42662.Doc
epo.xkzjnc.cn/08466.Doc
epo.xkzjnc.cn/06842.Doc
epo.xkzjnc.cn/08660.Doc
epo.xkzjnc.cn/88026.Doc
epo.xkzjnc.cn/02840.Doc
epo.xkzjnc.cn/00044.Doc
epo.xkzjnc.cn/86248.Doc
epi.xkzjnc.cn/60222.Doc
epi.xkzjnc.cn/26082.Doc
epi.xkzjnc.cn/28242.Doc
epi.xkzjnc.cn/64486.Doc
epi.xkzjnc.cn/40202.Doc
epi.xkzjnc.cn/84622.Doc
epi.xkzjnc.cn/22266.Doc
epi.xkzjnc.cn/28202.Doc
epi.xkzjnc.cn/86446.Doc
epi.xkzjnc.cn/02004.Doc
epu.xkzjnc.cn/22608.Doc
epu.xkzjnc.cn/00240.Doc
epu.xkzjnc.cn/04448.Doc
epu.xkzjnc.cn/64446.Doc
epu.xkzjnc.cn/60286.Doc
epu.xkzjnc.cn/08626.Doc
epu.xkzjnc.cn/06026.Doc
epu.xkzjnc.cn/22806.Doc
epu.xkzjnc.cn/84068.Doc
epu.xkzjnc.cn/06620.Doc
epy.xkzjnc.cn/02640.Doc
epy.xkzjnc.cn/66264.Doc
epy.xkzjnc.cn/86066.Doc
epy.xkzjnc.cn/80804.Doc
epy.xkzjnc.cn/40404.Doc
epy.xkzjnc.cn/42848.Doc
epy.xkzjnc.cn/44282.Doc
epy.xkzjnc.cn/04422.Doc
epy.xkzjnc.cn/82860.Doc
epy.xkzjnc.cn/44262.Doc
ept.xkzjnc.cn/28028.Doc
ept.xkzjnc.cn/80204.Doc
ept.xkzjnc.cn/08660.Doc
ept.xkzjnc.cn/46262.Doc
ept.xkzjnc.cn/00240.Doc
ept.xkzjnc.cn/66646.Doc
ept.xkzjnc.cn/86884.Doc
ept.xkzjnc.cn/68460.Doc
ept.xkzjnc.cn/84828.Doc
ept.xkzjnc.cn/26084.Doc
epr.xkzjnc.cn/28848.Doc
epr.xkzjnc.cn/64084.Doc
epr.xkzjnc.cn/00004.Doc
epr.xkzjnc.cn/62468.Doc
epr.xkzjnc.cn/66220.Doc
epr.xkzjnc.cn/80604.Doc
epr.xkzjnc.cn/62680.Doc
epr.xkzjnc.cn/26866.Doc
epr.xkzjnc.cn/62662.Doc
epr.xkzjnc.cn/40220.Doc
epe.xkzjnc.cn/24004.Doc
epe.xkzjnc.cn/68844.Doc
epe.xkzjnc.cn/46662.Doc
epe.xkzjnc.cn/28420.Doc
epe.xkzjnc.cn/08860.Doc
epe.xkzjnc.cn/00024.Doc
epe.xkzjnc.cn/86684.Doc
epe.xkzjnc.cn/00448.Doc
epe.xkzjnc.cn/48842.Doc
epe.xkzjnc.cn/88682.Doc
epw.xkzjnc.cn/22028.Doc
epw.xkzjnc.cn/62804.Doc
epw.xkzjnc.cn/20208.Doc
epw.xkzjnc.cn/86086.Doc
epw.xkzjnc.cn/46408.Doc
epw.xkzjnc.cn/48268.Doc
epw.xkzjnc.cn/26404.Doc
epw.xkzjnc.cn/06208.Doc
epw.xkzjnc.cn/46426.Doc
epw.xkzjnc.cn/08024.Doc
epq.xkzjnc.cn/44608.Doc
epq.xkzjnc.cn/82606.Doc
epq.xkzjnc.cn/22202.Doc
epq.xkzjnc.cn/44826.Doc
epq.xkzjnc.cn/88466.Doc
epq.xkzjnc.cn/06084.Doc
epq.xkzjnc.cn/88406.Doc
epq.xkzjnc.cn/24460.Doc
epq.xkzjnc.cn/42840.Doc
epq.xkzjnc.cn/84484.Doc
eom.xkzjnc.cn/68004.Doc
eom.xkzjnc.cn/62248.Doc
eom.xkzjnc.cn/08620.Doc
eom.xkzjnc.cn/80860.Doc
eom.xkzjnc.cn/24200.Doc
eom.xkzjnc.cn/66686.Doc
eom.xkzjnc.cn/42222.Doc
eom.xkzjnc.cn/80886.Doc
eom.xkzjnc.cn/62844.Doc
eom.xkzjnc.cn/88086.Doc
eon.xkzjnc.cn/20628.Doc
eon.xkzjnc.cn/20004.Doc
eon.xkzjnc.cn/42202.Doc
eon.xkzjnc.cn/02644.Doc
eon.xkzjnc.cn/88486.Doc
eon.xkzjnc.cn/84006.Doc
eon.xkzjnc.cn/08246.Doc
eon.xkzjnc.cn/80826.Doc
eon.xkzjnc.cn/26622.Doc
eon.xkzjnc.cn/62406.Doc
eob.xkzjnc.cn/04442.Doc
eob.xkzjnc.cn/62648.Doc
eob.xkzjnc.cn/28020.Doc
eob.xkzjnc.cn/60428.Doc
eob.xkzjnc.cn/04482.Doc
eob.xkzjnc.cn/60204.Doc
eob.xkzjnc.cn/08640.Doc
eob.xkzjnc.cn/88468.Doc
eob.xkzjnc.cn/46844.Doc
eob.xkzjnc.cn/22606.Doc
eov.xkzjnc.cn/08220.Doc
eov.xkzjnc.cn/68442.Doc
eov.xkzjnc.cn/20820.Doc
eov.xkzjnc.cn/42008.Doc
eov.xkzjnc.cn/66400.Doc
eov.xkzjnc.cn/26660.Doc
eov.xkzjnc.cn/20228.Doc
eov.xkzjnc.cn/88200.Doc
eov.xkzjnc.cn/80000.Doc
eov.xkzjnc.cn/62222.Doc
eoc.xkzjnc.cn/48640.Doc
eoc.xkzjnc.cn/82048.Doc
eoc.xkzjnc.cn/24066.Doc
eoc.xkzjnc.cn/00846.Doc
eoc.xkzjnc.cn/20020.Doc
eoc.xkzjnc.cn/04608.Doc
eoc.xkzjnc.cn/86486.Doc
eoc.xkzjnc.cn/08824.Doc
eoc.xkzjnc.cn/82666.Doc
eoc.xkzjnc.cn/42866.Doc
eox.xkzjnc.cn/40884.Doc
eox.xkzjnc.cn/44060.Doc
eox.xkzjnc.cn/02028.Doc
eox.xkzjnc.cn/60246.Doc
eox.xkzjnc.cn/82860.Doc
eox.xkzjnc.cn/24086.Doc
eox.xkzjnc.cn/62802.Doc
eox.xkzjnc.cn/06200.Doc
eox.xkzjnc.cn/48060.Doc
eox.xkzjnc.cn/44846.Doc
eoz.xkzjnc.cn/22006.Doc
eoz.xkzjnc.cn/86006.Doc
eoz.xkzjnc.cn/04820.Doc
eoz.xkzjnc.cn/42600.Doc
eoz.xkzjnc.cn/48066.Doc
eoz.xkzjnc.cn/20448.Doc
eoz.xkzjnc.cn/68240.Doc
eoz.xkzjnc.cn/68442.Doc
eoz.xkzjnc.cn/48424.Doc
eoz.xkzjnc.cn/22420.Doc
eol.xkzjnc.cn/80208.Doc
eol.xkzjnc.cn/64680.Doc
eol.xkzjnc.cn/84486.Doc
eol.xkzjnc.cn/06040.Doc
eol.xkzjnc.cn/60028.Doc
eol.xkzjnc.cn/66482.Doc
eol.xkzjnc.cn/06640.Doc
eol.xkzjnc.cn/66426.Doc
eol.xkzjnc.cn/48208.Doc
eol.xkzjnc.cn/82062.Doc
eok.xkzjnc.cn/82668.Doc
eok.xkzjnc.cn/22444.Doc
eok.xkzjnc.cn/84626.Doc
eok.xkzjnc.cn/06864.Doc
eok.xkzjnc.cn/88086.Doc
eok.xkzjnc.cn/66200.Doc
eok.xkzjnc.cn/88446.Doc
eok.xkzjnc.cn/04644.Doc
eok.xkzjnc.cn/48480.Doc
eok.xkzjnc.cn/84404.Doc
eoj.xkzjnc.cn/28048.Doc
eoj.xkzjnc.cn/02640.Doc
eoj.xkzjnc.cn/06824.Doc
eoj.xkzjnc.cn/04600.Doc
eoj.xkzjnc.cn/06060.Doc
eoj.xkzjnc.cn/44060.Doc
eoj.xkzjnc.cn/86860.Doc
eoj.xkzjnc.cn/24240.Doc
eoj.xkzjnc.cn/28068.Doc
eoj.xkzjnc.cn/20604.Doc
eoh.xkzjnc.cn/88244.Doc
eoh.xkzjnc.cn/06424.Doc
eoh.xkzjnc.cn/84000.Doc
eoh.xkzjnc.cn/22046.Doc
eoh.xkzjnc.cn/44686.Doc
eoh.xkzjnc.cn/66400.Doc
eoh.xkzjnc.cn/20008.Doc
eoh.xkzjnc.cn/86244.Doc
eoh.xkzjnc.cn/66488.Doc
eoh.xkzjnc.cn/84880.Doc
eog.xkzjnc.cn/60626.Doc
eog.xkzjnc.cn/86626.Doc
eog.xkzjnc.cn/46826.Doc
eog.xkzjnc.cn/20848.Doc
eog.xkzjnc.cn/06608.Doc
eog.xkzjnc.cn/00464.Doc
eog.xkzjnc.cn/26488.Doc
eog.xkzjnc.cn/20886.Doc
eog.xkzjnc.cn/24860.Doc
eog.xkzjnc.cn/66820.Doc
eof.xkzjnc.cn/82048.Doc
eof.xkzjnc.cn/64484.Doc
eof.xkzjnc.cn/04822.Doc
eof.xkzjnc.cn/64460.Doc
eof.xkzjnc.cn/24066.Doc
eof.xkzjnc.cn/84446.Doc
eof.xkzjnc.cn/84828.Doc
eof.xkzjnc.cn/08862.Doc
eof.xkzjnc.cn/48668.Doc
eof.xkzjnc.cn/24646.Doc
eod.xkzjnc.cn/84026.Doc
eod.xkzjnc.cn/60262.Doc
eod.xkzjnc.cn/08442.Doc
eod.xkzjnc.cn/22264.Doc
eod.xkzjnc.cn/04864.Doc
eod.xkzjnc.cn/42206.Doc
eod.xkzjnc.cn/28826.Doc
eod.xkzjnc.cn/46228.Doc
eod.xkzjnc.cn/84262.Doc
eod.xkzjnc.cn/40442.Doc
eos.xkzjnc.cn/82046.Doc
eos.xkzjnc.cn/88446.Doc
eos.xkzjnc.cn/66682.Doc
eos.xkzjnc.cn/60244.Doc
eos.xkzjnc.cn/42646.Doc
eos.xkzjnc.cn/26080.Doc
eos.xkzjnc.cn/66622.Doc
eos.xkzjnc.cn/62440.Doc
eos.xkzjnc.cn/46248.Doc
eos.xkzjnc.cn/44006.Doc
eoa.xkzjnc.cn/02004.Doc
eoa.xkzjnc.cn/28664.Doc
eoa.xkzjnc.cn/26688.Doc
eoa.xkzjnc.cn/64844.Doc
eoa.xkzjnc.cn/26660.Doc
eoa.xkzjnc.cn/62422.Doc
eoa.xkzjnc.cn/48202.Doc
eoa.xkzjnc.cn/62606.Doc
eoa.xkzjnc.cn/22026.Doc
eoa.xkzjnc.cn/22866.Doc
eop.xkzjnc.cn/02086.Doc
eop.xkzjnc.cn/00848.Doc
eop.xkzjnc.cn/24202.Doc
eop.xkzjnc.cn/46888.Doc
eop.xkzjnc.cn/28008.Doc
eop.xkzjnc.cn/20028.Doc
eop.xkzjnc.cn/24682.Doc
eop.xkzjnc.cn/66626.Doc
eop.xkzjnc.cn/20864.Doc
eop.xkzjnc.cn/84002.Doc
eoo.xkzjnc.cn/26006.Doc
eoo.xkzjnc.cn/88040.Doc
eoo.xkzjnc.cn/08402.Doc
eoo.xkzjnc.cn/40442.Doc
eoo.xkzjnc.cn/42646.Doc
eoo.xkzjnc.cn/64264.Doc
eoo.xkzjnc.cn/02248.Doc
eoo.xkzjnc.cn/64022.Doc
eoo.xkzjnc.cn/26604.Doc
eoo.xkzjnc.cn/60024.Doc
eoi.xkzjnc.cn/60262.Doc
eoi.xkzjnc.cn/88688.Doc
eoi.xkzjnc.cn/06264.Doc
eoi.xkzjnc.cn/80626.Doc
eoi.xkzjnc.cn/64244.Doc
eoi.xkzjnc.cn/42044.Doc
eoi.xkzjnc.cn/22480.Doc
eoi.xkzjnc.cn/22628.Doc
eoi.xkzjnc.cn/84080.Doc
eoi.xkzjnc.cn/42468.Doc
eou.xkzjnc.cn/06884.Doc
eou.xkzjnc.cn/22266.Doc
eou.xkzjnc.cn/84480.Doc
eou.xkzjnc.cn/02000.Doc
eou.xkzjnc.cn/46048.Doc
eou.xkzjnc.cn/00680.Doc
eou.xkzjnc.cn/88086.Doc
eou.xkzjnc.cn/82286.Doc
eou.xkzjnc.cn/60602.Doc
eou.xkzjnc.cn/48806.Doc
eoy.xkzjnc.cn/22406.Doc
eoy.xkzjnc.cn/80862.Doc
eoy.xkzjnc.cn/40044.Doc
eoy.xkzjnc.cn/88048.Doc
eoy.xkzjnc.cn/22002.Doc
eoy.xkzjnc.cn/22828.Doc
eoy.xkzjnc.cn/20208.Doc
eoy.xkzjnc.cn/44484.Doc
eoy.xkzjnc.cn/42866.Doc
eoy.xkzjnc.cn/04004.Doc
eot.xkzjnc.cn/86666.Doc
eot.xkzjnc.cn/06686.Doc
eot.xkzjnc.cn/22086.Doc
eot.xkzjnc.cn/26222.Doc
eot.xkzjnc.cn/82262.Doc
eot.xkzjnc.cn/60420.Doc
eot.xkzjnc.cn/48044.Doc
eot.xkzjnc.cn/48068.Doc
eot.xkzjnc.cn/84484.Doc
eot.xkzjnc.cn/84806.Doc
eor.xkzjnc.cn/60820.Doc
eor.xkzjnc.cn/84224.Doc
eor.xkzjnc.cn/60004.Doc
eor.xkzjnc.cn/06420.Doc
eor.xkzjnc.cn/24222.Doc
eor.xkzjnc.cn/28626.Doc
eor.xkzjnc.cn/80822.Doc
eor.xkzjnc.cn/00264.Doc
eor.xkzjnc.cn/68002.Doc
eor.xkzjnc.cn/22466.Doc
eoe.xkzjnc.cn/86800.Doc
eoe.xkzjnc.cn/62600.Doc
eoe.xkzjnc.cn/20208.Doc
eoe.xkzjnc.cn/28288.Doc
eoe.xkzjnc.cn/40288.Doc
eoe.xkzjnc.cn/44468.Doc
eoe.xkzjnc.cn/68284.Doc
eoe.xkzjnc.cn/04266.Doc
eoe.xkzjnc.cn/08260.Doc
eoe.xkzjnc.cn/06026.Doc
eow.xkzjnc.cn/68066.Doc
eow.xkzjnc.cn/62828.Doc
eow.xkzjnc.cn/28008.Doc
eow.xkzjnc.cn/22242.Doc
eow.xkzjnc.cn/86466.Doc
eow.xkzjnc.cn/84642.Doc
eow.xkzjnc.cn/06002.Doc
eow.xkzjnc.cn/06440.Doc
eow.xkzjnc.cn/82488.Doc
eow.xkzjnc.cn/44028.Doc
eoq.xkzjnc.cn/40428.Doc
eoq.xkzjnc.cn/80462.Doc
eoq.xkzjnc.cn/64622.Doc
eoq.xkzjnc.cn/62066.Doc
eoq.xkzjnc.cn/22424.Doc
eoq.xkzjnc.cn/00846.Doc
eoq.xkzjnc.cn/68808.Doc
eoq.xkzjnc.cn/66248.Doc
eoq.xkzjnc.cn/68844.Doc
eoq.xkzjnc.cn/86406.Doc
eim.xkzjnc.cn/08026.Doc
eim.xkzjnc.cn/86644.Doc
eim.xkzjnc.cn/60646.Doc
eim.xkzjnc.cn/48686.Doc
eim.xkzjnc.cn/04844.Doc
eim.xkzjnc.cn/20666.Doc
eim.xkzjnc.cn/68244.Doc
eim.xkzjnc.cn/24466.Doc
eim.xkzjnc.cn/80424.Doc
eim.xkzjnc.cn/44644.Doc
ein.xkzjnc.cn/08224.Doc
ein.xkzjnc.cn/62604.Doc
ein.xkzjnc.cn/08606.Doc
ein.xkzjnc.cn/60220.Doc
ein.xkzjnc.cn/40600.Doc
ein.xkzjnc.cn/68286.Doc
ein.xkzjnc.cn/04802.Doc
ein.xkzjnc.cn/62660.Doc
ein.xkzjnc.cn/80446.Doc
ein.xkzjnc.cn/08600.Doc
eib.xkzjnc.cn/24846.Doc
eib.xkzjnc.cn/80606.Doc
eib.xkzjnc.cn/86684.Doc
eib.xkzjnc.cn/24848.Doc
eib.xkzjnc.cn/28248.Doc
eib.xkzjnc.cn/68042.Doc
eib.xkzjnc.cn/62000.Doc
eib.xkzjnc.cn/02264.Doc
eib.xkzjnc.cn/48662.Doc
eib.xkzjnc.cn/20466.Doc
eiv.xkzjnc.cn/86868.Doc
eiv.xkzjnc.cn/88864.Doc
eiv.xkzjnc.cn/88464.Doc
eiv.xkzjnc.cn/46080.Doc
eiv.xkzjnc.cn/42040.Doc
eiv.xkzjnc.cn/40440.Doc
eiv.xkzjnc.cn/68626.Doc
eiv.xkzjnc.cn/46622.Doc
eiv.xkzjnc.cn/26866.Doc
eiv.xkzjnc.cn/08268.Doc
eic.xkzjnc.cn/44864.Doc
eic.xkzjnc.cn/08882.Doc
eic.xkzjnc.cn/46868.Doc
eic.xkzjnc.cn/26608.Doc
eic.xkzjnc.cn/64224.Doc
eic.xkzjnc.cn/02220.Doc
eic.xkzjnc.cn/48244.Doc
eic.xkzjnc.cn/88802.Doc
eic.xkzjnc.cn/28064.Doc
eic.xkzjnc.cn/68200.Doc
