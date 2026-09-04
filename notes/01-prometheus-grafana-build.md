# LemonLab: Prometheus + Grafana Build — Sunday Night Session Notes

*The observability stack: Prometheus in one container, Grafana in another, node_exporter on every host in the lab, pve-exporter on the Proxmox host exposing hypervisor-level metrics, and two community-authored dashboards rendering live data end-to-end. Written after the fact to cover a session that got skipped in the daily notes cycle.*

---

## Where we started

The previous session left the lab with three services running (Pi-hole, Uptime Kuma, Homepage) on three LXC containers, backed up nightly to a USB stick, all 2FA-protected, with Homepage pulling live API data from the other two. No visibility into resource usage over time. No historical trend data. No answer to "was the ThinkCentre struggling last Thursday afternoon."

By end of the Sunday night session: Prometheus scraping metrics from every host and container in the lab, Grafana rendering two mature community dashboards on that data, and enough monitoring infrastructure to answer questions like "what was CPU load during the Homepage rebuild" or "how much memory does Pi-hole actually use over 24 hours."

---

## Part 1: What Prometheus and Grafana actually are

Understanding the two-tool split matters because it's the standard pattern for observability across the industry.

### Prometheus is a time-series database with a scraper

Prometheus's job is to periodically hit an HTTP endpoint on each monitored machine, fetch a text-format response listing every metric that machine exposes at that instant, and store those numeric values with a timestamp in its own database. It does this every 15 seconds by default. That's it — it collects, stores, and lets you query.

The Prometheus web UI at port 9090 is a debugging tool, not a dashboard: you can run PromQL queries and see targets' health, but nobody builds pretty graphs there.

### Grafana is a query-and-render frontend

Grafana connects to Prometheus (or many other data sources), lets you write PromQL queries, and turns the results into graphs, gauges, tables, heatmaps. That's what the "dashboards" everyone shares screenshots of are: Grafana's rendering of PromQL queries against Prometheus's stored data.

### Why they're separate

You could imagine one tool that both scrapes and visualises, and some do (Netdata, for example). The separation is deliberate:

- **Prometheus's storage engine is optimised for one thing** — writing millions of numeric samples per second and querying ranges of them fast. Making it also render pixels would compromise that.
- **Grafana speaks to many backends** — Prometheus, InfluxDB, Elasticsearch, PostgreSQL, MySQL, Loki, CloudWatch. Same UI, different data sources. Learning Grafana once buys you visualisation across a dozen backends.
- **You can swap either side.** Switch from Prometheus to VictoriaMetrics for scale? Grafana still works. Switch from Grafana to Perses for a lighter UI? Prometheus doesn't care.

This is a common architectural pattern: split "storage/collection" from "presentation" so each can evolve independently.

### The exporter pattern

Prometheus doesn't know how to read CPU stats from a Linux box, or Proxmox stats from the Proxmox API, or database stats from Postgres. That's not its job. Instead, small programs called **exporters** run on the monitored host, expose an HTTP endpoint (usually `/metrics`), and translate whatever data source they cover into Prometheus's simple text format:

```
node_cpu_seconds_total{cpu="0",mode="idle"} 12345.67
node_memory_MemAvailable_bytes 8.5e+09
node_filesystem_free_bytes{mountpoint="/"} 1.2e+10
```

Prometheus scrapes that endpoint, stores the values, done. **The ecosystem has hundreds of exporters** — node_exporter (Linux OS metrics), pve-exporter (Proxmox), postgres_exporter, redis_exporter, blackbox_exporter (HTTP/TCP probing), snmp_exporter (network devices), and countless application-specific ones. Learning to install and configure an exporter is a repeatable pattern that transfers everywhere.

---

## Part 2: Prometheus container build

Standard LXC creation via the Proxmox UI:

- **CT 103** `prometheus` (LXC ID drift means this got .104 in the actual network)
- Debian 13 template, unprivileged, 1 core, 1024 MiB RAM, 8 GiB disk

Note the disk allocation: **larger than the other services** because Prometheus stores time-series data locally. Even at 15-second scrape intervals with modest metric counts, the database grows measurably over months. 8 GiB is enough for a homelab of this size for well over a year.

### Install

Inside the container:

```
apt update && apt upgrade -y
apt install -y prometheus prometheus-node-exporter
```

Debian ships Prometheus in its official repositories, so no third-party repos needed. The `prometheus-node-exporter` package installs node_exporter on this same container — useful because Prometheus can immediately test scraping against localhost before we deploy exporters elsewhere.

