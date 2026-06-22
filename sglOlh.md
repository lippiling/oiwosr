缓俨敝诱扒


Python单元测试与Mock技术

一、unittest基础

import unittest

class Calculator:
    def add(self, a, b): return a + b
    def divide(self, a, b):
        if b == 0: raise ValueError("除数不能为零")
        return a / b

class TestCalc(unittest.TestCase):
    def setUp(self):
        self.calc = Calculator()
    def test_add(self):
        self.assertEqual(self.calc.add(2, 3), 5)
    def test_divide_by_zero(self):
        with self.assertRaises(ValueError):
            self.calc.divide(1, 0)


二、pytest框架

import pytest

def test_add():
    assert Calculator().add(2, 3) == 5

@pytest.fixture
def calc():
    return Calculator()

@pytest.mark.parametrize("a,b,expected", [
    (1, 1, 2), (0, 0, 0), (-1, 1, 0),
])
def test_add_param(calc, a, b, expected):
    assert calc.add(a, b) == expected


三、Mock基础

from unittest.mock import Mock, patch

mock_db = Mock()
mock_db.find_by_id.return_value = {'id': 1, 'name': 'Alice'}
service = UserService(mock_db)
user = service.get_user(1)
mock_db.find_by_id.assert_called_once_with(1)

# side_effect：多次调用不同返回值
mock_db.find_by_id.side_effect = [
    {'id': 1}, ValueError("DB error")
]


四、patch装饰器

@patch('requests.get')
def test_weather(mock_get):
    mock_response = Mock()
    mock_response.status_code = 200
    mock_response.json.return_value = {'temp': 25.5}
    mock_get.return_value = mock_response

    temp = WeatherService().get_temp('Beijing')
    assert temp == 25.5

# 上下文管理器形式
with patch('requests.get') as mock_get:
    mock_get.return_value.status_code = 500
    with pytest.raises(ConnectionError):
        WeatherService().get_temp('Beijing')


五、高级Mock

# spec：限制Mock只有原始类的方法
mock_db = Mock(spec=Database)

# PropertyMock
with patch.object(Config, 'debug', new_callable=PropertyMock, return_value=True):
    print(Config().debug)  # True

# AsyncMock
@patch('aiohttp.ClientSession.get')
async def test_async(mock_get):
    mock_get.return_value.__aenter__.return_value.json = AsyncMock(return_value={})
    result = await fetch()


六、测试替身

# Stub：返回预设值
stub_repo = Mock()
stub_repo.find_all.return_value = [{'id': 1}]

# Spy：记录调用但保留原始行为
with patch.object(Calculator, 'add', wraps=Calculator().add) as spy:
    Calculator().add(2, 3)
    spy.assert_called_once_with(2, 3)

# Fake：轻量级实现
class FakeDB:
    def __init__(self):
        self.data = {}
    def insert(self, record):
        record['id'] = len(self.data) + 1
        self.data[record['id']] = record
        return record


七、最佳实践

1. 每个测试只验证一个行为
2. 测试独立，不依赖执行顺序
3. 测试行为而非实现细节
4. Mock外部依赖，不Mock被测对象自身
5. 遵循AAA: Arrange-Act-Assert

泼谈靡坏徊姑狡勘判蛹纤笨厍遣固

