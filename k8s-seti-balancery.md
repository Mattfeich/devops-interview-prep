---
layout: default
title: Kubernetes — под, сеть, Ingress, gRPC, балансировщики
---

[← Главная](./index.html) | [← Linux и сети для чайника](./linux-baza.html)

# Kubernetes: сущности, спавн пода, сеть, Ingress/Gateway API, gRPC, балансировщики

---

## 1. Основные сущности Kubernetes — какая за что отвечает

Отвечай группами, не вываливай списком:

**Запуск приложений:**
- **Pod** — минимальная единица: один или несколько контейнеров с общей сетью (один IP) и общими томами. Поды смертны — их не создают руками, ими управляют контроллеры.
- **ReplicaSet** — держит заданное число одинаковых подов (упал — пересоздал). Руками не пишется — его создаёт Deployment.
- **Deployment** — декларативное управление stateless-приложением: реплики + rolling update (постепенная замена подов при новой версии) + rollback.
- **StatefulSet** — то же для stateful (БД, Kafka, MinIO): стабильные имена и диски (см. п. 3).
- **DaemonSet** — по одному поду на каждой ноде (агенты: node-exporter, сборщик логов, CNI).
- **Job / CronJob** — разовая задача до успешного завершения / по расписанию.

**Сеть и доступ:**
- **Service** — постоянный виртуальный IP + DNS-имя над меняющимся набором подов; балансирует между ними (см. п. 6).
- **EndpointSlice** — актуальный список IP подов за сервисом (заполняется автоматически по label selector).
- **Ingress / Gateway API (HTTPRoute)** — маршрутизация внешнего HTTP(S) внутрь (см. пп. 7–8).
- **NetworkPolicy** — файрвол между подами.

**Конфигурация:**
- **ConfigMap** — нечувствительная конфигурация (файлы/переменные), **Secret** — то же для секретов (base64 — это кодирование, не шифрование! нужен encryption at rest или внешний vault).
- **Namespace** — логическое разделение кластера (команды, окружения) + границы для квот и RBAC.

**Диски:** **PV** (кусок хранилища), **PVC** (заявка пода на хранилище), **StorageClass** (тип хранилища + автоматическое создание PV через CSI-драйвер).

**Прочее нужное:** **HPA** (автомасштабирование по метрикам), **PDB** (сколько реплик нельзя ронять при работах), **ServiceAccount + RBAC** (п. 4), **CRD** (свои типы ресурсов — на них строятся операторы), **LimitRange/ResourceQuota** (лимиты по неймспейсу).

## 2. Как спавнится под — полный путь для чайника

Вопрос-фаворит: «что происходит после `kubectl apply -f deployment.yaml`?» Рассказывай как историю — каждый участник делает маленькое дело, никто не командует всем процессом:

**Шаг 1. kubectl → API-серверу.** kubectl читает YAML, превращает в JSON и шлёт HTTPS-запрос на **kube-apiserver**. Это единственная входная дверь кластера — все и всё общаются только через него.

**Шаг 2. API-сервер проверяет запрос.** Три фильтра по очереди:
- **Аутентификация** — кто ты (сертификат/токен)?
- **Авторизация (RBAC)** — можно ли тебе создавать Deployment в этом неймспейсе?
- **Admission controllers** — цепочка «дополнительных проверяльщиков»: **mutating** могут дописать поля (вставить sidecar, проставить дефолты), **validating** — запретить (нельзя без limits, нельзя образ из чужого registry). Это webhooks — сюда встраиваются Istio, Kyverno, OPA.

**Шаг 3. Запись в etcd.** API-сервер сохраняет объект Deployment в **etcd** — распределённую key-value базу, единственное место, где живёт всё состояние кластера. Всё, kubectl получил «created». Пода ещё нет — есть только *желание*.

**Шаг 4. Контроллеры доводят мир до желаемого.** В **controller-manager** крутятся циклы согласования (reconcile loop): «сравни желаемое с реальным, устрани разницу».
- Deployment-контроллер видит новый Deployment → создаёт **ReplicaSet**.
- ReplicaSet-контроллер видит «нужно 3 реплики, есть 0» → создаёт 3 объекта **Pod**. У подов пока нет ноды — статус **Pending**.

