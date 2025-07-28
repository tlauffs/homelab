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
- `docs/`: install/setup documention

## Docs

- [Proxmox Setup](./docs/proxmox_setup.md)
- [Deb lxc Ansible Playbook Guide](docs/deb_lxc.md)
- [WireGuard VPS Tunnel](./docs/wireguard_vps_proxy.md)
- [Mount and Backup](./docs/backup.md)


