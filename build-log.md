# Homelab Build Log
**Owner:** Darryl Briggs  
**Goal:** Build a functioning enterprise-style network to develop hands-on cybersecurity and IT skills in support of CompTIA certifications and a career transition into GRC and cybersecurity engineering.  
**Host:** iMac 2017 — Intel Core i7 — 48GB RAM — VirtualBox  
**Domain:** homelab.local  

---

## Entry 001 — Lab Planning & Platform Decision
**Date:** March 2026  
**Status:** ✅ Complete

**Decision:** Build a dedicated home lab environment to support hands-on learning for CompTIA A+, Network+, Security+, and cybersecurity engineering skills.

**Goals established:**
- Build a safe isolated environment to practice offensive and defensive security techniques
- Replicate a real enterprise network with domain authentication and managed workstations
- Support hands-on study for CompTIA certifications
- Develop skills applicable to SOC analyst and GRC analyst roles

**Initial platform:** MacBook Pro M1 Max (32GB RAM) running UTM hypervisor

---

## Entry 002 — Initial Lab Build on M1 Max (Deprecated)
**Date:** March–April 2026  
**Status:** ⚠️ Deprecated — Migrated to iMac

**Platform:** MacBook Pro M1 Max · 32GB RAM · UTM Hypervisor

**VMs built:**
- Kali Linux (ARM64 — native)
- Ubuntu Server (ARM64 — native)
- Ubuntu Desktop (ARM64 — native)
- Windows Server 2025 (x86 — QEMU emulation)
- Windows 11 Pro (x86 — QEMU emulation)

**Problem encountered:** Windows Server 2025 and Windows 11 Pro required x86 emulation via QEMU on Apple Silicon. This caused severe performance degradation — Active Directory Domain Services installation took several hours and rendered both machines nearly unusable for lab work.

**Resolution:** Decision made to migrate entire lab to a dedicated Intel x86 host where all VMs run natively without emulation overhead.

**Lesson learned:** Match your hypervisor platform to your VM architecture. ARM hosts running x86 emulation are not suitable for Windows Server workloads in a lab environment.

---

## Entry 003 — Platform Migration to iMac 2017
**Date:** May 2026  
**Status:** ✅ Complete

**Action:** Decommissioned the UTM-based lab on the M1 Max. Rebuilt the lab from scratch on a dedicated iMac 2017 running VirtualBox.

**New host specs:**
- CPU: Intel Core i7
- RAM: 48GB
- Hypervisor: VirtualBox (free, well-documented, widely used in professional environments)
- Network mode: Host-Only Adapter — all VMs fully isolated from the internet and physical home network

**VMs installed:**
- Kali Linux
- Windows Server 2025
- Windows 11 Pro

**Network configuration:** All VMs assigned static IP addresses on the VirtualBox host-only adapter for reliable inter-VM communication.

**Rationale for iMac:** Intel i7 architecture runs all VMs natively with zero emulation overhead. 48GB RAM supports five or more simultaneous VMs comfortably.

---

## Entry 004 — Network Connectivity Troubleshooting
**Date:** May 2026  
**Status:** ✅ Resolved

**Problem:** After assigning static IP addresses via the graphical interface on Kali Linux, the changes did not reflect when checking configuration from the command line using `ip a`.

**Root cause:** NetworkManager had not applied the new configuration. The GUI saves settings but the active network connection requires a service restart before new addresses take effect.

**Resolution:**
```bash
sudo systemctl restart NetworkManager
```

**Verified with:**
```bash
ip a
ping [Windows Server IP]
```

**Lesson learned:** On Linux systems using NetworkManager, GUI changes require a service restart or interface bounce to take effect. Always verify with `ip a` after making network changes, not just the GUI display.

---

## Entry 005 — Windows Firewall ICMP Configuration
**Date:** May 2026  
**Status:** ✅ Resolved

**Problem:** Kali Linux could not ping Windows Server 2025 or Windows 11 Pro despite all VMs being on the same host-only subnet with correct IP assignments.

