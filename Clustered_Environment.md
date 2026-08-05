# Building a High-Availability Cluster with Pacemaker and Corosync on CentOS Stream

## What is a clustered environment?

A clustered environment is a group of two or more servers, called nodes, that work together and act as a single system to the outside world. The nodes constantly talk to each other over the network, tracking whether their peers are alive and which services are currently running where. If one node fails, another node in the cluster picks up the workload automatically, usually within seconds.

This lab uses Pacemaker and Corosync, the standard high-availability stack on RHEL-based distributions. Corosync handles the low-level messaging between nodes (heartbeats, membership, quorum). Pacemaker sits on top of it and makes the actual decisions: which resources should run, where they should run, and what to do when a node goes down. `pcs` (Pacemaker/Corosync Configuration System) is the command-line tool used to configure both.

## Why use clustering?

A single server is a single point of failure. If it crashes, loses power, or needs a kernel update and a reboot, whatever it was serving goes down with it. Clustering exists to remove that single point of failure for services that need to stay up:

- Web servers and load balancers
- Databases
- File shares (NFS/Samba)
- Any service where downtime has a direct cost

The mechanism this lab focuses on is a floating IP (also called a virtual IP or VIP). Clients connect to one IP address, but the cluster is free to move that address between nodes. If the node currently holding the IP dies, Pacemaker relocates it to a healthy node, and clients keep working without reconfiguring anything on their end. This is the same underlying idea behind tools like Keepalived, and it's the building block most production HA setups (databases, load balancers, etc.) are built on top of.

<img width="340" height="255" alt="image" src="https://github.com/user-attachments/assets/ed6b28df-8dfe-4301-ab5e-a8aeb7df7d82" />


## Lab topology

| Role | Hostname | IP Address |
|---|---|---|
| Cluster node 1 | clusterVM1 | 192.168.100.218 |
| Cluster node 2 | clusterVM2 | 192.168.100.219 |
| Floating/Virtual IP | (owned by whichever node is active) | 192.168.100.220 |
| Test client | Windows machine | (same subnet, used to ping the floating IP) |

Both `clusterVM1` and `clusterVM2` are CentOS Stream 9 VMs on the same subnet. The Windows machine is not part of the cluster; it's just there to prove, from an outside client's perspective, that the floating IP survives a node failure.

---

## Step 1: Enable the HA repo and install packages (all nodes)

Run this on both `clusterVM1` and `clusterVM2`.

```bash
dnf repolist all
```
Lists every repo dnf knows about, enabled or not. This is just to confirm the HighAvailability repo isn't already there before adding it manually.

```bash
vim /etc/yum.repos.d/CentOS-Stream-HighAvailability.repo
```
Opens a new repo definition file. CentOS Stream ships Pacemaker and Corosync in a separate HighAvailability repo that isn't enabled by default, so it has to be added by hand. Paste in:

```ini
[HighAvailability]
name=CentOS Stream $stream - HighAvailability
baseurl=https://mirror.stream.centos.org/9-stream/HighAvailability/$basearch/os/
enabled=1
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-centosofficial
```
- `[HighAvailability]`: the repo ID, how dnf refers to it internally.
- `name=`: the human-readable label shown in `dnf repolist`.
- `baseurl=`: where to pull the packages from. `$stream` and `$basearch` are variables dnf fills in automatically (e.g. `9-stream`, `x86_64`).
- `enabled=1`: turns the repo on immediately.
- `gpgcheck=1`: verifies package signatures before installing anything from this repo.
- `gpgkey=`: points to the local CentOS GPG key used to do that verification.

```bash
dnf clean all
dnf makecache
```
`clean all` clears out dnf's local cache of repo metadata, and `makecache` rebuilds it, this time pulling in the new HighAvailability repo. Without this, dnf might still be looking at stale metadata that doesn't know the repo exists yet.

