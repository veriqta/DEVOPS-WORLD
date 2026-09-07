# Part 9: Linux Services, systemd, and the Boot Process

In Part 8, you learned how Linux processes work. You learned about PIDs, parent and child processes, foreground and background jobs, signals, CPU and memory usage, and process troubleshooting.

Now we build on that foundation.

A Linux server normally needs programs to run continuously. A web server must keep listening for requests. An SSH server must remain available for remote connections. Monitoring agents must keep collecting metrics. Databases must remain available to applications.

Linux needs a reliable way to start, stop, monitor, organize, and automatically start these programs when the machine boots.

On most modern Linux distributions, this responsibility is largely handled by `systemd`.

By the end of Part 9, you should be able to:

* Explain what services and daemons are.
* Understand why Linux servers depend on services.
* Explain what `systemd` does.
* Understand systemd units and service units.
* Manage services with `systemctl`.
* Distinguish starting a service from enabling it.
* Check whether services are active or enabled.
* Understand basic service dependencies.
* Explain the Linux boot process from firmware to userspace.
* Understand systemd targets.
* Identify services that start during boot.
* Read system and service logs with `journalctl`.
* Follow service logs in real time.
* Identify failed units.
* Understand basic service configuration.
* Troubleshoot services that fail to start or crash.
* Diagnose port conflicts.
* Diagnose configuration failures.
* Build and manage a simple web service.

---

# 1. What Is a Service?

A service is a program or workload designed to provide some function, often continuously or in the background.

Examples include:

* Web servers.
* SSH servers.
* Databases.
* DNS servers.
* Monitoring agents.
* Logging agents.
* Container runtimes.
* Time synchronization services.
* Scheduled-job infrastructure.

For example, NGINX can provide HTTP and HTTPS services.

An SSH server provides remote shell access.

A database service accepts database connections from applications.

Conceptually:

```text
Users
  |
  v
Linux Server
  |
  +---- SSH service
  |
  +---- Web service
  |
  +---- Database service
  |
  +---- Monitoring service
```

The server may need these services available without someone manually opening a terminal and starting each program.

---

# 2. Service vs Ordinary Command

Suppose you run:

```bash
date
```

The program runs, prints the date, and exits.

Conceptually:

```text
Start
  |
  v
date
  |
  v
Print result
  |
  v
Exit
```

Now consider a web server.

It needs to:

```text
Start
  |
  v
Listen for connections
  |
  v
Receive requests
  |
  v
Send responses
  |
  v
Keep waiting
```

It normally remains running.

That makes service management important.

---

# 3. What Is a Daemon?

A daemon is a background process that usually runs for an extended period and provides some system or application function.

Common examples include:

```text
sshd
cron
nginx
systemd-journald
```

Historically, many daemon names end in:

```text
d
```

For example:

```text
sshd
```

means the SSH daemon.

```text
httpd
```

is commonly the Apache HTTP Server daemon name on RHEL-family distributions.

But the naming convention is not mandatory.

Not every background process is necessarily called a daemon, and not every daemon name ends with `d`.

---

# 4. Service vs Daemon

The terms are related, but they describe slightly different ideas.

A daemon refers primarily to a long-running background process.

A service refers more broadly to functionality managed or provided by the system.

For example:

```text
systemd service unit
        |
        v
Starts NGINX
        |
        v
NGINX daemon process
        |
        v
Provides web service
```

In everyday Linux administration, engineers often use "service" and "daemon" somewhat interchangeably.

For beginners, remember:

```text
Daemon
A long-running background process.

Service
A managed function or workload provided by the system.
```

---

# 5. Why Servers Run Services

Imagine manually managing a production web application.

Every time the server reboots, an engineer would have to SSH into it and run:

```bash
/path/to/web-server
```

Then start:

```bash
/path/to/monitoring-agent
```

Then:

```bash
/path/to/logging-agent
```

Then:

```bash
/path/to/application
```

This is unreliable.

What happens if:

* The server reboots at 03:00?
* The application crashes?
* Services must start in a particular order?
* A database must be available before another application starts?
* You need consistent logging?
* You need to know whether something failed during boot?

Linux service managers solve these problems.

---

# 6. What Is systemd?

`systemd` is a system and service manager used by many modern Linux distributions.

It manages much more than services.

Its responsibilities can include:

* Starting userspace during boot.
* Managing services.
* Managing dependencies.
* Tracking service processes.
* Managing sockets.
* Managing timers.
* Managing mount points.
* Managing targets.
* Integrating with system logging.
* Managing certain system states.

On many modern Linux systems, `systemd` runs as:

```text
PID 1
```

Check:

```bash
ps -p 1 -o pid,comm,args
```

You may see something involving:

```text
systemd
```

Another useful check is:

```bash
cat /proc/1/comm
```

On a typical systemd host:

```text
systemd
```

may be returned.

---

# 7. Is systemd Used Everywhere?

No.

This is important.

Many current distributions use systemd, including commonly:

* Ubuntu.
* Debian.
* Fedora.
* RHEL.
* Rocky Linux.
* AlmaLinux.
* Amazon Linux 2 and Amazon Linux 2023.

But Linux does not require systemd.

Other init systems and service managers exist, including:

* OpenRC.
* runit.
* s6.
* SysV init.

Containers can also behave differently. A container may run the application itself as PID 1 rather than systemd.

Therefore:

```text
Linux = systemd
```

is incorrect.

A better statement is:

```text
systemd is the dominant init and service-management system on many modern Linux distributions.
```

---

# 8. Why systemd Matters in DevOps and Cloud

Suppose you deploy an application to a Linux virtual machine.

Your application needs to:

* Start automatically after reboot.
* Run under a specific user.
* Restart after certain failures.
* Write logs.
* Start after networking is available.
* Receive environment variables.
* Stop cleanly during shutdown.

You could create scripts to handle all of this manually.

Or you could define a systemd service.

Conceptually:

```text
Linux boots
    |
    v
systemd
    |
    v
Dependencies satisfied
    |
    v
Application service starts
    |
    v
Application process runs
```

This makes systemd directly relevant to DevOps and server administration.

---

# 9. What Is a systemd Unit?

systemd manages resources through objects called:

```text
units
```

A unit represents something systemd knows how to manage.

Units can represent:

* Services.
* Sockets.
* Devices.
* Mount points.
* Automount points.
* Swap.
* Timers.
* Paths.
* Targets.

Unit names normally have suffixes describing their type.

Examples:

```text
nginx.service
sshd.service
docker.service
ssh.service
multi-user.target
graphical.target
backup.timer
home.mount
```

---

# 10. Common systemd Unit Types

Some common unit suffixes include:

```text
.service
.socket
.target
.timer
.mount
.automount
.path
.device
.swap
```

For this part, our main focus is:

```text
.service
```

But understanding that systemd manages more than services will help later.

---

# 11. Service Units

A service unit tells systemd how to manage a service.

It may define things such as:

* Description.
* Dependencies.
* Ordering.
* Command to execute.
* User.
* Working directory.
* Environment.
* Restart behavior.
* Startup relationships.

A simple service file might conceptually look like:

```ini
[Unit]
Description=Example Application
After=network.target

[Service]
ExecStart=/usr/local/bin/example-app
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

Do not worry if every line does not yet make sense.

We will break service configuration down later.

---

# 12. systemctl

The main command used to interact with the systemd manager is:

```bash
systemctl
```

You use it for tasks such as:

```text
Check service status
Start service
Stop service
Restart service
Reload service
Enable service
Disable service
Check active state
Check enabled state
Find failed units
Inspect dependencies
```

The basic structure is:

```bash
systemctl ACTION UNIT
```

Example:

```bash
systemctl status ssh
```

or:

```bash
systemctl status sshd
```

The exact service name depends on the distribution.

---

# 13. Finding the Correct Service Name

A major beginner mistake is assuming every distribution uses identical service names.

For example, the OpenSSH server service is commonly:

Ubuntu/Debian:

```text
ssh.service
```

RHEL-family distributions:

```text
sshd.service
```

Apache commonly appears as:

Ubuntu/Debian:

```text
apache2.service
```

RHEL-family:

```text
httpd.service
```

Never assume.

You can search:

```bash
systemctl list-unit-files --type=service | grep -i ssh
```

Or:

```bash
systemctl list-units --type=service | grep -i ssh
```

---

# 14. Unit Name Shortcuts

When working with service units, systemctl can usually infer:

```text
.service
```

So:

```bash
systemctl status nginx
```

normally refers to:

```bash
systemctl status nginx.service
```

Both are commonly valid.

For clarity while learning, understand that the complete unit name is:

```text
nginx.service
```

---

# 15. Checking Service Status

Use:

```bash
systemctl status SERVICE
```

For example:

```bash
systemctl status ssh
```

or:

```bash
systemctl status sshd
```

depending on your system.

A typical status might contain information similar to:

```text
● ssh.service - OpenBSD Secure Shell server
     Loaded: loaded (...)
     Active: active (running)
       Docs: ...
   Main PID: 850 (sshd)
      Tasks: ...
     Memory: ...
        CPU: ...
```

The exact output varies.

---

# 16. Understanding `systemctl status`

Several fields matter.

## Loaded

Example:

```text
Loaded: loaded (...)
```

This means systemd successfully found and loaded the unit definition.

This does not mean the service is currently running.

---

# 17. Active

You might see:

```text
Active: active (running)
```

This normally means the service is currently running.

Other states can include:

```text
inactive
failed
activating
deactivating
```

The exact state depends on the unit and what it is doing.

---

# 18. Main PID

You may see:

```text
Main PID: 850
```

This connects what you learned in Part 8 with service management.

systemd is managing a service.

That service has processes.

Conceptually:

```text
systemd
   |
   v
ssh.service
   |
   v
sshd
PID 850
```

You can investigate that PID with:

```bash
ps -p 850 -f
```

---

# 19. Service Status and Logs

`systemctl status` may display some recent log messages near the bottom.

This is extremely useful when a service fails.

For example:

```text
Active: failed
```

may be accompanied by messages indicating:

* Configuration syntax error.
* Permission problem.
* Missing file.
* Port already in use.
* Invalid command.
* Dependency failure.

But `systemctl status` is not your complete logging tool.

For deeper investigation, use:

```bash
journalctl
```

---

# 20. Starting a Service

Use:

```bash
sudo systemctl start SERVICE
```

Example:

```bash
sudo systemctl start nginx
```

What happens conceptually:

```text
You
 |
 v
systemctl start nginx
 |
 v
Request sent to systemd
 |
 v
systemd reads nginx.service
 |
 v
systemd starts required process(es)
 |
 v
NGINX begins running
```

Verify:

```bash
systemctl status nginx
```

Or:

```bash
systemctl is-active nginx
```

---

# 21. Why `sudo` Is Often Required

Starting and stopping system services changes system-wide state.

Regular users generally do not have unrestricted permission to perform these operations.

Therefore you often use:

```bash
sudo systemctl start nginx
```

Do not interpret this as:

```text
Always add sudo when something fails.
```

If you get an error, read it first.

Use elevated privileges only when required and authorized.

---

# 22. Stopping a Service

Use:

```bash
sudo systemctl stop nginx
```

Conceptually:

```text
systemctl stop
      |
      v
