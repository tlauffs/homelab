# Running Servers
- Valheim IP: valheim.tlauffs.com:2456
- Minecraft IP: minecraft.tlauffs.com:25565

## Create Deb Game Server VM
- setup linux (deb) vm in proxmox 
- `https://cdimage.debian.org/debian-cd/current/amd64/iso-cd/debian-12.11.0-amd64-netinst.iso`
- setup debian as ssh server
- add nas storage vm > hardware > add harddisk
- add ssh key
- mount nas storage (temporary):
    - `sudo mkfs.ext4 /dev/sdb`
    - `sudo mkdir /data`
    - `sudo mount /dev/sdb /data`
    - `sudo chown -R tim:tim /data`
    - `lsblk`

## Nvida GPU setup
- add gpu hardware device to proxmox
- add /etc/apt/sources.list entry for nvidia drivers
    deb http://deb.debian.org/debian bookworm main contrib non-free non-free-firmware
    deb http://deb.debian.org/debian bookworm-updates main contrib non-free non-free-firmware
    deb http://security.debian.org/debian-security bookworm-security main contrib non-free non-free-firmware

- sudo apt update
- run `nvidia-detect`
- install recommended package
