# Part 20: Complete Linux Administration and Troubleshooting Project

## Final Practical Assessment

Parts 1 through 19 taught Linux as individual skills. You learned the filesystem, files, users, permissions, text processing, pipes, processes, services, packages, environment variables, networking, SSH, storage, performance, logs, Bash, automation, backups, and security.

Part 20 is different.

You will not learn those topics separately anymore.

You will operate one Linux server and use the skills together.

The project follows a realistic administration lifecycle:

```text
New Linux Server
       |
       v
Inspect and Baseline
       |
       v
Prepare Filesystem
       |
       v
Configure Users
       |
       v
Install Software
       |
       v
Configure Web Service
       |
       v
Configure Remote Access
       |
       v
Monitor
       |
       v
Automate
       |
       v
Back Up
       |
       v
Secure
       |
       v
Break Things Safely
       |
       v
Troubleshoot
       |
       v
Verify and Document
```

The objective is not to prove that you remember commands.

The objective is to prove that you understand how Linux behaves and can investigate a system when you do not immediately know the answer.

---

# 1. Project Scenario

You have joined a small engineering team.

The team gives you a new Linux server that will host a small internal application called:

```text
veriqta-app
```

The server has been created, but the engineering team has not prepared it.

You are responsible for:

* Inspecting the server.
* Understanding its current state.
* Creating the required filesystem structure.
* Creating users and groups.
* Configuring ownership and permissions.
* Installing software.
* Configuring a web server.
* Preparing remote SSH administration.
* Transferring application files.
* Monitoring system resources.
* Monitoring logs.
* Writing administrative scripts.
* Scheduling tasks.
* Creating backups.
* Hardening the server.
* Troubleshooting failures.
* Documenting the final environment.

Your target architecture is:

```text
Administrator Workstation
          |
          | SSH / SCP
          v
+----------------------------------+
|          Linux Server            |
|                                  |
|  SSH                             |
|   |                              |
|   +--> Administrator             |
|                                  |
|  Web Server                      |
|   |                              |
|   +--> veriqta-app               |
|                                  |
|  /opt/veriqta-app                |
|  /etc/veriqta-app                |
|  /var/log/veriqta-app            |
|  /var/backups/veriqta-app        |
|                                  |
|  systemd                         |
|  cron                            |
|  firewall                        |
|  system logs                     |
+----------------------------------+
```

This project should be completed on a disposable virtual machine or cloud lab server.

Do not intentionally break a production system.

---

# 2. Project Rules

During the project, follow five rules.

## Rule 1: Do not use root for everything

Use your regular administrative account.

Elevate only when necessary:

```bash
sudo command
```

Do not make:

```bash
sudo -i
```

your default working environment.

---

## Rule 2: Do not solve permissions with 777

Never use:

```bash
chmod -R 777 /
```

or blindly apply:

```bash
chmod 777 something
```

because access failed.

Investigate ownership and required access.

---

## Rule 3: Do not use kill -9 first

If a process needs termination, understand what it is and try graceful termination first.

---

## Rule 4: Do not delete unknown files to free space

Determine what is consuming storage before removing anything.

---

## Rule 5: Every fix requires verification

A command completing successfully does not prove the problem is solved.

Use:

```text
Change
  |
  v
Verify configuration
  |
  v
Verify service
  |
  v
Verify functionality
  |
  v
Check logs
```

---

# 3. Create an Administration Notebook

Before changing anything, create somewhere to record your findings.

In your home directory:

```bash
mkdir -p ~/linux-final-project
cd ~/linux-final-project
```

Create:

```bash
touch server-baseline.txt
touch change-log.txt
touch troubleshooting-notes.txt
```

Check:

```bash
ls -lh
```

You will use these throughout the assessment.

The goal is to develop the habit:

```text
Observe
→ Record
→ Change
→ Verify
→ Record
```

---

# Stage 1: Inspect the Server

Never administer an unfamiliar Linux server by immediately changing it.

First understand what you received.

# 4. Identify the Linux Distribution

Run:

```bash
cat /etc/os-release
```

Example:

```text
NAME="Ubuntu"
VERSION="24.04 LTS"
ID=ubuntu
```

Important fields can include:

* `NAME`
* `VERSION`
* `ID`
* `VERSION_ID`

Why this matters:

Package managers, service names, configuration locations, defaults, and support procedures can differ between distributions.

You should know whether you are working with Ubuntu, Debian, RHEL, Rocky Linux, AlmaLinux, Amazon Linux, Fedora, or another distribution.

Record it.

---

# 5. Identify the Kernel

Run:

```bash
uname -r
```

For additional information:

```bash
uname -a
```

Remember:

The distribution and kernel are not the same thing.

```text
Linux Distribution
       |
       +--> Linux kernel
       +--> utilities
       +--> package manager
       +--> libraries
       +--> system configuration
       +--> applications
```

Record the kernel version.

---

# 6. Check the Hostname

Run:

```bash
hostname
```

You can also use:

```bash
hostnamectl
```

where systemd provides it.

The hostname identifies the machine within many administrative and network contexts.

Record it.

---

# 7. Identify Your Current User

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

Determine:

* Username.
* UID.
* Primary GID.
* Supplementary groups.
* Whether the account has administrative privileges.

Check sudo:

```bash
sudo -l
```

Do not assume that because you can SSH into a machine, you automatically have administrative access.

---

# 8. Inspect CPU

Run:

```bash
lscpu
```

Identify:

* Architecture.
* CPU count.
* Cores.
* Threads.
* CPU model where exposed.

Also run:

```bash
nproc
```

This gives you the number of processing units available to the current environment.

Why does this matter?

Later, when you see:

```text
load average: 8.00
```

you need CPU capacity context.

A load of 8 means something different on a 2-CPU system than on a 32-CPU system.

---

# 9. Inspect Memory

Run:

```bash
free -h
```

Focus on:

```text
total
used
free
shared
buff/cache
available
```

Do not look only at `free`.

Linux deliberately uses otherwise idle memory for caches.

The `available` value is often more useful when asking how much memory can reasonably be made available to applications.

Check swap:

```bash
swapon --show
```

Record:

* Total RAM.
* Available RAM.
* Swap size.
* Current swap usage.

---

# 10. Inspect Storage

Run:

```bash
lsblk
```

Then:

```bash
df -h
```

Then:

```bash
df -i
```

You are investigating different things.

`lsblk` helps show block-device structure.

`df -h` shows filesystem capacity.

`df -i` shows inode usage.

Record:

* Root filesystem size.
* Root filesystem usage.
* Available storage.
* Important mount points.
* Inode usage.

---

# 11. Inspect Networking

Run:

```bash
ip addr
```

Then:

```bash
ip route
```

Look for:

* Network interfaces.
* IPv4 addresses.
* IPv6 addresses.
* Loopback interface.
* Default route.

Test basic connectivity where appropriate:

```bash
ping -c 4 1.1.1.1
```

Then test DNS separately:

```bash
getent hosts example.com
```

or:

```bash
dig example.com
```

if `dig` is installed.

This distinction matters.

If IP connectivity works but name resolution fails, you may have a DNS problem rather than a general network problem.

---

# 12. Inspect Listening Ports

Run:

```bash
sudo ss -lntup
```

Record:

```text
Protocol
Listening address
Port
Process
Purpose
```

Do not ignore unfamiliar ports.

Do not kill them either.

Investigate first.

---

# 13. Check Running Services

Run:

```bash
systemctl --type=service --state=running
```

Also check failed units:

```bash
systemctl --failed
```

For an unfamiliar service:

```bash
systemctl status service-name
```

The goal is to understand the initial server state before installing anything.

---

# 14. Stage 1 Baseline

Your `server-baseline.txt` should now contain something similar to:

```text
Hostname:
Distribution:
Distribution version:
Kernel:
Current user:
UID:
Administrative access:

CPU count:
Total memory:
Available memory:
Swap:

Root filesystem:
Root usage:
Root inode usage:

Primary network interface:
Server IP:
Default gateway:

Listening ports:
Running services:
Failed services:
```

This is your baseline.

Later, when something changes, you have something to compare against.

---

# Stage 2: Build the Directory Structure

The application requires separate locations for application files, configuration, logs, and backups.

We will use:

```text
/opt/veriqta-app
/etc/veriqta-app
/var/log/veriqta-app
/var/backups/veriqta-app
```

Why separate them?

Because Linux already has conventions for different types of data.

