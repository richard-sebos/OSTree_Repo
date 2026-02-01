# Article Additions - Client Rebase and Promotion Workflow

## Client Rebase Process Section

**Insert this after "Deploying Images" section (after line 122), before "Wrapping It Up"**

---

## Pointing Clients to Your Custom Repo

Now that the repo is up and running, you need to tell your Kinoite machines to actually use it. By default, they're still pulling updates from Fedora's official servers. Getting them to switch over is pretty straightforward.

On each Kinoite machine, you'll add your custom repo as a new remote and then rebase to it:

```bash
sudo ostree remote add --no-gpg-verify kinoite-prod \
  https://kinoite.sebostech.local:8443/repo/kinoite/prod

sudo rpm-ostree rebase kinoite-prod:fedora/x86_64/kinoite/stable
sudo reboot
```

The `--no-gpg-verify` flag is there because we're using self-signed certificates. In a production environment, you'd want to set up proper GPG signing and distribute the public key, but for an internal setup, this works fine.

After the reboot, the machine is running your custom image. From that point on, whenever you run `rpm-ostree upgrade`, it checks *your* repo instead of Fedora's. That's it—now you're in full control of what goes out to users.

---

## Promotion Workflow Section

**Insert this after "Composing an Image" section (after line 110), before "Deploying Images"**

---

## Moving from Dev to Production

Once you've composed a new image in the dev repo and tested it for a week or so, you'll want to move it to production so the rest of the organization can get it.

The promotion process is pretty simple. You're basically copying the tested commit from the dev repo to the prod repo:

```bash
sudo ostree pull-local \
  /srv/ostree/rpm-ostree/dev \
  fedora/x86_64/kinoite/dev \
  --repo=/srv/ostree/rpm-ostree/prod

sudo ostree summary -u --repo=/srv/ostree/rpm-ostree/prod
```

What this does is pull the commit from dev into prod without re-downloading or re-composing anything. It's efficient and safe—you're deploying the *exact* image you already tested, not building a new one that might differ.

If you've automated this with Ansible (which I'd recommend), you can just run a playbook instead:

```bash
ansible-playbook -i inventory/hosts.yml playbooks/promote-to-prod.yml
```

Either way, once the prod repo is updated, any machines pointing to it will see the new image on their next upgrade check. You control the rollout—users don't get updates until you're ready.

---

## Suggested Article Flow

With these additions, the article sections would flow like this:

1. **Introduction** (Your story about Windows updates)
2. **What Is OSTree?**
3. **Updating with rpm-ostree**
4. **Setting Up the OSTree Repo**
5. **SELinux Considerations**
6. **Composing an Image**
7. **Moving from Dev to Production** ← NEW SECTION
8. **Deploying Images**
9. **Pointing Clients to Your Custom Repo** ← NEW SECTION
10. **Wrapping It Up**

This creates a logical progression:
- Build infrastructure → Compose images → Test and promote → Make available → Point clients → Summary

---

## Tone Notes

These sections match your existing casual, conversational style by:

- Using contractions ("you'll", "you're", "that's")
- Speaking directly to the reader ("you need to tell")
- Acknowledging practical shortcuts (`--no-gpg-verify`)
- Noting "what's better in production" without being preachy
- Focusing on "here's what works" rather than theory
- Keeping explanations brief and action-oriented
- Using phrases like "pretty straightforward" and "That's it"
