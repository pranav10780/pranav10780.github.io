---
title: nginx-playbook-homelab
date: 2026-07-21
tags: ["linux", "homelab", "networking"]
---

This blog is part of an ongoing series of my homelab project [homelab](https://github.com/pranav10780/homelab)

# Nginx Ansible Playbook

## Goal

Automate the already-working manual Nginx setup on `server` into an Ansible
playbook, so the configuration is reproducible rather than relying on
one-off manual commands.

## Playbook: `ansible/playbooks/nginx.yml`

```yaml
---
- name: Configure server with nginx
  hosts: server
  become: yes

  tasks:
    - name: Install nginx
      apt:
        name: nginx
        state: present
        update_cache: yes

    - name: Ensure nginx is running and enabled on boot
      service:
        name: nginx
        state: started
        enabled: yes

    - name: Deploy custom index page
      copy:
        src: files/index.html
        dest: /var/www/html/index.html
```

## What each part does

- `hosts: server` — applies only to the `[server]` group in the inventory.
- `become: yes` — run these tasks with sudo.
- `apt` module — installs the package via Ubuntu's package manager;
  idempotent (does nothing if already installed).
- `service` module — ensures the systemd service is running now (`started`)
  and will start on boot (`enabled`); idempotent.
- `copy` module — pushes a local file (`playbooks/files/index.html`) to the
  remote path; only changes the remote file if content differs.

## Running it

```bash
ansible-playbook -i inventory.ini playbooks/nginx.yml
```

## Idempotency check

Running the same playbook twice in a row should show `changed=0` on the
second run for every task, since the desired state already matches reality:

```
PLAY RECAP *********************************************************
server     : ok=4    changed=0    unreachable=0    failed=0
```

## See also

- [homelab](https://github.com/pranav10780/homelab) — this project
- [My projects](/projects) — tools I have built
- [Experience](/experience) — my background
