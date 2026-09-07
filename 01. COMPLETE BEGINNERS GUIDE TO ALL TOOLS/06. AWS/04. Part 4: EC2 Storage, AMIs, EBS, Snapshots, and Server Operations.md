# Part 4: EC2 Storage, AMIs, EBS, Snapshots, and Server Operations

In Part 3, you built your first Linux server:

```text
Laptop
   |
Internet
   |
EC2 Linux Server
   |
Nginx
   |
Web Application
```

You installed software and created files on that server.

But this creates an important question:

**Where are those files actually stored?**

Consider:

```text
EC2
 |
 +-- Linux operating system
 +-- Nginx
 +-- Application
 +-- Configuration files
 +-- Logs
 +-- Uploaded files
```

What happens to this data if:

* The instance is stopped?
* The instance is rebooted?
* The instance is terminated?
* The disk fills up?
* The filesystem becomes damaged?
* The Availability Zone has a problem?
* An engineer accidentally deletes files?
* You need a larger disk?
* You need to rebuild the server?
* You need ten identical servers?

Launching a server is only the beginning.

Operating a server requires understanding its storage and recovery model.

This part introduces:

* Ephemeral vs persistent storage
* Amazon EBS
* EBS volumes
* Volume types
* Attaching additional storage
* Linux block devices
* Partitioning
* Filesystems
* Mount points
* `/etc/fstab`
* Storage expansion
* EBS snapshots
* Backup principles
* Custom AMIs
* EC2 recovery
* Monitoring
* Status checks
* Server troubleshooting
* Production operational practices

By the end, you should understand the path:

```text
EC2
 |
 v
EBS Volume
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
Mount Point
 |
 v
Application Data
```

That model is fundamental to Linux server operations in AWS.

---

# 1. Why EC2 Storage Matters

Suppose your application writes customer uploads to:

```text
/var/www/uploads
```

Everything works.

Then one day the server fails.

If you do not understand where that directory is stored, you cannot confidently answer:

```text
Can the data be recovered?

Was it backed up?

Can the disk be attached elsewhere?

Will terminating EC2 delete it?

Can we restore yesterday's version?

Can we expand the disk?

Can we replace the server without losing data?
```

These are production questions.

A DevOps engineer must understand them before an incident occurs.

---

# 2. Compute and Storage Are Different Concepts

An EC2 instance provides compute.

Storage holds data.

Think:

```text
EC2
 |
Compute
 |
CPU + Memory
```

and:

```text
Storage
 |
Operating System
Applications
Configuration
Data
Logs
```

The two work together, but they should not be treated as the same thing.

A simplified EC2 architecture is:

```text
EC2 Instance
 |
 +-- CPU
 |
 +-- Memory
 |
 +-- Network Interface
 |
 +-- Storage
```

AWS provides different storage options depending on the workload.

For this part, our primary focus is Amazon EBS.

---

# 3. Ephemeral vs Persistent Storage

Two important storage concepts are:

```text
Ephemeral Storage
```

and:

```text
Persistent Storage
```

Understanding the difference prevents serious data-loss mistakes.

---

# 4. What Is Ephemeral Storage?

Ephemeral means temporary.

Ephemeral storage is tied closely to the lifetime or placement of the compute environment providing it.

In EC2, instance store is an example of ephemeral storage.

Conceptually:

```text
EC2 Host
 |
Local Physical Storage
 |
Instance Store
 |
EC2 Instance
```

The storage is physically associated with the underlying host infrastructure.

It can provide high performance for suitable workloads.

But it should not be treated as durable storage for important data.

---

# 5. What Can Cause Instance Store Data Loss?

Instance-store data does not behave like EBS.

Data can be lost when the underlying instance lifecycle or hardware placement changes in ways that remove access to that local storage.

For example, stopping or terminating an instance that uses instance store can result in loss of that instance-store data.

Therefore:

```text
Instance Store
 |
Temporary data
 |
Not your only copy
```

Good uses may include:

* Temporary processing files
* Caches
* Scratch space
* Replicated data
* Re-creatable intermediate results

Bad use:

```text
Only copy of critical customer database
```

---

# 6. What Is Persistent Storage?

Persistent storage is designed so data can continue to exist independently of a particular running compute session.

Amazon EBS is the main persistent block-storage service used with EC2.

Conceptually:

```text
EC2 Running
   |
   v
EBS Volume
   |
   v
Data
```

Stop the instance:

```text
EC2 Stopped

EBS Volume
   |
   v
Data remains
```

Start again:

```text
EC2 Running
   |
   v
EBS Volume
   |
   v
Data available
```

This makes EBS suitable for operating systems, applications, and persistent server data.

However, persistent does not mean indestructible.

You still need backups.

---

# 7. Ephemeral vs Persistent

A simple comparison:

| Characteristic                     | Ephemeral Storage  | Persistent Storage             |
| ---------------------------------- | ------------------ | ------------------------------ |
| Intended lifetime                  | Temporary          | Longer-lived                   |
| Survives normal stop/start         | Not necessarily    | EBS does                       |
| Good for cache/scratch             | Yes                | Can be, but often unnecessary  |
| Good as only copy of critical data | No                 | Better, but still needs backup |
| Example                            | EC2 instance store | EBS                            |

The key principle:

```text
Persistent
does not mean
Backed up
```

We will return to this.

---

# 8. What Is Amazon EBS?

EBS stands for:

Elastic Block Store.

Amazon EBS provides block storage volumes designed for use with EC2 and related workloads.

Think of an EBS volume as a virtual disk.

If EC2 is the computer:

```text
EC2
 |
Virtual Computer
```

then EBS is similar conceptually to:

```text
Virtual Disk
```

Together:

```text
EC2
 |
 +-- CPU
 +-- RAM
 +-- Network
 |
 +-- EBS Volume
      |
      +-- Operating System
      +-- Applications
      +-- Files
```

---

# 9. What Is Block Storage?

Block storage presents storage to the operating system as blocks.

Linux sees an EBS volume as a block device.

Conceptually:

```text
AWS EBS Volume
      |
      v
Linux Block Device
      |
      v
Filesystem
      |
      v
Files and Directories
```

Applications usually do not interact directly with raw storage blocks.

Linux creates a filesystem on the device.

Then you work with familiar paths such as:

```text
/data
/var/lib/app
/opt/application
```

---

# 10. EBS Volumes

An EBS volume is an individual block-storage resource.

EBS volumes have characteristics such as:

* Volume ID
* Volume type
* Size
* Availability Zone
* IOPS configuration
* Throughput characteristics
* Encryption settings
* Snapshot relationships

An EBS volume ID looks like:

```text
vol-xxxxxxxxxxxxxxxxx
```

Example format:

```text
vol-0123456789abcdef0
```

Just as EC2 has:

```text
i-...
```

EBS uses:

```text
vol-...
```

Learn to recognize resource prefixes.

---

# 11. Root Volumes

When you launch an EBS-backed Linux EC2 instance, it normally receives a root volume.

The root volume contains the operating system.

Conceptually:

```text
EC2
 |
 v
Root EBS Volume
 |
 +-- /
 +-- /etc
 +-- /var
 +-- /home
 +-- /usr
```

The Linux root filesystem is mounted at:

```text
/
```

This is the top of the Linux filesystem hierarchy.

---

# 12. Additional EBS Volumes

You are not limited to the root volume.

You can attach additional volumes.

Example:

```text
EC2
 |
 +-- Root Volume
 |     |
 |     /
 |
 +-- Data Volume
       |
       /data
```

This can be useful when separating:

* Operating system
* Application data
* Logs
* Database data
* Build artifacts

For example:

```text
Root Volume
 |
Operating System

Data Volume
 |
Application Data
```

This separation can make storage management easier.

---

# 13. EBS and Availability Zones

An EBS volume exists in an Availability Zone.

Suppose your instance is:

```text
us-east-1a
```

and your EBS volume is:

```text
us-east-1b
```

You cannot simply attach that EBS volume directly across AZs as though AZ placement does not matter.

For a normal direct attachment, the instance and volume must be in the same Availability Zone.

Conceptually:

```text
AZ-A

EC2
 |
EBS
```

works.

But:

```text
AZ-A               AZ-B

EC2  ------------> EBS
```