**Шаг 5. Scheduler выбирает ноду.** **kube-scheduler** следит за подами без ноды. Для каждого:
- **Фильтрация**: отсеять ноды, куда нельзя (не хватает CPU/памяти под requests, мешает taint без toleration, не совпала nodeAffinity, нет нужного тома).
- **Скоринг**: оставшиеся ранжируются (меньше загружена, образ уже скачан, размазать по зонам).
Победителю под «назначается» — scheduler просто пишет `nodeName` в объект пода. Сам он ничего не запускает.

**Шаг 6. kubelet запускает.** **kubelet** — агент на каждой ноде — видит через API-сервер «мне назначили под»:
- Говорит container runtime (**containerd**) через CRI-интерфейс: скачай образы, создай контейнеры.
- Сначала создаётся **pause-контейнер** — крошечный контейнер-«держатель»: его единственная работа — владеть сетевым namespace пода. Все контейнеры пода подключаются к его сети — поэтому у них общий IP и localhost. Умрёт приложение и перезапустится — IP сохранится, потому что pause жив.
- containerd через **runc** создаёт процессы: namespaces + cgroups (лимиты из `resources`).

**Шаг 7. CNI выдаёт сеть.** kubelet зовёт **CNI-плагин** (Calico/Cilium/Flannel): тот создаёт виртуальный сетевой кабель (veth-пару), выдаёт поду **IP из подсети ноды**, настраивает маршруты. Теперь под достижим из любой точки кластера.

**Шаг 8. Пробы и Ready.** kubelet запускает startup/readiness/liveness пробы. Прошла readiness → под **Ready**.

**Шаг 9. Под попадает в балансировку.** Endpoint-контроллер видит Ready-под с нужными labels → добавляет его IP в **EndpointSlice** сервиса. **kube-proxy** на всех нодах обновляет правила (iptables/IPVS) — трафик на ClusterIP сервиса теперь может попасть и в новый под.

Запомни главное свойство: **это хореография, а не дирижёр** — нет компонента, который ведёт процесс от начала до конца. Каждый смотрит на своё и делает маленький шаг; из этого складывается самовосстановление: под умер → ReplicaSet-контроллер увидит расхождение и создаст новый.

Диагностика по шагам: не создаётся объект — RBAC/admission (ошибка сразу в kubectl); Pending — шаг 5 (`kubectl describe pod` → Events: почему нет подходящей ноды); ContainerCreating долго — шаги 6–7 (образ качается, том не цепляется, CNI); CrashLoopBackOff — приложение падает после старта (`kubectl logs --previous`).

## 3. StatefulSet vs Deployment

**Deployment** — для взаимозаменяемых копий (веб-серверы): поды со случайными именами (`web-7d9f8b-x2k4j`), общие для всех тома, при обновлении — в произвольном порядке, любой под может умереть и возродиться другим.

**StatefulSet** — для тех, у кого есть личность и свои данные (PostgreSQL, Kafka, MinIO):
- **Стабильные имена**: `kafka-0`, `kafka-1`, `kafka-2` — и стабильные DNS-адреса (`kafka-0.kafka-headless.ns.svc.cluster.local`) через headless service. Брокер №0 всегда «нулевой», где бы ни запустился.
- **Свой диск у каждого**: `volumeClaimTemplates` создаёт отдельный PVC на каждую реплику; `kafka-0` умер и переехал — его PVC поедет с ним. У Deployment все реплики делили бы один PVC.
- **Порядок**: создание 0→1→2, удаление в обратном, обновление по одному (важно для кворумов).

Формула для собеса: «Deployment — скот (cattle), StatefulSet — питомцы (pets): у пода есть стабильное имя, стабильный диск и порядок операций».

## 4. Какие сущности настраивают RBAC?

