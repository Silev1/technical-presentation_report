# Платформа автоматизации рекламных процессов в TikTok

## Общая концепция

Комплексное решение для управления рекламными активностями в TikTok: публикация контента, управление вовлечённостью, мониторинг аккаунтов — всё через единый веб-интерфейс и Telegram-ботов. Система состоит из веб-сайта (личный кабинет) и двух Telegram-ботов, объединённых общей базой данных и единой системой авторизации (sync-ключи).

-------

## Веб-сайт (Личный кабинет)

**Назначение:** Центральная точка входа для клиентов и администраторов. Terminal-style интерфейс, напоминающий работу в командной строке.

**Возможности:**
- Регистрация и активация доступа через sync-ключ
- Просмотр статуса подписки и активных услуг (Dashboard)
- Выбор тарифного плана и оплата через CryptoBot
- Панель администратора: управление пользователями, подписками, контентом сайта
- Загрузка видео через веб-интерфейс

**Технологии:**
- Фронтенд: Vanilla JS (без фреймворков), Vite
- Бэкенд: Python 3.11, FastAPI
- База данных: PostgreSQL 16
- Аутентификация: JWT

-------

## Telegram Bot auto coments — Управление контентом и вовлечённостью

**Назначение:** Автоматизация процессов публикации контента и управления взаимодействием с аудиторией TikTok.

**Возможности:**
- Публикация комментариев от имени нескольких аккаунтов
- Автоматизация процессов лайков и подписов рамках рекламных кампаний
- Поиск целевой аудитории по ключевым словам с автоматическим взаимодействием
- Управление пулом аккаунтов и прокси для распределения нагрузки
- Гибкое планирование задач с настраиваемыми задержками

**Технологии:**
- Python 3.14, aiogram 3.5 (Telegram Bot API)
- Playwright (автоматизация браузера Chromium)
- PostgreSQL 16, asyncpg
- Поддержка прокси (HTTP/HTTPS/SOCKS5) с ротацией
- Pydantic для конфигурации

-------

## Telegram Bot auto video — Публикация видео

**Назначение:** Автоматизированная загрузка и публикация видеоконтента в TikTok.

**Возможности:**
- Пакетная загрузка видео (ZIP-архив через Telegram или веб-интерфейс)
- Автоматическая публикация на нескольких аккаунтах по расписанию
- Два режима: round-robin (по одному видео на аккаунт) и бесконечная очередь
- Контроль суточных лимитов TikTok
- Обработка капчи при публикации
- Автоматическое переключение между API и DOM-загрузкой

**Технологии:**
- Python 3.14, aiogram 3.5
- Playwright (Chromium)
- PostgreSQL 16, asyncpg
- ffmpeg для обработки видео
- Поддержка прокси с ротацией

-------

## Инфраструктура

**Развёртывание:** Docker-контейнеры, объединённые единой сетью.

**Компоненты:**
- `techo-backend` — API сервер (FastAPI)
- `techo-frontend` — веб-интерфейс (nginx)
- `techo-bot` — бот управления контентом
- `tiktok-video-bot` — бот публикации видео
- `techo-db` — база данных PostgreSQL (production)

**Единая система авторизации:** Sync-ключи — одноразовые коды, генерируемые в Telegram-ботах и активируемые на сайте. Обеспечивают бесшовную интеграцию между ботами и личным кабинетом.

-------

## Тестирование

Полный цикл тестирования: 400 автоматических тестов.

- Backend (44 теста) — API, авторизация, платежи, загрузка
- Bot управления контентом (135 тестов) — БД, алгоритмы, работа с браузером
- Bot публикации видео (200 тестов) — загрузка, обработка, лимиты
- E2E (21 тест) — полный сценарий: регистрация → активация → работа в личном кабинете

-------

## Структура базы данных

Единая PostgreSQL (16), общая для всех компонентов системы.

### 1. `users` — Пользователи

| Колонка | Тип | Ограничения | Описание |
|---------|-----|-------------|---------|
| `id` | `SERIAL` | PK | Уникальный ID пользователя в системе |
| `telegram_id` | `BIGINT` | UNIQUE, NOT NULL | Telegram ID пользователя |
| `username` | `TEXT` | | Username в Telegram |
| `is_admin` | `INTEGER` | DEFAULT 0 | Флаг администратора (0/1) |
| `created_at` | `TIMESTAMPTZ` | DEFAULT NOW() | Дата регистрации |
| `peak_account_count` | `INTEGER` | DEFAULT 0 | Макс. количество аккаунтов |
| `last_seen` | `TIMESTAMP` | | Последняя активность |

