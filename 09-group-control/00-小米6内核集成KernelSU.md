# 小米6内核集成KernelSU

* 原来一直使用[zlm324](https://github.com/zlm324)老哥开源的版本，https://github.com/zlm324/android_kernel_xiaomi_msm8998_ksu
* 老哥的版本很久没有更新了，于是我就自己动手编译了一下，并记录一下编译流程

## 1. 编译器

* https://github.com/LineageOS/android_prebuilts_gcc_linux-x86_aarch64_aarch64-linux-gnu-9.3

## 2. 代码集成

1. zlm老哥已经集成好了，有2个关键的commit
2. 可以跟着术哥的文档再过一遍：https://kernelsu.org/zh_CN/guide/how-to-integrate-for-non-gki.html

## 3. 版本选择

* KernelSU对non-GKI内核最多支持到0.9.5，但[经实践](#095版本kernelsu管理器显示不支持)，小米6只支持到0.9.2
* 一开始想用[官方版本](https://github.com/MiCode/Xiaomi_Kernel_OpenSource)，但编译出来，发现没有Wifi驱动、无法解决，所以只能用LineageOS
* 想选择最新版LineageOS-22，Android15，编译很顺利、一直到KernelSU都正常使用，但LSPosed模块、frida工作都不太正常
* 最后还是选择了LineageOS-20，Android13，直接clone zlm老哥的版本，然后更新KernelSU到0.9.2，就可以编译了

## 4. 编译流程

```shell
git clone --depth=1 https://github.com/zlm324/android_kernel_xiaomi_msm8998_ksu
cd android_kernel_xiaomi_msm8998_ksu/
# 更新KernelSU到0.9.2
curl -LSs "https://raw.githubusercontent.com/tiann/KernelSU/main/kernel/setup.sh" | bash -s v0.9.2
git submodule init
git submodule update
cd KernelSU
git checkout main
git checkout v0.9.2
cd ..
# 编译
export PATH=$PATH:/tools/android_prebuilts_gcc_linux-x86_aarch64_aarch64-linux-gnu-9.3/bin
export CROSS_COMPILE=aarch64-linux-
mkdir -p out
make ARCH=arm64 O=out clean
make ARCH=arm64 O=out mrproper
make ARCH=arm64 O=out sagit_defconfig
make ARCH=arm64 O=out ksu.config
./scripts/config --file out/.config --disable CONFIG_COMPAT_VDSO
make ARCH=arm64 O=out -j$(nproc --all)
# 成品Image可在releases页面中找到
```

## 备注

### 0.9.5版本KernelSU，管理器显示不支持

* 经搜索，https://github.com/tiann/KernelSU/issues/1871
* 那就降级到0.9.2

### 重要参考

* non-GKI kernel的各种参考：https://github.com/tiann/KernelSU/issues/943
* 找老版本的LineageOS：https://web.archive.org/web/20240309112048/https://download.lineageos.org/devices/sagit/builds

### 一般参考

* LineageOS安装方法：https://wiki.lineageos.org/devices/sagit/
* LineageOS提示网络受限：https://juejin.cn/post/7415197896512716854