```text
/opt
Application/software content

/etc
Configuration

/var/log
Logs

/var/backups
Backup data on systems using this location
```

---

# 15. Create the Directories

Run:

```bash
sudo mkdir -p /opt/veriqta-app
sudo mkdir -p /etc/veriqta-app
sudo mkdir -p /var/log/veriqta-app
sudo mkdir -p /var/backups/veriqta-app
```

Why `-p`?

It creates parent directories where necessary and avoids an error if the requested directory already exists.

Verify:

```bash
ls -ld /opt/veriqta-app
ls -ld /etc/veriqta-app
ls -ld /var/log/veriqta-app
ls -ld /var/backups/veriqta-app
```

Do not configure permissions yet.

You have not created the application identity.

---

# Stage 3: Configure Users and Access

# 16. Why the Application Should Not Run as Root

Suppose the application is compromised.

If it runs as root:

```text
Application compromise
        |
        v
Root privileges
        |
        v
Potential system-wide impact
```

If it runs under a restricted application user:

```text
Application compromise
        |
        v
Restricted application account
        |
        v
Limited access
```

This is least privilege.

---

# 17. Create an Administration Group

Create:

```bash
sudo groupadd veriqta-admins
```

Verify:

```bash
getent group veriqta-admins
```

If the group already exists, investigate before recreating or changing it.

---

# 18. Create the Application User

For a service account, you generally do not need a normal interactive human login.

The exact user creation options vary by distribution and policy.

On a suitable lab system, for example:

```bash
sudo useradd --system \
  --home /opt/veriqta-app \
  --shell /usr/sbin/nologin \
  veriqta-app
```

Check:

```bash
id veriqta-app
```

Inspect:

```bash
getent passwd veriqta-app
```

Understand what you created.

This account exists so application processes and files can have an identity.

It is not intended to be your administrator login.

---

# 19. Add Your Administrator to the Group

Suppose your username is:

```text
ann
```

Use your actual lab username:

```bash
sudo usermod -aG veriqta-admins ann
```

Why `-aG`?

`-G` specifies supplementary groups.

`-a` appends instead of replacing existing supplementary group membership.

Forgetting `-a` can unintentionally alter group membership.

Verify:

```bash
id ann
```

A new login session may be required before your shell receives the new supplementary group membership.

---

# 20. Configure Ownership

Application directory:

```bash
sudo chown -R veriqta-app:veriqta-admins /opt/veriqta-app
```

Configuration:

```bash
sudo chown -R root:veriqta-admins /etc/veriqta-app
```

Logs:

```bash
sudo chown -R veriqta-app:veriqta-admins /var/log/veriqta-app
```

Backups:

```bash
sudo chown -R root:veriqta-admins /var/backups/veriqta-app
```

Now inspect:

```bash
ls -ld \
  /opt/veriqta-app \
  /etc/veriqta-app \
  /var/log/veriqta-app \
  /var/backups/veriqta-app
```

---

# 21. Configure Permissions

A simple lab policy might use:

```bash
sudo chmod 750 /opt/veriqta-app
sudo chmod 750 /etc/veriqta-app
sudo chmod 750 /var/log/veriqta-app
sudo chmod 750 /var/backups/veriqta-app
```

What does `750` mean?

```text
7 = owner = rwx
5 = group = r-x
0 = others = ---
```

But notice an important design question.

If `veriqta-app` owns the application directory and administrators need to modify files there, group permissions may need adjustment depending on the workflow.

Permissions should come from requirements, not from memorized numbers.

---

# 22. Test Access

As your administrator account:

```bash
ls /opt/veriqta-app
```

Test application-user access without opening an interactive login shell:

```bash
sudo -u veriqta-app ls /opt/veriqta-app
```

Test a location it should not be able to modify.

For example:

```bash
sudo -u veriqta-app touch /etc/veriqta-app/test-file
```

If your ownership and permissions are restrictive, this may fail.

That can be the correct result.

Security testing includes verifying that prohibited actions fail.

---

# Stage 4: Install and Manage Software

For the remainder of this example, assume Ubuntu or Debian.

If you use RHEL-family Linux, use the appropriate `dnf` workflow instead.

# 23. Update Package Metadata

Run:

```bash
sudo apt update
```

Remember:

```text
apt update
```

updates package metadata.

It does not mean all installed software has been upgraded.

---

# 24. Install Required Packages

Install useful lab packages:

```bash
sudo apt install nginx curl dnsutils zip unzip -y
```

Do not develop the habit of blindly adding `-y`.

In production, reviewing proposed package changes before accepting them can be important.

---

# 25. Verify Installation

Check:

```bash
command -v nginx
command -v curl
command -v dig
```

Then:

```bash
nginx -v
curl --version
dig -v
```

You are verifying:

```text
Package installed
       |
       v
Executable exists
       |
       v
Executable can run
       |
       v
Version identified
```

---

# 26. Locate Executables

Try:

```bash
which nginx
```

Then:

```bash
command -v nginx
```

Then:

```bash
whereis nginx
```

Understand the difference.

`command -v` tells you how the shell resolves the command.

`whereis` can locate related binary, source, and manual locations according to its search behavior.

---

# Stage 5: Configure a Web Service

We will use Nginx.

# 27. Understand the Service

The intended request flow is:

```text
Client
  |
  | HTTP
  v
Server IP:80
  |
  v
Nginx
  |
  v
Static application content
```

Before changing Nginx, inspect it.

---

# 28. Check Nginx Status

Run:

```bash
systemctl status nginx
```

If it is not running:

```bash
sudo systemctl start nginx
```

Verify:

```bash
systemctl is-active nginx
```

Expected:

```text
active
```

---

# 29. Enable Nginx at Boot

Run:

```bash
sudo systemctl enable nginx
```

Check:

```bash
systemctl is-enabled nginx
```

Expected:

```text
enabled
```

Remember:

```text
start
=
run now

enable
=
configure normal automatic startup
```

---

# 30. Test Locally

Run:

```bash
curl http://127.0.0.1
```

or:

```bash
curl http://localhost
```

If Nginx is serving correctly, you should receive HTTP content.

This proves:

```text
Nginx process
+
local networking
+
HTTP listener
```

are functioning.

It does not prove remote clients can reach the server.

---

# 31. Inspect the Listening Port

Run:

```bash
sudo ss -lntp | grep ':80'
```

You are looking for a listening socket associated with the web service.

Determine:

* Port.
* Address.
* Process.

---

# 32. Locate Nginx Configuration

Common Ubuntu/Debian locations include:

```text
/etc/nginx/nginx.conf
/etc/nginx/sites-available/
/etc/nginx/sites-enabled/
```

Inspect:

```bash
ls -l /etc/nginx
```

Then:

```bash
sudo less /etc/nginx/nginx.conf
```

Do not edit configuration before understanding its structure.

---

# 33. Create Application Content

Create:

```bash
sudo nano /opt/veriqta-app/index.html
```

Add:

```html
<!DOCTYPE html>
<html>
<head>
    <title>VERIQTA Linux Final Project</title>
</head>
<body>
    <h1>Linux Server Is Working</h1>
    <p>VERIQTA application server final project.</p>
</body>
</html>
```

Save.

Verify:

```bash
sudo cat /opt/veriqta-app/index.html
```

---

# 34. Configure Nginx for the Application

Create:

```bash
sudo nano /etc/nginx/sites-available/veriqta-app
```

Example:

```nginx
server {
    listen 80;
    listen [::]:80;

    server_name _;

    root /opt/veriqta-app;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

Now understand the important lines.

`listen 80`:

Nginx listens for IPv4 HTTP traffic on TCP port 80 according to the resulting socket configuration.

`listen [::]:80`:

Adds an IPv6 listener.

`root`:

Defines the directory containing content.

`index`:

Defines the default index file.

`location /`:

Controls requests beginning from the root URL.

---

# 35. Enable the Site

Create a symbolic link:

```bash
sudo ln -s \
  /etc/nginx/sites-available/veriqta-app \
  /etc/nginx/sites-enabled/veriqta-app
```

Check:

```bash
ls -l /etc/nginx/sites-enabled
```

You should see the symbolic link.

---

# 36. Validate Nginx Configuration

Before restarting:

```bash
sudo nginx -t
```

If configuration is valid, you should receive successful validation output.

If it fails, do not restart Nginx.

Investigate the reported file and line.

This pattern should now be familiar:

```text
Edit
 |
 v
