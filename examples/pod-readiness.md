### Було: контейнер слухає порт, готовність не перевіряється

Фрагмент одного елемента `spec.template.spec.containers`:

```yaml
name: app
image: <IMAGE_REFERENCE>
ports:
  - name: http
    containerPort: 8080
```

### Стало: трафік лише після успішної readiness

```yaml
name: app
image: <IMAGE_REFERENCE>
ports:
  - name: http
    containerPort: 8080
readinessProbe:
  httpGet:
    path: /ready
    port: http
  periodSeconds: 5
  timeoutSeconds: 2
  failureThreshold: 3
```

Цифри та `/ready` — приклад. Застосунок має реально підтримувати цей endpoint на цьому порту; відповідь повинна відображати можливість приймати запити. Решту контейнерної конфігурації, включно з ресурсами та securityContext, збережи.

Readiness не перезапускає контейнер. Startup і liveness за потреби налаштовуються окремо. Неправильна readiness може прибрати всі backend-и з трафіку, тому перевір її в тестовому середовищі й повертай попередню ревізію, якщо endpoint визначений неправильно.
