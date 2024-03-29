---
title: Kubernetes 入门 （2）
date: 2022-08-20 21:56:52
tags:
  - Kubernetes
  - DevOps
  - Docker
  - Cloud Native
---

我们之前说的都是用于部署 Pod 的资源，我们接下来介绍与创建 Pod 不相关的资源：储存与网络。

# 储存

其实我们之前已经接触过储存相关的内容了：在讲 Stateful Set 时我们提过 Stateful Set 创建出来的 Pod 都会有相互独立的储存；而讲 Daemon Set 时我们提到 K8s 推荐只在 Daemon Set 的 Pod 中访问宿主机磁盘。但独立的储存具体指什么？除了访问宿主机磁盘以外还有什么其他的储存？

在 Docker 中，我们可以把宿主机磁盘上的一个路径作为一个 Volume 来给容器绑定，或者直接使用 Docker Engine 管理的 Volume 来提供持久化存储或是容器间共享文件。在 K8s 里面也沿用了 Volume 这个概念，可以通过 Mount 绑定到容器内的路径，并通过实现 CSI 的各种引擎来提供更多样的存储。

> CSI: Container Storage Interface ，容器储存接口标准，是 K8s 提出的一种规范。不管是哪种储存引擎，只要编写一个对应的插件实现 CSI ，都可以在 K8s 中使用。

### K8s 中使用 Volume 与可用的 Volume 类型

其实 K8s 中使用 Volume 的例子我们一开始就已经接触过了。还记得一开始介绍 Pod 时的 Nginx 例子吗？

```yaml
metadata:
  name: simple-webapp
spec:
  containers:
    - name: main-application
      image: nginx
      volumeMounts:
        - name: shared-logs
          mountPath: /var/log/nginx
    - name: sidecar-container
      image: busybox
      command: ["sh","-c","while true; do cat /var/log/nginx/access.log; sleep 30; done"]
      volumeMounts:
        - name: shared-logs
          mountPath: /var/log/nginx
  volumes:
    - name: shared-logs
      emptyDir: {}
```

这个 Pod 描述中声明了一个种类为 `emptyDir` 的，名为 `shared-logs` 的 Volume ，然后 Pod 中的两个容器都分别 Mount 了这个 Volume 。

K8s 中默认提供了几种 Volume ，比如：

- emptyDir ：一个简单的空目录，一般用于储存临时数据或是 Pod 的容器之间共享数据。
- hostPath ：绑定到节点宿主机文件系统上的路径，一般在 Daemon Set 中使用。
- gitRepo ：这种 Volume 其实相当于 emptyDir ，不过在 Pod 启动时会从 Git 仓库 clone 一份内容作为默认数据。
- configMap 、 secret ：一般用于配置文件加载，需要与 configMap 、 secret 这两种资源一同使用。会将 configMap 、 secret 中对应的内容拷贝一份作为 Volume 绑到容器。（下一节中会展开讨论）
- nfs 、 glusterfs 、 ……：可以通过各种网络存储协议直接挂载一个网络存储
- (deprecated!) gcePersistentDisk 、 awsElasticBlockStore ……：可以调用各个云平台的 API ，创建一个块储存硬件挂载到宿主机上，再将那个硬件挂载到容器中。
- persistentVolumeClaim ：持久卷声明，用于把实际储存方式抽象化，使得 Pod 不需要关心具体的储存类型。这种类型会在下面详细介绍。

我们可以注意到， Volume 的声明是 Pod 的一个属性，而不是一种单独的资源。 Volume 是 Pod 的一部分，因此不同的 Pod 之间永远不可能共享同一个 Volume 。

> 但是 Volume 所指向的位置可以相同，比如 HostPath 类型的 Volume 就可以两个 Pod 可以绑定到宿主机上同一个路径，因此 Volume 里的数据还是能通过一定方式在 Pod 间共享。但当然 K8s 不推荐这么做。

另外，由于 Volume 是 Pod 的一部分， Volume 的生命周期也是跟随 Pod 的，当一个 Pod 被销毁时， Volume 也会被销毁，因此最主要还是用于 Pod 内容器间的文件共享。如果需要持久化储存，需要使用 Persistent Volume 。

> Volume 会被销毁不代表 Volume 指向的内容会被销毁。比如 hostPath 、 NFS 等类型 Volume 中的内容就会继续保留在宿主机或是 NAS 上。下面提到的 Presistent Volume Claim 也是，拥有 `persistentVolumeClaim` 类型 Volume 的 Pod 被删除后对应的 PVC 不一定会被删除。

### Presistent Volume 、 Presistent Volume Claim 、 Storage Class

如果需要在 Pod 声明中直接指定 NFS 、 awsElasticBlockStore 之类的信息，就需要应用的开发人员对真实可用的储存结构有所理解，违背了 K8s 的理念。因此 K8s 就弄出了小标题中的三种资源来将储存抽象化。

一个 Persistent Volume (PV) 对应云平台提供的一个块存储，或是 NAS 上的一个路径。可以简单地理解为 **PV 直接描述了一块可用的物理存储** 。因为 PV 直接对应到硬件，因此 PV 跟节点一样，是名称空间无关的。

而一个 **Persistent Volume Claim (PVC) 则是描述了怎样去使用储存** ：使用多少空间、只读还是读写等。一个 PVC 被创建后会且只会对应到一个 PV 。 PVC 从属于一个名称空间，并能被该名称空间下的 Pod 指定为一个 Volume 。

PV 与 PVC 这两种抽象是很必要的。试想一下用自己的物理机搭建一个 K8s 集群的场景。你会提前给物理机插上许多个储存硬件，这时你就需要用 PV 来描述这些硬件，之后才能在 K8s 里利用这些硬件的储存。而实际将应用部署到 K8s 中时，你才需要用 PVC 来描述 Pod 中需要怎么样的储存卷，然后 K8s 就会自动挑一个合适 PV 给这个 PVC 绑定上。这样实际部署应用的时候就不用再特意跑去机房给物理机插硬件了。