**Root cause:** Windows blocks incoming ICMP (ping) requests by default via Windows Defender Firewall.

**Resolution:** Created an inbound firewall rule to allow ICMPv4 echo requests:
```cmd
netsh advfirewall firewall add rule name="Allow ICMPv4" protocol=icmpv4:8,any dir=in action=allow
```

**Verified:** Successful ping responses from Kali to both Windows VMs.

**Lesson learned:** Windows Firewall blocks ICMP by default — a common gotcha in lab environments. In production, ICMP blocking is intentional security policy. In a lab, it needs to be explicitly permitted for connectivity testing.

---

## Entry 006 — Active Directory Domain Services Installation
**Date:** May–June 2026  
**Status:** ✅ Complete

**Action:** Installed the Active Directory Domain Services (AD DS) role on Windows Server 2025 and promoted the server to Domain Controller.

**Steps executed:**
1. Set static IP on Windows Server 2025
2. Renamed server to DC01 before promotion
3. Installed AD DS role via Server Manager → Add Roles and Features
4. Promoted server to Domain Controller — created new forest with root domain `homelab.local`
5. Set Directory Services Restore Mode (DSRM) password
6. Server rebooted automatically after promotion

**Verified:**
```powershell
Get-ADDomain
Get-ADForest
```

**Outcome:** Domain Controller operational. DNS integrated with AD. `homelab.local` domain active.

**Note:** AD DS installation on the iMac completed in normal time — a stark contrast to the multi-hour installation under QEMU emulation on the M1 Max. Native x86 confirmed as the correct platform decision.

---

## Entry 007 — Windows 11 Domain Join
**Date:** June 2026  
**Status:** ✅ Complete

**Action:** Joined Windows 11 Pro workstation to the `homelab.local` domain managed by DC01.

**Steps executed:**
1. Set DNS on Windows 11 to point to DC01's IP address
2. Verified connectivity: `nslookup homelab.local` returned DC01's IP
3. Joined domain via Settings → System → About → Domain or workgroup → Join a domain
4. Entered domain admin credentials when prompted
5. Rebooted Windows 11
6. Logged in with domain credentials

**Verified on DC01:**
```powershell
Get-ADComputer -Filter * | Select Name
```
Windows 11 workstation appeared in Active Directory Computers container.

**Outcome:** Windows 11 Pro successfully domain-joined. Domain authentication working.

---

## Entry 008 — Active Directory User Creation
**Date:** June 2026  
**Status:** ✅ Complete

**Action:** Created Organizational Units and user accounts in Active Directory to simulate an enterprise user directory.

**Method 1 — PowerShell:**
```powershell
New-ADOrganizationalUnit -Name "Users" -Path "DC=homelab,DC=local"

New-ADUser `
  -Name "John Doe" `
  -GivenName "John" `
  -Surname "Doe" `
  -SamAccountName "jdoe" `
  -UserPrincipalName "jdoe@homelab.local" `
  -Path "OU=Users,DC=homelab,DC=local" `
  -AccountPassword (ConvertTo-SecureString "Password123!" -AsPlainText -Force) `
  -Enabled $true
```

**Method 2 — GUI (Active Directory Users and Computers):**
- Opened ADUC via Windows Administrative Tools
- Created OU via right-click → New → Organizational Unit
- Created user via right-click → New → User
- Set password, unchecked "must change at next logon"

**Verified:** Domain login from Windows 11 using `HOMELAB\jdoe` credentials successful.

**Outcome:** AD user creation via both PowerShell and GUI confirmed working. OU structure established.

---

## Entry 009 — Domain Controller Naming Error and Full Rebuild
**Date:** June 6, 2026  
**Status:** ✅ Resolved

**Problem:** After Active Directory was fully operational, the Windows Server 2025 VM was renamed. This broke Active Directory services and rendered the domain controller non-functional.

