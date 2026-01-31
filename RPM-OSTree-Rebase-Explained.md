# Understanding rpm-ostree rebase

## What is rpm-ostree rebase?

`rpm-ostree rebase` **switches your Kinoite system from one OS image source to a completely different one**.

It changes the "upstream" repository that your system gets updates from - similar to changing the remote URL in Git.

## The Core Concept

### Git Analogy

If you're familiar with Git, think of it this way:

```bash
# Git: Switch from one remote repository to another
git remote set-url origin https://new-repository.git
git pull

# rpm-ostree: Switch from one OS repository to another
rpm-ostree rebase new-remote:ref
reboot
```

Both commands change where future updates come from.

## Visual Overview: Before and After Rebase

### Before Rebase (Stock Kinoite)

```
Your Kinoite Machine
├── Currently running: Fedora's official Kinoite image
├── Getting updates from: Fedora's servers (fedoraproject.org)
└── OSTree ref: fedora:fedora/41/x86_64/kinoite

Update source:
  Fedora Official Servers
         ↓
   Your Machine
```

When you run `rpm-ostree upgrade`:
- Checks Fedora's official servers
- Downloads Fedora's standard Kinoite updates
- Gets whatever Fedora ships (no customization)

### After Rebase (Custom Repository)

```
Your Kinoite Machine
├── Currently running: YOUR custom Kinoite image
├── Getting updates from: YOUR repository (192.168.35.35:8443)
└── OSTree ref: kinoite-prod:fedora/x86_64/kinoite/stable

Update source:
  Your Custom Repository
         ↓
   Your Machine
```

When you run `rpm-ostree upgrade`:
- Checks YOUR repository
- Downloads YOUR custom Kinoite updates
- Gets your customizations, pre-installed software, configurations

## How to Rebase: Step-by-Step

### Initial Setup - Switching to Your Custom Repository

```bash
# Step 1: Add your custom repository as a remote
sudo ostree remote add --no-gpg-verify kinoite-prod \
  https://192.168.35.35:8443/repo/kinoite/prod

# Step 2: Rebase to your custom image
sudo rpm-ostree rebase kinoite-prod:fedora/x86_64/kinoite/stable

# Step 3: Reboot to activate
sudo reboot
```

### What Happens During Each Step

#### Step 1: Add Remote

```bash
sudo ostree remote add --no-gpg-verify kinoite-prod \
  https://192.168.35.35:8443/repo/kinoite/prod
```

**What this does**:
- Tells OSTree about your repository
- Saves configuration to `/etc/ostree/remotes.d/`
- `--no-gpg-verify` disables signature checking (for self-signed certs)
- `kinoite-prod` is the nickname for your repository

**Result**: Your system now knows where to find your custom images

#### Step 2: Rebase

```bash
sudo rpm-ostree rebase kinoite-prod:fedora/x86_64/kinoite/stable
```

**What this does**:
1. Contacts your repository at `https://192.168.35.35:8443/repo/kinoite/prod`
2. Downloads your custom OS image commit
3. Downloads all necessary objects (files, directories, metadata)
4. Creates a new deployment with your image
5. Sets it as the default boot option
6. **Current running system stays untouched**

**Download size**:
- First time: ~2-3 GB (complete image)
- Subsequent rebases: Only changed objects (much smaller)

**Status after rebase (before reboot)**:

```
Deployments:
├── Deployment 0: YOUR CUSTOM IMAGE (pending, will boot next)
└── Deployment 1: FEDORA OFFICIAL (currently running)

You are still running Fedora's official image until reboot
```

#### Step 3: Reboot

```bash
sudo reboot
```

**What this does**:
- GRUB bootloader presents boot menu
- Default entry is now your custom image (Deployment 0)
- System boots into your custom image
- Previous deployment kept as rollback option

**Status after reboot**:

```
Deployments:
● Deployment 0: YOUR CUSTOM IMAGE (currently running)
  Deployment 1: FEDORA OFFICIAL (rollback option)

The ● indicates which deployment is currently active
```

