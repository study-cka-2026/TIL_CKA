# Authorization

## Authorization

- 권한 부여를 통해 관리자, 배포 애플리케이션(Jenkins), bots 등은 cluster 포드 배포, 관리 등의 작업을 수행할 수 있음
- 주로 TLS 인증서를 통해 access
- 하지만 역할에 따른 권한 limit이 필요

## Authorization Mechanisms

1. **Node**
- 사용자, kubelet → kube-api 를 통해 cluster 관리
- 이러한 요청은 system node or system node group의 사용자만 처리 가능
1. **ABAC**
- 사용자 속성 기반 권한 부여
- 사용자 또는 사용자 그룹에 policy file을 통해 kube-api에 대한 외부 액세스 제어
- policy file을 만들어서 api 서버로 전달 → 외부 사용자 또는 그룹 액세스 가능
- 하지만 policy file 수정 후 kube api server을 재시작 해야 함 → 관리하기 어려움
1. **RBAC**
- 역할 기반 액세스 제어
- 사용자 또는 그룹에 직접 권한을 연결하지 않는다
- 규칙을 만든 후 → 사용자 또는 그룹과 연결 → 즉시 권한 부여(반영)
1. **Webhook**
- 외부(thrid-party tool 등)에서 인증을 관리하는 방법
- k8s가 access 요구사항을 가지고 webhook을 통해 Open Policy Agent에 api call → Open Policy Agent가 사용자 허용 여부 결정

## 권한 설정 방법

- kube API의 authorization-mode를 통해 설정 가능
    - AlwaysAllow & AlwaysDeny
        - 권한을 항상 허용 또는 항상 거부
    - 여러 모드를 쉼표로 구분해서 제공 가능 → RBAC, Webhook..
        - 지정된 순서대로 승인 → RBAC 에서 요청이 거부된다면 다음 순서인 Webhook로 전달

# RBAC

## 역할 만들기

1. 역할 객체 생성
    - 역할 이름, 규칙으로 구성
    - 규칙 → apiGroups, 권한을 부여할 리소스, 해당 리소스에 허용할 행위(verbs)
    - 핵심 그룹 → apiGroups section 비워두는 경우도 있음
    
    ```yaml
    apiVersion: rbac.authorization.k8s.io/v1
    kind: Role
    metadata:
    	name: developer
    rules:
    - apiGroups: [""]
    	resources: ["pods"]
    	verbs: ["list", "get", "create", "update", "delete"]
    - apiGroups: [""]
    	resources: ["ConfigMap"]
    	verbs: ["create"]
    	
    kubectl create -f developer-role.yaml
    ```
    
2. 사용자와 규칙 연결
    - role-binding object 생성
    
    ```yaml
    apiVersion: rbac.authorization.k8s.io/v1
    kind: RoleBinding
    metadata:
    	name: devuser-developer-binding
    subjects:
    - kind: User
    	name: dev-user
    	apiGroup: rbac.authorization.k8s.io
    roleRef:
    	kind: Role
    	name: developer
    	apiGroup: rbac.authorization.k8s.io
    
    kubectl create -f devuser-developer-binding.yaml
    	
    ```
    
    - roles, role bindings → namespace scope에 속함
3. RABC 확인
    
    ```bash
    kubectl get roles #role 확인
    
    kubectl describe role developer
    
    kubectl get rolebindings #자세한 role-binding 확인
    
    kubectl describe rolebinding devuser-developer-binding
    ```
    

## 권한 확인

```bash
kubectl auth can-i 'verbs + resources' --as 'user_name'

kubectl auth can-i create deployments
```

### 특정 resource에 대한 access 설정?

- rules에 resourceName field를 추가해 특정 리소스에만 접근하도록 설정 가능
- 예시 → blue pods에만 접근 가능한 Role 설정
    
    ```yaml
    apiVersion: rbac.authorization.k8s.io/v1
    kind: Role
    metadata:
    	name: dev
    rules:
    - apiGroups: [""]
    	resources: ["pods"]
    	verbs: ["get","create","delete"]
    	resourceNames: ["blue"]
    	
    
    ```
    

