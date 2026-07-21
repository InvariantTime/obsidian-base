---
tags:
  - docker
  - devops
  - containers
topic: docker
subtopic: контейнеры
status: готово
difficulty: начинающий
created: 2026-06-24
aliases:
  - Docker Containers
  - Контейнеры Docker
---

# 📦 Docker — Контейнеры

> [!info] Навигация
> [[Docker MOC]] → [[Docker — Образы]] → **Контейнеры** → [[Docker — Тома (Volumes)]]

---

## docker run — запуск контейнера

docker run автоматически делает pull из docker hub при отсутствии образа в локальном хранилище. Далее создаётся и запускается контейнер.

```bash
docker run [OPTIONS] IMAGE [COMMAND] [ARGS]
```

### Основные флаги

```bash
# Режимы запуска
docker run nginx                          # foreground (блокирует терминал)
docker run -d nginx                       # detach — в фоне
docker run -it ubuntu bash                # interactive + tty (для интерактивных сессий)
docker run --rm nginx                     # удалить контейнер после остановки

# Именование
docker run --name my-nginx nginx

# Порты
docker run -p 8080:80 nginx               # хост:контейнер
docker run -p 127.0.0.1:8080:80 nginx    # только на localhost
docker run -P nginx                       # автоматический проброс всех EXPOSE портов

# Переменные окружения
docker run -e NODE_ENV=production myapp
docker run --env-file .env myapp

# Тома
docker run -v /host/path:/container/path nginx      # bind mount
docker run -v myvolume:/data nginx                  # named volume
docker run --mount type=bind,src=/host,dst=/app nginx

# Ресурсы
docker run --memory=512m --cpus=1.5 myapp
docker run --memory-swap=1g myapp

# Сеть
docker run --network mynet myapp
docker run --network host myapp           # использовать сеть хоста
docker run --network none myapp           # без сети

# Пользователь
docker run --user 1001:1001 myapp
docker run -u node myapp

# Restart policy
docker run --restart always nginx
docker run --restart on-failure:3 myapp   # перезапуск при ошибке, макс 3 раза
docker run --restart unless-stopped nginx

# Прочее
docker run --hostname mycontainer nginx
docker run --add-host db:192.168.1.100 myapp   # /etc/hosts запись
docker run --read-only myapp                   # read-only ФС
docker run --privileged myapp                  # расширенные права (опасно!)
```

---

## Управление жизненным циклом

```bash
# Просмотр
docker ps                    # запущенные контейнеры
docker ps -a                 # все (включая остановленные)
docker ps -q                 # только ID
docker ps -s                 # показать размер
docker ps --filter "status=running"
docker ps --filter "name=nginx"
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"

# Запуск / остановка
docker start my-nginx        # запустить остановленный
docker stop my-nginx         # SIGTERM → ждёт → SIGKILL (10s таймаут)
docker stop -t 30 my-nginx   # кастомный таймаут
docker kill my-nginx         # немедленный SIGKILL
docker kill -s SIGINT my-nginx  # произвольный сигнал
docker restart my-nginx      # stop + start

# Пауза
docker pause my-nginx        # заморозить (останавливает все процессы, но не удаляет их)
docker unpause my-nginx      # разморозить

# Удаление
docker rm my-nginx           # удалить остановленный
docker rm -f my-nginx        # принудительно (даже запущенный)
docker rm $(docker ps -aq)   # удалить все остановленные
docker container prune       # удалить все остановленные
```

---

## Выполнение команд в контейнере

```bash
docker exec my-nginx ls /etc/nginx
docker exec -it my-nginx bash          # интерактивная оболочка
docker exec -it my-nginx sh            # если bash недоступен (alpine)
docker exec -u root my-nginx whoami    # от другого пользователя
docker exec -e VAR=value my-nginx env  # с переменной окружения
docker exec -w /tmp my-nginx pwd       # в другой рабочей директории
```

>[!info]
>`docker exec` запускает новый процесс внутри контейнера

---

## Логи

```bash
docker logs my-nginx                   # все логи
docker logs -f my-nginx                # следить в реальном времени
docker logs --tail 100 my-nginx        # последние 100 строк
docker logs --since 2h my-nginx        # за последние 2 часа
docker logs --since "2024-01-01" my-nginx
docker logs --timestamps my-nginx      # с временными метками
docker logs 2>&1 | grep ERROR my-nginx # stderr тоже
```

---

## Инспекция и мониторинг

```bash
# Детальная информация
docker inspect my-nginx                          # полный JSON
docker inspect -f '{{.NetworkSettings.IPAddress}}' my-nginx  # IP адрес
docker inspect -f '{{.State.Status}}' my-nginx   # статус
docker inspect -f '{{json .HostConfig.PortBindings}}' my-nginx

# Статистика ресурсов
docker stats                           # все контейнеры, real-time
docker stats my-nginx                  # конкретный контейнер
docker stats --no-stream               # разово (для скриптов)
docker stats --format "table {{.Container}}\t{{.CPUPerc}}\t{{.MemUsage}}"

# Процессы внутри контейнера
docker top my-nginx

# Изменения в ФС
docker diff my-nginx                   # A=added, C=changed, D=deleted

# Порты
docker port my-nginx
docker port my-nginx 80
```

> [!info]
> `docker inspect` возвращает JSON с полной конфигурацией контейнера:
>
> - сеть
> - тома
> - переменные окружения
> - ресурсы
> - точки монтирования
> - статус
---

## Копирование файлов

```bash
docker cp my-nginx:/etc/nginx/nginx.conf ./nginx.conf   # из контейнера
docker cp ./index.html my-nginx:/usr/share/nginx/html/  # в контейнер
```

---

## Attach и взаимодействие

```bash
docker attach my-nginx          # подключиться к stdin/stdout контейнера
                                # ⚠️ Ctrl+C остановит контейнер!
                                # Используй Ctrl+P, Ctrl+Q для detach

docker wait my-nginx            # ждать завершения контейнера
```

---

## Сохранение состояния контейнера в образ

```bash
docker commit my-nginx my-custom-nginx:v1
docker commit -m "added config" -a "author" my-nginx custom:latest
```

> [!warning]
> `docker commit` — антипаттерн для production. Лучше всегда использовать [[Docker — Dockerfile]].

---

## Restart Policies

`docker run --restart unless-stopped`

| Политика         | Поведение                                     |
| ---------------- | --------------------------------------------- |
| `no`             | Не перезапускать (по умолчанию)               |
| `on-failure[:N]` | При ненулевом exit code, макс N попыток       |
| `always`         | Всегда, включая после `docker restart` daemon |
| `unless-stopped` | Всегда, кроме явной остановки пользователем   |

---

## 📎 Связанные заметки

```dataview
LIST
FROM #docker
WHERE file.name != this.file.name
SORT file.name ASC
```