is not the normal direct EBS attachment model.

Availability Zone placement matters.

---

# 14. EBS Volume Types

Different workloads need different storage performance.

AWS therefore provides multiple EBS volume types.

Broad categories include:

* General Purpose SSD
* Provisioned IOPS SSD
* Throughput Optimized HDD
* Cold HDD

The exact product options and performance limits can evolve, so always check current AWS documentation before making production decisions.

The important beginner skill is understanding **why different types exist**.

---

# 15. General Purpose SSD

General Purpose SSD volumes are commonly used for many normal workloads.

Examples:

* Boot volumes
* Web servers
* Development environments
* Application servers
* Moderate databases

You will commonly encounter:

```text
gp3
```

and older:

```text
gp2
```

For many new workloads, gp3 is a common starting point.

Do not choose storage solely because a tutorial used a particular type years ago.

---

# 16. Provisioned IOPS SSD

Some applications need consistently high numbers of storage operations.

Examples may include:

* High-performance databases
* Transaction-heavy systems
* Latency-sensitive workloads

Provisioned IOPS SSD families are designed for workloads requiring high and predictable I/O performance.

IOPS means:

Input/Output Operations Per Second.

Think:

```text
Application
   |
   v
Read / Write Operations
   |
   v
Storage
```

IOPS measures how many storage operations can be performed per second under relevant conditions.

---

# 17. Throughput

IOPS is not the only storage metric.

Another important concept is throughput.

Throughput describes how much data can be transferred over time.

Think:

```text
IOPS
How many operations?

Throughput
How much data?
```

A workload performing many tiny random operations can have different requirements from one reading very large sequential files.

This is why storage selection depends on workload behavior.

---

# 18. HDD Volume Types

AWS also offers HDD-backed EBS options for workloads where throughput and cost characteristics matter more than low-latency random I/O.

These are not normally appropriate as boot volumes.

Examples of suitable workload characteristics can include:

* Large sequential workloads
* Certain log-processing systems
* Large data sets
* Infrequently accessed data

Again, choose based on workload requirements.

---

# 19. Choosing an EBS Volume Type

Do not choose based only on:

```text
Which one sounds fastest?
```

Ask:

```text
What does the application do?

How much storage?

How many read/write operations?

How much throughput?

How sensitive is it to latency?

What performance is required?

What is the cost?
```

Then choose.

A practical workflow:

```text
Understand workload
       |
       v
Choose initial volume
       |
       v
Monitor
       |
       v
Measure
       |
       v
Adjust
```

---

# 20. Our Storage Lab

We will extend the Part 3 server.

Current:

```text
EC2
 |
Root EBS Volume
 |
Linux
 |
Nginx
```

We will add:

```text
EC2
 |
 +-- Root EBS
 |
 +-- New EBS Data Volume
       |
       v
     /data
```

The complete process will be:

```text
Create Volume
      |
Attach Volume
      |
Discover Device
      |
Partition if needed
      |
Create Filesystem
      |
Create Mount Point
      |
Mount
      |
Verify
      |
Configure /etc/fstab
```

Each step matters.

---

# 21. Creating an EBS Volume

In the AWS console, navigate to:

```text
EC2
 |
Elastic Block Store
 |
Volumes
```

Choose:

```text
Create Volume
```

For a simple lab, choose an appropriate small General Purpose SSD volume.

Most importantly:

**Select the same Availability Zone as your EC2 instance.**

If EC2 is in:

```text
us-east-1a
```

the volume should also be in:

```text
us-east-1a
```

for direct attachment.

---

# 22. Name the Volume

Use tags.

For example:

```text
Name = devops-lab-data-01
Environment = lab
Purpose = application-data
```

Now:

```text
vol-012345...
```

has useful human context.

Do not rely only on generated resource IDs.

---

# 23. Attach the Volume

Select the EBS volume and choose:

```text
Attach Volume
```

Select your EC2 instance.

AWS may display a device name suggestion such as:

```text
/dev/sdf
```

This is the requested attachment name.

However, Linux may expose the actual device differently depending on the EC2 platform and storage driver.

For example, on Nitro-based instances, EBS volumes may appear as NVMe devices.

This is why you should **discover the device inside Linux rather than blindly assuming its name**.

---

# 24. Linux Device Discovery

SSH into the server.

Then run:

```bash
lsblk
```

`lsblk` means:

List block devices.

It displays storage devices known to Linux.

Example output may resemble:

```text
NAME         SIZE TYPE MOUNTPOINTS
nvme0n1       20G disk
└─nvme0n1p1   20G part /
nvme1n1       10G disk
```

Your exact output will differ.

Interpretation:

```text
nvme0n1
 |
Existing disk

nvme0n1p1
 |
Partition mounted at /

nvme1n1
 |
New disk
```

The important question:

**Which device is new?**

---

# 25. Do Not Guess Device Names

Never immediately run destructive disk commands against:

```text
/dev/nvme0n1
```

just because you saw it first.

That may be your operating-system disk.

Formatting the wrong device can destroy data.

Before partitioning or creating a filesystem:

1. Inspect devices.
2. Compare sizes.
3. Check mount points.
4. Identify the new disk.
5. Verify again.

Storage commands can be destructive.

---

# 26. Useful Device Inspection Commands

Start with:

```bash
lsblk
```

Then:

```bash
df -h
```

`df` reports filesystem disk usage.

`-h` means:

Human-readable sizes.

Example:

```text
Filesystem      Size  Used Avail Use% Mounted on
/dev/root        20G   6G    14G  30% /
```

`lsblk` answers:

```text
What block devices exist?
```

`df -h` answers:

```text
What mounted filesystems exist and how full are they?
```

These are related but different questions.

---

# 27. Inspect Filesystem Signatures

You can also use:

```bash
sudo blkid
```

`blkid` displays information about block devices, including filesystem types and UUIDs where present.

This can help determine whether a volume already contains a filesystem.

Another useful command is:

```bash
sudo file -s /dev/DEVICE
```

Replace `/dev/DEVICE` with the device you have verified.

This can help inspect whether the device contains recognizable filesystem data.

Do not format an existing production volume merely because you expected it to be empty.

Inspect first.

---

# 28. What Is Partitioning?

A disk can be divided into partitions.

Conceptually:

```text
Disk
 |
 +-- Partition 1
 |
 +-- Partition 2
 |
 +-- Partition 3
```

Each partition can be used separately.

For a simple dedicated data volume, you may either use the whole block device directly for a filesystem or create a partition first, depending on your design.

For learning, we will understand the partitioned approach.

---

# 29. Disk vs Partition vs Filesystem

These terms are often confused.

Think:

```text
Disk
 |
 v
Partition
 |
 v
Filesystem
 |
 v
Directories and Files
```

Example:

```text
/dev/nvme1n1
 |
Disk

/dev/nvme1n1p1
 |
Partition

ext4
 |
Filesystem

/data
 |
Mount Point
```

These are different layers.

---

# 30. Creating a Partition

A common Linux partitioning tool is:

```bash
fdisk
```

For example:

```bash
sudo fdisk /dev/nvme1n1
```

**Warning:** Replace the device with the actual new lab disk you verified. Running partitioning commands against the wrong device can damage the system.

Inside `fdisk`, common actions include:

```text
p
Print partition table

n
Create new partition

w
Write changes
```

For a simple lab disk, you might create one partition using the available space.

Afterward:

```bash
lsblk
```

may show:

```text
nvme1n1
└─nvme1n1p1
```

Now you have a partition.

---

# 31. What Is a Filesystem?

A raw block device does not automatically provide the familiar file and directory structure you use in Linux.

A filesystem organizes data into:

* Files
* Directories
* Metadata
* Permissions
* Allocation structures

Common Linux filesystems include:

```text
ext4
xfs
```

Different Linux distributions and workloads may use different defaults.

---

# 32. Creating an ext4 Filesystem

For our verified new lab partition:

```bash
sudo mkfs.ext4 /dev/nvme1n1p1
```

Breakdown:

`mkfs`

Make filesystem.

`.ext4`

Create an ext4 filesystem.

`/dev/nvme1n1p1`

Target partition.

**Warning:** `mkfs` destroys existing filesystem data on the target device.

Never run this command until you have verified that the target is the correct new/empty storage.

