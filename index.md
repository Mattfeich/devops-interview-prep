---
layout: default
title: DevOps Interview Prep
---

# Подготовка к собеседованию: DevOps / SRE — Data Lake Platform

> **Разделы сайта:**
> 1. Эта страница — основной гайд по 9 блокам вакансии (уровень собеса)
> 2. [Linux и сети — разжёвано для чайника](./linux-baza.html) — bash, процессы/потоки, сигналы, TCP/UDP, DNS, HTTP(S), файрволы, права, загрузка ОС
> 3. [Kubernetes: под, сеть, Ingress, Gateway API, gRPC, балансировщики](./k8s-seti-balancery.html) — спавн пода по шагам, Service/сеть, nginx/HAProxy, логи и траблшутинг
> 4. [Docker, DevOps-подходы, безопасность — основы](./osnovy.html) — что такое Docker/K8s/DevOps, GitOps, CI/CD, Dockerfile, шифрование vs хэширование
> 5. [Блиц «в чём разница между…»](./raznitsa.html) — GET/POST, fork/pull, netstat/ss, roles/playbooks, syscall, privileged-контейнеры, ICMP и др.


Шпаргалка по всем блокам вакансии: теория → типовые вопросы → сильные ответы → подвохи. Читай блок, потом идём в тренировку вопросами.

**Как отвечать на собесе (общие правила):**

- Структура ответа: сначала короткий прямой ответ (1–2 фразы), потом детали. Не начинай с деталей.
- Не знаешь — говори честно: «С этим вплотную не работал, но по логике это устроено так…». Попытка рассуждать ценится, блеф — нет.
- На вопросы про troubleshooting всегда отвечай методологией: «сначала смотрю X, если там Y — иду в Z», а не перечислением утилит вразнобой.
- Везде, где можешь, приземляй на свой опыт: «у нас было так — …».

---

## Блок 1. Linux (экспертный уровень)

### 1.1 Что происходит при запуске команды (вопрос-фильтр)

Классика: «что происходит от Enter после `ls -l` до вывода». Сильный ответ по шагам:

1. **Bash читает строку** (readline), парсит: разворачивает алиасы, glob (`*`), переменные, проверяет — не builtin ли это (`cd`, `echo` — builtin; `ls` — нет).
2. **Поиск бинарника**: bash идёт по каталогам из `$PATH` (с кэшем в hash-таблице — `hash -r` его сбрасывает), находит `/usr/bin/ls`.
3. **fork()** — bash клонирует себя (copy-on-write: страницы памяти не копируются, а помечаются read-only и копируются при первой записи).
4. **execve("/usr/bin/ls", ["ls","-l"], envp)** — ядро загружает ELF: читает заголовок, видит интерпретатор `/lib64/ld-linux-x86-64.so.2` (динамический линкер), тот мапит libc и другие .so через `mmap`, делает релокации, передаёт управление в `main`.
5. **ls работает**: `openat(AT_FDCWD, ".")` → `getdents64()` (читает записи каталога) → `statx()` на каждый файл (права, владелец, размер — для `-l`) → обращения к `/etc/passwd`, `/etc/group` (uid→имя).
6. **Вывод**: `write(1, ...)` в stdout — это tty (или pipe, если `ls | ...`). Терминал рисует.
7. **Завершение**: `exit_group()`, bash ловит через `wait4()`, забирает exit code в `$?`, зомби убирается.

Подвох-добивка: «а если бинарника нет в PATH?» — bash вернёт 127. «А чем builtin отличается?» — не форкается, выполняется внутри процесса шелла (поэтому `cd` не может быть внешней командой).

### 1.2 Load Average — самый частый подвох

**LA — это не CPU usage.** Это среднее число процессов в состоянии **R (runnable)** + **D (uninterruptible sleep)** — экспоненциально сглаженное за 1/5/15 минут. В Linux (в отличие от классических Unix) D-процессы **входят** в LA.

Отсюда классический сценарий: **LA=40 при idle CPU 90%** → процессы стоят в D-state, ждут I/O: умирающий диск, перегруженный/отвалившийся NFS, зависший FUSE. Диагностика:

```
ps -eo pid,state,wchan:32,cmd | awk '$2=="D"'   # кто в D и чего ждёт
cat /proc/<PID>/stack                            # kernel stack — где именно застрял
iostat -x 1                                      # await, %util — какой диск
dmesg -T | tail                                  # ошибки ядра: I/O errors, hung task
```

Правило интерпретации: LA сравнивают с числом ядер (`nproc`). LA=8 на 8 ядрах — под завязку, LA=8 на 64 — тишина.

### 1.3 Память: free, cache, dirty pages

- **buff/cache** — page cache (кэш файлов) + буферы блочных устройств. Это **не занятая** память: ядро отдаст её приложениям при нехватке. Смотреть надо на **available**, а не на free. «Кэш занял 90% RAM» — это норма и хорошо (особенно для Kafka и MinIO, которые живут на page cache).
- **Dirty pages** — изменённые страницы кэша, ещё не записанные на диск. Пишутся writeback-потоками: по таймеру (`vm.dirty_expire_centisecs`), по порогу `vm.dirty_background_ratio` (фоново) и `vm.dirty_ratio` (процессы начинают писать синхронно — вот откуда «всё зависло при копировании большого файла»). Принудительно — `sync`/`fsync`.
- **available vs free**: free — прямо сейчас никем не занято; available — оценка, сколько можно выделить без ухода в swap (free + reclaimable-кэш).
- **swappiness** (`vm.swappiness`, 0–200): склонность свопить анонимную память vs выбрасывать page cache. Для БД/storage обычно 1–10.
- **Major/minor page fault**: minor — страница есть в памяти, нет маппинга; major — надо читать с диска (swap-in или чтение файла). Много major faults = memory pressure.

