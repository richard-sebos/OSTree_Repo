# rpm-ostree Repository Setup with Ansible

This playbook extends your existing Flatpak OSTree server infrastructure to support rpm-ostree for Kinoite system image distribution.

## Prerequisites

1. **Apache must already be installed and running** (from your Flatpak repo setup)
2. **SSL certificates must exist** (if using HTTPS - enabled by default)
   - `/etc/pki/tls/certs/flatpak-repo.crt`
   - `/etc/pki/tls/private/flatpak-repo.key`

**Note:** The playbook will automatically install `ostree` and `rpm-ostree` packages if they're not already present.

## Directory Structure

```
ansible-rpm-ostree-repo/
├── playbooks/
│   └── setup-rpm-ostree-repo.yml    # Main playbook
├── roles/
│   ├── rpm_ostree_repo/             # Repository initialization
│   ├── rpm_ostree_apache/           # Apache configuration
│   └── rpm_ostree_selinux/          # SELinux policies
├── inventory/
│   └── hosts.yml                     # Server inventory
└── group_vars/
    └── rpm_ostree_repo_servers.yml   # Configuration variables
```

## Quick Start

### 1. Configure Inventory

Edit `inventory/hosts.yml` and update with your server details:

```yaml
rpm_ostree_repo_servers:
  hosts:
    your-server-hostname:
      ansible_host: 192.168.1.100  # Your server IP
      ansible_user: root
```

### 2. Run the Playbook

```bash
cd ansible-rpm-ostree-repo
ansible-playbook -i inventory/hosts.yml playbooks/setup-rpm-ostree-repo.yml
```

### 3. Verify the Setup

Check that the repository was created:

```bash
ssh your-server "ls -la /srv/ostree/rpm-ostree/kinoite"
```

Test the HTTPS endpoint:

```bash
curl -k https://your-server:8443/repo/kinoite/config
```

Or test HTTP (if not using HTTPS):

```bash
curl http://your-server:8080/repo/kinoite/config
```

## Configuration

### Default Settings

The following defaults are set in `group_vars/rpm_ostree_repo_servers.yml`:

- **Repository Path:** `/srv/ostree/rpm-ostree/kinoite`
- **HTTP Alias:** `/repo/kinoite`
- **Repository Mode:** `archive-z2`
- **HTTPS:** `Enabled` (reuses Flatpak SSL certificates)
- **HTTPS Port:** `8443` (from Flatpak setup)
- **HTTP Port:** `8080` (from Flatpak setup)

### Customization

To customize, edit `group_vars/rpm_ostree_repo_servers.yml`:

```yaml
rpm_ostree_repo_base_path: /srv/ostree/rpm-ostree
rpm_ostree_repo_name: kinoite
rpm_ostree_http_alias: /repo/kinoite
rpm_ostree_enable_https: true  # Set to false for HTTP only

# SSL certificate paths (if using HTTPS)
ssl_cert_file: /etc/pki/tls/certs/flatpak-repo.crt
ssl_key_file: /etc/pki/tls/private/flatpak-repo.key
```

## Next Steps: Composing Your First Image

### 1. Create a Treefile

Create a `kinoite.json` file on your server. See `examples/kinoite.json` for a template.

### 2. Compose the Image

```bash
sudo rpm-ostree compose tree \
  --repo=/srv/ostree/rpm-ostree/kinoite \
  kinoite.json
```

### 3. Update Repository Summary

```bash
sudo ostree summary -u --repo=/srv/ostree/rpm-ostree/kinoite
```

### 4. Configure Kinoite Clients

On your Kinoite workstation:

```bash
# Add the remote (HTTPS - recommended)
sudo ostree remote add --no-gpg-verify kinoite \
  https://your-server:8443/repo/kinoite

# Or use HTTP if HTTPS is disabled
# sudo ostree remote add --no-gpg-verify kinoite \
#   http://your-server:8080/repo/kinoite

# Rebase to your custom image
sudo rpm-ostree rebase kinoite:fedora/x86_64/kinoite/stable

# Reboot to apply
systemctl reboot
```

## Architecture

This playbook integrates with your existing infrastructure:

```
┌─────────────────────────────────────────┐
│  Apache Server (existing from Flatpak) │
│  Port: 8080 (or your configured port)   │
├─────────────────────────────────────────┤
│                                         │
│  /repo/flatpak (Flatpak apps)          │
│    → /var/flatpak/repo                 │
│                                         │
│  /repo/kinoite (rpm-ostree) [NEW]      │
│    → /srv/ostree/rpm-ostree/kinoite    │
│                                         │
└─────────────────────────────────────────┘
```

Both repositories:
- Share the same Apache server
- Use the same SELinux context (`httpd_sys_content_t`)
- Use the same firewall rules
- Serve content on the same port

## Tags

Run specific parts of the playbook using tags:

```bash
# Only configure the repository
ansible-playbook -i inventory/hosts.yml playbooks/setup-rpm-ostree-repo.yml --tags repository

# Only configure Apache
ansible-playbook -i inventory/hosts.yml playbooks/setup-rpm-ostree-repo.yml --tags apache

# Only configure SELinux
ansible-playbook -i inventory/hosts.yml playbooks/setup-rpm-ostree-repo.yml --tags selinux
```

## Troubleshooting

### SELinux Issues

If you get permission denied errors:

```bash
# Check SELinux context
ls -lZ /srv/ostree/rpm-ostree/kinoite

# Manually restore context if needed
sudo restorecon -Rv /srv/ostree/rpm-ostree
```

### Apache Issues

Test Apache configuration:

```bash
sudo apachectl configtest
```

Check Apache error logs:

```bash
sudo tail -f /var/log/httpd/error_log
```

### Repository Issues

Verify repository was initialized:

```bash
ostree --repo=/srv/ostree/rpm-ostree/kinoite refs
```

## Version

- **Version:** 1.0.0
- **Last Updated:** 2026-01-25
- **Compatible with:** Fedora, RHEL, CentOS (with modifications for Debian/Ubuntu)
