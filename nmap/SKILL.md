---
name: nmap
description: Network reconnaissance using nmap and nmap-vulners. Covers host discovery, port scanning, service detection, OS fingerprinting, NSE scripts, CVE lookup via vulners.com, and timeout-safe progressive scanning for reliable agent execution.
---

# Nmap Network Reconnaissance

You are performing network reconnaissance using nmap. This skill covers the full recon pipeline from discovery through vulnerability identification, with agent-safe patterns for reliable execution.

## Installed Tools

| Tool | Purpose |
|------|---------|
| `nmap` | Port scanner, service detector, NSE script engine |
| `vulners.nse` | NSE script — queries vulners.com API for CVEs by service version |
| `ndiff` | Bundled with nmap — diffs two XML scan results |

### Dockerfile Setup

```dockerfile
# Install nmap
RUN apt-get install -y nmap

# Grant agent user raw socket access for privileged scan types (SYN, UDP, OS detection)
# Not required if running as root
RUN setcap cap_net_raw,cap_net_admin+eip /usr/bin/nmap

# Install vulners NSE script for CVE lookup via vulners.com API
RUN curl -fsSL https://raw.githubusercontent.com/vulnersCom/nmap-vulners/master/vulners.nse \
    -o /usr/share/nmap/scripts/vulners.nse \
    && nmap --script-updatedb
```

---

## Execution Guidelines

- Always phase scans: discover → ports → services → CVEs. Never combine them on a subnet.
- Always set `--host-timeout` on any scan covering more than one host.
- Always extract open ports from a fast scan before running `-sV` or scripts.
- Always use `/tmp` for output files unless instructed otherwise.
- Use `--open` to suppress closed/filtered noise from output.

---

## Agent-Optimized Scanning

Long-running scans that don't complete are the #1 failure mode for agent-driven nmap. The rules below guarantee predictable completion times.

### The Cardinal Rules

1. **Never combine discovery + service detection + scripts in one command on a network range.** Split them into phases.
2. **Always extract open ports from a fast scan before running service detection.** Never run `-sV` on all 65535 ports.
3. **Scan one host at a time for deep work.** Loop over a host list rather than passing an entire subnet to a slow scan.
4. **Every command touching more than one host must have `--host-timeout`.** Without it, one unresponsive host can block the entire scan indefinitely.
5. **Prefer `--top-ports N` over `-p-`.** Top 100 ports covers ~98% of real-world services. Use full port scan only when specifically needed for a single host.

### Phase-Based Time Budget

| Phase | Scope | Typical Command | Max Time |
|-------|-------|-----------------|----------|
| 1. Host discovery | /24 network | `-sn -PR -PE` | ~30s |
| 2. Quick port check | single host | `--top-ports 100 -T4` | ~15s |
| 3. Targeted port scan | single host, known ports | `-p $PORTS -T4` | ~30s |
| 4. Service detection | single host, open ports only | `-p $PORTS -sV` | ~60s |
| 5. CVE lookup | single host, open ports only | `-p $PORTS -sV --script vulners` | ~90s |
| 6. Vuln scripts | single host, specific ports | `-p $PORTS --script vuln` | ~120s |

**Never run Phase 4, 5, or 6 on a subnet range.** Always discover first, then process hosts individually.

### Safe Network Sweep Pattern

When asked to scan a network, always do this in two separate commands — not one:

```bash
# Command 1: fast discovery (30s max)
sudo nmap -sn -PR -PE -PS22,80,443 \
  --max-retries 1 --host-timeout 5s \
  192.168.1.0/24 -oG /tmp/hosts.gnmap

# Extract live hosts
grep "Up" /tmp/hosts.gnmap | awk '{print $2}' > /tmp/live-hosts.txt
cat /tmp/live-hosts.txt

# Command 2: top-1000 port scan on live hosts only (20s per host, host-timeout enforced)
# --open suppresses closed/filtered noise — only shows what's actually reachable
nmap --top-ports 1000 -T4 --open \
  --max-retries 2 --host-timeout 20s \
  -iL /tmp/live-hosts.txt \
  -oG /tmp/ports.gnmap
```

### Per-Host Processing Loop

For service detection or CVE lookup, process one host at a time. This keeps each command bounded and results readable immediately:

