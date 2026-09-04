# LemonLab: Fourth Day Study Notes

*A cleaner session than yesterday's: Prowlarr and Lidarr wired into Homepage as live widget tiles with proper Infrastructure/Media grouping (no debug spiral this time), followed by a real security hardening pass across the whole lab — SSH key auth everywhere, password SSH globally disabled, and Ubuntu SSH aliases so nothing needs typing longer than `ssh lidarr`.*

---

## Where we started

Yesterday ended with Prowlarr and Lidarr installed on their (finally correct) IPs at .106 and .107, both with web UI auth configured. Neither service was integrated with Homepage, and the storage decision for Lidarr was deferred pending clarity on the NAS timeline.

By end of tonight: both services rendering live widgets on Homepage under a new "Media" group, and — the bigger win — SSH key authentication set up across every container in the lab with password SSH globally disabled. What was an ad-hoc mix of "some containers need passwords, some don't, one had SSH re-enabled yesterday as a quick fix" is now a consistent, key-only access model.

---

## Part 1: Prowlarr and Lidarr as Homepage widgets

Yesterday's Grafana widget was a permission-model saga. Tonight's *arr integrations were the opposite: standard API-key auth, worked first time, no debug cycle. Worth noticing which of those experiences is the norm and which is the exception.

### Generating the API keys

Both apps expose their API key in the same place: **Settings → General → Security → API Key**. The key is auto-generated at first startup; you don't create it, you copy it.

Saved both to Bitwarden as `Prowlarr - API` and `Lidarr - API`, following the naming convention established earlier (`<System> - <Purpose>`).

### The Homepage services.yaml update

Yesterday's file was already well-structured — one Infrastructure group holding Proxmox, Pi-hole, Uptime Kuma, and Grafana. The question was whether to append Prowlarr and Lidarr to that group or create a new one.

**Decision: create a new Media group.** Small visual overhead now for only two services, but:

- Future *arr additions (Sonarr, Radarr, Bazarr) or a media server (Jellyfin/Plex) all belong logically under Media, not Infrastructure
- Distinguishing "infrastructure services that make the lab work" from "media services the infrastructure serves" is meaningful
- Homepage renders groups as separate columns/blocks by default, so more groups = better visual scanning

The Homepage YAML structure for grouping is straightforward:

```yaml
- Infrastructure:
    - Proxmox: {...}
    - Pi-hole: {...}

- Media:
    - Prowlarr: {...}
    - Lidarr: {...}
```

Each top-level `-` is a group; each nested `-` inside is a service. Homepage handles the layout automatically.

### The widget blocks

Both *arr apps use the same widget pattern:

```yaml
- Prowlarr:
    icon: prowlarr.png
    href: http://192.168.x.106:9696
    description: Indexer manager
    widget:
      type: prowlarr
      url: http://192.168.x.106:9696
      key: <api key from Bitwarden>
```

Note `key:` rather than `password:` — this is Homepage convention for API-key-based widgets, distinct from basic auth widgets which use `username:`/`password:`. Different auth mechanism, different YAML field.

### Data the widgets surface

Once rendered:

- **Prowlarr:** number of grabs, queries, failed grabs, failed queries — over some time window (24h by default)
- **Lidarr:** wanted albums, queued downloads, monitored artists

All zero right now because Lidarr has no indexers configured and no artists to monitor. That's fine — the widget is proving the wiring works. Actual values will appear when the services start doing real work.

### Contrast with the Grafana widget

Yesterday's Grafana widget took ~45 minutes of debugging because Grafana's Admin HTTP API refuses service account tokens by design — it requires basic auth with a Server Admin user. Homepage's widget doc doesn't flag this; Grafana's does but you have to know to look.

Tonight's *arr widgets worked first try because:
1. API keys are Prowlarr/Lidarr's *only* API auth mechanism (no user/service-account distinction)
2. Homepage's widget documentation for these matches reality exactly

**Lesson:** the pattern of "generate API key, paste into config" only works when the target service has a first-class notion of API keys. When it doesn't (Grafana's tiered permission model), you'll hit friction. Reading target service's own API docs — not the client's — is where you find the constraints.

---

## Part 2: The SSH hardening pass — motivation

Halfway through the session you flagged that mixing "some containers have SSH set up, some don't, one had it hacked-in yesterday to unblock Homepage editing" felt like security debt.

