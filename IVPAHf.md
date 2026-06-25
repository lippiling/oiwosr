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

ref.by1792.cn/06428.Doc
ref.by1792.cn/20868.Doc
ref.by1792.cn/97939.Doc
ref.by1792.cn/24828.Doc
ref.by1792.cn/84688.Doc
ref.by1792.cn/40228.Doc
ref.by1792.cn/80488.Doc
ref.by1792.cn/48688.Doc
ref.by1792.cn/24828.Doc
ref.by1792.cn/46084.Doc
red.by1792.cn/02808.Doc
red.by1792.cn/84406.Doc
red.by1792.cn/28444.Doc
red.by1792.cn/22060.Doc
red.by1792.cn/44080.Doc
red.by1792.cn/04404.Doc
red.by1792.cn/40242.Doc
red.by1792.cn/42040.Doc
red.by1792.cn/33997.Doc
red.by1792.cn/31751.Doc
res.by1792.cn/22208.Doc
res.by1792.cn/08088.Doc
res.by1792.cn/44840.Doc
res.by1792.cn/70448.Doc
res.by1792.cn/04644.Doc
res.by1792.cn/19557.Doc
res.by1792.cn/48844.Doc
res.by1792.cn/77135.Doc
res.by1792.cn/22468.Doc
res.by1792.cn/17737.Doc
rea.by1792.cn/46406.Doc
rea.by1792.cn/06684.Doc
rea.by1792.cn/04002.Doc
rea.by1792.cn/68624.Doc
rea.by1792.cn/26400.Doc
rea.by1792.cn/42660.Doc
rea.by1792.cn/08244.Doc
rea.by1792.cn/20062.Doc
rea.by1792.cn/84826.Doc
rea.by1792.cn/08026.Doc
rep.by1792.cn/80860.Doc
rep.by1792.cn/48264.Doc
rep.by1792.cn/88648.Doc
rep.by1792.cn/02064.Doc
rep.by1792.cn/48408.Doc
rep.by1792.cn/28204.Doc
rep.by1792.cn/64008.Doc
rep.by1792.cn/20226.Doc
rep.by1792.cn/66220.Doc
rep.by1792.cn/64446.Doc
reo.by1792.cn/06662.Doc
reo.by1792.cn/28288.Doc
reo.by1792.cn/84822.Doc
reo.by1792.cn/48082.Doc
reo.by1792.cn/62266.Doc
reo.by1792.cn/60486.Doc
reo.by1792.cn/46062.Doc
reo.by1792.cn/88442.Doc
reo.by1792.cn/08240.Doc
reo.by1792.cn/20686.Doc
rei.by1792.cn/66864.Doc
rei.by1792.cn/82624.Doc
rei.by1792.cn/88268.Doc
rei.by1792.cn/68204.Doc
rei.by1792.cn/31791.Doc
rei.by1792.cn/42624.Doc
rei.by1792.cn/28488.Doc
rei.by1792.cn/71955.Doc
rei.by1792.cn/02600.Doc
rei.by1792.cn/66206.Doc
reu.by1792.cn/46642.Doc
reu.by1792.cn/24288.Doc
reu.by1792.cn/28224.Doc
reu.by1792.cn/62208.Doc
reu.by1792.cn/68406.Doc
reu.by1792.cn/44202.Doc
reu.by1792.cn/02460.Doc
reu.by1792.cn/00686.Doc
reu.by1792.cn/82080.Doc
reu.by1792.cn/06682.Doc
rey.by1792.cn/60066.Doc
rey.by1792.cn/44442.Doc
rey.by1792.cn/64428.Doc
rey.by1792.cn/02004.Doc
rey.by1792.cn/60802.Doc
rey.by1792.cn/60448.Doc
rey.by1792.cn/80824.Doc
rey.by1792.cn/44028.Doc
rey.by1792.cn/60286.Doc
rey.by1792.cn/44404.Doc
ret.by1792.cn/42282.Doc
ret.by1792.cn/66626.Doc
ret.by1792.cn/04408.Doc
ret.by1792.cn/13555.Doc
ret.by1792.cn/80880.Doc
ret.by1792.cn/68002.Doc
ret.by1792.cn/82264.Doc
ret.by1792.cn/04644.Doc
ret.by1792.cn/42488.Doc
ret.by1792.cn/84266.Doc
rer.by1792.cn/93715.Doc
rer.by1792.cn/84626.Doc
rer.by1792.cn/26466.Doc
rer.by1792.cn/08628.Doc
rer.by1792.cn/66066.Doc
rer.by1792.cn/88204.Doc
rer.by1792.cn/08246.Doc
rer.by1792.cn/86064.Doc
rer.by1792.cn/64866.Doc
rer.by1792.cn/80406.Doc
ree.by1792.cn/95999.Doc
ree.by1792.cn/08800.Doc
ree.by1792.cn/84826.Doc
ree.by1792.cn/22446.Doc
ree.by1792.cn/44808.Doc
ree.by1792.cn/86624.Doc
ree.by1792.cn/44208.Doc
ree.by1792.cn/82202.Doc
ree.by1792.cn/20228.Doc
ree.by1792.cn/02482.Doc
rew.by1792.cn/20426.Doc
rew.by1792.cn/68686.Doc
rew.by1792.cn/40864.Doc
rew.by1792.cn/68628.Doc
rew.by1792.cn/40288.Doc
rew.by1792.cn/02266.Doc
rew.by1792.cn/60260.Doc
rew.by1792.cn/26622.Doc
rew.by1792.cn/88686.Doc
rew.by1792.cn/02622.Doc
req.by1792.cn/15973.Doc
req.by1792.cn/22666.Doc
req.by1792.cn/59553.Doc
req.by1792.cn/64260.Doc
req.by1792.cn/62468.Doc
req.by1792.cn/80464.Doc
req.by1792.cn/84024.Doc
req.by1792.cn/86622.Doc
req.by1792.cn/68602.Doc
req.by1792.cn/64820.Doc
rwm.by1792.cn/06064.Doc
rwm.by1792.cn/55791.Doc
rwm.by1792.cn/86600.Doc
rwm.by1792.cn/48804.Doc
rwm.by1792.cn/35597.Doc
rwm.by1792.cn/20826.Doc
rwm.by1792.cn/82026.Doc
rwm.by1792.cn/62280.Doc
rwm.by1792.cn/62460.Doc
rwm.by1792.cn/20608.Doc
rwn.by1792.cn/66624.Doc
rwn.by1792.cn/46280.Doc
rwn.by1792.cn/95771.Doc
rwn.by1792.cn/02088.Doc
rwn.by1792.cn/82040.Doc
rwn.by1792.cn/73173.Doc
rwn.by1792.cn/46284.Doc
rwn.by1792.cn/26802.Doc
rwn.by1792.cn/17551.Doc
rwn.by1792.cn/44466.Doc
rwb.by1792.cn/64808.Doc
rwb.by1792.cn/64604.Doc
rwb.by1792.cn/62862.Doc
rwb.by1792.cn/08862.Doc
rwb.by1792.cn/06446.Doc
rwb.by1792.cn/06202.Doc
rwb.by1792.cn/82802.Doc
rwb.by1792.cn/62802.Doc
rwb.by1792.cn/22608.Doc
rwb.by1792.cn/44222.Doc
rwv.by1792.cn/57339.Doc
rwv.by1792.cn/66820.Doc
rwv.by1792.cn/28488.Doc
rwv.by1792.cn/88802.Doc
rwv.by1792.cn/42020.Doc
rwv.by1792.cn/22602.Doc
rwv.by1792.cn/86222.Doc
rwv.by1792.cn/53319.Doc
rwv.by1792.cn/13311.Doc
rwv.by1792.cn/88660.Doc
rwc.by1792.cn/86460.Doc
rwc.by1792.cn/04062.Doc
rwc.by1792.cn/24062.Doc
rwc.by1792.cn/66086.Doc
rwc.by1792.cn/06264.Doc
rwc.by1792.cn/77951.Doc
rwc.by1792.cn/24688.Doc
rwc.by1792.cn/66682.Doc
rwc.by1792.cn/40246.Doc
rwc.by1792.cn/66042.Doc
rwx.by1792.cn/08840.Doc
rwx.by1792.cn/06280.Doc
rwx.by1792.cn/86442.Doc
rwx.by1792.cn/86480.Doc
rwx.by1792.cn/42046.Doc
rwx.by1792.cn/04280.Doc
rwx.by1792.cn/40824.Doc
rwx.by1792.cn/00688.Doc
rwx.by1792.cn/44000.Doc
rwx.by1792.cn/40600.Doc
rwz.by1792.cn/60040.Doc
rwz.by1792.cn/68006.Doc
rwz.by1792.cn/39719.Doc
rwz.by1792.cn/44442.Doc
rwz.by1792.cn/04208.Doc
rwz.by1792.cn/60062.Doc
rwz.by1792.cn/24462.Doc
rwz.by1792.cn/88024.Doc
rwz.by1792.cn/24608.Doc
rwz.by1792.cn/48408.Doc
rwl.by1792.cn/73935.Doc
rwl.by1792.cn/02806.Doc
rwl.by1792.cn/04664.Doc
rwl.by1792.cn/42468.Doc
rwl.by1792.cn/04460.Doc
rwl.by1792.cn/48486.Doc
rwl.by1792.cn/64040.Doc
rwl.by1792.cn/64206.Doc
rwl.by1792.cn/28204.Doc
rwl.by1792.cn/91957.Doc
rwk.by1792.cn/40202.Doc
rwk.by1792.cn/04084.Doc
rwk.by1792.cn/48628.Doc
rwk.by1792.cn/62406.Doc
rwk.by1792.cn/40640.Doc
rwk.by1792.cn/64644.Doc
rwk.by1792.cn/42842.Doc
rwk.by1792.cn/19739.Doc
rwk.by1792.cn/33931.Doc
rwk.by1792.cn/84068.Doc
rwj.by1792.cn/68004.Doc
rwj.by1792.cn/60228.Doc
rwj.by1792.cn/48884.Doc
rwj.by1792.cn/19951.Doc
rwj.by1792.cn/80880.Doc
rwj.by1792.cn/20464.Doc
rwj.by1792.cn/08004.Doc
rwj.by1792.cn/51553.Doc
rwj.by1792.cn/44468.Doc
rwj.by1792.cn/84824.Doc
rwh.by1792.cn/42886.Doc
rwh.by1792.cn/08024.Doc
rwh.by1792.cn/80860.Doc
rwh.by1792.cn/24884.Doc
rwh.by1792.cn/02620.Doc
rwh.by1792.cn/08484.Doc
rwh.by1792.cn/02008.Doc
rwh.by1792.cn/26082.Doc
rwh.by1792.cn/84684.Doc
rwh.by1792.cn/26842.Doc
rwg.by1792.cn/08240.Doc
rwg.by1792.cn/86260.Doc
rwg.by1792.cn/44460.Doc
rwg.by1792.cn/06628.Doc
rwg.by1792.cn/77955.Doc
rwg.by1792.cn/82260.Doc
rwg.by1792.cn/48462.Doc
rwg.by1792.cn/28440.Doc
rwg.by1792.cn/82666.Doc
rwg.by1792.cn/28820.Doc
rwf.by1792.cn/75379.Doc
rwf.by1792.cn/42684.Doc
rwf.by1792.cn/22824.Doc
rwf.by1792.cn/02284.Doc
rwf.by1792.cn/22002.Doc
rwf.by1792.cn/48622.Doc
rwf.by1792.cn/60220.Doc
rwf.by1792.cn/00248.Doc
rwf.by1792.cn/22808.Doc
rwf.by1792.cn/20022.Doc
rwd.by1792.cn/64226.Doc
rwd.by1792.cn/64862.Doc
rwd.by1792.cn/95711.Doc
rwd.by1792.cn/62466.Doc
rwd.by1792.cn/79117.Doc
rwd.by1792.cn/48064.Doc
rwd.by1792.cn/22242.Doc
rwd.by1792.cn/00046.Doc
rwd.by1792.cn/80288.Doc
rwd.by1792.cn/84048.Doc
rws.by1792.cn/62240.Doc
rws.by1792.cn/40248.Doc
rws.by1792.cn/62228.Doc
rws.by1792.cn/60400.Doc
rws.by1792.cn/24024.Doc
rws.by1792.cn/26206.Doc
rws.by1792.cn/46224.Doc
rws.by1792.cn/42640.Doc
rws.by1792.cn/86448.Doc
rws.by1792.cn/00428.Doc
rwa.by1792.cn/28624.Doc
rwa.by1792.cn/44062.Doc
rwa.by1792.cn/84448.Doc
rwa.by1792.cn/06482.Doc
rwa.by1792.cn/24446.Doc
rwa.by1792.cn/22446.Doc
rwa.by1792.cn/57157.Doc
rwa.by1792.cn/00486.Doc
rwa.by1792.cn/22824.Doc
rwa.by1792.cn/04488.Doc
rwp.by1792.cn/86668.Doc
rwp.by1792.cn/60608.Doc
rwp.by1792.cn/66068.Doc
rwp.by1792.cn/40884.Doc
rwp.by1792.cn/24046.Doc
rwp.by1792.cn/51571.Doc
rwp.by1792.cn/26464.Doc
rwp.by1792.cn/06844.Doc
rwp.by1792.cn/95371.Doc
rwp.by1792.cn/86688.Doc
rwo.by1792.cn/62446.Doc
rwo.by1792.cn/68448.Doc
rwo.by1792.cn/46468.Doc
rwo.by1792.cn/42600.Doc
rwo.by1792.cn/04020.Doc
rwo.by1792.cn/62666.Doc
rwo.by1792.cn/68266.Doc
rwo.by1792.cn/60206.Doc
rwo.by1792.cn/00480.Doc
rwo.by1792.cn/62404.Doc
rwi.by1792.cn/40604.Doc
rwi.by1792.cn/46684.Doc
rwi.by1792.cn/86460.Doc
rwi.by1792.cn/13359.Doc
rwi.by1792.cn/88240.Doc
rwi.by1792.cn/68860.Doc
rwi.by1792.cn/86808.Doc
rwi.by1792.cn/62244.Doc
rwi.by1792.cn/82482.Doc
rwi.by1792.cn/80408.Doc
rwu.by1792.cn/68866.Doc
rwu.by1792.cn/06428.Doc
rwu.by1792.cn/84840.Doc
rwu.by1792.cn/62480.Doc
rwu.by1792.cn/06026.Doc
rwu.by1792.cn/00468.Doc
rwu.by1792.cn/04000.Doc
rwu.by1792.cn/82424.Doc
rwu.by1792.cn/64044.Doc
rwu.by1792.cn/20484.Doc
rwy.by1792.cn/02044.Doc
rwy.by1792.cn/02022.Doc
rwy.by1792.cn/26024.Doc
rwy.by1792.cn/24206.Doc
rwy.by1792.cn/62806.Doc
rwy.by1792.cn/04828.Doc
rwy.by1792.cn/97717.Doc
rwy.by1792.cn/59179.Doc
rwy.by1792.cn/88048.Doc
rwy.by1792.cn/62862.Doc
rwt.by1792.cn/06806.Doc
rwt.by1792.cn/02666.Doc
rwt.by1792.cn/44682.Doc
rwt.by1792.cn/39533.Doc
rwt.by1792.cn/88408.Doc
rwt.by1792.cn/84024.Doc
rwt.by1792.cn/28842.Doc
rwt.by1792.cn/37535.Doc
rwt.by1792.cn/02440.Doc
rwt.by1792.cn/60268.Doc
rwr.by1792.cn/22448.Doc
rwr.by1792.cn/00640.Doc
rwr.by1792.cn/02602.Doc
rwr.by1792.cn/64462.Doc
rwr.by1792.cn/84604.Doc
rwr.by1792.cn/00662.Doc
rwr.by1792.cn/68628.Doc
rwr.by1792.cn/44026.Doc
rwr.by1792.cn/80840.Doc
rwr.by1792.cn/88640.Doc
rwe.by1792.cn/00624.Doc
rwe.by1792.cn/11757.Doc
rwe.by1792.cn/62460.Doc
rwe.by1792.cn/13991.Doc
rwe.by1792.cn/44222.Doc
rwe.by1792.cn/40426.Doc
rwe.by1792.cn/20682.Doc
rwe.by1792.cn/84806.Doc
rwe.by1792.cn/04428.Doc
rwe.by1792.cn/06886.Doc
rww.by1792.cn/64242.Doc
rww.by1792.cn/19155.Doc
rww.by1792.cn/86228.Doc
rww.by1792.cn/84204.Doc
rww.by1792.cn/40682.Doc
rww.by1792.cn/71513.Doc
rww.by1792.cn/24086.Doc
rww.by1792.cn/22480.Doc
rww.by1792.cn/02226.Doc
rww.by1792.cn/44860.Doc
rwq.by1792.cn/80462.Doc
rwq.by1792.cn/22860.Doc
rwq.by1792.cn/46002.Doc
rwq.by1792.cn/28280.Doc
rwq.by1792.cn/82882.Doc
rwq.by1792.cn/80400.Doc
rwq.by1792.cn/62042.Doc
rwq.by1792.cn/04022.Doc
rwq.by1792.cn/64226.Doc
rwq.by1792.cn/42866.Doc
rqm.by1792.cn/26640.Doc
rqm.by1792.cn/02604.Doc
rqm.by1792.cn/02864.Doc
rqm.by1792.cn/13135.Doc
rqm.by1792.cn/46688.Doc
rqm.by1792.cn/37137.Doc
rqm.by1792.cn/68288.Doc
rqm.by1792.cn/48400.Doc
rqm.by1792.cn/46440.Doc
rqm.by1792.cn/64082.Doc
rqn.by1792.cn/82666.Doc
rqn.by1792.cn/26664.Doc
rqn.by1792.cn/66824.Doc
rqn.by1792.cn/88006.Doc
rqn.by1792.cn/82808.Doc
rqn.by1792.cn/66424.Doc
rqn.by1792.cn/60868.Doc
rqn.by1792.cn/62860.Doc
rqn.by1792.cn/24084.Doc
rqn.by1792.cn/44024.Doc
rqb.by1792.cn/00206.Doc
rqb.by1792.cn/26884.Doc
rqb.by1792.cn/06006.Doc
rqb.by1792.cn/66824.Doc
rqb.by1792.cn/66602.Doc
rqb.by1792.cn/08208.Doc
rqb.by1792.cn/22262.Doc
rqb.by1792.cn/88228.Doc
rqb.by1792.cn/64082.Doc
rqb.by1792.cn/06866.Doc
rqv.by1792.cn/80000.Doc
rqv.by1792.cn/82084.Doc
rqv.by1792.cn/82626.Doc
rqv.by1792.cn/22440.Doc
rqv.by1792.cn/24240.Doc
rqv.by1792.cn/46220.Doc
rqv.by1792.cn/37995.Doc
rqv.by1792.cn/00286.Doc
rqv.by1792.cn/24804.Doc
rqv.by1792.cn/42284.Doc
rqc.by1792.cn/48624.Doc
rqc.by1792.cn/08688.Doc
rqc.by1792.cn/44464.Doc
rqc.by1792.cn/04626.Doc
rqc.by1792.cn/46662.Doc
rqc.by1792.cn/04808.Doc
rqc.by1792.cn/82006.Doc
rqc.by1792.cn/37131.Doc
rqc.by1792.cn/77395.Doc
rqc.by1792.cn/84244.Doc
rqx.by1792.cn/97113.Doc
rqx.by1792.cn/48682.Doc
rqx.by1792.cn/82848.Doc
rqx.by1792.cn/06288.Doc
rqx.by1792.cn/04024.Doc
rqx.by1792.cn/95793.Doc
rqx.by1792.cn/17711.Doc
rqx.by1792.cn/80064.Doc
rqx.by1792.cn/04822.Doc
rqx.by1792.cn/44228.Doc
rqz.by1792.cn/40266.Doc
rqz.by1792.cn/42462.Doc
rqz.by1792.cn/26000.Doc
rqz.by1792.cn/46820.Doc
rqz.by1792.cn/86206.Doc
rqz.by1792.cn/60660.Doc
rqz.by1792.cn/19379.Doc
rqz.by1792.cn/24820.Doc
rqz.by1792.cn/24884.Doc
rqz.by1792.cn/44046.Doc
rql.by1792.cn/44604.Doc
rql.by1792.cn/37737.Doc
rql.by1792.cn/08242.Doc
rql.by1792.cn/20666.Doc
rql.by1792.cn/86882.Doc
rql.by1792.cn/91799.Doc
rql.by1792.cn/84862.Doc
rql.by1792.cn/66628.Doc
rql.by1792.cn/06660.Doc
rql.by1792.cn/66482.Doc
rqk.by1792.cn/20660.Doc
rqk.by1792.cn/88428.Doc
rqk.by1792.cn/84408.Doc
rqk.by1792.cn/08068.Doc
rqk.by1792.cn/62666.Doc
rqk.by1792.cn/00084.Doc
rqk.by1792.cn/11993.Doc
rqk.by1792.cn/42664.Doc
rqk.by1792.cn/26686.Doc
rqk.by1792.cn/44826.Doc
