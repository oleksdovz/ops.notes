## Тепер корисне: перевіряй WSL, потім міняй систему

Нижче — команди для Windows PowerShell та Ubuntu у WSL. У вихідній переписці частина кроків лише пропонувалася; наведений код не є доказом, що команда вже виконувалася або спрацювала.

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

У вихідній переписці результатів цих двох тестів немає. Не оголошуй startup-файли причиною, доки тест цього не показав.

### 3. Якщо свідомо замінюєш дистрибутив — спочатку перевір export

Команди нижче не запускалися під час підготовки статті. `wsl --unregister` видаляє дані старої Ubuntu; переходь до нього лише після успішного експорту й перевірки файла.

У PowerShell:

```powershell
$backup = Join-Path $env:USERPROFILE "Ubuntu-backup-$(Get-Date -Format yyyyMMdd-HHmmss).tar"
wsl.exe --export Ubuntu "$backup"
if ($LASTEXITCODE -ne 0 -or -not (Test-Path $backup)) {
  throw "Export не підтверджено. Зупинись і не видаляй Ubuntu."
}
Get-Item $backup
```

Лише після того, як файл існує та має правдоподібний розмір, можна виконати деструктивний крок:

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

Переконайся, що Ubuntu має очікуваний `VERSION` і команда повертає `uid=...`. У вихідній переписці користувач підтвердив, що нова Ubuntu відкривається, але фінального `wsl -l -v` не навів.

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

Переконайся, що `zsh-syntax-highlighting` стоїть останнім у `plugins=(...)`. Вихідна переписка не підтверджує, що запропоновану правку застосували або що `kustomize` completion запрацював.