# Cluster Roles

- role, role-binding→ namespace 기반 (지정 안한다면 default ns 제어)
- namespace resources
    - 생성 시 namespace를 지정하는 리소스(지정 안한다면 default ns에 생성)
    - pod, replicaset, deployments, service, secret
    - `kubectl api-resources --namespaced=true`
- cluster resources
    - 생성 시 namespace를 지정하지 않는 리소스
    - node, persistentVolume, clusterroles, clusterrolebindings, namespace, certificatesigningrequests
    - `kubectl api-resources --namespaced=false`
    - 하지만 namespace resources에 대한 clusterRole도 만들 수 있음 → 이 경우 사용자는 모든 namespace에서 cluster resources에 access 가능

## ClusterRoles

- cluster resources에 대한 role 설정
- cluster-admin-role을 만들어서 → 생성, 삭제, 보기 등 권한 제공
- 예시
    
    ```yaml
    apiVersion: rbac.authorization.k8s.io/v1
    kind: ClusterRole
    metadata:
    	name: cluster-admin
    rules:
    - apiGroups: [""]
    	resources: ["nodes"]
    	verbs: ["list", "get", "create", "delete"]
    	
    ```
    

## ClusterRoleBindings

- 사용자와 cluster Role 연결
- 예시
    
    ```yaml
    apiVersion: rbac.authorization.k8s.io/v1
    kind: ClusterRoleBinding
    metadata:
    	name: cluster-admin-role-binding
    subjects:
    - kind: User
    	name: cluster-admin
    	apiGroup: rbac.authorization.k8s.io
    roleRef:
    	kind: ClusterRole
    	name: cluster-admin
    	apiGroup: rbac.authorization.k8s.io	
    ```
    

# Service Accounts

- service accounts → bot, application의 상호작용 하는데 사용
    - Prometheus, Jenkins api 폴링
- token으로 k8s api에 service account 인증

## 동작 방식

- k8s cluster 설정 → 자동으로 모든 namespace에 service account 생성 → pod 생성 시 자동으로 연결
    - service account는 pod내의 projected volume으로 자동 mount → pod 삭제 시 token 자동 만료
        - projected volume: k8s가 자동으로 pod 내부에 생성하는 동적 디렉토리
            - 저장 경로 → `/var/run/secrets/kubernetes.io/serviceaccount`
            - token 확인 가능

```bash
kubectl get serviceaccount #서비스 계정 확인 

kubectl describe serviceaccount default #서비스 계정 세부 정보 확인
```

## service account 생성

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
	name: dashboard-sa
	namespace: default
automountServiceAccountToken: false
# token이 pod 내에 자동으로 mount 되는 거 방지

kubectl create serviceaccount dashboard-sa
```

## service account 연결

- pods ↔ service account 연결

```yaml
apiVersion: v1
kind: Pod
metadata:
	name: my-dashboard
spec:
	containers:
		- name: my-dashboard
			image: my-dashboard
	serviceAccountName: dashboard-sa
	automountServiceAccountToken: false

```

## create Token

- pod가 service account 연결 → k8s는 만료일이 적힌 토큰 자동 마운트 → pod내 실행 중인 application이 token을 이용해 k8s api 호출
- cluster 외부에서 token을 사용한다면?
    - default → 1시간 후 만료. duration에 token 만료 시간 설정해주기
    - `kubectl create token <service account name> --duration <time>`

# Image Security

## Image

- `image: docker.io/library/nginx`
- name convention
    - image 이름 = image의 repository 이름
- library
    - 이미지 제공자
    - 사용자 이름, 계정 제공하지 않는다면 → `image: library/nginx`
    - 제공한다면 → `image: <name>/<repository-name>`
- registry
    - 이미지 저장소
    - docker registry → docker.io
    - google registry → gcr.io

## Private Repository

- private repository에 access 하려면?

```yaml
docker login private-registry.io #인증

docker run private-registry.io/apps/internal-app #private registry 사용해서 run

