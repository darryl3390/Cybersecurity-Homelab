# Homelab Build Log

*Ordered newest first. New entries are added at the top of their phase's section going forward.*

**Owner:** Darryl Briggs  
**Goal:** Build a functioning enterprise-style network to develop hands-on cybersecurity and IT skills in support of CompTIA certifications and a career transition into GRC and cybersecurity engineering.  
**Host:** iMac 2017 — Intel Core i7 — 48GB RAM — VirtualBox  
**Domain:** homelab.local  
**Architecture (current — Phase 2):** Segmented network behind pfSense — ATTACK, AD_LAB, and DEFENDER zones as isolated VirtualBox Internal Networks, with pfSense as the sole routing/firewall point between them. Splunk (SIEM) and Eramba (GRC) run in the DEFENDER segment. See Phase 2 entries below for the full build; Phase 1 entries document the original flat-network foundation this was built on top of.

---

# Phase 2 — Segmented Architecture

## Phase 2, Entry 002 — Session 2: VM Migration onto Segments

**Date:** July 14, 2026
**Status:** ✅ Complete

**Action:** Migrated all five existing VMs (DC01, WIN11, Ubuntu Desktop, Kali, Ubuntu Server) off the original flat network and onto their designated segments behind pfSense, then reconfigured all Splunk Universal Forwarders to point at Ubuntu Server's new address.

**Goal:** Complete the segmentation groundwork established in Entry 001 — every VM now sits on its intended zone (ATTACK/AD_LAB/DEFENDER), with pfSense as the only path between them.

---

### Step 1 — Snapshot strategy

Skipped a blanket pre-migration snapshot across all VMs as redundant, given a pre-pfSense snapshot already existed. Made one deliberate exception: **snapshotted DC01 specifically** before its migration, given Entry 009's history — a network-adjacent change to this exact VM previously cascaded into a full AD rebuild with no clean recovery path. Treated as targeted risk management, not blanket process.

### Step 2 — DC01 migration

Migrated to `intnet-adlab` with a static IP, gateway pointed at pfSense's LAN interface. Completed without incident.

### Step 3 — WIN11 migration and DNS dependency

- Set WIN11's static DNS to DC01's new AD_LAB address — required specifically because WIN11 is domain-joined (Kerberos/Group Policy depend on locating DC01 via DNS). Sequenced **after** DC01's migration was confirmed up, to avoid pointing WIN11 at a stale or nonexistent address.
- Migrated adapter to `intnet-adlab`, assigned static IP, same gateway as DC01.

**Problem encountered:** WIN11 became severely sluggish post-migration. Task Manager showed **100% CPU / 86% memory** at idle, while macOS Activity Monitor on the host showed no overload — confirming the bottleneck was WIN11's own allocation (2 vCPU/4GB, Microsoft's bare documented minimum), not the host.

**Resolution:** Removed FileZilla (unrelated leftover software from an earlier project) and bumped WIN11 to 3 vCPU/8GB RAM. Performance improved meaningfully, though not dramatically — expected given VM overhead, Windows 11's baseline weight, and constant domain-related background activity (GPO refresh, Kerberos ticket renewal) layered on top.

**Verified:** `nslookup homelab.local` resolves via DC01, domain login succeeds, `ping` to DC01's AD_LAB address succeeds.

**Problem encountered (separate, discovered afterward):** DC01 had been shut down earlier to conserve host resources, which broke WIN11's general internet access — not just domain functionality.

**Root cause:** WIN11's DNS is pointed solely at DC01 with no secondary resolver. Since **all** DNS resolution routes through DC01 (not just internal `homelab.local` lookups), losing DC01 blocks general internet browsing too, even though the underlying network path to WAN is unaffected. The symptom ("no internet") was actually "no DNS."

**Resolution / standing decision:** Treat DC01 and WIN11 as a bundled pair going forward rather than independently rotatable — WIN11 cannot be used meaningfully for domain-dependent exercises, or general browsing, with DC01 powered off.

### Step 4 — Ubuntu Desktop verification

Already migrated onto `intnet-adlab` as an incidental side effect of Entry 001's GUI-access troubleshooting. Re-verified static IP and gateway were still correctly set; no changes needed. Confirmed no requirement to match DC01's DNS, since Ubuntu Desktop isn't domain-joined.

