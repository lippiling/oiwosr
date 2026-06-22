怨寥晕驶帜


Python日志系统完全指南

一、logging基础

import logging

logging.basicConfig(
    level=logging.DEBUG,
    format='%(asctime)s [%(levelname)s] %(name)s: %(message)s',
)

logger = logging.getLogger(__name__)

logger.debug("调试")
logger.info("信息")
logger.warning("警告")
logger.error("错误")
logger.critical("严重")


二、Logger/Handler/Formatter架构

def setup_logger(name, log_file=None, level=logging.INFO):
    logger = logging.getLogger(name)
    logger.setLevel(level)

    detailed = logging.Formatter(
        '%(asctime)s [%(levelname)s] %(name)s:%(lineno)d - %(message)s'
    )

    console = logging.StreamHandler()
    console.setFormatter(detailed)
    logger.addHandler(console)

    if log_file:
        from logging.handlers import RotatingFileHandler
        handler = RotatingFileHandler(
            log_file, maxBytes=10*1024*1024, backupCount=5
        )
        handler.setFormatter(detailed)
        logger.addHandler(handler)

    return logger


三、结构化日志

import json

class JSONFormatter(logging.Formatter):
    def format(self, record):
        data = {
            'timestamp': self.formatTime(record),
            'level': record.levelname,
            'message': record.getMessage(),
            'module': record.module,
            'line': record.lineno,
        }
        if record.exc_info:
            data['exception'] = self.formatException(record.exc_info)
        return json.dumps(data)

handler.setFormatter(JSONFormatter())


四、上下文日志

class ContextFilter(logging.Filter):
    def filter(self, record):
        record.request_id = getattr(threading.current_thread(), 'request_id', '-')
        return True

logger.addFilter(ContextFilter())


五、异常日志

try:
    result = risky_operation()
except ValueError:
    logger.error("处理失败", exc_info=True)
    # 或
    logger.exception("处理失败")  # 自动包含exc_info


六、dictConfig

import logging.config

LOGGING = {
    'version': 1,
    'disable_existing_loggers': False,
    'formatters': {
        'standard': {
            'format': '%(asctime)s [%(levelname)s] %(name)s: %(message)s'
        },
    },
    'handlers': {
        'console': {
            'class': 'logging.StreamHandler',
            'level': 'INFO',
            'formatter': 'standard',
        },
        'file': {
            'class': 'logging.handlers.RotatingFileHandler',
            'level': 'DEBUG',
            'filename': 'app.log',
            'maxBytes': 10485760,
            'backupCount': 5,
        },
    },
    'loggers': {
        '': {'handlers': ['console', 'file'], 'level': 'INFO'},
        'api': {'handlers': ['console'], 'level': 'DEBUG', 'propagate': False},
    },
}

logging.config.dictConfig(LOGGING)


七、自定义Handler

class DatabaseHandler(logging.Handler):
    def emit(self, record):
        try:
            entry = {
                'level': record.levelname,
                'message': record.getMessage(),
                'module': record.module,
            }
            self._write_to_db(entry)
        except Exception:
            self.handleError(record)


八、最佳实践

1. 每个模块用 getLogger(__name__) 获取logger
2. 库代码不配置logging，只在应用入口配置
3. 使用适当级别：DEBUG开发、INFO生产
4. 敏感信息不要记录日志
5. 生产环境使用文件轮转

搅踪籽突哦仆屡俾旁燎星乱莆醒肯

