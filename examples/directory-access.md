### Було: каталог без права проходу

Умовний фрагмент виводу, не лог з реальної інсталяції:

```text
drwxr-xr-x  <OWNER> <GROUP> parent
drw-rw----  <OWNER> <GROUP> app-data
-rw-rw----  <OWNER> <GROUP> database.sqlite
```

Проблемний рядок — app-data: потрібний процес не може пройти через каталог. Права файла в останньому рядку цього не виправлять.

### Стало: потрібний користувач отримав search permission

Лише якщо встановлено, що застосунок працює саме як власник каталогу:

```bash
export DATA_DIRECTORY='<HOST_DATA_DIRECTORY>'
export ORIGINAL_MODE='<ORIGINAL_MODE>'
sudo chmod u+x -- "$DATA_DIRECTORY"
namei -l "$DATA_DIRECTORY/database.sqlite"
```

Для доступу через групу потрібна інша, підтверджена правка. Не копіюй рекурсивні chmod/chown із цього прикладу — їх тут немає. Перевір реальну операцію застосунку; mode bits не обходять read-only mount, ACL чи SELinux.

Відкат до зафіксованого mode:

```bash
sudo chmod "$ORIGINAL_MODE" -- "$DATA_DIRECTORY"
```
