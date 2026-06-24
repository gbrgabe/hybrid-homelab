# Proxmox VE 9.1.11 — Initial Setup & Configuration

## Overview
Initial configuration of Proxmox VE 9.1.11 on the lab server (PC1)

**Hardware:** Samsung laptop · Intel i5-7200U · 16GB RAM · ethernet connection

---

## 1. Repository Configuration

### Problem
Running `apt-get update` returned `401 Unauthorized` errors from the enterprise repositories:
- `https://enterprise.proxmox.com/debian/pve`
- `https://enterprise.proxmox.com/debian/ceph-squid`

### Steps

Located the enterprise repository files:
```bash
grep -r "enterprise.proxmox.com" /etc/apt/sources.list.d/
```

Output:
```
/etc/apt/sources.list.d/pve-enterprise.sources
/etc/apt/sources.list.d/ceph.sources
```

Disabled both by appending `Enabled: no`:
```bash
echo "Enabled: no" >> /etc/apt/sources.list.d/pve-enterprise.sources
echo "Enabled: no" >> /etc/apt/sources.list.d/ceph.sources
```

---

## 2. System Upgrade

```bash
apt-get update && apt-get upgrade -y
```

**Result:** 137 packages upgraded, including:
- pve-manager 9.1.11
- qemu-server 9.1.10
- Kernel 6.17.2-1-pve
- OpenSSL, Ceph, Corosync, LXC

Reboot to apply the new kernel:
```bash
reboot
```



