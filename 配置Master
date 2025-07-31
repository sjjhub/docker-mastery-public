# Master  节点初始化配置


* **Master节点初始化之前先安装kubeadm并启动kubelet**
```
# 1. 安装kubeadm（使用root用户）
su -
yum install -y kubeadm kubelet kubectl

注意：系统无法连接 CentOS 官方镜像源，导致无法安装 Kubernetes 组件

解决方案：

# 备份原有 repo 文件
mkdir -p /etc/yum.repos.d/backup
mv /etc/yum.repos.d/CentOS-* /etc/yum.repos.d/backup/

# 下载阿里云镜像源
curl -o /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo

# 清理并重建缓存
yum clean all
yum makecache

# 测试 yum 是否正常工作
yum install -y wget

# 添加 Kubernetes 阿里云镜像源
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF

# 安装必要组件
yum install -y kubelet-1.23.8 kubeadm-1.23.8 kubectl-1.23.8 --disableexcludes=kubernetes


# 2. 验证安装
which kubeadm 
# 应该返回/usr/bin/kubeadm
kubeadm version

# 3. 启动kubelet
systemctl enable --now kubelet
```

* **用户权限问题**

将admin用户添加到sudoers文件
```
# 使用root用户登录后执行：
su -
visudo
# 在文件中添加以下内容：
admin   ALL=(ALL)       ALL
# 保存退出(:wq)
```
确保admin用户在docker组中（如果使用docker作为运行时）
```
usermod -aG docker admin
```
确保使用root用户
```
su -
```
* **清理环境（如果之前尝试过）**
```
kubeadm reset -f
rm -rf /etc/kubernetes/
rm -rf ~/.kube/
```
*  **安装docker容器**

 1、首先创建docker 目录‘
 ```
mkdir -p /etc/docker
```
2、重新创建daemon.json配置文件
```
cat > /etc/docker/daemon.json <<EOF
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2",
  "registry-mirrors": ["https://registry.cn-hangzhou.aliyuncs.com"]
}
EOF
```
3、启动并启用Docker服务
```
systemctl daemon-reload
systemctl enable --now docker
```
4、验证Docker安装和配置
```
docker info | grep -i cgroup
# 应该显示：Cgroup Driver: systemd

docker version
# 应该显示客户端和服务端版本信息
```
* **关闭swap**
```
swapoff -a
sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```
* **处理防火墙告警**
```
# 完全关闭防火墙（推荐测试环境）
systemctl stop firewalld
systemctl disable firewalld

# 或者只开放必要端口（生产环境）
firewall-cmd --permanent --add-port=6443/tcp
firewall-cmd --permanent --add-port=10250/tcp
firewall-cmd --reload
```


## 1. 初始化控制平面（注意替换apiserver-advertise-address为Master实际IP）
```
sudo kubeadm init \
  --apiserver-advertise-address=192.168.1.100 \
  --pod-network-cidr=10.244.0.0/16 \
  --image-repository registry.aliyuncs.com/google_containers
  ```


## 2. 初始化成功后，配置kubectl普通用户权限
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
## 3. 保存join命令（后续Node节点加入使用）
```
kubeadm token create --print-join-command > join-command.sh
chmod +x join-command.sh
```
## 4. 安装网络插件（这里使用Flannel）
```
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```
## 5、验证集群状态
```
kubectl get nodes
kubectl get pods --all-namespaces

[root@localhost ~]# kubectl get nodes
NAME                    STATUS     ROLES                  AGE     VERSION
localhost.localdomain   NotReady   control-plane,master   3m43s   v1.23.8
[root@localhost ~]# kubectl get pods --all-namespaces
NAMESPACE      NAME                                            READY   STATUS     RESTARTS   AGE
kube-flannel   kube-flannel-ds-kkjpb                           0/1     Init:0/2   0          23s
kube-system    coredns-6d8c4cb4d-79m2s                         0/1     Pending    0          3m40s
kube-system    coredns-6d8c4cb4d-8l6hx                         0/1     Pending    0          3m40s
kube-system    etcd-localhost.localdomain                      1/1     Running    0          3m52s
kube-system    kube-apiserver-localhost.localdomain            1/1     Running    0          3m52s
kube-system    kube-controller-manager-localhost.localdomain   1/1     Running    0          3m55s
kube-system    kube-proxy-khcdl                                1/1     Running    0          3m40s
kube-system    kube-scheduler-localhost.localdomain            1/1     Running    0          3m52s
```
# Kubernetes 集群已经基本配置完成，但目前还有一些组件处于初始化状态
1、节点状态
```
NAME                    STATUS     ROLES                  AGE     VERSION
localhost.localdomain   NotReady   control-plane,master   3m43s   v1.23.8
```
* NotReady 状态是因为网络插件尚未完全就绪

2、Pod状态
* kube-flannel-ds-kkjpb：正在初始化（Init:0/2）

* coredns：处于 Pending 状态（等待网络插件）

* 其他控制平面组件正常运行

**解决方案**
第一步：检查Flannel初始化问题
```
# 查看Flannel Pod的详细状态
[root@localhost ~]# kubectl describe pod -n kube-flannel kube-flannel-ds-kkjpb
Name:                 kube-flannel-ds-kkjpb
Namespace:            kube-flannel
Priority:             2000001000
Priority Class Name:  system-node-critical
Node:                 localhost.localdomain/192.168.225.130
Start Time:           Sat, 26 Jul 2025 11:52:41 -0700
Labels:               app=flannel
                      controller-revision-hash=784756c477
                      pod-template-generation=1
                      tier=node
Annotations:          <none>
Status:               Pending
IP:                   192.168.225.130
IPs:
  IP:           192.168.225.130
Controlled By:  DaemonSet/kube-flannel-ds
Init Containers:
  install-cni-plugin:
    Container ID:  docker://6cd29026e4cf0f7e91f9bb216285bdf6c4cd033bd9f7b5af1e82e9f070060c53
    Image:         ghcr.io/flannel-io/flannel-cni-plugin:v1.7.1-flannel1
    Image ID:      docker-pullable://ghcr.io/flannel-io/flannel-cni-plugin@sha256:cb3176a2c9eae5fa0acd7f45397e706eacb4577dac33cad89f93b775ff5611df
    Port:          <none>
    Host Port:     <none>
    Command:
      cp
    Args:
      -f
      /flannel
      /opt/cni/bin/flannel
    State:          Terminated
      Reason:       Completed
      Exit Code:    0
      Started:      Sat, 26 Jul 2025 11:54:48 -0700
      Finished:     Sat, 26 Jul 2025 11:54:48 -0700
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /opt/cni/bin from cni-plugin (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-rjc4w (ro)
  install-cni:
    Container ID:  
    Image:         ghcr.io/flannel-io/flannel:v0.27.2
    Image ID:      
    Port:          <none>
    Host Port:     <none>
    Command:
      cp
    Args:
      -f
      /etc/kube-flannel/cni-conf.json
      /etc/cni/net.d/10-flannel.conflist
    State:          Waiting
      Reason:       PodInitializing
    Ready:          False
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /etc/cni/net.d from cni (rw)
      /etc/kube-flannel/ from flannel-cfg (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-rjc4w (ro)
Containers:
  kube-flannel:
    Container ID:  
    Image:         ghcr.io/flannel-io/flannel:v0.27.2
    Image ID:      
    Port:          <none>
    Host Port:     <none>
    Command:
      /opt/bin/flanneld
    Args:
      --ip-masq
      --kube-subnet-mgr
    State:          Waiting
      Reason:       PodInitializing
    Ready:          False
    Restart Count:  0
    Requests:
      cpu:     100m
      memory:  50Mi
    Environment:
      POD_NAME:                   kube-flannel-ds-kkjpb (v1:metadata.name)
      POD_NAMESPACE:              kube-flannel (v1:metadata.namespace)
      EVENT_QUEUE_DEPTH:          5000
      CONT_WHEN_CACHE_NOT_READY:  false
    Mounts:
      /etc/kube-flannel/ from flannel-cfg (rw)
      /run/flannel from run (rw)
      /run/xtables.lock from xtables-lock (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-rjc4w (ro)
Conditions:
  Type              Status
  Initialized       False 
  Ready             False 
  ContainersReady   False 
  PodScheduled      True 
Volumes:
  run:
    Type:          HostPath (bare host directory volume)
    Path:          /run/flannel
    HostPathType:  
  cni-plugin:
    Type:          HostPath (bare host directory volume)
    Path:          /opt/cni/bin
    HostPathType:  
  cni:
    Type:          HostPath (bare host directory volume)
    Path:          /etc/cni/net.d
    HostPathType:  
  flannel-cfg:
    Type:      ConfigMap (a volume populated by a ConfigMap)
    Name:      kube-flannel-cfg
    Optional:  false
  xtables-lock:
    Type:          HostPath (bare host directory volume)
    Path:          /run/xtables.lock
    HostPathType:  FileOrCreate
  kube-api-access-rjc4w:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    ConfigMapOptional:       <nil>
    DownwardAPI:             true
QoS Class:                   Burstable
Node-Selectors:              <none>
Tolerations:                 :NoSchedule op=Exists
                             node.kubernetes.io/disk-pressure:NoSchedule op=Exists
                             node.kubernetes.io/memory-pressure:NoSchedule op=Exists
                             node.kubernetes.io/network-unavailable:NoSchedule op=Exists
                             node.kubernetes.io/not-ready:NoExecute op=Exists
                             node.kubernetes.io/pid-pressure:NoSchedule op=Exists
                             node.kubernetes.io/unreachable:NoExecute op=Exists
                             node.kubernetes.io/unschedulable:NoSchedule op=Exists
Events:
  Type    Reason     Age   From               Message
  ----    ------     ----  ----               -------
  Normal  Scheduled  4m9s  default-scheduler  Successfully assigned kube-flannel/kube-flannel-ds-kkjpb to localhost.localdomain
  Normal  Pulling    4m9s  kubelet            Pulling image "ghcr.io/flannel-io/flannel-cni-plugin:v1.7.1-flannel1"
  Normal  Pulled     2m2s  kubelet            Successfully pulled image "ghcr.io/flannel-io/flannel-cni-plugin:v1.7.1-flannel1" in 2m6.345391268s
  Normal  Created    2m2s  kubelet            Created container install-cni-plugin
  Normal  Started    2m2s  kubelet            Started container install-cni-plugin
  Normal  Pulling    2m1s  kubelet            Pulling image "ghcr.io/flannel-io/flannel:v0.27.2"

# 查看初始化容器的日志（通常是install-cni容器）
[root@localhost ~]# kubectl logs -n kube-flannel kube-flannel-ds-kkjpb -c install-cni
Error from server (BadRequest): container "install-cni" in pod "kube-flannel-ds-kkjpb" is waiting to start: PodInitializing

# 检查kubelet日志
[root@localhost ~]# journalctl -u kubelet -f | grep -i flannel
^C
```
当执行 journalctl -u kubelet -f | grep -i flannel 命令卡住不动时，通常有以下几种原因和解决方案:
1、没有新的 kubelet 日志产生
* 原因：*-f* 参数会实时跟踪新日志，如果 kubelet 没有产生新的与 flannel 相关的日志，命令会一直等待
* 验证方法：

```
# 先查看已有历史日志（不加 -f 参数）
[root@localhost ~]# journalctl -u kubelet | grep -i flannel | tail -20
Jul 26 11:52:41 localhost.localdomain kubelet[61574]: I0726 11:52:41.634270   61574 reconciler.go:221] "operationExecutor.VerifyControllerAttachedVolume started for volume \"cni\" (UniqueName: \"kubernetes.io/host-path/5cb41e8e-4a2e-4222-83aa-2c832d8f54d6-cni\") pod \"kube-flannel-ds-kkjpb\" (UID: \"5cb41e8e-4a2e-4222-83aa-2c832d8f54d6\") " pod="kube-flannel/kube-flannel-ds-kkjpb"
Jul 26 11:52:41 localhost.localdomain kubelet[61574]: I0726 11:52:41.634978   61574 reconciler.go:221] "operationExecutor.VerifyControllerAttachedVolume started for volume \"xtables-lock\" (UniqueName: \"kubernetes.io/host-path/5cb41e8e-4a2e-4222-83aa-2c832d8f54d6-xtables-lock\") pod \"kube-flannel-ds-kkjpb\" (UID: \"5cb41e8e-4a2e-4222-83aa-2c832d8f54d6\") " pod="kube-flannel/kube-flannel-ds-kkjpb"
Jul 26 11:52:41 localhost.localdomain kubelet[61574]: I0726 11:52:41.635000   61574 reconciler.go:221] "operationExecutor.VerifyControllerAttachedVolume started for volume \"flannel-cfg\" (UniqueName: \"kubernetes.io/configmap/5cb41e8e-4a2e-4222-83aa-2c832d8f54d6-flannel-cfg\") pod \"kube-flannel-ds-kkjpb\" (UID: \"5cb41e8e-4a2e-4222-83aa-2c832d8f54d6\") " pod="kube-flannel/kube-flannel-ds-kkjpb"
Jul 26 11:52:41 localhost.localdomain kubelet[61574]: I0726 11:52:41.635015   61574 reconciler.go:221] "operationExecutor.VerifyControllerAttachedVolume started for volume \"kube-api-access-rjc4w\" (UniqueName: \"kubernetes.io/projected/5cb41e8e-4a2e-4222-83aa-2c832d8f54d6-kube-api-access-rjc4w\") pod \"kube-flannel-ds-kkjpb\" (UID: \"5cb41e8e-4a2e-4222-83aa-2c832d8f54d6\") " pod="kube-flannel/kube-flannel-ds-kkjpb"
Jul 26 11:52:41 localhost.localdomain kubelet[61574]: I0726 11:52:41.635027   61574 reconciler.go:221] "operationExecutor.VerifyControllerAttachedVolume started for volume \"run\" (UniqueName: \"kubernetes.io/host-path/5cb41e8e-4a2e-4222-83aa-2c832d8f54d6-run\") pod \"kube-flannel-ds-kkjpb\" (UID: \"5cb41e8e-4a2e-4222-83aa-2c832d8f54d6\") " pod="kube-flannel/kube-flannel-ds-kkjpb"
Jul 26 11:52:41 localhost.localdomain kubelet[61574]: I0726 11:52:41.635040   61574 reconciler.go:221] "operationExecutor.VerifyControllerAttachedVolume started for volume \"cni-plugin\" (UniqueName: \"kubernetes.io/host-path/5cb41e8e-4a2e-4222-83aa-2c832d8f54d6-cni-plugin\") pod \"kube-flannel-ds-kkjpb\" (UID: \"5cb41e8e-4a2e-4222-83aa-2c832d8f54d6\") " pod="kube-flannel/kube-flannel-ds-kkjpb"
```
**当前无卷挂载相关错误**
日志日志均为 Info 级别，没有出现 Error（错误，开头为 E）或 Warning（警告，开头为 W），说明至少在 “卷挂载验证” 这一步，flannel 的 Pod 没有问题（未出现卷缺失、权限不足等导致挂载失败的情况）

