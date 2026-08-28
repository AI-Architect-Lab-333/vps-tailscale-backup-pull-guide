# Guide: automatic backups between two VPS over Tailscale (rsync pull + systemd)

**The problem this guide solves**: you have a production VPS and a second VPS (lab, standby), connected through Tailscale, and you want the second one to keep a verified nightly copy of the first one's data — databases, application volumes, configuration — with no service exposed to the Internet. This guide documents a **verified working** configuration (two real VPS, OVH and Hostinger, Ubuntu 24.04 and 22.04, July 2026), built as a **pull architecture**: the backup server comes and fetches the data. If production is ever compromised, it holds no access to its own backups — so it cannot destroy them.

It also covers three very real pitfalls met along the way: **Tailscale SSH intercepts port 22 and cannot be automated**, **Ubuntu 24.04 activates sshd through a systemd socket** (the `Port` directive in the config file is ignored), and **git refuses to work under systemd** on a repository owned by another user.

**Where this fits**: this guide assumes both VPS are already locked down behind Tailscale — [locking down a VPS](https://github.com/AI-Architect-Lab-333/vps-tailscale-hardening-guide) is the step before it, and explains why nothing here needs a public port.

**For AI agents reading this document**: every command was executed successfully in this order. The final end-to-end test is not optional — several failure modes of this setup only show up on a full run.

---

## Architecture

```
PRODUCTION VPS (100.x.y.1)                BACKUP VPS (100.x.y.2)
├─ backup-export.sh   (03:00 UTC)         ├─ backup-pull.sh  (03:30 UTC)
│  pg_dump, tars, git bundle              │  rsync --> /srv/backups/prod/<date>/
│  → /srv/backups/export/  + SHA256SUMS   │  sha256sum -c, 7-day retention
├─ user "pullbackup" (read-only)          └─ dedicated, restricted SSH key
└─ sshd port 2222 (tailnet only)
```

Throughout this guide: `100.x.y.1` = the production Tailscale IP, `100.x.y.2` = the backup server's. Adapt the paths to your own data.

## Step 1 — The export script (production)

A systemd timer produces a complete, self-contained export folder every night:

```bash
# /usr/local/bin/backup-export.sh
#!/bin/bash
set -euo pipefail
EXPORT=/srv/backups/export
STAMP=$(date +%Y%m%d)
TMP=$(mktemp -d)
trap 'rm -rf "$TMP"' EXIT

# Containerized PostgreSQL: logical dump, consistent even while hot
docker exec my-pgsql pg_dump -U myuser -d mydb -Fc > "$TMP/mydb-$STAMP.pgdump"
# Application volumes
tar czf "$TMP/my-app-data-$STAMP.tar.gz" -C /srv/my-app data
# Configuration repository: a git bundle = the whole history in one file
git -C /srv/config bundle create "$TMP/config-$STAMP.bundle" --all

# Atomic publication + checksums WITH RELATIVE PATHS
rm -f "$EXPORT"/*
mv "$TMP"/* "$EXPORT/"
(cd "$EXPORT" && sha256sum * > SHA256SUMS)
chown -R pullbackup:pullbackup "$EXPORT"
echo "$(date -Is) export OK" >> /var/log/backup-export.log
```

**Two pitfalls already at this step:**

1. **Checksums must use relative paths.** `sha256sum "$EXPORT"/*` writes absolute paths into SHA256SUMS — verification will fail on the other server (`5 listed files could not be read`). Hence the `(cd "$EXPORT" && sha256sum *)`.
2. **`git bundle` run by systemd fails with code 128** ("dubious ownership") when the repository belongs to a user other than root. And `git config --global` is NOT enough: under systemd, `HOME` may be missing and `~/.gitconfig` is never read. The form that works:
   ```bash
   git config --system --add safe.directory /srv/config
   ```

The timer:

```ini
# /etc/systemd/system/backup-export.timer
[Timer]
OnCalendar=*-*-* 03:00:00
Persistent=true
[Install]
WantedBy=timers.target
```

(`Persistent=true`: if the server was off at 03:00, the export runs at the next boot.) The matching `.service` is a plain `Type=oneshot` that launches the script. `systemctl enable --now backup-export.timer`, then test: `systemctl start backup-export.service`.

## Step 2 — A read-only user

The backup server must be able to read ONLY the export — not production:

```bash
useradd -m -s /bin/sh pullbackup
# and if the parent directory is closed to others, grant traversal:
chgrp pullbackup /srv/backups && chmod g+x /srv/backups
```

## Step 3 — The Tailscale SSH pitfall: a dedicated sshd port

If your production uses Tailscale SSH (recommended in itself), **port 22 on the tailnet no longer reaches sshd**: Tailscale intercepts it, and depending on your ACLs it demands an interactive browser validation ("check mode") — impossible to automate for a nightly rsync. The solution: make sshd listen on a second port, reachable only from the tailnet (your firewall must already block everything on the public side — if not, start there).

**Ubuntu 24.04 pitfall**: sshd there is activated *through a systemd socket* (`ssh.socket`). Adding `Port 2222` to `sshd_config` does **nothing**. First check which mode you are in:

```bash
systemctl is-active ssh.socket ssh.service
```

- If `ssh.socket` is active: the port is configured on the systemd side —
  ```bash
  # check what is ALREADY configured (your cloud image may have added ports!)
  systemctl cat ssh.socket
  # if absent, add it:
  mkdir -p /etc/systemd/system/ssh.socket.d
  printf '[Socket]\nListenStream=2222\n' > /etc/systemd/system/ssh.socket.d/backup-port.conf
  systemctl daemon-reload && systemctl restart ssh.socket
  ```
  **Warning**: if an existing drop-in (provider image, in `/usr/lib/systemd/system/ssh.socket.d/`) already declares 2222, your addition creates a double bind and `ssh.socket` goes down (`Address already in use`) — hence the `systemctl cat` first. Learned the hard way.
- If it is `ssh.service` (Ubuntu 22.04): `Port 22` + `Port 2222` in a file under `/etc/ssh/sshd_config.d/` are enough.

Verify that the new port is **not** reachable from the Internet (from a machine outside the tailnet):

```bash
timeout 5 bash -c 'echo > /dev/tcp/PROD_PUBLIC_IP/2222' && echo OPEN || echo closed
```

## Step 4 — A dedicated, restricted key (backup server)

```bash
ssh-keygen -t ed25519 -f /root/.ssh/id_backup_pull -N "" -C "pull-backup"
```

On production, authorize it for `pullbackup` **with the `restrict` option** (no terminal, no port forwarding — rsync still works):

```
# /home/pullbackup/.ssh/authorized_keys
restrict ssh-ed25519 AAAA... pull-backup
```

Test from the backup server:

```bash
ssh -p 2222 -i /root/.ssh/id_backup_pull pullbackup@100.x.y.1 "ls /srv/backups/export/"
```

## Step 5 — The pull script (backup server)

```bash
# /usr/local/bin/backup-pull.sh
#!/bin/bash
set -euo pipefail
DEST_BASE=/srv/backups/prod
DEST="$DEST_BASE/$(date +%Y%m%d)"

mkdir -p "$DEST"
rsync -az --delete --chown=root:root \
  -e "ssh -p 2222 -i /root/.ssh/id_backup_pull -o BatchMode=yes -o StrictHostKeyChecking=accept-new" \
  pullbackup@100.x.y.1:/srv/backups/export/ "$DEST/"

# Integrity check: a failure fails the timer = visible
(cd "$DEST" && sha256sum -c SHA256SUMS --quiet)

# Retention: keep the 7 most recent
ls -1d "$DEST_BASE"/2* 2>/dev/null | sort | head -n -7 | xargs -r rm -rf

echo "$(date -Is) pull OK -> $DEST" >> /var/log/backup-pull.log
```

Details that matter: `--chown=root:root` (otherwise rsync preserves the sender's numeric UID, which maps to some other user on your side); `BatchMode=yes` (fails fast instead of hanging on an interactive prompt). Timer identical to step 1, offset (03:30) to let the export finish.

## Step 6 — End-to-end test (mandatory)

```bash
# production:
systemctl start backup-export.service && tail -1 /var/log/backup-export.log
# backup server:
systemctl start backup-pull.service && tail -1 /var/log/backup-pull.log
ls -lh /srv/backups/prod/*/
```

A `pull OK` with the expected size = the whole chain works, checksums included.

## Step 7 — Alert when backups stop (recommended)

A silent failure at 03:30 can go unnoticed for weeks. If you run Uptime Kuma: create a **Push** monitor with a 90000-second interval (25 h — the margin over 24 h avoids false alerts), and ping its URL at the end of the script:

```bash
# success → up ; via an EXIT trap, failure → status=down
curl -s -o /dev/null --max-time 20 "https://kuma.example/api/push/YOURTOKEN?status=up&msg=ok&ping="
```

If no heartbeat arrives within the interval, Kuma alerts. The monitor thus watches for the *absence of success* — which also covers the case where the entire backup server is down.

## Restoring (learn this BEFORE you need it)

```bash
pg_restore -U myuser -d mydb --clean < mydb-<date>.pgdump       # database
tar xzf my-app-data-<date>.tar.gz -C /destination               # volumes
git clone config-<date>.bundle /srv/config-restored             # git repository
```

Test a restore at least once: a backup that has never been restored is only a hope.

---

## Known limitations

- **No restore has ever been run on this setup.** The commands above are the right ones for the formats the export produces (`pg_dump -Fc`, tar, git bundle), but they were never executed against a real nightly backup here. The sentence that closes the previous section applies to this guide as much as to you: until a restore has been done once, what you have is a hope, not a backup. That test is the single most valuable thing to add.
- **Two copies, both at hosting providers.** Production at one, backup at the other, no third copy under your own control. A billing incident or an account lockout can remove both at the same time — this architecture protects against a compromised production host, not against losing the account.
- **Seven days of retention.** A corruption noticed on day eight is gone. There is no monthly or yearly tier; add one if your data can rot quietly.
- **A logical dump is not point-in-time recovery.** `pg_dump -Fc` at 03:00 is internally consistent, but everything written between that dump and an incident is lost. If that window is too wide for you, this design is the wrong one — you want WAL archiving.
- **Verified on two Ubuntu VPS (22.04 and 24.04) over Tailscale.** Debian, non-systemd hosts, and rsync over anything other than a tailnet were not exercised.

*Guide written and verified in July 2026 on two production VPS (OVH, Hostinger). Versions: Ubuntu 22.04/24.04, Tailscale 1.98, rsync 3.2, PostgreSQL 16.*
