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

```
aws ec2 create-volume --availability-zone=ap-southeast-2a --size=10 --volume-type=gp2 --tag-specifications 'ResourceType=volume,Tags=[{Key=Name,Value=EKS_EBS_SIZE-10G},]'
```

确保该区域与您在其中引入群集的区域匹配。并请检查大小和EBS卷类型是否适合您的使用。

![EKS_EBS_SIZE-10G](https://51k8s.oss-cn-shenzhen.aliyuncs.com/oss-cn-shenzhenEKS_EBS_SIZE-10G.png)

在控制台平面上也能够看到我们创建的 EBS

![AWS-EBS-10G](https://51k8s.oss-cn-shenzhen.aliyuncs.com/oss-cn-shenzhenAWS-EBS-10G.png)

通过AWS 创建EBS各参数可参考下面地址

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

```
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

```
kubectl create -f gp2-storage-class.yaml
```

输出：

```
storageclass "gp2" created
```

如果输出下面内容表明该`gp2` storageclasses 已存在

```
Error from server (AlreadyExists): error when creating "gp2-storage-class.yaml": storageclasses.storage.k8s.io "gp2" already exists
```

<span id='定义默认存储类'>**定义默认存储类**</span>

1. 列出集群的现有存储类。必须先定义存储类，然后才能将它设置为默认存储类。

```
kubectl get storageclass
```

输出

```
NAME      PROVISIONER             AGE
gp2       kubernetes.io/aws-ebs   8m
```

2. 选择一个存储类并通过设置 `storageclass.kubernetes.io/is-default-class=true` 注释来将该存储类设置为默认存储类。

```
kubectl annotate storageclass gp2 storageclass.kubernetes.io/is-default-class=true
```

输出：

```
storageclass "gp2" patched
```

3. 确认存储类现已设置为默认存储类。

```
kubectl get storageclass
```

输出：

```
gp2 (default)   kubernetes.io/aws-ebs   12m
```