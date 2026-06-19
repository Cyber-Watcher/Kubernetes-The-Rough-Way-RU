Оглавление

- [**3. Control Plane — детальный обзор**](#3-control-plane--детальный-обзор)
- [**3.1. kube-apiserver**](#31-kube-apiserver)
    - [Функции:](#функции)
    - [Особенности в нашем кластере:](#особенности-в-нашем-кластере)
- [**3.2. etcd**](#32-etcd)
    - [Функции:](#функции-1)
    - [Особенности в нашем кластере:](#особенности-в-нашем-кластере-1)
    - [Почему stacked etcd?](#почему-stacked-etcd)
- [**3.3. kube-scheduler**](#33-kube-scheduler)
    - [Функции:](#функции-2)
    - [Особенности:](#особенности)
- [**3.4. kube-controller-manager**](#34-kube-controller-manager)
    - [Особенности:](#особенности-1)
- [**3.5. haproxy**](#35-haproxy)
    - [Функции:](#функции-3)
    - [Особенности:](#особенности-2)
- [**3.6. keepalived**](#36-keepalived)
    - [Функции:](#функции-4)
    - [Особенности:](#особенности-3)
- [**3.7. kubelet (на CP‑нодах)**](#37-kubelet-на-cpнодах)
    - [Функции:](#функции-5)
    - [Особенности:](#особенности-4)
    - [Почему это важно](#почему-это-важно)
  - [но они **полноценные участники сетевой плоскости**.](#но-они-полноценные-участники-сетевой-плоскости)
- [**3.8. containerd (на CP‑нодах)**](#38-containerd-на-cpнодах)
    - [Особенности:](#особенности-5)
- [**3.9. Почему kubelet и containerd не были в списке компонентов Control Plane**](#39-почему-kubelet-и-containerd-не-были-в-списке-компонентов-control-plane)
  - [kubelet и containerd — это компоненты узла (node components)](#kubelet-и-containerd--это-компоненты-узла-node-components)
  - [Почему kubelet и containerd есть на control plane?](#почему-kubelet-и-containerd-есть-на-control-plane)
- [**3.10. kube-proxy и Calico на Control Plane**](#310-kube-proxy-и-calico-на-control-plane)
    - [**kube-proxy на CP‑нодах**](#kube-proxy-на-cpнодах)
    - [**Calico (CNI + BGP) на CP‑нодах**](#calico-cni--bgp-на-cpнодах)
    - [**Почему Calico ставится на Control Plane ноды**](#почему-calico-ставится-на-control-plane-ноды)
    - [**Почему не eBPF**](#почему-не-ebpf)
    - [**Итог**](#итог)
- [**3.11. systemd‑юниты (НЕ static pods)**](#311-systemdюниты-не-static-pods)
    - [Почему это важно:](#почему-это-важно-1)
    - [Какие сервисы запускаются через systemd:](#какие-сервисы-запускаются-через-systemd)
- [**3.12. Итоговая схема Control Plane**](#312-итоговая-схема-control-plane)
- [**3.13. Итоговое разделение ролей**](#313-итоговое-разделение-ролей)
    - [**Компоненты Control Plane =**](#компоненты-control-plane-)
    - [**Компоненты Worker Node =**](#компоненты-worker-node-)
- [**Следующий файл:**](#следующий-файл)


---

Текущий документ:

📘 `docs/00-overview/03-control-plane.md`  

# **3. Control Plane — детальный обзор**

Control Plane — это мозг Kubernetes.  
Он принимает решения, управляет состоянием, планирует Pod’ы, обеспечивает API и хранит данные.

В Kubernetes The Rough Way мы строим **полностью отказоустойчивый Control Plane**, состоящий из:

- 3 нод (cp‑01, cp‑02, cp‑03)  
- stacked etcd  
- HA API через VIP + keepalived + haproxy  
- scheduler + controller-manager с leader election  
- apiserver на каждой ноде  
- systemd‑юнитов (НЕ static pods)  
- сетевого стека Calico BGP + kube-proxy (IPVS)  

Это архитектура, максимально приближенная к реальному bare‑metal production.

---

# **3.1. kube-apiserver**

kube-apiserver — центральная точка входа в Kubernetes.

### Функции:

- принимает все запросы (kubectl, kubelet, controllers, scheduler)  
- валидирует объекты  
- авторизует запросы  
- общается с etcd  
- обеспечивает REST API  

### Особенности в нашем кластере:

- запущен на **всех трёх CP‑нодах**  
- доступен через **VIP 192.168.88.10**  
- балансируется через haproxy  
- использует PKI, построенную на Root CA → Kubernetes CA  
- слушает на `6443/tcp`  
- SAN включает:
  - IP всех CP‑нод  
  - VIP  
  - DNS имена  
  - `kubernetes.default.svc.cluster.local`  

kube-apiserver — **единственный компонент, который общается с etcd**.

---

# **3.2. etcd**

etcd — распределённое хранилище состояния Kubernetes.

### Функции:

- хранит все объекты Kubernetes  
- обеспечивает консистентность через Raft  
- критически важен для HA  

### Особенности в нашем кластере:

- 3 ноды: etcd‑1, etcd‑2, etcd‑3  
- размещены на cp‑01, cp‑02, cp‑03  
- peer‑коммуникация **только по IP**  
- TLS с отдельным etcd CA (под Root CA)  
- данные хранятся в `/var/lib/etcd` (отдельный LV)  

### Почему stacked etcd?

- меньше moving parts  
- проще в обслуживании  
- меньше сетевых зависимостей  
- соответствует рекомендациям Kubernetes SIG Cluster Lifecycle  

Worker‑ноды **никогда** не обращаются к etcd напрямую.

---

# **3.3. kube-scheduler**

kube-scheduler выбирает ноду для каждого Pod.

### Функции:

- анализирует ресурсы  
- учитывает taints/tolerations  
- affinity/anti-affinity  
- topology spread  
- выбирает оптимальную ноду  

### Особенности:

- запущен на всех CP‑нодах  
- работает в режиме **leader election**  
- только один scheduler активен  
- остальные — standby  

---

# **3.4. kube-controller-manager**

kube-controller-manager запускает десятки встроенных контроллеров:

- node controller  
- endpoints controller  
- replicaset controller  
- deployment controller  
- job controller  
- cronjob controller  
- namespace controller  
- serviceaccount controller  
- pv/pvc controllers  
- garbage collector  
- ttl controller  
- daemonset controller  
- statefulset controller  
- resourcequota controller  

### Особенности:

- запущен на всех CP‑нодах  
- работает в режиме **leader election**  
- только один активен  
- остальные — standby  

---

# **3.5. haproxy**

haproxy — балансировщик трафика к kube-apiserver.

### Функции:

- принимает трафик с VIP  
- распределяет между cp‑01, cp‑02, cp‑03  
- health‑checks API  
- обеспечивает HA  

### Особенности:

- работает на всех CP‑нодах  
- конфигурация одинакова  
- слушает на `6443`  

---

# **3.6. keepalived**

keepalived обеспечивает VIP через VRRP.

### Функции:

- один MASTER  
- два BACKUP  
- failover при падении ноды  
- VIP всегда доступен  

### Особенности:

- MASTER = cp‑01  
- BACKUP = cp‑02, cp‑03  
- VRRP priority:
  - cp‑01: 150  
  - cp‑02: 100  
  - cp‑03: 90  

---

# **3.7. kubelet (на CP‑нодах)**

Да, kubelet работает и на control plane.

### Функции:

- следит за состоянием ноды (NodeStatus)  
- управляет Pod‑sandbox’ами  
- взаимодействует с kube-apiserver
- обеспечивает работу CNI (Calico)
- обеспечивает работу kube-proxy
- обслуживает системные DaemonSet’ы (Calico, kube-proxy, node-exporter и т.п.)  

### Особенности:

- kubelet на CP‑нодах НЕ запускает workload Pod’ы  
- но он нужен для:
  - мониторинга ноды
  - обновления статуса ноды  
  - корректной работы сетевой плоскости  
  - взаимодействия с apiserver
  - запуска системных DaemonSet’ов
  - работы kube-proxy
  - работы Calico  

### Почему это важно

Потому что:

- kube-proxy — DaemonSet → запускается на всех нодах  
- Calico — DaemonSet → запускается на всех нодах  
- node-exporter — DaemonSet → запускается на всех нодах  
- kubelet обязан обслуживать эти Pod’ы  
- kubelet обязан обновлять NodeStatus  
- kubelet обязан выполнять CNI операции  
- kubelet обязан взаимодействовать с API

Control Plane ноды **не запускают workload**,  
но они **полноценные участники сетевой плоскости**.
---

# **3.8. containerd (на CP‑нодах)**

containerd — контейнерный runtime.

### Особенности:

- используется kubelet  
- запускает контейнеры control plane  
- полностью заменяет Docker  
- хранит данные в `/var/lib/containerd` (отдельный LV)  

---

# **3.9. Почему kubelet и containerd не были в списке компонентов Control Plane**

Потому что тот список описывал **только компоненты control plane**, а kubelet и containerd — это:

- **не control plane компоненты**
- **они не участвуют в принятии решений**
- **они не управляют кластером**
- **они не хранят состояние**
- **они не обеспечивают API**

Хотя kubelet и containerd присутствуют на control plane нодах, **но они не являются частью control plane как логической сущности**.

---

## kubelet и containerd — это компоненты узла (node components)

Они есть:

- на worker‑нодах  
- на control plane нодах  

Но они относятся к **node plane**, а не к **control plane**.

---

## Почему kubelet и containerd есть на control plane?

Потому что:

- kubelet нужен для мониторинга ноды  
- kubelet нужен для запуска Pod’ов (если они будут)  
- kubelet нужен для взаимодействия с API  
- containerd нужен kubelet’у  

Но:

- **kubelet НЕ является частью control plane**  
- **containerd НЕ является частью control plane**  

---

  
# **3.10. kube-proxy и Calico на Control Plane**

На CP‑нодах работают **kube-proxy в режиме IPVS** и **Calico (CNI + BGP)**.  
Control Plane ноды не запускают пользовательский workload, но **полноценно участвуют в сетевой плоскости**.

### **kube-proxy на CP‑нодах**

**Функции:**

- реализует Service‑абстракцию (ClusterIP, NodePort, LoadBalancer);  
- выполняет DNAT Service → Pod (через IPVS);  
- синхронизирует Endpoints с kube-apiserver;  
- работает одинаково на CP и worker‑нодах.

**Особенности:**

- не занимается маршрутизацией Pod‑IP между нодами;  
- не заменяет CNI;  
- нужен даже на CP‑нодах, потому что:
  - системные Pod’ы (DaemonSet’ы) могут жить на CP;  
  - Service должны работать одинаково на всех нодах.

---

### **Calico (CNI + BGP) на CP‑нодах**

Calico — это:

- CNI‑плагин (создаёт veth, namespace, IPAM);  
- BGP‑агент (анонсирует PodCIDR ноды).

**Функции на CP‑нодах:**

- обслуживает Pod’ы DaemonSet’ов (Calico сам, kube-proxy, node-exporter и т.п.);  
- создаёт Pod‑интерфейсы и network namespace для этих Pod’ов;  
- анонсирует PodCIDR CP‑ноды через BGP;  
- обеспечивает маршрутизацию Pod‑IP, если на CP появятся Pod’ы (системные или будущие).

**Важно:**

- если control plane компоненты (kube-apiserver, etcd, controller-manager, scheduler) запущены как **systemd‑сервисы**, kubelet **НЕ** создаёт для них Pod’ы, pause‑контейнеры и network namespace;  
- kubelet и Calico обслуживают **только те Pod’ы, которые пришли через kube-apiserver** (DaemonSet, StaticPod, Deployment и т.д.);  
- бинарники control plane, запущенные через systemd, живут в network namespace хоста и не зависят от CNI.

---

### **Почему Calico ставится на Control Plane ноды**

Потому что:

- kubelet работает на CP‑нодах;  
- DaemonSet’ы (Calico, kube-proxy, node-exporter и т.п.) должны запускаться на всех нодах;  
- PodCIDR есть у каждой ноды, включая CP;  
- BGP‑топология должна видеть все ноды как участники Pod‑сети;  
- сетевой стек должен быть единым для CP и worker‑нод.

---

### **Почему не eBPF**

- eBPF‑режим Calico отключает kube-proxy;  
- eBPF — другой dataplane (другая архитектура);  
- eBPF обычно не комбинируют с классическим BGP‑подходом, как у нас.

Мы сознательно используем:

> **классический Calico BGP + kube-proxy IPVS**  
> на **всех нодах, включая Control Plane**.

---

### **Итог**

- **kube-proxy IPVS работает на всех нодах.**  
- **Calico (CNI + BGP) работает на всех нодах.**  
- **Control Plane ноды — полноценные участники сетевой плоскости.**  
- kubelet/Calico не трогают systemd‑сервисы control plane, но обслуживают все Pod’ы, которые реально приходят в кластер.
---

# **3.11. systemd‑юниты (НЕ static pods)**

Control Plane запускается как systemd‑сервисы, а не как static pods.

### Почему это важно:

- systemd управляет процессами напрямую  
- нет зависимости “kubelet → apiserver → kubelet”  
- проще дебажить  
- проще рестартовать  
- проще логировать  
- ближе к реальному production  

### Какие сервисы запускаются через systemd:

- kube-apiserver  
- kube-scheduler  
- kube-controller-manager  
- etcd  
- haproxy  
- keepalived  

---

# **3.12. Итоговая схема Control Plane**

```
                    VIP 192.168.88.10
                           │
                           ▼
                   ┌───────────────┐
                   │    haproxy    │
                   └───────────────┘
                     /     |     \
                    /      |      \
                   ▼       ▼       ▼
            ┌────────┐ ┌────────┐ ┌────────┐
            │cp‑01   │ │cp‑02   │ │cp‑03   │
            │API     │ │API     │ │API     │
            │etcd‑1  │ │etcd‑2  │ │etcd‑3  │
            │sched   │ │sched   │ │sched   │
            │ctrl‑mgr│ │ctrl‑mgr│ │ctrl‑mgr│
            └────────┘ └────────┘ └────────┘
```

---

# **3.13. Итоговое разделение ролей**

### **Компоненты Control Plane =**
- kube-apiserver  
- etcd  
- kube-scheduler  
- kube-controller-manager  
- haproxy  
- keepalived
- kubelet  
- containerd
- kube-proxy
- Calico (BGP)  

### **Компоненты Worker Node =**
- kubelet  
- containerd  
- kube-proxy  
- Calico (BGP)  

---

# **Следующий файл:**

📘 `docs/00-overview/04-workers.md`  
**4. Worker Nodes**

