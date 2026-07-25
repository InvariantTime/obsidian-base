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

# Manifest / OCI index без ручной работы с auth registry
docker buildx imagetools inspect nginx:latest --raw | jq .
```

---

## Linux Namespaces

Docker использует несколько Linux namespaces. Но не все из них обязательно
создаются для каждого контейнера: набор зависит от режима сети, userns/rootless,
версии Engine и параметров запуска.

| Namespace | Изолирует | Флаг | Обычно |
|-----------|----------|------|--------|
| **PID** | Дерево процессов | `CLONE_NEWPID` | отдельный |
| **NET** | Сетевой стек | `CLONE_NEWNET` | отдельный, кроме `--network host` |
| **MNT** | Точки монтирования | `CLONE_NEWNS` | отдельный |
| **UTS** | Hostname, domainname | `CLONE_NEWUTS` | отдельный |
| **IPC** | SysV IPC, POSIX MQ | `CLONE_NEWIPC` | отдельный |
| **USER** | UID/GID mapping | `CLONE_NEWUSER` | при rootless/userns-remap |
| **CGROUP** | Видимую иерархию cgroup | `CLONE_NEWCGROUP` | зависит от конфигурации |
| **TIME** | Некоторые системные часы | `CLONE_NEWTIME` | обычно не используется Docker по умолчанию |

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
| Поддержка | Legacy-системы | Стандарт для современных дистрибутивов |
| Rootless Docker | Ресурсные лимиты сильно ограничены | Лимиты доступны при корректной настройке systemd/controllers |

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

## Слоистая файловая система (OverlayFS)

`overlay2` — распространённый storage driver классического Docker image store.
В свежих установках Docker Engine 29 по умолчанию используется containerd image
store, который на Linux также обычно опирается на OverlayFS через snapshotter.
Выбранный backend можно увидеть в `docker info`.

Для запущенного контейнера OverlayFS объединяет несколько каталогов:

| Каталог | Назначение |
|---|---|
| `lowerdir` | read-only слои образа |
| `upperdir` | изменения writable layer контейнера |
| `workdir` | служебный каталог OverlayFS |
| `merged` | объединённое представление, которое видит контейнер |

Конкретная структура под `/var/lib/docker/overlay2` — внутренняя деталь
реализации и может отличаться при другом storage driver или containerd image store.
Не изменяй эти файлы вручную.

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
# /var/run/docker.sock — локальный Engine API; доступ контролируется правами socket
# Доступ к rootful socket обычно эквивалентен root-доступу на хосте

# Опасно:
docker run -v /var/run/docker.sock:/var/run/docker.sock alpine

# Изнутри такого контейнера:
curl --unix-socket /var/run/docker.sock http://localhost/containers/json
# → полный контроль над rootful Docker daemon
```

> [!danger]
> Rootless Docker уменьшает последствия компрометации daemon, но публикация даже
> rootless socket внутри недоверенного контейнера всё равно отдаёт ему полный
> контроль над ресурсами этого daemon. Предпочитай узкий API-прокси с allowlist
> операций или вообще не предоставляй socket.

---

## 📎 Связанные заметки

```dataview
LIST
FROM #docker
WHERE file.name != this.file.name
SORT file.name ASC
```
