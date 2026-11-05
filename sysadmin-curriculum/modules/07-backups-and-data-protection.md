# Module 07: Backups and data protection

**Time:** about 4 hours.
**Prerequisites:** modules 01, 02, and 03. A second VM or second disk is useful for the stretch goal.

## By the end you can

- Explain 3-2-1, RPO, RTO, and the difference between full, incremental, and differential backups.
- Back up, schedule, and **restore** with a modern tool (restic), and verify the result.
- Say what happens to ownership and permissions on restore.
- Explain why a backup stored on the machine it protects isn't a backup.
- Tell how you'd know that last night's backup silently failed.

## Why this matters

*A backup you haven't restored is not a backup.* Most backup failures are discovered during the restore, on the worst day. The skill isn't running a backup command; it's designing something you can trust and proving it works.

## Concepts

- **3-2-1 rule:** 3 copies of the data, on 2 different kinds of media, with 1 offsite (or offline or immutable).
- **RPO (Recovery Point Objective):** how much data loss is acceptable. It sets backup frequency. "RPO of 24 hours" means a daily backup is enough.
- **RTO (Recovery Time Objective):** how long a restore may take. It sets the method and tooling. A restore that takes 3 days doesn't meet a 4-hour RTO.
- **Full, incremental, differential.** Full copies everything. Incremental copies what changed since the last backup of any kind. Differential copies what changed since the last full. Modern tools deduplicate, so each snapshot behaves like a full backup but stores only new data.
- **Backup is not replication or RAID.** Replication copies deletions and corruption instantly. Backups give you history.
- **Ransomware changed this.** Backups must be immutable or offline, with separate credentials, so an attacker who owns your server can't delete them.
- **What to back up:** data and **configuration**, not the whole OS when you can rebuild it. Know what you'd need to rebuild a server from scratch.
- **Restore testing:** regular, by someone other than the person who set up the job, and including permissions and ownership.
- **Monitoring the backup:** alert on failures *and* on silence. No news isn't good news.

## Lab

Snapshot first. restic is in EPEL on Rocky 9. (If it isn't available, download the static binary from restic's GitHub releases.)

### 1. Install and initialise

```bash
sudo dnf install -y epel-release
sudo dnf install -y restic

sudo -i                                          # backing up /etc needs root
echo 'lab-only-password' > /root/.restic-pass && chmod 600 /root/.restic-pass
export RESTIC_REPOSITORY=/srv/backup/repo
export RESTIC_PASSWORD_FILE=/root/.restic-pass

mkdir -p /srv/backup/repo /root/important-test-data
echo "customer list" > /root/important-test-data/customers.txt
chown nobody:nobody /root/important-test-data/customers.txt
chmod 640 /root/important-test-data/customers.txt

restic init
restic backup /etc /root/important-test-data
restic snapshots
restic check
```

### 2. The important part: restore

```bash
ls -l /root/important-test-data                   # note owner and mode
rm -rf /root/important-test-data                  # simulate the disaster
restic restore latest --target /tmp/restore --include /root/important-test-data
ls -l /tmp/restore/root/important-test-data/       # same owner and mode?
cat /tmp/restore/root/important-test-data/customers.txt
```

Time the restore. Write the time down.

### 3. Change and restore a file version

```bash
echo "more data" >> /tmp/restore/root/important-test-data/customers.txt
restic backup /tmp/restore
restic snapshots
restic diff <older-id> <newer-id>
restic restore <older-id> --target /tmp/restore2
```

### 4. Schedule it

Use a systemd timer (module 01) or cron. A minimal pair:

```ini
# /etc/systemd/system/lab-backup.service
[Unit]
Description=Lab backup

[Service]
Type=oneshot
Environment=RESTIC_REPOSITORY=/srv/backup/repo
Environment=RESTIC_PASSWORD_FILE=/root/.restic-pass
ExecStart=/usr/bin/restic backup /etc /root/important-test-data

# /etc/systemd/system/lab-backup.timer
[Unit]
Description=Daily lab backup

[Timer]
OnCalendar=daily
Persistent=true

[Install]
WantedBy=timers.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now lab-backup.timer
systemctl list-timers | grep lab-backup
sudo systemctl start lab-backup.service           # run it now to test
journalctl -u lab-backup.service -e
```

A plain `rsync -aH /source/ /dest/` is a fine alternative for simple cases. Warn yourself: `--delete` removes files at the destination, so always try `--dry-run` first.

### 5. Stretch: a repository that isn't on the same machine

A repository on the same disk protects against deletion but not against disk loss or a compromised server. Put it elsewhere: a second VM over SFTP (`restic -r sftp:user@host:/srv/backup/repo init`), a second disk, or object storage. That's the "2" and the "1" in 3-2-1.

### 6. Prune and retention

```bash
restic forget --keep-daily 7 --keep-weekly 4 --keep-monthly 6 --dry-run
```

What does "retention" mean for your RPO, for storage cost, and for how long you can recover from slow corruption you only notice later?

## Break it

1. **Corrupt the repository** (in a scratch copy) by overwriting a file under `data/`, then run `restic check`. See what a damaged backup looks like and what `check` tells you.
2. **Silent failure.** Change the password file so the scheduled job fails. How long until anyone notices? Add something that would tell you (monitoring in module 08, or a dead-man's-switch such as healthchecks.io).
3. **The wrong restore target.** Restore directly over a live directory (in the lab) and discuss why a restore to a side location first is safer.

## Check your understanding

- What are RPO and RTO, and which one decides how often you back up?
- How would you know if last night's backup silently failed?
- Where is your backup stored, and what happens if the server itself is compromised?
- Who has access to delete the backups?
- How often should restores be tested, and who should do it?
- What's the difference between a backup and replication or RAID?
- Does a restored file have the same owner and mode as the original? How do you check?
- Why do ransomware attackers go after backups first?

## Deliverable

A **backup and restore runbook** for the lab VM: what is backed up, where, how often, how long it's kept, the exact restore commands, the measured restore time, and how failures are detected. Include your RPO and RTO and whether you actually met them.

## Use AI well here

Ask Claude to review your backup job for failure modes (no alerting? no restore test? password handling? repository on the same disk?). Decide which points are valid and explain why. Then ask it to quiz you on RPO vs RTO with realistic examples.

## Go deeper

- restic docs (restic.readthedocs.io), BorgBackup docs
- The 3-2-1 and 3-2-1-1-0 rule write-ups
- Database backups (consistent snapshots, point-in-time recovery), and VM-level backup products
- Immutable storage and object lock
