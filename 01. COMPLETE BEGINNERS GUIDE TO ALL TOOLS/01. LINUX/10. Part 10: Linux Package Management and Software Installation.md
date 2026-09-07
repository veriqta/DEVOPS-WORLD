# Part 10: Linux Package Management and Software Installation

In Part 9, you learned how Linux services work, how systemd manages them, how services start during boot, and how to troubleshoot failed services.

Now we move to another fundamental Linux administration skill: installing and managing software.

A Linux server rarely contains every tool you will ever need. You may need to install:

* NGINX.
* Git.
* curl.
* Python.
* Docker dependencies.
* Monitoring tools.
* Database clients.
* Troubleshooting utilities.
* Security updates.

Linux distributions solve this through package-management systems.

By the end of Part 10, you should understand not only how to type:

```bash
sudo apt install nginx
```

but what actually happens when you run it, where the software comes from, how dependencies are resolved, how package databases work, how APT differs from DPKG, how DNF relates to RPM, and how to troubleshoot common package-management failures.

---

# 1. What Is a Software Package?

A software package is a structured bundle used to distribute software on Linux.

A package can contain:

* Executable programs.
* Libraries.
* Configuration files.
* Documentation.
* Service definitions.
* Metadata.
* Installation scripts.
* Dependency information.

Suppose you want to install NGINX.

You could theoretically:

1. Download its source code.
2. Install build tools.
3. Install required libraries.
4. Compile the source.
5. Copy binaries into appropriate directories.
6. Create configuration directories.
7. Create service files.
8. Track every installed file manually.
9. Repeat the process when an update becomes available.

That would be inconvenient for everyday system administration.

Instead, Linux distributions package software into manageable units.

Conceptually:

```text
Software source code
        |
        v
Distribution/package maintainer
        |
        v
Software package
        |
        v
Repository
        |
        v
Package manager
        |
        v
Your Linux system
```

---

# 2. What Does a Package Contain?

Imagine a fictional package called:

```text
veriqta-agent
```

It might install:

```text
/usr/bin/veriqta-agent
/etc/veriqta-agent/config.conf
/usr/share/doc/veriqta-agent/
/usr/lib/systemd/system/veriqta-agent.service
```

The package also contains metadata describing information such as:

```text
Package name
Version
Architecture
Dependencies
Description
Maintainer
Installed size
```

The package-management system records this information so Linux can later determine:

* What is installed.
* Which version is installed.
* Which files belong to the package.
* Which dependencies it requires.
* Whether an update is available.
* How to remove it.

---

# 3. Package Management vs Downloading a Program

Beginners often think installing software simply means:

```text
Download a file
        |
        v
Run it
```

Package management is more structured.

A package manager can handle:

```text
Find software
     |
     v
Determine correct version
     |
     v
Resolve dependencies
     |
     v
Download packages
     |
     v
Verify package information
     |
     v
Install files
     |
     v
Run required package scripts
     |
     v
Update package database
```

This is one reason package managers are preferable to downloading random software from the internet.

---

# 4. What Is a Package Repository?

A package repository is a collection of software packages and associated metadata made available to package-management tools.

Think of it as an organized software source for your Linux distribution.

Conceptually:

```text
Linux distribution repositories
            |
     +------+------+
     |      |      |
     v      v      v
   Git    NGINX   curl
     |      |      |
     +------+------+
            |
            v
      Package manager
            |
            v
        Your server
```

Repositories allow the package manager to discover:

* Available packages.
* Available versions.
* Dependencies.
* Updates.
* Package architecture.
* Package descriptions.

---

# 5. Why Repositories Exist

Without repositories, administrators would have to search the internet individually for software.

That creates several problems:

* Where should you download it?
* Which version is correct?
* Is it compatible with your Linux distribution?
* Is the download trustworthy?
* Which dependencies are required?
* How will updates be delivered?
* How will you know about security fixes?

Repositories provide a controlled software-distribution mechanism.

---

# 6. Official and Third-Party Repositories

Repositories generally fall into categories such as:

```text
Official distribution repositories

Vendor repositories

Third-party repositories

Internal company repositories
```

An official Ubuntu repository, for example, contains packages maintained or distributed through Ubuntu's packaging infrastructure.

A software vendor may also maintain its own repository.

Organizations may operate internal repositories containing:

* Approved software.
* Internal applications.
* Specific package versions.
* Company-built packages.

Do not assume every repository on the internet is trustworthy.

Adding a repository effectively allows that source to participate in software installation and updates on your system.

That is a security decision.

---

# 7. What Is Package Metadata?

Package metadata is information describing a package.

It can include:

```text
Name
Version
Architecture
Description
Dependencies
Conflicts
Repository
Maintainer
Installed size
Checksums
```

Suppose a package says:

```text
Package: example-web
Version: 2.1.0
Architecture: amd64
Depends: libc6, openssl
```

The package manager can use this information to determine what else must be installed.

---

# 8. Package Metadata vs Package Contents

These are different concepts.

Package metadata describes the package.

Package contents are the actual files installed by the package.

Think:

```text
Package
 |
 +---- Metadata
 |       |
 |       +---- name
 |       +---- version
 |       +---- dependencies
 |
 +---- Contents
         |
         +---- binaries
         +---- libraries
         +---- configuration
         +---- documentation
```

---

# 9. What Is a Dependency?

A dependency is another piece of software required for a package to function correctly.

Imagine:

```text
Application A
     |
     +---- requires Library B
     |
     +---- requires Library C
```

If you install Application A without B and C, it may not work.

Dependencies can themselves have dependencies:

```text
Application A
     |
     +--> Library B
     |       |
     |       +--> Library D
     |
     +--> Library C
```

This forms a:

```text
dependency graph
```

Package managers help resolve this graph.

---

# 10. Why Dependencies Exist

Developers usually do not rewrite every capability their programs need.

An application may depend on existing libraries for:

* Encryption.
* Compression.
* Networking.
* Database connectivity.
* XML processing.
* JSON handling.
* Graphics.
* Authentication.

This encourages software reuse.

But it means installation sometimes requires more than one package.

---

# 11. What Is Dependency Resolution?

Suppose you request:

```text
Install Package A
```

Package A requires:

```text
Package B
Package C
```

Package B requires:

```text
Package D
```

A higher-level package manager can calculate:

```text
A
|
+--> B
|    |
|    +--> D
|
+--> C
```

and install the necessary packages.

This is dependency resolution.

---

# 12. What Is a Package Manager?

A package manager is software that helps install, update, query, and remove software packages.

Different Linux distribution families use different package-management systems.

Two major families you will encounter are:

```text
Debian family
     |
     +--> .deb
     +--> dpkg
     +--> APT

Red Hat family
     |
     +--> .rpm
     +--> RPM
     +--> DNF
     +--> YUM
```

Understanding these relationships is more important than memorizing commands.

---

# 13. High-Level vs Low-Level Package Tools

A useful beginner model is:

```text
High-level package manager
        |
        +--> repositories
        +--> metadata
        +--> dependency resolution
        +--> updates
        |
        v
Low-level package tool
        |
        +--> package file
        +--> local package database
        +--> installation/removal mechanics
```

For Debian-based systems:

```text
APT
 |
 v
dpkg
```

For Red Hat-family systems:

```text
DNF
 |
 v
RPM
```

This is simplified, but it gives you the correct mental model.

---

# 14. Debian-Based Linux Systems

Debian-family distributions include systems such as:

* Debian.
* Ubuntu.
* Linux Mint.
* Many Debian or Ubuntu-derived distributions.

They commonly use:

```text
.deb packages
dpkg
APT
```

The primary beginner-facing tool is usually:

```bash
apt
```

---

# 15. What Is APT?

APT stands for:

```text
Advanced Package Tool
```

APT provides higher-level package-management functionality for Debian-based systems.

It can:

* Search repositories.
* Retrieve package metadata.
* Resolve dependencies.
* Download packages.
* Install software.
* Upgrade software.
* Remove software.

Common commands include:

```bash
sudo apt update
sudo apt upgrade
sudo apt install PACKAGE
sudo apt remove PACKAGE
sudo apt purge PACKAGE
sudo apt autoremove
apt search PACKAGE
apt show PACKAGE
```

We will examine each one.

---

# 16. Before Installing Software: Know Your Distribution

Do not blindly copy:

```bash
sudo apt install ...
```

because not every Linux distribution uses APT.

Check:

```bash
cat /etc/os-release
```

You may see information such as:

```text
NAME="Ubuntu"
ID=ubuntu
ID_LIKE=debian
```

or:

```text
NAME="Rocky Linux"
ID="rocky"
ID_LIKE="rhel centos fedora"
```

This helps determine the distribution family.

---

# 17. Finding Your Package Manager

You can check:

```bash
which apt
```

or:

```bash
which dnf
```

or:

```bash
which yum
```

or:

```bash
which rpm
```

or:

```bash
which dpkg
```

Remember from earlier parts:

```text
which
```

searches for commands using your shell's command search path.

---

# 18. `apt update`

One of the most misunderstood beginner commands is:

```bash
sudo apt update
```

It does not normally upgrade your installed software.

Instead, it refreshes your local knowledge of packages available from configured repositories.

Conceptually:

```text
Configured repositories
        |
        v
Download current metadata
        |
        v
Local package information updated
```

Think:

```text
apt update
=
Update my package lists.
```

Not:

```text
Update every installed application.
```

---

# 19. Why `apt update` Matters

Suppose your local package information says:

```text
nginx latest known version = X
```

But the repository now contains:

```text
nginx latest version = Y
```

Your machine may not know about Y until package metadata is refreshed.

Running:

```bash
sudo apt update
```

retrieves current repository information.

---

# 20. Understanding `apt update` Output

You may see lines beginning with terms such as:

```text
Hit:
Get:
Ign:
Err:
```

These indicate what APT is doing with configured repository sources.

At the end, you may see something similar to:

```text
Reading package lists... Done
Building dependency tree... Done
```

If packages can be upgraded, APT may report that too.

Read errors carefully.

Do not assume the operation succeeded merely because many lines were printed.

---

# 21. `apt upgrade`

After updating package metadata, you may run:

```bash
sudo apt upgrade
```

APT calculates available upgrades for installed packages under the rules of the `upgrade` operation.

It normally shows what it plans to do before asking for confirmation.

