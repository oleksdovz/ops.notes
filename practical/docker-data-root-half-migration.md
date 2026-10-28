## Тепер корисне: інвентаризація перед переїздом

Команди нижче — шаблони для Linux Docker Engine, не Docker Desktop. На серверах із rootless Docker або нестандартним config path спочатку уточни фактичний спосіб запуску daemon. Ці перевірки тут не виконувалися на твоєму хості.

### 1. Встанови, що саме зберігає дані

```bash
docker version --format 'Server={{.Server.Version}}'
docker info --format 'Root={{.DockerRootDir}} Driver={{.Driver}} DriverStatus={{json .DriverStatus}}'
sudo containerd config dump | awk '/^root =/ || /^state =/ {print}'
```

Перший вивід фіксує версію Engine. Другий — Docker Root Dir та ознаки storage backend. `DriverStatus` може містити `io.containerd.snapshotter.v1` для containerd image store; перевір доступність цього поля й інтерпретацію для своєї версії. Шлях containerd у config dump не доводить, що саме Docker використовує цей instance.

Якщо служба containerd запускається з іншим `--config`, dump треба отримати з тим самим config path. Відсутня команда containerd також не означає, що всі дані лежать у Docker Root Dir.

### 2. Перевір цільовий mount

```bash
export TARGET_ROOT='<TARGET_DATA_DIRECTORY>'
findmnt --target "$TARGET_ROOT" -o TARGET,SOURCE,FSTYPE,OPTIONS
df -h "$TARGET_ROOT"
df -i "$TARGET_ROOT"
```

`findmnt` показує файлову систему, яка фактично покриває шлях. Якщо очікував окремий диск, а бачиш кореневу файлову систему, перенесення ще не починай. Перевір вільні байти, inode та підтримку потрібних прав, ACL і extended attributes.

### 3. План зміни та повернення

Перед копіюванням: зафіксуй кількість контейнерів, образів і volumes; окремо врахуй bind mounts. Збережи конфігурацію daemon та старі каталоги. Сплануй простій і зупини компоненти, які пишуть у дані; копія активного сховища не стає узгодженою лише через rsync.

Довготривалі дані та runtime state не об’єднуй: `state` containerd під `/run` залишається runtime-станом. Конкретні правки залежать від активного storage backend; універсального apply-блока тут немає.

### 4. Що вважати результатом

Після зміни перевір Docker Root Dir, mounts, наявність очікуваних образів, контейнерів і volumes. Потім — читання старих даних та потрібну операцію застосунку. `docker ps` без помилок не перевіряє базу.

Відкат: зупинити відповідні служби, повернути попередню конфігурацію та старе узгоджене сховище. Якщо після переїзду вже були записи, стара копія відстала: спочатку потрібне узгодження даних, а не просте перемикання шляху.

