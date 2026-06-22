雇在纯亢仪


"""
Python 点击劫持防护 —— 通过 HTTP 响应头防御点击劫持攻击
涵盖 X-Frame-Options、CSP frame-ancestors 和 Django/Flask 中间件实现
"""

# 点击劫持（Clickjacking）攻击者通过透明 iframe 覆盖诱导用户点击
# 防御核心：禁止页面在 iframe 中加载或仅允许同源加载

from typing import Dict, Optional, List
from enum import Enum

# ========== 第一部分：X-Frame-Options 响应头 ==========

class XFrameOptions(Enum):
    """
    X-Frame-Options 头的三个可选值：
    - DENY：完全禁止在 iframe 中加载
    - SAMEORIGIN：仅允许同源页面在 iframe 中加载
    - ALLOW-FROM：允许指定来源加载（已废弃，改用 CSP）
    """
    DENY = "DENY"
    SAMEORIGIN = "SAMEORIGIN"
    ALLOW_FROM = "ALLOW-FROM"


# ========== 第二部分：Content-Security-Policy 头 ==========

class CSPDirectiveBuilder:
    """
    Content-Security-Policy 的 frame-ancestors 指令构建器。
    这是现代浏览器推荐的点击劫持防护方式，比 X-Frame-Options 更灵活。
    frame-ancestors 指定哪些来源可以将当前页面嵌入为 iframe。
    """

    def __init__(self):
        # 默认完全禁止
        self._directives: List[str] = []

    def deny(self) -> "CSPDirectiveBuilder":
        """
        禁止任何页面嵌入（等效于 X-Frame-Options: DENY）。
        """
        self._directives = ["'none'"]
        return self

    def same_origin(self) -> "CSPDirectiveBuilder":
        """
        仅允许同源页面嵌入（等效于 X-Frame-Options: SAMEORIGIN）。
        """
        self._directives = ["'self'"]
        return self

    def allow(self, *origins: str) -> "CSPDirectiveBuilder":
        """
        允许指定的来源嵌入页面。
        来源可以是域名、协议+域名或具体 URL。
        """
        for origin in origins:
            self._directives.append(origin)
        return self

    def build(self) -> str:
        """
        生成完整的 frame-ancestors 指令字符串。
        """
        if not self._directives:
            return "frame-ancestors 'none'"
        # 'self' 必须在最前面（如果有的话）
        directives = sorted(
            self._directives,
            key=lambda x: (0 if x == "'self'" else 1)
        )
        return f"frame-ancestors {' '.join(directives)}"


# ========== 第三部分：Flask 中间件实现 ==========

class FrameOptionsMiddleware:
    """
    Flask 中间件 —— 自动为响应添加点击劫持防护头。
    支持按路由配置不同的防护策略。
    """

    def __init__(self, app, default_policy: str = "DENY"):
        self.app = app
        self.default_policy = default_policy
        # 按端点配置例外策略
        self._exempt_routes: Dict[str, str] = {}

    def add_exemption(self, endpoint: str, policy: str = "SAMEORIGIN"):
        """
        为特定端点配置不同的帧策略。
        例如支付页面可能需要被允许在特定第三方嵌入。
        """
        self._exempt_routes[endpoint] = policy

    def __call__(self, environ, start_response):
        """
        WSGI 中间件调用入口。
        拦截响应并添加安全头。
        """

        def _start_response(status, headers, exc_info=None):
            # 获取当前请求路径
            path = environ.get("PATH_INFO", "/")

            # 确定使用的策略
            policy = self._exempt_routes.get(path, self.default_policy)

            # 添加 X-Frame-Options 头
            if policy == "DENY":
                headers.append(("X-Frame-Options", "DENY"))
            elif policy == "SAMEORIGIN":
                headers.append(("X-Frame-Options", "SAMEORIGIN"))
            elif policy.startswith("ALLOW-FROM"):
                headers.append(("X-Frame-Options", policy))

            # 同时添加 CSP frame-ancestors（双保险）
            csp_builder = CSPDirectiveBuilder()
            if policy == "DENY":
                csp_value = csp_builder.deny().build()
            elif policy == "SAMEORIGIN":
                csp_value = csp_builder.same_origin().build()
            elif policy.startswith("ALLOW-FROM"):
                # 提取允许的来源
                origin = policy.replace("ALLOW-FROM ", "")
                csp_value = csp_builder.allow(origin).build()
            else:
                csp_value = csp_builder.deny().build()

            headers.append(("Content-Security-Policy", csp_value))
            return start_response(status, headers, exc_info)

        return self.app(environ, start_response)


