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

```text
Image
├── manifest / index       ← config + упорядоченный список слоёв
├── config blob            ← Cmd, Env, Labels, History, diff IDs...
└── layer blobs            ← tar-архивы изменений файловой системы
```

> [!note]
> Это логическая схема OCI-образа. Точные имена каталогов после `docker save`
> зависят от формата экспорта и image store, поэтому ориентируйся на
> `manifest.json`/`index.json` и digest, а не на условный каталог `layers/`.

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
# → Проверит эффективность по правилам .dive-ci и вернёт ненулевой exit code,
#   если нарушены заданные пороги
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

# Сравнение новой версии с базовой
docker scout compare myapp:2.0 --to myapp:1.0

# Quick overview
docker scout quickview myapp:latest

# Интеграция с CI
docker scout cves --exit-code --only-severity critical myapp:latest
```

---

## SBOM — Software Bill of Materials

```bash
# Посмотреть SBOM через Docker Scout
docker scout sbom myapp:latest

# Syft: сохранить SBOM в стандартном формате
syft myapp:latest
syft myapp:latest -o spdx-json > sbom.json
syft myapp:latest -o cyclonedx-json > sbom.cyclonedx.json

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
RUN npm ci --omit=dev
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

FROM mcr.microsoft.com/dotnet/aspnet:8.0-jammy-chiseled AS final
WORKDIR /app
COPY --from=build /app/publish .
USER app
ENTRYPOINT ["dotnet", "MyApp.dll"]

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

| Образ | Shell | Пакетный менеджер | Важная особенность |
|-------|-------|-------------------|--------------------|
| Ubuntu / Debian | ✅ | `apt` | Удобная совместимость и диагностика |
| Alpine | `sh` | `apk` | Очень мал, но использует `musl` |
| .NET chiseled / distroless | ❌ | ❌ | Меньше поверхность атаки, сложнее отладка |
| `scratch` | ❌ | ❌ | Полностью пустая база; приложение приносит всё нужное само |

> [!note]
> Размеры тегов меняются, а Docker CLI показывает несжатый локальный размер.
> Сравнивай конкретные digest одной архитектуры, а не числа из статической таблицы.

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

```

> [!note]
> Локальный размер и объём скачивания из registry — разные величины: слои в
> registry сжаты и могут уже находиться в локальном cache. Поле размера descriptor
> самого manifest — это не сумма размеров его слоёв.

---

## Подпись и проверка образов

```bash
# Cosign (Sigstore)
cosign sign --key cosign.key myregistry.com/myapp:latest
cosign verify --key cosign.pub myregistry.com/myapp:latest
```

> [!danger] Docker Content Trust / Notary v1
> DCT устарел, удалён из Docker CLI 29 и сервис Notary v1 выводится из
> эксплуатации. Для новых систем используй Sigstore/Cosign, Notation и проверку
> attestations политиками платформы.

---

## Автоматизация в CI/CD

```yaml
# .github/workflows/security.yml
- name: Trivy vulnerability scanner
  uses: aquasecurity/trivy-action@v0.36.0
  with:
    image-ref: myapp:latest
    format: sarif
    output: trivy-results.sarif
    severity: CRITICAL,HIGH
    exit-code: 1

- name: Upload to GitHub Security
  uses: github/codeql-action/upload-sarif@v3
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
