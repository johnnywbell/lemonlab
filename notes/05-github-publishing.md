# LemonLab: GitHub Publishing Session Notes

*Second session of 2 September 2026, covering the first real Git use: understanding what GitHub is for when the repo isn't primarily code, sorting out a mental model that had been legitimately confused by looking at complex example repos, generating a dedicated SSH key for GitHub, and getting a README plus LICENSE published as the LemonLab repo's front door. This is a companion to the fifth day study notes covering the earlier session (Tailscale/Termux/CV) — same day, different focus.*

---

## Where we started

The Career Strategy document has "Publish LemonLab to GitHub" as its first numbered action, with a target date of Sunday 7 September. That's five days away. GitHub account (`johnnywbell`) had been created earlier in the day as a side-effect of setting up Tailscale (used GitHub OAuth as the identity provider). Everything else — the actual publishing — was untouched.

By the end of this session: repository exists at `github.com/johnnywbell/lemonlab`, a professional-looking README renders on the repo home page, MIT LICENSE file is in place, dedicated SSH key for GitHub authentication is generated and functioning, and the workflow for adding future content (notes, configs, diagrams) is understood.

---

## Part 1: The mental model correction — what GitHub actually is

Started with a real confusion that deserves capturing because it's a common blocker for people new to portfolio work on GitHub.

Looked at an example homelab repository (khuedoan/homelab) that turned out to be genuinely complex — Kubernetes, FluxCD, Terraform, declarative infrastructure-as-code. Came away feeling like "this is what a homelab repo looks like and I can't produce something like this." That felt genuinely defeating for a moment.

The correction was worth spelling out: **that repo is an outlier, not the standard.**

### GitHub, in one paragraph

GitHub was built as a version control platform for code, and that's still what most repositories are. But over the years people discovered it works well for *any* structured collection of text files with change history. So now it hosts code (its original purpose), documentation, configuration files, written work like blogs and books, homelab setups, recipes, personal notes, dotfiles — anything textual where you want a durable record and public accessibility.

**The important consequence:** you don't need "code" to have a GitHub repo. A folder of markdown files, well organised and well written, is a completely valid GitHub repository.

### What khuedoan/homelab actually is

That specific repo is at the extreme end of homelab documentation. The owner's homelab *is* code — everything runs on Kubernetes, every service is defined declaratively in YAML files that get committed and applied automatically via GitOps. Their repo is a working system, not documentation. Cloning it and running the right commands would (in theory) reproduce their entire lab.

That approach requires knowing Kubernetes, Terraform/OpenTofu, Ansible, FluxCD, Ansible, container orchestration, and being willing to convert every operational decision into declarative code. It's a legitimate style but a very different discipline from what most homelabbers do.

### What most people's homelab repos actually are

**A folder of documentation about a real infrastructure build.** Not code. Not automation. Documentation-as-portfolio.

The audience is:
- Hiring managers doing background research on the candidate
- Interviewers who want to see evidence of the work claimed on the CV
- The author themselves in six months, having forgotten why a particular decision was made

That's what LemonLab's repo is going to be. Not runnable, not reproducible, not automated. Just a well-organised written record of what got built and why.

### The three example repos looked at

Three repos surfaced via web search, ranked from simplest to most-structured:

**mattulau/HomeLab** — one commit, short README, prose-only description of what runs where. No code, no automation, no complex structure. This is the "beginner personal documentation repo" archetype and is genuinely achievable in a single sitting. Uses Proxmox + Cloudflare Zero Trust. The README is essentially: purpose paragraph, stack list, overview sentence per component. That's it.

**kippesikgithub/proxmox_hardware** — description of hardware setup with config examples. Casual first-person voice ("Welcome to my github repository, where you can find everything regarding our Proxmox environment..."), no pretence of being anything other than personal notes made public. Slightly more depth than the previous.

**ThmsRyn/homelab-proxmox-starter** — different purpose (tutorial aimed at helping others build a similar setup, not just documentation). But the *directory structure* is worth studying: a top-level README as the landing page, then subdirectories per topic (network/, vms/, backups/, monitoring/, security/), each with its own README that becomes the folder's landing page when opened on GitHub.

The lesson from studying all three: none tries to be runnable or reproducible. None claims automation. All accept that "documentation of a real thing, written well" is enough.

### Why this matters beyond GitHub

