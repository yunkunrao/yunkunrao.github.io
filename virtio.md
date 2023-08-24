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

# Virtqueue 介绍

Virtqueue 是对 desc table、vring 等数据结构的抽象，被上层 virtio 设备使用，通过内存 buffer 实现 host 和 guest 之间的数据交换能力。这些 virtio 设备使用 0 到多个 virtqueues，比如本文要讲的 virtio-net 使用了两个 virtqueues（如右半边所示），分别是 tx queue 和 rx queue，前者是 guest 向 host 发送数据，后者则是 host 向 guest 发送数据。需要注意的是一个 virtqueue 就具备双向传输数据的能力，virtio-net 使用两个 virtqueues 实现全双工通信的目的是提升效率、避免读写冲突。

![image](https://github.com/yunkunrao/yunkunrao.github.io/assets/20353538/b99b6412-f89d-43d8-a498-4405b690eb1a)

## Virtqueue

一个 virtqueue 是一块由 guest 申请的共享内存区域，guest 和 host 可以在这块内存中读或者写，然后被对端消费实现数据传递。一个 virtqueue 可以被认为由 desc table、avail ring、used ring 和 buffer 四个部分组成，其中 avail ring 和 used ring 是一个循环队列（一种著名的数据结构，如果不懂的话建议回炉重造 lol），也被称为 vring。

- Desc table：描述了已经被使用的 buffer 的信息。
- Avail ring：Driver 向 device 传输的数据。
- Used ring：Device 向 driver 传输的数据。

不难看出，一个 virtqueue 已经具备了全双工通信，所以 virtio-net 使用两个 virtqueue 真的是为了性能考虑的。这些数据结构都是由 driver 申请的（处于 guest 地址空间），因此 device 必须翻译到 host 地址空间才能使用，还是以网络虚拟化为例，翻译的过程包括：

- Emulated device：类似于 virtio-net，guest 地址空间就是 hypervisor 进程的地址空间，所以用不着翻译。
- 其他 emulated deivce：类似于 vhost-net 和 vhost-user-net，使用的是 POSIX 共享内存的方式，即 hypervisor 进程与内核（vhost-net）或者其他用户态应用（vhost-user-net）的地址空间不是同一个。
- IOMMU：暂略。

## Desc table

Desc table 是一个 virtq_desc 组成的数组，每一个入口项（entry）描述了一段内存区域（起始地址 addr 和长度 len），其数据结构如下所示。

```c
struct virtq_desc { 
        le64 addr;
        le32 len;
        le16 flags;
        le16 next;
};
```

Driver 需要获得一个 buffer 的途径有二：（1）从 buffer 中分配一个区域，并将其对应的入口项添加到 desc table。（2）从 used ring 中获取一个已经被 device 消费过的 buffer（在流程分析里再次提到，可以到那里再品品这句话）。

需要注意的是，只有 driver 有更新 desc table 的权限（绿色 read-wirte 线），device 只有读取 desc table 的权限（红色 read-only 线），但是 driver（绿色 read-only/write-only 线）和 device（红色 read-only/write-only 线）可以根据 VRING_DESC_F_WRITE 标志位从对应的 buffer 读取数据或者写入数据。Desc table 可以被理解为这些 buffer 的元数据（metadata）。

![image](https://github.com/yunkunrao/yunkunrao.github.io/assets/20353538/e93edbbe-ccef-4580-8c36-fb2e241cdf33)

![image](https://github.com/yunkunrao/yunkunrao.github.io/assets/20353538/858db9f2-4655-4e49-8aab-a69e5ddf44f7)

## Avail ring

Avail ring 用于传递等待 device 处理的描述符索引，该数据结构主要是由 driver 维护（对 device 只读）。Avail ring 的数据结构是 virtq_avail，如下所示。

```c
struct virtq_avail {
        le16 flags;
        le16 idx;
        le16 ring[ /* Queue Size */ ];
};
```

其中，
- Flags 是与 avail ring 相关的标志位，比如 VIRTQ_AVAIL_F_NO_INTERRUPT 用于标识在有新的 device 待处理的描述符索引被添加到 avail ring 之后是否立即通知 device（如果数据频繁到达，立即通知可能导致性能低下）。
- Idx 表示下一个入口项的索引（类似于环形队列的 tail 指针）。
- Ring 是存放描述符索引的数组，长度是 queue size。

Avail ring 与 desc table 关联如下图所示（图片来源 [1]），这个表示 device 需要在起始位置 0x8000 长度 2000 的 buffer 中写入一些数据回传给 driver。

![image](https://github.com/yunkunrao/yunkunrao.github.io/assets/20353538/37e27e23-8a7e-444d-9dcb-0dc5880665db)

## Used ring

Used ring 与 avail ring 在结构上基本一致，由 device 维护（对 driver 只读），其数据结构如下所示。

```c
struct virtq_used {
        le16 flags;
        le16 idx;
        struct virtq_used_elem ring[ /* Queue Size */];
};

struct virtq_used_elem {
        le32 id;
        le32 len;
};
```

与 avail ring 的区别是 ring 字段的类型变为了 virtq_used_elem，该结构体包含了一个描述符索引和长度，表示 device 在向 buffer 写入的数据的长度。为什么会有这个区别呢？我猜是因为 device 没有 desc table 的更新权限，它无法修改入口项的长度字段，因此不得不通过 used ring 来告知 driver 本次数据的长度是多少。

至于其他字段与 avail ring 保持一致，这里就不再赘述了。

![image](https://github.com/yunkunrao/yunkunrao.github.io/assets/20353538/3bd1a9b2-5ac7-4c46-af28-6e56adbee56f)

这个图是上文介绍所有内容的大杂烩，隐含的信息非常多。首先，desc table 的第 0 项和第 1 项组成了一个长度为 0x4000 的 chained buffer，它的权限是 device write-only。Driver 将 descriptors 放到了 avail ring 中，然后 device 消费并在该 buffer 中存放了长度为 0x3000 的数据，等待 driver 消费（或者 driver 已经消费过了）。

## 流程分析

### Guest -> host

![image](https://github.com/yunkunrao/yunkunrao.github.io/assets/20353538/affc2114-ce7b-435f-b63d-123e0c81e777)

步骤：

- ①A 表示分配一个 buffer 并添加到 virtqueue 中，①B 表示从 used ring 中获取一个buffer，这两种中选择一种方式；
- ② 表示将数据拷贝到 buffer中，用于传送；
- ③ 表示更新 avail ring 中的描述符索引值，注意，驱动中需要执行 memory barrier 操作，确保 device 能看到正确的值；
- ④ 与 ⑤ 表示 driver 通知 device 来取数据；
- ⑥ 表示 device 从 avail ring 中获取到描述符索引值；
- ⑦ 表示将描述符索引对应的地址中的数据取出来；
- ⑧ 表示 device 更新 used 队列中的描述符索引；
- ⑨ 与 ⑩ 表示 device 通知 driver 数据已经取完了。

### Host -> Guest

![image](https://github.com/yunkunrao/yunkunrao.github.io/assets/20353538/2a0cd14a-d45a-4f85-8cc3-468a12455131)

步骤：

- ① 表示 device 从 avail ring 中获取可用描述符索引值；
- ② 表示将数据拷贝至描述符索引对应的地址上；
- ③ 表示更新 used ring 中的描述符索引值；
- ④ 与 ⑤ 表示 device 通知 driver 来取数据；
- ⑥ 表示 driver 从 used ring 中获取已用描述符索引值；
- ⑦ 表示将描述符索引对应地址中的数据取出来；
- ⑧ 表示将 avail ring 中的描述符索引值进行更新；
- ⑨ 与 ⑩ 表示 driver 通知 device 有新的可用描述符；

# Linux虚拟化KVM-Qemu分析之ioeventfd与irqfd

## 1. 概述

![image](https://github.com/yunkunrao/yunkunrao.github.io/assets/20353538/18b15ae8-4203-4d46-8fff-2a2d91814acd)

Guest与KVM及Qemu之间的通知机制，如下图：

![image](https://github.com/yunkunrao/yunkunrao.github.io/assets/20353538/45f778d7-68b5-4e0a-95df-aaab951abb6f)

- irqfd：提供一种机制，可以通过文件描述fd来向Guest注入中断，路径为紫色线条所示；
- ioeventfd：提供一种机制，可以通过文件描述符fd来接收Guest的信号，路径为红色线条所示；
- eventfd和irqfd这两种机制，都是基于eventfd来实现的；

本文会先介绍eventfd机制，然后再分别从Qemu/KVM来介绍ioeventfd和irqfd，开始吧。

## 2. eventfd

eventfd的机制比较简单，大体框架如下图：

![image](https://github.com/yunkunrao/yunkunrao.github.io/assets/20353538/77466a81-e6b5-45b1-84f4-01695b603d6c)

- 内核中创建了一个struct eventfd_ctx结构体，该结构体中维护一个count计数，以及一个等待队列头；
- 线程/进程在读eventfd时，如果count值等于0时，将当前任务添加到等待队列中，并进行调度，让出CPU。读过程count值会进行减操作；
- 线程/进程在写eventfd时，如果count值超过最大值时，会将当前任务添加到等待队列中（特殊情况），写过程count值会进行加操作，并唤醒在等待队列上的任务；
- 内核的其他模块也可以通过eventfd_signal接口，将count值加操作，并唤醒在等待队列上的任务；

代码实现如下图：

![image](https://github.com/yunkunrao/yunkunrao.github.io/assets/20353538/f19993bf-7cee-435b-9b09-e633a43468b0)

- eventfd_read：如果count值为0，将自身添加到等待队列中，设置任务的状态后调用schedule让出CPU，等待被唤醒。读操作中会对count值进行减操作，最后再判断等待队列中是否有任务，有则进行唤醒；

- eventfd_write：判断count值在增加ucnt后是否会越界，越界则将自身添加到等待队列中，设置任务的状态后调用schedule让出CPU，等待被唤醒。写操作会对count值进行加操作，最后再判断等待队列中是否有任务，有则进行唤醒；

- 此外，还有eventfd_signal接口，比较简单，完成的工作就是对count值进行加操作，并唤醒等待任务；

## 3. ioeventfd

### 3.1 Qemu侧

以PCI设备为例：

![image](https://github.com/yunkunrao/yunkunrao.github.io/assets/20353538/bc3b70b3-86d1-437a-8ecb-711e00514abd)

- Qemu中模拟PCI设备时，在初始化过程中会调用memory_region_init_io来进行IO内存空间初始化，这个过程中会绑定内存区域的回调函数集，当Guest OS访问这个IO区域时，可能触发这些回调函数的调用；
- Guest OS中的Virtio驱动配置完成后会将状态位置上VIRTIO_CONFIG_S_DRIVER_OK，此时Qemu中的操作函数调用virtio_pci_common_write，并按图中的调用流逐级往下；
- event_notifier_init：完成eventfd的创建工作，它实际上就是调用系统调用eventfd()的接口，得到对应的文件描述符；
- memory_region_add_eventfd：为内存区域添加eventfd，将eventfd和对应的内存区域关联起来；

看一下memory_region_add_eventfd的流程：

![image](https://github.com/yunkunrao/yunkunrao.github.io/assets/20353538/eeccd81b-1ea7-450d-92b8-8ee16240effe)

- 内存区域MemoryRegion中的ioeventfds成员按照地址从小到大排序，memory_region_add_eventfd函数会选择合适的位置将ioeventfds插入，并提交更新；
- 提交更新过程中最终触发回调函数kvm_mem_ioeventfd_add的执行，这个函数指针的初始化是在Qemu进行kvm_init时，针对不同类型的内存区域注册了对应的memory_listener用于监听变化；
- kvm_vm_ioctl：向KVM注册ioeventfd；

Qemu中完成了初始化后，任务就转移到了KVM中。

### 3.2 KVM侧

KVM中的ioeventfd注册如下：

![image](https://github.com/yunkunrao/yunkunrao.github.io/assets/20353538/45fb819a-cb00-4479-9366-d15270b399bf)

- KVM中注册ioeventfd的核心函数为kvm_assign_ioeventfd_idx，该函数中主要工作包括：
1）根据用户空间传递过来的fd获取到内核中对应的struct eventfd_ctx结构体上下文；
2）使用ioeventfd_ops操作函数集来初始化IO设备操作；
3）向KVM注册IO总线，比如KVM_MMIO_BUS，注册了一段IO地址区域，当操作这段区域的时候出发对应的操作函数回调；

- 当Guest OS中进行IO操作时，触发VM异常退出，KVM进行捕获处理，最终调用注册的ioevnetfd_write，在该函数中调用eventfd_signal唤醒阻塞在eventfd上的任务，Qemu和KVM完成了闭环；

总体效果如下图：

![image](https://github.com/yunkunrao/yunkunrao.github.io/assets/20353538/6d6238b7-2f48-42ad-bfa2-0c05a5f7fc19)

### 4. irqfd

Qemu和KVM中的流程如下图：

![image](https://github.com/yunkunrao/yunkunrao.github.io/assets/20353538/a7aa1beb-6ea1-48d1-93a3-1278905a0b4e)

- Qemu中通过kvm_irqchip_assign_irqfd向KVM申请注册irqfd；
- 在KVM中，内核通过维护struct kvm_kernel_irqfd结构体来管理整个irqfd的流程；
- kvm_irqfd_assign：
1）分配struct kvm_kernel_irqfd结构体，并进行各个字段的初始化；
2）初始化工作队列任务，设置成irqfd_inject，用于向Guest OS注入虚拟中断；
3）初始化等待队列的唤醒函数，设置成irqfd_wakeup，当任务被唤醒时执行，在该函数中会去调度工作任务irqfd_inject；
4）初始化pll_table pt字段的处理函数，设置成irqfd_ptable_queue_proc，该函数实际是调用add_wait_queue将任务添加至eventfd的等待队列中，这个过程是在vfs_poll中完成的；
- 当Qemu通过irqfd机制发送信号时，将唤醒睡眠在eventfd上的任务，唤醒后执行irqfd_wakeup函数，在该函数中调度任务，调用irqfd_inject来注入中断；

总体效果如下图：

![image](https://github.com/yunkunrao/yunkunrao.github.io/assets/20353538/a6b0aed3-844f-4f78-9973-4fdd3e2cc478)

# vhost-net

## 1. 数据结构

vhost-net内核模块的层次结构如下图：

![image](https://github.com/yunkunrao/yunkunrao.github.io/assets/20353538/e6f5950a-75c3-46de-9510-bf6516db4542)

- struct vhost_net：用于描述Vhost-Net设备。它包含几个关键字段：1）struct vhost_dev，通用的vhost设备，可以类比struct device结构体内嵌在其他特定设备的结构体中；2）struct vhost_net_virtqueue，实际上对struct vhost_virtqueue进行了封装，用于网络包的数据传输；3）struct vhost_poll，用于socket的poll，以便在数据包接收与发送时进行任务调度；

- struct vhost_dev：描述通用的vhost设备，可内嵌在基于vhost机制的其他设备结构体中，比如struct vhost_net，struct vhost_scsi等。关键字段如下：1）vqs指针，指向已经分配好的struct vhost_virtqueue，对应数据传输；2）work_list，任务链表，用于放置需要在vhost_worker内核线程上执行的任务；3）worker，用于指向创建的内核线程，执行任务列表中的任务；

- struct vhost_virtqueue：用于描述设备对应的virtqueue，这部分内容可以参考之前virtqueue机制分析，本质上是将Qemu中virtqueue处理机制下沉到了Kernel中。关键字段如下：1）struct vhost_poll，用于poll eventfd对应的文件，当不满足处理请求时会添加到eventfd对应的等待队列中，而一旦被唤醒，该结构体中的struct vhost_work（执行函数被初始化为handle_tx_kick，以发送为例）将被放置到内核线程中去执行；

结构体的核心围绕着数据和通知机制，其中数据在vhost_virtqueue中体现，而通知主要是通过vhost_poll来实现，具体的细节下文将进一步描述。

## 2. 流程分析

### 2.1 初始化

vhost-net为内核模块，注册为misc设备，Qemu通过系统调用接口与内核交互，Qemu中的初始化如下图：

![image](https://github.com/yunkunrao/yunkunrao.github.io/assets/20353538/befcb5ca-6af5-499a-a68e-a3fc169b19b8)

- Qemu中tap设备初始化在net_init_tap中完成，其中net_init_tap_one打开vhost-net设备文件，用于与内核的vhost-net交互；
- vhost_set_backend_type：设置vhost的后端类型，以及vhost的操作函数集。目前有两种vhost后端，一种是在内核态实现的virtio后端，一种是在用户态中实现的virtio后端；
- kernel_ops：vhost的内核操作函数集，都是一些回调函数的实现，最终会通过vhost_kernel_call-->ioctl-->vhost-net.ko路径，进行配置；

ioctl系统调用，与驱动交互简单来说可以分为三大类，下边分别介绍几个关键的设置：

![image](https://github.com/yunkunrao/yunkunrao.github.io/assets/20353538/90eb65cc-49f0-4ea9-b23a-e4e764ebe4ce)

1. vhost net设置

- VHOST_SET_OWNER：底层会为调用者创建一个内核线程，对应到前文中数据结构中的vhost_worker，同时在vhost_dev结构体中还会保存调用者线程的内存空间数据结构；
- VHOST_NET_SET_BACKEND：设置vhost-net的后端设备，比如Qemu往内核态传递的tap设备对应的fd，从而让vhost-net直接与tap设备进行通信；

2. vhost dev设置

从Guest OS中的虚拟地址到最终的Host上的物理地址映射关系如上图所示，如果在Guest OS中要将数据发送出去，实际上只需要将Qemu中关于Guest OS的物理地址布局信息传递下去，此外再结合VHOST_SET_OWNER时传递的内存空间信息，就可以根据映射关系找到Guest OS中的数据对应到Host之上的物理地址，完成最后搬运即可；

VHOST_SET_MEM_TABLE：将Qemu中的虚拟机物理地址布局信息传递给内核，为了解释清楚这个问题，可以回顾一下之前内存虚拟化中的一张图：

![image](https://github.com/yunkunrao/yunkunrao.github.io/assets/20353538/3b867dc4-3e27-43c9-9e3c-f06dd00219c9)

3. vhost vring设置

- VHOST_SET_VRING_KICK：设置vhost-net模块前端virtio驱动发送通知时触发的eventfd，通知机制，最终触发handle_kick函数的执行；
- VHOST_SET_VRING_CALL：设置vhost-net后端到虚拟机virtio前端的中断通知，参考之前文章中的irqfd机制；
- 此外关于vring的设备还包括vring的大小，地址信息等；

上述的这些设置的流程路径如下，只画出了关键路径：

![image](https://github.com/yunkunrao/yunkunrao.github.io/assets/20353538/7b2d51af-7493-4672-9dc2-f96027aa0870)

- 当Guest OS中的virtio-net驱动完成初始化后，会通过vp_set_status来设置状态，以通知后端驱动已经ready，此时会触发VM的退出并进入KVM进行异常处理，最终路由给Qemu；
- Qemu中的vcpu线程监测异常，当检测到KVM_EXIT_MMIO时，去回调注册该IO区域的读写函数，比如virtio_pci_common_write函数，在该函数中逐级往下最终调用到vhost_net_start函数；
- 在vhost_net_start中最终去通过kernel_ops函数集去设置底层并交互；

初始化完成后，接下来让我们看看数据的发送与接收，为了能将整个流程表达清楚，我会将完整的图拆分成几个步骤来讲述。

 ### 2.2 数据发送

 1 ）

发送前的框图如下：

![image](https://github.com/yunkunrao/yunkunrao.github.io/assets/20353538/bcda233d-aea6-4936-9d7d-3f199e3ab0af)

- Guest OS中的virtio-net驱动中维护两个virtqueue，分别用于发送和接收；
- 图中的datagram表示的是需要发送的数据；
- KVM模块提供了ioeventfd和irqfd用于通知机制；
- vhost-net模块中创建好了vhost_worker内核线程，用于处理任务；

2）

![image](https://github.com/yunkunrao/yunkunrao.github.io/assets/20353538/84b04db3-4467-4177-8ca0-910fb25b1f63)

- 当数据包准备好之后，通过往kick fd上触发信号，从而唤醒vhost_worker内核线程来调用handle_tx_kick进行数据的发送；
- 当Tap/Tun不具备发送条件时，vhost_worker会poll在socket上，等待Tap/Tun的唤醒，一旦被唤醒后可以调用handle_tx_net发送；
- 最终的handle_tx完成具体的发送；

3）

![image](https://github.com/yunkunrao/yunkunrao.github.io/assets/20353538/b1b5abfc-cf10-4a13-88d4-da26efbcc0ba)

- vhost_get_vq_desc函数在vritqueue中查找可用的buffer，并将信息存储到iov中，以便更好的访问；
- sock->ops->sendmsg()函数，实际调用的是tun_sendmsg函数，在该函数中分配了skb结构体，并将iov[]中的信息传递过来，最终如图中所示完成数据的拷贝和发送，通过NIC发送出去；

4）

![image](https://github.com/yunkunrao/yunkunrao.github.io/assets/20353538/0fff4255-de4a-43c3-99dc-e9455c062d18)

- 数据发送完毕后，通过irqfd机制通知vcpu；

### 2.3 数据接收

数据的接收是发送的逆过程，流程一致：

1）

![image](https://github.com/yunkunrao/yunkunrao.github.io/assets/20353538/d71ea193-6024-4575-a431-4463e7fe74b3)

初始化部分与发送过程一致；
Tap/Tun驱动从NIC接收到数据包，准备发送给vhost-net；

2）

![image](https://github.com/yunkunrao/yunkunrao.github.io/assets/20353538/4165f08e-2d7e-4754-a62a-62aeeb59512c)

- vhost-net中的vhost_worker线程也poll在两个fd之上，与发送端类似；
- kick fd上触发信号时最终调用handle_rx_kick函数，Tap/Tun对应的socket上触发信号时，调用handle_rx_net函数；
- 最终通过handle_rx来完成实际的接收；

3）

![image](https://github.com/yunkunrao/yunkunrao.github.io/assets/20353538/5ad62427-3474-43ab-82c2-6f9c544c3a8d)

- 接收过程中，vhost_get_vq_desc获取virtqueue中的可用buffer，并将信息存储到iov[]中；
- sock->ops->recvmsg()函数实际指向tun_recvmsg函数，在该函数中最终完成数据的传递；

4）

![image](https://github.com/yunkunrao/yunkunrao.github.io/assets/20353538/3739c03f-89cb-41e7-af84-12366f64097e)

- 数据接收完成后，通过irqfd机制通过vcpu，从而在Guest OS中进行处理；



# 参考文献

- http://blog.vmsplice.net/2011/09/qemu-internals-vhost-architecture.html
- https://www.redhat.com/en/blog/virtqueues-and-virtio-ring-how-data-travels
- https://mp.weixin.qq.com/s/9Zv7XcsvlVwDDjTn85-SZQ
