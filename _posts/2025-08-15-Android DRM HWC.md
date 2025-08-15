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

通俗来说，Composer 像是“前台接待”，帮你建立一个会话，ComposerClient 像是“专属客服”，提供具体服务。

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

ComposerClient中提供了一大堆服务

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



## 0x00 Reference

[Android display pipeline]: https://lwn.net/Articles/809545/
[Android HWC intro]: https://source.android.com/docs/core/graphics/hwc

