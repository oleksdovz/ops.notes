## Тепер корисне: налаштування з нуля

Назви вузла, гілки, шляхів репозиторію та namespace нижче — узгоджені приклади для цього мануалу; переноси їх у свою структуру за потреби. Значення в кутових дужках — заповнювачі для твоїх реальних даних. Перевіряй параметри команд і полів CRD з документацією k3s, Flux, SOPS і age, посилання наведені під відповідними кроками.

### 1. k3s і локальні інструменти

Виконуй на k3s-хості. Встанови k3s на одному Linux-вузлі офіційним інсталятором і перевір, що служба піднялась і вузол готовий:

```bash
curl -sfL https://get.k3s.io | sh -
sudo systemctl status k3s --no-pager
sudo k3s kubectl get nodes -o wide
```

Очікувано: `systemctl status` показує `active (running)`, а вузол у списку має `STATUS` `Ready`. Kubeconfig сервера лежить у `/etc/rancher/k3s/k3s.yaml`; він типово читається лише root, обмеж права перед копіюванням і не публікуй його.

Далі виконуй на робочій станції. Перенеси вміст цього kubeconfig у свій `~/.kube/config`, заміни адресу `server:` на реальну адресу k3s-хоста замість `127.0.0.1` і обмеж права файлу:

```bash
chmod 600 ~/.kube/config
kubectl config get-contexts
kubectl config current-context
kubectl cluster-info
kubectl get nodes -o wide
```

Мій найшвидший тест доступу до кластера — `kubectl` відповів. Друге питання трохи незручніше: який саме кластер мені відповів? `kubectl config current-context` і `kubectl cluster-info` відповідають на нього прямо: перший показує context, другий — адресу API server, з якою ти зараз розмовляєш. Звір обидва значення з очікуваним k3s-хостом, а не лише з фактом, що команда не впала з помилкою; у `~/.kube/config` може лежати кілька контекстів з минулих стендів.

Встанови й перевір CLI для решти кроків, також на робочій станції:

```bash
curl -s https://fluxcd.io/install.sh | sudo bash
flux --version
flux check --pre
sops --version
age --version
age-keygen --version
```

`sops` і `age` зазвичай доступні через пакетний менеджер дистрибутива або офіційні релізи на GitHub; підбирай версію й платформу під свою робочу станцію і перевіряй checksum релізу перед встановленням. `flux check --pre` перевіряє сумісність kube-context із мінімальними вимогами Flux ще до будь-якого bootstrap.

### 2. Ілюстрація: Flux підключається до Git

Цей крок — лише схема. Підготовку репозиторію, видачу прав, SSH host key, deploy key і сам `flux bootstrap` у цій статті не виконуємо — жодна з цих команд тут не наведена.

```text
Робоча станція                         k3s-хост (flux-system)
───────────────                        ───────────────────────
Git-репозиторій (приклад)              GitRepository "flux-system"
  гілка: main                          ├─ spec.ref.branch: main
  шлях: clusters/lab                   ├─ перевірка SSH host key
      │                                │    репозиторію перед read
      │ bootstrap: запис з повним      │
      │ доступом один раз              ▼
      ▼                            read-only SSH deploy key
  репозиторій на Git-хості  ───────►  (лише читання, без push)
                                        │
                                        ▼
                                  Kustomization "flux-system"
                                  spec.path: ./clusters/lab
                                  spec.sourceRef → GitRepository
```

Flux прочитав Git. Мені вже хочеться здати всю роботу, хоча поки він тільки відкрив репозиторій. `GitRepository` і `Kustomization` — два різні об’єкти з різним станом, і перевіряти їх треба окремо:

```bash
kubectl get gitrepository flux-system -n flux-system \
  -o jsonpath='{.spec.ref.branch}{"\t"}{.status.artifact.revision}{"\n"}'
kubectl get kustomization flux-system -n flux-system \
  -o jsonpath='{.spec.path}{"\t"}{.status.lastAppliedRevision}{"\n"}'
```

