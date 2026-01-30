# rpm-ostree Repository Setup with Ansible

## Overview

Built an Ansible automation solution to set up an rpm-ostree repository server for hosting custom Fedora Kinoite images, extending existing Flatpak infrastructure with dev/prod environment separation.

## Infrastructure Details

### Server Configuration

- **Platform**: Extends existing Apache/Flatpak OSTree server
- **Protocol**: HTTPS on port 8443 (shared with Flatpak)
- **SSL**: Reuses existing Flatpak SSL certificates
- **SELinux**: Enforcing mode with proper contexts
- **Firewall**: firewalld with ports 8080/8443 open

### Repository Architecture

- **Base Path**: `/srv/ostree/rpm-ostree/`
- **Dev Repository**: `/srv/ostree/rpm-ostree/dev`
- **Prod Repository**: `/srv/ostree/rpm-ostree/prod`
- **Repository Mode**: archive-z2 (optimized for HTTP serving)
- **Dev URL**: `https://SERVER:8443/repo/kinoite/dev`
- **Prod URL**: `https://SERVER:8443/repo/kinoite/prod`

## Project Structure

```
ansible-rpm-ostree-repo/
├── inventory/
│   └── hosts.yml                    # Inventory file
├── group_vars/
│   └── rpm_ostree_repo_servers.yml  # Central configuration
├── playbooks/
│   ├── setup-rpm-ostree-repo.yml    # Main setup playbook
│   └── promote-to-prod.yml          # Promotion workflow
└── roles/
    ├── rpm_ostree_repo/             # Repository initialization
    ├── rpm_ostree_selinux/          # SELinux configuration
    ├── rpm_ostree_apache/           # Apache/HTTPS setup
    ├── rpm_ostree_firewall/         # Firewall configuration
    └── rpm_ostree_compose/          # Image composition
```

## Key Components

### 1. Repository Initialization Role (`rpm_ostree_repo`)

**Purpose**: Creates and initializes separate dev and prod OSTree repositories

**Key Files**:
- `roles/rpm_ostree_repo/tasks/main.yml`
- `roles/rpm_ostree_repo/defaults/main.yml`

**What It Does**:

```yaml
# Creates directory structure
/srv/ostree/rpm-ostree/
├── dev/
│   ├── config
│   ├── objects/
│   ├── refs/
│   └── tmp/
└── prod/
    ├── config
    ├── objects/
    ├── refs/
    └── tmp/
```

**Key Tasks**:

1. Creates base directory `/srv/ostree/rpm-ostree`
2. Creates separate dev and prod subdirectories
3. Checks if repositories already exist (idempotent)
4. Initializes OSTree repos with `ostree init --mode=archive-z2`
5. Sets proper permissions (0755, root:root)
6. Recurses through all subdirectories

**Default Variables**:

```yaml
rpm_ostree_repo_base_path: /srv/ostree/rpm-ostree
rpm_ostree_repo_mode: archive-z2
rpm_ostree_repo_dev_path: "{{ rpm_ostree_repo_base_path }}/dev"
rpm_ostree_repo_prod_path: "{{ rpm_ostree_repo_base_path }}/prod"
```

### 2. SELinux Configuration Role (`rpm_ostree_selinux`)

**Purpose**: Configures SELinux contexts for Apache to serve OSTree content

**Key Files**:
- `roles/rpm_ostree_selinux/tasks/main.yml`
- `roles/rpm_ostree_selinux/defaults/main.yml`

**What It Does**:

1. Checks if SELinux is enabled/enforcing
2. Sets file context policy: `httpd_sys_content_t`
3. Applies context with `restorecon -Rv`
4. Verifies context is applied correctly

**SELinux Context Applied**:

```bash
# Before
unconfined_u:object_r:default_t:s0 /srv/ostree/rpm-ostree/dev

# After
unconfined_u:object_r:httpd_sys_content_t:s0 /srv/ostree/rpm-ostree/dev
```

**Why This Matters**: Without the correct SELinux context, Apache would get "Permission Denied" errors even with correct file permissions.

**Default Variables**:

```yaml
selinux_permissive_mode: false
rpm_ostree_repo_base_path: /srv/ostree/rpm-ostree
```

### 3. Apache Configuration Role (`rpm_ostree_apache`)

**Purpose**: Extends existing Apache configuration to serve rpm-ostree repositories

**Key Files**:
- `roles/rpm_ostree_apache/tasks/main.yml`
- `roles/rpm_ostree_apache/templates/rpm-ostree-repo-https.conf.j2`
- `roles/rpm_ostree_apache/defaults/main.yml`

**Apache Configuration Template** (`rpm-ostree-repo-https.conf.j2`):

```apache
<VirtualHost *:{{ flatpak_repo_https_port | default('8443') }}>
    ServerName {{ ansible_fqdn }}

    SSLEngine on
    SSLCertificateFile {{ ssl_cert_file }}
    SSLCertificateKeyFile {{ ssl_key_file }}

    # DEV Repository
    Alias {{ rpm_ostree_http_alias_dev }} {{ rpm_ostree_repo_dev_path }}
    <Directory {{ rpm_ostree_repo_dev_path }}>
        Options Indexes FollowSymLinks
        AllowOverride None
        Require all granted
        Header set Access-Control-Allow-Origin "*"
    </Directory>

    # PROD Repository
    Alias {{ rpm_ostree_http_alias_prod }} {{ rpm_ostree_repo_prod_path }}
    <Directory {{ rpm_ostree_repo_prod_path }}>
        Options Indexes FollowSymLinks
        AllowOverride None
        Require all granted
        Header set Access-Control-Allow-Origin "*"
    </Directory>
</VirtualHost>
```

**Key Features**:
- Reuses existing SSL certificates from Flatpak setup
- Serves both dev and prod on same port with different URL paths
- Enables directory indexing for browsing
- Sets CORS headers for cross-origin access
- Automatic Apache reload via handler

**Default Variables**:

```yaml
rpm_ostree_enable_https: true
rpm_ostree_repo_base_path: /srv/ostree/rpm-ostree
rpm_ostree_repo_dev_path: "{{ rpm_ostree_repo_base_path }}/dev"
rpm_ostree_repo_prod_path: "{{ rpm_ostree_repo_base_path }}/prod"
rpm_ostree_http_alias_dev: /repo/kinoite/dev
rpm_ostree_http_alias_prod: /repo/kinoite/prod
ssl_cert_dir: /etc/pki/tls/certs
ssl_key_dir: /etc/pki/tls/private
ssl_cert_file: "{{ ssl_cert_dir }}/flatpak-repo.crt"
ssl_key_file: "{{ ssl_key_dir }}/flatpak-repo.key"
apache_service_name: "{{ 'httpd' if ansible_os_family == 'RedHat' else 'apache2' }}"
apache_config_dir: "{{ '/etc/httpd/conf.d' if ansible_os_family == 'RedHat' else '/etc/apache2/sites-available' }}"
```

### 4. Firewall Configuration Role (`rpm_ostree_firewall`)

**Purpose**: Ensures firewall ports are open for repository access

**Key Files**:
- `roles/rpm_ostree_firewall/tasks/main.yml`

**What It Does**:

1. Verifies firewalld is active
2. Checks if ports 8080 (HTTP) and 8443 (HTTPS) are open
3. Opens ports if not already configured
4. Makes changes permanent

**Note**: In this setup, the Flatpak repo already had the ports open, so this role just verifies them.

### 5. Compose Role (`rpm_ostree_compose`)

**Purpose**: Manages treefile deployment and optional image composition

**Key Files**:
- `roles/rpm_ostree_compose/tasks/main.yml`
- `roles/rpm_ostree_compose/templates/kinoite.json.j2`
- `roles/rpm_ostree_compose/defaults/main.yml`

**Treefile Structure** (`kinoite.json.j2`):