* 解决方案：
如果确实没有日志，说明 kubelet 近期没有处理 flannel 相关事件
尝试手动触发网络插件操作：

```
[root@localhost ~]# kubectl delete pod -n kube-flannel kube-flannel-ds-xxxx
Error from server (NotFound): pods "kube-flannel-ds-xxxx" not found
```
```
# 检查 Kubernetes 节点上 kubelet 服务状态
[root@localhost ~]# systemctl status kubelet
● kubelet.service - kubelet: The Kubernetes Node Agent
   Loaded: loaded (/usr/lib/systemd/system/kubelet.service; enabled; vendor preset: disabled)
  Drop-In: /usr/lib/systemd/system/kubelet.service.d
           └─10-kubeadm.conf
   Active: active (running) since Sat 2025-07-26 11:49:11 PDT; 10min ago
     Docs: https://kubernetes.io/docs/
 Main PID: 61574 (kubelet)
    Tasks: 16
   Memory: 93.3M
   CGroup: /system.slice/kubelet.service
           └─61574 /usr/bin/kubelet --bootstrap-kubeconfig=/etc/kubernetes/bo...

Jul 26 11:58:56 localhost.localdomain kubelet[61574]: I0726 11:58:56.828381  ...
Jul 26 11:58:58 localhost.localdomain kubelet[61574]: E0726 11:58:58.047529  ...
Jul 26 11:59:01 localhost.localdomain kubelet[61574]: I0726 11:59:01.829464  ...
Jul 26 11:59:03 localhost.localdomain kubelet[61574]: E0726 11:59:03.056192  ...
Jul 26 11:59:06 localhost.localdomain kubelet[61574]: I0726 11:59:06.830119  ...
Jul 26 11:59:08 localhost.localdomain kubelet[61574]: E0726 11:59:08.065773  ...
Jul 26 11:59:11 localhost.localdomain kubelet[61574]: I0726 11:59:11.830464  ...
Jul 26 11:59:13 localhost.localdomain kubelet[61574]: E0726 11:59:13.075030  ...
Jul 26 11:59:16 localhost.localdomain kubelet[61574]: I0726 11:59:16.832256  ...
Jul 26 11:59:18 localhost.localdomain kubelet[61574]: E0726 11:59:18.082541  ...
Hint: Some lines were ellipsized, use -l to show in full.

#查看kubelet 完整状态（包含完整日志）
systemctl status kubelet -l

#查看最近十分钟的详细日志（重点看ERROR级别的行）
journalctl -u kubelet --since "10min ago" | grep -i error
```
```
# 查看 systemd 日志服务（systemd-journald） 的运行状态；
# 该服务是 Linux 系统中负责收集、存储和管理系统日志（包括内核日志、服务日志、应用程序日志等）的核心组件，journalctl 命令能获取日志正依赖于它
[root@localhost ~]# systemctl status systemd-journald
● systemd-journald.service - Journal Service
   Loaded: loaded (/usr/lib/systemd/system/systemd-journald.service; static; vendor preset: disabled)
   Active: active (running) since Sat 2025-07-26 09:59:17 PDT; 2h 0min ago
     Docs: man:systemd-journald.service(8)
           man:journald.conf(5)
 Main PID: 410 (systemd-journal)
   Status: "Processing requests..."
    Tasks: 1
   Memory: 148.0K
   CGroup: /system.slice/systemd-journald.service
           └─410 /usr/lib/systemd/systemd-journald

Jul 26 09:59:17 localhost.localdomain systemd-journal[410]: Runtime journal i…).
Jul 26 09:59:17 localhost.localdomain systemd-journal[410]: Journal started
Hint: Some lines were ellipsized, use -l to show in full.
[root@localhost ~]# sudo systemctl restart systemd-journald
[root@localhost ~]# sudo systemctl restart kubelet
```
输出可以明确：**日志服务（systemd-journald）运行完全正常**，没有任何错误或异常，这排除了 “日志无法记录” 导致 journalctl -u kubelet -f | grep -i flannel 卡住的可能性。
```
# 实时监控 kubelet 服务中与 flannel 相关的动态（如 flannel Pod 的启动、挂载卷、错误等）日志，常用于排查 Kubernetes 中 flannel 网络插件的运行问题。
[root@localhost ~]# sudo journalctl -u kubelet -f | grep -i flannel
Jul 26 11:59:55 localhost.localdomain kubelet[64483]: E0726 11:59:55.606855   64483 configmap.go:200]Couldn't get configMap kube-flannel/kube-flannel-cfg: failed to sync configmap cache: timed out waiting for the condition
Jul 26 11:59:55 localhost.localdomain kubelet[64483]: E0726 11:59:55.606990   64483 nestedpendingoperations.go:335] Operation for "{volumeName:kubernetes.io/configmap/5cb41e8e-4a2e-4222-83aa-2c832d8f54d6-flannel-cfg podName:5cb41e8e-4a2e-4222-83aa-2c832d8f54d6 nodeName:}" failed. No retries permitted until 2025-07-26 11:59:56.10695987 -0700 PDT m=+3.018131295 (durationBeforeRetry 500ms). Error: MountVolume.SetUp failed for volume "flannel-cfg" (UniqueName: "kubernetes.io/configmap/5cb41e8e-4a2e-4222-83aa-2c832d8f54d6-flannel-cfg") pod "kube-flannel-ds-kkjpb" (UID: "5cb41e8e-4a2e-4222-83aa-2c832d8f54d6") : failed to sync configmap cache: timed out waiting for the condition
^C
```
**错误日志解读**
两条错误的核心信息一致：
1、Couldn't get configMap kube-flannel/kube-flannel-cfg: failed to sync configmap cache: timed out waiting for the condition
→ kubelet 尝试从 Kubernetes 集群获取 kube-flannel 命名空间下的 kube-flannel-cfg 配置映射（ConfigMap）时，同步缓存超时失败。
2、MountVolume.SetUp failed for volume "flannel-cfg"... failed to sync configmap cache: timed out waiting for the condition
→ 由于无法获取上述 ConfigMap，flannel 的 Pod（kube-flannel-ds-kkjpb）无法挂载依赖的 flannel-cfg 卷（该卷本质是 ConfigMap 的挂载），导致 Pod 启动失败或异常

**为什么会出现这个问题？**
ConfigMap kube-flannel-cfg 是 flannel 网络插件的核心配置（包含网络子网、后端类型等关键参数），kubelet 必须能从 kube-apiserver 获取该配置才能正常挂载卷并启动 flannel Pod

**总结**
当前问题的核心是**kubelet 无法获取 flannel 必需的 ConfigMa**，优先检查 ConfigMap 是否存在，再验证 kubelet 与 apiserver 的连接和 apiserver 状态。重新部署 flannel 或修复网络连接后，该错误通常会消失，flannel Pod 会正常挂载卷并启动，节点网络也会恢复正常。

