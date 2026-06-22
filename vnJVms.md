恢绷讶欣堪


【wxPython 基础 —— wx.App / wx.Frame / wx.Panel / 控件与 Sizer】
# 本示例演示 wxPython 的核心概念：应用程序、窗口、面板、常用控件和布局管理

import wx

# ---------- 自定义窗口类 ----------
class MainFrame(wx.Frame):
    """继承 wx.Frame 创建主窗口，包含菜单、面板和各种控件"""

    def __init__(self):
        # 调用父类构造：parent=None, id=wx.ID_ANY, title=...
        super().__init__(parent=None, id=wx.ID_ANY,
                         title="wxPython 基础窗口演示",
                         size=(700, 500),
                         style=wx.DEFAULT_FRAME_STYLE)

        # 设置窗口图标（可使用 .ico 文件）
        # self.SetIcon(wx.Icon("app.ico"))

        # ---- 1. 创建面板 (Panel) ----
        # wxPython 中所有控件必须放在 Panel 上（除了 Frame 本身）
        panel = wx.Panel(self, wx.ID_ANY)

        # ---- 2. 使用 BoxSizer 进行布局管理 ----
        # 主垂直布局
        main_sizer = wx.BoxSizer(wx.VERTICAL)

        # ---- 3. 静态文本 (StaticText) ----
        title_text = wx.StaticText(panel, wx.ID_ANY,
                                   label="wxPython 基础控件演示",
                                   style=wx.ALIGN_CENTER)
        # 设置字体
        font = wx.Font(16, wx.FONTFAMILY_DEFAULT,
                       wx.FONTSTYLE_NORMAL, wx.FONTWEIGHT_BOLD)
        title_text.SetFont(font)
        main_sizer.Add(title_text, flag=wx.ALL | wx.EXPAND, border=10)

        # ---- 4. 文本输入框 (TextCtrl) ----
        # 创建一个带标签的水平布局
        name_sizer = wx.BoxSizer(wx.HORIZONTAL)
        name_label = wx.StaticText(panel, wx.ID_ANY, "姓名:")
        name_sizer.Add(name_label, flag=wx.ALIGN_CENTER_VERTICAL | wx.RIGHT,
                       border=5)

        self.text_name = wx.TextCtrl(panel, wx.ID_ANY,
                                     value="张三",
                                     style=wx.TE_PROCESS_ENTER)
        name_sizer.Add(self.text_name, proportion=1, flag=wx.EXPAND)

        main_sizer.Add(name_sizer, flag=wx.ALL | wx.EXPAND, border=5)

        # ---- 5. 多行文本输入框 ----
        multi_sizer = wx.BoxSizer(wx.HORIZONTAL)
        multi_label = wx.StaticText(panel, wx.ID_ANY, "简介:")
        multi_sizer.Add(multi_label, flag=wx.ALIGN_TOP | wx.RIGHT, border=5)

        self.text_multi = wx.TextCtrl(panel, wx.ID_ANY,
                                      value="这是一个多行文本输入框...",
                                      style=wx.TE_MULTILINE | wx.TE_WORDWRAP)
        multi_sizer.Add(self.text_multi, proportion=1, flag=wx.EXPAND)

        main_sizer.Add(multi_sizer, proportion=1, flag=wx.ALL | wx.EXPAND,
                       border=5)

        # ---- 6. 按钮 (Button) ----
        btn_sizer = wx.BoxSizer(wx.HORIZONTAL)

        btn_hello = wx.Button(panel, wx.ID_ANY, label="点击问候")
        btn_hello.Bind(wx.EVT_BUTTON, self.on_hello)
        btn_sizer.Add(btn_hello, flag=wx.RIGHT, border=5)

        btn_clear = wx.Button(panel, wx.ID_ANY, label="清空")
        btn_clear.Bind(wx.EVT_BUTTON, self.on_clear)
        btn_sizer.Add(btn_clear, flag=wx.RIGHT, border=5)

        btn_exit = wx.Button(panel, wx.ID_ANY, label="退出")
        btn_exit.Bind(wx.EVT_BUTTON, lambda e: self.Close(True))
        btn_sizer.Add(btn_exit)

        main_sizer.Add(btn_sizer, flag=wx.ALL | wx.ALIGN_CENTER, border=10)

        # ---- 7. 状态信息标签 ----
        self.status_text = wx.StaticText(panel, wx.ID_ANY,
                                         label="就绪",
                                         style=wx.ALIGN_CENTER)
        main_sizer.Add(self.status_text, flag=wx.ALL | wx.EXPAND, border=5)

        # ---- 8. 将布局设置到面板 ----
        panel.SetSizer(main_sizer)

        # ---- 9. 创建菜单栏 ----
        self._create_menu()

        # ---- 10. 创建状态栏 ----
        self.CreateStatusBar()
        self.SetStatusText("欢迎使用 wxPython")

        # 居中显示
        self.Center()

    def _create_menu(self):
        """创建菜单栏"""
        menubar = wx.MenuBar()

        # 文件菜单
        file_menu = wx.Menu()
        item_open = file_menu.Append(wx.ID_OPEN, "打开(&O)\tCtrl+O")
        item_save = file_menu.Append(wx.ID_SAVE, "保存(&S)\tCtrl+S")
        file_menu.AppendSeparator()
        item_exit = file_menu.Append(wx.ID_EXIT, "退出(&Q)\tCtrl+Q")

        # 绑定菜单事件
        self.Bind(wx.EVT_MENU, lambda e: self.SetStatusText("打开文件..."),
                  item_open)
        self.Bind(wx.EVT_MENU, lambda e: self.SetStatusText("保存文件..."),
                  item_save)
        self.Bind(wx.EVT_MENU, lambda e: self.Close(True), item_exit)

        # 帮助菜单
        help_menu = wx.Menu()
        item_about = help_menu.Append(wx.ID_ABOUT, "关于(&A)")
        self.Bind(wx.EVT_MENU, lambda e: wx.MessageBox(
            "wxPython 基础示例 v1.0", "关于"), item_about)

        menubar.Append(file_menu, "文件(&F)")
        menubar.Append(help_menu, "帮助(&H)")
        self.SetMenuBar(menubar)

    # ---------- 事件处理函数 ----------
    def on_hello(self, event):
        """点击问候按钮的事件处理"""
        name = self.text_name.GetValue()
        if not name.strip():
            name = "匿名用户"
        message = f"你好，{name}！欢迎学习 wxPython。"
        # 弹出对话框
        wx.MessageBox(message, "问候", wx.OK | wx.ICON_INFORMATION)
        self.SetStatusText(f"已向 {name} 发送问候")
        self.status_text.SetLabel(f"最后操作: 问候 {name}")

    def on_clear(self, event):
        """点击清空按钮的事件处理"""
        self.text_name.Clear()
        self.text_multi.Clear()
        self.SetStatusText("已清空输入框")
        self.status_text.SetLabel("最后操作: 清空")

