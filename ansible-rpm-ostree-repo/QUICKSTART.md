# Quick Start Guide

Get your rpm-ostree repository running in 5 minutes!

## Step 1: Install Ansible Collections

```bash
cd ansible-rpm-ostree-repo
ansible-galaxy collection install -r requirements.yml
```

## Step 2: Configure Your Server

Edit `inventory/hosts.yml` and replace with your server details:

```yaml
rpm_ostree_repo_servers:
  hosts:
    myserver:
      ansible_host: 192.168.1.100  # Your actual server IP
      ansible_user: root            # Or your sudo user
```

## Step 3: Run the Playbook

```bash
ansible-playbook playbooks/setup-rpm-ostree-repo.yml
```

The playbook will:
- ✅ Create `/srv/ostree/rpm-ostree/kinoite` directory
- ✅ Initialize the OSTree repository
- ✅ Configure Apache to serve the repo at `/repo/kinoite`
- ✅ Set up SELinux contexts

## Step 4: Verify Installation

Test the endpoint (HTTPS):

```bash
curl -k https://YOUR-SERVER-IP:8443/repo/kinoite/config
```

Or HTTP:

```bash
curl http://YOUR-SERVER-IP:8080/repo/kinoite/config
```

You should see the OSTree repository config file!

**Note:** By default, HTTPS is enabled and reuses the SSL certificates from your Flatpak setup.

## Step 5: Create Your First Image

### On the server:

1. Copy the example treefile:
```bash
scp examples/kinoite.json your-server:/root/
```

2. SSH to the server and compose:
```bash
ssh your-server
sudo rpm-ostree compose tree \
  --repo=/srv/ostree/rpm-ostree/kinoite \
  /root/kinoite.json

sudo ostree summary -u --repo=/srv/ostree/rpm-ostree/kinoite
```

### On your Kinoite client:

```bash
# Add the remote (HTTPS - recommended)
sudo ostree remote add --no-gpg-verify kinoite \
  https://YOUR-SERVER-IP:8443/repo/kinoite

# Or use HTTP if HTTPS is disabled
# sudo ostree remote add --no-gpg-verify kinoite \
#   http://YOUR-SERVER-IP:8080/repo/kinoite

# Rebase to your custom image
sudo rpm-ostree rebase kinoite:fedora/x86_64/kinoite/stable

# Reboot
systemctl reboot
```

## What Gets Created

```
Server Infrastructure:
├── /srv/ostree/rpm-ostree/kinoite/  (OSTree repo)
├── /etc/httpd/conf.d/rpm-ostree-repo.conf (Apache config)
└── http://server:8080/repo/kinoite (Public endpoint)
```

## Next Steps

- Customize `examples/kinoite.json` to include your preferred packages
- Set up automated builds with a cron job or CI/CD pipeline
- Configure GPG signing for production use
- Create multiple variants (desktop, server, minimal, etc.)

## Troubleshooting

**Connection refused?**
- Check firewall: `sudo firewall-cmd --list-ports`
- The Flatpak setup should have already opened port 8080

**Permission denied?**
- Verify SELinux: `ls -lZ /srv/ostree/rpm-ostree/kinoite`
- Should show `httpd_sys_content_t`

**Empty directory?**
- You need to compose an image first using `rpm-ostree compose tree`

## Support

See `README.md` for detailed documentation and troubleshooting.
