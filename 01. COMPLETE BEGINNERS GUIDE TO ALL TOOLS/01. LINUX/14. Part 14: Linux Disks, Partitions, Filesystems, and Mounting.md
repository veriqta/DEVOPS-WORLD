# Part 14: Linux Disks, Partitions, Filesystems, and Mounting

In Part 13, you learned DNS, SSH, and remote Linux administration. You learned how to reach a remote server, authenticate securely, transfer files, and troubleshoot failures from name resolution through authentication.

Now we move to one of the most important areas of Linux administration: storage.

A Linux server can have perfectly healthy CPU, memory, networking, and applications and still fail because:

```text
The disk is full.
The wrong filesystem is mounted.
A mount disappeared after reboot.
The server ran out of inodes.
A filesystem became read-only.
An administrator mounted a disk over the wrong directory.
```

Storage problems are common in DevOps, Cloud, SRE, and Platform Engineering.

By the end of Part 14, you should understand the complete beginner storage chain:

```text
Physical / Virtual Disk
        |
        v
Block Device
        |
        v
Partition
        |
        v
Filesystem
        |
        v
Mount
        |
        v
Directory
        |
        v
Files
```

Most importantly, you should understand what each layer means instead of treating "disk" and "filesystem" as the same thing.

---

# 1. Storage Fundamentals

Computers need somewhere to store data after power is removed.

RAM is primarily temporary working memory.

Storage devices provide persistent storage.

Examples include:

* SSDs.
* HDDs.
* NVMe drives.
* Virtual disks.
* Cloud block-storage volumes.
* SAN-backed storage.

Linux applications eventually store information as files.

For example:

```text
Application
    |
    v
/var/lib/myapp/data
    |
    v
Filesystem
    |
    v
Storage device
```

But several layers exist between the application and the physical or virtual disk.

Understanding those layers is the foundation of Linux storage administration.

---

# 2. Why Storage Matters to Linux Engineers

Consider a web application.

It may store:

```text
Application files
Configuration
Logs
Temporary files
Uploaded files
Database files
Container data
Package data
```

Suppose logs grow continuously:

```text
/var/log/myapp/app.log
```

Eventually:

```text
Filesystem usage
40%
 |
 v
70%
 |
 v
90%
 |
 v
100%
```

Applications may begin failing.

You might see:

```text
No space left on device
```

But that message does not always mean the same thing.

Possible causes include:

```text
Filesystem blocks exhausted
```

or:

```text
Inodes exhausted
```

You need to know how Linux storage works to distinguish them.

---

# 3. Disk vs Partition vs Filesystem

These three concepts are commonly confused.

They are not interchangeable.

Think:

```text
Disk
 |
 +-- Partition 1
 |       |
 |       +-- Filesystem
 |
 +-- Partition 2
         |
         +-- Filesystem
```

A simplified definition:

| Term       | Meaning                                    |
| ---------- | ------------------------------------------ |
| Disk       | Storage device                             |
| Partition  | Logical region of a disk                   |
| Filesystem | Structure used to organize and store files |

We will examine each one separately.

---

# 4. What Is a Disk?

A disk is a storage device presented to the operating system.

It could represent:

* Physical HDD.
* Physical SSD.
* NVMe SSD.
* Virtual machine disk.
* Cloud block-storage volume.
* Storage provided by a hypervisor.

Linux may expose one as:

```text
/dev/sda
```

or:

```text
/dev/nvme0n1
```

These names represent devices.

They are not ordinary files containing your documents.

---

# 5. What Is a Partition?

A partition is a logical section of a storage device.

Imagine a 100 GB disk:

```text
Disk: 100 GB
+-------------------------------+
|                               |
|          Entire Disk          |
|                               |
+-------------------------------+
```

You could divide it:

```text
100 GB Disk
+----------+--------------------+
| 20 GB    | 80 GB              |
| Part 1   | Part 2             |
+----------+--------------------+
```

Linux might expose them as:

```text
/dev/sda1
/dev/sda2
```

Here:

```text
/dev/sda
```

represents the disk.

While:

```text
/dev/sda1
```

and:

```text
/dev/sda2
```

represent partitions.

---

# 6. What Is a Filesystem?

A filesystem defines how data is organized and managed on storage.

Examples include:

```text
ext4
XFS
```

Without a filesystem, a partition is essentially just a region of storage from the perspective of normal file operations.

The filesystem provides structures for managing:

* Files.
* Directories.
* File names.
* Metadata.
* Permissions.
* Ownership.
* Timestamps.
* Allocation of storage.

Think:

```text
Storage space
     |
     v
Filesystem organizes it
     |
     +-- directories
     +-- files
     +-- metadata
     +-- permissions
```

---

# 7. Disk vs Partition vs Filesystem Example

Suppose:

```text
/dev/sdb
```

is a 50 GB disk.

It contains:

```text
/dev/sdb1
```

as one partition.

That partition contains an ext4 filesystem.

Then:

```text
Disk
/dev/sdb
   |
   v
Partition
/dev/sdb1
   |
   v
Filesystem
ext4
```

But you still need another concept before users can conveniently access it.

That concept is mounting.

---

# 8. What Is Mounting?

Mounting makes a filesystem accessible at a location in the Linux directory tree.

Suppose:

```text
/dev/sdb1
```

contains an ext4 filesystem.

You create:

```text
/data
```

Then mount the filesystem there.

Conceptually:

```text
/dev/sdb1
   |
   | ext4 filesystem
   v
/data
```

Now files stored in that filesystem appear under:

```text
/data
```

The directory `/data` is called the:

```text
mount point
```

---

# 9. Complete Storage Model

You should begin memorizing this relationship:

```text
Disk
/dev/sdb
    |
    v
Partition
/dev/sdb1
    |
    v
Filesystem
ext4
    |
    v
Mount
    |
    v
Mount point
/data
    |
    v
Files
/data/app
/data/logs
/data/backups
```

This model will make the rest of Part 14 much easier.

---

# 10. What Is a Block Device?

Linux commonly represents storage devices as block devices.

A block device is a device that handles data in blocks and supports random access.

Examples include:

* Disks.
* SSDs.
* Partitions.
* Cloud block volumes.

You can inspect block devices with:

```bash
lsblk
```

The name means roughly:

```text
list block devices
```

---

# 11. The `/dev` Directory

Linux exposes device interfaces under:

```text
/dev
```

You encountered `/dev/null` in Part 7.

Storage devices also appear under `/dev`.

Examples:

```text
/dev/sda
/dev/sda1
/dev/sdb
/dev/nvme0n1
/dev/nvme0n1p1
```

Important:

```text
/dev/sda
```

is not simply a normal file you should edit.

It represents access to a device.

Writing directly to the wrong block device can destroy data.

---

# 12. Linux Device Naming

You may encounter several naming styles.

Traditional SATA/SCSI-style devices:

```text
/dev/sda
/dev/sdb
/dev/sdc
```

Partitions:

```text
/dev/sda1
/dev/sda2
/dev/sdb1
```

NVMe devices may look like:

```text
/dev/nvme0n1
/dev/nvme1n1
```

Partitions may look like:

```text
/dev/nvme0n1p1
/dev/nvme0n1p2
```

Cloud and virtualized environments can vary.

Never assume:

```text
/dev/sdb
```

will always be your new disk.

Inspect the system.

---

# 13. Understanding `/dev/sda`

Conceptually:

```text
sda
||| 
||+-- first disk
|+--- disk family/name
+---- storage device naming convention
```

Additional devices might appear as:

```text
sdb
sdc
sdd
```

But modern systems may use completely different naming conventions.

This is why administrators use inspection tools instead of assumptions.

---

# 14. Understanding NVMe Names

Example:

```text
/dev/nvme0n1
```

A partition might be:

```text
/dev/nvme0n1p1
```

Notice the `p`.

Compare:

```text
/dev/sda
/dev/sda1
```

with:

```text
/dev/nvme0n1
/dev/nvme0n1p1
```

The naming syntax differs, but the basic relationship remains:

```text
Device
   |
   +-- Partition
```

---

# 15. `lsblk`

One of your first storage commands should be:

```bash
lsblk
```

Example output might resemble:

```text
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sda      8:0    0   40G  0 disk
├─sda1   8:1    0   39G  0 part /
└─sda2   8:2    0    1G  0 part /boot
sdb      8:16   0   20G  0 disk
```

Do not memorize the exact output.

Learn how to read the hierarchy.

---

# 16. Reading `lsblk`

From:

```text
sda
├─sda1
└─sda2
```

you can see:

```text
sda
=
disk

sda1
=
partition belonging to sda

sda2
=
another partition belonging to sda
```

And:

```text
sdb
```

has no child partition in this example.

That may indicate a new or differently configured device.

Do not immediately format it.

First investigate.

---

# 17. Useful `lsblk` Columns

Important columns include:

```text
NAME
SIZE
TYPE
FSTYPE
UUID
MOUNTPOINTS
```

A useful command is:

```bash
lsblk -f
```

The `-f` option displays filesystem-related information.

Example:

```text
NAME   FSTYPE UUID                                 MOUNTPOINTS
sda
└─sda1 ext4   1111-2222-3333-4444                 /
sdb
└─sdb1 xfs    5555-6666-7777-8888                 /data
```

Now you can see:

```text
Device
Filesystem
UUID
Mount point
```

in one place.

---

# 18. Another Useful `lsblk` Command

You can explicitly choose columns:

```bash
lsblk -o NAME,SIZE,TYPE,FSTYPE,UUID,MOUNTPOINTS
```

This is useful when troubleshooting.

You might see:

```text
NAME    SIZE TYPE FSTYPE UUID       MOUNTPOINTS
sda      40G disk
└─sda1   40G part ext4   abc-123    /
sdb      50G disk
└─sdb1   50G part xfs    def-456    /data
```

Now the storage chain is visible.

---

# 19. `blkid`

Another important command is:

```bash
blkid
```

It displays information about block devices, particularly filesystem metadata such as:

* UUID.
* Filesystem type.
* Labels.

You may need elevated privileges to see all devices:

```bash
sudo blkid
```

Example:

```text
/dev/sda1: UUID="abc123" TYPE="ext4"
/dev/sdb1: UUID="def456" TYPE="xfs"
```

---

# 20. Why `blkid` Is Useful

Suppose you need the UUID of:

```text
/dev/sdb1
```

Run:

```bash
sudo blkid /dev/sdb1
```

You may receive:

```text
/dev/sdb1: UUID="def456..." TYPE="ext4"
```

That UUID becomes important when configuring persistent mounts.

---

# 21. What Is a Partition Table?

Before partitions can be defined, disks commonly use a partition table.

A partition table describes how a disk is divided into partitions.

Think:

```text
Disk
 |
 +-- Partition table
       |
       +-- Partition 1
       +-- Partition 2
       +-- Partition 3
```

Two partition-table formats you may encounter are:

```text
MBR
GPT
```

---

# 22. MBR Introduction

MBR stands for:

```text
Master Boot Record
```

It is an older partitioning scheme.

It has historical limitations, including constraints around large disks and partition structures.

You will still encounter MBR systems.

For modern systems, GPT is common.

---

# 23. GPT Introduction

GPT stands for:

```text
GUID Partition Table
```

GPT is a modern partitioning scheme.

It supports:

* Large disks.
* More partitions.
* More robust partition metadata than traditional MBR.

You do not need to master partition-table internals yet.

At this stage, remember:

```text
Disk
 |
 v
Partition table
 |
 v
Partitions
```

---

# 24. Partitioning Tools

Linux provides tools such as:

```text
fdisk
parted
gdisk
```

These can modify disk partition tables.

They are powerful and potentially destructive.

A mistake can destroy access to data.

For a beginner:

```text
Never modify a production disk just because a tutorial says "run fdisk."
```

Always identify:

* Device.
* Existing partitions.
* Existing filesystem.
* Existing mount.
* Data importance.
* Backup/recovery path.

before changing storage structures.

---

# 25. Inspecting Partitions Safely

You can inspect with:

```bash
lsblk
```

and:

```bash
sudo fdisk -l
```

The:

```text
-l
```

means list.

This is an inspection operation.

Example output may include:

```text
Disk /dev/sdb: 20 GiB
Device     Start      End  Sectors Size Type
/dev/sdb1   2048 41943006 ...      20G Linux filesystem
```

Focus on understanding what exists before learning to modify it.

---

# 26. Creating a Partition

For a safe disposable lab disk, an administrator might use:

```bash
sudo fdisk /dev/sdb
```

This opens an interactive partition editor.

Inside `fdisk`, commands commonly include:

```text
p
```

print current partition table.

```text
n
```

create a new partition.

```text
w
```

write changes.

```text
q
```

quit without saving changes.

Warning:

```text
Writing a partition table changes the disk.
```

Never practice this on your system disk.

Use a disposable VM disk or test cloud volume.

---

# 27. Why Device Verification Matters

Imagine you intend to partition:

```text
/dev/sdb
```

but accidentally run:

```bash
sudo fdisk /dev/sda
```

If `/dev/sda` is your operating-system disk, destructive changes could make the machine unbootable or destroy data.

Before partitioning, run:

```bash
lsblk
```

Then verify:

```text
Device name
Size
Existing partitions
Existing filesystems
Existing mount points
```

Do not rely on memory.

---

# 28. Filesystems Revisited

After creating a partition, it may still need a filesystem before being used as normal file storage.

Suppose:

```text
/dev/sdb1
```

exists.

You might create an ext4 filesystem:

```bash
sudo mkfs.ext4 /dev/sdb1
```

Or an XFS filesystem:

```bash
sudo mkfs.xfs /dev/sdb1
```

`mkfs` means:

```text
make filesystem
```

This operation is potentially destructive.

---

# 29. Critical `mkfs` Warning

Running:

```bash
sudo mkfs.ext4 /dev/sdb1
```

on a partition containing valuable data can destroy the existing filesystem structure and make existing data inaccessible.

Before `mkfs`, verify:

```bash
lsblk -f
```

and:

```bash
sudo blkid /dev/sdb1
```

Ask:

```text
Does this device already contain a filesystem?

Is it mounted?

Does it contain data?

Am I certain this is the intended device?
```

Never blindly format storage.

---

# 30. ext4

`ext4` stands for:

```text
Fourth Extended Filesystem
```

It is widely used on Linux.

Features include:

* Journaling.
* Large filesystem support.
* File permissions.
* Extended attributes.
* Mature Linux support.

You may encounter ext4 frequently on:

* Ubuntu.
* Debian.
* Virtual machines.
* Cloud systems.
* General-purpose Linux servers.

---

# 31. What Is Journaling?

A journaling filesystem records information about filesystem changes to help maintain consistency after unexpected interruptions.

For a beginner, think:

```text
Filesystem operation
       |
       v
Journal tracks relevant changes
       |
       v
Filesystem updated
```

If the machine crashes unexpectedly, the journal can help the filesystem recover consistency more efficiently.

Journaling is not a replacement for backups.

---

# 32. XFS

XFS is another widely used Linux filesystem.

It is common in enterprise Linux environments.

You may encounter it on:

* RHEL.
* Rocky Linux.
* AlmaLinux.
* Enterprise servers.
* Large storage workloads.

XFS supports:

* Journaling.
* Large filesystems.
* Large files.
* High-performance storage workloads.

---

# 33. ext4 vs XFS

At a beginner level:

| ext4                                                        | XFS                                                                |
| ----------------------------------------------------------- | ------------------------------------------------------------------ |
| Very common general-purpose Linux filesystem                | Very common enterprise filesystem                                  |
| Common on Debian/Ubuntu systems                             | Common on RHEL-family systems                                      |
| Mature and widely supported                                 | Mature and widely supported                                        |
| Can grow and can be shrunk offline with appropriate tooling | Can grow, but traditional XFS shrinking is not generally supported |

Do not choose a filesystem solely from a small comparison table.

Real decisions depend on:

* Operating system.
* Workload.
* Support requirements.
* Storage architecture.
* Backup strategy.
* Recovery requirements.
* Platform standards.

---

# 34. Identifying the Filesystem

Use:

```bash
lsblk -f
```

or:

```bash
sudo blkid
```

You might see:

```text
/dev/sdb1: UUID="..." TYPE="xfs"
```

That tells you the filesystem type.

You can also use:

```bash
df -T
```

for mounted filesystems.

The `-T` option includes filesystem type.

---

# 35. Creating an ext4 Filesystem

On a verified empty disposable partition:

```bash
sudo mkfs.ext4 /dev/sdb1
```

Afterward:

```bash
lsblk -f
```

might show:

```text
sdb
└─sdb1 ext4
```

The filesystem exists.

But it may not yet be mounted.

---

# 36. Creating an XFS Filesystem

On a verified empty disposable partition:

```bash
sudo mkfs.xfs /dev/sdb1
```

Then verify:

```bash
lsblk -f
```

If `mkfs.xfs` is unavailable, the relevant XFS utilities package may not be installed.

Do not install packages blindly on production systems.

---

# 37. What Is a Mount Point?

A mount point is a directory where a filesystem becomes accessible.

Examples:

```text
/
/boot
/home
/data
/mnt/data
/var/lib/database
```

Suppose:

```text
/dev/sdb1
```

contains an ext4 filesystem.

You create:

```bash
sudo mkdir -p /data
```

Then mount:

```bash
sudo mount /dev/sdb1 /data
```

Now:

```text
/dev/sdb1
      |
      v
    /data
```

---

# 38. The `mount` Command

General syntax:

```bash
mount DEVICE MOUNT_POINT
```

Example:

```bash
sudo mount /dev/sdb1 /data
```

This tells Linux:

```text
Make the filesystem on /dev/sdb1 accessible at /data.
```

---

# 39. Verify a Mount

Never assume it worked.

Run:

```bash
lsblk
```

You may see:

```text
sdb
└─sdb1  20G part /data
```

Also:

```bash
df -h /data
```

And:

```bash
mount | grep '/data'
```

You can also use:

```bash
findmnt /data
```

`findmnt` is particularly useful for examining mounted filesystems.

---

# 40. What Happens to Existing Files in a Mount Point?

This is extremely important.

Suppose `/data` already contains:

```text
/data/file1
/data/file2
```

Then you mount another filesystem on:

```text
/data
```

The original files are not necessarily deleted.

They become hidden behind the mounted filesystem while that mount is active.

Conceptually:

```text
Before mount:

/data
├── file1
└── file2
```

Mount `/dev/sdb1` on `/data`:

```text
/data
├── disk-file-a
└── disk-file-b
```

The original `file1` and `file2` are hidden beneath the mount.

---

# 41. Why Mounting Over a Directory Can Be Dangerous

Imagine:

```text
/var/lib/myapp
```

already contains application data.

You accidentally mount an empty disk over it.

The application suddenly sees:

```text
empty directory
```

You might incorrectly conclude:

```text
All application data was deleted.
```

But the data may simply be hidden beneath the mount.

Always inspect mount points before mounting.

---

# 42. `umount`

To unmount a filesystem:

```bash
sudo umount /data
```

Notice:

```text
umount
```

not:

```text
unmount
```

You can also unmount by device:

```bash
sudo umount /dev/sdb1
```

Using the mount point is often clearer for beginners.

---

# 43. What Unmounting Means

Unmounting disconnects the filesystem from that location in the directory tree.

Before:

```text
/dev/sdb1
   |
   v
/data
```

After:

```text
/dev/sdb1

/data
```

The mount point directory may remain.

The filesystem is simply no longer attached there.

---

# 44. "Target Is Busy"

You may run:

```bash
sudo umount /data
```

and receive an error similar to:

```text
target is busy
```

This means something is using the filesystem.

Possibilities include:

* Your shell's current directory is inside it.
* Another user is inside it.
* A process has a file open.
* An application is using files there.

