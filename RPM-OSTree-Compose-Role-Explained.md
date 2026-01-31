# Understanding the rpm_ostree_compose Role

## What Does the rpm_ostree_compose Role Do?

The `rpm_ostree_compose` role **manages the treefile configuration and optionally composes (builds) OS images** for your Kinoite repository. It's the role that defines what goes into your custom operating system images.

## Simple Analogy

Think of it like:
- **Recipe book**: The treefile is your recipe
- **Kitchen setup**: The compose role sets up your kitchen with the recipes
- **Cooking**: Optionally, it can also cook the meal (compose the OS image)
- **Menu**: It creates separate recipes for dev and prod environments

## What the Role Actually Does

### Primary Functions

```yaml
1. Creates treefile directory
   └─ /etc/rpm-ostree/treefiles/

2. Deploys treefile templates
   ├─ kinoite-dev.json (for development)
   └─ kinoite-prod.json (for production)

3. Optionally composes OS images (if enabled)
   ├─ Compose dev image
   └─ Compose prod image

4. Updates repository summaries (if composed)
   ├─ Update dev repo summary
   └─ Update prod repo summary

5. Displays instructions
   └─ Shows how to manually compose if not automatic
```

## Before and After

### Before the Role Runs

```bash
$ ls /etc/rpm-ostree/
ls: cannot access '/etc/rpm-ostree/': No such file or directory

$ ls /srv/ostree/rpm-ostree/dev/refs/heads/
# Empty - no OS images yet
```

### After the Role Runs (Default: No Auto-Compose)

```bash
$ ls /etc/rpm-ostree/treefiles/
kinoite-dev.json
kinoite-prod.json

$ cat /etc/rpm-ostree/treefiles/kinoite-dev.json
{
  "ref": "fedora/x86_64/kinoite/dev",
  "repos": ["fedora", "fedora-updates"],
  "packages": [
    "fedora-release-kinoite",
    "kernel",
    "systemd",
    ...
  ]
}

$ ls /srv/ostree/rpm-ostree/dev/refs/heads/
# Still empty - treefiles deployed but not composed yet
```

### After Manual Compose (Following Instructions)

```bash
$ sudo rpm-ostree compose tree \
    --repo=/srv/ostree/rpm-ostree/dev \
    /etc/rpm-ostree/treefiles/kinoite-dev.json

$ sudo ostree summary -u --repo=/srv/ostree/rpm-ostree/dev

$ ls /srv/ostree/rpm-ostree/dev/refs/heads/
fedora/
└── x86_64/
    └── kinoite/
        └── dev

# Now repository contains OS image!
```

## What is a Treefile?

A **treefile** is a JSON manifest that defines what goes into your operating system image. It's like a blueprint or recipe.

### Treefile Structure

```json
{
  "ref": "fedora/x86_64/kinoite/dev",
  "repos": [
    "fedora",
    "fedora-updates"
  ],
  "selinux": true,
  "boot-location": "new",
  "tmp-is-dir": true,

  "packages": [
    "fedora-release-kinoite",
    "kernel",
    "systemd",
    "rpm-ostree",
    "plasma-desktop",
    "plasma-workspace",
    "NetworkManager",
    "firewalld",
    "vim-enhanced",
    "git"
  ],

  "exclude-packages": [
    "PackageKit"
  ],

  "postprocess": [
    "ln -sf /usr/lib/systemd/system/graphical.target /etc/systemd/system/default.target",
    "systemctl enable sddm.service"
  ]
}
```

### Treefile Fields Explained

| Field | Purpose | Example |
|-------|---------|---------|
| **ref** | OSTree branch name | `fedora/x86_64/kinoite/dev` |
| **repos** | DNF repositories to use | `["fedora", "fedora-updates"]` |
| **selinux** | Enable SELinux | `true` |
| **boot-location** | Bootloader config location | `"new"` (modern systems) |
| **tmp-is-dir** | Make /tmp a directory | `true` |
| **packages** | RPMs to include in image | `["kernel", "plasma-desktop", ...]` |
| **exclude-packages** | RPMs to explicitly exclude | `["PackageKit"]` |
| **postprocess** | Commands to run after install | Enable services, create symlinks |