```json
{
  "ref": "{{ rpm_ostree_ref }}",
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
    "firewalld"
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

**What It Does**:

1. Creates treefile directory `/etc/rpm-ostree/treefiles/`
2. Deploys separate treefiles for dev and prod
   - `kinoite-dev.json` (ref: `fedora/x86_64/kinoite/dev`)
   - `kinoite-prod.json` (ref: `fedora/x86_64/kinoite/stable`)
3. Optionally composes images (disabled by default)
4. Updates repository summaries after composition
5. Displays manual composition instructions

**Default Variables**:

```yaml
rpm_ostree_repo_base_path: /srv/ostree/rpm-ostree
rpm_ostree_repo_dev_path: "{{ rpm_ostree_repo_base_path }}/dev"
rpm_ostree_repo_prod_path: "{{ rpm_ostree_repo_base_path }}/prod"
rpm_ostree_treefile_name: kinoite.json
rpm_ostree_treefile_path: /etc/rpm-ostree/treefiles
rpm_ostree_compose_on_setup: false
rpm_ostree_ref: fedora/x86_64/kinoite/stable
rpm_ostree_ref_dev: fedora/x86_64/kinoite/dev
rpm_ostree_ref_prod: fedora/x86_64/kinoite/stable
rpm_ostree_compose_environments: []
rpm_ostree_fedora_release: 41
```

## Configuration Files

### Central Configuration (`group_vars/rpm_ostree_repo_servers.yml`)

```yaml
---
# rpm-ostree Repository Configuration
rpm_ostree_repo_base_path: /srv/ostree/rpm-ostree
rpm_ostree_repo_mode: archive-z2

# Dev and prod paths
rpm_ostree_repo_dev_path: "{{ rpm_ostree_repo_base_path }}/dev"
rpm_ostree_repo_prod_path: "{{ rpm_ostree_repo_base_path }}/prod"

# HTTP aliases
rpm_ostree_http_alias_dev: /repo/kinoite/dev
rpm_ostree_http_alias_prod: /repo/kinoite/prod

# HTTPS configuration
rpm_ostree_enable_https: true
flatpak_repo_https_port: 8443
flatpak_repo_http_port: 8080

# SSL certificates (reused from Flatpak)
ssl_cert_dir: /etc/pki/tls/certs
ssl_key_dir: /etc/pki/tls/private
ssl_cert_file: "{{ ssl_cert_dir }}/flatpak-repo.crt"
ssl_key_file: "{{ ssl_key_dir }}/flatpak-repo.key"

# Composition settings (optional)
rpm_ostree_compose_on_setup: false
rpm_ostree_compose_environments: []  # ['dev', 'prod'] to enable
```

### Inventory File (`inventory/hosts.yml`)

```yaml
---
all:
  children:
    rpm_ostree_repo_servers:
      hosts:
        your-server-hostname:
          ansible_host: 192.168.35.35
          ansible_user: root
```

## Main Playbook (`playbooks/setup-rpm-ostree-repo.yml`)

**Structure**:

```yaml
---
- name: Setup rpm-ostree Repository Server
  hosts: rpm_ostree_repo_servers
  become: yes

  pre_tasks:
    - name: Display setup information
      # Shows what the playbook will do

    - name: Install required packages
      # Installs ostree and rpm-ostree

  roles:
    - role: rpm_ostree_repo
    - role: rpm_ostree_selinux
    - role: rpm_ostree_apache
    - role: rpm_ostree_firewall
    - role: rpm_ostree_compose

  post_tasks:
    - name: Display setup completion message
      # Shows repository URLs and next steps
```

**Automatic Package Installation**:

The playbook automatically installs:
- `ostree` - Core OSTree functionality
- `rpm-ostree` - RPM package layering for OSTree

## Promotion Workflow (`playbooks/promote-to-prod.yml`)

**Purpose**: Safely promotes tested images from dev to production

```yaml
---
- name: Promote rpm-ostree Image from Dev to Prod
  hosts: rpm_ostree_repo_servers
  become: yes

  tasks:
    - name: Verify dev repository has commits
      # Ensures dev has content to promote

    - name: Pull commits from dev to prod
      ansible.builtin.command:
        cmd: >
          ostree pull-local
          {{ rpm_ostree_repo_dev_path }}
          {{ rpm_ostree_ref_dev }}
          --repo={{ rpm_ostree_repo_prod_path }}

    - name: Create stable ref in prod
      # Creates fedora/x86_64/kinoite/stable ref

    - name: Update prod repository summary
      # Makes new content visible to clients
