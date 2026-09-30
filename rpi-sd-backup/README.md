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
rpi-sd-backup restore <file.img[.xz]> <disk>             записати образ на картку через dd
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
