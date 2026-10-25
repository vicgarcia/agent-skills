# nmap

Network reconnaissance for authorized security testing using [nmap](https://nmap.org) and the nmap-vulners NSE script, with agent-safe, timeout-aware scanning patterns.

The skill itself is [`SKILL.md`](SKILL.md). All content lives in that one file; there are no accessory reference files.

> **Authorized use only.** This skill is for pentesting engagements, CTF competitions, security research, and defensive use on systems you own or have explicit written permission to scan.

## What it covers

- **Host discovery, port scanning, service/version detection, and OS fingerprinting.**
- **Target and port specification**, scan types, and timing/performance tuning.
- **NSE scripts** and CVE lookup via nmap-vulners.
- **Output formats** and tracking changes between scans with `ndiff`.
- **Agent-optimized, progressive scanning** patterns for reliable execution within tool timeouts.
- **Interpreting results** and troubleshooting.

## Requirements

`nmap` installed. Raw-socket scan types need elevated privileges (or `setcap` for non-root use); CVE lookup needs the vulners NSE script. See the skill for the Docker/`setcap` setup.

## Canonical documentation

- [Nmap reference guide](https://nmap.org/book/man.html)
- [Nmap Scripting Engine (NSE)](https://nmap.org/book/nse.html)
- [nmap-vulners](https://github.com/vulnersCom/nmap-vulners)
