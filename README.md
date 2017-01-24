# Kubernetes-HA-Install
搭建CentOS 7.0 可用于生产环境的高可用ApiServer的Kubernetes集群  
HAProxy作为代理服务器实现HA  
CAlICO作为容器间网络实现
## 1.环境规划  
   本次kubernetes安装在4台机器上，其中两台作为ApiServer(含ContollerManager和Scheduler)，四台作为工作节点。  
   本次地址分配：  
   
	      10.1.245.224（主）  
	      10.1.245.225（主）  
	      10.1.245.8  
	      10.1.245.9  
	      10.1.245.8 kube-worker-1
	      10.1.245.9 kube-worker-2
              10.1.245.224 kube-worker-3
10.1.245.225 kube-worker-4
   本次安装操作系统为CentOS 7.1，Kubernetes 1.5.2（文档撰写时最新版），本次安装可能并不适合其他版本，请酌情参考。  
   本次 Kubernetes 安装未采用镜像模式或者kubeadm模式进行，而是采用了传统的 yum 加上二进制发行包模式。  
   在每个节点上执行 setenforce 0
## 2. 安装时间同步服务 NTP  
### 1) 在各台机器安装 NTP yum -y install ntp  
### 2) 配置 10.1.245.224 为 NTP 主服务器  修改 /etc/ntp.conf  

		     restrict default nomodify notrap nopeer noquery  
		     restrict default ignore  
		     restrict 10.1.245.0 mask 255.255.255.0  
		     server cn.pool.ntp.org prefer  
		     #server 10.1.245.224  
		     #fudge 10.1.245.224 stratum 8  

		     server 127.127.1.0   
		     fudge 127.127.1.0 stratum 8  
### 3) 启动服务： ntpd  
### 4) 其余节点向224同步 ：ntpdate 10.1.245.224，并在crontab 中增加：   
       00 */1 * * * root /usr/sbin/ntpdate 10.1.245.224;/sbin/hwclock -w  
       使用 crontab -l 查看
## 3. 安装ETCD 集群(非安全模式)  
     在各个节点安装 ETCD包  
     
     yum -y install etcd  
在各个节点配置ETCD（/etc/etcd/etcd.conf） 

     ETCD_LISTEN_PEER_URLS="http://0.0.0.0:2380"
     ETCD_LISTEN_CLIENT_URLS="http://0.0.0.0:2379"	
     ETCD_INITIAL_ADVERTISE_PEER_URLS="http://10.1.245.225:2380"
     ETCD_INITIAL_CLUSTER="kube-etcd-224=http://10.1.245.224:2380,kube-etcd-225=http://10.1.245.225:2380,kube-etcd-226=http://10.1.245.226:2380"
     ETCD_INITIAL_CLUSTER_STATE="existing"
     ETCD_INITIAL_CLUSTER_TOKEN="kube-etcd-cluster"
     
     ETCD_ADVERTISE_CLIENT_URLS="http://10.1.245.225:2379"
     
启动各个节点：systemctl enable etcd   
            systemctl start etcd  	
常用命令：
        
	etcdctl rm --recursive /registry
	etcdctl rm --recursive /calico
	etcdctl cluster-health
		member 6e5d6be96ebe4afa is healthy: got healthy result from http://10.1.245.225:2379
		member bdac6ad1e399cb3a is healthy: got healthy result from http://10.1.245.224:2379
		member eae8375054ea3a5a is healthy: got healthy result from http://10.1.245.226:2379
	etcdctl ls
	
	curl -L http://10.1.241.124:2379/v2/keys/mykey -XPUT -d value="this is awesome"
	curl http://10.1.241.124:2379/v2/keys/message -XPUT -d value="Hello world"
	curl http://10.1.241.124:2379/v2/keys/message 
	curl http://10.1.241.123:2379/v2/keys/message  	
检查各个节点的日志，确认etcd 集群工作正常

