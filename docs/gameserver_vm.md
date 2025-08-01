# Running Servers
- Valheim IP: valheim.tlauffs.com:2456
- Minecraft IP: minecraft.tlauffs.com:25565

## Create Deb Game Server VM
- setup linux (deb) vm in proxmox 
- `https://cdimage.debian.org/debian-cd/current/amd64/iso-cd/debian-12.11.0-amd64-netinst.iso`
- setup debian as ssh server
- add nas storage vm > hardware > add harddisk
- add ssh key
- mount nas storage:
    - `sudo mkfs.ext4 /dev/sdb`
    - `sudo mkdir /data`
    - `sudo mount /dev/sdb /data`
    - `sudo chown -R tim:tim /data`
    - `lsblk`