**Root cause:** Domain Controller names are embedded in Active Directory during the promotion process. The server's name is registered in AD DNS, Kerberos authentication tickets, replication metadata, and SYSVOL configuration. Renaming a DC after promotion causes cascading failures across all of these systems that are difficult to cleanly resolve — particularly in a single-DC lab environment with no replication partner to recover from.

**Attempted resolution:** Troubleshooting the broken AD state.

**Final resolution:** Made the decision to perform a full reinstall rather than continue troubleshooting a broken configuration. Steps taken:
1. Uninstalled Active Directory Domain Services role
2. Demoted the domain controller
3. Performed clean reinstall of Windows Server 2025
4. Named the server correctly before any role installation
5. Reinstalled AD DS role
6. Repromoted to Domain Controller with domain `homelab.local`
7. Reconfigured DNS and verified domain functionality

**Outcome:** Clean Active Directory installation confirmed. Domain `homelab.local` operational. DNS configured. Ready to rejoin Windows 11 workstation to domain.

**Lesson learned:** In Active Directory environments, the server name is a foundational identity — not a display label that can be changed freely. Always name your domain controller correctly before promotion. In production enterprise environments, renaming a DC is a carefully managed multi-step process documented by Microsoft. In a lab, knowing when to rebuild cleanly rather than chase a broken state is itself a valuable operational judgment call. Document the failure, understand why it happened, and move forward.

**Real-world relevance:** Change management exists in enterprise IT for exactly this reason. Configuration changes to critical infrastructure — especially identity systems like Active Directory — require planning, documentation, and a rollback plan before execution.

---

## Entry 010 — First Vulnerability Scan with Nessus Essentials

**Date:** June 27, 2026
**Status:** ✅ Complete

**Action:** Installed Nessus Essentials on the Windows 11 Pro VM and ran a vulnerability scan against the homelab.local network subnet. OpenVAS installation also in progress for comparison between the two scanners.

**Steps executed:**

- Installed Nessus Essentials on the Windows 11 Pro VM
- Configured a basic network scan targeting the VirtualBox host-only subnet
- Ran the scan and reviewed results in the Nessus dashboard

**Findings:**

- Windows 11 Pro was the only host that returned as active on the scanned network — other VMs (DC01, Kali) were not powered on at the time of the scan
- One vulnerability identified on the Windows 11 Pro VM:
  - **Self-signed X.509 certificate** — a certificate presented by the Nessus web interface running on the Windows 11 VM cannot be trusted because it is self-signed rather than issued by a recognized Certificate Authority
  - **CVSS 3.0 Base Score:** 6.5 (Medium)
  - **Recommended remediation per Nessus:** purchase or generate a proper SSL certificate from a trusted CA

**Verified:**

- Reviewed finding detail and CVSS breakdown in Nessus dashboard
- Confirmed the flagged service was the Nessus web interface itself, not a separate exposed service

**Outcome:** First successful vulnerability scan completed and documented. Confirmed ability to install, configure, and interpret results from an industry-standard vulnerability scanner. CVSS scoring and remediation guidance reviewed and understood in the context of a real, if minor, finding.

**Lesson learned:** A "clean-looking" scan with only one low-to-medium finding is still a valuable exercise — vulnerability management isn't only about finding critical flaws, it's about being able to read, score, and contextualize every finding a scanner returns, including ones related to the scanner's own service. Self-signed certificates are flagged because there's no trusted third party vouching for the identity of the service presenting them, which opens the door to man-in-the-middle attacks.

**Real-world relevance:** Production environments avoid this exact finding by issuing certificates through an internal or public Certificate Authority rather than relying on self-signed certs for anything beyond local testing. Next step is to power on the full VM network (DC01, Kali, Windows 11) during a scan to get a complete picture of the lab's exposed surface, and to run a parallel scan with OpenVAS once installation is complete for comparison between the two platforms.

---

## Entry 011 — Added Secondary NAT Adapter for Internet Access

**Date:** June 28, 2026
**Status:** ✅ Complete

