## Тепер корисне: причина завершення та межа

Потрібні kubectl, правильний context і read-доступ. Команди не перезапускають workload.

### 1. Прочитай статус конкретного контейнера

```bash
export NAMESPACE='<NAMESPACE>'
export POD_NAME='<POD_NAME>'
export CONTAINER_NAME='<CONTAINER_NAME>'
kubectl get pod "$POD_NAME" -n "$NAMESPACE" -o jsonpath='{range .status.containerStatuses[*]}{.name}{"\t"}{.restartCount}{"\t"}{.lastState.terminated.reason}{"\t"}{.lastState.terminated.exitCode}{"\n"}{end}'
kubectl logs "$POD_NAME" -n "$NAMESPACE" -c "$CONTAINER_NAME" \
  --previous --tail=100
```

lastState стосується попереднього запуску контейнера в цьому Pod; після заміни Pod старі дані можуть бути недоступні через цей запит. Exit code 137 сам по собі не доводить OOM: це також можливий наслідок іншого SIGKILL.

### 2. Перевір ресурси та події

```bash
kubectl get pod "$POD_NAME" -n "$NAMESPACE" \
  -o jsonpath='{range .spec.containers[*]}{.name}{"\t"}{.resources}{"\n"}{end}'
kubectl describe pod "$POD_NAME" -n "$NAMESPACE"
```

Зістав причину, limit і історичні метрики. Поточний знімок споживання після restart міг уже не містити піку. Якщо є metrics API, kubectl top корисний для поточного стану, але не замінює історію.

### 3. Перевір зміни під навантаженням

Ресурси змінюй у декларативному джерелі після оцінки місткості нод. Зміна може запустити rollout; урахуй доступність реплік. Після неї перевір рестарти, використання пам’яті та виконання завдань за репрезентативний інтервал. Відкат конфігурації повертає limit, але не відновлює перервані операції.

[Ресурси контейнерів у Kubernetes](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)