```bash
# Extract open ports for a specific host
PORTS=$(grep "^Host: 192.168.1.X " /tmp/ports.gnmap \
  | grep -oP '\d+/open' | cut -d'/' -f1 | tr '\n' ',' | sed 's/,$//')

# If no open ports found, skip this host
if [ -z "$PORTS" ]; then
  echo "No open ports on 192.168.1.X, skipping"
fi

# Service detection — only on extracted open ports (60s max)
nmap -p "$PORTS" -sV -T4 --open --host-timeout 60s \
  192.168.1.X -oN /tmp/services-192.168.1.X.txt

# CVE lookup — only on extracted open ports (90s max)
nmap -p "$PORTS" -sV --script vulners --script-args mincvss=5.0 \
  --script-timeout 30s -T4 --open --host-timeout 90s \
  192.168.1.X -oX /tmp/cves-192.168.1.X.xml
```

Repeat this block per host. Read each output file after its scan completes before moving to the next host.

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

Before running a scan, estimate time to decide if it's safe:

```
hosts × ports × 0.01s ≈ minimum scan time (no service detection)
hosts × ports × 0.05s ≈ with -sV service detection
hosts × open_ports × 2s ≈ with --script vulners
```

Examples:
- 10 hosts × 100 ports = ~10s for port scan
- 10 hosts × 100 ports with -sV = ~50s (borderline, use `--host-timeout`)
- 1 host × 1000 ports with -sV = ~50s (fine)
- 256 hosts × 1000 ports = ~2560s (never do this)

**Rule of thumb**: If `hosts × ports > 5000`, split the scan.

### Guaranteed-Completion Scan Profiles

Copy these verbatim for reliable agent execution:

```bash
# PROFILE: network-alive — find live hosts in a /24 (≤30s)
sudo nmap -sn -PR -PE -PS22,80,443 --max-retries 1 --host-timeout 5s \
  $NETWORK -oG /tmp/alive.gnmap

# PROFILE: host-quick — top 100 ports on one host (≤15s)
nmap --top-ports 100 -T4 --open --max-retries 2 --host-timeout 15s \
  $HOST -oG /tmp/quick-$HOST.gnmap

# PROFILE: network-ports — top 1000 ports across all live hosts (≤20s/host)
nmap --top-ports 1000 -T4 --open --max-retries 2 --host-timeout 20s \
  -iL /tmp/live-hosts.txt -oG /tmp/ports.gnmap

# PROFILE: host-ports — all 65535 ports on a single host (≤120s)
# Only use on one host at a time — never on a host list
sudo nmap -p- -T4 --min-rate 1000 --open --max-retries 2 --host-timeout 120s \
  $HOST -oG /tmp/allports-$HOST.gnmap

# PROFILE: host-services — service detection on known open ports (≤60s)
nmap -p $PORTS -sV --version-intensity 5 -T4 --open --host-timeout 60s \
  $HOST -oN /tmp/services-$HOST.txt

# PROFILE: host-cves — CVE lookup on known open ports (≤90s)
nmap -p $PORTS -sV --script vulners --script-args mincvss=5.0 \
  --script-timeout 30s -T4 --open --host-timeout 90s \
  $HOST -oX /tmp/cves-$HOST.xml

# PROFILE: host-vulns — built-in vuln scripts on specific ports (≤120s)
nmap -p $PORTS --script vuln --script-timeout 60s \
  -T4 --open --host-timeout 120s \
  $HOST -oN /tmp/vulns-$HOST.txt

# PROFILE: host-web — full web assessment on one host (≤120s)
nmap -p 80,443,8080,8443 -sV \
  --script http-enum,http-headers,ssl-cert,ssl-enum-ciphers,vulners \
  --script-args mincvss=5.0 --script-timeout 30s \
  -T4 --open --host-timeout 120s \
  $HOST -oN /tmp/web-$HOST.txt

# PROFILE: host-smb — SMB enumeration + vuln check on one host (≤90s)
nmap -p 445 -sV \
  --script smb-os-discovery,smb-security-mode,smb-enum-shares,smb-vuln-ms17-010,vulners \
  --script-timeout 30s -T4 --open --host-timeout 90s \
  $HOST -oN /tmp/smb-$HOST.txt

# PROFILE: host-udp — high-value UDP ports on one host (≤120s)
sudo nmap -sU -p 53,67,69,111,123,137,161,500,1900,5353 \
  --max-retries 1 -T4 --host-timeout 120s \
  $HOST -oX /tmp/udp-$HOST.xml
```