Четыре объекта + субъекты:
- **Role** — набор разрешений **внутри одного namespace**: «глаголы (get/list/create/delete) над ресурсами (pods/secrets/…)».
- **ClusterRole** — то же, но на весь кластер (или для некластерных ресурсов: nodes, PV).
- **RoleBinding** — привязывает Role (или ClusterRole!) к субъекту в рамках namespace.
- **ClusterRoleBinding** — привязывает ClusterRole на весь кластер.
- Субъекты: **User/Group** (люди — из сертификатов/OIDC, объектов в K8s для них нет) и **ServiceAccount** (учётка для программ/подов; под получает её токен и ходит в API от её имени).

Мини-пример, который стоит уметь написать:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata: { name: pod-reader, namespace: dev }
rules:
- apiGroups: [""]
  resources: ["pods", "pods/log"]
  verbs: ["get", "list", "watch"]
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata: { name: read-pods, namespace: dev }
subjects:
- kind: ServiceAccount
  name: ci-bot
  namespace: dev
roleRef: { kind: Role, name: pod-reader, apiGroup: rbac.authorization.k8s.io }
```

Проверка прав: `kubectl auth can-i delete pods --as system:serviceaccount:dev:ci-bot -n dev`. Нюанс на плюс: RBAC только **разрешает** (deny-правил нет; нет разрешения = запрещено), права аддитивны.

## 5. Что будет, если посмотреть память внутри контейнера? (вопрос-ловушка)

`free -h` или `top` внутри контейнера покажут **память всей ноды**, а не лимит контейнера. Причина: эти утилиты читают `/proc/meminfo`, а procfs **не изолирован namespace'ами** — контейнер видит хостовые цифры. То же с CPU: `nproc` покажет все ядра ноды, хотя лимит у тебя 0.5 CPU.

Как посмотреть правду — читать cgroup самого контейнера:

```bash
cat /sys/fs/cgroup/memory.max       # лимит (cgroup v2)
cat /sys/fs/cgroup/memory.current   # текущее потребление
kubectl top pod                      # снаружи, через metrics-server
```

Почему это важный практический вопрос: приложения, которые сами решают, сколько памяти взять, глядя в /proc — исторически JVM («возьму 25% RAM») — брали 25% от ноды, пробивали лимит контейнера и получали OOMKilled. Современные JVM (10+) умеют читать cgroups (`UseContainerSupport`, `-XX:MaxRAMPercentage`); у Go — `GOMEMLIMIT`, автоматизируется automemlimit/uber-go. Отвечая, обязательно назови и следствие: exit code 137 = OOMKilled по cgroup-лимиту, даже если на ноде памяти навалом.

---

## 5.1 Deployment vs ReplicaSet (и другие частые пары)

- **ReplicaSet vs Deployment**: ReplicaSet умеет одно — держать N копий пода живыми. Deployment — обёртка над ReplicaSet, добавляющая **версионирование и обновления**: на каждую новую версию шаблона пода Deployment создаёт **новый** ReplicaSet и постепенно перегоняет реплики из старого в новый (rolling update: параметры `maxSurge`/`maxUnavailable`), старые RS хранит для отката (`kubectl rollout undo`). Поэтому руками ReplicaSet не пишут никогда — всегда Deployment. Формула: «RS — про количество, Deployment — про количество + версии + откат».
- **Deployment vs StatefulSet** — см. п. 3 (имена, диски, порядок).
- **DaemonSet vs Deployment** — «по одному на каждой ноде» vs «N штук где угодно».
- **Job vs Deployment** — «выполнить до успеха и остановиться» vs «работать вечно».
- **kubectl apply vs create** — декларативное «приведи к этому состоянию» (можно повторять, считает diff) vs императивное «создай» (второй раз — ошибка AlreadyExists).
- **Service vs Ingress** — L4-балансировка внутри кластера vs L7-маршрутизация HTTP снаружи (см. ниже).

## 6. Как идёт обращение к подам: Service от и до

Проблема, которую решает Service: поды смертны, их IP меняются при каждом пересоздании. Нужен постоянный адрес.

**ClusterIP (дефолт)** — виртуальный IP + DNS-имя внутри кластера:

1. Создаёшь Service с `selector: app=api` → контроллер находит все Ready-поды с этим лейблом и складывает их IP в **EndpointSlice** (обновляется сам при каждом рождении/смерти пода).
2. Сервис получает **ClusterIP** (например `10.96.0.10`) — это IP-«призрак»: он не назначен ни одному интерфейсу, на него нельзя «зайти».
3. **kube-proxy** на каждой ноде пишет правила **iptables** (или IPVS): «пакет на 10.96.0.10:80 → DNAT в один из IP подов из EndpointSlice» (выбор случайный). То есть балансировка происходит **на ноде-отправителе**, в ядре, никакого процесса-посредника нет.
4. **DNS (CoreDNS)**: имя `api.myns.svc.cluster.local` → ClusterIP. Из того же неймспейса достаточно `http://api`.