### 1.4 OOM Killer

Когда памяти нет совсем (и reclaim не помог), ядро убивает процесс с наибольшим **oom_score** (примерно = доля потребляемой памяти, корректируется `oom_score_adj` от −1000 «не убивать» до +1000). Смотреть: `dmesg -T | grep -i oom` — там полный дамп: кто убит, сколько ел, состояние памяти. Важное для контейнеров: **cgroup OOM** — контейнер убивается при превышении своего лимита, даже если на ноде памяти полно; в K8s это `OOMKilled` (exit code 137).

### 1.5 Процесс в D-state не убивается kill -9

Почему: сигналы обрабатываются при возврате из syscall в userspace, а D-процесс сидит **внутри** ядерного вызова (обычно I/O) и не возвращается, пока I/O не завершится или не отвалится по таймауту. `kill -9` встанет в очередь и сработает, когда (если) процесс выйдет из ядра. Что делать: искать причину I/O (диск/NFS). NFS с `hard`-маунтом без `intr` — вечный D. Если ядро видит зависание >120с — `hung_task` warning в dmesg.

### 1.6 «No space left», а df показывает 40%

Три причины — назвать минимум две:

1. **Кончились inode**: `df -i`. Типично при миллионах мелких файлов (кэш-каталоги, sessions, maildir).
2. **Удалённые, но открытые файлы**: процесс держит fd на удалённый файл (часто — логи, удалённые rm вместо truncate). `df` показывает занято, `du` — нет. Найти: `lsof +L1`. Лечить: перезапуск/`truncate -s 0 /proc/PID/fd/N`.
3. Реже: квоты, кончилось место в конкретном tmpfs/overlay, зарезервированные 5% для root на ext4 (`tune2fs -m`).

### 1.7 Namespaces и cgroups — фундамент контейнеров

- **Namespaces** — изоляция видимости: pid, net, mnt, uts, ipc, user, cgroup, time. Контейнер = процесс в наборе namespaces.
- **cgroups** (v2 — единая иерархия, `/sys/fs/cgroup`) — лимиты ресурсов: cpu (weight/max), memory (max/high), io, pids. Docker/K8s лимиты = записи в cgroup.
- **CPU limit в контейнере** работает через CFS quota: `cpu.max = quota period` (например 100ms из 100ms = 1 CPU). Превысил — **throttling**: процесс замораживают до конца периода. Это причина латенси-спайков в K8s при низких limits — смотреть `container_cpu_cfs_throttled_periods_total`.

### 1.8 systemd

- Юниты: service, timer, mount, socket, target. `systemctl status/cat/edit`, drop-in'ы в `/etc/systemd/system/foo.service.d/`.
- Дебаг сервиса: `systemctl status foo` → `journalctl -u foo -e --since "-1h"` → `systemd-analyze verify`.
- Полезное в юнитах: `Restart=on-failure`, `LimitNOFILE`, `MemoryMax` (это cgroup), `After=/Requires=` (порядок vs зависимость — разные вещи!), `Type=notify/simple/forking`.
- `journalctl`: `-u`, `-p err`, `-f`, `--disk-usage`, persistent storage в `/var/log/journal`.

### 1.9 Сеть в Linux (база, глубже — в блоке troubleshooting)

- `ss -tlnp` (кто слушает), `ss -s` (сводка), `ip a/r`, `conntrack -S`.
- Файловые дескрипторы и лимиты: `ulimit -n`, `fs.file-max`, у systemd-сервисов — `LimitNOFILE`.
- TIME_WAIT — нормальное состояние закрывшей соединение стороны (2*MSL); тысячи TIME_WAIT — не проблема сама по себе; проблема — исчерпание эфемерных портов (`ip_local_port_range`).

### 1.10 Тюнинг под storage-нагрузку (пригодится для MinIO-вопросов)

- ФС: **XFS** (рекомендация MinIO), маунт `noatime`.
- I/O scheduler: для NVMe — `none`, для SATA/SAS — `mq-deadline`. `cat /sys/block/nvme0n1/queue/scheduler`.
- `vm.dirty_ratio`/`background_ratio` пониже при больших объёмах RAM, чтобы не копить гигантские writeback-всплески.
- THP (transparent huge pages) — выключать для БД/латенси-чувствительных нагрузок.
- NUMA: `numactl --hardware`, пиннинг для критичных сервисов.

### Типовые вопросы блока (проверь, что можешь ответить)

- inode — что хранит? (метаданные + указатели на блоки; имя файла — НЕ в inode, а в каталоге). Hardlink vs symlink.
- Зомби-процесс — что это и чем вреден? (завершился, родитель не сделал wait; вреден только занятой записью в таблице процессов; лечится починкой/убийством родителя).
- strace vs ltrace vs perf: syscalls / library calls / профилирование по сэмплам, flamegraph.
- Чем нить отличается от процесса? (общее адресное пространство; в ядре и то и то — task).
- Что такое file descriptor, что значит «everything is a file».
- iowait — что это на самом деле? (CPU idle, пока есть невыполненный I/O; ненадёжная метрика, на многоядерных может «прятаться»).
- Runlevel/targets, что происходит при загрузке: BIOS/UEFI → GRUB → ядро+initramfs → systemd → targets.

---

## Блок 2. S3 / Object Storage / MinIO / Distributed Storage

### 2.1 Object storage vs block vs file

