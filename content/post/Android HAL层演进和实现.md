---
layout:     post

title:      "Android HAL层演技和实现"
subtitle:   ""
excerpt: ""
author:     "iigeoywq button"
date:       2024-03-02
published: true 
tags:
    - 工程实战

categories: [ Tech ]
URL: "/2024/03/02"
---
## 一 Android HAL层解决了什么问题？

Android的HAL层解决了什么问题？
* HAL层向下屏蔽硬件实现，向上提供抽象接口，和所有的其他系统一样，提供硬件接口抽象层；
* 解决了Linux 开源协议问题，保护了厂商的利益；

![Android HAL层实现方案](/img/hal.png)

## 二 Android HAL层接口演进

Android HAL层接口经历了3个阶段



| 接口       | 系统版本     | 说明           | 接口定义                                                                       |
| -------- | -------- | ------------ | -------------------------------------------------------------------------- |
| 传统接口     | 4.4-7.0  | C风格接口        | https://source.android.com/reference/hal?hl=zh-cn                          |
| HIDL     | 8.0-10.0 | C++风格接口      | https://source.android.com/docs/core/architecture/hidl/code-style?hl=zh-cn |
| HAL-AIDL | 11.0-至今  | Android 风格接口 | https://source.android.com/docs/core/architecture/aidl/aidl-hals?hl=zh-cn  |



* HIDL接口为了解耦Framework和vendor，这样当更新Framework时候不用编译hal层，hal层实现可以由厂商单独编译在vendor分区更新；

* AIDL接口为了减少Android的binder通信通道；


先简单介绍一下这三种接口规范：
* 传统接口定义规范
hardware/libhardware/include/hardware/hardware.h

```
struct hw_module_t; //硬件模块名
struct hw_module_methods_t; //硬件模块方法名
struct hw_device_t;//硬件设备名称
```
* HIDL接口定义规范
hardware/interfaces/foo/1.0/IFoo.hal
```

/*
 * (License Notice)
 */

package android.hardware.foo@1.0;

import android.hardware.bar@1.0::IBar;

import IBaz;
import IFooClientCallback;

/**
 * IFoo is an interface that…
 */
interface IFoo {

    foo() generates (FooStatus result);

    powerCycle(IBar bar) generates (FooStatus result);

    /** Single line docstring. */
    baz();

    bar(IFooClientCallback clientCallback,
        IBaz baz,
        FooData data);

};
```
* HAD-AIDL接口定义规范
hardware/interfaces/foo/aidl/IFoo.aidl
```
    package my.package;

    import my.package.Baz; // defined elsewhere

    interface IFoo {
        void doFoo(Baz baz);
    }
```
## 三 Android HAL层接口和实现兼容
每个版本上hal层的新增模块要用新接口规范，但是还存在兼容旧版本问题，所以关于hal层的实现有5种实现方案。

![Hal层的五种实现方式](/img/hal五中实现.png)

## 四 Android HIDL和AIDL实战
//demo 验证中