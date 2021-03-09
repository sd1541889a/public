# kubectl命令工具
```
1、kubectl概述
   kubectl是 KUbernetes 集群的命令行工具，通过kubectl能够对集群本身进行管理，并能狗仔集群上进行容器化应用的安装部署
2、kubectl命令的语法
   kubectl [command] [TYPE] [NAME] [flags]
   1)comand: 指定对资源执行的操作，例如create 、get 、describe 和delete
   2)TYPE: 指定资源类型，资源类型是大小写敏感的，开发者能够以单数、复数和缩略的形式。例如：
           kubectl get pod pod1
   3)NAME: 指定资源的名称，名称也是大小写敏感的。如果省略名称，则会显示所有的资源，例如：
           kubectl get pods
   4)flags: 指定可选的参数。例如，可以用-s 或者 -server 参数指定kubernetes API server 的地址和端口。
 

```


# yaml文件
```
1、yaml文件概述
   k8s 集群中对资源管理和资源对象编排部署都可以通过生命样式(YAML)文件来解决，也就是可以把需求对资源对象操作编辑到yaml格式文件中，我们把这种文件叫做资源清单文件，通过kubectl命令直接使用资源清单文件就可以实现对大量的资源对象进行编排部署了。

2、yaml文件书写格式
 2.1yaml介绍
  yaml:仍是一种标记语言。为了强调这种语言以数据做为中心，而不是以标记语言为重点。
  yaml是一个可读性高，用来表达数据序列的格式。
 2.2yaml基本语法
  *使用空格做为缩进
  *缩进的空格数目不重要，只要相同层级的元素左侧对齐即可
  *低版本缩进时不允许使用Tab键,只允许使用空格
  *字符后缩进一个空格，比如冒号，逗号等后面
  *使用---表示新的yaml文件开始
  *使用#表示注释
 
 3、yaml文件组成部分
   apiVersion: apps/v1                                   #API版本
   kind: Deployment                                      #资源类型
   metadata:                                             #资源元数据
    name: nginx-deployment                               
    namespace: default
   spec:                                                 #资源规格
    replicas: 3                                          #副本数量
    selector:                                            #标签选择器
     matchLabels:
      app: nginx
    template:                                            #Pod模版
     metadata:                                           #Pod元数据
      labels:
       app: nginx
     spec:                                               #Pod规格
      containers:                                        #容器配置
      - name: nginx
        image: nginx:1.15
        ports:
        - containerPort: 80

如何快速编写yaml文件
第一种 使用kubectl create命令生成yaml文件
  kubectl create deployment web --image=nginx -o yaml --dry-run   尝试执行输入ymal内容
  kubectl create deployment web --image=nginx -o yaml --dry-run > nginx.yaml 输出到nginx.yaml中


第二种 使用kubectl get命令导出yaml文件
  kubectl get deploy nginx（名称） -o=yaml --export > getnginx.yaml（文件名称） 导出已运行的pod中提取yaml文件
```