iptables vs IPVS: iptables — линейный перебор правил (медленнее на тысячах сервисов), IPVS — хэш-таблицы + честные алгоритмы балансировки (rr, least conn). Cilium заменяет всё это eBPF без kube-proxy.

**Как вывести трафик наружу из кластера** (твой вопрос) — по нарастающей:

- **NodePort** — сервис получает порт из диапазона 30000–32767 **на каждой ноде**: пришёл на `любая-нода:31080` → DNAT в под (возможно, на другой ноде). Просто, но порты странные, по ноде на IP, нет умного роутинга. Обычно не используется напрямую — это кирпичик для LoadBalancer.
- **LoadBalancer** — NodePort + внешний балансировщик: в облаке контроллер создаёт настоящий облачный LB с публичным IP, который шлёт трафик на NodePort'ы нод. On-prem облачного API нет — ставят **MetalLB** (анонсирует IP через ARP или BGP). Минус: один LB (и один внешний IP) на каждый сервис — дорого.
- **Ingress / Gateway API** — правильный ответ для HTTP: **один** LoadBalancer на весь кластер, за ним ingress-контроллер маршрутизирует по доменам/путям во все сервисы (пп. 7–8).
- **ExternalName** — просто CNAME на внешнее имя (наоборот: доступ изнутри наружу под кластерным именем).
- **Headless** (`clusterIP: None`) — без виртуального IP: DNS отдаёт **список IP самих подов**. Нужен, когда клиент хочет знать всех поимённо: StatefulSet (kafka-0, kafka-1...), клиентская балансировка gRPC.
- Точечные случаи: `hostNetwork`/`hostPort` (под живёт в сети ноды — так работают ingress-контроллеры на bare-metal), `kubectl port-forward` (туннель для отладки, не для прода).

Нюанс на сеньорский плюс: `externalTrafficPolicy: Local` — слать только в поды на той ноде, куда пришёл трафик: сохраняется реальный IP клиента и нет лишнего хопа, но нужен health check LB по нодам.

## 7. Как устроена сеть в кубе (для чайника, но по-настоящему)

Три слоя, не путать их — это три разных вопроса:

**Слой 1: под ↔ под.** Правило модели K8s: у каждого пода свой IP, любой под достигает любой другой **без NAT**, плоская сеть. Как это сделано:
- Внутри ноды: у каждого пода свой network namespace; из него в хост торчит **veth-пара** (виртуальный патч-корд); все veth воткнуты в бридж (виртуальный свитч) ноды. Поды одной ноды общаются через этот бридж.
- Между нодами — два подхода CNI-плагинов:
  - **Overlay (VXLAN)** — Flannel, Calico в режиме vxlan: пакет пода заворачивается в UDP-пакет между нодами и разворачивается на той стороне. Работает поверх любой сети, чуть теряет в производительности. Здесь же классическая проблема **MTU** (инкапсуляция съедает ~50 байт).
  - **Маршрутизация (BGP)** — Calico: нода просто анонсирует «подсеть 10.244.3.0/24 — за мной», пакеты ходят нативно без заворачивания. Быстрее, но сеть должна разрешать.
  - **Cilium (eBPF)** — сеть, балансировка и политики программируются прямо в ядре, kube-proxy не нужен; сейчас это мейнстрим для новых кластеров.

**Слой 2: сервисы** — виртуальные IP поверх слоя 1 (п. 6: kube-proxy, iptables/IPVS DNAT).

