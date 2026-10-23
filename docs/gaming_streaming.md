# Gaming Backend VM (Streaming + Retro)

Stream Steam games and a retro library from a headless CachyOS VM on Proxmox to a
CachyOS thin-client mini PC in the living room, using Sunshine + Moonlight.

## Architecture

```
Proxmox host (headless once GPU moved)
  └── mediaserver VM (192.168.0.16)  — GPU removed, stays headless (jellyfin → CPU transcode)
  └── gaming VM (new, CachyOS)       — NVIDIA GPU via VFIO passthrough
        ├── Sunshine (host) + NVIDIA NVENC encode
        ├── Steam + Big Picture via gamescope
        ├── EmulationStation-DE + RetroArch (retro frontend)
        ├── Jellyfin web app entry (192.168.0.16:8096)
        └── KDE Plasma (autologin, the streamed desktop shell)
              └── HDMI dummy plug on GPU  (or NVIDIA EDID virtual display)

Client (CachyOS mini PC, living room)
  └── Moonlight (autostart) → streams gaming VM; app grid = [Steam BPM, EmulationStation, Jellyfin]
```

Notes:
- The client boots straight into Moonlight — that is the simple home/TV UI. Steam
  Big Picture is exposed as one of the backend's "apps".
- Alternative client shell: native Steam BPM on the client with a Moonlight
  non-Steam shortcut. Less unified; not recommended.

## Phase 1 — Proxmox host prep (one-time, before GPU move)

1. Confirm GPU IDs and IOMMU group:
   ```bash
   lspci -nnk | grep -i nvidia
   for g in /sys/kernel/iommu_groups/*; do echo "== $g =="; ls $g/devices/; done
   ```
   Verify the GPU (and its audio device) sit in their own IOMMU group.
2. Remove the GPU PCI device from the mediaserver VM:
   - Web UI → mediaserver VM → Hardware → remove `hostpci0`.
   - Jellyfin loses HW transcode; keep it running via CPU (or split later).
3. Host kernel config (IOMMU already enabled, `amd_iommu=on` in
   `/etc/default/grub`):
   - `/etc/modules`: add `vfio`, `vfio_iommu_type1`, `vfio_pci`.
   - `/etc/modprobe.d/blacklist.conf`: `blacklist nouveau`, `blacklist nvidia`,
     `blacklist nvidiafb`.
   - `/etc/modprobe.d/vfio.conf`:
     `options vfio-pci ids=10de:XXXX,10de:XXXX` (GPU + audio IDs from step 1).
4. Reboot host. Verify: `lspci -k | grep -A2 VGA` → kernel driver in use:
   `vfio-pci`.
5. Host has a single GPU → Proxmox goes fully headless once passed through.
   Manage via web UI / SSH as before.

## Phase 2 — Create the CachyOS VM

1. Download the CachyOS ISO.
2. Create VM with:
   - Machine: **q35**
   - BIOS: **OVMF (UEFI)**, add TPM 2.0
   - CPU: type **host**, 6+ cores
   - Memory: 16 GB+, **disable Ballooning**
   - Network: VirtIO, bridge vmbr0
   - Display: **std VGA** (for install/VNC before GPU takeover)
   - Disks: OS disk + second SSD/NVMe-emulated disk for the Steam library
3. Install CachyOS (KDE Plasma). Then:
   - Enable SSH (`sudo systemctl enable --now sshd`).
   - Enable auto-login for `tim` and disable screen/display sleep (Sunshine needs
     a logged-in user session).
   - Add SSH key: `ssh-copy-id tim@<gaming-ip>`.
4. Shut down VM, add passthrough GPU:
   - Web UI → VM → Hardware → Add → PCI Device → NVIDIA GPU.
   - Enable **Primary GPU**, **ROM-Bar**, **PCI-Express**.
   - `--vga none` (VNC will stop working; access via SSH / Moonlight after this).

## Phase 3 — Gaming VM software (CachyOS)

Install via pacman / AUR (`yay`):

- NVIDIA driver (current on CachyOS): `nvidia-dkms` handled by CachyOS defaults.
- Sunshine: `yay -S sunshine` (AUR). Configure:
  ```ini
  ~/.config/sunshine/sunshine.conf
  encoder = nvenc
  ```
  - Virtual input udev rule +
    `echo 'KERNEL=="uinput", SUBSYSTEM=="misc", OPTIONS+="static_node=uinput", TAG+="uaccess"' > /etc/udev/rules.d/85-sunshine-input.rules`
  - `sudo setcap cap_sys_admin+p $(readlink -f $(which sunshine))`
  - Enable user service: `systemctl --user enable --now sunshine`
- Steam + gamescope:
  - `sudo pacman -S steam gamescope`
  - Sunshine app entry: `gamescope -- steam -bigpicture`
  - Fallback if gamescope+NVIDIA misbehaves: plain `steam -bigpicture` on KDE.
- RetroArch + EmulationStation-DE:
  - `yay -S retroarch emulationstation-de`
  - Install cores: snes9x, mupen64plus, gba/mgba, genesis_plus_gx, etc.
  - ES-DE auto-detects RetroArch; add a Sunshine app entry for ES-DE.
- Jellyfin app entry in Sunshine apps.json: open `http://192.168.0.16:8096` in a
  kiosk browser (e.g. firefox-kde).
- Firewall (ufw, if enabled):
  ```
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
- PipeWire audio + persistent null sink so game audio only goes to the stream.
- ROMs storage: mount NAS share via fstab
  (`//192.168.0.104/data` from the storage LXC) at `/mnt/nas/roms` using the
  samba credentials pattern from `docs/storage_backup.md`.

## Phase 4 — Virtual display (headless NVIDIA, no monitor)

Without any display the GPU framebuffer is inactive and Sunshine streams black.

- Option A (recommended): **HDMI/DP dummy plug** on the GPU — fixed resolution
  (1080p60 / 4K60 EDID), simple and reliable.
- Option B (no hardware): **NVIDIA EDID virtual display** — kernel params
  `drm.edid_firmware=<conn>:edid/<file> video=<conn>:e` with a custom EDID from
  initramfs. Works 1080p/1440p/4K, but no HDR; modes fixed until reboot.

Start with Option A.

## Phase 5 — Homelab automation (repo)

- New `ansible/roles/gaming/` — pacman-based role (first Arch-role):
  install packages, configure Sunshine + app entries, gamescope Steam launcher,
  PipeWire sink, fstab NAS mount, autologin, systemd user services.
- Add `[gaming]` group to `ansible/inventory/hosts.ini` (IP e.g. 192.168.0.2xx).
- Playbook entry: `hosts: gaming` → role `gaming`.
- This doc doubles as the manual Proxmox-side setup reference (Phases 1–2).

## Phase 6 — Client (CachyOS mini PC)

- Install **Moonlight**.
- Add the gaming VM by LAN IP, pair with the PIN from
  `https://<gaming-vm>:47990`.
- List apps: Steam Big Picture, EmulationStation, Jellyfin, Desktop.
- Enable fullscreen autostart of Moonlight on boot.

## Open items before/while executing

- Exact GPU model (RTX 2xxx+ assumed) — needed for driver/EDID details.
- Dummy plug vs EDID virtual display (recommend dummy plug first).
- Client shell: Moonlight-as-home-UI (recommended) vs native Steam BPM.

## Ports (reference)

Sunshine needs: 47984, 47989, 47990, 48010 TCP; 47998–48000, 48002, 48010 UDP.