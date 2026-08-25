---
title: vm-setup-homelab
date: 2026-07-21
tags: ["linux", "homelab", "networking"]
---

This blog is part of an ongoing series of my homelab project [homelab](https://github.com/pranav10780/homelab)

# VM Setup

Create two Debian Server VMs (`server`, `client`) in VirtualBox that can
communicate with each other, and are reachable via SSH from the host machine.

## Environment
 
| | |
|---|---|
| Hypervisor | Oracle VirtualBox |
| Guest OS | Debian 13 "Trixie" (64-bit) |
| Install media | `debian-13.6.0-amd64-netinst.iso` |
| Base memory | 2048 MB per VM |
| Disk | 20.00 GB (SATA, `client.vdi` / `server.vdi`) |
| Graphics | VMSVGA, 16 MB video memory |
| Network adapter | Intel PRO/1000 MT Desktop |
| Network mode | NAT Network (VirtualBox), not plain NAT or Bridged |

## What was done

- Created `client` first as a fresh Debian 13 VM (no desktop environment —
  command-line only).
- Cloned `client` in VirtualBox and renamed the clone to `server`, to save
  time re-running the OS install.
- Created a VirtualBox **NAT Network** named `homelab` (rather than each VM
  using its own isolated default NAT) so both VMs share a network and can
  reach each other directly:
  - IPv4 prefix: `10.0.2.0/24`
  - DHCP server: enabled
- Both VMs attached to this `homelab` NAT Network, with DHCP assigning:
  - `client` -> `10.0.2.4`
  - `server` -> `10.0.2.5`
- Configured port forwarding on the `homelab` NAT Network so the host can
  reach each VM over SSH (see table below).
- Verified connectivity between VMs and from host to each VM over SSH.
- Installed and started Nginx on `server`; confirmed it serves a page and
  is managed correctly via `systemctl`.
## VirtualBox port forwarding

Configured under *VirtualBox Manager -> Network -> NAT Networks -> homelab -> 
Port Forwarding*:
 
| Name   | Protocol | Host IP | Host Port | Guest IP  | Guest Port |
|--------|----------|---------|-----------|-----------|------------|
| client | TCP      | (any)   | 2223      | 10.0.2.4  | 22         |
| server | TCP      | (any)   | 2222      | 10.0.2.5  | 22         |
 
This lets `ssh -p 2222 user@127.0.0.1` from the host reach `server`, and
`ssh -p 2223 user@127.0.0.1` reach `client`, without either VM needing to
be on a network routable from outside VirtualBox.
 
**Why a NAT Network instead of default per-VM NAT:** default NAT is
isolated per VM — two VMs on default NAT can't see each other at all. A
NAT Network is a shared virtual switch with its own DHCP, so VMs attached
to it get real IPs on the same subnet and can talk directly, while still
sharing the host's internet connection outward.
 
## Issue: cloning a VM duplicates its SSH host key

Because `server` was made by cloning `client` rather than installing
Ubuntu fresh, both VMs initially had **identical SSH host keys**
(`/etc/ssh/ssh_host_*_key`). Host keys are the machine's own identity —
generated once at first boot — separate from the personal user keys used to
log in (`id_homelab`, etc.).
`WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED` errors on the host
whenever switching between them (since both live at `127.0.0.1`, just on
different ports).

**Fix — regenerate host keys on the cloned VM:**
```bash
# on server
sudo rm /etc/ssh/ssh_host_*
sudo ssh-keygen -A
sudo systemctl restart ssh
```
`ssh-keygen -A` regenerates the full set of default host keys. After this,
`server` has its own unique identity, distinct from `client`.

**Lesson:** cloning a VM is fine for saving OS-install time, but always
regenerate the host keys (and double check `/etc/hostname` and `/etc/hosts`
are updated too) — otherwise the clone is indistinguishable from the
original at the network identity level.

## Verification

```bash
# From host, confirm SSH access to each VM
ssh -p 2222 vboxuser@127.0.0.1     # server
ssh -p 2223 vboxuser@127.0.0.1     # client

# On server, confirm nginx is active
sudo systemctl status nginx.service
```

Expected: `Active: active (running)`.

## Notes

- `/usr/sbin` (where nginx and other admin binaries live) is not on a
  regular user's `$PATH` by default on Debian — only relevant when calling
  binaries directly. Managing services via `systemctl` avoids this entirely.

## See also

- [homelab](https://github.com/pranav10780/homelab) — this project
- [My projects](/projects) — tools I have built
- [Experience](/experience) — my background
