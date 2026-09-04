# LemonLab: Third Day Study Notes

*Building on the previous nights: Prometheus + Grafana + node_exporter observability stack, the Grafana-on-Homepage debugging journey (that ended in an architectural correction), a real Wi-Fi diagnostic session, honest thinking about "dashboards for the sake of it," and the first two *arr services deployed — with an IP collision on the way that was worth learning from.*

---

## Where we started

The previous session ended with Proxmox running four LXC containers (Pi-hole, Uptime Kuma, Homepage, all with 2FA + backups + network-wide DNS) plus Prometheus and Grafana added at the end of Sunday night, with node_exporter running on all five containers and pve-exporter on the host. Two community dashboards were rendering live data (Node Exporter Full — ID 1860, Proxmox via Prometheus — ID 10347).

By the end of tonight: Grafana is integrated into Homepage as a live widget tile (after a substantial debugging detour that taught us something real about Grafana's permission model), Prowlarr and Lidarr are installed and reachable, an IP collision from careless allocation was diagnosed and fixed, and we had an honest conversation about whether the whole monitoring-first approach was upside-down.

---

## Part 1: The Grafana-on-Homepage widget — a lesson in Grafana's permission architecture

This was the longest debugging thread of the night, and the correction at the end is worth understanding properly because it reveals how Grafana actually thinks about permissions.

### The initial (wrong) approach: service account with API token

The plan followed the pattern established for Pi-hole, Uptime Kuma, and Proxmox: create a scoped credential, paste it into `services.yaml`, restart Homepage.

For Grafana, "scoped credential" meant a **service account**. Grafana service accounts are the modern replacement for the deprecated API keys — non-human identities you create in the UI, assign a role to (Viewer / Editor / Admin), and generate tokens against. The tokens carry the account's permissions.

We created a service account called `homepage`, assigned it the **Viewer** role (least-privilege default), generated a token, saved it to Bitwarden as `Grafana - LemonLab`, and pasted the token into `services.yaml` with `username: api_key` (Homepage's convention for token-based auth) and the token in the `password:` field.

Result: `403 Forbidden — Permissions needed: server.stats:read`.

### The (also wrong) fix: bump the account to Admin

The obvious response: the widget needs stats access, Viewer doesn't have it, promote to Admin.

Done in the UI. Restart Homepage. Same 403.

Direct curl test with the token also returned 403.

### The real (also wrong) hypothesis: tokens cache original permissions

Legitimate Grafana quirk: tokens generated under a role sometimes retain that role's permissions even after the account is bumped. Fix should be delete the token and generate a fresh one now that the account is Admin.

Done. Same 403.

### The actual root cause

This is the important bit. Homepage's Grafana widget calls **`/api/admin/stats`**, which is Grafana's **Admin HTTP API**. From Grafana's own documentation:

> You can't authenticate to the Admin HTTP API with service account tokens. Service accounts are limited to an organization and an organization role, and can't be granted Grafana server administrator permissions.

The permission the widget needs, `server.stats:read`, is a **server-level permission** — belonging to a Grafana Server Administrator role. Service accounts max out at **organization Admin**, which is a different, lower privilege level. The naming is confusing on purpose because it's a permissions model, but the two "Admin" levels aren't the same:

- **Organization Admin** — full permissions inside one organization: create/edit dashboards, manage data sources, invite users to that org
- **Server Admin** — instance-wide permissions: create organizations, manage all users globally, view server statistics

Service accounts, by design, live inside a single organization. They can't be granted server-wide permissions. No amount of role-bumping fixes this — it's an architectural limit.

### The actual fix: basic auth with a real user

Homepage's widget requires basic auth (username + password) with a real user account that has **Grafana Server Admin** — which normally means the built-in `admin` user you log in with.

Changed `services.yaml` to:

```yaml
- Grafana:
    icon: grafana.png
    href: http://192.168.x.105:3000
    description: Metrics dashboards
    widget:
      type: grafana
      url: http://192.168.x.105:3000
      username: admin
      password: <admin password>
```

Restart Homepage → widget rendered correctly.

### What to remember from this

Three lessons worth internalising:

1. **Service accounts vs users, "admin" vs "admin"** — many systems have this multi-tiered permission model. When something says "admin doesn't work," check whether you're talking about the right *kind* of admin. Kubernetes has the same trap (namespace admin ≠ cluster admin). AWS has the same trap (account root ≠ organization admin ≠ IAM user with AdministratorAccess policy).

2. **When you hit a permission wall, check the API endpoint the client uses.** The endpoint documentation tells you the exact permission required. In this case, `/api/admin/stats` is prominently marked in Grafana's docs as "requires Basic Auth, cannot be used with service account tokens." Reading the docs would have skipped 45 minutes of debugging.

3. **Homepage forces you into a security tradeoff you should acknowledge.** The Grafana widget requires storing your server admin password in plaintext in `services.yaml`. On a homelab this is acceptable, but the mitigation is to create a **dedicated Grafana user** (not the built-in admin) with Server Admin role and a unique password, so if the config file leaks you rotate one password instead of your primary admin credentials. Same principle as the Proxmox API token we set up earlier — the token is separable from your main Proxmox login.

---

## Part 2: Wi-Fi diagnostic — reading ping output as a story

Mid-afternoon, everything stopped working. Ubuntu was "connected to the internet" per the OS but no websites loaded. The diagnostic process is worth documenting because it's a pattern you'll use repeatedly.

### The initial hypothesis: DNS

Websites failing while ICMP works is almost always DNS. The test:

```
ping 1.1.1.1    # test raw connectivity
ping google.com # test DNS resolution
```

If the first works and the second doesn't → DNS problem. If both fail → connectivity problem.

Both failed. Not DNS.

### The actual pattern: catastrophic latency, not failure

The ping output told a story. Ping to the Pi-hole (192.168.x.101) on the local network showed:

- 33% packet loss
- Ping times of **8,000-9,000 milliseconds**

Ping times on a healthy LAN should be under 1ms. **8-9 seconds is a smoking gun for wireless congestion or a failing link.** The pattern of long silence followed by bursts of packets arriving together is classic buffered retransmission behaviour on a struggling wireless link.

### The three-layer diagnostic

You then walked the layers systematically:

1. **Confirm interface state**: `ip -brief link` showed `wlp40s0 UP`, `enp34s0 DOWN` — confirmed Wi-Fi (ethernet unplugged as expected)
2. **Ping the router**: `ping -c 10 192.168.x.1` returned mostly clean pings (4-19ms) with one 222ms spike — the router itself is fine, but the wireless path to it is intermittent
3. **Try external DNS**: `ping google.com` eventually worked with 125-305ms times — noticeably high for wired but within Wi-Fi normal range

### The diagnosis

Transient Wi-Fi issue, most likely one of:

- **Channel interference or congestion** — Wi-Fi is a shared medium; neighbours' networks, microwaves, Bluetooth all contend for the same spectrum
- **Roaming/signal drop** — the client and AP briefly lose good communication and retransmit heavily
- **Client-side driver hiccup** — Intel wireless stack occasionally enters a bad state and recovers

Nothing to fix — it recovered on its own. But the exercise reinforces:

**When something breaks, ping tells a story.** Whether packets arrive at all (loss %), how quickly they arrive (latency), and whether the pattern is consistent or bursty — these three signals together identify the layer of the problem. Layer 1 (physical/cable) usually looks like 100% loss. Layer 2 (Wi-Fi/switch) looks like intermittent loss with variable latency. Layer 3 (routing) usually shows loss to specific destinations only. DNS problems show clean pings to IPs but failed hostname resolution.

### The bigger picture

This is exactly why the Ethernet run is on your priority list. Wi-Fi is fine for phones and laptops; it's a poor foundation for infrastructure or reliable monitoring. Your Pi-hole and Proxmox host are already on ethernet (that's why the ThinkCentre is at .100 with a stable connection). Your Ubuntu desktop being on Wi-Fi is the weak link tonight; the sparky job resolves it permanently.

---

## Part 3: Pi-hole blocklists — the "more equals better" misconception

Between debugging sessions we had a substantive conversation about Pi-hole blocklists worth capturing.

### What blocklists actually are

A blocklist is a plain text file, one domain per line, that Pi-hole fetches on a schedule and merges into its gravity database. Common lists:

- **StevenBlack hosts** — the traditional community-maintained list, low false-positive, low aggression
- **HaGeZi Multi Normal** — well-maintained, more aggressive on trackers/telemetry
- **HaGeZi Pro / Ultimate** — increasingly strict
- **OISD Big** — actively removes reported false positives, tries to avoid breakage

### The dedupe misconception

The natural assumption: adding more lists = more protection. If Pi-hole dedupes (which it does — the gravity DB stores each unique domain once), stacking lists is "free" in disk/memory terms.

**What actually goes wrong when you stack lists:**

1. **False positives compound, they don't average.** If List A correctly blocks `cdn.example.com` (a tracker) but List B over-blocks `example.com` (the whole site), the union blocks both. You can't average blocklist correctness by combining them — the aggressive rule always wins.

2. **Debugging becomes hard.** When a shopping checkout breaks or an app stops working, you have to figure out *which list* added the offending domain. One list = obvious. Five lists = investigation.

3. **Maintenance philosophies drift.** Different maintainers have different update schedules and criteria. HaGeZi is aggressive about adding new trackers as they're discovered; OISD is aggressive about removing user-reported breakage. Stacking them means you inherit both behaviours simultaneously and can't reason about what your blocker is doing.

### The correct model

**Pick one primary list matched to your tolerance for false positives; add specialist lists only for specific problems.** If you own a smart TV that phones home to a specific set of telemetry servers, add a smart TV list. If you have a gambling addiction problem, add a categorical gambling list. Don't add lists speculatively.

You're on StevenBlack alone, which is a sensible default — low maintenance, well-understood, unlikely to break things. No reason to change unless you hit a specific problem.

---

## Part 4: The observability-vs-services realisation

Halfway through the evening you noticed something sharp: the LemonLab currently runs **four monitoring/dashboard tools** (Uptime Kuma, Homepage, Prometheus, Grafana) watching **one real service** (Pi-hole). That's upside-down.

This is worth capturing as a genuine insight because most homelabbers don't notice it. The observability stack was built before the thing to be observed. Two failure modes are lurking:

### Failure mode 1: Service sprawl to justify dashboards

The trap: deploying services (Nextcloud, Immich, Vaultwarden) just so the dashboards have something to display. You end up maintaining 15 services you don't use to feed 4 tools that watch them. Worse position, not better.

### Failure mode 2: Correct response — add services you'd actually use

The legitimate move: pick services that solve a real problem for you right now, and the dashboards become useful as a side effect rather than an end in themselves.

### The framing that matters

**The homelab isn't the destination, it's the CV artifact for the career pivot.** Under that lens:

- Prometheus + Grafana on 5 containers is fine even though it's overkill — you can talk about the setup, the exporters, the dashboards in an interview regardless of scale
- Uptime Kuma with 2 monitors is a problem — an interviewer sees "installed a tool, didn't really use it"
- More services helps here, but only if you can articulate *why* you deployed each one

**The test to apply before deploying any new service:** finish this sentence — "I'm deploying this because I currently have problem X, and it solves X by doing Y." If you can't, don't deploy it.

### What Homepage actually is (clarification)

Related — you realised Homepage's actual purpose wasn't clear before. It's not a dashboard-for-the-sake-of-it. It's a **start page for your homelab.** Every tile has a link (click → opens that service's UI) and optionally a widget (live data at a glance). The workflow is: open browser → Homepage is your home page → glance at status → click whichever service you need.

Its value scales with the number of services. 1 service = pointless. 10 services = genuinely useful. 20+ = essential. Currently sparse because you deployed it ahead of the thing it serves. Not wasted, just early.

---

## Part 5: The *arr ecosystem — what it is and what it isn't

Before deploying Prowlarr and Lidarr, we walked through what the *arr apps actually do.

### The core idea

The *arr apps automate "I want media, go get it." You tell them what you want; they find it, download it, name it correctly, file it in the right folder, and hand it to your media player.

### The lineup

| App | Purpose |
|---|---|
| **Sonarr** | TV shows — tracks series, grabs new episodes as they release |
| **Radarr** | Movies |
| **Lidarr** | Music — albums, discographies, new releases |
| **Readarr** | Books/audiobooks (development has been rocky) |
| **Bazarr** | Subtitles — companion to Sonarr/Radarr |
| **Prowlarr** | Indexer manager — one place to configure search sources, feeds the others |
| **Whisparr** | Adult content, same pattern |

### The flow

1. **Prowlarr** holds indexers (Usenet or torrent trackers)
2. **Sonarr/Radarr/Lidarr** ask Prowlarr "find me this"
3. Prowlarr searches all configured indexers, returns results
4. The *arr picks the best result based on quality rules you set
5. Sends it to a **download client** (qBittorrent, SABnzbd)
6. On completion, the *arr renames the file, files it in the right folder
7. **Media server** (Jellyfin, Plex, Emby) reads the folder and serves it

### Lidarr specifically

Music is genuinely harder than TV/movies for these tools. Metadata is messier, tagging conventions vary, MusicBrainz (Lidarr's metadata source) doesn't always match what indexers have. Availability of niche genres (djent, prog metal) at FLAC quality is thinner than mainstream chart music.

Your existing music workflow — Whipper ripping CDs to `/mnt/music`, Picard tagging, Strawberry playing — is *curated* from physical media you own. Lidarr is *automated acquisition* — a different use case. Combining them means either two parallel libraries or careful merging.

The strategy we agreed: separate folders. `/mnt/music/library` for the curated CD/Bandcamp collection, `/mnt/music/incoming` for Lidarr's grabs. Audition without polluting the library, promote favourites to the library after buying the CD or Bandcamp download.

---

## Part 6: LXC creation via community-scripts — the prompts explained

We installed Prowlarr and Lidarr via the community-scripts project (the project formerly known as tteck's Proxmox Helper Scripts, renamed and maintained by the community after tteck passed away). Each install prompts through ~20 questions in Advanced mode. Understanding what each option controls is educational.

### Install mode

- **Default** — script picks reasonable settings, minimal prompts. Fine when you don't need control.
- **Advanced** — full control over resources, network, features. What you want while learning.

### Resources

- **CPU cores** — how many cores the container can use. Prowlarr/Lidarr are light, 2 is fine.
- **RAM** — 1024 MB fits both comfortably. Media servers (Jellyfin) need more.
- **Disk** — 4-8GB for the app itself. Media storage is separate (bind mount or NFS).

### Network

- **Bridge** — `vmbr0` is the default Proxmox software bridge to your physical network
- **IPv4** — static IP in your subnet. Critical to check against existing allocations (learned tonight the hard way).
- **Gateway** — your router (`192.168.x.1`)
- **IPv6** — `none` for an IPv4-only home network. Selecting SLAAC/DHCP adds unnecessary complexity.
- **MTU** — blank = inherit from bridge (1500 standard). Only change for jumbo frames on dedicated storage networks or VPN tunnels.
- **DNS domain** — the search suffix appended to unqualified hostnames. Blank unless you have an internal DNS domain.
- **DNS server** — set to Pi-hole (`192.168.x.101`) so container queries get filtered and appear in the Pi-hole log.
- **MAC address** — blank = Proxmox generates unique. Only override for specific reasons (DHCP reservations tied to MAC, licensing).
- **VLAN tag** — blank = untagged (default VLAN). Set when you have VLANs configured (future post-sparky).

### Security features

- **Unprivileged container** — yes. Root inside container is not root on host. Massive security win. Only disable for specific compatibility reasons.
- **Enable root access** — yes for homelab. Lets you SSH in as root with the password you set. Cleaner than `pct enter` every time.
- **SSH key source** — choose `none` unless you've deliberately set up SSH key infrastructure across your lab. Password auth is fine for now.
- **Container Protection** — blocks accidental `pct destroy`. Off while learning (you want freedom to rebuild). Turn on selectively for irreplaceable services (e.g. future Vaultwarden).

### Container capabilities

- **FUSE** — filesystem-in-userspace mounting. Needed for rclone/cloud storage. Off unless required.
- **TUN/TAP** — virtual network interfaces for VPN clients running *inside* the container. Off unless you're running WireGuard or Tailscale inside this specific LXC.
- **Nesting** — allows containerised processes and systemd to work properly. On for most modern app installs. Doesn't grant container-escape on unprivileged LXCs.
- **GPU passthrough** — off for services with no GPU use. On for Jellyfin/Plex for hardware transcoding (future consideration).
- **mknod** — permission to create device nodes in `/dev`. Off unless the app manages block devices.
- **Filesystem mounts** (comma-separated list) — allow the container to mount specific FS types itself. Empty unless the container needs to mount NFS/CIFS on its own (usually you bind-mount from the host instead).

### Install-time behaviour

- **APT-Cacher-NG proxy** — caches Debian packages locally so multiple containers don't re-download. Only useful if you deploy that service. No = correct until then.
- **HTTP/HTTPS proxy** — for corporate networks with mandatory web proxies. No for home.
- **Anonymous telemetry** — install-count reporting to community-scripts. Deny it to stay consistent with your privacy stance.
- **Post-install hook script** — path to a `.sh` file the installer runs after setup. Advanced automation for repeatable custom deployments. Blank until you need it.
- **Verbose mode** — shows every command and output during install. On while learning, off once you've seen enough.

---

## Part 7: The IP collision — how it happened and what to remember

The mistake worth documenting because it'll happen again if you're not deliberate.

### What went wrong

I assigned Prowlarr `192.168.x.104` and Lidarr `192.168.x.105`, working from memory of your infrastructure that was wrong. Reality:

```
LXC 100 pihole       → 192.168.x.101
LXC 101 uptime-kuma  → 192.168.x.102
LXC 102 homepage     → 192.168.x.103
LXC 103 prometheus   → 192.168.x.104
LXC 104 grafana      → 192.168.x.105
LXC 105 prowlarr     → 192.168.x.104  ← collision with prometheus
LXC 106 lidarr       → 192.168.x.105  ← collision with grafana
```

Two IP collisions. Prowlarr appeared to work; Lidarr didn't respond on port 8686 despite being confirmed running.

### Why the symptoms were confusing

When two devices claim the same IP, whichever answers ARP first wins for that client's cache. Different clients can get different answers. Your Ubuntu happened to reach Prowlarr through .104 (ARP luck), but reached Grafana on .105 instead of Lidarr — hence "Prowlarr works but Lidarr doesn't." Both services were actually broken; only one broke visibly.

### The diagnostic that identified it

`ip neigh show | grep 192.168.x.105` on Ubuntu would have shown the MAC address cached for .105 and let us compare against the Lidarr container's actual MAC. Different MAC = wrong device, IP collision confirmed.

### The fix

Reassigned Prowlarr to .106 and Lidarr to .107 (verified free), then `pct reboot 105` and `pct reboot 106`. **Note:** the command is `pct reboot`, not `pct restart` — `pct` has different subcommands than `systemctl`.

### The broader issue

The LXC ID vs IP last-octet mismatch is a persistent trap. Currently:

```
LXC 100 pihole       → .101
LXC 101 uptime-kuma  → .102
LXC 102 homepage     → .103
LXC 103 prometheus   → .104
LXC 104 grafana      → .105
LXC 105 prowlarr     → .106
LXC 106 lidarr       → .107
```

Off-by-one throughout. A future "IP tidy-up v2" session should either accept the drift and document it clearly, or renumber IPs to match LXC IDs (LXC 105 = .105, LXC 106 = .106). The first is easier, the second is cleaner.

### What to remember

- **Always verify your allocation before assigning a new IP.** Two minutes with `pct list` and clicking through Network tabs prevents an hour of debugging.
- **IP collisions are silent for a while.** ARP cache means you might not notice for hours or days until a cache expires or a device reboots.
- **When something "works from one client but not another," suspect the network layer** (ARP, DNS, routing) before suspecting the service.

---

## Part 8: Verifying a running service — the diagnostic chain

When Lidarr wasn't responding externally, we walked through a diagnostic chain worth remembering as a general pattern.

### The chain

1. **Is the container running?** — Proxmox UI shows green/grey. `pct status 106` from host.
2. **Can the network reach it?** — `ping <IP>` from another host. Confirms Layer 3 connectivity.
3. **Is the service listening?** — from inside the container: `ss -tlnp | grep <port>`. Shows what's bound to what interfaces.
4. **Is the service responding?** — from inside the container: `curl -I http://localhost:<port>`. Bypasses network entirely, tests just the service.
5. **From external client?** — `curl -I http://<IP>:<port>` from Ubuntu. Adds the network path back in.

Each step isolates a layer. If step 3 shows `127.0.0.1:8686` instead of `*:8686`, the service is bound only to localhost and won't accept external connections. If step 4 succeeds but step 5 fails, something between the container and client is blocking (firewall, routing, IP collision).

### Reading `ss -tlnp` output

```
LISTEN 0 512 *:8686 *:* users:(("Lidarr",pid=4258,fd=349))
```

- `LISTEN 0 512` — listening state, 0 pending connections in queue, max queue depth 512
- `*:8686` — listening on all interfaces (0.0.0.0), port 8686
- `*:*` — accepting connections from any source
- `users:(("Lidarr",pid=4258,fd=349))` — the Lidarr process (PID 4258) has file descriptor 349 bound to this socket

If the third field shows `127.0.0.1:8686` instead of `*:8686`, the service only accepts localhost connections — an app config issue, not a network issue.

### Reading HTTP response codes during diagnostic

- **200 OK** — service is up and returning content
- **401 Unauthorized** — service is up, but you need to authenticate. This is what a healthy Prowlarr/Lidarr returns to unauthenticated curl. Confirms the service is running correctly.
- **403 Forbidden** — service is up, credentials are valid, but you're not permitted to do this action. This was the Grafana problem.
- **404 Not Found** — service is up but the URL path is wrong.
- **500 Internal Server Error** — service is up but something's broken inside it.
- **Connection refused** — nothing is listening on that port. Different from "no route to host" (network path broken) or "connection timeout" (something silently dropping packets).

Learning to read these codes fluently saves enormous amounts of debugging time. "It doesn't work" is uninformative; "returns 401 to unauthenticated curl on port 8686" tells you exactly where you are.

---

## What's actually running now

Physical: Unchanged from previous session — ThinkCentre M710e with Kingston 64GB USB stick, TP-Link Archer AX53.

Software stack:

- Debian 13 / Proxmox VE 9.2.11
- Seven LXC containers:
  - CT 100 `pihole` — DNS filtering (.101)
  - CT 101 `uptime-kuma` — service monitoring (.102)
  - CT 102 `homepage` — dashboard/launcher, now with Grafana widget (.103)
  - CT 103 `prometheus` — metrics collection (.104)
  - CT 104 `grafana` — metrics visualisation (.105)
  - CT 105 `prowlarr` — indexer manager (.106)
  - CT 106 `lidarr` — music acquisition (.107)

Monitoring:

- node_exporter on all 5 previous LXCs + Proxmox host
- pve-exporter on Proxmox host
- Prometheus scraping all exporters
- Grafana rendering dashboards 1860 (Node Exporter Full) and 10347 (Proxmox via Prometheus)
- Grafana now surfaced on Homepage via basic-auth widget

Storage decision deferred: Lidarr installed but not yet configured for actual downloads — needs shared music storage. Options are NFS from Ubuntu desktop (adds hard dependency on desktop being on 24/7) or waiting for the NAS build. Not resolving tonight.

---

## Things worth reading further

- **Grafana Admin HTTP API docs** — [grafana.com/docs/grafana/latest/developer-resources/api-reference/http-api/api-legacy/admin/](https://grafana.com/docs/grafana/latest/developer-resources/api-reference/http-api/api-legacy/admin/). The exact page that would have saved 45 minutes of debugging. Note the explicit "cannot use service account tokens" statement at the top.
- **Grafana service accounts** — [grafana.com/docs/grafana/latest/administration/service-accounts/](https://grafana.com/docs/grafana/latest/administration/service-accounts/). Worth reading to understand the organizational-role boundary.
- **Prowlarr documentation** — [wiki.servarr.com/prowlarr](https://wiki.servarr.com/prowlarr). The whole Servarr wiki is well-written; when you configure indexers next session, this is where to look.
- **Lidarr wiki** — [wiki.servarr.com/lidarr](https://wiki.servarr.com/lidarr). Same series; the FAQ section covers most first-time-user confusion.
- **Community-scripts homepage** — [community-scripts.github.io/ProxmoxVE/](https://community-scripts.github.io/ProxmoxVE/). Browse the catalog to see what else exists before you deploy things ad-hoc.
- **The Servarr guide** — [trash-guides.info/Servarr/](https://trash-guides.info/Servarr/). Community wisdom on quality profiles, custom formats, and indexer configuration. Deep and opinionated but worth knowing exists.

---

## Concepts that came up in passing worth knowing about

- **ARP (Address Resolution Protocol)** — how IP addresses get translated to MAC addresses on a local network. When two devices claim the same IP, ARP cache is what makes the symptom intermittent — whichever device answered the ARP request first "wins" for that client until the cache entry expires (usually minutes to hours).
- **`ss` (socket statistics)** — modern replacement for `netstat`. Faster, better output, same purpose. `ss -tlnp` is the go-to incantation: TCP, listening, numeric, with process names.
- **HTTP status code families** — 1xx informational (rare), 2xx success, 3xx redirect, 4xx client error (auth/not-found/bad request), 5xx server error (crashes/upstream failures). The specific code often tells you exactly what's wrong before you look at logs.
- **Basic Auth vs Token Auth vs OAuth vs API Keys** — different authentication schemes for HTTP APIs. Basic Auth: username+password base64-encoded in the request header (weak but universal). Token Auth: opaque or JWT tokens in the Authorization header. OAuth: multi-step delegation flow (used by "Sign in with Google"). API Keys: static long strings, usually in a custom header or query string. Different services support different combinations; know which each service uses before you try to integrate.
- **Kestrel** — the .NET web server that Lidarr, Sonarr, Radarr, and Prowlarr all use. When curl shows `Server: Kestrel` in the response header, you're looking at a .NET-based service. Sometimes matters when troubleshooting; usually just useful trivia.
- **The Servarr project family** — Sonarr/Radarr/Lidarr/Readarr/Prowlarr are all forks/derivatives of the same codebase (originally Sonarr, itself forked from NZBDrone). This is why they share the same UI patterns, use the same web server (Kestrel), and configure similarly. Learning one teaches you 80% of the others.
- **UID/GID mapping in unprivileged LXCs** — root inside an unprivileged container is mapped to UID 100000 on the host. This is why bind-mounted directories need permissions set correctly on the host side, and why NFS shares need `no_root_squash` or matching UIDs to allow writes from the container.
- **pct commands** — Proxmox's `pct` tool for container lifecycle: `pct list`, `pct enter <id>`, `pct start/stop/reboot <id>`, `pct config <id>`, `pct status <id>`. Note the command is `reboot`, not `restart` — different from `systemctl`.

---

*Written from the LemonLab session of 31 August 2026. Bookmark alongside the first- and second-night notes for reference.*
