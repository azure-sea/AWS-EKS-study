# 1. 创建EKS集群

## 1.1 通过 eksctl 创建集群

我们提供 `eksctl` 来创建集群

```shell
cat EKS_create.yml
```

```json
---
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: EKS-cluster03
  region: ap-southeast-2
  version: "1.19"

#vpc:
#  id: vpc-083b917d7766ce3d6
#  cidr: "172.25.0.0/16"
#  subnets:
#    private:
#      ap-southeast-1a:
#        id: subnet-0a01c9754dac81bca
#        cidr: "172.24.0.0/20"
#      ap-southeast-1b:
#        id: subnet-0aa4dfee396668a28
#        cidr: "172.24.16.0/20"
#      ap-southeast-1c:
#        id: subnet-0899d0471a6d3d13e
#        cidr: "172.24.32.0/20"
iam:
  withOIDC: true
  serviceAccounts:
  - metadata:
      name: s3-reader
      namespace: backend-apps
      labels: {aws-usage: "application"}
    attachPolicyARNs:
    - "arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess"
  - metadata:
      name: cache-access
      namespace: backend-apps
      labels: {aws-usage: "application"}
    attachPolicyARNs:
    - "arn:aws:iam::aws:policy/AmazonDynamoDBReadOnlyAccess"
    - "arn:aws:iam::aws:policy/AmazonElastiCacheFullAccess"
  - metadata:
      name: cluster-autoscaler
      namespace: kube-system
      labels: {aws-usage: "cluster-ops"}
    attachPolicy:
      Version: "2012-10-17"
      Statement:
      - Effect: Allow
        Action:
        - "autoscaling:DescribeAutoScalingGroups"
        - "autoscaling:DescribeAutoScalingInstances"
        - "autoscaling:DescribeLaunchConfigurations"
        - "autoscaling:DescribeTags"
        - "autoscaling:SetDesiredCapacity"
        - "autoscaling:TerminateInstanceInAutoScalingGroup"
        - "ec2:DescribeLaunchTemplateVersions"
        Resource: '*'
nodeGroups:
  - name: nodegroup-001
    instanceType: c5.large
    desiredCapacity: 1
    privateNetworking: true
    volumeSize: 100
    volumeType: gp2
    volumeEncrypted: true
    ssh: # use existing EC2 key
      allow: true
      publicKeyName: ustar-manage-key
#    securityGroups:
#      attachIDs: ["sg-09aaa2a9c9af68d8b", "sg-05bfbb3c6312a372a"]	

  - name: nodegroup-002
    instanceType: m5.large
    desiredCapacity: 1
    privateNetworking: true
    volumeSize: 100
    volumeType: gp2
    volumeEncrypted: true
    ssh: # use exsting EC2 key
      allow: true
      publicKeyName: ustar-manage-key
#    securityGroups:
#      attachIDs: ["sg-09aaa2a9c9af68d8b", "sg-05bfbb3c6312a372a"]
#  - name: nodegroup-003
#    tags:
      # EC2 tags required for cluster-autoscaler auto-discovery
#      k8s.io/cluster-autoscaler/enabled: "true"
#      k8s.io/cluster-autoscaler/cluster-13: "owned"
#    desiredCapacity: 1

cloudWatch:
  clusterLogging:
    # enable specific types of cluster control plane logs
    enableTypes: ["api", "audit", "authenticator", "controllerManager", "scheduler"]
    # all supported types: "api", "audit", "authenticator", "controllerManager", "scheduler"
    # supported special values: "*" and "all"
```

## 1.2 查看EKS集群工作节点

```
kubectl cluster-info
kubectl get node
```

![cluster-info](https://51k8s.oss-cn-shenzhen.aliyuncs.com/eks/images/cluster-info.png)



## 1.3 eksctl 命令介绍

1.  指定文件创建集群

```
eksctl create cluster -f EKS_create.yml
```

2. 指定名称与节点数量

```
eksctl create cluster --name=EKS-cluster03 --nodes=4
```

3. 指定版本创建

```
eksctl create cluster --version=1.19
```

4. 指定集群名称与节点数量范围

```
eksctl create cluster --name=EKS-cluster03 --nodes-min=3 --nodes-max=5
```

5. 删除集群

```
eksctl delete cluster -f EKS_create.yml
eksctl delete cluster --name=<name>
eksctl delete cluster --name EKS-cluster03 ##EKS-cluster03 集群名称
```

6. 查看集群信息

```
eksctl get cluster
eksctl get nodegroup --cluster=EKS-cluster03
```

![cluster-info2](https://51k8s.oss-cn-shenzhen.aliyuncs.com/eks/images/cluster-info2.png)

7. 创建nodegroup

```
eksctl create nodegroup --cluster=<clusterName>[--name=<nodegroupname>]
```

8. 列出所有 nodegroup

```
eksctl get nodegroup --cluster=<clustername>[--name=<nodegroupname>]
```

9. 伸缩 nodegroup

```
ekscli scale nodegroup --cluster=<clustername> --nodes=<desiredcount> --name=<nodegroupname>
```

10. 删除 nodegroup

```
eksctl delete nodegroup --cluster=<clustername> --name=<nodegroupname>
```

11. drain nodegroup

    如果nodegroup节点需要关机处理故障，此命令可以平稳的把nodegroup上面的节点自动迁移到其他nodegroup

```
eksctl drain nodegroup --cluster=<clustername> --name=<nodegroupname>
```

12. 升级控制平面

```
eksctl update cluster --name=<clustername>
```

13. 替换group，创建新的nodegroup

```
eksctl create nodegroup --cluster=<ClusterName> --name=<NewNodeGroupName>
```

## 1.4 部署一个测试应用 创建一个nginx.yaml

内容如下:

```shell
 cat << EOF >> nginx.yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: "service-nginx"
  annotations:
        service.beta.kubernetes.io/aws-load-balancer-type: nlb
spec:
  selector:
    app: nginx
  type: LoadBalancer
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
EOF
```

## 1.5 部署 nginx

````
kubectl apply -f nginx.yaml
kubectl get deploy
kubectl get svc
````

![测试创建一个nginx](https://51k8s.oss-cn-shenzhen.aliyuncs.com/eks/images/测试创建一个nginx.png)

查看部署结果

```
ELB=$(kubectl get svc service-nginx -o json |  jq -r '.status.loadBalancer.ingress[].hostname')
echo $ELB
curl $ELB
```

![测试nginx部署情况](https://51k8s.oss-cn-shenzhen.aliyuncs.com/eks/images/测试nginx部署情况.png)

清理测试环境

```
kubectl delete -f nginx.yaml
```

