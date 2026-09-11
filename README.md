# Gitea Setup — redhat Linux & Ubuntu (Secureon GIT)

Self-hosted Gitea (free open-source community edition), used as version control for
Splunk search head configuration.

- **Host:** `192.168.203.139`
- **Web:** `http://192.168.203.139:3000`
- **Site title:** Secureon GIT
- **Database:** SQLite3
- **OS service account:** `git`

> The free edition is the MIT-licensed open-source Gitea from `dl.gitea.com`. No
> license key, no trial, no signup. `about.gitea.com` is CommitGo's commercial site
> for Gitea Enterprise — not needed here.

## Distro differences at a glance

Steps are identical on both platforms except where marked. The summary:

| | Oracle Linux 8/9 | Ubuntu 22.04/24.04 |
|---|---|---|
| Packages | `dnf` | `apt` |
| Firewall | `firewalld` | `ufw` (often inactive by default) |
| MAC | SELinux — `restorecon`, booleans | AppArmor — no Gitea profile, nothing to do |
| Binary path, paths, systemd unit, app.ini | identical | identical |

Everything in sections 1, 6 and the nginx notes changes. Nothing else does.

---

## 1. Prerequisites

**Oracle Linux:**

```bash
sudo dnf install -y git git-lfs curl
```

**Ubuntu:**

```bash
sudo apt update
sudo apt install -y git git-lfs curl
```

Both:

```bash
git --version   # needs >= 2.7; stock packages on OL8/9 and Ubuntu 22.04/24.04 are fine
```

> On Ubuntu, `git-lfs` lives in `universe`. If `apt` can't find it:
> `sudo add-apt-repository universe && sudo apt update`

## 2. Create the service account

Identical on both:

```bash
sudo useradd \
  --system --shell /bin/bash \
  --comment 'Gitea Git Service' \
  --create-home --home-dir /home/git \
  git
```

The shell must be a real shell — Gitea uses this account for SSH transport and
substitutes its own restricted command handler.

> Ubuntu also offers `adduser --system`, but its defaults differ (no shell, different
> home handling). Use `useradd` as above on both platforms so the two builds match.

## 3. Install the binary

Identical on both:

```bash
GITEA_VER=1.27.2
sudo curl -fsSL -o /usr/local/bin/gitea \
  https://dl.gitea.com/gitea/${GITEA_VER}/gitea-${GITEA_VER}-linux-amd64
sudo chmod 755 /usr/local/bin/gitea
gitea --version
```

Check <https://dl.gitea.com/gitea/> for the current release before running this.

**Signature verification:** as of 1.27, release artifacts are signed with sigstore,
not the old GPG key. Use `cosign` if you want to verify. Older guides showing
`gpg --verify` against the Gitea signing key are out of date.

> Do **not** use the Ubuntu `gitea` apt package if one appears in a third-party repo —
> it installs to different paths and diverges from everything below. The upstream
> binary keeps both platforms identical.

## 4. Directory layout

Identical on both:

```bash
sudo mkdir -p /var/lib/gitea/{custom,data,log}
sudo chown -R git:git /var/lib/gitea/
sudo chmod -R 750 /var/lib/gitea/

sudo mkdir /etc/gitea
sudo chown root:git /etc/gitea
sudo chmod 770 /etc/gitea    # tightened after the web installer writes app.ini
```

## 5. systemd unit

Identical on both:

```bash
sudo tee /etc/systemd/system/gitea.service > /dev/null <<'EOF'
[Unit]
Description=Gitea
After=network.target

[Service]
RestartSec=2s
Type=simple
User=git
Group=git
WorkingDirectory=/var/lib/gitea/
ExecStart=/usr/local/bin/gitea web --config /etc/gitea/app.ini
Restart=always
Environment=USER=git HOME=/home/git GITEA_WORK_DIR=/var/lib/gitea

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now gitea
sudo systemctl status gitea
```

## 6. Security layer and firewall

