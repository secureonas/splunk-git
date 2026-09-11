# Gitea Setup — RedHat Linux and Ubuntu (Internal Git)

Self-hosted Gitea (free open-source community edition) on Oracle Linux 8/9, used as
version control for Splunk search head configuration.

- **Host:** `192.168.203.139`
- **Web:** `http://192.168.203.139:3000`
- **Site title:** Internal Git
- **Database:** SQLite3
- **OS service account:** `git`

> The free edition is the MIT-licensed open-source Gitea from `dl.gitea.com`. No
> license key, no trial, no signup. `about.gitea.com` is CommitGo's commercial site
> for Gitea Enterprise — not needed here.

---

## 1. Prerequisites

```bash
sudo dnf install -y git git-lfs
git --version   # needs >= 2.7; OL8/OL9 stock packages are fine
```

## 2. Create the service account

```bash
sudo useradd \
  --system --shell /bin/bash \
  --comment 'Gitea Git Service' \
  --create-home --home-dir /home/git \
  git
```

The shell must be a real shell — Gitea uses this account for SSH transport and
substitutes its own restricted command handler.

## 3. Install the binary

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

## 4. Directory layout

```bash
sudo mkdir -p /var/lib/gitea/{custom,data,log}
sudo chown -R git:git /var/lib/gitea/
sudo chmod -R 750 /var/lib/gitea/

sudo mkdir /etc/gitea
sudo chown root:git /etc/gitea
sudo chmod 770 /etc/gitea    # tightened after the web installer writes app.ini
```

## 5. systemd unit

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

## 6. SELinux and firewall

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

## 7. Web installer

Browse to `http://192.168.203.139:3000`.

| Field | Value |
|---|---|
| Database Type | SQLite3 |
| Path | `/var/lib/gitea/data/gitea.db` |
| Site Title | Internal Git |
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
hand. If DNS is available, use a hostname (`git.example.com`) instead of an IP.

## 8. Post-install hardening

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

### Generate a key per search head

Run as the account that will execute the commit job. If that's a cron under
`splunk`, generate it as `splunk` — not root, or the key lands in `/root/.ssh`
where the job can't read it.

```bash
sudo -u splunk -H ssh-keygen -t ed25519 -N '' \
  -f ~splunk/.ssh/gitea_sh01 \
  -C "splunk-sh01@example"
```

Pre-seed the host key, or the first non-interactive push will hang:

```bash
sudo -u splunk -H bash -c 'ssh-keyscan -H 192.168.203.139 >> ~/.ssh/known_hosts'
```

Pin the identity:

```bash
sudo -u splunk -H tee -a ~splunk/.ssh/config > /dev/null <<'EOF'
Host 192.168.203.139
    User git
    IdentityFile ~/.ssh/gitea_sh01
    IdentitiesOnly yes
EOF
sudo -u splunk chmod 600 ~splunk/.ssh/config
```

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
repo, so a compromised search head can only reach its own history. Note that a given
key can be a deploy key on only one repo — hence one keypair per search head.

### The SSH user is always `git`

Clone URLs are `git@192.168.203.139:owner/repo.git` regardless of which Gitea
account owns the repo. Identity comes from the key, not the username.

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

Set **per-repo**, not globally, so SH01 and SH02 stay distinguishable:

```bash
cd /opt/splunk/etc
sudo -u splunk git config user.name  "splunk-sh01"
sudo -u splunk git config user.email "splunk-sh01@example.com"
```

### .gitignore — write this BEFORE the first `git add`

`$SPLUNK_HOME/etc` has secrets scattered through it. Committing `auth/splunk.secret`
means every encrypted password on that instance is decryptable by anyone with repo
read access. Removing it afterwards requires rewriting history on both repos.

Create `/opt/splunk/etc/.gitignore`:

```gitignore
# --- Secrets ---
auth/splunk.secret
auth/distServerKeys/
auth/*.pem
auth/*.key
auth/audit/
passwd
*/passwd
*/local/passwd
*.pem
*.key
*.p12
*.jks

# --- Runtime state / bulk data ---
var/
*/lookups/
apps/*/lookups/
users/*/*/lookups/
instance.cfg
*/local/inputs.conf.bak
splunk-launch.conf

# --- Noise ---
*.bak
*.old
*.tmp
*.swp
*~
.DS_Store
licenses/
modules/
openldap/
```

Review this against the actual tree before committing. Adjust `*/lookups/` if any
lookups are genuinely config rather than data — some are small CSVs worth tracking,
most are bulk.