## 4.安装kubernetes
### 1）下载kubernetes发行包

	wget https://github.com/kubernetes/kubernetes/releases/download/v1.5.2/kubernetes.tar.gz
	tar zxvf kubernetes.tar.gz
	cd kubernetes
	cluster/get-kube-binaries.sh
	cd server
	tar zxvf kubernetes-manifests.tar.gz
 	tar zxvf kubernetes-salt.tar.gz 
	cd .. 
	cp ./server/kubernetes/server/bin/kube-apiserver /usr/bin/
 	cp ./server/kubernetes/server/bin/kube-controller-manager /usr/bin/
 	cp ./server/kubernetes/server/bin/kubectl /usr/bin/
 	cp ./server/kubernetes/server/bin/kube-discovery /usr/bin/
 	cp ./server/kubernetes/server/bin/kube-dns /usr/bin/
 	cp ./server/kubernetes/server/bin/kubefed /usr/bin/
 	cp ./server/kubernetes/server/bin/kubelet /usr/bin/
 	cp ./server/kubernetes/server/bin/kube-proxy /usr/bin/
 	cp ./server/kubernetes/server/bin/kube-scheduler /usr/bin/
使用 scp 命令将这些可执行文件分发到相应的节点目录下。

	scp 10.1.245.224:/usr/bin/kubectl /usr/bin/
	...
	
### 2) 准备各种证书  
#### 创建证书目录 ：mkdir -p /etc/kubernetes/pki  
#### 生成基本口令验证文件：echo 123456,admin,ots-pad>/etc/kubernetes/pki/tokens.csv  
     	口令文件为密码，用户名，用户id模式，在访问apiserver时需要在http头里进行相应的设置  
     	Authorization：Basic BASE64ENCODEDUSER:PASSWORD  
#### 生成根自签名证书

	openssl genrsa -out ca-key.pem 2048
	openssl req -x509 -new -nodes -key ca-key.pem -days 10000 -out ca.pem -subj "/CN=kube-ca"

#### 准备apiserver证书  
生成 openssl.cnf 

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
	DNS.5 = ${MASTER_DNS_NAME}
	IP.1 = ${K8S_SERVICE_IP}
	IP.2 = ${MASTER_HOST}
	IP.3 = ${MASTER_IP}
	IP.4 = ${MASTER_LOADBALANCER_IP}
	IP.5 = 192.168.0.1
替换相应的变量地址为实际地址  

	openssl genrsa -out apiserver-key.pem 2048
	openssl req -new -key apiserver-key.pem -out apiserver.csr -subj "/CN=kube-apiserver" -config openssl.cnf
	openssl x509 -req -in apiserver.csr -CA ca.pem -CAkey ca-key.pem -CAcreateserial -out apiserver.pem -days 365 -extensions v3_req -extfile openssl.cnf
#### 准备各个节点kubelet证书 
生成worker-openssl.cnf  

	[req]
	req_extensions = v3_req
	distinguished_name = req_distinguished_name
	[req_distinguished_name]
	[ v3_req ]
	basicConstraints = CA:FALSE
	keyUsage = nonRepudiation, digitalSignature, keyEncipherment
	subjectAltName = @alt_names
	[alt_names]
	IP.1 = $ENV::WORKER_IP
	
	export WORKER_IP=10.1.245.8
	export WORKER_FQDN=kube-work-1
	openssl genrsa -out ${WORKER_FQDN}-worker-key.pem 2048
	WORKER_IP=${WORKER_IP} openssl req -new -key ${WORKER_FQDN}-worker-key.pem -out ${WORKER_FQDN}-worker.csr -subj "/CN=${WORKER_FQDN}" -config worker-openssl.cnf
	WORKER_IP=${WORKER_IP} openssl x509 -req -in ${WORKER_FQDN}-worker.csr -CA ca.pem -CAkey ca-key.pem -CAcreateserial -out ${WORKER_FQDN}-worker.pem -days 365 -extensions v3_req -extfile worker-openssl.cnf
#### 生成管理秘钥对
	openssl genrsa -out admin-key.pem 2048
	openssl req -new -key admin-key.pem -out admin.csr -subj "/CN=kube-admin"
	openssl x509 -req -in admin.csr -CA ca.pem -CAkey ca-key.pem -CAcreateserial -out admin.pem -days 365
	
