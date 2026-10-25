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
3. **Manual one-time work stays manual.** Document it in `docs/` instead of
   automating it.
4. **Follow existing patterns.** Same role layout, compose template, tags, and
   `homelab_user` / `/data` / `/docker` conventions.
5. **Scope discipline.** Only touch the host you're working on; ask before
   changing shared vars or unrelated roles.
6. **Docs stay in sync.** Update the matching file in `docs/` in the same change.
7. **Verify narrowly.** Run with `-l <host>` and `--check` where possible before
   running across the whole inventory.
8. **Secrets never go in plaintext.** Use `ansible-vault`.