- **Block** (iSCSI, EBS, Ceph RBD): сырые блоки, поверх — ФС; низкая латенси, одна машина-потребитель.
- **File** (NFS, CephFS): иерархия, POSIX-семантика, локи, частичная запись.
- **Object** (S3, MinIO, Ceph RGW): плоское пространство ключей, HTTP API, объект **иммутабелен** (перезапись = новый объект целиком, нет частичной записи), метаданные с объектом, масштабируется горизонтально практически бесконечно. Плата: нет POSIX, нет rename (это copy+delete), LIST — дорогой.

### 2.2 S3 API — что надо знать

- Bucket/key, **multipart upload** (части от 5 MiB, до 10000 частей; параллельная заливка больших объектов; незавершённые куски копятся и жрут место — чистить lifecycle-правилом `AbortIncompleteMultipartUpload`).
- **Presigned URLs** — временный доступ по подписи без раздачи ключей.
- **Versioning**, delete marker; **Object Lock / WORM** (compliance).
- **Lifecycle**: транзишены между storage classes, экспирация, чистка версий.
- **Consistency в AWS S3**: с декабря 2020 — **strong read-after-write** для всех операций (раньше eventual для перезаписи/LIST — если спросят «S3 eventual consistent?», правильный ответ: исторически да, сейчас strong). MinIO — тоже strict consistency.
- Аутентификация: SigV4, ключи/роли/STS; шифрование SSE-S3/SSE-KMS/SSE-C.

### 2.3 Replication vs Erasure Coding — главная теория блока

- **Репликация** (например 3x): N полных копий. Просто, быстрый ремонт (копируешь целиком с живой реплики), быстрое чтение. Оверхед хранения ×3 (33% полезной ёмкости).
- **Erasure Coding (Reed–Solomon)**: объект бьётся на **K data-шардов + M parity-шардов**; любые K из (K+M) шардов достаточны для восстановления. Пример EC 8+4: переживает потерю 4 дисков, оверхед всего 1.5x (67% полезной ёмкости). Плата: CPU на кодирование, при чтении/ремонте надо собрать K шардов по сети (ремонт дороже, чем у репликации), выше латенси на мелких объектах.
- Правило выбора: горячие мелкие данные/метаданные — репликация; большие объёмы объектного хранилища — EC.

### 2.4 MinIO — архитектура

- **Distributed mode**: N серверов × M дисков. Диски объединяются в **erasure sets** (страйпы по 4–16 дисков), объекты распределяются по сетам хэшем от имени объекта — **нет отдельного метадата-сервера, нет central lookup** (детерминированное размещение). Метаданные объекта лежат рядом с данными в `xl.meta`.
- **EC по умолчанию**: parity `EC:4` (для сета из 16: 12 data + 4 parity). Настраивается storage class'ом (`MINIO_STORAGE_CLASS_STANDARD=EC:4`).
- **Кворумы**: запись требует **write quorum** (data+parity шардов достаточно, чтобы пережить отказ: N/2+1 дисков сета при дефолтной parity), чтение — **read quorum** (обычно N/2). Потеряли больше parity дисков, чем M — объект недоступен на запись/чтение соответственно.
- **Bitrot protection**: каждый шард хэшируется (HighwayHash) при записи, проверяется при чтении; тихая порча данных детектится и чинится из parity.
- **Healing**: автоматический background-скраб + `mc admin heal`. При замене диска MinIO сам восстанавливает шарды на новый.
- **Мелкие объекты** (< ~128 KiB) пишутся inline прямо в `xl.meta` — меньше IOPS.
- **Требования к железу**: JBOD, **без RAID** (EC сам обеспечивает избыточность, RAID только съест ёмкость и скорость), XFS, одинаковые диски/серверы, локальные диски (не NAS/SAN).
- **Расширение**: добавлением **server pool** (нельзя добавить один диск/ноду в существующий сет); данные не ребалансируются автоматически — новые объекты пишутся в менее заполненный пул (есть `mc admin rebalance`).
- **Репликация между кластерами**: bucket replication / site replication — асинхронная, для DR и геораспределённости.
- Consistency: **strict read-after-write** (запись подтверждается после записи кворума).

### 2.5 Эксплуатация MinIO — что спрашивают практики

- **Один медленный диск тормозит весь erasure set** (запись ждёт кворум): ловить через `mc admin speedtest`, node_exporter (iostat await по дискам), менять диск.
- **Мониторинг**: `/minio/v2/metrics/cluster` (Prometheus), ключевые метрики: капасити, offline disks/nodes, heal backlog, API latency/errors (5xx, slow calls).
- **LIST — дорогая операция**; миллионы объектов в одном «каталоге»-префиксе — боль; проектировать префиксы.
- **Данные не теряются при**: смерти ≤M дисков сета / ноды (если parity позволяет). Вопрос «сколько дисков можно потерять?» — отвечать через parity: EC:4 → любые 4 диска каждого сета, на чтение — до 4, кластер жив при живом кворуме нод (N/2+1 для записи).
- Upgrade MinIO — rolling, безопасен (совместимость форматов), но обновлять весь кластер (одна версия).

### 2.6 Distributed systems теория (спросят обязательно)

- **CAP**: при сетевом разделении выбираешь Consistency или Availability. S3/MinIO — CP-уклон (кворумы). Не говори «CAP — выбери 2 из 3» — P не опция, партишены случаются.
- **Кворумы**: W + R > N даёт строгую согласованность чтений (классика Dynamo). W=N/2+1, R=N/2+1.
- **Consistency-модели**: strong / eventual / read-your-writes / monotonic reads. Уметь привести примеры (S3 — strong; DNS, bucket-репликация между сайтами — eventual).
- **Fault tolerance**: отказ != катастрофа; проектирование от failure domains (диск → нода → стойка → ДЦ), blast radius, graceful degradation (потеря parity → деградированное чтение, но живое).
- **Split brain** и зачем нечётные кворумы (etcd/ZooKeeper — 3/5 нод).