### 3）生成systemctl 服务
#### kube-apiserver.service

	[Unit]
	Description=Kubernetes API Server
	Documentation=https://github.com/GoogleCloudPlatform/kubernetes
	After=network.target
	After=etcd.service

	[Service]
	EnvironmentFile=-/etc/kubernetes/config
	EnvironmentFile=-/etc/kubernetes/apiserver
	ExecStart=/usr/bin/kube-apiserver \
		    $KUBE_LOGTOSTDERR \
		    $KUBE_LOG_LEVEL \
		    $KUBE_ETCD_SERVERS \
		    $KUBE_API_ADDRESS \
		    $KUBE_API_PORT \
		    $KUBELET_PORT \
		    $KUBE_ALLOW_PRIV \
		    $KUBE_SERVICE_ADDRESSES \
		    $KUBE_ADMISSION_CONTROL \
		    $KUBE_API_ARGS
	Restart=on-failure
	Type=notify
	LimitNOFILE=65536

	[Install]
	WantedBy=multi-user.target
#### kube-controller-manager.service
	[Unit]
	Description=Kubernetes Controller Manager
	Documentation=https://github.com/GoogleCloudPlatform/kubernetes

	[Service]
	EnvironmentFile=-/etc/kubernetes/config
	EnvironmentFile=-/etc/kubernetes/controller-manager
	ExecStart=/usr/bin/kube-controller-manager \
		    $KUBE_LOGTOSTDERR \
		    $KUBE_LOG_LEVEL \
		    $KUBE_MASTER \
		    $KUBE_CONTROLLER_MANAGER_ARGS
	Restart=on-failure
	LimitNOFILE=65536

	[Install]
	WantedBy=multi-user.target
#### kube-scheduler.service
	[Unit]
	Description=Kubernetes Scheduler Plugin
	Documentation=https://github.com/GoogleCloudPlatform/kubernetes

	[Service]
	EnvironmentFile=-/etc/kubernetes/config
	EnvironmentFile=-/etc/kubernetes/scheduler
	ExecStart=/usr/bin/kube-scheduler \
		    $KUBE_LOGTOSTDERR \
		    $KUBE_LOG_LEVEL \
		    $KUBE_MASTER \
		    $KUBE_SCHEDULER_ARGS
	Restart=on-failure
	LimitNOFILE=65536

	[Install]
	WantedBy=multi-user.target
#### kubelet.service
	[Unit]
	Description=Kubernetes Kubelet Server
	Documentation=https://github.com/GoogleCloudPlatform/kubernetes
	After=docker.service
	Requires=docker.service

	[Service]
	WorkingDirectory=/var/lib/kubelet
	EnvironmentFile=-/etc/kubernetes/config
	EnvironmentFile=-/etc/kubernetes/kubelet
	ExecStart=/usr/bin/kubelet \
		    $KUBE_LOGTOSTDERR \
		    $KUBE_LOG_LEVEL \
		    $KUBELET_API_SERVER \
		    $KUBELET_ADDRESS \
		    $KUBELET_PORT \
		    $KUBELET_HOSTNAME \
		    $KUBE_ALLOW_PRIV \
		    $KUBELET_POD_INFRA_CONTAINER \
		    $KUBELET_ARGS
	Restart=on-failure

	[Install]
	WantedBy=multi-user.target
#### kube-proxy.service
	[Unit]
	Description=Kubernetes Kube-Proxy Server
	Documentation=https://github.com/GoogleCloudPlatform/kubernetes
	After=network.target

	[Service]
	EnvironmentFile=-/etc/kubernetes/config
	EnvironmentFile=-/etc/kubernetes/proxy
	ExecStart=/usr/bin/kube-proxy \
		    $KUBE_LOGTOSTDERR \
		    $KUBE_LOG_LEVEL \
		    $KUBE_MASTER \
		    $KUBE_PROXY_ARGS
	Restart=on-failure
	LimitNOFILE=65536

	[Install]
	WantedBy=multi-user.target