Verify nothing sensitive is staged:

```bash
cd /opt/splunk/etc
sudo -u splunk git add -A
sudo -u splunk git status --short | grep -Ei 'secret|passwd|\.pem|\.key'
```

That should return nothing.

### Initial commit

```bash
cd /opt/splunk/etc
sudo -u splunk git init -b main
sudo -u splunk git remote add origin git@192.168.203.139:admin/searchhead1.git
# .gitignore in place first — see above
sudo -u splunk git add -A
sudo -u splunk git commit -m "Initial SH01 config snapshot"
sudo -u splunk git push -u origin main
```

---

## Push contention between the two search heads

This is a git-level concern, not a database one.

| Situation | Approach |
|---|---|
| Two independent SHs (different tenants) | Separate repos. No contention. |
| Two members of the same SHC | Don't have both commit. Splunk already replicates config between cluster members — two members pushing near-identical diffs to one repo produces noisy, conflicting history. Commit from one designated member, or from the deployer. |
| Both must write to one repo | Give each a branch (`sh01`, `sh02`), or have the commit script run `git pull --rebase` before `git push` and retry once on failure. |

Pushing to the same branch from both without rebasing means the second push is
rejected as non-fast-forward.

---

## Troubleshooting

| Error | Cause |
|---|---|
| `Deploy Key: N:name is not authorized to write to owner/repo` | Write access not enabled on the deploy key. Delete and re-add with the box ticked. |
| `error: src refspec main does not match any` | Local branch is `master`. Run `git branch -m master main`. |
| `Permission denied (publickey)` | Running as a different user than the one holding the key, or the wrong identity is being offered. Diagnose with `ssh -vT git@192.168.203.139` and check which key it tries. |
| `PTY allocation request failed on channel 0` | Not an error. Expected — Gitea refuses shell access by design. |
| Push hangs on first run from cron | Host key not in `known_hosts`. Run `ssh-keyscan` as the committing user. |
| Repo shows empty in web UI despite a successful push | Default branch mismatch. Repo Settings → Branches → set default. |
| `database is locked` | Raise `SQLITE_TIMEOUT`, confirm `SQLITE_JOURNAL_MODE = WAL`. Persistent occurrences mean it's time for PostgreSQL. |

---

## Backup

`gitea dump` captures the database, repositories, config and attachments in one
archive:

```bash
sudo -u git bash -c 'cd /var/lib/gitea && \
  /usr/local/bin/gitea dump -c /etc/gitea/app.ini -f /var/lib/gitea/dump-$(date +%F).zip'
```

The dump contains `app.ini`, which holds `SECRET_KEY` and `INTERNAL_TOKEN` — treat
the archive as sensitive and store it accordingly.

For a SQLite deployment, copying `/var/lib/gitea/` and `/etc/gitea/` while the
service is stopped is also a valid full backup.

---

## Upgrades

```bash
sudo systemctl stop gitea
sudo cp /usr/local/bin/gitea /usr/local/bin/gitea.bak
sudo -u git /usr/local/bin/gitea dump -c /etc/gitea/app.ini   # backup first
GITEA_VER=<new>
sudo curl -fsSL -o /usr/local/bin/gitea \
  https://dl.gitea.com/gitea/${GITEA_VER}/gitea-${GITEA_VER}-linux-amd64
sudo chmod 755 /usr/local/bin/gitea
sudo restorecon -v /usr/local/bin/gitea
sudo systemctl start gitea
sudo journalctl -u gitea -f    # watch the migration run
```

Database migrations run automatically on first start of the new version and are
one-way. The backup is the rollback path.

If `setcap` was applied for low-port binding, re-apply it — replacing the binary
clears extended attributes:

```bash
sudo setcap 'cap_net_bind_service=+ep' /usr/local/bin/gitea
```

---

## Optional: TLS

Skip the reverse proxy unless one is already running. Gitea has built-in ACME:

```ini
[server]
PROTOCOL = https
DOMAIN = git.example.com
HTTP_PORT = 443
ENABLE_ACME = true
ACME_ACCEPT_TOS = true
ACME_EMAIL = admin@example.com
```

Requires `setcap 'cap_net_bind_service=+ep' /usr/local/bin/gitea` (re-apply after
every upgrade) and publicly reachable ports 80/443.

For internal-only, put nginx in front with an internal cert and run
`setsebool -P httpd_can_network_connect 1` so SELinux permits nginx → port 3000.
