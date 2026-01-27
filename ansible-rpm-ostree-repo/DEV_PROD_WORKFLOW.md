# Dev/Prod Workflow Guide

Complete guide for managing rpm-ostree images across development and production environments.

## Overview

The ansible-rpm-ostree-repo now supports **dual environments**:

- **DEV** (`/repo/kinoite/dev`) - For testing new images and patches
- **PROD** (`/repo/kinoite/prod`) - For stable, production-ready images

## Architecture

```
Server Infrastructure:
├── /srv/ostree/rpm-ostree/
│   ├── dev/                          # Development repository
│   │   └── refs: fedora/x86_64/kinoite/dev
│   └── prod/                         # Production repository
│       └── refs: fedora/x86_64/kinoite/stable
│
├── Apache Endpoints:
│   ├── https://server:8443/repo/kinoite/dev   # Dev repo
│   └── https://server:8443/repo/kinoite/prod  # Prod repo
│
└── Treefiles:
    ├── /etc/rpm-ostree/treefiles/kinoite-dev.json
    └── /etc/rpm-ostree/treefiles/kinoite-prod.json
```

## Workflow

### 1. Initial Setup

Run the playbook to create both repositories:

```bash
ansible-playbook -i inventory/hosts.yml playbooks/setup-rpm-ostree-repo.yml
```

This creates:
- ✅ Both dev and prod OSTree repositories
- ✅ Apache endpoints for both environments
- ✅ Separate treefiles for dev and prod
- ✅ SELinux contexts for both directories
- ✅ Firewall rules (shared ports)

### 2. Compose Dev Image

Test new patches in dev first:

```bash
# On the server
sudo rpm-ostree compose tree \
  --repo=/srv/ostree/rpm-ostree/dev \
  /etc/rpm-ostree/treefiles/kinoite-dev.json

sudo ostree summary -u --repo=/srv/ostree/rpm-ostree/dev
```

Or enable automatic dev composition in `group_vars/rpm_ostree_repo_servers.yml`:

```yaml
rpm_ostree_compose_on_setup: true
rpm_ostree_compose_environments:
  - dev
```

### 3. Test on Dev Clients

Configure test workstations to use dev:

```bash
# On Kinoite test workstation
sudo ostree remote add --no-gpg-verify kinoite-dev \
  https://192.168.35.35:8443/repo/kinoite/dev

sudo rpm-ostree rebase kinoite-dev:fedora/x86_64/kinoite/dev

sudo systemctl reboot
```

### 4. Validate Changes

Test the dev image thoroughly:
- ✅ All applications work
- ✅ System stability
- ✅ Performance testing
- ✅ Security validation

### 5. Promote to Production

Once dev is validated, promote to prod:

```bash
ansible-playbook -i inventory/hosts.yml playbooks/promote-to-prod.yml
```

This playbook:
1. Verifies dev and prod repositories exist
2. Gets the latest commit from dev
3. Pulls the commit into prod repository
4. Creates the prod ref pointing to the commit
5. Updates prod repository summary
6. Displays promotion confirmation

### 6. Deploy to Production Clients

Update production workstations:

```bash
# On production Kinoite workstations
sudo ostree remote add --no-gpg-verify kinoite-prod \
  https://192.168.35.35:8443/repo/kinoite/prod

sudo rpm-ostree rebase kinoite-prod:fedora/x86_64/kinoite/stable

sudo systemctl reboot
```

## Configuration

### Environment-Specific Settings

In `group_vars/rpm_ostree_repo_servers.yml`:

```yaml
# Enable/disable environments
rpm_ostree_environments:
  - name: dev
    enabled: true
  - name: prod
    enabled: true

# Repository paths
rpm_ostree_repo_dev_path: /srv/ostree/rpm-ostree/dev
rpm_ostree_repo_prod_path: /srv/ostree/rpm-ostree/prod

# Apache aliases
rpm_ostree_http_alias_dev: /repo/kinoite/dev
rpm_ostree_http_alias_prod: /repo/kinoite/prod

# OSTree refs
rpm_ostree_ref_dev: fedora/x86_64/kinoite/dev
rpm_ostree_ref_prod: fedora/x86_64/kinoite/stable

# Automatic composition
rpm_ostree_compose_on_setup: false
rpm_ostree_compose_environments:
  - dev
# - prod  # Only compose prod manually or via promotion
```

