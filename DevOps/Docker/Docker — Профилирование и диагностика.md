---
tags:
  - docker
  - devops
  - profiling
  - debugging
topic: docker
subtopic: профилирование
status: готово
difficulty: продвинутый
created: 2026-06-24
aliases:
  - Docker Profiling
  - Docker Debugging
  - Docker Диагностика
---

# 📊 Docker — Профилирование и диагностика

> [!info] Навигация
> [[Docker MOC]] → [[Docker — Контейнеры]] → **Профилирование** → [[Docker — Ресурсы и лимиты]]

---

## docker stats — подробно

```bash
docker stats                              # все контейнеры, real-time
docker stats myapp --no-stream            # разово (для скриптов, CI)
docker stats --format "{{json .}}"        # JSON для парсинга

# Кастомный формат
docker stats --format \
  "table {{.Container}}\t{{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}\t{{.MemPerc}}\t{{.NetIO}}\t{{.BlockIO}}\t{{.PIDs}}"
```

| Поле | Что означает | На что смотреть |
|------|-------------|----------------|
| `CPU %` | % от одного CPU-ядра (>100% = несколько ядер) | Стабильно высокий → утечка CPU |
| `MEM USAGE` | Память по данным cgroup (отображение cache зависит от ОС) | Постоянный рост → повод исследовать |
| `MEM LIMIT` | Установленный лимит (`--memory`) | Близко к лимиту → OOM-риск |
| `NET I/O` | Суммарный трафик с запуска | Неожиданный всплеск → проблема |
| `BLOCK I/O` | Суммарный disk I/O с запуска | Высокий → I/O bottleneck |
| `PIDs` | Количество процессов/потоков | Рост → утечка горутин/потоков |

---

## docker events — поток событий

```bash
# Real-time события
docker events

# Фильтрация
docker events --filter type=container
docker events --filter type=image
docker events --filter container=myapp
docker events --filter event=die       # только смерти контейнеров
docker events --filter event=oom       # OOM kill события!

# За период
docker events --since "2026-01-01T00:00:00" --until "2026-01-02T00:00:00"

# JSON формат для парсинга
docker events --format '{{json .}}' | jq 'select(.Action == "die")'
```

---

## Логи и драйверы

```bash
# Просмотр логов
docker logs myapp -f --tail 100 --timestamps

# Лог-драйвер контейнера
docker inspect myapp | jq '.[0].HostConfig.LogConfig'
```

`/etc/docker/daemon.json`:

```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "50m",
    "max-file": "3",
    "compress": "true"
  }
}
```

| Драйвер | Use-case | Особенности |
|---------|---------|------------|
| `json-file` | По умолчанию | Файл на диске, `docker logs` работает |
| `journald` | systemd-системы | Интеграция с journalctl |
| `fluentd` | EFK-стек | Отправка в Fluentd/Elasticsearch |
| `awslogs` | AWS | CloudWatch Logs |
| `splunk` | Enterprise | HTTP Event Collector |
| `none` | Без логов | Нет `docker logs` |
| `local` | Локальный, сжатый | Лучше json-file по производительности |

> [!warning]
> Для локальных логов Docker рекомендует драйвер `local`: он эффективнее и
> включает rotation по умолчанию. С remote-драйверами `docker logs` обычно
> работает через dual logging cache; если cache отключён, чтение может быть
> недоступно. Изменение daemon config действует только на **новые** контейнеры.

---

## nsenter — войти в namespace без docker exec

`nsenter` мощнее `docker exec` — работает даже если контейнер завис или нет shell:

