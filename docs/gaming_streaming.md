# Gaming Backend VM (Streaming + Retro)

Stream Steam games and a retro library from a headless CachyOS VM on Proxmox to a
living-room mini PC, using Sunshine + Moonlight.

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
- Hostname `gaming`, IP `192.168.0.79` (current DHCP lease; pin it with a DHCP
  reservation or a static config in CachyOS).
- GPU is an RTX 2070 (8 GB, Turing/NVENC), IOMMU group 15, currently passed
  through as `0000:09:00`.

## Status

Planned. There is no `gaming` role, inventory group, or playbook entry yet.

## Architecture

```
Proxmox host (headless; GPU already passed through to a VM)
  └── mediaserver VM (192.168.0.16) — GPU removed, Jellyfin on CPU transcode
  └── gaming VM (192.168.0.79, CachyOS) — NVIDIA RTX 2070 via VFIO
        ├── Sunshine (host) + NVIDIA NVENC encode
        ├── KDE Plasma (autologin, X11 first, the streamed desktop)
        ├── Steam Big Picture (`steam -gamepadui`)
        ├── EmulationStation-DE + RetroArch
        └── HDMI dummy plug (1080p60)

Client (living-room mini PC, wired Ethernet)
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

The GPU is already passed through to the mediaserver, so vfio is configured.
The card is an RTX 2070 at `09:00.0`, with audio, USB-C, and serial functions at
`09:00.1`–`09:00.3`. All four are in IOMMU group 15 and bound to `vfio-pci`.
The mediaserver passes the whole group as `0000:09:00`.

Verify before changing anything:

```bash
lspci -nnk | grep -iA3 -E 'vga|3d|nvidia'
readlink /sys/bus/pci/devices/0000:09:00.0/iommu_group
qm config 108 | grep hostpci
```

1. Shut down the mediaserver VM and the gaming VM.
2. Remove `hostpci0` from the mediaserver VM:
   `qm set 108 --delete hostpci0`
3. Add the GPU to the gaming VM:
   `qm set 200 --hostpci0 0000:09:00,pcie=1,rombar=1`
   In the web UI that is **Primary GPU**, **ROM-Bar**, **PCI-Express**.
4. Start the gaming VM. A host reboot is normally not needed.

If vfio is not already bound to the GPU, do the full host setup:

- `/etc/modules`: add `vfio`, `vfio_iommu_type1`, `vfio_pci`.
- `/etc/modprobe.d/blacklist.conf`: `blacklist nouveau`, `blacklist nvidia`,
  `blacklist nvidiafb`.
- `/etc/modprobe.d/vfio.conf`:
  `options vfio-pci ids=10de:1f02,10de:10f9` (RTX 2070 + audio).
- `update-initramfs -u`, reboot, then verify with
  `lspci -k | grep -A2 VGA` → kernel driver in use: `vfio-pci`.

## Phase 2 — Create the CachyOS VM

The Proxmox host is `192.168.0.204`. `local-lvm` is the 1 TB NVMe; `tank` is a
2×4 TB HDD mirror. Keep the OS and games on `local-lvm`.

1. Download and verify the ISO on the Proxmox host:

   ```bash
   cd /var/lib/vz/template/iso
   wget https://iso.cachyos.org/desktop/260809/cachyos-desktop-linux-260809.iso
   sha256sum cachyos-desktop-linux-260809.iso
   # 959f6577f45e25ee9fd8c220fd221b08e4ea79412c7315c0f922dd6d86d5e33c
   ```

2. Create VM `200` (`gaming`):
   - Machine: **q35**
   - BIOS: **OVMF (UEFI)**. No TPM needed for Linux.
   - CPU: type **host**, 6 cores
   - Memory: **12 GB**, **disable ballooning**. The host has 31 GB and the
     mediaserver uses 8 GB; raise this only after checking swap.
   - Network: VirtIO, bridge `vmbr0`, IP `192.168.0.79/24`, gateway
     `192.168.0.1`
   - Display: **std VGA** for install
   - Disks: 80 GB OS disk on `local-lvm` + 500 GB game disk on `local-lvm`
   - Options: **Start at boot** (`onboot=1`), QEMU agent enabled

   ```bash
   qm create 200 \
     --name gaming \
     --ostype l26 \
     --machine q35 \
     --bios ovmf \
     --efidisk0 local-lvm:1,efitype=4m,pre-enrolled-keys=0 \
     --cpu host \
     --sockets 1 \
     --cores 6 \
     --memory 12288 \
     --balloon 0 \
     --scsihw virtio-scsi-single \
     --scsi0 local-lvm:80,discard=on,ssd=1,iothread=1 \
     --scsi1 local-lvm:500,discard=on,ssd=1,iothread=1 \
     --ide2 local:iso/cachyos-desktop-linux-260809.iso,media=cdrom \
     --net0 virtio,bridge=vmbr0,firewall=0 \
     --vga std \
     --boot order=ide2 \
     --onboot 1 \
     --agent enabled=1
   qm start 200
   ```

3. Install CachyOS (KDE Plasma) **without the GPU**. The installer cannot
   detect the RTX 2070 yet, so do not select an NVIDIA option.
   - Keep Secure Boot off.
   - Install to the 80 GB disk. Leave the 500 GB disk alone for now.
   - Enable SSH (`sudo systemctl enable --now sshd`).
   - Set the static IP with `nmcli` or a DHCP reservation.
   - Enable autologin for `tim`, pick the **Plasma (X11)** session, and disable
     screen lock/display sleep.
   - Add the SSH key: `ssh-copy-id tim@192.168.0.79`.
   - Install the guest agent: `sudo pacman -S --needed qemu-guest-agent` and
     `sudo systemctl enable --now qemu-guest-agent`.
4. Remove the ISO and boot from disk:

   ```bash
   qm set 200 --delete ide2
   qm set 200 --boot order=scsi0
   ```

5. Shut down, add the GPU (see Phase 1), set display to `none`:
   `qm set 200 --vga none`
   - VNC stops working after this; use SSH.
6. Boot, then install the NVIDIA driver with CachyOS hardware detection:

   ```bash
   sudo chwd -a
   sudo reboot
   nvidia-smi
   ```

   If `chwd` picks the wrong profile, use
   `sudo chwd -i nvidia-dkms-580xx` (or `nvidia-open-dkms`). Turing is
   supported by both.

## Phase 3 — Gaming VM software

### Manual first run

- Log into Steam, enable Big Picture, and add the game disk as a library folder.
- Run EmulationStation-DE once and point it at the ROM directory.
- Open the Sunshine web UI (`https://192.168.0.79:47990`), set credentials,
  and pair Moonlight.

