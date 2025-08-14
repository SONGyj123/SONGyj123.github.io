[TOC]

# Linux - Graphic stack

![img](https://i-blog.csdnimg.cn/blog_migrate/b8530def6ef20500472895d5ff25f45b.png)

一个具有显存的虚拟GPU和ARMv8 CPU，有一个DMA可以在RAM和GPU显存之间来回搬运数据

## 0x00 ENV setup

### 1. 编译kernel

随便选择一版Linux源码

```bash
# 使用/arch/arm64/中的defconfig编译内核
# 这是一个aarch64通用的defconfig
export ARCH=arm64
export CROSS_COMPILE=aarch64-linux-gnu-
make defconfig
make -j12

# 如果没有export
# 需要两次命令中都有 ARCH=arm64 CROSS_COMPILE=aarch64-buildroot-linux-gnu-
# make ARCH=arm64 defconfig CROSS_COMPILE=aarch64-buildroot-linux-gnu-后直接make -j12会需要重复选择各种config
make ARCH=arm64 defconfig CROSS_COMPILE=aarch64-buildroot-linux-gnu-
make ARCH=arm64 CROSS_COMPILE=aarch64-buildroot-linux-gnu- -j12
```

### 2. 编译qemu-system-aarch64

随便选择一版QEMU源码，尽量將./configure --target-list=aarch64-softmmu之后输出的各种支持都装上主要是virtio 图形 网络的一些支持包

```bash
./configure --target-list=aarch64-softmmu
cd ./build
make -j12 qemu-system-aarch64
```

需要用到virtio相关

```bash
qemu-system-aarch64 \
    -M virt \
    -cpu cortex-a53 \
    -kernel /home/songyj/embedded/buildroot-2024.08.2/output/images/Image \
    -drive file=/home/songyj/embedded/buildroot-2024.08.2/output/images/rootfs.ext4,format=raw \
    -append "root=/dev/vda nokaslr ip=dhcp" \
    -device virtio-gpu-gl \
    -display gtk,gl=core \
    -device virtio-net-pci,netdev=net0 \
    -serial mon:stdio \ # 串口输出重定向到当前的终端
    # gdb调试时加入下面的switch 会停止在kernel加载的地方
    # -s -S
```

### 3. buildroot准备根文件系统

### 4. 环境验证

### 5. 命令

```bash
# qemu上linux获取交叉编译的应用程序
tftp 10.0.2.2 -c get drm_test

# gdb调试kernel panic
gdb-multiarch vmlinux
# 在gdb内部使用
target remote localhost：1234
b 错误函数
```

### 6. 问题

- ***nvim clangd返回-32001 Invalid AST***

在linux项目顶层添加.clangd文件，内容如下：

```bash
CompileFlags:
  Remove: [-march=*, -mabi=*]
```

参见https://github.com/clangd/clangd/issues/1582

- ***nvim clangd突然无法跳转 并显示一大堆错误***

应该正确使用clangd生成脚本，compile_commands.json生成在源码根目录

```
# /home/songyj/embedded/linux
python3 ./scripts/clang-tools/gen_compile_commands.py
```

随后打开文件，让clangd更新条目

参见https://blog.csdn.net/Roger_Spencer/article/details/135325340

- ***QEMU串口终端不能滚动***

```bash
# 將串口输出重定向到当前的终端
-serial mon:stdio 
```

https://fadeevab.com/how-to-setup-qemu-output-to-console-and-automate-using-shell-script/

## 0x01 DRM

> The scope of DRM has been expanded over the years to cover more functionality previously handled by user-space programs, such as framebuffer managing and [mode setting](https://en.wikipedia.org/wiki/Mode_setting), memory-sharing objects and memory synchronization.

DRM最开始是为应用程序使用GPU而设计的子系统，但是随着DRM子系统的发展，加入了越来越多的模块比如：KMS和GEM

- **KMS（kernel mode-setting）** a combination of [screen resolution](https://en.wikipedia.org/wiki/Screen_resolution), [color depth](https://en.wikipedia.org/wiki/Color_depth) and [refresh rate](https://en.wikipedia.org/wiki/Refresh_rate) 曾今在user space完成设置，现在在kernel space解决多进程并发等问题。

> To avoid breaking backwards compatibility of the DRM API, Kernel Mode-Setting is provided as an additional *driver feature* of certain DRM drivers.

为了兼容之前的DRM驱动，维持一致性，KMS作为一个属性被嵌入到DRM driver中，如果一个DRM driver支持KMS操作，需要置位**DRIVER_MODESET**，这些支持KMS的DRM驱动被叫做KMS driver和legacy的DRM驱动区别开，KMS驱动的支持程度非常高，哪怕这个驱动中没有实现DRM操作只有KMS操作（比如在不支持3D rendering的硬件上）

![img](https://i-blog.csdnimg.cn/blog_migrate/c0131052a585d4c497f612384d98ead4.png)

**KMS**所管理的display pipeline：

1. **FrameBuffer** 存储图像数据的内存区域，它是显存的一部分，包含了需要显示到屏幕上的像素数据。

2. **Planes** 不是任何硬件，它表示一个memory object containing a buffer，这个buffer随后会被送入CRTC

3. **CRTC** 控制屏幕输出的硬件模块，它决定了图像的显示方式（包括位置、大小、刷新率等）。

4. **Connector** 显示器或屏幕的接口，负责处理显示输出信号的连接。（HDMI、DP、VGA）

5. **Encoder** 将图像信号从 GPU 输出到显示器（通过 Connector）的硬件模块，将来自 CRTC 的图像数据转换为符合显示器要求的信号格

   式。

|                     KMS display pipeline                     |                     KMS Output Pipeline                      |
| :----------------------------------------------------------: | :----------------------------------------------------------: |
| ![KMS Display Pipeline](https://www.kernel.org/doc/html/latest/_images/DOT-dade12aa9127c64406e41cdf8d7f80694c134db2.svg) | ![KMS Output Pipeline](https://www.kernel.org/doc/html/latest/_images/DOT-6445c75fc4859992454fd377127d4d309e82f09a.svg) |



- **GEM（Graphics Execution Manager）**

在/dev/dri中，为什么有两种形式的节点？他们的区别？

> Each GPU detected by DRM is referred to as a *DRM device*, and a device file `/dev/dri/card*X*` (where *X* is a sequential number) is created to interface with it. User-space programs that want to talk to the GPU must [open](https://en.wikipedia.org/wiki/Open_(system_call)) this file and use [ioctl](https://en.wikipedia.org/wiki/Ioctl) calls to communicate with DRM. Different ioctls correspond to different functions of the DRM [API](https://en.wikipedia.org/wiki/Application_programming_interface).

- /dev/dri/cardX  这种节点被叫做**primary node**，最开始被用来完成both privileged (modesetting, other display control) and non-privileged (rendering, [GPGPU](https://en.wikipedia.org/wiki/GPGPU) compute) operations
- /dev/dri/renderD128 这种节点的出现为了解决单一primary node的不便（there must always be a running graphics server (the X Server, a Wayland compositor, ...) acting as DRM-Master of a DRM device so that other user space programs can be granted the use of the device, even in cases not involving any graphics display like GPGPU computations），將non-privileged的操作独立出来，因此操作/dev/dri/renderD128只能使用a subset of DRM api

https://en.wikipedia.org/wiki/Direct_Rendering_Manager

```c
// 部分重要成员
struct drm_device {
    struct device *dev;
    
    const struct drm_driver *driver;			// driver
	struct drm_minor *primary, *render, *accel; // 分别对应三种不同的功能 accel对应的应该是GPGPU
	u32 driver_features; 						// driver支持的功能 DRIVER_MODESET DRIVER_GEM等
	
    struct drm_mode_config mode_config;			// 管理了kms中的几乎所有硬件节点 property func
												// crtc encoder connector plane都被链表管理起来供驱动选择
    											// 包括drm_mode_config_funcs：fb_create atomic_commit等
}
```



### 1. libdrm

在深入DRM框架之前，需要先看一下libdrm中如何用代码描述pipeline。或许一开始难以理解，但是回过头来就变得顺其自然。

#### (1) DRM器件

在使用之前我们需要获取到当前这些drm器件的信息，通过drmModeResPtr:

```c
drmModeRes *resources = drmModeGetResources("/dev/dri/car0");
```

##### **drmModeRes**

drmModeResPtr指向的结构体中包含多个drm器件，表示当前pipeline中存在的各种资源

```c
typedef struct _drmModeRes {
	int count_fbs;
	uint32_t *fbs;
	
	int count_crtcs;
	uint32_t *crtcs;
	
	int count_connectors;
	uint32_t *connectors;
	
	int count_encoders;
	uint32_t *encoders;
	
	uint32_t min_width, max_width;
	uint32_t min_height, max_height;
} drmModeRes, *drmModeResPtr;
```

##### **drmModeConnector**

connector是最先被用的部件，mode用于描述屏幕的分辨率和刷新率等，mode数组按照显示最优到最差排列，还包含一些properties。mode中支持的宽高在后续会被用于创建framebuffer。

```c
typedef struct _drmModeConnector {
	uint32_t connector_id;
	uint32_t encoder_id;		// 当前连接的encoder id
	uint32_t connector_type;	// DRM_MODE_CONNECTOR_VGA || DRM_MODE_CONNECTOR_HDMIA
	uint32_t connector_type_id;
	drmModeConnection connection;
	uint32_t mmWidth, mmHeight; /**< HxW in millimeters */
	drmModeSubPixel subpixel;

	int count_modes;
	drmModeModeInfoPtr modes; // mode用于描述屏幕的分辨率刷新率等信息

	int count_props;
	uint32_t *props; /**< List of property ids */
	uint64_t *prop_values; /**< List of property values */

	int count_encoders;
	uint32_t *encoders; /**< List of encoder ids */
} drmModeConnector, *drmModeConnectorPtr;

typedef struct _drmModeModeInfo {
	uint32_t clock;
	uint16_t hdisplay, hsync_start, hsync_end, htotal, hskew;
	uint16_t vdisplay, vsync_start, vsync_end, vtotal, vscan;

	uint32_t vrefresh;

	uint32_t flags;
	uint32_t type;
	char name[DRM_DISPLAY_MODE_LEN];
} drmModeModeInfo, *drmModeModeInfoPtr;
```

##### **drmModeFB**

```
typedef struct _drmModeFB {
	uint32_t fb_id;
	uint32_t width, height;	// framebuffer的大小
	uint32_t pitch;
	uint32_t bpp;
	uint32_t depth;

	uint32_t handle;		// buffer handle用于传入内核查找到实际的内存
} drmModeFB, *drmModeFBPtr;
```

##### **drmModeCrtc**

crtc用于扫描framebuffer，虽然framebuffer与buffer handle结合在一起已经知道想要显示的内容，但是需要crtc控制实际显示区域的大小与位置，不一定整个fb都会显示在屏幕上。

![img](https://i-blog.csdnimg.cn/blog_migrate/c1f6c3fdeef9d3571a45e46580c7379b.png)

```
typedef struct _drmModeCrtc {
	uint32_t crtc_id;
	uint32_t buffer_id;		// fb id

	uint32_t x, y;			// 位置（x，y）
	uint32_t width, height;	// 扫描大小
	int mode_valid;
	drmModeModeInfo mode;

	int gamma_size; /**< Number of gamma stops */

} drmModeCrtc, *drmModeCrtcPtr;
```

##### **drmModeEncoder**

根据显示pipeline，前面我们已经有了显示的内容（framebuffer）和位置大小（crtc）以及显示器支持的协议（connector），似乎还缺少将framebuffer中的像素按照相应协议封装的部件。encoder的工作就是通过协议封装像素

这里面需要特别说明的是**uint32_t possible_crtcs**，这是一个掩码。最开始获取pipeline资源的时候得到的**drmModeRes**包含一个crtc列表，如果bit0被置位1，则表示drmModeRes->crtc[0]与当前的encoder是匹配的。 

```
typedef struct _drmModeEncoder {
	uint32_t encoder_id;
	uint32_t encoder_type;
	uint32_t crtc_id;
	uint32_t possible_crtcs;
	uint32_t possible_clones;
} drmModeEncoder, *drmModeEncoderPtr;
```

- [ ] 如何知道匹配



#### (2) 用户空间配置显示pipeline的步骤

![](/home/songyj/note/SONGyj123.github.io/image/display_pipeline.svg)

1. **drmModeGetResources获取pipeline资源**
2. **drmModeGetConnector获取connector**
3. **drmModeGetEncoder获取encoder** connector中有encoder列表
4. **选取可用的crtc** 逐位检查crtc掩码，寻找可用crtc
5. **申请内存用于创建显示buffer**
6. **创建framebuffer并绑定上一步申请的buffer**

上述步骤执行完后，显示pipeline就配置完成，理论上就可以进行显示了。这上面的每一步都是一个系统调用，后面我们会进一步分析内核如何返回结果。

```c
#define _GNU_SOURCE
#include <errno.h>
#include <fcntl.h>
#include <stdbool.h>
#include <stdint.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/mman.h>
#include <time.h>
#include <unistd.h>
#include <xf86drm.h>
#include <xf86drmMode.h>

struct buffer_object {
	uint32_t width;
	uint32_t height;
	uint32_t pitch;
	uint32_t handle;
	uint32_t size;
	uint8_t *vaddr;
	uint32_t fb_id;
};

struct buffer_object buf;

static int modeset_create_fb(int fd, struct buffer_object *bo)
{
	struct drm_mode_create_dumb create = {};
 	struct drm_mode_map_dumb map = {};

	/* create a dumb-buffer, the pixel format is XRGB888 */
	create.width = bo->width;
	create.height = bo->height;
	create.bpp = 32;
	drmIoctl(fd, DRM_IOCTL_MODE_CREATE_DUMB, &create);

	/* bind the dumb-buffer to an FB object */
	bo->pitch = create.pitch;
	bo->size = create.size;
	bo->handle = create.handle;
	drmModeAddFB(fd, bo->width, bo->height, 24, 32, bo->pitch,
			   bo->handle, &bo->fb_id);

	/* map the dumb-buffer to userspace */
	map.handle = create.handle;
	drmIoctl(fd, DRM_IOCTL_MODE_MAP_DUMB, &map);

	bo->vaddr = mmap(0, create.size, PROT_READ | PROT_WRITE,
			MAP_SHARED, fd, map.offset);

	/* initialize the dumb-buffer with white-color */
	memset(bo->vaddr, 0xff, bo->size);

	return 0;
}

static void modeset_destroy_fb(int fd, struct buffer_object *bo)
{
	struct drm_mode_destroy_dumb destroy = {};

	drmModeRmFB(fd, bo->fb_id);

	munmap(bo->vaddr, bo->size);

	destroy.handle = bo->handle;
	drmIoctl(fd, DRM_IOCTL_MODE_DESTROY_DUMB, &destroy);
}

int main(int argc, char **argv)
{
	int fd;
	drmModeConnector *conn;
	drmModeRes *res;
	uint32_t conn_id;
	uint32_t crtc_id;

	fd = open("/dev/dri/card0", O_RDWR | O_CLOEXEC);

	res = drmModeGetResources(fd);
	crtc_id = res->crtcs[0];
	conn_id = res->connectors[0];

	conn = drmModeGetConnector(fd, conn_id);
	buf.width = conn->modes[0].hdisplay;
	buf.height = conn->modes[0].vdisplay;

	modeset_create_fb(fd, &buf);

	drmModeSetCrtc(fd, crtc_id, buf.fb_id,
			0, 0, &conn_id, 1, &conn->modes[0]);

	getchar();

	modeset_destroy_fb(fd, &buf);

	drmModeFreeConnector(conn);
	drmModeFreeResources(res);

	close(fd);

	return 0;
}
```



#### (3) atomic接口

atomic关键词如何理解？

所有的property配置完成后进行commit，commit如果失败，所有的配置恢复到commit之前

atomic接口引入property，这些property的引入简化驱动层IOCTL函数

![img](https://i-blog.csdnimg.cn/blog_migrate/2df1c788392fd8240a9637afad6f1d6b.png)

crtc中的特性out_fence_ptr，HWC中的present fence就是他

***为什么在简单的 DRM 应用程序中不需要显式配置 encoder？***



### 2. DRM driver

![](file:///home/songyj/Pictures/Screenshots/Screenshot%20from%202024-12-15%2009-10-39.png)

DRM驱动其实不是一整个驱动而是图形栈各硬件驱动都被插入到DRM框架中，比如有kms驱动，gem驱动以及GPU驱动，其中KMS驱动中会包含有操作硬件的部分，这些硬件操作会被嵌入相应的drm模块中，

render和display是区别开的

libdrm中提供的接口是DRM driver暴露出来的，每一个DRM驱动的核心都是**struct drm_driver**。对于Linux驱动而言，一般都是一个结构体，随后通过register函数往总线或者驱动框架注册，DRM驱动也是一样：

1. [`drm_dev_alloc`](https://www.kernel.org/doc/html/v4.11/gpu/drm-internals.html#c.drm_dev_alloc) 为**struct drm_driver**分配内存
2. 填充**struct drm_driver**各成员
3. [`drm_dev_register`](https://www.kernel.org/doc/html/v4.11/gpu/drm-internals.html#c.drm_dev_register) 注册**struct drm_driver**到DRM驱动框架

**struct drm_driver**中的fops提供.unlocked_ioctl=drm_ioctl，使用drm core中的函数来处理用户空间的ioctl请求。

```
struct drm_driver
{
	一系列成员函数指针;
	
	.fops; // 这里面包含ioctl ioctl的函数可能最后还是调用到上面的成员函数 比如create_dumb
}

// This creates a new dumb buffer in the driver's backing storage manager (GEM, TTM or something else entirely) and // returns the resulting buffer handle. This handle can then be wrapped up into a framebuffer modeset object.
// 比如create_dumb接受ioctl传递的buffer参数（width height colorformat）凭这些参数计算buffer大小并申请
// framebuffer是buffer handle+metadata
```

drm_atomic_helper_commit_planes

初始化display pipeline的顺序很重要：connector->encoder->plane->crtc

drmModeSetCrtc ioctl的调用路径：

1. drm_mode_setcrtc
2. drm_atomic_helper_set_config
3. drm_atomic_commit
4. drm_atomic_helper_commit
5. drm_atomic_helper_commit_tail
6. drm_atomic_helper_commit_planes
7. plane.private->atomic_update



drm_atomic_helper_check_modset



### 3. KMS（Kernel Mode Setting）

DRM driver必须调用[`drm_mode_config_init`](https://www.kernel.org/doc/html/v4.11/gpu/drm-kms.html#c.drm_mode_config_init)

对**struct drm_driver**中的***driver_feature***进行置位，告诉DRM驱动框架当前驱动的能力，常见的标志位如下：

- **DRIVER_MODESET** 支持modesetting操作
- **DRIVER_GEM** 支持内存分配
- **DRIVER_ATOMIC** 支持atomic操作
- **DRIVER_RENDER** 支持渲染

调用fb_create的ioctl是**DRM_IOCTL_MODE_ADDFB2**，fb_create將gem obj handle和fb绑定，没有绑定handle的fb没有任何实际的显示内容

#### (1) ioctl接口

##### ioctl解析分发

drm框架中有一个ioctl数组，这里面把DRM_IOCTL_MODE_ADDFB2宏定义和drm_mode_addfb2_ioctl配对起来，使得用户空间可以直接调用到实际的函数。

- [ ] DRM_IOCTL_DEF具体原理

```c
// drm_ioctl.c
static const struct drm_ioctl_desc drm_ioctls[] = {
    ...
    DRM_IOCTL_DEF(DRM_IOCTL_MODE_ADDFB2, drm_mode_addfb2_ioctl, 0),
    ...
}
```

##### DRM_IOCTL_MODE_CREATE_DUMB

1. DRM_IOCTL_DEF(DRM_IOCTL_MODE_CREATE_DUMB, drm_mode_create_dumb_ioctl, 0)
2. drm_mode_create_dumb_ioctl调用drm_mode_create_dumb
3. drm_mode_create_dumb检查函数指针和入参是否合法后调用**dev->driver->dumb_create**

根据上述内容，create_dumb实际最后会调用到drm_driver中的dumb_create成员函数

#### 调试

临时开启drm中的打印

```bash
# 查看当前的debug日志等级
songyj@raspberrypi:~/driver/try_i2c $ sudo cat /sys/module/drm/parameters/debug
0

# 启用 KMS 调试（0x04）
echo 0x04 | sudo tee /sys/module/drm/parameters/debug
# 或者启用 KMS + ATOMIC 调试（0x04 + 0x10 = 0x14）
echo 0x14 | sudo tee /sys/module/drm/parameters/debug
# 查看当前值（确认是否生效）
cat /sys/module/drm/parameters/debug
```



### 4. GEM（Graphics Execution Manager）

GEM（）作为比ttm更简单的内存管理器，GEM本身通过shmem来分配buffer，shmem分配出的内存不一定是在物理内存上连续的，因此当希望使用物理内存连续的buffer时（DMA搬运需要连续的物理内存），drm驱动需要自己来申请buffer。只需要将gem_object内置在自定结构体中，GEM也可以进行管理。

> Anonymous pageable memory allocation is not always desired, for instance when the hardware requires physically contiguous system memory as is often the case in embedded devices. Drivers can create GEM objects with no shmfs backing (called private GEM objects) by initializing them with a call to drm_gem_private_object_init() instead of drm_gem_object_init(). Storage for private GEM objects must be managed by drivers.

例如exynos dpu中，drm_gem_object只作为一个管理标记嵌入到自定义的内存结构体，实际的内存分配不通过GEM进行

```c
struct exynos_drm_gem {
	struct drm_gem_object	base;
	unsigned int		flags;
	unsigned long		size;
	void			*cookie;
	void			*kvaddr;
	dma_addr_t		dma_addr;
	unsigned long		dma_attrs;
	struct sg_table		*sgt;
};
```

#### GEM handle

drm_gem_object很大一块，如果可以通过一个数字一样的index和drm_gem_object对应起来，就能快速找到平匹配的object，而传递的时候只需要传递这个index就好了。

这个index就是gem handle，驱动中通过drm_gem_handle_create进行drm_gem_object->gem_handle的转换。handle的数值怎么生成的呢？通过kernel中的idr分配

#### GEM mapping

- [ ] todo

### 5. dma-buf

dma-buf是一种共享机制，主要为了解决不同驱动之间buffer共享问题，在显示pipeline中，一块显示的buffer会经过多个模块的处理最后才能显示出来。



dma-buf在内核空间和用户空间中的表显形式不一样，在用户空间中是一个**fd**，在内核中是**struct dma_buf**

- No standardised memory allocation: It’s up to the exporter; 
- An arbitrary buffer can be wrapped into a DMA buffer; 
- An fd can be returned to the userspace to reference this buffer; 
- The fd can be passed to another process; 
- The fd can be mmapped and accessed by the CPU; 
- The fd can be imported by any driver supporting the user role

场景：

GPU 和显示控制器共享帧缓冲，GPU 使用 GEM 分配显存来存储绘制的图形帧。

GPU 驱动通过 `dma_buf_export` 将 GEM 对象导出为 dma-buf。

显示控制器驱动通过 `dma_buf_attach` 访问该缓冲区，并将内容直接显示。

下面列出一对用于在dma-buf和gem handle之间转换的ioctl

- drm_prime_fd_to_handle
- drm_prime_handle_to_fd

```
// 查看dma heap节点
ls /dev/dma_heap

struct dma_heap_allocation_data heap_data = {
    .len = 1048576, // 1meg
    .fd_flags = O_RDWR | O_CLOEXEC,
};

fd = open(“/dev/dma_heap/system”, O_RDONLY | O_CLOEXEC);
if (fd < 0) return fd;
ret = ioctl(fd, DMA_HEAP_IOCTL_ALLOC, &heap_data)
if (ret) return ret;
return heap_data.fd;
```



##### dma-buf heap（dmaheap）

作为分配能够申请分配dma-buf的驱动向用户空间提供接口，实际就是一个exporter驱动。

[dma-buf](https://www.youtube.com/watch?v=UsEVoWD_o0c)

#### helper function

##### helper function如何使用？

```c
// 定义某个helper function具体的操作
static void exynos_plane_atomic_update(struct drm_plane *plane, struct drm_atomic_state *state)
{
	struct drm_plane_state *new_state = drm_atomic_get_new_plane_state(state,
								           plane);
	struct exynos_drm_crtc *exynos_crtc = to_exynos_crtc(new_state->crtc);
	struct exynos_drm_plane *exynos_plane = to_exynos_plane(plane);

	if (!new_state->crtc)
		return;

	if (exynos_crtc->ops->update_plane)
		exynos_crtc->ops->update_plane(exynos_crtc, exynos_plane); // 硬件操作
}

// 定义某个硬件helper function集合
static const struct drm_plane_helper_funcs plane_helper_funcs = {
	.atomic_check = exynos_plane_atomic_check,
	.atomic_update = exynos_plane_atomic_update,
	.atomic_disable = exynos_plane_atomic_disable,
};

// 注册helper function到drm_plane
// 实际plane->helper_private = funcs;
drm_plane_helper_add(&exynos_plane->base, &plane_helper_funcs);
```

#### scatterlist

散列表的结构体如下，其中scatterlist可以被分成三种：一般，铰链，结束，这三种类型由page_link的最低两位决定

offset和length分别表示内存块在page上的偏移位置和内存长度

dma_address表示PA物理地址

**每一个scatterlist可以看作一个内存块，每一个sg_table则是一组链表，其中链表的每一个元素都是一个scaatterlist数组**

```
struct scatterlist {
	unsigned long	page_link;
	unsigned int	offset;
	unsigned int	length;
	dma_addr_t	dma_address;
#ifdef CONFIG_NEED_SG_DMA_LENGTH
	unsigned int	dma_length;
#endif
#ifdef CONFIG_NEED_SG_DMA_FLAGS
	unsigned int    dma_flags;
#endif
};

struct sg_table {
	struct scatterlist *sgl;	/* the list */
	unsigned int nents;		/* number of mapped entries */
	unsigned int orig_nents;	/* original size of list */
};
```

散列表遍历

```
#define for_each_sg(sglist, sg, nr, __i)	\
	for (__i = 0, sg = (sglist); __i < (nr); __i++, sg = sg_next(sg))
	
struct scatterlist *sg_next(struct scatterlist *sg)
{
	if (sg_is_last(sg))
		return NULL;

	sg++;
	if (unlikely(sg_is_chain(sg)))
		sg = sg_chain_ptr(sg);

	return sg;
}
EXPORT_SYMBOL(sg_next);

```

#### 两种显存传递方案

我们来思考一个问题，显存中的内容来自用户空间的应用程序，如何将他们传递到内核空间中去？两种办法：

1. 使用mmap进行内存映射 DRM_IOCTL_MODE_CREATE_DUMB创建  DRM_IOCTL_MODE_MAP_DUMB计算偏移 随后mmap
2. 使用dmaheap进行export import DMA_HEAP_IOCTL_ALLOC分配 DRM_IOCTL_PRIME_FD_TO_HANDLE进行export

#### 问题

**是否硬件相关的操作都要放在helper函数中？**

https://www.kernel.org/doc/html/v4.11/gpu/index.html

**fake_gpu 0000:00:01.0: BAR 0 [mem 0x00000000-0x0000001f]: not claimed; can't enable device**

## 0x02 QEMU - fake GPU device

在QEMU中引入一个新的虚拟GPU挂载到PCI总线

```c
#include "qemu/osdep.h"
#include "qom/object.h"
#include "ui/console.h"
#include "hw/pci/pci_device.h"

typedef struct Fake_GPU_State Fake_GPU_State;

#define TYPE_FAKE_GPU "fake_gpu"
OBJECT_DECLARE_SIMPLE_TYPE(Fake_GPU_State, FAKE_GPU)

#define REG_NUM     8
#define REG_SIZE    4

typedef struct Fake_GPU_State
{
    PCIDevice parent_obj;

    MemoryRegion bar;
    uint32_t regs[REG_NUM];
}Fake_GPU_State;

static void fake_gpu_write(void *opaque, hwaddr addr, uint64_t value, unsigned size)
{
    Fake_GPU_State *s = FAKE_GPU(opaque);
    printf("wr\r\n");
    s->reg0 = value;
}

static uint64_t fake_gpu_read(void *opaque, hwaddr addr, unsigned size)
{
    Fake_GPU_State *s = FAKE_GPU(opaque);
    printf("rd\r\n");
    return s->reg0;
}

static const MemoryRegionOps fake_gpu_ops = {
    .write = fake_gpu_write,
    .read = fake_gpu_read,
};

static void fake_gpu_realize(PCIDevice *dev, Error **errp)
{
    Fake_GPU_State *s = FAKE_GPU(dev);

    printf("realize start\r\n");

    memory_region_init_io(&s->bar, OBJECT(s), &fake_gpu_ops, s, "fake_gpu", 32);
    pci_register_bar(dev, 0, PCI_BASE_ADDRESS_SPACE_MEMORY, &s->bar);
}

static void fake_gpu_exit(PCIDevice *dev)
{
    return ;
}

static void fake_gpu_class_init(ObjectClass *klass, void *data)
{
    //DeviceClass *dc = DEVICE_CLASS(klass);
    PCIDeviceClass *k = PCI_DEVICE_CLASS(klass);

    k->realize = fake_gpu_realize;
    k->exit = fake_gpu_exit;
    k->vendor_id = 0x1234;
    k->device_id = 0xbeef;
    k->class_id = PCI_CLASS_OTHERS; // 一定要有否则报错 BAR0 not claimed
}

static const TypeInfo fake_gpu_info = {
    .name = TYPE_FAKE_GPU,
    .parent = TYPE_PCI_DEVICE,
    .instance_size = sizeof(Fake_GPU_State),
    .class_init    = fake_gpu_class_init,
    .interfaces = (InterfaceInfo[]) {
        { INTERFACE_CONVENTIONAL_PCI_DEVICE },
        { },
    }
};

static void fake_gpu_device_register_types(void)
{
    type_register_static(&fake_gpu_info);
}

type_init(fake_gpu_device_register_types);

```

QEMU启动相关配置：

```bash
qemu-system-aarch64 \
    -M virt \
    -cpu cortex-a53 \
    -kernel /home/songyj/embedded/buildroot-2024.08.2/output/images/Image \
    -drive file=/home/songyj/embedded/buildroot-2024.08.2/output/images/rootfs.ext4,format=raw \
    -append "root=/dev/vda nokaslr ip=dhcp" \
    -device virtio-gpu-gl \
    -display gtk,gl=core \
    -device virtio-net-pci,netdev=net0 \
    -netdev user,id=net0,tftp=/home/songyj/embedded/tftp_qemu \
    -device fake_gpu
```

确认新的pci设备编译到QEMU中：

```bash
# 编译完成之后通过qemu-system-aarch64查看设备信息
songyj@songyj-HP:~/embedded$ qemu-system-aarch64 -device help | grep fake_gpu
name "fake_gpu", bus PCI

# 在虚拟机中使用lspci也能看到相应的设备信息
```

[QEMU display](https://www.linux-kvm.org/images/1/1b/02x04-Aspen-Gerd_Hoffmann-QEMU_and_OpenGL.pdf)

QEMU添加显示支持：

```c
typedef struct Fake_GPU_State
{
    PCIDevice parent_obj;

    QemuConsole* console;

    MemoryRegion bar;
    uint32_t regs[REG_NUM];

    void *pixels;
}Fake_GPU_State;
```

```c
// 显示测试函数
static void fill_screen_with_yellow(uint32_t *buffer) {
    uint32_t *framebuffer = buffer;
    int total_pixels = 1024 * 768;

    // 黄色的 ARGB 格式
    uint32_t yellow_color = 0x00FFFF00;

    // 使用 memset 直接填充内存
    for (int i = 0; i < total_pixels; i++) {
        framebuffer[i] = yellow_color;
    }
}

// 初始化屏幕
static void fake_gpu_realize(PCIDevice *dev, Error **errp)
{
    Fake_GPU_State *s = FAKE_GPU(dev);

    printf("realize start\r\n");

    memory_region_init_io(&s->bar, OBJECT(s), &fake_gpu_ops, s, "fake_gpu", REG_NUM*REG_SIZE);
    pci_register_bar(dev, 0, PCI_BASE_ADDRESS_SPACE_MEMORY, &s->bar);

    s->pixels = malloc(PIXS_SIZE); // 为像素分配空间
    memset(s->pixels, 0, sizeof(PIXS_SIZE));

    s->console = graphic_console_init(DEVICE(dev), 0, &fake_gpu_graphic_ops, s);

    DisplaySurface* ds = qemu_create_displaysurface_from(1024, 768, PIXMAN_a8r8g8b8, 1024 * 4, s->pixels);
    fill_screen_with_yellow(s->pixels); // 初始化屏幕为黄色
    dpy_gfx_replace_surface(s->console, ds); // 替换画布
    dpy_gfx_update_full(s->console); // 显示
}
```

https://blog.davidv.dev/posts/pcie-driver-dma/

## 0x03 Linux - fake GPU device driver

```c
#include <linux/module.h>
#include <linux/pci.h>

#define DRV_NAME    "fake_gpu"
#define PCI_VENDOR_ID_FAKE_GPU    0x1234
#define PCI_DEVICE_ID_FAKE_GPU    0xbeef

volatile uint32_t __iomem *regs;

static const struct pci_device_id pci_table[] = {
    { PCI_DEVICE(PCI_VENDOR_ID_FAKE_GPU, PCI_DEVICE_ID_FAKE_GPU)},
    { },
};

static void fake_gpu_remove(struct pci_dev *pdev)
{

}

static int fake_gpu_probe(struct pci_dev *pdev, const struct pci_device_id *ent)
{
    printk("This is a pci driver");
 	int ret;

	if (!pdev) 
	{
		printk("Invalid pci device pointer\n");
		return -EINVAL;
	}

	ret = pci_enable_device(pdev);
	if(ret)
	{
		printk("pci_enable_devices failed");
		return ret;
	}
	else
	{
		printk("nice try");
	}

	ret = pci_request_regions(pdev, DRV_NAME);
	if(ret)
	{
		printk("pci_request_regions failed");
	}

	resource_size_t pciaddr = pci_resource_start(pdev, 0);

	regs = ioremap(pciaddr, 32);

	    u32 reg_value = readl(regs);
    printk("Register value: 0x%x\n", reg_value);

    // 写入寄存器
    writel(0xDEADBEEF, regs);
    printk("Written value: 0xDEADBEEF\n");

	    reg_value = readl(regs);
    printk("Register value: 0x%x\n", reg_value);

	return ret;
}

static struct pci_driver fake_gpu_driver = {
    .name = DRV_NAME,
    .id_table = pci_table,
    .probe = fake_gpu_probe,
    .remove = fake_gpu_remove,
};

module_pci_driver(fake_gpu_driver);
```

### 调试

（1）

应用程序在drmModeGetConnector(fd, res->connectors[0])获取属性时报错解引用空指针，并打印如下调用栈：

```c
[  369.873125] Call trace:
[  369.873138]  drm_atomic_get_property+0x46c/0x6e8 [drm]
[  369.873356]  __drm_object_property_get_value+0x9c/0xb0 [drm]
[  369.873576]  drm_mode_object_get_properties+0xd8/0x1e0 [drm]
[  369.873793]  drm_mode_getconnector+0x258/0x4b0 [drm]
[  369.874009]  drm_ioctl_kernel+0xd8/0x190 [drm]
[  369.874221]  drm_ioctl+0x220/0x4c0 [drm]
[  369.874432]  __arm64_sys_ioctl+0xb4/0x100
[  369.874465]  invoke_syscall+0x50/0x128
[  369.874491]  el0_svc_common.constprop.0+0x48/0xf0
[  369.874516]  do_el0_svc+0x24/0x38
[  369.874539]  el0_svc+0x40/0xe8
[  369.874565]  el0t_64_sync_handler+0x100/0x130
[  369.874584]  el0t_64_sync+0x190/0x198
[  369.874608] Code: 3944b822 17fffff4 92800002 17fffff2 (f9400420) 
[  369.874630] ---[ end trace 0000000000000000 ]---
[ 1905.586649] exit called
[ 1905.586749] ssd1306_remove
[ 1905.589816] ------------[ cut here ]------------
```

排查发现是没有使用drm_mode_config_reset(drm_dev)对connector的state进行初始化，导致后续对state进行解引用出错。drm_mode_config_reset会调用plane crtc encoder以及connector的reset函数对他们进行reset，在上述所有模块都配置完毕后，在drm_dev_register之前完成此操作。

```bash
# 初始化后可以正常获取到connector的各种信息
Available connector nums: 1
connector
Connector ID: 31
Type: Unknown
Type ID: 2
Connection: Connected
Modes: 0
Properties: 5
```

有些必要函数没有初始化在注册drm驱动时会收到warning：

```bash
[ 9221.234651] WARNING: CPU: 3 PID: 1186 at drivers/gpu/drm/drm_plane.c:262 __drm_universal_plane_init+0x360/0x5c8 [drm]
[ 9221.234918] Modules linked in: drm_ssd130x(O+) cmac algif_hash aes_arm64 aes_generic algif_skcipher af_alg bnep brcmfmac_wcc hci_uart vc4 btbcm brcmfmac bluetooth brcmutil cfg80211 snd_soc_hdmi_codec drm_display_helper cec raspberrypi_hwmon drm_dma_helper ecdh_generic ecc rfkill drm_kms_helper libaes binfmt_misc raspberrypi_gpiomem rpivid_hevc(C) bcm2835_v4l2(C) bcm2835_codec(C) bcm2835_isp(C) v4l2_mem2mem snd_soc_core bcm2835_mmal_vchiq(C) vc_sm_cma(C) videobuf2_dma_contig videobuf2_vmalloc snd_compress videobuf2_memops videobuf2_v4l2 joydev snd_bcm2835(C) snd_pcm_dmaengine videodev snd_pcm v3d videobuf2_common gpu_sched snd_timer mc drm_shmem_helper snd nvmem_rmem uio_pdrv_genirq uio drm i2c_dev fuse dm_mod drm_panel_orientation_quirks backlight ip_tables x_tables ipv6 hid_apple i2c_bcm2835 i2c_brcmstb [last unloaded: drm_ssd130x(O)]
[ 9221.235274] CPU: 3 PID: 1186 Comm: insmod Tainted: G        WC O       6.6.51+rpt-rpi-v8 #1  Debian 1:6.6.51-1+rpt3
[ 9221.235291] Hardware name: Raspberry Pi 4 Model B Rev 1.5 (DT)
[ 9221.235298] pstate: 80000005 (Nzcv daif -PAN -UAO -TCO -DIT -SSBS BTYPE=--)
[ 9221.235312] pc : __drm_universal_plane_init+0x360/0x5c8 [drm]
[ 9221.235529] lr : drm_universal_plane_init+0x6c/0xa8 [drm]
[ 9221.235737] sp : ffffffc0817b3700
[ 9221.235744] x29: ffffffc0817b3700 x28: ffffff80413f4010 x27: ffffffd2f1da1328
[ 9221.235770] x26: 0000000000000001 x25: ffffffd2f1da32c8 x24: 0000000000000000
[ 9221.235793] x23: ffffff80413f4fe0 x22: 0000000000000000 x21: ffffff80413f4fe0
[ 9221.235816] x20: 0000000000000001 x19: ffffff80413f4180 x18: ffffffffffffffff
[ 9221.235838] x17: 30302d312f312d63 x16: ffffffd3530a6538 x15: ffffffc0817b3590
[ 9221.235860] x14: ffffff8042632c0a x13: ffffffc0817b3810 x12: ffffffc0817b3818
[ 9221.235881] x11: 0000000000000000 x10: ffffffc0817b3810 x9 : ffffffd2f180d23c
[ 9221.235904] x8 : ffffffc0817b37c0 x7 : 0000000000000001 x6 : 0000000000000000
[ 9221.235925] x5 : 0000000000000001 x4 : ffffffd2f1da32c8 x3 : ffffffd2f1da1328
[ 9221.235947] x2 : 0000000000000000 x1 : 00000000ffffffff x0 : 0000000000000000
[ 9221.235968] Call trace:
[ 9221.235977]  __drm_universal_plane_init+0x360/0x5c8 [drm]
[ 9221.236186]  drm_universal_plane_init+0x6c/0xa8 [drm]
[ 9221.236391]  ssd1306_probe+0x148/0x2b8 [drm_ssd130x]
[ 9221.236421]  i2c_device_probe+0x14c/0x2a0
[ 9221.236445]  really_probe+0x150/0x2c0
[ 9221.236460]  __driver_probe_device+0x80/0x140
[ 9221.236472]  driver_probe_device+0xe0/0x170
[ 9221.236484]  __driver_attach+0x9c/0x1b0
[ 9221.236496]  bus_for_each_dev+0x80/0xe8
[ 9221.236507]  driver_attach+0x2c/0x40
[ 9221.236518]  bus_add_driver+0xec/0x218
[ 9221.236528]  driver_register+0x68/0x138
[ 9221.236541]  i2c_register_driver+0x50/0xd8
[ 9221.236556]  ssd1306_drv_init+0x60/0xff8 [drm_ssd130x]
[ 9221.236583]  do_one_initcall+0x60/0x2c0
[ 9221.236597]  do_init_module+0x60/0x218
[ 9221.236614]  load_module+0x1dd0/0x2080
[ 9221.236630]  init_module_from_file+0x8c/0xd8
[ 9221.236646]  __arm64_sys_finit_module+0x150/0x330
[ 9221.236662]  invoke_syscall+0x50/0x128
[ 9221.236680]  el0_svc_common.constprop.0+0x48/0xf0
[ 9221.236698]  do_el0_svc+0x24/0x38
[ 9221.236713]  el0_svc+0x40/0xe8
[ 9221.236733]  el0t_64_sync_handler+0x100/0x130
[ 9221.236743]  el0t_64_sync+0x190/0x198
[ 9221.236755] ---[ end trace 0000000000000000 ]---
[ 9221.236906] ------------[ cut here ]------------
```

回看kernel代码可知，需要填充atomic_destroy_state以及atomic_duplicate_state：

```c
WARN_ON(drm_drv_uses_atomic_modeset(dev) &&
		(!funcs->atomic_destroy_state ||
		 !funcs->atomic_duplicate_state));
```

又或者是某些初始化步骤遗忘：

```bash
[10710.755324] Bogus possible_crtcs: [ENCODER:32:ssd130x_encoder] possible_crtcs=0x0 (full crtc mask=0x1)
[10710.755398] WARNING: CPU: 2 PID: 1583 at drivers/gpu/drm/drm_mode_config.c:626 drm_mode_config_validate+0x1d0/0x4e8 [drm]
[10710.755627] Modules linked in: drm_ssd130x(O+) cmac algif_hash aes_arm64 aes_generic algif_skcipher af_alg bnep brcmfmac_wcc hci_uart vc4 btbcm brcmfmac bluetooth brcmutil cfg80211 snd_soc_hdmi_codec drm_display_helper cec raspberrypi_hwmon drm_dma_helper ecdh_generic ecc rfkill drm_kms_helper libaes binfmt_misc raspberrypi_gpiomem rpivid_hevc(C) bcm2835_v4l2(C) bcm2835_codec(C) bcm2835_isp(C) v4l2_mem2mem snd_soc_core bcm2835_mmal_vchiq(C) vc_sm_cma(C) videobuf2_dma_contig videobuf2_vmalloc snd_compress videobuf2_memops videobuf2_v4l2 joydev snd_bcm2835(C) snd_pcm_dmaengine videodev snd_pcm v3d videobuf2_common gpu_sched snd_timer mc drm_shmem_helper snd nvmem_rmem uio_pdrv_genirq uio drm i2c_dev fuse dm_mod drm_panel_orientation_quirks backlight ip_tables x_tables ipv6 hid_apple i2c_bcm2835 i2c_brcmstb [last unloaded: drm_ssd130x(O)]
[10710.755972] CPU: 2 PID: 1583 Comm: insmod Tainted: G        WC O       6.6.51+rpt-rpi-v8 #1  Debian 1:6.6.51-1+rpt3
[10710.755986] Hardware name: Raspberry Pi 4 Model B Rev 1.5 (DT)
[10710.755993] pstate: 60000005 (nZCv daif -PAN -UAO -TCO -DIT -SSBS BTYPE=--)
[10710.756006] pc : drm_mode_config_validate+0x1d0/0x4e8 [drm]
[10710.756219] lr : drm_mode_config_validate+0x1d0/0x4e8 [drm]
[10710.756431] sp : ffffffc081d63760
[10710.756437] x29: ffffffc081d63770 x28: ffffffd2f1da17b8 x27: ffffff8041a502c0
[10710.756460] x26: 0000000000000001 x25: ffffff8041a50b78 x24: ffffffd2f1876870
[10710.756483] x23: ffffffd2f18767f0 x22: ffffff8041a50010 x21: ffffff8041a502c0
[10710.756505] x20: 0000000000000001 x19: ffffff8041a502b8 x18: ffffffffffffffff
[10710.756527] x17: 637472635f656c62 x16: 6973736f70205d72 x15: 65646f636e655f78
[10710.756549] x14: 3033316473733a32 x13: 293178303d6b7361 x12: 6d2063747263206c
[10710.756571] x11: 6c75662820307830 x10: ffffffd3540b3710 x9 : ffffffd352b098ec
[10710.756593] x8 : 00000000ffffefff x7 : ffffffd3540b3710 x6 : 80000000fffff000
[10710.756615] x5 : ffffff807fb8ad48 x4 : 0000000000000000 x3 : 0000000000000027
[10710.756636] x2 : 0000000000000000 x1 : 0000000000000000 x0 : ffffff80447c3d80
[10710.756657] Call trace:
[10710.756665]  drm_mode_config_validate+0x1d0/0x4e8 [drm]
[10710.756877]  drm_dev_register+0x198/0x268 [drm]
[10710.757088]  ssd1306_probe+0x190/0x2b8 [drm_ssd130x]
[10710.757117]  i2c_device_probe+0x14c/0x2a0
[10710.757136]  really_probe+0x150/0x2c0
[10710.757149]  __driver_probe_device+0x80/0x140
[10710.757161]  driver_probe_device+0xe0/0x170
[10710.757173]  __driver_attach+0x9c/0x1b0
[10710.757184]  bus_for_each_dev+0x80/0xe8
[10710.757195]  driver_attach+0x2c/0x40
[10710.757206]  bus_add_driver+0xec/0x218
[10710.757216]  driver_register+0x68/0x138
[10710.757229]  i2c_register_driver+0x50/0xd8
[10710.757243]  ssd1306_drv_init+0x60/0xff8 [drm_ssd130x]
[10710.757269]  do_one_initcall+0x60/0x2c0
[10710.757281]  do_init_module+0x60/0x218
[10710.757298]  load_module+0x1dd0/0x2080
[10710.757313]  init_module_from_file+0x8c/0xd8
[10710.757329]  __arm64_sys_finit_module+0x150/0x330
[10710.757344]  invoke_syscall+0x50/0x128
[10710.757362]  el0_svc_common.constprop.0+0x48/0xf0
[10710.757378]  do_el0_svc+0x24/0x38
[10710.757394]  el0_svc+0x40/0xe8
[10710.757413]  el0t_64_sync_handler+0x100/0x130
[10710.757423]  el0t_64_sync+0x190/0x198
[10710.757434] ---[ end trace 0000000000000000 ]---
```

检查kernel代码，validate_encoder_possible_crtcs()函数会检查encoder中的possible_crtcs的配置情况：

```c
static void validate_encoder_possible_crtcs(struct drm_encoder *encoder)
{
	u32 crtc_mask = full_crtc_mask(encoder->dev);

	WARN((encoder->possible_crtcs & crtc_mask) == 0 ||
	     (encoder->possible_crtcs & ~crtc_mask) != 0,
	     "Bogus possible_crtcs: "
	     "[ENCODER:%d:%s] possible_crtcs=0x%x (full crtc mask=0x%x)\n",
	     encoder->base.id, encoder->name,
	     encoder->possible_crtcs, crtc_mask);
}

// 在modeconfig初始化中加入如下
encoder->possible_crtcs = drm_crtc_mask(crtc);
```

## 0x04 在RPI4B驱动ssd1306LED

问题

connector报错没有添加mode和fb，但是已经添加了fb和mode：

```bash
[ 3035.282877] [drm:drm_mode_setcrtc [drm]] Count connectors is 1 but no mode or fb set
```

检查发现使用drm_add_modes_noedid并没有成功，count应该返回添加了几个mode，实际count返回0，没有添加上。

```c
int count = drm_add_modes_noedid(connector, WIDTH, HEIGHT);
```

### 上板调试问题

- （1）调用drm_mode_setcrtc后发生trap

```bash
[13841.124867] Call trace:
[13841.124881]  drm_crtc_check_viewport+0x68/0xe0 [drm]
[13841.125096]  drm_mode_setcrtc+0x210/0x6c0 [drm]
[13841.125309]  drm_ioctl_kernel+0xd8/0x190 [drm]
[13841.125523]  drm_ioctl+0x220/0x4c0 [drm]
[13841.125735]  __arm64_sys_ioctl+0xb4/0x100
[13841.125768]  invoke_syscall+0x50/0x128
[13841.125794]  el0_svc_common.constprop.0+0x48/0xf0
[13841.125819]  do_el0_svc+0x24/0x38
[13841.125841]  el0_svc+0x40/0xe8
[13841.125867]  el0t_64_sync_handler+0x100/0x130
[13841.125886]  el0t_64_sync+0x190/0x198
[13841.125909] Code: f9404261 52800140 29460fe2 f9412c21 (b9404421) 
[13841.125931] ---[ end trace 0000000000000000 ]---
```

经分析crtc->primary->state->rotation解引用到空指针，其中crtc->primary是一个plane，state没有初始化

```bash
// 内核代码
if (crtc->state &&
	    drm_rotation_90_or_270(crtc->primary->state->rotation))
		swap(hdisplay, vdisplay);
		
// 驱动代码中没有加入reset函数 修改后如下
static struct drm_plane_funcs ssd130x_plane_funcs = {                                                                               273     .update_plane = drm_atomic_helper_update_plane,
    .disable_plane = drm_atomic_helper_disable_plane,
    .destroy = drm_plane_cleanup,
    .atomic_destroy_state = drm_atomic_helper_plane_destroy_state,
    .atomic_duplicate_state = drm_atomic_helper_plane_duplicate_state,
    .reset = drm_atomic_helper_plane_reset,
};  
```

- （2）依旧是调用drm_mode_setcrtc后发生trap

```bash
[   91.792200] CPU: 2 PID: 740 Comm: test_drm Tainted: G         C O       6.6.51+rpt-rpi-v8 #1  Debian 1:6.6.51-1+rpt3
[   91.792232] Hardware name: Raspberry Pi 4 Model B Rev 1.5 (DT)
[   91.792249] pstate: 60000005 (nZCv daif -PAN -UAO -TCO -DIT -SSBS BTYPE=--)
[   91.792271] pc : 0x0
[   91.792293] lr : drm_mode_setcrtc+0x434/0x6c0 [drm]
[   91.792546] sp : ffffffc080f63b40
[   91.792559] x29: ffffffc080f63b40 x28: 000000000000001f x27: ffffff80455ad800
[   91.792595] x26: 0000000000000001 x25: ffffff8042b04648 x24: 0000000000000001
[   91.792628] x23: ffffff804251f900 x22: ffffff804102ae80 x21: ffffff80455ad800
[   91.792660] x20: ffffffc080f63d48 x19: ffffff8042b04010 x18: 0000000000000000
[   91.792692] x17: 0000000000000000 x16: ffffffeeb6b04440 x15: 0000000000000000
[   91.792724] x14: 0000000000000000 x13: 0048000000000048 x12: 0044004200480040
[   91.792756] x11: 0000000000000040 x10: ffffff8042b04208 x9 : ffffffee9c8daa94
[   91.792788] x8 : ffffffee9c910000 x7 : 0000000000000000 x6 : 0000000000000000
[   91.792819] x5 : 0000000000000000 x4 : ffffff8042b04bf0 x3 : 00000000ffffffff
[   91.792850] x2 : 0000000000000000 x1 : ffffffc080f63c08 x0 : ffffffc080f63bd8
[   91.792881] Call trace:
[   91.792895]  0x0
[   91.792911]  drm_ioctl_kernel+0xd8/0x190 [drm]
[   91.793134]  drm_ioctl+0x220/0x4c0 [drm]
[   91.793350]  __arm64_sys_ioctl+0xb4/0x100
[   91.793384]  invoke_syscall+0x50/0x128
[   91.793410]  el0_svc_common.constprop.0+0x48/0xf0
[   91.793435]  do_el0_svc+0x24/0x38
[   91.793459]  el0_svc+0x40/0xe8
[   91.793486]  el0t_64_sync_handler+0x100/0x130
[   91.793504]  el0t_64_sync+0x190/0x198
[   91.793535] Code: ???????? ???????? ???????? ???????? (????????) 
[   91.793557] ---[ end trace 0000000000000000 ]---
```

```c
// 内核代码调用crtc->funcs->set_config
// set_config没有设置
if (drm_drv_uses_atomic_modeset(dev))
		ret = crtc->funcs->set_config(&set, &ctx);

// 驱动代码添加set_config
static struct drm_crtc_funcs ssd130x_crtc_funcs = {
    .reset = drm_atomic_helper_crtc_reset,
    .destroy = drm_crtc_cleanup,
    .atomic_duplicate_state = drm_atomic_helper_crtc_duplicate_state,
    .atomic_destroy_state = drm_atomic_helper_crtc_destroy_state,
    .set_config = drm_atomic_helper_set_config,                                                           
};
```







https://crab2313.github.io/post/drm-legacy-kms/

https://encelo.github.io/trip_through_graphics_pipeline_2011.html

https://adrian.geek.nz/graphics_docs/DRM.html

http://phd.mupuf.org/files/kr2014.pdf

https://github.com/ascent12/drm_doc

https://www.cnblogs.com/zyly/p/17686403.html#_label0_0