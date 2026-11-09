
## Kubernetes (Лабораторная работа №5)

### Описание

В лабораторной работе №5 приложение Personal Assistant развёртывается в локальном кластере Kubernetes с использованием Minikube.

Приложение состоит из трёх компонентов:
- FastAPI — основное веб-приложение;
- PostgreSQL — база данных для хранения заметок;
- Redis — система кэширования запросов погоды.

В отличие от предыдущей лабораторной работы, вместо Docker Compose используются ресурсы Kubernetes: Deployment, Service, ConfigMap, Secret и PersistentVolumeClaim.

### 1. Подготовка окружения

Для работы необходимы Docker Desktop, Minikube и kubectl.

Все команды выполняются из каталога `solutions/assist-py/`.

Запуск локального кластера Kubernetes:

```bash
minikube start --driver=docker --cpus=2 --memory=4096
```

Проверка состояния кластера:

```bash
minikube status
kubectl get nodes
```

Узел Kubernetes должен находиться в состоянии `Ready`.

### 2. Подготовка Docker-образа

Сборка образа приложения:

```bash
docker build -t meshcheriakov-assistant:lab5 .
```

Загрузка образа в Minikube:

```bash
minikube image load meshcheriakov-assistant:lab5
```

Проверка наличия образа:

```bash
minikube image ls | grep meshcheriakov-assistant
```

### 3. Развёртывание PostgreSQL

Сначала необходимо создать локальный файл `k8s/app-secret.yaml` на основе `k8s/app-secret.example.yaml` и указать в нём реальные значения `POSTGRES_PASSWORD` и `APP_SECRET`.

Применение Secret и манифестов PostgreSQL:

```bash
kubectl apply -f k8s/app-secret.yaml
kubectl apply -f k8s/postgres-pvc.yaml
kubectl apply -f k8s/postgres-deployment.yaml
kubectl apply -f k8s/postgres-service.yaml
```

Проверка:

```bash
kubectl get pods -l app=meshcheriakov-postgres
kubectl get pvc
```

Pod должен находиться в состоянии `Running`, а PVC — `Bound`.

### 4. Развёртывание Redis

Применение манифестов Redis:

```bash
kubectl apply -f k8s/redis-deployment.yaml
kubectl apply -f k8s/redis-service.yaml
```

Проверка Redis:

```bash
kubectl exec deployment/meshcheriakov-redis -- redis-cli PING
```

Ожидаемый результат: `PONG`.

### 5. Развёртывание FastAPI

Применение ConfigMap, Deployment и Service приложения:

```bash
kubectl apply -f k8s/app-configmap.yaml
kubectl apply -f k8s/app-deployment.yaml
kubectl apply -f k8s/app-service.yaml
```

Проверка развёртывания:

```bash
kubectl get pods
kubectl get services
```

Все три Pod должны находиться в состоянии `Running`.

### 6. Доступ к приложению

Для открытия FastAPI на macOS используется команда:

```bash
minikube service meshcheriakov-app
```

Minikube предоставляет локальный адрес приложения. Терминал с запущенным сервисным туннелем необходимо оставить открытым.

Для просмотра Swagger UI к адресу приложения нужно добавить `/docs`.

### 7. Проверка работоспособности

Проверка всех основных ресурсов:

```bash
kubectl get all
kubectl get pvc
```

Просмотр логов FastAPI:

```bash
kubectl logs deployment/meshcheriakov-app
```

Через Swagger UI необходимо проверить следующие эндпоинты:

- `GET /` — приветствие, hostname и server_time;
- `GET /health` — состояние PostgreSQL и Redis;
- `GET /weather/{city}` — первый запрос из API, повторный из Redis;
- `POST /notes` — создание заметки в PostgreSQL;
- `GET /notes` — получение сохранённых заметок.

Первый запрос погоды должен возвращать `"source": "api"`, повторный — `"source": "cache"`.

### 8. Проверка readinessProbe и livenessProbe

После настройки проб Kubernetes необходимо выполнить:

```bash
kubectl describe pod -l app=meshcheriakov-app
```

В описании Pod должны отображаться настройки readinessProbe и livenessProbe, а состояние готовности должно быть `Ready: True`.

### 9. Хранение конфигурации и секретов

Обычные настройки приложения, включая APP_OWNER, REDIS_URL и CACHE_TTL, хранятся в ConfigMap.

Пароль PostgreSQL и секретный ключ приложения хранятся в Kubernetes Secret.

Файл `k8s/app-secret.yaml` содержит реальные значения и исключён из Git с помощью `.gitignore`.

В репозитории публикуется только `k8s/app-secret.example.yaml` с учебными значениями.
