# Lab: Setting Up an Authoritative DNS Server with BIND

## Overview

DNS turns domain names into IP addresses. Without it, you'd be typing IPs into a browser instead of names, and nobody wants that. This lab covers building an authoritative DNS server from scratch using BIND (Berkeley Internet Name Domain) on a RHEL/CentOS-based system, including forward and reverse zones, zone file syntax, and the resource record types that make it all work.

The goal here isn't just "make `nslookup` return an answer." It's understanding what's actually happening when a client resolves a name, why zone files are structured the way they are, and how to troubleshoot the server when something breaks.

## Background: How DNS Resolution Works

DNS stores name-to-address mappings in a distributed database. Three record types cover most of the basics:

- **A record** — hostname to IP address (`clienta` → `192.168.100.240`)
- **PTR record** — IP address to hostname (`192.168.100.240` → `clienta`)
- **CNAME record** — hostname to hostname (an alias)

When a client requests a name, it queries a name server (usually over UDP port 53). That server checks its own resolver library for an authoritative answer or a cached one. If it has neither, it queries the root name servers to find who's authoritative for the domain, then queries that server directly for the answer.

BIND stores this data in **resource records (RR)**, organized under a fully qualified domain name (FQDN) in a tree-like hierarchy. BIND itself ships three main pieces: `named` (the actual name server daemon), `rndc` (an administration utility), and `dig` (a query/debugging tool).

Configuration lives in two places:

| Path | Purpose |
|---|---|
| `/etc/named.conf` | Main configuration file |
| `/etc/named/` | Auxiliary directory for included configuration files |

| | |
|---|---|
| Service/process name | `named` |
| Package name | `bind` |

## Lab Environment

- Domain: `lab.local`
- DNS server IP: `192.168.100.153` (on `enp0s3`)
- Client A: `192.168.100.240`
- Client B: `192.168.100.241`

## Step 1: Install the DNS Packages

```bash
dnf install bind bind-utils -y
```

## Step 2: Configure the Main BIND Config File

```bash
vi /etc/named.conf
```

Find this line:

```
listen-on port 53 { 127.0.0.1; };
```

and change it to include your server's IP:

```
listen-on port 53 { 127.0.0.1; 192.168.100.153; };
```

Then, before the `include` lines near the bottom of the file, add the forward and reverse zone definitions:

```
zone "lab.local" IN {
    type master;
    file "forward.lab";
    allow-update { none; };
};

zone "100.168.192.in-addr.arpa" IN {
    type master;
    file "reverse.lab";
    allow-update { none; };
};

include "/etc/named.rfc1912.zones";
include "/etc/named.root.key";
```

Save and quit.

> The reverse zone name is built from the network portion of the IP address, reversed. For `192.168.100.0/24`, that's `100.168.192.in-addr.arpa`.

## Step 3: Create the Zone Files

```bash
cd /var/named
touch forward.lab reverse.lab
```

**forward.lab:**

```
$TTL 86400
@   IN SOA masterdns.lab.local. root.lab.local. (
                2011071001 ; Serial
                3600       ; Refresh
                1800       ; Retry
                604800     ; Expire
                86400 )    ; Minimum TTL

@         IN NS  masterdns.lab.local.
@         IN A   192.168.100.153
masterdns IN A   192.168.100.153
clienta   IN A   192.168.100.240
clientb   IN A   192.168.100.241
```

**reverse.lab:**

```
$TTL 86400
@   IN SOA masterdns.lab.local. root.lab.local. (
                2011071001 ; Serial
                3600       ; Refresh
                1800       ; Retry
                604800     ; Expire
                86400 )    ; Minimum TTL

@         IN NS  masterdns.lab.local.
153       IN PTR masterdns.lab.local.
240       IN PTR clienta.lab.local.
241       IN PTR clientb.lab.local.
```

## Step 4: Start and Enable the DNS Server

```bash
systemctl start named
systemctl enable named
```

## Step 5: Disable firewalld

```bash
systemctl stop firewalld
systemctl disable firewalld
```

> Fine for an isolated lab. On anything production-facing, open UDP/TCP 53 instead of disabling the firewall outright.

## Step 6: Fix Permissions, Ownership, and SELinux Context