## The Role's Tasks Breakdown

Looking at `roles/rpm_ostree_compose/tasks/main.yml`:

### Task 1: Create Treefile Directory

```yaml
- name: Create treefile directory
  ansible.builtin.file:
    path: /etc/rpm-ostree/treefiles
    state: directory
    mode: '0755'
    owner: root
    group: root
```

**What this does**:
- Creates `/etc/rpm-ostree/treefiles/` directory
- Sets permissions to 755 (readable by all, writable by root)
- Standard location for treefile storage

**Result**:
```bash
$ ls -ld /etc/rpm-ostree/treefiles
drwxr-xr-x. 2 root root 4096 Jan 31 10:00 /etc/rpm-ostree/treefiles
```

### Task 2: Deploy Dev Treefile

```yaml
- name: Deploy dev treefile template
  ansible.builtin.template:
    src: kinoite.json.j2
    dest: /etc/rpm-ostree/treefiles/kinoite-dev.json
    mode: '0644'
    owner: root
    group: root
  vars:
    rpm_ostree_ref: "{{ rpm_ostree_ref_dev }}"
  register: treefile_dev_deployed
```

**What this does**:
- Takes template `kinoite.json.j2`
- Substitutes `rpm_ostree_ref` variable with dev value
- Creates `/etc/rpm-ostree/treefiles/kinoite-dev.json`
- Sets as read-only for non-root users (644)

**Template substitution**:
```jinja2
{
  "ref": "{{ rpm_ostree_ref }}",  # Becomes: fedora/x86_64/kinoite/dev
  ...
}
```

**Result**:
```bash
$ cat /etc/rpm-ostree/treefiles/kinoite-dev.json
{
  "ref": "fedora/x86_64/kinoite/dev",
  ...
}
```

### Task 3: Deploy Prod Treefile

```yaml
- name: Deploy prod treefile template
  ansible.builtin.template:
    src: kinoite.json.j2
    dest: /etc/rpm-ostree/treefiles/kinoite-prod.json
    mode: '0644'
    owner: root
    group: root
  vars:
    rpm_ostree_ref: "{{ rpm_ostree_ref_prod }}"
  register: treefile_prod_deployed
```

**What this does**:
- Same as Task 2, but for production
- Uses `rpm_ostree_ref_prod` variable
- Creates separate treefile for stable environment

**Result**:
```bash
$ cat /etc/rpm-ostree/treefiles/kinoite-prod.json
{
  "ref": "fedora/x86_64/kinoite/stable",
  ...
}
```

### Task 4: Display Treefile Locations

```yaml
- name: Display treefile locations
  ansible.builtin.debug:
    msg:
      - "Treefiles deployed:"
      - "  DEV:  /etc/rpm-ostree/treefiles/kinoite-dev.json (ref: {{ rpm_ostree_ref_dev }})"
      - "  PROD: /etc/rpm-ostree/treefiles/kinoite-prod.json (ref: {{ rpm_ostree_ref_prod }})"
```

**What this does**:
- Shows summary of deployed treefiles
- Confirms deployment success
- Shows OSTree ref for each environment

**Output**:
```
TASK [rpm_ostree_compose : Display treefile locations]
ok: [server] => {
    "msg": [
        "Treefiles deployed:",
        "  DEV:  /etc/rpm-ostree/treefiles/kinoite-dev.json (ref: fedora/x86_64/kinoite/dev)",
        "  PROD: /etc/rpm-ostree/treefiles/kinoite-prod.json (ref: fedora/x86_64/kinoite/stable)"
    ]
}
```

### Task 5: Check if Composition is Enabled

```yaml
- name: Check if image composition is enabled
  ansible.builtin.debug:
    msg: "Image composition will be performed for: {{ rpm_ostree_compose_environments | join(', ') }}"
  when: rpm_ostree_compose_on_setup | bool
```

**What this does**:
- Checks if automatic composition is enabled
- Shows which environments will be composed
- Only runs if `rpm_ostree_compose_on_setup: true`

