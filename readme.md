PVC=PersistentVolumeClaim（持久化卷声明）
# 大白话说明白K8S的PV / PVC / StorageClass
参考：https://zhuanlan.zhihu.com/p/655923057
## 理论
PV是对存储资源的抽象，由运维人员创建，供容器申请使用。
PVC是POD对存储资源的一个申请（包括存储空间，访问模式等）。
创建PV后，POD可以通过PVC向PV申请磁盘空间，类似应用向OS的D盘申请空间。
PV相当于磁盘分区，PVC相当于应用所需空间，它们之间建立绑定关系后，就可以使用了。
StorageClass会动态绑定PV和PVC，从而实现存储卷的按需创建。Provisioner是存储驱动。
StorageClass资源对象的主要包括：名称、Provisioner、存储的相关参数配置、回收策略。
PV生命周期：供应（Provisioning）、绑定（Binding）、使用（Using）、回收（Reclaiming）。
有了PVC、PV之后，所有Pod使用存储资源，保持一个原则：先规划 → 后申请 → 再使用。
## 实战
### Pod使用PV和PVC挂载存储卷
新建文件：pv.yaml
kubectl apply -f pv.yaml
kubectl get pv
新建文件：pvc.yaml
kubectl apply -f pvc.yaml
kubectl get pvc -n dev1
新建文件：pod.yaml
kubectl apply -f pod.yaml
kubectl get po -n dev1 -o wide
kubectl describe pod webapp -n dev1
mkdir -p /root/data/nfstest
vim /etc/exports，/root/data/nfstest 192.168.112.0/24(rw,no_root_squash)
systemctl restart nfs-server
showmount -e localhost
kubectl delete po webapp -n dev
kubectl apply -f pod.yaml
http://10.0.5.99:8081
新建文件：index.html
cd /root/data/nfstest/share/pv1/
vim index.html
http://10.0.5.99:8081
### Pod使用StorageClass自动挂载存储卷
sc=StorageClass不属于某个命名空间，它跟PV一样，是全局共享的。
#### 安装helm
下载：wget https://get.helm.sh/helm-v3.13.1-linux-amd64.tar.gz
解压：tar -zxvf helm-v3.13.1-linux-amd64.tar.gz
移动：mv linux-amd64/helm /usr/local/bin/helm
测试：helm version
#### 安装provisioner
创建目录：mkdir /root/data/nfstest/nfs-storage
安装（超时）：helm install nfs-subdir-external-provisioner nfs-subdir-external-provisioner/nfs-subdir-external-provisioner -n kube-system --set image.repository=dyrnq/nfs-subdir-external-provisioner --set nfs.server=192.168.112.173 --set nfs.path=/root/data/nfstest/nfs-storage
下载：curl -LO https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner/releases/download/nfs-subdir-external-provisioner-4.0.18/nfs-subdir-external-provisioner-4.0.18.tgz
安装：helm install nfs-subdir-external-provisioner ./nfs-subdir-external-provisioner-4.0.18.tgz -n kube-system --set image.repository=dyrnq/nfs-subdir-external-provisioner --set nfs.server=192.168.112.173 --set nfs.path=/root/data/nfstest/nfs-storage
查看POD：kubectl get pod -n kube-system -o wide | grep prov
nfs-subdir-external-provisioner-75d894d56c-ck79q   1/1     Running   0          117s    10.244.1.19   computer     <none>           <none>
查看结果：helm list -A
#### 使用StorageClass
新建文件：sc.yaml
kubectl apply -f sc.yaml
kubectl get sc -n dev1
NAME PROVISIONER RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
nfs-storage-1   cluster.local/nfs-subdir-external-provisioner   Delete          Immediate           false                  11s
新建文件：sc-pvc.yaml
kubectl apply -f sc-pvc.yaml
kubectl get pv,pvc -n dev1 | grep nfs-storage
persistentvolume/pvc-ea59fb5a-5d9f-47b5-a13c-5b2f5542cbcb 10Mi RWO Delete Bound dev1/nfs-storage-pvc-1 nfs-storage-1 28s
persistentvolumeclaim/nfs-storage-pvc-1 Bound pvc-ea59fb5a-5d9f-47b5-a13c-5b2f5542cbcb 10Mi RWO nfs-storage-1   28s
新建文件：sc-pod.yaml
kubectl apply -f sc-pod.yaml
kubectl get po -n dev1
NAME READY STATUS RESTARTS   AGE
nfs-storage-pod-1   0/1 Completed   0 4m5s
cat teststorage
111
一句话总结：
PV、PVC是K8S用来做存储管理的资源对象，它们让存储资源的使用变得可控，从而保障系统的稳定性、可靠性。
StorageClass则是为了减少人工的工作量而去自动化创建PV的组件。
所有Pod使用存储只有一个原则：先规划 → 后申请 → 再使用。