但是现在都云原生时代了，各供应商都有提供 API 可以直接创建一个块储存，还要想办法提前准备 PV 实在是太蠢了。于是便需要 Storage Class 这种资源。

使用 Storage Class 前需要先安装各种云供应商提供的插件（当然使用云服务提供的 K8s 的话一般已经准备好了），然后再创建一个 Storage Class 类型的资源（当然一般也已经准备好了）。下面是 AWS 上的 EKS 服务中默认自带的 Storage Class ：

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
  name: gp2
provisioner: kubernetes.io/aws-ebs
parameters:
  fsType: ext4
  type: gp2
# 当 PVC 被删除时会同时删除 PV
reclaimPolicy: Delete
# 只有当 PVC 被绑定为一个 Pod 的 Volume 时才会创建一个 PV
volumeBindingMode: WaitForFirstConsumer
```

可以看到 EKS 自带的 gp2 提供了一些默认的选项，我们也可以类似地去定义自己的 Storage Class 。有了 gp2 这个 Storage Class ，我们创建一个 PVC 后 K8s 就会调用 AWS 的 API ，创建一个块储存接到我们的节点上，然后 K8s 再自动创建一个 PV 并绑定到 PVC 上。

例如，我们部署 Kafka 时会创建一个这样的 PVC ：

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data-kafka-0
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: gp2
```

K8s 就会自动为我们创建出一个对应的 PV ：

```sh
# `pvc-` 开头这个是 AWS 自动给我们起的名字。它虽然是 `pvc-` 开头，但他其实是一个 PV 。
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                STORAGECLASS   REASON   AGE
pvc-3614c15f-5697-4d66-a13c-6ddf7eb89998   10Gi       RWO            Delete           Bound    kafka/data-kafka-0   gp2                     152d
```

要是打开 AWS Console 还会发现， K8s 调用了 AWS 的 API ，自动为我们创建了一个 EBS 块储存并绑定到了我们对应的宿主机上。

可以用下面这张图来表示 Pod 中的 Volume 、 PVC 、 PV 之间的关系：

```mermaid
flowchart TD

subgraph Pod[Pod: Kafka-0]
subgraph Container[Container: docker.io/bitnami/kafka:3.1.0]
vm[VolumeMount: /bitnami/kafka]
end
volume[(Volume: data)]
vm --> volume
end

pvc[pvc: data-kafka-0]
pv[pv: pvc-3614c15f-5697-4d66-a13c-6ddf7eb89998]
ebs[ebs: AWS 为我们创建的块储存硬件]

volume --> pvc
pvc --> pv
pv --> ebs
```

而 Storage Class 在上图中则负责读取我们提交的 PVC ，然后创建 PV 与 EBS 。

### 再说回 Stateful Set

之前我们提到 Stateful Set 时说到 Stateful Set 创建的 Pod 拥有固定的储存，到底是什么意思呢？跟 Deployment 的储存又有什么区别呢？

我们先来看看，如果要给 Deployment 创建出来的 Pod 挂载 PVC 需要怎么做。下面是一个部署 Nginx 的 Deployment 清单，其中 html 目录下的静态文件存放在 NFS 里，通过 PVC 挂载到 Pod 中：

```yaml
# 这里省略了 Service 相关的内容
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-dpl-with-nfs-pvc
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
        ports:
        - containerPort: 80
          name: web
        volumeMounts: #挂载容器中的目录到 pvc nfs 中的目录
        - name: www
          mountPath: /usr/share/nginx/html
      volumes:
      - name: www
        persistentVolumeClaim: #指定pvc
          claimName: nfs-pvc-for-nginx
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nfs-pvc-for-nginx
  namespace: default
spec:
  storageclassname: "" # 指定使用现有 PV ，不使用 StorageClass 创建 PV
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
---
# 这个例子中需要挂载 NFS 上的特定路径，所以手动定义了一个 PV
# 一般情况下我们不会手动创建 PV，而是使用 StorageClass 自动创建
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-pv-for-nginx
spec:
  capacity: 
    storage: 1Gi
  accessModes:
  - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  nfs:
    path: /nfs/sharefolder/nginx
    server: 81.70.4.171
```

这份清单我们主要关注前两个资源，我们可以看到除了一个 Deployment 资源以外我们还单独定义了一个 PVC 资源。然后在 Deployment 的 Pod 模板中声明并绑定了这个 PVC 。

可这样 apply 了之后会发生什么情况呢？因为我们只声明了一份 PVC ，当然我们只会拥有一个 PVC 资源。但这个 Deployment 的副本数是 3 ，因此我们会有 3 个相同的 Pod 去绑定同一个 PVC 。也就是最终会在 3 个容器里访问同一个 NFS 的同一个目录。如果我们在其中一个容器里对这个目录作修改，也会影响到另外两个容器。

> 注：这一现象不一定在任何情况下都适用。比如 AWS 的 EBS 卷只支持单个 AZ 内的绑定。如果 Pod 因为 Node Affinity 等设定被部署到了多个区，没法绑定同一个 EBS 卷，就会在 Scedule 的阶段报错。

很多时候我们都不希望多个 Pod 绑定到同一 PVC 。比如我们部署一个 DB 集群的时候，如果好不容易部署出来的多个实例居然用的是同一份储存，就会显得很呆。 Stateful Set 就是为了解决这种情况，会为其管理下的每个 Pod 都部署一个专用的 PVC 。

下面是给 Stateful Set 创建出来的 Pod 挂载 PVC 的一份清单：

```yaml
# 这里省略了 Service 相关的内容
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: "nginx"
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: k8s.gcr.io/nginx-slim:0.8
        ports:
        - containerPort: 80
          name: web
        volumeMounts:
        - name: www
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
  - metadata:
      name: www
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 1Gi
```

我们可以看到，部署 Stateful Set 时我们不能另外单独定义一份 PVC 了，只能作为 Stateful Set 定义的一部分，在 volumeClaimTemplates 字段中定义 PVC 的模板。这样一来， Stateful Set 会根据这个模板，为每个 Pod 创建一个对应的 PVC ，并作为 Pod 的 Volume 绑定上：

