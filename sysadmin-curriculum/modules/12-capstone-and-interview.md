# Module 12: Capstone and interview

**Time:** about 8 hours for the build, plus a 45-minute presentation and mock interview.
**Prerequisites:** all earlier modules (00-11).

## By the end you can

- Build a small service from scratch, the way you'd want to inherit one: repeatable, secured, monitored, backed up, and documented.
- Prove it works, and prove it recovers.
- Explain every decision, and answer scenario questions without looking things up.

## The project

Build **a small internal web service** on a clean snapshot of the lab VM, then defend it.

### What it must do

Starting from a clean snapshot, one command (`ansible-playbook site.yml`, or your own wrapper) from your git repository builds:

1. **Web tier.** Two Apache backends on local ports, each serving a different page.
2. **Front door.** HAProxy on 80 and 443, round-robin, with health checks and a stats page, terminating TLS with a certificate you generated.
3. **Firewall.** firewalld on, only the services needed open, and nothing more.
4. **Storage.** The web content lives on its own logical volume, mounted persistently via `fstab`, sized with room to grow.
5. **Access.** A deploy account with SSH key login and a narrowly scoped `sudo` rule, root login and password login for SSH disabled, and no shared accounts.
6. **Backups.** A scheduled backup of the web content and configuration to a repository that isn't on the same disk, with a **tested restore** and a measured restore time.
7. **Monitoring.** A check on the site from outside the proxy, a disk-space alert, a certificate expiry check, and a way to learn if the backup stops running.
8. **Documentation.** A README and runbook: what this is, how to rebuild it, how to check it, how to restart each piece, how to restore, and who owns what.

Anything that can't be automated yet (a UI step, a manual certificate request) should be listed honestly as a gap, not hidden.

### Rules

- Everything in git. No secrets in the repository.
- Principle of least privilege throughout. No `chmod 777`, and the firewall stays on.
- You do the work. You may use AI as described in [AI-USE.md](../AI-USE.md), but you must be able to explain every line.
- Time-box it. A smaller thing that works and is understood beats a big thing that isn't.

## Acceptance tests

The mentor runs these. Write your own versions first.

| # | Test | Pass if |
|---|------|---------|
| 1 | Restore the clean snapshot, run the build once | Finishes with no manual steps (or documented gaps) |
| 2 | Run the build a second time | `changed=0` |
| 3 | `curl` the site repeatedly | Alternates between backends |
| 4 | Stop one backend | No user-visible errors; alert fires |
| 5 | Reboot the VM | Everything comes back: mounts, services, firewall |
| 6 | Run `firewall-cmd --list-all`, `ss -tlnp` | Only what's needed is open and listening |
| 7 | Delete the web content, restore from backup | Content back with correct owner and mode; time recorded |
| 8 | Fill the disk past the threshold | Alert fires, then resolves when cleaned up |
| 9 | Check the certificate and the expiry monitor | Expiry is known and monitored |
| 10 | Read the README with no help | A stranger could rebuild it |

## Presentation (20 minutes)

Walk the mentor through:

1. A diagram of the whole thing.
2. The decisions you made and the trade-offs: what you chose, and what you rejected and why.
3. A live demo: rebuild, failover, restore.
4. What you would do next, and what's still weak.

The mentor will pick something and break it while you watch. Diagnose it out loud, using the method from module 00.

## Mock interview (about 20 minutes)

Scenario questions, not trivia. Think out loud.

1. "A Linux server is slow. Walk me through what you'd do."
2. "A certificate expired and the website is down. What do you do, and what do you change afterward?"
3. "Backups have been failing for a week and nobody noticed. What happened, and how do you prevent it?"
4. "A service works on localhost but not from another machine. What do you check?" (Listening address, firewalld, upstream network, then permissions.)
5. "A disk is 100% full but `du` doesn't add up. What could it be?"
6. "Someone leaves the company. What do you do about their access?"
7. "You need to change a configuration on a production server. What's your process?"
8. "How would you make sure two servers are configured the same way?"
9. "Tell me about a time you fixed something in production. What did you learn?"
10. "How do you learn a new tool you've never used?" (Docs, a lab, then ask people.)
11. "How do you use AI tools in your work, and what do you not trust them with?" (Good answers: explaining and drafting, verifying against docs, testing first, never pasting secrets.)

## Sign-off criteria

Signed off when the learner can:

- [ ] Rebuild the service from a clean snapshot, repeatably.
- [ ] Explain the purpose of every setting without looking it up.
- [ ] Diagnose a surprise fault using a method, not guesses.
- [ ] Show that backup, monitoring, and access control work, not merely exist.
- [ ] Describe the known gaps and what they'd do next.
- [ ] Use AI to understand, not just to get commands.

## Deliverable

The repository, the diagram, the runbook, the measured restore time, and your own written reflection: what was hardest, what you'd do differently, and what you want to learn next.

## What next

- Keep building a homelab (a few Rocky VMs, Ansible, containers).
- Consider **RHCSA**. It's hands-on, and the standard credential for RHEL shops. It covers SELinux, which this lab skips, so study it separately. CompTIA Linux+ and LFCS are alternatives.
- Keep your repository and notes. Concrete work beats claims in an interview.
