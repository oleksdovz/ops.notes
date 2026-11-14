### scripts/sops-encrypt.sh

Приймає шлях до одного файла, працює з будь-якого поточного каталогу, сам знаходить кореневий `.sops.yaml` потрібного checkout і замінює вхідний файл лише після успішного шифрування:

```bash
#!/usr/bin/env bash
set -euo pipefail

if [ "$#" -ne 1 ]; then
  echo "usage: sops-encrypt.sh path/to/file.sops.yaml" >&2
  exit 1
fi

input_path=$1

case "$input_path" in
  *.sops.yaml) ;;
  *)
    echo "error: $input_path does not end in .sops.yaml" >&2
    exit 1
    ;;
esac

if [ ! -f "$input_path" ]; then
  echo "error: $input_path not found" >&2
  exit 1
fi

repo_root=$(git -C "$(dirname "$input_path")" rev-parse --show-toplevel)
sops_config="$repo_root/.sops.yaml"

if [ ! -f "$sops_config" ]; then
  echo "error: no .sops.yaml at $repo_root" >&2
  exit 1
fi

abs_input=$(cd "$(dirname "$input_path")" && pwd)/$(basename "$input_path")
rel_path=${abs_input#"$repo_root"/}

echo "config: $sops_config"
echo "rule target: $rel_path"

tmp_output=$(mktemp "${abs_input}.XXXXXX")
trap 'rm -f "$tmp_output"' EXIT

if ! (cd "$repo_root" && sops --encrypt "$rel_path") >"$tmp_output"; then
  echo "error: sops encryption failed, $input_path left unchanged" >&2
  exit 1
fi

if ! grep -q '^sops:' "$tmp_output"; then
  echo "error: encrypted output missing sops metadata, $input_path left unchanged" >&2
  exit 1
fi

mv "$tmp_output" "$abs_input"
trap - EXIT
echo "encrypted in place: $input_path"
```

`rel_path` обчислюється відносно `repo_root`, а не відносно поточного каталогу запуску скрипта — саме цей шлях звіряється з `path_regex` у `.sops.yaml`, тому вибір правила відповідає реальному шляху файла в репозиторії незалежно від того, звідки скрипт викликали. Заміна вхідного файлу відбувається через `mv` тимчасового результату й лише після перевірки exit code `sops` та наявності блоку `sops:` у виводі.

### scripts/sops-decrypt.sh

За замовчуванням лише перевіряє, що розшифрування проходить, без виводу plaintext; prints plaintext лише з явним `--stdout`:

```bash
#!/usr/bin/env bash
set -euo pipefail

stdout_mode=0
input_path=""

for arg in "$@"; do
  case "$arg" in
    --stdout)
      stdout_mode=1
      ;;
    *)
      if [ -n "$input_path" ]; then
        echo "usage: sops-decrypt.sh [--stdout] path/to/file.sops.yaml" >&2
        exit 1
      fi
      input_path=$arg
      ;;
  esac
done

if [ -z "$input_path" ]; then
  echo "usage: sops-decrypt.sh [--stdout] path/to/file.sops.yaml" >&2
  exit 1
fi

case "$input_path" in
  *.sops.yaml) ;;
  *)
    echo "error: $input_path does not end in .sops.yaml" >&2
    exit 1
    ;;
esac

if [ ! -f "$input_path" ]; then
  echo "error: $input_path not found" >&2
  exit 1
fi

if [ "$stdout_mode" -eq 1 ]; then
  sops --decrypt "$input_path"
  exit 0
fi

if sops --decrypt "$input_path" >/dev/null; then
  echo "decrypt check ok: $input_path"
else
  echo "decrypt check failed: $input_path" >&2
  exit 1
fi
```

Режим перевірки виконує той самий `sops --decrypt`, що й `--stdout`, але відкидає plaintext у `/dev/null` і звітує лише exit code. `MAC mismatch` чи відсутня identity завершують цю команду помилкою до будь-якого виводу значення — скрипт лише передає цю помилку далі, не приховує й не замовчує її.