# Pod
```
1、Pod基本概念
  1.1 最小部署的单元
  1.2 包含多个容器（一组容器的集合）
  1.3 一个pod中容器共享网络命名空间
  1.4 pod 是短暂的
  
2、Pod存在意义
  2.1 创建容器使用docker，一个docker对应是一个容器，一个容器有进程，一个容器运行一个应用程序。
  2.2 Pod是多进程设计，运行多个应用程序。
  2.3 两个应用之间进行交互
  2.4 网络之间调用
  2.5 两个应用需要频繁调用 如后端和数据库之间调用

3、Pod实现机制
  容器本身是相互隔离的 通过namespace  和 group
  
  3.1 共享网络
      前提条件
        容器在同一个namespace 
        pod中生成Pause Info容器，在把你的容器加入到info容器中。让你所有的容器在同一个名称中，实现网络共享。
        
  3.2 共享存储
        Pod实现机制 共享存        
           Pod持久化数据
               日志数据
               业务数据
      引入数据库概念Volumn,使用数据卷进行持久化存储。
4、Pod镜像拉去策略
   实例：
        apiVersion: v1
        kind: Pod
        metadata: 
          name: mypod
        spec:
          containers:
            - name: nginx
              image: nginx:1.14
              imagePullPolicy: Always
        # IfNotPresent: 默认值，镜像在宿主机上不存在时才拉取
        # Always: 每次创建Pod都会重新拉取一次镜像
        # Never: Pod 永远不会主动拉取这个镜像
5、Pod资源限制
   实例：
        apiVersion: v1
        kind: Pod
        metadata:
          name: frontend
        spec:
          containers:
          - name: db
            image: mysql
            env:
            - name: MYSQL_ROOT_PASSWORD
              value: "password"
            
            resources:
              requests:                          #调度资源
                memory: "64Mi"
                cpu: "250m"
              limits:
                memory: "128Mi"                  #最大资源
                cpu: "500m"
 6、Pod重启机制
    实例：
         apiVersion: v1
         kind: Pod
         metadata:
           name: dns-test
         spec:
           containers:
           - name: busybox
             image: busybox:1.28.4
             args:
             - /bin/sh
             - -c
             - sleep 36000
           restartPolicy: Never
            
          # Always: 单签容器终止退出后，总是重启容器，默认策略。
          # OnFailure: 当容器异常退出（退出状态码非0）时， 才重启容器。
          # Never: 当容器终止退出，从不重启容器。
          
 7、Pod健康检查
    7.1容器检查
       虽然状态是runnning 但是存在一种进程在运行但是本身不能提供服务的状态。例如java堆内存溢出
    7.2应用层面检查
       7.2.1 livenessProbe (存活检查)
             如果检查失败，将杀死容器，根据Pod的restartPolicy来操作。
       7.2.2 readinessProbe (就绪检查)
             如果检查失败，Kubernetes会把Pod从service endpoints中剔除。
    实例：
         apiVersion： v1
         kind: Pod
         metadata:
           labels:
             test: liveness
           name: liveness-exec
         spec:
           containers:
           - name: liveness
             image: busybox
             args:
             - /bin/sh
             - -c                                                                 # Probe支持一下三种检查方法:
             - touch /tmp/healthy; sleep 30; rm -rf /tmp/healthy                  ## httpGet
             livenessProbe:                                                       ### 发送HTTP请求，返回200-400范围状态码为成功。
               exec:
                 command:                                                         ## exec
                 - cat                                                            ### 执行shell命令返回状态码0为成功。
                 - /tmp/healthy
               initialDelaySeconds: 5                                             ## tcpSocket
               periodSeconds: 5                                                   ### 发起TCP Socket建立成功。
  8、Pod调度策略
  
     8.1 Pod资源限制对Pod调用产生影响
         根据request找到足够node节点进行调度
         
     8.2 节点选择器标签影响Pod调度。需要创建标签
          kubectl label 节点名称 分组名称=名
          
     实例
         spec:
          nodeSelector:                                                           #节点选择器
            env_role: dev                                                         #组名称 dev
            
     8.3 节点亲和性影响Pod调度
          节点亲和性nodeAffinity和之前nodeSelector 基本一样，根据节点上标签约束来决定Pod调度到哪些节点上。
          8.3.1 硬亲和性
                约束条件必须满足
          8.3.2 软亲和性
                尝试满足无法保证绝对满足
          8.3.3 常用操作符（In、NotIn、Exists、Gt、LT、DoesNotExists）
          
     实例
         apiVersion: v1
         kind: Pod
         metadata:
           name: with-node-affinity
         spec:
           affinity:
             nodeAffinity:                                                        #硬亲和性
               requiredDuringSchedulingIgnoredDuringExecuion:
                 nodeSelectorTerms:
                 - matchExpressions:
                   - key: env_role                                                #标签名称
                     operator: In                                                 #表示在什么里 
                     values:                                                      #约束条件必须满足
                     - dev
                     - test
               preferredDuringSchedulingIgnoredDuringExecution:                    #软亲和性
               - weight: 1
                 preference:
                   matchExpressions:
                   - key: group
                     operator: In
                     values:
                     - otherprod
         containers:
         - name: webdemo
           image: nginx
          8.3.4 反亲和性
                使用 NotIn 和 DoesNotExists
          8.3.5 污点，节点不做普通分配调度，针对特定场景进行分配。是节点属性。
                使用场景：专用节点、配置特点硬件节点、基于Taint驱逐
                kubectl describe node 节点名称 |grep Taint 查看节点中污点。
                污点中的值有三种: NoSchedule                                     #一定不被调度
                                  PreferNoSchdule                                #尽量不被调度
                                  NoExecute                                      #不会调度，并且还会驱逐Node中已有的Pod
                kubectl taint node 节点名称 key=value:污点三个值,为节点添加污点。
                
                kubectl taint node 节点名称 env_role:NoSchedule- ,删除节点中的污点。
          8.3.6 污点容忍
                实例
                    spec:
                      tolerations:
                      - key: "key"
                        operator: "Equal"
                        value: "value"
                        effect: "NoSchedule"
                        
                        
                      containers:
                      - name: webdemo
                        image: nginx
```