```bash
chgrp -R named /var/named
chown -v root:named /etc/named.conf
restorecon -rv /var/named
restorecon /etc/named.conf
```

BIND is picky about ownership and SELinux context on its zone files. Skip this step and you'll get permission-denied errors in `/var/log/messages` even though the config file looks correct.

## Step 7: Validate the Configuration and Zone Files

```bash
named-checkconf /etc/named.conf
named-checkzone lab.local /var/named/forward.lab
named-checkzone lab.local /var/named/reverse.lab
```

Run this before restarting `named` after any zone file edit. A typo in a zone file will silently break resolution for that zone, and `named-checkzone` catches it before you go looking for the problem somewhere else.

## Step 8: Point the Network Interface at the New DNS Server

```bash
vi /etc/sysconfig/network-scripts/ifcfg-enp0s3
```

Add or update:

```
DNS=192.168.100.153
```

## Step 9: Restart the Network Service

```bash
systemctl restart NetworkManager
```

## Step 10: Confirm /etc/resolv.conf

```bash
nameserver 192.168.100.153
```

NetworkManager should populate this automatically after Step 9, but check it if resolution still isn't working.

## Step 11: Test the DNS Server

```bash
dig masterdns.lab.local
nslookup masterdns.lab.local
nslookup clienta.lab.local
nslookup clientb.lab.local
nslookup 192.168.100.240
nslookup 192.168.100.241
```

## Managing the Service with rndc

`rndc` lets you administer `named` locally or remotely without restarting the whole service:

```bash
rndc status                # current service status
rndc reload                # reload config + all zones (keeps cached data)
rndc reload lab.local      # reload a single zone
rndc reconfig              # reload config + newly added zones only
```

Its own config lives at `/etc/rndc.conf`. If that file doesn't exist, `rndc` falls back to the key in `/etc/rndc.key`, generated automatically at install time via `rndc-confgen -a`.

## Zone File Reference

### $TTL

Sets the default Time to Live for records in the zone — how long a resolver should cache the answer before asking again. Individual records can override this with their own TTL.

```
$TTL 1D
```

A longer TTL means fewer queries hit your server, but changes take longer to propagate.

### Common Resource Records

**A** — maps a hostname to an IPv4 address.

```
server1 IN A 10.0.1.3
        IN A 10.0.1.5
```

If you omit the hostname on a line, the record applies to the last hostname specified — that's why the second `IN A` above still applies to `server1`.

**CNAME** — aliases one name to another.

```
www IN CNAME server1
```

Rules worth remembering: a CNAME shouldn't point to another CNAME (this risks resolution loops), and other record types that need to resolve to a real FQDN — NS, MX, PTR — should never point at a CNAME.

**MX** — specifies the mail server(s) for a domain, with a preference value (lower wins).

```
example.com. IN MX 10 mail.example.com.
example.com. IN MX 20 mail2.example.com.
```

Multiple servers can share the same preference value for basic load balancing.

**NS** — declares an authoritative name server for the zone.

```
IN NS dns1.example.com.
IN NS dns2.example.com.
```

Both listed servers are considered authoritative, regardless of which is primary and which is secondary.

**PTR** — maps an address back to a name, used in reverse zones.

```
last-IP-digit IN PTR FQDN-of-system
```

**SOA** — Start of Authority. Always the first record in a zone file.

```
@ IN SOA primary-name-server hostmaster-email (
        serial-number
        time-to-refresh
        time-to-retry
        time-to-expire
        minimum-TTL )
```

