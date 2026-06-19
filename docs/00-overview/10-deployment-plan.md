
Оглавление

- [**10. План развёртывания Kubernetes The Rough Way**](#10-план-развёртывания-kubernetes-the-rough-way)
- [**10.1. Подготовка нод (выполняется до установки Kubernetes)**](#101-подготовка-нод-выполняется-до-установки-kubernetes)
  - [**10.1.1. Создание виртуальных машин**](#1011-создание-виртуальных-машин)
  - [**10.1.2. Сетевая схема**](#1012-сетевая-схема)
  - [**10.1.3. Установка ОС (на всех нодах)**](#1013-установка-ос-на-всех-нодах)
    - [**Подготовка ядра для kube-proxy (IPVS)**](#подготовка-ядра-для-kube-proxy-ipvs)
  - [**10.1.4. LVM‑разметка (на этапе установки ОС)**](#1014-lvmразметка-на-этапе-установки-ос)
    - [Control Plane:](#control-plane)
    - [Worker Nodes:](#worker-nodes)
  - [**10.1.5. DNS**](#1015-dns)
- [**10.2. Подготовка PKI (выполняется только на admin‑wks)**](#102-подготовка-pki-выполняется-только-на-adminwks)
  - [**10.2.1. Создание Root CA**](#1021-создание-root-ca)
  - [**10.2.2. Создание Kubernetes CA**](#1022-создание-kubernetes-ca)
  - [**10.2.3. Создание etcd CA**](#1023-создание-etcd-ca)
  - [**10.2.4. Создание front-proxy CA**](#1024-создание-front-proxy-ca)
  - [**10.2.5. Создание ServiceAccount keys**](#1025-создание-serviceaccount-keys)
  - [**10.2.6. Генерация сертификатов для всех компонентов**](#1026-генерация-сертификатов-для-всех-компонентов)
  - [**10.2.7. Раскладка сертификатов по нодам**](#1027-раскладка-сертификатов-по-нодам)
- [**10.3. Развёртывание Control Plane (строгая последовательность)**](#103-развёртывание-control-plane-строгая-последовательность)
  - [**10.3.1. Установка бинарников Kubernetes и etcd**](#1031-установка-бинарников-kubernetes-и-etcd)
  - [**10.3.2. Развёртывание etcd‑кластера**](#1032-развёртывание-etcdкластера)
  - [**10.3.3. Развёртывание HAProxy + VRRP**](#1033-развёртывание-haproxy--vrrp)
  - [**10.3.4. Развёртывание kube-apiserver**](#1034-развёртывание-kube-apiserver)
    - [**Обязательные параметры ServiceAccount для kube-apiserver**](#обязательные-параметры-serviceaccount-для-kube-apiserver)
  - [**10.3.5. Развёртывание controller-manager и scheduler**](#1035-развёртывание-controller-manager-и-scheduler)
- [**10.4. Развёртывание Worker Nodes**](#104-развёртывание-worker-nodes)
  - [**10.4.1. Установка бинарников**](#1041-установка-бинарников)
    - [**Ограничение размера логов containerd**](#ограничение-размера-логов-containerd)
  - [**10.4.2. Конфигурация kubelet**](#1042-конфигурация-kubelet)
    - [**Примечание о механике TLS Bootstrap**](#примечание-о-механике-tls-bootstrap)
      - [**Вариант A — Полностью ручной выпуск сертификатов (наш вариант)**](#вариант-a--полностью-ручной-выпуск-сертификатов-наш-вариант)
      - [**Вариант B — Автоматический bootstrap через CSR (не используется в нашем проекте)**](#вариант-b--автоматический-bootstrap-через-csr-не-используется-в-нашем-проекте)
- [**10.5. Развёртывание сетевого стека**](#105-развёртывание-сетевого-стека)
  - [**10.5.1. Развёртывание kube-proxy (DaemonSet)**](#1051-развёртывание-kube-proxy-daemonset)
  - [**10.5.2. Развёртывание Calico (DaemonSet)**](#1052-развёртывание-calico-daemonset)
    - [**Важно: tolerations для Control Plane нод**](#важно-tolerations-для-control-plane-нод)
- [**10.6. Развёртывание сервисов кластера**](#106-развёртывание-сервисов-кластера)
  - [**10.6.1. CoreDNS**](#1061-coredns)
  - [**10.6.2. Ingress Layer**](#1062-ingress-layer)
  - [**10.6.3. Observability Stack**](#1063-observability-stack)
    - [**Примечание о способе развёртывания Observability Stack**](#примечание-о-способе-развёртывания-observability-stack)
  - [**10.6.4. Security Stack**](#1064-security-stack)
- [**10.7. Финальная проверка**](#107-финальная-проверка)
- [**10.8. Итог**](#108-итог)

---

Текущий файл:
📘 `docs/00-overview/10-deployment-plan.md`  

# **10. План развёртывания Kubernetes The Rough Way**

План развёртывания — это строгая последовательность инженерных шагов, необходимых для построения self‑hosted Kubernetes‑кластера.  

Документ разделён на:

- **подготовку нод**  
- **развёртывание Control Plane**  
- **развёртывание Worker Nodes**  
- **развёртывание сетевого стека**  
- **развёртывание сервисов кластера**  
- **финальную проверку**

---

# **10.1. Подготовка нод (выполняется до установки Kubernetes)**

## **10.1.1. Создание виртуальных машин**

Создаём:

- 3 × Control Plane  
- 2 × Worker Nodes  
- 1 × admin‑wks (рабочая станция для PKI и kubeconfig)

## **10.1.2. Сетевая схема**

Все ноды находятся в одном L2‑сегменте:

```
Node CIDR: 192.168.88.0/24
gateway:   192.168.88.2
VIP:       192.168.88.10
```

Control Plane:

```
cp‑01: 192.168.88.11
cp‑02: 192.168.88.12
cp‑03: 192.168.88.13
```

Worker Nodes:

```
worker‑01: 192.168.88.21
worker‑02: 192.168.88.22
```

## **10.1.3. Установка ОС (на всех нодах)**

- Ubuntu LTS  
- systemd  
- journald  
- chrony  
- swap off  
- sysctl для сети  
- iptables/nftables  
- conntrack 

### **Подготовка ядра для kube-proxy (IPVS)**

Для работы kube‑proxy в режиме IPVS необходимо заранее загрузить модули ядра.  
На всех нодах создаём файл:

```
/etc/modules-load.d/ipvs.conf
```

С содержимым:

```
ip_vs
ip_vs_rr
ip_vs_wrr
ip_vs_sh
nf_conntrack
```

И применяем sysctl‑параметры:

```
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
```

Без этих настроек:

- kube‑proxy может не перейти в IPVS,  
- Calico может не обрабатывать трафик,  
- Pod → Pod и Service → Pod могут не работать. 

## **10.1.4. LVM‑разметка (на этапе установки ОС)**

### Control Plane:

```
/var/lib/etcd
/var/lib/kubelet
/var/lib/containerd
/var/log
```

### Worker Nodes:

```
/var/lib/kubelet
/var/lib/containerd
/var/log
```

## **10.1.5. DNS**

DNS‑имена:

```
k8s-vip
api.k8s.local
cp-01.k8s.local
cp-02.k8s.local
cp-03.k8s.local
```

---

# **10.2. Подготовка PKI (выполняется только на admin‑wks)**

## **10.2.1. Создание Root CA**

```
root-ca.pem
root-ca-key.pem
```

## **10.2.2. Создание Kubernetes CA**

```
kubernetes-ca.pem
kubernetes-ca-key.pem
```

## **10.2.3. Создание etcd CA**

```
etcd-ca.pem
etcd-ca-key.pem
```

## **10.2.4. Создание front-proxy CA**

```
front-proxy-ca.pem
front-proxy-ca-key.pem
```

## **10.2.5. Создание ServiceAccount keys**

```
sa.key
sa.pub
```

## **10.2.6. Генерация сертификатов для всех компонентов**

- kube-apiserver  
- kube-controller-manager  
- kube-scheduler  
- kubelet (каждая нода)  
- kube-proxy  
- admin (kubectl)  
- etcd (server, peer, client)  
- front-proxy-client  

## **10.2.7. Раскладка сертификатов по нодам**

Control Plane:

```
/etc/kubernetes/pki/
```

Worker Nodes:

```
/var/lib/kubelet/pki/
```

---

# **10.3. Развёртывание Control Plane (строгая последовательность)**

## **10.3.1. Установка бинарников Kubernetes и etcd**

На всех CP‑нодах:

```
/usr/local/bin/kube-apiserver
/usr/local/bin/kube-controller-manager
/usr/local/bin/kube-scheduler
/usr/local/bin/kubelet
/usr/local/bin/kube-proxy
/usr/local/bin/etcd
/usr/local/bin/etcdctl
```

## **10.3.2. Развёртывание etcd‑кластера**

На всех CP‑нодах:

- конфигурация peer TLS  
- конфигурация client TLS  
- SAN с IP и DNS  
- data‑dir → `/var/lib/etcd`  
- initial cluster → cp‑01, cp‑02, cp‑03  
- запуск через systemd

**etcd должен быть поднят первым.**

## **10.3.3. Развёртывание HAProxy + VRRP**

На всех CP‑нодах:

- установка HAProxy  
- установка Keepalived  
- конфигурация VRRP (VIP 192.168.88.10)  
- health‑check HAProxy  
- запуск через systemd

## **10.3.4. Развёртывание kube-apiserver**

На всех CP‑нодах:

- TLS  
- SAN (VIP + IP + DNS)  
- etcd endpoints  
- service CIDR: `10.96.0.0/12`  
- cluster CIDR: `10.244.0.0/16`  
- front-proxy  
- audit‑логирование  
- запуск через systemd
  
 ### **Обязательные параметры ServiceAccount для kube-apiserver**

Современные версии Kubernetes требуют явного указания эмитента и ключа подписи ServiceAccount токенов.  
В systemd‑юните `kube-apiserver` должны быть указаны параметры:

```
--service-account-issuer=https://kubernetes.default.svc.cluster.local
--service-account-signing-key-file=/etc/kubernetes/pki/sa.key
--service-account-key-file=/etc/kubernetes/pki/sa.pub
```

Без этих флагов:

- токены ServiceAccount будут считаться недействительными,  
- системные Pod’ы (CoreDNS, Calico, kube-proxy) не смогут пройти аутентификацию,  
- кластер будет частично неработоспособен.
 

## **10.3.5. Развёртывание controller-manager и scheduler**

На всех CP‑нодах:

- TLS  
- kubeconfig  
- leader election  
- запуск через systemd

---

# **10.4. Развёртывание Worker Nodes**

## **10.4.1. Установка бинарников**

На worker‑нодах:

```
/usr/local/bin/kubelet
/usr/local/bin/kube-proxy
```

### **Ограничение размера логов containerd**

Чтобы предотвратить переполнение `/var/log`, необходимо включить ротацию логов контейнеров.  
В файле:

```
/etc/containerd/config.toml
```

в секции:

```
[plugins."io.containerd.grpc.v1.cri".container]
```

добавляем:

```toml
log_max_size = 10485760   # 10 MB
log_max_files = 5
```

Это гарантирует:

- ограничение роста логов Pod’ов,  
- предсказуемое использование LV `/var/log`,  
- отсутствие DiskPressure,  
- согласованность с архитектурой хранения (глава 08).

## **10.4.2. Конфигурация kubelet**

- CRI: containerd  
- TLS bootstrap (ручной)  
- kubeconfig  
- CNI: Calico  
- Node IP  
- запуск через systemd

### **Примечание о механике TLS Bootstrap**

В Kubernetes существует два подхода к выдаче сертификатов kubelet:

#### **Вариант A — Полностью ручной выпуск сертификатов (наш вариант)**  
- На admin‑wks генерируются уникальные сертификаты для каждого kubelet.  
- Для каждой ноды создаётся индивидуальный kubeconfig.  
- Сертификаты копируются на ноды вручную.  

Это соответствует философии **The Rough Way** и обеспечивает полный контроль над PKI.

#### **Вариант B — Автоматический bootstrap через CSR (не используется в нашем проекте)**  
- kubelet стартует с bootstrap‑токеном;  
- отправляет CSR в kube-apiserver;  
- администратор вручную выполняет `kubectl certificate approve`;  
- требуется включённый контроллер `csrsigning` в kube-controller-manager.

**Мы используем Вариант A.**  
Поэтому термин «TLS bootstrap» следует понимать как:

> **Ручной выпуск индивидуальных сертификатов для kubelet.**

---

# **10.5. Развёртывание сетевого стека**

## **10.5.1. Развёртывание kube-proxy (DaemonSet)**

- режим IPVS  
- kubeconfig подписан Kubernetes CA

## **10.5.2. Развёртывание Calico (DaemonSet)**

- Pure BGP (No-Encap)  
- PodCIDR per node  
- BGP mesh  
- без overlay  
- запускается на всех нодах, включая CP

### **Важно: tolerations для Control Plane нод**

Control Plane ноды по умолчанию имеют taint:

```
node-role.kubernetes.io/control-plane:NoSchedule
```

Чтобы Calico DaemonSet запускался на CP‑нодах, в его манифесте **обязательно должны быть указаны tolerations**:

```
tolerations:
  - key: "node-role.kubernetes.io/control-plane"
    operator: "Exists"
    effect: "NoSchedule"
```

Без этих tolerations:

- Calico не поднимется на CP‑нодах,  
- PodCIDR CP‑нод не будет анонсирован через BGP,  
- kube-proxy на CP‑нодах не сможет работать,  
- кластер окажется частично изолирован.

---

# **10.6. Развёртывание сервисов кластера**

## **10.6.1. CoreDNS**

- Deployment  
- Service IP: `10.96.0.10`

## **10.6.2. Ingress Layer**

- NGINX Ingress Controller  
- Deployment или DaemonSet

## **10.6.3. Observability Stack**

- node-exporter  
- metrics-server  
- kube-state-metrics  
- Prometheus  
- Grafana
  
### **Примечание о способе развёртывания Observability Stack**

Развёртывание Prometheus, Grafana, kube-state-metrics и alertmanager «голыми» манифестами — это:

- десятки ConfigMap,  
- множество ClusterRole/RoleBinding,  
- StatefulSet/Deployment,  
- ServiceMonitor/PodMonitor (если используется CRD),  
- сложная конфигурация.

Чтобы сохранить философию **The Rough Way**, рекомендуется:

- использовать **ванильные манифесты** из репозитория `prometheus-operator` (kube-prometheus),  
- либо, **как исключение для Day‑2 операций**, разрешается использовать Helm‑чарт kube-prometheus-stack.

Это не нарушает архитектурный подход, потому что:

- Observability — это не часть control plane;  
- Helm используется только как инструмент упаковки, а не как «магия»;  
- все компоненты остаются прозрачными и управляемыми.

## **10.6.4. Security Stack**

- RBAC  
- NetworkPolicy  
- kubelet authn/authz  
- sysctl ограничения  
- TLS everywhere

---

# **10.7. Финальная проверка**

- etcd health  
- API health  
- kubelet Ready  
- CNI Ready  
- BGP mesh established  
- Pod → Pod  
- Pod → Service → Pod  
- DNS  
- Ingress  
- HAProxy failover  
- kubeconfig  
- logs  
- metrics  

---

# **10.8. Итог**

План развёртывания Kubernetes The Rough Way:

- строго последовательный;  
- разделён по ролям нод;  
- согласован с архитектурой 03–09;  
- полностью self‑hosted;  
- без kubeadm;  
- без Helm;  
- с ручной PKI;  
- с HA API;  
- с Calico BGP;  
- с LVM;  
- с предсказуемой сетевой и дисковой архитектурой.

---
