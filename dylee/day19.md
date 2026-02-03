# Cluster Maintenance

# OS Upgrade

## 1. node를 down 시킨 후 → upgrade

- node가 down되면 Pods가 접근하지 못하고 → 그 후 node를 upgrade 시킨다 → down 시킨 후 upgrade를 해야 할 상황이라면 Pod에 Replicas를 설정하자
    - replicaset으로 Pod를 배포했다면 → pod가 termination 된 후 → 다른 node에 자동으로 재배치
    - 하지만 replicaset으로 pod를 배포하지 않았다면 → pod eviction
        - eviction time = 5분이라 가정하면, 5분 전에 node 복구 → pod 복구, 5분 후 node 복구 → pod eviction
        - eviction time 후 복구된 node에는 아무 Pod도 없는 상태

## 2. drain으로 node의 Workload를 소모시켜 다른 노드로 옮기기

```bash
kubectl drain node-1
# drain mode로 어떤 Pod도 해당 node에 scheduling 되지 않도록 하고
# 노드에 있는 pod들 eviction 수행
# 컨트롤러가 관리하는 pod(Deployment/ReplicaSet/StatefulSet/DaemonSet 등 -> 다른 노드에 재생성
# drain mode를 끄지 전 까지는 다른 노드 스케줄링 x

```

```bash
kubectl uncordon node-1
# 새 Pod가 node-1에 스케줄링 x
# 이미 실행 중인 Pod -> 그대로 유지
kubectl cordon node-2
# 노드를 다시 schedulable 상태로 복구 -> 새 pod 스케줄링 가능
```

# Lab Solution - OS Upgrade

