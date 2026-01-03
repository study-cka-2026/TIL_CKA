# DAY1 - 0101

## Certification

- 시험 시간: 3시간, k8s 공식문서 참고 가능!

## Cluster Architecture

### 1. k8s intro

- k8s란 많은 instance를 쉽게 배포하고 활성화할 수 있도록 자동화된 방식으로 컨테이너화 하여 애플리케이션을 호스팅 하는 오케스트레이션 도구
- control plane과 worker node로 이루어져 있음

**Control plane(Master Node)**

- k8s clutser를 전체적으로 관리하고, scheduler, controller, etcd, kube-api server로 구성되어 있다
- scheduler →  컨테이너가 어떤 node에 배치될지 정하고 taint, tolerations, node affinity rules 등을 고려해 애플리케이션&컨테이너 스케줄링
- controller →  k8s cluster를 감시, 모니터링. node-controller, replication controller, controller-manager로 구성
- key-value store인 etcd → configuration, secrets 등을 저장
- kube-api server → 사용자가 관리 작업을 할 때 사용하는 api를 활성화 해. clutser 내 모든 작업 오케스트레이트

**Worker Node**

- container 구동하여 application 호스팅. kubelet, kube-proxy로 구성됨
- kubelet
    - kube-api server의 request를 받아 컨테이너 관리 → container, pods 수 보장
    - kube-api server는 kubelet 을 통해 node의 상태 모니터링
- kube-proxy
    - container과 service간 통신을 가능하게 함

**container runtime**

- 컨테이너 실행 소프트웨어로, 모든 node 들에 설치되어야 함
- 종류: docker, containred, rkt
- 배경
    1. k8s가 처음에는 도커를 오케스트레이션 하기 위해 만들어짐
    2. k8s가 컨테이너 오케스트레이션으로 인기를 얻자 다른 컨테이너 런타임들의 등장 
    3. cri(container runtime interface) interface의 등장 → oci standard 충족하면 k8s의 컨테이너 런타임으로 사용 가능하도록 정책을 세움
    4. 하지만 도커는 cri 표준을 지원하도록 만들어진 것이 아님 → 도커는 k8s 이전에 등장했기 때문..
    5. 그래서 k8s는 docker shim 도입해 cri 외부에서 docker를 계속 지원
        - docker = docker CLI + docker API + 이미지 빌드 + 컨테이너 런타임 +deamon → 무겁다
    6. 따라서 docker에서 runtime만 분리한 경량화된 CLI인 containred 등장 → k8s는 containerd를 CRI 플러그인을 통해 사용
        - 그러면 컨테이너 디버깅은?
            - 초기 → ctr
                - containred 개발자용 디버깅 도구
                - ctr로 기본 컨테이너 관련 작업 수행 가능 → image pull & run
                - 하지만 사용자 친화적x(컨테이너 디버깅을 위해 만들어졌기 때문)
            - 현재 → crictl
                - Kubernetes 관점에서 디버깅
                - CRI 표준을 구현한 모든 런타임과 호환 (containerd, CRI-O 등)
                - Pod, Container, Image를 Kubernetes 방식으로 다룸
                - 운영자/관리자용 도구
                - cri 호환 시스템과 상호작용하는 command line utility
                - crictl로 pod를 만들 수 있지만, kubelet은 이 pod를 관리하지 않아서(kubelet은 kubelet을 통해서 만들어진 pod의 수만 유지/관리) 해당 pod는 eviction됨
                - crictl의 연결 순서
                    - dockershim.sock → containred.sock → crio.sock → cri-dockerd.sock순으로 연결됨
                - Runtime Endpoint → crictl이 어떤 runtime과 통신할지 지정하는 소켓 경로
                    - 같은 node에 containerd, cri-o가 둘 다 있다면 → crictl이 어떤 runtime과 통신할지 명시해줘야 함

# DAY2 - 0102

## ETCD

### etcd 란?

- 분산, 신뢰할 수 있는 key-value 저장소로 간단하고, 안전하고 빠르다

### key-value store란?

- database
    - schema 기반 Table → 새 정보를 추가할 때 마다 전체 table이 영향을 받는다
    - 복잡한 쿼리가 필요하고, 구조화된(structured)환경에 적합
- document store
    - 데이터를 json 형태로 저장
    - 한 document 변경 → 다른 document에 영향 x
    - schema가 유연함
    - 반구조화된 환경에 적합