---

## Блок 3. Kubernetes

### 3.1 Архитектура (рассказать за 2 минуты без запинки)

**Control plane**: `kube-apiserver` (единственная точка входа, всё через него), `etcd` (хранилище состояния, Raft), `kube-scheduler` (выбирает ноду для пода), `kube-controller-manager` (циклы согласования: deployment→replicaset→pod, node lifecycle и т.д.). **На нодах**: `kubelet` (запускает поды через CRI — containerd), `kube-proxy` (правила iptables/IPVS для Services), CNI-плагин (сеть подов).

**Классика: «что происходит при `kubectl apply -f deployment.yaml`?»**
kubectl → apiserver (authn → authz/RBAC → admission webhooks) → объект в etcd → deployment-controller создаёт ReplicaSet → RS-controller создаёт Pod-объекты (Pending) → scheduler подбирает ноду (фильтры: ресурсы, taints, affinity; скоринг) и пишет nodeName → kubelet на ноде видит под, дёргает CRI (containerd) на pull образа и запуск контейнеров, CNI выдаёт IP, пробы стартуют → статус в apiserver → под Ready → попадает в Endpoints/EndpointSlice → kube-proxy обновляет правила.

### 3.2 Ресурсы, QoS, eviction

- **requests** — гарантия для шедулера (и вес CPU), **limits** — потолок (CPU → throttling через CFS quota; memory → OOMKill).
- **QoS-классы**: Guaranteed (requests==limits у всех контейнеров), Burstable, BestEffort — определяет порядок выселения при нехватке ресурсов на ноде.
- **OOMKilled (137)** — превысил memory limit (или нода в OOM). **Evicted** — kubelet выселил при node pressure (память/диск). Разные механизмы!
- **CPU throttling** — под тормозит при том, что CPU ноды свободен; смотреть `container_cpu_cfs_throttled_periods_total`; лечение — поднять limit или убрать CPU limit (частая практика: CPU limit не ставить, memory — обязательно).

### 3.3 Пробы

- **liveness** — «жив ли» → рестарт при провале. **readiness** — «готов ли принимать трафик» → выкидывание из Endpoints. **startup** — отключает liveness, пока приложение стартует.
- Классические фейлы: liveness ходит в зависимый сервис (падение зависимости → каскад рестартов всего кластера); слишком агрессивные таймауты → рестарт-штормы под нагрузкой.

### 3.4 Сеть

- Модель: у каждого пода свой IP, все поды видят друг друга без NAT. Реализация — CNI (Calico, Cilium, Flannel).
- **Service**: ClusterIP (virtual IP → правила kube-proxy: iptables DNAT или IPVS), NodePort, LoadBalancer, **headless** (`clusterIP: None` — DNS сразу отдаёт IP подов; так работают StatefulSet: `pod-0.svc`).
- **DNS**: CoreDNS. Классическая боль — `ndots:5`: короткие имена генерируют по 4–5 лишних запросов с суффиксами; при нагрузке — таймауты резолвинга. Лечение: FQDN с точкой, dnsConfig, NodeLocal DNSCache.
- **Ingress** / Gateway API — L7-маршрутизация; **NetworkPolicy** — файрвол на уровне подов (нужен поддерживающий CNI).

### 3.5 Storage в K8s

- PV / PVC / StorageClass (dynamic provisioning), CSI-драйверы. AccessModes: RWO/RWX/ROX.
- **StatefulSet**: стабильные имена (pod-0, pod-1), стабильные PVC (volumeClaimTemplates), упорядоченный rollout. Для Kafka/MinIO/БД.
- Для storage-нагрузок — **local PV** (диск ноды, привязка пода к ноде) vs сетевые тома: локальные быстрее, но нода умерла → данные ждут её (спасает репликация уровня приложения — Kafka/MinIO сами реплицируют).
- MinIO/Kafka в K8s обычно через **operator** (CRD + контроллер, автоматизирующий жизненный цикл: развёртывание, расширение, апгрейды).

### 3.6 Шедулинг

- nodeSelector / node affinity (required/preferred), pod affinity/anti-affinity (разнести реплики Kafka по нодам!), taints & tolerations (выделенные ноды под storage/GPU), topologySpreadConstraints (по зонам).
- PriorityClass + preemption; **PodDisruptionBudget** — сколько реплик можно уронить при drain (без PDB drain ноды может убить кворум).

### 3.7 Troubleshooting K8s (спросят кейсами)

- **Pending**: `kubectl describe pod` → Events. Причины: не хватает requests на нодах, taint без toleration, PVC не биндится, affinity невыполнима.
- **CrashLoopBackOff**: `kubectl logs --previous` (логи упавшего контейнера!), describe (exit code: 137 — OOM/сигнал, 1 — ошибка приложения), проверка команды/конфига/секретов.
- **ImagePullBackOff**: тег/регистри/imagePullSecrets/rate limit.
- **Под Ready, но трафика нет**: readiness, Endpoints (`kubectl get endpoints`), selector сервиса не совпадает с labels, NetworkPolicy.
- **Нода NotReady**: kubelet (journalctl), диск/память ноды (pressure), сеть до apiserver, CNI.
- **Всё медленно / API тупит**: etcd (latency fsync! etcd чувствителен к диску — `etcd_disk_wal_fsync_duration`), размер etcd, штормы контроллеров/операторов, throttling apiserver.
- `kubectl top`, events (`-A --sort-by=.lastTimestamp`), `kubectl debug node/`, ephemeral containers.

### 3.8 Эксплуатация кластера

