
Оглавление


---

Текущий файл:

📘 `docs/00-overview/09-tech-stack.md`  

# **9. Стек технологий Kubernetes The Rough Way**

Стек технологий определяет фундаментальные решения, на которых строится весь кластер.  
Он должен быть:

- предсказуемым,  
- прозрачным,  
- минималистичным,  
- полностью контролируемым,  
- независимым от kubeadm и Helm.

---

# **9.1. Kubernetes Core Components**

Мы устанавливаем бинарники Kubernetes вручную и размещаем их в:

```
/usr/local/bin
```

Это соответствует FHS и исключает конфликты с пакетными менеджерами.

### **Используемые компоненты:**

- `kube-apiserver`  
- `kube-controller-manager`  
- `kube-scheduler`  
- `kubelet`  
- `kube-proxy`  
- `kubectl`  
- `etcd` и `etcdctl`

### **Причины выбора:**

- полный контроль над версиями;  
- отсутствие зависимости от kubeadm;  
- предсказуемость поведения;  
- возможность обновлять компоненты независимо;  
- соответствие self‑hosted подходу;  
- согласованность с PKI (глава 06).

---

# **9.2. Container Runtime**

Мы используем **containerd** как единственный CRI.

### **Причины:**

- официально поддерживается Kubernetes;  
- минималистичный, стабильный, без лишних слоёв;  
- хранит данные в `/var/lib/containerd` (идеально для LVM);  
- простая интеграция с kubelet через CRI API;  
- предсказуемое поведение под нагрузкой.

---

# **9.3. CNI (Container Network Interface)**

Мы используем **Calico** в режиме:

- **Pure BGP (No-Encap)** — без overlay;  
- маршрутизация PodCIDR напрямую между нодами;  
- поддержка NetworkPolicy;  
- нативная интеграция с Pod CIDR.

### **Причины:**

- минимальная задержка;  
- отсутствие encapsulation overhead;  
- предсказуемость маршрутизации;  
- прозрачность dataplane;  
- согласованность с Node CIDR `192.168.88.0/24`.

### **Инвариант:**

Calico BGP без overlay работает идеально, потому что **все ноды находятся в одном L2‑сегменте**.  
Если появятся ноды в других подсетях — потребуется:

- BGP‑пиринг с маршрутизатором, или  
- включение IPIP/VXLAN.

(подробно описано в главе 07).

---

# **9.4. Load Balancing Layer для kube-apiserver**

Для HA API используется связка:

- **HAProxy** — L4/L7 балансировщик  
- **Keepalived (VRRP)** — виртуальный IP

### **VIP:**

```
192.168.88.10
```

### **Схема:**

```
kubelet / kubectl → VIP 192.168.88.10 → HAProxy → cp‑01 / cp‑02 / cp‑03
```

### **Причины выбора HAProxy:**

- высокая производительность;  
- нативная TCP‑балансировка;  
- простая конфигурация;  
- предсказуемость под нагрузкой.

### **Причины выбора Keepalived:**

- VRRP обеспечивает быстрый failover;  
- минимальная задержка переключения;  
- простота и надёжность.

---

# **9.5. Ingress Layer**

Для входящего HTTP/HTTPS‑трафика используется:

- **NGINX Ingress Controller**

### **Причины:**

- зрелый и стабильный проект;  
- гибкая маршрутизация;  
- поддержка TLS termination;  
- интеграция с Service CIDR;  
- предсказуемое поведение.

### **Важно:**

**NGINX не используется для API‑сервера.**  
API‑сервер обслуживается HAProxy + VRRP.

---

# **9.6. PKI Stack**

Вся криптография кластера строится на:

- **OpenSSL**  
- собственном Root CA  
- Kubernetes CA  
- etcd CA  
- front‑proxy CA  
- вручную созданных сертификатах

### **Сертификаты используются для:**

- etcd (peer + client)  
- kube-apiserver  
- kubelet  
- kube-proxy  
- kubectl  
- ingress (опционально)

**Принцип: TLS everywhere**

Полная архитектура PKI описана в главе 06.

---

# **9.7. Storage & Filesystem Stack**

Стек хранения согласован с LVM‑архитектурой (глава 08).

### **Используем:**

- **LVM** — изоляция и гибкость  
- **ext4** — стабильная ФС  
- отдельные LV для:

```
/var/lib/etcd
/var/lib/kubelet
/var/lib/containerd
/var/log
```

### **Причины:**

- etcd требует низкой латентности fsync;  
- containerd может разрастаться непредсказуемо;  
- kubelet создаёт временные тома;  
- journald может забивать диск.

---

# **9.8. OS & System Stack**

Операционная система и системные компоненты:

- Ubuntu LTS  
- systemd  
- journald  
- iptables / nftables  
- conntrack  
- базовые sysctl для сети  
- **swap off как инвариант**

### **Причины:**

- стабильность;  
- предсказуемость;  
- соответствие требованиям Kubernetes;  
- согласованность с сетевой архитектурой (глава 07).

---

# **9.9. Observability Stack**

Минимальный набор для мониторинга:

- metrics-server  
- kube-state-metrics  
- Prometheus  
- Grafana  
- node-exporter  
- (опционально) fluent-bit / loki / EFK

Этот стек будет раскрыт в отдельной главе.

---

# **9.10. Security Stack**

Минимальный набор безопасности:

- RBAC  
- TLS для всех компонентов  
- kubelet authentication  
- kubelet authorization  
- ограничение sysctl  
- минимальные capabilities  
- NetworkPolicy (Calico)

---

# **9.11. Инструменты разработки и CI/CD**

Используемые инструменты:

- kubectl  
- kustomize  
- helm (опционально)  
- GitHub/GitLab CI  
- GitOps‑подход (концептуально)

---

# **9.12. Итоговая сводка стека**

```
Компонент        | Технология           | Причина выбора
Control Plane    | kube-*               | Self-hosted, контроль версий
Runtime          | containerd           | CRI, стабильность
CNI              | Calico (BGP)         | L3, производительность
API LB           | HAProxy + VRRP       | HA для API
Ingress          | NGINX Ingress        | HTTP/HTTPS входной трафик
PKI              | OpenSSL              | Полный контроль
Storage          | LVM + ext4           | Изоляция и гибкость
OS               | Ubuntu LTS           | Стабильность
Security         | RBAC, TLS, NP        | Production-grade
Observability    | Prometheus stack     | CNCF стандарт
```

---

Следующий файл:

📘 `docs/00-overview/10-deployment-plan.md`  
**10. План развёртывания Kubernetes The Rough Way**

