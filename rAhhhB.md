Python并发编程：多线程与多进程

一、并发与并行

并发：多个任务在重叠时间段内交替执行。
并行：多个任务在同一时刻真正同时运行。

Python中多线程受GIL限制适合IO密集型，多进程适合CPU密集型。


二、threading基础

import threading, time

def download(url):
    print(f"下载 {url}")
    time.sleep(2)
    print(f"{url} 完成")

threads = [threading.Thread(target=download, args=(f"url-{i}",)) for i in range(5)]
for t in threads: t.start()
for t in threads: t.join()


三、线程同步

import threading

class Account:
    def __init__(self, balance):
        self.balance = balance
        self.lock = threading.Lock()

    def withdraw(self, amount):
        with self.lock:
            if self.balance >= amount:
                self.balance -= amount
                return True
            return False

其他同步工具：
- RLock：可重入锁
- Semaphore：信号量，限制并发数
- Event：事件通知
- Condition：条件变量


四、线程池

from concurrent.futures import ThreadPoolExecutor, as_completed

def fetch(url):
    return requests.get(url).status_code

urls = ['http://example.com'] * 10
with ThreadPoolExecutor(max_workers=5) as pool:
    results = list(pool.map(fetch, urls))


五、multiprocessing

from multiprocessing import Process, Queue, Pool

def cpu_work(n):
    return sum(i*i for i in range(n))

with Pool(processes=4) as pool:
    results = pool.map(cpu_work, [1000000] * 8)

进程间通信使用 Queue、Pipe、SharedMemory。


六、进程池

from concurrent.futures import ProcessPoolExecutor

def is_prime(n):
    if n < 2: return False
    for i in range(2, int(n**0.5)+1):
        if n % i == 0: return False
    return True

with ProcessPoolExecutor() as pool:
    results = pool.map(is_prime, range(100000, 101000))


七、选择建议

任务类型     推荐方案       原因
IO密集型     多线程/asyncio  GIL在IO时释放
CPU密集型    多进程          真正并行
混合型       多进程+线程池   各自处理对应任务

总结：IO密集型用线程，CPU密集型用进程，concurrent.futures 提供统一的高层接口。

