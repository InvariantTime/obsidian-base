---
tags:
  - docker
  - devops
  - containers
topic: docker
subtopic: volumes
status: готово
difficulty: средний
created: 2026-06-24
aliases:
  - Docker Volumes
  - Docker Тома
  - Тома Docker
---

# 💾 Docker — Тома (Volumes)

> [!info] Навигация
> [[Docker MOC]] → [[Docker — Контейнеры]] → **Volumes** → [[Docker — Сеть]]

---

## Проблема

По умолчанию все данные внутри контейнера существуют только пока контейнер жив.  
При удалении контейнера (`docker rm`) — данные теряются.

**Решение** — монтирование томов / директорий хоста.

---

## Типы монтирования

![[excalidraw-docker-volume-base]]

---

## Сравнение типов

| | Named Volume | Bind Mount | tmpfs |
|-|-------------|------------|-------|
| Управление | Docker | ОС / пользователь | — |
| Расположение | `/var/lib/docker/volumes/` | Любая папка хоста | RAM |
| Портативность | ✅ Высокая | ❌ Зависит от хоста | ✅ |
| Производительность | ✅ Высокая | Зависит от ОС | ✅ Очень высокая |
| Резервное копирование | Простое | Стандартные инструменты | ❌ Только в памяти |
| Use-case | БД, приложения | Dev-режим, конфиги | Секреты, кэш |

---

## Named Volumes — команды

```bash
# Создание
docker volume create myvolume
docker volume create --driver local myvolume
docker volume create   --driver local   --opt type=nfs   --opt o=addr=192.168.1.100,rw   --opt device=:/path/to/dir   nfs-volume

# Просмотр
docker volume ls
docker volume ls --filter "dangling=true"   # неиспользуемые тома
docker volume inspect myvolume              # подробности

# Использование
docker run -v myvolume:/data nginx
docker run --mount type=volume,src=myvolume,dst=/data nginx

# Удаление
docker volume rm myvolume
docker volume prune              # удалить все неиспользуемые тома
```

---

## Bind Mounts

```bash
# Старый синтаксис (-v)
docker run -v /host/path:/container/path nginx
docker run -v $(pwd)/src:/app/src myapp         # текущая папка

# Новый синтаксис (--mount) — более явный
docker run --mount type=bind,source=/host/path,target=/app nginx
docker run --mount type=bind,src=$(pwd),dst=/app,readonly nginx

# Только для чтения
docker run -v /host/path:/container/path:ro nginx
```

> [!tip] Dev-режим
> Bind mount идеален для разработки — изменения в коде на хосте немедленно видны в контейнере.

---

## tmpfs

```bash
# Монтирование в RAM (данные не сохраняются)
docker run --tmpfs /tmp myapp
docker run --mount type=tmpfs,dst=/tmp,tmpfs-size=100m myapp
```

>[!info]
>Файлы хранятся только в оперативной памяти и полностью исчезают после остановки контейнера.
---

## Резервное копирование томов

```bash
# Создать бэкап тома myvolume в текущую папку
docker run --rm   -v myvolume:/source:ro   -v $(pwd):/backup   alpine tar czf /backup/myvolume.tar.gz -C /source .

# Восстановить
docker run --rm   -v myvolume:/target   -v $(pwd):/backup   alpine tar xzf /backup/myvolume.tar.gz -C /target
```

---

## Volumes в Docker Compose

```yaml
version: "3.9"

services:
  db:
    image: postgres:16
    volumes:
      - pgdata:/var/lib/postgresql/data   # named volume
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql  # bind mount

  app:
    image: myapp:latest
    volumes:
      - ./src:/app/src                    # bind mount (dev)
      - app-cache:/app/.cache             # named volume для кэша
      - type: tmpfs                       # tmpfs
        target: /tmp

volumes:
  pgdata:                     # Docker создаёт автоматически
  app-cache:
    driver: local
    driver_opts:
      type: none
      o: bind
      device: /custom/path
```

---

## Шаринг данных между контейнерами

```bash
# Создать volume с данными
docker run -d --name data-container -v shared-data:/data alpine

# Другие контейнеры подключаются к тому же тому
docker run --volumes-from data-container nginx
docker run --volumes-from data-container:ro nginx  # только чтение
```

---

## Расположение данных

```bash
# Где хранится том на хосте
docker volume inspect myvolume | jq '.[0].Mountpoint'
# /var/lib/docker/volumes/myvolume/_data

# Посмотреть содержимое тома без контейнера
sudo ls /var/lib/docker/volumes/myvolume/_data
```

---

## 📎 Связанные заметки

```dataview
LIST
FROM #docker
WHERE file.name != this.file.name
SORT file.name ASC
```
