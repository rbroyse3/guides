# Standards: how we build, change, document, and name things

These are the house rules. Every deliverable in this curriculum is held to them, because the habits matter more than any single tool. If your organization works differently, edit this file; it's meant to be adapted. The modules refer back to it.

## The non-negotiables

1. **Everything in code and git.** Server configuration, scripts, and documentation live in a repository and are applied from it. If it isn't in git, it doesn't exist.
2. **No manual changes on managed or production servers.** If you hand-edit a managed server, the next run reverts it or hides the drift. Change the code, review it, then apply it.
3. **A restore is tested before go-live.** A backup you haven't restored is not a backup. The restore time is measured and written down.
4. **Monitoring is in place before go-live.** Checks exist, alerts have an owner and a next step, and you know how you'd hear about a silent failure.
5. **Documentation is written.** A service without a runbook isn't finished. Writing it is part of the work, not a favor afterward.

## Change process: ticket, peer review, window

Every change to production has all three:

1. **A ticket** that describes the change before it happens.
2. **A peer review** by a second person who isn't the author.
3. **A window**, an agreed time when the change is allowed, and when the people affected know to expect it.

A change ticket contains:

```
Title:
Why:              the problem or request
What changes:     systems, files, services
Steps:            exact commands or the PR to apply
Test plan:        how we'll know it worked
Rollback plan:    how we undo it, and how long that takes
Impact:           who or what is affected, and for how long
Window:           date, time, duration
Author / Reviewer:
Outcome:          filled in afterward: what happened, anything unexpected
```

The reviewer's job is to ask "what if this fails?" and "how do we know it worked?" A change without a rollback plan isn't ready. Record the outcome in the ticket when it's done.

**Emergencies.** If something is down right now, fix it first, record what you did immediately, and have it reviewed afterward. An emergency is a reason to move fast, not a reason to skip the record.

In the lab there's no real ticket system. Write the ticket anyway: the habit is the point.

## Documentation

Every service or system has a **runbook**, stored in the same repository as its code, as `README.md` plus `docs/runbook.md`. It's reviewed in the same pull request as the change it describes. If you change the system, you update the runbook.

A runbook contains:

- **Purpose:** what this is and why it exists
- **Owner:** a named person or team
- **Architecture:** a diagram or a short description of the pieces
- **Dependencies:** what it needs, and what depends on it
- **How to rebuild it** from nothing
- **How to check it's healthy**
- **How to restart it**
- **How to restore it** from backup, with the measured time
- **Common failures** and what to do about each
- **Contacts and escalation**
- **Last reviewed** date

Write for the person at 3 AM who has never seen this system. That might be you in six months.

## Scripts and code

- Bash scripts start with `#!/usr/bin/env bash` and `set -euo pipefail`, and pass `shellcheck` cleanly.
- A header comment states the purpose, usage, and exit codes.
- Quote variables. Guard destructive commands (`${DIR:?}`). Offer a `--dry-run` for anything that deletes or changes things.
- Log to stderr or the journal (`logger`), and exit non-zero on failure.
- No secrets in scripts or in git.
- Scripts are idempotent where possible, so running one twice is safe.
- When a Bash script grows past what you can hold in your head (complex data handling, lots of branching), move to a language with real data structures, such as Python.
- Code is reviewed before it goes to production.

## Commits and branches

Commit messages follow **Conventional Commits**:

```
<type>(<optional scope>): <short description>

<optional body: why, not just what>

<optional footer, e.g. Refs: TICKET-123>
```

Common types: `feat` (new capability), `fix` (bug fix), `docs` (documentation only), `chore` (maintenance, no behavior change), `refactor` (restructure without changing behavior), `test`, `ci`.

```
feat(proxy): add health check to the backend pool
fix(backup): use the correct repository path in the timer
docs(runbook): add restore steps for the web service
chore: update ansible collections
```

- Subject line in the imperative ("add", not "added"), about 72 characters or fewer, no trailing period.
- One logical change per commit.
- Reference the ticket in the footer.

Branches are named `<type>/<short-description>`, for example `feat/haproxy-health-check` or `fix/backup-path`. `main` is only changed through reviewed pull requests.

## Naming

**Hostnames** follow `<site><purpose><number>`:

- `<site>`: a short site code (letters)
- `<purpose>`: what the machine does (`web`, `db`, `proxy`, `puppet`, `backup`)
- `<number>`: two digits, starting at `01`

Examples with a made-up site code `xyz`: `xyzweb01`, `xyzweb02`, `xyzdb01`, `xyzproxy01`. Lowercase, no hyphens or underscores. Names that follow a pattern are sortable, scriptable, and guessable. In the lab, pick a fictional site code and name your VM and your inventory accordingly.

**Files, scripts, and units:** lowercase with hyphens (`disk-report.sh`, `lab-backup.service`).

**Accounts, tickets, and anything else:** follow your organization's scheme.

## Where things live

| What | Where |
|------|-------|
| Local admin scripts | `/usr/local/bin` |
| Application code | `/opt/<name>` |
| Configuration | `/etc` (use `conf.d` drop-in files rather than editing the main file) |
| Custom systemd units | `/etc/systemd/system` (never edit vendor units in `/usr/lib/systemd/system`; use drop-ins) |
| Service state | `/var/lib/<name>` |
| Data a server serves | `/srv/<name>` |
| Logs | `/var/log`, or the journal |
| Backups of config before you change it | next to the original with a dated suffix, for example `httpd.conf.2026-11-05.bak`, and in git |

## Everyday tooling

- **vim, not nano.** You'll be on servers where vim is all you get. Learn the basics early (module 01) and you'll be comfortable anywhere.
- **tmux for long sessions.** Work survives a dropped SSH connection, and you can reattach. Use it for anything that takes more than a few minutes.
- **Learn the defaults first.** Don't customize your shell or editor until you're comfortable on a bare server.