Both packages install as systemd services on install. `systemctl status prometheus` and `systemctl status prometheus-node-exporter` should both show active/running. Prometheus listens on port 9090, node_exporter on 9100.

### First smoke test

Open `http://192.168.x.104:9090` (or whichever IP the container landed on). Land on Prometheus's UI. **Status → Targets** shows scrape targets and their health. Out of the box there's one target — Prometheus's own metrics endpoint (`localhost:9090`). Should show `UP`.

That's the minimum viable Prometheus: scraping only itself. No graphs, no dashboards, just proof the plumbing works.

---

## Part 3: Grafana container build

Same LXC creation pattern:

- **CT 104** `grafana` — Debian 13, unprivileged, 1 core, 1024 MiB RAM, 8 GiB disk

### Install

Grafana isn't in Debian's default repos (or the versions there are lagging), so add Grafana's official APT repository first:

```
apt install -y apt-transport-https software-properties-common wget
mkdir -p /etc/apt/keyrings/
wget -q -O - https://apt.grafana.com/gpg.key | gpg --dearmor | tee /etc/apt/keyrings/grafana.gpg > /dev/null
echo "deb [signed-by=/etc/apt/keyrings/grafana.gpg] https://apt.grafana.com stable main" | tee /etc/apt/sources.list.d/grafana.list
apt update
apt install -y grafana
systemctl enable --now grafana-server
```

### What each step does

- **apt-transport-https / software-properties-common** — enables APT to fetch from HTTPS repos and manage third-party sources
- **`wget ... | gpg --dearmor`** — downloads Grafana's public signing key and converts it from ASCII-armored (`.asc`) to binary (`.gpg`) format because that's what modern apt wants under `/etc/apt/keyrings/`
- **`signed-by=` in the sources.list line** — pins this repo to only accept packages signed by that specific key. Better than the old `apt-key add` approach which trusted keys globally.
- **`apt update`** — refreshes the package index so apt learns about Grafana packages
- **`apt install grafana`** — installs the OSS version
- **`systemctl enable --now grafana-server`** — enables on boot AND starts immediately, both in one command

Grafana listens on port 3000. Open `http://<grafana-ip>:3000`, log in with default credentials `admin/admin`, and immediately change the password (Grafana forces this on first login).

### Add Prometheus as a data source

In Grafana UI: **Connections → Data sources → Add new data source → Prometheus.** URL: `http://<prometheus-ip>:9090`. Save & test.

That's Grafana wired to Prometheus. Grafana now knows where to send PromQL queries. Ready to render dashboards.

---

## Part 4: The community dashboard shortcut

Building good dashboards from scratch is genuinely hard — knowing which metrics matter, what visualisations suit which data, what units to use, how to lay out the panels. The Grafana community solves this via **grafana.com/dashboards** — a public gallery of dashboards you import by ID.

### Dashboard 1860 — Node Exporter Full

The de facto standard dashboard for node_exporter data. Shows CPU, memory, disk, network, and load metrics with sensible defaults and a **Nodename dropdown** at the top to switch between hosts.

**Grafana → Dashboards → New → Import → paste `1860` → Load → select Prometheus data source → Import.**

Immediately renders live graphs of the Prometheus container itself (the only node_exporter target at this point).

### The value of pre-built dashboards

You're not cheating by importing rather than building. The community dashboard authors have iterated for years on which metrics to show and how to visualise them. Learning to consume dashboards well is a legitimate skill; learning to build them from scratch is a separate (and later) skill. Import first, understand the queries by clicking into panels, modify or replace when you find gaps. That's the correct progression.

---

## Part 5: Rolling out node_exporter to every container

The pattern from now on: install `prometheus-node-exporter` on each container, then add it to Prometheus's scrape config as a target.

### Install on each container

For Pi-hole (100), Uptime Kuma (101), Homepage (102), and the Proxmox host itself:

```
apt install -y prometheus-node-exporter
systemctl status prometheus-node-exporter
```

Same package on all four. Each starts listening on port 9100 automatically. Verify with:

```
curl -s http://localhost:9100/metrics | head
```

Should return a wall of metric lines starting with `# HELP` and `# TYPE`.

### Update Prometheus's scrape config

Back on the Prometheus container, edit `/etc/prometheus/prometheus.yml`. Find the existing `node` job and extend the targets list:

```yaml
  - job_name: node
    static_configs:
      - targets:
          - 'localhost:9100'
          - '192.168.x.100:9100'
          - '192.168.x.101:9100'
          - '192.168.x.102:9100'
          - '192.168.x.103:9100'
```

Reload Prometheus without restarting (preserves state):