Read that transaction summary.

You may see information about:

* Packages to upgrade.
* Packages kept back.
* Additional disk space.
* Download size.

---

# 22. `apt update` vs `apt upgrade`

Remember:

```text
apt update
    |
    v
Refresh repository/package metadata

apt upgrade
    |
    v
Upgrade installed packages according to available metadata
```

A common workflow is:

```bash
sudo apt update
sudo apt upgrade
```

But on production systems, do not treat system-wide upgrades as a harmless routine command.

Updates can change libraries, services, kernels, application behavior, and compatibility.

---

# 23. `apt install`

To install a package:

```bash
sudo apt install PACKAGE
```

Example:

```bash
sudo apt install nginx
```

APT will:

1. Locate the package in configured repositories.
2. Determine an appropriate version.
3. Resolve dependencies.
4. Show the proposed transaction.
5. Download required packages.
6. Verify package data using the repository trust mechanisms.
7. Install packages.
8. Run relevant package-maintainer actions.
9. Update package state.

---

# 24. Always Read the Installation Plan

Before confirming, inspect what APT wants to do.

Suppose you request one package and APT says:

```text
The following additional packages will be installed:
...
```

Those are dependencies.

If it unexpectedly wants to:

* Remove important packages.
* Replace large parts of the system.
* Install an unexpectedly large dependency set.

stop and investigate before confirming.

Do not develop the habit of automatically pressing:

```text
Y
```

without reading.

---

# 25. Installing Multiple Packages

You can install multiple packages in one command:

```bash
sudo apt install git curl tree
```

Here:

```text
apt
```

is the program.

```text
install
```

is the action.

```text
git curl tree
```

are package-name arguments.

---

# 26. Verify Installation

Do not stop at:

```text
Installation completed.
```

Verify.

For Git:

```bash
git --version
```

For curl:

```bash
curl --version
```

For a command:

```bash
which git
```

You can also query the package database:

```bash
dpkg -l git
```

Different software requires different verification.

A service package might additionally require:

```bash
systemctl status nginx
```

and application-level testing.

---

# 27. Installing a Service Package

Suppose:

```bash
sudo apt install nginx
```

After installation, check:

```bash
nginx -v
```

Then:

```bash
systemctl status nginx
```

Then:

```bash
sudo ss -ltnp | grep ':80'
```

Then:

```bash
curl -I http://127.0.0.1
```

This connects Part 10 with Part 9.

Installation is only one layer.

```text
Package installed
      |
      v
Binary available
      |
      v
Service state
      |
      v
Port
      |
      v
Application functionality
```

---

# 28. `apt remove`

To remove an installed package:

```bash
sudo apt remove PACKAGE
```

Example:

```bash
sudo apt remove nginx
```

This removes the package but may preserve certain configuration files.

That can be useful if you intend to reinstall the package later while retaining configuration.

---

# 29. `apt purge`

To remove a package and its package-managed configuration files:

```bash
sudo apt purge PACKAGE
```

Example:

```bash
sudo apt purge nginx
```

The conceptual distinction is:

```text
remove
Remove package, preserve certain configuration.

purge
Remove package plus package-managed configuration.
```

Do not assume purge removes every file ever created by the application.

Application data, logs, user-created files, home-directory files, or files generated at runtime may remain.

---

# 30. Remove vs Purge

Suppose NGINX has configuration under:

```text
/etc/nginx/
```

A normal remove may preserve configuration state.

A purge performs a more complete package-level removal of configuration.

Before purging important software, inspect:

* Configuration backups.
* Application data.
* Secrets.
* Certificates.
* Custom files.

Do not use purge casually on production systems.

---

# 31. `apt autoremove`

Dependencies may have been installed automatically because another package required them.

Later, the original package may be removed.

Some automatically installed dependencies may no longer be needed.

APT can identify packages it considers no longer required:

```bash
sudo apt autoremove
```

Read the proposed removal list before confirming.

Do not assume every package marked for autoremove is irrelevant to your environment.

---

# 32. Previewing Autoremove

A safe habit is to inspect first.

Run:

```bash
sudo apt autoremove
```

and read the transaction before confirming.

You can cancel if the removal list looks wrong.

Package managers are tools.

They do not understand your business requirements or every manually designed dependency outside their package metadata.

---

# 33. Searching for Packages With APT

Suppose you want to find NGINX-related packages.

Run:

```bash
apt search nginx
```

This searches available package metadata.

You may receive many results.

Use:

```bash
apt search '^nginx$'
```

when you want to narrow a search using a regular expression.

For beginners, start with:

```bash
apt search nginx
```

Then inspect relevant results.

---

# 34. Package Information With `apt show`

Run:

```bash
apt show nginx
```

You may see fields such as:

```text
Package
Version
Priority
Section
Maintainer
Installed-Size
Depends
Homepage
Description
```

This helps answer:

```text
What is this package?

Which version is available?

What does it depend on?

How large is it?

What does it provide?
```

---

# 35. Checking Installed Packages With APT

You can list installed packages:

```bash
apt list --installed
```

This can be long.

Filter:

```bash
apt list --installed 2>/dev/null | grep nginx
```

You can also query package state with `dpkg`.

---

# 36. What Is DPKG?

`dpkg` is the lower-level package-management tool used with Debian packages.

APT and dpkg are related, but they are not identical.

A useful mental model:

```text
APT
 |
 +--> repository management
 +--> package selection
 +--> dependency resolution
 +--> downloading
 |
 v
dpkg
 |
 +--> install .deb contents
 +--> remove package
 +--> maintain local package database
 +--> query installed package state
```

APT generally uses lower-level Debian package infrastructure to perform installation operations.

---

# 37. What Is a `.deb` File?

A `.deb` file is a Debian binary package file.

Example:

```text
example-app_1.2.0_amd64.deb
```

The filename may tell you:

```text
example-app
Package name

1.2.0
Version

amd64
Architecture

.deb
Debian package format
```

Filename conventions can vary, so package metadata remains authoritative.

---

# 38. Inspecting a `.deb` Package

If you have:

```text
example.deb
```

you can inspect metadata with:

```bash
dpkg-deb -I example.deb
```

List its files:

```bash
dpkg-deb -c example.deb
```

This allows you to inspect a package before installing it.

That is a better habit than blindly executing downloaded software.

---

# 39. Installing a `.deb` With `dpkg`

The classic command is:

```bash
sudo dpkg -i package.deb
```

Here:

```text
dpkg
Debian package tool.

-i
Install.

package.deb
Local package file.
```

But there is an important limitation.

`dpkg` itself does not perform repository-based dependency resolution in the same way APT does.

If required dependencies are missing, installation can be left incomplete.

---

# 40. Installing a Local `.deb` With APT

On modern Debian-based systems, a convenient approach is often:

```bash
sudo apt install ./package.deb
```

The:

```text
./
```

tells APT you are referring to a local file in the current directory.

APT can then use its dependency-resolution capabilities while installing the local package.

This is often preferable to directly using:

```bash
sudo dpkg -i package.deb
```

when dependencies may need resolution.

---

# 41. Why `./package.deb` Matters

Compare:

```bash
sudo apt install example
```

with:

```bash
sudo apt install ./example.deb
```

The first asks APT for a package named:

```text
example
```

from configured package sources.

The second points to:

```text
a local file
```

in your current directory.

This is another example of why paths matter in Linux.

---

# 42. Querying Packages With `dpkg -l`

Run:

```bash
dpkg -l
```

This lists package information from the local dpkg database.

To search:

```bash
dpkg -l | grep nginx
```

Or query a specific package:

```bash
dpkg -l nginx
```

The output includes status codes, package names, versions, architectures, and descriptions.

---

# 43. Checking Whether a Debian Package Is Installed

A useful command is:

```bash
dpkg -s nginx
```

If installed, it displays package status and metadata.

Look for:

```text
Status: install ok installed
```

If it is not installed, dpkg reports that accordingly.

---

# 44. Finding Files Installed by a Package

Suppose you want to know:

```text
What files did nginx install?
```

Run:

```bash
dpkg -L nginx
```

This lists files associated with the installed package.

This can help locate:

* Configuration files.
* Documentation.
* Binaries.
* Service files.

---

# 45. Finding Which Package Owns a File

Suppose you find:

```text
/usr/bin/curl
```

and want to know which installed package owns it.

Run:

```bash
dpkg -S /usr/bin/curl
```

You may see:

```text
curl: /usr/bin/curl
```

This is very useful during troubleshooting.

---

# 46. Package Version vs Command Version

There are two useful questions:

```text
Which package version is installed?
```

and:

```text
Which version does the application itself report?
```

For example:

```bash
dpkg -s curl | grep '^Version:'
```

and:

```bash
curl --version
```

These values are related, but their formatting may differ.

Distribution package versions can contain additional packaging revisions.

---

# 47. Debian Package Architecture

Check your system architecture:

```bash
dpkg --print-architecture
```

Common result:

```text
amd64
```

Other architectures include:

```text
arm64
```

A package built for the wrong architecture may not install or run correctly.

This matters especially on:

* ARM cloud instances.
* Raspberry Pi systems.
* Apple Silicon virtualization.
* Mixed-architecture environments.

---

# 48. Debian Repository Configuration

APT needs to know which repositories to use.

Historically, repository configuration is commonly associated with:

```text
/etc/apt/sources.list
```

and:

```text
/etc/apt/sources.list.d/
```

Modern systems may also use `.sources` files in deb822 format under:

```text
/etc/apt/sources.list.d/
```

Do not assume every modern Ubuntu or Debian machine stores all repository definitions directly inside `/etc/apt/sources.list`.

---

# 49. Inspecting APT Sources

You can inspect:

```bash
cat /etc/apt/sources.list
```

if it exists.

And:

```bash
ls -l /etc/apt/sources.list.d/
```

Do not edit repository configuration merely to experiment.

A malformed repository entry can break package metadata refreshes.

---

# 50. Red Hat-Family Systems

Now we move to another major Linux ecosystem.

Red Hat-family systems include distributions such as:

* Red Hat Enterprise Linux.
* Rocky Linux.
* AlmaLinux.
* Fedora, although Fedora often introduces newer package-management behavior earlier.
* Amazon Linux, which is RPM-based and uses DNF or YUM-style tooling depending on version.

These systems commonly use:

```text
.rpm packages
RPM
DNF
YUM
```