---

## The Full Recon Kill Chain

The recommended end-to-end workflow for a complete network assessment. Steps 1-2 run against the whole network. Steps 3-6 run **per host** — never against the full subnet.

```bash
# Step 1: Discover live hosts — whole network (≤30s for /24)
sudo nmap -sn -PR -PE -PS22,80,443 --max-retries 1 --host-timeout 5s \
  192.168.1.0/24 -oG /tmp/hosts.gnmap

# Step 2: Extract live IPs
grep "Up" /tmp/hosts.gnmap | awk '{print $2}' > /tmp/live-hosts.txt
cat /tmp/live-hosts.txt

# Step 3: Quick port scan — whole network, top 1000, host-timeout enforced (≤20s/host)
# This gives a fast picture of the network without running forever.
# For a full port scan (-p-), do it per host in step 4.
nmap --top-ports 1000 -T4 --open \
  --max-retries 2 --host-timeout 20s \
  -iL /tmp/live-hosts.txt -oG /tmp/ports.gnmap

# ── per-host steps below — repeat for each IP of interest ──────────────────

# Step 4 (optional): Full port scan on a single target (≤120s)
# Only run this when top-1000 isn't enough and you need all 65535 ports.
sudo nmap -p- -T4 --min-rate 1000 --host-timeout 120s --open \
  192.168.1.X -oG /tmp/allports-192.168.1.X.gnmap

# Extract open ports from whichever scan you ran (step 3 or step 4)
PORTS=$(grep "^Host: 192.168.1.X " /tmp/ports.gnmap \
  | grep -oP '\d+/open' | cut -d'/' -f1 | tr '\n' ',' | sed 's/,$//')

# Step 5: Service detection on extracted open ports only (≤60s)
nmap -p "$PORTS" -sV --version-intensity 5 -T4 --host-timeout 60s --open \
  192.168.1.X -oX /tmp/services-192.168.1.X.xml -oN /tmp/services-192.168.1.X.txt

# Step 6: CVE lookup via vulners (≤90s)
nmap -p "$PORTS" -sV --script vulners --script-args mincvss=5.0 \
  --script-timeout 30s -T4 --host-timeout 90s \
  192.168.1.X -oX /tmp/cves-192.168.1.X.xml

# Step 7: OS detection (≤30s)
sudo nmap -p "$PORTS" -O --osscan-guess --max-os-tries 1 \
  192.168.1.X -oN /tmp/os-192.168.1.X.txt
```

---

## CVE Lookup with nmap-vulners

The `vulners` NSE script queries the vulners.com API with detected service/version banners and returns known CVEs with CVSS scores. **Requires `-sV` to detect versions first.**

### Basic Usage

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

The first number is the CVSS score. Anything 9.0+ is critical, 7.0-8.9 is high, 4.0-6.9 is medium.

### Notes
- Requires internet access from the scanning host to reach vulners.com
- Results depend on version accuracy — use `--version-intensity 5` or higher for best results
- Combine with `-sC` default scripts for maximum context

---

## Tracking Changes with ndiff

`ndiff` compares two nmap XML scans and shows what changed — new hosts, new ports, closed ports, version changes.

```bash
# Save baseline — port-level, no service detection (safe on subnets)
nmap --top-ports 1000 -T4 --open --host-timeout 20s \
  192.168.1.0/24 -oX /tmp/baseline.xml

# Later, save current state
nmap --top-ports 1000 -T4 --open --host-timeout 20s \
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
nmap -p T:80,U:53             # TCP 80, UDP 53
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

Extract MAC addresses and vendors from a discovery scan:
```bash
# Normal output format (-oN) uses "MAC Address:"
sudo nmap -sn 192.168.1.0/24 | grep -E "report for|MAC Address"

# Grepable output format (-oG) uses "MAC:" — different field name
grep "MAC:" /tmp/hosts.gnmap

