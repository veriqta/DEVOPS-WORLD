# Part 5: Linux Permissions, Ownership, and Access Control

In Part 4, you learned how Linux identifies users and groups.

You learned that every user has a UID, every group has a GID, and processes run under particular identities.

Now we connect identity to access.

Consider this question:

```text
Alice tries to open a file.

How does Linux decide whether Alice is allowed?
```

A simplified model is:

```text
User or Process
      |
      v
Linux Identity
UID + Groups
      |
      v
File or Directory
      |
      +-- Owner
      +-- Group
      +-- Permissions
      |
      v
Kernel checks access
      |
   +--+--+
   |     |
 Allow  Deny
```

This is the foundation of Linux filesystem security.

By the end of this part, you should understand not only how to use `chmod`, `chown`, and `chgrp`, but how to diagnose permission problems without blindly using `sudo` or `chmod 777`.

---

# 1. Why Linux Permissions Exist

Linux is a multi-user operating system.

A server might contain:

```text
Human users:
ann
alice
bob

Service accounts:
nginx
postgres
deploy

Administrator:
root
```

These identities should not automatically have access to everything.

Imagine a server containing:

```text
/home/ann/private-notes.txt
/etc/ssh/sshd_config
/var/log/application.log
/srv/company-app/
/home/deploy/.ssh/id_ed25519
```

Should every user be able to:

* Read private SSH keys?
* Modify SSH configuration?
* Delete application files?
* Change system logs?
* Execute every script?
* Modify another user's home directory?

No.

Linux permissions provide a basic access-control system.

They answer questions such as:

```text
Who owns this file?

Which group owns it?

Can the owner read it?

Can the owner modify it?

Can the group read it?

Can anyone else access it?

Can this file be executed?
```

---

# 2. Authentication vs Authorization

Before going further, understand two related concepts.

Authentication asks:

```text
Who are you?
```

Authorization asks:

```text
What are you allowed to do?
```

For example:

```text
SSH login
    |
    v
Authentication
"Are you really Alice?"
    |
    v
Alice's Linux identity
    |
    v
Authorization
"Can Alice read this file?"
```

Permissions are primarily about authorization.

Successfully logging into Linux does not mean you can access everything.

---

# 3. Linux Permission Mental Model

Traditional Linux permissions revolve around three permission classes:

```text
User
Group
Others
```

You will also hear `user` described as the file's `owner`.

For a resource, Linux stores information resembling:

```text
File
|
+-- User owner
|
+-- Group owner
|
+-- User permissions
|
+-- Group permissions
|
+-- Other permissions
```

For example:

```text
-rw-r----- 1 alice developers 4096 Aug 26 10:30 app.conf
```

This tells us:

```text
Owner:
alice

Group:
developers

Owner permissions:
rw-

Group permissions:
r--

Others:
---
```

We will decode every part.

---

# 4. Start With `ls -l`

One of the most important commands for permission troubleshooting is:

```bash
ls -l
```

The `-l` option means long listing format.

Suppose:

```bash
ls -l app.conf
```

returns:

```text
-rw-r----- 1 alice developers 4096 Aug 26 10:30 app.conf
```

At first, this looks complicated.

Break it into fields:

```text
-rw-r-----  1  alice  developers  4096  Aug 26 10:30  app.conf
|           |    |        |         |         |           |
|           |    |        |         |         |           +-- Name
|           |    |        |         |         +-------------- Time
|           |    |        |         +------------------------ Size
|           |    |        +---------------------------------- Group
|           |    +------------------------------------------- Owner
|           +------------------------------------------------ Link count
+------------------------------------------------------------ Type + permissions
```

For this part, our main focus is:

```text
-rw-r-----
alice
developers
```

---

# 5. Understanding the Permission String

Look at:

```text
-rw-r-----
```

It contains ten characters.

Separate them:

```text
- | rw- | r-- | ---
^     ^     ^     ^
|     |     |     |
|     |     |     +-- Others
|     |     +-------- Group
|     +-------------- User owner
+-------------------- File type
```

This structure is essential.

Think:

```text
Position 1     Resource type
Positions 2-4  Owner permissions
Positions 5-7  Group permissions
Positions 8-10 Other permissions
```

---

# 6. File Type Indicator

The first character is not a permission.

It describes the resource type.

For a normal file:

```text
-
```

Example:

```text
-rw-r--r--
```

For a directory:

```text
d
```

Example:

```text
drwxr-xr-x
```

For a symbolic link:

```text
l
```

Example:

```text
lrwxrwxrwx
```

Other special file types exist, but these three are the most important for beginners.

Remember:

```text
- = regular file
d = directory
l = symbolic link
```

Do not include the first character when converting ordinary permissions to numeric notation.

---

# 7. User Owner

Every file and directory has a user owner.

Example:

```text
-rw-r----- 1 alice developers 4096 Aug 26 10:30 app.conf
```

The owner is:

```text
alice
```

The first permission triplet:

```text
rw-
```

applies to Alice as the owner.

This means Alice can:

```text
r = read
w = write
- = not execute
```

---

# 8. Group Owner

The same file has a group owner:

```text
developers
```

The second permission triplet:

```text
r--
```

applies to the group class.

Members whose access is evaluated through that group can read the file but cannot write or execute it.

---

# 9. Others

The final triplet applies to:

```text
others
```

Others means users who are not receiving access through the owner or owning-group permission class for this check.

For:

```text
-rw-r-----
```

others receive:

```text
---
```

No read.

No write.

No execute.

---

# 10. The Three Basic Permissions

Traditional Linux permissions use:

```text
r = read
w = write
x = execute
```

But these permissions do not mean exactly the same thing for files and directories.

That distinction is critical.

---

# 11. Read Permission on a File

For a regular file:

```text
r
```

means the file's contents can be read.

For example:

```bash
cat report.txt
```

requires appropriate read access to the file, along with the necessary directory access to reach it.

Other programs that read the file also need read access.

Examples include:

```bash
less report.txt
head report.txt
tail report.txt
```

---

# 12. Write Permission on a File

For a regular file:

```text
w
```

means the file's contents can be modified.

For example, a text editor may be able to change:

```text
config.txt
```

if the user has appropriate write access.

Write permission does not necessarily mean the user can delete the file.

This surprises beginners.

Deletion depends heavily on permissions on the containing directory.

We will explain that shortly.

---

# 13. Execute Permission on a File

For a regular file:

```text
x
```

means Linux permits the file to be executed directly, assuming it is otherwise a valid executable or script and other requirements are satisfied.

Suppose:

```bash
ls -l deploy.sh
```

shows:

```text
-rw-r--r-- 1 ann ann 150 Aug 26 10:30 deploy.sh
```

There is no `x`.

Trying:

```bash
./deploy.sh
```

may produce:

```text
Permission denied
```

After appropriate execute permission is granted, direct execution may become possible.

---

# 14. `rwx`

The three permission positions are always conceptually:

```text
rwx
```

Examples:

```text
rwx
```

Read, write, execute.

```text
rw-
```

Read, write, no execute.

```text
r-x
```

Read, no write, execute.

```text
r--
```

Read only.

```text
---
```

No permissions in that class.

---

# 15. File Permissions vs Directory Permissions

This is one of the most important concepts in this entire part.

For files:

```text
r = read contents
w = modify contents
x = execute file
```

For directories, the meanings are different.

Simplified:

```text
r = list directory entries
w = modify directory entries
x = traverse or access entries through the directory
```

Let's examine each.

---

# 16. Read Permission on a Directory

For a directory:

```text
r
```

allows reading its directory entries, which is what listing filenames depends on.

For example:

```bash
ls project/
```

requires relevant directory permissions.

Without directory read permission, you may not be able to obtain the normal listing of names.

---

# 17. Write Permission on a Directory

Directory:

```text
w
```

allows modification of directory entries, subject to other required permissions and access controls.

This is related to operations such as:

* Creating files
* Removing files
* Renaming files
* Creating subdirectories

But directory write permission normally needs directory execute permission to be practically useful for these operations.

---

# 18. Execute Permission on a Directory

For directories:

```text
x
```

means traversal or search permission.

It allows you to pass through the directory and access entries when you know their names, subject to permissions on the entries and other path components.

This is why `x` on a directory does not mean:

```text
Run the directory.
```

It means:

```text
Traverse/search the directory.
```

---

# 19. Directory Permission Example

Imagine:

```text
/projects
    |
    +-- app/
         |
         +-- config.yaml
```

To reach:

```text
/projects/app/config.yaml
```

you need appropriate traversal access through directories in the path.

Conceptually:

```text
/
|
v
projects
x needed to traverse
|
v
app
x needed to traverse
|
v
config.yaml
r needed to read contents
```

A file can have read permission and still be inaccessible if you cannot traverse the directory path leading to it.

This is a major permission troubleshooting lesson.

---

# 20. Why File Deletion Depends on the Directory

Suppose:

```text
-r--r--r-- 1 alice developers 500 report.txt
```

The file itself is read-only.

Can someone still delete it?

Potentially, yes.

Why?

Because deleting a filename changes the containing directory's entries.

The operation is fundamentally associated with the directory.

This means:

```text
File write permission
```

and:

```text
Permission to delete the file from a directory
```

are not the same thing.

The sticky bit, which we cover later, can further restrict deletion in shared directories.

---

# 21. Symbolic Permissions

Permissions written with:

```text
r
w
x
-
```

are often called symbolic permissions.

Example:

```text
rwxr-x---
```

Break it into:

```text
rwx  r-x  ---
 |    |     |
 |    |     +-- Others
 |    +-------- Group
 +------------- Owner
```

Owner:

```text
rwx
```

Group:

```text
r-x
```

Others:

```text
---
```

---

# 22. Numeric Permissions

Linux permissions can also be represented numerically.

Examples:

```text
755
750
700
644
640
600
```

To understand these numbers, you need to understand how:

```text
4
2
1
```

represent the three basic permissions.

---

# 23. Understanding 4, 2, and 1

The permission values are:

```text
Read     = 4
Write    = 2
Execute  = 1
```

You combine them by addition.

For example:

```text
Read + Write

4 + 2 = 6
```

So:

```text
6 = rw-
```

Another:

```text
Read + Execute

4 + 1 = 5
```

Therefore:

```text
5 = r-x
```

All three:

```text
4 + 2 + 1 = 7
```

Therefore:

```text
7 = rwx
```

---

# 24. Complete Numeric Permission Table

| Number | Calculation | Permission |
| -----: | ----------- | ---------- |
|      0 | 0           | `---`      |
|      1 | 1           | `--x`      |
|      2 | 2           | `-w-`      |
|      3 | 2 + 1       | `-wx`      |
|      4 | 4           | `r--`      |
|      5 | 4 + 1       | `r-x`      |
|      6 | 4 + 2       | `rw-`      |
|      7 | 4 + 2 + 1   | `rwx`      |

You do not need to memorize this immediately.

With practice, values such as `755`, `644`, and `600` will become familiar.

---

# 25. Why This Is Sometimes Called a Binary Permission Model

Each permission can be thought of as either enabled or disabled.

Conceptually:

```text
Permission    Value

r             4
w             2
x             1
```

These correspond to three binary positions:

```text
r w x
4 2 1
```

For:

```text
r-x
```

we have:

```text
1 0 1
```

Binary `101` represents decimal `5`.

For:

```text
rw-
```

we have:

```text
1 1 0
```

Binary `110` represents decimal `6`.

For:

```text
rwx
```

we have:

```text
1 1 1
```

Binary `111` represents decimal `7`.

You do not need advanced binary mathematics to manage Linux permissions. The practical rule is:

```text
r = 4
w = 2
x = 1
```

---

# 26. Understanding Three-Digit Permissions

Consider:

```text
750
```

Each digit represents a permission class:

```text
7    5    0
|    |    |
|    |    +-- Others
|    +------- Group
+------------ Owner
```

Convert each digit:

```text
7 = rwx
5 = r-x
0 = ---
```

Therefore:

```text
750 = rwxr-x---
```

---

# 27. Understanding `777`

```text
777
```

means:

```text
Owner   = 7 = rwx
Group   = 7 = rwx
Others  = 7 = rwx
```

Therefore:

```text
rwxrwxrwx
```

Everyone receives read, write, and execute permissions through the basic mode bits.