Перша команда читає-only підтверджує, яку гілку очікує `GitRepository` і яку ревізію він фактично завантажив. Друга підтверджує шлях, який `Kustomization` застосовує, і яку ревізію вона застосувала востаннє. Завантажена `GitRepository` ревізія, яка ще не збіглася з `lastAppliedRevision` `Kustomization`, означає лише «Flux побачив коміт», а не «маніфести з цього коміту вже в кластері».

З наступного кроку вважаємо, що такий checkout на робочій станції вже існує локально, а Flux уже синхронізує цей репозиторій і гілку з попередньої схеми.

### 3. Age identity і shell

Виконуй на робочій станції. Створи приватну identity поза Git, з обмеженими правами доступу для свого користувача:

```bash
mkdir -p "<HOST_AGE_KEY_DIRECTORY>"
chmod 700 "<HOST_AGE_KEY_DIRECTORY>"
age-keygen -o "<HOST_AGE_KEY_DIRECTORY>/keys.txt"
chmod 600 "<HOST_AGE_KEY_DIRECTORY>/keys.txt"
age-keygen -y "<HOST_AGE_KEY_DIRECTORY>/keys.txt"
```

Останній рядок виводить лише публічний recipient у форматі `age1...` — саме це значення піде у `.sops.yaml`. Файл `keys.txt` не йде нікуди, крім резервної копії в менеджері секретів команди: не в Git, не на спільний диск, не в той самий бекап робочої станції, що й усе інше.

Для локального decrypt задай змінну в сесії, де будеш працювати з `sops`:

```bash
export SOPS_AGE_KEY_FILE="<HOST_AGE_KEY_DIRECTORY>/keys.txt"
```

Я зробив `export` у своєму shell. Орієнтуватися в моєму терміналі `kustomize-controller` чомусь відмовився. Ця змінна діє лише в дочірніх процесах поточної сесії shell: у новому терміналі, у systemd-юніті чи в Pod кластера вона не існує сама по собі. Для Flux це означає, що жодний локальний `export` не впливає на те, чим розшифровує `kustomize-controller` — йому потрібен окремий Kubernetes Secret, який ми створимо в кроці 6.

Коротко про сусідні змінні: `SOPS_AGE_RECIPIENTS` задає recipient напряму при шифруванні, в обхід `.sops.yaml` — корисно для разової команди без конфіг-файлу. `SOPS_AGE_KEY` передає саму identity значенням змінної, а не шляхом до файлу — варіант для середовищ, де писати ключ на диск небажано; значення такої змінної не повинне потрапляти в аргументи командного рядка чи в лог. У наскрізному прикладі цього мануалу `.sops.yaml` відповідає за шифрування, а `SOPS_AGE_KEY_FILE` — за локальний decrypt.

### 4. Конфігурація SOPS і тестові маніфести

Далі працюємо у вже наявному checkout на робочій станції. Ілюстративне дерево каталогів із файлами, які додає цей мануал:

```text
.
├── .sops.yaml
├── scripts/
│   ├── sops-encrypt.sh
│   └── sops-decrypt.sh
└── clusters/
    └── lab/
        ├── flux-system/            (створено bootstrap, поза цим мануалом)
        └── secrets-demo/
            ├── kustomization.yaml
            ├── namespace.yaml
            ├── secret.sops.yaml
            └── workload.yaml
```

Кореневий `.sops.yaml`:

```yaml
creation_rules:
  - path_regex: \.sops\.yaml$
    encrypted_regex: ^(data|stringData)$
    age: <AGE_RECIPIENT_PUBLIC_KEY>
```

`path_regex` звіряється зі шляхом файлу відносно каталогу, де лежить сам `.sops.yaml`; тому правило застосовується до будь-якого файлу з суфіксом `.sops.yaml` в усьому репозиторії. `encrypted_regex` обмежує шифрування полями `data` і `stringData` — решта полів Secret, включно з `metadata.namespace`, лишається читаною.

Тестовий Secret `clusters/lab/secrets-demo/secret.sops.yaml`, поки у відкритому вигляді — це локальний демонстраційний plaintext, у Git і staging має потрапити лише зашифрована версія:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: demo-secret
  namespace: secrets-demo