Validate
 |
 v
Apply
 |
 v
Verify
```

---

# 37. Reload Nginx

If validation succeeds:

```bash
sudo systemctl reload nginx
```

Then:

```bash
systemctl status nginx
```

Test:

```bash
curl http://127.0.0.1
```

You should see your application content.

---

# 38. Inspect Web Logs

Common Nginx logs include:

```text
/var/log/nginx/access.log
/var/log/nginx/error.log
```

Inspect:

```bash
sudo tail /var/log/nginx/access.log
```

Then:

```bash
sudo tail /var/log/nginx/error.log
```

Follow requests live:

```bash
sudo tail -f /var/log/nginx/access.log
```

From another terminal:

```bash
curl http://127.0.0.1
```

Watch the request appear.

You have connected:

```text
Client request
      |
      v
Nginx
      |
      v
HTTP response
      |
      v
Access log entry
```

---

# Stage 6: Configure Remote Administration

# 39. Inspect the Server Address

Run:

```bash
ip addr
```

Identify the appropriate server IP.

Then:

```bash
ip route
```

Identify the default route.

If this is a cloud VM, remember that the private interface address may differ from the public address used from your workstation.

---

# 40. Verify DNS

Test:

```bash
getent hosts example.com
```

Then:

```bash
dig example.com
```

Then inspect:

```bash
cat /etc/resolv.conf
```

Do not modify `/etc/resolv.conf` blindly. On many modern systems it may be generated or managed by another resolver/networking component.

---

# 41. Check SSH

On Ubuntu:

```bash
systemctl status ssh
```

On some systems:

```bash
systemctl status sshd
```

Check:

```bash
sudo ss -lntp | grep ':22'
```

If SSH uses a different port, investigate the actual listener.

---

# 42. Generate an SSH Key on the Client

On your workstation:

```bash
ssh-keygen -t ed25519
```

You may be prompted for:

* File location.
* Passphrase.

This creates a private/public key pair.

Typical files:

```text
~/.ssh/id_ed25519
~/.ssh/id_ed25519.pub
```

Never copy the private key onto the server as an authentication setup method.

The server needs the public key.

---

# 43. Connect Remotely

From your workstation:

```bash
ssh user@server
```

or with an explicit private key:

```bash
ssh -i ~/.ssh/id_ed25519 user@server
```

After connecting:

```bash
whoami
hostname
pwd
```

Confirm you are on the expected machine.

---

# 44. Transfer Files With SCP

Create a local test file:

```bash
echo "VERIQTA deployment test" > deployment.txt
```

Copy:

```bash
scp deployment.txt user@server:/tmp/
```

On the server:

```bash
ls -l /tmp/deployment.txt
```

Then:

```bash
cat /tmp/deployment.txt
```

---

# 45. Move the Deployment File

Suppose the file belongs in the application directory.

Attempt:

```bash
mv /tmp/deployment.txt /opt/veriqta-app/
```

If permission is denied, investigate.

Do not immediately use `chmod 777`.

Check:

```bash
whoami
ls -ld /opt/veriqta-app
ls -l /tmp/deployment.txt
```

Then determine whether administrative elevation is appropriate.

For example:

```bash
sudo mv /tmp/deployment.txt /opt/veriqta-app/
```

After moving:

```bash
ls -l /opt/veriqta-app/deployment.txt
```

Check ownership.

If necessary, correct it according to your ownership model:

```bash
sudo chown veriqta-app:veriqta-admins \
  /opt/veriqta-app/deployment.txt
```

Verify again.

---

# Stage 7: Monitor the System

A server being "up" does not mean it is healthy.

You need multiple signals.

```text
Server Health
 |
 +--> Processes
 +--> CPU
 +--> Memory
 +--> Load
 +--> Storage
 +--> Inodes
 +--> Services
 +--> Network
 +--> Logs
```

---

# 46. Inspect Processes

Run:

```bash
ps aux
```

Search Nginx:

```bash
ps aux | grep nginx
```

A cleaner option:

```bash
pgrep -a nginx
```

Understand:

* PID.
* User.
* CPU usage.
* Memory usage.
* Command.

---

# 47. Monitor Interactively

Run:

```bash
top
```

If installed:

```bash
htop
```

Observe:

* CPU.
* Memory.
* Load.
* Process consumption.

Do not terminate anything yet.

Learn to observe first.

---

# 48. Check Load

Run:

```bash
uptime
```

Example:

```text
load average: 0.12, 0.18, 0.15
```

These represent approximately:

* 1 minute.
* 5 minutes.
* 15 minutes.

Interpret load relative to CPU capacity.

First:

```bash
nproc
```

A load of 4 on a four-CPU machine has a different meaning from a load of 4 on a one-CPU machine.

---

# 49. Check Memory

Run:

```bash
free -h
```

Then inspect high-memory processes:

```bash
ps aux --sort=-%mem | head
```

Do not assume low `free` memory automatically means memory exhaustion.

Look at:

* Available memory.
* Swap.
* Process usage.
* System behavior.

---

# 50. Check CPU Consumers

Run:

```bash
ps aux --sort=-%cpu | head
```

Compare with:

```bash
top
```

Ask:

```text
Which process?
How much CPU?
How long?
Expected or unexpected?
One process or system-wide?
```

---

# 51. Check Disk Space

Run:

```bash
df -h
```

If a filesystem is becoming full:

```bash
sudo du -xhd1 /var 2>/dev/null
```

Then progressively investigate the large directory.

For example:

```bash
sudo du -xhd1 /var/log 2>/dev/null
```

Do not start by deleting files.

Find the consumer first.

---

# 52. Check Inodes

Run:

```bash
df -i
```

A filesystem can have free bytes but no available inodes.

This often happens when enormous numbers of small files are created.

The symptoms may look like a full disk even though:

```bash
df -h
```

shows space remaining.

That is why:

```text
df -h
+
df -i
```

should both be part of storage troubleshooting.

---

# 53. Inspect System Logs

Run:

```bash
journalctl -p warning --since "1 hour ago"
```

Inspect boot logs:

```bash
journalctl -b
```

Inspect Nginx:

```bash
journalctl -u nginx
```

Follow:

```bash
journalctl -u nginx -f
```

Use logs as evidence.

---

# Stage 8: Automate Administration

Now you will combine Bash scripting with system monitoring.

# 54. Health-Check Script Objective

Create:

```text
/usr/local/bin/veriqta-health-check.sh
```

The script should report:

* Timestamp.
* Hostname.
* Uptime.
* Nginx status.
* Disk usage.
* Inode usage.
* Memory.
* Load average.
* Basic network listener information.

Results should be written to:

```text
/var/log/veriqta-app/health.log
```

---

# 55. Create the Script

Run:

```bash
sudo nano /usr/local/bin/veriqta-health-check.sh
```

Add:

```bash
#!/bin/bash

LOG_FILE="/var/log/veriqta-app/health.log"

{
    echo "========================================"
    echo "VERIQTA Linux Health Check"
    echo "Timestamp: $(date)"
    echo "Hostname: $(hostname)"
    echo

    echo "Uptime:"
    uptime
    echo

    echo "Nginx status:"
    if systemctl is-active --quiet nginx; then
        echo "nginx: running"
    else
        echo "nginx: NOT RUNNING"
    fi
    echo

    echo "Memory:"
    free -h
    echo

    echo "Disk:"
    df -h /
    echo

    echo "Inodes:"
    df -i /
    echo

    echo "Listening HTTP ports:"
    ss -lnt | grep -E ':(80|443)[[:space:]]' || \
        echo "No HTTP/HTTPS listener found"

    echo
} >> "$LOG_FILE" 2>&1
```

---

# 56. Understand the Script

Shebang:

```bash
#!/bin/bash
```

specifies Bash as the interpreter when the script is executed directly.

Variable:

```bash
LOG_FILE="/var/log/veriqta-app/health.log"
```

stores the output path.

Command substitution:

```bash
$(date)
```

runs `date` and inserts its output.

Conditional:

```bash
if systemctl is-active --quiet nginx
```

tests whether Nginx is active.

Redirection:

```bash
>> "$LOG_FILE" 2>&1
```

appends standard output to the log and directs standard error to the same destination.

---

# 57. Make It Executable

Run:

```bash
sudo chmod 750 /usr/local/bin/veriqta-health-check.sh
```

Inspect:

```bash
ls -l /usr/local/bin/veriqta-health-check.sh
```

Then execute:

```bash
sudo /usr/local/bin/veriqta-health-check.sh
```

Check:

```bash
sudo tail -n 50 /var/log/veriqta-app/health.log
```

---

# 58. Test Failure Handling

Stop Nginx in the disposable lab:

```bash
sudo systemctl stop nginx
```

Run:

```bash
sudo /usr/local/bin/veriqta-health-check.sh
```

Inspect:

```bash
sudo tail -n 30 /var/log/veriqta-app/health.log
```

You should see:

```text
nginx: NOT RUNNING
```

Restart:

```bash
sudo systemctl start nginx
```

Verify:

```bash
systemctl is-active nginx
```

You intentionally created a failure, observed it, and recovered.

---

# Stage 9: Schedule and Back Up

# 59. Define the Backup Requirement

You want to back up:

```text
/etc/veriqta-app
/opt/veriqta-app
```

into:

```text
/var/backups/veriqta-app
```

The archive should contain a timestamp.

Example:

```text
veriqta-app-2026-08-27-020000.tar.gz
```

---

# 60. Create a Manual Backup

Run:

```bash
sudo tar -czf \
  /var/backups/veriqta-app/veriqta-app-$(date +%Y%m%d-%H%M%S).tar.gz \
  /etc/veriqta-app \
  /opt/veriqta-app
