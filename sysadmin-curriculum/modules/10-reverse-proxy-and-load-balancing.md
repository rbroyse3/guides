# Module 10: Reverse proxy and load balancing

**Time:** about 3 hours.
**Prerequisites:** modules 03, 04, and 09.

## By the end you can

- Explain reverse proxy vs load balancer vs forward proxy, and layer 4 vs layer 7.
- Put HAProxy in front of two backends with round-robin balancing and health checks.
- Test failover and read the stats page.
- Explain why backends see the proxy's IP and what `X-Forwarded-For` is for.
- Terminate TLS on the proxy, and say where certificate renewal now happens.

## Why this matters

*A proxy in front of your apps gives you one front door, health checks, failover, and one place to manage certificates.* It's also where a lot of production outages and subtle bugs live, so knowing how it works pays off.

## Concepts

- **Reverse proxy** sits in front of servers and forwards client requests to them. **Forward proxy** sits in front of clients. **Load balancer** is a reverse proxy that spreads requests over several backends.
- **Layer 4 (TCP) vs layer 7 (HTTP).** Layer 4 moves bytes without understanding them. Layer 7 understands HTTP, so it can route by hostname or path and add headers. This lab uses layer 7 (`mode http`).
- **HAProxy's building blocks:** a **frontend** (what it listens on), a **backend** (a pool of servers), and **servers** (the members).
- **Balancing algorithms:** `roundrobin` rotates; `leastconn` prefers the least busy server.
- **Health checks** remove a failed backend from rotation and add it back when it recovers.
- **Client IP.** The backend sees the proxy's address. The real client IP travels in the `X-Forwarded-For` header.
- **TLS termination.** The proxy handles HTTPS and talks plain HTTP to the backends. One place to manage certificates, but traffic behind the proxy is unencrypted unless you re-encrypt.
- **New single point of failure.** One proxy is now the failure point. Production pairs them (for example with keepalived/VRRP). That's a pointer, not a lab.

## Lab

Snapshot first. This runs on one VM; a second VM can host a backend as a stretch goal.

### 1. Two Apache backends on local-only ports

```bash
sudo mkdir -p /srv/app-a /srv/app-b
echo "backend A" | sudo tee /srv/app-a/index.html
echo "backend B" | sudo tee /srv/app-b/index.html
```

`/etc/httpd/conf.d/backends.conf`:

```apache
Listen 127.0.0.1:8081
Listen 127.0.0.1:8082
<VirtualHost 127.0.0.1:8081>
    DocumentRoot "/srv/app-a"
    <Directory "/srv/app-a">
        Require all granted
    </Directory>
</VirtualHost>
<VirtualHost 127.0.0.1:8082>
    DocumentRoot "/srv/app-b"
    <Directory "/srv/app-b">
        Require all granted
    </Directory>
</VirtualHost>
```

```bash
sudo apachectl configtest && sudo systemctl restart httpd
curl http://127.0.0.1:8081; curl http://127.0.0.1:8082
```

### 2. Free up port 80: a deliberate "address already in use" lesson

HAProxy will want port 80, which Apache has. Install HAProxy and let it fail:

```bash
sudo dnf install -y haproxy
sudo cp /etc/haproxy/haproxy.cfg /etc/haproxy/haproxy.cfg.orig        # always keep a backup
sudo systemctl start haproxy
```

Diagnose the failure with `ss -tlnp | grep :80` and `journalctl -u haproxy -e`. Then fix it: comment out `Listen 80` in `/etc/httpd/conf/httpd.conf`, and move `/etc/httpd/conf.d/lab.conf` (and `ssl.conf` if present) aside, for example `sudo mv lab.conf lab.conf.disabled`. Restart httpd.

### 3. Configure HAProxy

Keep the shipped `global` section. Replace everything from `defaults` onward with:

```
defaults
    mode http
    log global
    option httplog
    option forwardfor
    timeout connect 5s
    timeout client  30s
    timeout server  30s

frontend web
    bind *:80
    default_backend apps

backend apps
    balance roundrobin
    option httpchk GET /
    server a 127.0.0.1:8081 check
    server b 127.0.0.1:8082 check

listen stats
    bind *:8404
    stats enable
    stats uri /stats
```

```bash
sudo haproxy -c -f /etc/haproxy/haproxy.cfg           # validate before restarting
sudo systemctl enable --now haproxy
sudo firewall-cmd --permanent --add-service=http --add-port=8404/tcp && sudo firewall-cmd --reload
for i in 1 2 3 4; do curl -s http://localhost; done    # should alternate A, B, A, B
```

If HAProxy request logs aren't where you expect, check `journalctl -u haproxy` and the `log` line in the shipped `global` section. How logging is wired depends on the rsyslog setup.

### 4. Failover test

1. Browse to `http://<vm-ip>:8404/stats` and find both servers shown as UP.
2. Break backend B: `sudo mv /srv/app-b/index.html /srv/app-b/index.html.off`. Apache now returns 404 on `/`, so the health check fails.
3. Watch B go DOWN on the stats page. Run the `curl` loop again: only "backend A" comes back, and the user never saw an error.
4. Restore the file and watch B come back UP.

### 5. The client IP

Which IP does Apache log for requests that came through HAProxy? Where does the real client IP go? Apache can log `%{X-Forwarded-For}i`; change the `LogFormat` for the backends, restart, and prove it.

### 6. Stretch: terminate TLS on HAProxy

1. Make sure Apache's `ssl.conf` isn't listening on 443 (move it aside).
2. Build the combined PEM HAProxy expects:

```bash
sudo mkdir -p /etc/haproxy/certs
cat /etc/pki/tls/certs/lab.crt /etc/pki/tls/private/lab.key | sudo tee /etc/haproxy/certs/lab.pem > /dev/null
sudo chmod 600 /etc/haproxy/certs/lab.pem
```

3. Add `bind *:443 ssl crt /etc/haproxy/certs/lab.pem` to the `frontend web` block, open `https` in firewalld, then validate and reload.
4. Test: `curl https://lab.example.internal --resolve lab.example.internal:443:127.0.0.1`.
5. Write down: where would certificate renewal now happen, and what must you do on renewal day?

## Break it

1. **Both backends down.** What does the user see? What does the stats page show?
2. **A health check that lies.** Change `httpchk` to check a path that always returns 200 even when the app is broken. What would that miss? (App up but database down.)
3. **Config typo.** Break `haproxy.cfg`, run `haproxy -c -f`, and see what it catches before you reload.
4. **Slow backend.** Add a delay on one backend and compare `roundrobin` with `leastconn`.

## Check your understanding

- What did the user experience when B went down? What would it have looked like without HAProxy?
- What does the health check actually test, and what could it miss?
- Now there's one proxy. What's the new single point of failure?
- Which IP does Apache log for the client, and where does the real IP go?
- Which is better for this app, `roundrobin` or `leastconn`, and why?
- What's the difference between layer 4 and layer 7 proxying?
- If TLS ends at the proxy, what's the security trade-off?

## Deliverable

A one-page diagram and note: **how traffic flows** from a client to a backend through your proxy, what is checked, what fails over, and what doesn't. Include your `haproxy.cfg` and the failover test results.

## Use AI well here

Paste a sanitized `haproxy.cfg` and ask Claude for a review. Verify each suggestion with `haproxy -c` and the docs before applying it. Ask it what the single biggest remaining risk is in your setup, and decide whether you agree.

## Go deeper

- HAProxy docs (docs.haproxy.org), especially the configuration manual's sections on `frontend`, `backend`, and health checks
- keepalived and VRRP for proxy redundancy
- Layer 4 load balancing, SNI routing, and HTTP/2
- nginx as a reverse proxy, to compare
