- DAY 20 ~ DAY 23의 Certificates API 까지 완강
- 실습 부분은 간간히 보충하겠습니다..(+늦어도 설 연휴동안은 밀린 실습 완료하기..!!)
# Security Primitives

## Secure Hosts

- 기본 세팅
    - root access 비활성화
    - password 기반 인증 비활성화
    - ssh key 기반 인증만 사용할 수 있도록 설정

## Secure k8s

### 1. kube-api server access 제어

- kubectl ↔ kube-api-server에 access 해서 cluster의 작업 수행
    
    → kube-api-server에 대한 접근 제어가 필요
    
- 접근 제어 → 인증
    - Static Token file
    - Certificatesthird
    - Thrid-party service
        - External Authentication providers
    - Service Accounts
- 접근 제어 → 권한 → 권한에 따라 작업 정의
    - RBAC Authorization
    - ABAC Authorization
    - Node Authorization
    - Webhook Mode

## 2. TLS Certificates

- cluster 구성 요소와의 통신 → TLS 암호화를 통해 보안 유지
    - kube-apiserver와 etcd, kubelet, kube-proxy, kube-scheduler,kube-controller-manager 사이의 통신은 암호화됨
- cluster 내 application 간의 통신
    - network access를 설정해 pods가 cluster 내의 모든 pods에 access 할 수 있음

# Authentication

- k8s cluster → 여러 개의 node로 구성
    - cluster access 관리자, 애플리케이션 테스트 및 배포 관리자, 애플리케이션 사용자, thrid-party-application 등이 접근

# Authentication

## accounts

- 관리 목적으로 k8s cluster에 액세스 하는 사용자를 관리
    - 관리자, 개발자, 로봇
- k8s는 기본적으로 사용자 계정을 관리하지 않음
    - 사용자 관리를 위해서는 외부 서비스나 사용자 세부 정보, 인증서 등이 필요
    - k8s cluster에서 사용자를 생성하거나 사용자 목록을 볼 수 없음
- 하지만 service account에서는 k8s api를 사용해 서비스 계정 생성, 관리 할 수 있음 → k8s api server가 관리(kubectl or api access)
    
    ```bash
    kubectl create serviceaccount sa1
    kubeectl get serviceaccount
    ```
    
    - 요청 → 요청 인증 → kube-apiserver 요청 처리
        - 인증 매커니즘
            1. static token file → 사용자 이름, 토큰 저장
                1. user-token-detail.csv에 group 정보에 대한 열 추가 → 사용자를 특정 그룹에 할당
                2. token file을 kube-api-server에 전송 → 인증하는 동안 token에 권한 주기(Berear)
                3. 비번은 권장 x
                4. kubeadm 설정에서 → token file을 통한 인증 작업 시 이 file에 대한 볼륨 마운트도 고려해야 함!
            2. Certificates로 인증
                1. TLS Certifications
            3. 타사 인증 프로토콜 사용 → Ldap, Keberos

# TLS Certificates

## Certifcations

- 사용자가 웹 서버에 액세스 → TLS 인증서는 사용자 서버 간 통신이 암호화되어 있고, 해당 서버가 실제 서버임을 보장
    - 암호화를 통해 → 전송되는 데이터를 암호화해 → 네트워크 스니핑 방지
- 데이터 암호화 방법
- 숫자 + key → 데이터 암호화, 복호화
1. 대칭 암호화
    - 데이터 암호화, 복호화 할 때 동일한 key 사용
    - 수신자 간 key 교환 필요 → 공격자는 key도 스키핑 해서 복호화 가능
2. 비대칭 암호화
    - 데이터 암호화, 복호화 → 개인 키, 공개 키 쌍 사용
    - 공개키로 암호화된 데이터 → 개인키로만 열 수 있음

## SSH 인증

1. 공개 키, 비공개 키 생성(ssh keygen)
2. server의 ssh에 public Lock → 하지만 이 Lock은 비공개 키만 풀 수 있음
3. 다른 사용자가 접근 → private key + public Lock 생성 → 서버에 public Lock 등록
4. 대칭 키 전송, 복호화는 private key로

## 인증서

