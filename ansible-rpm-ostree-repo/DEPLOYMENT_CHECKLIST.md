# Deployment Checklist

Use this checklist to ensure a successful rpm-ostree repository deployment.

## Pre-Deployment Checklist

### ✅ Prerequisites

- [ ] Flatpak repository is already set up and running
- [ ] Apache is installed and serving on port 8080 (HTTP) and 8443 (HTTPS)
- [ ] SSL certificates exist on the server (if using HTTPS)
  - `/etc/pki/tls/certs/flatpak-repo.crt`
  - `/etc/pki/tls/private/flatpak-repo.key`
- [ ] Server is accessible via SSH
- [ ] You have sudo/root access to the server
- [ ] Ansible is installed on your control machine (`ansible --version`)

### ✅ Server Requirements

- [ ] Operating System: RHEL/Fedora/CentOS (or Debian/Ubuntu with modifications)
- [ ] Free disk space in `/srv` (at least 50GB recommended)
- [ ] Apache (`httpd`) must be installed and running (from Flatpak setup)
- [ ] SELinux tools installed (`policycoreutils-python-utils`)

**Note:** The playbook will automatically install `ostree` and `rpm-ostree` if needed.

### ✅ Network Requirements

- [ ] Port 8080 is accessible from clients (already open for Flatpak)
- [ ] Server has internet access (for downloading packages during compose)
- [ ] DNS/hostname resolution is working

## Deployment Steps

### Step 1: Prepare the Playbook

- [ ] Clone or download the `ansible-rpm-ostree-repo` directory
- [ ] Review `README.md` and `QUICKSTART.md`
- [ ] Install Ansible Galaxy dependencies:
  ```bash
  cd ansible-rpm-ostree-repo
  ansible-galaxy collection install -r requirements.yml
  ```

### Step 2: Configure Inventory

- [ ] Edit `inventory/hosts.yml`
- [ ] Set correct `ansible_host` (server IP or hostname)
- [ ] Set correct `ansible_user` (user with sudo access)
- [ ] Test connectivity:
  ```bash
  ansible -i inventory/hosts.yml all -m ping
  ```

### Step 3: Review Configuration

- [ ] Review `group_vars/rpm_ostree_repo_servers.yml`
- [ ] Verify default paths are acceptable:
  - `rpm_ostree_repo_base_path: /srv/ostree/rpm-ostree`
  - `rpm_ostree_repo_name: kinoite`
  - `rpm_ostree_http_alias: /repo/kinoite`
  - `rpm_ostree_enable_https: true` (set to `false` for HTTP only)
- [ ] Verify SSL certificate paths (if using HTTPS):
  - `ssl_cert_file: /etc/pki/tls/certs/flatpak-repo.crt`
  - `ssl_key_file: /etc/pki/tls/private/flatpak-repo.key`
- [ ] Customize if needed

### Step 4: Run Playbook (Dry Run)

- [ ] Run in check mode to see what would change:
  ```bash
  ansible-playbook -i inventory/hosts.yml \
    playbooks/setup-rpm-ostree-repo.yml \
    --check
  ```
- [ ] Review the output for any issues

### Step 5: Execute Deployment

- [ ] Run the playbook:
  ```bash
  ansible-playbook -i inventory/hosts.yml \
    playbooks/setup-rpm-ostree-repo.yml
  ```
- [ ] Monitor output for any errors
- [ ] Verify all tasks show "ok" or "changed" (not "failed")

### Step 6: Verify Installation

- [ ] Check repository directory exists:
  ```bash
  ssh SERVER "ls -la /srv/ostree/rpm-ostree/kinoite"
  ```
- [ ] Verify OSTree config:
  ```bash
  ssh SERVER "cat /srv/ostree/rpm-ostree/kinoite/config"
  ```
- [ ] Test HTTPS endpoint (if HTTPS enabled):
  ```bash
  curl -k https://SERVER:8443/repo/kinoite/config
  ```
- [ ] Test HTTP endpoint:
  ```bash
  curl http://SERVER:8080/repo/kinoite/config
  ```
- [ ] Check Apache configuration:
  ```bash
  ssh SERVER "cat /etc/httpd/conf.d/rpm-ostree-repo.conf"
  ```
- [ ] Verify SSL configuration is present (if HTTPS enabled):
  ```bash
  ssh SERVER "grep SSLEngine /etc/httpd/conf.d/rpm-ostree-repo.conf"
  ```
- [ ] Verify SELinux context:
  ```bash
  ssh SERVER "ls -lZ /srv/ostree/rpm-ostree/kinoite"
  ```
  Should show: `httpd_sys_content_t`

## Post-Deployment Tasks

### Compose Your First Image

- [ ] Copy treefile to server:
  ```bash
  scp examples/kinoite.json SERVER:/root/
  ```
