title: Windows添加网络打印机脚本
date: 2017-09-11T06:50:38.051Z
tags: [Windows,打印机]
categories: [Blog]
---
# Windows下通过脚本添加网络打印机
最近在内网搭了一台打印服务器，需要给实验室的同学共享，但是添加打印机的操作太繁琐，所以编写了一个添加打印机的脚本，可以代替人工操作过程。

<!-- more -->

```
@echo off
echo 添加网络打印机
echo 等待....
net use \\{IP}\IPC$ "{密码}" /user:"{用户名}"
rundll32 printui.dll,PrintUIEntry /in /u /z /q /n "\\{IP}\{打印机名}"
rundll32 printui.dll,PrintUIEntry /y /n "\\{IP}\{打印机名}"
echo 打印机添加成功
pause
```

1. {IP}替换为打印机的IP
2. {密码}替换为打印服务器的用户密码
3. {用户名}替换为打印服务器的用户名
4. {打印机名}替换为打印机的名称，可以在打印服务器中进行设置