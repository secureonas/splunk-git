# Git Version Control for Splunk — Search Head Setup

Installing Splunkbase app **4182** (*Git Version Control for Splunk*, Chris Younger)
on a search head, with the git metadata stored **outside** the Splunk install path
and pushing to Gitea at `192.168.203.139`.

- **App version:** 1.4.5 (13 Aug 2026) — Apache 2.0, developer supported
- **Compatible:** Splunk 10.5 → 9.0 (covers your 10.2.7 target)
- **Does not work on Splunk Cloud**

---

## Terminology — read this first

The app uses two separate paths and they are easy to confuse:

| Setting | What it is | Value here |
|---|---|---|
| `working_directory` | The tree being **tracked**. Must stay where Splunk's config actually lives. | `/opt/splunk/etc/` |
| `repository_directory` | The `.git` **metadata** — objects, refs, history. This is what moves out. | `/opt/splunk_git_repo/etc.git` |

So `/opt/splunk/etc` is still the working directory — you're relocating the *repository*,
not the tree. Reason: Splunk upgrades can't touch it, and repo growth doesn't sit inside
`$SPLUNK_HOME`.

Values below assume search head 01. For SH02, substitute `sh02` throughout and use a
**separate keypair and separate Gitea repo**.

---

## Distro differences at a glance

Steps are identical on Oracle Linux / RHEL-family and Ubuntu except where marked:

| | Oracle Linux / RHEL / Rocky | Ubuntu / Debian |
|---|---|---|
| Install git | `dnf install git` | `apt install git` |
| MAC layer | SELinux — `restorecon` after creating dirs, `ausearch` to diagnose | AppArmor — no Splunk or git profile, nothing to do |
| Splunk package | `.rpm` | `.deb` |
| Splunk service | `/etc/init.d/splunk` or `Splunkd.service` | same — depends on how Splunk was enabled, not the distro |

Only the git install and the SELinux lines actually change. Paths, the app config, the
key handling and everything in sections 3–9 are identical.

---

## 1. Confirm prerequisites

**Oracle Linux / RHEL family:**

```bash
sudo dnf install -y git
```

**Ubuntu / Debian:**

```bash
sudo apt update && sudo apt install -y git
```

Both:

```bash
# git must exist and be reachable
git --version

# what home does the splunk account actually have?
getent passwd splunk
```

Note the home directory but don't depend on it — see step 4.

> `git` must be on the `PATH` that **splunkd** sees, not just your login shell. Both
> distros put it in `/usr/bin`, which is normally fine, but see the troubleshooting
> table if the input reports `git: command not found`.

---

## 2. Install the app

Splunk Web → **Apps** → **Find More Apps** → search "Git Version Control for Splunk"
→ Install.

Or by CLI:

```bash
sudo -u splunk /opt/splunk/bin/splunk install app /path/to/git_for_splunk.spl
sudo -u splunk /opt/splunk/bin/splunk restart
```

Confirm the input spec before writing config — settings occasionally change between
versions:

```bash
cat /opt/splunk/etc/apps/git_for_splunk/README/inputs.conf.spec
```

---

## 3. Create the external repository

```bash
# Directory that will hold the git metadata and the SSH material
sudo mkdir -p /opt/splunk_git_repo
sudo chown splunk:splunk /opt/splunk_git_repo
sudo chmod 750 /opt/splunk_git_repo

# Initialise the repo with its work tree pointing back at Splunk's etc
sudo -u splunk git --git-dir=/opt/splunk_git_repo/etc.git \
                   --work-tree=/opt/splunk/etc init -b main

# Make the work tree permanent in the repo config, so it works without flags
sudo -u splunk git --git-dir=/opt/splunk_git_repo/etc.git config core.bare false
sudo -u splunk git --git-dir=/opt/splunk_git_repo/etc.git config core.worktree /opt/splunk/etc

# Commit identity — set per-repo so SH01 and SH02 stay distinguishable
sudo -u splunk git --git-dir=/opt/splunk_git_repo/etc.git config user.name  "splunk-sh01"
sudo -u splunk git --git-dir=/opt/splunk_git_repo/etc.git config user.email "splunk-sh01@example.com"
```

