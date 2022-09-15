# 生命周期管理与服务发现

## 深入理解Pod的生命周期

### Pod的生命周期

![image-20220319093945318](assets/image-20220319093945318.png)

#### Pod状态机

![image-20220319094043905](assets/image-20220319094043905.png)

#### Pod Phase

- Pending
- Running
- Succeeded
- Failed
- Unknown

kubectl get pod显示的status是根据pod status中的conditions和phase计算出来的

pod状态计算细节

![](assets/2022-09-09-16-06-08-image.png)

查看Pod细节：```kubectl get pod <podname> -oyaml```

查看Pod相关事件：```kubectl describe pod <podname>```

#### Pod的高可用

避免容器进程被终止、避免Pod被驱逐

- 设置合理的resource.memory limits，防止容器进程被OOM kill
- 设置合理的emptyDir.sizeLimit，确保数据写入不超过emptyDir的限制，防止Pod被驱逐

##### Pod的QoS class

1. Guaranteed
   
   - Pod中每个容器都设置了CPU和内存的资源
   - Pod中每个容器CPU和内存的limits和requests的值完全一致
   
   这一类型的主要作用是保护重要的Pod

2. Burstable
   
   - Pod中至少一个容器设置了CPU或内存的requests
   
   这一类型适合大多数场景，需要合理设置limits和requests，有利于提高集群资源利用率并减少Pod被驱逐的概率

3. BestEffort
   
   - Pod中没有容器设置内存和 CPU 的limits和requests
   
   这一类型应该在测试环境使用，可以确保大多数应用不会因为资源问题处于pending状态，只要有可用的节点，就可以将Pod调度上去

当计算节点检测到内存压力时，kubernetes会按照BestEffort、Burstable、Guaranteed的顺序驱逐Pod

