# Kamal Ansible Manager

Peter's private Ansible playbook to automatically optimize and secure servers for [Kamal](https://kamal-deploy.org/) across all `pbcbv` projects. Ubuntu only.

Forked from [guillaumebriday/kamal-ansible-manager](https://github.com/guillaumebriday/kamal-ansible-manager) — here's the original author's video walkthrough of how the repo works:

[![Watch the video](https://raw.githubusercontent.com/guillaumebriday/kamal-ansible-manager/main/thumbnail.png)](https://www.youtube.com/watch?v=WTYNPCEJSOo)

## What's inside?

It will automatically update your packages and configure these packages to secure your server(s):

- [Docker](https://docs.docker.com/engine/install/ubuntu/)
- [Fail2ban](https://github.com/fail2ban/fail2ban), with the `sshd` jail enabled and tuned (`jail.local`)
- [UFW](https://wiki.ubuntu.com/UncomplicatedFirewall), including a `DOCKER-USER` firewall rule so containers can't bypass UFW by publishing ports directly (see [Docker's docs](https://docs.docker.com/engine/network/packet-filtering-firewalls/#docker-and-ufw))
- [Chrony](https://ubuntu.com/server/docs/network-ntp) (NTP time synchronization)

The playbook also:
- Remove [Snap](https://snapcraft.io/).
- Disable ssh password login.
- Create a non-root `deploy` user (configurable via `deploy_user_name`) with `docker` group membership, so Kamal doesn't need to deploy as root. It gets root's SSH keys by default, or an explicit set if you list them in `deploy_user_authorized_keys`.
- Configure `swap` using [geerlingguy/ansible-role-swap](https://github.com/geerlingguy/ansible-role-swap).

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

## Configuring vars

Variables can be configured in the `playbook.yml` file.
Also, you can override default variables provided in [geerlingguy/ansible-role-swap](https://github.com/geerlingguy/ansible-role-swap/blob/master/defaults/main.yml) to adjust the swap settings.

For instance:
```yml
  vars:
    security_autoupdate_reboot: "true"
    security_autoupdate_reboot_time: "03:00"
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

## License

This project is released under the [MIT](http://opensource.org/licenses/MIT) license.
