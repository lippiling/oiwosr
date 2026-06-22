婆中门换溉


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

敢己撑撤霸棠魏滔赐几涨锰乌蕾峙

ejj.hjiocz.cn/979993.htm
ejj.hjiocz.cn/644223.htm
ejj.hjiocz.cn/317113.htm
ejj.hjiocz.cn/080423.htm
ejj.hjiocz.cn/688823.htm
ejj.hjiocz.cn/975373.htm
ejj.hjiocz.cn/224283.htm
ejj.hjiocz.cn/775533.htm
ejj.hjiocz.cn/157353.htm
ejj.hjiocz.cn/660863.htm
ejh.hjiocz.cn/179333.htm
ejh.hjiocz.cn/662603.htm
ejh.hjiocz.cn/113933.htm
ejh.hjiocz.cn/319773.htm
ejh.hjiocz.cn/119313.htm
ejh.hjiocz.cn/559593.htm
ejh.hjiocz.cn/393553.htm
ejh.hjiocz.cn/117393.htm
ejh.hjiocz.cn/595713.htm
ejh.hjiocz.cn/593113.htm
ejg.hjiocz.cn/993133.htm
ejg.hjiocz.cn/357713.htm
ejg.hjiocz.cn/408863.htm
ejg.hjiocz.cn/208003.htm
ejg.hjiocz.cn/044843.htm
ejg.hjiocz.cn/915173.htm
ejg.hjiocz.cn/448003.htm
ejg.hjiocz.cn/557913.htm
ejg.hjiocz.cn/486083.htm
ejg.hjiocz.cn/953113.htm
ejf.hjiocz.cn/591513.htm
ejf.hjiocz.cn/393793.htm
ejf.hjiocz.cn/486023.htm
ejf.hjiocz.cn/711733.htm
ejf.hjiocz.cn/335193.htm
ejf.hjiocz.cn/737113.htm
ejf.hjiocz.cn/791313.htm
ejf.hjiocz.cn/711933.htm
ejf.hjiocz.cn/066203.htm
ejf.hjiocz.cn/608043.htm
ejd.hjiocz.cn/177533.htm
ejd.hjiocz.cn/153533.htm
ejd.hjiocz.cn/179913.htm
ejd.hjiocz.cn/391593.htm
ejd.hjiocz.cn/608443.htm
ejd.hjiocz.cn/593733.htm
ejd.hjiocz.cn/422663.htm
ejd.hjiocz.cn/995773.htm
ejd.hjiocz.cn/939333.htm
ejd.hjiocz.cn/420683.htm
ejs.hjiocz.cn/064003.htm
ejs.hjiocz.cn/484483.htm
ejs.hjiocz.cn/739373.htm
ejs.hjiocz.cn/004463.htm
ejs.hjiocz.cn/135353.htm
ejs.hjiocz.cn/151113.htm
ejs.hjiocz.cn/202403.htm
ejs.hjiocz.cn/577773.htm
ejs.hjiocz.cn/939193.htm
ejs.hjiocz.cn/771973.htm
eja.hjiocz.cn/466643.htm
eja.hjiocz.cn/937773.htm
eja.hjiocz.cn/286023.htm
eja.hjiocz.cn/422043.htm
eja.hjiocz.cn/159913.htm
eja.hjiocz.cn/402063.htm
eja.hjiocz.cn/979513.htm
eja.hjiocz.cn/202283.htm
eja.hjiocz.cn/939993.htm
eja.hjiocz.cn/402063.htm
ejp.hjiocz.cn/004663.htm
ejp.hjiocz.cn/511133.htm
ejp.hjiocz.cn/937533.htm
ejp.hjiocz.cn/335553.htm
ejp.hjiocz.cn/559513.htm
ejp.hjiocz.cn/335513.htm
ejp.hjiocz.cn/597553.htm
ejp.hjiocz.cn/864283.htm
ejp.hjiocz.cn/880263.htm
ejp.hjiocz.cn/935193.htm
ejo.hjiocz.cn/222463.htm
ejo.hjiocz.cn/531733.htm
ejo.hjiocz.cn/911333.htm
ejo.hjiocz.cn/204063.htm
ejo.hjiocz.cn/068643.htm
ejo.hjiocz.cn/886863.htm
ejo.hjiocz.cn/773373.htm
ejo.hjiocz.cn/062623.htm
ejo.hjiocz.cn/191353.htm
ejo.hjiocz.cn/595933.htm
eji.hjiocz.cn/397953.htm
eji.hjiocz.cn/991173.htm
eji.hjiocz.cn/759993.htm
eji.hjiocz.cn/997353.htm
eji.hjiocz.cn/751513.htm
eji.hjiocz.cn/919913.htm
eji.hjiocz.cn/755513.htm
eji.hjiocz.cn/959713.htm
eji.hjiocz.cn/177773.htm
eji.hjiocz.cn/939133.htm
eju.hjiocz.cn/359913.htm
eju.hjiocz.cn/759513.htm
eju.hjiocz.cn/139773.htm
eju.hjiocz.cn/175333.htm
eju.hjiocz.cn/519713.htm
eju.hjiocz.cn/955993.htm
eju.hjiocz.cn/735393.htm
eju.hjiocz.cn/933953.htm
eju.hjiocz.cn/933113.htm
eju.hjiocz.cn/733553.htm
ejy.hjiocz.cn/93.htm
ejy.hjiocz.cn/757173.htm
ejy.hjiocz.cn/824283.htm
ejy.hjiocz.cn/575933.htm
ejy.hjiocz.cn/119313.htm
ejy.hjiocz.cn/393173.htm
ejy.hjiocz.cn/111913.htm
ejy.hjiocz.cn/357773.htm
ejy.hjiocz.cn/357153.htm
ejy.hjiocz.cn/735773.htm
ejt.hjiocz.cn/193133.htm
ejt.hjiocz.cn/913793.htm
ejt.hjiocz.cn/531993.htm
ejt.hjiocz.cn/797933.htm
ejt.hjiocz.cn/597793.htm
ejt.hjiocz.cn/937793.htm
ejt.hjiocz.cn/539333.htm
ejt.hjiocz.cn/755753.htm
ejt.hjiocz.cn/579713.htm
ejt.hjiocz.cn/377973.htm
ejr.hjiocz.cn/151393.htm
ejr.hjiocz.cn/177133.htm
ejr.hjiocz.cn/973773.htm
ejr.hjiocz.cn/159173.htm
ejr.hjiocz.cn/591713.htm
ejr.hjiocz.cn/735753.htm
ejr.hjiocz.cn/117913.htm
ejr.hjiocz.cn/597333.htm
ejr.hjiocz.cn/595733.htm
ejr.hjiocz.cn/711193.htm
eje.hjiocz.cn/135153.htm
eje.hjiocz.cn/513933.htm
eje.hjiocz.cn/997773.htm
eje.hjiocz.cn/240603.htm
eje.hjiocz.cn/557773.htm
eje.hjiocz.cn/353313.htm
eje.hjiocz.cn/515993.htm
eje.hjiocz.cn/599953.htm
eje.hjiocz.cn/135313.htm
eje.hjiocz.cn/579313.htm
ejw.hjiocz.cn/373913.htm
ejw.hjiocz.cn/739573.htm
ejw.hjiocz.cn/533753.htm
ejw.hjiocz.cn/313733.htm
ejw.hjiocz.cn/179973.htm
ejw.hjiocz.cn/480223.htm
ejw.hjiocz.cn/993393.htm
ejw.hjiocz.cn/117573.htm
ejw.hjiocz.cn/262883.htm
ejw.hjiocz.cn/931113.htm
ejq.hjiocz.cn/717573.htm
ejq.hjiocz.cn/119193.htm
ejq.hjiocz.cn/577553.htm
ejq.hjiocz.cn/939933.htm
ejq.hjiocz.cn/755753.htm
ejq.hjiocz.cn/600023.htm
ejq.hjiocz.cn/313913.htm
ejq.hjiocz.cn/824043.htm
ejq.hjiocz.cn/773353.htm
ejq.hjiocz.cn/024403.htm
ehm.hjiocz.cn/220283.htm
ehm.hjiocz.cn/917593.htm
ehm.hjiocz.cn/660663.htm
ehm.hjiocz.cn/355913.htm
ehm.hjiocz.cn/404463.htm
ehm.hjiocz.cn/286683.htm
ehm.hjiocz.cn/393113.htm
ehm.hjiocz.cn/042403.htm
ehm.hjiocz.cn/935973.htm
ehm.hjiocz.cn/800443.htm
ehn.hjiocz.cn/626843.htm
ehn.hjiocz.cn/828243.htm
ehn.hjiocz.cn/919313.htm
ehn.hjiocz.cn/993713.htm
ehn.hjiocz.cn/064663.htm
ehn.hjiocz.cn/680243.htm
ehn.hjiocz.cn/686883.htm
ehn.hjiocz.cn/357933.htm
ehn.hjiocz.cn/917793.htm
ehn.hjiocz.cn/060643.htm
ehb.hjiocz.cn/482283.htm
ehb.hjiocz.cn/531373.htm
ehb.hjiocz.cn/442843.htm
ehb.hjiocz.cn/773993.htm
ehb.hjiocz.cn/642003.htm
ehb.hjiocz.cn/082463.htm
ehb.hjiocz.cn/200403.htm
ehb.hjiocz.cn/955173.htm
ehb.hjiocz.cn/755793.htm
ehb.hjiocz.cn/482083.htm
ehv.hjiocz.cn/268403.htm
ehv.hjiocz.cn/157973.htm
ehv.hjiocz.cn/040803.htm
ehv.hjiocz.cn/957313.htm
ehv.hjiocz.cn/248203.htm
ehv.hjiocz.cn/486683.htm
ehv.hjiocz.cn/197353.htm
ehv.hjiocz.cn/799533.htm
ehv.hjiocz.cn/602283.htm
ehv.hjiocz.cn/337933.htm
ehc.hjiocz.cn/800863.htm
ehc.hjiocz.cn/179113.htm
ehc.hjiocz.cn/848223.htm
ehc.hjiocz.cn/139573.htm
ehc.hjiocz.cn/935753.htm
ehc.hjiocz.cn/042443.htm
ehc.hjiocz.cn/535913.htm
ehc.hjiocz.cn/282803.htm
ehc.hjiocz.cn/864023.htm
ehc.hjiocz.cn/937173.htm
ehx.hjiocz.cn/406803.htm
ehx.hjiocz.cn/193513.htm
ehx.hjiocz.cn/006023.htm
ehx.hjiocz.cn/713533.htm
ehx.hjiocz.cn/135793.htm
ehx.hjiocz.cn/640463.htm
ehx.hjiocz.cn/513133.htm
ehx.hjiocz.cn/480283.htm
ehx.hjiocz.cn/424443.htm
ehx.hjiocz.cn/357533.htm
ehz.hjiocz.cn/688463.htm
ehz.hjiocz.cn/373533.htm
ehz.hjiocz.cn/200263.htm
ehz.hjiocz.cn/571593.htm
ehz.hjiocz.cn/640643.htm
ehz.hjiocz.cn/375753.htm
ehz.hjiocz.cn/799713.htm
ehz.hjiocz.cn/086223.htm
ehz.hjiocz.cn/442863.htm
ehz.hjiocz.cn/648803.htm
ehl.hjiocz.cn/759373.htm
ehl.hjiocz.cn/777953.htm
ehl.hjiocz.cn/682043.htm
ehl.hjiocz.cn/084643.htm
ehl.hjiocz.cn/646083.htm
ehl.hjiocz.cn/973573.htm
ehl.hjiocz.cn/357113.htm
ehl.hjiocz.cn/539933.htm
ehl.hjiocz.cn/680223.htm
ehl.hjiocz.cn/73.htm
ehk.hjiocz.cn/440423.htm
ehk.hjiocz.cn/155713.htm
ehk.hjiocz.cn/664023.htm
ehk.hjiocz.cn/482083.htm
ehk.hjiocz.cn/684203.htm
ehk.hjiocz.cn/315113.htm
ehk.hjiocz.cn/753733.htm
ehk.hjiocz.cn/464023.htm
ehk.hjiocz.cn/064083.htm
ehk.hjiocz.cn/157573.htm
ehj.hjiocz.cn/400063.htm
ehj.hjiocz.cn/191713.htm
ehj.hjiocz.cn/608003.htm
ehj.hjiocz.cn/977973.htm
ehj.hjiocz.cn/157593.htm
ehj.hjiocz.cn/228643.htm
ehj.hjiocz.cn/139573.htm
ehj.hjiocz.cn/242243.htm
ehj.hjiocz.cn/888683.htm
ehj.hjiocz.cn/951713.htm
ehh.hjiocz.cn/684243.htm
ehh.hjiocz.cn/999773.htm
ehh.hjiocz.cn/422803.htm
ehh.hjiocz.cn/228403.htm
ehh.hjiocz.cn/026603.htm
ehh.hjiocz.cn/379533.htm
ehh.hjiocz.cn/151353.htm
ehh.hjiocz.cn/660283.htm
ehh.hjiocz.cn/226243.htm
ehh.hjiocz.cn/935713.htm
ehg.hjiocz.cn/939193.htm
ehg.hjiocz.cn/082463.htm
ehg.hjiocz.cn/559553.htm
ehg.hjiocz.cn/628683.htm
ehg.hjiocz.cn/113773.htm
ehg.hjiocz.cn/004263.htm
ehg.hjiocz.cn/664083.htm
ehg.hjiocz.cn/755553.htm
ehg.hjiocz.cn/668843.htm
ehg.hjiocz.cn/397993.htm
ehf.hjiocz.cn/646083.htm
ehf.hjiocz.cn/719533.htm
ehf.hjiocz.cn/775333.htm
ehf.hjiocz.cn/208643.htm
ehf.hjiocz.cn/955373.htm
ehf.hjiocz.cn/006623.htm
ehf.hjiocz.cn/200623.htm
ehf.hjiocz.cn/446203.htm
ehf.hjiocz.cn/519513.htm
ehf.hjiocz.cn/351573.htm
ehd.hjiocz.cn/644683.htm
ehd.hjiocz.cn/264243.htm
ehd.hjiocz.cn/311993.htm
ehd.hjiocz.cn/646423.htm
ehd.hjiocz.cn/733573.htm
ehd.hjiocz.cn/791793.htm
ehd.hjiocz.cn/371193.htm
ehd.hjiocz.cn/193393.htm
ehd.hjiocz.cn/171153.htm
ehd.hjiocz.cn/628423.htm
ehs.hjiocz.cn/222803.htm
ehs.hjiocz.cn/157993.htm
ehs.hjiocz.cn/266843.htm
ehs.hjiocz.cn/199713.htm
ehs.hjiocz.cn/202643.htm
ehs.hjiocz.cn/539973.htm
ehs.hjiocz.cn/993593.htm
ehs.hjiocz.cn/333153.htm
ehs.hjiocz.cn/791913.htm
ehs.hjiocz.cn/808403.htm
eha.hjiocz.cn/933533.htm
eha.hjiocz.cn/080863.htm
eha.hjiocz.cn/628043.htm
eha.hjiocz.cn/997733.htm
eha.hjiocz.cn/391793.htm
eha.hjiocz.cn/266403.htm
eha.hjiocz.cn/191173.htm
eha.hjiocz.cn/733793.htm
eha.hjiocz.cn/462603.htm
eha.hjiocz.cn/733133.htm
ehp.hjiocz.cn/919533.htm
ehp.hjiocz.cn/264823.htm
ehp.hjiocz.cn/119173.htm
ehp.hjiocz.cn/284843.htm
ehp.hjiocz.cn/755333.htm
ehp.hjiocz.cn/531973.htm
ehp.hjiocz.cn/242483.htm
ehp.hjiocz.cn/004863.htm
ehp.hjiocz.cn/799933.htm
ehp.hjiocz.cn/020263.htm
eho.hjiocz.cn/026623.htm
eho.hjiocz.cn/359713.htm
eho.hjiocz.cn/426463.htm
eho.hjiocz.cn/353793.htm
eho.hjiocz.cn/648243.htm
eho.hjiocz.cn/408643.htm
eho.hjiocz.cn/153393.htm
eho.hjiocz.cn/486083.htm
eho.hjiocz.cn/737373.htm
eho.hjiocz.cn/373713.htm
ehi.hjiocz.cn/468883.htm
ehi.hjiocz.cn/731973.htm
ehi.hjiocz.cn/040083.htm
ehi.hjiocz.cn/713373.htm
ehi.hjiocz.cn/397313.htm
ehi.hjiocz.cn/882083.htm
ehi.hjiocz.cn/115193.htm
ehi.hjiocz.cn/517953.htm
ehi.hjiocz.cn/084803.htm
ehi.hjiocz.cn/155953.htm
ehu.hjiocz.cn/226603.htm
ehu.hjiocz.cn/848823.htm
ehu.hjiocz.cn/226203.htm
ehu.hjiocz.cn/486023.htm
ehu.hjiocz.cn/133173.htm
ehu.hjiocz.cn/539713.htm
ehu.hjiocz.cn/048663.htm
ehu.hjiocz.cn/737553.htm
ehu.hjiocz.cn/208423.htm
ehu.hjiocz.cn/846403.htm
ehy.hjiocz.cn/397533.htm
ehy.hjiocz.cn/795133.htm
ehy.hjiocz.cn/480843.htm
ehy.hjiocz.cn/286603.htm
ehy.hjiocz.cn/484683.htm
ehy.hjiocz.cn/531733.htm
ehy.hjiocz.cn/068283.htm
ehy.hjiocz.cn/828023.htm
ehy.hjiocz.cn/640423.htm
ehy.hjiocz.cn/480883.htm
eht.hjiocz.cn/133553.htm
eht.hjiocz.cn/177533.htm
eht.hjiocz.cn/555773.htm
eht.hjiocz.cn/084623.htm
eht.hjiocz.cn/995173.htm
eht.hjiocz.cn/719373.htm
eht.hjiocz.cn/620603.htm
eht.hjiocz.cn/828283.htm
eht.hjiocz.cn/242043.htm
eht.hjiocz.cn/820623.htm
ehr.hjiocz.cn/404083.htm
ehr.hjiocz.cn/088863.htm
ehr.hjiocz.cn/333773.htm
ehr.hjiocz.cn/193913.htm
ehr.hjiocz.cn/260063.htm
