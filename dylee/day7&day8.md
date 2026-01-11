# Imperative vs Declarative

- Infrastructure → 관리를 위한 다양한 접근 방식 존재

## Imperative(명령형)

- 단계별 구성 → 인프라 구성
    - 예시: vm 프로비저닝 → nginx 설치 → 포트 설정(8080) → main path 설정 → git push → start server

## Declarative(선언적)

- 최종 인프라 상태 선언 → 시스템이 자동으로 구성
- IaC는 선언적 도구(Ansible, Terraform etc..)
    - 예시: 요구사항 선언하면 끝
        
        ```yaml
        VM Name: web-server
        Package: nginx
        Port: 8080
        Path: /var/www/nginx
        Code: Git Repo..
        ```
        
- 하지만 IaC를 이용해 인프라를 구성할 때, 절반만 구성된 상황이고, 남은 단계를 마저 구성하기 위해 같은 yaml또는 code를 제공한다면?
    
    →  이미 존재하는 system이 체크한 결과를 바탕으로 infra 구성 진행
    
    - ex) version upgrade → configuration file에서 update version 명시(직접 안해도 됨) → system이 처리

# Imperative & Declarative in k8s

## Imperative(명령형)

- kubectl로 선언하는 명령어들
    - create objects: run,create,expose로 새로운 object 생성
    - update objects: edit, scale, set
    
    **→ create, update는 yaml을 작성하지 않아도 됨 → 시험 환경에서 빠르다!**
    
    - 하지만 기능이 제한적이고 고급 명령어를 위해서는 복잡하며 이 명령어들은 한 번 실행되면 끝 → 사용자 Session에만 저장 → 큰 환경에서 debug가 쉽지 않다
    - 그렇다면 object configuration file로 관리한다면?
        - manifest, yaml..
        - craete 명령어로 yaml을 이용해 object 생성
        - yaml을 update했다면?
        1. kubetl edit로 object name 지정 → .yaml edit + save → 변경 사항 적용
            - 이 file은 k8s memory에 있는 Definition file(yaml x) → 따라서 kubectl edit로 pod update를 진행한다면 사용자가 작성한 .yaml엔 반영 x → 사용자 Session의 history에는 update했다는 기록이 있겠지만 yaml엔 없다!
        2. 따라서 `kubectl replace -f` 로 local version(직접 작성한 yaml file)을 편집해서 기록을 남기자
            - 만약에 object를 삭제하고 다시 만들고 싶다면?
                
                `kubectl replace —force -f nginx.yaml` 
                
                → 삭제하지 않고 create 명령어를 사용한다면 이미 Pod가 exists한다는 error 발생(system이 Pod가 존재한다고 인식 x)
                

```bash
kubectl run --image=nginx nginx
kubectl edit..
kubectl scale deployment nginx --replicas=5
kubectl set image deployment nginx nginx=nginx:1.18
kubectl create -f <file-name>.yaml
kubectl replace -f <file-name>.yaml
kubectl delete -f <file-name>.yaml
```

## Declarative(선언적)

- application의 기대 상태를 정의하는 file 집합(yaml) 생성
- k8s cluster 위에서 kubectl apply로 운영 → apply 명령어로 생성, 업데이트, 삭제 수행

```bash
kubectl apply -f nginx.yaml
```

- declarative approch는 configuration file + apply 명령어로 작업 수행
    1. object create & update → `kubectl apply -f <yaml-name>.yaml` 
        - directory도 설정 가능함
            
            `kubectl apply -f ./desktop/nginx <yaml-name>.yaml` 
            
        - system이 이미 pod가 exists한다는 것을 알고 있음 → pod만 update → pod exists 오류 발생 xExam Tips

# Exam Tips

## create objects

```bash
kubectl apply -f nginx.yaml
```

## update objects

```bash
kubectl apply -f nginx.yaml

kubectl run --image=nginx nginx

kubectl create deployment --image=nginx nginx

kubectl expose deployment nginx --port 80

kubectl edit deployment nginx

kubectl scale deployment nignx --replicas=5

kubectl set image deployment nginx nginx=nginx:1.18
```

