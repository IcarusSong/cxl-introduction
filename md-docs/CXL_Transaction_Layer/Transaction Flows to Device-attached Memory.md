
# Transaction Flows to Device-attached Memory
## 1.Flows for Back-Invalidate Snoops on CXL.mem
### 1.1BISnp Blocking Example
![](https://pic1.imgdb.cn/item/686278b958cb8da5c87ffde4.png)
- 起因：主机向设备发送一个读请求 MemRd X 。
- 问题：设备端的窥探过滤器 (SF) 满了，无法为地址 X 分配新的跟踪条目 。
- 设备的操作：
  - 暂停：设备必须暂停处理 MemRd X 请求，因为它无法在 SF 中记录这个事务 。
  - 腾空间：为了给 X 腾出空间，设备选择一个“受害者”地址 Y，并向主机发起一个 BISnpInv Y（反向无效化窥探），命令主机放弃对 Y 的缓存 。
- 主机的响应：
  - 主机收到 BISnpInv Y，发现自己有一个修改过的 Y 的副本（M 状态）。
  - 主机必须先把这个脏数据通过 MemWr Y 写回给设备 。
  - 写回完成后，主机才能通过BIRspI 响应，告诉设备：“Y 的窥探处理完毕，我已经不再缓存它了” 。
- 最终解决：
  - 设备收到 BIRspI，意味着 BISnp Y 流程结束，SF 中 Y 的条目被成功释放。
  - 现在 SF 有了空间，可以为地址 X 分配条目。
  - 至此，最初被阻塞的MemRd X 请求终于可以被处理，设备向主机返回数据和完成信号 。


### 1.2 Conflict Handling

这是 CXL.mem 协议中一个非常精巧的设计，用于解决当主机和设备几乎同时对同一内存地址进行操作时可能发生的竞态条件 (Race Condition)。

#### 冲突处理的核心机制

首先，理解冲突处理的三个基本步骤：

1. **冲突的定义**：当一个由主机发往设备的 `M2S Req`（如读请求）和一个由设备发往主机的 `S2M BISnp`（如无效化窥探）针对同一缓存行地址，并且两者同时处于“活动”状态时，就发生了冲突。
2. **冲突的握手**：
   - 主机检测到这个冲突后，会向设备发送一个 `BIConflict` 消息。
   - 设备收到 `BIConflict` 后，必须回应一个 `BIConflictAck` 消息。
3. **判断的关键**：主机通过观察自己收到的两个关键消息——`Cmp`（对自己原始 `M2S Req` 的完成响应）和 `BIConflictAck`（设备对冲突的确认）——的先后到达顺序，来明确判断冲突的性质。这依赖于一条基本排序规则：设备必须保证 `BIConflictAck` 不会越过同一事务的 `Cmp` 消息。

## 早期冲突 (Early Conflict) 详解

“早期冲突”可以通俗地理解为：主机的请求“跑慢了”。
![](https://pic1.imgdb.cn/item/68627a3d58cb8da5c8800876.png)
- **场景描述**:
  1. 主机发出一个 `M2S Req` 请求到设备。
  2. 但这个请求因为某种原因（例如被设备的 `BISnp` 流程阻塞）尚未被设备处理。
  3. 与此同时，设备发出的 `S2M BISnp` 先到达了主机，主机检测到了冲突。

- **主机观察到的现象 (判断依据)**:
  主机将先收到设备返回的 `BIConflictAck` 消息，然后才收到（或者一直没收到）对自己 `M2S Req` 的 `Cmp` 完成响应。

- **处理流程与结论**:
  看到 `BIConflictAck` 先到，主机就得出了明确的结论：“我的那个 `M2S Req` 请求，设备那边根本就还没处理。”
  既然自己的请求没被处理，事情就变得简单了。主机可以放心地先处理设备的 `BISnp` 请求。例如，在图 3-22 中，主机在收到 `BIConflictAck` 后，可以安全地回应一个 `BIRspI`，告诉设备窥探已完成。
  在设备端，收到 `BIRspI` 后，它自己的 `BISnp` 流程结束，阻塞被解除。然后，它才能开始处理那个被“耽搁”了的主机的 `M2S Req` 请求，并最终返回 `Cmp-E` 和数据。

**早期冲突总结**：主机“提早”发现了冲突，此时它的原始请求尚未对系统状态造成任何影响。因此，主机可以优先满足设备的一致性请求，然后再让自己的请求重新排队等待处理。

## 晚期冲突 (Late Conflict) 详解

“晚期冲突”可以通俗地理解为：主机的请求“跑快了”。
![](https://pic1.imgdb.cn/item/68627c3c58cb8da5c88016aa.png)
- **场景描述**:
  1. 主机发出一个 `M2S Req` 请求到设备。
  2. 设备已经成功处理了这个请求，并且对应的 `Cmp` 完成响应已经在返回主机的路上了。
  3. 但就在 `Cmp` 还在传输时，设备发出的 `S2M BISnp` 到达了主机，主机检测到了冲突。

- **主机观察到的现象 (判断依据)**:
  主机将先收到对自己 `M2S Req` 的 `Cmp` 完成响应（例如 `Cmp-E`），之后才收到设备返回的 `BIConflictAck` 消息。

- **处理流程与结论**:
  看到 `Cmp-E` 先到，主机也得出了明确的结论：“我的请求已经被成功处理了，那个缓存行的所有权已经转移给了设备。”
  现在，主机必须基于这个已经改变了的现实来处理设备的 `BISnp` 请求。它不能再像早期冲突那样简单地回应 `BIRspI` 了。
  主机必须审视自己当前的状态。例如，在图 3-23 中，`Cmp-E` 到达意味着主机认为设备现在拥有了 `E` 状态。当主机要处理 `BISnpInv` 时，它必须在自己的内部一致性逻辑中执行一次状态变更（例如 `E->I`），然后才能向设备发送 `BIRspI`，以正确响应这个窥探。

**晚期冲突总结**：主机“很晚”才发现冲突，此时它的原始请求已经“获胜”并改变了系统状态。因此，主机必须承认这个既成事实，并将设备的 `BISnp` 作为一个全新的事件，在当前最新的系统状态上进行处理。

通过这套机制，无论主机和设备的请求如何“竞争”，CXL 协议都能通过明确的握手和严格的顺序观察，确保最终结果的一致性和正确性。



好的，我们来详细讲解“3.5.1.4 Block Back-Invalidate Snoops”这一小节。

### 1.3 块反向无效化窥探 (Block Back-Invalidate Snoops) 详细讲解

这一节介绍的是 `BISnp` 机制的一个重要**性能优化**功能。

#### 1. 核心目的：提升窥探效率

* **问题背景**：在某些场景下，设备可能需要一次性让主机放弃对一块连续内存区域的缓存（例如，设备的窥探过滤器需要驱逐一大片连续的条目）。如果为这个区域中的每一条缓存行（64字节）都单独发送一个 `BISnp` 消息，将会产生巨大的协议开销，效率低下。
* **解决方案**：**块窥探 (Block Snoops)** 允许设备通过**一条消息**，同时对 2 个或 4 个连续的、自然对齐的缓存行发起窥探。这极大地减少了消息数量，降低了链路开销。

#### 2. 实现机制：如何定义一个“块”

设备通过 `BISnp*Blk` 这一系列特殊的操作码（如 `BISnpInvBlk`）来发起块窥探。同时，它利用了 `Address` 字段中的低位（`Address[7:6]`）来特殊编码，以指明这个“块”的大小和范围：

* **`Address[7:6]` 编码 (如表 3-48 所示)**:
    * `01b`: 表示窥探一个 128B 的块（地址的低半部分）。
    * `10b`: 表示窥探一个 128B 的块（地址的高半部分）。
    * `11b`: 表示窥探一个完整的 256B 的块。

**重要前提**：块窥探的起始地址必须与块的大小自然对齐（例如，256B 的块窥探，其起始地址的低 8 位必须为 0）。

#### 3. 主机的两种响应方式：效率与灵活性的平衡

CXL 协议为的灵活性，允许主机在收到块窥探后，可以从以下两种方式中选择一种进行响应：

* **方式一：块响应 (Block Response) - 最高效** (参考图 3-24)
  ![](https://pic1.imgdb.cn/item/68627e5858cb8da5c8801fba.png)
    * **场景**: 主机能够一次性处理完整个块中所有缓存行的一致性。
    * **流程**:
        1.  设备发送一条 `BISnpInvBlk` 消息，目标是一个 256B 的块。
        2.  主机内部并行处理这个块中的 4 条缓存行（例如，对需要窥探的 Y0 和 Y2 行进行窥探，而 Y1 和 Y3 已知是无效的）。
        3.  当所有 4 条行的一致性都解决后，主机**只发送一条**统一的块响应消息，例如 `BIRspIBIk`。
    * **结果**: 这条单一的响应消息向设备确认了**整个 256B 块**在主机端都已变为无效状态。这种方式用最少的消息完成了操作，效率最高。

* **方式二：逐缓存行响应 (Cacheline Response) - 更灵活** (参考图 3-25)
  ![](https://pic1.imgdb.cn/item/68627e8a58cb8da5c8802060.png)
    * **场景**: 主机处理块内不同缓存行的速度不同，或者希望以流水线的方式尽快返回已处理完部分的结果。
    * **流程**:
        1.  设备发送一条 `BISnpDataBlk` 消息，目标是一个 256B 的块。
        2.  主机逐个处理块内的缓存行。每处理完一条，就立刻发送一个**单独的**响应。
        3.  **关键字段 `LowAddr`**: 为了让设备知道每个单独的响应对应的是块中的哪一部分，每个响应消息中都必须包含 `LowAddr` 字段（地址的低两位）。例如，`BIRspS` 带着 `LA=0` 表示这是对块内第 0 条线的响应；`BIRspI` 带着 `LA=1` 表示这是对第 1 条线的响应。这些响应的到达顺序可以是任意的。
    * **结果**: 这种方式虽然消息数量变多了，但提供了更高的灵活性，主机不必等待最慢的那条线处理完才能开始响应，有助于提升整体的流水线效率。

**总结**：“块反向无效化窥探”是 CXL.mem 协议为了**提升大规模一致性操作效率**而设计的一个高级功能。它通过将多个窥探请求捆绑为单条消息来减少协议开销，同时又赋予了主机根据自身情况选择“打包回复”或“分开发送”的灵活性，兼顾了效率与实现弹性。

### 1.4 推测性内存读取 (Speculative Memory Read) 
![](https://pic1.imgdb.cn/item/6862810258cb8da5c88027af.png)
#### 一、核心目的：为降低延迟而“抢跑”

**问题背景**：在一次常规的内存读取中，主机（Home Agent）必须首先完成内部的一致性解析（例如，窥探其他 CPU 核心的缓存），然后才能向 CXL 内存设备发起正式的读请求。这个等待一致性解析的过程会增加总的内存访问延迟。

**解决方案**：为了缩短这段延迟，CXL 引入了 `MemSpecRd`（推测性内存读取）命令。它允许主机在进行内部一致性解析的同时，就推测性地向设备发起一个“预读取”请求。这就像在赛跑中，让设备“抢跑”去内存中取数据，期望当主机“冲过终点线”（完成一致性解析）时，数据已经准备好了。

#### 二、“推测性读取”的工作流程

这个流程的核心在于“提示 + 合并”：

1. **主机发送“提示” (`MemSpecRd`)**：主机向设备发送一个 `MemSpecRd` 命令。这个命令像一个非正式的通知，告诉设备：“我很有可能马上就需要这个地址的数据了，你现在可以先去取了。”
2. **无完成信号**：`MemSpecRd` 是一个“发后不理”的指令，它不会收到任何完成消息。设备可以自行决定是否执行它，甚至可以随时丢弃它。
3. **设备预取数据**：如果设备决定响应这个“提示”，它会开始从其内部的物理内存（如 DRAM 颗粒）中读取数据。
4. **主机发送“正式订单” (`MemRd`)**：当主机完成内部的一致性解析后，它会发送一个正式的、必须被响应的读请求 `MemRd`。
5. **设备“合并”并快速响应**：设备收到正式的 `MemRd` 请求后，如果它之前已经根据 `MemSpecRd` 的提示预取了数据，它就可以将这个请求与已经准备好的数据进行“合并”。由于数据已在手边，设备可以立即通过 `MemData` 将数据返回给主机，从而节省了从物理内存取数的全部时间，显著降低了访问延迟。

#### 三、实现时的注意事项与建议（重要）

`MemSpecRd` 虽然能优化延迟，但如果使用不当，反而会降低性能。因此规范给出了明确的建议：

**处理冲突**：如果在处理一个 `MemSpecRd` 时，设备发现已经有另一个针对同一地址的内存访问正在进行中，那么推荐的做法是直接丢弃这个 `MemSpecRd` 命令，以避免内部处理逻辑变得过于复杂。

**高负载下会“帮倒忙”**：推测性读取会消耗 CXL 链路和设备内存控制器的额外带宽。在系统已经很繁忙（高负载）的情况下，这种额外的带宽消耗反而会加剧拥塞，导致整体性能下降。

**处理建议：低优先级 + 择机丢弃**：
- 设备应该将 `MemSpecRd` 视为低优先级命令。
- 当检测到链路或内存负载很高时（例如，通过 QoS 遥测返回的 `DevLoad` 字段检测到），设备应该主动丢弃 `MemSpecRd` 请求，以优先保障正常的、非推测性的内存访问。

**总结**：`MemSpecRd` 是 CXL 提供的一个精巧的延迟优化工具。它通过“抢跑”机制，用可能被浪费的带宽来换取时间。但它是一个“晴天优化”，只在系统负载较低时才能发挥正面作用，在系统繁忙时则应该被“降级”或“忽略”，以避免好心办坏事。



### 1.5 Flows to HDM-H in a Type 3 Device

#### Read from Host
在此流程中，仅返回一条数据消息。
![](https://pic1.imgdb.cn/item/6862209c58cb8da5c87ed38c.png)

#### Write from Host
与读取操作不同，写操作总是以S2M NDR Cmp消息完成。这一通用的写入流程下图：
![](https://pic1.imgdb.cn/item/686220aa58cb8da5c87ed3ef.png)