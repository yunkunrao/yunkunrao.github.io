# Linux PCI 驱动架构分析

## 1. 概述

作为专题的第一篇，当然会先从硬件总线入手。
进入主题前，先讲点背景知识。
在PC时代，随着处理器的发展，经历了几代I/O总线的发展，解决的问题都是CPU主频提升与外部设备访问速度的问题：

第一代总线包含ISA、EISA、VESA和Micro Channel等；
第二代总线包含PCI、AGP、PCI-X等；
第三代总线包含PCIe、mPCIe、m.2等；
PCIe（PCI Express）是目前PC和嵌入式系统中最常用的高速总线，PCIe在PCI的基础上发展而来，在软件上PCIe与PCI是后向兼容的，PCI的系统软件可以用在PCIe系统中。

本文会分两部分展开，先介绍PCI总线，然后再介绍PCIe总线，方便在理解上的过渡，开始旅程吧。

## 2. PCI Local Bus

### 2.1 PCI总线组成

- PCI总线（Peripheral Component Interconnect，外部设备互联），由Intel公司提出，其主要功能是连接外部设备；
- PCI Local Bus，PCI局部总线，局部总线技术是PC体系结构发展的一次变革，是在ISA总线和CPU总线之间增加的一级总线或管理层，可将一些高速外设，如图形卡、硬盘控制器等从ISA总线上卸下，而通过局部总线直接挂接在CPU总线上，使之与高速CPU总线相匹配。PCI总线，指的就是PCI Local Bus。

先来看一下PCI Local Bus的系统架构图：

