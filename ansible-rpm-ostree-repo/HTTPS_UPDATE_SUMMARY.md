# HTTPS Support - Update Summary

## What Changed (Version 2.0.0)

The ansible-rpm-ostree-repo playbook now has **HTTPS enabled by default**!

### Key Updates

✅ **HTTPS Enabled by Default**
- The playbook now configures HTTPS automatically
- Reuses your existing SSL certificates from the Flatpak setup
- No additional certificate management required

✅ **Dual Protocol Support**
- Both HTTP (port 8080) and HTTPS (port 8443) work
- Clients can use either protocol
- HTTPS is recommended for security

✅ **Certificate Reuse**
- Automatically uses: `/etc/pki/tls/certs/flatpak-repo.crt`
- And: `/etc/pki/tls/private/flatpak-repo.key`
- Same certificates as your Flatpak repository

## Quick Start with HTTPS

### 1. Verify Prerequisites

Ensure your Flatpak setup has HTTPS configured:

```bash
ssh YOUR-SERVER "ls -l /etc/pki/tls/certs/flatpak-repo.crt"
ssh YOUR-SERVER "ls -l /etc/pki/tls/private/flatpak-repo.key"
```

If certificates don't exist, either:
- Run the Flatpak HTTPS setup first, OR
- Set `rpm_ostree_enable_https: false` in `group_vars/rpm_ostree_repo_servers.yml`

### 2. Run the Playbook

```bash
cd ansible-rpm-ostree-repo
ansible-playbook -i inventory/hosts.yml playbooks/setup-rpm-ostree-repo.yml
```

The playbook will automatically:
- Check for SSL certificates
- Configure Apache for HTTPS
- Set up both HTTP and HTTPS endpoints

### 3. Verify HTTPS Works

```bash
# Test HTTPS endpoint
curl -k https://YOUR-SERVER:8443/repo/kinoite/config

# Test HTTP endpoint (still works)
curl http://YOUR-SERVER:8080/repo/kinoite/config
```

### 4. Configure Clients with HTTPS

```bash
# On Kinoite workstation
sudo ostree remote add --no-gpg-verify kinoite \
  https://YOUR-SERVER:8443/repo/kinoite

sudo rpm-ostree rebase kinoite:fedora/x86_64/kinoite/stable
```

## Configuration

### Default Configuration (HTTPS Enabled)

In `group_vars/rpm_ostree_repo_servers.yml`:

```yaml
rpm_ostree_enable_https: true  # HTTPS enabled

# SSL certificates (reused from Flatpak)
ssl_cert_file: /etc/pki/tls/certs/flatpak-repo.crt
ssl_key_file: /etc/pki/tls/private/flatpak-repo.key
```

### HTTP-Only Configuration

If you prefer HTTP only:

```yaml
rpm_ostree_enable_https: false  # Disable HTTPS
```

### Custom Certificates

To use different SSL certificates:

```yaml
rpm_ostree_enable_https: true

ssl_cert_file: /path/to/your/certificate.crt
ssl_key_file: /path/to/your/private.key
```

## What's Available

### HTTP and HTTPS Endpoints

Both protocols work simultaneously:

```
HTTP:   http://server:8080/repo/kinoite
HTTPS:  https://server:8443/repo/kinoite
```

### Apache Configuration

Two templates are available:

1. **rpm-ostree-repo-http.conf.j2** - HTTP only (when `rpm_ostree_enable_https: false`)
2. **rpm-ostree-repo-https.conf.j2** - HTTPS with SSL (when `rpm_ostree_enable_https: true`)

The playbook automatically selects the correct template based on your configuration.

## Files Modified/Added

### New Files:
- `roles/rpm_ostree_apache/templates/rpm-ostree-repo-https.conf.j2` - HTTPS Apache config
- `HTTPS_SETUP.md` - Complete HTTPS documentation
- `HTTPS_UPDATE_SUMMARY.md` - This file

### Updated Files:
- `roles/rpm_ostree_apache/defaults/main.yml` - Added HTTPS variables
- `roles/rpm_ostree_apache/tasks/main.yml` - Added HTTPS configuration logic
- `roles/rpm_ostree_apache/templates/rpm-ostree-repo.conf.j2` → **renamed to** `rpm-ostree-repo-http.conf.j2`
- `group_vars/rpm_ostree_repo_servers.yml` - Added HTTPS settings
- `playbooks/setup-rpm-ostree-repo.yml` - Updated output messages
- `README.md` - Updated with HTTPS information
- `QUICKSTART.md` - Updated examples to use HTTPS
- `DEPLOYMENT_CHECKLIST.md` - Added HTTPS verification steps
- `IMPLEMENTATION_SUMMARY.md` - Reflected HTTPS changes

## Migration from Version 1.0.0

If you already deployed version 1.0.0 (HTTP only):

### Option 1: Upgrade to HTTPS (Recommended)

1. Ensure SSL certificates exist on the server
2. Pull the latest playbook version
3. Run the playbook again:
   ```bash
   ansible-playbook -i inventory/hosts.yml playbooks/setup-rpm-ostree-repo.yml
   ```
4. The playbook will detect the change and configure HTTPS
5. Apache will be reloaded (not restarted)
6. Update clients to use HTTPS URLs

### Option 2: Stay on HTTP Only

If you prefer to keep HTTP only:

1. Edit `group_vars/rpm_ostree_repo_servers.yml`
2. Set: `rpm_ostree_enable_https: false`
3. No need to re-run the playbook (it's already configured for HTTP)

## Benefits of HTTPS

✅ **Security**: Encrypted communication between clients and server
✅ **Integrity**: Prevents man-in-the-middle attacks
✅ **Trust**: Better security posture for enterprise environments
✅ **Compliance**: Meets security requirements for sensitive environments

## Common Issues

### "SSL certificate not found"

**Problem**: Playbook fails with SSL certificate not found error

**Solution**:
1. Run Flatpak HTTPS setup first to create certificates, OR
2. Set `rpm_ostree_enable_https: false` to use HTTP only

### "Certificate verification failed" on client

**Problem**: Clients can't verify SSL certificate

**Solution**:
For self-signed certificates, use `--no-gpg-verify`:
```bash
sudo ostree remote add --no-gpg-verify kinoite https://SERVER:8443/repo/kinoite
```

For production, use CA-signed certificates.

### Both HTTP and HTTPS work, which should I use?

**Recommendation**: Use HTTPS for production. HTTP is available as a fallback but HTTPS provides better security.

## Version History

- **Version 1.0.0** (2026-01-25): Initial release with HTTP support
- **Version 2.0.0** (2026-01-26): Added HTTPS support with automatic certificate reuse

## Documentation

For complete HTTPS documentation, see:
- **HTTPS_SETUP.md** - Detailed HTTPS configuration guide
- **README.md** - General setup and usage
- **QUICKSTART.md** - Fast 5-minute setup
- **DEPLOYMENT_CHECKLIST.md** - Verification steps

---

**Ready to deploy with HTTPS!** 🔒
