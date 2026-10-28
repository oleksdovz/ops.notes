### Було: ноду призначено напряму

Фрагмент `spec` Pod або `spec.template.spec` workload із claim, чий StorageClass використовує WaitForFirstConsumer:

```yaml
nodeName: <NODE_NAME>
volumes:
  - name: data
    persistentVolumeClaim:
      claimName: <CLAIM_NAME>
```

### Стало: вимогу передано scheduler

Прибери nodeName з декларативного джерела. Замість нього, якщо прив’язка до цього вузла справді потрібна:

```yaml
nodeSelector:
  kubernetes.io/hostname: <HOSTNAME_LABEL_VALUE>
volumes:
  - name: data
    persistentVolumeClaim:
      claimName: <CLAIM_NAME>
```

Підстав значення мітки `kubernetes.io/hostname` потрібного вузла, а не автоматично його object name. Решту Pod spec збережи. Selector має залишати сумісну ноду з достатніми ресурсами та доступним storage.

Не patch-и nodeName запущеного Pod: зміни джерело workload і сплануй заміну Pod. PVC не видаляй. Після зміни перевір scheduling events, binding і дані. Відкат workload не переміщує вже виділений volume між зонами.
