# 1. Configmap 简单介绍

​        ConfigMap 是一种 API 对象，用来将`非机密性的数据`保存到键值对中。使用时， [Pods](https://kubernetes.io/docs/concepts/workloads/pods/pod-overview/) 可以将其用作环境变量、命令行参数或者存储卷中的配置文件。

​         ConfigMap 将您的环境配置信息和 [容器镜像](https://kubernetes.io/zh/docs/reference/glossary/?all=true#term-image) 解耦，便于应用配置的修改。

> **注意：**
>
> ConfigMap 并不提供保密或者加密功能。 如果你想存储的数据是机密的，请使用 [Secret](https://kubernetes.io/zh/docs/concepts/configuration/secret/)， 或者使用其他第三方工具来保证你的数据的私密性，而不是用 ConfigMap。

**作用**

​    使用 ConfigMap 来将你的配置数据和应用程序代码分开。

​        在 ConfigMap 中保存的数据不可超过 1 MiB。如果你需要保存超出此尺寸限制的数据，你可能希望考虑挂载存储卷 或者使用独立的数据库或者文件服务。

## 2. 创建一个ConfigMap

​        您可以使用`kubectl create configmap`或ConfigMap生成器`kustomization.yaml`来创建ConfigMap。请注意，自1.14`kubectl`开始支持`kustomization.yaml`。

## 2.1 使用kubectl创建ConfigMap创建configmap

​        使用`kubectl create configmap`命令从[目录](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/#create-configmaps-from-directories)，[文件](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/#create-configmaps-from-files)或[文字值](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/#create-configmaps-from-literal-values)创建ConfigMap ：

```shell
kubectl create configmap <map-name> <data-source>
```

​        其中，<map-name>是要分配给ConfigMap的名称，而<data-source>是要从中提取数据的目录，文件或文字值。ConfigMap对象的名称必须是有效的 [DNS子域名](https://kubernetes.io/docs/concepts/overview/working-with-objects/names#dns-subdomain-names)。

​        当您基于文件创建ConfigMap时，<data-source>中的键默认为文件的基本名称，而值默认为文件内容。

​        您可以使用[`kubectl describe`](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands/#describe)或 [`kubectl get`](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands/#get)检索有关ConfigMap的信息。

## 2.2 从目录创建ConfigMap

​        您可以用来`kubectl create configmap`从同一目录中的多个文件创建ConfigMap。当您基于目录创建ConfigMap时，kubectl会识别基本名是目录中有效密钥的文件，并将每个文件打包到新的ConfigMap中。除常规文件外，所有目录条目都将被忽略（例如，子目录，符号链接，设备，管道等）。

  例如：

```shell
# Create the local directory
mkdir -p configure-pod-container/configmap/

# Download the sample files into `configure-pod-container/configmap/` directory
wget https://kubernetes.io/examples/configmap/game.properties -O configure-pod-container/configmap/game.properties
wget https://kubernetes.io/examples/configmap/ui.properties -O configure-pod-container/configmap/ui.properties

# Create the configmap
kubectl create configmap game-config --from-file=configure-pod-container/configmap/
```

在这种情况下，以上命令将每个文件打包，`game.properties`并将目录`ui.properties`中的每个文件打包`configure-pod-container/configmap/`到game-config ConfigMap中。您可以使用以下命令显示ConfigMap的详细信息：

```shell
kubectl describe configmaps game-config
```

输出类似于以下内容：

```
Name:         game-config
Namespace:    default
Labels:       <none>
Annotations:  <none>

Data
====
game.properties:
----
enemies=aliens
lives=3
enemies.cheat=true
enemies.cheat.level=noGoodRotten
secret.code.passphrase=UUDDLRLRBABAS
secret.code.allowed=true
secret.code.lives=30
ui.properties:
----
color.good=purple
color.bad=yellow
allow.textmode=true
how.nice.to.look=fairlyNice
```

目录中的`game.properties`和`ui.properties`文件在ConfigMap`configure-pod-container/configmap/`的`data`部分中表示。

```shell
kubectl get configmaps game-config -o yaml
```

输出类似于以下内容：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  creationTimestamp: 2016-02-18T18:52:05Z
  name: game-config
  namespace: default
  resourceVersion: "516"
  uid: b4952dc3-d670-11e5-8cd0-68f728db1985
data:
  game.properties: |
    enemies=aliens
    lives=3
    enemies.cheat=true
    enemies.cheat.level=noGoodRotten
    secret.code.passphrase=UUDDLRLRBABAS
    secret.code.allowed=true
    secret.code.lives=30    
  ui.properties: |
    color.good=purple
    color.bad=yellow
    allow.textmode=true
    how.nice.to.look=fairlyNice    
```

## 2.3 从文件创建ConfigMap

您可以用来`kubectl create configmap`从单个文件或多个文件创建ConfigMap。

例如，

```shell
kubectl create configmap game-config-2 --from-file=configure-pod-container/configmap/game.properties
```

将产生以下ConfigMap：

```shell
kubectl describe configmaps game-config-2
```

输出类似于以下内容：

```
Name:         game-config-2
Namespace:    default
Labels:       <none>
Annotations:  <none>

Data
====
game.properties:
----
enemies=aliens
lives=3
enemies.cheat=true
enemies.cheat.level=noGoodRotten
secret.code.passphrase=UUDDLRLRBABAS
secret.code.allowed=true
secret.code.lives=30
```

您可以`--from-file`多次传入参数以从多个数据源创建ConfigMap。

```shell
kubectl create configmap game-config-2 --from-file=configure-pod-container/configmap/game.properties --from-file=configure-pod-container/configmap/ui.properties
```

您可以`game-config-2`使用以下命令显示ConfigMap的详细信息：

```shell
kubectl describe configmaps game-config-2
```

输出类似于以下内容：

```
Name:         game-config-2
Namespace:    default
Labels:       <none>
Annotations:  <none>

Data
====
game.properties:
----
enemies=aliens
lives=3
enemies.cheat=true
enemies.cheat.level=noGoodRotten
secret.code.passphrase=UUDDLRLRBABAS
secret.code.allowed=true
secret.code.lives=30
ui.properties:
----
color.good=purple
color.bad=yellow
allow.textmode=true
how.nice.to.look=fairlyNice
```

## 2.4 从生成器创建ConfigMap

​        `kubectl``kustomization.yaml`自1.14开始支持。您还可以从生成器创建ConfigMap，然后将其应用于在Apiserver上创建对象。生成器应在`kustomization.yaml`目录内部指定。

**从文件生成ConfigMap**

例如，从文件生成ConfigMap `configure-pod-container/configmap/game.properties`

```shell
# Create a kustomization.yaml file with ConfigMapGenerator
cat <<EOF >./kustomization.yaml
configMapGenerator:
- name: game-config-4
  files:
  - configure-pod-container/configmap/game.properties
EOF
```

应用kustomization目录创建ConfigMap对象。

```shell
kubectl apply -k .
configmap/game-config-4-m9dm2f92bt created
```

您可以检查ConfigMap是按以下方式创建的：

```shell
kubectl get configmap
NAME                       DATA   AGE
game-config-4-m9dm2f92bt   1      37s


kubectl describe configmaps/game-config-4-m9dm2f92bt
Name:         game-config-4-m9dm2f92bt
Namespace:    default
Labels:       <none>
Annotations:  kubectl.kubernetes.io/last-applied-configuration:
                {"apiVersion":"v1","data":{"game.properties":"enemies=aliens\nlives=3\nenemies.cheat=true\nenemies.cheat.level=noGoodRotten\nsecret.code.p...

Data
====
game.properties:
----
enemies=aliens
lives=3
enemies.cheat=true
enemies.cheat.level=noGoodRotten
secret.code.passphrase=UUDDLRLRBABAS
secret.code.allowed=true
secret.code.lives=30
Events:  <none>
```

​        请注意，生成的ConfigMap名称具有对内容进行哈希处理后附加的后缀。这样可以确保每次修改内容时都会生成一个新的ConfigMap。