```
systemctl reload prometheus
```

**Status → Targets** in Prometheus UI now shows five node targets, all should be UP within 30 seconds.

In Grafana, the Node Exporter Full dashboard's **Nodename** dropdown now lists all five hostnames — pihole, uptime-kuma, homepage, proxmox, prometheus. Switch between them to see per-host metrics.

That's the value of the exporter pattern in action: five machines, one dashboard template, zero per-host UI configuration.

### YAML flexibility

Same content works in two forms:

```yaml
targets:
  - 'localhost:9100'
  - '192.168.x.100:9100'
```

Or:

```yaml
targets: ['localhost:9100', '192.168.x.100:9100']
```

Both are valid YAML lists — one uses block style, one uses flow style. Prometheus doesn't care. Block style scales better as the list grows; flow style is more compact for short lists.

---

## Part 6: pve-exporter — the Proxmox hypervisor view

node_exporter tells you what the OS thinks — "CPU is at 40%, 8 GiB memory available." That's the same view a Linux admin would have SSHing in.

**pve-exporter tells you what Proxmox thinks** — "container 100 is allocated 512 MiB and using 320 MiB; storage pool local-lvm has 45 GiB free; the cluster is healthy; VM 200 is stopped." That's the hypervisor's view of its guests, resources, and cluster state. Different data, complementary picture.

### Install (on the Proxmox host, not a container)

pve-exporter talks to the Proxmox API, so it lives on the host itself. From the Proxmox shell:

```
apt install -y python3-pip python3-venv
python3 -m venv /opt/pve-exporter
/opt/pve-exporter/bin/pip install prometheus-pve-exporter
```

**Why a virtualenv instead of a global `pip install`:** Debian 13 (Trixie) marks its Python environment as "externally managed" (PEP 668). Doing `pip install` globally fails with an explanatory error. Virtualenvs sidestep this cleanly — the exporter lives in its own isolated Python environment under `/opt/pve-exporter/`, doesn't touch the system's Python, doesn't clash with apt-managed packages, easy to remove (just `rm -rf` the directory).

This is the modern correct pattern for Python software on Debian. Old habits from Ubuntu 18/20 era of `sudo pip install` are actively broken now, and for good reason — global pip installs were a persistent source of package conflicts and system breakage.

### Create the API user

pve-exporter needs to authenticate to Proxmox to fetch data. Best practice: dedicated user, minimum permissions, dedicated password.

In Proxmox web UI:

- **Datacenter → Permissions → Users → Add**
  - User name: `pve-exporter`
  - Realm: `Proxmox VE authentication server` (the `pve` realm, not `pam`)
  - Password: strong, saved to Bitwarden as "PVE Exporter"
  - Enabled ✓

- **Datacenter → Permissions → Add → User Permission**
  - Path: `/`
  - User: `pve-exporter@pve`
  - Role: `PVEAuditor` (read-only across the whole datacenter)

Two realms exist by default: `pam` (Linux system users on the host) and `pve` (Proxmox-only users). Creating a `pve` realm user means the credentials only exist inside Proxmox — no shell access, no host user account. Correct hygiene.

**PVEAuditor is a built-in read-only role.** pve-exporter never needs to change anything; it just reads. Applying least-privilege here is easy because Proxmox ships the role you need.

### Config file and systemd unit

```
mkdir -p /etc/prometheus
nano /etc/prometheus/pve.yml
```

Contents:

```yaml
default:
    user: pve-exporter@pve
    password: <the-password-you-just-set>
    verify_ssl: false
```

`verify_ssl: false` because Proxmox uses a self-signed cert by default and pve-exporter would otherwise reject the connection. On a homelab this is acceptable; in production you'd install a proper cert.

Then the systemd unit, written with a heredoc so the whole thing goes in cleanly:

```
cat > /etc/systemd/system/pve-exporter.service << 'EOF'
[Unit]
Description=Proxmox VE Exporter
After=network.target

[Service]
Type=simple
ExecStart=/opt/pve-exporter/bin/pve_exporter --config.file /etc/prometheus/pve.yml
Restart=always
User=root

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable --now pve-exporter
systemctl status pve-exporter
ss -tlnp | grep 9221
```

pve-exporter listens on port 9221. Should show LISTEN. If it doesn't, `journalctl -u pve-exporter -n 50` tells you why (usually a typo in the config file or wrong credentials).

### The clever bit: relabel_configs

Adding pve-exporter to Prometheus is not as simple as adding another `node:9100` target. pve-exporter's endpoint accepts a `target` URL parameter telling it which Proxmox node to query — because in a real cluster you'd have one pve-exporter serving many nodes. Prometheus needs to be taught to rewrite the request accordingly.

