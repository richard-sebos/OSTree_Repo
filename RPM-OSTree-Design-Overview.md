# rpm-ostree Repository Infrastructure - Design Overview


- Updated should be a simple thing.
- When you have a couple of home device it can be but once you get into medium to large companies it becomes more complaicated.
- I worked for a company once that had about 50 to 60 endpoints and a Windows administrator sent an email to everyone to updates their device ASAP.
- I know most of the time Windows updates do causes issues but personally, I was not will to bet the business on it.
- I quickly told the critical departments to wait until the end of the day.
- We had not patch management or testing process.
- In the end, the update was successful, which I think allow the live patching of endpoint without developing plan to continue.

- In the last post the Kinoite Flatpak local repo that allow to test and certif application and publish them to the repo so users can install only approved allocations
- It used OSTree implement the repo and next the same OSTree will be used to store updates images to be rebase to the Kinoite installed.

## Updating
- in a traditional RHEL Linux system commands like `yum` or `dnf` are used to update the system.
- It download indivual updates and installs them on the base install.
- If there are issues, then either some type of rollback is needed or the updatesd need to be backout and error left dealt with
- Kinoite works differently, a new OS image is download and when the system reboots, the new image becomes active
- The existing OS image can be left on the system and the user can fall back to that image if there are issues with the update

## Setup OSTree Repo
- The Flatpak local repo setup from last post, setup an OSTree repo that will be used as a base for composing images.
- This will allow the website used by the Flatpak repo to be used by the OSTree repo, which includes the cert that was setup on the webserver.
- The only addiitional pached needed to install was was rpm-ostree

### RPM OSTree Repo
- before Kinoite images can be built, a repo storage needs to be created.
- A dev and prod location is created so dev images can be created and test and promoted to prod
```
     /srv/ostree/rpm-ostree/
     ├── dev/
     └── prod/

```
- 'ostree init` is run to build initialize the repos
- This builds the structure to compose the Kinooteimages to deploy

### SELinux Changes
- With web server installed, if the files permission are not setup right, the files are exposed.
- SELinux is a additional security tool built into RHEL family distros.
- There is a SELinux contact for Apache webserver that allow read access to file and directories but prevents:
* ❌ Write to files
* ❌ Create new files
* ❌ Delete files
* ❌ Modify files
* ❌ Execute files as programs
* ❌ Change permissions
* ❌ Change ownership
- The webserver is how OS Images are going to be access but the employees devices, this step allows the web server to have access to the files but not allowed to change them, securing the images.
- Since these images will be deployed across the business, securing is important.

### Deploying Images
- With the repo built and secured, it is time to expose access to the user devices.
- HTTPS was setup for Flatpak and the same will be used here.
- Additional configuration changes were needed to the Apache server to allow ` https://kinoite.sebostech.local:8443/repo/kinoite/dev/` and ` https://kinoite:8443/repo/kinoite/prod/` where the images can be access from.
- No additional firewall changes were needed.

### Composing an Image
- the last step is to build a new Kinoite image.
- On the first build, a `/etc/rpm-ostree/treefiles/kinoite-dev.json` and `/etc/rpm-ostree/treefiles/kinoite-prod.json` is needed to define  Critical packages to include in the image.
- `rpm-ostree compose tree` builds.
  - ✅ The filesystem tree (directory structure)
  - ✅ The OS image (complete operating system)
  - ✅ The OSTree commit (versioned snapshot)
- `ostree summary -u` create a repository summary, refreshes the repo's index/
- From there the dev image can be tested and once ready moved to production to be deployed.

## So why do this
- Installing Kinoite desktop allows for a common desktop across the orginaization.
- Creating a dev update process allows for testing of an update image before being deployed to production.
- Migrating dev images to production allows for secur
3. Optionally composes OS images (if enabled)
   ├─ Compose dev image
   └─ Compose prod image

4. Updates repository summaries (if composed)
   ├─ Update dev repo summary
   └─ Update prod repo summary

5. Displays instructions
   └─ Shows how to manually compose if not automatic

## Executive Summary

This document outlines the design and implementation of an automated rpm-ostree repository infrastructure for hosting custom Fedora Kinoite images. The solution extends existing Flatpak OSTree infrastructure and implements a dev/prod workflow for safe image deployment using Ansible automation.