## Rebase vs Upgrade: Understanding the Difference

| Command | What It Does | When to Use |
|---------|--------------|-------------|
| `rpm-ostree upgrade` | Gets newer version from **same source** | Regular updates from current repository |
| `rpm-ostree rebase` | Switches to **different source** | Changing repositories, switching environments, or major version upgrades |

### Upgrade Example (Same Source)

```bash
# You're using your custom repo, June version
$ rpm-ostree status
● kinoite-prod:fedora/x86_64/kinoite/stable
    Version: 41.20250615

# Check for updates
$ rpm-ostree upgrade

# Downloads July version from YOUR repo (same source)
● kinoite-prod:fedora/x86_64/kinoite/stable
    Version: 41.20250715

# Still using: kinoite-prod:fedora/x86_64/kinoite/stable
# Just a newer version from same repository
```

### Rebase Example (Different Source)

```bash
# Currently using Fedora official
$ rpm-ostree status
● fedora:fedora/41/x86_64/kinoite

# Switch to your custom repository
$ rpm-ostree rebase kinoite-prod:fedora/x86_64/kinoite/stable

# After reboot
$ rpm-ostree status
● kinoite-prod:fedora/x86_64/kinoite/stable

# Now using: different repository entirely
```

## Common Rebase Scenarios

### Scenario 1: New Workstation Setup

**Situation**: Fresh Kinoite installation using Fedora's official image. You want to switch to your company's custom repository.

```bash
# 1. Fresh install boots into stock Fedora Kinoite
# 2. Add company repository
sudo ostree remote add --no-gpg-verify company-kinoite \
  https://192.168.35.35:8443/repo/kinoite/prod

# 3. Rebase to company image
sudo rpm-ostree rebase company-kinoite:fedora/x86_64/kinoite/stable

# 4. Reboot
sudo reboot

# Result: Now running company-customized Kinoite with:
# - Pre-installed company tools
# - Company wallpaper/branding
# - Security hardening
# - VPN client
# - All required configurations
```

**Time required**: ~5 minutes (mostly download time)

### Scenario 2: Testing in Dev, Then Moving to Prod

**Situation**: You want to test new updates in dev environment first, then switch to stable prod.

```bash
# Initial setup - use dev repository for testing
sudo ostree remote add --no-gpg-verify kinoite-dev \
  https://192.168.35.35:8443/repo/kinoite/dev

sudo rpm-ostree rebase kinoite-dev:fedora/x86_64/kinoite/dev
sudo reboot

# Test for a week - everything works great!

# Switch same machine to prod repository
sudo ostree remote add --no-gpg-verify kinoite-prod \
  https://192.168.35.35:8443/repo/kinoite/prod

sudo rpm-ostree rebase kinoite-prod:fedora/x86_64/kinoite/stable
sudo reboot

# Now using stable production image
```

### Scenario 3: Fedora Version Upgrade

**Situation**: Upgrading from Fedora 40 to Fedora 41.

```bash
# Currently on Fedora 40 Kinoite
$ rpm-ostree status
● fedora:fedora/40/x86_64/kinoite

# Rebase to Fedora 41
sudo rpm-ostree rebase fedora:fedora/41/x86_64/kinoite
sudo reboot

# Now on Fedora 41 Kinoite
$ rpm-ostree status
● fedora:fedora/41/x86_64/kinoite
```

### Scenario 4: Developer vs Production Machines

**Situation**: Different roles need different update cadences.

```bash
# Developer's machine - wants latest updates
sudo ostree remote add --no-gpg-verify kinoite-dev \
  https://192.168.35.35:8443/repo/kinoite/dev

sudo rpm-ostree rebase kinoite-dev:fedora/x86_64/kinoite/dev
sudo reboot

# Gets: Weekly updates, latest features, cutting edge
```

