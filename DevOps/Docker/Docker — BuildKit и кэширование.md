---
tags:
  - docker
  - devops
  - buildkit
  - build
topic: docker
subtopic: buildkit
status: готово
difficulty: продвинутый
created: 2026-06-24
aliases:
  - BuildKit
  - Docker BuildKit
  - Docker кэш сборки
---

# 🏗️ Docker — BuildKit и кэширование сборки

> [!info] Навигация
> [[Docker MOC]] → [[Docker — Dockerfile]] → **BuildKit** → [[Docker — Многоплатформенные образы]]

---

## Что такое BuildKit?

**BuildKit** — современный движок сборки Docker-образов, заменивший классический
builder. В Docker Engine он используется по умолчанию начиная с версии 23.0.

| | Классический builder | BuildKit |
|-|---------------------|---------|
| Параллельность | ❌ Последовательно | ✅ Параллельно |
| Кэш монтирования | ❌ | ✅ `--mount=type=cache` |
| SSH-forwarding | ❌ | ✅ `--mount=type=ssh` |
| Секреты при сборке | ❌ | ✅ `--mount=type=secret` |
| Inline cache | ❌ | ✅ `--cache-from` |
| Многоплатформенность | ❌ | ✅ buildx |

```bash
# Включить принудительно (если старый Docker)
export DOCKER_BUILDKIT=1

# Проверить версию builder
docker buildx version
docker buildx ls
```

---

## Cache Mounts — главная фича BuildKit

**Проблема:** обычный layer cache полностью пропускает `RUN npm ci`, пока
предыдущие инструкции не изменились. Но если слой инвалидирован, установка
запускается заново и без отдельного package cache снова скачивает зависимости.

**Решение:** `--mount=type=cache` — переиспользуемая директория кэша **между
сборками**, содержимое которой не попадает в итоговый слой образа.

```dockerfile
# ── Node.js ───────────────────────────────────────────────────
FROM node:20-alpine
WORKDIR /app

COPY package*.json ./

# Кэш npm между сборками → не скачиваем повторно уже загруженные пакеты
RUN --mount=type=cache,target=/root/.npm \
    npm ci --prefer-offline

COPY . .
RUN npm run build
```

```dockerfile
# ── .NET / NuGet ──────────────────────────────────────────────
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src

COPY *.csproj .
# Кэш NuGet-пакетов — огромная экономия на CI при монолитных solution
RUN --mount=type=cache,target=/root/.nuget/packages \
    dotnet restore

COPY . .
RUN --mount=type=cache,target=/root/.nuget/packages \
    dotnet publish -c Release -o /app/publish --no-restore
```

```dockerfile
# ── Python / pip ──────────────────────────────────────────────
FROM python:3.12-slim
WORKDIR /app

COPY requirements.txt .
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install -r requirements.txt
```

```dockerfile
# ── Go ────────────────────────────────────────────────────────
FROM golang:1.22-alpine AS build
WORKDIR /app

COPY go.mod go.sum ./
RUN --mount=type=cache,target=/go/pkg/mod \
    --mount=type=cache,target=/root/.cache/go-build \
    go mod download

COPY . .
RUN --mount=type=cache,target=/go/pkg/mod \
    --mount=type=cache,target=/root/.cache/go-build \
    go build -o /bin/app ./cmd/app
```

```dockerfile
# ── apt / apk ─────────────────────────────────────────────────
FROM ubuntu:22.04
RUN rm -f /etc/apt/apt.conf.d/docker-clean \
    && echo 'Binary::apt::APT::Keep-Downloaded-Packages "true";' \
       > /etc/apt/apt.conf.d/keep-cache
RUN --mount=type=cache,target=/var/cache/apt,sharing=locked \
    --mount=type=cache,target=/var/lib/apt,sharing=locked \
    apt-get update \
    && apt-get install -y --no-install-recommends curl git
```

> [!tip] Параметры cache mount
> - `id=mycache` — уникальный идентификатор кэша (по умолчанию = target)
> - `sharing=shared` (default) / `locked` / `private`
> - `mode=0755` — права на директорию
> - `uid`, `gid` — владелец
>
> Сборка не должна зависеть от наличия cache mount: builder может очистить кэш,
> а параллельная сборка — изменить его содержимое.

