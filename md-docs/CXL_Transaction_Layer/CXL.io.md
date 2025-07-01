
# CXL.io
## 1.CXL.io协议
### 1.1 概述与核心作用

CXL.io 是 CXL (Compute Express Link) 协议栈中的基础层，它为系统中的 I/O 设备提供了非一致性的加载/存储 (load/store) 接口。从根本上说，CXL.io 的设计完全基于 PCIe® 规范，并直接复用了其核心机制。这意味着 CXL.io 在交易类型、包格式、基于信用的流量控制、虚拟通道管理和交易排序规则等方面都遵循 PCIe 的定义。

在整个 CXL Flex Bus 架构中，CXL.io 与 CXL.cache、CXL.mem 协议并行存在于交易层，它们共享底层的链路层和物理层。CXL.io 的核心作用可以理解为 CXL 体系的基础框架和管理通道，它不仅处理传统的 I/O 事务，还负责设备的发现、配置以及传输关键的控制消息。

### 1.2 设备端点与枚举

一个 CXL 设备能够被主机操作系统识别和管理，完全依赖于 CXL.io 提供的 PCIe 兼容接口。

  * **设备枚举** ：系统通过标准的 PCIe 枚举流程，经由 CXL.io 发现 CXL 设备及其功能。
  * **中断处理** ：规范对中断消息（INTx）有特殊规定。如果一个功能 (Function) 参与了 CXL.cache 或 CXL.mem 协议，它就不能生成 INTx 消息。不过，不参与这两个协议的纯 I/O 功能（通过 Non-CXL Function Map DVSEC 枚举）可以生成 INTx 消息，尽管这不被推荐。

### 1.3 CXL 特定的厂商定义消息 (VDM)

CXL.io 的一个关键用途是作为通道，来传输 CXL 协议专用的控制和状态消息。这是通过复用 PCIe 的厂商定义消息 (Vendor Defined Message, VDM) 机制实现的。这些 VDM 都使用 CXL 的专属厂商 ID `1E98h`。

- 电源管理 (PM) VDM

    - 这些消息用于协调主机和设备之间的电源状态转换，例如 PMREQ (电源请求)、PMRSP (电源响应) 等。

-  错误 (Error) VDM
   - 这些消息主要用于设备向主机异步报告错误或重要事件。

- CXL Fabric 相关 VDM
  - 在 CXL Fabric（一种更复杂的拓扑结构）中，VDM 被广泛用于各种管理和控制流程。

### 1.4. 关键交互与功能要求

- 强制要求的 PCIe 功能

  - 为了保证 CXL 的正常工作，规范要求所有 CXL 设备都必须支持以下几项在 PCIe 中被定义为 “可选” 的功能：

    * **传输器端的数据中毒 (Data Poisoning by transmitter)**。
    * **地址转换服务 (ATS)** ：对于支持 CXL.cache 的设备（Type 1 和 Type 2）是必需的。
    * **高级错误报告 (AER)**。

- 通过 ATS 实现内存访问类型指示

  - CXL.io 在协调 CXL.cache 访问方面扮演着至关重要的 “守门人” 角色。

    * **背景** ：某些内存区域（如被映射为 Uncacheable 类型的内存）只能通过 CXL.io 进行非一致性访问，而不能使用 CXL.cache 发起一致性请求。
    * **工作机制** ：
      1. CXL 设备在尝试通过 CXL.cache 访问某个地址前，必须先通过 **CXL.io** 路径发送一个 ATS (Address Translation Service) 请求，以获取主机物理地址 (HPA)。
      2. 主机在回复的 ATS 完成消息中，会包含一个特殊的 `CXL.io` 位。
      3. 如果这个 `CXL.io` 位被设置为 `1`，则表示该地址只允许通过 CXL.io 进行访问。设备若尝试使用 CXL.cache 访问该地址，则构成协议违规。

### 1.5 CXL Fabric 中的特定头部

在 PBR (Port Based Routing) Fabric 这种高级拓扑中，CXL.io 的 TLP (Transaction Layer Packet) 格式有一些特殊之处。

  * **PBR TLP 报头 (PTH)** ：在 PBR 链路上，几乎所有 io TLP 前面都会附加一个 1-DWORD 的 PTH。这个报头包含了源和目的端口 ID 等信息，是实现精确路由的关键。
  * **VendPrefixLO** ：在非 MLD（Multi-Logical Device）的边缘链路上，这个本地前缀被用来携带 PBR-ID，以支持主机和设备间的跨域通信。

### 1.6 总结

CXL.io 远不止是一个简单的 I/O 数据通道。它是整个 CXL 体系的基石和控制总线，承担着至关重要的底层任务：

  1. **身份与发现** ：它是系统识别和枚举 CXL 设备的唯一途径。
  2. **管理与控制** ：它承载了 CXL 特有的电源管理、错误报告和 Fabric 控制等关键消息。
  3. **规则与边界** ：它通过 ATS 机制，严格划分了 CXL.cache 和 CXL.io 的访问权限，是实现混合协议访问模型的关键。