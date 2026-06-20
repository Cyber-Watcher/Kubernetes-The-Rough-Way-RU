
Оглавление

- [11. Структура репозитория Kubernetes The Rough Way](#11-структура-репозитория-kubernetes-the-rough-way)
- [11.1. Верхний уровень репозитория](#111-верхний-уровень-репозитория)
- [11.2. docs/](#112-docs)
  - [11.2.1. `docs/00-overview/`](#1121-docs00-overview)
  - [11.2.2. `docs/01-install-cluster/`](#1122-docs01-install-cluster)
  - [11.2.3. `docs/02-deploy-apps/` (будущее)](#1123-docs02-deploy-apps-будущее)
  - [11.2.4. `docs/03-monitoring/` (будущее)](#1124-docs03-monitoring-будущее)
  - [11.2.5. `docs/04-operations/` (будущее)](#1125-docs04-operations-будущее)
- [11.3. `scripts/`](#113-scripts)
- [11.4. `manifests/`](#114-manifests)
- [11.5. `pki/` — PKI-шаблоны и конфиги](#115-pki--pki-шаблоны-и-конфиги)
- [11.6. `etcd/` — конфигурации etcd](#116-etcd--конфигурации-etcd)
- [11.7. `control-plane/` — конфиги control plane](#117-control-plane--конфиги-control-plane)
- [11.8. `workers/` — конфиги worker‑нод](#118-workers--конфиги-workerнод)
- [11.9. `networking/` — сетевые компоненты](#119-networking--сетевые-компоненты)
- [11.10. `storage/` — хранилище](#1110-storage--хранилище)


---


Текущий файл:

📘 `docs/00-overview/11-repo-structure.md`  

## 11. Структура репозитория Kubernetes The Rough Way

До завершения проекта это, пожалуй, самая часто меняемая часть.

---

## 11.1. Верхний уровень репозитория

```text
Kubernetes-The-Rough-Way-RU/
├── docs/               # Документация (архитектура + пошаговые гайды)
├── scripts/            # Скрипты для каждого этапа
├── manifests/          # YAML-манифесты
├── pki/                # PKI-шаблоны и конфиги
├── etcd/               # Конфигурации etcd
├── control-plane/      # Конфигурации control plane
├── workers/            # Конфигурации worker-нод
├── networking/         # Calico, kube-proxy, HAProxy, VRRP
├── storage/            # LVM, CSI, примеры PV/PVC
└── README.md
```

---

## 11.2. docs/

Документация делится на:

- **00-overview** — архитектура и концепции  
- **01-install-cluster** — реальные шаги по развёртыванию  
- далее — будущие разделы (deploy apps, monitoring, operations)

---

### 11.2.1. `docs/00-overview/`

Это то, что мы уже пишем — архитектурные главы, без команд:

```text
docs/00-overview/
├── 01-intro.md
├── 02-architecture.md
├── 03-control-plane.md
├── 04-controllers.md
├── 05-workers.md
├── 06-pki.md
├── 07-networking.md
├── 08-storage-architecture.md
├── 09-tech-stack.md
├── 10-deployment-plan.md
└── 11-repo-structure.md
```

---

### 11.2.2. `docs/01-install-cluster/`

Это **пошаговые инструкции**, строго соответствующие этапам из главы 10.

Каждый этап — отдельная папка с одинаковой внутренней структурой:

- `<stage>-steps.md` — пошаговый гайд  
- `systemd-units.md` — systemd-юниты, относящиеся к этапу  
- `checklist.md` — чек-лист проверки  
- `troubleshooting.md` — типовые ошибки и отладка  

```text
docs/01-install-cluster/
├── 00-infra/                   # Этап 0 — подготовка инфраструктуры
│   ├── 00-infra-steps.md
│   ├── systemd-units.md
│   ├── checklist.md
│   └── troubleshooting.md
│
├── 01-admin-wks/               # Этап 1 — подготовка admin-wks
│   ├── 01-admin-wks-steps.md
│   ├── systemd-units.md
│   ├── checklist.md
│   └── troubleshooting.md
│
├── 02-nodes/                   # Этап 2 — подготовка всех нод (OS, LVM, sysctl, IPVS)
│   ├── 02-nodes-steps.md
│   ├── systemd-units.md
│   ├── checklist.md
│   └── troubleshooting.md
│
├── 03-pki/                     # Этап 3 — PKI
│   ├── 03-pki-steps.md
│   ├── systemd-units.md
│   ├── checklist.md
│   └── troubleshooting.md
│
├── 04-etcd/                    # Этап 4 — etcd
│   ├── 04-etcd-steps.md
│   ├── systemd-units.md
│   ├── checklist.md
│   └── troubleshooting.md
│
├── 05-apiserver-ha/            # Этап 5 — kube-apiserver + HAProxy + VRRP
│   ├── 05-apiserver-ha-steps.md
│   ├── systemd-units.md
│   ├── checklist.md
│   └── troubleshooting.md
│
├── 06-controllers/             # Этап 6 — scheduler + controller-manager
│   ├── 06-controllers-steps.md
│   ├── systemd-units.md
│   ├── checklist.md
│   └── troubleshooting.md
│
├── 07-workers/                 # Этап 7 — worker nodes + kubelet
│   ├── 07-workers-steps.md
│   ├── systemd-units.md
│   ├── checklist.md
│   └── troubleshooting.md
│
├── 08-cni/                     # Этап 8 — Calico (CNI + BGP)
│   ├── 08-cni-steps.md
│   ├── systemd-units.md
│   ├── checklist.md
│   └── troubleshooting.md
│
├── 09-coredns/                 # Этап 9 — CoreDNS
│   ├── 09-coredns-steps.md
│   ├── systemd-units.md
│   ├── checklist.md
│   └── troubleshooting.md
│
├── 10-kube-proxy/              # Этап 10 — kube-proxy (IPVS)
│   ├── 10-kube-proxy-steps.md
│   ├── systemd-units.md
│   ├── checklist.md
│   └── troubleshooting.md
│
├── 11-ingress/                 # Этап 11 — ingress
│   ├── 11-ingress-steps.md
│   ├── systemd-units.md
│   ├── checklist.md
│   └── troubleshooting.md
│
├── 12-storage/                 # Этап 12 — storage
│   ├── 12-storage-steps.md
│   ├── systemd-units.md
│   ├── checklist.md
│   └── troubleshooting.md
│
├── 13-monitoring/              # Этап 13 — monitoring
│   ├── 13-monitoring-steps.md
│   ├── systemd-units.md
│   ├── checklist.md
│   └── troubleshooting.md
│
└── 14-operations/              # Этап 14 — operations
    ├── 14-operations-steps.md
    ├── systemd-units.md
    ├── checklist.md
    └── troubleshooting.md
```

---

### 11.2.3. `docs/02-deploy-apps/` (будущее)

```text
docs/02-deploy-apps/
├── 01-namespace-basics/
├── 02-deployments/
├── 03-services/
├── 04-ingress/
├── 05-configmaps-secrets/
└── 06-helm-basics/
```

---

### 11.2.4. `docs/03-monitoring/` (будущее)

- Prometheus  
- Grafana  
- Alertmanager  
- Loki  
- eBPF / BPF-based debug  

---

### 11.2.5. `docs/04-operations/` (будущее)

- backup/restore etcd  
- disaster recovery  
- обновление control plane и worker‑нод  
- обновление Calico, kubelet, kube‑proxy  
- отладка сети, BGP, IPVS  

---

## 11.3. `scripts/`

Скрипты зеркалят этапы установки кластера:

```text
scripts/
├── 00-infra/
├── 01-admin-wks/
├── 02-nodes/
├── 03-pki/
├── 04-etcd/
├── 05-apiserver-ha/
├── 06-controllers/
├── 07-workers/
├── 08-cni/
├── 09-coredns/
├── 10-kube-proxy/
├── 11-ingress/
├── 12-storage/
├── 13-monitoring/
└── 14-operations/
```

Внутри — `run.sh`, вспомогательные функции, всё, что можно запускать.

---

## 11.4. `manifests/`

Все YAML‑манифесты, на которые ссылаются шаги в `docs/01-install-cluster/`:

```text
manifests/
├── etcd/
├── apiserver/
├── controllers/
├── scheduler/
├── kubelet/
├── kube-proxy/
├── calico/
├── coredns/
├── ingress/
├── storage/
└── monitoring/
```

---

## 11.5. `pki/` — PKI-шаблоны и конфиги

Здесь важно вернуть детализацию, которую ты заметил:

```text
pki/
├── openssl/
│   ├── root-ca.cnf
│   ├── kubernetes-ca.cnf
│   ├── etcd-ca.cnf
│   └── front-proxy-ca.cnf
├── templates/
│   ├── kube-apiserver-csr.json
│   ├── kubelet-csr.json
│   └── etcd-csr.json
└── README.md
```

- `openssl/` — конфиги для генерации CA и сертификатов  
- `templates/` — CSR/JSON‑шаблоны  
- `README.md` — как этим пользоваться в рамках этапа 03‑pki

---

## 11.6. `etcd/` — конфигурации etcd

Тут тоже нужна конкретика, а не «папка ради папки»:

```text
etcd/
├── systemd/
│   └── etcd.service
└── config/
    └── etcd.conf.yml
```

- `systemd/etcd.service` — юнит, на который ссылается `docs/01-install-cluster/04-etcd/systemd-units.md`  
- `config/etcd.conf.yml` — базовый конфиг, который можно копировать/адаптировать

---

## 11.7. `control-plane/` — конфиги control plane

Аналогично — с явными файлами:

```text
control-plane/
├── systemd/
│   ├── kube-apiserver.service
│   ├── kube-controller-manager.service
│   └── kube-scheduler.service
└── config/
    ├── kube-apiserver.yaml
    ├── kube-controller-manager.yaml
    └── kube-scheduler.yaml
```

---

## 11.8. `workers/` — конфиги worker‑нод

```text
workers/
├── systemd/
│   └── kubelet.service
└── config/
    └── kubelet-config.yaml
```

---

## 11.9. `networking/` — сетевые компоненты

```text
networking/
├── calico/
│   ├── calico.yaml
│   └── bgp-examples/
├── kube-proxy/
│   └── kube-proxy-config.yaml
└── haproxy-keepalived/
    ├── haproxy.cfg
    └── keepalived.conf
```

---

## 11.10. `storage/` — хранилище

```text
storage/
├── lvm-layout-examples/
│   └── single-node-lvm-layout.md
└── csi-examples/
    ├── local-path-provisioner/
    └── generic-csi-driver/
```

---