```bash
# Regular employee's machine - wants stability
sudo ostree remote add --no-gpg-verify kinoite-prod \
  https://192.168.35.35:8443/repo/kinoite/prod

sudo rpm-ostree rebase kinoite-prod:fedora/x86_64/kinoite/stable
sudo reboot

# Gets: Monthly updates, tested & validated, stable
```

## What Rebase Changes vs Preserves

Understanding what changes and what stays the same is crucial.

### What Changes (New OS Image)

These come from the new repository's image:

- ✅ All system files in `/usr` (binaries, libraries, system files)
- ✅ Installed packages (from base image)
- ✅ System configuration defaults
- ✅ Kernel version
- ✅ Desktop environment version (KDE Plasma, GNOME, etc.)
- ✅ System services and daemons
- ✅ Default applications

### What Persists (Your Data/Customizations)

These survive the rebase:

- ✅ User files in `/home` (all your documents, pictures, etc.)
- ✅ Your customizations in `/etc` (using 3-way merge)
- ✅ Data in `/var` (logs, databases, container storage)
- ✅ Layered packages you installed with `rpm-ostree install`
- ✅ Flatpak applications (stored in /var)
- ✅ Podman containers and images
- ✅ User settings and preferences

### The 3-Way Merge for /etc

When you rebase, `/etc` uses intelligent merging:

```
Your Current /etc + New Image /etc Defaults = Merged /etc
    (your edits)     (new defaults)           (combined)

Example:
- You edited /etc/fstab → Your changes preserved
- New image has updated /etc/sysctl.conf → New defaults applied
- You edited /etc/hosts, image also changed it → Merge attempted, conflict reported if can't auto-merge
```

## The Complete Rebase Process

### Detailed Step-by-Step Flow

```
┌─────────────────────────────────────────────────────┐
│ Step 1: Add Remote Repository                      │
├─────────────────────────────────────────────────────┤
│ $ sudo ostree remote add kinoite-prod \            │
│     https://192.168.35.35:8443/repo/kinoite/prod   │
│                                                     │
│ Action: Tells OSTree about your repository         │
│ Creates: /etc/ostree/remotes.d/kinoite-prod.conf   │
│ Duration: Instant                                   │
└──────────────────────┬──────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────┐
│ Step 2: Rebase Command                             │
├─────────────────────────────────────────────────────┤
│ $ sudo rpm-ostree rebase \                         │
│     kinoite-prod:fedora/x86_64/kinoite/stable      │
│                                                     │
│ Actions:                                            │
│ 1. Contacts repository server                      │
│ 2. Downloads commit metadata                       │
│ 3. Downloads required objects                      │
│ 4. Verifies checksums                              │
│ 5. Creates new deployment                          │
│ 6. Updates bootloader configuration                │
│                                                     │
│ Download: ~2-3 GB (first time)                     │
│ Duration: 5-15 minutes (depends on connection)     │
└──────────────────────┬──────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────┐
│ Current Status (Before Reboot)                     │
├─────────────────────────────────────────────────────┤
│ $ rpm-ostree status                                │
│                                                     │
│ State: idle                                         │
│ Deployments:                                        │
│   kinoite-prod:fedora/x86_64/kinoite/stable        │
│     Version: 41.20250715 (pending)                 │
│ ● fedora:fedora/41/x86_64/kinoite                  │
│     Version: 41.20250701 (current)                 │
│                                                     │
│ Note: Still running Fedora official image          │
│ New image staged and ready for next boot           │
└──────────────────────┬──────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────┐
│ Step 3: Reboot                                     │
├─────────────────────────────────────────────────────┤
│ $ sudo reboot                                      │
│                                                     │
│ Actions:                                            │
│ 1. GRUB bootloader presents menu                   │
│ 2. Default entry is new deployment                 │
│ 3. Kernel boots with new OS image                  │
│ 4. systemd starts services from new image          │
│                                                     │
│ Duration: ~30-60 seconds                           │
└──────────────────────┬──────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────┐
│ After Reboot - New System Active                   │
├─────────────────────────────────────────────────────┤
│ $ rpm-ostree status                                │
│                                                     │
│ State: idle                                         │
│ Deployments:                                        │
│ ● kinoite-prod:fedora/x86_64/kinoite/stable        │
│     Version: 41.20250715 (current)                 │
│   fedora:fedora/41/x86_64/kinoite                  │
│     Version: 41.20250701 (rollback)                │
│                                                     │
│ Now running: Your custom image                     │
│ Future updates: Come from your repository          │
│ Rollback available: Previous deployment kept       │
└─────────────────────────────────────────────────────┘
```

