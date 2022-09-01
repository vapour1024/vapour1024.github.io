# etcd in action

---

# ch01 etcd的前世今生

他们希望在重启任意一节点的时候，用户的服务不会因此而宕机，导致无法提供服务，因此需要运行多个副本。但是多个副本之间如何协调，如何避免变更的时候所有副本不可用呢？

CoreOS 团队需要一个协调服务来存储服务配置信息、提供分布式锁

- 可用性角度：高可用。协调服务作为集群的控制面存储，它保存了各个服务的部署、运行信息。

- 数据一致性角度：提供读取“最新”数据的机制。既然协调服务必须具备高可用的目标，就必然不能存在单点故障（single point of failure），而多节点又引入了新的问题，即多个节点之间的数据一致性如何保障？

- 容量角度：低容量、仅存储关键元数据配置。协调服务保存的仅仅是服务、节点的配置信息（属于控制面配置），存储上不需要考虑数据分片。

- 功能：增删改查，监听数据变化的机制。协调服务保存了服务的状态信息，若服务有变更或异常，能快速推送变更事件给控制端。

- 运维复杂度：可维护性。在分布式系统中往往会遇到硬件 Bug、软件 Bug、人为操作错误导致节点宕机，以及新增、替换节点等运维场景，都需要对协调服务成员进行变更。若能提供 API 实现平滑地变更成员节点信息，就可以大大降低运维复杂度，减少运维成本，同时可避免因人工变更不规范可能导致的服务异常。

## etcd v1 和 v2 诞生

### etcd v1

共识算法：Raft

数据模型（Data Model）：基于目录的层次模式

API：REST API，提供了常用的 Get/Set/Delete/Watch 等 API，实现对 key-value 数据的查询、更新、删除、监听等操作。

存储引擎：简单内存树，含节点路径、值、孩子节点信息。这是一个典型的低容量设计，数据全放在内存，无需考虑数据分片，只能保存 key 的最新版本，简单易实现。

可维护性：Raft 算法提供了成员变更算法，可基于此实现成员在线、安全变更，同时此协调服务使用 Go 语言编写，无依赖，部署简单。

### etcd v2

v1版本实现了简单的 HTTP Get/Set/Delete/Watch API，但读数据一致性无法保证。v2 版本，支持通过指定 consistent 模式，从 Leader 读取数据，并将 Test And Set 机制修正为 CAS(Compare And Swap)，解决原子更新的问题。

## 为什么 Kubernetes 使用 etcd?

etcd 高可用、Watch 机制、CAS、TTL 等特性正是 Kubernetes 所需要的

当你使用 Kubernetes 声明式 API 部署服务的时候，Kubernetes 的控制器通过 etcd Watch 机制，会实时监听资源变化事件，对比实际状态与期望状态是否一致，并采取协调动作使其一致。Kubernetes 更新数据的时候，通过 CAS 机制保证并发场景下的原子更新，并通过对 key 设置 TTL 来存储 Event 事件，提升 Kubernetes 集群的可观测性，基于 TTL 特性，Event 事件 key 到期后可自动删除。

## etcd v3 诞生

### etcd v2的问题

- 功能问题
  
  - 不支持范围查询和分页查询，容易产生expensive request，进而导致严重的性能乃至雪崩问题。
  
  - 无法在一个事务中同时更新多个key。

- Watch 机制可靠性问题
  
  etcd v2 是内存型、不支持保存 key 历史版本的数据库，只在内存中使用滑动窗口保存了最近的 1000 条变更事件，当 etcd server 写请求较多、网络波动时等场景，很容易出现事件丢失问题。

- 性能瓶颈问题
  
  - HTTP/1.x
    
    - HTTP/1.x 协议没有压缩机制，拉取大量数据会导致出现 CPU 高负载、OOM、丢包等问题。
    
    - HTTP/1.x 不支持多路复用，etcd v2 client 会通过 HTTP 长连接轮询 Watch 事件，当 watcher 较多的时候，会创建大量的连接，消耗 server 端过多的 socket 和内存资源。
  
  - TTL
    
    不支持key的批量续期，如果大量key 拥有相同的 TTL，也需要分别为每个 key 发起续期操作，这会显著增加集群负载、导致集群性能显著下降。
  
  - 内存开销
    
    etcd v2 在内存维护了一颗树来保存所有节点 key 及 value，在数据量场景略大的场景会导致较大的内存开销，同时需要定时把全量内存树持久化到磁盘。这会消耗大量的 CPU 和磁盘 I/O 资源，对系统的稳定性造成一定影响。