**Default**: This task is skipped (composition disabled by default)

### Task 6: Compose Dev Image (Optional)

```yaml
- name: Compose dev rpm-ostree image
  ansible.builtin.command:
    cmd: >
      rpm-ostree compose tree
      --repo={{ rpm_ostree_repo_dev_path }}
      {{ rpm_ostree_treefile_path }}/kinoite-dev.json
  when:
    - rpm_ostree_compose_on_setup | bool
    - "'dev' in rpm_ostree_compose_environments"
  register: compose_dev_result
  async: 3600  # 1 hour timeout
  poll: 30     # Check every 30 seconds
  changed_when: true
```

**What this does** (if enabled):
- Runs `rpm-ostree compose tree` command
- Downloads ~2GB of RPM packages from Fedora repos
- Installs packages into temporary root
- Runs postprocess scripts
- Creates OSTree commit
- Stores in dev repository

**Command executed**:
```bash
rpm-ostree compose tree \
  --repo=/srv/ostree/rpm-ostree/dev \
  /etc/rpm-ostree/treefiles/kinoite-dev.json
```

**Async settings**:
- `async: 3600` - Maximum 1 hour to complete
- `poll: 30` - Check status every 30 seconds
- Allows long-running composition without timeout

**Default**: Skipped (composition disabled)

### Task 7: Compose Prod Image (Optional)

```yaml
- name: Compose prod rpm-ostree image
  ansible.builtin.command:
    cmd: >
      rpm-ostree compose tree
      --repo={{ rpm_ostree_repo_prod_path }}
      {{ rpm_ostree_treefile_path }}/kinoite-prod.json
  when:
    - rpm_ostree_compose_on_setup | bool
    - "'prod' in rpm_ostree_compose_environments"
  register: compose_prod_result
  async: 3600
  poll: 30
  changed_when: true
```

**What this does** (if enabled):
- Same as Task 6, but for production
- Creates stable OS image
- Stores in prod repository

**Default**: Skipped (composition disabled)

### Task 8-9: Display Compose Results (Optional)

```yaml
- name: Display dev compose result
  ansible.builtin.debug:
    msg: "DEV image composition completed successfully"
  when:
    - rpm_ostree_compose_on_setup | bool
    - "'dev' in rpm_ostree_compose_environments"
    - compose_dev_result is defined
    - compose_dev_result.rc == 0
```

**What this does** (if composition ran):
- Confirms successful composition
- Shows which environment completed

**Default**: Skipped

### Task 10-11: Update Repository Summaries (Optional)

```yaml
- name: Update dev repository summary
  ansible.builtin.command:
    cmd: ostree summary -u --repo={{ rpm_ostree_repo_dev_path }}
  when:
    - rpm_ostree_compose_on_setup | bool
    - "'dev' in rpm_ostree_compose_environments"
    - compose_dev_result is defined
    - compose_dev_result.rc == 0
  changed_when: true
```

**What this does** (if composition succeeded):
- Runs `ostree summary -u` command
- Updates repository metadata index
- Makes new OS image visible to clients

**Why this is needed**:
- OSTree clients check the summary file first
- Summary contains list of available refs (branches)
- Without summary update, clients won't see new commit

**Command executed**:
```bash
ostree summary -u --repo=/srv/ostree/rpm-ostree/dev
```

**Default**: Skipped

### Task 12-13: Verify Composed Images (Optional)

```yaml
- name: Verify dev composed image
  ansible.builtin.command:
    cmd: ostree refs --repo={{ rpm_ostree_repo_dev_path }}
  register: ostree_refs_dev
  when:
    - rpm_ostree_compose_on_setup | bool
    - "'dev' in rpm_ostree_compose_environments"
    - compose_dev_result is defined
    - compose_dev_result.rc == 0
  changed_when: false
```

**What this does** (if composition succeeded):
- Lists available refs in repository
- Verifies the compose created expected ref
- Stores output for display

**Command executed**:
```bash
ostree refs --repo=/srv/ostree/rpm-ostree/dev
```

**Expected output**:
```
fedora/x86_64/kinoite/dev
```