```
[root@localhost ~]# sudo usermod -aG adm $(whoami)

# 查看 kubelet 服务中与 flannel 相关的所有历史日志 的命令，与之前的实时监控命令相比，它会直接输出所有匹配的历史日志，而不进入分页模式
[root@localhost ~]# journalctl -u kubelet --no-pager | grep -i flannel
Jul 26 11:52:41 localhost.localdomain kubelet[61574]: I0726 11:52:41.634270   61574 reconciler.go:221] "operationExecutor.VerifyControllerAttachedVolume started for volume \"cni\" (UniqueName: \"kubernetes.io/host-path/5cb41e8e-4a2e-4222-83aa-2c832d8f54d6-cni\") pod \"kube-flannel-ds-kkjpb\" (UID: \"5cb41e8e-4a2e-4222-83aa-2c832d8f54d6\") " pod="kube-flannel/kube-flannel-ds-kkjpb"
Jul 26 11:52:41 localhost.localdomain kubelet[61574]: I0726 11:52:41.634978   61574 reconciler.go:221] "operationExecutor.VerifyControllerAttachedVolume started for volume \"xtables-lock\" (UniqueName: \"kubernetes.io/host-path/5cb41e8e-4a2e-4222-83aa-2c832d8f54d6-xtables-lock\") pod \"kube-flannel-ds-kkjpb\" (UID: \"5cb41e8e-4a2e-4222-83aa-2c832d8f54d6\") " pod="kube-flannel/kube-flannel-ds-kkjpb"
Jul 26 11:52:41 localhost.localdomain kubelet[61574]: I0726 11:52:41.635000   61574 reconciler.go:221] "operationExecutor.VerifyControllerAttachedVolume started for volume \"flannel-cfg\" (UniqueName: \"kubernetes.io/configmap/5cb41e8e-4a2e-4222-83aa-2c832d8f54d6-flannel-cfg\") pod \"kube-flannel-ds-kkjpb\" (UID: \"5cb41e8e-4a2e-4222-83aa-2c832d8f54d6\") " pod="kube-flannel/kube-flannel-ds-kkjpb"
Jul 26 11:52:41 localhost.localdomain kubelet[61574]: I0726 11:52:41.635015   61574 reconciler.go:221] "operationExecutor.VerifyControllerAttachedVolume started for volume \"kube-api-access-rjc4w\" (UniqueName: \"kubernetes.io/projected/5cb41e8e-4a2e-4222-83aa-2c832d8f54d6-kube-api-access-rjc4w\") pod \"kube-flannel-ds-kkjpb\" (UID: \"5cb41e8e-4a2e-4222-83aa-2c832d8f54d6\") " pod="kube-flannel/kube-flannel-ds-kkjpb"
Jul 26 11:52:41 localhost.localdomain kubelet[61574]: I0726 11:52:41.635027   61574 reconciler.go:221] "operationExecutor.VerifyControllerAttachedVolume started for volume \"run\" (UniqueName: \"kubernetes.io/host-path/5cb41e8e-4a2e-4222-83aa-2c832d8f54d6-run\") pod \"kube-flannel-ds-kkjpb\" (UID: \"5cb41e8e-4a2e-4222-83aa-2c832d8f54d6\") " pod="kube-flannel/kube-flannel-ds-kkjpb"
Jul 26 11:52:41 localhost.localdomain kubelet[61574]: I0726 11:52:41.635040   61574 reconciler.go:221] "operationExecutor.VerifyControllerAttachedVolume started for volume \"cni-plugin\" (UniqueName: \"kubernetes.io/host-path/5cb41e8e-4a2e-4222-83aa-2c832d8f54d6-cni-plugin\") pod \"kube-flannel-ds-kkjpb\" (UID: \"5cb41e8e-4a2e-4222-83aa-2c832d8f54d6\") " pod="kube-flannel/kube-flannel-ds-kkjpb"
Jul 26 11:59:54 localhost.localdomain kubelet[64483]: I0726 11:59:54.480416   64483 reconciler.go:221] "operationExecutor.VerifyControllerAttachedVolume started for volume \"cni-plugin\" (UniqueName: \"kubernetes.io/host-path/5cb41e8e-4a2e-4222-83aa-2c832d8f54d6-cni-plugin\") pod \"kube-flannel-ds-kkjpb\" (UID: \"5cb41e8e-4a2e-4222-83aa-2c832d8f54d6\") " pod="kube-flannel/kube-flannel-ds-kkjpb"
Jul 26 11:59:54 localhost.localdomain kubelet[64483]: I0726 11:59:54.480469   64483 reconciler.go:221] "operationExecutor.VerifyControllerAttachedVolume started for volume \"run\" (UniqueName: \"kubernetes.io/host-path/5cb41e8e-4a2e-4222-83aa-2c832d8f54d6-run\") pod \"kube-flannel-ds-kkjpb\" (UID: \"5cb41e8e-4a2e-4222-83aa-2c832d8f54d6\") " pod="kube-flannel/kube-flannel-ds-kkjpb"
Jul 26 11:59:54 localhost.localdomain kubelet[64483]: I0726 11:59:54.480513   64483 reconciler.go:221] "operationExecutor.VerifyControllerAttachedVolume started for volume \"flannel-cfg\" (UniqueName: \"kubernetes.io/configmap/5cb41e8e-4a2e-4222-83aa-2c832d8f54d6-flannel-cfg\") pod \"kube-flannel-ds-kkjpb\" (UID: \"5cb41e8e-4a2e-4222-83aa-2c832d8f54d6\") " pod="kube-flannel/kube-flannel-ds-kkjpb"
Jul 26 11:59:54 localhost.localdomain kubelet[64483]: I0726 11:59:54.480526   64483 reconciler.go:221] "operationExecutor.VerifyControllerAttachedVolume started for volume \"kube-api-access-rjc4w\" (UniqueName: \"kubernetes.io/projected/5cb41e8e-4a2e-4222-83aa-2c832d8f54d6-kube-api-access-rjc4w\") pod \"kube-flannel-ds-kkjpb\" (UID: \"5cb41e8e-4a2e-4222-83aa-2c832d8f54d6\") " pod="kube-flannel/kube-flannel-ds-kkjpb"
Jul 26 11:59:54 localhost.localdomain kubelet[64483]: I0726 11:59:54.480565   64483 reconciler.go:221] "operationExecutor.VerifyControllerAttachedVolume started for volume \"cni\" (UniqueName: \"kubernetes.io/host-path/5cb41e8e-4a2e-4222-83aa-2c832d8f54d6-cni\") pod \"kube-flannel-ds-kkjpb\" (UID: \"5cb41e8e-4a2e-4222-83aa-2c832d8f54d6\") " pod="kube-flannel/kube-flannel-ds-kkjpb"
Jul 26 11:59:54 localhost.localdomain kubelet[64483]: I0726 11:59:54.480577   64483 reconciler.go:221] "operationExecutor.VerifyControllerAttachedVolume started for volume \"xtables-lock\" (UniqueName: \"kubernetes.io/host-path/5cb41e8e-4a2e-4222-83aa-2c832d8f54d6-xtables-lock\") pod \"kube-flannel-ds-kkjpb\" (UID: \"5cb41e8e-4a2e-4222-83aa-2c832d8f54d6\") " pod="kube-flannel/kube-flannel-ds-kkjpb"
Jul 26 11:59:55 localhost.localdomain kubelet[64483]: E0726 11:59:55.606855   64483 configmap.go:200] Couldn't get configMap kube-flannel/kube-flannel-cfg: failed to sync configmap cache: timed out waiting for the condition
Jul 26 11:59:55 localhost.localdomain kubelet[64483]: E0726 11:59:55.606990   64483 nestedpendingoperations.go:335] Operation for "{volumeName:kubernetes.io/configmap/5cb41e8e-4a2e-4222-83aa-2c832d8f54d6-flannel-cfg podName:5cb41e8e-4a2e-4222-83aa-2c832d8f54d6 nodeName:}" failed. No retries permitted until 2025-07-26 11:59:56.10695987 -0700 PDT m=+3.018131295 (durationBeforeRetry 500ms). Error: MountVolume.SetUp failed for volume "flannel-cfg" (UniqueName: "kubernetes.io/configmap/5cb41e8e-4a2e-4222-83aa-2c832d8f54d6-flannel-cfg") pod "kube-flannel-ds-kkjpb" (UID: "5cb41e8e-4a2e-4222-83aa-2c832d8f54d6") : failed to sync configmap cache: timed out waiting for the condition
[root@localhost ~]# # 查询最近5分钟的日志
[root@localhost ~]# journalctl -u kubelet --since "5 minutes ago" | grep -i flannel
Jul 26 11:59:54 localhost.localdomain kubelet[64483]: I0726 11:59:54.480416   64483 reconciler.go:221] "operationExecutor.VerifyControllerAttachedVolume started for volume \"cni-plugin\" (UniqueName: \"kubernetes.io/host-path/5cb41e8e-4a2e-4222-83aa-2c832d8f54d6-cni-plugin\") pod \"kube-flannel-ds-kkjpb\" (UID: \"5cb41e8e-4a2e-4222-83aa-2c832d8f54d6\") " pod="kube-flannel/kube-flannel-ds-kkjpb"
Jul 26 11:59:54 localhost.localdomain kubelet[64483]: I0726 11:59:54.480469   64483 reconciler.go:221] "operationExecutor.VerifyControllerAttachedVolume started for volume \"run\" (UniqueName: \"kubernetes.io/host-path/5cb41e8e-4a2e-4222-83aa-2c832d8f54d6-run\") pod \"kube-flannel-ds-kkjpb\" (UID: \"5cb41e8e-4a2e-4222-83aa-2c832d8f54d6\") " pod="kube-flannel/kube-flannel-ds-kkjpb"
Jul 26 11:59:54 localhost.localdomain kubelet[64483]: I0726 11:59:54.480513   64483 reconciler.go:221] "operationExecutor.VerifyControllerAttachedVolume started for volume \"flannel-cfg\" (UniqueName: \"kubernetes.io/configmap/5cb41e8e-4a2e-4222-83aa-2c832d8f54d6-flannel-cfg\") pod \"kube-flannel-ds-kkjpb\" (UID: \"5cb41e8e-4a2e-4222-83aa-2c832d8f54d6\") " pod="kube-flannel/kube-flannel-ds-kkjpb"
Jul 26 11:59:54 localhost.localdomain kubelet[64483]: I0726 11:59:54.480526   64483 reconciler.go:221] "operationExecutor.VerifyControllerAttachedVolume started for volume \"kube-api-access-rjc4w\" (UniqueName: \"kubernetes.io/projected/5cb41e8e-4a2e-4222-83aa-2c832d8f54d6-kube-api-access-rjc4w\") pod \"kube-flannel-ds-kkjpb\" (UID: \"5cb41e8e-4a2e-4222-83aa-2c832d8f54d6\") " pod="kube-flannel/kube-flannel-ds-kkjpb"
Jul 26 11:59:54 localhost.localdomain kubelet[64483]: I0726 11:59:54.480565   64483 reconciler.go:221] "operationExecutor.VerifyControllerAttachedVolume started for volume \"cni\" (UniqueName: \"kubernetes.io/host-path/5cb41e8e-4a2e-4222-83aa-2c832d8f54d6-cni\") pod \"kube-flannel-ds-kkjpb\" (UID: \"5cb41e8e-4a2e-4222-83aa-2c832d8f54d6\") " pod="kube-flannel/kube-flannel-ds-kkjpb"
Jul 26 11:59:54 localhost.localdomain kubelet[64483]: I0726 11:59:54.480577   64483 reconciler.go:221] "operationExecutor.VerifyControllerAttachedVolume started for volume \"xtables-lock\" (UniqueName: \"kubernetes.io/host-path/5cb41e8e-4a2e-4222-83aa-2c832d8f54d6-xtables-lock\") pod \"kube-flannel-ds-kkjpb\" (UID: \"5cb41e8e-4a2e-4222-83aa-2c832d8f54d6\") " pod="kube-flannel/kube-flannel-ds-kkjpb"
Jul 26 11:59:55 localhost.localdomain kubelet[64483]: E0726 11:59:55.606855   64483 configmap.go:200] Couldn't get configMap kube-flannel/kube-flannel-cfg: failed to sync configmap cache: timed out waiting for the condition
Jul 26 11:59:55 localhost.localdomain kubelet[64483]: E0726 11:59:55.606990   64483 nestedpendingoperations.go:335] Operation for "{volumeName:kubernetes.io/configmap/5cb41e8e-4a2e-4222-83aa-2c832d8f54d6-flannel-cfg podName:5cb41e8e-4a2e-4222-83aa-2c832d8f54d6 nodeName:}" failed. No retries permitted until 2025-07-26 11:59:56.10695987 -0700 PDT m=+3.018131295 (durationBeforeRetry 500ms). Error: MountVolume.SetUp failed for volume "flannel-cfg" (UniqueName: "kubernetes.io/configmap/5cb41e8e-4a2e-4222-83aa-2c832d8f54d6-flannel-cfg") pod "kube-flannel-ds-kkjpb" (UID: "5cb41e8e-4a2e-4222-83aa-2c832d8f54d6") : failed to sync configmap cache: timed out waiting for the condition
```
***flannel 相关的操作经历了 “正常卷验证” 到 “ConfigMap 获取失败” 的过程**，核心问题集中在后期的 flannel-cfg ConfigMap 超时错误，这会直接导致 flannel 网络插件无法正常启动
```
# 查看 flannel 核心容器日志，用于直接获取 flannel 网络插件运行时的详细输出（包括启动过程、配置加载、网络初始化等信息），是排查 flannel 故障的核心手段，kube-flannel-ds-xxxx，flannel 的 Pod 名称（xxxx 是随机后缀，需替换为实际名称）
# 先通过以下命令获取实际名称：
[admin@localhost root]$ kubectl get pods -n kube-flannel
NAME                    READY   STATUS    RESTARTS   AGE
kube-flannel-ds-z9xdg   1/1     Running   0          3d
[admin@localhost root]$ kubectl logs -n kube-flannel kube-flannel-ds-z9xdg  -c kube-flannel
I0726 20:06:46.054702       1 main.go:213] CLI flags config: {etcdEndpoints:http://127.0.0.1:4001,http://127.0.0.1:2379 etcdPrefix:/coreos.com/network etcdKeyfile: etcdCertfile: etcdCAFile: etcdUsername: etcdPassword: version:false kubeSubnetMgr:true kubeApiUrl: kubeAnnotationPrefix:flannel.alpha.coreos.com kubeConfigFile: iface:[] ifaceRegex:[] ipMasq:true ifaceCanReach: subnetFile:/run/flannel/subnet.env publicIP: publicIPv6: subnetLeaseRenewMargin:60 healthzIP:0.0.0.0 healthzPort:0 iptablesResyncSeconds:5 iptablesForwardRules:true blackholeRoute:false netConfPath:/etc/kube-flannel/net-conf.json setNodeNetworkUnavailable:true}
W0726 20:06:46.054954       1 client_config.go:659] Neither --kubeconfig nor --master was specified.  Using the inClusterConfig.  This might not work.
I0726 20:06:46.079482       1 kube.go:139] Waiting 10m0s for node controller to sync
I0726 20:06:46.079697       1 kube.go:537] Starting kube subnet manager
I0726 20:06:46.082763       1 kube.go:558] Creating the node lease for IPv4. This is the n.Spec.PodCIDRs: [10.244.0.0/24]
I0726 20:06:47.079854       1 kube.go:163] Node controller sync successful
I0726 20:06:47.079897       1 main.go:239] Created subnet manager: Kubernetes Subnet Manager - localhost.localdomain
I0726 20:06:47.079902       1 main.go:242] Installing signal handlers
I0726 20:06:47.079977       1 main.go:519] Found network config - Backend type: vxlan
I0726 20:06:47.084018       1 kube.go:737] List of node(localhost.localdomain) annotations: map[string]string{"flannel.alpha.coreos.com/backend-data":"{\"VNI\":1,\"VtepMAC\":\"32:08:fc:60:1c:4d\"}", "flannel.alpha.coreos.com/backend-type":"vxlan", "flannel.alpha.coreos.com/kube-subnet-manager":"true", "flannel.alpha.coreos.com/public-ip":"192.168.225.130", "kubeadm.alpha.kubernetes.io/cri-socket":"/var/run/dockershim.sock", "node.alpha.kubernetes.io/ttl":"0", "volumes.kubernetes.io/controller-managed-attach-detach":"true"}
I0726 20:06:47.084085       1 match.go:211] Determining IP address of default interface
I0726 20:06:47.085558       1 match.go:264] Using interface with name ens33 and address 192.168.225.130
I0726 20:06:47.085593       1 match.go:286] Defaulting external address to interface address (192.168.225.130)
I0726 20:06:47.085667       1 vxlan.go:141] VXLAN config: VNI=1 Port=0 GBP=false Learning=false DirectRouting=false
I0726 20:06:47.088160       1 kube.go:704] List of node(localhost.localdomain) annotations: map[string]string{"flannel.alpha.coreos.com/backend-data":"{\"VNI\":1,\"VtepMAC\":\"32:08:fc:60:1c:4d\"}", "flannel.alpha.coreos.com/backend-type":"vxlan", "flannel.alpha.coreos.com/kube-subnet-manager":"true", "flannel.alpha.coreos.com/public-ip":"192.168.225.130", "kubeadm.alpha.kubernetes.io/cri-socket":"/var/run/dockershim.sock", "node.alpha.kubernetes.io/ttl":"0", "volumes.kubernetes.io/controller-managed-attach-detach":"true"}
I0726 20:06:47.088220       1 vxlan.go:155] Interface flannel.1 mac address set to: 32:08:fc:60:1c:4d
I0726 20:06:47.089022       1 main.go:375] Cleaning-up unused traffic manager rules
I0726 20:06:47.089058       1 nftables.go:278] Cleaning-up nftables rules...
I0726 20:06:47.129302       1 iptables.go:50] Starting flannel in iptables mode...
W0726 20:06:47.129829       1 main.go:608] no subnet found for key: FLANNEL_IPV6_NETWORK in file: /run/flannel/subnet.env
W0726 20:06:47.129880       1 main.go:608] no subnet found for key: FLANNEL_IPV6_SUBNET in file: /run/flannel/subnet.env
I0726 20:06:47.129892       1 iptables.go:110] Setting up masking rules
I0726 20:06:47.144119       1 iptables.go:211] Changing default FORWARD chain policy to ACCEPT
I0726 20:06:47.147933       1 main.go:463] Wrote subnet file to /run/flannel/subnet.env
I0726 20:06:47.147985       1 main.go:467] Running backend.
I0726 20:06:47.148436       1 vxlan_network.go:65] watching for new subnet leases
I0726 20:06:47.157030       1 main.go:488] Waiting for all goroutines to exit
I0726 20:06:47.164667       1 iptables.go:357] bootstrap done
I0726 20:06:47.176400       1 iptables.go:357] bootstrap done
```
```
# 查看 kube-flannel 命名空间中 DaemonSet 资源的状态。Flannel 网络插件通常通过 DaemonSet 部署（确保集群每个节点都运行一个 flannel Pod），因此该命令可验证 flannel 的部署是否正常
[root@localhost ~]# kubectl get daemonset -n kube-flannel
NAME              DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
kube-flannel-ds   1         1         0       1            0           <none>          8m46s
```
从输出结果 kube-flannel-ds 的 READY=0 来看，**flannel Pod 虽然已创建（CURRENT=1），但未处于就绪状态**，这表明网络插件尚未完全正常工作
```
# 查看集群中所有节点的污点（Taint）配置。污点是节点的一种属性，用于排斥特定的 Pod 调度到该节点（除非 Pod 配置了对应的容忍（Toleration））
[root@localhost ~]# kubectl describe node | grep -i taint
Taints:             node-role.kubernetes.io/master:NoSchedule
```
**node-role.kubernetes.io/master:NoSchedule **是 master 节点的默认保护机制，防止普通 Pod 占用资源。根据你的集群规模（单节点 / 多节点），可选择移除污点（单节点场景）或为特定 Pod 添加容忍（多节点场景需在 master 节点部署特殊组件时）。操作后，可通过 *kubectl get pods* 验证 Pod 调度状态是否恢复正常
```
# 查看当前系统内核中是否加载了与网桥（bridge）相关的模块，这些模块是 Kubernetes 网络插件（如 flannel、calico 等）正常工作的基础（网桥用于实现 Pod 之间及 Pod 与节点之间的网络通信）
[root@localhost ~]# lsmod | grep bridge
bridge                151336  1 br_netfilter
stp                    12976  1 bridge
llc                    14552  2 stp,bridge
```
**模块状态分析**
* bridge 模块（核心网桥功能）
输出显示 bridge 151336 1 br_netfilter，说明该模块已加载，且被 br_netfilter 模块依赖。
作用：提供 Linux 网桥的基础功能，支持创建虚拟网桥（如 flannel 需要的 cni0 网桥），实现 Pod 之间的二层网络通信。

