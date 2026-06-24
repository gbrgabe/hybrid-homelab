# OPNsense Firewall — Homelab Setup

## Overview

This document describes the installation and configuration of OPNsense as the firewall/gateway for my hybrid homelab running on Proxmox VE. The idea is for OPNsense to act as the network edge, isolating my internal lab network (`office.lab`) from my home network — basically creating a "bubble" where all my VMs live.

---

## Environment

| Component | Details |
|---|---|
| Hypervisor | Proxmox VE 9.1.11 |
| OPNsense Version | 26.1.6 (amd64) |
| Node | office |
| VM ID | 100 |

---

## VM Specifications

| Resource | Value |
|---|---|
| CPU | 2 cores (x86-64-v2-AES) |
| RAM | 2 GB |
| Disk | 20 GB (VirtIO SCSI) |
| BIOS | OVMF (UEFI) |
| NIC 0 (WAN) | VirtIO — vmbr0 |
| NIC 1 (LAN) | VirtIO — vmbr1 |

> **Note on RAM:** The official OPNsense docs say the minimum is 3 GB, and the installer even throws a warning about it. But since this is a homelab running basic firewall/NAT only — no IDS/IPS, no proxy — 2 GB is enough. I checked the community and confirmed this before proceeding.

---

## Network Architecture

```
[ Home Router / ISP ]
        |
    (physical NIC)
        |
      vmbr0  ←—— Proxmox management + OPNsense WAN
        |
   OPNsense VM
        |
      vmbr1  ←—— OPNsense LAN (internal lab network)
        |
   [ DC, SRV-MON, future VMs ]
```

| Interface | Bridge | Role | IP |
|---|---|---|---|
| vtnet0 | vmbr1 | LAN | 192.168.30.1/24 |
| vtnet1 | vmbr0 | WAN | 192.168.0.119/24 (DHCP) |

---

## Installation Steps

### 1. Proxmox Network Setup

Before even creating the VM, I needed a second Linux Bridge for the LAN side. By default Proxmox only has `vmbr0` which is tied to the physical NIC. So I went to **System → Network → Create → Linux Bridge** and created `vmbr1` with no IP and no physical port — it's purely internal. Then hit **Apply Configuration**.

### 2. VM Creation

Created the VM through the Proxmox GUI with the OPNsense 26.1.6 ISO attached. One thing to pay attention to: OPNsense needs **two NICs** — one for WAN and one for LAN. I added the first NIC (`vmbr0`) during VM creation, then added the second (`vmbr1`) in Hardware before booting for the first time.

> Important: do NOT start the VM before adding the second NIC, otherwise you'll have to assign interfaces manually from the console anyway.

### 3. OPNsense Installation

Booted from the ISO, logged in with `installer` / `opnsense` and went through the FreeBSD installer. I selected **UFS** as the filesystem — the installer warned me about the low RAM but I just hit "Proceed anyway" since I knew what I was doing. Set the root password and let it install.

Before rebooting, I removed the ISO from the CD/DVD drive in Proxmox Hardware so it wouldn't try to boot from the installer again.

### 4. Interface Assignment and Troubleshooting — WAN Not Getting an IP

After reboot, logged in as `root`. The console was already showing:

```
LAN (vtnet0) → 192.168.30.1/24
WAN (vtnet1) → (blank)
```

WAN had no IP. I went to **Option 1) Assign interfaces** to double check:

- No LAGGs
- No VLANs
- WAN → `vtnet1`
- LAN → `vtnet0`

Looked correct. Then tried **Option 2) Set interface IP address** → WAN → DHCP. Still nothing. Rebooted. Still blank.

After some investigation I found the actual root cause: the problem wasn't the interface assignment in OPNsense — it was how the NICs were mapped in Proxmox.

When I created the VM, I set `net0 → vmbr0` (physical NIC) and `net1 → vmbr1` (internal bridge). That meant:

- `vtnet0` = vmbr0 (physical, internet)
- `vtnet1` = vmbr1 (internal, no upstream)

So when I assigned WAN to `vtnet1` in the console, WAN was actually sitting on `vmbr1` — the isolated internal bridge with no connection to anything. Obviously DHCP wasn't going to work from there.

The fix was to go into the VM **Hardware** in Proxmox and swap the bridges:

- **net0** → `vmbr1` (LAN — internal bridge)
- **net1** → `vmbr0` (WAN — physical NIC with internet)

After that, rebooted and WAN immediately picked up `192.168.0.118/24` via DHCP.

### 6. Accessing the WebGUI — Second Problem

Tried to access `https://192.168.0.118` from my PC and it just kept loading. Turns out OPNsense blocks WebGUI access from the WAN interface by default, which makes total sense security-wise but caught me off guard.

The fix was to go back to the console, select **Option 8) Shell** and run:

```bash
pfctl -d
```

This temporarily disables the firewall, which let me access the WebGUI. Not a permanent solution — I still need to add a proper firewall rule for this — but it worked for the initial setup.

### 7. Initial Configuration Wizard

Went through the initial configuration wizard:

- **Hostname:** OPNsense
- **Domain:** office.lab
- **Language:** English
- **Timezone:** America/Sao_Paulo
- **DNS:** left blank while DC is not online 
- **LAN:** 192.168.30.1/24
- **DHCP Server:** Enabled on LAN
- **Block RFC1918:** Enabled
- **Block Bogon Networks:** Enabled

### 8. Firmware Update

Updated OPNsense via **System → Firmware → Updates**. (had to run couples pfctl -d)

---

## Notes

- **WebGUI WAN access:** Every time the firewall reloads, access from the WAN side gets blocked again. I need to add a permanent rule under **Firewall → Rules → WAN** allowing TCP 443 from my trusted IP. *Haven't done this yet*.
- **DNS:** Currently blank waiting on DC01
- **pfctl -d:** Useful for getting access back during setup, but it fully disables the firewall. Always reload services or reboot after configuring.

---

## Next Steps

- [ ] Provision DC (Windows Server Core) VM on Proxmox
- [ ] Configure AD DS, DNS and DHCP on the DC
- [ ] Update OPNsense DNS to point to the DC
- [ ] Configure static DHCP leases for lab VMs
- [ ] Document full network topology in Cisco Packet Tracer

---

## References

- [OPNsense Official Documentation](https://docs.opnsense.org)
- [OPNsense Hardware Requirements](https://docs.opnsense.org/manual/hardware.html)
- [Proxmox VE Documentation](https://pve.proxmox.com/wiki/Main_Page)
