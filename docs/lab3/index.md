# Лабораторная работа 3: Упаковка FastAPI приложения в Docker, Работа с источниками данных и Очереди

## Описание
Эта лабораторная работа объединяет FastAPI приложение из Lr1 с парсерами из Lr2 и упаковывает все в Docker контейнеры с поддержкой асинхронных очередей через Celery и Redis.

## Реализованные задачи

### Подзадача 1: Упаковка в Docker
- **FastAPI приложение** упаковано в Docker контейнер
- **База данных PostgreSQL** запускается в отдельном контейнере
- **Парсер данных** упакован в отдельный контейнер
- **HTTP API для парсера** реализован с помощью FastAPI
- **Docker Compose** для оркестрации всех сервисов
- **Dockerfile** для каждого сервиса с оптимизацией

### Подзадача 2: Вызов парсера из FastAPI
- **Эндпоинт `/parse`** для синхронного вызова парсера
- **HTTP интеграция** между основным приложением и парсером
- **Обработка ошибок** и таймаутов
- **Валидация данных** с помощью Pydantic

### Подзадача 3: Асинхронные очереди с Celery
- **Celery** для асинхронной обработки задач
- **Redis** как брокер сообщений
- **Эндпоинт `/parse/async`** для асинхронного парсинга
- **Мониторинг статуса задач** через API
- **Celery Worker** в отдельном контейнере

## Архитектура системы

```
┌─────────────────┐    HTTP запросы     ┌─────────────────┐
│   Client        │ ──────────────────► │   FastAPI App   │ :8000
│   (Browser/API) │                     │   (Task Mgmt)   │
└─────────────────┘                     └─────────┬───────┘
                                                  │ HTTP
                                                  ▼
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   PostgreSQL    │    │   Parser API    │    │   Redis         │
│   (Database)    │◄──►│   (GitHub)      │◄──►│   (Broker)      │
│   :5432         │    │   :8001         │    │   :6379         │
└─────────────────┘    └─────────┬───────┘    └─────────┬───────┘
                                 │                      │
                                 ▼                      ▼
                       ┌─────────────────┐    ┌─────────────────┐
                       │   SQLite        │    │  Celery Worker  │
                       │   (Parser DB)   │    │  (Async Tasks)  │
                       └─────────────────┘    └─────────────────┘
```

## Структура проекта
```
Lr3/
├── app/                    # FastAPI приложение (из Lr1)
│   ├── db/                 # Модели базы данных
│   │   ├── __init__.py
│   │   ├── database.py     # Подключение к БД
│   │   └── models.py       # SQLAlchemy модели
│   ├── rest/               # REST API эндпоинты
│   │   ├── task/           # API для задач
│   │   ├── sprint/         # API для спринтов
│   │   └── task_link/      # API для связей задач
│   ├── config.py           # Конфигурация приложения
│   └── main.py             # Основное приложение
├── parser/                 # Парсер приложение (из Lr2)
│   ├── parsers/            # Парсеры данных
│   │   ├── __init__.py
│   │   ├── config.py       # Конфигурация парсера
│   │   ├── db_models.py    # Модели для парсера
│   │   └── async_parser.py # Асинхронный парсер
│   ├── main.py             # HTTP API для парсера
│   └── celery_app.py       # Celery конфигурация
├── docker-compose.yml      # Оркестрация сервисов
├── Dockerfile.app          # Docker образ для FastAPI
├── Dockerfile.parser       # Docker образ для парсера
├── requirements.txt        # Зависимости
├── env.example             # Пример переменных окружения
├── Makefile               # Команды для управления
├── run_tests.py           # Скрипт тестирования
├── SETUP.md               # Инструкция по настройке
└── README.md              # Этот файл
```

## Сервисы
1. **app** - FastAPI приложение для управления задачами и спринтами
2. **parser** - HTTP API для парсинга GitHub issues
3. **celery-worker** - Воркер для асинхронной обработки задач
4. **redis** - Брокер сообщений для Celery
5. **postgres** - База данных PostgreSQL

## Быстрый старт

### 1. Настройка окружения
```bash
# Скопируйте пример переменных окружения
cp env.example .env

# Отредактируйте .env при необходимости
# Особенно важно установить GITHUB_TOKEN
```

