
Оглавление

- [**4. Контроллеры Kubernetes**](#4-контроллеры-kubernetes)
- [**4.1. Что такое контроллеры**](#41-что-такое-контроллеры)
- [**4.2. Где живут контроллеры**](#42-где-живут-контроллеры)
  - [**4.2.1. Встроенные контроллеры (core controllers)**](#421-встроенные-контроллеры-core-controllers)
  - [**4.2.2. Контроллеры, работающие как Pod’ы**](#422-контроллеры-работающие-как-podы)
- [**4.3. kube-controller-manager и leader election**](#43-kube-controller-manager-и-leader-election)
    - [Как это работает:](#как-это-работает)
- [**4.4. Полный список встроенных контроллеров (core controllers)**](#44-полный-список-встроенных-контроллеров-core-controllers)
  - [**4.4.1. Node Controller**](#441-node-controller)
  - [**4.4.2. Endpoints Controller**](#442-endpoints-controller)
  - [**4.4.3. ReplicaSet Controller**](#443-replicaset-controller)
  - [**4.4.4. Deployment Controller**](#444-deployment-controller)
  - [**4.4.5. StatefulSet Controller**](#445-statefulset-controller)
  - [**4.4.6. DaemonSet Controller**](#446-daemonset-controller)
  - [**4.4.7. Job Controller**](#447-job-controller)
  - [**4.4.8. CronJob Controller**](#448-cronjob-controller)
  - [**4.4.9. Namespace Controller**](#449-namespace-controller)
  - [**4.4.10. ServiceAccount Controller**](#4410-serviceaccount-controller)
  - [**4.4.11. PV Controller**](#4411-pv-controller)
  - [**4.4.12. PVC Controller**](#4412-pvc-controller)
  - [**4.4.13. ResourceQuota Controller**](#4413-resourcequota-controller)
  - [**4.4.14. HorizontalPodAutoscaler Controller**](#4414-horizontalpodautoscaler-controller)
  - [**4.4.15. Certificate Controller**](#4415-certificate-controller)
  - [**4.4.16. Token Controller**](#4416-token-controller)
  - [**4.4.17. Garbage Collector**](#4417-garbage-collector)
  - [**4.4.18. TTL Controller**](#4418-ttl-controller)
- [**4.5. Контроллеры, работающие как Pod’ы**](#45-контроллеры-работающие-как-podы)
  - [**4.5.1. kube-proxy (DaemonSet)**](#451-kube-proxy-daemonset)
  - [**4.5.2. Calico Controllers**](#452-calico-controllers)
  - [**4.5.3. CoreDNS**](#453-coredns)
  - [**4.5.4. NGINX Ingress Controller**](#454-nginx-ingress-controller)
  - [**4.5.5. Metrics-server**](#455-metrics-server)
  - [**4.5.6. CSI Controllers**](#456-csi-controllers)
- [**4.6. Как контроллеры взаимодействуют с API**](#46-как-контроллеры-взаимодействуют-с-api)
- [**4.7. Почему контроллеры — сердце Kubernetes**](#47-почему-контроллеры--сердце-kubernetes)


---

Текущий файл:

📘 `docs/00-overview/04-controllers.md` 

# **4. Контроллеры Kubernetes**

Этот документ описывает **все контроллеры Kubernetes**, их назначение, архитектуру, взаимодействие с API, механизм leader election и то, как они работают в нашем кластере Kubernetes The Rough Way.

Контроллеры — это сердце Kubernetes.  
Они обеспечивают self‑healing, reconciliation, автоматизацию и поддержание желаемого состояния.

---

# **4.1. Что такое контроллеры**

Контроллер — это **control loop**, который:

1. читает текущее состояние из kube-apiserver  
2. сравнивает его с желаемым состоянием  
3. предпринимает действия, чтобы привести систему в соответствие  

Это фундаментальная модель Kubernetes.

---

# **4.2. Где живут контроллеры**

В Kubernetes есть **два типа контроллеров**:
- Встроенные контроллеры (core controllers)
- Контроллеры, работающие как Pod’ы

---

## **4.2.1. Встроенные контроллеры (core controllers)**  
Живут **внутри бинарника `kube-controller-manager`**.

Мы НЕ устанавливаем их по отдельности.  
Мы запускаем **один бинарник**, и он запускает десятки контроллеров.

---

## **4.2.2. Контроллеры, работающие как Pod’ы**

Это:

- kube-proxy (DaemonSet)  
- Calico controllers  
- CoreDNS  
- Ingress controller  
- Metrics-server  
- CSI controllers  

Эти контроллеры устанавливаются **манифестами**, а не бинарниками.

---

# **4.3. kube-controller-manager и leader election**

kube-controller-manager работает на всех трёх control plane нодах:

| Компонент | cp‑01 | cp‑02 | cp‑03 | Режим |
|-----------|-------|-------|-------|--------|
| kube-controller-manager | ✔ | ✔ | ✔ | leader election |

### Как это работает:

- все три экземпляра подключаются к API  
- выбирают лидера через Lease API  
- лидер выполняет работу  
- остальные ждут  
- при падении лидера — мгновенное переключение  

Это обеспечивает отказоустойчивость.

---

# **4.4. Полный список встроенных контроллеров (core controllers)**

Ниже — **все контроллеры**, которые запускаются внутри kube-controller-manager.

---

## **4.4.1. Node Controller**
Следит за состоянием нод:

- помечает NotReady  
- удаляет Pod’ы с мёртвых нод  
- обновляет статус ноды  

---

## **4.4.2. Endpoints Controller**
Создаёт Endpoints для Service.

---

## **4.4.3. ReplicaSet Controller**
Поддерживает количество Pod’ов в ReplicaSet.

---

## **4.4.4. Deployment Controller**
Реализует:

- rolling updates  
- rollbacks  
- паузы/возобновления  

---

## **4.4.5. StatefulSet Controller**
Управляет StatefulSet:

- стабильные имена Pod’ов  
- стабильные PVC  
- упорядоченный запуск/остановка  

---

## **4.4.6. DaemonSet Controller**
Гарантирует, что Pod запущен на каждой ноде.

---

## **4.4.7. Job Controller**
Запускает Pod’ы до успешного завершения.

---

## **4.4.8. CronJob Controller**
Запускает Job по расписанию.

---

## **4.4.9. Namespace Controller**
Удаляет объекты при удалении namespace.

---

## **4.4.10. ServiceAccount Controller**
Создаёт токены сервисных аккаунтов.

---

## **4.4.11. PV Controller**
Управляет жизненным циклом PersistentVolume.

---

## **4.4.12. PVC Controller**
Обрабатывает PersistentVolumeClaim.

---

## **4.4.13. ResourceQuota Controller**
Следит за квотами в namespace.

---

## **4.4.14. HorizontalPodAutoscaler Controller**
Работает вместе с metrics-server.

---

## **4.4.15. Certificate Controller**
Подписывает CSR (если включено).

---

## **4.4.16. Token Controller**
Создаёт токены сервисных аккаунтов.

---

## **4.4.17. Garbage Collector**
Удаляет неиспользуемые объекты.

---

## **4.4.18. TTL Controller**
Удаляет завершённые Job.

---

# **4.5. Контроллеры, работающие как Pod’ы**

Эти контроллеры НЕ входят в kube-controller-manager.

---

## **4.5.1. kube-proxy (DaemonSet)**  
Реализует Service networking:

- iptables  
- ipvs  

---

## **4.5.2. Calico Controllers**
Включают:

- IPAM  
- маршрутизацию  
- синхронизацию BGP  
- NetworkPolicy  

---

## **4.5.3. CoreDNS**
DNS внутри кластера.

---

## **4.5.4. NGINX Ingress Controller**
HTTP/HTTPS вход к приложениям в кластере (подах).

---

## **4.5.5. Metrics-server**
Нужен для:

- HPA  
- `kubectl top`  

---

## **4.5.6. CSI Controllers**
Если будем ставить local-path-provisioner или Ceph.

---

# **4.6. Как контроллеры взаимодействуют с API**

Все контроллеры работают по одной схеме:

```
loop:
    read desired state from API
    read actual state from API
    compare
    apply changes
```

Ни один контроллер **не общается напрямую с etcd**.

---

# **4.7. Почему контроллеры — сердце Kubernetes**

Потому что они обеспечивают:

- self‑healing  
- reconciliation  
- автоматизацию  
- масштабирование  
- управление жизненным циклом  
- поддержание желаемого состояния  

Без контроллеров Kubernetes — просто набор процессов.

Следующий файл:

📘 `docs/00-overview/05-workers.md`  
**5. Worker Nodes — детальный обзор**