### Step 5 — Kali migration

Bumped Kali's allocation to match WIN11 (3 vCPU/8GB, up from 2/4) for reliability, matching the same "bare-minimum isn't enough in practice" lesson from Step 3. Noted this raises the Kali+WIN11 Standard-session pairing to 9 vCPU against the host's 8 threads — one over budget, accepted as a minor, known trade-off rather than reallocating further.

Migrated adapter to `intnet-attack`, assigned static IP via GUI. DNS left blank — no domain dependency, matching Ubuntu Desktop's reasoning.

**Problem encountered:** After setting the static IP via GUI, the old IP address remained active alongside the new one even after `sudo systemctl restart NetworkManager` (the fix that resolved a similar issue in Entry 004).

**Root cause:** NetworkManager had applied the new address without releasing the old one — a lingering DHCP-style artifact.

**Resolution:** `sudo ip addr flush dev <interface>` to clear all addresses on the interface, followed by bringing the connection back up. Confirmed via `ip a` showing only the new address.

**Expected, not a problem:** `ping` from Kali to pfSense's OPT1 (ATTACK) interface failed. Confirmed via `ip route` that routing/gateway configuration was correct. Root cause: OPT interfaces in pfSense carry **zero rules by default** (unlike LAN, which gets an automatic "allow to any" rule) — this is default-deny working exactly as designed, not a misconfiguration. Decision made to leave this unaddressed until Entry 003 (firewall rules) rather than add a temporary rule now.

### Step 6 — Ubuntu Server (Splunk/DEFENDER) migration

Snapshotted before starting. Edited Netplan config: static IP, route, and empty `nameservers` (same no-DNS-dependency reasoning as Kali/Ubuntu Desktop).

**Problem encountered:** After applying, `ip route` showed the default gateway as the **network address** (`.0`) rather than pfSense's actual OPT2 interface address — a Netplan `routes: via:` typo, separate from the interface's own static address (which was correctly set).

**Resolution:** Corrected `via:` to the actual gateway IP. Clarified in the process: `via:` takes a single plain IP with no subnet suffix (subnet notation belongs on `addresses:` only); `to: default` already covers all-destinations scope.

**Problem encountered:** `curl http://localhost:8000` failed to connect. Initial troubleshooting incorrectly suspected the network/route change; actual cause was unrelated — **Splunk itself was not running** (`splunk status` confirmed `splunkd` down). Localhost traffic never leaves the host, so this was never a network/segmentation issue at all.