**Action:** Added a second network adapter (Adapter 2) configured for NAT on Kali Linux, Windows Server 2025, and Windows 11 Pro — giving each VM outbound internet access while preserving the existing Host-Only adapter (Adapter 1) for isolated inter-VM communication.

**Steps executed:**

- Opened VM Settings → Network in VirtualBox for each VM
- Enabled Adapter 2 and set it to NAT
- Left Adapter 1 (Host-Only) untouched, maintaining the existing isolated homelab.local network
- Repeated for Kali Linux, Windows Server 2025, and Windows 11 Pro

**Use case:** Downloading OS and tool updates, and installing additional software (Nessus, OpenVAS, future tools) directly within each VM without needing to transfer files manually or temporarily reconfigure the primary network adapter.

**Outcome:** All three VMs now have dual network adapters — Adapter 1 (Host-Only) for isolated lab traffic and domain communication, Adapter 2 (NAT) for outbound internet access. Setup completed via the VirtualBox GUI with no issues. Internet access confirmed working alongside continued connectivity on the isolated homelab.local network.

**Lesson learned:** Running a dual-adapter configuration is a common pattern in real lab and test environments — it allows a system to stay isolated from production or sensitive networks on one interface while still being reachable or update-capable on another. Adapter 1 remaining Host-Only means the lab's internal isolation is unaffected by adding outbound internet access; the two networks don't bridge or route into each other through VirtualBox's NAT implementation.

**Real-world relevance:** This mirrors how enterprise environments often segment management or update traffic onto a separate interface or VLAN from production/internal traffic — keeping the sensitive network isolated while still allowing controlled access for patching and tool installation.

## Entry 012 — VM Storage Migration to External Drive

**Date:** July 1, 2026
**Status:** ✅ Complete

**Action:** Used VirtualBox's built-in Move function to completely migrate all VM folders — including virtual disk files, snapshots, and configuration files — from the iMac internal storage to an external HDD. The migration was prompted by snapshot files accumulating on the host machine and consuming significant internal storage space.

**Steps executed:**

- Identified internal storage pressure caused by accumulated VM snapshots on the host machine
- Powered down all VMs cleanly before initiating the move
- Used VirtualBox Machine → Move function for each VM — this relocates the entire VM folder including virtual disks, snapshots, and configuration files in a single operation
- Confirmed each VM's new path pointed to the external HDD after the move completed
- Powered on each VM and confirmed normal operation from the external drive

**Outcome:** All VM folders fully migrated to external HDD including snapshots. Internal storage space restored on the iMac host. No data loss or VM corruption during the migration. VirtualBox Move function handled all path updates automatically.

**Lesson learned:** VirtualBox VMs are portable — the virtual disk files can be moved between storage locations as long as the VM settings are updated to reflect the new path. Powering down cleanly before moving files is critical to avoid disk corruption. This mirrors real-world scenarios where virtual machine storage is migrated between datastores or storage arrays in enterprise environments.

**Real-world relevance:** Storage management is a routine operational task in enterprise IT environments. Virtual machines in production are regularly migrated between storage systems for capacity management, performance optimization, or hardware maintenance — all requiring careful planning and verification to avoid data loss or service interruption.

---

## Entry 013 — Full Network Vulnerability Scan with All VMs Active

**Date:** July 1, 2026
**Status:** ✅ Complete

**Action:** Conducted a full vulnerability scan of the homelab.local network using Nessus Essentials with all VMs powered on simultaneously — DC01 (Windows Server 2025), WIN11 (Windows 11 Pro), Kali Linux, Ubuntu Desktop, and Ubuntu Server.

**Steps executed:**

- Powered on all VMs on the VirtualBox host-only network
- Configured Nessus scan to target the full host-only subnet
- Ran the scan and reviewed results across all active hosts

**Findings:**

**Finding 1 — Self-signed X.509 Certificate (Windows 11 Pro)**
- Same finding as Entry 010 — Nessus web interface presenting a self-signed certificate
- CVSS 3.0 Base Score: 6.5 (Medium)
- Status: Accepted risk in lab environment — no action taken