In `/etc/prometheus/prometheus.yml`, add a new job:

```yaml
  - job_name: pve
    static_configs:
      - targets:
          - 192.168.x.100  # proxmox host
    metrics_path: /pve
    params:
      module: [default]
      cluster: ['1']
      node: ['1']
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: 192.168.x.100:9221  # pve-exporter address
```

### What relabel_configs actually does

Reading top-to-bottom:

1. **First relabel** — take whatever's in the `__address__` label (the original target, `192.168.x.100`) and copy it into a new label `__param_target`. Prometheus turns `__param_*` labels into URL query parameters on the scrape request, so this sets `?target=192.168.x.100` in the URL pve-exporter receives.

2. **Second relabel** — copy `__param_target` into `instance`. This is the label that shows up in metrics saying "this metric came from THIS Proxmox node." Without this, all metrics would be labelled with pve-exporter's own address, which is meaningless — you want them labelled by the *subject* of the query.

3. **Third relabel** — rewrite `__address__` to point at pve-exporter (`192.168.x.100:9221`) rather than the original target. This is where Prometheus actually sends the scrape request. So the flow becomes: "Ask pve-exporter at :9221 for metrics about node 192.168.x.100, with the query parameters `module=default&target=192.168.x.100`."

This relabelling pattern is genuinely important to understand because it's how Prometheus handles all indirect/proxy-style exporters — blackbox_exporter, snmp_exporter, and others all use variants of the same technique. Learning it once here transfers directly to those.

### Reload and verify

```
systemctl reload prometheus
```

Prometheus UI → **Status → Targets** → new `pve` job appears, should be UP within 30 seconds.

### Dashboard 10347 — Proxmox via Prometheus

Same pattern as before: **Dashboards → New → Import → 10347 → Load → select Prometheus data source → Import.**

Renders VM/container inventory, per-guest CPU and memory usage, storage utilisation, network throughput, cluster status. The Proxmox equivalent of what the Grafana Cloud pricing pages show for professional monitoring — running for free on your ThinkCentre.

**Gotcha during import:** the dashboard import screen has two Load buttons — one next to the ID field at the top, one under the JSON textarea below. Use the top one when importing by ID. The bottom one is for pasting a full dashboard JSON export.

---

## Part 7: What worked, what to remember

### The overall pattern

For any future service you want to monitor, the workflow is now settled:

1. Find an appropriate exporter (search "prometheus <thing> exporter")
2. Install it on the host that runs the thing
3. Add its address as a scrape target in Prometheus
4. Reload Prometheus
5. Verify target UP in Prometheus UI
6. Find a community Grafana dashboard for that exporter, import by ID
7. If no dashboard exists, or the ones available are wrong, build your own — but only after imports fail you

This scales indefinitely. Every service you add to the lab that has an exporter (which is most of them) plugs into this pipeline in ~10 minutes.

### The three pieces to internalise

- **The Prometheus/Grafana split** — one stores, one renders. Standard architecture across the industry.
- **The exporter pattern** — Prometheus never knows about the thing it monitors directly; an exporter translates. Learn one exporter well and the pattern generalises.
- **The relabel_configs mechanism** — for indirect exporters that scrape *on behalf of* a target, relabelling teaches Prometheus how to construct the right request and how to label the results correctly.

### The right level of scepticism about what we've built

By the end of the session there were four monitoring/dashboard tools (Prometheus, Grafana, Uptime Kuma, Homepage) watching one real service (Pi-hole). That imbalance was noticed properly in the next session and is worth flagging here too: **this is an educational build, not an operational necessity.** For a homelab of this size, Prometheus + Grafana is genuinely overkill. Its value is:

1. **Learning the pattern** — the same stack runs at basically every tech company. Building it in a lab teaches transferable knowledge.
2. **CV/portfolio evidence** — being able to say "I built and configured a Prometheus + Grafana + node_exporter + pve-exporter monitoring stack on Proxmox with X container targets" is legitimate infrastructure experience.
3. **Being ready for scale** — as more services get added, the monitoring is already there. Backfilling monitoring after the fact is worse than building it early.

None of those are the same as "I need to know my Pi-hole's memory usage over time." Be honest about the purpose.

---

## What was actually running by end of session

