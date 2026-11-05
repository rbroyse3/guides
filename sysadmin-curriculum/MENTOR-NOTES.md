# Mentor notes

Per-module review questions, what good answers sound like, and faults to inject. Don't give this file to the learner.

All fault injections are for a snapshotted lab VM. Don't tell the learner which fault you chose. Reverse it afterward, or restore the snapshot.

## Fault pool

| Fault | How you inject it | What they should find | Modules |
|-------|-------------------|-----------------------|---------|
| Service stopped | `sudo systemctl stop httpd` | `systemctl status`, nothing listening on the port | 01, 03 |
| Config typo | Change `Listen 80` to `Lisen 80`, restart | `apachectl configtest`, `journalctl -u httpd` | 03 |
| Firewall blocks it | `sudo firewall-cmd --remove-service=http` (runtime only; `--reload` restores it) | Works on localhost, times out from outside; `firewall-cmd --list-all` | 03, 04 |
| Listening on the wrong address | Change to `Listen 127.0.0.1:80`, restart | `ss -tlnp` shows only 127.0.0.1 | 03, 04 |
| Permissions 403 | `sudo chmod 700 /srv/www` | `error_log` "(13)Permission denied", `namei -l` | 03 |
| Wrong name resolution | Add a bad line to `/etc/hosts` for a name they use | `getent hosts` and `dig` disagree | 04 |
| Wrong DNS server | `nmcli connection modify <name> ipv4.dns 192.0.2.1 && nmcli connection up <name>` | Names fail, IPs work; `cat /etc/resolv.conf`, `dig` | 04 |
| Wrong default gateway | `sudo ip route del default` (runtime; reboot or reconnect restores it) | Local network works, nothing beyond; `ip route` | 04 |
| Disk full | `sudo fallocate -l 15G /var/fill.img` (size it to fit the disk) | `df -h`, then `du -sh /var/* \| sort -h` | 01, 02, 08 |
| Inodes exhausted | See module 02, break it 1 | `df -i` | 02 |
| Deleted but open file | See module 02, break it 2 | `lsof +L1` | 02 |
| Bad `fstab` entry | Wrong UUID in `/etc/fstab` with `nofail`, reboot | `systemctl --failed`, `journalctl -b`, `findmnt --verify` | 02 |
| CPU hog | `yes > /dev/null &` | `top`, `ps aux --sort=-%cpu`, `kill` | 01 |
| Wrong time | `sudo timedatectl set-ntp false; sudo date -s "2 years ago"` (restore: `sudo timedatectl set-ntp true`) | Cert and log timestamps look wrong | 01, 09 |
| Key login refused | `chmod 777 ~alice/.ssh` | `journalctl -u sshd`: bad ownership or modes | 06 |
| Backup job silently fails | Change the restic password file | Job exits non-zero; they must notice | 07 |
| Cert expired or wrong name | Install a 1-day cert, or a cert without the right SAN | `openssl s_client`, `-checkend`, `curl -v` | 09 |
| Health check fails | `sudo mv /srv/app-b/index.html{,.off}` | HAProxy stats page; B goes DOWN | 10 |
| Drift | Edit `/etc/httpd/conf.d/lab.conf` by hand | `ansible-playbook --check --diff` | 11 |

Combine two faults for a harder round. Start with the lower layer, since fixing the upper one first often hides the lower.

## 00: How a sysadmin thinks

**Review:** read the learner's checklist, change plan, and incident update.

**Good answers sound like:**
- Troubleshooting checklist: symptom and scope first, then "what changed," then a layered check (network, OS, service, application), one change at a time, verify, record.
- Change plan for the SSH port: test with a second session left open, keep the old port working until the new one is proven, open the firewall for the new port first, have console access as a fallback.
- Incident update: states impact, current action, and the next update time. No blame, no speculation as fact.

- A change ticket has a reason, exact steps, a test plan, a rollback plan, impact, and a window, and it names a reviewer who isn't the author.

**Watch for:** jumping to restart or reboot, "check the logs" with no idea which logs, no rollback plan, a ticket that says "update config" with no steps.

## 01: Linux fundamentals

**Skip check (10 minutes):** can they explain `rwxr-x---`, show a service's logs, install and query a package, write a small unit file, and find the highest CPU process? If yes, skip the module.

