## Тепер корисне: перевіряємо обидва маршрути

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
