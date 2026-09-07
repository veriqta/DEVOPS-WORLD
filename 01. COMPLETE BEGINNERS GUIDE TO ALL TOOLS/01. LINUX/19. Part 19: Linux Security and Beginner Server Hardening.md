# Part 19: Linux Security and Beginner Server Hardening

In the previous parts, you learned how to administer a Linux system. You can manage users, permissions, processes, services, packages, networking, SSH, storage, logs, scripts, automation, and backups.

Now you need to learn how to operate those systems safely.

Linux security is not about installing one security tool or running one hardening command. It is about reducing unnecessary access, protecting credentials, limiting exposed services, keeping software maintained, monitoring activity, and designing the server so that one mistake does not automatically become a complete system compromise.

By the end of this part, you should be able to take a new Linux server and perform a basic security review and beginner hardening process without blindly copying commands.

```text
New Linux Server
       |
       v
Understand the system
       |
       v
Review users and privileges
       |
       v
Secure authentication
       |
       v
Review permissions
       |
       v
Harden SSH
       |
       v
Review exposed services
       |
       v
Configure firewall
       |
       v
Install security updates
       |
       v
Protect secrets
       |
       v
Monitor logs
       |
       v
Maintain backups
       |
       v
Continuously review
```

Important: Perform the exercises in this part on a lab machine, VM, WSL environment where applicable, or disposable cloud server. Firewall and SSH configuration changes can disconnect you from a remote server.

# 1. Why Linux Security Matters

Linux powers a large amount of modern infrastructure.

You will find Linux running:

* Cloud virtual machines.
* Web servers.
* Application servers.
* Database servers.
* Kubernetes nodes.
* Container hosts.
* CI/CD runners.
* Monitoring systems.
* Build servers.
* Internal infrastructure.
* Network appliances.
* Development environments.

That means Linux systems frequently contain valuable resources.

A server might contain:

```text
Linux Server
|
+-- Application
|
+-- Customer data
|
+-- Configuration
|
+-- Credentials
|
+-- SSH keys
|
+-- Cloud permissions
|
+-- Database access
|
+-- Internal network access
```

Compromising one Linux server may therefore provide an attacker with more than access to that individual machine.

The server could become a path to other infrastructure.

For example:

```text
Compromised Server
       |
       +--> Application data
       |
       +--> Database credentials
       |
       +--> Cloud credentials
       |
       +--> Internal services
       |
       +--> Deployment systems
```

This is why Linux security matters to DevOps, SRE, Cloud, and Platform engineers.

---

# 2. Security Is a Process, Not One Command

There is no command such as:

```bash
secure-my-server
```

that permanently makes a Linux server secure.

Security involves many layers.

```text
Users
  |
  v
Authentication
  |
  v
Authorization
  |
  v
Permissions
  |
  v
Services
  |
  v
Network exposure
  |
  v
Software maintenance
  |
  v
Secrets
  |
  v
Logging
  |
  v
Backups
  |
  v
Continuous review
```

Even a well-configured server today can become less secure later.

For example:

1. A new service is installed.
2. The service starts listening publicly.
3. Nobody notices the new port.
4. The package becomes vulnerable.
5. Updates are ignored.
6. The service remains exposed.

Security therefore requires continued maintenance.

---

# 3. Three Questions to Keep Asking

As a beginner, use three questions whenever you make a system change:

```text
1. Does this need to exist?

2. Who needs access to it?

3. How much access do they actually need?
```

Suppose an application needs to read:

```text
/opt/myapp/config.yml
```

That does not automatically mean:

```bash
chmod 777 /opt/myapp/config.yml
```

The correct question is:

```text
Which user runs the application?

Which group should have access?

Does it need read access or write access?
```

Security starts with understanding requirements.

---

# 4. Principle of Least Privilege

Least privilege means giving a user, process, application, or service only the permissions necessary to perform its job.

Suppose:

```text
Application
needs
read access
to
configuration file
```

Then granting:

```text
read + write + execute
to
everyone
```

is unnecessary.

Think:

```text
Required access
      |
      v
Grant only that access
      |
      v
Nothing unnecessary
```

Least privilege applies to:

* Linux users.
* Groups.
* `sudo`.
* File permissions.
* Service accounts.
* SSH access.
* Database accounts.
* Cloud IAM.
* Containers.
* Kubernetes.
* CI/CD credentials.

It is one of the most important security concepts you will encounter.

---

# 5. Why Excessive Privilege Is Dangerous

Imagine an application runs as root.

The application becomes compromised.

The attacker's effective privileges may now be extremely powerful.

Compare:

```text
Compromised application
running as root
        |
        v
Potential root-level impact
```

with:

```text
Compromised application
running as restricted user
        |
        v
Impact constrained by
that user's permissions
```

Least privilege does not guarantee that attacks cannot happen.

It reduces what a successful compromise can do.

This concept is often called reducing the blast radius.

---

# 6. Understanding the Root Account

You learned in Part 4 that root is the Linux superuser.

Check your current user:

```bash
whoami
```

Check identity information:

```bash
id
```

The root user normally has:

```text
UID 0
```

Check:

```bash
id root
```

Typical output includes:

```text
uid=0(root)
```

Root can perform extremely powerful administrative operations.

That power is necessary for certain system tasks.

It is also dangerous.

---

# 7. Root-Account Risks

A command executed as root can affect the entire system.

For example, root can:

* Delete system files.
* Modify authentication configuration.
* Change firewall rules.
* Read protected files.
* Install software.
* Stop critical services.
* Change ownership.
* Create privileged users.
* Modify system logs.
* Change networking.
* Alter boot configuration.

A typo that produces:

```text
Permission denied
```

as a normal user may become destructive when executed as root.

This is one reason administrators should not use root casually.

---

# 8. Root Does Not Mean "Better"

Beginners sometimes develop this troubleshooting pattern:

```text
Command failed
     |
     v
Use sudo
     |
     v
Still failed
     |
     v
Become root
```

That is poor troubleshooting.

A permission failure is information.

If:

```bash
cat /some/file
```

returns:

```text
Permission denied
```

first investigate:

```bash
ls -l /some/file
```

Then:

```bash
ls -ld /some/directory
```

Ask:

* Who owns the file?
* Which group owns it?
* What are the permissions?
* Which user am I?
* Should my user actually have access?

Do not use root to bypass problems you do not understand.

---

# 9. What sudo Does

`sudo` allows an authorized user to execute a command with another user's privileges, commonly root privileges.

Example:

```bash
sudo apt update
```

Conceptually:

```text
Regular user
     |
     v
sudo
     |
     v
Authorization check
     |
     v
Allowed?
 /       \
Yes       No
 |         |
 v         v
Run       Reject
command
```

This is different from staying permanently logged in as root.

---

# 10. Why sudo Is Useful

Suppose you need administrative privileges only to install a package.

Instead of performing your entire session as root:

```text
root
 |
 +-- browse directories
 +-- edit files
 +-- run scripts
 +-- install package
 +-- delete files
```

you can remain a regular user:

```text
Regular user
 |
 +-- normal work
 |
 +-- sudo apt install nginx
         |
         v
     elevated only
     for required task
```

This better supports least privilege.

---

# 11. sudo Is Not Automatically Safe

Adding `sudo` does not make a dangerous command safe.

For example:

```bash
sudo rm -rf ...
```

can be extremely destructive.

`sudo` means:

```text
Execute with elevated privilege
```

not:

```text
Verify this command is sensible
```

Before using `sudo`, ask:

```text
Why does this command require elevated privileges?

What exactly will it change?

Can I verify the target first?
```

---

# 12. Check sudo Access

Depending on system configuration, you can inspect your permitted sudo operations with:

```bash
sudo -l
```

The command may request authentication.

It can show which commands your account is allowed to execute through `sudo`.

Do not assume every user should have unrestricted sudo privileges.

---

# 13. sudo Configuration

The primary sudo policy is commonly managed through:

```text
/etc/sudoers
```

and systems often use additional files under:

```text
/etc/sudoers.d/
```

Do not casually edit `/etc/sudoers` with a normal text editor.

A syntax error could break administrative access.

The traditional safe tool is:

```bash
sudo visudo
```

`visudo` performs syntax checking before installing changes.

For this beginner lesson, understand the principle before attempting complex sudo policies.

---

# 14. Avoid Giving Everyone Full sudo

A configuration that effectively allows every administrator to run everything as root creates a large privilege surface.

Real environments should ask:

```text
Does this user actually need sudo?

Which administrative operations are required?

Can access be restricted?
```

Least privilege should apply to administrative access too.

---

# 15. User-Account Hygiene

Security starts with knowing who can access the server.

Inspect user accounts:

```bash
cat /etc/passwd
```

But remember from Part 4:

`/etc/passwd` contains more than human users.

It also contains service and system accounts.

Do not delete accounts merely because you do not recognize their names.

---

# 16. Inspect Logged-In Users

Use:

```bash
who
```

and:

```bash
w
```

These help answer:

```text
Who is currently logged in?

From where?

What are they doing?
```

You can also inspect your own identity:

```bash
whoami
id
groups
```

---

# 17. Review Administrative Groups

On Ubuntu and Debian systems, users with sudo access are commonly associated with the:

```text
sudo
```

group.

Inspect:

```bash
getent group sudo
```

On some RHEL-family systems, administrative access is commonly associated with:

```text
wheel
```

Check:

```bash
getent group wheel
```

Do not assume group membership alone tells you the entire sudo policy. `/etc/sudoers` and `/etc/sudoers.d/` ultimately determine sudo authorization.

---

# 18. Remove Accounts That Are No Longer Required

Suppose an employee or temporary administrator no longer needs server access.

Leaving the account active creates unnecessary access.

Before deleting anything, investigate:

```bash
id username
```

Check:

* Does the account own files?
* Does it run services?
* Does it own cron jobs?
* Does it have SSH keys?
* Is it referenced by applications?
* Does it have active processes?

Removing accounts should be deliberate.

---

# 19. Locking an Account

On systems that support the expected `passwd` behavior, an administrator can lock password authentication for an account with:

```bash
sudo passwd -l username
```

Unlocking can be performed with:

```bash
sudo passwd -u username
```

But understand the limitation:

Locking a password does not necessarily disable every possible authentication method.

For example, SSH key access may still require separate review depending on configuration.

"Password locked" and "account completely unable to access the system" are not identical concepts.

---

# 20. Strong Authentication

Authentication answers:

```text
Who are you?
```

Authorization answers:

```text
What are you allowed to do?
```

These are different.

Examples of authentication mechanisms include:

* Passwords.
* SSH keys.
* Hardware-backed credentials.
* Multi-factor authentication.
* Centralized identity systems.

For this beginner lesson, we will focus primarily on passwords and SSH keys.

---

# 21. Password Security

Weak passwords increase risk.

Examples of poor password practices include:

* Reusing passwords.
* Sharing administrator passwords.
* Using predictable passwords.
* Storing passwords in scripts.
* Sending passwords through insecure channels.
* Using one shared Linux account for multiple people.

A better design gives administrators individual accounts.

That improves:

* Accountability.
* Access removal.
* Auditing.
* Privilege control.

---

# 22. File Permissions Are Security Controls

You learned Linux permissions in Part 5.

Now view them as security boundaries.

Example:

```text
-rw------- 1 ann ann secrets.conf
```

means:

```text
Owner:
read + write

Group:
none

Others:
none
```

This may be appropriate for a sensitive file that only its owner needs.

Compare:

```text
-rw-rw-rw-
```

Everyone can modify it.

For a sensitive configuration file, that could be dangerous.

---

# 23. Ownership Matters

Check:

```bash
ls -l file
```

Example:

```text
-rw-r----- 1 appuser appgroup app.conf
```

Interpretation:

```text
Owner = appuser
Group = appgroup

Owner:
read + write

Group:
read

Others:
nothing
```

The security question is not:

```text
Are these permissions restrictive?
```

It is:

```text
Are these the correct permissions for this file's purpose?
```

---

# 24. Sensitive Files

Examples of files or directories that may contain sensitive information include:

```text
/etc/shadow
~/.ssh/
/etc/ssh/
application configuration files
TLS private keys
cloud credentials
database credentials
backup archives
```

Do not assume every configuration file is harmless.

A configuration file might contain:

```text
DB_PASSWORD=...
API_KEY=...
PRIVATE_KEY=...
```

The sensitivity of a file depends on its contents and function.

---

# 25. Inspect Permissions Safely

For example:

```bash
ls -l ~/.ssh
```

You may see:

```text
id_ed25519
id_ed25519.pub
authorized_keys
known_hosts
```

These files do not all have the same security requirements.

The private key is especially sensitive.

---

# 26. SSH Security

SSH allows remote administration.

That makes it extremely useful.

It also means an SSH service may provide a remote path into the server.

Basic SSH security involves:

```text
Who can connect?

How do they authenticate?

Can root connect directly?

Which network can reach SSH?

Which keys are trusted?

How is the SSH server configured?
```

---

# 27. Review the SSH Service

Depending on the distribution, check:

```bash
systemctl status ssh
```

or:

```bash
systemctl status sshd
```

You can inspect listening sockets with:

```bash
ss -lntp
```

You may need appropriate privileges to see all process information.

A common SSH listener appears on TCP port:

```text
22
```

unless configured otherwise.

---

# 28. SSH Client vs SSH Server

Recall:

```text
Your computer
SSH client
     |
     | encrypted SSH connection
     v
Linux server
SSH server
```

The client initiates the connection.

The SSH server accepts and authenticates incoming connections.

Common client command:

```bash
ssh user@server
```

---

# 29. SSH Keys

SSH key authentication uses a key pair:

```text
Private key
     |
     | mathematically related
     |
Public key
```

The private key stays with the user.

The public key can be placed on the server.

Common generation command:

```bash
ssh-keygen
```

Modern OpenSSH commonly supports Ed25519 keys:

```bash
ssh-keygen -t ed25519
```

Follow the prompts.

A passphrase is generally a good protection for an interactive user's private key when practical.

---

# 30. Private Key vs Public Key

A common pair might look like:

```text
~/.ssh/id_ed25519
~/.ssh/id_ed25519.pub
```

The:

```text
id_ed25519
```

file is the private key.

The:

```text
id_ed25519.pub
```

file is the public key.

Never treat them as interchangeable.

The private key should remain private.

---

# 31. Protecting Private Keys

Check:

```bash
ls -l ~/.ssh/id_ed25519
```

A private key commonly uses restrictive permissions such as:

```text
-rw-------
```

which corresponds to:

```text
600
```

Set when appropriate:

```bash
chmod 600 ~/.ssh/id_ed25519
```

This means:

```text
Owner: read + write
Group: none
Others: none
```

OpenSSH may refuse to use private keys that are accessible too broadly.

---

# 32. Why Private-Key Protection Matters

Think of the private key as an authentication credential.

If an attacker obtains it and can use it successfully:

```text
Stolen private key
        |
        v
Attempt SSH authentication
        |
        v
Potential account access
```

A key passphrase can add another layer of protection if the key file is stolen.

But passphrases are not a substitute for protecting the key itself.

---

# 33. authorized_keys

A user's SSH public keys are commonly authorized through:

```text
~/.ssh/authorized_keys
```

Conceptually:

```text
Client private key
       |
       v
SSH authentication
       |
       v
Server checks
authorized_keys
       |
       v
Matching authorized
public key?
```

If an unauthorized public key appears there, someone may have created another path into the account.

That is why authorized keys should be reviewed.

---

# 34. Inspect Authorized Keys

Run as the relevant user:

```bash
cat ~/.ssh/authorized_keys
```

Do not remove a key simply because you do not immediately recognize it.

First determine:

* Who owns it?
* Is it used by automation?
* Is it used by an administrator?
* Is it required by a deployment system?

Then remove keys that are confirmed to be unauthorized or obsolete.

---

# 35. SSH Directory Permissions

Common OpenSSH setups expect restrictive permissions.

For a user's SSH directory, a common setting is:

```bash
chmod 700 ~/.ssh
```

For `authorized_keys`:

```bash
chmod 600 ~/.ssh/authorized_keys
```

However, ownership also matters.

Check:

```bash
ls -ld ~/.ssh
ls -l ~/.ssh/authorized_keys
```

If the files belong to the wrong user, permissions alone may not solve authentication problems.

---

# 36. Password Authentication Considerations

SSH can be configured to permit password authentication.

Advantages can include simpler initial setup.

Risks include exposure to:

* Password guessing.
* Credential reuse.
* Brute-force attempts.
* Stolen passwords.

Key-based authentication is commonly preferred for administrative server access when managed correctly.

But do not disable password authentication until you have proven that your alternative access works.

---

# 37. The Lockout Problem

Suppose you are connected remotely.

You change:

```text
PasswordAuthentication no
```

Then restart SSH.

But your key authentication does not actually work.

Your current session disconnects.

Now:

```text
Password login = disabled

Key login = broken
```

Result:

```text
You may be locked out.
```

This is one of the most important practical lessons in server hardening.

---

# 38. Safe SSH Configuration Workflow

Use this reasoning:

```text
Keep current session open
        |
        v
Configure key authentication
        |
        v
Open second terminal
        |
        v
Test new SSH login
        |
        v
Confirm sudo works
        |
        v
Validate SSH configuration
        |
        v
Only then disable unwanted method
        |
        v
Reload/restart carefully
        |
        v
Test again
```

Do not close your known-good session until the new configuration has been proven.

For cloud systems, know whether you have a provider console or other recovery path before changing remote-access controls.

---

# 39. SSH Server Configuration

OpenSSH server configuration is commonly stored in:

```text
/etc/ssh/sshd_config
```

Some installations also use additional configuration snippets, often under a directory such as:

```text
/etc/ssh/sshd_config.d/
```

Do not assume changing one line in the main file automatically determines the effective configuration. Included configuration files can matter.

Before editing:

```bash
sudo cp /etc/ssh/sshd_config /etc/ssh/sshd_config.backup
```

This provides a local copy, but remember that a configuration backup is useful only if you can still access the system to restore it.

---

# 40. Validate SSH Configuration

After editing, validate before restarting.

A common command is:

```bash
sudo sshd -t
```

If no syntax error is reported, the configuration passes this syntax check.

If `sshd` is not in your regular `PATH`, determine its location appropriately.

Do not restart SSH after a configuration change that fails validation.

---

# 41. Inspect Effective SSH Configuration

On many OpenSSH server installations:

```bash
sudo sshd -T
```

can display effective configuration values.

For example:

```bash
sudo sshd -T | grep -E 'passwordauthentication|permitrootlogin|pubkeyauthentication'
```

This can help answer what configuration SSH will actually use rather than relying only on what you think a file says.

Context-dependent `Match` rules can make effective configuration more complicated, so treat this as an introduction rather than the entire SSH policy model.

---

# 42. Root SSH Login Considerations

Direct remote root login creates a high-value authentication target.

A common administrative model is:

```text
SSH
 |
 v
Named regular user
 |
 v
sudo when required
```

instead of:

```text
SSH
 |
 v
root
```

This provides clearer user identity and supports least privilege.

---

# 43. PermitRootLogin

OpenSSH uses a configuration directive called:

```text
PermitRootLogin
```

Possible behavior depends on its value and OpenSSH version.

A hardened environment may choose:

```text
PermitRootLogin no
```

when administrators have a tested alternative method using named accounts and `sudo`.

Do not make this change blindly.

First prove:

1. Your regular user can connect.
2. Key authentication works if required.
3. The user has necessary sudo access.
4. You have a recovery path.

---

# 44. PasswordAuthentication

Another SSH directive is:

```text
PasswordAuthentication
```

For example:

```text
PasswordAuthentication no
```

can disable SSH password authentication.

This can reduce password-based attack exposure when key-based or another approved authentication mechanism is already functioning.

Again:

```text
Test alternative authentication first.
```

---

# 45. Do You Need to Change the SSH Port?

Beginners often encounter advice saying:

```text
Change port 22 and the server is secure.
```

Changing the port can reduce some automated noise and scanning against the default port.

It does not replace:

* Strong authentication.
* Key management.
* Least privilege.
* Firewall controls.
* Updates.
* Monitoring.

A service on port `2222` can still be vulnerable.

Port changes are not a substitute for actual security controls.

---

# 46. Exposed Network Services

Every listening network service deserves attention.

Run:

```bash
ss -lntup
```

Depending on privileges and platform, not all process information may be visible.

The important question is:

```text
What is listening?
```

You may see addresses such as:

```text
127.0.0.1:5432
0.0.0.0:22
0.0.0.0:80
[::]:443
```

These have important differences.

---

# 47. Listening Address Matters

Consider:

```text
127.0.0.1:5432
```

This indicates an IPv4 service bound to loopback.

Compare:

```text
0.0.0.0:5432
```

This generally indicates the service is listening on all IPv4 interfaces.

And:

```text
[::]:5432
```

may indicate listening on IPv6 interfaces, with exact dual-stack behavior depending on system configuration.

Security review requires checking both:

```text
Port
+
Listening address
```

---

# 48. Listening Does Not Automatically Mean Internet-Reachable

Suppose:

```bash
ss -lnt
```

shows:

```text
0.0.0.0:8080
```

That means the application is listening on local IPv4 interfaces.

It does not by itself prove that the Internet can reach port 8080.

Reachability may also depend on:

* Host firewall.
* Cloud security groups.
* Network ACLs.
* Router/firewall rules.
* Load balancers.
* Private/public addressing.
* NAT.
* Network routing.