## Business Requirements

### Primary Objectives
- Host custom Fedora Kinoite operating system images
- Leverage existing Apache/OSTree infrastructure
- Implement dev/prod environment separation for safe testing
- Automate repository setup and configuration
- Maintain security with SELinux enforcement and HTTPS

### Success Criteria
- Automated, repeatable deployment via Ansible
- Separate development and production repositories
- HTTPS access with existing SSL infrastructure
- Proper SELinux contexts for security compliance
- Safe promotion workflow from dev to production

## Architecture Overview

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     Apache HTTPS Server                      │
│                      (Port 8443/8080)                        │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  ┌─────────────────────┐      ┌─────────────────────┐       │
│  │  Flatpak Repository │      │  Kinoite Repository │       │
│  │  /repo/flatpak/     │      │  /repo/kinoite/     │       │
│  └─────────────────────┘      └─────────────────────┘       │
│                                         │                     │
│                         ┌───────────────┴────────────┐       │
│                         │                            │       │
│                    ┌────▼─────┐              ┌──────▼────┐  │
│                    │   DEV    │              │   PROD    │  │
│                    │  Repo    │─────────────▶│   Repo    │  │
│                    │ (Test)   │  Promotion   │ (Stable)  │  │
│                    └──────────┘              └───────────┘  │
│                                                               │
└─────────────────────────────────────────────────────────────┘
                              │
                 ┌────────────┴────────────┐
                 │                         │
         ┌───────▼────────┐       ┌───────▼────────┐
         │  Kinoite       │       │  Kinoite       │
         │  Dev Client    │       │  Prod Client   │
         └────────────────┘       └────────────────┘
```

### Infrastructure Components

| Component | Purpose | Technology |
|-----------|---------|------------|
| Web Server | Serve OSTree repositories via HTTP/S | Apache 2.4 |
| OSTree Repositories | Store immutable OS images | OSTree (archive-z2 mode) |
| SSL/TLS | Encrypted communication | Existing SSL certificates |
| Firewall | Port access control | firewalld |
| Security Context | Mandatory access control | SELinux (Enforcing) |
| Automation | Infrastructure as Code | Ansible 2.15+ |

## Network Design

### Port Configuration

| Port | Protocol | Service | Purpose |
|------|----------|---------|---------|
| 8080 | HTTP | Apache | Fallback access (redirects to HTTPS) |
| 8443 | HTTPS | Apache | Primary secure access |

### URL Schema

| Repository | Environment | URL |
|------------|-------------|-----|
| Flatpak | Production | `https://SERVER:8443/repo/flatpak/` |
| Kinoite | Development | `https://SERVER:8443/repo/kinoite/dev/` |
| Kinoite | Production | `https://SERVER:8443/repo/kinoite/prod/` |

**Design Decision**: Single port for multiple repositories using URL path differentiation
- Simplifies firewall configuration
- Single SSL certificate management
- Reduces attack surface

## Storage Architecture

### Directory Structure

```
/srv/ostree/
├── flatpak/              # Existing Flatpak repository
│   ├── config
│   └── ...
└── rpm-ostree/           # New rpm-ostree repository base
    ├── dev/              # Development environment
    │   ├── config
    │   ├── objects/      # Deduplicated object storage
    │   ├── refs/         # Branch references
    │   ├── state/
    │   └── tmp/
    └── prod/             # Production environment
        ├── config
        ├── objects/
        ├── refs/
        ├── state/
        └── tmp/
```

### Repository Mode: archive-z2

**Characteristics**:
- Optimized for HTTP serving
- Files stored compressed
- Content-addressable storage
- Object deduplication across commits

**Storage Efficiency**:
- Base OS image: ~2-3 GB
- Incremental updates: ~100-500 MB
- Shared objects between dev/prod when promoted

## Security Design

### SELinux Configuration

**Context Applied**: `httpd_sys_content_t`

```
File Path: /srv/ostree/rpm-ostree/
SELinux User: unconfined_u
SELinux Role: object_r
SELinux Type: httpd_sys_content_t
SELinux Level: s0
```

**Purpose**: Allows Apache to read repository content while maintaining system security

**Benefits**:
- Enforcing mode maintained
- No permissive exceptions required
- Follows principle of least privilege
- Prevents unauthorized access