**Default**: Skipped

### Task 14-15: Display Available Refs (Optional)

```yaml
- name: Display available dev refs
  ansible.builtin.debug:
    msg:
      - "Available OSTree refs in DEV repository:"
      - "{{ ostree_refs_dev.stdout_lines }}"
  when:
    - rpm_ostree_compose_on_setup | bool
    - "'dev' in rpm_ostree_compose_environments"
    - ostree_refs_dev is defined
```

**What this does** (if verification ran):
- Shows which OS images exist in repository
- Confirms successful composition

**Output**:
```
TASK [rpm_ostree_compose : Display available dev refs]
ok: [server] => {
    "msg": [
        "Available OSTree refs in DEV repository:",
        "fedora/x86_64/kinoite/dev"
    ]
}
```

**Default**: Skipped

### Task 16: Display Manual Composition Instructions

```yaml
- name: Display manual compose instructions
  ansible.builtin.debug:
    msg:
      - "=========================================="
      - "Image composition was NOT performed automatically"
      - "=========================================="
      - "To compose images manually:"
      - ""
      - "DEV:"
      - "  sudo rpm-ostree compose tree \\"
      - "    --repo=/srv/ostree/rpm-ostree/dev \\"
      - "    /etc/rpm-ostree/treefiles/kinoite-dev.json"
      - "  sudo ostree summary -u --repo=/srv/ostree/rpm-ostree/dev"
      - ""
      - "PROD:"
      - "  sudo rpm-ostree compose tree \\"
      - "    --repo=/srv/ostree/rpm-ostree/prod \\"
      - "    /etc/rpm-ostree/treefiles/kinoite-prod.json"
      - "  sudo ostree summary -u --repo=/srv/ostree/rpm-ostree/prod"
      - ""
      - "To enable automatic composition, set in group_vars:"
      - "  rpm_ostree_compose_on_setup: true"
      - "  rpm_ostree_compose_environments: [dev, prod]"
  when: not (rpm_ostree_compose_on_setup | bool)
```

**What this does** (default behavior):
- Shows how to manually compose images
- Provides exact commands to run
- Explains how to enable automatic composition

**Output**:
```
TASK [rpm_ostree_compose : Display manual compose instructions]
ok: [server] => {
    "msg": [
        "==========================================",
        "Image composition was NOT performed automatically",
        "==========================================",
        "To compose images manually:",
        "",
        "DEV:",
        "  sudo rpm-ostree compose tree \\",
        "    --repo=/srv/ostree/rpm-ostree/dev \\",
        "    /etc/rpm-ostree/treefiles/kinoite-dev.json",
        "  sudo ostree summary -u --repo=/srv/ostree/rpm-ostree/dev",
        ...
    ]
}
```

**Default**: This task RUNS (manual composition recommended)

## Role Configuration Variables

### From `roles/rpm_ostree_compose/defaults/main.yml`

```yaml
# Repository configuration
rpm_ostree_repo_base_path: /srv/ostree/rpm-ostree
rpm_ostree_repo_dev_path: "{{ rpm_ostree_repo_base_path }}/dev"
rpm_ostree_repo_prod_path: "{{ rpm_ostree_repo_base_path }}/prod"

# Treefile configuration
rpm_ostree_treefile_name: kinoite.json
rpm_ostree_treefile_path: /etc/rpm-ostree/treefiles

# Compose configuration
rpm_ostree_compose_on_setup: false  # Automatic composition disabled
rpm_ostree_ref: fedora/x86_64/kinoite/stable
rpm_ostree_ref_dev: fedora/x86_64/kinoite/dev
rpm_ostree_ref_prod: fedora/x86_64/kinoite/stable
rpm_ostree_compose_environments: []  # Empty = no auto-compose

# Fedora version
rpm_ostree_fedora_release: 41

# Base packages (minimal system)
rpm_ostree_base_packages:
  - fedora-release-kinoite
  - kernel
  - systemd
  - rpm-ostree
  - dracut-config-generic
  - nss-altfiles
  - sssd-client

# Desktop packages (KDE Plasma)
rpm_ostree_desktop_packages:
  - plasma-desktop
  - plasma-workspace
  - sddm
  - sddm-breeze
  - dolphin
  - konsole
  - kate

# System packages
rpm_ostree_system_packages:
  - NetworkManager
  - firewalld
  - chrony

# Additional packages (customizable)
rpm_ostree_additional_packages:
  - vim-enhanced
  - git
  - wget
  - curl
  - htop
  - tmux

# Packages to exclude
rpm_ostree_exclude_packages:
  - PackageKit

# Post-processing commands
rpm_ostree_postprocess_commands:
  - ln -sf /usr/lib/systemd/system/graphical.target /etc/systemd/system/default.target
  - systemctl enable sddm.service
```