systemd
      |
      v
Requests service shutdown
      |
      v
Processes terminate
```

Verify:

```bash
systemctl is-active nginx
```

You may receive:

```text
inactive
```

You can also inspect:

```bash
systemctl status nginx
```

---

# 23. Warning Before Stopping Services

Never blindly stop services on production systems.

Before:

```bash
sudo systemctl stop SERVICE
```

ask:

* What does this service do?
* Is it serving users?
* Does another service depend on it?
* Will stopping it disconnect me?
* Do I have a maintenance window?
* Is redundancy available?
* How will I verify recovery?

Stopping SSH remotely, networking components, databases, or critical infrastructure can have serious consequences.

Practice on a lab system.

---

# 24. Restarting a Service

Use:

```bash
sudo systemctl restart nginx
```

Restart generally means:

```text
Stop
 |
 v
Start again
```

A restart can be useful after configuration changes when the service requires a full restart.

But restarting can cause:

* Temporary downtime.
* Dropped connections.
* Interrupted work.

The exact impact depends on the service.

---

# 25. Reloading a Service

Some services can reload configuration without a complete stop and restart.

Example:

```bash
sudo systemctl reload nginx
```

Conceptually:

```text
Running service
      |
      v
Reload configuration
      |
      v
Continue operation
```

This can be less disruptive.

However:

```text
Not every service supports reload.
```

If reload is unsupported, systemd may return an error.

---

# 26. Restart vs Reload

Understand the distinction.

```text
restart
Stop and start the service again.

reload
Ask the running service to reload configuration without a complete restart, if supported.
```

Do not assume reload always exists.

For NGINX, configuration reload is commonly supported.

---

# 27. `reload-or-restart`

systemd also provides:

```bash
sudo systemctl reload-or-restart SERVICE
```

This requests a reload when supported and otherwise performs a restart.

It can be useful in automation, but beginners should still understand which operation their particular service actually supports and what operational impact it has.

---

# 28. Starting Is Not Enabling

This is one of the most important systemd concepts.

Suppose you run:

```bash
sudo systemctl start nginx
```

NGINX starts now.

But that command alone does not necessarily configure NGINX to start during future boots.

Think:

```text
start
Run now.
```

versus:

```text
enable
Configure startup relationship for boot.
```

---

# 29. Enabling a Service

Use:

```bash
sudo systemctl enable nginx
```

This configures the unit to participate in startup according to its `[Install]` configuration, commonly by creating symbolic links into target wants directories.

You may see output similar to:

```text
Created symlink ...
```

That should remind you of symbolic links from Part 3.

Check:

```bash
systemctl is-enabled nginx
```

Possible output:

```text
enabled
```

---

# 30. Does `enable` Start the Service Immediately?

Usually:

```text
No.
```

Running:

```bash
sudo systemctl enable nginx
```

normally configures future startup but does not itself mean the service was started right now.

If you want both:

```bash
sudo systemctl enable --now nginx
```

`--now` asks systemctl to enable and start the unit.

---

# 31. Disabling a Service

Use:

```bash
sudo systemctl disable nginx
```

This removes the enablement relationships created for startup.

It does not necessarily stop the currently running service.

This is another important distinction.

```text
disable
Do not automatically start through those enablement links in future.

stop
Stop running now.
```

To do both:

```bash
sudo systemctl disable --now nginx
```

Be careful on production systems.

---

# 32. Start, Stop, Enable, Disable

Remember this model:

```text
start
Current runtime state -> running

stop
Current runtime state -> stopped

enable
Boot configuration -> enabled

disable
Boot configuration -> disabled
```

Therefore a service can be:

```text
active + enabled
active + disabled
inactive + enabled
inactive + disabled
```

These are not contradictions.

---

# 33. Example: Active but Disabled

Suppose:

```bash
sudo systemctl start nginx
```

but you never enable it.

Then:

```bash
systemctl is-active nginx
```

may say:

```text
active
```

while:

```bash
systemctl is-enabled nginx
```

may say:

```text
disabled
```

Meaning:

```text
NGINX is running now, but is not enabled through the normal boot enablement mechanism.
```

---

# 34. Example: Inactive but Enabled

Suppose NGINX is enabled but you manually stop it:

```bash
sudo systemctl stop nginx
```

Then:

```bash
systemctl is-active nginx
```

may return:

```text
inactive
```

while:

```bash
systemctl is-enabled nginx
```

returns:

```text
enabled
```

Meaning:

```text
It is stopped now, but configured to start through its normal boot target on a future boot.
```

---

# 35. `is-active`

Use:

```bash
systemctl is-active nginx
```

Common results include:

```text
active
inactive
failed
activating
```

This command is particularly useful in scripts because its exit status can also indicate whether the unit is active.

Example:

```bash
if systemctl is-active --quiet nginx; then
    echo "NGINX is running"
else
    echo "NGINX is not running"
fi
```

You do not need to memorize the script yet.

The important point is that `is-active` is useful for machine-readable checks.

---

# 36. `is-enabled`

Use:

```bash
systemctl is-enabled nginx
```

Possible results can include states such as:

```text
enabled
disabled
static
masked
```

There are additional states depending on the unit.

For now:

```text
enabled
Configured to be pulled in through enablement.

disabled
Not enabled through that mechanism.
```

---

# 37. What Is a Static Unit?

You may run:

```bash
systemctl is-enabled some-unit
```

and receive:

```text
static
```

This does not automatically mean something is broken.

A static unit generally lacks normal `[Install]` enablement information and may instead be started as a dependency of another unit or through another activation mechanism.

Therefore:

```text
static != failed
```

---

# 38. Masked Services

A stronger form of preventing activation is:

```text
masked
```

A masked unit is configured so that it cannot normally be started.

You can mask a unit with:

```bash
sudo systemctl mask SERVICE
```

and undo it with:

```bash
sudo systemctl unmask SERVICE
```

Do not experiment with masking critical services on a system you depend on.

For beginners:

```text
disable
Removes normal enablement.

mask
Prevents normal activation more strongly.
```

---

# 39. Listing Running Services

Use:

```bash
systemctl list-units --type=service
```

This shows currently loaded service units, generally emphasizing units systemd currently knows as loaded.

To show active services:

```bash
systemctl list-units --type=service --state=running
```

This is useful when asking:

```text
What services are currently running?
```

---

# 40. Listing Installed Service Unit Files

Use:

```bash
systemctl list-unit-files --type=service
```

This is different from:

```bash
systemctl list-units --type=service
```

`list-unit-files` focuses on installed unit files and their enablement state.

It can show entries such as:

```text
nginx.service      enabled
ssh.service        enabled
example.service    disabled
```

---

# 41. Service Dependencies

Services rarely exist completely independently.

Imagine:

```text
Application
    |
    +---- needs networking
    |
    +---- needs mounted storage
    |
    +---- may need database access
```

systemd has dependency and ordering concepts that help describe relationships between units.

Important directives include:

```text
Requires=
Wants=
After=
Before=
```

These do not all mean the same thing.

---

# 42. `After=`

Example:

```ini
After=network.target
```

This is primarily an ordering relationship.

It means:

```text
Start this unit after network.target in the relevant transaction.
```

It does not by itself necessarily mean:

```text
network.target must be pulled in because this unit exists.
```

This distinction is important.

Ordering and requirement are different concepts.

---

# 43. `Before=`

`Before=` creates the opposite ordering relationship.

Example:

```ini
Before=example.service
```

means systemd should order this unit before `example.service` when both are being started as part of the relevant transaction.

---

# 44. `Requires=`

`Requires=` creates a stronger requirement relationship.

If unit A requires unit B, systemd pulls B in when A is started.

Failures and stop relationships can also affect A depending on exactly how the units are configured and ordered.

At beginner level:

```text
Requires=
Strong dependency relationship.
```

---

# 45. `Wants=`

`Wants=` creates a weaker dependency.

Conceptually:

```text
A wants B to be started too,
but B failing does not necessarily prevent A from being started.
```

This is widely used by systemd targets.

---

# 46. Dependency Does Not Automatically Mean Ordering

This is a subtle but important systemd concept.

For example:

```ini
Requires=database.service
```

does not by itself fully express:

```text
My application must start only after the database has finished starting.
```

Requirement and ordering are separate concepts.

You may need something conceptually like:

```ini
Requires=database.service
After=database.service
```

Even then, `After=` means startup ordering, not necessarily that the database is application-ready to accept queries.

Real application readiness may require application-specific health checks or retry behavior.

---

# 47. Viewing Dependencies

Use:

```bash
systemctl list-dependencies nginx
```

You can also inspect reverse relationships in some situations:

```bash
systemctl list-dependencies --reverse nginx
```

This can help answer:

```text
What does this unit depend on?

What other units may depend on this?
```

---

# 48. Inspecting a Unit

Use:

```bash
systemctl cat nginx
```

This displays the unit's configuration as systemd sees its unit-file fragments.

This is often safer than immediately editing a file because it lets you inspect first.

You can also use:

```bash
systemctl show nginx
```

This displays many properties.

It can be a lot of information.

For targeted properties:

```bash
systemctl show nginx -p MainPID -p ActiveState -p SubState
```

---

# 49. Introduction to the Linux Boot Process

Now we need to understand how systemd fits into boot.

When you press the power button, Linux is not instantly running.

Several stages occur.

A simplified boot sequence is:

```text
Power on
   |
   v
Firmware
   |
   v
Bootloader
   |
   v
Linux kernel
   |
   v
Early userspace / initramfs when used
   |
   v
Root filesystem
   |
   v
init / systemd
   |
   v
Targets and services
   |
   v
Login / usable system
```

The exact sequence varies by hardware, distribution, boot mode, and system configuration.

---

# 50. Stage 1: Firmware

When the machine powers on, firmware begins hardware initialization.

Modern systems commonly use:

```text
UEFI
```

Older systems commonly use:

```text
BIOS
```

Firmware performs tasks such as:

* Initial hardware initialization.
* Selecting a boot device.
* Locating boot software.
* Transferring control to the next stage.

Conceptually:

```text
Power
 |
 v
Firmware
 |
 v
Find boot path
```

---

# 51. BIOS vs UEFI

BIOS is the older firmware model.

UEFI is the modern firmware architecture used by most newer systems.

You do not need to master firmware internals to administer Linux services.

For now, remember:

```text
Firmware prepares the machine and begins the boot chain.
```

---

# 52. Stage 2: Bootloader

The bootloader's job includes loading the Linux kernel and often an initial RAM filesystem into memory, then transferring control to the kernel.

A common Linux bootloader is:

```text
GRUB
```

GRUB can present a boot menu and allow selection between:

* Different kernels.
* Different operating systems.
* Recovery options.

Conceptually:

```text
Firmware
   |
   v
Bootloader
   |
   +--> choose kernel
   |
   +--> load kernel
   |
   +--> load initramfs when used
   |
   v