### SSL/TLS Configuration

**Certificate Reuse Strategy**:
- Leverages existing Flatpak SSL certificates
- Single certificate for multiple repositories
- Reduces certificate management overhead

**Certificate Locations**:
- Certificate: `/etc/pki/tls/certs/flatpak-repo.crt`
- Private Key: `/etc/pki/tls/private/flatpak-repo.key`

**Current State**: Self-signed certificates for internal use

**Production Recommendation**:
- CA-signed certificates for external access
- Let's Encrypt for automated renewal
- Internal CA for enterprise environments

### Access Control

**Current Implementation**: `Require all granted`

**Rationale**: Internal repository server with network-level access control

**Production Considerations**:
- IP-based restrictions for dev environment
- Client certificate authentication (optional)
- VPN access requirement for external clients

## Workflow Design

### Development to Production Pipeline

```
┌──────────────┐
│   Compose    │
│  Dev Image   │
└──────┬───────┘
       │
       ▼
┌──────────────┐
│   Test in    │
│     Dev      │
└──────┬───────┘
       │
       ▼
┌──────────────┐      ┌──────────────┐
│   Validate   │──No──│  Fix Issues  │
│    Image     │      │  Re-compose  │
└──────┬───────┘      └──────┬───────┘
       │                     │
      Yes                    │
       │◄────────────────────┘
       ▼
┌──────────────┐
│   Promote    │
│  to Prod     │
└──────┬───────┘
       │
       ▼
┌──────────────┐
│  Deploy to   │
│   Clients    │
└──────────────┘
```

### Promotion Process

**Method**: `ostree pull-local`

**Advantages**:
- Local operation (no network transfer)
- Efficient object copying
- Preserves commit signatures
- Atomic operation

**Process**:
1. Verify dev repository has commits
2. Pull commits from dev to prod repository
3. Create/update stable ref in prod
4. Update repository summary
5. Clients automatically see new version

## Ansible Automation Design

### Role-Based Architecture

```
┌─────────────────────────────────────────┐
│         Main Playbook                   │
│   setup-rpm-ostree-repo.yml             │
└────────────┬────────────────────────────┘
             │
    ┌────────┴────────┐
    │   Pre-Tasks     │
    │  (Install pkgs) │
    └────────┬────────┘
             │
    ┌────────▼──────────────────────┐
    │                               │
    │  ┌────────────────────────┐   │
    │  │  rpm_ostree_repo       │   │  Repository initialization
    │  └────────────────────────┘   │
    │           │                   │
    │  ┌────────▼────────────────┐  │
    │  │  rpm_ostree_selinux     │  │  SELinux configuration
    │  └─────────────────────────┘  │
    │           │                   │
    │  ┌────────▼────────────────┐  │
    │  │  rpm_ostree_apache      │  │  Web server setup
    │  └─────────────────────────┘  │
    │           │                   │
    │  ┌────────▼────────────────┐  │
    │  │  rpm_ostree_firewall    │  │  Firewall configuration
    │  └─────────────────────────┘  │
    │           │                   │
    │  ┌────────▼────────────────┐  │
    │  │  rpm_ostree_compose     │  │  Image composition setup
    │  └─────────────────────────┘  │
    │                               │
    └───────────────────────────────┘
             │
    ┌────────▼────────┐
    │  Post-Tasks     │
    │ (Display info)  │
    └─────────────────┘
```

### Role Responsibilities

| Role | Responsibility | Key Actions |
|------|----------------|-------------|
| rpm_ostree_repo | Repository initialization | Create directories, initialize OSTree repos, set permissions |
| rpm_ostree_selinux | Security context | Configure SELinux policy, apply contexts, verify |
| rpm_ostree_apache | Web server configuration | Deploy Apache config, configure SSL, reload service |
| rpm_ostree_firewall | Network access | Verify/open firewall ports, make permanent |
| rpm_ostree_compose | Image composition | Deploy treefiles, optional image composition |

### Idempotency Design

**Principle**: Running playbook multiple times produces same result without errors

**Implementation Strategies**:
- Check before create (stat module)
- Template-based configuration (replaces if changed)
- Conditional task execution (when clauses)
- Firewall rule verification before adding

**Benefits**:
- Safe to re-run
- Self-healing on configuration drift
- Easy updates and modifications

