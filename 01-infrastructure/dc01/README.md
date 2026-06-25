# DC01 — Domain Controller

Windows Server 2022 Core — AD DS, DNS, DHCP — domain `office.lab`

---

## VM Specs

| Parameter | Value               |
| --------- | ------------------- |
| VM ID     | 101                 |
| Memory    | 3 GB                |
| Disk      | 60 GB — VirtIO SCSI |
| Network   | vmbr1 — VirtIO      |

---

## 1. VM Creation

> **Note:** First attempt used VirtIO SCSI disk — Windows installer couldn't detect it because the storage driver wasn't loaded yet. VM was deleted and recreated using SATA disk to complete the installation.

VM created with:
- Hard Disk: `sata0`
- Network: `vmbr1`

---

## 2. Initial Configuration (SConfig)

After first boot:

- Set computer name → `DC01`
- Enabled Remote Management
- Enabled server response to ping
- Enabled Remote Desktop (Network Level Authentication)

---

## 3. VirtIO Network Driver

`Get-NetAdapter` returned empty — no network adapter visible.

Shut down the VM, attached `virtio-win.iso` as CD drive, then booted:

```powershell
Get-Volume
# locate the VirtIO ISO drive letter (e.g. E:)

cd E:\NetKVM\2k22\amd64

pnputil /add-driver E:\NetKVM\2k22\amd64\netkvm.inf /install
```

Verify adapter is now visible:

```powershell
Get-NetAdapter
```

---

## 4. Static IP and DNS

```powershell
New-NetIPAddress -InterfaceAlias "Ethernet" -IPAddress 10.0.0.10 -PrefixLength 24

Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ServerAddresses 10.0.0.10
```

> If DNS doesn't persist via PowerShell, set manually: SConfig → Networking → DNS → `10.0.0.10`

Verify:

```powershell
ipconfig /all
```

---

## 5. Active Directory

Install AD DS:

```powershell
Install-WindowsFeature AD-Domain-Services -IncludeManagementTools
```

Promote to Domain Controller (new forest):

```powershell
Install-ADDSForest -DomainName "office.lab" -InstallDns
```

Validate after reboot:

```powershell
Get-ADDomain
Get-Service ntds
nslookup office.lab
nslookup 10.0.0.10
```

---

## 6. Storage Migration — SATA to VirtIO SCSI

With AD DS operational, the boot disk was migrated from SATA to VirtIO SCSI.

### 6.1 Install VirtIO SCSI Driver

With `virtio-win.iso` still attached:

```powershell
pnputil /add-driver E:\vioscsi\2k22\amd64\vioscsi.inf /install
```

### 6.2 Add a Temporary VirtIO SCSI Disk

Shut down the VM and add a new empty disk using the VirtIO SCSI controller. Keep the original SATA disk attached. Boot the VM — Windows will detect and load the VirtIO SCSI controller automatically.

### 6.3 Replace the SATA Disk

Shut down the VM, remove the SATA disk, and reattach it as a VirtIO SCSI device. Start the VM.

Verify:

```powershell
Get-Disk
```

Current Status

✔ Active Directory
✔ DNS
✔ Domain Controller
✔ Static IP
✔ VirtIO Network
✔ VirtIO SCSI
□ DHCP (coming next)
□ Domain Clients