---

# 51. What Is RPM?

RPM originally referred to Red Hat Package Manager and is now commonly expanded as the recursive name:

```text
RPM Package Manager
```

RPM is both:

* A package format.
* A low-level package-management system/tool.

RPM packages usually use:

```text
.rpm
```

Example:

```text
example-app-1.2.0-1.x86_64.rpm
```

---

# 52. RPM vs DNF

Think:

```text
DNF
 |
 +--> repositories
 +--> dependency resolution
 +--> downloads
 +--> updates
 |
 v
RPM
 |
 +--> install package payload
 +--> query package database
 +--> verify package information
```

This parallels:

```text
APT
 |
 v
dpkg
```

on Debian-based systems.

---

# 53. What Is DNF?

DNF is a high-level package manager used on modern RPM-based distributions.

It handles:

* Repository metadata.
* Package searching.
* Dependency resolution.
* Installation.
* Removal.
* Updates.
* Package queries.

Basic syntax:

```bash
sudo dnf install PACKAGE
```

Example:

```bash
sudo dnf install nginx
```

---

# 54. What Is YUM?

YUM is an older high-level package manager associated with RPM-based systems.

You will still see commands such as:

```bash
sudo yum install nginx
```

in tutorials, scripts, older servers, and documentation.

On some modern systems, `yum` is provided as a compatibility interface backed by DNF.

Do not assume that relationship is identical on every distribution or version.

For modern RHEL-family environments, learn DNF first while recognizing YUM commands.

---

# 55. Installing With DNF

Use:

```bash
sudo dnf install PACKAGE
```

Example:

```bash
sudo dnf install git
```

DNF calculates:

* Package version.
* Dependencies.
* Repository source.
* Download size.
* Installation transaction.

Read the transaction summary before confirming.

---

# 56. Installing Multiple Packages With DNF

Example:

```bash
sudo dnf install git curl tree
```

The package manager resolves all required dependencies and presents the transaction.

Afterward, verify:

```bash
git --version
curl --version
tree --version
```

if those commands are provided by the installed packages.

---

# 57. Removing Packages With DNF

Use:

```bash
sudo dnf remove PACKAGE
```

Example:

```bash
sudo dnf remove nginx
```

Read the proposed transaction carefully.

Removing one package can potentially affect dependent packages.

Do not automatically approve removal operations on production systems.

---

# 58. Updating Software With DNF

A common command is:

```bash
sudo dnf upgrade
```

You may also encounter:

```bash
sudo dnf update
```

On modern DNF systems these are generally treated equivalently for normal package upgrades.

Before applying broad updates on important systems:

* Review available changes.
* Understand service impact.
* Consider maintenance windows.
* Check application compatibility.
* Have rollback or recovery plans where appropriate.

---

# 59. Checking for Available Updates

Instead of immediately upgrading everything:

```bash
dnf check-update
```

can be used to check for available package updates.

An important scripting detail is that `dnf check-update` can use a nonzero exit code to indicate that updates are available.

So do not automatically interpret every nonzero exit status from this particular command as a command failure.

Context matters.

---

# 60. Searching Packages With DNF

Use:

```bash
dnf search nginx
```

For information:

```bash
dnf info nginx
```

For installed package information:

```bash
dnf info --installed nginx
```

Availability and output can vary somewhat by DNF version.

---

# 61. Listing Installed Packages With DNF

Use:

```bash
dnf list installed
```

Filter:

```bash
dnf list installed | grep nginx
```

You can also use RPM directly.

---

# 62. Querying Installed RPM Packages

List all installed RPM packages:

```bash
rpm -qa
```

Break it down:

```text
rpm
RPM command.

-q
Query.

-a
All installed packages.
```

Search:

```bash
rpm -qa | grep nginx
```

---

# 63. Querying a Specific RPM Package

Use:

```bash
rpm -q nginx
```

If installed, you may see something similar to:

```text
nginx-1.x.x-...
```

If not installed, RPM reports that the package is not installed.

---

# 64. Detailed RPM Package Information

Use:

```bash
rpm -qi nginx
```

Here:

```text
-q
Query.

-i
Information.
```

You may see:

```text
Name
Version
Release
Architecture
Install Date
Size
License
Summary
Description
```

---

# 65. Listing Files From an Installed RPM Package

Use:

```bash
rpm -ql nginx
```

Here:

```text
-l
List files.
```

This helps answer:

```text
Which files belong to this installed package?
```

---

# 66. Finding Which RPM Package Owns a File

Suppose:

```text
/usr/bin/curl
```

exists.

Run:

```bash
rpm -qf /usr/bin/curl
```

Here:

```text
-f
Query the package owning a file.
```

This is the RPM-family equivalent of the type of investigation you performed with:

```bash
dpkg -S /usr/bin/curl
```

---

# 67. Inspecting an RPM File Before Installation

If you downloaded:

```text
example.rpm
```

inspect package information:

```bash
rpm -qip example.rpm
```

List package contents:

```bash
rpm -qlp example.rpm
```

The:

```text
-p
```

means query a package file rather than an already installed package.

Inspection before installation is a useful security and troubleshooting habit.

---

# 68. Installing a Local RPM

RPM itself supports installation operations, such as:

```bash
sudo rpm -ivh package.rpm
```

But as with dpkg, directly using the low-level package tool can make dependency management less convenient.

For a local RPM on a DNF-based system, a better approach is often:

```bash
sudo dnf install ./package.rpm
```

DNF can resolve required dependencies from configured repositories.

---

# 69. Why Prefer DNF for Normal Installation?

Compare:

```text
rpm
Low-level package operations.

dnf
Higher-level repository and dependency management.
```

For routine administration:

```bash
sudo dnf install PACKAGE
```

is generally preferable to manually downloading and installing arbitrary RPM files.

---

# 70. RPM Package Architecture

Common RPM architecture labels include:

```text
x86_64
aarch64
noarch
```

`noarch` means the package is not tied to a particular CPU architecture in the same way compiled architecture-specific packages are.

Check machine architecture:

```bash
uname -m
```

Possible results:

```text
x86_64
aarch64
```

Architecture mismatches can cause installation failures.

---

# 71. RPM Repository Configuration

DNF repository definitions are commonly found under:

```text
/etc/yum.repos.d/
```

Inspect:

```bash
ls -l /etc/yum.repos.d/
```

Repository files commonly use:

```text
.repo
```

extensions.

Do not casually modify repository files on important systems.

---

# 72. Debian vs Red Hat Package Management

A useful comparison:

| Task                     | Debian Family            | Red Hat Family           |
| ------------------------ | ------------------------ | ------------------------ |
| Package format           | `.deb`                   | `.rpm`                   |
| Low-level tool           | `dpkg`                   | `rpm`                    |
| High-level manager       | APT                      | DNF                      |
| Older/common alternative | `apt-get`                | YUM                      |
| Install                  | `apt install`            | `dnf install`            |
| Remove                   | `apt remove`             | `dnf remove`             |
| Search                   | `apt search`             | `dnf search`             |
| Package info             | `apt show`               | `dnf info`               |
| Local package            | `apt install ./file.deb` | `dnf install ./file.rpm` |

Do not memorize the table without understanding the architecture behind it.

---

# 73. Checking Software Versions

There are several ways to check versions.

Application-specific commands are common:

```bash
git --version
```

```bash
curl --version
```

```bash
nginx -v
```

```bash
python3 --version
```

Not every application uses the same version flag.

You may encounter:

```text
--version
-version
-v
-V
```

Check:

```bash
COMMAND --help
```

or documentation when unsure.

---

# 74. Package Version vs Binary Version

Suppose:

```bash
nginx -v
```

shows an NGINX upstream software version.

The package database may show something like:

```text
1.x.x-distribution-revision
```

The distribution may append packaging information.

Therefore:

```text
Application version
```

and:

```text
Distribution package version string
```

may not look identical.

---

# 75. Finding Installed Software

There are multiple questions hidden inside:

```text
Is NGINX installed?
```

You might mean:

```text
Does the command exist?
```

Check:

```bash
command -v nginx
```

Or:

```bash
which nginx
```

You might mean:

```text
Is the package installed?
```

Debian:

```bash
dpkg -s nginx
```

RPM:

```bash
rpm -q nginx
```

You might mean:

```text
Is the service installed and known to systemd?
```

Check:

```bash
systemctl status nginx
```

These are related but different questions.

---

# 76. Binary Exists but Package Manager Does Not Know It

Suppose:

```bash
command -v example
```

returns:

```text
/usr/local/bin/example
```

but:

```bash
dpkg -S /usr/local/bin/example
```

finds nothing.

Possible explanation:

The program may have been:

* Compiled manually.
* Installed by a script.
* Copied manually.
* Installed by another language-specific package manager.
* Placed there by configuration management.

Not every file on Linux belongs to an OS package.

---

# 77. Package Installed but Command Not Found

The opposite can also happen.

A package may be installed, but:

```bash
example
```

returns:

```text
command not found
```

Possible reasons:

* Package does not provide a command with that name.
* Binary is outside your `PATH`.
* Command has a different name.
* Shell command cache needs refresh in unusual cases.
* Installation is incomplete.
* The package only provides libraries or data.

Investigate rather than reinstalling blindly.

---

# 78. Finding Package Contents

Debian:

```bash
dpkg -L PACKAGE
```

RPM:

```bash
rpm -ql PACKAGE
```

Then search for likely binaries:

```bash
dpkg -L PACKAGE | grep '/bin/'
```

or:

```bash
rpm -ql PACKAGE | grep '/bin/'
```

This helps determine what command a package actually installed.

---

# 79. Dependencies During Installation

Suppose:

```bash
sudo apt install example
```

requires:

```text
libA
libB
libC
```

APT handles those dependencies if suitable packages are available.

Likewise:

```bash
sudo dnf install example
```

performs dependency resolution using configured RPM repositories.

Problems occur when:

* Required dependency is unavailable.
* Versions conflict.
* Repository metadata is inconsistent.
* Third-party repositories conflict.
* A package installation was interrupted.
* Packages are held or pinned.
* System state is partially configured.

---

# 80. Dependency Failure Example

Imagine:

```text
application-x
requires libexample >= 3.0
```

but your repositories provide only:

```text
libexample 2.5
```

The package manager cannot simply invent version 3.0.

