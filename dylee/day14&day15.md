# Rolling Updates & Rollbacks

## Rollout & Versioning

1. first deployment → rollout trigger → new rollout이 new replicas 생성(revision 1)
2. container version update →  rollout trigger → new rollout이 new replicas 생성(revision 2)
    
    → 변경 사항 추적, 롤백
    

## Rollout Command

```bash
kubectl rollout status deployment -n <namespace-name>

kubectl rollout history deployment/<deployment>
```

## Deployment Strategy

1. Recreate
    - 한꺼번에 파괴하고 → 새로 만들면 새 버전 업데이트
    - 하지만 이 중간 과정에서 앱이 다운 → 사용자가 접근 x
2. Rolling Update(default)
    - 한 개씩 old version을 내리고, new verison을 업데이트
    - 애플리케이션이 끊기지 않고 업데이트 순차적으로 진행
    1. deployment yaml 수정 후 → `kubectl apply -f deployment-definition.yaml`  실행 → rolling update
    2. `kubectl set image <deployment-src> \ <image>` 실행 → rolling update 
        - 하지만 deployment-src에 있는 yaml은 update 되지 않는다

## Rolling Update 동작 방식

1. 사용자가 replicaset yaml 선언 → k8s가 Replica Set을 create & containers 스케줄링 + 배포
2. rolling update 전략에 따라 old replica set destroy
3. 만약 새 버전에 문제가 생긴다면? → 이전 버전 롤백
    - `kubectl rollout undo deployment <deployment-name>`

# Commands & Arguments

```bash
docker run --name ubuntu-sleeper ubuntu-sleeper #주어진 시간 동안 절전 모드로 전환
```

## dockerfile

- entrypoint: container 시작 시 실행되는 명령어
- cmd: args 로 전달되는 기본 매개변수
- entry point override
    - docker run 실행 → 새 entry point에서 container 구동

## pods

- spec.containers: container 시작 시 실행되는 명령어
- args: container에 전달되는 기본 매개변수
- command: docker의 entry point 명령어 → entry point override

# Environment Variables

## ENV Variables in k8s

### 1. Plain Key Value

- pod-definition에서 env field 사용하기

```yaml
apiVersion: v1
kind: Pod
metadata:
	naem: simple-wepapp
spec:
	containers:
	- name: simple-wepapp
		image: simple-wepapp
		ports:
			- containerPort: 8080
		env:
			- name: APP_COLOR 
				value: pink
```

### 2. ConfigMap & Secrets

- file로부터 주입

```yaml
env:
	- name: APP_COLOR
		valueFrom:
			configMapKeyRef: 
```

# ConfigMap

- 기존 env field 작성 방식 → 환경변수가 많아진다면 관리하기 쉽지 않음
- env를 중앙에서 관리 → ConfigMaps
- Pod 생성 → configMap을 Pods에 주입 → key-value를 env로 사용

**app-config.yaml**

```yaml
APP_COLOR: blue
APP_MODE: prod
```

**pod-definition.yaml**

```yaml
apiVersion: v1
kind: Pod
metadata:
	naem: simple-wepapp
spec:
	containers:
	- name: simple-wepapp
		image: simple-wepapp
		ports:
			- containerPort: 8080
		envFrom:
		- configMapRef:
				name: app-config #환경변수를 list로 만들어서 -> configMaps에 주입
```

## creating ConfigMaps

### 1. Imperative(명령형 방식)

```bash
kubectl create configmap <config-name> \
	--from-literal=<key>=<value>
	
kubectl create configmap app-config \
	--from-file=app_config.properties
```

- configMap.yaml 사용 x

### 2. Declarative(선언형 방식)

```yaml
kubectl create -f
```

- configMap.yaml 사용해서 생성

### viewing ConfigMaps

```bash
kubectl get configmaps #configmap 확인

kubectl describe configmaps #configmap 구성 데이터 확인

```

### env

- 단일 환경 변수로 주입할 수도 있고, 전체 데이터를 파일로 주입하거나, 볼륨으로 주입할 수 있음

# Secrets

- configMap → 평문으로 저장 → 암호를 저장한다면 문제가 생길수도
- secrets → 암호, key 를 인코딩, 해싱된 상태로 저장

## Creating Secrets

### 1. Imperative

- cmd line 이용해서 생성

```bash
kubectl create secret generic <secret-name> \ 
	--from-literal=<key>=<value>
```

- file directory 지정해서 secret 생성

```bash
kubectl create secret generic app-secret \
	--from-file=app_secret.properties
```

### 2. Declarative

1. secret.yaml 만들기

```yaml
apiVersion: v1
kind: Secret
metadata:
	name: app-secret
data:
	DB_Host: mysql
	DB_User: root
	DB_Password: passwd
```

1. 각각을 base64인코딩

```yaml
echo -n 'text' | base64
```

## Viewing Secrets

```bash
kubectl get secrets

kubectl describe secrets #무엇이 있는지는 보여주지만, 값 자체는 숨김

kubectl get secret app-secret -o yaml #해싱된 값도 볼 수 있음
```

## Decoding Secrets

```yaml
echo -n 'text' | base64 --decode
```

## Secrets in Pods

**app-secret.yaml**

```yaml
apiVersion: v1
kind: Secret
metadata:
	name: app-secret
data:
	DB_Host: <base-64 incode>
	DB_User: <base-64 incode>
	DB_Password: <base-64 incode>
```

**pod-definition.yaml**

```yaml
apiVersion: v1
kind: Pod
metadata:
	naem: simple-wepapp
spec:
	containers:
	- name: simple-wepapp
		image: simple-wepapp
		ports:
			- containerPort: 8080
		envFrom:
		- secretRef:
				name: app-secret #환경변수를 list로 만들어서 -> configMaps에 주입
```

- 단일 환경 변수 주입, 전체 secret.yaml 주입, file volume로 주입 모두 가능
