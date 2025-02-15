---
date: 2024-12-01
title: Network Layer
---


# Network Layer

> 路由表是两个平面的交叉点

1. Control Plane
    * **Network-wide logic**
        * 全局的逻辑，决定数据包的转发路径
        * 决定数据包如何在网络中移动（**以网络为单位**）
    * 2 个控制平面的方法：
        * 传统的路由算法
        * SDN（Software Defined Networking）

2. Data Plane
    * **Local, per-router function**：
        * 本地，每个路由器的功能
        * 决定每个到达的数据包应该往哪个输出端口转发
    * **Forwarding**
        * Traditional: 基于目标地址 + 转发表
        * SDN: 基于多个字段 + 流表 (forward, block, flood, modify...)

# Network Layer 1: Data Plane

## 1. Introduction
1. 网络服务模型
2. 转发和路由（四种组合：传统 or SDN）（转发：data plane，路由：control plane）
3. 路由器工作原理
4. 通用转发
5. TCP/IP 实例

### 1.1 Functions of Network Layer
1. Forwarding: move packets from router's input to appropriate router output (**Localized**)
    * Data plane: local, per-router function

2. Routing: determine route taken by packets from source to destination (**Global**)
    * Control plane: network-wide logic


### 1.2 Data Plane and Control Plane
* **Data Plane**：本地，每个路由器的功能
    * 决定每个到达的数据包应该往哪个输出端口转发
    * **Forwarding**
* **Control Plane**：网络范围的逻辑

### 1.3 Traditional and SDN
1. 传统路由算法
    * 逻辑集中在路由器内部 （控制平面和数据平面**紧耦合**）
    * 路由算法决定路由表，路由表决定数据包的转发