### Customizing Packages

**In `group_vars/rpm_ostree_repo_servers.yml`**:

```yaml
# Add custom packages
rpm_ostree_additional_packages:
  - vim-enhanced
  - git
  - tmux
  - company-vpn-client  # Custom package
  - development-tools   # Package group

# Exclude unwanted packages
rpm_ostree_exclude_packages:
  - PackageKit
  - firefox  # Use Flatpak version instead
```

### Enabling Automatic Composition

**In `group_vars/rpm_ostree_repo_servers.yml`**:

```yaml
# Enable automatic composition during playbook run
rpm_ostree_compose_on_setup: true

# Specify which environments to compose
rpm_ostree_compose_environments:
  - dev      # Compose dev only
  # - prod   # Uncomment to also compose prod
```

**Warning**: First-time composition downloads ~2GB and takes 15-30 minutes. Only enable for unattended runs.

## Manual Composition Workflow (Recommended)

### Step 1: Run the Playbook (Deploys Treefiles)

```bash
ansible-playbook -i inventory/hosts.yml playbooks/setup-rpm-ostree-repo.yml
```

**Result**: Treefiles deployed, but no OS images composed

### Step 2: Compose Dev Image Manually

```bash
# SSH to repository server
ssh root@192.168.35.35

# Compose dev image
sudo rpm-ostree compose tree \
  --repo=/srv/ostree/rpm-ostree/dev \
  /etc/rpm-ostree/treefiles/kinoite-dev.json
```

**What happens**:
```
Downloading packages from Fedora repositories...
[████████████████████] 1847/1847 packages

Installing packages...
Running postprocess scripts...
Creating OSTree commit...
Commit: a3f8b2c4d5e6f7a8b9c0d1e2f3a4b5c6d7e8f9a0

Composition complete!
```

**Time**: 15-30 minutes (first time, downloads packages)

### Step 3: Update Repository Summary

```bash
sudo ostree summary -u --repo=/srv/ostree/rpm-ostree/dev
```

**What this does**:
- Updates repository metadata
- Makes new commit visible to clients

### Step 4: Verify Composition

```bash
# List available refs
ostree refs --repo=/srv/ostree/rpm-ostree/dev
# Output: fedora/x86_64/kinoite/dev

# Show commit details
ostree log --repo=/srv/ostree/rpm-ostree/dev fedora/x86_64/kinoite/dev
```

**Output**:
```
commit a3f8b2c4d5e6f7a8b9c0d1e2f3a4b5c6d7e8f9a0
ContentChecksum: b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0a1b2c3
Date: 2025-01-31 10:30:00 +0000
Version: 41.20250131.0

    Fedora Kinoite 41 (Development)
```

### Step 5: Test on Dev Client

```bash
# On a test Kinoite machine
sudo ostree remote add --no-gpg-verify kinoite-dev \
  https://192.168.35.35:8443/repo/kinoite/dev

sudo rpm-ostree rebase kinoite-dev:fedora/x86_64/kinoite/dev
sudo reboot
```

### Step 6: Validate for 1 Week

Test the dev image thoroughly:
- Boot successfully
- All packages present
- Custom configurations applied
- No errors in logs
- Applications work correctly

### Step 7: Promote to Production

After successful validation:

```bash
ansible-playbook -i inventory/hosts.yml playbooks/promote-to-prod.yml
```

