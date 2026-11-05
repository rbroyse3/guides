# Module 06: Identity and access

**Time:** about 4 hours.
**Prerequisites:** module 01. A second VM or your own computer to SSH from.

## By the end you can

- Manage users and groups, and lock, expire, and remove accounts properly.
- Configure `sudo` safely with drop-in files and `visudo`.
- Set up SSH key authentication and harden `sshd` without locking yourself out.
- Find failed logins and see who has logged in.
- Describe joiner/mover/leaver handling and what a central directory (LDAP/Active Directory) adds.

## Why this matters

Most real-world breaches involve credentials or over-broad access. A sysadmin decides who can do what on every system, then has to prove it and clean it up when people leave. Getting it right is mostly habit and discipline.

## Concepts

- **Authentication vs authorization.** Authentication is proving who you are; authorization is what you're allowed to do.
- **Least privilege.** Give the minimum access for the job, and for as long as needed.
- **Individual accounts.** No shared logins, so actions can be traced to a person.
- **Service accounts.** Programs run as their own users with no login shell (`/sbin/nologin`) and no password.
- **Local accounts:** `/etc/passwd` (public info), `/etc/shadow` (hashed passwords, root-only), `/etc/group`.
- **sudo** grants specific elevated rights. Configure it through `visudo` and files in `/etc/sudoers.d/`, which validates syntax so a typo can't lock everyone out.
- **SSH keys.** A key pair has a **private** key (never leaves your machine, protect it with a passphrase) and a **public** key (goes in `~/.ssh/authorized_keys` on the server). Key auth is stronger than passwords and can't be brute-forced the same way.
- **Joiner/mover/leaver.** New people get access; people who change roles gain and **lose** access; people who leave lose it all, promptly. "Access creep" and orphaned accounts are common audit findings.
- **Central directories.** In larger environments, accounts live in LDAP or Active Directory and Linux hosts use **SSSD** (often joined with `realm join`) so users and groups come from one place. You'll manage local accounts in this lab; know that the model above scales up.
- **MFA** adds a second factor, and **break-glass accounts** are tightly controlled emergency logins.
- **Auditing.** Logs show who logged in and what they ran with `sudo`.

## Lab

Snapshot first. **Keep one SSH session open to the VM at all times while you change SSH settings.** If a change breaks new logins, the open session lets you undo it.

### 1. Users and groups

```bash
sudo groupadd webteam
sudo useradd -m -G webteam alice
sudo useradd -m bob
sudo passwd alice
id alice; getent passwd alice; getent group webteam
sudo usermod -aG webteam bob
sudo chage -l alice                       # password aging
sudo passwd -l bob                        # lock the account
sudo passwd -S bob                        # LK = locked
sudo passwd -u bob
```

A service account:

```bash
sudo useradd -r -s /sbin/nologin -d /var/lib/myapp myapp
getent passwd myapp
```

Why does it have `/sbin/nologin`? What does `-r` do?

### 2. sudo, safely

```bash
sudo visudo -c                                           # check the main file
echo '%webteam ALL=(root) /usr/bin/systemctl restart httpd, /usr/bin/systemctl status httpd' | sudo tee /etc/sudoers.d/webteam
sudo chmod 440 /etc/sudoers.d/webteam
sudo visudo -c -f /etc/sudoers.d/webteam
sudo -l -U alice                                         # what can alice run?
```

Test as `alice` (`su - alice`, then `sudo systemctl restart httpd`, and something she shouldn't be able to run). Explain why this is better than adding her to `wheel`.

### 3. SSH keys

On your own computer (not the VM):

```bash
ssh-keygen -t ed25519 -C "your-name lab key"       # use a passphrase
ssh-copy-id alice@<vm-ip>
ssh alice@<vm-ip>
```

On the VM, look at `~alice/.ssh/authorized_keys` and its permissions. Which permissions must it and `~/.ssh` have?

### 4. Harden `sshd`

```bash
sudo cp /etc/ssh/sshd_config /etc/ssh/sshd_config.bak
sudo grep -rn -E 'PermitRootLogin|PasswordAuthentication' /etc/ssh/sshd_config /etc/ssh/sshd_config.d/
```

Create `/etc/ssh/sshd_config.d/10-lab.conf`:

```
PermitRootLogin no
PasswordAuthentication no
```

```bash
sudo sshd -t                          # syntax check first
sudo systemctl reload sshd
```

**Before closing your first session**, open a second one and confirm key login still works. If you have no working key yet, don't disable passwords. Check the effective setting with `sudo sshd -T | grep -E 'permitrootlogin|passwordauthentication'`. Note the filename order matters in `sshd_config.d` and the first match wins, so check that nothing earlier overrides yours.

### 5. Who did what

```bash
last | head; lastb 2>/dev/null | head
sudo journalctl -u sshd --since "1 hour ago" | grep -i -E 'failed|accepted'
sudo grep sudo /var/log/secure | tail
who; w
```

From another machine, make a few failed login attempts, then find them in the logs.

### 6. Leaver process

Write and then run a leaver checklist for `bob`:

```bash
sudo passwd -l bob
sudo chage -E 0 bob                          # expire the account
sudo pkill -KILL -u bob 2>/dev/null          # end sessions
sudo ls -la ~bob/.ssh 2>/dev/null            # any keys to remove?
sudo find / -xdev -user bob 2>/dev/null | head   # what files does he own?
sudo userdel -r bob                           # remove (after you've decided what to keep)
```

What would you check outside this one server? (Other servers, cloud consoles, VPN, shared mailboxes, API tokens, shared passwords he knew.)

## Break it

1. **Key login refused.** Set `chmod 777 ~/.ssh` for `alice` (or make `authorized_keys` writable by group), and watch key login stop working. Find the reason in `sudo journalctl -u sshd` ("bad ownership or modes"). Fix it with the right permissions.
2. **A bad sudoers file.** In a scratch file (never directly in `/etc/sudoers`), write a syntax error and run `visudo -c -f <file>` to see what it catches. Explain what happens if you edit `/etc/sudoers` with a normal editor and make that mistake.
3. **Locked out.** With a snapshot taken, disable password auth before adding a key and see what the lockout looks like. Practise recovering from the VM console.

## Check your understanding

- What's the difference between authentication and authorization?
- Why do service accounts have `/sbin/nologin`?
- Why edit sudoers with `visudo` and `/etc/sudoers.d/`?
- What are the right permissions for `~/.ssh` and `authorized_keys`, and what happens if they're wrong?
- Why keep a second SSH session open when changing `sshd` config?
- What are the steps when someone leaves the company?
- What is the risk of a shared login?
- Why do we want MFA and a central directory in a larger environment?

## Deliverable

A one-page **joiner/mover/leaver checklist** for Linux servers, plus the `sudoers.d` file and `sshd_config.d` snippet you wrote, with a sentence on why each setting is there.

## Use AI well here

Ask Claude to review your `sshd` hardening and your sudoers rule for mistakes or privilege-escalation paths (for example, `sudo` rules that allow a shell escape). Check each point with `man sshd_config` and `man sudoers` before changing anything.

## Go deeper

- `man sshd_config`, `man sudoers`, `man ssh-keygen`
- Red Hat's RHEL 9 documentation on SSH and on managing users and groups
- Concepts: LDAP, Kerberos, SSSD, `realm`, single sign-on, privileged access management
- CIS Benchmarks for RHEL (a checklist of hardening settings)
