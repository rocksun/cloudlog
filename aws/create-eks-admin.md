# 创建 EKS 管理员

EKS 管理员不仅需要登录管理控制台，也需要通过 eksctl 管理集群，还需要能够管理 EC2 和 CloudFormation 等资源，所以需要较高的权限。

因为 eksctl 需要的权限很高，但是根据最小权限原则，我们又希望授予最小的权限，所以需要根据相关文档小心设置。

## 创建组并关联 Policy

[Minimum IAM policies for eksctl](https://eksctl.io/usage/minimum-iam-policies/) 为我们明确了 eksctl 所需要的权限，根据 IAM 最佳实践，我们会把这个权限加到一个组上。

首先创建一个 EKSAdminGroup 组：

```shell
$ aws iam create-group --group-name EKSAdminGroup
{                                                                                                                                                                                                             
    "Group": {
        "Path": "/",
        "GroupName": "EKSAdminGroup",
        "GroupId": "A11111111111111111W7WN",
        "Arn": "arn:aws-cn:iam::711111111110:group/EKSAdminGroup",
        "CreateDate": "2020-12-10T01:24:16+00:00"
    }
}
```

附加 AmazonEC2FullAccess 和 AWSCloudFormationFullAccess 两个 AWS 托管的 Policy:

```shell
$ aws iam attach-group-policy --policy-arn arn:aws-cn:iam::aws:policy/AmazonEC2FullAccess --group-name EKSAdminGroup
$ aws iam attach-group-policy --policy-arn arn:aws-cn:iam::aws:policy/AWSCloudFormationFullAccess --group-name EKSAdminGroup
```

请注意，在上面的 arn 中，国内用的是 arn:aws-cn ，而不是官方文档所说的 arn:aws 。

然后还需要创建两个自定义的 policy ，首先是 EksAllAccess.json ：

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": "eks:*",
            "Resource": "*"
        },
        {
            "Action": [
                "ssm:GetParameter",
                "ssm:GetParameters"
            ],
            "Resource": [
                "arn:aws-cn:ssm:*:711111111110:parameter/aws/*",
                "arn:aws-cn:ssm:*::parameter/aws/*"
            ],
            "Effect": "Allow"
        },
        {
             "Action": [
               "kms:CreateGrant",
               "kms:DescribeKey"
             ],
             "Resource": "*",
             "Effect": "Allow"
        }
    ]
}
```

然后是 IamLimitedAccess.json :

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "iam:CreateInstanceProfile",
                "iam:DeleteInstanceProfile",
                "iam:GetInstanceProfile",
                "iam:RemoveRoleFromInstanceProfile",
                "iam:GetRole",
                "iam:CreateRole",
                "iam:DeleteRole",
                "iam:AttachRolePolicy",
                "iam:PutRolePolicy",
                "iam:ListInstanceProfiles",
                "iam:AddRoleToInstanceProfile",
                "iam:ListInstanceProfilesForRole",
                "iam:PassRole",
                "iam:DetachRolePolicy",
                "iam:DeleteRolePolicy",
                "iam:GetRolePolicy",
                "iam:GetOpenIDConnectProvider",
                "iam:CreateOpenIDConnectProvider",
                "iam:DeleteOpenIDConnectProvider",
                "iam:ListAttachedRolePolicies",
                "iam:TagRole"
            ],
            "Resource": [
                "arn:aws-cn:iam::711111111110:instance-profile/eksctl-*",
                "arn:aws-cn:iam::711111111110:role/eksctl-*",
                "arn:aws-cn:iam::711111111110:oidc-provider/*",
                "arn:aws-cn:iam::711111111110:role/aws-service-role/eks-nodegroup.amazonaws.com/AWSServiceRoleForAmazonEKSNodegroup",
                "arn:aws-cn:iam::711111111110:role/eksctl-managed-*"
            ]
        },
        {
            "Effect": "Allow",
            "Action": [
                "iam:GetRole"
            ],
            "Resource": [
                "arn:aws-cn:iam::711111111110:role/*"
            ]
        },
        {
            "Effect": "Allow",
            "Action": [
                "iam:CreateServiceLinkedRole"
            ],
            "Resource": "*",
            "Condition": {
                "StringEquals": {
                    "iam:AWSServiceName": [
                        "eks.amazonaws.com",
                        "eks-nodegroup.amazonaws.com",
                        "eks-fargate.amazonaws.com"
                    ]
                }
            }
        }
    ]
}

```