### Variable Hierarchy

```
┌─────────────────────────────────┐
│  Role Defaults                  │  Lowest Priority
│  (defaults/main.yml)            │  Built-in fallbacks
└────────────┬────────────────────┘
             │
             │ Overridden by
             │
┌────────────▼────────────────────┐
│  Group Variables                │  Medium Priority
│  (group_vars/*.yml)             │  Environment-specific
└────────────┬────────────────────┘
             │
             │ Overridden by
             │
┌────────────▼────────────────────┐
│  Host Variables                 │  Highest Priority
│  (host_vars/*.yml)              │  Server-specific
└─────────────────────────────────┘
```

**Strategy**: Each role provides complete defaults, overridden centrally in group_vars

## Image Composition Design

### Treefile Architecture

**Purpose**: JSON manifest defining operating system composition

**Key Sections**:

| Section | Purpose | Example |
|---------|---------|---------|
| ref | Branch identifier | `fedora/x86_64/kinoite/dev` |
| repos | DNF repositories to use | `fedora`, `fedora-updates` |
| packages | Packages to include | `kernel`, `plasma-desktop` |
| exclude-packages | Packages to omit | `PackageKit` |
| postprocess | Post-install commands | `systemctl enable sddm` |

### Dev vs Prod Configuration

| Aspect | Development | Production |
|--------|-------------|------------|
| Repository | `/srv/ostree/rpm-ostree/dev` | `/srv/ostree/rpm-ostree/prod` |
| OSTree Ref | `fedora/x86_64/kinoite/dev` | `fedora/x86_64/kinoite/stable` |
| Treefile | `kinoite-dev.json` | `kinoite-prod.json` |
| Update Frequency | Weekly or as-needed | Monthly or after validation |
| Purpose | Testing and validation | Stable client deployments |

### Composition Process

```
┌─────────────────┐
│   Treefile      │  Define OS composition
│ (JSON manifest) │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  rpm-ostree     │  Download packages from DNF repos
│   compose tree  │  Apply postprocess scripts
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│   OSTree        │  Create immutable commit
│    Commit       │  Store in content-addressable format
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│    ostree       │  Generate metadata index
│   summary -u    │  Make visible to clients
└─────────────────┘
```

**Time Requirements**:
- First compose: 15-30 minutes (package downloads)
- Subsequent: 5-15 minutes (cached packages)

## Scalability Considerations

### Current Design Capacity

**Single Server Architecture**:
- Supports 50-100 concurrent clients comfortably
- Bandwidth-dependent for simultaneous updates
- Content deduplication reduces storage growth

### Scaling Strategies

**Horizontal Scaling**:
1. **Mirror Repositories**: Multiple servers with synced content
2. **Load Balancer**: Distribute client requests across mirrors
3. **CDN Integration**: Cache static OSTree objects

**Vertical Scaling**:
1. **Storage**: Network-attached storage for repositories
2. **Memory**: Faster repository operations
3. **Network**: Increased bandwidth for simultaneous updates

### Performance Optimization

**Repository Summary**:
- Cached by clients
- Small file size (~few KB)
- Enables efficient update checks

**Object Storage**:
- Content-addressable (SHA256)
- Automatic deduplication
- Only changed objects transferred

**Delta Updates**:
- Static deltas can be pre-generated
- Reduces client download size
- Trades server storage for client bandwidth

## Disaster Recovery

### Backup Strategy

**Critical Data**:
1. OSTree repositories: `/srv/ostree/rpm-ostree/`
2. Treefiles: `/etc/rpm-ostree/treefiles/`
3. Apache configuration: `/etc/httpd/conf.d/rpm-ostree-*.conf`
4. SSL certificates: `/etc/pki/tls/`

**Backup Frequency**:
- After each successful composition
- Before major configuration changes
- Daily automated backups recommended

### Recovery Procedures

**Repository Corruption**:
1. Stop Apache service
2. Restore from backup
3. Verify repository integrity (`ostree fsck`)
4. Regenerate summary
5. Restart Apache

**Complete Server Loss**:
1. Rebuild server with same IP/hostname
2. Restore Ansible inventory
3. Run setup playbook
4. Restore repository data from backup
5. Verify client connectivity

## Monitoring and Maintenance

### Health Checks