# 【K8s】基本存储、高级存储（PV和PVC）、配置存储
参考：https://blog.csdn.net/llg___/article/details/130674262
容器可能会被销毁，卷volume可作为持久化存储，作为共享目录，被多个POD使用。
卷类型：基本（EmptyDir，HostPath，NFS），高级（PV和PVC），配置（ConfigMap和Secret）。
## 基本存储（EmptyDir，HostPath，NFS）
### EmptyDir
适用场景：临时空间，共享目录。
创建一个volume-emptydir.yaml。
配置好两个容器目录和空目录挂载的关系，
创建POD：kubectl create -f volume-emptydir.yaml
查看POD：kubectl get pods volume-emptydir -n dev -o wide
观察日志：kubectl logs -f volume-emptydir -n dev -c busybox
生成日志：curl volume-ip:80
### HostPath
EmptyDir中数据会随着pod的销毁而丢失。想将数据持久化到主机中，可以选择HostPath。
新建文件：volume-hostpath.yaml
创建POD：kubectl create -f volume-hostpath.yaml
查看POD：kubectl get pods volume-hostpath -n dev -o wide
观察日志（在POD所在在计算节点）：ls /root/logs/
生成日志：curl volume-ip:80
### NFS
HostPath持久化的目录仅在pod所在的那个节点。NFS网络文件系统，POD漂移后，仍然可以访问。
安装NFS服务（控制节点）：yum install nfs-utils -y
创建共享目录（控制节点）：mkdir /root/data/nfs -pv
网段可读写（vim /etc/exports）：/root/data/nfs     192.168.112.0/24(rw,no_root_squash)
启动nfs服务：systemctl start nfs-server
查看nfs共享的目录（工作节点）：showmount -e 192.168.112.173
挂载测试（工作节点）：mount -t nfs -o vers=3 192.168.112.173:/root/data/nfs /mnt/nfs
新建文件：volume-nfs.yaml
创建POD：kubectl create -f volume-nfs.yaml
查看POD：kubectl get pods volume-nfs -n dev -o wide
查看日志：ls /root/data/nfs
生成日志：curl volume-ip:80
基本存储的三种方式中：EmptyDir的数据和Pod挂钩，HostPath的数据和某个node节点挂钩，NFS则相对较完善可行。
## 高级存储（PV和PVC）
使用NFS基本实现了pod数据持久化和容器数据共享，但得搭建NFS或者其他k8s所支持的文件存储系统。
为了屏蔽底层存储的实现细节，方便掌握，pv和pvc出现。
PV（Persistent Volume）是持久化卷的意思，是对底层的共享存储的一种抽象。
一般情况下PV由k8s管理员进行创建和配置，它与底层具体的共享存储技术有关，并通过插件完成与共享存储的对接。
PVC（Persistent Volume Claim）是持久卷声明的意思，是用户对于存储需求的一种声明。
换句话说，PVC其实就是用户向k8s系统发出的一种资源需求申请。

