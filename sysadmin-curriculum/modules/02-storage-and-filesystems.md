# Module 02: Storage: disks, partitions, LVM, and mounting

**Time:** about 5 hours.
**Prerequisites:** module 01. Two empty 5 GB virtual disks on the lab VM (or use the loop-device fallback below).

## By the end you can

- Identify disks, partitions, and filesystems, and know which one is the system disk.
- Partition a disk (GPT), create a filesystem, and mount it.
- Make a mount persistent with `/etc/fstab`, and verify it before rebooting.
- Use LVM to create, grow, and extend volumes.
- Explain why `df` and `du` disagree, and why a disk can be "full" with space free.
- Recover from a bad `fstab` entry.

## Why this matters

"Add a disk," "the disk is full," and "the volume needs to grow" are routine sysadmin tasks, and getting them wrong can destroy data or stop a server from booting. This is also where careless commands are most dangerous: one wrong device name wipes the wrong disk.

## Concepts

### The stack

```
physical or virtual disk   /dev/sdb
        |
partition table (GPT)      /dev/sdb1, /dev/sdb2
        |
(optional) LVM             physical volume -> volume group -> logical volume (/dev/vg_lab/lv_data)
        |
filesystem                 XFS (RHEL default), ext4
        |
mount point                /mnt/data
```

- **Block device.** Anything that stores data in blocks: `/dev/sda`, `/dev/nvme0n1`, `/dev/vda`. Partitions are numbered (`sda1`, or `nvme0n1p1`).
- **Partition table.** **GPT** is the modern standard (large disks, many partitions). **MBR** is the older one (2 TB limit, 4 primary partitions).
- **Filesystem.** **XFS** is the RHEL default and can be grown but not shrunk. **ext4** can be grown and (offline) shrunk. Each has its own repair tools.
- **LVM (Logical Volume Manager)** adds a layer so volumes can be resized, spanned over several disks, and snapshotted without repartitioning. Terms: **PV** (physical volume), **VG** (volume group, a pool), **LV** (logical volume, what you format and mount).
- **Mounting.** A filesystem becomes visible when it's attached to a directory. It disappears at reboot unless it's in `/etc/fstab`.
- **UUIDs.** Device names like `/dev/sdb1` can change between boots. UUIDs don't, so `fstab` should use them.
- **`/etc/fstab` fields:** `device  mountpoint  fstype  options  dump  fsck-order`. Useful options: `defaults`, `nofail` (don't stop booting if it's missing), `noexec`, `nosuid`, `noatime`.
- **Inodes.** Each file uses an inode. A filesystem can run out of inodes before it runs out of space.
- **RAID** (software with `mdadm`, or hardware) gives redundancy or speed. It is not a backup (see module 00).
- **Swap** is disk used as overflow memory.

## Lab

Snapshot first.

**The rule for this module: before any command that writes to a device, run `lsblk` and say out loud which disk you're about to change.** The system disk is the one with `/` mounted somewhere on it. Never partition or format that one.

```bash
sudo dnf install -y parted xfsprogs lvm2 lsof
```

### 1. Look before you touch

```bash
lsblk -f                  # disks, partitions, filesystems, mount points
lsblk -o NAME,SIZE,TYPE,MOUNTPOINT
blkid
sudo fdisk -l | head -40
findmnt                   # what's mounted where
df -hT                    # space, with filesystem type
cat /etc/fstab
```

Identify: which disk is the system disk? Which are the two empty disks (assumed here to be `/dev/sdb` and `/dev/sdc`, but check yours)? What filesystem types are in use?

### 2. Partition a disk

Use `parted` for one-liners, or `fdisk` interactively (`g` for a new GPT table, `n` for a new partition, `p` to print, `w` to write).

```bash
sudo parted /dev/sdb --script mklabel gpt
sudo parted /dev/sdb --script mkpart data xfs 1MiB 2GiB
sudo parted /dev/sdb --script mkpart more ext4 2GiB 100%
sudo parted /dev/sdb --script print
lsblk /dev/sdb
```

Make a note of what each argument did. Why start at 1 MiB?

### 3. Create a filesystem and mount it

```bash
sudo mkfs.xfs /dev/sdb1
sudo mkfs.ext4 /dev/sdb2
sudo mkdir -p /mnt/data /mnt/more
sudo mount /dev/sdb1 /mnt/data
sudo mount /dev/sdb2 /mnt/more
df -hT /mnt/data /mnt/more
echo hi | sudo tee /mnt/data/test.txt
```

Unmount and remount to see that the data is still there: `sudo umount /mnt/data && sudo mount /dev/sdb1 /mnt/data && cat /mnt/data/test.txt`.

### 4. Make it persistent

```bash
sudo blkid /dev/sdb1                         # copy the UUID
sudo cp /etc/fstab /etc/fstab.bak            # always back it up first
echo 'UUID=<paste-uuid-here>  /mnt/data  xfs  defaults,nofail  0 0' | sudo tee -a /etc/fstab
sudo systemctl daemon-reload
sudo umount /mnt/data
sudo mount -a                                # mounts everything in fstab; errors show up NOW, not at boot
findmnt /mnt/data
sudo findmnt --verify                        # checks fstab syntax and devices
```