type: Opaque
stringData:
  message: sops-and-age-demo-value
```

Мінімальний workload, який читає цей Secret і повідомляє лише статус, без самого значення, `clusters/lab/secrets-demo/workload.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-reader
  namespace: secrets-demo
spec:
  restartPolicy: Never
  containers:
    - name: secret-reader
      image: busybox:1.36
      command: ["sh", "-c", "if [ -n \"$MESSAGE\" ]; then echo status=ok; else echo status=missing; fi; sleep 3600"]
      env:
        - name: MESSAGE
          valueFrom:
            secretKeyRef:
              name: demo-secret
              key: message
```

`clusters/lab/secrets-demo/namespace.yaml` створює namespace `secrets-demo`, а `clusters/lab/secrets-demo/kustomization.yaml` включає всі три файли цього каталогу:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - namespace.yaml
  - secret.sops.yaml
  - workload.yaml
```

Цей каталог зв’язаний із Flux через `spec.path: ./clusters/lab` у `Kustomization` з кроку 2: Flux будує все дерево під цим шляхом, включно з `clusters/lab/secrets-demo`, якщо на нього є посилання з кореневого `kustomization.yaml` цього каталогу дерева.

### 5. Два повні Bash-скрипти

Повний текст `scripts/sops-encrypt.sh` і `scripts/sops-decrypt.sh` — у прикладах під цим блоком. Зроби їх виконуваними й перевір синтаксис перед першим запуском, з кореня checkout або будь-де всередині нього:

```bash
chmod +x scripts/sops-encrypt.sh scripts/sops-decrypt.sh
bash -n scripts/sops-encrypt.sh
bash -n scripts/sops-decrypt.sh
```

Зашифруй тестовий Secret із кроку 4 — скрипт сам знаходить кореневий `.sops.yaml` і підставляє шлях файлу відносно нього, тож правило `*.sops.yaml` застосовується коректно:

```bash
./scripts/sops-encrypt.sh clusters/lab/secrets-demo/secret.sops.yaml
```

Для правок уже зашифрованого файлу використовуй `sops edit` — він розшифровує у тимчасовий буфер, відкриває редактор і повторно шифрує результат, без проміжного plaintext-файлу на диску:

```bash
sops edit clusters/lab/secrets-demo/secret.sops.yaml
```

Побачив `ENC[...]` і потягнувся до commit. Добре, що staging можна перевірити раніше, ніж він стане історією Git. Самої наявності `ENC[...]` у файлі замало: це рядок тексту, він нічого не доводить про цілісність. Перевіряй саме ту версію, яку staged, а не робочу копію на диску:

```bash
git add clusters/lab/secrets-demo/secret.sops.yaml
staged_copy=$(mktemp -d)/staged-secret.sops.yaml
git show :clusters/lab/secrets-demo/secret.sops.yaml > "$staged_copy"
./scripts/sops-decrypt.sh "$staged_copy"
```

`sops-decrypt.sh` за замовчуванням лише перевіряє, що розшифрування проходить, і не виводить plaintext; успіх підтверджує і зашифрованість полів `data`/`stringData`, і цілісність структури файлу — несправний MAC чи зіпсована структура дадуть помилку на цьому кроці, а не пізніше у Flux.

### 6. Ключ для Flux

Створи Secret `sops-age` у тому самому namespace, де живе `Kustomization` з кроку 2 — у цьому мануалі це `flux-system`, а не namespace застосунку. Виконуй на робочій станції, передаючи ключ із файлу, без виводу його вмісту в термінал:

```bash
kubectl create secret generic sops-age \
  --namespace=flux-system \
  --from-file=age.agekey="<HOST_AGE_KEY_DIRECTORY>/keys.txt"
```

Додай розшифрування до вже наявного маніфесту `Kustomization` у репозиторії (`clusters/lab/flux-system/gotk-sync.yaml` зі схеми кроку 2) — редагуємо декларативне джерело, а не сам об’єкт у кластері:

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: flux-system
  namespace: flux-system