**What this does**:
- Copies tested commit from dev to prod
- Creates stable ref in prod repository
- Updates prod repository summary

### Step 8: Deploy to Production Clients

```bash
# On production Kinoite machines
sudo ostree remote add --no-gpg-verify kinoite-prod \
  https://192.168.35.35:8443/repo/kinoite/prod

sudo rpm-ostree rebase kinoite-prod:fedora/x86_64/kinoite/stable
sudo reboot
```

## Automatic Composition Workflow (Optional)

### Configuration

**In `group_vars/rpm_ostree_repo_servers.yml`**:

```yaml
rpm_ostree_compose_on_setup: true
rpm_ostree_compose_environments:
  - dev
```

### Running the Playbook

```bash
ansible-playbook -i inventory/hosts.yml playbooks/setup-rpm-ostree-repo.yml
```

**What happens**:
1. Deploys treefiles (same as manual)
2. **Automatically composes dev image**
3. **Updates dev repository summary**
4. Verifies composition
5. Shows available refs

**Playbook output**:
```
TASK [rpm_ostree_compose : Compose dev rpm-ostree image]
changed: [server]

TASK [rpm_ostree_compose : Display dev compose result]
ok: [server] => {
    "msg": "DEV image composition completed successfully"
}

TASK [rpm_ostree_compose : Update dev repository summary]
changed: [server]

TASK [rpm_ostree_compose : Display available dev refs]
ok: [server] => {
    "msg": [
        "Available OSTree refs in DEV repository:",
        "fedora/x86_64/kinoite/dev"
    ]
}
```

**Time**: Playbook takes 15-30 minutes longer (composition time)

### When to Use Automatic Composition

**Good use cases**:
- Scheduled/automated runs (cron, CI/CD)
- Initial server setup (one-time)
- Known working configuration

**Avoid automatic composition when**:
- First time testing configuration
- Making treefile changes (might fail)
- Interactive playbook runs (long wait time)
- Testing new packages

## Dev vs Prod Treefiles

### Key Differences

| Aspect | Dev Treefile | Prod Treefile |
|--------|--------------|---------------|
| **Ref** | `fedora/x86_64/kinoite/dev` | `fedora/x86_64/kinoite/stable` |
| **Purpose** | Testing, validation | Stable deployments |
| **Update Frequency** | Weekly or as-needed | Monthly or after validation |
| **Package Selection** | Same base + testing packages | Same base + validated packages |
| **Risk Tolerance** | Higher (testing ground) | Lower (production use) |

### Same Treefile Template

Both use the same `kinoite.json.j2` template, just with different `ref` values:

**Template**:
```jinja2
{
  "ref": "{{ rpm_ostree_ref }}",
  "packages": {{ rpm_ostree_all_packages | to_json }},
  ...
}
```

**Dev treefile** (after template substitution):
```json
{
  "ref": "fedora/x86_64/kinoite/dev",
  ...
}
```

**Prod treefile** (after template substitution):
```json
{
  "ref": "fedora/x86_64/kinoite/stable",
  ...
}
```

## Customizing the Treefile

### Adding Packages

**In `group_vars/rpm_ostree_repo_servers.yml`**:

```yaml
rpm_ostree_additional_packages:
  - vim-enhanced
  - git
  - tmux
  - htop
  - neofetch
  - company-vpn-client
```

### Excluding Packages

```yaml
rpm_ostree_exclude_packages:
  - PackageKit      # Conflicts with rpm-ostree workflow
  - firefox         # Using Flatpak version instead
  - thunderbird     # Not needed on all machines
```

### Adding Postprocess Commands

```yaml
rpm_ostree_postprocess_commands:
  - ln -sf /usr/lib/systemd/system/graphical.target /etc/systemd/system/default.target
  - systemctl enable sddm.service
  - systemctl enable firewalld.service
  - cp /usr/share/backgrounds/company-wallpaper.jpg /usr/share/backgrounds/default.jpg
  - echo "Welcome to Company Kinoite" > /etc/motd
```

### Customizing Fedora Repositories

**Edit the template** (`roles/rpm_ostree_compose/templates/kinoite.json.j2`):

