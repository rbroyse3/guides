# Module 04: Networking and DNS

**Time:** about 5 hours.
**Prerequisites:** modules 01 and 03. A second VM helps but isn't required.

## By the end you can

- Explain what happens, step by step, when you type a URL into a browser.
- Read an IP address and a CIDR block, and say which addresses are in the same subnet.
- Show a host's address, gateway, routes, and DNS settings, and say what each does.
- Query DNS directly and explain the common record types.
- Tell apart "DNS is wrong," "the route is wrong," "the port is closed," and "the firewall is blocking it."
- Capture traffic with `tcpdump` to see what's really on the wire.

## Why this matters

"It's the network" and "it's DNS" are the most common first guesses, and a sysadmin has to be able to prove or disprove them in minutes. Almost every other module depends on this: monitoring, certificates, load balancers, and backups all fail in networking ways.

## Concepts

### The practical model

Think in four layers and work from the bottom up when troubleshooting:

1. **Link:** is the interface up, does it have the right address? (`ip addr`)
2. **Network (IP):** can I reach the gateway, and from there the other host? (`ip route`, `ping`)
3. **Transport (TCP/UDP):** is the port open and is something listening? (`ss`, `nc`)
4. **Application:** does the service answer correctly? (`curl -v`)

### Addresses and subnets

- An **IPv4 address** is four numbers 0-255 (`192.168.1.25`). A **subnet mask** or **CIDR prefix** says which part is the network: `/24` = `255.255.255.0` = 256 addresses (254 usable hosts).
- Hosts in the same subnet talk directly. Anything else goes via the **default gateway**.
- **Private ranges:** `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`. These aren't routed on the public internet; **NAT** lets them share a public address.
- **ARP** maps an IP on the local network to a MAC address.
- Quick maths: `/24` = 256 addresses, `/25` = 128, `/26` = 64, `/27` = 32, `/28` = 16, `/30` = 4.

### Ports and protocols

- A **port** identifies a service on a host: `22` SSH, `80` HTTP, `443` HTTPS, `53` DNS, `25` SMTP.
- **TCP** is connection-based and reliable; **UDP** is connectionless and fast (DNS queries, many monitoring protocols).
- **Listening** means a process is waiting on a port. A listener on `127.0.0.1` is reachable only from the same host.

### DNS

DNS turns names into addresses (and more).

- **Record types:** `A` (name to IPv4), `AAAA` (name to IPv6), `CNAME` (alias to another name), `MX` (mail), `TXT` (arbitrary text, used for email auth and domain verification), `NS` (which servers are authoritative), `PTR` (address to name, "reverse").
- **Resolution path on a Linux host:** the application asks the resolver library, which consults `/etc/nsswitch.conf` (usually `files` then `dns`), so `/etc/hosts` is checked first, then the DNS servers in `/etc/resolv.conf`. On Rocky 9, NetworkManager manages `resolv.conf`.
- **TTL** is how long an answer can be cached. A low TTL makes changes take effect fast; a high TTL makes a bad record linger.
- **Recursive resolver** vs **authoritative server:** your resolver does the legwork; authoritative servers hold the truth for a zone.

## Lab

Snapshot first.

```bash
sudo dnf install -y bind-utils traceroute nmap-ncat tcpdump
```

### 1. My own network

```bash
ip -br addr                          # interfaces and addresses
ip route                             # routes, including the default gateway
ip neigh                             # ARP table
nmcli device status; nmcli connection show
cat /etc/resolv.conf
cat /etc/nsswitch.conf | grep hosts
hostname; hostname -I
```

Write down: your IP, prefix length, gateway, DNS servers. Which addresses are in your subnet? How many hosts could it hold?

### 2. Subnet exercises (paper)

For each, say the network address, broadcast address, and number of usable hosts, then check with `ipcalc`:

- `192.168.10.37/24`
- `10.1.4.130/26`
- `172.16.5.9/20`

```bash
ipcalc -n -b -m 10.1.4.130/26
```

