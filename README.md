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
      