```bash
controlplane ~ ➜  alias k=kubectl

controlplane ~ ➜  k get node
NAME           STATUS   ROLES           AGE   VERSION
controlplane   Ready    control-plane   12m   v1.34.0
node01         Ready    <none>          11m   v1.34.0

controlplane ~ ➜  k get deploy
NAME   READY   UP-TO-DATE   AVAILABLE   AGE
blue   3/3     3            3           16s

controlplane ~ ➜  k get nodes
NAME           STATUS   ROLES           AGE   VERSION
controlplane   Ready    control-plane   14m   v1.34.0
node01         Ready    <none>          14m   v1.34.0

controlplane ~ ➜  k drain node01
node/node01 cordoned
error: unable to drain node "node01" due to error: cannot delete DaemonSet-managed Pods (use --ignore-daemonsets to ignore): kube-flannel/kube-flannel-ds-pgbn8, kube-system/kube-proxy-wd2s8, continuing command...
There are pending nodes to be drained:
 node01
cannot delete DaemonSet-managed Pods (use --ignore-daemonsets to ignore): kube-flannel/kube-flannel-ds-pgbn8, kube-system/kube-proxy-wd2s8

controlplane ~ ✖ k drain node01 --ignore-daemonsets 
node/node01 already cordoned
Warning: ignoring DaemonSet-managed Pods: kube-flannel/kube-flannel-ds-pgbn8, kube-system/kube-proxy-wd2s8
evicting pod default/blue-759779556-bbh5c
evicting pod default/blue-759779556-522px
pod/blue-759779556-bbh5c evicted
pod/blue-759779556-522px evicted
node/node01 drained

controlplane ~ ➜  k get nodes
NAME           STATUS                     ROLES           AGE   VERSION
controlplane   Ready                      control-plane   18m   v1.34.0
node01         Ready,SchedulingDisabled   <none>          17m   v1.34.0

controlplane ~ ➜  k get deployments
NAME   READY   UP-TO-DATE   AVAILABLE   AGE
blue   3/3     3            3           6m17s

controlplane ~ ➜  k get pods -o wide
NAME                   READY   STATUS    RESTARTS   AGE     IP           NODE           NOMINATED NODE   READINESS GATES
blue-759779556-brsxq   1/1     Running   0          105s    172.17.0.5   controlplane   <none>           <none>
blue-759779556-hfds9   1/1     Running   0          105s    172.17.0.6   controlplane   <none>           <none>
blue-759779556-ppgvn   1/1     Running   0          6m47s   172.17.0.4   controlplane   <none>           <none>

controlplane ~ ➜  k uncordon node01
node/node01 uncordoned

controlplane ~ ➜  k get nodes
NAME           STATUS   ROLES           AGE   VERSION
controlplane   Ready    control-plane   20m   v1.34.0
node01         Ready    <none>          20m   v1.34.0

controlplane ~ ➜  k get pods -o node01
error: unable to match a printer suitable for the output format "node01", allowed formats are: custom-columns,custom-columns-file,go-template,go-template-file,json,jsonpath,jsonpath-as-json,jsonpath-file,name,template,templatefile,wide,yaml

controlplane ~ ✖ k et pdos -o wide
error: unknown command "et" for "kubectl"

Did you mean this?
        set
        get
        edit
        cp

controlplane ~ ✖ k get pods -o wide
NAME                   READY   STATUS    RESTARTS   AGE     IP           NODE           NOMINATED NODE   READINESS GATES
blue-759779556-brsxq   1/1     Running   0          3m58s   172.17.0.5   controlplane   <none>           <none>
blue-759779556-hfds9   1/1     Running   0          3m58s   172.17.0.6   controlplane   <none>           <none>
blue-759779556-ppgvn   1/1     Running   0          9m      172.17.0.4   controlplane   <none>           <none>

controlplane ~ ➜  k describe node controlplane
Name:               controlplane
Roles:              control-plane
Labels:             beta.kubernetes.io/arch=amd64
                    beta.kubernetes.io/os=linux
                    kubernetes.io/arch=amd64
                    kubernetes.io/hostname=controlplane
                    kubernetes.io/os=linux
                    node-role.kubernetes.io/control-plane=
                    node.kubernetes.io/exclude-from-external-load-balancers=
Annotations:        flannel.alpha.coreos.com/backend-data: {"VNI":1,"VtepMAC":"32:d6:0b:e2:13:10"}
                    flannel.alpha.coreos.com/backend-type: vxlan
                    flannel.alpha.coreos.com/kube-subnet-manager: true
                    flannel.alpha.coreos.com/public-ip: 10.244.8.150
                    node.alpha.kubernetes.io/ttl: 0
                    volumes.kubernetes.io/controller-managed-attach-detach: true
CreationTimestamp:  Tue, 03 Feb 2026 09:59:02 +0000
Taints:             <none>
Unschedulable:      false
Lease:
  HolderIdentity:  controlplane
  AcquireTime:     <unset>
  RenewTime:       Tue, 03 Feb 2026 10:22:23 +0000
Conditions:
  Type                 Status  LastHeartbeatTime                 LastTransitionTime                Reason                       Message
  ----                 ------  -----------------                 ------------------                ------                       -------
  NetworkUnavailable   False   Tue, 03 Feb 2026 09:59:14 +0000   Tue, 03 Feb 2026 09:59:14 +0000   FlannelIsUp                  Flannel is running on this node
  MemoryPressure       False   Tue, 03 Feb 2026 10:17:44 +0000   Tue, 03 Feb 2026 09:59:00 +0000   KubeletHasSufficientMemory   kubelet has sufficient memory available
  DiskPressure         False   Tue, 03 Feb 2026 10:17:44 +0000   Tue, 03 Feb 2026 09:59:00 +0000   KubeletHasNoDiskPressure     kubelet has no disk pressure
  PIDPressure          False   Tue, 03 Feb 2026 10:17:44 +0000   Tue, 03 Feb 2026 09:59:00 +0000   KubeletHasSufficientPID      kubelet has sufficient PID available
  Ready                True    Tue, 03 Feb 2026 10:17:44 +0000   Tue, 03 Feb 2026 09:59:12 +0000   KubeletReady                 kubelet is posting ready status
Addresses:
  InternalIP:  10.244.8.150
  Hostname:    controlplane
Capacity:
  cpu:                16
  ephemeral-storage:  457717264Ki
  hugepages-1Gi:      0
  hugepages-2Mi:      0
  memory:             64932560Ki
  pods:               110
Allocatable:
  cpu:                16
  ephemeral-storage:  421832229804
  hugepages-1Gi:      0
  hugepages-2Mi:      0
  memory:             64830160Ki
  pods:               110
System Info:
  Machine ID:                 0cbdd9d5c1884787a1b7c5a9cada9a4e
  System UUID:                dfa5b8be-58dc-11ef-a2dd-f7aae391efc8
  Boot ID:                    03a0943d-af3c-4bea-850e-c44890767670
  Kernel Version:             6.8.0-90-generic
  OS Image:                   Ubuntu 22.04.5 LTS
  Operating System:           linux
  Architecture:               amd64
  Container Runtime Version:  containerd://1.6.26
  Kubelet Version:            v1.34.0
  Kube-Proxy Version:         
PodCIDR:                      172.17.0.0/24
PodCIDRs:                     172.17.0.0/24
Non-terminated Pods:          (11 in total)
  Namespace                   Name                                    CPU Requests  CPU Limits  Memory Requests  Memory Limits  Age
  ---------                   ----                                    ------------  ----------  ---------------  -------------  ---
  default                     blue-759779556-brsxq                    0 (0%)        0 (0%)      0 (0%)           0 (0%)         5m58s
  default                     blue-759779556-hfds9                    0 (0%)        0 (0%)      0 (0%)           0 (0%)         5m58s
  default                     blue-759779556-ppgvn                    0 (0%)        0 (0%)      0 (0%)           0 (0%)         11m
  kube-flannel                kube-flannel-ds-qf7mk                   100m (0%)     0 (0%)      50Mi (0%)        0 (0%)         23m
  kube-system                 coredns-6678bcd974-wjln6                100m (0%)     0 (0%)      70Mi (0%)        170Mi (0%)     23m
  kube-system                 coredns-6678bcd974-wtt7g                100m (0%)     0 (0%)      70Mi (0%)        170Mi (0%)     23m
  kube-system                 etcd-controlplane                       100m (0%)     0 (0%)      100Mi (0%)       0 (0%)         23m
  kube-system                 kube-apiserver-controlplane             250m (1%)     0 (0%)      0 (0%)           0 (0%)         23m
  kube-system                 kube-controller-manager-controlplane    200m (1%)     0 (0%)      0 (0%)           0 (0%)         23m
  kube-system                 kube-proxy-6lrtr                        0 (0%)        0 (0%)      0 (0%)           0 (0%)         23m
  kube-system                 kube-scheduler-controlplane             100m (0%)     0 (0%)      0 (0%)           0 (0%)         23m
Allocated resources:
  (Total limits may be over 100 percent, i.e., overcommitted.)
  Resource           Requests    Limits
  --------           --------    ------
  cpu                950m (5%)   0 (0%)
  memory             290Mi (0%)  340Mi (0%)
  ephemeral-storage  0 (0%)      0 (0%)
  hugepages-1Gi      0 (0%)      0 (0%)
  hugepages-2Mi      0 (0%)      0 (0%)
Events:
  Type     Reason                   Age   From             Message
  ----     ------                   ----  ----             -------
  Normal   Starting                 23m   kube-proxy       
  Normal   Starting                 23m   kubelet          Starting kubelet.
  Warning  InvalidDiskCapacity      23m   kubelet          invalid capacity 0 on image filesystem
  Normal   NodeAllocatableEnforced  23m   kubelet          Updated Node Allocatable limit across pods
  Normal   NodeHasSufficientMemory  23m   kubelet          Node controlplane status is now: NodeHasSufficientMemory
  Normal   NodeHasNoDiskPressure    23m   kubelet          Node controlplane status is now: NodeHasNoDiskPressure
  Normal   NodeHasSufficientPID     23m   kubelet          Node controlplane status is now: NodeHasSufficientPID
  Normal   RegisteredNode           23m   node-controller  Node controlplane event: Registered Node controlplane in Controller
  Normal   NodeReady                23m   kubelet          Node controlplane status is now: NodeReady

controlplane ~ ➜  k drain node01 --ignore-daemonsets
node/node01 cordoned
error: unable to drain node "node01" due to error: cannot delete cannot delete Pods that declare no controller (use --force to override): default/hr-app, continuing command...
There are pending nodes to be drained:
 node01
cannot delete cannot delete Pods that declare no controller (use --force to override)
```

