# Services

## What is Services

- 애플리케이션 내외의 다양한 구성 요소 간 통신을 가능하게 하는 것으로, client → node로 요청을 매핑하는 중간 장치
    - 애플리케이션(pod) ↔ 애플리케이션(pod)
    - 애플리케이션 ↔ 사용자
- 애플리케이션 내 마이크로서비스 간의 느슨한 결합을 가능하게 함
- pod, replicasets, deployments 처럼 하나의 object
- node 내부의 virtual server
    - clutser 내부에서 자체 IP 주소를 가짐 → 이 ip 주소가 service의 clutserIP
- 하나의 service내에서 여러 개의 port 매핑을 가질 수 있음

## use-cases

- 외부 사용자가 애플리케이션에 접근하는 방법?
1. 기존 방식
    - client가 pod IP로 요청을 보내도, pod는 별도의 network에 있기 때문에 접근 x
    - 따라서 애플리케이션에 접근하려면, k8s node ssh 접속 → cluster 내부에서 node로 직접 접근해야 함(curl & GUI가 있다면 브라우저 실행 → Pod 웹 페이지 접근)
2. Service 이용한 방식
    - node에 ssh로 접속하지 않고도 웹 서버에 접근 가능!
    - Service는 client → node로 요청을 매핑

## Service Types

1. NodePort
    - service가 node의 port에서 internal pod를 접근 가능하게 하는 곳
2. ClutserIP
    - cluster 내에서 가상 IP를 생성 → cluster 내부에서의 통신
        - pod ↔ pod
        - service ↔ service
3. LoadBalancer
    - 부하분산

## NodePort

1. TargetPort
    - web Server가 설치된 pod의 port(80)
    - 10.244.0.2(pod ip)
2. port
    - service가 요청을 전달하는 포트(80)
    - 10.160.1.12(service ip)
3. NodePort
    - node 자체의 port로 웹 서버에 접속 가능
    - NodePort range: 30000 ~ 32627
- 운영 환경에서 한 개의 service에 매핑되는 여러 개의 pod가 있다면?
    - service 생성 → 동일한 label이 붙은 pod 찾음 → service는 외부 요청을 전달할 여러 개의 pod를 모두 자동으로 endpoint로 선택
    - 일종의 LB역할
- pod가 여러 worker node에 분산되어 있다면?
    - clutser 모든 node에 걸쳐 service 생성한 후 target port 매핑
    - cluster 내 모든 node에 nodePort 존재 → cluster 내 worker node의 ip를 사용해 동일한 ㅇapplication에 접근 가능

## service-definition.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
	name: myapp-service
spec:
	type: NodePort #service의 유형
	ports:
	- targetPort: 80 #service의 요청이 전달되는 Port
		port: 80 #service object의 port
		nodePort: 30008
	selector:
		app: myapp
		type: front-end
```

```bash
kubectl create -f service-definition.yaml
kubectl get services
#웹 서버 접근
curl http://server-ip:node-port 

```

- **port → 필수값**
- targetPort 제공 x → port와 동일하다 간주
- label & selector로 pod ↔ service 연결
    - selector: labels 목록 제공

# NameSpaces

- namespace안에 service, deployment, pods 존재
- k8s가 자동으로 생성하는 것들 → 사용자가 실수로 삭제,수정하지 않도록..!
    - default ns
    - 네트워킹 솔루션
    - DNS service
    - kube-system ns
    - kube-public ns
        - 모든 사용자에게 제공되어야 할 자원 생성
- namespace를 생성해 환경을 관리하자
- namespace는 정책에 대한 집합을 가질 수 있음
    - 리소스 할당
        - 이 ns 에 해당하는 object들은 정해놓은 리소스를 넘어가지 않도록 설정 가능

## DNS

- service 생성 → DNS 자동으로 추가
- service의 DNS로 mysql에 접근하는 방법?
- service name.namespace.svc.cluster.local
    - cluster.local → k8s domain

```bash
mysql.connect("db-service.dev.svc.cluster.local)"
```

## namespace use-cases

```bash
kubectl get pods --namespace=kube-system
kubectl create -f pod-def.yaml --namespace=dev
kubectl get pods --all-namespaces
#현재 context 식별 -> 원하는 namespace 설정
kubectl config set-context $(kubectl config current-context) --namespace=dev
```

```yaml
apiVersion: v1
kind: Pod
metadata:
	name: myapp-service
	namespace: dev
	labels:
		app: myapp
		type: front-end
spec:
	containers:
	- name: nginx-container
		image: nginx
```

- create ns
    1. yaml
    
    ```yaml
    apiVersion: v1
    kind: namespace
    metadata:
    	name: dev
    
    kubectl create -f namespace-dev.yaml
    ```
    
    1. terminal
    
    ```bash
    kubectl create namespace dev
    ```
    

## Resource Quota

- ns 내 자원 limit → resource quota를 만들어 namespace 생성

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
	name: compute-quota
	namespace: dev
spec:
	hard:
		pods: "10"
		requests.cpu: "4"
		requests.memory: 5Gi
		limits.cpu: "10"
		limits.memory: 10Gi
```
