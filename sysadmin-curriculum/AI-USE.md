# Using AI (Claude) as a sysadmin tool

An AI assistant like Claude is one of the most useful tools a sysadmin has. Used well, it's like a patient senior admin you can ask at any hour. Used badly, it's the easiest way to copy commands without learning anything. The rule:

**AI is a tutor and a second pair of eyes, not a replacement for understanding.**

## Where it genuinely helps

- **Explaining errors and logs.** Paste a journal or `error_log` excerpt and ask what it means and what to check.
- **Explaining unfamiliar output or config.** "What does each line of this `haproxy.cfg` / `openssl x509 -text` / `df` output mean?"
- **Drafting and reviewing scripts and configs.** Bash, systemd units and timers, Apache and HAProxy configs, Ansible playbooks, cron entries, `jq`, `awk`, regex.
- **Reviewing your work.** "What failure modes does this backup script have?"
- **Translating between tools.** Docker to Podman, cron to a systemd timer, Apache directives to nginx.
- **Writing documentation.** Runbooks, change notes, post-incident summaries, and READMEs from your rough notes.
- **Troubleshooting walk-throughs.** "A service works on localhost but not remotely. What do I check, in what order?"
- **Learning and practice.** "Quiz me on firewalld." "Explain RPO vs RTO with an example." "Give me a broken-lab scenario to diagnose."
- **Interview prep.** Mock interviews and critique of your answers.

## Rules

1. **Try first, then ask.** Read the error, check the logs, check the man page. Then ask, and include what you already tried. This keeps your troubleshooting skills growing.
2. **Understand before you run.** Read every command it gives you. If you can't say what it does, ask "explain each part" before running it. Never paste a command you don't understand into a real server.
3. **Test in the lab first.** Especially anything destructive: `rm`, `dd`, `--delete`, firewall changes, `chmod -R`, disk and LVM commands, and anything on production.
4. **Verify against the docs.** AI can be confidently wrong, mix up versions, or suggest an option that doesn't exist on Rocky 9 or RHEL. The man page and vendor docs are the source of truth.
5. **Protect sensitive data.** No passwords, private keys, API tokens, customer data, or internal hostnames and IPs in prompts unless your organization has approved the tool for that. Redact first. Find out your organization's AI policy before using it on the job.
6. **Give it context.** Good answers need good questions. Use the template below.
7. **Ask it to teach.** "Explain why, not just what to type." and "Quiz me on this."

## A good prompt

```
Environment: Rocky Linux 9, httpd 2.4, SELinux disabled, firewalld on.
Goal: what I'm trying to do.
What I ran: the exact commands.
What happened: the exact error text or log lines, with secrets removed.
What I already checked: what I tried.
Explain what's likely wrong and why, and give me checks to confirm
it before suggesting a fix.
```

Compare that with "apache not working help." The difference in answer quality is the lesson.

## Try this in every module

After you fix the break-it exercise yourself, ask the AI to review what you did, explain it back to you, and quiz you on it. If you can't explain a fix without it, you haven't learned it yet. Each module has a suggested AI exercise at the end.

## For mentors: keeping it honest

- **The "AI off" check.** Occasionally ask the learner to explain a command or fix without looking anything up. If they can't, they're copying, not learning.
- **Ask for the source.** "How do you know that's right?" The right answer is "I checked the docs" or "I tested it in the lab."
- **Watch the order.** If they go to the AI before reading the error, send them back to rule 1.
- **Agent-style tools are a bigger deal.** Tools that run commands on a machine for you (like Claude Code in a terminal) act with the user's permissions. Keep them out of production until the learner understands the commands, prefer read-only access and approval before each action, and follow your organization's policy.

## Questions to settle for your organization

- [ ] Is there an approved AI tool, and what data may go into it?
- [ ] Are there rules about AI-generated scripts or config changes (review, change control)?
- [ ] Is AI-assisted work expected, allowed, or restricted?