Think:

```text
Application listens
       |
       v
Host firewall
       |
       v
Network controls
       |
       v
Routing
       |
       v
Potential client
```

All layers matter.

---

# 49. Identify the Process Behind a Port

Suppose you see:

```text
*:8080
```

and do not know what it is.

Try:

```bash
sudo ss -lntp
```

Look for process information.

You can also investigate processes with tools learned earlier:

```bash
ps aux
```

and services:

```bash
systemctl --type=service --state=running
```

Do not immediately kill an unfamiliar process.

Identify it first.

---

# 50. Remove Unnecessary Services

Every unnecessary service increases:

* Attack surface.
* Maintenance requirements.
* Configuration complexity.
* Potential vulnerabilities.

Suppose you identify a service that is genuinely not needed.

Check:

```bash
systemctl status service-name
```

If confirmed unnecessary, you might stop it:

```bash
sudo systemctl stop service-name
```

and prevent normal automatic startup:

```bash
sudo systemctl disable service-name
```

But understand the application first.

Do not disable services merely because their names are unfamiliar.

Some services are dependencies for other system functions.

---

# 51. Stop vs Disable

Recall:

```bash
sudo systemctl stop nginx
```

means:

```text
Stop it now.
```

While:

```bash
sudo systemctl disable nginx
```

means:

```text
Do not automatically enable/start it through its normal enabled boot links.
```

These are different operations.

You may sometimes need both, depending on your goal.

---

# 52. Uninstalling Unnecessary Software

If a package is truly unnecessary, removal may reduce attack surface and maintenance work.

On Ubuntu/Debian:

```bash
sudo apt remove package-name
```

But first investigate:

```bash
apt show package-name
```

and understand dependencies.

Never remove packages from a production server simply because they look unfamiliar.

---

# 53. Firewall Fundamentals

A firewall controls network traffic according to rules.

At a beginner level:

```text
Network packet
      |
      v
Firewall rules
      |
   +--+--+
   |     |
 Allow  Deny
   |     |
   v     X
Service
```

A firewall can decide whether certain traffic should be allowed to reach or leave a system.

---

# 54. Host Firewall vs Cloud Firewall

A cloud server may have multiple security layers.

For example:

```text
Internet
   |
   v
Cloud firewall/security group
   |
   v
Linux host firewall
   |
   v
Application
```

On AWS, for example, an EC2 instance may have security groups controlling network access outside the Linux operating system.

The Linux server may also have its own host firewall.

One does not automatically replace the other.

---

# 55. What Is UFW?

UFW stands for Uncomplicated Firewall.

It provides a simpler interface for managing host firewall rules and is especially common on Ubuntu systems.

Check whether it exists:

```bash
command -v ufw
```

If available:

```bash
sudo ufw status
```

Possible output:

```text
Status: inactive
```

or:

```text
Status: active
```

Do not enable it immediately on a remote server until you understand the existing rules and required access.

---

# 56. Check UFW Status Verbosely

Run:

```bash
sudo ufw status verbose
```

This may show:

* Whether UFW is active.
* Default policies.
* Existing rules.
* Logging information.

You can also inspect numbered rules:

```bash
sudo ufw status numbered
```

This is useful when reviewing or deleting specific rules.

---

# 57. Default Firewall Policy

A common beginner server policy is conceptually:

```text
Incoming traffic:
deny unless explicitly required

Outgoing traffic:
allow unless policy requires otherwise
```

With UFW:

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
```

Do not run these blindly on a production or remote server.

Understand existing network requirements first.

---

# 58. Allow SSH Before Enabling UFW

This is critical on a remotely administered server.

If SSH uses the expected OpenSSH application profile:

```bash
sudo ufw allow OpenSSH
```

or, where appropriate:

```bash
sudo ufw allow 22/tcp
```

But verify your SSH configuration first.

If SSH actually listens on another port, allowing port 22 will not protect your current remote access.

Check:

```bash
sudo ss -lntp
```

and inspect the SSH configuration.

---

# 59. Why TCP Matters

SSH uses TCP.

So:

```bash
sudo ufw allow 22/tcp
```

specifically permits TCP traffic to port 22.

This is more precise than treating every protocol identically.

Recall from Part 12:

```text
TCP
=
connection-oriented transport

UDP
=
connectionless transport
```

Different services use different protocols.

---

# 60. Allowing Web Traffic

A web server may need:

```text
80/tcp
443/tcp
```

HTTP:

```bash
sudo ufw allow 80/tcp
```

HTTPS:

```bash
sudo ufw allow 443/tcp
```

But only open ports required by your server.

Do not create rules "just in case."

---

# 61. Firewall Allowlisting

Suppose SSH should only be accessible from a trusted administrative IP.

UFW can express more restrictive rules.

Conceptually:

```text
Internet
   |
   +--> Trusted admin IP --> SSH allowed
   |
   +--> Everyone else ----> SSH denied
```

An example form is:

```bash
sudo ufw allow from 203.0.113.10 to any port 22 proto tcp
```

`203.0.113.10` is a documentation example address. Replace it only with a verified address appropriate to your environment.

Be careful with this strategy if your administrative public IP changes.

You can lock yourself out.

---

# 62. Enabling UFW

After confirming required rules:

```bash
sudo ufw enable
```

UFW may warn that enabling the firewall could disrupt existing SSH connections.

Take that warning seriously.

Before enabling on a remote server:

1. Verify the SSH port.
2. Add the required SSH rule.
3. Verify the rule.
4. Keep the current session open.
5. Know your recovery path.
6. Enable the firewall.
7. Open another terminal.
8. Test a new SSH connection.

---

# 63. Verify Firewall Rules

After enabling:

```bash
sudo ufw status verbose
```

Then test the actual services.

Do not assume:

```text
Firewall says ALLOW
```

means:

```text
Application definitely works.
```

You still need to test end-to-end connectivity.

---

# 64. Denying Traffic

UFW can explicitly deny traffic.

For example:

```bash
sudo ufw deny 8080/tcp
```

But if your default incoming policy already denies traffic, an explicit deny rule may be unnecessary unless you need the policy to be explicit or override another rule.

Understand your complete ruleset.

---

# 65. Removing a UFW Rule

A convenient method is:

```bash
sudo ufw status numbered
```

You may see:

```text
[ 1] 22/tcp ALLOW IN Anywhere
[ 2] 80/tcp ALLOW IN Anywhere
```

Then:

```bash
sudo ufw delete 2
```

Be careful because rule numbers can change after deletions.

Always run:

```bash
sudo ufw status numbered
```

again before deleting another numbered rule.

---

# 66. Do Not Use Firewall Rules to Hide an Unnecessary Service

Suppose a database listens publicly on:

```text
0.0.0.0:5432
```

but the firewall blocks it.

That is better than exposing it directly, but ask:

```text
Does the database need to listen on all interfaces at all?
```

Defense in depth may look like:

```text
Database
binds only where required
       +
Host firewall restricts traffic
       +
Cloud firewall restricts traffic
```

Do not rely unnecessarily on a single control.

---

# 67. Firewall Troubleshooting

Problem:

After enabling UFW, an application becomes unreachable.

Do not immediately disable the entire firewall.

Investigate.

Check application:

```bash
systemctl status service-name
```

Check listening port:

```bash
ss -lntp
```

Check firewall:

```bash
sudo ufw status numbered
```

Check application logs:

```bash
journalctl -u service-name
```

Reason:

```text
Is service running?
       |
       v
Is it listening?
       |
       v
Correct address?
       |
       v
Correct port?
       |
       v
Firewall permits it?
       |
       v
Cloud/network permits it?
       |
       v
Client can reach it?
```

---

# 68. Software Updates and Security

Software contains bugs.

Some bugs create security vulnerabilities.

Vendors and distribution maintainers release updated packages that may contain security fixes.

An unmaintained server can accumulate known vulnerabilities.

This creates a basic security responsibility:

```text
Know what is installed
       |
       v
Know whether updates exist
       |
       v
Evaluate updates
       |
       v
Apply safely
       |
       v
Verify services
```

---

# 69. Updating Package Metadata

On Debian/Ubuntu:

```bash
sudo apt update
```

This refreshes package metadata.

It does not mean:

```text
Every installed package has now been upgraded.
```

That distinction is important.

---

# 70. Reviewing Available Upgrades

On Ubuntu/Debian systems:

```bash
apt list --upgradable
```

can show available package upgrades.

Then you can evaluate what will change.

In managed production environments, updates should follow an appropriate patching and change-management process.

---

# 71. Installing Updates

A common command is:

```bash
sudo apt upgrade
```

But do not treat production patching as:

```text
Run upgrade everywhere immediately.
```

Consider:

* Maintenance windows.
* Application compatibility.
* Service restarts.
* Kernel updates.
* Rollback/recovery plans.
* Backups.
* Testing.
* High availability.

Beginners should understand both sides:

```text
Ignoring updates is risky.

Blindly changing production is also risky.
```

---

# 72. Security Updates

Linux distributions provide security advisories and security fixes through their package ecosystems.

The exact commands and mechanisms differ across:

* Ubuntu.
* Debian.
* Fedora.
* RHEL.
* Rocky Linux.
* AlmaLinux.
* Amazon Linux.

Do not copy an Ubuntu patching procedure to every distribution.

Identify the operating system first:

```bash
cat /etc/os-release
```

Then use the correct package-management process.

---

# 73. Reboots After Updates

Some updates affect components that may require service restarts or a reboot to fully use the updated code.

Examples can include:

* Kernel.
* Core libraries.
* Long-running services.

Do not assume:

```text
Package installed
=
Every running process is now using the new code
```

Production patching requires verification.

---

# 74. What Is a Secret?

A secret is sensitive information used to authenticate, authorize, encrypt, sign, or otherwise gain protected access.

Examples include:

* Passwords.
* API keys.
* Database passwords.
* Private keys.
* Access tokens.
* Cloud credentials.
* Signing keys.
* Encryption keys.

A value being stored in a configuration file does not make it non-sensitive.

---

# 75. Secrets in Configuration Files

Suppose:

```text
app.conf
```

contains:

```text
DB_USER=appuser
DB_PASSWORD=SuperSecret123
```

Anyone who can read that file may gain database credentials.

Therefore ask:

```text
Who owns the file?

Who can read it?

Does the application need write access?