**Слой 3: DNS** — CoreDNS (поды в kube-system) резолвит `<svc>.<ns>.svc.cluster.local`. Знай классику **ndots:5**: в resolv.conf пода стоит «имена с < 5 точками сначала дополнять суффиксами» → на каждый запрос `api.example.com` пробуются `api.example.com.myns.svc.cluster.local.` и ещё 3–4 варианта = ×5 нагрузка на DNS и таймауты. Лечение: FQDN с точкой на конце (`api.example.com.`), кастомный dnsConfig, NodeLocal DNSCache.

**NetworkPolicy** — файрвол по лейблам: по умолчанию разрешено всё; появилась политика, выбирающая под → всё, кроме перечисленного, запрещено. Работает только если CNI это умеет (Calico/Cilium да, голый Flannel нет). Типовой паттерн: default-deny в неймспейсе + точечные разрешения.

## 8. Ingress — что это и как настраивать

**Ingress = правила L7-маршрутизации** («какой хост/путь → какой сервис») + **Ingress-контроллер** = движок, который эти правила исполняет (ingress-nginx, Traefik, HAProxy-ingress). Ingress без контроллера — просто мёртвая запись в etcd; это любимая ловушка на собесе.

Путь трафика целиком: клиент → внешний LB (Service type LoadBalancer контроллера) → под ingress-контроллера (реально nginx с автогенерируемым конфигом) → **напрямую в поды** приложения (контроллер берёт IP из EndpointSlice, минуя ClusterIP — сам балансирует).

Рабочий пример со всем, что спрашивают:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"       # http → https
    nginx.ingress.kubernetes.io/proxy-body-size: "50m"
    cert-manager.io/cluster-issuer: letsencrypt            # авто-сертификат
spec:
  ingressClassName: nginx            # каким контроллером обрабатывать
  tls:
  - hosts: [api.example.com]
    secretName: api-tls              # Secret с сертификатом (кладёт cert-manager)
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /v1
        pathType: Prefix             # Prefix | Exact
        backend:
          service: { name: api-v1, port: { number: 80 } }
      - path: /v2
        pathType: Prefix
        backend:
          service: { name: api-v2, port: { number: 80 } }
```

Что уметь рассказать вокруг: TLS-termination на контроллере (дальше внутрь — обычно plain HTTP), host-based и path-based routing, canary через аннотации (`canary-weight: "10"` — 10% трафика новой версии), таймауты/размеры тела — всё через аннотации. И главную боль тоже: аннотации — нестандарт, у каждого контроллера свои → это одна из причин появления Gateway API.

## 9. Gateway API и HTTPRoute — чем отличается от Ingress

**Gateway API** — преемник Ingress (GA с 2023). Три причины замены — так и отвечай:
1. **Ingress ограничен**: только HTTP, всё нестандартное — через вендорские аннотации (не переносимо, не типизировано).
2. **Нет разделения ролей**: в Ingress всё в одном объекте; в Gateway API инфраструктурная команда владеет Gateway (точки входа, TLS, IP), а команды приложений — своими Route'ами в своих неймспейсах.
3. **Больше протоколов**: HTTPRoute, **GRPCRoute**, TCPRoute, TLSRoute.

Сущности: **GatewayClass** (какая реализация: istio, cilium, nginx-gateway) → **Gateway** (конкретная точка входа: листенеры-порты, сертификаты) → **HTTPRoute** (правила маршрутизации, привязываются к Gateway).

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata: { name: main-gw, namespace: infra }
spec:
  gatewayClassName: nginx
  listeners:
  - name: https
    port: 443
    protocol: HTTPS
    tls: { certificateRefs: [{ name: wildcard-tls }] }
    allowedRoutes: { namespaces: { from: All } }   # чьи Route можно цеплять
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata: { name: api, namespace: team-a }
spec:
  parentRefs: [{ name: main-gw, namespace: infra }]
  hostnames: [api.example.com]
  rules:
  - matches:
    - path: { type: PathPrefix, value: /v1 }
      headers: [{ name: x-beta, value: "true" }]   # матчинг по заголовкам — Ingress так не умеет
    backendRefs:
    - { name: api-v1, port: 80, weight: 90 }        # взвешенный сплит трафика
    - { name: api-v2, port: 80, weight: 10 }        # канарейка без аннотаций, нативно
```

