# Create main Deb VM
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
ansible-playbook -i ansible/inventory/hosts.ini ansible/playbook.yml --ask-become-pass --ask-vault-pass -l debian_lxc
```

### Only Base System Setup
This will run only the tasks related to the base system configuration (packages, users, Docker installation, UFW).
```bash
ansible-playbook -i ansible/inventory/hosts.ini ansible/playbook.yml --tags "deb_lxc_setup" --ask-become-pass --ask-vault-pass  -l debian_lxc
```

### Only Docker Container Deployment
This will run only the tasks related to deploying Docker containers via `docker-compose`.
```bash
ansible-playbook -i ansible/inventory/hosts.ini ansible/playbook.yml --tags "deb_lxc_deploy_containers" --ask-become-pass --ask-vault-pass  -l debian_lxc
```