#yaml 작성시 이미지 경로 image 경로 -> private-registry.io/apps/internal-app
apiVersion: v1
kind: Pod
metadata:
	name: nginx-pod
spec:
	containers:
	- name: nginx
		image: private-registry.io/apps/internal-app
	imagePullSecrets:
	- name: regcred

```

- k8s에서 private registry에 access 하는 방법
    1. secret credentials 생성
        
        ```bash
        kubectl create secret docker-registry regcred \
        	--docker-server= private-registry.io \
        	--docker-username= registry-user  \
        	--docker-password= registry-pw \
        	--docker-email= registry-user@org.com \
        ```
        
    2. Pod 생성 → worker node의 kubelet은 secret의 credentials을 사용해서 Image pull

# Security In Docker

## docker host

- 여러 process, docker daemon, ssh 등 여러 프로세스 실행 중
- container → host의 kernel 공유하지만 container의 고유한 Namespace를 사용해서 격리 → host 자체에서 자체 namespace를 가진 container가 실행됨
- docker container → 자체 namespace에 존재하고, 자체 process만 볼 수 있음

## Security-Users

- docker → root 사용자로 container 안에서 process 실행
    - container 내부의 process의 root user가 모든 권한을 가지고 시스템의 모든 작업을 할 수 있음 → 보안상 위험할지도.. → 따라서 docker 에서는 container root user ≠ host root user
    - 만약 더 많은 권한을 container root에 주려면? → `cap app` 옵션 사용
    - 덜 주려면 → `cap drop` 옵션 사용
- container 안의 process가 root로 실행되지 않게 하려면
    1. docker run 시 사용자 지정
        
        ```bash
        docker run --user=1001
        ```
        
    2. image에 미리 사용자 정의 → 이 이미지를 실행한다면 process는 사용자 id 1000으로 실행(root x)
        
        ```docker
        #image
        FROM ubuntu
        
        USER 1000
        
        #image build
        docker build -t my-image .
        ```
        

# Security Contexts

## Container Security

- k8s에서도 docker 처럼 사용자 id, container cap add & drop 등으로 권한을 추가하거나 제거할 수 있음
- k8s → container가 pod로 캡슐화 → container level 또는 Pod level 에서 보안 설정 구성 가능

## Pods

- 보안 설정 → 모든 container에 적용

```yaml
apiVersion: v1
kind: Pod
metadata:
	name: web-pod
spec:
	containers:
		- name: ubuntu
			image: ubuntu
			command: ["sleep", "3600"]
			securityContext:
				runAsUser: 1000
				#기능
				capabilities:
					add: ["MAC_ADMIN"]

```

# Network Policy (19)

- ingress traffic: 들어오는 트래픽
- egress traffic: 나가는 트래픽

## Network Security

- k8s networking 전제 조건: 추가 설정 구성 없이 pod는 서로 통신 가능해야 함
    - 기본적으로 cluster 내의 pod, service 끼리는 모든 트래픽 허용

## Network Policy

- network policy를 설정해 pod의 Ingress, egress 정책 설정 가능하며, k8s cluster에 구현된 네트워크 솔루션에 의해 적용됨
    - 특정 pod끼리만 통신 가능하도록 설정
        - ex) DB pod는 API pod와만 통신 가능 → DB pod는 port:3306으로부터 오는 ingress traffic만 허용
    - Flannel은 network policy 지원 x
- network policy 연결 → Label & selector 사용
    - db pod와 연결하고, 모든 트래픽 차단 → 3306만 DB에 query할 수 있도록 설정

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
	name: db-policy
spec:
	podSelector:
		matchLabels:
			role: db
	policyTypes:
	- Ingress:
	ingress:
	- from:
		- podSelector:
				matchLabels:
					name: api-pod
		ports: #traffic이 통과할 수 있는 ports
		- protocol: TCP
			port: 3306
```

- 만약 label은 같지만, namespace가 다른 api pod가 여러개라면? → namespace selector 사용

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
	name: db-policy