Ключевые фичи HTTPRoute, которых нет в Ingress: матчинг по заголовкам/query/методу, weight-сплит из коробки, модификация заголовков, mirror трафика, кросс-неймспейсные ссылки с явными разрешениями (ReferenceGrant).

## 10. gRPC — что это и чем отличается

**Суть:** gRPC — фреймворк удалённого вызова процедур (RPC): ты описываешь API в `.proto`-файле, генерируешь клиент и сервер, и клиент зовёт удалённый метод как локальную функцию. Работает поверх **HTTP/2**, данные — бинарный **Protobuf**.

| | REST/JSON | gRPC |
|---|---|---|
| Формат | текстовый JSON | бинарный protobuf (меньше, быстрее) |
| Контракт | опциональный (OpenAPI) | обязательный `.proto`, типизированный, кодогенерация |
| Транспорт | HTTP/1.1+ | HTTP/2 (мультиплексирование) |
| Стриминг | нет (SSE/WS сбоку) | нативный: server/client/bidirectional streaming |
| Читаемость/отладка | curl и глаза | нужен grpcurl/grpcui |
| Где типично | публичные API, браузеры | внутренняя связь микросервисов |

**Почему с gRPC боль у балансировщиков** — вопрос со звёздочкой, отвечай обязательно: gRPC держит **одно долгоживущее HTTP/2-соединение** и гоняет все вызовы потоками внутри него. L4-балансировщик (и обычный ClusterIP!) балансирует **соединения**, а не запросы → клиент прилип к одному поду, остальные реплики курят. Решения:
1. **L7-балансировка**: ingress-nginx (аннотация `backend-protocol: "GRPC"`), Envoy, Linkerd/Istio (sidecar балансирует per-request), Gateway API **GRPCRoute**.
2. **Client-side**: headless service → клиент знает все IP подов и балансирует сам (`grpc.lb_policy=round_robin`).

Health checks: свой протокол (grpc_health_probe; в K8s ≥1.24 — нативная `grpc:` liveness/readiness проба). Из браузера напрямую нельзя — нужен grpc-web прокси.

---

## 11. Балансировщики: nginx и HAProxy

### 11.1 L4 vs L7 — база, с которой начинай любой ответ

- **L4 (транспортный)**: балансировщик видит только IP и порты. Принял TCP-соединение → пробросил в бэкенд. Быстро, дёшево, работает с любым протоколом, но не видит содержимого: не может маршрутизировать по URL, не может ретраить HTTP-запрос, балансирует соединения, а не запросы.
- **L7 (прикладной)**: парсит HTTP — видит хост, путь, заголовки, куки. Может: маршрутизация по контенту, TLS termination, ретраи, canary, sticky sessions по куке, изменение заголовков. Дороже по CPU.

### 11.2 nginx как reverse proxy / балансировщик

Архитектура (спрашивают!): **master-процесс** (читает конфиг, управляет) + **worker-процессы** (по числу ядер), каждый воркер — **event loop**, неблокирующе обслуживающий тысячи соединений (поэтому nginx лёгок против «поток-на-соединение» у старого Apache). `reload` — плавный: master читает новый конфиг, поднимает новых воркеров, старые дорабатывают свои соединения и умирают — ни одного разорванного запроса (`nginx -s reload`, под капотом SIGHUP; сначала всегда `nginx -t` — проверка конфига).

Скелет конфига балансировки, который стоит уметь написать от руки:

```nginx
upstream backend {
    least_conn;                          # алгоритм (дефолт — round-robin)
    server 10.0.0.11:8080 weight=2;      # веса
    server 10.0.0.12:8080 max_fails=3 fail_timeout=30s;  # passive health check
    server 10.0.0.13:8080 backup;        # только если остальные умерли
    keepalive 32;                        # пул keep-alive к бэкендам
}
server {
    listen 443 ssl;
    server_name api.example.com;
    ssl_certificate     /etc/nginx/tls/fullchain.pem;    # TLS termination
    ssl_certificate_key /etc/nginx/tls/privkey.pem;
    location / {
        proxy_pass http://backend;
        proxy_set_header Host $host;                      # иначе бэкенд получит "backend"
        proxy_set_header X-Real-IP $remote_addr;          # реальный IP клиента
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_connect_timeout 3s;
        proxy_read_timeout 30s;                           # его превышение = 504
        proxy_next_upstream error timeout http_502;       # ретрай на другом бэкенде
    }
}
```

