+++
date = '2026-06-08T09:27:45+08:00'
draft = false
title = 'Deploy'

+++

# 搭建`k8s`集群

## 搭建步骤

### 先决条件

三台`linux`虚拟机，可以相互通信

#### 设置主机名

`master`节点

~~~bash
hostnamectl set-hostname k8s-master
~~~

`node`节点

~~~bash
hostnamectl set-hostname k8s-node1
hostnamectl set-hostname k8s-node2
~~~

#### 配置`hosts`映射

打开文件

~~~bash
vim /etc/hosts
~~~

添加内容

~~~txt
192.168.111.129 k8s-master
192.168.111.131 k8s-node1
192.168.111.132 k8s-mode2
~~~

### 安装容器进行时

#### `CRI-O`

安装`CRI-O`的步骤是参考k8s文档的

[以独立模式运行 kubelet | Kubernetes](https://kubernetes.io/zh-cn/docs/tutorials/cluster-management/kubelet-standalone/#container-runtime)

下载静态二进制包脚本

~~~bash
curl https://raw.githubusercontent.com/cri-o/packaging/main/get > crio-install
~~~

运行安装脚本

~~~bash
sudo bash crio-install
~~~

启用并启动`crio`服务

~~~bash
sudo systemctl daemon-reload
sudo systemctl enable --now crio.service
~~~

快速测试

~~~bash
sudo systemctl is-active crio.service
~~~

成功应该输出`active`

### 安装`k8s`核心组件

需要安装`kubeadm`、`kubelet`、`kubectl`三个核心组件，安装步骤是参考`k8s`官方文档的，安装的是`1.36`版本

[安装 kubeadm | Kubernetes](https://kubernetes.io/zh-cn/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#installing-kubeadm-kubelet-and-kubectl)

更新`apt`包索引并安装使用`Kubernets apt`仓库所需要的包

~~~bash
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gpg
~~~

下载用于`Kubernetes`软件包仓库的公共签名密钥

~~~bash
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.36/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
~~~

更新`apt`包索引，安装`kubelet`、`kubeadm`和`kubectl`，并锁定其版本

~~~bash
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
~~~

### 初始化`master`节点

#### 集群初始化

~~~bash
kubeadm init
~~~

成功则输出`kubeadm join`命令，需要复制保存，`node`节点加入集群需要执行

~~~bash
kubeadm join 192.168.111.129:6443 --token x2fv6f.q8ulhapb4i4ux3kq \
        --discovery-token-ca-cert-hash sha256:ff54644f7b50af8a6d3279ede9a9974a16ac120b21fa0658a9b8ac69e1568df7 
~~~

#### 配置`kubectl`

创建`kubectl`配置目录并复制集群配置文件

~~~bash
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
~~~

配置权限并验证是否能看到`master`节点信息

~~~bash
chown $(id -u):$(id -g) $HOME/.kube/config
kubectl get nodes
~~~

#### 安装网络插件

下载并应用`Calico`配置，使用`3.32`最新版时有报错，这里使用`3.31.5`的版本

~~~bash
kubectl apply -f https://github.com/projectcalico/calico/raw/refs/tags/v3.31.5/manifests/calico.yaml
~~~

查看`Calico`组件状态

~~~bash
kubectl get pods -n kube-system
~~~

查看节点状态

~~~bash
kubectl get nodes
~~~

`master`节点状态变为`ready`时即成功

### `node`节点加入集群

执行`kubeadm init`输出的`kubeadm join`命令

~~~bash
kubeadm join 192.168.111.129:6443 --token x2fv6f.q8ulhapb4i4ux3kq \
        --discovery-token-ca-cert-hash sha256:ff54644f7b50af8a6d3279ede9a9974a16ac120b21fa0658a9b8ac69e1568df7 
~~~

在`master`节点中验证结果

~~~bash
kubectl get nodes
~~~

### 部署`Nginx`应用验证

创建`Nginx`部署并暴露端口

~~~bash
kubectl create deployment nginx --image=nginx:1.21
kubectl expose deployment nginx --port=80 --type=NodePort
~~~

查看部署状态并查看外部访问端口

~~~bash
kubectl get pods
kubectl get svc nginx
~~~

浏览器输入`http://192.168.111.129:31121`，即`node节点IP:外部访问端口`，能够看到`Nginx`默认页面，说明集群完全正常

## 关机后重新启动的步骤

启动`kubectl`服务

~~~bash
systemctl start kubelet
~~~

检查各项服务

~~~bash
kubectl get nodes
kubectl get pods
crictl ps -a
~~~

获取已运行的服务

~~~bash
kubectl get svc
~~~

根据输出访问`nginx`服务，看到`nginx`默认页面，重新启动成功

## 附加功能

### 部署仪表板

安装`helm`

~~~bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
~~~

部署`Kubernetes Dashboard`

~~~bash
helm repo add headlamp https://kubernetes-sigs.github.io/headlamp/
helm install my-headlamp headlamp/headlamp --namespace kube-system
~~~

获取服务`url`同时让宿主机能够访问

~~~bash
export POD_NAME=$(kubectl get pods --namespace kube-system -l "app.kubernetes.io/name=headlamp,app.kubernetes.io/instance=my-headlamp" -o jsonpath="{.items[0].metadata.name}")
export CONTAINER_PORT=$(kubectl get pod --namespace kube-system $POD_NAME -o jsonpath="{.spec.containers[0].ports[0].containerPort}")
kubectl --namespace kube-system port-forward --address 0.0.0.0 $POD_NAME 8080:$CONTAINER_PORT
~~~

浏览器打开`http://192.168.111.129:8080`，即服务器`ip`加`8080`端口，会看到需要填入`token`，执行命令创建`token`，默认有效期一个小时

`--duration 1h`修改有效期

~~~bash
kubectl create token my-headlamp --namespace kube-system
~~~

获取输出并在浏览器填入

**`url`可能会失效**：当这个`pod`运行结束时，`url`就会失效，此时`k8s`会重新运行一个`pod`，但是需要时间，可以查询`pod`是否运行起来了，运行起来后再转发端口

~~~bash
kubectl get pods --namespace kube-system
~~~

~~~bash
kubectl --namespace kube-system port-forward --address 0.0.0.0 svc/my-headlamp 8080:80
~~~

### 安装存储类

安装`local-path-provisioner`

~~~bash
kubectl apply -f https://github.com/rancher/local-path-provisioner/raw/refs/heads/master/deploy/local-path-storage.yaml
~~~

设置为默认存储类

~~~bash
kubectl patch storageclass local-path -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
~~~

验证安装

~~~bash
kubectl get storageclass
~~~

## 多服务部署

### 配置文件

编写每个服务的配置文件



































