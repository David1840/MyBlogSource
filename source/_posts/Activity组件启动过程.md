---
title: Activity组件启动过程
date: 2018-07-25 19:37:11
categories: 
- Android系统
tags:
- Android
- 系统
---

Activity是Android应用程序的四大组件之一，它负责管理Android应用程序的用户界面。

从应用程序的角度，我们将Activity分为两种类型：**根Activity**和**子Activity**。根Activity以快捷图标的形式显示在应用程序启动器中，它的启动过程代表了一个[Android应用程序的启动过程](https://david1840.github.io/2018/07/20/Android%E5%BA%94%E7%94%A8%E7%A8%8B%E5%BA%8F%E8%BF%9B%E7%A8%8B%E5%90%AF%E5%8A%A8/)。


## 根Activity启动过程


![](Activity组件启动过程/start_activity_process.jpg)

启动流程：

1. 点击桌面App图标，Launcher进程采用Binder IPC向system_server进程发起startActivity请求；
2. system_server进程接收到请求后，向zygote进程发送创建进程的请求；
3. Zygote进程fork出新的子进程，即App进程；
4. App进程，通过Binder IPC向sytem_server进程发起attachApplication请求；
5. system_server进程在收到请求后，进行一系列准备工作后，再通过binder IPC向App进程发送scheduleLaunchActivity请求；
6. App进程的binder线程（ApplicationThread）在收到请求后，通过handler向主线程发送LAUNCH_ACTIVITY消息；
7. 主线程在收到Message后，通过发射机制创建目标Activity，并回调Activity.onCreate()等方法。