# Cluster Upgrade Introduction

## k8s api version - software release

- kubectl get nodes에 나오는 version
    - v1.33.0
        - 1 → 메이저 버전
        - 33 → 마이너 버전
        - 0 → 패치 버전
- kube-apiserver, Controller-manager, kube-scheduler, kubelet, kube-proxy, kubectl → 같은 버전
- etcd cluster, coreDNS은 각각 다른 버전으로 관리

## control plane components

- kube-apiserver, Controller-manager, kube-scheduler, kubelet, kube-proxy, kubectl → 동일한 버전을 가져야 하는가? → kube api server가 주요 구성 요소 → 서로 다른 릴리스 버전에 있을 수 있음.
- 다른 컴포넌트들은 kube api server보다 같거나 낮은 버전이어야 함
- kube-apiserver → x(v1.10)
- controller-manager, kube-scheduler → x-1(v1.9, v1.10)
- kubelet, kube-proxy → x-2,(v1.8,v1.9,v1.10)
- kubectl → x+1 > x-1 → v1.11 ~ v1.8
    - 이 허용된 버전 편향 때문에 실시간 업그레이드 가능
    - k8s → 최근 3가지 버전까지만 제공

## version upgrade

- 한 단계씩(마이너 버전씩) 업그레이드 진행
    - 1.10 > 1.11 > 1.12 순서대로

