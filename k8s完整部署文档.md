# 环境准备
## 1.关闭防火墙
```
systemctl stop firewalld
systemctl disable firewalld
```

## 2.关闭swap交换分区
```
vim /etc/fstab，将swap配置注销
```

## 3.关闭selinux
```
vim /etc/selinux/config，修改enforcing为disabled
```

## 4.修改IPv4流量传递到iptables
```
cat > /etc/sysctl.d/k8s.conf << EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sysctl --system  #生效配置
```

## 5.同步时间
```
yum -y install ntpdate 
ntpdate time.windows.com
```
## 6.修改服务器hosts和hostname
```
cat >> /etc/hosts << EOF
服务器IP master
服务器IP node1
服务器IP node2
EOF

hostnamectl set-hostname 服务器名
```

# Docker环境安装
```
yum -y install yum-utils #安装yum查询版本工具
wget https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo -O /etc/yum.repos.d/docker-ce.repo #下载阿里云yum源
查询源下docker-ce版本
yum list docker --showduplicates |sort -r
安装docker版本
yum -y install docker-ce-19.03.9-3.el7 
systemctl start docker
修改docker镜像源地址到阿里云上
cat > /etc/docker/daemoin.json << EOF
{
  "registry-mirrors":["https://fdapudwh.mirror.aliyuncs.com"]
}
EOF
systemctl daemon-reload
systemctl restart docker
systemctl enable docker
```
# Kubernetes环境安装
## K8s使用阿里云源
```
cat > /etc/yum.repos.d/kubernetes.repo <<EOF
[kubernetes]
name=Kubernetes
baseurl=http://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
repo_gpgcheck=0
  gpgkey=http://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg
         http://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF

```
## Kubernetes二进制安装
### 自签证书
```
#创建文件夹并进入文件夹
mkdir -p /opt/k8s/work && cd /opt/k8s/work
#下载cfssl组件
curl -L https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64 -o /usr/local/bin/cfssljson
curl -L https://pkg.cfssl.org/R1.2/cfssl_linux-amd64 -o /usr/local/bin/cfssl
curl -L https://pkg.cfssl.org/R1.2/cfssl-certinfo_linux-amd64 -o /usr/local/bin/cfssl-certinfo
授权cfssl组件执行权限
chmod +x /usr/local/bin/cfssl /usr/local/bin/cfssljson /usr/local/bin/cfssl-certinfo
#创建根证书
添加json配置文件
cat > ca-config.json <<EOF
{
  "signing": {
     "default": {
       "expiry": "87600h"
       },
     "profiles": {
       "kubernetes": {
         "usages": [
            "signing",
             "key encipherment",
            "server auth",
             "client auth"
         ],
         "expiry": "87600h"
       }
     }
   }
 }
EOF
#创建证书签名请求文件
cat > ca-csr.json <<EOF
{
  "CN": "kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "ShenYang",
      "L": "ShenYang"
    }
  ]
}
EOF
#创建ca证书
cd /data/ssl/ && cfssl gencert -initca json/ca-csr.json | cfssljson -bare ca -
#创建/etc/k8s目录
mkdir -p /etc/kubernetes/cert
#复制CA证书和ca-config.json到目录中
cp ca*.pem ca-config.json /etc/kubernetes/cert/
#将证书和私钥下发到所有节点
for node_all_ip in 192.168.11.221 192.168.11.222
do
   echo ">>> ${node_all_ip}"
   ssh root@${node_all_ip} "mkdir -p /etc/kubernetes/cert"
   scp ca*.pem ca-config.json root@${node_all_ip}:/etc/kubernetes/cert
done
```
### 部署Etcd
```
#下载Etcd安装包
wget https://github.com/etcd-io/etcd/releases/download/v3.4.9/etcd-v3.4.9-linux-amd64.tar.gz
tar -xvf etcd-v3.4.9-linux-amd64.tar.gz
mkdir -p /data/etcd/bin
cp etcd-v3.4.9-linux-amd64/etcd* /data/etcd/bin/
#将etcd下发到所有节点
for node_all_ip in 192.168.11.221 192.168.11.222
do
   echo ">>> ${node_all_ip}"
   ssh root@${node_all_ip} "mkdir -p /opt/k8s/bin"
   scp etcd-v3.4.3-linux-amd64/etcd* root@${node_all_ip}:/opt/k8s/bin/
   ssh root@${node_all_ip} "chmod +x /opt/k8s/bin/*"
done
#建立证书
cat > etcd-csr.json <<EOF
{
  "CN": "etcd",
  "hosts": [
    "192.168.1.115",
    "192.168.1.116",
    "192.168.1.117"
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "ShenYang",
      "L": "ShenYang"
    }
  ]
}
EOF
配置说明：
hosts:制定授权使用该证书的etcd节点IP或域名列表，需要将etcd集群的三个节点IP都列在其中；

#生成证书和私钥
cfssl gencert -ca=/data/ssl/ca.pem \
   -ca-key=/data/ssl/ca-key.pem \
   -config=/data/ssl/json/ca-config.json \
   -profile=kubernetes /data/ssl/json/etcd-csr.json | cfssljson -bare etcd

建立etcd证书目录
mkdir -p /etc/etcd/cert
复制etcd证书到目录中
cp etcd*.pem /etc/etcd/cert/
将证书下发到所有etcd节点上
for node_all_ip in 192.168.11.221 192.168.11.222
do
   echo ">>> ${node_all_ip}"
   ssh root@${node_all_ip} "mkdir -p /etc/etcd/cert"
   scp etcd*.pem root@${node_all_ip}:/etc/etcd/cert/
done

#创建etcd的systemd unit模板脚本

----------------------------------
#!/bin/bash
#example: ./etcd.sh etcd01 192.168.11.220 etcd01=https://192.168.11.220:2380,etcd02=https://192.168.11.221:2380,etcd03=https://192.168.11.222:2380
 
NODE_ETCD_NAME=$1
NODE_ETCD_IP=$2
ETCD_NODES=$3
ETCD_DATA_DIR=/data/k8s/etcd/data
ETCD_WAL_DIR=/data/k8s/etcd/wal
 
if [ ! -d "/data/k8s/etcd/data /data/k8s/etcd/wal" ];then
  mkdir -p /data/k8s/etcd/data /data/k8s/etcd/wal
fi
 
cat > /etc/systemd/system/etcd.service <<EOF
[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target
Documentation=https://github.com/coreos
    
[Service]
Type=notify
WorkingDirectory=${ETCD_DATA_DIR}
ExecStart=/opt/k8s/bin/etcd \\
  --data-dir=${ETCD_DATA_DIR} \\
  --wal-dir=${ETCD_WAL_DIR} \\
  --name=${NODE_ETCD_NAME} \\
  --cert-file=/data/ssl/etcd.pem \\
  --key-file=/data/ssl/cert/etcd-key.pem \\
  --trusted-ca-file=/data/ssl/ca.pem \\
  --peer-cert-file=/data/ssl/etcd.pem \\
  --peer-key-file=/data/ssl/etcd-key.pem \\
  --peer-trusted-ca-file=/data/ssl/ca.pem \\
  --peer-client-cert-auth \\
  --client-cert-auth \\
  --listen-peer-urls=https://${NODE_ETCD_IP}:2380 \\
  --initial-advertise-peer-urls=https://${NODE_ETCD_IP}:2380 \\
  --listen-client-urls=https://${NODE_ETCD_IP}:2379,http://127.0.0.1:2379 \\
  --advertise-client-urls=https://${NODE_ETCD_IP}:2379 \\
  --initial-cluster-token=etcd-cluster-0 \\
  --initial-cluster=${ETCD_NODES} \\
  --initial-cluster-state=new \\
  --auto-compaction-mode=periodic \\
  --auto-compaction-retention=1 \\
  --max-request-bytes=33554432 \\
  --quota-backend-bytes=6442450944 \\
  --heartbeat-interval=250 \\
  --election-timeout=2000
Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF
 
systemctl daemon-reload
systemctl enable etcd
systemctl restart etcd
--------------------------------------------------

#启动etcd，此时会卡住不动，是因为etcd需要选举才能正常运行，所以要在另外两个节点也执行以下命令，记得修改$1和$2参数
/opt/k8s/work/etcd.sh etcd01 192.168.11.220 etcd01=https://192.168.11.220:2380,etcd02=https://192.168.11.221:2380,etcd03=https://192.168.11.222:2380
for node_all_ip in 192.168.11.221 192.168.11.222
do
  echo ">>> ${node_all_ip}"
  scp /opt/k8s/work/etcd.sh root@${node_all_ip}:/opt/k8s/
done
 
sh /opt/k8s/etcd.sh etcd02 192.168.11.221 etcd01=https://192.168.11.220:2380,etcd02=https://192.168.11.221:2380,etcd03=https://192.168.11.222:2380
sh /opt/k8s/etcd.sh etcd03 192.168.11.222 etcd01=https://192.168.11.220:2380,etcd02=https://192.168.11.221:2380,etcd03=https://192.168.11.222:2380

#当所有节点都执行完成后检查，状态是否都正常
 ETCDCTL_API=3 /data/etcd/bin/etcdctl --endpoints="https://192.168.1.115:2379,https://192.168.1.116:2379,https://192.168.1.117:2379" \
 --cacert=/data/ssl/ca.pem \
 --cert=/data/ssl/etcd.pem \
 --key=/data/ssl/etcd-key.pem endpoint health
 cd 
输出内容:
https://192.168.11.221:2379 is healthy: successfully committed proposal: took = 25.077164ms
https://192.168.11.220:2379 is healthy: successfully committed proposal: took = 38.10606ms
https://192.168.11.222:2379 is healthy: successfully committed proposal: took = 38.785388ms

#查看当前etcd集群工中的leader
for node_all_ip in 192.168.1.115 192.168.1.116 192.168.1.117
do
ETCDCTL_API=3 /data/etcd/bin/etcdctl -w table --cacert=/data/ssl/ca.pem \
--cert=/data/ssl/etcd.pem --key=/data/ssl/etcd-key.pem \
--endpoints=https://${node_all_ip}:2379 endpoint status
done

+-----------------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
|          ENDPOINT           |        ID        | VERSION | DB SIZE | IS LEADER | IS LEARNER | RAFT TERM | RAFT INDEX | RAFT APPLIED INDEX | ERRORS |
+-----------------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
| https://192.168.11.220:2379 | a6bbfb193c776e5c |   3.4.3 |   25 kB |      true |      false |       458 |         21 |                 21 |        |
+-----------------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
| https://192.168.11.221:2379 | 7b37d21aaf69f7d2 |   3.4.3 |   20 kB |     false |      false |       458 |         21 |                 21 |        |
+-----------------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
| https://192.168.11.222:2379 | 711ad351fe31c699 |   3.4.3 |   20 kB |     false |      false |       458 |         21 |                 21 |        |
+-----------------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
由上面结果可见，当前的leader节点为192.168.11.220

#如果报错：请检查所有服务器时间是否同步
rejected connection from "192.168.11.220:58360" (error "tls: failed to verify client's certificate: x509: certificate has expired or is not yet valid", ServerName "")
#如果遇到ETCD出现连接失败状况，导致创建实例失败，则etcd启动文件中的 --initial-cluster-state=new 改为 existing，重启则正常
```
### 部署ApiServer
```
#自签证书
##创建证书json文件
cd /data/ssl/json
cat > apiserver-csr.josn <<EOF
{
    "CN": "kubernetes",
    "hosts": [
      "10.0.0.1",
      "127.0.0.1",
      "kubernetes",
      "kubernetes.default",
      "kubernetes.default.svc",
      "kubernetes.default.svc.cluster",
      "kubernetes.default.svc.cluster.local",
      "192.168.1.115",
      "192.168.1.116",
      "192.168.1.117"
    ],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
     {
       "C": "CN",
       "ST": "ShenYang",
       "L": "ShenYang"
     }
    ]
}
EOF


#生成证书
cfssl gencert -ca=/data/ssl/ca.pem    -ca-key=/data/ssl/ca-key.pem    -config=/data/ssl/json/ca-config.json    -profile=kubernetes /data/ssl/json/apiserver-csr.json | cfssljson -bare apiserver

#生成token
head -c 16 /dev/urandom | od -An -t x | tr -d ' '  #升级16位token码

cat > /data/kubernetes/cert/token.csv << EOF
$(head -c 16 /dev/urandom | od -An -t x | tr -d ' '),kubelet-bootstrap,10001,"system:node-bootstrapper"
EOF


下载Master组件
下载地址： wget https://dl.k8s.io/v1.18.4/kubernetes-server-linux-amd64.tar.gz
上传tar包到服务器上，解压配置文件。
编写apiservice启动项
cat > /etc/systemd/system/kube-apiserver.service << EOF
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/kubernetes/kubernetes
[Service]
#EnvironmentFile=/opt/kubernetes/cfg/kube-apiserver.conf 
ExecStart=/data/k8s/server/bin/kube-apiserver --logtostderr=false --v=2 --log-dir=/data/k8s/logs --etcd-servers=https://192.168.1.115:2379,https://192.168.1.116:2379,https://192.168.1.117:2379 --bind-address=192.168.1.115 --secure-port=6443 --advertise-address=192.168.1.115 --allow-privileged=true --service-cluster-ip-range=10.0.0.0/24 --enable-admission-plugins=NamespaceLifecycle,LimitRanger,ServiceAccount,ResourceQuota,NodeRestriction --authorization-mode=RBAC,Node --enable-bootstrap-token-auth=true --token-auth-file=/data/ssl/token.csv --service-node-port-range=30000-32767 --kubelet-client-certificate=/data/ssl/apiserver.pem --kubelet-client-key=/data/ssl/apiserver-key.pem --tls-cert-file=/data/ssl/apiserver.pem --tls-private-key-file=/data/ssl/apiserver-key.pem --client-ca-file=/data/ssl/ca.pem --service-account-key-file=/data/ssl/ca-key.pem --etcd-cafile=/data/ssl/ca.pem --etcd-certfile=/data/ssl/etcd.pem --etcd-keyfile=/data/ssl/etcd-key.pem --audit-log-maxage=30 --audit-log-maxbackup=3 --audit-log-maxsize=100 --audit-log-path=/data/logs/k8s-audit.log
Restart=on-failure
[Install]
WantedBy=multi-user.target
EOF
重新加载systemctl配置
systemctl daemon-reload 
启动kube-apiserver
systemctl start kube-apiserver 
添加自起项
systemctl enable kube-apiserver
授权访问
/data/k8s/server/bin/kubectl create clusterrolebinding kubelet-bootstrap --clusterrole=system:node-bootstrapper --user=kubelet-bootstrap

参数说明：
–logtostderr：启用日志 
—v：日志等级 
–log-dir：日志目录 
–etcd-servers：etcd 集群地址 
–bind-address：监听地址 
–secure-port：https 安全端口 
–advertise-address：集群通告地址 
–allow-privileged：启用授权 
–service-cluster-ip-range：Service 虚拟 IP 地址段 
–enable-admission-plugins：准入控制模块
–authorization-mode：认证授权，启用 RBAC 授权和节点自管理 
–enable-bootstrap-token-auth：启用 TLS bootstrap 机制 
–token-auth-file：bootstrap token 文件 
–service-node-port-range：Service nodeport 类型默认分配端口范围 
–kubelet-client-xxx：apiserver 访问 kubelet 客户端证书 
–tls-xxx-file：apiserver https 证书 
–etcd-xxxfile：连接 Etcd 集群证书 
–audit-log-xxx：审计日志

```
### 部署kube-controller-manager
```
cat > /data/config/kube-controller-manager.conf << EOF
KUBE_CONTROLLER_MANAGER_OPTS="--logtostderr=false \\
--v=2 \\
--log-dir=/data/k8s/logs \\
--leader-elect=true \\
--master=127.0.0.1:8080 \\
--bind-address=127.0.0.1 \\
--allocate-node-cidrs=true \\
--cluster-cidr=10.244.0.0/16 \\ 
--service-cluster-ip-range=10.0.0.0/24 \\ 
--cluster-signing-cert-file=/data/ssl/ca.pem \\ 
--cluster-signing-key-file=/data/ssl/ca-key.pem \\ 
--root-ca-file=/data/ssl/ca.pem \\ 
--service-account-private-key-file=/data/ssl/ca-key.pem \\
--experimental-cluster-signing-duration=87600h0m0s"
EOF
参数说明：
--master:通过本地非安全本地端口8080连接apiserver。
--leader-elect:当该组件启动多个时，自动选举(HA)
--cluster-signing-cert-file/--cluster-signing-key-file:自动为kubelet颁发证书的CA，与apiserver 保持一致

/data/k8s/server/bin/kube-controller-manager --v=2 --log-dir=/data/k8s/logs --leader-elect=true --master=127.0.0.1:8080 --bind-address=127.0.0.1 \ --allocate-node-cidrs=true \ --cluster-cidr=10.244.0.0/16 \ --service-cluster-ip-range=10.0.0.0/24 \ --cluster-signing-cert-file=/opt/k8s/work/ca.pem \ --cluster-signing-key-file=/opt/k8s/work/ca-key.pem \ --root-ca-file=/opt/k8s/work/ca.pem \ --service-account-private-key-file=/opt/k8s/work/ca-key.pem \ --experimental-cluster-signing-duration=87600h0m0s“

systemctl管理kube-controller-manager
cat > /etc/systemd/system/kube-controller-manager.service << EOF
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/kubernetes/kubernetes
[Service]
EnvironmentFile=/data/config/kube-controller-manager.conf
ExecStart=/data/k8s/server/bin/kube-controller-manager $KUBE_CONTROLLER_MANAGER_OPTS
Restart=on-failure
[Install]
WantedBy=multi-user.target
EOF

启动并添加自启服务
systemctl daemon-reload && systemctl start kube-controller-manager && systemctl enable kube-controller-manager
```
### 部署kube-scheduler
```
创建配置文件
cat > /data/config/kube-scheduler.conf << EOF
KUBE_SCHEDULER_dataS="--logtostderr=false \\
--v=2 \\
--log-dir=/data/k8s/logs \\
--leader-elect \\
--master=127.0.0.1:8080 \\
--bind-address=127.0.0.1"
EOF
#参数说明
--master:通过本地非安全本地端口8080连接apiserver。
--leader-elect:当该组件启动多个时，自动选举(HA)
systemctl管理kube-scheduler
cat > /etc/systemd/system/kube-scheduler.service << EOF
[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/kubernetes/kubernetes
[Service]
EnvironmentFile=/data/kubernetes/cfg/kube-scheduler.conf
ExecStart=/data/k8s/server/bin/kube-scheduler \$KUBE_SCHEDULER_dataS
Restart=on-failure
[Install]
WantedBy=multi-user.target
EOF
另一种写法直接在service中加入参数
/data/k8s/server/bin/kube-scheduler --logtostderr=false --v=2 --log-dir=/data/k8s/logs--leader-elect --master=127.0.0.1:8080 --bind-address=127.0.0.1
启动服务并添加自启动项
systemctl daemon-reload && systemctl start kube-scheduler && systemctl enable kube-scheduler
```
### 验证部署结果
```
/data/k8s/server/bin/kubectl get cs

controller-manager   Healthy   ok
scheduler            Healthy   ok
etcd-2               Healthy   {"health":"true"}
etcd-0               Healthy   {"health":"true"}
etcd-1               Healthy   {"health":"true"}

```