---

# 33. Verify the Filesystem

Run:

```bash
sudo blkid /dev/nvme1n1p1
```

You may see information containing:

```text
TYPE="ext4"
```

and:

```text
UUID="..."
```

This confirms that the partition now contains an ext4 filesystem.

The UUID will become useful when configuring persistent mounts.

---

# 34. What Is Mounting?

Creating a filesystem is not enough.

Linux needs to know where that filesystem should appear in its directory tree.

This is called mounting.

Suppose we want the new disk accessible at:

```text
/data
```

The relationship becomes:

```text
EBS Volume
   |
Block Device
   |
Partition
   |
ext4 Filesystem
   |
Mounted at
   |
/data
```

---

# 35. Create a Mount Point

Create the directory:

```bash
sudo mkdir -p /data
```

Breakdown:

`mkdir`

Make directory.

`-p`

Creates parent directories if necessary and avoids an error when the directory already exists in suitable circumstances.

`/data`

The mount point.

At this stage, `/data` is only a directory.

The new filesystem is not mounted yet.

---

# 36. Mount the Filesystem

Run:

```bash
sudo mount /dev/nvme1n1p1 /data
```

Breakdown:

`mount`

Attach a filesystem into the Linux directory tree.

`/dev/nvme1n1p1`

Filesystem-containing partition.

`/data`

Mount point.

Now:

```text
/dev/nvme1n1p1
       |
       v
     /data
```

---

# 37. Verify the Mount

Run:

```bash
df -h
```

Look for:

```text
/dev/nvme1n1p1 ... /data
```

Then:

```bash
lsblk
```

You should also see:

```text
nvme1n1p1
 |
/data
```

Verification matters.

Do not assume `mount` worked merely because the command printed no error.

---

# 38. Test Writing Data

Create a test file:

```bash
sudo touch /data/storage-test.txt
```

`touch` creates an empty file if it does not already exist.

Verify:

```bash
ls -l /data
```

You should see:

```text
storage-test.txt
```

Now you have proven that the mounted filesystem can store files.

---

# 39. Permissions on the Mount Point

Your application may not run as root.

Suppose your application runs as:

```text
appuser
```

but `/data` is owned by root.

The application may receive:

```text
Permission denied
```

This is a Linux permissions issue, not an EBS failure.

You may need appropriate ownership or permissions.

For example, conceptually:

```bash
sudo chown appuser:appuser /data
```

Do not run this blindly.

The correct owner depends on which application or service should control the directory.

Storage security includes filesystem permissions.

---

# 40. What Happens After Reboot?

You mounted the disk manually:

```bash
sudo mount /dev/nvme1n1p1 /data
```

Everything works.

Then the server reboots.

The filesystem may not automatically mount again unless persistent mount configuration exists.

This introduces:

```text
/etc/fstab
```

---

# 41. What Is `/etc/fstab`?

`/etc/fstab` is a Linux configuration file describing filesystems that should be mounted according to system configuration.

`fstab` means:

Filesystem table.

Conceptually:

```text
Server Boots
    |
    v
Reads /etc/fstab
    |
    v
Mount configured filesystems
```

This makes storage mounts persistent across reboots.

---

# 42. Inspect `/etc/fstab`

Run:

```bash
cat /etc/fstab
```

`cat` displays file contents.

Do not modify the file yet.

First inspect it.

A typical entry contains fields resembling:

```text
UUID=... /data ext4 defaults,nofail 0 2
```

Each field has meaning.

---

# 43. Why Use UUID?

You could theoretically reference a device name such as:

```text
/dev/nvme1n1p1
```

But device naming can be less stable or obvious than persistent filesystem identifiers.

A UUID is a filesystem identifier.

Retrieve it with:

```bash
sudo blkid
```

Example:

```text
UUID="11111111-2222-3333-4444-555555555555"
```

Then `/etc/fstab` can reference:

```text
UUID=11111111-2222-3333-4444-555555555555
```

This is generally safer than depending entirely on a particular runtime device name.

---

# 44. Example `/etc/fstab` Entry

A lab entry might look like:

```text
UUID=11111111-2222-3333-4444-555555555555 /data ext4 defaults,nofail 0 2
```

Breakdown:

```text
UUID=...
 |
Which filesystem

/data
 |
Where to mount

ext4
 |
Filesystem type

defaults,nofail
 |
Mount options

0
 |
Dump field

2
 |
Filesystem check order
```

The exact appropriate options depend on the workload.

---

# 45. Why `nofail` Can Matter

If a non-critical secondary volume is unavailable during boot, a bad mount configuration can interfere with startup.

`nofail` tells the system not to treat failure to mount that filesystem as fatal in the same way as a required filesystem.

However, do not use `nofail` blindly.

If your application absolutely requires the volume, silently continuing without it can create another failure mode.

The correct behavior depends on the system design.

---

# 46. Back Up `/etc/fstab` Before Editing

Before editing:

```bash
sudo cp /etc/fstab /etc/fstab.backup
```

`cp`

Copies a file.

Now:

```text
/etc/fstab
```

has a backup:

```text
/etc/fstab.backup
```

This is a simple operational habit.

Configuration changes should be recoverable.

---

# 47. Test `/etc/fstab` Before Rebooting

A dangerous beginner workflow is:

```text
Edit /etc/fstab
      |
Reboot immediately
      |
Server fails to mount correctly
```

Instead, test.

After editing, you can use:

```bash
sudo mount -a
```

`-a`

attempts to mount filesystems described by `/etc/fstab` that are eligible for mounting.

If there is an error, investigate before rebooting.

Then verify:

```bash
df -h
```

and:

```bash
lsblk
```

The rule:

```text
Change
 |
Test
 |
Verify
 |
Then reboot if needed
```

---

# 48. Expanding an EBS Volume

Suppose your root volume is:

```text
20 GB
```

and usage reaches:

```text
19 GB
```

You need more capacity.

With EBS, you can often increase volume size without replacing the entire server.

But increasing the AWS EBS volume is only one part of the process.

This distinction is critical.

---

# 49. Cloud Layer vs Linux Layer

Suppose:

```text
AWS EBS Volume
20 GB
```

You increase it to:

```text
AWS EBS Volume
40 GB
```

AWS now provides a larger block device.

But Linux may still have:

```text
Partition: 20 GB
Filesystem: 20 GB
```

So:

```text
EBS Volume
40 GB

Partition
20 GB

Filesystem
20 GB
```

Your application may still see only 20 GB.

You may need to expand the partition and filesystem.

---

# 50. Storage Expansion Mental Model

The full process can be:

```text
Increase EBS Volume
        |
        v
Linux sees larger device
        |
        v
Expand partition if applicable
        |
        v
Expand filesystem
        |
        v
Verify new capacity
```

Never assume changing the EBS size automatically completes every operating-system layer.

---

# 51. Inspect Before Expanding

Start with:

```bash
lsblk
```

Then:

```bash
df -h
```

You are comparing:

```text
Block Device Size
```

against:

```text
Filesystem Size
```

Suppose:

```text
lsblk:
Disk = 40G
Partition = 20G

df -h:
Filesystem = 20G
```

This tells you:

AWS disk expansion happened, but Linux layers still need work.

---

# 52. Expanding a Partition

On systems where a partition needs to grow, a tool such as `growpart` may be used.

Example concept:

```bash
sudo growpart /dev/nvme0n1 1
```

Breakdown:

`growpart`

Expands a partition to use available space.

`/dev/nvme0n1`

Disk.

`1`

Partition number.

**Warning:** Device names differ between systems. Do not copy this command without verifying your actual disk and partition.

Afterward:

```bash
lsblk
```

should show the larger partition.

---

# 53. Expanding an ext4 Filesystem

If the filesystem is ext4, a common command is:

```bash
sudo resize2fs /dev/nvme0n1p1
```

`resize2fs`

resizes an ext2/ext3/ext4 filesystem.

Again:

Use the actual verified device.

Then:

```bash
df -h
```

should show the expanded filesystem.

---

# 54. Expanding XFS

XFS uses a different tool.

A common pattern is:

```bash
sudo xfs_growfs MOUNT_POINT
```

For example:

```bash
sudo xfs_growfs /
```

if the XFS filesystem being expanded is mounted at `/`.

