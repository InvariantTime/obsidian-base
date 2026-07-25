---
tags:
  - docker
  - devops
  - containers
  - networking
topic: docker
subtopic: сеть
status: готово
difficulty: средний
created: 2026-06-24
aliases:
  - Docker Network
  - Docker Networking
  - Сеть Docker
---

# 🌐 Docker — Сеть

> [!info] Навигация
> [[Docker MOC]] → [[Docker — Тома (Volumes)]] → **Сеть** → [[Docker — Compose]]

---
## Что такое сеть Docker?

Сеть Docker — это механизм, который позволяет:

- контейнерам обмениваться данными друг с другом;
- контейнерам взаимодействовать с хостом;
- публиковать сервисы наружу;
- изолировать группы контейнеров.

Каждый контейнер получает собственный сетевой namespace, поэтому имеет собственные:

- IP-адрес;
- сетевые интерфейсы;
- таблицу маршрутизации;
- DNS-настройки.
## Сетевые драйверы

| Драйвер     | Описание                                                                       | Use-case                                            |
| ----------- | ----------------------------------------------------------------------------- | --------------------------------------------------- |
| **bridge**  | Виртуальный L2-мост на одном Docker-хосте; драйвер по умолчанию                | Обычные приложения на одном хосте                   |
| **host**    | Контейнер использует сетевой namespace хоста                                   | Нет сетевой изоляции; иногда нужна минимальная накладная стоимость |
| **none**    | Контейнер получает только loopback-интерфейс                                   | Максимальная сетевая изоляция                       |
| **overlay** | Объединяет контейнеры на разных Docker-хостах в логическую сеть                | Docker Swarm, мультихостовые сети                   |
| **macvlan** | Контейнер выглядит в физической сети как отдельное устройство со своим MAC     | Legacy-системы и прямое включение в L2-сеть          |
| **ipvlan**  | Похож на macvlan, но контейнеры используют MAC родительского интерфейса        | Сети с ограничением числа MAC-адресов               |

---

## bridge — по умолчанию

```mermaid
flowchart LR
    Client["Клиент"] -->|"host:8080"| Host["Порт хоста"]
    Host -->|"publish -p 8080:80"| Web["web:80"]
    Web -->|"DNS: db:5432"| DB["db:5432"]
```

Публикация (`-p`) нужна для входа **с хоста или извне**. Контейнеры в одной
пользовательской сети общаются напрямую по имени и порту контейнера; публиковать
порт БД для этого не требуется.

> [!warning] Дефолтная bridge-сеть
> В дефолтной сети (`bridge`) контейнеры не видят друг друга по имени — только по IP.
> Создавай **custom bridge** сети — там работает DNS-резолюция по имени контейнера.

---

## Команды управления сетями

```bash
# Просмотр
docker network ls                          # список сетей
docker network inspect bridge             # детали сети
docker network inspect mynet --format '{{json .Containers}}'

# Создание
docker network create mynet                                  # bridge по умолчанию
docker network create --driver bridge mynet
docker network create --driver bridge   --subnet 172.20.0.0/16   --ip-range 172.20.1.0/24   --gateway 172.20.0.1   mynet

docker network create --driver overlay myswarmnet             # для Swarm

# Удаление
docker network rm mynet
docker network prune                       # удалить все неиспользуемые

# Подключение контейнеров
docker network connect mynet my-nginx      # подключить работающий контейнер
docker network disconnect mynet my-nginx   # отключить
docker network connect --ip 172.20.1.5 mynet my-nginx  # фиксированный IP
```

---

## Запуск контейнера в сети

```bash
docker run --network mynet --name my-nginx nginx
docker run --network host nginx                    # сеть хоста
docker run --network none myapp                    # без сети

# Несколько сетей
docker run --network net1 myapp
docker network connect net2 myapp   # добавить вторую сеть
```