**Service Monitoring**:
- Apache service status
- Firewall active and configured
- SELinux enforcing
- Disk space available

**Repository Integrity**:
```bash
# Verify repository integrity
ostree fsck --repo=/srv/ostree/rpm-ostree/dev
ostree fsck --repo=/srv/ostree/rpm-ostree/prod

# Check for available updates
ostree refs --repo=/srv/ostree/rpm-ostree/prod
```

**Access Verification**:
```bash
# Test HTTPS access
curl -k https://SERVER:8443/repo/kinoite/dev/config
curl -k https://SERVER:8443/repo/kinoite/prod/config
```

### Maintenance Tasks

| Task | Frequency | Command |
|------|-----------|---------|
| Compose dev image | Weekly | `rpm-ostree compose tree --repo=...` |
| Test dev image | After composition | Deploy to test client |
| Promote to prod | Monthly | `ansible-playbook promote-to-prod.yml` |
| Prune old commits | Monthly | `ostree prune --repo=... --keep-younger-than=...` |
| Update SSL cert | Before expiry | Certificate renewal process |
| Review logs | Weekly | `journalctl -u httpd` |

## Future Architecture Enhancements

### Phase 2 Enhancements

1. **GPG Signing**
   - Sign commits during composition
   - Distribute public keys to clients
   - Enable signature verification

2. **Automated Composition**
   - Systemd timer for scheduled composition
   - Automatic dev environment updates
   - Email notifications on success/failure

3. **CI/CD Integration**
   - GitLab/GitHub Actions integration
   - Automated testing pipeline
   - Automatic promotion on test success

### Phase 3 Enhancements

1. **Multi-Environment**
   - Add staging environment
   - Development → Staging → Production
   - Environment-specific treefiles

2. **Metrics and Monitoring**
   - Prometheus exporters
   - Grafana dashboards
   - Client update tracking
   - Repository size trending

3. **High Availability**
   - Repository mirroring
   - Automated failover
   - Geographic distribution

## Comparison with Alternatives

### OSTree HTTP Repository vs Package-Based Updates

| Aspect | rpm-ostree (This Design) | Traditional RPM |
|--------|--------------------------|-----------------|
| Update Model | Atomic, image-based | Package-by-package |
| Rollback | Built-in, instant | Manual, complex |
| Testing | Entire OS image | Individual packages |
| Bandwidth | Only changed objects | Full package downloads |
| Consistency | Guaranteed identical | May vary per system |
| Customization | Treefile + layers | Package selection |

### Build vs Host Approach

| Approach | This Design | Alternative |
|----------|-------------|-------------|
| **Build Server** | Centralized composition | Each client composes |
| Consistency | Identical across clients | Varies per client |
| Client Resources | Minimal | High (build tools) |
| Update Speed | Fast (pre-built) | Slow (local build) |
| Storage | Server-side | Client-side |
| Testing | Before distribution | After distribution |

## Technical Debt and Limitations

### Current Limitations

1. **No GPG Signing**: Clients use `--no-gpg-verify`
2. **Self-Signed SSL**: Requires `-k` flag for curl/wget
3. **Manual Promotion**: Requires running playbook
4. **No Automated Testing**: Manual testing in dev required
5. **Single Server**: No redundancy or failover

### Mitigation Strategies

**Short-Term** (0-3 months):
- Document GPG signing procedure
- Acquire trusted SSL certificate
- Create promotion documentation

**Medium-Term** (3-6 months):
- Implement GPG signing
- Deploy trusted SSL certificates
- Create automated testing framework

**Long-Term** (6-12 months):
- Implement CI/CD pipeline
- Deploy repository mirrors
- Automated composition scheduling

## Conclusion

This design provides a production-ready rpm-ostree repository infrastructure with:

✅ **Automated Deployment**: Complete Ansible automation
✅ **Security**: SELinux enforcing, HTTPS encryption
✅ **Safety**: Dev/prod separation with promotion workflow
✅ **Scalability**: Content deduplication, efficient storage
✅ **Maintainability**: Idempotent playbooks, clear documentation
✅ **Extensibility**: Modular role design for future enhancements

The architecture balances simplicity with functionality, providing a solid foundation for managing custom Fedora Kinoite deployments while maintaining clear upgrade paths for future enhancements.
