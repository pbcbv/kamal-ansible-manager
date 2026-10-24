# Kamal Ansible Manager

Ansible playbook to automatically optimize and secure servers for [Kamal](https://kamal-deploy.org/). Ubuntu only.

Forked from [guillaumebriday/kamal-ansible-manager](https://github.com/guillaumebriday/kamal-ansible-manager) — here's the original author's video walkthrough of how the repo works:

[![Watch the video](https://raw.githubusercontent.com/guillaumebriday/kamal-ansible-manager/main/thumbnail.png)](https://www.youtube.com/watch?v=WTYNPCEJSOo)

## What's inside?

It will automatically update your packages and configure these packages to secure your server(s):

- [Docker](https://docs.docker.com/engine/install/ubuntu/)
- [Fail2ban](https://github.com/fail2ban/fail2ban), with the `sshd` jail enabled and tuned (`jail.local`)
- [UFW](https://wiki.ubuntu.com/UncomplicatedFirewall), including a `DOCKER-USER` firewall rule so containers can't bypass UFW by publishing ports directly (see [Docker's docs](https://docs.docker.com/engine/network/packet-filtering-firewalls/#docker-and-ufw))
- [Chrony](https://ubuntu.com/server/docs/network-ntp) (NTP time synchronization)
- [Tailscale](https://tailscale.com/), joined with a `tag:server` auth key, as the only way to SSH in
- [cloudflared](https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/), as the only way web traffic gets in

The playbook also:
- Remove [Snap](https://snapcraft.io/).
- Disable ssh password login.
- Create a non-root `deploy` user (configurable via `deploy_user_name`) with `docker` group membership, so Kamal doesn't need to deploy as root. It gets root's SSH keys by default, or an explicit set if you list them in `deploy_user_authorized_keys`.
- Configure `swap` using [geerlingguy/ansible-role-swap](https://github.com/geerlingguy/ansible-role-swap).
- Close every public port, SSH included (see [Network lockdown](#network-lockdown-tailscale-and-cloudflare-tunnel)).

## Getting Started

Clone the repo:
```bash
$ git clone git@github.com:pbcbv/kamal-ansible-manager.git
$ cd kamal-ansible-manager
```

Copy the inventory example file:
```bash
$ cp hosts.ini.example hosts.ini
```

Update the `<host1>` with your server's IP address (you can have multiple servers):
```bash
$ vim hosts.ini
```

Install the requirements:
```bash
$ ansible-galaxy install -r requirements.yml
```

The machine you run Ansible from has to be on your tailnet, and the tailnet
needs the policy from [`tailscale/policy.hujson`](tailscale/policy.hujson).
See the next section.

## Network lockdown: Tailscale and Cloudflare Tunnel

SSH keys protect against password guessing, not against a bug in `sshd`, in
Docker's port publishing or in anything else listening on a public port. So the
playbook leaves nothing listening on one. There are two ways into a server and
each gets its own path:

- **The server itself (SSH)** is reachable only over [Tailscale](https://tailscale.com/),
  a private WireGuard network of your own devices.
- **The app (HTTP)** is reachable only through a [Cloudflare Tunnel](https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/),
  which puts Cloudflare's proxy and WAF in front of it.

Both connect *outbound* from the server, so the firewall can deny every
incoming connection on the public interface. Port scans of the public IP find
nothing.

### What the playbook does

- `tailscale` installs Tailscale from its apt repo, joins the tailnet with
  `TAILSCALE_AUTHKEY` and the tags in `tailscale_tags` (default `tag:server`),
  then **moves its own SSH connection onto the host's Tailscale IP** and checks
  it works. A fresh host can therefore be listed by its public IP for its first
  run. Hosts that join with a tagged key have key expiry disabled by Tailscale,
  so they don't drop off the tailnet after 180 days.
- `cloudflared` installs cloudflared from Cloudflare's apt repo and runs the
  tunnel whose token is in `cloudflared_tunnel_token`, set per host in
  `hosts.ini`. Hosts without a token (workers) get no tunnel. It waits for the
  tunnel to report a live connection to Cloudflare.
- `firewall` with `firewall_lockdown: true` (the default) deletes the public
  22/80/443 rules, allows SSH only in on `tailscale0`, and makes Docker's
  `DOCKER-USER` chain drop every new connection to a container that doesn't
  start at a container itself, on any interface. Containers can still reach
  the internet and each other. It refuses to run if `tailscale0` doesn't exist.
- Both `tailscale` and `cloudflared` add their apt repo to unattended-upgrades,
  which otherwise only installs Ubuntu's own updates, so fixes for the two
  services left facing the network land within a day. Tailscale's package
  restarts `tailscaled` itself; cloudflared's doesn't, so a systemd path unit
  restarts the tunnel whenever its binary is replaced.

### What you do once, by hand

These live in the Tailscale and Cloudflare dashboards and the playbook does not
touch them:

1. **Tailnet policy.** Paste [`tailscale/policy.hujson`](tailscale/policy.hujson)
   into the Tailscale admin console (Access controls). It replaces the
   allow-everything default: your own devices may reach `tag:server` hosts on
   SSH, and servers may not open connections to anything. A compromised server
   gives no way into the rest of your tailnet.
2. **Auth key.** Settings > Keys > Generate auth key: reusable, pre-approved,
   tagged `tag:server`, not ephemeral. Export it as `TAILSCALE_AUTHKEY` when you
   provision a new host.
3. **Tunnel.** Zero Trust > Networks > Tunnels > Create a tunnel (cloudflared).
   Copy its token into `hosts.ini` for that host and add a published
   application route per hostname, pointing at `http://localhost:80`
   (kamal-proxy).
4. **Provider firewall.** Once the host works over the tailnet and the tunnel,
   give it a provider-level firewall with no inbound rules (Hetzner Cloud
   Firewall, Scaleway security group with inbound policy `drop`). Tailscale and
   cloudflared keep working because they only connect outbound.

### Running it

New host:
```bash
$ TAILSCALE_AUTHKEY=tskey-auth-... ANSIBLE_HOST_KEY_CHECKING=False ansible-playbook -i hosts.ini playbook.yml
```

Existing host that is still serving traffic on its public IP: run once with the
lockdown off, move its DNS to the tunnel and check the site, then run again
with the default:
```bash
$ TAILSCALE_AUTHKEY=tskey-auth-... ansible-playbook -i hosts.ini playbook.yml -e firewall_lockdown=false
$ ansible-playbook -i hosts.ini playbook.yml
```

After the first run, change the host in `hosts.ini` to its Tailscale name (or
`100.x` address); its public IP no longer answers SSH.

### Kamal configuration

In each app's `config/deploy.yml`:

```yml
servers:
  web:
    - app-1            # Tailscale name, not the public IP

ssh:
  user: deploy

proxy:
  host: app.example.com
  ssl: false           # Cloudflare terminates TLS; kamal-proxy serves plain HTTP on :80
```

With `ssl: false`, kamal-proxy forwards `X-Forwarded-Proto` and
`X-Forwarded-For`, so Rails with `config.assume_ssl = true` and
`config.force_ssl = true` keeps working. Let's Encrypt can't be used anymore
because nothing reaches port 80 from outside. A deploy from CI needs the runner
on the tailnet too (see the commented `tag:ci` rule in the policy file).

## Configuring vars

Variables can be configured in the `playbook.yml` file.
Also, you can override default variables provided in [geerlingguy/ansible-role-swap](https://github.com/geerlingguy/ansible-role-swap/blob/master/defaults/main.yml) to adjust the swap settings.

For instance:
```yml
  vars:
    security_autoupdate_reboot: "true"
    security_autoupdate_reboot_time: "03:00"
    tailscale_tags: ["tag:server"]
    firewall_lockdown: true
    swap_file_size_mb: '1024'
```

## Running the playbook

Run the playbook:
```bash
$ ANSIBLE_HOST_KEY_CHECKING=False ansible-playbook -i hosts.ini playbook.yml
```

## Provisioning Servers with Scaleway (Optional)

If you want to automatically create new compute instances on Scaleway, you can use the [community.general.scaleway_compute module](https://docs.ansible.com/ansible/latest/collections/community/general/scaleway_compute_module.html). Follow these steps:

Copy the example variables file and adjust the variables as needed:
```bash
$ cp roles/scaleway/vars/main.yml.example roles/scaleway/vars/main.yml
```

Run the playbook:
```bash
$ ANSIBLE_HOST_KEY_CHECKING=False ansible-playbook scaleway.yml
```

Then, it will continue the provisioning process on the newly created servers.
Export `TAILSCALE_AUTHKEY` first. The new instances have no tunnel token yet, so
add them to `hosts.ini` with one and run `playbook.yml` again for the web hosts.

## License

This project is released under the [MIT](http://opensource.org/licenses/MIT) license.
