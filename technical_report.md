# Техническая документация проекта

## Обзор

Проект представляет собой инфраструктуру для автоматизации TikTok: Telegram-боты (лайки, комментарии), загрузки видео, а также веб-сайт (Terminal UI) с системой подписки, платежами и синхронизацией через sync-ключи. Все компоненты работают в единой Docker-сети с общей PostgreSQL.

-------

## 1. Techo Global (tg8378.xyz)

### Назначение
Веб-сайт в стиле Linux Terminal. Предоставляет:
- Публичное портфолио (Explorer Mode — about, skills, projects, contact)
- Система подписки на TikTok ботов (планы, платежи через CryptoBot)
- Активация sync-ключей (Member Mode — dashboard, subscriptions)
- Панель администратора (Admin Mode — управление пользователями, подписками, контентом)
- Загрузка ZIP-архивов с видео для публикации в TikTok

### Стек технологий

| Компонент | Технология |
|---|---|
| **Фронтенд** | Vanilla JS, CSS3, HTML5 (без фреймворков), Vite |
| **Бэкенд** | Python 3.11, FastAPI (async), Uvicorn |
| **База данных** | PostgreSQL 16 (asyncpg) |
| **ORM** | SQLAlchemy 2.0 (async), Alembic |
| **Аутентификация** | JWT (python-jose + cryptography) |
| **Рейт-лимитинг** | slowapi |
| **Реалтайм** | WebSocket, python-socketio |
| **Линтер** | Ruff, Black |
| **Пакетный менеджер** | Poetry (backend), npm (frontend) |
| **Docker** | Multi-stage: frontend (node → nginx), backend (python:3.11-slim) |

### Модули бэкенда (`backend/app/`)

- `api/auth.py` — активация sync-ключей, JWT, admin-login, refresh
- `api/user.py` — `/me`, `/subscriptions`
- `api/plans.py` — список тарифов
- `api/payment.py` — создание платежа, callback от CryptoBot
- `api/upload.py` — загрузка ZIP, валидация, список видео
- `api/admin.py` — CRUD пользователей, управление подписками, контентом

### Тестирование

| Тип | Фреймворк | Кол-во |
|---|---|---|
| **Модульные (backend)** | pytest 7.4 + pytest-asyncio | 44 теста |
| **E2E** | Playwright 1.52 (TypeScript, Chromium) | 21 тест |

E2E тесты используют отдельную БД (`tiktok_bot_test`). Seed: Python-скрипт `e2e_setup.py` (asyncpg).

### Docker

- `docker-compose.yml` — dev: backend:8000, frontend:8080
- `docker/docker-compose.yml` — production (с external bot-network)
- `e2e/docker-compose.e2e.yml` — E2E runner (Playwright + seed)

### CI/CD
- Отсутствует (ручной деплой через `deploy.sh` на VPS)

-------

## 2. tiktok-bot-ready (Telegram Bot — Automation)

### Назначение
Telegram-бот для автоматизации действий в TikTok: массовые лайки, комментарии. Пользователь создаёт задачу через Telegram, бот исполняет её через Playwright (браузерную автоматизацию) с использованием пула аккаунтов и прокси.

### Стек технологий

| Компонент | Технология |
|---|---|
| **Язык** | Python 3.14 |
| **Telegram Bot** | aiogram 3.5+ |
| **Браузер** | Playwright 1.40+ (Chromium) |
| **База данных** | PostgreSQL 16 (asyncpg 0.29+) |
| **Конфигурация** | pydantic-settings (.env + runtime override из БД) |
| **Логирование** | loguru |
| **Прокси** | Поддержка HTTP/HTTPS/SOCKS5, ротация, пул |
| **Docker** | python:3.14-slim, multi-stage |
| **Линтер** | ruff |

### Архитектура

```
run.py → Database.init() → TaskExecutor.start_scheduler() → BotApp.start()
         │                        │
         │                   Пулл pending-задач из БД каждые 5 с
         │                        │
         │                   _select_account() → AccountManager
         │                        │
         │                   BrowserManager (Playwright, proxy, cookies)
         │                        │
         │                   Выполнение: search → like / comment / follow
         │                        │
         │                   Запись результатов в reports
         │
         └─── Telegram UI (handlers/): accounts, scheduler, reports, settings
```

### Основные модули (`backhand/`)

- `executor.py` — TaskExecutor: планировщик, диспетчеризация задач
- `browser.py` — BrowserManager: Playwright persistent context, anti-detection
- `browser_pool.py` — UserBrowserPool: семафор для ограничения параллельных браузеров
- `cookies.py` — CookieStore: сохранение/загрузка cookies
- `accounts/manager.py` — AccountManager: выбор аккаунта, cooldown, прокси
- `accounts/cache.py` — AccountCache: TTL-кэш в памяти
- `accounts/proxy.py` — ProxyManager: список прокси с round-robin
- `accounts/proxy_pool.py` — GlobalProxyPool: единый пул для всех пользователей
- `tiktok/search.py` — Поиск видео через TikTok API (page.evaluate + fetch)
- `tiktok/likes.py` — Лайки через API + DOM
- `tiktok/comments.py` — Комментарии с captcha-handling
- `tiktok/follow.py` — Подписки
- `tiktok/auto_comment.py` — Многоаккаунтный массовый комментарий
- `tiktok/auto_search_comment.py` — Поиск → комментарий

### Тестирование

| Тип | Кол-во |
|---|---|
| **pytest + pytest-asyncio** | 130+ тестов |

