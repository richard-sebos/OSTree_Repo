# Role-Based Application Access with Multiple Repositories

## Overview

Different departments or user roles often need access to different sets of applications. This guide explains how to configure multiple Flatpak repositories so Finance users see financial applications while General users see only basic productivity tools.

## Architecture Options

### Option 1: Multiple Repositories (Recommended)

```
┌─────────────────────────────────────────────────────────┐
│  Repository Server (192.168.35.35)                      │
│                                                          │
│  ┌────────────────────────┐  ┌─────────────────────┐   │
│  │ Base Repository        │  │ Finance Repository  │   │
│  │ /var/flatpak/base-repo │  │ /var/flatpak/finance│   │
│  │                        │  │                     │   │
│  │ - Firefox             │  │ - QuickBooks        │   │
│  │ - LibreOffice         │  │ - Excel (via Wine)  │   │
│  │ - Thunderbird         │  │ - SAP GUI           │   │
│  │ - GIMP                │  │ - Tableau           │   │
│  └────────────────────────┘  └─────────────────────┘   │
│           │                            │                │
│           │ Port 8080                  │ Port 8081      │
└───────────┼────────────────────────────┼────────────────┘
            │                            │
            │                            │
    ┌───────┴───────┐           ┌───────┴────────┐
    │               │           │                │
┌───▼────┐    ┌────▼────┐  ┌──▼─────┐    ┌─────▼──┐
│General │    │General  │  │Finance │    │Finance │
│User 1  │    │User 2   │  │User 1  │    │User 2  │
│        │    │         │  │        │    │        │
│Base    │    │Base     │  │Base +  │    │Base +  │
│Only    │    │Only     │  │Finance │    │Finance │
└────────┘    └─────────┘  └────────┘    └────────┘
```

### Option 2: Single Repository with Ansible Group-Based Filtering

```
┌─────────────────────────────────────────────────────────┐
│  Repository Server (192.168.35.35)                      │
│                                                          │
│  ┌──────────────────────────────────────────────────┐  │
│  │ Single Repository: /var/flatpak/repo             │  │
│  │                                                   │  │
│  │ All Applications:                                │  │
│  │ - Firefox, LibreOffice (base)                    │  │
│  │ - QuickBooks, SAP (finance)                      │  │
│  │ - AutoCAD, SolidWorks (engineering)              │  │
│  └──────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────┘
                           │
            ┌──────────────┴──────────────┐
            │                             │
    Ansible configures          Ansible configures
    different remotes          different remotes
            │                             │
    ┌───────▼────────┐           ┌───────▼────────┐
    │ General Users  │           │ Finance Users  │
    │                │           │                │
    │ Can see:       │           │ Can see:       │
    │ - Firefox      │           │ - Firefox      │
    │ - LibreOffice  │           │ - LibreOffice  │
    │                │           │ - QuickBooks   │
    │                │           │ - SAP          │
    └────────────────┘           └────────────────┘
```

## Option 1: Multiple Repositories (Detailed Implementation)

### Step 1: Create Multiple Repository Directories

Create an Ansible playbook to set up multiple repositories:

```yaml
---
# playbooks/setup-multi-repos.yml
# Version: 1.0.0

- name: Setup Multiple Flatpak Repositories
  hosts: flatpak_repo_servers
  become: yes

  vars:
    repositories:
      - name: base-apps
        path: /var/flatpak/base-repo
        port: 8080
        collection_id: com.company.BaseApps
        description: "Base applications for all users"

      - name: finance-apps
        path: /var/flatpak/finance-repo
        port: 8081
        collection_id: com.company.FinanceApps
        description: "Finance department applications"

      - name: engineering-apps
        path: /var/flatpak/engineering-repo
        port: 8082
        collection_id: com.company.EngineeringApps
        description: "Engineering department applications"

  tasks:
    - name: Create repository directories
      ansible.builtin.file:
        path: "{{ item.path }}"
        state: directory
        mode: '0755'
        owner: root
        group: root
      loop: "{{ repositories }}"

    - name: Initialize OSTree repositories
      ansible.builtin.command:
        cmd: ostree init --mode=archive-z2 --repo={{ item.path }}
      args:
        creates: "{{ item.path }}/config"
      loop: "{{ repositories }}"

    - name: Configure collection IDs
      ansible.builtin.command:
        cmd: ostree config set --repo={{ item.path }} core.collection-id {{ item.collection_id }}
      loop: "{{ repositories }}"

    - name: Deploy Apache configs for each repository
      ansible.builtin.template:
        src: multi-repo-vhost.conf.j2
        dest: "/etc/httpd/conf.d/{{ item.name }}.conf"
        mode: '0644'
      loop: "{{ repositories }}"
      notify: Restart httpd

    - name: Open firewall ports
      ansible.posix.firewalld:
        port: "{{ item.port }}/tcp"
        permanent: yes
        immediate: yes
        state: enabled
      loop: "{{ repositories }}"

  handlers:
    - name: Restart httpd
      ansible.builtin.systemd:
        name: httpd
        state: restarted
```

