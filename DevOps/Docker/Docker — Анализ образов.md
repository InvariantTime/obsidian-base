---
tags:
  - docker
  - devops
  - security
  - images
topic: docker
subtopic: анализ-образов
status: готово
difficulty: продвинутый
created: 2026-06-24
aliases:
  - Docker Image Analysis
  - Анализ Docker образов
---

# 🔍 Docker — Анализ образов

> [!info] Навигация
> [[Docker MOC]] → [[Docker — Образы]] → **Анализ образов** → [[Docker — Пользователи и права]]

---

## Структура образа (OCI Image Spec)

```
Image
├── manifest.json          ← список слоёв + config
├── config.json            ← метаданные: Cmd, Env, Labels, History...
└── layers/
    ├── sha256:abc.tar.gz  ← слой 1 (базовый образ)
    ├── sha256:def.tar.gz  ← слой 2 (RUN apt install)
    └── sha256:ghi.tar.gz  ← слой 3 (COPY app/)
```

```bash
# Посмотреть manifest образа
docker manifest inspect nginx:latest
docker buildx imagetools inspect nginx:latest  # мультиплатформенный

# Экспортировать образ и изучить структуру
docker save myapp:latest | tar -xv -C /tmp/image-inspect/
ls /tmp/image-inspect/
cat /tmp/image-inspect/manifest.json | jq .
cat /tmp/image-inspect/<config-id>.json | jq '.config, .history'
```

---

## docker history — история слоёв

```bash
# Краткая история
docker history myapp:latest

# Полная (без обрезки)
docker history --no-trunc myapp:latest

# Только большие слои (> 1 MB)
docker history --no-trunc --format "{{.Size}}\t{{.CreatedBy}}" myapp:latest \
  | sort -rh | head -20

# Посмотреть размер каждого слоя
docker history --format "table {{.Size}}\t{{.CreatedBy}}" myapp:latest
```

> [!tip]
> `docker history` показывает команды Dockerfile в обратном порядке.
> ARG-значения могут быть видны здесь — именно поэтому BuildKit `--secret` важен.

---

## dive — интерактивный анализ слоёв

```bash
# Установка
brew install dive   # macOS
# или docker run:
alias dive='docker run --rm -it \
  -v /var/run/docker.sock:/var/run/docker.sock \
  wagoodman/dive'

# Запуск
dive myapp:latest

# CI-режим (проверить эффективность)
dive --ci myapp:latest
# → Провалится если есть файлы, которые были добавлены и удалены в разных слоях
#   (тратят место зря)
```

**Что искать в dive:**
- Файлы добавленные в слой N и удалённые в слое N+1 → зря занимают место
- Большие директории в слоях (node_modules, .cache, tmp)
- Дублированные файлы между слоями

---

## Сканирование уязвимостей: Trivy

```bash
# Установка
brew install trivy   # macOS
curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sh

# Сканирование образа
trivy image myapp:latest

# Только HIGH и CRITICAL
trivy image --severity HIGH,CRITICAL myapp:latest

# JSON для автоматизации
trivy image --format json --output report.json myapp:latest

# Игнорировать unfixed уязвимости
trivy image --ignore-unfixed myapp:latest

# Сканировать Dockerfile (misconfig)
trivy config Dockerfile

# Сканировать docker-compose.yml
trivy config docker-compose.yml

# Встроить в CI (не провалить pipeline на LOW)
trivy image --exit-code 1 --severity CRITICAL myapp:latest
```

---

## Docker Scout (встроено в Docker)

```bash
# Уязвимости
docker scout cves myapp:latest
docker scout cves --only-severity critical,high myapp:latest

# Рекомендации по базовому образу
docker scout recommendations myapp:latest

# Сравнение двух версий
docker scout compare myapp:1.0 myapp:2.0

# Quick overview
docker scout quickview myapp:latest

# Интеграция с CI
docker scout cves --exit-code --only-severity critical myapp:latest
```

---

## SBOM — Software Bill of Materials

