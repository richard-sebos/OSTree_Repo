# Understanding SELinux httpd_sys_content_t Context

## What is httpd_sys_content_t?

The `httpd_sys_content_t` SELinux type is a security label that tells the SELinux security system: **"Apache (httpd) is allowed to READ files with this label"**.

It provides an additional layer of security beyond traditional Unix file permissions, ensuring Apache can only access specific files it's supposed to serve, and nothing else.

## SELinux Protection Model

SELinux implements **Mandatory Access Control (MAC)** - even if file permissions allow access, SELinux policies must also permit it.

### Traditional Linux Security (Permissions Only)

```
User/Process tries to access file
         ↓
Check file permissions (rwx)
         ↓
   Allowed? → Access granted
   Denied?  → Access denied
```

**Problem**: If permissions are misconfigured, or process is compromised, attacker has full access.

### Linux with SELinux (Defense in Depth)

```
User/Process tries to access file
         ↓
Check file permissions (rwx)
         ↓
   Denied? → Access denied
   Allowed? ↓
         ↓
Check SELinux policy
         ↓
   Policy allows? → Access granted
   Policy denies? → Access denied (even if file permissions allow!)
```

**Benefit**: Two independent security checks. Both must pass for access.

## Why httpd_sys_content_t Matters

### Scenario 1: Wrong SELinux Context (Blocks Apache)

```bash
# Apache has correct file permissions (755)
$ ls -l /srv/ostree/rpm-ostree/
drwxr-xr-x. root root /srv/ostree/rpm-ostree/dev

# But WRONG SELinux context
$ ls -Z /srv/ostree/rpm-ostree/
unconfined_u:object_r:default_t:s0 dev

# Apache tries to serve files
# Result: PERMISSION DENIED ✗
```

**Apache error log**:
```
[error] [client 192.168.35.100] (13)Permission denied:
AH00132: file permissions deny server access:
/srv/ostree/rpm-ostree/dev/config
```

**What happened**:
- Unix permissions say "yes" (755 = read for everyone)
- SELinux policy says "no" (httpd cannot read default_t)
- SELinux wins → Access denied

### Scenario 2: Correct SELinux Context (Allows Apache)

```bash
# Apache has correct file permissions (755)
$ ls -l /srv/ostree/rpm-ostree/
drwxr-xr-x. root root /srv/ostree/rpm-ostree/dev

# And CORRECT SELinux context
$ ls -Z /srv/ostree/rpm-ostree/
unconfined_u:object_r:httpd_sys_content_t:s0 dev

# Apache tries to serve files
# Result: SUCCESS! ✓
```

**What happened**:
- Unix permissions say "yes" (755)
- SELinux policy says "yes" (httpd CAN read httpd_sys_content_t)
- Both agree → Access granted

## SELinux Context Structure

The full SELinux context: `unconfined_u:object_r:httpd_sys_content_t:s0`

```
unconfined_u : object_r : httpd_sys_content_t : s0
     ↓            ↓              ↓               ↓
   User         Role           Type           Level
```

### Breaking Down Each Field

| Field | Value | Purpose | Importance for Access Control |
|-------|-------|---------|-------------------------------|
| **User** | `unconfined_u` | SELinux user identity | Low - rarely used for access decisions |
| **Role** | `object_r` | Role for objects (files) | Low - standard for files |
| **Type** | `httpd_sys_content_t` | **The security label** | **HIGH - This controls access!** |
| **Level** | `s0` | Multi-Level Security level | Low - not used in targeted policy |

### The Critical Part: Type

The **Type** field (`httpd_sys_content_t`) is what SELinux uses for access control decisions.

## Common SELinux Types for Apache