That's a genuinely good instinct. **Security debt compounds:** doing this now with 7 containers is much easier than doing it in 6 months with 15+ containers. The pattern you learn also transfers everywhere — SSH keys are how Linux server administration actually works.

### What "hardening" means here

Three specific improvements:

1. **Every container reachable via SSH key from Ubuntu** — no manual password entry per container
2. **Password SSH globally disabled** — even if someone guessed a container root password, SSH would refuse
3. **Passphrase-protected key** — Ubuntu compromise doesn't automatically compromise the lab

### The threat model this fits

For a homelab on your LAN behind NAT:
- Password SSH from the LAN is a low but real risk (compromised device on Wi-Fi scanning for services)
- Password SSH exposed to the internet is a catastrophic risk (constant automated attacks, tens of thousands of attempts per day per SSH port)
- Key-only SSH is the industry-standard baseline; attackers with even sophisticated tools cannot brute-force a properly generated ed25519 key

You're on the low-risk end today (LAN only). But when Tailscale rolls out and the lab becomes reachable from your phone, or when you eventually port-forward something intentionally, the baseline needs to already be right. Fix it before you need it.

---

## Part 3: Audit — the starting state

First step of any hardening: know what you have.

### Password status check

```
for id in 100 101 102 103 104 105 106; do
  echo "=== LXC $id ==="
  pct exec $id -- passwd -S root 2>/dev/null || echo "container not running"
done
```

The `passwd -S <user>` command shows password status. Second field of output:

- **P** — password set (usable for login)
- **L** — locked (no password login possible; account can still be reached via key or su)
- **NP** — no password set at all

Result: all seven containers showed **P** with dates matching each build session. Nothing to backfill on the password front.

### Bitwarden cross-check

Passwords existing on the containers is only half the story — you need them stored somewhere retrievable. Confirmed all seven had corresponding Bitwarden entries, though naming was inconsistent (older ones as `PiHole CT Root`, newer as `Prowlarr LXC - root`). Not urgent to rename, worth flagging for a future tidy-up session.

### SSH config check

```
pct exec 100 -- grep -i permitrootlogin /etc/ssh/sshd_config
```

Result: `#PermitRootLogin prohibit-password` (commented out, defaults to `prohibit-password`).

**Meaning:** Debian defaults `PermitRootLogin` to `prohibit-password`, which allows root SSH only via key auth, not password. Since there was no key set up yet, root was effectively unreachable via SSH at all.

This is Debian's opinionated secure default: force password-less auth for root. The right position from a security standpoint, but it made the initial "let me SSH in and set up keys" chicken-and-egg tricky.

---

## Part 4: The chicken-and-egg — temporarily enabling password SSH

You can't `ssh-copy-id` a public key to a server that refuses password auth. So step one is to temporarily open the door.

### The batch fix

```
for id in 100 101 103 104; do
  echo "=== LXC $id ==="
  pct exec $id -- sed -i 's/#PermitRootLogin prohibit-password/PermitRootLogin yes/' /etc/ssh/sshd_config
  pct exec $id -- systemctl restart ssh
done
```

Skipped 102 (Homepage, enabled yesterday), 105 (Prowlarr, community-scripts install already enables it), 106 (Lidarr, same).

### Understanding the sed command

- `sed -i` — edit the file in place (as opposed to printing to stdout)
- `s/PATTERN/REPLACEMENT/` — substitute pattern with replacement
- `#PermitRootLogin prohibit-password` matches the commented-out default line
- `PermitRootLogin yes` is the replacement (uncommented, changed value)

This is idempotent-ish — running it twice doesn't break anything because the second run finds no match (the pattern is gone after the first run) and does nothing. Not truly idempotent because a manually-edited file with different formatting wouldn't match, but for consistent Debian defaults it's fine.

### Restart ssh, not sshd

The service name in Debian is `ssh`, not `sshd`. On Red Hat/CentOS/Fedora it's `sshd`. Minor cross-distro annoyance worth remembering. `systemctl restart ssh` on Debian; `systemctl restart sshd` on RHEL-family.

### Verify

```
for id in 100 101 102 103 104 105 106; do
  echo "=== LXC $id ==="
  pct exec $id -- grep -i "^PermitRootLogin" /etc/ssh/sshd_config
done
```

All seven showed `PermitRootLogin yes`. Door open, ready for key distribution.

Note the `^` in the grep — matches only lines starting with `PermitRootLogin`, filtering out the commented explanations that also mention it. Small refinement but important when auditing configs.

---