Is this the correct place to store the secret?
```

---

# 76. Environment Variables and Secrets

Applications commonly receive configuration through environment variables.

Example:

```bash
export DB_HOST="db.internal"
```

A secret might also be provided through an environment variable:

```bash
export DB_PASSWORD="..."
```

This can be preferable to hardcoding secrets into application source code in some workflows, but environment variables are not automatically a secure secret-management system.

They can leak through:

* Debugging output.
* Logs.
* Process environments under certain access conditions.
* Shell history depending on how values are entered.
* CI/CD configuration.
* Crash reports.
* Misconfigured diagnostic tooling.

The important lesson is:

```text
Environment variable
!=
secret vault
```

---

# 77. Avoid Typing Secrets Directly Into Commands

Suppose you run:

```bash
export API_KEY="actual-secret-value"
```

Depending on shell behavior and how commands are entered, sensitive values can become part of shell history or other operational records.

For production systems, use an approved secret-management method rather than casually pasting secrets into terminals.

Examples of secret-management approaches include:

* Cloud secret managers.
* Dedicated secret-management systems.
* CI/CD secret stores.
* Protected files with controlled permissions.
* Platform-native secret mechanisms.

The exact solution depends on the environment.

---

# 78. Why Secrets Must Not Be Committed to Git

Suppose:

```text
config.env
```

contains:

```text
DATABASE_PASSWORD=...
AWS_SECRET_ACCESS_KEY=...
API_TOKEN=...
```

Then:

```bash
git add config.env
git commit -m "Add configuration"
git push
```

may publish those secrets to the repository.

Deleting the file in a later commit does not necessarily remove it from repository history.

Think:

```text
Secret
  |
  v
Git commit
  |
  v
Repository history
  |
  +--> Developer clones
  |
  +--> CI systems
  |
  +--> Mirrors
  |
  +--> Backups
```

Once exposed, assume the credential may have been copied.

---

# 79. .gitignore Is Helpful but Not Secret Management

You might add:

```text
.env
```

to:

```text
.gitignore
```

This helps prevent an untracked `.env` file from being accidentally added in ordinary workflows.

But `.gitignore`:

* Does not encrypt the file.
* Does not remove previously committed secrets.
* Does not rotate compromised credentials.
* Does not prevent every accidental disclosure.

It is a useful development safeguard, not a complete secret-management system.

---

# 80. What to Do If a Secret Is Committed

Do not merely delete the file and assume the problem is solved.

Treat the secret as potentially compromised.

A typical response includes:

```text
Discover exposed secret
       |
       v
Revoke/disable old credential
       |
       v
Generate replacement
       |
       v
Update authorized systems
       |
       v
Remove exposure where appropriate
       |
       v
Investigate usage/logs
       |
       v
Prevent recurrence
```

Rotation is critical because copies may already exist elsewhere.

---

# 81. Logs and Security Monitoring

Security controls cannot prevent every incident.

Logs provide evidence.

Useful questions include:

```text
Who logged in?

When?

From where?

Did authentication fail?

Was sudo used?

Did a service fail?

Did configuration change?

Did unusual network behavior occur?
```

Logs help you investigate.

---

# 82. Authentication Logs

Locations vary by distribution.

Ubuntu/Debian commonly use:

```text
/var/log/auth.log
```

RHEL-family systems commonly use:

```text
/var/log/secure
```

On systemd-based systems, authentication-related information may also be available through the journal.

Do not assume every distribution stores security logs in the same file.

---

# 83. Reading Authentication Logs

On a system using:

```text
/var/log/auth.log
```

you might inspect:

```bash
sudo less /var/log/auth.log
```

Search:

```bash
sudo grep "Failed password" /var/log/auth.log
```

You might also search SSH-related entries:

```bash
sudo grep sshd /var/log/auth.log
```

Interpret results carefully.

One failed login does not automatically prove compromise.

---

# 84. Using journalctl for SSH

Depending on the service name:

```bash
sudo journalctl -u ssh
```

or:

```bash
sudo journalctl -u sshd
```

You can inspect recent entries:

```bash
sudo journalctl -u ssh --since "1 hour ago"
```

Exact service names vary.

Check:

```bash
systemctl status ssh
```

or:

```bash
systemctl status sshd
```

first.

---

# 85. Failed-Login Investigation

Suppose you see many failed SSH attempts.

Do not immediately conclude:

```text
The server was hacked.
```

Instead ask:

```text
Were attempts unsuccessful?

Was there eventually a successful login?

Which username was targeted?

Which source addresses appeared?

When did attempts occur?

Was the account valid?

What authentication method succeeded?

Was anything unusual after login?
```

Security investigation requires evidence.

---

# 86. Failed Login vs Successful Compromise

These are not equivalent:

```text
Failed authentication
```

means an authentication attempt was rejected.

```text
Successful unauthorized access
```

means an attacker actually gained access.

Many Internet-facing SSH servers receive automated login attempts.

The correct response is to review:

* Authentication configuration.
* Successful logins.
* Source information.
* Account activity.
* Privilege use.
* Relevant logs.

---

# 87. Check Recent Login History

A common command is:

```bash
last
```

It reads login history information and can help show previous sessions.

You may also use:

```bash
lastlog
```

where available.

Treat these as evidence sources, not infallible truth. Logs can be rotated, incomplete, or manipulated after a sufficiently privileged compromise.

---

# 88. Investigating sudo Activity

On systems where sudo events are recorded in the system journal or authentication logs, search for sudo activity.

For example:

```bash
sudo journalctl | grep sudo
```

or on Ubuntu:

```bash
sudo grep sudo /var/log/auth.log
```

You may find information about:

* User.
* Command.
* Time.
* Authentication.

This demonstrates another benefit of named user accounts plus sudo.

Actions can be easier to attribute than when everyone directly uses root.

---

# 89. Security Monitoring Is Not "Read Every Log Manually"

Production systems may generate enormous amounts of log data.

Real environments often use:

* Centralized logging.
* SIEM systems.
* Alerts.
* Metrics.
* Intrusion detection.
* Cloud security services.
* Automated detection.

But the beginner foundation remains:

```text
Know which evidence exists.

Know where to find it.

Know what normal behavior looks like.

Investigate deviations.
```

---

# 90. Secure Configuration Habits

A secure engineer develops habits before incidents occur.

Good habits include:

* Back up configuration before changing it.
* Understand defaults.
* Read documentation.
* Change one thing at a time when troubleshooting.
* Validate configuration syntax.
* Test before restarting services.
* Keep known-good remote sessions open.
* Use version control for appropriate configuration.
* Never commit secrets.
* Use least privilege.
* Document intentional exposure.
* Remove obsolete access.
* Review logs after security-sensitive changes.

---

# 91. Validate Before Restarting

You saw this with SSH:

```bash
sudo sshd -t
```

Many services have their own configuration validation commands.

The general pattern is:

```text
Edit configuration
       |
       v
Validate
       |
   +---+---+
   |       |
Valid    Invalid
   |       |
   v       v
Reload   Fix first
```

This can prevent avoidable outages.

---

# 92. Reload vs Restart

If a service supports configuration reloads:

```bash
sudo systemctl reload service-name
```

may apply configuration without completely stopping the process.

A restart:

```bash
sudo systemctl restart service-name
```

stops and starts the service.

Not every service supports reload.

Check the service's documentation and behavior.

Security changes should not be performed blindly just because a command exists.

---

# 93. Backups Are Part of Security

Security is not only about preventing unauthorized access.

Consider ransomware, malicious deletion, accidental deletion, or destructive configuration changes.

A secure system should be recoverable.

```text
Security
 |
 +--> Prevention
 |
 +--> Detection
 |
 +--> Response
 |
 +--> Recovery
```

Backups support recovery.

That is why Part 18 connects directly to Part 19.

---

# 94. Backups Need Security Too

A backup may contain:

* `/etc` configuration.
* Application secrets.
* Customer data.
* SSH configuration.
* Database exports.
* Certificates.

An attacker who cannot read the production file but can freely download the backup may still obtain the information.

Protect:

* Backup permissions.
* Backup credentials.
* Backup storage.
* Backup encryption where required.
* Retention.
* Access logs.

---

# 95. Backups Should Not Share Every Failure

If an attacker gains root access and can delete:

```text
Production data
+
all backups
```

from the same server, your recovery options may disappear.

This is another reason for separate backup locations and failure domains.

A stronger model looks like:

```text
Production server
       |
       v
Protected backup system
       |
       v
Additional protected/off-site copy
```

The exact architecture depends on the environment.

---

# 96. Beginner Server-Hardening Methodology

You now have enough knowledge to create a repeatable hardening workflow.

Do not begin with random commands from an Internet checklist.

Start by understanding the server.

```text
1. Identify system
2. Establish recovery access
3. Review users
4. Review privilege
5. Review authentication
6. Review permissions
7. Review SSH
8. Review listening services
9. Review firewall
10. Review packages and updates
11. Review secrets
12. Review logs
13. Review backups
14. Verify everything
```

---

# 97. Step 1: Identify the Server

Run:

```bash
cat /etc/os-release
```

Then:

```bash
uname -a
```

Then:

```bash
hostname
```

Then:

```bash
whoami
```

Then:

```bash
id
```

You should know:

```text
Distribution
Kernel
Hostname
Current user
Current privileges
```

before changing the system.

---

# 98. Step 2: Understand the Server's Purpose

Before disabling anything, ask:

```text
What does this server do?
```

Examples:

```text
Web server

Database server

CI runner

Kubernetes node

Monitoring server

Development server
```

The correct security configuration depends on the server's role.

A web server may legitimately expose:

```text
80/tcp
443/tcp
```

A database server may have completely different requirements.

Hardening without understanding purpose can break the system.

---

# 99. Step 3: Establish Recovery Access

Before changing SSH or firewall rules, know:

```text
How do I recover if remote access fails?
```

Possible answers include:

* Cloud provider console.
* VM console.
* Physical console.
* Out-of-band management.
* Another authorized administrator.
* Snapshot or image recovery.

Do not perform high-risk remote-access changes without understanding your recovery path.

---

# 100. Step 4: Review Users

Inspect:

```bash
cat /etc/passwd
```

Then identify actual human login accounts.

For specific users:

```bash
id username
```

Check administrative groups:

```bash
getent group sudo
```

or:

```bash
getent group wheel
```

Ask:

```text
Does each human account belong here?

Does it still require access?

