# Part 13: DNS, SSH, and Remote Server Administration

In Part 12, you learned how Linux networking works at a foundational level. You learned about IP addresses, interfaces, routes, TCP, UDP, ports, sockets, listening addresses, `ping`, `curl`, `ss`, and how to distinguish problems such as connection refusal and connection timeout.

Now we will build on that foundation.

A Linux administrator rarely works only on the computer physically in front of them. In real environments, servers may be running in:

* AWS.
* Azure.
* Google Cloud.
* Data centers.
* Private networks.
* Virtual machines.
* Development environments.
* Kubernetes worker nodes.
* CI/CD infrastructure.

Two technologies make this much easier:

```text
DNS
+
SSH
```

DNS helps you find systems by name.

SSH allows you to securely connect to and administer remote systems.

By the end of this part, you should understand not only how to run:

```bash
ssh user@server
```

but what happens before that command succeeds, how SSH keys work, how to transfer files securely, and how to systematically troubleshoot DNS and SSH failures.

---

# 1. The Problem DNS Solves

Imagine that your company has these servers:

```text
Web Server
10.20.1.15

Application Server
10.20.2.34

Database Server
10.20.3.48

Monitoring Server
10.20.4.21
```

You could remember all those IP addresses.

But humans are generally better at remembering names such as:

```text
web.example.com
api.example.com
database.internal.example.com
monitoring.example.com
```

DNS provides the system that translates names into information computers can use.

---

# 2. What Is DNS?

DNS stands for:

```text
Domain Name System
```

At a beginner level, DNS allows names such as:

```text
example.com
```

to be resolved to IP addresses such as:

```text
203.0.113.20
```

The simplified relationship is:

```text
Human-friendly name
example.com
       |
       v
      DNS
       |
       v
IP address
203.0.113.20
```

Computers ultimately need addressing and routing information to send IP traffic.

DNS allows applications and users to work with names instead of manually tracking addresses.

---

# 3. Why DNS Exists

Imagine changing the IP address of an application server.

Without DNS, users might connect directly to:

```text
203.0.113.20
```

If the server moves to:

```text
203.0.113.50
```

every client using the old address would need to know about the change.

With DNS:

```text
app.example.com
       |
       v
DNS
       |
       v
203.0.113.50
```

The name can remain:

```text
app.example.com
```

while the DNS information changes.

This creates an important layer of abstraction between:

```text
service identity
```

and:

```text
current network address
```

---

# 4. DNS in Real Infrastructure

Consider:

```text
User
 |
 v
app.example.com
 |
 v
DNS
 |
 v
203.0.113.50
 |
 v
Load Balancer
 |
 v
Linux Application Servers
```

The user does not need to know the load balancer's IP address.

The user knows:

```text
app.example.com
```

DNS handles the name-resolution step.

---

# 5. What Is a Domain Name?

A domain name is a hierarchical name used within DNS.

Example:

```text
www.example.com
```

Break it down:

```text
www . example . com
 |       |       |
 |       |       +-- Top-level domain
 |       |
 |       +---------- Domain
 |
 +------------------ Host/subdomain label
```

Another example:

```text
api.production.example.com
```

The hierarchy becomes:

```text
com
 |
 v
example
 |
 v
production
 |
 v
api
```

DNS is hierarchical rather than one giant flat list of names.

---

# 6. Fully Qualified Domain Names

You may hear the term:

```text
FQDN
```

It means:

```text
Fully Qualified Domain Name
```

An FQDN represents the complete DNS name of a host within the DNS hierarchy.

For example:

```text
server1.example.com
```

You may occasionally see the absolute DNS representation written with a trailing dot:

```text
server1.example.com.
```

The final dot represents the DNS root.

You normally do not need to type the trailing dot in everyday commands.

---

# 7. DNS Resolution

DNS resolution is the process of finding DNS information for a name.

Suppose you run:

```bash
curl https://example.com
```

Before `curl` can establish an IP connection, the hostname needs to be resolved.

Simplified:

```text
curl
 |
 v
example.com
 |
 v
Name resolution
 |
 v
IP address
 |
 v
TCP connection
 |
 v
HTTPS communication
```

If name resolution fails, the TCP connection to the intended host cannot even begin using that hostname.

---

# 8. Name Resolution Is Different From Connectivity

This distinction is extremely important.

Suppose:

```bash
curl http://10.20.1.15
```

works.

But:

```bash
curl http://web.example.com
```

fails with a name-resolution error.

The web server may be healthy.

The network route may be healthy.

The problem may simply be:

```text
web.example.com
       |
       X
Cannot resolve
```

Therefore:

```text
DNS problem
```

does not automatically mean:

```text
network connectivity problem
```

although DNS itself also depends on network connectivity to DNS servers in many cases.

---

# 9. Simplified DNS Resolution Flow

A simplified DNS lookup might look like:

```text
Application
    |
    v
Operating system resolver
    |
    +----> Local configuration/cache
    |
    v
Configured DNS resolver
    |
    v
DNS infrastructure
    |
    v
Answer
    |
    v
IP address
    |
    v
Application
```

The real DNS system can involve multiple caches and servers.

We will build the beginner model first.

---

# 10. DNS Hierarchy

DNS has a hierarchical structure.

Conceptually:

```text
Root
 .
 |
 +-- com
 |    |
 |    +-- example.com
 |           |
 |           +-- www.example.com
 |           |
 |           +-- api.example.com
 |
 +-- org
 |
 +-- net
```

Different DNS servers can be authoritative for different portions of this hierarchy.

---

# 11. Root DNS Servers

At the top of the DNS hierarchy is the root.

Represented as:

```text
.
```

Root DNS infrastructure does not contain every application's final IP address.

Instead, it helps direct DNS queries toward the appropriate top-level domain infrastructure.

Conceptually:

```text
Where is www.example.com?
          |
          v
        Root
          |
          v
       Ask .com
```

---

# 12. Top-Level Domains

Examples of top-level domains include:

```text
.com
.org
.net
.io
.dev
```

There are also country-code top-level domains such as:

```text
.uk
.mx
.ng
```

For:

```text
example.com
```

`.com` is the top-level domain.

---

# 13. Authoritative DNS Servers

An authoritative DNS server holds authoritative DNS information for a DNS zone.

For example, DNS infrastructure authoritative for:

```text
example.com
```

may contain records describing:

```text
www.example.com
api.example.com
mail.example.com
```

Conceptually:

```text
Resolver
   |
   v
Authoritative server for example.com
   |
   v
"www.example.com points to ..."
```

---

# 14. Recursive DNS Resolver

Your computer normally does not manually contact every level of DNS infrastructure for every lookup.

It sends queries to a DNS resolver.

Examples may include resolvers provided by:

* Your company.
* Your ISP.
* Your cloud environment.
* Your router.
* A public DNS provider.
* A local resolver service.

The resolver performs or coordinates the necessary lookup and returns an answer.

---

# 15. DNS Caching

DNS information can be cached.

Instead of repeating the complete lookup process every time:

```text
First request
   |
   v
DNS lookup
   |
   v
Answer cached
```

A later request may use cached information:

```text
Second request
   |
   v
Cached answer
```

Caching improves performance and reduces DNS traffic.

However, it also explains why DNS changes may not appear everywhere immediately.

---

# 16. TTL

DNS records commonly have a TTL.

TTL means:

```text
Time To Live
```

It tells caching systems how long a DNS record may generally be cached before it should be refreshed.

For example:

```text
TTL: 300
```

means:

```text
300 seconds
```

or:

```text
5 minutes
```

This is one reason DNS changes can appear to take time to propagate through caches.

---

# 17. DNS Records

DNS stores different types of information using records.

Important beginner-level records include:

```text
A
AAAA
CNAME
MX
NS
TXT
PTR
```

You do not need to memorize every DNS record type.

Understand what the common ones represent.

---

# 18. A Record

An `A` record maps a DNS name to an IPv4 address.

Example:

```text
app.example.com
        |
        | A
        v
203.0.113.20
```

Conceptually:

```text
app.example.com.  A  203.0.113.20
```

If you are troubleshooting an IPv4 application endpoint, the `A` record is often one of the first things you inspect.

---

# 19. AAAA Record

An `AAAA` record maps a DNS name to an IPv6 address.

Example:

```text
app.example.com
        |
        | AAAA
        v
2001:db8::20
```

Remember:

```text
A
=
IPv4

AAAA
=
IPv6
```

---

# 20. CNAME Record

A `CNAME` record creates an alias from one DNS name to another canonical name.

Example:

```text
www.example.com
       |
       | CNAME
       v
web.example.net
```

This does not directly mean:

```text
www.example.com = IP address
```

Instead:

```text
www.example.com
       |
       v
web.example.net
       |
       v
A/AAAA lookup
       |
       v
IP address
```

---

# 21. MX Record

`MX` means:

```text
Mail Exchange
```

MX records identify mail servers responsible for receiving email for a domain.

Example conceptually:

```text
example.com
    |
    | MX
    v
mail.example.com
```

You may encounter MX records when troubleshooting mail infrastructure.

---

# 22. NS Record

`NS` means:

```text
Name Server
```

NS records identify authoritative name servers for a DNS zone or delegation.

Example:

```text
example.com
   |
   +-- ns1.example-dns.com
   |
   +-- ns2.example-dns.com
```

---

# 23. TXT Record

TXT records store text data associated with a DNS name.

They are commonly used for things such as:

* Domain verification.
* Email security policies.
* Ownership verification.
* Service configuration.

You may encounter TXT records when configuring cloud platforms, email systems, certificates, or SaaS services.

---

# 24. PTR Record

A `PTR` record is commonly used for reverse DNS.

Normal DNS lookup:

```text
Name
 |
 v
IP address
```

Reverse DNS lookup:

```text
IP address
 |
 v
Name
```

PTR records support the reverse direction.

---

# 25. `/etc/resolv.conf`

Linux commonly uses:

```text
/etc/resolv.conf
```

to expose resolver configuration.

View it:

```bash
cat /etc/resolv.conf
```

You may see:

```text
nameserver 10.0.0.2
search example.internal
```

or:

```text
nameserver 127.0.0.53
```

The exact contents depend on your Linux distribution and network configuration.

---

# 26. Understanding `nameserver`

A line such as:

```text
nameserver 10.0.0.2
```

indicates a DNS resolver address the system can use.

Conceptually:

```text
Linux
 |
 | DNS query
 v
10.0.0.2
 |
 v
DNS answer
```

Multiple `nameserver` entries may exist.

---

# 27. Why You May See `127.0.0.53`

On some Linux systems using `systemd-resolved`, `/etc/resolv.conf` may contain:

```text
nameserver 127.0.0.53
```

This does not necessarily mean the machine is disconnected from external DNS.

A local resolver stub may receive the application's request and forward it using upstream DNS configuration.

Conceptually:

```text
Application
    |
    v
127.0.0.53
Local resolver
    |
    v
Upstream DNS server
    |
    v
DNS infrastructure
```

---

# 28. Do Not Blindly Edit `/etc/resolv.conf`

On many systems, `/etc/resolv.conf` is generated or managed by another service.

For example:

* NetworkManager.
* systemd-resolved.
* DHCP.
* Cloud networking.
* Container runtime.

If you manually change it, your changes may disappear.

Before editing, investigate who manages the file.

Run:

```bash
ls -l /etc/resolv.conf
```

It may be a symbolic link.

Remember what you learned about symbolic links in Part 3.

---

# 29. `/etc/hosts` Revisited

You already encountered:

```text
/etc/hosts
```

in Part 12.

Example:

```text
127.0.0.1 localhost
10.20.1.15 webserver
```

This provides local hostname mappings.

For example:

```bash
getent hosts webserver
```

may return:

```text
10.20.1.15 webserver
```

---

# 30. `/etc/hosts` vs DNS

These are not the same.

```text
/etc/hosts
=
Local mappings on one machine

DNS
=
Distributed naming system
```

If you add:

```text
10.20.1.15 webserver
```

to `/etc/hosts` on Server A, Server B does not automatically receive that mapping.

---

# 31. Why `/etc/hosts` Can Confuse Troubleshooting

Suppose DNS correctly says:

```text
app.example.com
        |
        v
10.20.1.50
```

But your server contains:

```text
10.20.1.20 app.example.com
```

in `/etc/hosts`.

Depending on the system's name service configuration, the local hosts entry may take precedence.

Your server could connect to the wrong address while other machines work correctly.

This creates the classic problem:

```text
Works everywhere except one server.
```

---

# 32. `nslookup`

`nslookup` is a DNS query tool.

Example:

```bash
nslookup example.com
```

You may see information about:

* DNS server used.
* Name queried.
* Address returned.

The exact output varies.

---

# 33. Query an A Record With `nslookup`

Run:

```bash
nslookup example.com
```

Look for IPv4 addresses in the response.

You can also specify a DNS server:

```bash
nslookup example.com 8.8.8.8
```

This asks the specified resolver instead of relying only on the system's default resolver path.

This can help compare DNS answers.

---

# 34. `dig`

`dig` means:

```text
Domain Information Groper
```

It is one of the most useful command-line DNS troubleshooting tools.

Run:

```bash
dig example.com
```

The output may contain sections such as:

```text
QUESTION SECTION
ANSWER SECTION
AUTHORITY SECTION
ADDITIONAL SECTION
```

It may also show:

* Query time.
* DNS server used.
* Status.
* Record TTL.

---

# 35. Basic `dig` Query

Run:

```bash
dig example.com
```

By default, `dig` commonly queries for an `A` record.

You can explicitly request:

```bash
dig example.com A
```

For IPv6:

```bash
dig example.com AAAA
```

---

# 36. `dig +short`

For compact output:

```bash
dig +short example.com
```

You may receive:

```text
203.0.113.20
```

or multiple addresses.

This is useful in scripts and quick troubleshooting.

---

# 37. Query Specific DNS Records

Examples:

```bash
dig example.com A
```

```bash
dig example.com AAAA
```

```bash
dig example.com MX
```

```bash
dig example.com NS
```

```bash
dig example.com TXT
```

This lets you inspect exactly the record type you care about.

---

# 38. Query a Specific DNS Server

Use:

```bash
dig @8.8.8.8 example.com
```

The:

```text
@8.8.8.8
```

means:

```text
Send this DNS query to this DNS server.
```

This is useful when comparing:

```text
Corporate DNS
vs
Public DNS
```

or:

```text
Old resolver
vs
New resolver
```

Do not assume a public resolver can resolve private internal DNS names.

---

# 39. Understanding `dig` Status

You may see:

```text
status: NOERROR
```

This generally means the DNS query completed successfully at the DNS protocol level.

You might also encounter:

```text
NXDOMAIN
```

which means:

```text
Non-Existent Domain
```

The queried name does not exist according to the responding DNS system.

Other statuses can indicate different DNS problems.

---

# 40. NOERROR With No Answer

This is important.

A DNS query can return:

```text
NOERROR
```

but have no answer for the record type you requested.

For example:

```bash
dig example.com AAAA
```

might complete successfully even if no IPv6 address exists.

Therefore:

```text
NOERROR
```

does not always mean:

```text
The record I expected exists.
```

Inspect the answer section.

---

# 41. `host`

Another simple DNS utility is:

```bash
host example.com
```

Possible output may show:

```text
example.com has address ...
```

You can query record types:

```bash
host -t MX example.com
```

or:

```bash
host -t NS example.com
```

`host` often provides more compact human-readable output than `dig`.

---

# 42. `nslookup` vs `dig` vs `host`

At a beginner level:

| Command    | Common use                 |
| ---------- | -------------------------- |
| `nslookup` | Simple DNS lookup          |
| `dig`      | Detailed DNS investigation |
| `host`     | Quick readable DNS query   |

For serious Linux DNS troubleshooting, becoming comfortable with `dig` is especially useful.

---

# 43. DNS Troubleshooting Mental Model

When a hostname fails:

```text
Application
    |
    v
Hostname
    |
    v
Local name-resolution configuration
    |
    v
DNS resolver
    |
    v
DNS response
    |
    v
IP address
    |
    v
Network connection
```

Do not immediately blame the application.

Find where resolution fails.

---

# 44. DNS Troubleshooting Step 1: Identify the Exact Name

Suppose:

```bash
ssh admin@app.example.com
```

fails.

First write down:

```text
Hostname:
app.example.com
```

Check for:

* Typing mistakes.
* Wrong environment.
* Wrong subdomain.
* Missing domain suffix.

For example:

```text
app-prod.example.com
```

and:

```text
app.production.example.com
```

may be completely different DNS names.

---

# 45. DNS Troubleshooting Step 2: Query the Name

Run:

```bash
dig app.example.com
```

or:

```bash
dig +short app.example.com
```

If you receive an IP address, record it.

Example:

```text
10.20.1.15
```

Now you know DNS returned an address.

You still have not proven the server is reachable.

---

# 46. DNS Troubleshooting Step 3: Compare Name and IP

Suppose:

```bash
ssh admin@app.example.com
```

fails.

But:

```bash
ssh admin@10.20.1.15
```

works.

That is strong evidence that SSH itself may be functioning while name resolution or the resolved destination is the problem.

Investigate DNS and local hostname mappings.

---

# 47. DNS Troubleshooting Step 4: Inspect `/etc/hosts`

Run:

```bash
grep -n 'app.example.com' /etc/hosts
```

If you see:

```text
10.20.1.99 app.example.com
```

but DNS returns:

```text
10.20.1.15
```

you have discovered conflicting information.

Do not immediately delete the entry.

Find out why it exists.

---

# 48. DNS Troubleshooting Step 5: Inspect Resolver Configuration

Run:

```bash
cat /etc/resolv.conf
```

Look for:

```text
nameserver
```

Then:

```bash
ls -l /etc/resolv.conf
```

Determine whether the file is managed elsewhere.

If your environment uses `systemd-resolved`, you may also encounter:

```bash
resolvectl status
```

This can display resolver configuration and DNS servers.

---

# 49. DNS Troubleshooting Step 6: Query the Configured Resolver

Suppose `/etc/resolv.conf` shows:

```text
nameserver 10.20.0.2
```

Try:

```bash
dig @10.20.0.2 app.example.com
```

If this fails while another resolver works, investigate the configured resolver or the network path to it.

---

# 50. DNS Troubleshooting Step 7: Check Connectivity to DNS

DNS itself requires communication.

If your configured resolver is:

```text
10.20.0.2
```

ask:

```text
Can this server reach the resolver?
```

Inspect:

```bash
ip route get 10.20.0.2
```

DNS commonly uses:

```text
UDP 53
```

and can also use:

```text
TCP 53
```

Do not assume a DNS failure is always caused by incorrect DNS records.

The client may simply be unable to reach its resolver.

---

# 51. DNS Problem Example

Problem:

```bash
curl https://api.example.com
```

returns a name-resolution error.

Investigation:

```bash
dig api.example.com
```

returns:

```text
10.20.2.25
```

But the application still fails.

Now inspect:

```bash
getent hosts api.example.com
```

Suppose it returns:

```text
10.20.2.10
```

Check:

```bash
grep api.example.com /etc/hosts
```

You discover:

```text
10.20.2.10 api.example.com
```

Root cause:

```text
Stale local hosts entry.
```

The problem was not the authoritative DNS record.

---

# 52. DNS Troubleshooting Checklist

When DNS appears broken, investigate:

```text
Correct hostname?
        |
        v
Local /etc/hosts entry?
        |
        v
Resolver configuration?
        |
        v
Can resolver be reached?
        |
        v
Does dig return an answer?
        |
        v
Correct record type?
        |
        v
Correct IP address?
        |
        v
Cached/stale result?
        |
        v
Can the resulting IP actually be reached?
```

Now we can use this DNS knowledge to understand SSH properly.

---

# 53. What Is SSH?

SSH stands for:

```text
Secure Shell
```

SSH is a protocol used for secure remote communication.

Its most common Linux administration use is:

```text
Securely connect to a remote Linux system and obtain a shell.
```

Example:

```bash
ssh admin@10.20.1.15
```

or:

```bash
ssh admin@server.example.com
```

---

# 54. Why Administrators Use SSH

Suppose your Linux server runs in a cloud data center hundreds or thousands of kilometers away.

You cannot simply sit in front of it.

Instead:

```text
Your computer
     |
     | SSH
     v
Internet/private network
     |
     v
Remote Linux server
     |
     v
Shell
```

Once connected, you can administer the remote system.

---

# 55. What Can You Do Through SSH?

After connecting, you can perform tasks such as:

* Navigate the filesystem.
* Inspect logs.
* Manage services.
* Edit configuration.
* Install packages.
* Inspect processes.
* Troubleshoot networking.
* Deploy applications.
* Run scripts.
* Transfer files.
* Manage permissions.
* Investigate incidents.

Almost everything you have learned in Parts 1 through 12 becomes useful on remote systems through SSH.

---

# 56. SSH Uses Encryption

SSH protects communication between the client and server using cryptographic mechanisms.

Without secure remote access, sensitive information such as:

* Commands.
* Credentials.
* Administrative output.
* File contents.

could be exposed to interception.

SSH was designed as a secure replacement for older insecure remote-login protocols such as Telnet.

---

# 57. SSH Client

The SSH client is the software that initiates the SSH connection.

On Linux or macOS, you can commonly use:

```bash
ssh
```

Check whether it exists:

```bash
which ssh
```

or:

```bash
command -v ssh
```

Check the version:

```bash
ssh -V
```

---

# 58. SSH Server

The remote machine needs SSH server software to accept SSH connections.

A common implementation is:

```text
OpenSSH Server
```

The SSH daemon is commonly:

```text
sshd
```

Think:

```text
SSH client
ssh
   |
   | network connection
   v
SSH server
sshd
```

---

# 59. SSH Client vs SSH Server

This distinction matters.

Your laptop may have:

```text
ssh client
```

but no:

```text
sshd server
```

That means:

```text
You can connect out
```

but the machine is not necessarily configured to:

```text
Accept incoming SSH connections.
```

---

# 60. SSH Port 22

SSH commonly listens on:

```text
TCP port 22
```

Conceptually:

```text
Client
10.20.1.10:temporary-port
        |
        | TCP
        v
Server
10.20.1.15:22
        |
        v
sshd
```

Port 22 is the default.

It is not mandatory.

An SSH server can be configured to use another port.

---

# 61. Connecting to SSH

Basic syntax:

```bash
ssh username@server
```

Example:

```bash
ssh ubuntu@10.20.1.15
```

Breakdown:

```text
ssh
=
SSH client command

ubuntu
=
Remote username

@
=
Separates user from destination

10.20.1.15
=
Remote server address
```

---

# 62. Connecting Using a Hostname

You can use:

```bash
ssh ubuntu@server.example.com
```

Before SSH can connect:

```text
server.example.com
       |
       v
Name resolution
       |
       v
IP address
       |
       v
TCP connection to SSH server
```

This is where DNS and SSH meet.

---

# 63. Connecting on a Different Port

Suppose SSH listens on:

```text
2222
```

Use:

```bash
ssh -p 2222 ubuntu@server.example.com
```

Important:

```text
-p
```

is lowercase for the SSH client port option.

Do not assume every SSH server uses port 22.

---

# 64. SSH Connection Flow

A simplified SSH connection looks like:

```text
ssh user@server
       |
       v
Resolve server name
       |
       v
Obtain destination IP
       |
       v
Find route
       |
       v
Establish TCP connection
       |
       v
SSH protocol negotiation
       |
       v
Verify server identity
       |
       v
Authenticate user
       |
       v
Create secure session
       |
       v
Remote shell
```

This sequence is extremely useful for troubleshooting.

---

# 65. Stage 1: Name Resolution

If you run:

```bash
ssh ubuntu@app.example.com
```

Linux needs to determine what address:

```text
app.example.com
```

represents.

If DNS fails, SSH cannot proceed normally using that name.

You might see an error such as:

```text
Could not resolve hostname
```

That points you toward name resolution, not immediately toward SSH authentication.

---

# 66. Stage 2: TCP Connection

After obtaining the server address, the SSH client attempts a TCP connection.

Usually:

```text
Destination port:
22
```

If the server is unreachable, blocked, or not listening, you may see:

```text
Connection refused
```

or:

```text
Connection timed out
```

Authentication has not even happened yet.

---

# 67. Stage 3: Server Identity

The client needs to determine whether the SSH server is the server it expects.

SSH uses:

```text
host keys
```

for this purpose.

This is different from your personal SSH authentication key.

Keep these concepts separate:

```text
Server host key
=
Identifies the SSH server

User key
=
Can authenticate the user
```

---

# 68. Stage 4: User Authentication

After establishing the secure protocol connection and verifying server identity, SSH authenticates the user.

Common authentication methods include:

```text
Password authentication
```

and:

```text
Public-key authentication
```

Servers may enable or disable particular methods.

---

# 69. Stage 5: Remote Session

After successful authentication:

```text
Local computer
     |
     | encrypted SSH connection
     v
Remote Linux server
     |
     v
Remote shell
```

You can now run commands on the remote machine.

For example:

```bash
whoami
```

may show:

```text
ubuntu
```

and:

```bash
hostname
```

may show:

```text
production-web-01
```

Those commands are now executing remotely.

---

# 70. Password Authentication

With password authentication, the SSH server verifies a password for the remote account.

Example:

```bash
ssh admin@server.example.com
```

You may be prompted:

```text
admin@server.example.com's password:
```

The password is used to authenticate the remote user.

---

# 71. Password Authentication Considerations

Passwords can be convenient, but security depends heavily on:

* Password strength.
* Account controls.
* Rate limiting.
* Network exposure.
* SSH configuration.
* Multi-factor mechanisms where applicable.

Many cloud environments prefer key-based SSH authentication instead of allowing password login.

---

# 72. Key-Based Authentication

SSH can authenticate users using cryptographic key pairs.

A key pair contains:

```text
Private key
+
Public key
```

Conceptually:

```text
Your computer

Private key
   |
   | proves possession
   v

Remote server

Public key
```

The private key should remain under your control.

The public key can be placed on systems you are authorized to access.

---

# 73. Public Key

The public key is the part you can distribute to systems where you need authorized SSH access.

A public key may look conceptually like:

```text
ssh-ed25519 AAAAC3NzaC1... user@laptop
```

The actual key is much longer.

It can be stored on the remote server.

---

# 74. Private Key

The private key must remain private.

Never casually:

* Email it.
* Post it online.
* Put it in a public Git repository.
* Paste it into documentation.
* Share it in chat channels.
* Store it in application source code.

Think:

```text
Public key
=
May be distributed appropriately

Private key
=
Protect carefully
```

---

# 75. How SSH Key Authentication Works

A simplified model:

```text
Client
Private key
    |
    | cryptographic proof
    v
SSH server
    |
    | checks corresponding public key
    v
Authentication succeeds
```

The private key itself does not need to be copied to the remote server.

The server stores the public key.

---

# 76. Why This Is Better Than Copying the Private Key

If the server required your private key to authenticate you, every remote server would need a copy of your secret.

That would be dangerous.

Instead:

```text
Laptop
Private key
KEEP SECRET
    |
    v
Cryptographic authentication
    |
    v
Server
Public key
```

The secret remains on the client.

---

# 77. `ssh-keygen`

`ssh-keygen` creates and manages SSH key material.

To create a modern Ed25519 key pair:

```bash
ssh-keygen -t ed25519
```

Breakdown:

```text
ssh-keygen
=
SSH key generation utility

-t
=
Key type

ed25519
=
Key algorithm
```

Follow the prompts.

---

# 78. Choosing the Key Location

You may see:

```text
Enter file in which to save the key
```

A common default location is:

```text
~/.ssh/id_ed25519
```

The corresponding public key is commonly:

```text
~/.ssh/id_ed25519.pub
```

Therefore:

```text
id_ed25519
=
private key

id_ed25519.pub
=
public key
```

Do not confuse them.

---

# 79. SSH Key Passphrase

`ssh-keygen` may ask for a passphrase.

A passphrase protects the private key file.

This creates another layer of protection if someone obtains the file.

Conceptually:

```text
Private key file stolen
        |
        v
Passphrase still required
```

A passphrase does not make careless private-key handling acceptable, but it reduces risk.

---

# 80. The `.ssh` Directory

SSH configuration and user key material commonly live in:

```text
~/.ssh
```

Remember:

```text
~
```

means your home directory.

For user `ann`:

```text
/home/ann/.ssh
```

might be the full path.

For root:

```text
/root/.ssh
```

would be different.

---

# 81. Inspect `.ssh`

Run:

```bash
ls -la ~/.ssh
```

You may see:

```text
id_ed25519
id_ed25519.pub
authorized_keys
known_hosts
config
```

Not every system will contain every file.

Each serves a different purpose.

---

# 82. `authorized_keys`

On the remote server, public keys authorized for a user are commonly stored in:

```text
~/.ssh/authorized_keys
```

For example:

```text
/home/ubuntu/.ssh/authorized_keys
```

belongs to the `ubuntu` account.

If your public key is authorized there, SSH may allow you to authenticate as that user using the corresponding private key.

---

# 83. User Matters

Suppose your public key exists in:

```text
/home/ubuntu/.ssh/authorized_keys
```

You try:

```bash
ssh ec2-user@server
```

Authentication may fail because you are attempting to authenticate as:

```text
ec2-user
```

not:

```text
ubuntu
```

SSH authorization is associated with the target account.

Always verify the correct username.

---

# 84. Common Cloud SSH Usernames

Different Linux images may use different default usernames.

Examples you may encounter include:

```text
ubuntu
ec2-user
admin
debian
rocky
almalinux
```

Do not guess indefinitely.

Check the documentation for the image or environment you are using.

A correct key with the wrong username can still produce:

```text
Permission denied
```

---

# 85. Installing a Public Key Manually

Suppose your public key is:

```text
~/.ssh/id_ed25519.pub
```

Its contents can be added to the remote user's:

```text
~/.ssh/authorized_keys
```

The directory and files must have appropriate ownership and permissions.

A common secure setup is:

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

These are applied on the remote account's SSH directory and authorization file.

---

# 86. `ssh-copy-id`

Where available and password login is permitted, you may use:

```bash
ssh-copy-id user@server
```

This helps install your public key into the remote user's authorized keys.

For a specific public key:

```bash
ssh-copy-id -i ~/.ssh/id_ed25519.pub user@server
```

The `.pub` file is what gets copied.

Do not copy your private key into `authorized_keys`.

---

# 87. `known_hosts`

The client commonly stores SSH server host identities in:

```text
~/.ssh/known_hosts
```

This is different from:

```text
authorized_keys
```

Remember:

```text
authorized_keys
=
Which user keys may authenticate to this account?

known_hosts
=
Which server host identities has this client seen/trusted?
```

These files solve completely different problems.

---

# 88. First SSH Connection