There is deliberately **no `.git` file or directory left in `/opt/splunk/etc`**. The app
is given both paths explicitly, so nothing needs to point back.

Define a shorthand for the rest of this guide:

```bash
alias sgit='sudo -u splunk git --git-dir=/opt/splunk_git_repo/etc.git'
```

---

## 4. SSH key — absolute paths, no `$HOME`

The modular input runs under `splunkd` and inherits whatever environment splunkd was
started with. `$HOME` may be `/root`, or unset. **Never rely on `~/.ssh` here** — it
works when you test with `sudo -u splunk -H` and then fails silently from the app.

```bash
sudo mkdir -p /opt/splunk_git_repo/.ssh
sudo chown splunk:splunk /opt/splunk_git_repo/.ssh
sudo chmod 700 /opt/splunk_git_repo/.ssh

# Generate as splunk, in place
sudo -u splunk ssh-keygen -t ed25519 -N '' \
  -f /opt/splunk_git_repo/.ssh/gitea_sh01 \
  -C "splunk-sh01@example"

# Host key, written as splunk so ownership is correct
sudo -u splunk bash -c \
  'ssh-keyscan -H 192.168.203.139 > /opt/splunk_git_repo/.ssh/known_hosts'
```

Pin the SSH command into the repo config:

```bash
sgit config core.sshCommand \
  "ssh -i /opt/splunk_git_repo/.ssh/gitea_sh01 -o IdentitiesOnly=yes -o UserKnownHostsFile=/opt/splunk_git_repo/.ssh/known_hosts -o StrictHostKeyChecking=yes"
```

**Oracle Linux / RHEL family** — relabel the new directory:

```bash
sudo restorecon -Rv /opt/splunk_git_repo
```

**Ubuntu** — nothing to do. AppArmor ships no profile for Splunk or git, so nothing
confines reads from `/opt/splunk_git_repo`. Skip this step entirely.

---

## 5. Add the deploy key in Gitea

```bash
sudo cat /opt/splunk_git_repo/.ssh/gitea_sh01.pub
```

In Gitea: repo → **Settings** → **Deploy Keys** → **Add Deploy Key**

1. Title: `splunk-sh01`
2. Paste the public key
3. **Tick "Enable Write Access"**

Deploy keys are read-only by default and there is no toggle to change an existing one —
it must be deleted and re-added. This is the most common failure in this setup.

---

## 6. `.gitignore` — before the first `git add`

`$SPLUNK_HOME/etc` has secrets scattered through it. Committing `auth/splunk.secret`
means every encrypted password on that instance is decryptable by anyone with repo read
access, and removing it later means rewriting history.

Create `/opt/splunk/etc/.gitignore`:

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

```bash
sudo chown splunk:splunk /opt/splunk/etc/.gitignore
```

Two notes:

- **Do not ignore the btool output directory.** If you enable `include_btool_output`,
  ignoring its output defeats the entire point — the normalised dumps are the reason the
  app is worth running. Confirm where it writes and leave that path tracked.
- `*/lookups/` is a guess at bulk data. Some lookups are genuinely config and worth
  tracking. Review against the real tree before committing.

If unsure what to ignore, the app author's own advice: run it with no `.gitignore` for a
week, look at the supplied dashboard to see which files churn most, then write rules and
recreate the repo.

---

## 7. First commit and push — manually, before enabling the input

Do this by hand. If it fails, you get a clear error instead of a silent modular input
failure buried in `_internal`.

```bash
sgit add -A

# Verify nothing sensitive is staged — this must return nothing
sgit status --short | grep -Ei 'secret|passwd|\.pem|\.key|auth/'

sgit commit -m "Initial SH01 config snapshot"
sgit remote add origin git@192.168.203.139:admin/searchhead1.git
sgit push -u origin main
```

Then test the way splunkd will actually run it — empty environment, no `$HOME`:

```bash
sudo -u splunk env -i \
  git --git-dir=/opt/splunk_git_repo/etc.git ls-remote origin
```

If that succeeds, the modular input will work regardless of splunkd's environment. If it
fails here but worked above, your config is depending on `$HOME` somewhere.

