## Конфігурація сервера

### Env-файл /etc/minio/minio.env

Повний мінімальний файл для цього unit. Значення нижче — placeholders: заміни їх секретами зі сховища до запуску. Не source цей файл у shell; його читає systemd.

```ini
MINIO_ROOT_USER=<MINIO_ADMIN_ACCESS_KEY>
MINIO_ROOT_PASSWORD=<MINIO_ADMIN_SECRET_KEY>
```

При використанні довільних спецсимволів врахуй синтаксис EnvironmentFile; безпечніше доручити render штатному secret-management механізму. Файл належить root:root із mode 0600. Не друкуй його для «швидкої перевірки».

[Змінні root-конфігурації MinIO](https://github.com/minio/minio/blob/master/docs/config/README.md)

### Unit /etc/systemd/system/minio.service

Повний приклад для виділеного mount `/srv/minio`; створи файл через sudoedit. Якщо mount або data path інші, узгодь усі відповідні рядки. Unit не форматує диск і не виконує міграцію даних.

```ini
[Unit]
Description=MinIO single-node object storage
Wants=network-online.target
After=network-online.target
RequiresMountsFor=/srv/minio

[Service]
Type=simple
User=minio-user
Group=minio-user
Environment=HOME=/var/lib/minio
EnvironmentFile=/etc/minio/minio.env
WorkingDirectory=/var/lib/minio
ExecStartPre=/usr/bin/mountpoint -q /srv/minio
ExecStart=/usr/local/bin/minio server --address 127.0.0.1:9000 --console-address 127.0.0.1:9001 /srv/minio/data
Restart=on-failure
RestartSec=5
TimeoutStopSec=300
LimitNOFILE=65536
UMask=0027
NoNewPrivileges=true
PrivateTmp=true
ProtectSystem=strict
ProtectHome=true
ReadWritePaths=/srv/minio/data /var/lib/minio

[Install]
WantedBy=multi-user.target
```

ExecStartPre зупиняє старт без mount, щоб сервіс не писав на root filesystem. Шляхи mountpoint і systemd-директиви звір на цільовій ОС. TimeoutStopSec і file limit — початкові значення прикладу, які треба узгодити з навантаженням; restart на single-node означає перерву доступності.

### HTTPS для S3 API

Фрагмент Caddyfile для вже встановленого Caddy на тому самому хості. DNS, ACME validation і firewall мають бути налаштовані окремо. Додай site block до чинного файла, не затирай інші virtual hosts:

```text
s3.example.com {
    reverse_proxy 127.0.0.1:9000
}
```

S3 endpoint розміщується в корені домену. Не обрізай path і не переписуй підписаний Host: це впливає на Signature V4. API й Console не міняють місцями. Зовні достатньо дозволеного HTTPS до proxy; console залишена для адміністративного tunnel.

```bash
sudo caddy validate --config /etc/caddy/Caddyfile &&
  sudo systemctl reload caddy
curl --fail --silent --show-error --output /dev/null \
  --write-out 'https live: %{http_code}\n' \
  https://s3.example.com/minio/health/live
```

Не використовуй -k як фікс TLS. Це перевірка маршруту й сертифіката, не перевірка S3-прав.

[Reverse proxy Caddy: headers і rewrites](https://caddyserver.com/docs/caddyfile/directives/reverse_proxy)

Для Console з робочої станції, якщо локальний порт вільний:

```bash
ssh -N -L 9001:127.0.0.1:9001 '<SSH_USER>@<MINIO_HOST>'
```

Поки tunnel працює, відкрий `http://127.0.0.1:9001`. Не перетворюй доступ адміністратора на загальний публічний console endpoint. Зупинка SSH завершує tunnel.

### Policy backup-writer.json

Це policy для звичайних object PUT/GET та multipart upload у одному bucket. Без DeleteObject, admin-доступу й переліку всіх buckets. JSON не містить ключів доступу.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetBucketLocation",
        "s3:ListBucket",
        "s3:ListBucketMultipartUploads"
      ],
      "Resource": ["arn:aws:s3:::backup"]
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:AbortMultipartUpload",
        "s3:ListMultipartUploadParts"
      ],
      "Resource": ["arn:aws:s3:::backup/*"]
    }
  ]
}
```

## Якщо не працює

### Сервіс не стартує

Почни з журналу й шляху, без restart по колу:

```bash
systemctl status minio --no-pager
sudo journalctl -u minio -n 100 --no-pager
findmnt --mountpoint /srv/minio
namei -l /srv/minio/data
sudo -u minio-user test -w /srv/minio/data
```

Якщо ExecStartPre завершується невдало, перевір mount. Permission denied перевіряй для всіх батьківських каталогів і обмежень unit; не лікуй chmod 777. test -w не перевіряє фактичний запис чи здоров’я диска. Помилка формату даних потребує окремого плану сумісності — не видаляй data directory для «чистого старту».

### Console відкривається, S3-клієнт отримує HTML

Перевір, що alias веде на API :9000 або його HTTPS proxy, а не Console :9001. HTTP 200 від сторінки login не є успіхом S3. Звір DNS і маршрут proxy до виправлення ключів.

### AccessDenied

На admin-хості звір конкретного користувача й policy:

```bash
mc admin user info minio-admin backup-writer
mc admin policy info minio-admin backup-writer
```

На job-хості перевір дозволений `backup-job/backup`, а не `mc ls backup-job`: останній може вимагати переліку всіх buckets, якого тут немає. Для KMS, retention або backup-програми з lock/prune потрібні додатково обґрунтовані дозволи. Не перемикай job на root, щоб сховати причину.

### SignatureDoesNotMatch або помилка TLS

Перевір URL, час і маршрут. На сервері:

```bash
timedatectl status
sudo ss -lntp '( sport = :9000 or sport = :9001 )'
```

На клієнті звір час і довіру до сертифіката. Далі перевір secret key, підписаний Host та path у proxy. Не відключай перевірку TLS й не вставляй credentials у debug-звіт.

### Місце закінчується, хоча файлів начебто мало

Порівняй filesystem, поточні об’єкти та незавершені multipart uploads:

```bash
df -hT /srv/minio
df -i /srv/minio
mc du --recursive minio-admin/backup
mc ls --incomplete --recursive minio-admin/backup
```

Якщо ввімкнене versioning, окремо врахуй старі версії й delete markers. Список incomplete не означає, що їх можна масово прибрати: спочатку виключи активні uploads. Не видаляй вручну файли з backend-каталогу MinIO.

## Зміна, перевірка й повернення

### Було: API слухає всі інтерфейси

Фрагмент параметрів запуску, а не опис зафіксованого інциденту:

```text
--address :9000 --console-address :9001
```

### Стало: proxy і tunnel на тому самому хості

```text
--address 127.0.0.1:9000 --console-address 127.0.0.1:9001
```

Перед такою зміною переведи клієнтів на перевірений HTTPS proxy: прямі LAN-клієнти втратять доступ. Збережи чинний unit, env і бінарний артефакт у захищеній backup-системі. Після зміни в погоджене вікно:

```bash
sudo systemd-analyze verify /etc/systemd/system/minio.service &&
  sudo systemctl daemon-reload &&
  sudo systemctl restart minio
sudo ss -lntp '( sport = :9000 or sport = :9001 )'
```

Повтори health, API upload/download і перевірку identity job. Для повернення мережевої зміни віднови попередній перевірений unit, виконай daemon-reload і restart; звір firewall перед відновленням широкого listener. Не запускай цей рецепт як автоматичний downgrade MinIO: старий binary може не читати змінений формат даних.

Повернення unit не повертає перезаписані об’єкти. Для даних потрібна незалежна копія й перевірений restore. Саме цей момент я раніше за всіх хочу оголосити «очевидним» — тому й залишаю його в плані перевірки.
