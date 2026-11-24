# Network Infrastructure & Troubleshooting Lab

A hands-on networking lab built on an Ubuntu Server virtual machine and a MacBook as the client. I mapped the network, tested DNS, published a small intranet web service, analyzed the traffic in Wireshark and solved three troubleshooting tickets, documenting the symptom, diagnosis, root cause, fix and verification for each one.

The whole session was recorded with `script`, so every command and result in this repository is real.

## Lab environment

| Item | Details |
|---|---|
| Server | Ubuntu Server 26.04.1 LTS (ARM64), VM in UTM, hostname `ubuntu-network-lab` |
| Client | MacBook Air, connected to the VM through the `bridge100` virtual interface |
| Network | `192.168.64.0/24` · server `192.168.64.2` · gateway `192.168.64.1` |
| Service | nginx serving an "intranet" page on port 80, local name `intranet.lab` |
| Tools | `ip`, `ipcalc`, `arp`, `ss`, `traceroute`, `dig`, `nslookup`, `getent`, `curl`, `nc`, `systemctl`, `journalctl`, `ufw`, Wireshark |

Before starting, I cloned my previous project's VM (Ubuntu Server Security Hardening) in UTM, so that lab stays intact as a backup.

## 1. Network discovery

| Check | Command | Result |
|---|---|---|
| Interfaces | `ip -br a` | `enp0s1` UP, `192.168.64.2/24` |
| Subnet | `ipcalc 192.168.64.2/24` | Network `192.168.64.0`, broadcast `.255`, 254 usable hosts |
| ARP (server) | `ip neigh` | Gateway `192.168.64.1` → `aa:8f:d9:84:00:64` REACHABLE |
| ARP (Mac) | `arp -a` | Server `192.168.64.2` → `26:0e:fc:28:df:37` on `bridge100` |
| Listening ports | `sudo ss -tulpn` | Only SSH (22) on all interfaces; DNS and NTP bound to localhost |
| Path to the internet | `traceroute -n 8.8.8.8` | 10 hops: Mac → home router → ISP → Google, ~8 ms |

## 2. DNS

- `dig google.com`: `NOERROR`, A record `142.251.41.14`, TTL 164, answered by the local resolver `127.0.0.53`.
- `dig +short google.com @8.8.8.8`: a different Google IP, which is normal for large sites served from many locations.
- `dig -x 8.8.8.8 +short`: reverse lookup returns `dns.google.`.
- I added `192.168.64.2 intranet.lab` to `/etc/hosts`. The name resolved locally (`getent`, `dig`, `ping`) but not on `8.8.8.8`, which shows it only exists on this host.

## 3. Intranet service

- I installed nginx and confirmed it was `active`, listening on port 80 and returning `HTTP/1.1 200 OK` locally.
- I replaced the default page with a custom intranet page.

## 4. Traffic analysis (Wireshark)

I captured on `bridge100` while pinging and loading the page from the Mac. The capture shows:

- ICMP echo request and reply
- the TCP three-way handshake (`SYN` → `SYN, ACK` → `ACK`)
- `GET / HTTP/1.1` and `HTTP/1.1 200 OK`
- the connection closing with `FIN, ACK`

## 5. Troubleshooting tickets

| Ticket | Symptom | Diagnosis | Root cause | Fix | Verification |
|---|---|---|---|---|---|
| #1 "The intranet won't open from my computer" | `curl` from the Mac: **connection timed out** | Ping OK, port 22 OK, port 80 no answer; nginx OK on the server | UFW (inherited from my security project) only allowed port 22 | `sudo ufw allow 80/tcp`, a specific rule rather than disabling the firewall | `200 OK` from the Mac, page loads in Safari |
| #2 "It worked this morning, now it errors" (simulated) | **Connection refused** immediately | Wireshark: `SYN` answered by `RST`; `systemctl status`: inactive; `ss`: nothing on port 80; journal: clean stop | nginx service stopped | `sudo systemctl start nginx` | `active`, listening on 80, `200 OK` from the Mac |
| #3 "Works by IP, not by name" (simulated) | By name: cannot connect; by IP: `200 OK` | `getent` → `192.168.64.99`; wrong line in `/etc/hosts`; ping: host unreachable; ARP: `FAILED` | Wrong IP for `intranet.lab` in `/etc/hosts` | Corrected the entry with `sed` | `getent` → `192.168.64.2`, `200 OK` by name |

**Key takeaway:** the error message already tells you where to look.

| Message | Meaning | Where to look |
|---|---|---|
| Connection timed out | No answer, packets dropped | Firewall |
| Connection refused | Host reachable, nothing listening | Service status, `ss` |
| Host unreachable | No device at that IP (ARP fails) | Name resolution, `/etc/hosts`, `ip neigh` |

## Screenshots

All 25 screenshots are in [`screenshots/`](screenshots/), numbered in the order of the work:

- `01`–`05`: updates, subnet, ARP, ports, traceroute
- `06`–`08`: DNS
- `09`–`15`: nginx and ticket #1
- `16`–`17`: Wireshark
- `18`–`25`: tickets #2 and #3

## What I learned

- Troubleshoot one layer at a time: network, then port, then service, then name.
- Always verify from the client, not only from the server.
- Prefer the most precise fix: open one port instead of turning off the firewall.
- Record the session (`script`) so documentation is based on what actually happened.

## Next steps

- Configure a static IP with netplan.
- Add a second client VM.
- Automate the checks in a script.
- Serve the intranet over HTTPS.