### 2. `accounts` — Аккаунты TikTok

| Колонка | Тип | Ограничения | Описание |
|---------|-----|-------------|---------|
| `id` | `SERIAL` | PK | Уникальный ID аккаунта |
| `user_id` | `INTEGER` | FK → `users(id)` ON DELETE CASCADE | Владелец аккаунта |
| `tiktok_username` | `TEXT` | NOT NULL | Username в TikTok |
| `cookies` | `TEXT` | DEFAULT '' | Cookies сессии (JSON) |
| `proxy` | `TEXT` | | Прокси для аккаунта |
| `is_active` | `INTEGER` | DEFAULT 1 | Активен (0/1) |
| `created_at` | `TIMESTAMP` | DEFAULT NOW() | Дата добавления |
| `last_used` | `TIMESTAMP` | | Последнее использование |
| `proxy_pool` | `TEXT` | | Пул прокси для аккаунта |
| `upload_capable` | `INTEGER` | DEFAULT 0 | Доступна загрузка видео (0/1) |

### 3. `subscriptions` — Подписки

| Колонка | Тип | Ограничения | Описание |
|---------|-----|-------------|---------|
| `id` | `SERIAL` | PK | Уникальный ID подписки |
| `user_id` | `INTEGER` | FK → `users(id)` | Владелец подписки |
| `plan` | `TEXT` | NOT NULL | Slug тарифа (mini, standard, premium, mix_...) |
| `account_limit` | `INTEGER` | NOT NULL | Лимит аккаунтов по тарифу |
| `price` | `NUMERIC(10,2)` | NOT NULL | Цена |
| `payment_method` | `TEXT` | DEFAULT 'crypto_bot' | Способ оплаты |
| `payment_id` | `TEXT` | | ID платежа во внешней системе |
| `expires_at` | `TIMESTAMPTZ` | | Дата истечения |
| `is_active` | `INTEGER` | DEFAULT 1 | Активна (0/1) |
| `created_at` | `TIMESTAMPTZ` | DEFAULT NOW() | Дата создания |
| `notified_at` | `TIMESTAMPTZ` | | Дата уведомления об истечении |
| `bot_type` | `TEXT` | DEFAULT 'main' | Тип бота (main/video/mix) |

### 4. `sync_keys` — Ключи синхронизации

| Колонка | Тип | Ограничения | Описание |
|---------|-----|-------------|---------|
| `id` | `SERIAL` | PK | Уникальный ID ключа |
| `user_id` | `INTEGER` | FK → `users(id)` | Владелец ключа |
| `key` | `TEXT` | UNIQUE, NOT NULL | Значение ключа (XXXX-XXXX) |
| `is_active` | `INTEGER` | DEFAULT 0 | Активирован на сайте (0/1) |
| `expires_at` | `TIMESTAMPTZ` | | Срок действия (15 мин) |
| `created_at` | `TIMESTAMPTZ` | DEFAULT NOW() | Дата генерации |

### 5. `tasks` — Задачи (бот управления контентом)

| Колонка | Тип | Ограничения | Описание |
|---------|-----|-------------|---------|
| `id` | `TEXT` | PK | UUID задачи |
| `user_id` | `INTEGER` | FK → `users(id)` ON DELETE CASCADE | Владелец задачи |
| `task_type` | `TEXT` | NOT NULL | Тип (comment, search, auto_comment и т.д.) |
| `target` | `TEXT` | NOT NULL | Цель (ссылка, поисковый запрос) |
| `comment_text` | `TEXT` | | Текст комментария |
| `count` | `INTEGER` | DEFAULT 1 | Количество действий |
| `status` | `TEXT` | DEFAULT 'pending' | Статус (pending/running/done/error) |
| `created_at` | `TIMESTAMP` | DEFAULT NOW() | Дата создания |
| `finished_at` | `TIMESTAMP` | | Дата завершения |
| `error` | `TEXT` | | Текст ошибки |

### 6. `reports` — Отчёты о выполнении задач