* br_netfilter 模块（网桥网络过滤）
作为 bridge 模块的依赖项存在，是 Kubernetes 网络功能的关键模块。
作用：允许 iptables 规则作用于网桥流量（如 cni0 网桥），支持 Kubernetes 的 Service 端口转发、网络策略（Network Policy）、Pod 间通信隔离等功能。

* stp 和 llc 模块（辅助功能）
stp（生成树协议）用于避免网桥网络中的环路问题，llc 是链路层控制协议模块，两者均为 bridge 模块的依赖项。
作用：保障网桥网络的稳定性，防止因网络环路导致的流量风暴。
对集群的影响
当前模块加载状态完全正常，不存在因网桥模块缺失导致的网络问题。结合你之前的 flannel 日志（已正常启动）和 DaemonSet 状态（CURRENT=1），网桥模块的正常加载为 flannel 创建 cni0 网桥和 flannel.1 虚拟接口提供了基础支持。
后续验证建议
若仍存在网络相关问题（如 Pod 无法通信），可进一步检查网桥接口状态：


**查看 flannel 创建的 cni0 网桥和 flannel.1 虚拟接口**
```
ip addr show cni0
ip addr show flannel.1
```
**查看网桥与 Pod 虚拟网卡的连接关系**
```
brctl show cni0  # 若未安装 brctl，可先执行 yum install -y bridge-utils
```
* 正常情况下：

cni0 应分配 Pod 网段的网关 IP（如 10.244.0.1/24）；
flannel.1 应分配节点在 flannel 网络中的 IP（如 10.244.0.0/32）；
brctl show cni0 应显示连接到该网桥的 Pod 虚拟网卡（vethxxxx）。
**总结**
网桥相关模块加载正常，排除了因内核模块缺失导致的网络插件故障。若后续出现网络问题，可重点排查 flannel 配置、Pod 状态或 iptables 规则，而非模块依赖。当前模块状态为集群网络功能提供了可靠的底层支持。

```
# 检查ConfigMap是否存在
[root@localhost ~]# kubectl get configmap -n kube-flannel
NAME               DATA   AGE
kube-flannel-cfg   2      16m
kube-root-ca.crt   1      16m
# 检查Flannel的ServiceAccount权限
[root@localhost ~]# kubectl describe clusterrolebinding flannel
Name:         flannel
Labels:       k8s-app=flannel
Annotations:  <none>
Role:
  Kind:  ClusterRole
  Name:  flannel
Subjects:
  Kind            Name     Namespace
  ----            ----     ---------
  ServiceAccount  flannel  kube-flannel
[root@localhost ~]# kubectl describe clusterrole flannel
Name:         flannel
Labels:       k8s-app=flannel
Annotations:  <none>
PolicyRule:
  Resources     Non-Resource URLs  Resource Names  Verbs
  ---------     -----------------  --------------  -----
  nodes         []                 []              [get list watch]
  pods          []                 []              [get]
  nodes/status  []                 []              [patch]
```
根据检查结果显示，**ConfigMap 已存在但无法挂载**，**Flannel权限正常**
```
# 检查ConfigMap内容是否有效，确认输出包含有效的 cni-conf.json 和 net-conf.json 数据
[root@localhost ~]# kubectl get configmap -n kube-flannel kube-flannel-cfg -o yaml
apiVersion: v1
data:
  cni-conf.json: |
    {
      "name": "cbr0",
      "cniVersion": "0.3.1",
      "plugins": [
        {
          "type": "flannel",
          "delegate": {
            "hairpinMode": true,
            "isDefaultGateway": true
          }
        },
        {
          "type": "portmap",
          "capabilities": {
            "portMappings": true
          }
        }
      ]
    }
  net-conf.json: |
    {
      "Network": "10.244.0.0/16",
      "EnableNFTables": false,
      "Backend": {
        "Type": "vxlan"
      }
    }
kind: ConfigMap
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","data":{"cni-conf.json":"{\n  \"name\": \"cbr0\",\n  \"cniVersion\": \"0.3.1\",\n  \"plugins\": [\n    {\n      \"type\": \"flannel\",\n      \"delegate\": {\n        \"hairpinMode\": true,\n        \"isDefaultGateway\": true\n      }\n    },\n    {\n      \"type\": \"portmap\",\n      \"capabilities\": {\n        \"portMappings\": true\n      }\n    }\n  ]\n}\n","net-conf.json":"{\n  \"Network\": \"10.244.0.0/16\",\n  \"EnableNFTables\": false,\n  \"Backend\": {\n    \"Type\": \"vxlan\"\n  }\n}\n"},"kind":"ConfigMap","metadata":{"annotations":{},"labels":{"app":"flannel","k8s-app":"flannel","tier":"node"},"name":"kube-flannel-cfg","namespace":"kube-flannel"}}
  creationTimestamp: "2025-07-26T18:52:41Z"
  labels:
    app: flannel
    k8s-app: flannel
    tier: node
  name: kube-flannel-cfg
  namespace: kube-flannel
  resourceVersion: "707"
  uid: 7a5e26ae-f7f5-4790-8b52-44635ff09aec
```
根据提供的 ConfigMap 详细配置，Flannel 的网络配置看起来完全正常
针对性解决方案：**检查kubelet的ConfigMap挂载能力**
```
# 检查kubelet日志中的挂载错误细节
[root@localhost ~]# sudo journalctl -u kubelet | grep -A 20 -B 20 "MountVolume.SetUp.*flannel-cfg"
Jul 26 11:59:54 localhost.localdomain kubelet[64483]: I0726 11:59:54.246915   64483 apiserver.go:52] "Watching apiserver"
Jul 26 11:59:54 localhost.localdomain kubelet[64483]: I0726 11:59:54.445713   64483 topology_manager.go:200] "Topology Admit Handler"
Jul 26 11:59:54 localhost.localdomain kubelet[64483]: I0726 11:59:54.445833   64483 topology_manager.go:200] "Topology Admit Handler"
Jul 26 11:59:54 localhost.localdomain kubelet[64483]: I0726 11:59:54.480416   64483 reconciler.go:221] "operationExecutor.VerifyControllerAttachedVolume started for volume \"cni-plugin\" (UniqueName: \"kubernetes.io/host-path/5cb41e8e-4a2e-4222-83aa-2c832d8f54d6-cni-plugin\") pod \"kube-flannel-ds-kkjpb\" (UID: \"5cb41e8e-4a2e-4222-83aa-2c832d8f54d6\") " pod="kube-flannel/kube-flannel-ds-kkjpb"
Jul 26 11:59:54 localhost.localdomain kubelet[64483]: I0726 11:59:54.480452   64483 reconciler.go:221] "operationExecutor.VerifyControllerAttachedVolume started for volume \"lib-modules\" (UniqueName: \"kubernetes.io/host-path/40b81ba1-25e3-4cdd-9388-ea2b1678291f-lib-modules\") pod \"kube-proxy-khcdl\" (UID: \"40b81ba1-25e3-4cdd-9388-ea2b1678291f\") " pod="kube-system/kube-proxy-khcdl"
Jul 26 11:59:54 localhost.localdomain kubelet[64483]: I0726 11:59:54.480469   64483 reconciler.go:221] "operationExecutor.VerifyControllerAttachedVolume started for volume \"run\" (UniqueName: \"kubernetes.io/host-path/5cb41e8e-4a2e-4222-83aa-2c832d8f54d6-run\") pod \"kube-flannel-ds-kkjpb\" (UID: \"5cb41e8e-4a2e-4222-83aa-2c832d8f54d6\") " pod="kube-flannel/kube-flannel-ds-kkjpb"
Jul 26 11:59:54 localhost.localdomain kubelet[64483]: I0726 11:59:54.480484   64483 reconciler.go:221] "operationExecutor.VerifyControllerAttachedVolume started for volume \"kube-api-access-fq9cp\" (UniqueName: \"kubernetes.io/projected/40b81ba1-25e3-4cdd-9388-ea2b1678291f-kube-api-access-fq9cp\") pod \"kube-proxy-khcdl\" (UID: \"40b81ba1-25e3-4cdd-9388-ea2b1678291f\") " pod="kube-system/kube-proxy-khcdl"
Jul 26 11:59:54 localhost.localdomain kubelet[64483]: I0726 11:59:54.480513   64483 reconciler.go:221] "operationExecutor.VerifyControllerAttachedVolume started for volume \"flannel-cfg\" (UniqueName: \"kubernetes.io/configmap/5cb41e8e-4a2e-4222-83aa-2c832d8f54d6-flannel-cfg\") pod \"kube-flannel-ds-kkjpb\" (UID: \"5cb41e8e-4a2e-4222-83aa-2c832d8f54d6\") " pod="kube-flannel/kube-flannel-ds-kkjpb"
Jul 26 11:59:54 localhost.localdomain kubelet[64483]: I0726 11:59:54.480526   64483 reconciler.go:221] "operationExecutor.VerifyControllerAttachedVolume started for volume \"kube-api-access-rjc4w\" (UniqueName: \"kubernetes.io/projected/5cb41e8e-4a2e-4222-83aa-2c832d8f54d6-kube-api-access-rjc4w\") pod \"kube-flannel-ds-kkjpb\" (UID: \"5cb41e8e-4a2e-4222-83aa-2c832d8f54d6\") " pod="kube-flannel/kube-flannel-ds-kkjpb"
Jul 26 11:59:54 localhost.localdomain kubelet[64483]: I0726 11:59:54.480539   64483 reconciler.go:221] "operationExecutor.VerifyControllerAttachedVolume started for volume \"kube-proxy\" (UniqueName: \"kubernetes.io/configmap/40b81ba1-25e3-4cdd-9388-ea2b1678291f-kube-proxy\") pod \"kube-proxy-khcdl\" (UID: \"40b81ba1-25e3-4cdd-9388-ea2b1678291f\") " pod="kube-system/kube-proxy-khcdl"
Jul 26 11:59:54 localhost.localdomain kubelet[64483]: I0726 11:59:54.480552   64483 reconciler.go:221] "operationExecutor.VerifyControllerAttachedVolume started for volume \"xtables-lock\" (UniqueName: \"kubernetes.io/host-path/40b81ba1-25e3-4cdd-9388-ea2b1678291f-xtables-lock\") pod \"kube-proxy-khcdl\" (UID: \"40b81ba1-25e3-4cdd-9388-ea2b1678291f\") " pod="kube-system/kube-proxy-khcdl"
Jul 26 11:59:54 localhost.localdomain kubelet[64483]: I0726 11:59:54.480565   64483 reconciler.go:221] "operationExecutor.VerifyControllerAttachedVolume started for volume \"cni\" (UniqueName: \"kubernetes.io/host-path/5cb41e8e-4a2e-4222-83aa-2c832d8f54d6-cni\") pod \"kube-flannel-ds-kkjpb\" (UID: \"5cb41e8e-4a2e-4222-83aa-2c832d8f54d6\") " pod="kube-flannel/kube-flannel-ds-kkjpb"
Jul 26 11:59:54 localhost.localdomain kubelet[64483]: I0726 11:59:54.480577   64483 reconciler.go:221] "operationExecutor.VerifyControllerAttachedVolume started for volume \"xtables-lock\" (UniqueName: \"kubernetes.io/host-path/5cb41e8e-4a2e-4222-83aa-2c832d8f54d6-xtables-lock\") pod \"kube-flannel-ds-kkjpb\" (UID: \"5cb41e8e-4a2e-4222-83aa-2c832d8f54d6\") " pod="kube-flannel/kube-flannel-ds-kkjpb"
Jul 26 11:59:54 localhost.localdomain kubelet[64483]: I0726 11:59:54.480588   64483 reconciler.go:157] "Reconciler: start to sync state"
Jul 26 11:59:54 localhost.localdomain kubelet[64483]: E0726 11:59:54.849569   64483 kubelet.go:1742] "Failed creating a mirror pod for" err="pods \"kube-scheduler-localhost.localdomain\" already exists" pod="kube-system/kube-scheduler-localhost.localdomain"
Jul 26 11:59:55 localhost.localdomain kubelet[64483]: E0726 11:59:55.047432   64483 kubelet.go:1742] "Failed creating a mirror pod for" err="pods \"kube-apiserver-localhost.localdomain\" already exists" pod="kube-system/kube-apiserver-localhost.localdomain"
Jul 26 11:59:55 localhost.localdomain kubelet[64483]: E0726 11:59:55.249335   64483 kubelet.go:1742] "Failed creating a mirror pod for" err="pods \"kube-controller-manager-localhost.localdomain\" already exists" pod="kube-system/kube-controller-manager-localhost.localdomain"
Jul 26 11:59:55 localhost.localdomain kubelet[64483]: I0726 11:59:55.443363   64483 request.go:665] Waited for 1.062413402s due to client-side throttling, not priority and fairness, request: POST:https://192.168.225.130:6443/api/v1/namespaces/kube-system/pods
Jul 26 11:59:55 localhost.localdomain kubelet[64483]: E0726 11:59:55.447647   64483 kubelet.go:1742] "Failed creating a mirror pod for" err="pods \"etcd-localhost.localdomain\" already exists" pod="kube-system/etcd-localhost.localdomain"
Jul 26 11:59:55 localhost.localdomain kubelet[64483]: E0726 11:59:55.606855   64483 configmap.go:200] Couldn't get configMap kube-flannel/kube-flannel-cfg: failed to sync configmap cache: timed out waiting for the condition
Jul 26 11:59:55 localhost.localdomain kubelet[64483]: E0726 11:59:55.606990   64483 nestedpendingoperations.go:335] Operation for "{volumeName:kubernetes.io/configmap/5cb41e8e-4a2e-4222-83aa-2c832d8f54d6-flannel-cfg podName:5cb41e8e-4a2e-4222-83aa-2c832d8f54d6 nodeName:}" failed. No retries permitted until 2025-07-26 11:59:56.10695987 -0700 PDT m=+3.018131295 (durationBeforeRetry 500ms). Error: MountVolume.SetUp failed for volume "flannel-cfg" (UniqueName: "kubernetes.io/configmap/5cb41e8e-4a2e-4222-83aa-2c832d8f54d6-flannel-cfg") pod "kube-flannel-ds-kkjpb" (UID: "5cb41e8e-4a2e-4222-83aa-2c832d8f54d6") : failed to sync configmap cache: timed out waiting for the condition
Jul 26 11:59:55 localhost.localdomain kubelet[64483]: E0726 11:59:55.606860   64483 configmap.go:200] Couldn't get configMap kube-system/kube-proxy: failed to sync configmap cache: timed out waiting for the condition
Jul 26 11:59:55 localhost.localdomain kubelet[64483]: E0726 11:59:55.607198   64483 nestedpendingoperations.go:335] Operation for "{volumeName:kubernetes.io/configmap/40b81ba1-25e3-4cdd-9388-ea2b1678291f-kube-proxy podName:40b81ba1-25e3-4cdd-9388-ea2b1678291f nodeName:}" failed. No retries permitted until 2025-07-26 11:59:56.107189562 -0700 PDT m=+3.018360987 (durationBeforeRetry 500ms). Error: MountVolume.SetUp failed for volume "kube-proxy" (UniqueName: "kubernetes.io/configmap/40b81ba1-25e3-4cdd-9388-ea2b1678291f-kube-proxy") pod "kube-proxy-khcdl" (UID: "40b81ba1-25e3-4cdd-9388-ea2b1678291f") : failed to sync configmap cache: timed out waiting for the condition
Jul 26 11:59:58 localhost.localdomain kubelet[64483]: I0726 11:59:58.221098   64483 cni.go:240] "Unable to update cni config" err="no networks found in /etc/cni/net.d"
Jul 26 11:59:58 localhost.localdomain kubelet[64483]: E0726 11:59:58.485725   64483 kubelet.go:2391] "Container runtime network not ready" networkReady="NetworkReady=false reason:NetworkPluginNotReady message:docker: network plugin is not ready: cni config uninitialized"
Jul 26 12:00:03 localhost.localdomain kubelet[64483]: I0726 12:00:03.222178   64483 cni.go:240] "Unable to update cni config" err="no networks found in /etc/cni/net.d"
Jul 26 12:00:03 localhost.localdomain kubelet[64483]: E0726 12:00:03.495433   64483 kubelet.go:2391] "Container runtime network not ready" networkReady="NetworkReady=false reason:NetworkPluginNotReady message:docker: network plugin is not ready: cni config uninitialized"
Jul 26 12:00:08 localhost.localdomain kubelet[64483]: I0726 12:00:08.223529   64483 cni.go:240] "Unable to update cni config" err="no networks found in /etc/cni/net.d"
Jul 26 12:00:08 localhost.localdomain kubelet[64483]: E0726 12:00:08.503192   64483 kubelet.go:2391] "Container runtime network not ready" networkReady="NetworkReady=false reason:NetworkPluginNotReady message:docker: network plugin is not ready: cni config uninitialized"
Jul 26 12:00:13 localhost.localdomain kubelet[64483]: I0726 12:00:13.224394   64483 cni.go:240] "Unable to update cni config" err="no networks found in /etc/cni/net.d"
Jul 26 12:00:13 localhost.localdomain kubelet[64483]: E0726 12:00:13.510724   64483 kubelet.go:2391] "Container runtime network not ready" networkReady="NetworkReady=false reason:NetworkPluginNotReady message:docker: network plugin is not ready: cni config uninitialized"
Jul 26 12:00:18 localhost.localdomain kubelet[64483]: I0726 12:00:18.225163   64483 cni.go:240] "Unable to update cni config" err="no networks found in /etc/cni/net.d"
Jul 26 12:00:18 localhost.localdomain kubelet[64483]: E0726 12:00:18.520045   64483 kubelet.go:2391] "Container runtime network not ready" networkReady="NetworkReady=false reason:NetworkPluginNotReady message:docker: network plugin is not ready: cni config uninitialized"
Jul 26 12:00:23 localhost.localdomain kubelet[64483]: I0726 12:00:23.225566   64483 cni.go:240] "Unable to update cni config" err="no networks found in /etc/cni/net.d"
Jul 26 12:00:23 localhost.localdomain kubelet[64483]: E0726 12:00:23.527926   64483 kubelet.go:2391] "Container runtime network not ready" networkReady="NetworkReady=false reason:NetworkPluginNotReady message:docker: network plugin is not ready: cni config uninitialized"
Jul 26 12:00:28 localhost.localdomain kubelet[64483]: I0726 12:00:28.227029   64483 cni.go:240] "Unable to update cni config" err="no networks found in /etc/cni/net.d"
Jul 26 12:00:28 localhost.localdomain kubelet[64483]: E0726 12:00:28.536470   64483 kubelet.go:2391] "Container runtime network not ready" networkReady="NetworkReady=false reason:NetworkPluginNotReady message:docker: network plugin is not ready: cni config uninitialized"
Jul 26 12:00:33 localhost.localdomain kubelet[64483]: I0726 12:00:33.228127   64483 cni.go:240] "Unable to update cni config" err="no networks found in /etc/cni/net.d"
Jul 26 12:00:33 localhost.localdomain kubelet[64483]: E0726 12:00:33.543675   64483 kubelet.go:2391] "Container runtime network not ready" networkReady="NetworkReady=false reason:NetworkPluginNotReady message:docker: network plugin is not ready: cni config uninitialized"
Jul 26 12:00:38 localhost.localdomain kubelet[64483]: I0726 12:00:38.229282   64483 cni.go:240] "Unable to update cni config" err="no networks found in /etc/cni/net.d"
Jul 26 12:00:38 localhost.localdomain kubelet[64483]: E0726 12:00:38.551565   64483 kubelet.go:2391] "Container runtime network not ready" networkReady="NetworkReady=false reason:NetworkPluginNotReady message:docker: network plugin is not ready: cni config uninitialized"
```
**API Server通信故障**
* 日志连续报错：
Couldn't get configMap kube-flannel/kube-flannel-cfg: failed to sync configmap cache
* kubelet 无法从 API Server 获取 ConfigMap 数据，导致 Volume 挂载失败