```json
{
  "ref": "{{ rpm_ostree_ref }}",
  "repos": [
    "fedora",
    "fedora-updates",
    "rpmfusion-free",
    "rpmfusion-free-updates",
    "company-internal-repo"
  ],
  ...
}
```

**Note**: Repository config files must exist on server in `/etc/yum.repos.d/`

## Composition Process Details

### What Happens During Composition

```
1. Preparation Phase
   ├─ Read treefile
   ├─ Parse package lists
   └─ Validate repository URLs

2. Download Phase
   ├─ Download RPM packages from repos (~2GB)
   ├─ Verify package signatures
   └─ Cache packages locally

3. Installation Phase
   ├─ Create temporary root filesystem
   ├─ Install all packages
   ├─ Resolve dependencies
   └─ Configure RPM database

4. Postprocess Phase
   ├─ Run postprocess scripts
   ├─ Enable services
   ├─ Create symlinks
   └─ Set default target

5. Commit Phase
   ├─ Create OSTree commit
   ├─ Calculate checksums
   ├─ Deduplicate objects
   └─ Store in repository

6. Finalization
   ├─ Update refs
   ├─ Clean up temporary files
   └─ Display commit hash
```

### Time and Resource Requirements

| Phase | Time (First Run) | Time (Subsequent) | Disk Space |
|-------|------------------|-------------------|------------|
| Download | 10-20 min | 2-5 min (cached) | ~2-3 GB |
| Installation | 3-5 min | 2-4 min | ~4 GB (temp) |
| Postprocess | 1-2 min | 1-2 min | Minimal |
| Commit | 1-2 min | 1-2 min | ~3 GB (repo) |
| **Total** | **15-30 min** | **6-13 min** | **~10 GB** |

**Network**: ~2 GB download (first time)
**CPU**: Moderate (package installation, compression)
**Memory**: ~2-4 GB during composition

## Verifying Treefile Deployment

### Check Treefile Exists

```bash
$ ls -l /etc/rpm-ostree/treefiles/
-rw-r--r--. 1 root root 1234 Jan 31 10:00 kinoite-dev.json
-rw-r--r--. 1 root root 1234 Jan 31 10:00 kinoite-prod.json
```

### Validate Treefile Syntax

```bash
# Check JSON is valid
$ python3 -m json.tool /etc/rpm-ostree/treefiles/kinoite-dev.json > /dev/null
# No output = valid JSON

# Or use jq
$ jq '.' /etc/rpm-ostree/treefiles/kinoite-dev.json > /dev/null
# No output = valid JSON
```

### View Treefile Contents

```bash
$ cat /etc/rpm-ostree/treefiles/kinoite-dev.json
```

### Check Package Count

```bash
$ jq '.packages | length' /etc/rpm-ostree/treefiles/kinoite-dev.json
45
```

### Diff Dev and Prod Treefiles

```bash
$ diff /etc/rpm-ostree/treefiles/kinoite-dev.json \
       /etc/rpm-ostree/treefiles/kinoite-prod.json
2c2
<   "ref": "fedora/x86_64/kinoite/dev",
---
>   "ref": "fedora/x86_64/kinoite/stable",
```

**Only difference**: The `ref` field (branch name)

## Troubleshooting

### Problem: Composition Fails - Package Not Found

**Error**:
```
error: Package 'nonexistent-package' not found
```

**Solution**: Remove or fix package name in treefile

```bash
# Edit treefile
sudo vim /etc/rpm-ostree/treefiles/kinoite-dev.json

# Remove or correct the package name

# Try composition again
sudo rpm-ostree compose tree \
  --repo=/srv/ostree/rpm-ostree/dev \
  /etc/rpm-ostree/treefiles/kinoite-dev.json
```

### Problem: Composition Fails - Repository Unreachable

**Error**:
```
error: Failed to synchronize cache for repo 'fedora'
```

**Solution**: Check network and repository configuration

```bash
# Test repository access
sudo dnf repolist

# Update repository metadata
sudo dnf clean all
sudo dnf makecache

# Retry composition
```

