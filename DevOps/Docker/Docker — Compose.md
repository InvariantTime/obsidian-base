---
tags:
  - docker
  - devops
  - containers
  - compose
topic: docker
subtopic: compose
status: готово
difficulty: средний
created: 2026-06-24
aliases:
  - Docker Compose
  - docker-compose
---

# 🚀 Docker — Compose

> [!info] Навигация
> [[Docker MOC]] → [[Docker — Сеть]] → **Compose**

---

## Что такое Docker Compose?

**Docker Compose** — инструмент для определения и запуска многоконтейнерных приложений через YAML-файл.

```bash
# Вместо нескольких docker run команд:
docker compose up -d     # поднять всё приложение
docker compose down      # остановить и удалить
```

>[!info]
>Docker Compose позволяет описать всё приложение как код.
>Вместо:
`docker run`
`docker network create`
`docker volume create`
мы описываем приложение в одном `compose.yaml` и запускаем его одной командой:
`docker compose up`
## Структура `compose.yaml`

```yaml
services:                   # контейнеры
  web:
    ...
  db:
    ...

volumes:                    # именованные тома
  pgdata:

networks:                   # сети
  backend:
```

>[!info]  
>Поле верхнего уровня `version` устарело и в новых файлах не требуется. Современный
>Docker Compose всегда использует актуальную Compose Specification; если указать
>`version`, Compose может показать предупреждение.

## Полный пример: веб-приложение

```yaml
services:

  # ── Nginx (реверс-прокси) ───────────────────────────────────
  nginx:
    image: nginx:1.25-alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./nginx/ssl:/etc/nginx/ssl:ro
    depends_on:
      app:
        condition: service_healthy
    networks:
      - frontend
    restart: unless-stopped

  # ── Приложение ──────────────────────────────────────────────
  app:
    build:
      context: .
      dockerfile: Dockerfile
      target: production
      args:
        - NODE_ENV=production
    image: myapp:latest
    environment:
      - NODE_ENV=production
      - DATABASE_URL=postgresql://user:pass@db:5432/mydb
      - REDIS_URL=redis://redis:6379
    env_file:
      - .env.production
    expose:
      - "3000"              # доступен другим сервисам сети, но не публикуется на хост
    volumes:
      - uploads:/app/uploads
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_started
    healthcheck:
      test:
        - CMD
        - node
        - -e
        - "fetch('http://localhost:3000/health').then(r => process.exit(r.ok ? 0 : 1)).catch(() => process.exit(1))"
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
    networks:
      - frontend
      - backend
    restart: unless-stopped
    deploy:
      resources:
        limits:
          cpus: "1.0"
          memory: 512M

  # ── PostgreSQL ──────────────────────────────────────────────
  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass
      POSTGRES_DB: mydb
    volumes:
      - pgdata:/var/lib/postgresql/data
      - ./db/init.sql:/docker-entrypoint-initdb.d/01-init.sql:ro
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U user -d mydb"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - backend
    restart: unless-stopped

  # ── Redis ────────────────────────────────────────────────────
  redis:
    image: redis:7-alpine
    command: redis-server --appendonly yes
    volumes:
      - redis-data:/data
    networks:
      - backend
    restart: unless-stopped

volumes:
  pgdata:
  redis-data:
  uploads:

networks:
  frontend:
    driver: bridge
  backend:
    driver: bridge
    internal: true           # без доступа в интернет
```

> [!warning]
> Пароли в этом примере упрощены для обучения. Не коммить реальные значения в
> Compose-файл или `.env`; для production используй [[Docker — Секреты]].

---

## Команды Docker Compose

```bash
# ── Запуск ─────────────────────────────────────────────────────
docker compose up                     # запустить (foreground)
docker compose up -d                  # detached (фон)
docker compose up --build             # пересобрать образы перед запуском
docker compose up app db              # выбранные сервисы и их зависимости
docker compose up --scale app=3      # масштабировать сервис

# ── Остановка ──────────────────────────────────────────────────
docker compose stop                   # остановить контейнеры
docker compose down                   # остановить + удалить контейнеры + сети
docker compose down -v                # + удалить тома
docker compose down --rmi all         # + удалить образы

# ── Просмотр ───────────────────────────────────────────────────
docker compose ps                     # статус сервисов
docker compose ps -a                  # включая остановленные
docker compose logs                   # все логи
docker compose logs -f app            # следить за логами сервиса
docker compose logs --tail 100 app

# ── Управление ─────────────────────────────────────────────────
docker compose restart app            # перезапустить сервис
docker compose start app
docker compose stop app
docker compose pause app
docker compose unpause app

# ── Выполнение команд ──────────────────────────────────────────
docker compose exec app bash          # exec в работающем контейнере
docker compose exec app sh            # для alpine
docker compose exec -u root app bash
docker compose run --rm app npm test  # одноразовый контейнер

# ── Сборка ─────────────────────────────────────────────────────
docker compose build                  # собрать образы
docker compose build app              # только один сервис
docker compose build --no-cache app

# ── Утилиты ────────────────────────────────────────────────────
docker compose config                 # проверить и вывести итоговый конфиг
docker compose config --services      # список сервисов
docker compose images                 # образы используемых сервисов
docker compose top                    # процессы в контейнерах
docker compose pull                   # скачать образы без запуска
```