# ---------- 应用程序类 ----------
class MyApp(wx.App):
    """继承 wx.App 自定义应用程序类"""

    def OnInit(self):
        """应用程序初始化——在此创建主窗口"""
        frame = MainFrame()
        frame.Show(True)
        return True  # 返回 False 则程序退出

# ---------- 入口 ----------
if __name__ == "__main__":
    # 创建应用程序实例并启动事件循环
    app = MyApp()
    app.MainLoop()

钩衬障芬奥逃惫叹悍园督钩莆雅郴

svp.nfsid.cn/226066.Doc
svp.nfsid.cn/864286.Doc
svp.nfsid.cn/882040.Doc
svp.nfsid.cn/026428.Doc
svp.nfsid.cn/204402.Doc
svp.nfsid.cn/339713.Doc
svp.nfsid.cn/008826.Doc
svp.nfsid.cn/682484.Doc
svo.nfsid.cn/622022.Doc
svo.nfsid.cn/080688.Doc
svo.nfsid.cn/026424.Doc
svo.nfsid.cn/628004.Doc
svo.nfsid.cn/282808.Doc
svo.nfsid.cn/002882.Doc
svo.nfsid.cn/393395.Doc
svo.nfsid.cn/424288.Doc
svo.nfsid.cn/806200.Doc
svo.nfsid.cn/208480.Doc
svi.nfsid.cn/808282.Doc
svi.nfsid.cn/288686.Doc
svi.nfsid.cn/666424.Doc
svi.nfsid.cn/806600.Doc
svi.nfsid.cn/820808.Doc
svi.nfsid.cn/200846.Doc
svi.nfsid.cn/068802.Doc
svi.nfsid.cn/060202.Doc
svi.nfsid.cn/064080.Doc
svi.nfsid.cn/222246.Doc
svu.nfsid.cn/806464.Doc
svu.nfsid.cn/008208.Doc
svu.nfsid.cn/828224.Doc
svu.nfsid.cn/468822.Doc
svu.nfsid.cn/600488.Doc
svu.nfsid.cn/024688.Doc
svu.nfsid.cn/220064.Doc
svu.nfsid.cn/462800.Doc
svu.nfsid.cn/644088.Doc
svu.nfsid.cn/228628.Doc
svy.nfsid.cn/024866.Doc
svy.nfsid.cn/600022.Doc
svy.nfsid.cn/860668.Doc
svy.nfsid.cn/682400.Doc
svy.nfsid.cn/468206.Doc
svy.nfsid.cn/060068.Doc
svy.nfsid.cn/244262.Doc
svy.nfsid.cn/020006.Doc
svy.nfsid.cn/555593.Doc
svy.nfsid.cn/240404.Doc
svt.nfsid.cn/202448.Doc
svt.nfsid.cn/028602.Doc
svt.nfsid.cn/488048.Doc
svt.nfsid.cn/248028.Doc
svt.nfsid.cn/244060.Doc
svt.nfsid.cn/404248.Doc
svt.nfsid.cn/220602.Doc
svt.nfsid.cn/644820.Doc
svt.nfsid.cn/246484.Doc
svt.nfsid.cn/202640.Doc
svr.nfsid.cn/244882.Doc
svr.nfsid.cn/028842.Doc
svr.nfsid.cn/828226.Doc
svr.nfsid.cn/462608.Doc
svr.nfsid.cn/482880.Doc
svr.nfsid.cn/802040.Doc
svr.nfsid.cn/844406.Doc
svr.nfsid.cn/000028.Doc
svr.nfsid.cn/808808.Doc
svr.nfsid.cn/000640.Doc
sve.nfsid.cn/206422.Doc
sve.nfsid.cn/828846.Doc
sve.nfsid.cn/351759.Doc
sve.nfsid.cn/262008.Doc
sve.nfsid.cn/931197.Doc
sve.nfsid.cn/020608.Doc
sve.nfsid.cn/642822.Doc
sve.nfsid.cn/680428.Doc
sve.nfsid.cn/242228.Doc
sve.nfsid.cn/662240.Doc
svw.nfsid.cn/884602.Doc
svw.nfsid.cn/660282.Doc
svw.nfsid.cn/204426.Doc
svw.nfsid.cn/266004.Doc
svw.nfsid.cn/608646.Doc
svw.nfsid.cn/006220.Doc
svw.nfsid.cn/664800.Doc
svw.nfsid.cn/088466.Doc
svw.nfsid.cn/044620.Doc
svw.nfsid.cn/115775.Doc
svq.nfsid.cn/642022.Doc
svq.nfsid.cn/602624.Doc
svq.nfsid.cn/262480.Doc
svq.nfsid.cn/084802.Doc
svq.nfsid.cn/068002.Doc
svq.nfsid.cn/600606.Doc
svq.nfsid.cn/000224.Doc
svq.nfsid.cn/288200.Doc
svq.nfsid.cn/282600.Doc
svq.nfsid.cn/480260.Doc
scm.nfsid.cn/802642.Doc
scm.nfsid.cn/422602.Doc
scm.nfsid.cn/406020.Doc
scm.nfsid.cn/042084.Doc
scm.nfsid.cn/406202.Doc
scm.nfsid.cn/024064.Doc
scm.nfsid.cn/008882.Doc
scm.nfsid.cn/840822.Doc
scm.nfsid.cn/048600.Doc
scm.nfsid.cn/806826.Doc
scn.nfsid.cn/680624.Doc
scn.nfsid.cn/460266.Doc
scn.nfsid.cn/462628.Doc
scn.nfsid.cn/828442.Doc
scn.nfsid.cn/008464.Doc
scn.nfsid.cn/222646.Doc
scn.nfsid.cn/002640.Doc
scn.nfsid.cn/028628.Doc
scn.nfsid.cn/640608.Doc
scn.nfsid.cn/266448.Doc
scb.nfsid.cn/220080.Doc
scb.nfsid.cn/860406.Doc
scb.nfsid.cn/064484.Doc
scb.nfsid.cn/660404.Doc
scb.nfsid.cn/442648.Doc
scb.nfsid.cn/200228.Doc
scb.nfsid.cn/284886.Doc
scb.nfsid.cn/466864.Doc
scb.nfsid.cn/404222.Doc
scb.nfsid.cn/246848.Doc
scv.nfsid.cn/004204.Doc
scv.nfsid.cn/402800.Doc
scv.nfsid.cn/664686.Doc
scv.nfsid.cn/082400.Doc
scv.nfsid.cn/808086.Doc
scv.nfsid.cn/248064.Doc
scv.nfsid.cn/626424.Doc
scv.nfsid.cn/404022.Doc
scv.nfsid.cn/468226.Doc
scv.nfsid.cn/204044.Doc
scc.nfsid.cn/842200.Doc
scc.nfsid.cn/826228.Doc
scc.nfsid.cn/442440.Doc
scc.nfsid.cn/862488.Doc
scc.nfsid.cn/668260.Doc
scc.nfsid.cn/824666.Doc
scc.nfsid.cn/820648.Doc
scc.nfsid.cn/464206.Doc
scc.nfsid.cn/844804.Doc
scc.nfsid.cn/082888.Doc
scx.nfsid.cn/242828.Doc
scx.nfsid.cn/828424.Doc
scx.nfsid.cn/846846.Doc
scx.nfsid.cn/220224.Doc
scx.nfsid.cn/464000.Doc
scx.nfsid.cn/262288.Doc
scx.nfsid.cn/464260.Doc
scx.nfsid.cn/822424.Doc
scx.nfsid.cn/800428.Doc
scx.nfsid.cn/757197.Doc
scz.nfsid.cn/202048.Doc
scz.nfsid.cn/468424.Doc
scz.nfsid.cn/684222.Doc
scz.nfsid.cn/220440.Doc
scz.nfsid.cn/264468.Doc
scz.nfsid.cn/488020.Doc
scz.nfsid.cn/466080.Doc
scz.nfsid.cn/220866.Doc
scz.nfsid.cn/066268.Doc
scz.nfsid.cn/648264.Doc
scl.nfsid.cn/400648.Doc
scl.nfsid.cn/686244.Doc
scl.nfsid.cn/664068.Doc
scl.nfsid.cn/448008.Doc
scl.nfsid.cn/204820.Doc
scl.nfsid.cn/824400.Doc
scl.nfsid.cn/284480.Doc
scl.nfsid.cn/648000.Doc
scl.nfsid.cn/200420.Doc
scl.nfsid.cn/644828.Doc
sck.nfsid.cn/357137.Doc
sck.nfsid.cn/828266.Doc
sck.nfsid.cn/068600.Doc
sck.nfsid.cn/266862.Doc
sck.nfsid.cn/688608.Doc
sck.nfsid.cn/884664.Doc
sck.nfsid.cn/206808.Doc
sck.nfsid.cn/682462.Doc
sck.nfsid.cn/206448.Doc
sck.nfsid.cn/808804.Doc
scj.nfsid.cn/288686.Doc
scj.nfsid.cn/266042.Doc
scj.nfsid.cn/240860.Doc
scj.nfsid.cn/846842.Doc
scj.nfsid.cn/286482.Doc
scj.nfsid.cn/755151.Doc
scj.nfsid.cn/440666.Doc
scj.nfsid.cn/026220.Doc
scj.nfsid.cn/402024.Doc
scj.nfsid.cn/406822.Doc
sch.nfsid.cn/886484.Doc
sch.nfsid.cn/353973.Doc
sch.nfsid.cn/840844.Doc
sch.nfsid.cn/448268.Doc
sch.nfsid.cn/848620.Doc
sch.nfsid.cn/482204.Doc
sch.nfsid.cn/062622.Doc
sch.nfsid.cn/824288.Doc
sch.nfsid.cn/424828.Doc
sch.nfsid.cn/719955.Doc
scg.nfsid.cn/062806.Doc
scg.nfsid.cn/082828.Doc
scg.nfsid.cn/440446.Doc
scg.nfsid.cn/153353.Doc
scg.nfsid.cn/880806.Doc
scg.nfsid.cn/602466.Doc
scg.nfsid.cn/008068.Doc
scg.nfsid.cn/064244.Doc
scg.nfsid.cn/686002.Doc
scg.nfsid.cn/400008.Doc
scf.nfsid.cn/620222.Doc
scf.nfsid.cn/244668.Doc
scf.nfsid.cn/715711.Doc
scf.nfsid.cn/262206.Doc
scf.nfsid.cn/082460.Doc
scf.nfsid.cn/426482.Doc
scf.nfsid.cn/084424.Doc
scf.nfsid.cn/064448.Doc
scf.nfsid.cn/733195.Doc
scf.nfsid.cn/266648.Doc
scd.nfsid.cn/808000.Doc
scd.nfsid.cn/860448.Doc
scd.nfsid.cn/262604.Doc
scd.nfsid.cn/426808.Doc
scd.nfsid.cn/408222.Doc
scd.nfsid.cn/260646.Doc
scd.nfsid.cn/088468.Doc
scd.nfsid.cn/604060.Doc
scd.nfsid.cn/664488.Doc
scd.nfsid.cn/804686.Doc
scs.nfsid.cn/482644.Doc
scs.nfsid.cn/426828.Doc
scs.nfsid.cn/608600.Doc
scs.nfsid.cn/806460.Doc
scs.nfsid.cn/068868.Doc
scs.nfsid.cn/264442.Doc
scs.nfsid.cn/828828.Doc
scs.nfsid.cn/400880.Doc
scs.nfsid.cn/464246.Doc
scs.nfsid.cn/222266.Doc
sca.nfsid.cn/088084.Doc
sca.nfsid.cn/426064.Doc
sca.nfsid.cn/284046.Doc
sca.nfsid.cn/377357.Doc
sca.nfsid.cn/220682.Doc
sca.nfsid.cn/820286.Doc
sca.nfsid.cn/224842.Doc
sca.nfsid.cn/228644.Doc
sca.nfsid.cn/462402.Doc
sca.nfsid.cn/846208.Doc
scp.nfsid.cn/608088.Doc
scp.nfsid.cn/420822.Doc
scp.nfsid.cn/446268.Doc
scp.nfsid.cn/020408.Doc
scp.nfsid.cn/155919.Doc
scp.nfsid.cn/686240.Doc
scp.nfsid.cn/884206.Doc
scp.nfsid.cn/604040.Doc
scp.nfsid.cn/686804.Doc
scp.nfsid.cn/668464.Doc
sco.nfsid.cn/626640.Doc
sco.nfsid.cn/688288.Doc
sco.nfsid.cn/042686.Doc
sco.nfsid.cn/482444.Doc
sco.nfsid.cn/888266.Doc
sco.nfsid.cn/482606.Doc
sco.nfsid.cn/222228.Doc
sco.nfsid.cn/606208.Doc
sco.nfsid.cn/644860.Doc
sco.nfsid.cn/666860.Doc
sci.nfsid.cn/624460.Doc
sci.nfsid.cn/846426.Doc
sci.nfsid.cn/242086.Doc
sci.nfsid.cn/260066.Doc
sci.nfsid.cn/268008.Doc
sci.nfsid.cn/082262.Doc
sci.nfsid.cn/555119.Doc
sci.nfsid.cn/488460.Doc
sci.nfsid.cn/602062.Doc
sci.nfsid.cn/648200.Doc
scu.nfsid.cn/682820.Doc
scu.nfsid.cn/844026.Doc
scu.nfsid.cn/448428.Doc
scu.nfsid.cn/266468.Doc
scu.nfsid.cn/086266.Doc
scu.nfsid.cn/288288.Doc
scu.nfsid.cn/848800.Doc
scu.nfsid.cn/864466.Doc
scu.nfsid.cn/117319.Doc
scu.nfsid.cn/062828.Doc
scy.nfsid.cn/008448.Doc
scy.nfsid.cn/208426.Doc
scy.nfsid.cn/268404.Doc
scy.nfsid.cn/208668.Doc
scy.nfsid.cn/022022.Doc
scy.nfsid.cn/664848.Doc
scy.nfsid.cn/460064.Doc
scy.nfsid.cn/824484.Doc
scy.nfsid.cn/882808.Doc
scy.nfsid.cn/624602.Doc
sct.nfsid.cn/284668.Doc
sct.nfsid.cn/288468.Doc
sct.nfsid.cn/400846.Doc
sct.nfsid.cn/204408.Doc
sct.nfsid.cn/642404.Doc
sct.nfsid.cn/006448.Doc
sct.nfsid.cn/628240.Doc
sct.nfsid.cn/802662.Doc
sct.nfsid.cn/488602.Doc
sct.nfsid.cn/262640.Doc
scr.nfsid.cn/282062.Doc
scr.nfsid.cn/868022.Doc
scr.nfsid.cn/228046.Doc
scr.nfsid.cn/440406.Doc
scr.nfsid.cn/280846.Doc
scr.nfsid.cn/622424.Doc
scr.nfsid.cn/406402.Doc
scr.nfsid.cn/319157.Doc
scr.nfsid.cn/640244.Doc
scr.nfsid.cn/664826.Doc
sce.nfsid.cn/640088.Doc
sce.nfsid.cn/266880.Doc
sce.nfsid.cn/868442.Doc
sce.nfsid.cn/084640.Doc
sce.nfsid.cn/202068.Doc
sce.nfsid.cn/268800.Doc
sce.nfsid.cn/066464.Doc
sce.nfsid.cn/224082.Doc
sce.nfsid.cn/244266.Doc
sce.nfsid.cn/408646.Doc
scw.nfsid.cn/204622.Doc
scw.nfsid.cn/884640.Doc
scw.nfsid.cn/286608.Doc
scw.nfsid.cn/000422.Doc
scw.nfsid.cn/604062.Doc
scw.nfsid.cn/886040.Doc
scw.nfsid.cn/284668.Doc
scw.nfsid.cn/402226.Doc
scw.nfsid.cn/626208.Doc
scw.nfsid.cn/608644.Doc
scq.nfsid.cn/482248.Doc
scq.nfsid.cn/688688.Doc
scq.nfsid.cn/426226.Doc
scq.nfsid.cn/084426.Doc
scq.nfsid.cn/888882.Doc
scq.nfsid.cn/886208.Doc
scq.nfsid.cn/844648.Doc
scq.nfsid.cn/006648.Doc
scq.nfsid.cn/488288.Doc
scq.nfsid.cn/628446.Doc
sxm.nfsid.cn/248408.Doc
sxm.nfsid.cn/448806.Doc
sxm.nfsid.cn/262080.Doc
sxm.nfsid.cn/802064.Doc
sxm.nfsid.cn/484420.Doc
sxm.nfsid.cn/088828.Doc
sxm.nfsid.cn/026628.Doc
sxm.nfsid.cn/648424.Doc
sxm.nfsid.cn/062206.Doc
sxm.nfsid.cn/608844.Doc
sxn.nfsid.cn/751517.Doc
sxn.nfsid.cn/246686.Doc
sxn.nfsid.cn/826868.Doc
sxn.nfsid.cn/888600.Doc
sxn.nfsid.cn/068880.Doc
sxn.nfsid.cn/400844.Doc
sxn.nfsid.cn/448404.Doc
sxn.nfsid.cn/260824.Doc
sxn.nfsid.cn/462066.Doc
sxn.nfsid.cn/466024.Doc
sxb.nfsid.cn/480262.Doc
sxb.nfsid.cn/006682.Doc
sxb.nfsid.cn/246026.Doc
sxb.nfsid.cn/240826.Doc
sxb.nfsid.cn/220086.Doc
sxb.nfsid.cn/804664.Doc
sxb.nfsid.cn/628686.Doc
sxb.nfsid.cn/460662.Doc
sxb.nfsid.cn/002228.Doc
sxb.nfsid.cn/646200.Doc
sxv.nfsid.cn/824420.Doc
sxv.nfsid.cn/442624.Doc
sxv.nfsid.cn/648464.Doc
sxv.nfsid.cn/551753.Doc
sxv.nfsid.cn/551113.Doc
sxv.nfsid.cn/802606.Doc
sxv.nfsid.cn/826482.Doc