This illustrates an important principle:

**Know the filesystem type before choosing the resize command.**

Check with:

```bash
df -T
```

`-T`

shows filesystem type.

Do not run ext4 tools against XFS or vice versa.

---

# 55. Storage Expansion Verification

After expansion:

```bash
lsblk
```

Verify device/partition size.

Then:

```bash
df -h
```

Verify filesystem capacity.

Then test the application.

A successful AWS console modification is not enough.

The operating system and application must also see the expected result.

---

# 56. What Is an EBS Snapshot?

An EBS snapshot is a point-in-time backup mechanism for EBS volumes.

Conceptually:

```text
EBS Volume
   |
   v
Snapshot
```

Later:

```text
Snapshot
   |
   v
New EBS Volume
```

Snapshots provide an important recovery capability.

---

# 57. Why Snapshots Matter

Suppose:

```text
Production EBS Volume
 |
Important data
```

An engineer accidentally damages the filesystem or deletes data.

Without a backup:

```text
Data lost
```

With a suitable snapshot:

```text
Snapshot
   |
Create replacement volume
   |
Recover data
```

Snapshots are therefore an important part of recovery planning.

---

# 58. Snapshots Are Incremental

After the initial snapshot, later snapshots generally store changed blocks relative to existing snapshot data rather than creating completely independent full physical copies every time.

Conceptually:

```text
Snapshot 1
 |
Initial blocks

Snapshot 2
 |
Changed blocks

Snapshot 3
 |
Additional changed blocks
```

AWS manages the underlying dependency structure.

From the user's perspective, snapshots can still be used to restore volumes.

---

# 59. Creating a Snapshot

In the AWS console:

```text
EC2
 |
Elastic Block Store
 |
Volumes
 |
Select Volume
 |
Create Snapshot
```

Use meaningful tags.

Example:

```text
Name = prod-web-root-before-upgrade
Purpose = pre-upgrade-backup
```

A name such as:

```text
snapshot1
```

does not explain why the snapshot exists.

---

# 60. Snapshot Consistency

Creating a storage snapshot does not automatically guarantee that every application has flushed all in-memory data into a perfectly application-consistent state.

Imagine:

```text
Database
 |
Data in memory
 |
Writes in progress
 |
EBS
```

A snapshot captures storage state.

For applications such as databases, you may need application-aware procedures to achieve the required consistency.

Depending on the workload, that can include:

* Flushing data
* Quiescing writes
* Coordinating with the application
* Using database-native backups
* Using managed backup capabilities

A snapshot is a tool.

A backup strategy requires more thought.

---

# 61. What Is a Backup?

A backup is a recoverable copy of data or system state intended to help restore from failure, deletion, corruption, or disaster.

The important word is:

```text
Recoverable
```

A file called "backup" is useless if nobody can restore it.

A proper backup strategy asks:

```text
What are we backing up?

How often?

Where is it stored?

How long is it retained?

Who can delete it?

Is it encrypted?

How do we restore it?

Have we tested restoration?
```

---

# 62. Backup vs High Availability

These are not the same.

High availability helps keep services running when components fail.

Backup helps recover data or system state.

Example:

```text
Two EC2 instances
 |
High availability
```

does not automatically mean:

```text
Deleted customer data can be restored.
```

If both instances access the same corrupted data, high availability does not solve the problem.

You may need both:

```text
High Availability
+
Backups
```

---

# 63. Backup vs Snapshot

A snapshot can be part of a backup strategy.

But:

```text
Snapshot
```

is not automatically equivalent to:

```text
Complete Backup Strategy
```

A complete strategy also considers:

* Scheduling
* Retention
* Encryption
* Permissions
* Cross-account protection
* Geographic resilience
* Restore testing
* Recovery objectives

---

# 64. RPO and RTO

Two important disaster-recovery concepts are:

```text
RPO
RTO
```

RPO means:

Recovery Point Objective.

It asks:

**How much data can we afford to lose?**

Suppose backups occur every 24 hours.

A failure just before the next backup could potentially mean nearly 24 hours of lost changes.

If the business can tolerate only five minutes of data loss, daily snapshots are not enough.

---

# 65. Recovery Time Objective

RTO means:

Recovery Time Objective.

It asks:

**How quickly must the service be restored?**

Example:

```text
Failure occurs
10:00

Service must be restored
10:30
```

RTO:

```text
30 minutes
```

Your backup and architecture strategy should support the business requirement.

---

# 66. Backup Thinking

Do not begin with:

```text
How many snapshots should I create?
```

Begin with:

```text
What failure are we protecting against?

How much data can we lose?

How quickly must we recover?

What systems depend on this data?
```

Then design the backup strategy.

---

# 67. Restoring an EBS Snapshot

A common recovery workflow is:

```text
Snapshot
   |
Create Volume
   |
Select Availability Zone
   |
Attach to EC2
   |
Discover device
   |
Mount filesystem
   |
Recover data
```

Notice how earlier Linux skills return.

Recovery still requires understanding:

* Devices
* Filesystems
* Mounting
* Permissions

Cloud backup knowledge without Linux knowledge is incomplete.

---

# 68. Snapshot Restore Lab

For a disposable lab volume:

1. Create a file:

```bash
echo "important lab data" | sudo tee /data/important.txt
```

`echo`

outputs text.

The pipe:

```text
|
```

passes the output to another command.

`tee`

writes input into a file.

2. Verify:

```bash
cat /data/important.txt
```

3. Create an EBS snapshot.

4. Wait until the snapshot is usable.

5. Create a new EBS volume from the snapshot.

6. Place it in the same AZ as the recovery EC2 instance.

7. Attach it.

8. Run:

```bash
lsblk
```

9. Do **not** format it.

The restored volume already contains a filesystem.

10. Create a mount point:

```bash
sudo mkdir -p /recovery
```

11. Mount the restored filesystem using the correct device or partition.

12. Verify:

```bash
cat /recovery/important.txt
```

If the file exists, you have performed an actual restore.

---

# 69. Why You Must Not Run `mkfs` During Restore

This is critical.

When you create a new empty volume:

```text
No filesystem
```

you may need:

```text
mkfs
```

When you restore from a snapshot:

```text
Existing filesystem and data
```

Running:

```text
mkfs
```

can overwrite filesystem structures and destroy the data you are trying to recover.

Always inspect first.

---

# 70. Creating Custom AMIs

Remember Part 3:

AMI means Amazon Machine Image.

You originally launched:

```text
Base AMI
 |
EC2
```

Then you configured:

```text
Linux
Nginx
Application packages
Configuration
Monitoring agent
```

Suppose you now need ten similar servers.

One approach is to create a custom AMI.

Conceptually:

```text
Configured EC2
      |
      v
Create Custom AMI
      |
      v
Custom Image
      |
 +----+----+----+
 |         |    |
EC2       EC2  EC2
```

---

# 71. Why Create Custom AMIs?

Custom AMIs can help with:

* Consistent server builds
* Faster instance launch
* Standard operating-system configuration
* Preinstalled packages
* Security baselines
* Disaster recovery
* Auto Scaling

Instead of rebuilding every package from scratch during every launch:

```text
Launch
 |
Install 50 packages
 |
Configure
 |
Wait
```

you can start from a prepared image:

```text
Custom AMI
 |
Launch
 |
Smaller final initialization
```

---

# 72. Golden Images

Organizations sometimes create standardized machine images called:

```text
Golden Images
```

A golden image may contain:

* Approved operating-system version
* Security patches
* Monitoring agents
* Company certificates
* Standard configuration
* Required baseline tools

Then workloads build from this controlled base.

Conceptually:

```text
Security-approved AMI
       |
       +--> Web Server
       +--> Worker
       +--> Application Server
```

This improves standardization.

---

# 73. What Should Not Be Baked into an AMI?

Do not put sensitive or instance-specific information into reusable images carelessly.

Examples:

* Permanent AWS access keys
* Private credentials
* User passwords
* Application secrets
* Private SSH keys
* Environment-specific secrets

Imagine:

```text
AMI
 |
Secret
 |
Launch 100 servers
 |
Secret copied 100 times
```

This creates serious credential-management problems.

Secrets should use appropriate secure delivery mechanisms.

---

# 74. AMI vs User Data