[Docker daemon: data directory та containerd image store](https://docs.docker.com/engine/daemon/)

## Конфіг і фікс: конкретний приклад

Нижче — приклад, не опис виконаних змін у кейсі. Умови: rootful Docker Engine на Linux із systemd; Docker використовує окремий системний containerd; його сховище справді потрібно переносити. Цей containerd не обслуговує Kubernetes чи сторонні workloads. Поточні шляхи — `/var/lib/docker` і `/var/lib/containerd`, ціль — окремий змонтований носій `/mnt/container-storage`. Усі шляхи прикладові: перед виконанням звір їх із сервером.

Якщо в тебе лише classic storage driver і немає даних цього Docker в окремому containerd store, гілка перенесення containerd не потрібна. Не змінюй backend або версію формату TOML одночасно з переміщенням даних.

### 1. Що саме змінюється

Docker: `/etc/docker/daemon.json`. Це мінімальний приклад файла; якщо параметрів більше, збережи їх і зміни лише data-root.

До — для випадку, коли файл існує та порожній за налаштуваннями:

```json
{}
```

Після:

```json
{
  "data-root": "/mnt/container-storage/docker"
}
```

containerd: `/etc/containerd/config.toml`. Це лише верхньорівневий фрагмент, а не заміна всього файла. Збережи його version, imports та всі plugins. Параметри мають бути до першої TOML-таблиці, не всередині `[plugins...]`.

До:

```toml
root = "/var/lib/containerd"
state = "/run/containerd"
```

Після:

```toml
root = "/mnt/container-storage/containerd"
state = "/run/containerd"
```

Переміщуємо постійні дані root. Runtime state під `/run` лишається на місці. Також перевір, чи root або state не перевизначені прапорцями запуску служби.

### 2. Перевір носій і збережи конфігурацію

Приклад передбачає, що обидва конфігураційні файли вже існують. Якщо ні — не запускай cp навмання: зафіксуй відсутність файла та фактичні defaults окремо. Для кожної спроби використовуй новий каталог backup.

```bash
export STORAGE_MOUNT='/mnt/container-storage'
export BACKUP_DIRECTORY='<NEW_ROOT_ONLY_BACKUP_DIRECTORY>'
findmnt --mountpoint "$STORAGE_MOUNT" -o TARGET,SOURCE,FSTYPE,OPTIONS
df -h "$STORAGE_MOUNT"
df -i "$STORAGE_MOUNT"
sudo install -d -m 0700 "$BACKUP_DIRECTORY"
sudo cp -a /etc/docker/daemon.json "$BACKUP_DIRECTORY/daemon.json"
sudo cp -a /etc/containerd/config.toml "$BACKUP_DIRECTORY/config.toml"
```

Продовжуй лише якщо findmnt підтверджує саме окремий цільовий mount, а місткості й можливостей файлової системи достатньо. Якщо команда не знайшла mount, не створюй на його місці дерево даних на кореневому диску.

### 3. Зупини записи перед копіюванням

Далі буде простій. Спочатку коректно зупини застосунки їхнім звичайним способом, особливо СУБД. Перевір, що не лишилося працюючих контейнерів або сторонніх завдань цього containerd. Live restore чи shims можуть залишити процеси активними навіть після зупинки daemon.

Після зупинки workloads:

```bash
sudo systemctl stop docker.service docker.socket
sudo systemctl stop containerd.service
sudo systemctl is-active docker.service docker.socket containerd.service
```

Очікувано всі три units неактивні; is-active для такого результату поверне ненульовий exit code. Якщо units називаються інакше або є інші споживачі runtime — спочатку уточни topology. Сам статус daemon не замінює перевірку відсутності записувачів.

### 4. Скопіюй у нові порожні каталоги

Приклад передбачає порожні цільові каталоги. rsync потрібен із підтримкою ACL та extended attributes; носій теж має їх підтримувати. Копіюємо вміст каталогів, тому завершальні `/` важливі.

```bash
sudo install -d -m 0700 "$STORAGE_MOUNT/docker" "$STORAGE_MOUNT/containerd"
sudo rsync -aHAXS --numeric-ids /var/lib/docker/ "$STORAGE_MOUNT/docker/"
sudo rsync -aHAXS --numeric-ids /var/lib/containerd/ "$STORAGE_MOUNT/containerd/"
```

Збережено owners, groups, modes, hard links, ACL, xattrs та sparse files. Не ігноруй ненульовий exit code. Старі каталоги не видаляємо. Якщо всередині source є окремі mounts, заздалегідь визнач, чи мають вони копіюватися: rsync за цим прикладом перетинає їх.

Поки служби зупинені, перевір копію без змін:

```bash
sudo rsync -aHAXScn --numeric-ids --itemize-changes \
  /var/lib/docker/ "$STORAGE_MOUNT/docker/"
sudo rsync -aHAXScn --numeric-ids --itemize-changes \
  /var/lib/containerd/ "$STORAGE_MOUNT/containerd/"
```

Порожній вивід за успішного завершення означає, що rsync не знайшов змін за цими правилами, включно з checksum файлів. Це не перевірка відновлення бази й не пошук зайвих файлів у destination. Для великих сховищ checksum-прохід може бути довгим.

### 5. Зміни конфіг і прив’яжи запуск до mount

Тепер відредагуй конфігурацію за фрагментами вище. Не перезаписуй інші налаштування. Перевір синтаксис без запуску daemon:

```bash
sudo dockerd --validate --config-file=/etc/docker/daemon.json
sudo containerd --config /etc/containerd/config.toml config dump \
  | awk '/^root =/ || /^state =/ {print}'
```

Для Docker очікується успішна валідація. У filtered dump containerd — новий root та попередній state. Помилку config dump у stderr не ігноруй; exit code awk не є доказом успіху попередньої команди. Ці перевірки не доводять сумісність mount, прав і даних після старту.

Щоб служби не стартували без потрібного mount, додай окремий drop-in для кожної. Файл `/etc/systemd/system/docker.service.d/storage.conf`:

```ini
[Unit]
RequiresMountsFor=/mnt/container-storage/docker
```

Файл `/etc/systemd/system/containerd.service.d/storage.conf`:

```ini
[Unit]
RequiresMountsFor=/mnt/container-storage/containerd
```

Сам mount має бути правильно оголошений у fstab або mount unit. Drop-in не підключає довільний диск за однією назвою каталогу. Якщо файл storage.conf уже існує, збережи його попередній вміст до редагування.

Після збереження файлів:

```bash
sudo systemctl daemon-reload
sudo systemctl start containerd.service
sudo systemctl start docker.service
sudo systemctl is-active containerd.service docker.service
```

Restart policies можуть запустити частину контейнерів автоматично. Тому rollback після цього моменту вже має враховувати можливі нові записи.

### 6. Доведи результат, а не лише запуск daemon

```bash
docker info --format 'DockerRootDir={{.DockerRootDir}}'
docker ps -a --format 'table {{.Names}}\t{{.Status}}\t{{.Image}}'
docker image ls
docker volume ls
```

У Docker Root Dir має бути новий шлях. Список контейнерів, образів і volumes зістав із зафіксованим до зупинки. Для containerd додатково перевір ефективні параметри служби та її журнал; після цього виконай контрольне читання й запис через потрібний застосунок. Не публікуй необроблені журнали або приватні назви образів.

### 7. Повернення до попереднього стану

Цей шлях підходить лише якщо стара копія все ще є прийнятною за даними. Після нових записів спочатку визнач, як їх зберегти або узгодити. Перед поверненням знову коректно зупини workloads, Docker і відповідний containerd.

Віднови попередні файли з власної резервної копії:

```bash
sudo cp -a "$BACKUP_DIRECTORY/daemon.json" /etc/docker/daemon.json
sudo cp -a "$BACKUP_DIRECTORY/config.toml" /etc/containerd/config.toml
sudo dockerd --validate --config-file=/etc/docker/daemon.json
```

Поверни попередні drop-ins. Якщо storage.conf був створений лише для цієї міграції, прибери саме цей доданий файл; не видаляй увесь каталог drop-ins. Потім daemon-reload, запуск containerd та Docker й повторна перевірка застосунків. Команд видалення старих даних у цьому прикладі немає.

[Docker: data-root і окреме сховище containerd](https://docs.docker.com/engine/daemon/)

[containerd: root, state та config path](https://github.com/containerd/containerd/blob/main/docs/ops.md)

[dockerd: перевірка конфігурації](https://docs.docker.com/reference/cli/dockerd/)
