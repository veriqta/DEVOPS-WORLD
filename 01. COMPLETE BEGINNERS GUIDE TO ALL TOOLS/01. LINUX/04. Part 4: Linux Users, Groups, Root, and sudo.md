# Part 4: Linux Users, Groups, Root, and sudo

In the previous parts, you learned how to navigate Linux and manage files and directories.

Now you need to understand a concept that controls almost everything you do on a Linux system:

Identity.

Linux constantly needs to answer questions such as:

* Who is running this command?
* Does this user have permission to read this file?
* Can this user modify this directory?
* Can this user install software?
* Can this user stop a service?
* Is this user allowed to become an administrator?
* Which groups does this user belong to?

These questions are handled through Linux users, groups, permissions, root privileges, and tools such as `sudo`.

This part builds the foundation you will need before studying file ownership and permissions in Part 5.

---

# 1. Linux Is a Multi-User Operating System

Linux was designed as a multi-user operating system.

This means multiple user accounts can exist on the same machine.

For example, a server might contain:

```text
Linux Server
|
+-- ann
|
+-- developer
|
+-- deploy
|
+-- monitoring
|
+-- backup
|
+-- root
```

Each account can have:

* Its own username
* Its own numerical user ID
* Its own home directory
* Its own files
* Its own groups
* Its own permissions
* Its own processes
* Different levels of access

Multiple users can also be logged into the same Linux system simultaneously.

This is normal on servers.

---

# 2. Why Does Linux Need Multiple Users?

Imagine a company running an application server.

The server might need:

```text
Developers
Operations engineers
Monitoring software
Web server processes
Database processes
Backup systems
Administrators
```

Giving all of them unrestricted administrative access would be dangerous.

Instead, Linux separates identities.

For example:

```text
Developer
    |
    +-- application files

Monitoring account
    |
    +-- monitoring resources

Web server account
    |
    +-- web application resources

Administrator
    |
    +-- privileged administration
```

This allows Linux to control who can do what.

---

# 3. Human Users and Service Accounts

Not every Linux user represents a person.

This surprises many beginners.

A Linux system can contain:

## Human users

Accounts used by people.

Examples:

```text
ann
alice
bob
devops
```

## System or service users

Accounts used by applications and operating-system services.

Depending on the installed software, you might encounter accounts associated with:

```text
www-data
nginx
postgres
mysql
```

For example, a web server may run under a dedicated account rather than as `root`.

Why?

Security.

If the web server is compromised, the attacker's access may be limited by the permissions assigned to that service account.

This is an example of least privilege, which you will study later in this part.

---

# 4. What Is a User Account?

A Linux user account is an identity recognized by the operating system.

It can contain information such as:

* Username
* UID
* Primary group
* Home directory
* Login shell
* Password-related information
* Group memberships

When you log in, Linux uses your account to determine your identity and access.

A simplified flow is:

```text
You log in
    |
    v
Linux authenticates you
    |
    v
Linux identifies your account
    |
    v
UID and groups are associated with your session
    |
    v
Commands run using that identity
    |
    v
Linux checks permissions
```

This is why your identity matters even when you are simply reading a file.

---

# 5. Username

A username is the human-readable name associated with an account.

Examples:

```text
ann
developer
deploy
backup
```

You might log into a server using:

```bash
ssh ann@server
```

Here:

```text
ann
```

is the username.

However, Linux does not rely only on the username internally.

It also uses a numeric identifier.

That identifier is the UID.

---

# 6. What Is a UID?

UID means:

```text
User Identifier
```

Each user has a numerical UID.

For example:

```text
Username       UID

root           0
ann            1000
developer      1001
deploy         1002
```

The exact UID values vary between systems.

Linux uses UIDs internally to identify users.

Think:

```text
Human-friendly identity
ann
 |
 v
Numeric identity
1000
```

The username makes administration easier for humans.

The UID is fundamental to how Linux represents ownership and identity internally.

---

# 7. Why Does Linux Use Numeric IDs?

Numbers are convenient for operating-system internals.

Suppose a file belongs to UID:

```text
1001
```

Linux can associate that UID with a username such as:

```text
developer
```

When you run:

```bash
ls -l
```

you normally see the username rather than the numeric UID.

For example:

```text
-rw-r--r-- 1 developer developers 2048 Aug 26 10:30 app.conf
```

Linux can also display numeric ownership.

You will study ownership more deeply in Part 5.

For now remember:

```text
Username = human-readable account name

UID = numerical user identifier
```

---

# 8. Root UID

Linux has a special administrative account called:

```text
root
```

The root account has UID:

```text
0
```

This is extremely important.

Check:

```bash
id root
```

You may see something similar to:

```text
uid=0(root) gid=0(root) groups=0(root)
```

The important part is:

```text
uid=0(root)
```

UID 0 represents the superuser identity.

Root has enormous power over the system.

---

# 9. What Are Groups?

Users are only one side of Linux identity management.

Linux also uses groups.

A group allows multiple users to be associated with a common identity for access-control purposes.

Imagine three engineers:

```text
alice
bob
charlie
```

All three need access to deployment resources.

Instead of managing access individually, an administrator could create:

```text
devops
```

and make the engineers members.

Conceptually:

```text
devops group
|
+-- alice
+-- bob
+-- charlie
```

Resources can then be assigned to the group.

This becomes very important when you learn permissions.

---

# 10. Why Groups Exist

Suppose 20 engineers need access to:

```text
/srv/application
```

Managing every engineer separately would become difficult.

Instead:

```text
Engineers
    |
    v
devops group
    |
    v
Application resources
```

When someone joins the team:

```text
Add user to devops
```

When someone leaves:

```text
Remove user from devops
```

Groups therefore make shared access easier to administer.

---

# 11. What Is a GID?

GID means:

```text
Group Identifier
```

Just as users have UIDs, groups have numerical GIDs.

Example:

```text
Group          GID

root           0
developers     1001
devops         1002
monitoring     1003
```

Again, exact values vary.

The relationship is:

```text
User
|
+-- Username
+-- UID

Group
|
+-- Group name
+-- GID
```

---

# 12. Primary Groups

Every user has a primary group.

Suppose:

```text
User: alice
UID: 1001
Primary group: alice
GID: 1001
```

You can inspect this using:

```bash
id alice
```

Possible output:

```text
uid=1001(alice) gid=1001(alice) groups=1001(alice),1002(devops)
```

The section:

```text
gid=1001(alice)
```

identifies the primary group.

The primary group is especially relevant when files are created.

You will see exactly how this affects file ownership in Part 5.

---

# 13. Supplementary Groups

A user can belong to additional groups beyond the primary group.

These are called supplementary groups.

For example:

```text
alice
|
+-- Primary group: alice
|
+-- Supplementary group: devops
|
+-- Supplementary group: docker
|
+-- Supplementary group: monitoring
```

This allows Alice to receive access associated with several groups.

Check using:

```bash
groups alice
```

You might see:

```text
alice : alice devops docker monitoring
```

---

# 14. Primary vs Supplementary Groups

Remember this distinction:

```text
Primary group
    |
    +-- One primary group associated with the account

Supplementary groups
    |
    +-- Additional group memberships
```

A user normally has one primary group at a time but may belong to multiple supplementary groups.

---

# 15. `whoami`

You already encountered:

```bash
whoami
```

This answers a simple question:

```text
Which user am I currently operating as?
```

Example:

```bash
whoami
```

Output:

```text
ann
```

This command is particularly useful when:

* Switching users
* Working on remote servers
* Troubleshooting permissions
* Using administrative shells
* Working across multiple environments

Never assume which identity you are using.

Verify it.

---

# 16. `id`

`id` provides more information.

Run:

```bash
id
```

Example:

```text
uid=1000(ann) gid=1000(ann) groups=1000(ann),27(sudo)
```

Break this down.

```text
uid=1000(ann)
```

Your UID is `1000`.

Your username is `ann`.

```text
gid=1000(ann)
```

Your primary GID is `1000`.

Your primary group is `ann`.

```text
groups=1000(ann),27(sudo)
```

These are your group memberships.

---

# 17. Inspecting Another User With `id`

You can also specify a username:

```bash
id root
```

or:

```bash
id alice
```

If the account exists, Linux displays information about it.

If it does not exist, you may see:

```text
no such user
```

This makes `id` useful during account troubleshooting.

---

# 18. `groups`

Use:

```bash
groups
```

to display groups associated with your current account.

Or:

```bash
groups alice
```

to inspect another account.

Example:

```text
alice : alice developers devops
```

This is useful when troubleshooting:

```text
Why can't Alice access a resource intended for the devops group?
```

One of the first checks should be:

```bash
groups alice
```

Perhaps Alice was never added to the group.

---

# 19. `who`

`who` displays users currently logged into the system.

Run:

```bash
who
```

Possible output:

```text
ann      pts/0    2026-08-26 10:20
alice    pts/1    2026-08-26 10:31
```

This tells you that multiple login sessions exist.

Depending on the system, output can include:

* Username
* Terminal
* Login time
* Remote host

This is useful on shared servers.

---

# 20. `w`

The:

```bash
w
```

command provides more information about logged-in users and system activity.

Run:

```bash
w
```

You might see fields such as:

```text
USER
TTY
FROM
LOGIN@
IDLE
JCPU
PCPU
WHAT
```

The exact output varies.

The important beginner idea is:

```text
who
Shows logged-in users

w
Shows logged-in users plus activity information
```

---

# 21. Where Does Linux Store User Information?

Linux needs persistent account information.

Three important files are:

```text
/etc/passwd
/etc/shadow
/etc/group
```

Each serves a different purpose.

Do not edit these files manually while learning.

Account-management commands exist specifically to modify account information safely.

---

# 22. `/etc/passwd`

The file:

```text
/etc/passwd
```

contains basic user-account information.

View it:

```bash
cat /etc/passwd
```

You may see many lines.

One might resemble:

```text
ann:x:1000:1000:Ann:/home/ann:/bin/bash
```

Each line represents an account.

Fields are separated by:

```text
:
```

---

# 23. Understanding `/etc/passwd` Fields

Consider:

```text
ann:x:1000:1000:Ann:/home/ann:/bin/bash
```

The general structure is:

```text
username:password-placeholder:UID:GID:comment:home:shell
```

Breaking it down:

```text
ann
```

Username.

```text
x
```

Password placeholder.

The actual password hash is normally stored elsewhere.

```text
1000
```

UID.

```text
1000
```

Primary GID.

```text
Ann
```

Comment or descriptive information, sometimes called the GECOS field.

```text
/home/ann
```

Home directory.

```text
/bin/bash
```

Login shell.

---

# 24. Why Is There an `x` in `/etc/passwd`?

Historically, password information was associated directly with `/etc/passwd`.

Modern Linux systems normally store password hashes in:

```text
/etc/shadow
```

The `x` in `/etc/passwd` indicates that password-related information is handled through the shadow password system.

Why separate them?

Security.

`/etc/passwd` needs to be readable by many processes.

