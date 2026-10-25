# Gaming Backend VM (Streaming + Retro)

Stream Steam games and a retro library from a headless CachyOS VM on Proxmox to a
CachyOS thin-client mini PC in the living room, using Sunshine + Moonlight.

## Decisions

- Dedicated CachyOS gaming VM. The NVIDIA GPU moves from the mediaserver VM.
- Jellyfin on the mediaserver falls back to CPU transcode. Target 1080p; 4K is
  direct-play only unless a second GPU is added later.
- Stream target is 1080p60 to start, using an HDMI dummy plug.
- The client boots into Moonlight. Steam Big Picture and EmulationStation are
  Sunshine apps; Jellyfin runs natively on the client.
- X11 session first. Wayland only if X11 causes problems.
- Plain `steam -gamepadui`; gamescope is a fallback, not the default.
- Ansible manages packages and config files. Proxmox setup, OS install, Steam
  login, and first-run app setup stay manual in this doc.
- Hostname `gaming`, IP `192.168.0.108` (confirm it is free).

## Status

Planned. There is no `gaming` role, inventory group, or playbook entry yet.

## Architecture

```
Proxmox host (headless; GPU already passed through to a VM)
  └── mediaserver VM (192.168.0.16) — GPU removed, Jellyfin on CPU transcode
  └── gaming VM (192.168.0.108, CachyOS) — NVIDIA GPU via VFIO
        ├── Sunshine (host) + NVIDIA NVENC encode
        ├── KDE Plasma (autologin, X11 first, the streamed desktop)
        ├── Steam Big Picture (`steam -gamepadui`)
        ├── EmulationStation-DE + RetroArch
        └── HDMI dummy plug (1080p60)

Client (CachyOS mini PC, wired Ethernet)
  ├── Moonlight (autostart) → gaming VM
  │     └── app grid = [Steam Big Picture, EmulationStation, Desktop]
  └── Jellyfin client → http://192.168.0.16:8096 (native, not streamed)
```

Notes:

- Moonlight is the simple home/TV UI. No kiosk browser on the gaming VM.
- Jellyfin is not a Sunshine app. Playing it natively avoids a second encode.
- The Proxmox host has a single GPU, so it is headless and managed via web UI
  or SSH.

## Phase 1 — Proxmox host prep

The GPU is currently passed through to the mediaserver, so vfio should already
be configured. Verify before changing anything:

```bash
lspci -nnk | grep -iA3 -E 'vga|3d|nvidia'
for g in /sys/kernel/iommu_groups/*; do echo "== $g =="; ls $g/devices/; done
qm config <mediaserver-vmid> | grep hostpci
```

Confirm the GPU and its audio device are both in `hostpci0` and in the same
IOMMU group.

1. Shut down the mediaserver VM and the gaming VM.
2. Remove `hostpci0` from the mediaserver VM (web UI → Hardware → remove).
3. Add the GPU to the gaming VM: **Primary GPU**, **ROM-Bar**,
   **PCI-Express**.
4. Start the gaming VM. A host reboot is normally not needed.

If vfio is not already bound to the GPU, do the full host setup:

- `/etc/modules`: add `vfio`, `vfio_iommu_type1`, `vfio_pci`.
- `/etc/modprobe.d/blacklist.conf`: `blacklist nouveau`, `blacklist nvidia`,
  `blacklist nvidiafb`.
- `/etc/modprobe.d/vfio.conf`:
  `options vfio-pci ids=10de:XXXX,10de:YYYY` (GPU + audio IDs).
- `update-initramfs -u`, reboot, then verify with
  `lspci -k | grep -A2 VGA` → kernel driver in use: `vfio-pci`.

## Phase 2 — Create the CachyOS VM

1. Download the CachyOS ISO.
2. Create the VM:
   - Machine: **q35**
   - BIOS: **OVMF (UEFI)**. No TPM needed for Linux.
   - CPU: type **host**, 6+ cores
   - Memory: 16 GB+, **disable ballooning**
   - Network: VirtIO, bridge `vmbr0`, IP `192.168.0.108/24`
   - Display: **std VGA** for install
   - Disks: OS disk + a **local** disk for the Steam library
   - Options: **Start at boot** (`onboot=1`)
3. Install CachyOS (KDE Plasma):
   - Select the NVIDIA driver option in the installer.
   - Keep Secure Boot off.
   - Enable SSH (`sudo systemctl enable --now sshd`).
   - Enable autologin for `tim` and disable screen lock/display sleep.
   - Add the SSH key: `ssh-copy-id tim@192.168.0.108`.
4. Shut down, add the GPU (see Phase 1), set display to `none`.
   - VNC stops working after this; use SSH or Moonlight.
5. Boot and verify:

   ```bash
   nvidia-smi
   ```

## Phase 3 — Gaming VM software

### Manual first run

- Log into Steam, enable Big Picture, and add the game disk as a library folder.
- Run EmulationStation-DE once and point it at the ROM directory.
- Open the Sunshine web UI (`https://192.168.0.108:47990`), set credentials,
  and pair Moonlight.

### Ansible-managed

Packages (pacman / AUR via `paru`):