### Oracle Linux — SELinux + firewalld

```bash
sudo restorecon -Rv /usr/local/bin/gitea /var/lib/gitea /etc/gitea
sudo firewall-cmd --permanent --add-port=3000/tcp
sudo firewall-cmd --reload
```

Custom systemd services run as `unconfined_service_t` under the default targeted
policy, so no custom SELinux module is required. `restorecon` only fixes labels on
the files created above.

If a reverse proxy is added later:

```bash
sudo setsebool -P httpd_can_network_connect 1
```

Diagnosing suspected SELinux denials:

```bash
sudo ausearch -m avc -ts recent
```

### Ubuntu — AppArmor + ufw

No AppArmor profile ships for Gitea and nothing confines a binary in `/usr/local/bin`,
so there is no labelling or boolean step. The SELinux commands above have no Ubuntu
equivalent — skip them entirely.

```bash
sudo ufw allow 3000/tcp
sudo ufw status
```

If `ufw` reports inactive, it isn't filtering anything and the port is already
reachable. Enable it only if this box should be firewalled:

```bash
sudo ufw allow OpenSSH     # do this FIRST or you lock yourself out
sudo ufw enable
```

No `httpd_can_network_connect` equivalent is needed for nginx on Ubuntu — the proxy
connection to port 3000 just works.

## 7. Web installer

Identical on both. Browse to `http://192.168.203.139:3000`.

| Field | Value |
|---|---|
| Database Type | SQLite3 |
| Path | `/var/lib/gitea/data/gitea.db` |
| Site Title | Secureon GIT |
| Repository Root Path | `/var/lib/gitea/data/gitea-repositories` |
| Git LFS Root Path | *(leave empty — disables LFS)* |
| Run As Username | `git` |
| Server Domain | `192.168.203.139` |
| SSH Server Port | `22` |
| Gitea HTTP Listen Port | `3000` |
| Gitea Base URL | `http://192.168.203.139:3000/` |
| Log Path | `/var/lib/gitea/log` |

**Before submitting**, expand the collapsed sections at the bottom of the page:

- **Administrator Account Settings** — create the admin here. If skipped, the first
  user to register becomes admin.
- **Server and Third-Party Service Settings** — tick *Disable Self-Registration* and
  *Require Sign-In to View Pages*. Setting these here avoids a window where anyone
  reaching port 3000 can register.

### Why LFS is disabled

Splunk `.conf` files, dashboards and lookups are small text. LFS adds a storage path
and a code path with no benefit. It can be enabled later per-repo if ever needed.

### Base URL warning

`DOMAIN` and `ROOT_URL` are baked into every clone URL Gitea hands out. Changing
them later means editing `app.ini` **and** fixing every remote on every client by
hand. If DNS is available, use a hostname (`git.secureon.si`) instead of an IP.

## 8. Post-install hardening

Identical on both:

```bash
sudo chmod 750 /etc/gitea
sudo chmod 640 /etc/gitea/app.ini
```

Edit `/etc/gitea/app.ini`:

```ini
[database]
DB_TYPE = sqlite3
PATH = /var/lib/gitea/data/gitea.db
SQLITE_JOURNAL_MODE = WAL
SQLITE_TIMEOUT = 20000

[service]
DISABLE_REGISTRATION = true
REQUIRE_SIGNIN_VIEW = true

[server]
DISABLE_SSH = false
SSH_PORT = 22
```

```bash
sudo systemctl restart gitea
```

**WAL mode:** Gitea's default journal mode is `DELETE`, which serializes readers
against writers. WAL lets the web UI read while a push is being recorded. Cheap
insurance; set it.

---

## Ports — what runs where

| | Port 3000 | Port 22 |
|---|---|---|
| Served by | Gitea itself | The host's `sshd` |
| Carries | Web UI, REST API, **and git over HTTP** | Git over SSH |
| Auth | Username + access token | SSH key |
| Used by | Browser, API clients | The search heads |

