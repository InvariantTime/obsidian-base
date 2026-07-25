---
tags:
  - docker
  - devops
  - security
  - secrets
topic: docker
subtopic: секреты
status: готово
difficulty: продвинутый
created: 2026-06-24
aliases:
  - Docker Secrets
  - Секреты Docker
---

# 🔑 Docker — Секреты

> [!info] Навигация
> [[Docker MOC]] → [[Docker — Пользователи и права]] → **Секреты** → [[Docker — Профилирование и диагностика]]

---

## Почему ENV — плохой способ хранить секреты

```bash
# Секрет "в ENV" — кажется удобным...
docker run -e DB_PASSWORD=supersecret myapp

# Но:
docker inspect myapp | jq '.[0].Config.Env'
# → ["DB_PASSWORD=supersecret", "PATH=/usr/local/..."]
# Любой, у кого есть доступ к docker socket, видит секрет!

docker history myapp
# Если ENV прописан в Dockerfile → виден в истории образа

# В docker-compose logs → может попасть в логи CI/CD
```

> [!danger] ENV секреты утекают через
> - `docker inspect`
> - `docker history`
> - Логи CI/CD систем
> - `ps auxe` (переменные среды процесса)
> - `/proc/PID/environ`

---

## Уровни секретов

| Момент | Проблема | Решение |
|--------|---------|---------|
| **Сборка образа** | ARG/RUN с секретами → в слоях | BuildKit `--secret` |
| **Runtime** | ENV видно в inspect | Docker Secrets / файлы |
| **Хранение** | .env файлы в репо | External secret managers |
| **Передача** | Секрет в command args | Secret/file mount или API внешнего хранилища |

---

## Сборочные секреты — BuildKit `--secret`

Секрет доступен только во время конкретного `RUN`, **не записывается в слои**:

```dockerfile
# syntax=docker/dockerfile:1
FROM node:20-alpine AS build
WORKDIR /app

COPY package*.json ./

# Secret временно монтируется прямо в ожидаемый путь
RUN --mount=type=secret,id=npmrc,target=/root/.npmrc,required=true \
    npm ci

COPY . .
RUN npm run build
```

```bash
# Передать секрет при сборке
docker build \
  --secret id=npmrc,src=$HOME/.npmrc \
  -t myapp:latest .

# Из переменной окружения
docker build \
  --secret id=github_token,env=GITHUB_TOKEN \
  -t myapp:latest .

# Проверить что секрет не попал в образ
docker history myapp:latest          # не видно
docker run --rm myapp cat /run/secrets/npmrc 2>/dev/null || echo "нет!"
```

---

## Runtime: Docker Secrets (Swarm Mode)

Зашифрованное хранение секретов в Raft-логе Swarm-кластера.  
Монтируются как **tmpfs файлы** в `/run/secrets/<name>`.

```bash
# Инициализировать Swarm (если не сделано)
docker swarm init

# Создать секрет
docker secret create db_password ./db_password.txt
docker secret create ssl_cert ./server.crt
docker secret create app_config - << EOF
{
  "apiKey": "abc123",
  "region": "eu-west-1"
}
EOF

# Просмотр секретов (значения не показываются!)
docker secret ls
docker secret inspect db_password

# Использование в сервисе
docker service create \
  --name myapp \
  --secret db_password \
  --secret source=ssl_cert,target=server.crt,mode=0400 \
  myapp:latest

# Внутри контейнера сервиса:
# /run/secrets/db_password  ← файл с паролем
# /run/secrets/server.crt   ← файл с сертификатом

# Удалить секрет
docker secret rm db_password
```

> [!warning]
> Не вводи секрет прямо в аргументах команды: он может остаться в shell history
> или кратковременно быть виден в списке процессов. После создания секрета
> безопасно удали исходный файл согласно правилам своей системы.

```yaml
# compose.yaml для последующего docker stack deploy

services:
  app:
    image: myapp:latest
    secrets:
      - db_password
      - source: ssl_cert
        target: /etc/ssl/certs/server.crt
        mode: 0440
    environment:
      # Читаем из файла в точке входа, не из ENV
      DB_PASSWORD_FILE: /run/secrets/db_password

secrets:
  db_password:
    external: true        # уже создан через docker secret create
  ssl_cert:
    external: true
```

