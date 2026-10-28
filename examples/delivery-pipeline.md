### Було: етап закінчується після відправлення маніфесту

Фрагмент steps для GitHub Actions. Context, обмежений Kubernetes-доступ і змінні середовища мають бути налаштовані попередніми кроками через дозволений механізм; credentials тут не показані.

```yaml
- name: Deploy
  shell: bash
  run: kubectl apply -f "$MANIFEST_PATH"
```

### Стало: доставка та перевірка результату — окремі кроки

Приклад для процесу, який справді керує Kubernetes через цей CI. Якщо станом керує Flux або інший GitOps-контролер, не додавай конкурентний apply: зміни джерело Git і дочекайся узгодження.

```yaml
- name: Deploy
  shell: bash
  run: kubectl apply -f "$MANIFEST_PATH"

- name: Wait for rollout
  shell: bash
  run: |
    set -euo pipefail
    kubectl rollout status "deployment/$DEPLOYMENT_NAME" \
      -n "$NAMESPACE" --timeout=120s

- name: Check read-only scenario
  shell: bash
  run: |
    set -euo pipefail
    curl --fail --silent --show-error --max-time 10 \
      "$SMOKE_TEST_URL" -o response.json
    jq -e --arg expected "$EXPECTED_RELEASE" \
      '.release == $expected' response.json >/dev/null
```

Тут припускається, що захищений від витоку даних read-only endpoint повертає JSON із полем release. Підстав контракт свого застосунку; jq та curl мають бути встановлені. HTTP 200 без потрібного поля більше не завершує цей тест успіхом.

Apply змінює кластер і може спричинити rollout. Зберігай попередню ревізію та перевір сумісність міграцій бази до deploy. Автоматичний rollback за будь-якого timeout не доданий: спочатку треба визначити, коли повернення безпечне для цього сервісу.