##### 将生成好的文件分发给相应的节点  
生成通用配置文件 /etc/kubernetes/config

	###
	# kubernetes system config
	#
	# The following values are used to configure various aspects of all
	# kubernetes services, including
	#
	#   kube-apiserver.service
	#   kube-controller-manager.service
	#   kube-scheduler.service
	#   kubelet.service
	#   kube-proxy.service
	# logging to stderr means we get it in the systemd journal
	KUBE_LOGTOSTDERR="--logtostderr=true"

	# journal message level, 0 is debug
	KUBE_LOG_LEVEL="--v=0"

	# Should this cluster be allowed to run privileged docker containers
	KUBE_ALLOW_PRIV="--allow-privileged=true"

	# How the controller-manager, scheduler, and proxy find the apiserver
	KUBE_MASTER="--master=http://10.1.245.224:9090"
#### /etc/kubernetes/apiserver(注意：此处监听地址和其他使用的地方地址并不一致，是后来搭建HaProxy的地址，在Haproxy没有搭建成功时，需要使用9090和9443地址)
	###
	# kubernetes system config
	#
	# The following values are used to configure the kube-apiserver
	#

	# The address on the local server to listen to.
	KUBE_API_ADDRESS="--insecure-bind-address=0.0.0.0"

	# The port on the local server to listen on.
	KUBE_API_PORT="--insecure-port=8080"

	# Port minions listen on
	# KUBELET_PORT="--kubelet-port=10250"

	# Comma separated list of nodes in the etcd cluster
	KUBE_ETCD_SERVERS="--etcd-servers=http://10.1.245.224:2379,http://10.1.245.225:2379,http://10.1.245.226:2379"

	# Address range to use for services
	KUBE_SERVICE_ADDRESSES="--service-cluster-ip-range=192.168.0.0/16"

	# default admission control policies
	KUBE_ADMISSION_CONTROL="--admission-control=NamespaceLifecycle,NamespaceExists,LimitRanger,SecurityContextDeny,ServiceAccount,ResourceQuota"

	# Add your own!
	KUBE_API_ARGS=" --advertise-address=10.1.245.224 --bind-address=0.0.0.0 --secure-port=8443 --basic-auth-file=/etc/kubernetes/pki/tokens.csv --client-ca-file=/etc/kubernetes/pki/ca.pem --tls-cert-file=/etc/kubernetes/pki/apiserver.pem --tls-private-key-file=/etc/kubernetes/pki/apiserver-key.pem"
#### /etc/kubernetes/controller-manager
	###
	# The following values are used to configure the kubernetes controller-manager

	# defaults from config and apiserver should be adequate

	# Add your own!
	KUBE_CONTROLLER_MANAGER_ARGS="  --master=10.1.245.224:9090 \
	  --leader-elect=true \
	  --cluster-signing-cert-file=/etc/kubernetes/pki/ca.pem \
	  --cluster-signing-key-file=/etc/kubernetes/pki/ca-key.pem \
	  --root-ca-file=/etc/kubernetes/pki/ca.pem \
	  --service-account-private-key-file=/etc/kubernetes/pki/apiserver-key.pem \
	  --node-monitor-grace-period=10s --node-monitor-period=5s --pod-eviction-timeout=5m0s"
####  /etc/kubernetes/scheduler 
	###
	# kubernetes scheduler config

	# default config should be adequate

	# Add your own!
	KUBE_SCHEDULER_ARGS=" --leader-elect=true"