The bigger pattern: **when looking at any technical artefact for reference, you have to separate "what this is" from "what I want to produce."** Looking at production Kubernetes clusters for reference when what you're building is a home Proxmox lab will always leave you feeling inadequate — you're comparing to the wrong thing. Same in writing (comparing your first blog post to established writers), in code (comparing your first project to popular OSS libraries), in career (comparing your junior CV to senior CVs).

Correct reference is *the same kind of thing at the same stage*, not the impressive-looking version of a different kind of thing.

---

## Part 2: Writing the README

Two decisions before drafting:

**Voice.** First person ("I built...") vs. third person ("This lab..."). Went with third person. Reasoning: third person reads slightly more professional, works better if the repo is referenced by someone else (a hiring manager describing it), and doesn't require constant "I" claims that can start feeling boastful. Downside: risks losing personal voice. Mitigated by keeping language conversational rather than corporate.

**Repository name.** `lemonlab` chosen. Lowercase, no spaces, matches how the project is referred to across all documentation. GitHub convention. Simple and memorable.

**License.** MIT chosen. Standard permissive license for portfolio work. Everyone recognises it. Alternatives considered (CC BY 4.0 for documentation-specifically, or unlicensed / all-rights-reserved) but MIT is the boring safe default and requires no explanation.

### The v1 draft

Structure that emerged (and stuck):

- **One-paragraph opening** describing what LemonLab is and why it exists, hardware included
- **Stack section** — bulleted list of services with one-line descriptions
- **Practices in place** — bulleted list of operational discipline (SSH keys, MFA, backups tested, docs)
- **Scope statement** — split into "real and running" vs "planned but not built"
- **Repository structure** — ASCII tree of the folder layout
- **Purpose** — one paragraph on career context
- **License** — one line

Deliberately short. The mattulau/HomeLab reference is similarly short and reads better for it. Longer READMEs feel like manuals; shorter ones feel like signposts.

### What was deliberately NOT included

- **No installation instructions** — there's nothing to install
- **No usage instructions** — there's nothing to use
- **No "how to reproduce"** — the lab exists on hardware, not as code
- **No architectural diagrams** — deferred until a real diagram exists (a placeholder would be worse than nothing)
- **No links to CV or LinkedIn** — the repo should stand alone
- **No embedded personal history** — leave that for interviews where it can be discussed properly

### The edit pass

The v1 draft was reviewed and edited before publication. Small changes, each with a specific purpose:

- "for reference and further study" added — softens portfolio-speak, acknowledges the personal-utility motivation
- "essentially a written guide on the structure and purpose of it" — plainer than the original phrasing
- "(to be upgraded in the future)" added to the USB backup line — flags awareness without overclaiming a NAS
- "making it easily searchable" added to the credentials line — practical justification for the naming convention
- "and the rationale behind them" added to the documentation line — emphasises the *why*, not just the *what*
- "in pipeline" replacing "staged" for the NAS build — less jargon-y
- "until it becomes plausible" replacing "until such time where plausible" — grammatical fix
- "latest" replacing "lastest" — typo fix

These edits collectively shifted the tone away from corporate polish and toward the actual voice of the writer. That mattered: a hiring manager reading a portfolio README should hear the candidate, not a professional editor.

### The honest scope statement — worth capturing why it matters

The most valuable single section in the README is the scope statement, specifically the "not a production environment" acknowledgement. Every homelab documentation repo faces the temptation to inflate — call things "production-grade," describe personal experiments as "enterprise-ready," imply operational maturity that doesn't exist.

Honest scope defuses this. A hiring manager reading "no HA, no on-call rotation, no formal change control, no dedicated secrets management, no audit log retention" thinks: *this person knows what production actually looks like and isn't pretending to have delivered it.* That's a stronger signal than any amount of "production-pattern" language would give.

Same principle from the CV source material work earlier the same day: overclaiming loses more credibility than honest scope loses opportunities.

---

## Part 3: The Git and GitHub mechanics

First real Git use. Setup went cleanly; capturing what each step actually does because rote copy-pasting Git commands is how people end up scared of Git.

### Step 1: Git configuration

```
git config --global user.name "Johnny Bell"
git config --global user.email "324048439+johnnywbell@users.noreply.github.com"
git config --global init.defaultBranch main
git config --global pull.rebase false
```

**What each does:**

