# Windows AD Home Lab — Build Report

A home lab built on **Parrot OS** using **VirtualBox**, consisting of a **Windows Server (Core)** promoted to an **Active Directory Domain Controller**, and a **Windows 8** client joined to that domain. Built for hands-on practice with Active Directory administration and general sysadmin skills in an isolated environment.

---

## Environment Summary

| Component | Details |
|---|---|
| Host machine | Parrot OS, Core i7 CPU, 8GB RAM |
| Hypervisor | Oracle VirtualBox |
| VM 1 | Windows Server (Server Core) — hostname `DC01` — Active Directory Domain Controller |
| VM 2 | Windows 8 — domain-joined client |
| Domain name | `lab.local` |
| Networking | VirtualBox Host-only Adapter (`vboxnet0`) for VM-to-VM traffic, plus NAT adapter for internet access |

---

## 1. Planning and Resource Allocation

With only 8GB of RAM on the host, resources were split to keep Parrot, the Server, and Windows 8 all usable at once:

| System | RAM Allocated | Notes |
|---|---|---|
| Parrot OS (host) | ~2.5–3 GB | Left free for host OS + tools |
| Windows Server (Core) | 2 GB | Server Core chosen (no GUI) to fit in 2GB |
| Windows 8 | 1.5–2 GB | Domain-joined client |

**Decision:** Active Directory / sysadmin-focused lab — Windows Server as a Domain Controller, Windows 8 as a domain-joined client, both hosted on Parrot OS via VirtualBox. Server Core (no GUI) was chosen over Desktop Experience to conserve RAM, and because Core is closer to how real-world production servers are managed.

---

## 2. Windows 8 VM Installation

Created a new VM in VirtualBox and installed Windows 8 from ISO. Verified the VM booted correctly and setup completed.

![Windows 8 installation / desktop after first boot](screenshots/01-windows8-install.jpg)

---

## 3. Windows Server Installation and Promotion to Domain Controller

### 3.1 Install Windows Server (Server Core)

Created a second VM (2GB RAM, 2 CPU cores, 50GB disk) and installed Windows Server, selecting the **Server Core** (no GUI) option to conserve RAM.

Hit a partition selection error (`There is an error selecting this partition for install`) — resolved by deleting the existing partition and letting Setup automatically recreate it on the unallocated space.

![Partition selection screen / error resolution](screenshots/02-partition-error.jpg)

### 3.2 Initial configuration via SConfig

After install, the server boots into **SConfig** (Server Configuration Tool) — the text-based menu used to manage Server Core.

Renamed the computer to `DC01` via SConfig option `2` (Computer name) and restarted.

![SConfig main menu after fresh install](screenshots/03-sconfig-menu.jpg)

### 3.3 Configure networking in VirtualBox

- Found Adapter 1 was initially set to **Bridged Adapter** (placing the VM on the physical Wi-Fi network) — not suitable for an isolated lab.
- Created a **Host-only Network** (`vboxnet0`) via VirtualBox's Host Network Manager, since it didn't exist by default.
- Reconfigured the Server VM's adapters:
  - **Adapter 1** → Host-only Adapter (`vboxnet0`) — isolated lab traffic
  - **Adapter 2** → NAT — internet access (Windows Update, etc.)

![VirtualBox VM Settings — Network — Adapter 1 configuration](screenshots/04-network-settings.jpg)

### 3.4 Set a static IP on the Server

Via SConfig → option `8` (Network Settings) → option `1` (Set network adapter address):

```
IP address:      192.168.56.10
Subnet mask:      255.255.255.0
Default gateway:  192.168.56.1   (placeholder — host-only networks have no real router)
```

![SConfig Network adapter settings showing static IP applied](screenshots/05-static-ip.jpg)

### 3.5 Install AD DS and promote to a Domain Controller

From SConfig option `15` (Exit to command line):

```powershell
Install-WindowsFeature AD-Domain-Services -IncludeManagementTools
Install-ADDSForest -DomainName "lab.local"
```

Set the DSRM (Directory Services Restore Mode) password when prompted. Warnings about DNS delegation and the NAT adapter lacking a static IP are expected in an isolated lab domain — no action needed. The server rebooted automatically to complete promotion.

![PowerShell output — Installing new forest / DNS configuration in progress](screenshots/06-adds-install.jpg)

### 3.6 Verify the Domain Controller and configure DNS

```powershell
Get-ADDomain
```

Confirmed `DC01.lab.local` as the domain controller with the expected AD containers (Users, System, DNS zones).

![Get-ADDomain output showing DC01.lab.local details](screenshots/07-get-addomain.jpg)

Set DNS on the host-only adapter to point to itself, since the DC now runs its own DNS service:

```powershell
Set-DnsClientServerAddress -InterfaceIndex 7 -ServerAddresses 127.0.0.1
Get-DnsClientServerAddress -InterfaceIndex 7
```

![Get-DnsClientServerAddress output confirming 127.0.0.1](screenshots/08-dns-loopback.jpg)

