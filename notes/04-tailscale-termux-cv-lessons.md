# LemonLab: Fifth Day Study Notes

*Today was three sessions in one: a scoping conversation that correctly killed the download-client work by recognising an unsatisfied gating item (no VPN), a full Tailscale rollout to give the lab off-network reachability, and a Termux setup on the phone with dedicated SSH key auth. Plus a CV source-material sidebar that produced two document versions and a good lesson about self-critique in technical writing.*

---

## Where we started

Yesterday finished with Homepage tiles rendering for Prowlarr and Lidarr, and a full SSH hardening pass across the lab (key-only auth, password SSH disabled everywhere, aliases on the client). Prowlarr and Lidarr were installed but not yet doing anything — no indexers, no storage, no download client.

The plan for today was to configure Prowlarr indexers and connect Lidarr as an "app" so the search chain would work end-to-end. That plan was correctly abandoned when the VPN dependency became clear.

By the end of the session: Tailscale is deployed on the Proxmox host with the phone as a second node, off-network access to Proxmox is proven end-to-end via mobile data, Termux is installed on the phone with its own passphrase-protected SSH key that has been authorised on Proxmox, and CV source material for the LemonLab work exists as v2 (after v1 was rejected for overclaiming — that lesson matters more than the technical work below and is captured properly further down).

---

## Part 1: The gating-item catch — why we didn't configure indexers

Started the session ready to configure Prowlarr indexers, discussed the three options (public torrent, Usenet, private tracker), agreed on public torrents as the sensible starting point. Then — mid-instructions — one question stopped everything:

> "Should I be doing any of this before I have AirVPN installed?"

The correct answer was: no. And the important thing is that the question came from you, not from a checklist. Naming this because it's a genuinely useful pattern to internalise.

### What was about to go wrong

Configuring indexers itself is low risk — Prowlarr's search queries are HTTPS to tracker search endpoints, no torrent traffic at all. That part is fine.

**The trap was momentum.** Once indexers work and Lidarr sees them, the natural next step is "add a download client so Lidarr can actually grab something." A qBittorrent LXC takes about 15 minutes to deploy. From there the trap is "just test one download to prove the pipeline works."

That test download would have been on your residential IP, unencrypted, joining a public swarm where anti-piracy monitoring is routine. First offence in the UK is usually an ISP letter; repeated offences lead to service warnings and potentially termination. Rights-holder speculative-invoicing letters (£500-£1000 to avoid court) are rare but real.

### The pattern worth remembering

**Recognise gating items before they gate you.** The gating item here was AirVPN (pending wages payment). The correct response was to pause the whole *arr work stream — not just download client setup, but everything downstream — until VPN is deployed with a leak-tested download client.

The wrong response would have been "let me set up the indexers now while I'm here, and we'll add the download client after AirVPN." That works only if you have iron discipline against the intermediate steps, and I'd bet against that discipline in almost anyone once the pieces are visibly on the table.

