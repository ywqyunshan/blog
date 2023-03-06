---
layout:     post

title:      "Android图形系统-窗口子系统"
subtitle:   "合成"
excerpt: ""
author:     "iigeoywq button"
date:       2023-03-03
published: true 
tags:
    - Istio 
categories: [ Tech ]
URL: "/2023/03/03/"
---
> 上一篇提到layer创建后，并申请buffer绘制，接着需要给layer设置对应的displayid，并设置合成类型（GPU/HWC），最终提交HWC合成。

## 一 窗口子系统（合成）架构
合成主要是在SF和HWC测完成,先说几个概念。
|  SF    | HWC Client  | HWC Server| 备注|
|  ----  | ----  |----|-----|
|   | HWC2::Device  |hw_device_t|dpu的抽象|
|DisplayDevice |HWC2::Display  | hw_display_t |显示屏幕抽象（物理屏幕或者虚拟显示屏）|
|Layer|HWC2::Layer |hw_layer_t|叠加图形抽象|


## 二 源码分析
接前2篇的分析，app进程收到vsync-app信号后，开始绘制，同时sf进程收到vsync-sf信号，开始寻找可以合并的layer。
合成layer需要设置displayid，首先需要通过HWC拿到display数组。
### 2.1 打开hwc拿到hw_device_t
系统启动后，init会解析hardware/interfaces/graphics/composer/2.x/default/android.hardware.graphics.composer@2.2-service.rc文件，启动服务进程，来到service.cpp的main函数，调用HwcLoader::load()方法。
```
/*hardware/interfaces/graphics/composer/2.2/default/service.cpp*/
int main() {
    ...
    // 1. 加载hwc模块
    android::sp<IComposer> composer = HwcLoader::load();
    if (composer == nullptr) {
        return 1;
    }
    if (composer->registerAsService() != android::NO_ERROR) {
        ALOGE("failed to register service");
        return 1;
    }
    ...
}

/*hardware/interfaces/graphics/composer/2.2/utils/passthrough/include/composer-passthrough/2.2/HwcLoader.h*/
static IComposer* load() {
    //1.1 hw_module_t 定义在hardware.h中，表示一个硬件模块,加载hwc模块
    const hw_module_t* module = loadModule();
    if (!module) {
        return nullptr;
    }
    //1.2 获取hwc2_device_t
    auto hal = createHalWithAdapter(module);
    if (!hal) {
        return nullptr;
    }
    return createComposer(std::move(hal)).release();
}

/*hardware/interfaces/graphics/composer/2.1/utils/passthrough/include/composer-passthrough/2.1/HwcLoader.h*/
static const hw_module_t* loadModule() {
    const hw_module_t* module;
    //1.1.1 加载厂商实现的so库，例如高通vendor/lib64/hw/hwcomposer.sdmxxx.so(HWC2)
    int error = hw_get_module(HWC_HARDWARE_MODULE_ID, &module);
    if (error) {
        ALOGI("falling back to gralloc module");
        error = hw_get_module(GRALLOC_HARDWARE_MODULE_ID, &module);
    }
    if (error) {
        ALOGE("failed to get hwcomposer or gralloc module");
        return nullptr;
    }
    return module;
}

// 接1.2 获取hwc2_device_t指针
static std::unique_ptr<hal::ComposerHal> createHalWithAdapter(const hw_module_t* module) {
        bool adapted;
        //1.2.1 
        hwc2_device_t* device = openDeviceWithAdapter(module, &adapted);
        if (!device) {
            return nullptr;
        }
        auto hal = std::make_unique<HwcHal>();
        //1.2.2
        return hal->initWithDevice(std::move(device), !adapted) ? std::move(hal) : nullptr;
}

open hwcomposer2 device, install an adapter if necessary
static hwc2_device_t* openDeviceWithAdapter(const hw_module_t* module, bool* outAdapted) {
    if (module->id && std::string(module->id) == GRALLOC_HARDWARE_MODULE_ID) {
        *outAdapted = true;
        return adaptGrallocModule(module);
    }
    hw_device_t* device;
    //1.2.1.1 打开驱动，拿到hwc2_device_t（代表一个hwc设备）
    int error = module->methods->open(module, HWC_HARDWARE_COMPOSER, &device);
    if (error) {
        ALOGE("failed to open hwcomposer device: %s", strerror(-error));
        return nullptr;
    }
    int major = (device->version >> 24) & 0xf;
    if (major != 2) {
        *outAdapted = true;
        return adaptHwc1Device(std::move(reinterpret_cast<hwc_composer_device_1*>(device)));
    }
    *outAdapted = false;
        return reinterpret_cast<hwc2_device_t*>(device);
}

```
接1.2.1 来到厂商实现
```
/*hardware/qcom/sdm845/display/sdm/libs/hwc2/hwc_session.cpp*/
int HWCSession::Open(const hw_module_t *module, const char *name, hw_device_t **device) {
  if (!module || !name || !device) {
    DLOGE("Invalid parameters.");
    return -EINVAL;
  }

  if (!strcmp(name, HWC_HARDWARE_COMPOSER)) {
    HWCSession *hwc_session = new HWCSession(module);
    if (!hwc_session) {
      return -ENOMEM;
    }

    int status = hwc_session->Init();
    if (status != 0) {
      delete hwc_session;
      hwc_session = NULL;
      return status;
    }

    hwc2_device_t *composer_device = hwc_session;
    *device = reinterpret_cast<hw_device_t *>(composer_device);
  }

  return 0;
}

int HWCSession::Init() {
  SCOPE_LOCK(locker_[HWC_DISPLAY_PRIMARY]);

  int status = -EINVAL;
  const char *qservice_name = "display.qservice";
  .....

  // If HDMI display is primary display, defer display creation until hotplug event is received.
  HWDisplayInterfaceInfo hw_disp_info = {};
  error = core_intf_->GetFirstDisplayInterfaceType(&hw_disp_info);
  ......

  if (hw_disp_info.type == kHDMI) {
    status = 0;
    hdmi_is_primary_ = true;
    // Create display if it is connected, else wait for hotplug connect event.
    if (hw_disp_info.is_connected) {
      //创建扩展屏幕
      status = CreateExternalDisplay(HWC_DISPLAY_PRIMARY, 0, 0, false);
    }
  } else {
    // Create and power on primary display
    //创建主屏幕
    status = HWCDisplayPrimary::Create(core_intf_, &buffer_allocator_, &callbacks_, qservice_,
                                       &hwc_display_[HWC_DISPLAY_PRIMARY]);
    color_mgr_ = HWCColorManager::CreateColorManager(&buffer_allocator_);
    if (!color_mgr_) {
      DLOGW("Failed to load HWCColorManager.");
    }
  }
  ......
  return 0;
}
```
### 2.2 SF测启动并向HWC服务注册callback，获取热插拔的display id 保存到displayDevices数组
sf进程启动后，做了好几件事情，我们接下主要关注hwc的初始化和hwc注册callback
```
/*frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp*/
void SurfaceFlinger::init() {
    ALOGI(  "SurfaceFlinger's main thread ready to run. "
            "Initializing graphics H/W...");
    Mutex::Autolock _l(mStateLock);

    // Get a RenderEngine for the given display / config (can't fail)
    // TODO(b/77156734): We need to stop casting and use HAL types when possible.
    // Sending maxFrameBufferAcquiredBuffers as the cache size is tightly tuned to single-display.
    //1. 创建RenderEngine对象（准备egl环境），用来做gpu合成
    mCompositionEngine->setRenderEngine(renderengine::RenderEngine::create(
            renderengine::RenderEngineCreationArgs::Builder()
                .setPixelFormat(static_cast<int32_t>(defaultCompositionPixelFormat))
                .setImageCacheSize(maxFrameBufferAcquiredBuffers)
                .setUseColorManagerment(useColorManagement)
                .setEnableProtectedContext(enable_protected_contents(false))
                .setPrecacheToneMapperShaderOnly(false)
                .setSupportsBackgroundBlur(mSupportsBlur)
                .setContextPriority(useContextPriority
                        ? renderengine::RenderEngine::ContextPriority::HIGH
                        : renderengine::RenderEngine::ContextPriority::MEDIUM)
                .build()));
    // 2.初始化hwc环境，最终和hwc server接口层hardware/interfaces/graphics/composer/2.1/IComposer.hal关联
    mCompositionEngine->setHwComposer(getFactory().createHWComposer(getBE().mHwcServiceName));
    //3 向hwc server注册，监听一些状态变化
    mCompositionEngine->getHwComposer().setConfiguration(this, getBE().mComposerSequenceId);
    // Process any initial hotplug and resulting display changes.
    //4 处理hwc server的热插拔回调
    processDisplayHotplugEventsLocked();
    const auto display = getDefaultDisplayDeviceLocked();
    LOG_ALWAYS_FATAL_IF(!display, "Missing internal display after registering composer callback.");
    LOG_ALWAYS_FATAL_IF(!getHwComposer().isConnected(*display->getId()),
                        "Internal display is disconnected.");

    // 5. initialize our drawing state
    //mDrawingState和mCurrentState用来比较layer是否变化
    mDrawingState = mCurrentState;

    // 6. set initial conditions (e.g. unblank default device)
    initializeDisplays();
    ....
}
```
#### 2.2.1 hwc初始化和server端关联
接2.2的2 来到SurfaceFlingerDefaultFactory的createHWComposer方法
```
/* frameworks/native/services/surfaceflinger/SurfaceFlingerDefaultFactory.cpp*/
std::unique_ptr<HWComposer> DefaultFactory::createHWComposer(const std::string& serviceName) {
    return std::make_unique<android::impl::HWComposer>(serviceName);
}

/*frameworks/native/services/surfaceflinger/DisplayHardware/HWComposer.cpp*/
namespace android {
    .....
namespace impl {
    .....
HWComposer::HWComposer(const std::string& composerServiceName)
      : mComposer(std::make_unique<Hwc2::impl::Composer>(composerServiceName)) {
}
    .....
    }
    }

/*frameworks/native/services/surfaceflinger/DisplayHardware/ComposerHal.cpp*/
namespace Hwc2 {
    .....
    namespace impl {
        ......
//1 创建mComposer对象
Composer::Composer(const std::string& serviceName)
    : mWriter(kWriterInitialSize),
      mIsUsingVrComposer(serviceName == std::string("vr"))
{
    //2 Composer和hardware/interfaces/graphics/composer/2.1/IComposer.hal关联
    mComposer = V2_1::IComposer::getService(serviceName);
    ....

    if (sp<IComposer> composer_2_4 = IComposer::castFrom(mComposer)) {
        //3.1 client和hardware/interfaces/graphics/composer/2.4/IComposerClient.hal关联
        composer_2_4->createClient_2_4([&](const auto& tmpError, const auto& tmpClient) {
            if (tmpError == V2_4::Error::NONE) {
                mClient = tmpClient;
                mClient_2_2 = tmpClient;
                mClient_2_3 = tmpClient;
                mClient_2_4 = tmpClient;
            }
        });
    } else if (sp<V2_3::IComposer> composer_2_3 = V2_3::IComposer::castFrom(mComposer)) {
        //3.2 client和hardware/interfaces/graphics/composer/2.3/IComposerClient.hal关联
        composer_2_3->createClient_2_3([&](const auto& tmpError, const auto& tmpClient) {
            if (tmpError == Error::NONE) {
                mClient = tmpClient;
                mClient_2_2 = tmpClient;
                mClient_2_3 = tmpClient;
            }
        });
    } else {
        //3.2 client和hardware/interfaces/graphics/composer/2.2/IComposerClient.hal关联
        mComposer->createClient([&](const auto& tmpError, const auto& tmpClient) {
            if (tmpError != Error::NONE) {
                return;
            }
            
            mClient = tmpClient;
            if (sp<V2_2::IComposer> composer_2_2 = V2_2::IComposer::castFrom(mComposer)) {
                mClient_2_2 = V2_2::IComposerClient::castFrom(mClient);
                LOG_ALWAYS_FATAL_IF(mClient_2_2 == nullptr,
                                    "IComposer 2.2 did not return IComposerClient 2.2");
            }
        });
    }
}
......
}
......
}
```
#### 2.2.2 SF向hwc server注册callback
接2.2的3 来到SurfaceFlingerDefaultFactory的createHWComposer方法
```
/*frameworks/native/services/surfaceflinger/DisplayHardware/HWComposer.cpp*/
void HWComposer::setConfiguration(HWC2::ComposerCallback* callback, int32_t sequenceId) {
    loadCapabilities();
    loadLayerMetadataSupport();

    if (mRegisteredCallback) {
        ALOGW("Callback already registered. Ignored extra registration attempt.");
        return;
    }
    mRegisteredCallback = true;
    sp<ComposerCallbackBridge> callbackBridge(
            new ComposerCallbackBridge(callback, sequenceId,
                                       mComposer->isVsyncPeriodSwitchSupported()));
    //这里的mComposer是2.2.1 中1 创建mComposer
    mComposer->registerCallback(callbackBridge);
}

/*frameworks/native/services/surfaceflinger/DisplayHardware/ComposerHal.cpp*/
void Composer::registerCallback(const sp<IComposerCallback>& callback)
{
    android::hardware::setMinSchedulerPolicy(callback, SCHED_FIFO, 2);
    auto ret = [&]() {
        if (mClient_2_4) {
            return mClient_2_4->registerCallback_2_4(callback);
        }
        //向hwc server 测的hardware/interfaces/graphics/composer/2.1/IComposerClient.hal注册callback
        return mClient->registerCallback(callback);
    }();
    ......
}

/*hardware/interfaces/graphics/composer/2.1/IComposerClient.hal*/ 这是Hwc server接口层
interface IComposerClient {
    ....
    @entry
    @callflow(next="*")
    registerCallback(IComposerCallback callback);
    }
    ....
    }
/*hardware/interfaces/graphics/composer/2.1/utils/hal/include/composer-hal/2.1/ComposerClient.h*/
    Return<void> registerCallback(const sp<IComposerCallback>& callback) override {
        // no locking as we require this function to be called only once
        mHalEventCallback = std::make_unique<HalEventCallback>(callback, mResources.get());
        mHal->registerEventCallback(mHalEventCallback.get());
        return Void();
    }

/* hardware/interfaces/graphics/composer/2.1/utils/passthrough/include/composer-passthrough/2.1/HwcHal.h*/
void registerEventCallback(hal::ComposerHal::EventCallback* callback) override {
        mEventCallback = callback;
        
        mDispatch.registerCallback(mDevice, HWC2_CALLBACK_HOTPLUG, this,
                                   reinterpret_cast<hwc2_function_pointer_t>(hotplugHook));
        mDispatch.registerCallback(mDevice, HWC2_CALLBACK_REFRESH, this,
                                   reinterpret_cast<hwc2_function_pointer_t>(refreshHook));
        mDispatch.registerCallback(mDevice, HWC2_CALLBACK_VSYNC, this,
                                   reinterpret_cast<hwc2_function_pointer_t>(vsyncHook));
    }
```
来到厂商的so库
```
/*hardware/qcom/sdm845/display/sdm/libs/hwc2/hwc_session.cpp*/
int32_t HWCSession::RegisterCallback(hwc2_device_t *device, int32_t descriptor,
                                     hwc2_callback_data_t callback_data,
                                     hwc2_function_pointer_t pointer) {
  if (!device) {
    return HWC2_ERROR_BAD_PARAMETER;
  }
  HWCSession *hwc_session = static_cast<HWCSession *>(device);
  SCOPE_LOCK(hwc_session->callbacks_lock_);
  auto desc = static_cast<HWC2::Callback>(descriptor);
  auto error = hwc_session->callbacks_.Register(desc, callback_data, pointer);
  DLOGD("%s callback: %s", pointer ? "Registering" : "Deregistering", to_string(desc).c_str());
  //最终调到显示设备，判断是否有热插拔，并层层回调给sf测
  if (descriptor == HWC2_CALLBACK_HOTPLUG) {
    if (hwc_session->hwc_display_[HWC_DISPLAY_PRIMARY]) {
      hwc_session->callbacks_.Hotplug(HWC_DISPLAY_PRIMARY, HWC2::Connection::Connected);
    }
  }
  hwc_session->need_invalidate_ = false;
  hwc_session->callbacks_lock_.Broadcast();
  return INT32(error);
}
```
### 2.3 SF测收到callback通知，保存displayid 数组
如果显示屏幕插拔，SF测通过2.2.2的层层回调收到通知，上述的callback包括一个onHotplugReceived通知；
来到
```
/*frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp*/

void SurfaceFlinger::onHotplugReceived(int32_t sequenceId, hal::HWDisplayId hwcDisplayId,
                                       hal::Connection connection) {
    ALOGV("%s(%d, %" PRIu64 ", %s)", __FUNCTION__, sequenceId, hwcDisplayId,
          connection == hal::Connection::CONNECTED ? "connected" : "disconnected");

    ....
    //1 保存热插拔事件
    mPendingHotplugEvents.emplace_back(HotplugEvent{hwcDisplayId, connection});

    if (std::this_thread::get_id() == mMainThreadId) {
        // Process all pending hot plug events immediately if we are on the main thread.
        processDisplayHotplugEventsLocked();
    }
    ...
    setTransactionFlags(eDisplayTransactionNeeded);
}


先看一下state数据结构
/*frameworks/native/services/surfaceflinger/SurfaceFlinger.h*/
class State {
    public:
        .....
        const LayerVector::StateSet stateSet = LayerVector::StateSet::Invalid;
        // layer 数组
        LayerVector layersSortedByZ;
        // display数组
        DefaultKeyedVector< wp<IBinder>, DisplayDeviceState> displays;
        ....
    };
sf中存2个状态
mCurrentState -当前需要修改的state
mDrawingState -上次的state

void SurfaceFlinger::processDisplayHotplugEventsLocked() {
    for (const auto& event : mPendingHotplugEvents) {
        std::optional<DisplayIdentificationInfo> info =
                getHwComposer().onHotplug(event.hwcDisplayId, event.connection);
        ....
        //循环获取displayid
        const auto displayId = info->id;
        const auto token = mPhysicalDisplayTokens.get(displayId);
        ....
        DisplayDeviceState state;
        state.physical = {displayId, getHwComposer().getDisplayConnectionType(displayId),
                                  event.hwcDisplayId};
        //mCurrentState displays数组保存数据
        mCurrentState.displays.add(token, state);
        ....
        processDisplayChangesLocked();
    }

    mPendingHotplugEvents.clear();
}


/*frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp*/
void SurfaceFlinger::processDisplayChangesLocked() {
    // here we take advantage of Vector's copy-on-write semantics to
    // improve performance by skipping the transaction entirely when
    // know that the lists are identical
    //比较前后stage中display的变化，可知display的增减
    const KeyedVector<wp<IBinder>, DisplayDeviceState>& curr(mCurrentState.displays);
    const KeyedVector<wp<IBinder>, DisplayDeviceState>& draw(mDrawingState.displays);
    if (!curr.isIdenticalTo(draw)) {
        mVisibleRegionsDirty = true;

        // find the displays that were removed
        // (ie: in drawing state but not in current state)
        // also handle displays that changed
        // (ie: displays that are in both lists)
        for (size_t i = 0; i < draw.size(); i++) {
            const wp<IBinder>& displayToken = draw.keyAt(i);
            const ssize_t j = curr.indexOfKey(displayToken);
            if (j < 0) {
                // in drawing state but not in current state
                processDisplayRemoved(displayToken);
            } else {
                // this display is in both lists. see if something changed.
                const DisplayDeviceState& currentState = curr[j];
                const DisplayDeviceState& drawingState = draw[i];
                processDisplayChanged(displayToken, currentState, drawingState);
            }
        }

        // find displays that were added
        // (ie: in current state but not in drawing state)
        for (size_t i = 0; i < curr.size(); i++) {
            const wp<IBinder>& displayToken = curr.keyAt(i);
            if (draw.indexOfKey(displayToken) < 0) {
                processDisplayAdded(displayToken, curr[i]);
            }
        }
    }

    mDrawingState.displays = mCurrentState.displays;
}

void SurfaceFlinger::processDisplayAdded(const wp<IBinder>& displayToken,
                                         const DisplayDeviceState& state) {
    .....

    compositionengine::DisplayCreationArgsBuilder builder;
    if (const auto& physical = state.physical) {
        builder.setId(physical->id);
    } else {
        builder.setId(acquireVirtualDisplay(resolution, pixelFormat));
    }

    builder.setPixels(resolution);
    builder.setIsSecure(state.isSecure);
    builder.setPowerAdvisor(mPowerAdvisor.get());
    builder.setName(state.displayName);
    auto compositionDisplay = getCompositionEngine().createDisplay(builder.build());
    compositionDisplay->setLayerCachingEnabled(mLayerCachingEnabled);

    sp<compositionengine::DisplaySurface> displaySurface;
    sp<IGraphicBufferProducer> producer;
    sp<IGraphicBufferProducer> bqProducer;
    sp<IGraphicBufferConsumer> bqConsumer;
    getFactory().createBufferQueue(&bqProducer, &bqConsumer, /*consumerIsSurfaceFlinger =*/false);

    if (state.isVirtual()) {
        const auto displayId = VirtualDisplayId::tryCast(compositionDisplay->getId());
        LOG_FATAL_IF(!displayId);
        auto surface = sp<VirtualDisplaySurface>::make(getHwComposer(), *displayId, state.surface,
                                                       bqProducer, bqConsumer, state.displayName);
        displaySurface = surface;
        producer = std::move(surface);
    } else {
        ALOGE_IF(state.surface != nullptr,
                 "adding a supported display, but rendering "
                 "surface is provided (%p), ignoring it",
                 state.surface.get());
        const auto displayId = PhysicalDisplayId::tryCast(compositionDisplay->getId());
        LOG_FATAL_IF(!displayId);
        displaySurface =
                sp<FramebufferSurface>::make(getHwComposer(), *displayId, bqConsumer,
                                             state.physical->activeMode->getResolution(),
                                             ui::Size(maxGraphicsWidth, maxGraphicsHeight));
        producer = bqProducer;
    }

    LOG_FATAL_IF(!displaySurface);
    auto display = setupNewDisplayDeviceInternal(displayToken, std::move(compositionDisplay), state,
                                                 displaySurface, producer);
    //最后把新增的display存入数组，至此sf测的display数组拿到了hwc测的displayid                                             
    mDisplays.try_emplace(displayToken, std::move(display)); 
}
```
### 2.4 sf进程收到vsync-sf信号开始合成




