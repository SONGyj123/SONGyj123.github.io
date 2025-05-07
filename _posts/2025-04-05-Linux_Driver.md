# Linux Driver

## 0x00 ENV Setup

### 共享网络 （可以ssh不能上网的情况）

ifconfig命令查看wifi网卡和有线网卡，需要將流量从wifi网卡转发到有线网卡，如下所示wifi卡（wlo1），有线（eno1）

```bash
$ ifconfig
docker0: flags=4099<UP,BROADCAST,MULTICAST>  mtu 1500
        inet 172.17.0.1  netmask 255.255.0.0  broadcast 172.17.255.255
        ether 02:42:46:24:2c:35  txqueuelen 0  (Ethernet)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 155 overruns 0  carrier 0  collisions 0

eno1: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 10.42.0.1  netmask 255.255.255.0  broadcast 10.42.0.255
        inet6 fe80::d8ea:b4fd:d1b0:1445  prefixlen 64  scopeid 0x20<link>
        ether 04:0e:3c:04:37:f0  txqueuelen 1000  (Ethernet)
        RX packets 19935  bytes 12312967 (12.3 MB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 14446  bytes 2897490 (2.8 MB)
        TX errors 0  dropped 18 overruns 0  carrier 0  collisions 0

enx0826ae35f08d: flags=4099<UP,BROADCAST,MULTICAST>  mtu 1500
        ether 08:26:ae:35:f0:8d  txqueuelen 1000  (Ethernet)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 2 overruns 0  carrier 0  collisions 0

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        inet6 ::1  prefixlen 128  scopeid 0x10<host>
        loop  txqueuelen 1000  (Local Loopback)
        RX packets 1089779  bytes 4159784997 (4.1 GB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 1089779  bytes 4159784997 (4.1 GB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

wlo1: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.31.182  netmask 255.255.255.0  broadcast 192.168.31.255
        inet6 fe80::5ded:4ab:d586:b60c  prefixlen 64  scopeid 0x20<link>
        ether 50:e0:85:e8:58:0f  txqueuelen 1000  (Ethernet)
        RX packets 3582838  bytes 4974553253 (4.9 GB)
        RX errors 0  dropped 6  overruns 0  frame 0
        TX packets 788343  bytes 101990756 (101.9 MB)
        TX errors 0  dropped 34 overruns 0  carrier 0  collisions 0

```

输入以下命令进行转发配置：

```bash
$ sudo iptables -t nat -A POSTROUTING -o wlo1 -j MASQUERADE
$ sudo iptables -A FORWARD -i eno1 -o wlo1 -j ACCEPT
$ sudo iptables -A FORWARD -i wlo1 -o eno1 -m state --state RELATED,ESTABLISHED -j ACCEPT
$ sudo netfilter-persistent save
```

随后在单板端创建文件/etc/netplan/50-cloud-init.yaml，内容如下

```yaml
network:
    version: 2
    ethernets:
        eth0:
            addresses: [10.42.0.218/24]  # 与PC同子网
            routes:
              - to: 0.0.0.0/0
                via: 10.42.0.1  # PC的有线IP
            nameservers:
                addresses: [8.8.8.8, 1.1.1.1]  # 手动指定DNS
```

使能刚才的配置并确认已经转发到对应端口并尝试ping外网

```bash
$ sudo netplan apply
$ ip route show | grep default
default via 10.42.0.1 dev enP4p65s0 proto dhcp metric 100
$ ping baidu.com
ING baidu.com (110.242.68.66) 56(84) bytes of data.
64 bytes from baidu.com (110.242.68.66): icmp_seq=1 ttl=49 time=84.8 ms
64 bytes from baidu.com (110.242.68.66): icmp_seq=3 ttl=49 time=101 ms
```

[rockpi 5B gpio pinout]: https://wiki.radxa.com/Rock5/hardware/5b/gpio

### tftp启动内核

