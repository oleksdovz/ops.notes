## Тепер корисне: розгортання на Debian

Це нова лабораторна інсталяція з sudo, systemd та окремим mount `/srv/minio`. На чинному сервері спочатку звір unit, версії та шляхи; наведені файли не накладай поверх робочих. У всіх блоках заміни placeholders перед виконанням. Секрети створи в password manager або штатному secret manager; їхні значення не потрібні в shell history.

### 1. Перевір диск і архітектуру

На сервері, без змін:

```bash
cat /etc/os-release
uname -m
findmnt --mountpoint /srv/minio
df -hT /srv/minio
df -i /srv/minio
```

Потрібний саме очікуваний mount, достатній вільний простір та inodes. Якщо mount відсутній, спочатку налаштуй постійне монтування. Не створюй data directory на root filesystem замість диска: після наступного reboot це буде окремий, дуже творчий експеримент.

### 2. Збери й зафіксуй артефакти

На окремому Linux build-хості встанови git, CA certificates і Go версії, яку вимагає go.mod обраної ревізії. Архітектура результату має відповідати серверу. Передавання й перевірка походження артефактів — частина твого deployment-процесу.

У новому build-каталозі, не в наявному checkout:

```bash
export MINIO_COMMIT='<REVIEWED_40_CHAR_MINIO_COMMIT>'
export MC_COMMIT='<REVIEWED_40_CHAR_MC_COMMIT>'
git clone https://github.com/minio/minio.git minio-src
git -C minio-src checkout --detach "$MINIO_COMMIT"
git -C minio-src rev-parse HEAD
git clone https://github.com/minio/mc.git mc-src
git -C mc-src checkout --detach "$MC_COMMIT"
git -C mc-src rev-parse HEAD
```

Звір обидва SHA з погодженими ревізіями. Після цього перевір go.mod і збирай; тут приклад для build-хоста тієї самої архітектури:

```bash
mkdir -p artifacts
(cd minio-src && go build -o ../artifacts/minio .)
(cd mc-src && go build -o ../artifacts/mc .)
./artifacts/minio --version
./artifacts/mc --version
(cd artifacts && sha256sum minio mc > SHA256SUMS)
```

Ревізії та toolchain запиши в deployment manifest. Checksum поруч з артефактом виявляє пошкодження, але не доводить довіру до джерела. Збірка архівного коду не додає йому супроводу чи security fixes.

На сервері в каталозі отриманих і довірених артефактів:

```bash
sha256sum -c SHA256SUMS &&
  sudo install -m 0755 minio mc /usr/local/bin/
```

### 3. Підготуй користувача та нові каталоги

Виконуй лише після підтвердження mount. Якщо користувач уже є, перевір його UID, home та групу замість повторного useradd. Існуючі data directory не перепризначай рекурсивним chown.

```bash
sudo useradd --system --user-group --shell /usr/sbin/nologin \
  --home-dir /var/lib/minio minio-user
sudo install -d -o minio-user -g minio-user -m 0750 \
  /var/lib/minio /srv/minio/data
sudo install -d -o root -g root -m 0700 /etc/minio
sudo install -o root -g root -m 0600 /dev/null /etc/minio/minio.env
sudoedit /etc/minio/minio.env
```

Останній install створює новий порожній файл; на наявному env його не запускай — він затре вміст. Заповнення файла й unit показане нижче. Root-пароль отримай із secret manager, не залишай placeholder і не публікуй файл. Permission 0600 не є шифруванням; root і процес сервісу все одно мають доступ до потрібних секретів.

### 4. Запусти й перевір сервіс

Після створення unit з наступного розділу:

```bash
sudo systemd-analyze verify /etc/systemd/system/minio.service &&
  sudo systemctl daemon-reload &&
  sudo systemctl enable --now minio
systemctl status minio --no-pager
sudo journalctl -u minio -n 60 --no-pager
sudo ss -lntp '( sport = :9000 or sport = :9001 )'
```

Очікуй active, потрібний mount і listener лише на 127.0.0.1. Помилки журналу перевір локально; перед передаванням прибери credentials та впізнавані endpoints.

```bash
curl --fail --silent --show-error --output /dev/null \
  --write-out 'live: %{http_code}\n' \
  http://127.0.0.1:9000/minio/health/live
curl --fail --silent --show-error --output /dev/null \
  --write-out 'ready: %{http_code}\n' \
  http://127.0.0.1:9000/minio/health/ready
```

HTTP 200 показує відповідь відповідного health endpoint. Він не перевіряє права backup job, наявність потрібних об’єктів або успішний restore.