- key-value store
    - 정보를 key-value 쌍으로 저장
    - value → 다양한 형태 저장 가능
    - schema가 없고, 복잡한 query 지원하지 않음 → 오직 key로만 조회 가능
    - 성능이 매우 빠르고 유연하며, schema 변경으로 인한 breaking이 없다 → 간단하고 빠른 조회에 최적화됨
    - etcd가 key-value store에 형태
- etcd 설치
    - 2379 → 클라이언트 통신용 (kubectl, API server,pod 등)
    - 2380 → etcd 서버 간 통신용(클러스터링)

### **k8s에서의 etcd**

- version
    - v2.0 → raft 알고리즘 재설계 → 초당 1000회 이상의 쓰기 지원
    - v3.1
        - api 버전 변경 → etcd 명령어도 변경
        - 트랜잭션 지원
    - v3.3
        - cncf project로 전환
    - v3.5
        - 2021.11 cncf graduated
- etcd는 k8s clutser에 관한 정보들을 저장
    
    → node, pod ,config, secrets, accounts, roles, binding …등등
    
- kubectl get으로 얻어지는 모든 정보는 etcd server에서 나오고, etcd server update 후 cluster에 변경 사항이 반영됨(node 추가, pod 배포,replicas 배포)
- etcd in HA Environment
    - cluster 내에 여러 개의 master node → 여러 개의 etcd 인스턴스 분산 → etcd 인스턴스가 서로 인식하도록 매개변수 설정 필요
- etcd 설치 by kubeadm
    - kubeadm이 etcd서버를 pod로 배포 → pod 내 etcd 제어 utility로 etcd db 탐색 가능
    - `kubectl exec etcd-matser -n kube-system etcdctl get / --prefix -keys-only`

### kube API Server

- cluster 에서 동작 방식
    - kube-control command → kube control utility가 실제로 kube api server에 도달
- 역할
    - etcd cluster에서 데이터를 가져와 요청한 정보 반환 (kubectl get nodes)
    - pod 생성
        1. `kubectl create deployment nginx --image=nginx --replicas=3`
        2.  kubectl → kube-apiserver (HTTP POST 요청)
        3. API Server: 인증 (Authentication) → 사용자가 누구인지 인증서, token 확인
        4.  API Server: 인가 (Authorization) → 이 사용자의 작업 권한 확인(RBAC 확인)
        5. API Server: 검증 (Admission Control) → 요청이 정책을 준수하는지 확인
        6. API Server → etcd에 Deployment 객체 저장 → etcd 서버 정보 업데이트
        7. API Server → kubectl 응답 반환(유저에게 pod가 생성되었다는 사실을 업데이트)
            1. 이 시점에는 아직 Pod가 없고 etcd에 저장만 됨
        8. Deployment Controller → API Server
            - etcd 변경 감지 →  새 Deployment 발견
            - ReplicaSet 생성 요청
        9. API Server → etcd에 ReplicaSet 저장
        10. ReplicaSet Controller  → API Server
            - etcd 변경 감지 → 새 ReplicaSet 발견
            - Pod 생성 요청
        11.  API Server → etcd에 Pod 3개 저장
            - Status: Pending (nodeName 필드가 비어있음)
        12. Scheduler
            - etcd 변경 감지: nodeName이 없는 Pod 3개 발견
            - Filtering: 후보 노드 선정
            - Scoring: 점수 매기기 → 최종적으로 배치할 worker node 선정
        13. Scheduler → API Server
            - Binding 정보 전달
        14. API Server → etcd 업데이트
            - Status: Pending → Scheduled
        15. Kubelet@worker
            - etcd 변경 감지:  노드에 할당된 Pod 발견
        16. Kubelet → CRI-O/containerd
            - 컨테이너 생성 요청
        17. CRI-O → runc
            - 실제 컨테이너 프로세스 생성
        18. Kubelet → API Server
            - Pod상태 업데이트: Running
        19. API Server → etcd 최종 업데이트
        Status: Scheduled → Running

### Controller Manager / Scheduler / Kubelet / Kube Proxy

