# 为 EKS 创建普通用户

本文介绍了如何根据 IAM 和 Kubernetes 的最佳实践，管理 IAM 和 EKS 用户。

根据 Kubernetes 的最佳实践，Kubernetes 的管理员一般会以 namespaces 隔离不同的项目的应用，所以某个项目的相关人员也会授予某个 namespace 的权限。

根据 IAM 的最佳实践，我们一般会给一组相同权限的人创建一个组，并为组授予相应的权限，然后将相关的用户添加到组里。

## 创建 IAM 组和用户

创建组：

```shell
$ aws iam create-group --group-name somegroup
{
    "Group": {
        "Path": "/",
        "GroupName": "somegroup",
        "GroupId": "AIDA3O2E111112XUJNNN",
        "Arn": "arn:aws-cn:iam::781111111120:group/somegroup",
        "CreateDate": "2020-12-09T05:35:32+00:00"
    }
}
```

创建用户：

```shell
$ aws iam create-user --user-name someuser
```

关联用户到组：

```shell
$ aws iam add-user-to-group --user-name someuser --group-name somegroup
$ aws iam get-group --group-name somegroup
{
    "Users": [
        {
            "Path": "/",
            "UserName": "someuser",
            "UserId": "AIDA31111111111XUJNNN",
            "Arn": "arn:aws-cn:iam::781111111120:user/someuser",
            "CreateDate": "2020-12-09T05:37:43+00:00"
        }
    ],
    "Group": {
        "Path": "/",
        "GroupName": "somegroup",
        "GroupId": "AIDA3O2E111112XUJNNN",
        "Arn": "arn:aws-cn:iam::781111111120:group/somegroup",
        "CreateDate": "2020-12-09T05:35:32+00:00"
    }
}
```

## 创建 IAM Policy 并授权给组

somegroup 的用户只会访问 kubernetes api，并不需要访问 AWS console，所以无需 IAM 上的特定权限。

不过，使用 aws 更新 kubeconfig 时需要 "eks:DescribeCluster" 权限，所以这里还是需要添加这个权限。

创建一个 EKSUserPolicy.json :

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "eks:DescribeCluster"
            ],
            "Resource": "*"
        }
    ]
} 
```

然后执行命令创建 Policy ：

```shell
$ aws iam create-policy --policy-name EKSUserPolicy --policy-document file://EKSUserPolicy.json
{                                                                                                                                                                
    "Policy": {
        "PolicyName": "EKSUserPolicy",
        "PolicyId": "ANP111111111111BEPEJT4",
        "Arn": "arn:aws-cn:iam::781111111120:policy/EKSUserPolicy",
        "Path": "/",
        "DefaultVersionId": "v1",
        "AttachmentCount": 0,
        "PermissionsBoundaryUsageCount": 0,
        "IsAttachable": true,
        "CreateDate": "2020-12-09T06:31:15+00:00",
        "UpdateDate": "2020-12-09T06:31:15+00:00"
    }
}
```

然后给组附加 Policy :

```shell
$ aws iam attach-group-policy --policy-arn arn:aws-cn:iam::781111111120:policy/EKSUserPolicy --group-name somegroup
$ aws iam list-attached-group-policies --group-name somegroup
{
    "AttachedPolicies": [
        {
            "PolicyName": "EKSUserPolicy",
            "PolicyArn": "arn:aws-cn:iam::781111111120:policy/EKSUserPolicy"
        }
    ]
}
```

以后新增加的组都可以附加这个 EKSUserPolicy ，各个组之间没有区别。 


## 关联 IAM 和 EKS 用户 

查看相关用户的 arn ：

```shell
$ aws iam get-user --user-name someuser
{
    "User": {
        "Path": "/",
        "UserName": "someuser",
        "UserId": "AIDA31111111111XUJNNN",
        "Arn": "arn:aws-cn:iam::781111111120:user/someuser",
        "CreateDate": "2020-12-09T05:37:43+00:00"
    }
}
```

然后将用户 someuser 关联到 EKS 集群：


```shell
$ eksctl create iamidentitymapping --cluster patos-cluster --arn arn:aws-cn:iam::781111111120:user/someuser --username someuser
```

## 配置 RoleBindings

配置 rolebing 文件 somenamespace-rolebinding.yaml ：

```yaml
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: somenamespace-edit
  namespace: somenamespace
subjects:
- kind: User
  name: "someuser"
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: edit
```

## 创建 access key

为 someuser 创建 access key:

```shell
$ aws iam create-access-key --user-name someuser
{
    "AccessKey": {
        "UserName": "someuser",
        "AccessKeyId": "AKIA1111111111111PRWB",
        "Status": "Active",
        "SecretAccessKey": "GKvT1/IhP1111111111111111clyXiN",                                                                                                                                        
        "CreateDate": "2020-12-09T09:00:42+00:00"                                                                                                                                                             
    }                                                                                                                                                                                                         
}   
```

## 用户使用 EKS

有了 access key，普通用户就可以配置使用集群了，配置 aws 命令行客户端：

```shell
$ aws configure
AWS Access Key ID [****************PRWB]: AKIA1111111111111PRWB
AWS Secret Access Key [****************yXiN]: GKvT1/IhP1111111111111111clyXiN
Default region name [cn-northwest-1]:
Default output format [json]:
```

然后执行 ```aws eks update-kubeconfig --region cn-northwest-1  --name some-cluster``` 即可完成配置。

然后使用```kubectl config set-context --current --namespace=somenamespace```切换命名空间，执行 ```kubectl get pods``` 可以查看结果。

## 参考

* [Amazon EKS identity-based policy examples](https://docs.aws.amazon.com/eks/latest/userguide/security_iam_id-based-policy-examples.html)
* [Security best practices in IAM](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html)