### Problem: Postprocess Script Fails

**Error**:
```
error: Postprocessing script failed with exit code 1
```

**Solution**: Debug postprocess commands

```bash
# Test script manually in a chroot
sudo systemd-nspawn -D /tmp/test-root

# Inside chroot, run postprocess commands manually
ln -sf /usr/lib/systemd/system/graphical.target /etc/systemd/system/default.target

# Fix the failing command in treefile
```

### Problem: Treefile Not Deployed

**Symptom**: Treefile doesn't exist after running playbook

**Diagnosis**:
```bash
$ ls /etc/rpm-ostree/treefiles/
ls: cannot access '/etc/rpm-ostree/treefiles/': No such file or directory
```

**Solution**: Check playbook output for errors

```bash
# Re-run playbook with verbose output
ansible-playbook -i inventory/hosts.yml \
  playbooks/setup-rpm-ostree-repo.yml -v

# Look for failed tasks in compose role
```

## The Role in the Overall Workflow

```
┌────────────────────────────────────────┐
│  1. rpm_ostree_repo                    │
│     Creates empty repositories         │
└─────────────────┬──────────────────────┘
                  │
┌─────────────────▼──────────────────────┐
│  2. rpm_ostree_selinux                 │
│     Sets SELinux contexts              │
└─────────────────┬──────────────────────┘
                  │
┌─────────────────▼──────────────────────┐
│  3. rpm_ostree_apache                  │
│     Configures web serving             │
└─────────────────┬──────────────────────┘
                  │
┌─────────────────▼──────────────────────┐
│  4. rpm_ostree_firewall                │
│     Opens necessary ports              │
└─────────────────┬──────────────────────┘
                  │
┌─────────────────▼──────────────────────┐
│  5. rpm_ostree_compose ← YOU ARE HERE  │
│     Deploys treefiles                  │
│     (Optionally composes images)       │
└─────────────────┬──────────────────────┘
                  │
         ┌────────┴────────┐
         │                 │
    Manual Mode      Automatic Mode
         │                 │
         ▼                 ▼
  Administrator       Playbook
   composes           composes
   manually           automatically
```

## Summary

### What the rpm_ostree_compose Role Does

1. **Creates treefile directory** (`/etc/rpm-ostree/treefiles/`)
2. **Deploys dev treefile** (recipe for dev OS image)
3. **Deploys prod treefile** (recipe for prod OS image)
4. **Optionally composes images** (if automatic composition enabled)
5. **Updates repository summaries** (if images composed)
6. **Displays instructions** (how to manually compose if needed)

### What It Does NOT Do (By Default)

- ❌ Does NOT automatically compose OS images (manual recommended)
- ❌ Does NOT modify existing commits
- ❌ Does NOT download packages (only when composing)
- ❌ Does NOT deploy to clients (that's done separately)

### Key Points

| Aspect | Details |
|--------|---------|
| **Primary Purpose** | Deploy treefile configurations |
| **Secondary Purpose** | Optionally compose OS images |
| **Default Behavior** | Deploy treefiles, show manual instructions |
| **Manual Composition** | Recommended for first-time and testing |
| **Automatic Composition** | Available but disabled by default |
| **Customization** | Via group_vars (packages, postprocess, etc.) |
| **Dev vs Prod** | Separate treefiles with different refs |

### In One Sentence

**The rpm_ostree_compose role deploys treefile configurations (OS image recipes) and optionally composes the actual OS images, with manual composition recommended for better control and testing**.

### Why This Role Matters

```
Without this role:
❌ No treefile → Cannot compose OS images
❌ No OS images → Repository is empty
❌ Empty repository → Clients have nothing to download
❌ Manual treefile creation → Error-prone, inconsistent

With this role:
✅ Treefiles deployed automatically
✅ Consistent configuration across environments
✅ Easy customization via variables
✅ Optional automatic composition
✅ Clear manual composition instructions
✅ Ready for image composition
```

**Bottom Line**: This role transforms your empty repository into a ready-to-use OS image factory by deploying the "recipes" (treefiles) that define what goes into your custom Kinoite images.
