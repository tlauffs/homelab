# Running Server
- Valheim IP: valheim.tlauffs.com:2456
# Create Deb Game Server
- setup linux (deb) vm in proxmox 
- `https://cdimage.debian.org/debian-cd/current/amd64/iso-cd/debian-12.11.0-amd64-netinst.iso`
- setup debian as ssh server
- add nas storage vm > hardware > add harddisk
- add ssh key: ssh-copy-id tim@{ip a}
- `apt update && apt upgrade -y`
- `adduser tim sudo`
- `sudo vim /etc/ssh/sshd_config`
- uncomment/add `PasswordAuthentication no`
- `sudo systemctl reload sshd`
- mount nas storage:
    - `sudo mkfs.ext4 /dev/sdb`
    - `sudo mkdir /data`
    - `sudo mount /dev/sdb /data`
    - `sudo chown -R tim:tim /data`
    - `lsblk`

# Setup with Ansible

## Running Ansible Playbooks

All Ansible automation is run via the `ansible/playbook.yml` file.

Variables and vaulted secrets are managed in `ansible/group_vars/all.yml`.

### Full Setup (Base System + Docker Containers)
This will run all tasks in the `gameserver` role.
```bash
ansible-playbook -i ansible/inventory/hosts.ini ansible/playbook.yml --ask-become-pass -l gameserver 
```

### Only Base System Setup
This will run only the tasks related to the base system configuration (packages, users, Docker installation, UFW).
```bash
ansible-playbook -i ansible/inventory/hosts.ini ansible/playbook.yml --tags "gameserver_setup" --ask-become-pass -l gameserver
```

### Only Docker Container Deployment
This will run only the tasks related to deploying Docker containers via `docker-compose`.
```bash
ansible-playbook -i ansible/inventory/hosts.ini ansible/playbook.yml --tags "gameserver_deploy_containers" --ask-become-pass --ask-vault-pass -l gameserver
```
