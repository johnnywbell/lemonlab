# LemonLab session notes

Session-by-session technical writeups of the LemonLab build. Written primarily as personal reference, made public as portfolio evidence.

Each note covers a working session: what was built, what was learned, what went wrong, and the rationale behind non-obvious decisions. Notes are chronological.

## Contents

- **[01-prometheus-grafana-build.md](01-prometheus-grafana-build.md)** — Building the observability stack from scratch. Prometheus and Grafana in separate LXCs, node_exporter across all hosts, pve-exporter for hypervisor metrics via the Proxmox API. Covers the exporter pattern, relabelling rules for indirect exporters, and community dashboard imports.

- **[02-grafana-widget-arr-planning.md](02-grafana-widget-arr-planning.md)** — Integrating Grafana into Homepage as a live widget (a 45-minute detour through Grafana's tiered permission model — service account tokens vs server admin), a Wi-Fi diagnostic session, an honest look at "dashboards for the sake of it," and the initial *arr stack deployment (Prowlarr, Lidarr).

- **[03-media-services-ssh-hardening.md](03-media-services-ssh-hardening.md)** — Wiring the *arr services into Homepage under a new Media group, followed by a full SSH hardening pass across the lab: key-only auth everywhere, password SSH globally disabled, client-side aliases for daily-use convenience.

- **[04-tailscale-termux-cv-lessons.md](04-tailscale-termux-cv-lessons.md)** — Two threads. Technical: Tailscale rollout for off-network access (Proxmox + phone as nodes, tested over mobile data), followed by Termux install and a dedicated phone SSH key so shell access works from anywhere with signal. Meta: a CV-writing sidebar where the first version got flagged for overclaiming and had to be rewritten — that lesson generalises well beyond CV work.

- **[05-github-publishing.md](05-github-publishing.md)** — First real Git use. Publishing this repository. Includes the mental model correction (GitHub isn't just for code), README-as-portfolio thinking, dedicated SSH key for GitHub (separate from lab access), and the mechanics of `git config` through `git push` with explanations of what each step actually does.

## Reading order

Chronological (01 → 05) is the intended order — later notes reference concepts and decisions from earlier ones. But each note is standalone enough to read out of order if a specific topic is of interest.

## Sanitisation

Internal network addresses have been redacted. The published versions use generic placeholders (`192.168.x.100`, `192.168.x.101`, etc.) in place of the original internal addresses. The last octet is preserved because it identifies which service is being discussed; the identifying subnet is removed. Similarly, the specific Tailscale IP is replaced with `100.x.x.x`. No credentials, tokens, or API keys are present in these notes — placeholders were used from the start.
