---
tags:
  - docker
  - devops
  - performance
  - cgroups
topic: docker
subtopic: ресурсы
status: готово
difficulty: продвинутый
created: 2026-06-24
aliases:
  - Docker Resources
  - Docker Limits
  - Docker Лимиты
---

# ⚡ Docker — Ресурсы и лимиты

> [!info] Навигация
> [[Docker MOC]] → [[Docker — Внутреннее устройство]] → **Ресурсы и лимиты** → [[Docker — Профилирование и диагностика]]

---

## Зачем устанавливать лимиты

> [!danger] Без лимитов — один контейнер может убить весь хост
> Утечка памяти, бесконечный цикл, flood-атака — без ограничений они захватят все ресурсы
> и спровоцируют OOM killer или 100% CPU на хост-машине.

---

## CPU

### Основные флаги

```bash
# Количество CPU (самый понятный способ)
docker run --cpus=1.5 myapp       # до 1.5 ядра, может использовать любые

# Относительный вес (для конкурирующих контейнеров)
docker run --cpu-shares=512 myapp  # дефолт 1024; 512 = вдвое меньший приоритет
# Важно: --cpu-shares работает ТОЛЬКО при нехватке CPU на хосте

# Привязка к конкретным ядрам (CPU pinning)
docker run --cpuset-cpus="0,2" myapp    # только ядра 0 и 2
docker run --cpuset-cpus="0-3" myapp    # ядра 0, 1, 2, 3

# Низкоуровневое управление через period/quota
# cpus=X  ≡  cpu-quota = X * cpu-period
docker run --cpu-period=100000 --cpu-quota=50000 myapp   # 50% одного ядра
docker run --cpu-period=100000 --cpu-quota=150000 myapp  # 1.5 ядра
```

### Как работает `--cpus`

```
--cpus=1.5 при cpu-period=100000 (100ms):
└─ cpu-quota = 150000 мкс = 150ms
   └─ за каждые 100ms контейнер получает 150ms CPU-времени
      (на 2+ ядрах выполняется параллельно)
```

```bash
# Проверить настройки CPU контейнера
docker inspect myapp | jq '.[0].HostConfig | {
  NanoCpus,
  CpuShares,
  CpuPeriod,
  CpuQuota,
  CpusetCpus
}'

# NanoCpus = cpus * 1e9 (1.5 cpus = 1500000000)

# В cgroup (v2)
PID=$(docker inspect --format '{{.State.Pid}}' myapp)
CGPATH=$(cat /proc/$PID/cgroup | grep '0::' | cut -d: -f3)
cat /sys/fs/cgroup${CGPATH}/cpu.max
# → "150000 100000"  (quota period)
```

---

## Memory

### Основные флаги

```bash
# Жёсткий лимит RAM (OOM kill если превышен)
docker run --memory=512m myapp
docker run --memory=1g myapp

# Лимит RAM + SWAP (memory-swap включает RAM!)
docker run --memory=512m --memory-swap=1g myapp   # 512m RAM + 512m swap
docker run --memory=512m --memory-swap=-1 myapp   # безлимитный swap (не рекомендуется)
docker run --memory=512m --memory-swap=512m myapp # swap отключён

# Мягкий лимит (рекомендация ядру, не жёсткое ограничение)
docker run --memory-reservation=256m myapp

# Swappiness (0 = не использовать swap, 100 = агрессивно)
docker run --memory-swappiness=0 myapp

# Отключить OOM killer (опасно! может зависнуть система)
docker run --oom-kill-disable myapp

# Приоритет OOM killer (-1000 до 1000, меньше = реже убивают)
docker run --oom-score-adj=-500 myapp
```

```bash
# Текущее потребление памяти
PID=$(docker inspect --format '{{.State.Pid}}' myapp)
CGPATH=$(cat /proc/$PID/cgroup | grep '0::' | cut -d: -f3)

cat /sys/fs/cgroup${CGPATH}/memory.current   # текущий RSS
cat /sys/fs/cgroup${CGPATH}/memory.max       # лимит
cat /sys/fs/cgroup${CGPATH}/memory.peak      # максимум за время жизни
cat /sys/fs/cgroup${CGPATH}/memory.events    # oom count!

# Детальная статистика памяти
cat /sys/fs/cgroup${CGPATH}/memory.stat
```

### OOM Kill: диагностика

```bash
# Определить был ли контейнер убит OOM
docker inspect myapp | jq '.[0].State | {
  OOMKilled,
  ExitCode,
  Error
}'
# OOMKilled: true — был убит OOM killer

# События ядра
dmesg | grep -E "(oom|OOM|killed)" | tail -20
journalctl -k --since="1 hour ago" | grep -i oom

# Предотвращение: правильно рассчитывать лимиты
# Для .NET: GC heap + stack + native mem + overhead ≈ * 1.3 от heapSize
```

---

## PIDs — ограничение процессов

