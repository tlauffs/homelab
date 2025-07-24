# Homelab Project

This repository contains configurations and automation for my homelab setup.

# Structure

- `ansible/`: Contains all Ansible automation for provisioning and deploying services.
    - `inventory/`: Hosts
    - `group_vars/`: 
        - `all.yml`: Global variables
    - `roles/`: 
        - `deb_lxc/`: Role for setting Debian LXC container.
            - `tasks/`:  
            - `templates/`:
    - `playbook.yml`: The main playbook to orchestrate the roles.

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

## Initial User Setup on Debian LXC
- `apt update && apt upgrade -y`
- `adduser tim && adduser tim sudo`
- Add your SSH key to the Debian host for passwordless access.

## Running Ansible Playbooks

All Ansible automation is run via the `ansible/playbook.yml` file.

Variables and vaulted secrets are managed in `ansible/group_vars/all.yml`.

### Full Setup (Base System + Docker Containers)
This will run all tasks in the `deb_lxc` role.
```bash
ansible-playbook -i ansible/inventory/hosts.ini ansible/playbook.yml --ask-become-pass --ask-vault-pass
```

### Only Base System Setup
This will run only the tasks related to the base system configuration (packages, users, Docker installation, UFW).
```bash
ansible-playbook -i ansible/inventory/hosts.ini ansible/playbook.yml --tags "deb_lxc_setup" --ask-become-pass --ask-vault-pass
```

### Only Docker Container Deployment
This will run only the tasks related to deploying Docker containers via `docker-compose`.
```bash
ansible-playbook -i ansible/inventory/hosts.ini ansible/playbook.yml --tags "deb_lxc_deploy_containers" --ask-become-pass --ask-vault-pass
```

# Mount / Backup

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

