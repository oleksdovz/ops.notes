## Тепер корисне: перевір весь шлях

На Linux-хості потрібен фактичний source bind mount. Приклад не використовує шлях з оригінального кейсу.

### 1. Права, власник і mount

```bash
export DATA_DIRECTORY='<HOST_DATA_DIRECTORY>'
namei -l "$DATA_DIRECTORY/database.sqlite"
stat -c 'mode=%a owner=%u group=%g path=%n' "$DATA_DIRECTORY"
findmnt --target "$DATA_DIRECTORY" -o TARGET,SOURCE,FSTYPE,OPTIONS
```

`namei` дозволяє побачити відсутній search permission у будь-якому батьківському каталозі. `stat` фіксує числових власника й групу. `findmnt` допоможе виявити read-only mount: chmod не перетворює його на read-write.

### 2. Зістав із контейнером

```bash
export CONTAINER_NAME='<CONTAINER_NAME>'
docker inspect "$CONTAINER_NAME" --format 'ConfiguredUser={{.Config.User}}'
docker inspect "$CONTAINER_NAME" --format '{{json .Mounts}}'
```

Порожній `Config.User` не визначає користувача всіх процесів: entrypoint може змінити UID. Перевір фактичний UID/GID процесу способом, доступним у конкретному образі. Для ACL, SELinux і user namespaces потрібна окрема перевірка — звичайні mode bits не описують усю модель доступу.

### 3. Найменше виправлення після підтвердження

Якщо підтверджено, що саме власнику каталогу бракує `x`, запиши поточний mode. Приклад змінює лише один каталог:

```bash
export ORIGINAL_MODE='<ORIGINAL_MODE>'
sudo chmod u+x -- "$DATA_DIRECTORY"
```

Це вже зміна доступу. Для групового доступу рішення буде іншим; не застосовуй рекурсивний `777`. Повтори операцію застосунку й перевір журнал. Якщо зміну потрібно відкотити, поверни зафіксований mode:

```bash
sudo chmod "$ORIGINAL_MODE" -- "$DATA_DIRECTORY"
```

Повернення mode не скасовує файли, які застосунок встиг створити.

[namei: розбір компонентів шляху](https://man7.org/linux/man-pages/man1/namei.1.html)