| Колонка | Тип | Ограничения | Описание |
|---------|-----|-------------|---------|
| `id` | `SERIAL` | PK | Уникальный ID отчёта |
| `task_id` | `TEXT` | FK → `tasks(id)` ON DELETE CASCADE | ID задачи |
| `account_id` | `INTEGER` | FK → `accounts(id)` ON DELETE CASCADE | Аккаунт исполнитель |
| `action` | `TEXT` | NOT NULL | Действие (comment, like, follow) |
| `target_url` | `TEXT` | NOT NULL | Целевая ссылка |
| `result` | `TEXT` | NOT NULL | Результат (success/error) |
| `message` | `TEXT` | | Детали результата |
| `created_at` | `TIMESTAMP` | DEFAULT NOW() | Дата отчёта |

### 7. `upload_sessions` — Сессии загрузки видео

| Колонка | Тип | Ограничения | Описание |
|---------|-----|-------------|---------|
| `id` | `SERIAL` | PK | Уникальный ID сессии |
| `user_id` | `INTEGER` | FK → `users(id)` | Владелец сессии |
| `zip_path` | `TEXT` | NOT NULL | Путь к ZIP-архиву |
| `zip_size_bytes` | `BIGINT` | DEFAULT 0 | Размер архива |
| `total_videos` | `INTEGER` | DEFAULT 0 | Количество видео |
| `description_text` | `TEXT` | DEFAULT '' | Описание видео |
| `hashtags_text` | `TEXT` | DEFAULT '' | Хэштеги |
| `status` | `TEXT` | DEFAULT 'pending' | Статус сессии |
| `progress_text` | `TEXT` | DEFAULT '' | Текст прогресса |
| `created_at` | `TIMESTAMP` | DEFAULT NOW() | Дата создания |
| `finished_at` | `TIMESTAMP` | | Дата завершения |

### 8. `videos` — Видео

| Колонка | Тип | Ограничения | Описание |
|---------|-----|-------------|---------|
| `id` | `SERIAL` | PK | Уникальный ID видео |
| `session_id` | `INTEGER` | FK → `upload_sessions(id)` | Сессия загрузки |
| `user_id` | `INTEGER` | FK → `users(id)` | Владелец |
| `original_name` | `TEXT` | NOT NULL | Имя файла в архиве |
| `tiktok_video_id` | `TEXT` | | ID видео в TikTok |
| `tiktok_url` | `TEXT` | | Ссылка на TikTok |
| `description` | `TEXT` | DEFAULT '' | Описание |
| `hashtags` | `TEXT` | DEFAULT '' | Хэштеги |
| `status` | `TEXT` | DEFAULT 'queued' | Статус (queued/uploading/published/error) |
| `error` | `TEXT` | | Ошибка публикации |
| `account_id` | `INTEGER` | FK → `accounts(id)` | Аккаунт публикации |
| `attempts` | `INTEGER` | DEFAULT 0 | Попыток публикации |
| `created_at` | `TIMESTAMP` | DEFAULT NOW() | Дата добавления |
| `published_at` | `TIMESTAMP` | | Дата публикации |
| `file_size_bytes` | `BIGINT` | DEFAULT 0 | Размер файла |

### 9. `daily_limits` — Суточные лимиты

| Колонка | Тип | Ограничения | Описание |
|---------|-----|-------------|---------|
| `id` | `SERIAL` | PK | Уникальный ID |
| `account_id` | `INTEGER` | FK → `accounts(id)` | Аккаунт |
| `upload_date` | `DATE` | NOT NULL, UNIQUE(account_id, upload_date) | Дата |
| `upload_count` | `INTEGER` | DEFAULT 0 | Загружено видео |

### 10. `proxies` — Прокси

| Колонка | Тип | Ограничения | Описание |
|---------|-----|-------------|---------|
| `id` | `SERIAL` | PK | Уникальный ID |
| `user_id` | `INTEGER` | FK → `users(id)` ON DELETE CASCADE | Владелец |
| `proxy` | `TEXT` | NOT NULL | Строка прокси (host:port:user:pass) |
| `is_active` | `INTEGER` | DEFAULT 1 | Активен (0/1) |
| `created_at` | `TIMESTAMP` | DEFAULT NOW() | Дата добавления |

### 11. `config_variables` — Конфигурационные переменные