Possible solutions depend on context:

* Use a repository appropriate for your distribution.
* Upgrade the distribution if required.
* Use a vendor-supported repository.
* Install a compatible application version.
* Correct repository configuration.

Do not randomly download libraries from unrelated websites and overwrite system files.

---

# 81. What Is Dependency Hell?

"Dependency hell" informally describes complicated dependency conflicts where software requires incompatible or unavailable versions.

Example:

```text
Application A
    |
    +--> requires Library X version 1

Application B
    |
    +--> requires Library X version 2
```

Modern package-management systems reduce these problems significantly, but conflicts can still occur, particularly with:

* Third-party repositories.
* Manual installations.
* Mixed distribution releases.
* Unsupported packages.
* Version pinning.
* Locally built software.

---

# 82. Repository Failure

Suppose:

```bash
sudo apt update
```

or:

```bash
sudo dnf makecache
```

fails.

Possible causes include:

* No internet connectivity.
* DNS failure.
* Repository server unavailable.
* Incorrect repository URL.
* Unsupported distribution release.
* TLS/certificate problems.
* Proxy configuration.
* Authentication failure for private repositories.
* Invalid repository signing configuration.
* Firewall restrictions.

Do not immediately edit repository files.

Determine which layer failed.

---

# 83. Repository Troubleshooting Workflow

Use:

```text
Package manager reports repository failure
              |
              v
Read exact repository error
              |
              v
Is basic network connectivity working?
              |
              v
Is DNS working?
              |
              v
Can the repository hostname be resolved?
              |
              v
Is HTTPS/TLS working?
              |
              v
Is repository configuration correct?
              |
              v
Is the repository supported and available?
              |
              v
Is authentication/signing configuration valid?
```

This uses troubleshooting concepts from earlier Linux parts.

---

# 84. Example: DNS Failure

Suppose APT says it cannot resolve a repository hostname.

Do not reinstall APT.

Check:

```bash
getent hosts archive.ubuntu.com
```

If name resolution fails, investigate DNS.

You can also inspect:

```bash
cat /etc/resolv.conf
```

depending on your system's resolver configuration.

The root problem may be networking, not package management.

---

# 85. Example: Repository Returns 404

A repository error such as:

```text
404 Not Found
```

can indicate:

* Incorrect repository path.
* Repository moved.
* Distribution release no longer supported there.
* Incorrect repository configuration.

Read the exact URL and repository entry.

Do not respond by disabling security verification.

---

# 86. Repository Trust and Package Signing

Linux repositories commonly use cryptographic signatures to help verify repository metadata and packages originate through trusted repository mechanisms.

If the package manager reports a signature or trust error, do not "fix" it by disabling verification.

Investigate:

* Whether the repository is legitimate.
* Whether its signing key changed.
* Whether the repository setup instructions are current.
* Whether your system clock is correct.
* Whether the repository configuration was tampered with.

Trust errors are security signals.

---

# 87. Package Locks

Package managers modify important system databases.

Allowing multiple package-management processes to modify those databases simultaneously could corrupt state.

Therefore package-management tools use locking mechanisms.

You may see errors involving:

```text
Could not get lock
```

or:

```text
Another app is currently holding the yum lock
```

depending on system and tool versions.

---

# 88. Why Package Locks Happen

Possible causes:

* Another package installation is running.
* Automatic updates are running.
* Another administrator is using the package manager.
* A background package service is active.
* A previous process crashed and left confusing state.

The beginner mistake is immediately deleting lock files.

Do not do that.

First determine whether another package-management process is legitimately running.

---

# 89. Investigating APT Lock Problems

Check processes:

```bash
ps aux | grep -E 'apt|dpkg'
```

A cleaner search may use:

```bash
pgrep -a apt
```

and:

```bash
pgrep -a dpkg
```

Ask:

```text
Is another package operation actually running?
```

If yes, wait for it to finish unless you have a specific reason and understand the consequences of stopping it.

---

# 90. Do Not Blindly Delete Lock Files

You may find internet tutorials telling you to immediately run commands that remove files under locations such as:

```text
/var/lib/dpkg/
```

or:

```text
/var/lib/apt/
```

Do not blindly follow them.

A lock usually exists to protect package database integrity.

Deleting a lock while another package manager is actively modifying the system can create a much larger problem.

---

# 91. Interrupted Debian Package Installation

Suppose an installation is interrupted.

You may later receive messages indicating packages are not fully configured.

One important recovery command is:

```bash
sudo dpkg --configure -a
```

This tells dpkg to configure unpacked packages that still need configuration.

Do not run recovery commands randomly.

Use them when the package manager's error indicates incomplete configuration or when your investigation supports that diagnosis.

---

# 92. Repairing Dependency State With APT

You may encounter:

```bash
sudo apt --fix-broken install
```

This asks APT to attempt to correct broken dependency relationships.

Again:

```text
--fix-broken
```

is not a universal magic command.

First read the package-manager output.

Understand:

* Which packages are broken.
* What APT proposes to install.
* What APT proposes to remove.

Then decide.

---

# 93. Broken Installation Scenario

Suppose:

```bash
sudo dpkg -i application.deb
```

reports missing dependencies.

You now have a partially installed or unconfigured package state.

One possible recovery workflow is:

```bash
sudo apt --fix-broken install
```

because APT can attempt dependency resolution using configured repositories.

A better prevention strategy is often:

```bash
sudo apt install ./application.deb
```

when installing a local Debian package.

---

# 94. RPM Dependency Failure

Suppose you try:

```bash
sudo rpm -ivh application.rpm
```

and RPM reports missing dependencies.

Do not search for each dependency on random download sites.

Instead, if appropriate, use:

```bash
sudo dnf install ./application.rpm
```

DNF can attempt to resolve dependencies using configured repositories.

---

# 95. Package Database Problems

Package-management systems maintain local state.

Problems can occur after:

* Interrupted upgrades.
* Disk-full events.
* Forced termination.
* Filesystem corruption.
* Manual deletion of package files.
* Incorrect repository mixing.

Treat package-database errors carefully.

Do not immediately delete databases or rebuild them from random instructions.

First:

1. Read the exact error.
2. Back up important systems.
3. Determine the package-management family.
4. Determine whether another operation is running.
5. Check disk space.
6. Check filesystem health if indicated.
7. Use distribution-supported recovery procedures.

---

# 96. Disk Space and Package Failures

Installing packages requires storage.

Check:

```bash
df -h
```

If:

```text
/
```

or:

```text
/var
```

is full, package operations may fail.

Remember that package managers may need space for:

* Downloaded packages.
* Temporary files.
* Extracted files.
* Package database updates.
* New kernels.
* Logs.

A "package installation failure" can actually be a disk-space problem.

---

# 97. Checking Inodes

A filesystem can also run out of inodes even when some byte capacity remains.

Check:

```bash
df -i
```

If inode usage reaches 100 percent, creating new files may fail.

Again, the package manager may only be the component exposing the underlying filesystem problem.

---

# 98. Safe Software Installation

Before installing software on an important Linux system, ask:

```text
What am I installing?

Why do I need it?

Which repository provides it?

Is the repository trusted?

Which version will be installed?

What dependencies will be added?

Will services be started?

Will configuration files change?

Will ports be opened?

Will the installation require a reboot?

Could it affect existing applications?

How will I verify the installation?

How will I remove or roll it back if necessary?
```

This is Linux administration, not merely command execution.

---

# 99. Prefer Distribution Package Managers When Appropriate

For system software, prefer established package-management mechanisms where practical.

Instead of randomly downloading:

```text
some-program.tar.gz
```

or:

```text
install.sh
```

first check whether an appropriate package exists in:

* Your distribution repository.
* An official vendor repository.
* Your organization's approved repository.

This gives you better:

* Version tracking.
* Dependency handling.
* Updates.
* Removal.
* Auditing.

---

# 100. The Risk of `curl | bash`

You will frequently see installation instructions like:

```bash
curl https://example.com/install.sh | bash
```

This downloads a script and immediately executes it.

The shell begins executing data arriving from the network.

That removes an important inspection step.

A safer general approach is:

```bash
curl -O https://example.com/install.sh
```

Then inspect:

```bash
less install.sh
```

Verify its source and integrity using the vendor's documented verification method when available.

Only then execute it if you trust it and understand what it does.

---

# 101. Why Piping to `sudo bash` Is Even More Sensitive

Consider:

```bash
curl https://example.com/install.sh | sudo bash
```

You are effectively saying:

```text
Download remote instructions
        |
        v
Immediately execute them
        |
        v
With root privileges
```

A malicious or compromised script could potentially:

* Create users.
* Install persistence.
* Read sensitive files.
* Modify SSH configuration.
* Add repositories.
* Install services.
* Change firewall rules.
* Exfiltrate secrets.
* Replace binaries.

Do not treat installation commands as harmless simply because they appear in a tutorial.

---

# 102. Questions to Ask Before Running an Installation Script

Ask:

1. Is this the official vendor?
2. Is HTTPS being used?
3. Can I inspect the script first?
4. Is a checksum or signature available?
5. Why does it require root?
6. Which files will it modify?
7. Does it add repositories?
8. Does it create users?
9. Does it install services?
10. Does it start those services?
11. Does it open network ports?
12. How is it removed?
13. Is there a package-manager-based installation method instead?

This habit becomes increasingly important in DevOps and cloud environments.

---

# 103. Third-Party Repository Risks

Adding a repository is not simply adding a download location.

You are extending the trust boundary of your package-management system.

A poorly maintained or compromised repository could distribute malicious or incompatible packages.

Before adding one:

* Confirm it is the official vendor repository.
* Confirm your distribution and release are supported.
* Follow current vendor instructions.
* Understand signing-key configuration.
* Avoid obsolete tutorials.
* Know how to remove the repository later.

---

# 104. Mixing Distribution Releases

Do not casually configure repositories from another Linux release.

For example, mixing packages designed for different distribution releases can create:

* Dependency conflicts.
* Library incompatibility.
* Broken upgrades.
* Unsupported system state.

Package compatibility involves more than:

```text
Both systems are Linux.
```

The distribution version matters.

---

# 105. Manual Installation vs Package Installation

Software can reach a Linux system through many methods:

```text
OS package manager
Source compilation
Vendor installer
Shell script
Language package manager
Container image
Standalone binary
Configuration-management system
```