Password hashes should be much more restricted.

---

# 25. `/etc/passwd` Does Not Mean It Contains Plaintext Passwords

The filename confuses beginners.

Modern:

```text
/etc/passwd
```

does not normally contain users' plaintext passwords.

It contains account information.

Password hashes are normally stored in:

```text
/etc/shadow
```

Never design a system that stores user passwords in plaintext.

---

# 26. `/etc/shadow`

The file:

```text
/etc/shadow
```

contains sensitive password and password-aging information.

A normal user may try:

```bash
cat /etc/shadow
```

and receive:

```text
Permission denied
```

That is expected.

Why?

Because password hashes are sensitive.

A privileged user can inspect it when genuinely required.

For example:

```bash
sudo cat /etc/shadow
```

Do not expose or copy real password hashes into public documentation, tickets, chat systems, or repositories.

---

# 27. What Is a Password Hash?

Linux does not normally need to store your plaintext password.

Instead, authentication systems use a cryptographic representation derived from the password.

Simplified:

```text
Password
   |
   v
Password hashing process
   |
   v
Stored password hash
```

During authentication, Linux can verify the supplied password against the stored authentication information.

The important beginner rule is:

```text
Password hashes are sensitive.

Do not treat them as harmless just because they are not plaintext passwords.
```

---

# 28. `/etc/group`

Group information is stored in:

```text
/etc/group
```

View:

```bash
cat /etc/group
```

You may see:

```text
devops:x:1002:alice,bob
```

The basic format is:

```text
group-name:password-placeholder:GID:members
```

Breaking it down:

```text
devops
```

Group name.

```text
x
```

Password-related placeholder.

```text
1002
```

GID.

```text
alice,bob
```

Listed group members.

---

# 29. Finding Specific Accounts

Instead of reading hundreds of lines, you can search.

For example:

```bash
grep '^root:' /etc/passwd
```

You may see:

```text
root:x:0:0:root:/root:/bin/bash
```

You will study `grep` properly later.

For now, understand that this searches for the line beginning with:

```text
root:
```

You can also use:

```bash
getent passwd root
```

On many Linux systems, `getent` is a useful way to query account databases because account information may come from sources beyond local files.

For beginner local systems, `/etc/passwd`, `/etc/shadow`, and `/etc/group` remain essential concepts.

---

# 30. Before Creating Users

User administration changes system configuration.

You normally need administrative privileges.

Before practicing, use:

* A disposable virtual machine
* A Linux lab server
* WSL where appropriate
* A cloud test server you control

Do not create random accounts on production servers.

Also remember:

Different Linux distributions provide slightly different account-management tools and defaults.

This is especially noticeable with:

```text
useradd
adduser
```

---

# 31. Creating Users With `useradd`

A common low-level user-management command is:

```bash
useradd
```

You normally need root privileges.

Example:

```bash
sudo useradd labuser
```

Break it down:

```text
sudo
```

Request authorized execution with elevated privileges.

```text
useradd
```

Create a user account.

```text
labuser
```

Username to create.

Verify:

```bash
id labuser
```

---

# 32. `useradd` Defaults Vary

A critical beginner point:

Running:

```bash
sudo useradd labuser
```

does not behave identically on every distribution.

Depending on configuration, it may:

* Create the account
* Assign a UID
* Assign a primary group
* Not create a home directory automatically
* Set a default shell according to local configuration

This is why blindly copying account commands between distributions can cause confusion.

Inspect the result.

---

# 33. Creating a Home Directory With `useradd -m`

A common option is:

```text
-m
```

which requests creation of the user's home directory.

Example:

```bash
sudo useradd -m labuser
```

Verify the account:

```bash
id labuser
```

Check the home directory:

```bash
ls -ld /home/labuser
```

You may see:

```text
drwxr-x--- ... labuser labuser ... /home/labuser
```

Exact permissions vary.

---

# 34. Choosing a Login Shell

You can specify a shell using:

```text
-s
```

Example:

```bash
sudo useradd -m -s /bin/bash labuser
```

Breakdown:

```text
-m
Create home directory

-s /bin/bash
Set /bin/bash as the login shell

labuser
Username
```

Verify:

```bash
getent passwd labuser
```

You may see:

```text
labuser:x:1001:1001::/home/labuser:/bin/bash
```

---

# 35. Creating Users With `adduser`

Some distributions, particularly Debian and Ubuntu systems, provide:

```bash
adduser
```

Example:

```bash
sudo adduser labuser
```

This is typically a more interactive, user-friendly front end.

It may:

* Create the account
* Create the home directory
* Ask for a password
* Ask for descriptive account information
* Apply distribution defaults

This often makes it convenient for manual administration.

---

# 36. `useradd` vs `adduser`

Beginners frequently confuse these.

A simplified distinction on Debian and Ubuntu systems is:

```text
useradd
Lower-level account creation command

adduser
Higher-level, friendlier account-management utility
```

But there is an important warning:

`adduser` is not universally identical across all Linux distributions.

Some systems may not provide the same `adduser` utility.

Therefore, do not assume every distribution behaves like Ubuntu.

For scripts and cross-distribution administration, always understand the target environment.

---

# 37. Setting Passwords With `passwd`

To set or change a password for another user as an administrator:

```bash
sudo passwd labuser
```

Linux will prompt for the new password.

You will not normally see the password characters appear while typing.

That is intentional.

Example interaction:

```text
New password:
Retype new password:
passwd: password updated successfully
```

Exact wording varies.

---

# 38. Changing Your Own Password

A regular user can usually change their own password with:

```bash
passwd
```

The system may ask for:

* Current password
* New password
* Confirmation

Password policies may reject weak passwords.

Do not use passwords such as:

```text
password
123456
admin
companyname
```

Use strong authentication practices appropriate to the environment.

---

# 39. What Does `usermod` Do?

`usermod` modifies an existing user account.

General structure:

```bash
sudo usermod OPTIONS username
```

It can modify things such as:

* Group membership
* Login name
* Home directory
* Login shell
* Account properties

Because account modifications can affect access, use it carefully.

---

# 40. Changing a User's Shell

Suppose:

```text
labuser
```

needs Bash.

You can use:

```bash
sudo usermod -s /bin/bash labuser
```

Verify:

```bash
getent passwd labuser
```

Check the final field.

It should show:

```text
/bin/bash
```

if the change succeeded.

---

# 41. Renaming a User

The:

```text
-l
```

option can change a login name.

Example:

```bash
sudo usermod -l appuser labuser
```

This changes the login name.

However, renaming an account requires more thought than simply changing its visible username.

Resources may still reference:

* Old home directory paths
* File ownership
* Scripts
* Cron jobs
* Application configuration

Do not casually rename production accounts.

---

# 42. Creating Groups With `groupadd`

Create a group with:

```bash
sudo groupadd developers
```

Verify:

```bash
getent group developers
```

Possible output:

```text
developers:x:1002:
```

You can also inspect:

```bash
grep '^developers:' /etc/group
```

---

# 43. Adding a User to a Group

Suppose:

```text
labuser
```

needs membership in:

```text
developers
```

A common command is:

```bash
sudo usermod -aG developers labuser
```

This command deserves careful explanation.

---

# 44. Understanding `usermod -aG`

Break:

```bash
sudo usermod -aG developers labuser
```

into pieces.

```text
sudo
```

Run with elevated privileges if authorized.

```text
usermod
```

Modify a user.

```text
-a
```

Append.

```text
-G
```

Specify supplementary groups.

```text
developers
```

Group.

```text
labuser
```

User being modified.

Conceptually:

```text
Existing supplementary groups
        +
developers
        |
        v
Updated supplementary groups
```

---

# 45. Why `-aG` Matters

This is a very important beginner safety lesson.

You will often see:

```bash
sudo usermod -aG developers labuser
```

The:

```text
-a
```

means append.

Without it, using `-G` can replace the user's existing supplementary-group list.

For example, imagine:

```text
labuser groups:

docker
monitoring
```

You want to add:

```text
developers
```

Using:

```bash
sudo usermod -aG developers labuser
```

preserves the existing supplementary memberships and adds the new one.

Always understand group-modification options before using them.

---

# 46. Verify Group Membership

After modifying membership:

```bash
id labuser
```

or:

```bash
groups labuser
```

Possible output:

```text
labuser : labuser developers
```

Never assume the change worked.

Verify it.

---

# 47. Why a New Group May Not Work Immediately

Suppose you add your currently logged-in account to a group:

```bash
sudo usermod -aG developers ann
```

Then:

```bash
groups
```

may not immediately reflect the new membership in the existing login session.

Why?

Group membership is associated with the user's session when it is established.

You may need to:

* Log out
* Log back in

before the new membership applies fully.

Depending on the environment, tools such as `newgrp` can start a shell with changed group context, but logging out and back in is often the simplest beginner approach.

This is a very common source of confusion.

---

# 48. Removing a User From a Group

The command differs somewhat by distribution.

On Debian and Ubuntu systems, you may use:

```bash
sudo deluser labuser developers
```

Another approach on systems providing GNU `gpasswd` is:

```bash
sudo gpasswd -d labuser developers
```

Possible output:

```text
Removing user labuser from group developers
```

Verify:

```bash
groups labuser
```

Do not assume every distribution provides `deluser`.

---

# 49. Changing the Primary Group

The:

```text
-g
```

option to `usermod` can change the primary group.

Example:

```bash
sudo usermod -g developers labuser
```

Notice the distinction:

```text
-g
Primary group

-G
Supplementary groups
```

Capitalization matters.

This is a good example of why Linux commands must be read carefully.

---

# 50. Deleting Users With `userdel`

To remove an account:

```bash
sudo userdel labuser
```

But this does not necessarily remove everything associated with the user.

Depending on the system and options, the user's home directory may remain.

Check:

```bash
ls -ld /home/labuser
```

This matters because deleting an account and deleting its data are different operations.

---

# 51. `userdel -r`

To remove an account along with certain associated files such as its home directory and mail spool, you may use:

```bash
sudo userdel -r labuser
```

The:

```text
-r
```

requests removal of associated user files handled by the command.

This is destructive.

Before using it, ask:

* Does the user have important data?
* Does anything need archiving?
* Does the account own files elsewhere?
* Is this a production account?
* Are processes still running as this user?

Do not treat account deletion as simple cleanup.

---

# 52. Deleting a User Does Not Guarantee All Their Files Are Gone

A user may own files outside:

```text
/home/username
```

For example:

```text
/srv/application
/opt/project
/var/lib/application
```

Deleting the account does not automatically mean every file owned by its UID disappears.

This can create orphaned ownership represented numerically.

Later, you may encounter files showing an owner such as:

```text
1002
```

instead of a username because the account associated with UID 1002 no longer exists.

This is one reason account deletion should be planned.

---

# 53. Deleting Groups

A group can commonly be deleted using:

```bash
sudo groupdel developers
```

Before deleting a group, consider:

* Is it someone's primary group?
* Do files use this group ownership?
* Do services depend on it?
* Are access-control processes built around it?

