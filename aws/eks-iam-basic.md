# IAM 快速入门

## 创建组和用户

一般情况下应该为一类用户创建一个组：

```shell
$ aws iam create-group --group-name GfsGroup
{
    "Group": {
        "Path": "/",
        "GroupName": "GfsGroup",
        "GroupId": "AGPA3O2E41111S4PIWM5K",
        "Arn": "arn:aws-cn:iam::11111111:group/GfsGroup",
        "CreateDate": "2020-12-07T04:06:14+00:00"
    }
}
```

创建一个用户，并关联到组：

```shell
$ aws iam create-user --user-name GfsUser
{
    "User": {
        "Path": "/",
        "UserName": "GfsUser",
        "UserId": "AIDA3O2E411111ZRQ7OTN",
        "Arn": "arn:aws-cn:iam::111111111:user/GfsUser",
        "CreateDate": "2020-12-07T04:24:28+00:00"
    }
}

$ aws iam add-user-to-group --user-name GfsUser --group-name GfsGroup

$ aws iam get-group --group-name GfsGroup
{
    "Users": [
        {
            "Path": "/",
            "UserName": "GfsUser",
            "UserId": "AIDA3O2E411111ZRQ7OTN",
            "Arn": "arn:aws-cn:iam::111111111:user/GfsUser",
            "CreateDate": "2020-12-07T04:24:28+00:00"
        }
    ],
    "Group": {
        "Path": "/",
        "GroupName": "GfsGroup",
        "GroupId": "AGPA3O2E41111S4PIWM5K",
        "Arn": "arn:aws-cn:iam::11111111:group/GfsGroup",
        "CreateDate": "2020-12-07T04:06:14+00:00"
    }
}

```


## 创建管理员用户和组

创建一个 EKS 管理员组：

```shell
$ aws iam create-group --group-name EKSAdmin
{
    "Group": {
        "Path": "/",
        "GroupName": "EKSAdmin",
        "GroupId": "AGPA3O2E1111MRUO4Z6QP",
        "Arn": "arn:aws-cn:iam::111111111:group/EKSAdmin",
        "CreateDate": "2020-12-07T05:36:36+00:00"
    }
}
```

创建一个管理员 Policy：





创建一个用户，并关联到组：

```shell
$ aws iam create-user --user-name sundaijun
{
    "User": {
        "Path": "/",
        "UserName": "sundaijun",
        "UserId": "AIDA3O2E4JFKPTV4OFTZX",
        "Arn": "arn:aws-cn:iam::787734088020:user/sundaijun",
        "CreateDate": "2020-12-07T05:37:09+00:00"
    }
}

$ aws iam add-user-to-group --user-name sundaijun --group-name EKSAdmin
```








## 参考


* [Example IAM identity-based policies](https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies_examples.html)
* [Using AWS Identity and Access Management from the AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/cli-services-iam.html)
* [Security best practices in IAM](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html)