Custom AMIs and user data solve related but different problems.

AMI:

```text
What does the server already contain at launch?
```

User data:

```text
What initialization should happen when it launches?
```

Example:

```text
Custom AMI
 |
Linux + Nginx + Monitoring Agent
 |
Launch
 |
User Data
 |
Configure environment-specific application settings
```

This can provide a useful balance.

---

# 75. Image Drift

Suppose you create:

```text
AMI-v1
```

Then months later:

* Security patches change.
* Packages change.
* Agents change.
* Configuration standards change.

If you keep launching the old AMI forever:

```text
New Servers
 |
Old Image
```

you create image drift and outdated infrastructure.

Custom images need lifecycle management.

A production image process may include:

```text
Build
 |
Test
 |
Security Scan
 |
Publish
 |
Deploy
 |
Replace old image
 |
Retire obsolete image
```

---

# 76. Recovering an EC2 Instance

Suppose your EC2 instance becomes unusable.

Do not immediately terminate it.

First ask:

```text
What failed?
```

Possible failure categories include:

```text
AWS infrastructure
Operating system
Filesystem
Network
Authentication
Application
Configuration
Storage capacity
CPU/memory pressure
```

Different failures require different recovery methods.

---

# 77. Recovery Scenario: Application Failure

Symptom:

```text
Website unavailable
```

But:

```text
EC2 running
SSH works
Disk healthy
```

Investigate the application.

For Nginx:

```bash
sudo systemctl status nginx
```

Then:

```bash
sudo journalctl -u nginx
```

`journalctl`

reads logs from the systemd journal.

`-u nginx`

filters for the Nginx service.

You may discover:

```text
Configuration syntax error
```

This is an application/service problem, not an EC2 infrastructure failure.

---

# 78. Recovery Scenario: Disk Full

Symptom:

```text
Application cannot write files
Logs stop
Service behaves strangely
```

Check:

```bash
df -h
```

Suppose:

```text
Filesystem      Size Used Avail Use%
/dev/root        20G  20G    0G 100%
```

Root cause:

```text
Filesystem full
```

Investigate what consumed space.

A useful command:

```bash
sudo du -xhd1 /
```

Breakdown:

`du`

Disk usage.

`-x`

Stay on one filesystem.

`-h`

Human-readable.

`-d1`

Show one level of directory depth on implementations supporting that option.

Then investigate large directories more carefully.

Do not immediately delete random files.

---

# 79. Disk Full Troubleshooting

A disciplined workflow:

```text
df -h
 |
Which filesystem is full?
 |
du
 |
Which directory is large?
 |
Inspect files
 |
Understand owner/application
 |
Remove/archive safely
 |
Verify
 |
Prevent recurrence
```

Possible causes:

* Logs growing without rotation
* Temporary files
* Application uploads
* Package caches
* Core dumps
* Container data
* Old releases

The correct fix depends on the cause.

---

# 80. Recovery Scenario: Bad `/etc/fstab`

Suppose an engineer adds an incorrect mount entry.

After reboot:

```text
Server does not boot normally
```

This is why you:

* Back up configuration.
* Use UUIDs carefully.
* Test `mount -a`.
* Understand mount options.
* Make changes incrementally.

Recovery may require accessing the root volume from another rescue instance or using appropriate AWS recovery capabilities.

---

# 81. Rescue by Attaching the Root Volume Elsewhere

A classic recovery technique for an EBS-backed instance is:

```text
Broken EC2
   |
Stop
   |
Detach root EBS carefully
   |
Attach to healthy rescue EC2
   |
Mount
   |
Repair configuration/files
   |
Detach
   |
Reattach correctly
   |
Start original EC2
```

This technique can help recover from:

* Broken filesystem configuration
* Bad SSH configuration
* Certain boot problems
* Incorrect system files

This is an advanced operation and must be performed carefully.

Device mapping and root-volume handling matter.

Always create a snapshot before risky recovery work when possible.

---

# 82. Recovery Principle: Preserve Evidence

During an incident, beginners often immediately:

```text
Reboot
Delete
Reinstall
Terminate
```

That can destroy useful evidence.

Instead:

```text
Observe
 |
Capture logs
 |
Record state
 |
Snapshot if appropriate
 |
Investigate
 |
Then repair
```

Production troubleshooting is not only about restoring service.

It is also about understanding what happened.

---

# 83. EC2 Monitoring Basics

A running EC2 instance produces operational signals.

Monitoring helps answer:

```text
Is the server healthy?

Is CPU overloaded?

Is networking unusual?

Are status checks failing?

Is the application available?

Is the disk filling?

Is the server behaving differently from normal?
```

AWS provides monitoring through services such as CloudWatch.

We will study CloudWatch deeply later.

For now, learn the basic EC2 monitoring model.

---

# 84. Metrics

A metric is a numerical measurement collected over time.

Examples:

```text
CPU utilization
Network bytes
Disk operations
Status check failures
```

Conceptually:

```text
EC2
 |
Metrics
 |
CloudWatch
 |
Graphs / Alarms
```

Metrics let you observe trends rather than relying on guesses.

---

# 85. CPU Utilization

A common EC2 metric is:

```text
CPUUtilization
```

Suppose CPU remains:

```text
95%
```

for long periods.

Possible causes:

* Legitimate heavy workload
* Traffic spike
* Runaway process
* Bad application code
* Insufficient instance capacity
* Unexpected activity

High CPU is a symptom.

It is not automatically the root cause.

---

# 86. Memory Monitoring

A common beginner surprise is that standard EC2 host metrics do not automatically provide every guest operating-system metric you may want, such as detailed memory utilization.

To collect guest-level metrics such as:

* Memory usage
* Disk usage
* Additional application metrics

you can use tools such as the CloudWatch Agent or another monitoring system.

Conceptually:

```text
Linux OS
 |
CloudWatch Agent
 |
Memory / Disk / Logs
 |
CloudWatch
```

We will configure this later.

---

# 87. Disk Usage vs EBS Metrics

Do not confuse:

```text
EBS performance metrics
```

with:

```text
Linux filesystem free space
```

EBS can report storage I/O characteristics.

But the operating system knows things such as:

```text
/data is 92% full
```

These are different layers.

Again:

```text
AWS Infrastructure Layer
        |
Linux Operating System Layer
        |
Application Layer
```

Good monitoring covers all relevant layers.

---

# 88. EC2 Status Checks

EC2 provides status checks that help identify certain instance problems.

Two important categories are:

```text
System Status Check

Instance Status Check
```

These point toward different failure domains.

---

# 89. System Status Check

A system status check focuses on problems involving the underlying AWS infrastructure supporting the instance.

Conceptually:

```text
Physical Host
Network Infrastructure
Power
Underlying AWS System
        |
        v
System Status Check
```

If this fails while the instance itself was previously healthy, the issue may be below your guest operating system.

---

# 90. Instance Status Check

An instance status check focuses more on the individual instance and its operating-system reachability.

Possible issues may include:

* Operating-system problems
* Network configuration problems
* Exhausted resources
* Filesystem issues
* Boot problems

Conceptually:

```text
EC2 Guest OS
 |
Instance Status Check
```

---

# 91. Why Status Checks Matter

Suppose SSH fails.

Do not immediately blame the Security Group.

Check:

```text
Instance State
Status Checks
```

If system status is failing:

```text
Underlying infrastructure issue
```

may be involved.

If instance status is failing:

```text
Guest operating-system problem
```

may be involved.

If both checks pass:

```text
Investigate networking, SSH, authentication, application
```

This helps narrow the search.

---

# 92. Server Troubleshooting Layers

When an EC2 server fails, use layers.

```text
Layer 1
AWS Resource State

Layer 2
Underlying Infrastructure

Layer 3
Network

Layer 4
Operating System

Layer 5
Storage

Layer 6
Service

Layer 7
Application
```

Do not randomly jump between layers.

---

# 93. Step 1: Check EC2 State

Ask:

```text
Is it running?
```

Possible states:

```text
running
stopped
pending
stopping
shutting-down
terminated
```

A stopped instance cannot serve your application.

---

# 94. Step 2: Check Status Checks

Ask:

```text
Are system checks passing?

Are instance checks passing?
```

This tells you whether AWS detects a lower-level problem.

---

