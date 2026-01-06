# Kubernetes TIL Study

> 목표: **30일 내 완강 + 1회독 실습 기반 학습**

이 레포지토리는 Kubernetes 전반을 **체계적으로 30일 동안 학습**하기 위한 개인 학습 플랜을 정리한 README입니다.
강의 진도 + 실습 + 복습을 명확히 분리하여, **개념 이해 → YAML 작성 → kubectl 숙련**까지 연결되는 구조로 설계되었습니다.

---

## 📅 전체 구조 요약

| 구분    | 내용                              |
| ----- | ------------------------------- |
| 강의일   | 새 섹션 진도 학습                      |
| 복습일   | 실습 반복 + YAML 작성 + kubectl 속도 향상 |
| 최종 목표 | 30일 내 완강 + 실습 기반 1회독 완료         |

---

## 🟦 1주차: Core Concepts & 기본 오브젝트

### Day 1 (01/01)

* Course Introduction
* Certification Overview
* Core Concepts Intro
* Cluster Architecture
* Docker vs containerd / Docker Deprecation

⏱ 약 60분

---

### Day 2 (01/02)

* ETCD for Beginners
* ETCD in Kubernetes
* Kube API Server
* Controller Manager / Scheduler / Kubelet / Kube Proxy

⏱ 약 65분

---

### Day 3 (01/03)

* Pods
* Pods with YAML
* Demo – Pods with YAML
* Lab – Pods (+ Solution 선택)

⏱ 약 70분

---

### Day 4 (01/05)

* ReplicaSets (Recap)
* Lab – ReplicaSets
* Deployments
* Lab – Deployments

⏱ 약 70분

---

### Day 5 (01/06)

* Services (ClusterIP / LoadBalancer)
* Lab – Services
* Namespaces
* Lab – Namespaces

⏱ 약 70분

---

### 🔁 Day 6 (복습)

* Pods / Deployment / Service / Namespace 개념 정리
* `kubectl run / create / apply` 반복 연습
* YAML 직접 작성 연습

---

## 🟦 2주차: Scheduling & Resource 관리

### Day 7
* Imperative vs Declarative
* `kubectl explain` / `kubectl apply`
* Lab – Imperative Commands

⏱ 약 65분

---

### Day 8

* Scheduling Intro
* Manual Scheduling
* Labels & Selectors
* Taints & Tolerations

⏱ 약 70분

---

### Day 9

* Node Selector / Node Affinity
* Resource Requests & Limits
* Lab – Resource Limits

⏱ 약 65분

---

### Day 10

* DaemonSets
* Static Pods
* Priority Classes

⏱ 약 65분

---

### Day 11

* Multiple Schedulers
* Scheduler Profiles
* Admission Controllers (2025 Updates 포함)

⏱ 약 70분

---

### 🔁 Day 12 (복습)

* Scheduling 관련 YAML 정리
* taint / affinity / resource limits 손코딩

---

## 🟦 3주차: Logging · App Lifecycle · Autoscaling

### Day 13

* Logging & Monitoring
* Monitor Cluster Components
* Application Logs

⏱ 약 55분

---

### Day 14

* Rolling Updates & Rollbacks
* Commands & Arguments
* Lab – Commands & Arguments

⏱ 약 70분

---

### Day 15

* Environment Variables
* ConfigMaps
* Secrets
* Lab – Env / Secrets

⏱ 약 70분

---

### Day 16

* Multi-container Pods
* InitContainers
* Self-healing Applications

⏱ 약 55분

---

### Day 17 

* Autoscaling

  * HPA
  * VPA
  * In-place Resize

⏱ 약 60분

---

### 🔁 Day 18 (복습)

* `rollout` / `scale` / `env` / `secret` 실습
* Autoscaling 개념 정리

---

## 🟦 4주차: Cluster · Security · Storage · Network

### Day 19

* Cluster Maintenance
* OS Upgrade
* Cluster Upgrade
* Backup & Restore

⏱ 약 70분

---

### Day 20

* Security Primitives
* Authentication
* TLS & Certificates

⏱ 약 70분

---

### Day 21

* RBAC / ClusterRole / RoleBinding
* Service Accounts
* Image Security

⏱ 약 65분

---

### Day 22

* Storage Overview
* PersistentVolume (PV)
* PersistentVolumeClaim (PVC)
* StorageClass
* Lab – Storage

⏱ 약 65분

---

### Day 23
* Networking Basics
* Pod Networking
* Service Networking
* DNS / CoreDNS

⏱ 약 70분

---

### 🔁 Day 24 (복습)

* RBAC + PV/PVC + Network YAML 정리
* `kubectl auth can-i` 반복 연습

---

## 🟦 5주차: Ingress · Helm · Kustomize · Troubleshooting

### Day 25

* Ingress
* Gateway API (2025)

⏱ 약 60분

---

### Day 26

* Cluster Design
* kubeadm
* Helm (Intro ~ Lifecycle)

⏱ 약 70분

---

### Day 27

* Kustomize

  * Patch
  * Overlay
  * Component

⏱ 약 70분

---

### Day 28

* Troubleshooting

  * Application
  * Control Plane
  * Node
* JSONPath

⏱ 약 65분

---

### Day 29

* Mock Exam 1
* Solution Review

⏱ 약 70분

---

### 🔁 Day 30 (최종 복습)

* Mock Exam 2 또는 3
* 자주 틀리는 유형 정리
* 시험용 `kubectl` 치트키 정리
