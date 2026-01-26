# Architecture Overview

## System Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     Your Server                             │
│                                                             │
│  ┌───────────────────────────────────────────────────────┐ │
│  │              Apache HTTP Server                        │ │
│  │              Port: 8080 (configurable)                │ │
│  │                                                        │ │
│  │  VirtualHost Configuration:                           │ │
│  │  ┌──────────────────────────────────────────────┐    │ │
│  │  │ /repo/flatpak (existing Flatpak repo)        │    │ │
│  │  │   → /var/flatpak/repo                        │    │ │
│  │  │   [From your Flatpak setup]                  │    │ │
│  │  └──────────────────────────────────────────────┘    │ │
│  │                                                        │ │
│  │  ┌──────────────────────────────────────────────┐    │ │
│  │  │ /repo/kinoite (NEW rpm-ostree repo)          │    │ │
│  │  │   → /srv/ostree/rpm-ostree/kinoite           │    │ │
│  │  │   [Added by this playbook]                   │    │ │
│  │  └──────────────────────────────────────────────┘    │ │
│  └───────────────────────────────────────────────────────┘ │
│                                                             │
│  ┌───────────────────────────────────────────────────────┐ │
│  │              Filesystem Layout                         │ │
│  │                                                        │ │
│  │  /var/flatpak/repo/                    (Flatpak)     │ │
│  │  ├── config                                           │ │
│  │  ├── objects/                                         │ │
│  │  ├── refs/                                            │ │
│  │  └── summary                                          │ │
│  │                                                        │ │
│  │  /srv/ostree/rpm-ostree/kinoite/       (rpm-ostree)  │ │
│  │  ├── config                                           │ │
│  │  ├── objects/                                         │ │
│  │  ├── refs/                                            │ │
│  │  └── summary                                          │ │
│  └───────────────────────────────────────────────────────┘ │
│                                                             │
│  ┌───────────────────────────────────────────────────────┐ │
│  │              SELinux Contexts                          │ │
│  │                                                        │ │
│  │  /var/flatpak/repo                                    │ │
│  │    → httpd_sys_content_t                              │ │
│  │                                                        │ │
│  │  /srv/ostree/rpm-ostree                               │ │
│  │    → httpd_sys_content_t (same policy)                │ │
│  └───────────────────────────────────────────────────────┘ │
│                                                             │
│  ┌───────────────────────────────────────────────────────┐ │
│  │              Firewall Configuration                    │ │
│  │  Port 8080/tcp → OPEN (from Flatpak setup)           │ │
│  │  [No changes needed]                                  │ │
│  └───────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

## Client Perspective

### Flatpak Clients

```
┌──────────────────┐
│  Flatpak Client  │
│                  │
│  flatpak remote  │
│    ↓             │
│  http://server:8080/repo/flatpak
│                  │
└──────────────────┘
```

### Kinoite/rpm-ostree Clients

```
┌──────────────────┐
│  Kinoite Client  │
│                  │
│  ostree remote   │
│    ↓             │
│  http://server:8080/repo/kinoite
│                  │
└──────────────────┘
```

## Data Flow

### rpm-ostree Image Composition

```
┌─────────────┐
│   Admin     │
│             │
└──────┬──────┘
       │
       │ 1. Create kinoite.json treefile
       ↓
┌─────────────────────────────────────┐
│  rpm-ostree compose tree            │
│  --repo=/srv/ostree/rpm-ostree/...  │
│  kinoite.json                       │
└──────┬──────────────────────────────┘
       │
       │ 2. Downloads packages from Fedora repos
       │    Composes OSTree commit
       ↓
┌─────────────────────────────────────┐
│  /srv/ostree/rpm-ostree/kinoite/    │
│  ├── objects/                       │
│  │   └── [commit objects]           │
│  └── refs/                          │
│      └── fedora/x86_64/kinoite/...  │
└──────┬──────────────────────────────┘
       │
       │ 3. Update summary
       ↓
┌─────────────────────────────────────┐
│  ostree summary -u                  │
└──────┬──────────────────────────────┘
       │
       │ 4. Available via HTTP
       ↓
┌─────────────────────────────────────┐
│  http://server:8080/repo/kinoite    │
└──────┬──────────────────────────────┘
       │
       │ 5. Clients can pull
       ↓
┌─────────────────────────────────────┐
│  Kinoite Workstation                │
│  rpm-ostree rebase                  │
└─────────────────────────────────────┘
```