### Ansible-managed

Packages (pacman / AUR via `paru`):

- `steam`
- `retroarch` plus cores (snes9x, mupen64plus, mgba, genesis_plus_gx, …)
- `emulationstation-de`
- `sunshine`
- `cifs-utils` only if ROMs stay on the NAS

NVIDIA is installed after passthrough with `sudo chwd -a`. The role only
verifies `nvidia-smi`; it does not manage DKMS.

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
sudo ufw allow 22/tcp
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
192.168.0.79 ansible_user=tim ansible_become=true
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

## Phase 6 — Client (living-room mini PC)

- CachyOS is fine if the client already runs it. If installing fresh and you
  want appliance-like behavior, Bazzite or ChimeraOS is simpler; the client only
  needs Moonlight and a Jellyfin client.
- Install **Moonlight** and pair it with the gaming VM using the PIN from
  `https://192.168.0.79:47990`.
- Apps: Steam Big Picture, EmulationStation, Desktop.
- Install a native Jellyfin client (Jellyfin Media Player or a browser) and
  point it at `http://192.168.0.16:8096`.
- Autostart Moonlight fullscreen on boot.
- Use wired Ethernet. Wi-Fi is the usual cause of stutter and input lag.

## Rollback

If the gaming VM does not work out:

1. Shut down the gaming VM.
2. Remove its `hostpci0`: `qm set 200 --delete hostpci0`
3. Add the GPU back to the mediaserver VM:
   `qm set 108 --hostpci0 0000:09:00`
4. Restore the NVIDIA tasks and Jellyfin `runtime: nvidia` in the mediaserver
   role, or re-run the previous commit.
5. Start the mediaserver and verify `nvidia-smi` and Jellyfin hardware
   transcoding.

## Acceptance checklist

- [ ] `nvidia-smi` works in the gaming VM
- [ ] Sunshine UI reachable at `https://192.168.0.79:47990`
- [ ] Moonlight pairs successfully
- [ ] Steam Big Picture streams at 1080p60
- [ ] Audio plays in the stream
- [ ] Controller input works
- [ ] EmulationStation launches and finds ROMs
- [ ] Jellyfin plays natively on the client
- [ ] Jellyfin on the mediaserver transcodes 1080p on CPU
- [ ] Gaming VM starts automatically after a host reboot

## Open items

- Confirm `192.168.0.79` is reserved or statically configured.
- Confirm 500 GB is enough for the game library.
- Decide whether ROMs are local or on the NAS.
- Confirm the client machine and that it is wired.
- Confirm the client OS choice (CachyOS vs Bazzite/ChimeraOS).
- Decide whether to add RAM before raising the gaming VM above 12 GB.
- Accept that 4K Jellyfin transcodes are out of scope for now.

## Ports (reference)

Sunshine needs: 47984, 47989, 47990, 48010 TCP; 47998–48000, 48002, 48010 UDP.