| Type | What Apache Can Do | Use Case |
|------|-------------------|----------|
| `httpd_sys_content_t` | ✅ **Read files** (serve via HTTP/HTTPS) | Static web content, repositories |
| `httpd_sys_rw_content_t` | ✅ Read ✅ **Write files** | Upload directories, cache |
| `httpd_sys_script_exec_t` | ✅ Read ✅ **Execute** | CGI scripts, binaries |
| `httpd_log_t` | ✅ **Write logs** | Apache log files |
| `httpd_config_t` | ✅ **Read config** | Apache configuration files |
| `default_t` | ❌ **Blocked** | Generic type, no Apache access |
| `user_home_t` | ❌ **Blocked** | User home directories (protected) |
| `etc_t` | ❌ **Blocked** | System configuration (protected) |
| `passwd_file_t` | ❌ **Blocked** | Password files (highly protected) |
| `mysqld_db_t` | ❌ **Blocked** | Database files (protected) |

## What httpd_sys_content_t Allows and Denies

### Allowed Operations by Apache

With `httpd_sys_content_t`, Apache (httpd process) can:

```bash
✅ Read files
✅ List directories
✅ Follow symlinks
✅ Stat files (check metadata)
✅ Serve content over HTTP/HTTPS
✅ Send files to clients
```

### Denied Operations by Apache

With `httpd_sys_content_t`, Apache (httpd process) CANNOT:

```bash
❌ Write to files
❌ Create new files
❌ Delete files
❌ Modify files
❌ Execute files as programs
❌ Change permissions
❌ Change ownership
```

### Why This Matters

**Read-only access** means:
- Apache can serve your OS images to clients
- Apache **cannot** corrupt or delete your OS images
- Even if Apache is compromised, repository integrity protected

## Real-World Attack Protection Examples

### Example 1: Directory Traversal Attack Prevention

**Attack Scenario**:
```
Attacker crafts malicious URL:
https://server:8443/../../../../../../etc/passwd
```

**Without SELinux**:
```bash
# If Apache has path traversal vulnerability
# Could expose /etc/passwd
$ curl https://server:8443/../../../../../../etc/passwd
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
...
# EXPOSED! ✗
```

**With SELinux Enforcing**:
```bash
# Check /etc/passwd context
$ ls -Z /etc/passwd
system_u:object_r:passwd_file_t:s0 /etc/passwd

# Apache tries to access via path traversal
# SELinux policy check:
#   - Apache runs as httpd_t
#   - /etc/passwd is passwd_file_t
#   - Policy: httpd_t CANNOT read passwd_file_t
# Result: BLOCKED by SELinux ✓

$ curl https://server:8443/../../../../../../etc/passwd
403 Forbidden
```

**Protection**: Even if Apache has bug allowing path traversal, SELinux blocks access to system files.

### Example 2: Web Shell Upload Prevention

**Attack Scenario**:
Attacker exploits vulnerability to upload malicious PHP file

**Without SELinux**:
```bash
# Attacker uploads evil.php to /srv/ostree/rpm-ostree/
# evil.php contains: <?php system($_GET['cmd']); ?>

# Attacker accesses:
https://server:8443/evil.php?cmd=cat%20/etc/shadow

# If Apache executes PHP in this directory:
# Result: System compromised ✗
```

**With SELinux + httpd_sys_content_t**:
```bash
# Even if attacker uploads file
$ ls -Z /srv/ostree/rpm-ostree/dev/evil.php
unconfined_u:object_r:httpd_sys_content_t:s0 evil.php

# Apache tries to execute evil.php
# SELinux policy check:
#   - httpd_sys_content_t is NOT executable type
#   - Policy: httpd cannot execute httpd_sys_content_t
# Result: BLOCKED by SELinux ✓

# File served as plain text, not executed
```

**Protection**: Files labeled `httpd_sys_content_t` cannot be executed, only read/served.

### Example 3: Containing Apache Compromise

**Attack Scenario**:
Zero-day vulnerability gives attacker shell access via Apache

**Without SELinux**:
```bash
# Attacker has shell through compromised Apache
# Tries to steal database credentials
$ cat /var/lib/mysql/passwords.txt
db_user=admin
db_pass=SuperSecret123
# Database compromised! ✗

# Tries to read private SSH keys
$ cat /root/.ssh/id_rsa
-----BEGIN RSA PRIVATE KEY-----
MIIEpAIBAAKCAQEA...
# SSH keys stolen! ✗

# Full system compromise
```