The first time you connect to a server, you may see something similar to:

```text
The authenticity of host 'server.example.com' can't be established.
```

SSH may show a host-key fingerprint and ask whether you want to continue.

Do not automatically type:

```text
yes
```

on sensitive systems.

The purpose of the fingerprint is to let you verify that you are connecting to the intended server.

---

# 89. Host-Key Fingerprint

A fingerprint is a compact representation of the server's host public key.

Instead of manually comparing a very long key, you compare something like:

```text
SHA256:...
```

In managed environments, obtain the expected fingerprint through a trusted source when possible.

This helps protect against connecting to an impersonated server.

---

# 90. What Happens After Accepting a Host Key?

The server's host key information is recorded in:

```text
~/.ssh/known_hosts
```

On later connections, SSH compares the presented server identity with the stored information.

If it unexpectedly changes, SSH warns you.

That warning matters.

---

# 91. Host-Key Changed Warning

You may encounter:

```text
WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!
```

Do not immediately delete the warning.

Possible legitimate causes include:

* Server rebuilt.
* VM replaced.
* SSH host keys regenerated.
* IP address reassigned.
* DNS now points to a different server.

But it can also indicate a security problem.

Verify the new host key through a trusted source before accepting it.

---

# 92. `ssh-keygen -R`

After you have verified that a host-key change is legitimate, you can remove the old entry.

Example:

```bash
ssh-keygen -R server.example.com
```

For an IP:

```bash
ssh-keygen -R 10.20.1.15
```

Then reconnect and verify the new fingerprint.

Do not use this command merely to silence an unexplained security warning.

---

# 93. Connecting With a Specific Private Key

Use:

```bash
ssh -i /path/to/private-key user@server
```

Example:

```bash
ssh -i ~/.ssh/devops-lab.pem ubuntu@10.20.1.15
```

Breakdown:

```text
ssh
=
SSH client

-i
=
Identity file

~/.ssh/devops-lab.pem
=
Private key

ubuntu
=
Remote user

10.20.1.15
=
Remote server
```

---

# 94. Relative vs Absolute Key Paths

Suppose your key is:

```text
/home/ann/Downloads/devops.pem
```

This works:

```bash
ssh -i /home/ann/Downloads/devops.pem ubuntu@server
```

If you are already in:

```text
/home/ann/Downloads
```

you could use:

```bash
ssh -i ./devops.pem ubuntu@server
```

But:

```bash
ssh -i devops.pem ubuntu@server
```

depends on the file being in your current working directory.

This is why:

```text
No such file or directory
```

often comes from incorrect paths rather than SSH itself.

---

# 95. Verify the Key Exists

Before troubleshooting authentication, check:

```bash
ls -l ~/.ssh/devops-lab.pem
```

or:

```bash
ls -l /full/path/to/devops-lab.pem
```

If Linux reports:

```text
No such file or directory
```

fix the path first.

SSH cannot use a key it cannot find.

---

# 96. Private-Key Permissions

SSH protects private keys from overly permissive file access.

Suppose:

```bash
ls -l devops.pem
```

shows permissions allowing other users too much access.

SSH may warn:

```text
WARNING: UNPROTECTED PRIVATE KEY FILE!
```

and refuse to use the key.

A common fix for a private SSH key is:

```bash
chmod 600 devops.pem
```

---

# 97. What `600` Means Again

From Part 5:

```text
6
=
4 + 2
=
read + write

0
=
no permissions

0
=
no permissions
```

Therefore:

```text
600
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

For a private key, this limits access to the file owner.

---

# 98. Why You May Also See `chmod 400`

Cloud documentation sometimes recommends:

```bash
chmod 400 key.pem
```

This means:

```text
Owner:
read only

Group:
none

Others:
none
```

Both `400` and `600` can satisfy common OpenSSH private-key security expectations, depending on your workflow.

The important principle is:

```text
The private key must not be accessible to unauthorized users.
```

---

# 99. Do Not Use `chmod 777` on a Private Key

Never solve SSH key permission problems with:

```bash
chmod 777 key.pem
```

That makes the key far more exposed.

Remember:

```text
777
=
owner rwx
group rwx
others rwx
```

That is the opposite of protecting a secret.

---

# 100. SSH Verbose Mode

One of the most useful SSH troubleshooting tools is:

```bash
ssh -v user@server
```

For more detail:

```bash
ssh -vv user@server
```

or:

```bash
ssh -vvv user@server
```

Verbose output can reveal:

* Which configuration is being used.
* Which address is attempted.
* Which keys are tried.
* Which authentication methods are available.
* Where the connection fails.

---

# 101. SSH With Key and Verbose Mode

Example:

```bash
ssh -vvv -i ~/.ssh/devops.pem ubuntu@server.example.com
```

This is extremely useful when:

```text
The key exists.
The server is reachable.
But authentication still fails.
```

Do not paste complete verbose SSH output publicly without reviewing it. Logs can contain hostnames, usernames, network details, and other environment information.

---

# 102. SSH Permission Denied

A common error is:

```text
Permission denied (publickey)
```

or:

```text
Permission denied (publickey,password)
```

This is primarily an authentication clue.

It usually means you got much further than someone experiencing a timeout.

Conceptually:

```text
DNS
✓

Network route
✓

TCP connection
✓

SSH server
✓

Authentication
X
```

---

# 103. Common Causes of `Permission denied`

Possible causes include:

* Wrong username.
* Wrong private key.
* Corresponding public key not installed.
* Incorrect `authorized_keys`.
* Wrong ownership.
* Wrong `.ssh` permissions.
* Wrong `authorized_keys` permissions.
* Server does not permit that authentication method.
* Account restrictions.
* Key not being offered by client.

Do not assume:

```text
Permission denied
=
wrong key permissions
```

There are many possibilities.

---

# 104. Troubleshooting `Permission denied`

First verify the username:

```bash
ssh -i ~/.ssh/devops.pem ubuntu@server
```

Are you sure it is:

```text
ubuntu
```

rather than:

```text
ec2-user
```

or another account?

Then verify:

```bash
ls -l ~/.ssh/devops.pem
```

Then:

```bash
ssh -vvv -i ~/.ssh/devops.pem ubuntu@server
```

Look for evidence that the intended key is actually being offered.

---

# 105. Server-Side Key Troubleshooting

If you have another legitimate way to access the server, inspect the target user's:

```bash
ls -ld ~/.ssh
ls -l ~/.ssh/authorized_keys
```

Check:

* Correct user.
* Correct ownership.
* Appropriate permissions.
* Correct public key.
* No accidental formatting problems.

Common permissions are:

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

But also ensure ownership is correct.

---

# 106. Ownership Matters

Suppose:

```text
/home/ubuntu/.ssh/authorized_keys
```

belongs to:

```text
root
```

when it should belong to:

```text
ubuntu
```

That can cause authentication problems depending on SSH configuration and permissions.

Check:

```bash
ls -ld /home/ubuntu/.ssh
ls -l /home/ubuntu/.ssh/authorized_keys
```

If ownership is wrong, an administrator may need to correct it:

```bash
sudo chown -R ubuntu:ubuntu /home/ubuntu/.ssh
```

Be careful with recursive ownership changes.

Confirm the path before running the command.

---

# 107. SSH Connection Refused

Suppose:

```bash
ssh ubuntu@10.20.1.15
```

returns:

```text
Connection refused
```

Think back to Part 12.

This suggests:

```text
TCP connection reached a point where it was actively rejected.
```

Possible causes include:

* SSH service stopped.
* Nothing listening on port 22.
* SSH configured on another port.
* Service bound to another address.
* Network device actively rejecting traffic.

---

# 108. Troubleshooting SSH Connection Refused

On the server, if you have console or alternative access:

```bash
sudo ss -tlnp | grep ':22'
```

If nothing appears, check:

```bash
sudo systemctl status ssh
```

or:

```bash
sudo systemctl status sshd
```

depending on the distribution.

Also inspect:

```bash
sudo journalctl -u ssh
```

or:

```bash
sudo journalctl -u sshd
```

Do not change firewall rules if no SSH service is listening.

---

# 109. SSH Timeout

Suppose:

```bash
ssh ubuntu@10.20.1.15
```

waits and eventually returns:

```text
Connection timed out
```

Possible causes include:

* Wrong IP.
* Missing route.
* Firewall.
* Cloud security group.
* Network ACL.
* VPN missing.
* Private server not reachable from your network.
* Host unavailable.
* Port blocked.

This is different from authentication failure.

---

# 110. Troubleshooting SSH Timeout

Start with:

```bash
ip route get 10.20.1.15
```

Then, where appropriate:

```bash
ping -c 4 10.20.1.15
```

Remember that ping failure alone proves little.

Try verbose SSH:

```bash
ssh -vv ubuntu@10.20.1.15
```

Determine where it stops.

If this is a cloud server, verify:

* Correct public/private IP.
* Security group.
* Network ACL.
* Route.
* Internet gateway or VPN path where relevant.
* SSH port.

---

# 111. Private IP SSH Problem

Suppose your cloud server has:

```text
Private IP:
10.20.1.15
```

and you are sitting at home.

You run:

```bash
ssh ubuntu@10.20.1.15
```

and it times out.

The SSH service may be perfectly healthy.

Your laptop may simply have no route into:

```text
10.20.1.0/24
```

You might require:

* VPN.
* Bastion host.
* Cloud management service.
* Public IP.
* Private connectivity.

This is a routing and architecture issue, not necessarily an SSH problem.

---

# 112. Wrong SSH Port

Suppose:

```bash
ssh ubuntu@server
```

fails.

You discover:

```bash
sudo ss -tlnp | grep ssh
```

shows:

```text
0.0.0.0:2222
```

Then connect with:

```bash
ssh -p 2222 ubuntu@server
```

The SSH server was healthy.

The client was simply contacting the wrong port.

---

# 113. SSH Host-Key Warning

Suppose you see:

```text
WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!
```

Do not treat this as:

```text
Annoying SSH error
```

Treat it as:

```text
The server identity I previously stored does not match the identity being presented now.
```

Investigate why.

---

# 114. Host-Key Troubleshooting Process

Use:

```text
Host-key warning
      |
      v
Was server rebuilt?
      |
      v
Was IP reassigned?
      |
      v
Did DNS change?
      |
      v
Were SSH host keys regenerated?
      |
      v
Can expected fingerprint be verified?
      |
      v
