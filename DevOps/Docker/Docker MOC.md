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
> Структурированная коллекция заметок по Docker — от основ до продвинутых практик.
> Используй **путь обучения** ниже или DataView-таблицы для навигации.

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
SORT file.name ASC
```

---

## 📊 Прогресс изучения

```dataview
TABLE WITHOUT ID
  file.link AS "Заметка",
  status AS "Статус",
  dateformat(date(created), "dd.MM.yyyy") AS "Добавлена"
FROM #docker
WHERE file.name != this.file.name
SORT status ASC
```

---

## 🗺️ Карта тем

| Блок | Заметка | Уровень |
|------|---------|---------|
| 🧠 Концепции | [[Docker — Основы]] | Начинающий |
| 🔧 Сборка | [[Docker — Dockerfile]] | Средний |
| 🖼️ Образы | [[Docker — Образы]] | Начинающий |
| 📦 Контейнеры | [[Docker — Контейнеры]] | Начинающий |
| 💾 Хранилище | [[Docker — Тома (Volumes)]] | Средний |
| 🌐 Сеть | [[Docker — Сеть]] | Средний |
| 🚀 Оркестрация | [[Docker — Compose]] | Средний |

---

## 🎯 Путь обучения

```dataview
TABLE WITHOUT ID
  file.link AS "Шаг",
  difficulty AS "Уровень",
  status AS "Статус"
FROM #docker
WHERE file.name != this.file.name
SORT difficulty ASC, file.name ASC
```

### Рекомендованный порядок

1. [[Docker — Основы]] — архитектура, ключевые понятия
2. [[Docker — Контейнеры]] — жизненный цикл и управление
3. [[Docker — Образы]] — работа с образами и реестрами
4. [[Docker — Dockerfile]] — создание собственных образов
5. [[Docker — Тома (Volumes)]] — персистентное хранилище
6. [[Docker — Сеть]] — сетевая изоляция и коммуникация
7. [[Docker — Compose]] — многоконтейнерные приложения

---

## 🔖 Быстрые ссылки

- [Docker Hub](https://hub.docker.com) — официальный реестр образов
- [Docker Docs](https://docs.docker.com) — официальная документация
- [Dockerfile Reference](https://docs.docker.com/engine/reference/builder/)
- [Compose Reference](https://docs.docker.com/compose/compose-file/)