```bash
# Получить PID контейнера
PID=$(docker inspect --format '{{.State.Pid}}' myapp)
echo "Container PID: $PID"

# Войти во все namespaces (как docker exec -it ... bash, но без docker)
sudo nsenter --target $PID --mount --uts --ipc --net --pid -- bash

# Только сетевой namespace (диагностика сети без shell в контейнере)
sudo nsenter --target $PID --net -- ss -tlnp
sudo nsenter --target $PID --net -- tcpdump -i eth0 -n port 5432

# Только mount namespace (увидеть ФС контейнера)
sudo nsenter --target $PID --mount -- ls /app

# Посмотреть открытые файлы процесса в контейнере
sudo nsenter --target $PID --mount --pid -- lsof -p 1

# strace внутри контейнера (нужен --cap-add SYS_PTRACE)
sudo nsenter --target $PID --pid -- strace -p 1 -f -e trace=network
```

---

## Диагностический контейнер (netshoot / toolbox)

```bash
# Запустить диагностический контейнер в СЕТИ другого контейнера
docker run --rm -it \
  --network container:myapp \
  nicolaka/netshoot \
  tcpdump -i eth0 -n 'port 5432'

# Диагностика с общим PID-namespace (видит процессы целевого контейнера)
docker run --rm -it \
  --pid container:myapp \
  --net container:myapp \
  nicolaka/netshoot bash

# Полезные инструменты в netshoot:
# tcpdump, ss, netstat, curl, dig, nslookup, iperf3
# strace, ltrace, lsof, htop, iotop
# iptables, iproute2, traceroute, mtr
```

---

## Memory: утечки и OOM

```bash
# OOM события ядра
dmesg | grep -i "out of memory"
dmesg | grep -i "oom_kill"
journalctl -k | grep -i oom

# cgroup memory stats (v2)
PID=$(docker inspect --format '{{.State.Pid}}' myapp)
CGPATH=$(cat /proc/$PID/cgroup | grep '0::' | cut -d: -f3)

cat /sys/fs/cgroup${CGPATH}/memory.current        # текущее использование
cat /sys/fs/cgroup${CGPATH}/memory.peak           # пик; доступно не на всех ядрах
cat /sys/fs/cgroup${CGPATH}/memory.stat           # детальная статистика
cat /sys/fs/cgroup${CGPATH}/memory.events         # oom events count

# Какие процессы потребляют память внутри контейнера
docker exec myapp ps aux --sort=-%mem | head -20

# smaps — детальное распределение памяти процесса (PID 1)
docker exec myapp cat /proc/1/smaps_rollup
```

```bash
# .NET специфика
docker exec myapp dotnet-counters monitor -p 1 \
  --counters System.Runtime[gc-heap-size,gen-0-gc-count,gen-1-gc-count,gen-2-gc-count]

# Дамп памяти .NET процесса
docker exec myapp dotnet-dump collect -p 1 -o /tmp/dump.dmp
docker cp myapp:/tmp/dump.dmp ./
dotnet-dump analyze dump.dmp
```

> [!note]
> `dotnet-counters`, `dotnet-dump` и `dotnet-trace` обычно отсутствуют в
> production runtime-образе. Подготовь отдельный debug target, используй
> `dotnet-monitor` или подключай диагностический контейнер; не устанавливай SDK
> и отладчики в финальный production-образ без необходимости.

---

## CPU: профилирование

```bash
# Загрузка CPU по cgroup
PID=$(docker inspect --format '{{.State.Pid}}' myapp)
CGPATH=$(cat /proc/$PID/cgroup | grep '0::' | cut -d: -f3)

watch -n1 cat /sys/fs/cgroup${CGPATH}/cpu.stat
# usage_usec — накопленное CPU-время в мкс

# perf на хосте для процесса контейнера
sudo perf top -p "$PID"

# .NET flamegraph
docker exec myapp dotnet-trace collect -p 1 \
  --duration 00:00:30 \
  --output /tmp/trace.nettrace
docker cp myapp:/tmp/trace.nettrace ./
# Открыть в speedscope.app или PerfView
```

---

## Disk I/O: анализ

```bash
# Block I/O cgroup stats
cat /sys/fs/cgroup${CGPATH}/io.stat

# iotop с host PID namespace (нужны повышенные права)
PID=$(docker inspect --format '{{.State.Pid}}' myapp)
docker run --rm -it --privileged --pid host nicolaka/netshoot iotop -p "$PID"

# Статистика I/O через docker stats
docker stats myapp --format "{{.BlockIO}}"
```