This is extremely permissive.

`chmod 777` is commonly abused as a quick fix for permission problems.

Do not develop that habit.

---

# 28. Why `chmod 777` Is Usually the Wrong Fix

Suppose an application cannot write to:

```text
/srv/app/uploads
```

A beginner searches online and runs:

```bash
sudo chmod -R 777 /srv/app
```

The application starts working.

But what actually happened?

Instead of identifying the access requirement, you granted extremely broad permissions across an entire directory tree.

The original problem may have been:

* Wrong owner
* Wrong group
* Missing group membership
* Incorrect service account
* One directory missing write access
* Incorrect application configuration

The correct question is not:

```text
How do I make Permission denied disappear?
```

It is:

```text
Which identity needs which access to which resource?
```

---

# 29. Understanding `755`

```text
755
```

becomes:

```text
7 = rwx
5 = r-x
5 = r-x
```

Therefore:

```text
rwxr-xr-x
```

This is commonly seen on directories and executable files.

It means:

```text
Owner:
read + write + execute

Group:
read + execute

Others:
read + execute
```

Whether `755` is appropriate depends on the resource.

Do not assume every directory should be `755`.

---

# 30. Understanding `750`

```text
750
```

means:

```text
Owner   rwx
Group   r-x
Others  ---
```

Symbolically:

```text
rwxr-x---
```

This can be useful when the owner needs full access, a group needs read/traversal access, and everyone else should receive none.

---

# 31. Understanding `700`

```text
700
```

means:

```text
Owner   rwx
Group   ---
Others  ---
```

Symbolically:

```text
rwx------
```

This is restrictive.

A private directory might use `700` when only its owner should access it through these basic permission classes.

---

# 32. Understanding `644`

```text
644
```

means:

```text
Owner   rw-
Group   r--
Others  r--
```

Symbolically:

```text
rw-r--r--
```

This is commonly seen for non-sensitive regular files intended to be readable by others but writable only by the owner.

Do not use `644` blindly for secrets or private files.

---

# 33. Understanding `640`

```text
640
```

means:

```text
Owner   rw-
Group   r--
Others  ---
```

Symbolically:

```text
rw-r-----
```

This can be useful when:

* Owner needs read/write
* Group needs read
* Everyone else should have no access

---

# 34. Understanding `600`

```text
600
```

means:

```text
Owner   rw-
Group   ---
Others  ---
```

Symbolically:

```text
rw-------
```

This is commonly appropriate for sensitive files that should be readable and writable only by their owner.

A common example is an SSH private key.

---

# 35. Permission Comparison

| Numeric | Symbolic    | Typical meaning                 |
| ------: | ----------- | ------------------------------- |
|     777 | `rwxrwxrwx` | Everyone gets everything        |
|     755 | `rwxr-xr-x` | Owner full, others read/execute |
|     750 | `rwxr-x---` | Owner full, group read/execute  |
|     700 | `rwx------` | Owner only                      |
|     644 | `rw-r--r--` | Owner writes, everyone reads    |
|     640 | `rw-r-----` | Owner writes, group reads       |
|     600 | `rw-------` | Owner read/write only           |

These are patterns, not universal prescriptions.

---

# 36. Changing Permissions With `chmod`

`chmod` means:

```text
change mode
```

It changes permission mode bits on files and directories.

General forms include:

```bash
chmod MODE FILE
```

For example:

```bash
chmod 600 private.txt
```

or:

```bash
chmod u+x deploy.sh
```

`chmod` supports both:

* Numeric mode
* Symbolic mode

---

# 37. Numeric `chmod`

Suppose:

```bash
ls -l report.txt
```

shows:

```text
-rw-r--r-- 1 ann ann 1000 Aug 26 11:00 report.txt
```

You want only the owner to read and write it.

Run:

```bash
chmod 600 report.txt
```

Verify:

```bash
ls -l report.txt
```

Expected permission portion:

```text
-rw-------
```

Breakdown:

```text
6 = rw-
0 = ---
0 = ---
```

---

# 38. Making a Script Executable Numerically

Suppose:

```bash
ls -l deploy.sh
```

shows:

```text
-rw-r--r--
```

You could use:

```bash
chmod 755 deploy.sh
```

Result:

```text
-rwxr-xr-x
```

But ask whether everyone actually needs execute or read access.

Perhaps:

```bash
chmod 750 deploy.sh
```

is more appropriate.

Or perhaps only the owner needs execution:

```bash
chmod 700 deploy.sh
```

Permissions should match requirements.

---

# 39. Symbolic `chmod`

Symbolic mode allows you to express changes such as:

```text
Add execute permission to the owner.
```

Instead of replacing the entire mode.

The classes are:

```text
u = user owner
g = group
o = others
a = all classes
```

Operations include:

```text
+ = add
- = remove
= = set exactly
```

Permissions remain:

```text
r
w
x
```

---

# 40. `chmod u+x`

Suppose:

```text
-rw-r--r--
```

You want to add execute permission only for the owner.

Run:

```bash
chmod u+x deploy.sh
```

Breakdown:

```text
u
User owner

+
Add

x
Execute
```

Result:

```text
-rwxr--r--
```

Existing read/write permissions remain.

---

# 41. Removing Write Permission

Suppose:

```text
-rw-rw-r--
```

You want to remove group write permission.

Run:

```bash
chmod g-w report.txt
```

Result:

```text
-rw-r--r--
```

---

# 42. Adding Permissions to Multiple Classes

You can use:

```bash
chmod go+r report.txt
```

Meaning:

```text
g = group
o = others
+ = add
r = read
```

This adds read permission for group and others.

---

# 43. Setting Exact Symbolic Permissions

You can use:

```bash
chmod u=rw,g=r,o= report.txt
```

This means:

```text
Owner:
rw-

Group:
r--

Others:
---
```

Equivalent basic permissions:

```text
640
```

The final:

```text
o=
```

means give others no basic permissions.

---

# 44. Numeric vs Symbolic `chmod`

Numeric mode is useful when you know the exact final permission state.

Example:

```bash
chmod 640 app.conf
```

Symbolic mode is useful when you want to modify a particular permission while preserving others.

Example:

```bash
chmod g+w shared.txt
```

Think:

```text
Numeric:
Set the permission pattern.

Symbolic:
Add, remove, or explicitly set selected permissions.
```

Both are important.

---

# 45. Who Can Change Permissions?

Normally, the file owner can change the mode of their file, subject to Linux rules.

Root can also perform broad administrative changes.

If:

```bash
chmod 600 file.txt
```

returns:

```text
Operation not permitted
```

check:

```bash
whoami
```

and:

```bash
ls -l file.txt
```

Do not immediately use `sudo`.

First determine whether you are supposed to own or administer that resource.

---

# 46. Ownership

Permissions alone are not enough.

You must also know:

```text
Who owns the file?

Which group owns the file?
```

Inspect:

```bash
ls -l app.conf
```

Example:

```text
-rw-r----- 1 alice developers 2048 Aug 26 11:10 app.conf
```

Owner:

```text
alice
```

Group:

```text
developers
```

Changing the permission string without understanding ownership can leave the actual access problem unresolved.

---

# 47. Changing Ownership With `chown`

`chown` means:

```text
change owner
```

General form:

```bash
chown NEWOWNER FILE
```

Changing file ownership generally requires appropriate privileges.

Example:

```bash
sudo chown alice report.txt
```

Verify:

```bash
ls -l report.txt
```

---

# 48. Changing Owner and Group Together

You can use:

```bash
sudo chown alice:developers report.txt
```

This means:

```text
User owner:
alice

Group owner:
developers
```

Verify:

```bash
ls -l report.txt
```

You might see:

```text
-rw-r----- 1 alice developers 1000 Aug 26 11:20 report.txt
```

---

# 49. Changing Only the Group With `chown`

You may also encounter:

```bash
sudo chown :developers report.txt
```

The missing username means the user owner remains unchanged while the group is changed.

However, Linux also provides a dedicated command for group ownership:

```bash
chgrp
```

---

# 50. Changing Groups With `chgrp`

`chgrp` means:

```text
change group
```

Example:

```bash
chgrp developers report.txt
```

Whether you can perform the change depends on your identity, ownership, and group memberships.

An administrator can use:

```bash
sudo chgrp developers report.txt
```

Verify:

```bash
ls -l report.txt
```

---

# 51. User Group Membership vs File Group Ownership

Do not confuse these two ideas.

This:

```bash
sudo usermod -aG developers alice
```

changes Alice's supplementary group membership.

This:

```bash
sudo chgrp developers app.conf
```

changes the group owner of `app.conf`.

Conceptually:

```text
User identity:
alice
|
+-- developers


File:
app.conf
|
+-- Owner: bob
+-- Group: developers
```

The group provides a relationship between the user and resource.

---

# 52. How Linux Evaluates Basic Owner, Group, and Other Permissions

A useful beginner mental model is:

1. If the process identity matches the file's owner, Linux evaluates the owner permission class.

2. Otherwise, if the process has the file's owning group as an applicable group, Linux evaluates the group class.

3. Otherwise, Linux evaluates the others class.

This produces an important result.

Linux does not normally say:

```text
The owner permissions deny access, so I will try group, then others until something allows it.
```

The matching class matters.

For example, if Alice owns a file but owner permissions do not grant read access, being in the file's group does not simply cause Linux to fall through to the group's read bit.

This distinction helps explain some surprising permission failures.

---

# 53. Path Permissions Matter Too

Suppose:

```text
/srv/private/config.txt
```

has:

```text
-rw-r--r--
```

You might think:

```text
Others can read it.
```

But if:

```text
/srv/private
```

is:

```text
drwx------
```

a different user cannot traverse the directory.

The file may therefore remain inaccessible.

When troubleshooting a path, inspect directories too.

A useful command is:

```bash
namei -l /srv/private/config.txt
```

if `namei` is available.

It can help display permissions along the path.

Otherwise inspect components with `ls -ld`.

---

# 54. Recursive Changes

The `-R` option means recursive for commands such as `chmod`, `chown`, and `chgrp`.

For example:

```bash
sudo chown -R alice:developers /srv/project
```

This applies ownership changes through the directory tree.

Similarly:

```bash
chmod -R 755 project/
```

changes modes recursively.

These commands can affect many files in seconds.

That makes them powerful and dangerous.

---

# 55. Why Recursive Permission Changes Are Risky

Consider:

```text
application/
|
+-- scripts/
|
+-- configs/
|
+-- public/
|
+-- private-keys/
|
+-- logs/
```

These resources should probably not all have identical permissions.

If you run:

```bash
chmod -R 777 application/
```

you may give broad read, write, and execute access to:

* Configuration files
* Secrets
* Scripts
* Logs
* Directories
* Private data

You also add execute bits to regular files that may never need to be executable.

Recursive permission changes ignore context unless you design them carefully.

---

# 56. Another Recursive Mistake

Suppose you run:

```bash
chmod -R 644 project/
```

You might think:

```text
Great. Files are readable but not executable.
```

But directories also receive `644`.

For a directory:

```text
644 = rw-r--r--
```

There is no execute bit.

Without directory execute permission, traversal breaks.

You may suddenly be unable to enter directories or reach files.

This is a classic beginner mistake.

---

# 57. Safer Recursive Thinking

Before running a recursive command, ask:

```text
What exactly is underneath this directory?

Are there files and directories?

Should they have the same permissions?

Are there scripts?

Are there secrets?

Are there application-owned files?

Could a service depend on these modes?

Could this affect production?
```

Inspect first:

```bash
ls -la
```

and, when appropriate:

```bash
find /path -maxdepth 2 -ls
```

Do not make broad recursive changes simply because one file reports `Permission denied`.

---

# 58. Different Permissions for Files and Directories

Suppose a directory tree needs:

```text
Directories: 755
Regular files: 644
```

Instead of:

```bash
chmod -R 755 project
```

an administrator might intentionally target directories and files separately.

For example:

```bash
find project -type d -exec chmod 755 {} \;
```

and:

```bash
find project -type f -exec chmod 644 {} \;
```

Do not copy these commands blindly.

Understand them.

The first says:

```text
Find directories under project
and apply chmod 755 to each.
```

The second says:

```text
Find regular files under project
and apply chmod 644 to each.
```

