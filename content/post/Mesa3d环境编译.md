---
layout:     post

title:      "Mesa3d环境编译"
subtitle:   ""
excerpt: ""
author:     "iigeoywq button"
date:       2024-08-21
published: true 
tags:
    - 工程实战

categories: [ Tech ]
URL: "/2024/08/21"
---
## 一 Mesa3d介绍

Mesa3ds 提供了OpenGL/OpengGL ES/OpenCL/Vulakn等图形API的实现，包含了如下一系列设备驱动程序，如freedreno/nouveau/tegra/radeonsi/asahi/amd/intel等。

## 二 编译

* 环境：
    * Ubuntu 22
    * GCC > 8.0.0
    * python > 3.6

* 下载
[mesa3d官方Release版本](https://archive.mesa3d.org/mesa-23.3.6.tar.xz)

* 预安装
```
# mako version>0.8.0
pip install mako 

# Flex versions >2.5.35 ,Bison version> 2.4.1
sudo apt-get install flex bison 
# mesa
sudo apt-get build-dep mesa

```

### 2.1 编译linux库
* 编译

```
meson setup builddir/ --prefix=/home/code/mesa-23.3.6/builddir/out

meson compile -C builddir/

sudo meson install -C builddir/

```
* 输出
```
xxx@xxxx:/home/code/mesa-23.3.6/builddir/out/lib/x86_64-linux-gnu$ ls -la
total 397276
drwxr-xr-x 5 root root      4096  8月 21 15:35 .
drwxr-xr-x 3 root root      4096  8月 21 15:35 ..
drwxr-xr-x 2 root root      4096  8月 21 15:35 dri
lrwxrwxrwx 1 root root        11  8月 21 15:35 libEGL.so -> libEGL.so.1
lrwxrwxrwx 1 root root        15  8月 21 15:35 libEGL.so.1 -> libEGL.so.1.0.0
-rwxr-xr-x 1 root root   1971224  8月 21 15:35 libEGL.so.1.0.0
lrwxrwxrwx 1 root root        11  8月 21 15:35 libgbm.so -> libgbm.so.1
lrwxrwxrwx 1 root root        15  8月 21 15:35 libgbm.so.1 -> libgbm.so.1.0.0
-rwxr-xr-x 1 root root    528592  8月 21 15:33 libgbm.so.1.0.0
lrwxrwxrwx 1 root root        13  8月 21 15:35 libglapi.so -> libglapi.so.0
lrwxrwxrwx 1 root root        17  8月 21 15:35 libglapi.so.0 -> libglapi.so.0.0.0
-rwxr-xr-x 1 root root    326368  8月 21 15:31 libglapi.so.0.0.0
lrwxrwxrwx 1 root root        17  8月 21 15:35 libGLESv1_CM.so -> libGLESv1_CM.so.1
lrwxrwxrwx 1 root root        21  8月 21 15:35 libGLESv1_CM.so.1 -> libGLESv1_CM.so.1.1.0
-rwxr-xr-x 1 root root     29872  8月 21 15:35 libGLESv1_CM.so.1.1.0
lrwxrwxrwx 1 root root        14  8月 21 15:35 libGLESv2.so -> libGLESv2.so.2
lrwxrwxrwx 1 root root        18  8月 21 15:35 libGLESv2.so.2 -> libGLESv2.so.2.0.0
-rwxr-xr-x 1 root root     56032  8月 21 15:35 libGLESv2.so.2.0.0
lrwxrwxrwx 1 root root        10  8月 21 15:35 libGL.so -> libGL.so.1
lrwxrwxrwx 1 root root        14  8月 21 15:35 libGL.so.1 -> libGL.so.1.2.0
-rwxr-xr-x 1 root root   3037248  8月 21 15:35 libGL.so.1.2.0
-rwxr-xr-x 1 root root  82404408  8月 21 15:33 libvulkan_intel_hasvk.so
-rwxr-xr-x 1 root root 123468616  8月 21 15:33 libvulkan_intel.so
-rwxr-xr-x 1 root root  58052120  8月 21 15:34 libvulkan_lvp.so
-rwxr-xr-x 1 root root  82866696  8月 21 15:33 libvulkan_radeon.so
lrwxrwxrwx 1 root root        17  8月 21 15:35 libxatracker.so -> libxatracker.so.2
lrwxrwxrwx 1 root root        21  8月 21 15:35 libxatracker.so.2 -> libxatracker.so.2.5.0
-rwxr-xr-x 1 root root  54022592  8月 21 15:34 libxatracker.so.2.5.0
drwxr-xr-x 2 root root      4096  8月 21 15:35 pkgconfig
drwxr-xr-x 2 root root      4096  8月 21 15:35 vdpau
```

### 2.2 交叉编译android库

* 预安装NDK环境
 [NDK官方下载网站](https://dl.google.com/android/repository/android-ndk-r27-linux.zip)
 我本地是通过android studio下载的NDK26.0.10792818

* 配置交叉编译文件
```
# 新建交叉编译文件
~/.local/share/meson/cross/android-aarch64

## 内容如下：
[binaries]
ar = '/home/Android/Sdk/ndk/26.0.10792818/toolchains/llvm/prebuilt/linux-x86_64/bin/llvm-ar'
c = ['ccache', '/home/Android/Sdk/ndk/26.0.10792818/toolchains/llvm/prebuilt/linux-x86_64/bin/aarch64-linux-android31-clang']
cpp = ['ccache', '/home/Android/Sdk/ndk/26.0.10792818/toolchains/llvm/prebuilt/linux-x86_64/bin/aarch64-linux-android31-clang++', '-fno-exceptions', '-fno-unwind-tables', '-fno-asynchronous-unwind-tables', '-static-libstdc++']
c_ld = 'lld'
cpp_ld = 'lld'
strip = '/home/Android/Sdk/ndk/26.0.10792818/toolchains/llvm/prebuilt/linux-x86_64/bin/aarch64-linux-android-strip'
# Android doesn't come with a pkg-config, but we need one for Meson to be happy not
# finding all the optional deps it looks for.  Use system pkg-config pointing at a
# directory we get to populate with any .pc files we want to add for Android
#pkg-config = '/usr/bin/pkg-config'

[host_machine]
system = 'android'
cpu_family = 'aarch64'
cpu = 'armv8'
endian = 'little'
```
* 编译
```
## Dgallium-drivers 代表opengl驱动类型，Dvulkan-drivers 代表vulkan驱动类型
meson setup build-android-aarch64 \
    --prefix=/home/code/mesa-23.3.6/build-android-aarch64/out
    --cross-file android-aarch64 \
    -Dplatforms=android \
    -Dplatform-sdk-version=31 \
    -Dandroid-stub=true \
    -Dgallium-drivers= \
    -Dvulkan-drivers=freedreno \
    -Dfreedreno-kmds=kgsl

meson compile -C build-android-aarch64

```
* 输出

```
build-android-aarch64/src/freedreno/vulkan/libvulkan_freedreno.so
```

### 2.3 aosp源码仓库用mk编译mesa3d
* 前置条件,这个很重，可以参考我之前的文章下载aosp源码，先整编通过

* 配置mesa3d库到soong编译环境
```
## 根据你要lunch的设备进入对应的目录
/home/code/Android_12_AOSP/device/google/bonito# vi device-sargo.mk

增加下面一行
# mesa
PRODUCT_SOONG_NAMESPACES += external/mesa3d

## 进入mesa3d 目录
/home/win/code/Android_12_AOSP/external/mesa3d# vi Android.mk

## 在文件开头增加下面宏
BOARD_USE_CUSTOMIZED_MESA := false
BOARD_GPU_DRIVERS := freedreno

```

* 编译
```
source build/envsetup.sh
lunch 33 (根据自己的设备，我是pixel3a)
cd external/mesa3d执行mm
```
* 输出

```
root@DESKTOP-BBMN2V1:/home/code/Android_12_AOSP/out/target/product/sargo/vendor/lib64# ls -la
total 600
drwxr-xr-x 5 root root   4096 Aug 18 12:23 .
drwxr-xr-x 4 root root   4096 Aug 17 23:01 ..
drwxr-xr-x 2 root root   4096 Aug 17 23:01 dri
drwxr-xr-x 2 root root   4096 Aug 17 23:01 egl
drwxr-xr-x 2 root root   4096 Aug 17 23:50 hw
-rwxr-xr-x 1 root root  81632 Aug 17 23:01 libdrm.so
-rwxr-xr-x 1 root root 131032 Aug 17 23:01 libdrm_intel.so
-rwxr-xr-x 1 root root  32944 Aug 17 23:01 libgbm_mesa.so
-rwxr-xr-x 1 root root 328784 Aug 17 23:01 libglapi.so
-rwxr-xr-x 1 root root  10584 Aug 18 12:23 meson.dummy.64.so
```








