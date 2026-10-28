## Тепер корисне: прочитай помилку pull

Потрібні read-права на Pod. Значення Secret виводити не потрібно.

### 1. Подія та ім’я образу

```bash
export NAMESPACE='<NAMESPACE>'
export POD_NAME='<POD_NAME>'
kubectl describe pod "$POD_NAME" -n "$NAMESPACE"
kubectl get pod "$POD_NAME" -n "$NAMESPACE" \
  -o jsonpath='{range .spec.containers[*]}{.name}{"\t"}{.image}{"\n"}{end}{.spec.imagePullSecrets[*].name}{"\n"}{.spec.serviceAccountName}{"\n"}'
```

Шукай конкретну причину в events: unauthorized, denied, manifest unknown, timeout, TLS чи помилку архітектури. Логи застосунку можуть бути відсутні, оскільки контейнер ще не запускався. Не замінюй точну подію припущенням за статусом.

### 2. Перевір лише тип і наявність Secret

Якщо кластер використовує imagePullSecrets і посилання вже відоме:

```bash
export PULL_SECRET_NAME='<PULL_SECRET_NAME>'
kubectl get secret "$PULL_SECRET_NAME" -n "$NAMESPACE" \
  -o custom-columns=NAME:.metadata.name,TYPE:.type
```

Наявність об’єкта не підтверджує правильність registry URL, credentials чи дозволу pull. Не виводь Secret через -o yaml у загальні логи. Оновлюй його через прийнятий secret-management процес; не вставляй пароль у командний рядок.

### 3. Перевір результат

Виправ image reference або конфігурацію доступу в декларативному джерелі. Потім перевір events, imageID, readiness і клієнтський запит. Попередній image digest та конфігурація потрібні для rollback, але доступ ноди до попереднього registry все одно має працювати.

[Kubernetes: images та ImagePullBackOff](https://kubernetes.io/docs/concepts/containers/images/)
