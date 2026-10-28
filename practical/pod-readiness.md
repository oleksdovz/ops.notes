## Тепер корисне: перевір шлях запиту

Потрібні kubectl, правильний context і read-доступ до потрібного namespace. Приклади не припускають установленого Ingress controller чи конкретної версії застосунку.

### 1. Pod, readiness, попередні логи

```bash
export NAMESPACE='<NAMESPACE>'
export POD_NAME='<POD_NAME>'
kubectl config current-context
kubectl get pod "$POD_NAME" -n "$NAMESPACE" -o wide
kubectl describe pod "$POD_NAME" -n "$NAMESPACE"
kubectl logs "$POD_NAME" -n "$NAMESPACE" --all-containers --tail=100
```

Дивись не лише на `STATUS`, а й на `READY`, restarts, conditions і events. Якщо контейнер перезапускався, окремо отримай його попередній журнал через `kubectl logs ... -c <CONTAINER_NAME> --previous`. Відсутність previous logs може означати відсутність попереднього екземпляра.

### 2. Чи є куди направляти трафік

```bash
export SERVICE_NAME='<SERVICE_NAME>'
kubectl get service "$SERVICE_NAME" -n "$NAMESPACE" -o yaml
kubectl get endpointslices.discovery.k8s.io -n "$NAMESPACE" \
  -l "kubernetes.io/service-name=$SERVICE_NAME" -o yaml
```

Зістав selector, targetPort, адреси endpoints і їхню ready condition. Порожній результат — причина перевірити selector та готовність Pod, а не доказ поломки DNS. Доступність EndpointSlice API залежить від кластера й RBAC.

### 3. Зовнішня перевірка

```bash
curl --fail --silent --show-error --max-time 10 \
  'https://<APPLICATION_HOST>/<READ_ONLY_TEST_PATH>'
```

Запит має йти реальним клієнтським маршрутом і перевіряти потрібний сценарій. Якщо endpoint захищений, використай дозволений спосіб автентифікації без секретів у CLI. Порт-форвард перевіряє інший шлях і сам по собі не доводить справність Ingress.

### Якщо змінюєш probes

Спочатку перевір фактичні endpoints і час старту. Readiness прибирає неготовий контейнер із сервісного трафіку; liveness може спричинити restart. Погана liveness здатна перетворити повільний старт на цикл перезапусків. Зберігай попередню конфігурацію в Git; після rollout перевір readiness та клієнтський запит, для повернення застосуй попередню погоджену ревізію.

[Налаштування probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)