2. SDN
    * 一个不同的（通常是远程的）控制器与本地控制代理（CAs）交互
    * 服务器、远程的控制器、网络操作系统 --><南向接口>--> 分组交换设备CAs
        * 集中式的控制
        * 远程的控制
        * *分组交换机（并非分组、并非交换机）*
        * （*CAs上报状态以供远程控制器计算流表*）
        ![SDN](https://raw.githubusercontent.com/rouge3877/ImageHosting/main/SDN.png)
    > 南向接口：上北下南，北向接口（与上层应用交互）和南向接口（与下层设备交互）


### 1.4 网络服务模型

1. 对于单个数据报的服务
* 可靠传送
* 延迟保证

2. 对于数据报**流**的服务
* 保序：保序数据报传送
* 延迟差：分组之间的延迟差
* 带宽：保证流的最小带宽

3. *连接建立* （有的网络采取***有连接***的服务，并非<u>面向</u>*连接（仅体现在端设备上）*，而是包含了路径上所有的路由器）

![网络服务模型](https://raw.githubusercontent.com/rouge3877/ImageHosting/main/ServModel.png)

## 2. Router Architecture

传统方式：路由器内部的数据平面和控制平面紧耦合
* 路由：运行路由选择算法/协议（RIP, OSPF, BGP...）-**生成路由表**
* 转发：从输入到输出链路交换数据报-**根据路由表进行转发**

![Router's big picture](https://raw.githubusercontent.com/rouge3877/ImageHosting/main/Router.png)


### 2.1 Input Port

![Input Port](https://raw.githubusercontent.com/rouge3877/ImageHosting/main/InputPort.png)

1. Line Terminal: bit级的接收（链路层物理信号转化为0101的bit信号）
2. Link Layer：判断哪里是帧头、哪里是帧尾；CRC；判断目标MAC是否匹配；取出数据部分交给网络层实体-在链路当中排队
3. 分布式交换：根据数据包头部的信息如：目的地址，在输入端口内存中的转发表中查找合适的输出端口（匹配+行动）
    * **基于目标的转发**：仅仅依赖于IP数据报的目标IP地址（传统方法）
    * **通用转发**：基于头部字段的任意集合进行转发

4. **输入端口缓存 (Queue)**：当交换机构的速率小于输入端口的汇聚速率时 --> 在输入端口肯呢个要排队
    * **<u>排队延迟以及由于输入缓存溢出造成丢失</u>**
    * **Head-of-the-Line (HOL) Blocking**: 排在队头的数据报阻止了队列中其他数据报向前移动


### 2.2 Switch Fabric
交换速率：大于N倍的输入端口速率（N为输入端口数）

![Switch Fabric](https://raw.githubusercontent.com/rouge3877/ImageHosting/main/SwitchFabric.png)

#### 2.2.1 Memory Switch：通过内存交换
第一代路由器（Cisco）：采用通用计算机，使用软件的方式进行转发

#### 2.2.2 Bus Switch：通过总线交换（Cisco 2500）
* 数据报通过共享总线
* **总线竞争**：交换速度受限于总线的带宽

#### 2.2.3 Crossbar Switch：通过交叉开关交换（Cisco 12000）
* 同时并发转发多个分组，克服了总线交换的瓶颈
* 互联网络的交换矩阵

### 2.3 Output Port

![Output Port](https://raw.githubusercontent.com/rouge3877/ImageHosting/main/OutputPort.png)

* 输出端口队列：当数据包从交换机构的到达速度比传输速率快，就需要输出端口**缓存**
    * 假设交换速率是输出速率的N倍
    * RFC 3439（拇指规则）
* 输出调度：选择哪个数据报发送到输出端口（调度：选择下一个要通过的数据报）
    * FIFO：FIFO调度策略下，数据报按照到达的顺序进行发送，不考虑数据报的优先级。同时Blocking策略下，如果输出端口的队列已满，那么新到达的数据报将被丢弃。
    * Priority：根据优先级进行调度，优先级高的数据报先发送，当且仅当优先级高的队列为空时，才会发送优先级低的数据报。
    * Round Robin：轮询调度策略下，数据报按照轮询的方式进行发送，每个队列发送一个数据报，然后再发送下一个队列的数据报。
    * Weighted Fair Queueing：加权公平队列调度策略下，每个队列都有一个权重，数据报按照权重进行发送，权重高的队列发送的数据报多，权重低的队列发送的数据报少。

## 3. IP: Internet Protocol

![IP Layer](https://raw.githubusercontent.com/rouge3877/ImageHosting/main/IP.png)
* **IP**：网络层的协议，提供了一种将数据包从源主机传输到目的主机的机制
    * 路由协议：RIP, OSPF, BGP
    * IP协议：IP地址、数据报格式、路由选择、分片、ICMP、ARP、DHCP、NAT、IPv6
    * ICMP协议：错误报文、路由器信令

### 3.1 IP Datagram Format

头部有20字节的固定长度，但是还会有一些OPTION字段：头部到底有多长？
* 头部以4个字节为单位
* 头部中有一个字段：IHL（IP Header Length），指示头部的长度（单位：4字节）
* 头部总长 - 20字节固定长度 = 选项字段的长度

![IP Datagram Format](https://raw.githubusercontent.com/rouge3877/ImageHosting/main/IPFormat.png)

#### 3.1.1 IP Datagram Structure

1. **Header**：20字节固定长度 + Options

| 名称                | 长度   | 用途       | 描述（括号中的内容）                                                                 |
|---------------------|--------|------------|--------------------------------------------------------------------------------------|
| **Version**         | 4位    | IP协议版本号 | IPv4：`0100`                                                                         |
| **IHL**             | 4位    | IP头部长度  | **<u>单位：4字节</u>**，因为IP头部长度是4字节的倍数，最小值是5-20字节                |
| **Type of Service** | 8位    | 服务类型    | 指示载荷类型，但是现在基本上已经不用了。*现在的QoS是通过IP头部的DSCP字段来实现的*   |
| **Total Length**    | 16位   | 数据报总长度 | **<u>单位：1字节</u>**                                                               |
| **Identification**  | 16位   | 标识符      | 用于分片                                                                             |
| **Flags**           | 3位    | 标志位      | 用于分片                                                                            |
| **Fragment Offset** | 13位   | 分段偏移    | 用于分片，**标识分片的位置，<u>8字节为单位</u>**                                                                             |
| **Time to Live**    | 8位    | 生存时间    | 每过一个路由器-1，为0时丢弃                                                          |
| **Protocol/Upper Layer** | 8位 | 协议       | 上层用户数据报的协议类型                                                             |
| **Header Checksum** | 16位   | 头部校验和  | 检查头部是否损坏                                                                     |
| **Source Address**  | 32位   | 源IP地址    |                                                                                      |
| **Destination Address** | 32位 | 目的IP地址 |                                                                                      |
| **Options**         | 可选项 |            |   E.g. 时戳，有路由器记录，指定所经过的路由器的列表                                             |
| **Padding**         | 填充   |            |                                                                                      |

2. **Data**：IP数据报的数据部分（载荷）

#### 3.1.2 Fragmentation and Reassembly

网络层相当于汇聚了所有的链路层，因此网络层需要处理链路层的MTU问题。网络下层的MTU（Maximum Transmission Unit）：链路层的最大传输单元
* 以太网：1500字节
* PPP：576字节
* 802.11：2000字节
* ...

**分片**：当数据报的长度大于链路层的MTU时，需要分片
* 例：链路层MTU=1500字节，而PDU = 20字节header + 3980字节body
    * 第一片：20字节header + 1480字节body
    * 第二片：20字节header + 1480字节body
    * 第三片：20字节header + 1020字节body
* 上面的操作就需要字段：`Identification`、`Flags`、`Fragment Offset`
    * `Identification`：标识符，用于标识一个数据报的所有分片 - **划分后的数据报就好像不同的数据报，所以需要一个标识符**
    * `Flags`：标志位，用于标识是否还有分片（1：还有分片，0：最后一个分片）
    * `Fragment Offset`：分段偏移，用于标识分片的位置 - **标识分片的位置，<u>8字节为单位</u>**
* **重组**：将分片的数据报重组成原始的数据报
    * 重组只在**目的主机**上进行：路由器不会重组数据报
    * 当目的主机接收到**一个分片后**，启动一个定时器，超时后丢弃所有分片


### 3.2 IP Address (IPv4)

IP地址：32为表示，对主机或者路由器的**接口**编址（主机和网络接口的那个连接点的标识，Identifier for the interface of a host or a router to the network）
* 接口：主机/路由器和物理链路的连接处
    * 路由器通常有多个接口
    * 主机也有可能有多个接口（多网卡）
* **IP地址和每一个接口关联，换句话说<u>一个IP地址和一个接口相关联</u>**
* *在一个IP子网内部，一跳可达（见后文）*

#### 3.2.1 Subnet
* **（纯）子网**：将一个大的IP地址空间划分为多个小的IP地址空间
    * 前缀一样（prefix）：子网的IP地址前缀相同(*有时候，只要前缀相同也算是子网（对外而言），并非纯子网。**路由聚集***)
    * （纯）子网内部，分组的发送和接收不需要路由器（可能需要交换机，但是在IP的层面一跳可达）

* 例子：下图有 **<u>6个子网</u>** -- 路由器之间的连接也是纯子网（点到点的连接）
    ![Subnet](https://raw.githubusercontent.com/rouge3877/ImageHosting/main/Subnet.png)

#### 3.2.2 IP Address Classes
路由时，是**以子网为单位**的，而不是每个IP地址为单位的。并且在路由的过程中还存在**路由聚集**，即将多个子网聚集到一个更大的子网中。

1. Class A: `[1'b0 + 7'b<network>] + [24'b<host>]`
    * **单播地址**
    * 126 networks: $2^7 - 2 = 126$, **全为0和全为1的网络号不可用**
    * 16,777,214 hosts: $2^{24} - 2 = 16,777,214$, **全为0表示网络号，全为1表示广播地址**
2. Class B: `[2'b10 + 14'b<network>] + [16'b<host>]`
    * **单播地址**
    * 16,384 networks: $2^{14} - 2 = 16,384$
    * 65,534 hosts: $2^{16} - 2 = 65,534$
3. Class C: `[3'b110 + 21'b<network>] + [8'b<host>]`
    * **单播地址**
    * 2,097,152 networks: $2^{21} - 2 = 2,097,152$
    * 254 hosts: $2^8 - 2 = 254$
4. Class D: `1110 + 28'b<multicast group>`
    * **Multicast**：组播地址 - 组播：发送给组播组内的所有主机
    * 一般**广播**只是局域网范围内的，而**组播**可以是全球范围内的
    * *单播、广播、组播、任播*
5. Class E: `1111 + 28'b<reserved>`
    * 预留地址：reserved for future

一些特殊的IP地址：
* 子网部分：全为0表示网络号（本网络）
* 主机部分：全为0表示本主机
* 主机部分：全为1表示广播地址（这个网络的所有主机）
* ![Special IP Address](https://raw.githubusercontent.com/rouge3877/ImageHosting/main/SpecialIP.png)
    * `255.255.255.255`: 广播地址（Broadcast Address）- 用于向同一网络中的所有主机发送数据报
    * `0.0.0.0`: **This host**
    * `<Network>.255...255`: Broadcast on a **distant** network
    * `0...0.<Host>`: **A host** on **this** network
    * `127.XXX.XXX.XXX`: 本地回环地址（Loopback Address）- 用于测试本机的TCP/IP协议栈（网络层以上）是否正常工作


* **内部（专用）IP地址**： 地址空间的一部分以供专用地址使用，永远不会被分配给公共互联网，不会与公用地址重复，
**只在局部网络中有意义，区分不同的设备。-><u>路由器不对目标地址是专用地址的分组进行转发</u>**
    * Class A: `10.0.0.0` - `10.255.255.255` | MASK = `255.0.0.0` | 1个网络，24位($2^{24}$个)主机
    * Class B: `172.16.0.0` - `172.31.255.255` | MASK = `255.255.0.0` | 16个网络，16位($2^{16}$个)主机
    * Class C: `192.168.0.0` - `192.168.255.255` | MASK = `255.255.255.0` | 256个网络，8位($2^{8}$个)主机

#### 3.2.3 CIDR (Classless Inter-Domain Routing)
* 子网部分可以在任意的位置
* 地址格式：`a.b.c.d/x`，其中`x`表示网络前缀的长度
* 子网掩码：IP地址在路由时是以**网络**为单位进行的，所以只需要知道网络号和子网掩码就可以了
    * **网络号：一个IP地址，子网部分为一个值而<u>主机部分全零</u>，表示一个网络号**

#### 3.2.4 转发表和转发算法（Forwarding Table and Forwarding Algorithm）

|Destination Subnet Num|Mask|Next Hop|Interface|
|----------------------|----|--------|---------|
|202.38.73.0 | 255.255.255.192| IPx | Lan1|
|202.38.64.0 | 255.255.255.192| IPy | Lan2|
|...|...|...|...|
|Default| - | IPz | Lan0|


* 转发表：**<u>目的地址</u>** 和 **<u>下一跳</u>** 的映射
    * **下一跳**：下一个路由器的IP地址
    * **接口**：下一个路由器的接口
* 如果`Des_IP_Addr && Mask_i == Destination_Subnet_Num_i`，则转发到对应的`Next Hop`/    `Interface`
* 如果都没有找到，则转发到`Default`的`Next Hop`/`Interface`
    * `Des_IP_Addr` -> `Destination_Subnet_Num_i` -> `Next Hop` -> `Des MAC` -> `Interface`
    * *`MASK` 不在IP报文中，而是在转发表中的表项*
        * *不会存在同时有两个匹配的情况吗？比如：*
            * `Des_IP_Addr <= 192.168.0.23`
            * `MASK_1 <= 255.255.255.0`
            * `MASK_2 <= 255.255.0.0`
            * `Destination_Subnet_Num_1 = 192.168.0.0`
            * `Destination_Subnet_Num_2 = 192.168.0.0`
        
#### 3.2.5 如何获得一个IP地址 和 DHCP
* 系统管理员将地址配置在一个文件中 (IP address | MASK | Default Gateway | Local Nameserver|)
    * Wintel: `C:\windows\winipcfg`
    * Unix/Linux: `/etc/rc.config`
* DHCP (Dynamic Host Configuration Protocol)：允许主机在加入网络时，动态的从服务器那里获得IP地址
    * 可以更新对主机在用IP地址的租用期-租期快到了
    * 重新启动时，允许重新使用以前用过的IP地址
    * 支持移动用户加入到该网络（短期在网）
    * UDP
* DHCP工作概况 (两次Request是因为一个子网内可能有多个DHCP服务器，或者有多个主机在申请IP地址（第一次是广播的）)
    * **广播:** 主机上线时广播`DHCP Discover`报文 (`dst ip add = 255.255.255.255`) [可选]（*src ip addr是什么？ - 事实上，这里主机由于还没有获得IP，所以只能采用`0.0.0.0`作为源地址表示自己*）
    * **广播:** DHCP 服务器用`DHCP offer`提供报文响应 [可选]（*dst ip addr是什么？ - 事实上，由于这里主机还没有确认/注册自己的IP地址，所以DHCP服务器只能通过广播的形式发送这一条消息*）
    * **单播:** 主机向DHCP服务器发送`DHCP Request`报文，请求IP地址
    * **单播:** DHCP服务器向主机发送`DHCP ACK`报文，确认IP地址
    * ![DHCP client-server scenario](https://raw.githubusercontent.com/rouge3877/ImageHosting/main/DHCP.png)

#### 3.2.6 如何获得一个网络的子网部分
一开始，一个大的机构会获得一个大的IP地址空间：20位网络号+12位主机号

机构的各个部门，将机构原始的12位主机号的高3位作为子网号，低9位作为主机号： [20位网络号+3位子网号]+9位主机号

进一步，各个部门的子部门，将部门的9位主机号的高3位作为子网号，低6位作为主机号： [20位网络号+6位子网号]+6位主机号

那么这个大机构，是如何获得其一开始的20位网络号的呢？更上级的权威机构 -> **ICANN**（Internet Corporation for Assigned Names and Numbers）
* 分配IP地址空间
* 管理DNS
* 分配域名，解决冲突

### 3.3 层次编址：路由聚集（Hierarchical Addressing: Route Aggregation）
* **路由聚集**：将多个子网聚集到一个更大的子网中
* **路由通告**：见上文，节[3.2.6](#326-如何获得一个网络的子网部分)的例子。
    * 每个部门，如`200.23.16.0/23`,`200.23.18.0/23`等。`200.23.16.0/23`子网的路由器向（总）机构的路由器通告：
    凡是子网前缀是`200.23.16.0`的网络，都将分组发给我 (IPx，IPx是子网路由器和总机构路由器的接口的IP地址)，**将我作为下一跳**
    * 那么机构在向上级机构通告时，无需将所有的23位的子网都通告，只需要通告`200.23.16.0/20`即可，这样就实现了**路由聚集**
    * ![Route Aggregation 1](https://raw.githubusercontent.com/rouge3877/ImageHosting/main/RouteAggregation1.png)
* “***吹牛***” —— **最长前缀匹配**：路由器在转发数据报时，会选择最长前缀匹配的路由表项（回答节[3.2.4](#324-转发表和转发算法forwarding-table-and-forwarding-algorithm)的问题）
    * 8个里面有7个，可以采用大概的前缀，如图[Route Aggregation 2](#Route-Aggregation-2)中的例子
    * ![Route Aggregation 2](https://raw.githubusercontent.com/rouge3877/ImageHosting/main/RouteAggregation2.png)

### 3.4 NAT: Network Address Translation (*何不食肉糜*)
* 回顾节[3.2.2](#322-ip-address-classes)的内部（专用）IP地址
* ![NAT](https://raw.githubusercontent.com/rouge3877/ImageHosting/main/NAT.png)
* **动机 - 本地网络只有一个有效IP地址**：
    * 不需要从ISP分配一块地址，可用一个IP地址用于所有的（局域网）设备——**省钱省IP**
    * 可以在局域网改变设备的地址情况下而无需通知外界
    * 可以改变ISP（地址变化）而不需要改变内部的设备地址
    * 局域网内部的设备没有明确的地址，对外是不可见的——**安全**
* **实现 - NAT路由器**：
    * NAT路由器中要有一个转换表，记录内部地址和外部地址的映射：`<内部地址，端口> -> <外部地址，端口>` (*端口很多，可以用来区分不同的设备*)
    * **外出数据包**：替换**源地址`src ip`和端口号`src port`**，将内部地址和端口号映射为外部地址和端口号
    * **进入数据包**：替换**目的地址`dst ip`和端口号`dst port`**，将外部地址和端口号映射为内部地址和端口号
    * ![NAT Router](https://raw.githubusercontent.com/rouge3877/ImageHosting/main/NATRouter.png)
* **争议**

* **NAT穿越**
    * 静态配置NAT：转发进来的对服务器特定端口连接请求，如`123.76.29.7:25000` 总是转发到内部的`10.0.0.1:80`
    * Universal Plug and Play (UPnP) / Internet Gateway Device Protocol (IGD)：允许设备自动配置NAT
        * 获知网络的公共IP地址
        * 列举存在的端口映射
        * 增删端口映射（在租用时间内）
    * 中继：STUN、TURN、ICE（先和中继进行连接）

### 3.5 IPv6 

* ref: [RFC 2460](https://tools.ietf.org/html/rfc2460), [RFC 4291](https://tools.ietf.org/html/rfc4291), [Blog](https://cshihong.github.io/2018/01/29/IPv6%E5%9F%BA%E7%A1%80/)

* 初始动机：32bit的IP地址空间不够用
    * 此外，还有一些其他动机：头部格式改变帮助加速处理和转发
        * `TTL-1` -> 头部变化，每次都要计算CheckSum
        * 分片
    * 头部格式改变帮助QoS (Quality of Service)

* 头部长度 **40 字节**
* 传输过程中不允许**分片**：如果分片太大，就会被丢弃，然后通过ICMP协议通知源主机，让源主机重新发送一个小的数据报
* IP地址长度由原先的32bit变为128bit

#### 3.5.1 IPv6 Format

![IPv6 Format](https://raw.githubusercontent.com/rouge3877/ImageHosting/main/IPv6Format.png)

* **Header**: 40 字节固定长度

| 名称 | 长度 | 用途 | 描述 |
|---------------------|--------|------------|---------------------------------------------------------------------|
| **Version**         | 4位    | IP协议版本号 | IPv6：`0110` |
| **Priority**        | 8位    | 服务类型    | 与IPv4的`Type of Service`类似，但是更加细致，如：流量类别、优先级等 |
| **Flow Label**      | 20位   | 流标签      | 用于区分不同的流，如：视频流、音频流等，试图让网络对同一个流的数据包进行同样的处理 (flow没有被明确定义) |
| **Payload Length**  | 16位   | 载荷长度    | 与IPv4的`Total Length`类似，但是不包括头部长度 |
| **Next Header**     | 8位    | 下一个头部  | 与IPv4的`Protocol`类似，但是更加细致，如：TCP、UDP、ICMP等 |
| **Hop Limit**       | 8位    | 跳数限制    | 与IPv4的`Time to Live`类似，但是不是每过一个路由器-1，而是每过一个路由器-1，为0时丢弃 |
| **Source Address**  | 128位  | 源IP地址    | |
| **Destination Address** | 128位 | 目的IP地址 | |


* **Data**：数据部分

#### 3.5.2 IPv6's `Next Header`

`Next header`：自解释, TLV

该字段定义紧跟在IPv6报头后面的第一个扩展报头（如果存在）的类型，或者上层协议数据单元中的协议类型。

![IPv6 Next Header](https://raw.githubusercontent.com/rouge3877/ImageHosting/main/IPv6NextHeader.png)

#### 3.5.3 IPv6 & IPv4
* 变化：
    * `Checksum`被移除，因为其降低了每一跳的处理速度
    * `option`允许，但在头部之外，被`next header`字段标识
    * `ICMPv6`：ICMP的新版本
        * 附加了报文类型，如"Packet Too Big"
        * 多播组管理功能
* ***<u>如何过渡??????????????????????????????????????????????????????????????????????</u>***
    * 孤岛，边缘有双栈协议，**隧道**，海洋变少，岛屿变大...

## 4. 通用转发 和 SDN

传统的路由器：
1. 每台设备上即实现控制功能、又实现数据平面
2. 控制功能分布式实现
3. 路由表-粘连

传统的缺陷：
1. 协议已经定好了，不容易更改
2. 越来越多的中间盒：路由器、交换机、防火墙、负载均衡器、NAT、代理、缓存、加密、解密、QoS、监控、管理、审计、...
3. **垂直集成**：硬件、软件、协议、应用、...
4. 集成->固化->僵化->不利于创新的生态

### 4.1 SDN

逻辑上集中的控制平面：一个不同的（通常是远程）控制器和CA交互，控制器决定分组转发的逻辑（可编程），CA所在设备执行逻辑
* 通用 **flow-based**, 基于流的匹配+行动（OpenFlow）（南向接口）
* 控制平面和数据平面的分离
* 控制平面功能在数据交换设备之外实现
* 可编程控制应用，在控制器之上以**网络应用**形式实现各种网络功能

![SDN split](https://raw.githubusercontent.com/rouge3877/ImageHosting/main/SDNsplit.png)

1. 网络设备数据平面和控制平面分离
2. 控制平面-分组交换机
    * 将路由表、交换机和目前大多数网络设备的功能进一步**抽象**成：按照流表（由控制平面设置的控制逻辑）給你选哪个PDU（帧、分组）的动作（包括转发、丢弃、拷贝、泛洪、阻塞）
    * **统一化**设备功能：SDN交换机（分组交换机，*此分组交换机非彼分组交换机*），执行控制逻辑
3. 控制平面-控制器+网络应用
    * 分离、集中
    * 计算和下发控制逻辑：流表（Flow Table）
4. **水平集成**，**集中式实现**控制逻辑，**分布式执行**控制逻辑

![SDN Example](https://raw.githubusercontent.com/rouge3877/ImageHosting/main/SDNExample.png)

### 4.2 通用转发和SDN

每个路由器包含一个**流表**（配逻辑上集中的控制器计算和分发）
![SDN Big Picture](https://raw.githubusercontent.com/rouge3877/ImageHosting/main/SDNBP.png)

* **流表**：每个路由器包含一个流表，用于匹配数据报的头部字段，然后执行相应的动作
    * **匹配字段**：源IP、目的IP、源端口、目的端口、协议类型、TTL、...
    * **动作**：转发、丢弃、拷贝、泛洪、阻塞、...

* **控制器**：计算和下发流表
    * **控制器应用**：路由选择、防火墙、负载均衡、NAT、代理、缓存、加密、解密、QoS、监控、管理、审计、...

### 4.3 OpenFlow 抽象

* **match+action：统一化各种网络设备提供的功能**

    * 路由器：
        * match：最长前缀匹配
        * action：通过一条链路转发
    * 交换机：
        * match：目标MAC地址
        * action：转发或者泛洪
    * 防火墙：
        * match：源IP、目的IP、端口
        * action：丢弃或者转发
    * NAT：
        * match：源IP、目的IP、端口
        * action：重写地址和端口
    * ![OpenFlow Abstract](https://raw.githubusercontent.com/rouge3877/ImageHosting/main/OpenFlowAbstract.png)

* **OpenFlow 数据平面抽象**
    * 流：由分组（帧）头部字段所定义
    * 通用转发：简单的分组处理规则
        * Pattern：将分组头部字段与流表中的规则进行匹配，然后执行相应的动作
        * Action：对于匹配上的分组，可以是丢弃、转发、修改、将匹配的分组转发给控制器
        * Priority：几个模式匹配了，优先采用哪个，消除歧义
        * Counter：对字节计数，对packet计数
    * ![OpenFlow](https://raw.githubusercontent.com/rouge3877/ImageHosting/main/OpenFlow.png)



----
----
----

# Network Layer 2: Control Plane
* 传统路由选择算法
* SDN控制器
* ICMP
* 以及他们在互联网上的实例：OSPF、BGP、RIP、OpenFlow、...

## 5. 路由选择算法

> 路由协议的**目标**：确定数据报从源主机传输到目的主机的“较好”的路径（确切来说，**子网**到**子网**的路由） 
>    * 路径：路由器的序列，分组将会沿着该序列从源主机到达最后的目标主机
>    * “较好”：最小“代价”、“最快的”、“最不拥塞”
>    * 路由 - 一个 “Top 10” 的问题

1. **路由**：按照某种指标（传输延迟、所经过的站点数目等）找到一条从源节点到目标节点的较好路径

2. **路由器-路由器**之间的最好路径：主机到主机之间的最好路径（子网到子网的最好路径）

3. **路由选择算法**：网络层软件的一部分，完成路由功能

4. 网络的**图抽象**：<!-- ![Graph Abstract](https://raw.githubusercontent.com/rouge3877/ImageHosting/main/GraphAbstract.png) -->
    * 图：G = (N, E)
    * 节点N：路由器、主机
    * 边E：路由器-路由器之间的链路或者网络
    * 边的权重：链路的代价（延迟、带宽、费用等）可能总为1，或者延迟、带宽、费用的倒数等
    * 最终目的：**汇集<u>树</u> (sink Tree)**，从一个根节点到所有其他节点的最短路径。根节点是网络的出口，其他节点是网络的内部节点

5. 路由算法分类：
    * **全局**：知道网络的全部拓扑结构和链路代价 (*Link State算法*)
    * **分布式**：只知道相邻节点的拓扑结构和链路代价 (*Distance Vector算法*)
    * **静态**：路由表不会改变 -> 非自适应算法 (non-adaptive algorithm)，不能适应网络的拓扑和通信量的变化，路由表是事先计算好的
    * **动态**：路由表会根据网络的变化而改变 -> 自适应算法 (adaptive algorithm)，能够适应网络的拓扑和通信量的变化，路由表是根据网络的变化而计算的


### 5.1 Link State (dijkstra | global | dynamic)

> **获得**网络拓扑和链路代价信息 -> 使用**Dijkstra**算法计算最短路径 -> *使用此路由表*

#### 5.1.1 **获得**网络拓扑和链路代价信息：
1. 发现相邻节点，获知对方网络地址
    1. **Hello协议**：路由器启动时，向相邻路由器发送 `Hello` 消息，相邻路由器回应
    2. 其他路由器收到 `Hello` 消息后，回送应答，在应答分组中，告知自己的名字（全局唯一的名字）
    3. 在LAN中，通过广播 `Hello` 消息，获得其他路由器的信息，可以认为引入一个人工节点
2. 测量到相邻节点的链路**代价**
    1. **实测法**：发送一个分区要求对方立即回应
    2. 回送一个ECHO分组
    3. 通过测量时间可以估算出延迟信息
3. 组装一个**LS分组**，描述它到相邻节点的代价情况
    1. **发送者**的名字
    2. **序号，年龄**。其中序号是为了防止循环，年龄是为了防止旧的信息被使用
    3. 列表：给出它的**相邻节点**，以及到这些节点的**代价**
4. 将分组通过扩散的方法发到所有其他路由器（***洪泛***）
    * *洪泛：一个节点向所有相邻节点发送信息，相邻节点再向所有相邻节点发送信息，直到所有节点都收到信息*，会有很多重复的信息，所以...
    1. **顺序号**：用于控制无穷的扩散，每个路由器都记录（源路由器、**顺序号**），发现重复或者老的信息就不再传播
        * 具体问题1：循环使用问题
        * 具体问题2：路由器崩溃后序号从0开始
        * 具体问题3：序号出现错误
    2. 解决问题的办法：**年龄字段 (age)**
        * 生成一个分组时，年龄字段不为0
        * 每一个时间段，AGE字段减1
        * AGE字段为0时，分组被丢弃
    3. 关于`Link State Packet`的数据结构：
        * Source：从哪个节点发出的LS分组
        * Sequence Number，Age：序号和年龄
        * Send Flags：发送标记，必须向指定的那些相邻站点转发LS分组
        * ACK Flags：应答标记，本站点必须向哪些相邻站点发送应答
        * DATA：来自source的LS分组
        * 例：节点B的数据结构
            * ![LS Data Structure](https://raw.githubusercontent.com/rouge3877/ImageHosting/main/LSDataStructure.png)
    

#### 5.1.2 使用**Dijkstra**算法计算最短路径（***这才是路由算法***）
1. 每个节点独立算出来到其他节点（路由器=网络）的最短路径
2. 迭代算法：第k步能够知道本节点到k个其他节点的最短路径
3. 例（每个节点计算自己的）：![Dijkstra](https://raw.githubusercontent.com/rouge3877/ImageHosting/main/Dijkstra.png)

关于**Dijkstra**算法的一些讨论
* **复杂度**：$O(n^2)$（n(n-1)/2次比较） -> **堆**：$O(n\log n)$
* **可能的震荡**：当链路代价发生变化时，可能会引起路由表的变化，然后又引起其他路由表的变化，最终又回到原来的路由表
    * **解决**：引入**计数器**，当链路代价发生变化时，计数器+1，当计数器达到一个阈值时，才会更新路由表

### 5.2 Distance Vector (Bellman-Ford | distributed | dynamic)

![Distance Vector](https://raw.githubusercontent.com/rouge3877/ImageHosting/main/FullBF.png)

动态路由选择算法的基本思想：（*向量：距离+方向 :)*）
* **每个节点**维护一个**距离向量**，包含到其他所有节点的距离估计
* **每个节点**周期性地将自己的**距离向量**发送给**相邻节点**
* **每个节点**根据**相邻节点**的**距离向量**更新自己的**距离向量**

路由表如下 (***当然，实际中Destination应该是网络号+子网掩码，此外还有一些其他信息***)
| Destination | Next Hop | Cost |
|-------------|----------|------|
| A           | B        | 14   |
| B           | B        | 0    |
| C           | B        | 10   |
|...|...|...|

#### 5.2.1 **获得**网络拓扑和链路代价信息：

* 代价及相邻节点间代价的获得
    * 跳数 (hops)，延迟 (delay)，队列长度
    * 相邻节点间代价的获得：**实测**
<!-- * 路由信息的更新：下面展示的是一个节点A的更新一个目标节点Z的距离向量的过程（*实际上可以同时更新多个目标节点的距离向量*）
    * 根据实测，得到本节点A到相邻节点B、C、D...的代价
    * 本节点A将自己的距离向量发送给相邻节点B、C、D...
    * 各个相邻节点生成它们到目标节点Z的代价
    * 本节点A根据相邻节点的代价更新自己到目标节点Z的距离向量
    * 找到一个最小的代价，和**相应的下一个节点C**，到达节点Z经过节点C的代价最小，代价为A-C-Z的代价
    * ![Distance Vector](https://raw.githubusercontent.com/rouge3877/ImageHosting/main/DistanceVector.png) -->


#### 5.2.2 使用**Bellman-Ford**算法计算最短路径（DP | ***这才是路由算法***）

1. Bellman-Ford算法
* 设：$d_x(y) := 从x到y的最小路径代价$
* 那么：$d_x(y) = \min\{c(x,v) + d_v(y)\}$
* 其中$v$是$x$的一个相邻节点，$c(x,v)$是$x$到邻居$v$的代价，$d_v(y)$是$v$到目标$y$的代价，而$\min$是对$x$所有的邻居$v$进行的

2. DV: 距离矢量算法（**分布式**，**异步**，**迭代**）![Distance Vector feature](https://raw.githubusercontent.com/rouge3877/ImageHosting/main/DistanceVectorFeature.png)
    * $D_x(y)$：节点$x$到节点$y$代价最小值的**估计** (x节点维护距离矢量$\textbf{D}_x = [D_x(y), \forall y\in N]$)
        * 知道所有邻居节点v的代价 $c(x,v)$
        * 收到并维护一个它邻居的距离矢量集 $\textbf{D}_v $

    1. 每个节点都将自己的距离矢量估计值传送给邻居，定时或者DV有变化时，**让对方去算**
    2. 当x从邻居v收到一个新的距离矢量时，更新自己的距离矢量
        * 采用B-F equation: $D_x(y) = \min\{c(x,v) + D_v(y)\}, \forall y\in N$
    3. $D_x(y)$**估计值**最终收敛于实际的最小代价$d_x(y)$


3. DV的无穷计算问题
    * DV的特点：
        * **<u>好消息传的快，坏消息传的慢</u>**
    * 好消息的传播以每个交换周期前进一个路由器的速度进行
        * **好消息**：某个路由器**接入**或者有**更短的路径**
        * 举例：![DV Good News](https://raw.githubusercontent.com/rouge3877/ImageHosting/main/DVGoodNews.png)
    * 坏消息的传播速度非常慢（**无穷计算问题**）
        * **坏消息**：某个路由器**断开**或者有**更长的路径**
        * 举例：![DV Bad News](https://raw.githubusercontent.com/rouge3877/ImageHosting/main/DVBadNews.png)
    * 避免 -> **水平分裂 (Split Horizon)** 算法 *(谦虚：关公面前不卖大刀)*
        * **不向邻居节点发送**它从邻居节点**接收到的**距离矢量
        * **不向邻居节点发送**它**自己**到邻居节点的距禙矢量
        * **不向邻居节点发送**它**自己**到邻居节点的距禙矢量中包含的**其他邻居节点**的距离矢量

### 5.3 LS vs. DV

1. 消息复杂度（DV胜出）
    * **LS**：有n个节点，E条链路，发送报文的复杂度是$O(nE)$
        * ***局部的路由信息，全局传播***
    * **DV**：只和**邻居**交换信息
        * ***全局的路由信息，局部传播***

2. 收敛时间（LS胜出）
    * **LS**：$O(n^2)$
    * **DV**：收敛时间不稳定，可能出现**计数到无穷**的问题

3. 健壮性（LS胜出）：路由器故障会发生什么（*有个路由器发疯*）
    * 节点会通告（泛洪）不正确的链路代价（*我到任意节点的代价是0*）
        * **LS**：不影响不经过该节点的链路
        * **DV**：会影响整个网络


> * **LS**：
>    * **全局**：每个节点都知道网络的全部拓扑结构和链路代价
>    * **Dijkstra**：计算最短路径
>    * **洪泛**：将自己的链路代价信息传播给所有其他节点
>    * **收敛**：每个节点都知道网络的最短路径
>    * **开销**：洪泛的开销大，但是计算开销小
>    * **稳定**：链路代价变化时，洪泛的开销大，但是计算开销小
>    * **循环**：不会出现循环
>    * **计算**：$O(n^2)$
>
> * **DV**：
>    * **分布式**：每个节点只知道相邻节点的拓扑结构和链路代价
>    * **Bellman-Ford**：计算最短路径
>    * **向量**：将自己的链路代价信息传播给相邻节点
>    * **收敛**：每个节点都知道网络的最短路径
>    * **开销**：洪泛的开销小，但是计算开销大
>    * **稳定**：链路代价变化时，洪泛的开销小，但是计算开销大
>    * **循环**：可能出现循环
>    * **计算**：$O(n^2)$

## 6. Intra-AS Routing in the Internet

### 6.1 RIP (Routing Information Protocol)
* 采用 **Distance Vector** 算法
    * **跳数**：链路代价（*最大16跳*）
    * 每隔30秒向所有邻居发送距离矢量（或者被请求时）
    * 每个通告包含到所有目的地的距离矢量（目标网络+跳数）（*最多包含AS内部的25个目标子网，相比于朴素的算法，协议还规定了传送子网的描述*）

* 链路失效和恢复
    * 如果**180s没有收到邻居的通告**，邻居（或者链路）被认为失效
    * 形成新的路由通告（活）邻居，（活）邻居因此发出新的通告（如果路由变化的话）
    * 链路失效会**快速的**在整个网络中传播
    * 使用**毒性逆转（Poison Reverse）**来避免**Ping-Pong**回路（不可达距离：16跳）

* **RIP**以进程的方式实现：`route-d (daemon)`
    * 报文通过**UDP**传输
    * 网络层的协议使用了传输层的服务，一应用层实体的方式实现：![RIP-Process](https://raw.githubusercontent.com/rouge3877/ImageHosting/main/RIPProcess.png)


### 6.2 OSPF (Open Shortest Path First)
* 使用 **Link State** 算法
    * LS Packet在一个AS内部传播（***洪泛*，在IP数据报上直接传送OSPF报文，而非*像RIP那样使用UDP***）
    * 全局网络拓扑，链路代价在AS的每一个节点中都保持
    * 路由计算采用Dijkstra算法
* OSPF通告信息中携带：每一个邻居路由器一个表项
* *IS-IS：OSPF的另一种实现*

* OSPF的**高级特性**（RIP所没有的）
    * **安全**：所有的OSPF报文都是经过认证的
    * 允许有**多个相等的路径**（*RIP只有一个最短路径*）
    * **支持**不同的服务质量（QoS）需求：有**多重代价矩阵**
        * 例如：最小的延迟、最大的带宽、最小的费用
    * 对单播和多播的集成支持：Multicast OSPF (MOSPF) 使用相同的拓扑矩阵库，就像在OSPF中一样
    * 在大型网络中支持**层次化**OSPF：![OSPF Hierarchical](https://raw.githubusercontent.com/rouge3877/ImageHosting/main/OSPFHierarchical.png)
        * **区域**：AS的一个子集，每个区域有一个区域内部的路由器，区域内部的路由器只知道区域内部的拓扑结构和链路代价
        * **骨干区域**：连接所有区域的区域
        * **边界路由器**：连接不同区域的路由器
        * **区域内部**：只知道区域内部的拓扑结构和链路代价
        * **区域间**：只知道区域间的拓扑结构和链路代价

## 7. Inter-AS Routing: BGP

> 在前面的章节 (节[6](#6-intra-as-routing-in-the-internet)) 中，讨论的是***一个平面***的路由问题 (*LS, DV, 所有路由器都要知道其他所有路由器（子网）如何走*)，也就是所有的路由器的 **<u>地位一样</u>**。然而：
> * 如果在**规模巨大**的的范围内，使用**一个平面**解决路由问题，有上亿个节点，上亿条链路，这是不现实的
> * 在**管理和安全**上，不同的网络所有者希望按照**自己的方式管理网络**；希望对外**隐藏**自己网络的细节；但同时还需要和其他网络互联

### 7.1 What's AS

为了解决上述问题，采用了两个层面的解决方案 **<u>(层次路由, Hierarchical Routing)</u>**：引入了**自治系统**（AS）的概念
1. **层次路由**：将互联网分成一个个AS（路由器区域）
    * 某个区域内的路由器集合称为：**自治系统 (Autonomous System, AS)**
    * 一个AS用AS Number (ASN)来**唯一**的标识
    * 一个ISP (Internet Service Provider)可能包含一个或多个AS
2. 路由变成了：**2个层次**的路由选择问题
    * **AS内部路由**：在同一个AS内部运行相同的路由协议 (Intra-AS Routing Protocol), 内部网关协议
        * **RIP, OSPF**
        * **网关路由器**： AS边缘路由器，可以连接到其他AS
    * **AS间路由**：在AS之间运行AS间路由协议 (Inter-AS Routing Protocol)
        * **BGP (Border Gateway Protocol)**
        * 解决AS之间的路由问题，完成AS之间的互联互通

层次路由的优点：
* 内部而言：AS内部的路由选择问题变得简单；AS内部的路由选择协议可以根据AS的特点进行设计；对外部隐藏了AS内部的细节
* 外部而言：只需要一个或少数节点表示一个AS

### 7.2 BGP Intro (Border Gateway Protocol)
> 自治区域间路由协议“事实上”的标准，将互联网各个AS黏在一起的胶水

BGP提供给每个AS以下方法（*有趣的是，这些协议通过TCP连接来传输*）：
* eBGP (external BGP)：AS之间的路由选择，收集AS内部的路由信息，向其他AS传递
* iBGP (internal BGP)：AS内部的路由选择，将eBGP传递的路由信息传递给AS内部的路由器
* 允许子网向互联网那个其他网络通告：***“我在这里”***
* 基于**距离矢量算法**
    * 但是不止告诉代价，还包括到达各个目标路径的详细路径（AS序号的列表），以提高收敛速度，避免环路）
* 如图[BGP](#BGP)所示，其中每一条虚线表示一个BGP协议的(***TCP**连接的*)通信关系
    * ![BGP](https://raw.githubusercontent.com/rouge3877/ImageHosting/main/BGP.png)

### 7.3 BGP Message (***<u>等待补充！！！ <----------> Wait for Supplement!!!</u>***)

* BGP会话：2个BGP路由器（“peers”）一开机就建立一个半永久的TCP连接。在这个**半永久的TCP连接**上，他们交换BGP消息

* BGP消息：4种类型
    * **OPEN**：打开一个BGP会话
    * **UPDATE**：传送路由信息
    * **KEEPALIVE**：保持连接
    * **NOTIFICATION**：报告错误

* 路径的属性 & BGP路由（BGP是一个“路径矢量”协议）
    * 当通告一个子网前缀时，通告包括 BGP 属性
        - prefix + attributes = “route”
    * 2个重要的属性:
        - **AS-PATH**: 前缀的通告所经过的AS列表: AS 67 AS 17
        - 检测环路；多路径选择
        - 在向其它AS转发时，需要将自己的AS号加在路径上
        - **NEXT-HOP**: 从当前AS到下一跳AS有多个链路，在NEXT-HOP属性中，告诉对方通过那个链路转发.
        - 其它属性：路由偏好指标，如何被插入的属性
    * 基于策略的路由：
        - 当一个网关路由器接收到了一个路由通告, 使用输入策略来接受或过滤（accept/decline.）
        - 过滤原因例1：不想经过某个AS，转发某些前缀的分组
        - 过滤原因例2：已经有了一条往某前缀的偏好路径
        - 策略也决定了是否向它别的邻居通告收到的这个路由信息
    * 路由器可能获得一个网络前缀的多个路径，路由器必须进行路径的选择，路由选择可以基于（一个前缀对应着多种路径，采用消除规则直到留下一条路径）
        1. 本地偏好值属性: 偏好策略决定
        2. 最短AS-PATH ：AS的跳数
        3. 最近的NEXT-HOP路由器:热土豆路由
        4. 附加的判据：使用BGP标示的属性


* 热土豆路由(***Hot Potato Routing***)
    * 当一个AS收到一个数据报，它会选择一个“最近”的边界路由器，将数据报传递给它
    * 选择的标准：最短的AS路径
    * 例：![Hot Potato Routing](https://raw.githubusercontent.com/rouge3877/ImageHosting/main/HotPotatoRouting.png)


### 7.4 BGP Example

![BGP Example1](https://raw.githubusercontent.com/rouge3877/ImageHosting/main/BGPExample1.png)
![BGP Example2](https://raw.githubusercontent.com/rouge3877/ImageHosting/main/BGPExample2.png)
![BGP Example3](https://raw.githubusercontent.com/rouge3877/ImageHosting/main/BGPExample3.png)

### 7.5 为什么Inter-AS 和 Intra-AS 路由选择不同

* **Intra-AS**：关注性能 - 一个管理者，因此无需考虑策略
* **Inter-AS**：关注策略 - AS之间的路由选择不仅仅是性能问题，还是政策问题、商业问题、政治问题

## 8. SDN: Control Plane

![per-router](https://raw.githubusercontent.com/rouge3877/ImageHosting/main/per-router.png)
![SDN-method](https://raw.githubusercontent.com/rouge3877/ImageHosting/main/SDN-method.png)
![SDN-architecture](https://raw.githubusercontent.com/rouge3877/ImageHosting/main/SDN-architecture.png)


见节[4](#4-通用转发-和-SDN)
|网络控制应用的界面层| 网络控制应用|控制的大脑：采用下层提供的服务(SDN控制器提供的API)，实现网络功能|
|---|---|---|
|北向接口|Northbound Interface||
|**网络范围的状态管理层**|**SDN 控制器(网络OS)**|**维护网络状态信息，逻辑上集中**|
|南向接口|Southbound Interface||
|**通信层**|**数据平面分组交换机**|**采用硬件实现通用转发功能**|


![SDN-example1](https://raw.githubusercontent.com/rouge3877/ImageHosting/main/SDN-example1.png)
![SDN-example2](https://raw.githubusercontent.com/rouge3877/ImageHosting/main/SDN-example2.png)

## 9. ICMP (Internet Control Message Protocol)

## 10. 网络管理和SNMP