Kernel
```

---

# 53. Stage 3: Linux Kernel

The bootloader transfers control to the Linux kernel.

The kernel begins initializing the core operating-system environment.

It handles things such as:

* CPU management.
* Memory management.
* Device drivers.
* Process management.
* Filesystems.
* Networking infrastructure.
* Hardware communication.

This is the same kernel concept you learned in Part 1.

---

# 54. Initial RAM Filesystem

Many Linux systems use an early userspace environment commonly called:

```text
initramfs
```

It provides files and tools needed before the real root filesystem can be fully mounted and used.

It may contain:

* Kernel modules.
* Storage drivers.
* Filesystem tools.
* Scripts.
* Device discovery components.

Conceptually:

```text
Kernel
  |
  v
initramfs
  |
  +--> prepare storage
  +--> find root filesystem
  +--> load needed components
  |
  v
Real root filesystem
```

Not every boot environment works identically, but this is a useful beginner model.

---

# 55. Stage 4: init/systemd

After the kernel and early userspace establish the environment needed for the real root filesystem, the kernel starts the initial userspace process.

On a typical systemd-based Linux host:

```text
systemd becomes PID 1
```

Check:

```bash
ps -p 1 -f
```

Conceptually:

```text
Kernel
  |
  v
systemd
PID 1
  |
  +--> mount-related units
  +--> device units
  +--> networking
  +--> logging
  +--> SSH
  +--> application services
  +--> login services
```

---

# 56. What PID 1 Makes Special

PID 1 has a special role in Linux userspace.

On a systemd host, systemd acts as the system manager.

It coordinates system initialization and service management.

This connects Part 8 and Part 9:

```text
Kernel
  |
  v
systemd
PID 1
  |
  v
Service units
  |
  v
Service processes
  |
  v
Child processes
```

---

# 57. systemd Targets

A target is a systemd unit used to group other units and represent synchronization points or broader system states.

Target units end with:

```text
.target
```

Examples include:

```text
multi-user.target
graphical.target
rescue.target
emergency.target
network.target
```

Targets are conceptually similar to collections or milestones.

---

# 58. `multi-user.target`

`multi-user.target` commonly represents a multi-user, non-graphical system state suitable for many servers.

It commonly pulls in services required for normal server operation.

Think:

```text
multi-user.target
       |
       +--> networking-related units
       +--> SSH
       +--> system services
       +--> enabled server workloads
```

Exact dependencies vary by distribution and configuration.

---

# 59. `graphical.target`

`graphical.target` represents a graphical system state.

It generally builds on:

```text
multi-user.target
```

and adds graphical-login functionality.

Conceptually:

```text
multi-user.target
       |
       v
graphical.target
       |
       v
Graphical login / desktop environment
```

A cloud server often has no need for a graphical desktop.

---

# 60. Rescue and Emergency Targets

systemd provides reduced environments useful for recovery.

Examples:

```text
rescue.target
emergency.target
```

These are useful when:

* Boot is broken.
* Filesystems need repair.
* Configuration prevents normal startup.
* Administrative recovery is required.

They are more advanced and can affect system availability, so do not switch production systems into these modes casually.

---

# 61. Checking the Default Target

Run:

```bash
systemctl get-default
```

You may see:

```text
multi-user.target
```

or:

```text
graphical.target
```

This tells you the target configured as the default boot target.

---

# 62. Changing the Default Target

The command is:

```bash
sudo systemctl set-default TARGET
```

For example:

```bash
sudo systemctl set-default multi-user.target
```

This changes future boot behavior.

Do not modify your default target merely for practice on a machine you rely on.

Knowing the command is enough until you have a disposable lab environment.

---

# 63. Boot-Time Services

When systemd boots toward the default target, dependencies cause many units to be started.

Enabled services often become linked into target dependency directories.

For example, a service may contain:

```ini
[Install]
WantedBy=multi-user.target
```

When enabled, systemd may create a symbolic link under a location such as:

```text
/etc/systemd/system/multi-user.target.wants/
```

This connects:

```text
multi-user.target
        |
        v
example.service
```

---

# 64. Viewing Target Dependencies

Try:

```bash
systemctl list-dependencies multi-user.target
```

You may see many units.

This gives you a better understanding of:

```text
What must or should be brought in to reach this target?
```

Do not expect identical output across Linux systems.

---

# 65. An Important Boot Mental Model

Do not imagine boot as:

```text
Start service 1
then service 2
then service 3
then service 4
```

systemd can start independent units in parallel when dependencies and ordering allow.

A better model is:

```text
                  target
                /   |   \
               /    |    \
              v     v     v
         service A  B   service C
              |           |
              v           v
         dependency D   mount E
```

systemd works with a dependency graph rather than a single simple sequential list.

---

# 66. Checking Boot Time

A useful systemd command is:

```bash
systemd-analyze
```

It may report time spent in stages such as:

```text
kernel
userspace
```

For example:

```text
Startup finished in ... (kernel) + ... (userspace) = ...
```

Exact output varies.

---

# 67. Finding Slow Boot Units

Use:

```bash
systemd-analyze blame
```

This lists units by initialization time.

Do not automatically assume the unit at the top is "the cause" of slow boot.

Dependencies and parallel startup complicate interpretation.

For deeper analysis:

```bash
systemd-analyze critical-chain
```

This can help identify time-critical ordering relationships.

These are useful tools, but detailed boot-performance analysis is beyond this beginner part.

---

# 68. What Is the systemd Journal?

systemd includes a logging service called:

```text
systemd-journald
```

It collects journal data from multiple sources.

The journal can contain information from:

* Services.
* Kernel.
* Boot process.
* system components.
* Applications writing through supported logging mechanisms.

You interact with journal data primarily using:

```bash
journalctl
```

---

# 69. Why Logs Matter

Suppose:

```bash
sudo systemctl start nginx
```

fails.

The status says:

```text
failed
```

That tells you the outcome.

It may not tell you the complete cause.

Logs can answer:

```text
What happened?

When did it happen?

What message did the application produce?

Was there a configuration error?

Was a file missing?

Was a port already occupied?

Was permission denied?
```

A good service troubleshooter checks logs early.

---

# 70. Basic `journalctl`

Run:

```bash
journalctl
```

This may produce a very large amount of output.

Depending on permissions, a regular user may not be allowed to see every system message.

Use:

```bash
sudo journalctl
```

only when broader administrative access is required and appropriate.

Because the journal can be huge, targeted queries are usually better.

---

# 71. Service-Specific Logs

To see journal messages associated with a unit:

```bash
journalctl -u nginx
```

Here:

```text
-u
```

means:

```text
unit
```

For another service:

```bash
journalctl -u ssh
```

or:

```bash
journalctl -u sshd
```

depending on your distribution.

---

# 72. Recent Service Logs

To show recent entries:

```bash
journalctl -u nginx -n 50
```

Here:

```text
-n 50
```

means:

```text
show the most recent 50 entries
```

This is useful during troubleshooting.

---

# 73. Following Service Logs

To watch logs as new messages arrive:

```bash
journalctl -u nginx -f
```

The:

```text
-f
```

means:

```text
follow
```

This is conceptually similar to:

```bash
tail -f
```

Press:

```text
Ctrl+C
```

to stop following.

Remember from Part 8 that this interrupts `journalctl`. It does not stop NGINX.

---

# 74. Logs From the Current Boot

Use:

```bash
journalctl -b
```

The:

```text
-b
```

means:

```text
current boot
```

This is useful when troubleshooting something that happened since the most recent reboot.

---

# 75. Previous Boot Logs

If persistent journal history is available, you can inspect previous boots.

List known boots:

```bash
journalctl --list-boots
```

Then:

```bash
journalctl -b -1
```

typically means:

```text
previous boot
```

This is extremely useful when a server failed during a previous boot but now starts successfully.

Availability depends on journal storage configuration.

---

# 76. Kernel Logs

Use:

```bash
journalctl -k
```

The:

```text
-k
```

limits output to kernel messages.

This can help investigate:

* Hardware problems.
* Driver issues.
* OOM events.
* Filesystem issues.
* Device problems.

For example:

```bash
journalctl -k | grep -i oom
```

connects with the OOM troubleshooting you learned in Part 8.

---

# 77. Logs Since a Time

You can filter by time.

Example:

```bash
journalctl --since "10 minutes ago"
```

For one service:

```bash
journalctl -u nginx --since "10 minutes ago"
```

You can also use timestamps.

Example:

```bash
journalctl -u nginx --since "2026-08-26 09:00:00"
```

This is extremely useful during incidents.

If a deployment occurred at 09:15, you can inspect logs around that time instead of searching the entire journal.

---

# 78. Logs With Explanatory Context

A useful troubleshooting option is:

```bash
journalctl -xe
```

Depending on the system and journal version, this displays recent journal messages with additional context.

For a service, a more targeted approach is often better:

```bash
journalctl -u SERVICE -n 100 --no-pager
```

Targeted logs reduce noise.

---

# 79. `--no-pager`

`journalctl` and `systemctl` may display output through a pager such as `less`.

For scripts or easy copying:

```bash
journalctl -u nginx --no-pager
```

You can combine:

```bash
journalctl -u nginx -n 50 --no-pager
```

---

# 80. Service Logs May Also Exist in Files

Not every application relies solely on the journal.

Applications may write to files such as:

```text
/var/log/nginx/access.log
/var/log/nginx/error.log
```

or other paths under:

```text
/var/log
```

A service may therefore have:

```text
systemd journal
+
application-specific log files
```

Both can matter.

---

# 81. Example: NGINX Logs

On a typical NGINX installation, you may find:

```text
/var/log/nginx/access.log
/var/log/nginx/error.log
```

The access log can show HTTP requests.

The error log can show application or server errors.

Follow:

```bash
sudo tail -f /var/log/nginx/access.log
```

or:

```bash
sudo tail -f /var/log/nginx/error.log
```

Paths and permissions can vary by configuration.

---

# 82. journalctl vs Application Logs

Do not think:

```text
journalctl replaces every application log.
```

Instead:

```text
journalctl
System/service lifecycle and journal messages.

