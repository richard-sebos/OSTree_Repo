# HTTPS Setup for rpm-ostree Repository

This playbook is configured to use HTTPS by default, reusing the SSL certificates from your Flatpak repository setup.

## Default Configuration

By default, HTTPS is **enabled** in `group_vars/rpm_ostree_repo_servers.yml`:

```yaml
rpm_ostree_enable_https: true
```

## SSL Certificate Reuse

The playbook automatically reuses your existing SSL certificates from the Flatpak setup:

```yaml
ssl_cert_file: /etc/pki/tls/certs/flatpak-repo.crt
ssl_key_file: /etc/pki/tls/private/flatpak-repo.key
```

### Prerequisites

Before running the playbook, ensure:

1. **Flatpak repository with HTTPS is already set up**
   - SSL certificates exist at the paths above
   - Apache is configured for HTTPS on port 8443

2. **Verify certificates exist:**
   ```bash
   ssh YOUR-SERVER "ls -l /etc/pki/tls/certs/flatpak-repo.crt"
   ssh YOUR-SERVER "ls -l /etc/pki/tls/private/flatpak-repo.key"
   ```

## How HTTPS Works

The rpm-ostree Apache configuration extends your existing HTTPS VirtualHost:

```
Existing (from Flatpak):
https://server:8443/repo/flatpak  → Flatpak repo

Added (by this playbook):
https://server:8443/repo/kinoite   → rpm-ostree repo
```

Both endpoints share:
- Same SSL certificate
- Same HTTPS port (8443)
- Same Apache VirtualHost

## Using HTTPS with Clients

### On Kinoite Workstations

Add the remote using HTTPS:

```bash
sudo ostree remote add --no-gpg-verify kinoite \
  https://YOUR-SERVER:8443/repo/kinoite
```

### Testing HTTPS Access

```bash
# Test with curl (use -k to skip certificate validation for self-signed certs)
curl -k https://YOUR-SERVER:8443/repo/kinoite/config

# Or verify the certificate
curl --cacert /path/to/ca-cert.pem https://YOUR-SERVER:8443/repo/kinoite/config
```

## Using Custom SSL Certificates

If you want to use different certificates than the Flatpak setup, edit `group_vars/rpm_ostree_repo_servers.yml`:

```yaml
ssl_cert_file: /etc/pki/tls/certs/your-custom.crt
ssl_key_file: /etc/pki/tls/private/your-custom.key
```

Make sure the certificates exist on the server before running the playbook.

## Disabling HTTPS (HTTP Only)

If you prefer HTTP only, set in `group_vars/rpm_ostree_repo_servers.yml`:

```yaml
rpm_ostree_enable_https: false
```

Then the repository will be available at:

```
http://server:8080/repo/kinoite
```

And clients should use:

```bash
sudo ostree remote add --no-gpg-verify kinoite \
  http://YOUR-SERVER:8080/repo/kinoite
```

## Security Recommendations

### For Production Use:

1. **Use Proper SSL Certificates**
   - Replace self-signed certificates with CA-signed ones
   - Use Let's Encrypt for free, valid certificates

2. **Enable GPG Signing**
   ```bash
   # Generate GPG key
   gpg --gen-key

   # Configure OSTree to sign commits
   sudo ostree config set --repo=/srv/ostree/rpm-ostree/kinoite \
     core.gpg-sign your-key-id
   ```

3. **Update Client Configuration**
   ```bash
   # Remove --no-gpg-verify flag
   sudo ostree remote add kinoite https://SERVER:8443/repo/kinoite

   # Import GPG key
   sudo ostree remote gpg-import kinoite --keyring=/path/to/pubkey.gpg
   ```

4. **Enforce HTTPS Only**
   - Ensure HTTP to HTTPS redirect is configured (already in Flatpak setup)
   - Block port 8080 externally, only allow 8443

## Troubleshooting HTTPS

### Certificate Not Found

**Error:** SSL certificate not found during playbook run

