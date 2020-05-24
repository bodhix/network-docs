## net namespace

### 简介

namespace（命名空间）为 Linux 上的资源对象提供多个视图，实现资源的隔离，是容器技术的基础。Linux 为众多的资源提供 namespace，这里介绍和网络相关的，网络命名空间（netns）([Linux man]( http://man7.org/linux/man-pages/man7/network_namespaces.7.html ))。

netns 隔离了一切的网络资源，包括网络设备、邻居表、路由器表等。不同 netns 之间的这些资源都是不可见的。netns 之间的连通，主要依靠 [veth pair](veth-pair-link)。

### 使用场景

- 容器技术。实现容器之间的网络隔离，包括 LXC/LXD、docker 等
- 转发隔离。主要是 VRF（virtual routing and forwarding）。在内核直接支持 VRF 之前，有些厂家或开源产品会使用 netns 实现数据面虚拟化（一虚多），本质上，还是网络资源的隔离。

### 使用方法

```c
/* 创建进程的时候，指定创建新的 namespace */
int clone(int (*fn)(void *), void *child_stack,
          int flags, void *arg, ...
          /* pid_t *ptid, void *newtls, pid_t *ctid */ );

/* 将当前进程设置到某个 namespace
 * fd 是打开 /proc/[pid]/ns/ 底下某个 namespace 获得的文件描述符
 * 主要作用是将当前进程和某个已存在进程使用同个 namespace
 */
int setns(int fd, int nstype);

/* 将当前进程脱离当前 namespace，相当于是创建新的 namespace */
int unshare(int flags);
```

```shell
root@bodhix:~# ip netns help
Usage: ip netns list
       ip netns add NAME
       ip netns set NAME NETNSID
       ip [-all] netns delete [NAME]
       ip netns identify [PID]
       ip netns pids NAME
       ip [-all] netns exec [NAME] cmd ...
       ip netns monitor
       ip netns list-id
root@bodhix:~# 

root@bodhix:~# ll /proc/self/ns/net 
lrwxrwxrwx 1 root root 0 May 24 17:32 /proc/self/ns/net -> 'net:[4026532001]'
root@bodhix:~# 
root@bodhix:~# ip netns exec ns0 bash
root@bodhix:~# ll /proc/self/ns/net 
lrwxrwxrwx 1 root root 0 May 24 17:32 /proc/self/ns/net -> 'net:[4026532682]'

:<<!
通过 ip netns exec [ns-name] ip link/route/neigh list 等查看命令，可以很明显看到，不同的 netns 之间的网络资源都是相互隔离的
!
```

### 原理分析

