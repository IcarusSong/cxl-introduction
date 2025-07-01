# CXL 交换技术概述 (7.1 Overview)

此部分提供了不同 CXL 交换器配置的架构概览。

## **1. 单一 VCS (Single Virtual CXL Switch)**
![](https://pic1.imgdb.cn/item/6862b6c558cb8da5c8805d32.png)
* **定义**: 由一个 CXL 上行端口 (USP) 和一个或多个下行端口 (DSPs) 组成。
* **核心规则**:
    * 必须有且仅有一个 USP。
    * 必须有一个或多个 DSPs。
    * DSPs 必须支持在 CXL 模式或 PCIe 模式下运行。
    * 下行交换端口必须能够支持 RCD (Retimer/CXL Device) 模式。
    * 对于单一 VCS，Fabric Manager (FM) 是可选的。


## **2. 多重 VCS (Multiple Virtual CXL Switch)**
![](https://pic1.imgdb.cn/item/6862b74c58cb8da5c8805d7b.png)
* **定义**: 由多个上行端口和每个 VCS 下的一个或多个下行端口组成。
* **核心规则**:
    * 必须拥有一个以上的 USP。
    * 每个 VCS 必须有一个或多个下行 vPPB。
    * 配置完成后，每个 USP 及其关联的下行 vPPB 作为一个单一 VCS Switch 运行，并遵循单一 VCS 的规则。
    * DSPs 必须支持在 CXL 模式或 PCIe 模式下运行。
    * DSPs 必须能够支持 RCD 模式。
    * 每个 DSP 必须绑定到一个 PPB 或 VPPB。
    * 每个 DSP 可以通过由 FM 协调的管理型热插拔流程，重新分配给不同的 VCS。
    * 对于需要绑定/解绑或支持 MLD 端口的多重 VCS，必须要有 FM。对于其他情况，FM 是可选的。
>RCD mode: 受限CXL设备模式（原CXL 1.1模式）。 该CXL工作模式存在诸多限制， 包括不支持热插拔，且仅支持68B Flit模式。
>VPPB:vPPB 是 虚拟PCI-to-PCI桥接器 (virtual PCI-to-PCI Bridge) 的缩写

## **3. 带 MLD 端口的多重 VCS (Multiple VCS with MLD Ports)**
![](https://pic1.imgdb.cn/item/6862b76158cb8da5c8805d7d.png)
* **定义**: 由多个上行端口和一个或多个支持 MLD (Multi-Logical Device) 的下行端口组合而成。
* **核心规则**:
    * 拥有一个以上的 USP。
    * 每个 VCS 有一个或多个下行 vPPB。
    * 一个支持 MLD 的 DSP 最多可以连接到 16 个 USP。
    * 配置完成后，每个 USP 及其关联的 vPPB 构成一个单一 VCS，并按其规则运行。
    * MLD 组件中的每个逻辑设备 (LD) 实例都可以通过 FM 协调的管理型热插拔流程，重新分配给不同的 VCS。
    * DSPs 必须支持在 CXL 模式或 PCIe 模式下运行。
    * DSPs 必须能够支持 RCD 模式。


## **vPPB 排序 (VPPB Ordering)**

**vPPB（Virtual PCI-to-PCI Bridge，虚拟PCI到PCI桥）**  
vPPB是CXL交换机中的一种虚拟化组件，用于模拟PCIe桥的功能。它在虚拟CXL交换机（VCS）中扮演关键角色，支持动态绑定和解绑物理端口或逻辑设备（LD）。vPPB的主要特点包括：  
1. **虚拟化功能**：允许物理端口或MLD（多逻辑设备）中的LD动态绑定到不同的虚拟层次结构（VCS）。  
2. **配置规则**：  
   - 在单VCS中，必须有一个上游端口（USP）和多个下游端口（DSP）。  
   - 在多VCS中，支持多个USP和动态重新分配DSP到不同的VCS。  
3. **MLD支持**：vPPB可绑定到MLD的特定LD，实现资源共享（如内存池化）。  

### vPPB排序

* VCS 内的 vPPB 排序遵循以下序列：首先是 USP vPPB，然后是 DSP vPPB，按设备号 (Device Number)、功能号 (Function Number) 的递增顺序排列。
* 这意味着先按设备号顺序排列所有功能号为 0 的 vPPB，然后是所有功能号为 1 的 vPPB，依此类推。