## version upgrade 과정

### Cloud provider service

- console에서 하라는 대로 하면 끗(돈이 좋다..)

### kubeadm upgrade 전략

1. control plane upgrade 후 순차적으로 worker node upgrade
    - api-server, scheduler, controller-maneger 등 control plane component down
    - 그동안 worker node에 호스팅된 workload → 사용자에게 서비스 제공 → 하지만 kubectl or k8s api-server로 Cluster에 접근 x, controller-manager도 작동 x, pod가 fail되어도 자동으로 생성 x
        
        → control plane이 upgrade 중이라 당연히 Cluster 관리 못함
        
2. control plane upgrade 하는 동안 worker node upgrade
    1. 한꺼번에 Pod upgrade → application access x → downtime 발생
    2. 한 번에 한 node upgrade
        - 한 node씩 순차적으로 upgrade
    3. 최신 버전의 cluster node에 새로운 node 추가
        - 클라우드 환경에서 신규 프로비저닝 굿 → 편함

### kubeadm으로 version upgrade 하는 과정

```bash
# 1. control plane upgrade
#현재 k8s version, kubeadm 버전, 최신 k8s version
kubeadm upgrade plan 

#한 번에 한 가지 minor version upgrade 가능
apt-get upgrade -y kubeadm=1.12.0-00
# 필요한 image 불러오고 clutser 구성 요소 업그레이드
# -> control plane component 1.12로 업그레이드
kubeadm upgrade apply v1.12.0 

# 하지만 이전 Version으로 나옴
# -> kube-api-server는 upgrade, node는 이전 Version(upgrade x)
kubectl get nodes

# kubelet upgrade
apt-get upgrade -y kubelet=1.12.0-00
systemctl restart kubelet

# 2. worker node 한 개씩 upgrade
# 스케줄링된 pod 종료 + 새 pod 스케줄링 안되게 설정한 후 업그레이드 진행
kubectl drain node-1
# upgrade
apt-get upgrade -y kubelet=1.12.0-00
apt-get upgrade -y kubeadm=1.12.0-00
kubeadm upgrade node config --kubelet-version v1.12.0
systemctl restart kubelet
# drain 해제
# node schedulable 상태
kubectl uncordon node-1
```
