# Quick Start Guide

Get your Flatpak repository running in 5 minutes!

## Prerequisites

1. **Rocky Linux 9 / RHEL 9 / Fedora** server
2. **Root or sudo access**
3. **Ansible installed** on your control machine

## Step-by-Step Setup

### Step 1: Install Ansible Collections

```bash
cd ansible-flatpak-repo
ansible-galaxy collection install -r requirements.yml
```

### Step 2: Configure for Local Installation

The default `inventory/hosts` file is already configured for localhost:

```ini
[flatpak_repo_servers]
localhost ansible_connection=local
```

### Step 3: Choose Your Configuration

**Option A: HTTP Only (Simplest)**

No changes needed! Just run the playbook.

**Option B: HTTPS with Self-Signed Certificate**

Edit `group_vars/flatpak_repo_servers.yml`:

```yaml
flatpak_repo_enable_https: true
```

### Step 4: Run the Playbook

**Basic HTTP setup:**
```bash
ansible-playbook playbooks/setup-flatpak-repo.yml
```

**With HTTPS (command line override):**
```bash
ansible-playbook playbooks/setup-flatpak-repo.yml -e "flatpak_repo_enable_https=true"
```

**Check what will happen (dry run):**
```bash
ansible-playbook playbooks/setup-flatpak-repo.yml --check
```

### Step 5: Verify Installation

```bash
# Check Apache is running
sudo systemctl status httpd

# Test HTTP access
curl -I http://localhost:8080/repo/

# If HTTPS enabled
curl -I https://localhost:8443/repo/
```

### Step 6: Add Your First Application

```bash
sudo flatpak-repo-add org.mozilla.firefox
```

### Step 7: Configure a Client

From a Kinoite workstation:

```bash
# Add to /etc/hosts
echo "192.168.35.35 flatpak-repo.local" | sudo tee -a /etc/hosts

# Configure repository
curl http://flatpak-repo.local:8080/flatpak-configure-client | sudo bash

# Install an app
flatpak install enterprise-apps org.mozilla.firefox
```

## Running Step-by-Step

If you want to see what each role does, run them individually:

```bash
# 1. Install dependencies
ansible-playbook playbooks/setup-flatpak-repo.yml --tags dependencies

# 2. Generate SSL certificates (if using HTTPS)
ansible-playbook playbooks/setup-flatpak-repo.yml --tags ssl -e "flatpak_repo_enable_https=true"

# 3. Initialize repository
ansible-playbook playbooks/setup-flatpak-repo.yml --tags repository

# 4. Configure Apache
ansible-playbook playbooks/setup-flatpak-repo.yml --tags apache

# 5. Configure SELinux
ansible-playbook playbooks/setup-flatpak-repo.yml --tags selinux

# 6. Configure firewall
ansible-playbook playbooks/setup-flatpak-repo.yml --tags firewall

# 7. Deploy client scripts
ansible-playbook playbooks/setup-flatpak-repo.yml --tags scripts
```

## Common Customizations

### Use Standard Ports (80/443)

Edit `group_vars/flatpak_repo_servers.yml`:

```yaml
flatpak_repo_http_port: 80
flatpak_repo_https_port: 443
flatpak_repo_enable_https: true
```

### Custom Repository Path

```yaml
flatpak_repo_path: /srv/flatpak/repo
```

### Enable GPG Signing

```bash
# Generate GPG key
gpg --full-generate-key

# Get key ID
gpg --list-secret-keys --keyid-format LONG

# Add to group_vars/flatpak_repo_servers.yml
flatpak_repo_gpg_key_id: "YOUR_KEY_ID_HERE"
```

## Troubleshooting

### Apache Won't Start

```bash
# Check configuration
sudo apachectl configtest

# View logs
sudo journalctl -u httpd -n 50
```

### SELinux Denials

```bash
# Check for denials
sudo ausearch -m avc -ts recent

# Re-run SELinux configuration
ansible-playbook playbooks/setup-flatpak-repo.yml --tags selinux
```

### Firewall Blocking

```bash
# Check firewall
sudo firewall-cmd --list-ports

# Re-run firewall configuration
ansible-playbook playbooks/setup-flatpak-repo.yml --tags firewall
```

### Need to Start Fresh?

```bash
# Stop Apache
sudo systemctl stop httpd

# Remove repository
sudo rm -rf /var/flatpak/repo

# Remove Apache config
sudo rm -f /etc/httpd/conf.d/flatpak-repo.conf

# Clean Listen directives
sudo sed -i '/^Listen 8080$/d' /etc/httpd/conf/httpd.conf
sudo sed -i '/^Listen 8443$/d' /etc/httpd/conf/httpd.conf

# Run playbook again
ansible-playbook playbooks/setup-flatpak-repo.yml
```

## What Gets Created

After running the playbook:

**Files:**
- `/var/flatpak/repo/` - Repository directory
- `/etc/httpd/conf.d/flatpak-repo.conf` - Apache config
- `/usr/local/bin/flatpak-repo-add` - Helper to add apps
- `/usr/local/bin/flatpak-configure-client` - Client config script
- `/var/www/html/repo` - Symlink to repository
- `/etc/pki/tls/certs/flatpak-repo.crt` - SSL cert (if HTTPS)
- `/etc/pki/tls/private/flatpak-repo.key` - SSL key (if HTTPS)

**Services:**
- `httpd.service` - Apache web server (enabled and started)

**Network:**
- Port 8080/tcp - HTTP (opened in firewall)
- Port 8443/tcp - HTTPS (opened if enabled)

**SELinux:**
- `httpd_sys_content_t` context on `/var/flatpak/repo`

## Next Steps

1. **Add more applications:**
   ```bash
   sudo flatpak-repo-add org.libreoffice.LibreOffice
   sudo flatpak-repo-add org.gimp.GIMP
   ```

2. **Configure additional clients**

3. **Set up automatic updates:**
   Create a cron job to update repository applications weekly

4. **Enable HTTPS in production:**
   Use Let's Encrypt or internal CA for proper certificates

5. **Set up monitoring:**
   Monitor Apache, disk space, and repository health

## Getting Help

- **README.md** - Complete documentation
- **Roles documentation** - Check `roles/*/tasks/main.yml` for details
- **Run with verbosity** - `ansible-playbook -vv` for debugging
- **Check mode** - `ansible-playbook --check` to see what would change

---

**Ready to deploy!** 🚀