```bash
# Ограничить количество процессов (защита от fork bomb)
docker run --pids-limit=100 myapp
docker run --pids-limit=-1 myapp   # снять ограничение

# Просмотр
docker inspect myapp | jq '.[0].HostConfig.PidsLimit'
cat /sys/fs/cgroup${CGPATH}/pids.max
cat /sys/fs/cgroup${CGPATH}/pids.current

# Текущие процессы
docker exec myapp ps aux
docker top myapp
```

> [!tip]
> Fork bomb: `:(){ :|:& };:` — без `--pids-limit` убьёт весь хост.

---

## Block I/O

```bash
# Относительный вес I/O (10-1000, дефолт 500)
docker run --blkio-weight=300 myapp   # более низкий приоритет

# Лимит чтения/записи (bytes per second)
docker run \
  --device-read-bps=/dev/sda:10mb \
  --device-write-bps=/dev/sda:5mb \
  myapp

# Лимит операций в секунду (IOPS)
docker run \
  --device-read-iops=/dev/sda:1000 \
  --device-write-iops=/dev/sda:500 \
  myapp

# Статистика I/O в cgroup
cat /sys/fs/cgroup${CGPATH}/io.stat
```

---

## ulimits

Ограничения на уровне процесса (из `getrlimit`/`setrlimit`):

```bash
# Открытые файловые дескрипторы (soft:hard)
docker run --ulimit nofile=1024:2048 myapp

# Количество процессов
docker run --ulimit nproc=100:200 myapp

# Размер core dump
docker run --ulimit core=0 myapp     # отключить core dumps

# Максимальный размер файла
docker run --ulimit fsize=1073741824 myapp  # 1GB

# Глобальный дефолт в /etc/docker/daemon.json
{
  "default-ulimits": {
    "nofile": {
      "Name": "nofile",
      "Hard": 64000,
      "Soft": 64000
    }
  }
}
```

```bash
# Проверить ulimits внутри контейнера
docker exec myapp sh -c "ulimit -a"
```

---

## Шаблон: ресурсы в docker-compose

```yaml
version: "3.9"

services:
  app:
    image: myapp:latest
    deploy:
      resources:
        limits:
          cpus: "1.0"          # жёсткий лимит CPU
          memory: 512M         # жёсткий лимит RAM
        reservations:
          cpus: "0.25"         # гарантированный минимум CPU
          memory: 128M         # гарантированный минимум RAM

  db:
    image: postgres:16
    deploy:
      resources:
        limits:
          cpus: "2.0"
          memory: 1G
        reservations:
          cpus: "0.5"
          memory: 256M

  redis:
    image: redis:7-alpine
    deploy:
      resources:
        limits:
          cpus: "0.5"
          memory: 256M
```

> [!note] deploy.resources в Compose
> В standalone Compose (не Swarm) секция `deploy.resources` применяется с версии Compose v2.
> Для standalone docker compose используй:
> ```yaml
> mem_limit: 512m
> cpus: 1.0
> ```

---

## Мониторинг ресурсов: Prometheus + cAdvisor

```yaml
version: "3.9"
services:
  cadvisor:
    image: gcr.io/cadvisor/cadvisor:latest
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro
    ports:
      - "8080:8080"

  prometheus:
    image: prom/prometheus:latest
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
    ports:
      - "9090:9090"

  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
    environment:
      GF_SECURITY_ADMIN_PASSWORD: admin
```

```yaml
# prometheus.yml
scrape_configs:
  - job_name: cadvisor
    scrape_interval: 15s
    static_configs:
      - targets: ['cadvisor:8080']
```

**Ключевые метрики Prometheus:**

```promql
# CPU usage по контейнеру (% за 5 минут)
rate(container_cpu_usage_seconds_total{name!=""}[5m]) * 100

# Memory usage
container_memory_usage_bytes{name!=""}

# OOM events
container_oom_events_total

# Throttled CPU (% времени когда был ограничен)
rate(container_cpu_throttled_seconds_total[5m]) /
rate(container_cpu_usage_seconds_total[5m]) * 100
```

---

## Расчёт лимитов для .NET приложений

```bash
# .NET GC heap = DOTNET_GCHeapHardLimit или ~75% от --memory
# Примерный расчёт:
# memory = GC heap + threadpool stacks + native interop + overhead
# ≈ heapSizeMB * 1.3 + 50MB

# Полезные переменные окружения .NET для контейнеров
DOTNET_GCHeapHardLimit=402653184        # 384 MB жёсткий лимит heap
DOTNET_GCConserve=1                      # более агрессивная сборка мусора
DOTNET_GCHeapHardLimitPercent=75        # 75% от cgroup memory limit
DOTNET_RUNNING_IN_CONTAINER=true        # .NET сам читает лимиты cgroup!

# Начиная с .NET 3.0+ — автоматически читает cgroup limits
# Не нужно вручную задавать GCHeapHardLimit если задан --memory
```

---

## 📎 Связанные заметки

```dataview
LIST
FROM #docker
WHERE file.name != this.file.name
SORT file.name ASC
```
