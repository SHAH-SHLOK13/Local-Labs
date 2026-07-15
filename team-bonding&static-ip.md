# Network Team Bonding & Static IP Configuration (RHEL 8/9)

## Static vs Dynamic IP - Quick Primer

Before diving into tools, it's worth being clear on the two ways a machine gets an IP address:

- **Dynamic IP**: Assigned automatically by a DHCP server on the network. The address can change between reboots or lease renewals. Easiest to set up, but not predictable — not ideal for servers, VMs you SSH into regularly, or anything other machines need to reliably find.
- **Static IP**: Manually configured and fixed. It never changes unless you change it yourself. Preferred for servers, network infrastructure, and any host that needs a consistent, known address (e.g., something you SSH into, or that other services depend on).

In corporate/production environments, static IPs are the norm for servers; DHCP is typically reserved for general client devices.

---

## Network Management Tools

**NetworkManager** is a service that provides a set of tools designed specifically to make it easier to manage networking configuration on Linux systems. It is the default network management service on **RHEL 8** and **RHEL 9**.

NetworkManager exposes its functionality through a few different tools:

| Tool | Type | Use Case |
|------|------|----------|
| **GUI** (GNOME Settings) | Graphical | Only useful if you have a GNOME desktop environment. Not usable for CLI/headless systems. |
| **nmtui** | Text UI (TUI) | Menu-driven, runs in any terminal. Good middle ground when GUI isn't available but you still want a guided interface. |
| **nmcli** | Command Line | Full control via scripting. Used when GUI access isn't available, or when network changes need to be automated/scripted. |
| **nm-connection-editor** | Graphical | A full graphical management tool with access to most NetworkManager configuration options. Only accessible through a desktop or console session — almost equivalent to `nmtui` in scope, but more GUI-oriented and generally easier to use. |

### Tool Breakdown

- **`nmcli`** — NetworkManager's command-line interface. Used for scripting network configuration changes, or in headless/no-GUI environments.
- **`nmtui`** — NetworkManager's Text User Interface. Runs inside any terminal window; changes are made through menu selections and entering data — no scripting knowledge required.
- **`nm-connection-editor`** — Full GUI management tool, accessible only via desktop/console session.
- **GNOME Settings → Network** — The Network screen of the GNOME Settings app; allows only basic network management tasks. Not available on CLI-only systems.

---

## Lab: Creating a Network Team (Bonding Replacement) with `nmtui`

> **Note:** "Team" interfaces are NetworkManager's implementation of link aggregation and are effectively the modern replacement for traditional Linux bonding, configurable through `nmtui`.

### Goal
Combine two physical interfaces (`enp0s3`, `enp0s8`) into a single logical **team** interface for redundancy/aggregation.

### Steps

### **NOTE**: Before you delete any of your enp0s3 or your enp0s8 (or whatever you may have) , first check if the `libteam/teamd` package is present ~ rpm -qa | grep teamd (or) ~ rpm -qa | grep libteam . Because , if not installed earlier then will lead to you getting completely cutoff to the internet , so install any of these first then proceed to delete the interface.
1. Launch `nmtui`.
2. **Delete existing connections** for both interfaces first:
   - Delete `enp0s3`
   - Delete `enp0s8`
3. Go to **Add** → select **Team**.
4. Inside the Team connection editor, go to the **Slaves** section:
   - Add `enp0s3` as a slave → choose **Ethernet** connection type → leave all defaults → **OK**.
   - Add `enp0s8` as a slave the same way → **Ethernet** → leave defaults → **OK**.
5. Under the **Team** connection's own IP section, leave everything **Automatic** (DHCP) — do **not** set static here — and click **OK**.
6. Both `enp0s3` and `enp0s8` are now slaves under the team interface.
7. Confirm which ports/NICs are active via `nmtui` → **Edit a connection** or check the active connection list, and verify the team is up.


### Useful `nmtui` Options
- Edit a connection
- Set system hostname
- Check active connections (which ports/NICs are currently up)

---

## `nmcli` Reference

`nmcli` can do everything `nmtui` can, but scriptably.

### Device & Connection Status

```bash
nmcli device                 # Shows status of all connections (device-level)
nmcli device show            # Detailed info for a device (or new VM)
nmcli device show enp0s3     # Detailed info for a specific device
nmcli connection show        # Shows connection status (in a table)
nmcli connection show enp0s3 # Detailed status of a specific connection
```

### Viewing IP Details

```bash
nmcli connection show enp0s3 | grep ipv4.addresses   # current IPv4 address(es)
nmcli connection show enp0s3 | grep ipv4.gateway      # current gateway
nmcli connection show enp0s3 | grep ipv4.dns          # current DNS
nmcli connection show enp0s3 | grep ipv4.method       # manual (static) or auto (DHCP)
```

### Bringing a Connection Down

```bash
nmcli connection down enp0s3   # take connection down (disconnect) — required before modifying if active
```

### Configuring a Static IP

```bash
nmcli connection modify enp0s3 ipv4.addresses 10.253.1.211/24
nmcli connection modify enp0s3 ipv4.gateway 10.253.1.1
nmcli connection modify enp0s3 ipv4.dns 8.8.8.8
nmcli connection modify enp0s3 ipv4.method manual
```

> ⚠️ **Correction from original notes:** the method used to set a static IP is `ipv4.method manual` (not `static`) — `manual` is the correct NetworkManager keyword. `nmcli` will reject `static` as an invalid value.

### Adding a Secondary Static IP

If a host needs more than one IP on the same interface (common in corporate environments running multiple network segments):

```bash
nmcli connection modify enp0s3 +ipv4.addresses 10.0.0.211/24
```

The `+` prefix **adds** to the existing `ipv4.addresses` list rather than replacing it — this is how you assign a secondary static IP alongside the primary one, similar to how the first static IP was assigned.

### Apply the Changes

```bash
nmcli connection reload      # reload connection profiles
systemctl restart NetworkManager   # or reboot the system
ip a                         # verify the new IP(s) on enp0s3
```

> **Correction:** `systemctl reboot` reboots the *entire system* — for just re-applying network config, `nmcli connection reload` (or bringing the connection down/up again) is usually sufficient and much faster. Reserve a full reboot for when a service restart genuinely isn't enough.

---

## GUI Alternatives

### `nm-connection-editor`
- Uses GUI.
- Only useful in environments where a GUI is available.
- Almost identical in function to `nmtui`, but easier to use with more graphical/visual controls for network editing.

### GNOME Settings
- Only useful if you have a GNOME desktop environment.
- Navigate: **Settings → Network → Edit/Delete, etc.**
- Not usable from CLI-only systems.

---

## Quick Command Cheat Sheet

```bash
# Status checks
nmcli device
nmcli device show <iface>
nmcli connection show
nmcli connection show <iface>

# Static IP setup
nmcli connection down <iface>
nmcli connection modify <iface> ipv4.addresses <IP/CIDR>
nmcli connection modify <iface> ipv4.gateway <gateway>
nmcli connection modify <iface> ipv4.dns <dns>
nmcli connection modify <iface> ipv4.method manual

# Secondary IP
nmcli connection modify <iface> +ipv4.addresses <IP/CIDR>

# Apply
nmcli connection reload
ip a
```

---

## Notes / Verification Checklist

- [ ] Confirm which ports/NICs are active before making changes.
- [ ] After creating a team, verify both slave interfaces show under it via `nmtui` or `nmcli connection show`.
- [ ] After setting a static IP, verify with `ip a` and test connectivity (`ping gateway`, `ssh` to/from the host).
- [ ] If adding a secondary IP, confirm both addresses appear under the interface without wiping the first.
