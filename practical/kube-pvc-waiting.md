## Тепер корисне: перевір узгодження Pod і storage

Read-only перевірки потребують доступу до PVC, Pod та StorageClass. Приклад не припускає конкретного CSI-драйвера.

### 1. Стан заявки й події

```bash
export NAMESPACE='<NAMESPACE>'
export CLAIM_NAME='<CLAIM_NAME>'
export POD_NAME='<POD_NAME>'
kubectl describe pvc "$CLAIM_NAME" -n "$NAMESPACE"
kubectl get pvc "$CLAIM_NAME" -n "$NAMESPACE" \
  -o jsonpath='{.spec.storageClassName}{"\t"}{.status.phase}{"\t"}{.spec.volumeName}{"\n"}'
```

Waiting for first consumer відрізняється від помилки provisioner. Якщо немає Pod, який використовує цей claim, саме очікування може бути правильним результатом. Якщо є помилка драйвера, квоти чи доступу, діагностуй її окремо.

### 2. Режим класу й обмеження Pod

```bash
export STORAGE_CLASS_NAME='<STORAGE_CLASS_NAME>'
kubectl get storageclass "$STORAGE_CLASS_NAME" \
  -o jsonpath='{.provisioner}{"\t"}{.volumeBindingMode}{"\n"}'
kubectl get pod "$POD_NAME" -n "$NAMESPACE" \
  -o jsonpath='{.spec.nodeName}{"\n"}{.spec.nodeSelector}{"\n"}{.spec.affinity}{"\n"}'
kubectl describe pod "$POD_NAME" -n "$NAMESPACE"
```

Заповнений nodeName у живому Pod сам по собі не доводить ручне призначення: scheduler теж записує це поле. Звір вихідний маніфест або template, а не лише поточний стан. Перевір також, що claimName у Pod відповідає потрібному PVC.

### 3. Зміна й результат

Виправ спосіб планування у джерелі Deployment/StatefulSet чи іншого власника Pod. Зміна може потребувати rollout або перестворення Pod; перед цим визнач стан даних і вплив на доступність. PVC для такого виправлення навмання не видаляй.

Результат: правильна нода, PVC Bound, успішний mount і доступ до потрібних даних. Для повернення збережи попередню ревізію workload; повернення маніфесту не переносить уже створений диск між зонами чи нодами.

[StorageClass та WaitForFirstConsumer](https://kubernetes.io/docs/concepts/storage/storage-classes/)
