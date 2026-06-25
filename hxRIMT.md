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

qlo.LreLnc.cn/40080.Doc
qlo.LreLnc.cn/99319.Doc
qlo.LreLnc.cn/73337.Doc
qlo.LreLnc.cn/53559.Doc
qlo.LreLnc.cn/02286.Doc
qlo.LreLnc.cn/86886.Doc
qlo.LreLnc.cn/77715.Doc
qlo.LreLnc.cn/77135.Doc
qlo.LreLnc.cn/39973.Doc
qlo.LreLnc.cn/77799.Doc
qli.LreLnc.cn/00248.Doc
qli.LreLnc.cn/93511.Doc
qli.LreLnc.cn/17173.Doc
qli.LreLnc.cn/48206.Doc
qli.LreLnc.cn/17157.Doc
qli.LreLnc.cn/59991.Doc
qli.LreLnc.cn/57995.Doc
qli.LreLnc.cn/51793.Doc
qli.LreLnc.cn/97911.Doc
qli.LreLnc.cn/39737.Doc
qlu.LreLnc.cn/33333.Doc
qlu.LreLnc.cn/79331.Doc
qlu.LreLnc.cn/15755.Doc
qlu.LreLnc.cn/99919.Doc
qlu.LreLnc.cn/31333.Doc
qlu.LreLnc.cn/79919.Doc
qlu.LreLnc.cn/28442.Doc
qlu.LreLnc.cn/33115.Doc
qlu.LreLnc.cn/95115.Doc
qlu.LreLnc.cn/33351.Doc
qly.LreLnc.cn/51739.Doc
qly.LreLnc.cn/11951.Doc
qly.LreLnc.cn/71179.Doc
qly.LreLnc.cn/95133.Doc
qly.LreLnc.cn/51113.Doc
qly.LreLnc.cn/77111.Doc
qly.LreLnc.cn/57751.Doc
qly.LreLnc.cn/35171.Doc
qly.LreLnc.cn/55775.Doc
qly.LreLnc.cn/97915.Doc
qlt.LreLnc.cn/99319.Doc
qlt.LreLnc.cn/73137.Doc
qlt.LreLnc.cn/15991.Doc
qlt.LreLnc.cn/15975.Doc
qlt.LreLnc.cn/53599.Doc
qlt.LreLnc.cn/19579.Doc
qlt.LreLnc.cn/11939.Doc
qlt.LreLnc.cn/73959.Doc
qlt.LreLnc.cn/11793.Doc
qlt.LreLnc.cn/51775.Doc
qlr.LreLnc.cn/91795.Doc
qlr.LreLnc.cn/93759.Doc
qlr.LreLnc.cn/31591.Doc
qlr.LreLnc.cn/82208.Doc
qlr.LreLnc.cn/39117.Doc
qlr.LreLnc.cn/35759.Doc
qlr.LreLnc.cn/17555.Doc
qlr.LreLnc.cn/11195.Doc
qlr.LreLnc.cn/53595.Doc
qlr.LreLnc.cn/71795.Doc
qle.LreLnc.cn/71777.Doc
qle.LreLnc.cn/53373.Doc
qle.LreLnc.cn/77315.Doc
qle.LreLnc.cn/06440.Doc
qle.LreLnc.cn/71711.Doc
qle.LreLnc.cn/79377.Doc
qle.LreLnc.cn/79737.Doc
qle.LreLnc.cn/68648.Doc
qle.LreLnc.cn/00044.Doc
qle.LreLnc.cn/55155.Doc
qlw.LreLnc.cn/97917.Doc
qlw.LreLnc.cn/68602.Doc
qlw.LreLnc.cn/17997.Doc
qlw.LreLnc.cn/17959.Doc
qlw.LreLnc.cn/57995.Doc
qlw.LreLnc.cn/15731.Doc
qlw.LreLnc.cn/13539.Doc
qlw.LreLnc.cn/93153.Doc
qlw.LreLnc.cn/99557.Doc
qlw.LreLnc.cn/51759.Doc
qlq.LreLnc.cn/55555.Doc
qlq.LreLnc.cn/06082.Doc
qlq.LreLnc.cn/51199.Doc
qlq.LreLnc.cn/15379.Doc
qlq.LreLnc.cn/17973.Doc
qlq.LreLnc.cn/51739.Doc
qlq.LreLnc.cn/31355.Doc
qlq.LreLnc.cn/79779.Doc
qlq.LreLnc.cn/11535.Doc
qlq.LreLnc.cn/13375.Doc
qkm.LreLnc.cn/57399.Doc
qkm.LreLnc.cn/17573.Doc
qkm.LreLnc.cn/17559.Doc
qkm.LreLnc.cn/75311.Doc
qkm.LreLnc.cn/53913.Doc
qkm.LreLnc.cn/97915.Doc
qkm.LreLnc.cn/15955.Doc
qkm.LreLnc.cn/33759.Doc
qkm.LreLnc.cn/39153.Doc
qkm.LreLnc.cn/71959.Doc
qkn.LreLnc.cn/04662.Doc
qkn.LreLnc.cn/91371.Doc
qkn.LreLnc.cn/97731.Doc
qkn.LreLnc.cn/42808.Doc
qkn.LreLnc.cn/39375.Doc
qkn.LreLnc.cn/55135.Doc
qkn.LreLnc.cn/51393.Doc
qkn.LreLnc.cn/11513.Doc
qkn.LreLnc.cn/71719.Doc
qkn.LreLnc.cn/35951.Doc
qkb.LreLnc.cn/51331.Doc
qkb.LreLnc.cn/73375.Doc
qkb.LreLnc.cn/91731.Doc
qkb.LreLnc.cn/26060.Doc
qkb.LreLnc.cn/95513.Doc
qkb.LreLnc.cn/59535.Doc
qkb.LreLnc.cn/79777.Doc
qkb.LreLnc.cn/53593.Doc
qkb.LreLnc.cn/11733.Doc
qkb.LreLnc.cn/59773.Doc
qkv.LreLnc.cn/77333.Doc
qkv.LreLnc.cn/51171.Doc
qkv.LreLnc.cn/86020.Doc
qkv.LreLnc.cn/79359.Doc
qkv.LreLnc.cn/71919.Doc
qkv.LreLnc.cn/62604.Doc
qkv.LreLnc.cn/91137.Doc
qkv.LreLnc.cn/77171.Doc
qkv.LreLnc.cn/15733.Doc
qkv.LreLnc.cn/55791.Doc
qkc.LreLnc.cn/31397.Doc
qkc.LreLnc.cn/91737.Doc
qkc.LreLnc.cn/13375.Doc
qkc.LreLnc.cn/53319.Doc
qkc.LreLnc.cn/99177.Doc
qkc.LreLnc.cn/37195.Doc
qkc.LreLnc.cn/93193.Doc
qkc.LreLnc.cn/97737.Doc
qkc.LreLnc.cn/75395.Doc
qkc.LreLnc.cn/99971.Doc
qkx.LreLnc.cn/31911.Doc
qkx.LreLnc.cn/79357.Doc
qkx.LreLnc.cn/95177.Doc
qkx.LreLnc.cn/97559.Doc
qkx.LreLnc.cn/57315.Doc
qkx.LreLnc.cn/51711.Doc
qkx.LreLnc.cn/73933.Doc
qkx.LreLnc.cn/57553.Doc
qkx.LreLnc.cn/93597.Doc
qkx.LreLnc.cn/53511.Doc
qkz.LreLnc.cn/44242.Doc
qkz.LreLnc.cn/11137.Doc
qkz.LreLnc.cn/93737.Doc
qkz.LreLnc.cn/51159.Doc
qkz.LreLnc.cn/31197.Doc
qkz.LreLnc.cn/33539.Doc
qkz.LreLnc.cn/59371.Doc
qkz.LreLnc.cn/73951.Doc
qkz.LreLnc.cn/73131.Doc
qkz.LreLnc.cn/11995.Doc
qkl.LreLnc.cn/84822.Doc
qkl.LreLnc.cn/35957.Doc
qkl.LreLnc.cn/99531.Doc
qkl.LreLnc.cn/17193.Doc
qkl.LreLnc.cn/59715.Doc
qkl.LreLnc.cn/61157.Doc
qkl.LreLnc.cn/97753.Doc
qkl.LreLnc.cn/19377.Doc
qkl.LreLnc.cn/75951.Doc
qkl.LreLnc.cn/35131.Doc
qkk.LreLnc.cn/77197.Doc
qkk.LreLnc.cn/75515.Doc
qkk.LreLnc.cn/99399.Doc
qkk.LreLnc.cn/37935.Doc
qkk.LreLnc.cn/39357.Doc
qkk.LreLnc.cn/19777.Doc
qkk.LreLnc.cn/57177.Doc
qkk.LreLnc.cn/51711.Doc
qkk.LreLnc.cn/51915.Doc
qkk.LreLnc.cn/53937.Doc
qkj.LreLnc.cn/60446.Doc
qkj.LreLnc.cn/93973.Doc
qkj.LreLnc.cn/11959.Doc
qkj.LreLnc.cn/13331.Doc
qkj.LreLnc.cn/82808.Doc
qkj.LreLnc.cn/15919.Doc
qkj.LreLnc.cn/99591.Doc
qkj.LreLnc.cn/17519.Doc
qkj.LreLnc.cn/35351.Doc
qkj.LreLnc.cn/97517.Doc
qkh.LreLnc.cn/39513.Doc
qkh.LreLnc.cn/75735.Doc
qkh.LreLnc.cn/79377.Doc
qkh.LreLnc.cn/60860.Doc
qkh.LreLnc.cn/33733.Doc
qkh.LreLnc.cn/95919.Doc
qkh.LreLnc.cn/97377.Doc
qkh.LreLnc.cn/42424.Doc
qkh.LreLnc.cn/79559.Doc
qkh.LreLnc.cn/31353.Doc
qkg.LreLnc.cn/31133.Doc
qkg.LreLnc.cn/28002.Doc
qkg.LreLnc.cn/97393.Doc
qkg.LreLnc.cn/79513.Doc
qkg.LreLnc.cn/37557.Doc
qkg.LreLnc.cn/17795.Doc
qkg.LreLnc.cn/66020.Doc
qkg.LreLnc.cn/31513.Doc
qkg.LreLnc.cn/39779.Doc
qkg.LreLnc.cn/57791.Doc
qkf.LreLnc.cn/55331.Doc
qkf.LreLnc.cn/19195.Doc
qkf.LreLnc.cn/59917.Doc
qkf.LreLnc.cn/91911.Doc
qkf.LreLnc.cn/55533.Doc
qkf.LreLnc.cn/62824.Doc
qkf.LreLnc.cn/86486.Doc
qkf.LreLnc.cn/64620.Doc
qkf.LreLnc.cn/26482.Doc
qkf.LreLnc.cn/06648.Doc
qkd.LreLnc.cn/68862.Doc
qkd.LreLnc.cn/02026.Doc
qkd.LreLnc.cn/28064.Doc
qkd.LreLnc.cn/60222.Doc
qkd.LreLnc.cn/53519.Doc
qkd.LreLnc.cn/22024.Doc
qkd.LreLnc.cn/82040.Doc
qkd.LreLnc.cn/80442.Doc
qkd.LreLnc.cn/33355.Doc
qkd.LreLnc.cn/28020.Doc
qks.LreLnc.cn/46666.Doc
qks.LreLnc.cn/46606.Doc
qks.LreLnc.cn/04226.Doc
qks.LreLnc.cn/66424.Doc
qks.LreLnc.cn/02608.Doc
qks.LreLnc.cn/60006.Doc
qks.LreLnc.cn/86224.Doc
qks.LreLnc.cn/80402.Doc
qks.LreLnc.cn/82464.Doc
qks.LreLnc.cn/75382.Doc
qka.LreLnc.cn/00288.Doc
qka.LreLnc.cn/04064.Doc
qka.LreLnc.cn/26020.Doc
qka.LreLnc.cn/60222.Doc
qka.LreLnc.cn/42000.Doc
qka.LreLnc.cn/24402.Doc
qka.LreLnc.cn/62042.Doc
qka.LreLnc.cn/20282.Doc
qka.LreLnc.cn/86648.Doc
qka.LreLnc.cn/04802.Doc
qkp.LreLnc.cn/24648.Doc
qkp.LreLnc.cn/84000.Doc
qkp.LreLnc.cn/64206.Doc
qkp.LreLnc.cn/66862.Doc
qkp.LreLnc.cn/00886.Doc
qkp.LreLnc.cn/02244.Doc
qkp.LreLnc.cn/00268.Doc
qkp.LreLnc.cn/20420.Doc
qkp.LreLnc.cn/44226.Doc
qkp.LreLnc.cn/24042.Doc
qko.LreLnc.cn/66624.Doc
qko.LreLnc.cn/86806.Doc
qko.LreLnc.cn/82842.Doc
qko.LreLnc.cn/02024.Doc
qko.LreLnc.cn/86848.Doc
qko.LreLnc.cn/60082.Doc
qko.LreLnc.cn/06420.Doc
qko.LreLnc.cn/44462.Doc
qko.LreLnc.cn/66882.Doc
qko.LreLnc.cn/44644.Doc
qki.LreLnc.cn/28420.Doc
qki.LreLnc.cn/26264.Doc
qki.LreLnc.cn/06040.Doc
qki.LreLnc.cn/62862.Doc
qki.LreLnc.cn/06860.Doc
qki.LreLnc.cn/84402.Doc
qki.LreLnc.cn/82622.Doc
qki.LreLnc.cn/22222.Doc
qki.LreLnc.cn/80028.Doc
qki.LreLnc.cn/06400.Doc
qku.LreLnc.cn/80200.Doc
qku.LreLnc.cn/60806.Doc
qku.LreLnc.cn/06404.Doc
qku.LreLnc.cn/86624.Doc
qku.LreLnc.cn/20204.Doc
qku.LreLnc.cn/24802.Doc
qku.LreLnc.cn/06062.Doc
qku.LreLnc.cn/84024.Doc
qku.LreLnc.cn/66806.Doc
qku.LreLnc.cn/84262.Doc
qky.LreLnc.cn/08006.Doc
qky.LreLnc.cn/00642.Doc
qky.LreLnc.cn/48480.Doc
qky.LreLnc.cn/08442.Doc
qky.LreLnc.cn/24068.Doc
qky.LreLnc.cn/06408.Doc
qky.LreLnc.cn/48428.Doc
qky.LreLnc.cn/42664.Doc
qky.LreLnc.cn/44662.Doc
qky.LreLnc.cn/26082.Doc
qkt.LreLnc.cn/06606.Doc
qkt.LreLnc.cn/42226.Doc
qkt.LreLnc.cn/06862.Doc
qkt.LreLnc.cn/60868.Doc
qkt.LreLnc.cn/48280.Doc
qkt.LreLnc.cn/64800.Doc
qkt.LreLnc.cn/24604.Doc
qkt.LreLnc.cn/64244.Doc
qkt.LreLnc.cn/04204.Doc
qkt.LreLnc.cn/84608.Doc
qkr.LreLnc.cn/22664.Doc
qkr.LreLnc.cn/02242.Doc
qkr.LreLnc.cn/26660.Doc
qkr.LreLnc.cn/66660.Doc
qkr.LreLnc.cn/46400.Doc
qkr.LreLnc.cn/02282.Doc
qkr.LreLnc.cn/86668.Doc
qkr.LreLnc.cn/62640.Doc
qkr.LreLnc.cn/42428.Doc
qkr.LreLnc.cn/40282.Doc
qke.LreLnc.cn/04282.Doc
qke.LreLnc.cn/84428.Doc
qke.LreLnc.cn/06880.Doc
qke.LreLnc.cn/91575.Doc
qke.LreLnc.cn/53757.Doc
qke.LreLnc.cn/44828.Doc
qke.LreLnc.cn/26424.Doc
qke.LreLnc.cn/28822.Doc
qke.LreLnc.cn/42248.Doc
qke.LreLnc.cn/40424.Doc
qkw.LreLnc.cn/24626.Doc
qkw.LreLnc.cn/42226.Doc
qkw.LreLnc.cn/80640.Doc
qkw.LreLnc.cn/82860.Doc
qkw.LreLnc.cn/42626.Doc
qkw.LreLnc.cn/33195.Doc
qkw.LreLnc.cn/06606.Doc
qkw.LreLnc.cn/06420.Doc
qkw.LreLnc.cn/77195.Doc
qkw.LreLnc.cn/04420.Doc
qkq.LreLnc.cn/06660.Doc
qkq.LreLnc.cn/00628.Doc
qkq.LreLnc.cn/62442.Doc
qkq.LreLnc.cn/77937.Doc
qkq.LreLnc.cn/86402.Doc
qkq.LreLnc.cn/48224.Doc
qkq.LreLnc.cn/86082.Doc
qkq.LreLnc.cn/33115.Doc
qkq.LreLnc.cn/79515.Doc
qkq.LreLnc.cn/17737.Doc
qjm.LreLnc.cn/99355.Doc
qjm.LreLnc.cn/55313.Doc
qjm.LreLnc.cn/37135.Doc
qjm.LreLnc.cn/71115.Doc
qjm.LreLnc.cn/35371.Doc
qjm.LreLnc.cn/93731.Doc
qjm.LreLnc.cn/97373.Doc
qjm.LreLnc.cn/59197.Doc
qjm.LreLnc.cn/75755.Doc
qjm.LreLnc.cn/37135.Doc
qjn.LreLnc.cn/99719.Doc
qjn.LreLnc.cn/93517.Doc
qjn.LreLnc.cn/31773.Doc
qjn.LreLnc.cn/17515.Doc
qjn.LreLnc.cn/97799.Doc
qjn.LreLnc.cn/17311.Doc
qjn.LreLnc.cn/79337.Doc
qjn.LreLnc.cn/93317.Doc
qjn.LreLnc.cn/99979.Doc
qjn.LreLnc.cn/71337.Doc
qjb.LreLnc.cn/11377.Doc
qjb.LreLnc.cn/71137.Doc
qjb.LreLnc.cn/93559.Doc
qjb.LreLnc.cn/75531.Doc
qjb.LreLnc.cn/17513.Doc
qjb.LreLnc.cn/51751.Doc
qjb.LreLnc.cn/97779.Doc
qjb.LreLnc.cn/17777.Doc
qjb.LreLnc.cn/17153.Doc
qjb.LreLnc.cn/53577.Doc
qjv.LreLnc.cn/75133.Doc
qjv.LreLnc.cn/99953.Doc
qjv.LreLnc.cn/37937.Doc
qjv.LreLnc.cn/42480.Doc
qjv.LreLnc.cn/57917.Doc
qjv.LreLnc.cn/77999.Doc
qjv.LreLnc.cn/35133.Doc
qjv.LreLnc.cn/55551.Doc
qjv.LreLnc.cn/99359.Doc
qjv.LreLnc.cn/13577.Doc
qjc.LreLnc.cn/51571.Doc
qjc.LreLnc.cn/53111.Doc
qjc.LreLnc.cn/99551.Doc
qjc.LreLnc.cn/13395.Doc
qjc.LreLnc.cn/75735.Doc
qjc.LreLnc.cn/86022.Doc
qjc.LreLnc.cn/59999.Doc
qjc.LreLnc.cn/55139.Doc
qjc.LreLnc.cn/73533.Doc
qjc.LreLnc.cn/17719.Doc
qjx.LreLnc.cn/95531.Doc
qjx.LreLnc.cn/99775.Doc
qjx.LreLnc.cn/33777.Doc
qjx.LreLnc.cn/44282.Doc
qjx.LreLnc.cn/11137.Doc
qjx.LreLnc.cn/59351.Doc
qjx.LreLnc.cn/77531.Doc
qjx.LreLnc.cn/91971.Doc
qjx.LreLnc.cn/99993.Doc
qjx.LreLnc.cn/15791.Doc
qjz.LreLnc.cn/17553.Doc
qjz.LreLnc.cn/95395.Doc
qjz.LreLnc.cn/75731.Doc
qjz.LreLnc.cn/35795.Doc
qjz.LreLnc.cn/97135.Doc
qjz.LreLnc.cn/97115.Doc
qjz.LreLnc.cn/37317.Doc
qjz.LreLnc.cn/97995.Doc
qjz.LreLnc.cn/97355.Doc
qjz.LreLnc.cn/97513.Doc
qjl.LreLnc.cn/53773.Doc
qjl.LreLnc.cn/59171.Doc
qjl.LreLnc.cn/75575.Doc
qjl.LreLnc.cn/35797.Doc
qjl.LreLnc.cn/13131.Doc
qjl.LreLnc.cn/57171.Doc
qjl.LreLnc.cn/53339.Doc
qjl.LreLnc.cn/33559.Doc
qjl.LreLnc.cn/57757.Doc
qjl.LreLnc.cn/71531.Doc
qjk.LreLnc.cn/19711.Doc
qjk.LreLnc.cn/37131.Doc
qjk.LreLnc.cn/11937.Doc
qjk.LreLnc.cn/11711.Doc
qjk.LreLnc.cn/71393.Doc
qjk.LreLnc.cn/15931.Doc
qjk.LreLnc.cn/37355.Doc
qjk.LreLnc.cn/59533.Doc
qjk.LreLnc.cn/95557.Doc
qjk.LreLnc.cn/84482.Doc
qjj.LreLnc.cn/19737.Doc
qjj.LreLnc.cn/44420.Doc
qjj.LreLnc.cn/28202.Doc
qjj.LreLnc.cn/06086.Doc
qjj.LreLnc.cn/62066.Doc
qjj.LreLnc.cn/60882.Doc
qjj.LreLnc.cn/15773.Doc
qjj.LreLnc.cn/22808.Doc
qjj.LreLnc.cn/28200.Doc
qjj.LreLnc.cn/93333.Doc
qjh.LreLnc.cn/02806.Doc
qjh.LreLnc.cn/66062.Doc
qjh.LreLnc.cn/84284.Doc
qjh.LreLnc.cn/42684.Doc
qjh.LreLnc.cn/63475.Doc
qjh.LreLnc.cn/26864.Doc
qjh.LreLnc.cn/06624.Doc
qjh.LreLnc.cn/04226.Doc
qjh.LreLnc.cn/86862.Doc
qjh.LreLnc.cn/62664.Doc
qjg.LreLnc.cn/46402.Doc
qjg.LreLnc.cn/42228.Doc
qjg.LreLnc.cn/62884.Doc
qjg.LreLnc.cn/80806.Doc
qjg.LreLnc.cn/86888.Doc
qjg.LreLnc.cn/02084.Doc
qjg.LreLnc.cn/22888.Doc
qjg.LreLnc.cn/44604.Doc
qjg.LreLnc.cn/26000.Doc
qjg.LreLnc.cn/46884.Doc
qjf.LreLnc.cn/66604.Doc
qjf.LreLnc.cn/40204.Doc
qjf.LreLnc.cn/06642.Doc
qjf.LreLnc.cn/08244.Doc
qjf.LreLnc.cn/82460.Doc
qjf.LreLnc.cn/02242.Doc
qjf.LreLnc.cn/24420.Doc
qjf.LreLnc.cn/28664.Doc
qjf.LreLnc.cn/46806.Doc
qjf.LreLnc.cn/82024.Doc
qjd.LreLnc.cn/66040.Doc
qjd.LreLnc.cn/04624.Doc
qjd.LreLnc.cn/06024.Doc
qjd.LreLnc.cn/88822.Doc
qjd.LreLnc.cn/04264.Doc
qjd.LreLnc.cn/60242.Doc
qjd.LreLnc.cn/28248.Doc
qjd.LreLnc.cn/42442.Doc
qjd.LreLnc.cn/46882.Doc
qjd.LreLnc.cn/48402.Doc
