# k8s集群快速搭建（官方推荐kubeadm）
### 集群基本介绍
- Debian11+Virtualbox
- Master Node 10.4.7.11
- Worker Node 10.4.7.11/.10.4.7.12/10.4.7.13
- kubernetes verison: 1.24.1

### 准备工作
1. 安装系统工具
    ```properties
    apt install -y gnupg2 lsb_release
    apt install -y apt-transport-https ca-certificates curl
    ```
2. 禁用交换分区（所有节点）
    - 因为在kube进行资源调度时，如果有些节点有swap,有些没有，则会认为有swap还有富余资源，从而将负载落入其中所以使用kubeadm进行k8s集群部署时，强制要求关闭swap.
    - swapoff -a 	# 临时关闭
    - vim /etc/fstab	# 重启有效，注释掉swap相关
3. 关闭防火墙，关闭selinux，因为在debian系统中默认没法有安装，这里不做说明
4. 安装docker（所有节点）
    -  注入源
        ```properties
        curl -fsSL https://download.docker.com/linux/debian/gpg | gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
        
        echo "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/debian $(lsb_release -cs) stable" | tee /etc/apt/sources.list.d/docker.list > /dev/null
        ```
    -  安装
        ```properties
        apt update
        apt install docker-ce
        ```
    - 配置,(国内源，cgroupdirver -> systemd,与k8s一致)
        ```properties
        vim /etc/docker/daemon.json
        ```
        ```json
        {
            "exec-opts": ["native.cgroupdriver=systemd"],
            "registry-mirrors": ["https://docker.mirrors.ustc.edu.cn"]
        }    
        ```
    - 重启
        ```properties
        systemctl daemon-reload
        systemctl restart docker
        ```
5. 网络（所有节点）
    - vim /etc/sysctl.d/k8s.conf
        ```properties
        net.ipv4.ip_forward = 1
        net.bridge.bridge-nf-call-ip6tables = 1
        net.bridge.bridge-nf-call-iptables = 1
        ```
    - sysctl --system
    - vim /etc/hosts (此处最好安装一个DNS服务器bind)
        ```properties
        10.4.7.11    debian-11.demo.com
        10.4.7.12    debian-12.demo.com
        10.4.7.13    debian-13.demo.com
        ```
6. 安装cri-dockerd（所有节点，非常重要）
    - k8s从1.24.0起，去除了docker-shim直接接入docker运行时方案，直接采用cri-dockerd
    - cri-dockerd将支持多种容器接口
    - 了解docker contained cri oci runc，请参考[here](https://www.xujun.org/note-147296.html)
    - cri-dockerd安装方法，请参考[here](https://github.com/Mirantis/cri-dockerd) 
    - cri-docker二进制文件可以直接从github下载，不用编绎安装，选择相应的系统版本，放入/usr/bin即可
    - 其中cri-docker.service启动脚本带上以下参数:
    ```properties
    ExecStart=/usr/bin/cri-dockerd --network-plugin=cni --pod-infra-container-image=registry.aliyuncs.com/google_containers/pause:3.7
    ```
    - 注意：pause镜像的版本请参照[这里](#jump1)，可以先按上面的版本，后续再做修改后，重启服务
### 安装kubectl kubeadm kubelet
1. 注入源（所有节点）
    ```properties
    curl -fsSL https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg.asc | gpg --dearmor -o /usr/share/keyrings/kubernetes-keyring.gpg

    echo "deb [arch=amd64 signed-by=/usr/share/keyrings/kubernetes-keyring.gpg] https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main" | tee /etc/apt/sources.list.d/kubernetes.list > /dev/null
    ``` 
2. 安装（所有节点）
    ```properties
    apt update
    apt install kubeadm kubectl kubelet
    ```
3. 启动kubelet(此时kubelet启动会失败，默认autostart,失败没关系，待后续组件自动安装完后会恢复正常)
    ```properties
    systemctl enable kubelet
    systemctl start kubelet
    ```
### 启动control-plane node(Master node)
1. <span id="jump1">拉取必要的镜像，因为kubernetes必要镜像在国外(Master节点操作即可)</span>
    ```properties
    vim image-pull.sh
    ``` 
    ```bash
    #!/bin/bash
    for i in `kubeadm config images list`; do 
        imageName=${i#k8s.gcr.io/}
        docker pull registry.aliyuncs.com/google_containers/$imageName
        docker tag registry.aliyuncs.com/google_containers/$imageName k8s.gcr.io/$imageName
        docker rmi registry.aliyuncs.com/google_containers/$imageName
    done;
    ```
    ```properties
    chmod +x imgae-pull.sh

    ./image-pull.sh
    ```
    - **注意：coredns镜像，可能是kubeadm的bug,请修改名称后手动拉取**
2. 初始化control-plane节点(Master节点)
    - 执行下面的命令，进行Master节点的初始化创建
    ```properties
    kubeadm init \
        --apiserver-advertise-address=10.4.7.11 \
        --image-repository registry.aliyuncs.com/google_containers \
        --kubernetes-version v1.24.1 \
        --service-cidr=10.96.0.0/12 \
        --pod-network-cidr=10.244.0.0/16 \
        --cri-socket=unix:///var/run/cri-dockerd.sock
    ```
    - 执行以下命令，进行Master节点的最后设置
    ```properties
    mkdir -p $HOME/.kube

    cp -i /etc/kubernetes/admin.conf $HOME/.kube/config

    chown $(id -u):$(id -g) $HOME/.kube/config
    ```
    - 添加网络组件（处理容器间的通讯问题，[参考官方文档](https://kubernetes.io/docs/concepts/cluster-administration/addons/)）
    ```properties
    kubectl apply -f https://docs.projectcalico.org/v3.23/manifests/calico.yaml
    ```
    - 查看节点状态
    ```properties
    kubectl get nodes     
    ```
### 启动woker node
1. 加入集群，(所有待加入的woker节点)
    - 执行以下相似命令
    - IP,PORT,Token请以自己的为准
    - 在Master节点创建成功时，会有类似以下的提示信息
    ```properties
    kubeadm join 10.4.7.11:6443 --token 6uaw9t.ux7gzbvdbslpn9sx --discovery-token-ca-cert-hash sha256:69e8fff3eb2a8b63106a210201505457b8ca618f4067c05bd7df4194a67b78f0 --cri-socket=unix:///var/run/cri-dockerd.sock
    ```
2. 验证
    - 在Master节点执行以下命令，查看所有节点的状态
    ```properties
    kubectl get nodes
    ```
