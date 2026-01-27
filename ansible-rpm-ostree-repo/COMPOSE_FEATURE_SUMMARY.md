# Image Composition Feature - Summary

## What Was Added

A complete automated image composition system for rpm-ostree!

### New Role: `rpm_ostree_compose`

**Files Created:**
```
roles/rpm_ostree_compose/
├── defaults/main.yml           # Package lists and configuration
├── tasks/main.yml              # Composition automation tasks
└── templates/
    └── kinoite.json.j2         # Treefile template
```

**New Documentation:**
- `COMPOSE_GUIDE.md` - Complete composition guide

## Features

### ✅ Automated Treefile Deployment
- Jinja2 template for customizable package lists
- Deployed to `/etc/rpm-ostree/treefiles/kinoite.json`
- Easy to version control and replicate

### ✅ Optional Automatic Composition
- Set `rpm_ostree_compose_on_setup: true` to compose during playbook run
- Handles long-running compose process (async with polling)
- Automatically updates repository summary
- Displays available refs after composition

### ✅ Configurable Package Lists
- Base system packages
- Desktop (KDE Plasma) packages
- System tools
- Additional custom packages
- Package exclusions

### ✅ Production Ready
- Timeout handling (1 hour)
- Progress monitoring (30 second polls)
- Error handling
- Verification steps

## Quick Usage

### Default Mode (Manual Composition)

Run the playbook:
```bash
ansible-playbook -i inventory/hosts.yml playbooks/setup-rpm-ostree-repo.yml
```

Then compose manually:
```bash
sudo rpm-ostree compose tree \
  --repo=/srv/ostree/rpm-ostree/kinoite \
  /etc/rpm-ostree/treefiles/kinoite.json

sudo ostree summary -u --repo=/srv/ostree/rpm-ostree/kinoite
```

### Automatic Mode

Edit `group_vars/rpm_ostree_repo_servers.yml`:
```yaml
rpm_ostree_compose_on_setup: true
```

Run the playbook:
```bash
ansible-playbook -i inventory/hosts.yml playbooks/setup-rpm-ostree-repo.yml
```

Image will be composed automatically!

## Configuration

### Basic Configuration

In `group_vars/rpm_ostree_repo_servers.yml`:

```yaml
# Enable automatic composition
rpm_ostree_compose_on_setup: false  # Set to true for automatic

# OSTree ref (branch)
rpm_ostree_ref: fedora/x86_64/kinoite/stable

# Fedora version
rpm_ostree_fedora_release: 41
```

### Add Custom Packages

In `group_vars/rpm_ostree_repo_servers.yml`:

```yaml
rpm_ostree_additional_packages:
  - firefox
  - thunderbird
  - libreoffice
  - gimp
  - vlc
  - docker
  - podman
```

### Advanced Customization

Override any package list in `group_vars/rpm_ostree_repo_servers.yml`:

```yaml
rpm_ostree_base_packages:
  - fedora-release-kinoite
  - kernel
  # ... your packages

rpm_ostree_desktop_packages:
  - plasma-desktop
  # ... your desktop packages

rpm_ostree_exclude_packages:
  - PackageKit
  # ... packages to exclude
```

## Default Packages

The default configuration includes:

**Base System:**
- fedora-release-kinoite, kernel, systemd, rpm-ostree
- dracut-config-generic, nss-altfiles, sssd-client

**KDE Plasma Desktop:**
- plasma-desktop, plasma-workspace, sddm, sddm-breeze
- dolphin, konsole, kate

**System Tools:**
- NetworkManager, firewalld, chrony

**Utilities:**
- vim-enhanced, git, wget, curl, htop, tmux

## Selective Execution

Run only the compose role:

```bash
ansible-playbook -i inventory/hosts.yml \
  playbooks/setup-rpm-ostree-repo.yml \
  --tags compose
```

## Integration

The compose role integrates with existing roles:

```
Setup Flow:
1. rpm_ostree_repo     - Create repository
2. rpm_ostree_selinux  - Configure SELinux
3. rpm_ostree_apache   - Configure web server
4. rpm_ostree_firewall - Configure firewall
5. rpm_ostree_compose  - Deploy treefile & compose (NEW!)
```

## Treefile Template

The treefile is generated from a Jinja2 template with:
- Dynamic package lists
- Conditional sections
- Post-processing commands
- Architecture-specific packages (x86_64)

Template features:
- Clean JSON output
- Proper comma handling
- Commented sections
- Easy to customize

## Timing

**Composition Duration:**
- First compose: 15-30 minutes (downloads all packages)
- Subsequent: 5-15 minutes (only changed packages)

**Playbook with auto-compose:**
- Total time: 20-35 minutes
- Without compose: 2-5 minutes

## Benefits

✅ **Reproducible builds** - Same treefile produces same image
✅ **Version controlled** - Treefile in git
✅ **Automated deployment** - One command to deploy
✅ **Easy customization** - Add packages via variables
✅ **CI/CD ready** - Can be integrated into pipelines
✅ **Idempotent** - Safe to run multiple times

## Documentation

- `COMPOSE_GUIDE.md` - Complete composition documentation
- `README.md` - Updated with compose instructions
- `QUICKSTART.md` - Quick start with auto-compose
- Role defaults - Inline documentation

## Example Workflows

### Development Setup

```yaml
rpm_ostree_compose_on_setup: true
rpm_ostree_additional_packages:
  - vim-enhanced
  - git
  - tmux
  - code
  - docker
  - nodejs
```

Run once, get a fully composed development image!

### Production Deployment

```yaml
rpm_ostree_compose_on_setup: true
rpm_ostree_additional_packages:
  - firefox
  - thunderbird
  - libreoffice
```

Deploy to multiple servers with consistent images.

### Scheduled Updates

Set up cron job on server:
```cron
0 2 * * * rpm-ostree compose tree --repo=/srv/ostree/rpm-ostree/kinoite /etc/rpm-ostree/treefiles/kinoite.json && ostree summary -u --repo=/srv/ostree/rpm-ostree/kinoite
```

Fresh image every night!

## Testing

After composition, verify:

```bash
# Check refs
sudo ostree refs --repo=/srv/ostree/rpm-ostree/kinoite

# Check summary
curl -k https://server:8443/repo/kinoite/summary

# Test client pull
sudo ostree pull kinoite:fedora/x86_64/kinoite/stable
```

## Next Steps

1. Run the playbook with default settings (manual compose)
2. Customize package lists in group_vars
3. Test manual composition
4. Enable automatic composition for production
5. Set up scheduled compositions for updates

---

**Ready to compose images with Ansible!** 🚀

See `COMPOSE_GUIDE.md` for complete documentation.