Identity objects may have dependencies.

Do not remove them blindly.

---

# 54. The Root Account

The most powerful standard account on Linux is:

```text
root
```

Root has UID:

```text
0
```

Root can perform privileged administrative actions that normal users cannot.

Examples may include:

* Managing users
* Installing system packages
* Modifying protected configuration
* Changing ownership
* Managing services
* Modifying networking
* Reading many protected files
* Shutting down the system

Root access is necessary for many administration tasks.

Permanent root usage is not.

---

# 55. Root's Home Directory

A regular user's home may look like:

```text
/home/ann
```

Root's home is normally:

```text
/root
```

Do not confuse:

```text
/
```

with:

```text
/root
```

They are different.

```text
/
```

is the root of the filesystem tree.

```text
/root
```

is normally the root user's home directory.

This distinction is extremely important.

---

# 56. `$` vs `#`

Shell prompts often use:

```text
$
```

for regular users and:

```text
#
```

for root.

Example:

```text
ann@server:~$
```

Regular user.

Example:

```text
root@server:~#
```

Root.

This is a convention, not a security guarantee.

Do not rely only on the prompt.

Verify:

```bash
whoami
```

and:

```bash
id
```

---

# 57. What Is `su`?

`su` is used to run a shell or command with another user identity.

Historically, it is often described as:

```text
substitute user
```

or associated with switching users.

A common use is:

```bash
su - username
```

For example:

```bash
su - labuser
```

This attempts to start a login-like shell as:

```text
labuser
```

Depending on configuration, you may need that user's password.

---

# 58. `su` vs `su -`

You may see:

```bash
su username
```

and:

```bash
su - username
```

They are not exactly the same.

The:

```text
-
```

requests a login-shell style environment.

This more closely resembles logging in directly as that user.

Environment variables, working directory, and startup files can differ.

For beginners, when intentionally switching into another user's normal login environment, you will often see:

```bash
su - username
```

---

# 59. Switching to Root With `su`

On systems where the root account has an enabled password, this may work:

```bash
su -
```

The system typically asks for the root password.

If successful:

```bash
whoami
```

returns:

```text
root
```

However, many modern distributions, including common Ubuntu configurations, discourage or disable direct root password login by default.

They rely heavily on `sudo` instead.

---

# 60. What Is `sudo`?

`sudo` allows authorized users to execute commands with another user's privileges, most commonly root.

Example:

```bash
sudo useradd -m developer
```

Your normal shell may still belong to:

```text
ann
```

but the specific command receives elevated privileges according to the system's sudo policy.

Conceptually:

```text
Regular user
     |
     v
sudo command
     |
     v
Check sudo policy
     |
     +---- Not allowed ---> Deny
     |
     +---- Allowed
              |
              v
        Run command with
        requested privilege
```

---

# 61. Why `sudo` Is Better Than Working as Root Permanently

Imagine you need to create one user.

Option A:

```text
Become root
Stay root
Create user
Continue doing unrelated work as root
```

Option B:

```text
Remain regular user
Use sudo for one privileged command
Return immediately to normal work
```

Option B reduces the amount of time you operate with maximum privileges.

That matters.

---

# 62. Why Permanent Root Usage Is Dangerous

As root, a simple mistake can affect the entire system.

For example:

* Deleting the wrong directory
* Changing critical permissions
* Modifying system configuration
* Stopping an essential service
* Overwriting a protected file
* Installing unsafe software

A regular user may be blocked from dangerous operations.

Root may not receive the same protection.

Consider:

```text
Regular account
      |
      +-- Limited blast radius

Root
      |
      +-- System-wide blast radius
```

This is why professional administration avoids unnecessary root sessions.

---

# 63. Root Does Not Mean "Use Whenever Something Fails"

A dangerous beginner troubleshooting pattern is:

```text
Command fails
     |
     v
Add sudo
     |
     v
Try again
```

Do not do this automatically.

Suppose:

```bash
rm file.txt
```

returns:

```text
Permission denied
```

The correct response is not immediately:

```bash
sudo rm file.txt
```

First investigate:

```bash
whoami
```

```bash
ls -l file.txt
```

```bash
ls -ld .
```

Ask:

* Who owns the file?
* Who owns the directory?
* Why don't I have permission?
* Should I be deleting this file at all?

Privilege escalation should follow understanding.

---

# 64. How `sudo` Works

At a simplified level:

```text
User runs sudo command
        |
        v
sudo identifies user
        |
        v
sudo reads applicable policy
        |
        v
Is this user authorized?
        |
     +--+--+
     |     |
    No    Yes
     |     |
     v     v
   Deny   Authentication may
          be requested
              |
              v
         Execute command
```

The exact behavior depends on system configuration.

---

# 65. Which Password Does `sudo` Ask For?

On common configurations, `sudo` asks for:

```text
your own password
```

not the root password.

This differs from:

```bash
su -
```

which commonly authenticates using the target account's password.

This is an important distinction.

---

# 66. Why `sudo` May Not Ask Every Time

You may run:

```bash
sudo command1
```

enter your password, then shortly afterward run:

```bash
sudo command2
```

without another password prompt.

This can happen because `sudo` commonly caches successful authentication for a limited period.

That does not mean the second command was unprivileged.

It still used `sudo`.

Authentication was simply not requested again at that moment.

---

# 67. Running a Command as Root

Example:

```bash
sudo whoami
```

Possible output:

```text
root
```

Now compare:

```bash
whoami
```

Possible output:

```text
ann
```

This demonstrates something important.

Your shell remains:

```text
ann
```

but:

```bash
sudo whoami
```

runs that particular command with elevated identity.

---

# 68. Running a Command as Another User

`sudo` can also run commands as users other than root.

A common form is:

```bash
sudo -u username command
```

Example:

```bash
sudo -u labuser whoami
```

Possible output:

```text
labuser
```

This can be useful when testing whether an application account can access something.

It also reinforces:

```text
sudo does not fundamentally mean "root."

sudo means execute according to sudo policy as another identity, commonly root.
```

---

# 69. Opening a Root Shell With `sudo`

You may encounter:

```bash
sudo -i
```

This starts a root login-style shell when permitted.

After:

```bash
sudo -i
```

check:

```bash
whoami
```

You should see:

```text
root
```

Your prompt may end in:

```text
#
```

Exit as soon as the privileged shell is no longer needed:

```bash
exit
```

Then:

```bash
whoami
```

should return your regular account.

Avoid remaining in privileged shells unnecessarily.

---

# 70. `sudo` Configuration Introduction

`sudo` permissions are controlled through sudo policy.

A key configuration file is:

```text
/etc/sudoers
```

Additional configuration may also exist under:

```text
/etc/sudoers.d/
```

Do not casually edit:

```text
/etc/sudoers
```

with an ordinary text editor.

A syntax mistake could interfere with administrative access.

---

# 71. `visudo`

The standard tool for safely editing sudoers configuration is:

```bash
sudo visudo
```

Why use `visudo`?

It provides safeguards such as checking configuration syntax before accepting changes.

You are not going to build complex sudo policies in this beginner part.

The important lesson is:

```text
Do not casually edit /etc/sudoers directly.

Use visudo when sudoers configuration genuinely needs modification.
```

---

# 72. Checking Your `sudo` Permissions

A useful command is:

```bash
sudo -l
```

This lists the sudo privileges available to the current user according to the applicable policy.

Depending on configuration, it may request authentication.

This is useful when troubleshooting:

```text
Am I actually authorized to run this command with sudo?
```

---

# 73. `sudo` Does Not Mean Every User Is an Administrator

A Linux account cannot automatically run every command through `sudo`.

The account must be authorized.

On Ubuntu, administrative users are commonly associated with a group named:

```text
sudo
```

On RHEL-family systems, you will commonly encounter:

```text
wheel
```

But actual authorization ultimately depends on sudo configuration.

Do not assume membership names are identical across every distribution.

---

# 74. Adding a User to the `sudo` Group on Ubuntu

On an Ubuntu-style system, an administrator might use:

```bash
sudo usermod -aG sudo labuser
```

Verify:

```bash
groups labuser
```

You might see:

```text
labuser : labuser sudo
```

The user may need to log out and log back in before a new session receives the membership.

This command grants potentially significant administrative capability.

Do not add users to administrative groups casually.

---

# 75. The `wheel` Group

On RHEL-family distributions such as:

* RHEL
* Rocky Linux
* AlmaLinux
* Fedora

administrative sudo access is commonly associated with:

```text
wheel
```

For example:

```bash
sudo usermod -aG wheel labuser
```

Again, actual authorization depends on configuration.

The larger lesson is:

```text
Linux distributions may implement administrative conventions differently.
```

Always know which distribution you are managing.

---

# 76. What Is Least Privilege?

Least privilege means:

```text
Give an identity only the access it needs to perform its job, and no more.
```

Suppose a deployment account only needs to:

* Read application artifacts
* Write to one deployment directory
* Restart one application service

Giving that account unrestricted root access may be unnecessary.

A better design is:

```text
Deployment account
|
+-- Required application access
|
+-- Required deployment permissions
|
+-- Required service operation
|
X-- Unrestricted system access
```

This reduces risk.

---

# 77. Why Least Privilege Matters

Least privilege helps limit:

* Accidental damage
* Security breaches
* Credential abuse
* Misconfiguration
* Unnecessary administrative access

Imagine two compromised accounts.

Account A can access everything.

Account B can access only:

```text
/srv/application
```

The potential damage from Account B is usually more limited.

This is often called reducing the:

```text
blast radius
```

---

# 78. Least Privilege in DevOps

This concept appears everywhere in modern infrastructure.

Examples include:

```text
Linux
Users and groups

AWS
IAM users and roles

Kubernetes
RBAC

GitHub
Repository permissions

CI/CD
Runner and deployment credentials

Databases
Database roles

Containers
Runtime users
```

The technology changes.

The security principle remains:

```text
Do not give an identity more authority than necessary.
```

---

# 79. A Realistic DevOps Example

Imagine a Linux deployment server.

Accounts include:

```text
ann
deploy
monitoring
backup
root
```

Possible responsibilities:

```text
ann
Human engineer

deploy
Application deployment automation

monitoring
Monitoring agent

backup
Backup process

root
System administration
```

A poor design would be:

```text
Every account = unrestricted root
```

A better design separates responsibilities.

```text
ann
|
+-- Normal account
+-- Controlled sudo privileges

deploy
|
+-- Deployment resources only

monitoring
|
+-- Monitoring resources only

backup
|
+-- Backup resources only

root
|
+-- Reserved for system administration
```

This is how Linux identity management becomes part of production security.

---

# 80. User Administration Troubleshooting Framework

When user access does not work, do not immediately change permissions.

Use a systematic process:

```text
Observe
   |
   v
Confirm identity
   |
   v
Confirm account exists
   |
   v
Check UID and GID
   |
   v
Check group memberships
   |
   v
Check current login session
   |
   v
Check resource ownership
   |
   v
Check permissions
   |
   v
Check sudo policy if relevant
   |
   v
Fix actual cause
   |
   v
Verify
```

