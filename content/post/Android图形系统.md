---
layout:     post

title:      "Android图形系统-架构"
subtitle:   ""
excerpt: ""
author:     "iigeoywq button"
date:       2023-01-19
published: true 
tags:
    - 系统理论 

categories: [ Tech ]
URL: "/2023/01/19/"
---

> Android系统源码繁杂，庞大，网上分析文章很多，本文结合Android11 源码和一些优秀的文章整理而成。
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
* [Android 重学系列 SurfaceFlinger的概述](https://www.jianshu.com/p/c954bcceb22a)
* [AndroidQ 图形系统（11）UI刷新，SurfaceFlinger，Vsync机制总结](https://blog.csdn.net/qq_34211365/article/details/107996767?spm=1001.2014.3001.5501)
* [Android图形系统系统篇之HWC](https://www.zybuluo.com/ltlovezh/note/1547782)
* [显示图形系统分析之默认DisplayDevice显示设备加载流程分析](https://juejin.cn/post/7056139802529234958)
* [Android中的GraphicBuffer同步机制-Fence](https://www.cnblogs.com/brucemengbm/p/6881925.html)
* [专题分纲目录 Android GUI系统之SurfaceFlinger](https://blog.csdn.net/vviccc/article/details/104860616)
* [内存管理 —— ION](https://kernel.meizu.com/memory%20management%20-%20ion.html)
<!--more-->

## 二 架构
### 2.1 Android图形系统学习路线：
* 从纵向来看，自上而下分为5层：应用层->Framework->Hal->Linux内核->硬件。其中Framework层有分为Java和Native。
* 从横向来看，分为窗口子系统,渲染子系统,显示子系统。其中窗口子系统有分为窗口管理器和合成管理器。
后面章节会按照窗口子系统-窗口管理器-->渲染子系统-->窗口子系统-合成管理器-->显示子系统路线梳理文章

### 2.2 整体架构
* 1.首先app层和Java Framework层和NatvieFramework层连接，创建一个窗口；
* 2.然后App进程收到vsync-app信号后，向NativeFramework层申请buffer，NativeFramework收到请求后，继续向hal层申请共享内存，最后通过层层回传句柄给应用进程；
* 3.接着，应用层构建Display list数据并通过厂商提供的opengl库走gpu绘制；
* 4.绘制完成后，提交buffer；
* 5.接着，sf进程收到vsync-sf信号后，检查layer，并创建hwc layer，并根据hwc layer的合成方式，先gpu合成，后hwc合成，最后送显。


![Android图形架构](/img/android%E5%9B%BE%E5%BD%A2%E6%9E%B6%E6%9E%84.png)

### 2.2 模块源码

<style type="text/css">
.tg  {border-collapse:collapse;border-spacing:0;}
.tg td{border-color:black;border-style:solid;border-width:1px;font-family:Arial, sans-serif;font-size:14px;
  overflow:hidden;padding:10px 5px;word-break:normal;}
.tg th{border-color:black;border-style:solid;border-width:1px;font-family:Arial, sans-serif;font-size:14px;
  font-weight:normal;overflow:hidden;padding:10px 5px;word-break:normal;}
.tg .tg-lboi{border-color:inherit;text-align:left;vertical-align:middle}
.tg .tg-uzvj{border-color:inherit;font-weight:bold;text-align:center;vertical-align:middle}
.tg .tg-7btt{border-color:inherit;font-weight:bold;text-align:center;vertical-align:top}
.tg .tg-0pky{border-color:inherit;text-align:left;vertical-align:top}
.tg .tg-0lax{text-align:left;vertical-align:top}
</style>
<table class="tg" style="undefined;table-layout: fixed; width: 1123px">
<colgroup>
<col style="width: 100.333333px">
<col style="width: 116.333333px">
<col style="width: 283.333333px">
<col style="width: 186.333333px">
<col style="width: 162.333333px">
<col style="width: 154.333333px">
<col style="width: 120.333333px">
</colgroup>
<thead>
  <tr>
    <th class="tg-uzvj" colspan="2" rowspan="2">模块</th>
    <th class="tg-7btt" colspan="2">源码路径</th>
    <th class="tg-7btt" colspan="2">类库</th>
    <th class="tg-uzvj" rowspan="2">备注</th>
  </tr>
  <tr>
    <th class="tg-uzvj">Android实现</th>
    <th class="tg-uzvj">厂商（高通）实现</th>
    <th class="tg-uzvj">Android</th>
    <th class="tg-uzvj">厂商（高通）</th>
  </tr>
</thead>
<tbody>
  <tr>
    <td class="tg-0pky" colspan="2">APP层</td>
    <td class="tg-0pky" colspan="2"></td>
    <td class="tg-0pky" colspan="2">APK</td>
    <td class="tg-0pky"></td>
  </tr>
  <tr>
    <td class="tg-lboi" rowspan="3">Java Framework层</td>
    <td class="tg-0pky">Client</td>
    <td class="tg-0pky" colspan="2">frameworks/base/core/</td>
    <td class="tg-0pky" colspan="2">system/framework/framework.jar</td>
    <td class="tg-0pky"></td>
  </tr>
  <tr>
    <td class="tg-0pky" rowspan="2">SystemServer</td>
    <td class="tg-0pky" colspan="2" rowspan="2">frameworks/base/services/core</td>
    <td class="tg-0pky" colspan="2" rowspan="2">system/framework/service.jar</td>
    <td class="tg-0pky"></td>
  </tr>
  <tr>
    <td class="tg-0pky"></td>
  </tr>
  <tr>
    <td class="tg-lboi" rowspan="6">Native Framework层<br></td>
    <td class="tg-0pky">Jni</td>
    <td class="tg-0pky" colspan="2">frameworks/base/core/jni<br></td>
    <td class="tg-0pky" colspan="2">/system/lib64/libandroid_runtime.so</td>
    <td class="tg-0pky"></td>
  </tr>
  <tr>
    <td class="tg-0lax">hwui</td>
    <td class="tg-0lax" colspan="2">frameworks/base/libs/hwui</td>
    <td class="tg-0lax" colspan="2">/system/lib64/libhwui.so</td>
    <td class="tg-0lax"></td>
  </tr>
  <tr>
    <td class="tg-0lax">gui</td>
    <td class="tg-0lax" colspan="2">frameworks/native/libs/gui</td>
    <td class="tg-0lax" colspan="2">/system/lib64/libgui.so</td>
    <td class="tg-0lax"></td>
  </tr>
  <tr>
    <td class="tg-0lax">skia</td>
    <td class="tg-0lax" colspan="2">external/skia/</td>
    <td class="tg-0lax" colspan="2">libskia.so</td>
    <td class="tg-0lax"></td>
  </tr>
  <tr>
    <td class="tg-0pky">opengl</td>
    <td class="tg-0pky" colspan="2">frameworks/native/opengl/libs/(EGL wrapper)<br><br>实现库：厂商实现</td>
    <td class="tg-0pky" colspan="2">/system/lib64/egl/</td>
    <td class="tg-0pky"></td>
  </tr>
  <tr>
    <td class="tg-0pky">Server-SurfaceFilnger</td>
    <td class="tg-0pky" colspan="2">frameworks/native/services/surfaceflinger<br></td>
    <td class="tg-0pky" colspan="2" rowspan="2">/system/bin/surfaceflinger</td>
    <td class="tg-0pky"></td>
  </tr>
  <tr>
    <td class="tg-0pky" rowspan="5">Hal层</td>
    <td class="tg-0pky">Client</td>
    <td class="tg-0pky" colspan="2">/frameworks/native/services/surfaceflinger/DisplayHardware<br><br></td>
    <td class="tg-0pky"></td>
  </tr>
  <tr>
    <td class="tg-0pky">Server-hal</td>
    <td class="tg-0pky" colspan="2">hardware/interfaces/graphics/</td>
    <td class="tg-0pky" colspan="2">vendor/lib64/android.hardware.graphics.xxx@2.1.so<br>vendor/lib64/hw/android.hardware.graphics.xxx@2.1-impl.so<br>vendor/bin/hw/android.hardware.graphics.xxx@2.1-service<br></td>
    <td class="tg-0pky"></td>
  </tr>
  <tr>
    <td class="tg-0pky">hal实现</td>
    <td class="tg-0pky">hardware/libhardware/modules<br><br>external/drm_hwcomposer/（HWC2 三方开源实现）</td>
    <td class="tg-0pky">hardware/qcom/sdm845/<br>display/sdm<br></td>
    <td class="tg-0pky">xxx.drm.so<br>(hwc2开源实现)<br><br>xxx.default.so</td>
    <td class="tg-0pky">vendor/lib64/hw/<br>xxx.sdmxxx.so<br></td>
    <td class="tg-0pky">高通的so都是闭源的，高通只开放了部分源码仅供参考，以高通官方文档为准</td>
  </tr>
  <tr>
    <td class="tg-0pky">drm接口层</td>
    <td class="tg-0pky" colspan="2">external/libdrm/libkms/</td>
    <td class="tg-0pky" colspan="2">vendor/lib64/libdrm.so</td>
    <td class="tg-0pky"></td>
  </tr>
  <tr>
    <td class="tg-0lax">共享内存</td>
    <td class="tg-0lax" colspan="2">system/memory/libion/</td>
    <td class="tg-0lax" colspan="2">/system/lib64/libion.so</td>
    <td class="tg-0lax"></td>
  </tr>
  <tr>
    <td class="tg-0pky">内核层</td>
    <td class="tg-0pky">DRM-KMS</td>
    <td class="tg-0pky">待确定</td>
    <td class="tg-0pky">common/drivers/gpu/drm/msm/</td>
    <td class="tg-0pky">待确定</td>
    <td class="tg-0pky">待确定</td>
    <td class="tg-0pky"></td>
  </tr>
</tbody>
</table>
更详细的源码目录可阅读后续窗口和渲染子系统系列目录。

## 三 图形系统文章系列目录
1.[Android图形系统-整体架构]()

2.[Android图形系统-窗口子系统-窗口](https://ywqyunshan.github.io/2023/02/04/)

3.[Android图形系统-渲染子系统](https://ywqyunshan.github.io/2023/02/22/)

4.[Android图形系统-窗口子系统-合成](https://ywqyunshan.github.io/2023/03/03/)

5.Android图形系统-显示子系统

6.Android图形系统-vsync机制