**Finding 2 — ICMP Timestamp Request/Reply Enabled (All VMs)**
- Affected hosts: Windows 11 Pro, Kali Linux, Ubuntu Desktop, Ubuntu Server
- CVSS 3.0 Base Score: 1.4 (Low)
- Description: All active hosts were responding to ICMP timestamp requests (Type 13) and sending timestamp replies (Type 14). This allows a remote attacker to determine the system clock on targeted machines, which can be used to defeat time-based authentication protocols such as Kerberos — particularly relevant in an Active Directory environment where Kerberos is the primary authentication mechanism
- Root cause: The broad ICMPv4 firewall rule created in Entry 005 permitted all ICMP types including timestamp requests and replies, not just echo requests (ping)
- Remediation decision: Remediate — apply OS-specific firewall rules to block ICMP Types 13 and 14 on all affected hosts while preserving echo request (Type 8) functionality for connectivity testing

**Outcome:** Full network scan completed with all VMs active. Two findings identified — one accepted, one flagged for immediate remediation. See Entry 014 for full remediation documentation.

**Lesson learned:** Running a scan with all VMs powered on reveals a materially different attack surface than scanning a single host. The ICMP timestamp finding appeared on every Linux and Windows host simultaneously — a finding that would have been missed entirely in the single-VM scan from Entry 010. Full network scans should be standard practice, not an afterthought.

**Real-world relevance:** In enterprise environments, vulnerability scans are run against the full network, not individual machines in isolation. A finding that affects every host simultaneously represents a systemic misconfiguration requiring coordinated remediation across multiple systems, which is exactly what this exercise produced.

---

## Entry 014 — ICMP Timestamp Vulnerability Remediation Across All VMs

**Date:** July 2, 2026
**Status:** ✅ Complete

**Action:** Remediated the ICMP timestamp request/reply vulnerability (Entry 013, Finding 2) across all four affected VMs using OS-appropriate firewall methods. Each operating system required a different approach.

**Remediation steps by VM:**

**Windows 11 Pro:**

- Deleted the original broad ICMPv4 rule from Entry 005:
```
netsh advfirewall firewall delete rule name="Allow ICMPv4"
```
- Added three specific replacement rules:
```
netsh advfirewall firewall add rule name="Allow ICMPv4-Echo" protocol=icmpv4:8,any dir=in action=allow
netsh advfirewall firewall add rule name="Block ICMPv4-Timestamp-In" protocol=icmpv4:13,any dir=in action=block
netsh advfirewall firewall add rule name="Block ICMPv4-Timestamp-Out" protocol=icmpv4:14,any dir=out action=block
```
- Persistence method: Native Windows Defender Firewall — rules persist automatically across reboots

**Kali Linux:**

- Added DROP rules via iptables using named ICMP types:
```
sudo iptables -A INPUT -p icmp --icmp-type timestamp-request -j DROP
sudo iptables -A OUTPUT -p icmp --icmp-type timestamp-reply -j DROP
```
- Issue encountered: OUTPUT rule initially written as Type 13 instead of Type 14 — identified via `iptables -L -n -v` and corrected by deleting the incorrect rule and re-adding with the correct type
- Persistence method: iptables-persistent package
```
sudo apt install iptables-persistent -y
sudo netfilter-persistent save
```

**Ubuntu Desktop:**

- Added DROP rules via iptables using numeric ICMP types (named type syntax not supported on this Ubuntu version):
```
sudo iptables -A INPUT -p icmp --icmp-type 13 -j DROP
sudo iptables -A OUTPUT -p icmp --icmp-type 14 -j DROP
```
- Persistence method: iptables-persistent package
```
sudo apt install iptables-persistent -y
sudo netfilter-persistent save
```

**Ubuntu Server:**