![image](https://github.com/yunkunrao/yunkunrao.github.io/assets/20353538/1943d9cf-2db4-4ed6-a06e-40162f60e649)

从图中看，与PCI总线相关的模块包括：

1. Host Bridge，比如PC中常见的North Bridge（北桥）；
图中处理器、Cache、内存子系统通过Host Bridge连接到PCI上，Host Bridge管理PCI总线域，是联系处理器和PCI设备的桥梁，完成处理器与PCI设备间的数据交换。其中数据交换，包含处理器访问PCI设备的地址空间和PCI设备使用DMA机制访问主存储器，在PCI设备用DMA访问存储器时，会存在Cache一致性问题，这个也是Host Bridge设计时需要考虑的；
此外，Host Bridge还可选的支持仲裁机制，热插拔等；

2. PCI Local Bus；
PCI总线，由Host Bridge或者PCI-to-PCI Bridge管理，用来连接各类设备，比如声卡、网卡、IDE接口等。可以通过PCI-to-PCI Bridge来扩展PCI总线，并构成多级总线的总线树，比如图中的PCI Local Bus #0和PCI Local Bus #1两条PCI总线就构成一颗总线树，同属一个总线域；

3. PCI-To-PCI Bridge；
PCI桥，用于扩展PCI总线，使采用PCI总线进行大规模系统互联成为可能，管理下游总线，并转发上下游总线之间的事务；

4. PCI Device；
PCI总线中有三类设备：PCI从设备，PCI主设备，桥设备。
PCI从设备：被动接收来自Host Bridge或者其他PCI设备的读写请求；
PCI主设备：可以通过总线仲裁获得PCI总线的使用权，主动向其他PCI设备或主存储器发起读写请求；
桥设备：管理下游的PCI总线，并转发上下游总线之间的总线事务，包括PCI桥、PCI-to-ISA桥、PCI-to-Cardbus桥等。

### 2.2 PCI总线信号定义

PCI总线是一条共享总线，可以挂接多个PCI设备，PCI设备通过一系列信号与PCI总线相连，包括：地址/数据信号、接口控制信号、仲裁信号、中断信号等。如下图：

![image](https://github.com/yunkunrao/yunkunrao.github.io/assets/20353538/6eb3bea7-c6d2-479e-a50e-74c276e4ee3a)

- 左侧红色框里表示的是PCI总线必需的信号，而右侧蓝色框里表示的是可选的信号；
- AD[31:00]：地址与数据信号复用，在传送时第一个时钟周期传送地址，下一个时钟周期传送数据；
- C/BE[3:0]#：PCI总线命令与字节使能信号复用，在地址周期中表示的是PCI总线命令，在数据周期中用于字节选择，可以进行单字节、字、双字访问；
- PAR：奇偶校验信号，确保AD[31:00]和C/BE[3:0]#传递的正确性；
- Interface Control：接口控制信号，主要作用是保证数据的正常传递，并根据PCI主从设备的状态，暂停、终止或者正常完成总线事务：
  - FRAME#：表示PCI总线事务的开始与结束；
  - IRDY#：信号由PCI主设备驱动，信号有效时表示PCI主设备数据已经ready；
  - TRDY#：信号由目标设备驱动，信号有效时表示目标设备数据已经ready；
  - STOP#：目标设备请求主设备停止当前总线事务；
  - DEVSEL#：PCI总线的目标设备已经准备好；
  - IDSEL：PCI总线在配置读写总线事务时，使用该信号选择PCI目标设备；
- Arbitration：仲裁信号，由REQ#和GNT#组成，与PCI总线的仲裁器直接相连，只有PCI主设备需要使用该组信号，每条PCI总线上都有一个总线仲裁器；
- Error Reporting：错误信号，包括PERR#奇偶校验错误和SERR系统错误；
- System：系统信号，包括时钟信号和复位信号；

看一下C/BE[3:0]都有哪些命令吧：

![image](https://github.com/yunkunrao/yunkunrao.github.io/assets/20353538/23f26655-b170-409e-a40c-44fc0ec4624e)

### 2.3 PCI事务模型

PCI使用三种模型用于数据的传输：

![image](https://github.com/yunkunrao/yunkunrao.github.io/assets/20353538/204e9695-276c-4dd1-9699-b37052d93808)

1. Programmed I/O：通过IO读写访问PCI设备空间；
2. DMA：PIO的方式比较低效，DMA的方式可以直接去访问主存储器而无需CPU干预，效率更高；
3. Peer-to-peer：两台PCI设备之间直接传送数据；

### 2.4 PCI总线地址空间映射

PCI体系架构支持三种地址空间：

![image](https://github.com/yunkunrao/yunkunrao.github.io/assets/20353538/c423fe81-b926-4412-9392-fd219b3db1f9)

1. memory空间：
针对32bit寻址，支持4G的地址空间，针对64bit寻址，支持16EB的地址空间；

2. I/O空间
PCI最大支持4G的IO空间，但受限于x86处理器的IO空间（16bits带宽），很多平台将PCI的IO地址空间限定在64KB；

3. 配置空间

x86 CPU可以直接访问memory空间和I/O空间，而配置空间则不能直接访问；
每个PCI功能最多可以有256字节的配置空间；
PCI总线在进行配置的时候，采用ID译码方式，使用设备的ID号，包括Bus Number，Device Number，Function Number和Register Number，每个系统支持256条总线，每条总线支持32个设备，每个设备支持8个功能，由于每个功能最多有256字节的配置空间，因此总的配置空间大小为：256B * 8 * 32 * 256 = 16M；

有必要再进一步介绍一下配置空间：
x86 CPU无法直接访问配置空间，通过IO映射的数据端口和地址端口间接访问PCI的配置空间，其中地址端口映射到0CF8h - 0CFBh，数据端口映射到0CFCh - 0CFFh；

![image](https://github.com/yunkunrao/yunkunrao.github.io/assets/20353538/564a373b-e03a-4708-96b6-aadf097547f1)

- 图为配置地址寄存器构成，PCI的配置过程分为两步：
CPU写CF8h端口，其中写的内容如图所示，BUS，Device，Function能标识出特定的设备功能，Doubleword来指定配置空间的具体某个寄存器；
CPU可以IO读写CFCh端口，用于读取步骤1中的指定寄存器内容，或者写入指定寄存器内容。这个过程有点类似于通过I2C去配置外接芯片；
那具体的配置空间寄存器都是什么样的呢？每个功能256Byte，前边64Byte是Header，剩余的192Byte支持可选功能。有种类型的PCI功能：Bridge和Device，两者的Header都不一样。

- Bridge
![image](https://github.com/yunkunrao/yunkunrao.github.io/assets/20353538/7622e2f2-804e-48cd-91c0-3b99e0e9047a)

- Device
![image](https://github.com/yunkunrao/yunkunrao.github.io/assets/20353538/2534ee5f-ef57-4802-aca3-2f774f7b899a)

配置空间中有个寄存器字段需要说明一下：Base Address Register，也就是BAR空间，当PCI设备的配置空间被初始化后，该设备在PCI总线上就会拥有一个独立的PCI总线地址空间，这个空间就是BAR空间，BAR空间可以存放IO地址空间，也可以存放存储器地址空间。

- PCI总线取得了很大的成功，但随着CPU的主频不断提高，PCI总线的带宽也捉襟见肘。此外，它本身存在一些架构上的缺陷，面临一系列挑战，包括带宽、流量控制、数据传送质量等；
- PCIe应运而生，能有效解决这些问题，所以PCIe才是我们的主角；

## 3. PCI Express

### 3.1 PCIe体系结构

先看一下PCIe架构的组成图：

![image](https://github.com/yunkunrao/yunkunrao.github.io/assets/20353538/d5352db7-50c3-45a3-91b2-7a89e653d43e)

- Root Complex：CPU和PCIe总线之间的接口可能会包含几个模块（处理器接口、DRAM接口等），甚至可能还会包含芯片，这个集合就称为Root Complex，它作为PCIe架构的根，代表CPU与系统其它部分进行交互。广义来说，Root Complex可以认为是CPU和PCIe拓扑之间的接口，Root Complex会将CPU的request转换成PCIe的4种不同的请求（Configuration、Memory、I/O、Message）；
- Switch：从图中可以看出，Swtich提供扇出能力，让更多的PCIe设备连接在PCIe端口上；
- Bridge：桥接设备，用于去连接其他的总线，比如PCI总线或PCI-X总线，甚至另外的PCIe总线；
- PCIe Endpoint：PCIe设备；
- 图中白色的小方块代表Downstream端口，灰色的小方块代表Upstream端口；

前文提到过，PCIe在软件上保持了后向兼容性，那么在PCIe的设计上，需要考虑在PCI总线上的软件视角，比如Root Complex的实现可能就如下图所示，从而看起来与PCI总线相差无异：

![image](https://github.com/yunkunrao/yunkunrao.github.io/assets/20353538/ece53b10-a69d-4350-9262-898cd6660496)

- Root Complex通常会实现一个内部总线结构和多个桥，从而扇出到多个端口上；
- Root Complex的内部实现不需要遵循标准，因此都是厂家specific的；

而Switch的实现可能如下图所示：

![image](https://github.com/yunkunrao/yunkunrao.github.io/assets/20353538/79376960-1f28-470b-8476-c3a3088d3068)

- Switch就是一个扩展设备，所以看起来像是各种桥的连接路由；

### 3.2 PCIe数据传输

![image](https://github.com/yunkunrao/yunkunrao.github.io/assets/20353538/2ad53e14-a4a9-4102-a997-d60c864c5230)

- 与PCI总线不同（PCI设备共享总线），PCIe总线使用端到端的连接方式，互为接收端和发送端，全双工，基于数据包的传输；
- 物理底层采用差分信号（PCI链路采用并行总线，而PCIe链路采用串行总线），一条Lane中有两组差分信号，共四根信号线，而PCIe Link可以由多条Lane组成，可以支持1、2、4、8、12、16、32条；

PCIe规范定义了分层的架构设计，包含三层：

![image](https://github.com/yunkunrao/yunkunrao.github.io/assets/20353538/cb20afce-6852-41fa-ab41-3fc8cda49176)

1. Transaction层
负责TLP包（Transaction Layer Packet）的封装与解封装，此外还负责QoS，流控、排序等功能；

2. Data Link层
负责DLLP包（Data Link Layer Packet）的封装与解封装，此外还负责链接错误检测和校正，使用Ack/Nak协议来确保传输可靠；

3. Physical层
负责Ordered-Set包的封装与解封装，物理层处理TLPs、DLLPs、Ordered-Set三种类型的包传输；

数据包的封装与解封装，与网络包的创建与解析很类似，如下图：

![image](https://github.com/yunkunrao/yunkunrao.github.io/assets/20353538/3d4f13fe-1f3c-4696-b937-ccafdbd7b6ab)

封装的时候，在Payload数据前添加各种包头，解析时是一个逆向的过程；

来一个更详细的PCIe分层图：

![image](https://github.com/yunkunrao/yunkunrao.github.io/assets/20353538/bb90c729-8814-4a0d-9b25-8ae405cf6573)

### 3.3 PCIe设备的配置空间

为了兼容PCI软件，PCIe保留了256Byte的配置空间，如下图：

![image](https://github.com/yunkunrao/yunkunrao.github.io/assets/20353538/5574a3ef-a409-4497-a27f-b5e2f3a514c8)

此外，在这个基础上将配置空间扩展到了4KB，还进行了功能的扩展，比如Capability、Power Management、MSI中断等：

![image](https://github.com/yunkunrao/yunkunrao.github.io/assets/20353538/9a8c12b8-9375-4bc0-afa8-4829cdabd052)

- 扩展后的区域将使用MMIO的方式进行访问；
 

## 4. 数据结构

### 4.1 PCI总线拓扑

![image](https://github.com/yunkunrao/yunkunrao.github.io/assets/20353538/47702d44-0f0c-456c-94bf-799fd339c776)

- PCI体系结构的拓扑关系如图所示，而图中的不同数据结构就是用于来描述对应的模块；
- Host Bridge连接CPU和PCI系统，由struct pci_host_bridge描述；
- struct pci_dev描述PCI设备，以及PCI-to-PCI桥设备；
- struct pci_bus用于描述PCI总线，struct pci_slot用于描述总线上的物理插槽；

来一张更详细的结构体组织图：

![image](https://github.com/yunkunrao/yunkunrao.github.io/assets/20353538/bbe8d288-7c35-4af7-81ac-f9e3869ada7c)

- 总体来看，数据结构对硬件模块进行了抽象，数据结构之间也能很便捷的构建一个类似PCI子系统物理拓扑的关系图；
- 顶层的结构为pci_host_bridge，这个结构一般由Host驱动负责来初始化创建；
- pci_host_bridge指向root bus，也就是编号为0的总线，在该总线下，可以挂接各种外设或物理slot，也可以通过PCI桥去扩展总线；

### 4.2 设备驱动模型

Linux PCI驱动框架，基于Linux设备驱动模型，因此有必要先简要介绍一下，实际上Linux设备驱动模型也是一个大的topic，先挖个坑，有空再来填。来张图吧：

![image](https://github.com/yunkunrao/yunkunrao.github.io/assets/20353538/57970cea-0d58-46ac-825e-2beb8b21103d)

- 简单来说，Linux内核建立了一个统一的设备模型，分别采用总线、设备、驱动三者进行抽象，其中设备与驱动都挂在总线上，当有新的设备注册或者新的驱动注册时，总线会去进行匹配操作（match函数），当发现驱动与设备能进行匹配时，就会执行probe函数的操作；
- 从数据结构中可以看出，bus_type会维护两个链表，分别用于挂接向其注册的设备和驱动，而match函数就负责匹配检测；
- 各类驱动框架也都是基于图中的机制来实现，在这之上进行封装，比如I2C总线框架等；
- 设备驱动模型中，包含了很多kset/kobject等内容，建议去看看之前的文章《linux设备模型之kset/kobj/ktype分析》

### 4.3 初始化

既然说到了设备驱动模型，那么首先我们要做的事情，就是先在内核里边创建一个PCI总线，用于挂接PCI设备和PCI驱动，我们的实现来到了pci_driver_init()函数：

![image](https://github.com/yunkunrao/yunkunrao.github.io/assets/20353538/8a87ce02-b732-483c-93e0-72470f2a8171)

内核在PCI框架初始化时会调用pci_driver_init()来创建一个PCI总线结构（全局变量pci_bus_type），这里描述的PCI总线结构，是指驱动匹配模型中的概念，PCI的设备和驱动都会挂在该PCI总线上；
从pci_bus_type的函数操作接口也能看出来，pci_bus_match用来检查设备与驱动是否匹配，一旦匹配了就会调用pci_device_probe函数，下边针对这两个函数稍加介绍；

#### 4.3.1 pci_bus_match

![image](https://github.com/yunkunrao/yunkunrao.github.io/assets/20353538/1b463ccf-e40e-4343-b200-beabf4405891)

设备或者驱动注册后，触发pci_bus_match函数的调用，实际会去比对vendor和device等信息，这个都是厂家固化的，在驱动中设置成PCI_ANY_ID就能支持所有设备；
一旦匹配成功后，就会去触发pci_device_probe的执行；

#### 4.3.2 pci_device_probe

![image](https://github.com/yunkunrao/yunkunrao.github.io/assets/20353538/ae097ab0-f34d-4dc5-abfd-3c7feee6b405)

实际的过程也是比较简单，无非就是进行匹配，一旦匹配上了，直接调用驱动程序的probe函数，写过驱动的同学应该就比较清楚后边的流程了；

### 4.4 枚举

- 我们还是顺着设备驱动匹配的思路继续开展；
- 3.2节描述的是总线的创建，那么本节中的枚举，显然就是设备的创建了；
- 所谓设备的创建，就是在Linux内核中维护一些数据结构来对硬件设备进行描述，而硬件的描述又跟上文中的数据结构能对应上；

枚举的入口函数：pci_host_probe

![image](https://github.com/yunkunrao/yunkunrao.github.io/assets/20353538/ba59e058-8dae-48d7-964e-84da872c7880)

- 设备的扫描从pci_scan_root_bus_bridge开始，首先需要先向系统注册一个host bridge，在注册的过程中需要创建一个root bus，也就是bus 0，在pci_register_host_bridge函数中，主要是一系列的初始化和注册工作，此外还为总线分配资源，包括地址空间等；
- pci_scan_child_bus开始，从bus 0向下扫描并添加设备，这个过程由pci_scan_child_bus_extend来完成；
- 从pci_scan_child_bus_extend的流程可以看出，主要有两大块：
  - PCI设备扫描，从循环也能看出来，每条总线支持32个设备，每个设备支持8个功能，扫描完设备后将设备注册进系统，pci_scan_device的过程中会去读取PCI设备的配置空间，获取到BAR的相关信息，细节不表了；
  - PCI桥设备扫描，PCI桥是用于连接上一级PCI总线和下一级PCI总线的，当发现有下一级总线时，创建子结构，并再次调用pci_scan_child_bus_extend的函数来扫描下一级的总线，从这个过程看，就是一个递归过程。
- 从设备的扫描过程看，这是一个典型的DFS（Depth First Search）过程，熟悉数据结构与算法的同学应该清楚，这就类似典型的走迷宫的过程；

如果你对上述的流程还不清楚，再来一张图：

![image](https://github.com/yunkunrao/yunkunrao.github.io/assets/20353538/f34f999e-82ea-44d6-9b50-ee4c0e22e065)

- 图中的数字代表的就是扫描的过程，当遍历到PCI桥设备的时候，会一直穷究到底，然后再返回来；
- 当枚举过程结束后，系统中就已经维护了PCI设备的各类信息了，在设备驱动匹配模型中，总线和设备都已经具备了，剩下的就是写个驱动了；