# Controller
```
1、什么是controller
   管理和运行容器的一个对象，确保容器副本数，是实际存在的。
2、Pod和controller关系
   Pod是通过controller控制器实现应用的运维,比如：伸缩、滚动升级等等。Pod是同label标签与Controller之间建立连接。
   selector：中添加Pod标签与Pod标签对应后，是Pod和Controller之间才关联。
   例如：
       spec:
        replicas: 1
        selector:                                                     #controller
          matchLabels:                                                #controller中标签
            app: nmg-test                                             #标签名称
        strategy: {}
        template:
          metadata:                                                   #Pod
            creationTimestamp: null
            labels:                                                   #Pod中标签
              app: nmg-test                                           #标签名称    controller和Pod中标签上下是对应的
          spec:
            containers:
            - image: nginx
              name: nginx
              resources: {}

3、Deployment控制器应用场景
      Deployment一般部署无状态应用，管理Pod的部署，滚动，升级等功能。一般应用在web和微服务中。
4、yaml文件说明
5、Deployment控制器部署应用
      kubectl apply -f yaml文件名称.yaml 创建相关配置
      kubectl create deployment 名称 --image=进项 --dry-run(常用运行) > 导出文件名称.yaml，用来创建yaml文件
      kubectl apply -f yaml文件名称.yaml启动文件
      kubectl expose deployment web --prot=Pod端口号 --type=NodePort节点端口类型 --name=名称 -o ymal > yaml名称.yaml  导出端口映射配置
      kubectl apply -f yaml文件名称.yaml设置端口映射
6、升级回滚
   6.1升级
      kubectl set image deploy 项目名称 nginx=nginx:1.15 ,升级Pod版本
   6.2回滚
      kubectl rollout status deployment 名称                         #查询操作结果
      kubectl rollout history deployment 名称                        #查询目前Pod版本
      kubectl rollout undo deployment 名称                           #回归到上一个版本
      kubectl rollout undo deployment 名称 --to-revision=版本        #指定版本回滚
7、弹性伸缩
      kubectl scale deployment 名称 --replicas=副本数  ，用来增加和删减Pod数量
      
      
 8、有状态pod部署
   8.1、有状态和无状态的特点
       8.1.1、无状态下，通常认为Pod都是一样的，切没有执行顺序要求，不用考虑在哪个node中运行，可以进行随意伸缩和扩展。
       8.1.2、有状态下，通常需要考虑无状态下的内容，认为每个pod都是独立的，需要保持pod启动顺序和唯一性，通过唯一的网络标识符区分和之久存储做到。有状态pod是有序的，例如mysql主从。
 9、守护进程
   9.1、部署守护进程
        目标：在每个node上运行一个pod,新加入的node也同样运行一个pod里面
        实例
            apiVersion: apps/v1
            kind: DaemonSet
            metadata:
              name: ds-test
              labels:
                app: filebeat
            spec:
              selector:
                matchLabels:
                  app: filebeat
              template:
                metadata:
                  labels:
                    app: filebeat
                spec:
                  containers:
                  - name: logs
                    image: nginx
                    ports:
                    - containerPort: 80
                    volumeMounts:
                    - name: varlog
                      mauntPath: /tmp/log
                  volumes:
                  - name: varlog
                    hostPath:
                      path: /var/log
  10、job（一次性任务）
      实例
          apiVersion: batch/v1
          kind: Job
          metadata:
            name: pi
          spec:
            template:
              sepc:
                containers:
                - name: pi
                  image: perl
                  command: ["perl","-Mbignum=bpi","-wle","print bpi(2000)"]
                restartPolicy: Never
            backorffLimit: 4
            
  11、cronjob(定时任务)
      实例
          apiVersion: batch/v1beta1
          kind: CronJob
          metadata:
            name: hello
          spec:
            schedule: "*/1 * * * * "
            jobTepmlate:
              spec:
                template:
                  spec:
                    containers:
                    -  name: hello
                       image: busybox
                       args:
                       - /bin/sh
                       - -c
                       - date;echo Hello from the Kubernetes cluster
                    restartPolicy: OnFailure

```
# Service
```
1、Service概念
   1.1、定义一组Pod的访问规则，防止Pod失联（服务发现）
   1.2、定义一组Pod的访问策略（负载均衡）
   1.3、Service根据label标签与Pod建立关系
2、Service常用类型
   2.1、ClusterIP:集群内部使用
        node内网部署应用，外网一般不能访问到的
   2.2、NodePort:对外访问应用使用   
   2.3、LoadBalancer:对外访问应用使用，公有云。负载均衡控制器。
3、无头Service
   ClusterIP为none的Service，为无头Service
   
   实例：
      apiVersion: apps/v1
      kind: Service
      metadata:
        name: nginx
        labels:
          app: nginx
      spec:
        ports:
        - port: 80
          name: web
        clusterIP: None
        selector:
          app: nginx
 deployment 和statefueset区别，是有身份的（唯一标识的）
   statefueset的规则 根据主机名+按照一定规则生成域名。每个pod有唯一主机名，唯一域名。格式如下：主机名称.service名称.名称空间.svc.cluster.local。
   例如：nginx-statefulset-0.nginx.default.svc.cluster.local
```

