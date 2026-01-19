# Multiple Schedulers

## 개념

- 커스텀 스케줄러는 별도의 kube-scheduler instance를 추가로 띄우는 것!
- 커스텀/조건 정책으로 스케줄링을 분리하고 싶을 때 사용(GPU 전용 등..)
- 여러 스케줄러 프로세스를 동시에 실행하는 옵션(기본 스케줄러 + 추가 스케줄러 분리해서 운영) →  Pod를 배포할 때 spec.schedulerName 옵션을 통해 스케줄러 지정 가능
- 기본 스케줄러 → `default-scheduler`
    - Pod에 schedulerName을 안 적으면 기본 스케줄러가 스케줄링

## 커스텀 스케줄러 생성 시 유의점?

### 1. schedulerName 일치

- config file의 profiles[].schedulerName
- Pod의 spec.schedulerName
- 위 두 가지가 일치해야 scheduler가 해당 Pod를 배치

### 2. Leader Election 충돌 방지

- 여러 개의 스케줄러 복사본이 서로 다른 master Node에서 실행될 때 사용
- 스케줄러가 여러 개 있거나, HA control plane에서 구동시키려면 → Leader Election lock이 필요
- `leaderElection.leaderElect: true`
- resourceName도 scheduler마다 달라야 충돌 발생 x

### 3. kubeconfig

- scheduler → API 서버 접근 필요 →`kubeconfig=/etc/kubernetes/scheduler.conf` 필요함
- Pod로 custom scheduler를 띄운다면 hostPath로 스케줄러 파일 마운트 필요!(그래야 pending x)

### 4. port, bind 충돌 방지

- 추가 스케줄러가 기본 스케줄러와 같은 노드에서 뜬다면 → metrics/health port, bind된 주소 등이 겹쳐서 실패할 수 있음
- 따라서 `--bind-address` , `--secure-port` 를 서로 다르게 설정

## 배포 방식

### 1. binary로 custom scheduler 실행

- custom scheduler는 다른 config를 읽어야 하고, 다른 schedulerName을 사용해야 함
1. config 작성

```bash
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
profiles:
- schedulerName: my-scheduler
```

1. systemd로 추가 스케줄러 실행
    - `ExecStart=/usr/local/bin/kube-scheduler \\ --config=/etc/kubernetes/config/my-scheduler-2-config.yaml`

### 2. kubeadm 환경에서 Pod로 custom scheduler 실행

- kubeadm cluster → control plane이 static pod로 배포 → kube-system을 이용해 scheduler pod를 배포할 수 있음
- `kubectl get events -o wide` 로 어떤 스케줄러에 pod가 스케줄링 되었는지 확인할 수 있고, `kubectl get pods -n kube-system` 으로 확인 가능
- `kubectl logs my-custom-scheduler --namespace=kube-system` 으로 로그를 생성해 스케줄러 팟 이름, 배포 정보 확인 가능
- my-custom-scheduler.yaml

```bash
apiVersion: v1
kind: Pod
metadata:
	name: my-custom-scheduler
	namespace: kube-system
spec:
	containers:
	- command:
		- kube-scheduler
		- --address=127.0.0.1
		#scheduler 경로
		- --kubeconfig=/etc/kubernetes/scheduler.conf
		#scheduler configuration
		- --config=/etc/kubernetes/my-scheduler-config.yaml
		image: k8s.gcr.io.kube-scheduler-amd64:v1:XX.X
		name: kube-scheduler
```

- my-scheduler-config.yaml

```bash
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
profiles:
#name으로 default scheduler와 custom scheduler 비교
- schedulerName: my-custom-scheduler1
- schedulerName: my-custom-scheduler2
#여러 개의 스케줄러 복사본이 서로 다른 master Node에서 실행될 때 사용 
leaderElection:
	leaderElect: true
	resourceNamespace: kube-system
	resourceName: lock-object
```

- 커스텀 스케줄러 배포 → schedulerName 지정해서 pod 배포
    
    ```bash
    apiVersion: v1
    kind: Pod
    metadata:
    	name: nginx
    spec:
    	containers:
    	- name: nginx
    		image: nginx
    	schedulerName: my-custom-scheduler
    ```
    
- 만약 pod가 pending 이라면? → pod describe로 custom scheduler를 다시 확인해 보자
    - 보통 **schedulerName 불일치**가 제일 흔한 원인

## Use cases

- 워크로드 분리
    - batch Job: custom scheduler
    - service traffic pod: default scheduler
- 특정 노드 전용 정책
    - gpu, 고성능 node 전용 scheduler 설정
- 정책/플러그인 구성 다르게
- 점진적 테스트
    - 새 스케줄링 정책을 일부 Pod에만 적용 → 검증