### Step 2: Apache Virtual Host Template

```jinja
{# templates/multi-repo-vhost.conf.j2 #}
# {{ item.description }}
<VirtualHost *:{{ item.port }}>
    ServerName flatpak-repo.local
    DocumentRoot {{ item.path }}

    <Directory {{ item.path }}>
        Options Indexes FollowSymLinks
        AllowOverride None
        Require all granted
        Header set Access-Control-Allow-Origin "*"
    </Directory>

    <Directory {{ item.path }}/objects>
        Options -Indexes
    </Directory>

    ErrorLog /var/log/httpd/{{ item.name }}-error.log
    CustomLog /var/log/httpd/{{ item.name }}-access.log combined
</VirtualHost>
```

### Step 3: Populate Repositories with Role-Specific Apps

```bash
# Base apps (available to everyone)
sudo flatpak install --system flathub org.mozilla.firefox
sudo flatpak build-export /var/flatpak/base-repo /var/lib/flatpak/app/org.mozilla.firefox
sudo flatpak install --system flathub org.libreoffice.LibreOffice
sudo flatpak build-export /var/flatpak/base-repo /var/lib/flatpak/app/org.libreoffice.LibreOffice
sudo flatpak build-update-repo /var/flatpak/base-repo

# Finance apps (only for finance users)
sudo flatpak install --system flathub org.gnome.Calculator
sudo flatpak build-export /var/flatpak/finance-repo /var/lib/flatpak/app/org.gnome.Calculator
# Add other finance-specific apps here
sudo flatpak build-update-repo /var/flatpak/finance-repo

# Engineering apps (only for engineers)
sudo flatpak install --system flathub org.freecadweb.FreeCAD
sudo flatpak build-export /var/flatpak/engineering-repo /var/lib/flatpak/app/org.freecadweb.FreeCAD
sudo flatpak build-update-repo /var/flatpak/engineering-repo
```

### Step 4: Configure Clients Based on Role

Create role-based client configuration playbooks:

```yaml
---
# playbooks/configure-general-users.yml
# Configures workstations for general users

- name: Configure General User Workstations
  hosts: general_workstations
  become: yes

  tasks:
    - name: Configure base-apps repository
      ansible.builtin.command:
        cmd: >
          flatpak remote-add --system --no-gpg-verify
          base-apps http://flatpak-repo.local:8080/
      args:
        creates: /var/lib/flatpak/repo/base-apps.trustedkeys.gpg

    - name: Set repository priority
      ansible.builtin.command:
        cmd: flatpak remote-modify --system --prio=1 base-apps
```

```yaml
---
# playbooks/configure-finance-users.yml
# Configures workstations for finance users

- name: Configure Finance User Workstations
  hosts: finance_workstations
  become: yes

  tasks:
    - name: Configure base-apps repository
      ansible.builtin.command:
        cmd: >
          flatpak remote-add --system --no-gpg-verify
          base-apps http://flatpak-repo.local:8080/
      args:
        creates: /var/lib/flatpak/repo/base-apps.trustedkeys.gpg

    - name: Configure finance-apps repository
      ansible.builtin.command:
        cmd: >
          flatpak remote-add --system --no-gpg-verify
          finance-apps http://flatpak-repo.local:8081/
      args:
        creates: /var/lib/flatpak/repo/finance-apps.trustedkeys.gpg

    - name: Set base-apps priority
      ansible.builtin.command:
        cmd: flatpak remote-modify --system --prio=1 base-apps

    - name: Set finance-apps priority
      ansible.builtin.command:
        cmd: flatpak remote-modify --system --prio=2 finance-apps
```

### Step 5: Inventory Organization