```bash
# Stateful Set 创建出来的 Pod ，名字都是按顺序的
$ kubectl get pods -l app=nginx
NAME      READY     STATUS    RESTARTS   AGE
web-0     1/1       Running   0          1m
web-1     1/1       Running   0          1m

# Stateful Set 创建出来的 PVC ，名字与 Pod 的名字一一对应
$ kubectl get pvc -l app=nginx
NAME        STATUS    VOLUME                                     CAPACITY   ACCESSMODES   AGE
www-web-0   Bound     pvc-15c268c7-b507-11e6-932f-42010a800002   1Gi        RWO           48s
www-web-1   Bound     pvc-15c79307-b507-11e6-932f-42010a800002   1Gi        RWO           48s
```

这样， Stateful Set 的多个 Pod 就会拥有自己的储存，不会相互打架了。另外，如果我们事先定义了 StorageClass ，还能根据 Stateful Set 的副本数动态配置 PV 。

### ConfigMap 与 Secret 挂载作为特殊的卷

有时候我们需要使用配置文件来配置应用（比如 Nginx 的配置文件），而且我们有时候会需要不重启 Pod 就热更新配置。如果用 PVC 来加载配置文件略微麻烦，这时候可以使用 Config Map 。

下面是 K8s 官网上 Config Map 的一个例子：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: game-demo
data:
  # 一个 Key 可以对应一个值
  player_initial_lives: "3"
  ui_properties_file_name: "user-interface.properties"

  # 一个 Key 也可以对应一个文件的内容
  game.properties: |
    enemy.types=aliens,monsters
    player.maximum-lives=5    
  user-interface.properties: |
    color.good=purple
    color.bad=yellow
    allow.textmode=true    
---
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
        # ConfigMap 的 Key 可以作为环境变量引用
        - name: PLAYER_INITIAL_LIVES
          valueFrom:
            configMapKeyRef:
              name: game-demo           # 从这个 Config Map 里
              key: player_initial_lives # 拿到这个 key 的值
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
    # 定义 Pod 的 Volume ，种类为 configMap
    - name: config
      configMap:
        name: game-demo # ConfigMap的名字
        # 需要作为文件放入 Volume 的 Key
        items:
        - key: "game.properties"
          path: "game.properties"
        - key: "user-interface.properties"
          path: "user-interface.properties"
```

我们可以看到 ConfigMap 里的 Key 可以作为文件或是环境变量加载到 Pod 中。另外，作为环境变量加载后其实还能作为命令行参数传入应用，实现各种配置方式。如果修改 Config map 的内容，也可以自动更新 Pod 中的文件。

然而， Config Map 的热更新有一些不太灵活的地方：

1. 作为环境变量加载的 Config Map 数据不会被热更新。想要更新这一部分数据需要重启 Pod。（当然，命令行参数也不能热更新）
2. 由于 Kubelet 会先将 Config Map 内容加载到本地作为缓存，因此修改 Config Map 后新的内容不会第一时间加载到 Pod 中。而且在旧版本的 K8s 中， Config Map 被更新直到缓存被刷新的时间间隔还会很长，新版本的 K8s 这一部分有了优化，可以设定刷新时间，但会导致 API Server 的负担加重。（这其实是一个 Known Issue ，被诟病多年： https://github.com/kubernetes/kubernetes/issues/22368 ）

除 Config Map 以外， K8s 还提供了一种叫 Secret 的资源，用法和 Config Map 几乎一样。对比 Config Map ，Secret 有以下几个特点：

1. 在 Pod 里， Secret 只会被加载到内存中，而永远不会被写到磁盘上。
2. 用 `kubectl get` 之类的命令显示的 Secret 内容会被用 base64 编码。（不过， well ，众所周知 base64 可不算是什么加密）
3. 可以通过 K8s 的 Service Account 等 RBAC 相关的资源来控制 Secret 的访问权限。

不过，由于 Secret 也是以明文的形式被存储在 K8s 的主节点中的，因此需要保证 K8s 主节点的安全。

> **Downward API 挂载作为特殊的卷**
> 
> 还有另外一种叫 Downward API 的东西，可以作为 Volume 或是环境变量被加载到 Pod 中。有一些参数我们很难事先在 Manifest 中定义（ e.g. Deployment 生成的 Pod 的名字），因此可以通过 Downward API 来实现。
> 
> ```yaml
> apiVersion: v1
> kind: Pod
> metadata:
>     name: test-volume-pod
>     namespace: kube-system
>     labels:
>         k8s-app: test-volume
>         node-env: test
> spec:
>     containers:
>     - name: test-volume-pod-container
>       image: busybox:latest
>       env:
>       - name: POD_NAME # 将 Pod 的名字作为环境变量 POD_NAME 加载到 Pod 中
>         valueFrom:
>           fieldRef:
>             fieldPath: metadata.name
>       command: ["sh", "-c"]
>       args:
>       - while true; do
>           cat /etc/podinfo/labels | echo;
>           env | sort | echo;
>           sleep 3600;
>         done;
>       volumeMounts:
>       - name: podinfo
>         mountPath: /etc/podinfo
>     volumes:
>     - name: podinfo
>       downwardAPI: # Downward API 类型的卷
>         items:
>         - path: "labels" # 将 Pod 的标签作为  labels 文件挂载到 Pod 中
>           fieldRef:
>             fieldPath: metadata.labels
> ```



# 网络

其实 Pod 只要部署好了，就会被分配到一个集群内部的 IP 地址，流量就可以通过 IP 地址来访问 Pod 了。然而通过可能会有很大问题： **Pod 随时会被杀死。** 虽然通过用 Deployment 等资源可以在挂掉后重新创建一个 Pod ，但那毕竟是不同的 Pod ， IP 已经改变。

另外， Deployment 等资源的就是为了能更方便的做到多副本部署及任意缩容扩容而存在的。如果在 K8s 中访问 Pod 还需要小心翼翼地去找到 Pod 的 IP 地址，或是去寻找 Pod 是否部署了新副本， Deployment 等资源就几乎没有存在价值了。

> 其实 Pod 部署好后不止会被分配 IP 地址，还会被分配到一个类似 `<pod-ip>.<namespace>.pod.cluster.local` 的 DNS 记录。例如一个位于 default 名字空间，IP 地址为 172.17.0.3 的 Pod ，对应 DNS 记录为 `172-17-0-3.default.pod.cluster.local` 。

### Service

在古代，人们是通过注册中心、服务发现、负载均衡等中间件来解决上面这些问题的，但这样很不云原生。于是 K8s 引入了 Service 这种资源，来实现简易的服务发现、 DNS 功能。

下面是一个经典的例子，部署了一个 Service 和一个 Deployment：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: auth-service
  labels:
    app: auth
spec:
  type: ClusterIP
  selector:
    app: auth # 指向 Deployment 创建的 Pod
  ports:
  - port: 80 # Service 暴露的端口
    targetPort: 8080 # Pod 的端口
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name:  auth
  labels:
    app: auth
spec:
  replicas: 2
  selector:
    matchLabels:
      app: auth
  template:
    metadata:
      name: auth
      labels:
        app: auth
    spec:
      containers:
      - name: auth
        image: xxxxx.dkr.ecr.ap-northeast-1.amazonaws.com/auth:xxxxx
        ports:
        - containerPort: 8080
```

