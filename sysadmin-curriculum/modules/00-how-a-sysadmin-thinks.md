# Module 00: How a sysadmin thinks

**Time:** about 2 hours of reading and written exercises. No lab.
**Prerequisites:** none.

## By the end you can

- Describe what a sysadmin is responsible for in terms of availability, security, and recoverability.
- Apply a step-by-step troubleshooting method instead of guessing.
- Plan a change so it can be tested and undone, and write it up as a change ticket.
- Prioritise problems by impact and urgency.
- Write a clear incident update.

## Why this matters

Tools change every few years. The habits behind them don't. A help desk tech is usually good at fixing the thing in front of them. A sysadmin is also expected to prevent problems, notice them early, document what they did, and explain it to others. Most of the rest of this curriculum is tools; this module is the thinking that makes the tools useful.

## Concepts

### 1. The job is mostly preventing and noticing

Keep services **available**, **secure**, and **recoverable**. Firefighting is the smaller part of the job. If you're always firefighting, something upstream (monitoring, patching, documentation, change control) is broken.

### 2. A troubleshooting method

When something breaks:

1. **Symptom.** What exactly is wrong, and who is affected? "The website is down" might mean one user, one office, or everyone.
2. **What changed?** Recent deploys, patches, config edits, certificate expirations, disk filling up.
3. **Reproduce it.** If you can't reproduce it, you can't tell whether you fixed it.
4. **Work through the layers.** Network, then the OS, then the service, then the application. Most problems are lower than people assume.
5. **One hypothesis, one change.** Change one thing, test, and undo it if it didn't help. Changing three things at once leaves you not knowing which one worked.
6. **Verify the fix** from the user's point of view, not just yours.
7. **Write it down.** What it was, how you found it, how you fixed it.

Red flags: restarting things at random, "it fixed itself," and fixes you can't explain.

### 3. Change safely

Before you change something:

- Know what it does and what depends on it.
- Test it somewhere that isn't production first.
- Have a **rollback plan**: a snapshot, a backup of the config, the previous package version.
- Pick a safe time and tell the people affected.
- Record what you did and when.

Ask yourself: "If this goes wrong at 4:55 PM on a Friday, what do I do?"

In a team this becomes a **change process**: a ticket describing the change, a second person reviewing it, and an agreed window. The details and the ticket template are in [STANDARDS.md](../STANDARDS.md).

### 4. Least privilege

Give people and processes only the access they need. Use individual accounts (no shared logins), `sudo` rather than a root login, and SSH keys rather than passwords. Remove access when it's no longer needed. This is why `chmod 777` is never the answer: it fixes the symptom by removing the protection.

### 5. Patching and inventory

Unpatched systems are a common way in for attackers. Test patches, schedule them, reboot when required, and know when software reaches end of life. You can't patch, renew, or back up what you don't know you have, so **inventory comes first**.

### 6. Documentation

Write for the person at 3 AM, who may be you in six months. A good runbook says what the system is, how to check it, how to restart it, who owns it, and what depends on it. Ticket notes count: what you saw, what you tried, what fixed it.

### 7. Automate what you do twice

If you do something twice, consider scripting it. Keep scripts and configs in git. The goal is the same result every time and fewer typos at 3 AM. Modules 05 and 11 cover this.

### 8. Redundancy is not backup

RAID, replication, and a second server keep things running when hardware fails. Backups bring data back after deletion, corruption, or ransomware. They solve different problems, and replication copies your mistakes instantly.

### 9. Security basics

Run only the services you need, open only the ports you need, patch, use MFA, keep secrets out of scripts and git, read the logs, and layer your defenses so one failure isn't fatal.

### 10. Working with people

- **Prioritise by impact and urgency**, not by who's loudest. A broken payroll system on payday outranks a cosmetic bug.
- **Give status updates before people ask.** Silence makes people assume the worst.
- **Escalate early**, and say what you've already tried.
- **Do blameless post-incident reviews:** what happened, why, and what changes. Not who to blame.
- "I don't know yet. I'll find out and update you by 3 PM" is a good answer.

### 11. Standards

Teams agree on how things are built, changed, documented, and named so that anyone can pick up anything. Read [STANDARDS.md](../STANDARDS.md) now. You'll be held to it in every deliverable: everything in git, no manual changes on managed servers, restores and monitoring proven before go-live, and documentation written.

### An incident update, in four lines

```
What's happening:   Customers can't reach the order site (HTTP 503).
Impact:             All web orders since about 09:10. Phone orders unaffected.
What we're doing:   Checking the load balancer and the app servers. Cause not known yet.
Next update:        10:00, or sooner if we find the cause.
```

## Exercises

These are written; there is no lab.

1. **Scenario:** "A user says the website is down." Write your first five checks, in order, and why that order.
2. **Scenario:** "Backups have been failing for a week and nobody noticed." Write what probably happened and the three changes that would prevent it.
3. **Change plan:** You need to change the SSH port on a server you can only reach over SSH. Write the plan: what could go wrong, how you test it, and how you roll back without locking yourself out. Write it as a change ticket using the template in [STANDARDS.md](../STANDARDS.md).
4. **Incident update:** Write an update for a made-up outage using the four-line format above.

## Check your understanding

- What is the difference between a symptom and a cause?
- Why change only one thing at a time?
- What are three things you'd want in place before changing a production server?
- Why does inventory come before patching and certificate renewal?
- Why isn't RAID a backup?
- What are the three parts of the change process, and what does the reviewer's job look like?
- Who should get a status update during an outage, and how often?

## Deliverable

A short document containing: your personal troubleshooting checklist (one page), your change ticket from exercise 3, and your incident update from exercise 4.

## Use AI well here

After you've written your checklist, ask Claude to critique it: "What would an experienced sysadmin add or reorder?" Decide which suggestions are valid and why. Then ask it to give you three new scenarios and answer them before you ask it for its own answers.

## Go deeper

- *The Practice of System and Network Administration* (Limoncelli, Hogan, Chalup) is the classic.
- Google's SRE book (free online), chapters on incident response and postmortems.
- Julia Evans' zines at wizardzines.com for debugging mindsets.
