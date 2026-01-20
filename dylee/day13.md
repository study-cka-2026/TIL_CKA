# Logging & Monitoring

- 초기 모니터링 → Heapster
- 현재 → 경량화된 Metrics Server
    - cpu, memory에 대한 리소스 메트릭만 제공 → 장기 저장/dashboard는 별도의 솔루션으로 고고
- **Monitoring(메트릭)**
    - CPU/메모리 같은 수치, kubectl top, HPA 입력
- **Logging(로그)**
    - 애플리케이션 stdout/stderr, kubectl logs, 장애 원인 파악

## Metrics Server

- 1 cluster → 1개의 metrics server
- node, pod에서 metrics 검색, 집계 → Memory에  저장 → in-memory solution이라 disk 에 저장 x → 따라서 메트릭 저장을 위해 Prometheus, Elastic Stack, DataDog, dynatrace 등의 솔루션 사용

### node에서 pod에 대한 Metrics 생성 방법

- kubelet → control-plane API server watch → 해당 신호에 원하는 상태를 맞춤(control loop)
- kubelet의 cAdvisor → Pod에서 metrics 검색, kubelet API 를 통해 metrics 노출 → 메트릭 서버에서 사용할 수 있게 함
    - cAdvisor(노드 내부) → kubelet(요약/노출) → Metrics Server(스크랩/집계) → Metrics API(aggregated API) → kubelet top/HPA
    - Metrics Server는 kube-api server에 [`metrics.k8s.io`](http://metrics.k8s.io) Metrics API 제공

## Viewing Metrics

```bash
#node의 cpu, memory 사용량 확인
kubectl top node 
#k8s pod 성능 메트릭
kubectl top pod -A #전체 ns에 대한 메트릭
kubectl top pod --containers #컨테이너 단위에 대한 메트릭
```

# Application Logs

```bash
#pods의 실시간 로그 확인
kubectl logs -f event-simulator-pod <container-name>

#컨테이너 재시작 시 이전 컨테이너 로그 확인
kubectl logs pod -c <container> --previous
--since=time #시간 제한
--tail=50 #라인 제한
```

## Use Cases

- Metrics Server: **빠른 리소스 상태 확인 + HPA 동작에 필요**
- Prometheus 등: **장기 저장/알림/대시보드/쿼리**
- kubectl logs: **Pod 단위 즉시 확인**
- 중앙 로그 스택: **여러 노드/파드 로그의 통합 검색 및 보관**
