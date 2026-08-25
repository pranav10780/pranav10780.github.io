---
title: ssh-key-auth-homelab
date: 2026-07-21
tags: ["linux", "homelab", "networking"]
---

This blog is part of an ongoing series of my homelab project [homelab](https://github.com/pranav10780/homelab)

# SSH Key-Based Authentication

## Goal

Allow the host to SSH into both VMs without a password, since Ansible needs
to connect unattended (playbooks can involve many tasks; a password prompt
per task would make automation unusable).

## Why not the existing personal SSH key

An existing SSH key was already in use, but it was protected with a
passphrase. A passphrase-protected key still requires manual input to
decrypt before use (via `ssh-agent`).

**Solution:** generate a separate, dedicated key pair for the homelab, with
no passphrase, used only for talking to local, non-sensitive lab VMs. The
original passphrase-protected key is kept for anything sensitive
(GitHub, real infrastructure). This limits the damage if the
lab key were ever compromised, while keeping automation frictionless.

## Commands used

```bash
# Generate a dedicated key for the homelab (no passphrase)
ssh-keygen -t ed25519 -f ~/.ssh/id_homelab -C "homelab-ansible"

# Copy the public key to each VM
ssh-copy-id -i ~/.ssh/id_homelab.pub -p 2222 vboxuser@127.0.0.1   # server
ssh-copy-id -i ~/.ssh/id_homelab.pub -p 2223 vboxuser@127.0.0.1   # client
```

## How it works

- `ssh-keygen` generates a private key (`id_homelab`, stays on host, never
  shared) and a public key (`id_homelab.pub`, safe to distribute).
- `ssh-copy-id` appends the public key to `~/.ssh/authorized_keys` on the
  target VM — the list of keys allowed to log in as that user.
- On future connections, SSH proves possession of the private key
  cryptographically, without ever transmitting it — no password exchanged
  at all.

## Verification

```bash
ssh -p 2222 vboxuser@127.0.0.1
```

Expected: logs straight in, no password prompt.

## Wiring this into Ansible

`ansible/inventory.ini` points each host at the dedicated key:

```ini
[server]
server ansible_host=127.0.0.1 ansible_port=2222 ansible_user=vboxuser ansible_ssh_private_key_file=~/.ssh/id_homelab

[clients]
client ansible_host=127.0.0.1 ansible_port=2223 ansible_user=vboxuser ansible_ssh_private_key_file=~/.ssh/id_homelab
```

## See also

- [homelab](https://github.com/pranav10780/homelab) — this project
- [My projects](/projects) — tools I have built
- [Experience](/experience) — my background