注意 711111111110 要替换成你们组织的 Account ID ， 还有 arn:aws 要 替换成 arn:aws-cn 。

创建 EksAllAccess 和 IamLimitedAccess 的 Policy: 

```shell
$ aws iam create-policy --policy-name EksAllAccess --policy-document file://EksAllAccess.json
{                                                                                                                                                                                                                    
    "Policy": {
        "PolicyName": "EksAllAccess",
        "PolicyId": "ANP111111111111111111M",
        "Arn": "arn:aws-cn:iam::711111111110:policy/EksAllAccess",
        "Path": "/",
        "DefaultVersionId": "v1",
        "AttachmentCount": 0,
        "PermissionsBoundaryUsageCount": 0,
        "IsAttachable": true,
        "CreateDate": "2020-12-10T02:32:49+00:00",
        "UpdateDate": "2020-12-10T02:32:49+00:00"
    }
}

$ aws iam create-policy --policy-name IamLimitedAccess --policy-document file://IamLimitedAccess.json
{
    "Policy": {
        "PolicyName": "IamLimitedAccess",
        "PolicyId": "AN111111111111111111A",
        "Arn": "arn:aws-cn:iam::711111111110:policy/IamLimitedAccess",
        "Path": "/",
        "DefaultVersionId": "v1",
        "AttachmentCount": 0,
        "PermissionsBoundaryUsageCount": 0,
        "IsAttachable": true,
        "CreateDate": "2020-12-10T03:15:30+00:00",
        "UpdateDate": "2020-12-10T03:15:30+00:00"
    }
}
```

然后附加 EksAllAccess 和 IamLimitedAccess 到 EKSAdminGroup 组 ：

```shell
$ aws iam attach-group-policy --policy-arn arn:aws-cn:iam::711111111110:policy/EksAllAccess --group-name EKSAdminGroup
$ aws iam attach-group-policy --policy-arn arn:aws-cn:iam::711111111110:policy/IamLimitedAccess --group-name EKSAdminGroup
```

## 为用户授权

执行以下命令，创建用户并添加到 EKSAdminGroup 组：

```shell
$ aws iam create-user --user-name someadmin
$ aws iam add-user-to-group --user-name someadmin --group-name EKSAdminGroup
```

然后这个用户就可以执行 eksctl 的所有命令了。

如果用户需要登录管理控制台，还需要为用户设置登录密码以及 iam:ChangePassword 权限。

通过 ```aws iam create-login-profile --cli-input-json file://create-login-profile.json``` 可以创建一个 login profile 文件，修改其内容匹配你的用户密码：

```json
{
    "UserName": "someadmin",
    "Password": "somepassword",
    "PasswordResetRequired": true
}
```

然后执行 ```aws iam create-login-profile --cli-input-json file://create-login-profile.json``` 即可完成密码设置。

但是只是设置了密码用户登陆时会提示需要修改密码，但是提交后又提示没有 iam:ChangePassword 权限，无法执行，所以还需要设置 IAMUserChangePassword 权限，可以通过以下命令完成：

```shell
$ aws iam attach-user-policy --policy-arn arn:aws-cn:iam::aws:policy/IAMUserChangePassword  --user-name someadmin
```

这样用户就可以登陆了，但是在控制台上访问之前的 EKS 集群时还是提示权限不足。折就需要把这个用户加到原来集群的管理组中，需要执行：

```
$ eksctl create iamidentitymapping --cluster old-cluster --arn arn:aws-cn:iam::711111111110:user/someadmin --group system:masters --username someadmin
```

## 参考

* [Minimum IAM policies for eksctl](https://eksctl.io/usage/minimum-iam-policies/)
* [Amazon EKS identity-based policy examples](https://docs.aws.amazon.com/eks/latest/userguide/security_iam_id-based-policy-examples.html)