Алгоритмы: **round-robin** (дефолт), **least_conn** (кому меньше соединений — для неравных по длительности запросов), **ip_hash** (клиент прилипает к бэкенду — примитивные sticky sessions), **hash <key> consistent** (свой ключ). Health checks: в open source только **пассивные** (max_fails — выкинуть бэкенд после N ошибок); активные (`health_check` — самому опрашивать) — в NGINX Plus; в HAProxy активные бесплатны — этим часто аргументируют выбор.

Типовые вопросы по nginx: разница 502/503/504 и где искать (см. страницу Linux, п. 3.6 + `error_log`); что такое `proxy_buffering` (nginx вычитывает ответ бэкенда в буфер и отдаёт медленному клиенту сам — бэкенд освобождается быстрее); `worker_connections`/`worker_processes`; отличие `return 301` от `rewrite`; как отдать статику мимо бэкенда (`location /static { root ...; }` — nginx это ещё и веб-сервер, и кэш: `proxy_cache`).

### 11.3 HAProxy

Чистый балансировщик (не веб-сервер: статику не отдаёт, кэша нет), зато лучший в своём деле: тонкая настройка TCP, богатые ACL, активные health checks, знаменитая стабильность.

```haproxy
frontend fe_https
    bind *:443 ssl crt /etc/haproxy/certs/site.pem
    mode http
    acl is_api  path_beg /api                    # ACL — условия
    acl is_grpc hdr(content-type) -m beg application/grpc
    use_backend be_api if is_api                 # маршрутизация по ACL
    default_backend be_web

backend be_api
    mode http
    balance leastconn                            # roundrobin | leastconn | source
    option httpchk GET /healthz                  # активный health check
    default-server check inter 2s fall 3 rise 2  # каждые 2с; 3 фейла — вон, 2 успеха — назад
    cookie SRV insert indirect nocache           # sticky sessions кукой
    server app1 10.0.0.11:8080 cookie a1
    server app2 10.0.0.12:8080 cookie a2

listen stats                                     # встроенная статистика
    bind *:8404
    stats enable
    stats uri /stats
```

Понятия: **frontend** (куда приходят клиенты) / **backend** (пулы серверов) / **listen** (совмещённый); `mode tcp` (L4) vs `mode http` (L7); **stick-tables** (таблицы липкости/лимитов — на них же rate limiting и защита от брутфорса); **PROXY protocol** — способ передать реальный IP клиента сквозь L4-цепочку балансировщиков (nginx тоже умеет `proxy_protocol`); seamless reload (`-x` передача сокетов новому процессу без потери соединений).

### 11.4 nginx vs HAProxy — как отвечать на «что выберешь?»

- nginx: универсал — веб-сервер + reverse proxy + кэш + статика + TLS; в K8s ingress-nginx — дефолтный контроллер. Балансировка в OSS-версии упрощённая (нет активных health checks).
- HAProxy: специализированный LB — активные health checks, ACL, stick-tables, детальная статистика, лучший контроль TCP.
- Честный ответ: «для edge/статики/универсального прокси — nginx; для чистой балансировки с жёсткими требованиями — HAProxy; в облаке чаще берём managed LB, в K8s — ingress-контроллер».

### 11.5 Высокая доступность самого балансировщика

Балансировщик — единая точка отказа, поэтому: пара LB **active-passive** с **keepalived (VRRP)** — общий виртуальный IP (VIP) держит мастер, умер — VIP за секунды переезжает на резервного. Дальше по нарастающей: DNS round-robin (грубо, TTL мешает), **anycast** (один IP анонсируется из многих точек — так живут CDN и 8.8.8.8). Плюс **connection draining** при выводе бэкенда: новых запросов не давать, старым дать дозавершиться.

---