Useful commands include:

```bash
whoami
id
groups
getent passwd username
getent group groupname
ls -l
ls -ld
sudo -l
```

---

# 81. Troubleshooting: User Does Not Exist

Suppose:

```bash
id appuser
```

returns something similar to:

```text
id: 'appuser': no such user
```

Investigate:

```bash
getent passwd appuser
```

If nothing is returned, verify spelling.

Remember:

```text
AppUser
```

and:

```text
appuser
```

are not automatically the same username.

Linux is case-sensitive.

---

# 82. Troubleshooting: User Cannot Access a Shared Resource

Scenario:

Alice should have access through:

```text
developers
```

but receives:

```text
Permission denied
```

Start with:

```bash
id alice
```

Then:

```bash
groups alice
```

Is `developers` listed?

If not, the account may not have the expected membership.

If it is listed, investigate the resource itself:

```bash
ls -ld /path/to/resource
```

The problem may involve:

* Ownership
* Permissions
* Parent-directory permissions
* ACLs
* Security systems such as SELinux

Do not assume group membership is the only possible cause.

---

# 83. Troubleshooting: User Was Added to a Group but Access Still Fails

You run:

```bash
sudo usermod -aG developers alice
```

Then Alice immediately retries and still receives:

```text
Permission denied
```

Check:

```bash
id alice
```

from an administrative context.

If the account database shows the new membership but Alice's existing session does not have it, she may need to log out and log back in.

Why?

Her current session may still be using the group list established when she originally logged in.

This is one of the most common beginner group-management issues.

---

# 84. Troubleshooting: `sudo: command not found`

If:

```bash
sudo something
```

returns:

```text
sudo: command not found
```

possible causes include:

* `sudo` is not installed.
* You are using a minimal Linux installation.
* Your command environment cannot locate `sudo`.

Do not assume every Linux environment includes `sudo`.

Some minimal containers, for example, may intentionally omit it.

---

# 85. Troubleshooting: User Is Not Allowed to Use `sudo`

You may see a message indicating that the user is not in the sudoers configuration or is not allowed to execute a command.

Possible cause:

The account has not been granted the necessary sudo privileges.

Investigate:

```bash
sudo -l
```

if the policy allows the query.

Also inspect the user's groups:

```bash
groups
```

Do not solve the issue by editing `/etc/sudoers` randomly.

Administrative access should be granted intentionally.

---

# 86. Troubleshooting: `Authentication failure` With `su`

Suppose:

```bash
su - root
```

returns:

```text
Authentication failure
```

Possible explanations include:

* Incorrect root password
* Root password login is disabled
* Account policy prevents the operation
* Authentication configuration differs from what you expected

On Ubuntu systems, direct root password login may not be the intended administrative workflow.

Use the system's designed privilege-management mechanism rather than trying random passwords.

---

# 87. Troubleshooting: User Has No Home Directory

Suppose:

```bash
getent passwd labuser
```

shows:

```text
/home/labuser
```

but:

```bash
ls -ld /home/labuser
```

returns:

```text
No such file or directory
```

One possible cause is that the account was created without creating its home directory.

This can happen depending on how `useradd` was invoked and system defaults.

Do not simply create random directories without checking ownership requirements.

A home directory needs appropriate ownership and configuration.

When creating new accounts, using the correct creation options from the beginning is usually cleaner.

---

# 88. Troubleshooting: Wrong Login Shell

Suppose:

```bash
getent passwd labuser
```

shows a final field you did not expect.

For example:

```text
/usr/sbin/nologin
```

An account configured with a non-login shell may intentionally be prevented from interactive login.

This is common for service accounts.

Do not automatically change it to:

```text
/bin/bash
```

First determine what the account is for.

A service account may intentionally not have interactive shell access.

---

# 89. Service Accounts Should Not Automatically Have Login Access

Suppose an application needs an account:

```text
webapp
```

The account may need to:

* Own application files
* Run a process

But it may not need:

* Password login
* SSH access
* Interactive shell access
* sudo privileges

Giving service accounts unnecessary login capability increases risk.

Again:

```text
Give only what is required.
```

---

# 90. Troubleshooting: Numeric Owner Appears Instead of Username

Suppose:

```bash
ls -l
```

shows:

```text
-rw-r--r-- 1 1005 1005 2048 Aug 26 10:30 application.log
```

instead of:

```text
-rw-r--r-- 1 alice developers ...
```

One possibility is that Linux has file ownership recorded for UID/GID values that no longer map to known local identities.

This can happen after:

* User deletion
* Group deletion
* Data migration between systems with different UID/GID mappings

This is why UID and GID values matter.

The filename owner you see is ultimately connected to numeric identifiers.

---

# 91. Troubleshooting: Wrong User Is Running a Command

Suppose you expect:

```text
deploy
```

but:

```bash
whoami
```

returns:

```text
root
```

Stop before continuing.

Ask:

* Did I forget to exit a root shell?
* Did I use `sudo -i`?
* Did I switch accounts?
* Am I connected to the server differently than expected?

Use:

```bash
whoami
```

and:

```bash
id
```

before risky operations.

Identity verification is a basic production habit.

---

# 92. Troubleshooting: Group Exists but User Is Not a Member

Check the group:

```bash
getent group developers
```

Then check the user:

```bash
id labuser
```

If the group exists but the user is missing, add the user intentionally:

```bash
sudo usermod -aG developers labuser
```

Verify:

```bash
id labuser
```

Remember that an existing session may need refreshing.

---

# 93. Do Not Manually Edit Account Files While Learning

You now know about:

```text
/etc/passwd
/etc/shadow
/etc/group
```

That does not mean you should begin editing them manually.

Use tools such as:

```text
useradd
adduser
usermod
userdel
groupadd
groupdel
passwd
gpasswd
```

These tools exist to manage account data more safely and consistently.

Manual editing mistakes can cause authentication and account problems.

---

# 94. Security Rules for User Administration

Develop these habits now.

## Rule 1: Work as a regular user by default

Do not remain root unnecessarily.

## Rule 2: Use `sudo` only when needed

Do not add `sudo` simply because a command failed.

## Rule 3: Verify identity

Use:

```bash
whoami
id
```

## Rule 4: Verify group changes

Use:

```bash
groups username
```

or:

```bash
id username
```

## Rule 5: Protect `/etc/shadow`

Never expose its contents unnecessarily.

## Rule 6: Do not casually edit sudoers

Use:

```bash
visudo
```

when configuration changes are genuinely required.

## Rule 7: Apply least privilege

Do not give users administrative access simply because it is convenient.

## Rule 8: Be careful deleting accounts

Understand what happens to home directories and files elsewhere.

## Rule 9: Separate human and service identities

Applications should not automatically run as your personal account or as root.

## Rule 10: Verify every administrative change

Creation, modification, group assignment, and deletion should all be checked.

---

# 95. Hands-On User Administration Lab

Now you will combine the major concepts from this part.

Use a disposable Linux VM or lab server.

Do not perform this exercise on an important production machine.

The goal is to create a small simulated engineering team.

You will create:

```text
Users:

alice
bob
deployuser

Groups:

developers
operations
```

The final identity model will resemble:

```text
Linux Lab Server
|
+-- developers
|   |
|   +-- alice
|   +-- bob
|
+-- operations
    |
    +-- bob
    +-- deployuser
```

---

# 96. Lab Step 1: Confirm Your Current Identity

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
groups
```

Write down:

* Username
* UID
* Primary group
* Supplementary groups

This establishes your starting identity.

---

# 97. Lab Step 2: Check Your Administrative Access

Run:

```bash
sudo -l
```

If your account is authorized, inspect the output.

The goal is to confirm whether your account can perform the administrative operations required by the lab.

If you do not have sudo privileges, do not attempt to bypass the system's access controls.

Use a lab environment where you have legitimate administrative access.

---

# 98. Lab Step 3: Inspect Root

Run:

```bash
id root
```

Look for:

```text
uid=0(root)
```

Then:

```bash
getent passwd root
```

Observe:

* Username
* UID
* GID
* Home directory
* Shell

Notice that root's home is normally:

```text
/root
```

not:

```text
/
```

---

# 99. Lab Step 4: Inspect Account Databases

View:

```bash
less /etc/passwd
```

Search inside `less` for:

```text
/root:
```

Press:

```text
q
```

to exit.

Now:

```bash
less /etc/group
```

Inspect several groups.

Do not modify these files.

---

# 100. Lab Step 5: Create Groups

Create:

```bash
sudo groupadd developers
```

Then:

```bash
sudo groupadd operations
```

Verify:

```bash
getent group developers
```

```bash
getent group operations
```

Both groups should exist.

---

# 101. Lab Step 6: Create Alice

Use:

```bash
sudo useradd -m -s /bin/bash alice
```

Breakdown:

```text
useradd
Create account

-m
Create home directory

-s /bin/bash
Set login shell

alice
Username
```

Verify:

```bash
id alice
```

Then:

```bash
getent passwd alice
```

Check:

```bash
ls -ld /home/alice
```

---

# 102. Lab Step 7: Create Bob

Run:

```bash
sudo useradd -m -s /bin/bash bob
```

Verify:

```bash
id bob
```

```bash
ls -ld /home/bob
```

---

# 103. Lab Step 8: Create the Deployment User

Run:

```bash
sudo useradd -m -s /bin/bash deployuser
```

Verify:

```bash
id deployuser
```

At this point, we are using an interactive-capable account for learning purposes.

In real production environments, a deployment service account may be configured more restrictively depending on its purpose.

---

# 104. Lab Step 9: Set Lab Passwords

If your lab environment supports password authentication and you want to practice `passwd`, run:

```bash
sudo passwd alice
```

Then:

```bash
sudo passwd bob
```

Then:

```bash
sudo passwd deployuser
```

Use passwords appropriate for a disposable lab.

Do not reuse real personal, work, banking, email, or production passwords.

---

# 105. Lab Step 10: Add Alice to Developers

Run:

```bash
sudo usermod -aG developers alice
```

Verify:

```bash
id alice
```

or:

```bash
groups alice
```

You should see membership in:

```text
developers
```

---

# 106. Lab Step 11: Add Bob to Two Groups

Run:

```bash
sudo usermod -aG developers,operations bob
```

Notice:

```text
developers,operations
```

is a comma-separated list.

Verify:

```bash
groups bob
```

Bob should belong to both supplementary groups.

---

# 107. Lab Step 12: Add Deployuser to Operations

Run:

```bash
sudo usermod -aG operations deployuser
```

Verify:

```bash
groups deployuser
```

---

# 108. Lab Step 13: Inspect the Team

Run:

```bash
id alice
```

```bash
id bob
```

```bash
id deployuser
```

Your logical model should resemble:

```text
developers
|
+-- alice
+-- bob

