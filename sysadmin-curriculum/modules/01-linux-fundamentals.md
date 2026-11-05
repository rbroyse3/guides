# Module 01: Linux fundamentals

**Time:** about 5 hours, including about 30 minutes for `vimtutor`.
**Prerequisites:** a working lab VM ([LAB-SETUP.md](../LAB-SETUP.md)), module 00.

## By the end you can

- Navigate the filesystem and explain what the main directories are for.
- Create users and groups and set permissions correctly.
- Install, query, and remove packages with `dnf` and `rpm`.
- Start, stop, enable, and inspect services with `systemctl`, and write a simple unit.
- Read logs with `journalctl` and from `/var/log`.
- Inspect and control processes.
- Check and set the time.
- Edit files comfortably in vim, and keep work alive across a dropped connection with tmux.

Storage (partitions, filesystems, mounting, LVM) has its own module, [02](02-storage-and-filesystems.md).

## Why this matters

Everything else in the role sits on top of this. When something breaks, you will be reading logs, checking services, and looking at permissions long before you touch anything fancy.

## Concepts

- **Filesystem layout.** `/etc` (config), `/var/log` (logs), `/var/lib` (service data), `/home` (users), `/opt` (add-on software), `/tmp` (temporary), `/srv` (data a server serves), `/usr/bin` and `/usr/sbin` (programs).
- **Everything is a file**, including devices and process info (`/dev`, `/proc`).
- **Users and groups.** `root` is UID 0. On RHEL-family systems, the `wheel` group grants `sudo`.
- **Permissions.** Owner, group, other; read, write, execute. `rwxr-x---` is `750`.
- **Packages.** `dnf` resolves dependencies and installs from repositories; `rpm` queries the installed database.
- **Services** are managed by `systemd`. A **unit** describes a service; a **timer** schedules it.
- **Logs.** `journald` collects service logs (read with `journalctl`); files in `/var/log` hold others.
- **Processes** can be sent **signals**. `SIGTERM` asks nicely; `SIGKILL` (`kill -9`) can't be caught and gives the process no chance to clean up.

## Lab

Snapshot first. Use `sudo` where needed.

### 1. Orientation

```bash
whoami; hostname; uname -a; cat /etc/os-release
pwd; ls -la /; ls -la /etc | head
df -h; free -h; uptime
```

For each of `/etc`, `/var/log`, `/home`, `/opt`, `/tmp`, `/srv`: say what it's for before you look it up.

### 2. Users and permissions

```bash
sudo useradd testuser
sudo passwd testuser
sudo usermod -aG wheel testuser
id testuser
ls -l /etc/passwd /etc/shadow
touch somefile; chmod 640 somefile; sudo chown testuser:testuser somefile
ls -l somefile; stat somefile
```

Answer: what does `rwxr-x---` mean? What is the difference between `chmod 755` and `chmod 700`? Why can anyone read `/etc/passwd` but not `/etc/shadow`?

### 3. Packages

```bash
sudo dnf install -y tree
dnf info tree; dnf provides /usr/bin/tree
rpm -qi tree; rpm -ql tree | head
dnf history
sudo dnf remove -y tree
```

### 4. Services

Use `chronyd` (time sync) or `crond`, not `sshd`. Stopping `sshd` over an SSH session can lock you out.

```bash
systemctl status chronyd
sudo systemctl stop chronyd; systemctl is-active chronyd
sudo systemctl start chronyd
systemctl is-enabled chronyd
journalctl -u chronyd -e
systemctl list-units --failed
```

Explain the difference between `start` and `enable`.

### 5. Write your own service and timer

