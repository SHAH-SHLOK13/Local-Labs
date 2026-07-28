# Nagios Core – Installation, Configuration & Host Monitoring

A step-by-step lab for installing Nagios Core from source on a RHEL-based system (RHEL/CentOS/Rocky/Alma using `dnf`), configuring it to monitor a remote Linux host over ICMP, and getting the web interface live.

## What is Nagios

Nagios is a monitoring and alerting tool for servers, network devices, and applications or services running on client machines. It runs scheduled checks against whatever you tell it to watch, logs the results, and fires off a notification the moment something goes from OK to a problem state. Every alert gets logged, so you have a history to go back to when something breaks at 3 AM and you need to know when it actually started.

The project was originally called NetSaint before being renamed to Nagios.

## Why use it

Nagios is one of the oldest and most widely deployed open-source monitoring tools, and it's a common baseline to learn before moving on to something like Prometheus/Grafana or Zabbix. Reasons it still gets used:

- Free and open source, with a huge library of community plugins for almost anything you'd want to check (disk space, HTTP endpoints, database connections, SNMP-capable network gear, custom scripts).
- Host and service checks are defined in plain text config files, which makes them easy to version-control and template.
- Works agentless for basic checks (ping, port checks) and can use NRPE or NCPA when you need to pull metrics from inside a remote host.
- Still common in enterprise environments, so understanding it is directly useful if you're heading into a sysadmin or NOC-type role.

## Lab environment

