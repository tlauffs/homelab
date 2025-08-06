# Setup Proxmox

## Change Repository
- Repositories: change Proxmox repository from enterprise to community/no subscription
- Update: Refresh and Upgrade (apt update & apt upgrade)

## Enable IOMMU
- `vim /etc/default/grub`
- to `GRUB_CMDLINE_LINUX_DEFAULT="quiet"` add: intel_iommu=on or amd_iommu=on.
- `GRUB_CMDLINE_LINUX_DEFAULT="quiet amd_iommu=on"`
- `update-grub`
- `update-initramfs -u`
- reboot
- test if IOMMU is enabled
```
dmesg | grep -e DMAR -e IOMMU
dmesg | grep 'remapping'
dmesg | grep -e IOMMU -e AMD-Vi
```

## Create Storage
- disks > ZFS > create: ZFS
- pick drives and raid level
