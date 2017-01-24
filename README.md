# Kubernetes-HA-Install
搭建CentOS 7.0 可用于生产环境的高可用ApiServer的Kubernetes集群

## 1.环境规划  
   本次kubernetes安装在4台机器上，其中两台作为ApiServer(含ContollerManager和Scheduler)，四台作为工作节点。  
   本次地址分配：  
   
		     10.1.245.224（主）  
		     10.1.245.225（主）  
		     10.1.245.8  
		     10.1.245.9  
   本次安装操作系统为CentOS 7.1，Kubernetes 1.5.2（文档撰写时最新版），本次安装可能并不适合其他版本，请酌情参考。  
   本次 Kubernetes 安装未采用镜像模式或者kubeadm模式进行，而是采用了传统的 yum 加上二进制发行包模式。  
   
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

## 3.安装kubernetes
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
创建证书目录 ：mkdir -p /etc/kubernetes/pki

### 2) 准备各种证书	
### 2) 准备各种证书	