**CNI配置缺失**
* 日志显示：
Unable to update cni config: no networks found in /etc/cni/net.d

根据最新提供的日志和进程信息，可以确认问题根源是 **kubelet 无法从 API Server 获取 ConfigMap 数据**，导致 Flannel 网络插件无法正常初始化。

```
# 验证kubelet配置
[root@localhost ~]# ps -ef | grep kubelet | grep -v grep
root      61362  61322  5 11:49 ?        00:01:14 kube-apiserver --advertise-address=192.168.225.130 --allow-privileged=true --authorization-mode=Node,RBAC --client-ca-file=/etc/kubernetes/pki/ca.crt --enable-admission-plugins=NodeRestriction --enable-bootstrap-token-auth=true --etcd-cafile=/etc/kubernetes/pki/etcd/ca.crt --etcd-certfile=/etc/kubernetes/pki/apiserver-etcd-client.crt --etcd-keyfile=/etc/kubernetes/pki/apiserver-etcd-client.key --etcd-servers=https://127.0.0.1:2379 --kubelet-client-certificate=/etc/kubernetes/pki/apiserver-kubelet-client.crt --kubelet-client-key=/etc/kubernetes/pki/apiserver-kubelet-client.key --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname --proxy-client-cert-file=/etc/kubernetes/pki/front-proxy-client.crt --proxy-client-key-file=/etc/kubernetes/pki/front-proxy-client.key --requestheader-allowed-names=front-proxy-client --requestheader-client-ca-file=/etc/kubernetes/pki/front-proxy-ca.crt --requestheader-extra-headers-prefix=X-Remote-Extra- --requestheader-group-headers=X-Remote-Group --requestheader-username-headers=X-Remote-User --secure-port=6443 --service-account-issuer=https://kubernetes.default.svc.cluster.local --service-account-key-file=/etc/kubernetes/pki/sa.pub --service-account-signing-key-file=/etc/kubernetes/pki/sa.key --service-cluster-ip-range=10.96.0.0/12 --tls-cert-file=/etc/kubernetes/pki/apiserver.crt --tls-private-key-file=/etc/kubernetes/pki/apiserver.key
root      64483      1  2 11:59 ?        00:00:17 /usr/bin/kubelet --bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf --config=/var/lib/kubelet/config.yaml --network-plugin=cni --pod-infra-container-image=registry.aliyuncs.com/google_containers/pause:3.6
```
**证书或认证问题**
* kubelet进程参数中缺少关键TLS证书配置：
ps -ef | grep kubelet
输出中缺少参数：
--client-ca-file=/etc/kubernetes/pki/ca.crt
--tls-cert-file=/etc/kubernetes/pki/kubelet.crt
--tls-private-key-file=/etc/kubernetes/pki/kubelet.key 