---

## Сеть: диагностика

```bash
# Анализ трафика между контейнерами
docker run --rm -it --net container:myapp nicolaka/netshoot \
  tcpdump -i any -n -A 'port 5432'

# Latency и потери пакетов
docker exec myapp ping -c 10 db

# DNS резолюция
docker exec myapp nslookup db
docker exec myapp dig db @127.0.0.11   # встроенный DNS Docker

# Открытые порты
docker exec myapp ss -tlnp
docker exec myapp netstat -tlnp

# HTTP запрос с временными метками
docker exec myapp curl -v -w "\n%{time_total}s\n" http://api:3000/health

# Bandwidth test между контейнерами
docker run -d --name iperf-server --network mynet nicolaka/netshoot iperf3 -s
docker run --rm --network mynet nicolaka/netshoot iperf3 -c iperf-server
```

---

## Healthcheck: мониторинг изнутри

```bash
# Состояние health
docker inspect myapp | jq '.[0].State.Health'
# → Status: starting | healthy | unhealthy
# → FailingStreak: N
# → Log: [последние 5 проверок]

docker events --filter event=health_status
```

---

## cAdvisor — метрики контейнеров

```bash
docker run -d \
  --name cadvisor \
  --volume /:/rootfs:ro \
  --volume /var/run:/var/run:ro \
  --volume /sys:/sys:ro \
  --volume /var/lib/docker/:/var/lib/docker:ro \
  --publish 8080:8080 \
  gcr.io/cadvisor/cadvisor:latest

# → http://localhost:8080  — UI с метриками
# → http://localhost:8080/metrics  — Prometheus endpoint
```

**Что экспортирует cAdvisor:**
- CPU: `container_cpu_usage_seconds_total`
- Memory: `container_memory_usage_bytes`, `container_memory_rss`
- Network: `container_network_receive_bytes_total`
- Filesystem: `container_fs_reads_bytes_total`

---

## Отладочная сборка образа

```dockerfile
# Многоэтапная сборка с debug-таргетом
FROM node:20-alpine AS base
WORKDIR /app
COPY package*.json ./
RUN npm ci

FROM base AS development
RUN npm install --save-dev @types/node ts-node nodemon
COPY . .
# Включить remote debugging
CMD ["node", "--inspect=0.0.0.0:9229", "-r", "ts-node/register", "src/index.ts"]

FROM base AS production
COPY . .
RUN npm run build
USER node
CMD ["node", "dist/index.js"]
```

```bash
# Запуск dev-версии с отладчиком
docker build --target development -t myapp:dev .
docker run -p 3000:3000 -p 9229:9229 \
  myapp:dev
# → подключить Chrome DevTools / VS Code debugger на порт 9229
```

---

## Полезные команды для отладки одной строкой

```bash
# Топ контейнеров по памяти
docker stats --no-stream --format "table {{.Name}}\t{{.MemPerc}}" \
  | sort -k2 -rn

# Все переменные окружения контейнера
docker exec myapp env | sort

# Эффективные переменные (из inspect)
docker inspect myapp | jq -r '.[0].Config.Env[]' | sort

# Проверить что контейнер не root
docker exec myapp id

# Список открытых файловых дескрипторов
docker exec myapp ls -la /proc/1/fd | wc -l

# Системные вызовы процесса (requires SYS_PTRACE)
docker exec myapp strace -p 1 -c -f 2>&1 | head -30

# Бинарный поиск — какой слой образа добавил большой файл
docker history --no-trunc myapp:latest
```

> [!note]
> Команды `ps`, `ss`, `strace`, `curl` и подобные могут отсутствовать в
> минимальном образе. Это не ошибка Docker — используй debug target, `nsenter`
> или отдельный диагностический контейнер.

---

## 📎 Связанные заметки

```dataview
LIST
FROM #docker
WHERE file.name != this.file.name
SORT file.name ASC
```
