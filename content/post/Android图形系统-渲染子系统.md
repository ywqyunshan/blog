---
layout:     post

title:      "Android图形系统-渲染子系统"
subtitle:   ""
excerpt: ""
author:     "iigeoywq button"
date:       2023-02-22
published: true 
tags:
    - Istio 

categories: [ Tech ]
URL: "/2023/02/22/"
---
> 渲染主要分申请buffer，cpu/gpu渲染和提交buffer。
本篇主要讲一下buffer的申请，以及渲染，提交buffer。

## 一 渲染子系统架构
本篇主要讲解app进程如何申请buffer和提交buffer，渲染只涉及renderthread线程构建displaylist，至于真正的gpu渲染，属于另一个学科，只讲一下opengl api的调用，不深入研究。架构图见[Android图形系统-整体架构]()渲染部分。
## 二 源码目录
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

## 三 源码分析

### 3.1 java层的Surface传入native层，并转换为ANativeWindow和EglSurface对象
先做个准备工作，收到vsync-app通知后，在[Android图形系统-窗口子系统]()篇3.2.2的第2个步骤，将app进程和wms进程共享的surface传入
```
/*frameworks/base/core/java/android/view/ThreadedRenderer.java*/
boolean initialize(Surface surface) throws OutOfResourcesException {
    ......
    setSurface(surface);
    ......
}

/*frameworks/base/libs/hwui/jni/android_graphics_HardwareRenderer.cpp*/
static void android_view_ThreadedRenderer_setSurface(JNIEnv* env, jobject clazz,
        jlong proxyPtr, jobject jsurface, jboolean discardBuffer) {
    RenderProxy* proxy = reinterpret_cast<RenderProxy*>(proxyPtr);
    ......
    proxy->setSurface(window, enableTimeout);
    ......
}

/*frameworks/base/libs/hwui/renderthread/RenderProxy.cpp*/
void RenderProxy::setSurface(ANativeWindow* window, bool enableTimeout) {
    ANativeWindow_acquire(window);
    mRenderThread.queue().post([this, win = window, enableTimeout]() mutable {
        mContext->setSurface(win, enableTimeout);
        ANativeWindow_release(win);
    });
}

/*frameworks/base/libs/hwui/renderthread/CanvasContext.cpp*/
void CanvasContext::setSurface(ANativeWindow* window, bool enableTimeout) {
    ATRACE_CALL();

    if (window) {
        mNativeSurface = std::make_unique<ReliableSurface>(window);
        mNativeSurface->init();
        ....
    } else {
        mNativeSurface = nullptr;
    }
    setupPipelineSurface();
}

void CanvasContext::setupPipelineSurface() {
    bool hasSurface = mRenderPipeline->setSurface(
            mNativeSurface ? mNativeSurface->getNativeWindow() : nullptr, mSwapBehavior);
    ....
}

/*frameworks/base/libs/hwui/pipeline/skia/SkiaOpenGLPipeline.cpp*/
bool SkiaOpenGLPipeline::setSurface(ANativeWindow* surface, SwapBehavior swapBehavior) {
    if (mEglSurface != EGL_NO_SURFACE) {
        mEglManager.destroySurface(mEglSurface);
        mEglSurface = EGL_NO_SURFACE;
    }

    if (surface) {
        mRenderThread.requireGlContext();
        auto newSurface = mEglManager.createSurface(surface, mColorMode, mSurfaceColorSpace);
        if (!newSurface) {
            return false;
        }
        //
        mEglSurface = newSurface.unwrap();
    }
    ...
    return false;
}
```


### 3.2 主线程构建DisplayList
收到vsync-app信号后，接[Android图形系统-窗口子系统]()篇3.2.2的第3个步骤,来到ViewRootImpl的draw方法，默认是打开硬件加速的，进入ThreadedRenderer（注意这里还是主线程）构建DisplayList。

