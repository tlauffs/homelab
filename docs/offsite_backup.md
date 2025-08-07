# Backup to backblaze
- install rclone on proxmox host
- `rclone config`: 
    - add remote backblaze
    - add remote crypt using backblaze
- rclone:
```
# auth
rclone copy /tank/subvol-104-disk-0/backup/Authentication b2crypt:authentication-backup --progress
# photos
zfs snapshot tank/subvol-106-disk-0@immich-backup && \
rclone copy /tank/subvol-106-disk-0/.zfs/snapshot/immich-backup b2crypt:immich-backup --progress && \
zfs destroy tank/subvol-106-disk-0@immich-backup

```
## cron
```
# photos backup with snapshot at 5 AM on Saturdays
0 5 * * 6 zfs snapshot tank/subvol-106-disk-0@immich-backup && rclone copy /tank/subvol-106-disk-0/.zfs/snapshot/immich-backup b2crypt:immich-backup --progress && zfs destroy tank/subvol-106-disk-0@immich-backup >> /var/log/immich-backup.log 2>&1
```

## restore
- mount: `rclone mount b2crypt: /mnt/backup-decrypted`
- copy: `rclone copy b2crypt:{path} /path/to/local/restore --progress`