# To get IP + MAC together from grepable output:
grep "Status: Up" /tmp/hosts.gnmap | awk '{print $2}' > /tmp/live.txt
grep "MAC:" /tmp/hosts.gnmap
```

**Note**: MAC addresses are only visible when scanning from the same Layer 2 network (same subnet). Scanning across a router will only show the router's MAC.

---

## Service and Version Detection

```bash
nmap -sV 192.168.1.100                    # Basic version detection
nmap -sV --version-intensity 0 ...        # Light (fastest)
nmap -sV --version-intensity 5 ...        # Default (balanced)
nmap -sV --version-intensity 9 ...        # Try all probes (slow)
nmap -p 22,80,443 -sV -T4 --host-timeout 90s 192.168.1.100   # Targeted (recommended)
```

---

## OS Detection

```bash
sudo nmap -O 192.168.1.100                    # Basic
sudo nmap -O --osscan-guess --max-os-tries 1  # Aggressive guess, single attempt
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

| Scan Type | Flags | Duration |
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
nmap --script default ...     # Safe, useful scripts (-sC)
nmap --script discovery ...   # Service discovery
nmap --script safe ...        # Won't harm target
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
nmap -p 1720 --script h323-info 192.168.1.100                         # H.323
nmap -p 5038 --script asterisk-info 192.168.1.100                     # Asterisk AMI
```

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

```bash
# Extract open ports for a specific host
grep "^Host: 192.168.1.100 " /tmp/scan.gnmap \
  | grep -oP '\d+/open' | cut -d'/' -f1 | tr '\n' ',' | sed 's/,$//'

# Extract all live hosts
grep "Up" /tmp/hosts.gnmap | awk '{print $2}'

# Extract all open ports across all hosts
grep "open" /tmp/scan.gnmap | grep -oP '\d+/open/tcp' | cut -d'/' -f1 | sort -un
```

### Parsing XML Output with Python

```bash
# Get all open ports from XML
python3 -c "
import xml.etree.ElementTree as ET
tree = ET.parse('/tmp/scan.xml')
for host in tree.findall('.//host'):
    addr = host.find('address').get('addr')
    ports = [p.get('portid') for p in host.findall('.//port') if p.find('state').get('state') == 'open']
    if ports:
        print(f'{addr}: {\", \".join(ports)}')
"

# Get service versions from XML
python3 -c "
import xml.etree.ElementTree as ET
tree = ET.parse('/tmp/scan.xml')
for host in tree.findall('.//host'):
    addr = host.find('address').get('addr')
    for port in host.findall('.//port'):
        state = port.find('state').get('state')
        if state == 'open':
            portid = port.get('portid')
            svc = port.find('service')
            if svc is not None:
                name = svc.get('name','')
                product = svc.get('product','')
                version = svc.get('version','')
                print(f'{addr}:{portid} {name} {product} {version}'.strip())
"
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
sudo nmap -g 80 ...              # Same as above
```

### Timing Evasion
```bash
sudo nmap -T1 --scan-delay 5s --max-parallelism 1 ...  # IDS evasion mode
sudo nmap -T2 --randomize-hosts 192.168.1.0/24 ...     # Random host order
```

### Full Evasion Combo
```bash
sudo nmap -sS -T1 -f --mtu 16 -D RND:5 --source-port 53 \
  --data-length 100 --scan-delay 2s 192.168.1.100
```

### Idle/Zombie Scan (Ultimate Stealth)
```bash
# Find a zombie host (needs incremental IP ID)
nmap --script ipidseq 192.168.1.0/24  # Look for "Incremental"

# Scan using zombie — your IP never touches the target
sudo nmap -sI zombie-ip:80 target-ip
```

---

## Advanced Techniques

### Broadcast Discovery (Find Hidden Devices)
```bash
nmap --script "broadcast-*" --script-args broadcast.timeout=30
nmap --script broadcast-upnp-info                       # UPnP/IoT
nmap --script broadcast-dhcp-discover                   # DHCP info
nmap --script broadcast-netbios-master-browser          # NetBIOS
nmap --script dns-service-discovery -p 5353 224.0.0.251 # mDNS/Bonjour
```

### SSL/TLS Deep Inspection
```bash
nmap -p 443 --script ssl-cert,ssl-enum-ciphers,ssl-heartbleed,ssl-dh-params 192.168.1.100
nmap -p 443 --script ssl-cert --script-args ssl-cert.full 192.168.1.100  # Full cert chain
```

### HTTP Deep Dive
```bash
nmap -p 80 --script http-enum,http-backup-finder,http-waf-detect,http-sitemap-generator 192.168.1.100
nmap -p 80 --script http-grep --script-args \
  'http-grep.match=[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}' 192.168.1.100