Only then remove old known_hosts entry
```

Security warnings should be investigated, not bypassed blindly.

---

# 115. Remote Command Execution

SSH can execute a command remotely without opening an interactive shell.

Example:

```bash
ssh user@server hostname
```

The remote server runs:

```bash
hostname
```

and sends the output back.

Another example:

```bash
ssh user@server 'df -h'
```

This is useful in administration and automation.

---

# 116. Be Careful With Remote Commands

Remember:

```bash
ssh user@server 'command'
```

runs:

```text
command
```

on the remote server.

Before running destructive commands, verify:

* Correct hostname.
* Correct environment.
* Correct account.
* Correct path.

A command that is safe on your laptop may be dangerous on production.

---

# 117. Copying Files With `scp`

`scp` stands for:

```text
Secure Copy
```

It transfers files using SSH-based secure transport.

You can copy:

```text
Local -> Remote
Remote -> Local
```

and directories recursively.

---

# 118. SCP Mental Model

```text
Local computer
      |
      | SSH-secured transfer
      v
Remote Linux server
```

You need:

* Network connectivity.
* SSH access.
* Correct authentication.
* Correct source path.
* Correct destination path.
* Appropriate filesystem permissions.

---

# 119. SCP Syntax

General structure:

```bash
scp SOURCE DESTINATION
```

One side can be remote.

Remote syntax commonly looks like:

```text
user@server:/path
```

The colon is important.

---

# 120. Copy Local File to Remote Server

Suppose locally you have:

```text
~/Desktop/app.conf
```

You want to copy it to:

```text
/home/ubuntu/
```

on:

```text
10.20.1.15
```

Run locally:

```bash
scp ~/Desktop/app.conf ubuntu@10.20.1.15:/home/ubuntu/
```

Breakdown:

```text
scp
=
Secure copy command

~/Desktop/app.conf
=
Local source

ubuntu@10.20.1.15
=
Remote account and server

:/home/ubuntu/
=
Remote destination
```

---

# 121. Verify the Remote Copy

Connect:

```bash
ssh ubuntu@10.20.1.15
```

Then:

```bash
ls -l /home/ubuntu/app.conf
```

Do not assume a copy succeeded simply because you ran the command.

Always verify.

---

# 122. SCP With a Private Key

If SSH requires:

```text
~/Downloads/devops.pem
```

use:

```bash
scp -i ~/Downloads/devops.pem ~/Desktop/app.conf ubuntu@10.20.1.15:/home/ubuntu/
```

Here:

```text
-i
```

selects the identity file.

---

# 123. Important SCP Port Difference

For `ssh`, a custom port uses lowercase:

```bash
ssh -p 2222 user@server
```

For traditional `scp`, the port option is uppercase:

```bash
scp -P 2222 file.txt user@server:/home/user/
```

This difference causes beginner mistakes.

Remember:

```text
ssh
-p

scp
-P
```

---

# 124. Copy Remote File to Local Machine

Suppose the remote server contains:

```text
/var/log/myapp/app.log
```

and you want to download it to your current local directory.

Run locally:

```bash
scp ubuntu@10.20.1.15:/var/log/myapp/app.log .
```

The final:

```text
.
```

means:

```text
current local directory
```

---

# 125. Verify the Local Copy

Run:

```bash
pwd
```

Then:

```bash
ls -lh app.log
```

This confirms where the downloaded file went.

Many SCP mistakes happen because users forget which machine or directory they are currently working on.

---

# 126. Copy to a Specific Local Directory

Example:

```bash
scp ubuntu@10.20.1.15:/var/log/myapp/app.log ~/Downloads/
```

This means:

```text
Remote source
        |
        v
Local Downloads directory
```

Again, run the command from the machine receiving the file.

---

# 127. Copy a Directory Recursively

To copy a directory:

```bash
scp -r ~/Desktop/project ubuntu@10.20.1.15:/home/ubuntu/
```

`-r` means:

```text
recursive
```

It copies the directory and its contents.

---

# 128. Recursive SCP With a Key

Example:

```bash
scp -i ~/Downloads/devops.pem -r ~/Desktop/project ubuntu@10.20.1.15:/home/ubuntu/
```

Afterward:

```bash
ssh -i ~/Downloads/devops.pem ubuntu@10.20.1.15
```

Then:

```bash
ls -la /home/ubuntu/project
```

Verify the transfer.

---

# 129. Common SCP Path Mistake

Suppose you run:

```bash
scp file.txt ubuntu@server:/logs/
```

and receive:

```text
No such file or directory
```

Possible cause:

```text
/logs/
```

does not exist on the remote server.

Maybe you intended:

```text
/home/ubuntu/logs/
```

These are completely different paths.

Remember:

```text
/logs
```

means a directory directly under root.

```text
/home/ubuntu/logs
```

means a directory inside the user's home hierarchy.

---

# 130. Another Common SCP Mistake

You are SSHed into the remote server and run:

```bash
scp ~/Desktop/project ubuntu@server:/home/ubuntu/
```

You expect:

```text
~/Desktop/project
```

to refer to your laptop.

It does not.

Because the command is running on the remote server:

```text
~
```

refers to the remote user's home directory.

Always ask:

```text
Which machine am I currently typing on?
```

---

# 131. Local vs Remote Mental Model

Before every `scp` command, label both sides.

Example:

```text
LOCAL
/Users/ann/Desktop/project

REMOTE
ubuntu@10.20.1.15:/home/ubuntu/project
```

Then build:

```bash
scp -r /Users/ann/Desktop/project ubuntu@10.20.1.15:/home/ubuntu/
```

This simple habit prevents many mistakes.

---

# 132. SCP Permission Denied

Suppose:

```bash
scp app.conf ubuntu@server:/etc/myapp/
```

returns:

```text
Permission denied
```

SSH authentication may have succeeded.

The problem may be:

```text
ubuntu
```

does not have permission to write directly into:

```text
/etc/myapp/
```

That is a filesystem permission problem.

---

# 133. Safe Way to Copy Into a Protected Directory

Instead of trying to make `scp` run remote `sudo`, copy to a location you own:

```bash
scp app.conf ubuntu@server:/home/ubuntu/
```

Then connect:

```bash
ssh ubuntu@server
```

Inspect:

```bash
ls -l ~/app.conf
```

Then move it with appropriate privileges:

```bash
sudo mv ~/app.conf /etc/myapp/app.conf
```

Verify:

```bash
ls -l /etc/myapp/app.conf
```

This is easier to reason about and safer for beginners.

---

# 134. Be Careful With Configuration Ownership

After copying configuration files, verify:

```bash
ls -l /etc/myapp/app.conf
```

The service may require specific:

* Owner.
* Group.
* Permissions.

Do not assume that successfully copying the file means the application can safely use it.

---

# 135. SFTP

SFTP means:

```text
SSH File Transfer Protocol
```

It provides file-transfer functionality over SSH.

Start an interactive session:

```bash
sftp user@server
```

With a key:

```bash
sftp -i ~/.ssh/devops.pem user@server
```

---

# 136. SFTP Session

After connecting, you may see:

```text
sftp>
```

Common commands include:

```text
ls
pwd
cd
get
put
mkdir
exit
```

Be careful because you are managing both local and remote locations.

---

# 137. Download With SFTP

Inside SFTP:

```text
get app.log
```

downloads a remote file to your local SFTP working directory.

You can check remote location with:

```text
pwd
```

and local location with:

```text
lpwd
```

The `l` prefix commonly refers to:

```text
local
```

---

# 138. Upload With SFTP

Inside SFTP:

```text
put app.conf
```

uploads a local file into the current remote SFTP directory.

You can use:

```text
lcd
```

to change your local directory and:

```text
cd
```

to change the remote directory.

This distinction is essential.

---

# 139. SCP vs SFTP

At a beginner level:

```text
scp
=
Convenient command-oriented copying

sftp
=
Interactive SSH-based file-transfer session
```

Both rely on SSH infrastructure.

Organizations may choose one or another based on workflows and security policies.

---

# 140. SSH Configuration File Introduction

Typing this repeatedly:

```bash
ssh -i ~/.ssh/devops.pem -p 2222 ubuntu@server.example.com
```

becomes inconvenient.

The SSH client can use:

```text
~/.ssh/config
```

Example:

```text
Host dev-server
    HostName server.example.com
    User ubuntu
    Port 2222
    IdentityFile ~/.ssh/devops.pem
```

Then:

```bash
ssh dev-server
```

can use those settings.

---

# 141. Understanding SSH Config

Example:

```text
Host dev-server
```

defines the alias you type.

```text
HostName server.example.com
```

defines the actual destination.

```text
User ubuntu
```

defines the remote account.

```text
Port 2222
```

defines the SSH port.

```text
IdentityFile ~/.ssh/devops.pem
```

defines the private key.

This reduces repetitive commands and mistakes.

---

# 142. Protect SSH Configuration

A common permission for:

```text
~/.ssh/config
```

is:

```bash
chmod 600 ~/.ssh/config
```

Also ensure your `.ssh` directory is appropriately protected:

```bash
chmod 700 ~/.ssh
```

Do not make your SSH directory world-writable.

---

# 143. Complete SSH Troubleshooting Model

When this fails:

```bash
ssh user@server
```

do not think:

```text
SSH is broken.
```

Think:

```text
1. Is "server" correct?
        |
        v
2. Does the name resolve?
        |
        v
3. What IP does it resolve to?
        |
        v
4. Do I have a route?
        |
        v
5. Can TCP reach the SSH port?
        |
        v
6. Is sshd listening?
        |
        v
7. Is it listening on the expected address?
        |
        v
8. Is the server host key expected?
        |
        v
9. Am I using the correct username?
        |
        v
10. Am I using the correct authentication method?
        |
        v
11. Is the correct key being offered?
        |
        v
12. Is the public key authorized?
        |
        v
13. Are ownership and permissions correct?
```

This transforms SSH troubleshooting from guessing into diagnosis.

---

# 144. Reading SSH Errors as Evidence

Compare:

```text
Could not resolve hostname
```

Likely stage:

```text
DNS/name resolution
```

---

```text
Connection timed out
```

Likely stage:

```text
Network path / filtering / destination availability
```

---

```text
Connection refused
```

Likely stage:

```text
TCP reached destination but SSH is not accepting there
```

---

```text
Permission denied (publickey)
```

Likely stage:

```text
Authentication
```

---

```text
REMOTE HOST IDENTIFICATION HAS CHANGED
```

Likely stage:

```text
Server identity verification
```

The error message tells you where to start.

---

# 145. Remote Linux Administration Lab

You will now combine DNS, SSH, key authentication, file transfers, permissions, and troubleshooting.

You need two systems where possible:

```text
System A
Client