```shell
setenv ipaddr 10.42.0.218
setenv netmask 255.255.255.0
setenv gatewayip 10.42.0.1
setenv serverip 10.42.0.1
setenv bootargs root=/dev/nfs rw nfsroot=${serverip}:/home/songyj/embedded/nfs,v3,tcp ip=${ipaddr}::${serverip}:255.255.255.0::eth0:off console=ttyS0,1500000n8
pci enum
tftpboot ${kernel_addr_r} Image
tftpboot ${fdt_addr_r} rk3588-rock-5b.dtb
booti ${kernel_addr_r} - ${fdt_addr_r}
```

[collabora uboot]: https://gitlab.collabora.com/hardware-enablement/rockchip-3588/notes-for-rockchip-3588/-/blob/main/upstream_uboot.md

### 如何编译驱动？

在PC上编译驱动需要内核源码以及交叉编译工具链，驱动文件可以在内核外随便找地方存放，编译之前需要先设置环境变量：

```bash
export ARCH=arm64
export CROSS_COMPILE=aarch64-linux-gnu-
```

在Makefile中指定内核路径。make -C表示到KDIR这个路径执行make，随后將生成的.ko复制到板上。

```makefile
KDIR ?= /home/songyj/embedded/rk5b/kernel
PWD := $(shell pwd)

obj-m += gpio.o

all:
        make -C ${KDIR} M=$(PWD) modules

clean:
        make -C ${KDIR} M=$(PWD) clean
```



## 0x01 Device Tree

### DT basics

![](/home/songyj/note/pics/dts_description.png)

冒号之前的是Lable，相当于节点的别名，如果后续需要修改节点的的property可以通过引用别名来追加修改或者新建property。

![](/home/songyj/note/pics/dts_label_add_property.png)

看一个例子，下面是截取rock5B的RGB LED的设备树节点。结合原理图可以确认user-led2是控制蓝色LED的引脚，RK_PB7和GPIO_ACTIVE_HIGH代表引脚编号以及电平设置，这两个宏定义在dt-binding相关的文档。这些格式都是内核文档中已经规定好的，在**Documentation/devicetree/bindings**中搜索具体的规则。

```c
// arch/arm64/boot/dts/rockchip/rk3588-rock-5b.dts
/{
	...
	
	gpio-leds {
        compatible = "gpio-leds";
        pinctrl-names = "default";

        user-led2 {
            gpios = <&gpio0 RK_PB7 GPIO_ACTIVE_HIGH>;
            linux,default-trigger = "heartbeat";
            default-state = "on";
    	};
	};
	
	...
}

// include/dt-bindings/pinctrl/rockchip.h
#define RK_PB7		15

// include/dt-bindings/gpio/gpio.h
#define GPIO_ACTIVE_HIGH 0
```

![](/home/songyj/note/pics/rk5b_rgb_led_sch.png)

### cells

DeveiceTree中有个比较重要的概念**cells**，**cells**表示property相关的32bits的数字。

```c
reg = <0x50027000>, <0x4>, <0x500273f0>, <0x10>;	// 4 entriy 4 cell
reg = <0x50027000 0x4 0x500273f0>, <0x10>;			// 2 entriy 4 cell
reg = <0x50027000>, <0x4 0x500273f0 0x10>;			// 2 entriy 4 cell
reg = <0x50027000 0x4 0x500273f0 0x10>;				// 1 entriy 4 cell
```

cells如何规定怎么填写property值，以下面设备树为例：在spi3节点中使用到之前定义的三个外设节点intc，rcc以及dmamux1，他们的cells与后面spi3使用时正好对应（注意类似*&rcc*和*&dmamux1*不是cell，因为他们不是和property相关的值，不计入cells数量）

```c
// 定义三个外设
intc: interrupt-controller@a0021000 {
    compatible = "arm,cortex-a7-gic";
    #interrupt-cells = <3>;
    interrupt-controller;
    reg = <0xa0021000 0x1000>, <0xa0022000 0x2000>;
};
rcc: rcc@50000000 {
    compatible = "st,stm32mp1-rcc", "syscon";
    reg = <0x50000000 0x1000>;
    #clock-cells = <1>;
    #reset-cells = <1>;
};
dmamux1: dma-router@48002000 {
    compatible = "st,stm32h7-dmamux";
    reg = <0x48002000 0x1c>;
    #dma-cells = <3>;
    clocks = <&rcc DMAMUX>;
    resets = <&rcc DMAMUX_R>;
};

// spi3需要配置这三个外设
spi3: spi@4000c000 {
    interrupts = <GIC_SPI 51 IRQ_TYPE_LEVEL_HIGH>;
    clocks = <&rcc SPI3_K>;
    resets = <&rcc SPI3_R>;
    dmas = <&dmamux1 61 0x400 0x05>, <&dmamux1 62 0x400 0x05>;
};
```

