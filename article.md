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
- Migrating dev images to production allows for verified images getting deployed to the users devices.
- Users devices are updates without needed to access external internet
- In the rare case there is an issues, the device can rollback to the prior image reducing downtime.
  
