# mil-usefull-tools

Набір невеликих CLI-інструментів. Кожен інструмент живе у власній теці зі своїм README.

| Інструмент | Призначення | Запуск |
|------------|-------------|--------|
| [rpi-sd-backup](rpi-sd-backup/) | Бекап SD-картки Raspberry Pi одним файлом `.img.xz` для Raspberry Pi Imager: читання, стиснення, звірка, огляд образу, відновлення (на більшу картку — з окремим розділом даних, `--data-part`) | нативно (macOS/Linux) або через Docker (`rpi-sd-backup-docker`) |

## Встановлення

```bash
git clone https://github.com/YaroslavVoitovych/mil-usefull-tools.git
cd mil-usefull-tools
ln -s "$PWD/rpi-sd-backup/rpi-sd-backup"        /usr/local/bin/rpi-sd-backup
ln -s "$PWD/rpi-sd-backup/rpi-sd-backup-docker" /usr/local/bin/rpi-sd-backup-docker
```
