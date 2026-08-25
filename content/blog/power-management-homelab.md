---
title: power-management-homelab
date: 2026-08-25
tags: ["linux", "homelab", "networking"]
---

This blog is part of an ongoing series of my homelab project [homelab](https://github.com/pranav10780/homelab)

# Power Management

## Goal

Bring all three homelab VMs up or down with a single command, instead of
manually starting/stopping each one through the VirtualBox GUI.

## Why two structurally different playbooks

Every other playbook in this repo (`router.yml`, `nginx.yml`,
`monitoring.yml`) connects to a VM **over SSH** and manages something
inside it. Power management can't fully work that way in both directions:

- **Turning a VM off** can be done over SSH — the VM is already running
  and reachable, Ansible just tells it to shut down.
- **Turning a VM on** *cannot* be done over SSH — if the VM is off, there
  is nothing listening on that port to connect to yet. Starting a VM has
  to happen from the **host**, using VirtualBox's own command-line tool,
  `VBoxManage`.

So the two playbooks are almost mirror images of each other in how they
connect.

## `shutdown.yml` — runs over SSH, into each VM

```yaml
---
- name: Shut down all VMs
  hosts: all
  become: yes
  gather_facts: no

  tasks:
    - name: Shut down the machine
      command: shutdown -h now
      async: 1
      poll: 0
      ignore_errors: yes
```

```bash
ansible-playbook -i inventory.ini playbooks/shutdown.yml
```

- **`hosts: all`** — the built-in group meaning every host in the
  inventory (`router`, `server`, `client` together).
- **`gather_facts: no`** — skips the normal "Gathering Facts" step, since
  none of that info is needed before immediately shutting the machine
  down.
- **`shutdown -h now`** — `-h` halts the system, `now` means no delay.
- **`async: 1` / `poll: 0`** — the important part. `shutdown -h now` kills
  the SSH connection Ansible is using *before* it can cleanly report
  "task completed" back — which would normally look like a broken
  connection (a failure) to Ansible, even though the shutdown itself
  worked. `async`/`poll: 0` tells Ansible to fire the command in the
  background and not wait around for a response that's never coming back
  cleanly.
- **`ignore_errors: yes`** — an extra safety net so a connection-teardown
  artifact doesn't show up as a false `failed=1` in the recap.

**Caveat:** `hosts: all` includes `router`, so `server`/`client` lose
DNS/internet the moment `router` shuts down, even mid-shutdown themselves.
Harmless in practice (they're shutting down anyway), but if ordering ever
mattered, this would need splitting into two plays with different
`hosts:` values — same pattern used in `router.yml`.

## `poweron.yml` — runs on the host, via `VBoxManage`

```yaml
---
- name: Power on homelab VMs
  hosts: localhost
  connection: local
  gather_facts: no

  vars:
    homelab_vms:
      - client
      - server
      - router

  tasks:
    - name: Check current state of each VM
      command: VBoxManage showvminfo {{ item }} --machinereadable
      register: vm_info
      changed_when: false
      loop: "{{ homelab_vms }}"

    - name: Start any VM that isn't already running
      command: VBoxManage startvm {{ item.item }} --type headless
      loop: "{{ vm_info.results }}"
      loop_control:
        label: "{{ item.item }}"
      when: "'VMState=\"running\"' not in item.stdout"

    - name: Wait for SSH to come up on each VM
      wait_for:
        host: 127.0.0.1
        port: "{{ item }}"
        delay: 5
        timeout: 90
      loop:
        - 2222   # server
        - 2223   # client
        - 2224   # router
```

```bash
ansible-playbook -i inventory.ini playbooks/poweron.yml
```

- **`hosts: localhost` / `connection: local`** — `VBoxManage` lives on
  the host, not inside any VM. This tells Ansible not to SSH anywhere for
  this play, just run the commands directly on the machine Ansible itself
  is running on.
- **Only `client`, `server`, `router` are listed** — `VBoxManage list vms`
  also shows unrelated VMs (`windows`, a Kali box) on this host; the list
  is scoped deliberately to just the homelab.
- **State check before starting** — `VBoxManage showvminfo --machinereadable`
  prints simple `key="value"` lines; the next task only calls `startvm`
  on a VM whose state isn't already `"running"`, which is what makes
  rerunning the playbook against already-running VMs safe (no duplicate
  start attempts).
- **`changed_when: false`** on the state-check task — it's read-only, so
  it should never itself be reported as a change.
- **`wait_for` at the end** — `startvm` returns almost immediately, before
  the guest OS has actually finished booting and started sshd. Without
  this, running another playbook right after `poweron.yml` could fail to
  connect because the VM is still mid-boot. `wait_for` polls each SSH
  port until something answers (or gives up after `timeout` seconds).

## Quick verification after either playbook

```bash
ansible client,server,router -i inventory.ini -m ping
```
All three replying `pong` confirms the VMs are up, booted, and reachable
— a good sanity check to run right after `poweron.yml` specifically,
since a successful play doesn't by itself guarantee every service inside
each VM has finished starting.

## See also

- [homelab](https://github.com/pranav10780/homelab) — this project
- [My projects](/projects) — tools I have built
- [Experience](/experience) — my background
