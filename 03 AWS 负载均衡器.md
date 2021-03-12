# 1. AWS 负载均衡器

## 1.1 AWS 负载均衡器介绍

AWS负载均衡器控制器为Kubernetes集群管理AWS弹性负载均衡器。控制器规定：

- 创建Kubernetes时的AWS应用程序负载均衡器（ALB）`Ingress`。
- 当您使用1.18或更高版本的Amazon EKS集群上的IP目标创建`Service`类型的Kubernetes时，将 `LoadBalancer`使用AWS Network Load Balancer（NLB）。如果您要对实例目标的网络流量进行负载均衡，则可以使用树内Kubernetes负载均衡器控制器，而无需安装此控制器。有关NLB目标类型的详细信息，请参阅《网络负载平衡器用户指南》中的“[目标类型](https://docs.aws.amazon.com/elasticloadbalancing/latest/network/load-balancer-target-groups.html#target-type)”。

该控制器以前称为*AWS ALB Ingress Controller*。这是一个[开源项目](https://github.com/kubernetes-sigs/aws-load-balancer-controller)在GitHub上进行管理。本主题可帮助您使用默认选项安装控制器。您可以查看完整的[文档](https://kubernetes-sigs.github.io/aws-load-balancer-controller/latest/)适用于GitHub上的控制器。部署控制器之前，我们建议您阅读的先决条件和注意事项[的应用负载上亚马逊EKS均衡](https://docs.aws.amazon.com/eks/latest/userguide/alb-ingress.html)和[网络负载在Amazon EKS平衡](https://docs.aws.amazon.com/eks/latest/userguide/load-balancing.html)。这些主题还包括部署示例应用程序的步骤，这些步骤要求控制器提供AWS资源。

## 1.2 部署 AWS Load Balancer控制器到Amazon EKS集群

在以下步骤中，用您自己的值替换`<example values>`

### 1. 确定您的群集是否有现有的IAM OIDC提供程序。

​        查看您群集的OIDC提供程序URL。

```
aws eks describe-cluster --name <cluster_name> --query "cluster.identity.oidc.issuer" --output text
```

​        输出示例：

```
https://oidc.eks.ap-southeast-2.amazonaws.com/id/F281AD9C01B223C09304173CC3105B59
```

​       列出您帐户中的IAM OIDC提供程序。用上一条命令返回的值替换 `<F281AD9C01B223C09304173CC3105B59>` 

```
aws iam list-open-id-connect-providers | grep <F281AD9C01B223C09304173CC3105B59>
```

​        输出示例

```
"Arn": "arn:aws:iam::111122223333:oidc-provider/oidc.eks.ap-southeast-2.amazonaws.com/id/F281AD9C01B223C09304173CC3105B59"
```

​        如果从上一条命令返回了输出，则说明您已经为群集提供了一个提供程序。如果未返回任何输出，则必须创建IAM OIDC提供程序。

​        要创建IAM OIDC提供程序，请参阅[为集群创建IAM OIDC提供程序](https://docs.aws.amazon.com/eks/latest/userguide/enable-iam-roles-for-service-accounts.html)。

### 2. 下载适用于AWS Load Balancer控制器的IAM策略，

使它可以代表您对AWS API进行调用。您可以查看[政策文件](https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2_ga/docs/install/iam_policy.json) 在GitHub上。

```
curl -o iam_policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.1.3/docs/install/iam_policy.json
```

### 3. 使用上一步中下载的策略创建IAM策略。

```
aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam_policy.json
```

​     记下返回的策略ARN。

### 4. 创建一个IAM角色 

并使用 eksctl 或AWS管理控制台和kubectl，为在AWS负载均衡器控制器`aws-load-balancer-controller`的`kube-system`名称空间中命名 的Kubernetes服务帐户添加注释

```
eksctl create iamserviceaccount \
  --cluster=my-cluster \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --attach-policy-arn=arn:aws:iam::<AWS_ACCOUNT_ID>:policy/AWSLoadBalancerControllerIAMPolicy \
  --override-existing-serviceaccounts \
  --approve
```

例如我的集群

```
eksctl create iamserviceaccount \
  --cluster=EKS-cluster03 \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --attach-policy-arn=arn:aws:iam::111122223333:policy/AWSLoadBalancerControllerIAMPolicy \
  --override-existing-serviceaccounts \
  --approve
```

### 5. 检测是否安装 AWS ALB 入口控制器

如果您当前已安装适用于 Kubernetes 的 AWS ALB 入口控制器，请卸载它。AWS Load Balancer 控制器取代了适用于 Kubernetes 的 AWS ALB 入口控制器的功能。

​          a. 检查当前是否安装了控制器。

```
kubectl get deployment -n kube-system alb-ingress-controller
```

​    输出（如果已安装）。跳至步骤 5b。

```
NAME                   READY UP-TO-DATE AVAILABLE AGE
alb-ingress-controller 1/1   1          1         122d
```

​    输出（如果尚未安装）。如果未安装，请跳至步骤 6。

```
Error from server (NotFound): deployments.apps "alb-ingress-controller" not found
```

​        b. 如果已安装控制器，请输入以下命令以将其删除。

```
kubectl delete -f https://raw.githubusercontent.com/kubernetes-sigs/aws-alb-ingress-controller/v1.1.8/docs/examples/alb-ingress-controller.yaml

kubectl delete -f https://raw.githubusercontent.com/kubernetes-sigs/aws-alb-ingress-controller/v1.1.8/docs/examples/rbac-role.yaml
```

​        c. 如果您删除了适用于 Kubernetes 的 AWS ALB 入口控制器，请将以下IAM策略添加到在步骤 4 中创建的 IAM 角色。该策略允许AWSLoad Balancer 控制器访问由适用于 Kubernetes 的 ALB 入口控制器创建的资源。

​            i. 下载策略IAM。您还可以[查看策略](https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/main/docs/install/iam_policy_v1_to_v2_additional.json)。

```
curl -o iam_policy_v1_to_v2_additional.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.1.0/docs/install/iam_policy_v1_to_v2_additional.json
```

​            ii. 创建IAM策略并记下返回的 ARN。

```
aws iam create-policy \
  --policy-name AWSLoadBalancerControllerAdditionalIAMPolicy \
  --policy-document file://iam_policy_v1_to_v2_additional.json
```

​            iii. 将IAM策略附加到您在步骤 4 中创建IAM的角色。将 `<your-role-name>` 替换为角色的名称。如果您已使用 创建角色`eksctl`，要查找已创建的角色名称，请打开 [AWS CloudFormation 控制台](https://console.aws.amazon.com//cloudformation)并选择 **eksctl-`<your-cluster-name>`-addon-iamserviceaccount-kube-system-aws-load-balancer-controller** 堆栈。选择 **Resources** 选项卡。角色名称位于 **Physical ID （物理** ID） 列中。如果您使用 AWS 管理控制台 创建角色，则角色名称是您命名的任何内容，例如 `AmazonEKSLoadBalancerControllerRole`。

```
aws iam attach-role-policy \
  --role-name eksctl-<your-role name>\
  --policy-arn arn:aws:iam::111122223333:policy/AWSLoadBalancerControllerAdditionalIAMPolicy
```

### 6. 使用以下方法之一安装AWSLoad Balancer控制器：

Helm v3 安装 AWSLoad Balancer控制器：

首先安装 helm

```
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
```

![helm安装](https://51k8s.oss-cn-shenzhen.aliyuncs.com/eks/images/helm安装.png)

a. 安装`TargetGroupBinding`自定义资源定义。

```
kubectl apply -k "github.com/aws/eks-charts/stable/aws-load-balancer-controller//crds?ref=master"
```

b. 添加`eks-charts`存储库。

```
helm repo add eks https://aws.github.io/eks-charts
```

c. 使用与您的集群所在的区域对应的命令安装 AWS Load Balancer 控制器。

> **重要**
>
> 如果您要将控制器部署到您Amazon EC2限制了从中访问[实例元数据服务 （IMDS）Amazon EC2 的](https://docs.aws.amazon.com/zh_cn/eks/latest/userguide/best-practices-security.html#restrict-ec2-credential-access)节点，或者要部署到 Fargate，则需要将以下标记添加到您运行的 命令：
>
> - `--set region=<region-code>`
> - `--set vpcId=<vpc-xxxxxxxx>`

```
helm upgrade -i aws-load-balancer-controller eks/aws-load-balancer-controller \
  --set clusterName=<cluster-name> \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  -n kube-system
```

### 7. 验证是否已安装控制器。

```
kubectl get deployment -n kube-system aws-load-balancer-controller
```

输出

```
NAME                           READY   UP-TO-DATE   AVAILABLE   AGE
aws-load-balancer-controller   1/1     1            1           70s
```

![ALB安装](https://51k8s.oss-cn-shenzhen.aliyuncs.com/eks/images/ALB安装.png)

### 8. 使用控制器预配置AWS资源

使用 控制器预配置AWS资源之前，您的集群必须满足特定要求。有关更多信息，请参阅 [上的应用程序负载均衡 Amazon EKS](https://docs.aws.amazon.com/zh_cn/eks/latest/userguide/alb-ingress.html) 和 [上的网络负载均衡 Amazon EKS](https://docs.aws.amazon.com/zh_cn/eks/latest/userguide/load-balancing.html).



# 2. 部署示例应用程序

您可以在仅具有Amazon EC2节点和/或 Fargate pod 的集群上运行示例应用程序。

1. 如果您要部署到 Fargate，请创建Fargate配置文件。如果您未部署以Fargate跳过此步骤。您可以通过运行以下命令创建配置文件，也可以使用 命令[中的 AWS 管理控制台 和 ](https://docs.aws.amazon.com/zh_cn/eks/latest/userguide/fargate-profile.html#create-fargate-profile)`name``namespace` 的相同值创建配置文件。

```
eksctl create fargateprofile --cluster <my-cluster> --region <region-code> --name <alb-sample-app> --namespace game-2048
```

2. 部署游戏 [2048](https://play2048.co/) 作为示例应用程序，以验证 AWS Load Balancer Controller 是否已创建 AWS ALB 作为入口对象的结果。

```
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.1.0/docs/examples/2048/2048_full.yaml
```

3. 几分钟后，验证是否已使用以下命令创建入口资源。

```
kubectl get ingress/ingress-2048 -n game-2048
```

​    输出：

```
NAME           CLASS    HOSTS   ADDRESS                                                                       PORTS   AGE
ingress-2048   <none>   *       k8s-game2048-ingress2-0d92478cc7-394946114.ap-southeast-2.elb.amazonaws.com   80      116s
```

> **注意**
>
> 如果您的入口在几分钟后尚未创建，请运行以下命令来查看Load Balancer控制器日志。这些日志包含可帮助您诊断部署中任何问题的错误消息。
>
> ```
> kubectl logs -n kube-system   deployment.apps/aws-load-balancer-controller
> ```

4. 打开浏览器并从上一命令输出导航到 `ADDRESS` URL 以查看示例应用程序。如果您没有看到任何内容，请等待几分钟并刷新浏览器。

   ![game_2048](https://51k8s.oss-cn-shenzhen.aliyuncs.com/eks/images/game_2048.png)

5. 在完成对示例应用程序的试验后，请使用以下命令将其删除。

```
kubectl delete -f https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.1.0/docs/examples/2048/2048_full.yaml
```

# 3. AWS LoadBalancer控制器 后续补充

## 3.1 Ingress annotations 介绍

[AWS Ingress annotations](https://kubernetes-sigs.github.io/aws-load-balancer-controller/latest/guide/ingress/annotations/) 介绍

可以向kubernetes入口和服务对象添加注释，以自定义它们的行为。

- Annotation 键和值只能是字符串。高级格式应如下编码：:
  - boolean: 'true'
  -  integer: '42'
  - stringList: s1,s2,s3
  - stringMap: k1=v1,k2=v2
  - json: 'jsonContent'

- Annotations applied to Service have higher priority over annotations applied to Ingress. `Location `

  column below indicates where that annotation can be applied to.

- Annotations that configures LoadBalancer / Listener behaviors have different merge behavior when IngressGroup feature is been used. `MergeBehavior` column below indicates how such annotation will be merged.
  - Exclusive: 此类注释仅应在IngressGroup内的单个Ingress上指定，或在IngressGroup内的所有Ingress上指定相同的值。
  - Merge: 可以在IngressGroup中的所有Ingress上指定此类注释，并将其合并在一起。



### 3.1.1 指定LoadBalancer是连接互联网

`alb.ingress.kubernetes.io/scheme`指定您的LoadBalancer是否面向互联网

internal:  连接互联网, 此值为默认值。

internet-facing:  内网负载均衡器。