Examples of language-specific package managers include:

```text
pip
npm
gem
cargo
```

These solve different problems from APT or DNF.

Do not assume:

```text
pip install
```

and:

```text
apt install
```

manage the same software scope in the same way.

---

# 106. System Packages vs Application Dependencies

Suppose a Python application requires:

```text
requests
```

You might install Python application dependencies using a Python environment and `pip`.

Meanwhile:

```text
python3
openssl
system libraries
```

may be managed by the operating-system package manager.

Conceptually:

```text
Operating system
     |
     +--> APT/DNF
     |      |
     |      +--> Python runtime
     |      +--> system libraries
     |
     v
Application environment
     |
     +--> pip
            |
            +--> Python dependencies
```

Understanding boundaries helps prevent dependency conflicts.

---

# 107. Production Update Safety

On your personal lab machine, you may casually run:

```bash
sudo apt upgrade
```

On a production server, think first.

An update might:

* Restart a service.
* Change a shared library.
* Install a new kernel.
* Modify package-managed configuration.
* Affect application compatibility.
* Require rebooting.
* Change security behavior.

A production update process may involve:

```text
Review updates
      |
      v
Assess impact
      |
      v
Test
      |
      v
Back up / confirm recovery
      |
      v
Schedule change
      |
      v
Apply
      |
      v
Verify services
      |
      v
Verify application
      |
      v
Monitor
```

---

# 108. Security Updates

Software contains vulnerabilities.

Linux distributions publish package updates containing security fixes.

Ignoring updates indefinitely can leave systems exposed.

But the answer is not necessarily:

```text
Automatically upgrade everything immediately on every production server.
```

Organizations usually balance:

* Security urgency.
* Testing.
* Compatibility.
* Availability.
* Change management.

Critical vulnerabilities may require accelerated patching.

---

# 109. Kernel Updates

The Linux kernel itself is commonly installed and updated through the package-management system.

After a kernel package update, the currently running kernel does not magically become the newly installed kernel.

Check:

```bash
uname -r
```

This shows the currently running kernel release.

A reboot is commonly required to begin running a newly installed kernel.

This connects package management with the boot process from Part 9.

---

# 110. Package Cache

Package managers may maintain downloaded metadata and package caches.

On Debian-based systems, downloaded package archives may involve:

```text
/var/cache/apt/
```

On DNF systems, cache data is maintained under locations such as:

```text
/var/cache/dnf/
```

Caches can improve performance but also consume disk space.

Do not delete arbitrary package-manager directories.

Use package-manager-supported cleanup commands.

---

# 111. APT Cache Cleanup

A command you may encounter is:

```bash
sudo apt clean
```

This removes downloaded package archive files from APT's cache.

Another command is:

```bash
sudo apt autoclean
```

Its cleanup behavior differs by focusing on package files that can no longer be downloaded under APT's criteria.

Do not confuse:

```text
clean
```

with:

```text
autoremove
```

`clean` deals with cached package files.

`autoremove` deals with installed packages considered no longer required.

---

# 112. DNF Cache

You can inspect DNF cache behavior with commands such as:

```bash
dnf clean all
```

This removes cached repository data according to DNF's cleanup behavior.

Metadata can later be rebuilt.

A common command is:

```bash
sudo dnf makecache
```

which downloads and caches repository metadata.

Do not repeatedly clear caches as a default troubleshooting ritual. First understand the actual problem.

---

# 113. Configuration Files During Upgrades

Package upgrades can encounter configuration files that you have modified.

Package-management systems try to avoid blindly destroying administrator configuration, but behavior depends on the packaging system and package.

You may be asked whether to:

* Keep your current configuration.
* Install the maintainer's version.
* Review differences.

Read prompts carefully.

Do not automatically accept replacement configuration on important systems.

---

# 114. Package Installation Scripts

Packages can contain scripts that execute during package lifecycle events.

These can perform tasks such as:

* Create users.
* Create directories.
* Update caches.
* Register services.
* Reload daemons.
* Migrate data.
* Adjust permissions.

This explains why installing a package is more powerful than merely copying a binary into `/usr/bin`.

Package installation changes system state.

---

# 115. Packages Can Start Services

Depending on the distribution and package, installing server software may create and potentially start or enable a service automatically.

After installing a service package, always check.

Example:

```bash
systemctl status nginx
```

Then:

```bash
systemctl is-enabled nginx
```

Then:

```bash
sudo ss -ltnp
```

Do not assume:

```text
I installed it, therefore nothing is listening.
```

Verify.

---

# 116. Packages and Firewall Rules

Installing a network service does not automatically mean remote clients can reach it.

The application may be:

```text
Installed
Running
Listening
```

while remote access is blocked by:

* Host firewall.
* Cloud security group.
* Network ACL.
* Routing.
* Load balancer configuration.
* Binding only to localhost.

Package installation and network accessibility are separate layers.

---

# 117. Troubleshooting Model

When software installation fails, use:

```text
Observe
   |
   v
Read exact error
   |
   v
Classify failure
   |
   +--> repository?
   +--> network?
   +--> DNS?
   +--> dependency?
   +--> package lock?
   +--> disk?
   +--> architecture?
   +--> signature/trust?
   +--> package database?
   |
   v
Gather evidence
   |
   v
Form hypothesis
   |
   v
Test
   |
   v
Fix smallest cause
   |
   v
Retry safely
   |
   v
Verify installation
```

Do not immediately copy the first command from a search result.

---

# 118. Problem: Package Not Found

You run:

```bash
sudo apt install example
```

and receive an error indicating the package cannot be located.

Possible causes:

* Package name is wrong.
* Package lists are stale.
* Required repository is not configured.
* Package is unavailable for your distribution release.
* Package has another name.
* Repository update failed.

Investigate:

```bash
apt search example
```

Then:

```bash
sudo apt update
```

Read errors.

Then inspect repository configuration if necessary.

---

# 119. Problem: Command Not Found After Installation

You install a package but:

```bash
example
```

returns:

```text
command not found
```

Investigate:

```bash
dpkg -L PACKAGE
```

or:

```bash
rpm -ql PACKAGE
```

Look for executable files.

Then:

```bash
echo "$PATH"
```

and:

```bash
command -v EXPECTED_COMMAND
```

Do not immediately reinstall the package.

---

# 120. Problem: Wrong Package Architecture

You downloaded a package and installation reports an architecture mismatch.

Check:

```bash
uname -m
```

Debian:

```bash
dpkg --print-architecture
```

Inspect the downloaded package.

For `.deb`:

```bash
dpkg-deb -I package.deb
```

For `.rpm`:

```bash
rpm -qip package.rpm
```

Compare architectures.

---

# 121. Problem: Repository Unreachable

First read the exact hostname in the error.

Check name resolution:

```bash
getent hosts REPOSITORY_HOSTNAME
```

Check general network state as appropriate.

If HTTPS access is involved, you may test with:

```bash
curl -I https://REPOSITORY_HOSTNAME
```

when the repository supports an appropriate HTTP endpoint.

Then inspect repository configuration.

Do not disable TLS verification just to make the error disappear.

---

# 122. Problem: Package Manager Is Locked

You see:

```text
Could not get lock
```

Investigate:

```bash
pgrep -a apt
pgrep -a dpkg
```

or on an RPM system:

```bash
pgrep -a dnf
pgrep -a yum
rpm-related checks as appropriate
```

If another legitimate transaction is running:

```text
Wait.
```

Do not kill it simply because you want your installation to start immediately.

---

# 123. Problem: Disk Full During Installation

Check:

```bash
df -h
```

Then:

```bash
df -i
```

Investigate large directories if necessary.

Do not randomly delete files from:

```text
/usr
/var/lib
/boot
```

Those directories may contain critical package, application, boot, or system state.

Identify safe cleanup targets.

---

# 124. Problem: Package Installed but Service Failed

You successfully run:

```bash
sudo apt install nginx
```

but:

```bash
systemctl status nginx
```

shows:

```text
failed
```

Package installation succeeded.

Service startup failed.

Those are different problems.

Now use Part 9:

```bash
systemctl status nginx
```

```bash
journalctl -u nginx -n 100 --no-pager
```

```bash
sudo nginx -t
```

```bash
sudo ss -ltnp | grep ':80'
```

Package management got the software onto the machine.

Service troubleshooting determines why it cannot operate.

---

# 125. Problem: Software Version Is Older Than Vendor Website

Suppose the vendor website advertises:

```text
Version 5
```

but your distribution repository provides:

```text
Version 4
```

This does not automatically mean your repository is broken.

Linux distributions often package versions according to their:

* Release lifecycle.
* Stability policies.
* Security-maintenance strategy.
* Compatibility requirements.

Some distributions backport security fixes without changing the upstream version in the way beginners expect.

Do not replace distribution packages merely because a vendor website displays a larger version number.

---

# 126. Package Management in DevOps

Package management appears everywhere in DevOps.

Example VM provisioning:

```bash
sudo apt update
sudo apt install nginx
```

An automation system might perform the equivalent operation declaratively.

A cloud-init configuration might install packages during VM creation.

A Dockerfile might contain:

```dockerfile
RUN apt-get update && apt-get install -y curl
```

A CI runner may install build dependencies.

A Kubernetes node image may contain carefully controlled package versions.

Understanding Linux package management therefore supports:

* Cloud engineering.
* CI/CD.
* Docker.
* Kubernetes.
* Configuration management.
* Server hardening.
* Vulnerability remediation.

---

# 127. Why Automation Often Uses `apt-get`

You may notice interactive tutorials often use:

```bash
apt
```

while scripts and Dockerfiles frequently use:

```bash
apt-get
```

`apt` is designed as a convenient interactive command interface.

Traditional scripting commonly uses lower-level/stabler command interfaces such as:

```bash
apt-get
```

For beginner interactive administration, `apt` is convenient.

Later, when writing automation, understand the tooling expectations of that environment rather than blindly replacing every `apt` with `apt-get` or vice versa.

---

# 128. Noninteractive Installation

You may see:

```bash
sudo apt-get install -y nginx
```

The:

```text
-y
```

automatically answers yes to prompts that the package manager can answer that way.

This is useful in controlled automation.

It can be dangerous during interactive troubleshooting because it removes your opportunity to review the proposed transaction.

While learning, prefer reading the transaction before approving it.

