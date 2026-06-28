---
tags:
  - docker
  - devops
  - buildx
  - multiarch
  - arm
topic: docker
subtopic: многоплатформенные-образы
status: готово
difficulty: продвинутый
created: 2026-06-24
aliases:
  - Docker Multi-arch
  - Docker buildx
  - Мультиплатформенные образы
---

# 🌍 Docker — Многоплатформенные образы

> [!info] Навигация
> [[Docker MOC]] → [[Docker — BuildKit и кэширование]] → **Многоплатформенные образы**

---

## Зачем нужны мультиплатформенные образы?

| Сценарий | Платформа |
|----------|-----------|
| CI/CD сервер (x86 облако) | `linux/amd64` |
| Apple Silicon, AWS Graviton | `linux/arm64` |
| Raspberry Pi | `linux/arm/v7` |
| Windows контейнеры | `windows/amd64` |
| Embedded устройства | `linux/arm/v6`, `linux/386` |

> [!info] Как Docker выбирает образ?
> При `docker pull nginx` Docker смотрит на платформу хоста и автоматически выбирает нужный вариант из **manifest list** (многоплатформенного образа).

---

## docker buildx — мультиплатформенный builder

```bash
# Посмотреть доступные builder-ы
docker buildx ls

# Создать новый builder с поддержкой multi-platform
docker buildx create --name multibuilder --driver docker-container --use
docker buildx inspect --bootstrap   # загрузить поддерживаемые платформы

# Сборка для одной платформы
docker buildx build --platform linux/arm64 -t myapp:arm64 .

# Сборка для нескольких платформ одновременно + push в registry
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  --tag myregistry/myapp:latest \
  --push \
  .

# Список поддерживаемых платформ builder-а
docker buildx inspect multibuilder | grep Platforms
```

> [!warning] Без `--push` нельзя загрузить multi-arch образ локально
> `--load` работает только для одной платформы. Multi-arch образ должен сразу уходить в registry.

---

## QEMU — эмуляция архитектур

Для сборки `linux/arm64` на `linux/amd64` машине используется **QEMU**-эмуляция:

```bash
# Установить QEMU (нужно один раз)
docker run --privileged --rm tonistiigi/binfmt --install all

# Проверить доступные эмуляции
ls /proc/sys/fs/binfmt_misc/
# qemu-aarch64, qemu-arm, qemu-riscv64 и т.д.
```

> [!note] Эмуляция vs кросс-компиляция
> - **QEMU эмуляция** — медленно (~10x), но работает для любого кода
> - **Кросс-компиляция** — быстро, требует отдельной настройки компилятора

---

## ARG BUILDPLATFORM / TARGETPLATFORM

BuildKit предоставляет автоматические ARG для управления платформой:

| ARG | Пример значения | Описание |
|-----|----------------|---------|
| `BUILDPLATFORM` | `linux/amd64` | Платформа **builder-а** (где идёт сборка) |
| `TARGETPLATFORM` | `linux/arm64` | Целевая платформа **образа** |
| `TARGETOS` | `linux` | ОС целевой платформы |
| `TARGETARCH` | `arm64` | Архитектура целевой платформы |
| `TARGETVARIANT` | `v7` | Вариант архитектуры (для ARM) |

```dockerfile
# syntax=docker/dockerfile:1
FROM --platform=$BUILDPLATFORM golang:1.22-alpine AS build

# Указать целевую платформу для компилятора
ARG TARGETOS TARGETARCH
RUN --mount=type=cache,target=/go/pkg/mod \
    --mount=type=cache,target=/root/.cache/go-build \
    GOOS=$TARGETOS GOARCH=$TARGETARCH go build -o /bin/app ./cmd/app
#   ↑ кросс-компиляция: сборщик остаётся amd64, бинарник компилируется для arm64

FROM --platform=$TARGETPLATFORM alpine:3.19
COPY --from=build /bin/app /bin/app
ENTRYPOINT ["/bin/app"]
```

```dockerfile
# ── .NET кросс-компиляция ─────────────────────────────────────
FROM --platform=$BUILDPLATFORM mcr.microsoft.com/dotnet/sdk:8.0 AS build
ARG TARGETARCH
WORKDIR /src

COPY *.csproj .
RUN dotnet restore -a $TARGETARCH    # restore под целевую arch

COPY . .
RUN dotnet publish -c Release -a $TARGETARCH -o /app/publish --no-restore

# Runtime образ — уже для $TARGETPLATFORM
FROM mcr.microsoft.com/dotnet/aspnet:8.0
WORKDIR /app
COPY --from=build /app/publish .
EXPOSE 8080
ENTRYPOINT ["dotnet", "MyApp.dll"]
```

> [!tip] Почему `FROM --platform=$BUILDPLATFORM` в builder-этапе?
> Если builder на amd64, а target — arm64, то `FROM golang:1.22-alpine` без `--platform` скачает arm64 образ и запустит его через QEMU. Это **медленно**. С `--platform=$BUILDPLATFORM` builder-инструменты (компилятор) работают нативно, кросс-компилируя в arm64.

