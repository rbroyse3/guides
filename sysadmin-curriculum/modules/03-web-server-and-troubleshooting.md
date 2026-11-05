# Module 03: Web server, firewall, and troubleshooting

**Time:** about 4 hours.
**Prerequisites:** modules 00-01. Module 02 is helpful but not required.

## By the end you can

- Install and configure Apache (`httpd`) and manage it as a service.
- Validate config before restarting, and read the logs when it fails.
- Use `firewalld` to open exactly the service or port you need, with runtime vs permanent config.
- Diagnose a "403 Forbidden" caused by file permissions and fix it with the least privilege that works.
- Work out why something works on `localhost` but not from another machine.

## Why this matters

A web server is a good place to practise the core skill: a service that should work doesn't, and you have to find out why using logs and tools. The same method applies to any service.

## Concepts

- **Apache (`httpd`).** Main config is `/etc/httpd/conf/httpd.conf`; drop-in files go in `/etc/httpd/conf.d/`. Logs are in `/var/log/httpd/`. Workers run as the `apache` user.
- **Validate before you restart:** `apachectl configtest`.
- **Listening vs reachable.** A service can be running and listening but still unreachable because of the firewall, the listening address, or a route.
- **firewalld.** Zones, services, and ports. **Runtime** config applies now and is lost on reload or reboot; **permanent** config survives. You normally apply both.
- **The three checks for "works on localhost, not remotely":** is it listening on the right address, is the firewall open, do the permissions and paths allow it.
- **Least privilege for files.** Give the web server user the access it needs and nobody else more than they need.

## Lab

Snapshot first.

### 1. Install and start

```bash
sudo dnf install -y httpd
sudo systemctl enable --now httpd
systemctl status httpd
ss -tlnp | grep :80
curl -I localhost
sudo tail /var/log/httpd/access_log
```

### 2. Exercise A: break and fix (a typo)

1. Break the config: change `Listen 80` to `Lisen 80` in `/etc/httpd/conf/httpd.conf`.
2. Restart httpd. It fails.
3. Find why using only `systemctl status httpd`, `journalctl -u httpd -e`, and `sudo apachectl configtest`.
4. Fix it, run `configtest` again, restart, and confirm with `curl localhost`.

Take the typo habit with you: always `configtest` before restarting.

### 3. Exercise B: firewalld

1. From your own computer (not the VM), browse to `http://<vm-ip>`. It times out, even though `curl localhost` works on the VM.
2. Find where it's blocked.

```bash
sudo firewall-cmd --state
sudo firewall-cmd --list-all
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --reload
sudo firewall-cmd --list-all
```

Make sure you understand `--permanent` plus `--reload`, and the difference between runtime and permanent config. Try `sudo firewall-cmd --add-service=http` (no `--permanent`), then `--reload`, and watch the rule disappear.

### 4. Homework-style exercise: custom web root and a 403

1. Create a web root only root can read:

```bash
sudo mkdir -p /srv/www
echo hello | sudo tee /srv/www/index.html
sudo chmod 700 /srv/www
```

2. Point Apache at it with `/etc/httpd/conf.d/lab.conf`:

```apache
<VirtualHost *:80>
    DocumentRoot "/srv/www"
    <Directory "/srv/www">
        Require all granted
    </Directory>
</VirtualHost>
```

Run `sudo apachectl configtest`, then `sudo systemctl restart httpd`. (Why is the `<Directory>` block needed? Without `Require all granted`, Apache denies access on its own, a different 403.)

3. Browse to the page. You'll get **403 Forbidden**.
4. Diagnose it with tools, not guesses:

```bash
sudo tail /var/log/httpd/error_log        # look for "(13)Permission denied"
ps aux | grep httpd                       # which user do the workers run as?
namei -l /srv/www/index.html              # permissions on every path component
```

5. Fix it without `chmod 777`:

```bash
sudo chown -R root:apache /srv/www
sudo chmod 750 /srv/www
sudo chmod 640 /srv/www/index.html
```

6. Confirm it works, then explain why this is the least permission that works.

## Break it

1. **Wrong listening address.** Change to `Listen 127.0.0.1:80`, restart. It works on the VM but not from your machine. Find why with `ss -tlnp`, not the firewall.
2. **Both at once.** Have a mentor make two faults at once (for example a firewall rule removed and a permissions problem) and find both, one at a time.
3. **The wrong fix.** Try `chmod 777 /srv/www`, then explain what you gave away and who can now write to your web root.

## Check your understanding

- A user says "the website is down." What are your first five checks?
- What's the difference between a process, a service, and a port?
- What does `--permanent` do, and why does it usually need a `--reload`?
- How do you tell "not listening" from "listening but blocked"?
- Why is `chmod 777` the wrong fix, and why is stopping firewalld the wrong fix?
- What does the `(13)Permission denied` line in `error_log` tell you?

## Deliverable

A one-page note: what caused the 403, how you found it (list the commands), what you changed, and why that is the least permission that works.

## Use AI well here

After solving the 403 yourself, ask Claude to review your fix and explain the `namei -l` output. Compare its explanation with your own notes. Then ask: "What else could cause a 403 on Apache that isn't file permissions?" and check each answer against the Apache docs.

## Go deeper

- Apache HTTP Server docs (httpd.apache.org/docs), especially "Require" and "DocumentRoot"
- Red Hat's RHEL 9 documentation on firewalld
- `man firewall-cmd`, `man namei`
