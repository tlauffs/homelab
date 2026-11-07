# Gaming VM (CachyOS + Sunshine)

Stream Steam games and a retro library from a headless CachyOS VM on Proxmox to
a living-room mini PC. Sunshine runs on the VM, Moonlight on the client.

**Status:** VM created, CachyOS installed, SSH working, GPU passed through with
`nvidia-smi` working, and native Sunshine installed with its web UI reachable
directly and through NPM. A monitor connected to the passed-through GPU is
currently providing a KMS output, and Sunshine now detects it and initializes
NVENC. Permanent headless capture still needs the HDMI dummy plug. Steam login
and ES-DE first-run setup are still to do.

## Decisions

- Dedicated CachyOS VM. The NVIDIA GPU moves from the mediaserver VM.
- Jellyfin on the mediaserver falls back to CPU transcode. Target 1080p; 4K is
  direct-play only unless a second GPU is added later.
- Stream at 1080p60 using an HDMI dummy plug.
- Wayland only: Plasma 6.8 has no X11 session. Use native Sunshine with KMS
  capture; Flatpak can't do KMS, and its XDG portal path needs an interactive
  screen-share prompt that a headless VM can't answer.
- Plain `steam -gamepadui`; gamescope is a fallback, not the default.
- Ansible owns packages, guest agent, autologin, firewall, Sunshine install and
  capability, and formatting and mounting the game disk. Proxmox setup, OS
  install, Steam login and first-run app setup stay manual in this doc.
- ROMs live on the game disk at `/mnt/games/ROMs`, reached through a `~/ROMs`
  symlink so ES-DE's own config never has to be templated.
- Hostname `gaming` (VM 200), IP `192.168.0.79`.
- GPU: RTX 2070 (8 GB, Turing/NVENC), IOMMU group 15, currently `0000:09:00`.

## Architecture

```text
Proxmox host 192.168.0.204 (headless, single GPU)
  ├── mediaserver VM 192.168.0.16 — GPU removed, Jellyfin on CPU transcode
  └── gaming VM 192.168.0.79 (CachyOS)
        ├── Sunshine (native, KMS) + NVIDIA NVENC
        ├── KDE Plasma (autologin, Wayland)
        ├── Steam Big Picture (`steam -gamepadui`)
        ├── EmulationStation-DE + RetroArch
        └── HDMI dummy plug (1080p60)

Client (living-room mini PC, wired Ethernet)
  ├── Moonlight → gaming VM
  ├── Jellyfin Media Player → https://jellyfin.tlauffs.com
  └── Seerr (browser, formerly Jellyseerr) → https://jellyseerr.tlauffs.com
```

Jellyfin is not a Sunshine app; playing it natively avoids a second encode.

## Manual steps

These are one-time. Run them on the Proxmox host or in the VM, not via Ansible.

### Create the VM — done

VM 200, name `gaming`. `local-lvm` is the 1 TB NVMe; keep OS and games there.

| Setting | Value |
| --- | --- |
| Machine / BIOS | q35, OVMF (UEFI), no TPM |
| CPU / memory | host, 6 cores, 12 GB, ballooning off |
| Disks | 80 GB OS + 500 GB games on `local-lvm` |
| Network | VirtIO on `vmbr0`, `192.168.0.79/24`, gateway `192.168.0.1` |
| Display | std VGA for install, `none` after GPU passthrough |
| Other | QEMU agent, start at boot |

Download the ISO to `/var/lib/vz/template/iso` and verify its published
checksum:

```bash
cd /var/lib/vz/template/iso
wget https://iso.cachyos.org/desktop/260809/cachyos-desktop-linux-260809.iso
sha256sum cachyos-desktop-linux-260809.iso
```

Install CachyOS (KDE Plasma) with **no GPU passed through** and Secure Boot off.
Install to the 80 GB disk and leave the 500 GB game disk unformatted; the role
formats and mounts it.

### SSH — done

```bash
sudo systemctl enable --now sshd
sudo ufw allow 22/tcp
ssh-copy-id tim@192.168.0.79
```

