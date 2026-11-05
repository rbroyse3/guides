# Module 08: Monitoring and alerting

**Time:** about 4 hours.
**Prerequisites:** modules 01, 03, and 04.

## By the end you can

- Say what to monitor on any Linux server, and the difference between up/down checks and resource monitoring.
- Stand up a monitoring tool and add checks for a web page, a TCP port, and disk space.
- Design an alert that is actionable: it has a threshold, a duration, an owner, and a next step.
- Explain alert fatigue and tune a noisy alert.
- Trigger an alert on purpose and watch it fire and resolve.

## Why this matters

*Monitoring exists so you find out before the users do.* Too many alerts are as bad as none, because people stop reading them. The skill is choosing what's worth alerting on, not collecting every metric.

## Concepts

- **Metrics, logs, and traces.** Metrics are numbers over time (CPU, disk), logs are events, traces follow a request across services. You mostly need metrics and logs.
- **Up/down checks vs resource monitoring.** "Is the website answering?" vs "is the disk 90% full?" You need both. Check from the user's point of view, not only from the server.
- **Thresholds and duration.** Alert on "disk over 85% for 10 minutes," not "disk over 85% right now." A short spike isn't an incident.
- **Severity levels.** Warning (look at it in business hours) vs critical (wake someone up). If nothing is critical, nothing is.
- **Actionable alerts.** Every alert has an owner and a next step. If it's ignored, tune it or delete it.
- **Alert fatigue.** Constant noise trains people to ignore alerts, including the real one. It's a safety problem.
- **Dead-man's switch (heartbeat).** Instead of alerting on a failure, alert on the *absence* of a success signal. This catches the monitor or the job itself dying silently.
- **Alert routing.** Who gets notified, how (chat, email, pager), and when. Escalate if nobody acknowledges.
- **Alerting on symptoms vs causes.** Users care that the site is slow; "CPU is 80%" is only useful if it explains something.
- **Agents vs agentless.** Some tools install an agent on each host; others poll from outside. Know which you're using, and which ports it needs open.

## Lab

Snapshot first. This lab uses Uptime Kuma, a lightweight tool that runs in Docker. If your team uses a different tool (Zabbix, Checkmk, Prometheus, a commercial product), the concepts carry over, and a mentor can substitute it. Prometheus with `node_exporter` and Grafana is the industry-standard open stack and makes a good stretch goal.

### 1. Install Docker

Docker isn't in Rocky's default repos, so install Docker CE from Docker's repo. (Docker's repo for RHEL-family distros is the `centos` one, which Rocky uses.)

```bash
sudo dnf install -y dnf-plugins-core
sudo dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
sudo dnf install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
sudo systemctl enable --now docker
sudo docker run -d --restart=always -p 3001:3001 \
  -v uptime-kuma:/app/data --name uptime-kuma louislam/uptime-kuma:1
sudo docker ps; sudo docker logs uptime-kuma
```

Check Uptime Kuma's current docs for the right image tag. `--restart=always` plus Docker enabled at boot means it comes back after a reboot; **prove it** by rebooting.

**Gotcha worth understanding:** Docker publishes ports by writing its own firewall rules, which typically bypass firewalld's normal port rules. So `-p 3001:3001` can be reachable from the network even if you never opened 3001 in firewalld. Check from another machine and compare with `sudo firewall-cmd --list-all`. That's a real-world surprise and a good "published ports and firewalls" security conversation. (RHEL-native shops often use Podman instead, which is mostly command-compatible.)

### 2. Add checks

Open `http://<vm-ip>:3001`, create an admin account, and add:

1. An **HTTP** check against the Apache page from module 03.
2. A **TCP port** check on SSH (22).
3. A **notification** (email, a Slack or Teams webhook, or Discord for the lab).

Now stop httpd and watch the alert fire. Start it again and watch it resolve. How long did each take, and why?

### 3. Tune the check

Change the retry count and interval. Make the check require 3 failures before alerting. What do you gain, and what do you risk?

### 4. A disk-space alert with a push monitor

Uptime Kuma can't read disk usage on its own, so use a **push monitor** (a good lesson in dead-man's-switch alerting).

1. Add a monitor of type **Push** with a 60-second heartbeat interval. Copy its push URL/token.
2. Run your `disk-report.sh` (from module 05), or this script, from a systemd timer or cron every minute. It only reports "up" while usage is under the threshold:

```bash
#!/bin/bash
USED=$(df --output=pcent / | tail -1 | tr -dc '0-9')
if [ "$USED" -lt 85 ]; then
  curl -fsS "http://localhost:3001/api/push/<TOKEN>?status=up&msg=OK&ping="
fi
```

3. Fill the disk past the threshold, confirm the alert fires, then remove the file and confirm it resolves:

```bash
sudo fallocate -l 5G /var/fill.img      # works on XFS; size it to push usage over the threshold
df -h /
sudo rm /var/fill.img
```

Notice that the alert fires because the heartbeat *stops*, not because a "bad" message arrives. Why is that useful?

### 5. Stretch: Prometheus and Grafana

Install `node_exporter`, scrape it with Prometheus, and graph disk usage in Grafana. Write one alert rule with a `for:` duration. It takes longer than a lab session, but it teaches the metrics model well.

## Break it

1. **A flapping check.** Make a service restart every few minutes (a loop that stops and starts httpd, on a snapshot) and see what a noisy alert looks like. Tune it so the right person is told only once.
2. **A silent monitor.** Stop the Uptime Kuma container. Who would know? What would you add to monitor the monitor?
3. **Wrong threshold.** Set the disk alert at 50% when the disk is already 55% full. It fires immediately and constantly. What should have been checked before enabling it?

## Check your understanding

- Name three things you'd monitor on any Linux server.
- What's the difference between a check failing once and failing consistently?
- A CPU alert fires at 3 AM, goes away on its own, and happens twice a week. What do you do?
- Who gets paged, and when is something "worth waking someone up"?
- Why alert on the absence of a heartbeat?
- What is alert fatigue, and why does it matter?
- Why can a Docker-published port be reachable through firewalld?

## Deliverable

A short **alerting plan** for the lab web server: what you monitor, thresholds with durations, severity, who is notified, and the first action for each alert. Include a screenshot or log of an alert firing and resolving.

## Use AI well here

After you've built the disk alert yourself, ask Claude how you'd tune it to cut noise (duration, thresholds, notifications). Decide which suggestions are valid and explain why. If you ask it to draft a check script, read and explain **every line**, and test it in the lab before you trust it.

## Go deeper

- Uptime Kuma, Prometheus, and Grafana docs
- Zabbix and Checkmk docs, if those are tools you'll use
- Google SRE book, chapter on monitoring distributed systems (the "four golden signals")
- Alerting philosophy: symptoms over causes, and write alerts you'd be happy to be woken for