> [!note]
> При масштабировании нельзя привязать один и тот же фиксированный порт хоста ко
> всем репликам. В примере наружу публикуется только Nginx, а реплики `app`
> доступны ему через сеть Compose. `docker compose run` по умолчанию не публикует
> порты сервиса; при необходимости добавь `--service-ports`.

---

## Переменные и конфигурация

### .env файл

```bash
# .env (автоматически подхватывается Compose)
POSTGRES_VERSION=16
APP_PORT=3000
NODE_ENV=production
```

> [!important] Два разных механизма
> `.env` и `--env-file` прежде всего задают значения для подстановки
> `${VARIABLE}` в Compose-файл. Чтобы переменная попала **в контейнер**, укажи её
> в `environment` или `env_file` конкретного сервиса. Проверить результат можно
> командой `docker compose config`.

```yaml
services:
  db:
    image: postgres:${POSTGRES_VERSION:-15}   # дефолт если не задано
  app:
    ports:
      - "${APP_PORT}:3000"
```

### Несколько env-файлов

```bash
docker compose --env-file .env.staging up -d
```

---

## Переопределение конфигурации

```bash
# compose.yaml          — базовая конфигурация
# compose.override.yaml — автоматически объединяется с базовой при up

# Production
docker compose -f compose.yaml -f compose.prod.yaml up -d

# Staging
docker compose -f compose.yaml -f compose.staging.yaml up -d
```

**compose.override.yaml** (для dev — монтирует исходники):

```yaml
services:
  app:
    build:
      target: development
    volumes:
      - .:/app
      - /app/node_modules
    environment:
      - NODE_ENV=development
    command: npm run dev
```

>[!info]
>Переопределение позволяет управлять различными версиями приложения, например разработкой, тестированием и production
---

## depends_on и порядок запуска

```yaml
services:
  app:
    depends_on:
      db:
        condition: service_healthy   # ждать healthcheck
      redis:
        condition: service_started   # просто запущен
      migrations:
        condition: service_completed_successfully  # одноразовый job

  migrations:
    image: myapp:latest
    command: npm run migrate
    depends_on:
      db:
        condition: service_healthy
```

> [!warning]
> Короткая форма `depends_on: [db]` гарантирует порядок запуска, но не готовность
> приложения внутри контейнера. Длинная форма с
> `condition: service_healthy` ждёт успешного `healthcheck`, а
> `service_completed_successfully` — успешного завершения одноразового сервиса.
> Само приложение всё равно должно повторять временно неудавшиеся подключения.
---

## Profiles — группы сервисов

```yaml
services:
  app:
    image: myapp
    # без профиля — запускается всегда

  adminer:
    image: adminer
    profiles:
      - tools        # запускается только с --profile tools

  tests:
    image: myapp
    command: npm test
    profiles:
      - testing
```

```bash
docker compose --profile tools up -d         # поднять + adminer
docker compose --profile testing up tests    # включить профиль testing
docker compose run --rm tests                # явный запуск сервиса возможен и без профиля
```

---

## Healthcheck

```yaml
healthcheck:
  test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
  # test: ["CMD-SHELL", "pg_isready -U postgres"]  # shell форма
  interval: 30s       # как часто проверять
  timeout: 10s        # таймаут на одну проверку
  retries: 3          # сколько неудач до unhealthy
  start_period: 40s   # grace period после запуска
  disable: false      # true — отключить
```

> [!note]
> Команда проверки выполняется **внутри** контейнера. Утилита `curl` должна
> присутствовать в образе; в минимальных образах вместо неё используй имеющийся
> инструмент или собственную команду приложения.

---

## 📎 Связанные заметки

```dataview
LIST
FROM #docker
WHERE file.name != this.file.name
SORT file.name ASC
```
