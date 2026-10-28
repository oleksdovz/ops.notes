### Було: workload посилається не на той claim

Цей варіант фіксу підходить лише якщо перевірка довела помилку в claimName, а правильний PVC уже існує в тому самому namespace та містить потрібні дані.

```yaml
# spec.template.spec, фрагмент workload
volumes:
  - name: data
    persistentVolumeClaim:
      claimName: <WRONG_CLAIM_NAME>
```

### Стало: посилання на перевірений PVC

```yaml
# spec.template.spec, фрагмент workload
volumes:
  - name: data
    persistentVolumeClaim:
      claimName: <EXISTING_VERIFIED_CLAIM_NAME>
```

VolumeMount у контейнері повинен і далі посилатися на volume `data`, а не на ім’я PVC. Змінюється джерело даних, тому перед rollout перевір власника volume, дані, access mode й допустимість одночасного доступу старого та нового Pod.

Якщо потрібного claim немає, ця правка не є фіксом: спочатку знайди старий volume або підготуй відновлення. Створення нового порожнього PVC тут навмисно не показане як універсальне рішення. Для повернення потрібна попередня ревізія та перевірка, що застосунок не записав несумісний стан.
