# Mount nas / Backup desktop 

- requires main nas deb lxc to be running
- mount: `sshfs tim@192.168.0.100:/ ~/nas`
- unmount: `sudo umount ~/nas`
- backup desktop:
```bash
rsync -av --delete --dryrun --info=progress2 \
        --exclude='nas' \
        --exclude='.*' \
        --exclude='yay' \
        --exclude='Games' \
        --exclude='Downloads' \
        ~/ tim@192.168.0.100:/data/backup/arch_desktop
```