```

**Usage**:

```bash
ansible-playbook -i inventory/hosts.yml playbooks/promote-to-prod.yml
```

## Key Technical Decisions

### 1. Variable Defaults Strategy

**Problem**: Initial implementation had undefined variable errors when roles were used in different contexts.

**Solution**: Added comprehensive defaults to every role's `defaults/main.yml` file:
- Each role is self-contained with sensible defaults
- Variables can be overridden in group_vars
- Roles work independently or together

**Example Pattern**:

```yaml
# In role defaults
rpm_ostree_repo_base_path: /srv/ostree/rpm-ostree
rpm_ostree_repo_dev_path: "{{ rpm_ostree_repo_base_path }}/dev"

# Can be overridden in group_vars if needed
```

### 2. Idempotency

**All roles are idempotent** - running the playbook multiple times produces the same result:
- Checks if repositories already exist before initializing
- Uses `stat` module to verify file/directory existence
- Apache configuration uses template deployment (replaces if changed)
- Firewall rules check before adding

### 3. Dev/Prod Separation

**Rationale**: Security best practice to test updates before production deployment

**Implementation**:
- Physically separate repositories on disk
- Separate URL endpoints
- Different OSTree refs (dev vs stable)
- Promotion workflow to migrate between environments

### 4. Apache Port Sharing

**Decision**: Use same port 8443 for both Flatpak and rpm-ostree repositories

**Benefits**:
- Single SSL certificate management
- Single firewall port
- Simplified client configuration
- Different URL paths distinguish services

**Configuration**:
- Flatpak: `https://SERVER:8443/repo/flatpak/`
- Kinoite Dev: `https://SERVER:8443/repo/kinoite/dev/`
- Kinoite Prod: `https://SERVER:8443/repo/kinoite/prod/`

### 5. SSL Certificate Reuse

**Decision**: Reuse existing Flatpak SSL certificates

**Benefits**:
- No additional certificate management
- Consistent trust model
- Simplified deployment

**Files Used**:
- Certificate: `/etc/pki/tls/certs/flatpak-repo.crt`
- Key: `/etc/pki/tls/private/flatpak-repo.key`

### 6. Archive-z2 Mode

**Decision**: Use `archive-z2` repository mode

**Rationale**:
- Optimized for HTTP serving
- Better compression than `bare` mode
- Industry standard for OSTree HTTP repositories
- Used by Fedora's official infrastructure

## Troubleshooting Journey

### Issue 1: Apache Service Detection

**Error**:
```
FAILED! => {"assertion": "apache_service_name in services"}
```

**Root Cause**: Systemd service names include `.service` suffix in some outputs

**Fix**: Updated assertion to check both formats:

```yaml
that:
  - "(apache_service_name ~ '.service') in services or apache_service_name in services"
```

### Issue 2: Port Mismatch (8453 vs 8443)

**Error**: Connection refused on port 8453

**Root Cause**: Initial configuration used 8453, but Apache was already listening on 8443 from Flatpak setup

**Fix**: Standardized on port 8443 across all configurations

**Verification**:

```bash
curl -k https://192.168.35.35:8443/repo/kinoite/dev/
```

### Issue 3: Firewall Blocking Connections

**Error**: `No route to host`

**Root Cause**: Firewall configuration not automated initially

**Fix**: Created `rpm_ostree_firewall` role to verify/open ports

### Issue 4: Undefined Variables

**Error**: Multiple `'variable_name' is undefined` errors

**Root Cause**: Complex conditional logic required variables that didn't exist in all contexts

**Fix Applied**:
1. Simplified conditional logic
2. Added comprehensive defaults to all roles
3. Used Jinja2 `| default()` filter for fallback values

**Pattern Used**:

```yaml
# In tasks
path: "{{ rpm_ostree_repo_dev_path | default(rpm_ostree_repo_base_path ~ '/dev') }}"

# In defaults
rpm_ostree_repo_dev_path: "{{ rpm_ostree_repo_base_path }}/dev"
```

