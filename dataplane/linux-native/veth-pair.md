## veth pair

### 简介

veth 是 Linux 上的虚拟以太网设备。它可以单独使用，但更多情况下是成对使用，也就是 veth pair。veth pair 中的一个设备收到数据包之后，会发送到对端设备。（[Linux man](http://man7.org/linux/man-pages/man4/veth.4.html)）

### 使用场景
veth pair 主要使用场景是实现不同网络命名空间（namespace）之间的通信。创建一对 veth pair，将两端分别添加到不同的网络命名空间，以实现不同命名空间之间的相互通信。veth pair 最重要的例子，应该就是容器（如 docker）之间的网络通信。

### 使用方法


```shell
root@bodhix:~# ip link list
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP mode DEFAULT group default qlen 1000
    link/ether 00:0c:29:5a:8b:9c brd ff:ff:ff:ff:ff:ff
root@bodhix:~# ip link add veth0 type veth peer name veth1
root@bodhix:~# ip link list
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP mode DEFAULT group default qlen 1000
    link/ether 00:0c:29:5a:8b:9c brd ff:ff:ff:ff:ff:ff
4: veth1@veth0: <BROADCAST,MULTICAST,M-DOWN> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/ether ee:ae:4c:64:f6:cb brd ff:ff:ff:ff:ff:ff
5: veth0@veth1: <BROADCAST,MULTICAST,M-DOWN> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/ether 8a:75:bc:77:c0:12 brd ff:ff:ff:ff:ff:ff
root@bodhix:~# 
```

```shell
root@bodhix:~# ip netns list
root@bodhix:~# ip netns add ns0
root@bodhix:~# ip netns add ns1
root@bodhix:~# ip netns list
ns1
ns0
root@bodhix:~#
```

```shell
root@bodhix:~# ip link set veth0 netns ns0
root@bodhix:~# ip link set veth1 netns ns1
root@bodhix:~# ip link list
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP mode DEFAULT group default qlen 1000
    link/ether 00:0c:29:5a:8b:9c brd ff:ff:ff:ff:ff:ff
root@bodhix:~# ip netns exec ns0 ip link list
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
5: veth0@if4: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/ether 8a:75:bc:77:c0:12 brd ff:ff:ff:ff:ff:ff link-netnsid 1
root@bodhix:~# ip netns exec ns1 ip link list
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
4: veth1@if5: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/ether ee:ae:4c:64:f6:cb brd ff:ff:ff:ff:ff:ff link-netnsid 0
root@bodhix:~# 
```

```shell
root@bodhix:~# ip netns exec ns0 ip addr add 10.1.1.1/24 dev veth0
root@bodhix:~# ip netns exec ns0 ip link set veth0 up
root@bodhix:~# ip netns exec ns1 ip addr add 10.1.1.2/24 dev veth1
root@bodhix:~# ip netns exec ns1 ip link set veth1 up
root@bodhix:~# ip netns exec ns0 ip addr list
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
5: veth0@if4: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 8a:75:bc:77:c0:12 brd ff:ff:ff:ff:ff:ff link-netnsid 1
    inet 10.1.1.1/24 scope global veth0
       valid_lft forever preferred_lft forever
    inet6 fe80::8875:bcff:fe77:c012/64 scope link 
       valid_lft forever preferred_lft forever
root@bodhix:~# ip netns exec ns1 ip addr list
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
4: veth1@if5: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether ee:ae:4c:64:f6:cb brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 10.1.1.2/24 scope global veth1
       valid_lft forever preferred_lft forever
    inet6 fe80::ecae:4cff:fe64:f6cb/64 scope link 
       valid_lft forever preferred_lft forever
root@bodhix:~# 
```

```shell
root@bodhix:~# ip netns exec ns1 ping 10.1.1.1 -c 4 -I veth1
PING 10.1.1.1 (10.1.1.1) from 10.1.1.2 veth1: 56(84) bytes of data.
64 bytes from 10.1.1.1: icmp_seq=1 ttl=64 time=0.061 ms
64 bytes from 10.1.1.1: icmp_seq=2 ttl=64 time=0.084 ms
64 bytes from 10.1.1.1: icmp_seq=3 ttl=64 time=0.088 ms
64 bytes from 10.1.1.1: icmp_seq=4 ttl=64 time=0.053 ms

--- 10.1.1.1 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3066ms
rtt min/avg/max/mdev = 0.053/0.071/0.088/0.017 ms
root@bodhix:~# ip netns exec ns0 ping 10.1.1.2 -c 4 -I veth0
PING 10.1.1.2 (10.1.1.2) from 10.1.1.1 veth0: 56(84) bytes of data.
64 bytes from 10.1.1.2: icmp_seq=1 ttl=64 time=0.036 ms
64 bytes from 10.1.1.2: icmp_seq=2 ttl=64 time=0.083 ms
64 bytes from 10.1.1.2: icmp_seq=3 ttl=64 time=0.077 ms
64 bytes from 10.1.1.2: icmp_seq=4 ttl=64 time=0.042 ms

--- 10.1.1.2 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3082ms
rtt min/avg/max/mdev = 0.036/0.059/0.083/0.022 ms
root@bodhix:~# 
```

### 原理分析

本小节，我们分析 veth-pair 是如何实现从一端设备收到数据包之后马上转发给对端设备的。

```c
[drivers/net/veth.c]
```
veth 通过其 net_device 的 priv 和对端的 veth 关联起来
```c
struct veth_priv {
	struct net_device __rcu	*peer; // 关联对端设备
	atomic64_t		dropped;
	unsigned		requested_headroom;
};
```
在内核协议栈中，设备进行数据包发送的时候，最终会调用 dev->netdev_ops->ndo_start_xmit。
veth 设备的 ndo_start_xmit 被设置为 veth_xmit。
```c
static const struct net_device_ops veth_netdev_ops = {
	.ndo_init            = veth_dev_init,
	.ndo_open            = veth_open,
	.ndo_stop            = veth_close,
	.ndo_start_xmit      = veth_xmit, // 设置发送函数
	...
};
```
veth_xmit 的主要调用堆栈如下：

```c
veth_xmit(struct sk_buff *skb, struct net_device *dev)
|-- dev_forward_skb(struct net_device *dev, struct sk_buff *skb)
	|-- __dev_forward_skb(struct net_device *dev, struct sk_buff *skb)
	|-- netif_rx_internal(struct sk_buff *skb)
    	|-- enqueue_to_backlog
    		|-- ____napi_schedule
/*
 主要流程如下：
 veth_xmit 通过本端 dev 找到对端的 dev，然后调用 dev_forward_skb 直接将数据包发送给对端 dev。
 dev_forward_skb 先调用 __dev_forward_skb 将 skb 的接收设备设置为 dev，然后再调用
 netif_rx_internal 将 skb 加入到 backlog，触发软中断，走网络设备收包流程。
 */
```

关于网络设备收发包的流程，请看 [待定](xxx) 。