# 95. Step 3: Check Network

Ask:

```text
Does the instance have the expected IP?

Did the public IP change?

Is the route correct?

Does the Security Group permit traffic?

Is the subnet configured correctly?

Is the network ACL interfering?
```

Parts 5 through 10 will make this layer much easier to diagnose.

---

# 96. Step 4: Check the Operating System

If SSH works:

```bash
uptime
```

`uptime` shows:

* How long the server has been running
* Logged-in user count
* Load averages

Then:

```bash
free -h
```

`free`

shows memory information.

`-h`

uses human-readable units.

Then:

```bash
df -h
```

checks filesystem capacity.

Then:

```bash
lsblk
```

checks block devices.

This quickly establishes basic server health.

---

# 97. Step 5: Check Processes

Use:

```bash
ps aux
```

`ps`

shows process information.

`a`, `u`, and `x` influence which processes and details are displayed.

For interactive inspection, systems may provide:

```bash
top
```

This can help identify:

* CPU-heavy processes
* Memory-heavy processes
* Load conditions

Do not kill processes randomly.

First identify what they are and why they are consuming resources.

---

# 98. Step 6: Check Services

For a systemd-managed application:

```bash
sudo systemctl status SERVICE
```

Example:

```bash
sudo systemctl status nginx
```

Ask:

```text
Is it active?

Did it fail?

When did it fail?

Is there an error message?
```

Then inspect logs.

---

# 99. Step 7: Check Logs

For systemd services:

```bash
sudo journalctl -u nginx
```

For recent system events:

```bash
sudo journalctl -xe
```

Applications may also write logs to locations such as:

```text
/var/log
```

For Nginx, common paths include:

```text
/var/log/nginx/access.log
/var/log/nginx/error.log
```

View recent lines:

```bash
sudo tail -n 50 /var/log/nginx/error.log
```

`tail`

shows the end of a file.

`-n 50`

shows the last 50 lines.

Logs are evidence.

Read them before guessing.

---

# 100. Step 8: Test Locally

If the application is a web service:

```bash
curl http://localhost
```

If that succeeds:

```text
Application works locally
```

Then external failure likely lies elsewhere.

If it fails:

```text
Investigate service/application locally
```

This simple test separates application problems from external network problems.

---

# 101. Troubleshooting Example: Website Down

Alert:

```text
Website unavailable
```

Step 1:

EC2:

```text
running
```

Step 2:

Status checks:

```text
passing
```

Step 3:

SSH:

```text
works
```

Step 4:

Run:

```bash
curl http://localhost
```

Result:

```text
Connection refused
```

Step 5:

```bash
sudo systemctl status nginx
```

Result:

```text
failed
```

Step 6:

Inspect logs.

You discover:

```text
Invalid Nginx configuration
```

Root cause:

Application/service configuration.

Not:

* AWS Region
* EC2 hardware
* Security Group
* EBS

Fix the configuration.

Restart:

```bash
sudo systemctl restart nginx
```

Verify:

```bash
curl http://localhost
```

Then test externally.

That is structured troubleshooting.

---

# 102. Troubleshooting Example: Application Cannot Write

Symptom:

```text
Application returns errors when uploading files
```

SSH works.

Application is running.

Check:

```bash
df -h
```

Result:

```text
/data
100% full
```

Root cause:

Storage exhaustion.

Next question:

```text
Why did /data fill?
```

Not merely:

```text
How do I make the disk bigger?
```

Maybe logs are growing uncontrollably.

If you only expand storage:

```text
10 GB
 |
20 GB
 |
40 GB
 |
80 GB
```

without fixing the log problem, the disk will fill again.

Fix root cause and capacity.

---

# 103. Troubleshooting Example: Volume Attached but Missing

You attach EBS.

But:

```bash
df -h
```

does not show it.

Beginner conclusion:

```text
EBS failed.
```

Not necessarily.

Remember the layers:

```text
EBS Attached
   |
Block Device
   |
Filesystem
   |
Mount
```

Run:

```bash
lsblk
```

If the device appears but has no mount point:

```text
AWS attachment succeeded
Linux mount is missing
```

You may still need:

* Partition
* Filesystem
* Mount

depending on the volume state.

---

# 104. Production EC2 Operational Practices

Launching an EC2 instance is easy.

Operating one safely over months or years requires discipline.

Production practices should address:

* Security
* Patching
* Monitoring
* Backups
* Recovery
* Capacity
* Automation
* Access
* Logging
* Cost
* Change management
* Replacement strategy

---

# 105. Practice 1: Avoid Manual Snowflake Servers

A snowflake server is a server that becomes unique because engineers repeatedly modify it manually.

Example:

```text
Server launched
 |
Engineer installs package
 |
Another engineer edits config
 |
Someone applies manual fix
 |
Nobody documents it
 |
Six months later
 |
Nobody can reproduce server
```

This is dangerous.

Prefer reproducible configuration.

Examples include:

* Infrastructure as Code
* User data
* Configuration management
* Image pipelines
* Deployment automation

The objective:

```text
Server can be rebuilt
```

not:

```text
Server must survive forever
```

---

# 106. Practice 2: Treat Servers as Replaceable

Traditional thinking:

```text
This server is precious.
Never lose it.
```

Cloud-native operational thinking:

```text
The service is important.
The individual server should be replaceable.
```

If one EC2 instance fails:

```text
Replace instance
 |
Restore/reconnect required state
 |
Continue service
```

This becomes easier when application state is separated from individual compute instances.

---

# 107. Practice 3: Automate Backups

Do not depend on an engineer remembering:

```text
Every Friday, click Create Snapshot.
```

Humans forget.

Production backups should be automated according to policy.

AWS provides services and capabilities for automating backup and snapshot lifecycles.

The exact implementation depends on the environment.

---

# 108. Practice 4: Test Restores

A backup that has never been restored is an assumption.

Good process:

```text
Create Backup
      |
      v
Restore Test
      |
      v
Validate Data
      |
      v
Measure Recovery Time
```

Ask:

```text
Did restoration work?

Was data complete?

How long did it take?

Could the team repeat it during an incident?
```

---

# 109. Practice 5: Monitor Disk Capacity

Do not wait for:

```text
Filesystem 100%
```

to discover storage problems.

Create monitoring and alerts before critical thresholds.

Example concept:

```text
Disk Usage
 |
70% --> Observe trend
 |
80% --> Warning
 |
90% --> Critical action
```

Exact thresholds depend on the workload.

The principle is:

Detect capacity problems before outage.

---

# 110. Practice 6: Patch Systems

EC2 does not remove your responsibility for guest operating-system security.

You need a process for:

* Security updates
* Package updates
* Kernel updates
* Application dependencies
* Reboots when necessary
* Testing before production rollout

Do not leave servers unpatched indefinitely.

---

# 111. Practice 7: Minimize SSH Exposure

Avoid:

```text
Port 22
Source 0.0.0.0/0
```

for routine administration.

Use more controlled access.

Possible production approaches can include:

* Restricted administrative networks
* Bastion patterns
* VPN/private connectivity
* AWS Systems Manager Session Manager
* Strong identity controls

The exact design depends on the environment.

---

# 112. Practice 8: Use IAM Roles

Do not store permanent AWS credentials on EC2.

Prefer:

```text
EC2
 |
IAM Role
 |
Temporary Credentials
```

and apply least privilege.

Part 2 should now feel connected to server operations.

---

# 113. Practice 9: Encrypt Storage

EBS supports encryption.

Encryption helps protect data at rest.

A production storage design should consider:

```text
Is the volume encrypted?

Are snapshots encrypted?

Who controls the encryption key?

Who can decrypt the data?
```

Encryption should be planned from the beginning rather than added only after a security incident.

---

# 114. Practice 10: Use Tags

Tag resources consistently.

For EC2:

```text
Name = prod-payments-web-01
Environment = production
Application = payments
Team = platform
```

For EBS:

```text
Name = prod-payments-web-01-root
Environment = production
Application = payments
```

For snapshots:

```text
Name = prod-payments-web-01-pre-upgrade
```

Good metadata improves:

* Operations
* Automation
* Cost management
* Ownership
* Incident response

---

# 115. Practice 11: Know Your Recovery Procedure

Before production failure, document:

```text
How do we restore a volume?

How do we rebuild EC2?

Where are AMIs?

Where are snapshots?

How do we recover application configuration?

Who has permission to perform recovery?

What is our RPO?

What is our RTO?
```

An incident is the worst time to discover nobody knows the recovery process.

---

# 116. Practice 12: Monitor Multiple Layers

Do not monitor only:

```text
EC2 is running.
```

A server can be running while the application is completely broken.

Monitor:

```text
Infrastructure
 |
CPU
Status checks
EBS

Operating System
 |
Memory
Disk
Processes

Service
 |
Nginx
Application runtime

Application
 |
HTTP health
Errors
Latency

Business
 |
Transactions
Requests
Failures
```

Monitoring should tell you whether the service works, not merely whether a virtual machine exists.

---

# 117. Practice 13: Separate Important Data from Replaceable Compute

Suppose:

```text
EC2
 |
Application
 |
Only copy of customer uploads
```

Now replacing EC2 becomes risky.

A better architecture may place durable state in appropriate external storage.

Depending on the data:

```text
Application
 |
 +--> S3
 +--> RDS
 +--> EBS
```

The correct service depends on the workload.

This is why production systems should distinguish:

```text
Compute
```

from:

```text
State
```

---

# 118. Practice 14: Do Not Store Everything on the Root Volume

For small systems, root-volume storage may be enough.

But larger systems may benefit from separating important data.

Example:

```text
EC2
 |
 +-- Root EBS
 |     |
 |     OS
 |
 +-- Data EBS
       |
       Application data
```

This can make:

* Recovery
* Expansion
* Backup
* Replacement

easier to reason about.

It is not mandatory for every workload.

Architecture should follow requirements.

---

# 119. Practice 15: Snapshot Before Risky Changes

Before major operations such as:

* Filesystem changes
* Large upgrades
* Major application migrations
* Risky storage operations

consider whether a recoverable snapshot is appropriate.

Conceptually:

```text
Current Working State
        |
        v
Snapshot
        |
        v
Risky Change
```

If something fails, you have a recovery point.

But still verify that the snapshot method provides the consistency your workload requires.

---

# 120. Production Server Operational Checklist

Before calling an EC2 server production-ready, ask:

```text
IDENTITY

Is access controlled?
Is MFA used for administrators?
Does EC2 use an IAM role?
Are permissions least privilege?

NETWORK

Is SSH restricted?
Are unnecessary ports closed?
Is the instance meant to be public?

STORAGE

Is EBS appropriately sized?
Is it encrypted?
Are backups configured?
Are restores tested?
Is disk usage monitored?

OPERATING SYSTEM

Is the OS supported?
Is patching managed?
Are unnecessary services disabled?
Are logs managed?

APPLICATION

Does it start automatically?
Is there a health check?
Are errors monitored?
Can it be redeployed?

RECOVERY

Can the server be rebuilt?
Is there a current AMI or build process?
Can data be restored?
Are RPO and RTO understood?

OPERATIONS

Are resources tagged?
Is monitoring configured?
Are alerts actionable?
Is capacity reviewed?
Are changes documented or automated?

COST

Is the instance right-sized?
Is storage right-sized?
Are old snapshots retained intentionally?
Are unused volumes removed?
```

This is much closer to real EC2 operations than simply knowing how to click "Launch Instance."

---

# 121. Our Architecture After Part 4

After Part 3:

```text
Laptop
   |
Internet
   |
EC2
   |
Nginx
   |
Application
```

After Part 4, you should see:

```text
                     AWS
                      |
                 EC2 Instance
                      |
        +-------------+-------------+
        |             |             |
        v             v             v
      Linux       IAM Role      Monitoring
        |
        +-----------------------------+
        |                             |
        v                             v
   Root EBS                       Data EBS
        |                             |
        v                             v
   Filesystem                    Filesystem
        |                             |
        v                             v
        /                           /data
                                      |
                                      v
                              Application Data

EBS Volumes
     |
     v
Snapshots
     |
     v
Recovery

Configured EC2
     |
     v
Custom AMI
     |
     v
Replacement Instances
```

Now EC2 is no longer just:

```text
A server.
```

You understand:

```text
Compute
Storage
Filesystem
Mounting
Backup
Images
Monitoring
Recovery
Operations
```

---

# What You Should Now Understand

After Part 4, you should understand that:

* Compute and storage are different infrastructure layers.
* Ephemeral storage should not contain the only copy of critical data.
* Instance store is an example of ephemeral EC2 storage.
* EBS provides persistent block storage.
* An EBS volume behaves conceptually like a virtual disk.
* Root EBS volumes commonly contain the operating system.
* Additional EBS volumes can store application data.
* EBS volumes and EC2 instances must normally be in the same Availability Zone for direct attachment.
* Different EBS volume types serve different performance requirements.
* IOPS and throughput describe different aspects of storage performance.
* Linux sees EBS as block devices.
* `lsblk` discovers block devices.
* `df -h` reports mounted filesystem capacity.
* A disk, partition, filesystem, and mount point are different layers.
* Partitioning divides a block device.
* `mkfs` creates a filesystem and can destroy existing data.
* Mounting connects a filesystem to the Linux directory hierarchy.
* `/etc/fstab` defines persistent mount configuration.
* UUIDs provide stable filesystem identification.
* `mount -a` can help test `/etc/fstab` before reboot.
* Increasing an EBS volume does not necessarily automatically expand its partition and filesystem.
* Filesystem-specific tools are required for storage expansion.
* EBS snapshots provide point-in-time storage recovery capabilities.
* Snapshots can form part of a backup strategy.
* Backup and high availability are not the same thing.
* Backups must be restorable.
* RPO describes acceptable data loss.
* RTO describes acceptable recovery time.
* Restored snapshot volumes already contain data and should not be reformatted.
* Custom AMIs can provide repeatable server images.
* Golden images help standardize infrastructure.
* Secrets should not be baked into reusable AMIs.
* EC2 recovery should begin with identifying the failed layer.
* Status checks help distinguish certain AWS infrastructure and guest-instance failures.
* CloudWatch provides infrastructure metrics.
* Guest-level monitoring may require an agent.
* A running EC2 instance does not prove the application is healthy.
* Production EC2 servers should be reproducible, monitored, backed up, patched, secured, and replaceable.

---

# Hands-On Practice

## Exercise 1: Inspect Your Existing EC2 Storage

On the Part 3 instance:

```bash
lsblk
```

Then:

```bash
df -h
```

Then:

```bash
df -T
```

Identify:

* Root disk
* Root partition
* Filesystem type
* Root filesystem size
* Used capacity
* Available capacity

Explain the difference between what `lsblk` and `df` show.

---

# Exercise 2: Attach a New EBS Volume

Create a small disposable EBS volume.

Ensure:

```text
EBS Availability Zone
=
EC2 Availability Zone
```

Attach it.

SSH into EC2.

Run:

```bash
lsblk
```

Identify the new device.

Do not continue until you are certain which disk is new.

---

# Exercise 3: Create Storage

On the verified new disk:

1. Create a partition if using the partitioned approach.
2. Create an ext4 filesystem.
3. Create `/data`.
4. Mount the filesystem.
5. Verify using `df -h`.
6. Create a test file.
7. Verify the file.

The final architecture should be:

```text
EBS
 |
Block Device
 |
Partition
 |
ext4
 |
/data
 |
storage-test.txt
```

---

# Exercise 4: Configure Persistent Mounting

Retrieve the UUID:

```bash
sudo blkid
```

Back up:

```bash
sudo cp /etc/fstab /etc/fstab.backup
```

Add the appropriate `/etc/fstab` entry.

Test:

```bash
sudo mount -a
```

Verify:

```bash
df -h
```

Then reboot the disposable lab instance.

After reconnecting:

```bash
df -h
```

Verify `/data` mounted automatically.

---

# Exercise 5: Snapshot and Restore

Create:

```bash
echo "VERIQTA recovery lab" | sudo tee /data/recovery.txt
```

Create an EBS snapshot.

Restore the snapshot to a new volume.

Attach the restored volume.

Discover it with:

```bash
lsblk
```

Do not format it.

Mount it at:

```text
/recovery
```

Verify:

```bash
cat /recovery/recovery.txt
```