```

### SMB Full Enumeration
```bash
nmap -p 445 --script \
  smb-enum-domains,smb-enum-groups,smb-enum-processes,smb-enum-services,\
  smb-enum-sessions,smb-enum-shares,smb-enum-users,\
  smb-os-discovery,smb-security-mode,smb-system-info \
  192.168.1.100
```

### Large Network Optimization
```bash
# Phase 1: Parallel host discovery
sudo nmap -sn -PR --min-hostgroup 256 --min-parallelism 100 10.0.0.0/16 -oG /tmp/hosts.gnmap

# Phase 2: Distributed port scanning per subnet
for i in $(seq 0 255); do
  nmap --top-ports 100 -T4 --max-retries 1 10.0.$i.0/24 -oG "/tmp/subnet_$i.gnmap" &
done
wait
cat /tmp/subnet_*.gnmap > /tmp/all_ports.gnmap
```

### Scan Through Proxies
```bash
nmap --proxy socks4://127.0.0.1:9050 -sT -Pn 192.168.1.100   # SOCKS4 / Tor
nmap --proxy http://proxy:8080 -sT -Pn 192.168.1.100          # HTTP proxy
```

### Custom Packet Crafting
```bash
sudo nmap --scanflags URGACKPSHRSTSYNFIN 192.168.1.100   # All TCP flags
sudo nmap --data-length 50 ...                           # Append random data
sudo nmap --badsum ...                                   # Bad checksums (test IDS)
```

### IPv6 Scanning
```bash
nmap -6 fe80::1                                   # Single IPv6 target
nmap -6 --top-ports 100 -T4 fe80::1              # Top ports on IPv6 host
nmap -6 -sn ff02::1                               # Multicast ping — find all IPv6 hosts on LAN
nmap -6 --script ipv6-node-info fe80::1           # Node info (hostname, addresses)
nmap --script targets-ipv6-multicast-*            # Discover IPv6 hosts from IPv4 scan
```
Use `-6` whenever the target is an IPv6 address. Most scan types and scripts work identically.

### Honeypot Detection
```bash
# Dedicated script check
nmap --script honeypot-detection 192.168.1.100

# Real systems have quirks — suspiciously clean responses suggest a honeypot
nmap -sV --version-all -p- --host-timeout 180s 192.168.1.100

# Honeypots often respond too fast or too consistently — check response timing
nmap -sV --script banner --script-args banner.timeout=100ms 192.168.1.100
```
Signs of a honeypot: every port returns the same banner, version strings are unusually generic, or response times are implausibly uniform across all services.

---

## Common Workflows

### Quick Network Snapshot (2-3 min for /24)
```bash
# Discover hosts
sudo nmap -sn -PR -PE -PS22,80,443 --max-retries 1 --host-timeout 3s \
  192.168.1.0/24 -oG /tmp/hosts.gnmap

# Top 100 ports on all live hosts
grep "Up" /tmp/hosts.gnmap | awk '{print $2}' | \
  xargs nmap --top-ports 100 -T4 --host-timeout 30s -oN /tmp/quick-scan.txt
```

### Single Host Deep Dive
```bash
# All ports
sudo nmap -p- -T4 --min-rate 1000 --host-timeout 120s 192.168.1.100 -oG /tmp/allports.gnmap

# Get open ports
PORTS=$(grep "^Host: 192.168.1.100 " /tmp/allports.gnmap \
  | grep -oP '\d+/open' | cut -d'/' -f1 | tr '\n' ',' | sed 's/,$//')

# Service detection
nmap -p $PORTS -sV -sC -T4 --host-timeout 90s 192.168.1.100 -oX /tmp/services.xml

# CVE lookup
nmap -p $PORTS -sV --script vulners --script-args mincvss=5.0 \
  -T4 --host-timeout 120s 192.168.1.100 -oX /tmp/cves.xml