spec:
  interval: 10m
  path: ./clusters/lab
  prune: true
  sourceRef:
    kind: GitRepository
    name: flux-system
  decryption:
    provider: sops
    secretRef:
      name: sops-age
```

`kustomize-controller` шукає Secret, названий у `secretRef.name`, саме в namespace цього `Kustomization` — не в namespace застосунку, якому належить зашифрований Secret, і не на твоїй робочій станції. Локальний decrypt проходить. У мене вже працює. Слово «у мене» тут виконує підозріло багато роботи: воно підтверджує лише те, що `sops` на моїй машині бачить правильний файл identity. Звір `secretRef.name` і namespace в `Kustomization` з фактичним namespace Secret `sops-age`, а потім перевір стан reconcile:

```bash
kubectl get secret sops-age -n flux-system
flux get kustomizations
```

Правка `clusters/lab/flux-system/gotk-sync.yaml` поки лише локальна; вона потрапить у Git разом з рештою файлів на наступному кроці.

### 7. Перевірка: від staging до Pod

Зафіксуй і відправ зміни з наявного checkout:

```bash
git add .sops.yaml scripts/sops-encrypt.sh scripts/sops-decrypt.sh \
  clusters/lab/flux-system/gotk-sync.yaml \
  clusters/lab/secrets-demo/
git commit -m "Add SOPS/age encrypted demo secret"
git push
```

Перевір, що Flux застосував саме цю ревізію, а не просто побачив репозиторій:

```bash
git rev-parse HEAD
flux reconcile source git flux-system
flux get kustomizations
kubectl get kustomization flux-system -n flux-system \
  -o jsonpath='{.status.lastAppliedRevision}{"\n"}'
