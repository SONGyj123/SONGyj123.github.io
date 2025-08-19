# Android drm_hwcomposer

## 0x00 What's HWC

HWC（Hardware Composer）是一个指导硬件合成画面的软件（Composer），比较出名的Composer如weston。它通过计算和判断生成最优的合成策略给到硬件合成器或者GPU。硬件合成器很多，比如三星的DPU（decon），rockchip的VOP就是一个硬件合成器，它通常带有显示的功能。HWC通过DRM接口操作DPU驱动进行合成与显示。不同的厂商对于HWC的实现是不同的，Google的AOSP中提供了一个官方的HWC[参考实现](https://android.googlesource.com/platform/external/drm_hwcomposer/)，我们后续就通过参考代码来了解HWC。

## 0x01 Android Graphic Stack

![[The Android display pipeline]](https://static.lwn.net/images/2020/adp-figure1.png)

## 0x02 Components in HWC

DRMHWC是一个服务，hwc3-drm.rc启动脚本将HWC作为一个init管理的后台服务程序

```bash
# drm_hwcomposer/hwc3/hwc3-drm.rc
service vendor.hwcomposer-3 /vendor/bin/hw/android.hardware.composer.hwc3-service.drm
    class hal animation
    interface aidl android.hardware.graphics.composer3.IComposer/default
    user system
    group graphics drmrpc
    capabilities SYS_NICE
    onrestart restart surfaceflinger ##当HWC重启时，surfaceflinger也会被重启，sf需要重新操作HWC配置pipeline
    task_profiles ServiceCapacityLow
```

DRMHWC的主程序位于service.cpp，主要做两件事情：

1. 创建composer
2. 注册hwc服务到binder

当composer注册到binder之后，sf就可以通过注册的服务名找到这个服务并且远程调用。

```c
int main(int /*argc*/, char* argv[]) {
	...
	auto composer = ndk::SharedRefBase::make<Composer>();
	...
	auto status = AServiceManager_addService(composer->asBinder().get(),
                                           instance.c_str());
}
```

### 1. Composer

composer中的接口很少，实际的操作都放在了composerclient中去实现，sf中获取到composer指针之后立即通过**createClient**方法创建出ComposerClient为后续操作做准备。

- SurfaceFlinger 是“客户”
- Composer 是“前台接待”，帮你建立一个会话
- ComposerClient 是“专属客服”，提供具体服务。

```c
// drm_hwcomposer/hwc3/Composer.h
class Composer : public BnComposer {
 public:
  Composer() = default;

  binder_status_t dump(int fd, const char** args, uint32_t num_args) override;

  // compser3 api
  ndk::ScopedAStatus createClient(
      std::shared_ptr<IComposerClient>* client) override;
  ndk::ScopedAStatus getCapabilities(std::vector<Capability>* caps) override;

 protected:
  ::ndk::SpAIBinder createBinder() override;

 private:
  std::weak_ptr<IComposerClient> client_;
};
```

### 2. ComposerClient

ComposerClient中提供了一大堆服务，主要看Init()和executeCommands()，其他都是大量获取特性的get函数。

```c++
class ComposerClient : public BnComposerClient {
 public:
  ComposerClient();
  ~ComposerClient() override;

  void Init();
  std::string Dump();

  // composer3 interface
  ndk::ScopedAStatus createLayer(int64_t display, int32_t buffer_slot_count,
                                 int64_t* layer) override;
  ndk::ScopedAStatus createVirtualDisplay(int32_t width, int32_t height,
                                          AidlPixelFormat format_hint,
                                          int32_t output_buffer_slot_count,
                                          VirtualDisplay* display) override;
  ndk::ScopedAStatus destroyLayer(int64_t display, int64_t layer) override;
  ndk::ScopedAStatus destroyVirtualDisplay(int64_t display) override;
  ndk::ScopedAStatus executeCommands(
      const std::vector<DisplayCommand>& commands,
      std::vector<CommandResultPayload>* results) override;
  ndk::ScopedAStatus getActiveConfig(int64_t display, int32_t* config) override;
  ndk::ScopedAStatus getColorModes(
      int64_t display, std::vector<ColorMode>* color_modes) override;
  
  ...
}
```

### 3. DrmXXX

说到底DRMHWC还是一个操作DRM驱动的应用程序，因此他还是需要对libdrm中的接口和抽象器件进行操作。我们在可以看到大量和drm相关的结构体：

```bash
├── DrmAtomicStateManager.cpp
├── DrmAtomicStateManager.h
├── DrmConnector.cpp
├── DrmConnector.h
├── DrmCrtc.cpp
├── DrmCrtc.h
├── DrmDevice.cpp
├── DrmDevice.h
├── DrmDisplayPipeline.cpp
├── DrmDisplayPipeline.h
├── DrmEncoder.cpp
├── DrmEncoder.h
├── DrmFbImporter.cpp
├── DrmFbImporter.h
├── DrmHwc.cpp
├── DrmHwc.h
├── DrmMode.cpp
├── DrmMode.h
├── DrmPlane.cpp
├── DrmPlane.h
├── DrmProperty.cpp
├── DrmProperty.h
├── DrmUnique.h
├── meson.build
├── ResourceManager.cpp
├── ResourceManager.h
├── UEventListener.cpp
├── UEventListener.h
├── VSyncWorker.cpp
└── VSyncWorker.h
```

## 0x03 HWC Init

- service添加composer服务到binder

  ```
  auto status = AServiceManager_addService(composer->asBinder().get(), instance.c_str());
  ```

  - composer->createclient()   SF获取到composer服务后，调用createclient创建composerclient，获取到composerclient指针进而远程调用相关接口

    - client->Init()    初始化composerclient

      - 创建一个DrmHwcThree实例

        ```c++
        void ComposerClient::Init() {
          DEBUG_FUNC();
          hwc_ = std::make_unique<DrmHwcThree>();
        }
        ```

  - ComposerClient->registerCallback()   注册hotplug回调函数

    - hwc_->Init(callback)   初始化DrmHwcThree实例

      - ResourceManager->Init()

        - DrmDevice::CreateInstance("/dev/dri/card%", this, 0)   根据drm节点路径打开drm设备 初始化drm设备

          - 设置drm的capability
          - 获取moderesources
          - 根据res获取CONN ENC CRTC PLANE

        - 注册回调函数

          ```c++
          uevent_listener_->RegisterHotplugHandler([this] {
            const std::unique_lock lock(GetMainLock());
            UpdateFrontendDisplays();
          });
          ```

        - 执行一次函数**UpdateFrontendDisplays()   更新Mode 创建pipeline 创建display 后面展开说**

UpdateFrontendDisplays作为hotplug回调函数在检测到热插拔时间后会进行调用，当HWC第一次初始化时也会调用一次，用于初始化显示pipeline。

- UpdateFrontendDisplays() 

  - GetOrderedConnectors()   获取当前的connector

    - conn->UpdateModes()   for循环中给每个conn更新显示mode

    - 判断当前conn的状态是否**DRM_MODE_CONNECTED**以及conn和pipeline配对状态

      - 如果**DRM_MODE_CONNECTED**且未配对，创建pipeline

        - DrmDisplayPipeline->CreatePipeline(*conn) 

        - BindDisplay(pipeline)   创建display并绑定到pipeline
        - attached_pipelines_[conn] = std::move(pipeline)   添加当前conn-pipeline配对

      - 如果**DRM_MODE_UNCONNECTED**，表示屏幕移除的情况
        - UnbindDisplay(pipeline)
        - attached_pipelines_.erase(conn)   移除当前conn-pipeline配对

    - FinalizeDisplayBinding()   创建display并SetPipeline

      - displays_[kPrimaryDisplay] = std::make_unique<HwcDisplay>(kPrimaryDisplay, HWC2::DisplayType::Physical, this)   创建display实例
      - displays_[kPrimaryDisplay]->SetPipeline({})   
        - HwcDisplay->Init()   初始化display
          - ChosePreferredConfig()   显示mode生成
            - 非headless模式   直接Update()将conn的mode获取到
            - headless模式（用于适配没有屏幕，android系统必须要一个primary屏幕否则重启，使用headless当一个物理屏幕都没有时用于欺骗SF防止重启）**GenFakeMode**(0, 0)
            - SetActiveConfig(configs_.**preferred_config_id**)   选择最优显示mode，保存下最优的condig_id
      - SendHotplugEventToClient(dhe.first, dhe.second)   发送热插拔事件给SF

上述工作完成后，理论上display pipeline已经通了，随后可以开始合成送显工作。

## 0x04 validatedisplay&presentdisplay

validatedisplay是一次带有test flag的atomic_commit，drm驱动识别到testflag之后并不会真的开始合成，而是返回是否可以合成的结果。因为硬件plane对于layer的合成能力不同，有的layer的特性plane可能并不支持，这时候需要将这个layer标记为GPU合成。

- display->ValidateStagedComposition()
  - headlessmode直接返回
  - backend_->ValidateDisplay(this, &num_types, &num_requests)
  - 判断是否需要进行validatedisplay操作 **testing_needed** = client_start != 0 || client_size != layers.size() 如何理解这个条件呢？ client_start == 0 && client_size == layers.size() 当所有的layer都被SF标记为client合成（GPU）时，不需要进行测试。换言之，只要有DPU硬件合成就需要进行测试
    - HwcDisplay::CreateComposition(AtomicCommitArgs &a_args)
      - 按z_order将layer排序，client_layer的zorder按照最小的client合成layer确定
      - import&populate 在此之前的layer没有实际的表示内容的那块内存，这一步通过importor将buffer_handle与layer绑定
      - DrmKmsPlan->CreateDrmKmsPlan   创建一个plan 所谓plan就是 **layer和plane的对应关系**
      - DrmAtomicStateManager->ExecuteAtomicCommit()
        - CommitFrame
          - 解析args入参，drmModeAtomicAddProperty添加
          - drmModeAtomicCommit(*drm->GetFd(), pset.get(), flags | **DRM_MODE_ATOMIC_TEST_ONLY**, drm);

presentdisplay是不带test flag的atomic_commit

- display->PresentStagedComposition()
  - headlessmode直接返回
  - HwcDisplay::CreateComposition(AtomicCommitArgs &a_args)同上

## 0x05 Q&A

1. **为什么 GPU 合成 (Client Composition) 最终只有一个 ClientLayer？**SurfaceFlinger 只会把合成好的这一个 Framebuffer 交回给 HWC，对 HWC 来说，不需要知道 GPU 合成内部有几个 Layer。随后这个返回的layer会和其他的硬件合成layer一起合成，这个clientlayer也会占用一个plane
2. **backend_是什么？**

## 0x00 Reference

[Android display pipeline]: https://lwn.net/Articles/809545/
[Android HWC intro]: https://source.android.com/docs/core/graphics/hwc