## Checking Your Current Status

### View Deployment Information

```bash
rpm-ostree status
```

**Example output**:

```
State: idle
Deployments:
● kinoite-prod:fedora/x86_64/kinoite/stable
    Version: 41.20250715.0 (2025-07-15T10:30:00Z)
    Commit: a3f8b2c4d5e6f7a8b9c0d1e2f3a4b5c6d7e8f9a0
    GPGSignature: (unsigned)

  fedora:fedora/41/x86_64/kinoite
    Version: 41.20250701.0 (2025-07-01T14:00:00Z)
    Commit: b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0a1b2c3
    GPGSignature: valid
```

**What the symbols mean**:
- `●` = Currently booted deployment
- No symbol = Available deployment (rollback option)

**Information shown**:
- **Remote:Ref** - Which repository and branch
- **Version** - Image version identifier
- **Commit** - Unique commit hash
- **Timestamp** - When image was composed
- **GPGSignature** - Signature verification status

### View Available Remotes

```bash
ostree remote list
```

**Example output**:
```
fedora
kinoite-prod
kinoite-dev
```

### View Remote Details

```bash
ostree remote show-url kinoite-prod
```

**Output**:
```
https://192.168.35.35:8443/repo/kinoite/prod
```

## Safety Features and Rollback

### Automatic Rollback Protection

After a rebase, your previous deployment is **automatically kept** as a safety net.

```
Current State:
├── Deployment 0: YOUR NEW IMAGE (current)
└── Deployment 1: PREVIOUS IMAGE (automatic rollback)
```

### How to Rollback

#### Method 1: Using rpm-ostree (Immediate)

```bash
# If you've rebooted and don't like the new image
sudo rpm-ostree rollback
sudo reboot

# System will boot into previous deployment
```

#### Method 2: Using GRUB Boot Menu (During Boot)

```
1. Reboot system
2. When GRUB menu appears, press ESC or hold Shift
3. Select the previous deployment entry
4. System boots into old image
5. If you want to make it permanent:
   sudo rpm-ostree rollback
```

### Before You Reboot - Cancel Pending Deployment

If you rebased but haven't rebooted yet and changed your mind:

```bash
# View pending deployment
rpm-ostree status

# Remove pending deployment
sudo rpm-ostree cleanup -p

# This removes the staged deployment
# Next reboot will use current deployment
```

## Real-World Use Case: Enterprise Deployment

### Scenario: 50 Employee Workstations

**Challenge**: Deploy custom Kinoite to all company workstations with:
- Company VPN client
- Custom security policies
- Pre-installed development tools
- Company branding
- Standardized configuration

**Solution Using Rebase**:

#### On Repository Server (Once)

```bash
# 1. Create custom treefile with required packages
# 2. Compose image
sudo rpm-ostree compose tree \
  --repo=/srv/ostree/rpm-ostree/prod \
  /etc/rpm-ostree/treefiles/kinoite-prod.json

# 3. Update repository summary
sudo ostree summary -u --repo=/srv/ostree/rpm-ostree/prod

# Time: 30 minutes
```

#### On Each Workstation (5 minutes per machine)

```bash
# 1. Install stock Fedora Kinoite from USB
# 2. Boot into system
# 3. Run setup script:

#!/bin/bash
# company-kinoite-setup.sh

# Add company repository
sudo ostree remote add --no-gpg-verify company-kinoite \
  https://repo.company.internal:8443/repo/kinoite/prod

# Rebase to company image
sudo rpm-ostree rebase company-kinoite:fedora/x86_64/kinoite/stable

# Reboot
echo "Rebooting in 10 seconds..."
sleep 10
sudo reboot
```