## 12. Логи и траблшутинг: Docker и Kubernetes

### 12.1 Docker

```bash
docker ps -a                        # все контейнеры (и мёртвые! смотри STATUS/Exited code)
docker logs -f --tail 100 app       # stdout/stderr контейнера (докер их перехватывает)
docker logs app 2>&1 | grep -i error
docker inspect app                  # вся конфигурация: env, mounts, сеть, causa смерти
docker inspect app --format '{{.State.OOMKilled}} {{.State.ExitCode}}'
docker exec -it app sh              # зайти внутрь живого контейнера
docker stats                        # live CPU/память/сеть по контейнерам
docker events                       # поток событий демона (что и когда умерло)
docker system df / docker system prune   # кто съел диск
```

Куда пишутся логи: приложение в контейнере должно писать в **stdout/stderr** (не в файлы!) — их подхватывает **logging driver** (по умолчанию `json-file`: файлы в `/var/lib/docker/containers/<id>/*-json.log`). Классическая авария «диск кончился из-за логов» лечится ротацией в `/etc/docker/daemon.json`: `{"log-driver":"json-file","log-opts":{"max-size":"50m","max-file":"3"}}`. Прод-агрегация: fluent-bit/vector читают эти файлы → Loki/ELK.

Алгоритм «контейнер не работает»: `docker ps -a` (жив? код выхода: 137 = OOM/kill, 126/127 = нет команды, 1 = ошибка приложения) → `docker logs` → `docker inspect` (OOMKilled? рестарты? env? volume смонтирован?) → `docker exec` внутрь → если не стартует вообще: `docker run -it --entrypoint sh образ` — зайти в образ мимо сломанного entrypoint.

### 12.2 Kubernetes

```bash
kubectl get pods -o wide                     # статусы, рестарты, ноды
kubectl describe pod app-xxx                 # СНАЧАЛА ЭТО: Events внизу — 90% ответов
kubectl logs app-xxx                         # логи контейнера
kubectl logs app-xxx --previous              # логи ПРЕДЫДУЩЕГО (упавшего) контейнера — ключ к CrashLoop
kubectl logs app-xxx -c sidecar              # конкретный контейнер пода
kubectl logs deploy/app --tail 100 -f        # сразу по deployment
kubectl get events -A --sort-by=.lastTimestamp   # события всего кластера по времени
kubectl exec -it app-xxx -- sh               # внутрь
kubectl debug -it app-xxx --image=nicolaka/netshoot --target=app   # ephemeral-контейнер с тулзами
                                             # (спасает в distroless-образах, где нет даже sh)
kubectl port-forward svc/app 8080:80         # дотянуться до сервиса локально
kubectl top pods / kubectl top nodes         # потребление ресурсов
kubectl rollout status/history/undo deploy/app
```

Дерево диагностики (расскажи его — это сильный ответ):
1. **Pending** → describe → Events: «Insufficient cpu/memory» (нет места — смотри requests и ноды), «had untolerated taint», «unbound PersistentVolumeClaim».
2. **ImagePullBackOff** → опечатка в теге / приватный registry без imagePullSecrets / rate limit DockerHub.
3. **CrashLoopBackOff** → `logs --previous` + exit code в describe: 137 — OOMKilled (поднять limits) или прибит liveness-пробой; 1 — ошибка приложения (конфиг, БД недоступна). Смотри также: пробы слишком агрессивные?
4. **Running, но не работает** → readiness провалена? `kubectl get endpoints app` пустой → селектор сервиса не совпадает с labels подов (классика!) / NetworkPolicy режет.
5. **Всё, что с сетью** → тестовый под с тулзами: `kubectl run tmp --rm -it --image=nicolaka/netshoot -- bash`, изнутри: nslookup сервиса, curl, трассировка.
6. Нода: `kubectl describe node` (Conditions: MemoryPressure/DiskPressure), на самой ноде — `journalctl -u kubelet`, `crictl ps` (контейнеры глазами runtime, когда kubectl уже не помогает).

[← Главная](./index.html) | [Linux и сети для чайника](./linux-baza.html) | [Docker, DevOps, основы →](./osnovy.html)
