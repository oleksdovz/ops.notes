## Тепер корисне: мінімальний знімок перед restart

Приклад для systemd на Linux. Потрібні права читання стану служби й journal; приклади не виконують перезапуск.

### 1. Стан і часовий інтервал

```bash
export UNIT_NAME='<SERVICE_NAME>.service'
export START_TIME='<START_TIME>'
export END_TIME='<END_TIME>'
systemctl show "$UNIT_NAME" \
  -p ActiveState -p SubState -p MainPID -p NRestarts -p Result
journalctl -u "$UNIT_NAME" --since "$START_TIME" \
  --until "$END_TIME" -o short-iso --no-pager
```

Зістав час помилки, restart count і повідомлення застосунку. Порожній журнал не доводить відсутність проблеми: перевір unit, часовий пояс, права й retention.

### 2. Ознаки браку ресурсів

```bash
free -h
df -h
df -i
journalctl -k --since "$START_TIME" --until "$END_TIME" --no-pager
```

Шукай OOM, I/O та filesystem errors, переповнений диск або inode. Високе використання RAM без контексту не доводить OOM. Не публікуй необроблені журнали: у них можуть бути запити, адреси й секрети.

### 3. Після відновлення

Повтори саме невдалий запит, звір помилки й затримку за новий інтервал. Запиши окремо: сервіс відновлений; причина підтверджена або ще ні. Restart може дати простій, а повернути попередній runtime-стан назад уже не можна. Перед ним збережи потрібні докази.

[journalctl: фільтри часу та unit](https://www.freedesktop.org/software/systemd/man/latest/journalctl.html)
