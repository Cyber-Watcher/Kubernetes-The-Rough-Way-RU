
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
- [**3.8. containerd (на CP‑нодах)**](#38-containerd-на-cpнодах)
    - [Особенности:](#особенности-5)
- [**3.9. Итоговая схема Control Plane**](#39-итоговая-схема-control-plane)
- [**Пояснения к этому документу**](#пояснения-к-этому-документу)
  - [1. Что значит “systemd‑юнитов (НЕ static pods)”](#1-что-значит-systemdюнитов-не-static-pods)
    - [**Способ 1 — static pods (как делает kubeadm)**](#способ-1--static-pods-как-делает-kubeadm)
    - [**Способ 2 — systemd‑юниты (как в реальных bare‑metal кластерах)**](#способ-2--systemdюниты-как-в-реальных-baremetal-кластерах)
      - [Почему это важно?](#почему-это-важно)
    - [**Итог:**](#итог)
  - [2. Почему kubelet и containerd не были в списке компонентов Control Plane](#2-почему-kubelet-и-containerd-не-были-в-списке-компонентов-control-plane)
    - [kubelet и containerd — это компоненты узла (node components)](#kubelet-и-containerd--это-компоненты-узла-node-components)
    - [Почему kubelet и containerd есть на control plane?](#почему-kubelet-и-containerd-есть-на-control-plane)
  - [Итоговое разъяснение (коротко и точно)](#итоговое-разъяснение-коротко-и-точно)
    - [**Control Plane =**](#control-plane-)
    - [**Node Components =**](#node-components-)
    - [**Почему systemd, а не static pods?**](#почему-systemd-а-не-static-pods)


---

Текущий файл:

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
- использует собственную PKI  
- слушает на `6443/tcp`  
- SAN включает:
  - IP всех CP‑нод  
  - VIP  
  - DNS имена  
  - `kubernetes.default.svc.cluster.local`  

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
- TLS с отдельным etcd CA  
- данные хранятся в `/var/lib/etcd` (отдельный LV)  

### Почему stacked etcd?

- проще  
- надёжнее  
- меньше moving parts  
- соответствует рекомендациям Kubernetes SIG Cluster Lifecycle  

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

- следит за локальными Pod’ами  
- управляет контейнерами  
- общается с apiserver  

### Особенности:

- kubelet на CP‑нодах НЕ запускает workload Pod’ы  
- но он нужен для:
  - мониторинга ноды  
  - запуска systemd‑юнитов  
  - взаимодействия с apiserver  

---

# **3.8. containerd (на CP‑нодах)**

containerd — контейнерный runtime.

### Особенности:

- используется kubelet  
- запускает контейнеры control plane  
- полностью заменяет Docker  

---

# **3.9. Итоговая схема Control Plane**

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

# **Пояснения к этому документу**

На мой взгляд пояснения требуют две вещи:

1) **Что значит “systemd‑юнитов (НЕ static pods)”?** 
2) **Почему в списке компонентов Control Plane не было kubelet и containerd?**

Разбираем по порядку.

## 1. Что значит “systemd‑юнитов (НЕ static pods)”

В Kubernetes есть **два способа запускать control plane компоненты**:

### **Способ 1 — static pods (как делает kubeadm)**

- kubelet читает `/etc/kubernetes/manifests/*.yaml`
- запускает apiserver, scheduler, controller-manager как Pod’ы
- контейнеры control plane управляются kubelet
- kubelet должен быть запущен первым
- kubelet должен быть правильно сконфигурирован
- kubelet должен доверять сертификатам
- kubelet должен уметь подключиться к API

Это **сложная цепочка зависимостей**, и она НЕ подходит для целей Kubernetes The Rough Way.

---

### **Способ 2 — systemd‑юниты (как в реальных bare‑metal кластерах)**

Мы запускаем:

- `/usr/local/bin/kube-apiserver`
- `/usr/local/bin/kube-scheduler`
- `/usr/local/bin/kube-controller-manager`
- `/usr/local/bin/etcd`

через **systemd**, например:

```
/etc/systemd/system/kube-apiserver.service
/etc/systemd/system/kube-scheduler.service
/etc/systemd/system/kube-controller-manager.service
/etc/systemd/system/etcd.service
```

#### Почему это важно?

Потому что:

- systemd управляет процессами **напрямую**, без kubelet  
- control plane запускается **раньше**, чем kubelet  
- нет зависимости “kubelet → apiserver → kubelet”  
- проще дебажить  
- проще контролировать  
- проще рестартовать  
- проще логировать  
- проще управлять ресурсами  
- это **ближе к реальному продакшену**, чем kubeadm‑подход  

---

### **Итог:**

👉 **Мы запускаем control plane как systemd‑сервисы, а не как Pod’ы.**  
👉 **Мы НЕ используем kubeadm‑подход.**  
👉 **Control plane НЕ зависит от kubelet.**

Это абсолютно правильный и продакшен‑ориентированный подход.

## 2. Почему kubelet и containerd не были в списке компонентов Control Plane

Потому что тот список описывал **только компоненты control plane**, а kubelet и containerd — это:

- **не control plane компоненты**
- **они не участвуют в принятии решений**
- **они не управляют кластером**
- **они не хранят состояние**
- **они не обеспечивают API**

Хотя kubelet и containerd присутствуют на control plane нодах, **но они не являются частью control plane как логической сущности**.

---

### kubelet и containerd — это компоненты узла (node components)

Они есть:

- на worker‑нодах  
- на control plane нодах  

Но они относятся к **node plane**, а не к **control plane**.

---

### Почему kubelet и containerd есть на control plane?

Потому что:

- kubelet нужен для мониторинга ноды  
- kubelet нужен для запуска Pod’ов (если они будут)  
- kubelet нужен для взаимодействия с API  
- containerd нужен kubelet’у  

Но:

- kubelet НЕ является частью control plane  
- containerd НЕ является частью control plane  


## Итоговое разъяснение (коротко и точно)

### **Control Plane =**
- kube-apiserver  
- etcd  
- kube-scheduler  
- kube-controller-manager  
- haproxy  
- keepalived  

### **Node Components =**
- kubelet  
- containerd  
- kube-proxy  
- CNI (Calico)  

### **Почему systemd, а не static pods?**
Потому что мы строим кластер вручную, без kubeadm, и нам нужен полный контроль над процессами.

---

Следующий файл:

📘 `docs/00-overview/04-controllers.md`  
**4. Контроллеры Kubernetes**
