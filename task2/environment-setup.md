## 1. Предварительные требования

- Установлены Docker, Minikube, kubectl, Helm 3.
- Клонирован репозиторий с заданием (где лежат `part1`, `part2`, `scaletestapp`, `scaletestapp.tar`).

Проверка:

```bash
docker version
minikube version
kubectl version --client
helm version
```

***

## 2. Запуск Minikube

```bash
minikube start
minikube addons enable metrics-server

kubectl get nodes
```

***

## 3. Загрузка образа приложения в Minikube

В корне репозитория (где лежит `scaletestapp.tar`):

```bash
# подключить docker внутри minikube (если нужно собирать)
minikube image load scaletestapp.tar
```

Образ должен называться так же, как используется в `part1/deployment.yaml` (если там `image: scaletestapp:latest`, то внутри tar должен быть `scaletestapp:latest`).

Проверка:

```bash
minikube image ls | grep scaletestapp
```

***

## 4. Часть 1 — HPA по памяти

### 4.1. Deployment

В директории `part1` лежит `deployment.yaml`.

```bash
cd part1
kubectl apply -f deployment.yaml
kubectl get pods
```

(Deployment разворачивает приложение `scaletestapp` с ресурсами и лимитами памяти, необходимыми для HPA по memory.)

### 4.2. Service

```bash
kubectl apply -f service.yaml
kubectl get svc scaletestapp-service
```

Получить URL для теста (используется в Locust):

```bash
minikube service scaletestapp-service --url
```

### 4.3. HPA по памяти

```bash
kubectl apply -f hpa.yaml
kubectl get hpa
kubectl describe hpa scaletestapp-hpa
```

HPA использует метрику `memory` (Utilization, целевой процент утилизации памяти на pod).

### 4.4. Нагрузочное тестирование (Locust)

В `part1/locustfile.py` уже описан сценарий:

```bash
cd part1
pip install locust
locust
```

- Открыть http://localhost:8089
- В поле Host указать URL из `minikube service scaletestapp-service --url`
- Запустить нагрузку (например, 200–500 пользователей, spawn rate 20)

Мониторинг HPA:

```bash
kubectl describe hpa scaletestapp-hpa
kubectl get pods
kubectl top pods
```

Скриншоты `part1/screenshots/*.png` показывают изменение числа реплик до/во время/после нагрузки.

***

## 5. Часть 2 — Prometheus и масштабирование по RPS

### 5.1. Установка Prometheus через Helm

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm install prometheus prometheus-community/prometheus \
  --namespace monitoring --create-namespace
```

Проверка:

```bash
kubectl get pods -n monitoring
kubectl get svc -n monitoring
```

### 5.2. Дополнительный scrape job (prometheus-values.yaml)

В папке `part2` лежит `prometheus-values.yaml`. Он добавляет `extraScrapeConfigs`, чтобы Prometheus скрейпил сервисы с аннотацией `prometheus.io/scrape: "true"`.

Обновление релиза:

```bash
cd part2
helm upgrade prometheus prometheus-community/prometheus \
  -n monitoring \
  -f prometheus-values.yaml
```

### 5.3. Service для приложения (с аннотациями для Prometheus)

В `part2/service.yaml` — сервис с аннотациями:

```bash
kubectl apply -f service.yaml
kubectl get svc scaletestapp-service
```

Важно: это тот же сервис (имя `scaletestapp-service`), что используется в части 1, но c Prometheus‑аннотациями (`prometheus.io/scrape`, `prometheus.io/path`, `prometheus.io/port`).

### 5.4. Проверка в Prometheus

Пробросить порт:

```bash
kubectl port-forward -n monitoring svc/prometheus-server 9090:80
```

Открыть http://localhost:9090:

- Status → Targets: должен быть job, где виден `default/scaletestapp-service` в состоянии UP
- Graph → ввести `http_requests_total` или `rate(http_requests_total[1m])` — должны быть метрики от приложения.

Скрин `part2/screenshots/prometheus.png` демонстрирует наличие метрик приложения в Prometheus.

***

## 6. Где что лежит

- `part1/deployment.yaml` — Deployment приложения
- `part1/service.yaml` — Service для приложения (вариант части 1)
- `part1/hpa.yaml` — HPA по памяти
- `part1/locustfile.py` — сценарий нагрузочного теста
- `part1/screenshots/*.png` — результат масштабирования по памяти

- `part2/service.yaml` — Service с аннотациями для Prometheus
- `part2/prometheus-values.yaml` — настройки Prometheus (extraScrapeConfigs)
- `part2/screenshots/prometheus.png` — наличие метрик приложения в Prometheus

- `scaletestapp/` — исходники тестового приложения (Go + Dockerfile)
- `scaletestapp.tar` — сохранённый Docker‑образ для загрузки в Minikube
