# 1. 为创建EKS的前期准备工作

EKS 创建 需要安装aws上一些工具，例如 AWS CLI，eksctl，kubectl等



## 1.1 安装 AWS CLI

在Linux上安装AWS CLI版本 2

从命令行按照以下步骤在Linux上安装AWS CLI。

官方地址https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2-linux.html

Linux x86（64位）最新版的AWS CLI 使用下面命令安装

```shell
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
```



## 1.2 安装 eksctl

`eksctl` 使用以下命令在Linux上安装或升级

1. `eksctl`使用以下命令下载并解压缩最新版本。

```shell
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
```

2. 将提取的二进制文件移到`/usr/local/bin`。

```shell
sudo mv /tmp/eksctl /usr/local/bin
```

3. 使用以下命令测试安装是否成功。

```shell
eksctl version
```



## 1.3 安装 kubectl

1. `kubectl`从Amazon S3下载适用于您集群的Kubernetes版本的Amazon EKS出售二进制文件。要下载Arm版本，`amd64`请`arm64`在运行命令之前更改为 。

- **Kubernetes 1.19：**

```
curl -o kubectl https://amazon-eks.s3.us-west-2.amazonaws.com/1.19.6/2021-01-05/bin/linux/amd64/kubectl
```

- **Kubernetes 1.18：**

```
curl -o kubectl https://amazon-eks.s3.us-west-2.amazonaws.com/1.18.9/2020-11-02/bin/linux/amd64/kubectl
```

- **Kubernetes 1.17：**

```
curl -o kubectl https://amazon-eks.s3.us-west-2.amazonaws.com/1.17.12/2020-11-02/bin/linux/amd64/kubectl
```

2. 将执行权限应用于二进制文件。

```
chmod +x ./kubectl
```

3. 将二进制文件复制到您的文件夹中`PATH`。如果您已经安装的版本`kubectl`，那么我们建议您创建一个，`$HOME/bin/kubectl`并确保该版本 `$HOME/bin`首先出现在您的中 `$PATH`。

```
mkdir -p $HOME/bin && cp ./kubectl $HOME/bin/kubectl && export PATH=$PATH:$HOME/bin
```

4. 将`$HOME/bin`路径添加到外壳初始化文件中，以便在打开外壳时对其进行配置。

```
echo 'export PATH=$PATH:$HOME/bin' >> ~/.bashrc
```

5. 安装后 `kubectl` ，可以使用以下命令验证其版本：

```
kubectl version --short --client
```

## 1.4 安装 jq

```
sudo yum install -y jq
```


