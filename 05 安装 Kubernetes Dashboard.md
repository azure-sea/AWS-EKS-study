# 部署 Kubernetes 控制面板 (Web UI)

下面介绍将 [Kubernetes 控制面板](https://github.com/kubernetes/dashboard)部署到Amazon EKS集群的过程，包括 CPU 和内存指标。它还帮助您创建可用于安全地连接到控制面板以查看和控制集群的 Amazon EKS 管理员服务账户。

![kubernetes-dashboard](https://51k8s.oss-cn-shenzhen.aliyuncs.com/oss-cn-shenzhenkubernetes-dashboard.png)



前提条件

- 您已通过执行 Amazon EKS中的步骤创建了一个 [开始使用 Amazon EKS](https://docs.aws.amazon.com/zh_cn/eks/latest/userguide/getting-started.html). 集群。
- 您已安装 Kubernetes Metrics Server。有关更多信息，请参阅[安装 Kubernetes Metrics Server](https://docs.aws.amazon.com/zh_cn/eks/latest/userguide/metrics-server.html).

## 1.1 部署 Kubernetes 控制面板

1.19 版本的EKS 使用 dashboard v2.0.5 具体情况参考官方 [releases日志](https://github.com/kubernetes/dashboard/releases)

```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.5/aio/deploy/recommended.yaml
```

![dashboard-release](https://51k8s.oss-cn-shenzhen.aliyuncs.com/oss-cn-shenzhendashboard-release.png)



输出：

> namespace/kubernetes-dashboard created
> serviceaccount/kubernetes-dashboard created
> service/kubernetes-dashboard created
> secret/kubernetes-dashboard-certs created
> secret/kubernetes-dashboard-csrf created
> secret/kubernetes-dashboard-key-holder created
> configmap/kubernetes-dashboard-settings created
> role.rbac.authorization.k8s.io/kubernetes-dashboard created
> clusterrole.rbac.authorization.k8s.io/kubernetes-dashboard created
> rolebinding.rbac.authorization.k8s.io/kubernetes-dashboard created
> clusterrolebinding.rbac.authorization.k8s.io/kubernetes-dashboard created
> deployment.apps/kubernetes-dashboard created
> service/dashboard-metrics-scraper created
> deployment.apps/dashboard-metrics-scraper created

## 1.2 创建 `eks-admin` 服务账户和集群角色绑定

默认情况下，Kubernetes 控制面板用户的权限是有限的。在此部分中，您创建一个可用于使用管理员级别权限安全地连接到控制面板的 `eks-admin` 服务账户和集群角色绑定。有关更多信息，请参阅 Kubernetes 文档中的[管理服务账户](https://kubernetes.io/docs/admin/service-accounts-admin/)。

**创建 `eks-admin` 服务账户和集群角色绑定**

> **重要**
>
> 使用此过程创建的示例服务账户在集群上具有完整的 `cluster-admin` (超级用户) 特权。有关更多信息，请参阅 Kubernetes 文档中的[使用 RBAC 授权](https://kubernetes.io/docs/admin/authorization/rbac/)。

1. 使用以下文本创建一个名为 `eks-admin-service-account.yaml` 的文件。此清单定义一个名为 的服务账户和集群角色绑定。`eks-admin`.

```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: eks-admin
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: eks-admin
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: eks-admin
  namespace: kube-system
```

2. 将此服务账户和集群角色绑定应用到您的集群。

```
kubectl apply -f eks-admin-service-account.yaml
```

输出：

```
serviceaccount "eks-admin" created
clusterrolebinding.rbac.authorization.k8s.io "eks-admin" created
```

注意

> apiVersion: rbac.authorization.k8s.io/v1
>
> 这个在v1.17版本已经被弃用，在v1.22+中不可用 所有我们使用rbac.authorization.k8s.io/v1



## 1.3 连接到控制面板

现在，已将 Kubernetes 控制面板部署到集群，并且您已具有可用于查看和控制集群的管理员服务账户，您可使用该服务账户连接到控制面板。

**连接到 Kubernetes 控制面板**

1. 检索 `eks-admin` 服务账户的身份验证令牌。从输出中复制 `<authentication_token>` 值。您可以使用此令牌连接到控制面板。

```
kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep eks-admin | awk '{print $1}')
```

输出：

```
Name:         eks-admin-token-cttrz
Namespace:    kube-system
Labels:       <none>
Annotations:  kubernetes.io/service-account.name: eks-admin
              kubernetes.io/service-account.uid: 8058a0f5-3fd9-4543-986a-403bd0660de2

Type:  kubernetes.io/service-account-token

Data
====
ca.crt:     1066 bytes
namespace:  11 bytes
token:     <authentication_token>
```

由于EKS在云上。我们想要访问的话，一般通过启用`kubectl proxy` 或者通过云上的负载均衡器

2. **下面我们就使用nlb实现**

```
wget https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.5/aio/deploy/recommended.yaml
```

```
vim recommended.yaml
```

修改下面部分内容

```
kind: Service
apiVersion: v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kubernetes-dashboard
  annotations:  # 增加的配置
      service.beta.kubernetes.io/aws-load-balancer-type: nlb  # 增加的配置
spec:
  ports:
    - port: 443
      targetPort: 8443
  selector:
    k8s-app: kubernetes-dashboard
  type: LoadBalancer     # 增加的配置
```

3. **重新应用一下**

```
kubectl apply -f recommended.yaml
```

查看一下svc

```
kubectl get svc -n kubernetes-dashboard
```

> NAME                        TYPE           CLUSTER-IP      EXTERNAL-IP                                                                          PORT(S)         AGE
> dashboard-metrics-scraper   ClusterIP      10.100.40.38    <none>                                                                               8000/TCP        21m
> kubernetes-dashboard        LoadBalancer   10.100.241.16   a122ef5e378334478b8f56aa18fc8d31-e2cff617634429b4.elb.ap-southeast-2.amazonaws.com   443:31636/TCP   21m

或者

```
ELB=$(kubectl get svc kubernetes-dashboard -n kubernetes-dashboard -o json |  jq -r '.status.loadBalancer.ingress[].hostname')
echo $ELB
```

```
a122ef5e378334478b8f56aa18fc8d31-e2cff617634429b4.elb.ap-southeast-2.amazonaws.com
```

4. **登录控制面板**

https://a122ef5e378334478b8f56aa18fc8d31-e2cff617634429b4.elb.ap-southeast-2.amazonaws.com/

![dashboard-login](https://51k8s.oss-cn-shenzhen.aliyuncs.com/oss-cn-shenzhendashboard-login.png)

选择 **Token** （令牌），将上一个命令`<authentication_token>`的输出粘贴到 **Token** （令牌） 字段中，然后选择 **SIGN IN** （登录）

5. 后续步骤

在连接到 Kubernetes 控制面板之后，可使用 `eks-admin` 服务账户查看和控制集群。有关使用控制面板的更多信息，请参阅 上的[GitHub项目文档](https://github.com/kubernetes/dashboard)

