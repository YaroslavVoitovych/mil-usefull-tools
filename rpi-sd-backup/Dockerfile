# rpi-sd-backup у контейнері: xz, e2fsprogs (inspect), mtools (читання FAT без монтування), pv (прогрес stdin).
# Контейнер не має доступу до блокових пристроїв хоста: картка надходить потоком через stdin
# (див. rpi-sd-backup-docker), результат пишеться у примонтовану теку /out.
FROM alpine:3.20

RUN apk add --no-cache bash coreutils util-linux xz e2fsprogs e2fsprogs-extra mtools pv

COPY rpi-sd-backup /usr/local/bin/rpi-sd-backup
RUN chmod +x /usr/local/bin/rpi-sd-backup

WORKDIR /out
ENTRYPOINT ["/usr/local/bin/rpi-sd-backup"]
CMD ["--help"]
