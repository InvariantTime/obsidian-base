---
tags:
  - docker
  - devops
  - internals
  - linux
topic: docker
subtopic: внутреннее-устройство
status: готово
difficulty: продвинутый
created: 2026-06-24
aliases:
  - Docker Internals
  - Устройство Docker
---

# ⚙️ Docker — Внутреннее устройство

> [!info] Навигация
> [[Docker MOC]] → [[Docker — Основы]] → **Внутреннее устройство** → [[Docker — Пользователи и права]]

---

## Стек компонентов

```
┌─────────────────────────────────────────────────────┐
│  docker CLI  /  Docker Desktop  /  CI/CD            │
└──────────────────────────┬──────────────────────────┘
                           │ HTTP REST API
                    /var/run/docker.sock
┌──────────────────────────▼──────────────────────────┐
│              dockerd  (Docker Daemon)                │
│  • REST API server                                   │
│  • Image management (pull, build, tag)               │
│  • Volume / Network management                       │
│  • Прокси к containerd                               │
└──────────────────────────┬──────────────────────────┘
                           │ gRPC  /run/containerd/containerd.sock
┌──────────────────────────▼──────────────────────────┐
│                    containerd                        │
│  • Жизненный цикл контейнеров (create/start/stop)   │
│  • Snapshot manager (слои образов на диске)          │
│  • Content store  (blobs образов)                    │
│  • Events stream                                     │
└──────────────────────────┬──────────────────────────┘
                           │ по одному на контейнер
┌──────────────────────────▼──────────────────────────┐
│          containerd-shim-runc-v2 (shim)              │
│  • «Хранитель» контейнера                            │
│  • containerd может рестартовать — контейнер живёт   │
│  • Пересылает stdin/stdout                           │
│  • Следит за exit-кодом                              │
└──────────────────────────┬──────────────────────────┘
                           │ OCI Runtime Spec JSON
┌──────────────────────────▼──────────────────────────┐
│                  runc  (libcontainer)                │
│  • Создаёт namespaces, cgroups                       │
│  • Монтирует rootfs (overlay)                        │
│  • Запускает точку входа (exec)                      │
│  • Завершается сразу после старта                    │
└──────────────────────────┬──────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────┐
│            Container Process (PID 1)                 │
│              node / dotnet / nginx / ...             │
└─────────────────────────────────────────────────────┘
```

---

## OCI — Open Container Initiative

Три спецификации, на которых строится весь мир контейнеров:

| Спецификация | Что описывает | Реализации |
|-------------|--------------|-----------|
| **Image Spec** | Формат образа (manifest, config, layers) | Docker, Podman, Buildah |
| **Runtime Spec** | Как запускать контейнер (`config.json`) | runc, crun, gVisor, Kata |
| **Distribution Spec** | Протокол push/pull из registry | Docker Hub, GHCR, ECR |

```bash
# Посмотреть OCI config.json для запущенного контейнера
docker inspect myapp | jq '.[0]'

# Image manifest через API registry
curl -H "Accept: application/vnd.docker.distribution.manifest.v2+json" \
  https://registry-1.docker.io/v2/library/nginx/manifests/latest
```

---

## Linux Namespaces

Каждый контейнер получает **изолированный набор namespaces**:

| Namespace | Изолирует | Флаг | Системный вызов |
|-----------|----------|------|----------------|
| **PID** | Дерево процессов | `CLONE_NEWPID` | `getpid()` → видит своё |
| **NET** | Сетевой стек | `CLONE_NEWNET` | eth0, lo, routing table |
| **MNT** | Точки монтирования | `CLONE_NEWNS` | `/proc/mounts` |
| **UTS** | Hostname, domainname | `CLONE_NEWUTS` | `hostname` внутри ≠ снаружи |
| **IPC** | SysV IPC, POSIX MQ | `CLONE_NEWIPC` | очереди сообщений |
| **USER** | UID/GID маппинг | `CLONE_NEWUSER` | root внутри → UID 100000 снаружи |
| **CGROUP** | Корень иерархии cgroup | `CLONE_NEWCGROUP` | с Linux 4.6 |
| **TIME** | Часы (CLOCK_*) | `CLONE_NEWTIME` | с Linux 5.6 |

```bash
# Посмотреть namespaces контейнера
PID=$(docker inspect --format '{{.State.Pid}}' myapp)
ls -la /proc/$PID/ns/

# Все namespaces процесса
lsns -p $PID

# Войти в namespace контейнера вручную (мощнее docker exec)
sudo nsenter --target $PID \
  --mount --uts --ipc --net --pid \
  -- bash

# Войти только в network namespace
sudo nsenter --target $PID --net -- ss -tlnp
```

---

## Cgroups (Control Groups)

Ограничивают и измеряют потребление ресурсов.

### cgroups v1 vs v2

| | cgroups v1 | cgroups v2 |
|-|-----------|-----------|
| Иерархия | Несколько деревьев (по подсистеме) | Единое дерево |
| Путь | `/sys/fs/cgroup/memory/...` | `/sys/fs/cgroup/...` |
| Контроль CPU | `cpu` + `cpuacct` | `cpu` (объединены) |
| Поддержка | Везде | Linux ≥ 4.5, systemd ≥ 244 |
| Rootless Docker | Ограничено | Полная поддержка |

