
Оглавление


---

Текущий файл:

📘 `docs/00-overview/06-pki.md`

# **6. PKI архитектура**

Глава 6 — одна из самых важных во всём 00‑overview, потому что **PKI — это фундамент безопасности Kubernetes**, и без правильной PKI кластер либо не поднимется, либо будет работать нестабильно.

Каждый компонент Kubernetes общается по TLS, и каждый компонент должен:

- доверять корневому сертификату (CA)  
- иметь свой собственный сертификат  
- иметь корректные SAN (IP + DNS)  
- иметь приватный ключ  
- иметь правильные права доступа  

В Kubernetes The Rough Way мы создаём **полностью собственную PKI**, без kubeadm, без автоматизации, вручную, с полным пониманием каждого сертификата.

---

# **6.1. Почему PKI — фундамент безопасности Kubernetes**

Kubernetes — это распределённая система, где:

- kubelet → apiserver  
- scheduler → apiserver  
- controller-manager → apiserver  
- apiserver → etcd  
- kubectl → apiserver  
- kube-proxy → apiserver  
- Calico → apiserver  

Все эти взаимодействия происходят **по TLS**.

Если PKI неправильная — кластер не работает.

---

# **6.2. Общая структура PKI**

Мы создаём **четыре независимых CA**:

1. **Root CA**  
2. **Kubernetes CA**  
3. **etcd CA**  
4. **Front-proxy CA**

И один набор ключей:

5. **Service Account keys**

Каждый CA отвечает за свою область.

---

# **6.3. Root CA**

Root CA — это корневой сертификат, которому доверяют:

- Kubernetes CA  
- etcd CA  
- front-proxy CA  

Root CA **не используется напрямую** компонентами Kubernetes.  
Он нужен только для подписи других CA.

### Назначение:

- единая точка доверия  
- возможность ротации подчинённых CA  
- безопасность иерархии  

### Файлы:

```
root-ca.pem
root-ca-key.pem
```

---

# **6.4. Kubernetes CA**

Этот CA подписывает сертификаты:

- kube-apiserver  
- kube-controller-manager  
- kube-scheduler  
- kubelet (все ноды)  
- admin (kubectl)  
- kube-proxy  
- Calico  
- CoreDNS  
- ingress controller  

### Файлы:

```
kubernetes-ca.pem
kubernetes-ca-key.pem
```

### SAN для kube-apiserver:

Обязательно включаем:

- IP всех control plane нод  
- VIP  
- DNS имена всех нод  
- `kubernetes`  
- `kubernetes.default`  
- `kubernetes.default.svc`  
- `kubernetes.default.svc.cluster.local`  

---

# **6.5. etcd CA**

etcd использует **отдельный CA**, потому что:

- etcd — критически важный компонент  
- его сертификаты не должны зависеть от Kubernetes CA  
- etcd общается только с apiserver и peer‑нодами  
- etcd использует **peer TLS** и **client TLS**  

### Файлы:

```
etcd-ca.pem
etcd-ca-key.pem
```

### Сертификаты etcd:

- etcd-server  
- etcd-peer  
- etcd-client (для apiserver)

### SAN для etcd-peer:

- IP ноды  
- DNS ноды  
- обязательно IP (DNS недостаточно)

---

# **6.6. Front-proxy CA**

Используется для:

- aggregator API  
- metrics-server  
- kube-apiserver → extension API servers  

### Файлы:

```
front-proxy-ca.pem
front-proxy-ca-key.pem
```

### Сертификаты:

- front-proxy-client (для apiserver)

---

# **6.7. Service Account keys**

ServiceAccount токены подписываются **ключами**, а не сертификатами.

Мы создаём:

```
sa.key
sa.pub
```

Эти ключи используются:

- kube-apiserver  
- controller-manager  

---

# **6.8. Структура каталогов PKI**

На admin‑wks:

```
/root/pki/
    root/
    kubernetes/
    etcd/
    front-proxy/
    sa/
```

На control plane нодах:

```
/etc/kubernetes/pki/
    apiserver.crt
    apiserver.key
    apiserver-kubelet-client.crt
    apiserver-kubelet-client.key
    front-proxy-client.crt
    front-proxy-client.key
    ca.crt
    front-proxy-ca.crt
    sa.pub
    sa.key
```

На worker‑нодах:

```
/var/lib/kubelet/pki/
    kubelet.crt
    kubelet.key
    ca.crt
```

---

# **6.9. Почему SAN должен включать IP и DNS**

Это критически важно.

### Причины:

### **1. kubelet подключается по IP**
Если в SAN нет IP → TLS error.

### **2. etcd peer-to-peer работает только по IP**
DNS не используется.

### **3. haproxy health checks используют IP**
VIP → IP → apiserver.

### **4. kubectl может использовать DNS**
Но kubelet — нет.

### **5. Calico BGP использует IP**
DNS не участвует.

---

# **6.10. kubeconfig файлы**

Мы создаём kubeconfig для:

- admin  
- kubelet (каждая нода)  
- controller-manager  
- scheduler  
- kube-proxy  

Каждый kubeconfig содержит:

- CA  
- client certificate  
- client key  
- server URL (VIP)  

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
```

ServiceAccount keys — отдельная ветка.

---

# **6.12. Итог**

PKI — это фундамент Kubernetes.  
Мы создаём:

- 4 CA  
- десятки сертификатов  
- kubeconfig для каждого компонента  
- SAN с IP и DNS  
- строгую структуру каталогов  

Это обеспечивает:

- безопасность  
- отказоустойчивость  
- предсказуемость  
- совместимость с продакшен‑кластерами  

---

Следующий файл:

📘 `docs/00-overview/07-networking.md`  
**7. Сетевая архитектура**
