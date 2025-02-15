---
date: 2024-12-02
title: Transport Layer
---


# Transport Layer

>* 理解Transport Layer的工作原理
>   * 多路复用/解复用：**多个端口 ---[通过`socket`]-> 一个IP实体**
>   * 可靠数据传输：**Reliable Data Transfer, 不可靠IP --[增加那些机制]-> 可靠的TCP传输**
>   * 流量控制：**点对点间的控制---收发双发速率不匹配的问题该如何解决**
>   * 拥塞控制：**来自路径上的拥塞---原因/表现/检测/对策**
>* 实例和各种链路层技术的实现

## 1. Overview and Transport Layer Services

* Something form IP layer can be developed: Reliable, Security, Multiplex...
* But Something can't: Delay, Bandwidth...

## 2. Multiplexing and Demultiplexing
> 这里的复用指的是多个应用层协议使用一个传输层传数据，解复用是指一个传输层把不同的数据正确的交付给不同应用

* `Port` was introduced to distinguish each `Process` on the `host`
* TCP socket
    * tetrabasic-set: $Socket = (IP_{src},\, PORT_{src},\, IP_{dst},\, PORT_{dst},\, \mathcal{PID})$
    * Source: **App-Layer Message + Socket(port) -> TCP segment + Socket(IP) -> Net-Layer**
    * Destination: **After got packet, use socket( tetrabasic set ) to find its socket and PID**
* UDP socket
    * dibasic-set: $Socket = (IP_{src}, PORT_{src}, \textit{PID})$
    * Source: **Message + Socket + *DST_IP + DST_Port* -> UDP Datagrem ...**
    * Destination: **相同的IP和Port就会到同一个socket**
        * **!**:*区别TCP,对于TCP而言，即使都发往目标主机的同一个port,但是如果这个TCP连接的源端口或者源IP不同，那么就不是同一个TCP Socket*
* Socket is just like a handler(`fd`)


## 3. Connectionless Transport: UDP
> User Datagram Protocol [RFC 768]

*  **UDP = IP + Multiplexing and Demultiplexing**
    * 尽力而为的服务: 丢失, 乱序, 出错
    * 无连接：UDP发送端和接收端之间没有握手, 每个UDP Datagram 都被独立地处理
    * UDP被用于：流媒体（丢失不敏感，对于速度有要求）或者事务性应用，一次往返搞定的: DNS, SNMP
    * 在UDP上实现可靠传输
        * 在**应用层**增加可靠性
        * 应用特定的差错恢复
    * **feature -- why UDP?**
        * No Connection (no delay)
        * **Low Expense**: payload / (head + payload) is higher than TCP's.
        * **Fast**: no congestion control and flow control (应用传输速率约等于主机的网络速率)
    
* **Head = 8 bytes**
    * $Source Port = 16 bits$
    * $Destination Port = 16 bits$
    * $Length = 16 bits$: include **head**
    * $Check Sum = 16 bits$: check if udp datagram goes wrong, **drop** it if wrong.

* UDP check sum (EDC: Error Detecting Code, D: Data+Head)
    * 残存错误： D 和 EDC 都错了但恰好符合关系
    * 保护范围： 一些头部+伪头部(IP)+数据(Message/Payload)
    * 进位回滚： C 加到和的有效位的末尾，然后取反码
        * **目标端：`(D+EDC == 16位全1)?1:0`**


## 4. Principles of Reliable Data Transfer
> rdt在应用层、传输层和数据链路层都很重要
> 信道的不可靠特点决定了rdt的复杂性
> **不出错+不重复/不失序+不丢失**