```bash
# Сгенерировать SBOM
docker sbom myapp:latest
docker sbom --format spdx-json myapp:latest > sbom.spdx.json
docker sbom --format cyclonedx-json myapp:latest > sbom.cyclonedx.json

# Syft (более гибкий инструмент)
syft myapp:latest
syft myapp:latest -o spdx-json > sbom.json

# Grype — сканировать SBOM на уязвимости
syft myapp:latest -o json | grype

# Записать SBOM как аттестацию в registry
docker buildx build \
  --sbom=true \
  --provenance=true \
  -t myapp:latest \
  --push .
```

---

## Distroless образы

Образы без shell, пакетного менеджера и прочих инструментов ОС.  
**Только runtime + приложение.**

```dockerfile
# ── Node.js ────────────────────────────────────────────────────
FROM node:20-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .

FROM gcr.io/distroless/nodejs20-debian12 AS final
WORKDIR /app
COPY --from=build /app .
USER 1001
CMD ["index.js"]

# ── .NET ───────────────────────────────────────────────────────
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src
COPY *.csproj .
RUN dotnet restore
COPY . .
RUN dotnet publish -c Release -o /app/publish

FROM gcr.io/distroless/dotnet8 AS final
WORKDIR /app
COPY --from=build /app/publish .
USER 1001
ENTRYPOINT ["MyApp"]

# ── Go (scratch) ───────────────────────────────────────────────
FROM golang:1.22 AS build
WORKDIR /src
COPY go.* ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -ldflags="-s -w" -o /app/server .

FROM scratch AS final
COPY --from=build /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
COPY --from=build /app/server /server
USER 65532:65532
ENTRYPOINT ["/server"]
```

| Образ | Размер | Shell | Пакеты |
|-------|--------|-------|--------|
| `ubuntu:22.04` | ~77 MB | bash | apt |
| `debian:slim` | ~75 MB | bash | apt |
| `alpine:3.19` | ~7 MB | sh | apk |
| `distroless/base` | ~20 MB | ❌ | ❌ |
| `distroless/nodejs20` | ~90 MB | ❌ | ❌ |
| `scratch` | 0 MB | ❌ | ❌ |

---

## Отладка distroless: debug-вариант

```bash
# Distroless предоставляет :debug тег с busybox shell
docker run --rm -it gcr.io/distroless/nodejs20-debian12:debug sh

# Или — ephemeral debug container (K8s)
kubectl debug -it pod/myapp --image=gcr.io/distroless/nodejs20-debian12:debug
```

---

## Benchmarking размера образа

```bash
# Сравнение размеров
docker images --format "table {{.Repository}}:{{.Tag}}\t{{.Size}}" \
  | grep myapp | sort -k2 -h

# Реальный размер с учётом shared layers
docker system df -v | grep myapp

# Сжатый размер (как в registry)
docker manifest inspect --verbose myapp:latest \
  | jq '.Descriptor.size' | numfmt --to=iec
```

---

## Политики подписи образов (Content Trust)

```bash
# Docker Content Trust (Notary v1)
export DOCKER_CONTENT_TRUST=1
docker pull myapp:latest    # проверит подпись
docker push myapp:latest    # потребует подписать

# Cosign (Sigstore — современный подход)
cosign sign --key cosign.key myregistry.com/myapp:latest
cosign verify --key cosign.pub myregistry.com/myapp:latest
```

---

## Автоматизация в CI/CD

```yaml
# .github/workflows/security.yml
- name: Trivy vulnerability scanner
  uses: aquasecurity/trivy-action@master
  with:
    image-ref: myapp:latest
    format: sarif
    output: trivy-results.sarif
    severity: CRITICAL,HIGH
    exit-code: 1

- name: Upload to GitHub Security
  uses: github/codeql-action/upload-sarif@v2
  with:
    sarif_file: trivy-results.sarif
```

---

## 📎 Связанные заметки

```dataview
LIST
FROM #docker
WHERE file.name != this.file.name
SORT file.name ASC
```