### Node节点部署
#### 部署kubelet
```
创建kubelet配置文件
cat > /data/config/kubelet.conf << EOF
KUBELET_OPTS="--logtostderr=false \\
--v=2 \\
--log-dir=/data/k8s/logs \\
--hostname-override=k8s-master \\
--network-plugin=cni \\
--kubeconfig=/data/k8s/config/kubelet.kubeconfig \\
--bootstrap-kubeconfig=/data/k8s/config/bootstrap.kubeconfig \\
--config=/data/k8s/config/kubelet-config.yml \\
--cert-dir=/data/k8s/ssl \\
--pod-infra-container-image=rancher/pause-amd64:3.1"
EOF
参数说明:
--hostname-override:显示名称，集群中唯一
--network-plugin:启用CNI
--kubeconfig:空路径，会自动生成，后面用于连接apiserver
--bootstrap-kubeconfig:首次启动向apiserver申请证书
--config:配置参数文件
--cert-dir:kubelet证书生成目录
--pod-infra-container-image:管理Pod网络容器的镜像

参数配置文件
cat > /data/config/kubelet-config.yml << EOF 
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
address: 0.0.0.0
port: 10250
readOnlyPort: 10255
cgroupDriver: cgroupfs
clusterDNS:
- 10.0.0.2
clusterDomain: cluster.local
failSwapOn: false
authentication:
  anonymous:
    enabled: false
  webhook:
    cacheTTL: 2m0s
    enabled: true
  x509:
    clientCAFile: /data/ssl/ca.pem
authorization:
  mode: Webhook
  webhook:
    cacheAuthorizedTTL: 5m0s
    cacheUnauthorizedTTL: 30s
evictionHard:
  imagefs.available: 15%
  memory.available: 100Mi
  nodefs.available: 10%
  nodefs.inodesFree: 5%
maxOpenFiles: 1000000
maxPods: 110
EOF

生成bootstrap.kubeconfig文件
/data/k8s/server/bin/kubectl config set-cluster kubernetes --certificate-authority=/data/ssl/ca.pem --embed-certs=true --server="https://192.168.1.115:6443" --kubeconfig=bootstrap.kubeconfig
/data/k8s/server/bin/kubectl config set-credentials "kubelet-bootstrap" --token="027749aa9af39ae28704d045cb1e7e6e" --kubeconfig=bootstrap.kubeconfig
/data/k8s/server/bin/kubectl config set-context default --cluster=kubernetes --user="kubelet-bootstrap" --kubeconfig=bootstrap.kubeconfig
/data/k8s/server/bin/kubectl config use-context default --kubeconfig=bootstrap.kubeconfig
/data/k8s/server/bin/kubectl create clusterrolebinding kubelet-bootstrap --clusterrole=system:node-bootstrapper --user=kubelet-bootstrap

systemctl管理kubelet
cat > /etc/systemd/system/kubelet.service << EOF 
[Unit]
Description=Kubernetes Kubelet
After=docker.service
[Service]
EnvironmentFile=/data/config/kubelet.conf
ExecStart=/data/k8s/server/bin/kubelet $KUBELET_OPTS
Restart=on-failure
LimitNOFILE=65536
[Install]
WantedBy=multi-user.target
EOF
#替换ExecStart=后的另一种写法
/data/k8s/server/bin/kubelet --logtostderr=false --v=2 --log-dir=/data/k8s/logs --hostname-override=k8s-master --network-plugin=cni --kubeconfig=/data/config/kubelet.kubeconfig --bootstrap-kubeconfig=/data/config/bootstrap.kubeconfig --config=/data/config/kubelet-config.yml --cert-dir=/data/ssl --pod-infra-container-image=rancher/pause-amd64:3.1

启动服务并添加自启项
systemctl daemon-reload && systemctl start kubelet && systemctl enable kubelet

#批准kubelet证书申请并加入集群
/data/k8s/server/bin/kubectl get csr 

NAME                                                   AGE     SIGNERNAME                                    REQUESTOR           CONDITION
node-csr-PANqxN71HG4bwdKjn8DksPh3ZCb_Dr2gRKgzq5t_kck   4m26s   kubernetes.io/kube-apiserver-client-kubelet   kubelet-bootstrap   Pending

批准加入
/data/k8s/server/bin/kubectl certificate approve node-csr-PANqxN71HG4bwdKjn8DksPh3ZCb_Dr2gRKgzq5t_kck

NAME                                                   AGE     SIGNERNAME                                    REQUESTOR           CONDITION
node-csr-PANqxN71HG4bwdKjn8DksPh3ZCb_Dr2gRKgzq5t_kck   5m46s   kubernetes.io/kube-apiserver-client-kubelet   kubelet-bootstrap   Approved,Issued

/data/k8s/server/bin/kubectl get node

NAME         STATUS     ROLES    AGE   VERSION
k8s-master   NotReady   <none>   49s   v1.18.4

```

