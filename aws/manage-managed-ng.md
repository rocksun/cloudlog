

# 在 EKS 上管理 NodeGroup

最初使用的 NodeGroup 的 InstantType 规格太低，不太好用，所以需要增加一个新的 NodeGroup 。

在之前的 cluster 配置文件 patos-cluster-with-mng.yaml 中，我们使用的是 nodeGroups ，这是非 Managed 的 NodeGroup ，在 EKS 的界面上是看不到的，根据官方文档的说法， Managed NodeGroup 是完全由 EKS 管理的 NodeGroup，所以应该是更好一点。

## 操作过程

```yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: patos-cluster
  region: cn-northwest-1
  version: '1.18'

nodeGroups:
  - name: ng-1
    instanceType: m5.large
    desiredCapacity: 2
    volumeSize: 80

managedNodeGroups:
- name: mng-1
  instanceType: t3a.2xlarge
  minSize: 2
  maxSize: 2

```

managedNodeGroups 部分即为新增加的内容，然后执行：

```shell
eksctl create nodegroup --config-file=patos-cluster-with-mng.yaml
```

执行完成后可以查看 NodeGroup :

```shell
eksctl get nodegroup --cluster patos-cluster

CLUSTER         NODEGROUP       STATUS          CREATED                 MIN SIZE        MAX SIZE        DESIRED CAPACITY        INSTANCE TYPE   IMAGE ID
patos-cluster   mng-1           CREATE_COMPLETE 2020-11-20T08:41:02Z    2               2               2
patos-cluster   ng-1            CREATE_COMPLETE 2020-11-13T01:08:35Z    2               2               2                       m5.large        ami-0
```

然后可以清理之前的 NodeGroup ：

```
eksctl delete nodegroup ng-1 --cluster patos-cluster
```

会提示删除成功。然后执行一下 kubectl 命令，一切正常。

然后我会清除一下 cluster 配置文件中的 nodeGroups 部分，让我的配置文件与实际的集群配置保持一致。

轻松愉快。

## 更新

### 关于 NAT 网关

默认情况下 eckctl 创建的 NodeGroup 都是 public 的，产生的节点都是有公网 IP ，这样虽然很好，但是也给我们有些工作带来了麻烦，比如说配置白名单，我们希望节点都是通过 NAT 网关访问互联网，这样白名单配置就可以相对简单一点。

eks 支持创建 private 的 NodeGroup ，这样 NodeGroup 的 Node 都不会有公网 IP ，会通过 NAT 网关实现对互联网的访问，这样在外部看到的 IP 会是 NAT 网关的 IP，相对稳定。

方法也很简单，前面创建 NodeGroup 时增加 privateNetworking 参数，例如以下示例：

```yaml
managedNodeGroups:
- name: private-mng-1
  instanceType: t3a.2xlarge
  minSize: 2
  maxSize: 2
  privateNetworking: true
```

对 nodeGroups 和 managedNodeGroups 都有效。

## 参考

* [eksctl 中的 NodeGroup 管理](https://eksctl.io/usage/managing-nodegroups/)
* [aws 文档中的 NodeGroup 部分](https://docs.aws.amazon.com/eks/latest/userguide/managed-node-groups.html)