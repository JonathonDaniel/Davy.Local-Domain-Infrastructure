# Davy.Local — Active Directory Homelab Build Log

**Author:** Jonathon Alexander Daniel  
**Domain:** `davy.local`  
**Hypervisor:** Proxmox VE 9.1.1 (bare metal)  
**Status:** Tier 1 Complete

> This document is a full build log of the davy.local Active Directory homelab. It includes every attempt, every failure, root cause analysis, architectural decisions, and final verification. Nothing was removed — the failures are as important as the successes.

---

## Table of Contents

1. [Lab Overview](#lab-overview)
2. [Act 1 — Proxmox on VirtualBox](#act-1--proxmox-on-virtualbox)
3. [Act 2 — Migration to Bare Metal](#act-2--migration-to-bare-metal)
4. [Network Architecture](#network-architecture)
5. [VM Specifications](#vm-specifications)
6. [pfSense Build & Configuration](#pfsense-build--configuration)
7. [DC-01 — Primary Domain Controller](#dc-01--primary-domain-controller)
8. [DC-02 — Read-Only Domain Controller](#dc-02--read-only-domain-controller)
9. [WIN-CLIENT — Domain Workstation](#win-client--domain-workstation)
10. [Verification & Health Checks](#verification--health-checks)
11. [Key Lessons Learned](#key-lessons-learned)
12. [Roadmap](#roadmap)

---

## Lab Overview

The goal of this project was to build a fully functional Active Directory domain from scratch, simulating an enterprise network environment. The lab runs on an isolated subnet completely separated from the home network, with pfSense acting as the sole firewall and gateway.

**Final architecture summary:**

```
Internet
    │
Home router (192.168.1.1)
    │
pfSense WAN (192.168.1.26)
pfSense LAN (10.10.10.1)
    │
    ├── DC-01        10.10.10.10   Primary DC · DNS · DHCP
    ├── DC-02        10.10.10.11   RODC replica
    └── WIN-CLIENT   10.10.10.100  Domain-joined workstation
```

**Domain:** `davy.local`  
**Lab subnet:** `10.10.10.0/24`  
**Home network:** `192.168.1.0/24` (completely separated)

---

## Act 1 — Proxmox on VirtualBox

### Context

The original plan was to run Proxmox VE inside Oracle VirtualBox on a Windows host machine. Proxmox is a Type 1 hypervisor and is not designed to run on Windows natively, so VirtualBox was used to virtualize the Proxmox ISO. Nested virtualization (SVM/VT-x) was required to allow VMs to run inside the virtualized Proxmox instance.

---

### Attempt 1 — 10:30

**Goal:** Install Proxmox VE ISO inside Oracle VirtualBox

**Issue:** VM hit a black screen after "Installing graphical interface" stage and never recovered

**Root cause:** VM OS type was not configured in VirtualBox before installation. Without the correct OS type set, VirtualBox could not negotiate display and hardware communication with the Proxmox installer, causing it to hang indefinitely.

**Status:** ❌ Failed

---

### Attempt 2 — 10:45

**Goal:** Retry with correct OS type selected

**Changes made:**
- Set VirtualBox VM OS type to `Linux → Debian 64-bit`

**Result:** Installation completed. However, the Proxmox web UI was unreachable via browser after install.

**Root cause:** Network adapter was left on NAT mode. Proxmox received a VirtualBox-internal IP address rather than a real LAN IP, making the management interface inaccessible from the host machine.

**Status:** ❌ Failed

---

### Attempt 3 — 11:07

**Goal:** Fresh install with corrected network and BIOS settings

**Changes made:**
- Switched VirtualBox network adapter from NAT to **Bridged** — Proxmox now requests an IP directly from the home router via the host NIC
- Enabled **SVM** in host BIOS, then enabled **Nested VT-x/AMD-V** in VirtualBox — required to run VMs inside the Proxmox guest
- Performed a clean reinstall to reset all prior configuration

**Result:** Proxmox web UI accessible at `https://<proxmox-ip>:8006`. Login confirmed working at 11:47.

**Proxmox guest specs at this stage:**
| Property | Value |
|---|---|
| CPU | 8 cores (nested VT-x enabled) |
| RAM | 32 GB |
| Storage | 125 GB (later increased to 232 GB) |
| Network | Bridged adapter via host ethernet |

**Status:** ✅ Success

---

### pfSense Attempts Inside VirtualBox

With Proxmox running, the next step was to deploy pfSense. Two critical problems were encountered:

**Problem 1 — Single network adapter:**
The pfSense VM was initially created with only one network adapter. pfSense requires two — one for WAN (facing the home router) and one for LAN (facing the lab network). Without both, WAN and LAN communication was impossible. Resolution: added a second adapter (`vmbr1`) as an internal-only bridge with no physical NIC attached.

**Problem 2 — WAN/LAN subnet conflict:**
pfSense LAN was configured on `192.168.1.x` — the same subnet as the home network on the WAN side. pfSense has a built-in protection that disables web UI access when WAN and LAN share the same subnet, as it cannot distinguish internal from external traffic. Resolution: LAN was reconfigured to `10.10.10.0/24`, a completely separate subnet.

**Problem 3 — WAN failed to obtain DHCP lease:**
Even with correct bridge configuration (`vmbr0` bridged to physical NIC with `bridge-ports nic0`), pfSense WAN could not receive an IP from the home router. Root cause: VirtualBox blocks traffic destined for MAC addresses it doesn't own by default. Fix: enabled **Promiscuous Mode → Allow All** on the VirtualBox bridged adapter. WAN obtained an IP after this change.

**Outcome of Act 1:** Despite resolving the promiscuous mode issue, a compounding set of nested virtualization problems remained — Windows 11 VM UEFI boot loops, unreliable VirtIO drivers, and persistent networking instability. A fundamental architectural decision was made: migrate Proxmox to bare metal.

---

## Act 2 — Migration to Bare Metal

### Decision

After diagnosing the root cause of all Act 1 problems — Proxmox is a Type 1 hypervisor designed to run directly on hardware, not as a guest inside a Type 2 hypervisor — the decision was made to repurpose a dedicated machine for the lab.

**Key insight:** Running Proxmox inside VirtualBox creates a Type 2 → Type 1 nesting problem. Every networking, driver, and boot issue encountered in Act 1 was a direct consequence of this architectural mismatch. Moving to bare metal eliminates all of these issues at the source.

---

### Hardware — Dell Inspiron 15 3530

| Property | Value |
|---|---|
| CPU | Intel — 6 cores |
| RAM | 8 GB DDR4 |
| Storage | 512 GB SSD |
| Network | No built-in ethernet port |

**Wi-Fi limitation discovered:**
Standard Wi-Fi frames only support three MAC addresses (Source, Destination, BSSID). Virtual machines have their own virtual MAC addresses. When VM traffic is pushed over a Wi-Fi connection, the router drops packets because it does not recognize the VM's MAC address coming from the physical Wi-Fi card. This made Wi-Fi bridging impossible for the lab.

**Resolution:** Ordered a USB-C ethernet adapter to provide a wired connection. All Proxmox networking runs over ethernet.

---

### Proxmox Bare Metal Installation

Proxmox VE was installed using **Ventoy** — a tool that allows multiple ISOs to be loaded from a single USB drive without reflashing.

**Post-install configuration:**
- Static IP `192.168.1.25` reserved in home router DHCP settings to prevent IP changes on reboot
- Two repositories added to Proxmox for updates without a subscription:
  - `No-Subscription` repository
  - `ceph-squid No-Subscription` repository

**Proxmox bare metal specs:**
| Property | Value |
|---|---|
| CPU | 4 cores |
| RAM | 8 GB |
| Storage | 512 GB SSD |
| Network | Bridged via USB-C ethernet adapter |
| Web UI | `https://192.168.1.25:8006` |

---

### ISOs Uploaded to Proxmox

| ISO | Purpose |
|---|---|
| Windows Server 2022 Evaluation | DC-01, DC-02 |
| Windows 11 Enterprise | WIN-CLIENT |
| pfSense CE 2.7.2 | Firewall / NAT / routing |

---

## Network Architecture

### Proxmox Bridge Configuration

| Bridge | Type | Physical NIC | Purpose |
|---|---|---|---|
| `vmbr0` | Linux Bridge | `nic0` (ethernet adapter) | WAN — carries traffic to/from home router |
| `vmbr1` | Linux Bridge | None | LAN — internal lab network only |

`vmbr1` has no physical NIC attached and no IP assigned. It is a purely internal virtual switch. VMs connected to `vmbr1` have no path to the outside world except through pfSense.

### IP Reference Table

| Device | IP | Role |
|---|---|---|
| Home router | `192.168.1.1` | Home network gateway |
| Proxmox host | `192.168.1.25` | Hypervisor management |
| pfSense WAN | `192.168.1.26` | Lab WAN interface |
| pfSense LAN | `10.10.10.1` | Lab default gateway |
| DC-01 | `10.10.10.10` | Primary DC · DNS server |
| DC-02 | `10.10.10.11` | RODC replica |
| WIN-CLIENT | `10.10.10.100` | Domain workstation (DHCP) |

### DNS Chain

```
Lab VM query (e.g. google.com)
    │
DC-01 (10.10.10.10) — authoritative for davy.local
    │ unknown external domain
pfSense (10.10.10.1) — forwarder
    │
Internet DNS (8.8.8.8)
```

All lab VMs point exclusively to DC-01 for DNS. DC-01 resolves `davy.local` internally and forwards all external queries to pfSense. pfSense forwards to the internet. No lab VM needs to know about pfSense or external DNS directly.

---

## VM Specifications

### pfSense

| Property | Value |
|---|---|
| OS | pfSense CE 2.7.2 |
| CPU | 1 core |
| RAM | 2 GB |
| Storage | 10 GB |
| net0 | `vmbr0` (WAN) |
| net1 | `vmbr1` (LAN) |
| BIOS | SeaBIOS |

### DC-01

| Property | Value |
|---|---|
| OS | Windows Server 2022 Standard Evaluation (Desktop Experience) |
| CPU | 2 cores |
| RAM | 2 GB |
| Storage | 60 GB (SCSI, thin provisioned) |
| Network | `vmbr1` |
| BIOS | OVMF (UEFI) |
| Machine | q35 |
| TPM | v2.0 |
| Static IP | `10.10.10.10` |

### DC-02

| Property | Value |
|---|---|
| OS | Windows Server 2022 Standard Evaluation (Desktop Experience) |
| CPU | 1 core |
| RAM | 2 GB |
| Storage | 60 GB (SCSI, thin provisioned) |
| Network | `vmbr1` |
| BIOS | OVMF (UEFI) |
| Machine | q35 |
| TPM | v2.0 |
| Static IP | `10.10.10.11` |

### WIN-CLIENT

| Property | Value |
|---|---|
| OS | Windows 11 Enterprise |
| CPU | 2 cores |
| RAM | 2 GB |
| Storage | 64 GB (SCSI, thin provisioned) |
| Network | `vmbr1` |
| BIOS | OVMF (UEFI) |
| Machine | q35 |
| TPM | v2.0 |
| IP | `10.10.10.100` (DHCP from pfSense) |

> **Note on RAM allocation:** The Dell Inspiron 15 3530 has 8 GB RAM total. All VMs were allocated 2 GB each to fit within this constraint. Windows Server 2022 operates stably at 2 GB in a lab environment with light load. A RAM upgrade to 16-24 GB is planned for future Tier 2 expansion.

---

## pfSense Build & Configuration

### VM Creation Notes

pfSense requires two network adapters added **before first boot**. The order determines interface assignment:
- `net0` → `vmbr0` → becomes WAN (vtnet0)
- `net1` → `vmbr1` → becomes LAN (vtnet1)

### Installation

- Filesystem: UFS (ZFS avoided — single disk provides no redundancy benefit from ZFS, and UFS avoids the disk selection requirement that caused initial confusion)
- Default install settings accepted throughout

### Interface Configuration

After installation, pfSense console was used to configure interfaces:

**WAN:**
- Interface: `vtnet0`
- Mode: DHCP — receives IP from home router automatically
- Result: `192.168.1.26/24`

**LAN:**
- Interface: `vtnet1`
- Mode: Static
- IP: `10.10.10.1/24`
- Gateway: none (pfSense IS the gateway)
- DHCP server enabled: yes
  - Range start: `10.10.10.100`
  - Range end: `10.10.10.200`

**Final console state:**
```
WAN (wan) -> vtnet0 -> v4/DHCP4: 192.168.1.26/24
LAN (lan) -> vtnet1 -> v4: 10.10.10.1/24
```

### Web UI Setup Wizard

Accessed at `https://10.10.10.1` from WIN-CLIENT after it received its DHCP lease. Default credentials: `admin` / `pfsense` — changed immediately.

Setup wizard configured:
- Hostname: `pfSense`
- Domain: `davy.local`
- DNS servers: forwarded to ISP
- Timezone set
- WAN DHCP confirmed
- LAN IP confirmed `10.10.10.1`

**Connectivity verification:**
- WIN-CLIENT received DHCP lease `10.10.10.100` ✅
- WIN-CLIENT can ping pfSense LAN `10.10.10.1` ✅
- WIN-CLIENT can ping home router `192.168.1.1` via pfSense ✅
- WIN-CLIENT can ping `8.8.8.8` ✅
- Internet browsing functional after setup wizard completed ✅

---

## DC-01 — Primary Domain Controller

### Pre-Promotion Configuration

1. Renamed server to `DC-01` via Settings → System → About → Rename → rebooted
2. Set static IP via network adapter properties:

| Setting | Value |
|---|---|
| IP | `10.10.10.10` |
| Subnet | `255.255.255.0` |
| Gateway | `10.10.10.1` |
| DNS | `127.0.0.1` (loopback — temporary until AD DS installed) |

> **Why loopback for DNS during setup:** AD DS installation creates the DNS server on DC-01. Pointing to `127.0.0.1` temporarily prevents chicken-and-egg DNS failures during promotion. After promotion, DNS was updated to `10.10.10.10` (its own real IP) for network consistency and replication compatibility.

### AD DS Role Installation

Server Manager → Add Roles and Features → Active Directory Domain Services → installed with all required features.

### Domain Controller Promotion

Promoted via Server Manager yellow flag → "Promote this server to a domain controller":

| Setting | Value |
|---|---|
| Operation | Add a new forest |
| Root domain | `davy.local` |
| Forest functional level | Windows Server 2016 |
| Domain functional level | Windows Server 2016 |
| DNS Server | ✅ enabled |
| Global Catalog | ✅ enabled |
| DSRM password | set and stored securely |

Server rebooted automatically after promotion. Login prompt changed to `DAVY\Administrator` — confirmed AD is operational.

### Post-Promotion Configuration

- DNS Manager confirmed forward lookup zone `davy.local` created automatically with SRV records populated
- pfSense added as DNS forwarder: DNS Manager → server Properties → Forwarders → `10.10.10.1`
- DNS on adapter updated from `127.0.0.1` to `10.10.10.10`

> **Why change from loopback to real IP:** `127.0.0.1` only resolves locally on the machine itself. `10.10.10.10` is the real network address visible to all machines. DC-02 replication and WIN-CLIENT domain joins both require DC-01 to resolve domain records over the actual network, not loopback.

---

## DC-02 — Read-Only Domain Controller

### Purpose of an RODC

A Read-Only Domain Controller holds a copy of the AD database that cannot be modified. Any write operation — creating users, joining computers, changing passwords — must be performed against a writable DC (DC-01). The RODC then receives those changes via replication.

RODCs are designed for environments where physical server security cannot be guaranteed (branch offices, remote sites). If an RODC is compromised, the attacker cannot modify the domain.

### Pre-Promotion Configuration

1. Renamed to `DC-02` → rebooted
2. Set static IP:

| Setting | Value |
|---|---|
| IP | `10.10.10.11` |
| Subnet | `255.255.255.0` |
| Gateway | `10.10.10.1` |
| DNS | `10.10.10.10` (DC-01) |

> DNS points to DC-01, not itself — DC-02 must locate and join the existing domain before it can serve as a DC.

### Domain Join (Member Server First)

Before promoting to DC, DC-02 was joined to `davy.local` as a standard member server:
- Settings → System → About → Domain → `davy.local`
- Credentials: `DAVY\Administrator`
- Rebooted — confirmed domain joined

### AD DS Role and RODC Promotion

Promotion settings:

| Setting | Value |
|---|---|
| Operation | Add a domain controller to an existing domain |
| Domain | `davy.local` |
| Read Only Domain Controller | ✅ enabled |
| DNS Server | ✅ enabled |
| Global Catalog | ✅ enabled |
| Site | Default-First-Site-Name |

Server rebooted after promotion.

### RODC Permission Discovery

During verification, write operations (creating and deleting computer objects) appeared to succeed when performed as `DAVY\Administrator` on DC-02. Investigation confirmed this is **expected behavior**:

Domain Admins have a built-in exemption that bypasses RODC read-only restrictions. The read-only enforcement applies to standard domain users, not privileged accounts. This is by design — administrators must be able to manage the domain from any DC regardless of its role.

**Verified with:**
```powershell
Get-ADDomainController -Filter * | Select-Object Name, IsReadOnly
```

Output:
```
Name   IsReadOnly
----   ----------
DC-01  False
DC-02  True
```

RODC confirmed correctly configured. ✅

---

## WIN-CLIENT — Domain Workstation

### Network Configuration

WIN-CLIENT received its IP automatically via DHCP from pfSense:

| Setting | Value |
|---|---|
| IP | `10.10.10.100` (DHCP) |
| Gateway | `10.10.10.1` (pfSense) |
| DNS | `10.10.10.10` (DC-01) |

> DNS must point to DC-01, not pfSense. The domain join process begins with a DNS lookup for `davy.local`. If DNS points to pfSense or an external server, that lookup fails and the join fails before it begins.

### Domain Join

- Settings → System → About → Domain → `davy.local`
- Credentials: `DAVY\Administrator`
- Rebooted — login prompt now shows `DAVY\` prefix

### Verification

WIN-CLIENT visible in AD Users and Computers → Computers container on DC-01 ✅  
WIN-CLIENT visible on DC-02 via replication ✅  
Internet browsing functional through pfSense ✅

---

## Verification & Health Checks

### DNS Health — DC-01

```powershell
dcdiag /test:dns
```

Expected: DC-01 passes all DNS tests. pfSense (`10.10.10.1`) will show failures — this is expected and harmless. pfSense is a firewall, not a domain DNS server, and holds none of the AD SRV records `dcdiag` is looking for. These errors can be ignored.

**Resolution:** Remove pfSense from the alternate DNS field on DC-01's network adapter. pfSense should only be referenced as a **forwarder** inside DNS Manager, not as an adapter-level DNS server. This stops pfSense from appearing in `dcdiag` results.

---

### Replication Health

Run on DC-01:
```powershell
repadmin /replsummary
repadmin /showrepl
```

Expected output shows both DC-01 and DC-02 with no errors and recent replication timestamps.

Run on DC-02:
```powershell
dcdiag /test:replications
```

All passing confirms DC-02 is actively syncing from DC-01. ✅

---

### Full Verification Checklist

| Check | Command / Method | Expected Result |
|---|---|---|
| pfSense LAN reachable | Ping `10.10.10.1` from WIN-CLIENT | Reply ✅ |
| pfSense web UI accessible | Browse `https://10.10.10.1` from WIN-CLIENT | Login page ✅ |
| Internet through pfSense | Ping `8.8.8.8` from WIN-CLIENT | Reply ✅ |
| DC-01 DNS healthy | `dcdiag /test:dns` on DC-01 | DC-01 passes ✅ |
| DC-01 reachable by name | `nslookup davy.local` from WIN-CLIENT | Returns `10.10.10.10` ✅ |
| Replication healthy | `repadmin /replsummary` on DC-01 | No errors ✅ |
| DC-02 RODC confirmed | `Get-ADDomainController -Filter * \| Select Name, IsReadOnly` | DC-02 IsReadOnly: True ✅ |
| WIN-CLIENT domain joined | AD Users and Computers → Computers | WIN-CLIENT visible ✅ |
| GPO applying | `gpresult /r` on WIN-CLIENT | Policy applied ✅ |
| WIN-CLIENT internet | Browse any site on WIN-CLIENT | Working ✅ |

---

## Key Lessons Learned

**Type 1 vs Type 2 hypervisors**  
Running a Type 1 hypervisor (Proxmox) inside a Type 2 (VirtualBox) creates cascading incompatibilities in networking, drivers, and UEFI boot. Proxmox must run on bare metal to function as intended. This was the single most important architectural lesson of the entire project.

**Subnet separation is non-negotiable with pfSense**  
pfSense cannot function correctly when WAN and LAN share the same subnet. It disables the web UI as a security measure. Network design must ensure the lab subnet is completely separate from any upstream network.

**Wi-Fi cannot bridge VM MAC addresses**  
Standard Wi-Fi frames only carry three MAC addresses. VM traffic with distinct virtual MAC addresses gets dropped by the router when pushed over Wi-Fi. Wired ethernet is required for any environment where VMs need bridged network access.

**DNS is the foundation of Active Directory**  
Every AD operation — domain joins, authentication, replication, GPO application — depends entirely on DNS. DC-01 must be the DNS authority for the domain and all clients must point to it. External DNS (pfSense, Google) must only be referenced as forwarders, never as primary DNS for domain-joined machines.

**RODC admin exemption**  
Domain Admins bypass RODC read-only restrictions by design. Testing RODC behavior requires a standard domain user account — admin accounts will always appear to have write access regardless of which DC they authenticate against.

**Thin provisioning saves significant storage**  
Using LVM-Thin in Proxmox allows disk allocations to exceed physical storage as long as actual usage stays within limits. Four VMs allocated 194 GB total only consume a fraction of that on disk. Monitor real usage with `lvs` on the Proxmox host.

---

## Roadmap

### Tier 1 — Core infrastructure ✅ Complete
- [x] Proxmox VE bare metal
- [x] pfSense firewall and network isolation
- [x] DC-01 primary domain controller
- [x] DC-02 RODC replica
- [x] WIN-CLIENT domain workstation

### Tier 2 — Attack & defense (upcoming)
- [ ] Kali Linux attack VM
- [ ] BloodHound — AD attack path visualization
- [ ] Impacket — Kerberoasting / Pass-the-Hash simulation
- [ ] CrackMapExec — lateral movement testing
- [ ] Wazuh SIEM — log collection and threat detection
- [ ] Sysmon — enhanced Windows event logging
- [ ] Velociraptor — endpoint visibility

### Tier 3 — Identity & automation (planned)
- [ ] ADCS — internal certificate authority, PKI
- [ ] Keycloak — SAML/OAuth2 SSO federation
- [ ] Ansible — automated AD provisioning
- [ ] HashiCorp Vault — secret management
- [ ] Gitea + Jenkins — CI/CD pipeline

---

*Built from scratch. Documented throughout. Every failure was a lesson.*
