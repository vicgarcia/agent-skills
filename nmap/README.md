# nmap

Network reconnaissance for authorized security testing using [nmap](https://nmap.org) and the nmap-vulners NSE script, with agent-safe, timeout-aware scanning patterns.

The skill itself is [`SKILL.md`](SKILL.md). All content lives in that one file; there are no accessory reference files.

> **Authorized use only.** This skill is for pentesting engagements, CTF competitions, security research, and defensive use on systems you own or have explicit written permission to scan.

## What it covers

- **Goal-based navigation** — jump from an investigation goal to a worked workflow and its detailed command reference.
- **Host discovery, port scanning, service/version detection, and OS fingerprinting.**
- **Target and port specification**, scan types, and timing/performance tuning.
- **NSE scripts** and CVE lookup via nmap-vulners.
- **Output formats** and tracking changes between scans with `ndiff`.
- **Agent-optimized, progressive scanning** patterns, with per-host budgets and explicit timeout/completeness caveats.
- **Worked assessments** — network snapshots, single-host deep dives, web/TLS, baseline comparisons, IoT, and UDP, including prerequisites, expected evidence, interpretation, and follow-up.
- **Advanced references** — firewall/IDS techniques, proxy limitations, IPv6, service enumeration, and evidence-based honeypot investigation.
- **Portable result extraction** — shell recipes for confirmed open TCP ports and Python XML examples for IP/MAC/vendor and service correlation.
- **Interpreting results** and troubleshooting.

## Requirements

`nmap` installed; examples use Bash, and the XML parsing recipes use Python 3. Raw-socket scan types need appropriate privileges (`setcap` setup is Linux-specific). CVE lookup needs the vulners NSE script and internet access; it sends product/version information to vulners.com. `ndiff` may need a separate package. Script examples were checked against the Nmap 7.99 catalog; verify availability and arguments on other versions. See the skill for Docker/`setcap` details.

## Canonical documentation

- [Nmap reference guide](https://nmap.org/book/man.html)
- [Nmap Scripting Engine (NSE)](https://nmap.org/book/nse.html)
- [nmap-vulners](https://github.com/vulnersCom/nmap-vulners)
