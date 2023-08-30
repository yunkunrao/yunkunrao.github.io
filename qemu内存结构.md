1.1 Memory Region & Address Space & Device

![image](https://github.com/yunkunrao/yunkunrao.github.io/assets/20353538/dafd975e-f386-4aa9-b5cb-e98d32d53608)

- 从下往上看，首先每一个Memory Region对象都包含了RAMBlock，RAMBlock是QEMU真正向Host alloc申请空间的数据结构，它的大小可以指定；而Memory Region是从RAMBlock进行了抽象，有助于Device操作内存空间。
  - QEMU维护了RAMList，每申请一个RAMBlock(Memory Region)添加到List中

- Address Space包含了一系列Memory Region，在当前上下文空间中，通过地址空间对当前一系列存储对象进行一个统一抽象，很抽象举个例子
  - 比如当前是CPU正在运行的状态，就有了一个Address Space名为CPU-system，此时的寻址空间，高位是操作系统内核空间，中间有0x40000000~0x7FFFFFFF共4G的地址是用于RAM内存空间，还有一些地址用于I/O
  - 后来需要I/O，进入I/O状态后，又有了一个Address Space名为I/O，此时的所有地址都是I/O地址

-  对于memory-backend，可以不用理解，只需要知道大多数device都需要和存储交互，操作的对象是Memory Region即可

![image](https://github.com/yunkunrao/yunkunrao.github.io/assets/20353538/569783e5-c9a9-4a61-8cdc-de3a3e2c86d0)

1.2 FlatView结构

![image](https://github.com/yunkunrao/yunkunrao.github.io/assets/20353538/df925706-8665-40e9-8874-d865966fc3d8)

每一个flat range都投射到同一个地址空间的平面上，而上图中的R1，R2等对应的则是struct MemoryRegionSection。

