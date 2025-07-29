# Create Deb Game Server
- see ./deb_lxc.md 

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
