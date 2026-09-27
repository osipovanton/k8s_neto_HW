# Домашнее задание 1.3 — «Запуск приложений в K8S»

## Задание 1. Deployment из nginx и multitool

Манифесты:

- [Deployment](task1-deployment.yaml)
- [Service](task1-service.yaml)
- [Pod multitool-client](task1-client.yaml)

### 1. Запуск Deployment с одной репликой

```bash
kubectl apply -f task1-deployment.yaml
kubectl get pods -l app=netology-app
```

В одном Pod находятся два контейнера: `nginx` и `multitool`.

Оба приложения по умолчанию используют HTTP-порт 80. Контейнеры внутри одного Pod используют общий network namespace, поэтому одновременно слушать один и тот же IP:port они не могут. Для `multitool` HTTP-порт изменён на `8080`, HTTPS-порт — на `8443` через переменные окружения `HTTP_PORT` и `HTTPS_PORT`.

**Для отчёта:** скриншот `kubectl get pods -l app=netology-app` с одной репликой.

### 2. Масштабирование до двух реплик

```bash
kubectl scale deployment netology-deployment --replicas=2
kubectl get pods -l app=netology-app -o wide
```

**Для отчёта:** скриншот двух работающих Pod.

### 3. Создание Service

```bash
kubectl apply -f task1-service.yaml
kubectl get svc netology-service
kubectl get endpointslice -l kubernetes.io/service-name=netology-service
```

Service публикует:

- `80/TCP` → nginx;
- `8080/TCP` → multitool.

### 4. Проверка доступа из отдельного Pod

```bash
kubectl apply -f task1-client.yaml
kubectl get pod multitool-client
```

Проверка nginx:

```bash
kubectl exec multitool-client -- curl -s http://netology-service:80
```

Проверка multitool:

```bash
kubectl exec multitool-client -- curl -s http://netology-service:8080
```

**Для отчёта:** скриншот успешных ответов обеих команд `curl`.

---

## Задание 2. Init-контейнер и запуск nginx после появления Service

Манифесты:

- [Deployment](task2-deployment.yaml)
- [Service](task2-service.yaml)

Init-контейнер `busybox` проверяет DNS-имя `nginx-init-service.default.svc.cluster.local`. Пока объект Service не создан, init-контейнер не завершается и основной контейнер nginx не запускается.

### 1. Сначала запускаем только Deployment

```bash
kubectl apply -f task2-deployment.yaml
kubectl get pods -l app=nginx-init-app
```

Ожидаемое состояние Pod:

```text
Init:0/1
```

Логи init-контейнера:

```bash
kubectl logs deployment/nginx-init-deployment -c wait-for-service
```

В логах должно повторяться:

```text
Waiting for nginx-init-service...
```

**Для отчёта:** скриншот Pod в состоянии `Init:0/1` и логов init-контейнера.

### 2. Создаём Service

```bash
kubectl apply -f task2-service.yaml
kubectl get svc nginx-init-service
kubectl get pods -l app=nginx-init-app -w
```

После обнаружения Service init-контейнер завершится, а nginx перейдёт в `Running`. После появления `1/1 Running` выйти из `-w` через `Ctrl+C`.

Проверка:

```bash
kubectl get pods -l app=nginx-init-app
kubectl logs deployment/nginx-init-deployment -c wait-for-service
kubectl exec multitool-client -- curl -s http://nginx-init-service
```

**Для отчёта:** скриншот Pod в состоянии `1/1 Running` и успешного `curl`.

---

## Очистка стенда после выполнения

```bash
kubectl delete -f task1-client.yaml
kubectl delete -f task1-service.yaml
kubectl delete deployment netology-deployment

kubectl delete -f task2-service.yaml
kubectl delete -f task2-deployment.yaml
```
