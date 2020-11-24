# 配置 Windows 节点组

## 开启 Windows 支持

创建 cluster 时我们没有指定 --install-vpc-controllers 参数，所以我们需要首先安装 vpc controller :

```
eksctl utils install-vpc-controllers --cluster some-cluster --approve
```

## 增加 Windows 节点组

根据原来的配置创建 some-cluster-with-windows.yaml ：

```yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: some-cluster
  region: cn-northwest-1
  version: '1.18'

managedNodeGroups:
- name: mng-1
  instanceType: t3a.2xlarge
  minSize: 2
  maxSize: 2

nodeGroups:
- name: mng-win-1
  instanceType: t3a.large
  minSize: 2
  maxSize: 2
  amiFamily: WindowsServer2019FullContainer
```

nodeGroups 是新加的节点组，受管节点组仅支持 AmazonLinux2 ，所以这里是只能是非受管节点组（nodeGroups）。然后执行：

```
eksctl create nodegroup --config-file=some-cluster-with-windows.yaml
```

## 应用部署注意事项

部署需要添加一些配置，对于 linux pod 需要添加：

```yaml
    nodeSelector:
        kubernetes.io/os: linux
        kubernetes.io/arch: amd64
```

对于 Windows pod，需要添加：

```yaml
    nodeSelector:
        kubernetes.io/os: windows
        kubernetes.io/arch: amd64
```

## 测试应用

创建 windows-server-iis.yaml:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: windows-server-iis
spec:
  selector:
    matchLabels:
      app: windows-server-iis
      tier: backend
      track: stable
  replicas: 1
  template:
    metadata:
      labels:
        app: windows-server-iis
        tier: backend
        track: stable
    spec:
      containers:
      - name: windows-server-iis
        image: mcr.microsoft.com/windows/servercore:1809
        ports:
        - name: http
          containerPort: 80
        imagePullPolicy: IfNotPresent
        command:
        - powershell.exe
        - -command
        - "Add-WindowsFeature Web-Server; Invoke-WebRequest -UseBasicParsing -Uri 'https://dotnetbinaries.blob.core.windows.net/servicemonitor/2.0.1.6/ServiceMonitor.exe' -OutFile 'C:\\ServiceMonitor.exe'; echo '<html><body><br/><br/><marquee><H1>Hello EKS!!!<H1><marquee></body><html>' > C:\\inetpub\\wwwroot\\default.html; C:\\ServiceMonitor.exe 'w3svc'; "
      nodeSelector:
        kubernetes.io/os: windows
---
apiVersion: v1
kind: Service
metadata:
  name: windows-server-iis-service
  namespace: default
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: windows-server-iis
    tier: backend
    track: stable
  sessionAffinity: None
  type: ClusterIP

```

执行 ```kubectl apply -f windows-server-iis.yaml``` ，然后执行：

```
kubectl get pods -o wide --watch
```

查看效果。


## 参考

* [Windows support](https://docs.aws.amazon.com/eks/latest/userguide/windows-support.html)
* [Launching self-managed Windows nodes](https://docs.aws.amazon.com/eks/latest/userguide/launch-windows-workers.html)
* [windows nodes examples](https://github.com/weaveworks/eksctl/blob/master/examples/14-windows-nodes.yaml)