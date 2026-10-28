## Тепер корисне: умова завершення deploy

Приклад перевірки для Kubernetes Deployment. Якщо платформа інша, збережи послідовність перевірок, а команди заміни відповідними для неї.

### 1. Дочекайся rollout і звір образ

```bash
export NAMESPACE='<NAMESPACE>'
export DEPLOYMENT_NAME='<DEPLOYMENT_NAME>'
kubectl rollout status "deployment/$DEPLOYMENT_NAME" \
  -n "$NAMESPACE" --timeout=120s
kubectl get deployment "$DEPLOYMENT_NAME" -n "$NAMESPACE" \
  -o jsonpath='{range .spec.template.spec.containers[*]}{.name}{"\t"}{.image}{"\n"}{end}'
```

Команди запускай із обмеженим service account і правильним context. У CI невдалий rollout має зупиняти наступний крок, а не губитися в pipeline без перевірки exit code. Образ у spec — бажаний; фактичний imageID перевіряй в containerStatuses відповідних Pod. Для однозначного артефакту використовуй digest.

### 2. Smoke test через клієнтський маршрут

```bash
curl --fail --silent --show-error --max-time 10 \
  'https://<APPLICATION_HOST>/<READ_ONLY_TEST_PATH>'
```

Успіх — це очікуваний статус і зміст відповіді для обраного сценарію, не просто наявність HTTP. `curl --fail` сам по собі не перевіряє JSON, бізнес-дані або правильну версію. Додай перевірку очікуваного поля чи результату у тесті свого застосунку.

### 3. Коли дозволяти повернення

Заздалегідь визнач попередній image digest, ревізію конфігурації та умову запуску rollback. Перевір сумісність старого коду з уже виконаною міграцією бази. Повернення Deployment не повертає дані й не скасовує зовнішні побічні ефекти.

Після rollback повтори rollout status і smoke test. Завершений rollback — теж результат, який треба довести. Інакше pipeline просто стане зеленим удруге.

[kubectl rollout status](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_rollout/kubectl_rollout_status/)