---

## SSH Mount — приватные репозитории

Позволяет использовать SSH-ключ хоста при сборке, **не копируя его в образ**:

```dockerfile
# syntax=docker/dockerfile:1
FROM golang:1.22-alpine

# Установить SSH-клиент и доверять конкретному ключу хоста
RUN apk add --no-cache git openssh-client \
    && mkdir -p -m 0700 /root/.ssh \
    && ssh-keyscan github.com >> /root/.ssh/known_hosts \
    && \
    git config --global url."git@github.com:".insteadOf "https://github.com/" && \
    go env -w GOPRIVATE="github.com/myorg/*"

COPY go.mod go.sum ./
RUN --mount=type=ssh \
    --mount=type=cache,target=/go/pkg/mod \
    go mod download

COPY . .
RUN --mount=type=cache,target=/root/.cache/go-build \
    go build ./...
```

> [!warning]
> `ssh-keyscan` удобно для примера, но в CI безопаснее сверять полученный host key
> с заранее закреплённым fingerprint, иначе возможна атака man-in-the-middle.

```bash
# Сборка с forwarding SSH-агента
eval $(ssh-agent)
ssh-add ~/.ssh/id_rsa
docker build --ssh default .

# Или явно указать ключ
docker build --ssh mykey=~/.ssh/id_rsa .
# В Dockerfile: --mount=type=ssh,id=mykey
```

---

## Bind Mount в RUN — без копирования

Монтирование исходных файлов только на время шага сборки: сами смонтированные
файлы не копируются в слой, но изменения, которые команда запишет вне mount,
по-прежнему войдут в результат `RUN`.

```dockerfile
# Установка зависимостей без постоянного COPY package.json
RUN --mount=type=bind,source=package.json,target=package.json \
    --mount=type=bind,source=package-lock.json,target=package-lock.json \
    --mount=type=cache,target=/root/.npm \
    npm ci

# Более распространённый вариант
COPY package*.json ./
RUN npm ci
```

---

## Secret Mount — секреты при сборке

Секрет доступен только во время шага RUN, не попадает в слои:

```dockerfile
# Готовый .npmrc временно подменяет файл конфигурации
RUN --mount=type=secret,id=npmrc,target=/root/.npmrc,required=true \
    npm ci

# NuGet.Config с credentials существует только во время restore
RUN --mount=type=secret,id=nuget_config,target=/root/.nuget/NuGet/NuGet.Config,required=true \
    dotnet restore
```

```bash
docker build --secret id=npmrc,src="$HOME/.npmrc" .
docker build --secret id=nuget_config,src=NuGet.Config .
```

> [!danger]
> Не выполняй `npm config set ...TOKEN...` или `dotnet nuget add source
> --password ...` в обычной файловой системе шага: такие команды могут записать
> секрет в конфигурационный файл, который попадёт в слой. Монтируй сам файл
> конфигурации как secret или используй документированный способ передачи
> credentials конкретного package manager.

---

## Inline Cache — кэш в образе для CI

Позволяет использовать образ из registry как кэш для следующих сборок:

```bash
# Сборка с сохранением метаданных кэша
docker buildx build \
  --cache-to type=inline \
  --tag myapp:latest \
  --push .

# Следующая сборка использует кэш из registry
docker buildx build \
  --cache-from type=registry,ref=myapp:latest \
  --tag myapp:new-version \
  --push .
```

```yaml
# GitHub Actions
- name: Build and push
  uses: docker/build-push-action@v7
  with:
    cache-from: type=gha
    cache-to: type=gha,mode=max
    push: true
    tags: ghcr.io/org/app:latest
```

> [!note]
> `cache-to: type=gha` сохраняет layer cache BuildKit, но cache mounts из
> `RUN --mount=type=cache` автоматически в GitHub Actions cache не экспортирует.
> Для них нужен отдельный механизм или runner с постоянным builder storage.

---

## Многоэтапная сборка: оптимизация кэша

Порядок инструкций критически важен — каждое изменение инвалидирует все последующие слои:

```dockerfile
# ❌ Плохо: изменение любого файла сбрасывает кэш npm ci
FROM node:20-alpine
COPY . .                    # <- весь код
RUN npm ci                  # <- всегда заново

# ✅ Хорошо: npm ci кэшируется пока package.json не изменился
FROM node:20-alpine
COPY package*.json ./       # <- только манифест зависимостей
RUN npm ci                  # <- кэшировано!
COPY src/ ./src/            # <- код (частые изменения)
RUN npm run build
```

```dockerfile
# ✅ .NET — оптимальный порядок с cache mount
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src

# 1. Только project файлы → restore кэшируется
# В multi-project solution сохраняй структуру каталогов.
# Универсального COPY с сохранением путей нет — перечисли project-файлы явно
# или генерируй минимальный restore-контекст отдельным инструментом.
COPY MyApp.sln ./
# Следующая строка нужна, только если эти файлы есть в solution
COPY Directory.Build.props Directory.Packages.props ./
COPY src/MyApp/MyApp.csproj src/MyApp/
COPY tests/MyApp.Tests/MyApp.Tests.csproj tests/MyApp.Tests/
RUN --mount=type=cache,target=/root/.nuget/packages \
    dotnet restore MyApp.sln

# 2. Исходники (меняются часто)
COPY . .
RUN --mount=type=cache,target=/root/.nuget/packages \
    dotnet build -c Release --no-restore

# 3. Тесты
FROM build AS test
RUN dotnet test --no-build -c Release

# 4. Публикация
FROM build AS publish
RUN dotnet publish -c Release -o /app/publish --no-build

# 5. Runtime образ
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS final
WORKDIR /app
COPY --from=publish /app/publish .
EXPOSE 8080
ENTRYPOINT ["dotnet", "MyApp.dll"]
```

---

## Dockerfile Heredoc — многострочные команды

```dockerfile
# syntax=docker/dockerfile:1

# Heredoc для RUN
RUN <<EOF
  set -eu
  apt-get update
  apt-get install -y curl git
  rm -rf /var/lib/apt/lists/*
EOF

# Heredoc для создания файла
COPY <<EOF /etc/nginx/conf.d/default.conf
server {
    listen 80;
    location / {
        proxy_pass http://app:3000;
    }
}
EOF

# Python скрипт прямо в Dockerfile
RUN python3 <<EOF
import json, os
config = {"debug": False, "port": 8080}
with open("/app/config.json", "w") as f:
    json.dump(config, f)
EOF
```

---

## docker buildx bake — декларативная сборка

`bake` позволяет описать несколько образов в одном HCL/JSON/Compose файле:

```hcl
# docker-bake.hcl
variable "TAG" {
  default = "latest"
}

group "default" {
  targets = ["app", "migrations", "worker"]
}

target "app" {
  context    = "."
  dockerfile = "Dockerfile"
  target     = "production"
  tags       = ["myregistry/app:${TAG}"]
  platforms  = ["linux/amd64", "linux/arm64"]
  cache-from = ["type=registry,ref=myregistry/app:cache"]
  cache-to   = ["type=registry,ref=myregistry/app:cache,mode=max"]
}

target "migrations" {
  context    = "."
  dockerfile = "migrations/Dockerfile"
  tags       = ["myregistry/migrations:${TAG}"]
}

target "worker" {
  inherits = ["app"]          # наследует все настройки app
  target   = "worker"
  tags     = ["myregistry/worker:${TAG}"]
}
```

```bash
# Собрать все targets
docker buildx bake

# Только один target
docker buildx bake app

# С переопределением переменной
TAG=v1.2.3 docker buildx bake --push

# CI/CD: dry-run (посмотреть конфиг без сборки)
docker buildx bake --print
```

---

## Анализ кэша и оптимизации

```bash
# Посмотреть что занимает место в кэше
docker buildx du

# Очистить кэш buildx
docker buildx prune
docker buildx prune --keep-storage 10gb   # оставить до 10 ГБ

# Узнать что инвалидировало кэш (debug режим)
docker buildx build --progress=plain . 2>&1 | grep -E "CACHED|RUN"
# CACHED означает использование кэша
# Строка без CACHED = кэш промахнулся
```

---

## 📎 Связанные заметки

```dataview
LIST
FROM #docker
WHERE file.name != this.file.name
SORT file.name ASC
```
