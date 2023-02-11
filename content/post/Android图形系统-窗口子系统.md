---
layout:     post

title:      "Android图形系统-窗口子系统"
subtitle:   "窗口管理"
excerpt: ""
author:     "iigeoywq button"
date:       2023-02-04
published: true 
tags:
    - Istio 

categories: [ Tech ]
URL: "/2022/02/04/"
---

> 对于图形系统而言，首先要有一个窗口，接下来在窗口中拿到buffer去渲染，最后合成图层并送显，本片主要讲一下，如何获取一个窗口，当然Android上的窗口是一个抽象概念。


## 一 窗口管理架构
Android中的窗口是个抽象概念：本质上是一块画布，但是没有申请buffer，绘制的时候才会申请buffer。

纵向来说每层都有个对应的概念，依次为Window(抽象概念）-->WindowState(SurfaceView无)-->Surface-->Layer
注意：SurfaceView不走WMS，直接Jni调用Native Client端创建surface。

* 应用层：有三种类型

  * Phonewindow：每个Acitivity对应一个PhoneWindow，但是它是个抽象，具体的实现是DecorView，DecorView是一个Acitivity最顶层的View，在WMS测WindowManager.addView()方法添加的view就是DecorView。
  * 系统级的窗口:例如状态栏，没有PhoneWindow概念，本质上是一个view，在WMS测WindowManager.addView()方法直接添加该View。
  * SurfaceView：本质上是view，没有window概念，WMS测不需要管理，直接创建surface。

* Java Framework层：
    对应创建windowState和应用层的window对应，但是SurfaceView没有这个概念。
* Native Framework层：
  * client：Surface
  * server（SurfaceFligner）：Layer

总的来说，Android中的Window对应的是native client测的Surface，在应用层测有不同的实现方式。但是Native测实现方式相同。


## 二 源码分析
以Activity中的surfce创建来分析：

首先系统会为应用创建一个进程，并最终通过ActivityThread创建自己的Activity,调用到它的performLaunchActivity方法 
(篇幅所限进入attach方法前的流程请自行阅读源码)

frameworks/base/core/java/android/app/ActivityThread.java
```
/**  Core implementation of activity launch. */
    private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
        ...
        //1.创建Activity的Context
        ContextImpl appContext = createBaseContextForActivity(r);
        try {
            ...
            //2.反射创建Activity对象
            activity = mInstrumentation.newActivity(
                    cl, component.getClassName(), r.intent);
            ...
        } catch (Exception e) {
            ...
        }

        try {
            if (activity != null) {
                ...
                //3.执行Activity的attach动作
                activity.attach(...);
                ...
                // 4.执行应用Activity的onCreate生命周期函数,并在setContentView调用中创建DecorView对象
                mInstrumentation.callActivityOnCreate(...);
                ...
            }

        } catch (SuperNotCalledException e) {
            ...
        } catch (Exception e) {
            ...
        }
        ...
    }
```
frameworks/base/core/java/android/app/Activity.java
```
@UnsupportedAppUsage
    final void attach(Context context, 
        ...
        //3.1 创建表示应用窗口的PhoneWindow对象
        mWindow = new PhoneWindow(this, 
        ...
        //3.2 为PhoneWindow配置WindowManager
        mWindow.setWindowManager(
                (WindowManager)context.getSystemService(Context.WINDOW_SERVICE),
                mToken, mComponent.flattenToString(),
                (info.flags & ActivityInfo.FLAG_HARDWARE_ACCELERATED) != 0);
        ...
    }
    ```