- `@` refers to `$ORIGIN` (or the zone name if `$ORIGIN` isn't set).
- **primary-name-server** — the authoritative primary server for this domain.
- **hostmaster-email** — contact address for the zone (written with a dot instead of `@`).
- **serial-number** — incremented on every edit; tells secondaries it's time to reload.
- **time-to-refresh** — how often secondaries check the primary for updates.
- **time-to-retry** — how long a secondary waits before retrying if the primary doesn't respond.
- **time-to-expire** — how long a secondary keeps answering as authoritative if the primary stays unreachable.
- **minimum-TTL** — in BIND 9, this defines how long negative answers (NXDOMAIN) are cached, capped at 3 hours.

BIND accepts time abbreviations instead of raw seconds:

| Seconds | Equivalent |
|---|---|
| 60 | 1M |
| 1800 | 30M |
| 3600 | 1H |
| 10800 | 3H |
| 21600 | 6H |
| 43200 | 12H |
| 86400 | 1D |
| 259200 | 3D |
| 604800 | 1W |
| 31536000 | 365D |

### Example Zone File

```
$ORIGIN example.com.
$TTL 86400
@ IN SOA dns1.example.com. hostmaster.example.com. (
                2001062501 ; serial
                21600      ; refresh after 6 hours
                3600       ; retry after 1 hour
                604800     ; expire after 1 week
                86400 )    ; minimum TTL of 1 day

        IN NS dns1.example.com.
        IN NS dns2.example.com.

dns1 IN A     10.0.1.1
dns2 IN A     10.0.1.2
dns1 IN CNAME server1
```

## Interview / Reference Q&A

**What does BIND stand for?**
Berkeley Internet Name Domain.

**What is BIND, fundamentally?**
A hierarchical, distributed database implementation of DNS. It maps hostnames to IPs and back, handles mail routing data, and answers queries through a resolver library. The BIND 9 distribution ships the `named` daemon and the `liblwres` resolver library.

**What port does BIND use?**
53, both TCP and UDP. Queries and normal responses go over UDP; if a response is too large for a single UDP packet, it's retried over TCP.

**How is a domain name structured?**
As a tree, read right to left, labels separated by dots. `mail.linuxtechi.com` — `com` is the top-level domain, `linuxtechi` is a subdomain of it, and `mail` names the host. A label only needs to be unique within its parent domain.

**What's a zone file?**
The file holding the actual data for a zone, made up of resource records. Every zone file starts with an SOA record.

**What types of DNS servers are there?**

- **Primary (master)** — holds the authoritative, human-edited (or generated) copy of the zone data.
- **Secondary (slave)** — replicates zone data from the primary (or another secondary) via zone transfer. A secondary can itself serve as a master to another secondary.
- **Caching name server** — not authoritative for anything; forwards queries it can't answer from cache and caches the responses.
- **Forwarding server** — forwards all queries to a fixed list of other name servers.

**How does DNS achieve load balancing?**
By returning multiple A records for one name. If `www.example.com` has three A records pointing at `10.0.0.1`, `10.0.0.2`, and `10.0.0.3`, BIND rotates the order it returns them in on each query. Most clients use whichever record comes first, so requests spread out roughly evenly across the three servers.

**How do you check `named.conf` syntax?**

```bash
named-checkconf /etc/named.conf
```

In a chroot environment:

```bash
named-checkconf -t /var/named/chroot /etc/named.conf
```

**What resource record types exist in BIND?**

| Record | Purpose |
|---|---|
| SOA | Start of authority for the zone |
| NS | Name server |
| A | Name-to-address mapping |
| PTR | Address-to-name mapping |
| CNAME | Alias for another name |
| MX | Mail exchanger |
| TXT | Free-form text |
| RP | Responsible person / contact |
| WKS | Well-known services |
| HINFO | Host information |

Comments in zone files start with `;` and run to the end of the line.

**What's a BIND chroot environment?**
Running `named` jailed to a subdirectory (`/var/named/chroot`), so a compromised process can't reach the rest of the filesystem. Standard sandboxing for reducing blast radius.

**What is domain delegation?**
Handing off authority for a subdomain to a different name server:

```
squid.linuxtechi.com  IN NS  ns2.linuxtechi.com.
ns2.linuxtechi.com     IN A   192.168.1.51
```

## Key Takeaways

- Forward zones resolve names to IPs; reverse zones resolve IPs to names. Both need their own zone file and their own entry in `named.conf`.
- The reverse zone name is derived from the network address in reverse octet order plus `.in-addr.arpa`.
- Always run `named-checkconf` and `named-checkzone` before restarting `named` after editing a zone file — a broken zone file fails silently otherwise.
- SELinux and file ownership on `/var/named` trip people up more often than the actual BIND configuration does. `restorecon` and `chown root:named` are easy to forget and annoying to debug around.
- `rndc reload` is the standard way to pick up zone changes without dropping cached answers, which matters if you're iterating on a zone file and don't want to force every client to re-resolve everything.