**Resolution:** Started Splunk as the non-root `splunk` user (consistent with Entry 016's original setup). Configured boot-start persistence with `splunk enable boot-start -user splunk` — the `-user` flag is required here specifically because this deployment runs Splunk as a dedicated non-root user, not the plain root-default boot-start command. Verified via `systemctl status Splunkd` (enabled) and a full reboot test, confirming `splunkd` came back up automatically and owned by `splunk`, not `root`.

**Verified:** `curl http://localhost:8000` succeeded post-fix.

### Step 7 — Forwarder reconfiguration (DC01, WIN11, Ubuntu Desktop)

**Problem encountered:** Attempted `splunk edit forward-server <old>:9997 -new <new>:9997` — `edit` is not a valid action for this command; only `add`, `remove`, and `list` are supported.

**Resolution:** Standardized on `list forward-server` (confirm current value) → `remove forward-server <old>` → `add forward-server <new>` → `list forward-server` (confirm change) → `restart`, applied identically across all three hosts (PowerShell syntax on DC01/WIN11, bash on Ubuntu Desktop).

**Observed:** Immediately after reconfiguration, DC01's forwarder showed **"configured but not active."** Expected at that moment, given DEFENDER (OPT2) had no allow rule yet. After a forwarder service restart, it flipped to **active** — sooner than expected.

**Root cause (significant finding):** pfSense evaluates firewall rules based on the **ingress** interface, not the destination. Traffic from DC01 (AD_LAB/LAN) enters on **LAN**, which — unlike ATTACK/DEFENDER (OPT interfaces, zero rules by default) — automatically received a default **"allow LAN net to any"** rule when it was configured. This meant AD_LAB→DEFENDER traffic was already succeeding, but via LAN's overly permissive default, not because a deliberate rule existed yet. The same default rule means **AD_LAB currently has open access to WAN and everything else** — segmentation is not yet actually enforced on traffic originating from AD_LAB.

**Implication carried into Entry 003:** the next session isn't just "add allow rules to OPT interfaces" — it must also actively **replace LAN's default permissive rule** with the specific intended rule set (AD_LAB→DEFENDER allow, AD_LAB→WAN block, default deny after). Flagged explicitly so this isn't mistaken for "segmentation already works."

Repeated the same remove/add/list/restart sequence on WIN11 and Ubuntu Desktop with the same outcome.

### Step 8 — Forwarder verification tooling

**Splunk's Forwarder Management page was not present** — expected, since Entry 016 documented Deployment Server as skipped during install, and that page is a Deployment Server feature.

Verified forwarders instead via `index=main | stats count by host` (all three present), then confirmed **currency** (not stale historical data) by re-running with a narrowed time range (last 15–60 minutes) — same three hosts still present within that window, confirming live, current reporting.

Located **Monitoring Console → Forwarder Monitoring** as the intended purpose-built view. Confirmed **Standalone mode** (not Distributed) is correct — Distributed mode governs multiple full Splunk Enterprise instances, and Universal Forwarders don't count toward that regardless of how many are connected. Left the **data collection interval at its 15-minute default**, appropriate given only 3 forwarders and no cost concern at this scale.

**Enhancement:** Added a custom panel to the existing **SOC Baseline** dashboard (Dashboard Studio) reflecting forwarder connectivity. Considered pulling from the same lookup backing the Monitoring Console view (`| inputlookup dmc_forwarder_assets.csv`), but chose a true real-time query directly against `metrics.log` instead, since it doesn't wait on the 15-minute collection interval:
```
index=_internal source=*metrics.log group=tcpin_connections | stats latest(_time) as last_seen by sourceHost, hostname
```
set to a real-time (5-minute window) time range. Noted this is one of the few cases where continuous real-time search mode is actually appropriate rather than resource-wasteful, given the small, fixed number of forwarders involved. Titled the panel **"Forwarder connectivity status (live)."**

### Step 9 — Unrelated cleanup: orphaned VM

Identified a second Ubuntu Server VM present in the environment with no memory of its original purpose and no ties to the current network configuration or any active segment. Investigated briefly (installed packages, running services, hostname) before acting rather than deleting blind; found nothing indicating it was in use or part of a planned build. Snapshotted, then removed via VirtualBox (Remove → Delete all files) to reclaim disk space. Not connected to any Phase 2 segment at any point, so no impact on the migration work above.

---

**Outcome:** All five VMs now correctly reside on their designated segments. All three Splunk forwarders successfully reconfigured and confirmed reporting, though currently via LAN's unintended permissive default rather than deliberate rule design — a gap explicitly carried forward into Entry 003. Custom real-time forwarder connectivity panel added to the SOC Baseline dashboard.

**Lesson learned:** This session repeatedly surfaced the same underlying theme — verifying assumptions rather than trusting that a setting "should" work. Vendor-minimum VM specs (WIN11, Kali) proved insufficient in practice despite meeting documented minimums; NetworkManager silently left a stale IP in place after a GUI change; a Netplan gateway typo was mistaken for a network problem when the real fault was a service that simply wasn't running; and most significantly, forwarder connectivity succeeding before firewall rules existed masked a fundamental gap — LAN's default permissive rule — that could easily have been mistaken for "segmentation is working" instead of correctly identified as "segmentation on this interface hasn't started yet." Confirming *why* something works is as important as confirming *that* it works.

**Real-world relevance:** Several patterns here mirror real enterprise operations directly: right-sizing VM/instance resources against observed load rather than vendor-stated minimums; the operational risk of a single point of DNS dependency in a domain environment; correct systemd service configuration for services intentionally run as non-root; and — most transferable to a SOC/security engineering context — the exact failure mode this session caught: assuming a security boundary is enforced because *some* interfaces are locked down, when a default or legacy rule elsewhere (LAN, in this case) can silently undermine the entire segmentation model. Catching that before Entry 003's rule-writing, rather than after, is precisely the kind of verification discipline real firewall audits are designed to catch.

---

## Phase 2, Entry 001 — pfSense Deployment: Network Segmentation Firewall/Router

**Date:** July 14, 2026
**Status:** ✅ Complete

**Action:** Deployed pfSense as a VM to enforce network segmentation across the `homelab.local` environment — splitting the previously flat network into three isolated zones (ATTACK, AD_LAB, DEFENDER) with pfSense as the only device routing between them.

**Goal:** Close the gap identified in the flat-network design: zero boundary enforcement, no firewall rules to write, no perimeter logging. This establishes the segmentation architecture the rest of the lab (Splunk correlation, Nessus scanning, GRC control mapping) builds on top of.

*This marks the start of Phase 2 — a deliberate re-architecture of the original flat-network lab into a segmented, firewall-fronted environment. Entry numbering restarts here by design: Phase 1 (see earlier entries) documents the foundational build; Phase 2 documents the evolution into an enterprise-style segmented architecture with a dedicated GRC layer. Kept in the same repo intentionally — the progression from Phase 1 to Phase 2 is itself part of the story, showing how the design matured as new gaps were identified and addressed.*

---

### Step 1 — VM creation and network adapter configuration

- Created pfSense VM (1 vCPU, 2GB RAM, 20GB disk)
- Configured 4 network adapters: Adapter 1 → NAT (WAN), Adapters 2–4 → three new VirtualBox Internal Networks (`intnet-attack`, `intnet-adlab`, `intnet-defender`)
- Snapshotted all existing VMs (DC01, WIN11, Kali, Ubuntu Desktop, Ubuntu Server) before making any changes, per standard change-management practice

### Step 2 — Disk access failure and resolution

**Issue:** pfSense VDI reported "inaccessible," VM failed to boot.

**Root cause investigation:**
- Ruled out external HDD mount timing and Virtual Media Manager stale references
- Confirmed the actual cause: external drive (exFAT format) was showing "Custom" permissions rather than Read & Write
- `chmod -R 775` initially failed with "Operation not permitted" — traced to Terminal lacking Full Disk Access
- Path also contained a space in the drive name, causing a secondary "No such file or directory" error until the path was quoted

**Resolution:** Corrected Terminal's Full Disk Access permission, quoted the drive path correctly, successfully applied read/write permissions. Given uncertainty about whether the original VDI had been corrupted during the initial failed boot attempts, performed a clean reinstall of pfSense rather than attempting to repair a potentially 0-byte disk image.

**Lesson learned:** exFAT doesn't store Unix permission bits persistently — the permission fix applies to the current mount only and may need to be reapplied if the drive is ever disconnected and reconnected under conditions that reset its default mount behavior. Documented as a known characteristic of the storage layer, not a one-time fix.

### Step 3 — Interface assignment

- Assigned WAN, LAN, OPT1, and OPT2 at the console following the clean reinstall
- Initial assignment briefly mismatched em1 against the intended AD_LAB segment; corrected by verifying actual VirtualBox adapter-to-segment mapping directly rather than assuming slot order, before finalizing assignment
- Confirmed WAN interface (internet-facing detection) resolved correctly during install

### Step 4 — IP addressing (via web GUI)

Console/wizard only configures WAN and LAN; OPT interfaces require the web GUI. Initial GUI access required temporarily migrating Ubuntu Desktop onto `intnet-adlab` early (originally scheduled for Session 2) with a manually configured static IP, since Internal Networks don't bridge to the host by default.

Configured:

| Interface | Segment | IPv4 |
|---|---|---|
| LAN | AD_LAB | Static (private /24) |
| OPT1 | ATTACK | Static (private /24) |
| OPT2 | DEFENDER | Static (private /24) |

All 4 interfaces (including WAN) confirmed **up** with correct addressing in Status → Interfaces.

### Step 5 — Admin account hardening

- Created a new administrative user, added to the `admins` group
- Verified GUI login and full access under the new account before making any further changes
- Verified console/VGA access also worked correctly under the new account (a documented pfSense limitation exists where non-default admin accounts can sometimes be restricted to a limited console shell — tested explicitly to rule this out)
- Disabled (not deleted) the default `admin` account, retaining it as a recovery fallback

### Step 6 — IPv6 cleanup (LAN rename blocker)

**Issue:** Attempting to rename the LAN interface description (cosmetic change to reflect "AD_LAB") failed with: *"DHCPv6 Server is active on this interface and can only be used with a static IPv6 configuration."*

**Root cause:** DHCPv6 Server and Router Advertisements were both enabled by default on LAN, despite this lab being IPv4-only by design.

**Resolution:** Disabled DHCPv6 Server and Router Advertisements on LAN (and proactively on OPT1/OPT2, to prevent the same error when those interfaces are touched later). LAN description then renamed successfully.

---

**Outcome:** pfSense fully deployed with all 4 interfaces up and correctly addressed, hardened admin access in place, and IPv6 services disabled lab-wide to match the IPv4-only design. ATTACK, AD_LAB, and DEFENDER segments now exist as isolated Internal Networks with pfSense as the sole routing point between them. Ubuntu Desktop is incidentally already migrated onto its target segment as a side effect of GUI-access troubleshooting.

**Lesson learned:** A session scoped as "install and configure pfSense" surfaced five distinct unplanned issues (storage permissions/corruption, interface assignment ambiguity, GUI reachability, admin account console-access risk, and IPv6 service defaults). None of these were scope creep — each was a legitimate configuration decision or failure mode that a from-scratch firewall deployment predictably produces. Budgeting real troubleshooting time into an "implementation" session, rather than treating the scripted task list as the complete scope, is the realistic expectation going forward.

**Real-world relevance:** Every one of these issues maps directly to real firewall deployment work: storage/permissions failures during provisioning, interface-to-physical-port mapping verification, out-of-band management access design, admin account privilege verification before decommissioning a default credential, and disabling unused protocol services (IPv6) as a hardening step consistent with least-privilege / reduced-attack-surface principles (CIS Control 4).


---

# Phase 1 — Foundational Build (Historical)

## Entry 016 — Splunk Enterprise SIEM Deployment

**Date:** July 11, 2026
**Status:** ✅ Complete

**Action:** Deployed Splunk Enterprise 10.4.1 on the defender Ubuntu Server and configured Universal Forwarders on DC01, WIN11, and Ubuntu Desktop to centralize log collection across the `homelab.local` environment.

**Goal:** Establish a working SIEM to support log analysis and baseline "normal" authentication/process activity, in support of SOC Analyst Tier 1 skill-building.

---

### Splunk Enterprise install (defender Ubuntu Server)

Downloaded Splunk Enterprise 10.4.1 using the signed wget command provided directly on the splunk.com download page (a time-limited, tokenized URL — works directly via `wget` on a headless server, no browser session required). Installed via `dpkg -i`.

**Problem encountered:** Splunk 10.4 no longer allows running as the root user by default. Initial start used the `--run-as-root` flag to get past this quickly, which worked but is not the recommended configuration.

**Resolution:** Stopped the instance, created a dedicated non-root `splunk` user, transferred ownership of `/opt/splunk` to that user (`chown -R splunk:splunk`), and restarted explicitly as that user (`sudo -u splunk /opt/splunk/bin/splunk start`). Verified via `ps -ef | grep splunkd` showing the process owned by `splunk`, and confirmed the root-deprecation warning no longer appeared on subsequent starts.

Admin Web credentials were created during the initial start prompt (separate from the `splunk` Linux system account).

Enabled receiving on port 9997 via Settings → Forwarding and receiving → Configure receiving.

**Verified:** Web UI accessible from Ubuntu Desktop at `http://<splunk-host>:8000` — no firewall issue encountered on this port specifically.

---

### DC01 (Windows Server 2025) forwarder

Installed the Universal Forwarder with: service account = Local System; Event Logs = Application, Security, System; Active Directory monitoring = enabled; Deployment Server = skipped; Receiving Indexer = `<splunk-host>:9997`.

**Problem encountered:** AD monitoring required separate authentication. Used the domain Administrator account to satisfy this. Noting for the record that this is broader access than the task needs — a dedicated service account with delegated read-only AD permissions would be the correct least-privilege choice in a production environment; used Administrator here for lab expediency.

**Problem encountered:** `Test-NetConnection -ComputerName <splunk-host> -Port 9997` from DC01 returned `False`. Root cause: `ufw` on the Splunk server was blocking port 9997 by default, despite all traffic being confined to the isolated Host-Only network. Resolved with `sudo ufw allow 9997/tcp`. Re-ran the test — returned `True`.

**Verified:** `index=main host=DC01` returning events in Search & Reporting.

**Pattern noted:** `ufw` blocks each new Splunk-related port by default. Checking `ufw status` proactively before troubleshooting further is now the standing approach for any new port introduced going forward (port 8089 is the likely next one, if remote management is used later).

---

### WIN11 forwarder

**Problem encountered:** Universal Forwarder installer returned "this account does not have admin privileges." Investigated further — traced to a broader AD structure issue:

**Root cause:** WIN11 was sitting in the default Active Directory "Computers" container, which is invisible to Group Policy Management (GPOs can only link to Organizational Units, never the default container). A Restricted Groups GPO built earlier to grant a dedicated admin account local admin rights on workstations had no way to reach WIN11 as a result.

**Resolution attempted:** Moved WIN11 into a new `Workstations` OU (via Active Directory Users and Computers) and linked the Restricted Groups GPO to it. Ran `gpupdate /force`. Privilege elevation still did not take effect on first test.

**Workaround used to proceed:** Logged into WIN11 with the domain Administrator account to complete the forwarder install rather than block progress further.

**Open item:** Restricted Groups still not confirmed working on WIN11 as of this entry. To revisit: check `gpresult /r` on WIN11 for whether the GPO is actually applying now that domain admin rights (see below) are in place; if applying but still not granting rights, check for a conflicting "Deny log on locally" entry. A non-functioning security policy left in place risks false confidence about what's actually enforced and should be fixed or removed cleanly rather than left ambiguous.

Installed forwarder (via Administrator account): Local System; Application/Security/System event logs; AD monitoring not offered (not a domain controller); Deployment Server skipped; Receiving Indexer `<splunk-host>:9997`.

**Verified:** connectivity test returned `True` immediately (port already open from the DC01 fix). `index=main host=WIN11` returning events.

---

### Ubuntu Desktop forwarder

**Problem encountered:** First attempt to configure the forwarder (`add forward-server`, `add monitor`) failed with "user splunk not found." Investigation showed the forwarder wasn't actually installed on this machine at all — `dpkg -l | grep splunk` returned nothing, and `find / -iname "splunkforwarder*.deb"` found no installer file anywhere on disk. The install had never actually completed here; each machine requires its own independent download.

**Resolution:** Re-downloaded via the same signed wget pattern used on the indexer, run directly on Ubuntu Desktop.

**Problem encountered:** Subsequent `dpkg -i` failed because the command referenced an assumed filename rather than the file that actually downloaded. Resolved by running `ls *.deb` to confirm the real filename before installing.

Created a dedicated `splunk` user on this machine (local OS accounts don't carry over between VMs — this had to be repeated fresh, separate from the indexer's `splunk` user), transferred ownership of `/opt/splunkforwarder`, and started the forwarder as that user.

**Problem encountered:** `enable boot-start -user splunk` failed with "permission denied" when run as `sudo -u splunk`. Root cause: writing the systemd service file to `/etc/systemd/system/` requires root, regardless of `/opt/splunkforwarder` ownership. Resolved by running this one specific command as plain `sudo` (not `sudo -u splunk`) while still passing `-user splunk` — the one legitimate exception to the "always run as splunk" rule in this deployment, since it modifies system-level startup configuration rather than the Splunk process itself. A follow-up "command not found" on retry was traced to a typo, not a real issue.

**Problem encountered:** Expected log paths `/var/log/syslog` and `/var/log/auth.log` did not populate as expected. Checked for `ufw` to rule out a firewall block — found `ufw` isn't installed at all on this Ubuntu Desktop image (unlike Ubuntu Server), so no firewall rules were needed for outbound connectivity. Continued investigating — the file/path naming turned out to differ from what was assumed, and the actual reporting hostname for this VM is `unumtu-desktop` (apparent typo from original VM naming, confirmed as the literal hostname rather than a transcription error). Once the correct host was identified, the already-registered `add monitor` config picked up the log data without needing to be re-added.

**Verified:** `index=main host=unumtu-desktop` returning events.

---

### Domain-wide audit policy (GPO)

Given AD DS was already in place, chose to enable richer auditing centrally via GPO rather than local policy per machine. Edited the Default Domain Policy → Computer Configuration → Windows Settings → Security Settings → Advanced Audit Policy Configuration, enabling:
- Logon/Logoff → Audit Logon (Success and Failure)
- Detailed Tracking → Audit Process Creation (Success)
- Account Logon → Audit Kerberos Authentication Service (Success and Failure)

**Problem encountered:** "Computer Configuration" did not appear at all in the Group Policy Management Editor on first attempt. `gpresult /r /scope:computer` on DC01 showed "N/A" for Applied Group Policy Objects, confirming the logged-in account lacked sufficient rights to view or edit domain-level policy.

**Root cause:** the account being used was not a member of Domain Admins.

**Resolution:** Added the dedicated admin account to Domain Admins via Active Directory Users and Computers, then logged off and back on (group membership changes require a fresh logon, not just `gpupdate`). Confirmed via `whoami /groups` showing `HOMELAB\Domain Admins`. Computer Configuration was then fully visible and editable.

**Note on scope:** chose full Domain Admins since this account is intended as an ongoing admin account, not a one-time fix. For read-only GPO result viewing specifically, a delegated "Generate Resultant Set of Policy (Logging)" right would have been the stricter least-privilege option — noted for future reference.

This also resolves the open item from the WIN11 forwarder install above — the Restricted Groups GPO configuration issue likely traced to the same underlying rights gap. Worth re-testing now that Domain Admin rights are in place.

Ran `gpupdate /force` on DC01 and WIN11 following the policy change.

**Verified:** `index=main EventCode=4688` (Process Creation, previously empty) and `index=main EventCode=4624 host=DC01` both returned events after the policy applied.

---

### Verification and dashboards

Ran `index=main | stats count by host` — confirmed `DC01`, `WIN11`, and `unumtu-desktop` all reporting.

Built a Dashboard Studio dashboard (`SOC Baseline`, Grid layout) with three panels:
1. Domain authentication activity — `index=main (EventCode=4624 OR EventCode=4625) host=DC01`
2. Process creation across endpoints — `index=main EventCode=4688 | table _time, host, Account_Name, New_Process_Name | sort -_time` (table)
3. Linux auth activity — `index=main host=unumtu-desktop ("Failed password" OR "session opened") | table _time, host, _raw | sort -_time` (table)

**Outcome:** Full Splunk SIEM deployment complete. Splunk Enterprise 10.4.1 running as a non-root user on the defender Ubuntu Server. Three Universal Forwarders (DC01, WIN11, Ubuntu Desktop) confirmed sending data. Domain-wide audit policy enabled via GPO, enriching Windows event data with logon, process creation, and Kerberos detail. One working baseline dashboard built with three panels covering domain authentication, process creation, and Linux authentication activity.

**Lesson learned:** this build surfaced far more troubleshooting than a clean install would have, and nearly all of it maps onto real enterprise patterns rather than lab-only quirks: Splunk 10.4's root-execution deprecation and its one systemd-related exception; `ufw` silently blocking new ports by default on Ubuntu Server while being absent entirely on Ubuntu Desktop; new AD-joined machines landing in the default Computers container and staying invisible to Group Policy Management until moved to a proper OU; local admin rights and Domain Admin rights being distinct, non-interchangeable requirements, with group membership changes needing a fresh logon rather than a policy refresh; each machine requiring a fully independent install and non-root user setup with nothing carrying over between hosts; and assumed standard Linux log paths and hostnames not holding up without direct verification. Nearly every step that "should have just worked" surfaced a real, documentable gotcha.

**Real-world relevance:** Centralizing log collection from a domain controller and its member workstations mirrors how enterprise SOCs ingest Windows Event Logs at scale — DC-level authentication logs are typically the highest-value source in any AD environment, since Kerberos ticket activity touches every domain resource access. The troubleshooting throughout this entry — non-root service accounts, firewall rule management, AD OU/GPO structure, and least-privilege account design — reflects the actual day-to-day skill set of a SOC/security engineer working in a real Windows/AD enterprise environment, arguably more so than the SIEM configuration itself.

---

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
*Log continues as the lab grows. Every new configuration, exercise, troubleshooting event, and rebuild is documented here. New entries are added at the top of the current phase's section.*
