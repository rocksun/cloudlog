# EKS 授权管理

使用云服务提供的 Kubernetes 集群都要解决一个问题，即将云服务的账号映射到 kubernetes 集群，然后给相应的用户授权。

在 EKS 中，通过 eksctl 创建的集群会自动把创建者加到 system:masters 组中，拥有最高的权限。

其他 AWS 用户，可以通过本文的步骤授予相应的权限。

## 关联 AWS 用户到 Kubernetes 集群

EKS 使用 kube-system 下的 ConfigMap 存放 AWS 用户和 Kubernetes 用户的关联，可以使用这个命令直接编辑 mapUsers 部分：

```
kubectl edit -n kube-system configmap/aws-auth
···
  mapUsers: |
    - groups:
      - system:masters
      userarn: arn:aws-cn:iam::111111:user/someuser
      username: someuser
···
```

不过，直接编辑 yaml 文件容易出错，所以 eksctl 提供了一个命令实现了相同的功能。通过以下命令可以查看已经关联的用户（也可以是role）：

```
eksctl get iamidentitymapping --cluster some-cluster
```

通过以下命令获取用户的 arn :

```
aws iam get-user --user-name someuser
```

通过以下命令创建关联，并加入到 system:masters 组，对应的 kubernetes 下的用户名是 someuser：

```
eksctl create iamidentitymapping --cluster some-cluster --arn arn:aws-cn:iam::111111:user/someuser --group system:masters --username someuser
```

但一般情况下我们需要创建一个一般权限的用户，所以不会加到 system:masters 组里：

```
eksctl create iamidentitymapping --cluster some-cluster --arn arn:aws-cn:iam::111111:user/someuser --username someuser
```

如果添加有误，可以用以下命令删除关联：

```
eksctl delete iamidentitymapping --cluster some-cluster --arn arn:aws-cn:iam::111111:user/someuser
```

## 用户授权 - 内置 Role

我们一般不能给 kubernetes 用户所有的权限，而只会给集群中某个命名空间的权限。

通过 ```kubectl get clusterroles``` 可以查看集群内置的 ClusterRole 。可以看到其中内置了一个 edit 角色 。我们创建一个 mynamespace-rolebinding.yaml :

```yaml
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: mynamespace-edit
  namespace: mynamespace
subjects:
- kind: User
  name: "someuser"
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: edit
```

执行 ```kubectl apply -f mynamespace-rolebinding.yaml``` 后，可以用以下命令验证用户是否有权限：

```
kubectl get deployments --as=someuser --namespace mynamespace
```

以后要增加其他用户，只需要在 subjects 下增加即可。

## 用户添加 kubeconfig

这时候用户可以添加 config 到自己的客户端。用户在自己的环境下执行：

```
aws eks --region cn-northwest-1 update-kubeconfig --name some-cluster
```

即可将 config 加入到本机环境。

## 参考

* [Manage IAM users and roles](https://eksctl.io/usage/iam-identity-mappings/)
* [Create a kubeconfig for Amazon EKS](https://docs.aws.amazon.com/eks/latest/userguide/create-kubeconfig.html)
* [Using RBAC Authorization](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)
* [Rancher Kubernetes RBAC integration](https://rancher.com/docs/rancher/v1.6/en/kubernetes/rbac/)