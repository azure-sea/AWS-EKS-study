# 安装 Kubernetes Metrics Server

Kubernetes Metrics Server 是集群中资源使用情况数据的聚合器，它在 Amazon EKS 集群中默认不部署。有关更多信息，请参阅 上的 [Kubernetes Metrics](https://github.com/kubernetes-sigs/metrics-server) GitHubServer。 Metrics Server 通常由其他 Kubernetes 添加对象使用，例如 [Horizontal Pod Autoscaler](https://docs.aws.amazon.com/zh_cn/eks/latest/userguide/horizontal-pod-autoscaler.html) 或 [Kubernetes 控制面板](https://docs.aws.amazon.com/zh_cn/eks/latest/userguide/dashboard-tutorial.html)。有关更多信息，请参阅 Kubernetes 文档中的[资源指标管道](https://kubernetes.io/docs/tasks/debug-application-cluster/resource-metrics-pipeline/)。下面介绍了如何在 Amazon EKS 集群上部署 Kubernetes Metrics Server。

## 1.1 部署 Metrics Server

1. 使用以下命令部署 Metrics Server：

```
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

2. 使用以下命令验证 `metrics-server` 部署是否运行所需数量的 Pod：

```
kubectl get deployment metrics-server -n kube-system
```

​    输出

```
NAME             READY   UP-TO-DATE   AVAILABLE   AGE
metrics-server   1/1     1            1           53s
```