gqh.vwbnt.cn/044068.Doc
gqh.vwbnt.cn/066064.Doc
gqh.vwbnt.cn/422062.Doc
gqh.vwbnt.cn/824426.Doc
gqh.vwbnt.cn/606440.Doc
gqh.vwbnt.cn/082842.Doc
gqh.vwbnt.cn/848206.Doc
gqh.vwbnt.cn/335737.Doc
gqh.vwbnt.cn/822004.Doc
gqh.vwbnt.cn/756504.Doc
gqg.vwbnt.cn/640242.Doc
gqg.vwbnt.cn/233436.Doc
gqg.vwbnt.cn/624628.Doc
gqg.vwbnt.cn/644224.Doc
gqg.vwbnt.cn/626006.Doc
gqg.vwbnt.cn/440006.Doc
gqg.vwbnt.cn/408804.Doc
gqg.vwbnt.cn/600406.Doc
gqg.vwbnt.cn/222444.Doc
gqg.vwbnt.cn/066482.Doc
gqf.vwbnt.cn/686882.Doc
gqf.vwbnt.cn/268286.Doc
gqf.vwbnt.cn/008284.Doc
gqf.vwbnt.cn/028846.Doc
gqf.vwbnt.cn/288088.Doc
gqf.vwbnt.cn/240044.Doc
gqf.vwbnt.cn/064620.Doc
gqf.vwbnt.cn/608624.Doc
gqf.vwbnt.cn/185168.Doc
gqf.vwbnt.cn/660400.Doc
gqd.vwbnt.cn/048246.Doc
gqd.vwbnt.cn/892390.Doc
gqd.vwbnt.cn/555393.Doc
gqd.vwbnt.cn/316327.Doc
gqd.vwbnt.cn/444646.Doc
gqd.vwbnt.cn/446640.Doc
gqd.vwbnt.cn/222802.Doc
gqd.vwbnt.cn/266042.Doc
gqd.vwbnt.cn/284686.Doc
gqd.vwbnt.cn/680824.Doc
gqs.vwbnt.cn/820240.Doc
gqs.vwbnt.cn/717135.Doc
gqs.vwbnt.cn/048064.Doc
gqs.vwbnt.cn/333599.Doc
gqs.vwbnt.cn/694961.Doc
gqs.vwbnt.cn/008842.Doc
gqs.vwbnt.cn/486640.Doc
gqs.vwbnt.cn/604646.Doc
gqs.vwbnt.cn/228026.Doc
gqs.vwbnt.cn/880264.Doc
gqa.vwbnt.cn/113933.Doc
gqa.vwbnt.cn/842006.Doc
gqa.vwbnt.cn/028026.Doc
gqa.vwbnt.cn/208826.Doc
gqa.vwbnt.cn/604820.Doc
gqa.vwbnt.cn/886228.Doc
gqa.vwbnt.cn/808166.Doc
gqa.vwbnt.cn/002826.Doc
gqa.vwbnt.cn/668040.Doc
gqa.vwbnt.cn/286884.Doc
gqp.vwbnt.cn/662224.Doc
gqp.vwbnt.cn/915139.Doc
gqp.vwbnt.cn/226400.Doc
gqp.vwbnt.cn/074828.Doc
gqp.vwbnt.cn/848600.Doc
gqp.vwbnt.cn/076667.Doc
gqp.vwbnt.cn/260442.Doc
gqp.vwbnt.cn/688822.Doc
gqp.vwbnt.cn/242220.Doc
gqp.vwbnt.cn/044080.Doc
gqo.vwbnt.cn/202084.Doc
gqo.vwbnt.cn/597173.Doc
gqo.vwbnt.cn/204826.Doc
gqo.vwbnt.cn/062040.Doc
gqo.vwbnt.cn/462648.Doc
gqo.vwbnt.cn/937053.Doc
gqo.vwbnt.cn/224827.Doc
gqo.vwbnt.cn/484406.Doc
gqo.vwbnt.cn/608404.Doc
gqo.vwbnt.cn/781524.Doc
gqi.vwbnt.cn/680024.Doc
gqi.vwbnt.cn/992606.Doc
gqi.vwbnt.cn/939858.Doc
gqi.vwbnt.cn/200466.Doc
gqi.vwbnt.cn/484486.Doc
gqi.vwbnt.cn/220248.Doc
gqi.vwbnt.cn/008044.Doc
gqi.vwbnt.cn/840008.Doc
gqi.vwbnt.cn/808482.Doc
gqi.vwbnt.cn/421648.Doc
gqu.vwbnt.cn/208888.Doc
gqu.vwbnt.cn/604444.Doc
gqu.vwbnt.cn/888248.Doc
gqu.vwbnt.cn/682440.Doc
gqu.vwbnt.cn/840260.Doc
gqu.vwbnt.cn/696579.Doc
gqu.vwbnt.cn/608864.Doc
gqu.vwbnt.cn/420202.Doc
gqu.vwbnt.cn/660486.Doc
gqu.vwbnt.cn/260082.Doc
gqy.vwbnt.cn/840464.Doc
gqy.vwbnt.cn/864464.Doc
gqy.vwbnt.cn/818504.Doc
gqy.vwbnt.cn/240286.Doc
gqy.vwbnt.cn/624848.Doc
gqy.vwbnt.cn/084680.Doc
gqy.vwbnt.cn/698404.Doc
gqy.vwbnt.cn/048064.Doc
gqy.vwbnt.cn/117999.Doc
gqy.vwbnt.cn/828064.Doc
gqt.vwbnt.cn/024482.Doc
gqt.vwbnt.cn/886240.Doc
gqt.vwbnt.cn/420286.Doc
gqt.vwbnt.cn/662066.Doc
gqt.vwbnt.cn/468686.Doc
gqt.vwbnt.cn/484135.Doc
gqt.vwbnt.cn/884480.Doc
gqt.vwbnt.cn/064260.Doc
gqt.vwbnt.cn/082062.Doc
gqt.vwbnt.cn/828048.Doc
gqr.vwbnt.cn/648688.Doc
gqr.vwbnt.cn/288002.Doc
gqr.vwbnt.cn/288240.Doc
gqr.vwbnt.cn/600482.Doc
gqr.vwbnt.cn/428068.Doc
gqr.vwbnt.cn/886806.Doc
gqr.vwbnt.cn/224802.Doc
gqr.vwbnt.cn/400494.Doc
gqr.vwbnt.cn/608464.Doc
gqr.vwbnt.cn/026860.Doc
gqe.vwbnt.cn/466826.Doc
gqe.vwbnt.cn/616874.Doc
gqe.vwbnt.cn/082882.Doc
gqe.vwbnt.cn/204404.Doc
gqe.vwbnt.cn/266604.Doc
gqe.vwbnt.cn/642288.Doc
gqe.vwbnt.cn/844484.Doc
gqe.vwbnt.cn/030673.Doc
gqe.vwbnt.cn/428246.Doc
gqe.vwbnt.cn/068240.Doc
gqw.vwbnt.cn/729151.Doc
gqw.vwbnt.cn/336857.Doc
gqw.vwbnt.cn/022266.Doc
gqw.vwbnt.cn/511515.Doc
gqw.vwbnt.cn/841995.Doc
gqw.vwbnt.cn/286400.Doc
gqw.vwbnt.cn/202246.Doc
gqw.vwbnt.cn/040666.Doc
gqw.vwbnt.cn/668280.Doc
gqw.vwbnt.cn/682280.Doc
gqq.vwbnt.cn/626882.Doc
gqq.vwbnt.cn/082062.Doc
gqq.vwbnt.cn/086402.Doc
gqq.vwbnt.cn/828624.Doc
gqq.vwbnt.cn/402886.Doc
gqq.vwbnt.cn/060646.Doc
gqq.vwbnt.cn/044880.Doc
gqq.vwbnt.cn/406868.Doc
gqq.vwbnt.cn/246840.Doc
gqq.vwbnt.cn/153997.Doc
fmm.vwbnt.cn/444806.Doc
fmm.vwbnt.cn/484404.Doc
fmm.vwbnt.cn/064626.Doc
fmm.vwbnt.cn/425744.Doc
fmm.vwbnt.cn/622480.Doc
fmm.vwbnt.cn/373741.Doc
fmm.vwbnt.cn/822824.Doc
fmm.vwbnt.cn/400424.Doc
fmm.vwbnt.cn/242422.Doc
fmm.vwbnt.cn/486688.Doc
fmn.vwbnt.cn/199377.Doc
fmn.vwbnt.cn/866400.Doc
fmn.vwbnt.cn/866842.Doc
fmn.vwbnt.cn/264028.Doc
fmn.vwbnt.cn/668804.Doc
fmn.vwbnt.cn/646444.Doc
fmn.vwbnt.cn/717737.Doc
fmn.vwbnt.cn/746616.Doc
fmn.vwbnt.cn/515155.Doc
fmn.vwbnt.cn/228026.Doc
fmb.vwbnt.cn/133792.Doc
fmb.vwbnt.cn/870831.Doc
fmb.vwbnt.cn/333939.Doc
fmb.vwbnt.cn/208868.Doc
fmb.vwbnt.cn/749246.Doc
fmb.vwbnt.cn/301593.Doc
fmb.vwbnt.cn/448200.Doc
fmb.vwbnt.cn/222208.Doc
fmb.vwbnt.cn/084208.Doc
fmb.vwbnt.cn/846426.Doc
fmv.vwbnt.cn/345933.Doc
fmv.vwbnt.cn/428866.Doc
fmv.vwbnt.cn/022280.Doc
fmv.vwbnt.cn/208048.Doc
fmv.vwbnt.cn/284648.Doc
fmv.vwbnt.cn/262484.Doc
fmv.vwbnt.cn/228066.Doc
fmv.vwbnt.cn/620280.Doc
fmv.vwbnt.cn/840826.Doc
fmv.vwbnt.cn/240048.Doc
fmc.vwbnt.cn/482888.Doc
fmc.vwbnt.cn/448806.Doc
fmc.vwbnt.cn/224260.Doc
fmc.vwbnt.cn/286206.Doc
fmc.vwbnt.cn/244666.Doc
fmc.vwbnt.cn/886482.Doc
fmc.vwbnt.cn/836174.Doc
fmc.vwbnt.cn/649908.Doc
fmc.vwbnt.cn/882240.Doc
fmc.vwbnt.cn/204242.Doc
fmx.vwbnt.cn/448880.Doc
fmx.vwbnt.cn/660462.Doc
fmx.vwbnt.cn/404460.Doc
fmx.vwbnt.cn/420260.Doc
fmx.vwbnt.cn/068288.Doc
fmx.vwbnt.cn/442084.Doc
fmx.vwbnt.cn/391991.Doc
fmx.vwbnt.cn/688226.Doc
fmx.vwbnt.cn/404206.Doc
fmx.vwbnt.cn/862268.Doc
fmz.vwbnt.cn/202197.Doc
fmz.vwbnt.cn/600622.Doc
fmz.vwbnt.cn/804204.Doc
fmz.vwbnt.cn/204640.Doc
fmz.vwbnt.cn/206660.Doc
fmz.vwbnt.cn/422354.Doc
fmz.vwbnt.cn/282022.Doc
fmz.vwbnt.cn/026220.Doc
fmz.vwbnt.cn/480226.Doc
fmz.vwbnt.cn/024004.Doc
fml.vwbnt.cn/828840.Doc
fml.vwbnt.cn/735959.Doc
fml.vwbnt.cn/719317.Doc
fml.vwbnt.cn/066088.Doc
fml.vwbnt.cn/800460.Doc
fml.vwbnt.cn/282671.Doc
fml.vwbnt.cn/844002.Doc
fml.vwbnt.cn/420042.Doc
fml.vwbnt.cn/844644.Doc
fml.vwbnt.cn/082274.Doc
fmk.vwbnt.cn/921873.Doc
fmk.vwbnt.cn/060402.Doc
fmk.vwbnt.cn/240446.Doc
fmk.vwbnt.cn/113970.Doc
fmk.vwbnt.cn/822260.Doc
fmk.vwbnt.cn/028686.Doc
fmk.vwbnt.cn/046648.Doc
fmk.vwbnt.cn/600808.Doc
fmk.vwbnt.cn/467037.Doc
fmk.vwbnt.cn/228886.Doc
fmj.vwbnt.cn/064242.Doc
fmj.vwbnt.cn/319878.Doc
fmj.vwbnt.cn/692908.Doc
fmj.vwbnt.cn/466604.Doc
fmj.vwbnt.cn/468088.Doc
fmj.vwbnt.cn/662464.Doc
fmj.vwbnt.cn/680240.Doc
fmj.vwbnt.cn/824464.Doc
fmj.vwbnt.cn/248240.Doc
fmj.vwbnt.cn/956457.Doc
fmh.vwbnt.cn/804420.Doc
fmh.vwbnt.cn/282060.Doc
fmh.vwbnt.cn/266662.Doc
fmh.vwbnt.cn/557195.Doc
fmh.vwbnt.cn/026846.Doc
fmh.vwbnt.cn/153319.Doc
fmh.vwbnt.cn/482066.Doc
fmh.vwbnt.cn/620662.Doc
fmh.vwbnt.cn/240204.Doc
fmh.vwbnt.cn/860848.Doc
fmg.vwbnt.cn/826288.Doc
fmg.vwbnt.cn/883564.Doc
fmg.vwbnt.cn/208266.Doc
fmg.vwbnt.cn/507593.Doc
fmg.vwbnt.cn/266264.Doc
fmg.vwbnt.cn/314670.Doc
fmg.vwbnt.cn/862828.Doc
fmg.vwbnt.cn/939413.Doc
fmg.vwbnt.cn/844066.Doc
fmg.vwbnt.cn/282826.Doc
fmf.nfsid.cn/062884.Doc
fmf.nfsid.cn/193524.Doc
fmf.nfsid.cn/445937.Doc
fmf.nfsid.cn/462206.Doc
fmf.nfsid.cn/426668.Doc
fmf.nfsid.cn/795539.Doc
fmf.nfsid.cn/802400.Doc
fmf.nfsid.cn/800206.Doc
fmf.nfsid.cn/222620.Doc
fmf.nfsid.cn/442286.Doc
fmd.nfsid.cn/846024.Doc
fmd.nfsid.cn/022800.Doc
fmd.nfsid.cn/824046.Doc
fmd.nfsid.cn/682800.Doc
fmd.nfsid.cn/228222.Doc
fmd.nfsid.cn/842880.Doc
fmd.nfsid.cn/826444.Doc
fmd.nfsid.cn/064682.Doc
fmd.nfsid.cn/420844.Doc
fmd.nfsid.cn/440684.Doc
fms.nfsid.cn/684620.Doc
fms.nfsid.cn/806024.Doc
fms.nfsid.cn/006648.Doc
fms.nfsid.cn/484846.Doc
fms.nfsid.cn/220680.Doc
fms.nfsid.cn/688640.Doc
fms.nfsid.cn/382763.Doc
fms.nfsid.cn/822648.Doc
fms.nfsid.cn/626170.Doc
fms.nfsid.cn/606244.Doc
fma.nfsid.cn/400886.Doc
fma.nfsid.cn/082880.Doc
fma.nfsid.cn/684838.Doc
fma.nfsid.cn/864400.Doc
fma.nfsid.cn/208004.Doc
fma.nfsid.cn/753573.Doc
fma.nfsid.cn/355111.Doc
fma.nfsid.cn/660220.Doc
fma.nfsid.cn/828602.Doc
fma.nfsid.cn/688044.Doc
fmp.nfsid.cn/722673.Doc
fmp.nfsid.cn/260008.Doc
fmp.nfsid.cn/402604.Doc
fmp.nfsid.cn/064406.Doc
fmp.nfsid.cn/004442.Doc
fmp.nfsid.cn/426684.Doc
fmp.nfsid.cn/080046.Doc
fmp.nfsid.cn/862044.Doc
fmp.nfsid.cn/846884.Doc
fmp.nfsid.cn/688833.Doc
fmo.nfsid.cn/446802.Doc
fmo.nfsid.cn/068808.Doc
fmo.nfsid.cn/802286.Doc
fmo.nfsid.cn/020802.Doc
fmo.nfsid.cn/080842.Doc
fmo.nfsid.cn/915599.Doc
fmo.nfsid.cn/000666.Doc
fmo.nfsid.cn/004222.Doc
fmo.nfsid.cn/024828.Doc
fmo.nfsid.cn/846600.Doc
fmi.nfsid.cn/884842.Doc
fmi.nfsid.cn/919535.Doc
fmi.nfsid.cn/460808.Doc
fmi.nfsid.cn/860862.Doc
fmi.nfsid.cn/642600.Doc
fmi.nfsid.cn/010128.Doc
fmi.nfsid.cn/262680.Doc
fmi.nfsid.cn/442680.Doc
fmi.nfsid.cn/066000.Doc
fmi.nfsid.cn/375357.Doc
fmu.nfsid.cn/060022.Doc
fmu.nfsid.cn/028426.Doc
fmu.nfsid.cn/228208.Doc
fmu.nfsid.cn/864624.Doc
fmu.nfsid.cn/264662.Doc
fmu.nfsid.cn/660664.Doc
fmu.nfsid.cn/385242.Doc
fmu.nfsid.cn/256543.Doc
fmu.nfsid.cn/620280.Doc
fmu.nfsid.cn/406880.Doc
fmy.nfsid.cn/752967.Doc
fmy.nfsid.cn/084628.Doc
fmy.nfsid.cn/868466.Doc
fmy.nfsid.cn/284064.Doc
fmy.nfsid.cn/864088.Doc
fmy.nfsid.cn/822224.Doc
fmy.nfsid.cn/220884.Doc
fmy.nfsid.cn/806008.Doc
fmy.nfsid.cn/804062.Doc
fmy.nfsid.cn/373357.Doc
fmt.nfsid.cn/424088.Doc
fmt.nfsid.cn/064400.Doc
fmt.nfsid.cn/953440.Doc
fmt.nfsid.cn/557393.Doc
fmt.nfsid.cn/210926.Doc
fmt.nfsid.cn/444826.Doc
fmt.nfsid.cn/130348.Doc
fmt.nfsid.cn/931131.Doc
fmt.nfsid.cn/684404.Doc
fmt.nfsid.cn/977975.Doc
fmr.nfsid.cn/066222.Doc
fmr.nfsid.cn/802882.Doc
fmr.nfsid.cn/464264.Doc
fmr.nfsid.cn/828240.Doc
fmr.nfsid.cn/575337.Doc
fmr.nfsid.cn/682268.Doc
fmr.nfsid.cn/626402.Doc
fmr.nfsid.cn/080262.Doc
fmr.nfsid.cn/800868.Doc
fmr.nfsid.cn/688824.Doc
fme.nfsid.cn/533113.Doc
fme.nfsid.cn/202028.Doc
fme.nfsid.cn/442644.Doc
fme.nfsid.cn/608484.Doc
fme.nfsid.cn/880640.Doc