- `user.name` and `user.email` — identity attached to every commit. Immutable per-commit, changing later requires rewriting history. Getting this right at the start matters.
- `init.defaultBranch main` — when initialising a new repo, name the default branch `main` rather than `master`. Modern default; GitHub uses `main` by default too.
- `pull.rebase false` — when running `git pull`, use merge behaviour by default rather than rebase. Sane default for beginners; rebase-vs-merge is a longer conversation.

### The GitHub noreply email

Worth spelling out because it's an easy security improvement most people don't know about.

GitHub associates commits with an email address. That address is embedded in commit metadata and visible to anyone viewing the commit publicly. Using your real email means it's discoverable via any public commit you make — a very common vector for scraping-based spam and phishing.

GitHub provides a **noreply email** in the format `<numeric-id>+<username>@users.noreply.github.com`. Commits made with this address still attribute correctly (GitHub links them to your account) but don't expose your real email.

Found under: GitHub → Settings → Emails → look for the address ending in `@users.noreply.github.com`. Also tick "Keep my email addresses private" for belt-and-braces protection.

**Critical:** set this up *before* your first commit. Retroactively changing the email on committed work requires rewriting history and re-pushing (`git commit --amend` on the last commit only; more painful for older commits).

### Step 2: Dedicated SSH key for GitHub

Existing LemonLab SSH key (`~/.ssh/id_ed25519`) was already set up from earlier work. Could have reused it for GitHub. Decided against.

Reasons for a dedicated GitHub key:

- **Different threat models.** GitHub is a public-internet service accessed via SSH. LemonLab is private infrastructure. Compromise of one shouldn't require rotating access to the other.
- **Standard practice.** GitHub's own documentation recommends per-service keys.
- **Consistent with the pattern already established** — Ubuntu key for LemonLab, Termux key for phone-based access, GitHub key for GitHub. Each device or service gets its own key. If any one is compromised, only that one needs rotating.

Generated:

```
ssh-keygen -t ed25519 -C "johnnywbell@github" -f ~/.ssh/id_ed25519_github
```

Non-default flags:
- `-C "johnnywbell@github"` — comment identifies this specific key when scrolling through `~/.ssh/`
- `-f ~/.ssh/id_ed25519_github` — filename explicitly specified, kept separate from the default `id_ed25519`

Passphrase-protected. Saved to Bitwarden as `SSH Key - GitHub passphrase`.

### Adding the public key to GitHub

Displayed the public key content:

```
cat ~/.ssh/id_ed25519_github.pub
```

Copied the single-line output. In the GitHub web UI: Settings → SSH and GPG keys → New SSH key → titled "Ubuntu desktop (LemonLab)" → key type "Authentication Key" → pasted the public key → saved.

### Configuring SSH client to select the right key

Multiple SSH keys need explicit routing. Edited `~/.ssh/config` to add:

```
Host github.com
    HostName github.com
    User git
    IdentityFile ~/.ssh/id_ed25519_github
    IdentitiesOnly yes
```

**What each line does:**