---

## 8. Configure the modular input

Create `/opt/splunk/etc/apps/git_for_splunk/local/inputs.conf`:

```ini
[gitforsplunk://sh01_git]
repository_directory = /opt/splunk_git_repo/etc.git
working_directory = /opt/splunk/etc/
btool_conf_files = inputs, indexes, fields, props, limits, outputs, transforms, savedsearches, macros
include_btool_output = 1
remote_push = 1
interval = 900
index = _internal
disabled = 0
```

```bash
sudo chown -R splunk:splunk /opt/splunk/etc/apps/git_for_splunk/local
sudo -u splunk /opt/splunk/bin/splunk restart
```

`interval = 900` is every 15 minutes. Raise it if the repo grows faster than you want —
config on a search head rarely changes more than a few times a day.

---

## 9. Verify

Watch the input run:

```spl
index=_internal sourcetype=gitforsplunk OR source=*git_for_splunk* earliest=-1h
```

Then confirm commits are landing:

```bash
sgit log --oneline -5
```

And check the Gitea web UI shows the same commits.

The app also ships a changes dashboard (Apps → Git Version Control for Splunk) and an
email alert action that reports which files changed — worth enabling once the basics work.

---

## 10. Repeat for SH02

Everything above, with these changed:

- Keypair: `/opt/splunk_git_repo/.ssh/gitea_sh02`
- Deploy key title in Gitea: `splunk-sh02`
- **A different Gitea repo** — a key can only be a deploy key on one repo, and two SHs
  pushing to one repo will fight over the branch
- Input stanza: `[gitforsplunk://sh02_git]`
- Commit identity: `splunk-sh02` / `splunk-sh02@example.com`

If these two are members of the same search head cluster, don't run this on both. Splunk
already replicates config between cluster members, so you'd get near-identical diffs in
two repos. Pick one member, or run it on the deployer.

---

## Troubleshooting

| Symptom | Cause |
|---|---|
| No commits, nothing obvious in the UI | Check `index=_internal` for the input. Modular input failures often don't surface in the dashboard. |
| `Permission denied (publickey)` | Key not readable by `splunk`, or `core.sshCommand` not set on the repo. |
| `not authorized to write` | Deploy key lacks write access. Delete and re-add with the box ticked. |
| Push hangs or times out | `known_hosts` missing or wrong ownership. |
| Works manually, fails from the app | `$HOME` dependency. Re-run the `env -i` test in step 7. |
| `git: command not found` in `_internal` | splunkd's `PATH` doesn't include git. Check whether the app offers a git binary path setting. |
| Push denied after a Splunk upgrade | Confirm `/opt/splunk_git_repo` ownership survived. On RHEL family also re-run `restorecon`. |
| Permission errors with no obvious cause (**Oracle Linux / RHEL**) | Check `sudo ausearch -m avc -ts recent` before assuming it's a git or Splunk problem. |
| Permission errors with no obvious cause (**Ubuntu**) | Not AppArmor — no profile applies. Check plain filesystem ownership: `ls -la /opt/splunk_git_repo /opt/splunk_git_repo/.ssh`. |

## Splunk upgrades

Because the repository lives outside `$SPLUNK_HOME`, upgrades leave it intact. After each
upgrade, confirm:

```bash
ls -ld /opt/splunk_git_repo
sudo -u splunk env -i git --git-dir=/opt/splunk_git_repo/etc.git ls-remote origin
```

> Oracle Linux / RHEL family: also `sudo restorecon -Rv /opt/splunk_git_repo` if the
> upgrade touched ownership. Ubuntu: nothing extra.

A `.deb` or `.rpm` upgrade that changes ownership under `/opt/splunk` does **not** touch
`/opt/splunk_git_repo` — that's the point of keeping it outside. But if you ran a
`chown -R splunk:splunk /opt/splunk` to fix a failed precheck, confirm the git directory
still belongs to `splunk` too, since it sits alongside rather than inside.

The app itself must be re-checked for compatibility on major version jumps — 1.4.5 covers
through 10.5, so you're clear for the 10.2.7 and 10.4.x steps.