**Good answers sound like:**
- A service won't start: `systemctl status`, `journalctl -u`, check the config syntax, check permissions and paths, check dependencies and ports.
- `stop` vs `disable` vs `mask`: stop ends it now; disable stops it starting at boot; mask prevents it starting at all, even manually or as a dependency.
- `kill` sends SIGTERM and lets the process clean up; `kill -9` can't be caught, so there's no cleanup.
- `-rw-r----- root apache`: root can read and write; members of `apache` can read; everyone else has nothing.

- Vim: they can save, quit without saving, search, and replace without looking it up. tmux: they can detach, reconnect, and reattach.

**Watch for:** `chmod 777`, rebooting to fix a service, running everything as root, reaching for nano.

## 02: Storage

**Skip check (10 minutes):** have them add and mount a disk from scratch. They should run `lsblk` before they write anything, pick the right device, and verify with `mount -a` before rebooting.

**Good answers sound like:**
- Partition vs VG vs LV vs filesystem: the partition is a slice of a disk; the VG is a pool of PVs; the LV is a volume carved from the pool; the filesystem is the structure on top.
- UUIDs in `fstab` because device names can change between boots.
- `nofail` stops a missing non-critical disk from blocking boot, but you wouldn't use it for something the system can't run without (for example `/var`).
- `mount -a` surfaces errors now, not at 3 AM during boot.
- 95% full: extend the LV (`lvextend -r`); if the VG has no free space, add a PV with `vgextend` first. XFS can't be shrunk, ext4 can (offline).
- `df` 100% vs `du` 60%: deleted-but-open file (`lsof +L1`), a mount hiding files underneath (files in `/mnt/data` written before the disk was mounted), or reserved blocks.

**Watch for:** formatting without checking the device, editing `fstab` and rebooting immediately, treating LVM as a backup.

## 03: Web server, firewall, troubleshooting

**Good answers sound like:**
- "Website is down" first five: is it down for everyone, can I reproduce it, is the service running, is it listening on the right address and port, is the firewall open, what do the logs say, what changed.
- `--permanent` writes to disk but doesn't change runtime; `--reload` loads permanent into runtime.
- Not listening vs blocked: `ss -tlnp` on the server shows listening; from the client, "refused" means nothing is listening (or a reject rule), a timeout usually means a drop.
- The 403: `chown root:apache`, `750` on the directory, `640` on the file, because Apache needs read on the file and execute (traverse) on the directory.

**Watch for:** disabling the firewall, `chmod -R 777`, restarting without `configtest`.

## 04: Networking and DNS

**Good answers sound like:**
- Typing a URL: DNS lookup (cache, `/etc/hosts`, resolver), route to the gateway if remote, TCP handshake, TLS handshake, HTTP request, response.
- Can ping an IP but not a name: DNS. Confirm with `dig` against the configured server and against `1.1.1.1`.
- "Connection refused": reached the host but nothing is listening. "Timed out": packets are dropped somewhere (firewall, route, host down).
- `10.1.4.130/26`: network `10.1.4.128`, broadcast `10.1.4.191`, 62 usable hosts. `10.1.4.100` is not in the subnet; `10.1.4.190` is.
- `192.168.10.37/24`: network `192.168.10.0`, broadcast `192.168.10.255`, 254 hosts. `172.16.5.9/20`: network `172.16.0.0`, broadcast `172.16.15.255`, 4094 hosts.

**Watch for:** blaming "the network" without a test, ignoring the difference between refused and timed out.

## 05: Git and scripting

**Good answers sound like:**
- `set -euo pipefail`: exit on error, treat unset variables as errors, fail a pipeline if any stage fails.
- Quote variables to survive spaces and globbing. `rm -rf "$DIR/"*` with empty `$DIR` is catastrophic.
- Password pushed to git: rotate it first, then remove it from history if needed. Deleting the file is not enough.
- `git revert` adds a new commit that undoes an old one (safe for shared history); `git reset --hard` discards work.

- Commits look like `fix(backup): use correct repository path`, not "updated stuff". The body says why, and the footer references the ticket.

**Watch for:** scripts with no error handling, no dry run on anything destructive, unquoted variables, pasting AI scripts they can't explain.

