# Image Composition Guide

The ansible-rpm-ostree-repo playbook includes automated image composition capabilities.

## Overview

The `rpm_ostree_compose` role:
- ✅ Deploys a customizable treefile template
- ✅ Optionally composes the image during setup
- ✅ Supports manual composition
- ✅ Configurable package lists

## Quick Start

### Option 1: Manual Composition (Default)

By default, the playbook deploys the treefile but does NOT compose the image automatically.

1. **Run the playbook:**
   ```bash
   ansible-playbook -i inventory/hosts.yml playbooks/setup-rpm-ostree-repo.yml
   ```

2. **Manually compose the image on the server:**
   ```bash
   sudo rpm-ostree compose tree \
     --repo=/srv/ostree/rpm-ostree/kinoite \
     /etc/rpm-ostree/treefiles/kinoite.json

   sudo ostree summary -u --repo=/srv/ostree/rpm-ostree/kinoite
   ```

### Option 2: Automatic Composition During Setup

To compose the image automatically during playbook execution:

1. **Enable composition in `group_vars/rpm_ostree_repo_servers.yml`:**
   ```yaml
   rpm_ostree_compose_on_setup: true
   ```

2. **Run the playbook:**
   ```bash
   ansible-playbook -i inventory/hosts.yml playbooks/setup-rpm-ostree-repo.yml
   ```

   The playbook will:
   - Deploy the treefile
   - Compose the image (this takes 10-30 minutes)
   - Update the repository summary
   - Display available refs

## Configuration

### Basic Configuration

Edit `group_vars/rpm_ostree_repo_servers.yml`:

```yaml
# Enable/disable automatic composition
rpm_ostree_compose_on_setup: false  # true = compose during setup

# OSTree ref (branch name)
rpm_ostree_ref: fedora/x86_64/kinoite/stable

# Fedora version
rpm_ostree_fedora_release: 41
```

### Package Customization

Add your own packages to `group_vars/rpm_ostree_repo_servers.yml`:

```yaml
rpm_ostree_additional_packages:
  - firefox
  - thunderbird
  - libreoffice
  - gimp
  - vlc
  - code  # VS Code (if repo configured)
  - docker
```

### Advanced Configuration

For more control, override defaults in `group_vars/rpm_ostree_repo_servers.yml`:

```yaml
# Override base packages
rpm_ostree_base_packages:
  - fedora-release-kinoite
  - kernel
  - systemd
  # ... add more

# Override desktop packages
rpm_ostree_desktop_packages:
  - plasma-desktop
  - plasma-workspace
  # ... add more

# Exclude packages
rpm_ostree_exclude_packages:
  - PackageKit
  - gnome-software
```

## Treefile Location

The treefile is deployed to:
```
/etc/rpm-ostree/treefiles/kinoite.json
```

You can manually edit this file on the server and recompose if needed.

## Manual Composition

### Compose Image

```bash
sudo rpm-ostree compose tree \
  --repo=/srv/ostree/rpm-ostree/kinoite \
  /etc/rpm-ostree/treefiles/kinoite.json
```

### Update Summary

After composition, always update the summary:

```bash
sudo ostree summary -u --repo=/srv/ostree/rpm-ostree/kinoite
```

### Verify Refs

Check available refs:

```bash
sudo ostree refs --repo=/srv/ostree/rpm-ostree/kinoite
```

Should show:
```
fedora/x86_64/kinoite/stable
```

## Automated Composition

### Using Tags

Compose without full setup:

```bash
# Only run compose role
ansible-playbook -i inventory/hosts.yml \
  playbooks/setup-rpm-ostree-repo.yml \
  --tags compose
```

### Scheduled Composition

Create a cron job for automatic updates:

```bash
# On the server
sudo crontab -e

# Add this line to compose nightly at 2 AM
0 2 * * * rpm-ostree compose tree --repo=/srv/ostree/rpm-ostree/kinoite /etc/rpm-ostree/treefiles/kinoite.json && ostree summary -u --repo=/srv/ostree/rpm-ostree/kinoite
```

## Composition Time

Image composition typically takes:
- **First compose:** 15-30 minutes (downloads all packages)
- **Subsequent composes:** 5-15 minutes (only changed packages)

Factors affecting time:
- Number of packages
- Server CPU/RAM
- Network speed
- Package cache

## Troubleshooting

### Composition Fails

**Check logs:**
```bash
sudo journalctl -u rpm-ostreed -f
```

**Common issues:**
1. **Insufficient disk space** - Need at least 20GB free in `/srv`
2. **Network issues** - Can't download packages
3. **Invalid package names** - Package doesn't exist in repos
4. **Conflicting packages** - Dependencies can't be resolved

### Verify Repository Configuration

```bash
# Check if repo files exist
ls -la /etc/yum.repos.d/

# Test repository access
sudo dnf repolist
```

### Force Clean Composition

If composition is stuck or corrupted:

```bash
# Remove temporary files
sudo rm -rf /srv/ostree/rpm-ostree/kinoite/tmp/*

# Try composition again
sudo rpm-ostree compose tree \
  --repo=/srv/ostree/rpm-ostree/kinoite \
  /etc/rpm-ostree/treefiles/kinoite.json
```

## Package Lists

### Default Packages

The default configuration includes:

**Base System:**
- fedora-release-kinoite
- kernel
- systemd
- rpm-ostree

**KDE Plasma Desktop:**
- plasma-desktop
- plasma-workspace
- sddm
- dolphin
- konsole
- kate

**System Tools:**
- NetworkManager
- firewalld
- vim-enhanced
- git
- wget
- curl

### Adding Custom Repositories

To add packages from custom repositories, edit the treefile on the server:

```bash
sudo vim /etc/rpm-ostree/treefiles/kinoite.json
```

Add repository URLs:

```json
{
  "ref": "fedora/x86_64/kinoite/stable",
  "repos": [
    "fedora",
    "fedora-updates",
    "rpmfusion-free",
    "rpmfusion-free-updates"
  ],
  ...
}
```

## Example Workflows

### Development Workstation

```yaml
# group_vars/rpm_ostree_repo_servers.yml
rpm_ostree_additional_packages:
  - vim-enhanced
  - git
  - tmux
  - htop
  - code
  - docker
  - podman
  - buildah
  - nodejs
  - python3-pip
```

### Minimal Desktop

```yaml
# group_vars/rpm_ostree_repo_servers.yml
rpm_ostree_desktop_packages:
  - plasma-desktop
  - sddm
  - konsole

rpm_ostree_additional_packages:
  - firefox
```

### Kiosk System

```yaml
# group_vars/rpm_ostree_repo_servers.yml
rpm_ostree_additional_packages:
  - firefox
  - cage  # Wayland kiosk compositor

rpm_ostree_exclude_packages:
  - PackageKit
  - gnome-software
  - discover  # KDE software center
```

## Next Steps

After composition:

1. **Verify image:** `curl -k https://server:8443/repo/kinoite/config`
2. **Configure clients:** Add remote on Kinoite workstations
3. **Rebase clients:** `rpm-ostree rebase kinoite:fedora/x86_64/kinoite/stable`
4. **Set up automation:** Create cron jobs for regular updates

## Best Practices

✅ Test compositions on a staging server first
✅ Keep treefile in version control
✅ Document custom package additions
✅ Schedule compositions during low-usage periods
✅ Monitor disk space regularly
✅ Keep Fedora base repos enabled
✅ Validate images before deploying to production clients