### 3.7 Create a test Active Directory user

```powershell
New-ADUser -Name "Test User" -SamAccountName "testuser" `
  -UserPrincipalName "testuser@lab.local" `
  -AccountPassword (ConvertTo-SecureString "REDACTED" -AsPlainText -Force) `
  -Enabled $true

Get-ADUser -Filter *
```

![Get-ADUser output showing testuser and Administrator accounts](screenshots/09-get-aduser.jpg)

---

## 4. Joining Windows 8 to the Domain

### 4.1 Configure Windows 8 networking

Set the Windows 8 VM's adapter (in VirtualBox) to the same Host-only Adapter (`vboxnet0`) as the Server. On Windows 8, manually configured TCP/IPv4:

```
IP address:         192.168.56.20
Subnet mask:         255.255.255.0
Preferred DNS server: 192.168.56.10   (points to the Domain Controller)
```

![Windows 8 TCP/IPv4 Properties — static IP and DNS settings](screenshots/10-win8-tcpip.jpg)

### 4.2 Verify connectivity to the Domain Controller

```
ping 192.168.56.10
nslookup lab.local
```

Both succeeded — `nslookup` correctly resolved `lab.local` to `192.168.56.10`, confirming Windows 8 could reach and resolve records through the DC.

![Command Prompt — successful ping and nslookup](screenshots/11-ping-nslookup.jpg)

### 4.3 Join the domain

Advanced System Settings → Computer Name → Change → Domain → `lab.local`, authenticated with domain Administrator credentials, then restarted to complete the join.

![Domain join confirmation — Welcome to the lab.local domain](screenshots/12-domain-join.jpg)

---

## 5. Snapshots

Took a VirtualBox snapshot of the Server VM immediately after successful DC promotion, as a clean rollback point.

![VirtualBox snapshot list showing Snapshot 1](screenshots/13-snapshot.jpg)

---

---

## 6. Next Steps (Phase 1 — Completed)

- [x] Log into Windows 8 with the domain `testuser` account to confirm authentication against the DC
- [x] Install RSAT (Remote Server Administration Tools) on Windows 8 for GUI-based AD management
- [x] Explore Group Policy Objects (GPOs) for centralized client configuration
- [x] Begin Parrot OS recon/testing (e.g. `nmap`) against the lab network once both VMs are confirmed stable

See sections 7–12 below for the full write-up of each item.

---

## 7. Domain Login Verification

Logged into the Windows 8.1 client using the domain account `LAB\testuser`, confirming the full authentication chain — DNS resolution, Kerberos ticket issuance, and domain trust — was working end-to-end.

![Windows 8.1 successful domain logon as testuser](TestUser.png)

---

## 8. RSAT Installation (GUI-Based AD Management)

### 8.1 Install RSAT

Downloaded and installed the official Microsoft RSAT package for Windows 8.1 (x64) to enable GUI-based management tools — **Active Directory Users and Computers**, **Group Policy Management Console** — without needing a desktop environment on the Server Core DC itself.

During installation, hit a UAC elevation issue while logged in as the standard domain account (`testuser`), since it lacked local admin rights on the client. Resolved by temporarily adding `testuser` to Domain Admins from the DC:

```powershell
Add-ADGroupMember -Identity "Domain Admins" -Members testuser
```

![RSAT installation progress/completion](screenshots/15-rsat-install.jpg)

### 8.2 Verify RSAT works

Opened **Active Directory Users and Computers** (`dsa.msc`) from Windows 8.1 and successfully performed a password reset on `testuser` through the GUI, confirming RSAT was correctly talking to `DC01`.

![Active Directory Users and Computers — password reset via GUI](screenshots/16-aduc-password-reset.jpg)

---

## 9. Group Policy Deployment

### 9.1 Create and link a test GPO

Using `gpmc.msc` on Windows 8.1, created a new GPO named `Test-Wallpaper-Policy`, linked to the `lab.local` domain.

Configured it under `User Configuration → Policies → Administrative Templates → Desktop → Desktop → Desktop Wallpaper`, pointing to a confirmed valid wallpaper path (checked in advance to avoid a bad-path issue):

```
dir C:\Windows\Web\Wallpaper /s /b
```

![Group Policy Editor — Desktop Wallpaper setting configured](screenshots/17-gpo-wallpaper-editor.jpg)

### 9.2 Apply and verify

```
gpupdate /force
gpresult /r
```

Output confirmed `Test-Wallpaper-Policy` under **Applied Group Policy Objects**, sourced from `DC01.lab.local`. After a full logoff/logon cycle, the desktop wallpaper updated automatically — confirming successful end-to-end GPO deployment from the DC to the client.

![gpresult /r output showing the GPO applied](screenshots/18-gpresult-applied.jpg)
![Windows 8.1 desktop with the new wallpaper applied](screenshots/19-wallpaper-applied.jpg)

---

## 10. Network Reconnaissance from Parrot OS

With the domain fully operational, Parrot OS was used in its intended role as the lab's recon/attacker workstation, scanning the lab's own host-only network (`192.168.56.0/24`).

### 10.1 Host discovery

```bash
sudo nmap -sn 192.168.56.0/24
```

| IP Address | Host | Notes |
|---|---|---|
| 192.168.56.1 | VirtualBox host-only adapter | Represents the Parrot host on this network |
| 192.168.56.10 | `DC01` (Windows Server) | Domain Controller |
| 192.168.56.20 | `OPKI` (Windows 8.1) | Domain-joined client |
| 192.168.56.100 | Unidentified, no open ports | Investigated — host-only DHCP confirmed **Disabled**; host responds to ping but exposes no services. Not a security concern, likely a reserved/phantom address on the virtual network. |

![nmap -sn host discovery results](screenshots/20-nmap-discovery.jpg)

### 10.2 Full port + service scan — Domain Controller

```bash
sudo nmap -sV -p- 192.168.56.10
```

A textbook AD service footprint — confirmed the DC's role purely from exposed services:

| Port | Service | Purpose |
|---|---|---|
| 53 | DNS | Name resolution for `lab.local` |
| 88 | Kerberos | Domain authentication |
| 135 / 49664+ | RPC / dynamic RPC | Internal Windows service communication |
| 139 / 445 | NetBIOS / SMB | File sharing, core AD communication |
| 389 | LDAP (unencrypted) | Directory queries |
| 636 | LDAPS (encrypted) | Secure directory queries |
| 464 | kpasswd | Kerberos password changes |
| 3268 / 3269 | Global Catalog | Forest-wide directory search |
| 3389 | RDP | Remote administration (enabled earlier in this lab) |
| 5985 | WinRM | PowerShell remoting |

![nmap full scan results against DC01](screenshots/21-nmap-dc-fullscan.jpg)

### 10.3 Full port + service scan — Windows 8.1 client

```bash
sudo nmap -sV -p- 192.168.56.20
```

A dramatically smaller footprint than the DC — only **3 open ports** (135/RPC, 445/SMB, one dynamic RPC port). Confirms the expected principle that domain clients expose minimal services and lean on the DC for authentication, directory, and policy — which is exactly why servers are the higher-value target in a real environment.

![nmap full scan results against Windows 8.1](screenshots/22-nmap-win8-fullscan.jpg)

---

## 11. Findings and Observations

- **Finding 1 — Unencrypted LDAP exposed alongside LDAPS.** Port 389 (plain LDAP) is open on the DC alongside port 636 (LDAPS). In production, this is a common hardening finding — directory queries and bind credentials over 389 aren't encrypted and could be intercepted. Recommended remediation: enforce LDAP signing/channel binding, or restrict/disable plaintext LDAP where clients support LDAPS.
- **Finding 2 — Server vs. client attack surface.** The DC exposes ~20 services (DNS, Kerberos, LDAP, SMB, RPC, RDP, WinRM, etc.), while the client exposes only 3. Illustrates why DCs are the highest-priority hardening/monitoring target — compromising the DC compromises the whole domain.
- **Finding 3 — RDP exposed on the DC.** Port 3389 is open, consistent with RDP being enabled earlier for GUI-assisted admin. In production, RDP exposure on Domain Controllers is a common ransomware entry point and is typically restricted to jump hosts or VPN-gated access rather than exposed broadly.

---

## 12. Updated Next Steps

- [ ] Harden LDAP to require signing/encryption, then re-scan to confirm port 389 is no longer usable in plaintext
- [ ] Create Organizational Units (OUs) and apply differentiated GPOs per OU
- [ ] Set up centralized logging/auditing on the DC and correlate against Parrot-side scan activity
- [ ] Explore deeper vulnerability scanning (Nmap NSE scripts or a dedicated scanner) against the lab network

---

## Where to Add Screenshots

All screenshots go in the `screenshots/` folder at the repo root, named to match the references above:

| Filename | Description |
|---|---|
| `14-domain-login-testuser.jpg` | Windows 8.1 successful domain logon |
| `15-rsat-install.jpg` | RSAT installation progress/completion |
| `16-aduc-password-reset.jpg` | AD Users and Computers GUI password reset |
| `17-gpo-wallpaper-editor.jpg` | Group Policy Editor, Desktop Wallpaper setting |
| `18-gpresult-applied.jpg` | `gpresult /r` showing the GPO applied |
| `19-wallpaper-applied.jpg` | Windows 8.1 desktop with new wallpaper |
| `20-nmap-discovery.jpg` | `nmap -sn` host discovery results |
| `21-nmap-dc-fullscan.jpg` | `nmap -sV -p-` results against the DC |
| `22-nmap-win8-fullscan.jpg` | `nmap -sV -p-` results against Windows 8.1 |

Just drop the images into `screenshots/` with these exact filenames and they'll render automatically in this README on GitHub — no other changes needed.

---

## Notes

- All passwords shown in this repo are placeholders/examples only — do not reuse real credentials.
- This lab runs entirely on an isolated host-only virtual network, not exposed to the internet or the home LAN.
