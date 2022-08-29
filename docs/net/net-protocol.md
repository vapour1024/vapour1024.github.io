# 网络协议学习笔记

# ch01 为什么要学习网络协议

## 协议三要素

计算机语言作为程序员控制一台计算机工作的协议，具备了协议的三要素。

- 语法，就是这一段内容要符合一定的规则和格式。例如，括号要成对，结束要使用分号等。
  
- 语义，就是这一段内容要代表某种意义。例如数字减去数字是有意义的，数字减去文本一般来说就没有意义。
  
- 顺序，就是先干啥，后干啥。例如，可以先加上某个数值，然后再减去某个数值。
  

## 我们常用的网络协议有哪些？

以访问一个网站为例

1. 获取网站域名对应的IP地址，使用DNS/HTTPDNS
  
2. 浏览器开始打包请求，使用HTTP/HTTPS
  
3. TCP（面向连接，网站应用端口、浏览器端口）/UDP（无连接）
  
4. IP（客户端IP、服务端IP）
  
5. DHCP（IP及默认网关配置）/ARP（IP到MAC的映射）
  
6. 路由协议，使用OSPF和BGP
  
7. 进程间通信，使用RPC
  

| 层级  | 协议  |
| --- | --- |
| 应用层 | DHCP HTTP HTTPS RTMP P2P GTP RPC |
| 传输层 | UDP TCP SCTP |
| 网络层 | ICMP IP OSPF BGP IPSex GRE |
| 链路层 | ARP VLAN STP |