```bash
# Определить версию cgroups на хосте
stat -fc %T /sys/fs/cgroup   # tmpfs=v1, cgroup2fs=v2

# Путь к cgroup контейнера
PID=$(docker inspect --format '{{.State.Pid}}' myapp)
cat /proc/$PID/cgroup

# Лимит памяти контейнера (cgroups v2)
CGPATH=$(cat /proc/$PID/cgroup | grep '0::' | cut -d: -f3)
cat /sys/fs/cgroup${CGPATH}/memory.max

# Текущее использование памяти
cat /sys/fs/cgroup${CGPATH}/memory.current

# CPU stats
cat /sys/fs/cgroup${CGPATH}/cpu.stat
```

```bash
# Через docker
docker inspect myapp | jq '.[0].HostConfig | {
  Memory,
  CpuShares,
  NanoCpus,
  CpuPeriod,
  CpuQuota
}'
```

---

## Union Filesystem (overlay2)

**overlay2** — драйвер ФС по умолчанию. Реализует слоистую структуру образов.

```
/var/lib/docker/overlay2/<layer-id>/
├── diff/        ← что изменилось в этом слое
├── link         ← короткий id (симлинк)
├── lower        ← список нижележащих слоёв
├── merged/      ← смонтированный результат (контейнер видит это)
├── upper/       ← записываемый слой контейнера (copy-on-write)
└── work/        ← служебная директория overlayfs
```

```
┌─────────────────────────────────────────────┐
│          MERGED (что видит процесс)         │
│  upper + lower1 + lower2 + lower3           │
├─────────────────────────────────────────────┤
│  UPPER (writable, container layer)          │  ← docker commit → новый слой
├─────────────────────────────────────────────┤
│  LOWER 3: COPY src/ /app/  (read-only)      │
├─────────────────────────────────────────────┤
│  LOWER 2: RUN npm ci      (read-only)       │
├─────────────────────────────────────────────┤
│  LOWER 1: FROM node:20-alpine (read-only)   │
└─────────────────────────────────────────────┘
```

**Copy-on-Write (CoW):** при первой записи в файл из нижнего слоя — файл копируется в `upper/`, изменения применяются там.

```bash
# Посмотреть слои образа на диске
docker inspect nginx:latest | jq '.[0].GraphDriver.Data'

# Реальный размер на диске (с учётом шаринга слоёв)
docker system df -v

# Diff контейнера (что изменилось в upper)
docker diff mycontainer
```

---

## Как работает `docker run` (шаг за шагом)

```
1. docker CLI отправляет POST /containers/create → dockerd
   └─ Body: image, cmd, env, mounts, network config...

2. dockerd проверяет наличие образа локально
   └─ если нет → pull из registry (слой за слоем)

3. dockerd вызывает containerd: create container
   └─ containerd создаёт snapshot (overlay2 layers)
   └─ генерирует OCI bundle (rootfs + config.json)

4. containerd запускает containerd-shim-runc-v2
   └─ shim вызывает runc run

5. runc:
   └─ создаёт namespaces (unshare)
   └─ настраивает cgroups (limits)
   └─ монтирует rootfs (overlayfs)
   └─ монтирует /proc, /sys, /dev
   └─ устанавливает capabilities, seccomp
   └─ exec → PID 1 (точка входа)
   └─ завершается (runc больше не нужен)

6. shim остаётся «хранителем»:
   └─ пересылает stdin/stdout через FIFO
   └─ фиксирует exit code при завершении
   └─ сообщает containerd о смерти контейнера

7. containerd уведомляет dockerd
   └─ dockerd обновляет статус контейнера
```

---

## Сеть изнутри

```bash
# veth pair: один конец в контейнере, другой на хосте
ip link show | grep veth   # на хосте

# Какой veth соответствует контейнеру
PID=$(docker inspect --format '{{.State.Pid}}' myapp)
# Внутри контейнера:
sudo nsenter --target $PID --net -- ip link
# Смотрим ifindex, находим пару на хосте

# NAT правила (проброс портов)
sudo iptables -t nat -L -n --line-numbers | grep DOCKER

# Маршруты трафика в контейнер
sudo iptables -L DOCKER -n

# docker0 bridge
ip addr show docker0
bridge link
```

---

## docker.sock — поверхность атаки

```bash
# /var/run/docker.sock — это REST API без аутентификации
# Монтирование в контейнер = root на хост-машине

# Опасно:
docker run -v /var/run/docker.sock:/var/run/docker.sock alpine

# Изнутри такого контейнера:
curl --unix-socket /var/run/docker.sock http://localhost/containers/json
# → полный доступ ко всем контейнерам, созданию новых с privileged флагом

# Безопасная альтернатива: rootless Docker или Podman
```

---

## 📎 Связанные заметки

```dataview
LIST
FROM #docker
WHERE file.name != this.file.name
SORT file.name ASC
```