- CT 103 `prometheus` — Prometheus + node_exporter
- CT 104 `grafana` — Grafana OSS
- node_exporter on Proxmox host and all four service containers
- pve-exporter on Proxmox host with dedicated `pve-exporter@pve` API user (PVEAuditor role)
- Prometheus scraping: 5 node_exporter targets + 1 pve-exporter target, all UP
- Grafana rendering: dashboard 1860 (Node Exporter Full) and dashboard 10347 (Proxmox via Prometheus)
- Both dashboards showing live data end-to-end

Not yet done at session end (some completed the following night):

- Homepage tile for Grafana (done Monday, in a substantial detour that taught us about Grafana's server-admin vs org-admin permission model — see the "third day" notes)
- Custom dashboards specific to LemonLab services
- Alerting rules in Prometheus (Alertmanager not deployed)
- Retention tuning (Prometheus defaults are usually fine for lab scale, but worth revisiting)

---

## Things worth reading further

- **Prometheus documentation** — [prometheus.io/docs/](https://prometheus.io/docs/). The "First Steps" and "Configuration" sections are the essentials. The PromQL section repays a slow read once you're comfortable with the basics.
- **The Prometheus book** — *Prometheus: Up & Running* by Brian Brazil (O'Reilly). Written by a former Prometheus core developer. Best single reference for going deep.
- **Grafana documentation** — [grafana.com/docs/](https://grafana.com/docs/). The panel-building and PromQL-in-Grafana sections are where you'll live once you start customising dashboards.
- **prometheus-pve-exporter** — [github.com/prometheus-pve/prometheus-pve-exporter](https://github.com/prometheus-pve/prometheus-pve-exporter). The exporter's README covers its config options, all query parameters, and the metric set it exposes.
- **Grafana dashboards gallery** — [grafana.com/grafana/dashboards/](https://grafana.com/grafana/dashboards/). Browse by data source. Find dashboards for every exporter you deploy.
- **PromLabs PromQL cheat sheet** — [promlabs.com/promql-cheat-sheet/](https://promlabs.com/promql-cheat-sheet/). Compact reference for the query language, worth bookmarking.

---

## Concepts that came up in passing worth knowing about

- **Time-series database (TSDB)** — a database optimised for storing timestamped numeric values, typically with heavy write throughput and range-query patterns. Prometheus, InfluxDB, VictoriaMetrics, TimescaleDB are all TSDBs. Different from relational (Postgres/MySQL) or document (MongoDB) databases in fundamental ways.
- **PromQL (Prometheus Query Language)** — Prometheus's own query language, purpose-built for time-series operations (rate over time, aggregation across labels, alerting predicates). Not SQL, and not translatable to SQL. Learning PromQL is a specific skill; expect it to feel foreign for the first few hours.
- **Pull vs push metrics** — Prometheus is a *pull* system: it reaches out to targets and scrapes them. Alternatives (Graphite, StatsD, InfluxDB in some modes) use *push*: the application sends metrics to the collector. Pull is easier to secure (collector inside your network, targets don't need to know where to send) and easier to debug (Prometheus UI shows exactly which targets are healthy). Push scales better in dynamic environments where targets come and go. Both patterns are widely used.
- **Scrape interval** — how often Prometheus asks each target for its metrics. Default is 15 seconds; can go down to 1 second for high-resolution needs, or up to minutes for coarse monitoring of many targets. Every metric you store costs disk space; longer intervals mean smaller storage but coarser data. For a homelab, defaults are fine.
- **The metrics endpoint format** — plaintext, one metric per line, optional labels in braces, numeric value at the end, optional timestamp. Deliberately simple so implementing an exporter in any language is trivial. Human-readable when you curl it.
- **systemd unit file basics** — the `[Unit]`, `[Service]`, `[Install]` sections you keep seeing follow a standard pattern. `After=network.target` = don't start until networking is up. `Type=simple` = the ExecStart process is the service itself (as opposed to `Type=forking` for daemons that fork into the background). `Restart=always` = if the process dies, restart it. `WantedBy=multi-user.target` = enable this at normal system boot. Reading `man systemd.service` explains all of these properly.
- **APT signing keys under /etc/apt/keyrings/** — the modern location and format. Old tutorials show `apt-key add`, which is deprecated because it trusted the key globally for any repo. The `signed-by=` approach in `sources.list` scopes each key to only its repository. Follow modern tutorials, not old ones.
- **PEP 668 externally-managed environments** — the Python packaging change that broke `sudo pip install` on modern Debian/Ubuntu. The fix is virtualenvs (`python3 -m venv`) for any Python software you install outside apt. Once you internalise this you stop fighting it.

---

*Written retrospectively from the LemonLab session of 30 August 2026, filling a gap between the second-night and third-night notes.*
