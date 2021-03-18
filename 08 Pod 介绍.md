# 1. Pod 介绍

1. 上面是pod

   Pod 是可以在 Kubernetes 中创建和管理的、最小的可部署的计算单元。

   Pod 是一组（一个或多个）容器, 每个Pod还包含了一个Pause容器，Pause容器是Pod的父容器，主要负责僵尸进程的回收管理，通过Pause容器可以使同一个Pod里面的多个容器共享存储、网络、PID、IPC等。

# 2. Pod 定义和使用

```json
apiVersion: v1 # 必选，API的版本号
kind: Pod       # 必选，类型Pod
metadata:       # 必选，元数据
  name: nginx   # 必选，符合RFC 1035规范的Pod名称
  # namespace: default # 可选，Pod所在的命名空间，不指定默认为default，可以使用-n 指定namespace 
  labels:       # 可选，标签选择器，一般用于过滤和区分Pod
    app: nginx
    role: frontend # 可以写多个
  annotations:  # 可选，注释列表，可以写多个
    app: nginx
spec:   # 必选，用于定义容器的详细信息
#  initContainers: # 初始化容器，在容器启动之前执行的一些初始化操作
#  - command:
#    - sh
#    - -c
#    - echo "I am InitContainer for init some configuration"
#    image: busybox
#    imagePullPolicy: IfNotPresent
#    name: init-container
  containers:   # 必选，容器列表
  - name: nginx # 必选，符合RFC 1035规范的容器名称
    image: nginx:1.15.2    # 必选，容器所用的镜像的地址
    imagePullPolicy: IfNotPresent     # 可选，镜像拉取策略, IfNotPresent: 如果宿主机有这个镜像，那就不需要拉取了. Always: 总是拉取, Never: 不管是否存储都不拉去
    command: # 可选，容器启动执行的命令 ENTRYPOINT, arg --> cmd
    - nginx 
    - -g
    - "daemon off;"
    workingDir: /usr/share/nginx/html       # 可选，容器的工作目录
#    volumeMounts:   # 可选，存储卷配置，可以配置多个
#    - name: webroot # 存储卷名称
#      mountPath: /usr/share/nginx/html # 挂载目录
#      readOnly: true        # 只读
    ports:  # 可选，容器需要暴露的端口号列表
    - name: http    # 端口名称
      containerPort: 80     # 端口号
      protocol: TCP # 端口协议，默认TCP
    env:    # 可选，环境变量配置列表
    - name: TZ      # 变量名
      value: Asia/Shanghai # 变量的值
    - name: LANG
      value: en_US.utf8
#    resources:      # 可选，资源限制和资源请求限制
#      limits:       # 最大限制设置
#        cpu: 1000m
#        memory: 1024Mi
#      requests:     # 启动所需的资源
#        cpu: 100m
#        memory: 512Mi
#    startupProbe: # 可选，检测容器内进程是否完成启动。注意三种检查方式同时只能使用一种。
#      httpGet:      # httpGet检测方式，生产环境建议使用httpGet实现接口级健康检查，健康检查由应用程序提供。
#            path: /api/successStart # 检查路径
#            port: 80
#    readinessProbe: # 可选，健康检查。注意三种检查方式同时只能使用一种。
#      httpGet:      # httpGet检测方式，生产环境建议使用httpGet实现接口级健康检查，健康检查由应用程序提供。
#            path: / # 检查路径
#            port: 80        # 监控端口
#    livenessProbe:  # 可选，健康检查
      #exec:        # 执行容器命令检测方式
            #command: 
            #- cat
            #- /health
    #httpGet:       # httpGet检测方式
    #   path: /_health # 检查路径
    #   port: 8080
    #   httpHeaders: # 检查的请求头
    #   - name: end-user
    #     value: Jason 
#      tcpSocket:    # 端口检测方式
#            port: 80
#      initialDelaySeconds: 60       # 初始化时间
#      timeoutSeconds: 2     # 超时时间
#      periodSeconds: 5      # 检测间隔
#      successThreshold: 1 # 检查成功为2次表示就绪
#      failureThreshold: 2 # 检测失败1次表示未就绪
#    lifecycle:
#      postStart: # 容器创建完成后执行的指令, 可以是exec httpGet TCPSocket
#        exec:
#          command:
#          - sh
#          - -c
#          - 'mkdir /data/ '
#      preStop:
#        httpGet:      
#              path: /
#              port: 80
      #  exec:
      #    command:
      #    - sh
      #    - -c
      #    - sleep 9
  restartPolicy: Always   # 可选，默认为Always，容器故障或者没有启动成功，那就自动该容器，Onfailure: 容器以不为0的状态终止，自动重启该容器, Never:无论何种状态，都不会重启
  #nodeSelector: # 可选，指定Node节点
  #      region: subnet7
#  imagePullSecrets:     # 可选，拉取镜像使用的secret，可以配置多个
#  - name: default-dockercfg-86258
#  hostNetwork: false    # 可选，是否为主机模式，如是，会占用主机端口
#  volumes:      # 共享存储卷列表
#  - name: webroot # 名称，与上述对应
#    emptyDir: {}    # 挂载目录
#        #hostPath:              # 挂载本机目录
#        #  path: /etc/hosts
```