```

You may see a message about removing the leading `/` from member names. Understand the message rather than assuming it is a failure.

---

# 61. Verify the Backup Exists

Run:

```bash
sudo ls -lh /var/backups/veriqta-app
```

Do not stop there.

List archive contents:

```bash
sudo tar -tzf /var/backups/veriqta-app/archive-name.tar.gz
```

Replace `archive-name.tar.gz` with the actual archive.

A backup file existing does not prove it contains what you expected.

---

# 62. Create a Backup Script

Create:

```bash
sudo nano /usr/local/bin/veriqta-backup.sh
```

Add:

```bash
#!/bin/bash

set -u

BACKUP_DIR="/var/backups/veriqta-app"
LOG_FILE="/var/log/veriqta-app/backup.log"
TIMESTAMP="$(date +%Y%m%d-%H%M%S)"
ARCHIVE="$BACKUP_DIR/veriqta-app-$TIMESTAMP.tar.gz"

mkdir -p "$BACKUP_DIR"

echo "$(date) Starting backup" >> "$LOG_FILE"

if tar -czf "$ARCHIVE" \
    /etc/veriqta-app \
    /opt/veriqta-app >> "$LOG_FILE" 2>&1; then

    echo "$(date) Backup created: $ARCHIVE" >> "$LOG_FILE"

    if tar -tzf "$ARCHIVE" > /dev/null 2>&1; then
        echo "$(date) Archive verification passed" >> "$LOG_FILE"
        exit 0
    else
        echo "$(date) ERROR: Archive verification failed" >> "$LOG_FILE"
        exit 2
    fi

else
    echo "$(date) ERROR: Backup creation failed" >> "$LOG_FILE"
    exit 1
fi
```

---

# 63. Why We Are Not Blindly Using set -e

You learned about:

```bash
set -e
```

in Part 17.

It can be useful, but its behavior has important context-dependent details.

Here we explicitly test the critical `tar` command:

```bash
if tar ...; then
```

This makes our intended error handling easier for a beginner to see.

---

# 64. Test the Backup Script

Set permissions:

```bash
sudo chmod 750 /usr/local/bin/veriqta-backup.sh
```

Run:

```bash
sudo /usr/local/bin/veriqta-backup.sh
```

Inspect:

```bash
sudo tail -n 30 /var/log/veriqta-app/backup.log
```

Check:

```bash
sudo ls -lh /var/backups/veriqta-app
```

---

# 65. Schedule With Cron

Edit root's crontab:

```bash
sudo crontab -e
```

For a lab, schedule a temporary frequent test such as:

```text
*/5 * * * * /usr/local/bin/veriqta-backup.sh
```

This runs every five minutes.

Do not keep a five-minute production backup schedule merely because it was useful for testing.

---

# 66. Why Absolute Paths Matter in Cron

Cron runs with a smaller environment than your interactive shell.

A script may work manually because your shell has:

```text
PATH
aliases
working directory
environment variables
```

that cron does not.

Use absolute paths when needed and do not assume cron starts from your normal working directory.

---

# 67. Verify Cron

After the expected execution time:

```bash
sudo ls -lh /var/backups/veriqta-app
```

Then:

```bash
sudo tail /var/log/veriqta-app/backup.log
```

Depending on distribution, cron events may be available through:

```bash
journalctl -u cron
```

or another cron service/log location.

The verification question is not:

```text
Did I add the cron line?
```

It is:

```text
Did cron execute it?

Did the script succeed?

Was the expected backup produced?
```

---

# Stage 10: Secure the Server

You now have a functioning server.

Next, review its security.

# 68. Review Users

Inspect:

```bash
cat /etc/passwd
```

Check relevant accounts:

```bash
id veriqta-app
id your-admin-user
```

Ask:

* Does the application account need interactive login?
* Does every human account still require access?
* Are unexpected accounts present?
* Does any account have unnecessary group membership?

---

# 69. Review sudo

Run:

```bash
sudo -l
```

Review administrative group membership:

```bash
getent group sudo
```

or:

```bash
getent group wheel
```

depending on distribution.

Apply least privilege.

---

# 70. Review Permissions

Check:

```bash
ls -ld \
  /opt/veriqta-app \
  /etc/veriqta-app \
  /var/log/veriqta-app \
  /var/backups/veriqta-app
```

Then:

```bash
find /opt/veriqta-app -maxdepth 2 -ls
```

Ask:

* Does the application own only what it should?
* Can ordinary users read configuration?
* Can the application modify configuration unnecessarily?
* Are backups exposed?
* Are logs writable only by appropriate identities?

---

# 71. Review SSH

Check:

```bash
systemctl status ssh
```

or:

```bash
systemctl status sshd
```

Inspect effective configuration where available:

```bash
sudo sshd -T | grep -E \
'port|permitrootlogin|passwordauthentication|pubkeyauthentication'
```

Before changing authentication, ensure key access works from a second terminal.

Keep the existing session open.

---

# 72. Review Listening Ports

Run:

```bash
sudo ss -lntup
```

For each listener ask:

```text
What process owns this?

Why is it running?

Does it need network access?

Does it need to listen on every interface?

Who should be able to reach it?
```

---

# 73. Configure Basic Firewall Rules

If using UFW, first inspect:

```bash
sudo ufw status verbose
```

If SSH is required:

```bash
sudo ufw allow OpenSSH
```

Allow HTTP:

```bash
sudo ufw allow 80/tcp
```

Allow HTTPS only if you actually configure HTTPS:

```bash
sudo ufw allow 443/tcp
```

Then carefully:

```bash
sudo ufw enable
```

Verify:

```bash
sudo ufw status verbose
```

From another terminal, verify SSH before closing your current session.

---

# 74. Review Updates

Run:

```bash
sudo apt update
```

Then:

```bash
apt list --upgradable
```

Understand what requires updating.

For production, follow the environment's maintenance process rather than blindly upgrading everything.

---

# 75. Review Logs

Inspect:

```bash
sudo journalctl -p warning --since today
```

Then:

```bash
sudo journalctl -u nginx --since today
```

Inspect authentication logs where available.

On Ubuntu:

```bash
sudo less /var/log/auth.log
```

Look for:

* Failed authentication.
* Successful authentication.
* Sudo activity.
* Service failures.
* Unexpected events.

---

# Stage 11: Troubleshooting Simulation

This is the most important stage.

You will intentionally introduce failures into your disposable environment and troubleshoot them.

For every incident use:

```text
OBSERVE
   |
   v
DEFINE THE SYMPTOM
   |
   v
DETERMINE SCOPE
   |
   v
GATHER EVIDENCE
   |
   v
FORM HYPOTHESIS
   |
   v
TEST HYPOTHESIS
   |
   v
IDENTIFY ROOT CAUSE
   |
   v
APPLY FIX
   |
   v
VERIFY RECOVERY
   |
   v
DOCUMENT
   |
   v