这里的驱逐与Pod的优先级也有关系，具体可参考官方文档[节点压力驱逐](https://kubernetes.io/zh/docs/concepts/scheduling-eviction/node-pressure-eviction/#pod-selection-for-kubelet-eviction)

##### 基于Taints的Evictions

kubernetes会自动为Pod增加not-raedy和unreachable的tolerations

可以通过增加not-ready或unreachable的toleration Seconds，避免因为短时间的节点状态不正常而被驱逐

##### 健康检查探针

类型：

- livenessProbe：存活状态检查
  
  默认状态为Success，失败意味着容器运行状态不正常，kubelet 会杀死容器，并需要根据restartPolicy对容器执行操作

- readinessProbe：就绪状态检查
  
  默认状态为Failure，失败意味着应用已无法正常提供服务，kubelet会将该容器从服务列表中删除

- startupProbe：初始化阶段的健康检查
  
  指示容器中的应用是否已经启动。如果提供了启动探针，则所有其他探针都会被 禁用，直到此探针成功为止。如果启动探测失败，kubelet将杀死容器，而容器依其restartPolicy对容器执行操作

探测方法：

- ExecAction
  
  在容器内部运行指定命令，返回码为0表示探测成功

- TCPSocketAction
  
  kubelet通过TCP协议检查容器IP和端口，端口可达即为探测成功

- HTTPGetAction
  
  kubelet对Pod的IP和指定端口发起HTTP GET请求，返回码在200-400之间表示探测成功

探针属性：

- initialDelaySeconds：初始执行延迟时间

- periodSeconds：检查间隔时间

- timeoutSeconds：执行超时时间

- successThreshold：连续成功次数

- failureThreshold：连续出错次数

readinessGates

可扩展的就绪条件（conditions）

新引入的readinessGates condition需要状态为True后，Pod才可以为就绪状态，此状态应该由控制器修改

##### Post-start与Pre-stop

Post-start

在启动容器时执行Post-start脚本，容器的Entrypoint与Post-Start脚本没有绝对的先后顺序，无法保证哪一个先执行

![image-20220319120110654](assets/image-20220319120110654.png)

Pre-stop

首先执行Pre-Stop脚本，此时grace period开始计时，执行完成后，将发生SIGTERM信号给容器，如果容器进程响应SIGTERM，则容器可以优雅停止，如果到了grace period达到预定时间，容器还没有停止，将发送SIGKILL信号，强行终止容器。

![image-20220319120238899](assets/image-20220319120238899.png)

如果Pre-Stop脚本或者对于SIGTERM信号的处理时间较长，为了保证处理过程正常结束，需要手动延长terminationGracePeriodSeconds

![image-20220321103357234](assets/image-20220321103357234.png)

注意：由于bash/sh会忽略SIGTERM信号，所以SIGTERM会永远超时，如果应用使用bash/sh作为Entrypoint，应避免使用过长的grace period

terminationGracePeriodSeconds的默认时长为30秒，如果不关心终止时长，则不需要特殊处理

如果需要优雅终止，可以选择在Pre-Stop脚本中主动退出进程，还可以在容器中使用特定的初始化进程，通过响应SIGTERM信号的方式优雅终止

一个优雅的初始化进程[tini](https://github.com/krallin/tini)

- 正确处理系统信号，将信号转发给子进程
- 在主进程退出前，确保所有子进程退出
- 监控并清理孤儿进程

### 在kubernetes上部署应用的挑战

#### 资源规划

- 计算资源
  - CPU/GPU
  - memory
- 存储资源
  - 容量大小
  - 存储位置：本地或网盘
  - IO性能
- 网络资源
  - QPS和带宽

#### 存储的挑战

多容器共享存储最简单的是emptyDir

- 对emptyDir设置size limit
- 达到size limit后，容器将会被驱逐，原Pod的配置信息会丢失

#### 应用配置

传入方式：

- Environment Variables
- Volume Mount

数据来源：

- ConfigMap
- Secret
- Download API

数据存储

![](assets/2022-09-10-16-55-52-image.png)

- emptyDir
  
  Pod重启后数据会丢失

- hostPath
  
  需要额外权限控制

- Local Volume
  
  需要额外备份机制

- Network Volume
  
  速度与稳定性问题

- rootFS
  
  不建议写任何数据

#### 容器应用可能面临的进程中断

![](assets/2022-09-10-16-59-11-image.png)

- kubelet升级
  
  容器重建：冗余部署，跨故障域部署

- 节点操作系统升级
  
  Pod进程被终止：跨故障域部署，增加Liveness、Readiness探针

- 节点下架、送修
  
  节点被排空，从集群中删除：跨故障域部署，配置PoddisruptionBudget控制失效情况，利用Pre-Stop脚本做数据备份

- 节点崩溃
  
  跨故障域部署

#### 高可用部署方式

- 多实例跨故障域部署
- 合理的更新策略：
  - maxSurge：先创建新版本的Pod，再删除旧版本的Pod
  - maxUnavailable：有Pod状态异常时，暂停更新

## 服务发现

### 服务发布

不同的服务发布类型

- ClusterIP（Headless）

- NodePort

- LoadBalancer

- ExternalName

服务发布的挑战

kube-dbs

- DNS TTL问题

Service

- ClusterIP只能对内

- kube-proxy支持的iptables/ipvs支持的规模有限

- ipvs的性能和生产化问题

- kube-proxy的drift问题

- 频繁的pod变动导致LB频繁变更

- 对外发布的Service需要与企业LB集成

- 不支持gRPC

- 不支持自定义DNS和高级路由功能

Ingress

- Spec的成熟度不足

跨地域部署

- 需要多少实例

- 失败域控制

- 精细的流量控制

- 按地域的顺序更新

- 发布与回滚

## 微服务架构下的高可用

## Service

## kube-proxy

- userspace
  
  用户空间监听端口，

- iptables

- ipvs

- winuserspace

出节点流量转换 通过iptables的IPset