```yaml
# inventory/hosts.yml
---
all:
  children:
    flatpak_repo_servers:
      hosts:
        flatpak:
          ansible_host: 192.168.35.35
          ansible_user: ansible-admin

    workstations:
      children:
        general_workstations:
          hosts:
            ws-reception01:
              ansible_host: 192.168.35.201
              ansible_user: ansible-admin
            ws-admin01:
              ansible_host: 192.168.35.231
              ansible_user: ansible-admin

        finance_workstations:
          hosts:
            ws-finance01:
              ansible_host: 192.168.35.241
              ansible_user: ansible-admin
            ws-finance02:
              ansible_host: 192.168.35.242
              ansible_user: ansible-admin

        engineering_workstations:
          hosts:
            ws-eng01:
              ansible_host: 192.168.35.251
              ansible_user: ansible-admin
```

## Option 2: Single Repository with Access Control

### Approach A: Network-Based Access Control (Apache)

```apache
# /etc/httpd/conf.d/flatpak-repo-acl.conf

<VirtualHost *:8080>
    ServerName flatpak-repo.local
    DocumentRoot /var/flatpak/repo

    # Base applications - available to all
    <Directory /var/flatpak/repo>
        Options Indexes FollowSymLinks
        Require all granted
    </Directory>

    # Finance applications - restricted by IP
    <Directory /var/flatpak/repo/refs/heads/app/com.finance.*>
        Require ip 192.168.35.240/28  # Finance subnet only
    </Directory>

    # Engineering applications - restricted by IP
    <Directory /var/flatpak/repo/refs/heads/app/com.engineering.*>
        Require ip 192.168.35.250/28  # Engineering subnet only
    </Directory>
</VirtualHost>
```

### Approach B: Application Naming Convention

Use consistent naming to organize applications:

```
Repository Structure:
/var/flatpak/repo/refs/heads/app/
├── com.company.base.Firefox
├── com.company.base.LibreOffice
├── com.company.finance.QuickBooks
├── com.company.finance.SAP
├── com.company.engineering.AutoCAD
└── com.company.engineering.FreeCAD
```

Then configure clients to only see specific prefixes:

```bash
# General users
flatpak remote-add --system --filter=com.company.base.* base-apps http://flatpak-repo.local:8080/

# Finance users
flatpak remote-add --system --filter=com.company.base.* base-apps http://flatpak-repo.local:8080/
flatpak remote-add --system --filter=com.company.finance.* finance-apps http://flatpak-repo.local:8080/
```

## Complete Example: Finance vs General Users

### Server Setup

```bash
# Run multi-repository setup
ansible-playbook playbooks/setup-multi-repos.yml

# Populate base repository
cat << 'EOF' > base-apps.txt
org.mozilla.firefox
org.libreoffice.LibreOffice
org.mozilla.Thunderbird
org.gnome.Calendar
EOF

# Populate finance repository
cat << 'EOF' > finance-apps.txt
org.gnome.Calculator
# Add QuickBooks, SAP, etc.
EOF

# Add apps to repositories
for app in $(cat base-apps.txt); do
    flatpak install --system flathub $app
    flatpak build-export /var/flatpak/base-repo /var/lib/flatpak/app/$app
done
flatpak build-update-repo /var/flatpak/base-repo

for app in $(cat finance-apps.txt); do
    flatpak install --system flathub $app
    flatpak build-export /var/flatpak/finance-repo /var/lib/flatpak/app/$app
done
flatpak build-update-repo /var/flatpak/finance-repo
```

### Client Configuration

```bash
# Deploy to general workstations
ansible-playbook playbooks/configure-general-users.yml

# Deploy to finance workstations
ansible-playbook playbooks/configure-finance-users.yml

# Deploy to engineering workstations
ansible-playbook playbooks/configure-engineering-users.yml
```

### User Experience

**General User:**
```bash
[user@ws-reception01 ~]$ flatpak remote-ls
base-apps  org.mozilla.firefox
base-apps  org.libreoffice.LibreOffice
base-apps  org.mozilla.Thunderbird
base-apps  org.gnome.Calendar

[user@ws-reception01 ~]$ flatpak install base-apps org.gnome.Calculator
error: Remote 'base-apps' doesn't have ref 'app/org.gnome.Calculator/x86_64/stable'
```

**Finance User:**
```bash
[user@ws-finance01 ~]$ flatpak remote-ls
base-apps     org.mozilla.firefox
base-apps     org.libreoffice.LibreOffice
finance-apps  org.gnome.Calculator
finance-apps  com.company.QuickBooks
finance-apps  com.sap.gui

[user@ws-finance01 ~]$ flatpak install finance-apps org.gnome.Calculator
Installing: org.gnome.Calculator
✓ Success!
```

## Advanced: Dynamic Role Assignment

### Using LDAP/Active Directory Groups

