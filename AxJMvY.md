运敬赫平谠


【curses 文本用户界面 —— 终端中的图形界面编程】
# 本示例演示 Python curses 库的核心功能：屏幕控制、键盘输入、窗口和面板、颜色、菜单导航

import curses
import curses.textpad

# ---------- 主函数 ----------
def main(stdscr):
    """curses 程序入口——所有 curses 操作需要在包装函数中完成"""

    # ---------- 1. 初始化设置 ----------
    # 关闭键盘回显（输入字符不在屏幕上显示）
    curses.noecho()

    # 关闭行缓冲（立即读取按键，无需按回车）
    curses.cbreak()

    # 启用特殊键码解析（方向键、功能键等）
    stdscr.keypad(True)

    # 设置光标不可见（0=可见, 1=非常可见, 2=完全不可见）
    curses.curs_set(0)

    # ---------- 2. 颜色支持 ----------
    # 检查终端是否支持颜色
    if curses.has_colors():
        curses.start_color()
        # 初始化颜色对：数字索引, 前景色, 背景色
        curses.init_pair(1, curses.COLOR_CYAN, curses.COLOR_BLACK)     # 标题
        curses.init_pair(2, curses.COLOR_GREEN, curses.COLOR_BLACK)    # 选中项
        curses.init_pair(3, curses.COLOR_YELLOW, curses.COLOR_BLACK)   # 提示
        curses.init_pair(4, curses.COLOR_RED, curses.COLOR_BLACK)      # 警告
        curses.init_pair(5, curses.COLOR_WHITE, curses.COLOR_BLUE)     # 高亮
        curses.init_pair(6, curses.COLOR_BLACK, curses.COLOR_WHITE)    # 反色

    # 获取颜色对（方便调用）
    COLOR_TITLE = curses.color_pair(1) | curses.A_BOLD
    COLOR_SELECTED = curses.color_pair(2) | curses.A_BOLD
    COLOR_HINT = curses.color_pair(3)
    COLOR_WARN = curses.color_pair(4) | curses.A_BOLD
    COLOR_HIGHLIGHT = curses.color_pair(5)
    COLOR_INVERSE = curses.color_pair(6)

    # ---------- 3. 获取终端尺寸 ----------
    max_y, max_x = stdscr.getmaxyx()

    # ---------- 4. 创建子窗口（窗口和面板演示）----------
    # 标题窗口
    title_win = curses.newwin(3, max_x, 0, 0)

    # 菜单窗口（左侧导航）
    menu_win = curses.newwin(max_y - 6, 20, 3, 0)

    # 内容窗口（右侧主区域）
    content_win = curses.newwin(max_y - 6, max_x - 21, 3, 20)

    # 状态栏窗口（底部）
    status_win = curses.newwin(3, max_x, max_y - 3, 0)

    # ---------- 5. 定义菜单数据 ----------
    menu_items = [
        ("首页", "欢迎使用 curses 文本界面\n\ncurses 是 Unix/Linux 下\n开发 TUI 的标准库。"),
        ("文件管理", "文件管理功能演示区域\n\n- 新建文件\n- 打开文件\n- 保存文件"),
        ("系统设置", "系统设置项\n\n1. 显示设置\n2. 声音设置\n3. 网络设置"),
        ("帮助", "操作指南\n\n↑/↓ 移动菜单\nEnter 选中\nq 退出程序"),
    ]
    current_idx = 0  # 当前选中的菜单索引

    # ---------- 6. 绘制函数 ----------
    def draw_title():
        """绘制标题栏"""
        title_win.clear()
        title_win.bkgd(curses.color_pair(5))
        title_text = " curses 文本用户界面演示 "
        x_center = max_x // 2 - len(title_text) // 2
        title_win.addstr(1, x_center, title_text, COLOR_INVERSE)
        title_win.noutrefresh()

    def draw_menu():
        """绘制菜单列表"""
        menu_win.clear()
        menu_win.box()

        for idx, (item_name, _) in enumerate(menu_items):
            if idx == current_idx:
                # 当前选中项使用高亮颜色
                menu_win.addstr(2 + idx * 2, 2, f"> {item_name}",
                                COLOR_SELECTED)
            else:
                menu_win.addstr(2 + idx * 2, 2, f"  {item_name}",
                                COLOR_TITLE)

        menu_win.noutrefresh()

    def draw_content():
        """绘制内容区域"""
        content_win.clear()
        content_win.box()

        # 获取当前选中菜单的内容
        _, content = menu_items[current_idx]
        lines = content.split("\n")
        for i, line in enumerate(lines):
            if i == 0:
                # 第一行作为内容标题
                content_win.addstr(1, 2, line,
                                   COLOR_TITLE if current_idx != current_idx
                                   else COLOR_SELECTED)
            else:
                content_win.addstr(3 + i - 1, 4, line)

        content_win.noutrefresh()

    def draw_status():
        """绘制状态栏"""
        status_win.clear()
        status_win.bkgd(curses.color_pair(5))
        status_text = (f" 选中: {menu_items[current_idx][0]} "
                       f"| 终端: {max_y}x{max_x} "
                       f"| 使用方向键导航 | q 退出")
        status_win.addstr(1, 2, status_text, COLOR_INVERSE)
        status_win.noutrefresh()

    # ---------- 7. 初始绘制 ----------
    draw_title()
    draw_menu()
    draw_content()
    draw_status()
    # doupdate() 一次性刷新所有窗口（比单独 refresh 更高效）
    curses.doupdate()

    # ---------- 8. 主事件循环 ----------
    while True:
        # 获取按键输入（阻塞等待）
        key = stdscr.getch()

        # ---- 菜单导航 ----
        if key == curses.KEY_UP:
            current_idx = (current_idx - 1) % len(menu_items)
        elif key == curses.KEY_DOWN:
            current_idx = (current_idx + 1) % len(menu_items)
        elif key == curses.KEY_ENTER or key in (10, 13):
            # 回车键选中当前项目（显示确认消息）
            content_win.addstr(8, 4, f">>> 已选中: {menu_items[current_idx][0]} <<<",
                               COLOR_WARN)
            content_win.noutrefresh()
            curses.doupdate()
            curses.napms(500)  # 暂停 500ms
        elif key == ord("q") or key == ord("Q"):
            # 退出程序
            break
        elif key == curses.KEY_RESIZE:
            # 终端尺寸变化时重新获取
            max_y, max_x = stdscr.getmaxyx()
            # 重新调整窗口大小
            title_win.resize(3, max_x)
            menu_win.resize(max_y - 6, 20)
            content_win.resize(max_y - 6, max_x - 21)
            status_win.resize(3, max_x)
            # 移动窗口位置
            content_win.mvwin(3, 20)
            status_win.mvwin(max_y - 3, 0)

        # ---- 重绘所有窗口 ----
        draw_title()
        draw_menu()
        draw_content()
        draw_status()
        curses.doupdate()

    # ---------- 9. 恢复终端设置 ----------
    curses.nocbreak()
    stdscr.keypad(False)
    curses.echo()
    curses.curs_set(1)
    curses.endwin()