Тесты: utils, database (CRUD), cache, comments, follow, likes, proxy, search. Используют тестовую БД (`techo-db-test:5432/tiktok_bot_test`).

### Docker

- `Dockerfile` — multi-stage: builder (gcc → wheels) + runtime (playwright install chromium)
- `docker-compose.yml` — сервис `bot` (container_name: `techo-bot`)
- Ограничение памяти: 2GB
- Volume mounts: database.py, config.py, models.py (hot-reload)
- Сеть: `techo-network` (external)

### CI/CD
- Makefile: `make test` (запуск тестов внутри контейнера)
- GitHub Actions отсутствуют

-------

## 3. tiktok-video (Telegram Bot — Video Upload)

### Назначение
Telegram-бот для автоматической загрузки видео в TikTok. Пользователь отправляет ZIP-архив с видео, бот извлекает, валидирует и публикует их на TikTok через автоматизированный браузер. Поддерживаются два режима публикации: по одному на аккаунт (round-robin) и бесконечная очередь.

### Стек технологий

| Компонент | Технология |
|---|---|
| **Язык** | Python 3.14 |
| **Telegram Bot** | aiogram 3.5+ |
| **Браузер** | Playwright 1.40+ (Chromium) |
| **База данных** | PostgreSQL 16 (asyncpg 0.29+) |
| **Конфигурация** | pydantic-settings |
| **Логирование** | loguru |
| **Обработка видео** | ffmpeg (встроен в Docker-образ) |
| **Docker** | python:3.14-slim, multi-stage |
| **Линтер** | ruff |

### Архитектура

```
run.py → Database.init() → BotApp.start() + background cleaners
         │                        │
         │                   /upload → ZIP validation → session → confirm
         │                        │
         │                   VideoUploadExecutor
         │                        │
         │                   AccountManager → pick account + proxy
         │                        │
         │                   BrowserManager (Playwright, login by cookies)
         │                        │
         │                   upload_api.py (XHR/fetch) → fallback → upload_dom.py (file input)
         │                        │
         │                   DailyLimitChecker → CaptchaHandler → публикация
         │
         └─── Telegram UI (handlers/): upload, videos, processes, posting, accounts, proxy, settings
```

### Основные модули

- `backhand/tiktok/upload.py` — VideoUploadExecutor: основной оркестратор загрузки
- `backhand/tiktok/upload_api.py` — API-загрузка (XHR/fetch через браузер)
- `backhand/tiktok/upload_dom.py` — DOM-загрузка (file input → форма → post)
- `backhand/tiktok/captcha.py` — CaptchaHandler (обнаружение и ожидание)
- `backhand/tiktok/limits.py` — DailyLimitChecker (суточные лимиты TikTok)
- `backhand/browser.py` — BrowserManager (запуск, куки, anti-detection)
- `backhand/sanitizer.py` — Валидация входных данных
- `website_upload_endpoint.py` — Опциональный FastAPI-endpoint для веб-загрузки ZIP

### Тестирование

| Тип |                                |Кол-во |
| **pytest + pytest-asyncio** | 200 тестов, 14 файлов |

Тесты: database, handlers (posting, upload, start), browser, upload (executor, API, DOM), captcha, cache, proxy, limits, sanitizer, models. Используют тестовую БД.

### Docker

- `Dockerfile` — multi-stage: builder + runtime (python:3.14-slim, ffmpeg, playwright chromium)
- `docker-compose.yml` — сервис `video-bot` (container_name: `tiktok-video-bot`)
- Ограничение памяти: 2GB
- Volume: `uploads_data` shared
- Сеть: `techo-network`

### CI/CD
- Makefile: `make test`
- GitHub Actions отсутствуют

-------

## Инфраструктура (общая)

### Docker-сеть
- **Сеть**: `techo-network` (external, bridge)
- **PostgreSQL**: `techo-db:5432` (production: `tiktok_bot`), `techo-db-test:5432` (test: `tiktok_bot_test`)

### Контейнеры

| Имя | Сервис | Порт |
|---|---|---|
| `techo-backend` | FastAPI | 8000 |
| `techo-frontend` | nginx (static) | 8080 |
| `techo-bot` | tiktok-bot-ready | — |
| `tiktok-video-bot` | tiktok-video | — |
| `techo-db` | PostgreSQL 16 (prod) | 5432 |
| `techo-db-test` | PostgreSQL 16 (test) | 5433 |
| `techo-e2e` | Playwright (запускается по требованию) | — |

### Единая БД
Все проекты используют одну production БД (`tiktok_bot` на `techo-db:5432`). Sync-ключи создаются ботами, активируются на сайте. Таблицы создаются ботами (через `CREATE TABLE IF NOT EXISTS`) и бэкендом (через SQLAlchemy `create_all`).

### Тестирование
- Backend: `docker exec techo-backend python -m pytest backend/tests/ -v`
- tiktok-bot-ready: `make test` (внутри контейнера techo-bot)
- tiktok-video: `docker exec tiktok-video-bot python -m pytest -q --tb=short`
- E2E: `npx playwright test` (через `e2e/docker-compose.e2e.yml`)
- Все тесты запускаются внутри Docker-контейнеров (asyncpg + Windows несовместимы)
- Тесты используют `tiktok_bot_test`, production не затрагивают

### Итого по тестам
- **Backend**: 44
- **tiktok-bot-ready**: 135
- **tiktok-video**: 200
- **E2E**: 21
- **Всего**: 400 тестов