## Part 5: Generating the SSH key

**On Ubuntu (client), never on Proxmox (server).** The key represents *you* — it belongs to the machine you're SSH'ing *from*.

```
ssh-keygen -t ed25519 -C "johnny@ubuntu-lemonlab"
```

### Ed25519 vs RSA

`-t ed25519` picks Ed25519, an elliptic-curve algorithm. Modern default since ~2019. Advantages:

- **Shorter keys** — 68 characters vs 400+ for RSA-4096
- **Faster generation, faster auth** — negligible on modern hardware, matters at scale
- **Cryptographically stronger for the equivalent key size**

Only reason to still use RSA is compatibility with genuinely ancient SSH servers (Debian oldstable from 5+ years ago, certain embedded systems). Nothing in your lab qualifies.

### The comment field

`-C "johnny@ubuntu-lemonlab"` embeds a comment in the public key. It's purely for human identification — appears at the end of the public key file, useful when you accumulate multiple keys and need to know which is which. Format is convention (`user@host`), not enforced.

### The passphrase decision

Two paths presented:

- **Passphrase:** key file encrypted at rest, requires typing/pasting the phrase to unlock. Compromise of your Ubuntu account/disk doesn't automatically compromise the lab.
- **No passphrase:** SSH just works with zero friction, but the key file is directly usable by anyone who reads it.

Chose passphrase. Correct choice for a lab that's also a security learning environment — the friction is trivial (once per session via ssh-agent), the security upside is real.

Passphrase saved to Bitwarden as `SSH Key - LemonLab passphrase` **before** typing it into the ssh-keygen prompt — because if you mistype the confirmation twice, you'll lock yourself out of the key you just generated.

### The randomart

When the key generates, SSH prints a colourful ASCII-art "randomart" — a visual fingerprint of the public key. Purpose: if a server ever presents a different randomart than expected on connection, that's a signal something's wrong (server key changed, potential man-in-the-middle). Nobody memorises them; it's a "you'd notice a change" feature rather than "verify every time."

### The resulting files

```
~/.ssh/id_ed25519       # private key — 600 permissions, never share
~/.ssh/id_ed25519.pub   # public key — 644 permissions, distributed to servers
```

Permissions matter: SSH refuses to use a private key that's world-readable (mode 644 on `id_ed25519` would silently fail with a permissions error). This is a safety feature, not a bug.

---

## Part 6: Distributing the public key with ssh-copy-id

`ssh-copy-id` is a small helper that automates the "read public key, SSH into target, append to `~/.ssh/authorized_keys`" workflow.

```
ssh-copy-id root@192.168.x.101
```

### What it actually does

1. Reads your default public key (`~/.ssh/id_ed25519.pub`)
2. Tries to connect via key auth first (checks if you're already set up)
3. Falls back to password auth if not
4. Once in, appends your public key to the target's `~/.ssh/authorized_keys`
5. Sets correct permissions on the target's `.ssh` directory and file

### The alternative

Without `ssh-copy-id` you'd do this manually:

```
cat ~/.ssh/id_ed25519.pub | ssh root@target 'mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 700 ~/.ssh && chmod 600 ~/.ssh/authorized_keys'
```

`ssh-copy-id` is just this baked into a friendlier command. Worth knowing the manual form for when `ssh-copy-id` isn't available (some minimal container images don't ship it).

### The batch

Ran through all seven containers, one at a time. Each prompted for the container's root password once, then succeeded. After each: verified with `ssh root@<ip> hostname` — should print the container's hostname silently with no prompt at all.

### ssh-agent caching

First `ssh-copy-id` prompted for the master passphrase (unlocking your private key to use it). After that, every SSH command in the session ran passphrase-free — ssh-agent had cached the unlocked key.

**Explicit invocation if agent didn't auto-cache:**

```
ssh-add ~/.ssh/id_ed25519
```

Enter passphrase once, cached for the duration of the Ubuntu login session.

**Verify agent is running:**

```
echo $SSH_AUTH_SOCK
```

If it prints a path (e.g. `/run/user/1000/gcr/ssh`), agent is running. If empty, need to start it explicitly with `eval "$(ssh-agent -s)"`.

Modern Ubuntu with a graphical session starts an agent automatically as part of the desktop startup — you rarely need to manage this manually.

---

## Part 7: Disabling password SSH globally

With keys working on all seven containers, close the temporary door.

### The batch