**Result**:
- All 50 machines identical
- VPN client pre-installed
- Security policies applied
- Development tools ready
- Company branding applied
- No manual configuration needed
- Future updates controlled centrally

### Time Comparison

**Without Custom Repository (Manual Setup)**:
```
50 machines × 2 hours each = 100 hours
(Installing software, configuring, troubleshooting individual failures)
```

**With Custom Repository (Using Rebase)**:
```
Server composition: 30 minutes
50 machines × 10 minutes each = 500 minutes (8.3 hours)
Total: ~9 hours
```

**Savings**: 91 hours of work!

## Advanced Rebase Options

### Rebase with Custom Options

```bash
# Rebase and immediately reboot (no confirmation)
sudo rpm-ostree rebase kinoite-prod:fedora/x86_64/kinoite/stable --reboot

# Rebase to specific commit hash (pinning)
sudo rpm-ostree rebase kinoite-prod:fedora/x86_64/kinoite/stable \
  --commit=a3f8b2c4d5e6f7a8b9c0d1e2f3a4b5c6d7e8f9a0

# Rebase without pulling (use cached data)
sudo rpm-ostree rebase kinoite-prod:fedora/x86_64/kinoite/stable --cache-only

# Preview what would change (dry run)
sudo rpm-ostree rebase kinoite-prod:fedora/x86_64/kinoite/stable --preview
```

### Rebase with Package Layering

Your layered packages are preserved during rebase:

```bash
# Current state: Fedora official + vim installed
$ rpm-ostree status
● fedora:fedora/41/x86_64/kinoite
  LayeredPackages: vim

# Rebase to custom repo
$ sudo rpm-ostree rebase kinoite-prod:fedora/x86_64/kinoite/stable

# After rebase: Custom image + vim still layered
$ rpm-ostree status
● kinoite-prod:fedora/x86_64/kinoite/stable
  LayeredPackages: vim
```

## Troubleshooting Common Issues

### Issue 1: "Repository not found" Error

```bash
$ sudo rpm-ostree rebase kinoite-prod:fedora/x86_64/kinoite/stable
error: Remote "kinoite-prod" not found
```

**Solution**: Add the remote first

```bash
sudo ostree remote add --no-gpg-verify kinoite-prod \
  https://192.168.35.35:8443/repo/kinoite/prod
```

### Issue 2: Connection Timeout

```bash
$ sudo rpm-ostree rebase kinoite-prod:fedora/x86_64/kinoite/stable
error: Timeout while fetching metadata
```

**Possible causes and solutions**:

1. **Repository server not accessible**
   ```bash
   # Test connectivity
   curl -k https://192.168.35.35:8443/repo/kinoite/prod/config
   ```

2. **Firewall blocking**
   ```bash
   # On repository server, verify firewall
   sudo firewall-cmd --list-ports
   # Should show: 8443/tcp
   ```

3. **Apache not running**
   ```bash
   # On repository server
   sudo systemctl status httpd
   ```

### Issue 3: GPG Signature Verification Failed

```bash
$ sudo rpm-ostree rebase kinoite-prod:fedora/x86_64/kinoite/stable
error: GPG signature verification failed
```

**Solution**: Use `--no-gpg-verify` flag (for unsigned commits)

```bash
sudo ostree remote add --no-gpg-verify kinoite-prod \
  https://192.168.35.35:8443/repo/kinoite/prod
```

Or import GPG key if commits are signed:

```bash
sudo ostree remote add kinoite-prod \
  https://192.168.35.35:8443/repo/kinoite/prod \
  --gpg-import=/path/to/public-key.asc
```

### Issue 4: Insufficient Disk Space

```bash
$ sudo rpm-ostree rebase kinoite-prod:fedora/x86_64/kinoite/stable
error: Not enough space in /sysroot
```