- 인증서는 누구나 발급할 수 있다 → 따라서 인증서를 가지고 암호화된 통신을 하는 것 만으로는 해당 서버가 믿을만한 서버인지 알 수 없음
    - 그렇다면 신뢰할 수 있는 인증서인지 확인하려면?
        - 서명을 확인하자 → 이건 브라우저에서 자동으로 해줌
    - 합법적인 인증서를 생성하고, 권한이 있는 사람이 서명한 인증서를 받는 방법?
        - CA기관 이용
            - 이 CA가 위조된 CA인지 아닌지 알아내는 방법?
            - ca의 공개 key, 비공개 key 이용
                - ca 비공개 key로 인증서에 서명
                - 브라우저에 ca 공개 key 내장 → ca 공개 key를 사용하여 인증서가 ca를 통해 직접 서명했는지 확인
            - 내부망 → private CA를 활용하자

# TLS in k8s

1. openSSL로 개인 key 생성 → key로 인증서 서명 요청
    1. 서명 요청 시 이름 지정 → Kubenetes.CA
    2. ca로 인증서에 대한 서명 진행
    3. 이제 CA는 개인 key와 루트 인증서 파일을 갖게 됨!
2. 클라이언트 인증서 생성
    - 관리자 → 관리자 사용자가 k8s cluster에서 인증하는 데 사용
        1. openSSL로 관리자의 개인 key 생성
        2. CSR 생성 → kube admin 지정
        3. openSSL -09로 서명된 인증서 생성
            1. ca 인증서, key 지정해서 인증 → 클러스터 내에서 유효한 인증서 생성 → 인증서 = ID, key = 비밀번호
        4. 그룹 상세 정보 추가
            - system master group
                - k8s cluster 내에 존재하는 관리 권한이 있는 그룹

## kube Scheduler, kube controller Manager, kube-proxy

- k8s control plane 구성 요소 → 이름 앞에 system prefix 붙여야 함
- 모두 Kube-api server 접속 → 클라이언트 인증서 사용

## 인증서 유스케이스

- 클러스터 관리를 위해 관리자 권한이 있는 인증서 사용
- kube api server 에서 권한이 필요한 작업하는  rest api 호출하는 방법
    1. option으로 키, ca 인증서 설정
        1. curl + ca 인증서
    2. 모든 매개 변수를 Kube config configuration으로 이동
        1. 이 file에 api 서버 엔드포인트, 세부정보, 사용할 인증서 지정
- 쿠버네티스에서는 다양한 구성 요소가 서로를 확인하려면 모두 CA의 루트 인증서 사본이 필요 → 인증서가 있는 서버, 클라이언트 구성할 때 마다 CA 루트 인증서도 지정해야 함

## etcd Servers

- etcd server는 고가용성 환경에서 여러 server에 거쳐 cluster로 배포 가능
- 인증 과정
    1. etcd용 인증서 생성 → etcd server 시작하면서 인증서 지정
    2. cluster 의 구성 요소 간의 통신을 위한 추가 peer 인증서 생성

## kube-api server

- 모든 작업, 통신은 kube-api server을 통해 이루어짐 → alias 필요
1. kube-api 서버용 인증서 생성 → openSSL 생성 후 이름 지정 → 인증서 서명 요청 생성 시 configuration file을 option으로 전달
    1. 적절한 이름(ex/kubernetes)을 설정해 이 이름으로 kube-api server를 참조하는 사람들이 kube-apiserver에 연결해서 작업할 수 있도록 해야 함
    2. openSSL의 alt_names에 api-server가 사용하는 모든 DNS이름 + IP 주소 포함
    
- key 지정 위치
    - etcd, kubelet 서버와 client로 통신하는 동안 api client의 인증서를 고려해야 함 → 이 인증서는 kube-apiserver의 실행 파일 or 서비스 구성 파일에 전달됨
    1. CA file 전달 → 모든 component는 client를 확인하기 위해 CA 인증서가 필요
    2. TLS 인증서 옵션에서 API 인증서 제공
    3. CA 파일로 etcd 서버에 연결하는 데 사용되는 클라이언트 인증서 지정
    4. kube-api server client 인증서 지정
        - kubelet에 연결하는 데 사용

## kubelet Nodes

- kubelet서버는 각 노드에서 실행되는 https API server
- API서버가 노드 모니터링 → 이 node에서 예약할 pod에 관한 정보 전송하기 위해 통신하는 대상 → 따라서 클러스터의 각 node에 대해 key certificaties 쌍이 필요
    - 이때 인증서들 이름은 node의 이름에 따라서 지어짐
- 인증서 생성 시 kubelet configuration file에서 사용
    - root 인증서 지정 → kubelet의 node 인증서 제공