Gitea does not listen on 22. `sshd` authenticates the incoming key against
`/home/git/.ssh/authorized_keys`, which **Gitea writes and rewrites itself**, wrapping
every key in a forced command that invokes `gitea serv`. That's why SSH returns a
"no shell access" message instead of a prompt.

**Never hand-edit `/home/git/.ssh/authorized_keys`** — Gitea overwrites it whenever
keys change. Add keys through the UI only.

To disable the HTTP git path and leave 3000 as UI-only:

```ini
[repository]
DISABLE_HTTP_GIT = true
```

---

## Is SQLite enough?

For two search heads pushing config changes — yes, comfortably. That's tens of
commits a day, and a `git push` barely touches the database (a few rows for the push
event, webhook queue, activity feed). Git objects live on the filesystem, not in SQL.

Revisit PostgreSQL only when adding:

- Gitea Actions runners doing real CI
- The code indexer across many large repos
- A significant number of concurrent interactive users
- Repo mirroring

Migration at that point is `gitea dump` → restore, not a rewrite. No cost to
starting on SQLite.

---

## Per-search-head repository setup

### Naming

One repo per search head. Prefix them if this instance will ever hold repos for
other systems or clients — an unprefixed `searchhead1` gets ambiguous fast:

- `splunk-searchhead-01`
- `splunk-searchhead-02`

Consider creating an org (e.g. `splunk`) to own them rather than a personal account.
If the owning account is renamed or disabled, clone URLs change.

Create both repos **empty** — no README, no `.gitignore` from the UI. Pushing
existing content into a repo with an initial commit produces an
unrelated-histories merge.

### The SSH user is always `git`

Clone URLs are `git@192.168.203.139:owner/repo.git` regardless of which Gitea
account owns the repo. Identity comes from the key, not the username.

### Generate a key per search head

Store the key at an **absolute path**, not under a home directory — the Splunk
modular input runs under `splunkd` and may have `$HOME` unset or pointing at `/root`.

```bash
sudo mkdir -p /opt/splunk_git_repo/.ssh
sudo chown splunk:splunk /opt/splunk_git_repo/.ssh
sudo chmod 700 /opt/splunk_git_repo/.ssh

sudo -u splunk ssh-keygen -t ed25519 -N '' \
  -f /opt/splunk_git_repo/.ssh/gitea_sh01 \
  -C "splunk-sh01@secureon"

sudo -u splunk bash -c \
  'ssh-keyscan -H 192.168.203.139 > /opt/splunk_git_repo/.ssh/known_hosts'
```

> Oracle Linux / RHEL family only: `sudo restorecon -Rv /opt/splunk_git_repo`

### Add the deploy key in Gitea

Repo → **Settings** → **Deploy Keys** → **Add Deploy Key**

1. Title: `splunk-sh01` (not the default `root@hostname` — two machines produce
   indistinguishable entries)
2. Paste the `.pub` contents
3. **Tick "Enable Write Access"**

> Deploy keys are **read-only by default**. There is no toggle to grant write to an
> existing key — it must be deleted and re-added. This is the single most common
> failure in this setup.

**Why deploy keys and not a user SSH key:** a user key grants that key the account's
full permissions across every repo on the instance. A deploy key is scoped to one
repo, so a compromised search head can only reach its own history. A given key can be
a deploy key on only one repo — hence one keypair per search head.

### Verify

```bash
# 1. Authentication
ssh -T git@192.168.203.139
# Expect: "Hi there! You've successfully authenticated with the deploy key named ...
#          but Gitea does not provide shell access."
# "PTY allocation request failed on channel 0" is normal, not an error.

# 2. Repo path and read access
git ls-remote git@192.168.203.139:admin/searchhead1.git; echo "exit=$?"
# exit=0 with no output = repo exists and is empty.

# 3. Network path from the search head (not from the Gitea box)
nc -zv 192.168.203.139 22
```

Step 1 succeeding proves only that the key is valid — **not** that it has write
access, and not which repo it's scoped to.