**With SELinux Enforcing**:
```bash
# Attacker has shell through compromised Apache
# But Apache runs as httpd_t

# Tries to read database credentials
$ cat /var/lib/mysql/passwords.txt
cat: /var/lib/mysql/passwords.txt: Permission denied

# Check why
$ ls -Z /var/lib/mysql/passwords.txt
system_u:object_r:mysqld_db_t:s0 passwords.txt

# SELinux policy: httpd_t CANNOT read mysqld_db_t
# BLOCKED! ✓

# Tries to read SSH keys
$ cat /root/.ssh/id_rsa
cat: /root/.ssh/id_rsa: Permission denied

# Check why
$ ls -Z /root/.ssh/id_rsa
system_u:object_r:ssh_home_t:s0 id_rsa

# SELinux policy: httpd_t CANNOT read ssh_home_t
# BLOCKED! ✓

# Attacker confined to reading only httpd_sys_content_t files
# Damage limited to web content directory only
```

**Protection**: Compromised Apache process confined by SELinux policy. Can only access files explicitly allowed.

### Example 4: Repository Modification Prevention

**Attack Scenario**:
Attacker tries to corrupt your OS repository

**Without SELinux**:
```bash
# If Apache file permissions misconfigured (writable)
# Attacker uses Apache to modify repository

# Corrupts OS image
$ echo "malicious code" > /srv/ostree/rpm-ostree/prod/config
# Repository corrupted! ✗

# Deletes commits
$ rm -rf /srv/ostree/rpm-ostree/prod/objects/*
# All OS images deleted! ✗
```

**With SELinux + httpd_sys_content_t**:
```bash
# Attacker tries to modify repository
$ echo "malicious code" > /srv/ostree/rpm-ostree/prod/config
bash: /srv/ostree/rpm-ostree/prod/config: Permission denied

# SELinux policy check:
#   - Apache (httpd_t) can READ httpd_sys_content_t
#   - Apache CANNOT WRITE httpd_sys_content_t
# BLOCKED! ✓

# Tries to delete commits
$ rm /srv/ostree/rpm-ostree/prod/objects/a3/f8b2c4...
rm: cannot remove: Permission denied

# BLOCKED! ✓

# Repository integrity maintained
```

**Protection**: Even with compromised Apache, repository remains read-only. Cannot be corrupted or deleted.

## SELinux Policy Rules for httpd_sys_content_t

### The Actual Policy (Simplified)

```
# Apache can read httpd_sys_content_t files
allow httpd_t httpd_sys_content_t:file { read open getattr };

# Apache can read/search httpd_sys_content_t directories
allow httpd_t httpd_sys_content_t:dir { read search getattr };

# Apache CANNOT write httpd_sys_content_t
# (no write rule = denied by default)

# Apache CANNOT read other types (default deny)
# These denials are implicit (not explicitly written):
deny httpd_t passwd_file_t:file { read };        # /etc/passwd
deny httpd_t shadow_t:file { read };             # /etc/shadow
deny httpd_t mysqld_db_t:file { read };          # Database files
deny httpd_t ssh_home_t:file { read };           # SSH keys
deny httpd_t user_home_t:file { read };          # Home directories
```

### SELinux Default Deny Model

**Important**: SELinux uses **default deny** - if there's no explicit allow rule, access is denied.

```
Policy exists: "allow httpd_t httpd_sys_content_t:file read"
  → Apache CAN read httpd_sys_content_t

No policy exists: "allow httpd_t passwd_file_t:file read"
  → Apache CANNOT read passwd_file_t (denied by default)
```

## How Your Setup Applies httpd_sys_content_t

### What the SELinux Role Does

From `roles/rpm_ostree_selinux/tasks/main.yml`:

#### Step 1: Create Persistent Policy

```yaml
- name: Configure SELinux file context for rpm-ostree repository
  community.general.sefcontext:
    target: '/srv/ostree/rpm-ostree(/.*)?'
    setype: httpd_sys_content_t
    state: present
```

