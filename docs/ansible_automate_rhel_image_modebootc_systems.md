---
title: "How to use Ansible to automate RHEL Image Mode/bootc systems"
permalink: /ansible_automate_rhel_image_modebootc_systems.md/
---

# How to use Ansible to automate RHEL Image Mode/bootc systems

## Intro

I'm currently converting my home lab from running [Fedora](https://fedoraproject.org/) in the familiar "package mode" to running Fedora as a [bootable container](https://docs.fedoraproject.org/en-US/bootc/) using "[bootc](https://github.com/containers/bootc)", i.e. the upstream version of [Image mode for Red Hat Enterprise Linux](https://www.redhat.com/en/technologies/linux-platforms/enterprise-linux/image-mode).

Part of this has involved me updating my existing Ansible automation to work with bootc. The biggest challenge is that many playbooks and roles attempt to install or update packages on the host, which is not possible using dnf/yum locally on the host as /usr is a read-only filesystem at runtime. The only way to add packages permanently is to bake them into the container image the system is built from by adding them to the Containerfile, like so:

```
FROM quay.io/fedora/fedora-bootc:41
RUN dnf -y install httpd vim tmux
```

So how do I address this?

## Dealing with package installs in my own automation

There's no magic bullet here. I need to take any package installs or updates in my own Ansible code and do one of three things:

- Remove it from my code altogether  
- tag the tasks that run an install or update and then run the playbook excluding tasks with that tag (`--skip-tags install-packages`)  
- Incorporate some logic that checks if the system is bootc (or rpm-ostree) based and then errors out *in a helpful way*

I took the path of the tag skipping option, as it allows me to preserve the playbook for execution on both package mode hosts and bootc hosts.

## Dealing with package installs in RHEL/Linux system roles

Red Hat Enterprise Linux (RHEL) system roles are a collection of Ansible roles that ... "help automate the management and configuration of RHEL systems". These are derived from the upstream Linux System Roles project ([https://linux-system-roles.github.io/](https://linux-system-roles.github.io/)). I use these to configure storage, systemd units, cockpit (RHEL Web Console), containers with podman, and more, because the roles make it easy to configure my systems in a well defined and repeatable way.

However they have the same problem as my home grown Ansible roles. Bootc (and rpm-ostree) based systems will still bomb out or error during role execution if the appropriate packages are not installed or cannot be installed.

Fortunately the authors have dealt with this cleverly by providing documentation and an accompanying shell script to generate the package list for each role. You can find an example from the storage role right here: [https://github.com/linux-system-roles/storage/blob/main/README-ostree.md](https://github.com/linux-system-roles/storage/blob/main/README-ostree.md)

So using that script and the instructions I can generate the list of packages required by the storage:

```
$ pwd
/var/home/bblasco/.ansible/collections/ansible_collections/fedora/linux_system_roles/roles
$ storage/.ostree/get_ostree_data.sh packages runtime Fedora-41 raw
cryptsetup
e2fsprogs
kpartx
libblockdev-crypto
libblockdev-dm
libblockdev-lvm
libblockdev-mdraid
libblockdev-swap
lvm2
python3-blivet
stratis-cli
stratisd
xfsprogs
```

That list may be a superset of the packages I need for the role to run, but it is the complete set. I can now add that list of packages to my Containerfile so I can run the role successfully:

```
FROM quay.io/fedora/fedora-bootc:41
RUN dnf -y install httpd vim tmux \
    cryptsetup e2fsprogs kpartx libblockdev-crypto \
    libblockdev-dm libblockdev-lvm libblockdev-mdraid libblockdev-swap \
    lvm2 python3-blivet stratis-cli stratisd \
    xfsprogs
```

I then need to repeat that process for every role I want to use on my bootc-based system. I am lazy, like any good system administrator, so here's a little bash snippet to generate the list for all the roles I am using.

```
$ for i in `echo "storage firewall timesync cockpit podman"`; do echo ROLE $i; $i/.ostree/get_ostree_data.sh packages runtime Fedora-41 raw; done
```

Note: There may be some repetition in the package list, which you can identify with the following addition to the above command. This will show you which packages appear more than once, and exactly how many times they appear.

```
$ for i in `echo "storage firewall timesync cockpit podman"`; do echo ROLE $i; $i/.ostree/get_ostree_data.sh packages runtime Fedora-41 raw; done | sort | uniq -cd
```

## Testing the automation after updating the Containerfile

Once the packages have been added to the Containerfile and a new image built, it’s really important that I test automation against a host to verify that it works, and that it behaves as intended.

This is done on an existing host by 

1. Pushing the updated container image to my registry  
2. Updating the existing host using the `bootc upgrade` or `bootc switch` command  
3. Rebooting the host  
4. Running the Ansible playbook against the host.    
5. Running whatever checks I need to ensure that the outcome satisfies my requirements.

If an existing host is not available, then a new bootc-based system would need to be deployed using the same container image and following a similar process.

## Wrap up

Adding packages straight into your Containerfile for bootc based images is the only way to handle the package requirements in Ansible playbooks and roles. This ensures a known and predictable (not to mention revertible) set of packages on the system. At the same time it still allows pre-existing automation to be used in bootc-based systems to configure and manage Linux systems at scale.

In my case, taking this approach has significantly reduced the effort for me to migrate my workloads to use the bootc paradigm. Hopefully it can do the same for you.
