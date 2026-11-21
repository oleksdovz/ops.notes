## Тепер корисне: перевіряй WSL, потім міняй систему

Команди призначені для Windows PowerShell і Ubuntu у WSL. Експорт і видалення — окремі кроки; не запускай `wsl --unregister`, доки не перевірив резервну копію.

### 1. Відрізни default від версії Ubuntu

У PowerShell:

```powershell
wsl.exe --status
wsl.exe -l -v
```

`Default Version` — версія для нових дистрибутивів. Для вже встановленої Ubuntu дивись її окремий рядок і поле `VERSION`.

### 2. Перевір, чи відкривається чистий Bash

З PowerShell:

```powershell
wsl.exe -d Ubuntu --exec /bin/bash --noprofile --norc -i
```

Якщо з’явився Bash prompt, введи `exit` і спробуй звичайний запуск. Якщо вікно лишається порожнім, натисни Ctrl+C і перевір просту команду:

```powershell
wsl.exe -d Ubuntu --exec /usr/bin/id
```

Якщо чистий Bash відкривається, а звичайний запуск ні, перевіряй startup-файли. Якщо не виконується й `id`, збери помилку WSL до перевстановлення.

### 3. Якщо свідомо замінюєш дистрибутив — спочатку перевір export

`wsl --unregister` видаляє дані старої Ubuntu. Спершу переконайся, що export завершився і файл створився.

У PowerShell:

```powershell
$backup = Join-Path $env:USERPROFILE "Ubuntu-backup-$(Get-Date -Format yyyyMMdd-HHmmss).tar"
wsl.exe --export Ubuntu "$backup"
if ($LASTEXITCODE -ne 0 -or -not (Test-Path $backup)) {
  throw "Export не підтверджено. Зупинись і не видаляй Ubuntu."
}
Get-Item $backup
```

Перевір файл резервної копії, перш ніж виконувати деструктивний крок:

```powershell
wsl.exe --unregister Ubuntu
if ($LASTEXITCODE -ne 0) {
  throw "Unregister не завершився успішно. Зупинись."
}
```

Потім, окремою командою:

```powershell
wsl.exe --install -d Ubuntu
```

### 4. Перевір нову інсталяцію, не вгадуй її версію

```powershell
wsl.exe -l -v
wsl.exe -d Ubuntu --exec /usr/bin/id
```

Переконайся, що Ubuntu має очікуваний `VERSION` і команда повертає `uid=...`.

### 5. Знайди справжнє джерело Oh My Zsh warnings

У WSL перевір усі згадки plugins, `kustomize` і `zsh-completions`, перш ніж редагувати конфіг:

```zsh
rg -n 'plugins=|kustomize|zsh-completions' ~/.zshrc ~/.oh-my-zsh/custom
```

Додай completion directories після Homebrew `shellenv`, але до `source` Oh My Zsh. Не записуй `zsh-completions` у `plugins` лише тому, що встановлено пакет completions. `kustomize` також не додавай як Oh My Zsh plugin без реального plugin-каталогу.

```zsh
# Homebrew shellenv має бути виконаний вище.
if [[ -d "$HOME/.local/share/zsh/completions" ]]; then
  fpath=("$HOME/.local/share/zsh/completions" "${fpath[@]}")
fi
if [[ -d "$HOMEBREW_PREFIX/share/zsh-completions" ]]; then
  fpath=("$HOMEBREW_PREFIX/share/zsh-completions" "${fpath[@]}")
fi

plugins=(
  git kubectl docker docker-compose aws terraform helm
  zsh-autosuggestions zsh-syntax-highlighting
)
source "$ZSH/oh-my-zsh.sh"
```

Переконайся, що `zsh-syntax-highlighting` стоїть останнім у `plugins=(...)`. Після правки перезапусти login shell і перевір окремо, що зникли warnings і працює потрібне completion.