# kubectl explain

## api-resource

- 모든 리소스 나열
- yaml 만들 때 apiVersion 모르겠다면 사용하자

```bash
kubectl api-resource
```

## explain

```bash
#pods 내 모든 최상위 필드 나열
kubectl explain pods

#pods.spec의 서브 필드
kubectl explain pods.spec

#pods의 yaml을 만들 때 작성해야 하는 field 나열
#pods 리소스에서 제공되는 모든 필드 목록 출력
kubectl explain pods --recursive
```

# kubectl apply

- apply 명령어 → local configuration file(last applied configuration file)을 고려해서 → 객체를 생성 또는 수정
1. local file을 이용해 object 생성 → local system에 저장

```bash
kubectl apply -f nginx.yaml
```

1. object 생성 후 k8s cluster에 local file과 유사한 Live Object Configuration 생성 → k8s memory에 저장
2. local file이 json 형식으로 변환 → last applied configuration으로 저장(객체의 업데이트에 대해, 마지막으로 적용된 구성 저장) → k8s cluster live-object 설정에 주석으로 저장되어 있음
3. 3가지 파일(local file, last applied configuration, live objeect configuration)을 비교해 live object에 어떤 변화가 필요한지 파악

```bash
#yaml
...
spec:
	containers: 
	- name: nginx-container
		image: nginx:1.19

kubectl apply -f nginx.yaml

```

1. local file 이 nginx:1.18에서 nginx:1.19로 업데이트
2. live object configuration과 비교 → new value로 update
3.  last applied configuration(json)도 최신 버전으로 업데이트
- last applied configuration이 필요한 이유?
    - local file에서 field 삭제 → kubectl apply → last applied configuration에는 local file에 삭제된 field가 존재하지만, live object configuration에서는 해당 field가 삭제된 상태
    - 따라서 last applied configuration을 확인한다면 기존과 비교해 local file에서 어떤 Field가 삭제되었는지 파악할 수 있음(디버깅)

# Manual Scheduling

- node에서 pod를 수동으로 스케줄링 하는 방법!
- 

## How Scheduling Works

```yaml
apiVersion: v1
kind: Pod
metadata:
	name: nginx
spec:
	containers:
	- name: nginx
		image: nginx
		ports:
		- containerPort: 8080
	#k8s 가 자동으로 추가
 	nodeName: node02
```

1. scheduling을 원하는 pod가 있는지 찾기
    1. scheduler는 모든 Pod를 점검해 nodeName이 설정되어 있지 않은 pod를 찾는다
2. 어떤 노드에서 schedling 할 지 결정하기
    1. 스케줄링 알고리즘을 실행해 pod의 적절한 node를 식별한다
3. 스케줄링 실행 → pod를 Node에 바인딩
    1. 바인딩 한 node위에 node name 속성을 설정해 pod scheduling

## 스케줄러가 없다면?

- pod → pending
- 따라서 직접 노드에 pod를 할당해야 함 → pod 생성 시 yaml의 nodeName field에 직접 설정해 줘야 함
- 만약 pod가 이미 생성되어 있는데, 이 Pod를 Node와 바인딩 하고 싶다면?
    - k8s는 pod의 nodeName 속성을 수정할 수 없다
    - 따라서 binding object를 만들어 post 요청을 보내야 함
    
    ```bash
    apiVersion: v1
    kind: Binding
    metadata:
    	name: nginx
    target:
    	apiVersion: v1
    	kind: Node
    	name:
    	
    	curl --header "Content-Type:application/json" --request POST --data '{"apiVersion":"v1", "kind": "Binding" ...} http://$SERVER/api/v1/namespaces/default/pods/$PODNAME/binding/
    ```
    

# Labels&Selectors

- 어떤 기준으로 그룹화, 필터링을 할 것인가?

## Labels & Selector in k8s

- 등장 배경
    - pods, replicas, deployments, service.. 는 모두 다른 object → 서비스가 확장될수록 cluster에 엄청나게 많은 object들이 생성 → type, application, function 별로그룹화, 필터링 등이 필요

