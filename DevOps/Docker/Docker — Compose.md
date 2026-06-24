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

---

## Структура docker-compose.yml

```yaml
version: "3.9"              # версия спецификации Compose

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

---

## Полный пример: веб-приложение

```yaml
version: "3.9"

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
    ports:
      - "3000:3000"
    volumes:
      - uploads:/app/uploads
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_started
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
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
    command: redis-server --appendonly yes --requirepass mypassword
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

---

## Команды Docker Compose

```bash
# ── Запуск ─────────────────────────────────────────────────────
docker compose up                     # запустить (foreground)
docker compose up -d                  # detached (фон)
docker compose up --build             # пересобрать образы перед запуском
docker compose up app db              # только конкретные сервисы
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
docker compose start / stop / pause / unpause

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

---

## Переменные и конфигурация

### .env файл

```bash
# .env (автоматически подхватывается Compose)
POSTGRES_VERSION=16
APP_PORT=3000
NODE_ENV=production
```

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
# docker-compose.yml       — базовая конфигурация
# docker-compose.override.yml — автоматически мержится при up

# Production
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d

# Staging
docker compose -f docker-compose.yml -f docker-compose.staging.yml up -d
```

**docker-compose.override.yml** (для dev — монтирует исходники):

```yaml
version: "3.9"
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
docker compose --profile testing run tests   # запустить тесты
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

---

## 📎 Связанные заметки

```dataview
LIST
FROM #docker
WHERE file.name != this.file.name
SORT file.name ASC
```