---

# 45. Simple Busy-Mount Example

Suppose:

```bash
cd /data
```

Then:

```bash
sudo umount /data
```

may fail because your own shell is using `/data`.

Move out:

```bash
cd /
```

Then retry:

```bash
sudo umount /data
```

This is a classic beginner mistake.

---

# 46. Investigating a Busy Mount

Useful commands can include:

```bash
lsof +D /data
```

or:

```bash
fuser -vm /data
```

These may help identify processes using the mount.

Be careful before killing processes.

A database, application, or important service may be using the filesystem.

---

# 47. Temporary Mounts

Suppose you run:

```bash
sudo mount /dev/sdb1 /data
```

The mount works.

You reboot.

After reboot, `/data` may no longer contain that mounted filesystem.

Why?

Because manually mounting a filesystem does not automatically create persistent boot configuration.

For persistence, Linux commonly uses:

```text
/etc/fstab
```

---

# 48. What Is a UUID?

UUID means:

```text
Universally Unique Identifier
```

Filesystems commonly receive UUIDs.

Example:

```text
6e3a9b7d-0c2f-4f98-9e82-123456789abc
```

You can view UUIDs with:

```bash
lsblk -f
```

or:

```bash
sudo blkid
```

---

# 49. Why UUIDs Are Useful

Device names can potentially change depending on hardware discovery and environment.

For example, relying only on:

```text
/dev/sdb1
```

may be less stable than identifying the filesystem by UUID.

A UUID identifies the filesystem itself.

Conceptually:

```text
Device name:
Where Linux currently exposes the device.

UUID:
Identifier associated with the filesystem.
```

This is why `/etc/fstab` commonly uses UUIDs.

---

# 50. `/etc/fstab`

The file:

```text
/etc/fstab
```

defines filesystems that Linux should mount according to configured rules.

`fstab` means:

```text
filesystem table
```

View it:

```bash
cat /etc/fstab
```

Do not modify it casually.

A bad entry can cause boot or mount problems.

---

# 51. Example `/etc/fstab` Entry

An entry might look like:

```text
UUID=abc123 /data ext4 defaults 0 2
```

There are six fields:

```text
UUID=abc123
/data
ext4
defaults
0
2
```

Let's understand each one.

---

# 52. fstab Field 1: Device

```text
UUID=abc123
```

This identifies the filesystem.

You may also see device paths such as:

```text
/dev/sdb1
```

but UUIDs are commonly preferred for persistent configuration.

---

# 53. fstab Field 2: Mount Point

```text
/data
```

This tells Linux where the filesystem should be mounted.

The directory should exist.

Create if appropriate:

```bash
sudo mkdir -p /data
```

---

# 54. fstab Field 3: Filesystem Type

Example:

```text
ext4
```

or:

```text
xfs
```

This identifies the filesystem type.

If the actual filesystem is XFS but you incorrectly specify ext4, mounting may fail.

---

# 55. fstab Field 4: Mount Options

Example:

```text
defaults
```

Mount options control filesystem behavior.

`defaults` is a common general-purpose starting option.

There are many other mount options.

Some influence:

* Read/write behavior.
* Execution.
* Security.
* Performance.
* Device handling.

You do not need to memorize them all yet.

---

# 56. fstab Field 5: Dump

Example:

```text
0
```

This historically relates to the `dump` backup utility.

Many modern configurations use:

```text
0
```

You do not need deep knowledge of this field at beginner level.

---

# 57. fstab Field 6: Filesystem Check Order

Example:

```text
2
```

This can influence filesystem checking order during boot for filesystems where applicable.

Common conceptual values:

```text
0
=
do not check through this mechanism

1
=
usually root filesystem

2
=
other eligible filesystems
```

Exact behavior depends on filesystem and system configuration.

---

# 58. Creating a Persistent Mount

Suppose:

```text
Device:
/dev/sdb1

Filesystem:
ext4

Mount point:
/data
```

First obtain the UUID:

```bash
sudo blkid /dev/sdb1
```

Suppose it returns:

```text
UUID="abc123"
```

Create mount point if needed:

```bash
sudo mkdir -p /data
```

Then carefully add:

```text
UUID=abc123 /data ext4 defaults 0 2
```

to:

```text
/etc/fstab
```

But do not reboot yet.

Test it first.

---

# 59. Back Up `/etc/fstab`

Before editing:

```bash
sudo cp /etc/fstab /etc/fstab.backup
```

Verify:

```bash
ls -l /etc/fstab*
```

This gives you a copy to reference or restore if needed.

Do not treat a local backup as a complete recovery strategy for a server that may fail to boot. Remote systems should also have console or recovery access.

---

# 60. Editing `/etc/fstab`

For example:

```bash
sudo nano /etc/fstab
```

Add the correct entry.

Be extremely careful with:

* UUID.
* Mount point.
* Filesystem type.
* Options.
* Spaces.
* Field count.

Do not invent values.

Obtain them from the system.

---

# 61. Test fstab Before Reboot

After editing, test the configuration.

A common command is:

```bash
sudo mount -a
```

This attempts to mount filesystems configured for mounting in `/etc/fstab`, subject to their options.

If an error appears, investigate it.

Do not reboot until you understand and fix the error.

---

# 62. Verify After `mount -a`

Run:

```bash
findmnt /data
```

Then:

```bash
df -h /data
```

Then:

```bash
lsblk -f
```

Confirm:

```text
Correct filesystem?
Correct mount point?
Correct device?
Expected capacity?
```

---

# 63. Why Incorrect fstab Entries Are Dangerous

Imagine:

```text
UUID=WRONG /data ext4 defaults 0 2
```

Linux may fail to find the filesystem during boot.

Depending on system configuration, boot may:

* Pause.
* Enter emergency mode.
* Delay while waiting.
* Require manual intervention.
* Leave an expected application filesystem unavailable.

This can turn a simple typo into an outage.

---

# 64. Remote Server fstab Safety

On a remote server, ask:

```text
If this server does not boot correctly, how will I recover it?
```

Possible recovery methods include:

* Cloud console.
* Serial console.
* Rescue mode.
* Attaching the disk to another VM.
* Snapshot restoration.
* Hypervisor console.

Do not make boot-critical storage changes without a recovery plan.

---

# 65. Persistent Mount Troubleshooting

Suppose `/data` works now but disappears after reboot.

Check:

```bash
findmnt /data
```

Then:

```bash
grep -v '^#' /etc/fstab
```

Check:

```text
Is an entry present?
Is the UUID correct?
Does the mount point exist?
Is the filesystem type correct?
```

Then compare:

```bash
sudo blkid
```

with the UUID in `/etc/fstab`.

---

# 66. `df`

`df` reports filesystem disk-space usage.

Run:

```bash
df
```

Output may use blocks that are less convenient to read.

Most administrators commonly use:

```bash
df -h
```

---

# 67. Human-Readable `df`

The:

```text
-h
```

means:

```text
human-readable
```

Run:

```bash
df -h
```

Example:

```text
Filesystem      Size  Used Avail Use% Mounted on
/dev/sda1        40G   22G   16G  58% /
/dev/sdb1       100G   81G   19G  81% /data
```

Now you can quickly see capacity.

---

# 68. Reading `df -h`

Important columns:

```text
Filesystem
Size
Used
Avail
Use%
Mounted on
```

Example:

```text
/dev/sdb1 100G 81G 19G 81% /data
```

means approximately:

```text
Filesystem:
/dev/sdb1

Total:
100 GB

Used:
81 GB

Available:
19 GB

Usage:
81%

Mounted at:
/data
```

Exact filesystem accounting can make totals differ slightly from simple arithmetic.

---

# 69. Check a Specific Filesystem

Instead of reading everything:

```bash
df -h /data
```

This shows the filesystem containing `/data`.

Similarly:

```bash
df -h /var/log
```

shows the filesystem containing `/var/log`.

This is very useful during incidents.

---

# 70. `df` Reports Filesystem Usage

This distinction matters.

`df` answers:

```text
How much capacity does this filesystem have and how much is available?
```

It does not directly tell you:

```text
Which directories consumed the space?
```

For that, use:

```text
du
```

---

# 71. `du`

`du` means:

```text
disk usage
```

It estimates space used by files and directories.

Example:

```bash
du /var/log
```

The output may be large and difficult to read.

Use:

```bash
du -h /var/log
```

for human-readable output.

---

# 72. Directory Size With `du -sh`

A very useful pattern:

```bash
du -sh /var/log
```

Here:

```text
-s
=
summary

-h
=
human-readable
```

Possible output:

```text
2.4G    /var/log
```

This tells you the total estimated usage of that directory.

---

# 73. Compare Directory Sizes

Suppose `/var` is large.

Run:

```bash
sudo du -sh /var/*
```

You might see:

```text
120M    /var/cache
6.8G    /var/lib
12G     /var/log
40M     /var/tmp
```

Now:

```text
/var/log
```

looks suspicious.

Continue drilling down.

---

# 74. Drill Down Into Large Directories

Run:

```bash
sudo du -sh /var/log/*
```

Suppose:

```text
50M     /var/log/apt
11G     /var/log/myapp
500M    /var/log/journal
```

Now investigate:

```bash
sudo du -sh /var/log/myapp/*
```

This is a systematic storage troubleshooting method:

```text
Filesystem full
      |
      v
Find large top-level directory
      |
      v
Drill down
      |
      v
Find large subdirectory
      |
      v
Find large files
```

---

# 75. `du --max-depth`

On GNU systems, you can use:

```bash
sudo du -h --max-depth=1 /var
```

This shows one level of directory usage.

Example:

```text
12G /var/log
7G  /var/lib
...
```

Then inspect the largest one.

---

# 76. Finding Large Directories With `sort`

Combine commands from Part 7:

```bash
sudo du -h --max-depth=1 /var | sort -h
```

What happens?

```text
du
 |
 | directory sizes
 v
sort -h
 |
 | sorts human-readable sizes
 v
ordered output
```

