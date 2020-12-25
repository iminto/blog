---
title: "TUI ConsoleLauncher 可定制化geek命令行桌面启动器"
date: 2020-12-25T11:24:02+08:00
tags: [闲扯淡]
---
[TUI ConsoleLauncher](https://github.com/fAndreuzzi/TUI-ConsoleLauncher)  basically transforms your Android into a terminal window, requiring you to type out commands to start apps and explore your phone's system as opposed to the familiar process of tapping on icons. It's a great way to practice or learn about Linux commands, and it has the added benefit of securing your phone against unwanted access.

常用命令：

```bash
-- 不允许别人用exit命令退出
alias add exit=echo "No"
-- 定制化界面，取消不必要元素
config -set show_session_info false
config -set show_storage_info false
config -set show_device_name false
config -set show_ram false
config -set show_network_info false
-- 优化
config -set time_size 20 --调大时间字体
config -set system_wallpaper true --显示系统壁纸
config -set fullscreen true
config -set enable_music true
-- 记日志/备忘录
note -add 明早九点加班
```

修改配置后需要restart才能生效。

结合shellcommand等，还可以自定义出各种小工具。

