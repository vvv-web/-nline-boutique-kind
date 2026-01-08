# PROJECT_LOG — Online Boutique on Kind + GitHub Actions (self-hosted, WSL2)

Дата: 2026-01-08  
Repo: https://github.com/vvv-web/-nline-boutique-kind  
Окружение: Windows + WSL2 (Ubuntu), Kubernetes в Kind (локально)

## 1) Что развернули в Kubernetes (Kind)
- Развернули Google Online Boutique (microservices-demo) в Kind.
- Проверка: `kubectl get pods` — все сервисы в namespace `default` в статусе `Running` (adservice, cartservice, checkoutservice, currencyservice, emailservice, frontend, loadgenerator, paymentservice, productcatalogservice, recommendationservice, redis-cart, shippingservice).

## 2) GitHub репозиторий и доступ
- Создали репозиторий в GitHub: `vvv-web/-nline-boutique-kind` (опечатка в имени оставлена как есть).
- Запушили код из `~/microservices-demo` в ветку `main`.
- Проблема: git/ssh по умолчанию подключался как другой аккаунт (Valera1488-s).
- Решение: добавили SSH key в аккаунт `vvv-web` и настроили SSH:
  - Проверка: `ssh -T git@github.com` показывает `Hi vvv-web! ...`
  - Файл: `~/.ssh/config` закреплён на нужный ключ:
    Host github.com
      HostName github.com
      User git
      IdentityFile ~/.ssh/id_ed25519
      IdentitiesOnly yes

## 3) Self-hosted runner (WSL2)
Цель: выполнять GitHub Actions на локальной машине (WSL2), чтобы деплоить в локальный Kind.
- В репозитории GitHub: Settings → Actions → Runners → New self-hosted runner (Linux x64).
- В WSL2 создали каталог runner’а: `~/actions-runner`.
- Скачали и установили GitHub Actions Runner v2.330.0:
  - Файл: `actions-runner-linux-x64-2.330.0.tar.gz`
  - Проверили SHA256 (OK)
  - Распаковали: `tar xzf ...`
  - Зарегистрировали runner командой `./config.sh --url ... --token ...`
  - Запустили: `./run.sh`
- Runner статус: "Listening for Jobs" и успешно выполняет jobs.

## 4) Доступ GitHub Actions к Kubernetes (ServiceAccount + kubeconfig)
Цель: дать workflow доступ к API Kubernetes через bearer token ServiceAccount.
- Создали ServiceAccount:
  - `kubectl create serviceaccount gh-actions-deployer -n default`
- Выдали права (для лаборатории):
  - `kubectl create clusterrolebinding gh-actions-deployer-binding --clusterrole=cluster-admin --serviceaccount=default:gh-actions-deployer`
- Создали токен (с ограничением по времени):
  - `kubectl create token gh-actions-deployer -n default --duration 24h`
  - Важно: токен не хранить в репозитории/логах.
- API endpoint Kind:
  - `kubectl cluster-info` → `https://127.0.0.1:45553`
- Создали kubeconfig-файл (локально):
  - Файл: `~/microservices-demo/kubeconfig-kind`
  - В kubeconfig используется `users.user.token` (bearer token).
- Проверили kubeconfig:
  - `KUBECONFIG=$PWD/kubeconfig-kind kubectl get nodes`

## 5) GitHub Secret
- В репозитории GitHub создали secret:
  - Name: `KUBE_CONFIG`
  - Value: содержимое `kubeconfig-kind` (полностью)

## 6) GitHub Actions workflow для деплоя в Kind
- Создали workflow файл:
  - `.github/workflows/deploy-kind.yaml`
- Логика workflow:
  1) Checkout кода
  2) Создать временный файл `kubeconfig` из секрета `KUBE_CONFIG`
  3) Выполнить `kubectl apply -f release/kubernetes-manifests.yaml`
  4) Показать `kubectl get pods`
- Настроили, чтобы деплой запускался вручную:
  - `on: workflow_dispatch`
  - В GitHub появилась кнопка "Run workflow"
- Проверка: ручной запуск выполняется, runner пишет:
  - "Running job: deploy"
  - "Job deploy completed with result: Succeeded"

## 7) Текущее состояние
- CI/CD деплой в локальный Kind через GitHub Actions self-hosted runner работает.
- Следующий шаг (план): мониторинг (Prometheus + Grafana) в Kind, затем сбор метрик с Online Boutique.

## Важные файлы/пути
- Repo: https://github.com/vvv-web/-nline-boutique-kind
- Local project: `~/microservices-demo`
- Runner: `~/actions-runner`
- Workflow: `.github/workflows/deploy-kind.yaml`
- kubeconfig (локально, НЕ коммитить): `~/microservices-demo/kubeconfig-kind`
- Secret в GitHub: `KUBE_CONFIG`

## Официальные доки (на которые опирались)
- GitHub: Adding self-hosted runners / использование self-hosted runner в workflow / ручной запуск workflow (workflow_dispatch).
- Kubernetes: `kubectl create token` (ServiceAccount token) и kubeconfig `users.user.token`.
