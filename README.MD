# Personal Fedora Server Playbook

This repository contains the Ansible playbook I use to provision my own
server. It is tailored to my infrastructure, service providers, network
layout, domains, and operational preferences.

It was never designed for use by others and should not be treated as a
general-purpose, reusable, or supported server configuration. The example
inventory is illustrative only. Anyone running this playbook must review and
adapt the complete configuration first.

## What it does

In short, the playbook:

- Configures a Fedora 40 or newer system, root SSH access, cron, hostname, and
  timezone.
- Replaces NetworkManager with a static dual-stack `systemd-networkd` setup
  and configures `systemd-resolved`.
- Installs a managed IPv4/IPv6 firewall, restricts SSH to trusted inventory
  addresses, and maintains IPv4 threat-feed ipsets.
- Installs Nginx with a self-signed default HTTPS host and initially provisions
  application virtual hosts without overwriting later local changes.
- Issues Let's Encrypt wildcard certificates through Gcore DNS using acme.sh.
- Installs MariaDB, PHP-FPM, and a dedicated PHP-FPM pool for each configured
  virtual host.
- Applies the required service, file permission, and SELinux configuration.

## Important

`inventory.yml` is intentionally ignored by Git because it contains private,
host-specific values. The playbook manages networking, firewall rules, root
authorized keys, and TLS configuration authoritatively. Incorrect inventory
values can make the server unreachable.

The checked-in `inventory.example.yml` contains documentation addresses and
placeholders; it is not a ready-to-run configuration.

## Running it

Install the required collections and run the playbook only after reviewing the
inventory and every role:

```bash
ansible-galaxy collection install -r requirements.yml
ansible-playbook --syntax-check main.yml
ansible-playbook main.yml
```