- Added DROP rules via iptables using numeric ICMP types:
```
sudo iptables -A INPUT -p icmp --icmp-type 13 -j DROP
sudo iptables -A OUTPUT -p icmp --icmp-type 14 -j DROP
```
- Issue encountered: iptables-persistent could not be installed — repository package lists were missing and the update process was disabled due to unsigned repository sources. Did not use `--allow-unauthenticated` flag due to security risk
- Persistence method: Manual save and startup script
```
sudo mkdir -p /etc/iptables
sudo sh -c "iptables-save > /etc/iptables/rules.v4"
sudo nano /etc/network/if-pre-up.d/iptables
```
- Script contents:
```
#!/bin/sh
iptables-restore < /etc/iptables/rules.v4
```
```
sudo chmod +x /etc/network/if-pre-up.d/iptables
```
- Verified executable: `-rwxr-xr-x 1 root root 52`

**Verification:**

- Ran follow-up Nessus scan across full network with all VMs active
- ICMP timestamp finding cleared on all four hosts
- Ping (ICMP Type 8 echo) confirmed still working between VMs — connectivity preserved
- SSL certificate finding on Windows 11 persists as expected — accepted risk, no action taken

**Outcome:** ICMP timestamp vulnerability fully remediated across all VMs. Each OS required a different implementation approach — Windows Firewall rules, iptables-persistent on Kali and Ubuntu Desktop, and a manual save with startup script on Ubuntu Server. Full remediation cycle completed: scan → identify → remediate → verify.

**Lesson learned:** The same vulnerability requires different remediation approaches depending on the operating system. On Linux, iptables syntax also varies between distributions — named ICMP types worked on Kali but required numeric types on Ubuntu. Always verify rule syntax with `iptables -L -n -v` after adding rules, and confirm persistence separately from rule creation — adding rules and saving rules are not the same action.

**Real-world relevance:** In enterprise environments, a finding affecting multiple systems with different operating systems requires coordinated remediation across different teams and toolsets. The Ubuntu Server repository issue mirrors a real-world scenario where a system's package manager is misconfigured or behind on maintenance, requiring a workaround rather than the standard installation path.

## Entry 015 — Full VM Migration to External HDD Using VirtualBox Move Function

**Date:** July 7, 2026
**Status:** ✅ Complete

**Action:** Performed a complete migration of all VM folders from the iMac internal storage to an external HDD using VirtualBox's built-in Move function. This move was prompted by accumulated snapshot files consuming significant internal storage space on the host machine — a follow-up to the partial migration documented in Entry 012.

**Steps executed:**

- Identified continued internal storage pressure caused by snapshot files that had been collecting on the host machine since the lab was first built
- Powered down all VMs cleanly before initiating the move
- Used VirtualBox Machine → Move for each VM — this function relocates the entire VM folder including virtual disk files, snapshots, and configuration files in a single operation and automatically updates all internal VirtualBox path references
- Confirmed each VM's new path pointed to the external HDD after each move completed
- Powered on each VM and confirmed normal operation from the external HDD

**Outcome:** All VM folders fully migrated to external HDD including snapshots and configuration files. Internal storage space fully restored on the iMac host. No data loss or VM corruption during the migration. VirtualBox Move function handled all path updates automatically — no manual reconfiguration required.

**Lesson learned:** VirtualBox's Move function is the correct tool for a complete VM relocation — it moves the entire VM folder and updates all internal references in one operation. This is preferable to manually moving files and updating paths separately, which risks breaking references if any path is missed. Snapshot management should be part of routine lab hygiene — snapshots accumulate quickly and consume significant storage if not monitored.

**Real-world relevance:** Storage capacity management is a routine operational responsibility in enterprise IT environments. Virtual machines in production are regularly migrated between storage systems for capacity management, performance optimization, or hardware refresh cycles. Knowing when to move workloads and how to do so cleanly without data loss or service interruption is a foundational sysadmin skill.

---
*Log continues as the lab grows. Every new configuration, exercise, troubleshooting event, and rebuild is documented here.*