## Ansible Playbook Execution Flow

```
┌────────────────────────────────────────────────┐
│  ansible-playbook setup-rpm-ostree-repo.yml    │
└────────────┬───────────────────────────────────┘
             │
             ├─→ [rpm_ostree_repo role]
             │   ├── Create /srv/ostree/rpm-ostree/kinoite
             │   ├── Initialize OSTree repository
             │   └── Set permissions
             │
             ├─→ [rpm_ostree_selinux role]
             │   ├── Check SELinux status
             │   ├── Set httpd_sys_content_t context
             │   └── Apply with restorecon
             │
             └─→ [rpm_ostree_apache role]
                 ├── Verify Apache is running
                 ├── Deploy rpm-ostree-repo.conf
                 │   └── Alias /repo/kinoite → /srv/ostree/...
                 ├── Test Apache config
                 └── Reload Apache
```

## Network Topology

```
                    Internet
                       │
                       │
        ┌──────────────┼──────────────┐
        │              │              │
        │     Internal Network        │
        │                             │
   ┌────┴─────┐              ┌───────┴────────┐
   │  Server  │              │  Kinoite Clients│
   │  :8080   │◄─────────────┤                 │
   │          │  HTTP Pulls  │  - Workstation 1│
   │ Flatpak  │              │  - Workstation 2│
   │ rpm-ostree│             │  - VM Test Env  │
   └──────────┘              └─────────────────┘
```

## Security Model

```
┌─────────────────────────────────────────────┐
│            Security Layers                  │
│                                             │
│  1. Firewall (from Flatpak setup)          │
│     └─ Port 8080 allowed                   │
│                                             │
│  2. SELinux                                 │
│     └─ httpd_sys_content_t on repos        │
│                                             │
│  3. Apache Permissions                      │
│     └─ Require all granted                 │
│     └─ CORS headers enabled                │
│                                             │
│  4. OSTree Security (optional)              │
│     └─ GPG signing (configure manually)    │
│                                             │
└─────────────────────────────────────────────┘
```

## Scalability

### Horizontal Scaling

```
        Load Balancer
              │
    ┌─────────┼─────────┐
    │         │         │
┌───┴───┐ ┌───┴───┐ ┌───┴───┐
│Server1│ │Server2│ │Server3│
│ :8080 │ │ :8080 │ │ :8080 │
└───────┘ └───────┘ └───────┘
    │         │         │
    └─────────┼─────────┘
         Shared Storage
         (NFS, Ceph, etc.)
              │
        /srv/ostree/...
```

### Content Delivery Network (CDN)

```
┌──────────────┐
│ Origin Server│
│ (Your Server)│
└──────┬───────┘
       │ rsync/sync
       ↓
┌──────────────┐
│  CDN Edge    │
│  Nodes       │
└──────┬───────┘
       │
       ↓
┌──────────────┐
│   Clients    │
└──────────────┘
```

## Directory Permissions

```
/srv/ostree/rpm-ostree/
├── [drwxr-xr-x root root] kinoite/
    ├── [drwxr-xr-x root root] objects/
    ├── [drwxr-xr-x root root] refs/
    ├── [drwxr-xr-x root root] state/
    ├── [-rw-r--r-- root root] config
    └── [-rw-r--r-- root root] summary
```

SELinux context: `httpd_sys_content_t`

## Integration Points

### With Existing Flatpak Infrastructure

```
Shared Components:
├── Apache HTTP Server ✓
├── Port 8080          ✓
├── SELinux policies   ✓
├── Firewall rules     ✓
└── Monitoring (if any)✓

Separate Components:
├── Repository paths   ✗
├── OSTree refs        ✗
├── Client tools       ✗
└── Content type       ✗
```

## Maintenance Workflow

```
Regular Updates:
1. Compose new image
   └─ rpm-ostree compose tree
2. Update summary
   └─ ostree summary -u
3. Clients pull updates
   └─ rpm-ostree upgrade

Monitoring:
1. Apache logs
   └─ /var/log/httpd/*-access.log
2. Disk space
   └─ /srv/ostree/rpm-ostree
3. SELinux denials
   └─ ausearch -m avc
```
