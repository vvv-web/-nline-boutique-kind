# PROJECT_LOG — Online Boutique on Kind + GitHub Actions (self-hosted, WSL2)

Дата: 2026-01-08
Repo: https://github.com/vvv-web/-nline-boutique-kind
Окружение: Windows + WSL2 (Ubuntu), Kubernetes в Kind (локально)

## Чек-лист этапов
- [x] Этап 0 — База: Kind + Online Boutique подняты и работают
- [x] Этап CI/CD — GitHub repo + self-hosted runner + manual deploy (workflow_dispatch)
- [x] Этап 1 — Monitoring: kube-prometheus-stack + Grafana доступна
- [ ] Этап 2 — Kafka: кластер Kafka поднят + метрики в Prometheus
- [ ] Этап 3 — Tracing: Jaeger поднят + UI доступен + есть трейсы (demo)
- [ ] Этап 4 — (опц.) Интеграция Online Boutique: метрики/трейсы приложения

---

## 1) Kubernetes (Kind) + Online Boutique
- Развернули Google Online Boutique (microservices-demo) в Kind.
- Проверка: `kubectl get pods` — все сервисы в namespace `default` в статусе `Running`:
  adservice, cartservice, checkoutservice, currencyservice, emailservice, frontend, loadgenerator,
  paymentservice, productcatalogservice, recommendationservice, redis-cart, shippingservice.

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
- В WSL2 каталог runner’а: `~/actions-runner`.
- Установлен GitHub Actions Runner v2.330.0:
  - Архив: `actions-runner-linux-x64-2.330.0.tar.gz`
  - SHA256 проверен (OK)
  - Установка: `tar xzf ...`
  - Регистрация: `./config.sh --url ... --token ...`
  - Запуск: `./run.sh`
- Runner статус: "Listening for Jobs" и успешно выполняет jobs.

## 4) Доступ GitHub Actions к Kubernetes (ServiceAccount + kubeconfig)
Цель: дать workflow доступ к API Kubernetes через bearer token ServiceAccount.

- ServiceAccount:
  - `kubectl create serviceaccount gh-actions-deployer -n default`
- Права (лабораторный вариант):
  - `kubectl create clusterrolebinding gh-actions-deployer-binding --clusterrole=cluster-admin --serviceaccount=default:gh-actions-deployer`
- Токен (ограниченный по времени):
  - `kubectl create token gh-actions-deployer -n default --duration 24h`
  - Важно: токен не хранить в репозитории/логах.
- API endpoint Kind:
  - `kubectl cluster-info` → `https://127.0.0.1:45553`
- kubeconfig (локально, НЕ коммитить):
  - Файл: `~/microservices-demo/kubeconfig-kind`
  - Проверка: `KUBECONFIG=$PWD/kubeconfig-kind kubectl get nodes`

## 5) GitHub Secret
- В репозитории GitHub создали secret:
  - Name: `KUBE_CONFIG`
  - Value: содержимое `kubeconfig-kind` (полностью)

## 6) GitHub Actions workflow для деплоя в Kind
- Workflow файл: `.github/workflows/deploy-kind.yaml`
- Логика workflow:
  1) Checkout кода
  2) Создать временный файл `kubeconfig` из секрета `KUBE_CONFIG`
  3) Выполнить `kubectl apply -f release/kubernetes-manifests.yaml`
  4) Показать `kubectl get pods`
- Перевели деплой на ручной запуск:
  - `on: workflow_dispatch`
  - В GitHub появилась кнопка "Run workflow"
- Проверка: ручной запуск выполняется, runner пишет:
  - "Running job: deploy"
  - "Job deploy completed with result: Succeeded"

---

## 7) Этап 1 — Monitoring (Prometheus + Grafana) ✅
Цель: поставить kube-prometheus-stack в Kind и открыть Grafana.

- Helm установлен: `helm version` → v3.19.2.
- Установили kube-prometheus-stack:
  - Release: `kps`
  - Namespace: `monitoring`
- Проверка: `kubectl get pods -n monitoring` — все ключевые поды Running:
  alertmanager, grafana, operator, kube-state-metrics, node-exporter, prometheus.
- Grafana доступ:
  - `kubectl port-forward -n monitoring svc/kps-grafana 3000:80`
  - URL: http://localhost:3000
- Secret с кредами Grafana:
  - Namespace: monitoring
  - Secret: `kps-grafana`
  - Получение пароля/логина (не коммитить и не светить публично):
    `kubectl get secret -n monitoring kps-grafana -o jsonpath="{.data.admin-user}" | base64 --decode ; echo`
    `kubectl get secret -n monitoring kps-grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo`

---

## План: Observability + Kafka + Tracing (простая версия)
Цель: поставить в Kind базовый стек наблюдаемости и платформенные компоненты, не переписывая Online Boutique.

### Этап 2 — Kafka (как отдельная платформа)
- [ ] Установить Kafka в namespace `kafka` (выбрать рабочий Helm источник; у bitnami repo сейчас 403).
- [ ] Включить метрики (exporter/JMX), чтобы Prometheus их видел.
- [ ] (Пока без интеграции с Online Boutique.)

### Этап 3 — Tracing (Jaeger “standalone”)
- [ ] Установить Jaeger через Helm chart в namespace `observability`.
- [ ] Поднять demo (например hotrod) чтобы появились трейсы.
- [ ] Открыть Jaeger UI через `kubectl port-forward`.
- [ ] (Важно: OTEL Collector — отдельным шагом, если понадобится.)

### Этап 4 — (опц.) Интеграция приложения
- [ ] Подключить Online Boutique к метрикам/трейсам (сложный этап: настройки сервисов/OTel).