To see largest entries first:

```bash
sudo du -h --max-depth=1 /var | sort -hr
```

---

# 77. Finding Large Files

You can use `find`.

Example:

```bash
sudo find /var -type f -size +1G -print
```

Breakdown:

```text
/var
=
search location

-type f
=
files only

-size +1G
=
larger than 1 GiB

-print
=
display matching paths
```

Do not automatically delete whatever appears.

Investigate what each file is used for.

---

# 78. Finding Large Files With Sizes

A useful workflow might be:

```bash
sudo find /var -type f -size +500M -exec ls -lh {} \;
```

This displays file details.

But remember:

```text
Large file
```

does not automatically mean:

```text
Safe to delete.
```

A large file might be:

* Database file.
* Application data.
* Container image layer.
* Active log.
* Backup.
* Swap file.
* Package cache.
* Virtual disk.

Understand it first.

---

# 79. `df` vs `du`

This distinction is fundamental.

```text
df
=
Filesystem-level capacity

du
=
File/directory-level usage
```

During a disk-full incident:

```bash
df -h
```

answers:

```text
Which filesystem is full?
```

Then:

```bash
du
```

helps answer:

```text
Which visible files/directories are consuming space?
```

---

# 80. Why `df` and `du` Can Disagree

Sometimes:

```bash
df -h
```

reports a filesystem is nearly full.

But:

```bash
du
```

cannot account for all the used space.

One important cause is:

```text
Deleted file still held open by a running process.
```

---

# 81. Deleted But Open Files

Imagine an application has a 10 GB log open.

Someone runs:

```bash
rm application.log
```

The directory entry disappears.

But the process still has the file open.

Linux may keep the underlying storage allocated until the process closes the file.

Conceptually:

```text
10 GB log
   |
   +-- directory name deleted
   |
   +-- process still holds file open
               |
               v
        space still allocated
```

---

# 82. Finding Deleted Open Files

A common investigation command is:

```bash
sudo lsof +L1
```

or:

```bash
sudo lsof | grep '(deleted)'
```

You may find a process holding a deleted large file.

Do not immediately kill the process.

Determine:

* Which service owns it.
* Whether it can be safely restarted or reloaded.
* Why the file became so large.
* Whether log rotation is configured correctly.

---

# 83. What Is an Inode?

A filesystem needs metadata structures to describe files.

On filesystems such as ext4, these structures include inodes.

An inode stores metadata about a filesystem object.

At a beginner level, think:

```text
File name
    |
    v
Directory entry
    |
    v
Inode
    |
    +-- ownership
    +-- permissions
    +-- timestamps
    +-- size
    +-- pointers/references to data
```

The filename itself is associated through directory entries rather than being the inode's core identity.

---

# 84. Why Inodes Matter

A filesystem can have:

```text
Available disk capacity
```

but no:

```text
Available inodes
```

If that happens, creating new files can fail.

This often occurs when a filesystem contains enormous numbers of tiny files.

For example:

```text
Millions of tiny cache files
```

may consume inodes before consuming all available bytes.

---

# 85. Check Inodes With `df -i`

Run:

```bash
df -i
```

Example:

```text
Filesystem      Inodes   IUsed   IFree IUse% Mounted on
/dev/sda1      2621440 2500000 121440   96% /
```

Important columns:

```text
Inodes
IUsed
IFree
IUse%
```

If:

```text
IUse%
```

reaches:

```text
100%
```

the filesystem may be unable to create new files.

---

# 86. Disk Capacity vs Inode Capacity

Compare:

```bash
df -h
```

with:

```bash
df -i
```

Scenario:

```text
df -h:
60% used

df -i:
100% used
```

The problem is not byte capacity.

The problem is inode exhaustion.

This is why:

```text
No space left on device
```

does not automatically mean the disk has zero free bytes.

---

# 87. Inode Exhaustion Example

Suppose an application creates:

```text
/cache/item-000001
/cache/item-000002
/cache/item-000003
...
```

Eventually there are millions of tiny files.

Each file consumes filesystem metadata resources.

The total data may only be a few gigabytes.

But the filesystem can run out of available inodes.

---

# 88. Finding Directories With Many Files

One simple investigation:

```bash
sudo find /var -xdev -type f | wc -l
```

This counts files within the same filesystem under `/var`, assuming `/var` is within the filesystem being investigated.

You can inspect suspicious directories individually.

For example:

```bash
find /var/cache/myapp -type f | wc -l
```

If you see:

```text
2800000
```

that directory deserves investigation.

---

# 89. Disk-Full Incident

Suppose an application fails with:

```text
No space left on device
```

Do not immediately delete files.

Use a systematic process.

Start:

```bash
df -h
```

Question:

```text
Which filesystem is full?
```

---

# 90. Disk-Full Investigation Step 1

Suppose:

```bash
df -h
```

shows:

```text
/dev/sdb1  100G  100G  0  100% /data
```

Now you know:

```text
/data
```

is on the affected filesystem.

Do not waste time deleting unrelated files from `/home` if `/home` belongs to another filesystem.

---

# 91. Disk-Full Investigation Step 2

Check inodes:

```bash
df -i /data
```

Suppose:

```text
IUse% = 20%
```

Inodes are not exhausted.

Continue investigating byte usage.

---

# 92. Disk-Full Investigation Step 3

Run:

```bash
sudo du -h --max-depth=1 /data | sort -hr
```

Suppose:

```text
90G /data/logs
5G  /data/app
3G  /data/backups
```

Now investigate:

```bash
sudo du -h --max-depth=1 /data/logs | sort -hr
```

---

# 93. Disk-Full Investigation Step 4

Suppose:

```text
/data/logs/payment
```

uses:

```text
85G
```

Find large files:

```bash
sudo find /data/logs/payment -type f -size +1G -exec ls -lh {} \;
```

Suppose:

```text
82G payment-debug.log
```

Now you have evidence.

---

# 94. Do Not Immediately Run `rm`

Ask:

```text
Why is this file 82 GB?
```

Possible reasons:

* Debug logging accidentally enabled.
* Log rotation failed.
* Application loop generated repeated errors.
* Log retention misconfigured.
* Service writes excessively.
* Rotation configuration points to wrong file.

Deleting the file may provide temporary relief while leaving the root cause intact.

---

# 95. Safer Disk-Full Response

A good process is:

```text
Observe
  |
  v
Confirm affected filesystem
  |
  v
Check bytes
  |
  v
Check inodes
  |
  v
Identify largest consumers
  |
  v
Understand ownership/purpose
  |
  v
Choose safe remediation
  |
  v
Verify recovered capacity
  |
  v
Fix root cause
  |
  v
Prevent recurrence
```

This is much better than:

```text
Disk full
   |
   v
rm random files
```

---

# 96. Safe Ways to Recover Space

Depending on the cause and organizational policy, options may include:

* Rotate logs.
* Remove approved old logs.
* Clean temporary data.
* Remove obsolete caches.
* Move archived data.
* Increase storage capacity.
* Correct retention policies.
* Fix runaway logging.
* Clean failed application artifacts.
* Remove unused packages through the package manager where appropriate.

The correct action depends on what is consuming space.

---

# 97. Never Randomly Delete From System Directories

Do not blindly delete files from:

```text
/var/lib
/usr
/etc
/boot
```

because they appear large.

For example:

```text
/var/lib
```

may contain:

* Databases.
* Package databases.
* Container runtime data.
* Application state.

Deleting the wrong data can destroy services.

---

# 98. Inode Exhaustion Incident

Suppose:

```bash
df -h /data
```

shows:

```text
45% used
```

but applications receive:

```text
No space left on device
```

Check:

```bash
df -i /data
```

Suppose:

```text
IUse% 100%
```

Now the root problem is:

```text
inode exhaustion
```

not normal capacity exhaustion.

---

# 99. Investigating Inode Exhaustion

Look for directories containing huge numbers of files.

For a suspected directory:

```bash
find /data/cache -xdev -type f | wc -l
```

Then inspect subdirectories.

A practical approach can be:

```bash
for dir in /data/*; do
    printf '%s: ' "$dir"
    find "$dir" -xdev -type f 2>/dev/null | wc -l
done
```

For a beginner, understand the logic before using it:

```text
For every top-level item under /data
        |
        v
count files within it
        |
        v
compare counts
```

Do not run broad `find` operations carelessly on extremely large production filesystems. They can be expensive.

---

# 100. Preventing Inode Exhaustion

Possible preventive measures include:

* Proper cache cleanup.
* Retention policies.
* Log rotation.
* Application design changes.
* Monitoring inode usage.
* Choosing suitable filesystem parameters.
* Avoiding uncontrolled temporary-file creation.

Monitoring only:

```text
disk percentage
```

may miss inode exhaustion.

Monitor both:

```text
capacity
+
inodes
```

where relevant.

---

# 101. Mount Failures

Suppose:

```bash
sudo mount /dev/sdb1 /data
```

fails.

Possible causes include:

* Device does not exist.
* Wrong partition.
* Filesystem missing.
* Filesystem damaged.
* Wrong filesystem type.
* Mount point missing.
* Device already mounted.
* Incorrect mount options.
* Storage unavailable.

Do not repeatedly rerun the same command without reading the error.

---

# 102. Mount Troubleshooting Step 1: Confirm Device

Run:

```bash
lsblk -f
```

Does:

```text
/dev/sdb1
```

actually exist?

If not, the command is targeting the wrong device.

---

# 103. Mount Troubleshooting Step 2: Check Filesystem

Run:

```bash
sudo blkid /dev/sdb1
```

Does it show:

```text
TYPE="ext4"
```

or:

```text
TYPE="xfs"
```

If no recognizable filesystem exists, investigate whether the device is intentionally unformatted.

Do not immediately run `mkfs`.

---

# 104. Mount Troubleshooting Step 3: Check Mount Point

Run:

```bash
ls -ld /data
```

If:

```text
No such file or directory
```