Related pattern: **if a critical dependency is pending on a real-world event (wages, a delivery, a person's availability), don't optimise around it.** Do work that doesn't depend on the pending thing. When the dependency lands, do the dependent work. Skipping this discipline is how people end up with half-configured systems that never quite get finished properly.

---

## Part 2: The Frigate/camera detour

Related but standalone thread — the discussion of self-hosted video/NVR software.

### AgentDVR vs Frigate quick comparison

Both do the same job (record from IP cameras, motion detection, playback, alerts). They differ significantly:

- **AgentDVR** is Windows-first, GUI-heavy, freemium (paid tier for advanced features), cloud-optional AI
- **Frigate** is Linux/Docker-native, config-file driven (YAML), open source with no paid tier, has built-in local object detection via TensorFlow/Google Coral/Intel iGPU

For a self-hosting lab that aligns with de-Google/de-cloud principles, **Frigate is the right architectural fit.** No cloud dependency, config-as-code matches your existing workflow (Homepage, Prometheus, LXC scripts), Prometheus can scrape its metrics.

### But — the honest recommendation was to defer

Frigate's real value is only unlocked with:
- **Cameras that actually exist** (you have one webcam)
- **A network capable of isolating cameras on their own VLAN** (blocked pending sparky job)
- **Storage for continuous recording** (blocked pending NAS)
- **Object-detection hardware** (Coral USB accelerator ~£60, or Intel iGPU passthrough on the host)

Deploying Frigate now with none of those would repeat the "monitoring stack watching one service" pattern. Correct call: defer.

### The immediate practical outcome

You mentioned an active reason to want a camera up quickly. The recommended path was:

- **Reolink E1 Zoom** — small indoor camera, £35-50, RTSP-capable (works with any client, any OS)
- **microSD card in the camera** for local recording independent of any host
- **Motion on Ubuntu** for immediate NVR functionality — 24/7 stream monitoring, motion-triggered clips, optional ntfy push alerts
- **Same camera plugs into Frigate later** when the network buildout is done — no re-buying

This is a good example of "solve the actual problem now with the boring available answer, keep the architectural flexibility for later."

---

## Part 3: Tailscale — what it is and what it isn't

Before installing, worth a moment on the distinction that catches people out:

### Tailscale vs commercial VPN

Different problems, different tools. Easy to conflate because both are called "VPN."

**Tailscale (mesh VPN):**
- Lets *your devices* reach *each other* privately, regardless of physical location
- Your phone on mobile data can reach your Proxmox host at home as if they were on the same LAN
- Doesn't hide your traffic from your ISP
- Doesn't route torrent or general internet traffic through anywhere useful
- Free tier: 100 devices, ample for a homelab

**Commercial VPN (AirVPN, Mullvad, ProtonVPN):**
- Routes your outbound internet traffic through the provider's servers
- Destination sees the VPN provider's IP, not yours
- Hides content from your ISP
- Used for torrenting, geo-unblocking, general privacy from network operators

You need both eventually. They serve different jobs and shouldn't be conflated.

### How Tailscale works, in one paragraph

Every device you install Tailscale on registers with Tailscale's coordination server, gets assigned an IP in the 100.64.0.0/10 range (specifically 100.64-127.x.x per device), and authenticates to the mesh via your chosen identity provider (Google/Microsoft/GitHub/etc.). When one Tailscale device wants to reach another, they negotiate a direct encrypted peer-to-peer connection using WireGuard. Traffic goes device-to-device, not through Tailscale's servers — the coordinator only handles introductions and NAT traversal. That's why it's fast (wireguard is fast) and secure (peer-to-peer means Tailscale itself can't read your traffic).

### The architectural decision made

Two ways to deploy Tailscale in a Proxmox environment:

1. **Host-only:** install Tailscale on the Proxmox host itself. From outside your network you reach Proxmox directly, then use `pct enter` or LAN IPs to reach containers.

2. **Per-container:** install Tailscale on each LXC as well. Each container becomes its own Tailscale node, reachable directly from anywhere on the tailnet.

Went with **host-only** for now. Reasons: simpler starting point, fewer things to manage, proves the concept works before adding complexity. Per-container can come later if the pattern of "SSH to Proxmox then jump to container" becomes tedious.

A third option — **Proxmox as a subnet router advertising the whole 192.168.x.0/24 to the tailnet** — is more elegant than per-container installs and worth exploring later. Requires enabling IP forwarding on the host and approving the routes in the Tailscale admin console.

---

## Part 4: Installing Tailscale on Proxmox

Standard install pattern for their supported platforms.

### The install script

```
curl -fsSL https://tailscale.com/install.sh | sh
```

**What the script does:**
1. Detects the OS and package manager (apt on Debian 13)
2. Adds Tailscale's official APT repository and GPG signing key
3. Installs `tailscale` (CLI + client) and `tailscaled` (daemon) packages
4. Enables and starts the `tailscaled` systemd service

Nothing exotic. Modern install scripts from reputable projects (Tailscale, Docker, Rust) are safe to pipe to `sh` — Tailscale's is well-audited and their repository/signing infrastructure is standard.

### Bringing the node online

```
tailscale up
```

This prints an authentication URL like `https://login.tailscale.com/a/xxxxxxxxxx`. Copy it, open in a browser where you're logged into Tailscale's admin console (previously auth'd via GitHub SSO), and authorise the machine.

Behind the scenes: `tailscale up` starts the WireGuard tunnel process, but requires interactive authorisation on first use to prevent unauthorised machines from joining your tailnet. Subsequent restarts don't require re-authorisation — the node identity persists.

### Verifying the node

```
tailscale status
```

Shows all machines on your tailnet with their Tailscale IPs. Proxmox landed on `100.x.x.x`.

```
tailscale ip -4
```

Just the IPv4 Tailscale address of the current machine — useful in scripts.

### Why the Tailscale IP looks weird

`100.64.0.0/10` is the carrier-grade NAT (CGNAT) range, reserved for ISPs to use internally when they run out of public IPv4 addresses. Tailscale deliberately uses this range because:

- It's guaranteed not to conflict with your home network (192.168.x.x)
- It's guaranteed not to conflict with the public internet (nothing routes to these addresses publicly)
- Most systems don't have static routes for this range, so Tailscale's own routing takes precedence cleanly

Downside: on some networks that already use CGNAT (some mobile carriers, some hotel Wi-Fi), you might see conflicts. Tailscale handles this reasonably well but it's the reason for occasional weird routing issues.

---

## Part 5: The phone-side install and the "connected but not connected" bug

Installed Tailscale on the Android phone from Google Play. Logged in with the same GitHub identity. Phone appeared in the admin console immediately.

### The bug worth documenting

The Tailscale Android app currently has a UI desynchronisation bug: the notification shade correctly showed "Connected", but the app's own home screen showed "Not connected" with a big blue "Connect" button that did nothing when tapped.

This confused the initial troubleshooting. Tried force-stop, tried tapping various toggles, tried the notification-panel disconnect/reconnect. Eventually realised the app UI was lying — the actual VPN tunnel was up regardless of what the app screen claimed. Confirmed by simply attempting the target action (browsing to Proxmox from the phone) and finding it worked.

**Lesson:** when an app's status display disagrees with the OS-level status (notification shade, system settings), **trust the OS-level status**. App UIs can lie or lag; the OS knows what network state actually exists.

Related lesson: always try the actual target action to test whether something works, rather than trusting the software's self-report. "The app says connected" is a claim; "I can reach the target host" is evidence.

### The off-network test

Turned off Wi-Fi on the phone. Enabled mobile data only. Opened browser and went to `https://100.x.x.x:8006` (Proxmox's Tailscale IP + web UI port).

Got the classic certificate warning: "Your connection is not private — NET::ERR_CERT_AUTHORITY_INVALID."

**This warning was actually the win.** Getting a cert error means:

- The TCP connection to the target succeeded (network reached Proxmox)
- TLS handshake completed (Proxmox responded with a certificate)
- The browser only complained because the certificate was self-signed

If Tailscale had failed, you'd have gotten "connection timed out" or "unable to connect" — no response at all. Getting a working TLS handshake with a valid-but-untrusted certificate means the whole path worked. The warning is just Proxmox using a self-signed cert (its default), which is fine on your own tailnet where you know it's you.

Click Advanced → Proceed → land at Proxmox login page. Test passed.

### Why self-signed certificates trigger big warnings

Browsers trust certificates signed by known Certificate Authorities (Let's Encrypt, DigiCert, GlobalSign, etc.) via a chain-of-trust rooted in the browser/OS certificate store. A self-signed certificate has no such chain — it's just Proxmox saying "trust me, I'm Proxmox," and the browser has no way to independently verify that claim.

On the public internet, this is important — a self-signed cert on a login page is genuinely a red flag (could be a phishing site). On your own tailnet reaching your own Proxmox, you know it's you, so click-through is safe.

**Longer-term fixes** (not tonight): Proxmox has built-in ACME support for Let's Encrypt certificates. Or Tailscale has its own HTTPS certificate provisioning for tailnet-only services. Either eliminates the warning permanently.

---

## Part 6: Termux and the second SSH key

Getting the Proxmox web UI to work from the phone was the main test. Getting shell access from the phone was the follow-up — genuinely useful for "something's broken while I'm away and I need to fix it."

### Termux — what it actually is

Termux is a terminal emulator for Android with a full Linux userspace shipped inside the app's own storage. Includes a proper package manager (`pkg`, wrapper around apt), tools like SSH, git, Python, curl, and hundreds of others. Runs entirely in the app sandbox — no root required, no system modifications.

Common misconception is that Termux needs a rooted phone. It doesn't. It runs as an ordinary Android app in ordinary Android userspace.

### Where to get it

**Do not install from Google Play.** Google forced Termux there to stay on old Android API levels, which broke it. The Play Store version is effectively abandoned.

**Correct sources:**
- **F-Droid** — recommended, aligns with de-Googling
- **GitHub releases** — official builds

Both are the actively-maintained current version. Installed via F-Droid.

### The wrinkle nobody warns you about: VPN traffic routing

First SSH attempt from Termux (`ssh root@100.x.x.x`) failed with `Network is unreachable`. This is not obvious.

Cause: **Android's per-app VPN routing.** When Tailscale establishes its VPN tunnel, Android decides which apps' traffic goes through the tunnel. By default this includes normal apps (browsers, mail), but Termux — because it runs in its own sandbox with its own network handling — sometimes gets excluded.

**Fix**: turn Wi-Fi off entirely, use mobile data only. This forces all traffic through mobile, and Tailscale correctly captures it including Termux.

Alternative fix (didn't need it): Tailscale app has a per-app inclusion/exclusion setting where you can explicitly include Termux.

Once Wi-Fi was off, `ssh root@100.x.x.x` reached Proxmox — but got refused because Proxmox only allows key auth and Termux has no keys.

### The right approach: separate SSH key on the phone

The tempting shortcut would have been to temporarily re-enable password SSH on Proxmox, prove Termux can reach it, then disable password SSH again.

**Correct approach: generate a separate SSH key on the phone.** Same discipline as the Ubuntu setup. Each device has its own key. If the phone is lost/compromised, revoke that key alone — the Ubuntu key remains valid.

### Key generation on Termux

```
ssh-keygen -t ed25519 -C "johnny@android-termux"
```

Same command as on Ubuntu, same key type, different comment for identification. Passphrase-protected, saved to Bitwarden as `SSH Key - Termux passphrase`.

The public key was displayed with `cat ~/.ssh/id_ed25519.pub` and copied out for distribution.

### Distributing the public key

From Ubuntu (which already has SSH key access to Proxmox), the Termux public key was appended to `/root/.ssh/authorized_keys` on the Proxmox host:

```
ssh proxmox
echo 'ssh-ed25519 AAAA... johnny@android-termux' >> ~/.ssh/authorized_keys
tail -3 ~/.ssh/authorized_keys
```

Verified with `tail -3` that the file now contained three keys: the original Proxmox root key, the Ubuntu key from yesterday's setup, and the new Termux key.

### The verification

Back on the phone in Termux:

```
ssh root@100.x.x.x
```

Got the fingerprint prompt (first-time host verification), typed `yes`, then got prompted for the Termux key's passphrase. Entered it. Landed at `root@proxmox:~#`.

Full stack proven end-to-end: Android phone on mobile data → Tailscale WireGuard tunnel → Proxmox host → sshd → key auth against Termux key.

### The debugging insight

During this process there was a "Connection closed by 100.x.x.x" error that was misleading. The instinct was to check the key on Proxmox — but the sshd logs (`journalctl -u ssh -n 20`) showed no failed connections from the phone's IP at all. That meant the SSH attempt wasn't reaching Proxmox's sshd at that point — it was being terminated somewhere in the network path.

The fix turned out to be Tailscale reconnecting on the phone, but the debugging pattern is worth remembering: **when a service reports a failure, check the service's own logs for evidence of the failure request having arrived**. If logs show nothing, the failure is happening before the request reaches the service, and you should shift focus to the network/transport layer rather than debugging the service.

---

## Part 7: The CV source material sidebar and the lesson about overclaiming

Midway through the session, a request came in from another chat (the CV creation chat) to write source material summarising the LemonLab work in CV-usable form. Wrote a first version. It was then reviewed by that other chat, which flagged multiple overclaims and forced a v2.

The feedback was correct on every point. Worth capturing what got flagged because the pattern generalises beyond CV writing.

### The specific overclaims

1. **"Production-pattern" as inflating language.** The phrase appeared eight times across the v1 document, doing suspicious work — implying LemonLab has properties of production (HA, on-call, change control, formal RBAC, audit logging, dedicated secrets management) that it doesn't have. A sharp interviewer would call this out, and the CV writer would have to backpedal on the CV's own words.

2. **Skills matrix depth overclaimed.** Things done once, from a guided walkthrough, were rated "Working competence." The reviewer's test: could you write it cold in an interview? If not, downgrade to "Basic hands-on / familiar with pattern." Applied to Prometheus relabel rules and pve-exporter systemd unit creation.

3. **"Least-privilege API user design" as retconning.** What actually happened was applying an existing built-in Proxmox role (PVEAuditor). That's not design; it's role assignment. Downgraded to accurate language.

4. **Listing `nano` as a skill.** The reviewer's exact words: "Delete. It undermines the credibility of the whole matrix. Nobody lists nano."

5. **Listing planned/unpublished Git.** Also a delete — listing intent, not achievement.

6. **SQL and Power BI rated "Strong" while separately admitting the skills need refresh before an interview.** Internal contradiction. Downgraded to "Working competence — requires refresh before interview."

7. **200-word positioning paragraph.** Cover-letter length, not CV length. Cut to three sentences.

8. **"Systematically walked a five-step diagnostic chain" for the IP collision debug.** Retconning ad-hoc troubleshooting as systematic process. Reframed as "layered checks" without the false structure.

### The generalisable lesson

**Self-critique in technical writing is genuinely difficult.** Especially for one's own work, especially when writing for an audience who might not know enough to challenge you. The v1 document was written by me and it drifted toward inflating without me noticing — every individual sentence sounded plausible, but the cumulative effect was overclaiming.

The correction only happened because the other chat pushed back on specifics. If it had just said "looks good," the overclaims would have made it into the CV and eventually into an interview.

**Pattern to internalise:** when writing anything that will be judged (CV, technical writeup, project description), assume you're overclaiming somewhere and hunt for the specific instances. Common tells:

- Adjectives doing heavy lifting ("production," "enterprise," "professional")
- Skills rated at your best moment rather than your current state
- Impressive-sounding phrases for ordinary work ("systematic," "design," "engineering")
- Contradictions between different sections that a careful reader would spot
- Anything you couldn't defend under close questioning

The correction cost 30 minutes. Missing the correction and having it surface in an interview would have cost the interview.

---

## What's actually running now

Unchanged since yesterday on the service front. New this session:

- **Tailscale on Proxmox host** — Tailscale IP `100.x.x.x`, connected via GitHub SSO identity
- **Tailscale on Android phone** — second node, connected same identity
- **Off-network access proven** — Proxmox web UI reachable over mobile data via HTTPS
- **Termux on the phone** — F-Droid install, ed25519 key generated, passphrase-protected, key installed on Proxmox
- **Phone shell access** — `ssh root@100.x.x.x` from Termux over mobile data lands at Proxmox shell
- **GitHub account** — `johnnywbell`, created for CV publication gating item

Bitwarden entries added:
- `SSH Key - Termux passphrase`
- Consider adding: `Tailscale - Account` (login method), notes on the Proxmox entry that it's reachable at `100.x.x.x` from off-network

---

## Things worth reading further

- **Tailscale documentation** — [tailscale.com/kb](https://tailscale.com/kb). The concepts pages (how it works, MagicDNS, ACLs, subnet routers) are worth a slow read once you're comfortable with the basics.
- **WireGuard whitepaper** — [wireguard.com/papers/wireguard.pdf](https://www.wireguard.com/papers/wireguard.pdf). Short (12 pages), readable, explains the cryptographic protocol Tailscale is built on. Worth reading if you want to understand what's actually happening in the tunnel.
- **Termux wiki** — [wiki.termux.com](https://wiki.termux.com/wiki/Main_Page). Especially the "Package Management" and "Remote Access" sections.
- **Frigate documentation** — [docs.frigate.video](https://docs.frigate.video). For when the sparky/NAS work is done and Frigate becomes deployable.
- **Motion documentation** — [motion-project.github.io](https://motion-project.github.io/). For the immediate camera use case with the current webcam.
- **Reolink E1 Zoom product page** — check current UK pricing on Amazon; models change but the RTSP feature is standard across their line.

---

## Concepts that came up in passing worth knowing about

- **Mesh VPN vs commercial VPN vs site-to-site VPN.** Three different architectures for encrypted network traffic. Mesh (Tailscale, ZeroTier): device-to-device across a private overlay. Commercial (AirVPN, Mullvad): hide your outbound traffic behind a provider. Site-to-site (traditional corporate VPN, IPsec tunnels): permanently connect two networks. All called "VPN," all solve different problems.
- **WireGuard** — modern VPN protocol underlying Tailscale, faster and simpler than OpenVPN or IPsec. Kernel-level implementation, small codebase, formally verified in parts. Standard choice for new VPN deployments since ~2020.
- **CGNAT (Carrier-Grade NAT) and the 100.64.0.0/10 range.** IP addresses reserved for ISP internal use, not routable on the public internet, not conflicting with private networks. Tailscale uses this range for its overlay addresses because of these guarantees.
- **Self-signed certificates vs CA-issued certificates.** Both provide encryption; only CA-issued provide identity verification via chain-of-trust. Self-signed is fine for internal services where you already trust the endpoint; the browser warning is a feature protecting you from cases where you shouldn't.
- **First-time SSH host verification and `~/.ssh/known_hosts`.** SSH stores each server's public key fingerprint on first connection and checks against it every subsequent time. A mismatch triggers a loud warning as a defence against man-in-the-middle attacks. The fingerprint prompt on first connection is where you're supposed to verify (out-of-band ideally) that the server key is what you expect.
- **Systemd log inspection with `journalctl`.** `journalctl -u <service> -n 20` shows the last 20 log lines from a systemd service. `--no-pager` prevents it from opening in less. Essential debugging tool — when a service reports failure, check `journalctl` for evidence of what failed and where.
- **Per-app VPN routing on Android.** Android's VPN implementation lets apps be included in or excluded from the VPN tunnel individually. Most VPN apps default to "all apps included" but the mechanism exists. Termux is a common edge case because it runs in its own sandbox with its own network handling.
- **F-Droid** — a curated repository of exclusively free-and-open-source Android software. Alternative to Google Play. Rebuilds apps from source in a reproducible way, verifies the build against the developer's original. Slower to add new apps but higher trust profile than Play Store for open-source software specifically.
- **`pkg install <name>` on Termux** — wrapper around `apt install` that handles Termux's specific package repositories. Same command surface as Debian apt for the most part.

---

*Written from the LemonLab session of 2 September 2026. Bookmark alongside the previous notes for reference.*
