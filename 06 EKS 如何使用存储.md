# 1. <span id='Storage'>Storage</span>

下面介绍了 Amazon EKS 集群的存储选项

- [Storage classes](#storage classes) 
- [Amazon EBS CSI 驱动程序](#Amazon EBS CSI 驱动程序)
- [Amazon EFS CSI 驱动程序](#Amazon EFS CSI 驱动程序)

> **注意:**
>
> 现在[in-tree Amazon EBS plugin](https://kubernetes.io/docs/concepts/storage/volumes/#awselasticblockstore) 仍然可以使用，但是通过使用CSI 驱动程序，您可以从 Kubernetes 上游发布周期与 CSI 驱动程序发布周期的分离中受益。最终，[in-tree Amazon EBS plugin](https://kubernetes.io/docs/concepts/storage/volumes/#awselasticblockstore) 将停用，用以支持 CSI 驱动程序



**awsElasticBlockStore**

下面简单介绍下 `awsElasticBlockStore`

 一个`awsElasticBlockStore`卷将Amazon Web Services（AWS） [EBS卷](https://aws.amazon.com/ebs/)装载到您的Pod中。在移除pod时 `emptyDir` 中数据将被清除 ，而EBS卷的内容将保留并卸载。这意味着可以用数据预填充EBS卷，并且可以在Pod之间共享数据。

> 注意点:
>
> 必须先使用aws ec2 create-volume或AWS 控制台创建EBS卷，然后才能使用它。

使用`awsElasticBlockStore`卷时有一些限制：

- 运行Pod的节点必须是AWS EC2实例
- 这些实例需要与EBS卷位于相同的区域和可用区中
- EBS的一个卷只能绑定一个EC2实例 (无法多个ec2实例绑定一个EBS卷)

创建一个AWS EBS卷

```shell
aws ec2 create-volume --availability-zone=ap-southeast-2a --size=10 --volume-type=gp2 --tag-specifications 'ResourceType=volume,Tags=[{Key=Name,Value=EKS_EBS_SIZE-10G},]'
```

确保该区域与您在其中引入群集的区域匹配。并请检查大小和EBS卷类型是否适合您的使用。

![EKS_EBS_SIZE-10G](https://51k8s.oss-cn-shenzhen.aliyuncs.com/oss-cn-shenzhenEKS_EBS_SIZE-10G.png)

在控制台平面上也能够看到我们创建的 EBS

![AWS-EBS-10G](https://51k8s.oss-cn-shenzhen.aliyuncs.com/oss-cn-shenzhenAWS-EBS-10G.png)

通过AWS cli 创建EBS各参数可参考下面地址

https://docs.aws.amazon.com/cli/latest/reference/ec2/create-volume.html

**AWS EBS 配置示例**

```
apiVersion: v1
kind: Pod
metadata:
  name: test-ebs
spec:
  containers:
  - image: k8s.gcr.io/test-webserver
    name: test-container
    volumeMounts:
    - mountPath: /test-ebs
      name: test-volume
  volumes:
  - name: test-volume
    # This AWS EBS volume must already exist.
    awsElasticBlockStore:
      volumeID: "<volume id>"
      fsType: ext4
```

如果EBS卷已分区，则可以提供可选字段`partition: "<partition number>"`以指定要安装在哪个分区上。

## 1.1 <span id='storage classes'>Storage classes</span>

Amazon EKS在 Kubernetes 版本 1.11 之前创建的 集群未创建有任何存储类。您必须为要使用的集群定义 `storage-classes`，并且应为持久卷声明定义默认存储类。有关更多信息，请参阅 Kubernetes 文档中的 [storage-classes](https://kubernetes.io/docs/concepts/storage/storage-classes)

**为 AWS 集群创建 Amazon EKS 存储类**

1. 确定您的集群已有的存储类。

```shell
kubectl get storageclass
```

输出

```
NAME            PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
gp2 (default)   kubernetes.io/aws-ebs   Delete          WaitForFirstConsumer   false                  145m
```

如果您的集群返回先前的输出，则它已经具有在剩余步骤中定义的 存储类。您可以使用[Storage](#Storage)本章中部署任何 CSI 驱动程序的步骤定义其他存储类。部署后，您可以将其中一个存储类设置为[默认](#定义默认存储类)存储类。

2. 为存储类创建 AWS 存储类清单文件。下面的 `gp2-storage-class.yaml` 示例定义一

名为 `gp2` 的存储类，该类使用 Amazon EBS `gp2` 卷类型。

有关可用于 AWS 存储类的选项的更多信息，请参阅 Kubernetes 文档中的 [AWS](https://kubernetes.io/docs/concepts/storage/storage-classes/#aws-ebs) EBS。

```
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: gp2
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp2
  fsType: ext4 
```

3. 使用 从清单文件`kubectl`创建存储类。

```shell
kubectl create -f gp2-storage-class.yaml
```

输出：

```shell
storageclass "gp2" created
```

如果输出下面内容表明该`gp2` storageclasses 已存在

```
Error from server (AlreadyExists): error when creating "gp2-storage-class.yaml": storageclasses.storage.k8s.io "gp2" already exists
```

<span id='定义默认存储类'>**定义默认存储类**</span>

1. 列出集群的现有存储类。必须先定义存储类，然后才能将它设置为默认存储类。

```shell
kubectl get storageclass
```

输出

```
NAME      PROVISIONER             AGE
gp2       kubernetes.io/aws-ebs   8m
```

2. 选择一个存储类并通过设置 `storageclass.kubernetes.io/is-default-class=true` 注释来将该存储类设置为默认存储类。

```shell
kubectl annotate storageclass gp2 storageclass.kubernetes.io/is-default-class=true
```

输出：

```
storageclass "gp2" patched
```

3. 确认存储类现已设置为默认存储类。

```shell
kubectl get storageclass
```

输出：

```
gp2 (default)   kubernetes.io/aws-ebs   12m
```

## 1.2 <span id='Amazon EBS CSI 驱动程序'>Amazon EBS CSI 驱动程序</span>

[Amazon Elastic Block Store （Amazon EBS） 容器存储接口 （CSI） 驱动程序](https://github.com/kubernetes-sigs/aws-ebs-csi-driver)提供了一个 CSI 接口，允许 Amazon Elastic Kubernetes Service （Amazon EKS） 集群管理持久性Amazon EBS卷的卷的生命周期。

下面介绍如何部署 Amazon EBS CSI 驱动程序到 Amazon EKS 集群，并验证它是否正常工作。我们建议使用驱动程序版本 v0.9.0

> **注意:**
>
> 驱动程序在 上不受支持Fargate。Amazon EBS 集群上不支持 Amazon EKS CSI 驱动程序的 Alpha 功能。驱动程序处于 Beta 版本。

**先决条件**

- 存在一个EKS集群

- 集群中现有 IAM OpenID Connect （OIDC） 提供程序。要确定您是否已经有或要创建一个，请参阅为集群[创建 IAM OIDC 提供商](#IAM OIDC 提供商)

- AWS CLI 版本 1.18.210 或更高版本安装在您的计算机

- `kubectl` 版本 1.15 或更高版本已安装在您的计算机或 [AWS CloudShell ](https://docs.aws.amazon.com/cloudshell/latest/userguide/welcome.html)上。 要安装或升级 kubectl

  <span id='IAM OIDC 提供商'>**检查集群是否存在IAM OIDC 提供商**</span>

2. 确定您的集群是否有现有的 IAM OIDC 提供商。

   查看集群的 OIDC 提供商 URL。

```shell
aws eks describe-cluster --name <cluster_name> --query "cluster.identity.oidc.issuer" --output text
```

输出示例：

```shell
https://oidc.eks.ap-southeast-2a.amazonaws.com/id/EXAMPLED539D4633E53DE1B716D3041E
```

列出您账户中的 IAM OIDC 提供商。Replace `<EXAMPLED539D4633E53DE1B716D3041E>`  替换为从上一个命令返回的值。

```shell
aws iam list-open-id-connect-providers | grep <EXAMPLED539D4633E53DE1B716D3041E>
```

输出示例

```shell
"Arn": "arn:aws:iam::111122223333:oidc-provider/oidc.eks.ap-southeast-2a.amazonaws.com/id/EXAMPLED539D4633E53DE1B716D3041E"
```

返回上面类似输出，则表示您的集群已有一个提供程序。如果未返回任何输出，则必须创建 IAM OIDC 提供商

2. 使用以下命令为您的集群创建 IAM OIDC 身份提供商。将 `<cluster_name>` 替换为您自己的值。

```shell
eksctl utils associate-iam-oidc-provider --cluster <cluster_name> --approve
```



**将 Amazon EBS CSI 驱动程序部署到 Amazon EKS 集群**

### 1.2.1 创建 IAM角色和策略

1. 创建允许 CSI 驱动程序的服务账户代表IAM AWS您调用 的 APIs 策略。您可以在 [GitHub上查看策略文档](https://github.com/kubernetes-sigs/aws-ebs-csi-driver/blob/v0.8.0/docs/example-iam-policy.json)。

   a. 从 下载IAM策略文档GitHub。

   ```shell
   curl -o example-iam-policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-ebs-csi-driver/v0.9.0/docs/example-iam-policy.json
   ```

   b. 创建策略。您可以将 `AmazonEKS_EBS_CSI_Driver_Policy` 更改为其他名称，但如果您这样做，请确保在后续步骤中也对其进行更改。

   ```shell
   aws iam create-policy --policy-name AmazonEKS_EBS_CSI_Driver_Policy \
     --policy-document file://example-iam-policy.json
   ```

### 1.2.2 创建一个 IAM 角色并将 IAM 策略附加到该角色。

​        a. 查看集群的 OIDC 提供商 URL。将 `<cluster_name>` 替换为您的集群名称。如果 命令的输出是 `None`，请查看**先决条件**。

```shell
aws eks describe-cluster --name <cluster_name> --query "cluster.identity.oidc.issuer" --output text
```

输出

```shell
https://oidc.eks.ap-southeast-2a.amazonaws.com/id/XXXXXXXXXX45D83924220DC4815XXXXX
```

​        b. 创建 IAM 角色。

​               i. 将以下内容复制到名为 的文件中`trust-policy.json`。 将 `<AWS_ACCOUNT_ID>` （包括 `<>`）替换为您的账户 ID，并将 `<XXXXXXXXXX45D83924220DC4815XXXXX>` 替换为上一步中返回的值。

```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::<AWS_ACCOUNT_ID>:oidc-provider/oidc.eks.us-west-2.amazonaws.com/id/<XXXXXXXXXX45D83924220DC4815XXXXX>"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "oidc.eks.ap-southeast-2a.amazonaws.com/id/<XXXXXXXXXX45D83924220DC4815XXXXX>:sub": "system:serviceaccount:kube-system:ebs-csi-controller-sa"
        }
      }
    }
  ]
}
```

> 注意: 如果不知道自己的 `AWS_ACCOUNT_ID` 可以通过下面命令获取
>
> ```shell
> aws sts get-caller-identity
> ```

​                ii. 创建 角色。您可以将 `AmazonEKS_EBS_CSI_DriverRole` 更改为其他名称，但如果您这样做，请确保在后面的步骤中也对其进行更改。

```shell
aws iam create-role \
  --role-name AmazonEKS_EBS_CSI_DriverRole \
  --assume-role-policy-document file://"trust-policy.json"
```

​        c. 将 IAM 策略附加到该角色。将 `<AWS_ACCOUNT_ID>` 替换为您的 账户 ID。

```shell
aws iam attach-role-policy \
  --policy-arn arn:aws:iam::<AWS_ACCOUNT_ID>:policy/AmazonEKS_EBS_CSI_Driver_Policy \
  --role-name AmazonEKS_EBS_CSI_DriverRole
```

### 1.2.3  部署驱动程序

1. 部署驱动程序，以便它使用您指定的标签标记它创建的所有Amazon EBS卷。

   a. 将 [Amazon EBS Container Storage Interface （CSI） 驱动程序](https://github.com/kubernetes-sigs/aws-ebs-csi-driver)GitHub存储库克隆到您的计算机。

```shell
git clone -b release-0.9 https://github.com/kubernetes-sigs/aws-ebs-csi-driver.git
```

​        b. 切换到`base`示例文件夹。

```shell
cd aws-ebs-csi-driver/deploy/kubernetes/base/
```

​        c. 编辑 `controller.yaml` 文件。查找包含以下文本在文件中的位置，并将`--extra-tags`将其添加到其中。以下文本显示 文件的 部分以及现有文本和添加的文本。此示例将导致控制器将 `department` 和 `environment` 标签添加到它创建的所有卷。

```json
...
      containers:
        - name: ebs-plugin
          image: k8s.gcr.io/provider-aws/aws-ebs-csi-driver:v0.9.1
          imagePullPolicy: IfNotPresent
          args:
            # - {all,controller,node} # specify the driver mode
            - --endpoint=$(CSI_ENDPOINT)
            - --logtostderr
            - --v=5
            - --extra-tags=department=accounting,environment=dev
...
```

拉取不到该镜像可以使用下面镜像地址

`image: amazon/aws-ebs-csi-driver:v0.9.1`

​        d. 将修改后的清单应用于您的集群。

```
kubectl apply -k ../base
```

2. 部署驱动程序时间不使用您指定的标签标记其创建的Amazon EBS卷。

```
kubectl apply -k "github.com/kubernetes-sigs/aws-ebs-csi-driver/deploy/kubernetes/overlays/stable/?ref=release-0.9"
```

### 1.2.4  修改`ebs-csi-controller-sa` 角色

使用您之前创建的 `ebs-csi-controller-sa` 角色的 ARN 对 IAM Kubernetes 服务账户进行注释。将 `<AWS_ACCOUNT_ID>` 替换为您的账户 ID。

```
kubectl annotate serviceaccount ebs-csi-controller-sa \
  -n kube-system \
  eks.amazonaws.com/role-arn=arn:aws:iam::<AWS_ACCOUNT_ID>:role/AmazonEKS_EBS_CSI_DriverRole
```

### 1.2.5 删除驱动程序 Pod。它们使用分配给角色的IAM策略中的IAM权限并自动重新部署。

```
kubectl delete pods \
  -n kube-system \
  -l=app=ebs-csi-controller
```

### 1.2.6  部署一个示例应用程序并验证 CSI 驱动程序是否正常运行

此过程使用https://github.com/kubernetes-sigs/aws-ebs-csi-driver/tree/master/examples/kubernetes/dynamic-provisioning来自容器存储接口 （CSI） 驱动程序[Amazon EBS存储库的](https://github.com/kubernetes-sigs/aws-ebs-csi-driver)动态卷预置GitHub示例来使用动态预置的Amazon EBS卷。

1. 将 [Amazon EBS Container Storage Interface （CSI） 驱动程序](https://github.com/kubernetes-sigs/aws-ebs-csi-driver)GitHub存储库克隆到您的本地系统。

   ```
   git clone -b release-0.9 https://github.com/kubernetes-sigs/aws-ebs-csi-driver.git
   ```

2. 导航到 `dynamic-provisioning` 示例目录。

   ```
   cd aws-ebs-csi-driver/examples/kubernetes/dynamic-provisioning/
   ```

3. 从 `ebs-sc` 目录部署 `ebs-claim` 存储类、`app` 持久性卷声明和 `specs` 示例应用程序。

   ```
   kubectl apply -f specs/
   ```

4. 描述 `ebs-sc` 存储类。

   ```
   kubectl describe storageclass ebs-sc
   ```

   输出：

   ```
   Name:            ebs-sc
   IsDefaultClass:  No
   Annotations:     kubectl.kubernetes.io/last-applied-configuration={"apiVersion":"storage.k8s.io/v1","kind":"StorageClass","metadata":{"annotations":{},"name":"ebs-sc"},"provisioner":"ebs.csi.aws.com","volumeBindingMode":"WaitForFirstConsumer"}
   
   Provisioner:           ebs.csi.aws.com
   Parameters:            <none>
   AllowVolumeExpansion:  <unset>
   MountOptions:          <none>
   ReclaimPolicy:         Delete
   VolumeBindingMode:     WaitForFirstConsumer
   Events:                <none>
   ```

   请注意，存储类使用 `WaitForFirstConsumer` 卷绑定模式。这意味着，在 Pod 进行持久性卷声明之前，不会动态预配置卷。有关更多信息，请参阅 Kubernetes 文档中的[卷绑定模式](https://kubernetes.io/docs/concepts/storage/storage-classes/#volume-binding-mode)。

5. 查看默认命名空间中的 Pod 并等待 `app` Pod 的状态变为 `Running`。

   ```
   kubectl get pods --watch
   ```

6. 列出默认命名空间中的持久性卷。查找具有 `default/ebs-claim` 声明的持久性卷。

   ```
   kubectl get pv
   ```

   输出：

   ```
   NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM               STORAGECLASS   REASON   AGE
   pvc-c5d8c175-cb20-49aa-a280-d2cb055d7d4e   4Gi        RWO            Delete           Bound    default/ebs-claim   ebs-sc                  30s
   ```

7. 描述持久性卷，将 `<pvc-c5d8c175-cb20-49aa-a280-d2cb055d7d4e>` 替换为上一步中的输出中的 值。

   ```
   kubectl describe pv <pvc-c5d8c175-cb20-49aa-a280-d2cb055d7d4e>
   ```

   输出：

   ```
   Name:              pvc-c5d8c175-cb20-49aa-a280-d2cb055d7d4e
   Labels:            <none>
   Annotations:       pv.kubernetes.io/provisioned-by: ebs.csi.aws.com
   Finalizers:        [kubernetes.io/pv-protection external-attacher/ebs-csi-aws-com]
   StorageClass:      ebs-sc
   Status:            Bound
   Claim:             default/ebs-claim
   Reclaim Policy:    Delete
   Access Modes:      RWO
   VolumeMode:        Filesystem
   Capacity:          4Gi
   Node Affinity:     
     Required Terms:  
       Term 0:        topology.ebs.csi.aws.com/zone in [ap-southeast-2a]
   Message:           
   Source:
       Type:              CSI (a Container Storage Interface (CSI) volume source)
       Driver:            ebs.csi.aws.com
       FSType:            ext4
       VolumeHandle:      vol-0f0da1a796bb61807
       ReadOnly:          false
       VolumeAttributes:      storage.kubernetes.io/csiProvisionerIdentity=1615709281774-8081-ebs.csi.aws.com
   Events:                <none>
   ```

   Amazon EBS 卷 ID 作为 `VolumeHandle`. 列出。

8. 验证 Pod 成功将数据写入卷。

   ```
   kubectl exec -it app -- cat /data/out.txt
   ```

   输出：

   ```
   Sun Mar 14 10:02:14 UTC 2021
   Sun Mar 14 10:02:19 UTC 2021
   Sun Mar 14 10:02:24 UTC 2021
   Sun Mar 14 10:02:29 UTC 2021
   Sun Mar 14 10:02:34 UTC 2021
   Sun Mar 14 10:02:39 UTC 2021
   ...
   ```

9. 完成试验时，请删除此示例应用程序来清除资源。

   ```
   kubectl delete -f specs/
   ```

![storage_classes_volume](https://51k8s.oss-cn-shenzhen.aliyuncs.com/oss-cn-shenzhenstorage_classes_volume.png)

## 1.3 <span id='Amazon EFS CSI 驱动程序'>Amazon EFS CSI 驱动程序</span>

[Amazon EFS 容器存储接口 （CSI） 驱动程序](https://github.com/kubernetes-sigs/aws-efs-csi-driver)提供了一个 CSI 接口，允许 上运行的 Kubernetes AWS 集群管理Amazon EFS文件系统的生命周期。

### 1.3.1 将 Amazon EFS CSI 驱动程序部署到 Amazon EKS 集群

**将 Amazon EFS CSI 驱动程序部署到 Amazon EKS 集群**

- 使用与您的集群所在的 区域对应的命令部署 Amazon EFS CSI 驱动程序。如果您的集群包含节点（它还可以包含 AWS Fargate Pod），请使用以下命令部署驱动程序。

  - 中国区域以外的所有区域。

    ```
    kubectl apply -k "github.com/kubernetes-sigs/aws-efs-csi-driver/deploy/kubernetes/overlays/stable/ecr/?ref=release-1.0"
    ```

  - 北京和宁夏 中国区域。

    ```
    kubectl apply -k "github.com/kubernetes-sigs/aws-efs-csi-driver/deploy/kubernetes/overlays/stable/?ref=release-1.0"
    ```

  如果您的集群仅包含 Fargate Pod（无节点），请使用以下命令部署驱动程序。

  ```
  kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-efs-csi-driver/master/deploy/kubernetes/base/csidriver.yaml
  ```

> **注意**
>
> - 从 1.0.0 版本开始，默认情况下启用使用 TLS 加密传输中的数据。使用传输[中](http://aws.amazon.com/blogs/aws/new-encryption-of-data-in-transit-for-amazon-efs/)加密，数据将在通过网络转换到 Amazon EFS 服务期间被加密。要使用 禁用它并挂载卷NFSv4，请在持久性卷清单`volumeAttributes`中`encryptInTransit`将 `"false"` 字段设置为 。有关示例清单，请参阅 上的传输中[加密示例](https://github.com/kubernetes-sigs/aws-efs-csi-driver/blob/master/examples/kubernetes/encryption_in_transit/specs/pv.yaml)GitHub。
> - 仅支持静态卷预置。这意味着，在集群中的 pod 使用 Amazon EFS 之前，需要在 外部创建 Amazon EKS 文件系统。

**Amazon EFS 访问点**

Amazon EFS CSI 驱动程序支持[Amazon EFS访问点](https://docs.aws.amazon.com/efs/latest/ug/efs-access-points.html)，这些访问点是 Amazon EFS 文件系统中特定于应用程序的入口点，便于在多个 Pod 之间共享文件系统。访问点可以为通过访问点发出的所有文件系统请求强制执行用户身份，并为每个 Pod 强制执行根目录。有关更多信息，请参阅 上的[Amazon EFS访问点](https://github.com/kubernetes-sigs/aws-efs-csi-driver/blob/master/examples/kubernetes/access_points/README.md)GitHub。

### 1.3.2 为 Amazon EFS 集群创建 Amazon EKS 文件系统

1. 找到您 Amazon EKS 集群的 VPC ID。您可以在 Amazon EKS 控制台中查找此 ID，或者使用以下 AWS CLI 命令。将 `<cluster_name>` 替换为您自己的值 （包括 `<>`）。

1. ```
   aws eks describe-cluster --name <cluster_name> --query "cluster.resourcesVpcConfig.vpcId" --output text
   ```

   输出：

   ```
   vpc-<exampledb76d3e813>
   ```

2. 查找您集群 VPC 的 CIDR 范围。您可以在 Amazon VPC 控制台中查找此信息，或者使用以下 AWS CLI 命令。

   ```
   aws ec2 describe-vpcs --vpc-ids vpc-<exampledb76d3e813> --query "Vpcs[].CidrBlock" --output text
   ```

   输出：

   ```
   192.168.0.0/16
   ```

3. 创建一个安全组，该安全组允许您 Amazon EFS 装载点的入站 NFS 流量。

   1. 打开 Amazon VPC 控制台 https://console.aws.amazon.com/vpc/。
   2. 在左侧导航面板中选择 **Security Groups** （安全组），然后选择 **Create security group （创建安全组**）。
   3. 为您的安全组输入名称和描述，然后选择您的 Amazon EKS 集群使用的 VPC。
   4. 在 **Inbound rules** （入站规则） 下，选择 **Add rule （添加规则**）。
   5. 在 **Type** （类型） 下，选择 NFS。
   6. 在 **Source** （源） 下，选择 Custom （**自定义**），然后粘贴您在上一步中获取的 VPC CIDR 范围。
   7. 选择**创建安全组**.

4. 为您的 Amazon EFS 集群创建 Amazon EKS 文件系统。

   1. 通过 https://console.aws.amazon.com/efs/ 打开 Amazon Elastic File System 控制台。

   2. 在左侧导航窗格中选择 **File** systems （文件系统），然后选择 **Create file system** （创建文件系统）。

   3. 在 **Create file system** （创建文件系统） 页面上，选择 **Customize** （自定义）。

   4. 在 **File system settings （文件系统设置**） 页面上，您无需输入或选择任何信息，但可以根据需要选择 **Next** （下一步）。

   5. 在 **Network access （网络访问**） 页面上，对于 **Virtual Private Cloud （**VPC），选择您的 VPC。

      **注意**

      如果您在控制台的右上角没有看到您的 VPC，请确保已选中您的 VPC 所在的区域。

   6. 在 **Mount** targets （挂载目标） 下，如果已列出默认安全组，请选择框右上角的 **X** 以从每个挂载点删除它，选择您在上一步中为每个挂载目标创建的安全组，然后选择 **Next** （下一步）。

   7. 在 **File system policy （文件系统策略**） 页面上，选择 **Next** （下一步）。

   8. 在 **Review and create （审核和创建**） 页面上，选择 **Create** （创建）。

> **重要**
>
> 默认情况下，新 Amazon EFS 文件系统由 `root:root` 拥有，并且只有 `root` 用户 (UID 0) 具有读取、写入和执行权限。如果您的容器没有作为 `root` 运行，则必须更改 Amazon EFS 文件系统权限，以允许其他用户修改文件系统。有关更多信息，请参阅 中的在网络文件系统 （NFS） https://docs.aws.amazon.com/efs/latest/ug/accessing-fs-nfs-permissions.html *级别Amazon Elastic File System 用户指南*使用用户、组和权限。

### 1.3.3 部署一个示例应用程序并验证 CSI 驱动程序是否正常运行

此过程使用https://github.com/kubernetes-sigs/aws-efs-csi-driver/tree/master/examples/kubernetes/multiple_pods来自 容器存储接口 （CSI） 驱动程序[Amazon EFS存储库](https://github.com/kubernetes-sigs/aws-efs-csi-driver)的多个 Pod 读取写入示例GitHub来使用静态预配置的Amazon EFS持久性卷并从具有 `ReadWriteMany` 访问模式的多个 Pod 访问它。

1. 将 [Amazon EFS Container Storage Interface （CSI） 驱动程序](https://github.com/kubernetes-sigs/aws-efs-csi-driver)GitHub存储库克隆到您的本地系统。

   ```
   git clone -b release-1.0 https://github.com/kubernetes-sigs/aws-efs-csi-driver.git
   ```

2. 导航到 `multiple_pods` 示例目录。

   ```
   cd aws-efs-csi-driver/examples/kubernetes/multiple_pods/
   ```

3. 检索 Amazon EFS 文件系统 ID。您可以在 Amazon EFS 控制台中查找此信息，或者使用以下 AWS CLI 命令。

   ```
   aws efs describe-file-systems --query "FileSystems[*].FileSystemId" --output text
   ```

   输出：

   ```
   fs-<582a03f3>
   ```

4. 编辑 `specs/pv.yaml` 文件并将 `volumeHandle` 值替换为您的 Amazon EFS 文件系统 ID。

   ```
   apiVersion: v1
   kind: PersistentVolume
   metadata:
     name: efs-pv
   spec:
     capacity:
       storage: 5Gi
     volumeMode: Filesystem
     accessModes:
       - ReadWriteMany
     persistentVolumeReclaimPolicy: Retain
     storageClassName: efs-sc
     csi:
       driver: efs.csi.aws.com
       volumeHandle: fs-<582a03f3>
   ```

   **注意**

   由于 Amazon EFS 是弹性文件系统，它不会强制实施任何文件系统容量限制。在创建系统时，不使用持久性卷和持久性卷声明中的实际存储容量值。但是，由于存储容量是 Kubernetes 中的必需字段，您必须指定有效值，例如`5Gi`，在此示例中。此值不会限制 Amazon EFS 文件系统的大小。

5. 从 目录部署`efs-sc`存储类`efs-claim`、持久性卷声明和`efs-pv`持久性卷`specs`。

   ```
   kubectl apply -f specs/pv.yaml
   kubectl apply -f specs/claim.yaml
   kubectl apply -f specs/storageclass.yaml
   ```

6. 列出默认命名空间中的持久性卷。查找具有 `default/efs-claim` 声明的持久性卷。

   ```
   kubectl get pv -w
   ```

   输出：

   ```
   NAME     CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM               STORAGECLASS   REASON   AGE
   efs-pv   5Gi        RWX            Retain           Bound    default/efs-claim   efs-sc                  2m50s
   ```

   在 `STATUS` 变为 之前，请勿继续执行下一步`Bound`。

7. 从 目录部署 `app1` 和`app2`示例应用程序`specs`。

   ```
   kubectl apply -f specs/pod1.yaml
   kubectl apply -f specs/pod2.yaml
   ```

8. 查看默认命名空间中的 Pod，并等待 `app1` 和 `app2` Pod `STATUS` 变为 `Running`。

   ```
   kubectl get pods --watch
   ```

   **注意**

   Pod 达到 `Running` 状态可能需要几分钟时间。

9. 描述持久性卷。

   ```
   kubectl describe pv efs-pv
   ```

   输出：

   ```
   Name:            efs-pv
   Labels:          none
   Annotations:     kubectl.kubernetes.io/last-applied-configuration:
                      {"apiVersion":"v1","kind":"PersistentVolume","metadata":{"annotations":{},"name":"efs-pv"},"spec":{"accessModes":["ReadWriteMany"],"capaci...
                    pv.kubernetes.io/bound-by-controller: yes
   Finalizers:      [kubernetes.io/pv-protection]
   StorageClass:    efs-sc
   Status:          Bound
   Claim:           default/efs-claim
   Reclaim Policy:  Retain
   Access Modes:    RWX
   VolumeMode:      Filesystem
   Capacity:        5Gi
   Node Affinity:   none
   Message:
   Source:
       Type:              CSI (a Container Storage Interface (CSI) volume source)
       Driver:            efs.csi.aws.com
       VolumeHandle:      fs-582a03f3
       ReadOnly:          false
       VolumeAttributes:  none
   Events:                none
   ```

   Amazon EFS 文件系统 ID 将作为 `VolumeHandle`. 列出。

10. 验证 `app1` Pod 成功将数据写入卷。

    ```
    kubectl exec -ti app1 -- tail /data/out1.txt
    ```

    输出：

    ```
    Sun Mar 14 11:02:14 UTC 2021
    Sun Mar 14 11:02:19 UTC 2021
    Sun Mar 14 11:02:24 UTC 2021
    Sun Mar 14 11:02:29 UTC 2021
    Sun Mar 14 11:02:34 UTC 2021
    Sun Mar 14 11:02:39 UTC 2021
    ```

11. 验证 `app2` pod 显示`app1`写入到卷的卷中的相同数据。

    ```
    kubectl exec -ti app2 -- tail /data/out1.txt
    ```

    输出：

    ```
    Sun Mar 14 11:02:14 UTC 2021
    Sun Mar 14 11:02:19 UTC 2021
    Sun Mar 14 11:02:24 UTC 2021
    Sun Mar 14 11:02:29 UTC 2021
    Sun Mar 14 11:02:34 UTC 2021
    Sun Mar 14 11:02:39 UTC 2021
    ```

12. 完成试验时，请删除此示例应用程序来清除资源。

    ```
    kubectl delete -f specs/
    ```

    您还可以手动删除您创建的文件系统和安全组。 

## 1.4 PV(PersistentVolume)详解  

上面介绍了怎么部署CSI 驱动程序并实现了一个演示案例, 接触了几个 专业词语 `pv` 、`pcv`  、`StorageClass`  下面就介绍一下这些名词的解释的小案例。

`PV` 作为存储资源，主要包括存储能力、访问能力、存储类型、回收策略、后端存储类型等关键信息的设置。

### 1.4.1 PV的关键配置参数

1. 存储能力 (Capacity)

   描述存储设备具备的能力，容量属性是使用 PV 对象的 `capacity` 属性来设置的，参考 Kubernetes [资源模型（Resource Model）](https://git.k8s.io/community/contributors/design-proposals/scheduling/resources.md) 设计提案，了解 `capacity` 字段可以接受的单位。目前仅支持对存储空间的设置（storage=xx），未来可能会加入IOPS、吞吐率等指标的设置。

   ```json
   apiVersion: v1
   kind: PersistentVolume
   metadata:
     name: pv0001
   spec:
     capacity:
       storage: 5Gi
   ...
   ```

2. 存储卷模式 (Volume Mode)

   ​        针对 PV 持久卷，Kuberneretes 支持两种卷模式（`volumeModes`）：`Filesystem（文件系统）` 和 `Block（块）`。 `volumeMode` 是一个可选的 API 参数。 如果该参数被省略，默认的卷模式是 `Filesystem`。

   ​        `volumeMode` 属性设置为 `Filesystem` 的卷会被 Pod *挂载（Mount）* 到某个目录。 如果卷的存储来自某块设备而该设备目前为空，Kuberneretes 会在第一次挂载卷之前 在设备上创建文件系统。

   ​        你可以将 `volumeMode` 设置为 `Block`，以便将卷作为原始块设备来使用。 这类卷以块设备的方式交给 Pod 使用，其上没有任何文件系统。 这种模式对于为 Pod 提供一种使用最快可能方式来访问卷而言很有帮助，Pod 和 卷之间不存在文件系统层。另外，Pod 中运行的应用必须知道如何处理原始块设备。 关于如何在 Pod 中使用 `volumeMode: Block` 的卷，可参阅 [原始块卷支持](https://kubernetes.io/zh/docs/concepts/storage/persistent-volumes/#raw-block-volume-support)。

3. 访问模式 (Access Modes)

   ​        PersistentVolume 卷可以用资源提供者所支持的任何方式挂载到宿主系统上。 如下表所示，提供者（驱动）的能力不同，每个 PV 卷的访问模式都会设置为对应卷所支持的模式值。 例如，NFS 可以支持多个读写客户，但是某个特定的 NFS PV 卷可能在服务器 上以只读的方式导出。每个 PV 卷都会获得自身的访问模式集合，描述的是 特定 PV 卷的能力。

   访问模式有：

   - ReadWriteOnce -- 卷可以被一个节点以读写方式挂载；
   - ReadOnlyMany -- 卷可以被多个节点以只读方式挂载；
   - ReadWriteMany -- 卷可以被多个节点以读写方式挂载。

   在命令行接口（CLI）中，访问模式也使用以下缩写形式：

   - RWO - ReadWriteOnce
   - ROX - ReadOnlyMany
   - RWX - ReadWriteMany

   > **重要提醒！** 每个卷只能同一时刻只能以一种访问模式挂载，即使该卷能够支持 多种访问模式。例如，一个 GCEPersistentDisk 卷可以被某节点以 ReadWriteOnce 模式挂载，或者被多个节点以 ReadOnlyMany 模式挂载，但不可以同时以两种模式 挂载。

   # ![pv-access-modes](https://51k8s.oss-cn-shenzhen.aliyuncs.com/oss-cn-shenzhenpv-access-modes.png)

4. 存储类别 (Class)

   ​        PV可以设定其存储的类别，通过`storageClassName`参数指定一个`StorageClass`资源对象的名称。具有特定类别的PV只能与请求了该类别的PVC进行绑定。未设定类别的PV则只能与不请求任何类别的PVC进行绑定。

   ​        早前，Kubernetes 使用注解 `volume.beta.kubernetes.io/storage-class` 而不是 `storageClassName` 属性。这一注解目前仍然起作用，不过在将来的 Kubernetes 发布版本中该注解会被彻底废弃

5. 回收策略 (Reclaim Policy)

   通过PV定义中的`persistentVolumeReclaimPolicy`字段进行设置回收策略, 目前的回收策略有:

   - Retain -- 会保留数据，需要管理员手动回收。
   - Recycle -- 擦除 例如使用(`rm -rf /thevolume/*`)。
   - Delete -- 与PV相连的后端存储完成 `Volume` 的删除操作 诸如 AWS EBS、GCE PD、Azure Disk 或 OpenStack Cinder 卷这类关联存储资产还被删除。

   目前，仅 NFS 和 HostPath 支持回收（Recycle）。 AWS EBS、GCE PD、Azure Disk 和 Cinder 卷都支持删除（Delete）。

6. 挂载参数 (Mount Option)

   ​        在将PV挂载到一个Node上时，根据后端存储的特点，可能需要设置额外的挂载参数，可以根据PV定义中的mountOptions字段进行设置。

   ​        过去，使用注释`volume.beta.kubernetes.io/mount-options`代替`mountOptions`属性。此注释仍在起作用；但是，它将在以后的Kubernetes版本中完全弃用。

7. 节点亲和性 (Node Affinity)

   ​        PV可以设置[节点亲和性](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.20/#volumenodeaffinity-v1-core)来限制只能通过某些Node访问Volume，可以在PV定义中的nodeAffinity字段进行设置。使用这些Volume的Pod将被调度到满足条件的Node上

   > **说明：** 对大多数类型的卷而言，你不需要设置节点亲和性字段。 [AWS EBS](https://kubernetes.io/zh/docs/concepts/storage/volumes/#awselasticblockstore)、 [GCE PD](https://kubernetes.io/zh/docs/concepts/storage/volumes/#gcepersistentdisk) 和 [Azure Disk](https://kubernetes.io/zh/docs/concepts/storage/volumes/#azuredisk) 卷类型都能自动设置相关字段。 你需要为 [local](https://kubernetes.io/zh/docs/concepts/storage/volumes/#local) 设置此属性。

```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv002
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Delete
  storageClassName: locat-storage
  local:
    path: /mnt/disks/ssd1
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
      - key: kubernetes.io/hostname
        operator: In
        values:
        - my-node
```

### 1.4.2 PV的生命周期

每个卷会处于以下阶段（Phase）之一：

- Available（可用）-- 卷是一个空闲资源，尚未绑定到任何PVC；
- Bound（已绑定）-- 该卷已经绑定到PVC；
- Released（已释放）-- 所绑定的PVC已被删除，但是资源尚未被集群回收；
- Failed（失败）-- 卷的自动回收操作失败。

## 1.5 PVC (PersistentVolumeClaims) 详解

​        PVC作为用户对存储资源的需求申请，主要包括存储空间请求、访问模式、PV选择条件和存储类别等信息的设置。

​        下例声明的 PVC (pvc001) 具有如下属性：申请8GiB存储空间，访问模式为ReadWriteOnce，PV选择条件为包含标签“ release=stable”并且包含条件为“environment In [dev]”的标签，存储类别为“slow”（要求在系统中已存在名为slow的StorageClass）：

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc001
spec:
  accessModes:
    - ReadWriteOnce
  volumeMode: Filesystem
  resources:
    requests:
      storage: 8Gi
  storageClassName: slow
  selector:
    matchLabels:
      release: "stable"
    matchExpressions:
      - {key: environment, operator: In, values: [dev]}
```

- 资源请求（Resources）：描述对存储资源的请求，目前仅支持request.storage的设置，即存储空间大小。
- 访问模式（Access Modes）：PVC也可以设置访问模式，用于描述用户应用对存储资源的访问权限。其三种访问模式的设置与PV的设置相同。
- 存储卷模式（Volume Modes）：PVC也可以设置存储卷模式，用于描述希望使用的PV存储卷模式，包括文件系统和块设备。
- PV选择条件（Selector）：通过对Label Selector的设置，可使PVC对于系统中已存在的各种PV进行筛选。系统将根据标签选出合适的PV与该PVC进行绑定。选择条件可以使用matchLabels和matchExpressions进行设置，如果两个字段都设置了，则Selector的逻辑将是两组条件同时满足才能完成匹配。
- 存储类别（Class）： PVC在定义时可以设定需要的后端存储的类别（通过storageClassName字段指定），以减少对后端存储特性的详细信息的依赖。只有设置了该Class的PV才能被系统选出，并与该PVC进行绑定。



​        PVC也可以不设置Class需求。如果storageClassName字段的值被设置为空（storageClassName=""），则表示该PVC不要求特定的Class，系统将只选择未设定Class的PV与之匹配和绑定。PVC也可以完全不设置storageClassName字段，此时将根据系统是否启用了名为DefaultStorageClass的admissioncontroller进行相应的操作。

- 未启用DefaultStorageClass ：等效于PVC设置storageClassName的值为空（storageClassName=""），即只能选择未设定Class的PV与之匹配和绑定。

- 启用DefaultStorageClass：要求集群管理员已定义默认的StorageClass。如果在系统中不存在默认的StorageClass，则等效于不启用DefaultStorageClass的情况。如果存在默认的StorageClass，则系统将自动为PVC创建一个PV（使用默认StorageClass的后端存储），并将它们进行绑定。集群管理员设置默认StorageClass的方法为，在StorageClass的定义中加上一个annotation“storageclass.kubernetes.io/is-default-class= true”。如果管理员将多个StorageClass都定义为default，则由于不唯一，系统将无法为PVC创建相应的PV。


> 注意，PVC和PV都受限于Namespace，PVC在选择PV时受到Namespace的限制，只有相同Namespace中的PV才可能与PVC绑定。Pod在引用PVC时同样受Namespace的限制，只有相同Namespace中的PVC才能挂载到Pod内。

​        当Selector和Class都进行了设置时，系统将选择两个条件同时满足的PV与之匹配。另外，如果资源供应使用的是动态模式，即管理员没有预先定义PV，仅通过StorageClass交给系统自动完成PV的动态创建，那么PVC再设定Selector时，系统将无法为其供应任何存储资源。

​        在启用动态供应模式的情况下，一旦用户删除了PVC，与之绑定的PV也将根据其默认的回收策略“Delete”被删除。如果需要保留PV（用户数据），则在动态绑定成功后，用户需要将系统自动生成PV的回收策略从“Delete”改成“Retain”。

## 1.6 PV和PVC 的生命周期

我们可以将PV看作可用的存储资源，PVC则是对存储资源的需求，PV和PVC的相互关系遵循如下图所示的生命周期。

![pv-pvc-lifi_cycle](https://51k8s.oss-cn-shenzhen.aliyuncs.com/oss-cn-shenzhenpv-pvc-lifi_cycle.jpg)

### 1.6.1 资源供应

Kubernetes支持两种资源的供应模式：静态模式（Static）和动态模式（Dynamic）。资源供应的结果就是创建好的PV。

- 静态模式：集群管理员手工创建许多PV，在定义PV时需要将后端存储的特性进行设置。
- 动态模式：集群管理员无须手工创建PV，而是通过StorageClass的设置对后端存储进行描述，标记为某种类型。此时要求PVC对存储的类型进行声明，系统将自动完成PV的创建及与PVC的绑定。PVC可以声明Class为""，说明该PVC禁止使用动态模式。

### 1.6.2 资源绑定

​        在用户定义好PVC之后，系统将根据PVC对存储资源的请求（存储空间和访问模式）在已存在的PV中选择一个满足PVC要求的PV，一旦找到，就将该PV与用户定义的PVC进行绑定，用户的应用就可以使用这个PVC了。如果在系统中没有满足PVC要求的PV，PVC则会无限期处于Pending状态，直到等到系统管理员创建了一个符合其要求的PV。PV一旦绑定到某个PVC上，就会被这个PVC独占，不能再与其他PVC进行绑定了。在这种情况下，当PVC申请的存储空间比PV的少时，整个PV的空间就都能够为PVC所用，可能会造成资源的浪费。如果资源供应使用的是动态模式，则系统在为PVC找到合适的StorageClass后，将自动创建一个PV并完成与PVC的绑定。

### 1.6.3 资源使用

​        Pod使用Volume的定义，将PVC挂载到容器内的某个路径进行使用。Volume的类型为persistentVolumeClaim，在后面的示例中再进行详细说明。在容器应用挂载了一个PVC后，就能被持续独占使用。不过，多个Pod可以挂载同一个PVC，应用程序需要考虑多个实例共同访问一块存储空间的问题。

### 1.6.4 资源释放

​        当用户对存储资源使用完毕后，用户可以删除PVC，与该PVC绑定的PV将会被标记为“已释放”，但还不能立刻与其他PVC进行绑定。通过之前PVC写入的数据可能还被留在存储设备上，只有在清除之后该PV才能再次使用。

### 1.6.5 资源回收

​        对于PV，管理员可以设定回收策略，用于设置与之绑定的PVC释放资源之后如何处理遗留数据的问题。只有PV的存储空间完成回收，才能供新的PVC绑定和使用。回收策略详见下节的说明。

​        下面通过两张图分别对在静态资源供应模式和动态资源供应模式下，PV、PVC、StorageClass及Pod使用PVC的原理进行说明。

​        下图描述了在静态资源供应模式下，通过PV和PVC完成绑定，并供Pod使用的存储管理机制。

![static_res-pv-pvc](https://51k8s.oss-cn-shenzhen.aliyuncs.com/oss-cn-shenzhenstatic_res-pv-pvc.jpg)

​        下图描述了在动态资源供应模式下，通过StorageClass和PVC完成资源动态绑定（系统自动生成PV），并供Pod使用的存储管理机制。

![auto_res-pv-pvc](https://51k8s.oss-cn-shenzhen.aliyuncs.com/oss-cn-shenzhenauto_res-pv-pvc.jpg)

## 1.7 StorageClass详解

​        StorageClass作为对存储资源的抽象定义，对用户设置的PVC申请屏蔽后端存储的细节，一方面减少了用户对于存储资源细节的关注，另一方面减轻了管理员手工管理PV的工作，由系统自动完成PV的创建和绑定，实现了动态的资源供应。基于StorageClass的动态资源供应模式将逐步成为云平台的标准存储配置模式。

​        StorageClass的定义主要包括名称、后端存储的提供者（provisioner）和后端存储的相关参数配置。StorageClass一旦被创建出来，则将无法修改。如需更改，则只能删除原StorageClass的定义重建。下例定义了一个名为standard的StorageClass，提供者为aws-ebs，其参数设置了一个type，值为gp2：

```json
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: standard
#  annotations:
#    storageclass.kubernetes.io/is-default-class: "true"
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp2
```

### 1.7.1  StorageClass的关键配置参数

1. 提供者（Provisioner）

   ​        描述存储资源的提供者，也可以看作后端存储驱动。目前Kubernetes支持的Provisioner都以“ kubernetes.io/”为开头，用户也可以使用自定义的后端存储提供者。为了符合StorageClass的用法，自定义Provisioner需要符合存储卷的开发规范，详见https://kubernetes.io/docs/concepts/storage/persistent-volumes的说明。

2. 参数（Parameters）

   ​       后端存储资源提供者的参数设置，不同的Provisioner包括不同的参数设置。某些参数可以不显示设定，Provisioner将使用其默认值。接下来通过几种常见的Provisioner对StorageClass的定义进行详细说明。

**1）AWS EBS存储卷**

```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: slow
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp2
  zone: ap-southeast-2a
  iopsPerGB: "10"
```

参数说明如下（详细说明请参考[AWS EBS文档](https://kubernetes.io/docs/concepts/storage/storage-classes/#aws-ebs)）。

- `type`：可选项为io1，gp2，sc1，st1，默认值为gp2。后面还增加了 gp3
- `zone`(弃用)：AWS zone的名称。如果没有指定 `zone` 和 `zones`， 通常卷会在 Kubernetes 集群节点所在的活动区域中轮询调度分配。 `zone` 和 `zones` 参数不能同时使用。
- `zones`(弃用):  以逗号分隔的 AWS 区域列表。 如果没有指定 `zone` 和 `zones`，通常卷会在 Kubernetes 集群节点所在的 活动区域中轮询调度分配。`zone`和`zones`参数不能同时使用。
- `iopsPerGB`：仅用于io1类型的Volume，意为每秒每GiB的I/O操作数量。 AWS 卷插件将其与请求卷的大小相乘以计算 IOPS 的容量， 并将其限制在 20000 IOPS（AWS 支持的最高值，请参阅 [AWS 文档](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/EBSVolumeTypes.html)。 这里需要输入一个字符串，即 `"10"`，而不是 `10`。
- `fsType`：受 Kubernetes 支持的文件类型。默认值：`"ext4"`。
- `encrypted`：是否加密。值为 `"true"` 或者 `"false"`。
- `kmsKeyId`：加密时的Amazon Resource Name。如果没有提供，但 `encrypted` 值为 true，AWS 将生成一个密钥。关于有效的 ARN 值，请参阅 AWS 文档。

> **注意：** `zone`和`zones`参数已弃用，并替换为 [allowedTopologies](https://kubernetes.io/docs/concepts/storage/storage-classes/#allowed-topologies)

**Allowed Topologies**

当集群操作员指定`WaitForFirstConsumer`卷绑定模式时，在大多数情况下，不再需要将配置限制为特定的拓扑。但是，如果仍然需要，`allowedTopologies`可以指定。

如下:

```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: standard
provisioner: kubernetes.io/gce-pd
parameters:
  type: pd-standard
volumeBindingMode: WaitForFirstConsumer
allowedTopologies:
- matchLabelExpressions:
  - key: failure-domain.beta.kubernetes.io/zone
    values:
    - ap-southeast-2a
    - ap-southeast-2b
```

**2）GCE PD存储卷**

```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: slow
provisioner: kubernetes.io/gce-pd
parameters:
  type: pd-standard
  fstype: ext4
  replication-type: none
```

type：可选项为pd-standard、pd-ssd，默认值为pd-standard。

- zone(弃用)：GCE zone名称 
- zones(弃用)：GCE zone名称 逗号分隔列表，如果既未指定`zone`也未指定`zones`，则通常会在Kubernetes集群具有节点的所有活动区域中进行卷循环。`zone`和`zones`参数不能同时使用。
- `fstype`：`ext4`或`xfs`。默认值：`ext4`。主机操作系统必须支持定义的文件系统类型。
- `replication-type`：`none`或`regional-pd`。默认值：`none`。

如果`replication-type`设置为`none`，则将设置常规（区域）PD。

​        如果`replication-type`设置为`regional-pd`， 则将设置[区域永久磁盘](https://cloud.google.com/compute/docs/disks/#repds) 。强烈建议进行 `volumeBindingMode: WaitForFirstConsumer`设置，在这种情况下，当您创建一个使用使用此StorageClass的PersistentVolumeClaim的Pod时，将为区域永久磁盘配备两个区域。一个区域与Pod计划在其中的区域相同。另一区域是从群集可用的区域中随机选择的。可以使用进一步限制磁盘区域`allowedTopologies`。

### 1.7.2 设置默认的StorageClass