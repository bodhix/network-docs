## VMware

VMware NSX-T Load Balancer 分析

### 关键组件

#### load balancer

LB，关联在 tier-1 网关上。只能关联在 tier-1 网关上，每个 tier-1 网关只能关联一个 LB。

#### virtual server

一个 LB 可以定义多个 virtual server。每个 server 会有一个 vip 和 tcp / udp 端口。每个 virtual server 可以定义独立的策略。

#### pool

提供相同服务的一组服务器，可以使用 IP 地址，也可以使用 Group。LB 可以将流量导到多个 pool，一个 pool 也可以被多个 virtual server 引用。

#### monitor

检测服务 / 应用的可用性。检测的方式可以是简单的 icmp，也可以是复杂的 https 请求。一个 pool 只能使用一个 monitor，但是一个 monitor 可以被多个 pool 使用。

### 部署模式

#### In-line load balancing

图一：

![1589557307319](https://bodhix.github.io/network-docs/services/images/1589557307319.png)

#### 特点

1、客户端和服务器在 LB 的两侧：如 客户端在 Tier-1 的上行链路侧，服务器在 Tier-1 的下行链路侧。【外部 client 访问内部 server】

2、client 请求到 LB 之后，LB 将请求包发送到 server（目的 IP 变为 server-ip）。server 答复包回到 LB，LB 发送给 client（源 IP 变为 vip-ip）。

3、不需要做 LB SNAT，简单。

#### 缺点

1、集中式的，位于 Tier-1 网关（Service Router）上。不同 Tier-1 上的 segment 的东西向的流量需要被导到 Edge node 到达 SR，即使流量不需要经过 LB。

#### One-arm load balancing

图二：

![1589557182035](https://bodhix.github.io/network-docs/services/images/1589557182035.png)

图三：

![1589557527735](https://bodhix.github.io/network-docs/services/images/1589557527735.png)

#### 特点

1、客户端请求流量和 LB 到 server 的转发流量，使用 LB 的**同个网口**。

2、需要使用 SNAT，保证从 server 出来的流量可以穿过 LB 回到 client。

某些场景下可以

这种情况下，需要使用 SNAT，保证从 server 出来的流量可以穿过 LB 回到 client。

#### 其他

1、client 和 server 在同个 segment【client 和 server 都在内部，如上图二】

2、client 和 server 在不同的 segment【如上图三】

注意，该场景下，tier-1 底下不同的 segment 可以有不同的 LB。这种情况下，东西向流量可以走分布式逻辑。

### 高可用

1、Edge node 的高可用：Edge node 之间的心跳

2、应用之间的通知（事件驱动、非心跳）

3、LB 主备之间的状态同步

monitor 是实现在 LB 上的。是分布式的。

### 流量

逻辑上看，LB 是实现在 Tier-1 网关上的，看着是单个实体，实际上，它是在 Tier-1 网关 Service Router 上实例化的。运行在 Edge Node 上。