# ========== 第四部分：Django 中间件实现 ==========

class DjangoFrameOptionsMiddleware:
    """
    Django 中间件 —— 自动添加帧防护头。
    在 Django 的 settings.MIDDLEWARE 中注册使用。
    """

    def __init__(self, get_response):
        self.get_response = get_response
        # 按 URL 前缀配置例外
        self._exempt_prefixes: Dict[str, str] = {}

    def add_exempt_prefix(self, prefix: str, policy: str = "SAMEORIGIN"):
        """为特定 URL 前缀设置不同的帧策略"""
        self._exempt_prefixes[prefix] = policy

    def __call__(self, request):
        response = self.get_response(request)

        # 判断使用哪种策略
        policy = self.default_policy
        for prefix, exempt_policy in self._exempt_prefixes.items():
            if request.path.startswith(prefix):
                policy = exempt_policy
                break

        # 添加安全头
        response["X-Frame-Options"] = policy

        # 添加 CSP 头（如果尚未设置）
        if "Content-Security-Policy" not in response:
            csp = self._build_csp(policy)
            response["Content-Security-Policy"] = csp

        return response

    def _build_csp(self, policy: str) -> str:
        """根据策略生成对应的 CSP 值"""
        builder = CSPDirectiveBuilder()
        if policy == "DENY":
            return builder.deny().build()
        elif policy == "SAMEORIGIN":
            return builder.same_origin().build()
        return builder.deny().build()

    @property
    def default_policy(self) -> str:
        return "DENY"


# ========== 第五部分：HTML 级别的额外防护 ==========

def generate_frame_busting_script() -> str:
    """
    生成客户端框架破坏（Frame Busting）JavaScript 代码。
    作为 HTTP 头的补充防护措施，兼容不支持 X-Frame-Options 的旧浏览器。
    注意：某些现代浏览器可能限制这种脚本的执行。
    """
    return """
    <script>
    // 框架破坏脚本 —— 防止页面被嵌入 iframe
    (function() {
        // 检查当前页面是否在最顶层窗口
        if (window !== window.top) {
            // 如果被嵌入 iframe，将顶层窗口重定向到当前页面
            window.top.location = window.location;
        }
        // 防御"双 iframe"绕过技术（sandbox 属性阻止跳转）
        var preventBypass = function() {
            if (window !== window.top) {
                // 使用循环跳转防止被 sandbox 属性阻止
                var body = document.getElementsByTagName('body')[0];
                if (body) {
                    body.style.display = 'none';
                }
                setTimeout(function() {
                    window.top.location = window.location;
                }, 100);
            }
        };
        // 在页面加载完成后再次检查
        if (window.addEventListener) {
            window.addEventListener('load', preventBypass, false);
        } else {
            window.attachEvent('onload', preventBypass);
        }
    })();
    </script>
    """


# ========== 第六部分：演示 ==========

def demo_clickjacking_protection():
    """
    演示各种点击劫持防护策略的配置和效果。
    """
    print("=== 点击劫持防护演示 ===\n")

    # 1. 显示 X-Frame-Options 策略
    print("--- X-Frame-Options 策略 ---")
    for policy in XFrameOptions:
        print(f"  {policy.value}: {policy.name}")

    # 2. 演示 CSP frame-ancestors 构建
    print("\n--- CSP frame-ancestors 构建 ---")

    builder = CSPDirectiveBuilder()
    print(f"  DENY: {builder.deny().build()}")

    builder = CSPDirectiveBuilder()
    print(f"  SAMEORIGIN: {builder.same_origin().build()}")

    builder = CSPDirectiveBuilder()
    print(f"  允许特定来源: {builder.allow('https://trusted.com', 'https://partner.com').build()}")

    # 3. 综合建议
    print("\n--- 推荐配置 ---")
    print("""
    生产环境推荐采用"纵深防御"策略：
    1. HTTP 头：X-Frame-Options: SAMEORIGIN
    2. CSP 头：Content-Security-Policy: frame-ancestors 'self'
    3. 敏感页面（如支付）：X-Frame-Options: DENY
    4. 需要被第三方嵌入的页面：使用 CSP 白名单方式
    5. 关键操作增加二次确认步骤（如验证码）
    """)

    # 4. 生成 Frame Busting 脚本示例
    print("--- Frame Busting 脚本（备用防护）---")
    print(generate_frame_busting_script()[:100] + "...")


