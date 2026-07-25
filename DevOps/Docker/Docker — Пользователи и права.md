---
tags:
  - docker
  - devops
  - security
  - linux
topic: docker
subtopic: безопасность-пользователи
status: готово
difficulty: продвинутый
created: 2026-06-24
aliases:
  - Docker Security
  - Docker Users
  - Docker Безопасность
---

# 🔐 Docker — Пользователи и права

> [!info] Навигация
> [[Docker MOC]] → [[Docker — Внутреннее устройство]] → **Пользователи и права** → [[Docker — Секреты]]

---

## Проблема: root в контейнере

По умолчанию процессы в контейнере запускаются от **root (UID 0)**:

```bash
docker run --rm alpine whoami   # → root
docker run --rm alpine id       # → uid=0(root) gid=0(root)
```

> [!danger] Почему это опасно
> 1. При успешной эксплуатации уязвимости (container escape) процесс может получить привилегии **за пределами контейнера**, а запуск контейнера от root нередко облегчает эксплуатацию подобных атак.
> 2. Файлы, создаваемые через bind mount, могут принадлежать root на хосте.
> 3. Нарушается принцип минимальных привилегий.
> 4. Многие исторические Docker/Kubernetes CVE становились значительно опаснее при запуске контейнера от root.

---

## USER в Dockerfile

```dockerfile
# ── Вариант 1: системный пользователь ─────────────────────────
FROM node:20-alpine

WORKDIR /app
COPY package*.json ./
RUN npm ci --omit=dev
COPY . .

# Создать группу и пользователя
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
# Отдать файлы новому владельцу
RUN chown -R appuser:appgroup /app

USER appuser          # все последующие команды и CMD — от этого пользователя
EXPOSE 3000
CMD ["node", "index.js"]
```

```dockerfile
# ── Вариант 2: числовой UID (без /etc/passwd) ─────────────────
FROM gcr.io/distroless/nodejs20-debian12
COPY --from=builder /app/dist /app
USER 1001             # distroless не имеет adduser, используем числовой UID
CMD ["/app/index.js"]
```

```dockerfile
# ── Вариант 3: Ubuntu/Debian стиль ────────────────────────────
RUN groupadd -r -g 1001 appgroup && \
    useradd -r -u 1001 -g appgroup -s /sbin/nologin -d /app appuser
USER appuser:appgroup
```

> [!info]
>Владельца копируемых файлов можно задать сразу:
>```dockerfile
>COPY --chown=appuser:appgroup . .
>```
>Это не создаёт отдельный слой с рекурсивным `chown` и обычно ускоряет сборку.

> [!info]
>`USER` влияет на последующие `RUN`, а также задаёт пользователя для `CMD` и
>`ENTRYPOINT`. `COPY` без `--chown` по умолчанию всё равно создаёт файлы с
>владельцем UID/GID `0`.

---

## Linux Capabilities

Root — это не монолитная привилегия, а набор **capabilities**:

| Capability          | Что позволяет                      | Нужна ли в типичном приложении |
| ------------------- | ---------------------------------- | ------------------------------ |
| `CHOWN`             | Менять владельца файлов            | Редко                          |
| `DAC_OVERRIDE`      | Обходить проверки прав на файлы    | Нет                            |
| `NET_BIND_SERVICE`  | Биндить порты < 1024               | Иногда                         |
| `NET_ADMIN`         | Конфигурировать сетевые интерфейсы | Нет                            |
| `SYS_ADMIN`         | Монтирование, ptrace, много чего   | **Почти никогда**              |
| `SYS_PTRACE`        | Трассировка процессов              | Нет                            |
| `SETUID` / `SETGID` | Менять UID/GID                     | Нет                            |

Docker по умолчанию предоставляет контейнеру ограниченный набор Linux Capabilities и не выдаёт многие потенциально опасные возможности (`SYS_ADMIN`, `SYS_MODULE`, `SYS_BOOT` и др.).

```bash
# Посмотреть capabilities процесса
cat /proc/1/status | grep Cap
capsh --decode=00000000a80425fb   # декодировать bitmap

# Запуск с минимальными capabilities (нет ничего лишнего)
docker run --cap-drop ALL --cap-add NET_BIND_SERVICE nginx

# Опасно — почти полностью снимает стандартные ограничения контейнера
docker run --privileged myapp

# Посмотреть дефолтные capability Docker
docker run --rm alpine cat /proc/1/status | grep CapBnd
```

```dockerfile
# В docker-compose
services:
  app:
    cap_drop:
      - ALL
    cap_add:
      - NET_BIND_SERVICE
      - CHOWN
```

>[!warning]
>`--privileged` практически отключает большинство механизмов изоляции Docker: контейнер получает почти все Linux Capabilities, доступ к устройствам и значительно более широкие права. Использовать следует только при крайней необходимости.
---

## Seccomp — фильтр системных вызовов

**Seccomp** (Secure Computing Mode) — профиль, определяющий, какие системные
вызовы разрешены, запрещены или приводят к ошибке.

Стандартный профиль Docker блокирует около 44 из 300+ системных вызовов. Точный
набор меняется вместе с Docker, ядром и `libseccomp`.