Expected:

```text
VERIQTA recovery lab
```

You have now tested a restore instead of merely creating a backup.

---

# Exercise 6: Disk Usage Investigation

Run:

```bash
df -h
```

Then inspect directory usage:

```bash
sudo du -xhd1 /
```

Identify:

* Largest filesystem
* Largest top-level directories
* Which directories contain logs
* Which contain application data

Do not delete anything merely because it is large.

Understand it first.

---

# Exercise 7: Service Troubleshooting

Check:

```bash
sudo systemctl status nginx
```

Stop Nginx intentionally:

```bash
sudo systemctl stop nginx
```

Test:

```bash
curl http://localhost
```

Observe the failure.

Check:

```bash
sudo systemctl status nginx
```

Start it:

```bash
sudo systemctl start nginx
```

Verify:

```bash
curl http://localhost
```

You have performed:

```text
Break
 |
Observe
 |
Investigate
 |
Fix
 |
Verify
```

---

# Exercise 8: Create a Custom AMI

On a disposable configured EC2 instance:

1. Install Nginx.
2. Create your VERIQTA lab page.
3. Verify the application.
4. Create a custom AMI.
5. Launch a new EC2 instance from the custom AMI.
6. Verify which configuration was preserved.
7. Identify what still needs environment-specific configuration.

Do not put secrets into the image.

---

# Exercise 9: Review Monitoring

From the EC2 console, inspect:

* CPU utilization
* Network metrics
* Status checks

Then SSH into the server and compare with:

```bash
uptime
```

```bash
free -h
```

```bash
df -h
```

```bash
top
```

Understand the distinction between:

```text
AWS-level metrics
```

and:

```text
Guest operating-system information
```

---

# Exercise 10: Clean Up

After completing the lab:

* Terminate disposable instances you no longer need.
* Delete unused EBS volumes.
* Review snapshots.
* Delete lab snapshots you intentionally no longer need.
* Review AMIs created for the exercise.
* Review associated snapshots before deregistering/deleting image-related resources.
* Check your AWS cost tools.

Do not leave resources running simply because they were part of a tutorial.

---

# Beginner Knowledge Check

1. What is the difference between compute and storage?

2. What does ephemeral mean?

3. What is EC2 instance store?

4. Should instance store contain the only copy of critical data?

5. What does EBS stand for?

6. What type of storage does EBS provide?

7. What is an EBS volume?

8. What is a root volume?

9. Can an EC2 instance have more than one EBS volume?

10. Why does Availability Zone placement matter for EBS?

11. What is IOPS?

12. What is throughput?

13. Why are there different EBS volume types?

14. What does `lsblk` show?

15. What does `df -h` show?

16. What is the difference between a disk and partition?

17. What is a filesystem?

18. What does `mkfs` do?

19. Why is `mkfs` dangerous on an existing data volume?

20. What is a mount point?

21. What does `mount` do?

22. What is `/etc/fstab`?

23. Why can UUIDs be useful in `/etc/fstab`?

24. Why should `/etc/fstab` be tested before reboot?

25. What does `mount -a` do?

26. If an EBS volume grows from 20 GB to 40 GB, why might Linux still show 20 GB?

27. Why do ext4 and XFS require different resize commands?

28. What is an EBS snapshot?

29. Are snapshots a replacement for all backup planning?

30. What is the difference between backup and high availability?

31. What does RPO measure?

32. What does RTO measure?

33. Why should backups be restore-tested?

34. When restoring a volume from a snapshot, why should you not immediately run `mkfs`?

35. What is a custom AMI?

36. What is a golden image?

37. Why should secrets not be baked into an AMI?

38. What is image drift?

39. What is a system status check?

40. What is an instance status check?

41. Why can an EC2 instance be running while the application is unavailable?

42. What does `free -h` show?

43. What does `systemctl status nginx` help determine?

44. What does `journalctl -u nginx` show?

45. Why should you test an application with `curl localhost` during troubleshooting?

46. What is a snowflake server?

47. Why should cloud servers be replaceable?

48. Why should backups be automated?

49. Why should disk usage be monitored before reaching 100%?

50. What should happen before a risky production storage change?

---

# Knowledge Check Answers

1. Compute provides processing resources such as CPU and memory. Storage holds operating systems, applications, files, and data.

2. Temporary or tied to a resource lifecycle in a way that should not be considered durable.

3. Local storage physically associated with the underlying EC2 host infrastructure and available to supported instance types.

4. No.

5. Elastic Block Store.

6. Persistent block storage.

7. A virtual block-storage resource that can be attached to supported compute resources such as EC2.

8. The storage volume containing the operating system and root filesystem of an EBS-backed EC2 instance.

9. Yes.

10. EBS volumes are AZ-scoped and must normally be in the same AZ as an EC2 instance for direct attachment.

11. Input/Output Operations Per Second.

12. The amount of data transferred over time.

13. Different applications have different performance, latency, throughput, and cost requirements.

14. Linux block devices, partitions, sizes, and mount relationships.

15. Mounted filesystem capacity, usage, available space, and mount points.

16. A disk is the overall block device. A partition is a logical section of that disk.

17. A structure that organizes data into files, directories, metadata, and permissions.

18. It creates a filesystem.

19. It can overwrite filesystem structures and destroy existing data.

20. A directory where a filesystem becomes accessible in the Linux directory hierarchy.

21. It attaches a filesystem to a mount point.

22. A Linux configuration file describing filesystems and how they should be mounted.

23. UUIDs provide persistent filesystem identifiers rather than relying only on runtime device names.

24. A bad entry can interfere with boot or filesystem mounting.

25. It attempts to mount eligible filesystems defined in `/etc/fstab`.

26. The block device may have grown while the partition and/or filesystem has not yet been expanded.

27. Different filesystem implementations use different management tools.

28. A point-in-time backup mechanism for EBS volumes.

29. No. Snapshots are one component of a broader backup and recovery strategy.

30. High availability helps keep services operating during failures. Backups help recover lost or damaged data/system state.

31. The acceptable amount of data loss measured in time.

32. The target amount of time allowed to restore service.

33. Because a backup has little operational value if it cannot be restored correctly when needed.

34. A restored snapshot already contains a filesystem and data. Formatting it can destroy the data.

35. An Amazon Machine Image created from a configured system for future instance launches.

36. A standardized approved machine image used as a controlled server baseline.

37. Every instance launched from the image could receive copies of those secrets.

38. The growing difference between an old image and current security/configuration standards.

39. A check focused on certain underlying AWS infrastructure problems affecting an instance.

40. A check focused more on the individual EC2 instance and guest operating-system reachability.

41. EC2 state only tells you the virtual machine is running. The application, web server, storage, or configuration can still fail.

42. Memory usage information.

43. Whether the Nginx service is running and information about its state.

44. Journal entries associated with the Nginx systemd service.

45. It verifies whether the application works locally, helping separate application failures from external network failures.

46. A manually modified server that becomes difficult or impossible to reproduce reliably.

47. So failure can be handled by rebuilding or replacing infrastructure instead of depending permanently on one machine.

48. Manual backup processes are inconsistent and easy to forget.

49. Full filesystems can cause application, logging, database, and operating-system failures.

50. Understand the change, verify backups/recovery where appropriate, preserve a recovery point, execute carefully, and verify afterward.

---

# Part 4 Complete

In Part 3, you learned:

```text
How to create a server.
```

In Part 4, you learned:

```text
How to operate and recover one.
```

The complete mental model is now:

```text
                       EC2
                        |
        +---------------+---------------+
        |               |               |
        v               v               v
     Compute         Network          Storage
                                        |
                                        v
                                      EBS
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
                                   Mount Point
                                        |
                                        v
                                Application Data
                                        |
                                        v
                                    Snapshot
                                        |
                                        v
                                     Restore
```

And for reproducible compute:

```text
Configured EC2
      |
      v
Custom AMI
      |
      v
New EC2 Instances
```

The larger operational lesson is:

```text
Do not build a server
that you are afraid to lose.

Build a system
that you know how to recover.
```

Part 5 moves beneath the server into the network itself: **VPC fundamentals, IPv4 addressing, CIDR notation, public and private addressing, network ranges, ENIs, Elastic IPs, DNS, and designing your first custom AWS network.**