- `steam`
- `retroarch` plus cores (snes9x, mupen64plus, mgba, genesis_plus_gx, …)
- `emulationstation-de`
- `sunshine`
- `cifs-utils` only if ROMs stay on the NAS

NVIDIA is installed by the CachyOS installer. The role only verifies
`nvidia-smi`; it does not manage DKMS.

Sunshine config (`~/.config/sunshine/sunshine.conf`):

```ini
encoder = nvenc
```

Sunshine setup:

```bash
echo 'KERNEL=="uinput", SUBSYSTEM=="misc", OPTIONS+="static_node=uinput", TAG+="uaccess"' \
  | sudo tee /etc/udev/rules.d/85-sunshine-input.rules
sudo udevadm control --reload-rules
sudo udevadm trigger
sudo setcap cap_sys_admin+p "$(readlink -f "$(command -v sunshine)")"
systemctl --user enable --now sunshine
```

Note: `setcap` is lost when the Sunshine package updates. The role reapplies it.

Steam:

- Default app command: `steam -gamepadui`
- Fallback if the stream is black or scaled wrong: try `gamescope -- steam -gamepadui`
- Do not add gamescope until it is actually needed.

EmulationStation-DE:

- Add a Sunshine app entry that launches `emulationstation`.
- RetroArch is detected automatically.

Audio:

- Create a persistent PipeWire null sink and select its monitor as the Sunshine
  audio sink. This keeps game audio in the stream and off any physical output.
- Verify audio in Moonlight before adding more apps.

ROMs:

- Prefer a local copy on the game disk. ES-DE scans faster and the stream does
  not depend on the NAS.
- If space is tight, mount the NAS share from `docs/storage_backup.md` at
  `/mnt/nas/roms` and point ES-DE there.

Firewall (only if ufw is enabled):

```bash
sudo ufw allow 47984/tcp
sudo ufw allow 47989/tcp
sudo ufw allow 47990/tcp
sudo ufw allow 48010/tcp
sudo ufw allow 47998/udp
sudo ufw allow 47999/udp
sudo ufw allow 48000/udp
sudo ufw allow 48002/udp
sudo ufw allow 48010/udp
```

## Phase 4 — Virtual display

Without a display the GPU framebuffer is inactive and Sunshine streams black.

- **Use an HDMI dummy plug at 1080p60.** Simple and reliable.
- EDID virtual display is a later option if the plug is not enough. It needs
  kernel params and a custom EDID, and modes are fixed until reboot.

## Phase 5 — Ansible

Add:

```
ansible/roles/gaming/
  tasks/main.yml       # imports setup.yml + sunshine.yml with tags
  tasks/setup.yml      # packages, nvidia check, directories
  tasks/sunshine.yml   # config, udev rule, setcap, user service
  templates/sunshine.conf.j2
  templates/apps.json.j2
```

Inventory:

```ini
[gaming]
192.168.0.108 ansible_user=tim ansible_become=true
```

Playbook:

```yaml
- name: Configure gaming VM
  hosts: gaming
  become: true
  roles:
    - gaming
```

Tags: `gaming_setup`, `gaming_sunshine`.

In the same change that moves the GPU, remove the NVIDIA tasks and the Jellyfin
`runtime: nvidia` from the mediaserver role. Do not remove them before the move,
or the current mediaserver breaks.

## Phase 6 — Client (CachyOS mini PC)

- Install **Moonlight** and pair it with the gaming VM using the PIN from
  `https://192.168.0.108:47990`.
- Apps: Steam Big Picture, EmulationStation, Desktop.
- Install a native Jellyfin client (Jellyfin Media Player or a browser) and
  point it at `http://192.168.0.16:8096`.
- Autostart Moonlight fullscreen on boot.
- Use wired Ethernet. Wi-Fi is the usual cause of stutter and input lag.

## Rollback

If the gaming VM does not work out:

1. Shut down the gaming VM.
2. Remove its `hostpci0`.
3. Add the GPU back to the mediaserver VM.
4. Restore the NVIDIA tasks and Jellyfin `runtime: nvidia` in the mediaserver
   role, or re-run the previous commit.
5. Start the mediaserver and verify `nvidia-smi` and Jellyfin hardware
   transcoding.

## Acceptance checklist

- [ ] `nvidia-smi` works in the gaming VM
- [ ] Sunshine UI reachable at `https://192.168.0.108:47990`
- [ ] Moonlight pairs successfully
- [ ] Steam Big Picture streams at 1080p60
- [ ] Audio plays in the stream
- [ ] Controller input works
- [ ] EmulationStation launches and finds ROMs
- [ ] Jellyfin plays natively on the client
- [ ] Jellyfin on the mediaserver transcodes 1080p on CPU
- [ ] Gaming VM starts automatically after a host reboot

## Open items

- Confirm the GPU model and IOMMU group.
- Confirm `192.168.0.108` is free.
- Decide the Steam library disk size.
- Decide whether ROMs are local or on the NAS.
- Confirm the client machine (`.204`?) and that it is wired.
- Accept that 4K Jellyfin transcodes are out of scope for now.

## Ports (reference)

Sunshine needs: 47984, 47989, 47990, 48010 TCP; 47998–48000, 48002, 48010 UDP.
