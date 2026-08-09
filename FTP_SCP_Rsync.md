# File Transfer in Linux: FTP, SCP, and Rsync

A hands-on lab covering three ways to move files between Linux machines - FTP (the old standard), SCP (the secure one-off copy), and rsync (the fast, incremental sync tool). Same two-VM setup as the other labs: one box acts as the server, one as the client, both reachable over a private network.

Replace `192.168.1.X` below with your actual server IP throughout.

---

## Why three tools for the same job?

They all move files from one machine to another, but they solve different problems:

| Tool | Port | Encrypted? | Best for |
|------|------|------------|----------|
| FTP  | 21 (control) | No — credentials and data are plaintext | Legacy systems, internal networks where encryption isn't a concern |
| SCP  | 22 (over SSH) | Yes | Quick one-off copies where you already have SSH access |
| rsync | 22 (over SSH, by default) | Yes (when tunneled through SSH) | Backups, repeated syncs, large directories, anything where re-copying everything every time is wasteful |

FTP predates SSH and was never designed with security in mind - usernames, passwords, and file contents all go over the wire in the clear. It still shows up in older infrastructure and some internal-only setups, which is why it's worth knowing. SCP and rsync both ride on top of SSH, so anything they send is encrypted the same way an SSH session is.

The real reason to reach for rsync over the other two: it only transfers what changed. Copy a 20 MB file with FTP or SCP twice and you move 40 MB. Change one line in it and rsync again, and rsync sends a few KB — it diffs the file against the destination and only ships the delta.

---

## Lab setup

- Two RHEL/CentOS machines (or one VM playing both roles) — call them **server** and **client**
- Both able to reach each other over the network (confirm with `ping`)
- A non-root user on both sides — this lab uses `shlok`

```bash
ping www.google.com   # sanity check that the network is up before anything else
```

---

## Part 1 — FTP (File Transfer Protocol)


<img width="800" height="400" alt="image" src="https://github.com/user-attachments/assets/e1fcb8a3-b898-477f-8527-c8db849694f9" />


FTP is a standard network protocol for transferring files between a client and a server. It runs as a client/server pair: the server runs an FTP daemon (`vsftpd` here) that listens for connections, and the client connects to it to push or pull files.

- **Default port:** 21
- **Architecture:**

```
[client] ---FTP (port 21)---> [server, vsftpd running]
```

### 1.1 Server side

Become root and check whether `vsftpd` is already installed before doing anything else:

```bash
rpm -qa | grep vsftpd
```

If nothing comes back, install it:

```bash
yum install vsftpd
```

Back up the config before touching it - you'll want the original if something breaks:

```bash
cp /etc/vsftpd/vsftpd.conf /etc/vsftpd/vsftpd.conf.bak
vi /etc/vsftpd/vsftpd.conf
```

Find and set the following:

```ini
anonymous_enable=NO          # disable anonymous login — anyone could connect otherwise
ascii_upload_enable=YES      # uncomment
ascii_download_enable=YES    # uncomment
ftpd_banner=Welcome to UNIXMEN FTP service.   # uncomment, change the message if you like
```

Add this line at the end of the file:

```ini
use_localtime=YES
```

Save and quit.

Two directives that aren't in the notes but matter - check they're set to `YES` (they usually are by default in the stock RHEL config, but confirm rather than assume):

```ini
local_enable=YES     # lets local system users log in via FTP
write_enable=YES     # without this, uploads (put) fail even with correct permissions
```

Start the service and open it up for the lab:

```bash
systemctl start vsftpd
systemctl enable vsftpd

systemctl stop firewalld     # lab environment only — never do this on a real server
systemctl disable firewalld  # see "Common errors" below for the production-safe version

useradd shlok    # only if the user doesn't already exist
```

### 1.2 Client side

```bash
su -                 # become root
yum install ftp       # ftp client package
exit                  # back to the shlok user
touch testfile         # or any file you want to transfer
```

### 1.3 Transferring the file

From the client:

```bash
ftp 192.168.1.X        # IP of the server
```

You'll be prompted for the server's username and password. Once connected:

```
bin              # switch to binary transfer mode — important for anything that isn't plain text
hash             # optional: prints a # for each block transferred, just a progress indicator
put testfile     # uploads testfile from client to server
bye              # closes the FTP session
```

---

## Part 2 — SCP (Secure Copy Protocol)


  <img width="860" height="483" alt="image" src="https://github.com/user-attachments/assets/825af048-ee76-469c-986d-4346a5b6e264" />


SCP does the same job as FTP but rides over SSH, so authentication and file contents are both encrypted.

- **Default port:** 22 (same as SSH - no separate daemon to configure)
- **Architecture:**

```
[client] ---scp (uses SSH)---> [server]
                                sshd on port 22
```

Because it uses SSH, there's no separate server setup - if SSH is already running (it almost always is), SCP works out of the box.

### 2.1 Transfer from client to server

```bash
touch jack                                          # any file to transfer
scp jack shlok@192.168.1.X:/home/shlok/              # push it to the server
```

