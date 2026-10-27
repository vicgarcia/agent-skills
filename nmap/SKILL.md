---
name: nmap
description: Network reconnaissance using nmap and nmap-vulners. Covers host discovery, port scanning, service detection, OS fingerprinting, NSE scripts, CVE lookup via vulners.com, and timeout-safe progressive scanning for reliable agent execution.
compatibility: Requires nmap. Raw-socket scan types (SYN, UDP, OS detection) need root or setcap on the nmap binary; CVE lookup needs the vulners NSE script. For authorized testing only.
---

# Nmap Network Reconnaissance

You are performing network reconnaissance using nmap. This skill covers the full recon pipeline from discovery through vulnerability identification, with agent-safe patterns for reliable execution.

## Find the Right Workflow

Use the worked recipes for an end-to-end task and the references when composing a custom scan. Repeated commands in those two contexts are intentional.

| I want to… | Start here | Then inspect |
|---|---|---|
| Find live machines on a network | [Quick Network Snapshot](#quick-network-snapshot) | [Host Discovery](#host-discovery) for blocked probes and MAC/vendor evidence |
| Investigate one machine thoroughly | [Single Host Deep Dive](#single-host-deep-dive) | [Service and Version Detection](#service-and-version-detection), [OS Detection](#os-detection) |
| Run discovery → inventory → CVE checks | [The Full Recon Kill Chain](#the-full-recon-kill-chain) | [Interpreting Results](#interpreting-results) |
| Choose targets, ports, or scan techniques | [Target Specification](#target-specification), [Port Specification](#port-specification) | [Scan Types](#scan-types), [Common Port Reference](#common-port-reference) |
| Assess web and TLS services | [Web Server Assessment](#web-server-assessment) | [SSL/TLS Deep Inspection](#ssltls-deep-inspection), [HTTP Deep Dive](#http-deep-dive) |
| Investigate SSH, SMB, DNS, databases, or industrial services | [High-Value Script Combos](#high-value-script-combos) | [Common Scripts by Service](#common-scripts-by-service) |
| Identify home-network or IoT services | [IoT / Smart Home Survey](#iot--smart-home-survey) | [Broadcast Discovery](#broadcast-discovery-find-hidden-devices) |
| Find UDP services and interpret ambiguous ports | [UDP Network Sweep](#udp-network-sweep) | [UDP Scan](#udp-scan), [Port States](#port-states) |
| Look up likely CVEs by detected version | [CVE Lookup with nmap-vulners](#cve-lookup-with-nmap-vulners) | [Vulnerability Scanning](#vulnerability-scanning) for active checks |
| Track what changed between observations | [Network Baseline](#network-baseline-save--diff) | [Tracking Changes with ndiff](#tracking-changes-with-ndiff) |
| Extract hosts, ports, or service records | [Output Formats](#output-formats) | [Parsing XML Output with Python](#parsing-xml-output-with-python) |
| Diagnose slow scans, missing hosts, or unclear banners | [Troubleshooting](#troubleshooting) | [Timing and Performance](#timing-and-performance), [Interpreting Results](#interpreting-results) |
| Work with larger ranges or IPv6 | [Advanced Techniques](#advanced-techniques) | [Large Network Optimization](#large-network-optimization), [IPv6 Scanning](#ipv6-scanning) |
| Test firewall/IDS behavior or investigate deceptive services | [Firewall / IDS Evasion](#firewall--ids-evasion) | [Scan Through Proxies](#scan-through-proxies), [Honeypot Detection](#honeypot-detection) for limitations |
| Recall a command quickly | [Quick Reference](#quick-reference) | The linked workflow/reference above for prerequisites and interpretation |

Examples use Bash and placeholder targets; replace them with authorized scope. `sudo` examples require suitable privileges. Vulnerability checks, credential attempts, broadcast probes, and firewall/IDS tests have different side effects—select them to match the agreed assessment, not just because a port is open.

## Installed Tools

| Tool | Purpose |
|------|---------|
| `nmap` | Port scanner, service detector, NSE script engine |
| `vulners.nse` | NSE script — queries vulners.com API for CVEs by service version |
| `ndiff` | Diffs two XML scan results; may be packaged separately |

Check the installed build and script catalog before using a recipe:

```bash
nmap --version
nmap --script-help vulners
nmap --script-help 'ssl-cert,smb-os-discovery'
command -v ndiff
```

The script names below were checked against Nmap 7.99; packages and third-party script sets differ. Use [NSE script documentation](https://nmap.org/nsedoc/) and `--script-help NAME` to verify arguments and availability. Python XML recipes require Python 3; raw-packet privileges and interface names are platform-specific.

### Dockerfile Setup

```dockerfile
# Install nmap
RUN apt-get update && apt-get install -y nmap libcap2-bin curl ca-certificates

# Grant agent user raw socket access for privileged scan types (SYN, UDP, OS detection)
# Linux only; not required if running as root. Container runtime must also
# permit the capabilities. Use --privileged with nmap when relying on file caps
# so it recognizes the non-root process's raw-packet permissions.
# This is Nmap's --privileged flag, not Docker's unrelated --privileged mode.
RUN setcap cap_net_raw,cap_net_admin+eip /usr/bin/nmap

# Install vulners NSE script for CVE lookup via vulners.com API
RUN curl -fsSL https://raw.githubusercontent.com/vulnersCom/nmap-vulners/master/vulners.nse \
    -o /usr/share/nmap/scripts/vulners.nse \
    && nmap --script-updatedb
```

Many packages already include `vulners`; check before replacing it. For a reproducible image, review and pin the upstream script revision instead of relying on the moving `master` URL above. `--privileged` tells Nmap to assume the OS permissions exist; it does not grant missing capabilities. These Linux `setcap` instructions do not apply to macOS.

---

## Execution Guidelines

- Always phase scans: discover → ports → services → CVEs. Never combine them on a subnet.
- Always set `--host-timeout` on any scan covering more than one host.
- Always extract open ports from a fast scan before running `-sV` or scripts.
- Always use `/tmp` for output files unless instructed otherwise.
- Use `--open` for compact inventory output; it also includes potentially open states such as UDP `open|filtered`. Omit it when preserving closed/filtered evidence matters, especially for baselines.
- Treat the `/tmp` names below as example artifacts. Use a task-specific directory or prefix when multiple runs must coexist, and record paths between tool calls; shell variables do not necessarily persist across calls.

---

## Agent-Optimized Scanning

Long-running scans that don't complete are the #1 failure mode for agent-driven nmap. The patterns below keep work bounded and make progress visible; they do not guarantee a wall-clock completion time. `--host-timeout` applies per host, not to the whole command. A host that times out may have no port/version/OS results in the saved report; a script timeout can suppress that script's output. Use the harness's actual wall-clock timeout controls and report incomplete coverage.

### The Cardinal Rules

1. **Never combine discovery + service detection + scripts in one command on a network range.** Split them into phases.
2. **Always extract open ports from a fast scan before running service detection.** Never run `-sV` on all 65535 ports.
3. **Scan one host at a time for deep work.** Loop over a host list rather than passing an entire subnet to a slow scan.
4. **Every command touching more than one host must have `--host-timeout`.** Without it, one unresponsive host can block the entire scan indefinitely.
5. **Prefer `--top-ports N` over `-p-`.** Top 100 ports is a quick sample of common services, not a guaranteed coverage percentage. Use full port scan only when specifically needed for a single host.

### Phase-Based Time Budget

| Phase | Scope | Typical Command | Illustrative budget (not a guarantee) |
|-------|-------|-----------------|----------|
| 1. Host discovery | /24 network | `-sn -PR -PE` | ~30s |
| 2. Quick port check | single host | `--top-ports 100 -T4` | ~15s |
| 3. Targeted port scan | single host, known ports | `-p $PORTS -T4` | ~30s |
| 4. Service detection | single host, open ports only | `-p $PORTS -sV` | ~60s |
| 5. CVE lookup | single host, open ports only | `-p $PORTS -sV --script vulners` | ~90s |
| 6. Vuln scripts | single host, specific ports | `-p $PORTS --script vuln` | ~120s |

**Never run Phase 4, 5, or 6 on a subnet range.** Always discover first, then process hosts individually.

### Safe Network Sweep Pattern

**Use when:** you need discovery and a common-port inventory before choosing hosts for deeper work. **Requires:** authorized subnet scope and suitable privileges for the discovery probes. Keep these as separate phases so you can inspect the host list before scanning ports.

```bash
# Command 1: discovery (roughly 30s on some responsive /24 LANs; not a wall-clock limit)
sudo nmap -sn -PR -PE -PS22,80,443 \
  --max-retries 1 --host-timeout 5s \
  192.168.1.0/24 -oG /tmp/hosts.gnmap || exit 1

# Extract live hosts, and stop if discovery supplied no targets.
awk '/Status: Up/ {print $2}' /tmp/hosts.gnmap > /tmp/live-hosts.txt
[ -s /tmp/live-hosts.txt ] || { echo 'No discovered targets; inspect discovery first'; exit 0; }
cat /tmp/live-hosts.txt

# Command 2: top-1000 port scan on live hosts only (20s per host, host-timeout enforced)
# --open suppresses closed/filtered noise — only shows what's actually reachable
nmap --top-ports 1000 -T4 --open \
  --max-retries 2 --host-timeout 20s \
  -iL /tmp/live-hosts.txt \
  -oG /tmp/ports.gnmap
```

**Expected/next:** `hosts.gnmap` records discovery and `ports.gnmap` records the top-port inventory. Verify completion/timeout diagnostics and choose hosts for the [per-host loop](#per-host-processing-loop). Missing discovery results or an incomplete port table require follow-up, not a conclusion that no services exist.

### Per-Host Processing Loop

**Use when:** the port sweep has finished and selected hosts need version/CVE investigation. **Requires:** `/tmp/ports.gnmap` from the sweep and permission to send service-identification data to vulners.com. Replace the example host list with the hosts of interest; use the service-only command if external lookup is out of scope.

```bash
for HOST in 192.168.1.100 192.168.1.101; do
  # Exact address comparison; portable grep -E works on macOS and Linux.
  PORTS=$(awk -v host="$HOST" '$1 == "Host:" && $2 == host' /tmp/ports.gnmap \
    | grep -oE '[0-9]+/open/tcp' | cut -d/ -f1 | paste -sd, -)

  if [ -z "$PORTS" ]; then
    printf 'No confirmed open TCP ports for %s; skipping deep scans\n' "$HOST"
    continue
  fi

  nmap -p "$PORTS" -sV -T4 --open --host-timeout 60s \
    "$HOST" -oN "/tmp/services-$HOST.txt" || continue

  nmap -p "$PORTS" -sV --script vulners --script-args mincvss=5.0 \
    --script-timeout 30s -T4 --open --host-timeout 90s \
    "$HOST" -oX "/tmp/cves-$HOST.xml"
done
```

**Expected:** one service report and one CVE XML file per processed host. No confirmed ports means no downstream command is launched. An empty port list can also mean a missing/timed-out host or an unscanned port set—not necessarily no services.

**Next:** inspect each report and scan diagnostics before expanding the host list. Confirm candidate CVEs against vendor/package evidence; retry an incomplete host separately with an appropriate budget rather than treating absent output as a clean result.

### What Makes Scans Slow (Avoid These)

| Dangerous Pattern | Why It Hangs | Safe Alternative |
|-------------------|--------------|-----------------|
| `nmap -sV 192.168.1.0/24` | service detection on 256 hosts | discover first, then per-host |
| `nmap -p- 192.168.1.0/24` | all 65535 ports × 256 hosts | `--top-ports 100` on network, `-p-` only on single hosts |
| `nmap --script vuln 192.168.1.0/24` | vuln scripts on entire subnet | run scripts per-host only |
| `nmap -sV -p-` without `--host-timeout` | no ceiling, runs forever | always set `--host-timeout 120s` |
| `nmap -sU 192.168.1.0/24` | UDP is extremely slow | `--top-ports 20` + `--host-timeout 60s` per host |
| `-T5` on a slow/filtered network | drops packets, needs retries anyway | use `-T4`, not `T5` |

### Scan Time Estimator

Use a small representative scan to estimate runtime on the actual network. Record elapsed time, live-host count, ports attempted, retries, and how many hosts/scripts timed out. A hosts × ports formula is not reliable: probes overlap, filtering/rate limits cause retries, and service/NSE execution depends on the protocol.

For planning, scale the observed sample cautiously and leave headroom for slower hosts. Time budgets in this skill are starting estimates, not measured benchmarks for every network. If a scan exceeds the available tool window, split the existing work into smaller host groups or run it through a supported background facility. Preserve the target list and output so that an interrupted run can be distinguished from a completed one.

### Bounded Scan Profiles

Choose the relevant invocation from this catalog rather than executing the entire block. Set `NETWORK`, `HOST`, and (where used) a nonempty comma-separated `PORTS` list first. The timeout numbers below are configured per-host budgets, not promises about total wall time.

```bash
# PROFILE: network-alive — discover a /24; 5s host timeout
sudo nmap -sn -PR -PE -PS22,80,443 --max-retries 1 --host-timeout 5s \
  "$NETWORK" -oG /tmp/alive.gnmap

# PROFILE: host-quick — top 100 ports; 15s host timeout
nmap --top-ports 100 -T4 --open --max-retries 2 --host-timeout 15s \
  "$HOST" -oG "/tmp/quick-$HOST.gnmap"

# PROFILE: network-ports — top 1000 ports on live hosts; 20s host timeout
nmap --top-ports 1000 -T4 --open --max-retries 2 --host-timeout 20s \
  -iL /tmp/live-hosts.txt -oG /tmp/ports.gnmap

# PROFILE: host-ports — all 65535 TCP ports on one host; 120s host timeout
sudo nmap -p- -T4 --min-rate 1000 --open --max-retries 2 --host-timeout 120s \
  "$HOST" -oG "/tmp/allports-$HOST.gnmap"

# PROFILE: host-services — known open ports; 60s host timeout
nmap -p "$PORTS" -sV --version-intensity 5 -T4 --open --host-timeout 60s \
  "$HOST" -oN "/tmp/services-$HOST.txt"

# PROFILE: host-cves — known open ports; 90s host timeout
nmap -p "$PORTS" -sV --script vulners --script-args mincvss=5.0 \
  --script-timeout 30s -T4 --open --host-timeout 90s \
  "$HOST" -oX "/tmp/cves-$HOST.xml"

# PROFILE: host-vulns — specific ports; 120s host timeout
nmap -p "$PORTS" --script vuln --script-timeout 60s \
  -T4 --open --host-timeout 120s \
  "$HOST" -oN "/tmp/vulns-$HOST.txt"

# PROFILE: host-web — web service assessment; 120s host timeout
nmap -p 80,443,8080,8443 -sV \
  --script http-enum,http-headers,ssl-cert,ssl-enum-ciphers,vulners \
  --script-args mincvss=5.0 --script-timeout 30s \
  -T4 --open --host-timeout 120s \
  "$HOST" -oN "/tmp/web-$HOST.txt"

# PROFILE: host-smb — SMB enumeration + selected vuln check; 90s host timeout
nmap -p 445 -sV \
  --script smb-os-discovery,smb-security-mode,smb-enum-shares,smb-vuln-ms17-010,vulners \
  --script-timeout 30s -T4 --open --host-timeout 90s \
  "$HOST" -oN "/tmp/smb-$HOST.txt"

# PROFILE: host-udp — high-value UDP ports; 120s host timeout
sudo nmap -sU -p 53,67,69,111,123,137,161,500,1900,5353 \
  --max-retries 1 -T4 --host-timeout 120s \
  "$HOST" -oX "/tmp/udp-$HOST.xml"
```

---

## The Full Recon Kill Chain

**Use when:** an authorized assessment needs discovery, port inventory, service identification, candidate CVEs, and an OS estimate. **Requires:** Nmap/raw-packet privileges, the vulners script and internet access for the CVE step, and permission for that external disclosure. This is a workflow, not proof of complete security coverage.

Steps 1–3 cover the network; steps 4–7 investigate one host at a time. Each code block runs in the same shell, or carry its variables/paths into subsequent tool calls.

```bash
NETWORK=192.168.1.0/24

# 1–2. Discover and extract live IPs.
sudo nmap -sn -PR -PE -PS22,80,443 --max-retries 1 --host-timeout 5s \
  "$NETWORK" -oG /tmp/hosts.gnmap || exit 1
awk '/Status: Up/ {print $2}' /tmp/hosts.gnmap > /tmp/live-hosts.txt
[ -s /tmp/live-hosts.txt ] || { echo 'No discovered targets; check scope/probes/diagnostics'; exit 0; }

# 3. Inventory common ports on discovered targets.
nmap --top-ports 1000 -T4 --open --max-retries 2 --host-timeout 20s \
  -iL /tmp/live-hosts.txt -oG /tmp/ports.gnmap || exit 1

HOST=192.168.1.100
PORT_SCAN=/tmp/ports.gnmap
```

**Step 4, optional:** if the common-port inventory is insufficient, run the following on that host. Updating `PORT_SCAN` is essential: subsequent steps must read the full-port results, not accidentally reuse the top-1000 file.

```bash
sudo nmap -p- -T4 --min-rate 1000 --host-timeout 120s --open \
  "$HOST" -oG "/tmp/allports-$HOST.gnmap" || exit 1
PORT_SCAN="/tmp/allports-$HOST.gnmap"
```

Continue from the selected inventory:

```bash
PORTS=$(awk -v host="$HOST" '$1 == "Host:" && $2 == host' "$PORT_SCAN" \
  | grep -oE '[0-9]+/open/tcp' | cut -d/ -f1 | paste -sd, -)
[ -n "$PORTS" ] || { echo 'No confirmed open TCP ports; inspect coverage before continuing'; exit 0; }

# 5. Service identification; intensity 5 is lighter than the default 7.
nmap -p "$PORTS" -sV --version-intensity 5 -T4 --host-timeout 60s --open \
  "$HOST" -oX "/tmp/services-$HOST.xml" -oN "/tmp/services-$HOST.txt" || exit 1

# 6. Candidate CVEs from detected service versions.
nmap -p "$PORTS" -sV --script vulners --script-args mincvss=5.0 \
  --script-timeout 30s -T4 --host-timeout 90s \
  "$HOST" -oX "/tmp/cves-$HOST.xml"

# 7. OS detection uses a normal port set to seek both open and closed ports.
sudo nmap -O --osscan-guess --max-os-tries 1 --host-timeout 90s \
  "$HOST" -oN "/tmp/os-$HOST.txt"
```

**Expected:** a live-host list, a port inventory, and host-specific service/CVE/OS reports. Review scan completion and timeout messages: a zero exit status or an existing file alone does not guarantee every host/script completed. OS matching works best with an open and a closed TCP port; scanning only known open ports can weaken it.

**Interpret and follow up:** separate observed ports, inferred products/OS, and version-based CVE candidates in the report. Deepen uncertain services with the relevant [service script reference](#common-scripts-by-service), validate CVE applicability, and report unscanned ports or timed-out hosts explicitly.

---

## CVE Lookup with nmap-vulners

The `vulners` NSE script queries the vulners.com API with detected service/version banners and returns known CVEs with CVSS scores. **Requires `-sV` to detect versions first.**

### Basic Usage

**Use when:** detected software versions need comparison with known vulnerability records. **Requires:** a locally available `vulners` script, reachable API, and approval to disclose product/CPE details. Start with known open ports from inventory; the unconstrained examples below illustrate the script invocation on the default port set.

```bash
# Scan and check CVEs in one pass
nmap -sV --script vulners 192.168.1.100

# Only report CVEs with CVSS score >= 7.0 (high/critical)
nmap -sV --script vulners --script-args mincvss=7.0 192.168.1.100

# Specific ports only (faster, more targeted)
nmap -p 22,80,443,3306 -sV --script vulners --script-args mincvss=5.0 192.168.1.100

# Full service detection + CVE lookup
nmap -p 22,80,443 -sV --version-intensity 5 --script vulners,ssl-cert,http-headers \
  --script-args mincvss=4.0 -T4 --host-timeout 120s 192.168.1.100
```

### Script Arguments

| Argument | Description | Example |
|----------|-------------|---------|
| `mincvss` | Minimum CVSS score to report | `mincvss=7.0` |

### Reading Vulners Output

```
22/tcp open  ssh     OpenSSH 8.4p1 Debian
| vulners:
|   cpe:/a:openbsd:openssh:8.4p1:
|     CVE-2023-38408   9.8   https://vulners.com/cve/CVE-2023-38408
|     CVE-2021-41617   7.0   https://vulners.com/cve/CVE-2021-41617
|     CVE-2021-28041   4.6   https://vulners.com/cve/CVE-2021-28041
```

The number after the CVE ID is the CVSS score: 9.0+ critical, 7.0–8.9 high, 4.0–6.9 medium. Keep the detected CPE/version and record link with the finding. A high score prioritizes investigation; it does not establish that this particular deployment is vulnerable.

### Notes
- Requires internet access; the script sends product/version or CPE information to vulners.com. Confirm this disclosure fits the assessment.
- Version intensity 7 is the default; 5 is a lighter explicit choice and 9 tries every probe. More probing may improve identification, but does not validate CVE applicability.
- Results are candidate matches, not proof of exploitability. Check vendor advisories, package backports, configuration, and prerequisites before reporting a confirmed vulnerability.
- No output can mean missing identification, API failure, timeout, a score filter, or no matches. Capture diagnostics and distinguish those cases before calling the host clean.
- `-sC` can add context, but default scripts are not uniformly harmless; inspect relevant scripts before combining them.

---

## Tracking Changes with ndiff

`ndiff` compares two nmap XML scans and shows what changed — new hosts, new ports, closed ports, version changes. **Requires:** two completed observations with the same target/port scope and scan options, plus `ndiff` installed. Preserve closed/filtered states rather than using `--open` for these baselines.

```bash
# Save baseline — port-level, no service detection
nmap --top-ports 1000 -T4 --host-timeout 20s \
  192.168.1.0/24 -oX /tmp/baseline.xml

# Later, save current state using the same coverage
nmap --top-ports 1000 -T4 --host-timeout 20s \
  192.168.1.0/24 -oX /tmp/current.xml

# Diff them
ndiff /tmp/baseline.xml /tmp/current.xml

# Show only additions (new hosts, new ports)
ndiff /tmp/baseline.xml /tmp/current.xml | grep "^+"

# Show only removals (hosts down, ports closed)
ndiff /tmp/baseline.xml /tmp/current.xml | grep "^-"
```

**ndiff output symbols:**
- `+` line: appeared in current scan, not in baseline (new)
- `-` line: was in baseline, gone now (removed)
- No prefix: unchanged

`ndiff` exit status is 0 for no differences, 1 for differences, and 2 for an error. Do not treat a useful difference report as command failure. Inspect the full report before filtering: an absent port/host may reflect timeout, packet loss, or discovery changes rather than a real removal. See the [baseline workflow](#network-baseline-save--diff) for follow-up.

---

## Target Specification

### Single Host
```bash
nmap 192.168.1.100
nmap scanme.nmap.org
```

### Multiple Hosts
```bash
nmap 192.168.1.1 192.168.1.10 192.168.1.100
```

### IP Ranges
```bash
nmap 192.168.1.1-50           # IPs 1 through 50
nmap 192.168.1-5.1-254        # Multiple subnets
```

### CIDR Notation
```bash
nmap 192.168.1.0/24           # 256 hosts
nmap 10.0.0.0/16              # 65536 hosts (use discovery only!)
```

### From File
```bash
nmap -iL targets.txt          # One target per line
```

### Random Targets
```bash
nmap -iR 10                   # 10 random internet hosts
```

### Exclusions
```bash
nmap 192.168.1.0/24 --exclude 192.168.1.1,192.168.1.254
nmap 192.168.1.0/24 --excludefile exclude.txt
```

---

## Port Specification

```bash
nmap -p 22                    # Single port
nmap -p 22,80,443             # Multiple ports
nmap -p 1-1024                # Port range
nmap -p-                      # All 65535 ports (slow - use timeout!)
nmap --top-ports 100          # Top 100 most common
nmap --top-ports 1000         # Default coverage
sudo nmap -sS -sU -p T:80,U:53 192.168.1.100  # TCP 80 + UDP 53; both scan types selected
```

---

## Common Port Reference

| Port | Service | Notes |
|------|---------|-------|
| 21 | FTP | File transfer |
| 22 | SSH | Secure shell |
| 23 | Telnet | Insecure remote access |
| 25 | SMTP | Email |
| 53 | DNS | Domain resolution |
| 80 | HTTP | Web |
| 88 | Kerberos | AD authentication |
| 110 | POP3 | Email |
| 111 | RPC | Remote procedure call |
| 135 | MSRPC | Windows RPC |
| 139 | NetBIOS | Windows networking |
| 143 | IMAP | Email |
| 389 | LDAP | Directory services |
| 443 | HTTPS | Secure web |
| 445 | SMB | Windows file sharing |
| 636 | LDAPS | Secure LDAP |
| 993 | IMAPS | Secure IMAP |
| 995 | POP3S | Secure POP3 |
| 1433 | MSSQL | Microsoft SQL |
| 1521 | Oracle | Oracle DB |
| 1720 | H.323 | VoIP signaling |
| 3306 | MySQL | MySQL DB |
| 3389 | RDP | Remote desktop |
| 5060 | SIP | VoIP signaling (UDP) |
| 5432 | PostgreSQL | PostgreSQL DB |
| 5900 | VNC | Remote desktop |
| 6379 | Redis | Redis DB |
| 8080 | HTTP-Alt | Alt web / proxy |
| 8443 | HTTPS-Alt | Alt secure web |
| 27017 | MongoDB | MongoDB |

---

## Scan Types

### TCP SYN Scan (Default, Stealth)
```bash
sudo nmap -sS 192.168.1.100
```
Requires root. Fast, doesn't complete TCP handshake. Default when root.

### TCP Connect Scan
```bash
nmap -sT 192.168.1.100
```
No root required. Completes full TCP handshake. More detectable.

### UDP Scan
```bash
# Top 20 UDP ports on one host (≤120s)
sudo nmap -sU --top-ports 20 --max-retries 1 -T4 --host-timeout 120s 192.168.1.100

# Targeted high-value UDP ports — faster and more focused
sudo nmap -sU -p 53,67,69,111,123,137,161,500,1900,5353 \
  --max-retries 1 -T4 --host-timeout 120s 192.168.1.100

# Combined TCP SYN + UDP on one host
sudo nmap -sS -sU --top-ports 20 -T4 --host-timeout 180s 192.168.1.100
```
UDP is extremely slow — always use `--max-retries 1` and `--host-timeout`. Never use `-p-` for UDP. High-value targets: DNS (53), DHCP (67/68), TFTP (69), NTP (123), NetBIOS (137), SNMP (161), IKE/VPN (500), UPnP/SSDP (1900), mDNS (5353).

**`open|filtered`**: Most UDP ports return this state because there's no response — nmap can't distinguish a firewall drop from a listening service. Confirm with service-specific scripts (`--script snmp-info`, `--script dns-service-discovery`, `--script ntp-info`).

### Ping Scan (Host Discovery Only)
```bash
nmap -sn 192.168.1.0/24
```
No port scan, just find live hosts. Fast for network surveys.

### ACK Scan (Firewall Detection)
```bash
sudo nmap -sA 192.168.1.100
```
Maps firewall rules (filtered vs unfiltered). Doesn't find open ports.

### Window Scan
```bash
sudo nmap -sW 192.168.1.100
```
Like ACK scan but can sometimes distinguish open from closed based on TCP window size. Useful when ACK scan isn't enough to differentiate.

### Protocol Scan
```bash
sudo nmap -sO 192.168.1.100
```
Discovers which IP protocols the host supports (TCP, UDP, ICMP, IGMP, etc.) — not ports, but protocol numbers.

### FIN/NULL/Xmas Scans (Firewall Evasion)
```bash
sudo nmap -sF 192.168.1.100   # FIN scan
sudo nmap -sN 192.168.1.100   # NULL scan
sudo nmap -sX 192.168.1.100   # Xmas scan (FIN+PSH+URG)
```

---

## Host Discovery

### Discovery Methods
```bash
nmap -sn -PR 192.168.1.0/24           # ARP ping (local net, fastest)
nmap -sn -PE 192.168.1.0/24           # ICMP echo
nmap -sn -PS22,80,443 192.168.1.0/24  # TCP SYN ping
nmap -sn -PA80,443 192.168.1.0/24     # TCP ACK ping
nmap -sn -PU53,161 192.168.1.0/24     # UDP ping
```

### Skip Discovery
```bash
nmap -Pn 192.168.1.100        # Treat host as up (bypass ping)
```

### Recommended Fast Discovery
```bash
sudo nmap -sn -PR -PE -PS22,80,443 --max-retries 1 --host-timeout 3s 192.168.1.0/24
```

### MAC Address and Vendor Identification

On local networks, `-sn` discovery automatically captures MAC addresses and resolves the vendor from the OUI prefix. This lets you identify device types without scanning a single port — a MAC starting with an Apple OUI is probably a Mac or iPhone, a Raspberry Pi Foundation OUI is a Pi, Ubiquiti means a network appliance, etc.

```
Nmap scan report for 192.168.1.1
Host is up (0.0030s latency).
MAC Address: DC:A6:32:XX:XX:XX (Raspberry Pi Trading)

Nmap scan report for 192.168.1.45
Host is up (0.0010s latency).
MAC Address: 3C:22:FB:XX:XX:XX (Apple)
```

Extract MAC addresses and vendors from normal or XML output. Grepable output does **not** contain a usable `MAC:` field, so keep XML for correlation:

```bash
sudo nmap -sn 192.168.1.0/24 -oN /tmp/hosts.txt -oX /tmp/hosts.xml

# Compact human view of the normal report
grep -E 'report for|MAC Address:' /tmp/hosts.txt

# Structured IP / MAC / vendor correlation
python3 - <<'PY'
import xml.etree.ElementTree as ET

for host in ET.parse('/tmp/hosts.xml').findall('host'):
    ip = next((a.get('addr') for a in host.findall('address')
               if a.get('addrtype') in {'ipv4', 'ipv6'}), None)
    mac = host.find("address[@addrtype='mac']")
    if ip:
        print(ip, mac.get('addr', '-') if mac is not None else '-',
              mac.get('vendor', '-') if mac is not None else '-', sep='\t')
PY
```

**Interpretation:** MAC information is normally available only for directly connected Layer-2 peers; a remote scan generally cannot obtain the target's MAC. Do not report the next-hop router's MAC as the remote device's identity. An OUI is a manufacturer hint, not a model/OS guarantee; virtual and randomized addresses weaken it. Correlate with hostname and service evidence in the [IoT workflow](#iot--smart-home-survey).

---

## Service and Version Detection

```bash
nmap -sV 192.168.1.100                    # Basic version detection
nmap -sV --version-intensity 0 ...        # Light (fastest)
nmap -sV --version-intensity 5 ...        # Lighter than default (7)
nmap -sV --version-intensity 9 ...        # Try all probes (slow)
nmap -p 22,80,443 -sV -T4 --host-timeout 90s 192.168.1.100   # Targeted (recommended)
```

---

## OS Detection

```bash
sudo nmap -O 192.168.1.100                    # Basic
sudo nmap -O --osscan-guess --max-os-tries 1 192.168.1.100  # More speculative matching, single attempt
sudo nmap -O -sV 192.168.1.100                # Combined OS + version
```

---

## Timing and Performance

```bash
nmap -T0 ...   # Paranoid   - IDS evasion, very slow
nmap -T1 ...   # Sneaky     - IDS evasion, slow
nmap -T2 ...   # Polite     - Reduced load
nmap -T3 ...   # Normal     - Default
nmap -T4 ...   # Aggressive - Fast, modern networks (use this)
nmap -T5 ...   # Insane     - Very fast, may miss results
```

### Recommended Timeout Flags

| Scan Type | Flags | Illustrative duration (network-dependent) |
|-----------|-------|----------|
| Host discovery /24 | `--max-retries 1 --host-timeout 5s` | 10-30s |
| Top 100 ports | `--max-retries 2 -T4` | 5-15s |
| Top 1000 ports | `--host-timeout 60s -T4` | 15-45s |
| All ports (-p-) | `--host-timeout 120s --min-rate 1000 -T4` | 60-120s |
| Service detection | `--host-timeout 90s -T4` | 30-90s |
| UDP scan | `--host-timeout 120s --max-retries 1 -T4` | 60-180s |
| Vulners CVE lookup | `--host-timeout 120s -T4` | 30-90s |

---

## NSE Scripts

### Script Categories
```bash
nmap --script auth ...        # Authentication bypass/checks
nmap --script brute ...       # Brute force attacks
nmap --script default ...     # Default selection (-sC); not a guarantee of harmlessness
nmap --script discovery ...   # Service discovery
nmap --script safe ...        # Classified lower-risk; still active, not risk-free
nmap --script vuln ...        # Built-in vulnerability detection
nmap --script vulners ...     # CVE lookup via vulners.com (installed)
```

### Running Scripts
```bash
nmap --script <name> ...              # Single script
nmap --script <name1>,<name2> ...     # Multiple scripts
nmap --script "http-*" ...            # Wildcard
nmap --script "default and safe" ...  # Boolean expression
nmap -sC ...                          # Same as --script default
```

### High-Value Script Combos

**Use when:** inventory has identified a service and you want a focused bundle rather than the entire script catalog. **Requires:** the named scripts and matching ports from your inventory; these default ports are examples. The vulners component requires external API access. Database/SMB enumeration or empty-password tests can be more intrusive than banner reads; select the bundle to fit the authorized scope.

| Bundle | Evidence to expect | Interpretation and next step |
|---|---|---|
| Web | Paths, headers, methods, certificates/ciphers, CVE candidates | Verify the intended vhost and investigate relevant paths/TLS findings; see [Web Server Assessment](#web-server-assessment) |
| SSH | Host keys, authentication methods, algorithm lists, version candidates | Compare keys with trusted inventory and algorithms with policy; an offered method is not proof a login succeeds |
| SMB | OS/domain hints, signing settings, share enumeration, selected active checks | Distinguish guest/anonymous visibility from authenticated access; verify the actual SMB dialect and permissions |
| Databases | Product details and results of selected empty-password checks | A successful check is access evidence, not permission to read/export application data; confirm the affected instance and auth configuration |
| Vulnerability sweep | Output from multiple active checks plus external version matches | Keep script-specific evidence separate from banner/CPE candidates; inspect missing/error/timeout output before reporting coverage |

**Web servers**
```bash
nmap -p 80,443 -sV --script http-enum,http-headers,http-methods,ssl-cert,ssl-enum-ciphers,vulners \
  --script-args mincvss=5.0 --script-timeout 30s -T4 --open --host-timeout 120s 192.168.1.100
```

**SSH**
```bash
nmap -p 22 -sV --script ssh-hostkey,ssh-auth-methods,ssh2-enum-algos,vulners \
  --script-args mincvss=5.0 --script-timeout 30s -T4 --open --host-timeout 60s 192.168.1.100
```

**SMB**
```bash
nmap -p 445 -sV --script smb-os-discovery,smb-security-mode,smb-enum-shares,smb-vuln-ms17-010,vulners \
  --script-args mincvss=5.0 --script-timeout 30s -T4 --open --host-timeout 90s 192.168.1.100
```

**Databases**
```bash
nmap -p 3306,5432,1433,27017,6379 -sV --script "*-info,*-empty-password,vulners" \
  --script-args mincvss=5.0 --script-timeout 30s -T4 --open --host-timeout 90s 192.168.1.100
```

**Full vulnerability sweep**
```bash
nmap -p 22,80,443,445,3306 -sV --script vuln,vulners \
  --script-args mincvss=5.0 --script-timeout 60s -T4 --open --host-timeout 180s 192.168.1.100
```

### Common Scripts by Service

**HTTP (80, 443, 8080)**
```bash
nmap -p 80 --script http-enum,http-methods,http-headers,http-title,http-robots.txt ...
nmap -p 443 --script ssl-cert,ssl-enum-ciphers,ssl-heartbleed ...
nmap -p 80 --script http-waf-detect,http-waf-fingerprint ...
nmap -p 80 --script http-backup-finder,http-config-backup ...
```

**SSH (22)**
```bash
nmap -p 22 --script ssh-hostkey,ssh-auth-methods,ssh2-enum-algos ...
```

**SMB (445)**
```bash
nmap -p 445 --script smb-enum-domains,smb-enum-shares,smb-enum-users,smb-os-discovery,smb-security-mode ...
nmap -p 445 --script smb-vuln-ms17-010,smb-vuln-ms08-067,smb-vuln-ms10-054 ...
```

**DNS (53)**
```bash
nmap -p 53 --script dns-zone-transfer,dns-brute ...
```

**SNMP (161 UDP)**
```bash
nmap -sU -p 161 --script snmp-info,snmp-interfaces,snmp-brute ...
```

**NTP (123 UDP)**
```bash
nmap -sU -p 123 --script ntp-info ...           # Version, stratum, reference clock
nmap -sU -p 123 --script ntp-monlist ...        # Connected clients (if enabled — often reveals device list)
```

**mDNS / Bonjour (5353 UDP)**
```bash
nmap -sU -p 5353 --script dns-service-discovery 192.168.1.0/24   # Discover services on local LAN
```
Reveals device names, services (AirPlay, AirPrint, Spotify Connect, Chromecast, etc.) without touching any TCP ports. Very effective on home networks.

**DHCP (67 UDP)**
```bash
nmap -sU -p 67 --script dhcp-discover ...       # DHCP server info — reveals router, subnet, DNS
```

**FTP (21)**
```bash
nmap -p 21 --script ftp-anon,ftp-bounce,ftp-brute ...
```

**Databases**
```bash
nmap -p 3306 --script mysql-info,mysql-databases,mysql-users,mysql-dump-hashes ...
nmap -p 1433 --script ms-sql-info,ms-sql-config,ms-sql-empty-password,ms-sql-ntlm-info ...
nmap -p 6379 --script redis-info,redis-brute ...
nmap -p 27017 --script mongodb-databases,mongodb-info ...
```

**IoT / Industrial**
```bash
nmap -p 502 --script modbus-discover ...          # Modbus
nmap -sU -p 47808 --script bacnet-info ...        # BACnet
nmap -p 102 --script s7-info ...                  # Siemens S7
nmap -sU -p 1900 --script upnp-info ...           # UPnP
nmap --script "broadcast-upnp-info" ...           # Broadcast UPnP discovery
```

**VoIP / Telephony**
```bash
nmap -sU -p 5060 --script sip-methods,sip-enum-users 192.168.1.100   # SIP
nmap -p 1720 -sV --version-all 192.168.1.100                          # H.323 fingerprinting
nmap -p 5038 -sV --script banner 192.168.1.100                       # Asterisk AMI banner
```

`h323-info` and `asterisk-info` are not bundled in the checked Nmap 7.99 script set and have no current official NSE documentation page. Do not assume those names exist. The built-in version probes/banner script above provide a starting point; label any separately sourced enumeration script with its dependency and verify its help before use.

**LDAP / Active Directory**
```bash
nmap -p 389,636 --script ldap-rootdse,ldap-search,ldap-brute 192.168.1.100
nmap -p 88 --script krb5-enum-users --script-args krb5-enum-users.realm='DOMAIN.COM' 192.168.1.100
```

---

## Output Formats

```bash
nmap -oN /tmp/scan.txt ...         # Normal (human readable)
nmap -oG /tmp/scan.gnmap ...       # Grepable (parseable)
nmap -oX /tmp/scan.xml ...         # XML (for tools, ndiff)
nmap -oA /tmp/scan ...             # All formats at once
```

### Parsing Grepable Output

Grepable output is useful for quick shell work but is deprecated for structured integrations; prefer XML for MACs, script results, and protocol/state fidelity. These examples use portable `grep -E`, not GNU-only `grep -P`:

```bash
# Confirmed open TCP ports for exactly one address; output is a comma-separated list.
awk '$1 == "Host:" && $2 == "192.168.1.100"' /tmp/scan.gnmap \
  | grep -oE '[0-9]+/open/tcp' | cut -d/ -f1 | paste -sd, -

# Discovered live hosts
awk '/Status: Up/ {print $2}' /tmp/hosts.gnmap

# Union of confirmed open TCP ports across all hosts (not a per-host mapping)
grep -oE '[0-9]+/open/tcp' /tmp/scan.gnmap | cut -d/ -f1 | sort -un
```

An empty result must stop downstream `-p "$PORTS"` commands. Inspect scan errors/timeouts before concluding there are no ports. The `/open/tcp` match deliberately excludes UDP and `open|filtered`; do not feed an all-host port union into a report claiming those ports are open on every host.

### Parsing XML Output with Python

```bash
# Confirmed open ports by IP, preserving TCP/UDP identity.
python3 - <<'PY'
import xml.etree.ElementTree as ET

for host in ET.parse('/tmp/scan.xml').findall('host'):
    addr = next((a.get('addr') for a in host.findall('address')
                 if a.get('addrtype') in {'ipv4', 'ipv6'}), None)
    if not addr:
        continue
    ports = []
    for port in host.findall('ports/port'):
        state = port.find('state')
        if state is not None and state.get('state') == 'open':
            ports.append(f"{port.get('portid')}/{port.get('protocol')}")
    if ports:
        print(f'{addr}: {", ".join(ports)}')
PY

# Service records as TSV: IP, protocol, port, service, product, version.
python3 - <<'PY'
import xml.etree.ElementTree as ET

for host in ET.parse('/tmp/scan.xml').findall('host'):
    addr = next((a.get('addr') for a in host.findall('address')
                 if a.get('addrtype') in {'ipv4', 'ipv6'}), None)
    if not addr:
        continue
    for port in host.findall('ports/port'):
        state, svc = port.find('state'), port.find('service')
        if state is not None and state.get('state') == 'open' and svc is not None:
            print(addr, port.get('protocol', ''), port.get('portid', ''),
                  svc.get('name', ''), svc.get('product', ''), svc.get('version', ''), sep='\t')
PY
```

---

## Vulnerability Scanning

### Built-in Vuln Scripts
```bash
# Run all vuln-category scripts (comprehensive but slow)
nmap -p 22,80,443,445 --script vuln --host-timeout 180s 192.168.1.100

# EternalBlue (MS17-010) check
nmap -p 445 --script smb-vuln-ms17-010 192.168.1.100

# SSL/TLS vulnerabilities
nmap -p 443 --script ssl-heartbleed,ssl-poodle,ssl-ccs-injection,ssl-dh-params 192.168.1.100

# Shellshock
nmap -p 80 --script http-shellshock --script-args uri=/cgi-bin/test.cgi 192.168.1.100
```

### CVE Lookup with vulners (installed)
```bash
# Fast CVE check on common ports
nmap -p 22,80,443 -sV --script vulners --script-args mincvss=7.0 192.168.1.100

# All open ports, show CVEs 5.0+
nmap -p $PORTS -sV --script vulners --script-args mincvss=5.0 \
  -T4 --host-timeout 120s 192.168.1.100

# Combined: built-in vuln scripts + vulners CVE lookup
nmap -p $PORTS -sV --script "vuln,vulners" --script-args mincvss=5.0 \
  --host-timeout 180s 192.168.1.100 -oX /tmp/vulnscan.xml
```

---

## Firewall / IDS Evasion

### Fragment Packets
```bash
sudo nmap -f ...              # Fragment packets
sudo nmap --mtu 16 ...        # Specify MTU
```

### Decoys
```bash
sudo nmap -D RND:10 ...       # 10 random decoys
sudo nmap -D 192.168.1.5,ME,192.168.1.6 ...
```

### Source Port Manipulation
```bash
sudo nmap --source-port 53 ...   # From port 53 (DNS, often trusted)
sudo nmap -g 80 ...              # Alias for --source-port 80
```

### Timing Evasion
```bash
sudo nmap -T1 --scan-delay 5s --max-parallelism 1 ...  # IDS evasion mode
sudo nmap -T2 --randomize-hosts 192.168.1.0/24 ...     # Random host order
```

### Full Evasion Combo
```bash
sudo nmap -sS -T1 --mtu 16 -D RND:5 --source-port 53 \
  --data-length 100 --scan-delay 2s 192.168.1.100
```

### Idle/Zombie Scan (Ultimate Stealth)
```bash
# Find a zombie host (needs incremental IP ID)
nmap --script ipidseq 192.168.1.0/24  # Look for "Incremental"

# Idle scan using a separately authorized intermediary; stealth is not guaranteed
sudo nmap -sI zombie-ip:80 target-ip
```

**Interpretation:** this method infers target responses from changes in the intermediary's IP ID sequence; it depends on a suitable, quiet host and compatible network paths. Authorize both the target and intermediary. Do not claim anonymity: discovery or other enabled scan phases may contact the target directly, and the intermediary/scanner traffic can be observed. Inconsistent IP IDs make the inference unreliable rather than proving a target port state.

---

## Advanced Techniques

### Broadcast Discovery (Find Hidden Devices)
```bash
nmap --script "broadcast-*" --script-args broadcast.timeout=30
nmap --script broadcast-upnp-info                       # UPnP/IoT
nmap --script broadcast-dhcp-discover                   # DHCP info
nmap --script broadcast-netbios-master-browser          # NetBIOS
sudo nmap -e en0 --script broadcast-dns-service-discovery --script-timeout 30s  # mDNS/Bonjour on this LAN
```

### SSL/TLS Deep Inspection

**Use when:** a confirmed TLS service needs certificate, protocol/cipher, or specific vulnerability checks. **Requires:** the correct port/hostname and authorization for active handshake/vulnerability probes. `ssl-enum-ciphers` makes many connections; compare its offered suites with policy rather than interpreting every supported suite as exploitable.

```bash
nmap -p 443 --script ssl-cert,ssl-enum-ciphers,ssl-heartbleed,ssl-dh-params 192.168.1.100
nmap -p 443 --script ssl-cert -vv 192.168.1.100  # Verbose certificate details and PEM (not a guaranteed full chain)
```

**Expected/next:** inspect certificate names, issuer, validity and fingerprint, accepted TLS versions/ciphers, and each vulnerability script's explicit evidence. A valid certificate does not establish application security; confirm hostname/SNI and verify actionable findings independently. An empty script result may be a timeout or unsupported handshake.

### HTTP Deep Dive

**Use when:** you need candidate paths, server/WAF hints, or matching text from an identified HTTP service. **Requires:** the intended host/vhost and permission for enumeration/crawling. Inspect crawl limits in `nmap --script-help http-grep` and the [HTTP library options](https://nmap.org/nsedoc/lib/http.html).

```bash
nmap -p 80 --script http-enum,http-backup-finder,http-waf-detect,http-sitemap-generator 192.168.1.100
nmap -p 80 --script http-grep --script-args \
  'http-grep.match="[A-Za-z0-9._%+%-]+@[A-Za-z0-9.%-]+%.[A-Za-z]+"' 192.168.1.100
```

`http-grep.match` uses **Lua patterns**, not PCRE: use `%.` for a literal dot, and do not use regex quantifiers such as `{2,}` or alternation as if this were grep. The email-like pattern is a discovery hint, not an address validator. Verify matching content and scope before following URLs returned by a crawler.

### SMB Full Enumeration

**Use when:** a known SMB host needs domain, share, user, service, or session inventory. **Requires:** authorized enumeration and credentials where required; without them, some scripts return partial data or access-denied results. Do not embed real passwords in committed examples or reports; consult script help for authentication arguments supported by the installed version.

```bash
nmap -p 445 --script \
  smb-enum-domains,smb-enum-groups,smb-enum-processes,smb-enum-services,\
  smb-enum-sessions,smb-enum-shares,smb-enum-users,\
  smb-os-discovery,smb-security-mode,smb-system-info \
  192.168.1.100
```

**Expected/next:** correlate domain/OS details, shares, visible accounts, services, and sessions. Distinguish an advertised share from confirmed readable/writable access. Access denied is an authentication/authorization observation, not proof the resource does not exist. Validate surprising permissions with the intended account and report which identity produced each result.

### Large Network Optimization

**Use when:** the authorized range is too large for an interactive single-host investigation. **Requires:** an explicit range, acceptable scanner/network load, sufficient runtime, and retained per-subnet artifacts. The example below launches local concurrent processes—it is not a distributed scanner. Its fan-out can overwhelm a scanner or network; select concurrency and budgets for the actual environment before running it.

```bash
# Phase 1: Parallel host discovery
sudo nmap -sn -PR --min-hostgroup 256 --min-parallelism 100 10.0.0.0/16 -oG /tmp/hosts.gnmap

# Phase 2: Local concurrent port scans per subnet (review fan-out before running)
for i in $(seq 0 255); do
  nmap --top-ports 100 -T4 --max-retries 1 10.0.$i.0/24 -oG "/tmp/subnet_$i.gnmap" &
done
wait
cat /tmp/subnet_*.gnmap > /tmp/all_ports.gnmap
```

**Expected/next:** separate files identify which subnet scans produced output. Preserve them and each process's diagnostics; a concatenated grepable file has multiple scan headers and is not a single authoritative completion record. Check for missing/failed jobs before processing hosts with the [per-host loop](#per-host-processing-loop). Discovery on a routed range can miss silent/firewalled hosts.

### Scan Through Proxies

Nmap's option is `--proxies`. It affects supported **NSE and version-detection connections**, not host discovery, port scanning, or OS detection. The following commands still perform a direct TCP port scan and are **not anonymous scans through Tor**. The local SOCKS4 endpoint must actually support that protocol; HTTP/SOCKS4 support does not imply SOCKS5 or remote DNS support.

```bash
# Direct TCP port scan; supported -sV / NSE connections may use the proxy
nmap --proxies socks4://127.0.0.1:9050 -sT -Pn -p 80 -sV --script http-title 192.168.1.100
nmap --proxies http://proxy:8080 -sT -Pn -p 80 -sV --script http-title 192.168.1.100
```

**Verify:** compare observed connections with proxy logs; do not infer privacy from the option being accepted. DNS resolution and unsupported connections may bypass the proxy. See the installed version's `--proxies` documentation before relying on it.

### Custom Packet Crafting
```bash
sudo nmap --scanflags URGACKPSHRSTSYNFIN 192.168.1.100   # All TCP flags
sudo nmap --data-length 50 ...                           # Append random data
sudo nmap --badsum ...                                   # Bad checksums (test IDS)
```

### IPv6 Scanning

**Use when:** the target/network uses IPv6; IPv4 discovery does not inventory IPv6-only addresses. **Requires:** IPv6 connectivity and an explicit interface for link-local/multicast work. Replace `en0` with the local interface and the example address with an authorized peer.

```bash
nmap -6 -e en0 fe80::1                              # Link-local target on the selected interface
nmap -6 -e en0 --top-ports 100 -T4 fe80::1            # Common TCP ports
sudo nmap -6 -e en0 --script ipv6-node-info fe80::1   # Node info, if supported by the peer

# Local multicast discovery via a dedicated prerule script, not an ordinary
# port scan of the multicast address. Does not scan discovered targets here.
sudo nmap -6 -sL --script targets-ipv6-multicast-echo \
  --script-args interface=en0
```

**Expected/next:** multicast responders provide candidate IPv6 addresses; follow up selected authorized addresses with unicast inventory. Silence is not proof of absence. Avoid an IPv4-style exhaustive sweep of a /64; learn addresses from discovery and existing inventory. Scan types and NSE scripts differ in IPv6 support—verify each one instead of assuming parity with IPv4. Inspect other `targets-ipv6-multicast-*` scripts individually before selecting them.

### Honeypot Detection

There is no bundled `honeypot-detection` script in the checked Nmap 7.99 catalog. Keep this investigation evidence-based rather than depending on an invented script name. Start with ports discovered by the [Single Host Deep Dive](#single-host-deep-dive):

```bash
# Set PORTS to the confirmed open ports for this authorized host first.
: "${PORTS:?Set PORTS from the host inventory before probing}"
nmap -p "$PORTS" -sV --version-all --host-timeout 180s 192.168.1.100

# Collect banners for comparison; a short timeout can miss slow real services.
nmap -p "$PORTS" -sV --script banner --script-args banner.timeout=2s \
  --script-timeout 15s --host-timeout 180s 192.168.1.100
```

**Look for:** identical banners on unrelated ports, protocol behavior inconsistent with the claimed product, and conflicting service/OS evidence. Uniform timing or generic versions alone are weak clues: proxies, load balancers, and real appliances can behave similarly. Record a suspicion with supporting observations, not a definitive honeypot classification. Follow up with a targeted protocol query and compare results over time.

---

## Common Workflows

### Quick Network Snapshot

**Use when:** you need a first inventory of reachable devices and common TCP services. **Requires:** the authorized subnet and raw-packet privileges for the discovery probes below. On a responsive /24 this may take a few minutes; firewall/rate-limit behavior can change that substantially.

```bash
sudo nmap -sn -PR -PE -PS22,80,443 --max-retries 1 --host-timeout 3s \
  192.168.1.0/24 -oG /tmp/hosts.gnmap || exit 1
awk '/Status: Up/ {print $2}' /tmp/hosts.gnmap > /tmp/live-hosts.txt

# -iL avoids xargs splitting a large target list into scans that overwrite one file.
if [ -s /tmp/live-hosts.txt ]; then
  nmap --top-ports 100 -T4 --host-timeout 30s \
    -iL /tmp/live-hosts.txt -oN /tmp/quick-scan.txt -oX /tmp/quick-scan.xml
else
  echo 'No discovered hosts; inspect discovery diagnostics before scanning further'
fi
```

**Expected:** `/tmp/live-hosts.txt` lists discovered IPs; the normal/XML reports contain common-port states for scanned hosts. This is not an all-port inventory. A host absent from discovery may still be online, and a timed-out host may have no port table.

**Next:** choose a host for the [Single Host Deep Dive](#single-host-deep-dive). If a known machine is missing, consult [When `-Pn` Is Needed](#when--pn-is-needed) and test that host explicitly rather than assuming the whole subnet is empty.

### Single Host Deep Dive

**Use when:** one host deserves a full TCP port inventory plus deeper identification. **Requires:** an authorized host, raw-packet privileges for SYN/OS scans, reviewed default scripts for `-sC`, and vulners/API access for CVE lookup. The rate setting is suitable only when the network can tolerate it; packet loss and the host timeout can make the inventory incomplete.

```bash
HOST=192.168.1.100
sudo nmap -p- -T4 --min-rate 1000 --host-timeout 120s \
  "$HOST" -oG /tmp/allports.gnmap || exit 1

PORTS=$(awk -v host="$HOST" '$1 == "Host:" && $2 == host' /tmp/allports.gnmap \
  | grep -oE '[0-9]+/open/tcp' | cut -d/ -f1 | paste -sd, -)
[ -n "$PORTS" ] || { echo 'No confirmed open TCP ports; inspect the inventory before continuing'; exit 0; }

nmap -p "$PORTS" -sV -sC --script-timeout 30s -T4 --host-timeout 90s \
  "$HOST" -oX /tmp/services.xml
nmap -p "$PORTS" -sV --script vulners --script-args mincvss=5.0 \
  --script-timeout 30s -T4 --host-timeout 120s "$HOST" -oX /tmp/cves.xml

# Let OS detection find both open and closed ports rather than restricting it to PORTS.
sudo nmap -O --osscan-guess --max-os-tries 1 --host-timeout 90s \
  "$HOST" -oN /tmp/os.txt
```

**Expected:** discovered TCP ports in `allports.gnmap`, per-port products/versions and default-script evidence in `services.xml`, candidate matches in `cves.xml`, and an OS estimate in `os.txt`. Keep diagnostics and note failed or timed-out phases.

**Next:** use protocol-specific scripts for uncertain products, and verify vendor/package applicability for CVEs. Treat `tcpwrapped`, generic service names, and speculative OS matches as unresolved evidence. Add the targeted [UDP workflow](#udp-network-sweep) if UDP coverage is needed; a TCP deep dive says nothing about UDP.

### Web Server Assessment

**Use when:** inventory found HTTP/TLS services and you need application hints, headers, certificate/cipher details, and candidate CVEs. **Requires:** reviewed web/NSE probes, vulners external access, and the intended hostname if the server uses virtual hosting. Enumeration and cipher testing generate multiple requests; they are not passive inspection.

```bash
nmap -p 80,443,8080,8443 -sV \
  --script http-enum,http-methods,http-headers,http-title,http-waf-detect,ssl-cert,ssl-enum-ciphers,vulners \
  --script-args mincvss=5.0 --script-timeout 30s \
  -T4 --host-timeout 120s 192.168.1.100 -oA /tmp/web-assessment
```

**Expected:** page titles and headers identify the served application; `http-enum` may list candidate paths; TLS scripts show certificate attributes and accepted protocols/ciphers; vulners may add version matches. A returned method/path or inferred WAF is not by itself a vulnerability.

**Next:** investigate relevant paths and unexpected TLS settings with a protocol-specific client. If an IP returns a default site or unexpected certificate, verify the intended HTTP Host/TLS SNI before interpreting it. For supported scripts, `--script-args 'http.host=app.example.com,tls.servername=app.example.com,mincvss=5.0'` supplies those names; verify with each script's help. A missing result can be a timeout, unsupported probe, or routing mismatch rather than proof of absence.

### Network Baseline (Save + Diff)

**Use when:** you want evidence of new/removed hosts, changed port states, or changed service versions between observations. **Requires:** `ndiff`, retained scan files, and matching target/port coverage, privileges, discovery options, and scanner vantage point. Choose separate observation filenames; do not overwrite the baseline when collecting current results.

```bash
# At the baseline observation: preserve all reported states, not just --open.
nmap --top-ports 1000 -T4 --host-timeout 20s \
  192.168.1.0/24 -oX /tmp/baseline.xml

# At a later observation: same scope/options, distinct output.
nmap --top-ports 1000 -T4 --host-timeout 20s \
  192.168.1.0/24 -oX /tmp/current.xml

ndiff /tmp/baseline.xml /tmp/current.xml
# Status 0 = unchanged, 1 = differences, 2 = error.
```

**Expected:** `+` and `-` records for changes, with surrounding host/port context in the full report. No differences means no observed differences in this scan's coverage, not proof that the entire network is unchanged.

For service-level comparison, run matched per-host snapshots after inventory. Keep the same explicit `PORTS` set at both observations; deriving only currently open ports would hide some state changes:

```bash
HOST=192.168.1.100
PORTS=22,80,443       # use the relevant inventory set; hold it constant for this comparison
nmap -p "$PORTS" -sV -T4 --host-timeout 60s "$HOST" -oX "/tmp/baseline-$HOST.xml"
# Later, repeat the same command with -oX "/tmp/current-$HOST.xml", then ndiff those files.
```

**Next:** verify significant changes with a targeted rescan and check scan diagnostics for timeouts or missing discovery results. A disappeared host may be asleep or filtered; a changed version banner may reflect a proxy or probe difference. Confirm the observation before treating it as a device/service removal or patch event.

### IoT / Smart Home Survey

**Use when:** you need to identify services advertised by local devices and correlate them with reachable ports. **Requires:** an authorized local Layer-2 network, a known interface for broadcasts, and device tolerance for service probes. Broadcast discovery reaches nearby devices beyond a single IP target; confirm that scope. Nmap normally excludes printer port 9100 from version probing to avoid printing probe data—do not casually override that protection with `--allports`.

```bash
sudo nmap -sn -PR --host-timeout 10s 192.168.1.0/24 \
  -oN /tmp/iot-discovery.txt -oG /tmp/hosts.gnmap || exit 1
awk '/Status: Up/ {print $2}' /tmp/hosts.gnmap > /tmp/live-hosts.txt

# Replace en0 with the authorized LAN interface; Linux names often differ.
sudo nmap -e en0 --script 'broadcast-upnp-info,broadcast-dns-service-discovery' \
  --script-timeout 30s -oN /tmp/iot-advertisements.txt

# Common TCP IoT ports; scan only when discovery supplied targets.
if [ -s /tmp/live-hosts.txt ]; then
  nmap -p 23,80,443,554,1883,5683,8080,8443,9100 -sV --host-timeout 60s \
    -iL /tmp/live-hosts.txt -oN /tmp/iot-scan.txt
fi
```

**Expected:** local discovery gives IP/MAC/vendor hints; broadcast output may supply device names, types, and service URLs; the TCP scan identifies reachable services. UDP discovery is separate—TCP port 5683 alone does not cover the usual UDP CoAP service. Advertised addresses/ports may be stale or unreachable from the scanning segment.

**Next:** correlate IP, MAC, hostname, advertised type, and actual service evidence rather than treating the vendor OUI as a device model. Follow up relevant UDP services with the [UDP sweep](#udp-network-sweep) and targeted service scripts. Missing broadcasts can reflect VLAN isolation, the wrong interface, disabled discovery, or timeouts.

### UDP Network Sweep

**Use when:** you need UDP coverage for DNS, DHCP, NTP, SNMP, UPnP, or mDNS after TCP inventory. **Requires:** authorized targets, raw-packet privileges, and permission for subsequent service queries. This is an active sweep. The selected-port list bounds scope, not total wall time.

```bash
# Step 1: Discover live hosts first (reuse from TCP sweep if already done)
sudo nmap -sn -PR -PE -PS22,80,443 --max-retries 1 --host-timeout 5s \
  192.168.1.0/24 -oG /tmp/hosts.gnmap || exit 1
awk '/Status: Up/ {print $2}' /tmp/hosts.gnmap > /tmp/live-hosts.txt
[ -s /tmp/live-hosts.txt ] || { echo 'No discovered targets; inspect discovery first'; exit 0; }

# Step 2: Targeted UDP scan across all live hosts — 120s host budget, not total runtime
# Never run a broad UDP scan on a subnet — do it per-host or with tight port targeting
sudo nmap -sU -p 53,67,69,111,123,137,161,500,1900,5353 \
  --max-retries 1 -T4 --host-timeout 120s \
  -iL /tmp/live-hosts.txt -oX /tmp/udp-sweep.xml

# Step 3: Select a host with SNMP evidence from the UDP report.
HOST=192.168.1.100
sudo nmap -sU -p 161 --script snmp-info,snmp-interfaces,snmp-sysdescr \
  --script-timeout 30s --host-timeout 60s "$HOST" -oX "/tmp/snmp-$HOST.xml"

# Step 4: mDNS sweep — actively queries device names and advertised services
sudo nmap -sU -p 5353 --script dns-service-discovery \
  --script-timeout 20s --host-timeout 30s -iL /tmp/live-hosts.txt -oN /tmp/mdns-sweep.txt
```

**Expected:** UDP port states in `udp-sweep.xml`; responding SNMP or mDNS scripts can add model/interface information or advertised service names. `open|filtered` is ambiguous, not confirmed open. An empty SNMP result may mean the community/auth settings are wrong rather than no SNMP service; do not silently switch to credential guessing.

**Next:** query an ambiguous port with a reviewed, service-specific probe and compare with the device's configuration. DHCP is often broadcast-based and does not behave like an ordinary per-host UDP request. mDNS visibility is link-local; a unicast sweep is not a complete cross-VLAN service inventory. Record host/script timeouts and retry only unresolved hosts as needed.

**What each port reveals on a home network:**
| Port | Service | What you get |
|------|---------|--------------|
| 53 | DNS | Router/Pi-hole — zone transfer attempts, version |
| 67 | DHCP | Subnet config, router IP, DNS servers |
| 123 | NTP | Device type, sometimes connected client list |
| 137 | NetBIOS | Windows hostnames, workgroup |
| 161 | SNMP | Device model, interfaces, routing table, firmware version |
| 1900 | UPnP/SSDP | IoT device type and model |
| 5353 | mDNS | Device name, OS, all advertised services |

---

## Interpreting Results

### Port States

| State | Meaning | What to do |
|-------|---------|------------|
| `open` | A service is actively accepting connections on this port | Investigate — run service detection and CVE lookup |
| `closed` | Port is reachable but no service is listening | Note it, not interesting unless tracking changes |
| `filtered` | Probes/responses were blocked or lost; open/closed is unknown | Investigate filtering or packet loss with scoped probes; `-Pn` bypasses discovery, not a port firewall |
| `open\|filtered` | Can't determine — common with UDP or FIN/NULL/Xmas scans | Use a service-specific UDP probe for UDP, or a TCP SYN/connect scan for ambiguous TCP results |
| `unfiltered` | Reachable but state unknown — seen with ACK scans | Run a SYN scan to confirm open/closed |

**Key distinction**: `filtered` does not mean the service doesn't exist. Filtering or packet loss can prevent a conclusive response. Many real services can be behind `filtered` ports. If a host looks unresponsive but you have reason to believe it's up, use `-Pn` to skip host discovery and scan anyway.

### When `-Pn` Is Needed

Hosts that block all ICMP and common probe ports will appear as "down" during discovery even when services are running. Signs you need `-Pn`:
- Host responds to ping from another machine but nmap shows it as down
- You know the IP is in use (from ARP table, DHCP lease, or prior scan) but nmap reports "Host seems down"
- Windows hosts with host firewall enabled often block ICMP

```bash
# Force scan a host regardless of ping response
nmap -Pn --top-ports 100 -T4 --host-timeout 30s 192.168.1.100
```

### Reading Service Banners

After `-sV` runs, service lines look like:
```
22/tcp  open  ssh      OpenSSH 8.9p1 Ubuntu 3ubuntu0.6 (Ubuntu Linux; protocol 2.0)
80/tcp  open  http     nginx 1.24.0
3306/tcp open  mysql   MySQL 8.0.35
```

- The version string (`OpenSSH 8.9p1`, `nginx 1.24.0`) is what vulners uses for CVE lookup — the more specific, the better
- `Ubuntu Linux` in the SSH banner reveals the OS even without `-O`
- Generic or missing versions (`http?`, `tcpwrapped`) mean nmap couldn't fingerprint — try `--version-intensity 9` or a targeted script

### `tcpwrapped` Ports

`tcpwrapped` means the TCP handshake completed but the service closed the connection before identifying itself — usually a security measure or a service that only responds to authorized clients. It's real but unidentified. Try:
```bash
# Grab the banner manually
nmap -p $PORT --script banner 192.168.1.100
```

### Reading vulners Output

```
22/tcp open  ssh  OpenSSH 8.4p1
| vulners:
|   cpe:/a:openbsd:openssh:8.4p1:
|     CVE-2023-38408  9.8  https://vulners.com/cve/CVE-2023-38408
|     CVE-2021-41617  7.0  https://vulners.com/cve/CVE-2021-41617
```

- First column after CVE ID is the CVSS score: 9.0+ critical, 7.0-8.9 high, 4.0-6.9 medium
- No vulners output can mean missing versions, no matches, a score filter, an API failure, or a script timeout; it is not a clean bill of health
- If you see no results on any port: confirm `-sV` is returning version strings before assuming the host is clean

---

## Troubleshooting

### "Host seems down"
```bash
nmap -Pn ...                  # Skip host discovery, treat as up
```

### Scan too slow
```bash
-T4 --min-rate 500 --max-retries 1 --top-ports 100
```

### Permission denied
```bash
sudo nmap ...                 # SYN/UDP/OS/raw scans need root
```

### Filtered ports (firewall)
```bash
sudo nmap -sA ...             # ACK scan to map firewall rules
nmap --source-port 53 ...     # Use trusted source port
sudo nmap -sF ...             # FIN scan to bypass simple firewalls
```

### vulners returns no CVEs
```
- Version not detected: increase --version-intensity
- API unreachable: check internet access from container
- CVEs below threshold: lower mincvss or remove the argument
```

---

## Quick Reference

These are invocation reminders, not standalone evidence of a complete assessment. Set a nonempty `PORTS` list before using it; see the corresponding workflow for prerequisites, diagnostics, and interpretation.

| Goal | Command |
|------|---------|
| Find live hosts | `sudo nmap -sn -PR 192.168.1.0/24 --max-retries 1` |
| Quick port check | `nmap --top-ports 100 -T4 192.168.1.100` |
| All ports | `sudo nmap -p- -T4 --min-rate 1000 --host-timeout 120s 192.168.1.100` |
| Service versions | `nmap -p $PORTS -sV -T4 --host-timeout 90s 192.168.1.100` |
| Default scripts | `nmap -p $PORTS -sC -T4 192.168.1.100` |
| OS detection | `sudo nmap -O --osscan-guess 192.168.1.100` |
| CVE lookup | `nmap -p $PORTS -sV --script vulners --script-args mincvss=5.0 192.168.1.100` |
| Built-in vuln scan | `nmap --script vuln --host-timeout 180s 192.168.1.100` |
| Both vuln + CVE | `nmap -p $PORTS -sV --script "vuln,vulners" --script-args mincvss=5.0 192.168.1.100` |
| EternalBlue | `nmap -p 445 --script smb-vuln-ms17-010 192.168.1.100` |
| UDP scan | `sudo nmap -sU --top-ports 20 -T4 --host-timeout 120s 192.168.1.100` |
| Stealth scan | `sudo nmap -sS -T2 --max-retries 1 192.168.1.100` |
| IDS evasion | `sudo nmap -sS -T1 -f -D RND:5 --source-port 53 192.168.1.100` |
| Zombie scan | `sudo nmap -sI zombie:80 192.168.1.100` |
| Baseline diff | `ndiff /tmp/baseline.xml /tmp/current.xml` |
| Proxy-supported version/NSE connections (port scan remains direct) | `nmap --proxies socks4://127.0.0.1:9050 -sT -Pn -p 80 -sV --script http-title 192.168.1.100` |
| SMB full enum | `nmap -p 445 --script "smb-enum-*,smb-vuln-*" 192.168.1.100` |
| IoT/SCADA | `nmap -p 102,502,47808 --script "*-info,modbus-*" 192.168.1.100` |