```bash
dnf install -y pacemaker corosync pcs resource-agents fence-agents-all
```
Installs the actual cluster stack:
- `pacemaker`: the cluster resource manager, decides what runs where.
- `corosync`: the cluster communication layer, handles node membership and heartbeats.
- `pcs`: the CLI used to configure and query the cluster.
- `resource-agents`: the scripts Pacemaker uses to start/stop/monitor things like IP addresses, filesystems, and services.
- `fence-agents-all`: scripts for fencing (STONITH), the mechanism that forcibly powers off or isolates a misbehaving node. Installed here for completeness even though this lab disables fencing later (see Step 7).

```bash
systemctl enable --now pcsd
systemctl status pcsd
```
`pcsd` is the daemon that lets `pcs` on one node talk to `pcs` on another node over the network, which is how cluster setup gets pushed out to both machines from one place. `enable --now` starts it immediately and sets it to start on every future boot. `status` confirms it's actually running before moving on.

## Step 2: Open the firewall for cluster traffic (all nodes)

```bash
firewall-cmd --add-service=high-availability --permanent
firewall-cmd --reload
firewall-cmd --list-services | grep high-availability
```
Corosync and Pacemaker need specific TCP/UDP ports open between the nodes (for heartbeats, pcsd communication, and resource state sync). Rather than opening individual ports, `firewalld` ships a predefined `high-availability` service group that bundles all of them.
- `--add-service=high-availability --permanent` adds the rule to the persistent firewall config, but doesn't apply it live yet.
- `--reload` re-reads the firewall config so the new rule actually takes effect.
- The `grep` at the end is just a sanity check that the service is now listed as allowed.

## Step 3: Authenticate the nodes to each other (all nodes)

```bash
passwd hacluster
```
The `hacluster` system user is created automatically when the `pcs` package is installed, but it has no usable password by default. This sets one. Use the same password on both nodes, since the next command relies on it.

```bash
pcs host auth clusterVM1 clusterVM2 -u hacluster
```
Run once, from either node (this lab treats `clusterVM1` as the main node). This authenticates `pcsd` on the local machine against `pcsd` on both listed hosts, using the `hacluster` account and the password set above. It's what lets `pcs` push cluster configuration to both nodes from a single command later, instead of having to log into each node separately.

#NOTE : If this shows error then vi into /etc/hosts file and add IP corresponding to its VM name in that file on both the servers and save them and run this  above command again  

## Step 4: Set up the cluster (main node only)

```bash
pcs cluster setup mycluster 192.168.100.218 192.168.100.219
```
This is the command that actually creates the cluster. `mycluster` is an arbitrary name for the cluster (used internally by Corosync). The two IPs are `clusterVM1` and `clusterVM2`, the members of the cluster. Behind the scenes this generates and distributes `corosync.conf` to both nodes, which is what Corosync reads to know who its peers are.

## Step 5: Time sync via NTP (all nodes)

```bash
systemctl enable --now chronyd
timedatectl
```
Cluster nodes need to agree on the time. Log correlation, certificate validation, and some resource agents assume clocks are in sync, and a large clock drift between nodes can cause confusing failures that have nothing to do with the actual cluster config. `chronyd` is the NTP client that keeps the system clock synced against a time server. `timedatectl` just shows the current time/sync status so you can confirm it's working.

## Step 6: Restart the cluster daemons (all nodes)

If this is a lab that also needs shared storage (a SAN, iSCSI target, or NFS export both nodes can read/write), that would be configured at this point, before the daemons pick up any storage-dependent resources. This lab keeps things simple and only clusters a floating IP, so there's no shared storage step here.

```bash
systemctl restart corosync
systemctl restart pacemaker
```
Restarting both daemons ensures they pick up the `corosync.conf` generated in Step 4 cleanly, rather than relying on whatever state they were in from installation.

## Step 7: Relax fencing and quorum (main node only)

```bash
pcs property set stonith-enabled=false
pcs property set no-quorum-policy=ignore
```
Two cluster-wide properties, both being loosened for lab purposes:

