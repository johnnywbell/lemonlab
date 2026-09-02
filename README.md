# LemonLab
 
A self-directed infrastructure lab running on Proxmox VE, built to develop and demonstrate the sysadmin, monitoring, and security-conscious operational skills expected of infrastructure and engineering roles.
 
Started 29 August 2026 on a repurposed Lenovo ThinkCentre M710e (Intel i5-7400, 24GB RAM). Documented per session in [`notes/`](notes/) for reference and further study. This repository is a portfolio, not based on automation — the lab exists on hardware; this repo is essentially a written guide on the structure and purpose of it.
 
## Stack
 
- **Hypervisor:** Proxmox VE 9.2
- **Containers:** 7 unprivileged LXCs
- **DNS filtering:** Pi-hole with StevenBlack blocklist
- **Uptime monitoring:** Uptime Kuma
- **Unified dashboard:** Homepage
- **Observability:** Prometheus + Grafana with node_exporter (all hosts) and pve-exporter (hypervisor)
- **Media services:** Prowlarr, Lidarr
- **Remote access:** Tailscale mesh VPN (host + phone nodes)
- **Backups:** Nightly Proxmox backup jobs to external USB (to be upgraded in the future), ZSTD-compressed, layered retention (3 last / 7 daily / 4 weekly / 3 monthly), restore verified end-to-end
## Practices in place
 
- SSH key-only authentication across every host; password SSH globally disabled
- TOTP MFA on every service that supports it (Proxmox web UI, Pi-hole, Uptime Kuma)
- Network-wide DNS filtering for ads, trackers, telemetry, and known malware/phishing domains
- Least-privilege API accounts using built-in Proxmox roles (PVEAuditor for read-only monitoring access)
- Credentials managed in a password manager with a consistent naming conventions to making it easily searchable
- Per-session technical documentation covering non-obvious decisions and the rationale behind them
## Scope statement
 
Real and running as of latest update:
 
- Proxmox VE hypervisor, seven LXCs, all healthy
- Full monitoring stack rendering live metrics
- Nightly automated backups with a tested restore procedure
- Off-network access via Tailscale, proven end-to-end from mobile data
Planned but not yet built:
 
- Full media stack completion (blocked pending VPN and storage decisions)
- NAS build (hardware selection finalised, parts acquisition in pipeline)
- VLAN segmentation via pfSense/OPNsense (blocked pending structural Ethernet install)
- Security Onion detection lab (blocked pending network buildout)
- Frigate NVR (blocked pending cameras and VLAN)
Not a production environment. No HA, no on-call rotation, no formal change control, no dedicated secrets management, no audit log retention. The lab exists to develop and demonstrate transferable patterns, not to run business-critical services.
 
## Repository structure
 
```
lemonlab/
├── README.md              This file
├── LICENSE                MIT
└── notes/                 Session-by-session technical writeups
```
 
The `notes/` folder contains the detailed writeups. Each note covers a working session: what was built, what was learned, what went wrong, what got documented for future reference. Written primarily as a personal reference, made public as portfolio evidence.
 
## Purpose
 
Built as part of a career pivot toward infrastructure and security engineering. The lab is deliberately built on-premises with plain LXC rather than cloud services or heavy automation frameworks, prioritising understanding of fundamentals over convenience until it becomes plausible. Cloud and orchestration concepts are being developed separately.
 
## License
 
MIT — see [LICENSE](LICENSE).
