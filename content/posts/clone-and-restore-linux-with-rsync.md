---
title: Clone and Restore Linux With rsync
description: Use rsync to create a file-based clone of a Linux root filesystem
date: 2021-12-28
tags: [linux, backup]
---

I recently had an SSD fail in my Macbook Pro. I was lucky enough to still be
able to read data from it (albeit slowly), and I found myself needing to
migrate my existing Arch Linux installation to a new SSD. There are many ways
to clone or migrate an existing Linux installation. One could use `dd` to
create a block-level backup, with the filesystem included. I decided to use
`rsync` to create a file-based clone of the root filesystem. This is an outline
of the process I followed.

{{< admonition note >}}
As a bit of background, I also have backups via
[BorgBackup](https://www.borgbackup.org/) but I didn't end up needing them in
this case as the drive was still readable. If it wasn't, my "clone" would have
instantly become a "restore", and I would have gone a different path entirely.
{{< /admonition >}}

# Creating A Clone

*Adapted from Arch Wiki[^1]*

There are two options:

- Clone the live system
- Mount the filesystem and clone from a separate system

What you choose will depend on your situation. The process is essentially the
same in either case.

## Cloning A Live System

To clone a live system, use the following command (as root):

```shell
rsync
    --archive \
    --hard-links \
    --acls \
    --xattrs \
    --sparse \
    --exclude={"/dev/*","/proc/*","/sys/*","/tmp/*","/run/*","/mnt/*","/media/*"} \
    / ${DESTINATION}
```

## Cloning A Mounted Filesystem

To clone a mounted filesystem, use the following command (as root):

```shell
rsync
    --archive \
    --hard-links \
    --acls \
    --xattrs \
    --sparse \
    --numeric-ids \
    --exclude={"/dev/*","/proc/*","/sys/*","/tmp/*","/run/*","/mnt/*","/media/*"} \
    ${SOURCE} ${DESTINATION}
```

## Notes on `rsync`

From the `rsync(1)` man page[^2]:

> A trailing slash on the source changes this behavior to avoid creating an
additional directory level at the destination. You can think of a trailing / on
a source as meaning "copy the contents of this directory" as opposed to "copy
the directory by name", but in both cases the attributes of the containing
directory are transferred to the containing directory on the destination.

Basically, use a trailing slash on your source so you don't get an additional
directory inserted in your destination path.

{{< admonition tip >}}
Use the `--info=progress2` parameter to show transfer speed
{{< /admonition >}}

{{< admonition warning >}}
Append any filesystem-specific folders (such as `/lost+found` on ext4) to the
`--excludes` parameter
{{< /admonition >}}

# Restoring the Clone

To restore the clone, we have to accomplish a few things:

## Block Device Initialization

The first step is to initialize the target block device. In some cases, this
device only needs the root filesystem. In most cases however, it will likely
need several partition, such as for `/boot` or swap space. Setup the partition table using something like
`fdisk`, and create the filesystems on the resultant partitions.

{{< admonition tip >}}
The filesystem types do not necessarily have to match the originals. You can
use this as an opportunity to swap filesystems for your root device if desired. 
You can also migrate to a new boot loader, but that would involve additional
work that is outside the scope of this guide.
{{< /admonition >}}

## Restore

*Adapted from Arch Wiki[^3]*

Once the block device is ready, mount the filesystems. Mount the `/boot`
partition inside of the `/` filesystem, so the boot filesystem is restored
properly. It should look roughly like this:

```shell
NAME             TYPE FSTYPE TYPE FSTYPE MOUNTPOINT
/dev/sdb         disk        disk
├─/dev/sdb1      part vfat   part vfat   /mnt/root/boot
└─/dev/sdb2      part xfs    part xfs    /mnt/root
```

When ready, begin the `rsync`:

```shell
rsync
    --archive \
    --hard-links \
    --acls \
    --xattrs \
    --sparse \
    --numeric-ids \
    ${SOURCE} /mnt/root
```

## Update Partition/Filesystem IDs

The clone has now been restored in it's entirety. The new block device and
filesystem(s) are nearly complete. There are a few places we need to update to
match the new device, partition table, and filesystems.

{{< admonition tip >}}
Block, partition, and filesystem IDs are easily obtainable via `blkid`:

```shell
$ blkid
/dev/sdb1: UUID="6bc9c81a-3df1-41b8-83ed-e3accf0735bc" BLOCK_SIZE="512" TYPE="xfs" PARTUUID="4efb81c4-2bd7-40a7-8c89-d002730bf4f1"
/dev/sdb2: UUID="6265-2A8D" BLOCK_SIZE="512" TYPE="vfat" PARTUUID="15dea608-1037-439a-ad36-a7b6b69f1fd1"
```
{{< /admonition >}}

### Boot Loader Entries

This will be highly system specific. You'll essentially need to install the
bootloader such that any partition IDs or `/dev`-tree mappings line up with
the new block device.

In the case of `system-boot`, this means updating the entry (or entries) in
`/boot/loader/entries`:

```
==> /boot/loader/entries/arch.conf <==
title   Arch Linux
linux   /vmlinuz-linux
initrd  /intel-ucode.img
initrd  /initramfs-linux.img
options root=UUID=6bc9c81a-3df1-41b8-83ed-e3accf0735bc rw quiet
```

The UUID of the root filesystem needs to be updated to match the new
filesystem.

### `/etc/fstab`

The `/etc/fstab` file that was restored likely uses partition or filesystem IDs from the
old device. These should be updated to match the new device.

In the case of the following example, the filesystem UUIDs would again need to
be updated:

```
==> /etc/fstab <==
# Static information about the filesystems.
# See fstab(5) for details.

# <file system> <dir> <type> <options> <dump> <pass>
UUID=d3f679c6-77ec-41b9-84ba-82377e3cab5e / xfs rw,relatime 0 1
UUID=6265-2A8D /boot vfat rw,relatime,errors=remount-ro 0 2
```

## Leftovers

If the only thing being changed is the block device, this should be all that
needs updated. In the event of a migration, more hardware device IDs are going
to change, and those need to be updated as well. Be sure to update network
device MAC addresses and other unique IDs. See the ArchWiki guide on hardware
migrations for more details.[^3]

# Conclusion

Keep in mind that there are a lot of variables here. Much of the details
depend on the distribution in use, and how it's configured on the source
installation. By following this outline, and filling in the gaps for your
system's particulars, you should be able to get from source to clone to done.

Big shout out and thank you to those who contribute to the ArchWiki. It's an
excellent resource, regardless of the distro you're actually running.


[^1]: Source: https://wiki.archlinux.org/title/Rsync#File_system_cloning
[^2]: Source: https://man.archlinux.org/man/rsync.1.en
[^3]: Source: https://wiki.archlinux.org/title/Migrate_installation_to_new_hardware#Top_to_bottom
