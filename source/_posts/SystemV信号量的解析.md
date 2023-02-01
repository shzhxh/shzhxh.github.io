---
layout: post
title:  "SystemV信号量的解析"
date:   2021-12-04 18:38:00 +0800
categories: Linux
---

glibc提供了两种信号量。POSIX信号量和SystemV信号量。

SystemV信号量是在内核里的，向用户暴露为信号量的集合。

控制信号量的三个函数都是系统调用。semget, semctl, semop。