### PV持久化卷
PV元数据中，没有NameSpace，因为PV是集群级别的资源，可以跨NameSpace使用。
PV的关键配置参数：存储类型（NFS），存储能力（storage:2Gi），访问模式（只读，读写），回收策略（保留，删除），存储类别。
PV的生命周期4阶段：Available（未跟PVC绑定），Bound（已绑定），Released（PVC被删除），Failed（自动回收失败）。
创建目录（3个PV）：mkdir /root/data/{pv1,pv2,pv3} -pv
暴露资源（vim /etc/exports）：/root/data/pv1     192.168.112.0/24(rw,no_root_squash)
重启NFS服务：systemctl restart nfs-server
查看NFS目录：showmount -e localhost
新建文件：volume-pv.yaml
创建PV资源：kubectl create -f volume-pv.yaml
查看PV信息：kubectl get pv -o wide
### PVC持久化卷的声明
PVC是有命名空间的，跟PV不是同一个级别的。
PVC声明了访问模式，存储类别，请求空间等，据此，可以找到合适的PV与之绑定。
新建文件：volume-pvc.yaml
创建PVC资源（最后一个会绑定失败）：kubectl create -f volume-pvc.yaml
查看PVC信息（最后一个一直是Pending）：kubectl get pvc -o wide -n dev
查看PV信息（最后一个是可用状态）：kubectl get pv
新建文件：volume-pod.yaml
创建POD：kubectl create -f volume-pod.yaml
查看POD：kubectl get pods -n dev -o wide
查看NFS：tail -f /root/data/pv1/out.txt
删除POD：kubectl delete -f volume-pod.yaml
删除PVC：kubectl delete -f volume-pvc.yaml
查看PV信息（状态是Released）：kubectl get pv
### PV和PVC的生命周期
PVC和PV是一一对应的，二者相互联系，其生命周期的阶段有：
1、资源供应。管理员手工创建底层的存储和PV。
2、资源绑定。用户创建PVC，k8s负责根据PVC的声明去查找PV并绑定。
3、资源使用。用户可以在POD中，像使用volume意义，使用PVC。
4、资源释放。用户删除PVC来释放PV。数据有保留，清除后才能再次使用。
5、资源回收。k8s根据PV设置的回收策略，进行资源的回收。

## 配置（ConfigMap和Secret）
### ConfigMap配置映射
cm=ConfigMap，它是一种比较特殊的存储卷，其作用是用来存储配置信息的。
新建文件：cm.yaml
创建CM：kubectl create -f cm.yaml
查看CM详情：kubectl describe cm cm1 -n dev
新建文件：cm-pod.yaml
创建POD：kubectl create -f cm-pod.yaml
查看POD：kubectl get po -n dev
进入POD：kubectl exec -it pod-cm -n dev bash
查看配置文件内容：cat /cm/config/info
可以看到，映射已经成功。每个cm都映射成一个目录，key（info）是文件，value（xxx）是内容。
## Secret秘密
Secret和ConfigMap相似，但它不是明文存储。主要用于存储敏感信息，例如密码、秘钥、证书等等。
Base64对admin编码（返回：YWRtaW4=）：echo -n '123456' | base64
Base64对123456编码（返回：MTIzNDU2）：echo -n '123456' | base64
新建文件：secret.yaml
创建秘密：kubectl create -f secret.yaml
查看秘密：kubectl get secret -n dev
新建文件：secret-pod.yaml
创建POD：kubectl create -f secret-pod.yaml
查看POD：kubectl get po -n dev
进入POD：kubectl exec -it pod-secret -n dev bash
查看配置文件内容：cat /secret/config/password
可以看到，这些信息，已经自动被解码了。
这些加密的信息，存储在k8s中，而不是某个pod中，这有助于提高安全性。

# 在新建k8s集群中创建sc
Provisioner粮食供应商，Harvester收割机（rancher），Longhorn长角牛（开源）。
mount -t nfs -o vers=4 10.0.1.220:/data/k8s-nfs-storage-new /mnt/nfs
参考：https://blog.csdn.net/qq_30051761/article/details/131055705
新建文件：nfs-rbac.yaml
kubectl apply -f nfs-rbac.yaml
kubectl get sa -A | grep nfs
kubectl get Roles -A | grep nfs
kubectl get RoleBindings -A | grep nfs
kubectl get ClusterRoles | grep nfs
kubectl get ClusterRoleBindings | grep nfs
新建文件：nfs-deploy.yaml
kubectl apply -f nfs-deploy.yaml
kubectl get deploy -A | grep nfs
kubectl get pod -A | grep nfs
kubectl describe deploy/nfs-client-provisioner -n kube-system
kubectl delete deploy/nfs-client-provisioner -n kube-system
新建文件：nfs-sc.yaml
kubectl apply -f nfs-sc.yaml
kubectl get sc
新建文件：nfs-pvc.yaml
kubectl apply -f nfs-pvc.yaml
kubectl get pvc
kubectl get pv
ll /mnt/nfs/
kubectl delete -f nfs-pvc.yaml