[全面理解DNS及HTTPDNS - 掘金 (juejin.cn)](https://juejin.cn/post/6844903987796246542)

---

# ch02 网络分层的真实含义是什么？

## 网络为什么要分层？

因为复杂的程序都要分层。

理解计算机网络中的概念，一个很好的角度是，想象网络包就是一段 Buffer，或者一块内存，是有格式的。

同时，想象自己是一个处理网络包的程序，而且这个程序可以跑在电脑上，可以跑在服务器上，可以跑在交换机上，也可以跑在路由器上。你想象自己有很多的网口，从某个口拿进一个网络包来，用自己的程序处理一下，再从另一个网口发送出去。

## 揭秘层与层之间的关系

那 TCP 在三次握手的时候，IP 层和 MAC 层在做什么呢？当然是 TCP 发送每一个消息，都会带着 IP 层和 MAC 层了。因为，TCP 每发送一个消息，IP 层和 MAC 层的所有机制都要运行一遍。而你只看到 TCP 三次握手了，其实，IP 层和 MAC 层为此也忙活好久了。

这里要记住一点：

只要是在网络上跑的包，都是完整的。可以有下层没上层，绝对不可能有上层没下层。

---

# ch03 ifconfig

## 查看 IP 地址

- `ifconfig` net-tools
  
- `ip addr` iproute2
  

```shell
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 08:00:27:92:a6:b9 brd ff:ff:ff:ff:ff:ff
    inet 192.168.56.188/24 brd 192.168.56.255 scope global noprefixroute enp0s3
       valid_lft forever preferred_lft forever
    inet6 fe80::8c4c:b644:d253:9fa8/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever
```

IPV4：32bit

IPV4：128bit

IP地址分类：ABCDE

## 无类型域间选路（CIDR）

这种方式打破了原来设计的几类地址的做法，将 32 位的 IP 地址一分为二，前面是网络号，后面是主机号。从哪里分呢？你如果注意观察的话可以看到，10.100.122.2/24，这个 IP 地址中有一个斜杠，斜杠后面有个数字 24。这种地址表示形式，就是 CIDR。后面 24 的意思是，32 位中，前 24 位是网络号，后 8 位是主机号。

伴随着 CIDR 存在的，一个是广播地址，10.100.122.255。如果发送这个地址，所有 10.100.122 网络里面的机器都可以收到。另一个是子网掩码，255.255.255.0。

将子网掩码和 IP 地址进行 AND 计算（按位计算 AND），就可得到网络号。

## 公有 IP 地址和私有 IP 地址

- 私有 IP 地址
  
  平时我们看到的数据中心里，办公室、家里或学校的 IP 地址，一般都是私有 IP 地址段。因为这些地址允许组织内部的 IT 人员自己管理、自己分配，而且可以重复。因此，你学校的某个私有 IP 地址段和我学校的可以是一样的。
  
- 公有 IP 地址。
  
  公有 IP 地址有个组织统一分配，你需要去买。如果你搭建一个网站，给你学校的人使用，让你们学校的 IT 人员给你一个 IP 地址就行。但是假如你要做一个类似网易 163 这样的网站，就需要有公有 IP 地址，这样全世界的人才能访问。
  

例如Wi-Fi 路由器的地址是 192.168.0.1，而 192.168.0.255 就是广播地址。一旦发送这个地址，整个 192.168.0 网络里面的所有机器都能收到。

在 IP 地址的后面有个 scope，global说明这张网卡是可以对外的，可以接收来自各个地方的包。host说明这张网卡仅仅可以供本机相互通信。

lo 全称是 loopback，又称环回接口，往往会被分配到 127.0.0.1 这个地址。这个地址用于本机通信，经过内核处理后直接返回，不会在任何网络中出现。

## MAC 地址

MAC 地址是一个网卡的物理地址，用十六进制，6 个 byte 表示。

MAC 地址号称全局唯一，不会有两个网卡有相同的 MAC 地址，而且网卡自生产出来，就带着这个地址。

一个网络包要从一个地方传到另一个地方，除了要有确定的地址，还需要有定位功能。而定位功能需要IP地址

MAC 地址更像是身份证，是一个唯一的标识

MAC 地址是有一定定位功能的，只不过局限在一个子网里面

## 网络设备的状态标识

`enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000`

- UP 表示网卡处于启动的状态
  
- BROADCAST 表示这个网卡有广播地址，可以发送广播包
  
- MULTICAST 表示网卡可以发送多播包
  
- LOWER_UP 表示物理连接正常（网线插着）
  
- MTU1500 表示以太网最大传输单元 MTU 为 1500
  
- qdisc pfifo_fast
  
  qdisc 全称是 queueing discipline，中文叫排队规则。内核如果需要通过某个网络接口发送数据包，它都需要按照为这个接口配置的 qdisc（排队规则）把数据包加入队列。
  
  - pfifo：不对进入的数据包做任何的处理，数据包采用先入先出的方式通过队列
    
  - pfifo_fast：它的队列包括三个波段（band）。在每个波段里面，使用先进先出规则。三个波段（band）的优先级也不相同。band 0 的优先级最高，band 2 的最低。如果 band 0 里面有数据包，系统就不会处理 band 1 里面的数据包。
    
    数据包是按照服务类型（Type of Service，TOS）被分配到三个波段（band）里面的。TOS 是 IP 头里面的一个字段，代表了当前的包是高优先级的，还是低优先级的。
    

## 小结

- IP 是地址，有定位功能
  
- MAC 是身份证，无定位功能
  
- CIDR 可以用来判断是不是本地人
  
- IP 分公有的 IP 和私有的 IP
  

> net-tools起源于BSD，自2001年起，Linux社区已经对其停止维护，而iproute2旨在取代net-tools，并提供了一些新功能。一些Linux发行版已经停止支持net-tools，只支持iproute2。
> net-tools通过procfs(/proc)和ioctl系统调用去访问和改变内核网络配置，而iproute2则通过netlink套接字接口与内核通讯。
> net-tools中工具的名字比较杂乱，而iproute2则相对整齐和直观，基本是ip命令加后面的子命令。
> 虽然取代意图很明显，但是这么多年过去了，net-tool依然还在被广泛使用

---

# ch04 DHCP与PXE：IP是怎么来的，又是怎么没的？

## 如何配置 IP 地址？

### 使用 net-tools

```shell
sudo ifconfig eth1 10.0.0.1/24
sudo ifconfig eth1 up
```

### 使用 iproute2：

```shell
sudo ip addr add 10.0.0.1/24 dev eth1
sudo ip link set up eth1
```

## 动态主机配置协议（DHCP）

只需要管理员配置一段共享的 IP 地址。每一台新接入的机器都通过 DHCP 协议，来这个共享的 IP 地址里申请，然后自动配置好。使用完了还回去，其他的机器也能用。

### 解析 DHCP 的工作方式

1. DHCP Discover
  
  新来的机器使用 IP 地址 0.0.0.0 发送了一个广播包，目的 IP 地址为 255.255.255.255。广播包封装了 UDP，UDP 封装了 BOOTP。其实 DHCP 是 BOOTP 的增强版，但是如果你去抓包的话，很可能看到的名称还是 BOOTP 协议。
  
2. DHCP offer
  
  DHCP server找到一个可用的 IP 地址，然后发起广播（包含分配的IP地址），DHCP Server 为此客户保留为它提供的 IP 地址，从而不会为其他 DHCP 客户分配此 IP 地址。
  
3. DHCP request
  
  DHCP server可能有多个，DHCP Offer也可能有多个，新来的机器会选择其中一个DHCP Offer，一般是最先到达的那个，并且会向网络发送一个 DHCP Request 广播数据包，包中包含客户端的 MAC 地址、接受的租约中的 IP 地址、提供此租约的 DHCP 服务器地址等，并告诉所有 DHCP Server 它将接受哪一台服务器提供的 IP 地址，告诉其他 DHCP 服务器，谢谢你们的接纳，并请求撤销它们提供的 IP 地址，以便提供给下一个 IP 租用请求者。
  
4. DHCP ACK
  
  当 DHCP Server 接收到客户机的 DHCP request 之后，会广播返回给客户机一个 DHCP ACK 消息包，表明已经接受客户机的选择，并将这一 IP 地址的合法租用信息和其他的配置信息都放入该广播包，发给客户机，欢迎它加入网络大家庭。
  

### IP 地址的收回和续租

DHCP分配的IP地址是有租期的。客户机会在租期过去 50% 的时候，直接向为其提供 IP 地址的 DHCP Server 发送 DHCP request 消息包。客户机接收到该服务器回应的 DHCP ACK 消息包，会根据包中所提供的新的租期以及其他已经更新的 TCP/IP 参数，更新自己的配置。这样，IP 租用更新就完成了。

## 预启动执行环境（PXE）

PXE 协议分为客户端和服务器端，由于还没有操作系统，只能先把客户端放在 BIOS 里面。当计算机启动时，BIOS 把 PXE 客户端调入内存里面，就可以连接到服务端做一些操作了。

首先，PXE 客户端自己也需要有个 IP 地址。因为 PXE 的客户端启动起来，就可以发送一个 DHCP 的请求，让 DHCP Server 给它分配一个地址。

然后，PXE 客户端还需要知道 PXE 服务器的信息，需要在DHCP server中配置 next-server，指向 PXE 服务器的地址，另外要配置初始启动文件 filename。

### 解析 PXE 的工作过程

接下来我们来详细看一下 PXE 的工作过程。

1. 启动 PXE 客户端。
  
2. 通过 DHCP 协议告诉 DHCP Server，请求租一个 IP 地址，同时还有 PXE 服务器的地址、启动文件 pxelinux.0。
  
3. 通过执行pxelinux.0，PXE 客户端知道要去 PXE 服务器下载这个文件后，就可以初始化机器。于是便开始下载，下载的时候使用的是 TFTP 协议。所以 PXE 服务器上，往往还需要有一个 TFTP 服务器。PXE 客户端向 TFTP 服务器请求下载这个文件，TFTP 服务器将这个文件传给它，PXE 客户端收到这个文件后，就开始执行这个文件。
  
4. 这个文件会指示 PXE 客户端，向 TFTP 服务器请求计算机的配置信息 pxelinux.cfg。TFTP 服务器会给 PXE 客户端一个配置文件，里面会说内核在哪里、initramfs 在哪里。
  
5. PXE 客户端会请求这些文件。最后，启动 Linux 内核。