- Апгрейд: control plane → ноды, по одной минорной версии, drain/cordon + PDB. Совместимость kubelet ↔ apiserver (n-2).
- Бэкап etcd (`etcdctl snapshot save`) — единственный настоящий бэкап кластера.
- RBAC: Role/ClusterRole + Binding, ServiceAccount. Секреты: base64 ≠ шифрование; encryption at rest / external secrets.
- GitOps: ArgoCD/Flux — желаемое состояние в git, кластер сходится к нему.

---

## Блок 4. Data Lake: Kafka, Spark, Airflow, Flink, Iceberg

### 4.1 Kafka (самый вероятный кандидат на глубокие вопросы)

**Архитектура**: брокеры; топик → **партиции** (единица параллелизма и порядка — порядок гарантирован только внутри партиции); у каждой партиции — leader и followers (**репликация**, обычно RF=3). Клиенты пишут/читают только через leader. Контроллер кластера: **KRaft** (Raft-кворум вместо ZooKeeper — знать, что ZK ушёл в прошлое).

**Гарантии записи**:
- `acks=0` (fire-and-forget), `acks=1` (leader записал), `acks=all` (все **ISR** записали).
- **ISR** (in-sync replicas) — реплики, не отстающие от лидера. `min.insync.replicas=2` + `acks=all` → запись подтверждается минимум двумя; если ISR сжался ниже — producer получает ошибку (доступность приносится в жертву согласованности).
- **unclean.leader.election=false** — не давать отставшей реплике становиться лидером (иначе потеря данных).
- Идемпотентный producer (дедупликация ретраев), transactions → exactly-once в пределах Kafka.

**Consumer**: consumer group — партиции распределяются между консьюмерами (консьюмеров > партиций → лишние простаивают); **rebalance** при изменении состава группы (стоп-мир у классического протокола; cooperative/incremental — мягче); offsets хранятся в топике `__consumer_offsets`; **lag** = разница между концом партиции и офсетом группы — главная метрика здоровья пайплайна.

**Хранение**: партиция = сегменты на диске, запись последовательная, чтение через **page cache + zero-copy (sendfile)** — поэтому Kafka так быстра и поэтому ей нужна свободная RAM под кэш, а не большой heap. Retention по времени/размеру; **log compaction** — хранится последнее значение по ключу (changelog-топики).

**Эксплуатация — типовые кейсы**:
- **Растёт lag**: консьюмер не успевает (масштабировать до числа партиций, искать медленную обработку), rebalance-шторм (частые перезапуски, таймауты `max.poll.interval.ms`), перекос по ключам в одну партицию.
- **Under-replicated partitions** > 0: брокер отстаёт/умер — смотреть диск/сеть/GC на брокере.
- **Диск заполняется**: retention, размер сегментов, компакция не успевает.
- Нельзя уменьшить число партиций; увеличение ломает key-ordering.

### 4.2 Spark

**Модель**: driver (план, DAG, координация) + executors (задачи). Ленивые transformations → job при action. Job → **stages** (границы по **shuffle**) → tasks (по партициям).
- **Narrow vs wide** трансформации: map/filter — narrow; groupBy/join — wide → **shuffle** (перетасовка данных по сети через диск) — главный источник тормозов.
- **Память executor'а**: heap делится на execution (шаффлы, агрегации) и storage (кэш), unified — перетекают; + overhead (off-heap). **OOM причины**: перекос данных (**skew** — одна задача-гигант; лечение: соль в ключах, AQE skew-join), огромные collect() на driver, недостаточный memoryOverhead (частая причина убийства подов в K8s).
- **Spark on K8s**: driver-под создаёт executor-поды; dynamic allocation, spot-ноды для executors. Смотреть Spark UI: медленные stages, spill (memory→disk), skew по задачам.
- **Small files problem**: тысячи мелких файлов в data lake → оверхед на листинги/таски; лечение — repartition/coalesce перед записью, компакция в Iceberg.

### 4.3 Airflow