# Scheduler Profiles

- Profiles → Extension Points에 어떤 플러그인을 **enabled/disabled 할지**를 **프로필 별로 다르게** 가져가는 기능
- scheduler 1개(binary 1개) 안에서 schedulerName 별로 다른 정책 적용 가능
    - Multiple Schedulers → 스케줄러 프로세스 여러 개 띄움

## Scheduling Plugins

- Pod가 스케줄링 되는 과정 구현
- scheduler 1개의 동작을 profiles 단위로 분기 → schedulerName 별로 서로 다른 plugin 적용
- 각 Extension Point에서 실행되는 플러그인들로 스케줄링 프레임워크 구성
1. Scheduling Queue
    - Priority Sort Plugins가 pod를 우선순위 순서대로 정렬
    - QueueSort + backoff(pod가 스케줄링 불가한 상태)등을 고려해서 → 다음으로 처리할 pod결정
2. Filtering
    - Node Resource Fit Plugins가 pod를 실행시키기에 충분한 자원을 가지지 않은 node를 필터링
    - NodeName → pod명세에 명시된 node 이름이 아닌 node 필터링
    - NodeAffinity/NodeSelector, Taints/Tolerations, Volume 제약, Topology 제약 등등 고려함
    - NodeUnschedulable → unschedulable 가진 node 필터링 → 어떤 pod도 해당 node에서 실행 x
        - 하지만 DaemonSet/static pod 등은 실행될 수 있음
        - toleration, 기타 조건에 따라 스케줄링 가능할수도?
3. Scoring
    - NodeResourcesFit → node에 점수를 부여하고 → 이 점수에 따라 Pod 할당
    - ImageLocality → 이미 pod들이 사용하는 컨테이너 이미지를 가진 node에 점수 부여
4. Binding
    - 최고 점수 Node에 Pod를 바인딩
    - DefaultBinder → pod와 바인딩하는 매커니즘 제공

## Extension Points

- Pod가 스케줄링되는 stage마다 plugins를 연결할 수 있는 extension points
- Scheduling Queue
    - queueSort plugins
- Filtering
    - prefilter extension
    - filter extension, score ..
- Scoring
    - preScore, score, reserve
- Binding
    - permit , preBind, bind..
- plugin를 만들어 custom code를 만들 수 있다 → k8s의 확장 가능한 특징 때문

## Use Cases

- 환경: 1개의 default scheduler, 2개의 custom schedulers
- multiple scheduler 배포 → race condition 문제가 발생할수도 있음
    - 같은 노드에서 동시에 workload 스케줄링
    - 설정/락(leader election) 충돌
    - 포트 충돌(health/metrics)
    - RBAC/kubeconfig 문제
    - schedulerName 불일치로 Pending

### v 1.18(Profiles 도입 → 단일 scheduler에서 multiple profiles)

- 단일 스케줄러에서 multiple profiles 지원 → scheduler 별로 각자의 profile 생성 → 이 Profile 자체가 별도의 스케줄러 역할을 함→ 하지만 스케줄러들이 같은 binary로 묶여서 실행됨(한 File 안에 같이 있음)
- kube-scheduler 하나가 여러 schedulerName을 논리적 스케줄러처럼 처리하는 구조

```bash
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
profiles:
#multiple profile
- schedulerName: my-custom-scheduler1
- schedulerName: my-custom-scheduler2
#여러 개의 스케줄러 복사본이 서로 다른 master Node에서 실행될 때 사용 
leaderElection:
	leaderElect: true
	resourceNamespace: kube-system
	resourceName: lock-object
```

- 이 설정 → 스케줄러 프로세스 복제(HA) 시 필요함
- Profiles 여러 개를 둔다고 해서 resourceName을 프로필마다 따로 두는 건 아님(같은 프로세스 안이기 때문)
    - 하지만 다른 스케줄러 프로세스를 추가로 띄우는 경우(Multiple Schedulers)에는 resourceName 충돌이 문제될 수 있다

### profile 별로 별개의 scheduler 처럼 동작시키는 방법