Reboot (`sudo reboot`) and confirm it came back mounted. **The habit: never reboot after editing `fstab` until `mount -a` and `findmnt --verify` are clean.**

### 5. LVM on the second disk

```bash
sudo pvcreate /dev/sdc
sudo vgcreate vg_lab /dev/sdc
sudo lvcreate -n lv_data -L 2G vg_lab
sudo mkfs.xfs /dev/vg_lab/lv_data
sudo mkdir -p /mnt/lvm
sudo mount /dev/vg_lab/lv_data /mnt/lvm
sudo pvs; sudo vgs; sudo lvs
df -h /mnt/lvm
```

Now grow it while it's mounted and in use:

```bash
sudo lvextend -L +1G -r /dev/vg_lab/lv_data   # -r also grows the filesystem
df -h /mnt/lvm
```

Add it to `fstab` using the UUID from `blkid /dev/vg_lab/lv_data` (or the `/dev/mapper/vg_lab-lv_data` path), and verify with `mount -a`.

Answer: why is growing easy and shrinking XFS impossible? What would you do if the volume group ran out of space? (Hint: `vgextend` with a new disk.)

### 6. Mount options, inodes, and swap

```bash
sudo mount -o remount,ro /mnt/data          # read-only
echo test | sudo tee /mnt/data/blocked.txt  # fails: read-only filesystem
sudo mount -o remount,rw /mnt/data
df -i /mnt/data                              # inode usage
swapon --show; free -h
```

Optional: create a swap volume (`sudo lvcreate -n lv_swap -L 512M vg_lab`, `sudo mkswap /dev/vg_lab/lv_swap`, `sudo swapon /dev/vg_lab/lv_swap`, `swapon --show`), then `swapoff` it.

### No spare disks? Loop-device fallback

```bash
sudo truncate -s 5G /root/disk1.img
sudo losetup -fP --show /root/disk1.img      # prints /dev/loopN; use it in place of /dev/sdb
```

Then partition and format `/dev/loopN` as above. Remember to `sudo losetup -d /dev/loopN` when done. Loop devices don't survive a reboot, so the "reboot and check" step won't apply.

## Break it

1. **Disk is full, but `df` says there's space.** Inode exhaustion:

   ```bash
   sudo truncate -s 100M /root/small.img
   sudo mkfs.ext4 -N 1000 /root/small.img
   sudo mkdir -p /mnt/small && sudo mount -o loop /root/small.img /mnt/small
   cd /mnt/small && for i in $(seq 1 1500); do sudo touch f$i || break; done; cd ~
   df -h /mnt/small; df -i /mnt/small
   ```

   `touch` fails with "No space left on device" while `df -h` shows free space. Find the cause with `df -i`. Why does XFS rarely hit this?

2. **Disk is full, but `du` can't find it.** A deleted file still held open by a process:

   ```bash
   sudo dd if=/dev/zero of=/mnt/data/big bs=1M count=500
   sudo tail -f /mnt/data/big > /dev/null &
   sudo rm /mnt/data/big
   df -h /mnt/data; sudo du -sh /mnt/data
   sudo lsof +L1 | grep /mnt/data               # deleted files that are still open
   ```

   Free the space by stopping the process (find its PID, then `kill`). Why isn't the space released when you `rm` the file?

3. **A bad `fstab` entry.** Your mentor puts a wrong UUID in `fstab` (with `nofail`) and reboots. Find what failed with `systemctl --failed`, `journalctl -b | grep -i mount`, and `findmnt --verify`. Then repeat on a snapshot **without** `nofail` to see the emergency mode prompt, and practise fixing it from there. Only do the second part on a snapshotted lab VM.

4. **The wrong disk.** With `lsblk` output in front of you, explain which device you'd change to "add a data disk" and why you would not touch the one that holds `/`.

## Check your understanding

- What is the difference between a partition, a volume group, a logical volume, and a filesystem?
- Why does `fstab` use UUIDs?
- What does `nofail` do, and when would you not want it?
- What does `mount -a` check, and why run it before rebooting?
- A volume is 95% full. Walk through growing it with LVM, and what changes if the filesystem is XFS vs ext4.
- `df` shows 100% and `du` shows 60%. What are two explanations?
- Why isn't RAID a backup?

## Deliverable

A runbook: **"Add a new disk to a server and keep it mounted across reboots."** Include the exact commands, the verification steps (`lsblk`, `mount -a`, `findmnt --verify`), how you'd roll back, and the `fstab` line you used. Also include your `lsblk`, `vgs`, and `lvs` output from the lab. Keep it in your notes repo and follow the documentation standard in [STANDARDS.md](../STANDARDS.md).

## Use AI well here

Paste your `fstab` (sanitized if it has anything sensitive) and your runbook into Claude and ask: "What could go wrong with each step? What would you check before rebooting?" Verify each risk it lists against `man fstab`, `man mount`, and the RHEL docs. Then ask it to quiz you on LVM terms.

## Go deeper

- `man fstab`, `man mount`, `man lvm`, `man parted`, `man xfs_growfs`
- Red Hat's RHEL 9 documentation: *Managing storage devices* and *Configuring and managing logical volumes*
- Software RAID with `mdadm`, LVM snapshots, and thin provisioning (not covered here)
