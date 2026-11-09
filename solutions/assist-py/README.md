
# Containerization & Orchestration — Kubernetes и CI/CD

[![CI](https://github.com/KellyLamb123/Konteyner/actions/workflows/ci.yml/badge.svg?branch=lab-6)](https://github.com/KellyLamb123/Konteyner/actions/workflows/ci.yml)

## CI/CD и кластер

### Описание проекта

Лабораторная работа №6 посвящена развёртыванию FastAPI-приложения в двухузловом Kubernetes-кластере k3s, автоматической сборке Docker-образов через GitHub Actions и публикации в GitHub Container Registry (GHCR).

Приложение использует PostgreSQL для хранения заметок и Redis для кэширования погоды.

### Архитектура кластера

Кластер развёрнут на двух виртуальных машинах Ubuntu Server 24.04 ARM64 в UTM на macOS. Виртуальные машины объединены локальной сетью в режиме Bridged.

- Master: `meshcheriakov-master`, `192.168.1.88`, роль `control-plane`, метка `role=db`
- Worker: `meshcheriakov-worker`, `192.168.1.48`, метка `role=app`
- FastAPI: 4 реплики, размещаемые на обоих узлах
- PostgreSQL: 1 реплика на Master, PVC с StorageClass `local-path`
- Redis: 1 реплика, общий кэш приложения
- NodePort: `30000`

```text
                  GitHub Actions
                        |
                GHCR Docker Image
                        |
               Kubernetes (k3s)
                        |
          +-------------+-------------+
          |                           |
      Master                       Worker
   192.168.1.88                192.168.1.48
   control-plane                k3s-agent
          |                           |
    FastAPI pods                 FastAPI pods
    PostgreSQL                   Redis
    PVC (local-path)
          |                           |
          +------- NodePort ----------+
                       30000
```

### CI — GitHub Actions

Workflow расположен в `.github/workflows/ci.yml`.

При push в ветку `lab-6` выполняются:

1. `test` — установка Python 3.11 и зависимостей, проверка синтаксиса приложения.
2. `build-and-push` — сборка Docker-образа для `linux/amd64` и `linux/arm64` и публикация в GHCR.

Реестр образов:

`ghcr.io/kellylamb123/konteyner`

### Развёртывание приложения

Из каталога `solutions/assist-py/` на Master:

```bash
kubectl apply -f k8s/
kubectl get pods -o wide
kubectl get deployments
```

Настоящий файл `k8s/app-secret.yaml` создаётся локально и не публикуется в GitHub.

### Обновление приложения (Rolling Update)

После изменения кода и успешной сборки в GitHub Actions необходимо использовать SHA-тег нового образа:

```bash
kubectl set image deployment/meshcheriakov-app \
  fastapi=ghcr.io/kellylamb123/konteyner:sha-НОВЫЙ_SHA

kubectl rollout status deployment/meshcheriakov-app
kubectl get pods -o wide
```

### Откат обновления (Rollback)

```bash
kubectl rollout history deployment/meshcheriakov-app
kubectl rollout undo deployment/meshcheriakov-app
kubectl rollout status deployment/meshcheriakov-app
```

При необходимости используется `--to-revision=N`, где N — номер нужной ревизии из истории.

### Проверка работы

```bash
curl http://192.168.1.88:30000/
curl http://192.168.1.48:30000/health
kubectl get nodes -o wide
kubectl get pods -o wide
```

Redis используется совместно всеми репликами, а заметки сохраняются в общей PostgreSQL.

Для доступа к внешнему Open-Meteo в учебной сети временно использовался HTTPS-прокси на Mac с VPN. Это локальная настройка окружения, не обязательная для запуска Kubernetes.

### Проверка отказоустойчивости

На Worker:

```bash
sudo systemctl stop k3s-agent
sudo systemctl start k3s-agent
```

На Master:

```bash
kubectl get nodes
kubectl get pods -o wide
```

Эксперимент подтвердил переход Worker в `NotReady` и последующее восстановление `Ready`.

### Файлы проекта

- `solutions/assist-py/` — исходный код приложения и Kubernetes-манифесты.
- `.github/workflows/ci.yml` — CI workflow.
- `solutions/reports/lab-6/` — отчёт лабораторной работы.
```
