# 1. Configmap 简单介绍

​        ConfigMap 是一种 API 对象，用来将`非机密性的数据`保存到键值对中。使用时， [Pods](https://kubernetes.io/docs/concepts/workloads/pods/pod-overview/) 可以将其用作环境变量、命令行参数或者存储卷中的配置文件。

​         ConfigMap 将您的环境配置信息和 [容器镜像](https://kubernetes.io/zh/docs/reference/glossary/?all=true#term-image) 解耦，便于应用配置的修改。

> **注意：**
>
> ConfigMap 并不提供保密或者加密功能。 如果你想存储的数据是机密的，请使用 [Secret](https://kubernetes.io/zh/docs/concepts/configuration/secret/)， 或者使用其他第三方工具来保证你的数据的私密性，而不是用 ConfigMap。

**作用**

​    使用 ConfigMap 来将你的配置数据和应用程序代码分开。

​        在 ConfigMap 中保存的数据不可超过 1 MiB。如果你需要保存超出此尺寸限制的数据，你可能希望考虑挂载存储卷 或者使用独立的数据库或者文件服务。

# 2. 创建一个ConfigMap

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

# 3 ConfigMaps 和 Pods 

你可以写一个引用 ConfigMap 的 Pod 的 `spec`，并根据 ConfigMap 中的数据 在该 Pod 中配置容器。这个 Pod 和 ConfigMap 必须要在同一个 [名字空间](https://kubernetes.io/zh/docs/concepts/overview/working-with-objects/namespaces/) 中。

这是一个 ConfigMap 的示例，它的一些键只有一个值，其他键的值看起来像是 配置的片段格式。

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: game-demo
data:
  # 类属性键；每一个键都映射到一个简单的值
  player_initial_lives: "3"
  ui_properties_file_name: "user-interface.properties"

  # 类文件键
  game.properties: |
    enemy.types=aliens,monsters
    player.maximum-lives=5    
  user-interface.properties: |
    color.good=purple
    color.bad=yellow
    allow.textmode=true    
```

你可以使用四种方式来使用 ConfigMap 配置 Pod 中的容器：

1. 在容器命令和参数内
2. 容器的环境变量
3. 在只读卷里面添加一个文件，让应用来读取
4. 编写代码在 Pod 中运行，使用 Kubernetes API 来读取 ConfigMap

这些不同的方法适用于不同的数据使用方式。 对前三个方法，[kubelet](https://kubernetes.io/docs/reference/generated/kubelet) 使用 ConfigMap 中的数据在 Pod 中启动容器。

第四种方法意味着你必须编写代码才能读取 ConfigMap 和它的数据。然而， 由于你是直接使用 Kubernetes API，因此只要 ConfigMap 发生更改，你的 应用就能够通过订阅来获取更新，并且在这样的情况发生的时候做出反应。 通过直接进入 Kubernetes API，这个技术也可以让你能够获取到不同的名字空间 里的 ConfigMap。

下面是一个 Pod 的示例，它通过使用 `game-demo` 中的值来配置一个 Pod：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: configmap-demo-pod
spec:
  containers:
    - name: demo
      image: alpine
      command: ["sleep", "3600"]
      env:
        # 定义环境变量
        - name: PLAYER_INITIAL_LIVES # 请注意这里和 ConfigMap 中的键名是不一样的
          valueFrom:
            configMapKeyRef:
              name: game-demo           # 这个值来自 ConfigMap
              key: player_initial_lives # 需要取值的键
        - name: UI_PROPERTIES_FILE_NAME
          valueFrom:
            configMapKeyRef:
              name: game-demo
              key: ui_properties_file_name
      volumeMounts:
      - name: config
        mountPath: "/config"
        readOnly: true
  volumes:
    # 你可以在 Pod 级别设置卷，然后将其挂载到 Pod 内的容器中
    - name: config
      configMap:
        # 提供你想要挂载的 ConfigMap 的名字
        name: game-demo
        # 来自 ConfigMap 的一组键，将被创建为文件
        items:
        - key: "game.properties"
          path: "game.properties"
        - key: "user-interface.properties"
          path: "user-interface.properties"
```

ConfigMap 不会区分单行属性值和多行类似文件的值，重要的是 Pods 和其他对象 如何使用这些值。

上面的例子定义了一个卷并将它作为 `/config` 文件夹挂载到 `demo` 容器内， 创建两个文件，`/config/game.properties` 和 `/config/user-interface.properties`， 尽管 ConfigMap 中包含了四个键。 这是因为 Pod 定义中在 `volumes` 节指定了一个 `items` 数组。 如果你完全忽略 `items` 数组，则 ConfigMap 中的每个键都会变成一个与 该键同名的文件，因此你会得到四个文件。

# 4. 使用 ConfigMap

ConfigMap 可以作为数据卷挂载。ConfigMap 也可被系统的其他组件使用，而不一定直接暴露给 Pod。例如，

ConfigMap 可以保存系统中其他组件要使用 的配置数据。

ConfigMap 最常见的用法是为同一命名空间里某 Pod 中运行的容器执行配置。 你也可以单独使用 ConfigMap。

比如，你可能会遇到基于 ConfigMap 来调整其行为的 [插件](https://kubernetes.io/zh/docs/concepts/cluster-administration/addons/) 或者 [operator](https://kubernetes.io/zh/docs/concepts/extend-kubernetes/operator/)。

### 在 Pod 中将 ConfigMap 当做文件使用

1. 创建一个 ConfigMap 对象或者使用现有的 ConfigMap 对象。多个 Pod 可以引用同一个 ConfigMap。
2. 修改 Pod 定义，在 `spec.volumes[]` 下添加一个卷。 为该卷设置任意名称，之后将 `spec.volumes[].configMap.name` 字段设置为对 你的 ConfigMap 对象的引用。
3. 为每个需要该 ConfigMap 的容器添加一个 `.spec.containers[].volumeMounts[]`。 设置 `.spec.containers[].volumeMounts[].readOnly=true` 并将 `.spec.containers[].volumeMounts[].mountPath` 设置为一个未使用的目录名， ConfigMap 的内容将出现在该目录中。
4. 更改你的镜像或者命令行，以便程序能够从该目录中查找文件。ConfigMap 中的每个 `data` 键会变成 `mountPath` 下面的一个文件名。

下面是一个将 ConfigMap 以卷的形式进行挂载的 Pod 示例：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  containers:
  - name: mypod
    image: redis
    volumeMounts:
    - name: foo
      mountPath: "/etc/foo"
      readOnly: true
  volumes:
  - name: foo
    configMap:
      name: myconfigmap
```

你希望使用的每个 ConfigMap 都需要在 `spec.volumes` 中被引用到。

如果 Pod 中有多个容器，则每个容器都需要自己的 `volumeMounts` 块，但针对 每个 ConfigMap，你只需要设置一个 `spec.volumes` 块。

#### 被挂载的 ConfigMap 内容会被自动更新

当卷中使用的 ConfigMap 被更新时，所投射的键最终也会被更新。 kubelet 组件会在每次周期性同步时检查所挂载的 ConfigMap 是否为最新。 不过，kubelet 使用的是其本地的高速缓存来获得 ConfigMap 的当前值。 高速缓存的类型可以通过 [KubeletConfiguration 结构](https://github.com/kubernetes/kubernetes/blob/master/staging/src/k8s.io/kubelet/config/v1beta1/types.go) 的 `ConfigMapAndSecretChangeDetectionStrategy` 字段来配置。

ConfigMap 既可以通过 watch 操作实现内容传播（默认形式），也可实现基于 TTL 的缓存，还可以直接经过所有请求重定向到 API 服务器。 因此，从 ConfigMap 被更新的那一刻算起，到新的主键被投射到 Pod 中去，这一 时间跨度可能与 kubelet 的同步周期加上高速缓存的传播延迟相等。 这里的传播延迟取决于所选的高速缓存类型 （分别对应 watch 操作的传播延迟、高速缓存的 TTL 时长或者 0）。

以环境变量方式使用的 ConfigMap 数据不会被自动更新。 更新这些数据需要重新启动 Pod。

## 不可变更的 ConfigMap

**FEATURE STATE:** `Kubernetes v1.19 [beta]`

Kubernetes Beta 特性 *不可变更的 Secret 和 ConfigMap* 提供了一种将各个 Secret 和 ConfigMap 设置为不可变更的选项。对于大量使用 ConfigMap 的 集群（至少有数万个各不相同的 ConfigMap 给 Pod 挂载）而言，禁止更改 ConfigMap 的数据有以下好处：

- 保护应用，使之免受意外（不想要的）更新所带来的负面影响。
- 通过大幅降低对 kube-apiserver 的压力提升集群性能，这是因为系统会关闭 对已标记为不可变更的 ConfigMap 的监视操作。

此功能特性由 `ImmutableEphemeralVolumes` [特性门控](https://kubernetes.io/zh/docs/reference/command-line-tools-reference/feature-gates/) 来控制。你可以通过将 `immutable` 字段设置为 `true` 创建不可变更的 ConfigMap。 例如：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  ...
data:
  ...
immutable: true
```

一旦某 ConfigMap 被标记为不可变更，则 *无法* 逆转这一变化，，也无法更改 `data` 或 `binaryData` 字段的内容。你只能删除并重建 ConfigMap。 因为现有的 Pod 会维护一个对已删除的 ConfigMap 的挂载点，建议重新创建 这些 Pods。