You studied `find` earlier. This is a practical example of combining Linux commands.

---

# 59. Default Permissions

When you create a file, you do not normally specify permissions manually every time.

For example:

```bash
touch report.txt
```

The resulting mode is influenced by:

* The program creating the file
* Its requested creation mode
* The process's `umask`
* Possibly additional filesystem or access-control mechanisms

A common beginner simplification is:

```text
Files start from a maximum-style base of 666.

Directories start from a maximum-style base of 777.

umask removes permissions.
```

This model is useful for understanding normal shell-created files and directories.

---

# 60. Why New Files Usually Do Not Start as `777`

Regular files normally do not need execute permission simply because they were created.

A typical base for ordinary files is:

```text
666
```

which represents:

```text
rw-rw-rw-
```

Directories need execute permission for traversal, so the common base is:

```text
777
```

which represents:

```text
rwxrwxrwx
```

Then the `umask` removes selected permissions.

---

# 61. What Is `umask`?

`umask` means the file creation mode mask.

Check yours:

```bash
umask
```

You may see:

```text
0022
```

or:

```text
0002
```

or another value.

The exact default depends on the system and environment.

The mask tells the system which permission bits should be removed from requested creation permissions.

---

# 62. Understanding `umask 022`

Suppose:

```text
umask = 022
```

For a typical regular file:

```text
Base:
666
```

Remove group write and other write.

Result:

```text
644
```

Symbolically:

```text
rw-r--r--
```

For a directory:

```text
Base:
777
```

Apply the same mask.

Result:

```text
755
```

Symbolically:

```text
rwxr-xr-x
```

So a common `022` umask produces:

```text
Files:
644

Directories:
755
```

for typical creation requests.

---

# 63. Understanding `umask 002`

With:

```text
umask 002
```

a typical new file may become:

```text
664
```

which is:

```text
rw-rw-r--
```

A new directory may become:

```text
775
```

which is:

```text
rwxrwxr-x
```

This is more group-friendly because group write permission is not masked.

It can be useful in collaborative environments when combined with appropriate ownership and group design.

---

# 64. Understanding `umask 077`

A more restrictive mask is:

```text
077
```

Typical file:

```text
600
```

Typical directory:

```text
700
```

Meaning other users and groups receive no basic permissions.

This can be useful for private resources.

---

# 65. `umask` Is a Mask, Not a Final Permission Number

A common beginner mistake is thinking:

```bash
umask 022
```

means:

```text
Make files permission 022.
```

It does not.

It means roughly:

```text
Remove the permission bits represented by the mask from permissions requested during creation.
```

The final result depends on what the creating program requested.

This is why describing `umask` as simply "subtract this number" is an oversimplification.

For common examples, subtraction appears to work, but thinking in terms of masked bits is more accurate.

---

# 66. Testing `umask`

Check:

```bash
umask
```

Suppose the output is:

```text
0022
```

Create:

```bash
touch umask-file.txt
```

Then:

```bash
mkdir umask-directory
```

Inspect:

```bash
ls -ld umask-file.txt umask-directory
```

You may see permissions resembling:

```text
-rw-r--r-- ... umask-file.txt
drwxr-xr-x ... umask-directory
```

This lets you observe `umask` rather than simply memorize it.

---

# 67. Temporarily Changing `umask`

You can change the mask for the current shell:

```bash
umask 077
```

Then create:

```bash
touch private-file.txt
mkdir private-directory
```

Inspect:

```bash
ls -ld private-file.txt private-directory
```

You may see:

```text
-rw------- ... private-file.txt
drwx------ ... private-directory
```

Changing `umask` in a shell affects subsequent creations in that environment.

Be careful when experimenting because later files created from that shell may receive different defaults.

---

# 68. Special Permission Bits

Traditional `rwx` permissions are not the entire story.

Linux also supports special mode bits:

```text
SUID
SGID
Sticky bit
```

At beginner level, you need to understand what they are and why they exist.

Do not randomly apply them.

They can have important security consequences.

---

# 69. SUID Introduction

SUID means:

```text
Set User ID
```

When applied to certain executable files, SUID can cause the program to run with the effective user identity of the file's owner rather than simply the invoking user's identity.

This is powerful.

It exists because some operations require controlled privileged behavior.

---

# 70. A Common SUID Example

On many Linux systems, inspect:

```bash
ls -l /usr/bin/passwd
```

You may see something resembling:

```text
-rwsr-xr-x 1 root root ... /usr/bin/passwd
```

Notice:

```text
rws
```

instead of:

```text
rwx
```

The:

```text
s
```

in the owner's execute position indicates SUID together with execute permission.

Why would `passwd` need special privilege?

A regular user needs to change their password, but sensitive authentication data is protected.

The `passwd` program is designed to perform the necessary privileged operation in a controlled way.

The exact implementation and security mechanisms vary, but this demonstrates why special permission mechanisms exist.

---

# 71. Setting SUID

For learning purposes, recognize syntax such as:

```bash
chmod u+s executable
```

or a numeric form using a leading special-permission digit.

But do not experiment by adding SUID to arbitrary executables on important systems.

SUID programs can create serious security risks if poorly designed or misconfigured.

For now, focus on recognizing and understanding SUID.

---

# 72. SGID Introduction

SGID means:

```text
Set Group ID
```

Its behavior depends on whether it is applied to an executable file or a directory.

On executable files, it can affect effective group identity.

On directories, it has a particularly useful collaboration behavior.

Files and subdirectories created inside an SGID directory can inherit the directory's group ownership rather than simply following the creator's normal primary-group behavior.

---

# 73. SGID Directory Example

Imagine:

```text
/srv/devops-project
```

is shared by the:

```text
developers
```

group.

You want files created inside to continue belonging to:

```text
developers
```

rather than constantly receiving different group ownership depending on who creates them.

SGID on the directory can help.

Conceptually:

```text
/srv/devops-project
Group: developers
SGID enabled
        |
        +-- alice creates app.conf
        |       |
        |       +-- Group: developers
        |
        +-- bob creates deploy.sh
                |
                +-- Group: developers
```

This is useful for collaborative directories.

---

# 74. Recognizing SGID

A directory might appear as:

```text
drwxrws--- alice developers shared
```

Notice the:

```text
s
```

in the group execute position:

```text
rws
```

That indicates SGID with group execute permission.

You may see it configured using:

```bash
chmod g+s shared
```

Again, use special bits intentionally.

---

# 75. Sticky Bit Introduction

The sticky bit is especially important on shared writable directories.

A classic example is:

```text
/tmp
```

Inspect:

```bash
ls -ld /tmp
```

You may see:

```text
drwxrwxrwt
```

Notice the final:

```text
t
```

This represents the sticky bit in the others execute position.

---

# 76. Why `/tmp` Uses the Sticky Bit

`/tmp` is commonly writable by many users.

Without additional restrictions, broad directory write access could allow users to remove or rename other users' entries more freely.

The sticky bit restricts deletion and renaming within the directory so that, broadly speaking, users cannot simply delete one another's files merely because the directory is writable to them. The file owner, directory owner, and privileged users retain relevant control.

Conceptually:

```text
Shared directory
Everyone can create files
        |
        v
Sticky bit
        |
        v
Alice cannot simply delete Bob's file
just because the directory is shared
```

This makes sticky bit valuable for shared writable directories.

---

# 77. Setting the Sticky Bit

Symbolically:

```bash
chmod +t shared-directory
```

A common numeric example is:

```text
1777
```

where the leading `1` represents the sticky bit.

Again, `1777` should not be used simply because `/tmp` uses it.

The permission design must match the directory's purpose.

---

# 78. Four-Digit Permission Notation

You may encounter:

```text
4755
2755
1777
```

The first digit represents special mode bits.

Common values include:

```text
4 = SUID
2 = SGID
1 = Sticky bit
```

The remaining three digits are the familiar:

```text
Owner
Group
Others
```

Examples:

```text
4755
SUID + 755

2755
SGID + 755

1777
Sticky bit + 777
```

At beginner level, learn to recognize these before trying to use them extensively.

---

# 79. ACL Introduction

Traditional Linux permissions provide three classes:

```text
Owner
Group
Others
```

Sometimes that is not flexible enough.

Imagine:

```text
File owner:
alice

File group:
developers
```

But you want:

```text
bob
```

to have read access without:

* Changing the file owner
* Adding Bob to the developers group
* Giving all others read access

Access Control Lists, or ACLs, can provide more granular permissions.

---

# 80. What Is an ACL?

ACL means:

```text
Access Control List
```

ACLs can allow permissions for additional named users or groups beyond the traditional owner, group, and others model.

Conceptually:

```text
report.txt
|
+-- Owner: alice
|
+-- Group: finance
|
+-- Basic permissions
|
+-- ACL:
    bob -> read
```

Now Bob can receive specific access without becoming the owner or changing broad permissions.

---

# 81. Viewing ACLs With `getfacl`

If ACL tools are installed and supported, you can inspect ACLs with:

```bash
getfacl report.txt
```

Possible output may resemble:

```text
# file: report.txt
# owner: alice
# group: developers
user::rw-
user:bob:r--
group::r--
mask::r--
other::---
```

Notice:

```text
user:bob:r--
```

Bob has a named ACL entry.

---

# 82. Setting an ACL With `setfacl`

An administrator or authorized owner might use:

```bash
setfacl -m u:bob:r report.txt
```

Conceptually:

```text
-m
Modify ACL

u:bob:r
Give user bob read permission

report.txt
Target file
```

Verify:

```bash
getfacl report.txt
```

ACLs are more advanced than basic permissions, but you should know they exist because `ls -l` alone may not always explain every access decision.

---

# 83. The `+` in `ls -l`

Sometimes you may see:

```text
-rw-r-----+
```

The trailing:

```text
+
```

can indicate extended ACL information.

If permissions appear correct but access behaves unexpectedly, inspect ACLs:

```bash
getfacl filename
```

Do not assume `chmod` is the whole story.

---

# 84. ACL Mask

ACLs introduce another important concept called the ACL mask.

The mask can limit effective permissions for named users and groups covered by it.

For example, an ACL may appear to give a user:

```text
rwx
```

but the effective permission can be more restrictive because of the mask.

At beginner level, remember:

```text
When ACLs exist, inspect the complete getfacl output.

Do not interpret only the basic ls -l permission string.
```

You can study advanced ACL design later.

---

# 85. `Permission denied`

One of the most common Linux errors is:

```text
Permission denied
```

This is a symptom.

It is not the root cause.

Possible causes include:

* Wrong file owner
* Wrong group owner
* Missing group membership
* Missing read permission
* Missing write permission
* Missing execute permission
* Missing directory traversal permission
* ACL restrictions
* Security mechanisms such as SELinux
* Filesystem mounted read-only
* Application running under a different user
* SSH refusing an insecure private key
* Wrong assumptions about the target path

Do not solve every `Permission denied` with `sudo`.

---

# 86. Permission Troubleshooting Method

Use this reasoning process:

```text
Observe
   |
   v
What operation failed?
   |
   v
Which user/process performed it?
   |
   v
What resource was accessed?
   |
   v
Inspect owner and group
   |
   v
Inspect permission bits
   |
   v
Inspect path directories
   |
   v
Inspect groups
   |
   v
Check ACL/special controls if necessary
   |
   v
Form hypothesis
   |
   v
Apply smallest correct fix
   |
   v
Retry
   |
   v
Verify
```

---

# 87. Step 1: Confirm Identity

Start:

```bash
whoami
```

Then:

```bash
id
```

If troubleshooting another account:

```bash
id deployuser
```

Ask:

```text
Which UID is actually performing the operation?

Which groups does it belong to?
```

Do not assume.

---

# 88. Step 2: Inspect the Resource

Run:

```bash
ls -l filename
```

For a directory:

```bash
ls -ld directory
```

Why `-d`?

Without it:

```bash
ls -l directory
```

usually lists the directory's contents.

With:

```bash
ls -ld directory
```

you inspect the directory itself.

This distinction is important during permission troubleshooting.

---

# 89. Step 3: Inspect Parent Directories

Suppose the file is:

```text
/srv/company/private/config.yaml
```

Inspect:

```bash
ls -ld /srv
```

```bash
ls -ld /srv/company
```

```bash
ls -ld /srv/company/private
```