# ---------- 包装器入口 ----------
if __name__ == "__main__":
    # curses.wrapper() 负责初始化、异常处理和终端恢复
    curses.wrapper(main)

俚碧掠缚毡厍冻冠扰沿骋毫尚材啃

https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/MCCgjp.md
https://github.com/lippiling/oiwosr/blob/main/aStVlQ.md
https://github.com/lippiling/oiwosr/blob/main/qEdIGW.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/MCCgjp.md
https://github.com/lippiling/oiwosr/blob/main/aStVlQ.md
https://github.com/lippiling/oiwosr/blob/main/qEdIGW.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/MCCgjp.md
https://github.com/lippiling/oiwosr/blob/main/aStVlQ.md
https://github.com/lippiling/oiwosr/blob/main/qEdIGW.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/MCCgjp.md
https://github.com/lippiling/oiwosr/blob/main/aStVlQ.md
https://github.com/lippiling/oiwosr/blob/main/qEdIGW.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/MCCgjp.md
https://github.com/lippiling/oiwosr/blob/main/aStVlQ.md
https://github.com/lippiling/oiwosr/blob/main/qEdIGW.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/MCCgjp.md
https://github.com/lippiling/oiwosr/blob/main/aStVlQ.md
https://github.com/lippiling/oiwosr/blob/main/qEdIGW.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/MCCgjp.md
https://github.com/lippiling/oiwosr/blob/main/aStVlQ.md
https://github.com/lippiling/oiwosr/blob/main/qEdIGW.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/MCCgjp.md
https://github.com/lippiling/oiwosr/blob/main/aStVlQ.md
https://github.com/lippiling/oiwosr/blob/main/qEdIGW.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/MCCgjp.md
https://github.com/lippiling/oiwosr/blob/main/aStVlQ.md
https://github.com/lippiling/oiwosr/blob/main/qEdIGW.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/MCCgjp.md
https://github.com/lippiling/oiwosr/blob/main/aStVlQ.md
https://github.com/lippiling/oiwosr/blob/main/qEdIGW.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/MCCgjp.md
https://github.com/lippiling/oiwosr/blob/main/aStVlQ.md
https://github.com/lippiling/oiwosr/blob/main/qEdIGW.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/MCCgjp.md
https://github.com/lippiling/oiwosr/blob/main/aStVlQ.md
https://github.com/lippiling/oiwosr/blob/main/qEdIGW.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/MCCgjp.md
https://github.com/lippiling/oiwosr/blob/main/aStVlQ.md
https://github.com/lippiling/oiwosr/blob/main/qEdIGW.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/MCCgjp.md
https://github.com/lippiling/oiwosr/blob/main/aStVlQ.md
https://github.com/lippiling/oiwosr/blob/main/qEdIGW.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/MCCgjp.md
https://github.com/lippiling/oiwosr/blob/main/aStVlQ.md
https://github.com/lippiling/oiwosr/blob/main/qEdIGW.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/MCCgjp.md
https://github.com/lippiling/oiwosr/blob/main/aStVlQ.md
https://github.com/lippiling/oiwosr/blob/main/qEdIGW.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/MCCgjp.md
https://github.com/lippiling/oiwosr/blob/main/aStVlQ.md
https://github.com/lippiling/oiwosr/blob/main/qEdIGW.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/MCCgjp.md
https://github.com/lippiling/oiwosr/blob/main/aStVlQ.md
https://github.com/lippiling/oiwosr/blob/main/qEdIGW.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/MCCgjp.md
https://github.com/lippiling/oiwosr/blob/main/aStVlQ.md
https://github.com/lippiling/oiwosr/blob/main/qEdIGW.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/MCCgjp.md
https://github.com/lippiling/oiwosr/blob/main/aStVlQ.md
https://github.com/lippiling/oiwosr/blob/main/qEdIGW.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/MCCgjp.md
https://github.com/lippiling/oiwosr/blob/main/aStVlQ.md
https://github.com/lippiling/oiwosr/blob/main/qEdIGW.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/MCCgjp.md
https://github.com/lippiling/oiwosr/blob/main/aStVlQ.md
https://github.com/lippiling/oiwosr/blob/main/qEdIGW.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/MCCgjp.md
https://github.com/lippiling/oiwosr/blob/main/aStVlQ.md
https://github.com/lippiling/oiwosr/blob/main/qEdIGW.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/MCCgjp.md
https://github.com/lippiling/oiwosr/blob/main/aStVlQ.md
https://github.com/lippiling/oiwosr/blob/main/qEdIGW.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/MCCgjp.md
https://github.com/lippiling/oiwosr/blob/main/aStVlQ.md
https://github.com/lippiling/oiwosr/blob/main/qEdIGW.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/MCCgjp.md
https://github.com/lippiling/oiwosr/blob/main/aStVlQ.md
https://github.com/lippiling/oiwosr/blob/main/qEdIGW.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/MCCgjp.md
https://github.com/lippiling/oiwosr/blob/main/aStVlQ.md
https://github.com/lippiling/oiwosr/blob/main/qEdIGW.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/MCCgjp.md
https://github.com/lippiling/oiwosr/blob/main/aStVlQ.md
https://github.com/lippiling/oiwosr/blob/main/qEdIGW.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/MCCgjp.md
https://github.com/lippiling/oiwosr/blob/main/aStVlQ.md
https://github.com/lippiling/oiwosr/blob/main/qEdIGW.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/MCCgjp.md
https://github.com/lippiling/oiwosr/blob/main/aStVlQ.md
https://github.com/lippiling/oiwosr/blob/main/qEdIGW.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/MCCgjp.md
https://github.com/lippiling/oiwosr/blob/main/aStVlQ.md
https://github.com/lippiling/oiwosr/blob/main/qEdIGW.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/MCCgjp.md
https://github.com/lippiling/oiwosr/blob/main/aStVlQ.md
https://github.com/lippiling/oiwosr/blob/main/qEdIGW.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
https://github.com/lippiling/oiwosr/blob/main/PhscnU.md
https://github.com/lippiling/oiwosr/blob/main/hpUxno.md
https://github.com/lippiling/oiwosr/blob/main/XRbFeO.md
https://github.com/lippiling/oiwosr/blob/main/MCCgjp.md
https://github.com/lippiling/oiwosr/blob/main/aStVlQ.md
https://github.com/lippiling/oiwosr/blob/main/qEdIGW.md
https://github.com/lippiling/oiwosr/blob/main/QipRQd.md
https://github.com/lippiling/oiwosr/blob/main/GPsCnq.md
https://github.com/lippiling/oiwosr/blob/main/pXBzfr.md
https://github.com/lippiling/oiwosr/blob/main/AVnhTG.md
https://github.com/lippiling/oiwosr/blob/main/NlUHDh.md
https://github.com/lippiling/oiwosr/blob/main/DiXIuC.md
