# Backup desktop 
```bash
rsync -av --delete --dryrun --info=progress2 \
        --exclude='nas' \
        --exclude='.*' \
        --exclude='yay' \
        --exclude='Games' \
        --exclude='Downloads' \
        ~/ tim@192.168.0.10x:/data/backup/arch_desktop
```

# Mount

## with sshfs
- mount: `sshfs tim@192.168.0.10x:/ ~/nas/x`
- unmount: `sudo umount ~/nas`

## with fstab (permanently)

- create samba user `sudo smbpasswd -a tim`
### mount on client
- `sudo pacman -Syu cifs-utils smbclient`
- test: `smbclient //192.168.0.10x/data -U tim`

- sudo vim /etc/samba/credentials_tim:
```
username=tim
password=...
```
- `sudo chmod 600 /etc/samba/credentials_tim`
- sudo vim /etc/fstab: 
```
//192.168.0.10x/data  /home/tim/nas cifs  credentials=/etc/samba/credentials_tim,uid=1000,gid=1000,iocharset=utf8  0  0
```
- mount: `mount -a`
- unmount: `sudo umount /home/tim/nas`

# Backup Proxmox vms
- in pve:
- create a backup directory in the tank pool: `mkdir -p /tank/vm_backup`
    - `chown root:root /tank/vm_backup`
    - `chmod 755 /tank/vm_backup`
- add as directory to proxmox storage:
    - Datacenter > Storage > Add > Directory
        - content: backup
- auto backups: datacenter > backup > add

