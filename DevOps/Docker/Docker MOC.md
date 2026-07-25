---
tags:
  - docker
  - devops
  - moc
topic: docker
status: готово
created: 2026-06-24
---

# 🐳 Docker — Map of Content

> [!info] О коллекции
> Структурированная база знаний по Docker: от основ до продвинутых практик.
> DataView-таблицы строятся автоматически по frontmatter всех заметок с тегом `#docker`.

---

## 📚 Все заметки

```dataview
TABLE WITHOUT ID
  file.link AS "📄 Заметка",
  subtopic AS "Тема",
  difficulty AS "Уровень",
  status AS "Статус"
FROM #docker
WHERE file.name != this.file.name
SORT choice(difficulty = "начинающий", 1, choice(difficulty = "средний", 2, 3)) ASC, file.name ASC
```

---

## 🟢 Начинающий

```dataview
TABLE WITHOUT ID
  file.link AS "Заметка",
  subtopic AS "Тема",
  status AS "Статус"
FROM #docker
WHERE difficulty = "начинающий" AND file.name != this.file.name
SORT file.name ASC
```

## 🟡 Средний

```dataview
TABLE WITHOUT ID
  file.link AS "Заметка",
  subtopic AS "Тема",
  status AS "Статус"
FROM #docker
WHERE difficulty = "средний" AND file.name != this.file.name
SORT file.name ASC
```

## 🔴 Продвинутый

```dataview
TABLE WITHOUT ID
  file.link AS "Заметка",
  subtopic AS "Тема",
  status AS "Статус"
FROM #docker
WHERE difficulty = "продвинутый" AND file.name != this.file.name
SORT file.name ASC
```

---

## 🗺️ Карта тем

### 🧱 Основы
| Тема | Заметка |
|------|---------|
| Архитектура, концепции, жизненный цикл | [[Docker — Основы]] |
| Управление контейнерами | [[Docker — Контейнеры]] |
| Работа с образами и реестрами | [[Docker — Образы]] |
| Dockerfile — синтаксис и best practices | [[Docker — Dockerfile]] |
| Персистентное хранилище | [[Docker — Тома (Volumes)]] |
| Сетевые драйверы и коммуникация | [[Docker — Сеть]] |
| Многоконтейнерные приложения | [[Docker — Compose]] |

### ⚙️ Под капотом
| Тема | Заметка |
|------|---------|
| Namespaces, cgroups, OverlayFS, runc | [[Docker — Внутреннее устройство]] |
| BuildKit, cache mounts, SSH, bake | [[Docker — BuildKit и кэширование]] |
| Многоплатформенные образы (buildx) | [[Docker — Многоплатформенные образы]] |

### 🔐 Безопасность
| Тема | Заметка |
|------|---------|
| Пользователи, capabilities, seccomp, rootless | [[Docker — Пользователи и права]] |
| Управление секретами | [[Docker — Секреты]] |
| Анализ и сканирование образов | [[Docker — Анализ образов]] |

### 📊 Эксплуатация
| Тема | Заметка |
|------|---------|
| CPU, RAM, I/O, PID лимиты (cgroups) | [[Docker — Ресурсы и лимиты]] |
| Профилирование, диагностика, отладка | [[Docker — Профилирование и диагностика]] |

---

## 📊 Прогресс изучения

```dataview
TABLE WITHOUT ID
  file.link AS "Заметка",
  difficulty AS "Уровень",
  status AS "Статус",
  dateformat(date(created), "dd.MM.yyyy") AS "Добавлена"
FROM #docker
WHERE file.name != this.file.name
SORT choice(difficulty = "начинающий", 1, choice(difficulty = "средний", 2, 3)) ASC, file.name ASC
```

---

## 🎯 Путь обучения

### Фундамент (начинающий)
1. [[Docker — Основы]] — архитектура, ключевые понятия, изоляция
2. [[Docker — Контейнеры]] — жизненный цикл, команды, мониторинг
3. [[Docker — Образы]] — pull/push/build, слои, реестры
4. [[Docker — Dockerfile]] — синтаксис, multi-stage, .dockerignore
5. [[Docker — Тома (Volumes)]] — named volume, bind mount, tmpfs
6. [[Docker — Сеть]] — bridge, host, overlay, DNS
7. [[Docker — Compose]] — yaml, profiles, depends_on, healthcheck

### Углубление (продвинутый)
8. [[Docker — Внутреннее устройство]] — как Docker работает под капотом
9. [[Docker — BuildKit и кэширование]] — cache mounts, SSH, bake
10. [[Docker — Пользователи и права]] — безопасность, capabilities, rootless
11. [[Docker — Секреты]] — BuildKit secrets, Swarm, внешние хранилища
12. [[Docker — Анализ образов]] — dive, trivy, SBOM, signing
13. [[Docker — Ресурсы и лимиты]] — cgroups, OOM, production настройки
14. [[Docker — Профилирование и диагностика]] — nsenter, cAdvisor, perf
15. [[Docker — Многоплатформенные образы]] — buildx, multi-arch, CI/CD

---

## 🔖 Быстрые ссылки

- [Docker Docs](https://docs.docker.com)
- [Dockerfile Reference](https://docs.docker.com/reference/dockerfile/)
- [Compose Spec](https://docs.docker.com/compose/compose-file/)
- [BuildKit Docs](https://docs.docker.com/build/buildkit/)
- [OCI Specs](https://opencontainers.org)
- [dive](https://github.com/wagoodman/dive) — анализ слоёв образа
- [Trivy](https://trivy.dev) — сканирование уязвимостей
- [docker-bench-security](https://github.com/docker/docker-bench-security) — CIS Benchmark