### Issue 5: SELinux Variable Name Change

**Error**: `'rpm_ostree_repo_path' is undefined` in SELinux role

**Root Cause**: Variable renamed from single path to base path + dev/prod paths

**Fix**: Updated SELinux role to use `rpm_ostree_repo_base_path`

### Issue 6: Missing Compose Variables

**Error**: `'rpm_ostree_ref_dev' is undefined`

**Root Cause**: Compose role expected dev/prod specific ref variables

**Fix**: Added to compose role defaults:

```yaml
rpm_ostree_ref_dev: fedora/x86_64/kinoite/dev
rpm_ostree_ref_prod: fedora/x86_64/kinoite/stable
rpm_ostree_compose_environments: []
```

### Issue 7: Playbook Completion Message

**Error**: `'rpm_ostree_repo_path' is undefined` in final playbook message

**Root Cause**: Completion message used old single-repo variable names

**Fix**: Updated to display both dev and prod information:

```yaml
msg:
  - "DEV Repository URL: https://{{ ansible_default_ipv4.address }}:8443/repo/kinoite/dev"
  - "PROD Repository URL: https://{{ ansible_default_ipv4.address }}:8443/repo/kinoite/prod"
```

## Usage

### Initial Setup

1. **Clone/Copy the Ansible structure**:

```bash
cd ~/github/OSTree_Repo
git pull
cd /opt/OSTree-repo/dev
cp -R ~/github/OSTree_Repo/ansible-rpm-ostree-repo/* /opt/OSTree-repo/dev
```

2. **Configure inventory**:

Edit `inventory/hosts.yml` with your server details

3. **Customize configuration** (optional):

Edit `group_vars/rpm_ostree_repo_servers.yml`

4. **Run the playbook**:

```bash
ansible-playbook -i inventory/hosts.yml playbooks/setup-rpm-ostree-repo.yml
```

5. **Verify installation**:

```bash
curl -k https://192.168.35.35:8443/repo/kinoite/dev/
curl -k https://192.168.35.35:8443/repo/kinoite/prod/
```

### Composing Images

**Manual Composition** (recommended for first time):

```bash
# Compose dev image
sudo rpm-ostree compose tree \
  --repo=/srv/ostree/rpm-ostree/dev \
  /etc/rpm-ostree/treefiles/kinoite-dev.json

# Update repository summary
sudo ostree summary -u --repo=/srv/ostree/rpm-ostree/dev

# Verify the image
ostree refs --repo=/srv/ostree/rpm-ostree/dev
```

**Automatic Composition** (set in group_vars):

```yaml
rpm_ostree_compose_on_setup: true
rpm_ostree_compose_environments: ['dev']  # or ['dev', 'prod']
```

### Client Configuration

**On Fedora Kinoite/Silverblue clients**:

```bash
# Add dev remote
sudo ostree remote add --no-gpg-verify kinoite-dev \
  https://192.168.35.35:8443/repo/kinoite/dev

# Rebase to custom dev image
sudo rpm-ostree rebase kinoite-dev:fedora/x86_64/kinoite/dev

# Reboot to apply
systemctl reboot
```

**For production**:

```bash
# Add prod remote
sudo ostree remote add --no-gpg-verify kinoite-prod \
  https://192.168.35.35:8443/repo/kinoite/prod

# Rebase to production image
sudo rpm-ostree rebase kinoite-prod:fedora/x86_64/kinoite/stable

systemctl reboot
```

### Promoting Dev to Prod

After testing in dev:

```bash
ansible-playbook -i inventory/hosts.yml playbooks/promote-to-prod.yml
```

This safely copies the tested dev image to production.

## Verification Steps

### 1. Repository Structure

```bash
ssh root@192.168.35.35
tree -L 2 /srv/ostree/rpm-ostree/
```

Expected output:

```
/srv/ostree/rpm-ostree/
├── dev
│   ├── config
│   ├── extensions
│   ├── objects
│   ├── refs
│   ├── state
│   └── tmp
└── prod
    ├── config
    ├── extensions
    ├── objects
    ├── refs
    ├── state
    └── tmp
```

