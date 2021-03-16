# 1. Pod 的弹性伸缩介绍

​        使用Pod 中我们可能会考虑对Pod进行资源的限制, 但是一但在流量高峰时期。pod需要的资源可能超过设置的资源限制，这时我们会考虑将pod资源限制扩大,或者在启动一个pod去承载这个部分超额流量。但是在手动实现的过程考虑需要时间和人力响应速度。 下面就介绍一下 K8s 如何实现自动伸缩。

## 1.1 Vertical Pod Autoscaler (VPA) 介绍

​        Kubernetes [Vertical Pod Autoscaler](https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler) 自动调整 Pod 的 CPU 和内存预留，以帮助“正确大小”您的应用程序。此调整可以帮助提高集群资源利用率并释放 CPU 和内存供其他 Pod 使用。

前提条件:

- 需要一个现有的  Amazon EKS 集群。

- 您已安装 Kubernetes Metrics Server。有关更多信息，请参阅[安装 Kubernetes Metrics Server](https://github.com/azure-sea/AWS-EKS-study/blob/master/04%20%E5%AE%89%E8%A3%85%20Metrics%20Server.md).

- 使用的是`kubectl`配置的 [集群Amazon EKS进行通信的](https://docs.aws.amazon.com/zh_cn/eks/latest/userguide/getting-started-console.html#eks-configure-kubectl)客户端。

  ![vpa-architecture](https://51k8s.oss-cn-shenzhen.aliyuncs.com/eks/images/vpa-architecture.png)

### 1.1.1 部署 Vertical Pod Autoscaler

**部署 Vertical Pod Autoscaler**

1. 打开终端窗口，导航到您要下载 Vertical Pod Autoscaler 源代码的目录。

2. 克隆 [kubernetes/autoscaler](https://github.com/kubernetes/autoscaler) GitHub 存储库。

   ```
   git clone https://github.com/kubernetes/autoscaler.git
   ```

3. 切换到 `vertical-pod-autoscaler` 目录。

   ```
   cd autoscaler/vertical-pod-autoscaler/
   ```

4. （可选）如果您已经部署另一个版本的 Vertical Pod Autoscaler，请使用以下命令将其删除。

   ```
   ./hack/vpa-down.sh
   ```

5. 使用以下命令将 Vertical Pod Autoscaler 部署到您的集群。

```
./hack/vpa-up.sh
```

6. 验证已成功创建 Vertical Pod Autoscaler Pod。

```
kubectl get pods -n kube-system
```

输出：

```
NAME                                        READY   STATUS    RESTARTS   AGE
aws-node-949vx                              1/1     Running   0          122m
aws-node-b4nj8                              1/1     Running   0          122m
coredns-6c75b69b98-r9x68                    1/1     Running   0          133m
coredns-6c75b69b98-rt9bp                    1/1     Running   0          133m
kube-proxy-bkm6b                            1/1     Running   0          122m
kube-proxy-hpqm2                            1/1     Running   0          122m
metrics-server-8459fc497-kfj8w              1/1     Running   0          83m
vpa-admission-controller-68c748777d-ppspd   1/1     Running   0          7s
vpa-recommender-6fc8c67d85-gljpl            1/1     Running   0          8s
vpa-updater-786b96955c-bgp9d                1/1     Running   0          8s
```

### 1.1.2 测试Vertical Pod Autoscaler 安装

**测试 Vertical Pod Autoscaler 安装**

1. 使用以下命令部署 `hamster.yaml` Vertical Pod Autoscaler 示例。

   ```
   kubectl apply -f examples/hamster.yaml
   ```

2. 从 `hamster` 示例应用程序获取 Pod。

   ```
   kubectl get pods -l app=hamster
   ```

   输出：

   ```
   hamster-c7d89d6db-rglf5   1/1     Running   0          48s
   hamster-c7d89d6db-znvz5   1/1     Running   0          48s
   ```

3. 查看一个 Pod 以查看其 CPU 和内存预留。

   ```
   kubectl describe pod hamster-<c7d89d6db-rglf5>
   ```

   输出：

   ```
   Name:           hamster-c7d89d6db-rglf5
   Namespace:      default
   Priority:       0
   Node:           ip-192-168-9-44.<region-code>.compute.internal/192.168.9.44
   Start Time:     Fri, 27 Sep 2019 10:35:15 -0700
   Labels:         app=hamster
                   pod-template-hash=c7d89d6db
   Annotations:    kubernetes.io/psp: eks.privileged
                   vpaUpdates: Pod resources updated by hamster-vpa: container 0:
   Status:         Running
   IP:             192.168.23.42
   IPs:            <none>
   Controlled By:  ReplicaSet/hamster-c7d89d6db
   Containers:
     hamster:
       Container ID:  docker://e76c2413fc720ac395c33b64588c82094fc8e5d590e373d5f818f3978f577e24
       Image:         k8s.gcr.io/ubuntu-slim:0.1
       Image ID:      docker-pullable://k8s.gcr.io/ubuntu-slim@sha256:b6f8c3885f5880a4f1a7cf717c07242eb4858fdd5a84b5ffe35b1cf680ea17b1
       Port:          <none>
       Host Port:     <none>
       Command:
         /bin/sh
       Args:
         -c
         while true; do timeout 0.5s yes >/dev/null; sleep 0.5s; done
       State:          Running
         Started:      Fri, 27 Sep 2019 10:35:16 -0700
       Ready:          True
       Restart Count:  0
       Requests:
         cpu:        100m
         memory:     50Mi
   ...
   ```

   您可以看到原始 Pod 预留了 100 millicpu 的 CPU 和 50 MB 的内存。对于本示例应用程序，100 millicpu 小于 Pod 运行所需的数量，因此 CPU 受限。它预留的内存也远小于所需的数量。Vertical Pod Autoscaler `vpa-recommender` 部署分析 `hamster` Pod 以查看 CPU 和内存需求是否合适。如果需要调整，`vpa-updater` 使用更新后的值重新启动 Pod。

4. 等待 `vpa-updater` 启动新 `hamster` Pod。这大概需要一两分钟。您可以使用以下命令监控 Pod。

   **注意**

   如果您不确定已经启动了新 Pod，请将 Pod 名称与您之前的列表比较。新 Pod 启动时，您会看到新 Pod 名称。

   ```
   kubectl get --watch pods -l app=hamster
   ```

5. 当新 `hamster` Pod 启动时，描述它并查看更新后的 CPU 和内存预留。

   ```
   kubectl describe pod hamster-<c7d89d6db-jxgfv>
   ```

   输出：

   ```
   Name:           hamster-c7d89d6db-jxgfv
   Namespace:      default
   Priority:       0
   Node:           ip-192-168-9-44.<region-code>.compute.internal/192.168.9.44
   Start Time:     Fri, 27 Sep 2019 10:37:08 -0700
   Labels:         app=hamster
                   pod-template-hash=c7d89d6db
   Annotations:    kubernetes.io/psp: eks.privileged
                   vpaUpdates: Pod resources updated by hamster-vpa: container 0: cpu request, memory request
   Status:         Running
   IP:             192.168.3.140
   IPs:            <none>
   Controlled By:  ReplicaSet/hamster-c7d89d6db
   Containers:
     hamster:
       Container ID:  docker://2c3e7b6fb7ce0d8c86444334df654af6fb3fc88aad4c5d710eac3b1e7c58f7db
       Image:         k8s.gcr.io/ubuntu-slim:0.1
       Image ID:      docker-pullable://k8s.gcr.io/ubuntu-slim@sha256:b6f8c3885f5880a4f1a7cf717c07242eb4858fdd5a84b5ffe35b1cf680ea17b1
       Port:          <none>
       Host Port:     <none>
       Command:
         /bin/sh
       Args:
         -c
         while true; do timeout 0.5s yes >/dev/null; sleep 0.5s; done
       State:          Running
         Started:      Fri, 27 Sep 2019 10:37:08 -0700
       Ready:          True
       Restart Count:  0
       Requests:
         cpu:        587m
         memory:     262144k
   ...
   ```

   此时可以看到 CPU 预留提升到了 587 个 millicpu，这是原始值的五倍多。内存已增加到 262144 KB，即大约 250 MB，即原始值的五倍。此 Pod 资源不足，Vertical Pod Autoscaler 使用更为合适的值纠正了我们的估计值。

6. 描述 `hamster-vpa` 资源以查看新的建议。

   ```
   kubectl describe vpa/hamster-vpa
   ```

   输出：

   ```
   Name:         hamster-vpa
   Namespace:    default
   Labels:       <none>
   Annotations:  kubectl.kubernetes.io/last-applied-configuration:
                   {"apiVersion":"autoscaling.k8s.io/v1beta2","kind":"VerticalPodAutoscaler","metadata":{"annotations":{},"name":"hamster-vpa","namespace":"d...
   API Version:  autoscaling.k8s.io/v1beta2
   Kind:         VerticalPodAutoscaler
   Metadata:
     Creation Timestamp:  2019-09-27T18:22:51Z
     Generation:          23
     Resource Version:    14411
     Self Link:           /apis/autoscaling.k8s.io/v1beta2/namespaces/default/verticalpodautoscalers/hamster-vpa
     UID:                 d0d85fb9-e153-11e9-ae53-0205785d75b0
   Spec:
     Target Ref:
       API Version:  apps/v1
       Kind:         Deployment
       Name:         hamster
   Status:
     Conditions:
       Last Transition Time:  2019-09-27T18:23:28Z
       Status:                True
       Type:                  RecommendationProvided
     Recommendation:
       Container Recommendations:
         Container Name:  hamster
         Lower Bound:
           Cpu:     550m
           Memory:  262144k
         Target:
           Cpu:     587m
           Memory:  262144k
         Uncapped Target:
           Cpu:     587m
           Memory:  262144k
         Upper Bound:
           Cpu:     21147m
           Memory:  387863636
   Events:          <none>
   ```

7. 在完成对示例应用程序的试验后，使用以下命令可将其删除。

   ```
   kubectl delete -f examples/hamster.yaml
   ```

### 1.1.3 Vertical Pod Autoscaler 注意事项

从上面实验我们可以看到 `Vertical Pod Autoscaler` 的实现伸缩的过程。

那我们应该注意 `Vertical Pod Autoscaler` 的一些使用限制

- 更新正在运行的Pod是VPA的一项功能，每当VPA更新Pod资源时，都会重新创建Pod，这会导致所有正在运行的容器重新启动。可以在其他节点上重新创建Pod。这个也就意味着我们需要配置良好的探针。
- VPA对大多数内存不足事件做出反应，但并非在所有情况下都做出反应
- VPA建议可能会超出可用资源（例如，节点大小，可用大小，可用配额），并导致**Pod处于待处理状态**。所以这个需要集群中拥有足够的资源用于Pod 的调度。与[Cluster Autoscaler](https://github.com/kubernetes/autoscaler/blob/master/cluster-autoscaler/FAQ.md#basics)一起使用，可以部分解决此问题。
- 目前，**不得在CPU或内存上将**Vertical Pod Autoscaler 与 [水平Pod Autoscaler](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)（HPA）一起使用。



## 1.2 Horizontal Pod Autoscaler 介绍

​        Kubernetes [Horizontal Pod Autoscaler](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/) 根据资源的 CPU 利用率自动扩展部署、复制控制器或副本集中的 Pod 数量。这可帮助您的应用程序进行扩展以满足增长的需求，或在不需要资源时进行缩减，从而为其他应用程序释放节点。当您设置目标 CPU 利用率百分比时，Horizontal Pod Autoscaler 扩展或缩减应用程序来尝试满足该目标。

​        Horizontal Pod Autoscaler 是 Kubernetes 中的标准 API 资源，只需在 Amazon EKS 集群上安装一个指标源（如 Kubernetes Metrics Server）即可正常运行。您不需要在集群上部署或安装 Horizontal Pod Autoscaler 以开始扩展您的应用程序。有关更多信息，请参阅 Kubernetes 文档中的 [Horizontal Pod](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/) Autoscaler。

### 1.2.1 测试 Horizontal Pod Autoscaler 安装

1. 使用以下命令部署一个简单的 Apache Web 服务器应用程序。

   ```
   kubectl apply -f https://k8s.io/examples/application/php-apache.yaml
   ```

   向此 Apache Web 服务器 Pod 提供 500 millicpu 的 CPU 限制，并在端口 80 上提供服务。

2. 为 `php-apache` 部署创建 Horizontal Pod Autoscaler 资源。

   ```
   kubectl autoscale deployment php-apache --cpu-percent=50 --min=1 --max=10
   ```

   此命令创建为部署定位 50% CPU 利用率的 Autoscaler，最少一个 Pod，最多十个 Pod。当平均 CPU 负载低于 50 时，Autoscaler 尝试减少部署中的 Pod 数量，最低一个。当负载大于 50% 时，Autoscaler 尝试增加部署中的 Pod 数量，最高十个。有关更多信息，请参阅 Kubernetes 文档中的 Horizontal Pod Autoscaler [的工作原理。](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/#how-does-the-horizontal-pod-autoscaler-work)

3. 使用以下命令描述 Autoscaler 以查看其详细信息。

   ```
   kubectl describe hpa
   ```

   输出：

   ```
   Name:                                                  php-apache
   Namespace:                                             default
   Labels:                                                <none>
   Annotations:                                           <none>
   CreationTimestamp:                                     Thu, 11 Jun 2020 16:05:41 -0500
   Reference:                                             Deployment/php-apache
   Metrics:                                               ( current / target )
     resource cpu on pods  (as a percentage of request):  <unknown> / 50%
   Min replicas:                                          1
   Max replicas:                                          10
   Deployment pods:                                       1 current / 0 desired
   Conditions:
     Type           Status  Reason                   Message
     ----           ------  ------                   -------
     AbleToScale    True    SucceededGetScale        the HPA controller was able to get the target's current scale
     ScalingActive  False   FailedGetResourceMetric  the HPA was unable to compute the replica count: did not receive metrics for any ready pods
   Events:
     Type     Reason                        Age                From                       Message
     ----     ------                        ----               ----                       -------
     Warning  FailedGetResourceMetric       42s (x2 over 57s)  horizontal-pod-autoscaler  unable to get metrics for resource cpu: no metrics returned from resource metrics API
     Warning  FailedComputeMetricsReplicas  42s (x2 over 57s)  horizontal-pod-autoscaler  invalid metrics (1 invalid out of 1), first error is: failed to get cpu utilization: unable to get metrics for resource cpu: no metrics returned from resource metrics API
     Warning  FailedGetResourceMetric       12s (x2 over 27s)  horizontal-pod-autoscaler  did not receive metrics for any ready pods
     Warning  FailedComputeMetricsReplicas  12s (x2 over 27s)  horizontal-pod-autoscaler  invalid metrics (1 invalid out of 1), first error is: failed to get cpu utilization: did not receive metrics for any ready pods
   ```

   如您所见，当前 CPU 负载是 `<unknown>`，因为服务器上尚没有负载。pod 计数已处于其最低边界 (1)，因此无法缩减。

4. 通过运行容器为 Web 服务器创建负载。

   ```
   kubectl run -it --rm load-generator --image=busybox /bin/sh --generator=run-pod/v1
   ```

   如果您在几秒钟后未收到命令提示符，则可能需要按 `Enter`。 在命令提示符处，输入以下命令以生成负载并促使 Autoscaler 扩展部署。

   ```
   while true; do wget -q -O- http://php-apache; done
   ```

5. 要监视部署的向外扩展情况，请在与执行上一步骤的终端不同的终端上定期运行以下命令。

   ```
   kubectl get hpa
   ```

   输出：

   ```
   NAME         REFERENCE               TARGETS    MINPODS   MAXPODS   REPLICAS   AGE
   php-apache   Deployment/php-apache   250%/50%   1         10        5          4m44s
   ```

   只要实际 CPU 百分比高于目标百分比，副本计数就会增加（最大值为 10）。在此情况下，此百分比为 `250%`，因此 `REPLICAS` 数会继续增加。

   **注意**

   可能需要在几分钟后，您才能看到副本计数达到其最大值。例如，如果只需 6 个副本即可让 CPU 负载小于或等于 50％，则负载将不会超过 6 个副本。

6. 停止负载。在正在生成负载（从步骤 4）的终端窗口中，通过按住 `Ctrl+C` 键来停止负载。通过再次运行以下命令，您会看到副本数缩减回 1。

   ```
   kubectl get hpa
   ```

   输出

   ```
   NAME         REFERENCE               TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
   php-apache   Deployment/php-apache   0%/50%    1         10        1          25m
   ```

   **注意**

   缩减的默认时间范围是 5 分钟，因此，在您再次看到副本计数达到 1 之前，即使当前 CPU 百分比为 0%，也可能需要一些时间。时间范围是可修改的。有关更多信息，请参阅 Kubernetes 文档中的 [Horizontal Pod](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/) Autoscaler。

7. 完成示例应用程序的试验之后，删除 `php-apache` 资源。

   ```
   kubectl delete deployment.apps/php-apache service/php-apache horizontalpodautoscaler.autoscaling/php-apache
   ```