---

# 129. Never Add `-y` Automatically

Compare:

```bash
sudo apt install nginx
```

with:

```bash
sudo apt install -y nginx
```

The second can proceed without your manual confirmation.

In automation, that may be intentional.

On an unfamiliar production server, automatic confirmation can hide important proposed changes.

Understand the transaction first.

---

# 130. Package Management Lab

We will now combine the concepts into a practical lab.

Use a disposable Linux VM or cloud server.

Do not perform intentional package failures on a production system.

Choose the section matching your distribution.

---

# 131. Lab Step 1: Identify the Distribution

Run:

```bash
cat /etc/os-release
```

Record:

```text
NAME
VERSION
ID
ID_LIKE
```

Then:

```bash
uname -m
```

Determine:

* Debian family?
* Red Hat family?
* CPU architecture?

---

# 132. Lab Step 2: Identify Available Package Tools

Run:

```bash
command -v apt
command -v dpkg
command -v dnf
command -v yum
command -v rpm
```

Some commands may return nothing.

That is expected.

Write down which package-management stack your machine uses.

---

# 133. Debian Lab: Refresh Metadata

If using Debian/Ubuntu:

```bash
sudo apt update
```

Read the output.

Do not immediately move on.

Identify:

* Which repositories were contacted.
* Whether any failed.
* Whether updates are available.

Explain in your own words:

```text
What changed when I ran apt update?
```

Correct concept:

```text
Local repository/package metadata was refreshed.
```

---

# 134. Debian Lab: Search for a Package

Search:

```bash
apt search tree
```

Then inspect:

```bash
apt show tree
```

Identify:

* Package name.
* Version.
* Architecture.
* Description.
* Dependencies if shown.
* Download or installed size information where available.

---

# 135. Debian Lab: Install `tree`

Run:

```bash
sudo apt install tree
```

Before confirming, inspect the transaction.

Then verify:

```bash
tree --version
```

Find:

```bash
command -v tree
```

Then:

```bash
dpkg -s tree
```

---

# 136. Debian Lab: Inspect Package Files

Run:

```bash
dpkg -L tree
```

Find the executable.

Then:

```bash
dpkg -S "$(command -v tree)"
```

Confirm which package owns the binary.

You have now connected:

```text
Package database
       |
       v
Installed package
       |
       v
Installed files
       |
       v
Executable
```

---

# 137. Debian Lab: Inspect Version Information

Run:

```bash
tree --version
```

Then:

```bash
dpkg -s tree | grep '^Version:'
```

Compare the application-reported version and package version.

Explain why their formatting may differ.

---

# 138. Debian Lab: Remove the Package

Run:

```bash
sudo apt remove tree
```

Read the transaction before confirming.

Then:

```bash
command -v tree
```

and:

```bash
dpkg -s tree
```

Observe the resulting package state.

---

# 139. Debian Lab: Reinstall

Run:

```bash
sudo apt install tree
```

Verify again:

```bash
tree --version
```

This demonstrates predictable package installation and removal.

---

# 140. Debian Lab: Installed Package Search

Run:

```bash
apt list --installed 2>/dev/null | grep -E 'curl|git|tree'
```

Then:

```bash
dpkg -l | grep -E 'curl|git|tree'
```

Compare the output.

---

# 141. Debian Lab: Inspect Repository Configuration

Run:

```bash
ls -l /etc/apt/
```

Then:

```bash
ls -l /etc/apt/sources.list.d/
```

If `/etc/apt/sources.list` exists:

```bash
cat /etc/apt/sources.list
```

Do not edit anything.

Your goal is to understand:

```text
Where does APT learn about repositories?
```

---

# 142. Debian Lab: Package Cache

Inspect:

```bash
du -sh /var/cache/apt/ 2>/dev/null
```

Do not delete files manually.

Learn that package management creates local cache and state.

---

# 143. Red Hat Lab: Inspect Repositories

If using a DNF-based system:

```bash
dnf repolist
```

This lists enabled repositories.

Then:

```bash
ls -l /etc/yum.repos.d/
```

Do not modify them.

Identify which repositories your machine trusts for package installation.

---

# 144. Red Hat Lab: Search for `tree`

Run:

```bash
dnf search tree
```

Then:

```bash
dnf info tree
```

Identify:

* Name.
* Version.
* Release.
* Architecture.
* Repository.
* Description.

---

# 145. Red Hat Lab: Install `tree`

Run:

```bash
sudo dnf install tree
```

Read the transaction.

Then verify:

```bash
tree --version
```

Find:

```bash
command -v tree
```

Query:

```bash
rpm -q tree
```

---

# 146. Red Hat Lab: Inspect Package Information

Run:

```bash
rpm -qi tree
```

Then:

```bash
rpm -ql tree
```

Find the executable.

Then:

```bash
rpm -qf "$(command -v tree)"
```

Confirm the package ownership.

---

# 147. Red Hat Lab: Remove and Reinstall

Remove:

```bash
sudo dnf remove tree
```

Verify:

```bash
command -v tree
```

Then reinstall:

```bash
sudo dnf install tree
```

Verify:

```bash
tree --version
rpm -q tree
```

---

# 148. Lab: Check Disk Space Before Installation

Run:

```bash
df -h
```

Then:

```bash
df -i
```

Identify the filesystem containing:

```text
/
```

Ask:

* Is there sufficient space?
* Is inode usage healthy?

This should become part of your troubleshooting thinking when installations fail unexpectedly.

---

# 149. Lab: Install a Service Package

On a disposable system, install NGINX using the appropriate package manager.

Debian:

```bash
sudo apt install nginx
```

Red Hat family:

```bash
sudo dnf install nginx
```

Availability and repository configuration can vary.

After installation:

```bash
nginx -v
```

Then:

```bash
systemctl status nginx
```

Then:

```bash
systemctl is-enabled nginx
```

Then:

```bash
sudo ss -ltnp | grep ':80'
```

Then:

```bash
curl -I http://127.0.0.1
```

You are combining Parts 8, 9, and 10.

---

# 150. Lab: Find Which Package Installed NGINX

Debian:

```bash
dpkg -S "$(command -v nginx)"
```

RPM:

```bash
rpm -qf "$(command -v nginx)"
```

Then list package contents.

Debian:

```bash
dpkg -L nginx
```

RPM:

```bash
rpm -ql nginx
```

Find:

* Executable.
* Configuration.
* Documentation.
* Service-related files where applicable.

---

# 151. Lab: Trace Package to Running Process

Start with the package:

```text
nginx package
```

Find binary:

```bash
command -v nginx
```

Check service:

```bash
systemctl status nginx
```

Find process:

```bash
pgrep -a nginx
```

Find listening port:

```bash
sudo ss -ltnp | grep ':80'
```

Test:

```bash
curl -I http://127.0.0.1
```

You have now followed:

```text
Repository
    |
    v
Package manager
    |
    v
Package
    |
    v
Files
    |
    v
Binary
    |
    v
systemd service
    |
    v
Process
    |
    v
Listening socket
    |
    v
HTTP response
```

This is the bigger Linux picture.

---

# 152. Safe Failure Lab: Package Does Not Exist

On Debian:

```bash
sudo apt install veriqta-package-that-does-not-exist
```

On DNF:

```bash
sudo dnf install veriqta-package-that-does-not-exist
```

Observe the error.

Do not search the internet for a random package.

Instead:

```bash
apt search veriqta
```

or:

```bash
dnf search veriqta
```

Learn to distinguish:

```text
Package unavailable
```

from:

```text
Package manager broken.
```

---

# 153. Safe Failure Lab: Wrong Command Name

Install `tree`.

Then intentionally type:

```bash
tre
```

You should receive something similar to:

```text
command not found
```

Now troubleshoot:

```bash
command -v tree
```

Then:

```bash
tree --version
```

The package was fine.

The problem was:

```text
Typing mistake.
```

Always investigate the simplest explanation first.

---

# 154. Safe Failure Lab: Wrong Path to Local Package

Imagine you have:

```text
~/Downloads/example.deb
```

but from another directory you run:

```bash
sudo apt install ./example.deb
```

APT cannot find the file in your current directory.

Check:

```bash
pwd
```

Then:

```bash
ls -l
```

Then:

```bash
ls -l ~/Downloads/example.deb
```

The issue is filesystem navigation, not package management.

This connects Part 2 with Part 10.

---

# 155. Troubleshooting Scenario: `apt update` Fails

Use this workflow:

```text
apt update fails
      |
      v
Read exact error
      |
      +--> DNS?
      |
      +--> network?
      |
      +--> repository URL?
      |
      +--> unsupported release?
      |
      +--> signing problem?
      |
      +--> proxy?
      |
      v
Investigate relevant layer
      |
      v
Fix root cause
      |
      v
Run apt update again
      |
      v
Confirm clean result
```

Do not begin by deleting package state.

---

# 156. Troubleshooting Scenario: Installation Hangs on Lock

Suppose:

```text
Could not get lock
```

Check:

```bash
pgrep -a apt
pgrep -a dpkg
```

If another package process is running:

```text
Wait for it.
```

If it appears stuck, investigate:

* How long has it run?
* Is it using CPU?
* Is it performing package configuration?
* Is an automatic update active?
* Is another administrator working?

Do not use `kill -9` as your first response.

---

# 157. Troubleshooting Scenario: NGINX Package Installs but Web Server Fails

Package manager says installation completed.

But:

```bash
systemctl status nginx
```

says:

```text
failed
```

Run:

```bash
journalctl -u nginx -n 100 --no-pager
```

Suppose:

```text
Address already in use
```

Then:

```bash
sudo ss -ltnp | grep ':80'
```

You discover another process already owns port 80.

The package-management system succeeded.

The service failed because of a runtime port conflict.

Correct diagnosis:

```text
Installation
    |
    v
Succeeded

Service startup
    |
    v
Failed

Cause
    |
    v
Port conflict
```

---

# 158. Troubleshooting Scenario: Installation Fails Because Disk Is Full

You see errors involving:

```text
No space left on device
```

Check:

```bash
df -h
```

Then:

```bash
df -i
```

Determine whether:

* Storage capacity is exhausted.
* Inodes are exhausted.

Then investigate disk usage safely.

Do not keep retrying the installation.

---

# 159. Troubleshooting Scenario: Local `.deb` Has Missing Dependencies