- [ ] SSH to server
- [ ] Customize treefile for your needs
- [ ] Run composition:
  ```bash
  sudo rpm-ostree compose tree \
    --repo=/srv/ostree/rpm-ostree/kinoite \
    /root/kinoite.json
  ```
- [ ] Update summary:
  ```bash
  sudo ostree summary -u --repo=/srv/ostree/rpm-ostree/kinoite
  ```
- [ ] Verify refs:
  ```bash
  sudo ostree refs --repo=/srv/ostree/rpm-ostree/kinoite
  ```

### Test with Client

- [ ] Set up a Kinoite VM or workstation for testing
- [ ] Add the remote (HTTPS recommended):
  ```bash
  sudo ostree remote add --no-gpg-verify kinoite \
    https://SERVER:8443/repo/kinoite
  ```
  Or HTTP if HTTPS is disabled:
  ```bash
  sudo ostree remote add --no-gpg-verify kinoite \
    http://SERVER:8080/repo/kinoite
  ```
- [ ] Verify remote is added:
  ```bash
  ostree remote list -u
  ```
- [ ] Test pulling summary:
  ```bash
  sudo ostree remote summary kinoite
  ```
- [ ] Rebase to your custom image:
  ```bash
  sudo rpm-ostree rebase kinoite:fedora/x86_64/kinoite/stable
  ```
- [ ] Reboot and verify:
  ```bash
  systemctl reboot
  # After reboot:
  rpm-ostree status
  ```

## Security Hardening (Optional but Recommended)

- [ ] Set up GPG signing
  - [ ] Generate GPG key
  - [ ] Configure signing in OSTree
  - [ ] Distribute public key to clients
  - [ ] Remove `--no-gpg-verify` from client commands

- [ ] Enable HTTPS (if not already)
  - [ ] Obtain SSL certificates
  - [ ] Configure Apache for HTTPS
  - [ ] Update firewall for port 443
  - [ ] Test HTTPS access

- [ ] Set up authentication (if needed)
  - [ ] Configure Apache basic auth
  - [ ] Or implement OAuth/SSO

## Monitoring Setup

- [ ] Set up log rotation for Apache logs
- [ ] Monitor disk space in `/srv/ostree`
- [ ] Set up alerts for failed compositions
- [ ] Monitor Apache access logs for client pull activity
- [ ] Set up SELinux denial monitoring

## Documentation

- [ ] Document your custom treefile configuration
- [ ] Create runbook for routine maintenance
- [ ] Document client setup procedure for your organization
- [ ] Create troubleshooting guide for common issues

## Automation (Optional)

- [ ] Set up CI/CD pipeline for automatic builds
- [ ] Configure cron job for regular updates
- [ ] Implement testing before promoting to production
- [ ] Set up notifications for build status

## Troubleshooting Checklist

If something goes wrong, check these:

### Repository Access Issues
- [ ] Verify Apache is running: `systemctl status httpd`
- [ ] Check Apache error logs: `tail -f /var/log/httpd/error_log`
- [ ] Test local access: `curl localhost:8080/repo/kinoite/config`
- [ ] Check firewall: `firewall-cmd --list-ports`

### SELinux Denials
- [ ] Check for denials: `ausearch -m avc -ts recent`
- [ ] Verify context: `ls -lZ /srv/ostree/rpm-ostree/kinoite`
- [ ] Restore context: `restorecon -Rv /srv/ostree/rpm-ostree`
- [ ] Check SELinux mode: `getenforce`

### Composition Failures
- [ ] Check available disk space: `df -h /srv`
- [ ] Verify treefile syntax: `cat /root/kinoite.json | jq`
- [ ] Check network connectivity for package downloads
- [ ] Review rpm-ostree logs: `journalctl -u rpm-ostreed`

### Client Pull Failures
- [ ] Verify network connectivity from client to server
- [ ] Check repository summary exists on server
- [ ] Verify ref exists: `ostree refs --repo=/srv/ostree/rpm-ostree/kinoite`
- [ ] Test manual pull: `ostree pull kinoite:fedora/x86_64/kinoite/stable`

## Rollback Plan

If you need to rollback:

- [ ] Apache config is separate (`rpm-ostree-repo.conf`) - just remove it
- [ ] Flatpak repo is unaffected
- [ ] Repository data in `/srv/ostree/rpm-ostree` can be removed
- [ ] No system packages were modified

To rollback:
```bash
# Remove Apache config
sudo rm /etc/httpd/conf.d/rpm-ostree-repo.conf
sudo systemctl reload httpd

# Remove repository (if desired)
sudo rm -rf /srv/ostree/rpm-ostree

# Remove SELinux context
sudo semanage fcontext -d "/srv/ostree/rpm-ostree(/.*)?"
```

## Sign-Off

Deployment completed by: ________________
Date: ________________
Server: ________________
Status: ☐ Success  ☐ Issues (documented below)

Notes:
_________________________________________________________________
_________________________________________________________________
_________________________________________________________________