- Компоненты: scheduler (парсит DAG'и, ставит таски), webserver, workers, metadata DB (Postgres). **Executors**: LocalExecutor, CeleryExecutor (пул воркеров + Redis/RabbitMQ), **KubernetesExecutor** (под на таск — изоляция, автоскейл, но старт медленнее).
- Здоровье: метрики scheduler loop / dag parsing time (тяжёлый код на верхнем уровне DAG-файла — боль), queued tasks, pool starvation, зомби-таски (воркер умер, heartbeat пропал).
- Практики: идемпотентность тасок + catchup/backfill, retries с exponential backoff, SLA/alerts, изоляция зависимостей (KubernetesPodOperator).

### 4.4 Flink

- Streaming с состоянием: JobManager (координатор) + TaskManagers (слоты). Отличие от Spark: настоящий поток, а не микробатчи.
- **Checkpoints** — консистентные снапшоты состояния (алгоритм Chandy–Lamport: **barriers** текут по потоку с данными); при падении — восстановление с последнего чекпоинта + перечитывание из Kafka → **exactly-once** state-семантика (end-to-end — с транзакционными sink'ами, two-phase commit).
- **Savepoints** — ручные снапшоты для апгрейдов/миграций джоба.
- **State backend**: HashMap (в памяти) vs **RocksDB** (на диске, для большого стейта; инкрементальные чекпоинты). Чекпоинты — в S3/MinIO (вот и связь с object storage!).
- **Backpressure**: sink/оператор не успевает → давление вверх по графу до источника; диагностика во Flink UI (busy/backpressured по операторам); лечение — параллелизм узкого места, ресурсы, устранение skew. Симптом: растёт checkpoint duration → чекпоинты по таймауту → падения.
- **Watermarks** — механизм event time: «данные до момента T приехали», управляет срабатыванием окон при опоздавших событиях.

### 4.5 Apache Iceberg

**Что это**: открытый **table format** поверх файлов (Parquet) в object storage — превращает набор файлов в таблицу с ACID.

**Слои метаданных** (уметь нарисовать): catalog (указатель на текущий metadata file; Hive Metastore / REST / Nessie / Glue) → **metadata.json** (схема, партиционирование, снапшоты) → **manifest list** (снапшот = список манифестов) → **manifests** (списки data-файлов со статистикой min/max по колонкам) → data files (Parquet).

**Что даёт**:
- **Атомарные коммиты**: новый снапшот публикуется атомарной подменой указателя в каталоге (optimistic concurrency) — читатели всегда видят консистентный снимок; нет «полузаписанных» данных.
- **Snapshots / time travel** (`VERSION AS OF`), rollback.
- **Schema evolution** без переписывания данных (колонки по ID, а не по имени/позиции).
- **Hidden partitioning**: партиции как трансформации (`days(ts)`, `bucket(16, id)`) — запросы не обязаны знать про партиционные колонки; partition evolution.
- **Нет directory listing** — план запроса строится по метаданным и статистикам (prune по min/max) → дружит с S3 (LIST дорогой).

**Эксплуатация** (то, что важно для этой вакансии):
- **Компакция**: `rewrite_data_files` — сшивание мелких файлов (стриминг из Flink генерирует их массово).
- **expire_snapshots** — чистка старых снапшотов и осиротевших файлов (`remove_orphan_files`); иначе storage пухнет бесконечно.
- rewrite_manifests при разросшихся метаданных.
- Конфликты параллельных коммитов (retry в optimistic concurrency), нагрузка на catalog.
- **Почему не Hive-таблицы**: Hive трекает данные на уровне каталогов (листинги, нет атомарности между партициями, переименования смертельны на S3) — Iceberg трекает на уровне файлов с ACID. Это вопрос-фаворит.

---

## Блок 5. Performance Troubleshooting (CPU, RAM, network, disk)

### 5.1 Методология — говори её первой

- **USE** (Brendan Gregg) для ресурсов: по каждому ресурсу проверь **Utilization, Saturation, Errors**.
- **RED** для сервисов: Rate, Errors, Duration.
- Чек-лист первых 60 секунд на хосте:

```
uptime                 # LA: тренд 1/5/15
dmesg -T | tail        # OOM, I/O errors, hung tasks
vmstat 1               # r (run queue), b (blocked=D), si/so (swap!), us/sy/wa
mpstat -P ALL 1        # перекос по ядрам (одно ядро 100% = однопоточный bottleneck/IRQ)
pidstat 1              # кто ест CPU по процессам
iostat -xz 1           # await, aqu-sz, %util по дискам
free -h                # available, swap
sar -n DEV 1           # трафик по интерфейсам, близко ли к line rate
sar -n TCP,ETCP 1      # retrans/s — ретрансмиты!
top / htop
```

### 5.2 CPU

- Разложение: us (приложение), sy (ядро — много sy? syscalls, сеть, спинлоки), **wa** (iowait), **st** (steal — у нас в виртуалке сосед отжирает/CPU overcommit гипервизора), si/hi (прерывания).
- Saturation: run queue (`vmstat` колонка r) > числа ядер; context switches (`cs`) аномально высокие.
- Глубже: `perf top`, `perf record -g` + **flamegraph** — где именно горит CPU. `strace -c` — профиль syscalls (но тормозит процесс!).
- В контейнерах: **CFS throttling** — под упирается в limit при свободной ноде (метрика cfs_throttled).

### 5.3 Память

- Смотреть **available** и **si/so в vmstat** (активный swap-in/out = беда), major page faults (`pidstat -r`).
- Кто съел: `ps aux --sort=-rss`, `smem`, `/proc/meminfo`; slab (`slabtop` — иногда ядро само, например dentry cache на миллионах файлов).
- OOM killer в dmesg. cgroup-лимиты: `memory.current` vs `memory.max`.
- Утечка vs кэш: RSS растёт монотонно без плато — утечка; page cache — reclaimable, не паниковать.

### 5.4 Диск / I/O

- `iostat -x 1`: **await** (средняя латенси I/O — главное число: HDD норм ~10ms, SSD ~1ms, NVMe <0.5ms), **aqu-sz** (очередь), r/s, w/s (IOPS), rkB/s wkB/s (throughput). **%util обманчив на NVMe** (параллелизм — 100% ≠ потолок).
- Кто грузит: `iotop`, `pidstat -d 1`.
- Латенси vs throughput: последовательный поток большими блоками — throughput (МБ/с); random мелкими — IOPS и латенси. fio для бейзлайна.
- fsync-чувствительные (etcd, БД, Kafka acks): их убивает латенси записи, а не пропускная.
- Диск умирает: рост await при том же трафике, SMART (`smartctl -a`), I/O errors в dmesg, у MinIO — один такой диск тащит вниз весь erasure set.

### 5.5 Сеть

- `ss -s` (сводка), `nstat`/`netstat -s`: **TCP retransmits** (потери → латенси-спайки: смотреть сеть/NIC/перегрузку), listen drops/overflow (переполнение accept-очереди — `somaxconn`, backlog приложения).
- Дропы на интерфейсе: `ip -s link` (RX dropped: ring buffer — `ethtool -g/-G`, softirq не успевает — смотреть `%si` в top, RSS/RPS).
- **conntrack table full** (`nf_conntrack: table full, dropping packet` в dmesg) — классика на нагруженных nat/k8s нодах; `net.netfilter.nf_conntrack_max`.
- Пропускная: `iperf3`; latency: `ping`/`mtr` (где по пути теряется); DNS-задержки — вечный подозреваемый (см. ndots в K8s).
- MTU mismatch (туннели/overlay CNI!): мелкие пакеты ходят, большие — нет; `ping -M do -s 1472`.
- `tcpdump`/`tshark` — последний довод: ретрансмиты, RST, zero window (приёмник не успевает читать).

### 5.6 Как отвечать на кейс «всё тормозит»

1. Определи симптом числом: латенси? throughput? ошибки? С какого момента, что менялось (деплой, рост трафика)?
2. Сузь слой: клиент → сеть → LB → приложение → зависимости (БД/S3/Kafka) → хост. Distributed tracing/метрики по каждому хопу.
3. На подозреваемом хосте — USE-проход (чек-лист выше).
4. Сформулируй гипотезу → проверь → почини → **проверь метрикой, что симптом ушёл**.

---

## Блок 6. Observability: Prometheus, Grafana, алертинг, логи

### 6.1 Prometheus

- **Pull-модель**: сам скрейпит `/metrics` по service discovery (в K8s — из API: ServiceMonitor у prometheus-operator). Плюс pull: контроль частоты, target down = сразу видно (`up == 0`). Для батчей/короткоживущих — Pushgateway (антипаттерн для всего остального).
- Типы метрик: **counter** (только растёт → `rate()`), **gauge**, **histogram** (бакеты + `histogram_quantile`), summary.
- TSDB: локальное хранение, retention; **HA/долгое хранение — Thanos / Mimir / VictoriaMetrics** (дедупликация двух прометеев, downsampling, S3 как бэкенд — снова object storage!).
- **Кардинальность — вопрос-фаворит**: каждая комбинация лейблов = отдельный time series; user_id/URL/pod-hash в лейблах → миллионы серий → Prometheus умирает по памяти. Лечить: не класть unbounded-значения в лейблы, `metric_relabel_configs`, лимиты; искать виновных через `topk(10, count by (__name__)({__name__=~".+"}))`.
- Recording rules — предвычисление тяжёлых запросов.

### 6.2 PromQL — минимум, который надо писать с листа

```promql
rate(http_requests_total{code=~"5.."}[5m])                # RPS ошибок
sum by (instance) (rate(node_cpu_seconds_total{mode!="idle"}[5m]))  # CPU usage
histogram_quantile(0.99, sum by (le) (rate(http_request_duration_seconds_bucket[5m])))  # p99
node_filesystem_avail_bytes / node_filesystem_size_bytes < 0.1      # диск <10%
predict_linear(node_filesystem_avail_bytes[6h], 4*3600) < 0         # кончится за 4ч
up == 0                                                              # таргет умер
```
- `rate` vs `irate`: сглаженная скорость за окно vs по двум последним точкам (алерты — только rate).
- Почему p99 нельзя усреднять по инстансам и почему квантиль — из бакетов гистограммы (сложить `rate` бакетов, потом quantile).

### 6.3 Алертинг — философия (это спрашивают у SRE обязательно)

- **Алертить по симптомам, не по причинам**: «SLO по латенси горит», «error rate > X» — пейджить; «CPU 90%» — не пейджить (может быть нормой). Cause-based — в тикеты/дашборды.
- Каждый алерт: **actionable** (есть runbook), срочный, требует человека. Иначе — alert fatigue, и настоящий пейдж утонет.
- **SLI/SLO/error budget**: SLI — измеримый показатель (доля успешных запросов), SLO — цель (99.9% за 30 дней), error budget — допустимые 0.1%; burn rate — скорость сжигания бюджета; **multi-window multi-burn-rate** алерты (быстрое окно ловит пожар, длинное — тление).
- **Alertmanager**: grouping (один пейдж вместо 100), inhibition (нода умерла → подавить алерты её подов), silences (плановые работы), routing по severity/team.

### 6.4 Логи и трейсы

- Централизация: **Loki** (индексирует только лейблы — дёшево; grep по содержимому) vs **ELK/OpenSearch** (полнотекстовый индекс — мощно и дорого). Сборщики: promtail/fluent-bit/vector.
- Structured logging (JSON) — фильтруемость. Обязательные поля: timestamp, level, service, trace_id.
- **Три столпа**: metrics (что и когда сломалось) → logs (почему) → traces (где в цепочке; OpenTelemetry, Jaeger/Tempo). Связка по trace_id/exemplars.

---

## Блок 7. Автоматизация: Ansible, Terraform, Bash, Python

### 7.1 Ansible

- Агентless (SSH), push. Плейбуки → роли (`tasks/handlers/templates/defaults/vars`), инвентори (+dynamic), group_vars.
- **Идемпотентность** — ключевое слово: модуль приводит к состоянию, `changed` только при реальном изменении. `shell/command` идемпотентность ломают — оборачивать `creates=`, `changed_when`, или брать нормальный модуль.
- **Handlers** — срабатывают по notify один раз в конце (рестарт сервиса после изменения конфига).
- `--check --diff` (dry-run), `--limit`, tags, `serial` (rolling по батчам нод!), `ansible-vault` для секретов, molecule для тестов ролей.
- Когда Ansible плох: непрерывное согласование состояния (он one-shot, дрейф между запусками не ловит), сложная логика (лучше Python).

### 7.2 Terraform

- Декларативный IaC: `plan` (diff желаемого и **state**) → `apply`. Провайдеры.
- **State — главная тема вопросов**: маппинг конфиг↔реальные ресурсы; хранить в **remote backend** (S3 + locking) — никогда в git; содержит секреты; конфликт параллельных apply решает lock.
- Дрейф (руками поменяли в консоли): `plan` покажет, `apply` вернёт. Существующий ресурс — `import`. Хирургия state: `state mv/rm` (переименование ресурса без пересоздания).
- Опасность: изменение, требующее replace (destroy+create) — читать plan глазами перед apply; `prevent_destroy`, `create_before_destroy`.
- Модули, workspaces/окружения, пиновка версий провайдеров, CI: plan в PR → ревью → apply из main.
- **Terraform vs Ansible**: провижининг инфраструктуры (создать VM/сеть/бакеты) vs конфигурация того, что внутри (пакеты, конфиги, сервисы). Обычно вместе.

### 7.3 Bash

- Строгий режим: `set -euo pipefail` (падать на ошибке, на неопределённой переменной, ловить ошибку в середине пайпа) + `trap cleanup EXIT`.
- Кавычки всегда: `"$var"`, `"$(cmd)"`; массивы `"${arr[@]}"`; `[[ ]]` вместо `[ ]`; shellcheck в CI.
- Правило: скрипт >100 строк / нужны структуры данных / парсинг JSON → переписывай на Python (или хотя бы `jq`).

### 7.4 Python для эксплуатации

- Стандарт: requests/httpx, boto3 (S3!), kubernetes client, subprocess, argparse/click, логирование.
- Спросить могут: как распараллелить обход 1000 хостов (ThreadPoolExecutor / asyncio), как безопасно хранить секреты (env/vault, не в коде), ретраи с backoff.

---

## Блок 8. SRE: инциденты, RCA, postmortem, reliability

### 8.1 Incident management

- **Severity-уровни** (например: SEV1 — платформа лежит/теряем данные, SEV2 — деградация ключевой функции, SEV3 — некритично) и что каждый триггерит (пейдж, war room, комms).
- **Роли**: Incident Commander (координирует, НЕ чинит руками), техлиды по направлениям, comms (статусы стейкхолдерам каждые N минут). Один человек не должен одновременно чинить и координировать.
- **Главный принцип: сначала mitigate, потом root cause.** Откат деплоя / переключение трафика / feature flag — вместо героической отладки в проде. Вопрос «выкатили релиз, всё упало, твои действия?» — правильный ответ начинается со слова «откатываю».
- Во время инцидента: писать таймлайн в канал (кто что сделал и когда — потом это основа postmortem), не делать необратимых действий без второй пары глаз.
- Incident vs problem management (ITIL): инцидент — восстановить сервис сейчас; проблема — устранить корневую причину класса инцидентов.

### 8.2 RCA и postmortem

- **Blameless** — фокус на системе, а не на виноватом: «почему система позволила ошибке человека дойти до прода», иначе люди начнут скрывать факты.
- Методы: таймлайн событий, **5 whys** (аккуратно — линеен), contributing factors (обычно причин несколько: триггер + латентные условия). Root cause ≠ «человек ошибся» — копать до процессов/защит.
- Структура postmortem: impact (кол-во пользователей/запросов/денег, длительность) → timeline (детект, эскалация, mitigation) → root cause + contributing factors → что сработало/не сработало (детект? алерты молчали?) → **action items** (конкретные, с владельцем и сроком, трекаются как обычные задачи; иначе постмортем — театр).
- Метрики: MTTD/MTTR, частота повторных инцидентов.

### 8.3 SRE-практики

- **Error budget** как механика баланса скорость/надёжность: бюджет сожжён → фичи стоп, работаем над reliability.
- **Toil** — ручная, повторяющаяся, автоматизируемая работа без долгосрочной ценности; SRE-цель — держать toil < 50%, остальное — инжиниринг.
- Runbooks у каждого алерта; game days / chaos engineering (проверять отказоустойчивость до того, как проверит жизнь); capacity planning (predict_linear, тренды).
- Изменения — главный источник инцидентов: canary/progressive rollout, автооткат по метрикам, freeze-окна.

---

## Блок 9. Поведенческие вопросы

Подготовь заранее **4 истории** по формату **STAR** (Situation → Task → Action → Result, результат — с цифрами):

1. **Крупный инцидент, который ты разрулил** (сюда же RCA: что было причиной, какие action items, что не повторилось).
2. **Автоматизация с измеримым эффектом** («ручной процесс на 2 часа в неделю → плейбук/скрипт, ошибок стало 0»).
3. **Твой факап** — обязательно будет. Формула: честно описать ошибку → что сделал сразу (mitigation, сообщил команде — не скрывал!) → какой системный вывод (защита, чтобы класс ошибок стал невозможен). Факап без вывода — красный флаг, «не факапил» — тоже.
4. **Конфликт/взаимодействие с другой командой** (для этой вакансии идеально — с data engineers: они хотят ресурсов/скорости, ты — стабильности; как договорились: квоты, SLO, приоритизация).

**Твои вопросы им** (иметь 3–4, показывают уровень): какой масштаб платформы (нод/петабайт/пайплайнов)? Что чаще всего болит — storage, Spark, Kafka? Как устроен on-call и постмортемы? Какая доля времени — инциденты vs проекты по автоматизации? Что для команды успех новичка за первые 3 месяца?

---

## Мини-план подготовки

1. Прочитай блоки 1, 2, 5 (Linux + storage + troubleshooting) — ядро этой вакансии.
2. Потом 3 и 4 (K8s + Data Lake) — здесь глубина по Kafka и Iceberg даёт больше всего очков.
3. 6–8 за один заход — там меньше зубрёжки, больше здравого смысла.
4. Напиши свои 4 STAR-истории (блок 9) — письменно, по 5 предложений.
5. Возвращайся — гоняю по вопросам в режиме собеса, блок за блоком.
