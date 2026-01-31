# Kinoite Update Model - Image-Based vs Package-Based Updates

## Executive Summary

Kinoite (and all rpm-ostree based systems) use an **image-based update model**, not traditional package-level updates. This is a fundamental architectural difference from traditional Fedora that affects how updates are composed, distributed, and applied.

## The Fundamental Difference

### Traditional Fedora (Package-Based Updates)

```
DNF/RPM → Downloads individual packages → Installs them one-by-one → Modifies running system
```

**Characteristics**:
- Updates modify the currently running system
- Each package is downloaded and installed individually
- System state changes incrementally
- Difficult to rollback
- System can be in inconsistent state during updates

### Kinoite (Image-Based Updates)

```
OSTree → Downloads entire OS image commit → Deploys as new bootable image → Reboot to switch
```

**Characteristics**:
- Updates deploy a complete new OS image
- Running system remains untouched during update
- New image activated on next reboot
- Easy rollback to previous image
- System always in consistent state

## Detailed Comparison

| Aspect | Traditional Fedora | Kinoite (rpm-ostree) |
|--------|-------------------|----------------------|
| **Update Unit** | Individual RPM packages | Entire OS image (commit) |
| **Update Method** | Package manager (DNF) | Image-based (OSTree) |
| **File System** | Mutable (writable /usr) | Immutable (read-only /usr) |
| **Updates Applied** | To running system | To new deployment |
| **Active During Update** | System being modified | System untouched |
| **Activation** | Immediate | Next reboot |
| **Rollback** | Complex/manual restoration | Simple reboot to previous image |
| **Interrupted Update** | Can corrupt system | No impact on current system |
| **Consistency** | Can vary between systems | Identical across all systems |
| **Testing** | After installation | Before distribution |
| **Customization** | Install any package | Layered packages + base image |

## The rpm-ostree Hybrid Approach

The name "rpm-ostree" reflects its hybrid nature - it uses RPM packages as **input** but delivers OSTree images as **output**.

### Server Side: Composition (RPM Input)

On your repository server, rpm-ostree **composes** OS images from RPM packages:

```bash
rpm-ostree compose tree --repo=/srv/ostree/rpm-ostree/dev kinoite-dev.json
```

**What happens**:
1. Reads treefile (kinoite-dev.json)
2. Downloads RPM packages from Fedora repositories
3. Assembles packages into a complete OS tree
4. Runs postprocess scripts
5. Creates an immutable OSTree commit (snapshot)
6. Stores in content-addressable format

**Input**: RPM packages from DNF repositories
**Output**: OSTree commit (complete OS image)

### Client Side: Deployment (OSTree Output)

On Kinoite desktops, clients consume OSTree images:

```bash
rpm-ostree upgrade
```

**What happens**:
1. Checks repository for new OS commit
2. Downloads only changed objects (deduplication)
3. Creates new bootloader entry
4. Sets new image as default boot option
5. Running system remains unchanged
6. Reboot activates new image

**Input**: OSTree commit from repository
**Output**: New bootable OS deployment

## Visual Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                     Repository Server                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  ┌──────────────┐     ┌──────────────┐     ┌────────────────┐  │
│  │   Fedora     │     │  rpm-ostree  │     │    OSTree      │  │
│  │ RPM Repos    │────▶│   compose    │────▶│    Commit      │  │
│  │ (packages)   │     │   (build)    │     │   (image)      │  │
│  └──────────────┘     └──────────────┘     └────────┬───────┘  │
│                                                      │           │
│                                             Stored in Repository │
│                                                      │           │
└──────────────────────────────────────────────────────┼───────────┘
                                                       │
                                          HTTPS (OSTree protocol)
                                                       │
                    ┌──────────────────────────────────┼──────────────┐
                    │                                  │              │
          ┌─────────▼────────┐              ┌─────────▼────────┐     │
          │  Kinoite Client  │              │  Kinoite Client  │     │
          │                  │              │                  │     │
          │  Deployment A    │              │  Deployment A    │     │
          │  (Current)       │              │  (Previous)      │     │
          │                  │              │                  │     │
          │  Deployment B    │              │  Deployment B    │     │
          │  (New - Pending) │              │  (Current)       │     │
          └──────────────────┘              └──────────────────┘     │
                    │                                  │              │
               Reboot switches                    Reboot switches     │
               to Deployment B                    to Deployment A     │
                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

