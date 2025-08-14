Armbian快速构建uboot和kernel

```bash
sudo dd if=./idbloader.img of=/dev/sdb seek=64 conv=notrunc
sudo dd if=./u-boot.itb of=/dev/sdb seek=16384 conv=notrunc
sync
```

打patch

```bash
# uboot
./compile.sh uboot-patch BOARD=rock-3c BRANCH=current

# kernel
./compile.sh kernel-patch BOARD=rock-3c BRANCH=current BUILD_DESKTOP=no BUILD_MINIMAL=no KERNEL_CONFIGURE=no RELEASE=noble
```



```
./compile.sh build BOARD=rock-3c BRANCH=current BUILD_DESKTOP=no BUILD_MINIMAL=no KERNEL_CONFIGURE=no RELEASE=noble
```



vmlinux	ELF	否	通用	否（太大）	编译的原始产物
Image	Binary	否	ARM64 等	是（部分支持）	由 vmlinux 转换
Image.gz	Binary + Gzip	是	ARM64 等	是（需要支持 gzip）	gzip Image
zImage	自解压格式	是	ARM	是	包含解压启动代码的镜像
uImage	U-Boot格式	任意	U-Boot 平台	是	用 mkimage 包装