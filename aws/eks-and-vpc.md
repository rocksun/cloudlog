# EKS 的 VPC 详解

通过 eksctl 创建集群，默认情况下会创建一个专门的 VPC 以及相关的资源，看起来较为复杂，所以有必要了解一下默认的 VPC ，然后才能更好的实现更个性化的配置。

## EKS 集群的默认 VPC 

### 公网子网

eksctl 创建集群时，会新创建一个专门的 VPC ，这个 VPC 会创建 6 个子网，其中有三个是公网子网，例如：

* eksctl-some-cluster-cluster/SubnetPublicCNNORTHWEST1A
* eksctl-some-cluster-cluster/SubnetPublicCNNORTHWEST1B
* eksctl-some-cluster-cluster/SubnetPublicCNNORTHWEST1C

这几个子网会位于不同的可用区(AZ)，会设置成自动分配公有 IPv4 地址(mapPublicIpOnLaunch)，另外会配置一条默认路由，指向互联网网关(internet gateway)。

关联到这些公网的节点组创建的 EC2 节点，默认情况下会自动分配一个公网 IPv4 地址，节点可以直接访问互联网。

### 私网子网

另外，会创建三个私网子网：

* eksctl-some-cluster-cluster/SubnetPrivateCNNORTHWEST1A
* eksctl-some-cluster-cluster/SubnetPrivateCNNORTHWEST1B
* eksctl-some-cluster-cluster/SubnetPrivateCNNORTHWEST1C

这三个子网没有设置自动分配公有 IPv4 地址，默认路由指向的是一个 NAT 网关，这个 NAT 网关位于前述的一个公网子网上，有公网 IPv4 地址。

关联到这些私网的节点组创建的 EC2 节点，默认情况下不会有 IPv4 地址，但这些节点可以通过 NAT 网关访问互联网。

### Ingress 和 LoadBalancer

当我们部署 Ingress (参考[在 AWS 中国使用 eksctl 配置集群和 Ingress Controller](http://yylives.cc/2020/11/16/create-eks-cluster-and-alb-ingress/)) 时， EKS 会自动创建一个 ALB （Application Load Balancer），这个 ALB 有一个固定的 DNS 名称，还会包含 3 个公网 IPv4 地址，分别位于前述的公网子网中。

所以用户可以通过互联网访问 ALB 的 DNS 域名，域名会解析到某个子网负载均衡 IPv4 地址，ALB 再将相应的流量的转发到相应的 Pod 上，这个过程全部在 VPC 中进行。

## EKS 对于 VPC 使用的最佳实践

eksctl 默认创建的 EKS 集群基本就是一种比较合理的使用方式，唯一可能需要调整就是 NodeGroup 所在的子网。

默认情况下创建的节点组会在公网子网中，创建的节点会有公网 IPv4 地址，可以直接访问互联网。而我们在实践中其实可以考虑将节点组创建到私网当中（具体操作办法参考[在 EKS 上管理 NodeGroup](http://yylives.cc/2020/11/20/manage-managed-ng/)），节点只能通过 NAT 网关访问互联网。


## 参考

* [De-mystifying cluster networking for Amazon EKS worker nodes](https://aws.amazon.com/cn/blogs/containers/de-mystifying-cluster-networking-for-amazon-eks-worker-nodes/)
* [What is Amazon VPC?](https://docs.aws.amazon.com/vpc/latest/userguide/what-is-amazon-vpc.html)
* [在 AWS 中国使用 eksctl 配置集群和 Ingress Controller](http://yylives.cc/2020/11/16/create-eks-cluster-and-alb-ingress/)
* [在 EKS 上管理 NodeGroup](http://yylives.cc/2020/11/20/manage-managed-ng/)