## Conceptual Analogies

### Traditional Package Updates = Replacing Parts in a Running Car

```
┌──────────────────────────────────────────┐
│  Your Running Car (System)               │
├──────────────────────────────────────────┤
│  ❌ Replace engine while driving          │
│  ❌ Swap transmission in motion           │
│  ❌ Change tires at highway speed         │
│  ❌ If something fails, car breaks down   │
└──────────────────────────────────────────┘
```

**Problems**:
- Dangerous to modify running system
- Failure can leave system broken
- Hard to undo changes
- System state unpredictable during update

### Image-Based Updates = Multiple Complete Cars in Garage

```
┌──────────────────────────────────────────┐
│  Your Garage (Boot Options)              │
├──────────────────────────────────────────┤
│  🚗 Car A (Current OS - June 2025)       │
│  🚙 Car B (New OS - July 2025)           │
│  🚕 Car C (Old OS - May 2025)            │
└──────────────────────────────────────────┘

Switch cars by choosing which one to drive (boot)
If new car has problems, drive the old one
```

**Advantages**:
- Safe: current car untouched while new one prepared
- Reliable: new car fully built and tested
- Rollback: just choose previous car
- Predictable: each car is complete and tested

## The Update Workflow in Detail

### Repository Server Workflow

```
1. Administrator creates/updates treefile
   └─ Defines: packages, repos, postprocess scripts

2. Compose new image
   └─ rpm-ostree compose tree --repo=/path/to/repo treefile.json
      ├─ Downloads RPMs from Fedora repos
      ├─ Installs packages into temporary root
      ├─ Runs postprocess commands
      ├─ Commits to OSTree repository
      └─ Generates commit hash (e.g., a3f8b2c...)

3. Update repository summary
   └─ ostree summary -u --repo=/path/to/repo
      └─ Updates metadata index for clients

4. (Optional) Promote to production
   └─ ostree pull-local dev-repo stable-ref
      └─ Copies tested image to production
```

### Client Update Workflow

```
1. Check for updates
   └─ rpm-ostree upgrade --check
      └─ Queries repository summary

2. Download new image
   └─ rpm-ostree upgrade
      ├─ Downloads new commit metadata
      ├─ Downloads only changed objects (deduplication!)
      ├─ Verifies checksums
      └─ Creates new deployment

3. Current state after download
   ├─ Deployment 0: NEW (pending, will boot next)
   ├─ Deployment 1: CURRENT (currently running)
   └─ Deployment 2: PREVIOUS (rollback option)

4. Reboot to activate
   └─ systemctl reboot
      └─ GRUB boots into Deployment 0

5. After reboot
   ├─ Deployment 0: CURRENT (now running the new image)
   ├─ Deployment 1: PREVIOUS (old image, rollback option)
   └─ Deployment 2: (cleaned up or kept as second rollback)
```

## File System Structure Comparison

### Traditional Fedora

```
/
├── usr/         (read-write - packages modify this)
├── etc/         (read-write - configuration)
├── var/         (read-write - logs, data)
└── home/        (read-write - user files)
```

**Characteristics**:
- All directories writable
- Package updates modify /usr directly
- System state changes over time
- Configuration drift possible

### Kinoite (rpm-ostree)

```
/
├── usr/         (read-only - from OSTree image)
├── etc/         (read-write - configuration merged from image)
├── var/         (read-write - logs, data, containers)
└── home/        (read-write - user files)

Special:
├── /ostree/     (OSTree repository - multiple OS versions)
└── /sysroot/    (actual root filesystem)
```