# OS detection
sudo nmap -p $PORTS -O --osscan-guess 192.168.1.100 -oN /tmp/os.txt
```

### Web Server Assessment
```bash
nmap -p 80,443,8080,8443 -sV \
  --script http-enum,http-methods,http-headers,http-title,http-waf-detect,\
  ssl-cert,ssl-enum-ciphers,vulners \
  --script-args mincvss=5.0 \
  -T4 --host-timeout 120s 192.168.1.100
```

### Network Baseline (Save + Diff)
```bash
# Save baseline — port-level only, no service detection (fast, safe on subnets)
nmap --top-ports 1000 -T4 --open --host-timeout 20s \
  192.168.1.0/24 -oX /tmp/baseline.xml

# Later: current state
nmap --top-ports 1000 -T4 --open --host-timeout 20s \
  192.168.1.0/24 -oX /tmp/current.xml

# Diff — shows new hosts, new ports, removed hosts, closed ports
ndiff /tmp/baseline.xml /tmp/current.xml | grep "^[+-]"
```

For a service-level baseline, run per-host after discovery:
```bash
# Per-host service snapshot (add to baseline after port scan)
nmap -p "$PORTS" -sV -T4 --host-timeout 60s --open \
  192.168.1.X -oX /tmp/baseline-192.168.1.X.xml
```

### IoT / Smart Home Survey
```bash
# Discover IoT devices
sudo nmap -sn -PR 192.168.1.0/24 -oG /tmp/hosts.gnmap
# Find UPnP / Bonjour / mDNS devices
nmap --script "broadcast-upnp-info,broadcast-dns-service-discovery"
# Scan common IoT ports
nmap -p 23,80,443,554,1883,5683,8080,8443,9100 -sV --host-timeout 60s \
  -iL /tmp/live-hosts.txt -oN /tmp/iot-scan.txt
```

### UDP Network Sweep
```bash
# Step 1: Discover live hosts first (reuse from TCP sweep if already done)
sudo nmap -sn -PR -PE -PS22,80,443 --max-retries 1 --host-timeout 5s \
  192.168.1.0/24 -oG /tmp/hosts.gnmap
grep "Up" /tmp/hosts.gnmap | awk '{print $2}' > /tmp/live-hosts.txt

# Step 2: Targeted UDP scan across all live hosts — high-value ports only (≤120s/host)
# Never run a broad UDP scan on a subnet — do it per-host or with tight port targeting
sudo nmap -sU -p 53,67,69,111,123,137,161,500,1900,5353 \
  --max-retries 1 -T4 --host-timeout 120s \
  -iL /tmp/live-hosts.txt -oX /tmp/udp-sweep.xml

# Step 3: For hosts with SNMP open, enumerate device info
sudo nmap -sU -p 161 --script snmp-info,snmp-interfaces,snmp-sysdescr \
  --host-timeout 60s 192.168.1.X -oX /tmp/snmp-192.168.1.X.xml

# Step 4: mDNS sweep — reveals device names and services passively
sudo nmap -sU -p 5353 --script dns-service-discovery \
  --host-timeout 30s -iL /tmp/live-hosts.txt -oN /tmp/mdns-sweep.txt
```

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
| `filtered` | Firewall is dropping/blocking packets — nmap can't tell if open or closed | May be worth probing with `-Pn`, source port tricks, or ACK scan to map the firewall |
| `open\|filtered` | Can't determine — common with UDP or FIN/NULL/Xmas scans | Try TCP connect scan or different probe to confirm |
| `unfiltered` | Reachable but state unknown — seen with ACK scans | Run a SYN scan to confirm open/closed |

**Key distinction**: `filtered` does not mean the service doesn't exist. A firewall is actively blocking the probe. Many real services hide behind `filtered` ports. If a host looks unresponsive but you have reason to believe it's up, use `-Pn` to skip host discovery and scan anyway.

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
- No vulners output on a port = either no version was detected or no known CVEs at that version
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
| Through Tor | `nmap --proxy socks4://127.0.0.1:9050 -sT -Pn 192.168.1.100` |
| SMB full enum | `nmap -p 445 --script smb-enum-*,smb-vuln-* 192.168.1.100` |
| IoT/SCADA | `nmap -p 102,502,47808 --script "*-info,modbus-*" 192.168.1.100` |