#### /etc/kubernetes/kubelet
	###
	# kubernetes kubelet (minion) config

	# The address for the info server to serve on (set to 0.0.0.0 or "" for all interfaces)
	KUBELET_ADDRESS="--address=0.0.0.0"

	# The port for the info server to serve on
	# KUBELET_PORT="--port=10250"

	# You may leave this blank to use the actual hostname
	KUBELET_HOSTNAME="--hostname-override=10.1.245.224"

	# location of the api-server
	KUBELET_API_SERVER="--api-servers=http://10.1.245.224:9090,http://10.1.245.225:9090"

	# pod infrastructure container
	KUBELET_POD_INFRA_CONTAINER="--pod-infra-container-image=registry.access.redhat.com/rhel7/pod-infrastructure:latest"

	# Add your own!
	KUBELET_ARGS=" --cluster_dns=192.168.1.1 --cluster_domain=cluster.local --network-plugin=cni --network-plugin-dir=/etc/cni/net.d --cni-bin-dir=/opt/cni/bin --register-node=true --allow-privileged=true --kubeconfig=/etc/kubernetes/worker-kubeconfig.yaml --tls-cert-file=/etc/kubernetes/ssl/worker.pem --tls-private-key-file=/etc/kubernetes/ssl/worker-key.pem --node-ip=10.1.245.224 --cluster-domain=cluster.local --image-gc-high-threshold=90 --image-gc-low-threshold=80"
#### /etc/kubernetes/proxy
	###
	# kubernetes proxy config

	# default config should be adequate

	# Add your own!
	KUBE_PROXY_ARGS=" --proxy-mode=iptables --hostname-override=10.1.245.224 --kubeconfig=/etc/kubernetes/worker-kubeconfig.yaml"
#### 将各个配置文件分发到对应节点，并进行相应的调整
##### 启动 kubernetes，在相应的节点启动相应的服务并查看服务状态的日志，看是否正常启动  
	systemctl daemon-reload
	systemctl enable kube-apiserver
	systemctl enable kube-controller-manager
	systemctl enable kube-scheduler
	systemctl enable kube-kubelet
	systemctl enable kube-proxy
	systemctl start kube-apiserver
	systemctl start kube-controller-manager
	systemctl start kube-scheduler
	systemctl start kube-kubelet
	systemctl start kube-proxy
	
	systemctl status kube-apiserver
	journalctl -xe
	journalctl -F
##### 验证api 服务是否正常
	curl http://127.0.0.1:8080 
	curl -k -u admin:123456 https://127.0.0.1:8443 	
#### 生成本地  kubeconfig
	kubectl config set-cluster local --insecure-skip-tls-verify=true --server=http://10.1.245.224:9090 --api-version=v1 
	kubectl config set-context default/local --user=admin --namespace=default --cluster=local
	kubectl config use-context default/local
#### 发布验证  		
	kubectl run nginx --replicas=2 --image=nginx
	kubectl get pod -o wide
	kubectl get service -o wide
	kubectl get pod -o wide --namespace=kube-system
#### 如果正常运行了容器则部署成功
## 5. 安装 Calico 网络  
	本次网络安装不采用容器模式，采用常规模式，以便调试容器间网络
### 1）下载 calicoctl
	wget https://github.com/projectcalico/calicoctl/releases/download/v0.23.1/calicoctl
	cp calicoctl /usr/bin
### 2) 创建calico-node 服务  
	/usr/lib/systemd/system/calico-node.service

	[Unit]
	Description=calicoctl node
	After=docker.service
	Requires=docker.service

	[Service]
	User=root
	Environment=ETCD_ENDPOINTS=http://10.1.245.224:2379,http://10.1.245.225:2379,http://10.1.245.226:237
	PermissionsStartOnly=true
	ExecStart=/usr/bin/docker run --net=host --privileged --name=calico-node \
	  -e ETCD_ENDPOINTS=${ETCD_ENDPOINTS} \
	  -e NODENAME=${HOSTNAME} \
	  -e IP= \
	  -e NO_DEFAULT_POOLS= \
	  -e AS= \
	  -e CALICO_LIBNETWORK_ENABLED=true \
	  -e IP6= \
	  -e CALICO_NETWORKING_BACKEND=bird \
	  -v /var/run/calico:/var/run/calico \
	  -v /lib/modules:/lib/modules \
	  -v /run/docker/plugins:/run/docker/plugins \
	  -v /var/run/docker.sock:/var/run/docker.sock \
	  -v /var/log/calico:/var/log/calico \
	  calico/node:latest
	ExecStop=/usr/bin/docker rm -f calico-node
	Restart=always
	RestartSec=10

	[Install]
	WantedBy=multi-user.target
