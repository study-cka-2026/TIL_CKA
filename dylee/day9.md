# Node Selector

- node selector로 더 높은 성능이 필요한 작업이 실행되는 pod에 node를 할당할 수 있음

```yaml
apiVersion:
kind: Pod
metadata:
	name: myapp-pod
spec:
	containers:
	- name: data-processor
		image: data-processor
	nodeSelector:
		#node에 할당된 label
		size: Large
```

- node에 할당된 Large Label을 통해 → pod를 배치할 노드 매칭 + 식별
- node에 label을 붙이고 → node selector을 사용해 pod를 만들면 → Large Label을 가진 node에 pod가 배치됨

```bash
kubectl label nodes <node-name> <label-key>=<label-value>
```

## 한계

- 요구사항이 복잡하다면?
    - large node 또는 medium node에 배치
    - 미래에 노드의 label이 변경된다면?

# Node Affinity

## PODs to Nodes

- pod이 특정 node에 호스팅 되도록 보장

## Node Affinity

- 특정 Pod 배치를 제한

```yaml
apiVerison:
kind: Pod
metadata:
	name: myapp-pod
spec:
	containers:
	- name: data-processor
		image: data-processor
	#pod가 matchExperssions 목록에 있는 Label 크기의 node에 배치되도록 보장
	affinity:
		nodeAffinity:
			requiredDuringSchedulingIgnoredDuringExecution:
				nodeSelectorTerms:
				- matchExpressions:
					- key: size
						operator: In
						values:
						- Large
						- Medium
```

- Large, Meduim Label를 가진 Node에 Pod 배치

```yaml
					- key: size
						operator: NotIn
						values:
						- Small
```

- Small Label 을 가지지 않은 Node에 Pod 배치

## Affinity에 명시된 Label을 가진 Node가 없는 경우?

## Node Affinity Types

- scheduler가 node affinity에 어떻게 행동하는지 정의

### Available

`requiredDuringSchedulingIgnoredDuringExecution` 

- 일치하는 label을 가진 node가 없다면? → Pod 스케줄링 x
- pod의 상태가 중요한 case에 사용

`preferredDuringSchedulingIgnoredDuringExecution`

- 일치하는 label을 가진 node가 없다면? → 다른 node에 pod 스케줄링
- pod의 배치가 중요한 case에 사용
- `DuringExecution`
    - pod가 실행되고, 환경에 변화가 발생하는 상태
        - node의 label 변화
    - `Ignored`
        - node affinity가 Ignored로 설정되어 있다면, 환경에 변화가 발생해도 pod는 계속 작동

### Planned

`requiredDuringSchedulingRequiredDuringEx` 

- 

# Node Affinity vs Taints and Tolerations

## Taints and Tolerations

- node에 taint 설정 → node에 대한 Tolerations가 있는 Pod만 해당 node에 스케줄링 가능 → Taint가 설정된 노드 또는 tolerations가 없는 노드에 스케줄링

## Node Affinity

- node의 label과 pod의 node selector으로 node-pods 연결 → Pod들은 label이 일치하는 node에 배치
- 하지만 label이 일치하지 않는 Pod도 Node에 스케줄링 될 수 있음

## Taints and Tolerations + Node Affinity

- node에 Taint + Pod에 tolerations 설정(해당 node에 스케줄링 가능) → 그 후 노드에 label + 파드에 node affinity(required)를 주면 label node에 스케줄링 강제 → 원하는 node에만 Pod가 스케줄링

# Resource Limits

- k8s scheduler → pod가 어떤 node로 갈 지 결정
    - scheduler는 pod가 필요로 하는 자원 양과 사용 가능한 자원을 고려해서 node를 결정
    - 만약 어떤 node에도 자원이 충분하지 않다면 → pending
    - pending의 원인을 파악하는 명령어? → `kubecontrol describe pod command`

## Resource Requests

- container가 요청하는 최소 cpu, memory 양
- pod를 만들 때 cpu, memory 양을 지정하는 방법?
    
    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
    	name: simple-webapp-color
    	labels:
    		name: simple-webapp-color
    	spec:
    		containers:
    		- name: simple-webapp-color
    			image: simple-webapp-color
    			ports:
    				- containerPort: 8080
    			resources:
    				requests:
    					memory: "4Gi"
    					cpu: 2
    ```
    

## Resource-CPU

- 1 core

## Resource-Memory

- Gi → 기가바이트
- Mi → 메가바이트

## Resource Limits

```yaml
apiVersion: v1
kind: Pod
metadata:
	name: simple-webapp-color
	labels:
		name: simple-webapp-color
	spec:
		containers:
		- name: simple-webapp-color
			image: simple-webapp-color
			ports:
				- containerPort: 8080
			resources:
				requests:
					memory: "4Gi"
					cpu: 2
				limits:
					memory: "2Gi"
					cpu: 2
```

- memory, cpu limit
- 각 컨테이너마다 요청, 제한 설정 가능

## Exceed Limits

- cpu → pod가 limits 넘으려고 하면 한계를 넘지 않도록 throttles 조절 → 한계 이상으로 cpu 사용 x
- memory → pod가 더 많은 메모리를 소모하려고 하면 → pod 종료, log 확인 시 오류 발생 (OOM)

## Default Behavior - cpu

- limit가 설정되지 않는다면, 한 pod가 node의 모든 cpu 자원을 소모할 수 있음
- 추가 cpu를 소모하지 않도록 제한하고 싶지 않다면(No Limits 설정) → Requests를 설정해서 pod에 자원 할당을 보장하자

## Default Behavior - memory

- cpu와 달리 memory가 한 번 pod에 할당하면 memory를 제한할 수 없음
    - memory를 free 하려면 pod를 Kill해야 함..

## LimitRange

- 생성된 pod내 container에 cpu, memory 기본 값 설정하는 yaml
- pod가 생성될 때 적용 → 기존에 생성된 pod에는 영향 x

```yaml
apiVersion: v1
kind: LimitRange
metadata:
	name: cpu-resource-constraint
spec:
	limits:
	- default: #limit
			cpu: 500m
		defaultRequest: #request
			cpu: 500m
		max: #limit
			cpu: "1"
		min: #request
			cpu: 100m
		type: Container
```

```yaml
apiVersion: v1
kind: LimitRange
metadata:
	name: memory-resource-constraint
spec:
	limits:
	- default: #limit
			memory: 1Gi
		defaultRequest: #request
			memory: 1Gi
		max: #limit
			memory: 1Gi
		min: #request
			memory: 500Mi
		type: Container
```

## Resource Quotas

- namespace 수준에서 Quotas를 만들면 namsepace에 해당하는 모든 pod들의 hardware를 제한할 수 있다

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
	name: my-resource-quota
spec:
	hard:
		requests.cpu: 4
		requests.memory: 4Gi
		limits.cpu: 10
		limits.memory: 10Gi
```