PREVENT RECURRENCE
```

Do not skip directly from error message to random commands.

---

# Incident 1: Permission Denied

## Symptom

You attempt:

```bash
echo "new configuration" > /etc/veriqta-app/app.conf
```

and receive:

```text
Permission denied
```

## Observe

Do not immediately run:

```bash
sudo chmod 777 /etc/veriqta-app
```

## Determine scope

Ask:

* Can I access the directory?
* Can I read existing files?
* Can I create files?
* Is this only my user?
* Is the application user affected?

## Gather evidence

```bash
whoami
id
ls -ld /etc/veriqta-app
ls -l /etc/veriqta-app
```

## Hypothesis

Your user does not have write permission.

## Test

Try:

```bash
touch /etc/veriqta-app/test
```

If it fails consistently, the evidence supports the hypothesis.

## Fix

Decide whether your user should actually have write access.

If configuration should only be changed administratively, use an appropriate privileged editor or controlled administrative workflow.

Do not widen permissions unnecessarily.

## Verify

Check:

```bash
ls -l /etc/veriqta-app
```

Confirm the intended operation works and unintended users remain restricted.

## Prevent

Document the ownership model.

---

# Incident 2: No Such File or Directory

## Symptom

```bash
cat logs/app.log
```

returns:

```text
No such file or directory
```

## Gather evidence

```bash
pwd
ls
ls -l logs
```

Search:

```bash
find /var/log/veriqta-app -name 'app.log'
```

## Possible root causes

* Wrong working directory.
* Wrong relative path.
* Typo.
* File does not exist.
* Directory renamed.
* Incorrect capitalization.

Remember Linux is case-sensitive.

```text
app.log
```

and:

```text
App.log
```

are different names.

## Fix

Use the correct path.

## Prevention

Understand absolute vs relative paths.

---

# Incident 3: command not found

## Symptom

```text
command not found
```

## Gather evidence

```bash
command -v command-name
which command-name
echo "$PATH"
```

If it should come from a package:

```bash
dpkg -l | grep package-name
```

or use your distribution's package query tools.

## Possible causes

* Package not installed.
* Typo.
* Executable not in PATH.
* Script lacks correct invocation.
* Environment differs.
* Cron has a different PATH.

## Root cause

Do not assume.

Prove it.

---

# Incident 4: Service Fails to Start

Suppose Nginx fails.

## Observe

```bash
sudo systemctl start nginx
```

fails.

## Gather evidence

```bash
systemctl status nginx
```

Then:

```bash
journalctl -u nginx -n 50
```

Then validate:

```bash
sudo nginx -t
```

## Possible root causes

* Configuration syntax.
* Missing file.
* Permission issue.
* Port conflict.
* Invalid directive.

## Fix

Correct the actual reported problem.

## Verify

```bash
sudo nginx -t
sudo systemctl start nginx
systemctl is-active nginx
curl http://127.0.0.1
```

---

# Incident 5: Service Repeatedly Crashes

A service can successfully start and then immediately terminate.

That differs from never starting.

## Investigate

```bash
systemctl status service-name
```

Then:

```bash
journalctl -u service-name -n 100
```

Check process:

```bash
pgrep -a process-name
```

Look for:

* Runtime errors.
* Missing dependencies.
* Missing files.
* Permission failures.
* Resource exhaustion.
* Bad environment variables.
* Application exceptions.

The root cause is usually in the evidence, not in repeatedly running:

```bash
systemctl restart service-name
```

---

# Incident 6: Port Already in Use

## Symptom

Nginx reports something similar to:

```text
Address already in use
```

## Reasoning

A TCP port/address combination generally cannot be owned by two ordinary listening services simultaneously in the same way.

Find the existing listener:

```bash
sudo ss -lntp | grep ':80'
```

You can also use, where installed:

```bash
sudo lsof -i :80
```

## Investigate

Determine:

* Which process?
* Which service?
* Why is it using port 80?
* Is it expected?

Do not immediately kill it.

## Fix

Depending on requirements:

* Stop the unnecessary service.
* Reconfigure one service.
* Change architecture.

## Verify

```bash
sudo ss -lntp | grep ':80'
sudo nginx -t
sudo systemctl start nginx
curl http://127.0.0.1
```

---

# Incident 7: Application Works Locally but Not Remotely

This is a classic Linux/cloud troubleshooting problem.

Locally:

```bash
curl http://127.0.0.1
```

works.

Remote browser does not.

This tells you something important.

The application itself may be functioning.

Now investigate the path:

```text
Remote Client
     |
     v
DNS
     |
     v
Public Network
     |
     v
Cloud Firewall
     |
     v
Host Firewall
     |
     v
Listening Socket
     |
     v
Nginx
```

Check:

```bash
ss -lntp
```

Then:

```bash
sudo ufw status
```

Then cloud/network controls where applicable.

Possible root causes:

* Service bound only to loopback.
* Host firewall.
* Cloud security group.
* Wrong IP.
* Wrong port.
* Routing.
* DNS.
* Upstream network controls.

Do not reinstall Nginx when local HTTP already proves Nginx works.

---

# Incident 8: SSH Authentication Fails

## Symptom

```text
Permission denied (publickey)
```

## Important distinction

This means something different from:

```text
Connection refused
```

and:

```text
Connection timed out
```

Authentication failure means you generally reached an SSH service but authentication was rejected.

## Client investigation

```bash
ssh -v user@server
```

Check:

* Username.
* Key being offered.
* Server.
* Port.

## Server investigation

From another valid session or console:

```bash
journalctl -u ssh -n 50
```

or:

```bash
journalctl -u sshd -n 50
```

Check:

```bash
ls -ld ~/.ssh
ls -l ~/.ssh/authorized_keys
```

Possible causes:

* Wrong username.
* Wrong private key.
* Missing public key.
* Wrong ownership.
* Incorrect permissions.
* SSH policy.
* Account restrictions.

Fix the proven cause.

---

# Incident 9: DNS Resolution Fails

## Symptom

```bash
ping -c 2 1.1.1.1
```

works.

But:

```bash
ping -c 2 example.com
```

fails.

This strongly suggests you should investigate name resolution.

Check:

```bash
getent hosts example.com
```

Then:

```bash
dig example.com
```

Inspect:

```bash
cat /etc/resolv.conf
```

Investigate:

* Configured DNS resolver.
* Reachability of resolver.
* Search domains.
* Local resolver service.
* Network configuration.

Do not change DNS configuration randomly.

---

# Incident 10: Disk Reaches 100 Percent

## Symptom

```bash
df -h
```

shows:

```text
100%
```

## Do not start with rm -rf

First identify where space is being consumed.

```bash
df -h
```

Identify the affected filesystem.

Then:

```bash
sudo du -xhd1 / 2>/dev/null
```

If `/var` is large:

```bash
sudo du -xhd1 /var 2>/dev/null
```

If `/var/log` is large:

```bash
sudo du -xhd1 /var/log 2>/dev/null
```

Investigate:

* Logs.
* Backups.
* Application data.
* Temporary files.
* Package caches.
* Container data if relevant.

Root cause example:

```text
Backup scheduled every 5 minutes
+
No retention
+
Same filesystem
=
Disk exhaustion
```

This connects automation, backups, storage, and troubleshooting.

---

# Incident 11: Inodes Exhausted

## Symptom

Applications report:

```text
No space left on device
```

But:

```bash
df -h
```

shows free capacity.

Check:

```bash
df -i
```

If inode usage is:

```text
100%
```

the filesystem cannot create more files even though bytes remain available.

## Investigate

Find directories containing huge numbers of files.

For targeted directories, use file counts carefully.

For example:

```bash
find /var/log/veriqta-app -xdev -type f | wc -l
```

Possible causes:

* Millions of tiny temporary files.
* Runaway logging.
* Application cache.
* Failed cleanup jobs.

Fix the generating process, not only the accumulated symptom.

---

# Incident 12: CPU Usage Spikes

## Observe

Users report that the application is slow.

Check:

```bash
uptime
```

Then:

```bash
top
```

Then:

```bash
ps aux --sort=-%cpu | head
```

Ask:

* Which process?
* How much CPU?
* Is load high?
* How many CPUs?
* Is the process expected?
* Is the spike temporary?
* Is there an application event corresponding to it?

Do not start by killing the highest CPU process.

It may be doing legitimate work.

---

# Incident 13: Memory Becomes Constrained

Check:

```bash
free -h
```

Then:

```bash
ps aux --sort=-%mem | head
```

Then:

```bash
top
```

Check swap:

```bash
swapon --show
```

Look for OOM evidence:

```bash
journalctl -k | grep -i -E 'oom|out of memory|killed process'
```

Possible causes:

* Memory leak.
* Excessive process count.
* Application workload.
* Incorrect memory configuration.
* Insufficient server capacity.

Do not treat cache as automatically wasted memory.

---

# Incident 14: Cron Works Manually but Not Automatically

This is extremely common.

Your script works:

```bash
sudo /usr/local/bin/veriqta-backup.sh
```

but cron does not produce a backup.

Investigate:

```bash
sudo crontab -l
```

Then cron logs:

```bash
journalctl -u cron
```

where applicable.

Check the script:

```bash
ls -l /usr/local/bin/veriqta-backup.sh
```

Check shebang.

Check absolute paths.

Check environment assumptions.

Common root causes:

* Different PATH.
* Different working directory.
* Missing environment variables.
* Permissions.
* Incorrect cron syntax.
* Script not executable.
* Relative file paths.
* Wrong user crontab.

The key lesson is:

```text
Interactive shell environment
!=
Cron environment
```

---

# Incident 15: Backup Script Fails

## Observe

No new archive appears.

Check:

```bash
sudo tail -n 100 /var/log/veriqta-app/backup.log
```

Then execute manually:

```bash
sudo /usr/local/bin/veriqta-backup.sh
```

Check exit code:

```bash
echo $?
```

Possible causes:

* Source directory missing.
* Backup directory missing.
* Permission denied.
* Disk full.
* Inodes exhausted.
* `tar` failure.
* Wrong path.
* Script syntax error.

Debug:

```bash
sudo bash -x /usr/local/bin/veriqta-backup.sh
```

Be careful when using `bash -x` with scripts containing real secrets because tracing can expose variable values.

---

# Incident 16: Incorrect Ownership Breaks the Application

Suppose someone runs:

```bash
sudo chown -R root:root /opt/veriqta-app
```

The application later cannot access something it needs.

## Observe

Check application/service logs.

Then:

```bash
ls -ld /opt/veriqta-app
find /opt/veriqta-app -maxdepth 2 -ls
```

Check service identity:

```bash
ps aux | grep relevant-process
```

or inspect its systemd unit.

Ask:

```text
Which user runs the process?