and `/data` is the intended mount point:

```bash
sudo mkdir -p /data
```

Then retry.

---

# 105. Mount Troubleshooting Step 4: Check Existing Mounts

Run:

```bash
findmnt /dev/sdb1
```

or:

```bash
mount | grep '/dev/sdb1'
```

The filesystem may already be mounted somewhere else.

Do not accidentally mount the same filesystem in multiple places without understanding the consequences.

---

# 106. Mount Troubleshooting Step 5: Check Kernel Messages

If mounting fails because of a filesystem or device problem, recent kernel messages may provide clues.

Use:

```bash
dmesg | tail -n 30
```

or on systems where access requires privilege:

```bash
sudo dmesg | tail -n 30
```

Look for storage-related errors.

Do not modify or format the filesystem just because you see an error.

Preserve evidence and determine the cause.

---

# 107. Read-Only Filesystem

Filesystems can be mounted:

```text
read-write
```

or:

```text
read-only
```

Read-write means files can normally be modified according to permissions.

Read-only means modifications are not allowed through that mount.

You may see:

```text
Read-only file system
```

when attempting to create or modify files.

---

# 108. Read-Only Is Different From Permission Denied

Compare:

```text
Permission denied
```

with:

```text
Read-only file system
```

`Permission denied` may indicate:

* User permissions.
* Ownership.
* ACLs.
* Security controls.

`Read-only file system` means the filesystem or mount itself is not permitting writes.

Do not immediately run:

```bash
chmod 777
```

It will not fix a filesystem mounted read-only.

---

# 109. Check Mount Options

Use:

```bash
findmnt -no TARGET,SOURCE,FSTYPE,OPTIONS /data
```

You may see options containing:

```text
rw
```

meaning read-write.

Or:

```text
ro
```

meaning read-only.

Example:

```text
/data /dev/sdb1 ext4 rw,relatime
```

---

# 110. Why a Filesystem May Become Read-Only

Possible causes include:

* It was intentionally mounted read-only.
* `/etc/fstab` specifies read-only options.
* Filesystem errors occurred.
* Storage problems occurred.
* Kernel remounted it read-only to protect consistency.
* Underlying storage became unhealthy.

A filesystem unexpectedly becoming read-only can be a serious incident.

---

# 111. Do Not Blindly Remount Read-Write

You may find commands online such as:

```bash
sudo mount -o remount,rw /data
```

Do not blindly use this when the filesystem unexpectedly became read-only.

First ask:

```text
Why did it become read-only?
```

If Linux changed it to read-only because of filesystem or storage errors, forcing writes may worsen the problem.

Investigate:

```bash
findmnt /data
```

and:

```bash
sudo dmesg | tail -n 100
```

Then follow the appropriate filesystem and storage recovery process.

---

# 112. Filesystem Checking Introduction

For ext-family filesystems, you may encounter:

```text
fsck
```

which means:

```text
filesystem check
```

For XFS, you may encounter tools such as:

```text
xfs_repair
```

These are repair tools.

They should not be used casually on mounted production filesystems.

Recovery depends on:

* Filesystem type.
* Whether it is mounted.
* Whether storage is healthy.
* Backup availability.
* Recovery procedures.

This part introduces the concept. Filesystem repair deserves deeper study separately.

---

# 113. Safe Storage Practices

Storage commands can be destructive.

Before changing storage, follow this process:

```text
Identify
   |
   v
Verify
   |
   v
Understand existing state
   |
   v
Confirm backups/recovery
   |
   v
Make smallest necessary change
   |
   v
Verify
```

---

# 114. Safe Practice 1: Inspect Before Acting

Before partitioning:

```bash
lsblk -f
```

Before formatting:

```bash
lsblk -f
sudo blkid
```

Before mounting:

```bash
findmnt
```

Before deleting data:

```bash
du
```

Before editing fstab:

```bash
sudo cp /etc/fstab /etc/fstab.backup
```

Evidence first.

Changes second.

---

# 115. Safe Practice 2: Never Assume Device Names

Do not say:

```text
The new disk is always /dev/sdb.
```

It may not be.

Use:

```bash
lsblk
```

and compare:

* Size.
* Device type.
* Existing partitions.
* Filesystem.
* Mount point.

Cloud and virtual environments may present devices differently.

---

# 116. Safe Practice 3: Verify Before `mkfs`

Before:

```bash
sudo mkfs.ext4 /dev/sdb1
```

verify:

```bash
lsblk -f /dev/sdb
```

and:

```bash
sudo blkid /dev/sdb1
```

Ask:

```text
Is this definitely the disposable target?

Does it contain existing data?

Is it mounted?

Do I have a recovery path?
```

Formatting the wrong device can be catastrophic.

---

# 117. Safe Practice 4: Test fstab Before Reboot

After changing `/etc/fstab`:

```bash
sudo mount -a
```

Investigate every unexpected error.

Then:

```bash
findmnt
```

and:

```bash
df -h
```

Verify the expected mount.

Do not make a typo and discover it only after rebooting a remote production server.

---

# 118. Safe Practice 5: Do Not Delete Before Understanding

Suppose:

```text
/var/lib/docker
```

uses 70 GB.

Do not run:

```bash
sudo rm -rf /var/lib/docker/*
```

because you want disk space.

That directory may contain critical container runtime state.

Use the application's supported cleanup mechanisms where appropriate.

The same principle applies to:

* Databases.
* Package managers.
* Logging systems.
* Container runtimes.
* Monitoring systems.

---

# 119. Safe Practice 6: Monitor Storage

Production storage should not be discovered only after:

```text
100% full
```

Useful metrics include:

```text
Filesystem usage %
Available bytes
Inode usage %
Filesystem read-only state
Storage latency
Storage errors
```

You will later connect these ideas to monitoring tools such as Prometheus and cloud monitoring systems.

---

# 120. Storage Troubleshooting Mental Model

When someone says:

```text
"The disk is broken."
```

do not accept that as a diagnosis.

Break the problem into layers:

```text
Application
    |
    v
Path
    |
    v
Mount point
    |
    v
Filesystem
    |
    v
Partition
    |
    v
Block device
    |
    v
Underlying storage
```

Then determine which layer is failing.

---

# 121. Problem: "No Space Left on Device"

Start:

```bash
df -h
```

Then:

```bash
df -i
```

If bytes are exhausted:

```text
Find large consumers with du/find.
```

If inodes are exhausted:

```text
Find directories containing huge numbers of files.
```

If neither appears exhausted:

```text
Investigate mount state, quotas where applicable, deleted-open files, application behavior, and filesystem/storage errors.
```

Do not guess.

---

# 122. Problem: "My Data Disappeared"

First ask:

```text
Was another filesystem mounted over the directory?
```

Check:

```bash
findmnt
```

and:

```bash
lsblk -f
```

A mount can hide files beneath the mount point.

Do not immediately conclude the data was deleted.

---

# 123. Problem: "The Disk Works Until Reboot"

Check:

```bash
cat /etc/fstab
```

Then:

```bash
sudo blkid
```

Compare UUIDs.

Check:

```bash
findmnt
```

Likely possibilities include:

* No fstab entry.
* Wrong UUID.
* Wrong filesystem type.
* Wrong mount point.
* Mount dependency/timing issue.
* Invalid options.

---

# 124. Problem: "I Cannot Unmount"

Run:

```bash
pwd
```

Are you currently inside the mount?

Then:

```bash
fuser -vm /data
```

or appropriate `lsof` inspection.

Identify who is using the filesystem.

Do not immediately use forceful unmount techniques.

---

# 125. Problem: "Read-Only File System"

Check:

```bash
findmnt /data
```

Then:

```bash
sudo dmesg | tail -n 100
```

Ask:

```text
Was it intentionally mounted ro?

Did the kernel detect filesystem errors?

Did the storage layer report errors?

Was fstab configured incorrectly?
```

Do not try to solve it with `chmod`.

---

# 126. Problem: "df Says Full, du Does Not"

Check:

```bash
sudo lsof +L1
```

Look for deleted files still held open.

Also verify that you are comparing:

```text
the same filesystem
```

and not crossing other mounted filesystems during `du` calculations.

Storage troubleshooting requires precise scope.

---

# 127. Storage Troubleshooting Lab

This lab should be performed on a disposable virtual machine or cloud test server with an additional disposable disk.

Do not practice partition creation or formatting on a disk containing important data.

Assume your lab has:

```text
System disk
+
New disposable 5 GB disk
```

The device name below will use:

```text
/dev/sdb
```

only as an example.

Your actual device may be different.

---

# 128. Lab Step 1: Inspect Existing Storage

Run:

```bash
lsblk
```

Then:

```bash
lsblk -f
```

Then:

```bash
df -h
```

Record:

```text
Root device:
Root filesystem:
Root filesystem size:
Root mount point:
Additional disk:
```

Do not continue until you can identify the system disk and test disk.

---

# 129. Lab Step 2: Inspect UUIDs

Run:

```bash
sudo blkid
```

Compare the output with:

```bash
lsblk -f
```

Identify:

```text
Device
Filesystem type
UUID
Mount point
```

for existing filesystems.

---

# 130. Lab Step 3: Inspect Partition Table

For your verified disposable disk:

```bash
sudo fdisk -l /dev/sdb
```

Replace `/dev/sdb` with your actual test device.

Observe:

* Disk size.
* Partition-table type.
* Existing partitions.

If the disk contains unexpected partitions or data, stop.

---

# 131. Lab Step 4: Create a Partition

Only on the verified disposable disk:

```bash
sudo fdisk /dev/sdb
```

Inside:

```text
p
```

Inspect current layout.

Then:

```text
n
```

Create a partition.

For a simple lab, accept appropriate defaults to use available space.

Inspect again:

```text
p
```

If correct:

```text
w
```

writes the changes.

If anything looks wrong:

```text
q
```

exits without intentionally writing your proposed changes.

---

# 132. Lab Step 5: Verify the Partition

Run:

```bash
lsblk
```

