# Module 09: Certificates and TLS

**Time:** about 4 hours.
**Prerequisites:** modules 01, 03, and 04.

## By the end you can

- Explain private key, public key, CSR, certificate, CA, intermediate, and chain of trust.
- Inspect a live certificate and check how long it has left.
- Generate a key and CSR, issue a certificate, and install it on Apache.
- Add a certificate to the system trust store the RHEL way.
- Recognise and explain the common certificate errors.
- Say why certificate renewal must be automated and why inventory is the hard part.

## Why this matters

*Certificate expirations cause preventable outages.* The job is knowing what you have, when it expires, and automating renewal. Lifetimes are getting shorter (the industry is moving toward 90 days and below), so manual renewal is a countdown to an outage.

## Concepts

- **Key pair.** A private key (secret, never shared) and a public key (shared). Anything signed with the private key can be verified with the public one.
- **CSR (Certificate Signing Request).** A request to a CA that includes your public key and the name you want certified.
- **Certificate.** A signed statement: "this public key belongs to this name, valid between these dates."
- **CA (Certificate Authority) and chain of trust.** Your certificate is signed by an intermediate CA, which is signed by a root CA your system already trusts. A server must send the full chain except the root.
- **Where certs live:** web servers, load balancers, VPNs, internal apps, email, code signing, and databases.
- **Internal CA vs public CA.** Public CAs (Let's Encrypt, commercial) are trusted by browsers. Internal CAs are trusted only by systems you configure.
- **SAN (Subject Alternative Name).** The names a certificate is valid for. Modern clients check SAN, not the old common name.
- **Lifetimes and automation.** Short lifetimes force automation (ACME and `certbot`), which removes the "someone forgot" failure.
- **Inventory is the hard part.** Tools don't help if you don't know what exists. Track every certificate, its owner, expiry, and where it's installed.
- **RHEL conventions.** Certs in `/etc/pki/tls/certs/`, keys in `/etc/pki/tls/private/` (root-only), and the system trust store managed by `update-ca-trust`.
- **Protecting private keys.** Anyone with a copy can impersonate your server. Restrict permissions, don't email them, and rotate if exposed.

## Lab

Snapshot first.

### 1. Inspect a live certificate

```bash
echo | openssl s_client -connect example.com:443 -servername example.com 2>/dev/null \
  | openssl x509 -noout -subject -issuer -dates

# Is it expiring in the next 30 days? (exit code 1 means yes)
echo | openssl s_client -connect example.com:443 -servername example.com 2>/dev/null \
  | openssl x509 -noout -checkend 2592000; echo "exit: $?"

# The full chain the server sends
echo | openssl s_client -connect example.com:443 -servername example.com -showcerts 2>/dev/null | grep -E 's:|i:'
```

Read the output: who issued it, when does it expire, and what are the SANs (`openssl x509 -noout -ext subjectAltName`)?

### 2. Generate a key and CSR

```bash
openssl req -new -newkey rsa:2048 -nodes \
  -keyout lab.key -out lab.csr -subj "/CN=lab.example.internal"
openssl req -in lab.csr -noout -text | head -20
```

Note that the key stays with you and only the CSR is sent to a CA.

### 3. Issue a certificate and put it on Apache

No public domain is needed; a self-signed cert is enough to learn the mechanics.

```bash
sudo openssl req -x509 -newkey rsa:2048 -nodes -days 30 \
  -keyout /etc/pki/tls/private/lab.key -out /etc/pki/tls/certs/lab.crt \
  -subj "/CN=lab.example.internal" \
  -addext "subjectAltName=DNS:lab.example.internal"
sudo chmod 600 /etc/pki/tls/private/lab.key
ls -l /etc/pki/tls/private/lab.key /etc/pki/tls/certs/lab.crt
```

Install TLS support:

```bash
sudo dnf install -y mod_ssl
openssl x509 -in /etc/pki/tls/certs/localhost.crt -noout -subject -dates     # the default self-signed cert
```

`mod_ssl` adds `/etc/httpd/conf.d/ssl.conf` with a default HTTPS virtual host. Point `SSLCertificateFile` at `/etc/pki/tls/certs/lab.crt` and `SSLCertificateKeyFile` at `/etc/pki/tls/private/lab.key`, then:

```bash
sudo apachectl configtest
sudo systemctl restart httpd
sudo firewall-cmd --permanent --add-service=https && sudo firewall-cmd --reload
```

### 4. Trust it the RHEL way

```bash
curl https://lab.example.internal --resolve lab.example.internal:443:127.0.0.1    # fails: untrusted
sudo cp /etc/pki/tls/certs/lab.crt /etc/pki/ca-trust/source/anchors/
sudo update-ca-trust
curl https://lab.example.internal --resolve lab.example.internal:443:127.0.0.1    # works
```

Why does it fail before, and work after? What did `update-ca-trust` do?

### 5. Public-domain option

If you own a domain, use certbot from EPEL (`sudo dnf install -y certbot python3-certbot-apache`), then `sudo certbot renew --dry-run`, and look at `systemctl list-timers | grep -i certbot` to see how renewal is scheduled. Package details can vary, so verify against the current EPEL package and Let's Encrypt docs.

### 6. Write a certificate expiry check

Using what you learned in module 05, write a script that takes a hostname and port, checks the days remaining with `openssl s_client` and `-checkend`, and exits non-zero if the cert expires within N days. Then wire it to your monitoring (module 08).

## Break it

1. **Expired.** Create a cert valid for 1 day with `-days 1`, then set the system clock forward (on a snapshot) or use `openssl x509 -checkend` with a large value to see what an expired cert looks like in `curl` and in the browser.
2. **Wrong name.** Browse to the site using a name not in the SAN, and read the error.
3. **Missing intermediate.** Serve only the leaf certificate from a chain (a mentor can supply one). Note that some clients work and others fail, and why.
4. **Key mismatch.** Point Apache at a certificate and a key that don't belong together, and find the error with `configtest` and the logs. Check a pair with `openssl x509 -noout -modulus -in lab.crt | openssl md5` vs the same on the key.

## Check your understanding

- A certificate on a production site expired this morning. What are your first steps, and what do you change afterward?
- How would you track expiry dates across 200 certificates?
- What's the risk if someone gets a copy of a private key?
- Why automate renewal rather than use a calendar reminder?
- Who should be able to read a private key, and how do you enforce that?
- What's the difference between a self-signed certificate and one from a CA?
- What is a SAN, and why do clients check it?

## Deliverable

A **certificate runbook**: how to inspect a certificate, how to renew one on Apache, how to check a key matches a certificate, where keys live and who can read them, and how expiry is monitored. Add your expiry-check script to your git repo.

## Use AI well here

Ask Claude to explain the `openssl x509 -text` output of a real certificate field by field, then ask it to quiz you. Verify details against the `openssl` man pages. Never paste a private key into an AI tool.

## Go deeper

- Let's Encrypt docs and the ACME protocol
- `man openssl-x509`, `man openssl-req`, `man update-ca-trust`
- Cloudflare's "SSL/TLS handshake" explainers
- Certificate Transparency logs, and how to find certificates issued for your domains
