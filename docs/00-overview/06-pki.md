
Оглавление

- [**6. PKI архитектура**](#6-pki-архитектура)
- [**6.1. Почему PKI — фундамент Kubernetes**](#61-почему-pki--фундамент-kubernetes)
- [**6.2. Общая структура PKI**](#62-общая-структура-pki)
- [**6.3. Root CA**](#63-root-ca)
    - [Файлы:](#файлы)
- [**6.4. Kubernetes CA**](#64-kubernetes-ca)
    - [Файлы:](#файлы-1)
- [**6.4.1. SAN для kube-apiserver**](#641-san-для-kube-apiserver)
    - [**IP‑адреса:**](#ipадреса)
    - [**DNS‑имена VIP:**](#dnsимена-vip)
    - [**DNS‑имена control plane нод:**](#dnsимена-control-plane-нод)
    - [**Стандартные Kubernetes DNS:**](#стандартные-kubernetes-dns)
- [**6.5. Почему SAN должен включать IP и DNS**](#65-почему-san-должен-включать-ip-и-dns)
    - [**1. kubelet подключается по IP**](#1-kubelet-подключается-по-ip)
    - [**2. etcd peer-to-peer работает по IP**](#2-etcd-peer-to-peer-работает-по-ip)
    - [**3. haproxy health checks используют IP и VIP**](#3-haproxy-health-checks-используют-ip-и-vip)
    - [**4. kubectl использует DNS**](#4-kubectl-использует-dns)
    - [**5. Calico, kube-proxy и другие агенты используют VIP/IP**](#5-calico-kube-proxy-и-другие-агенты-используют-vipip)
- [**6.6. etcd CA**](#66-etcd-ca)
    - [Файлы:](#файлы-2)
    - [Сертификаты etcd:](#сертификаты-etcd)
    - [SAN для etcd-peer:](#san-для-etcd-peer)
- [**6.7. Front-proxy CA**](#67-front-proxy-ca)
    - [Файлы:](#файлы-3)
    - [Сертификаты:](#сертификаты)
- [**6.8. Service Account keys**](#68-service-account-keys)
- [**6.9. Структура каталогов PKI**](#69-структура-каталогов-pki)
  - [**На admin‑wks (машина, где генерируем PKI):**](#на-adminwks-машина-где-генерируем-pki)
  - [**На control plane нодах:**](#на-control-plane-нодах)
  - [**На worker‑нодах:**](#на-workerнодах)
- [**6.10. kubeconfig файлы**](#610-kubeconfig-файлы)
- [**6.11. Полная схема PKI**](#611-полная-схема-pki)
- [**6.12. Итог**](#612-итог)


---

Текущий файл:

📘 `docs/00-overview/06-pki.md`

# **6. PKI архитектура**

Глава 6 — одна из самых важных во всём `00-overview`, потому что **PKI — это фундамент безопасности Kubernetes**, и без правильной PKI кластер либо не поднимется, либо будет работать нестабильно.

Каждый компонент Kubernetes общается по TLS, и каждый компонент должен:

- доверять корневому сертификату (CA);  
- иметь свой собственный сертификат;  
- иметь корректные SAN (IP + DNS);  
- иметь приватный ключ;  
- иметь правильные права доступа.

В Kubernetes The Rough Way мы создаём **полностью собственную PKI**, без kubeadm, вручную, с полным пониманием каждого сертификата.  

Модель PKI в нашем проекте это **осознанное архитектурное решение проекта**, а не обязательное требование Kubernetes.  
Её можно реализовать иначе, но мы выбираем **продакшен‑ориентированную иерархическую модель**.

---

# **6.1. Почему PKI — фундамент Kubernetes**

Kubernetes — это распределённая система, где каждый компонент общается по TLS:

- kubelet → kube-apiserver  
- scheduler → kube-apiserver  
- controller-manager → kube-apiserver  
- kube-proxy → kube-apiserver  
- Calico → kube-apiserver  
- kubectl → kube-apiserver  
- kube-apiserver → etcd  

Если хотя бы один сертификат:

- подписан не тем CA,  
- имеет неправильный SAN,  
- имеет неверные права,  
- отсутствует на ноде,

— кластер перестаёт работать.

Поэтому PKI — это **инвариант архитектуры**.

---

# **6.2. Общая структура PKI**

Мы создаём **четыре независимых CA** и один набор ключей:

1. **Root CA**  
2. **Kubernetes CA**  
3. **etcd CA**  
4. **Front-proxy CA**  
5. **Service Account keys** (не CA)

Каждый CA отвечает за свою область ответственности.

Такое разделение:

- упрощает ротацию сертификатов;  
- ограничивает область компрометации;  
- делает архитектуру ближе к продакшен‑кластерам;  
- позволяет независимо управлять Kubernetes, etcd и aggregator API.

---

# **6.3. Root CA**

Root CA — корневой сертификат, которому доверяют:

- Kubernetes CA  
- etcd CA  
- front-proxy CA  

Root CA **не используется напрямую** компонентами Kubernetes.  
Он нужен только для подписи подчинённых CA.

Это **архитектурное решение проекта**:

- Kubernetes может работать и без Root CA (с независимыми CA);  
- мы сознательно вводим Root CA как единую точку доверия;  
- это позволяет менять Kubernetes/etcd/front-proxy CA без смены корня.

### Файлы:

```
root-ca.pem
root-ca-key.pem
```

---

# **6.4. Kubernetes CA**

Kubernetes CA подписывает сертификаты всех компонентов, которые обращаются к kube-apiserver:

- kube-apiserver  
- kube-controller-manager  
- kube-scheduler  
- kubelet (все ноды)  
- kube-proxy  
- admin (kubectl)  
- Calico  
- CoreDNS  
- ingress controller  
- metrics-server  
- любые add-on контроллеры  

### Файлы:

```
kubernetes-ca.pem
kubernetes-ca-key.pem
```

---

# **6.4.1. SAN для kube-apiserver**

SAN — критически важная часть сертификата kube-apiserver.  
Он должен включать **все IP и DNS**, по которым компоненты могут обращаться к API.

### **IP‑адреса:**

- 192.168.88.10 (VIP)  
- 192.168.88.11 (cp‑01)  
- 192.168.88.12 (cp‑02)  
- 192.168.88.13 (cp‑03)  

### **DNS‑имена VIP:**

- k8s-vip  
- api.k8s.local  

### **DNS‑имена control plane нод:**

- cp-01  
- cp-02  
- cp-03  
- cp-01.k8s.local  
- cp-02.k8s.local  
- cp-03.k8s.local  

### **Стандартные Kubernetes DNS:**

- kubernetes  
- kubernetes.default  
- kubernetes.default.svc  
- kubernetes.default.svc.cluster.local  

---

# **6.5. Почему SAN должен включать IP и DNS**

Это критически важно.

### **1. kubelet подключается по IP**  
Если в SAN нет IP kube-apiserver или VIP → TLS error.

### **2. etcd peer-to-peer работает по IP**  
DNS может быть недоступен.  
IP обязателен.

### **3. haproxy health checks используют IP и VIP**  
VIP → IP → apiserver — всё должно быть валидно.

### **4. kubectl использует DNS**  
Если DNS нет в SAN → TLS error.

### **5. Calico, kube-proxy и другие агенты используют VIP/IP**  
Отсутствие IP в SAN ломает их подключение.


> **SAN ВСЕГДА включает и IP, и DNS.**

---

# **6.6. etcd CA**

etcd использует **отдельный CA**, потому что:

- etcd — критически важное хранилище;  
- его сертификаты не должны зависеть от Kubernetes CA;  
- etcd использует peer TLS и client TLS;  
- etcd общается только с apiserver и peer‑нодами.

### Файлы:

```
etcd-ca.pem
etcd-ca-key.pem
```

### Сертификаты etcd:

- `etcd-server` — серверный сертификат etcd;  
- `etcd-peer` — для peer‑коммуникации между нодами;  
- `etcd-client` — для kube-apiserver.

### SAN для etcd-peer:

- IP ноды  
- DNS ноды  

Причины:

- peer‑коммуникация etcd идёт по IP;  
- DNS может быть нестабилен;  
- отсутствие IP в SAN приводит к TLS‑ошибкам.

---

# **6.7. Front-proxy CA**

Используется для:

- aggregator API;  
- metrics-server;  
- kube-apiserver → extension API servers.

### Файлы:

```
front-proxy-ca.pem
front-proxy-ca-key.pem
```

### Сертификаты:

- `front-proxy-client` — используется kube-apiserver.

---

# **6.8. Service Account keys**

ServiceAccount токены подписываются **ключами**, а не сертификатами.

Мы создаём:

```
sa.key
sa.pub
```

Эти ключи используются:

- kube-apiserver — проверка токенов;  
- controller-manager — генерация токенов.

ServiceAccount keys:

- не являются CA;  
- не участвуют в TLS‑рукопожатии;  
- используются только для подписи и проверки JWT‑токенов.

---

# **6.9. Структура каталогов PKI**

## **На admin‑wks (машина, где генерируем PKI):**

```
/root/pki/
    root/
        root-ca.pem
        root-ca-key.pem
    kubernetes/
        kubernetes-ca.pem
        kubernetes-ca-key.pem
    etcd/
        etcd-ca.pem
        etcd-ca-key.pem
    front-proxy/
        front-proxy-ca.pem
        front-proxy-ca-key.pem
    sa/
        sa.key
        sa.pub
```

---

## **На control plane нодах:**

```
/etc/kubernetes/pki/
    kubernetes-ca.crt
    apiserver.crt
    apiserver.key
    apiserver-kubelet-client.crt
    apiserver-kubelet-client.key

    front-proxy-ca.crt
    front-proxy-client.crt
    front-proxy-client.key

    etcd-ca.crt
    etcd-server.crt
    etcd-server.key
    etcd-peer.crt
    etcd-peer.key
    etcd-client.crt
    etcd-client.key

    sa.pub
    sa.key
```

---

## **На worker‑нодах:**

```
/var/lib/kubelet/pki/
    kubelet.crt
    kubelet.key
    kubernetes-ca.crt
```

---

# **6.10. kubeconfig файлы**

Мы создаём kubeconfig для:

- admin (kubectl);  
- kubelet (каждая нода);  
- controller-manager;  
- scheduler;  
- kube-proxy;  
- add-on контроллеров (если требуется).

Каждый kubeconfig содержит:

- CA (обычно `kubernetes-ca.pem`);  
- client certificate;  
- client key;  
- server URL (всегда VIP: `https://192.168.88.10:6443`).

Это гарантирует:

- все компоненты обращаются к API через VIP;  
- смена конкретной control plane ноды не ломает конфигурацию;  
- TLS‑цепочка всегда валидна.

---

# **6.11. Полная схема PKI**

```
                         Root CA
                            │
        ┌───────────────────┼────────────────────┐
        ▼                   ▼                    ▼
 Kubernetes CA         etcd CA           Front-proxy CA
        │                   │                    │
        │                   │                    │
        ▼                   ▼                    ▼
kube-apiserver        etcd-server        front-proxy-client
kubelet               etcd-peer
scheduler             etcd-client
controller-manager
kube-proxy
admin
Calico
CoreDNS
Ingress Controller

ServiceAccount keys — отдельная ветка:
    sa.key / sa.pub
```

---

# **6.12. Итог**

В Kubernetes The Rough Way мы создаём:

- 1 Root CA  
- 3 подчинённых CA  
- десятки сертификатов  
- kubeconfig для каждого компонента  
- SAN, включающий ВСЕ IP и DNS  
- строгую структуру каталогов  
- полную прозрачность PKI  

Это обеспечивает:

- безопасность  
- отказоустойчивость  
- предсказуемость  
- совместимость с продакшен‑кластерами  
- контроль над каждым компонентом  

Следующий файл:

📘 `docs/00-overview/07-networking.md`  
**7. Сетевая архитектура Kubernetes**