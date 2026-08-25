---
title: ansible-setup-homelab
date: 2026-07-21
tags: ["linux", "homelab", "networking"]
---

This blog is part of an ongoing series of my homelab project [homelab](https://github.com/pranav10780/homelab)

# Ansible Setup

## Goal

Install Ansible on the host and confirm it can reach and manage both VMs.

## Install

```bash
python -m venv .venv
source .venv/bin/activate
pip install ansible
ansible --version
```

## Inventory

`ansible/inventory.ini` lists the managed nodes and how to reach them:

```ini
[server]
server ansible_host=127.0.0.1 ansible_port=2222 ansible_user=vboxuser

[clients]
client ansible_host=127.0.0.1 ansible_port=2223 ansible_user=vboxuser

[all:vars]
ansible_python_interpreter=/usr/bin/python3
```

- `[server]` / `[clients]` are host **groups** — playbooks can target a
  whole group at once.
- Both VMs are reached via `127.0.0.1` on their respective forwarded SSH
  ports, since VirtualBox NAT + port forwarding is being used.

## Connectivity test

```bash
ansible server -i inventory.ini -m ping
```

Expected:
```
server | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

`-m ping` uses Ansible's `ping` module — this checks SSH connectivity and
that Python is available on the remote host, it is not an ICMP ping.

## Running tasks that need root (`become`)

Installing packages / managing services requires elevated privileges.
Ansible handles this with `become: yes` in a playbook (equivalent to
prefixing a command with `sudo`).

First run, no passwordless sudo configured yet:
```bash
ansible-playbook -i inventory.ini playbooks/nginx.yml --ask-become-pass
```
This prompts once for the sudo password on the target host(s) before
running.

## Passwordless sudo for automation

For a lab-only user with no sensitive access, passwordless sudo removes the
need for `--ask-become-pass` on every run:

```bash
ansible-playbook -i inventory.ini playbooks/nginx.yml
```

```bash
# on the VM
sudo visudo
# add:
vboxuser ALL=(ALL) NOPASSWD:ALL
```

**Security note (intentional, documented tradeoff):** this is only safe
because these are isolated local lab VMs with no real data or external
exposure. On production infrastructure this would be a real risk — if the
account were ever compromised, an attacker would have immediate root with
no additional barrier.

## See also

- [homelab](https://github.com/pranav10780/homelab) — this project
- [My projects](/projects) — tools I have built
- [Experience](/experience) — my background