- 기본 스케줄러처럼 작동하지 않고 별개의 커스텀 스케줄러처럼 작동하게 설정해서, 여러 개의 스케줄러 프로필을 다르게 설정하려면?
    - 각 스케줄러 프로필 아래에서 플러그인을 커스텀 하기 → plugin section에서 확장 포인트 지정, 플러그인 이름으로 활성화 또는 비활성화
    - pod spec에서 schedulerName이 profile의 schedulerName과 일치해야 해당 Profile로 스케줄링 → 지정해주지 않으면 기본 profile(default-scheduler)에 pod가 스케줄링 됨
    
    ```yaml
    apiVersion: kubescheduler.config.k8s.io/v1
    kind: KubeSchedulerConfiguration
    profiles:
    #multiple profile
    - schedulerName: my-custom-scheduler1
    	plugins:
    		score:
    			disabled:
    			- name: TaintToleration
    			enabled:
    			- name: MyCustomPluginA
    			- name: MyCustomPluginB
    - schedulerName: my-custom-scheduler2
    	plugins:
    		preScore:
    			disabled:
    			- name: '*'
    			score:
    				disabled:
    				- name: '*'
    
    #여러 개의 스케줄러 복사본이 서로 다른 master Node에서 실행될 때 사용 
    leaderElection:
    	leaderElect: true
    	resourceNamespace: kube-system
    	resourceName: lock-object
    ```
    

# Admission Controllers

- kubelet → kube-api-server → Authentication(certification) → Authorization → Admission(Validating/Mutating) → etcd 저장
- Authentication
    - 이때 요청이 kubectl을 통해 전송된다면 → kube config file에 인증서가 설정되어 있음
- Authorization
    - role-based로 사용자 권한 확인 → RBAC
    - --authorization-mode=Node,RBAC 처럼 **여러 모드가 함께** 켜질 수 있음
        
        ```yaml
        role.yaml
        apiVersion: rbac.authorization.k8s.io/v1
        kind: Role
        metadata:
        	name: developer
        rules:
        - apiGroups: [""]
        	resources: ["pods"]
        	verbs: ["list", "get", "create", "update", "delete"]
        ```
        

## Authorization - RBAC

### RABC(Authorization)이 할 수 있는것

- create pods 가능/불가능

### RABC(Authorization)가 못하는 것

- 이미지가 **docker hub인지/private registry인지**
- container가 **특정 capability를 추가했는지**
- label이 **반드시 포함됐는지**
- spec이 **정책을 만족하는지** (예: privileged 금지)
    
    같은 “요청 내용(spec)의 품질/정책 준수”는 판단 못 해.
    

→ 따라서 Admission controller에서 role-based access control로 API level의 여러 가지 restrictions 설정 → 생성, 삭제, etc.. → 특정 리소스(pods, namespace 등등)에 대한 접근 제한도 가능!

## Admission Controllers

- authentification → authorization(RBAC) → Admission(정책 적용) → etcd 저장
1. Mutating
    - 요청 객체를 **수정/추가**해서 통과시키는 쪽
    - ex)
        - 라벨이 없으면 자동으로 붙여주기
        - storageClassName 없으면 기본값 채우기
2. Validating
- 요청 객체를 **검사해서 허용/거부**
- ex)
    - docker hub 이미지면 거부
    - 특권(capabilities/privileged) 있으면 거부
    - label이 없으면 거부
- 사용 옵션
    - AlwaysPullImages → ValidatingAdmissionWebhook으로 많이 구현함(imagePullPolicy를 일괄적으로 강제하고 싶을 때 사용)
    - Only permit certain capabilitie → Pod Security Admission(PSA) 또는 Validating Webhook으로 처리
- DefaultStorageClass
    - 사용자가 PVC 만들 때 storageClassName 안 적어도 클러스터의 “기본 StorageClass”를 자동으로 붙여주고 싶을 때 사용
- EventRateLimit
    - 특정 클라이언트/네임스페이스/유저가 이벤트를 너무 많이 만들어서 API 서버를 압박하는 걸 막고 싶을 때 사용 → Event 생성 요청을 Admission에서 **rate limit** 규칙으로 제한
- NamespaceAutoPrivision
    - namespace 존재 x → 자동으로 생성
    - default → pod 생성 x
- admission controller 확인
    
    ```bash
    kube-apiserver -h | grep enable-admission-plugins
    ```
    
    ```bash
    kubectl exec kube-apiserver-controlplane -n kube-system -- kube-apiserver -h | grep enable-admission-plugins
    ```
    

## Enabling Admission Controllers

```bash
apiVersion: v1
kind: Pod
metadata:
	creationTimestamp: null
	name: kube-apiserver
	namespace: kube-system
spec:
	containers:
	- command:
		- kube-apiserver
		- --authorization-mode=Node,RBAC
		- --advertise-address=172.17.0.107
		- --enable-admission-plugins=AlwaysPullImages,EventRateLimit,...

```

- **NamespaceAutoProvision 같은 기능이 켜져 있고, 조건을 만족한다면**
존재하지 않는 namespace에서 pod 프로비저닝 → 요청이 인증, 승인, 네임스페이스 자동 생성 → pod 생성