### 2. Запуск всех сервисов
```bash
# Используя Makefile
make up-build

# Или напрямую
docker-compose up -d --build
```

### 3. Проверка работоспособности
```bash
# Автоматические тесты
make test

# Ручная проверка
curl http://localhost:8000/health
curl http://localhost:8001/health
```

## API Endpoints

### FastAPI приложение (порт 8000)
- `GET /` - Главная страница с информацией об API
- `GET /health` - Проверка состояния сервиса
- `GET /tasks/` - Получить все задачи
- `POST /tasks/` - Создать задачу
- `GET /tasks/{id}` - Получить задачу по ID
- `PATCH /tasks/{id}` - Обновить задачу
- `DELETE /tasks/{id}` - Удалить задачу
- `GET /sprints/` - Получить все спринты
- `POST /sprints/` - Создать спринт
- `GET /sprints/{id}` - Получить спринт по ID
- `PATCH /sprints/{id}` - Обновить спринт
- `DELETE /sprints/{id}` - Удалить спринт
- `POST /parse` - Синхронный вызов парсера
- `POST /parse/async` - Асинхронный вызов парсера

### Парсер API (порт 8001)
- `GET /` - Информация о парсере
- `GET /health` - Проверка состояния парсера
- `POST /parse` - Синхронный парсинг репозиториев
- `POST /parse/async` - Асинхронный парсинг через Celery
- `GET /task/{task_id}` - Статус асинхронной задачи

## Примеры использования

### Синхронный парсинг
```bash
curl -X POST "http://localhost:8001/parse" \
  -H "Content-Type: application/json" \
  -d '{"repositories": ["python/cpython"]}'
```

### Асинхронный парсинг
```bash
# Запуск задачи
curl -X POST "http://localhost:8001/parse/async" \
  -H "Content-Type: application/json" \
  -d '{"repositories": ["python/cpython", "django/django"]}'

# Проверка статуса (замените TASK_ID)
curl -X GET "http://localhost:8001/task/TASK_ID"
```

### Вызов парсера из основного приложения
```bash
# Синхронный
curl -X POST "http://localhost:8000/parse" \
  -H "Content-Type: application/json" \
  -d '{"repositories": ["python/cpython"]}'

# Асинхронный
curl -X POST "http://localhost:8000/parse/async" \
  -H "Content-Type: application/json" \
  -d '{"repositories": ["django/django"]}'
```

### Создание задачи
```bash
curl -X POST "http://localhost:8000/tasks/" \
  -H "Content-Type: application/json" \
  -d '{
    "summary": "Implement new feature",
    "priority": "major",
    "description": "Add user authentication",
    "planned_end_at": "2024-01-15T12:00:00",
    "status": "open",
    "sprint_id": null
  }'
```

## Технологии

### Backend
- **FastAPI** - современный веб-фреймворк для Python
- **SQLAlchemy** - ORM для работы с базой данных
- **Pydantic** - валидация данных и сериализация
- **Alembic** - миграции базы данных
- **aiohttp** - асинхронные HTTP запросы

### Асинхронность и очереди
- **Celery** - распределенная очередь задач
- **Redis** - брокер сообщений и кэш

### База данных
- **PostgreSQL** - реляционная база данных

### Контейнеризация
- **Docker** - контейнеризация приложений
- **Docker Compose** - оркестрация контейнеров

### Интеграции
- **GitHub API** - парсинг issues из репозиториев
- **HTTP API** - взаимодействие между сервисами

## Мониторинг и отладка

### Логи
```bash
# Все сервисы
make logs

# Конкретный сервис
make logs-app
make logs-parser
make logs-celery
```

### Подключение к сервисам
```bash
# База данных
make db-shell

# Redis
make redis-cli

# Контейнеры
make shell-app
make shell-parser
```

### Swagger UI
- FastAPI App: http://localhost:8000/docs
- Parser API: http://localhost:8001/docs

## Команды управления

Все команды доступны через Makefile:

```bash
make help          # Показать все доступные команды
make up-build       # Собрать и запустить все сервисы
make down           # Остановить все сервисы
make restart        # Перезапустить сервисы
make logs           # Показать логи
make test           # Запустить тесты
make clean          # Очистить Docker ресурсы
make status         # Показать статус сервисов
```