Application log
Application-specific operational information.
```

During troubleshooting, you may need both.

---

# 83. Failed Units

To list failed units:

```bash
systemctl --failed
```

You may see something like:

```text
UNIT                 LOAD   ACTIVE SUB    DESCRIPTION
example.service      loaded failed failed Example Service
```

This is an important troubleshooting command.

---

# 84. What Does "Failed" Mean?

A failed unit means systemd recorded a failure state.

Possible causes include:

* Process exited with an error.
* Startup timeout.
* Invalid executable.
* Permission problem.
* Configuration failure.
* Dependency failure.
* Resource conflict.

The word:

```text
failed
```

is the beginning of your investigation, not the diagnosis.

---

# 85. The Core Service Troubleshooting Workflow

When a service fails:

```text
1. Check status.
2. Read the exact error.
3. Check journal logs.
4. Check application logs.
5. Inspect the unit definition.
6. Validate configuration.
7. Check files and permissions.
8. Check ports.
9. Check dependencies.
10. Check process state.
11. Fix the actual cause.
12. Restart or start safely.
13. Verify service state.
14. Verify application functionality.
15. Check logs again.
```

This workflow is far better than repeatedly running:

```bash
sudo systemctl restart SERVICE
```

---

# 86. Step 1: Check Status

Start with:

```bash
systemctl status SERVICE
```

Example:

```bash
systemctl status nginx
```

Look for:

```text
Loaded
Active
Main PID
exit status
recent log messages
```

Do not skip the error text.

---

# 87. Step 2: Check the Journal

Run:

```bash
journalctl -u SERVICE -n 100 --no-pager
```

Example:

```bash
journalctl -u nginx -n 100 --no-pager
```

If necessary:

```bash
sudo journalctl -u nginx -n 100 --no-pager
```

Look for the first meaningful error, not merely the final message saying the service failed.

---

# 88. Why the First Error Matters

Suppose logs say:

```text
Configuration syntax error
Failed to start process
Unit entered failed state
```

The important message is:

```text
Configuration syntax error
```

The later messages are consequences.

Troubleshooting means finding:

```text
Cause
   |
   v
Failure
   |
   v
Secondary messages
```

not treating every line as a separate root cause.

---

# 89. Service Configuration

Service behavior is defined through unit configuration.

Common system unit locations include:

```text
/usr/lib/systemd/system/
/lib/systemd/system/
/etc/systemd/system/
```

Exact layout varies by distribution.

Administrators generally use:

```text
/etc/systemd/system/
```

for local unit definitions and overrides.

Vendor/package-provided unit files are commonly stored under `/usr/lib/systemd/system/` or `/lib/systemd/system/`.

---

# 90. Do Not Casually Edit Vendor Unit Files

Suppose:

```bash
systemctl cat nginx
```

shows a file under:

```text
/lib/systemd/system/nginx.service
```

or:

```text
/usr/lib/systemd/system/nginx.service
```

Editing the package's file directly is usually a poor administrative practice because package updates can replace it.

For local modifications, systemd supports drop-in overrides.

A common command is:

```bash
sudo systemctl edit nginx
```

This creates or edits an override configuration.

---

# 91. Anatomy of a Service Unit

A basic service unit commonly contains sections such as:

```ini
[Unit]

[Service]

[Install]
```

Each section has a different purpose.

---

# 92. `[Unit]`

The `[Unit]` section contains general metadata and dependency relationships.

Example:

```ini
[Unit]
Description=VERIQTA Demo Web Service
After=network.target
```

`Description=` provides human-readable information.

`After=` defines ordering.

---

# 93. `[Service]`

The `[Service]` section describes how the service runs.

Example:

```ini
[Service]
Type=simple
User=veriqta
WorkingDirectory=/opt/veriqta-app
ExecStart=/usr/bin/python3 -m http.server 8080
Restart=on-failure
```

Important directives include:

```text
Type=
User=
Group=
WorkingDirectory=
ExecStart=
ExecReload=
ExecStop=
Environment=
EnvironmentFile=
Restart=
RestartSec=
```

You do not need all of them for every service.

---

# 94. `ExecStart=`

`ExecStart=` defines the command systemd starts.

Example:

```ini
ExecStart=/usr/bin/python3 /opt/app/app.py
```

A common failure is:

```text
Executable does not exist.
```

Check:

```bash
ls -l /usr/bin/python3
```

and:

```bash
ls -l /opt/app/app.py
```

Also confirm the service user can access required files.

---

# 95. `User=`

Example:

```ini
User=veriqta
```

This tells systemd to run the service process as that user.

This supports least privilege.

Instead of:

```text
Every application runs as root
```

you can use:

```text
Dedicated service user
        |
        v
Only required permissions
```

This limits damage if the application is compromised.

---

# 96. `WorkingDirectory=`

Example:

```ini
WorkingDirectory=/opt/veriqta-app
```

This defines the service's working directory.

If the directory does not exist, startup may fail.

Check:

```bash
ls -ld /opt/veriqta-app
```

Also check permissions:

```bash
namei -l /opt/veriqta-app
```

if `namei` is available.

Remember that directory traversal requires appropriate execute permission on parent directories.

---

# 97. `Restart=`

Example:

```ini
Restart=on-failure
```

This can ask systemd to restart the service after certain failures.

Common values include:

```text
no
on-success
on-failure
on-abnormal
on-abort
always
```

The exact behavior has details beyond this beginner section.

A common application setting is:

```ini
Restart=on-failure
```

---

# 98. Restart Policies Need Care

Automatic restart sounds perfect:

```text
Application crashes
        |
        v
systemd restarts it
```

But imagine the application has a permanent configuration error:

```text
Start
 |
 v
Crash
 |
 v
Restart
 |
 v
Crash
 |
 v
Restart
```

This creates a crash loop.

Restart policies improve resilience, but they do not replace root-cause troubleshooting.

---

# 99. `[Install]`

The `[Install]` section describes how the unit participates in enablement.

Example:

```ini
[Install]
WantedBy=multi-user.target
```

When you run:

```bash
sudo systemctl enable example.service
```

systemd uses this information to establish the relevant boot-time relationship.

---

# 100. What Is `daemon-reload`?

Suppose you create or modify a unit file.

systemd may still be operating with its previously loaded unit definitions.

Run:

```bash
sudo systemctl daemon-reload
```

This tells systemd to reload unit-file configuration.

Then you can:

```bash
sudo systemctl restart example.service
```

or:

```bash
sudo systemctl start example.service
```

Do not confuse:

```text
daemon-reload
```

with:

```text
reload SERVICE
```

They do different things.

---

# 101. `daemon-reload` vs Service Reload

```bash
sudo systemctl daemon-reload
```

means:

```text
systemd, reread unit files and rebuild manager configuration as needed.
```

While:

```bash
sudo systemctl reload nginx
```

means:

```text
NGINX, reload your application configuration.
```

Very different operations.

---

# 102. Service Will Not Start

Scenario:

```bash
sudo systemctl start example.service
```

fails.

Do not keep repeating the command.

Start with:

```bash
systemctl status example.service
```

Then:

```bash
journalctl -u example.service -n 100 --no-pager
```

Then investigate based on the evidence.

---

# 103. Common Cause: Executable Missing

Suppose logs indicate:

```text
No such file or directory
```

Inspect:

```bash
systemctl cat example.service
```

Find:

```ini
ExecStart=/opt/example/bin/server
```

Then:

```bash
ls -l /opt/example/bin/server
```

If the file does not exist, you found a likely cause.

Possible reasons:

* Wrong path.
* Application not installed.
* File moved.
* Typo.
* Deployment incomplete.

---

# 104. Common Cause: Permission Denied

Suppose logs say:

```text
Permission denied
```

Check the service user:

```bash
systemctl show example.service -p User -p Group
```

Inspect relevant files:

```bash
ls -l /path/to/file
```

Directories:

```bash
ls -ld /path/to/directory
```

Remember Part 5:

A process may need:

* Read permission on configuration.
* Execute/traverse permission on directories.
* Execute permission on scripts or binaries where applicable.
* Write permission on log or data directories.

Do not "fix" this with:

```bash
chmod -R 777 /
```

or broad `777` permissions.

Find the specific access requirement.

---

# 105. Common Cause: Wrong Working Directory

Suppose:

```ini
WorkingDirectory=/opt/application
```

but:

```bash
ls -ld /opt/application
```

returns:

```text
No such file or directory
```

The service may fail before the application even starts.

Correct the directory or service configuration.

Then:

```bash
sudo systemctl daemon-reload
sudo systemctl restart example.service
```

Verify afterward.

---

# 106. Service Starts but Immediately Crashes

This is different from:

```text
systemd cannot execute the program.
```

The process may successfully start and then exit because the application itself encounters an error.

Possible causes:

* Invalid application configuration.
* Missing environment variable.
* Database unavailable.
* Port conflict.
* Missing dependency.
* Permission problem.
* Application bug.
* Missing secret.
* Incorrect command-line argument.

---

# 107. Investigating a Crash

Use:

```bash
systemctl status example.service
```

Then:

```bash
journalctl -u example.service -n 100 --no-pager
```

If application logs exist:

```bash
tail -n 100 /path/to/application.log
```

Look for the earliest application error.

Then inspect:

```bash
systemctl cat example.service
```

Ask:

```text
What command is being executed?

Which user runs it?

What directory does it use?

Which environment does it receive?

Which dependencies does it need?
```

---

# 108. Running the Command Manually

Sometimes it is useful in a safe lab to run the `ExecStart` command manually to expose an application's direct error output.

For example, if:

```ini
ExecStart=/usr/bin/python3 /opt/app/app.py
```

you might test:

```bash
/usr/bin/python3 /opt/app/app.py
```

But there is an important limitation.

Your interactive shell environment may differ from systemd's environment.

Differences can include:

* User.
* Working directory.
* Environment variables.
* PATH.
* Permissions.
* Resource limits.

So:

```text
It works manually
```

does not prove:

```text
The systemd unit is correct.
```

---

# 109. Environment Variables

Applications commonly need environment variables.

For example:

```text
APP_ENV=production
PORT=8080
```

A unit may contain:

```ini
Environment="APP_ENV=production"
Environment="PORT=8080"
```

or:

```ini
EnvironmentFile=/etc/example/example.env
```

A missing environment variable can cause an application to crash even though its executable exists.

---

# 110. Secrets and Environment Files

Do not casually place passwords, API keys, or secrets directly into world-readable unit files.

If an environment file contains sensitive data, permissions matter.

For example:

```bash
ls -l /etc/example/example.env
```

A file containing secrets should not be readable by every user.

The exact secure design depends on the application and environment.

The key beginner principle is:

```text
Configuration convenience does not remove secret-management requirements.
```

---

# 111. Port Conflicts

A network service often needs to listen on a port.

Example:

```text
NGINX -> TCP port 80
Application -> TCP port 8080
SSH -> TCP port 22 by default
```

Only one listener can normally bind the same IP address, protocol, and port combination unless special socket-sharing behavior is configured.

If another process already owns the required listening socket, startup may fail.

---

# 112. Typical Port Conflict Error

An application may report something similar to:

```text
Address already in use
```

or:

```text
bind() failed
```

The beginner mistake is:

```text
Kill random processes until it works.
```

Instead, identify the owner.

---

# 113. Checking Listening Ports With `ss`

Use:

```bash
sudo ss -ltnp
```

Break it down:

```text
ss
Socket inspection command.

-l
Show listening sockets.

-t
Show TCP sockets.

-n
Show numeric addresses and ports.

-p
Show process information when permissions allow.
```

To search for port 8080:

```bash
sudo ss -ltnp | grep ':8080'
```

You may see something similar to:

```text
LISTEN ... 0.0.0.0:8080 ... users:(("python3",pid=4200,fd=3))
```

Now you have evidence.

---

# 114. Investigating the Port Owner

Suppose PID 4200 owns port 8080.

Run:

```bash
ps -p 4200 -o pid,ppid,user,stat,etime,cmd
```

Then determine:

```text
What application is it?