- kubelet이 kube-apiserver와 통신하는데 사용할 client 인증서 세트
    - kubelet이 Kube-apiserver에 인증하는 데 사용됨
    - api server의 적절한 권한 부여 필요 → 올바른 형식의 이름을 지정해야 함
        - format: system keyword + node + nodeName
        - 권한 부여: node를 systemNode group에 추가

## 전체 cluster의 모든 인증서에 대한 상태 확인

# View Certificate Details

- kubeadm 사용 시
    - cluster 생성, 구성하는 동안 모든 컴포넌트를 네이티브 서비스로 배포 → 이를 pod로 배포함

### kubeadm 프로비저닝 클러스터

- health check → 시스템에서 사용되는 모든 인증서 식별
1. k8s manifest 폴더에서 kube-apiserver 정의 파일 찾기
    - api 서버를 시작하는 데 사용되는 모든 인증서에 대한 정보가 있음
2. 각 인증서를 가져와 해당 인증서에 대한 정보 살펴보기
    - openSSL x509 → 인증서 파일 입력 → 인증서 디코딩 → kube-apiserver 인증서 확인 후 → 유효 기간 확인해서 만료일과 인증서 발급자 확인 + 유효성 확인
    - kubeadm은 쿠버네티스 CA 이름 = kubernetes

### Inspecting Service Logs

- kubeadm으로 cluster 구성 → 다양한 구성 요소가 pod로 배포됨 → 로그 보려면 kubectl + pod name으로 확인하면 됨
    - 하지만 api-server나 etcd가 다운되면 kubectl 명령어 사용 불가
    
    → 이땐 docker로 이동해서 로그를 가져와야 함
    
    - `crictl ps -a` → 모든 컨테이너 나열
    - `crictl logs container-id`  → `crictl ps -a` 로 확인한 컨테이너 id로 로그 확인

# Certificates API

## 인증서 관리 방법

1. CA 서버 + component에 대한 인증서 설정
2. 인증서 + key로 cluster 관리
3. 새 관리자 합류 → cluster access 필요
4. 새 관리자 개인 key 생성
5. 생성 요청을 관리자에게 보냄
6. 인증서 서명 요청을 CA 서버로 가져감
7. CA 서버 개인키 + effect(루트 인증서) → CA 서버에서 서명 → 새 인증서 생성
8. 새 인증서를 새 관리자에게 전달
9. 인증서 유효 기간 만료 시 → 새 CSR 생성 + CA 서명 받는 동일한 process 따름

## k8s certificates API

- CA
    - key - certificate 쌍
    - CA에 대한 접근 권한을 얻은 사용자 → k8s cluster에 대한 인증서에 서명할 수 있음
    - 이 파일은 안전한 서버에 저장해야 함
- RCA server
    - 인증서가 배치된 서버
    - master Node → 인증서 배치
- kubeadm
    - CA 파일 생성 + 마스터 노드에 저장
- 인증서 api를 이용해 자동화된 방법으로 인증서 서명 요청 관리 + 만료 시 교체
    1. 관리자가 master Node에 로그인
        - 사용자가 key 생성 → 인증서 서명 요청 생성 → 관리자에게 요청을 보냄
            
            ```bash
            openssl genrsa -out jane.key 2048
            
            openssl req -new -key jane.key -subj "/CN=jane" -out jane.csr
            ```
            
    2. 인증서 서명 요청 kubeapi object 생성
        - 관리자는 key를 받아 인증서 서명 요청 객체 생성(manifest로)
        
        ```bash
        apiVersion: certificates.k8s.io/v1
        kind: CerrtificatesSigningRequest
        metadata:
        	name: jane
        spec:
        	expirationSeconds: 600
        	usage:
        	- digital signature
        	- key encipherment
        	- server auth
        	request:
        		jane.car base 64 incodeing한 내용
        ```
        
    3. 이 요청을 cluster 의 관리자가 검토, 승인
        - `kubectl get scr` 로 인증서 서명 요청 확인
        - `kubectl certificate approve jane` 으로 인증서 승인
    4. 인증서 추출, 공유
        - 인증서 추출 → yaml 형태
        - `kubectl get csr jane -o yaml`

## 동작 방식

- k8s control plane → kube-api server, scheduler, etcd, controller-manager
- 인증 관련 작업 → controller-manager
    - csr-approving
    - csr-signing
    - 인증서 서명 → CCA 서버의 루트 인증서 + 개인 key 필요 → 컨트롤러 관리자 서비스 구성에 이를 지정할 수 있는 옵션이 있음
