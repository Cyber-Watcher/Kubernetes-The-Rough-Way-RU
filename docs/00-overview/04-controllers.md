
Оглавление

- [**4. Контроллеры Kubernetes**](#4-контроллеры-kubernetes)
  - [**4.1. Что такое контроллеры**](#41-что-такое-контроллеры)
  - [**4.2. Где живут контроллеры**](#42-где-живут-контроллеры)
  - [**4.3. Встроенные контроллеры (core controllers)**](#43-встроенные-контроллеры-core-controllers)
  - [**4.4. kube-scheduler как контроллер**](#44-kube-scheduler-как-контроллер)
  - [**4.5. Полный список встроенных контроллеров и их роль**](#45-полный-список-встроенных-контроллеров-и-их-роль)
    - [**4.5.1. Node Controller**](#451-node-controller)
    - [**4.5.2. Endpoints Controller**](#452-endpoints-controller)
    - [**4.5.3. ReplicaSet Controller**](#453-replicaset-controller)
    - [**4.5.4. Deployment Controller**](#454-deployment-controller)
    - [**4.5.5. StatefulSet Controller**](#455-statefulset-controller)
    - [**4.5.6. DaemonSet Controller**](#456-daemonset-controller)
    - [**4.5.7. Job Controller**](#457-job-controller)
    - [**4.5.8. CronJob Controller**](#458-cronjob-controller)
    - [**4.5.9. Namespace Controller**](#459-namespace-controller)
    - [**4.5.10. ServiceAccount Controller**](#4510-serviceaccount-controller)
    - [**4.5.11. PV Controller**](#4511-pv-controller)
    - [**4.5.12. PVC Controller**](#4512-pvc-controller)
    - [**4.5.13. ResourceQuota Controller**](#4513-resourcequota-controller)
    - [**4.5.14. HorizontalPodAutoscaler Controller**](#4514-horizontalpodautoscaler-controller)
    - [**4.5.15. Certificate Controller**](#4515-certificate-controller)
    - [**4.5.16. Token Controller**](#4516-token-controller)
    - [**4.5.17. Garbage Collector**](#4517-garbage-collector)
    - [**4.5.18. TTL Controller**](#4518-ttl-controller)
  - [**4.6. Компоненты, работающие как Pod’ы (add-on)**](#46-компоненты-работающие-как-podы-add-on)
    - [**kube-proxy (DaemonSet)** — *сетевой агент*](#kube-proxy-daemonset--сетевой-агент)
    - [**Calico (DaemonSet)** — *CNI + BGP агент*](#calico-daemonset--cni--bgp-агент)
    - [**CoreDNS (Deployment)** — *DNS‑сервер кластера*](#coredns-deployment--dnsсервер-кластера)
    - [**Ingress Controller (Deployment/DaemonSet)** — *L7‑прокси*](#ingress-controller-deploymentdaemonset--l7прокси)
    - [**metrics-server (Deployment)** — *агент метрик*](#metrics-server-deployment--агент-метрик)
    - [**CSI Controllers (Deployment/StatefulSet)** — *контроллеры хранения, но не часть control plane*](#csi-controllers-deploymentstatefulset--контроллеры-хранения-но-не-часть-control-plane)
  - [**4.7. Операторы и CRD**](#47-операторы-и-crd)
  - [**4.8. Как контроллеры взаимодействуют с API**](#48-как-контроллеры-взаимодействуют-с-api)
  - [**4.9. Почему контроллеры — сердце Kubernetes**](#49-почему-контроллеры--сердце-kubernetes)


---

Текущий файл:

📘 `docs/00-overview/04-controllers.md` 

# **4. Контроллеры Kubernetes**

Этот документ описывает контроллеры Kubernetes, их назначение, архитектуру, взаимодействие с API и то, как они работают в нашем кластере Kubernetes The Rough Way.

Контроллеры — это сердце Kubernetes.  
Они обеспечивают self‑healing, reconciliation, автоматизацию и поддержание желаемого состояния.

---

## **4.1. Что такое контроллеры**

Контроллер — это control loop, который:

- читает текущее состояние из kube-apiserver;  
- сравнивает его с желаемым состоянием (spec);  
- предпринимает действия, чтобы привести систему в соответствие;  
- записывает изменения обратно через kube-apiserver.

Это фундаментальная модель Kubernetes: **контроллеры не меняют spec, они приводят status к spec**.  
Ни один контроллер не общается напрямую с etcd — только через API.

---

## **4.2. Где живут контроллеры**

Контроллеры бывают:

- **встроенные (core/built-in)** — часть `kube-controller-manager` и логики планировщика (`kube-scheduler`);  
- **add-on контроллеры** — CoreDNS, Calico, Ingress‑контроллеры, metrics-server, CSI;  
- **пользовательские (custom)** — операторы, работающие с CRD.

В нашем кластере:

- встроенные контроллеры работают внутри бинарника `kube-controller-manager`;  
- планировщик (`kube-scheduler`) реализует контроллерный паттерн для назначения нод;  
- Calico, NGINX Ingress, CoreDNS, metrics-server и т.п. работают как Pod’ы (Deployment/DaemonSet).

---

## **4.3. Встроенные контроллеры (core controllers)**

Встроенные контроллеры живут внутри бинарника `kube-controller-manager`.  
Мы **не устанавливаем их по отдельности**: запускаем один бинарник, и он поднимает десятки контроллеров.  
Все они используют один и тот же паттерн: наблюдать объекты через API, сравнивать spec/status, вносить изменения через API.

`kube-controller-manager` запущен на всех трёх control plane‑нодах и работает в режиме **leader election**:  
один экземпляр активен, остальные ждут. При падении лидера управление автоматически переходит к другому экземпляру.

---

## **4.4. kube-scheduler как контроллер**

`kube-scheduler` — это отдельный бинарник, но логически он тоже контроллер.

Он:

- наблюдает за Pod’ами без `spec.nodeName`;  
- анализирует ресурсы нод, taints/tolerations, affinity/anti-affinity, topology;  
- выбирает подходящую ноду и записывает её в `spec.nodeName` через kube-apiserver.

Он не запускает Pod’ы — этим занимается kubelet.  
Как и `kube-controller-manager`, scheduler работает на всех CP‑нодах с leader election: активен только один экземпляр.

---

## **4.5. Полный список встроенных контроллеров и их роль**

Ниже — основные контроллеры, работающие внутри `kube-controller-manager`.  
Каждый из них реализует свой reconciliation loop.

### **4.5.1. Node Controller**

Следит за состоянием нод (Ready/NotReady).  
При потере heartbeat помечает ноду NotReady и инициирует эвакуацию Pod’ов.  
Обновляет статус ноды и запускает дальнейшие действия (taint, удаление Pod’ов).

### **4.5.2. Endpoints Controller**

Создаёт и обновляет объекты Endpoints для Service.  
Следит за тем, чтобы список Pod’ов‑backend’ов соответствовал селекторам Service.  
Обеспечивает корректную маршрутизацию трафика к живым Pod’ам.

### **4.5.3. ReplicaSet Controller**

Поддерживает нужное количество Pod’ов в ReplicaSet.  
Если Pod’ов меньше, чем указано в spec — создаёт новые; если больше — удаляет лишние.  
Обеспечивает базовый механизм поддержания реплик.

### **4.5.4. Deployment Controller**

Управляет Deployment поверх ReplicaSet.  
Реализует rolling update, rollback, паузу/возобновление развёртывания.  
Следит за историей ReplicaSet’ов и корректным переходом между версиями.

### **4.5.5. StatefulSet Controller**

Управляет StatefulSet — состоянием и идентичностью Pod’ов.  
Обеспечивает стабильные имена Pod’ов и привязку к PVC.  
Гарантирует упорядоченный запуск и остановку Pod’ов.

### **4.5.6. DaemonSet Controller**

Гарантирует, что Pod DaemonSet’а запущен на каждой подходящей ноде.  
Реагирует на добавление/удаление нод, создавая или удаляя Pod’ы.  
Используется для системных агентов (kube-proxy, Calico, node-exporter и т.п.).

### **4.5.7. Job Controller**

Управляет Job — одноразовыми задачами.  
Следит за тем, чтобы нужное количество Pod’ов успешно завершилось.  
Перезапускает Pod’ы при неуспехе, если это предусмотрено spec.

### **4.5.8. CronJob Controller**

Запускает Job по расписанию.  
Следит за соблюдением cron‑графика и создаёт Job в нужные моменты времени.  
Управляет историей выполненных Job и их очисткой.

### **4.5.9. Namespace Controller**

Обрабатывает удаление namespace.  
При удалении namespace последовательно удаляет все объекты внутри него.  
Гарантирует, что namespace не останется “подвешенным” с зависимыми ресурсами.

### **4.5.10. ServiceAccount Controller**

Создаёт сервисные аккаунты по умолчанию в новых namespace.  
Обеспечивает наличие `default` ServiceAccount.  
Интегрируется с токенами и механизмами аутентификации.

### **4.5.11. PV Controller**

Управляет жизненным циклом PersistentVolume.  
Следит за состоянием PV (Available, Bound, Released, Failed).  
Выполняет привязку и освобождение томов в зависимости от политики.

### **4.5.12. PVC Controller**

Обрабатывает PersistentVolumeClaim.  
Ищет подходящий PV под запрос PVC или инициирует динамическое выделение.  
Обновляет статус PVC (Pending, Bound).

### **4.5.13. ResourceQuota Controller**

Следит за квотами в namespace.  
Не даёт создавать ресурсы, если квоты превышены.  
Обновляет usage и статус квот при изменении объектов.

### **4.5.14. HorizontalPodAutoscaler Controller**

Работает вместе с metrics-server.  
Считывает метрики (CPU/Memory/кастомные) и масштабирует Deployment/ReplicaSet.  
Обеспечивает автоматическое горизонтальное масштабирование.

### **4.5.15. Certificate Controller**

Обрабатывает CSR (CertificateSigningRequest), если включён соответствующий функционал.  
Может автоматически подписывать сертификаты для kubelet и других компонентов.  
Интегрируется с Kubernetes CA.

### **4.5.16. Token Controller**

Создаёт и обновляет токены сервисных аккаунтов (legacy‑механизм).  
Обеспечивает доступ Pod’ов к API через ServiceAccount.  
В новых версиях частично заменяется другими механизмами, но концептуально остаётся контроллером.

### **4.5.17. Garbage Collector**

Удаляет объекты, на которые больше никто не ссылается (ownerReferences).  
Очищает “мусор” после удаления родительских объектов.  
Предотвращает накопление неиспользуемых ресурсов.

### **4.5.18. TTL Controller**

Удаляет завершённые Job и другие объекты с TTL.  
Следит за временем жизни объектов, у которых задано ограничение.  
Помогает автоматически очищать устаревшие ресурсы.

---

## **4.6. Компоненты, работающие как Pod’ы (add-on)**

Эти компоненты **не входят** в состав `kube-controller-manager` и **не являются контроллерами** в архитектурном смысле.

Они **не являются контроллерами**, потому что не реализуют классическую модель Kubernetes‑контроллера — то есть не выполняют reconciliation loop: *«desired state → observe → reconcile»*.  
Эти компоненты **не управляют объектами API** и **не изменяют состояние кластера**, а лишь выполняют системные функции (сетевой dataplane, DNS, ingress), работая как агенты или сервисы, а не как контроллеры.

Они устанавливаются манифестами и работают как Deployment или DaemonSet.

### **kube-proxy (DaemonSet)** — *сетевой агент*

kube-proxy — это **сетевой компонент Kubernetes**, который:

- реализует абстракцию Service;  
- синхронизирует Service и Endpoints с kube-apiserver;  
- создаёт правила IPVS/iptables для DNAT:  
  ```
  ClusterIP → PodIP
  NodePort → PodIP
  LoadBalancer → NodePort → PodIP
  ```
- работает на всех нодах, включая control plane;  
- не занимается маршрутизацией Pod‑IP (это делает Calico).

**Важно:**  
kube-proxy — это **агент**, а не контроллер. Он только *реагирует* на изменения объектов в API и транслирует их в правила dataplane. 
Он не выполняет reconciliation loop и не управляет объектами API.

### **Calico (DaemonSet)** — *CNI + BGP агент*

Calico — это:

- CNI‑плагин (создаёт veth, namespace, IPAM);  
- BGP‑агент (анонсирует PodCIDR ноды);  
- сетевой dataplane для Pod → Pod трафика.

Calico работает на всех нодах, включая control plane.

Calico не управляет Kubernetes‑объектами — он обслуживает сетевой dataplane.

---

### **CoreDNS (Deployment)** — *DNS‑сервер кластера*

CoreDNS:

- обслуживает DNS‑запросы Pod’ов;  
- резолвит `*.svc.cluster.local`;  
- работает как Deployment.

---

### **Ingress Controller (Deployment/DaemonSet)** — *L7‑прокси*

Например, NGINX Ingress Controller:

- принимает внешний HTTP/HTTPS трафик;  
- маршрутизирует его в Service;  
- работает как Deployment или DaemonSet.

**Почему не контроллер:**  
Ingress Controller не выполняет reconciliation loop и не управляет объектами API — он лишь конфигурирует свой L7‑прокси на основе Ingress‑ресурсов.

### **metrics-server (Deployment)** — *агент метрик*

metrics-server:

- собирает “живые” метрики с kubelet (CPU, память);  
- предоставляет данные для `kubectl top` и HorizontalPodAutoscaler;  
- не хранит долгосрочные метрики;  
- не является частью control plane.

**Почему не контроллер:**  
metrics-server не изменяет состояние кластера — он только предоставляет данные.

### **CSI Controllers (Deployment/StatefulSet)** — *контроллеры хранения, но не часть control plane*

CSI‑контроллеры включают:

- **provisioner** — создаёт тома;  
- **attacher** — подключает тома к Pod’ам;  
- **resizer** — изменяет размер PVC;  
- **snapshotter** — управляет снапшотами.

Они действительно являются *контроллерами* в терминах CSI‑спецификации, но **не относятся к control plane Kubernetes** — это внешние контроллеры, работающие поверх CSI API.

**Почему не контроллеры Kubernetes:**  
Они управляют *томами*, а не Kubernetes‑объектами.  
Они не входят в `kube-controller-manager` и не являются частью control plane.

---

## **4.7. Операторы и CRD**

Операторы — это контроллеры, работающие с Custom Resource Definitions (CRD).

Они:

- добавляют новые типы объектов (CRD) в API;  
- реализуют сложную доменную логику (БД, очереди, мониторинг);  
- используют тот же паттерн reconciliation: spec vs status.

Примеры:

- оператор PostgreSQL;  
- оператор Kafka;  
- оператор Prometheus.

В нашем кластере операторы будут использоваться позже, но архитектурно мы рассматриваем их как **частный случай контроллеров + CRD**.

---

## **4.8. Как контроллеры взаимодействуют с API**

Все контроллеры работают по одной схеме:

```text
loop:
    read desired state from API
    read actual state from API
    compare
    apply changes via API
```

Ключевые инварианты:

- контроллеры **никогда не пишут напрямую в etcd**;  
- все операции идут через kube-apiserver;  
- контроллеры не меняют spec, они приводят status к spec.

---

## **4.9. Почему контроллеры — сердце Kubernetes**

Контроллеры обеспечивают:

- self‑healing (автоматическое восстановление);  
- reconciliation (приведение к желаемому состоянию);  
- автоматизацию и масштабирование;  
- управление жизненным циклом объектов;  
- предсказуемость поведения кластера.

Без контроллеров Kubernetes был бы просто набором процессов и API.  
Именно контроллеры превращают его в систему, которая “сама приводит себя в порядок”.

---

Следующий файл:

📘 `docs/00-overview/05-workers.md`  
**5. Worker Nodes — детальный обзор**
