---
title: '[Flink][4][设置Flink开发环境]'
date: 2020-11-2 19:08:53
tags:
    - 大数据
    - Flink
    - 流处理
categories:
    - Flink
---
## 第4章 设置Flink开发环境



Flink有一种适合开发时使用的执行模式：当程序的`execute()`方法被调用时，会在同一个JVM中以独立线程的方式启动一个JobManager线程和一个TaskManager线程。这样，整个Flink应用会以多线程的方式在同一个JVM进程中执行。该模式可用于**在IDE中执行Flink应用**。



由于单JVM执行模式的存在，你可以像调试其他程序一样在IDE中调试Flink应用。