spec:
	podSelector:
		matchLabels:
			role: db
	policyTypes:
	- Ingress:
	ingress:
	- from:
		- podSelector:
				matchLabels:
					name: api-pod
				namespaceSelector:
					matchLabels:
						kubernetes.io/metadata.name: prod
		ports: 
		- protocol: TCP
			port: 3306
```

- 특정 namespace(prod)만 db pod에 접근 가능한 경우

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
	name: db-policy
spec:
	podSelector:
		matchLabels:
			role: db
	policyTypes:
	- Ingress:
	ingress:
	- from:
			namespaceSelector:
				matchLabels:
					kubernetes.io/metadata.name: prod
		ports: 
		- protocol: TCP
			port: 3306
```

- cluster 외부 서버의 요청을 허용하고 싶은 경우 → ipBlock

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
	name: db-policy
spec:
	podSelector:
		matchLabels:
			role: db
	policyTypes:
	- Ingress:
	ingress:
	- from:
		- podSelector:
				matchLabels:
					name: api-pod
				namespaceSelector:
					matchLabels:
						kubernetes.io/metadata.name: prod
		- ipBlock:
					cidr: 192.168.5.10/32
		ports: 
		- protocol: TCP
			port: 3306
```

- db pod의 내용을 외부 서버에 백업하고 싶은 경우 → egress

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
	name: db-policy
spec:
	podSelector:
		matchLabels:
			role: db
	policyTypes:
	- Ingress
	- Egress
	ingress:
	- from:
		- podSelector:
				matchLabels:
					name: api-pod
				namespaceSelector:
					matchLabels:
						kubernetes.io/metadata.name: prod
		ports: 
		- protocol: TCP
			port: 3306
		egress:
		- to:
			- ipBlock:
					cidr: 192.168.5.10/32
			ports:
			- protocol: TCP
				port: 80
		
```

# CRD(Custom Resource Definitions)

- deployment 생성 → 이 정보를 etcd에 저장 → deployment 삭제 → etcd 데이터 저장소에서 삭제 or 수정

## deployment controller

- 백그라운드에서 실행되는 프로세스로, resource 모니터링 → etcd 저장소의 상태(deployment 생성, 업데이트, 삭제 등 상태 변화)에 따라 cluster의 변경 수행

## custom resources definition

- 사용자 리소스 정의에 대해서 k8s에 알려줘야 함 → .yaml로 생성 → etcd에 해당 리소스 저장
- custom controller로 custom resource 생성, .yaml(명세)에 따른 동작 수행

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
	name: <api-group-name>
spec:
	scope: Namespaced
	group: <group-name>
	names:
		kind: <resource-kind>
		singular: <name> # kubectl 출력에 resource 유형을 표시하는데 사용
		plural: <name> # kubectl api-resource 출력에서 표시하는데 사용
		shortNames:
			- <name> # kubectl get <name> 에서 사용
	versions:
		- name: v1
			served: true
			storage: true
			schema:
				openAPIV3Schema: # spec section에서 지정할 수 있는 모든 매개변수 정의
												 # 객체 생성 시 어떤 필드 지원, 어떤 값 처리하는지 정의