```
# 检查关键证书是否存在
[root@localhost ~]# ls -l /etc/kubernetes/pki/kubelet.*
ls: cannot access /etc/kubernetes/pki/kubelet.*: No such file or directory
[root@localhost ~]# ls -l /etc/kubernetes/pki/ca.crt
-rw-r--r--. 1 root root 1099 Jul 26 11:48 /etc/kubernetes/pki/ca.crt
```
根据检查结果，确认 **kubelet 的 TLS 证书缺失** 是导致问题的根本原因。以下是完整的修复方案：
```
# 备份现有配置
[root@localhost ~]# sudo mv /etc/kubernetes/kubelet.conf /etc/kubernetes/kubelet.conf.bak
# 如果证书不存在，重新生成kubelet证书和配置
[root@localhost ~]# sudo kubeadm init phase kubeconfig kubelet --apiserver-advertise-address 192.168.225.130
I0726 12:17:02.752793   69868 version.go:255] remote version is much newer: v1.33.3; falling back to: stable-1.23
[kubeconfig] Writing "kubelet.conf" kubeconfig file
# 查看生成的证书
[root@localhost ~]# ls -l /etc/kubernetes/pki/kubelet.*
ls: cannot access /etc/kubernetes/pki/kubelet.*: No such file or directory
```
虽然重新生成了 **kubelet.conf 配置文件，但关键的 kubelet 服务证书仍未生成**。这是导致网络插件无法正常工作的根本原因。以下是完整的修复方案：
```
# 创建证书签名请求（CSR）
[root@localhost ~]# cat <<EOF | sudo tee /etc/kubernetes/pki/kubelet-csr.json
> {
>   "CN": "system:node:$(hostname)",
>   "key": {
>     "algo": "rsa",
>     "size": 2048
>   },
>   "names": [
>     {
>       "O": "system:nodes"
>     }
>   ]
> }
> EOF
{
  "CN": "system:node:localhost.localdomain",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "O": "system:nodes"
    }
  ]
}
# 使用集群CA签发证书
[root@localhost ~]# sudo cfssl gencert \
>   -ca=/etc/kubernetes/pki/ca.crt \
>   -ca-key=/etc/kubernetes/pki/ca.key \
>   -config=/etc/kubernetes/pki/ca-config.json \
>   -hostname=$(hostname),192.168.225.130 \
>   -profile=kubernetes \
>   /etc/kubernetes/pki/kubelet-csr.json | sudo cfssljson -bare /etc/kubernetes/pki/kubelet
sudo: cfssljson: command not found
sudo: cfssl: command not found
```
从报错信息来看，您的系统缺少 **cfssl** 和 **cfssljson** 工具，这是生成证书所需的工具。以下是完整的解决方案：
**安装CFSSL工具**
```
# 下载cfssl工具
[root@localhost ~]# wget https://pkg.cfssl.org/R1.2/cfssl_linux-amd64
--2025-07-26 12:18:56--  https://pkg.cfssl.org/R1.2/cfssl_linux-amd64
Resolving pkg.cfssl.org (pkg.cfssl.org)... 104.18.17.120, 104.18.16.120, 2606:4700::6812:1178, ...
Connecting to pkg.cfssl.org (pkg.cfssl.org)|104.18.17.120|:443... connected.
HTTP request sent, awaiting response... 301 Moved Permanently
Location: https://github.com/cloudflare/cfssl/releases/download/1.2.0/cfssl_linux-amd64 [following]
--2025-07-26 12:18:57--  https://github.com/cloudflare/cfssl/releases/download/1.2.0/cfssl_linux-amd64
Resolving github.com (github.com)... 20.205.243.166
Connecting to github.com (github.com)|20.205.243.166|:443... connected.
HTTP request sent, awaiting response... 302 Found
Location: https://release-assets.githubusercontent.com/github-production-release-asset/21591001/6deaa080-9ebe-11eb-919d-cbab8a7bb20b?sp=r&sv=2018-11-09&sr=b&spr=https&se=2025-07-26T20%3A16%3A05Z&rscd=attachment%3B+filename%3Dcfssl_linux-amd64&rsct=application%2Foctet-stream&skoid=96c2d410-5711-43a1-aedd-ab1947aa7ab0&sktid=398a6654-997b-47e9-b12b-9515b896b4de&skt=2025-07-26T19%3A15%3A13Z&ske=2025-07-26T20%3A16%3A05Z&sks=b&skv=2018-11-09&sig=oHnFvqheTY58DNSzr1IXTPHjmUbpjWEH2kpz%2FxkBQKI%3D&jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmVsZWFzZS1hc3NldHMuZ2l0aHVidXNlcmNvbnRlbnQuY29tIiwia2V5Ijoia2V5MSIsImV4cCI6MTc1MzU1NzgzOCwibmJmIjoxNzUzNTU3NTM4LCJwYXRoIjoicmVsZWFzZWFzc2V0cHJvZHVjdGlvbi5ibG9iLmNvcmUud2luZG93cy5uZXQifQ.L7huQRpyZJbtvsCj9o10qvcypyk7ij9G6ZvEB6yYnRk&response-content-disposition=attachment%3B%20filename%3Dcfssl_linux-amd64&response-content-type=application%2Foctet-stream [following]
--2025-07-26 12:18:58--  https://release-assets.githubusercontent.com/github-production-release-asset/21591001/6deaa080-9ebe-11eb-919d-cbab8a7bb20b?sp=r&sv=2018-11-09&sr=b&spr=https&se=2025-07-26T20%3A16%3A05Z&rscd=attachment%3B+filename%3Dcfssl_linux-amd64&rsct=application%2Foctet-stream&skoid=96c2d410-5711-43a1-aedd-ab1947aa7ab0&sktid=398a6654-997b-47e9-b12b-9515b896b4de&skt=2025-07-26T19%3A15%3A13Z&ske=2025-07-26T20%3A16%3A05Z&sks=b&skv=2018-11-09&sig=oHnFvqheTY58DNSzr1IXTPHjmUbpjWEH2kpz%2FxkBQKI%3D&jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmVsZWFzZS1hc3NldHMuZ2l0aHVidXNlcmNvbnRlbnQuY29tIiwia2V5Ijoia2V5MSIsImV4cCI6MTc1MzU1NzgzOCwibmJmIjoxNzUzNTU3NTM4LCJwYXRoIjoicmVsZWFzZWFzc2V0cHJvZHVjdGlvbi5ibG9iLmNvcmUud2luZG93cy5uZXQifQ.L7huQRpyZJbtvsCj9o10qvcypyk7ij9G6ZvEB6yYnRk&response-content-disposition=attachment%3B%20filename%3Dcfssl_linux-amd64&response-content-type=application%2Foctet-stream
Resolving release-assets.githubusercontent.com (release-assets.githubusercontent.com)... 185.199.109.133, 185.199.110.133, 185.199.111.133
Connecting to release-assets.githubusercontent.com (release-assets.githubusercontent.com)|185.199.109.133|:443... failed: Connection refused.
Connecting to release-assets.githubusercontent.com (release-assets.githubusercontent.com)|185.199.110.133|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 10376657 (9.9M) [application/octet-stream]
Saving to: ‘cfssl_linux-amd64’

100%[======================================>] 10,376,657  13.9KB/s   in 13m 33s

2025-07-26 12:32:55 (12.5 KB/s) - ‘cfssl_linux-amd64’ saved [10376657/10376657]

[root@localhost ~]# wget https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
--2025-07-26 12:35:07--  https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
Resolving pkg.cfssl.org (pkg.cfssl.org)... 104.18.17.120, 104.18.16.120, 2606:4700::6812:1078, ...
Connecting to pkg.cfssl.org (pkg.cfssl.org)|104.18.17.120|:443... connected.
HTTP request sent, awaiting response... 301 Moved Permanently
Location: https://github.com/cloudflare/cfssl/releases/download/1.2.0/cfssljson_linux-amd64 [following]
--2025-07-26 12:35:08--  https://github.com/cloudflare/cfssl/releases/download/1.2.0/cfssljson_linux-amd64
Resolving github.com (github.com)... 20.205.243.166
Connecting to github.com (github.com)|20.205.243.166|:443... connected.
HTTP request sent, awaiting response... 302 Found
Location: https://release-assets.githubusercontent.com/github-production-release-asset/21591001/8a86d880-9ebe-11eb-9d16-2fd0c4fe9f34?sp=r&sv=2018-11-09&sr=b&spr=https&se=2025-07-26T20%3A32%3A15Z&rscd=attachment%3B+filename%3Dcfssljson_linux-amd64&rsct=application%2Foctet-stream&skoid=96c2d410-5711-43a1-aedd-ab1947aa7ab0&sktid=398a6654-997b-47e9-b12b-9515b896b4de&skt=2025-07-26T19%3A32%3A03Z&ske=2025-07-26T20%3A32%3A15Z&sks=b&skv=2018-11-09&sig=t85KLTLoLzUZ7XUdKY0XhhG8X%2BuhJaXY8X6NLkGEN7o%3D&jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmVsZWFzZS1hc3NldHMuZ2l0aHVidXNlcmNvbnRlbnQuY29tIiwia2V5Ijoia2V5MSIsImV4cCI6MTc1MzU1ODgwOSwibmJmIjoxNzUzNTU4NTA5LCJwYXRoIjoicmVsZWFzZWFzc2V0cHJvZHVjdGlvbi5ibG9iLmNvcmUud2luZG93cy5uZXQifQ.xhPID-jmjN3jhVpgS0vPU0wPjermGBOKtrBQrt763JM&response-content-disposition=attachment%3B%20filename%3Dcfssljson_linux-amd64&response-content-type=application%2Foctet-stream [following]
--2025-07-26 12:35:09--  https://release-assets.githubusercontent.com/github-production-release-asset/21591001/8a86d880-9ebe-11eb-9d16-2fd0c4fe9f34?sp=r&sv=2018-11-09&sr=b&spr=https&se=2025-07-26T20%3A32%3A15Z&rscd=attachment%3B+filename%3Dcfssljson_linux-amd64&rsct=application%2Foctet-stream&skoid=96c2d410-5711-43a1-aedd-ab1947aa7ab0&sktid=398a6654-997b-47e9-b12b-9515b896b4de&skt=2025-07-26T19%3A32%3A03Z&ske=2025-07-26T20%3A32%3A15Z&sks=b&skv=2018-11-09&sig=t85KLTLoLzUZ7XUdKY0XhhG8X%2BuhJaXY8X6NLkGEN7o%3D&jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmVsZWFzZS1hc3NldHMuZ2l0aHVidXNlcmNvbnRlbnQuY29tIiwia2V5Ijoia2V5MSIsImV4cCI6MTc1MzU1ODgwOSwibmJmIjoxNzUzNTU4NTA5LCJwYXRoIjoicmVsZWFzZWFzc2V0cHJvZHVjdGlvbi5ibG9iLmNvcmUud2luZG93cy5uZXQifQ.xhPID-jmjN3jhVpgS0vPU0wPjermGBOKtrBQrt763JM&response-content-disposition=attachment%3B%20filename%3Dcfssljson_linux-amd64&response-content-type=application%2Foctet-stream
Resolving release-assets.githubusercontent.com (release-assets.githubusercontent.com)... 185.199.108.133, 185.199.110.133, 185.199.111.133
Connecting to release-assets.githubusercontent.com (release-assets.githubusercontent.com)|185.199.108.133|:443... failed: Connection refused.
Connecting to release-assets.githubusercontent.com (release-assets.githubusercontent.com)|185.199.110.133|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 2277873 (2.2M) [application/octet-stream]
Saving to: ‘cfssljson_linux-amd64’

100%[======================================>] 2,277,873   12.8KB/s   in 2m 39s 

2025-07-26 12:38:10 (14.0 KB/s) - ‘cfssljson_linux-amd64’ saved [2277873/2277873]
# 安装到系统路径
[root@localhost ~]# chmod +x cfssl_linux-amd64 cfssljson_linux-amd64
[root@localhost ~]# sudo mv cfssl_linux-amd64 /usr/local/bin/cfssl
[root@localhost ~]# sudo mv cfssljson_linux-amd64 /usr/local/bin/cfssljson
# 验证安装
[root@localhost ~]# cfssl version
Version: 1.2.0
Revision: dev
Runtime: go1.6
[root@localhost ~]# cfssljson -h
Usage of cfssljson:
  -bare
    	the response from CFSSL is not wrapped in the API standard response
  -f string
    	JSON input (default "-")
  -stdout
    	output the response instead of saving to a file
```

**重新生成证书**
```
# 确保CA配置文件存在
[root@localhost ~]# sudo cat <<EOF | sudo tee /etc/kubernetes/pki/ca-config.json
> {
>   "signing": {
>     "default": {
>       "expiry": "8760h"
>     },
>     "profiles": {
>       "kubernetes": {
>         "usages": ["signing", "key encipherment", "server auth", "client auth"],
>         "expiry": "8760h"
>       }
>     }
>   }
> }
> EOF
{
  "signing": {
    "default": {
      "expiry": "8760h"
    },
    "profiles": {
      "kubernetes": {
        "usages": ["signing", "key encipherment", "server auth", "client auth"],
        "expiry": "8760h"
      }
    }
  }
}
# 生成证书
[root@localhost ~]# sudo cfssl gencert \
>   -ca=/etc/kubernetes/pki/ca.crt \
>   -ca-key=/etc/kubernetes/pki/ca.key \
>   -config=/etc/kubernetes/pki/ca-config.json \
>   -hostname=localhost.localdomain,192.168.225.130 \
>   -profile=kubernetes \
>   /etc/kubernetes/pki/kubelet-csr.json | sudo cfssljson -bare /etc/kubernetes/pki/kubelet
sudo: cfssljson: command not foundsudo
: cfssl: command not found
```
在最后一步执行 **cfssl** 命令时仍然出现问题，尽管我们已经安装了这些工具。这可能是由于环境变量或路径问题导致的。
```
# 检查工具是否在正确路径
[root@localhost ~]# ls -l /usr/local/bin/cfssl*
-rwxr-xr-x. 1 root root 10376657 Dec  6  2021 /usr/local/bin/cfssl
-rwxr-xr-x. 1 root root  2277873 Dec  6  2021 /usr/local/bin/cfssljson
[root@localhost ~]# which cfssl
/usr/local/bin/cfssl
[root@localhost ~]# which cfssljson
/usr/local/bin/cfssljson
```
根据检查结果，**cfssl** 和 **cfssljson** 工具已正确安装在 /usr/local/bin 目录下，且位于系统 PATH 中。现在证书生成失败可能是由于环境变量或权限问题导致

