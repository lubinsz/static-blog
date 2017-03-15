---
title: Kubernetes 手動安裝教學
date: 2017-1-14 17:08:54
layout: page
categories:
- Kubernetes
tags:
- Kubernetes
- Linux
- Docker
---
Kubernetes 提供了許多雲端平台與作業系統的安裝方式，本章將以`全手動安裝方式`來部署，主要是學習與了解 Kubernetes 建置流程。若想要瞭解更多平台的部署可以參考 [Picking the Right Solution](https://kubernetes.io/docs/getting-started-guides/)來選擇自己最喜歡的方式。

本次安裝版本為：
* Kubernetes v1.5.4
* Etcd v3.1.1
* Flannel v0.7.0

<!--more-->

## 預先準備資訊
本教學將以下列節點數與規格來進行部署 Kubernetes 叢集，作業系統可採用`Ubuntu 16.x`與`CentOS 7.x`：

| IP Address  |   Role   |   CPU    |   Memory   |
|-------------|----------|----------|------------|
|172.16.35.12 |  master1 |    1     |     2G     |
|172.16.35.10 |  node1   |    1     |     2G     |
|172.16.35.11 |  node2   |    1     |     2G     |

> 這邊 master 為主要控制節點，node 為應用程式工作節點。

首先安裝前要確認以下幾項都已將準備完成：
* 所有節點彼此網路互通，並且不需要 SSH 密碼即可登入。
* 在`所有`節點安裝 Docker engine：
```sh
$ curl -fsSL https://get.docker.com/ | sh
```

## Etcd 安裝與設定
在開始安裝 Kubernetes 之前，需要先將一些必要系統建置完成，其中 Etcd 就是 Kubernetes 最為需要的一環，Kubernetes 會將部分資訊儲存於 Etcd 上，來提供給其他節點索取，以確保整個叢集的狀態。

首先在`master`節點下載 Etcd，並解壓縮放到 /opt 底下與安裝：
```sh
$ cd /opt
$ wget -qO- "https://github.com/coreos/etcd/releases/download/v3.1.1/etcd-v3.1.1-linux-amd64.tar.gz" | tar -zx
$ mv etcd-v3.1.1-linux-amd64 etcd
$ cd etcd/ && ln etcd /usr/bin/ && ln etcdctl /usr/bin/
```

完成後新建 Etcd Group 與 User，並建立 Etcd 設定檔目錄：
```sh
$ groupadd etcd
$ useradd -c "Etcd user" -g etcd -s /sbin/nologin -r etcd
$ mkdir /etc/etcd
```

新增`/etc/etcd/etcd.conf`檔案，加入以下內容：
```sh
$ cat <<EOF > /etc/etcd/etcd.conf
ETCD_NAME=master1
ETCD_DATA_DIR=/var/lib/etcd
ETCD_INITIAL_ADVERTISE_PEER_URLS=http://172.16.35.12:2380
ETCD_INITIAL_CLUSTER=master1=http://172.16.35.12:2380
ETCD_INITIAL_CLUSTER_STATE=new
ETCD_INITIAL_CLUSTER_TOKEN=etcd-k8s-cluster
ETCD_LISTEN_PEER_URLS=http://0.0.0.0:2380
ETCD_ADVERTISE_CLIENT_URLS=http://172.16.35.12:2379
ETCD_LISTEN_CLIENT_URLS="http://0.0.0.0:2379"
ETCD_PROXY="off"
EOF
```

新增`/lib/systemd/system/etcd.service`來管理 Etcd，並加入以下內容：
```sh
$ cat <<EOF > /lib/systemd/system/etcd.service
[Unit]
Description=Etcd - Highly Available key value store
Documentation=man:etcd
After=network.target
Wants=network-online.target

[Service]
Environment=DAEMON_ARGS=
Environment=ETCD_NAME=%H
Environment=ETCD_DATA_DIR=/var/lib/etcd/default
EnvironmentFile=-/etc/default/%p
EnvironmentFile=-/etc/etcd/etcd.conf
Type=notify
User=etcd
PermissionsStartOnly=true
ExecStart=/usr/bin/etcd \$DAEMON_ARGS
Restart=on-abnormal
#RestartSec=10s
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
Alias=etcd2.service
EOF
```

建立 var 存放資訊，然後啟動 Etcd 服務:
```sh
$ mkdir -p /var/lib/etcd && chown etcd:etcd -R /var/lib/etcd
$ systemctl enable etcd.service
$ systemctl start etcd.service
```

透過簡單指令驗證：
```sh
$ etcdctl cluster-health
member 95b428c288413b46 is healthy: got healthy result from http://172.16.35.12:2379
cluster is healthy
```

## Flannel 安裝與設定
Flannel 是 CoreOS 團隊針對 Kubernetes 設計的一個`覆蓋網絡(Overlay Network)`工具，其目的在於幫助每一個使用 Kuberentes 的主機擁有一個完整的子網路。

首先在`所有`節點下載 Flannel，並執行以下步驟。首先解壓縮放到 /opt 底下與安裝：
```sh
$ cd /opt && mkdir flannel
$ wget -qO- "https://github.com/coreos/flannel/releases/download/v0.7.0/flannel-v0.7.0-linux-amd64.tar.gz" | tar -zxC flannel/
$ cd flannel/ && ln flanneld /usr/bin/ && ln mk-docker-opts.sh /usr/bin/
```

建立 Docker Drop-in 目錄，並新增`flannel.conf`檔案：
```sh
$ mkdir -p /etc/systemd/system/docker.service.d
$ cat <<EOF > /etc/systemd/system/docker.service.d/flannel.conf
[Service]
EnvironmentFile=-/run/flannel/docker
EOF
```

新增`/etc/default/flanneld`檔案，加入以下內容：
```sh
$ cat <<EOF > /etc/default/flanneld
FLANNEL_ETCD_ENDPOINTS="http://172.16.35.12:2379"
FLANNEL_ETCD_PREFIX="/atomic.io/network"
FLANNEL_OPTIONS="--iface=enp0s8"
EOF
```
> `FLANNEL_OPTIONS`可以依據需求加入，這邊主要指定 flannel 使用的網卡。

新增`/lib/systemd/system/flanneld.service`來管理 Flannel：
```sh
$ cat <<EOF > /lib/systemd/system/flanneld.service
[Unit]
Description=Flanneld overlay address etcd agent
After=network.target
After=network-online.target
Wants=network-online.target
After=etcd.service
Before=docker.service

[Service]
Type=notify
EnvironmentFile=/etc/default/flanneld
ExecStart=/usr/bin/flanneld -etcd-endpoints=\${FLANNEL_ETCD_ENDPOINTS} -etcd-prefix=\${FLANNEL_ETCD_PREFIX} \${FLANNEL_OPTIONS}
ExecStartPost=/usr/bin/mk-docker-opts.sh -d /run/flannel/docker
Restart=on-failure

[Install]
WantedBy=multi-user.target
RequiredBy=docker.service
EOF
```

接著回到`master1`節點，新增一個`/tmp/flannel-config.json`檔，並加入以下內容：
```sh
$ cat <<EOF > /tmp/flannel-config.json
{ "Network": "172.20.0.0/16", "SubnetLen": 24, "Backend": { "Type": "vxlan"}}
EOF
```

然後將 Flannel 網路設定儲存到 etcd 中：
```sh
$ etcdctl --no-sync set /atomic.io/network/config < /tmp/flannel-config.json
$ etcdctl ls /atomic.io/network/
/atomic.io/network/config
```

之後到`每台`節點啟動 Flannel:
```sh
$ systemctl enable flanneld.service
$ systemctl start flanneld.service
```

完成後透過以下指令簡單驗證：
```sh
$ ip -4 addr show flannel.1

5: flannel.1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UNKNOWN group default
    inet 172.20.96.0/32 scope global flannel.1
       valid_lft forever preferred_lft forever
```

確認有網路後，修改`/lib/systemd/system/docker.service`檔案以下內容：
```
ExecStart=/usr/bin/dockerd -H fd:// $DOCKER_OPTS
```

重新啟動 Docker 來使用 Flannel：
```sh
$ systemctl daemon-reload
$ systemctl restart docker
$ ip -4 a show docker0

4: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default
    inet 172.20.96.1/24 scope global docker0
       valid_lft forever preferred_lft forever
```

最後在任一台節點去 Ping 其他節點的 docker0 網路。

## Kubernetes Master 安裝與設定
Master 是 Kubernetes 的大總管，主要建置`API Server`、`Controller Manager Server`與`Scheduler`來元件管理所有 Node。首先加入取得 Packages 來源，並安裝：
```sh
$ curl -s "https://packages.cloud.google.com/apt/doc/apt-key.gpg" | apt-key add -
$ echo "deb http://apt.kubernetes.io/ kubernetes-xenial main" > /etc/apt/sources.list.d/kubernetes.list
$ apt-get update && apt-get install -y kubectl kubelet kubernetes-cni
```

然後準備 OpenSSL 的設定檔資訊：
```sh
$ mkdir -p /etc/kubernetes/pki
$ DIR=/etc/kubernetes/pki
$ cat <<EOF > ${DIR}/openssl.conf
[req]
req_extensions = v3_req
distinguished_name = req_distinguished_name
[req_distinguished_name]
[ v3_req ]
basicConstraints = CA:FALSE
keyUsage = nonRepudiation, digitalSignature, keyEncipherment
subjectAltName = @alt_names
[alt_names]
DNS.1 = kubernetes
DNS.2 = kubernetes.default
DNS.3 = kubernetes.default.svc
DNS.4 = kubernetes.default.svc.cluster.local
DNS.5 = test.com
IP.1 = 192.160.0.1
IP.2 = 172.16.35.12
EOF

$ cat <<EOF > ${DIR}/openssl-etcd.conf
[req]
req_extensions = v3_req
distinguished_name = req_distinguished_name
[req_distinguished_name]
[ v3_req ]
extendedKeyUsage = clientAuth,serverAuth
basicConstraints = CA:FALSE
keyUsage = nonRepudiation, digitalSignature, keyEncipherment
subjectAltName = @alt_names
[alt_names]
DNS.1 = test.com
IP.1 = 172.16.35.12
IP.2 = 172.16.35.10
IP.3 = 172.16.35.11
EOF
```
> 細節請參考[Cluster TLS using OpenSSL](https://coreos.com/kubernetes/docs/latest/openssl.html)。

建立 OpenSSL Keypairs 與 Certificate：
```sh
openssl genrsa -out ${DIR}/ca-key.pem 2048
openssl req -x509 -new -nodes -key ${DIR}/ca-key.pem -days 1000 -out ${DIR}/ca.pem -subj '/CN=kube-ca'
openssl genrsa -out ${DIR}/admin-key.pem 2048
openssl req -new -key ${DIR}/admin-key.pem -out ${DIR}/admin.csr -subj '/CN=kube-admin'
openssl x509 -req -in ${DIR}/admin.csr -CA ${DIR}/ca.pem -CAkey ${DIR}/ca-key.pem -CAcreateserial -out ${DIR}/admin.pem -days 1000
openssl genrsa -out ${DIR}/apiserver-key.pem 2048
openssl req -new -key ${DIR}/apiserver-key.pem -out ${DIR}/apiserver.csr -subj '/CN=kube-apiserver' -config ${DIR}/openssl.conf
openssl x509 -req -in ${DIR}/apiserver.csr -CA ${DIR}/ca.pem -CAkey ${DIR}/ca-key.pem -CAcreateserial -out ${DIR}/apiserver.pem -days 1000 -extensions v3_req -extfile ${DIR}/openssl.conf
```
> 細節請參考[Cluster TLS using OpenSSL](https://coreos.com/kubernetes/docs/latest/openssl.html)。


接著下載 Kubernetes 相關檔案至`/etc/kubernetes`：
```sh
$ cd /etc/kubernetes/
$ URL="https://gist.githubusercontent.com/kairen/08077dff404e0bcd949cd97536c03595/raw/bbb3ae52d9e2504e61f932dfe6e59eee22acf97a/"
$ wget ${URL}/kube-apiserver.json -O manifests/kube-apiserver.json
$ wget ${URL}/kube-controller-manager.json -O manifests/kube-controller-manager.json
$ wget ${URL}/kube-scheduler.json -O manifests/kube-scheduler.json
$ wget ${URL}/admin.conf -O admin.conf
$ wget ${URL}/kubelet -O kubelet
```
> 若`IP`與教學設定不同的話，請記得修改。

新增`/lib/systemd/system/kubelet.service`來管理 kubelet：
```sh
$ cat <<EOF > /lib/systemd/system/kubelet.service
[Unit]
Description=Kubernetes Kubelet Server
After=docker.service
Requires=docker.service

[Service]
WorkingDirectory=/var/lib/kubelet
EnvironmentFile=-/etc/kubernetes/kubelet
ExecStart=/usr/bin/kubelet \$KUBELET_ADDRESS \$KUBELET_POD_INFRA_CONTAINER \
\$KUBELET_ARGS \$KUBE_NODE_LABEL $KUBE_LOGTOSTDERR \
\$KUBE_ALLOW_PRIV \$KUBELET_NETWORK_ARGS \
\$KUBELET_DNS_ARGS
Restart=always

[Install]
WantedBy=multi-user.target
EOF
```

最後建立 var 存放資訊，然後啟動 kubelet 服務:
```sh
$ mkdir -p /var/lib/kubelet
$ systemctl daemon-reload
$ systemctl restart kubelet.service
```

完成後會需要一段時間下載元件的 Docker image，可以用以下指令去檢查是否已啟動：
```sh
$ netstat -ntlp
tcp   0  0 127.0.0.1:10248  0.0.0.0:*  LISTEN  20613/kubelet
tcp   0  0 127.0.0.1:10251  0.0.0.0:*  LISTEN  19968/kube-schedule
tcp   0  0 127.0.0.1:10252  0.0.0.0:*  LISTEN  20815/kube-controll
tcp6  0  0 :::8080          :::*       LISTEN  20333/kube-apiserve
```

透過簡單指令驗證：
```sh
$ kubectl get node
NAME      STATUS         AGE
master1   Ready,master   1m

$ kubectl get po --all-namespaces
kube-system   kube-apiserver-master1            1/1       Running   0          12m
kube-system   kube-controller-manager-master1   1/1       Running   0          7m
kube-system   kube-scheduler-master1            1/1       Running   0          12
```

## Kubernetes Node 安裝與設定
Node 是主要的工作節點，上面將運行許多容器應用。到所有`node`節點加入取得 Packages 來源，並安裝：
```sh
$ curl -s "https://packages.cloud.google.com/apt/doc/apt-key.gpg" | apt-key add -
$ echo "deb http://apt.kubernetes.io/ kubernetes-xenial main" > /etc/apt/sources.list.d/kubernetes.list
$ apt-get update && apt-get install -y kubelet kubernetes-cni
```

然後準備 OpenSSL 的設定檔資訊：
```sh
$ mkdir -p /etc/kubernetes/pki
$ DIR=/etc/kubernetes/pki
$ cat <<EOF > ${DIR}/openssl.conf
[req]
req_extensions = v3_req
distinguished_name = req_distinguished_name
[req_distinguished_name]
[ v3_req ]
basicConstraints = CA:FALSE
keyUsage = nonRepudiation, digitalSignature, keyEncipherment
subjectAltName = @alt_names
[alt_names]
IP.1=172.16.35.10
DNS.1=node1
EOF
```
> P.S.這邊`IP.1`與`DNS.1`需要隨機器不同設定。細節請參考[Cluster TLS using OpenSSL](https://coreos.com/kubernetes/docs/latest/openssl.html)。

將`master1`上的 OpenSSL key 複製到`/etc/kubernetes/pki`：
```sh
for file in ca-key.pem ca.pem admin.pem admin-key.pem; do
  scp master1:/etc/kubernetes/pki/${file} /etc/kubernetes/pki/
done
```

建立 OpenSSL Keypairs 與 Certificate：
```sh
openssl genrsa -out ${DIR}/node-key.pem 2048
openssl req -new -key ${DIR}/node-key.pem -out ${DIR}/node.csr -subj '/CN=kube-node' -config ${DIR}/openssl.conf
openssl x509 -req -in ${DIR}/node.csr -CA ${DIR}/ca.pem -CAkey ${DIR}/ca-key.pem -CAcreateserial -out ${DIR}/node.pem -days 1000 -extensions v3_req -extfile ${DIR}/openssl.conf
```
> 細節請參考[Cluster TLS using OpenSSL](https://coreos.com/kubernetes/docs/latest/openssl.html)。


接著下載 Kubernetes 相關檔案至`/etc/kubernetes/`：
```sh
$ cd /etc/kubernetes/
$ URL="https://gist.githubusercontent.com/kairen/bd51f31d12cb7c2acc7f5e0a4b4fde49/raw/974d4d3a295658d92b1805558448fd38d61d76cc/"
$ wget ${URL}/kubelet.conf -O kubelet.conf
$ wget ${URL}/admin.conf -O admin.conf
$ wget ${URL}/kubelet -O kubelet
```
> 若`IP`與教學設定不同的話，請記得修改。

新增`/lib/systemd/system/kubelet.service`來管理 kubelet：
```sh
$ cat <<EOF > /lib/systemd/system/kubelet.service
[Unit]
Description=Kubernetes Kubelet Server
After=docker.service
Requires=docker.service

[Service]
WorkingDirectory=/var/lib/kubelet
EnvironmentFile=-/etc/kubernetes/kubelet
ExecStart=/usr/bin/kubelet \$KUBELET_ADDRESS \$KUBELET_POD_INFRA_CONTAINER \
\$KUBELET_ARGS \$KUBE_LOGTOSTDERR \
\$KUBE_ALLOW_PRIV \$KUBELET_NETWORK_ARGS \
\$KUBELET_DNS_ARGS
Restart=always

[Install]
WantedBy=multi-user.target
EOF
```

最後建立 var 存放資訊，然後啟動 kubelet 服務:
```sh
$ mkdir -p /var/lib/kubelet
$ systemctl daemon-reload
$ systemctl restart kubelet.service
```

當所有節點都完成後，回到`master`透過簡單指令驗證：
```sh
$ kubectl get node

NAME      STATUS         AGE
master1   Ready,master   10m
node1     Ready          4m
node2     Ready          1m
```

## Kubernetes Addons 部署
當環境都建置完成後，就可以進行部署附加元件，首先到`master1`，並進入`/etc/kubernetes/`目錄下載 Addons 檔案：
```sh
$ cd /etc/kubernetes/ && mkdir addons
$ URL="https://gist.githubusercontent.com/kairen/815886f6fcdebe48c798ee4c8f812f7b/raw/fe20323fa040a8d2cd12b94d909d4d4ff6fdfdb4/"
$ wget ${URL}/proxy.yaml -O addons/proxy.yaml
$ wget ${URL}/dns.yaml -O addons/dns.yaml
$ wget ${URL}/dashboard.yaml -O addons/dashboard.yaml
$ wget ${URL}/monitoring.yaml -O addons/monitoring.yaml
```
> 若`IP`與教學設定不同的話，請記得修改。

接著透過 kubectl 來指定檔案建立附加元件：
```sh
for file in proxy.yaml dns.yaml dashboard.yaml monitoring.yaml; do
  kubectl create -f addons/${file}
done
```
> 若想要刪除則將`create`改成`delete`即可。

透過以下指令來驗證部署是否有效：
```sh
$ kubectl get po --all-namespaces

NAMESPACE     NAME                                    READY     STATUS    RESTARTS   AGE
kube-system   heapster-v1.2.0-371757285-7qq02         1/1       Running   0          2m
kube-system   influxdb-grafana-704917236-zdx0m        2/2       Running   0          2m
kube-system   kube-dns-r5wx4                          3/3       Running   0          2m
kube-system   kube-proxy-amd64-2ck40                  1/1       Running   0          2m
kube-system   kube-proxy-amd64-lk0ql                  1/1       Running   0          2m
kube-system   kube-proxy-amd64-rxgxz                  1/1       Running   0          2m
kube-system   kubernetes-dashboard-3697905830-6nffv   1/1       Running   1          3m
```

確定都啟動後，可以開啟 [Dashboard](http://172.16.35.12/) 來查看。
![](/images/kube-dash-preview.png)

## 簡單部署 Nginx 服務
Kubernetes 可以選擇使用指令直接建立應用程式與服務，或者撰寫 YAML 與 JSON 檔案來描述部署應用程式的配置，以下將建立一個簡單的 Nginx 服務：
```sh
$ kubectl run nginx --image=nginx --replicas=1 --port=80
$ kubectl get pods -o wide

NAME                     READY     STATUS    RESTARTS   AGE       IP            NODE
nginx-4087004473-9c161   1/1       Running   0          18m       172.20.19.2   node1
```

完成後要接著建立 svc(Service)，來提供外部網路存取應用程式，使用以下指令建立：
```sh
$ kubectl expose deploy nginx --port=80 --type=LoadBalancer --external-ip=172.16.35.12
$ kubectl get svc

NAME             CLUSTER-IP       EXTERNAL-IP     PORT(S)        AGE
svc/kubernetes   192.160.0.1      <none>          443/TCP        2h
svc/nginx        192.160.57.181   ,172.16.35.12   80:32054/TCP   21s
```
> 這邊`type`可以選擇 NodePort 與 LoadBalancer。

確認沒問題後即可在瀏覽器存取 <EXTERNAL-IP>:32054。

### 擴展服務數量
若叢集`node`節點增加了，而想讓 Nginx 服務提供可靠性的話，可以透過以下方式來擴展服務的副本：
```sh
$ kubectl scale deploy nginx --replicas=2

NAME                     READY     STATUS    RESTARTS   AGE       IP            NODE
nginx-3449338310-9jsjx   1/1       Running   0          6m        172.20.19.2   node1
nginx-3449338310-rnfs9   1/1       Running   0          20s       172.20.98.5   node2
```