### etcd v3的改进

在内存开销、Watch 事件可靠性、功能局限上

- 引入 B-tree、boltdb 实现一个 MVCC 数据库，数据模型从层次型目录结构改成扁平的 key-value，提供稳定可靠的事件通知，实现了事务，支持多 key 原子更新，

- 基于 boltdb 的持久化存储，显著降低了 etcd 的内存占用、避免了 etcd v2 定期生成快照时的昂贵的资源开销。

性能上

- 使用了 gRPC API，使用 protobuf 定义消息，消息编解码性能相比 JSON 超过 2 倍以上，并通过 HTTP/2.0 多路复用机制，减少了大量 watcher 等场景下的连接数。

- 使用 Lease 优化 TTL 机制，每个 Lease 具有一个 TTL，相同的 TTL 的 key 关联一个 Lease，Lease 过期的时候自动删除相关联的所有 key，不再需要为每个 key 单独续期。

- 支持范围、分页查询，可避免大包等 expensive request。

---

# ch02 基础架构

## 基础架构

![](assets/471ab5dff3ee930a2b53bebd7c3a1ebae1731129.png)

etcd 可分为 Client 层、API 网络层、Raft 算法层、逻辑层和存储层。

这些层的功能如下：

- Client 层
  
  Client 层包括 client v2 和 v3 两个大版本 API 客户端库，提供了简洁易用的 API，同时支持负载均衡、节点间故障自动转移，可极大降低业务使用 etcd 复杂度，提升开发效率、服务可用性。

- API 网络层
  
  API 网络层主要包括 client 访问 server 和 server 节点之间的通信协议。一方面，client 访问 etcd server 的 API 分为 v2 和 v3 两个大版本。v2 API 使用 HTTP/1.x 协议，v3 API 使用 gRPC 协议。同时 v3 通过 etcd grpc-gateway 组件也支持 HTTP/1.x 协议，便于各种语言的服务调用。另一方面，server 之间通信协议，是指节点间通过 Raft 算法实现数据复制和 Leader 选举等功能时使用的 HTTP 协议。

- Raft 算法层
  
  Raft 算法层实现了 Leader 选举、日志复制、ReadIndex 等核心算法特性，用于保障 etcd 多个节点间的数据一致性、提升服务可用性等，是 etcd 的基石和亮点。

- 功能逻辑层
  
  etcd 核心特性实现层，如典型的 KVServer 模块、MVCC 模块、Auth 鉴权模块、Lease 租约模块、Compactor 压缩模块等，其中 MVCC 模块主要由 treeIndex 模块和 boltdb 模块组成。

- 存储层
  
  存储层包含预写日志 (WAL) 模块、快照 (Snapshot) 模块、boltdb 模块。其中 WAL 可保障 etcd crash 后数据不丢失，boltdb 则保存了集群元数据和用户写入的数据。

### client

1. etcdctl 对命令中的参数进行解析

2. 根据请求参数后会创建一个 clientv3 库对象，使用 KVServer 模块的 API 来访问 etcd server。

3. 针对每一个请求，Round-robin 算法通过轮询的方式依次从 endpoint 列表中选择一个 endpoint 访问 (长连接)，使 etcd server 负载尽量均衡。

4. 选择好 etcd server 节点，client 就可调用KVServer 模块的 Range RPC 方法，把请求发送给 etcd server。

> client 和 server 之间的通信，使用的是基于 HTTP/2 的 gRPC 协议。相比 etcd v2 的 HTTP/1.x，HTTP/2 是基于二进制而不是文本、支持多路复用而不再有序且阻塞、支持数据压缩以减少包大小、支持 server push 等特性。因此，基于 HTTP/2 的 gRPC 协议具有低延迟、高性能的特点，有效解决了我们在上一讲中提到的 etcd v2 中 HTTP/1.x 性能问题。

### KVServer

etcd 提供了丰富的 metrics、日志、请求行为检查等机制，可记录所有请求的执行耗时及错误码、来源 IP 等，也可控制请求是否允许通过，比如 etcd Learner 节点只允许指定接口和参数的访问，帮助大家定位问题、提高服务可观测性等，而这些特性是怎么非侵入式的实现呢？答案就是拦截器。

