# KVMTOOL

1.下载linux并编译linux内核源码

```shell
wget https://cdn.kernel.org/pub/linux/kernel/v4.x/linux-4.9.229.tar.xz
tar -xf linux-4.9.229.tar.xz
cd linux-4.9.229.tar.xz
make defconfig
# (optional) 自定义配置.config
make -j10

```

2.编译busybox

```shell
wget https://busybox.net/downloads/busybox-1.32.0.tar.bz2
tar -xf busybox-1.32.0.tar.bz2
cd busybox-1.32.0
make menu_config
```

<img width="1341" alt="image" src="https://user-images.githubusercontent.com/20353538/157057259-41dde3ca-e128-4ccf-9172-85d6b86a73b9.png">


3.准备一个initrd

4.启动虚拟机


## 参考文档
- https://github.com/kvmtool/kvmtool
- https://blog.csdn.net/caojinhuajy/article/details/119277087
- https://www.bilibili.com/read/cv7118525
