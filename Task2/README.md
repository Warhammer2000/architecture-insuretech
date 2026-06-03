# Task2. Динамическое масштабирование контейнеров

Здесь лежит конфигурация динамического масштабирования тестового приложения scaletestapp в Kubernetes. Масштабирование настроено двумя способами, по утилизации памяти и по количеству запросов в секунду на под.

## Файлы

- `deployment.yaml` запускает приложение с лимитом памяти 30Mi и одной репликой на старте
- `service.yaml` открывает доступ к приложению внутри кластера по порту 8080
- `hpa-memory.yaml` масштабирует приложение по утилизации памяти с целевым значением 80 процентов
- `servicemonitor.yaml` подключает сбор метрик приложения в Prometheus
- `prometheus-adapter-values.yaml` превращает счётчик http_requests_total в метрику http_requests_per_second для Kubernetes
- `hpa-rps.yaml` масштабирует приложение по количеству запросов в секунду на под
- `locustfile.py` сценарий нагрузки для Locust
- папка `logs` хранит подтверждающие логи и скриншоты по двум частям задания

## Часть 1. Масштабирование по памяти

1. Поднимаю локальный кластер и включаю metrics-server

   ```
   minikube start
   minikube addons enable metrics-server
   ```

2. Применяю манифесты приложения и автоскейлера

   ```
   kubectl apply -f deployment.yaml
   kubectl apply -f service.yaml
   kubectl apply -f hpa-memory.yaml
   ```

3. Пробрасываю порт и запускаю нагрузку

   ```
   kubectl port-forward svc/scaletestapp 8080:8080
   locust -f locustfile.py --host http://localhost:8080
   ```

   В интерфейсе Locust по адресу http://localhost:8089 задаю количество пользователей и hatch rate, затем запускаю тест.

4. Наблюдаю за изменением количества реплик и сохраняю вывод в папку `logs`

   ```
   kubectl get hpa scaletestapp -w
   kubectl get pods -l app=scaletestapp -w
   ```

## Часть 2. Масштабирование по количеству запросов в секунду

1. Ставлю стек Prometheus через Helm

   ```
   helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
   helm repo update
   helm install prometheus prometheus-community/kube-prometheus-stack -n monitoring --create-namespace
   ```

2. Подключаю сбор метрик приложения

   ```
   kubectl apply -f servicemonitor.yaml
   ```

   В Prometheus Web UI открываю раздел Targets и Graph и проверяю, что метрика http_requests_total поступает. Скриншот кладу в папку `logs`.

3. Ставлю Prometheus Adapter с правилом для метрики http_requests_per_second

   ```
   helm install prometheus-adapter prometheus-community/prometheus-adapter -n monitoring -f prometheus-adapter-values.yaml
   ```

   Проверяю, что кастомная метрика стала доступна

   ```
   kubectl get --raw /apis/custom.metrics.k8s.io/v1beta1
   ```

4. Переключаю автоскейлер на масштабирование по RPS

   ```
   kubectl delete hpa scaletestapp
   kubectl apply -f hpa-rps.yaml
   ```

5. Повторяю нагрузку Locust, наблюдаю за репликами и сохраняю вывод в папку `logs`.

## Примечания

- Метрика http_requests_per_second отдаётся как количество запросов в секунду на под, целевое значение в `hpa-rps.yaml` равно 10.
- Адрес Prometheus в `prometheus-adapter-values.yaml` рассчитан на установку стека с именем релиза prometheus в неймспейсе monitoring. Если имя релиза другое, адрес нужно поправить.
- Лейбл release prometheus в `servicemonitor.yaml` нужен, чтобы оператор kube-prometheus-stack подхватил монитор.

## Результаты прогона

- Часть 1. Логи в `logs/part1-memory.log` и заметка `logs/part1-memory-note.md`. Память приложения стоит на месте, HPA по памяти держит одну реплику и при этом работает корректно.
- Часть 2. Логи в `logs/part2-rps.log`, `logs/part2-rps-scaleup.log`, `logs/prometheus-metric.log` и заметка `logs/part2-rps-note.md`. Под нагрузкой реплики выросли с 1 до 10 по метрике запросов в секунду.
