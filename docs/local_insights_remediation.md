---
title: "How to run an Insights remediation locally on a RHEL 7 host"
permalink: /local_insights_remediation/
---

# How to run an Insights remediation locally on a RHEL 7 host

## Introduction

Insights for Red Hat Enterprise Linux, available at [console.redhat.com](console.redhat.com), is a great way to identify issues that may compromise the security, stability, and performance of your RHEL systems.  It also gives you the ability to remediate issues using the power of Ansible.  There are many ways to execute the remediation depending on your environment:

- Through Red Hat Satellite
- Through Ansible Automation Platform (formerly known as Ansible Tower)
- Directly to the host (RHEL 8.5 onwards)
- "Manually" on a host with Ansible installed, including the host being remediated itself

Here is an example where we take a remediation, in the form of an Ansible playbook downloaded from Insights (ie the Hybrid Cloud Console at [console.redhat.com](console.redhat.com)), and run it locally on the host we are remediating.

## What we need to do on our RHEL 7 host

To run a remediation locally, we need to:
1. Download the playbook to the host
2. Install Ansible on the host
3. Run the remediation playbook using Ansible

Note: The limitation of this method is that it will only enable us to run a remediation playbook for that host.  Running a remediation that covers _multiple_ hosts is not in the scope of this example.

## Commands to run

For this example, we will use an example playbook called "playbook.yml", located in the user's home directory.  Note that your playbook name will change according to what you named it in Insights.

Firstly, let's enable the Ansible repository and install Ansible:

```
sudo subscription-manager repos --enable rhel-7-server-ansible-2-rpms
sudo yum install -y ansible
```

Now that Ansible is installed, we can run the playbook.  Here are the commands, but don't run them yet!
```
HOSTNAME=`cat /etc/hostname`
ansible-playbook -i "$HOSTNAME," -c local playbook.yml
```

Before running the playbook, let's break these last commands down:
- The first command grabs the system hostname and stores it in a variable called `HOSTNAME` that we can refer to later.  This must match the hostname of system as defined in Insights (and therefore inside playbook.yml)
- `ansible-playbook` is the command to run an Ansible playbook
- `-i "$HOSTNAME,"` generates an ad-hoc inventory that only contains this host
- `-c local` ensures that we are using a local connection to the host (rather than the default SSH)
- `playbook.yml` is the name of the playbook we are running

We probably want to execute a non-invasive dry-run before running the playbook that changes the host, to ensure that there are no missing package dependencies etc.  We do this with the `--check` parameter:
```
ansible-playbook -i "$HOSTNAME," -c local playbook.yml --check
```

We may have a situation where the user requires the sudo password to become the root user, in which case the command needs to have the `--ask-become-pass` parameter passed to it:
```
ansible-playbook -i "$HOSTNAME," -c local playbook.yml --ask-become-pass
```

So with that, here's an example of the output you would see when running a simple remediation on the host called "rhel79":

```
[user@rhel79 ~]$ ansible-playbook -i "$HOSTNAME," -c local playbook.yml

PLAY [update packages] ****************************************************************************************************************

TASK [Gathering Facts] ****************************************************************************************************************
ok: [rhel79]

TASK [check for update] ***************************************************************************************************************
changed: [rhel79]

TASK [upgrade package] ****************************************************************************************************************
changed: [rhel79]

TASK [set reboot fact] ****************************************************************************************************************
ok: [rhel79]

PLAY [run insights] *******************************************************************************************************************

TASK [run insights] *******************************************************************************************************************
ok: [rhel79]

PLAY RECAP ****************************************************************************************************************************
rhel79             : ok=5    changed=2    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

The above output tells us that a package upgrade took place, ie "changed", on the system, and everything ran successfully.  It also shows that the Insights client was run afterwards, updating the data available in the Hybrid Cloud Console.

## What if I want to centralise my automation?

The local example above provides the lowest effort way to run a playbook on a single host, but doesn't really scale.  Ideally we want to install Ansible on a single host and use it as the controller, or centralised location from where we run our automation.  This is what we would do differently:

1. Install Ansible on a host that can reach all the other hosts that want to automate against.  We will refer to this as the control node.
2. Define an inventory of RHEL hosts that we run our Insights-generated playbooks against.  Assuming that you have working DNS and that the DNS names match the hostnames defined for your Insights inventory, your inventory (which lives in `/etc/ansible/hosts`) would look something like this:
```
dbserver.example.com
webserver.example.com
mailserver.example.com
```
3. Ensure that:
   - The same Linux user is defined on on control node and the hosts in the inventory
   - The control node's public SSH key has been shared with all the hosts in the inventory (via `ssh-copy-id`)
   - That Linux user can sudo to root without a password.  If this won't be possible you will need to look up the Ansible `--ask-become-pass` parameter and read further

With all of the above setup done, we just need to copy the remediation playbook to our control node, and run it as follows:
```
ansible-playbook playbook.yml
```

This configuration allows us to run playbooks that are automating against multiple hosts, for example when applying the same patch to several systems.

## Further reading

If you want to learn more about Ansible, check out the following helpful resources:
- [Intro to playbooks](https://docs.ansible.com/ansible/latest/user_guide/playbooks_intro.html)
- [Learning Ansible basics](https://www.redhat.com/en/topics/automation/learning-ansible-tutorial)
- [How to build your inventory](https://docs.ansible.com/ansible/latest/user_guide/intro_inventory.html)