### Було: Service шукає іншу мітку

Два окремі фрагменти: `spec` Service та `spec.template.metadata` Deployment. Ім’я app — умовне, решту маніфестів треба зберегти.

```yaml
# Service: spec
selector:
  app: web-old
ports:
  - port: 80
    targetPort: http
---
# Deployment: spec.template.metadata
labels:
  app: web
```

### Стало: selector відповідає міткам Pod

```yaml
# Service: spec
selector:
  app: web
ports:
  - port: 80
    targetPort: http
```

У контейнера має бути named port `http` із фактичним портом застосунку. Якщо його немає — використай перевірений числовий targetPort. Не змінюй selector самого Deployment за цим прикладом: для нього діють окремі обмеження.

Після правки Service перевір EndpointSlice й клієнтський запит. Selector може одразу перемкнути трафік; для відкату поверни його попередню ревізію. У GitOps змінюй джерело, а не лише живий об’єкт.
