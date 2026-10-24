# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

Ansible playbook that hardens and optimizes Ubuntu servers for [Kamal](https://kamal-deploy.org/) deployments. It is Ubuntu-only and targets the `webservers` inventory group.

## Commands

```bash
# Install role/collection dependencies (geerlingguy.swap, community.general)
ansible-galaxy install -r requirements.yml

# Run the playbook against an existing inventory
ANSIBLE_HOST_KEY_CHECKING=False ansible-playbook -i hosts.ini playbook.yml

# Provision Scaleway instances, then run the full playbook on them
ANSIBLE_HOST_KEY_CHECKING=False ansible-playbook scaleway.yml

# Lint (also run in CI on every push/PR via .github/workflows/ansible-lint.yml)
ansible-lint
```

Inventory is created by copying `hosts.ini.example` to `hosts.ini` and filling in host IPs. `hosts.ini` and `roles/scaleway/vars/main.yml` are gitignored.

## Architecture

Two entry-point playbooks:

- **`playbook.yml`** — the main playbook. Runs against the `webservers` group with `gather_facts: false` (facts are gathered explicitly by the `wait_for_connection` role instead, so the play can connect to freshly-booted hosts before they're fully ready). Roles run in this order, and the order matters:
  1. `wait_for_connection` — waits for SSH, then runs `setup` to gather facts.
  2. `packages` — apt upgrade + install base packages (fail2ban, ufw, chrony, duf, htop, unattended-upgrades, etc.); removes snap; cleans up a legacy Docker apt key/repo from `trusted.gpg.d`. fail2ban and chrony are started via handlers.
  3. `docker` — installs Docker CE from Docker's official apt repo.
  4. `tailscale` — installs Tailscale and joins the tailnet with `tailscale_authkey` (from `TAILSCALE_AUTHKEY`) and `tailscale_tags` (default `tag:server`; tagged devices get key expiry disabled). Then it `set_fact`s `ansible_host` to the host's Tailscale IP and waits for SSH there, so every later task, the reboot included, goes over the tailnet. This is what lets a fresh host be listed by its public IP and still survive the lockdown in the same run. It also adds the Tailscale repo to unattended-upgrades' `Origins-Pattern` (its own file in `apt.conf.d`, which apt merges with the stock list); the package restarts `tailscaled` on upgrade.
  5. `cloudflared` — only on hosts with `cloudflared_tunnel_token` (a role default, so it can be set per host in `hosts.ini`). Installs cloudflared, writes the token to a root-only file and runs a remotely-managed tunnel from its own systemd unit, then waits for `/ready` on the pinned metrics address. Hostnames and the origin (`http://localhost:80`, kamal-proxy) are configured in the Cloudflare dashboard, never by Ansible. Like `tailscale`, it adds its repo to unattended-upgrades. The cloudflared package restarts nothing on upgrade, so `cloudflared-upgrade.path` watches `/usr/bin/cloudflared` and `try-restart`s the tunnel when dpkg replaces it.
  6. `deploy_user` — creates the `{{ deploy_user_name }}` user (default `deploy`) and adds it to the `docker` group. Who may SSH in as that user is decided by `deploy_user_authorized_keys`: set it and the playbook writes exactly that list, empty (the default) and it copies root's `authorized_keys` as before. Either way the file is rewritten on every run, so it is managed state, not a place to append a key by hand. Runs before `security` so root SSH access is still guaranteed while the deploy user is being provisioned.
  7. `firewall` — UFW: deny incoming / allow outgoing, SSH allowed in on `tailscale0`. With `firewall_lockdown` (default true) the public 22/80/443 rules are deleted and `DOCKER-USER` drops new traffic on the public interface; it asserts `tailscale0` exists first. With it false, the old public rules (limit 22, allow 80/443) are kept, which is the migration step for a live server. `DOCKER-USER` first lets through traffic arriving from container bridges (`docker+`, `br-+`), because the chain also sees connections containers open themselves; without that, the DROP breaks outbound SMTP, external databases and, with br_netfilter loaded, container-to-container traffic. Everything else new is dropped on every interface (only 80/443 excepted in migration mode). Do not scope the DROP to the default-route interface: published ports would then be open on a provider private network, a second NIC or `tailscale0`.
  8. `security` — hardens `sshd_config` (disables password login, root password login) and configures unattended-upgrades auto-reboot.
  9. `geerlingguy.swap` — external role (from `requirements.yml`) that configures swap.
  10. `reboot_if_needed` — reboots if `/var/run/reboot-required` exists.

- **`scaleway.yml`** — optional. Runs the `scaleway` role on `localhost` to create compute instances via `community.general.scaleway_compute`, dynamically adds their public IPs to the `webservers` group with `add_host`, then `import_playbook: playbook.yml` provisions them in the same run.
- **`tailscale/policy.hujson`** — the tailnet policy the servers are designed around (members may reach `tag:server` on SSH; servers may reach nothing). It is pasted into the Tailscale admin console by hand, not pushed by Ansible, because it replaces the whole tailnet's policy.

### Conventions

- All built-in modules use fully-qualified collection names (`ansible.builtin.*`) — enforced by the `fqcn-builtins` rule in `.ansible-lint`. Match this style in new tasks.
- Service start/restart is done via handlers (`roles/*/handlers/main.yml`) triggered by `notify`, not inline.
- Tunable variables (`security_autoupdate_reboot`, `security_autoupdate_reboot_time`, `deploy_user_name`, `deploy_user_authorized_keys`, `tailscale_authkey`, `tailscale_tags`, `tailscale_hostname`, `firewall_lockdown`, and any `geerlingguy.swap` defaults like `swap_file_size_mb`) are set under `vars:` in `playbook.yml`.
- Boolean tunables are read with `| bool`, since `-e firewall_lockdown=false` arrives as a string.
- Per-host values that must be overridable from inventory (like `cloudflared_tunnel_token`) go in the role's `defaults/main.yml`, because play `vars:` outrank `hosts.ini` host vars.
- Scaleway settings live in `roles/scaleway/vars/main.yml` (copy from `main.yml.example`); this holds the API token and is gitignored.