```
# 编辑kubelet服务配置
sudo vi /usr/lib/systemd/system/kubelet.service.d/10-kubeadm.conf
# 在[Service]部分添加以下内容
[Service]
Environment="KUBELET_TLS_ARGS=--tls-cert-file=/etc/kubernetes/pki/kubelet.crt --tls-private-key-file=/etc/kubernetes/pki/kubelet.key"
# 修改 ExecStart行
ExecStart=/usr/bin/kubelet $KUBELET_KUBECONFIG_ARGS $KUBELET_CONFIG_ARGS $KUBELET_KUBEADM_ARGS $KUBELET_TLS_ARGS $KUBELET_EXTRA_ARGS
```
```
[root@localhost ~]# cat /usr/lib/systemd/system/kubelet.service.d/10-kubeadm.conf
# Note: This dropin only works with kubeadm and kubelet v1.11+
[Service]
Environment="KUBELET_KUBECONFIG_ARGS=--bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf"

Environment="KUBELET_CONFIG_ARGS=--config=/var/lib/kubelet/config.yaml"

# This is a file that "kubeadm init" and "kubeadm join" generates at runtime, populating the KUBELET_KUBEADM_ARGS variable dynamically

Environment="KUBELET_TLS_ARGS=--tls-cert-file=/etc/kubernetes/pki/kubelet.crt --tls-private-key-file=/etc/kubernetes/pki/kubelet.key"

EnvironmentFile=-/var/lib/kubelet/kubeadm-flags.env

# This is a file that the user can use for overrides of the kubelet args as a last resort. Preferably, the user should use
# the .NodeRegistration.KubeletExtraArgs object in the configuration files instead. KUBELET_EXTRA_ARGS should be sourced from this file.

EnvironmentFile=-/etc/sysconfig/kubelet
ExecStart=
ExecStart=/usr/bin/kubelet $KUBELET_KUBECONFIG_ARGS $KUBELET_CONFIG_ARGS $KUBELET_KUBEADM_ARGS $KUBELET_TLS_ARGS $KUBELET_EXTRA_ARGS

[root@localhost ~]# sudo systemctl daemon-reload
[root@localhost ~]# sudo systemctl restart kubelet
# 验证参数是否生效
[root@localhost ~]# ps -ef | grep kubelet | grep tls
root      61362  61322  4 11:49 ?        00:03:10 kube-apiserver --advertise-address=192.168.225.130 --allow-privileged=true --authorization-mode=Node,RBAC --client-ca-file=/etc/kubernetes/pki/ca.crt --enable-admission-plugins=NodeRestriction --enable-bootstrap-token-auth=true --etcd-cafile=/etc/kubernetes/pki/etcd/ca.crt --etcd-certfile=/etc/kubernetes/pki/apiserver-etcd-client.crt --etcd-keyfile=/etc/kubernetes/pki/apiserver-etcd-client.key --etcd-servers=https://127.0.0.1:2379 --kubelet-client-certificate=/etc/kubernetes/pki/apiserver-kubelet-client.crt --kubelet-client-key=/etc/kubernetes/pki/apiserver-kubelet-client.key --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname --proxy-client-cert-file=/etc/kubernetes/pki/front-proxy-client.crt --proxy-client-key-file=/etc/kubernetes/pki/front-proxy-client.key --requestheader-allowed-names=front-proxy-client --requestheader-client-ca-file=/etc/kubernetes/pki/front-proxy-ca.crt --requestheader-extra-headers-prefix=X-Remote-Extra- --requestheader-group-headers=X-Remote-Group --requestheader-username-headers=X-Remote-User --secure-port=6443 --service-account-issuer=https://kubernetes.default.svc.cluster.local --service-account-key-file=/etc/kubernetes/pki/sa.pub --service-account-signing-key-file=/etc/kubernetes/pki/sa.key --service-cluster-ip-range=10.96.0.0/12 --tls-cert-file=/etc/kubernetes/pki/apiserver.crt --tls-private-key-file=/etc/kubernetes/pki/apiserver.key
root      90502      1  8 12:57 ?        00:00:00 /usr/bin/kubelet --bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf --config=/var/lib/kubelet/config.yaml --network-plugin=cni --pod-infra-container-image=registry.aliyuncs.com/google_containers/pause:3.6 --tls-cert-file=/etc/kubernetes/pki/kubelet.crt --tls-private-key-file=/etc/kubernetes/pki/kubelet.key
```
```
# 检查服务状态
[root@localhost ~]# sudo journalctl -u kubelet -n 20 --no-pager
-- Logs begin at Sat 2025-07-26 09:59:16 PDT, end at Sat 2025-07-26 12:57:50 PDT. --
Jul 26 12:57:32 localhost.localdomain kubelet[90502]: I0726 12:57:32.844377   90502 reconciler.go:221] "operationExecutor.VerifyControllerAttachedVolume started for volume \"flannel-cfg\" (UniqueName: \"kubernetes.io/configmap/5cb41e8e-4a2e-4222-83aa-2c832d8f54d6-flannel-cfg\") pod \"kube-flannel-ds-kkjpb\" (UID: \"5cb41e8e-4a2e-4222-83aa-2c832d8f54d6\") " pod="kube-flannel/kube-flannel-ds-kkjpb"
Jul 26 12:57:32 localhost.localdomain kubelet[90502]: I0726 12:57:32.844392   90502 reconciler.go:221] "operationExecutor.VerifyControllerAttachedVolume started for volume \"kube-api-access-rjc4w\" (UniqueName: \"kubernetes.io/projected/5cb41e8e-4a2e-4222-83aa-2c832d8f54d6-kube-api-access-rjc4w\") pod \"kube-flannel-ds-kkjpb\" (UID: \"5cb41e8e-4a2e-4222-83aa-2c832d8f54d6\") " pod="kube-flannel/kube-flannel-ds-kkjpb"
Jul 26 12:57:32 localhost.localdomain kubelet[90502]: I0726 12:57:32.844405   90502 reconciler.go:221] "operationExecutor.VerifyControllerAttachedVolume started for volume \"config-volume\" (UniqueName: \"kubernetes.io/configmap/c47a199f-24a9-48c2-8721-91c83c77c3ba-config-volume\") pod \"coredns-6d8c4cb4d-8l6hx\" (UID: \"c47a199f-24a9-48c2-8721-91c83c77c3ba\") " pod="kube-system/coredns-6d8c4cb4d-8l6hx"
Jul 26 12:57:32 localhost.localdomain kubelet[90502]: I0726 12:57:32.844418   90502 reconciler.go:221] "operationExecutor.VerifyControllerAttachedVolume started for volume \"kube-api-access-5qkvm\" (UniqueName: \"kubernetes.io/projected/c47a199f-24a9-48c2-8721-91c83c77c3ba-kube-api-access-5qkvm\") pod \"coredns-6d8c4cb4d-8l6hx\" (UID: \"c47a199f-24a9-48c2-8721-91c83c77c3ba\") " pod="kube-system/coredns-6d8c4cb4d-8l6hx"
Jul 26 12:57:32 localhost.localdomain kubelet[90502]: I0726 12:57:32.844430   90502 reconciler.go:221] "operationExecutor.VerifyControllerAttachedVolume started for volume \"lib-modules\" (UniqueName: \"kubernetes.io/host-path/40b81ba1-25e3-4cdd-9388-ea2b1678291f-lib-modules\") pod \"kube-proxy-khcdl\" (UID: \"40b81ba1-25e3-4cdd-9388-ea2b1678291f\") " pod="kube-system/kube-proxy-khcdl"
Jul 26 12:57:32 localhost.localdomain kubelet[90502]: I0726 12:57:32.844445   90502 reconciler.go:221] "operationExecutor.VerifyControllerAttachedVolume started for volume \"kube-api-access-fq9cp\" (UniqueName: \"kubernetes.io/projected/40b81ba1-25e3-4cdd-9388-ea2b1678291f-kube-api-access-fq9cp\") pod \"kube-proxy-khcdl\" (UID: \"40b81ba1-25e3-4cdd-9388-ea2b1678291f\") " pod="kube-system/kube-proxy-khcdl"
Jul 26 12:57:32 localhost.localdomain kubelet[90502]: I0726 12:57:32.844456   90502 reconciler.go:221] "operationExecutor.VerifyControllerAttachedVolume started for volume \"xtables-lock\" (UniqueName: \"kubernetes.io/host-path/5cb41e8e-4a2e-4222-83aa-2c832d8f54d6-xtables-lock\") pod \"kube-flannel-ds-kkjpb\" (UID: \"5cb41e8e-4a2e-4222-83aa-2c832d8f54d6\") " pod="kube-flannel/kube-flannel-ds-kkjpb"
Jul 26 12:57:32 localhost.localdomain kubelet[90502]: I0726 12:57:32.844469   90502 reconciler.go:221] "operationExecutor.VerifyControllerAttachedVolume started for volume \"kube-api-access-j7tcs\" (UniqueName: \"kubernetes.io/projected/889283d4-bde7-4331-bc62-e8094cc06955-kube-api-access-j7tcs\") pod \"coredns-6d8c4cb4d-79m2s\" (UID: \"889283d4-bde7-4331-bc62-e8094cc06955\") " pod="kube-system/coredns-6d8c4cb4d-79m2s"
Jul 26 12:57:32 localhost.localdomain kubelet[90502]: I0726 12:57:32.844479   90502 reconciler.go:221] "operationExecutor.VerifyControllerAttachedVolume started for volume \"run\" (UniqueName: \"kubernetes.io/host-path/5cb41e8e-4a2e-4222-83aa-2c832d8f54d6-run\") pod \"kube-flannel-ds-kkjpb\" (UID: \"5cb41e8e-4a2e-4222-83aa-2c832d8f54d6\") " pod="kube-flannel/kube-flannel-ds-kkjpb"
Jul 26 12:57:32 localhost.localdomain kubelet[90502]: I0726 12:57:32.844571   90502 reconciler.go:221] "operationExecutor.VerifyControllerAttachedVolume started for volume \"kube-proxy\" (UniqueName: \"kubernetes.io/configmap/40b81ba1-25e3-4cdd-9388-ea2b1678291f-kube-proxy\") pod \"kube-proxy-khcdl\" (UID: \"40b81ba1-25e3-4cdd-9388-ea2b1678291f\") " pod="kube-system/kube-proxy-khcdl"
Jul 26 12:57:32 localhost.localdomain kubelet[90502]: I0726 12:57:32.844583   90502 reconciler.go:221] "operationExecutor.VerifyControllerAttachedVolume started for volume \"xtables-lock\" (UniqueName: \"kubernetes.io/host-path/40b81ba1-25e3-4cdd-9388-ea2b1678291f-xtables-lock\") pod \"kube-proxy-khcdl\" (UID: \"40b81ba1-25e3-4cdd-9388-ea2b1678291f\") " pod="kube-system/kube-proxy-khcdl"
Jul 26 12:57:32 localhost.localdomain kubelet[90502]: I0726 12:57:32.844594   90502 reconciler.go:221] "operationExecutor.VerifyControllerAttachedVolume started for volume \"cni-plugin\" (UniqueName: \"kubernetes.io/host-path/5cb41e8e-4a2e-4222-83aa-2c832d8f54d6-cni-plugin\") pod \"kube-flannel-ds-kkjpb\" (UID: \"5cb41e8e-4a2e-4222-83aa-2c832d8f54d6\") " pod="kube-flannel/kube-flannel-ds-kkjpb"
Jul 26 12:57:32 localhost.localdomain kubelet[90502]: I0726 12:57:32.844609   90502 reconciler.go:157] "Reconciler: start to sync state"
Jul 26 12:57:33 localhost.localdomain kubelet[90502]: E0726 12:57:33.946405   90502 configmap.go:200] Couldn't get configMap kube-system/kube-proxy: failed to sync configmap cache: timed out waiting for the condition
Jul 26 12:57:33 localhost.localdomain kubelet[90502]: E0726 12:57:33.946627   90502 nestedpendingoperations.go:335] Operation for "{volumeName:kubernetes.io/configmap/40b81ba1-25e3-4cdd-9388-ea2b1678291f-kube-proxy podName:40b81ba1-25e3-4cdd-9388-ea2b1678291f nodeName:}" failed. No retries permitted until 2025-07-26 12:57:34.446574529 -0700 PDT m=+3.014522913 (durationBeforeRetry 500ms). Error: MountVolume.SetUp failed for volume "kube-proxy" (UniqueName: "kubernetes.io/configmap/40b81ba1-25e3-4cdd-9388-ea2b1678291f-kube-proxy") pod "kube-proxy-khcdl" (UID: "40b81ba1-25e3-4cdd-9388-ea2b1678291f") : failed to sync configmap cache: timed out waiting for the condition
Jul 26 12:57:34 localhost.localdomain kubelet[90502]: I0726 12:57:34.006771   90502 request.go:665] Waited for 1.195972338s due to client-side throttling, not priority and fairness, request: GET:https://192.168.225.130:6443/api/v1/namespaces/kube-system/configmaps?fieldSelector=metadata.name%3Dkube-proxy&limit=500&resourceVersion=0
Jul 26 12:57:34 localhost.localdomain kubelet[90502]: E0726 12:57:34.210828   90502 kubelet.go:1742] "Failed creating a mirror pod for" err="pods \"etcd-localhost.localdomain\" already exists" pod="kube-system/etcd-localhost.localdomain"
Jul 26 12:57:34 localhost.localdomain kubelet[90502]: E0726 12:57:34.411224   90502 kubelet.go:1742] "Failed creating a mirror pod for" err="pods \"kube-scheduler-localhost.localdomain\" already exists" pod="kube-system/kube-scheduler-localhost.localdomain"
Jul 26 12:57:34 localhost.localdomain kubelet[90502]: E0726 12:57:34.610375   90502 kubelet.go:1742] "Failed creating a mirror pod for" err="pods \"kube-controller-manager-localhost.localdomain\" already exists" pod="kube-system/kube-controller-manager-localhost.localdomain"
Jul 26 12:57:34 localhost.localdomain kubelet[90502]: E0726 12:57:34.811570   90502 kubelet.go:1742] "Failed creating a mirror pod for" err="pods \"kube-apiserver-localhost.localdomain\" already exists" pod="kube-system/kube-apiserver-localhost.localdomain"
# 证书文件验证
[root@localhost ~]# sudo ls -l /etc/kubernetes/pki/kubelet.crt /etc/kubernetes/pki/kubelet.key
-rw-------. 1 root root 1294 Jul 26 12:44 /etc/kubernetes/pki/kubelet.crt
-rw-------. 1 root root 1675 Jul 26 12:44 /etc/kubernetes/pki/kubelet.key
```
根据检查结果，kubelet 的 TLS 证书参数已成功添加并生效（从 **ps -ef** 输出中可见 TLS 参数），但集群仍存在以下关键问题：
1、**ConfigMap**同步失败（核心问题）：
Couldn't get configMap kube-system/kube-proxy: failed to sync configmap cache
这表明kubelet虽然能启动，但与API  Server的通信仍有问题

