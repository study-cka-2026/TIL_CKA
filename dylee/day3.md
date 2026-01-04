# Pods

- k8s는 worker node에 직접 컨테이너 배포 x → pods에 container가 캡슐화 되어 있음
- pods = 애플리케이션의 단일 인스턴스, container과 1:1 관계
- pod에 여러 개의 컨테이너를 넣을 수 있지만, 일반적으로는 1 pod = 1 container
- 서비스 확장을 위해 더 많은 인스턴스가 필요하다면 ?
    - 새로운 pods 생성해서 scale-up
- helper container가 필요하다면?
    - 같은 Pod 내에 application container + hepler container
    - 두 컨테이너 → localhost로 서로 통신 가능,같은 storage 공유
    - application container ↔ helper container 연결된 것의 map을 유지해야 함
        - link, custom, volume 할당 등으로 이 컨테이너를 연결하고, volume을 Container 간에 공유해야 함
        - application kill → helper container도 kill , application deploy → helper도 deploy
        - k8s는 이 작업들을 처리해 줌
            - pods가 어떤 Container로 구성되어 있는지 정의하면 → pod내 container들은 동일 저장소, 같은 namespace, 같이 생성되고 삭제됨
- 실행 명령어
    
    ```bash
    #image docker hub에서 다운로드 -> pod 생성 -> container 배포
    kubectl run nginx --image nginx
    
    #cluster 내 pods 확인
    kubectl get pods
    ```
    

# Pods with YAML

- k8s의 yaml → replicas, pods, deployments,service, etc.. 생성할 때 사용
- yaml & pod 생성

```yaml
apiVerison: #string-value
kind: #string-value
metedata:
	name: myapp-pod #dictionary
	labels:
		app: myapp
		type: front-end

spec: #dictionary, list/array
	containers:
		- name: nginx-container
			image: nginx
```

```bash
kubectl create -f pod-definition.yaml #pod 생성
kubectl get pods #pod 목록 확인
kubectl describe pod myapp-pod #pod의 디테일한 정보 확인
```

- 필수 filed → apiVersion, kind, metadata, spec
- apiVersion: object 생성할 때 사용하는 k8s api 버전
- kind: object 종류
    
    
    | kind | version |
    | --- | --- |
    | Pod | v1 |
    | Service | v1 |
    | ReplicaSet | apps/v1 |
    | Deployment | apps/v1 |
- metadata: object에 대한 data → 들여쓰기 주의!, name, label 등 specify한 것만 지정 가능
    - name: pod의 이름
    - labels: metadata안의 dictionary
        - type: pods grouping
- spec: specification, object와 관련된 추가 정보를 k8s에 제공
    - containers: list속성 → pods안에 여러 컨테이너를 가질 수 있기 때문

# Pods with YAML

```yaml
#vim pod.yaml

#pod.yaml
apiVersion: v1
kind: Pod
metadata:
	name: nginx
	labels:
		app: nginx
		tier: frontend
spec:
	containers: 
	- name: nginx
		image: nginx
	#pod 안의 additional container
	- name: busybox
		image: busybox

kubectl apply -f pod.yaml #새 object 생성
kubectl get pods #pods list 
kubectl describe pod nginx #pod information
```

- 들여쓰기 → 2칸으로 고정 ㄱㄱ
