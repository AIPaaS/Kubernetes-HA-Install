# zookeeper 3节点集群在k8s上的部署  
下面的搭建模式为利用多个服务来组建zk集群，目前kubernetes的有状态服务还没有完全发布。  
前提是DNS已经配置好  
需要先执行服务的创建，然后再创建容器  
		kubectl create -f zk-svc.yaml
		kubectl create -f zk-rc.yaml