```

# Custom Controllers

- 반복적으로 실행되며 k8s를 지속적으로 모니터링하는 프로세스, 코드
- controller 구축은 가능하면 Go로 하자

# Storage in Docker

## File System

- docker가 로컬 파일 시스템에 데이터 어케 저장하는지?

## Layered Architecture

- image → 계층화된 아키텍처로 빌드
- 각 Layer은 이전 레이어의 변경 사항 저장 → 새 이미지 build시 Layer 재사용 → Docker image 더 빠른 빌드, 디스크 공간 효율적 사용 가능
- 하지만 build 완료 → Layer의 콘텐츠 수정 x
    - 수정하고 싶다면 다시 build 해야 함
- build된 image로 container 생성 → image Layer 위에 write 가능한 새 Layer(container Layer) 생성
    - container Layer
        - container에서 생성한 data 저장(로그 등)
        - container가 사라지면 내용도 사라짐
        - 이 데이터를 유지하려면 persistent volume 추가 필요
- 예시
    1. 우분투 계층 → apt 명령어 계층 → pip install 계층 → 소스 파일 복사 계층 → 이미지 엔트리포인트 업데이트 계층 ⇒ 총 5가지의 Layer로 이루어진 이미지
    2. #2 이미지는 #1에서 구축한 1~3단계 레이어를 cache에서 꺼내 재사용하고, 4,5단계 레이어만 재생성

```yaml
# 1
FROM ubuntu
RUN apt-get update
RUN pip install ~
COPY ./opt/source-code
ENTRYPOINT FLASK_APP=/opt/source/app.py flask run
```

```yaml
# 2
FROM ubuntu
RUN apt-get update
RUN pip install ~
COPY app2.py ./opt/source-code
ENTRYPOINT FLASK_APP=/opt/source/app2.py flask run
```

## persistent Volume

- 볼륨 디렉토리에서 볼륨 마운트

```bash
docker volume create data_volume

#mysql container 실행 시 도커 컨테이너 layer에 mount
#새 container 생성 시 data volume이 container 내부의 /var/lib/mysql에 마운트
docker run -v data_volume:/var/lib/mysql mysql

```

- 외부 디렉토리에 마운팅을 하고 싶은 경우 → 마운트하려는 홀더의 전체 경로 제공(bind mounting)

```bash
docker run -v /data/mysql:/var/lib/mysql mysql

```

## Storage Driver

- Docker는 storage driver로 Layered Architecture 활성화 → image, container storage 관리
    - 레이어 아키텍처 유지, write 가능한 Layer 생성, 레이어 간 파일 이동 ..
- aufs, overlay, overlay2 .. → os마다 달라짐 → docker는 os에 따라 최적화된 storage driver 선택

# Volume Driver Plugins in Docker

- volume → volume driver plugins이 처리(storage driver x)
- default volume driver plugins → local
    - docker host에 볼륨 생성
    - `/var/lib/docker/volume` 에 저장
- 특정 볼륨 드라이버 선택 가능
    - docker container 실행 시 특정 볼륨 드라이버를 사용해 aws EBS에서 볼륨 프로비저닝 → container 생성 후 data가 cloud에 저장

# Container Storage Interface

## container runtime interface

- 과거 container runtime → containerd
- 다양한 컨테이너 런타임에서 작동, 확장 하고 특정 벤더에 종속되지 않기 위해 container runtime interface 개발
- k8s(오케트스레이션 솔루션)이 docker와 같은 container runtime과 통신하는 방법을 정의하는 표준 → 새 container runtime 개발 시 cri 표준을 따른다면 k8s source code 변경 없이 사용 가능

## container Network Interface

- interface 기반 → 다양한 네트워킹 솔루션 제공하기 좋은 환경
- cni 표준만 따르면 k8s source code 변경 없이 사용 가능

## container storage Interface

- interface 기반 → 여러 스토리지 솔루션 지원
- 하지만 csi는 쿠버네티스 전용 표준은 아님 → 범용 표준 → 따라서 모든 컨테이너 오케스트레이션 도구에서 플러그인을 통해 사용 가능

### CSI

- 컨테이너 오케트스트레이터가 호출할 원격 프로시저 호출 집합인 rpcs 정의
    - rpcs → 스토리지 드라이버에 의해 구현되어야 함
- 예시
    - Pod 생성 → k8s가 볼륨 생성 RPC 호출 + volume 등 세부 정보 전달 → storage driver는 RPC 구현, 요청 처리, storage array에 새 볼륨 프로비저닝 후 작업 결과 반환
    - volume 삭제 → k8s가 볼륨 삭제 RPC 호출 → storage driver는 storage array에서 볼륨 해제하는 구현체 필요

# Volumes

## Docker Volumes

- 일반적으로 container 사라지면 → data도 함께 사라짐 → 하지만 볼륨 마운트를 한다면 data 유지

## k8s Volumes

### single node cluster

- pod에 volume 할당 → data가 volume에 저장 → pod가 삭제되어도 데이터 유지

```yaml
apiVersion: v1
kind: Pod
metadata:
	name: random-number-generator