```
for id in 100 101 102 103 104 105 106; do
  echo "=== LXC $id ==="
  pct exec $id -- sed -i 's/^PermitRootLogin yes/PermitRootLogin prohibit-password/' /etc/ssh/sshd_config
  pct exec $id -- systemctl restart ssh
done
```

Flipped every container from `PermitRootLogin yes` back to `PermitRootLogin prohibit-password`. That's Debian's default, and its meaning is:

- **Key-based root SSH:** allowed
- **Password-based root SSH:** refused

Result: root can still SSH in from Ubuntu (because keys are set up), but nothing on the network can brute-force a password.

### Testing "refused" properly

Attempted a deliberate password-only SSH:

```
ssh -o PreferredAuthentications=password -o PubkeyAuthentication=no root@192.168.x.101
```

The `-o` flags force SSH to only offer password auth and not attempt key auth. This tests what happens when a password-only client tries to connect.

Result: password prompt appeared, but entering *any* password returned "Permission denied." The prompt is cosmetic — sshd asks even though it won't accept the answer. What matters is the rejection after typing.

**This is subtle and worth remembering:** the prompt appearing is not the same as "password auth working." The only definitive test is "did it actually let me in with the correct password?" If it prompts and rejects, password auth is disabled correctly.

### Verify key auth still works

```
ssh root@192.168.x.101 hostname
```

Silent output of `pihole` — no prompts. Key auth working, password auth blocked. Hardening successful.

---

## Part 8: SSH aliases via ~/.ssh/config

Typing `ssh root@192.168.x.107` gets old fast. SSH client supports host aliases via a config file.

### The file

```
nano ~/.ssh/config
```

Contents:

```
Host proxmox
    HostName 192.168.x.100
    User root

Host pihole
    HostName 192.168.x.101
    User root

# ... one block per container
```

### Permissions

```
chmod 600 ~/.ssh/config
```