```bash
ls -l /srv/company/private/config.yaml
```

A parent directory may be blocking traversal.

If available:

```bash
namei -l /srv/company/private/config.yaml
```

can make this process easier.

---

# 90. Step 4: Compare Identity With Ownership

Suppose:

```bash
id deployuser
```

shows:

```text
groups=...developers...
```

and:

```bash
ls -l app.conf
```

shows:

```text
-rw-r----- 1 alice developers ... app.conf
```

Then ask:

```text
Is deployuser the owner?

No.

Is deployuser in developers?

Yes.

Which permission triplet applies?

Group:
r--
```

Therefore deployuser can read through the group class but cannot write through the basic group permissions.

This is permission reasoning.

---

# 91. Ownership-Related Failure Example

Suppose an application runs as:

```text
appuser
```

and needs to write to:

```text
/var/lib/myapp
```

Inspect:

```bash
ls -ld /var/lib/myapp
```

Output:

```text
drwxr-xr-x 2 root root ... /var/lib/myapp
```

`appuser` is neither owner nor a member receiving write permission through the owning group.

A poor response is:

```bash
sudo chmod 777 /var/lib/myapp
```

A better investigation asks:

```text
Should appuser own this directory?

Should a dedicated application group own it?

Does only one subdirectory need to be writable?

What does the application's documented ownership model require?
```

Then apply the smallest correct ownership and permission change.

---

# 92. Executable Permission Problems

Create a script:

```bash
printf '#!/bin/bash\necho "Deployment check passed"\n' > deploy-check.sh
```

Inspect:

```bash
ls -l deploy-check.sh
```

You may see:

```text
-rw-r--r--
```

Try:

```bash
./deploy-check.sh
```

You may receive:

```text
Permission denied
```

Why?

The file does not have execute permission.

---

# 93. Fixing the Executable Permission

Add execute permission for the owner:

```bash
chmod u+x deploy-check.sh
```

Verify:

```bash
ls -l deploy-check.sh
```

You should see something like:

```text
-rwxr--r--
```

Now:

```bash
./deploy-check.sh
```

Expected:

```text
Deployment check passed
```

You observed:

```text
Symptom
Permission denied

Evidence
No x bit

Hypothesis
Direct execution is blocked

Fix
chmod u+x

Verification
Script executes
```

That is the troubleshooting process you should learn.

---

# 94. Execute Permission Is Not the Only Script Requirement

If a script still fails after adding `x`, investigate further.

Possible problems include:

* Invalid shebang
* Interpreter missing
* Windows-style line endings
* Script located on a filesystem mounted with `noexec`
* Directory traversal restrictions
* Script syntax errors
* Interpreter permissions

Permission troubleshooting should be evidence-based.

Do not assume every execution failure means missing `x`.

---

# 95. SSH Private-Key Permission Example

SSH private keys are a practical example of permissions being used for security.

Suppose your key is:

```text
~/Downloads/server-key.pem
```

Check:

```bash
ls -l ~/Downloads/server-key.pem
```

If permissions are too open, SSH may reject the key.

You might see a warning resembling:

```text
WARNING: UNPROTECTED PRIVATE KEY FILE!
```

and SSH may ignore the key.

Why?

A private key is supposed to be private.

---

# 96. Fixing SSH Private-Key Permissions

A common restrictive mode is:

```bash
chmod 400 ~/Downloads/server-key.pem
```

This means:

```text
Owner:
read

Group:
none

Others:
none
```

Symbolically:

```text
r--------
```

Another common private-key mode is:

```bash
chmod 600 ~/Downloads/server-key.pem
```

which gives the owner:

```text
read + write
```

and no permissions to group or others.

Which is appropriate depends on your use case and SSH implementation requirements, but the key principle is:

```text
Private keys must not be broadly accessible.
```

---

# 97. Why `chmod 400` Helps

Break down:

```text
400
```

Owner:

```text
4 = r--
```

Group:

```text
0 = ---
```

Others:

```text
0 = ---
```

Result:

```text
r--------
```

The private key remains readable by its owner but is not writable or accessible through the group/others classes.

---

# 98. SSH Permission Troubleshooting Workflow

If SSH reports private-key permission problems:

First:

```bash
ls -l ~/Downloads/server-key.pem
```

Then determine:

```text
Who owns the key?

Can group members read it?

Can others read it?

Can others write it?
```

Apply an appropriately restrictive mode, for example:

```bash
chmod 400 ~/Downloads/server-key.pem
```

Verify:

```bash
ls -l ~/Downloads/server-key.pem
```

Then retry SSH.

Do not run:

```bash
chmod 777 server-key.pem
```

That moves in exactly the wrong security direction.

---

# 99. Permission Denied While Copying to a Server

Suppose you use:

```bash
scp file.txt ubuntu@server:/protected/location/
```

and receive:

```text
Permission denied
```

Do not assume the SSH key is wrong.

If authentication succeeded and the error occurred while writing the destination, investigate the remote directory.

On the server:

```bash
whoami
```

```bash
id
```

```bash
ls -ld /protected/location
```

Perhaps your SSH account cannot write there.

A common safer workflow is:

```text
Local machine
     |
     v
scp to user's home directory
     |
     v
SSH into server
     |
     v
Use authorized sudo operation
to move file if necessary
```

This is often preferable to weakening the destination permissions.

---

# 100. Least-Privilege Permissions

Least privilege applies to files just as it applies to users.

The question is:

```text
What is the minimum access required?
```

For example:

A private SSH key may need:

```text
Owner:
read

Everyone else:
nothing
```

An application configuration might need:

```text
Owner:
read/write

Application group:
read

Others:
nothing
```

A shared project directory might need:

```text
Owner:
full

Team group:
appropriate collaboration permissions

Others:
nothing
```

The correct permission is determined by requirements.

Not convenience.

---

# 101. Avoid Permission Numbers Without Context

Do not learn:

```text
600 = good
644 = good
755 = good
777 = bad
```

as absolute rules.

Instead learn:

```text
Who needs access?

What type of resource is this?

What actions must they perform?

What should everyone else be prevented from doing?
```

For example:

`600` makes sense for many private files.

But `600` on a directory is usually problematic because there is no execute/traversal permission.

Likewise, `755` may be sensible for a public directory but inappropriate for a private secrets directory.

Context determines correctness.

---

# 102. Permission Design Example

Suppose you have:

```text
/srv/veriqta-app/
```

Requirements:

```text
deployuser:
Needs full management access

developers group:
Needs read and traversal access

everyone else:
Should have no access
```

A possible design could involve:

```text
Owner:
deployuser

Group:
developers

Directory mode:
750
```

Conceptually:

```text
/srv/veriqta-app
|
+-- deployuser
|   rwx
|
+-- developers
|   r-x
|
+-- others
    ---
```

Whether this exact design is correct depends on what developers and the application need to do.

---

# 103. Permission Design Is About Users, Resources, and Actions

A useful model is:

```text
WHO
|
+-- alice
+-- developers
+-- nginx
+-- deployuser

needs to do WHAT
|
+-- read
+-- modify
+-- execute
+-- traverse
+-- create/delete

to WHICH RESOURCE
|
+-- configuration
+-- logs
+-- scripts
+-- application directory
+-- secrets
```

Then configure the minimum access necessary.

This is much stronger than memorizing random `chmod` commands.

---

# 104. Intentional Permission Failure and Repair Lab

Now you will intentionally create permission failures and troubleshoot them.

Use a disposable Linux VM or lab machine.

Do not perform destructive permission experiments on:

```text
/etc
/usr
/bin
/sbin
/var
/home
/
```

or production application directories.

Create a dedicated lab.

---

# 105. Lab Step 1: Create the Workspace

Run:

```bash
mkdir -p ~/permission-lab/{configs,scripts,logs,private}
```

Move into it:

```bash
cd ~/permission-lab
```

Check:

```bash
pwd
```

You should be somewhere similar to:

```text
/home/yourusername/permission-lab
```

List:

```bash
ls -l
```

---

# 106. Lab Step 2: Create Test Files

Run:

```bash
touch configs/app.conf
touch logs/application.log
touch private/credentials.txt
```

Create a script:

```bash
printf '#!/bin/bash\necho "VERIQTA permission lab works"\n' > scripts/check.sh
```

Inspect:

```bash
ls -l configs
ls -l logs
ls -l private
ls -l scripts
```

---

# 107. Lab Step 3: Inspect Default Permissions

Run:

```bash
umask
```

Write down the result.

Then:

```bash
ls -ld configs scripts logs private
```

and:

```bash
ls -l configs/app.conf logs/application.log private/credentials.txt scripts/check.sh
```

Ask:

```text
Why do directories have execute permission while normal files usually do not?
```

Because directory execute means traversal, while ordinary files do not automatically need direct execution.

---

# 108. Lab Step 4: Protect a Private File

Inspect:

```bash
ls -l private/credentials.txt
```

Set:

```bash
chmod 600 private/credentials.txt
```

Verify:

```bash
ls -l private/credentials.txt
```

Expected permission string:

```text
-rw-------
```

Explain it:

```text
Owner:
read + write

Group:
none

Others:
none
```

---

# 109. Lab Step 5: Make the Private Directory Private

Run:

```bash
chmod 700 private
```

Verify:

```bash
ls -ld private
```

Expected:

```text
drwx------
```

Now both the directory and file are restricted through basic permissions.

---

# 110. Lab Step 6: Intentionally Break Script Execution

Inspect:

```bash
ls -l scripts/check.sh
```

Make sure the owner does not have execute permission:

```bash
chmod 644 scripts/check.sh
```

Try:

```bash
./scripts/check.sh
```

You should likely receive:

```text
Permission denied
```

Do not fix it yet.

Diagnose it.

---

# 111. Lab Step 7: Diagnose the Script

Run:

```bash
whoami
```

Then:

```bash
id
```

Then:

```bash
ls -l scripts/check.sh
```

You should see:

```text
-rw-r--r--
```

Ask:

```text
Am I the owner?

Yes, normally.

What owner permissions exist?

rw-

Is execute present?

No.
```

Hypothesis:

```text
Direct execution fails because owner execute permission is missing.
```

---

# 112. Lab Step 8: Repair the Script

Run:

```bash
chmod u+x scripts/check.sh
```

Verify:

```bash
ls -l scripts/check.sh
```

You should see:

```text
-rwxr--r--
```

Now:

```bash
./scripts/check.sh
```

Expected:

```text
VERIQTA permission lab works
```

You repaired the actual cause.

You did not use:

```bash
chmod 777 scripts/check.sh
```

---

# 113. Lab Step 9: Break Directory Traversal

Create:

```bash
mkdir restricted
```

Then:

```bash
echo "production configuration" > restricted/app.txt
```

Inspect:

```bash
ls -ld restricted
ls -l restricted/app.txt
```

Now remove execute permission from the directory:

```bash
chmod a-x restricted
```

Inspect:

```bash
ls -ld restricted
```

Depending on your remaining permissions, listing names may still behave differently from accessing file metadata or contents.

Try:

```bash
cat restricted/app.txt
```

You should encounter an access problem because traversal is missing.

---

# 114. Lab Step 10: Understand the Failure

Inspect:

```bash
ls -ld restricted
```

Suppose it resembles:

```text
drw-r--r--
```

The directory may have read permission but no execute permission.

Remember:

```text
Directory read:
Read/list directory entries

Directory execute:
Traverse/search entries
```

Knowing that:

```text
app.txt
```

exists does not mean you can traverse the directory to access it.

---

# 115. Lab Step 11: Repair Directory Traversal

Restore an appropriate owner mode:

```bash
chmod 700 restricted
```

Verify:

```bash
ls -ld restricted
```

Then:

```bash
cat restricted/app.txt
```

It should work for the owner.

Again:

```text
Observe
Diagnose
Repair
Verify
```

---

# 116. Lab Step 12: Practice Numeric Modes

Create:

```bash
touch mode-test.txt
```

Apply:

```bash
chmod 600 mode-test.txt
```

Inspect:

```bash
ls -l mode-test.txt
```

Predict before continuing:

```text
rw-------
```

Now:

```bash
chmod 640 mode-test.txt
```

Expected:

```text
rw-r-----
```

Then:

```bash
chmod 644 mode-test.txt
```

