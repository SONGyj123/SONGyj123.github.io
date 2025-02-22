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

使用libdrm提供的接口可以完成喜爱那是设置，下面先看一个simple_drm.c程序，程序使用drm接口配置屏幕显示全白：

```c
#include <stddef.h>
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

***为什么將节点换成/dev/dri/renderD128之后segfault？***



#### legacy接口

- **drmModeSetCrtc**
- **drmModePageFlip** （只会在vsync的时候进行切换framebuffer，避免画面撕裂）
- **drmModeSetPlane**（只显示framebuffer的部分，之前都是全部显示）使用前需要先调用drmModeSetCrtc打通链路

![img](https://i-blog.csdnimg.cn/blog_migrate/c1f6c3fdeef9d3571a45e46580c7379b.png)

#### atomic接口

atomic关键词如何理解？

所有的property配置完成后进行commit，commit如果失败，所有的配置恢复到commit之前

atomic接口引入property，这些property的引入简化驱动层IOCTL函数

![img](https://i-blog.csdnimg.cn/blog_migrate/2df1c788392fd8240a9637afad6f1d6b.png)

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



KMS

DRM driver必须调用[`drm_mode_config_init`](https://www.kernel.org/doc/html/v4.11/gpu/drm-kms.html#c.drm_mode_config_init)

对**struct drm_driver**中的***driver_feature***进行置位，告诉DRM驱动框架当前驱动的能力，常见的标志位如下：

- **DRIVER_MODESET** 支持modesetting操作
- **DRIVER_GEM** 支持内存分配
- **DRIVER_ATOMIC** 支持atomic操作
- **DRIVER_RENDER** 支持渲染

调用fb_create的ioctl是**DRM_IOCTL_MODE_ADDFB2**，fb_create將gem obj handle和fb绑定，没有绑定handle的fb没有任何实际的显示内容



#### 综合实现分析（exynos为例）

完整的KMS驱动中间含有很多sub drivers不是以前那种简单的一个.c的char driver所能相比的，这种驱动被叫做aggregate driver。不过依然可以找到一条主线来分析它：

```
├── exynos5433_drm_decon.c
├── exynos5433_drm_decon.o
├── exynos7_drm_decon.c
├── exynos7_drm_decon.o
├── exynos_dp.c
├── exynos_drm_crtc.c
├── exynos_drm_crtc.h
├── exynos_drm_crtc.o
├── exynos_drm_dma.c
├── exynos_drm_dma.o
├── exynos_drm_dpi.c
├── exynos_drm_drv.c
├── exynos_drm_drv.h
├── exynos_drm_drv.o
├── exynos_drm_dsi.c
├── exynos_drm_dsi.o
├── exynos_drm_fb.c
├── exynos_drm_fbdev.c
├── exynos_drm_fbdev.h
├── exynos_drm_fbdev.o
├── exynos_drm_fb.h
├── exynos_drm_fb.o
├── exynos_drm_fimc.c
├── exynos_drm_fimd.c
├── exynos_drm_g2d.c
├── exynos_drm_g2d.h
├── exynos_drm_gem.c
├── exynos_drm_gem.h
├── exynos_drm_gem.o
├── exynos_drm_gsc.c
├── exynos_drm_ipp.c
├── exynos_drm_ipp.h
├── exynosdrm.ko
├── exynos_drm_mic.c
├── exynos_drm_mic.o
├── exynosdrm.mod
├── exynosdrm.mod.c
├── exynosdrm.mod.o
├── exynosdrm.o
├── exynos_drm_plane.c
├── exynos_drm_plane.h
├── exynos_drm_plane.o
├── exynos_drm_rotator.c
├── exynos_drm_scaler.c
├── exynos_drm_vidi.c
├── exynos_drm_vidi.h
├── exynos_hdmi.c
├── exynos_hdmi.o
├── exynos_mixer.c
├── Kconfig
├── Makefile
├── modules.order
├── regs-decon5433.h
├── regs-decon7.h
├── regs-fimc.h
├── regs-gsc.h
├── regs-hdmi.h
├── regs-mixer.h
├── regs-rotator.h
├── regs-scaler.h
└── regs-vp.h
```

从exynos_drm_drv.c开始看起

exynos_drm_init()作为入口函数被module_init宏设置到**内核入口**

exynos_drm_register_devices()

exynos_drm_register_drivers()

#### DRM解析分发ioctl

用户程序接口的名字和ioctl的名字不一定一致

atomic ioctl由一个ioctl统一分发



GEM是内存分配机制，dma-buf是内存共享机制，GEM和dma-buf结合使用。

#### dma-buf

dma-buf在内核空间和用户空间中的表显形式不一样，在用户空间中是一个**fd**，在内核中是**struct dma_buf**

> struct dma_buf - shared buffer object
>
> This represents a shared buffer, created by calling dma_buf_export(). The userspace representation is a normal file descriptor, which can be created by calling dma_buf_fd().
>
> Shared dma buffers are reference counted using dma_buf_put() and get_dma_buf().
>
> Device DMA access is handled by the separate &struct dma_buf_attachment.

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

https://crab2313.github.io/post/drm-legacy-kms/

https://encelo.github.io/trip_through_graphics_pipeline_2011.html

https://adrian.geek.nz/graphics_docs/DRM.html

http://phd.mupuf.org/files/kr2014.pdf