#### 部署kube-proxy
```
创建proxy配置文件
cat > /data/config/kube-proxy.conf << EOF 
KUBE_PROXY_OPTS="--logtostderr=false \\
--v=2 \\
--log-dir=/data/k8s/logs \\
--config=/data/config/kube-proxy-config.yml"
EOF

配置参数文件
cat > /data/config/kube-proxy-config.yml << EOF
kind: KubeProxyConfiguration
apiVersion: kubeproxy.config.k8s.io/v1alpha1
bindAddress: 0.0.0.0
metricsBindAddress: 0.0.0.0:10249
clientConnection:
  kubeconfig: /data/config/kube-proxy.kubeconfig
hostnameOverride: k8s-master
clusterCIDR: 10.0.0.0/24
EOF

生成kube-proxy.kubeconfig文件
#签发证书
cd /data/ssl/json
cat > kube-proxy-csr.json << EOF
{
    "CN":"kubernetes",
    "hosts":[

    ],
    "key":{
        "algo":"rsa",
        "size":2048
    },
    "names":[
        {
            "C":"CN",
            "L":"ShenYang",
            "ST":"ShenYang"
        }
    ]
}
EOF

生成证书
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=json/ca-config.json -profile=kubernetes json/kube-proxy-csr.json | cfssljson -bare kube-proxy

生成kubeconfig配置文件
/data/k8s/server/bin/kubectl config set-cluster kubernetes --certificate-authority=/data/ssl/ca.pem --embed-certs=true --server="https://192.168.1.115:6443" --kubeconfig=/data/config/kube-proxy.kubeconfig
/data/k8s/server/bin/kubectl config set-credentials kube-proxy --client-certificate=/data/ssl/kube-proxy.pem --client-key=/data/ssl/kube-proxy-key.pem --embed-certs=true --kubeconfig=/data/config/kube-proxy.kubeconfig
/data/k8s/server/bin/kubectl config set-context default --cluster=kubernetes --user=kube-proxy --kubeconfig=/data/config/kube-proxy.kubeconfig
/data/k8s/server/bin/kubectl config use-context default --kubeconfig=/data/config/kube-proxy.kubeconfig

systemctl 管理 kube-porxy
cat > /etc/systemd/system/kube-proxy.service << EOF
[Unit]
Description=Kubernetes Proxy
After=network.target
[Service]
EnvironmentFile=/data/config/kube-proxy.conf
ExecStart=/data/k8s/server/bin/kube-proxy $KUBE_PROXY_OPTS
Restart=on-failure
LimitNOFILE=65536
[Install]
WantedBy=multi-user.target
EOF
 #替换ExecStart=后的另一种写法
/data/k8s/server/bin/kube-proxy --logtostderr=false --v=2 --log-dir=/data/kubernetes/logs --config=/data/config/kube-proxy-config.yml

启动服务并设置开机启动
systemctl daemon-reload && systemctl start kube-proxy && systemctl enable kube-proxy
```
#### 部署CNI
```
cd ~
wget https://github.com/containernetworking/plugins/releases/download/v0.8.6/cni-plugins-linux-amd64-v0.8.6.tgz &&  mkdir /opt/cni/bin -p
tar -zxvf cni-plugins-linux-amd64-v0.8.6.tgz -C /opt/cni/bin/
```