Should it be running?

Is it managed by systemd?

Is 8080 intentionally assigned to it?
```

Do not immediately:

```bash
kill -9 4200
```

The correct fix may be to:

* Change one service's port.
* Stop an obsolete service.
* Correct duplicate deployments.
* Fix configuration management.

---

# 115. Configuration Failures

Many services provide a configuration validation command.

This is extremely valuable.

For NGINX:

```bash
sudo nginx -t
```

This checks NGINX configuration syntax and certain referenced configuration conditions.

If valid, you may see output indicating the syntax is okay and the test succeeded.

If invalid, NGINX normally reports the file and often a line number or related context.

---

# 116. Why Validate Before Restarting

Imagine NGINX is currently serving production traffic.

You edit its configuration incorrectly.

If you immediately:

```bash
sudo systemctl restart nginx
```

you may stop a working service and fail to start it again.

A safer workflow is:

```text
Edit configuration
      |
      v
Validate configuration
      |
      +---- invalid -> fix it
      |
      v
Valid
      |
      v
Reload/restart as appropriate
      |
      v
Verify
```

For NGINX:

```bash
sudo nginx -t
```

then, if valid:

```bash
sudo systemctl reload nginx
```

when a reload is appropriate.

---

# 117. Configuration Failure Workflow

Use:

```text
1. Identify configuration change.
2. Find configuration file.
3. Validate syntax if the application provides a validator.
4. Read exact error.
5. Fix the smallest incorrect part.
6. Validate again.
7. Reload or restart appropriately.
8. Check service status.
9. Check logs.
10. Test actual functionality.
```

---

# 118. Service Status Is Not Application Health

This is one of the most important lessons in this part.

Suppose:

```bash
systemctl is-active nginx
```

returns:

```text
active
```

Does that prove users can access the website?

No.

It proves systemd considers the service active.

You still need application-level verification.

For a local web service:

```bash
curl http://localhost
```

or:

```bash
curl -I http://localhost
```

may help verify HTTP functionality.

---

# 119. Service Health Has Multiple Layers

Think:

```text
Layer 1
Is the service active?

Layer 2
Is the process alive?

Layer 3
Is the port listening?

Layer 4
Does the application respond?

Layer 5
Does it return the correct response?

Layer 6
Can real clients reach it through the network?
```

`systemctl status` only answers part of the problem.

---

# 120. Example Web Troubleshooting Flow

Users report:

```text
Website unavailable.
```

Check:

```bash
systemctl status nginx
```

If active:

```bash
sudo ss -ltnp | grep ':80'
```

Then:

```bash
curl -I http://localhost
```

If local access works but remote users cannot connect, investigate areas such as:

* Firewall.
* Cloud security rules.
* Load balancer.
* DNS.
* Routing.
* Reverse proxy configuration.

Do not restart NGINX simply because the website is unreachable.

---

# 121. Service Troubleshooting Decision Tree

Use this mental model:

```text
Service problem
     |
     v
systemctl status
     |
     +---- failed
     |       |
     |       v
     |   journalctl
     |       |
     |       +--> configuration?
     |       +--> permissions?
     |       +--> missing file?
     |       +--> port conflict?
     |       +--> dependency?
     |
     +---- active
             |
             v
       Is application working?
             |
        +----+----+
        |         |
       yes        no
                  |
                  v
             Check port
                  |
                  v
             Check logs
                  |
                  v
             Test locally
                  |
                  v
             Check dependencies
                  |
                  v
             Check network path
```

---

# 122. Web-Service Management Lab

We will now create a safe beginner lab.

This lab uses Python's built-in HTTP server as a simple web application and systemd to manage it.

Requirements:

* A Linux system using systemd.
* `python3`.
* `sudo` privileges.
* A disposable lab environment.

Do not perform this lab on a production server.

---

# 123. Lab Architecture

We will build:

```text
Browser / curl
      |
      v
TCP 8080
      |
      v
systemd
      |
      v
veriqta-web.service
      |
      v
python3 -m http.server
      |
      v
/opt/veriqta-web
      |
      v
index.html
```

This teaches:

* Service creation.
* Dedicated service users.
* Unit files.
* `systemctl`.
* `journalctl`.
* Ports.
* Permissions.
* Failure troubleshooting.
* Boot enablement.

---

# 124. Step 1: Verify systemd

Run:

```bash
ps -p 1 -o comm=
```

If the output is:

```text
systemd
```

continue.

Also:

```bash
systemctl --version
```

If systemd is not PID 1, this particular lab may not work as written.

This commonly happens inside some containers or WSL configurations.

---

# 125. Step 2: Verify Python

Run:

```bash
python3 --version
```

You should receive a Python 3 version.

Find the executable:

```bash
which python3
```

Possible result:

```text
/usr/bin/python3
```

We will use the real path in `ExecStart`.

---

# 126. Step 3: Create a Dedicated Service User

Create a system user for the application.

On many distributions:

```bash
sudo useradd --system --shell /usr/sbin/nologin veriqta-web
```

Paths and options can vary between distributions.

Check:

```bash
id veriqta-web
```

Why not run the service as root?

Because our simple web server does not need unrestricted root privileges.

This applies least privilege from Parts 4 and 5.

---

# 127. Step 4: Create the Web Directory

Run:

```bash
sudo mkdir -p /opt/veriqta-web
```

Create the page:

```bash
sudo nano /opt/veriqta-web/index.html
```

Add:

```html
<!DOCTYPE html>
<html>
<head>
    <title>VERIQTA Linux Lab</title>
</head>
<body>
    <h1>VERIQTA Linux Service Lab</h1>
    <p>The systemd-managed web service is working.</p>
</body>
</html>
```

Save and exit.

---

# 128. Step 5: Set Ownership

Run:

```bash
sudo chown -R veriqta-web:veriqta-web /opt/veriqta-web
```

Check:

```bash
ls -ld /opt/veriqta-web
ls -l /opt/veriqta-web
```

Why?

Our service will run as:

```text
veriqta-web
```

It needs appropriate access to its web files.

For this read-only example, broad write access is not required.

---

# 129. Step 6: Test the Application Manually

Before adding systemd, test the underlying command.

Run:

```bash
cd /opt/veriqta-web
```

Then:

```bash
sudo -u veriqta-web /usr/bin/python3 -m http.server 8080
```

You should see a message indicating it is serving HTTP.

Open another terminal and run:

```bash
curl http://127.0.0.1:8080
```

You should receive your HTML.

Return to the server command and press:

```text
Ctrl+C
```

Why test manually first?

Because we separate:

```text
Does the application command work?
```

from:

```text
Does the systemd service definition work?
```

This reduces troubleshooting complexity.

---

# 130. Step 7: Create the Unit File

Create:

```bash
sudo nano /etc/systemd/system/veriqta-web.service
```

Add:

```ini
[Unit]
Description=VERIQTA Beginner Web Service
After=network.target

[Service]
Type=simple
User=veriqta-web
Group=veriqta-web
WorkingDirectory=/opt/veriqta-web
ExecStart=/usr/bin/python3 -m http.server 8080
Restart=on-failure
RestartSec=3

[Install]
WantedBy=multi-user.target
```

Save and exit.

---

# 131. Understanding the Unit File

## `[Unit]`

```ini
Description=VERIQTA Beginner Web Service
```

Human-readable description.

```ini
After=network.target
```

Defines startup ordering relative to `network.target`.

It does not guarantee that every external network dependency is fully reachable.

---

# 132. Understanding `[Service]`

```ini
Type=simple
```

For this service, systemd treats the process started by `ExecStart` as the main service process.

```ini
User=veriqta-web
Group=veriqta-web
```

Runs the process with limited privileges.

```ini
WorkingDirectory=/opt/veriqta-web
```

Makes our web directory the process's working directory.

```ini
ExecStart=/usr/bin/python3 -m http.server 8080
```

Starts Python's HTTP server on port 8080.

```ini
Restart=on-failure
```

Requests restart after qualifying failures.

```ini
RestartSec=3
```

Waits before restarting.

---

# 133. Understanding `[Install]`

```ini
WantedBy=multi-user.target
```

This provides enablement information.

When we enable the service, systemd can create the relationship needed to pull it into the normal multi-user boot target.

---

# 134. Step 8: Reload systemd Unit Configuration

Run:

```bash
sudo systemctl daemon-reload
```

Why?

Because we added a new unit file.

Without reloading, systemd may not know about the new definition yet.

---

# 135. Step 9: Start the Service

Run:

```bash
sudo systemctl start veriqta-web
```

Check:

```bash
systemctl status veriqta-web
```

You should ideally see:

```text
Active: active (running)
```

---

# 136. Step 10: Verify With `is-active`

Run:

```bash
systemctl is-active veriqta-web
```

Expected:

```text
active
```

But remember:

```text
active
```

does not yet prove HTTP works.

---

# 137. Step 11: Check the Process

Run:

```bash
systemctl show veriqta-web -p MainPID
```

Suppose:

```text
MainPID=6000
```

Then:

```bash
ps -p 6000 -o pid,ppid,user,stat,%cpu,%mem,etime,cmd
```

Verify the process is running as:

```text
veriqta-web
```

not root.

---

# 138. Step 12: Check the Listening Port

Run:

```bash
sudo ss -ltnp | grep ':8080'
```

You should find a listener associated with Python, subject to process-information permissions.

This proves another layer:

```text
Service active
+
process running
+
port listening
```

---

# 139. Step 13: Test HTTP

Run:

```bash
curl http://127.0.0.1:8080
```

Expected response includes:

```html
<h1>VERIQTA Linux Service Lab</h1>
```

Now you have application-level verification.

---

# 140. Step 14: Inspect Logs

Run:

```bash
journalctl -u veriqta-web -n 50 --no-pager
```

Generate another request:

```bash
curl http://127.0.0.1:8080
```

Then check logs again.

You should see request-related output from Python's HTTP server.

---

# 141. Step 15: Follow Logs

Run:

```bash
journalctl -u veriqta-web -f
```

In another terminal:

```bash
curl http://127.0.0.1:8080
```

Watch the new log entry appear.

Press:

```text
Ctrl+C
```

to stop following the logs.

The web service continues running.

---

# 142. Step 16: Enable the Service

First:

```bash
systemctl is-enabled veriqta-web
```

It may say:

```text
disabled
```

Enable:

```bash
sudo systemctl enable veriqta-web
```

Check:

```bash
systemctl is-enabled veriqta-web
```

Expected:

```text
enabled
```

Remember:

The service was already running because we started it.

Enablement changes boot-time configuration.

---

# 143. Step 17: Inspect Enablement

Run:

```bash
ls -l /etc/systemd/system/multi-user.target.wants/ | grep veriqta
```

You may see a symbolic link pointing to:

```text
/etc/systemd/system/veriqta-web.service
```

This connects your symbolic-link knowledge to systemd enablement.

---

# 144. Step 18: Stop the Service

Run:

```bash
sudo systemctl stop veriqta-web
```

Check:

```bash
systemctl is-active veriqta-web
```

Expected:

```text
inactive
```

Now:

```bash
curl http://127.0.0.1:8080
```

should fail to connect if nothing else is listening there.

This demonstrates:

```text
enabled
```

does not mean:

```text
currently running
```

---

# 145. Step 19: Start It Again

Run:

```bash
sudo systemctl start veriqta-web
```

Verify:

```bash
systemctl is-active veriqta-web
curl http://127.0.0.1:8080
```

Both service state and application behavior should be checked.

---

# 146. Step 20: Restart the Service

Record the PID:

```bash
systemctl show veriqta-web -p MainPID
```

Then:

```bash
sudo systemctl restart veriqta-web
```

Check the PID again:

```bash
systemctl show veriqta-web -p MainPID
```

It will normally be a new process.

This connects service restart with process lifecycle.

---

# 147. Failure Lab 1: Wrong Executable

Back up your service file:

```bash
sudo cp /etc/systemd/system/veriqta-web.service \
/etc/systemd/system/veriqta-web.service.backup
```

Edit:

```bash
sudo nano /etc/systemd/system/veriqta-web.service
```

Change:

```ini
ExecStart=/usr/bin/python3 -m http.server 8080
```

to an intentionally wrong path:

```ini
ExecStart=/usr/bin/python-does-not-exist -m http.server 8080
```

Save.

Then:

```bash
sudo systemctl daemon-reload
sudo systemctl restart veriqta-web
```

It should fail.

---

# 148. Diagnose the Wrong Executable

Do not fix it immediately.

Run:

```bash
systemctl status veriqta-web
```

Then:

```bash
journalctl -u veriqta-web -n 50 --no-pager
```

Inspect:

```bash
systemctl cat veriqta-web
```

Verify the executable:

```bash
ls -l /usr/bin/python-does-not-exist
```

Now you have evidence:

```text
Configured executable path
        |
        v
