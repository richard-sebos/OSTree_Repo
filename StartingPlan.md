## ✅ Updated Strategy for Part 1

### Goal:

Use your **existing Flatpak OSTree server infrastructure (Apache + Ansible)** to **add rpm-OSTree support** for Kinoite system image distribution.

---

## 🧩 Minimal Changes Needed

### 🔁 Shared OSTree Concepts

* Both Flatpak and rpm-OSTree use the same **OSTree format**
* Apache config, SELinux policies (`httpd_sys_content_t`), and firewall rules can be reused

---

## 🛠️ Action Plan: Extending Your Setup

### 1. **Create a Separate Directory for rpm-OSTree**

To keep your **Flatpak and OS image content cleanly separated**, do:

```bash
sudo mkdir -p /srv/ostree/rpm-ostree/kinoite
sudo ostree --repo=/srv/ostree/rpm-ostree/kinoite init --mode=archive-z2
```

Make sure SELinux is configured:

```bash
semanage fcontext -a -t httpd_sys_content_t "/srv/ostree/rpm-ostree(/.*)?"
restorecon -Rv /srv/ostree/rpm-ostree
```

---

### 2. **Extend Apache to Serve This Repo**

In your Apache vhost config (likely already used for Flatpak):

```apache
# /etc/httpd/conf.d/ostree-repos.conf

Alias /repo/kinoite /srv/ostree/rpm-ostree/kinoite

<Directory /srv/ostree/rpm-ostree/kinoite>
    Options Indexes FollowSymLinks
    AllowOverride None
    Require all granted
</Directory>
```

> ⚠️ Flatpak might already use `/repo/flatpak`. You can mirror that convention.

Then reload:

```bash
sudo systemctl reload httpd
```

Test:

```bash
curl http://yourserver/repo/kinoite/summary
```

---

### 3. **Compose a Custom Kinoite Tree**

You’ll need a `kinoite.json` treefile. Here’s a basic example:

```json
{
  "ref": "fedora/x86_64/kinoite/stable",
  "repos": ["fedora", "fedora-updates"],
  "selinux": true,
  "packages": [
    "fedora-release-kinoite",
    "gnome-terminal",
    "vim-enhanced"
  ]
}
```

Build it:

```bash
sudo rpm-ostree compose tree \
  --repo=/srv/ostree/rpm-ostree/kinoite \
  kinoite.json
```

Then:

```bash
ostree --repo=/srv/ostree/rpm-ostree/kinoite summary -u
```

---


### 5. **Test with Kinoite VM**

On a Kinoite client:

```bash
rpm-ostree rebase kinoite-remote:fedora/x86_64/kinoite/stable
```

Where `kinoite-remote` is defined as:

```bash
ostree remote add --no-gpg-verify kinoite-remote http://yourserver/repo/kinoite
```