Reserve the VM's MAC address to `.79` on the router, or configure the address
statically in CachyOS, so the inventory entry keeps working.

### GPU passthrough — done

vfio is already configured because the mediaserver had the card. Verify before
changing anything:

```bash
lspci -nnk | grep -iA3 -E 'vga|3d|nvidia'
qm config 108 | grep hostpci
```

1. Shut down the mediaserver and gaming VMs.
2. Remove the GPU from the mediaserver: `qm set 108 --delete hostpci0`
3. Add it to the gaming VM: `qm set 200 --hostpci0 0000:09:00,pcie=1,rombar=1`
   (web UI: **Primary GPU**, **ROM-Bar**, **PCI-Express**).
4. `qm set 200 --vga none`. VNC stops working after this; use SSH. The web UI
   console then reports `Failed to run vncproxy`, which is expected without an
   emulated display.
5. Start the VM and install the NVIDIA driver with CachyOS hardware detection:

   ```bash
   sudo chwd -a
   sudo reboot
   nvidia-smi
   ```

   If `chwd` picks the wrong profile, use `sudo chwd -i nvidia-dkms-580xx`;
   Turing is also supported by `nvidia-open-dkms`.
6. On the mediaserver, disable Jellyfin hardware transcoding: **Dashboard →
   Playback → Transcoding → Hardware acceleration: None**. The LinuxServer
   image has no environment variable for this, so it stays manual (stored in
   `/config/encoding.xml`). Without it, Jellyfin keeps trying NVENC and playback
   fails.

#### Console access

`vga none` means there is no emulated display, so the web UI console fails with
`qmp command 'set_password' failed` / `Failed to run vncproxy`. That is expected;
administer the VM over SSH.

For a one-off fix, shut down the VM,
temporarily `qm set 200 --vga std`, start it, and remove it again afterwards.

#### Local keyboard and mouse

`vga none` removes the console, but the physical USB keyboard and mouse still
belong to the Proxmox host. The monitor is on the passed-through GPU, so the
host cannot show you where the pointer is. Use one of these:

1. **Moonlight from a laptop or phone** (no hardware changes). Pair Moonlight
   with Sunshine, using the PIN from `https://192.168.0.79:47990`, and launch
   the **Desktop** app. The client's mouse and keyboard then control the VM.
   This also tests the same path the living-room client will use.
2. **USB passthrough** (physical mouse and keyboard at the VM monitor). On the
   Proxmox host, identify the device and attach it to VM 200:

   ```bash
   lsusb
   # Example: ID 046d:c52b Logitech Unifying Receiver
   qm set 200 -usb0 host=046d:c52b
   qm reboot 200
   ```

   Pass the USB receiver for a wireless keyboard/mouse combo; both devices then
   work in the VM. Use `-usb1` for a second device. If several identical
   receivers are present, use the bus-port form from `lsusb -t`, for example
   `host=1-2`. Remove it later with:

   ```bash
   qm set 200 --delete usb0
   qm reboot 200
   ```

   The host loses that keyboard/mouse while it is passed through, but SSH still
   works. If you need to share them, use a USB switch or pass through a
   dedicated USB controller.

### Sunshine web UI via NPM — done

NPM provides HTTPS for the web UI only; Moonlight still connects directly to the
VM. The proxy host is:

| Setting | Value |
| --- | --- |
| Domain | `sunshine.tlauffs.com` |
| Scheme | `https` |
| Forward host / port | `192.168.0.79` / `47990` |
| Websockets | enabled |
| SSL verification | off (Sunshine uses a self-signed certificate) |
| Path | none |

Point the DNS record at NPM and make sure the VM's firewall allows `47990`. The
VM role allows both the direct and NPM URLs for CSRF (see below).

### First run — in progress

The role formats and mounts the game disk at `/mnt/games`. The remaining steps
are account logins and physical setup:

- Plug an HDMI dummy plug into the passed-through GPU at 1080p60 for permanent
  headless capture. It is an EDID emulator, not a monitor: KMS capture needs a
  connected output even though the client is the only real display. Without it
  Sunshine finds no display and no encoder. A monitor connected to the GPU can
  stand in temporarily; keep it on the VM input while testing, because switching
  it back to the PC may remove the KMS output. The monitor must be on the VM
  input at boot, since Sunshine enumerates KMS at startup. This is the remaining
  hardware item.
- Open the Sunshine web UI (`https://192.168.0.79:47990` or
  `https://sunshine.tlauffs.com` through NPM), set credentials, and pair
  Moonlight once KMS sees the display. Native Sunshine uses KMS capture, so
  there is no portal prompt. Reboot after the role run so autologin starts it.
- Log into Steam, enable Big Picture, and add `/mnt/games/SteamLibrary` as a
  library folder.
- Run EmulationStation-DE once and point it at the ROM directory. Its Sunshine
  app entry is already managed by the role; after a reboot it appears in
  Moonlight as **EmulationStation**.
- After the ROM directories exist, open **Main Menu → Alternative Emulators**
  and select the installed emulator per system:
  - NES → **Nestopia UE**
  - PlayStation → **Beetle PSX HW** (the software **Beetle PSX** is also
    installed as a fallback)
  - PlayStation 2 → **PCSX2 (Standalone)**
  - GameCube and Wii → **Dolphin (Standalone)**
  - PSP → **PPSSPP (Standalone)**

  ES-DE defaults to the first command in its system list, which is often a
  RetroArch core that is not installed. Setting the alternative emulator once
  avoids launch failures.

## Ansible

The role follows the existing one-role-per-host pattern:

```text
ansible/roles/gaming/
  tasks/main.yml       # setup + config + sunshine
  tasks/setup.yml      # LizardByte repo, packages, disk format + mount, ES-DE
  tasks/config.yml     # Wayland autologin, ufw, guest agent, lock/sleep
  tasks/sunshine.yml   # capability, config, app entries, autostart
  templates/autologin.conf.j2
  templates/sunshine.desktop.j2
  templates/apps.json.j2
```

Run it with:

```bash
ansible-playbook -i ansible/inventory/hosts.ini ansible/playbook.yml \
  --ask-become-pass --ask-vault-pass -l gaming
```

`config.yml` detects whether CachyOS uses SDDM or Plasma Login Manager and writes
the autologin file to the matching config directory.

| Ansible | Manual |
| --- | --- |
| Packages, guest agent, autologin (Wayland) | Proxmox VM and GPU passthrough |
| ufw, disk format, fstab mount by UUID, ROM directory tree | CachyOS install |
| Sunshine native package, capability, config, app entries, autostart, ES-DE AppImage | Steam login, ES-DE first run, BIOS files, ROM content (rsync), Sunshine credentials/pairing, NPM proxy |

`setup.yml` formats `/dev/sdb` only if it has no filesystem yet, then mounts it
by UUID at `/mnt/games`. An existing filesystem with data is never overwritten.

It also creates `/mnt/games/ROMs` with one directory per supported system and
links `~/ROMs` to it. ES-DE takes the system from the directory name and ignores
game files anywhere else, so games have to land in the matching folder:

```text
nes  snes  n64  gb  gbc  gba  nds  megadrive  mastersystem  gamegear
sg-1000  psx  ps2  gc  wii  psp  dreamcast
```

Copy games over SSH, keeping the layout, and run it as `homelab_user` so the
files keep the right ownership:

```bash
rsync -av --dry-run ~/roms/ tim@192.168.0.79:/mnt/games/ROMs/
rsync -av ~/roms/ tim@192.168.0.79:/mnt/games/ROMs/
```

Then use **Utilities → Rescan ROM directory** in ES-DE, or restart it. Systems
with no game files stay hidden, so the menu grows as the library does.

