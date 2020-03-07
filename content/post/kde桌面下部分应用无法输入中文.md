---
title: "Kde桌面下自带应用无法输入中文"
date: 2020-03-07T21:07:54+08:00
archives: "2020"
tags: [linux]
author: waitfox
---

​      这个问题纠结我快一年了。某次manjaro升级后，我的manjaro在系统自带应用上如konsole,kate,Yakuake上都不能切换输入法（目测系统自带的软件都不能），鼠标放键盘图标上提示“无输入窗口”。但是浏览器和其他软件是可以的。这个问题不是太影响使用，就忍了很久，大不了其他地方写好了再复制到kate里，但是就好像衣服上落沾了一坨黄泥，始终感觉不爽，每隔一两个月就要尝试解决一次，始终无果。

  尝试过的解决方案有下面几种：

1. 修改/etc/profile
2. 修改.xprofile
3. 修改/etc/environment
4. 更换其他中文输入法
5. 用fcitx-diagnose诊断配置

配置文件肯定是正确的，按照网上说的 Linux下输入中文的配置也检查了很多，作为一个有六七年经验的 Linux老司机，怎么可能翻车呢。搜了很多文章，死马当作活马医，一直无解。

fcitx-diagnose诊断结果如下，然而确认配置了，可为什么就是不认识一直没理解。

```bash
" 而不是 "fcitx". 请检查您是否在某个初始化文件中错误的设置了它的值.**
    **您可能会在 qt4 程序中使用 fcitx 时遇到问题.**

    **请使用您发行版提供的工具将环境变量 QT_IM_MODULE 设为 "fcitx" 或者将 `export QT_IM_MODULE=fcitx` 添加到您的 `~/.xprofile` 中. 参见 [输入法相关的环境变量: QT_IM_MODULE](http://fcitx-im.org/wiki/Input_method_related_environment_variables/zh-cn#QT_IM_MODULE).**

gtk - `${GTK_IM_MODULE}`:

" 而不是 "fcitx". 请检查您是否在某个初始化文件中错误的设置了它的值.**
    **您可能会在 gtk 程序中使用 fcitx 时遇到问题.**

    **请使用您发行版提供的工具将环境变量 GTK_IM_MODULE 设为 "fcitx" 或者将 `export GTK_IM_MODULE=fcitx` 添加到您的 `~/.xprofile` 中. 参见 [输入法相关的环境变量: GTK_IM_MODULE](http://fcitx-im.org/wiki/Input_method_related_environment_variables/zh-cn#GTK_IM_MODULE).**

```

直到今天，偶尔搜到了[这篇文章](https://bbs.archlinuxcn.org/viewtopic.php?id=10048)，出现的问题和我遇到的一模一样。原来是我的.xprofile里环境变量 XMODIFIERS、QT_IM_MODULE、GTK_IM_MODULE 值的末尾有一个回车符。也就是说，设置这些环境变量的那个文件错误地使用了 DOS / Windows 的换行符。

解决方案就很简单了，在 Vim 中打开并执行 :set ff=unix，然后保存并退出 :wq。

重新注销，解决了。

然后想起来，快一年前更新系统，结果挂了。这是唯一一次更新 manjaro滚挂了（这是manjaro软件源的一次bug 导致），修复系统后用了百度复制来的代码，竟然疏忽了。

要是没那篇文章，真不知何年何月能解决。放狗一搜，还有许许多多受害者遇到这种情况至今没有解决，甚至在manjaro官网提问也无解。所以记录下，希望更多人能看到。

