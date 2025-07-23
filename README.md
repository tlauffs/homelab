# Structure

- `ansible/`: all configuration logic
- `docker/`: docker-compose services

# Setup Proxmox

## Change Repository
- Repositories: change Proxmox repository from enterprise to community/no subscription
- Update: Refresh and Upgrade (apt update & apt upgrade)

## Enable IOMMU
- `vim /etc/default/grub`
- to `GRUB_CMDLINE_LINUX_DEFAULT="quiet"` add: intel_iommu=on or amd_iommu=on.
- `GRUB_CMDLINE_LINUX_DEFAULT="quiet amd_iommu=on"`
- `update-grub` 
- `update-initramfs -u`
- reboot
- test if IOMMU is enabled
```
dmesg | grep -e DMAR -e IOMMU
dmesg | grep 'remapping'
dmesg | grep -e IOMMU -e AMD-Vi
```

## Create Storage
- disks > ZFS > create: ZFS
- pick drives and raid level

## Create main Deb VM
- Download Template: local > CT Templates > Templates > debian
- Create Container: pve > Create Container
    - set specs
    - set Network (ipv4): CIDR: 192.168.0.100/24 and Gateway `ip route | grep default`
- add nas storage:
    - vm > resources > add > mount point (500gb)
- add docker mount point
    - vm > resources > add > mount point (32gb)

# Setup with Ansible

## create user
- `apt update && apt upgrade -y`
- `adduser tim && adduser tim sudo`

# setup debian vm
- add ssh key to debian host
- `ssh tim@192.168.0.100`
- setup debian vm: ` ansible-playbook -i ansible/inventory/hosts.ini ansible/playbooks/setup-deb.yml --ask-become-pass`

# mount / backup
- mount: `sshfs tim@192.168.0.100:/ ~/nas`
- unmount: `sudo umount ~/nas`
- backup desktop:
```
rsync -av --delete --dryrun --info=progress2 \
        --exclude='nas' \
        --exclude='.*' \
        --exclude='yay' \
        --exclude='Games' \
        --exclude='Downloads' \
        ~/ tim@192.168.0.100:/data/backup/arch_desktop
```