---

## First real push

### Default branch

```bash
git config --global init.defaultBranch main
```

Do this on both search heads before initialising. Otherwise `git init` creates
`master`, and `git push -u origin main` fails with:

```
error: src refspec main does not match any
```

Fix on an existing repo:

```bash
git branch -m master main
```

Confirm Gitea's expectation matches: **Site Administration → Repository →
`DEFAULT_BRANCH`**. If Gitea expects `main` and the repo only has `master`, the web
UI shows an empty repo view until the default branch is corrected in repo settings.

### Commit identity

Deploy keys authenticate the push, but the commit author comes from git config.
Without it, commits are attributed to `splunk@<hostname>.(none)`.

Set **per-repo**, not globally, so SH01 and SH02 stay distinguishable.

### .gitignore — write this BEFORE the first `git add`

`$SPLUNK_HOME/etc` has secrets scattered through it. Committing `auth/splunk.secret`
means every encrypted password on that instance is decryptable by anyone with repo
read access. Removing it afterwards requires rewriting history on both repos.

```gitignore
# --- Secrets ---
auth/
passwd
*/passwd
*/local/passwd
*.secret
*.pem
*.key
*.p12
*.jks

# --- Runtime state / bulk data ---
var/
instance.cfg
splunk.version
splunk-launch.conf
*/lookups/
apps/*/lookups/
users/*/*/history/

# --- Noise ---
*.bak
*.old
*.tmp
*.pyc
*.swp
*~
.DS_Store
ui-prefs.conf
licenses/
modules/
openldap/
```

Review against the actual tree before committing — some lookups are genuinely config
rather than bulk data.

Verify nothing sensitive is staged:

```bash
git status --short | grep -Ei 'secret|passwd|\.pem|\.key|auth/'
```

That should return nothing.

---

## Push contention between the two search heads

This is a git-level concern, not a database one.

| Situation | Approach |
|---|---|
| Two independent SHs (different tenants) | Separate repos. No contention. |
| Two members of the same SHC | Don't have both commit. Splunk already replicates config between cluster members — two members pushing near-identical diffs to one repo produces noisy, conflicting history. Commit from one designated member, or from the deployer. |
| Both must write to one repo | Give each a branch (`sh01`, `sh02`), or have the commit script run `git pull --rebase` before `git push` and retry once on failure. |

---

## Troubleshooting

| Error | Cause |
|---|---|
| `Deploy Key: N:name is not authorized to write to owner/repo` | Write access not enabled on the deploy key. Delete and re-add with the box ticked. |
| `error: src refspec main does not match any` | Local branch is `master`. Run `git branch -m master main`. |
| `Permission denied (publickey)` | Running as a different user than the one holding the key, or the wrong identity is being offered. Diagnose with `ssh -vT git@192.168.203.139`. |
| `PTY allocation request failed on channel 0` | Not an error. Expected — Gitea refuses shell access by design. |
| Push hangs on first run from cron | Host key not in `known_hosts`. Run `ssh-keyscan` as the committing user. |
| Repo shows empty in web UI despite a successful push | Default branch mismatch. Repo Settings → Branches → set default. |
| `database is locked` | Raise `SQLITE_TIMEOUT`, confirm `SQLITE_JOURNAL_MODE = WAL`. Persistent occurrences mean it's time for PostgreSQL. |
| Connection refused on 3000 (Oracle Linux) | `firewall-cmd --list-ports` — the rule wasn't reloaded. |
| Connection refused on 3000 (Ubuntu) | `ufw status` — rule missing, or ufw was enabled after the rule was added. |
| Permission errors that make no sense (Oracle Linux) | Check `ausearch -m avc -ts recent` before assuming it's a Gitea problem. |

---

## Backup

Identical on both:

```bash
sudo -u git bash -c 'cd /var/lib/gitea && \
  /usr/local/bin/gitea dump -c /etc/gitea/app.ini -f /var/lib/gitea/dump-$(date +%F).zip'
```

