# DaemonSets

- replicaset와 비슷함 → 모든 node가 pod의 복사본을 실행하도록 보장 → clutser 내 각 노드에 pod 한 개씩 복사본을 두는 역할
- 새 node가 cluster에 추가될 때 마다 pod의 복제본이 자동으로 node에 추가됨

## Use-Cases

- daemonSets로 monitoring 도구 배포 → 모든 node의 pods 모니터링 하기 좋음
    - sidecar과 함께한다면 좋을듯?

## Use-Cases-kube-proxy

- cluster내 모든 node에는 kube-proxy 필수 → kube-proxy component를 daemonsets로 배포한다면? → 모든 node에 자동으로 배포

## Use-Cases-Networking

- networking solution(calico, flannel, cilium) → cluster 각 node에 agent를 배치해야 함 → deamonSets를 이용해서 모든 node에 자동으로 배포하자

## DaemonSet Definition

```yaml
apiVerison: apps/v1
kind: ReplicaSet
metadata:
	name: monitoring-daemon
spec:
	#daemonSet과 매치할 labels
	selector:
		matchLabels:
			app: monitoring-agent
		#pods 사양
		template:
			metadata:
				labels:
					app: monitoring-agent
			spec:
				containers:
				- name: monitoring-agent
					image: monitoring-agent
```

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
	name: monitoring-daemon
spec:
	selector:
		matchLabels:
			app: monitoring-agent
		template:
			metadata:
				labels:
					app: monitoring-agent
			spec:
				containers:
				- name: monitoring-agent
					image: monitoring-agent
```

## 명령어

```bash
kubectl create -f daemon-set-definition.yaml
kubectl get daemonsets
kubectl describe daemonsets monitoring-daemon
```

## 동작 방식

- 스케줄러 우회가 필요함 → pod를 node에 직접 배치하기 위해 pod에 설치
- Till v 1.12 → use Default Scheduler
    1. Pod yaml에 nodeName 속성 명시
    2. 해당 nodeName에 pod 배포
- From v1.12 → Uses NodeAffinity and Default Scheduler

# Static Pods

## 정의

- kubelet에 의해 해당 node에 배치된 Pods
- api 서버 없이 node의 kubelet daemon에 의해 관리됨
- pod 수준에서 작동 → Pod 수준에서 할 수 있는 일만 가능

## 동작 방식

- 일반적인 Pods
    - kubelet이 node에 pod을 배치하려면? → kube Api Server에 의존 → 이때 스케줄링의 주체는 etcd의 kube-scheduler
- Static Pods
- kube-api server없이 Pods.yaml(명세)를 Kubelet에 전달 → pod 생성

### 1. File System을 통한 Static Pods 생성

1. static pods 실행할 node 선택
2. directory를 지정 → pod 정의
    
    ```bash
      # kubelet 이 동작하고 있는 노드에서 이 명령을 수행한다.
    mkdir -p /etc/kubernetes/manifests/
    cat <<EOF >/etc/kubernetes/manifests/static-web.yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: static-web
      labels:
        role: myrole
    spec:
      containers:
        - name: web
          image: nginx
          ports:
            - name: web
              containerPort: 80
              protocol: TCP
    EOF
    ```
    
3. pod 디렉토리 위치가 kubelet에 option으로 전달
    - 디렉토리 위치 = Pod Manifest Path
4. kubelet은  주기적으로 해당 directory에서 file 확인 → create pods, 변경 사항이 있다면 update pods
5. directory에서 pod definition file 삭제 → Pod 자동으로 삭제
- static pod 확인: `docker ps`
    - kube control 명령어가 구동되지 않는 이유 → node의 kubelet daemon에 의해 관리,pod 수준에서 작동 → control plane api server 로 관리할 수 없고, Kube control utility도 존재하지 않음 → 따라서 docker or podman등의 용어를 사용해야 함!

### 2. http api endpoint를 통한 Static Pods 생성

- kube api server가 Kubelet에 입력 제공
- kubelet으로 Static Pod을 만들 수 있다
- API 서버는 Static Pod를 인지하고 있다
    - kubelet이 static pod 생성 → 읽기 전용 mirror object 함께 생성 → `kubectl get pods` 실행 시 mirror pods를 확인할 수 있다! (원본 Pods x)
    - node의 manifest floder 에서 file을 수정해야 삭제할 수 있음

## Use Cases

- kubeadm tool에서 control plane을 구성하는 pod들을 static pod으로 배포
    - 그래서 kubectl에서 control plane 구성 요소들이 Pod처럼 보이지만 → kubelet이 만든 static pod에 대해 API Server에 mirror pod 형태로 등록되어 보임
- static pods → Kubernetes control plane(특히 API Server)에 의존하지 않고 kubelet이 관리 → api server, controller, etcd 정의 파일(yaml)을 manifest folder에 넣으면 kubelet이 배포 담당 → control plane 구성요소를 pod로 실행할 수 있음 → binary를 다운로드, 서비스 설정을 직접 하지 않아도 됨 → 컨테이너가 다운되면 kubelet이 재시작

## Static Pods vs DaemonSets

- 둘 다 일반 Pod처럼 kube-scheduler가 노드 선택 을 해서 배치하는 흐름이 아니다
    - Static Pod: 특정 node의 kubelet이 띄움 → 해당 node에 고정
    - DaemonSet Pod: DaemonSet 컨트롤러가 각 노드에 하나씩 만들고 nodeName을 채움 → 스케줄러 우회

### 차이점

1. yaml
- Static Pod:노드 로컬 파일 시스템(manifest 파일) → etcd에 desired state로 저장되지 않음
- DaemonSet: 클러스터 리소스(ETCD) → kubectl apply -f ds.yaml로 관리.
1. control plane 의존성
    - Static Pod: API Server가 죽어도 kubelet이 계속 관리 가능
    - DaemonSet: DaemonSet Controller(= control plane) 가 동작해야 새 노드에 배포/업데이트 등이 정상 작동
2. 운영
    - DaemonSet은 updateStrategy(RollingUpdate 등), tolerations, nodeSelector/affinity, resources, securityContext 같은 운영 옵션을 표준적으로 씀
    - Static Pod은 클러스터 컨트롤러 기반의 배포/업데이트 전략이 아니라 노드별 파일 수정이라 운영 자동화가 약함