Which files must it read?

Which directories must it traverse?

Which files must it write?

Who owns those objects?
```

Fix ownership according to the intended model.

Do not simply make everything writable.

---

# 76. Troubleshooting Documentation Template

For every incident, write:

```text
Incident:
Date:
Server:
Reported symptom:

1. Observation
What exactly happened?

2. Scope
Who or what was affected?

3. Evidence
Commands:
Logs:
Metrics:
Configuration:

4. Hypotheses
What possible causes were considered?

5. Tests
How were those hypotheses tested?

6. Root cause
What actually caused the problem?

7. Fix
What was changed?

8. Verification
How was recovery confirmed?

9. Prevention
What would prevent recurrence?

10. Lessons learned
What should an administrator remember?
```

This converts troubleshooting from random command execution into engineering reasoning.

---

# Stage 12: Combined Failure Scenario

Now stop troubleshooting isolated errors.

You receive this report:

> "The application is down."

That is all.

You are not told the cause.

Your job is to investigate.

---

# 77. Start With Scope

Ask:

```text
Is the server reachable?

Can I SSH?

Is only the application affected?

Does localhost work?

Is DNS resolving?

Is the service running?

Is the machine resource constrained?
```

Start broad, then narrow.

---

# 78. Check Server Reachability

If you can SSH:

```bash
hostname
uptime
```

You now know the server itself is accessible.

If SSH fails, distinguish:

```text
Timeout
Refused
Authentication failure
DNS failure
```

These point toward different layers.

---

# 79. Check Application Locally

Run:

```bash
curl -v http://127.0.0.1
```

If this works:

```text
Application/web layer locally works
```

Investigate external networking.

If it fails:

```text
Investigate local service/application
```

This single test helps divide the problem.

---

# 80. Check Service State

```bash
systemctl status nginx
```

If failed:

```bash
journalctl -u nginx -n 100
```

Then:

```bash
sudo nginx -t
```

Do not restart repeatedly before understanding the failure.

---

# 81. Check Listening Socket

```bash
sudo ss -lntp | grep ':80'
```

If Nginx claims to be running but nothing is listening where expected, investigate configuration and process state.

---

# 82. Check Resources

Run:

```bash
free -h
df -h
df -i
uptime
```

Then:

```bash
ps aux --sort=-%cpu | head
ps aux --sort=-%mem | head
```

Now you have evidence about:

* Memory.
* Storage.
* Inodes.
* Load.
* CPU consumers.
* Memory consumers.

---

# 83. Check Logs

```bash
journalctl -u nginx -n 100
```

Then:

```bash
sudo tail -n 100 /var/log/nginx/error.log
```

Then application logs if present:

```bash
sudo tail -n 100 /var/log/veriqta-app/app.log
```

if that file exists.

Do not invent log locations. Confirm them.

---

# 84. Check Permissions

```bash
ls -ld /opt/veriqta-app
ls -l /opt/veriqta-app
```

Ask whether Nginx or the application process can traverse and read the required files.

Remember directory execute permission controls traversal.

A readable file inside an inaccessible parent directory may still be unusable.

---

# 85. Check Network Controls

If localhost works but remote access fails:

```bash
sudo ufw status verbose
```

Then check cloud firewall/security-group rules where applicable.

Also verify:

```bash
ip addr
ip route
```

and the actual destination clients use.

---

# 86. Check DNS

If users connect using:

```text
app.example.com
```

verify:

```bash
dig app.example.com
```

or:

```bash
getent hosts app.example.com
```

Ask:

```text
Does the hostname resolve?

Does it resolve to the correct address?

Can the client reach that address?
```

---

# 87. The Troubleshooting Layer Model

When an application fails, think in layers:

```text
User
 |
 v
DNS
 |
 v
Network
 |
 v
Firewall
 |
 v
Listening Port
 |
 v
Service
 |
 v
Application
 |
 v
Configuration
 |
 v
Filesystem
 |
 v
Permissions
 |
 v
Dependencies
 |
 v
CPU / Memory / Storage
 |
 v
Operating System
```

Start with evidence.

Move through the layers logically.

---

# Stage 13: Recovery Test

A backup you have never restored is only an assumption.

# 88. Create a Restore Directory

Do not overwrite the live application during the first test.

Create:

```bash
sudo mkdir -p /tmp/veriqta-restore-test
```

Choose one archive:

```bash
sudo ls -lh /var/backups/veriqta-app
```

Inspect it:

```bash
sudo tar -tzf /var/backups/veriqta-app/archive.tar.gz
```

Then extract safely into the test directory:

```bash
sudo tar -xzf \
  /var/backups/veriqta-app/archive.tar.gz \
  -C /tmp/veriqta-restore-test
```

Inspect:

```bash
sudo find /tmp/veriqta-restore-test -maxdepth 4 -ls
```

Verify expected application and configuration files exist.

---

# 89. What Backup Verification Really Means

There are several levels:

```text
Backup command returned success
            |
            v
Archive file exists
            |
            v
Archive can be listed
            |
            v
Archive can be extracted
            |
            v
Expected files exist
            |
            v
Application can actually be recovered
```

The last level matters most.

---

# Stage 14: Reboot and Persistence Test

A server can appear healthy until reboot.

This stage tests whether your configuration survives.

# 90. Before Reboot

Record:

```bash
systemctl is-enabled nginx
sudo crontab -l
sudo ufw status
ls -l /etc/nginx/sites-enabled
```

Make sure you have a safe recovery method.

Then, on a disposable lab server:

```bash
sudo reboot
```

Your SSH connection will close.

---

# 91. After Reboot

Reconnect.

Run:

```bash
uptime
```

Then:

```bash
systemctl is-active nginx
```

Then:

```bash
systemctl is-enabled nginx
```

Then:

```bash
curl http://127.0.0.1
```

Then:

```bash
sudo ss -lntp
```

Then:

```bash
sudo ufw status
```

Then verify cron and backups later.

This tests persistence.

---

# 92. Inspect Boot Problems

If something did not return after reboot:

```bash
systemctl --failed
```

Then:

```bash
journalctl -b -p warning
```

For a service:

```bash
journalctl -b -u service-name
```

This is why boot logs matter.

---

# Stage 15: Final Server Audit

Now pretend another administrator configured this server and you are reviewing their work.

Do not rely on memory.

Audit it.

# 93. System Audit

Check:

```bash
cat /etc/os-release
uname -r
hostname
uptime
```

Record the final state.

---

# 94. Identity Audit

Check:

```bash
whoami
id
getent passwd
getent group veriqta-admins
```

Review:

* Human users.
* Application user.
* Administrative group.
* Sudo access.

---

# 95. Filesystem Audit

Check:

```bash
ls -ld \
  /opt/veriqta-app \
  /etc/veriqta-app \
  /var/log/veriqta-app \
  /var/backups/veriqta-app