### 2. SELinux Contexts

```bash
ls -Z /srv/ostree/rpm-ostree/
```

Expected output:

```
unconfined_u:object_r:httpd_sys_content_t:s0 dev
unconfined_u:object_r:httpd_sys_content_t:s0 prod
```

### 3. Apache Configuration

```bash
cat /etc/httpd/conf.d/rpm-ostree-repo-https.conf
apachectl -t
systemctl status httpd
```

### 4. Firewall Rules

```bash
firewall-cmd --list-ports
# Should show: 8080/tcp 8443/tcp
```

### 5. Treefiles

```bash
ls -l /etc/rpm-ostree/treefiles/
cat /etc/rpm-ostree/treefiles/kinoite-dev.json
```

### 6. HTTPS Access

```bash
curl -k https://192.168.35.35:8443/repo/kinoite/dev/
curl -k https://192.168.35.35:8443/repo/kinoite/dev/config
```

## Security Considerations

### 1. GPG Signing

**Current State**: No GPG signing (using `--no-gpg-verify`)

**Production Recommendation**:
- Generate GPG key for signing commits
- Configure rpm-ostree to sign during composition
- Distribute public key to clients
- Remove `--no-gpg-verify` flag

### 2. Certificate Trust

**Current State**: Self-signed certificates (using `-k` with curl)

**Production Recommendation**:
- Use certificates from trusted CA
- Distribute CA certificate to clients
- Or use Let's Encrypt for public-facing servers

### 3. Access Control

**Current State**: `Require all granted` in Apache config

**Production Recommendation**:
- Implement IP-based restrictions if needed
- Consider authentication for dev environment
- Keep prod open for client access

### 4. SELinux

**Current State**: Enforcing with proper contexts

**Status**: ✅ Production ready
- Proper `httpd_sys_content_t` context applied
- No permissive mode required

## Performance Considerations

### Repository Size

- OSTree uses content-addressable storage
- Deduplication across commits
- Dev and prod share base path but are separate repos
- Initial Kinoite image: ~2-3 GB
- Incremental updates: typically <500 MB

### Network Requirements

- HTTPS on port 8443
- Clients pull only changed objects
- Delta updates minimize bandwidth
- Repository summary is small (~few KB)

### Composition Time

- First compose: 15-30 minutes (downloads packages)
- Subsequent composes: 5-15 minutes (cached packages)
- Depends on network speed and package count

## Best Practices

### 1. Workflow Pattern

```
[Compose in DEV] → [Test] → [Promote to PROD] → [Deploy to Clients]
```

### 2. Testing in Dev

- Always compose to dev first
- Test on non-critical systems
- Verify functionality before promotion
- Document any issues discovered

### 3. Version Tracking

- Use meaningful commit messages
- Tag important releases
- Keep a changelog
- Document custom packages added

### 4. Backup Strategy

- Back up `/srv/ostree/rpm-ostree/`
- Back up treefiles in `/etc/rpm-ostree/treefiles/`
- Back up Apache configs
- Consider repository mirroring

### 5. Update Cadence

- **Dev**: Weekly or as-needed for testing
- **Prod**: Monthly or after dev validation
- Security updates: As soon as tested
- Major updates: After thorough dev testing

## Future Enhancements

### Potential Additions

1. **Automated composition scheduling** (cron/systemd timers)
2. **GPG signing integration**
3. **Multiple environments** (dev, staging, prod)
4. **Repository monitoring/metrics**
5. **Automated testing in dev**
6. **Rollback capabilities**
7. **Integration with CI/CD pipelines**
8. **Client update notifications**

## Summary

This Ansible-based solution provides:

- ✅ Automated rpm-ostree repository setup
- ✅ Dev/prod environment separation
- ✅ HTTPS support with existing infrastructure
- ✅ Proper SELinux configuration
- ✅ Idempotent, repeatable deployment
- ✅ Safe promotion workflow
- ✅ Comprehensive error handling
- ✅ Full integration with existing Flatpak server

The system is production-ready for internal use and provides a solid foundation for managing custom Fedora Kinoite images across your infrastructure.
