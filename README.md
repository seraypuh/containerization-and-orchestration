# Лабораторная работа 6 -- Kubernetes на двух машинах и CI/CD: автоматическая сборка образа и выкатка в кластер

[![ci](https://github.com/seraypuh/containerization-and-orchestration/actions/workflows/ci.yml/badge.svg?branch=lab-6)](https://github.com/seraypuh/containerization-and-orchestration/actions/workflows/ci.yml)

Веб-приложение на базе **FastAPI**, **PostgreSQL** и **Redis**, развёрнутое в двухнодовом кластере **k3s** с автоматической сборкой мультиархитектурных Docker-образов (`linux/amd64`, `linux/arm64`) в **GitHub Actions** и публикацией в **GitHub Container Registry (GHCR)**.

---

## Содержание

- [Персональные параметры](#персональные-параметры)
- [Структура проекта](#структура-проекта)
- [CI/CD и кластер](#cicd-и-кластер)
  - [Схема узлов и их роли](#схема-узлов-и-их-роли)
  - [Как выкатить новую версию](#как-выкатить-новую-версию)
  - [Как откатить версию](#как-откатить-версию)
- [Развёртывание с нуля](#развёртывание-с-нуля)
- [Проверка работоспособности](#проверка-работоспособности)

---

## Персональные параметры

| Параметр | Значение |
| :--- | :--- |
| **Автор** | Пухарев Сергей Валерьевич |
| **Номер ИСУ** | `467203` |
| **Персональный префикс** | `pukharev` |
| **NodePort приложения** | `30003` |
| **Имя базы данных** | `pukharev03` |
| **Образ в GHCR** | `ghcr.io/seraypuh/containerization-and-orchestration` |

---

## Структура проекта

```text
.
├── .github/
│   └── workflows/
│       └── ci.yml                  # CI-пайплайн: проверка синтаксиса, сборка и публикация в GHCR
└── solutions/
    └── assist-py/
        ├── app/                    # Исходный код FastAPI-приложения
        │   ├── main.py
        │   ├── database.py
        │   ├── models.py
        │   └── crud.py
        ├── k8s/                    # Манифесты Kubernetes
        │   ├── app-configmap.yaml
        │   ├── app-deployment.yaml
        │   ├── app-service.yaml
        │   ├── postgres-deployment.yaml
        │   ├── postgres-pvc.yaml
        │   ├── postgres-service.yaml
        │   ├── redis-deployment.yaml
        │   └── redis-service.yaml
        ├── app-secret.example.yaml # Шаблон файла секретов (сам app-secret.yaml в .gitignore)
        ├── Dockerfile
        ├── requirements.txt
        └── README.md
```

---

## CI/CD и кластер

### Схема узлов и их роли

Кластер развёрнут на базе **k3s** на двух виртуальных машинах Ubuntu 24.04 (Multipass), объединённых в общую локальную сеть (Bridged / виртуальный коммутатор):

```text
                        [ Внешний запрос: NodePort :30003 ]
                                         │
                        ┌────────────────┴────────────────┐
                        ▼                                 ▼
         ┌─────────────────────────────┐   ┌─────────────────────────────┐
         │      pukharev-master        │   │      pukharev-worker        │
         │  Role: control-plane, db    │   │         Role: app           │
         │                             │   │                             │
         │  • k3s server (API, etcd)   │   │  • k3s agent (kubelet)      │
         │  • pukharev-app (2 pods)    │◄─►│  • pukharev-app (2 pods)    │
         │  • pukharev-postgres (1 pod)│   │  • pukharev-redis (1 pod)   │
         │  • PVC (local-path storage) │   │                             │
         └─────────────────────────────┘   └─────────────────────────────┘
```

- **`pukharev-master`** (`role=db`, `control-plane`) — управляющий узел кластера. Запускает системные компоненты Kubernetes, 2 реплики приложения `pukharev-app` и единственный инстанс базы данных `pukharev-postgres` (жёстко привязан через `nodeSelector: role=db` из-за использования локального хранилища `local-path`).
- **`pukharev-worker`** (`role=app`) — рабочий узел. Запускает 2 реплики приложения `pukharev-app` и кэш `pukharev-redis`. Равномерное распределение подов приложения между двумя узлами (`2 + 2`) обеспечивается правилом `topologySpreadConstraints` (`maxSkew: 1`).

### Как выкатить новую версию

1. Внести изменения в код приложения (`solutions/assist-py/app/`) и отправить коммит в ветку `lab-6`:
   ```bash
   git add .
   git commit -m "feat: update greeting message"
   git push origin lab-6
   ```
2. Дождаться успешного выполнения обоих джобов (`test` и `build-and-push`) во вкладке **GitHub Actions** и скопировать новый тег с коротким SHA коммита (например, `sha-<новый_sha>`).
3. Выполнить обновление образа в кластере без простоя (Rolling Update):
   ```bash
   kubectl set image deployment/pukharev-app assist-py-app=ghcr.io/seraypuh/containerization-and-orchestration:sha-<новый_sha>
   kubectl rollout status deployment/pukharev-app
   ```
4. Синхронизировать манифест в репозитории (GitOps): обновить поле `image:` в `solutions/assist-py/k8s/app-deployment.yaml`, закоммитить изменение и выполнить `kubectl apply -f k8s/`.

### Как откатить версию

1. Просмотреть историю ревизий деплоймента:
   ```bash
   kubectl rollout history deployment/pukharev-app
   ```
2. Откатить приложение на предыдущую рабочую ревизию (или на конкретную ревизию через `--to-revision=<N>`):
   ```bash
   kubectl rollout undo deployment/pukharev-app
   kubectl rollout status deployment/pukharev-app
   ```

---

## Развёртывание с нуля

1. Перейдите в каталог приложения на управляющем узле (`pukharev-master`):
   ```bash
   cd solutions/assist-py
   ```
2. Создайте файл секретов `k8s/app-secret.yaml` на основе шаблона `app-secret.example.yaml` (файл исключён из Git через `.gitignore`):
   ```bash
   cp app-secret.example.yaml k8s/app-secret.yaml
   ```
   Заполните в `k8s/app-secret.yaml` значения `APP_SECRET`, `POSTGRES_PASSWORD` и `DATABASE_URL`.
3. Примените манифесты Kubernetes:
   ```bash
   kubectl apply -f k8s/
   ```
4. Убедитесь, что все поды перешли в состояние `Running` и распределены по обоим узлам:
   ```bash
   kubectl get pods -o wide
   ```

---

## Проверка работоспособности

```bash
# Проверка статуса сервиса и владельца приложения (APP_OWNER)
curl http://<IP_узла>:30003/
curl http://<IP_узла>:30003/health

# Проверка балансировки нагрузки между репликами на разных узлах
for i in $(seq 1 10); do
  curl -s http://<IP_узла>:30003/ | python3 -c "import sys,json; print(json.load(sys.stdin).get('hostname'))"
done

# Проверка кэширования в Redis (первый запрос -> api, второй -> cache)
curl http://<IP_master>:30003/weather/Moscow
curl http://<IP_worker>:30003/weather/Moscow

# Проверка записи и чтения из PostgreSQL через разные узлы
curl -X POST http://<IP_master>:30003/notes \
  -H "Content-Type: application/json" \
  -d '{"text": "ЛР6: кластер из двух узлов"}'
curl http://<IP_worker>:30003/notes
```