| Component | Details |
|---|---|
| Nagios server | RHEL-based VM (dnf package manager) |
| Client/monitored host | Any Linux VM reachable over the network |
| Nagios Core version | Check the [official downloads page](https://www.nagios.org/downloads/nagios-core/) for the current stable release — this guide uses the version numbers as placeholders |
| Nagios Plugins version | Check the [nagios-plugins GitHub releases](https://github.com/nagios-plugins/nagios-plugins/releases) — the old `nagios-plugins.org` download link is unreliable |

> **Note on versions:** the original lab notes this was built from referenced `nagios-4.5.4` and `nagios-plugins-2.4.11`. Rather than hardcode a version that will eventually go stale, this guide uses `<NAGIOS_VERSION>` and `<PLUGINS_VERSION>` as placeholders — substitute in whatever the current release is when you follow along.

---

## 1. Install required dependencies

Nagios Core is built from source, so you need a compiler, a web server to serve the interface, and PHP for the front end.

```bash
dnf install -y httpd php gcc glibc glibc-common gd gd-devel make net-snmp unzip wget openssl-devel
```

What each of these is for:
- `httpd` – Apache web server, serves the Nagios web interface
- `php` – renders the interface pages
- `gcc`, `make`, `glibc`, `glibc-common` – build toolchain
- `gd`, `gd-devel` – graphics library, used by Nagios to generate status graphs
- `net-snmp` – SNMP support for network device checks
- `unzip`, `wget` – for pulling and extracting the source tarballs
- `openssl-devel` – needed when compiling Nagios Core itself (missing this causes the `./configure` step below to fail)

## 2. Create the nagios user and command group

Nagios runs as its own unprivileged user. The `nagcmd` group is what lets Apache (running as the `apache` user) and Nagios talk to each other through the external command file — this is what lets you issue commands from the web interface.

```bash
useradd nagios
groupadd nagcmd
usermod -a -G nagcmd nagios
usermod -a -G nagcmd apache

# confirm both users picked up the group
id nagios
id apache
```

## 3. Download and extract Nagios Core

```bash
cd /tmp
wget -O nagios-<NAGIOS_VERSION>.tar.gz "https://assets.nagios.com/downloads/nagioscore/releases/nagios-<NAGIOS_VERSION>.tar.gz"
tar xzf nagios-<NAGIOS_VERSION>.tar.gz
cd nagios-<NAGIOS_VERSION>
```

## 4. Compile Nagios

```bash
./configure --with-command-group=nagcmd
```

This is the step that fails if `openssl-devel` isn't installed — the configure script checks for it before generating the Makefile.

## 5. Build and install

```bash
make all
make install
make install-commandmode
make install-init
make install-config
make install-webconf
```

Each `make install-*` target installs a different piece: the binaries, the command-mode permissions, the systemd service, the default config files, and the Apache config for the web interface, respectively.

## 6. Download, build, and install Nagios Plugins

The core package only ships the engine — the actual checks (`check_ping`, `check_http`, `check_disk`, etc.) come from a separate plugins package.

```bash
cd /tmp
wget -O nagios-plugins-<PLUGINS_VERSION>.tar.gz "https://github.com/nagios-plugins/nagios-plugins/releases/download/release-<PLUGINS_VERSION>/nagios-plugins-<PLUGINS_VERSION>.tar.gz"
tar xzf nagios-plugins-<PLUGINS_VERSION>.tar.gz
cd nagios-plugins-<PLUGINS_VERSION>

./configure --with-nagios-user=nagios --with-nagios-group=nagios
make
make install
```

## 7. Open the firewall for the web interface

The original lab notes for this step said to stop and disable `firewalld` entirely. **Don't do that** — it's the opposite of the hardening advice later in the same notebook, which explicitly says to keep firewalls enabled. Instead, just open the port the Nagios web interface actually needs:

```bash
firewall-cmd --permanent --add-service=http
firewall-cmd --reload
```

If your monitored hosts also need SNMP polling back from this box, open that too (`--add-port=161/udp`), but there's no reason to disable the firewall as a whole for this lab.

## 8. Check SELinux (commonly skipped, commonly breaks things)

RHEL-based systems ship with SELinux enforcing by default, and it's a very common reason Nagios "installs fine but the web interface won't load" or checks silently fail. Before troubleting for an hour, check the booleans:

```bash
getenforce
setsebool -P httpd_can_network_connect 1
```

If you see `AVC denial` messages in `/var/log/audit/audit.log` referencing httpd or nagios, `audit2allow` can help build a targeted policy rather than disabling SELinux outright.

## 9. Set the web interface password and enable services

```bash
htpasswd -c /usr/local/nagios/etc/htpasswd.users nagiosadmin

systemctl start httpd
systemctl enable httpd

systemctl start nagios
systemctl enable nagios

systemctl status nagios
```

## 10. Configure a host to monitor

Nagios ships with a sample `localhost.cfg` you can look at as a reference:

```bash
cd /usr/local/nagios/etc/objects/
ls -ltr
vi localhost.cfg   # read through it, or copy it as a template
```

Create a new file for the host you actually want to monitor:

```bash
vi hosts.cfg
```

```cfg
define host {
    use                     linux-server
    host_name               shloklinux
    alias                   nagiosserver
    address                 192.168.100.162
    max_check_attempts      5
    check_period            24x7
    notification_interval   30
    notification_period     24x7
}

define service {
    use                     generic-service
    host_name               shloklinux
    service_description     PING
    check_command           check_ping!100.0,20%!500.0,60%
    max_check_attempts      5
    normal_check_interval   5
    retry_check_interval    1
    check_period            24x7
    notification_interval   30
    notification_period     24x7
}
```

`address` is the IP of the client/remote machine you want to monitor, not the Nagios server itself. The `check_command` line for the PING service was `check-ping` in the handwritten notes — the actual command name uses an underscore, `check_ping`. A hyphen there will fail with an "undefined command" error when you verify the config.

## 11. Register the new config file

Nagios only reads config files it's explicitly told about in `nagios.cfg`.

```bash
cd /usr/local/nagios/etc
vi nagios.cfg
```

Add a line below the existing `localhost.cfg` entry:

```cfg
cfg_file=/usr/local/nagios/etc/objects/hosts.cfg
```

## 12. Verify and start

Always verify the config before restarting — a syntax error here will stop Nagios from starting at all.

```bash
/usr/local/nagios/bin/nagios -v /usr/local/nagios/etc/nagios.cfg
```

Look for `Things look okay` at the bottom of the output. If it flags errors, fix them and re-run this before touching the service.

```bash
systemctl restart nagios
```

## 13. Access the web interface

```
http://<nagios-server-ip>/nagios
```

Log in with `nagiosadmin` and the password set in step 9. Your new host and its PING service should show up under **Hosts** / **Services** within a few minutes, depending on the check interval.

---

## Troubleshooting notes

- **Web page loads but shows a blank or broken layout** — usually a PHP or Apache config issue from `make install-webconf` not being picked up; restart `httpd`.
- **"Things look okay" doesn't appear during verify** — read the actual error above it, it names the exact config file and line number.
- **Host shows as unreachable but you can ping it manually** — check that the firewall is open (step 7) and that the plugins actually built (step 6); `check_ping` living in `/usr/local/nagios/libexec/` is a good sanity check.
- **Permission denied on the external command file** — almost always means the `nagcmd` group membership from step 2 wasn't applied, or Apache wasn't restarted after being added to the group.
