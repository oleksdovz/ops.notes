### Було: приватний образ без посилання на credentials

Фрагмент `spec.template.spec` workload. Це не повний Deployment і не інструкція вимикати автентифікацію.

```yaml
containers:
  - name: app
    image: registry.example.com/team/app:<IMAGE_TAG>
```

### Стало: використовується наявний pull Secret

```yaml
imagePullSecrets:
  - name: <PULL_SECRET_NAME>
containers:
  - name: app
    image: registry.example.com/team/app@sha256:<IMAGE_DIGEST>
```

Secret має існувати в тому самому namespace, а його credentials — давати pull-доступ саме до цього registry. Створи або онови його через прийнятий secret-management процес; значень Secret у прикладі немає. Ресурси, probes і securityContext контейнера тут опущені, а не запропоновані до видалення.

Цей фікс відповідає підтвердженій відсутності credentials. Для неправильного image reference чи мережевого timeout потрібна інша зміна. Після rollout звір imageID і readiness. Повернення до попередньої ревізії теж потребує доступу до попереднього образу.
