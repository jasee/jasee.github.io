---
layout: post
title: 使用iTerm2的Python API
category: 工具
description: 使用iTerm2的Python API解决一个需求
tags: ["Python", "iTerm2"]
---

最近看到iTerm2提供了Python API，尝试用其解决自己一个小需求。

先描述一下需求吧。我有几批机器“孤悬海外”，与现有的环境都不连通，每批机器内部内网互通，一批机器里有一台作为管理机，管理机需要通过机房的某台机器才能连上去。偶尔需要操作这几批机器时，需要先连接到堡垒机，再连接到机房某机器，然后从该机器跳到管理机上。由于iTerm2有`Broadcast Input`功能，总体还算方便，只是每次从机房机器连接管理机的时候需要分别复制粘贴几次管理机IP，现在尝试通过Python
API解决了这个重复劳动问题，核心就是创建iTerm窗口的时候传递一个序号进去，后面用序号拿到本窗口需要连接的管理机IP，具体实现看代码：

```python
#!/usr/bin/env python3.7

import iterm2
import sys
import asyncio

SESSIONS = 4
SESSION_ORDER = "user.session_order"
# 记录各个session的屏幕输出，仅保留最后一行以供匹配
SCREENS = {}


async def tail_screen(session):
    async with session.get_screen_streamer() as streamer:
        while True:
            sc = await streamer.async_get()
            for l in range(sc.number_of_lines):
                if sc.line(l).string:
                    SCREENS[session.session_id] = sc.line(l).string


async def wait_run(session, cmd, text):
    while True:
        if session.session_id in SCREENS.keys():
            if text in SCREENS[session.session_id]:
                await session.async_send_text(cmd + "\n")
                return
        await asyncio.sleep(0.2)


# iterm2.Prompt检测远程不方便，直接检测screen输出
async def login_jump(session):
    await session.async_send_text("sj\n") # ssh jumpserver别名
    await wait_run(session, "机房机器IP", "堡垒机就绪提示")
    await wait_run(session, "sudo su -l", "xx@机房机器名")
    await wait_run(session, f"ssh $(sed -n '{await session.async_get_variable(SESSION_ORDER)}p' IPlist)", "root@机房机器名")
    await wait_run(session, "cd /tmp", "root@管理机名前缀")
    await wait_run(session, "ll", " tmp]")


async def main(connection):
    app = await iterm2.async_get_app(connection)
    await iterm2.Window.async_create(connection)
    window = app.current_terminal_window
    if window is not None:
        # 创建Windows时候已经创建了一个tab，所以还需要创建TABS-1个
        for i in range(SESSIONS-1):
            await window.async_create_tab()
    else:
        print("No current window")
        sys.exit(0)

    order = 1  # session序号，将此参数传递给不同session作为标记
    tasks = []
    tail_tasks = []
    domain = iterm2.broadcast.BroadcastDomain()
    for tab in window.tabs:
        session = tab.sessions[0]  # 每个tab只有一个session
        await session.async_set_variable(SESSION_ORDER, order)
        tail_tasks.append(asyncio.create_task(tail_screen(session)))
        tasks.append(asyncio.create_task(login_jump(session)))
        domain.add_session(session)
        order += 1
    await asyncio.wait(tasks)
    # 取消tail_tasks
    for _ in tail_tasks:
        _.cancel()
    await asyncio.wait(tail_tasks)
    # 设置broadcast input
    await iterm2.async_set_broadcast_domains(connection, [domain])

iterm2.run_until_complete(main)
```

*[参考文档]*
*[iTerm2 Python API](https://www.iterm2.com/python-api/)*