## 06: Identity and access

**Good answers sound like:**
- Authentication proves identity; authorization decides what you can do.
- `nologin` for service accounts: nobody should be able to log in interactively as them.
- `visudo` validates syntax; a broken `/etc/sudoers` can lock everyone out.
- `~/.ssh` must be `700` and `authorized_keys` `600`, owned by the user; otherwise `sshd` ignores the key.
- Leaver: disable first (lock and expire), end sessions, remove keys, review owned files and cron jobs, and also revoke access outside the server (VPN, cloud, shared secrets, API tokens). Record it.

**Watch for:** shared accounts, giving `wheel` to everyone, disabling password auth before testing keys, closing the only SSH session while changing `sshd`.

## 07: Backups and data protection

**Good answers sound like:**
- RPO sets backup frequency; RTO sets method and tooling.
- Silent failure: monitor the job's exit status, alert on the absence of a successful run (dead-man's switch), test restores on a schedule.
- A backup on the same server doesn't protect against the server being compromised or the disk dying. Off-box and with separate credentials.
- Restores should preserve owner and mode (restic does; `rsync -a` does; `cp` may not). Check with `ls -l`.
- Replication copies a deletion immediately; a backup keeps history.

**Watch for:** never having restored, the repository on the same disk, "we have RAID."

## 08: Monitoring and alerting

**Good answers sound like:**
- Three things to monitor on any Linux server: disk space, whether the service answers, and CPU/memory. Also certificate expiry, backups, and time sync.
- Require a duration, not a single sample, so a spike isn't an incident.
- The 3 AM CPU alert that fixes itself: find the cause (a scheduled job?), then either tune the threshold and duration or fix the job. Don't just mute it.
- Worth waking someone: customer-facing impact right now, or imminent data loss.
- A heartbeat alert catches the case where the check or the job dies silently.

**Watch for:** wanting to alert on everything, thresholds with no owner or action, forgetting to monitor the monitor.

## 09: Certificates and TLS

**Good answers sound like:**
- Expired cert in production: confirm with `openssl s_client`, get a new cert (renew or reissue), install the full chain, reload the service, verify from outside, then find out why it wasn't renewed and fix the process or automate it.
- 200 certs: an inventory with owner, expiry, and location; automated expiry checks; alerts at 30/14/7 days; ideally automated issuance and renewal.
- A leaked private key means anyone can impersonate the server; revoke and reissue, and rotate.
- Automation beats reminders because it doesn't depend on someone remembering, and lifetimes are getting shorter.
- Keys: root-only (`600`, owned by root), not in git or email.

**Watch for:** manual renewal with no inventory, keys in shared folders, missing intermediates.

## 10: Reverse proxy and load balancing

**Good answers sound like:**
- When B went down the user saw nothing, because the health check removed it from rotation. Without the proxy, a share of requests would fail.
- The health check only tests what it requests (a static page). It can miss a broken database or a broken page deeper in the app.
- The proxy is now the single point of failure. Real answer: run two with a floating IP (keepalived), or use a managed load balancer.
- Apache logs the proxy's IP; the real client IP is in `X-Forwarded-For`.
- `leastconn` suits long or uneven requests; `roundrobin` suits uniform short ones.
- With TLS terminated at the proxy, renewal happens there, and the proxy-to-backend hop is plaintext unless re-encrypted.

**Watch for:** not validating the config with `haproxy -c`, forgetting to open the firewall, overlooking the new single point of failure.

## 11: Configuration management

**Good answers sound like:**
- Drift is servers diverging through manual changes; it produces "works on A, not B."
- Idempotent means running again changes nothing. Prove it with a second run showing `changed=0`.
- Handlers run only when notified by a changed task, so the service restarts only when config actually changed.
- `command`/`shell` aren't idempotent unless guarded; prefer a module, or add `creates:`/`when:`.
- Don't hand-edit managed servers: the next run reverts it, or it hides the drift. Change the code.

**Watch for:** secrets in the repository, playbooks that only work once, skipping `--check`.

## 12: Capstone and interview

Run the acceptance tests in the module and then break something live. Score against the sign-off criteria. A smaller, understood result beats a larger, copied one. Ask about gaps: a learner who names their own weaknesses is usually further along than one who says everything works.
