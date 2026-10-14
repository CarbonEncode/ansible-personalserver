# Personal Fedora Web Server

This Ansible playbook provisions a Fedora web server with Nginx, PHP-FPM,
MariaDB, automated wildcard certificates, host networking, and a managed
firewall.

## What the playbook configures

The roles in `main.yml` run in the following order:

1. **General system configuration**
   - Updates installed packages.
   - Sets the hostname and forces the timezone to UTC.
   - Replaces root's authorized keys with the inventory-defined list.
   - Disables SSH password authentication and restricts `webusers` to SFTP.
2. **Networking and firewall**
   - Configures static IPv4 and IPv6 addressing with `systemd-networkd`.
   - Enables `systemd-networkd` and disables NetworkManager.
   - Installs the `iptables-nft` compatibility tools.
   - Deploys `/opt/firewall/bin/iptables-rules.sh` and its systemd service.
   - Allows inbound SSH only from inventory-defined IPv4 and IPv6 sources.
   - Allows inbound HTTP and HTTPS on ports 80 and 443.
   - Configures Google Public DNS over TLS and enables BBR.
3. **Nginx and certificates**
   - Creates one system user and document root per virtual host.
   - Applies persistent SELinux read/write contexts to each `public_html` tree.
   - Installs acme.sh under `/opt` and uses Gcore DNS validation with Let's
     Encrypt.
   - Issues certificates containing the zone apex and wildcard name.
   - Verifies certificate files before enabling SSL virtual hosts.
   - Creates `sites-enabled` symlinks and validates Nginx before every reload.
4. **MariaDB**
   - Installs MariaDB and restricts it to localhost.
5. **PHP-FPM**
   - Installs PHP and OPcache.
   - Creates a dedicated PHP-FPM pool, socket, and OPcache configuration for
     every virtual host.

## Requirements

- Fedora 40 or newer with root SSH access. The playbook rejects other targets
  before applying any roles.
- Ansible installed on the control machine.
- The DNS zones hosted by Gcore and a Gcore API key with permission to manage
  their DNS records.
- Public DNS records for the configured virtual hosts pointing to the server.

Install the required Ansible collections:

```bash
ansible-galaxy collection install -r requirements.yml
```

## Inventory setup

The real `inventory.yml` is ignored by Git because it contains host-specific
configuration and the Gcore API key. Create it from the sanitized example:

```bash
cp inventory.example.yml inventory.yml
```

At minimum, replace:

- `ansible_host`
- `system_hostname`
- `network_interface`
- the addresses and gateway under `network_ipv4` and `network_ipv6`
- the trusted SSH sources under `firewall_ssh_whitelist.ipv4` and `.ipv6`
- every entry in `root_authorized_keys`
- `gcore_api_key`
- `acme_email`
- `acme_wildcard_domains`
- `nginx_vhosts`

The `certificate` value for each virtual host must match an entry in
`acme_wildcard_domains`. For example, `app.apps.example.com` uses the
`apps.example.com` certificate, which contains both `apps.example.com` and
`*.apps.example.com`.

The root SSH key list is authoritative. Keys not present in
`root_authorized_keys` are removed from `/root/.ssh/authorized_keys`.

The network settings are also authoritative. The network role writes
`/etc/systemd/network/10-static.network`, disables DHCP and IPv6 router
advertisement configuration, and stops and disables NetworkManager. Confirm
the interface name, CIDR addresses, and gateways from the server provider
before applying the playbook, or the host may become unreachable.

SSH firewall access is also authoritative. Each entry in
`firewall_ssh_whitelist.ipv4` and `firewall_ssh_whitelist.ipv6` may be an
individual address or a CIDR network. Both lists must contain at least one
entry. Confirm that the address from which Ansible connects is included before
applying the playbook. Existing SSH connections remain established, but new
connections from sources outside the lists are rejected.

### Protecting the Gcore API key

Do not commit a real API key. Ansible Vault can encrypt the value or the whole
inventory file. For example:

```bash
ansible-vault encrypt_string --name gcore_api_key 'YOUR_GCORE_API_KEY'
```

Place the generated Vault block in `inventory.yml` and use `--ask-vault-pass`
or your configured vault-password mechanism when running the playbook.

## Running the playbook

Validate the playbook and inventory first:

```bash
ansible-playbook --syntax-check main.yml
ansible-inventory --graph
```

Apply the configuration:

```bash
ansible-playbook main.yml
```

For a Vault-encrypted API key:

```bash
ansible-playbook main.yml --ask-vault-pass
```

## Managed paths

| Purpose | Path |
| --- | --- |
| Web roots | `/var/www/<domain>/public_html` |
| Nginx available vhosts | `/etc/nginx/sites-available` |
| Nginx enabled vhosts | `/etc/nginx/sites-enabled` |
| Installed certificates | `/etc/pki/tls/acme/<zone>` |
| acme.sh runtime | `/opt/acme.sh` |
| acme.sh source | `/opt/acme.sh-source` |
| Firewall script | `/opt/firewall/bin/iptables-rules.sh` |
| Firewall data | `/opt/firewall/etc` |
| Static network configuration | `/etc/systemd/network/10-static.network` |
| PHP-FPM pools | `/etc/php-fpm.d` |

## Operational notes

- Initial certificate issuance requires access to the Gcore API and can wait
  for DNS propagation.
- Initial Diffie-Hellman parameter generation may take several minutes.
- SSL virtual hosts are not deployed unless their full chain and private key
  exist and are non-empty.
- The firewall script flushes existing IPv4 and IPv6 filter rules before
  applying the managed rules. Add required ports to the script before running
  the playbook.
- Port 22 is not globally open. Its IPv4 and IPv6 source rules are generated
  from `firewall_ssh_whitelist` in the inventory.
- The playbook validates Nginx with `nginx -t` before reloads and certificate
  renewal reload commands.
- `iptables-nft` is a compatibility interface. Native nftables should be
  considered for a future RHEL migration.