Enter the remote user's password when prompted. The trailing slash on the destination matters - it tells SCP to drop the file inside that directory rather than trying to use it as a filename.

---

## Part 3 — Rsync (Remote Synchronization)

Rsync is built for efficiently transferring and *syncing* files, whether that's between two directories on the same machine or between a local and a remote one. It's the tool to reach for over FTP or SCP whenever you're repeating a transfer — backups, mirrored directories, anything that runs more than once.

- **Default port:** 22 (it tunnels over SSH by default, same as SCP)
- **Architecture:**

```
[client] ---rsync (uses SSH)---> [server]
                                  sshd on port 22
```

### 3.1 Why it's faster on repeat transfers


<img width="495" height="268" alt="image" src="https://github.com/user-attachments/assets/208049af-af14-43a3-8e9b-7eb68bca9223" />


Rsync uses a delta-transfer algorithm - it doesn't resend a whole file just because it grew or changed slightly. It compares the source and destination and only ships the difference.

Example: a file grows from 2 MB to 3 MB to 20 MB across three runs. FTP or SCP would copy 2 MB, then 3 MB, then 20 MB again in full each time — 25 MB total. Rsync copies the full 2 MB once, then only the ~1 MB delta on the second run, then only the ~17 MB delta on the third. It's copying the *difference in size*, not the whole file, on every run after the first.

### 3.2 Basic syntax

```bash
rsync [options] source destination
```

### 3.3 Install

```bash
yum install rsync          # CentOS / RHEL
apt-get install rsync      # Ubuntu / Debian
```

### 3.4 Sync a single file, locally

```bash
tar cvf backup.tar .              # tar up the current directory
mkdir /tmp/backups
rsync -zvh backup.tar /tmp/backups/
```

Flags: `-z` compresses during transfer, `-v` is verbose, `-h` prints sizes in human-readable form (KB/MB instead of raw bytes).

### 3.5 Sync a directory, locally

```bash
rsync -azvh /home/shlok /tmp/backups/
```

`-a` is archive mode — it preserves permissions, timestamps, and symlinks, and recurses into subdirectories. Rsync only copies what's missing or changed at the destination, not the whole tree every time.

### 3.6 Push a file to a remote machine

On the server:

```bash
mkdir /tmp/backups
```

From the client:

```bash
rsync -avz backup.tar shlok@192.168.1.X:/tmp/backups
```

### 3.7 Pull a file or directory from a remote machine

Same idea, reversed — the remote path goes first as the source:

```bash
rsync -avzh shlok@192.168.1.X:/home/shlok/backups /tmp/backups
```

This pulls `/home/shlok/backups` from the server down to `/tmp/backups` on the client.

### 3.8 Bonus: quick downloads with wget

Not part of the FTP/SCP/rsync family, but it comes up in the same context — grabbing a file straight from a URL rather than from another host you control:

```bash
wget http://website.com/filename
```

---

## Common errors and fixes

A few things that will trip you up that aren't obvious from the config alone:

- **`530 Login incorrect` on FTP even with the right password.** Check `local_enable=YES` in `vsftpd.conf` — without it, local system accounts can't authenticate at all.
- **Login works but `put` fails with `550 Permission denied`.** Two separate things to check, in order:
  1. `write_enable=YES` in `vsftpd.conf`.
  2. SELinux. Even with the config correct, SELinux will block vsftpd from writing to a user's home directory by default on RHEL/CentOS:
     ```bash
     setsebool -P ftpd_full_access on
     ```
     Or check the specific denial with `ausearch -m avc -ts recent` if that's too broad for your setup.
- **FTP connects but then hangs on directory listing (`ls`) or transfer.** Classic passive-mode-through-a-firewall problem. If you can't just disable the firewall (i.e., anywhere outside a lab), open a passive port range in both `vsftpd.conf` and firewalld instead:
  ```ini
  pasv_enable=YES
  pasv_min_port=40000
  pasv_max_port=40100
  ```
  ```bash
  firewall-cmd --permanent --add-port=40000-40100/tcp
  firewall-cmd --permanent --add-service=ftp
  firewall-cmd --reload
  ```
- **`systemctl disable firewalld` in a real environment.** Fine for a throwaway lab VM, wrong everywhere else. The production-safe version opens only the ports you need instead of turning the firewall off entirely:
  ```bash
  firewall-cmd --permanent --add-service=ftp
  firewall-cmd --reload
  ```
- **First SCP/rsync connection to a new host hangs on a fingerprint prompt.** That's SSH asking you to confirm the host key — expected on the first connection to any new machine, type `yes`.

---

## Key takeaways

- **FTP** is plaintext end to end — credentials included. Only reasonable on a trusted internal network, or for legacy systems where nothing better is an option.
- **SCP** is the quickest secure option when you're moving a file once and already have SSH access. No server-side setup beyond SSH itself.
- **Rsync** is the right default for anything recurring — backups, mirrored directories, syncing across machines — because it only moves what changed instead of the whole file every time.
