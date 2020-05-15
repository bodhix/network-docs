## 负载均衡

### load balancer

#### 负载均衡的多层意义

- 链路层负载均衡：链路聚合、bond / channel
- IP 负载均衡：ECMP、DNS round-robin
- 服务负载均衡：各种负载均衡器（LB）

#### LB 最基本的两个作用

- 聚合多个组件的能力，实现可扩展
- 对外提供虚拟 IP，实现高可用

#### LB 的高级功能

- 应用分发等

#### 常见的开源 LB

- LVS：Linux Virtual Server，内核 ipvs 模块，四层 LB
- Nginx：七层 LB
- HAProxy：四层、七层 LB

#### 参考资料

1. [HAProxy 介绍文档](https://github.com/haproxy/haproxy/blob/v2.0.0/doc/intro.txt)
2. 

