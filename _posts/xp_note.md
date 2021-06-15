---
layout: post
title:  "逆向笔记"
date:   2020-05-20 16:54:00 +0800
categories: reverse
---

逆向小技巧记录！

<!-- more -->

## 打开 App debuggable a 

adb shell #adb进入命令行模式
su #切换至超级用户
magisk resetprop ro.debuggable 1
stop;start; #一定要通过该方式重启