**Characteristics**:
- /usr is immutable (read-only)
- /etc uses 3-way merge (base image + admin changes + upgrades)
- /var and /home persist across updates
- Multiple OS versions stored in /ostree

## Content Deduplication in Action

OSTree uses content-addressable storage, making updates efficient:

### Example Update Scenario

**Base Image (June 2025)**:
- kernel-6.8.0
- plasma-desktop-5.27
- systemd-255
- 2000 other packages
- **Total: 3 GB**

**Updated Image (July 2025)**:
- kernel-6.8.1 ← Changed
- plasma-desktop-5.27 ← Same
- systemd-255 ← Same
- 2000 other packages ← Mostly same

**Download Size**:
- Only kernel changed: ~50 MB
- Shared objects not re-downloaded
- **Actual download: ~200 MB** (not 3 GB!)

### How Deduplication Works

```
OSTree Storage (Content-Addressable)
├── objects/
│   ├── a3/f8b2c... (kernel 6.8.0)
│   ├── b4/a9d1e... (kernel 6.8.1) ← New
│   ├── c5/e7f3a... (plasma-desktop) ← Shared
│   ├── d6/b8c4f... (systemd) ← Shared
│   └── ... (2000+ other objects) ← Most shared
└── refs/
    ├── fedora/x86_64/kinoite/june → Points to objects: a3, c5, d6...
    └── fedora/x86_64/kinoite/july → Points to objects: b4, c5, d6...
```

**Key Insight**: Both images reference the same plasma-desktop and systemd objects. Only kernel object is new.

## Package Layering (Client Customization)

Clients can still install additional packages, but it works differently:

### Traditional Fedora
```bash
sudo dnf install vim
# Modifies /usr immediately
```

### Kinoite
```bash
rpm-ostree install vim
# Creates new deployment with vim added
# Requires reboot to activate
```

**What happens**:
1. Downloads vim RPM
2. Creates new deployment = base image + vim
3. New deployment includes both base packages and vim
4. Reboot to activate

**Layered packages persist** across base image updates:
```
Base Image Update + Your Layered Packages = New Deployment
(from server)         (client additions)      (automatic merge)
```

## Your Repository Setup - The Complete Flow

### On Your Repository Server

```bash
# 1. Compose dev image
sudo rpm-ostree compose tree \
  --repo=/srv/ostree/rpm-ostree/dev \
  /etc/rpm-ostree/treefiles/kinoite-dev.json

# What this does:
# - Reads package list from treefile
# - Downloads ~2GB of RPM packages from Fedora
# - Assembles into complete OS tree
# - Commits to OSTree repo
# - Takes 15-30 minutes

# 2. Update repository summary
sudo ostree summary -u --repo=/srv/ostree/rpm-ostree/dev

# 3. After testing, promote to production
ansible-playbook -i inventory/hosts.yml playbooks/promote-to-prod.yml

# What this does:
# - Copies commit from dev to prod
# - Updates prod summary
# - Clients can now download stable image
```

### On Kinoite Clients

```bash
# 1. Add your repository
sudo ostree remote add --no-gpg-verify kinoite-prod \
  https://192.168.35.35:8443/repo/kinoite/prod

# 2. Rebase to your custom image
sudo rpm-ostree rebase kinoite-prod:fedora/x86_64/kinoite/stable

# What this does:
# - Downloads ~2-3GB (first time)
# - Creates new deployment
# - Sets as default boot

# 3. Reboot to activate
systemctl reboot

# 4. Future updates
sudo rpm-ostree upgrade

# What this does:
# - Checks for new commits
# - Downloads only changed objects (~100-500 MB typically)
# - Creates new deployment
# - Reboot to activate
```

## Benefits of Image-Based Updates

