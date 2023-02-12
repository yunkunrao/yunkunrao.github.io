# BPF 珠玑

- https://www.brendangregg.com/perf.html
- https://www.ebpf.top/post/bpf_learn_path/
- http://arthurchiao.art/blog/ebpf-and-k8s-zh/
- http://arthurchiao.art/blog/understanding-ebpf-datapath-in-cilium-zh/
- http://arthurchiao.art/blog/advanced-bpf-kernel-features-for-container-age-zh/
- http://arthurchiao.art/blog/cilium-life-of-a-packet-pod-to-service-zh/

# KVMTOOL 🐂刀小试

## 1. 下载linux并编译linux内核源码

```shell
wget https://cdn.kernel.org/pub/linux/kernel/v4.x/linux-4.9.229.tar.xz
tar -xf linux-4.9.229.tar.xz
cd linux-4.9.229.tar.xz
make defconfig
# (optional) 自定义配置.config
make -j10

```

## 2. 编译busybox

```shell
wget https://busybox.net/downloads/busybox-1.32.0.tar.bz2
tar -xf busybox-1.32.0.tar.bz2
cd busybox-1.32.0
make menuconfig
```

感谢容器技术，把开发人员从日常编译环境依赖的问题中解放出来 🤣

```shell
cat << EOF > ./Dockerfile
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
EOF
        
docker build . -t teeny-linux-builder
docker run -ti -v `pwd`:/teeny teeny-linux-builder
cd busybox-1.32.0
make menuconfig
```

<img width="1341" alt="image" src="https://user-images.githubusercontent.com/20353538/157057259-41dde3ca-e128-4ccf-9172-85d6b86a73b9.png">

```shell
make -j10 && make install
```

## 3. 准备一个rootfs

```shell
cd busybox-1.32.0/_install
find . | cpio -o --format=newc > root_fs.cpio
```

## 4. 启动虚拟机

```shell
kvmtool/lkvm run -k /root/yunkun/kvm/linux-4.9.229/arch/x86/boot/bzImage -i ./root_fs.cpio -m 2048
```

<img width="1210" alt="image" src="https://user-images.githubusercontent.com/20353538/157068147-6a06dc21-1063-47e8-812a-c99cae505d71.png">


## 参考文档
- https://github.com/kvmtool/kvmtool
- https://blog.csdn.net/caojinhuajy/article/details/119277087
- https://www.bilibili.com/read/cv7118525
- https://mgalgs.github.io/2021/03/23/how-to-build-a-custom-linux-kernel-for-qemu-using-docker.html

# Debugging the Linux Kernel with Qemu and GDB 正题

## 1. Build your own Custom Kernel 【需要开启DBG】

```shell
cd /root
git clone https://mirrors.tuna.tsinghua.edu.cn/git/linux.git
cd linux
git checkout v4.18
cp /boot/config-`uname -r` /root/linux/.config

# 确保 CONFIG_DEBUG_INFO=y 
make olddefconfig # or make menuconfig
make -j`nproc`
make -j`nproc` modules_install 
```

## 2. Building qemu from source 【需要开启DBG】

```shell
cd /root/
git clone git://git.qemu-project.org/qemu.git
cd qemu
git checkout v4.1.0
mkdir build && cd build
../configure --target-list=x86_64-softmmu --enable-debug
make -j`nproc`
make install
```

## 3. Make ROOTFS

### 3.1 Download busybox

```shell
cd /root
wget https://busybox.net/downloads/busybox-1.31.1.tar.bz2
tar -jxvf busybox-1.31.1.tar.bz2
```

```shell
Settings -> Build Options -> Build static binary (no shared libs)
```

## 3.2 Creating rootfs with busybox

```shell
dd if=/dev/zero of=/root/busybox_rootfs.img bs=1M count=10
mkfs.ext3 /root/busybox_rootfs.img

mkdir rootfs_mount
sudo mount -t ext3 -o loop /root/busybox_rootfs.img /root/busybox-1.31.1/rootfs_mount
```

## 3.3 Compile busybox

```shell
cd /root/busybox-1.31.1
cat << EOF > ./Dockerfile
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
EOF

docker build . -t busybox-builder
docker run -ti -v `pwd`:/busybox busybox-builder
[container]: cd busybox

# After unpacking and entering the busybox folder, first configure it using make gconfig or make menuconfig, which requires the following options to be enabled.
[container] make menuconfig
[container] make install CONFIG_PREFIX=/busybox/rootfs_mount/

# Finally, to configure busybox init and uninstall rootfs.
[container] mkdir /busybox/rootfs_mount/proc
[container] mkdir /busybox/rootfs_mount/dev
[container] mkdir /busybox/rootfs_mount/etc
[container] cp /busybox/examples/bootfloppy/* /busybox/rootfs_mount/etc/
umount /root/busybox-1.31.1/rootfs_mount
```

# 4. Booting the kernel with Qemu

## 4.1 正常启动Qemu

```shell
qemu-system-x86_64 \
  -kernel /root/linux/arch/x86_64/boot/bzImage \
  -hda /root/busybox_rootfs.img \
  -serial stdio \
  -append "root=/dev/sda console=ttyS0 nokaslr"
```
<img width="1217" alt="image" src="https://user-images.githubusercontent.com/20353538/213609094-14de9e77-8232-43a9-a05b-5d01fa6e304c.png">

## 4.2 指定Qemu在启动时暂停并启动gdb server，等待gdb的连入（端口默认为1234）

```shell
qemu-system-x86_64 \
  -kernel /root/linux/arch/x86_64/boot/bzImage \
  -hda /root/busybox_rootfs.img \
  -serial stdio \
  -append "root=/dev/sda console=ttyS0 nokaslr" \
  -s -S
```
<img width="1241" alt="image" src="https://user-images.githubusercontent.com/20353538/213609185-7910eb91-8c44-4d3a-a4b0-fafa170729d8.png">

## 4.3 Debugging the kernel with GDB

```shell
gdb /root/linux/vmlinux # 指定调试文件为包含调试信息的内核文件

(gdb) target remote:1234
(gdb) b start_kernel
(gdb) c
```

<img width="733" alt="image" src="https://user-images.githubusercontent.com/20353538/213609447-7898c81b-9a75-4969-baf3-7c104af9c59f.png">

## 4.4 Debugging QEMU with gdb

https://www.cnblogs.com/root-wang/p/8005212.html

## 参考文档
- https://pnx9.github.io/thehive/Debugging-Linux-Kernel.html
- https://www.sobyte.net/post/2022-02/debug-linux-kernel-with-qemu-and-gdb/#download-and-compile-busybox
- https://www.cnblogs.com/root-wang/p/8005212.html
