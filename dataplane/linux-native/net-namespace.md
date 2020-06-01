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

接下来，我们从 ```  ip netns``` 使用入手，来分析 netns 的原理。备注：对于一个陌生的领域，这是一个个人认为比较实用的方法，从功能出发，了解每个操作背后都做了什么，加上一点背景知识和经验，就可以很容易地理解其大致的原理。当然，再详细的内容，就需要看代码了。先来看看以下四个流程的用户空间操作：

```c
/* # ip netns add netns0 */
netns_add [ip/ipnetns.c]
    |-- create_netns_dir：创建 /var/run/netns 目录
    |-- mount
    |-- openat：在 /var/run/netns 目录底下创建 netns0 文件
    |-- unshare：参数为 CLONE_NETNS，前面有介绍，就是创建一个新的命名空间
    |-- mount：mount /proc/self/ns/net 到 /var/run/netns/netns0
```

```c
/* # ip netns del netns0 */
/* 操作很简单，就是 umount /var/run/netns/xxx，然后 unlink
 * 在 umount 的时候，内核应该会对应的做很多清理操作
 */
```

```c
/* # ip netns exec netns0 ip link list */
/* 在使用 execvp 执行一个命令之前，先使用 netns_switch 切换一下 netns */
netns_switch [lib/namespace.c]
    |-- open：就是打开 /var/run/netns/xxx
    |-- setns：将当前进程 netns 设置为 xxx
    |-- unshare：? 这个比较奇怪，前面使用 setns，这里为什么还 unshare ?
    |-- ...
```

```c
/* # ip link set dev netns netns0 */
/* 用户空间基本没有操作，就是发送 rtnl 给内核
 * 指定修改设备的 NET_NS 属性（ NET_NS_FD / NET_NS_PID）
 */
```

接下来，根据上面的内容，关注对应的内核空间的原理。

```c
unshare [kernel/fork.c]
    |-- unshare_nsproxy_namespaces
    	|-- create_new_namespaces
    		|-- copy_net_ns
    			|-- inc_net_namespaces：增加 user_name 中 netns 的计数
    			|-- net_alloc：分配一个 struct net 结构（表示一个 netns）
                |-- get_user_ns：增加 user_ns 的引用
                |-- setup_net：调用 netns 所需功能的初始化函数
                    |-- 遍历 pernet_list，调用 ops_init，也就是调用 pernet_list 元素的 init
                |-- 将新建的 net 加入到 net_namespace_list 中（全局变量）
	|-- switch_task_namespaces：将 task_struct 里面的 nsproxy 换成前面创建出来的 nsproxy 结构

setns [kernel/nsproxy.c]
	|-- proc_ns_fget：通过 fd 获取对应的 struct file 结构
    |-- get_proc_ns： 通过 file 获取对应的 struct ns_common 结构 ns
    |-- create_new_namespaces：返回 struct nsproxy 结构 new_nsproxy
    |-- ns->ops->install：为当前进程设置新的 netns
    |-- switch_stack_namespaces
/* 注意 create_new_namespace 只是创建出一个新的 nsproxy，但是里面的内容都是 copy 老的
 * 这里就需要通过 ops->install 去将新的 netns 设置到 nsproxy 中
 */

/* 从上面可以看到，net namespace 有几个关键的数据结构，而且和文件系统关联起来了 */
struct nsproxy; // 表示一个完整的 namespace（fs/uts/ipc/net/cgroup...）
struct net; // 表示一个 netns
struct ns_common; // 所有 namespace 通用的一些逻辑，内嵌在各个 namespace 结构里
extern list_head net_namespace_list; // 将系统中的所有 netns 串起来
```