File does not exist
        |
        v
Service cannot execute program
```

---

# 149. Repair Failure 1

Restore:

```bash
sudo cp /etc/systemd/system/veriqta-web.service.backup \
/etc/systemd/system/veriqta-web.service
```

Then:

```bash
sudo systemctl daemon-reload
sudo systemctl restart veriqta-web
```

Verify:

```bash
systemctl is-active veriqta-web
curl http://127.0.0.1:8080
```

---

# 150. Failure Lab 2: Wrong Working Directory

Edit:

```bash
sudo nano /etc/systemd/system/veriqta-web.service
```

Change:

```ini
WorkingDirectory=/opt/veriqta-web
```

to:

```ini
WorkingDirectory=/opt/directory-does-not-exist
```

Then:

```bash
sudo systemctl daemon-reload
sudo systemctl restart veriqta-web
```

Investigate:

```bash
systemctl status veriqta-web
journalctl -u veriqta-web -n 50 --no-pager
```

Check:

```bash
ls -ld /opt/directory-does-not-exist
```

Now repair it.

---

# 151. Failure Lab 3: Port Conflict

First restore the valid service and ensure it is stopped:

```bash
sudo systemctl stop veriqta-web
```

Manually occupy port 8080:

```bash
cd /tmp
python3 -m http.server 8080 &
```

Record the PID.

Verify:

```bash
ss -ltn | grep ':8080'
```

Now attempt:

```bash
sudo systemctl start veriqta-web
```

The service should fail because port 8080 is already in use.

---

# 152. Diagnose the Port Conflict

Check:

```bash
systemctl status veriqta-web
```

Then:

```bash
journalctl -u veriqta-web -n 50 --no-pager
```

Then:

```bash
sudo ss -ltnp | grep ':8080'
```

Identify the process.

Inspect:

```bash
ps -p PID -o pid,ppid,user,stat,etime,cmd
```

You now know:

```text
Service needs port 8080
        |
        v
Another process owns 8080
        |
        v
bind fails
        |
        v
service exits
```

---

# 153. Repair the Port Conflict

Stop only the manual Python process you created:

```bash
kill -TERM PID
```

Verify:

```bash
ss -ltn | grep ':8080'
```

If nothing is listening, start:

```bash
sudo systemctl start veriqta-web
```

Verify:

```bash
systemctl is-active veriqta-web
curl http://127.0.0.1:8080
```

---

# 154. Failure Lab 4: Permission Problem

Create a file the service user cannot read:

```bash
sudo touch /opt/veriqta-web/private.html
sudo chown root:root /opt/veriqta-web/private.html
sudo chmod 600 /opt/veriqta-web/private.html
```

Check:

```bash
ls -l /opt/veriqta-web/private.html
```

Request:

```bash
curl -i http://127.0.0.1:8080/private.html
```

Depending on Python's server behavior, you should see that the service cannot successfully serve the protected file.

This demonstrates that:

```text
service active
```

does not mean:

```text
service user can access every required resource.
```

---

# 155. Repair the Permission Problem

Do not use:

```bash
sudo chmod 777 /opt/veriqta-web/private.html
```

Instead decide what access is actually required.

For this lab, if the service user should own the file:

```bash
sudo chown veriqta-web:veriqta-web /opt/veriqta-web/private.html
sudo chmod 644 /opt/veriqta-web/private.html
```

Check:

```bash
ls -l /opt/veriqta-web/private.html
```

Then test again.

The exact permissions should always reflect the real security requirement.

---

# 156. Failure Lab 5: Crash and Restart

Create a deliberately failing script:

```bash
sudo nano /opt/veriqta-web/fail.sh
```

Add:

```bash
#!/bin/bash
echo "VERIQTA test service is starting"
sleep 2
echo "VERIQTA test service is failing"
exit 1
```

Make it executable:

```bash
sudo chmod 755 /opt/veriqta-web/fail.sh
```

Create:

```bash
sudo nano /etc/systemd/system/veriqta-fail.service
```

Add:

```ini
[Unit]
Description=VERIQTA Failure Demonstration Service

[Service]
Type=simple
ExecStart=/opt/veriqta-web/fail.sh
Restart=on-failure
RestartSec=3
```

---

# 157. Observe Restart Behavior

Reload:

```bash
sudo systemctl daemon-reload
```

Start:

```bash
sudo systemctl start veriqta-fail
```

Follow:

```bash
journalctl -u veriqta-fail -f
```

You should observe:

```text
start
failure
restart
start
failure
restart
```

Press:

```text
Ctrl+C
```

to stop following.

Then stop the intentionally failing service:

```bash
sudo systemctl stop veriqta-fail
```

This demonstrates why restart policies can create crash loops.

---

# 158. Cleaning Up the Failure Service

Remove the test unit:

```bash
sudo rm /etc/systemd/system/veriqta-fail.service
```

Then:

```bash
sudo systemctl daemon-reload
```

You can reset recorded failed state if necessary:

```bash
sudo systemctl reset-failed
```

This clears failed-state records. It does not fix underlying problems.

---

# 159. `reset-failed`

Suppose a service failed earlier but you have now repaired it.

systemd can retain failed state information.

Use:

```bash
sudo systemctl reset-failed SERVICE
```

or all failed units:

```bash
sudo systemctl reset-failed
```

Important:

```text
reset-failed
```

does not repair a broken service.

It only clears recorded failure state and related counters.

Fix the cause first.

---

# 160. Lab: Check Failed Units

Run:

```bash
systemctl --failed
```

If no failures exist, that is fine.

Do not deliberately break important system services just to populate this list.

Use only disposable practice services.

---

# 161. Lab: Inspect Service Dependencies

Run:

```bash
systemctl list-dependencies veriqta-web
```

Then:

```bash
systemctl list-dependencies multi-user.target
```

Find your service if it is enabled and represented through that target's dependency structure.

This helps connect:

```text
enablement
+
targets
+
dependencies
+
boot
```

---

# 162. Lab: Inspect the Unit

Run:

```bash
systemctl cat veriqta-web
```

Then:

```bash
systemctl show veriqta-web \
-p ActiveState \
-p SubState \
-p MainPID \
-p User \
-p FragmentPath
```

Identify:

* Active state.
* Sub-state.
* PID.
* User.
* Unit-file path.

---

# 163. Lab: Current Boot Logs

Run:

```bash
journalctl -b -n 50 --no-pager
```

Look at messages from the current boot.

Then:

```bash
journalctl --list-boots
```

If previous boot data exists:

```bash
journalctl -b -1 -n 50 --no-pager
```

Do not worry if previous boot logs are unavailable. Journal persistence depends on system configuration.

---

# 164. Lab: Boot Performance

Run:

```bash
systemd-analyze
```

Then:

```bash
systemd-analyze blame | head -n 20
```

Then:

```bash
systemd-analyze critical-chain
```

Do not change anything yet.

Your task is only to observe.

Ask:

```text
Which units took time?

Which units were on the critical chain?

Does the slowest individual unit automatically prove it caused total boot delay?
```

Answer:

No.

---

# 165. Lab: Full Web-Service Verification

For `veriqta-web`, perform:

```bash
systemctl status veriqta-web
```

Then:

```bash
systemctl is-active veriqta-web
```

Then:

```bash
systemctl is-enabled veriqta-web
```

Then:

```bash
systemctl show veriqta-web -p MainPID
```

Then:

```bash
sudo ss -ltnp | grep ':8080'
```

Then:

```bash
curl -I http://127.0.0.1:8080
```

Then:

```bash
journalctl -u veriqta-web -n 20 --no-pager
```

You just verified:

```text
Unit
 |
 v
Service state
 |
 v
Process
 |
 v
Port
 |
 v
Application
 |
 v
Logs
```

This is a much stronger verification than:

```text
systemctl says active.
```

---

# 166. Production Scenario 1: Service Will Not Start

Alert:

```text
payments-api.service failed
```

Begin:

```bash
systemctl status payments-api
```

Suppose:

```text
Active: failed
```

Then:

```bash
journalctl -u payments-api -n 100 --no-pager
```

Logs indicate:

```text
Unable to open configuration file
```

Next:

```bash
systemctl cat payments-api
```

Find the expected configuration path.

Then:

```bash
ls -l /path/to/config
```

Possible discovery:

```text
File exists but service user cannot read it.
```

Correct workflow:

```text
Alert
 |
 v
Status
 |
 v
Logs
 |
 v
Configuration path
 |
 v
Permissions
 |
 v
Fix least privilege
 |
 v
Start
 |
 v
Verify
```

Not:

```text
Alert
 |
 v
restart
 |
 v
restart
 |
 v