**Solution**: Free up space

```bash
# Remove old deployments
sudo rpm-ostree cleanup -b

# Remove old OSTree objects
sudo ostree prune --refs-only --depth=2

# Check disk space
df -h /sysroot
```

### Issue 5: Rebase Interrupted

If rebase is interrupted (network failure, power loss):

```bash
# The current running system is unaffected
# Just re-run the rebase command
sudo rpm-ostree rebase kinoite-prod:fedora/x86_64/kinoite/stable

# rpm-ostree will resume or restart download as needed
```

## Best Practices

### 1. Test Before Wide Deployment

```bash
# Test on a single machine first
sudo rpm-ostree rebase kinoite-dev:fedora/x86_64/kinoite/dev
# Validate for 1 week

# Then rebase to prod and deploy widely
sudo rpm-ostree rebase kinoite-prod:fedora/x86_64/kinoite/stable
```

### 2. Document Your Repository Configuration

Create a setup script for consistency:

```bash
#!/bin/bash
# setup-company-kinoite.sh

REPO_URL="https://repo.company.internal:8443/repo/kinoite/prod"
REMOTE_NAME="company-kinoite"
REF="fedora/x86_64/kinoite/stable"

sudo ostree remote add --no-gpg-verify "$REMOTE_NAME" "$REPO_URL"
sudo rpm-ostree rebase "$REMOTE_NAME:$REF"
echo "Rebase complete. Reboot to activate."
```

### 3. Keep One Old Deployment

```bash
# Don't clean up too aggressively
# Keep at least one rollback option

# This keeps only the current and one previous
sudo rpm-ostree cleanup -b
```

### 4. Verify After Rebase

```bash
# After rebooting into new image
rpm-ostree status

# Verify you're on the correct remote/ref
# Check that expected packages are present
rpm -qa | grep <important-package>
```

### 5. Schedule Rebases During Maintenance Windows

```bash
# For production systems, schedule during off-hours
# Create a cron job or systemd timer

# Example: Reboot at 2 AM on Sunday
sudo systemctl edit --full --force rebase-and-reboot.timer

[Unit]
Description=Weekly Rebase and Reboot

[Timer]
OnCalendar=Sun *-*-* 02:00:00
Persistent=true

[Install]
WantedBy=timers.target
```

## Summary

### What rpm-ostree rebase Does

**In Simple Terms**:
Switches your Kinoite system from one OS image source to another, changing where future updates come from.

**Key Points**:
- ✅ Changes which repository you get updates from
- ✅ Downloads complete new OS image
- ✅ Creates new deployment (old one kept for rollback)
- ✅ Requires reboot to activate
- ✅ Preserves user data and configurations
- ✅ Safe - can always rollback

### When to Use Rebase

- **Initial Setup**: Switch from Fedora official to your custom repository
- **Environment Switching**: Move between dev and prod repositories
- **Version Upgrades**: Upgrade to new Fedora version
- **Repository Migration**: Change to different repository server

### The Relationship to Your Custom Repository

**Your repository is only useful if clients rebase to it**:

1. You compose custom images on your repository server
2. Repository serves images via HTTPS
3. **Clients must rebase to actually use your custom images**
4. After rebase, clients get all future updates from your repository

**Without rebase**: Clients stay on Fedora official images (your repository unused)

**With rebase**: Clients use your custom images (your repository actively serving your configurations)

### Command Quick Reference

```bash
# Add repository remote
sudo ostree remote add --no-gpg-verify <name> <url>

# Rebase to new repository
sudo rpm-ostree rebase <remote>:<ref>

# Reboot to activate
sudo reboot

# Check status
rpm-ostree status

# Rollback if needed
sudo rpm-ostree rollback

# Clean up old deployments
sudo rpm-ostree cleanup -b
```

---

**rpm-ostree rebase is the bridge between your custom repository and your clients** - it's the command that makes your entire infrastructure useful by actually deploying your customized images to end-user machines.