```yaml
---
# playbooks/configure-workstation-dynamic.yml
# Automatically configures repos based on AD group membership

- name: Configure Workstation Based on User Role
  hosts: workstations
  become: yes

  tasks:
    - name: Get user's AD groups
      ansible.builtin.command:
        cmd: "id -Gn {{ ansible_user }}"
      register: user_groups
      changed_when: false

    - name: Configure base repository (all users)
      ansible.builtin.command:
        cmd: >
          flatpak remote-add --system --no-gpg-verify
          base-apps http://flatpak-repo.local:8080/
      args:
        creates: /var/lib/flatpak/repo/base-apps.trustedkeys.gpg

    - name: Configure finance repository (if in finance group)
      ansible.builtin.command:
        cmd: >
          flatpak remote-add --system --no-gpg-verify
          finance-apps http://flatpak-repo.local:8081/
      args:
        creates: /var/lib/flatpak/repo/finance-apps.trustedkeys.gpg
      when: "'finance' in user_groups.stdout"

    - name: Configure engineering repository (if in engineering group)
      ansible.builtin.command:
        cmd: >
          flatpak remote-add --system --no-gpg-verify
          engineering-apps http://flatpak-repo.local:8082/
      args:
        creates: /var/lib/flatpak/repo/engineering-apps.trustedkeys.gpg
      when: "'engineering' in user_groups.stdout"
```

## Maintenance

### Adding Apps to Specific Repositories

```bash
# Add to base repository
sudo flatpak install --system flathub org.gimp.GIMP
sudo flatpak build-export /var/flatpak/base-repo /var/lib/flatpak/app/org.gimp.GIMP
sudo flatpak build-update-repo /var/flatpak/base-repo

# Add to finance repository
sudo flatpak install --system flathub com.microsoft.Edge
sudo flatpak build-export /var/flatpak/finance-repo /var/lib/flatpak/app/com.microsoft.Edge
sudo flatpak build-update-repo /var/flatpak/finance-repo
```

### Monitoring Access by Role

```bash
# View who's accessing which repository
tail -f /var/log/httpd/base-apps-access.log
tail -f /var/log/httpd/finance-apps-access.log
tail -f /var/log/httpd/engineering-apps-access.log

# Generate usage report
grep "GET.*summary" /var/log/httpd/*-access.log | \
    awk '{print $1, $7}' | \
    sort | uniq -c
```

## Security Best Practices

1. **Network Segmentation**
   ```
   Base Apps:        0.0.0.0:8080 (all networks)
   Finance Apps:     192.168.35.240/28 (finance VLAN only)
   Engineering Apps: 192.168.35.250/28 (engineering VLAN only)
   ```

2. **Firewall Rules**
   ```bash
   # Restrict finance port to finance subnet
   firewall-cmd --permanent --zone=finance --add-port=8081/tcp
   firewall-cmd --permanent --zone=finance --add-source=192.168.35.240/28
   ```

3. **Audit Logging**
   ```bash
   # Log all repository access
   CustomLog /var/log/httpd/flatpak-audit.log "%h %t \"%r\" %>s %b \"%{Referer}i\""
   ```

4. **Regular Reviews**
   - Quarterly review of who has access to which repositories
   - Audit logs for unauthorized access attempts
   - Remove access for departed employees

## Comparison: Multiple Repos vs Single Repo

| Feature | Multiple Repos | Single Repo + Filters |
|---------|---------------|----------------------|
| **Separation** | Complete isolation | Logical separation only |
| **Complexity** | Higher (multiple configs) | Lower (one config) |
| **Access Control** | Port/IP based | App-level filtering |
| **Bandwidth** | Higher (duplicate base apps) | Lower (shared base) |
| **Management** | More complex | Simpler |
| **Security** | Better (network level) | Good (app level) |
| **Scalability** | Moderate | Better |
| **Recommendation** | High-security environments | Most enterprises |

## Recommended Approach

For most enterprises: **Multiple Repositories (Option 1)**

**Reasons:**
1. ✅ Clear separation of concerns
2. ✅ Better security (network-level isolation)
3. ✅ Easier compliance auditing
4. ✅ Simpler troubleshooting
5. ✅ Different update schedules per department

**Trade-off:** Slightly more complex to manage, but Ansible automation handles this well.

---

**Next Steps:**
1. Identify your organizational roles/departments
2. Define which apps each role needs
3. Deploy multi-repository setup with Ansible
4. Configure clients based on role
5. Monitor and audit access

**Version:** 1.0.0
**Last Updated:** 2026-01-24
