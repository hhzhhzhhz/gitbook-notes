# Minikube - Kubernetes本地实验环境

进入etcd

```
kubectl exec -it --namespace kube-system etcd-minikube sh
```

安装minikube，缺省驱动

```
minikube start --cpus=4 --memory=4096mb
```

安装minikube，Docker驱动

```
minikube start --driver=docker
```

安装minikube，KVM2驱动

```
minikube start --driver=kvm2
```

打开Kubernetes控制台

```
minikube dashboard
```

![image](https://yqfile.alicdn.com/45690620c348d7be4a804880e3b7046f19e74c29.png))

对于使用Hyper-V环境的用户，首先应该打开Hyper-V管理器创建一个外部虚拟交换机，

![create](https://yqfile.alicdn.com/d165308ee88baf4adbe46c09b6d2596dea7bdfef.png)

![hyper_v](https://yqfile.alicdn.com/208a65dae18028cab8e9782803c7784ad110e0a6.png)

之后，我们可以用如下命令来创建基于Hyper-V的Kubernetes测试环境

```
.\minikube.exe start --image-mirror-country cn \
    --iso-url=https://kubernetes.oss-cn-hangzhou.aliyuncs.com/minikube/iso/minikube-v1.5.0.iso \
    --registry-mirror=https://xxxxxx.mirror.aliyuncs.com \
    --vm-driver="hyperv" \
    --hyperv-virtual-switch="MinikubeSwitch" \
    --memory=4096 
```

注：需要管理员权限来创建Hyper-V虚拟机

## 使用Minikube

Minikube利用本地虚拟机环境部署Kubernetes，其基本架构如下图所示。
![4](https://yqfile.alicdn.com/c03a43e0731ca579d1844fb44269fd2fd257bfb3.jpeg)

用户使用Minikube CLI管理虚拟机上的Kubernetes环境，比如：启动，停止，删除，获取状态等。一旦Minikube虚拟机启动，用户就可以使用熟悉的Kubectl CLI在Kubernetes集群上执行操作。

#### 参考资料

```
https://minikube.sigs.k8s.io/docs/start/
https://developer.aliyun.com/article/221687
https://kubernetes.io/zh/docs/tutorials/kubernetes-basics/
https://github.com/kubernetes/minikube
```