The dump contains `app.ini`, which holds `SECRET_KEY` and `INTERNAL_TOKEN` — treat
the archive as sensitive and store it accordingly.

For a SQLite deployment, copying `/var/lib/gitea/` and `/etc/gitea/` while the
service is stopped is also a valid full backup.

## Upgrades

```bash
sudo systemctl stop gitea
sudo cp /usr/local/bin/gitea /usr/local/bin/gitea.bak
sudo -u git /usr/local/bin/gitea dump -c /etc/gitea/app.ini   # backup first
GITEA_VER=<new>
sudo curl -fsSL -o /usr/local/bin/gitea \
  https://dl.gitea.com/gitea/${GITEA_VER}/gitea-${GITEA_VER}-linux-amd64
sudo chmod 755 /usr/local/bin/gitea
sudo systemctl start gitea
sudo journalctl -u gitea -f    # watch the migration run
```

> Oracle Linux only: add `sudo restorecon -v /usr/local/bin/gitea` before starting.

Database migrations run automatically on first start of the new version and are
one-way. The backup is the rollback path.

If `setcap` was applied for low-port binding, re-apply it — replacing the binary
clears extended attributes. See the TLS section for the systemd alternative that
avoids this.

---

## Optional: Gitea on port 443

Skip the reverse proxy unless one is already running on the box.

### Grant the port capability

Use a systemd drop-in rather than `setcap`. Replacing the binary on upgrade clears
extended attributes, so `setcap` must be re-applied every time — forget once and
Gitea silently fails to start. The drop-in survives.

Identical on both platforms:

```bash
sudo mkdir -p /etc/systemd/system/gitea.service.d
sudo tee /etc/systemd/system/gitea.service.d/override.conf > /dev/null <<'EOF'
[Service]
AmbientCapabilities=CAP_NET_BIND_SERVICE
EOF
sudo systemctl daemon-reload
```

### Certificate

ACME won't work against an IP or an internal-only host — Let's Encrypt won't issue.
Use an internal CA, or self-sign:

```bash
sudo -u git /usr/local/bin/gitea cert --host 192.168.203.139 --duration 8760h
sudo mv cert.pem key.pem /etc/gitea/
sudo chown git:git /etc/gitea/cert.pem /etc/gitea/key.pem
sudo chmod 640 /etc/gitea/key.pem
```

`gitea cert` puts the IP in the SAN correctly. Self-signed means a browser warning
every time and `GIT_SSL_NO_VERIFY` workarounds on HTTP clients — prefer an internal CA.

### app.ini

```ini
[server]
PROTOCOL   = https
DOMAIN     = 192.168.203.139
HTTP_PORT  = 443
ROOT_URL   = https://192.168.203.139/
CERT_FILE  = /etc/gitea/cert.pem
KEY_FILE   = /etc/gitea/key.pem
SSH_PORT   = 22
```

### Firewall

**Oracle Linux:**

```bash
sudo firewall-cmd --permanent --add-service=https
sudo firewall-cmd --permanent --remove-port=3000/tcp
sudo firewall-cmd --reload
```

**Ubuntu:**

```bash
sudo ufw allow 443/tcp
sudo ufw delete allow 3000/tcp
sudo ufw status
```

```bash
sudo systemctl restart gitea
```

### If you use nginx instead

Set `HTTP_ADDR = 127.0.0.1` in app.ini so 3000 isn't externally reachable, proxy to
`127.0.0.1:3000`, and set `ROOT_URL` to the external HTTPS URL.

> Oracle Linux only: `sudo setsebool -P httpd_can_network_connect 1`
> Ubuntu: nothing required.

### Before changing ROOT_URL

Any clone URL handed out as `http://192.168.203.139:3000/...` stops matching. SSH
remotes are unaffected — SSH transport doesn't consult `ROOT_URL`. Do this while the
repos are still effectively empty.