根据前面的知识我们知道，这份文件会部署 Deployment 会创建 2 个相同的 Pod 副本。另外还会部署一个名为 auth-service 的 Service 资源。这个 Service 暴露了一个 80 端口，并且指向那两个 Pod 的 8080 端口。

而这份文件部署后， Service 资源就会在集群中注册一个 DNS A 记录（或 AAAA 记录），集群内其他 Pod （为了辨别我们叫它 Client ）就可以通过相同的 DNS 名称来访问 Deployment 部署的这 2 个 Pod ：

```sh
curl http://auth-service.<namespace>.svc.cluster.local:80
# 或者省略掉后面的一大串
curl http://auth-service.<namespace>:80
# 如果 Client 和 Service 在同一个 Namespace 中，还可以：
curl http://auth-service:80
```

像这样 Client 通过 Service 来访问时，会随机访问到其中一个 Pod ，这样一来无论 Deployment 到底创建了多少个副本，只要副本的标签相同，就能通过同一个 DNS 名称来访问，还能自动实现一些简单的负载均衡。

> **为什么 DNS 名称可以简化？**
> 
> Pod 被部署时， kubelet 会为每个 Pod 注入一个类似如下的 `/etc/resolv.conf` 文件：
> 
> ```
> nameserver 10.32.0.10
> search <namespace>.svc.cluster.local svc.cluster.local cluster.local
> options ndots:5
> ```
> 
> Pod 中进行 DNS 查询时，默认会先读取这个文件，然后按照 `search` 选项中的内容展开 DNS 。例如，在 test 名称空间中的 Pod ，访问 data 时的查询可能被展开为 data.test.svc.cluster.local 。
> 更多关于 `/etc/resolv.conf` 文件的内容可参考 https://www.man7.org/linux/man-pages/man5/resolv.conf.5.html

### Service 的种类

我们上面的例子中，可以看到 Service 资源有个字段 `type:ClusterIP` 。其实 Service 资源有以下几个种类：

| 种类           | 作用                                                                                                                                                                                                                                                                                             |
| :------------- | :----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `ClusterIP`    | 这个类型的 Service 会在集群内创建一条 DNS A 记录并通过一定方法将流量代理到其指向的 Pod 上。这种 Service 不会暴露到集群外。这是最基础的 Service 种类。                                                                                                                                            |
| `NodePort`     | 这种 Service 会在 ClusterIP 的基础上，在所有节点上各暴露一个端口，并把端口的流量也代理到指向的 Pod 上。可以通过这种方法从集群外访问集群内的资源。                                                                                                                                                |
| `LoadBalancer` | 这种 Service 会在 ClusterIP 的基础上，在所有节点上各暴露一个端口，并在集群外创建一个负载均衡器来将外部流量路由到暴露的端口，再把流量代理到指向的 Pod 上。这种 Service 一般需要调用云服务提供的 API 或是额外安装的插件。如果什么插件都没安装的话，这种 Service 部署后会与 `NodePort` 的表现一样。 |
| `ExternalName` | 这种 Service 不需要 selector 字段指定后端，而是用 externalName 字段指定一个外部 DNS 记录，然后将流量全部指向外部服务。如果打算将集群内的服务迁移到集群外、或是集群外迁移到集群内，这种类型的 Service 可以实现无缝迁移。                                                                          |

### 虚拟 IP 与 Headless Service

如果你在集群内尝试对 Service 对应的 DNS 记录进行域名解析，会发现返回来的 IP 地址与 Service 指向的任何一个 Pod 对应的 IP 地址都不相同。如果你还尝试了去 Ping 这个 IP 地址，会发现不能 Ping 通。为什么会这样呢？

原来，每个 Service 被部署后， K8s 都会给他分配一个集群内部的 IP 地址，也就是 Cluster IP （这也是最基础的 Service 种类会起名叫 Cluster IP 的原因）。

但是这个 Cluster IP 不会绑定任何的网卡，是一个虚拟 IP 。然后 K8s 中有一个叫 kube-proxy 的组件（这里叫他做组件，是因为 kube-proxy 与 Service 、 Deployment 等不一样，不是一种资源而是 K8s 的一部分）， kube-proxy 通过修改 iptables ，将虚拟 IP 的流量经过一定的负载均衡规则后代理到 Pod 上。