### Different Package Sets

You can customize packages per environment by overriding in the treefile.

**Option 1: Edit treefiles on server**

```bash
sudo vim /etc/rpm-ostree/treefiles/kinoite-dev.json
# Add debug tools, development packages

sudo vim /etc/rpm-ostree/treefiles/kinoite-prod.json
# Keep minimal, production-ready packages
```

**Option 2: Use separate group_vars files**

Create `group_vars/dev_servers.yml` and `group_vars/prod_servers.yml` with different package lists.

## Manual Operations

### Manual Composition

**Dev:**
```bash
sudo rpm-ostree compose tree \
  --repo=/srv/ostree/rpm-ostree/dev \
  /etc/rpm-ostree/treefiles/kinoite-dev.json

sudo ostree summary -u --repo=/srv/ostree/rpm-ostree/dev
```

**Prod:**
```bash
sudo rpm-ostree compose tree \
  --repo=/srv/ostree/rpm-ostree/prod \
  /etc/rpm-ostree/treefiles/kinoite-prod.json

sudo ostree summary -u --repo=/srv/ostree/rpm-ostree/prod
```

### Manual Promotion

If you prefer manual promotion without Ansible:

```bash
# Get latest dev commit
DEV_COMMIT=$(ostree rev-parse --repo=/srv/ostree/rpm-ostree/dev fedora/x86_64/kinoite/dev)

# Pull into prod
sudo ostree pull-local \
  --repo=/srv/ostree/rpm-ostree/prod \
  /srv/ostree/rpm-ostree/dev \
  fedora/x86_64/kinoite/dev

# Create prod ref
sudo ostree refs \
  --repo=/srv/ostree/rpm-ostree/prod \
  --create=fedora/x86_64/kinoite/stable \
  $DEV_COMMIT

# Update summary
sudo ostree summary -u --repo=/srv/ostree/rpm-ostree/prod
```

### Check Current Versions

**On server:**
```bash
# Dev version
ostree log --repo=/srv/ostree/rpm-ostree/dev fedora/x86_64/kinoite/dev | head -1

# Prod version
ostree log --repo=/srv/ostree/rpm-ostree/prod fedora/x86_64/kinoite/stable | head -1
```

**On clients:**
```bash
rpm-ostree status
```

## Client Management

### Dev Clients

```bash
# Configure dev remote
sudo ostree remote add --no-gpg-verify kinoite-dev \
  https://192.168.35.35:8443/repo/kinoite/dev

# Initial deployment
sudo rpm-ostree rebase kinoite-dev:fedora/x86_64/kinoite/dev

# Update to latest dev
sudo rpm-ostree upgrade
sudo systemctl reboot
```

### Prod Clients

```bash
# Configure prod remote
sudo ostree remote add --no-gpg-verify kinoite-prod \
  https://192.168.35.35:8443/repo/kinoite/prod

# Initial deployment
sudo rpm-ostree rebase kinoite-prod:fedora/x86_64/kinoite/stable

# Update to latest prod
sudo rpm-ostree upgrade
sudo systemctl reboot
```

### Switch Between Environments

**From prod to dev (for testing):**
```bash
sudo rpm-ostree rebase kinoite-dev:fedora/x86_64/kinoite/dev
sudo systemctl reboot
```

**From dev back to prod:**
```bash
sudo rpm-ostree rebase kinoite-prod:fedora/x86_64/kinoite/stable
sudo systemctl reboot
```

## Best Practices

### ✅ Development Workflow

1. **Always test in dev first**
   - Compose new images in dev
   - Deploy to test workstations
   - Validate for at least 24-48 hours

2. **Use version tags**
   - Tag commits with version numbers
   - Document changes in commit messages