spec:
	containers:
	- image: alpine
		name: alpine
		command: ["/bin/sh", "-c"]
		args: ["shuf -i 0-100 -n 1 >> /opt/number.out;"]
	volumeMounts:
	- mountPath: /opt
		name: data-volume
	volumes:
	- name: data-volume
	hostPath:
		path: /data
		type: Directory
```

- container → /opt에 데이터 마운트, host → /data 디렉토리에 컨테이너 내부의 데이터 저장

### multi node cluster

- multi node cluster에서 볼륨 마운트.yaml을 사용한다면 → 모든 노드(host)의 똑같은 디렉토리에 데이터들 저장 → multi node cluster에서는 사용 권장 x
- 외부 스토리지(AWS EBS 등)에 pod의 데이터를 저장하도록 하자

# Persistent Volumes

## volumes

- 사용자가 pod.yaml에 volume storage를 직접 구성해야 함 → 대규모 클러스터에서 관리가 쉽지 않음

## persistent Volumes

- 관리자가 구성한 storage pool → 사용자는 persistent volume claime을 사용해서 pool에서 storage 선택

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
	name: pv-viol1
spec:
	accessModes: # 어디까지 허용할 것인지
	- ReadWriteOnce
	capacity: # 저장 용량
		storage: 1Gi
	hostPath: # prod 환경에서는 사용 권장 x
		path: /tmp/data
```

# Persistent Volume Claims

- persistent volme claims로 pod에서 할당받은 pv(storage)를 사용할 수 있음
- 관리자 → pv 생성, 사용자 → pvc를 생성해서 storage를 사용
- pv : pvc = 1:1 → 다른 pvc는 pv의 용량이 남아도 사용할 수 x
- pvc 요청 → 사용 가능한 pv가 없다면 → pvc는 pending 상태로 유지됨

## 할당 과정

1. pvc 생성
2. k8s는 pvc의 요청대로 충분한 용량을 가진 pv를 찾아서 → 바인딩
    1. 특정 volume를 사용하려면 label &selector를 사용해서 pv ↔ pvc 바인딩
3. pvc 삭제 
    1. default: pv 유지(pvc 재사용도 가능)
    2. 하지만 삭제할 수 있음
    3. rm -rf 옵션으로 삭제하는 것 만으로는 충분하지 않음(마운트 해제, 분리, 파일 시스템, 포맷, 스냅샷, 암호화 등) → 최근에는 storage class + CSI 드라이버를 통한 동적 프로비저닝으로 이식성, 보안 격차 해결

## PVC

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
	name: myclaim
spec:
	accessModes:
	- ReadWriteOnce
	resources:
		requests:
			storage: 500Mi
```

# Storage Class

## 정적 프로비저닝 볼륨

- 클라우드 프로바이드 서비스에서 disk 수동 프로비저닝 →  k8s cluster의 pv 파일 생성(이때 file 이름 = 클라우드 프로바이드 서비스에서 생성한 디스크 이름)

## 동적 프로비저닝 볼륨

- storage class를 활용해서 클라우드 프로바이더 서비스에서 storage 자동 프로비저닝 + claim 발생 시 바인딩
- pv에 대한 storage class 생성 → 이 storage를 pvc가 가져다가 사용 → 이 pvc에 pod의 데이터 저장(볼륨 마운트) → 따라서 pv가 더 이상 필요 없음!
- storage class로 원하는 유형을 지정해서 → 원하는 옵션(SSD, region 등)을 부여 → 이렇게 생성된 storage class를 pvc에 지정하면 pod의 데이터 영구 저장 가능
- 예시

```yaml
# storage-class.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
	name: google-storage
provisioner: kubernetes.io/gce-pd
```

```yaml
# pvc-def.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
	name: myclaim
spec:
	accessModes:
	- ReadWriteOnce
	storageClassName: google-storage
	resources:
		requests:
			storage: 500Mi
```