System B
Linux server
```

If you only have one machine, you can still practice many DNS and SSH commands, but a VM or disposable cloud server provides a better experience.

Do not expose a lab SSH server directly to the Internet without appropriate security controls.

---

# 146. Lab Step 1: Inspect the Client

On your client:

```bash
whoami
hostname
pwd
```

Then:

```bash
ip -br addr
```

Write down:

```text
Client username:
Client hostname:
Client IP:
```

---

# 147. Lab Step 2: Inspect SSH Client

Run:

```bash
command -v ssh
```

Then:

```bash
ssh -V
```

Confirm that an SSH client is installed.

---

# 148. Lab Step 3: Inspect DNS Configuration

Run:

```bash
cat /etc/resolv.conf
```

Then:

```bash
ls -l /etc/resolv.conf
```

Record:

```text
Configured resolver:
Is resolv.conf a regular file or symbolic link?
```

Do not modify it.

---

# 149. Lab Step 4: Perform DNS Queries

Run:

```bash
nslookup example.com
```

Then:

```bash
dig example.com
```

Then:

```bash
dig +short example.com
```

Then:

```bash
host example.com
```

Compare the output.

---

# 150. Lab Step 5: Query Different Record Types

Run:

```bash
dig example.com A
```

Then:

```bash
dig example.com AAAA
```

Then:

```bash
dig example.com NS
```

Observe how the answer changes based on the record type.

---

# 151. Lab Step 6: Test a Nonexistent Name

Use a deliberately invalid subdomain under a domain you control for a predictable lab, or use a clearly fabricated test name in an appropriate reserved namespace.

Run a query such as:

```bash
dig does-not-exist.example.invalid
```

Observe the DNS status.

Look for:

```text
NXDOMAIN
```

Understand:

```text
The DNS system responded,
but the requested domain name does not exist.
```

That differs from being unable to contact a DNS resolver at all.

---

# 152. Lab Step 7: Identify the Remote Server

Record:

```text
Remote hostname:
Remote IP:
Remote SSH username:
Remote SSH port:
```

Do not continue until you know these values.

For example:

```text
Remote hostname:
dev-server.example.com

Remote IP:
10.20.1.15

Remote username:
ubuntu

Remote port:
22
```

---

# 153. Lab Step 8: Verify Name Resolution

If you have a hostname:

```bash
dig +short dev-server.example.com
```

Compare the returned IP with the expected server IP.

If they differ, investigate before connecting.

---

# 154. Lab Step 9: Inspect the Route

Run:

```bash
ip route get 10.20.1.15
```

Verify Linux has a route to the destination.

Remember:

```text
Having a route
```

does not prove:

```text
SSH is reachable.
```

It only tells you how Linux intends to send the traffic.

---

# 155. Lab Step 10: Generate a Lab Key Pair

On the client:

```bash
ssh-keygen -t ed25519 -f ~/.ssh/veriqta-linux-lab
```

You will create:

```text
~/.ssh/veriqta-linux-lab
```

and:

```text
~/.ssh/veriqta-linux-lab.pub
```

Inspect:

```bash
ls -l ~/.ssh/veriqta-linux-lab*
```

Identify which is private and which is public.

---

# 156. Lab Step 11: Inspect the Public Key

Run:

```bash
cat ~/.ssh/veriqta-linux-lab.pub
```

You may safely inspect your public key.

Do not display your private key merely for practice.

You should know it exists without printing its contents.

---

# 157. Lab Step 12: Protect the Private Key

Run:

```bash
chmod 600 ~/.ssh/veriqta-linux-lab
```

Verify:

```bash
ls -l ~/.ssh/veriqta-linux-lab
```

You should see permissions equivalent to:

```text
-rw-------
```

Explain to yourself what each permission means.

---

# 158. Lab Step 13: Install the Public Key

If password SSH access is available:

```bash
ssh-copy-id -i ~/.ssh/veriqta-linux-lab.pub user@server
```

Otherwise, use the authorized method provided by your cloud provider or administrator to place the public key in:

```text
~/.ssh/authorized_keys
```

on the remote account.

Never upload:

```text
~/.ssh/veriqta-linux-lab
```

the private key to the remote server.

---

# 159. Lab Step 14: Connect With the Key

Run:

```bash
ssh -i ~/.ssh/veriqta-linux-lab user@server
```

If connecting for the first time, carefully inspect the host-key prompt.

Verify the fingerprint if you have a trusted source.

Then connect.

---

# 160. Lab Step 15: Prove You Are Remote

After connecting:

```bash
whoami
```

Then:

```bash
hostname
```

Then:

```bash
pwd
```

Then:

```bash
ip -br addr
```

Compare these values with your local machine.

This proves that your shell is now running on the remote system.

---

# 161. Lab Step 16: Create a Remote Workspace

On the remote server:

```bash
mkdir -p ~/veriqta-remote-lab/{logs,configs,scripts}
```

Verify:

```bash
find ~/veriqta-remote-lab -maxdepth 2 -type d
```

You have created:

```text
veriqta-remote-lab/
|
+-- logs/
+-- configs/
+-- scripts/
```

---

# 162. Lab Step 17: Create a Local File

Exit the remote session:

```bash
exit
```

On your local machine:

```bash
mkdir -p ~/veriqta-client-lab
```

Create:

```bash
printf '%s\n' 'environment=development' > ~/veriqta-client-lab/app.conf
```

Verify:

```bash
cat ~/veriqta-client-lab/app.conf
```

---

# 163. Lab Step 18: Copy Local to Remote

Run locally:

```bash
scp -i ~/.ssh/veriqta-linux-lab ~/veriqta-client-lab/app.conf user@server:~/veriqta-remote-lab/configs/
```

Then connect:

```bash
ssh -i ~/.ssh/veriqta-linux-lab user@server
```

Verify:

```bash
cat ~/veriqta-remote-lab/configs/app.conf
```

---

# 164. Lab Step 19: Create a Remote Log

On the remote server:

```bash
printf '%s\n' \
'INFO application started' \
'INFO database connected' \
'ERROR payment service unavailable' \
> ~/veriqta-remote-lab/logs/app.log
```

Verify:

```bash
cat ~/veriqta-remote-lab/logs/app.log
```

---

# 165. Lab Step 20: Copy Remote to Local

Exit:

```bash
exit
```

Then locally:

```bash
scp -i ~/.ssh/veriqta-linux-lab user@server:~/veriqta-remote-lab/logs/app.log ~/veriqta-client-lab/
```

Verify:

```bash
cat ~/veriqta-client-lab/app.log
```

You have now performed:

```text
Local -> Remote
```

and:

```text
Remote -> Local
```

transfers.

---

# 166. Lab Step 21: Copy a Directory

Create locally:

```bash
mkdir -p ~/veriqta-client-lab/scripts
```

Create two scripts:

```bash
printf '%s\n' '#!/bin/bash' 'hostname' > ~/veriqta-client-lab/scripts/show-host.sh
```

```bash
printf '%s\n' '#!/bin/bash' 'date' > ~/veriqta-client-lab/scripts/show-date.sh
```

Copy recursively:

```bash
scp -i ~/.ssh/veriqta-linux-lab -r ~/veriqta-client-lab/scripts user@server:~/veriqta-remote-lab/
```

Verify remotely.

---

# 167. Lab Step 22: Create an Intentional Wrong-Key Failure

If you have another nonmatching test key, try connecting with it.

You may receive:

```text
Permission denied (publickey)
```

Do not alter production authentication for this exercise.

The purpose is to understand:

```text
Network connection succeeded
but
user authentication failed.
```

---

# 168. Lab Step 23: Create an Intentional Wrong-Username Failure

Use a nonexistent or unauthorized lab username:

```bash
ssh -i ~/.ssh/veriqta-linux-lab wronguser@server
```

You may receive:

```text
Permission denied
```

Now compare this with a network timeout.

These errors occur at different stages.

---

# 169. Lab Step 24: Create an Intentional Wrong-Port Test

If the server uses port 22, test an unused lab port:

```bash
ssh -p 2222 user@server
```

You might see:

```text
Connection refused
```

or:

```text
Connection timed out
```

depending on the network.

Do not assume the exact error will always be the same.

Network filtering affects the result.

---

# 170. Lab Step 25: Use Verbose SSH

Run:

```bash
ssh -vv -i ~/.ssh/veriqta-linux-lab user@server
```

Identify:

```text
Destination
Port
Identity file
Authentication methods
Key being offered
Authentication result
```

Do not memorize every debug line.

Learn to identify the stage.

---

# 171. Lab Step 26: Inspect Server SSH Socket

On the server:

```bash
sudo ss -tlnp | grep ':22'
```

if SSH uses port 22.

Identify:

```text
Listening address:
Port:
Process:
```

Then ask:

```text
Is SSH listening where I expected?
```

---

# 172. Lab Step 27: Inspect SSH Service

Depending on your distribution:

```bash
systemctl status ssh
```

or:

```bash
systemctl status sshd
```

Identify:

```text
Active?
Enabled?
Failed?
```

Do not stop the SSH service on a remote server unless you have another recovery path.

You could lock yourself out.

---

# 173. Important Remote Administration Safety Rule

Before changing:

* SSH configuration.
* SSH port.
* Firewall rules.
* Network interfaces.
* Routing.
* User permissions.
* `authorized_keys`.

on a remote machine, ask:

```text
If I make a mistake, how will I get back into this server?
```

Possible recovery methods include:

* Cloud console.
* Serial console.
* Another administrator session.
* Bastion host.
* Management agent.
* Snapshot or image.
* Provider recovery tools.

Remote administration requires planning for mistakes.

---

# 174. Lab Step 28: Use SFTP

From your local machine:

```bash
sftp -i ~/.ssh/veriqta-linux-lab user@server
```

Inside SFTP:

```text
pwd
```

Then:

```text
lpwd
```

Observe:

```text
pwd
=
remote directory