```
/*frameworks/base/core/java/android/view/ViewRootImpl.java*/
private boolean draw(boolean fullRedrawNeeded) {
    ...
    if (mAttachInfo.mThreadedRenderer != null && mAttachInfo.mThreadedRenderer.isEnabled()) {
        ...
        // 走gpu绘制的流程
        mAttachInfo.mThreadedRenderer.draw(mView, mAttachInfo, this);
    } else {
        // 否则走软件绘制的流程
        if (!drawSoftware(surface, mAttachInfo, xOffset, yOffset,
                        scalingRequired, dirty, surfaceInsets)) {
                    return false;
         }
    }
}
```
#### 3.2.1 ThreadedRenderer类构建DisplayList
```
/*frameworks/base/core/java/android/view/ThreadedRenderer.java*/

void draw(View view, AttachInfo attachInfo, DrawCallbacks callbacks) {
    ...
    // 1.从DecorView根节点出发，递归遍历View控件树，记录每个View节点的绘制操作命令，完成绘制操作命令树的构建
    updateRootDisplayList(view, callbacks);
    ...
    // 2.JNI调用同步Java层构建的绘制命令树到Native层的RenderThread渲染线程，并唤醒渲染线程利用OpenGL执行渲染任务；
    int syncResult = syncAndDrawFrame(choreographer.mFrameInfo);
    ...
}

private void updateRootDisplayList(View view, DrawCallbacks callbacks) {
    ......
    updateViewTreeDisplayList(view);
    ......
}
private void updateViewTreeDisplayList(View view) {
    ......
    //这里开始调用DecorView的updateDisplayListIfDirty
    view.updateDisplayListIfDirty();
    ......
}

/*frameworks/base/core/java/android/view/View.java*/
public RenderNode updateDisplayListIfDirty() {
     ...
     // 1.利用`View`对象构造时创建的`RenderNode`获取一个`SkiaRecordingCanvas`“画布”；
     final RecordingCanvas canvas = renderNode.beginRecording(width, height);
     try {
         ...
         if ((mPrivateFlags & PFLAG_SKIP_DRAW) == PFLAG_SKIP_DRAW) {
              // 如果仅仅是ViewGroup，并且自身不用绘制，直接递归子View
              dispatchDraw(canvas);
              ...
         } else {
              // 2.利用SkiaRecordingCanvas，在每个子View控件的onDraw绘制函数中调用drawLine、drawRect等绘制操作时，创建对应的DisplayListOp绘制命令，并缓存记录到其内部的SkiaDisplayList持有的DisplayListData中；
              draw(canvas);
         }
     } finally {
         // 3.将包含有`DisplayListOp`绘制命令缓存的`SkiaDisplayList`对象设置填充到`RenderNode`中；
         renderNode.endRecording();
         ...
     }
     ...
}

public void draw(Canvas canvas) {
    ...
    // draw the content(View自己实现的onDraw绘制，由应用开发者自己实现)
    onDraw(canvas);
    ...
    // draw the children
    dispatchDraw(canvas);
    ...
}

/*frameworks/base/graphics/java/android/graphics/RenderNode.java*/
public void endRecording() {
        ...
        // 从SkiaRecordingCanvas中获取SkiaDisplayList对象
        long displayList = canvas.finishRecording();
        // 将SkiaDisplayList对象填充到RenderNode中
        nSetDisplayList(mNativeRenderNode, displayList);
        canvas.recycle();
}
```
### 3.3 renderthread线程渲染
接3.2.1 第二步来到HardwareRenderer的syncAndDrawFrame方法，并jni调用进入hwui库。
```
/*frameworks/base/graphics/java/android/graphics/HardwareRenderer.java*/
public int syncAndDrawFrame(@NonNull FrameInfo frameInfo) {
    // JNI调用native层的相关函数
    return nSyncAndDrawFrame(mNativeProxy, frameInfo.frameInfo, frameInfo.frameInfo.length);
}

/*frameworks/base/libs/hwui/jni/android_graphics_HardwareRenderer.cpp*/
static int android_view_ThreadedRenderer_syncAndDrawFrame(JNIEnv* env, jobject clazz,
        jlong proxyPtr, jlongArray frameInfo, jint frameInfoSize) {
    ...
    RenderProxy* proxy = reinterpret_cast<RenderProxy*>(proxyPtr);
    env->GetLongArrayRegion(frameInfo, 0, frameInfoSize, proxy->frameInfo());
    return proxy->syncAndDrawFrame();
}

/*frameworks/base/libs/hwui/renderthread/RenderProxy.cpp*/
int RenderProxy::syncAndDrawFrame() {
    // 唤醒RenderThread渲染线程，执行DrawFrame绘制任务
    return mDrawFrameTask.drawFrame();
}

/*frameworks/base/libs/hwui/renderthread/DrawFrameTask.cpp*/
int DrawFrameTask::drawFrame() {
    ...
    postAndWait();
    ...
}

void DrawFrameTask::postAndWait() {
    AutoMutex _lock(mLock);
    // 向RenderThread渲染线程的MessageQueue消息队列放入一个待执行任务，以将其唤醒执行run函数
    mRenderThread->queue().post([this]() { run(); });
    // UI线程暂时进入wait等待状态
    mSignal.wait(mLock);
}
```
#### 3.3.1 进入Renderthread线程
```
/*frameworks/base/libs/hwui/renderthread/DrawFrameTask.cpp*/
void DrawFrameTask::run() {
    // 原生标识一帧渲染绘制任务的systrace tag
    ATRACE_NAME("DrawFrame");
    ...
    {
        TreeInfo info(TreeInfo::MODE_FULL, *mContext);
        //1.将UI线程构建的DisplayListOp绘制命令树同步到RenderThread渲染线程
        canUnblockUiThread = syncFrameState(info);
        ...
    }
    ...
    if (CC_LIKELY(canDrawThisFrame)) {
        // 2.执行draw渲染绘制动作
        context->draw();
    } else {
        ...
    }
    ...
}

/*frameworks/base/libs/hwui/renderthread/CanvasContext.cpp*/
void CanvasContext::draw() {
    ......
    // 2.1 这里会通过3.1小节传入的Surface 向galloc dequeueBuffer
    Frame frame = mRenderPipeline->getFrame();
    ......
    // 2.2 调用OpenGL库使用GPU，按照构建好的绘制命令完成界面的渲染
    bool drew = mRenderPipeline->draw(frame, windowDirty, dirty, mLightGeometry, &mLayerUpdateQueue,
                                      mContentDrawBounds, mOpaque, mLightInfo, mRenderNodes,
                                      &(profiler()));
    ......
    waitOnFences();
    ......
    // 2.3 这里最终用到Surface的queueBuffer
    bool didSwap =
            mRenderPipeline->swapBuffers(frame, drew, windowDirty, mCurrentFrameInfo, &requireSwap);
    ......
}
```
### 3.4 app进程申请buffer
#### 3.4.1 调用gl库
接 3.3.1小节的2.1 来到SkiaOpenGLPipeline或者SkiaVulkanPipeline类的getFrame()方法。
```
/*frameworks/base/libs/hwui/pipeline/skia/SkiaOpenGLPipeline.cpp*/

Frame SkiaOpenGLPipeline::getFrame() {
    LOG_ALWAYS_FATAL_IF(mEglSurface == EGL_NO_SURFACE,
                        "drawRenderNode called on a context with no surface!");
    return mEglManager.beginFrame(mEglSurface);
}

/*frameworks/base/libs/hwui/renderthread/EglManager.cpp*/
Frame EglManager::beginFrame(EGLSurface surface) {
    LOG_ALWAYS_FATAL_IF(surface == EGL_NO_SURFACE, "Tried to beginFrame on EGL_NO_SURFACE!");

    ...
    makeCurrent(surface);
    ...
    frame.mBufferAge = queryBufferAge(surface);
    eglBeginFrame(mEglDisplay, surface);
    return frame;
}

bool EglManager::makeCurrent(EGLSurface surface, EGLint* errOut, bool force) {
    if (!force && isCurrent(surface)) return false;

    if (surface == EGL_NO_SURFACE) {
        // Ensure we always have a valid surface & context
        surface = mPBufferSurface;
    }
    // 进入egl库
    if (!eglMakeCurrent(mEglDisplay, surface, surface, mEglContext)) {
        ....
    }
    ...
    return true;
}

//进入egl库
/*frameworks/native/opengl/libs/EGL/eglApi.cpp*/
EGLBoolean eglMakeCurrent(EGLDisplay dpy, EGLSurface draw, EGLSurface read, EGLContext ctx) {
    clearError();

    egl_connection_t* const cnx = &gEGLImpl;
    return cnx->platform.eglMakeCurrent(dpy, draw, read, ctx);
}

//最终进入opengl实现库（厂商实现），andorid有个开源实现库libagl，但是在11及以上删除了。
这里按照Android 10的libagl看一下实现
/*frameworks/native/opengl/libagl/egl.cpp*/ （Android 10 及以下libagl源码目录）
EGLBoolean eglMakeCurrent(  EGLDisplay dpy, EGLSurface draw,
                            EGLSurface read, EGLContext ctx)
{
    ....
    if (d->connect() == EGL_FALSE) {
                    return EGL_FALSE;
    }
    ...
    return setError(EGL_BAD_ACCESS, EGL_FALSE);
}

EGLBoolean egl_window_surface_v2_t::connect() 
{
    // we're intending to do software rendering
    native_window_set_usage(nativeWindow, 
            GRALLOC_USAGE_SW_READ_OFTEN | GRALLOC_USAGE_SW_WRITE_OFTEN);
    ...
    // dequeue a buffer
    int fenceFd = -1;
    //申请buffer，开始进入libgui库，这里的nativewindow就是3.1那一步传入的surface转换来的。
    if (nativeWindow->dequeueBuffer(nativeWindow, &buffer,
            &fenceFd) != NO_ERROR) {
        return setError(EGL_BAD_ALLOC, EGL_FALSE);
    }
    ...
    return EGL_TRUE;
}
```
#### 3.4.2 调用libgui库并跨进程创建buffer
接上节参考这篇文章的[Android画面显示流程分析(3)](ttps://www.jianshu.com/p/3c61375cc15b)3.2，先进入libgui和SF进程建立连接，并最终来到surfaceflinger进程，这里还可以看韦东山课程的这张图
![Buffer申请](/img/buffer%E5%88%9B%E5%BB%BA.jpg)

```
/*frameworks/native/libs/gui/BufferQueueProducer.cpp */
status_t BufferQueueProducer::dequeueBuffer(int* outSlot, sp<android::Fence>* outFence,
                                            uint32_t width, uint32_t height, PixelFormat format,
                                            uint64_t usage, uint64_t* outBufferAge,
                                            FrameEventHistoryDelta* outTimestamps) {
    if ((buffer == NULL) ||
        buffer->needsReallocation(width, height, format, BQ_LAYER_COUNT, usage))//检查是否已分配了GraphicBuffer
    {
        ......
        returnFlags |= BUFFER_NEEDS_REALLOCATION;//发现需要分配buffer,置个标记
    }
    ......
    if (returnFlags & BUFFER_NEEDS_REALLOCATION) {
        ......
        //新创建一个新的GraphicBuffer给到对应的slot
        sp<GraphicBuffer> graphicBuffer = new GraphicBuffer(
               width, height, format, BQ_LAYER_COUNT, usage,
               {mConsumerName.string(), mConsumerName.size()});
        ......
               mSlots[*outSlot].mGraphicBuffer = graphicBuffer;//把GraphicBuffer给到对应的slot
        ......
    }
    ......
    return returnFlags;//注意在应用第一次请求buffer, dequeueBuffer返回时对应的GraphicBuffer已经创建完成并给到了对应的slot上，但返回给应用的flags里还是带有BUFFER_NEEDS_REALLOCATION标记的
}
```
#### 3.4.3 调用libui库
接3.4.2 来到GraphicBuffer的构造函数
```
/*frameworks/native/libs/ui/GraphicBuffer.cpp*/
GraphicBuffer::GraphicBuffer(uint32_t inWidth, uint32_t inHeight, PixelFormat inFormat,
                             uint32_t inLayerCount, uint64_t inUsage, std::string requestorName)
      : GraphicBuffer() {
    mInitCheck = initWithSize(inWidth, inHeight, inFormat, inLayerCount, inUsage,
                              std::move(requestorName));
}

status_t GraphicBuffer::initWithSize(uint32_t inWidth, uint32_t inHeight,
        PixelFormat inFormat, uint32_t inLayerCount, uint64_t inUsage,
        std::string requestorName)
{
    //1.创建GraphicBufferAllocator最终通过hal层调用galloc模块
    GraphicBufferAllocator& allocator = GraphicBufferAllocator::get();
    uint32_t outStride = 0;
    //2.分配buffer，最终也调到hal层的galloc模块
    status_t err = allocator.allocate(inWidth, inHeight, inFormat, inLayerCount,
            inUsage, &handle, &outStride, mId,
            std::move(requestorName));
    if (err == NO_ERROR) {
        mBufferMapper.getTransportSize(handle, &mTransportNumFds, &mTransportNumInts);

        width = static_cast<int>(inWidth);
        height = static_cast<int>(inHeight);
        format = inFormat;
        layerCount = inLayerCount;
        usage = inUsage;
        usage_deprecated = int(usage);
        stride = static_cast<int>(outStride);
    }
    return err;
}

/*frameworks/native/libs/ui/GraphicBufferAllocator.cpp*/

GraphicBufferAllocator::GraphicBufferAllocator() : mMapper(GraphicBufferMapper::getInstance()) {
    //1.1 创建Gralloc4Allocator对象
    mAllocator = std::make_unique<const Gralloc4Allocator>(
            reinterpret_cast<const Gralloc4Mapper&>(mMapper.getGrallocMapper()));
    if (mAllocator->isLoaded()) {
        return;
    }
    mAllocator = std::make_unique<const Gralloc3Allocator>(
            reinterpret_cast<const Gralloc3Mapper&>(mMapper.getGrallocMapper()));
    if (mAllocator->isLoaded()) {
        return;
    }
    mAllocator = std::make_unique<const Gralloc2Allocator>(
            reinterpret_cast<const Gralloc2Mapper&>(mMapper.getGrallocMapper()));
    if (mAllocator->isLoaded()) {
        return;
    }

    LOG_ALWAYS_FATAL("gralloc-allocator is missing");
}

/* frameworks/native/libs/ui/Gralloc4.cpp*/
Gralloc4Mapper::Gralloc4Mapper() {
    //1.1.1 跨进程调用hidl服务
    mMapper = IMapper::getService();
    if (mMapper == nullptr) {
        ALOGI("mapper 4.x is not supported");
        return;
    }
    if (mMapper->isRemote()) {
        LOG_ALWAYS_FATAL("gralloc-mapper must be in passthrough mode");
    }
}
最终调到hal层，并去指定目录下（高通的/vendor/lib64/hw/galloc.xxx.so）下加载厂商的动态库
高通galloc库的源码（hardware/qcom/sdm845/display/gralloc/）
/*hardware/interfaces/graphics/mapper/4.0/IMapper.hal*/

```
#### 3.4.4 hidl跨进程调用厂商动态库打开ion驱动分配buffer

```
/*hardware/qcom/sdm845/display/gralloc/gr_ion_alloc.cpp*/
int IonAlloc::AllocBuffer(AllocData *data) {
  ATRACE_CALL();
  int err = 0;
  struct ion_handle_data handle_data;
  struct ion_fd_data fd_data;
  struct ion_allocation_data ion_alloc_data;

  ion_alloc_data.len = data->size;
  ion_alloc_data.align = data->align;
  ion_alloc_data.heap_id_mask = data->heap_id;
  ion_alloc_data.flags = data->flags;
  ion_alloc_data.flags |= data->uncached ? 0 : ION_FLAG_CACHED;
  std::string tag_name{};
  if (ATRACE_ENABLED()) {
    tag_name = "ION_IOC_ALLOC size: " + std::to_string(data->size);
  }

  ATRACE_BEGIN(tag_name.c_str());
  //调用libion库打开ion驱动
  if (ioctl(ion_dev_fd_, INT(ION_IOC_ALLOC), &ion_alloc_data)) {
    err = -errno;
    ALOGE("ION_IOC_ALLOC failed with error - %s", strerror(errno));
    return err;
  }
  ATRACE_END();

  fd_data.handle = ion_alloc_data.handle;
  handle_data.handle = ion_alloc_data.handle;
  ATRACE_BEGIN("ION_IOC_MAP");
  if (ioctl(ion_dev_fd_, INT(ION_IOC_MAP), &fd_data)) {
    err = -errno;
    ALOGE("%s: ION_IOC_MAP failed with error - %s", __FUNCTION__, strerror(errno));
    ioctl(ion_dev_fd_, INT(ION_IOC_FREE), &handle_data);
    return err;
  }
  ATRACE_END();

  data->fd = fd_data.fd;
  data->ion_handle = handle_data.handle;
  ALOGD_IF(DEBUG, "ion: Allocated buffer size:%zu fd:%d handle:0x%x", ion_alloc_data.len, data->fd,
           data->ion_handle);

  return 0;
}
```
#### 3.4.5 打开ion驱动最终调用到drm分享buffer
```
/*system/memory/libion/ion.c */
int ion_alloc(int fd, size_t len, size_t align, unsigned int heap_mask, unsigned int flags,
              ion_user_handle_t* handle) {
    int ret = 0;

    if ((handle == NULL) || (!ion_is_legacy(fd))) return -EINVAL;

    struct ion_allocation_data data = {
        .len = len, .align = align, .heap_id_mask = heap_mask, .flags = flags,
    };

    ret = ion_ioctl(fd, ION_IOC_ALLOC, &data);
    if (ret < 0) return ret;

    *handle = data.handle;

    return ret;
}
```

### 3.5 绘制
接3.3.1小节的2.2 来到SkiaOpenGLPipeline或者SkiaVulkanPipeline类的getFrame()方法。

```
/*frameworks/base/libs/hwui/pipeline/skia/SkiaOpenGLPipeline.cpp*/
bool SkiaOpenGLPipeline::draw(const Frame& frame, const SkRect& screenDirty, const SkRect& dirty,
                              const LightGeometry& lightGeometry,
                              LayerUpdateQueue* layerUpdateQueue, const Rect& contentDrawBounds,
                              bool opaque, const LightInfo& lightInfo,
                              const std::vector<sp<RenderNode>>& renderNodes,
                              FrameInfoVisualizer* profiler) {
    ......
    renderFrame(*layerUpdateQueue, dirty, renderNodes, opaque, contentDrawBounds, surface,
                SkMatrix::I());
    ....

    return true;
}


void SkiaPipeline::renderFrame(const LayerUpdateQueue& layers, const SkRect& clip,
                               const std::vector<sp<RenderNode>>& nodes, bool opaque,
                               const Rect& contentDrawBounds, sk_sp<SkSurface> surface,
                               const SkMatrix& preTransform) {
    ......

    renderFrameImpl(clip, nodes, opaque, contentDrawBounds, canvas, preTransform);

    endCapture(surface.get());
    ....
    if (CC_UNLIKELY(Properties::debugOverdraw)) {
        renderOverdraw(clip, nodes, contentDrawBounds, surface, preTransform);
    }
    .....
}

void SkiaPipeline::renderOverdraw(const SkRect& clip,
                                  const std::vector<sp<RenderNode>>& nodes,
                                  const Rect& contentDrawBounds, sk_sp<SkSurface> surface,
                                  const SkMatrix& preTransform) {
    // Set up the overdraw canvas.
    ....
    surface->getCanvas()->drawImage(counts.get(), 0.0f, 0.0f, SkSamplingOptions(), &paint);
}

进入libskia库最终转化为gl指令，发给GPU,这里还要在学习。
/*external/skia/src/core/SkCanvas.cpp*/

void SkCanvas::drawImage(const SkImage* image, SkScalar x, SkScalar y,
                         const SkSamplingOptions& sampling, const SkPaint* paint) {
    TRACE_EVENT0("skia", TRACE_FUNC);
    RETURN_ON_NULL(image);
    this->onDrawImage2(image, x, y, sampling, paint);
}

```
### 3.6 app进程提交buffer
绘制完成后，接3.3.1 小节的2.3 提交buffer到前台，并唤醒vsync线程，开始合成。
#### 3.6.1 hwui库和egl库的调用
接3.3.1 小节的2.3来到SkiaOpenGLPipeline的swapBuffers方法
```
/*frameworks/base/libs/hwui/pipeline/skia/SkiaOpenGLPipeline.cpp */
bool SkiaOpenGLPipeline::swapBuffers(const Frame& frame, bool drew, const SkRect& screenDirty,
                                     FrameInfo* currentFrameInfo, bool* requireSwap) {
    GL_CHECKPOINT(LOW);
    ......
    *requireSwap = drew || mEglManager.damageRequiresSwap();
    //调用EglManager
    if (*requireSwap && (CC_UNLIKELY(!mEglManager.swapBuffers(frame, screenDirty)))) {
        return false;
    }
    .....
}

/*frameworks/base/libs/hwui/renderthread/EglManager.cpp*/
bool EglManager::swapBuffers(const Frame& frame, const SkRect& screenDirty) {
    if (CC_UNLIKELY(Properties::waitForGpuCompletion)) {
        ATRACE_NAME("Finishing GPU work");
        fence();
    }

    EGLint rects[4];
    frame.map(screenDirty, rects);
    //调用egl接口
    eglSwapBuffersWithDamageKHR(mEglDisplay, frame.mSurface, rects, screenDirty.isEmpty() ? 0 : 1);
    LOG_ALWAYS_FATAL("Encountered EGL error %d %s during rendering", err, egl_error_str(err));
    // Impossible to hit this, but the compiler doesn't know that
    return false;
}

/*frameworks/native/opengl/libs/EGL/eglApi.cpp*/
EGLBoolean eglSwapBuffersWithDamageKHR(EGLDisplay dpy, EGLSurface draw, EGLint* rects,
                                       EGLint n_rects) {
    ATRACE_CALL();
    clearError();

    egl_connection_t* const cnx = &gEGLImpl;
    return cnx->platform.eglSwapBuffersWithDamageKHR(dpy, draw, rects, n_rects);
}

```
#### 3.6.2 opengl厂商实现库的调用
//最终进入opengl实现库（厂商实现），andorid有个开源实现库libagl，但是在11及以上删除了。
这里按照Android 10的libagl看一下实现
```
/*frameworks/native/opengl/libagl/egl.cpp*/ （Android 10 及以下libagl源码目录）
EGLBoolean eglSwapBuffers(EGLDisplay dpy, EGLSurface draw)
{
    if (egl_display_t::is_valid(dpy) == EGL_FALSE)
        return setError(EGL_BAD_DISPLAY, EGL_FALSE);

    egl_surface_t* d = static_cast<egl_surface_t*>(draw);
    // 这里d（egl_surface_t）是3.1小节surface转换过来的。是一个AnativeWindow对象
    d->swapBuffers();
    ...

    return EGL_TRUE;
}
接上，调到了egl_window_surface_v2_t的swapBuffers方法

EGLBoolean egl_window_surface_v2_t::swapBuffers()
{
    ....
    //nativeWindow本质上是一个surface,调用mGraphicBufferProducer的方法
    nativeWindow->queueBuffer(nativeWindow, buffer, -1);
    .....
    buffer = 0;
    return EGL_TRUE;
}
```
#### 3.6.3 来到libgui的surface调用
```
/*frameworks/native/libs/gui/Surface.cpp*/
int Surface::queueBuffer(android_native_buffer_t* buffer, int fenceFd) {
    ......
    sp<Fence> fence(fenceFd >= 0 ? new Fence(fenceFd) : Fence::NO_FENCE);
    IGraphicBufferProducer::QueueBufferOutput output;
    IGraphicBufferProducer::QueueBufferInput input(timestamp, isAutoTimestamp,
            static_cast<android_dataspace>(mDataSpace), crop, mScalingMode,
            mTransform ^ mStickyTransform, fence, mStickyTransform,
            mEnableFrameTimestamps);

    status_t err = mGraphicBufferProducer->queueBuffer(i, input, &output);
    ......
    return err;
}

/*frameworks/native/libs/gui/BufferQueueProducer.cpp*/
status_t BufferQueueProducer::queueBuffer(int slot,
        const QueueBufferInput &input, QueueBufferOutput *output) {
    ......
    //消费者代理
    sp<IConsumerListener> frameAvailableListener;
    sp<IConsumerListener> frameReplacedListener;
        .....
        // 保存到队列
        mCore->mQueue.push_back(item);
        frameAvailableListener = mCore->mConsumerListener;
        ......
        
        if (frameAvailableListener != nullptr) {
        //通知消费者代理类
        frameAvailableListener->onFrameAvailable(item);
        }
        ......
    } // Autolock scope
    .....
    return NO_ERROR;
}
```
上面最终唤醒vsync线程，发送vsync信号，接上了[Android图形系统-窗口子系统-合成](https://ywqyunshan.github.io/2023/03/03/)的3.4.1，并开始合成。