Store disc games compressed rather than as 1:1 images: `chdman createcd` (PS1,
Dreamcast) or `createdvd` (PS2) to `.chd`, Dolphin's `.rvz` for GameCube and
Wii, `.cso` for PSP. Use `.m3u` playlists for multi-disc PS1 games so one game
is one entry. Prefer NTSC/USA per system where the choice exists, because PAL
titles ran at 50 Hz — the exception is disc games already owned in PAL, and
nearly every PAL GameCube title has a 60 Hz option in its boot prompt.

Packages from the official repos: `steam`, `retroarch` plus cores
(`libretro-snes9x`, `libretro-mupen64plus-next`, `libretro-mgba`,
`libretro-melonds`, `libretro-genesis-plus-gx`, `libretro-flycast`,
`libretro-nestopia`, `libretro-beetle-psx`, `libretro-beetle-psx-hw`),
`dolphin-emu`, `ppsspp`, `flatpak`, `qemu-guest-agent`, `fuse2`, and
`cifs-utils` only if ROMs stay on the NAS. The
role verifies `nvidia-smi` but does not manage DKMS.

Sunshine, EmulationStation-DE, and PCSX2 are not in the official Arch repos. Use
upstream packages instead of the AUR:

- Sunshine: the native package from LizardByte's pacman repo. The role adds the
  repo, installs `sunshine`, and re-applies `cap_sys_admin,cap_sys_nice` so KMS
  capture works on Wayland. The package's install script also sets up the
  `uinput`/`uhid` udev rules. Flatpak can't do KMS, and its portal capture needs
  an interactive consent dialog that a headless VM can't answer.
- EmulationStation-DE: the upstream Linux x86_64 AppImage from `es-de.org`,
  installed as `es-de`.
- PCSX2: the official Flathub Flatpak (`net.pcsx2.PCSX2`). ES-DE detects it
  automatically; no Sunshine app entry is needed.

ES-DE's bundled defaults do not always match what is installed here, and it picks
a system from the folder name before it ever looks at the file. Map the
mismatches once via **Main Menu → Alternative Emulators**:

| System | ES-DE default | Select instead |
| --- | --- | --- |
| gc, wii | Dolphin (libretro core) | Dolphin (Standalone) |
| nds | melonDS DS | melonDS |
| nes | Mesen | Nestopia UE |
| gb, gbc | Gambatte | mGBA |
| psp | PPSSPP (core) | PPSSPP (Standalone) |
| ps2 | PCEE2 | PCSX2 (Standalone) |

`snes`, `megadrive`, `mastersystem`, `gamegear`, `sg-1000`, `n64`, `gba`, `psx`
and `dreamcast` work with the installed cores as shipped, so they need no
mapping at all. For DS, set the melonDS core option screen layout to
top/bottom and turn on integer scaling in RetroArch; the touch layer is unused
by most of the library, including Pokémon.

PS1 and PS2 need BIOS files, and PSP firmware is optional. Put the PS1 BIOS in
`~/.config/retroarch/system`; PCSX2's Flatpak config and BIOS directory is
`~/.var/app/net.pcsx2.PCSX2/config/PCSX2`. Dreamcast and DS need no dump:
Flycast's HLE BIOS and melonDS's free BIOS are enough for game playback.
GameCube and Wii games run through Dolphin, and Wii games boot without a system
menu — only the menu itself needs one. BIOS, ROMs, and controller mappings stay
manual.

The role installs Sunshine and starts it at login. Sunshine uses KMS capture and
auto-detects NVENC on the passed-through card. Ports 47984, 47989, 47990,
48010/tcp and 47998–48000, 48002, 48010/udp are opened when ufw is active.

Only EmulationStation is exposed as a Sunshine app; standalone emulators are
launched by ES-DE. Add only the systems you actually use, otherwise the
controller/BIOS setup becomes a maintenance project.

The role also manages `apps.json` with Desktop, Steam Big Picture, and
EmulationStation. Low Res Desktop is omitted because its `xrandr` prep command
is X11-only. Sunshine reads the app list at startup, so restart Sunshine or
reboot after changing it.

The role allows both the direct LAN URL and the NPM HTTPS URL for CSRF:

```ini
csrf_allowed_origins = https://192.168.0.79:47990,https://sunshine.tlauffs.com
```

Update this line if either address changes. NPM proxies only the web UI; Moonlight
still connects directly to the VM.

Inventory:

```ini
[gaming]
192.168.0.79 ansible_user=tim ansible_become=true
```

Playbook: one play for `hosts: gaming` with `roles: [gaming]`.
Tags: `gaming_setup`, `gaming_config`, `gaming_sunshine`.

The mediaserver role no longer installs the NVIDIA driver/toolkit or requests
the `nvidia` runtime; Jellyfin runs on CPU transcode. The removed tasks are in
git history if you need to roll back.

## Client (living-room mini PC)

- Install **Moonlight** and pair it with the gaming VM. Pairing PIN comes from
  `https://192.168.0.79:47990`.
- Apps: Steam Big Picture, EmulationStation, Desktop.
- Install **Jellyfin Media Player** and point it at
  `https://jellyfin.tlauffs.com`. Play Jellyfin natively, not through Sunshine.
- Seerr (formerly Jellyseerr) is a request UI, not a player. Use
  `https://jellyseerr.tlauffs.com` in a browser; a phone is the most convenient
  place for requests. The NPM proxy host can be renamed to
  `seerr.tlauffs.com` later; it still targets port 5055. If it should be on the
  TV, add a browser shortcut but expect it to be less remote-friendly than
  Jellyfin.
- Put Moonlight, Jellyfin, and a Seerr browser shortcut in the app grid or
  desktop. Decide the boot target:
  - Gaming-first: autostart Moonlight fullscreen, then exit it for Jellyfin.
  - Mixed use: boot to the desktop/app grid instead, so Moonlight and Jellyfin
    are both one click away.
  Jellyfin is not a Moonlight/Sunshine app, so it will never appear in
  Moonlight's app list.
- Use wired Ethernet; Wi-Fi is the usual cause of stutter and input lag.
- Bazzite is the simplest fresh client: Jellyfin Media Player and a browser are
  available as Flatpaks. CachyOS is fine if it is already installed; ChimeraOS
  is more appliance-like but less flexible for a mixed Moonlight/Jellyfin box.

## Rollback

If the gaming VM does not work out:

1. Shut down the gaming VM.
2. `qm set 200 --delete hostpci0`
3. `qm set 108 --hostpci0 0000:09:00`
4. Restore the NVIDIA tasks and Jellyfin `runtime: nvidia` in the mediaserver
   role, or re-run the previous commit.
5. Start the mediaserver and verify `nvidia-smi` and Jellyfin hardware
   transcoding.

## Acceptance checklist

- [x] `nvidia-smi` works in the gaming VM
- [x] Sunshine UI reachable directly and through NPM
- [x] KMS output detected (monitor or dummy plug)
- [ ] HDMI dummy plug installed for permanent headless capture
- [ ] EmulationStation appears in the Sunshine/Moonlight app list
- [ ] Moonlight pairs successfully
- [ ] Steam Big Picture streams at 1080p60
- [ ] Audio plays in the stream
- [ ] Controller input works
- [ ] EmulationStation launches and finds ROMs
- [ ] One game each launches for NES, PS1, PS2, GameCube/Wii, and PSP
- [ ] Jellyfin plays natively on the client
- [ ] Jellyfin on the mediaserver transcodes 1080p on CPU
- [ ] Gaming VM starts automatically after a host reboot

## Open items

- Install the HDMI dummy plug for permanent headless capture. A monitor can stand
  in temporarily, but switching it back to the PC may remove the KMS output.
- Confirm `192.168.0.79` is reserved or statically configured.
- Confirm 500 GB is enough for the game library.
- Decide whether ROMs are local or on the NAS.
- Add PS1 and PS2 BIOS files, and optionally PSP firmware.
- Confirm the client machine, that it is wired, and its OS.
- Decide whether to add RAM before raising the gaming VM above 12 GB.