lpwd
=
local directory
```

---

# 175. Lab Step 29: Upload Through SFTP

Change your local SFTP directory if necessary:

```text
lcd ~/veriqta-client-lab
```

Change remote directory:

```text
cd veriqta-remote-lab/configs
```

Upload:

```text
put app.conf
```

Verify:

```text
ls
```

Then exit:

```text
exit
```

---

# 176. Lab Step 30: Create an SSH Alias

Locally:

```bash
nano ~/.ssh/config
```

Add:

```text
Host veriqta-lab
    HostName server.example.com
    User your-user
    IdentityFile ~/.ssh/veriqta-linux-lab
    Port 22
```

Replace the values with your actual lab information.

Save and exit.

Protect:

```bash
chmod 600 ~/.ssh/config
```

---

# 177. Lab Step 31: Connect Using the Alias

Instead of:

```bash
ssh -i ~/.ssh/veriqta-linux-lab user@server.example.com
```

run:

```bash
ssh veriqta-lab
```

If correctly configured, SSH uses the settings from:

```text
~/.ssh/config
```

This becomes extremely useful when managing many servers.

---

# 178. Lab Step 32: Run a Remote Command

From your local machine:

```bash
ssh veriqta-lab hostname
```

Then:

```bash
ssh veriqta-lab 'uptime'
```

Then:

```bash
ssh veriqta-lab 'df -h'
```

Observe that you receive remote command output without remaining in an interactive remote shell.

---

# 179. Final Troubleshooting Scenario

You receive this report:

```text
I cannot SSH to production-web.example.com.
```

Do not start changing settings.

Investigate systematically.

---

# 180. Step 1: Resolve the Name

Run:

```bash
dig +short production-web.example.com
```

Suppose:

```text
10.20.1.15
```

is returned.

Now DNS produced an address.

---

# 181. Step 2: Confirm Local Resolution

Run:

```bash
getent hosts production-web.example.com
```

If it returns another address, inspect:

```bash
grep production-web.example.com /etc/hosts
```

You may have a local override.

---

# 182. Step 3: Check Routing

Run:

```bash
ip route get 10.20.1.15
```

If no usable route exists, solve the routing or connectivity problem before investigating SSH keys.

---

# 183. Step 4: Attempt SSH Verbosely

Run:

```bash
ssh -vv ubuntu@production-web.example.com
```

Observe where the process stops.

If it times out before establishing TCP:

```text
Do not troubleshoot authorized_keys yet.
```

Authentication has not started.

---

# 184. Step 5: Inspect the Server Listener

If you have console access:

```bash
sudo ss -tlnp | grep ':22'
```

Suppose:

```text
0.0.0.0:22
```

appears.

The SSH server is listening.

Next investigate the network path and security controls if the client still cannot reach it.

---

# 185. Step 6: Inspect SSH Service

Run:

```bash
sudo systemctl status ssh
```

or:

```bash
sudo systemctl status sshd
```

If failed:

```bash
sudo journalctl -u ssh
```

or:

```bash
sudo journalctl -u sshd
```

Find the actual service failure.

---

# 186. Step 7: Investigate Authentication

Suppose the TCP connection succeeds but you receive:

```text
Permission denied (publickey)
```

Now investigate:

```text
Correct username?
Correct private key?
Correct public key installed?
Correct authorized_keys?
Correct ownership?
Correct permissions?
Correct SSH configuration?
```

This is the appropriate stage for key troubleshooting.

---

# 187. Step 8: Verify Host Identity

If SSH reports:

```text
REMOTE HOST IDENTIFICATION HAS CHANGED
```

stop and verify the server identity.

Do not bypass it merely because you need quick access.

Only after confirming the change is legitimate should you update `known_hosts`.

---

# 188. Step 9: Verify Successful Login

Once connected:

```bash
whoami
hostname
pwd
```

Confirm:

```text
Correct user?
Correct server?
Correct environment?
```

Before making administrative changes.

This is particularly important when servers have similar names.

---

# 189. Step 10: Document the Root Cause

Suppose the actual cause was:

```text
Security group allowed SSH from the old office IP but not the new administrator network.
```

Do not document:

```text
SSH issue fixed.
```

Document:

```text
Symptom:
SSH connection timed out.

Evidence:
DNS resolved correctly.
Route existed.
sshd listened on TCP 22.
Connection from approved network worked.
Connection from new administrator network timed out.

Root cause:
Cloud security rule did not allow TCP 22 from the new administrator network.

Fix:
Authorized required source according to security policy.