- `stonith-enabled=false` turns off fencing (STONITH: "Shoot The Other Node In The Head"). In production, when a node stops responding, the cluster can't tell if it's actually dead or just unreachable, so fencing forcibly powers it off through an out-of-band mechanism (IPMI, a smart PDU, etc.) before starting resources anywhere else, to guarantee the same resource never runs on two nodes at once. That hardware isn't available in a 2-VM lab, so fencing is disabled here. This is fine for testing, but a real production cluster should not run with `stonith-enabled=false`.
- `no-quorum-policy=ignore` tells the cluster to keep running resources even if it loses quorum (i.e., fewer than half the configured nodes are reachable). With only 2 nodes, quorum math doesn't work well by default (losing 1 node out of 2 means losing "half"), so this is set to ignore quorum loss rather than have the cluster shut everything down.

## Step 8: Start and enable the cluster

```bash
pcs cluster start --all
pcs cluster enable --all
pcs status
```
- `cluster start --all` starts Corosync and Pacemaker on every node in the cluster, run from just one node.
- `cluster enable --all` sets the cluster services to start automatically on boot, on every node.
- `pcs status` is the main status command for the whole lab. It shows which nodes are online, which resources are configured, and where each resource is currently running. Worth running after almost every step from here on to confirm nothing broke.

## Step 9: Create the floating IP resource

```bash
pcs resource create ClusterIP ocf:heartbeat:IPaddr2 ip=192.168.100.220 cidr_netmask=24 op monitor interval=30s
pcs resource
```
This defines the actual thing the cluster is protecting.
- `pcs resource create ClusterIP`: creates a new cluster resource named `ClusterIP`.
- `ocf:heartbeat:IPaddr2`: the resource agent used to manage it. `ocf` is the standard, `heartbeat` is the provider namespace, `IPaddr2` is the specific agent that knows how to bring an IP address up and down on a network interface.
- `ip=192.168.100.220`: the floating IP itself, separate from either node's own static IP.
- `cidr_netmask=24`: the subnet mask for that IP, matching the lab's /24 network.
- `op monitor interval=30s`: tells Pacemaker to check every 30 seconds that the IP is actually still up and responding. If the check fails, Pacemaker treats the resource as failed and moves it.

`pcs resource` lists all configured resources and shows which node each one is currently active on. At this point `ClusterIP` should show as running on `clusterVM1` (or whichever node claimed it).

## Step 10: Test the floating IP from the Windows client

```
ping 192.168.100.220
```
Run from the Windows machine, not either cluster node. This confirms the floating IP is reachable from outside the cluster, exactly like a real client hitting a service. At this stage it's just proving the resource is up; the real test is what happens to this ping during a failover.

## Step 11: Simulate a failover

```bash
pcs cluster stop 192.168.100.218
pcs status
```
Stops the cluster service on `clusterVM1` specifically (identified here by its IP), simulating that node going down or being taken offline for maintenance. `pcs status`, still run from a node that's still up, should now show `clusterVM1` as offline and `ClusterIP` relocated to `clusterVM2`.

```
ping 192.168.100.220
```
Run again from the Windows client, ideally kept running continuously through the failover rather than run fresh afterward. A handful of dropped pings during the actual switchover is normal and expected; what matters is that it recovers and keeps responding, now being answered by `clusterVM2` instead of `clusterVM1`, without any change on the Windows machine's end.

```bash
pcs status
```
Run this last check on the other node (`clusterVM2`) to confirm its view of the cluster agrees: itself online, `clusterVM1` offline, and `ClusterIP` now running locally.

---

## What this demonstrates

At the end of this lab, a client on the network only ever talks to one address, `192.168.100.220`, and has no idea which physical VM is actually answering at any given moment. Kill either node and the other takes over the IP within the monitor interval. This is the same pattern used to keep databases, load balancers, and file services available in production, just applied here to the simplest possible resource (a single IP) to keep the moving parts visible.

## Notes and caveats

- `stonith-enabled=false` is a lab-only shortcut. Any real deployment needs proper fencing hardware/agents configured, or a split-brain scenario (both nodes think they own the resource) becomes possible.
- This lab only clusters an IP address. A more realistic setup would add a resource group tying the IP to an actual service (e.g. `httpd`), plus shared or replicated storage, so failover moves the whole stack, not just the address.
- `no-quorum-policy=ignore` is standard practice for 2-node clusters specifically, since quorum requires a majority and 2 nodes can't produce one on their own without a third tie-breaker (a quorum device). It would need revisiting for a cluster with 3+ nodes.
