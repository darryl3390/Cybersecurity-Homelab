# Homelab Build Log

*Ordered newest first. New entries are added at the top of their phase's section going forward.*
**Owner:** Darryl Briggs  
**Goal:** Build a functioning enterprise-style network to develop hands-on cybersecurity and IT skills in support of CompTIA certifications and a career transition into GRC and cybersecurity engineering.  
**Host:** iMac 2017 — Intel Core i7 — 48GB RAM — VirtualBox  
**Domain:** homelab.local  
**Architecture (current — Phase 2):** Segmented network behind pfSense — ATTACK, AD_LAB, and DEFENDER zones as isolated VirtualBox Internal Networks, with pfSense as the sole routing/firewall point between them. Splunk (SIEM) and Eramba (GRC) run in the DEFENDER segment. See Phase 2 entries below for the full build; Phase 1 entries document the original flat-network foundation this was built on top of.

---

# Phase 2 — Segmented Architecture

## Phase 2, Entry 008 — Session 8: Wazuh-to-Splunk Integration, Actually Verified and Fixed

**Date:** August 27, 2026
**Status:** ✅ Complete — Wazuh alert data confirmed flowing into Splunk

**Action:** Returned to verify whether Wazuh's findings were genuinely reaching Splunk, following Entry 007's premature "confirmed working" claim. They weren't. Traced the real cause through a full chain of individually-ruled-out possibilities to the actual, final root cause.

**Goal:** Get real, verified Wazuh alert data landing in Splunk — not just configuration commands that ran without error.

---

### The diagnostic chain, in order, each step ruling out one real possibility

1. **Checked whether the forward-server destination was even active:**
   ```bash
   sudo /opt/splunkforwarder/bin/splunk list forward-server
   ```
   Showed `Active forwards: 10.10.30.10:9997` — genuinely active. Ruled out: destination not configured.

2. **Searched Splunk for any Wazuh-sourced data** — nothing found, confirming the actual symptom was real, not a search-syntax issue.

3. **Noticed Wazuh's own API showing "Offline"** on the dashboard's API Connections page — investigated as a possible cause. Confirmed via `systemctl status wazuh-manager` and a direct `alerts.json` timestamp check that the **core manager was healthy and actively writing real alerts** — the API being down was a separate, unrelated issue (affects dashboard config actions, not alert generation or the file itself). Ruled out.

4. **Checked Splunk's receiving side directly:**
   ```bash
   sudo ss -tulnp | grep 9997
   ```
   Confirmed `splunkd` genuinely listening on `0.0.0.0:9997`. Ruled out: receiving side not listening.

5. **Tested the raw network path directly, bypassing both Splunk and Wazuh's own reporting:**
   ```bash
   nc -zv 10.10.30.10 9997
   ```
   Succeeded. Also checked the Wazuh manager VM's own `iptables -L OUTPUT` — default `ACCEPT`, no blocking rules. Ruled out: network/firewall path.

6. **Checked the forwarder's actual `outputs.conf` on disk** (not trusting the CLI's report of it) — genuinely correct, `server = 10.10.30.10:9997` present and accurate.

7. **Checked the forwarder's `inputs.conf` on disk — found the real root cause:** the file **did not exist at all.** The earlier `add monitor` command (Entry 007, after fixing the shell-quoting bug) had appeared to succeed with no error, but never actually persisted a monitor definition to disk. The forwarder had a correct destination and nothing whatsoever telling it to watch `alerts.json`.

### The fix

Created the file directly rather than trusting the CLI a second time:
```bash
sudo mkdir -p /opt/splunkforwarder/etc/system/local
sudo nano /opt/splunkforwarder/etc/system/local/inputs.conf
```
```ini
[monitor:///var/ossec/logs/alerts/alerts.json]
disabled = false
sourcetype = wazuh_alerts
index = main
```
Restarted the forwarder, then verified with real data in Splunk:
```
index=main sourcetype=wazuh_alerts
```
Confirmed: real Wazuh alert data now populating Splunk.

---

**Outcome:** Wazuh's own findings are now genuinely, verifiably unified into Splunk — not assumed, actually confirmed with real returned events. Six independent possibilities were checked and ruled out in sequence (destination config, manager health, receiving port, network path, output config) before the actual cause (a silently-failed monitor write) was found.

**Lesson learned:** A CLI command returning no error is not the same as a config change actually persisting to disk. This is the second time this exact class of gap has shown up tonight in different forms (the earlier API-timeout/YAML crash also stemmed from a config edit not behaving as expected) — when a tool's own reporting can't be fully trusted, checking the actual file on disk is the more reliable ground truth. Also worth remembering: don't mark something "confirmed working" in documentation until the actual downstream result has been checked, not just the command that was supposed to produce it — Entry 007's premature claim here is exactly the kind of gap this whole build log has otherwise been careful to catch.

**Real-world relevance:** This is precisely what real troubleshooting under uncertainty looks like — a genuine chain of elimination across every layer of a data pipeline (source, transport, network, destination), rather than guessing at the first plausible cause. The specific failure mode (a CLI reporting success while silently not writing its config) is a real, transferable lesson for working with any tool whose state you can't fully verify without checking the underlying files directly.

---

## Phase 2, Entry 007 — Session 7: Wazuh EDR Deployment Across the Environment

**Date:** August 24, 2026
**Status:** ✅ Complete (Wazuh manager, four agents, Sysmon, auditd, centralized config) — ⬜ Deferred (Metasploitable2 exploitation test, DC01-specific monitoring group, Wazuh-to-Splunk integration verification — see Entry 008)

