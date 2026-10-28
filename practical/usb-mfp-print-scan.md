## Тепер корисне: налаштування AirPrint і AirScan з нуля

Це рецепт нової native-інсталяції на Debian 12 з systemd і sudo. Для іншого дистрибутива звір назви пакетів та man pages. Він доповнює зафіксований кейс, а не відтворює невідомий driver чинної черги. На робочому сервері пропускай установлення й створення вже наявної queue.

AirPrint — друк без окремого драйвера на Apple-клієнті. Bonjour знаходить пристрій, IPP передає завдання, а endpoint повинен підтримувати потрібні формати й capabilities. AirScan — окремий шлях сканування через eSCL. Сам напис Shared не обіцяє обох функцій. Я теж люблю один checkbox, але тут у нього немає таких повноважень.

### 1. Перевір сумісність і поточну конфігурацію

МФУ має підтримувати IPP-over-USB та eSCL. Цей рецепт не перетворює довільний USB scanner на AirScan-сервер. На Linux спочатку подивись систему, USB-пристрої та наявні черги:

```bash
cat /etc/os-release
lsusb
lpstat -t
lpstat -v
```

Відсутня команда означає, що відповідну утиліту ще треба встановити. USB ID звір із документацією виробника; сама присутність у lsusb не доводить підтримку протоколу. URI може містити serial — не публікуй сирий вивід.

### 2. Підготуй Linux-сервер

Для нового Debian-сервера встанови компоненти. CUPS обслуговує чергу, Avahi публікує discovery, ipp-usb надає доступ до сумісного USB-МФУ. sane-airscan — клієнтський backend для Linux-перевірки; він не є сервером для Mac.

```bash
sudo apt-get update
sudo apt-get install cups cups-client avahi-daemon avahi-utils \
  ipp-usb sane-airscan sane-utils usbutils
sudo systemctl enable --now cups avahi-daemon
sudo systemctl start ipp-usb
sudo ipp-usb check
sudo ipp-usb status
```

Підключи й увімкни МФУ перед перевіркою. Очікуй знайдений пристрій, а не лише успішний exit code. ipp-usb може мати static unit — для нього тут немає enable. Конфлікти з іншим USB backend розбирай за журналом; не вимикай робочий драйвер навмання.

### 3. Налаштуй AirScan у LAN

Нижче, у блоці конфігурації ipp-usb, є точні backup, зміна та повернення. У наявній секції `[network]` файла `/etc/ipp-usb/ipp-usb.conf` потрібні `interface = all` і `dns-sd = enable`. Після обмеження доступу довіреною LAN перевір конфігурацію й перезапусти proxy за тим блоком. Це може перервати активні jobs.

Для Mac не потрібні saned, scanservjs або встановлення sane-airscan на macOS. Discovery виконує Avahi, запити сканування приймає ipp-usb. Перевір на сервері:

```bash
scanimage -L
avahi-browse -rt _uscan._tcp
sudo ss -lntp | grep -F 'ipp-usb'
```

Шукай AirScan device і реальний мережевий endpoint. Дозволь його фактичний TCP-порт тільки потрібним клієнтам; для discovery у цьому рецепті сервер і Mac мають бути в одному LAN-сегменті з mDNS UDP 5353. `all` не обмежує доступ лише цим сегментом — це завдання firewall.

### 4. Створи CUPS-чергу лише якщо її ще немає

Спочатку знайди IPP endpoint, який опублікував ipp-usb:

```bash
lpinfo -v
avahi-browse -rt _ipp._tcp
```

Візьми точний IPP URI з discovery. Не підставляй usb:// чи hp:// до model everywhere: він опитує саме IPP-принтер. Для сумісного IPP endpoint нова черга може виглядати так:

```bash
export PRINT_QUEUE='<NEW_QUEUE_NAME>'
export IPP_DEVICE_URI='<DISCOVERED_IPP_URI>'
sudo lpadmin -p "$PRINT_QUEUE" -E -v "$IPP_DEVICE_URI" -m everywhere
lpstat -p "$PRINT_QUEUE" -l
lpstat -v "$PRINT_QUEUE"
```

Підстав нове ім’я без пробілів. Якщо ім’я вже існує, lpadmin може змінити чинну queue — цей крок пропусти. Помилка опитування capabilities означає, що треба перевірити endpoint і підтримку; raw або випадковий Generic PostScript не є універсальним виправленням.

### 5. Увімкни мережевий друк і Bonjour

Цей профіль для довіреної спільної підмережі. Перед зміною дочекайся завершення jobs і збережи CUPS-конфігурацію. Каталог backup має містити саме стан перед цією зміною; якщо він уже існує, перевір його перед продовженням.

```bash
sudo install -d -m 0700 /var/backups/cups-before-lan
sudo cp -an /etc/cups/. /var/backups/cups-before-lan/
sudo cupsctl --share-printers
export PRINT_QUEUE='<EXISTING_OR_NEW_QUEUE_NAME>'
sudo lpadmin -p "$PRINT_QUEUE" -o printer-is-shared=true
sudo cupsd -t
sudo ss -lntp 'sport = :631'
avahi-browse -rt _ipp._tcp
```

Server sharing і sharing конкретної queue — обидва потрібні. cupsctl змінює конфігурацію служби; restart без потреби не додавай. Перевір LAN listener CUPS на TCP 631, доступ лише з довіреної мережі й оголошення через Avahi. Не додавай remote-any, remote-admin або анонімний admin заради друку. Наявні нестандартні ACL треба перевірити окремо.

Якщо до зміни queue була приватною, повернути тільки її sharing можна так:

```bash
export PRINT_QUEUE='<QUEUE_NAME>'
sudo lpadmin -p "$PRINT_QUEUE" -o printer-is-shared=false
```

Це не повертає глобальну конфігурацію сервера. Для повного повернення в погоджене вікно зупини CUPS, віднови змінені файли з перевіреної root-only копії, виконай cupsd -t і запусти службу. Звір різницю перед відновленням: backup не має затерти пізніші потрібні зміни.

### 6. Додай на Mac і перевір AirPrint

Відкрий System Settings → Printers & Scanners → Add Printer, Scanner, or Fax. Вибери потрібне Bonjour-оголошення, перевір поле Use: AirPrint, якщо macOS пропонує його для цього endpoint. Додай принтер і надрукуй PDF. У цій схемі можуть бути видимі і CUPS-черга, і прямий endpoint ipp-usb: звір назву, сервер і URI, щоб перевіряти обраний маршрут.

На Mac discovery можна подивитися окремо; команди працюють до Ctrl-C:

```bash
dns-sd -B _ipp._tcp local.
```

Для оголошень AirPrint перевір також subtype:

```bash
dns-sd -B _universal._sub._ipp._tcp local.
```

Наявність оголошення не доводить підтримку форматів і успішний друк. Якщо доступний лише інший driver, не називай цей результат перевіреним AirPrint. Не копіюй випадковий Avahi XML із чужими TXT capabilities: неправдиве оголошення не додає принтеру можливостей. Друк з iPhone/iPad у вихідному кейсі не перевіряли.

### 7. Додай сканер на Mac

Відкрий Image Capture / Цифровізація та вибери scanner у Shared / Devices. Якщо його немає, перевір додавання пристрою в Printers & Scanners. У Show Details задай область, роздільність, формат і папку, виконай Overview, потім Scan. Відкрий збережений файл: саме це завершує перевірку AirScan.

Для окремої перевірки discovery на Mac:

```bash
dns-sd -B _uscan._tcp local.
```

Зупини Ctrl-C. Якщо оголошення є, а сканування не починається, перевір досяжність його TCP endpoint; mDNS і передача документа — різні з’єднання. Відкриття CUPS web UI не перевіряє scan endpoint.

[Налаштування sharing CUPS](https://openprinting.github.io/cups/doc/sharing.html)

[Створення черги та model everywhere](https://openprinting.github.io/cups/doc/man-lpadmin.html)

[Додавання принтера на Mac](https://support.apple.com/guide/mac-help/add-and-manage-printers-mh14004/mac)

[Image Capture на Mac](https://support.apple.com/guide/image-capture/welcome/mac)

## Якщо після налаштування щось не працює

Приклади для native Linux-хоста з CUPS, ipp-usb, SANE та Avahi. Для Debian-подібної системи утиліти зазвичай надходять із cups-client, sane-utils, sane-airscan та avahi-utils. Вони потрібні на сервері; ці команди не виконувалися в межах підготовки статті.

### 1. Встанови поточний стан без змін

На сервері:

```bash
systemctl show cups ipp-usb avahi-daemon \
  -p Id -p LoadState -p ActiveState -p UnitFileState
lpstat -t
scanimage -L
```

Очікуй доступну чергу й виявлений scanner. У цьому маршруті локальне сканування підтверджувалося через AirScan; інші показані SANE backends не вважай автоматично несправними. Відсутність queue та відсутність scan device — різні гілки діагностики.

### 2. Якщо локально видно, а з Mac — ні

На Linux-сервері перевір listener і оголошення сканера:

```bash
sudo ss -lntp | rg 'ipp-usb'
avahi-browse -rt _uscan._tcp
sudo journalctl -u ipp-usb -n 50 --no-pager
```

Для `rg` потрібен ripgrep; якщо його немає, у цьому фільтрі можна використати `grep -F 'ipp-usb'`. Якщо listener лише на 127.0.0.1 або ::1, він не приймає прямі LAN-запити. Якщо listener уже мережевий, далі перевіряй firewall, адреси announcement, VLAN та client isolation. Оголошення, видиме на сервері, ще не доводить його видимість на Mac.

Для CUPS окремо перевір порт 631 та оголошення друку:

```bash
sudo ss -lntp 'sport = :631'
avahi-browse -rt _ipp._tcp
```

### 3. Перевір сторінку, а не лише discovery

На сервері поклади тестову сторінку на скло й підстав точний AirScan device з scanimage -L. Використай документ без чутливих даних:

```bash
export SCAN_DEVICE='<AIRSCAN_DEVICE_FROM_DISCOVERY>'
SCAN_FILE="$(mktemp /tmp/scan-check.XXXXXX)"
scanimage -d "$SCAN_DEVICE" --format=png > "$SCAN_FILE" &&
  test -s "$SCAN_FILE" &&
  file "$SCAN_FILE"
```

Утиліта file має визначити непорожній PNG; відкрий файл і перевір вміст. Це локальна перевірка. Не переносиш її результат на Mac автоматично.

На Mac відкрий Image Capture, вибери scanner у Shared, виконай Overview, потім Scan у вибрану папку. Відкрий збережений файл. Для друку виконай фізичну тестову сторінку з черги, потім документ із клієнта.

### 4. Якщо ipp-usb не дозволяє enable

Спочатку перевір UnitFileState та журнал. Static unit може запускатися іншим механізмом пакета. Не створюй додаткову службу лише через відмову enable. На робочому МФУ start/restart проводь після завершення print/scan jobs; перезапуск proxy може перервати обидва маршрути, якщо друк також використовує його.