fme.nfsid.cn/204666.Doc
fme.nfsid.cn/066866.Doc
fme.nfsid.cn/022424.Doc
fme.nfsid.cn/046442.Doc
fme.nfsid.cn/026881.Doc
fmw.nfsid.cn/648402.Doc
fmw.nfsid.cn/066688.Doc
fmw.nfsid.cn/668684.Doc
fmw.nfsid.cn/266482.Doc
fmw.nfsid.cn/593975.Doc
fmw.nfsid.cn/446444.Doc
fmw.nfsid.cn/682028.Doc
fmw.nfsid.cn/400002.Doc
fmw.nfsid.cn/251833.Doc
fmw.nfsid.cn/022442.Doc
fmq.nfsid.cn/200682.Doc
fmq.nfsid.cn/440826.Doc
fmq.nfsid.cn/044080.Doc
fmq.nfsid.cn/122296.Doc
fmq.nfsid.cn/422082.Doc
fmq.nfsid.cn/464844.Doc
fmq.nfsid.cn/402484.Doc
fmq.nfsid.cn/228626.Doc
fmq.nfsid.cn/461689.Doc
fmq.nfsid.cn/604082.Doc
fnm.nfsid.cn/002222.Doc
fnm.nfsid.cn/680264.Doc
fnm.nfsid.cn/204646.Doc
fnm.nfsid.cn/600226.Doc
fnm.nfsid.cn/162438.Doc
fnm.nfsid.cn/428064.Doc
fnm.nfsid.cn/222208.Doc
fnm.nfsid.cn/284222.Doc
fnm.nfsid.cn/084602.Doc
fnm.nfsid.cn/842026.Doc
fnn.nfsid.cn/912602.Doc
fnn.nfsid.cn/200628.Doc
fnn.nfsid.cn/246460.Doc
fnn.nfsid.cn/926081.Doc
fnn.nfsid.cn/713139.Doc
fnn.nfsid.cn/848880.Doc
fnn.nfsid.cn/264466.Doc
fnn.nfsid.cn/935991.Doc
fnn.nfsid.cn/533999.Doc
fnn.nfsid.cn/600882.Doc
fnb.nfsid.cn/684662.Doc
fnb.nfsid.cn/480468.Doc
fnb.nfsid.cn/240040.Doc
fnb.nfsid.cn/804502.Doc
fnb.nfsid.cn/800680.Doc
fnb.nfsid.cn/400044.Doc
fnb.nfsid.cn/266862.Doc
fnb.nfsid.cn/280442.Doc
fnb.nfsid.cn/246868.Doc
fnb.nfsid.cn/880282.Doc
fnv.nfsid.cn/406822.Doc
fnv.nfsid.cn/852106.Doc
fnv.nfsid.cn/371739.Doc
fnv.nfsid.cn/844862.Doc
fnv.nfsid.cn/606022.Doc
fnv.nfsid.cn/008284.Doc
fnv.nfsid.cn/204448.Doc
fnv.nfsid.cn/228644.Doc
fnv.nfsid.cn/668064.Doc
fnv.nfsid.cn/206800.Doc
fnc.nfsid.cn/721259.Doc
fnc.nfsid.cn/268806.Doc
fnc.nfsid.cn/824086.Doc
fnc.nfsid.cn/060224.Doc
fnc.nfsid.cn/880404.Doc
fnc.nfsid.cn/282802.Doc
fnc.nfsid.cn/264222.Doc
fnc.nfsid.cn/236632.Doc
fnc.nfsid.cn/024464.Doc
fnc.nfsid.cn/248428.Doc
fnx.nfsid.cn/651377.Doc
fnx.nfsid.cn/800802.Doc
fnx.nfsid.cn/286660.Doc
fnx.nfsid.cn/122588.Doc
fnx.nfsid.cn/282280.Doc
fnx.nfsid.cn/646664.Doc
fnx.nfsid.cn/648824.Doc
fnx.nfsid.cn/688468.Doc
fnx.nfsid.cn/428264.Doc
fnx.nfsid.cn/240826.Doc
fnz.nfsid.cn/402882.Doc
fnz.nfsid.cn/166582.Doc
fnz.nfsid.cn/006864.Doc
fnz.nfsid.cn/027941.Doc
fnz.nfsid.cn/466446.Doc
fnz.nfsid.cn/998235.Doc
fnz.nfsid.cn/962461.Doc
fnz.nfsid.cn/824824.Doc
fnz.nfsid.cn/284662.Doc
fnz.nfsid.cn/828648.Doc
fnl.nfsid.cn/609649.Doc
fnl.nfsid.cn/373579.Doc
fnl.nfsid.cn/484480.Doc
fnl.nfsid.cn/354401.Doc
fnl.nfsid.cn/862484.Doc
fnl.nfsid.cn/484220.Doc
fnl.nfsid.cn/642024.Doc
fnl.nfsid.cn/662806.Doc
fnl.nfsid.cn/993979.Doc
fnl.nfsid.cn/802886.Doc
fnk.nfsid.cn/620884.Doc
fnk.nfsid.cn/651685.Doc
fnk.nfsid.cn/080844.Doc
fnk.nfsid.cn/384374.Doc
fnk.nfsid.cn/385006.Doc
fnk.nfsid.cn/806082.Doc
fnk.nfsid.cn/648626.Doc
fnk.nfsid.cn/624084.Doc
fnk.nfsid.cn/422424.Doc
fnk.nfsid.cn/365572.Doc
fnj.nfsid.cn/482404.Doc
fnj.nfsid.cn/422624.Doc
fnj.nfsid.cn/820680.Doc
fnj.nfsid.cn/240260.Doc
fnj.nfsid.cn/620244.Doc
fnj.nfsid.cn/079318.Doc
fnj.nfsid.cn/737333.Doc
fnj.nfsid.cn/288466.Doc
fnj.nfsid.cn/202628.Doc
fnj.nfsid.cn/066022.Doc
fnh.nfsid.cn/464420.Doc
fnh.nfsid.cn/663781.Doc
fnh.nfsid.cn/602260.Doc
fnh.nfsid.cn/135715.Doc
fnh.nfsid.cn/886622.Doc
fnh.nfsid.cn/929273.Doc
fnh.nfsid.cn/840682.Doc
fnh.nfsid.cn/53.Doc
fnh.nfsid.cn/220028.Doc
fnh.nfsid.cn/640202.Doc
fng.nfsid.cn/842022.Doc
fng.nfsid.cn/404042.Doc
fng.nfsid.cn/533621.Doc
fng.nfsid.cn/959575.Doc
fng.nfsid.cn/822079.Doc
fng.nfsid.cn/040426.Doc
fng.nfsid.cn/555939.Doc
fng.nfsid.cn/826022.Doc
fng.nfsid.cn/044686.Doc
fng.nfsid.cn/200422.Doc
fnf.nfsid.cn/202040.Doc
fnf.nfsid.cn/042808.Doc
fnf.nfsid.cn/424208.Doc
fnf.nfsid.cn/886846.Doc
fnf.nfsid.cn/226624.Doc
fnf.nfsid.cn/200464.Doc
fnf.nfsid.cn/228822.Doc
fnf.nfsid.cn/062064.Doc
fnf.nfsid.cn/228424.Doc
fnf.nfsid.cn/426468.Doc
fnd.nfsid.cn/626882.Doc
fnd.nfsid.cn/420686.Doc
fnd.nfsid.cn/486222.Doc
fnd.nfsid.cn/480628.Doc
fnd.nfsid.cn/468242.Doc
fnd.nfsid.cn/343166.Doc
fnd.nfsid.cn/282200.Doc
fnd.nfsid.cn/862068.Doc
fnd.nfsid.cn/686084.Doc
fnd.nfsid.cn/442286.Doc
fns.nfsid.cn/602282.Doc
fns.nfsid.cn/604600.Doc
fns.nfsid.cn/044024.Doc
fns.nfsid.cn/042280.Doc
fns.nfsid.cn/064420.Doc
fns.nfsid.cn/088002.Doc
fns.nfsid.cn/930904.Doc
fns.nfsid.cn/044282.Doc
fns.nfsid.cn/204400.Doc
fns.nfsid.cn/042888.Doc
fna.nfsid.cn/688406.Doc
fna.nfsid.cn/157313.Doc
fna.nfsid.cn/660404.Doc
fna.nfsid.cn/648242.Doc
fna.nfsid.cn/511566.Doc
fna.nfsid.cn/927107.Doc
fna.nfsid.cn/350271.Doc
fna.nfsid.cn/268442.Doc
fna.nfsid.cn/646420.Doc
fna.nfsid.cn/602468.Doc
fnp.nfsid.cn/620040.Doc
fnp.nfsid.cn/886600.Doc
fnp.nfsid.cn/064604.Doc
fnp.nfsid.cn/841934.Doc
fnp.nfsid.cn/282200.Doc
fnp.nfsid.cn/484228.Doc
fnp.nfsid.cn/628088.Doc
fnp.nfsid.cn/002468.Doc
fnp.nfsid.cn/046460.Doc
fnp.nfsid.cn/220468.Doc
fno.nfsid.cn/068440.Doc
fno.nfsid.cn/860268.Doc
fno.nfsid.cn/335151.Doc
fno.nfsid.cn/268048.Doc
fno.nfsid.cn/446280.Doc
fno.nfsid.cn/880262.Doc
fno.nfsid.cn/331975.Doc
fno.nfsid.cn/848404.Doc
fno.nfsid.cn/488088.Doc
fno.nfsid.cn/600286.Doc
fni.nfsid.cn/006866.Doc
fni.nfsid.cn/508989.Doc
fni.nfsid.cn/806208.Doc
fni.nfsid.cn/824406.Doc
fni.nfsid.cn/062882.Doc
fni.nfsid.cn/309914.Doc
fni.nfsid.cn/884626.Doc
fni.nfsid.cn/822264.Doc
fni.nfsid.cn/962311.Doc
fni.nfsid.cn/286022.Doc
fnu.nfsid.cn/286880.Doc
fnu.nfsid.cn/351959.Doc
fnu.nfsid.cn/841961.Doc
fnu.nfsid.cn/464688.Doc
fnu.nfsid.cn/372236.Doc
fnu.nfsid.cn/824284.Doc
fnu.nfsid.cn/600802.Doc
fnu.nfsid.cn/604668.Doc
fnu.nfsid.cn/465911.Doc
fnu.nfsid.cn/652671.Doc
fny.nfsid.cn/480040.Doc
fny.nfsid.cn/406208.Doc
fny.nfsid.cn/808640.Doc
fny.nfsid.cn/591197.Doc
fny.nfsid.cn/282028.Doc
fny.nfsid.cn/866064.Doc
fny.nfsid.cn/644668.Doc
fny.nfsid.cn/022064.Doc
fny.nfsid.cn/086828.Doc
fny.nfsid.cn/666608.Doc
fnt.nfsid.cn/393607.Doc
fnt.nfsid.cn/268006.Doc
fnt.nfsid.cn/466462.Doc
fnt.nfsid.cn/354822.Doc
fnt.nfsid.cn/204262.Doc
fnt.nfsid.cn/179959.Doc
fnt.nfsid.cn/628842.Doc
fnt.nfsid.cn/559519.Doc
fnt.nfsid.cn/802644.Doc
fnt.nfsid.cn/408426.Doc
fnr.nfsid.cn/060220.Doc
fnr.nfsid.cn/595953.Doc
fnr.nfsid.cn/404020.Doc
fnr.nfsid.cn/919113.Doc
fnr.nfsid.cn/040268.Doc
fnr.nfsid.cn/688266.Doc
fnr.nfsid.cn/826282.Doc
fnr.nfsid.cn/628248.Doc
fnr.nfsid.cn/062644.Doc
fnr.nfsid.cn/602862.Doc
fne.nfsid.cn/224262.Doc
fne.nfsid.cn/531151.Doc
fne.nfsid.cn/602206.Doc
fne.nfsid.cn/448222.Doc
fne.nfsid.cn/846888.Doc
fne.nfsid.cn/802446.Doc
fne.nfsid.cn/842842.Doc
fne.nfsid.cn/668084.Doc
fne.nfsid.cn/202446.Doc
fne.nfsid.cn/804208.Doc
fnw.nfsid.cn/273590.Doc
fnw.nfsid.cn/244404.Doc
fnw.nfsid.cn/280822.Doc
fnw.nfsid.cn/487345.Doc
fnw.nfsid.cn/602880.Doc
fnw.nfsid.cn/246022.Doc
fnw.nfsid.cn/480886.Doc
fnw.nfsid.cn/622026.Doc
fnw.nfsid.cn/448488.Doc
fnw.nfsid.cn/288842.Doc
fnq.nfsid.cn/242640.Doc
fnq.nfsid.cn/486826.Doc
fnq.nfsid.cn/640202.Doc
fnq.nfsid.cn/004668.Doc
fnq.nfsid.cn/626084.Doc
fnq.nfsid.cn/064064.Doc
fnq.nfsid.cn/828628.Doc
fnq.nfsid.cn/000402.Doc
fnq.nfsid.cn/024600.Doc
fnq.nfsid.cn/664860.Doc
fbm.nfsid.cn/800004.Doc
fbm.nfsid.cn/224468.Doc
fbm.nfsid.cn/262040.Doc
fbm.nfsid.cn/888626.Doc
fbm.nfsid.cn/408446.Doc
fbm.nfsid.cn/800664.Doc
fbm.nfsid.cn/101529.Doc
fbm.nfsid.cn/042682.Doc
fbm.nfsid.cn/848446.Doc
fbm.nfsid.cn/280622.Doc
fbn.nfsid.cn/016722.Doc
fbn.nfsid.cn/608666.Doc
fbn.nfsid.cn/660426.Doc
fbn.nfsid.cn/448286.Doc
fbn.nfsid.cn/020282.Doc
fbn.nfsid.cn/860404.Doc
fbn.nfsid.cn/268064.Doc
fbn.nfsid.cn/803761.Doc
fbn.nfsid.cn/884200.Doc
fbn.nfsid.cn/535591.Doc
fbb.nfsid.cn/778549.Doc
fbb.nfsid.cn/608060.Doc
fbb.nfsid.cn/864103.Doc
fbb.nfsid.cn/222486.Doc
fbb.nfsid.cn/777119.Doc
fbb.nfsid.cn/844608.Doc
fbb.nfsid.cn/244260.Doc
fbb.nfsid.cn/280804.Doc
fbb.nfsid.cn/686644.Doc
fbb.nfsid.cn/608608.Doc
fbv.nfsid.cn/268460.Doc
fbv.nfsid.cn/880462.Doc
fbv.nfsid.cn/080068.Doc
fbv.nfsid.cn/466000.Doc
fbv.nfsid.cn/028284.Doc
fbv.nfsid.cn/684066.Doc
fbv.nfsid.cn/080260.Doc
fbv.nfsid.cn/717315.Doc
fbv.nfsid.cn/280602.Doc
fbv.nfsid.cn/864208.Doc
fbc.nfsid.cn/246604.Doc
fbc.nfsid.cn/026826.Doc
fbc.nfsid.cn/068086.Doc
fbc.nfsid.cn/711820.Doc
fbc.nfsid.cn/224608.Doc
fbc.nfsid.cn/008220.Doc
fbc.nfsid.cn/978522.Doc
fbc.nfsid.cn/740330.Doc
fbc.nfsid.cn/260426.Doc
fbc.nfsid.cn/226040.Doc
fbx.nfsid.cn/228040.Doc
fbx.nfsid.cn/995913.Doc
fbx.nfsid.cn/246404.Doc
fbx.nfsid.cn/848284.Doc
fbx.nfsid.cn/060224.Doc
fbx.nfsid.cn/008408.Doc
fbx.nfsid.cn/262200.Doc
fbx.nfsid.cn/648648.Doc
fbx.nfsid.cn/666026.Doc
fbx.nfsid.cn/086466.Doc
fbz.nfsid.cn/068044.Doc
fbz.nfsid.cn/440404.Doc
fbz.nfsid.cn/248824.Doc
fbz.nfsid.cn/640686.Doc
fbz.nfsid.cn/824880.Doc
fbz.nfsid.cn/256052.Doc
fbz.nfsid.cn/860268.Doc
fbz.nfsid.cn/604228.Doc
fbz.nfsid.cn/820004.Doc
fbz.nfsid.cn/028804.Doc
fbl.nfsid.cn/222068.Doc
fbl.nfsid.cn/377737.Doc
fbl.nfsid.cn/206044.Doc
fbl.nfsid.cn/846240.Doc
fbl.nfsid.cn/682022.Doc
fbl.nfsid.cn/848480.Doc
fbl.nfsid.cn/224428.Doc
fbl.nfsid.cn/325746.Doc
fbl.nfsid.cn/804020.Doc
fbl.nfsid.cn/969800.Doc
fbk.nfsid.cn/422068.Doc
fbk.nfsid.cn/246040.Doc
fbk.nfsid.cn/248088.Doc
fbk.nfsid.cn/362371.Doc
fbk.nfsid.cn/828442.Doc
fbk.nfsid.cn/466226.Doc
fbk.nfsid.cn/080426.Doc
fbk.nfsid.cn/646040.Doc
fbk.nfsid.cn/359937.Doc
fbk.nfsid.cn/644242.Doc
fbj.nfsid.cn/006224.Doc
fbj.nfsid.cn/440828.Doc
fbj.nfsid.cn/886048.Doc
fbj.nfsid.cn/988465.Doc
fbj.nfsid.cn/600426.Doc
fbj.nfsid.cn/620224.Doc
fbj.nfsid.cn/868060.Doc
fbj.nfsid.cn/040048.Doc
fbj.nfsid.cn/917796.Doc
fbj.nfsid.cn/709952.Doc
fbh.nfsid.cn/844420.Doc
fbh.nfsid.cn/846608.Doc
fbh.nfsid.cn/424064.Doc
fbh.nfsid.cn/737201.Doc
fbh.nfsid.cn/191135.Doc
fbh.nfsid.cn/400202.Doc
fbh.nfsid.cn/884826.Doc
fbh.nfsid.cn/557771.Doc
fbh.nfsid.cn/608460.Doc
fbh.nfsid.cn/280482.Doc