#### 部署flnnel
```
cd ~
wget https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
sed -i -r "s#quay.io/coreos/flannel:.*-amd64#lizhenliang/flannel:v0.12.0-amd64#g" kube-flannel.yml
/data/k8s/server/bin/kubectl apply -f kube-flannel.yml
```

#### 添加work节点
```
# kubelet,kube-proxy拷贝
scp /data/kubernetes/bin/kubelet root@192.168.56.15:/data/kubernetes/bin/
scp /data/kubernetes/bin/kube-proxy root@192.168.56.15:/data/kubernetes/bin/
scp /data/kubernetes/bin/kubelet root@192.168.56.16:/data/kubernetes/bin/
scp /data/kubernetes/bin/kube-proxy root@192.168.56.16:/data/kubernetes/bin/
# cni插件拷贝
scp -rp /opt/cni/ root@192.168.56.15:/opt
scp -rp /opt/cni/ root@192.168.56.16:/opt
# 证书拷贝
scp /data/kubernetes/ssl/ca.pem 192.168.56.15:/data/kubernetes/ssl/
scp /data/kubernetes/ssl/ca.pem 192.168.56.16:/data/kubernetes/ssl/
# 配置文件拷贝
scp /data/kubernetes/cfg/kube-proxy* 192.168.56.15:/data/kubernetes/cfg/
scp /data/kubernetes/cfg/kube-proxy* 192.168.56.16:/data/kubernetes/cfg/
scp /data/kubernetes/cfg/kubelet* 192.168.56.15:/data/kubernetes/cfg/
scp /data/kubernetes/cfg/kubelet* 192.168.56.16:/data/kubernetes/cfg/
scp /data/kubernetes/cfg/bootstrap.kubeconfig 192.168.56.15:/data/kubernetes/cfg/
scp /data/kubernetes/cfg/bootstrap.kubeconfig 192.168.56.16:/data/kubernetes/cfg/
# 启动文件拷贝
scp /usr/lib/systemd/system/kubelet.service 192.168.56.15:/usr/lib/systemd/system/
scp /usr/lib/systemd/system/kubelet.service 192.168.56.16:/usr/lib/systemd/system/
scp /usr/lib/systemd/system/kube-proxy.service 192.168.56.15:/usr/lib/systemd/system/
scp /usr/lib/systemd/system/kube-proxy.service 192.168.56.16:/usr/lib/systemd/system/

重新生成证书和配置文件

删除证书和配置文件
rm /data/config/kubelet.kubeconfig
rm -f /datassl/kubelet*

配置节点信息
修改kubelet和kube-proxy配置文件
vim /data/config/kubelet.conf 
--hostname-override=node1
vim /data/config/kube-proxy-config.yml 
hostnameOverride: node1

配置kubectl和kube-proxy开机启动
systemctl daemon-reload && systemctl start kubelet && systemctl start kube-proxy
systemctl enable kubelet && systemctl enable kube-proxy

```

