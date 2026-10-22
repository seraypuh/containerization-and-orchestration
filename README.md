# Лабораторная 4 -- Docker compose

Проект представляет собой перенос микросервисной архитектуры ассистента на базу **Docker Compose v2** с интеграцией кэширования через **Redis** и поддержкой горизонтального масштабирования.

---

## Стек технологий

* **App:** FastAPI (Python 3.12)
* **Database:** PostgreSQL 15 (персистентное хранилище с Named Volume)
* **Cache:** Redis 7 (кэширование внешних запросов к Open-Meteo API)
* **Orchestration:** Docker Compose v2

---

## Структура окружения (`.env`)

Перед запуском убедитесь, что в корне каталога `solutions/assist-py/` создан файл `.env` на основе шаблона `.env.example`:

```env
APP_OWNER=pukharev
APP_SECRET=secret03pukharev
APP_PORT=8003
POSTGRES_USER=user_03
POSTGRES_PASSWORD=pgpass_03
POSTGRES_DB=assistant_03
PG_PORT=5433
REDIS_PORT=6373
CACHE_TTL=120

```

---

## Запуск и управление

### 1. Стандартный режим (db + redis + app)

Сборка образов и запуск всех сервисов в фоновом режиме:

```bash
docker compose up -d --build

```

Сервисы поднимаются с контролем очередности: приложение стартует только после прохождения `healthcheck` со стороны PostgreSQL и Redis.

Проверка статуса:

```bash
docker compose ps

```

Остановка стека:

```bash
docker compose down

```

---

### 2. Кэширование запросов погоды

Эндпоинт `/weather/{city}` выполняет кэширование ответов от погодного провайдера в Redis с временем жизни ключа `CACHE_TTL`:

* **Первый запрос** — обращение к внешнему API (`"source": "api"`) с сохранением результата в Redis.
* **Повторные запросы** — мгновенная отдача из кэша (`"source": "cache"`).

Проверка статистики кэша:

```bash
curl http://localhost:8003/cache/stats

```

---

### 3. Горизонтальное масштабирование сервиса `app`

Для масштабирования используется конфигурация `docker-compose.scale.yml`, где снято статическое имя контейнера, а порт приложения привязан динамически.

Запуск 3 экземпляров приложения:

```bash
docker compose down
docker compose -f docker-compose.scale.yml up -d --build --scale app=3

```

Проверка назначенных хостовых портов:

```bash
docker compose -f docker-compose.scale.yml ps app

```

Все экземпляры используют общую изолированную сеть `pukharev-backend` и делят общий кэш в Redis.

Возврат к базовой конфигурации:

```bash
docker compose -f docker-compose.scale.yml down
docker compose up -d

```

```

```