Expected:

```text
rw-r--r--
```

Do not just look at the output.

Translate each number mentally.

---

# 117. Lab Step 13: Practice Symbolic Modes

Starting with:

```text
rw-r--r--
```

run:

```bash
chmod g+w mode-test.txt
```

Predict:

```text
rw-rw-r--
```

Verify:

```bash
ls -l mode-test.txt
```

Now:

```bash
chmod o-r mode-test.txt
```

Predict:

```text
rw-rw----
```

Verify.

Now:

```bash
chmod u-w mode-test.txt
```

Predict:

```text
r--rw----
```

Verify.

Then restore:

```bash
chmod 640 mode-test.txt
```

---

# 118. Lab Step 14: Inspect Ownership

Run:

```bash
ls -l mode-test.txt
```

Identify:

```text
User owner
Group owner
```

Then:

```bash
id
```

Compare your current username and groups with the file's ownership.

This connects Part 4 directly to Part 5.

---

# 119. Lab Step 15: Create a Shared Group

If you have sudo access on this disposable lab:

```bash
sudo groupadd permissionlab
```

If the group already exists, do not recreate it.

Check:

```bash
getent group permissionlab
```

Add your account to it:

```bash
sudo usermod -aG permissionlab "$(whoami)"
```

Remember that your current session may not immediately receive the new supplementary group.

A fresh login may be required.

---

# 120. Lab Step 16: Change Group Ownership

Once your environment has the necessary membership or administrative privilege, create:

```bash
mkdir shared
```

Change its group:

```bash
sudo chgrp permissionlab shared
```

Inspect:

```bash
ls -ld shared
```

You should see:

```text
permissionlab
```

as the group owner.

---

# 121. Lab Step 17: Configure Shared Directory Permissions

Set:

```bash
chmod 770 shared
```

Inspect:

```bash
ls -ld shared
```

Expected permission pattern:

```text
drwxrwx---
```

Meaning:

```text
Owner:
rwx

Group:
rwx

Others:
---
```

This creates a basic team-style access model.

---

# 122. Lab Step 18: Add SGID to the Shared Directory

Run:

```bash
chmod g+s shared
```

Inspect:

```bash
ls -ld shared
```

You may see:

```text
drwxrws---
```

Now files created in this directory can inherit the directory's group according to SGID directory behavior.

Create:

```bash
touch shared/team-file.txt
```

Inspect:

```bash
ls -l shared/team-file.txt
```

Look at the group owner.

---

# 123. Lab Step 19: Observe `umask`

Run:

```bash
umask
```

Save the current value mentally or in your lab notes.

Set temporarily:

```bash
umask 077
```

Create:

```bash
touch private-default.txt
mkdir private-default-dir
```

Inspect:

```bash
ls -ld private-default.txt private-default-dir
```

You should typically see:

```text
File:
600

Directory:
700
```

Now restore your previous `umask`.

For example, if it was `022`:

```bash
umask 022
```

Do not assume `022` was your original value. Restore the value you actually recorded.

---

# 124. Lab Step 20: Explore `/tmp`

Inspect:

```bash
ls -ld /tmp
```

Look for something resembling:

```text
drwxrwxrwt
```

Identify:

```text
d
Directory

rwx
Owner permissions

rwx
Group permissions

rwt
Others permissions + sticky bit
```

Do not change `/tmp`.

This step is observation only.

---

# 125. Lab Step 21: Inspect a SUID Program

Run:

```bash
ls -l /usr/bin/passwd
```

On many systems you will see SUID indicated by:

```text
s
```

in the owner execute position.

Do not modify it.

This is observation only.

Changing special permissions on system executables can damage security or functionality.

---

# 126. Lab Step 22: ACL Practice

First determine whether the tools exist:

```bash
which getfacl
which setfacl
```

If available, create:

```bash
touch acl-test.txt
```

Inspect:

```bash
getfacl acl-test.txt
```

If you have another safe lab user such as `alice`, you can grant read permission:

```bash
setfacl -m u:alice:r acl-test.txt
```

Inspect:

```bash
getfacl acl-test.txt
```

Then:

```bash
ls -l acl-test.txt
```

Look for the ACL indicator.

If the commands are unavailable, do not install packages merely for this exercise unless you understand your environment. Knowing ACLs exist is enough at this stage.

---

# 127. Lab Step 23: Create an Intentional Ownership Failure

This step requires a disposable lab account and sudo access.

Suppose `alice` exists.

Create:

```bash
mkdir ownership-test
```

Create:

```bash
touch ownership-test/config.txt
```

Change ownership:

```bash
sudo chown alice:alice ownership-test/config.txt
```

Set:

```bash
sudo chmod 600 ownership-test/config.txt
```

Inspect:

```bash
ls -l ownership-test/config.txt
```

If you are not Alice, try:

```bash
cat ownership-test/config.txt
```

You should receive:

```text
Permission denied
```

provided no other privilege or ACL changes the result.

---

# 128. Lab Step 24: Diagnose the Ownership Failure

Do not use `sudo cat` yet.

Run:

```bash
whoami
```

Then:

```bash
id
```

Then:

```bash
ls -l ownership-test/config.txt
```

Reason:

```text
Owner:
alice

Mode:
600

Owner:
rw-

Group:
---

Others:
---

Current user:
not alice
```

Therefore your normal account has no basic read access.

That explains the failure.

---

# 129. Lab Step 25: Choose a Correct Repair

There is no universal repair.

Ask what the requirement is.

If the file should remain Alice's private file:

```text
Do nothing.

The denial is correct.
```

If your team should read it:

Possible design:

```text
Assign an appropriate group.
Give that group read permission.
```

If you should own it:

Ownership may need changing.

The critical lesson is:

```text
A permission failure is not automatically a misconfiguration.

Sometimes denial is exactly the intended security behavior.
```

---

# 130. Lab Step 26: Clean Up

Return:

```bash
cd ~
```

Confirm:

```bash
pwd
```

If the lab contains nothing important:

```bash
rm -r ~/permission-lab
```

Before pressing Enter, inspect the path carefully.

You are intentionally deleting:

```text
~/permission-lab
```

not:

```text
~/
```

and definitely not:

```text
/
```

If you created a temporary group and no longer need it:

```bash
sudo groupdel permissionlab
```

Only do this if the group is genuinely disposable and not used elsewhere.

---

# 131. Troubleshooting Challenge 1: The Script Will Not Run

You see:

```text
-rw-r--r-- 1 deploy deploy 300 deploy.sh
```

Then:

```bash
./deploy.sh
```

returns:

```text
Permission denied
```

What is your first hypothesis?

The owner does not have execute permission.

Possible minimal fix:

```bash
chmod u+x deploy.sh
```

Then verify:

```bash
ls -l deploy.sh
```

and retry.

---

# 132. Troubleshooting Challenge 2: User Cannot Read a File

File:

```text
-rw-r----- 1 alice developers 200 app.conf
```

User:

```text
bob
```

Check:

```bash
id bob
```

Suppose Bob is not in:

```text
developers
```

Then Bob receives the:

```text
others
```

class:

```text
---
```

He cannot read the file through basic permissions.

Possible solutions depend on requirements:

* Add Bob to the appropriate group
* Use a different group
* Use an ACL
* Change the permission design

Do not automatically grant read access to everyone.

---

# 133. Troubleshooting Challenge 3: Group Membership Exists but Access Fails

Suppose:

```bash
id bob
```

shows:

```text
developers
```

The file is:

```text
-rw-r----- alice developers app.conf
```

Yet:

```bash
cat /srv/private/app.conf
```

fails.

What next?

Inspect the path:

```bash
ls -ld /srv
ls -ld /srv/private
ls -l /srv/private/app.conf
```

Perhaps Bob cannot traverse:

```text
/srv/private
```

File permissions alone do not determine path access.

---

# 134. Troubleshooting Challenge 4: Application Cannot Write Logs

Application runs as:

```text
appuser
```

Directory:

```text
drwxr-xr-x root root /var/log/myapp
```

The application reports:

```text
Permission denied
```

Do not run:

```bash
chmod 777 /var/log/myapp
```

Investigate:

```bash
id appuser
```

```bash
ls -ld /var/log/myapp
```

Then determine the intended ownership model.

Perhaps:

```text
Owner:
appuser
```

or:

```text
Group:
application group
```

should receive write access.

Apply the smallest appropriate change.

---

# 135. Troubleshooting Challenge 5: Recursive `644` Broke an Application

Someone runs:

```bash
chmod -R 644 /srv/application
```

Now the application cannot access directories properly.

Why?

Directories received:

```text
rw-r--r--
```

There is no execute/traversal permission.

The repair should not be:

```bash
chmod -R 777 /srv/application
```

Instead, determine the intended permissions for:

* Directories
* Regular files
* Executable scripts
* Secrets
* Writable application directories

Then repair each category appropriately.

---

# 136. Troubleshooting Challenge 6: SSH Rejects a Private Key

You run:

```bash
ssh -i server-key.pem ubuntu@server
```

SSH warns that the private key permissions are too open.

Check:

```bash
ls -l server-key.pem
```

Suppose:

```text
-rw-r--r--
```

This means group and others can read the key.

A restrictive repair may be:

```bash
chmod 400 server-key.pem
```

Verify:

```bash
ls -l server-key.pem
```

Then retry.

The lesson is not merely:

```text
SSH likes 400.
```

The real lesson is:

```text
Private authentication material must be protected from other users.
```

---

# 137. Troubleshooting Challenge 7: `chmod` Fails

You run:

```bash
chmod 600 application.conf
```

and receive:

```text
Operation not permitted
```

Investigate:

```bash
whoami
```

```bash
ls -l application.conf
```

Questions:

```text
Do I own the file?

Am I authorized to change its mode?

Is there another filesystem or security restriction?
```

Do not automatically prepend `sudo`.

Understand why the operation is restricted.

---

# 138. Troubleshooting Challenge 8: File Looks Readable but Is Not

You see:

```text
-rw-r--r-- app.conf
```

and assume everyone can read it.

But another user gets:

```text
Permission denied
```

Possible explanation:

A parent directory lacks execute/traversal permission.

Investigate the entire path.

This is one of the strongest habits you can develop:

```text
Never inspect only the final file.
```

---

# 139. Troubleshooting Challenge 9: `ls -l` Looks Correct but Access Is Strange

Suppose the mode appears correct, but a specific user still has unexpected access.

Look for:

```text
+
```

after the permission string.

Then inspect:

```bash
getfacl filename
```

ACLs may provide additional access rules.

If ACLs do not explain it, other security mechanisms may be involved.

---

# 140. Troubleshooting Challenge 10: Root Can Access It, Application Cannot

You test:

```bash
sudo cat /srv/app/config
```

and it works.

Then conclude:

```text
The file is fine.
```

That conclusion is wrong.

Your application may run as:

```text
appuser
```

Test the actual identity when appropriate:

```bash
sudo -u appuser cat /srv/app/config
```

If that fails, you have reproduced the problem much more accurately.

Production troubleshooting should imitate the failing identity.

---

# 141. Permission Troubleshooting Command Set

You should become comfortable with:

```bash
whoami
id
groups
ls -l
ls -ld
namei -l
getfacl
```

For modification:

```bash
chmod
chown
chgrp
setfacl
```

For finding affected resources:

```bash
find
```

The diagnostic commands should usually come before modification commands.

---

# 142. A Better Permission Troubleshooting Habit

Avoid:

```text
Permission denied
      |
      v
sudo
      |
      v
Still broken?
      |
      v
chmod 777
```

Use:

```text
Permission denied
      |
      v
What operation failed?
      |
      v
Who performed it?
      |
      v
whoami / id
      |
      v
Who owns the resource?
      |
      v
ls -l / ls -ld
      |
      v
Which permission class applies?
      |
      v
Check parent directories
      |
      v
Check groups
      |
      v
Check ACLs if needed
      |
      v
Identify actual cause
      |
      v
Apply minimum required change
      |
      v
Retest as failing identity
```

That is the mindset you want.

---

# 143. Real-World DevOps Scenario

Imagine:

```text
GitHub Actions
      |
      v
Deployment runner
      |
      v
SSH
      |
      v
Linux Server
      |
      v
deployuser
      |
      v
/srv/application
```