[Health endpoints MinIO](https://github.com/minio/minio/blob/master/docs/metrics/healthcheck/README.md)

## Як користуватися: bucket і окремий backup-користувач

### 1. Додай admin alias

На сервері виконуй mc під окремим адміністративним Linux-користувачем. Команда запитає access key та приховано secret key в інтерактивному terminal; не передавай їх аргументами й не додавай --json або --debug до credential-команд.

```bash
umask 077
mc alias set minio-admin http://127.0.0.1:9000
mc admin info minio-admin
mc mb --ignore-existing minio-admin/backup
```

mc зберігає ключі у своєму локальному config у home. Не додавай цей файл до Git чи незахищених копій. Для іншого хоста використовуй HTTPS endpoint із довіреним сертифікатом, а не відкритий HTTP. Успіх admin info перевіряє admin-доступ, ще не права майбутнього job.

[Інтерактивне введення mc alias set](https://github.com/minio/mc/blob/master/cmd/alias-set.go)

### 2. Створи користувача й прикріпи policy

Збережи JSON policy з наступного розділу як `backup-writer.json` на admin-хості. Створи нову окрему identity; при повторному виконанні user add не використовуй ім’я чинного користувача навмання — можна змінити його пароль.

```bash
mc admin policy create minio-admin backup-writer backup-writer.json
mc admin user add minio-admin
```

В інтерактивних полях введи access key `backup-writer` і новий secret key із password manager. Збережи його там для job. Потім:

```bash
mc admin policy attach minio-admin backup-writer --user backup-writer
mc admin user info minio-admin backup-writer
```

Policy дозволяє читання й запис об’єктів лише в backup bucket; видалення не дозволене. Це зменшує права, але PutObject усе одно дозволяє перезаписати відомий ключ. Не називай такий доступ immutable. Для backup-програми, якій потрібні prune, locks або інші дії, перевір її S3-вимоги й налаштуй окрему policy — ця не універсальна.

[Інтерактивне створення користувача](https://github.com/minio/mc/blob/master/cmd/admin-user-add.go)

### 3. Перевір саме identity job

На клієнтському хості з mc зафіксованої версії додай новий alias, ввівши ключі backup-writer. Підстав свій HTTPS endpoint; у лабораторії на самому сервері можна використати loopback URL.

```bash
mc alias set backup-job https://s3.example.com
mc ls backup-job/backup
mc admin info backup-job
```

List дозволеного bucket має працювати, admin info — завершитися AccessDenied. Несподіваний успіх admin-команди означає, що треба перевірити identity та всі її прямі й групові policies. Відмова на одному admin API ще не доводить відсутність усіх зайвих прав.

### 4. Завантаж і поверни тестовий файл

На клієнті створюється окремий локальний каталог і унікальний object key. Цей тест не перезаписує існуючий backup:

```bash
CHECK_DIR="$(mktemp -d)"
OBJECT_KEY="checks/$(date -u +%Y%m%dT%H%M%SZ)-$(basename "$CHECK_DIR").txt"
printf 'restore check\n' > "$CHECK_DIR/source.txt"
mc cp "$CHECK_DIR/source.txt" "backup-job/backup/$OBJECT_KEY" &&
  mc cp "backup-job/backup/$OBJECT_KEY" "$CHECK_DIR/restored.txt" &&
  cmp "$CHECK_DIR/source.txt" "$CHECK_DIR/restored.txt"
```

cmp без виводу з exit code 0 означає однаковий вміст. Тестовий об’єкт залишається в bucket: cleanup виконує адміністратор за точним ключем, бо job не має DeleteObject. Для реального бекапу повтори перевірку з архівом та відновленням застосунку в ізольованому середовищі.

### 5. Копіюй лише підготовлені дані

Для статичного, вже підготовленого каталогу спочатку подивись план mirror:

```bash
export BACKUP_SOURCE='<PREPARED_BACKUP_DIRECTORY>'
mc mirror --dry-run "$BACKUP_SOURCE/" backup-job/backup/snapshots/run-001/
```

Після перевірки списку виконай копіювання в новий prefix:

```bash
mc mirror "$BACKUP_SOURCE/" backup-job/backup/snapshots/run-001/
```

`run-001` — приклад нового ідентифікатора запуску. Наступного разу використовуй новий prefix. Не додавай автоматично --overwrite або --remove. Не копіюй усі приховані каталоги home за принципом «зайве не завадить»: там живуть клієнтські credentials. Mirror не зберігає весь стан POSIX filesystem як системний backup; потрібні metadata й консистентність забезпечує обраний backup-інструмент.

[mc mirror: прапорці та режим dry-run](https://github.com/minio/mc/blob/master/cmd/mirror-main.go)
