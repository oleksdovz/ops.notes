### Було: proxy доступний лише самому серверу

Це порівняння режимів, а не твердження про зафіксовану причину початкової проблеми. Фрагмент `/etc/ipp-usb/ipp-usb.conf` для версії, що підтримує ці ключі:

```ini
[network]
interface = loopback
dns-sd = enable
```

### Стало: доступ для клієнтів довіреної LAN

Зміни лише наявну мережеву секцію, решту файлу збережи:

```ini
[network]
interface = all
dns-sd = enable
```

Параметри належать до `[network]`, а не довільної секції `[server]`. `all` означає мережеві інтерфейси, а не обмеження лише потрібною LAN. Перед зміною firewall має обмежувати фактичний endpoint довіреними джерелами. Локальні UID/GID правила proxy не замінюють мережеву автентифікацію.

### Застосування після перевірки

На сервері спочатку дочекайся завершення jobs. Збережи конфігурацію в root-only каталозі; повторний backup не повинен затирати потрібну точку повернення:

```bash
sudo install -d -m 0700 /var/backups/ipp-usb
sudo cp -an /etc/ipp-usb/ipp-usb.conf \
  /var/backups/ipp-usb/ipp-usb.conf.before-lan
sudoedit /etc/ipp-usb/ipp-usb.conf
```

Якщо backup уже існує, перевір, що це потрібна копія. Після редагування:

```bash
sudo ipp-usb check && sudo systemctl restart ipp-usb
systemctl is-active ipp-usb
sudo ss -lntp | grep -F 'ipp-usb'
```

Check перевіряє конфігурацію; це ще не тест remote scan. Порт пристрою визначай за фактичним listener. Не відкривай навмання весь типовий діапазон 60000–65535. Для discovery у локальному сегменті потрібний mDNS, UDP 5353; сам scan іде до TCP endpoint proxy. У різних VLAN потрібен окремо спроєктований доступ і discovery.

### Повернення

Віднови саме попередню конфігурацію, перевір її й лише після успіху перезапусти службу:

```bash
sudo cp -a /var/backups/ipp-usb/ipp-usb.conf.before-lan \
  /etc/ipp-usb/ipp-usb.conf
sudo ipp-usb check && sudo systemctl restart ipp-usb
```

Повернення до loopback прибирає прямий мережевий доступ до proxy. Воно не видаляє CUPS-черги й не відновлює перерваний scan. Перевір друк окремо, якщо його backend використовує той самий proxy.