Verification:
SSH connection succeeded from intended administrator network.
```

That is useful incident documentation.

---

# 190. What You Should Now Understand

Before moving to Part 14, you should confidently understand:

* What DNS is.
* Why DNS exists.
* What a domain name is.
* What an FQDN is.
* How DNS resolution works at a beginner level.
* Why DNS and connectivity are different problems.
* The basic DNS hierarchy.
* What root DNS infrastructure does.
* What top-level domains are.
* What authoritative DNS servers do.
* What recursive resolvers do.
* Why DNS caching exists.
* What TTL means.
* What an A record is.
* What an AAAA record is.
* What a CNAME record is.
* What an MX record is.
* What an NS record is.
* What a TXT record is.
* What a PTR record is.
* What `/etc/resolv.conf` is.
* Why `127.0.0.53` may appear as a resolver.
* Why `/etc/resolv.conf` should not always be manually edited.
* What `/etc/hosts` does.
* How `/etc/hosts` differs from DNS.
* How stale hosts entries cause problems.
* How to use `nslookup`.
* How to use `dig`.
* How to use `dig +short`.
* How to query specific DNS record types.
* How to query a specific DNS server.
* What `NXDOMAIN` means.
* Why `NOERROR` does not guarantee your requested record exists.
* How to use `host`.
* How to troubleshoot DNS systematically.
* What SSH is.
* Why Linux administrators use SSH.
* What an SSH client is.
* What an SSH server is.
* What `sshd` is.
* Why SSH commonly uses TCP port 22.
* How an SSH connection progresses from DNS through authentication.
* What password authentication is.
* What key-based authentication is.
* What public keys are.
* What private keys are.
* Why private keys must remain secret.
* Why private keys should not be copied to remote servers.
* How to use `ssh-keygen`.
* What `~/.ssh` is.
* What `authorized_keys` does.
* What `known_hosts` does.
* Why `authorized_keys` and `known_hosts` are completely different.
* What server host keys do.
* Why host-key warnings matter.
* How `ssh-keygen -R` is used after legitimate verification.
* How to use `ssh user@server`.
* How to use `ssh -i`.
* How to connect to a nondefault SSH port.
* Why the correct remote username matters.
* Why private-key permissions matter.
* What `chmod 600` means.
* Why `chmod 777` is dangerous for private keys.
* How `ssh -v`, `-vv`, and `-vvv` help troubleshooting.
* What `Permission denied (publickey)` indicates.
* What SSH connection refusal indicates.
* What SSH timeout indicates.
* Why a private IP may not be reachable from your laptop.
* How to execute remote commands with SSH.
* What `scp` does.
* How to copy local files to remote systems.
* How to copy remote files locally.
* How to recursively copy directories.
* Why `scp -P` differs from `ssh -p`.
* Why SCP path mistakes are common.
* Why you must know which machine a command is running on.
* How filesystem permissions can cause SCP failures.
* Why protected directories should be handled carefully.
* What SFTP is.
* How `get` and `put` work.
* What `pwd` and `lpwd` mean in SFTP.
* What `~/.ssh/config` does.
* How SSH aliases simplify administration.
* How to troubleshoot remote access from DNS through authentication.

---

# 191. Hands-On Practice

Complete these tasks yourself:

1. Inspect `/etc/resolv.conf`.
2. Identify your configured DNS resolver.
3. Determine whether `/etc/resolv.conf` is a symbolic link.
4. Query a domain with `nslookup`.
5. Query the same domain with `dig`.
6. Query it with `host`.
7. Query only its A records.
8. Query its AAAA records.
9. Query its NS records.
10. Query its MX records if available.
11. Use `dig +short`.
12. Query a specific resolver.
13. Explain what TTL means.
14. Explain `NXDOMAIN`.
15. Explain why `NOERROR` can have an empty answer.
16. Inspect `/etc/hosts`.
17. Explain how `/etc/hosts` can override expected DNS behavior.
18. Identify your SSH client.
19. Display the SSH client version.
20. Explain the difference between `ssh` and `sshd`.
21. Identify the default SSH port.
22. Generate an Ed25519 key pair.
23. Identify the private key.
24. Identify the public key.
25. Protect the private key.
26. Explain its permissions.
27. Inspect `~/.ssh`.
28. Explain `authorized_keys`.
29. Explain `known_hosts`.
30. Explain the difference between user keys and server host keys.
31. Install a public key on a lab server.
32. Connect with the matching private key.
33. Verify the remote username.
34. Verify the remote hostname.
35. Verify the remote IP addresses.
36. Exit the session.
37. Execute `hostname` remotely without an interactive shell.
38. Use verbose SSH.
39. Identify which key SSH attempts.
40. Create an intentional wrong-username test.
41. Observe the authentication failure.
42. Test an unused SSH port.
43. Compare refusal or timeout with authentication failure.
44. Create a local file.
45. Copy it to the remote server with SCP.
46. Verify it remotely.
47. Create a remote file.
48. Copy it locally.
49. Verify it locally.
50. Create a local directory with several files.
51. Copy it recursively.
52. Explain `scp -r`.
53. Explain `scp -P`.
54. Create an SFTP session.
55. Display the remote SFTP working directory.
56. Display the local SFTP working directory.
57. Upload a file.
58. Download a file.
59. Exit SFTP.
60. Create an SSH client alias.
61. Connect through the alias.
62. Run a remote command through the alias.
63. Inspect the remote SSH listening socket.
64. Inspect the SSH service.
65. Explain why stopping SSH remotely can be dangerous.
66. Explain what `Permission denied (publickey)` tells you.
67. Explain what `Connection refused` tells you.
68. Explain what `Connection timed out` tells you.
69. Explain what a host-key warning tells you.
70. Write your own SSH troubleshooting checklist.

---

# 192. Beginner Knowledge Check

1. What does DNS stand for?
2. Why does DNS exist?
3. What is a domain name?
4. What is an FQDN?
5. What is DNS resolution?
6. Why must a hostname normally be resolved before an IP connection is made?
7. What is a DNS resolver?
8. What is an authoritative DNS server?
9. What is DNS caching?
10. What does TTL mean?
11. What does an A record contain?
12. What does an AAAA record contain?
13. What does a CNAME record do?
14. What is an MX record used for?
15. What is an NS record?
16. What is a TXT record commonly used for?
17. What is a PTR record?
18. What is `/etc/resolv.conf`?
19. What does `nameserver` mean inside it?
20. Why might `127.0.0.53` appear there?
21. Why should `/etc/resolv.conf` not always be edited directly?
22. What is `/etc/hosts`?
23. How does `/etc/hosts` differ from DNS?
24. Can a stale `/etc/hosts` entry cause one server to resolve a name differently?
25. What does `nslookup` do?
26. What does `dig` do?
27. What does `dig +short` do?
28. How do you request an MX record with `dig`?
29. What does `dig @SERVER NAME` do?
30. What does `NXDOMAIN` mean?
31. Can `NOERROR` occur without the record you expected?
32. What does `host` do?
33. If an IP works but a hostname does not, what area should you investigate?
34. Does successful DNS resolution prove the application is reachable?
35. What does SSH stand for?
36. Why is SSH used?
37. What is an SSH client?
38. What is an SSH server?
39. What is `sshd`?
40. What is SSH's common default port?
41. Does SSH have to use port 22?
42. What happens before SSH authentication?
43. What is password authentication?
44. What is public-key authentication?
45. What is a public key?
46. What is a private key?
47. Which key must remain secret?
48. Should the private key be copied into `authorized_keys`?
49. What does `ssh-keygen` do?
50. What does `-t ed25519` mean?
51. Where are SSH files commonly stored?
52. What is `authorized_keys`?
53. What is `known_hosts`?
54. Are `authorized_keys` and `known_hosts` interchangeable?
55. What does a server host key identify?
56. Why should you verify first-time SSH fingerprints?
57. What does a changed host-key warning mean?
58. Should you automatically delete a conflicting `known_hosts` entry?
59. What does `ssh-keygen -R` do?
60. What is the syntax for basic SSH login?
61. What does `ssh -i` do?
62. What does `ssh -p` do?
63. Why does the remote username matter?
64. Why do private-key permissions matter?
65. What does permission `600` mean?
66. Why is `777` inappropriate for a private key?
67. What does `ssh -v` provide?
68. What does `Permission denied (publickey)` suggest?
69. Does it normally indicate DNS failed?
70. What does `Connection refused` suggest?
71. What does SSH timeout suggest?
72. Why might SSH to a private IP from home fail?
73. How can you execute one command remotely?
74. What does SCP do?
75. What is the basic SCP syntax?
76. How do you copy local to remote?
77. How do you copy remote to local?
78. What does `scp -r` do?
79. What does `scp -i` do?
80. What option specifies a custom SCP port?
81. Why is `/logs` different from `~/logs`?
82. Why does your current machine matter when using SCP?
83. Why might SCP to `/etc` return permission denied?
84. What is a safer way to place a file into a privileged directory?
85. What is SFTP?
86. What does `get` do in SFTP?
87. What does `put` do?
88. What does `pwd` show in SFTP?
89. What does `lpwd` show?
90. What does `~/.ssh/config` do?
91. Why are SSH aliases useful?
92. Why should you verify `whoami` after connecting?
93. Why should you verify `hostname` after connecting?
94. Why is stopping SSH remotely dangerous?
95. What should you investigate first for a hostname-resolution failure?
96. What should you investigate first for connection refusal?
97. What should you investigate for a timeout?
98. What should you investigate for `Permission denied (publickey)`?
99. What should you do when the remote host identification unexpectedly changes?
100. What is the most important SSH troubleshooting principle?

---

# 193. Beginner Knowledge Check Answers

1. Domain Name System.

2. To provide a distributed naming system that lets applications use names instead of manually depending on IP addresses.

3. A hierarchical name used within DNS.

4. A Fully Qualified Domain Name, representing the complete DNS name of a host.

5. The process of obtaining DNS information for a name.

6. IP networking requires an address to route traffic to the destination.

7. A service that receives DNS queries and obtains or returns DNS answers.

8. A DNS server that provides authoritative information for a DNS zone.

9. Temporarily storing DNS answers to reduce repeated lookups.

10. Time To Live.

11. An IPv4 address for a name.

12. An IPv6 address for a name.

13. It aliases one DNS name to another canonical DNS name.

14. Identifying mail servers for a domain.

15. A record identifying authoritative name servers.

16. Text-based information such as verification and policy data.

17. A record commonly used for reverse DNS mapping.

18. A Linux resolver-configuration file or interface to resolver configuration.

19. It identifies a DNS resolver the system can use.

20. A local system resolver stub such as `systemd-resolved` may be handling queries.

21. Another network-management service may generate it and overwrite manual changes.

22. A local hostname-to-address mapping file.

23. `/etc/hosts` is local to a machine. DNS is a distributed naming system.

24. Yes.

25. It performs DNS queries.

26. It performs detailed DNS queries and exposes DNS response information.

27. It displays compact query results.

28. `dig example.com MX`.

29. It sends the DNS query to the specified DNS server.

30. The queried domain name does not exist according to the responding DNS system.

31. Yes.

32. It performs DNS lookups with relatively concise output.

33. Name resolution, including `/etc/hosts`, resolver configuration, and DNS.

34. No.

35. Secure Shell.

36. Secure remote communication and administration.

37. Software that initiates an SSH connection.

38. Software that accepts SSH connections.

39. The common OpenSSH server daemon.

40. TCP 22.

41. No.

42. Name resolution if needed, routing, TCP connection establishment, SSH negotiation, and server identity verification occur before successful user authentication.

43. Authentication using the remote account's password.

44. Authentication using cryptographic proof associated with a key pair.

45. The distributable part of a cryptographic key pair used by the remote system to verify authentication.

46. The secret part of the key pair controlled by the client.

47. The private key.

48. No.

49. It creates and manages SSH keys.

50. Generate an Ed25519 key.

51. `~/.ssh`.

52. A file containing public keys authorized to authenticate to a particular account.

53. A client-side file containing known SSH server identities.

54. No.

55. The SSH server.

56. To verify that you are connecting to the intended server.

57. The server identity presented now differs from the one previously stored.

58. No. Investigate and verify the new identity first.

59. It removes stored host-key entries for the specified host from known-hosts data.

60. `ssh user@server`.

61. Specifies an identity file, usually a private key.

62. Specifies the destination SSH port.

63. Authentication and authorization occur for a particular remote account.

64. SSH may reject private keys that unauthorized users can access.

65. Owner read and write, no permissions for group or others.

66. It exposes the key to other users and violates least privilege.

67. Debugging information about the SSH connection.

68. The network connection likely progressed to authentication, but the offered credentials were not accepted.

69. No.

70. The TCP endpoint actively rejected the connection, often because SSH is not listening at the expected address and port.

71. A network-path, filtering, routing, destination, or reachability problem may exist.

72. Private addresses are not generally directly routable from the public Internet, and you may lack a VPN or other route into that network.

73. `ssh user@server 'command'`.

74. Securely copies files using SSH-based transport.

75. `scp SOURCE DESTINATION`.

76. Use a local path as the source and `user@server:/path` as the destination.

77. Use `user@server:/path` as the source and a local path as the destination.

78. Recursively copies directories.

79. Specifies the SSH identity file.

80. Uppercase `-P`.

81. `/logs` is under the filesystem root. `~/logs` is inside the current user's home directory.

82. Local paths are interpreted by the machine where the SCP command is running.

83. The remote user may not have filesystem write permission there.

84. Copy into a directory you own, SSH to the server, then use `sudo` to move the file if authorized.

85. SSH File Transfer Protocol.

86. Downloads a remote file.

87. Uploads a local file.

88. The current remote directory.

89. The current local directory.

90. It stores SSH client configuration such as aliases, usernames, hosts, ports, and identity files.

91. They reduce repetitive commands and configuration mistakes.

92. To verify which remote account you are using.

93. To confirm that you connected to the intended machine.

94. You can terminate your only remote-access path and lock yourself out.

95. The exact hostname, local host mappings, resolver configuration, and DNS response.

96. Whether the SSH service is running and listening on the expected address and port.

97. Routing, firewalls, cloud security rules, network ACLs, VPN/private connectivity, destination availability, and the requested port.

98. Username, private key, key selection, public-key installation, `authorized_keys`, ownership, permissions, and SSH authentication configuration.

99. Stop and verify the server's new host-key fingerprint through a trusted source before updating the stored identity.

100. Identify the exact stage where the connection fails before changing anything.

# 194. Final Mental Model

DNS and SSH should now fit into one complete sequence:

```text
You type:

ssh ubuntu@server.example.com

             |
             v

      server.example.com
             |
             v
       Name resolution
             |
             v
           DNS
             |
             v
        10.20.1.15
             |
             v
       Routing decision
             |
             v
       Network path
             |
             v
      10.20.1.15:22
             |
             v
           sshd
             |
             v
   Server identity verification
             |
             v
      User authentication
             |
      +------+------+
      |             |
      v             v
   Password      SSH key
                    |
                    v
              Private key
              on client
                    |
                    v
              Public key
              authorized
              on server
                    |
                    v
          Authentication succeeds
                    |
                    v
              Secure session
                    |
                    v
               Linux shell
```

When something fails, locate the stage.

```text
Could not resolve hostname
        |
        v
DNS / name resolution
```

```text
Connection timed out
        |
        v
Network path / filtering / reachability
```

```text
Connection refused
        |
        v
Destination port / listener / SSH service
```

```text
Permission denied (publickey)
        |
        v
User authentication
```

```text
REMOTE HOST IDENTIFICATION HAS CHANGED
        |
        v
Server identity verification
```

And for file transfers:

```text
Local file
    |
    v
scp / sftp
    |
    v
SSH authentication
    |
    v
Encrypted network connection
    |
    v
Remote filesystem
    |
    v
Linux permissions and ownership
```

This is the key lesson of Part 13.

When:

```bash
ssh user@server
```

fails, there is no single thing called an "SSH problem."

DNS may have failed.

The hostname may resolve to the wrong address.

Linux may have no route.

A firewall may drop TCP 22.

`sshd` may not be running.

SSH may be listening on another port.

The username may be wrong.

The private key may be wrong.

The public key may not be authorized.

File ownership may be wrong.

The server identity may have legitimately changed, or the warning may indicate something that requires security investigation.

A strong Linux administrator does not randomly replace keys, open ports, delete `known_hosts`, or restart servers.

They identify the stage where the connection stops, gather evidence, fix the actual cause, and verify the complete connection path.
