# Лабораторная работа 5 — Введение в Kubernetes: развёртывание приложения с PostgreSQL и Redis

Веб-сервис персонального ассистента на базе **FastAPI** с использованием **PostgreSQL** для хранения данных и **Redis** для кэширования запросов, развёрнутый в локальном кластере Kubernetes (`minikube`).

## Архитектура и стек технологий

* **Backend:** Python 3.12, FastAPI, Uvicorn, SQLAlchemy
* **База данных:** PostgreSQL 15 (с постоянным хранилищем `PersistentVolumeClaim`)
* **Кэш:** Redis 7 (Alpine)
* **Оркестрация:** Kubernetes (`minikube`, `kubectl`)
* **Маршрутизация:** `NodePort` Service + Nginx `Ingress`

### Компоненты Kubernetes (`k8s/`)

| Ресурс | Имя объекта | Назначение |
| :--- | :--- | :--- |
| `PersistentVolumeClaim` | `pukharev-postgres-pvc` | Постоянный том (`1Gi`, `ReadWriteOnce`) для хранения данных PostgreSQL |
| `Deployment` | `pukharev-postgres` | Под с СУБД PostgreSQL 15 (порт `5432`) |
| `Service` (`ClusterIP`) | `pukharev-postgres` | Внутренний DNS-доступ к БД внутри кластера |
| `Deployment` | `pukharev-redis` | Под с кэшем Redis 7 (порт `6379`) |
| `Service` (`ClusterIP`) | `pukharev-redis` | Внутренний DNS-доступ к Redis внутри кластера |
| `ConfigMap` | `pukharev-configmap` | Несекретные переменные окружения (`APP_OWNER`, `CACHE_TTL`, `REDIS_URL`, и др.) |
| `Secret` | `pukharev-secret` | Чувствительные данные (`APP_SECRET`, `POSTGRES_PASSWORD`, `DATABASE_URL`) |
| `Deployment` | `pukharev-app` | Под с FastAPI-приложением (порт `5433`), настроены `readinessProbe` и `livenessProbe` |
| `Service` (`NodePort`) | `pukharev-app` | Проброс порта сервиса `8003` на `targetPort: 5433` |
| `Ingress` | `pukharev-ingress` | L7-маршрутизация внешнего трафика по домену `pukharev.local` |

## Структура проекта

```text
assist-py/
├── app/                         # Исходный код FastAPI-приложения
│   ├── crud.py                  # Операции с БД
│   ├── database.py              # Подключение к PostgreSQL
│   ├── main.py                  # Эндпоинты API и интеграция с Redis
│   └── models.py                # ORM-модели SQLAlchemy
├── config/
│   └── app.conf                 # Конфигурация приложения
├── k8s/                         # Манифесты Kubernetes
│   ├── app-configmap.yaml       # Конфигурация приложения (ConfigMap)
│   ├── app-deployment.yaml      # Деплоймент приложения с health-пробами
│   ├── app-ingress.yaml         # Правила маршрутизации Nginx Ingress
│   ├── app-secret.yaml          # Секреты (исключён из Git через .gitignore)
│   ├── app-service.yaml         # Сервис NodePort для приложения
│   ├── postgres-deployment.yaml # Деплоймент PostgreSQL с монтированием PVC
│   ├── postgres-pvc.yaml        # Запрос на выделение постоянного тома
│   ├── postgres-service.yaml    # Сервис ClusterIP для PostgreSQL
│   ├── redis-deployment.yaml    # Деплоймент Redis
│   └── redis-service.yaml       # Сервис ClusterIP для Redis
├── app-secret.example.yaml      # Шаблон манифеста Secret без реальных паролей
├── Dockerfile                   # Сборка образа приложения
├── docker-compose.yml           # Локальный запуск через Docker Compose
├── requirements.txt             # Python-зависимости
└── README.md
```

## Быстрый старт (Kubernetes / Minikube)

### 1. Предварительные требования
* Установленные `docker`, `minikube` и `kubectl`
* Запущенный кластер:
  ```bash
  minikube start --driver=docker
  ```

### 2. Сборка и загрузка локального Docker-образа
Соберите образ приложения и загрузите его во внутренний реестр `minikube`:
```bash
docker build -t assist-py-app:latest .
minikube image load assist-py-app:latest
```

### 3. Подготовка секретов
Скопируйте шаблон секрета в рабочую директорию манифестов и заполните реальными значениями:
```bash
cp app-secret.example.yaml k8s/app-secret.yaml
```

### 4. Развёртывание в кластере
Примените все манифесты и дождитесь перехода подов в статус `Running`:
```bash
kubectl apply -f k8s/
kubectl get pods,svc,pvc
```

## Доступ к приложению и проверка API

### Вариант А: Через туннель NodePort
Откройте туннель до сервиса `pukharev-app` (терминал должен оставаться открытым):
```bash
minikube service pukharev-app --url
```

### Вариант Б: Через Nginx Ingress (`pukharev.local`)
1. Включите аддон Ingress в `minikube`:
   ```bash
   minikube addons enable ingress
   ```
2. Добавьте локальный домен в `/etc/hosts` и запустите туннель (для macOS с Docker-драйвером):
   ```bash
   echo "127.0.0.1 pukharev.local" | sudo tee -a /etc/hosts
   sudo minikube tunnel
   ```

### Примеры запросов для проверки
Задайте базовый URL (например, `BASE_URL=http://pukharev.local` или адрес из `minikube service`):

```bash
# Информация о сервисе и имени пода
curl $BASE_URL/

# Проверка состояния PostgreSQL и Redis
curl $BASE_URL/health

# Проверка кэширования погоды (1-й запрос — API, 2-й — Redis cache)
curl $BASE_URL/weather/Moscow
curl $BASE_URL/weather/Moscow
curl $BASE_URL/cache/stats

# Работа с заметками в PostgreSQL
curl -X POST $BASE_URL/notes \
  -H "Content-Type: application/json" \
  -d '{"text": "Заметки работают"}'
curl $BASE_URL/notes
```

## Остановка и очистка ресурсов

Удаление ресурсов из кластера и остановка `minikube`:
```bash
kubectl delete -f k8s/
minikube stop
```