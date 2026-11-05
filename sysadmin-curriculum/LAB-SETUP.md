# Lab setup

You need one Rocky Linux 9 VM you can break and reset. Everything in the modules runs on it.

## The VM

- **OS:** Rocky Linux 9, minimal or "Server" install. (It's binary-compatible with RHEL 9. The main differences on real RHEL are `subscription-manager` and Red Hat's docs portal.)
- **Size:** 2 CPU, 2 GB RAM, 20 GB for the main disk is enough.
- **Extra disks:** add **two extra small virtual disks (5 GB each)** to the VM. Module 02 (storage) partitions and formats them, so they must be empty and disposable. If you can't add disks, module 02 has a loop-device fallback that works with files instead.
- **Where:** VMware, VirtualBox, Hyper-V, Proxmox, or a cheap cloud VPS all work. A mentor may hand you a prebuilt VM instead.
- **Network:** you must be able to reach it from your own machine, because several labs test access from outside the VM.

## Setup checklist

Whoever builds the VM does these. If a mentor gives you one, verify each item yourself as a first exercise.

- [ ] Rocky Linux 9 installed and updated: `sudo dnf update -y`
- [ ] A non-root account with `sudo` (the `wheel` group on RHEL-family systems)
- [ ] SSH access from your own machine works
- [ ] **SELinux disabled** for this lab: `sudo sed -i 's/^SELINUX=.*/SELINUX=disabled/' /etc/selinux/config && sudo reboot`. Afterward `getenforce` says `Disabled`.
- [ ] **firewalld running**: `sudo firewall-cmd --state` says `running`. Leave it on.
- [ ] A clean **snapshot** of this state, named something like `clean-rocky9`

Why SELinux is off: it's a large topic that would overwhelm the early modules, and the lab goal is to learn permissions, services, and networking first. It is not a recommendation for production. See the note in the README about adding an SELinux module if your servers enforce it.

Why firewalld stays on: real servers have firewalls, and finding "what's blocking this" is a core skill. Don't stop it to fix things.

## Snapshots and resets

- Take a snapshot before each module if you can, and name it for the module.
- Several modules break things on purpose. If you break the VM so badly you can't recover it, restore the snapshot. Try to fix it yourself first.
- Some labs install packages and change config. Restore the clean snapshot between modules if you want a fresh start. Module 11 specifically asks you to rebuild from a clean snapshot.

## Optional second VM

A second Rocky VM is useful for:

- **Module 04:** testing reachability between hosts.
- **Module 07:** a real backup target, so the backup isn't on the machine it protects.
- **Module 10:** real backends instead of local ports.
- **Module 11:** configuring a host from another host with Ansible.

Everything also works on one VM; the module says when a second one is a stretch goal.

## Verify the lab

```bash
cat /etc/os-release | head -2
getenforce                      # Disabled
sudo firewall-cmd --state       # running
ip -br addr                     # note your IP
lsblk                           # you should see the two extra, empty disks
sudo whoami                     # root
```

From your own machine: `ssh <user>@<vm-ip>`.

## Note

The commands in the modules are written for Rocky Linux 9. Package names and defaults can change between releases, so check the current docs if something doesn't match. If you find an error, fix the module and tell your mentor.
