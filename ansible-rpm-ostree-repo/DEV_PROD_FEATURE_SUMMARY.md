# Dev/Prod Environment Feature - Summary

## What Was Added

Complete dev/prod separation for rpm-ostree image management with automated promotion workflow!

### Dual Repository System

**Before (Single Repo):**
```
/srv/ostree/rpm-ostree/kinoite/
https://server:8443/repo/kinoite
```

**Now (Dev + Prod):**
```
/srv/ostree/rpm-ostree/
├── dev/   → https://server:8443/repo/kinoite/dev
└── prod/  → https://server:8443/repo/kinoite/prod
```

## Key Features

### ✅ Separate Environments
- **DEV**: For testing new patches and packages
- **PROD**: For stable, production-ready images
- Independent OSTree repositories
- Separate Apache endpoints
- Individual treefiles

### ✅ Automated Promotion
- New playbook: `promote-to-prod.yml`
- Safely migrates validated images from dev to prod
- Uses `ostree pull-local` for efficient transfer
- Automatic summary updates
- Verification steps

### ✅ Flexible Composition
- Compose dev independently
- Compose prod separately
- Or compose both together
- Configurable via group_vars

## Files Modified/Created

### Updated Files:
- `group_vars/rpm_ostree_repo_servers.yml` - Multi-environment configuration
- `roles/rpm_ostree_repo/tasks/main.yml` - Creates both repos
- `roles/rpm_ostree_apache/templates/*.conf.j2` - Both endpoints
- `roles/rpm_ostree_compose/tasks/main.yml` - Multi-env composition

### New Files:
- `playbooks/promote-to-prod.yml` - Promotion automation
- `DEV_PROD_WORKFLOW.md` - Complete workflow guide
- `DEV_PROD_FEATURE_SUMMARY.md` - This file

## Quick Start

### 1. Initial Setup

Run the playbook (creates both dev and prod):

```bash
ansible-playbook -i inventory/hosts.yml playbooks/setup-rpm-ostree-repo.yml
```

### 2. Compose Dev Image

```bash
# Manual
ssh server
sudo rpm-ostree compose tree \
  --repo=/srv/ostree/rpm-ostree/dev \
  /etc/rpm-ostree/treefiles/kinoite-dev.json
sudo ostree summary -u --repo=/srv/ostree/rpm-ostree/dev

# Or automatic
# Set in group_vars: rpm_ostree_compose_on_setup: true
# Set in group_vars: rpm_ostree_compose_environments: [dev]
```

### 3. Test on Dev Clients

```bash
sudo ostree remote add --no-gpg-verify kinoite-dev \
  https://192.168.35.35:8443/repo/kinoite/dev

sudo rpm-ostree rebase kinoite-dev:fedora/x86_64/kinoite/dev
sudo systemctl reboot
```

### 4. Promote to Prod

After validation:

```bash
ansible-playbook -i inventory/hosts.yml playbooks/promote-to-prod.yml
```

### 5. Deploy to Prod Clients

```bash
sudo ostree remote add --no-gpg-verify kinoite-prod \
  https://192.168.35.35:8443/repo/kinoite/prod

sudo rpm-ostree rebase kinoite-prod:fedora/x86_64/kinoite/stable
sudo systemctl reboot
```

## Configuration

### group_vars/rpm_ostree_repo_servers.yml

```yaml
# Environment configuration
rpm_ostree_environments:
  - name: dev
    enabled: true
  - name: prod
    enabled: true

# Repository paths
rpm_ostree_repo_dev_path: /srv/ostree/rpm-ostree/dev
rpm_ostree_repo_prod_path: /srv/ostree/rpm-ostree/prod

# Apache endpoints
rpm_ostree_http_alias_dev: /repo/kinoite/dev
rpm_ostree_http_alias_prod: /repo/kinoite/prod

# OSTree refs
rpm_ostree_ref_dev: fedora/x86_64/kinoite/dev
rpm_ostree_ref_prod: fedora/x86_64/kinoite/stable

# Composition settings
rpm_ostree_compose_on_setup: false
rpm_ostree_compose_environments:
  - dev  # Usually only compose dev automatically
```