1. 只考虑单向数据传输：但是控制信息是双向流动的（**反馈的概念**）
2. 双向的数据传输问题实际上是2个单向数据传输问题的综合
3. 使用FSM来表述发送方和接收方
    1. rdt1.0: *显然略*
    2. rdt2.0: 会出错+不丢失
        * **停止等待协议**---添加反馈机制：ack, nak
        * **进一步的**，ack/nak 可能出错
            * rdt2.1: 此时添加序号, 仅需(1, 0)
                * 若未听清，一律重发Packet0，只有正确听清ack是才发packet1
                * 一位代表分组序号就可以了 (1, 0)
                * 一次只发送一个未确认的分组
                * 只在接收到对应序号的packet才返回ack,否则一律nak
            * **rdt2.2**: 功能同2.1, 但只包含ack, 不含nak (ack 需要编号)
                * 不谢谢1(nak) = 谢谢0 -> **对当前分组的<u>反向确认</u>可以使用<u>前面分组的正向确认</u>来替代**
                * 顺便解决了**失序**的问题
    3. rdt3.0: 会出错+会丢失
        * 引入**超时重传机制** (rtt+delta)
        * 超时定时器的时间计算方式!!!_RELOCATE_ENTRY_
    4. rdt4.0: 提升利用率 -> 一次发一个pdu对于信道的利用率而言太低了
        * `pipeline`协议：一次发送多个 `pdu`，按理说rtt越大，一次同时发送的pde越多
        * 分为 `Go Back N` 和 `Select Repite` , 通用的协议 --- `Sliding Window`协议
            * **不同index的packet和ack都有一个自己的定时器**
            * Sliding Window
                * **Sending Window(发送窗口, 已发送但未确认的分组区域) <= 发送缓冲区**
                * **Receiving Windows(接收窗口, 只有落在接收缓冲区内的分组可以接收) = 接收缓冲区**
                * $SW = 1, RW = 1$: Stop and Wait: *rdt2.0-3.0*
                * $SW > 1, RW = 1$: GBN (pipeline): 只能**顺序**接收，顺序到来的最高分组的确认(累积确认)，只记录当前窗口最低位的timer，更新时重置timer
                * $SW > 1, RW > 1$: SR (pipeline): 可以**乱序**接收，每接收一个就要确认一个(单独确认，非累积确认，收到n不代表收到了<n的)
        * n bit编号：
            * GBN: $2^n - 1$
            * SR: $2^{n-1}$

## 5. Connection-Oriented Transport: TCP

>   一对一（不提供一到多，多到一，多到多...）
>
>   可靠的、按顺序的**字节流**
>
>   *   不出错、不重复、不丢失、不失序
>   *   **不提供报文的界限**
>
>   管道化、流水线的服务
>
>   *   根据MSS分段，每个MSS前加上head，形成TCP Segment -> 正好封装在一个以太网帧内（不存在分片问题）
>
>    面向连接

### 5.1 Segment Structure

*   根据MSS分段，每个MSS前加上head，形成TCP Segment -> 正好封装在一个以太网帧内（不存在分片问题）
*   序号：**以字节为单位的确认号**---分段后，每个Segment头字节在整个Message中的 $offset+x$
    *   初始不一定为0，握手时商量好一个初始序号 $x$
        *   放置老的段对新的连接产生影响
    *   $x, x+mss, x+2*mss, ..., x+n*mss$
*   确认号：**以字节为单位的序号**---和之前的理论相比**ack的含义相差1**
    *   **ack=555意味着接收方受到554及之前的所有packet，要求对方从555开始发送**
    *   累积确认，期望从另一方收到的下一个字节的序号
*   首部长度，可选项
*   **设置TCP超时定时器 ! ! ! **
    *   比RTT要长，但长多少？   自适应：平均值+4倍方差：
    *   $EstimatedRTT = (1-\alpha)\cdot EstimatedRTT + \alpha\cdot SampleRTT$
        *   指数加权移动平均
        *   过去样本的影响呈指数规律下降
        *   $\alpha$ 推荐值 $0.125$
    *   $DevRTT = (1-\beta)\cdot DevRTT + \beta\cdot|SampleRTT - EstimatedRTT|$
        *   $\beta$ 推荐值 $0.25$
        *   EstimatedRTT 变化大（方差大） -> 较大的安全边界时间
        *   当前的采样值离平均值的移动平均偏差值
    *   $TimeoutInterval = EstimatedRTT + 4\cdot DevRTT$

### 5.2 Reliable Data Transfer

*   pipeline协议：是GBN和SR的混合体
    *   累积确认（**字节编号，ack=555意味着接收方受到554及之前的所有packet，要求对方从555开始发送**）（*like GBN*）
    *   单个重传定时器，每个滑动窗口**最老的segment**那个有关（*like GBN*）
        *   超时时只重传**最老**的那个segment（*like SR*）
        *   重传：超时 or **连续的3个冗余确认**（4个确认，这个叫**快速重传**）
    *   是否可以接受乱序的，没有规范如何处理乱序到达

### 5.3 Flow Control

* 接收缓冲区
  * 发送方TCP实体写，上层应用进程读
  * 通过头部中的**`receive window`**发给对方**剩余缓冲区的大小**
* 捎带技术(Piggybacking)
  * 在R返回给S的data中**包含**S给R数据传送的确认信号
  * 反馈

### 5.4 Connection Management

* 在正式数据交换之前，发送方和接收方握手建立通讯信关系：
  * **同意建立连接**：每一方都知道对方愿意连接建立
  * **同步连接参数**：segment的起始序号（sequence number），ack的起始序号