You should now see something such as:

```text
sdb
└─sdb1
```

Confirm:

```text
Disk:
/dev/sdb

Partition:
/dev/sdb1
```

Do not format until you have verified this.

---

# 133. Lab Step 6: Create a Filesystem

For the disposable partition:

```bash
sudo mkfs.ext4 /dev/sdb1
```

Read the output.

Then verify:

```bash
lsblk -f
```

You should see:

```text
sdb1
ext4
```

and a UUID.

---

# 134. Lab Step 7: Inspect Filesystem Metadata

Run:

```bash
sudo blkid /dev/sdb1
```

Record:

```text
Filesystem:
UUID:
```

You will use this later.

---

# 135. Lab Step 8: Create Mount Point

Run:

```bash
sudo mkdir -p /veriqta-storage-lab
```

Verify:

```bash
ls -ld /veriqta-storage-lab
```

At this stage, it is simply a directory.

---

# 136. Lab Step 9: Mount the Filesystem

Run:

```bash
sudo mount /dev/sdb1 /veriqta-storage-lab
```

Verify:

```bash
findmnt /veriqta-storage-lab
```

Then:

```bash
lsblk
```

Then:

```bash
df -h /veriqta-storage-lab
```

You have now connected:

```text
/dev/sdb1
     |
     v
/veriqta-storage-lab
```

---

# 137. Lab Step 10: Create Data

The mount point may initially be owned by root.

For a simple lab, create directories with sudo:

```bash
sudo mkdir -p /veriqta-storage-lab/{logs,data,backups}
```

Create files:

```bash
sudo touch /veriqta-storage-lab/logs/app.log
sudo touch /veriqta-storage-lab/data/users.dat
sudo touch /veriqta-storage-lab/backups/backup.txt
```

Inspect:

```bash
find /veriqta-storage-lab -maxdepth 2 -ls
```

---

# 138. Lab Step 11: Check Capacity

Run:

```bash
df -h /veriqta-storage-lab
```

Record:

```text
Size:
Used:
Available:
Use%:
```

Then:

```bash
sudo du -sh /veriqta-storage-lab
```

Explain why `df` and `du` are answering different questions.

---

# 139. Lab Step 12: Check Inodes

Run:

```bash
df -i /veriqta-storage-lab
```

Record:

```text
Total inodes:
Used:
Free:
Usage percentage:
```

Now you have checked both:

```text
byte capacity
```

and:

```text
inode capacity
```

---

# 140. Lab Step 13: Practice Directory Analysis

Create some test data safely:

```bash
sudo mkdir -p /veriqta-storage-lab/data/project{1,2,3}
```

Create sample files:

```bash
sudo truncate -s 10M /veriqta-storage-lab/data/project1/file1.bin
sudo truncate -s 20M /veriqta-storage-lab/data/project2/file2.bin
sudo truncate -s 30M /veriqta-storage-lab/data/project3/file3.bin
```

`truncate -s 10M` sets the apparent file size.

Note that sparse-file behavior may mean physical disk consumption differs from the apparent size, depending on how the file is created and filesystem behavior.

For actual allocation practice, you can use controlled test-data creation tools, but avoid filling filesystems unnecessarily.

---

# 141. Lab Step 14: Find the Largest Directory

Run:

```bash
sudo du -h --max-depth=1 /veriqta-storage-lab/data | sort -hr
```

Identify the largest directory.

Then:

```bash
sudo du -h --max-depth=1 /veriqta-storage-lab/data/project3
```

Practice drilling down.

---

# 142. Lab Step 15: Unmount

First move outside:

```bash
cd /
```

Then:

```bash
sudo umount /veriqta-storage-lab
```

Verify:

```bash
findmnt /veriqta-storage-lab
```

You should no longer see an active mount there.

---

# 143. Lab Step 16: Confirm Mount Point Still Exists

Run:

```bash
ls -ld /veriqta-storage-lab
```

The directory still exists.

But:

```bash
ls -la /veriqta-storage-lab
```

will no longer show the files from `/dev/sdb1`.

Those files were not deleted.

They are stored in the unmounted filesystem.

---

# 144. Lab Step 17: Mount Again

Run:

```bash
sudo mount /dev/sdb1 /veriqta-storage-lab
```

Then:

```bash
ls -la /veriqta-storage-lab
```

Your directories return.

This demonstrates an important principle:

```text
Unmounting does not delete the filesystem's files.
```

It removes the filesystem from that location in the directory tree.

---

# 145. Lab Step 18: Practice a Busy Mount

Run:

```bash
cd /veriqta-storage-lab
```

Then attempt:

```bash
sudo umount /veriqta-storage-lab
```

It may report:

```text
target is busy
```

Check:

```bash
pwd
```

Then:

```bash
cd /
```

Retry:

```bash
sudo umount /veriqta-storage-lab
```

You have now intentionally created and fixed a common mount failure.

---

# 146. Lab Step 19: Prepare Persistent Mount

Obtain UUID:

```bash
sudo blkid /dev/sdb1
```

Copy the exact UUID.

Back up:

```bash
sudo cp /etc/fstab /etc/fstab.backup
```

Verify:

```bash
ls -l /etc/fstab*
```

---

# 147. Lab Step 20: Add fstab Entry

Edit:

```bash
sudo nano /etc/fstab
```

Add:

```text
UUID=YOUR-REAL-UUID /veriqta-storage-lab ext4 defaults 0 2
```

Replace:

```text
YOUR-REAL-UUID
```

with the actual value.

Do not literally enter:

```text
YOUR-REAL-UUID
```

---

# 148. Lab Step 21: Test fstab

Make sure the filesystem is currently unmounted:

```bash
sudo umount /veriqta-storage-lab
```

Then:

```bash
sudo mount -a
```

Read all output.

If no error appears, verify:

```bash
findmnt /veriqta-storage-lab
```

Then:

```bash
df -h /veriqta-storage-lab
```

Then:

```bash
ls /veriqta-storage-lab
```

---

# 149. Lab Step 22: Do Not Intentionally Break Boot

You may be tempted to intentionally put a bad UUID into `/etc/fstab` and reboot.

Do not do this on a machine where you do not have reliable console recovery.

Instead, if you want to study an incorrect entry, make the change temporarily and test it using:

```bash
sudo mount -a
```

Then immediately restore the valid configuration without rebooting.

The objective is to learn safely, not manufacture an unnecessary boot outage.

---

# 150. Lab Step 23: Simulate fstab Error Safely

Back up your valid file first.

Temporarily change the lab UUID to an invalid value.

Run:

```bash
sudo mount -a
```

Observe the error.

Then immediately restore:

```bash
sudo cp /etc/fstab.backup /etc/fstab
```

But be careful.

If your backup was created before adding the valid lab entry, restoring it will remove the lab entry entirely.

Alternatively, manually restore the valid line.

Then run:

```bash
sudo mount -a
```

again.

Verify there are no unexpected errors.

---

# 151. Lab Step 24: Practice Storage Investigation

Run:

```bash
df -h
```

Identify the filesystem with the highest percentage usage.

Do not delete anything.

For a safe lab path, run:

```bash
sudo du -h --max-depth=1 /veriqta-storage-lab | sort -hr
```

Then identify the largest subdirectory.

This practices investigation without destructive remediation.

---

# 152. Lab Step 25: Practice Inode Investigation

Run:

```bash
df -i /veriqta-storage-lab
```

Create a test directory:

```bash
sudo mkdir -p /veriqta-storage-lab/inode-test
```

Create 1,000 empty files:

```bash
sudo bash -c 'for i in $(seq 1 1000); do touch "/veriqta-storage-lab/inode-test/file-$i"; done'
```

Then:

```bash
df -i /veriqta-storage-lab
```

Observe the change in used inodes.

---

# 153. Lab Step 26: Count the Files

Run:

```bash
find /veriqta-storage-lab/inode-test -type f | wc -l
```

Expected:

```text
1000
```

Now you can see the relationship:

```text
Many files
   |
   v
More filesystem metadata/inodes consumed
```

---

# 154. Lab Step 27: Clean Up the Inode Test

Verify the path carefully:

```bash
ls -ld /veriqta-storage-lab/inode-test
```

Then:

```bash
sudo rm -r /veriqta-storage-lab/inode-test
```

Verify:

```bash
test ! -e /veriqta-storage-lab/inode-test && echo "Removed"
```

Then:

```bash
df -i /veriqta-storage-lab
```

Never use:

```bash
rm -rf
```

automatically just because cleanup is needed.

---

# 155. Lab Step 28: Inspect Mount Options

Run:

```bash
findmnt -no SOURCE,TARGET,FSTYPE,OPTIONS /veriqta-storage-lab
```

Identify:

```text
Source:
Target:
Filesystem:
Options:
```

Look for:

```text
rw
```

which indicates read-write.

---

# 156. Lab Step 29: Final Verification

Run:

```bash
lsblk -f
```

Then:

```bash
findmnt /veriqta-storage-lab
```

Then:

```bash
df -h /veriqta-storage-lab
```

Then:

```bash
df -i /veriqta-storage-lab
```

Then:

```bash
sudo du -sh /veriqta-storage-lab
```

You should now understand what each command tells you.

---

# 157. Real-World Scenario: Production Disk Full

You receive an alert:

```text
Filesystem /var is 97% full.
```

Do not SSH in and start deleting files.

Begin:

```bash
df -h /var
```

Suppose:

```text
Filesystem      Size Used Avail Use% Mounted on
/dev/sdb1        50G  48G   2G   97% /var
```

Now check:

```bash
df -i /var
```

Suppose:

```text
IUse% 18%
```

This is capacity pressure, not inode exhaustion.

---

# 158. Find the Consumer

Run:

```bash
sudo du -h --max-depth=1 /var | sort -hr
```

Suppose:

```text
47G /var
35G /var/log
8G  /var/lib
2G  /var/cache
```

Continue:

```bash
sudo du -h --max-depth=1 /var/log | sort -hr
```