esg.xkfrnc.cn/64282.Doc
esg.xkfrnc.cn/02448.Doc
esg.xkfrnc.cn/40884.Doc
esg.xkfrnc.cn/08406.Doc
esg.xkfrnc.cn/46462.Doc
esg.xkfrnc.cn/84862.Doc
esg.xkfrnc.cn/40084.Doc
esg.xkfrnc.cn/22646.Doc
esg.xkfrnc.cn/02686.Doc
esg.xkfrnc.cn/20668.Doc
esf.xkfrnc.cn/60828.Doc
esf.xkfrnc.cn/20648.Doc
esf.xkfrnc.cn/44600.Doc
esf.xkfrnc.cn/24824.Doc
esf.xkfrnc.cn/62002.Doc
esf.xkfrnc.cn/04664.Doc
esf.xkfrnc.cn/26440.Doc
esf.xkfrnc.cn/82248.Doc
esf.xkfrnc.cn/80462.Doc
esf.xkfrnc.cn/80466.Doc
esd.xkfrnc.cn/42420.Doc
esd.xkfrnc.cn/46626.Doc
esd.xkfrnc.cn/62662.Doc
esd.xkfrnc.cn/08664.Doc
esd.xkfrnc.cn/88804.Doc
esd.xkfrnc.cn/62648.Doc
esd.xkfrnc.cn/95401.Doc
esd.xkfrnc.cn/84844.Doc
esd.xkfrnc.cn/88044.Doc
esd.xkfrnc.cn/80882.Doc
ess.xkfrnc.cn/88024.Doc
ess.xkfrnc.cn/24062.Doc
ess.xkfrnc.cn/48884.Doc
ess.xkfrnc.cn/42486.Doc
ess.xkfrnc.cn/08620.Doc
ess.xkfrnc.cn/68628.Doc
ess.xkfrnc.cn/44824.Doc
ess.xkfrnc.cn/20008.Doc
ess.xkfrnc.cn/50956.Doc
ess.xkfrnc.cn/28048.Doc
esa.xkfrnc.cn/00660.Doc
esa.xkfrnc.cn/46048.Doc
esa.xkfrnc.cn/22000.Doc
esa.xkfrnc.cn/48428.Doc
esa.xkfrnc.cn/02448.Doc
esa.xkfrnc.cn/84446.Doc
esa.xkfrnc.cn/42422.Doc
esa.xkfrnc.cn/86642.Doc
esa.xkfrnc.cn/80202.Doc
esa.xkfrnc.cn/26662.Doc
esp.xkfrnc.cn/86402.Doc
esp.xkfrnc.cn/48688.Doc
esp.xkfrnc.cn/06626.Doc
esp.xkfrnc.cn/26095.Doc
esp.xkfrnc.cn/62933.Doc
esp.xkfrnc.cn/02800.Doc
esp.xkfrnc.cn/08600.Doc
esp.xkfrnc.cn/24040.Doc
esp.xkfrnc.cn/08488.Doc
esp.xkfrnc.cn/06284.Doc
eso.xkfrnc.cn/22460.Doc
eso.xkfrnc.cn/84806.Doc
eso.xkfrnc.cn/86684.Doc
eso.xkfrnc.cn/42680.Doc
eso.xkfrnc.cn/64422.Doc
eso.xkfrnc.cn/80020.Doc
eso.xkfrnc.cn/24866.Doc
eso.xkfrnc.cn/22228.Doc
eso.xkfrnc.cn/64800.Doc
eso.xkfrnc.cn/48406.Doc
esi.xkfrnc.cn/06444.Doc
esi.xkfrnc.cn/60608.Doc
esi.xkfrnc.cn/62242.Doc
esi.xkfrnc.cn/84046.Doc
esi.xkfrnc.cn/04842.Doc
esi.xkfrnc.cn/04066.Doc
esi.xkfrnc.cn/42684.Doc
esi.xkfrnc.cn/66486.Doc
esi.xkfrnc.cn/68480.Doc
esi.xkfrnc.cn/68648.Doc
esu.xkfrnc.cn/48042.Doc
esu.xkfrnc.cn/82484.Doc
esu.xkfrnc.cn/84848.Doc
esu.xkfrnc.cn/67212.Doc
esu.xkfrnc.cn/00668.Doc
esu.xkfrnc.cn/68242.Doc
esu.xkfrnc.cn/42200.Doc
esu.xkfrnc.cn/82048.Doc
esu.xkfrnc.cn/88064.Doc
esu.xkfrnc.cn/26066.Doc
esy.xkfrnc.cn/89012.Doc
esy.xkfrnc.cn/80808.Doc
esy.xkfrnc.cn/24226.Doc
esy.xkfrnc.cn/44420.Doc
esy.xkfrnc.cn/80262.Doc
esy.xkfrnc.cn/26848.Doc
esy.xkfrnc.cn/20802.Doc
esy.xkfrnc.cn/06660.Doc
esy.xkfrnc.cn/40404.Doc
esy.xkfrnc.cn/48240.Doc
est.xkfrnc.cn/42060.Doc
est.xkfrnc.cn/24480.Doc
est.xkfrnc.cn/42686.Doc
est.xkfrnc.cn/02208.Doc
est.xkfrnc.cn/62028.Doc
est.xkfrnc.cn/26442.Doc
est.xkfrnc.cn/64084.Doc
est.xkfrnc.cn/62466.Doc
est.xkfrnc.cn/15212.Doc
est.xkfrnc.cn/62628.Doc
esr.xkfrnc.cn/08226.Doc
esr.xkfrnc.cn/06248.Doc
esr.xkfrnc.cn/48871.Doc
esr.xkfrnc.cn/22664.Doc
esr.xkfrnc.cn/66608.Doc
esr.xkfrnc.cn/60862.Doc
esr.xkfrnc.cn/46086.Doc
esr.xkfrnc.cn/26206.Doc
esr.xkfrnc.cn/42640.Doc
esr.xkfrnc.cn/04228.Doc
ese.xkfrnc.cn/26000.Doc
ese.xkfrnc.cn/46488.Doc
ese.xkfrnc.cn/04080.Doc
ese.xkfrnc.cn/22848.Doc
ese.xkfrnc.cn/48024.Doc
ese.xkfrnc.cn/80826.Doc
ese.xkfrnc.cn/86462.Doc
ese.xkfrnc.cn/08400.Doc
ese.xkfrnc.cn/24862.Doc
ese.xkfrnc.cn/64028.Doc
esw.xkfrnc.cn/22266.Doc
esw.xkfrnc.cn/40600.Doc
esw.xkfrnc.cn/63484.Doc
esw.xkfrnc.cn/62402.Doc
esw.xkfrnc.cn/66244.Doc
esw.xkfrnc.cn/46608.Doc
esw.xkfrnc.cn/42480.Doc
esw.xkfrnc.cn/88406.Doc
esw.xkfrnc.cn/86048.Doc
esw.xkfrnc.cn/68068.Doc
esq.xkfrnc.cn/64488.Doc
esq.xkfrnc.cn/62604.Doc
esq.xkfrnc.cn/15793.Doc
esq.xkfrnc.cn/22082.Doc
esq.xkfrnc.cn/62084.Doc
esq.xkfrnc.cn/02862.Doc
esq.xkfrnc.cn/48000.Doc
esq.xkfrnc.cn/60846.Doc
esq.xkfrnc.cn/24202.Doc
esq.xkfrnc.cn/84488.Doc
eam.xkfrnc.cn/76062.Doc
eam.xkfrnc.cn/28600.Doc
eam.xkfrnc.cn/02202.Doc
eam.xkfrnc.cn/22684.Doc
eam.xkfrnc.cn/20408.Doc
eam.xkfrnc.cn/64224.Doc
eam.xkfrnc.cn/62408.Doc
eam.xkfrnc.cn/60804.Doc
eam.xkfrnc.cn/60844.Doc
eam.xkfrnc.cn/08200.Doc
ean.xkfrnc.cn/02868.Doc
ean.xkfrnc.cn/20404.Doc
ean.xkfrnc.cn/42888.Doc
ean.xkfrnc.cn/22123.Doc
ean.xkfrnc.cn/06446.Doc
ean.xkfrnc.cn/66026.Doc
ean.xkfrnc.cn/42644.Doc
ean.xkfrnc.cn/64426.Doc
ean.xkfrnc.cn/44680.Doc
ean.xkfrnc.cn/48846.Doc
eab.xkfrnc.cn/24624.Doc
eab.xkfrnc.cn/08448.Doc
eab.xkfrnc.cn/82420.Doc
eab.xkfrnc.cn/88268.Doc
eab.xkfrnc.cn/40680.Doc
eab.xkfrnc.cn/88888.Doc
eab.xkfrnc.cn/84208.Doc
eab.xkfrnc.cn/42226.Doc
eab.xkfrnc.cn/06488.Doc
eab.xkfrnc.cn/00626.Doc
eav.xkfrnc.cn/00400.Doc
eav.xkfrnc.cn/60022.Doc
eav.xkfrnc.cn/86424.Doc
eav.xkfrnc.cn/60408.Doc
eav.xkfrnc.cn/44080.Doc
eav.xkfrnc.cn/06022.Doc
eav.xkfrnc.cn/42666.Doc
eav.xkfrnc.cn/80866.Doc
eav.xkfrnc.cn/09173.Doc
eav.xkfrnc.cn/68640.Doc
eac.xkfrnc.cn/20882.Doc
eac.xkfrnc.cn/62680.Doc
eac.xkfrnc.cn/84808.Doc
eac.xkfrnc.cn/60844.Doc
eac.xkfrnc.cn/60260.Doc
eac.xkfrnc.cn/20426.Doc
eac.xkfrnc.cn/60622.Doc
eac.xkfrnc.cn/64082.Doc
eac.xkfrnc.cn/86684.Doc
eac.xkfrnc.cn/46442.Doc
eax.xkfrnc.cn/02620.Doc
eax.xkfrnc.cn/42660.Doc
eax.xkfrnc.cn/08400.Doc
eax.xkfrnc.cn/02848.Doc
eax.xkfrnc.cn/88060.Doc
eax.xkfrnc.cn/00084.Doc
eax.xkfrnc.cn/14873.Doc
eax.xkfrnc.cn/46244.Doc
eax.xkfrnc.cn/46864.Doc
eax.xkfrnc.cn/64066.Doc
eaz.xkfrnc.cn/24064.Doc
eaz.xkfrnc.cn/80820.Doc
eaz.xkfrnc.cn/40622.Doc
eaz.xkfrnc.cn/60204.Doc
eaz.xkfrnc.cn/08226.Doc
eaz.xkfrnc.cn/68400.Doc
eaz.xkfrnc.cn/80888.Doc
eaz.xkfrnc.cn/44620.Doc
eaz.xkfrnc.cn/08688.Doc
eaz.xkfrnc.cn/82266.Doc
eal.xkfrnc.cn/22204.Doc
eal.xkfrnc.cn/22046.Doc
eal.xkfrnc.cn/26224.Doc
eal.xkfrnc.cn/06862.Doc
eal.xkfrnc.cn/48802.Doc
eal.xkfrnc.cn/06846.Doc
eal.xkfrnc.cn/02802.Doc
eal.xkfrnc.cn/86260.Doc
eal.xkfrnc.cn/62642.Doc
eal.xkfrnc.cn/24064.Doc
eak.xkfrnc.cn/00220.Doc
eak.xkfrnc.cn/48228.Doc
eak.xkfrnc.cn/80688.Doc
eak.xkfrnc.cn/06835.Doc
eak.xkfrnc.cn/28002.Doc
eak.xkfrnc.cn/20220.Doc
eak.xkfrnc.cn/46644.Doc
eak.xkfrnc.cn/64040.Doc
eak.xkfrnc.cn/60242.Doc
eak.xkfrnc.cn/28682.Doc
eaj.xkfrnc.cn/22820.Doc
eaj.xkfrnc.cn/06062.Doc
eaj.xkfrnc.cn/06860.Doc
eaj.xkfrnc.cn/04220.Doc
eaj.xkfrnc.cn/24662.Doc
eaj.xkfrnc.cn/84026.Doc
eaj.xkfrnc.cn/46002.Doc
eaj.xkfrnc.cn/40024.Doc
eaj.xkfrnc.cn/68882.Doc
eaj.xkfrnc.cn/68240.Doc
eah.xkfrnc.cn/86824.Doc
eah.xkfrnc.cn/68266.Doc
eah.xkfrnc.cn/26802.Doc
eah.xkfrnc.cn/88020.Doc
eah.xkfrnc.cn/44202.Doc
eah.xkfrnc.cn/68648.Doc
eah.xkfrnc.cn/68048.Doc
eah.xkfrnc.cn/08464.Doc
eah.xkfrnc.cn/40404.Doc
eah.xkfrnc.cn/68660.Doc
eag.xkfrnc.cn/79395.Doc
eag.xkfrnc.cn/00402.Doc
eag.xkfrnc.cn/42668.Doc
eag.xkfrnc.cn/04426.Doc
eag.xkfrnc.cn/24062.Doc
eag.xkfrnc.cn/64248.Doc
eag.xkfrnc.cn/82208.Doc
eag.xkfrnc.cn/86446.Doc
eag.xkfrnc.cn/00848.Doc
eag.xkfrnc.cn/04486.Doc
eaf.xkfrnc.cn/04006.Doc
eaf.xkfrnc.cn/08480.Doc
eaf.xkfrnc.cn/24220.Doc
eaf.xkfrnc.cn/64480.Doc
eaf.xkfrnc.cn/02660.Doc
eaf.xkfrnc.cn/88004.Doc
eaf.xkfrnc.cn/60620.Doc
eaf.xkfrnc.cn/91828.Doc
eaf.xkfrnc.cn/44844.Doc
eaf.xkfrnc.cn/48804.Doc
ead.xkfrnc.cn/64260.Doc
ead.xkfrnc.cn/01506.Doc
ead.xkfrnc.cn/42482.Doc
ead.xkfrnc.cn/80680.Doc
ead.xkfrnc.cn/86468.Doc
ead.xkfrnc.cn/22488.Doc
ead.xkfrnc.cn/06400.Doc
ead.xkfrnc.cn/22622.Doc
ead.xkfrnc.cn/20206.Doc
ead.xkfrnc.cn/46066.Doc
eas.xkfrnc.cn/30560.Doc
eas.xkfrnc.cn/68512.Doc
eas.xkfrnc.cn/62848.Doc
eas.xkfrnc.cn/24880.Doc
eas.xkfrnc.cn/26644.Doc
eas.xkfrnc.cn/06888.Doc
eas.xkfrnc.cn/82868.Doc
eas.xkfrnc.cn/64468.Doc
eas.xkfrnc.cn/88046.Doc
eas.xkfrnc.cn/46895.Doc
eaa.xkfrnc.cn/46664.Doc
eaa.xkfrnc.cn/24686.Doc
eaa.xkfrnc.cn/20862.Doc
eaa.xkfrnc.cn/60068.Doc
eaa.xkfrnc.cn/68820.Doc
eaa.xkfrnc.cn/28040.Doc
eaa.xkfrnc.cn/24673.Doc
eaa.xkfrnc.cn/26200.Doc
eaa.xkfrnc.cn/62242.Doc
eaa.xkfrnc.cn/99073.Doc
eap.xkfrnc.cn/88068.Doc
eap.xkfrnc.cn/64284.Doc
eap.xkfrnc.cn/28042.Doc
eap.xkfrnc.cn/44606.Doc
eap.xkfrnc.cn/46448.Doc
eap.xkfrnc.cn/66084.Doc
eap.xkfrnc.cn/68022.Doc
eap.xkfrnc.cn/64822.Doc
eap.xkfrnc.cn/24480.Doc
eap.xkfrnc.cn/64488.Doc
eao.xkfrnc.cn/46484.Doc
eao.xkfrnc.cn/20046.Doc
eao.xkfrnc.cn/40066.Doc
eao.xkfrnc.cn/80680.Doc
eao.xkfrnc.cn/82222.Doc
eao.xkfrnc.cn/60084.Doc
eao.xkfrnc.cn/46800.Doc
eao.xkfrnc.cn/64622.Doc
eao.xkfrnc.cn/04442.Doc
eao.xkfrnc.cn/86860.Doc
eai.xkfrnc.cn/22806.Doc
eai.xkfrnc.cn/40420.Doc
eai.xkfrnc.cn/04880.Doc
eai.xkfrnc.cn/68628.Doc
eai.xkfrnc.cn/04442.Doc
eai.xkfrnc.cn/62646.Doc
eai.xkfrnc.cn/66828.Doc
eai.xkfrnc.cn/00668.Doc
eai.xkfrnc.cn/48644.Doc
eai.xkfrnc.cn/88420.Doc
eau.xkfrnc.cn/82202.Doc
eau.xkfrnc.cn/86460.Doc
eau.xkfrnc.cn/84604.Doc
eau.xkfrnc.cn/88288.Doc
eau.xkfrnc.cn/48268.Doc
eau.xkfrnc.cn/24086.Doc
eau.xkfrnc.cn/22044.Doc
eau.xkfrnc.cn/04248.Doc
eau.xkfrnc.cn/46240.Doc
eau.xkfrnc.cn/08402.Doc
eay.xkfrnc.cn/40006.Doc
eay.xkfrnc.cn/80408.Doc
eay.xkfrnc.cn/04684.Doc
eay.xkfrnc.cn/00840.Doc
eay.xkfrnc.cn/68868.Doc
eay.xkfrnc.cn/88220.Doc
eay.xkfrnc.cn/62260.Doc
eay.xkfrnc.cn/86824.Doc
eay.xkfrnc.cn/02262.Doc
eay.xkfrnc.cn/68480.Doc
eat.xkfrnc.cn/02688.Doc
eat.xkfrnc.cn/80422.Doc
eat.xkfrnc.cn/44028.Doc
eat.xkfrnc.cn/86860.Doc
eat.xkfrnc.cn/62600.Doc
eat.xkfrnc.cn/20402.Doc
eat.xkfrnc.cn/80602.Doc
eat.xkfrnc.cn/86826.Doc
eat.xkfrnc.cn/22066.Doc
eat.xkfrnc.cn/04844.Doc
ear.xkfrnc.cn/60084.Doc
ear.xkfrnc.cn/48204.Doc
ear.xkfrnc.cn/62662.Doc
ear.xkfrnc.cn/22400.Doc
ear.xkfrnc.cn/26286.Doc
ear.xkfrnc.cn/04426.Doc
ear.xkfrnc.cn/00408.Doc
ear.xkfrnc.cn/48026.Doc
ear.xkfrnc.cn/40824.Doc
ear.xkfrnc.cn/62646.Doc
eae.xkfrnc.cn/66480.Doc
eae.xkfrnc.cn/64242.Doc
eae.xkfrnc.cn/60400.Doc
eae.xkfrnc.cn/68206.Doc
eae.xkfrnc.cn/80482.Doc
eae.xkfrnc.cn/68402.Doc
eae.xkfrnc.cn/04084.Doc
eae.xkfrnc.cn/80666.Doc
eae.xkfrnc.cn/66206.Doc
eae.xkfrnc.cn/82240.Doc
eaw.xkfrnc.cn/46646.Doc
eaw.xkfrnc.cn/84240.Doc
eaw.xkfrnc.cn/40002.Doc
eaw.xkfrnc.cn/48222.Doc
eaw.xkfrnc.cn/88028.Doc
eaw.xkfrnc.cn/82480.Doc
eaw.xkfrnc.cn/04060.Doc
eaw.xkfrnc.cn/48068.Doc
eaw.xkfrnc.cn/28204.Doc
eaw.xkfrnc.cn/46828.Doc
eaq.xkfrnc.cn/66848.Doc
eaq.xkfrnc.cn/28020.Doc
eaq.xkfrnc.cn/84200.Doc
eaq.xkfrnc.cn/26602.Doc
eaq.xkfrnc.cn/08422.Doc
eaq.xkfrnc.cn/08882.Doc
eaq.xkfrnc.cn/62642.Doc
eaq.xkfrnc.cn/20042.Doc
eaq.xkfrnc.cn/22422.Doc
eaq.xkfrnc.cn/28828.Doc
epm.xkfrnc.cn/88208.Doc
epm.xkfrnc.cn/60220.Doc
epm.xkfrnc.cn/66864.Doc
epm.xkfrnc.cn/68886.Doc
epm.xkfrnc.cn/24840.Doc
epm.xkfrnc.cn/64682.Doc
epm.xkfrnc.cn/66400.Doc
epm.xkfrnc.cn/88266.Doc
epm.xkfrnc.cn/66262.Doc
epm.xkfrnc.cn/60662.Doc
epn.xkfrnc.cn/66060.Doc
epn.xkfrnc.cn/62400.Doc
epn.xkfrnc.cn/80062.Doc
epn.xkfrnc.cn/28864.Doc
epn.xkfrnc.cn/02080.Doc
epn.xkfrnc.cn/04628.Doc
epn.xkfrnc.cn/42060.Doc
epn.xkfrnc.cn/40006.Doc
epn.xkfrnc.cn/66448.Doc
epn.xkfrnc.cn/60286.Doc
epb.xkfrnc.cn/64426.Doc
epb.xkfrnc.cn/26202.Doc
epb.xkfrnc.cn/42482.Doc
epb.xkfrnc.cn/04024.Doc
epb.xkfrnc.cn/26862.Doc
epb.xkfrnc.cn/66842.Doc
epb.xkfrnc.cn/66040.Doc
epb.xkfrnc.cn/80626.Doc
epb.xkfrnc.cn/82044.Doc
epb.xkfrnc.cn/82244.Doc
epv.xkfrnc.cn/20226.Doc
epv.xkfrnc.cn/84642.Doc
epv.xkfrnc.cn/42040.Doc
epv.xkfrnc.cn/60462.Doc
epv.xkfrnc.cn/88426.Doc
epv.xkfrnc.cn/46242.Doc
epv.xkfrnc.cn/24088.Doc
epv.xkfrnc.cn/82440.Doc
epv.xkfrnc.cn/88220.Doc
epv.xkfrnc.cn/00086.Doc
epc.xkfrnc.cn/04082.Doc
epc.xkfrnc.cn/26000.Doc
epc.xkfrnc.cn/02420.Doc
epc.xkfrnc.cn/64202.Doc
epc.xkfrnc.cn/60460.Doc
epc.xkfrnc.cn/08862.Doc
epc.xkfrnc.cn/84646.Doc
epc.xkfrnc.cn/64686.Doc
epc.xkfrnc.cn/04286.Doc
epc.xkfrnc.cn/28866.Doc
epx.xkfrnc.cn/60442.Doc
epx.xkfrnc.cn/66288.Doc
epx.xkfrnc.cn/44806.Doc
epx.xkfrnc.cn/44880.Doc
epx.xkfrnc.cn/24848.Doc
epx.xkfrnc.cn/84806.Doc
epx.xkfrnc.cn/22484.Doc
epx.xkfrnc.cn/26600.Doc
epx.xkfrnc.cn/42448.Doc
epx.xkfrnc.cn/22260.Doc
epz.xkfrnc.cn/68686.Doc
epz.xkfrnc.cn/26004.Doc
epz.xkfrnc.cn/48284.Doc
epz.xkfrnc.cn/42668.Doc
epz.xkfrnc.cn/26482.Doc
epz.xkfrnc.cn/46684.Doc
epz.xkfrnc.cn/80022.Doc
epz.xkfrnc.cn/62480.Doc
epz.xkfrnc.cn/88060.Doc
epz.xkfrnc.cn/28804.Doc
epl.xkfrnc.cn/04044.Doc
epl.xkfrnc.cn/00602.Doc
epl.xkfrnc.cn/02684.Doc
epl.xkfrnc.cn/40408.Doc
epl.xkfrnc.cn/66044.Doc
epl.xkfrnc.cn/46202.Doc
epl.xkfrnc.cn/02200.Doc
epl.xkfrnc.cn/60206.Doc
epl.xkfrnc.cn/20228.Doc
epl.xkfrnc.cn/46228.Doc