## Workflow Summary

```
┌─────────────┐
│ 1. Compose  │
│    DEV      │  New patches/packages
└──────┬──────┘
       │
       ↓
┌─────────────┐
│ 2. Test on  │
│ Dev Clients │  24-48 hours validation
└──────┬──────┘
       │
       ↓
┌─────────────┐
│ 3. Promote  │
│  to PROD    │  ansible-playbook promote-to-prod.yml
└──────┬──────┘
       │
       ↓
┌─────────────┐
│ 4. Deploy   │
│ Prod Clients│  Controlled rollout
└─────────────┘
```

## Promotion Playbook Details

**What it does:**
1. Verifies both repositories exist
2. Checks for commits in dev
3. Gets latest dev commit
4. Pulls commit into prod using `ostree pull-local`
5. Creates prod ref pointing to commit
6. Updates prod repository summary
7. Displays promotion confirmation

**Safety features:**
- Validates repositories before operation
- Fails if dev has no commits
- Shows what will be promoted
- Verifies after promotion

## Benefits

### ✅ Risk Mitigation
- Test patches in dev before prod
- Isolate unstable changes
- Quick rollback capability

### ✅ Controlled Deployment
- Validate in dev environment
- Promote only tested images
- Staged rollout to prod clients

### ✅ Operational Excellence
- Clear separation of concerns
- Audit trail of promotions
- Reproducible deployments

### ✅ Flexibility
- Different packages per environment
- Independent update schedules
- Per-environment configuration

## Use Cases

### Security Patches
1. Compose new image with patch in dev
2. Test on dev workstations
3. Verify patch effectiveness
4. Promote to prod
5. Roll out to all clients

### Major Updates
1. Test Fedora version upgrade in dev
2. Extensive validation period
3. Fix any compatibility issues
4. Promote to prod when stable
5. Gradual prod deployment

### Application Updates
1. Add new applications to dev
2. User acceptance testing
3. Gather feedback
4. Promote approved changes
5. Deploy organization-wide

## Monitoring

### Check Current State

**Dev:**
```bash
curl -k https://192.168.35.35:8443/repo/kinoite/dev/config
ostree log --repo=/srv/ostree/rpm-ostree/dev fedora/x86_64/kinoite/dev | head -5
```

**Prod:**
```bash
curl -k https://192.168.35.35:8443/repo/kinoite/prod/config
ostree log --repo=/srv/ostree/rpm-ostree/prod fedora/x86_64/kinoite/stable | head -5
```

### Compare Versions

```bash
# On server
DEV_COMMIT=$(ostree rev-parse --repo=/srv/ostree/rpm-ostree/dev fedora/x86_64/kinoite/dev)
PROD_COMMIT=$(ostree rev-parse --repo=/srv/ostree/rpm-ostree/prod fedora/x86_64/kinoite/stable)

echo "DEV:  $DEV_COMMIT"
echo "PROD: $PROD_COMMIT"
```

## Next Steps

1. **Run initial setup** - Creates both dev and prod repos
2. **Compose dev image** - Test new configurations
3. **Set up dev clients** - Point test workstations to dev
4. **Validate thoroughly** - Ensure stability
5. **Promote to prod** - Use promotion playbook
6. **Deploy to prod clients** - Controlled rollout

## Documentation

- `DEV_PROD_WORKFLOW.md` - Complete workflow guide with examples
- `README.md` - Updated with dev/prod information
- `COMPOSE_GUIDE.md` - Composition for both environments
- Role defaults - Inline documentation

---

**Enterprise-ready dev/prod workflow for rpm-ostree!** 🚀

See `DEV_PROD_WORKFLOW.md` for complete documentation.
