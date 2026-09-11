# Gitea Setup — Oracle Linux & Ubuntu (Internal Git)

Self-hosted Gitea (free open-source community edition), used as version control for
Splunk search head configuration.

- **Host:** `192.168.203.139`
- **Web:** `http://192.168.203.139:3000`
- **Site title:** Internal Git
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

## Client setup — separate document

Everything past this point on the *client* side — creating repositories, generating
per-search-head keys, deploy keys, `.gitignore`, the first push, and the
`git_for_splunk` app — is covered in **[SplunkGitClient.md](SplunkGitClient.md)**.

Three server-side facts worth knowing before you go there:

- **Deploy keys are read-only by default.** There is no toggle to grant write to an
  existing key — it must be deleted and re-added with *Enable Write Access* ticked.
  This is the single most common failure when onboarding a client.
- **The SSH user is always `git`.** Clone URLs are `git@<host>:owner/repo.git`
  regardless of which Gitea account owns the repo. Identity comes from the key.
- **Set the default branch before creating repos.** Site Administration → Repository →
  `DEFAULT_BRANCH`. A mismatch between this and what clients push shows up as an
  apparently empty repository in the web UI.

Create repositories **empty** — no README, no `.gitignore` from the UI. Clients are
pushing existing content, and an initial commit produces an unrelated-histories merge.

## Troubleshooting

Server-side issues only. For push, key and client errors see
[SplunkGitClient.md](SplunkGitClient.md).

| Error | Cause |
|---|---|
| `database is locked` | Raise `SQLITE_TIMEOUT`, confirm `SQLITE_JOURNAL_MODE = WAL`. Persistent occurrences mean it's time for PostgreSQL. |
| Connection refused on 3000 (Oracle Linux) | `firewall-cmd --list-ports` — the rule wasn't reloaded. |
| Connection refused on 3000 (Ubuntu) | `ufw status` — rule missing, or ufw was enabled after the rule was added. |
| Permission errors that make no sense (Oracle Linux) | Check `sudo ausearch -m avc -ts recent` before assuming it's a Gitea problem. |
| Permission errors that make no sense (Ubuntu) | Not AppArmor — no Gitea profile exists. Check ownership under `/var/lib/gitea` and `/etc/gitea`. |
| Service won't start after an upgrade | `journalctl -u gitea -n 50`. Database migrations run on first start of a new version and are one-way — restore from the pre-upgrade dump. |
| `PTY allocation request failed on channel 0` when testing SSH | Not an error. Expected — Gitea refuses shell access by design. |
| Repo shows empty in the web UI despite a client reporting a successful push | Default branch mismatch. Repo Settings → Branches → set default. |

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

## Enable HTTPS

Self-signed certificate on port 3000. Chosen because the web UI is used locally to
create repositories and review changes, while all Splunk traffic goes over SSH and
never touches TLS.

Staying on 3000 rather than 443 avoids the privileged-port capability entirely — one
less thing to re-apply after a binary upgrade.

### 1. Generate the certificate

`gitea cert` sets `subjectAltName` correctly, which matters for an IP address. A plain
`openssl req` without a SAN produces a certificate modern browsers reject outright
rather than warn about.

```bash
cd /etc/gitea
sudo -u git /usr/local/bin/gitea cert \
  --host 192.168.203.139,localhost \
  --duration 26280h

sudo chown git:git /etc/gitea/cert.pem /etc/gitea/key.pem
sudo chmod 644 /etc/gitea/cert.pem
sudo chmod 600 /etc/gitea/key.pem
```

> Oracle Linux only: `sudo restorecon -v /etc/gitea/cert.pem /etc/gitea/key.pem`
> Ubuntu: nothing required.

**List every name and IP you might ever use, now.** Adding one later means
regenerating and re-accepting the browser warning everywhere. If `git.example.com` is
plausible, include it today — it costs nothing:

```bash
--host 192.168.203.139,git.example.com,localhost
```

`26280h` is three years. For an internal certificate there's no reason to churn
annually.

### 2. app.ini

```ini
[server]
PROTOCOL  = https
DOMAIN    = 192.168.203.139
HTTP_PORT = 3000
ROOT_URL  = https://192.168.203.139:3000/
CERT_FILE = /etc/gitea/cert.pem
KEY_FILE  = /etc/gitea/key.pem
SSH_PORT  = 22

[repository]
DISABLE_HTTP_GIT = true
```

`DISABLE_HTTP_GIT = true` closes the git-over-HTTP path. Safe here because the search
heads clone and push over SSH — remove the block if you want to keep HTTP cloning
available.

```bash
sudo systemctl restart gitea
sudo journalctl -u gitea -n 30
```

### 3. Verify

```bash
openssl s_client -connect 192.168.203.139:3000 -servername 192.168.203.139 </dev/null 2>/dev/null \
  | openssl x509 -noout -subject -dates -ext subjectAltName
```

Confirm the IP appears under `subjectAltName` and the dates are as expected.

Then browse to `https://192.168.203.139:3000/` and accept the warning once per browser.

### Not trusting the certificate on clients

Deliberate here — the browser warning is a one-time click-through and no automated
client uses HTTPS.

Be aware of the consequence: **git fails hard on an untrusted certificate, with no
prompt.** If anyone ever clones over HTTPS:

```
fatal: unable to access '...': SSL certificate problem: self-signed certificate
```

Per-repo workaround for whoever clones:

```bash
git -c http.sslVerify=false clone https://192.168.203.139:3000/admin/searchhead1.git
cd searchhead1 && git config http.sslVerify false
```

Scoped to that repo only. **Never set `http.sslVerify false` globally** — it disables
certificate verification for every HTTPS remote on that machine, github.com included.

`curl` needs `-k`; Python `requests` needs `verify=False`.

If HTTPS clients ever become common, trusting the certificate is three commands and
less trouble than scattering `sslVerify false` around:

```bash
# RHEL family
sudo cp /etc/gitea/cert.pem /etc/pki/ca-trust/source/anchors/gitea-internal.crt
sudo update-ca-trust

# Ubuntu
sudo cp /etc/gitea/cert.pem /usr/local/share/ca-certificates/gitea-internal.crt
sudo update-ca-certificates
```

Firefox keeps its own certificate store and needs a separate import under Settings →
Privacy & Security → Certificates. Chrome and Edge follow the OS store.

If you have an internal CA, issuing from it is the better long-term move — one CA
distributed once covers every future internal service, instead of repeating this per
box.

### What does not change

- **SSH pushes are unaffected.** They authenticate on the host key fingerprint, not
  TLS. Nothing in the Splunk search head setup changes.
- **Plain HTTP stops working.** `http://192.168.203.139:3000` now returns a protocol
  error rather than redirecting. If that will confuse people, `REDIRECT_OTHER_PORT`
  and `PORT_TO_REDIRECT` under `[server]` can catch plain HTTP on a second port and
  bounce it to HTTPS.
- **`ROOT_URL` changed.** Any existing HTTP clone URL stops matching. SSH remotes are
  unaffected — SSH transport does not consult `ROOT_URL`.

### Alternative considered: loopback only

If the UI were needed rarely, `HTTP_ADDR = 127.0.0.1` plus an SSH tunnel
(`ssh -L 3000:127.0.0.1:3000 user@192.168.203.139`) would remove the web port from the
network entirely and need no certificate at all. Rejected here because the UI is used
regularly for repository creation and change review, and needing a tunnel each time is
friction that outweighs the gain.