Suppose:

```text
30G /var/log/myapp
```

Continue.

---

# 159. Find the Large File

Run:

```bash
sudo find /var/log/myapp -type f -size +500M -exec ls -lh {} \;
```

You discover:

```text
29G /var/log/myapp/debug.log
```

Now inspect:

```bash
sudo tail -n 50 /var/log/myapp/debug.log
```

Suppose the same error appears thousands of times.

The root cause may be:

```text
Application entered an error loop while debug logging was enabled.
```

---

# 160. Correct Incident Thinking

The problem is not merely:

```text
Large log.
```

A stronger analysis is:

```text
Symptom:
Filesystem /var reached 97%.

Immediate cause:
29 GB application debug log.

Underlying cause:
Application repeatedly logged an error at debug level.

Control failure:
Log rotation or retention did not prevent uncontrolled growth.
```

Remediation may require:

```text
Restore safe free capacity
+
fix application behavior
+
fix logging configuration
+
verify rotation
+
add/verify monitoring
```

That is production reasoning.

---

# 161. Real-World Scenario: Inode Exhaustion

Alert:

```text
Application cannot create session files.
```

Application reports:

```text
No space left on device
```

You run:

```bash
df -h /var
```

Result:

```text
48% used
```

A beginner might say:

```text
The error makes no sense.
```

You know better.

Run:

```bash
df -i /var
```

Result:

```text
100% inode usage
```

Now investigate file counts.

---

# 162. Root Cause of Inode Incident

You discover:

```text
/var/cache/myapp
```

contains millions of tiny files.

Root cause:

```text
Cache cleanup job stopped running.
```

Correct response:

```text
Safely clean eligible cache data
+
restore inode availability
+
repair cleanup mechanism
+
monitor inode usage
```

Not:

```text
Increase disk size immediately.
```

Increasing bytes may not solve the actual design problem.

---

# 163. Real-World Scenario: Mount Missing After Reboot

Application fails after server restart.

Expected:

```text
/data
```

contains database data.

But:

```bash
findmnt /data
```

returns nothing.

Check:

```bash
cat /etc/fstab
```

No `/data` entry exists.

Root cause:

```text
Administrator mounted the volume manually but never configured persistence.
```

The original mount worked only until reboot.

---

# 164. Real-World Scenario: Wrong UUID

After reboot, a data filesystem does not mount.

Check:

```bash
grep '/data' /etc/fstab
```

Then:

```bash
sudo blkid
```

The UUIDs differ.

Root cause:

```text
fstab references the wrong filesystem UUID.
```

Correct the configuration carefully.

Test:

```bash
sudo mount -a
```

Then verify:

```bash
findmnt /data
```

---

# 165. Real-World Scenario: Data Appears Missing

Application team reports:

```text
Everything under /data disappeared.
```

You check:

```bash
findmnt /data
```

A newly attached empty filesystem is mounted there.

The original files may still exist beneath the mount.

After safely stopping anything using the new mount and following the change procedure, unmounting may reveal the original directory contents.

Lesson:

```text
Mounting over a populated directory can hide existing data.
```

---

# 166. Real-World Scenario: Filesystem Becomes Read-Only

Application starts failing:

```text
Read-only file system
```

Do not:

```bash
chmod 777
```

Check:

```bash
findmnt /data
```

Then:

```bash
sudo dmesg | tail -n 100
```

Suppose kernel messages show storage I/O errors.

Now the incident is potentially:

```text
storage/filesystem health
```

rather than:

```text
permissions
```

Escalate and recover according to the storage architecture and filesystem procedures.

---

# 167. Commands You Should Remember From Part 14

Inspection:

```bash
lsblk
lsblk -f
sudo blkid
sudo fdisk -l
findmnt
```

Capacity:

```bash
df -h
df -i
du -sh
```

Mounting:

```bash
mount
umount
```

Persistent configuration:

```text
/etc/fstab
```

Filesystem creation:

```bash
mkfs.ext4
mkfs.xfs
```

Troubleshooting:

```bash
lsof
fuser
dmesg
find
```

But remember:

```text
Knowing the command name is not the goal.

Knowing when and why to use it is the goal.
```

---

# 168. What You Should Now Understand

Before continuing to Part 15, you should confidently understand:

* What persistent storage is.
* Why Linux servers need storage.
* What a disk is.
* What a partition is.
* What a filesystem is.
* Why disk, partition, and filesystem are different concepts.
* What a block device is.
* What `/dev` represents.
* How Linux names common storage devices.
* The difference between `/dev/sda` and `/dev/sda1`.
* How NVMe naming differs.
* What `lsblk` does.
* How to read the `lsblk` hierarchy.
* What `lsblk -f` adds.
* What `blkid` does.
* What a partition table is.
* What MBR is at a beginner level.
* What GPT is at a beginner level.
* Why partitioning is potentially destructive.
* What `fdisk` does.
* Why you must verify a device before modifying it.
* What ext4 is.
* What XFS is.
* What journaling means at a beginner level.
* What `mkfs` does.
* Why `mkfs` is dangerous on an existing filesystem.
* What mounting means.
* What a mount point is.
* How to use `mount`.
* How to verify a mount.
* What `findmnt` does.
* What happens when you mount over a populated directory.
* What `umount` does.
* Why a mount may be busy.
* How to investigate a busy mount.
* Why manual mounts may disappear after reboot.
* What a UUID is.
* Why UUIDs are useful.
* What `/etc/fstab` is.
* What the six basic fstab fields represent.
* How persistent mounts work.
* Why incorrect fstab entries can cause outages.
* Why fstab should be backed up before modification.
* Why `mount -a` should be used to test appropriate fstab changes before reboot.
* What `df` reports.
* What `df -h` reports.
* What `du` reports.
* The difference between `df` and `du`.
* How to investigate large directories.
* How to find large files.
* Why large files should not automatically be deleted.
* Why `df` and `du` can disagree.
* What deleted-but-open files are.
* What an inode is.
* Why inodes matter.
* How to use `df -i`.
* What inode exhaustion is.
* Why `No space left on device` does not always mean byte capacity is exhausted.
* How to troubleshoot disk-full incidents.
* How to troubleshoot inode exhaustion.
* How to troubleshoot mount failures.
* What a read-only filesystem is.
* Why read-only filesystem errors differ from permission errors.
* Why an unexpectedly read-only filesystem can indicate a serious storage problem.
* Why storage changes require backups and recovery planning.
* Why production storage should be monitored.

---

# 169. Hands-On Practice

Complete these tasks on a disposable Linux environment:

1. Run `lsblk`.
2. Identify your operating-system disk.
3. Identify its partitions.
4. Run `lsblk -f`.
5. Identify filesystem types.
6. Identify mount points.
7. Run `sudo blkid`.
8. Identify filesystem UUIDs.
9. Explain the difference between a disk and partition.
10. Explain the difference between a partition and filesystem.
11. Explain what a block device is.
12. Identify a `/dev/sdX` device if present.
13. Identify an NVMe device if present.
14. Inspect a disposable disk's partition table.
15. Create a partition on a disposable disk.
16. Verify the partition.
17. Create an ext4 filesystem on it.
18. Verify the filesystem.
19. Record its UUID.
20. Create a mount point.
21. Mount the filesystem.
22. Verify with `findmnt`.
23. Verify with `lsblk`.
24. Verify with `df -h`.
25. Create directories on the mounted filesystem.
26. Create files.
27. Check their ownership.
28. Check their permissions.
29. Run `du -sh` against the mount.
30. Run `df -h` against the mount.
31. Explain why the results differ.
32. Run `df -i`.
33. Record inode usage.
34. Create multiple small files.
35. Observe inode usage change.
36. Remove the test files.
37. Observe inode usage again.
38. Unmount the filesystem.
39. Verify it is unmounted.
40. Confirm the mount-point directory still exists.
41. Mount the filesystem again.
42. Verify its files reappear.
43. Create a busy-mount condition safely.
44. Observe the unmount failure.
45. Fix the busy-mount condition.
46. Unmount successfully.
47. Back up `/etc/fstab`.
48. Inspect the six fields of an existing fstab entry.
49. Add the disposable filesystem using its UUID.
50. Test with `mount -a`.
51. Verify the persistent mount.
52. Compare the fstab UUID with `blkid`.
53. Find the largest directory in your lab filesystem.
54. Find large files in the lab.
55. Explain why large files should not automatically be deleted.
56. Use `findmnt` to inspect mount options.
57. Identify `rw` or `ro`.
58. Explain the difference between `Permission denied` and `Read-only file system`.
59. Explain how a mount can hide existing files.
60. Write your own disk-full troubleshooting checklist.

---

# 170. Beginner Knowledge Check