```

Review ownership and permissions.

---

# 96. Service Audit

Check:

```bash
systemctl status nginx
systemctl is-enabled nginx
systemctl --failed
```

Then:

```bash
sudo nginx -t
```

---

# 97. Network Audit

Run:

```bash
ip addr
ip route
sudo ss -lntup
```

Document every important listener.

---

# 98. Firewall Audit

Run:

```bash
sudo ufw status verbose
```

Compare allowed ports with listening ports.

Ask:

```text
Is every exposed port intentional?

Does every intentional service have an owner and purpose?
```

---

# 99. Resource Audit

Run:

```bash
nproc
free -h
df -h
df -i
uptime
```

Record current resource state.

---

# 100. Logging Audit

Check:

```bash
journalctl -p warning --since today
```

Then:

```bash
journalctl -u nginx --since today
```

Then:

```bash
sudo tail /var/log/veriqta-app/health.log
sudo tail /var/log/veriqta-app/backup.log
```

Confirm administrative automation is leaving evidence.

---

# 101. Backup Audit

Run:

```bash
sudo ls -lh /var/backups/veriqta-app
```

Inspect the latest archive.

Verify:

```bash
sudo tar -tzf /var/backups/veriqta-app/latest-archive.tar.gz
```

Then confirm you have successfully performed a restore test.

---

# 102. Automation Audit

Inspect:

```bash
sudo crontab -l
```

Inspect scripts:

```bash
sudo ls -l /usr/local/bin/veriqta-*.sh
```

Check:

* Ownership.
* Permissions.
* Shebang.
* Absolute paths.
* Logs.
* Error handling.

---

# 103. Security Audit

Review:

```text
Users
Groups
sudo
SSH
SSH keys
Permissions
Ownership
Listening ports
Firewall
Packages
Updates
Secrets
Logs
Backups
```

Security is not a separate feature.

It touches every stage of Linux administration.

---

# 104. Final Architecture

Your completed lab now resembles:

```text
                     Administrator
                          |
                     SSH / SCP
                          |
                          v
              +----------------------+
              |       Firewall       |
              +----------+-----------+
                         |
              +----------v-----------+
              |     Linux Server     |
              |                      |
              | SSH                  |
              | Nginx                |
              | systemd              |
              | cron                 |
              |                      |
              | /opt/veriqta-app     |
              | Application          |
              |                      |
              | /etc/veriqta-app     |
              | Configuration        |
              |                      |
              | /var/log/...         |
              | Logs                 |
              |                      |
              | /var/backups/...     |
              | Backups              |
              +----------+-----------+
                         |
                  Monitoring Data
                         |
             +-----------+-----------+
             |                       |
          Logs                    Metrics
```

You built this from Linux fundamentals rather than from a preconfigured platform.

---

# 105. Final Practical Examination

Now repeat the project without copying the previous commands line by line.

You receive another fresh Linux VM.

Your task is:

1. Identify the operating system and kernel.
2. Record the hostname.
3. Identify your current user and privileges.
4. Inspect CPU capacity.
5. Inspect memory and swap.
6. Inspect disks and mounted filesystems.
7. Inspect inode usage.
8. Inspect network interfaces.
9. Identify the default route.
10. Test basic connectivity.
11. Test DNS separately.
12. Inventory listening ports.
13. Inventory running services.
14. Check failed systemd units.
15. Create an application filesystem structure.
16. Create an application service account.
17. Create an administration group.
18. Configure group membership.
19. Configure ownership.
20. Configure least-privilege permissions.
21. Prove allowed access works.
22. Prove prohibited access fails.
23. Update package metadata.
24. Install a web server.
25. Locate its executable.
26. Check its version.
27. Find its configuration.
28. Create application content.
29. Configure the web service.
30. Validate configuration.
31. Start the service.
32. Enable it at boot.
33. Verify its state.
34. Test it locally.
35. Verify the listening socket.
36. Inspect service logs.
37. Verify DNS.
38. Configure SSH key access.
39. Test SSH from a second terminal.
40. Transfer a file using SCP.
41. Move the file into the appropriate location.
42. Correct ownership if required.
43. Inspect processes.
44. Inspect CPU usage.
45. Inspect memory.
46. Inspect load average.
47. Interpret load relative to CPU count.
48. Inspect storage capacity.
49. Inspect inode usage.
50. Inspect system logs.
51. Follow application logs live.
52. Create a health-check Bash script.
53. Add timestamps.
54. Check service state from the script.
55. Check memory from the script.
56. Check storage from the script.
57. Check inodes from the script.
58. Log the results.
59. Create an intentional service failure.
60. Confirm the script detects it.
61. Recover the service.
62. Create a backup script.
63. Create timestamped archives.
64. Verify archives.
65. Schedule backups.
66. Confirm cron actually executes them.
67. Review users.
68. Review administrative privileges.
69. Review filesystem permissions.
70. Review SSH.
71. Review authorized keys.
72. Review listening ports.
73. Configure the host firewall.
74. Preserve SSH access.
75. Allow only required application traffic.
76. Review package updates.
77. Review authentication logs.
78. Perform a restore test.
79. Reboot.
80. Confirm services return.
81. Confirm firewall state persists.
82. Confirm application functionality.
83. Confirm automation continues.
84. Troubleshoot a permission failure.
85. Troubleshoot a missing path.
86. Troubleshoot `command not found`.
87. Troubleshoot a failed service.
88. Troubleshoot a crashing service.
89. Troubleshoot a port conflict.
90. Troubleshoot remote application connectivity.
91. Troubleshoot SSH authentication.
92. Troubleshoot DNS.
93. Troubleshoot a full filesystem.
94. Troubleshoot inode exhaustion.
95. Troubleshoot high CPU.
96. Troubleshoot memory pressure.
97. Troubleshoot cron.
98. Troubleshoot backup failure.
99. Troubleshoot incorrect ownership.
100. Produce a final server audit and incident report.

If you can perform these tasks while explaining why you are doing them, you have moved beyond simply memorizing Linux commands.

---

# 106. What You Should Now Understand

After Parts 1 through 20, you should understand the Linux system as connected layers.

```text
Hardware
   |
   v
Linux Kernel
   |
   v
Processes and Memory
   |
   v
Filesystem and Storage
   |
   v
Users and Permissions
   |
   v
Services
   |
   v
Networking
   |
   v
Applications
   |
   v
Logs and Monitoring
   |
   v
Automation
   |
   v
Security and Recovery
```

You should understand how a problem in one layer can appear as a symptom somewhere else.

For example:

```text
Wrong ownership
      |
      v
Application cannot read configuration
      |
      v
Application crashes
      |
      v
systemd reports service failure
      |
      v
Port stops listening
      |
      v
Users receive connection errors
```

The user's symptom is:

```text
"The website is down."
```

But the root cause is:

```text
Incorrect Linux file ownership.
```

That relationship is the heart of systems troubleshooting.

---

# 107. Linux Commands You Should Now Be Comfortable With

You do not need to memorize every Linux command that exists.

You should, however, understand the purpose and basic use of commands including:

```bash
pwd
cd
ls
touch
mkdir
cp
mv
rm
find
file
ln

cat
less
head
tail
grep
wc
sort
uniq
cut
tr
sed
awk

whoami
id
groups
who
w

useradd
usermod
userdel
groupadd
passwd

chmod
chown
chgrp

ps
top
htop
pgrep
pkill
kill
jobs
bg
fg
nohup

systemctl
journalctl

apt
dpkg
dnf
rpm

env
printenv
export
which
command

ip
ping
ss
curl
wget
dig
host
nslookup

ssh
scp
sftp
ssh-keygen

lsblk
blkid
mount
umount
df
du

free
uptime

tar
gzip
zip
unzip

crontab

ufw

history
man
whatis
apropos
```

More important than memorizing this list is knowing:

```text
What information do I need?

Which Linux subsystem owns that information?