operations
|
+-- bob
+-- deployuser
```

Bob belongs to both.

This demonstrates why supplementary groups are useful.

---

# 109. Lab Step 14: Inspect `/etc/group`

Run:

```bash
getent group developers
```

Then:

```bash
getent group operations
```

Compare the results with:

```bash
groups alice
```

```bash
groups bob
```

```bash
groups deployuser
```

You are connecting the account database to the commands that display user identity.

---

# 110. Lab Step 15: Switch Users

If your lab configuration supports it, try:

```bash
su - alice
```

Then:

```bash
whoami
```

Expected:

```text
alice
```

Run:

```bash
id
```

Then:

```bash
pwd
```

You should normally be in:

```text
/home/alice
```

Exit:

```bash
exit
```

Verify:

```bash
whoami
```

You should be back to your original account.

---

# 111. Lab Step 16: Compare Normal and Elevated Identity

Run:

```bash
whoami
```

Then:

```bash
sudo whoami
```

You may see:

```text
ann
root
```

The lesson is:

```text
Normal command
Runs as your current user

sudo command
Runs with elevated identity according to policy
```

Your entire session does not need to become root just because one command requires privilege.

---

# 112. Lab Step 17: Run a Command as Another User

If permitted:

```bash
sudo -u alice whoami
```

Expected:

```text
alice
```

Try:

```bash
sudo -u deployuser whoami
```

Expected:

```text
deployuser
```

This demonstrates that `sudo` can select identities other than root.

---

# 113. Lab Step 18: Remove Alice From Developers

If your system provides `gpasswd`, run:

```bash
sudo gpasswd -d alice developers
```

Verify:

```bash
groups alice
```

Alice should no longer be listed as a member of `developers`.

Now add her back:

```bash
sudo usermod -aG developers alice
```

Verify again:

```bash
groups alice
```

---

# 114. Lab Step 19: Practice Session Awareness

Switch to Alice:

```bash
su - alice
```

Check:

```bash
groups
```

Exit:

```bash
exit
```

As your administrator account, modify Alice's supplementary groups if needed.

Then understand this key point:

An already-running Alice session may not automatically receive new group membership.

A fresh login session normally obtains the updated group list.

This explains many real Linux access problems.

---

# 115. Lab Step 20: Inspect User Records

Run:

```bash
getent passwd alice
```

Identify:

* Username
* UID
* Primary GID
* Home directory
* Shell

Then:

```bash
id alice
```

Connect the numerical and human-readable information.

Repeat for:

```bash
id bob
```

and:

```bash
id deployuser
```

---

# 116. Lab Step 21: Create a Temporary User

Create:

```bash
sudo useradd -m -s /bin/bash temporaryuser
```

Verify:

```bash
id temporaryuser
```

Check:

```bash
ls -ld /home/temporaryuser
```

Now delete only the account:

```bash
sudo userdel temporaryuser
```

Check:

```bash
id temporaryuser
```

It should no longer exist.

Now inspect:

```bash
ls -ld /home/temporaryuser
```

Depending on the command and distribution defaults, the home directory may still exist.

This demonstrates the difference between deleting an account and deleting associated data.

---

# 117. Lab Step 22: Practice `userdel -r`

Create another disposable account:

```bash
sudo useradd -m -s /bin/bash deletetest
```

Verify:

```bash
ls -ld /home/deletetest
```

Because this account exists only for deletion practice, remove it using:

```bash
sudo userdel -r deletetest
```

Verify:

```bash
id deletetest
```

Then:

```bash
ls -ld /home/deletetest
```

The account and its home directory should normally be gone.

Do not use `userdel -r` casually on real accounts.

---

# 118. Lab Step 23: Final Identity Inspection

Run:

```bash
id alice
```

```bash
id bob
```

```bash
id deployuser
```

Then:

```bash
getent group developers
```

```bash
getent group operations
```

Confirm that the final state matches your intended design.

---

# 119. Lab Step 24: Clean Up

If this is a disposable lab and you want to remove the practice accounts, first inspect them:

```bash
id alice
id bob
id deployuser
```

Then consider whether their home directories contain anything important.

Only if they are purely disposable lab accounts:

```bash
sudo userdel -r alice
```

```bash
sudo userdel -r bob
```

```bash
sudo userdel -r deployuser
```

Then remove the lab groups if they are no longer needed:

```bash
sudo groupdel developers
```

```bash
sudo groupdel operations
```

Verify:

```bash
getent passwd alice
```

```bash
getent group developers
```

No result should be returned if the identities have been successfully removed.

---

# 120. Troubleshooting Challenge 1

A developer says:

> "I was added to the developers group, but I still cannot access the shared directory."

What should you investigate?

Start with:

```bash
id developer
```

Then:

```bash
groups developer
```

Check the current session.

Then inspect the target:

```bash
ls -ld /path/to/shared-directory
```

Possible causes include:

* Membership was not added correctly
* Existing session has old group information
* Directory does not actually grant the group access
* Parent-directory permissions block traversal
* Ownership is wrong
* ACLs or another security mechanism affects access

Do not randomly run `chmod 777`.

You will learn why in Part 5.

---

# 121. Troubleshooting Challenge 2

You run:

```bash
sudo useradd -m appuser
```

and receive:

```text
user 'appuser' already exists
```

Do not delete the user automatically.

Investigate:

```bash
id appuser
```

Then:

```bash
getent passwd appuser
```

Ask:

* Who created this account?
* Is it used by an application?
* Does it own files?
* Is it a service account?

An existing account may be important.

---

# 122. Troubleshooting Challenge 3

You see:

```text
-rw-r----- 1 1007 developers 5000 Aug 26 application.log
```

Why is the owner displayed as:

```text
1007
```

instead of a username?

One possibility is that UID 1007 no longer maps to a known account.

Investigate:

```bash
getent passwd 1007
```

If no account is returned, the file may belong to a UID whose account has been deleted or is unavailable through the current identity database.

---

# 123. Troubleshooting Challenge 4

A command works when you run:

```bash
sudo command
```

but fails without `sudo`.

What does this tell you?

It suggests that the elevated identity has permission or capability that your normal identity does not.

It does not automatically mean:

```text
Always use sudo.
```

Investigate why your normal account lacks access.

Perhaps:

* That restriction is correct
* Ownership is wrong
* Group membership is missing
* The task genuinely requires administration

Understand the permission boundary before changing it.

---

# 124. Troubleshooting Challenge 5

You run:

```bash
sudo -u deployuser command
```

and the application behaves differently than when you run it yourself.

Why?

The process is now running with a different identity.

That can affect:

* File access
* Home directory
* Environment variables
* Group memberships
* Configuration locations
* Credentials
* PATH
* Shell behavior

This is why reproducing problems under the same identity as the application can be important during troubleshooting.

---

# 125. A Production Troubleshooting Example

Imagine an application deployment suddenly fails.

The deployment system reports:

```text
Permission denied
```

A beginner might immediately run:

```text
sudo chmod 777 ...
```

A better engineer investigates identity first.

```text
Deployment fails
      |
      v
Which user runs deployment?
      |
      v
id deployuser
      |
      v
Which groups?
      |
      v
groups deployuser
      |
      v
Who owns destination?
      |
      v
ls -ld /srv/application
      |
      v
What permissions exist?
      |
      v
Determine mismatch
      |
      v
Apply smallest necessary fix
      |
      v
