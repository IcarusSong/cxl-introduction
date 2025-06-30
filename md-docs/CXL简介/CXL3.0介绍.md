
# CXL3.0介绍
CXL3.0引入/拓展了以下功能：
1. Fabric capabilities and fabric attached memory
2. Enhance fabric management framework
3. Memory pooling and sharing
4. Peer-to-peer memory access
5. Multi-level switching 
6. Near memory processing
7. Multi-headed devices
8. Multiple Type 1/Type 2 devices per root port 
9. Supports PCIe® 6.0 

## 双倍带宽
每引脚带宽翻倍，同时保持延迟不变
CXL3.0采用PCIe 6.0® PHY @ 64 GT/s，采用256B Flit
![](https://pic1.imgdb.cn/item/6862215458cb8da5c87edb02.png)
- **Flit 模式**：
    
    - **256 字节**：高速模式下整合数据 + **Reed-Solomon FEC** 纠错，带宽优先。
        
    - **LO 模式**：分拆为两半（128 字节），前半含头部 + **精简 CRC**，支持提前处理，**降低延迟 2ns**。
        
    - **68 字节**：兼容旧设备，低速率强制支持。

## 多级交换互连
在CXL2.0中，Host到Device之间只能由一个CXL Switch（单层交换），而在CXL3.0中可以**支持多级交换**。
![](https://pic1.imgdb.cn/item/6862216658cb8da5c87edbb4.png)
在CXL2.0中，Host想要连接多种设备，只能通过root port连接CXL Switch进行连接多种设备；而在**CXL3.0中，使得单个主机端口可以同时挂载多种设备，并且设备数目也可以更多（最多16个CXL.cache设备）**。

## 内存池化与内存共享
![](https://pic1.imgdb.cn/item/6862217358cb8da5c87edc3a.png)
CXL3.0支持池化内存共享，**device的一块内存区域可以被多个host共享**，由于在CXL2.0定义中只有CXL.cache协议可以被用来交换cache一致性信息，但是他的优先级没有M2S的内存读的操作优先级高，因此会被block， 由此在memory sharing的场景中可能会导致读写的数据是失效的。

为了解决共享内存在多host中的一致性（Type2、Type3），CXL3.0在CXL.mem中增加了Back-Invalidation机制，**当由一个host对共享内存进行修改时，设备可以通过Back-Invalidation通知其它host使得对该共享内存对应的cache line进行无效化。**
![](https://pic1.imgdb.cn/item/6862218a58cb8da5c87edd23.png)
### Global Fabric Attached Memory (GFAM)
CXL3.0 还引入了**全局网络结构的附加内存（Global Fabric Attached Memory，GFAM）**的概念，它通过**将内存从处理单元中解聚出来，实现了共享大内存池**。这种架构允许内存与多个处理器直接连接，或者通过 CXL 交换机进行连接，从而提高了内存的灵活性和利用率。GFAM 架构的一个关键特点是，它允许**内存池中的内存可以是相同类型或不同类型**。GFAM最多**可以支持4095个host使用**。

## Multi-Headed Device
在 CXL 3.0 规范中，**多头设备（Multi-Head Device，MH-Device）** 是指具有多个 CXL 端口的 Type 3 设备。每个端口被称为一个“头（Head）”，这些端口允许设备与多个主机或交换机进行连接，以实现更高的带宽、冗余性和灵活的拓扑结构。

**多头设备的类型**：

1. **MH-SLD（Multi-Head Single Logical Device）**：那么每个CXL端口(头)上呈现的都是SLD（虽然也是在内部进行了MLD的逻辑设备划分）

2. **MH-MLD（Multi-Head Multiple Logical Device）**：：在任意CXL端口上都可能呈现的是MLD。

多头设备需要经过两层解码，首先是CXL端口到逻辑设备的映射。每个逻辑设备的资源管理按照MLD来。

逻辑设备在内存资源，状态，上下文，管理上都是隔离的；
每个逻辑设备一次只能映射给一个头，但是一个头可以被多个虚拟层次连接，实现内存共享。
![](https://pic1.imgdb.cn/item/686221a058cb8da5c87eddce.png)

## Fabrics
![](https://pic1.imgdb.cn/item/686221b158cb8da5c87ede42.png)
![](https://pic1.imgdb.cn/item/686221bf58cb8da5c87edea8.png)
CXL3.0中引入了**多级交换，并且突破了树状结构，成为一个网状的Fabrics，并且不再严格区分host和device，每个Node都可以是host或者device**。
## Device peer-to-peer communication (P2P)
![](https://pic1.imgdb.cn/item/686221e058cb8da5c87edf9a.png)
CXL 3.0通过**back-invalidate来支持内存共享，同时结合UIO实现Device不经过Host的点到点的通信**

无序IO（Unordered I/O ）提出的背景：是PCIe上运行的软件依赖于消费者-生产者排序模型（producer-consumer ordering model），这个排序模型针对是整个系统，不管事务访问的设备是否相同，不管访问的地址位置是否相同，不管访问的属性是否相同（比如可缓存和不可缓存）。消费者和生产者排序模型如下图(a)所示，编程模型保证生产者将数据写到一个内存位置后，后续再写入一个flag，那么系统中所有可能的消费者只要读到了这个flag就一定可以看到生产者写入的数据。在原始的PCIe系统中遵循者树状拓扑结构（层次化路由，Hierarchy Based Routing，HBR，如图42左边），设备跟设备之间只存在唯一的路径，进行保序比较容易实现。但是CXL3.0为了扩展到更多的设备高带宽低延迟互联，提出多层交换机架构和基于端口的路由（Port-Based Routing，PBR，如图42右边），打破了原有树状结构，使得节点到节点的路径不止一条。这时为了避免在路由中引入过多的保序复杂度，同时维护消费者和生产者排序模型，就引入了UIO。UIO的写操作不管是posted还是non-posted的属性，一律当作non-posted来对待，non-posted写规定必须接收到完成事务（Completion）后才能继续后面的写入，因此排序的强制要求放在了发起事务的源头来做。
![](https://pic1.imgdb.cn/item/686221f258cb8da5c87ee029.png)
![](https://pic1.imgdb.cn/item/6862220058cb8da5c87ee090.png)



### 入虚拟通道（VC）与“无序”事务

- **虚拟通道（VC）机制**  
  CXL 3.0（继承自 PCIe）通过引入多个独立的虚拟通道（VCs）来实现“无序”读/写及完成事务。每个 VC 内部通过三种流控（FC）类别管理：
  - **Posted (P)**：处理如 Memory Write 的事务。  
  - **Non-Posted (NP)**：处理如 Memory Read 的事务。  
  - **Completion (C)**：为 NP 事务提供完成响应。
  
- **无序事务的实现**  
  在这种机制下，虽然每个 VC 内部不保证事务的全局顺序，但生产者端（例如某设备）通过对同一 VC 内的事务顺序进行控制，实现了 producer-consumer 模型的顺序语义。  
  - 例如，每个 UIO Write（用户 I/O 写）必须等待其完成响应（UIO Write Completion），才能向下发出后续的“标志”信号，表明数据已就绪。

- **UIO（Unordered I/O）机制**  
  UIO Write 和 UIO Read 分别对应于 P 类和 NP 类的事务，UIO Completion 则在 C 类中处理。这样设计的目的是：  
  - 在不依赖主机 CPU 中转的前提下，实现设备间直接通信，同时保持生产者与消费者之间的必要顺序。


### Back-Invalidate（BI）机制及其作用
  
1. **直接设备间通信**：  
    - 借助 UIO 和 BI 流，设备可以直接访问 HDM-DB（Host-managed Device Memory Device-coherent with Back-invalidate support）内存，而无需经过主机 CPU 的处理，从而降低延迟和减少主机负担。  
    - 当设备对内存发起读写时，若缓存行状态不符合直接访问要求，则内存控制器通过 BI 流通知主机进行必要的缓存失效处理。
  

### 点对点通信实现的关键总结

- **多路径传输**  
  通过虚拟通道的无序事务机制，使得 CXL.io（以及继承的 PCIe 事务）能够在非树形、具有多条路径的拓扑中传输，而不会破坏生产者与消费者之间的顺序要求。

- **独立控制与局部顺序**  
  每个 VC 内部的事务顺序由发送端（生产者）自己控制，确保了数据传输的正确顺序，而不同 VC 之间则相互独立，可以在多个路径上并行传输。

- **直接设备间数据交换**  
  结合 BI 流，CXL 3.0 实现了设备可以直接访问和管理 HDM-DB 内存，既支持设备直接点对点通信，也能支持大规模、共享一致性内存的应用场景。