if __name__ == "__main__":
    demo_clickjacking_protection()

俸侵硕硬阅倘猿坪独性涨晕炎炮负

rov.mmmxz.cn/066443.htm
rov.mmmxz.cn/484243.htm
rov.mmmxz.cn/791173.htm
rov.mmmxz.cn/753393.htm
rov.mmmxz.cn/713353.htm
rov.mmmxz.cn/064643.htm
rov.mmmxz.cn/802643.htm
rov.mmmxz.cn/371393.htm
rov.mmmxz.cn/793593.htm
rov.mmmxz.cn/460683.htm
roc.mmmxz.cn/117953.htm
roc.mmmxz.cn/046483.htm
roc.mmmxz.cn/919933.htm
roc.mmmxz.cn/137193.htm
roc.mmmxz.cn/662883.htm
roc.mmmxz.cn/151973.htm
roc.mmmxz.cn/802663.htm
roc.mmmxz.cn/137553.htm
roc.mmmxz.cn/175393.htm
roc.mmmxz.cn/082663.htm
rox.mmmxz.cn/733793.htm
rox.mmmxz.cn/664643.htm
rox.mmmxz.cn/391713.htm
rox.mmmxz.cn/866223.htm
rox.mmmxz.cn/535193.htm
rox.mmmxz.cn/202203.htm
rox.mmmxz.cn/288083.htm
rox.mmmxz.cn/955993.htm
rox.mmmxz.cn/000443.htm
rox.mmmxz.cn/331973.htm
roz.mmmxz.cn/537173.htm
roz.mmmxz.cn/515133.htm
roz.mmmxz.cn/244463.htm
roz.mmmxz.cn/842003.htm
roz.mmmxz.cn/000263.htm
roz.mmmxz.cn/262283.htm
roz.mmmxz.cn/462803.htm
roz.mmmxz.cn/517133.htm
roz.mmmxz.cn/444083.htm
roz.mmmxz.cn/642823.htm
rol.mmmxz.cn/957573.htm
rol.mmmxz.cn/400863.htm
rol.mmmxz.cn/008643.htm
rol.mmmxz.cn/602663.htm
rol.mmmxz.cn/339753.htm
rol.mmmxz.cn/644203.htm
rol.mmmxz.cn/559193.htm
rol.mmmxz.cn/240063.htm
rol.mmmxz.cn/246403.htm
rol.mmmxz.cn/260403.htm
rok.mmmxz.cn/226643.htm
rok.mmmxz.cn/775953.htm
rok.mmmxz.cn/882603.htm
rok.mmmxz.cn/119393.htm
rok.mmmxz.cn/448823.htm
rok.mmmxz.cn/157133.htm
rok.mmmxz.cn/400463.htm
rok.mmmxz.cn/539133.htm
rok.mmmxz.cn/799393.htm
rok.mmmxz.cn/191753.htm
roj.mmmxz.cn/406483.htm
roj.mmmxz.cn/604203.htm
roj.mmmxz.cn/579993.htm
roj.mmmxz.cn/599193.htm
roj.mmmxz.cn/397173.htm
roj.mmmxz.cn/266683.htm
roj.mmmxz.cn/991553.htm
roj.mmmxz.cn/735333.htm
roj.mmmxz.cn/424443.htm
roj.mmmxz.cn/864803.htm
roh.mmmxz.cn/486403.htm
roh.mmmxz.cn/824283.htm
roh.mmmxz.cn/357993.htm
roh.mmmxz.cn/642423.htm
roh.mmmxz.cn/971353.htm
roh.mmmxz.cn/973973.htm
roh.mmmxz.cn/868023.htm
roh.mmmxz.cn/139173.htm
roh.mmmxz.cn/913113.htm
roh.mmmxz.cn/480843.htm
rog.mmmxz.cn/206603.htm
rog.mmmxz.cn/628883.htm
rog.mmmxz.cn/739953.htm
rog.mmmxz.cn/666623.htm
rog.mmmxz.cn/684203.htm
rog.mmmxz.cn/957793.htm
rog.mmmxz.cn/159533.htm
rog.mmmxz.cn/840063.htm
rog.mmmxz.cn/660203.htm
rog.mmmxz.cn/119313.htm
rof.mmmxz.cn/200423.htm
rof.mmmxz.cn/226203.htm
rof.mmmxz.cn/088043.htm
rof.mmmxz.cn/351593.htm
rof.mmmxz.cn/357573.htm
rof.mmmxz.cn/628463.htm
rof.mmmxz.cn/539513.htm
rof.mmmxz.cn/719353.htm
rof.mmmxz.cn/800223.htm
rof.mmmxz.cn/046843.htm
rod.mmmxz.cn/868843.htm
rod.mmmxz.cn/424623.htm
rod.mmmxz.cn/131573.htm
rod.mmmxz.cn/840603.htm
rod.mmmxz.cn/402883.htm
rod.mmmxz.cn/020403.htm
rod.mmmxz.cn/200283.htm
rod.mmmxz.cn/513353.htm
rod.mmmxz.cn/246463.htm
rod.mmmxz.cn/288483.htm
ros.mmmxz.cn/797793.htm
ros.mmmxz.cn/484443.htm
ros.mmmxz.cn/204643.htm
ros.mmmxz.cn/935573.htm
ros.mmmxz.cn/006643.htm
ros.mmmxz.cn/628063.htm
ros.mmmxz.cn/288863.htm
ros.mmmxz.cn/826203.htm
ros.mmmxz.cn/808843.htm
ros.mmmxz.cn/557553.htm
roa.mmmxz.cn/806663.htm
roa.mmmxz.cn/006223.htm
roa.mmmxz.cn/537953.htm
roa.mmmxz.cn/442403.htm
roa.mmmxz.cn/624843.htm
roa.mmmxz.cn/193393.htm
roa.mmmxz.cn/420403.htm
roa.mmmxz.cn/733933.htm
roa.mmmxz.cn/882823.htm
roa.mmmxz.cn/331913.htm
rop.mmmxz.cn/995953.htm
rop.mmmxz.cn/139773.htm
rop.mmmxz.cn/800223.htm
rop.mmmxz.cn/333173.htm
rop.mmmxz.cn/977353.htm
rop.mmmxz.cn/137773.htm
rop.mmmxz.cn/022603.htm
rop.mmmxz.cn/486603.htm
rop.mmmxz.cn/640463.htm
rop.mmmxz.cn/353913.htm
roo.mmmxz.cn/519753.htm
roo.mmmxz.cn/020403.htm
roo.mmmxz.cn/644843.htm
roo.mmmxz.cn/824043.htm
roo.mmmxz.cn/719153.htm
roo.mmmxz.cn/442223.htm
roo.mmmxz.cn/244403.htm
roo.mmmxz.cn/802263.htm
roo.mmmxz.cn/551713.htm
roo.mmmxz.cn/131713.htm
roi.mmmxz.cn/606003.htm
roi.mmmxz.cn/866063.htm
roi.mmmxz.cn/977373.htm
roi.mmmxz.cn/642643.htm
roi.mmmxz.cn/591713.htm
roi.mmmxz.cn/446883.htm
roi.mmmxz.cn/408443.htm
roi.mmmxz.cn/159593.htm
roi.mmmxz.cn/135753.htm
roi.mmmxz.cn/133513.htm
rou.mmmxz.cn/620423.htm
rou.mmmxz.cn/824603.htm
rou.mmmxz.cn/402223.htm
rou.mmmxz.cn/193773.htm
rou.mmmxz.cn/539993.htm
rou.mmmxz.cn/593913.htm
rou.mmmxz.cn/204803.htm
rou.mmmxz.cn/660603.htm
rou.mmmxz.cn/997173.htm
rou.mmmxz.cn/640263.htm
roy.mmmxz.cn/711993.htm
roy.mmmxz.cn/931313.htm
roy.mmmxz.cn/191393.htm
roy.mmmxz.cn/642823.htm
roy.mmmxz.cn/042483.htm
roy.mmmxz.cn/797113.htm
roy.mmmxz.cn/006843.htm
roy.mmmxz.cn/973793.htm
roy.mmmxz.cn/222043.htm
roy.mmmxz.cn/397353.htm
rot.mmmxz.cn/399513.htm
rot.mmmxz.cn/668843.htm
rot.mmmxz.cn/084283.htm
rot.mmmxz.cn/440843.htm
rot.mmmxz.cn/686863.htm
rot.mmmxz.cn/511313.htm
rot.mmmxz.cn/060403.htm
rot.mmmxz.cn/397393.htm
rot.mmmxz.cn/179373.htm
rot.mmmxz.cn/426843.htm
ror.mmmxz.cn/331393.htm
ror.mmmxz.cn/402463.htm
ror.mmmxz.cn/606683.htm
ror.mmmxz.cn/393393.htm
ror.mmmxz.cn/846443.htm
ror.mmmxz.cn/977793.htm
ror.mmmxz.cn/644463.htm
ror.mmmxz.cn/688203.htm
ror.mmmxz.cn/808843.htm
ror.mmmxz.cn/088043.htm
roe.mmmxz.cn/846263.htm
roe.mmmxz.cn/244683.htm
roe.mmmxz.cn/771753.htm
roe.mmmxz.cn/759713.htm
roe.mmmxz.cn/575193.htm
roe.mmmxz.cn/488403.htm
roe.mmmxz.cn/975393.htm
roe.mmmxz.cn/000003.htm
roe.mmmxz.cn/997753.htm
roe.mmmxz.cn/604803.htm
row.mmmxz.cn/264403.htm
row.mmmxz.cn/593333.htm
row.mmmxz.cn/288083.htm
row.mmmxz.cn/157353.htm
row.mmmxz.cn/379593.htm
row.mmmxz.cn/800043.htm
row.mmmxz.cn/080843.htm
row.mmmxz.cn/191913.htm
row.mmmxz.cn/000423.htm
row.mmmxz.cn/131173.htm
roq.mmmxz.cn/064483.htm
roq.mmmxz.cn/531913.htm
roq.mmmxz.cn/480463.htm
roq.mmmxz.cn/751713.htm
roq.mmmxz.cn/488063.htm
roq.mmmxz.cn/951513.htm
roq.mmmxz.cn/866243.htm
roq.mmmxz.cn/488223.htm
roq.mmmxz.cn/204003.htm
roq.mmmxz.cn/137733.htm
rim.mmmxz.cn/088043.htm
rim.mmmxz.cn/420263.htm
rim.mmmxz.cn/064023.htm
rim.mmmxz.cn/155793.htm
rim.mmmxz.cn/842263.htm
rim.mmmxz.cn/424023.htm
rim.mmmxz.cn/133173.htm
rim.mmmxz.cn/684823.htm
rim.mmmxz.cn/951313.htm
rim.mmmxz.cn/224243.htm
rin.mmmxz.cn/173933.htm
rin.mmmxz.cn/482223.htm
rin.mmmxz.cn/860003.htm
rin.mmmxz.cn/371533.htm
rin.mmmxz.cn/860683.htm
rin.mmmxz.cn/888863.htm
rin.mmmxz.cn/408243.htm
rin.mmmxz.cn/571773.htm
rin.mmmxz.cn/828023.htm
rin.mmmxz.cn/351173.htm
rib.mmmxz.cn/644063.htm
rib.mmmxz.cn/993313.htm
rib.mmmxz.cn/339913.htm
rib.mmmxz.cn/113353.htm
rib.mmmxz.cn/573753.htm
rib.mmmxz.cn/284683.htm
rib.mmmxz.cn/808643.htm
rib.mmmxz.cn/608803.htm
rib.mmmxz.cn/717513.htm
rib.mmmxz.cn/040863.htm
riv.mmmxz.cn/026683.htm
riv.mmmxz.cn/333113.htm
riv.mmmxz.cn/462003.htm
riv.mmmxz.cn/577333.htm
riv.mmmxz.cn/599553.htm
riv.mmmxz.cn/531553.htm
riv.mmmxz.cn/351333.htm
riv.mmmxz.cn/666223.htm
riv.mmmxz.cn/488683.htm
riv.mmmxz.cn/995793.htm
ric.mmmxz.cn/400283.htm
ric.mmmxz.cn/208663.htm
ric.mmmxz.cn/797993.htm
ric.mmmxz.cn/444463.htm
ric.mmmxz.cn/242403.htm
ric.mmmxz.cn/979353.htm
ric.mmmxz.cn/975373.htm
ric.mmmxz.cn/313953.htm
ric.mmmxz.cn/666883.htm
ric.mmmxz.cn/191793.htm
rix.mmmxz.cn/573533.htm
rix.mmmxz.cn/422463.htm
rix.mmmxz.cn/395793.htm
rix.mmmxz.cn/844243.htm
rix.mmmxz.cn/828803.htm
rix.mmmxz.cn/402043.htm
rix.mmmxz.cn/846603.htm
rix.mmmxz.cn/086063.htm
rix.mmmxz.cn/975913.htm
rix.mmmxz.cn/668203.htm
riz.mmmxz.cn/286863.htm
riz.mmmxz.cn/579513.htm
riz.mmmxz.cn/466843.htm
riz.mmmxz.cn/139113.htm
riz.mmmxz.cn/464243.htm
riz.mmmxz.cn/228863.htm
riz.mmmxz.cn/004623.htm
riz.mmmxz.cn/177793.htm
riz.mmmxz.cn/353133.htm
riz.mmmxz.cn/688083.htm
ril.mmmxz.cn/573773.htm
ril.mmmxz.cn/244883.htm
ril.mmmxz.cn/515553.htm
ril.mmmxz.cn/175133.htm
ril.mmmxz.cn/151753.htm
ril.mmmxz.cn/993373.htm
ril.mmmxz.cn/933113.htm
ril.mmmxz.cn/377353.htm
ril.mmmxz.cn/286023.htm
ril.mmmxz.cn/480403.htm
rik.mmmxz.cn/119953.htm
rik.mmmxz.cn/204443.htm
rik.mmmxz.cn/442683.htm
rik.mmmxz.cn/377573.htm
rik.mmmxz.cn/755553.htm
rik.mmmxz.cn/931193.htm
rik.mmmxz.cn/753513.htm
rik.mmmxz.cn/464043.htm
rik.mmmxz.cn/466403.htm
rik.mmmxz.cn/266023.htm
rij.mmmxz.cn/953153.htm
rij.mmmxz.cn/860803.htm
rij.mmmxz.cn/688623.htm
rij.mmmxz.cn/537993.htm
rij.mmmxz.cn/226043.htm
rij.mmmxz.cn/971133.htm
rij.mmmxz.cn/577313.htm
rij.mmmxz.cn/191933.htm
rij.mmmxz.cn/048023.htm
rij.mmmxz.cn/139173.htm
rih.mmmxz.cn/842863.htm
rih.mmmxz.cn/088623.htm
rih.mmmxz.cn/599333.htm
rih.mmmxz.cn/571113.htm
rih.mmmxz.cn/373133.htm
rih.mmmxz.cn/353133.htm
rih.mmmxz.cn/206663.htm
rih.mmmxz.cn/559173.htm
rih.mmmxz.cn/288403.htm
rih.mmmxz.cn/666623.htm
rig.sthxr.cn/335993.htm
rig.sthxr.cn/466023.htm
rig.sthxr.cn/244063.htm
rig.sthxr.cn/991113.htm
rig.sthxr.cn/882823.htm
rig.sthxr.cn/822643.htm
rig.sthxr.cn/060403.htm
rig.sthxr.cn/282403.htm
rig.sthxr.cn/159153.htm
rig.sthxr.cn/735933.htm
rif.sthxr.cn/466823.htm
rif.sthxr.cn/240443.htm
rif.sthxr.cn/737713.htm
rif.sthxr.cn/460203.htm
rif.sthxr.cn/993913.htm
rif.sthxr.cn/979133.htm
rif.sthxr.cn/537713.htm
rif.sthxr.cn/462043.htm
rif.sthxr.cn/313913.htm
rif.sthxr.cn/288083.htm
rid.sthxr.cn/062243.htm
rid.sthxr.cn/428483.htm
rid.sthxr.cn/517373.htm
rid.sthxr.cn/624883.htm
rid.sthxr.cn/359393.htm
rid.sthxr.cn/424283.htm
rid.sthxr.cn/133773.htm
rid.sthxr.cn/357353.htm
rid.sthxr.cn/191753.htm
rid.sthxr.cn/826423.htm
ris.sthxr.cn/919513.htm
ris.sthxr.cn/777953.htm
ris.sthxr.cn/400603.htm
ris.sthxr.cn/406643.htm
ris.sthxr.cn/173913.htm
ris.sthxr.cn/608803.htm
ris.sthxr.cn/882623.htm
ris.sthxr.cn/957133.htm
ris.sthxr.cn/642043.htm
ris.sthxr.cn/624003.htm
ria.sthxr.cn/933733.htm
ria.sthxr.cn/884443.htm
ria.sthxr.cn/808643.htm
ria.sthxr.cn/757793.htm
ria.sthxr.cn/377913.htm
ria.sthxr.cn/599993.htm
ria.sthxr.cn/024823.htm
ria.sthxr.cn/953993.htm
ria.sthxr.cn/022263.htm
ria.sthxr.cn/268023.htm
rip.sthxr.cn/200423.htm
rip.sthxr.cn/440403.htm
rip.sthxr.cn/997933.htm
rip.sthxr.cn/606663.htm
rip.sthxr.cn/648063.htm