### 3）将此服务分发到四个节点，并启动
      docker logs 
      docker exec -ti 
      查看是否正常启动
      export ETCD_ENDPOINTS=http://10.1.245.224:2379,http://10.1.245.225:2379,http://10.1.245.226:2379
      cailcoctl status
### 4) 创建kubernetes 监听 calico-controller
	calico-policy-controller.yml 
	
	# Create this manifest using kubectl to deploy
	# the Calico policy controller on Kubernetes.
	# It deploys a single instance of the policy controller.
	apiVersion: extensions/v1beta1
	kind: Deployment 
	metadata:
	  name: calico-policy-controller
	  namespace: kube-system
	  labels:
	    k8s-app: calico-policy
	spec:
	  # Only a single instance of the policy controller should be 
	  # active at a time.  Since this pod is run as a Deployment,
	  # Kubernetes will ensure the pod is recreated in case of failure,
	  # removing the need for passive backups.
	  replicas: 1
	  strategy:
	    type: Recreate
	  template:
	    metadata:
	      name: calico-policy-controller
	      namespace: kube-system
	      labels:
		k8s-app: calico-policy
	    spec:
	      hostNetwork: true
	      containers:
		- name: calico-policy-controller
		  # Make sure to pin this to your desired version.
		  image: calico/kube-policy-controller:latest
		  env:
		    # Configure the policy controller with the location of 
		    # your etcd cluster.
		    - name: ETCD_ENDPOINTS
		      value: "http://10.1.245.224:2379"
		    # Location of the Kubernetes API - this shouldn't need to be 
		    # changed so long as it is used in conjunction with 
		    # CONFIGURE_ETC_HOSTS="true".
		    - name: K8S_API
		      value: "http://10.1.245.224:9090"
		    # Configure /etc/hosts within the container to resolve 
		    # the kubernetes.default Service to the correct clusterIP
		    # using the environment provided by the kubelet.
		    # This removes the need for KubeDNS to resolve the Service.
		    - name: CONFIGURE_ETC_HOSTS
		      value: "true"
	kubectl -f calico-policy-controller.yml
### 5)验证网络服务
	calicoctl status 显示每个节点都已经联通
	calico-node container is running. Status: Up 3 days

	IPv4 BGP status
	IP: 10.1.245.224    AS Number: 64511 (inherited)
	+--------------+-------------------+-------+------------+-------------+
	| Peer address |     Peer type     | State |   Since    |     Info    |
	+--------------+-------------------+-------+------------+-------------+
	| 10.1.245.225 | node-to-node mesh |   up  | 2017-01-21 | Established |
	|  10.1.245.8  | node-to-node mesh |   up  | 2017-01-21 | Established |
	|  10.1.245.9  | node-to-node mesh |   up  | 2017-01-21 | Established |
	+--------------+-------------------+-------+------------+-------------+
	
	calicoctl profile show 
	+--------------------+
	|        Name        |
	+--------------------+
	|   k8s_ns.default   |
	| k8s_ns.kube-system |
	+--------------------+
	calicoctl profile k8s_ns.default rule show
	Inbound rules:
	   1 allow
	Outbound rules:
	   1 allow
	calicoctl pool show
	+----------------+-------------------+
	|   IPv4 CIDR    |      Options      |
	+----------------+-------------------+
	| 192.168.0.0/16 | ipip,nat-outgoing |
	+----------------+-------------------+
	+--------------------------+---------+
	|        IPv6 CIDR         | Options |
	+--------------------------+---------+
	| fd80:24e2:f998:72d6::/64 |         |
	+--------------------------+---------+
	
#### 如果以上信息显示不一致，则说明搭建不成功。如果pool没有显示 ipip等，需要删除pool，自己手动添加。
#### 可以使用ip route 命令查看并删除不需要的路由，使用iptables -L,iptable -F, iptables -X, iptable -Z查看或者清空防火墙规则
#### 按照上面的创建容器方法重新创建两个容器，在容器内互相ping 对方，如果可以ping通，则网络完成。如果不通，需要查看各个部分的日志
