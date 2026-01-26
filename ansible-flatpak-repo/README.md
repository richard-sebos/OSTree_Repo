# Flatpak Repository Ansible Automation

Ansible playbooks and roles to automate the deployment and management of a Flatpak repository server.

## Structure

```
ansible-flatpak-repo/
├── ansible.cfg                 # Ansible configuration
├── playbooks/
│   └── setup-flatpak-repo.yml  # Main playbook
├── inventory/
│   └── hosts                   # Inventory file
├── group_vars/
│   ├── flatpak_repo_servers.yml # Server variables
│   └── flatpak_clients.yml      # Client variables
└── roles/
    ├── dependencies/            # Install required packages
    ├── ssl_certificates/        # Generate SSL certificates
    ├── repository/              # Initialize Flatpak repository
    ├── apache/                  # Configure Apache web server
    ├── selinux/                 # Configure SELinux
    ├── firewall/                # Configure firewall
    └── client_scripts/          # Deploy helper scripts
```

## Quick Start

### 1. Configure Variables

Edit `group_vars/flatpak_repo_servers.yml` to customize your setup:

```yaml
flatpak_repo_enable_https: true  # Enable HTTPS
flatpak_repo_http_port: 8080     # HTTP port
flatpak_repo_https_port: 8443    # HTTPS port
```

### 2. Run the Playbook

**HTTP only (default):**
```bash
cd ansible-flatpak-repo
ansible-playbook playbooks/setup-flatpak-repo.yml
```

**With HTTPS:**
```bash
ansible-playbook playbooks/setup-flatpak-repo.yml -e "flatpak_repo_enable_https=true"
```

**Step-by-step (with tags):**
```bash
# Install dependencies only
ansible-playbook playbooks/setup-flatpak-repo.yml --tags dependencies

# Setup SSL certificates
ansible-playbook playbooks/setup-flatpak-repo.yml --tags ssl

# Configure Apache
ansible-playbook playbooks/setup-flatpak-repo.yml --tags apache

# Configure SELinux
ansible-playbook playbooks/setup-flatpak-repo.yml --tags selinux

# Configure firewall
ansible-playbook playbooks/setup-flatpak-repo.yml --tags firewall
```

### 3. Add Applications

After setup, add applications to the repository:

```bash
sudo flatpak-repo-add org.mozilla.firefox
sudo flatpak-repo-add org.libreoffice.LibreOffice
```

## Configuration Options

### Server Variables (`group_vars/flatpak_repo_servers.yml`)

| Variable | Default | Description |
|----------|---------|-------------|
| `flatpak_repo_path` | `/var/flatpak/repo` | Repository file path |
| `flatpak_repo_name` | `enterprise-apps` | Repository name |
| `flatpak_repo_http_port` | `8080` | HTTP port |
| `flatpak_repo_https_port` | `8443` | HTTPS port |
| `flatpak_repo_enable_https` | `false` | Enable HTTPS |
| `flatpak_repo_server_name` | `flatpak-repo.local` | Server hostname |
| `flatpak_repo_collection_id` | `com.enterprise.Apps` | Collection ID |
| `flatpak_repo_gpg_key_id` | `""` | GPG key for signing |
| `ssl_cert_days_valid` | `3650` | Certificate validity (days) |
| `selinux_permissive_mode` | `false` | SELinux permissive mode |

## Roles

### 1. dependencies
Installs required packages:
- flatpak
- ostree
- httpd (RHEL) / apache2 (Debian)
- policycoreutils-python-utils
- mod_ssl (for HTTPS)

### 2. ssl_certificates
- Generates self-signed SSL certificate (if HTTPS enabled)
- Configures certificate for 10-year validity
- Sets proper file permissions

### 3. repository
- Initializes OSTree repository
- Configures collection ID for P2P
- Optional GPG signing configuration
- Generates repository summary

### 4. apache
- Deploys Apache virtual host configuration
- Creates symlink for clean URLs (/repo)
- Manages Listen directives (no duplicates!)
- Enables required modules
- Tests configuration before applying

### 5. selinux
- Sets httpd_sys_content_t context on repository
- Enables httpd_can_network_connect (for HTTPS)
- Applies contexts with restorecon
- Optional permissive mode for testing

### 6. firewall
- Opens HTTP port (8080)
- Opens HTTPS port (8443 if enabled)
- Uses firewalld for RHEL/Fedora
- Gracefully handles firewall not running

### 7. client_scripts
- Creates flatpak-configure-client script
- Creates flatpak-repo-add helper
- Deploys scripts to /usr/local/bin
- Makes client script web-accessible

## Tags

Run specific parts of the playbook:

```bash
# Run only dependency installation
ansible-playbook playbooks/setup-flatpak-repo.yml --tags dependencies

# Setup and configure security (SELinux + firewall)
ansible-playbook playbooks/setup-flatpak-repo.yml --tags security

# Full setup
ansible-playbook playbooks/setup-flatpak-repo.yml --tags setup

# Skip SELinux configuration
ansible-playbook playbooks/setup-flatpak-repo.yml --skip-tags selinux
```

Available tags:
- `dependencies` - Package installation
- `ssl` - SSL certificate generation
- `https` - HTTPS configuration
- `repository` - Repository initialization
- `apache` - Web server setup
- `web` - Web server setup (alias)
- `selinux` - SELinux configuration
- `security` - SELinux + firewall
- `firewall` - Firewall configuration
- `scripts` - Helper scripts
- `setup` - Full setup (all roles)

## Examples

### Example 1: HTTP Only Setup

```bash
ansible-playbook playbooks/setup-flatpak-repo.yml
```

### Example 2: HTTPS Setup

```bash
ansible-playbook playbooks/setup-flatpak-repo.yml \
    -e "flatpak_repo_enable_https=true"
```

### Example 3: Custom Ports

```bash
ansible-playbook playbooks/setup-flatpak-repo.yml \
    -e "flatpak_repo_http_port=80" \
    -e "flatpak_repo_https_port=443" \
    -e "flatpak_repo_enable_https=true"
```

### Example 4: Custom Repository Path

```bash
ansible-playbook playbooks/setup-flatpak-repo.yml \
    -e "flatpak_repo_path=/srv/flatpak/repo"
```

### Example 5: Dry Run (Check Mode)

```bash
ansible-playbook playbooks/setup-flatpak-repo.yml --check
```

### Example 6: With Verbosity

```bash
ansible-playbook playbooks/setup-flatpak-repo.yml -vv
```

## Idempotency

All roles are idempotent - you can run the playbook multiple times safely:
- Packages won't be reinstalled if already present
- SSL certificates won't be regenerated if they exist
- Repository won't be reinitialized if it exists
- Listen directives are cleaned up before adding (no duplicates!)
- Firewall rules are only added if not present

## Troubleshooting

### Check Apache Configuration
```bash
ansible-playbook playbooks/setup-flatpak-repo.yml --tags apache -vv
```

### Verify SELinux Contexts
```bash
ansible all -m shell -a "ls -laZ /var/flatpak/repo"
```

### Test Repository Access
```bash
curl -I http://localhost:8080/repo/
```

### View Apache Logs
```bash
ansible all -m shell -a "tail -20 /var/log/httpd/flatpak-repo-error.log"
```

## Requirements

### Control Node
- Ansible 2.9+
- Python 3.6+

### Managed Nodes
- RHEL/Rocky/Alma 8+ or Fedora 35+
- Python 3.6+
- sudo access

### Ansible Collections
```bash
ansible-galaxy collection install ansible.posix
ansible-galaxy collection install community.general
```

## License

MIT

## Author

System Administration Team
