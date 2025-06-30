
# CXL2.0介绍
## CXL2.0
CXL2.0增加了单级交换、资源池化、持久化、QoS保障、热插拔的特性。
### 池化
CXL2.0增加了**单级交换，host和device直接可以通过一个CXL Switch互连起来**。

为了支持多主机连接和设备池化，每个主机都表示CXL topology作为一个虚拟层次（VH），包括交换机和机的端口以及每个具有设备资源的端口的Virtual Bridge一个物理CXL交换机在内部虚拟成多个虚拟CXL交换机(VCS，Virtual CXL Switch)。连接上CXL交换机的每个主机可以看到一个独立的VCS，VCS由Virtual Bridge组成，通过树状拓扑结构连接到所挂载的CXL设备上。
![](https://pic1.imgdb.cn/item/6862212e58cb8da5c87ed931.png)

CXL2.0支持**两种池化形式，分别是SLD(Single Logical Devices)和MLD(Multiple Logical Devices)**
![](https://pic1.imgdb.cn/item/6862213d58cb8da5c87ed9d8.png)

FM（Fabric Manager）动态配置主机VCS中虚拟桥跟特定设备之间的连接。在实践中，FM 可以是在主机上运行的软件、 或 CXL 交换机中的固件，或者是一颗在arm芯片。。FM可以控制每个VCS跟哪些设备进行绑定/解绑，因此也就完成了资源池化(通过MLD特性，将资源细粒度按需分配给需要的VCS；对SLD的重新绑定完成资源的重新分配)的功能。

对于MLD，允许将单个 CXL.mem(CXL2.0)设备的资源划分为**最多 16 个逻辑设备**，这些逻辑设备可以同时分配给不同的主机。CXL2.0池化功能兼容CXL1.0/1.1，但是CXL1.0/1.1设备只能进行SLD形式池化。

### QoS
由于交换、MLD 以及可能的不同类型的内存（可能不基于 DRAM）资源在单个交换机中可能变得过载，从而导致 QoS 问题。例如，一个设备的低性能可能导致使用交换机的所有代理的拥塞。为了缓解这个问题，CXL 2.0 在 CXL.mem 响应消息中引入了 DevLoad 字段，以通知主机其访问的设备中观察到的负载。主机应使用此负载信息来减少向该设备发送 CXL 请求的速率。