- controller manager
    - k8s에서 컨트롤러 관리
    - 컨트롤러 → 다양한 상태를 지속적으로 모니터링하는 프로세스
    - 전체 시스템을 설정해놓은 기능으로 유지하기 위해 노력
    - node controller → 노드 상태 모니터링, 애플리케이션 계속 실행하기 위한 actions 수행 → kube-api server를 통해 node 상태 확인(node monirot period=5)
        - mode monitor grace period = 40s
        - pod eviction timeout = 5m
    - replication controller
        - replicas sets 모니터링 → set 안의 pod 수 유지
        - pod 죽으면 다시 만들어짐
    - controller → k8s cluster 어디에 위치?
        - 이 single process → k8s-controller-manager라는 단일 Process로 묶여 있음
    - 설치 + 확인
        - 컨트롤러 커스터마이즈 가능
        - 컨트롤러 작동x 존재 X 커스터마이즈 가능
        - kube-admin-tool → kube controller 관리자 Pod 로 배포 → kube-system ns에 존재
        - /etc/kubernetes/manifests/kube-controller-manager.yaml에서 확인 가능
        - /etc/kubernetes/manifests/kube-controller-manager.service에서 옵션 검사 가능
- kube scheduler
    - pod와 node 스케줄링
    - 어떤 포드가 어떤 노드에 배치될지 결정 → 실제 배치는 kubelet이 함
    - 특정 기준에 따라 pod의 resource 요구량이 다를 수 있음 → kube scheduler가 최적의 노드를 찾음 → 해당 worker node에 배치 결정
        1. 스케줄러가 pod의 profile에 맞지 않는 worker node 필터링(부족한 cpu memory..)
        2. 노드 정렬 → rank nodes → 적절한 노드 선정 → 직접 커스텀 + 스케줄러 작성 가능
    - Resource requirements and limits
    - tains and tolerations
    - node selectores/affinity
    - kubeadm으로 설치 → kube-system ns에 shceduler pods 존재
- kubelet
    - 해당 node를 k8s cluster에 등록
    - worker node에 pod load 명령 → container runtime 요청(cri-o, containred)
    - pod , container 모니터링
    - kube api 서버로 어쩌고..
    - 항상 Worker node에 Kubelet설치해야 함(kubedam 딸깍 x)
- kubeproxy
    - k8s clsuter 내부의 proxy는 서로 닿을 수 있음 → cluster에 networking solution 배포함으로써 가능
    - pod network → cluster 내 모든 노드에 걸쳐 있는 내부 가상 네트워크 → Pod 간 통신
    - 같은 cluster 내 배포된 application pods (a)와 db pods(b) 통신 과정
        - a → b service이름을 사용해 db pods에 접근
        - svc → 실제 svc가 아니므로 pod network에 접근 x → 컨테이너가 아니고, 쿠버네티스 메모리에만 존재하는 가상 컴포넌트 → 서비스가 clsuter 내 모든 node에서 접근 가능한 이유? → kube-proxy 덕분
        - kube-proxy
            - k8s cluster 각 노드에서 실행되는 프로세스
            - 새 svc를 찾아서 → 생성될 때 마다 각 노드에 적절한 규칙을 만들어 pod로 트래픽 전달
            - iptables 규칙으로 구현 → 클러스터 내 iptables 규칙을 만들어 트래픽 전달
            - 서비스 ip로 들어옴 → 실제 pod ip로 전달
            - 설치 → kubeadm
            - daemonset으로 배포 → clutser 각 노드에 항상 단일 pod 배치
    
- Kube API Server
    - kube-control command → kube control utility가 실제로 kube api server에 도달
    - 요청 인증, 검증 → etcd cluster에서 데이터를 가져와 요청한 정보 반환 (kubectl get nodes)
    - pod 만들기
        - 요청 인증, 검증 → etcd 서버 정보 업데이트 → 유저에게 pod가 생성되었다는 사실을 업데이트 → scheduler은 api 서버 모니터링하며 노드가 없는 새 pod 인식 → 새 pod를 배치할 node 식별 + api 서버로 이를 다시 전달 → api 서버가 etcd clutser 내 정보 업데이트 → api 서버는 해당 정보를 해당 worker node의 Kubelet에 전달 → kubelet은 node에서 Pod 생성 + 컨테이너 런타임 엔진에 배포 지시 → 배포 완료 후 kubelet은 상태를 api 서버에 업데이트 + api 서버도 etcd cluster에 있는 data update
        - kube-api server만 etcd data store와 직접 상호작용 함
        - 다른 구성 요소들은 api를 사용
        - k8s architecture 구성 요소 → 모두 인증서를 가지고 있음
            - 인증, 암호, 보안
        - etcd-servers → etcd 서버의 위치 지정 → kube-api server와 etcd 서버에 연결
    - kube admin tool → kube 관리자가 kube api 서버를 pod로 배포(master node의 ns에..)
        - kubectl get pods -n kube-system
        - cat /etc/kubernetes/manifests/kube-apiserver.yaml → 옵션 확인 가능