> [!note]
> `--network host` наиболее прямолинеен на Linux. В Docker Desktop поведение и
> доступность режима зависят от версии и настроек. В host-режиме флаг `-p`
> не нужен: процесс слушает порт непосредственно в сети хоста.

---

## DNS в пользовательских сетях

В custom bridge сети встроен DNS-сервер:

```bash
# Контейнеры в одной пользовательской сети видят друг друга по имени!
docker network create mynet
docker run -d --name db --network mynet postgres
docker run -d --name app --network mynet myapp

# Из контейнера app можно обращаться к db:
# ping db → работает
# psql -h db → работает
```

```bash
# Псевдонимы для DNS
docker run -d --name postgres --network mynet \
  --network-alias database -e POSTGRES_PASSWORD=secret postgres:16
# контейнер доступен как "postgres" и по alias "database"
```

---

## Проброс портов

```bash
# -p хост_порт:контейнер_порт
docker run -p 8080:80 nginx              # 0.0.0.0:8080 → контейнер:80
docker run -p 127.0.0.1:8080:80 nginx   # только localhost
docker run -p 8080:80/udp nginx         # UDP

# Диапазон портов
docker run -p 8000-8010:8000-8010 myapp

# Автоматический проброс всех EXPOSE портов
docker run -P nginx                      # случайные порты на хосте

# Посмотреть пробросы
docker port my-nginx
docker port my-nginx 80
```

> [!warning]
> Если IP хоста не указан, опубликованный порт обычно слушает на всех интерфейсах
> (`0.0.0.0` и, в зависимости от конфигурации, IPv6). Для локальной разработки
> безопаснее явно писать `127.0.0.1:8080:80`.

---

## Коммуникация контейнер ↔ хост

```bash
# Из контейнера обратиться к хосту
# Linux:
host.docker.internal    # в Docker Engine на Linux может потребовать настройку
172.17.0.1             # частый, но не гарантированный адрес default bridge

# Docker Desktop (Mac/Windows):
host.docker.internal    # всегда работает

# Добавить вручную
docker run --add-host host.docker.internal:host-gateway myapp
```

>[!info]
>В обычном Docker Engine на Linux чаще используют `--add-host host.docker.internal:host-gateway` или IP шлюза bridge-сети (например, `172.17.0.1`).
>IP лучше получать из конфигурации сети, а не зашивать в приложение.
---

## Примеры типичных сценариев

### Приложение + БД

```bash
docker network create app-net

docker run -d \
  --name postgres \
  --network app-net \
  -e POSTGRES_PASSWORD=secret \
  -e POSTGRES_DB=mydb \
  postgres:16

docker run -d   --name myapp   --network app-net   -e DATABASE_URL="postgresql://postgres:secret@postgres:5432/mydb"   -p 3000:3000   myapp:latest

# myapp обращается к postgres по имени "postgres"
```

### Изоляция: фронтенд + бэкенд + БД

```bash
docker network create frontend-net
docker network create backend-net

# nginx — только в frontend-net
docker run -d --name nginx --network frontend-net -p 80:80 nginx

# app — в обоих сетях (мост между ними)
docker run -d --name app --network frontend-net myapp
docker network connect backend-net app

# db — только в backend-net (недоступна снаружи)
docker run -d --name db --network backend-net \
  -e POSTGRES_PASSWORD=secret postgres:16
```

---

## Диагностика сети

```bash
# Inspect сети
docker network inspect mynet

# IP контейнера
docker inspect -f '{{.NetworkSettings.IPAddress}}' my-nginx
docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' my-nginx

# Тест из контейнера
docker exec my-nginx ping db
docker exec my-nginx curl http://api:3000/health
docker exec my-nginx nslookup db

# Временный контейнер для диагностики
docker run --rm --network mynet nicolaka/netshoot ping db
docker run --rm --network mynet alpine sh -c "apk add curl && curl http://api"
```

---

## 📎 Связанные заметки

```dataview
LIST
FROM #docker
WHERE file.name != this.file.name
SORT file.name ASC
```