Retest as deployuser
```

This is the reasoning pattern you should develop.

---

# 126. What You Should Now Understand

Before continuing, you should understand:

* Why Linux is a multi-user operating system
* What a user account represents
* The difference between human and service users
* What a username is
* What a UID is
* Why root has UID 0
* What a group is
* What a GID is
* What a primary group is
* What supplementary groups are
* How `whoami` identifies your current user
* How `id` displays UID, GID, and groups
* How `groups` displays group membership
* What `who` displays
* What `w` displays
* What `/etc/passwd` contains
* What the fields in `/etc/passwd` mean
* Why `/etc/passwd` does not normally contain plaintext passwords
* What `/etc/shadow` contains
* Why `/etc/shadow` is protected
* What `/etc/group` contains
* How `useradd` creates users
* Why `useradd` behavior can vary
* How `adduser` differs on systems such as Ubuntu
* How `passwd` sets passwords
* What `usermod` does
* How to create groups with `groupadd`
* How to add users to supplementary groups
* Why `usermod -aG` must be used carefully
* How to remove users from groups
* How `userdel` removes accounts
* What `userdel -r` changes
* Why account deletion does not guarantee all owned files disappear
* What the root account is
* Why `/` and `/root` are different
* What `su` does
* What `sudo` does
* How `su` and `sudo` differ
* Why permanent root usage is risky
* Why adding `sudo` is not a troubleshooting strategy
* What sudo policy does
* Why `visudo` exists
* How `sudo -l` helps inspect authorization
* Why Ubuntu commonly uses `sudo` and RHEL-family systems commonly use `wheel`
* What least privilege means
* Why identity verification matters during troubleshooting

---

# 127. Commands You Should Now Recognize

You should recognize and understand the purpose of:

```bash
whoami
id
groups
who
w
getent
useradd
adduser
passwd
usermod
userdel
groupadd
groupdel
gpasswd
su
sudo
sudo -u
sudo -i
sudo -l
visudo
```

You should also recognize these important files:

```text
/etc/passwd
/etc/shadow
/etc/group
/etc/sudoers
/etc/sudoers.d/
```

Do not memorize commands without understanding their effect.

---

# 128. Beginner Knowledge Check

Answer these before reading the solutions.

1. What does multi-user mean in Linux?

2. Does every Linux user represent a human?

3. What is a username?

4. What is a UID?

5. Which UID is associated with root?

6. What is a Linux group?

7. What is a GID?

8. What is a primary group?

9. What is a supplementary group?

10. Can a user belong to multiple supplementary groups?

11. What does `whoami` tell you?

12. What information does `id` provide?

13. What does `groups` show?

14. What does `who` show?

15. How does `w` differ from `who`?

16. What is stored in `/etc/passwd`?

17. Does `/etc/passwd` normally contain plaintext passwords?

18. Where are password hashes normally stored?

19. Why is `/etc/shadow` restricted?

20. What does `/etc/group` contain?

21. What does `useradd` do?

22. Why might `useradd labuser` not create a home directory on some systems?

23. What does `useradd -m` request?

24. What does `useradd -s /bin/bash` configure?

25. What is `adduser` commonly used for on Ubuntu and Debian?

26. What does `passwd` do?

27. What does `usermod` do?

28. What does `groupadd` do?

29. What does `usermod -aG developers alice` do?

30. Why is the `-a` important when modifying supplementary groups?

31. Why might a newly added group not work in an existing login session?

32. What does `userdel` do?

33. What can `userdel -r` remove in addition to the account?

34. Can files owned by a deleted user remain elsewhere on the filesystem?

35. What is root?

36. What is the difference between `/` and `/root`?

37. What does `su` do?

38. What does `su - username` change compared with a basic user switch?

39. What does `sudo` do?

40. Does `sudo` always mean the entire shell becomes root?

41. Which password does `sudo` commonly request?

42. Why might `sudo` not request a password every time?

43. What does `sudo -u alice command` mean?

44. What does `sudo -i` commonly do?

45. Why is staying logged in as root dangerous?

46. What is `/etc/sudoers`?

47. Why should `visudo` be used?

48. What does `sudo -l` show?

49. What administrative group is commonly used on Ubuntu?

50. What administrative group is commonly used on RHEL-family systems?

51. What is least privilege?

52. Why should service accounts not automatically receive interactive login or sudo access?

53. Why might `ls -l` display a numeric UID instead of a username?

54. What should you investigate before adding `sudo` to a failed command?

55. Why should user and group changes always be verified?

---

# 129. Knowledge Check Answers

1. Multiple user accounts can exist and multiple users can potentially use the same Linux system while maintaining separate identities and access.

2. No. Linux also uses system and service accounts.

3. The human-readable name associated with a Linux account.

4. The numerical user identifier used by Linux.

5. UID `0`.

6. A named identity used to organize users and manage shared access.

7. Group Identifier, the numerical identifier for a group.

8. The user's main group.

9. An additional group to which the user belongs.

10. Yes.

11. Your current effective username in the usual case.

12. UID, primary GID, username, primary group, and group memberships.

13. Group memberships.

14. Logged-in users.

15. `w` provides logged-in user information plus additional session and activity information.

16. Basic account information such as username, UID, primary GID, home directory, and login shell.

17. No.

18.

```text
/etc/shadow
```

19. Because it contains sensitive authentication information such as password hashes.

20. Group names, GIDs, and membership information.

21. It creates a user account.

22. `useradd` follows system-specific defaults, and home-directory creation is not identical across all distributions and configurations.

23. Creation of the user's home directory.

24. The account's login shell.

25. It is commonly a friendlier, higher-level account-management utility.

26. It sets or changes passwords and related password information.

27. It modifies an existing user account.

28. It creates a group.

29. It appends Alice to the `developers` supplementary group.

30. Without append behavior, the specified supplementary-group list can replace existing supplementary memberships.

31. Existing sessions may still have the group list established when the user logged in.

32. It removes a user account.

33. It can also remove certain associated resources, commonly including the home directory and mail spool.

34. Yes.

35. The privileged superuser account associated with UID 0.

36. `/` is the root of the filesystem tree. `/root` is normally root's home directory.

37. It runs a shell or command as another user.

38. The `-` requests a login-style environment for the target user.

39. It allows authorized execution of commands according to sudo policy, commonly with root privileges.

40. No. `sudo command` elevates the specified command. Commands such as `sudo -i` can start a privileged shell.

41. Commonly the invoking user's own password.

42. Successful sudo authentication may be cached temporarily.

43. Run `command` as Alice, if policy allows it.

44. Start a root login-style shell when authorized.

45. Mistakes have a much larger potential impact because root has extensive system privileges.

46. A major configuration file controlling sudo policy.

47. It provides safer sudoers editing and syntax validation.

48. The sudo privileges available to the current user.

49.

```text
sudo
```

50.

```text
wheel
```

51. Giving an identity only the access necessary to perform its job.

52. Because unnecessary capabilities increase security risk and the impact of compromise.

53. The UID may no longer map to a known username, or identity resolution may be unavailable.

54. Your current identity, ownership, permissions, group membership, and whether the operation genuinely requires administrative privileges.

55. Because successful administration requires confirming that the intended identity and access state actually changed.

---

# 130. Final Hands-On Challenge

Perform this only in a disposable Linux lab.

Without following the earlier steps line by line, try to complete this scenario.

Your company has three new team members:

```text
sarah
michael
deploybot
```

You need two groups:

```text
engineering
deployment
```

Requirements:

1. Create `engineering`.

2. Create `deployment`.

3. Create Sarah with a home directory and Bash shell.

4. Create Michael with a home directory and Bash shell.

5. Create `deploybot`.

6. Add Sarah to `engineering`.

7. Add Michael to both `engineering` and `deployment`.

8. Add `deploybot` to `deployment`.

9. Verify Sarah's UID, GID, and groups.

10. Verify Michael's memberships.

11. Verify `deploybot`'s memberships.

12. Inspect their `/etc/passwd` information safely using `getent`.

13. Inspect the two groups.

14. Switch into Sarah's account if your lab authentication configuration permits it.

15. Confirm Sarah's identity with `whoami`.

16. Confirm her groups with `id`.

17. Exit back to your administrator account.

18. Run `whoami`.

19. Run `sudo whoami`.

20. Explain why the outputs differ.

21. Remove Sarah from `engineering`.

22. Verify the change.

23. Add Sarah back.

24. Explain why an already-running Sarah session might not immediately receive the change.

25. Create a disposable account called `removeme`.

26. Confirm that `/home/removeme` exists.

27. Delete `removeme` using `userdel -r`.

28. Verify that the account is gone.

29. Verify what happened to its home directory.

30. Explain why deleting real production users requires more investigation than simply running `userdel -r`.

If you can complete this exercise and explain why every command is being used, you now have a strong beginner foundation in Linux identity management.

# 131. The Main Mental Model to Remember

Linux access begins with identity.

Keep this model in your head:

```text
Person or Process
       |
       v
Linux User
       |
       +-- Username
       |
       +-- UID
       |
       +-- Primary Group
       |
       +-- Supplementary Groups
                |
                v
         Access Decisions
                |
                v
       Files, Directories,
       Services and System
```

When additional privilege is required:

```text
Regular User
     |
     v
Is privileged action necessary?
     |
  +--+--+
  |     |
 No    Yes
  |     |
  v     v
Stay   Use controlled
normal privilege escalation
        |
        v
       sudo
```

And when troubleshooting:

```text
Who am I?
    |
    v
What UID and groups do I have?
    |
    v
Who owns the resource?
    |
    v
What access is permitted?
    |
    v
Does this task actually require privilege?
    |
    v
Apply the smallest correct change
    |
    v