Which of `10.1.4.100`, `10.1.4.190`, and `10.1.4.130` is in the same subnet as `10.1.4.130/26`?

### 3. Walk the layers

```bash
ping -c 3 <your-gateway>             # layer 3: gateway reachable?
ping -c 3 1.1.1.1                    # beyond the gateway, no DNS involved
ping -c 3 example.com                # DNS plus reachability
traceroute example.com               # where does it stop, if it does?
nc -zv example.com 443               # layer 4: is the port reachable?
curl -v https://example.com -o /dev/null    # layer 7: the full conversation
```

If `ping 1.1.1.1` works and `ping example.com` fails, what does that tell you? What if `ping` works but `curl` fails?

### 4. DNS up close

```bash
getent hosts example.com             # what the OS resolver returns (honours /etc/hosts)
dig example.com                      # full answer, with TTL
dig example.com +short
dig example.com MX +short
dig example.com NS +short
dig -x 1.1.1.1 +short                # reverse lookup
dig @1.1.1.1 example.com +short      # ask a specific server
dig example.com +trace               # follow delegation from the root
```

Run `dig example.com` twice and watch the TTL count down on the second run. What does that tell you about caching?

### 5. See it on the wire

In one terminal:

```bash
sudo tcpdump -nn -i any port 53
```

In another:

```bash
dig example.com +short
curl -s https://example.com -o /dev/null
```

Identify the DNS query and response. Then capture HTTP: `sudo tcpdump -nn -i any -A port 80` while running `curl http://localhost` and see the request in plain text. What does that tell you about why TLS matters?

### 6. Run your own tiny DNS server

This is self-contained and uses an unprivileged port so it doesn't interfere with the system resolver.

```bash
sudo dnf install -y dnsmasq
sudo dnsmasq --no-daemon --port=5353 --address=/lab.test/10.0.0.5 &
dig @127.0.0.1 -p 5353 lab.test +short          # 10.0.0.5
sudo kill %1
```

What did `--address` do? How is this different from adding a line to `/etc/hosts`?

### 7. Optional: two VMs

With a second VM, ping between them, open a port with `nc -l 8080` on one and connect with `nc <ip> 8080` from the other, then block it with `firewall-cmd` and observe the difference between "connection refused" (nothing listening) and a timeout (filtered).

## Break it

Your mentor will inject faults for you to find. Good ones for this module:

1. **A bad `/etc/hosts` entry.** `getent hosts` and `dig` disagree. Find why.
2. **Wrong DNS server.** Set a bad server for the connection (`nmcli connection modify <name> ipv4.dns 192.0.2.1`, `nmcli connection up <name>`). Names stop resolving but IPs work. Find it and restore it.
3. **Port blocked vs not listening.** Show the difference using `nc -zv` against a closed port and a firewalled port: refused vs timed out.

## Check your understanding

- What happens, step by step, when you type `https://example.com` in a browser?
- What's the difference between a hostname, an IP address, and a port?
- You can ping an IP but not a name. What's wrong, and how do you confirm it?
- What does "connection refused" mean compared with "connection timed out"?
- Why is a listener on `127.0.0.1` unreachable from elsewhere?
- What is the default gateway for?
- What does a low TTL do?
- What does `/etc/nsswitch.conf` control?

## Deliverable

A one-page **"Web server unreachable"** checklist, in your own words, that works through the layers with the exact command for each step and what each result means. Test it against a fault your mentor injects.

## Use AI well here

Ask Claude to explain a `tcpdump` capture line by line, then ask it to give you a broken-network scenario (symptoms only) and diagnose it out loud before asking for the answer. Verify any DNS or networking claims with `man dig` and the RHEL docs.

## Go deeper

- Julia Evans' zines on networking and DNS (wizardzines.com)
- `man ip`, `man ss`, `man dig`, `man nmcli`, `man tcpdump`
- Red Hat's RHEL 9 documentation: *Configuring and managing networking*
- Subnetting practice sites; a few minutes a day makes `/24` through `/28` automatic
