## Тепер корисне: claim, volume, дані

Працюй у правильному context; потрібен read-доступ до Pod/PVC та, для перевірки PV, cluster-scoped доступ.

### 1. Визнач, який claim потрібен

```bash
export NAMESPACE='<NAMESPACE>'
export POD_NAME='<POD_NAME>'
kubectl get pod "$POD_NAME" -n "$NAMESPACE" -o jsonpath='{range .spec.volumes[*]}{.name}{"\t"}{.persistentVolumeClaim.claimName}{"\n"}{end}'
kubectl get pvc -n "$NAMESPACE" -o wide
kubectl describe pod "$POD_NAME" -n "$NAMESPACE"
```

Для volumes інших типів claimName буде порожнім. Відсутній PVC перевіряй у namespace Pod; claim в іншому namespace не задовольняє це посилання.

### 2. Якщо claim існує — переходь до PV

```bash
export CLAIM_NAME='<CLAIM_NAME>'
kubectl get pvc "$CLAIM_NAME" -n "$NAMESPACE" \
  -o jsonpath='{.status.phase}{"\t"}{.spec.volumeName}{"\n"}'
kubectl get pv -o wide
```

Для конкретного bound PV прочитай reclaim policy, storage class, backend та node affinity. `Bound` підтверджує зв’язок об’єктів, але не доступність даних. Для CSI перевірки можуть включати VolumeAttachment; для local PV зміна affinity не переносить вміст диска.

### 3. Перед створенням заміни

Знайди попередній volume й копії даних. Не видаляй finalizers і не перевикористовуй `Released` PV до встановлення його власника та вмісту. Новий порожній claim може запустити сервіс із порожнім станом — scheduler буде задоволений першим.

Після відновлення перевір Pod, mount і потрібні дані через застосунок. Для повернення збережи попередню декларативну конфігурацію й копію стану. Політика `Delete` може видаляти зовнішній ресурс після звільнення; не тестуй rollback видаленням claim.

[Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)