Deployment fails:

```text
mkdir: cannot create directory '/srv/application/releases/104': Permission denied
```

A weak troubleshooting approach:

```bash
sudo chmod -R 777 /srv/application
```

A better approach:

### Step 1: Identify the deployment user

```bash
whoami
```

or determine the account used by the automation.

### Step 2: Inspect identity

```bash
id deployuser
```

### Step 3: Inspect the destination

```bash
ls -ld /srv/application
```

### Step 4: Inspect releases

```bash
ls -ld /srv/application/releases
```

### Step 5: Determine required operation

The deployment needs to create:

```text
/srv/application/releases/104
```

Therefore it needs appropriate write and traversal access to the relevant parent directory.

### Step 6: Determine intended access model

Perhaps:

```text
Owner:
deployuser

Group:
deployers

Mode:
appropriate team-restricted permissions
```

### Step 7: Apply the smallest change

Do not broaden access beyond the requirement.

### Step 8: Test as the actual account

For example, when authorized:

```bash
sudo -u deployuser mkdir /srv/application/releases/permission-test
```

### Step 9: Verify

```bash
ls -ld /srv/application/releases/permission-test
```

Then remove the test directory if appropriate.

This is real permission troubleshooting.

---

# 144. Another Real-World Scenario: Web Application Uploads

Architecture:

```text
Users
  |
  v
Web Application
  |
  v
nginx / application process
  |
  v
/srv/app/uploads
```

Users report that file uploads fail.

Application log:

```text
Permission denied: /srv/app/uploads/image.jpg
```

You need to ask:

```text
Which Linux user runs the application?
```

Check the process or service configuration.

Suppose:

```text
appuser
```

Then inspect:

```bash
id appuser
```

and:

```bash
ls -ld /srv/app/uploads
```

Suppose:

```text
drwxr-xr-x root root /srv/app/uploads
```

`appuser` receives no write permission.

Now you have evidence.

The solution should reflect the application's security model rather than simply making the directory world-writable.

---

# 145. Security: Why Ownership Can Be Better Than Broad Permissions

Imagine an application needs exclusive write access.

Option A:

```text
root:root
777
```

Everyone can write.

Option B:

```text
appuser:appgroup
750 or another requirement-based mode
```

Only intended identities receive access.

The exact mode depends on the workload, but Option B represents the better design principle:

```text
Correct identity
+
Minimum necessary permissions
```

rather than:

```text
Wrong identity
+
Excessive permissions
```

---

# 146. Security: Secrets

Files containing:

* API keys
* Private SSH keys
* Database credentials
* Tokens
* Certificates with private keys
* Application secrets

should not receive broad read permissions.

For example:

```text
-rw-r--r--
```

may expose a secret to users who should not read it.

A more restrictive mode such as:

```text
600
```

may be appropriate when only one owner needs access.

For service environments, ownership and group requirements may differ.

Always design around the actual consumer.

---

# 147. Security: Executable Files

Do not add execute permission to every file.

A configuration file:

```text
app.yaml
```

usually does not need:

```text
x
```

A log:

```text
application.log
```

usually does not need:

```text
x
```

A private key:

```text
id_ed25519
```

does not need:

```text
x
```

A script intended for direct execution may need it.

Permissions should describe actual intended use.

---

# 148. Security: Recursive `chown`

This can be just as dangerous as recursive `chmod`:

```bash
sudo chown -R user:group /path
```

If `/path` is wrong, you can alter ownership across critical resources.

Before running recursive ownership changes:

```bash
pwd
```

```bash
ls -ld /path
```

```bash
find /path -maxdepth 2 -ls
```

Check exactly what will be affected.

A single incorrect `/` can completely change the scope of a command.

---

# 149. What You Should Now Understand

Before continuing to Part 6, you should understand:

* Why Linux permissions exist
* Authentication vs authorization
* How users and groups relate to filesystem access
* How to read `ls -l`
* What the first file-type character means
* What owner, group, and others mean
* What `r`, `w`, and `x` mean
* Why permissions behave differently on files and directories
* Why directory execute means traversal
* Why deleting a file depends heavily on directory permissions
* What symbolic permissions are
* What numeric permissions are
* Why `4`, `2`, and `1` represent read, write, and execute
* How to convert permission numbers to `rwx`
* What `777` means
* Why `chmod 777` is usually a poor troubleshooting response
* What `755`, `750`, `700`, `644`, `640`, and `600` mean
* How `chmod` changes permissions
* How symbolic `chmod` works
* How numeric `chmod` works
* What `chown` does
* What `chgrp` does
* The difference between user group membership and file group ownership
* Why parent-directory permissions matter
* What recursive permission changes do
* Why `chmod -R` can be dangerous
* Why files and directories often need different permissions
* What default permissions are
* What `umask` does
* How common masks such as `022`, `002`, and `077` affect creation
* Why `umask` is a mask rather than simply a final permission value
* What SUID is
* What SGID is
* Why SGID directories help collaborative groups
* What the sticky bit does
* Why `/tmp` commonly uses the sticky bit
* What ACLs are
* Why `getfacl` and `setfacl` exist
* Why a `+` may appear after an `ls -l` permission string
* How ownership problems cause permission failures
* How missing execute permission affects scripts
* Why SSH private keys require restrictive permissions
* Why `Permission denied` is a symptom rather than a diagnosis
* How to troubleshoot permissions systematically
* Why least privilege should determine access

---

# 150. Commands You Should Now Recognize

Inspection:

```bash
ls -l
ls -ld
whoami
id
groups
umask
namei -l
getfacl
```

Permission management:

```bash
chmod
```

Ownership management:

```bash
chown
chgrp
```

ACL management:

```bash
getfacl
setfacl
```

Useful supporting commands:

```bash
find
sudo
sudo -u
```

You should understand the purpose of these before moving forward.

---

# 151. Beginner Knowledge Check

Answer these without looking back.

1. Why does Linux need filesystem permissions?

2. What is the difference between authentication and authorization?

3. What does `ls -l` show?

4. What does the first character in `-rw-r--r--` represent?

5. What does `d` mean?

6. What does `l` mean?

7. Which permission triplet belongs to the owner?

8. Which belongs to the group?

9. Which belongs to others?

10. What does `r` mean for a regular file?

11. What does `w` mean for a regular file?

12. What does `x` mean for a regular file?

13. What does `r` mean for a directory?

14. What does `w` mean for a directory?

15. What does `x` mean for a directory?

16. Why can a readable file still be inaccessible?

17. Why does deleting a file depend on directory permissions?

18. What does `rwx` mean?

19. What numeric value represents read?

20. What numeric value represents write?

21. What numeric value represents execute?

22. What does `7` represent?

23. What does `6` represent?

24. What does `5` represent?

25. What does `0` represent?

26. What does `777` mean?

27. Why is `777` dangerous as a generic fix?

28. What does `755` mean?

29. What does `750` mean?

30. What does `700` mean?

31. What does `644` mean?

32. What does `640` mean?

33. What does `600` mean?

34. What does `chmod` do?

35. What does `chmod u+x script.sh` do?

36. What does `chmod g-w file` do?

37. What does `chmod 600 file` do?

38. What does `chown` do?

39. What does `chgrp` do?

40. How can `chown` change owner and group simultaneously?

41. What does recursive `-R` mean?

42. Why is recursive `chmod` dangerous?

43. Why can `chmod -R 644 directory` break directory access?

44. What is `umask`?

45. What common file mode can result from a `022` umask?

46. What common directory mode can result from `022`?

47. What does `077` commonly produce for files and directories?

48. What is SUID?

49. What is SGID?

50. Why is SGID useful on shared directories?

51. What is the sticky bit?

52. Why is the sticky bit commonly used on `/tmp`?

53. What is an ACL?

54. What does `getfacl` do?

55. What does `setfacl` do?

56. What might a `+` after permissions in `ls -l` indicate?

57. Why might SSH reject a private key with broad permissions?

58. What does `chmod 400 private-key.pem` mean?

59. Why should you check `whoami` and `id` when troubleshooting permissions?

60. Why should you inspect parent directories?

61. Why can root successfully access something while an application still fails?

62. What does least privilege mean for file permissions?

63. What should you investigate before running `chmod 777`?

64. Is every `Permission denied` error actually a misconfiguration?

65. Why should you test a fix using the same identity that experienced the failure?

---

# 152. Knowledge Check Answers

1. To control which identities can read, modify, execute, traverse, create, or otherwise access filesystem resources.

2. Authentication establishes identity. Authorization determines what that identity may do.

3. Long-format information including type, permissions, ownership, size, timestamps, and name.

4. Resource type.

5. Directory.

6. Symbolic link.

7. Characters 2 through 4.

8. Characters 5 through 7.

9. Characters 8 through 10.

10. Permission to read file contents.

11. Permission to modify file contents.

12. Permission to execute the file directly when other execution requirements are satisfied.

13. Permission related to reading/listing directory entries.

14. Permission to modify directory entries, normally in combination with traversal permission.

15. Permission to traverse/search the directory.

16. A parent directory may block traversal.

17. Removing a filename modifies the containing directory.

18. Read, write, execute.

19. `4`.

20. `2`.

21. `1`.

22.

```text
rwx
```

23.

```text
rw-
```

24.

```text
r-x
```

25.

```text
---
```

26.

```text
rwxrwxrwx
```

27. It grants extremely broad access and often hides the actual ownership or permission problem.

28.

```text
rwxr-xr-x
```

29.

```text
rwxr-x---
```

30.

```text
rwx------
```

31.

```text
rw-r--r--
```

32.

```text
rw-r-----
```

33.

```text
rw-------
```

34. Changes file or directory permission mode bits.

35. Adds execute permission for the owner.

36. Removes group write permission.

37. Gives the owner read/write permission and removes basic permissions from group and others.

38. Changes ownership.

39. Changes group ownership.

40.

```bash
chown user:group file
```

41. Apply the operation through a directory tree.

42. It can modify large numbers of resources, including files that require different permissions.

43. Directories lose execute/traversal permission.

44. A creation mask that removes selected permission bits from requested permissions when resources are created.

45. Commonly `644`.

46. Commonly `755`.

47. Commonly `600` for files and `700` for directories.

48. Set User ID, a special mode that can affect the effective user identity when certain executables run.

49. Set Group ID.

50. New resources can inherit the directory's group ownership, helping maintain shared group access.

51. A special directory permission that restricts deletion and renaming behavior in shared writable directories.

52. `/tmp` is commonly writable by many users, so the sticky bit helps prevent users from arbitrarily deleting one another's files.

53. Access Control List, a mechanism for more granular access beyond basic owner, group, and others permissions.

54. Displays ACL information.

55. Modifies ACL entries.

56. Extended ACL information may exist.

57. Private authentication material should not be accessible to unauthorized users, and SSH clients enforce restrictions on private-key exposure.

58. Owner read only, group none, others none.

59. Because access decisions depend on the identity actually performing the operation.

60. Directory traversal permissions along the path can block access even when the final file's mode appears permissive.

61. Root has privileges the application's user does not. Testing as root does not reproduce the application's authorization context.

62. Grant only the access required for the resource's intended use.

63. Identity, owner, group, group membership, file mode, directory modes, ACLs, and the actual access requirement.

64. No. The denial may be the intended security behavior.

65. Because successful testing as a different or more privileged identity does not prove that the original user's access problem has been resolved.

---

# 153. Final Hands-On Challenge

Complete this in a disposable Linux environment without following the previous lab line by line.

Create:

```text
~/linux-access-project/
|
+-- public/
|
+-- configs/
|
+-- scripts/
|
+-- secrets/
|
+-- logs/
|
+-- shared/
```

Your requirements are:

1. `public` should be accessible for normal reading and traversal.

2. `configs` should allow the owner full directory access while others receive only the access you intentionally choose.

3. `scripts` should contain a Bash script called `deploy.sh`.

4. `deploy.sh` should be executable by the owner.

5. `secrets` should be accessible only to its owner through basic permissions.

6. Create:

```text
secrets/api-key.txt
```

7. Protect `api-key.txt` so only the owner can read and write it.

8. Create:

```text
logs/application.log
```

9. Give the log an intentional owner/group/read model and explain why.

10. Create a group for shared project work if you have administrative access.

