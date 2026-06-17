
Оглавление

- [**1. Архитектура кластера**](#1-архитектура-кластера)
- [**1.1. Общая концепция архитектуры**](#11-общая-концепция-архитектуры)
- [**1.2. Node сеть**](#12-node-сеть)
- [**1.3. Node DNS domain (`k8s.local`)**](#13-node-dns-domain-k8slocal)
- [**1.4. Cluster DNS domain (`cluster.local`)**](#14-cluster-dns-domain-clusterlocal)
- [**1.5. Подсети Kubernetes**](#15-подсети-kubernetes)
  - [**1.5.1. Pod CIDR**](#151-pod-cidr)
  - [**1.5.2. Service CIDR**](#152-service-cidr)
- [**1.6. VIP и HA архитектура**](#16-vip-и-ha-архитектура)
- [**1.7. Список всех машин**](#17-список-всех-машин)
- [**1.8. Роли машин**](#18-роли-машин)
  - [**1.8.1. admin-wks**](#181-admin-wks)
  - [**1.8.2. Control Plane**](#182-control-plane)
    - [Компоненты Control Plane:](#компоненты-control-plane)
  - [**1.8.3. Worker Nodes**](#183-worker-nodes)
- [**1.9. Таблица IP + DNS + роли**](#19-таблица-ip--dns--роли)
- [**1.10. Диаграмма архитектуры (ASCII)**](#110-диаграмма-архитектуры-ascii)


---

Текущий файл:

📘 `docs/00-overview/01-architecture.md`  

# **1. Архитектура кластера**

Архитектура — это фундамент Kubernetes The Rough Way.  
Здесь мы фиксируем:

- физическую сеть  
- доменную модель  
- подсети Kubernetes  
- список всех машин  
- роли нод  
- VIP и HA  
- сетевую модель взаимодействия  

Это — основа, на которой будет строиться весь кластер.

---

# **1.1. Общая концепция архитектуры**

Кластер состоит из:

- **1 рабочей станции администратора (admin-wks)**  
- **3 control plane нод (stacked etcd)**  
- **2 worker‑нод**  
- **1 виртуального IP (VIP)**  

Все машины находятся в одной физической сети VMware:

```
192.168.88.0/24
Gateway: 192.168.88.2
```

Control plane построен по схеме:

- **stacked etcd** (etcd на тех же нодах, что и apiserver)  
- **HA API через VIP + keepalived + haproxy**  
- **scheduler и controller-manager на всех CP‑нодах (leader election)**  

Worker‑ноды выполняют Pod’ы и обеспечивают сетевую связность.

---

# **1.2. Node сеть**

Node сеть — это единственная L2/L3‑сеть, в которой живут все виртуальные машины (ноды):

```
192.168.88.0/24
Gateway: 192.168.88.2
DNS: 1.1.1.1, 8.8.8.8
```

Все ноды имеют статические IP.

---

# **1.3. Node DNS domain (`k8s.local`)**

Этот домен используется для:

- сертификатов kube-apiserver  
- сертификатов etcd  
- kubelet → apiserver  
- kubectl  
- haproxy/keepalived  
- /etc/hosts  

Примеры:

```
cp-01.k8s.local
worker-01.k8s.local
api.k8s.local
admin-wks.k8s.local
```

Это **внешний DNS‑мир**, не связанный с Kubernetes DNS.

---

# **1.4. Cluster DNS domain (`cluster.local`)**

Это внутренний сервисный домен Kubernetes.

Используется CoreDNS для сервисов:

```
kubernetes.default.svc.cluster.local
coredns.kube-system.svc.cluster.local
```

Это **внутренний DNS‑мир**, полностью изолированный от node DNS.

---

# **1.5. Подсети Kubernetes**

Kubernetes использует две виртуальные подсети:

---

## **1.5.1. Pod CIDR**

```
10.244.0.0/16
```

Назначение подсетей:

- worker-01 → `10.244.1.0/24`  
- worker-02 → `10.244.2.0/24`  

Эти подсети создаёт Calico.

---

## **1.5.2. Service CIDR**

```
10.96.0.0/12
```

Примеры:

- API service → `10.96.0.1`  
- CoreDNS → `10.96.0.10`  

Service IP — виртуальные, существуют только внутри kube-proxy.

---

# **1.6. VIP и HA архитектура**

Для обеспечения отказоустойчивости API используется:

- **VIP:** `192.168.88.10`  
- **DNS:** `api.k8s.local`  
- **keepalived** — VRRP, обеспечивает MASTER/BACKUP  
- **haproxy** — балансировка между apiserver’ами  

Все клиенты (kubelet, kubectl, контроллеры) подключаются к:

```
https://api.k8s.local:6443
или
https://192.168.88.10:6443
```

---

# **1.7. Список всех машин**

| Хост | IP | DNS | Роль | CPU | RAM | Disk |
|------|------|------|------|------|------|------|
| admin-wks | 192.168.88.9 | admin-wks.k8s.local | PKI, kubectl | 1 | 1GB | 20GB |
| cp-01 | 192.168.88.11 | cp-01.k8s.local | etcd-1, apiserver-1, haproxy, keepalived MASTER | 2 | 2GB | 20GB |
| cp-02 | 192.168.88.12 | cp-02.k8s.local | etcd-2, apiserver-2, haproxy, keepalived BACKUP | 2 | 2GB | 20GB |
| cp-03 | 192.168.88.13 | cp-03.k8s.local | etcd-3, apiserver-3, haproxy, keepalived BACKUP | 2 | 2GB | 20GB |
| worker-01 | 192.168.88.21 | worker-01.k8s.local | kubelet, containerd | 2 | 4GB | 30GB |
| worker-02 | 192.168.88.22 | worker-02.k8s.local | kubelet, containerd | 2 | 4GB | 30GB |

---

# **1.8. Роли машин**

## **1.8.1. admin-wks**

- центр PKI  
- генерация сертификатов  
- генерация kubeconfig  
- kubectl  
- хранение документации  

---

## **1.8.2. Control Plane**

Каждая CP‑нода запускает:

- etcd  
- kube-apiserver  
- kube-scheduler  
- kube-controller-manager  
- haproxy  
- keepalived  

### Компоненты Control Plane:

| Компонент | Реплики | Активность |
|-----------|----------|------------|
| etcd | 3 | все активны (Raft) |
| kube-apiserver | 3 | все активны |
| scheduler | 3 | 1 лидер |
| controller-manager | 3 | 1 лидер |
| haproxy | 3 | все активны |
| keepalived | 3 | 1 MASTER, 2 BACKUP |

---

## **1.8.3. Worker Nodes**

Каждая worker‑нода запускает:

- kubelet  
- containerd  
- kube-proxy  
- calico-node  

---

# **1.9. Таблица IP + DNS + роли**

```
192.168.88.9   admin-wks   admin-wks.k8s.local
192.168.88.10  k8s-vip     api.k8s.local
192.168.88.11  cp-01       cp-01.k8s.local
192.168.88.12  cp-02       cp-02.k8s.local
192.168.88.13  cp-03       cp-03.k8s.local
192.168.88.21  worker-01   worker-01.k8s.local
192.168.88.22  worker-02   worker-02.k8s.local
```

---

# **1.10. Диаграмма архитектуры (ASCII)**

```
                        ┌──────────────────────────────┐
                        │         Node сеть            │
                        │      192.168.88.0/24         │
                        │   Gateway: 192.168.88.2      │
                        └──────────────────────────────┘
                                      │
                    ┌────────────────────────────────────────┐
                    │       admin-wks (PKI, kubectl)         │
                    │    192.168.88.9  admin-wks.k8s.local   │
                    └────────────────────────────────────────┘
                                      │
                        ┌────────────────────────────────┐
                        │   VIP (keepalived + haproxy)   │
                        │         192.168.88.10          │
                        │         api.k8s.local          │
                        └────────────────────────────────┘
                                      │
          ┌─────────────────────────────────────────────────────────┐
          │            CONTROL PLANE (stacked etcd)                 │
          │                                                         │
          │   cp-01 192.168.88.11 cp-01.k8s.local                   │
          │      etcd-1, apiserver-1, haproxy, keepalived MASTER    │
          │                                                         │
          │   cp-02 192.168.88.12 cp-02.k8s.local                   │
          │      etcd-2, apiserver-2, haproxy, keepalived BACKUP    │
          │                                                         │
          │   cp-03 192.168.88.13 cp-03.k8s.local                   │
          │      etcd-3, apiserver-3, haproxy, keepalived BACKUP    │
          │                                                         │
          └─────────────────────────────────────────────────────────┘
                                    │
            ┌─────────────────────────────────────────────────┐
            │                 WORKER NODES                    │
            │                                                 │
            │   worker-01 192.168.88.21 worker-01.k8s.local   │
            │   worker-02 192.168.88.22 worker-02.k8s.local   │
            │                                                 │
            └─────────────────────────────────────────────────┘
```

---

Следующий файл:

📘 `docs/00-overview/02-components.md`  
**2. Компоненты Kubernetes (Big Picture)**

