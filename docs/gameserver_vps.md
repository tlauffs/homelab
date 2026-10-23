# Game servers

Valheim and Minecraft run on a Hetzner VPS, not in the homelab.

- `valheim.tlauffs.com`
- `minecraft.tlauffs.com`

## Deploy

```bash
ansible-playbook -i ansible/inventory/hosts.ini ansible/playbook.yml \
  --ask-vault-pass -l gameserver_vps
```

## Profiles

The VPS is small, so only one server runs at a time. Each service is behind a
Docker profile: `mc` or `valheim`.

```bash
cd /docker
docker compose --profile mc up -d
```

Set `gameserver_profile` in the inventory to match. Stop the running one before
starting another.

## Backups

Enable backups in the Hetzner console, and turn on the images' own world
backups as well (Hetzner's are whole-disk snapshots, not world-consistent).
