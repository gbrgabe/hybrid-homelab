# DC01 — Domain Controller

Windows Server 2022 Core — AD DS, DNS, DHCP — domain `inthelli.lab`

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
New-NetIPAddress -InterfaceAlias "LAN" -IPAddress 10.0.0.10 -PrefixLength 24 

Set-DnsClientServerAddress -InterfaceAlias "LAN" -ServerAddresses 10.0.0.10
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
Install-ADDSForest -DomainName "inthelli.lab" -InstallDns
```

Validate after reboot:

```powershell
Get-ADDomain
Get-Service ntds
nslookup inthelli.lab
nslookup 10.0.0.10
```

---

## 6. Default Gateway

During the initial deployment, the Domain Controller had temporary Internet access through a secondary network interface connected directly to the home network.

After the initial configuration was completed, the temporary network interface was removed and the default route was configured to use the OPNsense LAN interface (`10.0.0.1`).

Configure the default route to use OPNsense:

```powershell
New-NetRoute -DestinationPrefix "0.0.0.0/0" -InterfaceAlias "LAN" -NextHop 10.0.0.1
```

Verify:

```powershell
route print
ipconfig /all
```
---

## 7. Storage Migration — SATA to VirtIO SCSI

With AD DS operational, the boot disk was migrated from SATA to VirtIO SCSI.

### 7.1 Install VirtIO SCSI Driver

With `virtio-win.iso` still attached:

```powershell
pnputil /add-driver E:\vioscsi\2k22\amd64\vioscsi.inf /install
```

### 7.2 Add a Temporary VirtIO SCSI Disk

Shut down the VM and add a new empty disk using the VirtIO SCSI controller. Keep the original SATA disk attached. Boot the VM — Windows will detect and load the VirtIO SCSI controller automatically.

### 7.3 Replace the SATA Disk

Shut down the VM, remove the SATA disk, and reattach it as a VirtIO SCSI device. Start the VM.

Verify:

```powershell
Get-Disk
```
---

## 8. DHCP Server

Install the DHCP Server role:

```powershell
Install-WindowsFeature DHCP -IncludeManagementTools
```

### Create the DHCP Scope

```powershell
Add-DhcpServerv4Scope -Name "inthelli.lab" -StartRange 10.0.0.150 -EndRange 10.0.0.200 -SubnetMask 255.255.255.0
```

### Configure Scope Options

```powershell
Set-DhcpServerv4OptionValue -ScopeId 10.0.0.10 -Router 10.0.0.1 -DnsServer 10.0.0.10 -DnsDomain "inthelli.lab"
```

Verify:

```powershell
Get-DhcpServerv4OptionValue -ScopeId 10.0.0.10
```

### Authorize DHCP in Active Directory

```powershell
Add-DhcpServerInDC -DnsName "DC01.inthelli.lab" -IPAddress 10.0.0.10
```

Verify:

```powershell
Get-DhcpServerInDC
```

### Configure DHCP Reservations

```powershell
Add-DhcpServerv4Reservation -ScopeId 10.0.0.10 -IPAddress 10.0.0.101 -ClientId "00-0C-29-5F-C6-0C" -Name "PC01"

Add-DhcpServerv4Reservation -ScopeId 10.0.0.10 -IPAddress 10.0.0.102 -ClientId "00-0C-29-CE-26-93" -Name "PC02"

Add-DhcpServerv4Reservation -ScopeId 10.0.0.10 -IPAddress 10.0.0.103 -ClientId "00-0C-29-73-01-F6" -Name "PC03"

Add-DhcpServerv4Reservation -ScopeId 10.0.0.10 -IPAddress 10.0.0.104 -ClientId "00-0C-29-17-51-99" -Name "PC04"
```

Verify:

```powershell
Get-DhcpServerv4Reservation -ScopeId 10.0.0.10
```

## Current Status

- [x] Active Directory
- [x] DNS
- [x] Domain Controller
- [x] Static IP
- [x] Default Gateway (OPNsense)
- [x] VirtIO Network
- [x] VirtIO SCSI
- [x] DHCP Server
- [x] DHCP Scope
- [x] DHCP Reservations
- [x] DHCP Authorized in Active Directory
- [x] OPNsense DHCP Disabled
- [ ] Domain Clients (waiting on Microsoft Cloud infrastructure)
- [ ] DHCP Validation (waiting on Microsoft Cloud infrastructure)