1. What is persistent storage?
2. What is a disk?
3. What is a partition?
4. What is a filesystem?
5. Are a disk and filesystem the same thing?
6. Can one disk contain multiple partitions?
7. What is a block device?
8. Where are device interfaces commonly represented in Linux?
9. What might `/dev/sda` represent?
10. What might `/dev/sda1` represent?
11. What does `/dev/nvme0n1` commonly represent?
12. What might `/dev/nvme0n1p1` represent?
13. What does `lsblk` do?
14. What does `lsblk -f` add?
15. What does `blkid` show?
16. What is a partition table?
17. What does MBR stand for?
18. What does GPT stand for?
19. Why is partitioning potentially dangerous?
20. What does `fdisk -l` do?
21. What is ext4?
22. What is XFS?
23. What is filesystem journaling?
24. Does journaling replace backups?
25. What does `mkfs` mean?
26. Why is `mkfs` dangerous?
27. What does mounting mean?
28. What is a mount point?
29. How do you mount `/dev/sdb1` on `/data`?
30. How do you verify the mount?
31. What does `findmnt` do?
32. What happens if you mount over a directory containing files?
33. Are the underlying files necessarily deleted?
34. What does `umount` do?
35. Why might `umount` report that the target is busy?
36. How can your current working directory prevent an unmount?
37. What tools can help identify processes using a mount?
38. Does a manual `mount` automatically persist after reboot?
39. What does UUID stand for?
40. Why are UUIDs useful for persistent mounts?
41. What is `/etc/fstab`?
42. What does the first fstab field identify?
43. What does the second field identify?
44. What does the third field identify?
45. What does the fourth field contain?
46. What do the final two fields broadly control?
47. Why can a bad fstab entry be dangerous?
48. Why should fstab be backed up before editing?
49. What does `mount -a` do?
50. Why should appropriate fstab changes be tested before reboot?
51. What does `df` report?
52. What does `df -h` add?
53. What does `du` report?
54. What does `du -sh` do?
55. What is the difference between `df` and `du`?
56. How can you find large directories?
57. How can `find` help during disk investigations?
58. Should you automatically delete the largest file?
59. Why might `df` and `du` disagree?
60. What is a deleted-but-open file?
61. How can `lsof` help find one?
62. What is an inode?
63. Why are inodes important?
64. What does `df -i` report?
65. Can a filesystem run out of inodes while still having free bytes?
66. What type of workload commonly causes inode exhaustion?
67. What error might inode exhaustion produce?
68. What should you check first during a disk-full incident?
69. Why should you check both `df -h` and `df -i`?
70. Why should you determine the affected filesystem before deleting anything?
71. What are common causes of mount failure?
72. How do you verify a device exists?
73. How do you identify its filesystem?
74. Why should you not immediately run `mkfs` when a mount fails?
75. What does a read-only filesystem mean?
76. Is read-only the same as permission denied?
77. What does `rw` mean in mount options?
78. What does `ro` mean?
79. Why might Linux unexpectedly remount a filesystem read-only?
80. Why is blindly remounting it read-write dangerous?
81. What can `dmesg` reveal during storage troubleshooting?
82. What is `fsck` used for at a high level?
83. Should filesystem repair tools be run casually?
84. Why should you verify device names before modifying storage?
85. Why should production storage changes have a recovery plan?
86. Why is `/var/lib` dangerous to clean blindly?
87. Why is deleting a large log not necessarily a complete solution?
88. What is log rotation intended to help control?
89. Why should inode usage be monitored?
90. What happens to files on an unmounted filesystem?
91. Does unmounting delete them?
92. Why might a mount disappear after reboot?
93. How can you make an appropriate mount persistent?
94. Why is the UUID often preferable to a raw device name in fstab?
95. What should you do after changing fstab?
96. What should you investigate if `/data` appears empty unexpectedly?
97. What should you investigate when `df` reports 100% usage?
98. What should you investigate when `df -i` reports 100%?
99. What should you investigate when a filesystem becomes read-only unexpectedly?
100. What is the most important rule of Linux storage administration?

---

# 171. Beginner Knowledge Check Answers

1. Storage that retains data beyond the immediate running state and power cycle.

2. A physical or virtual storage device presented to the operating system.

3. A logical region of a disk.

4. A structure that organizes and manages files, directories, metadata, and storage allocation.

5. No.

6. Yes.

7. A device that provides block-oriented storage access.

8. Under `/dev`.

9. A disk.

10. The first partition on that disk.

11. An NVMe device.

12. A partition on that NVMe device.

13. It lists block devices and their relationships.

14. Filesystem information such as filesystem type, UUID, and mount points.

15. Block-device filesystem metadata, including UUIDs and filesystem types.

16. Metadata describing how a disk is divided into partitions.

17. Master Boot Record.

18. GUID Partition Table.

19. Incorrect changes can destroy partition information or make data inaccessible.

20. It lists disk and partition information.

21. A widely used Linux filesystem.

22. A widely used high-performance Linux filesystem common in enterprise environments.

23. A mechanism for recording information about filesystem changes to help maintain or recover consistency.

24. No.

25. Make filesystem.

26. Creating a new filesystem on storage containing valuable data can destroy access to the existing filesystem and data.

27. Attaching a filesystem to a location in the Linux directory tree.

28. The directory where a filesystem becomes accessible.

29. `sudo mount /dev/sdb1 /data`.

30. With commands such as `findmnt`, `lsblk`, or `df`.

31. It displays mounted-filesystem relationships and mount information.

32. The existing files beneath that directory become hidden while the new filesystem is mounted there.

33. No.

34. It detaches a mounted filesystem from its mount point.

35. A process or shell is using the filesystem.

36. If your shell is currently inside the filesystem, it can keep the mount busy.

37. `lsof` and `fuser`.

38. No.

39. Universally Unique Identifier.

40. They provide a stable identifier for a filesystem.

41. The filesystem table used to define persistent and other configured mounts.

42. The filesystem or device.

43. The mount point.

44. The filesystem type.

45. Mount options.

46. Dump behavior and filesystem-check ordering where applicable.

47. It can cause mount failures, boot delays, or emergency/recovery situations.

48. To preserve a known previous configuration.

49. It attempts configured mounts from `/etc/fstab` according to their options.

50. To identify configuration mistakes before a reboot makes recovery more difficult.

51. Filesystem capacity and usage.

52. Human-readable sizes.

53. File and directory disk usage.

54. Displays a summarized human-readable usage estimate.

55. `df` reports filesystem-level capacity. `du` measures visible file/directory usage.

56. Use `du`, often combined with `sort`.

57. It can locate files matching size and other criteria.

58. No. Determine what the file is and why it is large.

59. One reason is deleted files still held open by running processes. Mount boundaries and filesystem accounting can also matter.

60. A file whose directory entry was removed while a process still holds it open.

61. It can show open files, including deleted files still held by processes.

62. A filesystem metadata structure used to describe filesystem objects on filesystems that use inodes.

63. Files require filesystem metadata resources, and those resources can be exhausted.

64. Inode usage.

65. Yes.

66. Workloads that create enormous numbers of small files.

67. `No space left on device`.

68. Identify the affected filesystem with `df -h`.

69. One checks byte capacity. The other checks inode capacity.

70. Deleting from another filesystem will not solve the affected filesystem's capacity problem.

71. Wrong device, missing filesystem, incorrect filesystem type, missing mount point, filesystem damage, unavailable storage, or incorrect options.

72. `lsblk`.

73. `lsblk -f` or `blkid`.

74. The device may contain an existing filesystem and valuable data.

75. The filesystem is mounted or operating without allowing normal writes.

76. No.

77. Read-write.

78. Read-only.

79. Filesystem or storage errors may cause protective read-only behavior, among other causes.

80. The underlying filesystem or storage may be unhealthy, and forcing writes can worsen the situation.

81. Kernel-level storage, filesystem, and device messages.

82. Filesystem checking and, depending on filesystem and mode, repair-related operations.

83. No.

84. Modifying the wrong device can destroy data or make a system unbootable.

85. Storage mistakes can make data inaccessible or prevent systems from booting.

86. It often contains important application and system state.

87. The underlying application or log-management problem may continue and refill the filesystem.

88. Uncontrolled accumulation of log data.

89. A filesystem can exhaust inodes before exhausting byte capacity.

90. They remain on the filesystem.

91. No.

92. It may have been mounted manually without persistent configuration, or persistent configuration may have failed.

93. Configure it appropriately in `/etc/fstab` or the relevant system mount mechanism.

94. A filesystem UUID is generally more stable than relying solely on discovery-based device naming.

95. Test the configuration, verify the mount, and investigate all errors before reboot.

96. Check whether another filesystem was mounted over `/data`.

97. Which directories and files consume the filesystem, plus deleted-open files and other filesystem-specific causes.

98. Directories or applications creating huge numbers of files.

99. Mount options, kernel messages, filesystem health, and underlying storage health.

100. Identify and verify exactly what storage object you are changing before making a destructive change.

# 172. Final Storage Mental Model

Everything in this part should now fit together:

```text
Physical / Cloud / Virtual Storage
               |
               v
          Block Device
           /dev/sdb
               |
               v
        Partition Table
               |
               v
           Partition
           /dev/sdb1
               |
               v
           Filesystem
              ext4
               |
               | UUID
               v
             Mount
               |
               v
          Mount Point
             /data
               |
       +-------+-------+
       |       |       |
       v       v       v
      logs    app    backups
```

And your main inspection tools map to different questions:

```text
What disks and partitions exist?
        |
        v
      lsblk
```

```text
What filesystem and UUID exist?
        |
        v
 lsblk -f / blkid
```

```text
What is mounted where?
        |
        v
      findmnt
```

```text
How full is the filesystem?
        |
        v
       df -h
```

```text
Are filesystem metadata/inodes exhausted?
        |
        v
       df -i
```

```text
Which directories use the space?
        |
        v
         du
```

```text
Which large files exist?
        |
        v
        find
```

```text
Why does df show space that du cannot explain?
        |
        v
   inspect open/deleted files
        |
        v
        lsof
```

```text
Why did a mount fail or become read-only?
        |
        v
lsblk + blkid + findmnt + kernel logs
```

The most important habit from Part 14 is simple:

```text
Observe
   |
   v
Identify the exact filesystem
   |
   v
Identify the device
   |
   v
Understand the current state
   |
   v
Gather evidence
   |
   v
Make the smallest safe change
   |
   v
Verify
```

Never treat storage administration as a sequence of commands to memorize.

When you see:

```text
No space left on device
```

you should now ask:

```text
Which filesystem?

Bytes or inodes?

What is consuming them?

Is there a deleted-open file?

Why did usage grow?

What is safe to remove?

How do we prevent recurrence?
```

When a filesystem disappears after reboot:

```text
Was it ever configured persistently?

Is the fstab entry correct?

Does the UUID match?

Does the mount point exist?

Does mount -a succeed?
```

When files appear to disappear:

```text
Was another filesystem mounted over the directory?
```

And when a filesystem suddenly becomes read-only:

```text
Is this an intentional mount option,
or is Linux protecting the filesystem because something is wrong?
```

That reasoning is the foundation you need before moving deeper into Linux storage and production server administration.