# 3. Pod探针

1. StartupProbe(启动探针)：k8s1.16版本后新加的探测方式，用于判断容器内应用程序是否已经启动。如果配置了startupProbe，就会先禁止其他的探测，直到它成功为止，成功后将不在进行探测。

2. LivenessProbe(存活探针)：用于探测容器是否运行，如果探测失败，kubelet会根据配置的重启策略进行相应的处理。若没有配置该探针，默认就是success。

3. ReadinessProbe(就绪探针)：一般用于探测容器内的程序是否健康，它的返回值如果为success，那么久代表这个容器已经完成启动，并且程序已经是可以接受流量的状态。

   

[kubelet](https://kubernetes.io/zh/docs/reference/command-line-tools-reference/kubelet/) 使用存活探测器来知道什么时候要重启容器。 例如，存活探测器可以捕捉到死锁（应用程序在运行，但是无法继续执行后面的步骤）。 这样的情况下重启容器有助于让应用程序在有问题的情况下更可用。

kubelet 使用就绪探测器可以知道容器什么时候准备好了并可以开始接受请求流量， 当一个 Pod 内的所有容器都准备好了，才能把这个 Pod 看作就绪了。 这种信号的一个用途就是控制哪个 Pod 作为 Service 的后端。 在 Pod 还没有准备好的时候，会从 Service 的负载均衡器中被剔除的。

kubelet 使用启动探测器可以知道应用程序容器什么时候启动了。 如果配置了这类探测器，就可以控制容器在启动成功后再进行存活性和就绪检查， 确保这些存活、就绪探测器不会影响应用程序的启动。 这可以用于对慢启动容器进行存活性检测，避免它们在启动运行之前就被杀掉。

> 注意:
>
> 一般情况LivenessProbe和ReadinessProbe可以配置相同的 http探测，但是一些存在数据初始化的情况需要等待初始化完成(ReadinessProbe)。这时在加上ReadinessProbe 的 http探测就能够保证容器可以处理请求

# 4. Pod探针的检测方式

- ExecAction：在容器内执行一个命令，如果返回值为0，则认为容器健康。
- TCPSocketAction：通过TCP连接检查容器内的端口是否是通的，如果是通的就认为容器健康。
- HTTPGetAction：通过应用程序暴露的API地址来检查程序是否是正常的，如果状态码为200~400之间，则认为容器健康。

下面以 `LivenessProbe`  介绍三种检测方式

**ExecAction** 

```json
apiVersion: v1
kind: Pod
metadata:
  labels:
    test: liveness
  name: liveness-exec
spec:
  containers:
  - name: liveness
    image: k8s.gcr.io/busybox
    args:
    - /bin/sh
    - -c
    - touch /tmp/healthy; sleep 30; rm -rf /tmp/healthy; sleep 600
    livenessProbe:
      exec:
        command:
        - cat
        - /tmp/healthy
      initialDelaySeconds: 5
      periodSeconds: 5
```

​        在这个配置文件中，可以看到 Pod 中只有一个容器。 `periodSeconds` 字段指定了 kubelet 应该每 5 秒执行一次存活探测。 `initialDelaySeconds` 字段告诉 kubelet 在执行第一次探测前应该等待 5 秒。 kubelet 在容器内执行命令 `cat /tmp/healthy` 来进行探测。 如果命令执行成功并且返回值为 0，kubelet 就会认为这个容器是健康存活的。 如果这个命令返回非 0 值，kubelet 会杀死这个容器并重新启动它。

**TCPSocketAction**

```json
apiVersion: v1
kind: Pod
metadata:
  name: goproxy
  labels:
    app: goproxy
spec:
  containers:
  - name: goproxy
    image: k8s.gcr.io/goproxy:0.1
    ports:
    - containerPort: 8080
    readinessProbe:
      tcpSocket:
        port: 8080
      initialDelaySeconds: 5
      periodSeconds: 10
    livenessProbe:
      tcpSocket:
        port: 8080
      initialDelaySeconds: 15
      periodSeconds: 20
```

​        这个例子同时使用就绪和存活探测器。kubelet 会在容器启动 5 秒后发送第一个就绪探测。 这会尝试连接 `goproxy` 容器的 8080 端口。 如果探测成功，这个 Pod 会被标记为就绪状态，kubelet 将继续每隔 10 秒运行一次检测。

​        除了就绪探测，这个配置包括了一个存活探测。 kubelet 会在容器启动 15 秒后进行第一次存活探测。 就像就绪探测一样，会尝试连接 `goproxy` 容器的 8080 端口。 如果存活探测失败，这个容器会被重新启动。

**HTTPGetAction**

```json
apiVersion: v1
kind: Pod
metadata:
  labels:
    test: liveness
  name: liveness-http
spec:
  containers:
  - name: liveness
    image: k8s.gcr.io/liveness
    args:
    - /server
    livenessProbe:
      httpGet:
        path: /healthz
        port: 8080
        httpHeaders:
        - name: Custom-Header
          value: Awesome
      initialDelaySeconds: 3
      periodSeconds: 3
```

​        在这个配置文件中，可以看到 Pod 也只有一个容器。 `periodSeconds` 字段指定了 kubelet 每隔 3 秒执行一次存活探测。 `initialDelaySeconds` 字段告诉 kubelet 在执行第一次探测前应该等待 3 秒。 kubelet 会向容器内运行的服务（服务会监听 8080 端口）发送一个 HTTP GET 请求来执行探测。 如果服务器上 `/healthz` 路径下的处理程序返回成功代码，则 kubelet 认为容器是健康存活的。 如果处理程序返回失败代码，则 kubelet 会杀死这个容器并且重新启动它。

任何大于或等于 200 并且小于 400 的返回代码标示成功，其它返回代码都标示失败。

可以在这里看服务的源码 [server.go](https://github.com/kubernetes/kubernetes/blob/master/test/images/agnhost/liveness/server.go)。

容器存活的最开始 10 秒中，`/healthz` 处理程序返回一个 200 的状态码。之后处理程序返回 500 的状态码。

```go
http.HandleFunc("/healthz", func(w http.ResponseWriter, r *http.Request) {
    duration := time.Now().Sub(started)
    if duration.Seconds() > 10 {
        w.WriteHeader(500)
        w.Write([]byte(fmt.Sprintf("error: %v", duration.Seconds())))
    } else {
        w.WriteHeader(200)
        w.Write([]byte("ok"))
    }
})
```

​        kubelet 在容器启动之后 3 秒开始执行健康检测。所以前几次健康检查都是成功的。 但是 10 秒之后，健康检查会失败，并且 kubelet 会杀死容器再重新启动容器。

创建一个 Pod 来测试 HTTP 的存活检测：

```shell
kubectl apply -f https://k8s.io/examples/pods/probe/http-liveness.yaml
```

10 秒之后，通过看 Pod 事件来检测存活探测器已经失败了并且容器被重新启动了。

```shell
kubectl describe pod liveness-http
```

# 5.  探针检查参数配置

initialDelaySeconds: 60    # 初始化时间 等待多少秒后存活和就绪探测器才被初始化，默认是 0 秒，最小值是 0。

timeoutSeconds: 2   # 超时时间（单位是秒）。默认是 1 秒。最小值是 1。

periodSeconds: 5   # 检测间隔（单位是秒）。默认是 10 秒。最小值是 1。

successThreshold: 1 # 检查成功为1次表示就绪  默认值是，最小值是 1。

failureThreshold: 2 # 检测失败2次表示未就绪，存活探测情况下的失败就意味着重新启动容器。 就绪探测情况下的失败的 Pod 会被打上未就绪的标签。默认值是 3。最小值是 1。