**Action:** Deployed Wazuh as a full EDR layer across the environment — a dedicated manager, agents on all four existing endpoints (both Linux hosts, both Windows hosts), Sysmon on the Windows side, auditd on the Linux side, and centralized configuration via Wazuh groups. A Splunk Universal Forwarder was also configured on the Wazuh manager to feed its findings back into the SIEM, though this integration wasn't actually verified working until Entry 008.

**Goal:** Move beyond network-layer detection (Splunk + pfSense) into genuine host-level EDR — real process, file integrity, and syscall visibility on every endpoint, correlated through Wazuh and unified back into Splunk as the single pane of glass.

---

### Step 1 — Wazuh manager: new dedicated VM, DEFENDER segment

Built a fresh Ubuntu Server VM (8GB RAM, 50GB disk intended) specifically for Wazuh, kept separate from Splunk's existing host given Wazuh's real resource footprint (indexer + server + dashboard together).

**Real install friction, in order:**
- Initial install attempt used a literal `4.x` in the install script URL — corrected to the actual current version path (`4.14`) after verifying against Wazuh's own documentation
- A `-O` vs `-0` typo (letter O vs. zero) on the `curl` command caused the install script to print to the terminal instead of saving to a file — corrected
- Post-install, `systemctl status wazuh.service` / `dashboard.service` both returned "could not be found" — real services were running under different unit names (`wazuh-manager.service`, `wazuh-dashboard.service`), confirmed via `systemctl list-units | grep wazuh`, not an actual failure

### Step 2 — Real disk sizing bug found and fixed

The dashboard began throwing `TOO_MANY_REQUESTS / disk usage exceeded flood-stage watermark` errors — OpenSearch's own safety mechanism locking indices read-only under disk pressure. `df -h` confirmed the root cause: the VM's logical volume was only **29GB**, not the 50GB intended at creation — a real, known Ubuntu installer quirk where LVM under-allocates relative to the actual virtual disk. Fixed via `lvextend -l +100%FREE` and `resize2fs`, recovering the full disk without rebuilding the VM.

### Step 3 — Firewall rules for Wazuh, added deliberately and scoped

Three new rules, `AD_LAB → DEFENDER`, matching the same least-privilege pattern as every other rule in this build:
- TCP 443 — dashboard access
- TCP+UDP 1514 — agent-to-manager communication
- TCP 1515 — agent enrollment

All confirmed against Wazuh's own current documentation before adding, rather than assumed from memory.

### Step 4 — Ubuntu Desktop agent: a real, deliberate proxy policy exception

Attempting to install the agent via `apt` failed with `403 CONNECT denied` — traced to a decision made back in Session 5, where apt-cacher-ng's HTTP tunneling was deliberately disabled to prevent proxy-based firewall bypass. Rather than reverting that decision broadly, added a single scoped exception in apt-cacher-ng's config (`PassThroughPattern: ^packages\.wazuh\.com:443$`), preserving the original security intent while permitting this one legitimate case.

### Step 5 — Splunk VM agent: no relay needed

Since Splunk's VM already sits on DEFENDER with its own real WAN access (Session 3), the agent installed directly, no proxy or relay required — the one genuinely simple deployment of the whole session.

### Step 6 — auditd on both Linux hosts, and a corrected assumption

Initially assumed Wazuh auto-configures auditd rules on connection — incorrect. Confirmed via research that Wazuh and auditd are two independent systems: Wazuh reads whatever auditd is told to watch, but does not define those rules itself. Corrected by manually writing a real rule set (identity file watches, `execve` tracking) directly via `auditctl`/`augenrules`.

### Step 7 — Centralized configuration, and a real manager crash along the way

Built a `linux` Wazuh group to centrally push the auditd `<localfile>` config to both Linux agents, rather than hand-editing `ossec.conf` per host.

**A real, serious problem surfaced attempting this:** dashboard config saves began failing with API timeouts. Diagnosed via `api.log`, which showed the *save itself* succeeding, but a separate automatic follow-up call (`/manager/configuration/validation`) consistently timing out at 10 seconds. Attempted to raise the API's `request_timeout` — this **crashed the Wazuh manager entirely**, traced via `journalctl` to a YAML syntax error: the new `request_timeout: 30` value had been added under a still-commented-out parent key (`# intervals:`), producing an orphaned, invalid config the API refused to start against. Corrected by uncommenting both the parent and child keys together; manager restarted cleanly with the longer timeout active.

### Step 8 — DC01 agent and Sysmon: the most friction of any single endpoint tonight

- Netcat-based relay (the method used all night for Linux file transfers) doesn't work for Windows, which has no built-in `nc` — switched to a VirtualBox Shared Folder instead, a cleaner, network-independent method
- Running the agent MSI without the `/q` silent flag opened a manual GUI enrollment tool instead of installing normally — corrected by using the proper silent install command
- The manager address landed as `0.0.0.0` post-install rather than the real IP — same class of bug seen earlier on Ubuntu Desktop — fixed via direct `ossec.conf` edit
- Attempted `Enable-WindowsOptionalFeature -FeatureName Sysmon` to install Sysmon — this is not a real Windows feature; the command "succeeded" while installing nothing at all
- The actual Sysmon executable had never been downloaded, only the config file — corrected
- The SwiftOnSecurity config download turned out to be a corrupted HTML error page saved with an `.xml` extension, not real config content — caught via Sysmon's own parser error, corrected by pulling the raw file directly from GitHub rather than a rendered page

### Step 9 — "windows" Wazuh group, DC01 added, config confirmed pushed

Created deliberately parallel to the `linux` group, holding the shared Sysmon `<localfile>` config both Windows hosts would need.

### Step 10 — WIN11 agent and Sysmon: same process, genuinely faster

