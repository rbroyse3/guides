# Sysadmin Curriculum

A self-paced, hands-on path to bring a co-op, junior admin, or help desk tech up to speed on what a good generalist Linux sysadmin should know, and how to build things repeatably and correctly.

It is built around Rocky Linux 9 (binary-compatible with RHEL 9), so everything transfers to RHEL shops. The tools are open and free, so you can run all of it in a home lab or a spare VM.

## Who it's for

- New co-ops and interns who are comfortable with computers but new to systems administration.
- Help desk techs moving toward a sysadmin role.
- Junior admins who learned by doing and want the fundamentals filled in.

It is not a certification course. It will help with RHCSA, but it doesn't follow the exam objectives.

## How it works

Each module follows the same loop:

1. **Read** the concepts (about 30-45 minutes).
2. **Do** the hands-on lab. You type; nobody types for you.
3. **Break it** on purpose, then find and fix the fault using the logs and tools.
4. **Check yourself** with the questions at the end. If you can't answer without looking it up, go back.
5. **Produce the deliverable**, a short written note, script, or config that proves you did it.
6. **Review with a mentor** for 30 minutes. The mentor asks questions and picks a fault for you to find. When you can explain it without notes, the module is signed off.

The estimated times below are for the hands-on part. They are estimates from planning, not measurements; adjust them after your first run.

## Modules

| # | Module | Hands-on time | You can by the end |
|---|--------|---------------|--------------------|
| 00 | [How a sysadmin thinks](modules/00-how-a-sysadmin-thinks.md) | 2 h | Apply a troubleshooting method, plan a safe change, write an incident update |
| 01 | [Linux fundamentals](modules/01-linux-fundamentals.md) | 4 h | Navigate, manage users, packages, services, logs, and processes |
| 02 | [Storage: disks, partitions, LVM, and mounting](modules/02-storage-and-filesystems.md) | 5 h | Partition and format a disk, use LVM, mount it persistently, and grow it |
| 03 | [Web server, firewall, and troubleshooting](modules/03-web-server-and-troubleshooting.md) | 4 h | Run Apache, use firewalld, diagnose a 403 without `chmod 777` |
| 04 | [Networking and DNS](modules/04-networking-and-dns.md) | 5 h | Explain and debug IP, routing, DNS, and ports |
| 05 | [Git and scripting](modules/05-git-and-scripting.md) | 5 h | Write safe Bash, keep work in git |
| 06 | [Identity and access](modules/06-identity-and-access.md) | 4 h | Manage accounts, sudo, and SSH keys with least privilege |
| 07 | [Backups and data protection](modules/07-backups-and-data-protection.md) | 4 h | Back up, schedule, and restore, and prove the restore works |
| 08 | [Monitoring and alerting](modules/08-monitoring-and-alerting.md) | 4 h | Add checks, design alerts that are actionable, avoid noise |
| 09 | [Certificates and TLS](modules/09-certificates-and-tls.md) | 4 h | Inspect, issue, trust, and renew certificates |
| 10 | [Reverse proxy and load balancing](modules/10-reverse-proxy-and-load-balancing.md) | 3 h | Put HAProxy in front of apps, test failover, terminate TLS |
| 11 | [Configuration management](modules/11-configuration-management.md) | 5 h | Build a server from code and rebuild it identically |
| 12 | [Capstone and interview](modules/12-capstone-and-interview.md) | 8 h | Build a small service end to end, then defend it |

About 57 hours of hands-on work plus thirteen 30-minute reviews. At 4-5 hours a week that's roughly 12-14 weeks. A full-time co-op could finish in 2-3 weeks.

### Order

```
00 -> 01 -> 02 -> 03 -> 04
        |           |     |
        |           |     +-> 08 (monitoring), 09 (certs) -> 10 (proxy)
        |           +-> 07 (backups needs 02 and 03)
        +-> 05 (git, scripting) -> 11 (config management, also needs 03)
        +-> 06 (identity)
                         all -> 12 (capstone)
```

Do 00-03 first, in order. After that, 04-06 can go in any order. 07 needs 02 and 03, 08 and 09 need 04, 10 needs 03 and 09, 11 needs 03 and 05, and 12 comes last. Someone who already knows Linux can skip 01 after passing its check questions (see the mentor notes).

## Before you start

1. Read [LAB-SETUP.md](LAB-SETUP.md) and get a lab VM.
2. Read [AI-USE.md](AI-USE.md). AI is encouraged, but there's a right way to use it that keeps you learning.
3. Copy [PROGRESS-TRACKER.md](PROGRESS-TRACKER.md) and fill in the starting-point checklist so you and your mentor know where to start.

## Ground rules

- **You do the typing.** A mentor asks questions and points the way.
- **Look it up first.** Try `man`, `--help`, the logs, and the official docs. Ten honest minutes, then ask.
- **Break things.** Troubleshooting is learned from broken systems, not working ones.
- **No shortcuts that hide the problem.** No `chmod 777`, and no stopping the firewall to "fix" connectivity. Find out why it's blocked, then open only what's needed.
- **Snapshot before you experiment**, and know how to roll back.
- **Write things down.** Each module's deliverable becomes your notes and your portfolio.

## For mentors

Start with [MENTOR-GUIDE.md](MENTOR-GUIDE.md). Per-module review questions, what good answers sound like, and a pool of faults to inject are in [MENTOR-NOTES.md](MENTOR-NOTES.md). Those two files contain answers, so don't hand them to the learner.

## Making it yours

The modules use open tools so the curriculum is portable. If your team uses different tools (a commercial backup product, a different monitoring system, a specific config management tool), keep a short separate "environment guide" that maps each module to your tools. The concepts don't change; only the buttons do. Keep that guide next to this one, not inside it.

## Not written yet

These are natural extensions, not covered here: SELinux, containers (Docker/Podman) in depth, Windows and Active Directory, cloud (IaaS) basics, and Kubernetes. SELinux is disabled in the lab to keep the early modules focused. If your production servers run it enforcing, add an SELinux module before sign-off.

## Contents

| File | Audience | Purpose |
|------|----------|---------|
| [README.md](README.md) | Everyone | This overview |
| [LAB-SETUP.md](LAB-SETUP.md) | Everyone | Build the lab VM |
| [AI-USE.md](AI-USE.md) | Everyone | Using Claude or another AI assistant well |
| [PROGRESS-TRACKER.md](PROGRESS-TRACKER.md) | Learner and mentor | Track modules and starting level |
| [modules/](modules/) | Learner | The thirteen modules (00-12) |
| [MENTOR-GUIDE.md](MENTOR-GUIDE.md) | Mentor | How to run it |
| [MENTOR-NOTES.md](MENTOR-NOTES.md) | Mentor | Review questions, good answers, fault pool |