**Solution:**
1. Run the Flatpak HTTPS setup first:
   ```bash
   ansible-playbook -i ../ansible-flatpak-repo/inventory/hosts.yml \
     ../ansible-flatpak-repo/playbooks/setup-flatpak-repo.yml \
     --tags ssl
   ```

2. Or disable HTTPS in rpm-ostree setup:
   ```yaml
   rpm_ostree_enable_https: false
   ```

### SSL Handshake Errors on Client

**Error:** Certificate verification failed

**Solutions:**

1. **For self-signed certificates:**
   ```bash
   # Use --no-gpg-verify flag
   sudo ostree remote add --no-gpg-verify kinoite https://SERVER:8443/repo/kinoite
   ```

2. **For proper certificates:**
   ```bash
   # Ensure CA certificate is trusted
   sudo cp your-ca-cert.crt /etc/pki/ca-trust/source/anchors/
   sudo update-ca-trust
   ```

### Mixed Content Warnings

If you see mixed content warnings, ensure:

1. All repository URLs use `https://` (not `http://`)
2. HTTP to HTTPS redirect is working
3. No hardcoded HTTP URLs in treefiles or configs

## Port Configuration

By default:
- **HTTP:** Port 8080 (redirects to HTTPS if redirect is configured)
- **HTTPS:** Port 8443

These ports are inherited from your Flatpak setup. To verify:

```bash
ssh YOUR-SERVER "netstat -tlnp | grep httpd"
```

## Apache Configuration Details

### HTTPS Template

The HTTPS template (`rpm-ostree-repo-https.conf.j2`) creates:

```apache
<VirtualHost *:8443>
    ServerName your-server-name

    SSLEngine on
    SSLCertificateFile /etc/pki/tls/certs/flatpak-repo.crt
    SSLCertificateKeyFile /etc/pki/tls/private/flatpak-repo.key

    Alias /repo/kinoite /srv/ostree/rpm-ostree/kinoite

    <Directory /srv/ostree/rpm-ostree/kinoite>
        Options Indexes FollowSymLinks
        AllowOverride None
        Require all granted
        Header set Access-Control-Allow-Origin "*"
    </Directory>
</VirtualHost>
```

### Verifying Apache Configuration

```bash
# Test Apache config
sudo apachectl configtest

# Check loaded VirtualHosts
sudo apachectl -S

# View rpm-ostree Apache config
sudo cat /etc/httpd/conf.d/rpm-ostree-repo.conf
```

## Complete Example: HTTPS Setup

```bash
# 1. Ensure Flatpak HTTPS is set up
ansible-playbook -i ../ansible-flatpak-repo/inventory/hosts.yml \
  ../ansible-flatpak-repo/playbooks/setup-flatpak-repo.yml

# 2. Run rpm-ostree setup (HTTPS enabled by default)
ansible-playbook -i inventory/hosts.yml \
  playbooks/setup-rpm-ostree-repo.yml

# 3. Verify HTTPS works
curl -k https://YOUR-SERVER:8443/repo/kinoite/config

# 4. Configure client
sudo ostree remote add --no-gpg-verify kinoite \
  https://YOUR-SERVER:8443/repo/kinoite

# 5. Pull and rebase
sudo rpm-ostree rebase kinoite:fedora/x86_64/kinoite/stable
```

## Firewall Considerations

Ensure port 8443 is open (should already be from Flatpak setup):

```bash
sudo firewall-cmd --list-ports
# Should show: 8080/tcp 8443/tcp

# If not, add it:
sudo firewall-cmd --permanent --add-port=8443/tcp
sudo firewall-cmd --reload
```

## Summary

- ✅ **HTTPS is enabled by default**
- ✅ **Reuses Flatpak SSL certificates automatically**
- ✅ **No additional certificate management needed**
- ✅ **Works alongside Flatpak HTTPS on same port**
- ✅ **Can be disabled if HTTP-only is preferred**
