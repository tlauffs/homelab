# Homelab Project

This repository contains configurations and automation for my homelab setup.

# Structure

- `ansible/`: Contains all Ansible automation for provisioning and deploying services.
    - `inventory/`: Hosts
    - `group_vars/`: 
        - `all.yml`: Global variables
    - `roles/`: Roles for setting up VMs
    - `playbook.yml`: The main playbook to orchestrate the roles.
- `docs/`: install/setup documention

## Docs

- [Proxmox Setup](./docs/proxmox_setup.md)
- [Backup/Storage Setup](docs/storage_backup.md)
- [WireGuard VPS Tunnel](./docs/wireguard_vps_proxy.md)
- [Server Setup (media/game)](./docs/server_vms.md)
- [Monitoring](./docs/monitoring.md)

## setup VMs with ansible
 
### Setup VM with roles
```bash
ansible-playbook -i ansible/inventory/hosts.ini ansible/playbook.yml --ask-become-pass --ask-vault-pass -l {role} --tags {optional tags}
```

## Base config of Debian based lxcs/vms:
- Download Template: local > CT Templates > Templates > debian
- Create Container: pve > Create Container
    - set specs
    - set Network (ipv4): CIDR: 192.168.0.10x/24 and Gateway `ip route | grep default`
- add nas storage:
    - vm > resources > add > mount point (xxxgb) > /data
- add docker mount point
    - vm > resources > add > mount point (xxgb) > /docker

### Initial User Setup on Debian LXC
- `apt update && apt upgrade -y`
- `adduser tim && adduser tim sudo`
- add ssh key: ssh-copy-id tim@{ip a}
- `sudo vim /etc/ssh/sshd_config`
    - uncomment/add `PasswordAuthentication no`
- `sudo systemctl reload sshd`