#### 部署Dashboard和CoreDNS
```
部署Dashboard
下载recommended.yaml
地址：https://github.com/kubernetes/dashboard/blob/master/aio/deploy/recommended.yaml

vim recommended.yaml 只有一处直接修改

kind: Service
apiVersion: v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kubernetes-dashboard
spec:
  ports:
    - port: 443
      targetPort: 8443
      nodePort: 30001
  type: NodePort
  selector:
    k8s-app: kubernetes-dashboard

部署Dashborad镜像
kubectl apply -f recommended.yml 

查看部署状态
kubectl get pods,svc -n kubernetes-dashboard

NAME                                             READY   STATUS    RESTARTS   AGE
pod/dashboard-metrics-scraper-6b4884c9d5-m25wt   1/1     Running   0          6m41s
pod/kubernetes-dashboard-7f99b75bf4-g26pq        1/1     Running   0          6m41s

NAME                                TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)    AGE
service/dashboard-metrics-scraper   ClusterIP   10.0.0.198   <none>        8000/TCP   6m41s
service/kubernetes-dashboard        NodePort    10.0.0.35    <none>        443:30001/TCP   40m


创建dashboard访问token
kubectl create serviceaccount dashboard-admin -n kube-system
kubectl create clusterrolebinding dashboard-admin --clusterrole=cluster-admin --serviceaccount=kube-system:dashboard-admin
kubectl describe secrets -n kube-system $(kubectl -n kube-system get secret | awk '/dashboard-admin/{print $1}')     #获取token

Name:         dashboard-admin-token-td855
Namespace:    kube-system
Labels:       <none>
Annotations:  kubernetes.io/service-account.name: dashboard-admin
              kubernetes.io/service-account.uid: d916f278-f00d-41dc-823a-d55e7644a36f

Type:  kubernetes.io/service-account-token

Data
====
ca.crt:     1281 bytes
namespace:  11 bytes
token:      eyJhbGciOiJSUzI1NiIsImtpZCI6IjVjTl84WGZ2YThhN3U2Szc5Qk5kOEIzNV9nMVA4Yy1vMHlSMzJuUzNTaDgifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJkYXNoYm9hcmQtYWRtaW4tdG9rZW4tdGQ4NTUiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC5uYW1lIjoiZGFzaGJvYXJkLWFkbWluIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQudWlkIjoiZDkxNmYyNzgtZjAwZC00MWRjLTgyM2EtZDU1ZTc2NDRhMzZmIiwic3ViIjoic3lzdGVtOnNlcnZpY2VhY2NvdW50Omt1YmUtc3lzdGVtOmRhc2hib2FyZC1hZG1pbiJ9.Gh0_rq6lXC9JrBcuFA8p15gfTQglkrWn0D6JMjZeO_XmOcWTW2PPDNlXUtxA3HfBpAIgqVbPJOH4Gu4Z-1J1DvwmjKVdN-tMmNsLDAWZTkY-F0mJSyNPgOlVb_reYoPqOMSa0q0K7oJL77gnbV_azwMblbw0rP8ygN9Z-F_zDFst9C29-DV4_qcL8xaBqS4ewl7f4DNmwAfl3xNTwoR9CD_x4lFSMYrB3kmuVWQipuepbNkobC8ZwMqiCaD3IqvrySEABqH8P5rAytmLLatHVS8qf2VINtLopPr6ZprNXn9PxD1__e22M6xICqV7mDtDMBTxjyFqzDl3xdY3NuMxlQ

谷歌浏览器访问的时候证书存在问题，需要重新自签证书才能访问
cd /data/ssl/
cat > dashboard-csr.json <<EOF
{
    "CN":"system:kubernetes-dashboard",
    "hosts":[

    ],
    "key":{
        "algo":"rsa",
        "size":2048
    },
    "names":[
        {
            "C":"CN",
            "L":"BeiJing",
            "ST":"BeiJing",
            "O":"k8s",
            "OU":"System"
        }
    ]
}
EOF

建立证书
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=json/ca-config.json -profile=kubernetes json/dashboard-csr.json | cfssljson -bare kubernetes-dashboard

删除默认的secret,使用自签证书创建新的secret
#删除secret
kubectl delete secret kubernetes-dashboard-certs -n kubernetes-dashboard
#自签证书创建新的secret
kubectl create secret generic kubernetes-dashboard-certs --from-file=/data/ssl/kubernetes-dashboard-key.pem --from-file=/data/ssl/kubernetes-dashboard.pem -n kubernetes-dashboard

修改dasoboard.yml (recommanded.yml)
位置只有一直接修改即可
args:
            - --auto-generate-certificates
            - --tls-key-file=kubernetes-dashboard-key.pem
            - --tls-cert-file=kubernetes-dashboard.pem
            - --namespace=kubernetes-dashboard
# apply
kubectl apply -f recommended.yaml
# 查看token，使用这个token，可以直接访问https://NodeIP:30001  NodeIP也就是宿主机的IP
kubectl describe secrets -n kube-system $(kubectl -n kube-system get secret | awk '/dashboard-admin/{print $1}')
Name:         dashboard-admin-token-td855
Namespace:    kube-system
Labels:       <none>
Annotations:  kubernetes.io/service-account.name: dashboard-admin
              kubernetes.io/service-account.uid: d916f278-f00d-41dc-823a-d55e7644a36f

Type:  kubernetes.io/service-account-token

Data
====
ca.crt:     1281 bytes
namespace:  11 bytes
token:      eyJhbGciOiJSUzI1NiIsImtpZCI6IjVjTl84WGZ2YThhN3U2Szc5Qk5kOEIzNV9nMVA4Yy1vMHlSMzJuUzNTaDgifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJkYXNoYm9hcmQtYWRtaW4tdG9rZW4tdGQ4NTUiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC5uYW1lIjoiZGFzaGJvYXJkLWFkbWluIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQudWlkIjoiZDkxNmYyNzgtZjAwZC00MWRjLTgyM2EtZDU1ZTc2NDRhMzZmIiwic3ViIjoic3lzdGVtOnNlcnZpY2VhY2NvdW50Omt1YmUtc3lzdGVtOmRhc2hib2FyZC1hZG1pbiJ9.Gh0_rq6lXC9JrBcuFA8p15gfTQglkrWn0D6JMjZeO_XmOcWTW2PPDNlXUtxA3HfBpAIgqVbPJOH4Gu4Z-1J1DvwmjKVdN-tMmNsLDAWZTkY-F0mJSyNPgOlVb_reYoPqOMSa0q0K7oJL77gnbV_azwMblbw0rP8ygN9Z-F_zDFst9C29-DV4_qcL8xaBqS4ewl7f4DNmwAfl3xNTwoR9CD_x4lFSMYrB3kmuVWQipuepbNkobC8ZwMqiCaD3IqvrySEABqH8P5rAytmLLatHVS8qf2VINtLopPr6ZprNXn9PxD1__e22M6xICqV7mDtDMBTxjyFqzDl3xdY3NuMxlQ

https://宿主机IP:30001 使用tonken登录

```