3. **Automated testing**
   - Run automated tests on dev images
   - Check system logs for errors
   - Monitor performance metrics

### ✅ Production Workflow

1. **Controlled rollouts**
   - Promote only validated dev images
   - Deploy to a subset of prod clients first
   - Monitor for issues before full deployment

2. **Rollback plan**
   - Keep previous prod commits available
   - Test rollback procedures
   - Document rollback steps

3. **Change management**
   - Document all promotions
   - Track which commits are in prod
   - Maintain changelog

### ✅ Security

1. **Separate access controls**
   - Limit who can promote to prod
   - Audit promotion operations
   - Use GPG signing for prod images

2. **Testing requirements**
   - Security scanning on dev images
   - Vulnerability assessments
   - Compliance checks before promotion

## Automation

### Scheduled Dev Builds

Create a cron job for nightly dev builds:

```bash
# On the server
sudo crontab -e

# Build dev nightly at 2 AM
0 2 * * * rpm-ostree compose tree --repo=/srv/ostree/rpm-ostree/dev /etc/rpm-ostree/treefiles/kinoite-dev.json && ostree summary -u --repo=/srv/ostree/rpm-ostree/dev
```

### Automated Promotion

Create a promotion script with validation:

```bash
#!/bin/bash
# promote-if-validated.sh

# Run tests
./run-dev-tests.sh || exit 1

# If tests pass, promote
cd /path/to/ansible-rpm-ostree-repo
ansible-playbook -i inventory/hosts.yml playbooks/promote-to-prod.yml
```

### CI/CD Integration

Integrate with Jenkins, GitLab CI, or GitHub Actions:

```yaml
# Example GitLab CI
dev-compose:
  stage: build
  script:
    - ansible-playbook -i inventory/hosts.yml playbooks/setup-rpm-ostree-repo.yml --tags compose
  only:
    - dev

prod-promote:
  stage: deploy
  script:
    - ansible-playbook -i inventory/hosts.yml playbooks/promote-to-prod.yml
  when: manual
  only:
    - main
```

## Troubleshooting

### Dev and Prod Out of Sync

```bash
# Compare commits
ostree log --repo=/srv/ostree/rpm-ostree/dev fedora/x86_64/kinoite/dev | head -5
ostree log --repo=/srv/ostree/rpm-ostree/prod fedora/x86_64/kinoite/stable | head -5
```

### Promotion Fails

Check for:
1. Disk space in `/srv`
2. OSTree repository integrity
3. Network issues between dev and prod repos (if on different servers)

### Clients Can't Pull

```bash
# Verify endpoints
curl -k https://192.168.35.35:8443/repo/kinoite/dev/config
curl -k https://192.168.35.35:8443/repo/kinoite/prod/config

# Check Apache config
sudo cat /etc/httpd/conf.d/rpm-ostree-repo.conf

# Verify SELinux
ls -lZ /srv/ostree/rpm-ostree/dev
ls -lZ /srv/ostree/rpm-ostree/prod
```

## Example Workflow

### Week 1: Development

```bash
# Day 1: Compose new dev image
ansible-playbook -i inventory/hosts.yml playbooks/setup-rpm-ostree-repo.yml --tags compose

# Day 2-7: Test on dev clients
# Monitor, gather feedback, fix issues
```

### Week 2: Promotion

```bash
# Monday: Final validation
# Run test suite on dev clients

# Tuesday: Promote to prod
ansible-playbook -i inventory/hosts.yml playbooks/promote-to-prod.yml

# Wednesday: Pilot deployment
# Update 10% of prod clients

# Thursday-Friday: Full rollout
# Update remaining prod clients
```

## Summary

**Dev Environment:**
- Fast iteration
- Latest packages and patches
- Testing ground
- Higher risk tolerance

**Prod Environment:**
- Stable, validated images
- Controlled updates
- Production workloads
- Zero downtime requirements

**Promotion Process:**
- Automated with Ansible
- Validates before promoting
- Updates summaries
- Ready for immediate client deployment

---

**Ready for enterprise-grade rpm-ostree management!** 🚀
