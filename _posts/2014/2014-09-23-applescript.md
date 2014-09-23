---
layout: post
title: 两个AppleScript脚本
category: 工具
description: 两个AppleScript脚本展示基本用法。简单但强大。
tags: ["AppleScript", "Mac", "OSX"]
---

### Chrome标签页轮播

之前有人想要在监控墙的电视上轮播浏览器的各个标签，这个东西在OSX下通过AppleScript很容易实现，下面这个简单的脚本，可以实现标签页轮播，并且能提前重载下一个标签页~

```applescript
tell application "Google Chrome"
    activate
    repeat while true
        repeat with theWindow in every window
            set theTabindex to 0
            repeat with theTab in every tab of theWindow
                tell theTab to reload
                delay 10
                set theTabindex to theTabindex + 1
                set active tab index of theWindow to theTabindex
            end repeat
        end repeat
    end repeat
end tell
```

### 显示器横竖屏切换

每次进行显示器横竖屏切换都比较烦，切换到横屏还好说，切换到竖屏的时候需要在10s内确认，这时候总是点不到确认按钮。通过AppleScript模拟操作可以完成这个功能，`Accessibility Inspector`可以非常方便的获取页面元素，脚本如下：

```applescript
tell application "System Preferences"
    activate
    set the current pane to pane id "com.apple.preference.displays"
    reveal anchor "displaysDisplayTab" of pane id "com.apple.preference.displays"
end tell

tell application "System Events" to tell process "System Preferences"
    --tell window "DELL U2414H"
    tell window 1 --master desktop
        --click checkbox "Show mirroring options in the menu bar when available"
        --get UI elements
        tell pop up button "Rotation:" of tab group 1
            set P to get value
            click
            if P is equal to "Standard" then
                --keystroke (ASCII character 31) --down arrow key
                click menu item "90°" of menu 1
            else
                --keystroke (ASCII character 30) --up arrow key
                click menu item "Standard" of menu 1
            end if
            --keystroke return
        end tell
        if P is equal to "Standard" then
            delay 2
            click button "Confirm" of sheet 1
        end if
    end tell
end tell

tell application "System Preferences" to quit
```

AppleScript还是比较好玩的，在这方面，OSX应该比Windows更易用吧。
