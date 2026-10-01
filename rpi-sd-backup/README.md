# rpi-sd-backup

Бекап SD-картки Raspberry Pi одним файлом `.img.xz`, який можна одразу прошити через
Raspberry Pi Imager на іншу (зокрема більшу) картку.

Це посекторна копія картки без жодних змін: після розпакування sha256 образу дорівнює
sha256 того, що було прочитано з картки. Дані читаються один раз і паралельно хешуються
та стискаються, тож сирий 16–64 GB файл на диску не потрібен (хіба з `--raw`).

Два способи запуску, один і той самий код:

| Спосіб | Файл | Де працює |
|--------|------|-----------|
| Нативно | `rpi-sd-backup` | macOS (diskutil), Linux (lsblk). bash 3.2+, xz |
| Docker | `rpi-sd-backup-docker` + `Dockerfile` | будь-який хост із Docker; обгортка для macOS/Linux, на Windows — WSL2 або файл `.img` |

## Нативно (macOS / Linux)

```bash
# macOS
brew install xz            # багатопотоковий xz
brew install e2fsprogs     # лише для inspect
brew install mtools        # лише для inspect (читання cmdline.txt із FAT), інакше hdiutil
# Debian/Ubuntu
sudo apt install xz-utils e2fsprogs mtools

ln -s "$PWD/rpi-sd-backup/rpi-sd-backup" /usr/local/bin/rpi-sd-backup
```

```
rpi-sd-backup list                                       знімні диски (кандидати на SD-картку)
rpi-sd-backup backup  <disk|file.img> [опції]            повний образ -> NAME.img.xz (+ sha256, звірка)
rpi-sd-backup verify  <file.img.xz> [raw.sha256|hex]     перевірити архів
rpi-sd-backup inspect <file.img>                         стан ext4, cloud-init, cmdline.txt у сирому образі
rpi-sd-backup restore <file.img[.xz]> <disk> [опції]     записати образ на картку через dd (+ опційно розділ даних)
```

`<disk>`: macOS — `disk5` або `/dev/disk5`; Linux — `/dev/sdb`, `/dev/mmcblk0` (цілий диск, не розділ).
Джерелом `backup` може бути й сирий `.img` (наприклад, знятий Win32DiskImager): тоді він лише
стискається і звіряється.

Опції `backup`:

| Опція | Значення |
|-------|----------|
| `-o, --out DIR` | тека для результату (типово поточна) |
| `-n, --name NAME` | базове ім'я (типово `rpi-sd-<disk>-<дата-час>`) |
| `-l, --level N` | рівень xz 0..9 (типово 6; 9 дає ~3–5 % менше, але вдвічі довше) |
| `-T, --threads N` | потоки xz (типово 0 = усі ядра) |
| `--raw` | додатково зберегти нестиснутий `NAME.img` |
| `--no-verify` | не розпаковувати архів для звірки (економить 1–3 хв) |
| `--force` | дозволити не знімний диск (напр. образ, підключений через hdiutil) |
| `--stdin --size N` | джерело — stdin (так працює Docker-обгортка) |

Опції `restore`:

| Опція | Значення |
|-------|----------|
| `--data-part NG` | після запису додати в кінець картки розділ 3 (Linux, ext4) на N GiB; root виросте до його початку сам на першому завантаженні (див. нижче) |
| `--data-label L` | мітка ФС розділу даних (типово `recordings`) |
| `--data-owner U:G` | числові UID:GID власника кореня розділу даних (типово `0:0`) |
| `--force` | дозволити не знімний диск (напр. образ, підключений через hdiutil) |
| `--stdout` | не писати на диск, а віддати розпакований потік у stdout (так працює Docker-обгортка) |

### Типовий сценарій

```bash
rpi-sd-backup list
# /dev/disk5  15.9 GB  Secure Digital  Removable  <- кандидат
#       1: Windows_FAT_32 system-boot 536.9 MB disk5s1
#       2: Linux                      15.4 GB  disk5s2

rpi-sd-backup backup disk5 -o ~/rpi-backup -n rpi5-ubuntu-user-desktop
```

Результат у `~/rpi-backup/`:

- `rpi5-ubuntu-user-desktop.img.xz` — образ для Imager
- `rpi5-ubuntu-user-desktop.img.xz.sha256` — хеш архіву
- `rpi5-ubuntu-user-desktop.img.sha256` — хеш сирого потоку з картки (для `verify`)

Пристрій картки належить root, тому скрипт попросить пароль `sudo` для `dd` (лише читання).
Орієнтир для картки 16 GB: читання ~6 хв при 47 MB/s, стиснення йде паралельно, звірка ще 1–2 хв.

## Docker

```bash
rpi-sd-backup-docker build                     # один раз: збирає образ rpi-sd-backup:local
rpi-sd-backup-docker list
rpi-sd-backup-docker backup disk5 -o ~/rpi-backup -n rpi5-ubuntu-user-desktop
rpi-sd-backup-docker verify  ~/rpi-backup/rpi5-ubuntu-user-desktop.img.xz
rpi-sd-backup-docker inspect ~/rpi-backup/rpi5-ubuntu-user-desktop.img
rpi-sd-backup-docker restore ~/rpi-backup/rpi5-ubuntu-user-desktop.img.xz disk5
```

Як це влаштовано: контейнер не бачить блокових пристроїв хоста (у Docker Desktop їх немає взагалі),
тому обгортка на хості знаходить і відмонтовує картку, читає її `dd` і передає потік у контейнер
через stdin. У контейнері (Alpine + xz + e2fsprogs + mtools + pv) відбуваються хешування, стиснення
та звірка, результат лягає у примонтовану теку `-o`. Файли для `verify`/`inspect`/`restore`
монтуються лише для читання, `restore` віддає розпакований потік назад на хост у `dd`.

Без обгортки, напряму (будь-яка ОС, де є Docker і спосіб отримати сирий потік або файл):

```bash
# потік із картки (Linux)
sudo dd if=/dev/sdb bs=4M | docker run --rm -i -v "$PWD:/out" rpi-sd-backup:local \
    backup --stdin --size "$(lsblk -bdno SIZE /dev/sdb)" -o /out -n my-pi

# сирий файл (Windows PowerShell; card.img знято, напр., Win32DiskImager)
docker run --rm -v "${PWD}:/work" rpi-sd-backup:local backup /work/card.img -o /work
docker run --rm -v "${PWD}:/work" rpi-sd-backup:local inspect /work/card.img
```

## Прошивка через Raspberry Pi Imager

1. Choose Device → модель Raspberry Pi.
2. Choose OS → в самому низу списку **Use custom** → вибрати `.img.xz` (не розпаковувати).
3. Choose Storage → нова картка. Звірити розмір і назву.
4. Next → на питання про OS customisation відповісти **No**, інакше Imager змінить hostname,
   користувача або Wi-Fi і система перестане бути ідентичною.
5. Дочекатися Writing та Verifying.

## Більша картка: розширення root-розділу

Образ містить розділи в розмірі оригінальної картки. На більшій картці:

- **Ubuntu for Raspberry Pi**: cloud-init (модулі `growpart` + `resizefs`, працюють на кожному
  завантаженні) розширить root сам. Перевірити: `df -h /`.
- **Raspberry Pi OS** (уже налаштована система, не свіжий образ): `sudo raspi-config --expand-rootfs && sudo reboot`.
- Універсально: `sudo growpart /dev/mmcblk0 2 && sudo resize2fs /dev/mmcblk0p2`.

`rpi-sd-backup inspect file.img` показує, який із цих випадків у вас (потрібен сирий `.img`:
`xz -dk file.img.xz`).

## Більша картка: окремий розділ даних (`--data-part`)

Для систем, що пишуть дані (відео, логи) безперервно, краще тримати їх на окремому розділі: переповнення
чи пошкодження цієї ФС не торкається root. `restore --data-part` робить це з хоста, без жодних дій на Pi:

```bash
rpi-sd-backup restore image.img.xz disk5 --data-part 100G --data-label recordings --data-owner 997:984
```

Що відбувається (усе на цілому диску за зсувом, картку не треба виймати між кроками):

1. Образ записується як звичайно.
2. У MBR додається запис 3: Linux (0x83), рівно N GiB, вирівняний на 1 MiB, **у кінці картки**. План
   і помилки (немає `mke2fs`, не вміщається) показуються до підтвердження, а не після 10 хв запису.