11. Assign `shared` to that group.

12. Configure `shared` so intended group members can collaborate.

13. Enable SGID on `shared`.

14. Create a file inside it.

15. Verify the file's group ownership.

16. Check your current `umask`.

17. Explain what it means.

18. Temporarily use `umask 077`.

19. Create a file and directory.

20. Predict their permissions before checking.

21. Verify them.

22. Restore your original `umask`.

23. Intentionally remove execute permission from `deploy.sh`.

24. Attempt to execute it.

25. Diagnose the error using `ls -l`.

26. Repair only the missing permission.

27. Execute the script successfully.

28. Intentionally remove traversal permission from a test directory.

29. Demonstrate the resulting access failure.

30. Repair it.

31. Inspect `/tmp`.

32. Identify its sticky bit.

33. Inspect `/usr/bin/passwd`.

34. Identify SUID if your distribution uses it there.

35. Create a file and inspect its ACL if ACL tools are available.

36. Explain the difference between `chmod`, `chown`, and `chgrp`.

37. Explain the difference between file permissions and directory permissions.

38. Explain why `chmod 777` would be inappropriate for `secrets`.

39. Explain why `chmod -R 644` would break a directory tree.

40. Explain how you would investigate a production `Permission denied` error before changing anything.

---

# 154. The Mental Model to Keep

When Linux returns:

```text
Permission denied
```

think:

```text
Who?
 |
 v
Which user/process is performing the operation?
 |
 v
What?
 |
 v
Read, write, execute, traverse, create, delete?
 |
 v
Where?
 |
 v
Which file or directory?
 |
 v
Ownership
 |
 +-- User owner
 +-- Group owner
 |
 v
Basic permissions
 |
 +-- Owner
 +-- Group
 +-- Others
 |
 v
Path permissions
 |
 v
ACLs / special controls if relevant
 |
 v
What access is actually required?
 |
 v
Apply smallest correct change
 |
 v
Test as the original identity
 |
 v
Verify
```

Do not build this mental model:

```text
Permission denied
      |
      v
sudo
      |
      v
chmod 777
```

The first model teaches you Linux.

The second hides the problem.


---


| Command                                                                          | What it does                                                                                    | How to use it                                                                                                                                      |
| -------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------- |
| `ls -l`                                                                          | Shows a long listing with file type, permissions, owner, group, size, time, and name.           | Run `ls -l` to inspect files in the current directory.                                                                                             |
| `ls -l app.conf`                                                                 | Shows detailed information about one file, including its permissions and ownership.             | Put the filename after `ls -l`. Example: `ls -l app.conf`                                                                                          |
| `ls -ld directory`                                                               | Shows the permissions and ownership of the directory itself instead of listing its contents.    | Use `ls -ld` followed by the directory. Example: `ls -ld /srv/private`                                                                             |
| `ls -la`                                                                         | Shows a detailed listing including hidden entries.                                              | Run `ls -la` before broad permission changes when you need to inspect what is present.                                                             |
| `cat report.txt`                                                                 | Reads and displays a file's contents.                                                           | Run `cat` followed by the filename. You need appropriate file read and directory traversal permissions.                                            |
| `cat restricted/app.txt`                                                         | Attempts to read a file through a directory path.                                               | Useful in the lab for demonstrating that missing directory traversal permission can prevent access.                                                |
| `less report.txt`                                                                | Opens a file for interactive reading.                                                           | Run `less report.txt` when you have read access to the file and can traverse its path.                                                             |
| `head report.txt`                                                                | Reads the beginning of a file.                                                                  | Run `head` followed by the filename.                                                                                                               |
| `tail report.txt`                                                                | Reads the end of a file.                                                                        | Run `tail` followed by the filename.                                                                                                               |
| `./deploy.sh`                                                                    | Attempts to execute `deploy.sh` directly.                                                       | Run it from the directory containing the script. The file needs appropriate execute permission and other execution requirements must be satisfied. |
| `chmod MODE FILE`                                                                | Changes permission mode bits on a file or directory.                                            | Replace `MODE` with numeric or symbolic permissions and `FILE` with the target.                                                                    |
| `chmod 600 private.txt`                                                          | Gives the owner read/write access and gives group and others no basic permissions.              | Use when this exact restrictive permission design is required.                                                                                     |
| `chmod 600 report.txt`                                                           | Sets `report.txt` to `rw-------`.                                                               | Run `chmod 600 report.txt`, then verify with `ls -l report.txt`.                                                                                   |
| `chmod 755 deploy.sh`                                                            | Gives the owner `rwx`, and group and others `r-x`.                                              | Use when the script genuinely needs this access model.                                                                                             |
| `chmod 750 deploy.sh`                                                            | Gives the owner `rwx`, group `r-x`, and others no permissions.                                  | Use when the group should read/execute but others should have no access.                                                                           |
| `chmod 700 deploy.sh`                                                            | Gives only the owner `rwx`.                                                                     | Use when only the owner should read, modify, and execute the script.                                                                               |
| `chmod u+x deploy.sh`                                                            | Adds execute permission for the file owner without replacing the other permission bits.         | `u` means owner, `+` means add, and `x` means execute.                                                                                             |
| `chmod g-w report.txt`                                                           | Removes write permission from the group.                                                        | `g` means group, `-` means remove, and `w` means write.                                                                                            |
| `chmod go+r report.txt`                                                          | Adds read permission for both group and others.                                                 | `g` means group, `o` means others, `+r` means add read.                                                                                            |
| `chmod u=rw,g=r,o= report.txt`                                                   | Sets exact symbolic permissions: owner `rw-`, group `r--`, others `---`.                        | Use `=` when you want to set the specified class exactly. This is equivalent to basic mode `640`.                                                  |
| `chmod 640 app.conf`                                                             | Sets owner `rw-`, group `r--`, others `---`.                                                    | Use when that exact access model is required.                                                                                                      |
| `chmod g+w shared.txt`                                                           | Adds group write permission while preserving other permissions.                                 | Useful when you only need to modify the group's write bit.                                                                                         |
| `chmod -R 755 project/`                                                          | Recursively applies mode `755` through the directory tree.                                      | `-R` means recursive. The note warns that files and directories often should not all receive the same mode.                                        |
| `chmod -R 777 application/`                                                      | Recursively gives owner, group, and others `rwx`.                                               | Shown in the note as a dangerous example. Do not use it as a generic permission fix.                                                               |
| `chmod -R 644 project/`                                                          | Recursively sets files and directories to `644`.                                                | Shown as a common mistake because directories lose execute/traversal permission.                                                                   |
| `chmod u+s executable`                                                           | Sets SUID symbolically on an executable.                                                        | `u+s` adds the Set User ID special mode. The note says not to apply SUID randomly.                                                                 |
| `chmod g+s shared`                                                               | Enables SGID on a directory.                                                                    | Useful for a shared directory when newly created resources should inherit the directory's group ownership.                                         |
| `chmod +t shared-directory`                                                      | Adds the sticky bit to a directory.                                                             | Used for shared writable directory designs when sticky-bit behavior is required.                                                                   |
| `chmod 400 ~/Downloads/server-key.pem`                                           | Gives only the owner read permission to the private key.                                        | Common restrictive private-key example from the note. Verify afterward with `ls -l`.                                                               |
| `chmod 600 ~/Downloads/server-key.pem`                                           | Gives the owner read/write access and no basic access to group or others.                       | Another restrictive private-key mode shown in the note.                                                                                            |
| `chmod 700 private`                                                              | Gives the owner full directory access and group/others none.                                    | Used in the lab to make the `private` directory accessible only through the owner class.                                                           |
| `chmod 644 scripts/check.sh`                                                     | Sets the script to owner read/write and everyone else read only. It removes execute permission. | Used deliberately in the lab to cause a direct-execution failure.                                                                                  |
| `chmod u+x scripts/check.sh`                                                     | Restores only the missing owner execute permission.                                             | Used after diagnosing why `./scripts/check.sh` returns `Permission denied`.                                                                        |
| `chmod a-x restricted`                                                           | Removes execute permission from owner, group, and others.                                       | `a` means all classes. Used in the lab to demonstrate loss of directory traversal.                                                                 |
| `chmod 700 restricted`                                                           | Restores owner read/write/traversal access to the test directory.                               | Used to repair the intentional directory traversal failure.                                                                                        |
| `chmod 600 mode-test.txt`                                                        | Sets the test file to `rw-------`.                                                              | Used to practice numeric modes.                                                                                                                    |
| `chmod 640 mode-test.txt`                                                        | Sets the test file to `rw-r-----`.                                                              | Used to practice numeric modes and restore the lab file.                                                                                           |
| `chmod 644 mode-test.txt`                                                        | Sets the test file to `rw-r--r--`.                                                              | Used to practice numeric permission conversion.                                                                                                    |
| `chmod g+w mode-test.txt`                                                        | Adds group write permission.                                                                    | Changes `rw-r--r--` to `rw-rw-r--` in the lab example.                                                                                             |
| `chmod o-r mode-test.txt`                                                        | Removes read permission from others.                                                            | Used to practice symbolic permission changes.                                                                                                      |
| `chmod u-w mode-test.txt`                                                        | Removes owner write permission.                                                                 | Used to practice symbolic permission changes.                                                                                                      |
| `chmod 770 shared`                                                               | Gives owner and group `rwx`, and others no permissions.                                         | Used to create the basic shared team-directory access model.                                                                                       |
| `chown NEWOWNER FILE`                                                            | Changes the user owner of a file.                                                               | Replace `NEWOWNER` with the new owner and `FILE` with the target.                                                                                  |
| `sudo chown alice report.txt`                                                    | Changes `report.txt` so Alice becomes its owner.                                                | Use administrative privilege when required, then verify with `ls -l report.txt`.                                                                   |
| `sudo chown alice:developers report.txt`                                         | Changes both the user owner and group owner.                                                    | The part before `:` is the user owner. The part after `:` is the group owner.                                                                      |
| `sudo chown :developers report.txt`                                              | Changes only the file's group using `chown`, leaving its user owner unchanged.                  | Leave the username empty before `:` and specify the group.                                                                                         |
| `sudo chown -R alice:developers /srv/project`                                    | Recursively changes user and group ownership throughout a directory tree.                       | Use `-R` for recursive changes. The note warns to inspect the target carefully first.                                                              |
| `sudo chown alice:alice ownership-test/config.txt`                               | Changes the test file's owner and group to Alice.                                               | Used in the lab to create an intentional ownership-related permission failure.                                                                     |
| `chgrp developers report.txt`                                                    | Changes the group owner of a file to `developers`.                                              | Run when your identity and group memberships allow the change.                                                                                     |
| `sudo chgrp developers report.txt`                                               | Changes the file's group owner with administrative privileges.                                  | Use when elevated privileges are required.                                                                                                         |
| `sudo chgrp permissionlab shared`                                                | Changes the `shared` directory's group owner to `permissionlab`.                                | Used in the shared-directory lab.                                                                                                                  |
| `sudo usermod -aG developers alice`                                              | Adds Alice to the `developers` supplementary group.                                             | This changes Alice's group membership, not the group ownership of a file.                                                                          |
| `whoami`                                                                         | Shows the identity of the current user.                                                         | Run before troubleshooting so you know which user is performing the operation.                                                                     |
| `id`                                                                             | Shows the current user's UID, GID, and group memberships.                                       | Run `id` when diagnosing which permission classes may apply.                                                                                       |
| `id deployuser`                                                                  | Shows UID, GID, and groups for `deployuser`.                                                    | Put the username after `id` when troubleshooting another account.                                                                                  |
| `id appuser`                                                                     | Shows the identity and groups of the application account.                                       | Useful when an application reports `Permission denied`.                                                                                            |
| `id bob`                                                                         | Shows Bob's identity and group memberships.                                                     | Used to determine whether Bob belongs to a file's owning group.                                                                                    |
| `groups`                                                                         | Shows the groups associated with the current account.                                           | Run `groups` when checking group membership during permission troubleshooting.                                                                     |
| `umask`                                                                          | Displays the current file-creation mode mask.                                                   | Run `umask` before creating test resources or when investigating default permissions.                                                              |
| `umask 077`                                                                      | Sets a restrictive creation mask for the current shell.                                         | Subsequent typical files may become `600` and directories `700`. Restore the original mask afterward.                                              |
| `umask 022`                                                                      | Sets mask `022` for the current shell.                                                          | The note uses it as an example of restoring a previously recorded `022` mask. Do not assume this was your original value.                          |
| `touch report.txt`                                                               | Creates an empty file if it does not already exist.                                             | The resulting mode is affected by the requested creation mode and `umask`.                                                                         |
| `touch umask-file.txt`                                                           | Creates a file for observing the effect of `umask`.                                             | Create it, then inspect it with `ls -ld` or `ls -l`.                                                                                               |
| `mkdir umask-directory`                                                          | Creates a directory for observing `umask`.                                                      | Create it and inspect its resulting permissions.                                                                                                   |
| `touch private-file.txt`                                                         | Creates a test file after changing the `umask`.                                                 | Used with `umask 077` to observe restrictive default permissions.                                                                                  |
| `mkdir private-directory`                                                        | Creates a test directory after changing the `umask`.                                            | Used to observe how `umask 077` affects a directory.                                                                                               |
| `ls -l /usr/bin/passwd`                                                          | Inspects the permissions and ownership of the `passwd` executable.                              | Used to observe SUID on systems where `/usr/bin/passwd` uses it. Do not modify it.                                                                 |
| `ls -ld /tmp`                                                                    | Shows the permissions on `/tmp` itself.                                                         | Used to identify the sticky bit, commonly shown by `t` in `drwxrwxrwt`. Do not change `/tmp`.                                                      |
| `namei -l /srv/private/config.txt`                                               | Displays path components and their permission information when `namei` is available.            | Useful for finding which directory in a path is blocking traversal.                                                                                |
| `namei -l /srv/company/private/config.yaml`                                      | Inspects the components leading to the configuration file.                                      | Use it when the final file's permissions appear correct but path access fails.                                                                     |
| `find /path -maxdepth 2 -ls`                                                     | Inspects resources under a path to a maximum depth of two levels.                               | Useful before dangerous recursive permission or ownership changes.                                                                                 |
| `find project -type d -exec chmod 755 {} \;`                                     | Finds directories under `project` and applies `755` to each one.                                | Used when directories intentionally need a different mode from regular files. The note says not to copy it blindly.                                |
| `find project -type f -exec chmod 644 {} \;`                                     | Finds regular files under `project` and applies `644` to each.                                  | Used to target files separately from directories.                                                                                                  |
| `getfacl report.txt`                                                             | Displays the file's ACL information.                                                            | Use when ACL tools are available, especially when basic `ls -l` permissions do not fully explain access.                                           |
| `getfacl filename`                                                               | Displays ACL information for a specified file.                                                  | Replace `filename` with the resource being investigated.                                                                                           |
| `setfacl -m u:bob:r report.txt`                                                  | Adds or modifies an ACL entry giving Bob read permission on `report.txt`.                       | `-m` modifies the ACL, `u:bob:r` gives user Bob read access.                                                                                       |
| `which getfacl`                                                                  | Checks whether the `getfacl` command is available.                                              | Run before the ACL lab.                                                                                                                            |
| `which setfacl`                                                                  | Checks whether the `setfacl` command is available.                                              | Run before attempting to modify ACLs.                                                                                                              |
| `mkdir -p ~/permission-lab/{configs,scripts,logs,private}`                       | Creates the permission lab and its four subdirectories.                                         | `-p` allows creation of the required directory structure.                                                                                          |
| `cd ~/permission-lab`                                                            | Moves into the permission lab directory.                                                        | Run after creating the lab workspace.                                                                                                              |
| `pwd`                                                                            | Shows your current working directory.                                                           | Use it to confirm where you are before performing permission or recursive operations.                                                              |
| `touch configs/app.conf`                                                         | Creates the lab configuration file.                                                             | Run from inside `~/permission-lab`.                                                                                                                |
| `touch logs/application.log`                                                     | Creates the lab log file.                                                                       | Run from inside the permission lab.                                                                                                                |
| `touch private/credentials.txt`                                                  | Creates the lab credentials file.                                                               | Used to practice restrictive permissions.                                                                                                          |
| `printf '#!/bin/bash\necho "VERIQTA permission lab works"\n' > scripts/check.sh` | Creates a small Bash script containing a shebang and an `echo` command.                         | Run from inside the permission lab. The `>` writes the output into `scripts/check.sh`.                                                             |
| `mkdir restricted`                                                               | Creates the `restricted` test directory.                                                        | Used for the intentional directory-traversal failure.                                                                                              |
| `echo "production configuration" > restricted/app.txt`                           | Writes the text `production configuration` into `restricted/app.txt`.                           | Used to create a file inside the directory before removing traversal permission.                                                                   |
| `touch mode-test.txt`                                                            | Creates a test file for practicing numeric and symbolic modes.                                  | Change its permissions with the `chmod` examples in the lab.                                                                                       |
| `sudo groupadd permissionlab`                                                    | Creates the `permissionlab` group.                                                              | Run only in the disposable lab when you have sudo access and the group does not already exist.                                                     |
| `getent group permissionlab`                                                     | Looks up the `permissionlab` group.                                                             | Use it to check whether the group exists and inspect its record.                                                                                   |
| `sudo usermod -aG permissionlab "$(whoami)"`                                     | Adds your current account to the `permissionlab` supplementary group.                           | `$(whoami)` supplies your current username. The current session may not immediately receive the new membership.                                    |
| `mkdir shared`                                                                   | Creates the shared test directory.                                                              | Used for the shared-group and SGID exercise.                                                                                                       |
| `touch shared/team-file.txt`                                                     | Creates a file inside the SGID shared directory.                                                | Inspect it afterward to see which group owns the new file.                                                                                         |
| `touch private-default.txt`                                                      | Creates a file after temporarily setting `umask 077`.                                           | Inspect it to observe the resulting restrictive mode.                                                                                              |
| `mkdir private-default-dir`                                                      | Creates a directory after setting `umask 077`.                                                  | Inspect it to observe the resulting directory mode.                                                                                                |
| `touch acl-test.txt`                                                             | Creates a file for ACL practice.                                                                | Use only as a safe lab file.                                                                                                                       |
| `setfacl -m u:alice:r acl-test.txt`                                              | Gives the lab user Alice read access through an ACL.                                            | Use only if Alice exists and ACL tools are available.                                                                                              |
| `getfacl acl-test.txt`                                                           | Displays the ACL of the ACL test file.                                                          | Run before and after `setfacl` to compare the ACL.                                                                                                 |
| `mkdir ownership-test`                                                           | Creates the ownership troubleshooting directory.                                                | Used in the intentional ownership-failure exercise.                                                                                                |
| `touch ownership-test/config.txt`                                                | Creates the file used for the ownership failure.                                                | Its ownership and mode are later changed deliberately.                                                                                             |
| `cat ownership-test/config.txt`                                                  | Attempts to read the ownership-test file.                                                       | If it belongs to Alice with mode `600` and you are not Alice, the note expects `Permission denied` under the stated conditions.                    |
| `cd ~`                                                                           | Returns to your home directory.                                                                 | Used before cleaning up the permission lab.                                                                                                        |
| `rm -r ~/permission-lab`                                                         | Recursively deletes the disposable permission lab.                                              | Verify the path carefully before pressing Enter.                                                                                                   |
| `sudo groupdel permissionlab`                                                    | Deletes the temporary `permissionlab` group.                                                    | Use only when the group is genuinely disposable and is not needed elsewhere.                                                                       |
| `ssh -i server-key.pem ubuntu@server`                                            | Connects to a server using the specified private key.                                           | `-i` specifies the identity/private-key file. The note uses this when discussing private-key permissions.                                          |
| `scp file.txt ubuntu@server:/protected/location/`                                | Copies `file.txt` to a remote server location.                                                  | If authentication works but writing the destination fails, inspect the remote directory permissions instead of assuming the key is wrong.          |
| `sudo cat /srv/app/config`                                                       | Reads the configuration as root.                                                                | The note uses this to show why successful root access does not prove that the application user has access.                                         |
| `sudo -u appuser cat /srv/app/config`                                            | Attempts to read the configuration as `appuser`.                                                | Useful for reproducing the permission problem using the application's actual identity.                                                             |
| `sudo -u deployuser mkdir /srv/application/releases/permission-test`             | Attempts to create a directory as `deployuser`.                                                 | Used to test whether the deployment account has the required write and traversal permissions.                                                      |
| `mkdir /srv/application/releases/permission-test`                                | Creates the test release directory when the current identity has sufficient access.             | The note specifically demonstrates the `sudo -u deployuser` version when testing as the deployment account.                                        |