Does it have more group membership than necessary?
```

---

# 101. Step 5: Review sudo

For your account:

```bash
sudo -l
```

For broader policy, authorized administrators can inspect:

```bash
sudo visudo
```

and configuration under:

```text
/etc/sudoers.d/
```

The goal is not:

```text
Remove all sudo.
```

The goal is:

```text
Provide required administration
without unnecessary privilege.
```

---

# 102. Step 6: Review Sensitive Permissions

Check important files and directories.

Examples:

```bash
ls -ld ~/.ssh
ls -l ~/.ssh
```

For application configuration:

```bash
ls -l /path/to/application/config
```

For system files:

```bash
ls -l /etc/passwd
ls -l /etc/shadow
```

Do not modify system permissions just because they look different from what you expected.

First understand why they exist.

---

# 103. Step 7: Review SSH

Check service:

```bash
systemctl status ssh
```

or:

```bash
systemctl status sshd
```

Check listener:

```bash
sudo ss -lntp
```

Inspect configuration:

```bash
sudo less /etc/ssh/sshd_config
```

Check effective settings where supported:

```bash
sudo sshd -T
```

Review:

* Public-key authentication.
* Password authentication.
* Root login.
* Listening address.
* Listening port.
* Authorized keys.

---

# 104. Step 8: Test Key Authentication

From another terminal:

```bash
ssh -i ~/.ssh/private_key user@server
```

If successful:

```bash
whoami
```

Then verify administrative access if required:

```bash
sudo -l
```

Do not close your original working session yet.

This second-session test is part of the hardening procedure, not an optional extra.

---

# 105. Step 9: Review Listening Ports

Run:

```bash
sudo ss -lntup
```

Create a mental inventory:

```text
Port    Service       Needed?      Exposure
22      SSH           Yes          Admin access
80      HTTP          Maybe        Public
443     HTTPS         Maybe        Public
3306    Database      Depends      Usually restricted
8080    Application   Depends      Review
```

Do not rely on this example as your actual server policy.

Your server's role determines what is required.

---

# 106. Step 10: Review Running Services

Run:

```bash
systemctl --type=service --state=running
```

For anything unfamiliar:

```bash
systemctl status service-name
```

Then determine:

```text
What installed it?

What depends on it?

Does it listen on the network?

Is it required?
```

Only then consider stopping, disabling, or uninstalling it.

---

# 107. Step 11: Review Firewall

If using UFW:

```bash
sudo ufw status verbose
```

Check:

* Is it active?
* What is the default incoming policy?
* Which ports are allowed?
* Are rules broader than necessary?
* Is SSH protected before changes?
* Are IPv6 rules considered where relevant?

Then compare firewall rules with actual listening services.

---

# 108. Step 12: Compare Listening vs Allowed

This is important.

Create two lists:

```text
LISTENING
22
80
443
8080
```

and:

```text
FIREWALL ALLOWED
22
80
443
```

Now ask:

```text
Why is 8080 listening?

Is it intentionally inaccessible externally?

Should it bind only to localhost?

Should it be behind a reverse proxy?

Is another network layer exposing it?
```

Security review is about relationships, not isolated commands.

---

# 109. Step 13: Review Updates

Identify distribution:

```bash
cat /etc/os-release
```

For Ubuntu/Debian:

```bash
sudo apt update
```

Review:

```bash
apt list --upgradable
```

Then follow the appropriate update process for the environment.

After updates, verify important services.

For example:

```bash
systemctl --failed
```

Then check application-specific health.

---

# 110. Step 14: Review Secrets

Search carefully.

Do not run broad secret-search commands that dump sensitive values into shared terminals or logs.

Instead inspect known application configuration locations and ask:

```text
Where does this application receive credentials?

Are secrets hardcoded?

Are they stored in Git?

Who can read them?

Are logs exposing them?

Are backups protecting them?
```

The goal is to understand secret flow.

---

# 111. Step 15: Review Authentication Logs

Depending on your distribution:

```bash
sudo less /var/log/auth.log
```

or:

```bash
sudo less /var/log/secure
```

For systemd:

```bash
sudo journalctl -u ssh
```

or:

```bash
sudo journalctl -u sshd
```

Look for:

* Failed logins.
* Successful logins.
* Unexpected usernames.
* Unexpected sources.
* Repeated attempts.
* Unusual sudo activity.

Do not panic at one unfamiliar line. Investigate context.

---

# 112. Step 16: Review Backups

Ask:

```text
What is backed up?

How often?

Where?

Who can access backups?

Are backups encrypted if required?

Can one compromised server delete all copies?

When was the last successful backup?

When was the last restore test?
```

A security review that ignores recovery is incomplete.

---

# 113. Beginner Linux Server-Hardening Project

You will now harden a fictional Linux web server.

Scenario:

You have received a new Ubuntu server.

It will host a web application.

The intended architecture is:

```text
Internet
   |
   v
Firewall
   |
   +---- 22/tcp --> SSH administration
   |
   +---- 80/tcp --> HTTP
   |
   +---- 443/tcp -> HTTPS
   |
   v
Linux Server
   |
   +---- Web service
   |
   +---- Application
   |
   +---- Logs
   |
   +---- Backup process
```

Your job is to review and harden it without locking yourself out.

Use a disposable VM or lab server.

---

# 114. Project Phase 1: Baseline the System

Run:

```bash
cat /etc/os-release
hostname
uname -a
whoami
id
```

Record the results.

Then:

```bash
date
```

and:

```bash
uptime
```

The purpose is to establish basic context before making changes.

---

# 115. Project Phase 2: Review Accounts

Run:

```bash
cat /etc/passwd
```

Do not delete anything.

Identify:

* Your account.
* Root.
* Obvious service accounts.
* Any unexpected human accounts.

Check yourself:

```bash
id
groups
```

Check administrative group:

```bash
getent group sudo
```

Ask whether every administrative account is expected.

---

# 116. Project Phase 3: Review sudo

Run:

```bash
sudo -l
```

Understand what your user can execute.

Do not modify sudo policy unless necessary for the lab.

The objective is to learn:

```text
Identity
+
Privileges
=
Security context
```

---

# 117. Project Phase 4: Review SSH Files

Run:

```bash
ls -ld ~/.ssh
```

Then:

```bash
ls -la ~/.ssh
```

If you have a private key locally, inspect its permissions:

```bash
ls -l ~/.ssh/id_ed25519
```

where applicable.

Do not print the private key contents.

Check only metadata unless you have a specific reason to inspect sensitive content.

---

# 118. Project Phase 5: Review Authorized Keys

On the server:

```bash
ls -l ~/.ssh/authorized_keys
```

Then, if appropriate:

```bash
cat ~/.ssh/authorized_keys
```

Identify the keys.

In real environments, maintain a process for knowing which person or automation owns each authorized key.

Unknown persistent access should not become normal.

---

# 119. Project Phase 6: Inspect SSH Configuration

Run:

```bash
sudo less /etc/ssh/sshd_config
```

Look for settings related to:

```text
Port
PermitRootLogin
PasswordAuthentication
PubkeyAuthentication
```

Then inspect effective values:

```bash
sudo sshd -T | grep -E 'port|permitrootlogin|passwordauthentication|pubkeyauthentication'
```

Do not change anything yet.

First understand current behavior.

---

# 120. Project Phase 7: Test a Second Connection

Keep your current SSH session open.

Open another terminal.

Connect:

```bash
ssh -i /path/to/private-key user@server
```

Then:

```bash
whoami
```

and:

```bash
sudo -l
```

Only continue with SSH hardening after this alternative connection is proven.

---

# 121. Project Phase 8: Create an SSH Configuration Backup

On the server:

```bash
sudo cp /etc/ssh/sshd_config \
    /etc/ssh/sshd_config.pre-hardening
```

Verify:

```bash
sudo ls -l /etc/ssh/sshd_config*
```

Remember:

This backup does not prevent lockout by itself.

It simply gives you a known previous file if you retain a recovery path.

---

# 122. Project Phase 9: Consider Root SSH Policy

If:

* Your named user works.
* Key authentication works.
* Sudo works.
* Recovery access exists.

then a server policy may disable direct root SSH login.

For example:

```text
PermitRootLogin no
```

Do not apply this simply because this article shows it.

Verify that it matches your environment.

---

# 123. Project Phase 10: Consider Password Authentication

If approved key-based authentication works reliably and your environment does not require SSH password authentication, you may consider:

```text
PasswordAuthentication no
```

Again:

```text
Never disable your current access method
before proving its replacement.
```

---

# 124. Project Phase 11: Validate SSH

Before applying a restart:

```bash
sudo sshd -t
```

If an error appears:

```text
STOP.
```

Fix the configuration first.

If validation succeeds, use the appropriate service operation for your system, such as reload when supported.

Then test from the second terminal.

Keep the original session open until successful verification.

---

# 125. Project Phase 12: Inventory Network Exposure

Run:

```bash
sudo ss -lntup
```

For each listener, record:

```text
Protocol
Address
Port
Process
Purpose
Required?
```

Example:

```text
tcp
0.0.0.0
22
sshd
remote administration
yes
```

Do this for every network-facing service you can identify.

---

# 126. Project Phase 13: Review Running Services

Run:

```bash
systemctl --type=service --state=running
```

Choose an unfamiliar service.

Investigate:

```bash
systemctl status service-name
```

Do not disable it yet.

Find out what it does.

This exercise teaches the correct security habit:

```text
Investigate before changing.
```

---

# 127. Project Phase 14: Prepare Firewall Rules

Check:

```bash
sudo ufw status verbose
```

If the intended server needs:

```text
22/tcp
80/tcp
443/tcp
```

prepare rules appropriate to the environment.

For example:

```bash
sudo ufw allow OpenSSH
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
```

Before enabling UFW, verify:

```bash
sudo ufw status numbered
```

and confirm the SSH rule matches the port actually used.

---

# 128. Project Phase 15: Enable Firewall Carefully

Only after verifying remote-access requirements:

```bash
sudo ufw enable
```

Then:

```bash
sudo ufw status verbose
```

Keep the existing SSH session open.

From another terminal, test:

```bash
ssh user@server
```

If the test fails, investigate while your original session still works.

---

# 129. Project Phase 16: Test Web Ports

If your lab has a web service:

```bash
curl http://server-address
```

For HTTPS where configured:

```bash
curl https://server-address
```

If it fails, troubleshoot:

```text
Application running?
        |
        v
Listening?
        |
        v
Correct address?
        |
        v
Host firewall?
        |
        v
Cloud firewall?
        |
        v
Routing?
        |
        v
Client?
```

Do not automatically blame the firewall.

---

# 130. Project Phase 17: Review Packages

Run:

```bash
sudo apt update
```

Then:

```bash
apt list --upgradable
```

Review available updates.

For a disposable lab, you can practice the normal update workflow according to the distribution.

Afterward:

```bash
systemctl --failed
```

and check important services.

---

# 131. Project Phase 18: Create a Sensitive Test File

Create a lab directory:

```bash
mkdir -p ~/security-lab
```

Create:

```bash
printf '%s\n' \
    'DB_USER=appuser' \
    'DB_PASSWORD=training-example-only' \
    > ~/security-lab/app.env
