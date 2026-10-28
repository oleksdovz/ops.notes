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