![K8s 官网上的虚拟 IP 图](https://d33wubrfki0l68.cloudfront.net/27b2978647a8d7bdc2a96b213f0c0d3242ef9ce0/e8c9b/images/docs/services-iptables-overview.svg)

> **为什么不使用 DNS 轮询？**
> 
> 为什么 K8s 不配置多条 DNS A 记录，然后通过轮询名称来解析？为什么需要搞出虚拟 IP 这么复杂的东西？这个问题 K8s 官网上也有特别提到原因：
> 
> - DNS 实现的历史由来已久，它不遵守记录 TTL，并且在名称查找结果到期后对其进行缓存。
> - 有些应用程序仅执行一次 DNS 查找，并无限期地缓存结果。
> - 即使应用和库进行了适当的重新解析，DNS 记录上的 TTL 值低或为零也可能会给 DNS 带来高负载，从而使管理变得困难。

有些时候（比如想使用自己的服务发现机制或是自己的负载均衡机制时）我们确实也会想越过虚拟 IP ，直接获取背后 Pod 的 IP 地址。这时候我们可以将 Service 的 `spec.clusterIP` 字段指定为 `None` ，这样 K8s 就不会给这个 Service 分配一个 Cluster IP 。这样的 Service 被称为 **Headless Service** 。

Headless Service 资源会创建一组 A 记录直接指向背后的 Pod ，可以通过 DNS 轮询等方式直接获得其中一个 Pod 的 IP 地址。另外更重要的一点， Headless Service 还会创建一组 SRV 记录，包含了指向各个 Pod 的 DNS 记录，可以通过 SRV 记录来发现所有 Pod 。

我们可以在集群里用 nsloopup 或 dig 命令去验证一下：

```sh
# 在集群的 Pod 内部运行
$ nslookup kafka-headless.kafka.svc.cluster.local
Server:     10.96.0.10
Address:    10.96.0.10#53

Name:   kafka-headless.kafka.svc.cluster.local
Address: 172.17.0.6
Name:   kafka-headless.kafka.svc.cluster.local
Address: 172.17.0.5
Name:   kafka-headless.kafka.svc.cluster.local
Address: 172.17.0.4

$ dig SRV kafka-headless.kafka.svc.cluster.local
# .....
;; ANSWER SECTION:
kafka-headless.kafka.svc.cluster.local.      30      IN      SRV     0 20 9092 kafka-0.kafka-headless.kafka.svc.cluster.local.
kafka-headless.kafka.svc.cluster.local.      30      IN      SRV     0 20 9092 kafka-1.kafka-headless.kafka.svc.cluster.local.
kakfa-headless.kafka.svc.cluster.local.      30      IN      SRV     0 20 9092 kafka-2.kafka-headless.kafka.svc.cluster.local.

;; ADDITIONAL SECTION:
kafka-0.kafka-headless.kafka.svc.cluster.local. 30 IN A  172.17.0.6
kafka-1.kafka-headless.kafka.svc.cluster.local. 30 IN A  172.17.0.5
kafka-2.kafka-headless.kafka.svc.cluster.local. 30 IN A  172.17.0.4
```

> 拥有 Cluster IP 的 Service 其实也有 SRV 记录。但这种情况的 SRV 记录中对应的 Target 仍为 Service 自己的 FQDN 。

### 第三次回到 Stateful Set

在上面 Headless Service 的例子中，我们看到，各个 Pod 对应的 DNS A 记录格式为 `<pod_name>.<svc_name>.<namespace>.svc.cluster.local` 。不对啊，之前的小知识里不是说过 Pod 被分配的 DNS A 记录格式应该是 `172-17-0-3.default.pod.cluster.local` 的吗？

其实 Headless Service 还有一个众所周知的隐藏功能。 Pod 这种资源本身的参数中有 `subdomain` 字段和 `hostname` 字段，如果设置了这两个字段，这个 Pod 就拥有了形如 `<hostname>.<subdomain>.<namespace>.svc.cluster.local` 的 FQDN （全限定域名）。如果这时刚好在同一名称空间下有与 `subdomain` 同名的 Headless Service ， DNS 就会用为这个 Pod 用它的 FQDN 来创建一条 DNS A 记录。

比如 Pod1 在 `kafka` 名称空间中， `hostname` 为 `kafka-1` ， `subdomain` 为 `kafka-headless` ，那么 Pod1 的 FQDN 就是 `kafka-1.kafka-headless.kakfa.svc.cluster.local` 。而同样在 `kafka` 名称空间中，刚好又有一个 `kafka-headless` 的 Headless Service ，那么 DNS 就会创建一条 A 记录，就可以通过 `kafka-1.kafka-headless.kafka.svc.cluster.local` 来访问 Pod1 了。当然，由于 DNS 展开，也可以用 `kafka-1.kafka-headless.kafka` 甚至是 `kafka-1.kafka-headless` 来访问这个 Pod 。

其实这些 Pod 是用 Stateful Set 来部署的，这一部分其实是 Stateful Set 相关的功能。之前我们说到 Stateful Set 有唯一稳定的网络标识。我们现在就来详细讲讲，这“唯一稳定的网络标识”到底是在指什么。

我们来看一下这个 kafka Stateful Set 到底是怎么部署的：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: kafka-headless
spec:
  clusterIP: None # 这是一个 headless service
  ports:
  - name: tcp-client
    port: 9092
    protocol: TCP
    targetPort: kafka-client
  selector:
    select-label: kafka-label
  type: ClusterIP
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: kafka
spec:
  replicas: 3
  serviceName: kafka-headless # 注意到这里有 serviceName 字段
  selector:
    matchLabels:
      select-label: kafka-label
  template:
    metadata:
      labels:
        select-label: kafka-label
    spec:
      containers:
      - name: kafka
        image: docker.io/bitnami/kafka:3.1.0-debian-10-r52
        # 接下来 Pod 相关部分省略
  # 下面 Volume 相关部分也省略
```

我们看到， Stateful Set 的定义中必须要用 `spec.serviceName` 字段指定一个 Headless Service 。 Stateful Set 创建 Pod 时，会自动给 Pod 指定 `hostname` 和 `subdomain` 字段。这样一来，每个 Pod 才有了唯一固定的 hostname ，唯一固定的 FQDN ，以及通过与 Headless Service 共同部署而获得唯一固定的 A 记录。（此外，其实当 Pod 因为版本升级等原因被重新创建时，相同序号的 Pod 还会被分配到相同固定的集群内 IP 。）

> **关于 Stateful Set 中 `serviceName` 字段的争议**
> 
> Stateful Set 中的 serviceName 字段是必填字段。这个字段唯一的作用其实就是给 Pod 指定 subdomain 。其实这样会有一些问题：
> 
> 1. Stateful Set 部署时不会检查是否真的存在这么一个 Headless Service 。如果 serviceName 乱填一个值，会导致虽然 Pod 的 `hostname` 和 `subdomain` 都指定了却没有创建 A 记录的情况。
> 2. 有时 Stateful Set 的 Pod 不需要接收流量，也不需要相互发现，这时候还强行需要指定一个 serviceName 显得有点多余。
> 
> 在 GitHub 上有关于这个问题的 Issue ： https://github.com/kubernetes/kubernetes/issues/69608

### 从集群外部访问

在 K8s 集群里把应用部署好了，可是如何让集群外部的客户端访问我们集群中的应用呢？这可能是大家最关心的问题。

不过有认真听的同学估计已经有这个问题的答案了。之前我们讲过 NodePort 和 LoadBalancer 这两种 Service 类型。

其中 NodePort Service 只是简单地在节点机器上各开一个端口，而如何路由、如何负载均衡等则一概不管。

而 LoadBalancer Service 则是在 NodePort 的基础上再加一个一个负载均衡器，然后把节点暴露的端口注册到这个负载均衡器上。这样一来，集群外部的客户端就可以通过同一个 IP 来访问集群中的应用。但是要使用 LoadBalancer Service ，一般需要先安装云供应商提供的 Controller ，或是安装其他第三方的 Controller （比如 Nginx Controller ）。

在 Service 之外还另有一种资源类型叫 Ingress ，也可以用来实现集群外部访问集群内部应用的功能。 Ingress 其实也会在集群外创建一个负载均衡器，因此也需要预先安装云供应商的 Controller 。但 Ingress 与 Service 不同的是，它还会管理一定的路由逻辑，接收流量后可以根据路由来分配给不同的 Service 。

| 类型                 | OSI 模型工作层数 | 依赖于云平台或其他插件 |
| :------------------- | :--------------- | :--------------------- |
| NodePort Service     | 第四层           | 否                     |
| LoadBalancer Service | 第四层           | 是                     |
| Ingress              | 第七层           | 是                     |

特别再详细说一下 Ingress 这种资源。 Ingress 本身不会在集群内的 DNS 上创建记录，一般也不会主动去路由集群内的流量（除非你在集群内强行访问 Ingress 的负载均衡器…… 不过一般也没什么理由要这样做对吧）。但 Ingress 可以根据 HTTP 的 hostname 和 path 来路由流量，把流量分发到不同的 Service 上。 Ingress 也是 K8s 的原生资源里唯一能看到 OSI 第七层的资源。

下面是 AWS 的 EKS 服务中部署的一个 Ingress 的例子（集群中已安装 AWS Load Balancer Controller ）：

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/backend-protocol-version: GRPC
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTPS":443}]'
    alb.ingress.kubernetes.io/healthcheck-path: /grpc.health.v1.Health/Check
    alb.ingress.kubernetes.io/healthcheck-protocol: HTTP
    alb.ingress.kubernetes.io/success-codes: 0,12
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:xxxxxxxxxx:certificate/xxxxxxxxxx

    external-dns.alpha.kubernetes.io/hostname: sample.example.com
  
  name: gateway-ingress
spec:
  rules:
  - host: sample.example.com
    http:
      paths:
      - path: /grpc.health.v1.Health
        pathType: Prefix
        backend:
          service:
            name: health-service
            port:
              number: 50051
      - path: /proto.sample.v1.Sample
        pathType: Prefix
        backend:
          service:
            name: sample-service
            port:
              number: 50051
```

可以看到， Ingress 资源可以通过 `spec.rules` 字段中定义各条规则，通过 hostname 或是 path 等第七层的信息来进行路由。 Ingress 部署下去后， AWS Load Balancer Controller 会读取会根据的配置，并在云上创建一个 AWS Application Load Balancer （ALB），而 `spec.rules` 会应用到 ALB 上，由 ALB 来负责流量的路由。

我们也会注意到，怎么 `metadata.annotations` 里有这么多奇奇怪怪的字段！ Ingress 本身的功能都是 AWS Load Balancer Controller 调用 AWS 的 API 创建 ALB 来实现的。但 AWS 的 ALB 能实现的功能可不止 Ingress 字段定义的这些，比如安装 TLS 证书、 health check 等 spec 字段中描述不下的功能，就只能是通过 annotation 的形式来定义了。

> 小彩蛋：可以看到例子中的 Ingress 资源 annotation 字段里还有一行 `external-dns.alpha.kubernetes.io/hostname: sample.example.com` 。其实这个 K8s 集群中还安装了 external-dns 这个应用，它可以根据 annotation 来在外部 DNS 上直接创建 DNS 记录！有了这个插件我们可不用再慢慢打开公共 DNS 管理页面，再小心翼翼地记下 IP 地址去添加 A 记录了。

# 更高级的部署方式（一）

一路说道这里， K8s 中最基础的资源大部分都已经介绍了。但是，这么多资源之间又需要相互配合，只部署一种资源基本没什么生产能力。

比如只部署 Deployment 的话，我们确实是能在一组多副本的 Pod 里跑起可执行程序，但这组 Pod 却几乎没办法接受集群里其他 Pod 的流量（只能通过制定 Pod 的 IP 来访问，但 Pod 的 IP 是会变的）。因此一般来说一个 Deployment 都会搭配一个 Service 来使用。这还是最简单的一种搭配了。

假若我们现在要在自己的 K8s 里安装一个别人提供的应用。当然由于 K8s 是基于容器的，只要别人提供了他应用的 yaml 清单，我们只用把清单用 `kubectl apply -f` 提交给 K8s ，然后让 K8s 把清单中的镜像拉下来就能跑了。可如果我们需要根据环境来改一些参数呢？

如果别人提供的 yaml 文件比较简单还好说，改改对应的字段就好了。如果别人的应用比较复杂，那改 yaml 文件可就是一个大难题了。比如 AWS 的 Load Balancer Controller ，它的 yaml 清单文件可是多达 939 行！

[[aws-elb-controller-lines.png]]

在这种复杂的场景下，我们就需要一些更高级的部署方式了。

### Helm

首先来介绍的是 Helm 。 Helm 是一个包管理工具，可以类比一下 CentOS 中的 yum 工具。它可以把一组 K8s 资源发布成一个 Chart ，然后我们可以用 Helm 来安装这个 Chart ，并且可以通过参数设值来改变 Chart 中的部分资源。利用 Helm 安装 Chart 后还可以管理 Chart 的升级、回滚、卸载。

使用别人提供的 Helm Chart 前，需要先 add 一下 Chart 的仓库，然后再安装仓库里提供的 Chart 。比如我们要安装 bitnami 提供的 Kafka Chart 时：

```bash
# 添加 https://charts.bitnami.com/bitnami 这个仓库，命名为 bitnami
helm repo add bitnami https://charts.bitnami.com/bitnami

# 在 kafka 名称空间里安装 bitnami 仓库里的 kafka Chart ，并通过参数设置为 3 个副本，并同时安装一个 3 副本的 Zookeeper
helm install kafka -n kafka \
  --set replicaCount=3 \
  --set zookeeper.enabled=true \
  --set zookeeper.replicaCount=3 \
  bitnami/kafka
```

命令执行后， helm 就会根据参数与 Chart 的内容，在 K8s 里安装 StatefulSet 、 Service 、 ConfigMap 等一切所需要的资源。

```sh
$ k -n kafka get all,cm
NAME                    READY   STATUS    RESTARTS      AGE
pod/kafka-0             1/1     Running   1             46d
pod/kafka-1             1/1     Running   3             46d
pod/kafka-2             1/1     Running   3             46d
pod/kafka-zookeeper-0   1/1     Running   0             46d
pod/kafka-zookeeper-1   1/1     Running   0             46d
pod/kafka-zookeeper-2   1/1     Running   0             46d

NAME                               TYPE           CLUSTER-IP       EXTERNAL-IP      PORT(S)                      AGE
service/kafka                      ClusterIP      172.20.1.196     <none>           9092/TCP                     164d
service/kafka-headless             ClusterIP      None             <none>           9092/TCP,9093/TCP            164d
service/kafka-zookeeper            ClusterIP      172.20.227.236   <none>           2181/TCP,2888/TCP,3888/TCP   164d
service/kafka-zookeeper-headless   ClusterIP      None             <none>           2181/TCP,2888/TCP,3888/TCP   164d

NAME                               READY   AGE
statefulset.apps/kafka             3/3     164d
statefulset.apps/kafka-zookeeper   3/3     164d

NAME                                DATA   AGE
configmap/kafka-scripts             2      164d
configmap/kafka-zookeeper-scripts   2      164d
configmap/kube-root-ca.crt          1      165d
```

甚至， Helm 可以通过模板生成的 Pod 环境变量，来预先设置好 Kafka 的配置，让他找得到 Zookeeper 服务：

```yaml
apiVersion: v1
kind: Pod
# 略去无关信息
spec:
  containers:
  - name: kafka
    command:
    - /scripts/setup.sh
    env:
    - name: KAFKA_CFG_ZOOKEEPER_CONNECT
      value: kafka-zookeeper
    # ...
```

通过设置 `KAFKA_CFG_ZOOKEEPER_CONNECT` 这个环境变量，指定了 Kafka Broker 可以通过访问 `kafka-zookeeper` 来找到 zookeeper 服务。（还记得 zookeeper 的 Service 名字是 `kafka-zookeeper` 吗？ zookeeper 与 kafka 部署在同一个名称空间里，因此可以直接通过 Service 名访问。）

如果我们打开这个 helm chart 对应的[代码仓库](https://github.com/bitnami/charts/tree/master/bitnami/kafka)，会发现原来有一组 go template 文件，以及一个 `values.yaml` 文件和 `Chart.yaml` 文件：

```sh
.
├── Chart.lock
├── Chart.yaml
├── README.md
├── templates
│   ├── NOTES.txt # 这里定义的是 helm 工具的命令行信息
│   ├── _helpers.tpl # 这里面是一些定义好的 go template 代码块可以供其他模板使用
│   ├── configmap.yaml
│   ├── statefulset.yaml
│   ├── svc-headless.yaml
│   ├── svc.yaml
│   └── # 以下省略若干模板文件
└── values.yaml
```

- `Chart.yaml` 中定义了这个 Chart 的基本信息，包括名称、版本、描述、依赖等。
- `values.yaml` 中定义了这个 Chart 的默认参数，包括各种资源的默认配置、副本数量、镜像版本等。其中的值都可以通过 `helm install` 命令的 `--set` 参数来覆盖。
- `templates/` 文件夹下的都是 go template 的模板文件。

`helm install` 就是通过用 `values.yaml` 中预定义的参数，渲染 `templates/` 文件夹下的 go template 文件，生成最终的 yaml 文件，然后再通过 kubectl apply -f 的方式，将 yaml 文件里的资源部署到 K8s 里。然后通过忘资源里注入一些特殊 annotation 的方式来记住自己部署了那些资源，进而提供 `update` 、 `uninstall` 等功能。

关于更多 Helm 的内容，可以参考[官方文档](https://helm.sh/docs/)。

### Kustomize

另一个部署工具是 Kustomize 。之前提到 Config Map 时的例子中，将配置文件的内容直接写进了 yaml 清单的一个字段里：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: game-demo
data:
  # 一个 Key 可以对应一个值
  player_initial_lives: "3"
  ui_properties_file_name: "user-interface.properties"

  # 一个 Key 也可以对应一个文件的内容
  game.properties: |
    enemy.types=aliens,monsters
    player.maximum-lives=5    
  user-interface.properties: |
    color.good=purple
    color.bad=yellow
    allow.textmode=true    
```

其实这样很不好，先不说这样写没办法在 IDE 里用配置文件自己的语法检查，每行还需要一定的缩进，如果配置文件有好几百行，你甚至会忘了这一行到底是哪个配置文件！此时我们就会自然而然的想把每个配置文件以单独文件的形式保存。

Kustomize 就是这样一个工具，它可以帮助我们把每个配置文件以单独文件的形式保存，然后再通过一个 `kustomization.yaml` 文件，将这些配置文件组合起来，生成最终的 yaml 文件。

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  # 其他资源也可以单独使用一个文件定义
  - deployment.yaml

# 用 configMapGenerator 从文件中生成 ConfigMap
configMapGenerator:
  - name: game-demo
    literals:
      - "ui_properties_file_name=user-interface.properties"
      - "player_initial_lives=3"
    # 从文件中读取内容
    files:
      - game.properties
      - user-interface.properties
# 有多个 configMap 时，可以通过统一的 generatorOptions 来设置一些通用的选项
generatorOptions:
  disableNameSuffixHash: true
```

然后两个配置文件的内容可以单独用文件定义，此时可以结合 IDE 的语法检查，以及代码补全功能，来编写配置文件。

```properties
# user-interface.properties
color.good=purple
color.bad=yellow
allow.textmode=true    
```

然后将 `kustomization.yaml` 和其他所需的文件都放在同一个目录下：

```bash
.
├── kustomization.yaml
├── deployment.yaml
├── game.properties
└── user-interface.properties
```

然后就可以通过 `kubectl apply -k ./` 来将整个 kustomize 文件夹转换为 yaml 清单直接部署到 K8s 中。
（没错，现在 Kustomize 已经成为 kubectl 中的内置功能！可以不用先 `kustomize build` 生成 yaml 文件再 `kubectl apply` 两步走了！）

值得提醒的是，虽然 `kustomization.yaml` 有 `apiVersion` 和 `kind` 字段，长得很像一个资源清单，但其实 K8s 的 API server 并不认识他。 Kustomize 的工作原理其实是先根据 `kustomization.yaml` 生成 K8s 认识的 yaml 资源清单，然后再通过 `kubectl apply` 来部署。

除了可以直接将 ConfigMap 与 Secret 中的文件字段内容用单独的文件定义外， Kustomize 还有其他比如为部署的资源添加统一的名称前缀、添加统一字段等功能。这些大家可以阅读 Kustomize 的[官方文档](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/kustomization/)来了解。

### 各种工具的优缺点

我们目前已经知道有三种在 K8s 中部署资源的方式： `kubectl apply`、Helm 和 Kustomize 。

其中 `kubectl apply` 的优缺点很明确，优点是最简单直接，缺点是会导致要么 yaml 清单过长，要么需要分多文件多次部署，使集群中产生中间状态。

而 Helm 与 Kustomize 我们上面也分析过，其实都是基于 `kubectl apply` 的。 Helm 是通过 go template 先生成 yaml 文件再 `kubectl apply` ，而 Kustomize 是通过 `kustomization.yaml` 中的定义用自己的一套逻辑生成 yaml 文件，然后再 `kubectl apply` 。

Helm 的优点是 Helm Chart 安装时可以直接使用别人 Helm 仓库中已经上传好的 Chart ，只需要设置参数就可以使用。这也是 Kustomize 的缺点：如果想要使用别人提供的 Kustomization 而只修改其中的一些配置，必须要先把放 `kustomization.yaml` 的整个文件夹下载下来才能做修改。

而 Helm 的缺点也是明显的， Helm 依赖于往资源里注入特殊的 annotation 来管理 Chart 生成的资源，这可能会很难与集群中现有的一些系统（比如 Service Mesh 或是 GitOps 系统等）放一起管理。而 Kustomize 生成的 yaml 清单就是很干净的 K8s 资源，原先的 K8s 资源该是什么表现就是什么表现，与现有的系统兼容一般会比较好。

而另外，由于 Helm 与 Kustomize 都是基于 `kubectl apply` 的，因此他们有共同的缺点，就是不能做 `kubectl apply` 不能做的事情。

什么叫 `kubectl apply` 不能做的事情呢？比如说我们要在 K8s 中部署 Redis 集群。聪明的你可能就想到要用 Stateful Set 、 PVC 、 Headless Service 来一套组合拳。这确实可以部署一个多节点、有状态的 Redis Cluster 。可是如果我们要往 Redis Cluster 里加一个节点呢？

你当然可以把 Stateful Set 中的 `Replicas` 字段加个 1 然后用 `kubectl apply` 部署，可是这实际上只能增加一个一个 Redis 实例 —— 然后什么都没发生。其他节点不认识这个新的节点，访问这个新节点也不能拿到正确的数据。要知道往 Redis Cluster 里加节点，是要先让集群发现这个新节点，然后还要迁移 slot 的！ `kubectl apply` 可不会做这些事。

> Well, 其实这些也是可以通过增加 initContainer 、修改镜像增加启动脚本等方式，实现用 `kubectl apply` 部署的。可是，这会让整个 Pod 资源变得很难理解，也不好维护。而且，如果不是因为做不到，谁会想去修改别人的镜像呢？

我们接下来会介绍 K8s 的核心架构，来理解我们之前讲的这些资源到底是怎么工作的。最后会引出一组新的概念： Operator 与自定义资源（ Custom Resource Definition ，简称 CRD ）。通过 Operator 与 CRD ，我们可以做到 `kubectl apply` 所不能做到的事，包括 Redis Cluster 的扩容。

> DIO: `kubectl apply` 的能力是有限的……
> 越是部署复杂的应用，就越会发现 `kubectl apply` 的能力是有极限的……除非超越 `kubectl apply` 。
> 
> JOJO: 你到底想说什么？
> 
> DIO: 我不用 `kubectl apply` 了！ JOJO ！
> （其实还是要用的）

<a rel="license" href="http://creativecommons.org/licenses/by-nc/4.0/"><img alt="Creative Commons License" style="border-width:0" src="https://i.creativecommons.org/l/by-nc/4.0/88x31.png" /></a><br />This work is licensed under a <a rel="license" href="http://creativecommons.org/licenses/by-nc/4.0/">Creative Commons Attribution-NonCommercial 4.0 International License</a>.