```

Check:

```bash
ls -l ~/security-lab/app.env
```

Now restrict:

```bash
chmod 600 ~/security-lab/app.env
```

Verify:

```bash
ls -l ~/security-lab/app.env
```

Expected permission pattern:

```text
-rw-------
```

This teaches permission control. Do not use real credentials in the lab.

---

# 132. Project Phase 19: Practice Git Secret Awareness

Inside a disposable Git lab repository, create:

```text
.env
```

containing only fake values.

Add:

```text
.env
```

to:

```text
.gitignore
```

Then:

```bash
git status
```

Observe that an untracked ignored file is normally excluded from ordinary staging.

Now understand the limitation:

If `.env` had already been committed before `.gitignore` was added, `.gitignore` would not magically erase its history.

Do not practice by committing real credentials.

---

# 133. Project Phase 20: Inspect Authentication Activity

Depending on your system:

```bash
sudo grep sshd /var/log/auth.log
```

or:

```bash
sudo journalctl -u ssh
```

On systems using `sshd`:

```bash
sudo journalctl -u sshd
```

Identify:

* Successful authentication.
* Failed authentication.
* Connection times.
* Usernames.
* Sources where recorded.

Learn what your own successful login looks like.

That creates a baseline for recognizing unusual activity later.

---

# 134. Project Phase 21: Create a Failed Login Safely

From your own lab environment, intentionally attempt one SSH login with an incorrect credential.

Then inspect the logs.

The goal is to connect:

```text
Action
   |
   v
Authentication failure
   |
   v
Log event
```

Do not repeatedly generate failed authentication attempts against systems you do not own or administer.

---

# 135. Project Phase 22: Review Backup Security

Inspect your Part 18 backup directory:

```bash
ls -lh ~/veriqta-backup/backups
```

Then:

```bash
ls -l ~/veriqta-backup/backups
```

Ask:

```text
Who can read these?

What sensitive information is inside?

Can the production server delete all copies?

Are restore tests performed?

Where should another copy live?
```

Security includes recovery.

---

# 136. Project Phase 23: Build a Hardening Checklist

Create:

```bash
nano ~/security-lab/server-hardening-checklist.txt
```

Write:

```text
SYSTEM
[ ] Distribution identified
[ ] Kernel identified
[ ] Server purpose documented

ACCESS
[ ] Human users reviewed
[ ] Unnecessary accounts reviewed
[ ] Administrative groups reviewed
[ ] sudo access reviewed

SSH
[ ] SSH keys reviewed
[ ] Private keys protected
[ ] authorized_keys reviewed
[ ] Root SSH policy reviewed
[ ] Password authentication policy reviewed
[ ] SSH configuration validated
[ ] Second connection tested

NETWORK
[ ] Listening ports reviewed
[ ] Running services reviewed
[ ] Unnecessary services investigated
[ ] Firewall status reviewed
[ ] Required ports documented
[ ] Cloud firewall reviewed where applicable

PATCHING
[ ] Package metadata current
[ ] Available updates reviewed
[ ] Security updates addressed
[ ] Services verified after patching

SECRETS
[ ] Sensitive configuration identified
[ ] Permissions reviewed
[ ] No real secrets committed to Git
[ ] Secret storage method reviewed

MONITORING
[ ] Authentication logs located
[ ] Successful logins reviewed
[ ] Failed logins reviewed
[ ] sudo activity review understood

RECOVERY
[ ] Backups exist
[ ] Backup permissions reviewed
[ ] Backup location reviewed
[ ] Retention understood
[ ] Restore tested
```

You have now turned security into a repeatable process.

---

# 137. Common Beginner Security Mistake: chmod 777

Problem:

An application receives:

```text
Permission denied
```

Beginner response:

```bash
chmod -R 777 application/
```

Why this is dangerous:

`777` grants:

```text
Owner: read + write + execute
Group: read + write + execute
Others: read + write + execute
```

Instead investigate:

```bash
whoami
ls -ld application
ls -l application
```

Determine:

* Which user runs the application?
* Which file needs access?
* Which operation requires access?
* Should ownership change?
* Should a group be used?
* Is execute permission actually needed?

Fix the minimum necessary permission.

---

# 138. Common Mistake: Running Everything With sudo

Problem:

```text
Permission denied
```

Response:

```text
sudo everything
```

This hides the underlying access model.

Instead:

```text
Observe error
    |
    v
Identify object
    |
    v
Check ownership
    |
    v
Check permissions
    |
    v
Determine required access
    |
    v
Fix correctly
```

---

# 139. Common Mistake: Disabling SSH Passwords Too Early

Problem:

Administrator reads:

```text
Disable password authentication.
```

They immediately configure:

```text
PasswordAuthentication no
```

Their key does not work.

They restart SSH.

They disconnect.

They cannot reconnect.

Correct approach:

```text
Configure key
     |
     v
Test second connection
     |
     v
Verify sudo
     |
     v
Validate config
     |
     v
Disable unwanted auth
     |
     v
Test again
```

---

# 140. Common Mistake: Enabling Firewall Before SSH

Problem:

Remote server.

Administrator configures default deny and enables firewall without allowing the actual SSH path.

Result:

```text
Current connection may survive temporarily
but new SSH connections fail.
```

Always protect your management path before enabling restrictive firewall rules.

---

# 141. Common Mistake: Opening Every Port

Problem:

Application does not work.

Response:

```text
Allow all traffic.
```

That may make the application reachable, but it destroys the value of the firewall.

Instead determine:

```text
Which service?

Which protocol?

Which port?

Which source?

Which destination?
```

Then allow only what is required.

---

# 142. Common Mistake: Assuming No Firewall Means No Network Filtering

A cloud server might show:

```text
UFW inactive
```

while cloud security groups still restrict traffic.

Or UFW may allow a port while the cloud firewall blocks it.

Always understand the complete network path.

---

# 143. Common Mistake: Ignoring IPv6

You secure:

```text
0.0.0.0
```

but forget that a service may also listen on:

```text
[::]
```

Modern server security must consider both IPv4 and IPv6 when IPv6 is enabled.

Inspect:

```bash
ss -lntup
```

and review firewall behavior for both protocols.

---

# 144. Common Mistake: Treating .gitignore as a Vault

`.gitignore` is not encryption.

It does not protect a secret that:

* Was already committed.
* Was pasted into an issue.
* Appeared in CI logs.
* Was copied elsewhere.
* Exists in repository history.

Once a secret is exposed, rotate it.

---

# 145. Common Mistake: Ignoring Logs Until an Incident

If you have never looked at normal authentication logs, unusual logs are harder to interpret.

Learn your normal system.

Know:

* Normal usernames.
* Normal login sources.
* Normal service behavior.
* Normal sudo activity.
* Normal listening ports.

Security monitoring depends heavily on understanding baseline behavior.

---

# 146. Common Mistake: Hardening Without Documentation

Suppose an engineer sees:

```text
Port 8443
```

and blocks it.

The next day, production deployment fails because 8443 served an internal platform component.

Security changes need context.

Document:

```text
Service
Purpose
Owner
Port
Exposure
Authentication
Dependencies
```

Security and reliability must work together.

---

# 147. Common Mistake: Security by Obscurity

Examples include:

```text
Nobody knows this port.

The directory has a strange name.

The API URL is hidden.

We changed SSH from 22 to 49222.
```

These may reduce casual discovery in some situations.

They do not replace real access controls.

Assume attackers can eventually discover exposed services.

Protect them properly.

---

# 148. Common Mistake: No Recovery Plan

Security changes can break systems.

Before modifying:

* SSH.
* Firewall.
* Authentication.
* Permissions.
* Boot configuration.
* Critical services.

know how to recover.

A secure configuration that permanently locks legitimate administrators out is still an operational failure.

---

# 149. Security Troubleshooting Method

Use the reasoning process from Part 16:

```text
OBSERVE
   |
   v
What happened?

ESTABLISH SCOPE
   |
   v
One user?
One service?
Whole server?

GATHER EVIDENCE
   |
   +--> Logs
   +--> Permissions
   +--> Processes
   +--> Ports
   +--> Configuration

FORM HYPOTHESIS
   |
   v
What could explain this?

TEST
   |
   v
Does evidence support it?

FIX
   |
   v
Change the root cause

VERIFY
   |
   v
Does intended behavior work?

MONITOR
   |
   v
Does problem recur?
```

Do not make ten security changes simultaneously and then wonder which one caused the outage.

---

# 150. Scenario: SSH Says Permission Denied

You run:

```bash
ssh user@server
```

and receive:

```text
Permission denied
```

Possible causes include:

* Wrong username.
* Wrong password.
* Wrong private key.
* Public key missing from `authorized_keys`.
* Wrong `.ssh` ownership.
* Wrong `.ssh` permissions.
* Password authentication disabled.
* Public-key authentication disabled.
* Account restrictions.
* SSH configuration.

Investigate systematically.

Client-side verbose output can help:

```bash
ssh -v user@server
```

For deeper debugging:

```bash
ssh -vv user@server
```

Avoid publishing verbose logs without checking whether they contain sensitive environmental details.

---

# 151. Scenario: SSH Connection Refused

Error:

```text
Connection refused
```

This differs from:

```text
Permission denied
```

`Permission denied` usually means you reached an SSH service but authentication failed.

`Connection refused` commonly means no service accepted the connection on that destination port.

Investigate:

```bash
systemctl status ssh
```

or:

```bash
systemctl status sshd
```

Then:

```bash
ss -lntp
```

Check:

* Is SSH running?
* Is it listening?
* Which port?
* Which address?

---

# 152. Scenario: SSH Times Out

A timeout often points toward a network-path problem rather than an authentication failure.

Investigate:

```text
Client
 |
 v
Network
 |
 v
Cloud firewall
 |
 v
Host firewall
 |
 v
SSH listener
```

Check:

* Correct IP address.
* Routing.
* Cloud firewall.
* Host firewall.
* SSH port.
* Server availability.

Different symptoms suggest different layers.

---

# 153. Scenario: Web Server Is Running but Unreachable

You run:

```bash
systemctl status nginx
```

and see:

```text
active (running)
```

But users cannot connect.

Check:

```bash
ss -lntp
```

Then:

```bash
sudo ufw status
```

Then external network controls.

Possible issue:

```text
nginx running
but
firewall blocks 443
```

or:

```text
nginx running
but
listening only on localhost
```

Service status alone does not prove network reachability.

---

# 154. Scenario: Unexpected Port Appears

You run:

```bash
sudo ss -lntup
```

and discover:

```text
0.0.0.0:9000
```

Do not immediately:

```bash
kill -9 process
```

Instead:

1. Identify the process.
2. Identify its service.
3. Determine who installed it.
4. Determine its purpose.
5. Check configuration.
6. Determine whether exposure is intentional.
7. Restrict, stop, or remove it only after understanding impact.

Security investigation still requires operational discipline.

---

# 155. Scenario: Suspicious Failed Logins

You find hundreds of:

```text
Failed password
```

entries.

Investigate:

```bash
sudo grep "Failed password" /var/log/auth.log
```

where applicable.

Then check successful logins.

Use:

```bash
last
```

Review SSH configuration.

Check authorized keys.

Review administrative activity.

The reasoning is:

```text
Attempts occurred
      |
      v
Did any succeed?
      |
      v
What account?
      |
      v