You run:

```bash
sudo dpkg -i application.deb
```

and dependency errors appear.

Reason:

```text
dpkg understands the package file,
but missing dependencies still need resolution.
```

Investigate package state.

Where appropriate:

```bash
sudo apt --fix-broken install
```

may repair dependency state.

For future local installations, consider:

```bash
sudo apt install ./application.deb
```

when appropriate.

---

# 160. Troubleshooting Scenario: Local RPM Has Missing Dependencies

You run:

```bash
sudo rpm -ivh application.rpm
```

and receive missing dependency errors.

Instead of downloading each missing RPM manually from unrelated sites, use the high-level manager where appropriate:

```bash
sudo dnf install ./application.rpm
```

This allows DNF to resolve dependencies through configured repositories.

---

# 161. Troubleshooting Scenario: Wrong Repository

Suppose a tutorial tells you to add a repository for a different distribution release.

Afterward, dependency problems begin.

Ask:

```text
What repository did I add?

Which distribution release is this machine?

Does the vendor officially support it?

Which packages came from that repository?
```

Do not keep adding more repositories to solve conflicts created by the first one.

Every repository changes your package-resolution environment.

---

# 162. Troubleshooting Scenario: Package Updated but Old Program Still Running

Suppose a package containing a service is upgraded.

The package files on disk may now be new, but an already running process may still be using code or libraries loaded before the update.

Depending on the package and update mechanism, a service restart may be required.

Investigate:

```bash
systemctl status SERVICE
```

and follow distribution/application guidance.

This is one reason update verification includes service state, not just package version.

---

# 163. Package Installation Verification Checklist

After installing software, verify at the appropriate layers.

For a command-line tool:

```text
Package installed?
      |
      v
Binary exists?
      |
      v
Correct version?
      |
      v
Command works?
```

For a service:

```text
Package installed?
      |
      v
Binary exists?
      |
      v
Configuration valid?
      |
      v
Service active?
      |
      v
Process running?
      |
      v
Port listening?
      |
      v
Application responds?
      |
      v
Logs clean?
```

---

# 164. Safe Removal Checklist

Before removing software:

```text
Why am I removing it?

Is another package dependent on it?

Does it contain important configuration?

Does it store application data?

Will the service stop?

Will users be affected?

Do I need a backup?

Am I removing or purging?

What additional packages does the package manager propose removing?
```

Read the transaction.

Then verify afterward.

---

# 165. Commands You Should Now Recognize

Debian-family:

```bash
sudo apt update
sudo apt upgrade
sudo apt install PACKAGE
sudo apt remove PACKAGE
sudo apt purge PACKAGE
sudo apt autoremove
apt search PACKAGE
apt show PACKAGE
apt list --installed
dpkg -l
dpkg -s PACKAGE
dpkg -L PACKAGE
dpkg -S FILE
dpkg-deb -I package.deb
dpkg-deb -c package.deb
sudo dpkg -i package.deb
sudo apt install ./package.deb
sudo dpkg --configure -a
sudo apt --fix-broken install
```

RPM-family:

```bash
sudo dnf install PACKAGE
sudo dnf remove PACKAGE
sudo dnf upgrade
dnf check-update
dnf search PACKAGE
dnf info PACKAGE
dnf list installed
dnf repolist
rpm -qa
rpm -q PACKAGE
rpm -qi PACKAGE
rpm -ql PACKAGE
rpm -qf FILE
rpm -qip package.rpm
rpm -qlp package.rpm
sudo rpm -ivh package.rpm
sudo dnf install ./package.rpm
```

---

# 166. What You Should Now Understand

Before moving to Part 11, you should confidently understand:

* What a Linux software package is.
* Why Linux distributions package software.
* What package contents are.
* What package metadata is.
* What a repository is.
* Why repositories exist.
* The difference between official, vendor, third-party, and internal repositories.
* Why repository trust matters.
* What dependencies are.
* Why dependencies exist.
* What dependency resolution means.
* What a dependency graph is.
* What package managers do.
* The difference between high-level and low-level package tools.
* Debian-family package management.
* What `.deb` means.
* What APT is.
* What dpkg is.
* How APT and dpkg relate.
* What `apt update` does.
* Why `apt update` does not upgrade installed packages.
* What `apt upgrade` does.
* How to install packages.
* How to verify installation.
* How to install multiple packages.
* How `apt remove` works.
* How `apt purge` differs from remove.
* What `apt autoremove` does.
* Why autoremove transactions should be reviewed.
* How to search packages.
* How to inspect package metadata.
* How to list installed packages.
* How to query package state with dpkg.
* How to list package files.
* How to find which package owns a file.
* What a `.deb` file is.
* How to inspect `.deb` files.
* How to install local `.deb` files.
* Why APT can be preferable to direct `dpkg -i` for local packages.
* What package architecture means.
* Where APT repository configuration commonly lives.
* Red Hat-family package management.
* What `.rpm` means.
* What RPM is.
* What DNF is.
* What YUM is.
* How DNF and RPM relate.
* How to install packages with DNF.
* How to remove packages.
* How to upgrade software.
* How to check for updates.
* How to search packages.
* How to query installed RPM packages.
* How to inspect RPM package information.
* How to list files belonging to an RPM package.
* How to determine which RPM owns a file.
* How to inspect local RPM files.
* Why DNF is generally preferable for dependency-aware local RPM installation.
* How to check application versions.
* How package versions can differ in formatting from upstream versions.
* How to determine whether software is installed.
* Why a binary can exist without belonging to an OS package.
* Why an installed package may not provide the command name you expected.
* What dependency failures look like.
* Why random dependency downloads are dangerous.
* What repository failures are.
* How networking and DNS can cause package-manager failures.
* Why package signature errors should not simply be bypassed.
* What package locks are.
* Why package locks exist.
* Why you should not blindly delete lock files.
* How interrupted Debian installations can be recovered.
* What `dpkg --configure -a` does.
* What `apt --fix-broken install` does.
* Why those commands are not universal magic fixes.
* How disk-space problems can break package installation.
* How inode exhaustion can affect installation.
* Why software installation is a security-sensitive operation.
* Why random installation scripts are risky.
* Why `curl | bash` deserves scrutiny.
* Why `curl | sudo bash` is especially sensitive.
* What to inspect before trusting an installation script.
* Why third-party repositories expand your trust boundary.
* Why mixing distribution releases can break package management.
* The difference between OS package managers and language package managers.
* Why production upgrades require planning.
* Why security updates matter.
* Why kernel updates may require rebooting.
* What package caches are.
* Why packages can run installation scripts.
* Why installing a package can create or start services.
* Why package installation does not automatically mean network accessibility.
* How to troubleshoot package-management problems systematically.
* How package management connects with filesystems, permissions, processes, services, networking, and troubleshooting.

---

# 167. Hands-On Practice

Complete these tasks on a disposable Linux system.

1. Display `/etc/os-release`.
2. Identify your distribution.
3. Identify the distribution family.
4. Check your CPU architecture.
5. Determine whether APT or DNF is available.
6. Identify the low-level package tool.
7. Explain the difference between the high-level and low-level tools.
8. List configured repositories.
9. Refresh package metadata.
10. Explain what changed.
11. Search for `tree`.
12. Display its package information.
13. Identify its available version.
14. Install it.
15. Read the installation transaction before confirming.
16. Verify the binary exists.
17. Check the application version.
18. Check the package version.
19. List the files belonging to the package.
20. Find which package owns the `tree` binary.
21. Remove `tree`.
22. Verify its command is unavailable.
23. Reinstall it.
24. List installed packages.
25. Search the installed-package list for `tree`.
26. Check disk usage.
27. Check inode usage.
28. Inspect package cache locations.
29. Identify repository configuration files.
30. Do not modify them.
31. Search for NGINX.
32. Inspect NGINX package metadata.
33. Install NGINX on a disposable lab.
34. Check its package version.
35. Check its application version.
36. Find its executable.
37. Determine which package owns that executable.
38. List package files.
39. Check the systemd service.
40. Determine whether it is active.
41. Determine whether it is enabled.
42. Find its processes.
43. Check its listening port.
44. Test it locally with `curl`.
45. Check its logs.
46. Explain the path from repository to running service.
47. Attempt to install a deliberately nonexistent package.
48. Read the error.
49. Explain why the package manager itself is not necessarily broken.
50. Explain what you would check if `apt update` or `dnf` repository access failed.
51. Explain how DNS could affect package management.
52. Explain what a package lock protects.
53. Explain why deleting locks blindly is dangerous.
54. Explain `dpkg --configure -a` if using Debian.
55. Explain `apt --fix-broken install` if using Debian.
56. Explain why DNF is preferable to manually resolving RPM dependencies.
57. Explain the difference between remove and purge on Debian.
58. Explain autoremove.
59. Explain why you should inspect an autoremove transaction.
60. Explain the risk of `curl | bash`.
61. Explain the additional risk of `curl | sudo bash`.
62. List ten questions to ask before running an installation script.
63. Explain why adding a repository is a security decision.
64. Explain why mixing repositories from different releases can be dangerous.
65. Explain why a package can install successfully while its service still fails.
66. Explain why an active service may still not be reachable remotely.
67. Explain how disk space can break package installation.
68. Explain how inode exhaustion can break package installation.
69. Explain why a kernel update may require rebooting.
70. Describe your complete package-installation verification process.

---

# 168. Beginner Knowledge Check