### DTB使用

- 设备树编译命令

```bash
dtc -@ -I dts -O dtb -o ./gpio.dtbo ./gpio.dts
```

- 使能自己的dtbo

**方法1：/boot/extlinux/extlinux.conf**

这个配置文件是uboot的启动参数，l0是默认启动参数，l0r是recovery启动参数，添加*fdtoverlays /lib/firmware/5.10.0-1012-rockchip/device-tree/rockchip/gpio.dtbo*指定要加载的dtbo文件。设置好路径之后重启设备，在*/sys/firmware/devicetree/base*中检查有没有对应的节点。

```bash
## /boot/extlinux/extlinux.conf
##
## IMPORTANT WARNING
##
## The configuration of this file is generated automatically.
## Do not edit this file manually, use: u-boot-update

default l0
menu title U-Boot menu
prompt 1
timeout 20

label l0
        menu label Ubuntu 22.04.5 LTS 5.10.0-1012-rockchip
        linux /boot/vmlinuz-5.10.0-1012-rockchip
        initrd /boot/initrd.img-5.10.0-1012-rockchip
        fdtdir /lib/firmware/5.10.0-1012-rockchip/device-tree/
        fdtoverlays /lib/firmware/5.10.0-1012-rockchip/device-tree/rockchip/overlay/gpio.dtbo

        append root=UUID=cbf6ee10-c4b9-49e9-b246-f17a818913e1 rootwait rw console=ttyS2,1500000 console=tty1 cgroup_enable=cpuset cgroup_memory=1 cgroup_enable=memory quiet splash plymouth.ignore-serial-consoles

label l0r
        menu label Ubuntu 22.04.5 LTS 5.10.0-1012-rockchip (rescue target)
        linux /boot/vmlinuz-5.10.0-1012-rockchip
        initrd /boot/initrd.img-5.10.0-1012-rockchip
        fdtdir /lib/firmware/5.10.0-1012-rockchip/device-tree/
        append root=UUID=cbf6ee10-c4b9-49e9-b246-f17a818913e1 rootwait rw console=ttyS2,1500000 console=tty1 cgroup_enable=cpuset cgroup_memory=1 cgroup_enable=memory splash plymouth.ignore-serial-consoles single
```

**dtbo有时候很难给原设备树节点覆盖上**

## 0x02 Linux设备驱动模型

驱动通过module_init()和module_exit()在device和device driver匹配时被调用，驱动需要申请device和

```c
module_init(drv_init_fn);
module_exit(drv_exit_fn);
```

但是很多时候我们看到的是类似下面的方式，难免感到疑惑

```c
module_i2c_driver(i2c_driver);
module_pci_driver(pci_driver);
```

翻看这几个宏定义可知，依然是init和exit的方式，只是helper宏定义帮忙向bus注册了driver

```c
// module_i2c_driver
#define module_i2c_driver(__i2c_driver) \
	module_driver(__i2c_driver, i2c_add_driver, \
			i2c_del_driver)

// module_pci_driver
#define module_pci_driver(__pci_driver) \
	module_driver(__pci_driver, pci_register_driver, pci_unregister_driver)

// module_driver宏定义
#define module_driver(__driver, __register, __unregister, ...) \
static int __init __driver##_init(void) \
{ \
	return __register(&(__driver) , ##__VA_ARGS__); \
} \
module_init(__driver##_init); \
static void __exit __driver##_exit(void) \
{ \
	__unregister(&(__driver) , ##__VA_ARGS__); \
} \
module_exit(__driver##_exit);
```