restart
```

---

# 167. Production Scenario 2: Service Is Active but Website Is Down

You run:

```bash
systemctl is-active nginx
```

Result:

```text
active
```

Do not stop.

Next:

```bash
sudo ss -ltnp | grep ':80'
```

Suppose port 80 is listening.

Then:

```bash
curl -I http://127.0.0.1
```

Suppose:

```text
HTTP/1.1 200 OK
```

Now you know:

```text
NGINX active
Port listening
Local HTTP working
```

The failure is probably elsewhere in the path.

Investigate:

```text
Load balancer
Firewall
Cloud security group
DNS
Routing
Reverse proxy path
Client-side connectivity
```

Restarting NGINX would have been an unsupported guess.

---

# 168. Production Scenario 3: Port Already in Use

Deployment completes.

Application fails.

Logs say:

```text
Address already in use
```

Run:

```bash
sudo ss -ltnp | grep ':8080'
```

You find:

```text
old-application
PID 7100
```

Now investigate:

```bash
ps -p 7100 -o pid,ppid,user,etime,cmd
```

Then determine whether:

* Old deployment was not stopped.
* Two services have the same port configuration.
* An unmanaged process is running.
* A duplicate service exists.

The port conflict is the symptom.

The root cause might be a deployment or configuration-management problem.

---

# 169. Production Scenario 4: Service Keeps Restarting

You run:

```bash
systemctl status api
```

and notice repeated restarts.

Check:

```bash
journalctl -u api --since "10 minutes ago"
```

Suppose:

```text
Missing environment variable DATABASE_URL
```

The problem is not:

```text
systemd restart policy.
```

The restart policy is exposing the application's repeated failure.

The root cause is:

```text
Missing required configuration.
```

Fix configuration first.

---

# 170. Production Scenario 5: Service Does Not Start After Reboot

The application works after:

```bash
sudo systemctl start myapp
```

but after every reboot it is absent.

Check:

```bash
systemctl is-enabled myapp
```

Suppose:

```text
disabled
```

The problem becomes clear.

The service can run, but boot enablement was never configured.

If appropriate:

```bash
sudo systemctl enable myapp
```

Then verify:

```bash
systemctl is-enabled myapp
```

---

# 171. Production Scenario 6: Enabled but Not Running

You run:

```bash
systemctl is-enabled myapp
```

Result:

```text
enabled
```

Then:

```bash
systemctl is-active myapp
```

Result:

```text
failed
```

Do not say:

```text
But it is enabled, so it should be running.
```

Enabled means it is configured to be pulled in during appropriate startup.

It can still fail.

Investigate:

```bash
systemctl status myapp
journalctl -u myapp -n 100 --no-pager
```

---

# 172. Common Beginner Mistakes

## Mistake 1: Confusing active and enabled

Active means runtime state.

Enabled means startup configuration.

## Mistake 2: Restarting without reading logs

You may repeatedly reproduce the same failure.

## Mistake 3: Using `kill -9` on a systemd service

systemd may simply restart it, and you bypass normal service-management procedures.

Use the service manager when appropriate:

```bash
sudo systemctl stop SERVICE
```

## Mistake 4: Editing package unit files directly

Prefer appropriate local overrides.

## Mistake 5: Forgetting `daemon-reload`

After changing unit files, systemd may still have the old configuration loaded.

## Mistake 6: Assuming `daemon-reload` restarts the application

It does not.

## Mistake 7: Assuming active means healthy

Test the application itself.

## Mistake 8: Ignoring ports

A running process can still fail to bind its required port.

## Mistake 9: Using `chmod 777` to fix permissions

Fix the specific ownership or permission requirement.

## Mistake 10: Assuming all distributions use identical service names

`ssh` vs `sshd`, `apache2` vs `httpd`, and other differences exist.

---

# 173. Service Management Command Reference

Check status:

```bash
systemctl status SERVICE
```

Start:

```bash
sudo systemctl start SERVICE
```

Stop:

```bash
sudo systemctl stop SERVICE
```

Restart:

```bash
sudo systemctl restart SERVICE
```

Reload:

```bash
sudo systemctl reload SERVICE
```

Enable:

```bash
sudo systemctl enable SERVICE
```

Disable:

```bash
sudo systemctl disable SERVICE
```

Enable and start:

```bash
sudo systemctl enable --now SERVICE
```

Disable and stop:

```bash
sudo systemctl disable --now SERVICE
```

Check active state:

```bash
systemctl is-active SERVICE
```

Check enablement:

```bash
systemctl is-enabled SERVICE
```

---

# 174. Inspection Command Reference

List running services:

```bash
systemctl list-units --type=service --state=running
```

List service unit files:

```bash
systemctl list-unit-files --type=service
```

Failed units:

```bash
systemctl --failed
```

View unit:

```bash
systemctl cat SERVICE
```

View properties:

```bash
systemctl show SERVICE
```

Dependencies:

```bash
systemctl list-dependencies SERVICE
```

Reload systemd unit configuration:

```bash
sudo systemctl daemon-reload
```

Clear failure state:

```bash
sudo systemctl reset-failed SERVICE
```

---

# 175. Journal Command Reference

Service logs:

```bash
journalctl -u SERVICE
```

Recent logs:

```bash
journalctl -u SERVICE -n 50
```

Follow logs:

```bash
journalctl -u SERVICE -f
```

Current boot:

```bash
journalctl -b
```

List boots:

```bash
journalctl --list-boots
```

Previous boot:

```bash
journalctl -b -1
```

Kernel logs:

```bash
journalctl -k
```

Recent time window:

```bash
journalctl --since "10 minutes ago"
```

Service plus time:

```bash
journalctl -u SERVICE --since "10 minutes ago"
```

No pager:

```bash
journalctl -u SERVICE -n 100 --no-pager
```

---

# 176. Networking Checks for Services

Listening TCP sockets:

```bash
ss -ltn
```

With process information when permitted:

```bash
sudo ss -ltnp
```

Specific port:

```bash
sudo ss -ltnp | grep ':8080'
```

Test HTTP:

```bash
curl http://127.0.0.1:8080
```

Headers only:

```bash
curl -I http://127.0.0.1:8080
```

These commands connect service management with networking.

---

# 177. Boot Command Reference

PID 1:

```bash
ps -p 1 -f
```

Default target:

```bash
systemctl get-default
```

Target dependencies:

```bash
systemctl list-dependencies multi-user.target
```

Boot timing:

```bash
systemd-analyze
```

Slow unit initialization:

```bash
systemd-analyze blame
```

Critical chain:

```bash
systemd-analyze critical-chain
```

---

# 178. What You Should Now Understand

Before continuing to Part 10, you should confidently understand:

* What a Linux service is.
* What a daemon is.
* The relationship between services and processes.
* Why servers need service managers.
* What systemd is.
* Why systemd commonly runs as PID 1.
* Why not every Linux environment uses systemd.
* What a systemd unit is.
* What a service unit is.
* Common unit types.
* How to use `systemctl`.
* How to find service names.
* How to check service status.
* What Loaded, Active, and Main PID mean.
* How to start a service.
* How to stop a service.
* How to restart a service.
* How reload differs from restart.
* Why not every service supports reload.
* What enabling means.
* What disabling means.
* Why start and enable are different.
* Why stop and disable are different.
* What `enable --now` does.
* What `disable --now` does.
* How `is-active` works.
* How `is-enabled` works.
* What static means at a beginner level.
* What masking means.
* How to list services.
* How to list service unit files.
* Basic service dependency concepts.
* The difference between dependency and ordering.
* `Wants=`.
* `Requires=`.
* `After=`.
* `Before=`.
* How to inspect dependencies.
* The basic Linux boot process.
* Firmware.
* BIOS and UEFI at a beginner level.
* Bootloaders.
* GRUB.
* The Linux kernel's role during boot.
* What initramfs does.
* How systemd enters the boot process.
* What targets are.
* `multi-user.target`.
* `graphical.target`.
* Rescue and emergency targets at a basic level.
* How to check the default target.
* How enabled services relate to boot.
* Why systemd boot is dependency-driven rather than simply sequential.
* What `systemd-analyze` does.
* What the systemd journal is.
* What `journalctl` does.
* How to view service-specific logs.
* How to follow logs.
* How to inspect current-boot logs.
* How to inspect previous-boot logs when available.
* How to inspect kernel logs.
* How to filter logs by time.
* Why application logs may exist separately from journal logs.
* How to find failed units.
* Why "failed" is a symptom, not the root cause.
* Where systemd unit files can exist.
* Why vendor unit files should not normally be edited directly.
* What `[Unit]`, `[Service]`, and `[Install]` mean.
* What `ExecStart=` does.
* What `User=` does.
* What `WorkingDirectory=` does.
* What `Restart=` does.
* Why restart policies can create crash loops.
* What `WantedBy=` does.
* What `daemon-reload` does.
* Why `daemon-reload` and service reload are different.
* How to troubleshoot a service that will not start.
* How to investigate missing executables.
* How to investigate permissions.
* How to investigate wrong directories.
* How to investigate applications that immediately crash.
* Why manually running an application is useful but not definitive.
* How environment variables affect services.
* Why secrets require secure handling.
* How port conflicts prevent service startup.
* How to identify a port owner with `ss`.
* Why you should not blindly kill the process owning a port.
* Why configuration validation matters.
* How NGINX configuration can be tested with `nginx -t`.
* Why configuration should often be validated before restart.
* Why an active service is not necessarily a healthy application.
* How to verify service state, process state, port state, application response, and logs separately.
* How to create a basic systemd service.
* How to run it as a dedicated user.
* How to intentionally break it safely.
* How to troubleshoot the failure.
* How to repair and verify it.

---

# 179. Hands-On Practice

Complete these tasks on a disposable Linux lab machine.

1. Identify PID 1.

2. Determine whether your machine uses systemd.

3. Display the systemd version.

4. List running services.

5. List installed service unit files.

6. Find the SSH service name on your distribution.

7. Check its status without modifying it.

8. Identify its Main PID.

9. Inspect that PID with `ps`.

10. Determine whether SSH is active.

11. Determine whether SSH is enabled.

12. Explain the difference.

13. Find one active but disabled service if your system has one.

14. Find one inactive but enabled service if your system has one.

15. List failed units.

16. Inspect the journal for one safe service.

17. Show only its latest 20 entries.

18. Follow its logs.

19. Stop following without stopping the service.

20. Inspect current-boot logs.

21. List available previous boots.

22. Inspect the previous boot if available.

23. Inspect kernel journal messages.

24. Display your default target.

25. List dependencies of the default target.

26. Explain `multi-user.target`.

27. Explain `graphical.target`.

28. Run `systemd-analyze`.

29. Run `systemd-analyze blame`.

30. Run `systemd-analyze critical-chain`.

31. Explain why the slowest unit does not automatically prove the root cause of slow boot.

32. Create `/opt/veriqta-web`.

33. Create a dedicated service user.

34. Create an HTML page.

35. Run Python's HTTP server manually as the service user.

36. Test it with `curl`.

37. Stop it.

38. Create `veriqta-web.service`.

39. Explain every directive in the unit.

40. Run `daemon-reload`.

41. Start the service.

42. Check its status.

43. Check `is-active`.

44. Check `is-enabled`.

45. Identify Main PID.

46. Inspect the process owner.

47. Verify port 8080.

48. Test HTTP.

49. Inspect service logs.

50. Follow service logs.

51. Generate HTTP requests and observe new logs.

52. Enable the service.

53. Inspect the resulting boot relationship.

54. Stop the service.

55. Confirm it is inactive but still enabled.

56. Start it again.

57. Restart it.

58. Compare the PID before and after restart.

59. Introduce an incorrect executable path.

60. Diagnose the resulting failure.

61. Repair it.

62. Introduce an incorrect working directory.

63. Diagnose the failure.

64. Repair it.

65. Create a port conflict.

66. Diagnose it using `ss`.

67. Identify the conflicting PID.

68. Inspect that process.

69. Resolve the conflict safely.

70. Verify the service works again.

71. Create a permission problem.

72. Diagnose it without using `777`.

73. Correct the smallest necessary permission problem.

74. Create the controlled crash-loop service.

75. Follow its logs.

76. Explain why it restarts.

77. Stop the test service.

78. Remove it.

79. Run `systemctl --failed`.

80. Explain the complete service troubleshooting workflow from memory.

---

# 180. Beginner Knowledge Check

1. What is a service?

2. What is a daemon?

3. Are all background processes daemons?

4. Why do Linux servers need service management?

5. What is systemd?

6. Does every Linux distribution use systemd?

7. What PID does systemd commonly have on a systemd-based host?

8. What is a systemd unit?

9. What is a service unit?

10. What does `.service` mean?

11. What is `systemctl`?

12. How do you check service status?

13. What does `Loaded` mean?

14. What does `Active` mean?

15. What does Main PID mean?

16. How do you start a service?

17. How do you stop a service?

18. How do you restart one?

19. How does reload differ from restart?

20. Does every service support reload?

21. What does enable mean?

22. Does enable necessarily start the service immediately?

23. What does disable mean?

24. Does disable necessarily stop the running service?

25. What does `enable --now` do?

26. What does `is-active` tell you?

27. What does `is-enabled` tell you?

28. Can a service be active but disabled?

29. Can a service be inactive but enabled?

30. What is a static unit?

31. What is a masked unit?

32. How does masking differ from disabling?

33. How do you list running services?

34. How do you list installed service unit files?

35. What is a service dependency?

36. What does `After=` mean?

37. Does `After=` by itself create a requirement dependency?

38. What does `Requires=` mean?

39. What does `Wants=` mean?

40. Why are ordering and dependency different concepts?

41. What are the major stages of Linux boot?

42. What does firmware do?

43. What are BIOS and UEFI?

44. What is a bootloader?

45. What is GRUB?

46. What happens when the kernel starts?

47. What is initramfs?

48. What does systemd do after userspace begins?

49. What is a target?

50. What is `multi-user.target`?

51. What is `graphical.target`?

52. What are rescue and emergency targets used for?

53. How do you check the default target?

54. How do enabled services relate to targets?

55. Does systemd always start services one at a time?

56. What does `systemd-analyze` do?

57. What does `systemd-analyze blame` show?

58. What is the systemd journal?

59. What is `journalctl`?

60. How do you view logs for one service?

61. How do you follow service logs?

62. How do you inspect current-boot logs?

63. How do you list recorded boots?

64. How do you inspect kernel logs?

65. Why would you filter logs by time?

66. Can applications have log files outside the journal?

67. How do you list failed units?

68. Does "failed" tell you the root cause?

69. Where can systemd unit files be stored?

70. Why should you avoid directly editing package-provided unit files?

71. What does `[Unit]` contain?

72. What does `[Service]` contain?

73. What does `[Install]` contain?

74. What does `ExecStart=` do?

75. What does `User=` do?

76. Why should services avoid root when unnecessary?

77. What does `WorkingDirectory=` do?

78. What does `Restart=on-failure` do?

79. Why can restart policies create crash loops?

80. What does `WantedBy=multi-user.target` mean in the context of enablement?

81. When should `daemon-reload` be used?

82. Does `daemon-reload` restart your application?

83. What is the difference between `daemon-reload` and `systemctl reload nginx`?

84. What should you check first when a service will not start?

85. Why should you check journal logs?

86. How can a wrong executable path break a service?

87. How can permissions break a service?

88. Why is `chmod 777` usually the wrong troubleshooting response?

89. How can a wrong working directory break startup?

90. Why can an application work manually but fail through systemd?

91. How can environment variables affect service startup?

92. Why should secrets be protected?

93. What is a port conflict?

94. What error might a port conflict produce?

95. How do you find the process listening on a TCP port?

96. Why should you inspect the port owner before killing it?

97. Why should configuration be validated before restarting a production service?

98. What does `nginx -t` do?

99. Does `systemctl is-active` prove that users can reach the application?

100. What additional checks should be performed?

---

# 181. Beginner Knowledge Check Answers

1. A managed program or workload that provides system or application functionality, often continuously.

2. A long-running background process that provides a function.

3. No.

4. Servers need reliable startup, shutdown, dependency handling, boot integration, monitoring, and process management.

5. A system and service manager used by many modern Linux distributions.

6. No.

7. PID 1 on typical systemd-based hosts.

8. An object representing a resource systemd manages.

9. A unit describing how systemd manages a service.

10. It identifies a service unit.

11. The main command-line interface for controlling and inspecting systemd.

12.

```bash
systemctl status SERVICE
```

13. systemd found and loaded the unit definition.

14. The unit's current runtime state.

15. The PID systemd considers the service's main process.

16.

```bash
sudo systemctl start SERVICE
```

17.

```bash
sudo systemctl stop SERVICE
```

18.

```bash
sudo systemctl restart SERVICE
```

19. Restart stops and starts the service. Reload asks a running service to reread configuration when supported.

20. No.

21. Configure the unit to participate in its normal startup relationships.

22. No.

23. Remove normal enablement relationships.

24. No.

25. Enables the unit and starts it now.

26. Its runtime active state.

27. Its enablement state.

28. Yes.

29. Yes.

30. A unit without normal `[Install]` enablement that may be activated through dependencies or another mechanism.

31. A unit prevented from normal activation.

32. Disable removes normal enablement. Masking prevents activation more strongly.

33.

```bash
systemctl list-units --type=service --state=running
```

34.

```bash
systemctl list-unit-files --type=service
```

35. A relationship between systemd units.

36. An ordering relationship.

37. No.

38. A stronger requirement relationship that pulls in another unit.

39. A weaker relationship requesting that another unit be started too.

40. A unit can require another unit without that statement alone specifying the complete startup order.

41. Firmware, bootloader, kernel, early userspace/initramfs where applicable, root filesystem, init/systemd, targets/services, usable system.

42. Initializes hardware and starts the boot chain.

43. Firmware interfaces, with UEFI being the modern architecture and BIOS the older model.

44. Software that loads the operating-system kernel and transfers control to it.

45. A common Linux bootloader.

46. It initializes core kernel functionality including memory, CPU, drivers, filesystems, and other system resources.

47. An early userspace environment used to prepare access to the real root filesystem.

48. It coordinates userspace initialization, units, services, and targets.

49. A unit used to group other units and represent synchronization points or system states.

50. A common multi-user non-graphical target used by servers.

51. A target associated with graphical system operation.

52. System recovery and maintenance.

53.

```bash
systemctl get-default
```

54. Enablement commonly establishes relationships that allow targets to pull services in during boot.

55. No. Independent units can start in parallel.

56. Reports boot timing information.

57. Unit initialization times.

58. systemd's centralized journal data store and logging infrastructure.

59. The command used to query journal data.

60.

```bash
journalctl -u SERVICE
```

61.

```bash
journalctl -u SERVICE -f
```

62.

```bash
journalctl -b
```

63.

```bash
journalctl --list-boots
```

64.

```bash
journalctl -k
```

65. To focus investigation on the period when an incident occurred.

66. Yes.

67.

```bash
systemctl --failed
```

68. No. It identifies the state, not necessarily the underlying cause.

69. Common locations include `/etc/systemd/system`, `/usr/lib/systemd/system`, and `/lib/systemd/system`, depending on the distribution.

70. Package updates may overwrite package-managed files. Local overrides are generally safer.

71. Metadata, ordering, and dependency relationships.

72. How the service process should run.

73. Enablement information.

74. Defines the command used to start the service.

75. Defines which user runs the service.

76. Least privilege reduces unnecessary access and potential impact.

77. Defines the process's working directory.

78. Requests restart after qualifying failures.

79. A permanent failure can cause repeated start, crash, restart cycles.

80. It allows enablement to associate the service with `multi-user.target`.

81. After adding or modifying systemd unit-file configuration.

82. No.

83. `daemon-reload` makes systemd reread unit configuration. Service reload asks a particular running application to reload its own configuration.

84. `systemctl status SERVICE`.

85. They often contain the actual error causing startup failure.

86. systemd cannot execute a program that does not exist at the configured path.

87. The service user may lack required read, write, execute, or directory traversal access.

88. It grants unnecessary permissions and ignores the real ownership/access requirement.

89. systemd may be unable to enter the configured directory.

90. systemd may use a different user, environment, working directory, PATH, permissions, and resource context.

91. Required application settings may be absent or incorrect.

92. Unauthorized users should not be able to read credentials or sensitive configuration.

93. Two services or processes attempting to use the same listening address, protocol, and port where sharing is not configured.

94. Errors such as `Address already in use`.

95.

```bash
sudo ss -ltnp
```

96. The process may be legitimate or critical, and killing it may not be the correct fix.

97. It reduces the risk of replacing a working service with one that cannot start because of invalid configuration.

98. It tests NGINX configuration.

99. No.

100. Check the process, listening socket, local application response, logs, dependencies, and the network path used by real clients.

---

# 182. Final Mental Model

You should now see Linux service management as several connected layers:

```text
                   Linux Boot
                       |
                       v
              Firmware / UEFI
                       |
                       v
                  Bootloader
                       |
                       v
                 Linux Kernel
                       |
                       v
                    systemd
                    PID 1
                       |
          +------------+------------+
          |            |            |
          v            v            v
       Targets      Services      Other Units
          |            |
          |            v
          |        Processes
          |            |
          |            v
          |          Ports
          |            |
          |            v
          +------> Applications
                       |
                       v
                     Users
```

When an application fails, do not think only:

```text
Restart the service.
```

Think:

```text
What is systemd reporting?
        |
        v
What do the logs say?
        |
        v
Did the process actually start?
        |
        v
Which user is running it?
        |
        v
Can it access its files?
        |
        v
Is its configuration valid?
        |
        v
Does it have its required environment?
        |
        v
Can it bind its port?
        |
        v
Are its dependencies available?
        |
        v
Is the service active?
        |
        v
Is the application actually responding?
        |
        v
Can real users reach it?
```

A beginner sees:

```text
Failed to start nginx.service
```

and repeatedly types:

```bash
sudo systemctl restart nginx
```

A Linux engineer sees the same failure and asks:

```text
Why did it fail?
```

Then works through:

```text
Status
   |
   v
Logs
   |
   v
Configuration
   |
   v
Permissions
   |
   v
Dependencies
   |
   v
Ports
   |
   v
Processes
   |
   v
Fix
   |
   v
Verify service
   |
   v
Verify application
```

That difference in reasoning is the foundation of real Linux service administration and prepares you for managing web servers, databases, Docker, Kubernetes nodes, monitoring agents, CI/CD runners, and production applications.
