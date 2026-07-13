# Lab: Ansible - Configuration Management & Automation

## What is Ansible?

Ansible is an open-source automation tool — a suite of software that enables infrastructure as code. It handles:

- **Provisioning**
- **Configuration management**
- **Application deployment**

All without needing to manually log into every machine.

### The problem it solves

Say you're a system admin at a company and you're told to manage 100 servers. If all 100 need the same software installed, logging into each one individually is a nightmare. Instead, you write a single **playbook** - a set of instructions (install this, configure that, manage this service) - and run it *once*. Ansible connects to every server in your inventory and executes the same steps on all of them.

If you need to manage multiple systems, you list all their hostnames/IPs in an **inventory file**, and Ansible knows exactly which machines to connect to and run the playbook against.

---

## Core Concepts

| Term | Meaning |
|---|---|
| **hosts** | The target machine(s) the play runs against (e.g. `localhost`, `webservers`) |
| **become** | Whether the task needs to run with elevated (root/admin) privileges |
| **tasks** | The list of individual actions a play performs |
| **playbook** | A YAML file containing one or more "plays" — always written in `.yml` |

Example task block:
```yaml
- name: Install Nano text editor
  dnf:
    name: nano
    state: present
```
- `name` → a human-readable description of what the task does (shows up in output, helps track relationships between tasks)
- `dnf` → the module used (package manager module here)
- `state: present` → ensures the package is installed

---

## Step 1: Install Ansible

```bash
yum -y install ansible
# or
dnf install ansible -y
# or
dnf install ansible-core -y

ansible --version
```

## Step 2: Configure the Inventory File

```bash
vi /etc/ansible/hosts
```

Add a group and the local machine under it:
```ini
[webservers]
localhost ansible_connection=local
```

This lets you target `localhost` directly without needing SSH.

## Step 3: Add a Remote Server to the Inventory

Before Ansible can manage a remote machine, it needs passwordless SSH access to it.

```bash
ssh-keygen -t rsa -b 2048
ssh-copy-id shlok@192.168.100.169      # a different server — NOT the control node running Ansible
ssh shlok@192.168.100.169              # verify the connection works
exit
```

Then edit the hosts file again and add the remote server under the same group:

```bash
vi /etc/ansible/hosts
```
```ini
[webservers]
localhost ansible_connection=local
ShlokServer ansible_host=192.168.100.169 ansible_user=shlok
```
- `ShlokServer` → arbitrary hostname/alias you choose
- `ansible_host` → the actual IP of that server
- `ansible_user` → the username used to SSH in

## Step 4: Test Connectivity

```bash
ansible -m ping webservers
ansible -m ping all
```

A `ping` response from every host confirms Ansible can reach and authenticate to all of them.

---

## Step 5: Write a Playbook - Install Apache on Both Machines

```bash
vi install-httpd.yml
```

```yaml
# Ansible Playbook to install Apache on 2 machines: local & ShlokServer
---
- hosts: webservers
  become: yes
  become_user: root
  tasks:
    - name: Install httpd package
      dnf:
        name: httpd
        state: present
```

Save and quit.

## Step 6: Run the Playbook

```bash
ansible-playbook install-httpd.yml --ask-become-pass
```
`--ask-become-pass` prompts for the sudo/root password of the remote machine(s) where the package is being installed.

## Step 7: Verify

On both the local machine and `ShlokServer`:
```bash
rpm -qa | grep httpd
```
Confirms `httpd` was successfully installed on every host targeted by the play.

---

## Key Takeaway

One playbook, one command, both machines configured identically - no manual per-server login required. This is the core value of Ansible: **write once, run everywhere in your inventory.**
