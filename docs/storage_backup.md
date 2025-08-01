# Backup desktop 
```bash
rsync -av --delete --dryrun --info=progress2 \
        --exclude='nas' \
        --exclude='.*' \
        --exclude='yay' \
        --exclude='Games' \
        --exclude='Downloads' \
        ~/ tim@192.168.0.104:/data/backup/arch_desktop
```

# Mount

## with sshfs
- mount: `sshfs tim@192.168.0.10x:/ ~/nas/x`
- unmount: `sudo umount ~/nas`

## with fstab (permanently)

- create samba user `sudo smbpasswd -a tim`
### mount on client
- `sudo pacman -Syu cifs-utils smbclient`
- test: `smbclient //192.168.0.104/data -U tim`

- sudo vim /etc/samba/credentials_tim:
```
username=tim
password=...
```
- `sudo chmod 600 /etc/samba/credentials_tim`
- sudo vim /etc/fstab: 
```
//192.168.0.104/data  /home/tim/nas cifs  credentials=/etc/samba/credentials_tim,uid=1000,gid=1000,iocharset=utf8  0  0
```
- mount: `mount -a`
- unmount: `sudo umount /home/tim/nas`

