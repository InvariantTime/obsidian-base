---
tags:
  - docker
  - devops
  - containers
topic: docker
subtopic: dockerfile
status: готово
difficulty: средний
created: 2026-06-24
aliases:
  - Dockerfile
---

# 🔧 Docker — Dockerfile

> [!info] Навигация
> [[Docker MOC]] → [[Docker — Основы]] → **Dockerfile** → [[Docker — Образы]]

---

## Что такое Dockerfile?

**Dockerfile** — текстовый файл с инструкциями для сборки Docker-образа. Каждая инструкция создаёт новый **слой (layer)** в образе.

```bash
docker build -t myapp:1.0 .        # сборка образа из Dockerfile в текущей папке
docker build -f path/Dockerfile .  # указать путь к Dockerfile
```

---

## Все инструкции Dockerfile

### `FROM` — базовый образ

```dockerfile
FROM ubuntu:22.04
FROM node:20-alpine          # легковесный alpine
FROM scratch                 # пустой образ (для Go/C бинарников)
```

> [!tip] Выбор базового образа
> Предпочитай `-alpine` или `-slim` варианты — они значительно меньше по размеру.

---

### `WORKDIR` — рабочая директория

```dockerfile
WORKDIR /app        # создаёт папку и делает её текущей
WORKDIR /app/src    # вложенные папки создаются автоматически
```

---

### `COPY` и `ADD`

```dockerfile
# COPY — простое копирование файлов
COPY package.json .
COPY src/ ./src/
COPY --chown=node:node . .   # сменить владельца

# ADD — то же самое + распаковка архивов + URL
ADD app.tar.gz /app/         # автоматически распакует
ADD https://example.com/file.txt /tmp/  # скачает
```

> [!warning]
> Используй `COPY` по умолчанию, `ADD` — только когда нужна распаковка архива.

---

### `RUN` — выполнение команд при сборке

```dockerfile
RUN apt-get update && apt-get install -y     curl     git     && rm -rf /var/lib/apt/lists/*    # чистим кэш в том же слое!

RUN npm ci --only=production
```

> [!important] Объединяй команды через `&&`
> Каждый `RUN` — новый слой. Очищай кэш в том же `RUN`, где устанавливал пакеты.

---

### `ENV` — переменные окружения

```dockerfile
ENV NODE_ENV=production
ENV PORT=3000 HOST=0.0.0.0   # несколько переменных
```

---

### `ARG` — аргументы сборки

```dockerfile
ARG VERSION=1.0
ARG BUILD_DATE

RUN echo "Building version $VERSION"
```

```bash
docker build --build-arg VERSION=2.0 .
```

> [!note] `ENV` vs `ARG`
> `ARG` доступен только во время сборки, `ENV` — и во время сборки, и в контейнере.

---

### `EXPOSE` — объявление портов

```dockerfile
EXPOSE 3000        # TCP по умолчанию
EXPOSE 8080/tcp
EXPOSE 9090/udp
```

> [!note]
> `EXPOSE` — документация, а не реальный проброс портов. Реальный проброс — через `-p` при `docker run`.

---

### `VOLUME` — объявление томов

```dockerfile
VOLUME ["/data", "/logs"]
VOLUME /var/lib/mysql
```

---

### `USER` — пользователь для запуска

```dockerfile
RUN groupadd -r appuser && useradd -r -g appuser appuser
USER appuser        # никогда не запускай контейнер от root!
```

---

### `CMD` и `ENTRYPOINT`

```dockerfile
# CMD — команда по умолчанию (можно переопределить при docker run)
CMD ["node", "index.js"]           # exec форма (предпочтительно)
CMD node index.js                  # shell форма

# ENTRYPOINT — основная команда (сложнее переопределить)
ENTRYPOINT ["docker-entrypoint.sh"]
ENTRYPOINT ["npm", "run"]

# Комбинация: ENTRYPOINT + CMD
ENTRYPOINT ["npm", "run"]
CMD ["start"]                      # docker run myapp → npm run start
                                   # docker run myapp test → npm run test
```

| | `CMD` | `ENTRYPOINT` |
|-|-------|--------------|
| Переопределение | `docker run img другая_cmd` | `docker run --entrypoint другой img` |
| Аргументы из `CMD` | — | Используются как аргументы по умолчанию |
| Рекомендуется для | Дефолтных аргументов | Основного процесса контейнера |

---

### `HEALTHCHECK` — проверка состояния

```dockerfile
HEALTHCHECK --interval=30s --timeout=10s --retries=3   CMD curl -f http://localhost:3000/health || exit 1

HEALTHCHECK NONE   # отключить healthcheck из базового образа
```

---

### `LABEL` — метаданные

```dockerfile
LABEL maintainer="dev@example.com"
LABEL version="1.0" description="My App"
LABEL org.opencontainers.image.source="https://github.com/org/repo"
```

---

### `ONBUILD` — инструкция для дочерних образов

```dockerfile
ONBUILD COPY . /app
ONBUILD RUN npm ci
```

---

## Многоэтапная сборка (Multi-stage build)

Позволяет использовать один Dockerfile для сборки и создания финального лёгкого образа:

```dockerfile
# ── Этап 1: сборка ───────────────────────────────
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build      # компиляция TypeScript → JS

# ── Этап 2: финальный образ ──────────────────────
FROM node:20-alpine AS production
WORKDIR /app

# Копируем ТОЛЬКО нужное из этапа сборки
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
COPY package.json .

RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser

EXPOSE 3000
CMD ["node", "dist/index.js"]
```

> [!success] Результат
> Финальный образ не содержит исходников, devDependencies и инструментов сборки.

---

## Пример: ASP.NET Core

```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src
COPY *.csproj .
RUN dotnet restore
COPY . .
RUN dotnet publish -c Release -o /app/publish

FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS final
WORKDIR /app
COPY --from=build /app/publish .
EXPOSE 8080
ENTRYPOINT ["dotnet", "MyApp.dll"]
```

---

## .dockerignore

Исключает файлы из контекста сборки (аналог `.gitignore`):

```
node_modules/
.git/
.env
*.log
dist/
coverage/
.DS_Store
```

---

## Best Practices

> [!tip] Чеклист хорошего Dockerfile

- [ ] Используй конкретный тег образа (`node:20-alpine`, не `node:latest`)
- [ ] Сначала копируй `package.json` → `RUN npm ci` → потом исходники (кэширование слоёв!)
- [ ] Объединяй `RUN` команды и чисти кэш пакетного менеджера в том же слое
- [ ] Никогда не запускай от `root` — создай отдельного пользователя
- [ ] Добавь `.dockerignore`
- [ ] Используй multi-stage build для production
- [ ] Добавь `HEALTHCHECK`
- [ ] Используй exec форму для `CMD` и `ENTRYPOINT`: `["node", "app.js"]`

---

## 📎 Связанные заметки

```dataview
LIST
FROM #docker
WHERE file.name != this.file.name
SORT file.name ASC
```