# Secret
```
作用：加密数据存在etcd里面，让Pod容器以挂载Volume方式进行访问。
场景：凭证
使用base64加密
1.1、建立一个secret。实例：
     apiVersion: v1
     kind: Secret
     metadata:
       name: mysecret
     type: Opaque
     data:
        username: YWRtaW4=
        password: MTIzNDU2
创建secret,kubectl create -f secret.yaml
kubectl get secret ，查数据创建是否成功。
1.2、绑定secret到pod中,实例：
     apiVersion: v1
     kind: Pod
     metadata:
       name: mypod
     spec:
       containers:
       - name: nginx
         image: nginx
         env:
           - name: SECRET_USERNAME                #挂载secret中username,key名称
             valueFrom:
               secretKeyRef:
                 name: mysecret
                 key: username
           - name: SECRET_PASSWORD                #挂载secret中password，key名称
             valueFrom:
               secretKeyRef:
                 name: mysecret
                 key: password
 1.3、已数据卷形式,将secret挂载到Pod中。实例：
      apiVersion: v1
      kind: Pod
      metadata:
        name:mypod
      spec:
        containers:
        - name: nginx
          image: nginx
          volumeMounts:
          - name: foo
            mountPath: "/etc/foo"
            readOnly: true
        volumes:
        - name: foo
          secret:
            secretName: mysecret
```
# ConfigMap
```
作用： 存储不加密数据到etcd,让Pod以变量或者Volume挂载到容器中。
场景： 配置文件
实例：
     创建配置文件，创建configMap。
     kubectl create configmap redis-config --from-file=redis.properties ，创建configmap，from-file添加文件目录。
     建立cm.yaml文件
     apiVersion: v1
     kind: Pod
     metadata:
       name: mypod
     spec:
       containers:
       - name: busybox
         image: busybox
         command: ["/bin/sh","-c","cat /etc/config/redis.properties"]
       volumeMounts:
       - name: config-volume
         mountPath: /et/config
       volumes:
         - name: config-volume
           configMap:
             name: redis-config
       restartPolicy: Never

```
# 集群安全机制
```
1、k8s安全机制流程
   1.1、概念
        访问k8s集群时候，需要经过三个步骤完成具体操作。
        
        第一步 认证
        传输安全： 对外不暴露8080端口，只能内部访问，对外使用端口6443。
        认证，客户端身份认证常用方式：
        https 证书认证，基于ca证书。http token认证，通过token识别用户。http基本认证，用户名+密码认证。
        
        第二步 鉴权（授权）
        
        基于RBAC进行鉴权操作，基于角色访问控制。
        角色：Role、ClusterRole。
              Role： 授权特定命名空间访问权限。
              ClusterRole： 所有命名空间访问权限。
              
        主体：user、group、serviceaccount。
              user: 用户    group：用户组   serviceAccount: 服务账号（一般用于pod访问）
              
        绑定：将角色和主体绑定。让某个主体属于某个角色。
              rolebinding: 角色绑定到主体
              ClusterRoleBinding: 集群角色绑定到主体。
              
        规划：确定角色控制权限，如资源（pod、node），对资源控制（get、create）。规划与角色绑定。
        
        第三步 准入控制
        就是准入控制器的列表，如果列表有请求内容，通过，没有拒绝。
        
   1.2、进行访问时候，过程中都需要经过apiserver，apiserver做统一协调，比如门卫。
        访问过程中需要证书、token、或者用户名+密码。如果访问pod，需要serviceAccount
        
   1.3、部署实例
           创建命名空间
           kubectl create ns roledemo
           命名空间下建立一个pod
           kubectl run nginx --image=nginx -n 命名空间
           创建角色
           vim rbac-role.yaml
           ------
           kind: Role
           apiVersion: rbac.authorization.k8s.io/v1
           metadata:
             namespace: ctnrs
             name: pod-reader
           rules:
             apiGroups: [""]
             resources: ["pods"]                     #对象
             verbs: ["get","watch","list"]           #权限
           ------
           kubectl apply -f rbac-role.yaml
           
           绑定角色
           vim rbac-rolebinding.yaml
           ------
           kind: RoleBinding
           apiVersion: rbac.authorization.k8s.io/v1
           metadata:
             name: read-pods
             namespace: roletest
           subjects:
             kind: User
             name: BDragon
             apiGroup: rbac.authorization.k8s.io
           roleRef:
              kind: Role
              name: pod-reader
              apiGroup: rbac.authorization.k8s.io
           ------
           kubectl apply -f rbac-rolebinding.yaml
           
           创建证书，识别角色身份
           -----
           vim BDragon-csr.json
           {
              "CN": "BDragon",
              "hosts": [],
              "key":{
                "algo": "rsa",
                "size": 2048
              },
              "names": [
                {
                   "C": "CN",
                   "L": "ShenYang",
                   "ST": "ShenYang"
                }
              ]
           }
           -----
           
           添加用户证书
           openssl 生成用户证书 【kubeadm】
           openssl genrsa -out BDragon.key 2048 生成秘钥
           openssl req -new -key BDragon.key -out BDragon.csr -subj "/CN=BDragon" 生成csr证书
           openssl x509 -req -in BDragon.csr -CA /etc/kubernetes/pki/ca.crt -CAkey /etc/kubernetes/pki/ca.key -CAcreateserial -out BDragon.crt -days 3650  生成BDragon.key证书时长10年。
           openssl x509 -in BDragon.crt -text -noout  测试
           kubectl config set-credentials BDragon --client-certificate=./BDragon.crt --client-key=./BDragon.key --embed-certs=true 添加用户证书
           kubectl config set-context BDragon@kubernetes --cluster=kubernetes --user=BDragon  向k8s中添加用户
           kubectl config use-context BDragon@kubernetes 切换用户 。  k8s超级管理员账号 kubernetes-admin@kubernetes
           测试
           kubectl get pods -n roledemo  查询命名空间的pods
           kubectl get svc -n roledemo  查看命名空间的svc，没有权限报错。

```

