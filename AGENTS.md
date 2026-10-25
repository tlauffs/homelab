# AGENTS.md

## What this repo is

Infrastructure-as-code for a personal homelab: Proxmox LXCs/VMs plus one
Hetzner VPS. Ansible does the repeatable work; `docs/` holds manual and
one-time setup.

This is not a product or a platform. Keep it small enough that one person can
understand and repair it.

## Layout

- `ansible/inventory/hosts.ini` — one group per host, used with `-l <group>`
- `ansible/group_vars/all.yml` — shared vars and vaulted secrets
- `ansible/playbook.yml` — one play per host
- `ansible/roles/<host>/` — `setup.yml`, `deploy_containers.yml`, `templates/`
- `docs/` — per-topic setup notes and runbooks

## Architecture

- One role per host/VM/LXC. Role name == inventory group == playbook limit.
- `setup_lxc` is the shared base for Debian LXCs. The mediaserver VM and
  gameserver VPS have their own setup tasks.
- Services run in Docker. Each role templates `docker-compose.yml.j2` to
  `/docker/docker-compose.yml` and applies it with
  `community.docker.docker_compose_v2`.
- Persistent data goes on `/data`; container config goes on `/docker`.
- Secrets are vaulted in `ansible/group_vars/all.yml` (`!vault`). Never plaintext.
- Proxmox-side and client-side manual steps stay in `docs/`, not in Ansible.

## Commands

```bash
# everything (prompts for become + vault password)
ansible-playbook -i ansible/inventory/hosts.ini ansible/playbook.yml \
  --ask-become-pass --ask-vault-pass

# one host
ansible-playbook -i ansible/inventory/hosts.ini ansible/playbook.yml \
  --ask-vault-pass -l mediaserver

# one part of a role
ansible-playbook -i ansible/inventory/hosts.ini ansible/playbook.yml \
  --ask-vault-pass -l mediaserver --tags mediaserver_deploy
```

## Rules for changes

1. **Simple first.** Prefer the smallest change that solves the problem.
2. **Prefer fewer layers.** Avoid adding tooling, collections, roles, or
   abstractions when the current setup can do the job. If there is a good reason
   to abstract — or something like CI/CD, Kubernetes, Terraform, or a custom
   framework genuinely helps — ask the user first.
3. **Automate repeatable work.** If Ansible can do a task safely, put it in the
   role instead of documenting it as a manual step. Manual steps are the
   exception. Guard destructive actions (for example, format a disk only when
   it is empty). Only work that genuinely cannot be automated — creating the
   VM, installing the OS, logins/pairing, physical setup — stays manual in
   `docs/`.
4. **Prefer official packages and Docker.** Use the official repositories
   first, and run services in Docker instead of installing them on the host.
   Avoid AUR packages; prefer a container, Flatpak, or upstream release. Ask
   before adding an AUR dependency. Host-session tools that need direct access
   to hardware or the desktop are the exception.
5. **Follow existing patterns.** Same role layout, compose template, tags, and
   `homelab_user` / `/data` / `/docker` conventions.
6. **Scope discipline.** Only touch the host you're working on; ask before
   changing shared vars or unrelated roles.
7. **Docs stay in sync.** Update the matching file in `docs/` in the same change.
8. **Verify narrowly.** Run with `-l <host>` and `--check` where possible before
   running across the whole inventory.
9. **Secrets never go in plaintext.** Use `ansible-vault`.
10. **Never connect to remote hosts.** Do not run `ssh`, `ansible`, or any other
    command against a host. The user runs deployment and verification commands;
    propose the exact command instead. Local repo inspection is fine.
