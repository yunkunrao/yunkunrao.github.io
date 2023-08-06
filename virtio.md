# virtio-net 架构图

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