3. `mke2fs -t ext4 -E offset=...,root_owner=U:G -m 0` створює ФС на цьому розділі; мітка читається назад для контролю.
   Після кожного запису на цілий диск ОС перечитує таблицю і macOS за секунду сам монтує FAT-розділ,
   від чого диск стає «busy»; тому запис MBR і `mke2fs` виконуються як «відмонтувати → спробувати →
   при busy повторити». Якщо `mke2fs` усе ж не вдасться, образ і MBR уже на картці, а повідомлення
   про помилку містить готову команду для ручного завершення.
4. На першому завантаженні cloud-init `growpart` розтягує root (розділ 2) до початку розділу 3,
   тобто root отримує «решту», а розділ даних — рівно N GiB.

Вимоги: `mke2fs` на хості (macOS: `brew install e2fsprogs`), образ з MBR, де розділ 2 — Linux, а
записи 3 і 4 порожні. Монтування розділу даних має бути **вже в `/etc/fstab` образу**: інструмент
не чіпає вміст root-ФС. Рекомендований рядок (порядок відносно сервісу-споживача гарантує
`x-systemd.before`, бо з `nofail` монтування не впорядковане перед `local-fs.target`; `nofail`
лишає завантаження і резервний запис на root, якщо розділу немає, напр. на меншій картці):

```
LABEL=recordings  /home/recordings  ext4  defaults,noatime,nofail,x-systemd.device-timeout=10s,x-systemd.before=my-service.service  0  2
```

Як одноразово вписати цей рядок у наявний образ на macOS без Linux (ext4 не монтується, але
e2fsprogs уміє редагувати ФС напряму). Нюанси: e2fsck не може зробити replay журналу через
`file?offset=N` і не може писати у raw `/dev/rdiskN` (невирівняний запис суперблоку), тому
працюємо через буферизований `/dev/diskNsM`:

```bash
E2=/opt/homebrew/opt/e2fsprogs/sbin
cp -c orig.img new.img                                      # APFS-клон, миттєво, оригінал не чіпаємо
hdiutil attach -nomount -imagekey diskimage-class=CRawDiskImage new.img      # -> /dev/diskN
$E2/e2fsck -f -p /dev/diskNs2                               # replay журналу + повна перевірка
$E2/debugfs -R "cat /etc/fstab" /dev/diskNs2 > fstab.new && echo 'LABEL=recordings ...' >> fstab.new
printf 'rm /etc/fstab\nwrite fstab.new /etc/fstab\nsif /etc/fstab mode 0100644\nsif /etc/fstab uid 0\nsif /etc/fstab gid 0\n' \
    | $E2/debugfs -w /dev/diskNs2
$E2/e2fsck -fn /dev/diskNs2 && hdiutil detach /dev/diskN   # має бути exit 0
rpi-sd-backup backup new.img -o . -n image-v2               # -> image-v2.img.xz + sha256
```

У Docker-обгортці `--data-part` не підтримується (контейнер не бачить картку, а `mke2fs` потрібен на хості).

## Незакритий журнал ext4

Якщо картку вийняли з увімкненого Raspberry Pi, у ext4 лишається незакритий журнал. Інструмент
навмисно нічого не «лікує»: ядро Linux відтворить журнал при першому монтуванні на новій картці
так само, як зробило б на оригінальній. `inspect` попередить про такий стан.

## Що робить `backup` під капотом

```
джерело (dd /dev/rdiskN | dd /dev/sdX | файл | stdin)
   └─> tee ──> xz -T0 -6 > NAME.img.xz      (архів)
          ├─> >(sha256)                     (хеш сирого потоку)
          └─> NAME.img                      (лише з --raw)
```

Після запису:

1. `xz --robot -l` — нестиснутий розмір архіву має дорівнювати розміру джерела в байтах.
2. `xz -dc NAME.img.xz | sha256` — має дорівнювати хешу сирого потоку (пропускається з `--no-verify`).
3. Записуються `NAME.img.sha256` і `NAME.img.xz.sha256`.

Запобіжники: відмова працювати з системним диском; без `--force` диск має бути SD/USB або
позначений як Removable; `restore` вимагає ввести ідентифікатор диска вручну; при помилці
неповні файли видаляються.