拦截器提供了在执行一个请求前后的 hook 能力，除了我们上面提到的 debug 日志、metrics 统计、对 etcd Learner 节点请求接口和参数限制等能力，etcd 还基于它实现了以下特性:

### 串行读与线性读

我先简单提下写流程。如下图所示，当 client 发起一个更新 hello 为 world 请求后，若 Leader 收到写请求，它会将此请求持久化到 WAL 日志，并广播给各个节点，若一半以上节点持久化成功，则该请求对应的日志条目被标识为已提交，etcdserver 模块异步从 Raft 模块获取已提交的日志条目，应用到状态机 (boltdb 等)。

在多节点 etcd 集群中，各个节点的状态机数据一致性存在差异

- 对数据敏感度较低的场景
  
  如果业务场景对实对数据时效性要求并不高，读请求可直接从节点的状态机获取数据。即便数据落后一点，也不影响业务。
  
  这种直接读状态机数据返回、无需通过 Raft 协议与集群进行交互的模式，在 etcd 里叫做串行读，它具有低延时、高吞吐量的特点，适合对数据一致性要求不高的场景。

- 对数据敏感性高的场景
  
  如果业务场景对数据实时性要求高，必须读取到最新的数据，可以通过线性读来满足。
  
  线性读是指，一旦一个值更新成功，随后任何通过线性读的 client 都能及时访问到。虽然集群中有多个节点，但 client 通过线性读就如访问一个节点一样。etcd 默认读模式是线性读，因为它需要经过 Raft 协议模块，反应的是集群共识，因此在延时和吞吐量上相比串行读略差一点，适用于对数据一致性要求高的场景。

### ReadIndex

线性读的实现机制

1. follower收到一个线性读请求并转发给ReadIndex

2. ReadIndex从 Leader 获取集群最新的已提交的日志索引 (committed index)

3. Leader 向 Follower 节点发送心跳确认，一半以上节点确认 Leader 身份后才能将已提交的索引 (committed index) 返回给接受请求的节点

4. C 节点则会等待，直到状态机已应用索引 (applied index) 大于等于 Leader 的已提交索引时 (committed Index)， 通知KVServer，数据已赶上 Leader

5. 去状态机中访问数据并返回

### MVCC

多版本并发控制 (Multiversion concurrency control) 模块是为了解决上一讲我们提到 etcd v2 不支持保存 key 的历史版本、不支持多 key 事务等问题而产生的。

由内存树形索引模块 (treeIndex) 和嵌入式的 KV 持久化存储库 boltdb 组成。

首先我们需要简单了解下 boltdb，它是个基于 B+ tree 实现的 key-value 键值库，支持事务，提供 Get/Put 等简易 API 给 etcd 操作。

- boltdb
  
  key是全局递增的版本号(revision)，value 是用户 key、value 等字段组合成的结构体

- treeIndex
  
  key是用户 key，value是版本号

对一个读事务而言

1. 获取key的版本号(revision)

2. 根据版本号在buffer查询，命中则返回

3. 未命中则从boltdb查询value

### treeIndex

treeIndex基于B-tree 数据结构保存用户 key 与版本号之间的映射关系

treeIndex 模块只会保存用户的 key 和相关版本号信息，用户 key 的 value 数据存储在 boltdb 里面，而它需要从 treeIndex 模块中获取 hello 这个 key 对应的版本号信息。

treeIndex 模块基于 B-tree 快速查找此 key，返回此 key 对应的索引项 keyIndex 即可。索引项中包含版本号等信息。

### buffer

etcd 出于数据一致性、性能等考虑，在访问 boltdb 前，首先会从一个内存读事务 buffer 中，二分查找你要访问 key 是否在 buffer 里面，若命中则直接返回。

### boltdb

若 buffer 未命中，此时就真正需要向 boltdb 模块查询数据。

boltdb 是如何隔离集群元数据与用户数据的呢？

答案是 bucket。boltdb 里每个 bucket 类似对应 MySQL 一个表，用户的 key 数据存放的 bucket 名字的是 key，etcd MVCC 元数据存放的 bucket 是 meta。

因 boltdb 使用 B+ tree 来组织用户的 key-value 数据，获取 bucket key 对象后，通过 boltdb 的游标 Cursor 可快速在 B+ tree 找到 key 对应的 value 数据，返回给 client。
