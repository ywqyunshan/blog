---
layout:     post

title:      "Android图形系统-架构"
subtitle:   ""
excerpt: ""
author:     "iigeoywq button"
date:       2023-01-19
published: true 
tags:
    - Istio 

categories: [ Tech ]
URL: "/2022/01/19/"
---

> Android系统源码繁杂，庞大，网上分析文章很多，本文结合Android12 源码和一些优秀的文章整理而成。
<!--more-->

## 一 前言
### 硬件环境:
* 源码编译环境：Win11+WSL2+Ubuntu22.04
* 手机：Pixel3A
### 引用资料：
* 环境搭建：
  * [使用 WSL 在 Windows 上安装 Linux](https://learn.microsoft.com/zh-cn/windows/wsl/install)
  * [Android 系统开发系列（1）：Android 12 源代码下载、编译和刷机](https://androidperformance.com/2021/10/26/build-android-12/)
  * [写给Windows用户的AOSP构建开发环境配置指南](https://juejin.cn/post/7188078567022919739)
* 官方文档
  * https://source.android.com/docs/core/graphics?hl=zh-cn
  * [Android官方源码在线浏览仓库](https://cs.android.com/androidy)

* 努比亚技术团队的五篇经典文章
  * [Android画面显示流程分析(1)](https://www.jianshu.com/p/df46e4b39428)
  * [Android画面显示流程分析(2)](https://www.jianshu.com/p/f96ab6646ae3)
  * [Android画面显示流程分析(3)](https://www.jianshu.com/p/3c61375cc15b)
  * [Android画面显示流程分析(4)](https://www.jianshu.com/p/7a18666a43ce)
  * [Android画面显示流程分析(5)](https://www.jianshu.com/p/dcaf1eeddeb1)

* [韦东山Android显示系统专题视频教程](http://download.100ask.org/videos_tutorial/android/display/index.html)

  * https://www.cnblogs.com/liusiluandzhangkun/category/1225276.html

* 沧浪之水的几篇文章
  * [自上而下解读Android显示流程（上）](https://zhuanlan.zhihu.com/p/261169653)
  * [自上而下解读Android显示流程（中上)](https://zhuanlan.zhihu.com/p/421532503)
  * [自上而下解读Android显示流程（中下）：Graphic Buffer的共享、分配、同步](https://zhuanlan.zhihu.com/p/261914515)
  * [自上而下解读Android显示流程（下）-Display Processor的设计](https://zhuanlan.zhihu.com/p/261657281)

* [深入理解Android图形系统](https://www.51cto.com/article/717713.html)

* Linaro connect 的几篇文章
  * [DRM/KMS for Android](https://lpc.events/event/5/contributions/319/attachments/442/696/Android_DRM_KMS_Update_XDC2019.pdf)
  * [drm_hwcomposer](https://static.linaro.org/connect/yvr18/presentations/yvr18-204.pdf)
  * [Graphics Stack Update](https://static.linaro.org/connect/bkk16/Presentations/Wednesday/BKK16-315.pdf)

* [Android Q SurfaceFlinger合成](https://wizzie.top/Blog/2020/10/31/2020/201031_android_SurfaceFlinger2/)
<!--more-->

## 二 架构
### 2.1 Android图形系统学习路线：
* 从纵向来看，自上而下分为5层：应用层->Framework->Hal->Linux内核->硬件。其中Framework层有分为Java和Native。
* 从横向来看，分为窗口子系统,渲染子系统,显示子系统。其中窗口子系统有分为窗口管理器和合成管理器。
后面章节会按照窗口子系统-窗口管理器-->渲染子系统-->窗口子系统-合成管理器-->显示子系统路线梳理文章

![Android图形架构](/img/android图形架构.png)

### 2.2 模块源码

```
* 各层源码仓库地址：
* App 层:frameworks/base/core/java
* Java Framework SystemServer ：frameworks/base/services/core

* Jni：frameworks/base/core/jni/Android.bp
* Native Framework Surfaceflinger client测：
frameworks\native\libs\gui

*  Native Framework Surfaceflinger server测：
frameworks/native/services/surfaceflinger
hwserivice client frameworks\native\services\surfaceflinger\displayhardware


* 渲染库 frameworks\base\libs\hwui


* Hal层：
* google软件实现：
hardware/libhardware/modules/gralloc/
hardware/libhardware/modules/hwcomposer/


* 厂商实现：黑盒子以so方式提供，google仓库有部分开源实现
hardware/qcom/display/msm8084/libgralloc/
hardware/qcom/display/msm8084/libhwcomposer/
```

## 三 图形系统文章系列目录
1.Android图形系统-整体架构

2.[Android图形系统-窗口子系统（窗口）]()

3.Android图形系统-渲染子系统

4.Android图形系统-窗口子系统（合成）

5.Android图形系统-显示子系统

6.Android图形系统-vsync机制


## 四 Q/A
//todo