#### 部署coredns
```
下载coredns.yaml，上传到服务器
kubectl apply -f coredns.yaml 部署coredns
tail -f /data/k8s/logs/kubelet.INFO 查看部署完成
验证coredns
kubectl run -it --rm dns-test --image=busybox:1.28.4 sh
If you don't see a command prompt, try pressing enter.
/ # nslookup kubernetes
Server: 10.0.0.2
Address 1: 10.0.0.2 kube-dns.kube-system.svc.cluster.local
Name: kubernetes
Address 1: 10.0.0.1 kubernetes.default.svc.cluster.local
```

## kubeadm安装
### Master节点
```
安装k8s组件
yum install -y kubelet-1.18.12 kubelet-1.18.12 kubectl-1.18.12 kubeadm-1.18.12
添加k8s启动项
systemctl enable kubelet
kubeadm安装环境
kubeadm init --apiserver-advertise-address=master主机IP --image-repository registry.aliyuncs.com/google_containers --kubernetes-version v1.18.12 --service-cidr=10.50.0.0/12 --pod-network-cidr=10.200.0.0/16
等待安装完成，完成后查看Your Kubernetes control-plane has initialized successfully!下信息

执行如下命令，master上需要执行：

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config


加入节点命令如下，sha256 token是24小时有效，失效后需要重新生成。刚装好时需要查看一下：

Then you can join any number of worker nodes by running the following on each as root:
kubeadm join 192.168.1.115:6443 --token zir09t.18llsxdaworfjb1h \
    --discovery-token-ca-cert-hash sha256:27ba50bf5aaa686ef05ea514170dbac65e15bbee5e82a5c5ebafbea42301e55b


coredns 是运行在master节点上的，如果运行在node中，会造成无法访问master上的kubernetes的443端口

```
### Node节点
```
安装k8s组件
yum install -y kubelet-1.18.12 kubelet-1.18.12 kubectl-1.18.12 kubeadm-1.18.12
添加k8s启动项
systemctl enable kubelet

向集群添加新节点,执行在kubeadm init 输出的kubeadm join命令：
kubeadm join MasterIP:端口 --token esce21.q6hetwm8si29qxwn --discovery-token-ca-cert-hash
默认token有效期为24小时，当过期之后，该token就不可用了。这时就需要重新创建token,操作如下：
kubeadm token create --print-join-command
```
### 部署CNI网络
```
wget https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
默认镜像地址无法访问，sed命令修改为docker hub镜像仓库,kube-flannel.yml文件git地址:https://github.com/coreos/flannel/blob/master/Documentation。
下载仓库文件到master服务器上，执行如下命令：
kubectl apply -f kube-flannel.yml

kubectl get pods -n kube-system
```