Verify
```

That reasoning is more important than memorizing `sudo`.



---



| Command                                      | What it does                                                                                                                               | How to use it                                                                                                    |
| -------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------- |
| `ssh ann@server`                             | Connects to a remote server using a specific username.                                                                                     | Replace `ann` with the username and `server` with the server name or address. Example: `ssh ann@server`          |
| `ls -l`                                      | Shows a detailed listing of files and directories, including ownership information.                                                        | Run `ls -l` in the directory you want to inspect.                                                                |
| `ls -ld /path/to/directory`                  | Shows information about the directory itself, including its owner, group, and permissions.                                                 | Put the directory path after `ls -ld`. Example: `ls -ld /home/labuser`                                           |
| `whoami`                                     | Shows which user you are currently operating as.                                                                                           | Run `whoami`. This is useful before administrative or risky operations.                                          |
| `id`                                         | Shows your UID, primary GID, username, primary group, and supplementary groups.                                                            | Run `id` to inspect your current identity.                                                                       |
| `id username`                                | Shows identity information for another user.                                                                                               | Put the username after `id`. Example: `id alice`                                                                 |
| `id root`                                    | Shows root's UID, GID, and groups.                                                                                                         | Run `id root`. Root should have UID `0`.                                                                         |
| `groups`                                     | Shows the groups associated with your current account.                                                                                     | Run `groups`.                                                                                                    |
| `groups username`                            | Shows the groups another user belongs to.                                                                                                  | Put the username after `groups`. Example: `groups alice`                                                         |
| `who`                                        | Shows users who are currently logged into the system.                                                                                      | Run `who`. It can show usernames, terminals, login times, and other session information.                         |
| `w`                                          | Shows logged-in users plus additional information about their activity and sessions.                                                       | Run `w`. It provides more information than `who`.                                                                |
| `cat /etc/passwd`                            | Displays the local user-account information stored in `/etc/passwd`.                                                                       | Run `cat /etc/passwd`. Use it to inspect account records, not to edit them.                                      |
| `cat /etc/group`                             | Displays group information stored in `/etc/group`.                                                                                         | Run `cat /etc/group`.                                                                                            |
| `cat /etc/shadow`                            | Attempts to display sensitive password and password-aging information. A normal user will usually receive `Permission denied`.             | Run `cat /etc/shadow` only to understand that normal users normally cannot read it.                              |
| `sudo cat /etc/shadow`                       | Displays `/etc/shadow` with elevated privileges when the user is authorized.                                                               | Prefix the command with `sudo`. The note warns not to expose or copy real password hashes.                       |
| `less /etc/passwd`                           | Opens `/etc/passwd` in an interactive viewer.                                                                                              | Run `less /etc/passwd`. Search inside `less` when needed and press `q` to exit.                                  |
| `less /etc/group`                            | Opens `/etc/group` in an interactive viewer.                                                                                               | Run `less /etc/group`, inspect the records, then press `q` to exit.                                              |
| `/root:`                                     | Searches for `root:` while you are inside `less`.                                                                                          | Type `/root:` and press Enter while viewing the file with `less`.                                                |
| `q`                                          | Exits `less`.                                                                                                                              | Press `q` while inside `less`.                                                                                   |
| `grep '^root:' /etc/passwd`                  | Searches `/etc/passwd` for the line beginning with `root:`.                                                                                | Run the command exactly as shown when looking for the root account record.                                       |
| `grep '^developers:' /etc/group`             | Searches `/etc/group` for the `developers` group.                                                                                          | Replace `developers` with the group you want to search for.                                                      |
| `getent passwd root`                         | Queries the account database for the root account.                                                                                         | Use `getent passwd` followed by the username.                                                                    |
| `getent passwd username`                     | Looks up a user's account information through the system's account database.                                                               | Replace `username` with the account you want to inspect. Example: `getent passwd alice`                          |
| `getent group groupname`                     | Looks up information about a group.                                                                                                        | Replace `groupname` with the group you want to inspect. Example: `getent group developers`                       |
| `sudo useradd labuser`                       | Creates a user account using `useradd`. Whether a home directory is created depends on system defaults.                                    | Use `sudo useradd` followed by the username. Example: `sudo useradd labuser`                                     |
| `sudo useradd -m labuser`                    | Creates a user and requests creation of the user's home directory.                                                                         | Add `-m` before the username.                                                                                    |
| `sudo useradd -m -s /bin/bash labuser`       | Creates a user, creates a home directory, and sets Bash as the login shell.                                                                | Use `-m` for the home directory and `-s /bin/bash` for the login shell.                                          |
| `sudo adduser labuser`                       | Creates a user using the friendlier `adduser` utility available on systems such as Debian and Ubuntu.                                      | Run `sudo adduser` followed by the username and follow the prompts.                                              |
| `passwd`                                     | Changes your own password.                                                                                                                 | Run `passwd` and follow the password prompts.                                                                    |
| `sudo passwd username`                       | Sets or changes another user's password as an administrator.                                                                               | Replace `username` with the account. Example: `sudo passwd alice`                                                |
| `sudo usermod OPTIONS username`              | Modifies an existing user account.                                                                                                         | Replace `OPTIONS` with the required modification and provide the username.                                       |
| `sudo usermod -s /bin/bash labuser`          | Changes an existing user's login shell to Bash.                                                                                            | Use `-s` followed by the shell path and username.                                                                |
| `sudo usermod -l appuser labuser`            | Changes the user's login name from `labuser` to `appuser`.                                                                                 | Use `-l`, followed by the new login name and then the current username.                                          |
| `sudo groupadd developers`                   | Creates a new Linux group.                                                                                                                 | Put the new group name after `groupadd`.                                                                         |
| `sudo usermod -aG developers labuser`        | Adds a user to a supplementary group while preserving existing supplementary group memberships.                                            | `-a` means append. `-G` specifies supplementary groups. Then provide the group and username.                     |
| `sudo usermod -aG developers,operations bob` | Adds one user to multiple supplementary groups at once.                                                                                    | Separate the group names with commas and do not put spaces between them.                                         |
| `sudo usermod -g developers labuser`         | Changes the user's primary group.                                                                                                          | Use lowercase `-g` followed by the new primary group and username.                                               |
| `sudo usermod -aG sudo labuser`              | Adds a user to the `sudo` group on an Ubuntu-style system.                                                                                 | Use `-aG sudo` followed by the username. This can grant significant administrative access.                       |
| `sudo usermod -aG wheel labuser`             | Adds a user to the `wheel` group on systems where that group is used for administrative access.                                            | Use `-aG wheel` followed by the username. Actual authorization still depends on system configuration.            |
| `newgrp`                                     | Starts a shell with a changed group context. The note mentions it as one way group changes may be applied without a full logout and login. | The note mentions `newgrp` conceptually but does not provide a specific command example beyond the command name. |
| `sudo deluser labuser developers`            | Removes a user from a group on Debian and Ubuntu systems that provide `deluser`.                                                           | Give the username first and group second.                                                                        |
| `sudo gpasswd -d labuser developers`         | Removes a user from a group using `gpasswd`.                                                                                               | Use `-d`, followed by the username and group.                                                                    |
| `sudo userdel labuser`                       | Deletes a user account but does not necessarily delete the user's home directory or all files owned by that user.                          | Put the username after `userdel`.                                                                                |
| `sudo userdel -r labuser`                    | Deletes a user and requests removal of associated resources such as the home directory and mail spool.                                     | Add `-r` before the username. Use carefully because it is destructive.                                           |
| `sudo groupdel developers`                   | Deletes a group.                                                                                                                           | Put the group name after `groupdel`. Check dependencies before deleting it.                                      |
| `su - username`                              | Starts a login-style shell as another user.                                                                                                | Replace `username` with the target user. Example: `su - alice`                                                   |
| `su username`                                | Switches to another user without requesting the same login-style environment as `su - username`.                                           | Put the target username after `su`.                                                                              |
| `su -`                                       | Attempts to start a root login-style shell on systems where root password authentication is enabled.                                       | Run `su -` and provide the required root authentication if the system permits it.                                |
| `su - root`                                  | Explicitly attempts to start a login-style shell as root.                                                                                  | Run `su - root`. The note explains that this can fail where direct root password login is disabled.              |
| `sudo command`                               | Runs a particular command with elevated privileges according to the system's sudo policy.                                                  | Put `sudo` before the command that genuinely requires elevated access.                                           |
| `sudo whoami`                                | Runs `whoami` with elevated identity. It commonly returns `root`.                                                                          | Run `sudo whoami` and compare it with normal `whoami`.                                                           |
| `sudo -u username command`                   | Runs a command as a specified user rather than automatically as root.                                                                      | Replace `username` and `command`. Example: `sudo -u alice whoami`                                                |
| `sudo -u labuser whoami`                     | Tests a command specifically as `labuser`.                                                                                                 | Run it when authorized to confirm the effective user of that command.                                            |
| `sudo -i`                                    | Starts a root login-style shell when permitted.                                                                                            | Run `sudo -i`, verify with `whoami`, and use `exit` when finished.                                               |
| `exit`                                       | Leaves the current switched-user or privileged shell.                                                                                      | Run `exit` when you want to return to the previous shell.                                                        |
| `sudo -l`                                    | Lists the sudo privileges available to your current account according to the applicable policy.                                            | Run `sudo -l`. It may request authentication.                                                                    |
| `sudo visudo`                                | Opens the sudoers configuration using the safer `visudo` mechanism, which can check syntax.                                                | Run `sudo visudo` when sudoers configuration genuinely needs to be changed.                                      |
| `visudo`                                     | The tool used for safely editing sudoers configuration.                                                                                    | The note recommends using `visudo` rather than casually editing `/etc/sudoers` directly.                         |
| `rm file.txt`                                | Attempts to delete a file as your current user. It appears in the note as part of permission troubleshooting.                              | Run `rm file.txt`. If it fails with `Permission denied`, investigate before adding `sudo`.                       |
| `sudo rm file.txt`                           | Deletes the file with elevated privileges when authorized. The note specifically warns against using this automatically after `rm` fails.  | Only use it after confirming why elevated privilege is necessary.                                                |
| `pwd`                                        | Shows your current working directory.                                                                                                      | Run `pwd`. The lab uses it after switching users to confirm the current location.                                |

### The main user and group administration commands from this note

For quick revision, these are the commands Part 4 specifically says you should recognize:

| Command    | Main purpose                                                         | Basic use                                            |
| ---------- | -------------------------------------------------------------------- | ---------------------------------------------------- |
| `whoami`   | Check your current user.                                             | `whoami`                                             |
| `id`       | Check UID, GID, and groups.                                          | `id` or `id username`                                |
| `groups`   | Check group memberships.                                             | `groups` or `groups username`                        |
| `who`      | See logged-in users.                                                 | `who`                                                |
| `w`        | See logged-in users and their activity.                              | `w`                                                  |
| `getent`   | Query user and group databases.                                      | `getent passwd username` or `getent group groupname` |
| `useradd`  | Create a user.                                                       | `sudo useradd -m -s /bin/bash username`              |
| `adduser`  | Create a user using a friendlier utility on supported distributions. | `sudo adduser username`                              |
| `passwd`   | Set or change a password.                                            | `passwd` or `sudo passwd username`                   |
| `usermod`  | Modify an existing user.                                             | `sudo usermod OPTIONS username`                      |
| `userdel`  | Delete a user.                                                       | `sudo userdel username`                              |
| `groupadd` | Create a group.                                                      | `sudo groupadd groupname`                            |
| `groupdel` | Delete a group.                                                      | `sudo groupdel groupname`                            |
| `gpasswd`  | Manage group membership.                                             | `sudo gpasswd -d username groupname`                 |
| `su`       | Switch to another user.                                              | `su - username`                                      |
| `sudo`     | Run an authorized command with another identity, commonly root.      | `sudo command`                                       |
| `sudo -u`  | Run a command as a particular user.                                  | `sudo -u username command`                           |
| `sudo -i`  | Start a root login-style shell.                                      | `sudo -i`                                            |
| `sudo -l`  | Check your sudo permissions.                                         | `sudo -l`                                            |
| `visudo`   | Safely work with sudoers configuration.                              | `sudo visudo`                                        |

One especially important distinction from your note is `-g` versus `-G`. Lowercase `-g` changes the primary group, while uppercase `-G` specifies supplementary groups. When adding supplementary membership with `usermod`, the note emphasizes using `-aG` so you append the new group instead of accidentally replacing the user's existing supplementary groups. 






---

Important: do this only on a disposable Linux lab or VM, as the exercise says. User and group administration changes the system.

# Linux Identity Management Lab

## Scenario

Your company has three new team members:

```text
sarah
michael
deploybot
```

You need two groups:

```text
engineering
deployment
```

The goal is to build this basic structure:

```text
Linux Lab
|
+-- engineering
|   |
|   +-- sarah
|   +-- michael
|
+-- deployment
    |
    +-- michael
    +-- deploybot
```

---

## Step 1: Confirm who you are

Before making administrative changes, check your current identity.

```bash
whoami
```

You might see:

```text
ann
```

Then run:

```bash
id
```

You might see something similar to:

```text
uid=1000(ann) gid=1000(ann) groups=1000(ann),27(sudo)
```

Why?

`whoami` confirms which account you are currently using. `id` gives you more detail, including your UID, primary GID, and group memberships.

This is a good habit before making administrative changes.

---

# Part A: Create the groups

## Step 2: Create `engineering`

Run:

```bash
sudo groupadd engineering
```

What this does:

* `sudo` runs the command with elevated privileges.
* `groupadd` creates a Linux group.
* `engineering` is the group being created.

Verify it:

```bash
getent group engineering
```

You may see something like:

```text
engineering:x:1001:
```

The exact GID may be different on your system.

---

## Step 3: Create `deployment`

Run:

```bash
sudo groupadd deployment
```

Verify:

```bash
getent group deployment
```

You may see:

```text
deployment:x:1002:
```

At this point, both groups should exist.

You can check them individually with:

```bash
getent group engineering
getent group deployment
```

---

# Part B: Create the users

## Step 4: Create Sarah

Sarah needs:

* A user account
* A home directory
* Bash as her login shell

Run:

```bash
sudo useradd -m -s /bin/bash sarah
```

Breaking that down:

```text
sudo
```

Run the administrative command with elevated privileges.

```text
useradd
```

Create a user.

```text
-m
```

Create the user's home directory.

```text
-s /bin/bash
```

Set Bash as the login shell.

```text
sarah
```

The username.

Verify Sarah:

```bash
id sarah
```

Then check her account information:

```bash
getent passwd sarah
```

You may see something similar to:

```text
sarah:x:1001:1003::/home/sarah:/bin/bash
```

Your UID and GID numbers may be different.

Confirm her home directory:

```bash
ls -ld /home/sarah
```

---

## Step 5: Create Michael

Michael also needs a home directory and Bash shell.

```bash
sudo useradd -m -s /bin/bash michael
```

Verify:

```bash
id michael
```

Check his account:

```bash
getent passwd michael
```

Confirm his home directory:

```bash
ls -ld /home/michael
```

---

## Step 6: Create `deploybot`

The requirement only says to create `deploybot`. It does not specifically require a home directory or Bash shell.

So stay with the requirement:

```bash
sudo useradd deploybot
```

Verify:

```bash
id deploybot
```

This is important because `useradd` defaults can vary between Linux distributions.

---

# Part C: Assign group memberships

## Step 7: Add Sarah to `engineering`

Run:

```bash
sudo usermod -aG engineering sarah
```

Breaking it down:

```text
usermod
```

Modifies an existing user.

```text
-a
```

Means append.

```text
-G
```

Specifies supplementary groups.

```text
engineering
```

The group being added.

```text
sarah
```

The user being modified.

The `-a` is important. You want to add the group without replacing Sarah's other supplementary groups.

---

## Step 8: Add Michael to both groups

Michael needs:

```text
engineering
deployment
```

Run:

```bash
sudo usermod -aG engineering,deployment michael
```

Notice that the group names are separated with a comma.

Do not put a space between them.

Correct:

```bash
engineering,deployment
```

---

## Step 9: Add `deploybot` to `deployment`

Run:

```bash
sudo usermod -aG deployment deploybot
```

Now your intended structure is:

```text
engineering
|
+-- sarah
+-- michael

