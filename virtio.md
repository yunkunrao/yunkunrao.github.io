# 虚拟化模型

当前主流的虚拟化技术的实现架构可分为三类：

1. Hypervisor模型

![image](https://github.com/yunkunrao/yunkunrao.github.io/assets/20353538/dc06f2ce-8c62-46bf-9e92-0a68d15b23b8)

这种方式是比较高效的，然而I/O设备种类繁多，管理所有设备就意味着大量的驱动开发工作。在实际的产品中，厂商会根据产品定位，有选择的支持一些I/O设备，而不是对所有的I/O设备都提供支持。

2. Host模型（宿主机）

![image](https://github.com/yunkunrao/yunkunrao.github.io/assets/20353538/bd9bf652-b21f-4fa0-abcf-3cf2c2c4a173)

Host模型最大的优点就是可以充分利用现有操作系统的设备驱动程序，VMM不需要为各种I/O设备重新实现驱动，可以专注于物理资源的虚拟化；缺点在于，由于VMM是借助host OS的服务来操作硬件，而不是直接操作硬件，因此受限于host OS服务的支持，可能导致硬件利用的不充分。

从架构上看，由Qumranet公司开发的KVM（Kernel-based Virtual Machine）就是属于host模型的，kernel-based，顾名思义就是基于操作系统内核。KVM于2007年被集成到Linux内核2.6.20版本，并于2008年被Red Hat收购。

随着越来越多的虚拟化功能被加入到Linux内核当中，Linux已经越来越像一个hypervisor了，从这个角度看，KVM也可以算是hypervisor模型了。

3. 混合模型

![image](https://github.com/yunkunrao/yunkunrao.github.io/assets/20353538/de521324-ee63-48d9-8d55-3af6267304ab)


混合模型可以说是结合了上述两种模型的优点，既不需要另外开发I/O设备驱动程序，又可以通过直接控制CPU和内存实现对这些物理资源的充分利用，以提高效率。

但它也是存在缺点的，当来自guest OS的I/O请求发送到VMM后，VMM需要将这些请求转发到service OS，这无疑增加了上下文的开销。混合模型的代表有Xen，Intel最近推出的Acrn，以及我国工程师写的minos。

**还有一种划分方法是将VMM分为基于bare-metal的type-1和基于OS的type-2，从这个角度划分的话，hypervisor模型和混合模型都是属于type-1的，host模型则是属于type-2的【1】。**

# virtio-net 架构

virtio-net 一套网络半虚拟化驱动 + 设备的方案。

![image](https://github.com/yunkunrao/yunkunrao.github.io/assets/20353538/a58f0b84-329e-474d-889c-a8152799425f)

说明：

- 控制通道（control path） 用来管理 前端（Front-end） 和 后端（Back-end） 数据通道和参数配置，如图中蓝实线所示
- 数据通道（data path） 传输数据，如图中 Host User Space 中红实线所示
- vring 虚拟机中与物理机共享的内存
- 通知通道（notifications） 传递 vmexit/vCPU irq 等，如图中红虚线所示
- virtio data path interface Qemu 和宿主机内核 tap 通信

## 网络通信流程
虚拟机中 virtio-net driver 通过虚拟 PCIE 总线 连接到 QEMU 模拟的 virtio-net device；驱动初始化后，两者建立 控制通道（control path），协商通信能力，虚拟机分配 vring 并与 QEMU 共享
网络发包：虚拟机更新 vring，通过 KVM通知 QEMU，QEMU 在将报文发送到宿主机内核的 TAP 设备
网络收包：QEMU 收到报文，填充 vring，通过 KVM 向虚拟机触发虚拟中断，虚拟机完成收包

## 问题
上下文切换多，报文和通知需要在虚拟机、KVM、QEMU 和 宿主机内核 四者之间多次切换
依赖 QEMU 传递数据，效率比较低

# vhost-net/virtio-net 架构

vhost 是 virtio 的一种后端实现方案（一般实现在用户态的QEMU中，但 vhost 实现在宿主机内核中 vhost-net.ko），vhost 通过将 virtio 的数据通道卸载到 内核态（如vhost-net） 或 用户态 来实现性能的提升

![image](https://github.com/yunkunrao/yunkunrao.github.io/assets/20353538/193b4558-160a-4e36-b0f4-d71b886a0118)

说明：

- virtio-net driver 作为前端，运行在虚拟机内核态
- vhost-net 作为后端，运行在宿主机内核态
- virtio 控制平面（control plane） 将 vring 协商好后，数据 脱离 QEMU，被 卸载 到vhost-net，仅负责控制层面的事情
- vhost 会启动内核线程，负责处理虚拟机的数据。其中，虚拟机的vring 直接映射到宿主机内核态，比较高效
- 与 virtio-net 相比，vhost-net 处理数据在内核态，在发送到 tap 的时候少了一次数据的拷贝

vhost 与 kvm 的事件通信通过 eventfd 机制来实现，主要包括两个方向的 event：

- 一个是 guest 到 vhost 方向的 kick event，通过 ioeventfd 实现
- 另一个是 vhost 到 guest 方向的 call event，通过 irqfd 实现

# 硬件实现的vDPA 架构

![image](https://github.com/yunkunrao/yunkunrao.github.io/assets/20353538/57950b61-1d1c-4754-a0fd-2c54eeb9b4fc)

满足对virtio data plane兼容性的设备可被称为"vDPA device"，但不同的设备厂商在control plane部分的实现往往存在差异。vDPA对此的解决办法是：vendor的control plane的驱动需要包含一个vDPA的add-on，然后通过vDPA driver，将其转化成virtio形式的control plane。


# Virtio：一种 I/O 半虚拟化框架

Virtio 是一套 I/O 半虚拟化的程序，是对半虚拟化 Hypervisor 中的一组通用 I/O 设备（如网络设备、块设备等）的抽象，为各种 I/O 设备 的虚拟化提供了通用的、标准化的前端接口，增加了各个虚拟化平台的代码复用。

## Virtio 架构

除了 Front-end drivers 和 Back-end Drivers 之外，Virtio 还定义了两层来支持虚拟机与 Hypervisor 进行通讯，Virtio 四层如下：

- Front-end层 guest 中各种驱动程序模块
- virtio层 是 虚拟队列（virtual queue） 接口，它是 前端驱动程序 和 后端驱动程序 通信的桥梁。驱动程序可以根据需要使用零个或多个队列。例如：
  virtio-net 网络驱动程序使用两个虚拟队列(一个用于接收，一个用于发送)
  virtio-blk 块驱动程序只使用一个队列
- virtio-ring层 是 virtio层 的具体实现，它实现了两个环形缓冲区，分别用于保存前端驱动程序和后端处理程序执行的信息。virtio-ring 实现了 virtio 的具体 通信机制 和 数据流程
  - 虚拟队列（virtual queue）是虚拟的，它实际上被实现为一个环（rings），用于遍历 guest-to-hypervisor 过渡。它也可以以其他方式实现，只要guest和  hypervisor以相同的方式实现即可
  - virtio 的核心机制就是通过共享内存在前端驱动与后端实现间进行数据传输，共享内存区域被称作 vring
- Back-end Hypervisor（实现在Qemu上）中的处理程序模块

![image](https://github.com/yunkunrao/yunkunrao.github.io/assets/20353538/2878094e-b1d3-417a-9a7c-5c17f58afa46)

上图，列出五个前端驱动程序：

- virtio-blk （如磁盘）
- virtio-net 网络设备
- virtio-pci PCI模拟
- virtio-balloon （用于动态管理客户内存使用）
- virtio-console 控制台

每个前端驱动程序在hypervisor中都有一个对应的后端驱动程序。

## 层次结构概念

![image](https://github.com/yunkunrao/yunkunrao.github.io/assets/20353538/c8062b20-265d-42cf-a6d2-f47a538ca801)

- virtio_driver表示客户机中的前端驱动。与驱动相匹配的设备被封装在virtio_device（在客户机中表示设备），其中有成员config指向virtio_config_ops结构（其中定义了配置virtio设备的操作）
- virtqueue 中有成员vdev指向virtio_device（也就是指向它所服务的某一设备virtio_device）
- 每个virtio_queue中有个类型为virtqueue_ops的对象，其中定义了与hypervisor交互的虚拟队列操作

## vring 组成

客户机驱动（前端）与hypervisor（后端）通过缓冲区进行通信。对于一次I/O，客户机提供一个或多个缓冲区表示请求。