### 1. Atomic Updates
- Update either succeeds completely or not at all
- No partial/broken states
- Safe to interrupt (power loss, network failure)

### 2. Reliable Rollback
```bash
# Before reboot: Don't like the pending update?
rpm-ostree rollback

# After reboot: New version has problems?
# Just select previous entry in GRUB menu
# Or: rpm-ostree rollback && reboot
```

### 3. Predictable State
- All systems with same commit are **identical**
- No "works on my machine" problems
- Easier troubleshooting

### 4. Safe Testing
- Test complete image in dev
- Same image goes to prod
- No surprises from different package versions

### 5. Efficient Distribution
- Deduplication reduces bandwidth
- Only changed objects transferred
- Multiple versions stored efficiently

## Common Misconceptions

### ❌ Misconception 1: "rpm-ostree updates individual packages"
**Reality**: rpm-ostree updates entire OS images. RPMs are only used during composition on the server.

### ❌ Misconception 2: "You can't install additional software"
**Reality**: You can layer packages with `rpm-ostree install`, or use Flatpak/containers for applications.

### ❌ Misconception 3: "Updates are slower than DNF"
**Reality**: Downloads may be larger initially, but deduplication makes subsequent updates efficient. Plus, you can download in background and reboot when convenient.

### ❌ Misconception 4: "Rollback requires backups"
**Reality**: Previous deployments are kept automatically. Rollback is instant (just a reboot).

### ❌ Misconception 5: "The filesystem is completely read-only"
**Reality**: Only /usr is read-only. /etc, /var, and /home are writable. Configuration and data persist.

## When to Use Which Model

### Use Traditional Package-Based (Fedora Workstation)
- Need frequent package installations/removals
- Development machine with many tools
- Custom software compilation
- Maximum flexibility required

### Use Image-Based (Kinoite)
- Stability and reliability critical
- Multiple identical systems to manage
- Want easy rollback capability
- Prefer containerized applications (Flatpak, Podman)
- Server/appliance use cases

## Technical Deep Dive: OSTree Commit Structure

### What is an OSTree Commit?

An OSTree commit is like a Git commit, but for an entire operating system:

```
Commit: a3f8b2c4d5e6f7a8b9c0d1e2f3a4b5c6d7e8f9a0
Parent: b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0a1b2c3
Date: 2025-07-15 14:30:00
Subject: Fedora Kinoite 41.20250715

├── /usr/
│   ├── bin/
│   ├── lib/
│   ├── lib64/
│   ├── share/
│   └── ...
└── /etc/  (default configuration)

Metadata:
- Version: fedora-kinoite-41.20250715
- Packages: 2134 packages (list included)
- Size: 3.2 GB (uncompressed)
- Objects: 45,678 content objects
```

### Content Addressing

Every file is stored by its SHA256 hash:

```
File: /usr/bin/bash
├── Content: [binary data]
├── SHA256: e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855
└── Stored as: objects/e3/b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855

If another file has same content → same object (deduplication)
If file changes → new object, old one kept for previous commits
```

## Summary

### Key Takeaways

1. **Kinoite uses image-based updates, not package-based updates**
   - Clients download complete OS images
   - RPMs are only used during server-side composition

2. **rpm-ostree is a hybrid system**
   - Input: RPM packages (familiar package format)
   - Output: OSTree commits (immutable images)

3. **Updates are atomic and safe**
   - Running system never modified during update
   - Easy rollback to previous images
   - Guaranteed consistency

4. **Efficient distribution**
   - Content deduplication reduces download size
   - Only changed objects transferred
   - Multiple versions stored efficiently

5. **Different mental model required**
   - Think "deployments" not "installed packages"
   - Think "reboot to activate" not "immediate changes"
   - Think "base image + layers" not "package collection"

### The rpm-ostree Philosophy

> "Operating systems should be managed like application containers: immutable, versioned, and reproducible."

This is the fundamental shift from traditional package management to image-based delivery.