```bash
sudo tee /usr/local/bin/hello.sh > /dev/null <<'EOF'
#!/bin/bash
echo "hello from $(hostname) at $(date)"
EOF
sudo chmod 755 /usr/local/bin/hello.sh

sudo tee /etc/systemd/system/hello.service > /dev/null <<'EOF'
[Unit]
Description=Hello service

[Service]
Type=oneshot
ExecStart=/usr/local/bin/hello.sh
EOF

sudo tee /etc/systemd/system/hello.timer > /dev/null <<'EOF'
[Unit]
Description=Run hello every minute

[Timer]
OnCalendar=minutely

[Install]
WantedBy=timers.target
EOF

sudo systemctl daemon-reload
sudo systemctl start hello.service
journalctl -u hello.service -e
sudo systemctl enable --now hello.timer
systemctl list-timers | grep hello
```

Why `daemon-reload`? What's the difference between a cron job and a timer?

### 6. Logs and searching

```bash
journalctl -p err -b                 # errors since boot
journalctl -u sshd --since "1 hour ago"
sudo grep -ri "error" /var/log/ 2>/dev/null | head
ps aux | grep -i chron
```

### 7. Processes and signals

```bash
sleep 600 &
jobs; ps aux | grep "sleep 600"
kill %1                               # SIGTERM
sleep 600 &
kill -9 %1                            # SIGKILL: what's different?
top                                   # press q to quit; find the highest CPU process
nice -n 10 sleep 600 &                # lower priority
```

### 8. Time

```bash
timedatectl
chronyc tracking
```

Why does the wrong time matter? (Certificates, logs, Kerberos, scheduled jobs.)

### 9. Editors and tmux

You'll edit config on servers where vim is the editor you can count on, and you'll run long jobs over SSH connections that drop. Learn both now.

```bash
sudo dnf install -y vim-enhanced tmux
vimtutor                              # about 30 minutes; do it once, properly
export EDITOR=vim                      # used by visudo, crontab -e, and others
```

Vim minimum: `i` to insert, `Esc` to leave insert mode, `:w` save, `:q` quit, `:wq` save and quit, `:q!` quit without saving, `/text` search (`n` for next), `:%s/old/new/g` replace, `dd` delete a line, `yy` copy a line, `p` paste, `u` undo, `gg` and `G` for top and bottom, `:42` to go to line 42.

tmux minimum:

```bash
tmux new -s lab                        # start a named session
# inside: Ctrl-b d  detaches, Ctrl-b c new window, Ctrl-b %  split, Ctrl-b "  split the other way
tmux ls
tmux attach -t lab
```

Exercise: start `sleep 600` inside a tmux session, detach, close your SSH connection completely, reconnect, and reattach. Then do the same without tmux and see what happens to the job.

## Break it

Ask your mentor to set up the first one without telling you what it is. The other two you can do on your own:

1. **A broken unit.** Introduce a typo into `hello.service` (for example `ExecStar=`), reload, and start it. Find out why using only `systemctl status`, `journalctl -u hello.service`, and `systemd-analyze verify /etc/systemd/system/hello.service`.
2. **Permission denied.** Create a file as `testuser`, mode `600`, and try to read it as another user. Explain the error, and fix it with the least change.
3. **A runaway process.** Run `yes > /dev/null &`, find it with `top`, and stop it.

## Check your understanding

- What are the first three things you check when a service won't start?
- What's the difference between `systemctl stop`, `disable`, and `mask`?
- What does `dnf provides` do and when is it useful?
- A file is `-rw-r-----  root  apache`. Who can read it? Who can write it?
- What's the difference between `kill` and `kill -9`, and why prefer the first?
- Why does `sudo` beat logging in as root?
- How do you save and quit vim, quit without saving, and search for a word?
- What does tmux give you that a plain SSH session doesn't?

## Deliverable

A one-page note, "How I diagnose a service that won't start," using your own commands and the unit you broke in the lab. Commit it to your notes repo if you have one (module 05).

## Use AI well here

After you've fixed the broken unit yourself, ask Claude to explain the `journalctl` output line by line, then ask: "What other mistakes could a unit file have that would fail in the same way?" Check anything new against `man systemd.service`.

## Go deeper

- *The Linux Command Line* by William Shotts (free book)
- `man systemd.unit`, `man systemd.service`, `man systemd.timer`
- Red Hat's RHEL 9 documentation on managing services with systemd
- linuxjourney.com