---

## Платформо-зависимые инструкции

```dockerfile
FROM alpine:3.19

ARG TARGETARCH

# Разная логика для разных архитектур
RUN case "$TARGETARCH" in \
    amd64) URL="https://releases.example.com/tool-linux-amd64.tar.gz" ;; \
    arm64) URL="https://releases.example.com/tool-linux-arm64.tar.gz" ;; \
    arm)   URL="https://releases.example.com/tool-linux-armv7.tar.gz" ;; \
    *) echo "Unsupported arch: $TARGETARCH" && exit 1 ;; \
    esac && \
    wget -qO- "$URL" | tar xz -C /usr/local/bin/
```

---

## Manifest List (Multi-arch образ)

**Manifest list** — "оглавление", которое указывает Docker какой образ выбрать по платформе:

```bash
# Посмотреть manifest list образа
docker buildx imagetools inspect nginx:latest

# Пример вывода:
# Name:      docker.io/library/nginx:latest
# MediaType: application/vnd.oci.image.index.v1+json
# Digest:    sha256:...
#
# Manifests:
#   Name:      .../nginx:latest@sha256:aaa  Platform: linux/amd64
#   Name:      .../nginx:latest@sha256:bbb  Platform: linux/arm64
#   Name:      .../nginx:latest@sha256:ccc  Platform: linux/arm/v7

# Создать manifest list вручную из уже загруженных образов
docker buildx imagetools create \
  --tag myregistry/myapp:latest \
  myregistry/myapp:amd64 \
  myregistry/myapp:arm64
```

---

## CI/CD: GitHub Actions

```yaml
name: Build multi-arch image

on:
  push:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write

    steps:
      - uses: actions/checkout@v4

      # Установить QEMU для кросс-платформенной эмуляции
      - name: Set up QEMU
        uses: docker/setup-qemu-action@v3

      # Создать buildx builder с multi-platform поддержкой
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Log in to GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      # Генерация тегов и labels
      - name: Docker meta
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ghcr.io/${{ github.repository }}
          tags: |
            type=sha
            type=ref,event=branch
            type=semver,pattern={{version}}

      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          platforms: linux/amd64,linux/arm64
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

---

## GitLab CI

```yaml
variables:
  PLATFORMS: linux/amd64,linux/arm64

build-multiarch:
  image: docker:latest
  services:
    - docker:dind
  before_script:
    - docker buildx create --use --driver docker-container
    - docker buildx inspect --bootstrap
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
  script:
    - docker buildx build
        --platform $PLATFORMS
        --tag $CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA
        --tag $CI_REGISTRY_IMAGE:latest
        --push
        .
```

---

## Стратегии сборки

### Стратегия 1: QEMU эмуляция (простая, медленная)

```bash
docker buildx build --platform linux/amd64,linux/arm64 --push .
# Все архитектуры собираются через эмуляцию на одной машине
```

**Плюсы:** Просто настроить, один CI runner  
**Минусы:** arm64 через эмуляцию может занять в 5-10 раз дольше

### Стратегия 2: Нативные runners (быстрая)

```yaml
# GitHub Actions: параллельная сборка на нативных runner-ах
jobs:
  build:
    strategy:
      matrix:
        include:
          - platform: linux/amd64
            runner: ubuntu-latest
          - platform: linux/arm64
            runner: ubuntu-latest-arm64   # ARM runner
    runs-on: ${{ matrix.runner }}
    steps:
      - name: Build
        run: |
          docker buildx build \
            --platform ${{ matrix.platform }} \
            --tag ghcr.io/org/app:${{ github.sha }}-${{ matrix.runner }} \
            --push .

  merge:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - name: Create manifest list
        run: |
          docker buildx imagetools create \
            --tag ghcr.io/org/app:latest \
            ghcr.io/org/app:${{ github.sha }}-ubuntu-latest \
            ghcr.io/org/app:${{ github.sha }}-ubuntu-latest-arm64
```

### Стратегия 3: Кросс-компиляция (идеально для Go/.NET)

```dockerfile
# Весь build-этап работает на нативной amd64 архитектуре
# только итоговый бинарник скомпилирован для arm64
FROM --platform=$BUILDPLATFORM mcr.microsoft.com/dotnet/sdk:8.0 AS build
ARG TARGETARCH
RUN dotnet publish -a $TARGETARCH ...
```

---

## Полезные команды

```bash
# Сколько платформ поддерживает образ
docker buildx imagetools inspect myimage:tag | grep Platform

# Скачать образ для конкретной платформы
docker pull --platform linux/arm64 nginx:latest

# Запустить образ для другой платформы (через QEMU)
docker run --platform linux/arm64 --rm alpine uname -m
# → aarch64

# Удалить builder
docker buildx rm multibuilder

# Использовать стандартный builder
docker buildx use default
```

---

## 📎 Связанные заметки

```dataview
LIST
FROM #docker
WHERE file.name != this.file.name
SORT file.name ASC
```