**What this does**:
- Creates persistent SELinux policy rule
- Says: "Everything under `/srv/ostree/rpm-ostree` should be `httpd_sys_content_t`"
- Survives reboots
- Automatically labels new files created in this directory

**Stored in**: `/etc/selinux/targeted/contexts/files/file_contexts.local`

**View the rule**:
```bash
$ sudo semanage fcontext -l | grep rpm-ostree
/srv/ostree/rpm-ostree(/.*)?    all files    system_u:object_r:httpd_sys_content_t:s0
```

#### Step 2: Apply Labels to Existing Files

```yaml
- name: Apply SELinux context to rpm-ostree repository
  ansible.builtin.command:
    cmd: restorecon -Rv /srv/ostree/rpm-ostree
```

**What this does**:
- Applies the policy to all existing files
- Recursively labels everything under `/srv/ostree/rpm-ostree`
- Changes file contexts from `default_t` to `httpd_sys_content_t`

**Manual equivalent**:
```bash
sudo restorecon -Rv /srv/ostree/rpm-ostree
```

**Output**:
```
Relabeled /srv/ostree/rpm-ostree from unconfined_u:object_r:default_t:s0 to unconfined_u:object_r:httpd_sys_content_t:s0
Relabeled /srv/ostree/rpm-ostree/dev from unconfined_u:object_r:default_t:s0 to unconfined_u:object_r:httpd_sys_content_t:s0
Relabeled /srv/ostree/rpm-ostree/prod from unconfined_u:object_r:default_t:s0 to unconfined_u:object_r:httpd_sys_content_t:s0
...
```

#### Step 3: Verify Labels

```yaml
- name: Verify SELinux context is applied
  ansible.builtin.command:
    cmd: ls -Z /srv/ostree/rpm-ostree
  register: selinux_context
```

**Expected output**:
```
unconfined_u:object_r:httpd_sys_content_t:s0 dev
unconfined_u:object_r:httpd_sys_content_t:s0 prod
```

## Verification and Testing

### Check Current Context

```bash
$ ls -Z /srv/ostree/rpm-ostree/
unconfined_u:object_r:httpd_sys_content_t:s0 dev
unconfined_u:object_r:httpd_sys_content_t:s0 prod
```

**What to look for**: Type field should be `httpd_sys_content_t`

### Check Context Recursively

```bash
$ ls -lZR /srv/ostree/rpm-ostree/ | head -20
```

**All files should have**: `httpd_sys_content_t`

### Test Apache Access

```bash
# Test that Apache can access repository
$ curl -k https://192.168.35.35:8443/repo/kinoite/dev/config
[core]
repo_version=1
mode=archive-z2

# Success! ✓
```

### Test SELinux Is Enforcing

```bash
$ getenforce
Enforcing
```

**Should return**: `Enforcing` (not `Permissive` or `Disabled`)

### Check SELinux Status

```bash
$ sestatus
SELinux status:                 enabled
SELinuxfs mount:                /sys/fs/selinux
SELinux root directory:         /etc/selinux
Loaded policy name:             targeted
Current mode:                   enforcing
Mode from config file:          enforcing
Policy MLS status:              enabled
Policy deny_unknown status:     allowed
Memory protection checking:     actual (secure)
Max kernel policy version:      33
```

**Key fields**:
- `SELinux status: enabled`
- `Current mode: enforcing`

## SELinux Audit Logs

When SELinux blocks access, it logs the denial for troubleshooting.

### View Recent Denials

```bash
$ sudo ausearch -m avc -ts recent
```

### Example Denial Log Entry

```
type=AVC msg=audit(1642534567.890:1234): avc:  denied  { read }
  for pid=12345 comm="httpd" name="config"
  dev="sda1" ino=67890
  scontext=system_u:system_r:httpd_t:s0
  tcontext=unconfined_u:object_r:default_t:s0
  tclass=file permissive=0
```

### Decoding the Log Entry