> [!note]
> Для Swarm файл разворачивается командой `docker stack deploy`, а не обычным
> `docker compose up`. Обычный Compose тоже поддерживает `secrets`, но локальный
> secret из `file:` — это файловое монтирование, а не зашифрованное Swarm-хранилище.

---

## Паттерн `_FILE` переменных

Многие официальные образы поддерживают `VAR_FILE`:

```bash
# Например, официальные образы PostgreSQL и MySQL поддерживают *_FILE:
docker service create \
  --secret pg_password \
  -e POSTGRES_PASSWORD_FILE=/run/secrets/pg_password \
  postgres:16

# Для своего приложения — entrypoint.sh паттерн:
```

```bash
#!/bin/sh
# entrypoint.sh

# Загрузить секреты из файлов в переменные окружения
if [ -f "$DB_PASSWORD_FILE" ]; then
  export DB_PASSWORD=$(cat "$DB_PASSWORD_FILE")
fi

if [ -f "$API_KEY_FILE" ]; then
  export API_KEY=$(cat "$API_KEY_FILE")
fi

# Запустить приложение
exec "$@"
```

```dockerfile
COPY entrypoint.sh /usr/local/bin/
RUN chmod +x /usr/local/bin/entrypoint.sh
ENTRYPOINT ["entrypoint.sh"]
CMD ["dotnet", "MyApp.dll"]
```

---

## Runtime без Swarm: secrets в Compose

```yaml
services:
  app:
    image: myapp:latest
    secrets:
      - db_config

secrets:
  db_config:
    file: ./secrets/db_config.json
```

Compose предоставит файл контейнеру как `/run/secrets/db_config`. Но в отличие
от Swarm это **не зашифрованное распределённое хранилище**: исходный файл остаётся
на хосте. Ограничь права (`chmod 600`), не добавляй каталог в Git и настрой
ротацию.

Если секрет вообще не должен храниться на диске хоста, приложение или доверенный
agent должен получать его из Vault/cloud secret manager через workload identity.
Универсальный «init-контейнер + общий tmpfs» в обычном Compose ненадёжен: tmpfs
привязан к mount namespace и не предназначен для совместного постоянного тома.

---

## Внешние системы управления секретами

### HashiCorp Vault

```bash
# Vault Agent Injector (Kubernetes) или sidecar паттерн
docker run -d \
  --name vault-agent \
  -v vault-config:/vault/config \
  -v app-secrets:/secrets \
  hashicorp/vault agent -config=/vault/config/agent.hcl

# agent.hcl
# template {
#   destination = "/secrets/db_password"
#   contents = "{{ with secret \"secret/myapp/db\" }}{{ .Data.password }}{{ end }}"
# }
```

### AWS Secrets Manager

```python
# В приложении (Python пример)
import boto3, json

def get_secret(secret_name):
    client = boto3.client('secretsmanager', region_name='eu-west-1')
    response = client.get_secret_value(SecretId=secret_name)
    return json.loads(response['SecretString'])

secrets = get_secret('prod/myapp/db')
DB_PASSWORD = secrets['password']
```

### .NET / ASP.NET Core

```csharp
// Program.cs — читать секреты из файла (Docker Secrets паттерн)
builder.Configuration.AddKeyPerFile(
    directoryPath: "/run/secrets",
    optional: true
);

// Для AWS / Azure / Vault используй официальный или проверенный provider-пакет.
// Точный extension method и схема ключей зависят от выбранного пакета.
```

---

## Сканирование секретов в образах

```bash
# truffleHog — поиск секретов в истории git и образах
trufflehog docker --image myapp:latest

# detect-secrets — при коммите
detect-secrets scan .

# Trivy — секреты + уязвимости
trivy image --scanners secret myapp:latest
```

---

## Антипаттерны — никогда так не делай

```dockerfile
# ❌ Секрет в ARG/RUN → может попасть в history, provenance, cache или логи
ARG NPM_TOKEN
RUN npm install --registry https://npm.pkg.github.com //...authToken=${NPM_TOKEN}

# ❌ Секрет в ENV → виден в docker inspect
ENV DB_PASSWORD=supersecret

# ❌ Секрет скопирован в образ
COPY .env /app/.env

# ❌ Секрет в имени файла или пути
COPY secrets/production.key /app/
```

---

## 📎 Связанные заметки

```dataview
LIST
FROM #docker
WHERE file.name != this.file.name
SORT file.name ASC
```