| Колонка | Тип | Ограничения | Описание |
|---------|-----|-------------|---------|
| `key` | `TEXT` | PK | Название переменной |
| `value` | `TEXT` | NOT NULL | Значение |
| `value_type` | `TEXT` | DEFAULT 'str' | Тип (str/int/float/bool/json_list) |
| `category` | `TEXT` | DEFAULT 'general' | Категория |
| `description` | `TEXT` | DEFAULT '' | Описание |
| `is_secret` | `INTEGER` | DEFAULT 0 | Секретное (маскировать) |
| `updated_by` | `INTEGER` | FK → `users(id)` ON DELETE SET NULL | Кто изменил |
| `updated_at` | `TIMESTAMP` | DEFAULT NOW() | Дата изменения |

### 12. `user_comments` — Комментарии пользователей

| Колонка | Тип | Ограничения | Описание |
|---------|-----|-------------|---------|
| `id` | `SERIAL` | PK | Уникальный ID |
| `user_id` | `INTEGER` | FK → `users(id)` ON DELETE CASCADE | Владелец |
| `comment_text` | `TEXT` | NOT NULL | Текст |
| `created_at` | `TIMESTAMP` | DEFAULT NOW() | Дата создания |

### 13. `account_rotation` — Ротация аккаунтов

| Колонка | Тип | Ограничения | Описание |
|---------|-----|-------------|---------|
| `user_id` | `INTEGER` | PK, FK → `users(id)` ON DELETE CASCADE | Пользователь |
| `last_account_index` | `INTEGER` | DEFAULT 0 | Индекс последнего аккаунта |

### 14. `processed_videos` — Обработанные видео (кэш)

| Колонка | Тип | Ограничения | Описание |
|---------|-----|-------------|---------|
| `url` | `TEXT` | PK | URL видео |
| `processed_at` | `TIMESTAMP` | DEFAULT NOW() | Дата обработки |

### 15. `site_content` — Контент сайта (CMS)

| Колонка | Тип | Ограничения | Описание |
|---------|-----|-------------|---------|
| `id` | `SERIAL` | PK | Уникальный ID |
| `section` | `TEXT` | UNIQUE, NOT NULL | Раздел сайта |
| `content` | `JSONB` | NOT NULL, DEFAULT '{}' | Содержимое (JSON) |
| `updated_at` | `TIMESTAMP` | DEFAULT NOW() | Дата обновления |

### 16. `alembic_version` — Миграции

| Колонка | Тип | Ограничения | Описание |
|---------|-----|-------------|---------|
| `version_num` | `VARCHAR(32)` | PK | Номер версии миграции |

-------

### Схема связей (FK)

```
users ──┬── accounts (user_id)
        ├── subscriptions (user_id)
        ├── sync_keys (user_id)
        ├── tasks (user_id)
        ├── upload_sessions (user_id)
        ├── videos (user_id)
        ├── proxies (user_id)
        ├── user_comments (user_id)
        ├── account_rotation (user_id)
        ├── config_variables (updated_by)
        │
accounts ──┬── daily_limits (account_id)
           ├── reports (account_id)
           ├── videos (account_id)
           │
tasks ──┬── reports (task_id)
        │
upload_sessions ──┬── videos (session_id)
```







### Сводка по таблицам

| # | Таблица | Назначение | Строк |
|---|---------|-----------|-------|
| 1 | `users` | Пользователи системы | ядро |
| 2 | `accounts` | Аккаунты TikTok | ядро |
| 3 | `subscriptions` | Подписки и тарифы | ядро |
| 4 | `sync_keys` | Ключи синхронизации бот↔сайт | ядро |
| 5 | `tasks` | Задачи автоматизации | бот комментов |
| 6 | `reports` | Отчёты выполнения задач | бот комментов |
| 7 | `upload_sessions` | Сессии загрузки видео | бот видео |
| 8 | `videos` | Видеофайлы | бот видео |
| 9 | `daily_limits` | Суточные лимиты публикаций | бот видео |
| 10 | `proxies` | Прокси-сервера | общая |
| 11 | `config_variables` | Конфигурация (из ботов) | общая |
| 12 | `user_comments` | Библиотека комментариев | бот комментов |
| 13 | `account_rotation` | Состояние ротации аккаунтов | бот комментов |
| 14 | `processed_videos` | Кэш обработанных видео | бот комментов |
| 15 | `site_content` | CMS контент сайта | сайт |
| 16 | `alembic_version` | Миграции БД | сайт |