What happened afterward?
```

Do not equate attempted attack with successful compromise.

---

# 156. Scenario: Secret Accidentally Pushed to Git

You discover an API key in repository history.

Wrong response:

```text
Delete file
Commit
Done
```

Correct reasoning:

```text
Credential exposed
      |
      v
Assume compromise
      |
      v
Revoke credential
      |
      v
Generate replacement
      |
      v
Update legitimate systems
      |
      v
Investigate use
      |
      v
Address repository exposure
      |
      v
Add prevention controls
```

The credential itself is the immediate security problem.

---

# 157. Scenario: Security Update Requires Restart

A critical package has been updated.

Do not assume the running service automatically loaded the new code.

Check:

* Which package changed?
* Which service uses it?
* Does the service need restarting?
* Does the system need rebooting?
* Is there a maintenance window?
* Is redundancy available?
* How will you verify application health afterward?

Security and availability need to be managed together.

---

# 158. Defense in Depth

Defense in depth means using multiple security controls so that failure of one control does not automatically expose everything.

Example:

```text
Internet
   |
   v
Cloud firewall
   |
   v
Host firewall
   |
   v
Restricted listening service
   |
   v
Strong authentication
   |
   v
Least-privileged user
   |
   v
Restricted file permissions
   |
   v
Monitoring
```

No individual layer is perfect.

Together they reduce risk.

---

# 159. Attack Surface

Attack surface means the collection of paths through which a system might be attacked.

Examples:

* Network services.
* User accounts.
* APIs.
* Installed software.
* SSH.
* Web applications.
* Credentials.
* Misconfigured permissions.

Reducing attack surface means removing unnecessary opportunities.

For example:

```text
10 unnecessary services
```

create more things to:

* Configure.
* Patch.
* Monitor.
* Protect.

If a service has no purpose, removing it may be better than trying to harden something you do not need.

---

# 160. Hardening Is Not Breaking Everything

A "secure" server where:

```text
SSH does not work
application cannot access files
monitoring cannot connect
deployments fail
backups fail
```

is not a successful hardening project.

The goal is:

```text
Required functionality
        +
Minimum necessary exposure
        +