## Labels

- app, function 등에 따라 label을 붙임
- 그 후 선택할 때 특정 object를 필터링 하는 조건 지정
- k8s에서 label을 지정하는 방법?
    
    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
    	name: nginx
    	labels:
    		app: App1
    		Function: Front
    spec:
    	containers:
    	- name: nginx
    		image: nginx
    		ports:
    		- containerPort: 8080
    	#k8s 가 자동으로 추가
     	nodeName: node02
    ```
    
- label이 붙은 pod를 select 하는 방법
    
    ```bash
    kubectl get pods --selector app=App1 <app=label.name>
    ```
    
- k8s object는 내부적으로 label, selector를 사용해 서로 다른 객체를 연결함
- label&selector
    
    ```yaml
    apiVersion: apps/v1
    kind: ReplicaSet
    metadata:
    	name: webapp
    	#replicaSet의 pods
    	labels:
    		app: App1
    		Function: Front-end
    spec:
    	replicas: 3
    	selector:
    		matchLabels:
    			app: App1
    		#pod에 설정된 label
    		template:
    			metadata:
    				labels:
    					app: App1
    					function: Front-end
    	
    ```
    
    - label을 붙이고,
    - selector로 ReplicaSet의 pods를 그룹화
    - pod - replicaSet의 label이 일치한다면 replicaSet가 성공적으로 생성된다
- label&selector in service
    
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
    		app: app!
    		type: front-end
    ```
    

## annotations

- label & selector → object를 그룹화, 선택
- annotations → object 정보 기록(버전, 이메일, 등)

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
	name: webapp
	#replicaSet의 pods
	labels:
		app: App1
		Function: Front-end
	annotations:
		buildversion: 1.34
spec:
	replicas: 3
	selector:
		matchLabels:
			app: App1
		#pod에 설정된 label
		template:
			metadata:
				labels:
					app: App1
					function: Front-end
	
```

# Taints & Tolerations

- node에서 scheduling 할 수 있는 pod에 대한 제한을 설정할 때 사용
- node에 taint를 설정하면, 관련된 tolerations이 있는 pod만 해당 node에 스케줄링 가능
- 예시
    
    배경: node1에 전용 리소스 존재 → 이 리소스가 필요한 podA만 node1에 배치하고 싶음
    
    1. node1에 taint를 붙임
    2. 기본적으로 pod는 tolerations가 없어, node1에 할당될 수 없음
    3. 해당 자원이 필요한 podA에 tolerations add
    4. scheduler가 podA를 node1에 배치 → 통과 → podA가 node1에서 구동됨

## Taints-Node

```bash
kubectl taint nodes node-name key=value:taint-effect
```

- taint-effect: pods가 taint를 tolerations 할 수 없을 때 어떤 effect가 일어날 지 정의한 것
    - 종류?
        - NoSchedule
        - PreferNoSchedule
        - NoExecute
            - node, 기존 pod에 새 pod 스케줄링 x

## Tolerations-PODs

```yaml
apiVersion: v1
kind: Pod
metadata:
	name: myapp-pod
	spec:
	containers:
	- name: nginx-container
		image: nginx
	tolerations:
	- key: "app"
		operator: "Equal"
		value: "blue"
		effect: "NoSchedule"
```

## Taint-NoExecute

- node에 taint 적용 → tolerations이 없는 스케줄링된 pods는 해당 node에서 killed
- tolerations이 설정된 Pod가 꼭 taint가 적용된 node에만 올릴 수 있는 것은 아님!

## Taint-MasterNode

- scheduler는 master node에 pod를 스케줄링 하지 x
    - k8s cluster 가 처음 설정될 때, master node에 자동으로 taint 설정 → master node에는 어떤 pod도 scheduling 되지 못한다!(application workload는 master node에 배포하지 않는 걸 권장)
- 확인 방법?
    
    ```yaml
    kubectl describe node kubemaster | grep Taint
    ```