* TCP 3次握手：解决**半连接**和**接收旧数据**的问题
  * `SYNbit = 1, Seq# = x`
  * `ACKbit = 1, ACK# = y`
  * `ACKbit = 1, ACK# = y+1`，此时通常和数据传输在一起
* 初始序号随机选取，能够大概率解决**接收旧数据**的问题
* TCP 连接释放：两个 **半连接**，每个方向单独拆除
  * `C -> S`
    * `FINbit = 1, SEQ# = x`
    * `ACKbit = 1, ACK# = x + 1`
  *  `S -> C`
    * `FINbit = 1, SEQ# = y`
    * `ACKbit = 1, ACK# = y + 1`
  * **最后一段是不可靠的，对称释放并不完美**（两军问题）
    * `TIMED_WAIT`
    * time waited for 2*max segment lifetime

## 6. Implementation of Congestion Control

>  非正式定义：太多数据传输超过了网络的处理能力---延迟高、丢包率高
>
> 原因/表现/检测/对策(atm, tcp)

1. **原因/表现**

    * 网络状态良好时，$\lambda_{in} \approx \lambda_{out}$
    * 网络状态变差时，$\lambda_{in} >> \lambda_{out}$
      * 丢失：重传
      * 超时：重传
      * **加剧网络的变坏**
    * 网络出现死锁
    * 当分组丢失时，其上游传输能力会被全部浪费，$\lambda_{in} >> \lambda_{out}\approx 0$


2. **检测**：不发生拥塞的前提下，尽可能提高发送速率
  * 网络向端系统**反馈**：**网络辅助信息的拥塞控制**
  * 端到端的自己判断：端到端根据超时信息、冗余ack...
3. **对策**：不发生拥塞的前提下，尽可能提高发送速率
  * **ATM网络：反馈来自网络内部提供的信息，以信元为单位**
    * ABR (available bit rate)：弹性服务
      * 轻载时：可以轻微大于承诺带宽
      * 拥塞时：要小于等于承诺带宽
    * RM（资源管理）信元
      * 由发送端发送，在数据信元中间隔插入
      * RM信元中的一些bit被**网络中的交换机**置位（NI bit, CI bit）
      * **网络中的交换机**在RM信元中的一些段写入其本身提供多少带宽，这样到了源主机就能知道网络的最小带宽
  * TCP网络：[见 #7. TCP Congestion Control](#7. TCP Congestion Control)


## 7. TCP Congestion Control

> 不同于ATM来自网络中的信息反馈，TCP把复杂性留在了网络边缘（传输层及以上，网络核心提供最小的服务集）。**路由器不向主机有关拥塞的反馈信息**，由端系统自己判断是否发生拥塞。

### 7.1 如何感知拥塞

> 轻微拥塞、拥塞
>
> 超时、有关某个段的3个冗余ACK

1. 拥塞和轻度拥塞
   * 超时：拥塞
     * 原因一：网络拥塞（概率大）
     * 原因二：未通过校验，被丢弃（丢弃发生在各个Layer上，概率小）
   * 有关某个段的3个冗余ACK：轻度拥塞
     * 快速重传
2. 如何控制发送方向网络中注入的速率：
   * $rate \approx \cfrac{Conges\_Win}{RTT} \,\text{byte/sec}$
   * **维护一个拥塞窗口的值$Conges\_Win$，是动态的，是感知网络拥塞程度的函数**
     * 超时或者3个重复ACK，$Conges\_Win \downarrow$
       * **超时重传时**：$Conges\_Win$降为1MSS，进入SS阶段然后，再成倍增加$Conges\_Win/2$（每个RTT），从而进如CA阶段
       * **3个冗余ACK**：$Conges\_Win$降为$Conges\_Win/2$，进入CA阶段
     * 否则（正常收到ACK，没有发生以上的情况）：$Conges\_Win\uparrow$
       * **SS：慢启动阶段——加倍阶段（每个RTT乘2）**
       * **CA：拥塞避免阶段——线性增加（每个RTT增加1个MSS）**
   * $Send\_Win = \min{Conges\_Win, Recv\_win}$：要同时满足拥塞控制和流量控制

### 7.2 控制策略$w/2 - w$

> 在拥塞发生时如何动作降低速率（拥塞、轻微拥塞不同）
>
> 在拥塞解决时如何动作提升速率

1. 慢启动(SS)
   * 刚建立连接时为1
   * 每收到一个ack，加1个MSS：指数型增加
2. AIMD：线性增，乘性减
   * 锯齿状
3. 超时事件后的保守策略

### 7.3 TCP吞吐量与公平性

$$
T = \cfrac{\frac{w}{2}+w}{2\cdot RTT}= \frac{3}{4} \frac{W}{RTT}
$$

* 慢启动阶段可以忽略不计
* ​