Every failure mode from DC01 was already known — silent install flag used from the start, manager address verified immediately, Sysmon config file-content verified (`head -5`) before transfer rather than after a failure. Added to the `windows` group; centralized config confirmed pushed correctly.

### Step 11 — Wazuh-to-Splunk integration: closing the loop

Realized mid-session that Wazuh's own findings were not actually flowing into Splunk, despite an earlier assumption that they were — corrected directly rather than left standing. Installed Splunk Universal Forwarder on the Wazuh manager VM itself (not an `apt` package — required a direct, account-gated download from Splunk, unlike Wazuh's own repository-based install).

**Real troubleshooting to get it working:**
- `add monitor` initially failed with a misleading "event not found" error against `/var/ossec/logs/alerts/alerts.json`
- First suspected and ruled out: wrong Splunk credentials, then group-membership/permissions (partially correct — the forwarder's user did need adding to the `wazuh` group to read the file at all)
- The actual final root cause: special characters in the Splunk password being mishandled by the shell before ever reaching the `add monitor` command — resolved by single-quoting the `-auth` argument
- Configuration commands completed without error — `add monitor`, `add forward-server` (10.10.30.10:9997), `splunk restart` — but **actual data flow was never verified before ending the session.** This turned out to matter: see Entry 008 for what was actually still broken.

---

**Outcome:** Full EDR coverage now exists across all four endpoints — Sysmon on the Windows hosts, auditd on the Linux hosts, centralized configuration management proven working across two groups, and Wazuh's own analysis now unified into Splunk rather than sitting in a separate, disconnected dashboard. The manager survived a real crash (a self-inflicted YAML error) and a real disk-sizing bug, both fully diagnosed and resolved rather than patched around.

**Lesson learned:** Several of tonight's hardest problems were caused by fixes for *other* problems — the manager crash came from fixing the API timeout; the "event not found" error looked like a permissions issue and partially was, but the real final cause was a shell-quoting problem underneath it. Layered debugging like this rewards checking one variable at a time rather than changing several things and hoping — nearly every dead end tonight resolved once isolated properly rather than guessed at.

**Real-world relevance:** This is what real EDR rollout actually looks like at small scale — not a single clean deployment, but per-platform quirks (Windows vs. Linux transfer methods, install flags, config validation), a genuine infrastructure bug in the manager itself, and the often-overlooked step of actually unifying a new tool's output with existing detection infrastructure rather than letting it become one more disconnected dashboard.

---

## Phase 2, Entry 006 — Session 6: Log Forwarding, Host Firewall Regression, and DISA STIG Automation

**Date:** August 23–24, 2026
**Status:** ✅ Complete (log forwarding, Ubuntu STIG automation) — ⬜ Deferred (Windows STIG auditing)

**Action:** Configured pfSense-to-Splunk log forwarding, caught and resolved a real host-level firewall regression on the Splunk VM discovered in the process, then pursued automated DISA STIG compliance scanning against Ubuntu Server — working through a chain of real, independent tooling issues to reach a genuine, defensible conclusion.

**Goal:** Eliminate the manual cross-checking between pfSense's own log viewer and Splunk by forwarding firewall events directly into the SIEM, then move DISA STIG auditing from a slow manual process into an automated one.

---

### Step 1 — pfSense → Splunk log forwarding

Enabled Remote Logging on pfSense (Status → System Logs → Settings), pointed at Splunk's IP on a UDP data input. Straightforward to configure, but exposed a real, independent problem the moment testing began.

### Step 2 — Problem encountered: host firewall regression on the Splunk VM

**Root cause:** at some earlier point, removing `ufw` on this VM left orphaned-but-still-functional iptables chains behind — real traffic counters proved they'd been active — while silently reverting the effective policy to `ACCEPT` by default. The one existing rule (permitting port 6514) was the only thing standing between this host and a fully open `INPUT` chain.

**Resolution, in order:**
1. Flushed all leftover ufw chains (`iptables -F`, `-X`) rather than append new rules on top of stale scaffolding
2. Deliberately declined to add SSH back in — confirmed this VM has only ever been managed via VirtualBox console, so reopening port 22 would have been unnecessary standing attack surface, not a real requirement
3. Rebuilt explicit allow rules (established/related, loopback, Splunk web UI, forwarder port, syslog port) *before* flipping the default policy to `DROP` — same anti-lockout discipline as pfSense's own built-in protection
4. **Second problem encountered:** after the rebuild, syslog forwarding broke again — root cause was a protocol mismatch, the rebuilt port 6514 rule was accidentally created as TCP instead of UDP, silently failing to match pfSense's actual traffic
5. Corrected to UDP, verified real traffic arriving, saved the ruleset persistently (`netfilter-persistent save`)

### Step 3 — DISA STIG manual review, then a decision to automate

Installed STIG Viewer 3 (now an Electron application, not Java-based) via the established Mac → Kali → netcat relay pattern, since AD_LAB has no path to `public.cyber.mil`. Hit and resolved two independent issues along the way: a corrupted transfer requiring a retry, and an Electron sandbox failure inside the VM resolved with `--no-sandbox`. Imported the Ubuntu STIG benchmark and began manual review — with 200+ individual findings, decided a manual pass alone wasn't the efficient path forward and pursued automation instead.

### Step 4 — SCC attempted, correctly abandoned

Installed DISA's official SCC tool (5.14.1) via the same relay method. Discovered no Ubuntu-specific SCAP content is published for SCC's ecosystem (its published content is overwhelmingly RHEL/SLES/Oracle-Linux focused). Cleanly uninstalled rather than force a mismatched tool onto the task.

### Step 5 — OpenSCAP: a real, multi-layered troubleshooting chain

**Problem 1:** `apt install` reported packages as unable to locate. **Root cause:** the `universe` repository wasn't enabled on this VM. Enabled it, resolved.

**Problem 2:** first scan attempt failed outright — `oscap` couldn't find its expected default CPE dictionary file. **Root cause:** confirmed as a long-documented, real OpenSCAP packaging gap (tracked publicly for years, still recurring in current reports) — the package simply doesn't ship this file. Fixed by creating a minimal, schema-correct placeholder, cross-checked against OpenSCAP's actual real source file structure rather than guessed blind.

**Problem 3:** the scan then completed but scored 0% with "no rules evaluated." **Root cause:** the placeholder CPE dictionary was too minimal to support the platform-applicability check `oscap` runs before testing anything. Fixed by pointing the scan explicitly at the real, content-specific CPE dictionary shipped alongside the Ubuntu datastream itself (`--cpe` flag), rather than relying on the generic default path.

**Problem 4:** the scan then completed but marked every single rule "Not Applicable." **Root cause, verified through direct research rather than assumption:** the host runs Ubuntu 26.04, but no SCAP content exists for that release anywhere — confirmed by checking the SSG project's own release history, Ubuntu's package archives, and independent current guides, none of which show anything past 24.04. Investigated further and found a genuine, structural reason: Ubuntu 26.04 appears to be a short-support interim release, not an LTS — compliance-content projects deliberately prioritize LTS releases, since those are what's actually deployed in production for years. This is an external scope gap, not a temporary omission.

**Resolution:** ran the scan using the 24.04 datastream as a deliberate, documented substitute (one LTS cycle removed, same core architecture), rather than continuing to chase a fix for content that doesn't exist. A same-version `os-release` edit workaround was also attempted and did not resolve the issue, consistent with this being a genuine content gap rather than a version-string detection problem.

### Step 6 — Report access

Hit two final, minor issues viewing the completed report: files owned by `root` from the `sudo`-executed scan (fixed via `chown`/`chmod` on both the file and its directory), then a GTK/Firefox display issue opening the HTML report (resolved by using `w3m`, a text-based browser, instead).

---

**Outcome:** Splunk now receives pfSense's firewall events directly, closing the manual cross-checking gap from Session 4. A real, previously-unknown host firewall regression on the Splunk VM was found and corrected — arguably the most important single fix of this session, since it had left one of the lab's most sensitive hosts silently unprotected at the network-adjacent layer. DISA STIG auditing against Ubuntu Server is functioning end-to-end, automated, with a clearly documented methodology note about the 24.04-for-26.04 substitution and why it's necessary. Windows-side STIG auditing (WIN11, DC01) remains outstanding — deferred to a future session.

**Lesson learned:** Nearly every problem this session surfaced was a real, independent issue, not one root cause wearing different masks — a stale package removal silently weakening a firewall, a missing file in a mature open-source tool, an under-specified default breaking a downstream check, and a genuine content-availability gap tied to a release's support tier. Treating each as its own problem, verified rather than assumed, was what actually closed all of them out rather than compounding a wrong guess. Worth remembering: a plausible-sounding explanation for a failure ("edit the version string") isn't the same as the actual root cause, and testing that gap honestly (checking whether the fix worked, not assuming it should have) mattered more than any individual fix.

**Real-world relevance:** Discovering a host firewall silently reverted to permissive-by-default, well after the fact, is precisely the kind of gap real security audits exist to catch — and catching it here came from the same instinct (verify, don't assume) that any real compliance or SOC review depends on. The DISA STIG automation chain, meanwhile, is a realistic picture of what real compliance tooling work actually involves: not a single clean command, but a sequence of genuine environment-specific issues, each requiring its own diagnosis, each resolved on its own evidence rather than a guess.

---

## Phase 2, Entry 005 — Session 5: Eramba Deployment, Credentialed Scanning, and Controlled-Egress Proxy

**Date:** July 16–17, 2026
**Status:** ✅ Complete

**Action:** Deployed Eramba (Community Edition, via Docker) as the lab's GRC platform, hardened its access, then worked through a credentialed Nessus scan across the AD_LAB segment — which surfaced a real architectural gap (no controlled internet access for AD_LAB hosts) and led to standing up a dedicated internal package-caching proxy as the enterprise-pattern fix. Closed the session by triaging real scan findings into Eramba's risk register.

**Goal:** Complete the last piece of the original pfSense implementation project (Eramba), then use real vulnerability data — not just unauthenticated scan results — to populate the risk register with genuine findings.

---

### Step 1 — Eramba VM and Docker deployment

Created a dedicated Ubuntu Server VM for Eramba (2 vCPU/8GB, matching Eramba's own published spec), placed on `intnet-defender`, static IP with DNS pointed at pfSense's own resolver — same centralized-DNS pattern as every other DEFENDER host.

Chose Docker deployment over Eramba's pre-built OVF appliance after weighing both explicitly: the OVF ships with default OS credentials and an unknown, unverifiable hardening baseline — directly against the "verify everything, trust no default" discipline applied everywhere else in this build. Docker costs more setup time but keeps the OS layer self-built and self-known.

**Problem encountered:** `apt-key`/`add-apt-repository` (used to add Docker's own repo) produced deprecation warnings — `apt-key` and `trusted.gpg` are both deprecated, and store all trusted keys in one shared file trusted for every repository on the system, not just Docker's.

**Resolution:** Replaced with the modern per-repository keyring method — deleted the old key, generated a new one into `/usr/share/keyrings/docker.gpg`, and rewrote the repo's source line to reference it explicitly via `signed-by=`. A leftover source-list entry without the `signed-by=` reference caused a follow-up "public key unavailable" error, traced via `grep -rn "download.docker.com" /etc/apt/` and corrected in place.

Cloned Eramba's official Docker repo, set real (non-default) DB and app passwords in `.env`, declined the HTTP-tunneling prompt during `apt-cacher-ng`'s later install for the same reason — tunneling bypasses firewall restrictions by design, which isn't needed for simple package caching and works against the segmentation model. Ran `docker compose -f docker-compose.simple-install.yml up -d` — all four containers (MySQL, Redis, app, Cron) came up clean.

### Step 2 — GUI access and admin hardening

Same access gap as Splunk's dashboard in Session 3: Ubuntu Desktop (the only VM with a browser) sits on AD_LAB, not DEFENDER. Added one more explicit, scoped LAN rule — `AD_LAB → DEFENDER, TCP 8443` — following the same pattern already used for Splunk's port 8000.

Completed Eramba's first-run setup; confirmed the superadmin username is fixed by the wizard (not a real gap — password is what matters, and a strong, unique one was set). Created a second, named admin account, verified its access, then deactivated (not deleted) the default — identical pattern to pfSense's Entry 001 hardening.

### Step 3 — Credentialed Nessus scan: the AD_LAB internet gap surfaces

Attempting a credentialed scan required getting Nessus's SSH credential working against Ubuntu Desktop — which needed `openssh-server` installed. AD_LAB's Session 3 WAN block (correctly) prevented a direct `apt install`. Also discovered Kali itself has no standing WAN path either (no ATTACK→WAN rule exists) — ruling out the "download on Kali, relay to Ubuntu Desktop" workaround used earlier for `ufw`.

**Decision — build a controlled-egress proxy instead of opening a rule.** Considered a standing ATTACK→WAN rule (same shape as DEFENDER's outbound access) and rejected it: DEFENDER's outbound need is continuous and low-risk; ATTACK's is occasional, and ATTACK specifically carries more risk as the segment running untrusted tooling. Instead, stood up **apt-cacher-ng** on its own new, dedicated VM (not shared with Splunk) — a deliberate least-functionality decision, keeping the SIEM host from also carrying an unrelated proxy service. Verified this matches real enterprise practice: forcing internal hosts through a single controlled proxy for package management, rather than direct internet access, is standard in secured production environments.

Installed cleanly, declined HTTP tunneling as noted above, added one new LAN rule (`AD_LAB → DEFENDER, TCP 3142`), pointed Ubuntu Desktop's `apt` config at the proxy. Verified end-to-end with both a fresh install (`openssh-server`) and a full `apt upgrade`/`full-upgrade` — both pulled cleanly through the proxy with zero direct WAN access from AD_LAB.

### Step 4 — Credentialed scan authentication troubleshooting

**WIN11 auth failure:** traced to UAC Remote Restrictions (`LocalAccountTokenFilterPolicy`) — a documented Windows behavior where domain accounts authenticate fine on a DC but are blocked from remote administrative actions on a regular domain-joined workstation unless this registry key is set. Corrected via registry edit and reboot.

**Ubuntu Desktop auth failure:** resolved once `openssh-server` was actually installed and running (Step 3) — no separate SSH-side misconfiguration once the service existed.

**Problem encountered (real, reproducible Nessus quirk):** even after both hosts authenticated successfully, the scan's summary host-list **Auth column** continued to show "Fail" for both WIN11 and Ubuntu Desktop, while the detailed **"Target Credential Status by Authentication Protocol"** plugin output explicitly confirmed "Valid Credentials Provided" and "no privilege or access problems" / "sufficient privileges for all planned checks" for both hosts. Confirmed reproducible across two independently completed scan runs (a third was intentionally allowed to finish as a final check, matching the two prior results).

**Resolution:** treated the detailed per-protocol plugin output as authoritative over the summary column, given its explicit specificity. Documented as a known Nessus UI/summary discrepancy rather than a real auth problem — not pursued further once the underlying scan data was confirmed sound and consistent across repeated runs.

### Step 5 — Real findings triaged into Eramba's risk register

From the completed credentialed scan (DC01: 232 findings, WIN11: 78, Ubuntu Desktop: 27):

1. **WinVerifyTrust Authenticode validation not enforced (CVE-2013-3900)** — WIN11, High/8.8 CVSS. Treatment: remediate via the documented registry-based strict-validation fix.
2. **Windows Defender signatures stale (>3 days)** — DC01, High. Root cause traced directly to the lab's own segmentation: DC01/AD_LAB has no WAN access, so Defender cannot reach Microsoft's update servers. Logged as a new item in the Remediation Tracker (#6) — proposed fix mirrors the same controlled-proxy pattern just built for Linux (a scoped temporary AD_LAB→WAN rule during patch windows, or a WSUS-style internal update server on DEFENDER).
3. **pfSense LAN interface self-signed certificate** — Medium/6.5, both SSL-related Nessus plugin findings tied to the same single certificate. Treatment: **accept** — internal-only management interface on an isolated network; cost of a real certificate or internal CA isn't justified by the actual risk.

All three entered into Eramba with framework mapping (NIST CSF categories) and status.

### Step 6 — Deferred: Eramba/Splunk REST API integration

Investigated bi-directional webhook integration (Splunk detections auto-creating Eramba incidents; Eramba control/risk changes triggering Splunk events). Verified against Eramba's own official documentation rather than a secondhand summary — confirmed the capability is real (REST API, Swagger docs, Basic Auth over TLS, webhooks attached to Eramba's notification/dynamic-status system, not a standalone "automations" page as first described). Deferred as a future project: current data volume doesn't justify automation over manual review, it isn't required by any target certification or job posting, and it competes directly with the GRC content and remediation sprint still outstanding. Logged as an optional future-project item with full rationale.

---

**Outcome:** Eramba fully deployed, hardened, and populated with three genuine, credentialed-scan-sourced risk register entries. A real architectural gap (AD_LAB has no controlled path for security updates) was discovered as a direct byproduct of trying to run a scan, and addressed with an enterprise-pattern fix (a dedicated, least-privilege internal proxy) rather than a firewall rule that would have undermined the segmentation model. A second, parallel finding (DC01's stale Defender signatures) shows that same gap already causing a real, current security consequence — logged for future remediation rather than fixed same-session.

**Lesson learned:** Two of tonight's most valuable outcomes came from resistance, not from things going smoothly — Ubuntu Desktop's SSH install failing pushed toward building a proper controlled-egress solution instead of a one-off workaround, and DC01's stale Defender signatures turned an abstract architectural decision (AD_LAB has no WAN access) into a concrete, current finding. Friction that surfaces a real gap is more valuable than friction that's just solved and forgotten — worth documenting the "why," not just the "what," in both cases.

**Real-world relevance:** Standing up a dedicated caching/egress proxy specifically to avoid opening broad internet access from a security-conscious network segment is textbook enterprise network architecture, not a lab workaround. The WinVerifyTrust and stale-signature findings are both real, current, well-documented Windows vulnerability classes — the kind actually seen in production vulnerability management programs, not synthetic lab-only issues. And discovering that your own segmentation design has a real maintenance cost (patching becomes harder, not just attacking) is an accurate reflection of the genuine trade-off every real segmented network makes.

---

## Phase 2, Entry 004 — Session 4: Positive/Negative Detection Testing

**Date:** July 15, 2026
**Status:** ✅ Complete (core objective) — ⬜ Deferred (host-layer logging enhancement)

**Action:** Ran the first real positive and negative detection tests against the segmented architecture — confirming an attack is actually detected and a blocked path is actually enforced, rather than just configured.

**Goal:** Verify Session 3's rule set under live conditions: does ATTACK→AD_LAB traffic actually get seen, and does AD_LAB→WAN actually get stopped.

---

### Step 1 — Scoping the session's resource footprint

Considered running all six VMs simultaneously (all were already up from earlier work) and briefly treated this as an informal "stress test." On reflection, discarded that framing — an unstructured stress test with no baseline or defined metrics isn't rigorous enough to be worth pursuing deliberately, and risked confounding the actual security-verification results with unrelated performance noise. Powered off DC01 and WIN11 (not needed for this session's tests) and proceeded with a 4-VM footprint (pfSense, Splunk, Kali, Ubuntu Desktop) — sustainable without throttling, and a cleaner signal for the tests that mattered.

### Step 2 — Positive test: attack generation

Selected Ubuntu Desktop as the AD_LAB test target (no domain dependency, unlike DC01/WIN11 — avoids the Session 2 DNS single-point-of-failure entirely). Originally planned an SSH brute-force test, but SSH could not be installed on Ubuntu Desktop — AD_LAB's explicit WAN block (Entry 003) correctly prevented the package download, confirming the block rule extends beyond DNS to all outbound traffic. Switched to an Nmap scan instead, which requires no new installation on the target:
```
nmap -sV -T4 <ubuntu-desktop-ip>
```

### Step 3 — Positive test: detection gap discovered

**Problem encountered:** The scan produced no results in Splunk (`index=main host=<ubuntu-desktop-hostname>`).

**Root cause:** Ubuntu Desktop has no local firewall (`ufw`/`iptables`) generating connection-attempt logs, and nothing meaningful was listening on the scanned ports. With no local log source, the Universal Forwarder had nothing to forward — this is a genuine host-layer visibility gap, not a forwarder or Splunk misconfiguration.

**Resolution — checked network-layer detection instead:** pfSense's own firewall log (Status → System Logs → Firewall) showed the scan clearly — logged as **Pass** on OPT1, matching the ATTACK→AD_LAB rule (Log enabled in Entry 003). Confirmed the specific matched rule via the log's expanded detail view.

**Reframed as a finding, not a failure:** host-based detection (Splunk/forwarders) and network-based detection (pfSense firewall logs) cover different blind spots — a scan against a target with no local firewall is invisible to endpoint logging but fully visible at the network boundary. This is the practical reason real environments run both a SIEM and network-level logging/IDS rather than relying on one layer alone.

**Minor side item — unresolved:** pfSense's log display showed timestamps in the wrong timezone (system default, not local). Attempted to correct via System → General Setup, but the display did not actually change despite the setting update — left unresolved rather than reported as fixed. Does not block reading the logs (entries are still correctly ordered and attributable, just labeled with an offset timestamp), but worth revisiting as a small open item rather than treating it as closed.

### Step 4 — Negative test: WAN block confirmed

From Ubuntu Desktop, attempted outbound connectivity (`ping 8.8.8.8` / `1.1.1.1`) — failed as expected. Confirmed in pfSense's firewall log as a **Block** entry on LAN, matching the explicit AD_LAB→WAN rule from Entry 003. Clean contrast against the Pass/Block pair now both captured for the build log and screenshots.

### Step 5 — Attempted fix for the host-layer gap (`ufw` install) — not completed

Attempted to close the Step 3 visibility gap by installing `ufw` on Ubuntu Desktop. This surfaced a long chain of blockers, each correctly diagnosed before moving to the next:

- `ufw: command not found` — checked `dpkg -l | grep ufw`, found status **`rc`** (removed, configuration remaining) — the package had been present at some point and removed, not simply never installed.
- Attempted to reinstall via `apt`, which requires AD_LAB→WAN access — blocked by design (Entry 003). Considered adding a standing ATTACK→WAN and AD_LAB→WAN rule set mirroring DEFENDER's outbound rules, but reconsidered on least-privilege grounds: DEFENDER's outbound need is continuous (ongoing patching of production-style services), while Kali/AD_LAB's need here was a one-off, and ATTACK specifically carries higher risk as the segment running untrusted/adversarial tooling. Declined to add standing rules for this reason.
- Explored downloading the package on the host Mac instead, entirely outside the lab network. Ruled out a persistent VirtualBox Shared Folder as leaving a standing host-guest channel open longer than necessary for a one-time transfer; used drag-and-drop instead, which is on-demand only, once Guest Additions were confirmed present.
- Drag-and-drop failed on Ubuntu Desktop directly (Guest Additions not present/configured) but succeeded on Kali (Guest Additions present on the official Kali image by default).
- Relaying the file from Kali to Ubuntu Desktop via `scp` failed — Ubuntu Desktop has no SSH server (the same missing-package situation from Step 2, now blocking the fix for Step 3's gap as well).
- Used `netcat` as a push-based transfer instead (`nc -l` listener on Ubuntu Desktop, `nc` send from Kali) — permitted under the existing ATTACK→AD_LAB allow rule, no new firewall changes needed. Transfer itself completed almost instantly given the small file size, but both terminals appeared to hang — normal `nc` behavior (it doesn't auto-close after EOF without the `-q` flag), not a failed transfer. Confirmed the file had already arrived by checking its size in a separate session before interrupting both hung terminals.
- `dpkg -i` on the transferred file flagged that installing `ufw` would conflict with **`iptables-persistent`**, the tool actively enforcing the Entry 014 ICMP timestamp remediation on this same VM.

**Decision — stopped rather than proceeded:** installing `ufw` risked overwriting or conflicting with an already-verified, documented security control (Entry 014) for the sake of a visibility enhancement that wasn't required to complete this session's actual goal. Declined the install. `iptables-persistent` remains the system's firewall/logging manager on Ubuntu Desktop, unchanged.

---

**Outcome:** Both core verification tests passed. ATTACK→AD_LAB traffic is detected (at the network layer); AD_LAB→WAN traffic is blocked and logged. The intended host-layer logging enhancement (`ufw`) was not completed, and was deliberately abandoned once it risked a real conflict with existing, verified infrastructure — a legitimate stopping point, not an unresolved failure.

**Lesson learned:** A test that "fails" against one detection layer isn't necessarily a real gap — it may simply reveal which layer is actually doing the work, and checking the other layer before assuming something is broken is the right instinct. Separately, a secondary fix attempted mid-session should be weighed against what it risks breaking elsewhere, not just what it would add — stopping short of a conflict with a previously verified control (Entry 014) was the correct call, even though it left today's enhancement incomplete.

**Real-world relevance:** The host-vs-network detection gap discovered here is a standard, well-known blind spot in real security operations — exactly why defense-in-depth architectures pair endpoint logging with network-level monitoring rather than trusting either alone. The decision to halt a change once it threatened an existing control mirrors real change-management discipline: verifying whether a new configuration conflicts with something already in production before applying it, and being willing to roll back or decline rather than push through a known risk for a lower-priority improvement.

---

## Phase 2, Entry 003 — Session 3: Firewall Rules and DNS Chain

**Date:** July 15, 2026
**Status:** ✅ Complete

**Action:** Replaced LAN's overly permissive default rule (identified in Entry 002) with an explicit, scoped rule set across all three internal interfaces, then resolved a full chain of downstream consequences that only surfaced once real least-privilege restrictions were in place.

**Goal:** Close the gap flagged at the end of Entry 002 — actual enforcement of the ATTACK/AD_LAB/DEFENDER segmentation, not just interfaces that happened to work because of a leftover permissive default.

---

### Step 1 — Snapshots and pre-work

Applied `phase2-entry003-pre-rules` snapshot naming across all VMs before making changes, per the standing "snapshot before a higher-risk change" practice established in prior sessions. Briefly reviewed pfSense's built-in anti-lockout rule (protects GUI/SSH access to pfSense itself from LAN) — confirmed it operates independently of the custom rule set being written and requires no changes or repositioning.

### Step 2 — Aliases

Created reusable aliases instead of typing raw subnets/ports into every rule:
- `ATTACK_NET`, `ADLAB_NET`, `DEFENDER_NET` — Type: Network(s)
- `SPLUNK_FWD` (port 9997) — Type: Ports
- `WEB_HTTP` (port 80), `WEB_HTTPS` (port 443) — Type: Ports, kept as two separate aliases rather than one combined `WEB` alias, specifically to allow scoping HTTP and HTTPS independently later if desired (e.g., restricting to HTTPS-only as a future hardening step)

### Step 3 — Rule set applied

**OPT1 (ATTACK):**
- ATTACK → AD_LAB, Protocol: Any — allow (broad scope intentional; ATTACK is the adversary-simulation segment, so restricting protocol here would be artificial)
- ATTACK → DEFENDER, Protocol: TCP, Port: `SPLUNK_FWD` — allow (covers Kali's optional forwarder, per Design Decision #01)

**LAN (AD_LAB):** default "allow LAN net to any" rule removed first, replaced with:
- AD_LAB → DEFENDER, Protocol: TCP, Port: `SPLUNK_FWD` — allow
- AD_LAB → Any, Protocol: Any — explicit block (made explicit rather than relying on implicit default-deny, since this rule is the actual segmentation enforcement worth pointing to directly)

**OPT2 (DEFENDER):**
- DEFENDER → WAN, Protocol: TCP, Port: `WEB_HTTP` — allow
- DEFENDER → WAN, Protocol: TCP, Port: `WEB_HTTPS` — allow
- (Both added as a deliberate choice to allow direct patching of Splunk/Eramba's host, rather than the alternative of leaving DEFENDER fully closed and requiring a temporary rule for every future update — a real trade-off, not an oversight)

### Step 4 — Verification round 1: Splunk dashboard access broke

**Problem encountered:** Splunk's web UI (port 8000) became unreachable from Ubuntu Desktop immediately after the new LAN rules were applied.

**Root cause:** The new AD_LAB→DEFENDER rule was correctly scoped to port 9997 (forwarder traffic) only — port 8000 (the web UI) was never covered, and previously worked only because LAN's old permissive default allowed it by accident.

**Resolution:** Added one more explicit rule — AD_LAB → DEFENDER, TCP, port 8000 — allow. Confirmed dashboard access restored.

### Step 5 — Verification round 2: Kali → AD_LAB ping failed

**Problem encountered:** `ping` from Kali to an AD_LAB host failed despite the ATTACK→AD_LAB allow rule existing.

**Root cause:** The rule's Protocol field had been set to TCP instead of Any — ICMP (ping) isn't TCP, so it fell through to the block. Likely picked up from the adjacent DEFENDER rule directly below it during entry.

**Resolution:** Corrected Protocol to Any. Retested — ping succeeded.

### Step 6 — Verification round 3: DNS chain

**Confirmed working as expected:** Ubuntu Desktop → internet (`ping 8.8.8.8`) failed — correct, confirms the AD_LAB block rule is enforcing.

**Problem encountered:** Testing DEFENDER → internet — `curl` wasn't installed on Ubuntu Server, so used `sudo apt update` instead. Failed with "temporary failure resolving" errors across multiple package sources.

**Root cause (two-part):** (1) Ubuntu Server's Netplan `nameservers` had been deliberately left empty in Entry 002 — correct at the time, since nothing needed hostname resolution yet, but now a real unmet dependency given DEFENDER's new outbound access. (2) No firewall rule existed permitting DNS traffic (port 53) outbound at all.

**Decision — enterprise-mirroring DNS design:** rather than pointing Ubuntu Server directly at a public resolver, chose to centralize DNS the way a real enterprise typically does — internal hosts query an internal resolver, which forwards upstream. pfSense's built-in DNS Resolver (unbound) serves this role. Configured:
- pfSense System → General Setup → DNS Servers: **9.9.9.9** (Quad9 primary) — chosen specifically for its DNS-layer malicious-domain blocking, a real, defensible security rationale rather than an arbitrary pick
- Added **149.112.112.112** (Quad9 secondary) for redundancy — same single-point-of-failure reasoning already applied to the DC01/WIN11 DNS dependency in Entry 002
- Added a DEFENDER → WAN rule, port 53, for DNS traffic
- Updated Ubuntu Server's Netplan `nameservers` to point at **pfSense's own DEFENDER-segment interface IP** (not Quad9 directly) — keeping DNS resolution routed through the same central enforcement point as everything else in this design

**Problem encountered:** After all of the above, resolution still failed.

**Root cause:** A typo in the configuration (self-identified, not further diagnosed via tooling).

**Resolution:** Corrected the typo. Retested `sudo apt update` — succeeded.

### Step 7 — Patch and reboot verification

Ran `sudo apt upgrade -y`, followed by `sudo apt full-upgrade -y` after the first pass left 20 packages upgradable (expected — `apt upgrade` intentionally won't add/remove dependent packages; `full-upgrade` completes what regular upgrade holds back). Rebooted Ubuntu Server and re-verified Splunk's boot-start persistence (`enable boot-start -user splunk`, set in Entry 002) — confirmed `splunkd` came back up automatically, still running as the `splunk` user, not root.

### Step 8 — Screenshots

Captured the finalized Firewall → Rules views for LAN and OPT1/OPT2 (pairing with the "before" LAN-default-rule screenshot from Entry 001) and the DNS Resolver/General Setup screen showing Quad9 primary/secondary configured.

### Step 9 — Post-verification snapshot

Initially planned to skip a snapshot between Sessions 3 and 4 since all VMs were already running. Reconsidered: the only existing snapshot at that point (`phase2-entry003-pre-rules`) predated the rule set entirely, meaning any rollback need during Session 4 would have undone all of this session's verified work along with whatever went wrong in Session 4. Shut down all six VMs cleanly and applied a new snapshot — `phase2-entry003-rules-verified` — capturing the fully working, documented post-Session-3 state as its own restore point before proceeding.

---

**Outcome:** Segmentation is now actually enforced, not just configured. All three interfaces carry explicit, purpose-built rules; LAN's dangerous permissive default is gone. Splunk dashboard access, Kali-to-AD_LAB connectivity, AD_LAB's internet block, and DEFENDER's scoped outbound access (web + DNS) are all verified working. Ubuntu Server is fully patched with confirmed boot-persistent Splunk.

**Lesson learned:** Nearly every problem this session surfaced was a direct, predictable consequence of previous sessions' correct-at-the-time decisions (Entry 002's empty DNS config) meeting a new requirement (DEFENDER needing real outbound access) — not new mistakes, but dependencies finally being exercised for the first time. The two actual entry errors (TCP instead of Any on the Kali rule; the DNS typo) were both simple, human, and easy to reproduce again — worth explicitly re-checking every new rule's Protocol field individually rather than assuming it carried over correctly from context, going forward.

**Real-world relevance:** This session is close to a textbook firewall-hardening exercise: moving from an implicit permissive default to explicit least-privilege rules, discovering and fixing the real (not hypothetical) breakage that causes, and building a centralized DNS resolution chain through a single enforcement point — mirroring how enterprise networks typically handle DNS rather than letting every host resolve independently. Choosing Quad9 specifically for its security filtering, and adding redundancy proactively rather than after an outage, both reflect deliberate, defensible operational decisions rather than default choices made without reasoning.

---

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