| Field | Value | Meaning |
|-------|-------|---------|
| `denied { read }` | read | Apache tried to read a file |
| `comm="httpd"` | httpd | Command/process name (Apache) |
| `scontext=...httpd_t` | httpd_t | Source context (Apache process) |
| `tcontext=...default_t` | default_t | Target context (the file) |
| `tclass=file` | file | Object class (it's a file) |
| `permissive=0` | 0 | Enforcing mode (denial enforced) |

**Translation**:
Apache (httpd_t) tried to read a file labeled default_t, and SELinux denied it because there's no policy allowing httpd_t to read default_t.

### If You See This Denial

**Cause**: File has wrong SELinux context

**Fix**:
```bash
# Re-apply correct context
sudo restorecon -Rv /srv/ostree/rpm-ostree
```

## Troubleshooting SELinux Issues

### Problem: Apache Gets "Permission Denied"

**Symptoms**:
```bash
$ curl -k https://192.168.35.35:8443/repo/kinoite/dev/
403 Forbidden
```

**Check Apache error log**:
```bash
$ sudo tail /var/log/httpd/error_log
[error] (13)Permission denied: /srv/ostree/rpm-ostree/dev/config
```

**Diagnosis Steps**:

#### Step 1: Check File Permissions
```bash
$ ls -l /srv/ostree/rpm-ostree/dev/config
-rw-r--r--. 1 root root 43 Jan 31 10:00 config
```

**Should be**: At least `644` (readable by others)

#### Step 2: Check SELinux Context
```bash
$ ls -Z /srv/ostree/rpm-ostree/dev/config
unconfined_u:object_r:default_t:s0 config
```

**Problem**: Type is `default_t` (should be `httpd_sys_content_t`)

#### Step 3: Fix SELinux Context
```bash
$ sudo restorecon -v /srv/ostree/rpm-ostree/dev/config
Relabeled /srv/ostree/rpm-ostree/dev/config from unconfined_u:object_r:default_t:s0 to unconfined_u:object_r:httpd_sys_content_t:s0
```

#### Step 4: Verify Fix
```bash
$ curl -k https://192.168.35.35:8443/repo/kinoite/dev/config
[core]
repo_version=1
mode=archive-z2

# Success! ✓
```

### Problem: SELinux Labels Lost After File Copy

**Scenario**: You copy files and they lose SELinux context

```bash
# Copy file
$ sudo cp /tmp/newfile /srv/ostree/rpm-ostree/dev/

# Check context
$ ls -Z /srv/ostree/rpm-ostree/dev/newfile
unconfined_u:object_r:admin_home_t:s0 newfile

# Wrong context! (inherited from /tmp)
```

**Solution**: Always restore context after copying

```bash
$ sudo restorecon -v /srv/ostree/rpm-ostree/dev/newfile
Relabeled /srv/ostree/rpm-ostree/dev/newfile from ... to ...httpd_sys_content_t:s0
```

**Better**: Use `cp -Z` or `install` to preserve context

```bash
# Copy and preserve SELinux context
$ sudo cp --preserve=context /source/file /srv/ostree/rpm-ostree/dev/

# Or use install command
$ sudo install -Z -m 644 /source/file /srv/ostree/rpm-ostree/dev/
```

## Comparison: SELinux Modes

### Disabled Mode

```bash
$ sudo setenforce 0
$ sudo sed -i 's/SELINUX=enforcing/SELINUX=disabled/' /etc/selinux/config
$ reboot
```

**After reboot**:
```bash
$ getenforce
Disabled
```

**Security posture**:
```
Apache Process Power:
✅ Read any file (if Unix permissions allow)
✅ Write any file (if Unix permissions allow)
✅ Execute any file (if Unix permissions allow)
✅ Access database files
✅ Read /etc/shadow (if permissions misconfigured)
✅ Modify system files (if permissions misconfigured)

Protection Layers:
- Firewall ✓
- Unix permissions ✓
- SELinux ✗ (disabled)

Risk Level: HIGH
One misconfiguration = potential full compromise
```

### Permissive Mode

```bash
$ sudo setenforce 0
```

**Check mode**:
```bash
$ getenforce
Permissive
```

**Security posture**:
```
Apache Process Power:
✅ Read any file (SELinux logs but allows)
✅ Write any file (SELinux logs but allows)
✅ Execute any file (SELinux logs but allows)

Protection Layers:
- Firewall ✓
- Unix permissions ✓
- SELinux ⚠️ (logs violations but doesn't block)

Risk Level: MEDIUM-HIGH
Useful for troubleshooting, but not for production

Benefit: Can identify what SELinux would block without breaking service
```

**Use case**: Troubleshooting SELinux denials

```bash
# Enable permissive mode temporarily
$ sudo setenforce 0

# Test Apache access
$ curl -k https://192.168.35.35:8443/repo/kinoite/dev/

# Check what would have been blocked
$ sudo ausearch -m avc -ts recent

# Fix SELinux contexts based on denials
$ sudo restorecon -Rv /srv/ostree/rpm-ostree

# Re-enable enforcing
$ sudo setenforce 1
```

### Enforcing Mode (Recommended)

```bash
$ sudo setenforce 1
```

**Check mode**:
```bash
$ getenforce
Enforcing
```

**Security posture**:
```
Apache Process Power:
✅ Read httpd_sys_content_t files ONLY
❌ Write httpd_sys_content_t files (blocked)
❌ Access database files (blocked)
❌ Read /etc/shadow (blocked)
❌ Modify system files (blocked)
❌ Read other types (blocked)

Protection Layers:
- Firewall ✓
- Unix permissions ✓
- SELinux ✓ (actively enforcing)

Risk Level: LOW
Misconfiguration contained by SELinux policy
Even compromised Apache limited in damage

Benefit: Defense in depth, privilege separation
```

## Benefits of httpd_sys_content_t

### 1. Principle of Least Privilege

**Concept**: Process should have minimum access needed to function

**Implementation**:
```
Apache needs to:
✅ Read web content → httpd_sys_content_t allows read

Apache does NOT need to:
❌ Write to repository → httpd_sys_content_t denies write
❌ Execute files → httpd_sys_content_t denies execute
❌ Read database files → Policy denies access to mysqld_db_t
❌ Read SSH keys → Policy denies access to ssh_home_t
```

**Result**: Apache has exactly the access it needs, nothing more.

### 2. Defense in Depth

**Multiple Security Layers**:

```
Layer 1: Network Firewall
├─ Blocks unauthorized ports
└─ Allows: 8080, 8443

Layer 2: Apache Configuration
├─ Virtual host restrictions
├─ Directory access controls
└─ Allows: /repo/kinoite/* only

Layer 3: Unix File Permissions
├─ User/Group/Other (rwx)
└─ Allows: Owner=rw, Group=r, Other=r

Layer 4: SELinux (httpd_sys_content_t)
├─ Type enforcement
├─ Mandatory Access Control
└─ Allows: Apache read httpd_sys_content_t ONLY

Result: Attacker must bypass ALL layers to compromise system
```

### 3. Damage Containment

**If Apache is compromised**:

**Without SELinux**:
```
Compromised Apache Process:
└─ Can access anything Unix permissions allow
    ├─ Read database files
    ├─ Read SSH keys
    ├─ Read user files
    ├─ Modify web content
    └─ Potentially escalate privileges

Impact: Full system compromise possible
```

**With SELinux + httpd_sys_content_t**:
```
Compromised Apache Process:
└─ Confined by SELinux policy
    ├─ Can read web content ONLY
    ├─ CANNOT read database files (blocked by policy)
    ├─ CANNOT read SSH keys (blocked by policy)
    ├─ CANNOT read user files (blocked by policy)
    ├─ CANNOT modify web content (read-only)
    └─ CANNOT escalate privileges (blocked by policy)

Impact: Damage limited to reading existing web content
        System compromise prevented
```

### 4. Prevents Configuration Errors

**Scenario**: Administrator accidentally makes repository writable

```bash
# Oops! Made repository writable by Apache
$ sudo chown -R apache:apache /srv/ostree/rpm-ostree
$ sudo chmod -R 755 /srv/ostree/rpm-ostree
```

**Without SELinux**:
```
Result: Apache can now modify/delete repository
        One wrong configuration = data loss risk
```

**With SELinux**:
```
Result: Unix permissions allow write
        BUT SELinux policy still denies write to httpd_sys_content_t
        Repository protected despite misconfiguration
```

**SELinux saves you from yourself!**

### 5. Audit Trail

**SELinux logs all denials**:

```bash
$ sudo ausearch -m avc -ts today

# Shows:
# - What Apache tried to access
# - When it tried
# - Why it was blocked
# - Source and target contexts
```

**Benefits**:
- Detect intrusion attempts
- Identify misconfigurations
- Forensic analysis after incident
- Compliance audit requirements

## Summary

### What httpd_sys_content_t Provides

| Protection | How It Works | Benefit |
|------------|--------------|---------|
| **Read-only access** | Apache can read but not write/execute | Repository integrity protected |
| **Type enforcement** | Apache confined to specific file types | Cannot access system files |
| **Privilege limitation** | Apache cannot escalate beyond web content | Prevents lateral movement |
| **Breach containment** | Compromised Apache limited to web serving | Damage minimization |
| **Defense in depth** | Extra layer beyond file permissions | Multiple security barriers |
| **Audit capability** | All denials logged | Intrusion detection and compliance |

### Security Comparison

| Scenario | Unix Permissions Only | Unix Permissions + SELinux |
|----------|----------------------|---------------------------|
| **Normal operation** | ✅ Works | ✅ Works |
| **Misconfigured permissions** | ✗ Vulnerability exposed | ✓ SELinux prevents exploit |
| **Apache compromised** | ✗ Full system access possible | ✓ Confined to web content |
| **Path traversal bug** | ✗ Can read system files | ✓ SELinux blocks access |
| **Malicious file upload** | ✗ Can execute uploaded files | ✓ SELinux prevents execution |
| **Database access attempt** | ✗ Depends on file permissions | ✓ SELinux always blocks |

### In One Sentence

**`httpd_sys_content_t` tells SELinux: "Apache may READ these files to serve via web, but cannot write/execute/modify them, and cannot access ANY other files on the system"**.

### Why It Matters for Your rpm-ostree Repository

```
Your Repository Context:
├─ Location: /srv/ostree/rpm-ostree/
├─ Content: Operating system images (highly sensitive)
├─ Access: Must be served to clients via Apache
└─ Protection: Must never be corrupted or deleted

With httpd_sys_content_t:
✅ Apache serves OS images to authorized clients
✅ Apache CANNOT modify repository contents
✅ Apache CANNOT delete commits
✅ Apache CANNOT access other system services
✅ Compromised Apache limited to read-only serving
✅ Repository integrity guaranteed by SELinux

Without httpd_sys_content_t:
❌ Apache might not be able to serve files (wrong context)
❌ Apache might have too much access (misconfigured permissions)
❌ Compromised Apache could corrupt repository
❌ No containment if breach occurs
```

### Production Recommendation

**Always run SELinux in Enforcing mode with proper contexts**:

```bash
# Check current mode
$ getenforce
Enforcing

# Verify repository context
$ ls -Z /srv/ostree/rpm-ostree/
... httpd_sys_content_t ...

# If not enforcing, enable it
$ sudo setenforce 1
$ sudo sed -i 's/SELINUX=.*/SELINUX=enforcing/' /etc/selinux/config
```

**SELinux is not optional for production security** - it's a critical defense layer that protects your infrastructure even when other security measures fail.

---

**Bottom Line**: `httpd_sys_content_t` is a security guardrail that ensures even if something goes wrong with Apache, the damage is contained to read-only access of web content. Your OS images can be served but never modified or deleted through Apache, protecting your repository integrity and preventing system-wide compromise.