- `Host github.com` — this block applies when connecting to `github.com`
- `HostName github.com` — the actual hostname (same as `Host` here, but they can differ for aliasing)
- `User git` — GitHub SSH auth uses the username `git` for all users (GitHub's design)
- `IdentityFile ~/.ssh/id_ed25519_github` — use this specific key
- `IdentitiesOnly yes` — don't offer other keys first. Important: without this, SSH tries all your keys in sequence, and GitHub throttles or rejects clients that offer too many wrong keys before the right one. Ended up here specifically to avoid that failure mode.

### Testing the SSH connection

```
ssh -T git@github.com
```

**What this does:**

- `ssh -T` — connect but don't request a shell (`-T` disables pseudo-terminal allocation)
- `git@github.com` — user `git` at host `github.com`

First-time fingerprint prompt appeared. Typed `yes`. Should also have verified the fingerprint against GitHub's official published fingerprints at [docs.github.com/en/authentication/keys-to-verify/githubs-ssh-key-fingerprints](https://docs.github.com/en/authentication/keys-to-verify/githubs-ssh-key-fingerprints) — the received fingerprint `SHA256:+DiY3wvvV6TuJJhbpZisF/zLDA0zPMSvHdkr4UvCOqU` matches GitHub's ED25519 key, so it was legitimate, but the discipline of checking is worth building.

Success message:

```
Hi johnnywbell! You've successfully authenticated, but GitHub does not provide shell access.
```

That's what you want — GitHub confirms it recognises the key and knows who you are, but doesn't provide a shell (they're a code hosting service, not a hosting provider).

### Creating the repo on GitHub

Web UI:

1. + icon → New repository
2. Name: `lemonlab`
3. Description: `Self-directed infrastructure lab on Proxmox VE — portfolio and technical writeups`
4. Public
5. **Do NOT** tick "Add a README file" (would conflict with the local README when pushed)
6. **Do NOT** tick "Add .gitignore" (not needed yet)
7. **Do NOT** tick "Choose a license" (LICENSE file will be added locally)

The three "Do NOT" items matter. Ticking any of them creates files in the GitHub-side repo before your local push arrives, causing conflicts on first push. Empty remote + fully-prepared local = smooth first push.

### Local repo setup

```
mkdir -p ~/repos/lemonlab
cd ~/repos/lemonlab
git init
git remote add origin git@github.com:johnnywbell/lemonlab.git
```

**Each command:**

- `mkdir -p` — create directory, including parents if needed. `-p` prevents error if parent doesn't exist.
- `cd` — change directory. All subsequent commands run relative to here.
- `git init` — create the `.git/` folder inside the current directory, marking it as a Git-tracked project. Doesn't touch existing files.
- `git remote add origin git@github.com:...` — register the GitHub URL as a remote called `origin`. Convention: `origin` is the primary remote. Format for GitHub SSH URLs: `git@github.com:USER/REPO.git`.

Verified with `git remote -v` — shows the URL for both fetch and push (usually identical).

### The staging → commit → push cycle

Wrote README.md and LICENSE files in the local repo directory.

```
git status
```

Shows files as "Untracked" — Git sees them but isn't managing them yet.

```
git add README.md LICENSE
```

Moves files into the staging area. Alternative shorthand `git add .` stages everything; explicit filenames are safer while learning.

```
git status
```

Now shows files as "Changes to be committed" — moved from untracked to staged.

```
git commit -m "Initial commit: README and LICENSE"
```

Snapshots the staged files into a permanent point in Git history. `-m "..."` provides the message inline; without it, Git opens a default editor for a longer message.

```
git log
```

Shows the commit history. First commit had hash `1c4964e92881096d2c7abf440eacdce0e50cf211`. That hash is a cryptographic fingerprint of the commit's contents and identity — future references to this specific commit use it.

```
git push -u origin main
```

Uploads local commits to GitHub. `-u` tells Git to remember that local `main` tracks remote `main` — after this initial push, `git push` with no arguments works.

Output confirmed success:

```
* [new branch]      main -> main
branch 'main' set up to track 'origin/main'.
```

Refreshed GitHub in the browser. README rendered with proper formatting on the repo home page. LICENSE visible in the file listing.

### The staging area — why it exists

Worth spelling out because it's the concept new Git users find most annoying and then, months later, appreciate.

Git distinguishes:
- **Working directory** — the files on disk
- **Staging area** — a holding zone for the next commit
- **Repository** — the permanent history of commits

Changes flow: working directory → staging area → repository.

For two files, staging feels like ceremony. But when you're editing 30 files and only want to commit 10 of them in a meaningful group, the staging area is the mechanism. It lets you compose commits deliberately rather than committing whatever happens to be modified.

Understand it once. Learn the shortcut `git add .` for bulk staging when you actually want it. Move on.

---

## Part 4: What's next — a workflow, not a task list

The README publication was the hardest part conceptually. Adding future content is now mechanical:

**For each new file to publish (a sanitised study note, a config file, a diagram):**

1. Sanitise the content (remove real IPs, credentials, personal specifics)
2. Save the file in `~/repos/lemonlab/notes/` (or appropriate subdirectory)
3. `git add <file>`
4. `git commit -m "descriptive message"`
5. `git push`
6. Refresh GitHub to verify

Each of those steps is 10-30 seconds once the workflow is familiar. The bottleneck becomes sanitisation, not Git.

**Suggested near-term additions to the repo:**

- `notes/README.md` as the notes-folder landing page, listing what's inside with brief descriptions
- Each existing study note as `notes/01-initial-build.md`, `notes/02-monitoring-stack.md`, etc.
- A `configs/` subdirectory with sanitised versions of key config files (Prometheus, Homepage services.yaml, Proxmox backup job definition)
- Eventually: architectural diagrams as PNGs

**The sanitisation checklist** for anything going into the repo:

- Replace real IP addresses (`192.168.1.x`, Tailscale `100.x.x.x`) with generic placeholders (`192.168.x.y` or the RFC1918 examples like `10.0.0.x`)
- Remove any API keys, tokens, passwords in full
- Remove references to specific Bitwarden entry names
- Remove personal identifiers beyond the professional email address
- Check for any hostnames that identify specific hardware in a way that could inform an attacker
- Cross-check against the study notes — some of the raw notes have credentials visible in code blocks (especially the fourth-day notes with Homepage services.yaml)

Nothing needs to happen tonight. This is a workflow for future sessions.

---

## Things worth reading further

- **Pro Git** — the free canonical Git book — [git-scm.com/book/en/v2](https://git-scm.com/book/en/v2). Chapters 1-3 cover everything used in this session and more. Worth reading straight through once; you'll recognise everything you did tonight and understand why each step existed.
- **GitHub Docs — SSH setup** — [docs.github.com/en/authentication/connecting-to-github-with-ssh](https://docs.github.com/en/authentication/connecting-to-github-with-ssh). Reference for the SSH key setup we walked through. Worth bookmarking.
- **GitHub Docs — noreply email verification** — [docs.github.com/en/account-and-profile/setting-up-and-managing-your-personal-account-on-github/managing-email-preferences/setting-your-commit-email-address](https://docs.github.com/en/account-and-profile/setting-up-and-managing-your-personal-account-on-github/managing-email-preferences/setting-your-commit-email-address). Full documentation on the noreply email pattern.
- **Conventional Commits** — [conventionalcommits.org](https://www.conventionalcommits.org/en/v1.0.0/). Optional but worth knowing about as a commit message convention. Not required for personal repos.
- **README best practices** — [github.com/othneildrew/Best-README-Template](https://github.com/othneildrew/Best-README-Template). Template that goes further than needed for a personal portfolio repo but useful as a reference for what's *possible* in a README.

---

## Concepts that came up in passing worth knowing about

- **Version control fundamentals** — snapshot-based history, branching, merging, distributed vs centralised models. Git is snapshot-based (each commit is a full snapshot of tracked files, deduplicated via content-addressed storage) and distributed (every clone is a full repo with full history). Understanding this shapes how Git commands feel — everything is a local operation until you push.
- **SHA-1 hashing for commit identifiers** — each Git commit is identified by a 40-character hexadecimal hash of its contents (commit message, author, parent commit, tree of files). Two commits with identical contents produce identical hashes; any difference produces different hashes. This gives Git its integrity properties — you cannot silently modify history without breaking the hash chain.
- **The three areas of Git** — working directory, staging area (also called "index"), and repository. Commands move content between these areas. `git status` shows the current state across all three.
- **HEAD, branches, and refs** — HEAD is a pointer to your current position (usually pointing at a branch, which is itself a pointer to a commit). Branches are cheap in Git — just a file containing a commit hash. Understanding this demystifies most branch-related error messages.
- **Remotes** — named references to other Git repositories. `origin` is the conventional name for "the main one you push to." A repo can have multiple remotes for different purposes (forks, mirrors, backup destinations).
- **SSH host key verification** — SSH's defence against man-in-the-middle attacks. First connection prompts you to verify the server's key fingerprint; verified keys are stored in `~/.ssh/known_hosts`. Every subsequent connection checks the presented key against the stored one, with loud warnings on mismatch.
- **The `git` user in GitHub SSH URLs** — every GitHub SSH URL uses the username `git`, regardless of which user account is actually authenticating. GitHub uses the SSH key you present to identify the human account. This design lets one server serve all users without per-user Unix accounts.
- **Public vs private repos on GitHub free tier** — both are available on the free plan. Public repos are visible to everyone; private repos are visible only to explicitly invited collaborators. For portfolio work, public is the point.
- **`.gitignore` files** — text file listing patterns that Git should not track. Common uses: build artifacts, dependency directories (`node_modules/`), environment files with secrets (`.env`), OS-specific junk (`.DS_Store` on macOS). Not needed for a pure-documentation repo but worth knowing about for future code projects.
- **Markdown as GitHub's rendering format** — GitHub renders `.md` files with a specific flavour of Markdown that includes standard formatting (headings, lists, links, code blocks) plus GitHub-specific extensions (task lists, tables, syntax highlighting for code blocks). `README.md` is rendered automatically when someone visits a repository or opens a folder.

---

*Written from the LemonLab session of 2 September 2026 (evening). Companion to the fifth day study notes covering the earlier session same day. Bookmark alongside the previous notes for reference.*