1. What is a software package?
2. What kinds of files can packages contain?
3. What is package metadata?
4. What is a package repository?
5. Why are repositories useful?
6. What is a third-party repository?
7. Why does repository trust matter?
8. What is a dependency?
9. Why do applications have dependencies?
10. What is dependency resolution?
11. What is a dependency graph?
12. What is a package manager?
13. What is the difference between high-level and low-level package-management tools?
14. What package format does Debian use?
15. What is APT?
16. What is dpkg?
17. How do APT and dpkg relate?
18. What does `apt update` do?
19. Does `apt update` upgrade installed packages?
20. What does `apt upgrade` do?
21. How do you install a package with APT?
22. Why should you read the proposed transaction?
23. How do you install multiple packages?
24. How do you remove a package?
25. What is the difference between remove and purge?
26. Does purge guarantee removal of every file the application ever created?
27. What does autoremove do?
28. Why should autoremove be reviewed?
29. How do you search for an APT package?
30. How do you inspect package information?
31. What is a `.deb` file?
32. How do you inspect `.deb` metadata?
33. How do you list files inside a `.deb`?
34. How do you install a `.deb` with dpkg?
35. What limitation does direct dpkg installation have?
36. Why might `apt install ./package.deb` be preferable?
37. How do you check whether a Debian package is installed?
38. How do you list files installed by a Debian package?
39. How do you find which Debian package owns a file?
40. How do you check Debian architecture?
41. Where is APT repository configuration commonly stored?
42. What package format does the Red Hat family use?
43. What is RPM?
44. What is DNF?
45. What is YUM?
46. How do DNF and RPM relate?
47. How do you install a package with DNF?
48. How do you remove one?
49. How do you check for updates?
50. How do you search DNF packages?
51. How do you list all installed RPM packages?
52. How do you query a particular RPM package?
53. How do you display detailed RPM package information?
54. How do you list files belonging to an installed RPM package?
55. How do you determine which RPM package owns a file?
56. How do you inspect an RPM file before installation?
57. Why can `dnf install ./package.rpm` be preferable to direct RPM installation?
58. What is package architecture?
59. What is `x86_64`?
60. What is `aarch64`?
61. What does `noarch` mean?
62. How do you check the machine architecture?
63. How do you check a program's version?
64. Why might the package version and application version strings differ?
65. Does `command -v program` prove the OS package manager installed it?
66. Can a package be installed without providing the command name you expected?
67. What can cause dependency failures?
68. Why is manually downloading random dependency packages dangerous?
69. What can cause repository failures?
70. How can DNS cause package-management failure?
71. What should you do when you see a repository signature error?
72. What is a package lock?
73. Why do package managers use locks?
74. What should you do before attempting to remove a lock?
75. Why is blindly deleting lock files dangerous?
76. What does `dpkg --configure -a` do?
77. What does `apt --fix-broken install` do?
78. Are those universal repair commands?
79. How can a full filesystem affect package installation?
80. What is inode exhaustion?
81. How do you check disk capacity?
82. How do you check inode usage?
83. Why should you prefer trusted package sources?
84. What is risky about `curl | bash`?
85. Why is `curl | sudo bash` even more sensitive?
86. Why should an installation script be inspected?
87. Why is adding a repository a security decision?
88. Why can mixing distribution releases cause problems?
89. What is the difference between APT/DNF and language package managers such as pip?
90. Why should production upgrades be planned?
91. Why are security updates important?
92. Why might a kernel update require rebooting?
93. What is a package cache?
94. What is the difference between `apt clean` and `apt autoremove`?
95. Can packages execute scripts during installation?
96. Can installing a package create or start a service?
97. Does successful installation prove the application works?
98. Does a running service prove remote clients can reach it?
99. What should you check when package installation fails?
100. What should you verify after installing a network service?

---

# 169. Beginner Knowledge Check Answers

1. A structured bundle used to distribute and manage software.

2. Binaries, libraries, configuration files, documentation, service files, metadata, and package lifecycle scripts.

3. Information describing a package, such as its name, version, architecture, dependencies, and description.

4. A source containing packages and associated metadata.

5. They provide organized, manageable software distribution and updates.

6. A repository maintained outside the distribution's standard repository infrastructure.

7. Packages installed from a repository can change system state and may execute privileged installation actions.

8. Software another package requires to function.

9. Applications reuse libraries and other components rather than implementing everything themselves.

10. Determining and satisfying the packages required by requested software.

11. The relationships between packages and their dependencies.

12. Software used to install, remove, update, search, and query packages.

13. High-level tools handle repositories and dependency resolution. Low-level tools handle package files and local package state more directly.

14. `.deb`.

15. Advanced Package Tool.

16. The lower-level Debian package-management tool.

17. APT provides higher-level repository and dependency management while relying on Debian package infrastructure including dpkg for package operations.

18. It refreshes package metadata from configured repositories.

19. No.

20. It upgrades installed packages according to available repository metadata and the operation's dependency rules.

21.

```bash
sudo apt install PACKAGE
```

22. Because dependencies, removals, replacements, and other system changes may occur.

23.

```bash
sudo apt install PACKAGE1 PACKAGE2 PACKAGE3
```

24.

```bash
sudo apt remove PACKAGE
```

25. Remove normally preserves certain configuration. Purge removes package-managed configuration as well.

26. No.

27. It proposes removal of automatically installed packages that APT considers no longer required.

28. Because the proposed removals may affect your environment.

29.

```bash
apt search PACKAGE
```

30.

```bash
apt show PACKAGE
```

31. A Debian binary package file.

32.

```bash
dpkg-deb -I package.deb
```

33.

```bash
dpkg-deb -c package.deb
```

34.

```bash
sudo dpkg -i package.deb
```

35. It does not provide repository-based dependency resolution in the same way APT does.

36. APT can resolve dependencies using configured repositories.

37.

```bash
dpkg -s PACKAGE
```

38.

```bash
dpkg -L PACKAGE
```

39.

```bash
dpkg -S FILE
```

40.

```bash
dpkg --print-architecture
```

41. Commonly `/etc/apt/sources.list` and files under `/etc/apt/sources.list.d/`.

42. `.rpm`.

43. The RPM package format and low-level package-management system/tool.

44. A high-level package manager for modern RPM-based distributions.

45. An older high-level RPM package manager whose command remains common and may act as a DNF-compatible interface on some modern systems.

46. DNF handles repositories and dependencies while RPM provides lower-level package operations and database functionality.

47.

```bash
sudo dnf install PACKAGE
```

48.

```bash
sudo dnf remove PACKAGE
```

49.

```bash
dnf check-update
```

50.

```bash
dnf search PACKAGE
```

51.

```bash
rpm -qa
```

52.

```bash
rpm -q PACKAGE
```

53.

```bash
rpm -qi PACKAGE
```

54.

```bash
rpm -ql PACKAGE
```

55.

```bash
rpm -qf FILE
```

56.

```bash
rpm -qip package.rpm
```

and:

```bash
rpm -qlp package.rpm
```

57. DNF can resolve dependencies through configured repositories.

58. The hardware/software architecture for which a package is built.

59. A common 64-bit x86 architecture label.

60. A common 64-bit ARM architecture label.

61. A package that is architecture-independent for packaging purposes.

62.

```bash
uname -m
```

63. Often with a command such as:

```bash
PROGRAM --version
```

but syntax depends on the application.

64. Distribution packaging may add release and packaging revision information.

65. No. The program could have been installed manually or by another tool.

66. Yes.

67. Missing packages, incompatible versions, repository problems, mixed repositories, interrupted installations, or unsupported packages.

68. They may be incompatible, untrusted, malicious, or conflict with package-managed system libraries.

69. Networking, DNS, incorrect URLs, unsupported releases, TLS problems, authentication, proxy issues, or signing problems.

70. The package manager may be unable to resolve repository hostnames.

71. Investigate the repository and trust configuration rather than disabling verification.

72. A mechanism preventing conflicting simultaneous modifications to package-management state.

73. To protect package databases and installation state.

74. Determine whether another legitimate package-management process is running.

75. Another process may still be modifying the package database.

76. It attempts to configure unpacked Debian packages that remain unconfigured.

77. It asks APT to attempt to correct broken dependency state.

78. No.

79. The package manager may be unable to download, unpack, create, or update required files.

80. Exhaustion of the filesystem structures used to represent files and directories.

81.

```bash
df -h
```

82.

```bash
df -i
```

83. Software installation changes the system and can execute privileged actions.

84. Remote content is downloaded and immediately executed without an inspection step.

85. The downloaded instructions execute with root privileges.

86. To understand what it changes and assess whether it is trustworthy.

87. It allows another source to participate in software distribution and potentially updates.

88. Different releases may require incompatible package and library versions.

89. APT and DNF primarily manage operating-system packages. Language package managers manage dependencies within their language ecosystems.

90. Updates can affect applications, libraries, services, kernels, and availability.

91. They fix vulnerabilities and other software defects.

92. The currently running kernel remains loaded until the system boots into the newly installed kernel.

93. Locally stored package or repository data used by the package manager.

94. `apt clean` removes cached package archives. `apt autoremove` removes installed packages considered no longer required.

95. Yes.

96. Yes.

97. No.

98. No.

99. Read the exact error, classify the failure, check repositories, dependencies, locks, networking, DNS, storage, architecture, trust, and package database state as appropriate.

100. Verify package state, binary/version, configuration, service state, process, listening port, application response, logs, and required network accessibility.

---

# 170. Final Mental Model

Package management should now make sense as a complete system:

```text
                  Repository
                      |
                      v
              Package Metadata
                      |
                      v
               Package Manager
                 /         \
                /           \
               v             v
        Dependency       Package
        Resolution       Download
               \             /
                \           /
                 v         v
               Installation
                    |
        +-----------+-----------+
        |           |           |
        v           v           v
     Binaries    Config      Libraries
        |
        v
     Service
        |
        v
     Process
        |
        v
      Port
        |
        v
   Application
```

On Debian-based systems:

```text
Repositories
     |
     v
    APT
     |
     v
Debian package infrastructure / dpkg
     |
     v
 .deb packages
     |
     v
Installed software
```

On Red Hat-family systems:

```text
Repositories
     |
     v
    DNF
     |
     v
    RPM
     |
     v
 .rpm packages
     |
     v
Installed software
```

When an installation fails, do not think:

```text
APT is broken.
```

or:

```text
DNF is broken.
```

Think:

```text
What exactly failed?
       |
       v
Repository metadata?
       |
       v
Network or DNS?
       |
       v
Package availability?
       |
       v
Dependency resolution?
       |
       v
Package architecture?
       |
       v
Trust/signature verification?
       |
       v
Package lock?
       |
       v
Disk space?
       |
       v
Package database?
       |
       v
Installation scripts?
```

And when installation succeeds, do not stop at:

```text
Package installed successfully.
```

For a server application, continue:

```text
Package installed
      |
      v
Correct files present
      |
      v
Correct version
      |
      v
Configuration valid
      |
      v
Service running
      |
      v
Process healthy
      |
      v
Port listening
      |
      v
Application responding
      |
      v
Logs checked
      |
      v
Real access verified
```

That is the difference between simply knowing package-manager commands and understanding Linux software administration.
