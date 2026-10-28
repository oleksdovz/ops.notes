## Тепер корисне: selector → Pod → endpoints

Команди читають об’єкти поточного кластера. Спочатку перевір context і namespace.

### 1. Зістав мітки

```bash
export NAMESPACE='<NAMESPACE>'
export SERVICE_NAME='<SERVICE_NAME>'
kubectl config current-context
kubectl get service "$SERVICE_NAME" -n "$NAMESPACE" \
  -o jsonpath='{.spec.selector}{"\n"}{.spec.ports}{"\n"}'
kubectl get pods -n "$NAMESPACE" --show-labels
```

Selector має вибирати саме потрібні Pod. Коли перенесеш його в kubectl get pods -l, використовуй фактичні ключі й значення. Service без selector потребує іншого розбору: відсутність selector там може бути свідомою конфігурацією.

### 2. Прочитай адреси та готовність

```bash
kubectl get endpointslices.discovery.k8s.io -n "$NAMESPACE" \
  -l "kubernetes.io/service-name=$SERVICE_NAME" -o yaml
```

Зістав addresses, ports і conditions.ready із потрібними Pod. Наявність EndpointSlice не означає наявність придатних backend-ів. Нестандартні параметри Service, зокрема publishNotReadyAddresses, змінюють звичайну поведінку готовності.

### 3. Виправ джерело конфігурації

Правку Service або Pod labels роби в Helm/Kustomize чи іншому джерелі декларативного стану. У GitOps ручний patch може бути скасований контролером. Збережи попередню ревізію; неправильний selector здатен переключити живий трафік.

Після зміни перевір endpoints і запит із дозволеного клієнта всередині кластера. Успішний port-forward до одного Pod обходить Service й не закриває цей тест.

[Діагностика Kubernetes Service](https://kubernetes.io/docs/tasks/debug/debug-application/debug-service/)