2、**证书验证**
* TLS证书文件已存在且权限正确（600）
* kubelet进程已加载TLS参数

**1、修复API Server连接问题**
```
# 检查 API Server 证书
[root@localhost ~]# openssl x509 -in /etc/kubernetes/pki/apiserver.crt -noout -text | grep -A 2 Validity
        Validity
            Not Before: Jul 26 18:48:59 2025 GMT
            Not After : Jul 26 18:48:59 2026 GMT

# 检查 kubelet 与 API Server 的连接
[root@localhost ~]# curl -k --cert /etc/kubernetes/pki/kubelet.crt --key /etc/kubernetes/pki/kubelet.key https://localhost:6443/healthz
ok
# 应返回 "ok"
```
**当前状态确认**
✅ API Server 证书有效（有效期 1 年）

✅ kubelet 能通过 TLS 证书访问 API Server（**/healthz** 返回 ok）

✅ kubelet 证书文件权限正确
**网络修复方案**
**1、强制重建Flannel网络组件**
```
# 删除现有Flannel网络组件
[root@localhost ~]# kubectl delete -f https://raw.githubusercontent.com/coreoslannel/master/Documentation/kube-flannel.yml --ignore-not-found
namespace "kube-flannel" deleted
clusterrole.rbac.authorization.k8s.io "flannel" deleted
clusterrolebinding.rbac.authorization.k8s.io "flannel" deleted
serviceaccount "flannel" deleted
configmap "kube-flannel-cfg" deleted
daemonset.apps "kube-flannel-ds" deleted
# 清理残留网络配置
[root@localhost ~]# sudo rm -rf /etc/cni/net.d/*
[root@localhost ~]# sudo ip link delete cni0
[root@localhost ~]# sudo ip link delete flannel.1
# 重新部署Flannel （使用国内镜像源）
[root@localhost ~]# curl -sSL https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml | sed 's/quay.io\/coreos\/flannel/registry.cn-hangzhou.aliyuncs.com\/google_containers\/flannel/g' | kubectl apply -f -
namespace/kube-flannel created
clusterrole.rbac.authorization.k8s.io/flannel created
clusterrolebinding.rbac.authorization.k8s.io/flannel created
serviceaccount/flannel created
configmap/kube-flannel-cfg created
daemonset.apps/kube-flannel-ds created
```
**2、检查并修复CNI配置**
```
#检查CNI配置文件
[root@localhost ~]# ls -l /etc/cni/net.d/
total 4
-rw-r--r--. 1 root root 292 Jul 26 13:02 10-flannel.conflist
```
**3、重启关键服务**
```
[root@localhost ~]# sudo systemctl restart kubelet
[root@localhost ~]# sudo systemctl restart docker
```
**4、验证网络状态**
```
[root@localhost ~]# kubectl logs -n kube-flannel $(kubectl get pods -n kube-flannel -o name) -c kube-flannel
I0726 20:04:04.709203       1 main.go:213] CLI flags config: {etcdEndpoints:http://127.0.0.1:4001,http://127.0.0.1:2379 etcdPrefix:/coreos.com/network etcdKeyfile: etcdCertfile: etcdCAFile: etcdUsername: etcdPassword: version:false kubeSubnetMgr:true kubeApiUrl: kubeAnnotationPrefix:flannel.alpha.coreos.com kubeConfigFile: iface:[] ifaceRegex:[] ipMasq:true ifaceCanReach: subnetFile:/run/flannel/subnet.env publicIP: publicIPv6: subnetLeaseRenewMargin:60 healthzIP:0.0.0.0 healthzPort:0 iptablesResyncSeconds:5 iptablesForwardRules:true blackholeRoute:false netConfPath:/etc/kube-flannel/net-conf.json setNodeNetworkUnavailable:true}
W0726 20:04:04.709566       1 client_config.go:659] Neither --kubeconfig nor --master was specified.  Using the inClusterConfig.  This might not work.
E0726 20:04:06.743430       1 main.go:235] Failed to create SubnetManager: error retrieving pod spec for 'kube-flannel/kube-flannel-ds-4mftf': pods "kube-flannel-ds-4mftf" is forbidden: User "system:serviceaccount:kube-flannel:flannel" cannot get resource "pods" in API group "" in the namespace "kube-flannel"
# 检查节点网络配置
[root@localhost ~]# ip addr show cni0
11: cni0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP group default qlen 1000
    link/ether ba:3a:ed:2e:29:42 brd ff:ff:ff:ff:ff:ff
    inet 10.244.0.1/24 brd 10.244.0.255 scope global cni0
       valid_lft forever preferred_lft forever
    inet6 fe80::b83a:edff:fe2e:2942/64 scope link 
       valid_lft forever preferred_lft forever
[root@localhost ~]# ip route show
default via 192.168.225.2 dev ens33 proto dhcp metric 100 
10.244.0.0/24 dev cni0 proto kernel scope link src 10.244.0.1 
172.17.0.0/16 dev docker0 proto kernel scope link src 172.17.0.1 
192.168.122.0/24 dev virbr0 proto kernel scope link src 192.168.122.1 
192.168.225.0/24 dev ens33 proto kernel scope link src 192.168.225.130 metric 100 
```
**当前状态**
* CNI网络配置已生成（10-flannel.conflist)
* cni0网桥已创建并分配了ip地址10.244.0.1/24
* 路由表显示Flannel子网10.244.0.0/24

**遇到的问题**
Flannel Pod日志显示权限错误：
User "system:serviceaccount:kube-flannel:flannel" cannot get resource "pods" in API group "" in the namespace "kube-flannel"
这表明Flannel Pod的服务账户没有足够的权限获取Pod信息

**解决方案**
需要为 Flannel服务账户添加必要的RBAC权限：
1、创建一个ClusterRole绑定，允许Flannel服务账户获取Pod信息：
```
[root@localhost ~]# kubectl create clusterrolebinding flannel-pod-access \
>   --clusterrole=view \
>   --serviceaccount=kube-flannel:flannel
clusterrolebinding.rbac.authorization.k8s.io/flannel-pod-access created
```
2、删除现有的Flannel  Pod以触发重建：
```
[root@localhost ~]# kubectl delete pods -n kube-flannel --all
pod "kube-flannel-ds-4mftf" deleted
```
**验证**
检查新的Flannel Pod日志是否不再显示权限错误：
```
[root@localhost ~]# kubectl logs -n kube-flannel $(kubectl get pods -n kube-flannel -o name) -c kube-flannel
I0726 20:06:46.054702       1 main.go:213] CLI flags config: {etcdEndpoints:http://127.0.0.1:4001,http://127.0.0.1:2379 etcdPrefix:/coreos.com/network etcdKeyfile: etcdCertfile: etcdCAFile: etcdUsername: etcdPassword: version:false kubeSubnetMgr:true kubeApiUrl: kubeAnnotationPrefix:flannel.alpha.coreos.com kubeConfigFile: iface:[] ifaceRegex:[] ipMasq:true ifaceCanReach: subnetFile:/run/flannel/subnet.env publicIP: publicIPv6: subnetLeaseRenewMargin:60 healthzIP:0.0.0.0 healthzPort:0 iptablesResyncSeconds:5 iptablesForwardRules:true blackholeRoute:false netConfPath:/etc/kube-flannel/net-conf.json setNodeNetworkUnavailable:true}
W0726 20:06:46.054954       1 client_config.go:659] Neither --kubeconfig nor --master was specified.  Using the inClusterConfig.  This might not work.
I0726 20:06:46.079482       1 kube.go:139] Waiting 10m0s for node controller to sync
I0726 20:06:46.079697       1 kube.go:537] Starting kube subnet manager
I0726 20:06:46.082763       1 kube.go:558] Creating the node lease for IPv4. This is the n.Spec.PodCIDRs: [10.244.0.0/24]
I0726 20:06:47.079854       1 kube.go:163] Node controller sync successful
I0726 20:06:47.079897       1 main.go:239] Created subnet manager: Kubernetes Subnet Manager - localhost.localdomain
I0726 20:06:47.079902       1 main.go:242] Installing signal handlers
I0726 20:06:47.079977       1 main.go:519] Found network config - Backend type: vxlan
I0726 20:06:47.084018       1 kube.go:737] List of node(localhost.localdomain) annotations: map[string]string{"flannel.alpha.coreos.com/backend-data":"{\"VNI\":1,\"VtepMAC\":\"32:08:fc:60:1c:4d\"}", "flannel.alpha.coreos.com/backend-type":"vxlan", "flannel.alpha.coreos.com/kube-subnet-manager":"true", "flannel.alpha.coreos.com/public-ip":"192.168.225.130", "kubeadm.alpha.kubernetes.io/cri-socket":"/var/run/dockershim.sock", "node.alpha.kubernetes.io/ttl":"0", "volumes.kubernetes.io/controller-managed-attach-detach":"true"}
I0726 20:06:47.084085       1 match.go:211] Determining IP address of default interface
I0726 20:06:47.085558       1 match.go:264] Using interface with name ens33 and address 192.168.225.130
I0726 20:06:47.085593       1 match.go:286] Defaulting external address to interface address (192.168.225.130)
I0726 20:06:47.085667       1 vxlan.go:141] VXLAN config: VNI=1 Port=0 GBP=false Learning=false DirectRouting=false
I0726 20:06:47.088160       1 kube.go:704] List of node(localhost.localdomain) annotations: map[string]string{"flannel.alpha.coreos.com/backend-data":"{\"VNI\":1,\"VtepMAC\":\"32:08:fc:60:1c:4d\"}", "flannel.alpha.coreos.com/backend-type":"vxlan", "flannel.alpha.coreos.com/kube-subnet-manager":"true", "flannel.alpha.coreos.com/public-ip":"192.168.225.130", "kubeadm.alpha.kubernetes.io/cri-socket":"/var/run/dockershim.sock", "node.alpha.kubernetes.io/ttl":"0", "volumes.kubernetes.io/controller-managed-attach-detach":"true"}
I0726 20:06:47.088220       1 vxlan.go:155] Interface flannel.1 mac address set to: 32:08:fc:60:1c:4d
I0726 20:06:47.089022       1 main.go:375] Cleaning-up unused traffic manager rules
I0726 20:06:47.089058       1 nftables.go:278] Cleaning-up nftables rules...
I0726 20:06:47.129302       1 iptables.go:50] Starting flannel in iptables mode...
W0726 20:06:47.129829       1 main.go:608] no subnet found for key: FLANNEL_IPV6_NETWORK in file: /run/flannel/subnet.env
W0726 20:06:47.129880       1 main.go:608] no subnet found for key: FLANNEL_IPV6_SUBNET in file: /run/flannel/subnet.env
I0726 20:06:47.129892       1 iptables.go:110] Setting up masking rules
I0726 20:06:47.144119       1 iptables.go:211] Changing default FORWARD chain policy to ACCEPT
I0726 20:06:47.147933       1 main.go:463] Wrote subnet file to /run/flannel/subnet.env
I0726 20:06:47.147985       1 main.go:467] Running backend.
I0726 20:06:47.148436       1 vxlan_network.go:65] watching for new subnet leases
I0726 20:06:47.157030       1 main.go:488] Waiting for all goroutines to exit
I0726 20:06:47.164667       1 iptables.go:357] bootstrap done
I0726 20:06:47.176400       1 iptables.go:357] bootstrap done
```
**结论**
Flannel 网络插件已成功部署并正常运行，节点上的 Pod 现在应该能够获得 IP 地址并正常通信。

**网络验证**
1、检查Flannel Pod状态
应该显示kube-flannel-ds-xxxx Pod状态为Running 
```
[root@localhost ~]# kubectl get pods -n kube-flannel
NAME                    READY   STATUS    RESTARTS   AGE
kube-flannel-ds-z9xdg   1/1     Running   0          48s
```
2、检查节点网络注释
应该包含Flannel相关的网络配置信息
```
[root@localhost ~]# kubectl describe node localhost.localdomain | grep -A 10 Annotations
Annotations:        flannel.alpha.coreos.com/backend-data: {"VNI":1,"VtepMAC":"32:08:fc:60:1c:4d"}
                    flannel.alpha.coreos.com/backend-type: vxlan
                    flannel.alpha.coreos.com/kube-subnet-manager: true
                    flannel.alpha.coreos.com/public-ip: 192.168.225.130
                    kubeadm.alpha.kubernetes.io/cri-socket: /var/run/dockershim.sock
                    node.alpha.kubernetes.io/ttl: 0
                    volumes.kubernetes.io/controller-managed-attach-detach: true
CreationTimestamp:  Sat, 26 Jul 2025 11:49:08 -0700
Taints:             node-role.kubernetes.io/master:NoSchedule
Unschedulable:      false
Lease:
```
**节点网络注解确认**
节点注解中包含完整的 Flannel 配置：
后端类型为 VXLAN
VXLAN 网络标识符 VNI=1
VTEP MAC 地址已设置
节点公网 IP 为 192.168.225.130
启用了 kube-subnet-manager
当前所有迹象表明 Flannel 网络插件已正确安装并运行正常!
