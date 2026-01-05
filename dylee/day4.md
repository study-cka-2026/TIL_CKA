# DAY4 - 0105    

# Replicasets

replica가 무엇이고 replication controller가 왜 필요한지

## Replication

- k8s clutser에서 단일 pod의 여러 instance를 실행할 수 있도록 하는 것. Replication controller는 항상 지정된 수의 pod가 실행되고 있음을 보장
    - single pod를 사용해도 pod가 죽으면 →  replication controller는 자동으로 새 pod 생성
- 고가용성
- 부하분산
    - replicaSet(replication controller)은 clutser 내 여러 노드에 분산되어 있음
    - 이 시스템은 서로 다른 node의 여러 pod에 걸쳐 부하분산 + 애플리케이션 확장 굿

## Replication Controller

```yaml
apiVersion: v1
kind: ReplicationController
#Replication Controller 전용 metadata
metadata:
	name: myapp-rc
	labels:
		app: myapp
		type: front-end
#Replication Controller 전용 spec
spec:
	#replication controller가 사용할 pod template
	template: 
	#Pod 전용 metadata
		metadata: myapp-pod
		labels:
			app: myapp
			type: front-end
		#Pod 전용 spec	
		spec:
			containers:
			- name: nginx-container
				image: nginx
replicas: 3 #replication 수
```

- repilcaset으로 대체되는 legacy

```bash
kubectl create -f rc-definition.yaml
#replication controller 확인
kubectl get replicationcontroller
#replication controller가 만든 pods 확인
kubectl get pods
```

- apiVersion, kind, metadata, spec, replicas → 같은 level

## ReplicaSet

```yaml
apiVersion: apps/v1 #replicaSet의 k8sApiversion -> apps/v1
kind: ReplicaSet
#ReplicatSet 전용 metadata
metadata:
	name: myapp-replicaset
	labels:
		app: myapp
		type: front-end
#Replicat 전용 spec
spec:
	#ReplicaSet가 사용할 pod template
	#pod 죽음
	#ReplicaSet가 pod 수 유지를 위해새 pod를 생성할 때 pod template가 필요
	template: 
	#Pod 전용 metadata
		metadata: myapp-pod
		labels:
			app: myapp
			type: front-end
		#Pod 전용 spec	
		spec:
			containers:
			- name: nginx-container
				image: nginx
replicas: 3 #replication 수
#replicaSet가 어떤 pod에 속하는지 식별
selector:
	matchLabels:
		type: front-end
```

```bash
#replicaSet 숫자 조정 방법
#1. 새로 만들기
kubectl create -f replicaset-definition.yaml
#2. scalce-replicas
#하지만 yaml의 replicas 숫자는 그대로(3)
kubectl scale --replicas=6 -f replicaset-definition.yaml
```

- replication 설정하는 새로운 recommendation
- pod 모니터링 → 고장나면 새로운 pod 배치 → 이때 replicaSet은 labels을 기준으로 pod 모니터링
- apiVersion, kind, metadata, spec, replicas, selector → 같은 level
    - selector: replicaSet가 어떤 pod에 속하는지 식별 → pod 범주화
- replicaSet은 replicaSet에 포함되지 않은 pod여도 selector에 있는 label에 포함된다면 같이 관리

# Deployment

```yaml
apiVersion: apps/v1 #replicaSet의 k8sApiversion -> apps/v1
kind: Deployment
#Deployment 전용 metadata
metadata:
	name: myapp-replicaset
	labels:
		app: myapp
		type: front-end
#Deployment 전용 spec
spec:
	#ReplicaSet가 사용할 pod template
	#pod 죽음
	#ReplicaSet가 pod 수 유지를 위해새 pod를 생성할 때 pod template가 필요
	template: 
	#Pod 전용 metadata
		metadata: myapp-pod
		labels:
			app: myapp
			type: front-end
		#Pod 전용 spec	
		spec:
			containers:
			- name: nginx-container
				image: nginx
replicas: 3 #replication 수
#replicaSet가 어떤 pod에 속하는지 식별
selector:
	matchLabels:
		type: front-end
```

```yaml
kubectl get deployments
kubectl get replicaset #deployment가 만든 replicaset확인
kubectl get all #모든 생성된 object 확인

```

- rolling update
    - instance upgrade시 한꺼번에 upgrade 하지 않고 차례로 upgrade 하는 것 →
- rolling update시 문제 발생 → rollback 하고 싶음
- pod로 캡슐화된 container들을 replicaSet을 이용해 배포
- Deployment
    - higher hierarchy인 k8s object
    - 앱 배포를 위한 선언적 컨트롤러
    - 자동으로 ReplicaSet 을 생성/관리
        - ReplicaSet이 Pod 숫자 유지
    - 기능
        - **RollingUpdate**(기본): Pod를 순차적으로 새 버전으로 교체
        - **Rollback(undo)**: 문제 발생 시 이전 revision으로 되돌리기
        - **Pause/Resume**: 배포 일시정지/재개
