Оглавление

- [**2. Компоненты Kubernetes (Big Picture)**](#2-компоненты-kubernetes-big-picture)
- [**2.1. Два плана Kubernetes**](#21-два-плана-kubernetes)
- [**2.2. Компоненты Control Plane**](#22-компоненты-control-plane)
    - [**2.2.1. kube-apiserver**](#221-kube-apiserver)
    - [**2.2.2. etcd**](#222-etcd)
    - [**2.2.3. kube-scheduler**](#223-kube-scheduler)
    - [**2.2.4. kube-controller-manager**](#224-kube-controller-manager)
    - [**2.2.5. haproxy**](#225-haproxy)
    - [**2.2.6. keepalived**](#226-keepalived)
- [**2.3. Компоненты Worker Nodes**](#23-компоненты-worker-nodes)
    - [**2.3.1. kubelet**](#231-kubelet)
    - [**2.3.2. containerd**](#232-containerd)
    - [**2.3.3. kube-proxy**](#233-kube-proxy)
    - [**2.3.4. Calico (CNI)**](#234-calico-cni)
- [**2.4. Компоненты, работающие как Pod’ы (Add-ons)**](#24-компоненты-работающие-как-podы-add-ons)
    - [**2.4.1. CoreDNS**](#241-coredns)
    - [**2.4.2. NGINX Ingress Controller**](#242-nginx-ingress-controller)
    - [**2.4.3. Metrics-server**](#243-metrics-server)
    - [**2.4.4. Calico kube-controllers**](#244-calico-kube-controllers)
    - [**2.4.5. Storage provisioner (опционально)**](#245-storage-provisioner-опционально)
- [**2.5. Как компоненты взаимодействуют**](#25-как-компоненты-взаимодействуют)
- [**2.6. Почему Kubernetes — это система с чётким разделением обязанностей**](#26-почему-kubernetes--это-система-с-чётким-разделением-обязанностей)
- [**2.7. Компоненты, которые мы НЕ используем**](#27-компоненты-которые-мы-не-используем)
- [**2.8. Компоненты, которые мы будем устанавливать вручную**](#28-компоненты-которые-мы-будем-устанавливать-вручную)
    - [Бинарники:](#бинарники)
    - [Манифестами:](#манифестами)


---

Текущий файл:
📘 `docs/00-overview/02-components.md`  

# **2. Компоненты Kubernetes (Big Picture)**

Kubernetes — это распределённая система, состоящая из множества компонентов, каждый из которых выполняет строго определённую роль.  
В этой главе мы рассматриваем **все основные компоненты**, которые будут установлены и настроены в рамках Kubernetes The Rough Way.

---

# **2.1. Два плана Kubernetes**

Kubernetes разделён на два логических уровня:

- **Control Plane** — мозг кластера  
- **Workload Plane (Worker Nodes)** — место, где запускаются Pod’ы

---

# **2.2. Компоненты Control Plane**

Control Plane управляет кластером, принимает решения, хранит состояние и обеспечивает API.

В нашем кластере control plane состоит из:

### **2.2.1. kube-apiserver**
- центральная точка входа  
- REST API  
- единственный компонент, который общается с etcd  
- все остальные компоненты общаются только с ним  

### **2.2.2. etcd**
- распределённое хранилище состояния  
- хранит все объекты Kubernetes  
- работает в режиме Raft (3 реплики)  
- критически важен для HA  

### **2.2.3. kube-scheduler**
- выбирает ноду для каждого Pod  
- учитывает ресурсы, taints, affinity, topology  

### **2.2.4. kube-controller-manager**
- запускает десятки встроенных контроллеров  
- поддерживает желаемое состояние  
- работает в режиме leader election  

### **2.2.5. haproxy**
- балансирует трафик к apiserver  
- работает на всех CP‑нодах  

### **2.2.6. keepalived**
- обеспечивает VIP  
- VRRP: один MASTER, два BACKUP  

---

# **2.3. Компоненты Worker Nodes**

Worker‑ноды выполняют Pod’ы и обеспечивают сетевую связность.

### **2.3.1. kubelet**
- агент на каждой ноде  
- запускает Pod’ы  
- следит за их состоянием  
- общается с apiserver  

### **2.3.2. containerd**
- контейнерный runtime  
- запускает контейнеры  
- взаимодействует с kubelet через CRI  

### **2.3.3. kube-proxy**
- реализует Service networking  
- iptables/ipvs  
- перенаправляет трафик к Pod’ам  

### **2.3.4. Calico (CNI)**
- Pod‑сеть  
- маршрутизация (BGP mode)  
- NetworkPolicy  
- распределённый routing без overlay  

---

# **2.4. Компоненты, работающие как Pod’ы (Add-ons)**

Эти компоненты устанавливаются манифестами и работают внутри кластера.

### **2.4.1. CoreDNS**
- DNS внутри кластера  
- обслуживает `*.svc.cluster.local`  

### **2.4.2. NGINX Ingress Controller**
- HTTP/HTTPS вход в кластер  
- маршрутизация внешнего трафика к сервисам  

### **2.4.3. Metrics-server**
- предоставляет метрики CPU/RAM  
- нужен для HPA и `kubectl top`  

### **2.4.4. Calico kube-controllers**
- контроллеры Calico  
- синхронизация IPAM, маршрутов, BGP  

### **2.4.5. Storage provisioner (опционально)**
- например, local-path-provisioner  
- создаёт PVC/PV автоматически  

---

# **2.5. Как компоненты взаимодействуют**

Ниже — краткая схема взаимодействия (детальные диаграммы будут в главе 7):

```
kubectl → kube-apiserver → etcd
kubelet → kube-apiserver
scheduler → kube-apiserver
controller-manager → kube-apiserver
kube-proxy → kube-apiserver
Calico → BGP routing between nodes
CoreDNS → kube-apiserver (watch), отвечает на DNS
Ingress → Service → Pod
```

---

# **2.6. Почему Kubernetes — это система с чётким разделением обязанностей**

Каждый компонент выполняет строго определённую роль:

- apiserver — API  
- etcd — состояние  
- scheduler — выбор ноды  
- controller-manager — поддержание состояния  
- kubelet — запуск Pod’ов  
- kube-proxy — сетевой трафик  
- CNI — Pod‑сеть  
- CoreDNS — DNS  
- ingress — HTTP вход  

---

# **2.7. Компоненты, которые мы НЕ используем**

Мы **не используем**:

- kubeadm  
- cloud-controller-manager  
- overlay‑сети (VXLAN)  
- flannel  
- docker (dockershim deprecated)  
- kube-dashboard  
- metallb (у нас keepalived + haproxy)  

---

# **2.8. Компоненты, которые мы будем устанавливать вручную**

### Бинарники:
- kube-apiserver  
- kube-controller-manager  
- kube-scheduler  
- etcd  
- kubelet  
- kube-proxy  
- containerd  

### Манифестами:
- Calico  
- CoreDNS  
- NGINX ingress  
- metrics-server  
- storage provisioner  

---

Следующий файл:

📘 `docs/00-overview/03-control-plane.md`  
**3. Control Plane — детальный обзор**