deployment
|
+-- michael
+-- deploybot
```

---

# Part D: Verify everything

Creating or modifying an account is not enough. Verify the result.

## Step 10: Verify Sarah's UID, GID, and groups

Run:

```bash
id sarah
```

You might see something similar to:

```text
uid=1001(sarah) gid=1003(sarah) groups=1003(sarah),1001(engineering)
```

The numbers will vary.

What matters is that you can identify:

* Sarah's UID
* Sarah's primary GID
* Sarah's primary group
* `engineering` as a supplementary group

You can also run:

```bash
groups sarah
```

You should see `engineering` listed.

---

## Step 11: Verify Michael's memberships

Run:

```bash
id michael
```

Then:

```bash
groups michael
```

You should see both:

```text
engineering
deployment
```

Michael should therefore belong to his primary group plus the two supplementary groups.

---

## Step 12: Verify `deploybot`

Run:

```bash
id deploybot
```

Then:

```bash
groups deploybot
```

You should see `deployment` among its memberships.

---

# Part E: Inspect `/etc/passwd` information safely

Instead of manually editing `/etc/passwd`, use `getent`.

## Step 13: Inspect Sarah

```bash
getent passwd sarah
```

You may see:

```text
sarah:x:1001:1003::/home/sarah:/bin/bash
```

The important parts include:

```text
sarah
```

Username.

```text
1001
```

UID in this example.

```text
1003
```

Primary GID in this example.

```text
/home/sarah
```

Home directory.

```text
/bin/bash
```

Login shell.

---

## Step 14: Inspect Michael

```bash
getent passwd michael
```

Confirm that his record includes:

```text
/home/michael
```

and:

```text
/bin/bash
```

---

## Step 15: Inspect `deploybot`

```bash
getent passwd deploybot
```

Pay attention to the result.

Because you created it using:

```bash
sudo useradd deploybot
```

its defaults may differ from Sarah's and Michael's.

That is intentional in this exercise. It reinforces that `useradd` behavior can depend on system configuration.

---

# Part F: Inspect the groups

## Step 16: Inspect `engineering`

Run:

```bash
getent group engineering
```

You may see something similar to:

```text
engineering:x:1001:sarah,michael
```

This confirms Sarah and Michael were added as supplementary members.

---

## Step 17: Inspect `deployment`

Run:

```bash
getent group deployment
```

You may see:

```text
deployment:x:1002:michael,deploybot
```

You have now verified both sides of the configuration.

You checked the users with:

```bash
id
groups
```

And the groups with:

```bash
getent group
```

---

# Part G: Switch to Sarah

## Step 18: Set a password if your lab requires one

The exercise says to switch into Sarah's account if your lab authentication configuration permits it.

Because Sarah was created using `useradd`, you may need to give her a password before using `su`.

As administrator:

```bash
sudo passwd sarah
```

Enter the temporary lab password when prompted.

You may see:

```text
New password:
Retype new password:
passwd: password updated successfully
```

The characters normally do not appear while you type the password.

---

## Step 19: Switch to Sarah

Run:

```bash
su - sarah
```

The `-` requests a login-style environment for Sarah.

If authentication is required, enter Sarah's password.

Your prompt should now represent Sarah's session.

---

## Step 20: Confirm Sarah's identity

Run:

```bash
whoami
```

Expected output:

```text
sarah
```

This confirms that your current shell is operating as Sarah.

---

## Step 21: Confirm Sarah's groups

Run:

```bash
id
```

You should see Sarah's:

* UID
* Primary GID
* Primary group
* Supplementary groups

`engineering` should be included.

You can also run:

```bash
groups
```

---

# Part H: Return to your administrator account

## Step 22: Exit Sarah's shell

Run:

```bash
exit
```

You should return to your previous administrator shell.

Confirm:

```bash
whoami
```

For example:

```text
ann
```

---

# Part I: Understand `whoami` versus `sudo whoami`

## Step 23: Run normal `whoami`

```bash
whoami
```

Suppose the output is:

```text
ann
```

That tells you your current shell belongs to `ann`.

Now run:

```bash
sudo whoami
```

You may see:

```text
root
```

## Why are the outputs different?

They are answering the same question under different execution identities.

When you run:

```bash
whoami
```

the command runs as your normal user.

Conceptually:

```text
ann
 |
 v
whoami
 |
 v
ann
```

When you run:

```bash
sudo whoami
```

`sudo` runs that particular command with elevated identity according to the sudo policy, commonly as root.

```text
ann
 |
 v
sudo whoami
 |
 v
root
```

Your whole shell did not become root.

If you run:

```bash
whoami
```

again afterward, it should still return:

```text
ann
```

That is one of the major reasons `sudo` is useful. You can elevate a particular command without remaining logged in as root.

---

# Part J: Remove Sarah from `engineering`

## Step 24: Remove Sarah

The note gives two possible approaches depending on the distribution.

On Debian or Ubuntu, you may use:

```bash
sudo deluser sarah engineering
```

Or, on a system providing GNU `gpasswd`:

```bash
sudo gpasswd -d sarah engineering
```

For example:

```bash
sudo gpasswd -d sarah engineering
```

may report something similar to:

```text
Removing user sarah from group engineering
```

---

## Step 25: Verify the removal

Run:

```bash
groups sarah
```

Also run:

```bash
id sarah
```

`engineering` should no longer appear as one of Sarah's supplementary groups.

You can also inspect the group:

```bash
getent group engineering
```

Sarah should no longer be listed there as a supplementary member.

---

# Part K: Add Sarah back

## Step 26: Add her back to `engineering`

Run:

```bash
sudo usermod -aG engineering sarah
```

Verify:

```bash
id sarah
```

And:

```bash
groups sarah
```

`engineering` should be back.

---

# Part L: Why might Sarah's existing session not immediately see the change?

This is an important part of the exercise.

Suppose Sarah was already logged in before you changed her group membership.

You add her:

```bash
sudo usermod -aG engineering sarah
```

The account database can now show that Sarah belongs to `engineering`.

For example, from the administrator account:

```bash
id sarah
```

may show the new membership.

But an already-running Sarah login session may still be using the group list that was established when that session started.

Conceptually:

```text
Sarah logs in
      |
      v
Session receives group information
      |
      v
Administrator changes Sarah's groups
      |
      v
Account database is updated
      |
      v
Old Sarah session may still have
its existing group context
```

The simple beginner solution is for Sarah to:

```text
log out
log back in
```

The new session can then receive the updated group membership.

The note also mentions `newgrp` as a tool that can start a shell with changed group context, but logging out and back in is the simpler approach for this exercise.

---

# Part M: Create the disposable `removeme` account

Now you will practice deleting an account.

## Step 27: Create `removeme`

Because the exercise specifically wants you to confirm that `/home/removeme` exists, create the account with `-m`:

```bash
sudo useradd -m removeme
```

The `-m` requests creation of the home directory.

Verify the account:

```bash
id removeme
```

---

## Step 28: Confirm the home directory exists

Run:

```bash
ls -ld /home/removeme
```

You should see information for:

```text
/home/removeme
```

You can also inspect the account:

```bash
getent passwd removeme
```

The account now exists, and its home directory exists.

---

# Part N: Delete `removeme`

## Step 29: Delete the account with `userdel -r`

Run:

```bash
sudo userdel -r removeme
```

Breaking it down:

```text
userdel
```

Deletes the user account.

```text
-r
```

Requests removal of associated user files handled by the command, such as the home directory and mail spool.

```text
removeme
```

The account being deleted.

This is destructive, which is why you are practicing on a disposable account.

---

# Part O: Verify that the account is gone

## Step 30: Check with `id`

Run:

```bash
id removeme
```

You should receive a message indicating that the user does not exist, such as:

```text
id: 'removeme': no such user
```

Also check:

```bash
getent passwd removeme
```

You should no longer receive the account record.

This verifies that the account has been removed.

---

# Part P: Check the home directory

Run:

```bash
ls -ld /home/removeme
```

You would normally expect something similar to:

```text
ls: cannot access '/home/removeme': No such file or directory
```

That demonstrates the difference between:

```bash
sudo userdel removeme
```

and:

```bash
sudo userdel -r removeme
```

The `-r` version requests removal of associated resources such as the user's home directory.

---

# Part Q: Why `userdel -r` is not enough for a real production user

This is one of the most important lessons in the lab.

A real production user may own or depend on much more than:

```text
/home/username
```

They may have files or responsibilities elsewhere, for example:

```text
/srv/application
/opt/project
/var/lib/application
```

So before deleting a real production account, you need to investigate.

You would want to know:

* Does the user have important data that must be preserved?
* Does anything need to be archived?
* Does the account own files outside its home directory?
* Are applications or automation using the account?
* Are processes still running as the user?
* Does the account have group memberships that provide production access?
* Do scripts, scheduled jobs, or configuration reference the username?
* What happens to files still associated with the user's UID after the account disappears?

Deleting the account does not automatically mean every file associated with its UID disappears.

You could end up seeing numeric ownership such as:

```text
1005
```

instead of a username if files remain but the account that previously mapped to that UID has been deleted.

So:

```bash
sudo userdel -r username
```

should never mean:

```text
"The employee left, so just run this command."
```

The correct production thinking is:

```text
Investigate
    |
    v
Understand dependencies
    |
    v
Preserve required data
    |
    v
Understand ownership
    |
    v
Remove access
    |
    v
Delete when appropriate
    |
    v
Verify
```

# Final completed configuration

Before the temporary removal and re-addition exercise, your intended user and group configuration should be:

```text
Linux Lab
|
+-- engineering
|   |
|   +-- sarah
|   +-- michael
|
+-- deployment
    |
    +-- michael
    +-- deploybot
```

Your key verification commands are:

```bash
id sarah
id michael
id deploybot

groups sarah
groups michael
groups deploybot

getent passwd sarah
getent passwd michael
getent passwd deploybot

getent group engineering
getent group deployment
```

And the main lesson from the entire exercise is not simply how to create users. It is the full administration pattern:

```text
Create
  |
  v
Configure
  |
  v
Verify
  |
  v
Test identity
  |
  v
Modify
  |
  v
Verify again
  |
  v
Delete carefully
  |
  v
Verify again
```

That is the habit you want to carry into real Linux and DevOps work.
---


# Next Part

Part 5 should build directly on this identity foundation:

Linux File Ownership and Permissions.

You will connect users and groups to actual filesystem access, including ownership, `rwx`, permission classes, numeric permissions, `chmod`, `chown`, `chgrp`, directory permissions, `umask`, special permissions, permission troubleshooting, and why `chmod 777` is usually the wrong response to a permissions problem.