```

`lastAppliedRevision` має містити той самий hash, що й локальний `git rev-parse HEAD` для гілки `main`. Далі перевір цільовий Secret і статус Pod, не виводячи саме значення:

```bash
kubectl get secret demo-secret -n secrets-demo
kubectl get pod secret-reader -n secrets-demo
kubectl logs secret-reader -n secrets-demo
```

Очікувано: Secret існує в namespace `secrets-demo`, Pod у фазі `Running`, а лог містить рядок `status=ok`. `status=missing` означає, що Pod піднявся, але Secret не передав очікуваного значення — симптом для розбору нижче, не підтвердження готовності.

Короткий порядок відновлення, якщо на новій робочій станції відсутня age identity: відновити файл identity з резервної копії в менеджері секретів команди, повторити `export SOPS_AGE_KEY_FILE=...` з кроку 3 і перевірити розшифрування без виводу plaintext.

```bash
./scripts/sops-decrypt.sh clusters/lab/secrets-demo/secret.sops.yaml
```

У плані відновлення в мене один дуже гарний рядок — «забрати все з Git». Для приватного ключа доведеться виділити другий: Git містить лише зашифровані значення й публічний recipient, приватна identity в ньому ніколи не з’являється, і без її резервної копії цей рядок плану не виконається. Успішний `sops-decrypt.sh` без `--stdout` на відомому зашифрованому файлі підтверджує, що відновлена identity — та самісінька, без показу вмісту секрету. Повернення доступу Flux до самого Git-репозиторію (deploy key, host key) лишається ілюстрацією з кроку 2 і в цьому мануалі окремо не виконується.

## Дебаг

Для кожного симптому — read-only перевірка, очікувана ознака у виводі, межі висновку і наступний крок. Приклади виводу позначені як приклади.

### Неправильний kube-context

```bash
kubectl config current-context
kubectl cluster-info
```

Очікувано: назва context і адреса API server збігаються з k3s-хостом із кроку 1. Якщо `kubectl` відповідає, але з іншого кластера, усі подальші перевірки в цьому мануалі стосуватимуться не того середовища. Переключення: було — `kubectl config current-context` показує сторонній context; стало — `kubectl config use-context <CONTEXT_NAME>`; перевірка — повторний `kubectl cluster-info` з очікуваною адресою.

### Відсутній доступ Flux до Git

```bash
flux get sources git
kubectl get gitrepository flux-system -n flux-system -o yaml
```

Шукай у `status.conditions` умову `Ready: False` з повідомленням про автентифікацію чи SSH host key. Це підтверджує лише, що `GitRepository` не може прочитати репозиторій; причина — зниклий чи відкликаний deploy key або змінений host key — лежить на стороні доступів із кроку 2. Відновлення читання для read-only deploy key й host key в цьому мануалі лише ілюстративне; конкретні команди для свого Git-хостингу перевір в його документації.

### Правило `.sops.yaml` не підходить до шляху файла

```bash
sops --decrypt clusters/lab/secrets-demo/secret.sops.yaml | head -1
```

Якщо `data`/`stringData` у файлі лишились plaintext, а не `ENC[...]`, жодне правило `creation_rules` не збіглося зі шляхом файлу під час шифрування. Було — `path_regex`, що не покриває реальний шлях файла в репозиторії; стало — `path_regex`, звірений із тим самим шляхом, яким `sops` викликали (нагадаю: відносно каталогу з `.sops.yaml`); перевірка — повторне шифрування і наявність блоку `sops:` у результаті.

### Недоступна age identity

```bash
echo "$SOPS_AGE_KEY_FILE"
ls -l "$SOPS_AGE_KEY_FILE"
```

Порожня змінна або відсутній файл — пряма причина, чому локальний `sops --decrypt` завершиться помилкою отримання ключа даних. Було — втрачений чи недоступний файл identity; стало — файл, відновлений з резервної копії менеджера секретів, з правами, обмеженими власнику; перевірка — `./scripts/sops-decrypt.sh` без `--stdout` на відомому зашифрованому файлі.

### `MAC mismatch`

`MAC mismatch` означає збій перевірки цілісності: SOPS зберігає код автентифікації повідомлення для всього зашифрованого вмісту, і будь-яка зміна файлу поза самим `sops` (ручне редагування ciphertext, конфлікт merge, часткове відновлення з іншого джерела) цей код ламає.

```bash
./scripts/sops-decrypt.sh clusters/lab/secrets-demo/secret.sops.yaml
```

Межа висновку: ця помилка підтверджує лише розбіжність між вмістом файлу і збереженим MAC, не причину розбіжності. Було — файл, змінений поза `sops`/`sops edit`; стало — версія файлу з останньої коректної ревізії Git (`git show <COMMIT>:<PATH>` або `git checkout` потрібної ревізії для цього файлу), з повторним застосуванням правок через `sops edit`; перевірка — повторний `sops-decrypt.sh` без помилки.

### Відсутній `sops-age`

```bash
kubectl get secret sops-age -n flux-system
flux get kustomizations
```

Відсутній Secret або помилка `Ready: False` в `Kustomization` з повідомленням про розшифрування — підтверджує лише те, що `kustomize-controller` не має ключа в очікуваному namespace. Було — відсутній або видалений Secret `sops-age`; стало — Secret, відтворений командою з кроку 6 із файлу identity, без виводу вмісту в термінал; перевірка — повторний `flux get kustomizations` з `Ready: True` та очікуваним `lastAppliedRevision`.

### Namespace цільового Secret

```bash
kubectl get secret demo-secret -n secrets-demo
kubectl get pod secret-reader -n secrets-demo -o jsonpath='{.spec.containers[0].env}'
```

Невідповідність namespace у маніфесті Secret і namespace, де Pod його шукає, дає `status=missing` у логах Pod навіть якщо розшифрування пройшло без помилок. Namespace — поле метаданих, воно лишається plaintext навіть у зашифрованому файлі, тож цю помилку видно без decrypt. Було — `metadata.namespace` Secret, що не збігається з namespace workload; стало — виправлене поле в тому самому манфесті, без повторного шифрування `data`/`stringData`, бо вони не змінювались; перевірка — `kubectl logs secret-reader -n secrets-demo` з `status=ok` після наступного reconcile.

[Flux: Manage Kubernetes secrets with SOPS](https://fluxcd.io/flux/guides/mozilla-sops/)

[SOPS: config file reference](https://getsops.io/docs/usage/identities/config-file/)

[k3s: Quick Start Guide](https://docs.k3s.io/quick-start)
