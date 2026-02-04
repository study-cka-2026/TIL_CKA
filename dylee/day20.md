# Backup & Restore

## 무엇을 백업해야 하는지?

- etcd cluster
    - cluster 관련 정보 저장소 → deployment, pod, service definition files
- persist volume
    - 영구 저장소

## k8s cluster에서 resource 생성법

1. 명령형 → `kubectl create`
2. 선언적 방식 → yaml 만들고(정의 파일) → `kubectl apply -f ~.yaml`
    
    → git 같은 곳에 저장 해 놓고 유사시 백업하자!
    

## Backup - Resource Configs

1. kube-api server에 접근해 모든 리소스 저장
    1. kubectl 사용
    2. api 서버 직접 접근
    - ex) 백업 스크립트 작성
        - 모든 pod, deployments를 api 서버에서 가져오기
        
        ```bash
        kubectl get all -all-namespaces -o yaml > all-deploy-services.yaml
        ```
        
2. thrid-party service 사용하기
    - velero

## Backup - etcd

- etcd cluster → cluster 내에서 생성된 자원에 대한 모든 정보 저장 → etcd 자체를 백업한다면 → k8s cluster 전체 백업 끝
- etcd cluster
    - master node에 호스팅 되어 있고
    - `--data-dir=/var/lib/etcd` ← 백업 디렉토리
    - snapshot
        - `etcdctl \ snapshot save snapshot.db` 로 백업 상태 확인
- 백업 방법
    
    ```bash
    # 백업 상태 확인
    etcdctl \ snapshot save snapshot.db
    
    # k8s api server 중지
    systemctl stop kube-apiserver
    
    # 백업파일 경로로 etcd snapshot 복원
    etcdctl \ 
    snapshot restore snapshot.db \ #snapshot db file
    --data-dir /var/lib/etcd-from-backup # /var/lib/etcd에 새 데이터 디렉토리 생성
    
    # etcd backup & cluster 구성 초기화 & member 구성
    
    # etcd configuration file을 새 데이터 디렉터리를 사용하도록 설정
    etcd.service
    
    # service demon 다시 불러오고 + etcd 서비스 재시작
    # cluster 백업
    systemctl damon-reload
    
    systemctl restart etcd
    
    systemctl start kube-apiserver
    ```
    
    - 이때 인증서 경로 꼭 snapshot에 같이 저장해야 함!
        
        ```bash
        etcdctl \ 
        snapshot restore snapshot.db \ #snapshot db file
        --cacert = 경로
        --cert = 경로
        --key = 경로
        ```
        
- 하지만 관리형 cluster → etcd에 접근을 못할수도..
    - 그렇다면 kube api server를 query 해서 백업 고고