Which command can give me evidence?
```

---

# 108. The Final Troubleshooting Mindset

A beginner often troubleshoots like this:

```text
Error
 |
 v
Search Internet
 |
 v
Copy command
 |
 v
Run command
 |
 v
Hope
```

You should now be moving toward:

```text
Error
 |
 v
Observe exact symptom
 |
 v
Determine scope
 |
 v
Identify affected layer
 |
 v
Gather evidence
 |
 v
Read logs
 |
 v
Check current state
 |
 v
Form hypothesis
 |
 v
Test hypothesis
 |
 v
Identify root cause
 |
 v
Make smallest safe change
 |
 v
Verify recovery
 |
 v
Monitor
 |
 v
Document
 |
 v
Prevent recurrence
```

That difference matters far more than knowing hundreds of commands.

---

# 109. Final Knowledge Check

You should now be able to explain, without simply repeating definitions:

1. Why should you inspect a server before changing it?
2. How do you determine which Linux distribution you are using?
3. What is the difference between the distribution and kernel?
4. Why does CPU count matter when interpreting load average?
5. Why is `available` memory useful?
6. What is swap?
7. What is the difference between `df -h` and `df -i`?
8. Why can a filesystem report "No space left on device" while storage capacity remains?
9. What does `ip addr` tell you?
10. What does `ip route` tell you?
11. Why should IP connectivity and DNS be tested separately?
12. What does `ss -lntup` help investigate?
13. Why should you inventory running services?
14. Why should an application avoid running as root?
15. What is least privilege?
16. Why use a dedicated service account?
17. Why use groups?
18. What does `750` mean?
19. Why should permissions come from requirements rather than memorized numbers?
20. Why should denied access sometimes be considered a successful test?
21. What does `apt update` do?
22. How do you verify that software was actually installed?
23. What does `command -v` tell you?
24. What is a systemd service?
25. What is the difference between starting and enabling a service?
26. Why should configuration be validated before reload or restart?
27. What does a successful localhost test prove?
28. What does it not prove?
29. Why are listening addresses important?
30. What information can service logs provide?
31. What is DNS?
32. What does `/etc/resolv.conf` relate to?
33. Why should it not always be edited directly?
34. Why is SSH important for server administration?
35. What is the difference between public and private SSH keys?
36. Which SSH key belongs on the server?
37. Why must private keys be protected?
38. What does SCP do?
39. Why can transferred files have unexpected ownership?
40. How do you investigate a permission failure?
41. What is a PID?
42. How do you find CPU-intensive processes?
43. How do you find memory-intensive processes?
44. What is load average?
45. Why must load be interpreted relative to system capacity?
46. Why should you check both disk capacity and inodes?
47. Why are logs evidence?
48. What does `journalctl` provide?
49. Why is `tail -f` useful?
50. What is a Bash script?
51. What does a shebang do?
52. Why should administrative scripts produce logs?
53. What is an exit code?
54. Why should scripts handle failures?
55. What does `tar` do?
56. What does gzip compression add?
57. Why should backups use timestamps?
58. Why is archive creation not enough to prove backup success?
59. Why should backups be restored during testing?
60. What does cron do?
61. Why can a script work manually but fail under cron?
62. Why are absolute paths useful in scheduled jobs?
63. Why should cron execution be verified?
64. Why should user accounts be periodically reviewed?
65. Why should sudo privileges be reviewed?
66. Why should SSH configuration be reviewed?
67. Why should listening ports be compared with firewall rules?
68. Why should unnecessary services be removed or restricted?
69. Why must you protect SSH access before changing firewall rules remotely?
70. Why are updates a security concern?
71. Why are authentication logs important?
72. Why should security and backup strategy be connected?
73. What does `Permission denied` actually tell you?
74. What can cause `No such file or directory`?
75. What can cause `command not found`?
76. What evidence should you collect when a service fails?
77. Why should you not repeatedly restart a crashing service?
78. How do you investigate a port conflict?
79. What does local success plus remote failure suggest?
80. What is the difference between SSH authentication failure, connection refused, and timeout?
81. How can you separate DNS failure from general network failure?
82. What should you do first when a filesystem reaches 100 percent?
83. Why should you not immediately delete files?
84. What is inode exhaustion?
85. How do you investigate high CPU?
86. How do you investigate memory pressure?
87. What does an OOM event indicate?
88. How do you troubleshoot cron?
89. How do you troubleshoot a failed backup?
90. Why can `bash -x` be dangerous with secrets?
91. How can incorrect ownership break an application?
92. Why should you understand which user a process runs as?
93. Why does directory execute permission matter?
94. Why should every fix be verified?
95. Why should incidents be documented?
96. What is root cause?
97. What is the difference between a symptom and a root cause?
98. Why should recurrence prevention be part of troubleshooting?
99. Why is understanding Linux more valuable than memorizing commands?
100. What should your troubleshooting process be when somebody says only, "The server is down"?

---

# 110. Final Outcome of the 20-Part Linux Series

A learner who completes all 20 parts and genuinely performs the labs should have a strong beginner Linux foundation with an introduction to intermediate administration.

They should be able to:

* Navigate Linux confidently.
* Understand the Linux filesystem hierarchy.
* Work with absolute and relative paths.
* Create, copy, move, rename, find, link, and remove files.
* Create and manage directories.
* Understand users, groups, UIDs, and GIDs.
* Manage Linux users and groups.
* Understand root and sudo.
* Apply least privilege.
* Understand file and directory permissions.
* Use symbolic and numeric permissions.
* Manage ownership.
* Diagnose permission failures.
* View and edit text.
* Search files and logs.
* Use `grep`, `sed`, `awk`, `cut`, `sort`, `uniq`, and related tools at a foundational level.
* Understand standard input, output, and error.
* Use pipes and redirection.
* Build useful command pipelines.
* Understand processes.
* Inspect CPU and memory consumption by process.
* Manage foreground and background jobs.
* Understand signals.
* Manage systemd services.
* Understand the basic Linux boot process.
* Use `systemctl`.
* Investigate services with `journalctl`.
* Install and remove packages.
* Understand package repositories and dependencies.
* Understand shell and environment variables.
* Understand `PATH`.
* Troubleshoot `command not found`.
* Understand Linux networking fundamentals.
* Inspect interfaces and routes.
* Understand IP addresses, TCP, UDP, ports, clients, servers, and sockets.
* Test connectivity.
* Inspect listening ports.
* Understand DNS resolution.
* Troubleshoot DNS failures.
* Understand SSH.
* Authenticate using SSH keys.
* Administer remote Linux systems.
* Transfer files with SCP.
* Understand disks, partitions, filesystems, and mounts.
* Inspect disk usage.
* Understand inode exhaustion.
* Understand CPU, RAM, swap, and load.
* Investigate performance problems.
* Understand Linux logs.
* Use logs as troubleshooting evidence.
* Follow logs in real time.
* Write basic Bash scripts.
* Use variables, conditionals, loops, functions, command substitution, and exit codes.
* Debug basic Bash scripts.
* Automate administration.
* Schedule tasks with cron.
* Create archives.
* Create backups.
* Verify backups.
* Perform restore tests.
* Understand beginner Linux security.
* Protect SSH access.
* Review users and privileges.
* Review exposed services.
* Configure a basic host firewall.
* Protect secrets.
* Understand why patching matters.
* Investigate authentication activity.
* Baseline a new Linux server.
* Configure an application environment.
* Operate a web service.
* Monitor a server.
* Secure a server at a beginner level.
* Troubleshoot common Linux incidents systematically.
* Document changes and incidents.
* Verify recovery rather than assuming a command fixed the problem.

Most importantly, the learner should no longer see Linux as a collection of unrelated commands.

They should see a system:

```text
Users
  |
Permissions
  |
Files and Configuration
  |
Processes
  |
Services
  |
Networking
  |
Applications
  |
Logs
  |
Storage and Resources
  |
Automation
  |
Security
  |
Backups and Recovery
```

When something breaks, they should understand that these layers interact.

The final skill is therefore not:

```text
"I know Linux commands."
```

It is:

```text
"I can enter a Linux server,
understand what I am looking at,
find the information I need,
make controlled changes,
operate the system,
recognize when something is wrong,
gather evidence,
troubleshoot the problem,
verify the recovery,
and explain what happened."
```

That is the foundation the learner needs before moving into DevOps, Cloud, SRE, Platform Engineering, Docker, Kubernetes, CI/CD, Terraform, configuration management, observability, and infrastructure automation.
