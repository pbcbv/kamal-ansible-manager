# Kamal Ansible Manager

An Ansible playbook that prepares Ubuntu servers for [Kamal](https://kamal-deploy.org/) deployments and takes them off the public internet. After a run, SSH is reachable only over [Tailscale](https://tailscale.com/), the apps only through a [Cloudflare Tunnel](https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/), and nothing answers on the server's public IP.

Tested on Ubuntu 24.04 with ansible-core 2.22. Forked from [guillaumebriday/kamal-ansible-manager](https://github.com/guillaumebriday/kamal-ansible-manager), whose author walks through the original in [this video](https://www.youtube.com/watch?v=WTYNPCEJSOo).

## Quick start

This assumes a tailnet with the policy from [`tailscale/policy.hujson`](tailscale/policy.hujson) in place, a Cloudflare zone for your domain, and a machine on that tailnet to run Ansible from. If that is not set up yet, start with [One-time setup](#one-time-setup).

1. **Install Ansible and the dependencies.**

   ```sh
   pipx install --include-deps ansible
   git clone git@github.com:pbcbv/kamal-ansible-manager.git
   cd kamal-ansible-manager
   ansible-galaxy install -r requirements.yml
   ```

2. **Create the server** at your provider with your SSH key in root's `authorized_keys`. Leave the provider firewall open for now: the first run connects over public SSH. Right after the server reports as running, sshd can refuse connections for a few seconds; wait until `nc -vz -w5 <public-ip> 22` succeeds.

3. **Get the two tokens.**
   - Tailscale admin console > Settings > Keys > Generate auth key: tagged `tag:server`, single-use, not ephemeral, pre-approved, expiry one day.
   - Cloudflare dashboard > Networking > Tunnels > Create a tunnel > Cloudflared. Name it after the server and copy only the token from the install command (the `eyJ...` part). Don't run that command; the playbook installs cloudflared.

   Keep both out of the repository, your shell history and chat, in a file only you can read:

   ```sh
   mkdir -m 700 -p ~/.config/kamal-lockdown
   install -m 600 /dev/null ~/.config/kamal-lockdown/app-1.env
   # then edit it to hold:
   #   TAILSCALE_AUTHKEY=tskey-auth-...
   #   CLOUDFLARED_TUNNEL_TOKEN=eyJ...
   ```

4. **Write the inventory.** `hosts.ini` is ignored by git.

   ```ini
   [webservers:vars]
   ansible_become_method=su
   ansible_user=root

   [webservers]
   app-1 ansible_host=203.0.113.10 cloudflared_tunnel_token="{{ lookup('env', 'CLOUDFLARED_TUNNEL_TOKEN') }}"
   ```

5. **Run the playbook.**

   ```sh
   set -a; . ~/.config/kamal-lockdown/app-1.env; set +a
   ANSIBLE_HOST_KEY_CHECKING=False ansible-playbook -i hosts.ini playbook.yml
   ```

   Partway through, the play joins the tailnet, moves its own SSH connection to the server's Tailscale address and checks that it works. Only then does it close the public ports. Expect a few minutes for a new server.

6. **Route the hostname.** Back in the tunnel, add a published application route: your hostname, Service URL `http://localhost:80`. The dashboard offers this step only once the connector is connected, which the playbook just did. Cloudflare creates the proxied DNS record itself.

7. **Point the inventory at the tailnet.** Remove `ansible_host` from the line in `hosts.ini`, so `app-1` resolves to its Tailscale (MagicDNS) name. The public IP no longer answers SSH.

8. **Close the provider firewall.** Give the server a provider firewall with no inbound rules (see [Provider firewall](#provider-firewall)).

9. **Deploy** with the Kamal settings in [Kamal configuration](#kamal-configuration), then run the checks in [Verifying a server](#verifying-a-server).

## What the playbook does

Roles run in this order, against the `webservers` group:

| Role | What it does |
| --- | --- |
| `wait_for_connection` | Waits for SSH, then gathers facts. The play skips the automatic fact gathering so it can reach a server that is still booting. |
| `packages` | Upgrades all packages, installs the base set (fail2ban, ufw, chrony, unattended-upgrades and a few tools), removes snap. |
| `docker` | Installs Docker CE from Docker's apt repository. |
| `tailscale` | Installs Tailscale, joins the tailnet with `TAILSCALE_AUTHKEY` and `tailscale_tags`, then moves the play's connection onto the Tailscale address. A server already on the tailnet needs no key. |
| `cloudflared` | Only on hosts with a `cloudflared_tunnel_token`. Installs cloudflared, writes the token to a root-only file, runs the tunnel from its own systemd unit and waits until it reports a connection. |
| `deploy_user` | Creates the `deploy` user in the `docker` group, so Kamal does not deploy as root. Its `authorized_keys` is copied from root unless you set `deploy_user_authorized_keys`. |
| `firewall` | ufw denies everything incoming except SSH on `tailscale0`, and Docker's `DOCKER-USER` chain drops every new connection to a container that a container did not start. With `firewall_lockdown=false`, public 22, 80 and 443 stay open. It refuses to close public SSH if `tailscale0` does not exist. |
| `security` | Disables SSH password login, enables the fail2ban `sshd` jail (the tailnet is never banned) and, when `security_autoupdate_reboot` is on, sets when unattended-upgrades may reboot. |
| `geerlingguy.swap` | Creates a swap file, 512 MB by default. |
| `reboot_if_needed` | Reboots when an update asks for it. The reboot goes over the tailnet. |

Tailscale and cloudflared are the two services left facing the network, so both roles add their apt repository to unattended-upgrades, which otherwise only installs Ubuntu's own updates. Fixes for them land within a day. Tailscale's package restarts itself; for cloudflared a systemd path unit restarts the tunnel when its binary is replaced. Docker is left out on purpose: an upgrade restarts every container, so it comes in when you run the playbook.

## How the lockdown works

SSH keys stop password guessing. They do nothing against a bug in `sshd`, in Docker's port publishing or in anything else listening on a public port, so the playbook leaves nothing listening on one:

- **Server access** goes over Tailscale, a private WireGuard network of your own devices.
- **App traffic** goes through a Cloudflare Tunnel, with Cloudflare's proxy and WAF in front of it.

Both open their connections outbound from the server, so every inbound connection on the public side can be dropped. Three layers do that:

1. **The provider firewall** with no inbound rules drops packets before they reach the server.
2. **ufw** denies all incoming traffic except SSH arriving on `tailscale0`.
3. **Docker's `DOCKER-USER` chain** closes the hole Docker normally opens past ufw when it publishes a container port. It lets through what containers start themselves (`docker+` and `br-+` interfaces) and drops every other new connection, on every interface, for IPv4 and IPv6. Containers can still reach the internet and each other.

Because that drop covers `tailscale0` too, published ports are not reachable over the tailnet either: `curl http://100.x.y.z` gets no answer, by design. Test through the tunnel, or SSH in and use `localhost`. cloudflared reaches kamal-proxy on `localhost:80`, which the rules don't touch.

The role trusts the `docker` and `br-` interface prefixes, so it first checks that every interface with those names is one Docker created. A host bridge called `br-lan` stops the run; rename it before running.

## One-time setup

### The machine you run Ansible from

It needs Ansible (`pipx install --include-deps ansible`), `ansible-galaxy install -r requirements.yml` once per checkout, an SSH key in root's `authorized_keys` on the servers, and Tailscale, logged in to the tailnet (`tailscale status` shows it). Turn on MagicDNS in the Tailscale admin console so servers can be reached by name.

### Tailnet policy

A new tailnet lets every device reach every other device. [`tailscale/policy.hujson`](tailscale/policy.hujson) replaces that: your devices reach each other, members reach `tag:server` hosts on SSH and ping, and servers may open connections to nothing. A compromised server gives no way into the rest of the tailnet.

1. Check the Machines page for exit nodes, subnet routers and shared devices first. The policy does not allow them; add grants for any you use.
2. Admin console > Access controls > JSON editor. Replace a default policy with the file; merge by hand if yours has customizations. ("Policy file management" under Settings is the lock for managing the policy from git, not the editor.)
3. In the `tests` section, uncomment the two tests and fill in your login email. They prove you can reach the servers and the servers can't reach your devices, and the console runs them on every save. Keep that filled-in copy out of the repository.

The policy is applied by hand because it replaces the whole tailnet's policy; a provisioning run should not be able to do that.

### Tailscale auth keys

Generate one per server, just before provisioning: tagged `tag:server`, single-use, not ephemeral, pre-approved, expiring in a day. A tagged device is owned by the tag instead of a person, and Tailscale disables key expiry for it, so the server does not drop off the tailnet after 180 days. For a batch of servers a reusable key is fine; revoke it when the batch is done.

### Provider firewall

Do this last, after SSH over the tailnet works.

- **Hetzner Cloud:** Firewalls > Create Firewall, delete the default inbound rules so there are none, leave outbound empty (everything allowed), apply it to the server.
- **Scaleway:** a security group with inbound policy `drop`, outbound `accept`, stateful on and no rules.

With it on, Tailscale may relay through its DERP servers instead of connecting directly. That is fine for SSH and deploys; `tailscale ping app-1` shows which path is in use.

## Configuration

Set variables under `vars:` in `playbook.yml`. Pass a boolean on the command line with `-e`, for example `-e firewall_lockdown=false`.

| Variable | Default | Purpose |
| --- | --- | --- |
| `tailscale_authkey` | `TAILSCALE_AUTHKEY` from the environment | Only needed the first time a host joins the tailnet. |
| `tailscale_tags` | `["tag:server"]` | Tags the host joins with. The policy is written around `tag:server`. |
| `tailscale_hostname` | the host's hostname | Its machine and MagicDNS name on the tailnet. |
| `firewall_lockdown` | `true` | Close every public port. `false` keeps public 22, 80 and 443 open, for migrating a live server. |
| `deploy_user_name` | `deploy` | The user Kamal deploys as. |
| `deploy_user_authorized_keys` | `[]` | When set, written verbatim to that user's `authorized_keys` on every run. When empty, root's keys are copied. |
| `security_autoupdate_reboot` | `"false"` | Let unattended-upgrades reboot when an update needs it. |
| `security_autoupdate_reboot_time` | `"03:00"` | When that reboot happens. |
| `swap_file_size_mb` | `'512'` | Any [geerlingguy.swap](https://github.com/geerlingguy/ansible-role-swap/blob/master/defaults/main.yml) variable can be set the same way. |

`cloudflared_tunnel_token` is set per host in `hosts.ini` rather than in the playbook, because play variables would override it. Hosts without one, such as workers, get no tunnel. Two hosts with the same token become replicas of one tunnel.

## Kamal configuration

In each app's `config/deploy.yml`:

```yaml
servers:
  web:
    - app-1              # Tailscale name, not the public IP

ssh:
  user: deploy

proxy:
  host: app.example.com
  ssl: false             # Cloudflare terminates TLS; kamal-proxy serves HTTP for cloudflared
  forward_headers: true
  run:                   # one kamal-proxy per server: identical in every app on it
    bind_ips:
      - 127.0.0.1
    http_port: 80
    https_port: 8443
```

- **`ssl: false`.** Visitors get Cloudflare's edge certificate and the tunnel is encrypted, so there is no certificate on the server. Let's Encrypt can't work anyway, since nothing reaches port 80 from outside. Rails with `config.assume_ssl = true` and `config.force_ssl = true` keeps generating `https://` URLs and secure cookies.
- **`bind_ips: 127.0.0.1`.** kamal-proxy is published on the loopback only, which is all cloudflared needs, so its ports stay closed on the public IP even with the lockdown off.
- **Several apps on one server.** They share one tunnel (one route per hostname, all to `http://localhost:80`) and one kamal-proxy, which routes by host. The `run` block configures that shared proxy, so every app must carry the same values, or a deploy of one app reconfigures the proxy for all of them.
- **The visitor's IP.** It arrives in `X-Forwarded-For` and `CF-Connecting-IP`. After the first deploy, check that `request.remote_ip` shows real visitor addresses, not `127.0.0.1` or a Cloudflare address; rate limits and logs depend on it.
- **Webhooks.** Cloudflare's Bot Fight Mode and WAF challenges can block them. Test each webhook through the tunnel and add a WAF skip rule for its path if needed.
- **Deploying from CI.** A CI runner is not on the tailnet. Uncomment the `tag:ci` lines in the policy, create a Tailscale OAuth client for that tag and join the runner with `tailscale/github-action` before the deploy step.

## A tailnet-only app

An app that must never be public, such as one holding private finances, gets no route in the tunnel. HTTPS ends on the server at `tailscale serve`, with a certificate for the machine's tailnet name, and only devices the policy names can open it. It can share a server, and its kamal-proxy, with public apps.

1. **The policy.** Give the server a tag of its own next to `tag:server`, say `tag:app`, and grant only the people who use the app port 443 on it. Save the policy before tagging: a tag can be applied only once it is in `tagOwners`.

   ```hujson
   "tagOwners": { "tag:server": [], "tag:app": [] },
   "grants": [
   	{ "src": ["you@example.com"], "dst": ["tag:app"], "ip": ["tcp:443"] },
   ],
   "tests": [
   	{ "src": "you@example.com", "accept": ["tag:app:443"] },
   	{ "src": "tag:server", "deny": ["tag:app:443"] },
   ],
   ```

   Then Machines > the server > Edit ACL tags: add `tag:app` and keep `tag:server`, which the SSH grant needs. Or pass both in `tailscale_tags` when provisioning.

2. **HTTPS on the tailnet.** On the server:

   ```sh
   tailscale serve --bg --https=443 http://127.0.0.1:80
   ```

   The first time on a tailnet it stops with "Serve is not enabled on your tailnet" and a link, and waits. The page behind it enables HTTPS certificates and offers Tailscale Funnel, **ticked by default. Untick Funnel**: it publishes nodes on the internet and edits the policy to let every member do so. Apps behind the tunnel never needed this step, because Cloudflare holds their certificate. `tailscale serve status` then shows the name, `<server>.<tailnet>.ts.net`, with "(tailnet only)".

   A certificate writes that full name to the public Certificate Transparency logs, permanently. It reveals the name, not a way in.

3. **No firewall rule.** tailscaled's own `ts-input` chain comes first in `INPUT` and accepts everything on `tailscale0` before ufw sees it (`iptables -S INPUT` starts with `-j ts-input`). The policy is the only gate, so the grant above is what keeps everyone else out. Until it exists, the page times out even from your own devices.

4. **Kamal.** The same `ssh`, `ssl: false` and `run` block as above. The app gets every request kamal-proxy has no other host for: give it `proxy.host` with the tailnet name, or, if that name must stay out of the repository, leave `host` out and have the app refuse every other name (Rails: `config.hosts`). Either way a tunnel route added by mistake gets an error, not the app.

5. **Check** from a device the grant names, before the first deploy: `curl -sI https://<server>.<tailnet>.ts.net/up` answers with a 404 from kamal-proxy, which has no app for that name yet. A timeout means the grant or the tag is missing. From any other device it must time out.

Apps on one server share one Docker network, so a public app that is taken over can open connections to the tailnet-only app's container directly. That app's own login is what stands there.

## Moving a live server

On a server that already serves traffic on its public IP, the tunnel has to carry the traffic before ports 80 and 443 close:

1. Create the tunnel and add the token to `hosts.ini`, but no route yet if the hostname already has a DNS record: Cloudflare refuses a route for a hostname that has one.
2. Run with the lockdown off. The server joins the tailnet and the tunnel connects; every public port stays open.

   ```sh
   ansible-playbook -i hosts.ini playbook.yml -e firewall_lockdown=false
   ```

3. Switch `hosts.ini` to the Tailscale name and check `ssh root@app-1` works.
4. Deploy the app with the Kamal settings above. Then, right away, delete the hostname's A/AAAA record and add the route in the tunnel. Between the deploy and the new route, visitors on the old record get errors, so do both outside busy hours.
5. Check the site through the tunnel, including logins, uploads and webhooks.
6. Run again with the lockdown on (`ansible-playbook -i hosts.ini playbook.yml`), then add the provider firewall.

## Verifying a server

From the machine you run Ansible from:

```sh
ssh root@app-1 'tailscale status | head -3'                # SSH over the tailnet works
nc -vz -w5 <public-ip> 22                                  # expect: timed out (also 80, 443)
curl -sI https://app.example.com/up | grep -iE '^(HTTP|server)'   # 200, server: cloudflare
```

"Connection refused" from `nc` means something answered with a reset; the firewall should drop silently. If your machine has no IPv6, test the IPv6 address from a network that has it, or check the server's rules below.

On the server:

```sh
ufw status verbose                       # only 22/tcp on tailscale0
iptables -S DOCKER-USER                  # ends in -j DROP (ip6tables too)
docker run --rm alpine:3.20 nc -zw4 smtp.gmail.com 587 && echo outbound-ok
systemctl is-active tailscaled cloudflared
curl -s http://127.0.0.1:20241/ready     # {"status":200,"readyConnections":4,...}
unattended-upgrade --dry-run --debug 2>&1 | grep 'Allowed origins'   # includes Tailscale and cloudflared
nc -vz -w5 <your-tailnet-ip> 22          # expect: timed out, servers can't reach your devices
```

In the Tailscale admin console the server shows `tag:server` and "Expiry disabled".

## Keeping servers current

- **Monthly:** run the playbook against all servers. It upgrades every package, Docker included, and puts back anything that drifted. A second run in a row should report no changes.
- **New server:** follow the quick start with a fresh auth key.
- **Rotating a tunnel token:** rotate it in Cloudflare, update the env file and run the playbook. The role rewrites the token file and restarts cloudflared.
- **Removing a server:** delete it in the Tailscale admin console, delete its tunnel (and any leftover DNS records) in Cloudflare, and remove it from `hosts.ini`.
- **A security advisory for Tailscale or cloudflared:** don't wait for the daily timer; run `unattended-upgrade` on each server or run the playbook.

## When you can't get in

With public SSH closed, a Tailscale problem becomes an access problem. In order of likelihood:

1. **A policy mistake.** Fix it in the Tailscale admin console from any device.
2. **Your machine is off the tailnet.** `tailscale up`, then `tailscale status`.
3. **The server is off the tailnet.** Use the provider's web console. Hetzner's needs a root password, which key-only servers don't have, so reset it from the Rescue tab first. If you need SSH from rescue, temporarily allow your own IP in the provider firewall. Once in: `systemctl restart tailscaled`. If the machine was removed from the tailnet, join again with a fresh tagged key: `tailscale up --auth-key=tskey-auth-... --advertise-tags=tag:server`.
4. **Last resort:** temporarily allow SSH from your IP in the provider firewall and run with `-e firewall_lockdown=false`. Close it again afterwards.

## Scaleway (optional)

`scaleway.yml` creates instances with [community.general.scaleway_compute](https://docs.ansible.com/ansible/latest/collections/community/general/scaleway_compute_module.html) and then runs the main playbook on them.

```sh
cp roles/scaleway/vars/main.yml.example roles/scaleway/vars/main.yml   # holds the API token; ignored by git
set -a; . ~/.config/kamal-lockdown/app-1.env; set +a
ANSIBLE_HOST_KEY_CHECKING=False ansible-playbook scaleway.yml
```

The new instances have no tunnel token yet. Add the web hosts to `hosts.ini` with one and run `playbook.yml` again.

## Development

Run `ansible-lint` before pushing; CI runs it on every push and pull request. Conventions (fully qualified module names, handlers for service restarts, per-host values in role defaults) are in [CLAUDE.md](CLAUDE.md).

## License

[MIT](http://opensource.org/licenses/MIT).