## Permission values used in the note

These are not separate commands, but you need them to understand the `chmod` commands above.

| Number | Permission     | Meaning                                                       |
| -----: | -------------- | ------------------------------------------------------------- |
|    `0` | `---`          | No permissions                                                |
|    `1` | `--x`          | Execute only                                                  |
|    `2` | `-w-`          | Write only                                                    |
|    `3` | `-wx`          | Write and execute                                             |
|    `4` | `r--`          | Read only                                                     |
|    `5` | `r-x`          | Read and execute                                              |
|    `6` | `rw-`          | Read and write                                                |
|    `7` | `rwx`          | Read, write, and execute                                      |
|  `777` | `rwxrwxrwx`    | Owner, group, and others all receive read, write, and execute |
|  `755` | `rwxr-xr-x`    | Owner full access, group and others read/execute              |
|  `750` | `rwxr-x---`    | Owner full access, group read/execute, others none            |
|  `700` | `rwx------`    | Owner full access, group and others none                      |
|  `644` | `rw-r--r--`    | Owner read/write, group and others read                       |
|  `640` | `rw-r-----`    | Owner read/write, group read, others none                     |
|  `600` | `rw-------`    | Owner read/write, group and others none                       |
|  `400` | `r--------`    | Owner read only, group and others none                        |
|  `770` | `rwxrwx---`    | Owner and group full access, others none                      |
| `4755` | SUID + `755`   | SUID special bit plus `rwxr-xr-x`                             |
| `2755` | SGID + `755`   | SGID special bit plus `rwxr-xr-x`                             |
| `1777` | Sticky + `777` | Sticky bit plus `rwxrwxrwx`                                   |

## The main commands Part 5 expects you to recognize



| Command    | Main purpose                                                       | Typical use from this note                |
| ---------- | ------------------------------------------------------------------ | ----------------------------------------- |
| `ls -l`    | Inspect files, permissions, and ownership                          | `ls -l filename`                          |
| `ls -ld`   | Inspect a directory itself                                         | `ls -ld directory`                        |
| `whoami`   | Identify the current user                                          | `whoami`                                  |
| `id`       | Inspect UID, GID, and groups                                       | `id` or `id username`                     |
| `groups`   | Inspect group memberships                                          | `groups`                                  |
| `umask`    | Inspect or change the creation mask                                | `umask`, `umask 077`                      |
| `namei -l` | Inspect permissions along a path                                   | `namei -l /path/to/file`                  |
| `chmod`    | Change permission mode bits                                        | `chmod 600 file` or `chmod u+x script.sh` |
| `chown`    | Change ownership                                                   | `sudo chown alice:developers file`        |
| `chgrp`    | Change group ownership                                             | `sudo chgrp developers file`              |
| `getfacl`  | Inspect ACLs                                                       | `getfacl filename`                        |
| `setfacl`  | Modify ACLs                                                        | `setfacl -m u:bob:r file`                 |
| `find`     | Find resources, including before or during controlled bulk changes | `find /path -maxdepth 2 -ls`              |
| `sudo`     | Run an authorized command with elevated privileges                 | `sudo chown ...`                          |
| `sudo -u`  | Run a command as a specified user                                  | `sudo -u appuser cat /srv/app/config`     |

The biggest lesson from this note is that the diagnostic commands should normally come before the modification commands. When you see `Permission denied`, first use commands such as `whoami`, `id`, `ls -l`, `ls -ld`, `namei -l`, and `getfacl` to understand the problem. Only then decide whether `chmod`, `chown`, `chgrp`, or `setfacl` is actually the correct fix.


Part 5 gives you the access-control foundation needed for the next stages of Linux administration, where users, files, processes, services, logs, networking, applications, and automation begin interacting with one another.
