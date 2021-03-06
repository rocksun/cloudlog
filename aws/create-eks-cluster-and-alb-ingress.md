# 在 AWS 中国使用 eksctl 配置集群和 Ingress Controller

[AWS官方文档](https://docs.aws.amazon.com/eks/latest/userguide/alb-ingress.html)说的比较清楚，但我们在中国区实际操作时发现了很多小问题，所以写了这个文章，希望能给大家提供一些帮助。

## 安装命令行工具

直接参考[快速入门](https://docs.aws.amazon.com/eks/latest/userguide/getting-started-eksctl.html)安装相关命令：

* aws
* eksctl
* kubectl

我是 windows 环境，尽管文档里提示使用 chocolatey 安装 eksctl，但我发现我用 [scoop](https://scoop.sh/) 安装也是可以的。

## 配置认证授权

登录 AWS 国内账号，进入[我的安全凭证](https://console.amazonaws.cn/iam/home#/security_credentials)配置一个“用于访问 CLI、开发工具包和 API 的访问密钥”。

命令行里执行：

```
>aws configure
AWS Access Key ID [****************VXHO]: ********JKOJGURK
AWS Secret Access Key [****************1axG]: *************i+ik1xTBFiAgZkjpcw
Default region name [ap-northeast-1]: cn-northwest-1
Default output format [json]:
```

## 创建集群

创建集群配置文件 rocksun-cluster.yaml :

```yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: rocksun-cluster
  region: cn-northwest-1
  version: '1.18'

nodeGroups:
  - name: ng-1
    instanceType: m5.large
    desiredCapacity: 2
    volumeSize: 80
```

然后执行：

```
eksctl create cluster -f rocksun-cluster.yaml  
```

这样就会创建一个名为 rocksun-cluster 集群，一个 Node Group ，以及对应的一个包含两个实例的 Auto Scaling 组。

## 创建 ingress-controller

1. 创建  IAM OIDC provider 并与集群关联：

```
eksctl utils associate-iam-oidc-provider --region cn-northwest-1 --cluster rocksun-cluster --approve
[ℹ]  eksctl version 0.31.0
[ℹ]  using region cn-northwest-1
[ℹ]  will create IAM Open ID Connect provider for cluster "rocksun-cluster" in "cn-northwest-1"
[✔]  created IAM Open ID Connect provider for cluster "rocksun-cluster" in "cn-northwest-1"
```

2. 国内的 policy 与海外并不相同，所以需要下载 [iam_policy_cn.json](https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/main/docs/install/iam_policy_cn.json)。

3. 创建 IAM policy ：

```
aws iam create-policy --policy-name ALBIngressControllerIAMPolicy --policy-document file://E:\projects\cluster-config\iam_policy_cn.json
{
    "Policy": {
        "PolicyName": "ALBIngressControllerIAMPolicy",
        "PolicyId": "ANPA3O2*******E2S4RVR",
        "Arn": "arn:aws-cn:iam::787*******020:policy/ALBIngressControllerIAMPolicy",
        "Path": "/",
        "DefaultVersionId": "v1",
        "AttachmentCount": 0,
        "PermissionsBoundaryUsageCount": 0,
        "IsAttachable": true,
        "CreateDate": "2020-11-12T07:44:04+00:00",
        "UpdateDate": "2020-11-12T07:44:04+00:00"
    }
}

```
需要注意 --policy-document 参数的写法，必须按照上述格式。记录上述步骤得到的 Arn 。

4. 创建 IAM 和 Kubernetes service account，以及 cluster role 以及 role binding：

```
eksctl create iamserviceaccount --cluster=patos-cluster --namespace=kube-system --name=aws-load-balancer-controller --attach-policy-arn=arn:aws-cn:iam::787734088020:policy/ALBIngressControllerIAMPolicy --override-existing-serviceaccounts --approve
```

5. 下载 [cert-manager.yaml](https://github.com/jetstack/cert-manager/releases/download/v1.0.2/cert-manager.yaml) 和 [v2_0_0_full.yaml](https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/main/docs/install/v2_0_0_full.yaml) 并手工安装 controller 


修改 --cluster-name=your-cluster-name 为 --cluster-name=rocksun-cluster 。删除其中的 ServiceAccount 资源。

因为国内 AWS 环境缺少了一些相关的服务，还需要在 Deployment 下增加这几个参数：

```yaml
    spec:
      containers:
        - args:
         ....
            - --enable-shield=false
            - --enable-waf=false
            - --enable-wafv2=false

```

然后执行：


```
kubectl apply --validate=false -f cert-manager.yaml
kubectl apply -f v2_0_0_full.yaml
```

## 应用部署测试

下载[测试应用](https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/main/docs/examples/2048/2048_full.yaml)，因为国内的一些原因默认端口是不能访问，所以还需要编辑 2048_full.yaml，在 Ingress 资源的 annotations 中增加：

```yaml
alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 8081}]'
```

执行 kubectl apply -f 2048_full.yaml ，然后过一会儿执行：

```
kubectl get ingress/ingress-2048 -n game-2048

NAME           CLASS    HOSTS   ADDRESS                                                                          PORTS   AGE
ingress-2048   <none>   *       k8s-game2048-ingress2-*****************.cn-northwest-1.elb.amazonaws.com.cn   80      16m
```

然后浏览器里访问 http://k8s-game2048-ingress2-*****************.cn-northwest-1.elb.amazonaws.com.cn:8081 即可看到 2048 的界面。


## 其他用户授权

通过 eksctl 创建的集群会自动把创建者加到 system:masters 组中，拥有最高的权限。如果需要添加其他的用户，需要首先创建 arn 到用户的映射。首先需要获取用户的 arn ：

```shell
aws iam get-user --user-name someuser
{
    "User": {
        "Path": "/",
        "UserName": "someuser",
        "UserId": "AIDA**********3K6",
        "Arn": "arn:aws-cn:iam::7877****020:user/someuser",
        "CreateDate": "2020-10-29T09:47:55+00:00",
        "PasswordLastUsed": "2020-11-16T00:52:21+00:00"
    }
}
```

然后创建映射：

```
eksctl create iamidentitymapping --cluster rocksun-cluster --arn arn:aws-cn:iam::7877****020:user/someuser --group system:masters --username someuser
```

通过 --group 参数，我们也将该用户添加到了 system:masters ，所以这个用户也具备了集群的管理员权限。


## 问题与心得

### Application Load Balancer（ALB）很强大也很重

使用一般的 ingress controller 时，创建一个 ingress 只是在 ingress controller 服务上添加了一个配置，但是对于[AWS Load Balancer Controller](https://github.com/kubernetes-sigs/aws-load-balancer-controller)来说要复杂的多，会在 AWS 上创建一个 Application Load Balancer 和多个 Target Groups 。一个 ALB 会包含多个网络接口和许多其他相关资源，这带来了一系列的问题。

* 因为创建 ALB 的过程较慢，创建期间验证 Ingress 功能时会失败，所以需要耐心观测 ALB 的状态再验证。
* 每次新建 ingress 时对应的 ADDRESS 也会发生变化，所以应用部署时应该避免删除 Ingress，而应该采用更新方式。
* 因为每个 ingress 对应了一组 AWS 资源，所以删除时也需要通过 ingress 删除。特别是删除集群前，需要将所有的 ingress 删除完，否则手工删除 ingress 相关资源会有点麻烦。

### AWS 有门槛，必须使用命令行

AWS 的多可用区（AZ）以及其他设计，架构上更加完美，再加上不太快的 UI 访问速度，确实会会给 AWS 的日常管理带来很大的成本，会使很多习惯于在云服务界面上点来点去的同事感觉到不适应。不过，很多程序员还是喜欢命令行的，可以让自己所做的操作有一个清晰的描述，日久天长会大大提高工作效率。

### Windows 实例巨贵

知道 Windows 会有 license 的成本问题，但是没想到会贵这么多（其他云的 Linux 和 Windows 实例都是一个价格，其中的原理是？），让我望而却步，各位有 windows 实例需求的需要三思。

### AWS 的定位

目前我们还只是把 AWS 当作测试环境，但是 AWS 更加复杂的底层设计确实更适合生产环境。

## 参考

* [eksctl 快速入门](https://docs.aws.amazon.com/eks/latest/userguide/getting-started-eksctl.html)
* [创建 ingress controller](https://docs.aws.amazon.com/eks/latest/userguide/alb-ingress.html)
* [AWS Load Balancer Controller](https://github.com/kubernetes-sigs/aws-load-balancer-controller)