Appropriate controls
```

Security supports the system's purpose.

It does not replace it.

---

# 161. Final Server-Hardening Verification

After making changes, verify:

```bash
whoami
id
```

Check SSH:

```bash
systemctl status ssh
```

or:

```bash
systemctl status sshd
```

Check configuration:

```bash
sudo sshd -t
```

Check ports:

```bash
sudo ss -lntup
```

Check firewall:

```bash
sudo ufw status verbose
```

Check services:

```bash
systemctl --failed
```

Check logs:

```bash
sudo journalctl -p warning --since "1 hour ago"
```

Then test actual functionality:

* New SSH login.
* `sudo`.
* Application access.
* Web traffic.
* Backup execution.
* Restore process where appropriate.

Verification must test what users and administrators actually need.

---

# 162. What You Should Now Understand

Before moving to Part 20, you should understand:

* Why Linux security matters.
* Why Linux servers are valuable targets.
* Why security is an ongoing process.
* The principle of least privilege.
* Why excessive privilege increases impact.
* Root-account risks.
* Why root should not be the default troubleshooting solution.
* What `sudo` does.
* Why sudo is useful.
* Why sudo does not make unsafe commands safe.
* How `sudo -l` helps inspect privilege.
* Why sudo policy should be carefully controlled.
* User-account hygiene.
* Why obsolete accounts increase risk.
* Why password locking does not necessarily disable every authentication mechanism.
* Authentication vs authorization.
* Strong password practices.
* Why individual administrator accounts are useful.
* File permissions as security controls.
* Ownership as a security control.
* Why sensitive files require restricted access.
* SSH security fundamentals.
* SSH clients and servers.
* SSH key pairs.
* Public vs private keys.
* Why private keys require protection.
* Why passphrases can add protection.
* What `authorized_keys` does.
* Why authorized keys should be reviewed.
* SSH directory permissions.
* Password authentication considerations.
* Why SSH changes can lock administrators out.
* Safe SSH-hardening workflow.
* `sshd_config`.
* SSH configuration snippets.
* `sshd -t`.
* `sshd -T`.
* Root SSH login considerations.
* `PermitRootLogin`.
* `PasswordAuthentication`.
* Why changing port 22 is not sufficient security.
* Network service exposure.
* Listening ports.
* Listening addresses.
* `127.0.0.1`.
* `0.0.0.0`.
* IPv6 listeners.
* Why listening does not automatically mean Internet-reachable.
* How to identify processes behind ports.
* Why unnecessary services increase attack surface.
* `systemctl stop`.
* `systemctl disable`.
* Why unfamiliar services should be investigated before removal.
* Firewall fundamentals.
* Host firewalls vs cloud firewalls.
* What UFW is.
* `ufw status`.
* `ufw status verbose`.
* `ufw status numbered`.
* Default incoming and outgoing policies.
* Why SSH must be considered before enabling a firewall remotely.
* How to allow required ports.
* Why protocols matter.
* How to remove UFW rules.
* Why firewall rules should not replace proper service binding.
* Firewall troubleshooting.
* Why software updates matter to security.
* `apt update`.
* `apt list --upgradable`.
* `apt upgrade`.
* Why production patching needs planning.
* Why Linux distributions have different update mechanisms.
* Why updates may require service restarts or reboots.
* What secrets are.
* Why configuration files may be sensitive.
* Environment variables and secrets.
* Why environment variables are not secret vaults.
* Why secrets should not be committed to Git.
* Why `.gitignore` is not complete secret management.
* What to do after secret exposure.
* Why exposed credentials should be rotated.
* Authentication logs.
* `/var/log/auth.log`.
* `/var/log/secure`.
* `journalctl`.
* Failed-login investigation.
* Successful vs failed authentication.
* `last`.
* `lastlog`.
* Sudo activity.
* Why logs are security evidence.
* Why normal behavior should be understood before incidents.
* Secure configuration habits.
* Configuration validation.
* Reload vs restart.
* Why backups are a security control.
* Why backups themselves need protection.
* Why separate failure domains matter.
* A repeatable server-hardening methodology.
* Defense in depth.
* Attack surface reduction.
* Why security and reliability must work together.

# 163. Hands-On Practice

Complete these exercises on a disposable Linux lab system.

1. Identify your distribution with `cat /etc/os-release`.
2. Check your kernel.
3. Check your hostname.
4. Check your current user.
5. Inspect your UID and groups.
6. Explain why UID 0 is special.
7. Run `sudo -l`.
8. Explain what privileges your account has.
9. Inspect `/etc/passwd`.
10. Identify human accounts and service accounts without deleting anything.
11. Check the `sudo` or `wheel` group as appropriate.
12. Inspect currently logged-in users.
13. Inspect your `.ssh` directory.
14. Check private-key permissions where applicable.
15. Explain why a private key must not be public.
16. Inspect `authorized_keys`.
17. Explain what each authorized key represents.
18. Check SSH service status.
19. Determine whether your system calls it `ssh` or `sshd`.
20. Find the SSH listening port.
21. Find the SSH listening address.
22. Inspect `sshd_config`.
23. Locate any SSH configuration snippets.
24. Inspect effective SSH settings with `sshd -T` where supported.
25. Check `PermitRootLogin`.
26. Check `PasswordAuthentication`.
27. Check `PubkeyAuthentication`.
28. Create a backup of the SSH configuration.
29. Validate SSH configuration with `sshd -t`.
30. Open a second SSH connection.
31. Confirm key authentication works.
32. Confirm sudo works in the second session.
33. Explain why the first session should remain open.
34. Run `ss -lntup`.
35. Create an inventory of every listening port.
36. Identify the process behind each important listener.
37. Determine whether each service is required.
38. List running systemd services.
39. Investigate one unfamiliar service.
40. Explain why you should not disable it immediately.
41. Check whether UFW exists.
42. Check UFW status.
43. Inspect verbose firewall status.
44. Inspect numbered rules.
45. Identify the SSH firewall requirement.
46. Identify required application ports.
47. Explain the difference between host and cloud firewalls.
48. Explain why `0.0.0.0:8080` does not prove Internet reachability.
49. Explain why `127.0.0.1:8080` differs from `0.0.0.0:8080`.
50. Check IPv6 listeners.
51. Create a firewall plan before changing rules.
52. Ensure remote management access is accounted for.
53. Enable UFW only on a safe lab system.
54. Verify a second SSH connection afterward.
55. Test an allowed service.
56. Test a blocked port.
57. Review firewall rules again.
58. Update package metadata.
59. List available upgrades.
60. Identify your distribution's security-update mechanism.
61. Explain why production updates require planning.
62. Check failed systemd units after a lab update.
63. Create a fake sensitive configuration file.
64. Set it to `600`.
65. Explain the permission.
66. Change it temporarily to an overly permissive mode.
67. Explain why that is dangerous.
68. Restore secure permissions.
69. Create a disposable Git repository.
70. Create a fake `.env`.
71. Add `.env` to `.gitignore`.
72. Confirm Git ignores it.
73. Explain why `.gitignore` does not protect previously committed secrets.
74. Describe the response to a committed real credential.
75. Locate authentication logs.
76. Find your own successful SSH login.
77. Generate one intentional failed login against your own lab.
78. Locate the corresponding event.
79. Compare successful and failed authentication events.
80. Run `last`.
81. Inspect recent login history.
82. Search for sudo activity.
83. Explain why named accounts improve accountability.
84. Review backup permissions.
85. Determine whether backups contain sensitive data.
86. Explain why backups on the same server are insufficient against some failures.
87. Perform a restore test.
88. Build a written hardening checklist.
89. Document every required listening port.
90. Document every administrator account.
91. Document your SSH authentication policy.
92. Document your firewall policy.
93. Document your patching process.
94. Document how secrets are stored.
95. Document where authentication logs live.
96. Document how backups are protected.
97. Reboot your disposable lab if appropriate.
98. Confirm required services return.
99. Test SSH again.
100. Perform the complete hardening review again from the beginning without following the commands blindly.

# 164. Beginner Knowledge Check

1. Why does Linux security matter to DevOps engineers?
2. Why is security a process rather than a command?
3. What is least privilege?
4. What is blast radius?
5. Why is root dangerous?
6. What is UID 0?
7. Why should root not be your default troubleshooting solution?
8. What does `sudo` do?
9. Does sudo make a command safe?
10. What does `sudo -l` show?
11. Why should sudo access be restricted?
12. Why should administrators use individual accounts?
13. Why are obsolete user accounts risky?
14. Does locking a password necessarily disable SSH key access?
15. What is authentication?
16. What is authorization?
17. Why do file permissions matter to security?
18. Why does file ownership matter?
19. What does `600` mean?
20. What types of files may contain secrets?
21. What is SSH?
22. What is an SSH key pair?
23. Which key must remain private?
24. What is the purpose of a key passphrase?
25. What does `authorized_keys` do?
26. Why should authorized keys be reviewed?
27. Why can SSH reject an overly exposed private key?
28. What does `PasswordAuthentication` control?
29. Why might key authentication be preferred for server administration?
30. Why must key login be tested before disabling password login?
31. What does `PermitRootLogin` control?
32. Why might direct root SSH login be disabled?
33. What does `sshd -t` do?
34. What does `sshd -T` help inspect?
35. Why should you keep a known-good SSH session open during hardening?
36. Does changing SSH from port 22 make SSH secure?
37. What does `ss -lntup` help show?
38. What does `127.0.0.1` mean?
39. What does `0.0.0.0` usually indicate for a listener?
40. Does a listener on `0.0.0.0` prove Internet reachability?
41. Why should IPv6 be considered?
42. What is attack surface?
43. Why should unnecessary services be removed or disabled?
44. Why should unfamiliar services be investigated first?
45. What does `systemctl stop` do?
46. What does `systemctl disable` do?
47. What is a firewall?
48. What is UFW?
49. What does `ufw status` show?
50. Why should SSH be allowed before enabling a restrictive firewall remotely?
51. What does `ufw allow 22/tcp` mean?
52. Why should only required ports be opened?
53. What is the difference between a host firewall and a cloud firewall?
54. Why might a service still be unreachable when UFW allows it?
55. Why might a service still be protected when UFW is inactive?
56. Why should unnecessary services bind only where required?
57. Why are software updates part of security?
58. What does `apt update` do?
59. Does `apt update` install every upgrade?
60. What does `apt list --upgradable` show?
61. Why should production patching be planned?
62. Why might a reboot be required after some updates?
63. What is a secret?
64. Give four examples of secrets.
65. Why can configuration files be sensitive?
66. Are environment variables automatically secure secret storage?
67. How can environment variables leak?
68. Why should secrets never be committed to Git?
69. Does deleting a secret in a later Git commit remove every copy?
70. What should you do when a credential is exposed?
71. What does credential rotation mean?
72. What does `.gitignore` do?
73. Why is `.gitignore` not secret management?
74. Why are logs useful for security?
75. Where are authentication logs commonly found on Ubuntu?
76. Where are they commonly found on RHEL-family systems?
77. How can `journalctl` help?
78. Does a failed SSH login prove compromise?
79. What should you investigate after repeated failed logins?
80. What does `last` help show?
81. Why is sudo activity useful for auditing?
82. What is a security baseline?
83. Why should you know normal system behavior?
84. Why should configuration be validated before restarting services?
85. What is the difference between reload and restart?
86. Why are backups part of security?
87. Why must backups themselves be protected?
88. Why should backups exist in separate failure domains?
89. What is defense in depth?
90. Give four layers of defense in depth for an SSH server.
91. Why is `chmod 777` usually a poor permission fix?
92. Why is "run it with sudo" poor troubleshooting?
93. Why can firewall hardening cause lockout?
94. Why is opening every port a bad troubleshooting strategy?
95. Why is security by obscurity insufficient?
96. Why should server purpose be understood before hardening?
97. Why should security changes be documented?
98. Why should changes be tested after hardening?
99. Why must security and reliability be considered together?
100. What should your first action be when asked to secure an unfamiliar Linux server?

# 165. Knowledge Check Answers

1. Linux hosts applications, cloud workloads, CI/CD systems, containers, credentials, data, and infrastructure components that engineers must protect.

2. Systems, software, users, threats, and configurations change continuously.

3. Grant only the access required to perform a task.

4. The amount of damage or scope of impact that a failure or compromise can affect.

5. Root has extremely broad control over the operating system.

6. The traditional numeric user identifier for root.

7. It bypasses permission boundaries without explaining why access failed.

8. It allows authorized execution of commands with another user's privileges, commonly root.

9. No. It only changes execution privilege.

10. The commands and privileges the current user is permitted to use through sudo.

11. Excessive administrative access increases potential damage from mistakes and compromise.

12. Individual accounts improve accountability, revocation, auditing, and privilege control.

13. They provide unnecessary potential access paths.

14. No. Other authentication methods may still work.

15. Verifying identity.

16. Determining what an authenticated identity is allowed to do.

17. They control which users and groups can read, modify, or execute resources.

18. Permissions are interpreted in relation to owners and groups.

19. Owner read/write, group none, others none.

20. SSH keys, application configuration, credential files, TLS private keys, cloud credentials, database configuration, and backups.

21. A secure protocol commonly used for remote administration.

22. A mathematically related public and private key used for cryptographic authentication.

23. The private key.

24. It can add protection if the private key file is stolen.

25. It defines public keys authorized to authenticate to an account.

26. Obsolete or unauthorized keys can provide persistent access.

27. OpenSSH deliberately rejects insecure private-key permissions in many situations.

28. Whether SSH password authentication is permitted according to the effective SSH configuration.

29. It avoids some password-guessing and credential-reuse risks when keys are properly managed.

30. Otherwise you may remove your only working authentication method.

31. Whether and under what conditions root may authenticate directly through SSH.

32. Named users plus sudo can provide stronger accountability and reduce direct exposure of the most privileged account.

33. It validates SSH server configuration syntax.

34. Effective SSH server configuration.

35. It provides a recovery path while you test new settings.

36. No.

37. Listening TCP/UDP sockets and, where privileges allow, associated processes.

38. IPv4 loopback, meaning the local host.

39. Listening on all IPv4 interfaces.

40. No. Firewalls, routing, cloud controls, and other network layers still matter.

41. A service may be reachable over IPv6 even when you focused only on IPv4.

42. The collection of potential paths through which a system can be attacked.

43. Every unnecessary service adds software, configuration, exposure, and maintenance requirements.

44. It may be required by the operating system or another application.

45. Stops a service now.

46. Removes its normal enabled-at-boot behavior.

47. A system that controls network traffic according to rules.

48. Uncomplicated Firewall, a simplified firewall-management interface common on Ubuntu.

49. UFW's activation state and configured rules, depending on options used.

50. Otherwise you can prevent new remote SSH connections.

51. Allow incoming TCP traffic to port 22 under the relevant UFW policy.

52. Every unnecessary opening increases exposure.

53. A host firewall operates on the server. A cloud firewall or security group operates at the cloud/network layer.

54. The application may not be listening, may use another port/address, or another network control may block traffic.

55. Cloud or upstream network controls may still restrict access.

56. Binding only where required reduces exposure before firewall controls are even considered.

57. Updates frequently contain vulnerability fixes.

58. Refreshes package repository metadata on Debian/Ubuntu systems.

59. No.

60. Packages for which newer versions are available according to current package metadata.

61. Updates can restart services, change behavior, introduce compatibility issues, or require rebooting.

62. Running components such as kernels or long-lived processes may need restart/reboot to begin using updated code.

63. Sensitive information used for authentication, authorization, encryption, signing, or protected access.

64. Password, API key, private key, access token.

65. They may contain credentials and sensitive application settings.

66. No.

67. Logs, debugging, process environments, CI/CD systems, shell history, and diagnostic tooling can expose them depending on usage and permissions.

68. Git preserves history and distributes repository data to multiple systems and users.

69. No.

70. Revoke it, replace it, update legitimate consumers, investigate possible use, and address the exposure.

71. Replacing an old credential with a new one and invalidating the old credential.

72. Tells Git which untracked paths should normally be ignored.

73. It does not encrypt secrets, remove repository history, or rotate compromised credentials.

74. They provide evidence about authentication, services, privileges, and system activity.

75. Commonly `/var/log/auth.log`.

76. Commonly `/var/log/secure`.

77. It provides access to systemd journal events, including service and authentication-related records where logged.

78. No.

79. Whether attempts succeeded, targeted users, source information, authentication method, and subsequent activity.

80. Historical login sessions recorded by the system's login accounting data.

81. It can help associate privileged operations with named users.

82. An understanding or record of expected system configuration and behavior.

83. Deviations are easier to recognize when normal behavior is understood.

84. Validation can catch errors before they cause service outages.

85. Reload asks a running service to reread configuration where supported. Restart stops and starts the service.

86. Security includes recovery from malicious destruction, corruption, and accidental loss.

87. Backups may contain the same sensitive information as production.

88. One incident should not be able to destroy every copy.

89. Using multiple security controls so failure of one does not automatically compromise the entire system.

90. Network firewall, restricted SSH listener, strong authentication, least-privileged accounts. Logging and monitoring can add further layers.

91. It grants excessive permissions instead of solving the actual ownership or access requirement.

92. It bypasses the symptom without understanding the permission model.

93. Incorrect SSH or firewall changes can block the administrator's own management connection.

94. It exposes services that do not need external access.

95. Hidden names and unusual ports can eventually be discovered and do not provide proper authorization or authentication.

96. Required services, users, ports, and access depend on what the server is supposed to do.

97. Future administrators need to know which exposure and controls are intentional and why.

98. A configuration can be syntactically valid yet still break required functionality.

99. Infrastructure must resist unauthorized access while continuing to provide its required service.

100. Understand and baseline the server before changing it.

# 166. Final Beginner Server-Hardening Mental Model

Do not think:

```text
SECURE LINUX
     =
Disable root
+
Disable passwords
+
Enable firewall
+
Done
```

Think:

```text
UNDERSTAND
    |
    v
What is this server for?
    |
    v
IDENTIFY
    |
    +--> Users
    +--> Privileges
    +--> Services
    +--> Ports
    +--> Data
    +--> Secrets
    |
    v
REDUCE
    |
    +--> Unnecessary users
    +--> Excessive privileges
    +--> Unnecessary services
    +--> Unnecessary network exposure
    |
    v
PROTECT
    |
    +--> Authentication
    +--> SSH keys
    +--> Permissions
    +--> Secrets
    +--> Firewall
    +--> Backups
    |
    v
MAINTAIN
    |
    +--> Updates
    +--> Key rotation
    +--> Access reviews
    +--> Configuration reviews
    |
    v
MONITOR
    |
    +--> Authentication
    +--> Logs
    +--> Privileged activity
    +--> Service behavior
    |
    v
RECOVER
    |
    +--> Backups
    +--> Restore tests
    +--> Recovery access
    |
    v
VERIFY
```

A secure Linux administrator does not ask only:

```text
"What hardening commands should I run?"
```

They ask:

```text
What is running?

Why is it running?

Who can access it?

How do they authenticate?

What can they do after authentication?

Which network paths reach it?

What data and credentials can it access?

Is the software maintained?

Will suspicious activity leave evidence?

What happens if this control fails?

Can I recover if something goes wrong?
```

Once you start asking those questions, Linux security stops being a collection of commands and becomes part of how you administer the system.

Part 20 can now bring the entire Linux beginner-to-intermediate series together in one final practical environment, where the learner has to build, administer, automate, secure, break, troubleshoot, and recover a Linux server using the skills from Parts 1 through 19.