# Ingress
```
1、回顾
   1.1、把端口号对外暴露，通过IP+端口号进行访问。
        使用Service里面NodePort实现。三种类型 NodePort、ClusterPort等
   1.2、NodePort缺陷
        在每个节点上都会起到端口，在访问时候通过任何节点，通过节点ip+暴露端口号实现访问。
        意味着每个端口只能使用一次，一个端口对应一个应用。
        实际访问中都是用域名，根据不同域名跳转到不同端口服务中。
   1.3、Ingress和Pod关系
        pod和ingress通过service关联的。将ingress作为统一入口，由service关联一组pod。
        实际访问中都是用域名，根据不同域名跳转到不同端口服务中。
   1.4、Ingress工作流程
        域名→ingress入口→对应service→对应pod
2、部署实例
   使用ingress
   第一步 部署ingress Controller 
   第二步 创建ingress规则
          实例使用官方nginx控制器部署。
          
```
# helm
```
1、helm引入
   1.1、之前方式部署应用基本过程，编写yaml文件→建立deployment→Service→Ingress，部署单一应用，少数服务的应用，比较适合。
     缺陷：比如部署微服务项目，可能有几十个服务，每个服务都有一套yaml文件，需要维护大量yaml文件，版本管理特别不方便。
   1.2、使用helm可以解决哪些问题？
         a)使用helm可以把这些yaml作为一个整理管理。
         b)实现yaml文件的高效复用。
         c)应用级别的版本管理。
         
 2、helm介绍 
    helm是一个Kubernetes的包管理工具，就像linux下的包管理器，如yum/apt等，可以很方便的将之前打包好的yaml文件部署到kubernetes上。
    2.1、重要概念
         a)helm：一个命令行客户端工具，主要用于Kubernetes引用chart的创建、打包、发布和管理。
         b)Chart：应用描述，一系列用于描述K8s资源相关文件的集合。
         c)Release：将在k8s中创建出真实运行的资源对象（版本管理）。
    2.2、helm v3 变化
         19年11月13日，helm发布了v3版本，和之前版本相比有变化
         a)Tiller在V3版本中已被剔除（删除）。在早期版本中通过Tiller连接k8s集群，在新版本中已经修改为其他方式。
         b)release在V3版本中可以在不同命名空间重用，早期版本并不支持。
         c)chart在V3版本中可以将chart文件推送到docker仓库中。
    2.3、架构变化
         操作流程的变化。
         a)早期版本
           部署helm chart → tiller → kube-apiserver → 操作k8s集群
         b)V3版本
           部署helm chart → kube-config → kube-apiserver → 操作k8s集群
  
 3、安装helm
    3.1 helm安装
        安装教程：https://helm.sh/zh/docs/intro/install/
        下载地址：https://github.com/helm/helm/releases/tag/v3.3.4
        截止笔记时间2021.01.18版本说明如下：
        Helm 版本	支持的 Kubernetes 版本
         3.4.x	1.19.x - 1.16.x
         3.3.x	1.18.x - 1.15.x
         3.2.x	1.18.x - 1.15.x
         3.1.x	1.17.x - 1.14.x
    3.2 添加仓库
        helm repo add 仓库名称  仓库地址
        阿里云： helm repo add aliyun https://kubernetes.oss-cn-hangzhou.aliyuncs.com/charts
        更新源
        helm repo update
        查看配置的存储库
        helm repo list
        helm search repo stable 
        删除存储库：
        helm repo remove aliyun
  4、使用helm快速部署应用
    第一步 使用命令搜索应用
       helm search repo 名称 （weave）
    第二步 根据搜索内容选择安装
       helm install 名称 搜索应用名称 
       查看安装状态
       helm list
       helm status安装之后名称
       
```
# PV和PVC
```
PV描述：
  PersistentVolme,PV(持久卷)是集群中的一块存储，可以由管理员事先供应，或者使用Storage Class(存储类)来动态供应。持久卷是集中
pv模版：
apiversion: v1
kind: PersistentVolume    #PV
metadata:
  name: local-pv          #名称
spec:
   storageClassName: standard     #用于和PVC绑定
   capacity:
     storage: 250Mi               #空间大小250Mi
   accessModes:
   - ReadWriteOnce                #访问模式单节点访问
   hostPath:
     path: "/tmp/data02"          #主机路径
     type: DirectoryOrCreate      
     
pvc模版：
apiversion: v1
kind: PersistenVolumeClaim         #PVC
metadata:
  name: mysql-pvc                  #名称
spec:
  storageClassName: standard       #用于和PV绑定
  accessModes:
    - ReadWriteOnce                #访问模式：单一节点访问
  rescources:
    requests:
      storage: 250Mi               #空间大小
```
