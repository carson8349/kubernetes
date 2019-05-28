kubernetes


MASTER安装配置

1. 安装并配置Kubernetes master

    安装

        yum -y install etcd kubernetes

    配置etcd，编辑/etc/etcd/etcd.conf修改下面地方

        ETCD_NAME=default
        ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
        ETCD_LISTEN_CLIENT_URLS="http://0.0.0.0:2379"
        ETCD_ADVERTISE_CLIENT_URLS="http://localhost:2379"

    配置kubernetes，编辑/etc/kubernetes/apiserver修改下面地方

        KUBE_API_ADDRESS="--address=0.0.0.0"
        KUBE_API_PORT="--port=8080"
        KUBELET_PORT="--kubelet_port=10250"
        KUBE_ETCD_SERVERS="--etcd_servers=http://127.0.0.1:2379"
        KUBE_SERVICE_ADDRESSES="--service-cluster-ip-range=10.254.0.0/16"
        KUBE_ADMISSION_CONTROL="--admission_control=NamespaceLifecycle,NamespaceExists,LimitRanger,SecurityContextDeny,ResourceQuota"
        KUBE_API_ARGS=""

    启动服务

        for services in etcd kube-apiserver kube-controller-manager kube-scheduler; do 
            echo "start " $services " =========================================== "    
            systemctl restart $services
            systemctl enable $services    
            systemctl status $services
        done

    etcd设置项（后面会被运行在node上的flannel自动获取并用来设置docker的IP地址）

        etcdctl -C http://192.168.56.110:2379 set /atomic.io/network/config '{"Network":"10.1.0.0/16"}

    运行kubectl get nodes可以查看有多少node在运行，以及其状态，我们这里才刚刚开始安装没有任何资源。


2. 安装并配置Kubernetes node

    安装

        yum -y install docker-engine flannel kubernetes

    配置flannel，编辑/etc/sysconfig/flanneld 

        FLANNEL_ETCD="http://192.168.50.130:2379"

    配置kubernetes，编辑/etc/kubernetes/config 

        KUBE_MASTER="--master=http://192.168.50.130:8080"
        KUBE_ETCD_SERVERS="--etcd_servers=http://192.168.50.130:2379"

    配置kubernetes，编辑/etc/kubernetes/kubelet（请使用每台minion自己的IP地址比如：192.168.50.131代替下面的$LOCALIP） 

        KUBELET_ADDRESS="--address=0.0.0.0"
        KUBELET_PORT="--port=10250"
        # change the hostname to this host’s IP address
        KUBELET_HOSTNAME="--hostname_override=$LOCALIP"
        KUBELET_API_SERVER="--api_servers=http://192.168.50.130:8080"
        KUBELET_ARGS=""

    启动服务

        如果本来机器上已经运行过docker的用以下命令删除
            ip link delete docker0

        正式启动各节点服务

            for services in kube-proxy kubelet flanneld docker;do
                    echo "start " $services " =========================================== "
                    systemctl restart $services
                    systemctl enable $services
                    systemctl status $services
            done

3. 配置完成验证安装
确定两台node和一台master都已经成功的安装配置并且服务都已经启动了，切换到master机器上，运行命令kubectl get nodes  
 
    kubectl get nodes

4. 安装

在master创建RC/Pod/Server

kubectl create -f redis-master-Rcontroller.yml
kubectl create -f redis-master-service.yml
kubectl create -f redis-slave-Rcontroller.yml
kubectl create -f redis-slave-service.yml
kubectl create -f frontend-controller.yml


修改副本命令

kubectl scale rc redis-master  --replicas=1