```bash
# Проверить профиль контейнера
docker inspect myapp | jq '.[0].HostConfig.SecurityOpt'

# Отключить seccomp (для отладки)
docker run --security-opt seccomp=unconfined myapp

# Использовать кастомный профиль
docker run --security-opt seccomp=my-profile.json myapp
```

> [!warning]
> Не составляй минимальный профиль простым перечислением «очевидных» syscall:
> реальные runtime и libc используют значительно больше вызовов, а набор зависит
> от архитектуры. Начинай со стандартного профиля Docker, фиксируй необходимые
> исключения и тестируй приложение. Отключение seccomp допустимо только как
> кратковременный диагностический шаг.

---

## AppArmor

MAC (Mandatory Access Control) — ограничивает доступ к файлам, сетям, capabilities.

```bash
# Проверить профиль
docker inspect myapp | jq '.[0].AppArmorProfile'
# → "docker-default"

# Отключить AppArmor
docker run --security-opt apparmor=unconfined myapp

# Посмотреть загруженные профили
sudo aa-status | grep docker-default
```

>[!info]
>AppArmor работает на уровне ядра Linux и ограничивает действия процесса даже при наличии обычных UNIX-прав.
---

## User Namespace Remapping

**Идея:** root (UID 0) внутри контейнера → непривилегированный UID на хосте.

```
Внутри контейнера: UID 0 (root)
         ↓  user namespace mapping
На хосте:          UID 100000  (обычный пользователь)
```

```bash
# Включить в /etc/docker/daemon.json
{
  "userns-remap": "default"
}
# → Docker создаст пользователя "dockremap"
# → /etc/subuid: dockremap:100000:65536
# → /etc/subgid: dockremap:100000:65536

sudo systemctl restart docker

# Проверить
docker run --rm alpine id
# uid=0(root) gid=0(root)   ← внутри контейнера

# На хосте процесс контейнера виден как:
ps aux | grep node
# dockremap  100000  ...    ← непривилегированный UID
```

>[!info]
>Внутри контейнера пользователь по-прежнему видится как root, однако ядро отображает его на непривилегированный UID хоста.

> [!warning] Ограничения userns-remap
> - Тома с конкретными UID могут потребовать `chown`
> - Некоторые образы (Docker-in-Docker) не работают
> - Отдельные директории хранилища образов для каждого user namespace

---

## Rootless Docker

Запуск **самого Docker daemon** без root-привилегий:

```bash
# Установка rootless Docker
dockerd-rootless-setuptool.sh install

# Запуск
export DOCKER_HOST=unix://$XDG_RUNTIME_DIR/docker.sock
docker run hello-world

# Путь хранилища: ~/.local/share/docker/

# Особенности:
# - для портов < 1024 может потребоваться настройка
#   net.ipv4.ip_unprivileged_port_start или RootlessKit
# - сеть реализуется через RootlessKit и может иметь другую производительность
# - host network, AppArmor и cgroup-лимиты имеют системные ограничения
```

>[!success]
>Даже если злоумышленник получит полный контроль над контейнером, сам Docker daemon не обладает root-привилегиями, что существенно снижает последствия компрометации.
---

## Read-only контейнер

```bash
# ФС контейнера read-only — защита от изменений
docker run --read-only myapp

# Но приложению нужны writable директории → tmpfs
docker run --read-only \
  --tmpfs /tmp \
  --tmpfs /var/run \
  --tmpfs /app/cache \
  myapp
```

```yaml
# docker-compose
services:
  app:
    read_only: true
    tmpfs:
      - /tmp
      - /var/run
```

---

## no-new-privileges

Запрещает процессам внутри контейнера повышать привилегии через `setuid` бинарники:

```bash
docker run --security-opt no-new-privileges myapp
```

```yaml
services:
  app:
    security_opt:
      - no-new-privileges:true
```

---

## Проверка безопасности: Docker Bench

```bash
docker run --rm -it \
  --net host --pid host --userns host --cap-add audit_control \
  -v /etc:/etc:ro \
  -v /usr/bin/containerd:/usr/bin/containerd:ro \
  -v /usr/bin/runc:/usr/bin/runc:ro \
  -v /usr/lib/systemd:/usr/lib/systemd:ro \
  -v /var/lib:/var/lib:ro \
  -v /var/run/docker.sock:/var/run/docker.sock:ro \
  docker/docker-bench-security
```

---

## Чеклист безопасности

```dataview
TABLE WITHOUT ID
  file.link AS "Тема",
  difficulty AS "Уровень"
FROM #docker AND #security
SORT file.name ASC
```

> [!tip] Минимальный чеклист
> - [ ] Запускать от не-root USER
> - [ ] `--cap-drop ALL --cap-add <только нужное>`
> - [ ] `--read-only` + tmpfs для writable dirs
> - [ ] `--security-opt no-new-privileges`
> - [ ] Использовать минимальный совместимый базовый образ (slim / chiseled / distroless / Alpine)
> - [ ] Регулярно сканировать образы (trivy, docker scout)
> - [ ] Не монтировать `/var/run/docker.sock` без необходимости
> - [ ] Включить `userns-remap` или rootless Docker
> - [ ] Устанавливать `--memory` и `--pids-limit`

---

## 📎 Связанные заметки

```dataview
LIST
FROM #docker
WHERE file.name != this.file.name
SORT file.name ASC
```