SSH refuses to use a config file with permissions loose enough that other users could read it. Not because the file contains secrets (it usually doesn't), but as a defence against attackers modifying your config to redirect SSH connections to malicious hosts.

### What you get

```
ssh pihole                    # → ssh root@192.168.x.101
scp file.txt homepage:/tmp/   # works with aliases too
ssh grafana hostname          # tab-completes "grafana" from your config
```

Every SSH-based tool (`scp`, `rsync`, `sftp`, `git` over SSH) respects the same config. Set up once, benefit everywhere.

### Additional options worth knowing

The alias block can carry more than just Host/User. Common additions:

```
Host proxmox
    HostName 192.168.x.100
    User root
    IdentityFile ~/.ssh/id_ed25519       # explicit key selection if you have multiple
    ServerAliveInterval 60                # send keepalive every 60s to prevent NAT timeout
    ForwardAgent yes                      # allow SSH-through-SSH (jump host scenarios)
```

Not needed for this lab yet, but worth knowing exists.

---

## Part 9: Recovery paths worth documenting

Locking yourself out of your own systems is the classic hardening self-inflicted wound. Two recovery paths matter:

### Path 1: Ubuntu SSH key is lost or Ubuntu dies

You cannot SSH into any container. Recovery:

1. Proxmox web UI → each container → Console tab
2. Log in with the root password from Bitwarden (still works via console)
3. Re-enable password SSH temporarily (`sed` command from Part 4)
4. From a new Ubuntu install: generate new SSH key, `ssh-copy-id` to each container
5. Disable password SSH again

The Proxmox web console is the safety valve. It bypasses SSH entirely and uses Proxmox's own web-based terminal, authenticated by Proxmox login (which is separate from container credentials).

### Path 2: Proxmox login is lost

More painful. Proxmox root password is stored where? Confirm it's in Bitwarden. If Proxmox login is also lost, the recovery is physical console access to the ThinkCentre — plug in monitor and keyboard, boot into single-user mode or a recovery image, reset the password.

**Action item:** confirm Bitwarden has an entry for Proxmox root login itself, not just the containers. If not, add it now.

---

## What's actually running now

Unchanged from yesterday on the service front:

- Proxmox host + 7 LXCs, all healthy
- Homepage now displays two groups: Infrastructure (4 services) + Media (2 services)
- All widgets rendering live data

New this session:

- SSH key auth across all containers, keys managed from Ubuntu via ssh-agent
- Password SSH disabled on every container
- SSH aliases on Ubuntu — `ssh <name>` works for every container
- Bitwarden entries for the new API keys (`Prowlarr - API`, `Lidarr - API`) and the SSH passphrase (`SSH Key - LemonLab passphrase`)

Still deferred:

- Storage for Lidarr — pending NAS timeline decision
- Indexer configuration on Prowlarr
- Tailscale for remote access

---

## Things worth reading further

- **OpenSSH server configuration** — [man.openbsd.org/sshd_config](https://man.openbsd.org/sshd_config). The canonical reference. Every option Debian/Ubuntu ships is documented here. The `PermitRootLogin` section is particularly worth reading — it explains the four options (`yes`, `prohibit-password`, `forced-commands-only`, `no`) and their implications.
- **OpenSSH client configuration** — [man.openbsd.org/ssh_config](https://man.openbsd.org/ssh_config). Everything `~/.ssh/config` supports. Worth skimming to see what's available beyond the basics.
- **SSH Mastery** by Michael W. Lucas — short technical book, best single reference for going deep on SSH. Under 200 pages, worth reading straight through if SSH is going to be a daily tool for you (which it now is).
- **The "Security keys are the answer" article by Filippo Valsorda** — [blog.filippo.io/on-the-web-authentication-arms-race/](https://blog.filippo.io/) — broader context on why cryptographic auth beats password auth, extending to hardware keys/passkeys as the next step.
- **Homepage widget docs** — [gethomepage.dev/widgets/services/](https://gethomepage.dev/widgets/services/). Full catalog of widget types and their config options. Worth bookmarking as you add more services.

---

## Concepts that came up in passing worth knowing about

- **Ed25519** — elliptic-curve signature scheme designed by Daniel Bernstein and others. Standardised in RFC 8032. Widely used beyond SSH: TLS, TOR, cryptocurrency wallets, code signing. Understanding "this is the modern default asymmetric algorithm" is enough for most purposes; the mathematics is a rabbit hole.
- **ssh-agent** — background process that holds unlocked private keys in memory for the duration of a session. Uses a Unix socket (`$SSH_AUTH_SOCK`) for other processes to request signing operations without ever seeing the key material. Standard on every modern Unix desktop.
- **`~/.ssh/authorized_keys`** — the file on a server listing which public keys are permitted to log in as that user. One public key per line. When you `ssh-copy-id`, this is what gets appended. Reading it manually is instructive: `cat ~/.ssh/authorized_keys` on any container will show your Ubuntu public key.
- **Known-hosts and host key verification** — the first time SSH connects to a new host, it stores that server's public key fingerprint in `~/.ssh/known_hosts`. Every subsequent connection checks the presented key against this record — a mismatch triggers the loud "REMOTE HOST IDENTIFICATION HAS CHANGED" warning. This is the client-side defence against man-in-the-middle attacks.
- **`sed -i`** — stream editor, edit files in place. The `s/old/new/` substitution pattern is one of the most-used shell idioms. Worth learning slowly — the `sed` man page is dense but the fundamentals repay the investment.
- **`for id in 100 101 102...`** — shell for-loop syntax. `for VAR in LIST; do COMMANDS; done`. When you have parallel operations across similar items (like all your container IDs), this pattern saves enormous amounts of typing. Combines with `pct exec` beautifully for lab-wide operations.
- **`PubkeyAuthentication` vs `PasswordAuthentication` vs `PermitRootLogin`** — three related but distinct SSH server options. The first two control which auth *methods* are allowed for all users. The third restricts what specifically the root user can do, and can override the general settings for root. Their interaction is subtle; sshd_config's ordering and defaults matter. When debugging "why can I/can't I log in as root", check all three.
- **File modes and SSH strictness** — SSH is unusually paranoid about permissions on its own config files. Config too-loose = SSH refuses to use it. This catches new users out but exists because SSH failing safe is preferable to SSH silently using a config an attacker could have modified. Modes matter: 700 on `~/.ssh/`, 600 on `~/.ssh/config`, `~/.ssh/id_ed25519`, and `~/.ssh/authorized_keys`. 644 on `~/.ssh/id_ed25519.pub` is fine (public keys are, by definition, public).
- **Shell command chains and error handling** — the `for` loops we ran don't stop on error. If one container's sshd restart failed, the loop would carry on and you'd have to spot it in the output. For anything critical, add `|| break` or set `set -e` at the top of the script. For quick admin, the loose behaviour is usually what you want.

---

*Written from the LemonLab session of 1 September 2026. Bookmark alongside the previous notes for reference.*
