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

感谢容器技术，把开发人员从日常编译环境依赖的问题中解放出来 🤣

```shell
cat ./Dockerfile
FROM debian:10.8-slim

RUN apt-get update
RUN apt-get install -y \
        bc \
        bison \
        build-essential \
        cpio \
        flex \
        libelf-dev \
        libncurses-dev \
        libssl-dev \
        vim-tiny
        
docker build . -t teeny-linux-builder
docker run -ti -v `pwd`:/teeny teeny-linux-builder
cd busybox-1.32.0
make menuconfig
```

<img width="1341" alt="image" src="https://user-images.githubusercontent.com/20353538/157057259-41dde3ca-e128-4ccf-9172-85d6b86a73b9.png">

```shell
make -j10 && make install
```

3.准备一个initrd

```shell
cd busybox-1.32.0/_install
find . | cpio -o --format=newc > root_fs.cpio
```

4.启动虚拟机

```shell
kvmtool/lkvm run -k /root/yunkun/kvm/linux-4.9.229/arch/x86/boot/bzImage -i ./root_fs.cpio -m 2048
```

<img width="1210" alt="image" src="https://user-images.githubusercontent.com/20353538/157068147-6a06dc21-1063-47e8-812a-c99cae505d71.png">


## 参考文档
- https://github.com/kvmtool/kvmtool
- https://blog.csdn.net/caojinhuajy/article/details/119277087
- https://www.bilibili.com/read/cv7118525
- https://mgalgs.github.io/2021/03/23/how-to-build-a-custom-linux-kernel-for-qemu-using-docker.html
