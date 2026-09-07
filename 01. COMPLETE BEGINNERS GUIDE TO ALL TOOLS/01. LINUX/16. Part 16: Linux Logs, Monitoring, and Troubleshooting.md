# Part 16: Linux Logs, Monitoring, and Troubleshooting

In Part 15, you learned how to investigate CPU, memory, swap, load average, processes, and disk I/O.

Now we move into one of the most important Linux administration skills: using logs and system evidence to understand what actually happened.

When something fails, beginners often react to the visible symptom:

```text
Website is down
→ Restart Nginx

SSH does not work
→ Restart SSH

Application crashes
→ Start it again

Disk fills up
→ Delete some files
```

Those actions may temporarily restore service, but they do not necessarily explain the failure.

A stronger Linux administrator asks:

```text
What happened?
When did it happen?
What component failed?
What changed immediately before it?
What evidence supports the suspected cause?
What is the safest fix?
How can I prove the fix worked?
```

Logs help answer those questions.

By the end of Part 16, you should be able to:

* Understand why Linux systems generate logs.
* Know where common logs are stored.
* Distinguish system, application, authentication, and kernel logs.
* Use `journalctl`.
* Filter logs by service, time, boot, priority, and other criteria.
* Follow logs while an event happens.
* Search traditional log files.
* Understand `dmesg`.
* Understand timestamps.
* Understand basic log rotation.
* Use logs alongside commands from previous parts.
* Separate symptoms from root causes.
* Establish the scope of an incident.
* Build and test troubleshooting hypotheses.
* Verify that a fix actually solved the problem.
* Investigate common Linux incidents methodically.

---

# 1. Why Linux Systems Generate Logs

A Linux system constantly performs activities.

Examples include:

* Starting services.
* Stopping services.
* Authenticating users.
* Creating network connections.
* Running applications.
* Loading kernel drivers.
* Mounting filesystems.
* Running scheduled jobs.
* Encountering errors.
* Rejecting authentication attempts.
* Starting during boot.
* Detecting hardware problems.

If none of this activity were recorded, troubleshooting would be much harder.

Imagine a service stops at 03:17.

Without logs:

```text
03:17
Service stopped

Why?

Unknown
```

With logs:

```text
03:16:58 Configuration reloaded
03:17:01 Configuration parsing failed
03:17:01 Invalid directive on line 42
03:17:01 Service exited with status 1
```

Now you have evidence.

---

# 2. What Is a Log?

A log is a record of events produced by a system, application, service, or component.

A simplified log entry might look like:

```text
Aug 27 14:22:16 web01 sshd[4812]: Failed password for admin from 203.0.113.10 port 51922 ssh2
```

This single line contains several useful pieces of information:

```text
Aug 27 14:22:16
|
+-- When it happened

web01
|
+-- Which host recorded it

sshd
|
+-- Which service produced it

[4812]
|
+-- Process ID

Failed password...
|
+-- Event/message
```

Not every log format is identical.

Applications may use completely different formats.

---

# 3. Logs Answer Questions

Logs are most useful when you approach them with a question.

Instead of:

```text
Let me look through every log.
```

ask:

```text
Why did nginx fail at approximately 14:20?
```

or:

```text
Was this SSH login rejected because of authentication or networking?
```

or:

```text
Did the kernel kill the application because of memory pressure?
```

This makes your search focused.

---

# 4. Logs Are Evidence, Not Automatically the Root Cause

Suppose you see:

```text
connection refused
```

That is evidence.

It does not automatically tell you the root cause.

Possible causes include:

```text
Application not running
Wrong destination port
Application listening on another address
Service crashed
Dependency unavailable
Container not running
```

You still need to investigate.

---

# 5. Major Types of Linux Logs

For beginner administration, think about four broad categories:

```text
Linux Logs
   |
   +-- System logs
   |
   +-- Application/service logs
   |
   +-- Authentication/security logs
   |
   +-- Kernel logs
```

There can be many additional categories, but this gives you a useful starting model.

---

# 6. System Logs

System logs record events related to the operating system and services.

Depending on the distribution and logging configuration, you may encounter files such as:

```text
/var/log/syslog
```

or:

```text
/var/log/messages
```

These can contain information about:

* Services.
* Background processes.
* Scheduled jobs.
* General system events.
* Errors.
* Warnings.

Modern systemd-based distributions also store many system events in the systemd journal.

---

# 7. Application Logs

Applications frequently maintain their own logs.

For example, Nginx commonly uses paths such as:

```text
/var/log/nginx/access.log
```

and:

```text
/var/log/nginx/error.log
```

An application might instead use:

```text
/opt/myapp/logs/application.log
```

or:

```text
/var/log/myapp/application.log
```

or log only to standard output and standard error, which may then be captured by systemd, a container runtime, or another logging system.

There is no universal application log path.

---

# 8. Access Logs vs Error Logs

A web server may separate requests from errors.

An access log might record:

```text
192.0.2.20 - - [27/Aug/2026:10:42:11] "GET /api/orders HTTP/1.1" 200 1842
```

This tells you a request occurred.

An error log might contain:

```text
upstream timed out while reading response header from upstream
```

The two logs answer different questions.

Access logs can help answer:

```text
Who requested something?
When?
Which endpoint?
What HTTP status was returned?
```

Error logs can help answer:

```text
What failed?
Why did the server report an error?
Which dependency or configuration was involved?
```

---

# 9. Authentication and Security Logs

Authentication events are particularly important.

Depending on distribution, you may encounter:

Debian and Ubuntu:

```text
/var/log/auth.log
```

RHEL-family systems commonly use:

```text
/var/log/secure
```

These may contain events involving:

* SSH authentication.
* `sudo`.
* Login failures.
* Successful logins.
* PAM authentication.
* Privilege escalation.

On systemd systems, relevant events may also be available through the journal.

---

# 10. Why Authentication Logs Matter

Suppose someone reports:

```text
I cannot SSH into the server.
```

That statement could describe:

```text
Network timeout
Connection refused
Authentication failure
Wrong username
Wrong SSH key
Bad permissions
SSH server problem
```

Authentication logs can help distinguish between them.

For example:

```text
Failed publickey for deploy from 192.0.2.40
```

tells you the SSH server received the connection and rejected authentication.

That is very different from a client receiving:

```text
Connection timed out
```

before the SSH server ever sees the request.

---

# 11. `/var/log`

A traditional central location for Linux logs is:

```text
/var/log
```

List it:

```bash
ls -l /var/log
```

For human-readable sizes:

```bash
ls -lh /var/log
```

You may see entries such as:

```text
auth.log
syslog
kern.log
messages
secure
nginx/
apache2/
apt/
dnf.log
journal/
```

What exists depends on:

* Distribution.
* Installed software.
* Logging configuration.
* Whether persistent journaling is enabled.

Do not expect every Linux server to have exactly the same files.

---

# 12. Why Logs Are Stored Under `/var`

You learned the Linux filesystem hierarchy earlier.

`/var` generally contains variable data, meaning data that changes while the system operates.

Logs constantly change.

Therefore:

```text
/var
 |
 +-- log
```

is a logical place for them.

---

# 13. Inspecting a Log Directory

Run:

```bash
ls -lh /var/log
```

Suppose you see:

```text
-rw-r----- 1 syslog adm  12M Aug 27 12:10 syslog
-rw-r----- 1 syslog adm 4.2M Aug 27 12:09 auth.log
drwxr-xr-x 2 root   adm 4.0K Aug 27 00:00 nginx
```

You can immediately inspect:

* File names.
* Owners.
* Groups.
* Permissions.
* Sizes.
* Modification times.

This can itself reveal problems.

For example, an application unable to write its log may have a permissions problem.

---

# 14. Reading Logs With `less`

For a large log:

```bash
less /var/log/syslog
```

`less` is usually better than:

```bash
cat /var/log/syslog
```

for a large file.

Why?

Because `less` lets you:

* Scroll.
* Search.
* Move backward.
* Move forward.
* Exit without flooding the terminal.

Press:

```text
q
```

to quit.

---

# 15. Searching Inside `less`

While inside `less`, type:

```text
/error
```

and press Enter.

This searches forward for:

```text
error
```

Press:

```text
n
```

for the next match.

This is useful when examining a large log interactively.

---

# 16. Viewing the End of a Log

Recent events are often near the bottom of traditional append-oriented logs.

Use:

```bash
tail /var/log/syslog
```

By default, `tail` usually shows the last ten lines.

For more:

```bash
tail -n 50 /var/log/syslog
```

This displays the last 50 lines.

---

# 17. Following Logs Live

Use:

```bash
tail -f /var/log/myapp/application.log
```

`-f` means follow.

Instead of exiting after displaying the current end of the file, `tail` waits and displays new lines as they are appended.

Conceptually:

```text
Application
     |
     v
Writes new log line
     |
     v
application.log
     |
     v
tail -f
     |
     v
Your terminal
```

Stop following with:

```text
Ctrl+C
```

---

# 18. Why Live Log Following Is Useful

Suppose an application fails only when a user submits an order.

You can:

1. Open the application log.
2. Follow it live.
3. Reproduce the problem safely.
4. Watch what the application records.

For example:

```bash
tail -f /var/log/myapp/application.log
```

Then reproduce the failure.

You might immediately see:

```text
ERROR database connection refused
```

That gives you a direction.

---

# 19. Be Careful With Busy Logs

A production log may generate thousands of lines per second.

Running:

```bash
tail -f huge-busy-log
```

may produce overwhelming output.

You may need filtering.

For example:

```bash
tail -f application.log | grep ERROR
```

Now only lines containing `ERROR` are displayed.

But remember that case matters unless you specify otherwise.

---

# 20. Searching Logs With `grep`

Suppose:

```text
/var/log/myapp/application.log
```

contains thousands of lines.

Search for errors:

```bash
grep "ERROR" /var/log/myapp/application.log
```

Case-insensitive search:

```bash
grep -i "error" /var/log/myapp/application.log
```

Show line numbers:

```bash
grep -in "error" /var/log/myapp/application.log
```

---

# 21. Searching for Multiple Patterns

You can use extended regular expressions:

```bash
grep -Ei "error|failed|critical" /var/log/myapp/application.log
```

Here:

```text
-E
```

enables extended regular expressions.

```text
-i
```

makes matching case-insensitive.

The pattern:

```text
error|failed|critical
```

means match any of those alternatives.

---

# 22. Do Not Search Only for the Word `error`

Many failures do not literally contain:

```text
ERROR
```

Useful terms might include:

```text
failed
failure
denied
timeout
refused
unreachable
killed
oom
invalid
fatal
critical
panic
reset
```

More importantly, search for the actual:

* Service.
* PID.
* Username.
* IP address.
* Request ID.
* Timestamp.
* File path.
* Error code.

Focused searches are stronger than generic searches.

---

# 23. Timestamps Matter

Suppose users report:

```text
The application failed around 14:32.
```

You discover:

```text
13:05 ERROR ...
13:20 ERROR ...
14:31 ERROR ...
14:32 FATAL ...
14:33 service restarted
16:00 ERROR ...
```

The entries around 14:32 are immediately more relevant.

Troubleshooting without considering time creates noise.

---

# 24. Correlating Events by Time

Imagine:

```text
14:20:01 deployment begins
14:20:35 configuration changed
14:20:41 service restarts
14:20:42 service fails
14:20:45 users receive 502 errors
```

The timeline provides a strong investigative direction.

A useful incident timeline is:

```text
Before
  |
  v
Change
  |
  v
Failure begins
  |
  v
User impact
  |
  v
Investigation
  |
  v
Remediation
  |
  v
Recovery
```

---

# 25. Time Zones Can Cause Confusion

Do not assume every system uses the same timezone.

Check:

```bash
date
```

On systemd systems:

```bash
timedatectl
```

You may have:

```text
Laptop: local time
Server: UTC
Application: UTC
External monitoring: another configured timezone
```

If you ignore this, you may search the wrong period.

---

# 26. What Is the systemd Journal?

On systems using systemd, the journal provides centralized structured logging managed by:

```text
systemd-journald
```

Many events from:

* Services.
* Kernel.
* Boot process.
* System components.

can be queried using:

```bash
journalctl
```

Think:

```text
Services -----\
Kernel --------\
System ---------> Journal
Applications ---/    |
                    v
                journalctl
                    |
                    v
                 Engineer
```

---

# 27. Your First `journalctl` Command

Run:

```bash
journalctl
```

This may display a very large amount of data.

You can navigate similarly to `less`.

Press:

```text
q
```

to exit.

On some systems, permissions may limit which journal entries your regular account can read.

---

# 28. Show Recent Journal Entries

A useful command is:

```bash
journalctl -n 50
```

This displays the most recent 50 entries.

The `-n` option controls the number of recent entries.

You can also use:

```bash
journalctl -n 100
```

---

# 29. Follow the Journal Live

Use:

```bash
journalctl -f
```

This behaves conceptually like:

```bash
tail -f
```

for journal events.

New entries appear as they are recorded.

Stop with:

```text
Ctrl+C
```

---

# 30. Filtering by Service

One of the most useful `journalctl` capabilities is filtering by systemd unit.

Suppose Nginx is failing.

Use:

```bash
journalctl -u nginx
```

Here:

```text
-u
```

means filter by unit.

You are asking:

```text
Show me journal entries associated with the nginx unit.
```

---

# 31. Service Status First

When a service fails, a useful first command is:

```bash
systemctl status nginx
```

This can show:

* Whether the service is active.
* Whether it failed.
* Main PID.
* Exit status.
* Recent journal messages.

Then investigate deeper:

```bash
journalctl -u nginx
```

A practical workflow becomes:

```text
Service failure
      |
      v
systemctl status
      |
      v
journalctl -u
      |
      v
Configuration/application evidence
```

---

# 32. Recent Service Logs

Instead of reading the entire service history:

```bash
journalctl -u nginx -n 50
```

This gives the most recent entries for the Nginx unit.

For live monitoring:

```bash
journalctl -u nginx -f
```

Now you can watch the service's journal entries in real time.

---

# 33. Filtering by Time

Suppose the problem began at 14:00.

You do not need logs from three days ago.

Use:

```bash
journalctl --since "14:00"
```

You can also specify a full date and time:

```bash
journalctl --since "2026-08-27 14:00:00"
```

To stop at a particular time:

```bash
journalctl --since "2026-08-27 14:00:00" --until "2026-08-27 14:30:00"
```

This is extremely useful during incident investigation.

---

# 34. Relative Time Filters

You can often use expressions such as:

```bash
journalctl --since "1 hour ago"
```

or:

```bash
journalctl --since "today"
```

Support for accepted date expressions can depend on the underlying systemd version.

For important investigations, explicit timestamps reduce ambiguity.

---

# 35. Combine Service and Time Filters

You can combine filters.

For example:

```bash
journalctl -u nginx --since "2026-08-27 14:00:00" --until "2026-08-27 14:30:00"
```

This asks:

```text
Service = nginx

AND

Time = 14:00 through 14:30
```

This is much better than searching thousands of unrelated messages.

---

# 36. Boot Logs

You can inspect logs from the current boot:

```bash
journalctl -b
```

The:

```text
-b
```

means boot.

This is useful when investigating:

* Boot failures.
* Services failing during startup.
* Mount failures.
* Driver issues.
* Networking startup.
* Filesystem problems.

---

# 37. Previous Boot

If persistent journal history exists, you can inspect an earlier boot.

List boots:

```bash
journalctl --list-boots
```

You might see entries representing current and previous boots.

Then:

```bash
journalctl -b -1
```

typically means:

```text
Previous boot
```

This is extremely useful when a machine rebooted after an incident.

---

# 38. Why Previous-Boot Logs Matter

Imagine:

```text
03:10 server unhealthy
03:15 server rebooted
03:20 engineer connects
```

If you inspect only the current boot, you may miss the events that caused the problem.

Instead:

```bash
journalctl -b -1
```

can help investigate what happened before the reboot.

This assumes the relevant previous journal data was retained.

---

# 39. Journal Persistence

Journal data may be:

* Persistent across reboots.
* Stored only temporarily.

Persistent journal data is commonly stored under:

```text
/var/log/journal
```

Temporary journal data may be under:

```text
/run/log/journal
```

Exact behavior depends on system configuration.

Therefore, do not assume:

```text
journalctl -b -1
```

will always have previous-boot data.

---

# 40. Filtering by Priority

Logs can have priorities.

Common syslog-style priority levels include:

```text
emerg
alert
crit
err
warning
notice
info
debug
```

You can filter journal messages by priority.

For example:

```bash
journalctl -p err
```

This shows messages at error priority and more severe levels.

---

# 41. Priority Is Helpful but Not Perfect

Do not assume:

```text
journalctl -p err
```

shows every problem.

Applications may:

* Use incorrect severity levels.
* Log important events as warnings.
* Put errors in their own files.
* Fail without producing an `err` priority journal entry.

Priority filtering reduces noise.

It does not replace investigation.

---

# 42. Kernel Logs Through `journalctl`

To inspect kernel messages:

```bash
journalctl -k
```

The:

```text
-k
```

filters kernel messages.

This can help investigate:

* OOM events.
* Storage problems.
* Filesystem errors.
* Network-interface problems.
* Driver problems.
* Hardware-related events.

---

# 43. What Is `dmesg`?

`dmesg` displays messages from the kernel ring buffer.

Run:

```bash
dmesg
```

You may see messages involving:

* Hardware detection.
* Storage devices.
* Network interfaces.
* Drivers.
* Filesystems.
* Memory.
* Kernel warnings.

On some systems, access to kernel messages is restricted to privileged users.

---

# 44. `dmesg` vs `journalctl -k`

Both can expose kernel-related information, but they are not identical interfaces.

Think of them at beginner level as:

```text
dmesg
|
+-- Kernel ring-buffer messages
```

```text
journalctl -k
|
+-- Kernel messages available through the journal
```

The journal can provide richer filtering and retained history depending on configuration.

---

# 45. Human-Readable `dmesg` Timestamps

Depending on the implementation, you may use:

```bash
dmesg -T
```

to display more human-readable timestamps.

However, converted timestamps may have limitations, especially around time changes or suspend/resume behavior.

For incident correlation, journal timestamps are often easier to work with when available.

---

# 46. Searching Kernel Messages

Suppose you suspect an OOM event:

```bash
journalctl -k | grep -Ei "oom|out of memory|killed process"
```

Or:

```bash
dmesg | grep -Ei "oom|out of memory|killed process"
```

Suppose you suspect disk errors:

```bash
journalctl -k | grep -Ei "error|fail|I/O"
```

But generic searches can produce unrelated results.

Use the evidence to narrow the search further.

---

# 47. Combining `grep` and `journalctl`

Because `journalctl` writes text output, you can pipe it to `grep`.

Example:

```bash
journalctl -u nginx | grep -i "error"
```

Flow:

```text
journalctl
    |
    v
Nginx journal entries
    |
    v
grep
    |
    v
Only lines matching "error"
```

---

# 48. Better Time-Bounded Searches

Instead of:

```bash
journalctl -u nginx | grep -i error
```

across an enormous history, use:

```bash
journalctl -u nginx --since "30 minutes ago" | grep -i error
```

Now your search has:

```text
Service scope
+
Time scope
+
Text filter
```

That is much more focused.

---

# 49. Searching for a PID

Suppose:

```text
PID 4812
```

was involved in an incident.

You might search:

```bash
journalctl | grep "4812"
```

But `journalctl` also supports structured fields and can often filter directly using metadata.

For example:

```bash
journalctl _PID=4812
```

This is one advantage of structured journal data.

---

# 50. Searching by Executable

Where appropriate, the journal can be queried using structured fields such as an executable path.

Example:

```bash
journalctl _EXE=/usr/sbin/sshd
```

Structured filtering can be more precise than generic text matching.

The available fields depend on the journal entries.

---

# 51. Viewing Journal Fields

For deeper exploration:

```bash
journalctl -o verbose -n 1
```

You may see metadata such as:

```text
_PID=
_UID=
_GID=
_SYSTEMD_UNIT=
_EXE=
_COMM=
_HOSTNAME=
```

You do not need to memorize all journal fields now.

The important idea is that the journal stores more than plain text.

---

# 52. Output Formats

`journalctl` supports different output formats.

A useful concise format is:

```bash
journalctl -o short-iso
```

This can make timestamps easier to correlate.

For structured processing, JSON output is also available:

```bash
journalctl -o json
```

At beginner level, focus primarily on readable output.

---

# 53. `journalctl --no-pager`

When using journal output inside scripts or pipelines, the pager may be inconvenient.

Use:

```bash
journalctl --no-pager
```

Example:

```bash
journalctl -u nginx -n 50 --no-pager
```

This prints directly to the terminal and exits.

---

# 54. What Is Monitoring?

Logging and monitoring are related, but they are not the same thing.

Logging answers questions such as:

```text
What events happened?
What error was recorded?
What did the application say?
```

Monitoring answers questions such as:

```text
Is the system healthy now?
How has CPU changed over time?
Is disk usage increasing?
Did latency cross a threshold?
```

A simplified distinction:

```text
Logs
=
event records

Metrics
=
numerical measurements over time

Monitoring
=
observing system/application health using evidence
```

---

# 55. Point-in-Time Monitoring

Commands you already know provide immediate observations:

```bash
uptime
free -h
top
vmstat 1
df -h
df -i
ss -lntp
systemctl status
```

These answer:

```text
What does the system look like right now?
```

But an incident may have happened three hours ago.

That is why logs and historical monitoring matter.

---

# 56. Historical Evidence

Imagine:

```text
02:00 CPU = 20%
02:15 CPU = 95%
02:20 application errors begin
02:25 OOM event
02:27 service dies
02:30 CPU = 10%
```

If you log in at:

```text
03:00
```

current CPU may look completely healthy.

Without historical evidence, you could miss the incident.

---

# 57. Logs and Metrics Work Together

Suppose monitoring shows:

```text
14:20
Memory available falls sharply
```

Then logs show:

```text
14:21
Application worker count increased
```

Then:

```text
14:24
Out of memory: Killed process ...
```

Together, the evidence tells a much stronger story.

Think:

```text
Metrics tell you:
Something changed.

Logs tell you:
What events occurred around that change.
```

---

# 58. What Is Log Rotation?

Logs can grow continuously.

Imagine:

```text
Day 1     100 MB
Day 10      1 GB
Day 100    10 GB
```

If nothing manages the logs:

```text
Log grows
   |
   v
Filesystem fills
   |
   v
Applications cannot write
   |
   v
Services may fail
```

Log rotation helps manage this.

---

# 59. What Does Log Rotation Do?

Log rotation can:

* Rename old logs.
* Create or allow creation of new logs.
* Compress older logs.
* Keep a defined number of historical logs.
* Remove sufficiently old logs.
* Rotate based on time or size.

You might see:

```text
application.log
application.log.1
application.log.2.gz
application.log.3.gz
```

The exact naming depends on configuration.

---

# 60. `logrotate`

A common Linux utility for rotating traditional log files is:

```text
logrotate
```

Its configuration is commonly found in:

```text
/etc/logrotate.conf
```

with additional configuration files under:

```text
/etc/logrotate.d/
```

Inspect:

```bash
cat /etc/logrotate.conf
```

and:

```bash
ls -l /etc/logrotate.d/
```

Do not modify production rotation rules until you understand their effects.

---

# 61. Basic `logrotate` Configuration

A simplified configuration might look like:

```text
/var/log/myapp/application.log {
    daily
    rotate 7
    compress
    missingok
    notifempty
}
```

Let's understand it.

---

# 62. `daily`

```text
daily
```

means the log is eligible for rotation daily according to `logrotate`'s execution and state.

This does not mean `logrotate` itself magically runs continuously.

Something such as a systemd timer or scheduled mechanism normally invokes it.

---

# 63. `rotate 7`

```text
rotate 7
```

means retain a configured number of rotated log generations.

This helps limit indefinite accumulation.

---

# 64. `compress`

```text
compress
```

compresses older rotated logs.

You may then see:

```text
application.log.2.gz
```

Compression reduces storage usage.

---

# 65. `missingok`

```text
missingok
```

tells `logrotate` not to treat a missing log file as an error for that rule.

This can be useful when a log may not always exist.

---

# 66. `notifempty`

```text
notifempty
```

means do not rotate an empty log.

There is little benefit in rotating a file containing nothing.

---

# 67. Rotation and Running Applications

There is an important complication.

Suppose:

```text
Application
    |
    v
application.log
```

The application may keep the file open.

Simply renaming the file does not always make the application automatically begin writing to a new file.

Some applications need:

* A signal.
* A reload.
* Their own logging mechanism.
* A post-rotation command.

This is why real `logrotate` configurations may contain directives such as:

```text
postrotate
...
endscript
```

Do not blindly copy rotation configurations between applications.

---

# 68. Why `copytruncate` Requires Understanding

You may encounter:

```text
copytruncate
```

This copies a log and then truncates the original file.

It can help applications that cannot reopen logs easily, but there can be a small window where log data may be lost during the copy/truncate operation.

Therefore:

```text
copytruncate
```

is not automatically the best rotation method.

Understand the application's logging behavior first.

---

# 69. systemd Journal Rotation

The systemd journal manages its own storage and retention mechanisms.

Do not assume:

```text
logrotate
```

manages all journal data.

Traditional file logs and journal storage are related logging concepts but have different management mechanisms.

---

# 70. Troubleshooting With Evidence

Now we combine everything.

The core troubleshooting model is:

```text
SYMPTOM
   |
   v
SCOPE
   |
   v
EVIDENCE
   |
   v
HYPOTHESIS
   |
   v
TEST
   |
   v
ROOT CAUSE
   |
   v
FIX
   |
   v
VERIFY
```

This is more valuable than memorizing hundreds of commands.

---

# 71. Symptoms vs Root Causes

A symptom is what you observe.

A root cause is the underlying reason it happened.

Example:

```text
Symptom:
Website returns 502
```

Possible root cause:

```text
Application service crashed because its configuration referenced a missing environment variable.
```

Another example:

```text
Symptom:
SSH says Permission denied
```

Possible root cause:

```text
Private key does not match the public key configured for that user.
```

Another:

```text
Symptom:
Application cannot write logs
```

Possible root cause:

```text
Log directory is owned by root while application runs as appuser.
```

---

# 72. Why Symptoms Can Mislead You

Suppose:

```text
Website unavailable
```

You might assume:

```text
Nginx is broken.
```

But the real flow could be:

```text
User
 |
 v
Nginx
 |
 v
Application
 |
 v
Database
```

Nginx may be healthy.

The application may be failing because the database is unavailable.

The visible symptom appears at the web layer, but the root cause exists deeper.

---

# 73. Establishing Scope

Before troubleshooting, determine how large the problem is.

Ask:

```text
One user or everyone?

One service or every service?

One server or all servers?

One network path or all traffic?

One filesystem or the entire machine?

One deployment or every version?

Constant or intermittent?

Started when?
```

Scope eliminates possibilities.

---

# 74. Example of Scope

Suppose:

```text
User A cannot SSH.
User B can SSH.
User C can SSH.
```

This immediately suggests a different investigation than:

```text
Nobody can SSH.
```

The first might involve:

* User account.
* User key.
* User-specific configuration.

The second might involve:

* SSH service.
* Network.
* Firewall.
* Server availability.

Scope matters.

---

# 75. Build a Timeline

A timeline is one of the strongest troubleshooting tools.

Write:

```text
13:55 System healthy
14:00 Deployment begins
14:02 Service restarted
14:03 First errors
14:04 User reports begin
14:07 CPU normal
14:08 Application logs show database authentication failures
```

Now the incident becomes easier to reason about.

---

# 76. Build a Hypothesis

A hypothesis is a testable explanation.

Bad hypothesis:

```text
Linux is broken.
```

Better:

```text
The application is failing because it cannot authenticate to the database after the deployment changed its database credentials.
```

Now you can test it.

---

# 77. Test One Hypothesis at a Time

Suppose you think:

```text
Nginx cannot start because port 80 is already occupied.
```

Test:

```bash
sudo ss -lntp
```

Look for:

```text
:80
```

If another process is listening there, your hypothesis gains evidence.

If nothing is using port 80, you should reconsider it.

Do not force evidence to fit your assumption.

---

# 78. Avoid Random Changes

A weak troubleshooting pattern is:

```text
Restart service
Change permissions
Disable firewall
Reboot
Reinstall package
Try random command
```

Eventually something may work, but you will not know why.

Worse, you may introduce new problems.

Prefer:

```text
Observe
→ Hypothesis
→ Test
→ Evidence
→ Controlled change
→ Verify
```

---

# 79. Verify the Fix

Suppose:

```bash
sudo systemctl restart nginx
```

returns successfully.

That proves only that the command completed.

You should still check:

```bash
systemctl status nginx
```

Then:

```bash
curl http://localhost
```

when appropriate.

Then check logs.

Then test from the user's actual path if possible.

A service being:

```text
active (running)
```

does not automatically mean the application works end to end.

---

# 80. Common Incident 1: Permission Denied

Suppose:

```bash
cat /opt/myapp/config.yml
```

returns:

```text
Permission denied
```

Do not immediately run:

```bash
sudo chmod 777 /opt/myapp/config.yml
```

That is not proper troubleshooting.

---

# 81. Investigating Permission Denied

First identify yourself:

```bash
whoami
```

Then:

```bash
id
```

Inspect the target:

```bash
ls -l /opt/myapp/config.yml
```

Also inspect the directories in the path:

```bash
namei -l /opt/myapp/config.yml
```

if `namei` is available.

Remember that directory traversal requires appropriate execute permission.

---

# 82. Permission Hypothesis

Suppose:

```text
-rw------- 1 root root config.yml
```

and you are:

```text
appuser
```

Now the likely explanation is clear.

Root owns the file and only the owner has read/write permission.

The correct fix depends on who should legitimately access the configuration.

Possible fixes may involve:

* Correct ownership.
* Correct group membership.
* Appropriate group permissions.
* ACLs.
* Running the service under the intended account.

Do not use `777` as a universal solution.

---

# 83. Verify a Permission Fix

After making the appropriate controlled change:

```bash
ls -l /opt/myapp/config.yml
```

Then test access as the relevant user.

For services, restart or reload only if necessary and then inspect:

```bash
systemctl status myapp
```

and:

```bash
journalctl -u myapp -n 50
```

---

# 84. Common Incident 2: Service Will Not Start

Suppose:

```bash
sudo systemctl start nginx
```

fails.

Start with:

```bash
systemctl status nginx
```

You might see:

```text
Active: failed
```

and an exit code.

Then:

```bash
journalctl -u nginx -n 100
```

---

# 85. Service Failure Example

Suppose the journal contains:

```text
bind() to 0.0.0.0:80 failed: Address already in use
```

Now you have a strong clue.

Check:

```bash
sudo ss -lntp
```

Find the process using port 80.

The root cause is not:

```text
Nginx does not work.
```

It is:

```text
Another process already occupies the address/port Nginx is configured to use.
```

---

# 86. Configuration Failure Example

Suppose Nginx fails after editing its configuration.

Before restarting repeatedly, validate configuration:

```bash
sudo nginx -t
```

You might see:

```text
syntax is ok
test is successful
```

or an error identifying a file and line.

Many services provide configuration validation commands.

Use them when available.

---

# 87. Common Incident 3: Resource Problem

Users report:

```text
Server is extremely slow.
```

Start with:

```bash
uptime
nproc
free -h
top
```

Then, as necessary:

```bash
vmstat 1
```

and:

```bash
ps aux --sort=-%cpu | head
```

or:

```bash
ps aux --sort=-%mem | head
```

Now combine metrics with logs.

---

# 88. Resource Problem With OOM Evidence

Suppose:

```bash
free -h
```

shows very little available memory.

Then:

```bash
journalctl -k | grep -Ei "oom|out of memory|killed process"
```

returns:

```text
Out of memory: Killed process 7218 (java)
```

Now the service crash has a stronger explanation.

Next investigate why memory demand became excessive.

---

# 89. Common Incident 4: Disk Full

Suppose an application reports:

```text
No space left on device
```

Start:

```bash
df -h
```

Then:

```bash
df -i
```

Why both?

Because:

```text
No space left on device
```

can involve:

* Byte capacity.
* Inode exhaustion.

If capacity is full, identify where space is being consumed.

---

# 90. Finding Large Directories

For example:

```bash
sudo du -xhd1 /var 2>/dev/null | sort -h
```

This means:

```text
du
|
+-- estimate disk usage

-x
|
+-- stay on one filesystem

-h
|
+-- human-readable

-d1
|
+-- show one level deep
```

Then:

```text
2>/dev/null
```

suppresses error output such as inaccessible paths.

Finally:

```text
sort -h
```

sorts human-readable sizes.

Do not blindly delete the largest directory.

Identify what it contains and why it grew.

---

# 91. Logs Can Fill Filesystems

Suppose:

```text
/var
```

is full.

You discover:

```text
/var/log/myapp/application.log
```

is:

```text
80 GB
```

Deleting it immediately may not be the best response.

Ask:

```text
Why did the log become 80 GB?

Is log rotation configured?

Is the application producing an error loop?

Does the application still hold the deleted file open?

Can old logs be safely archived or removed?
```

The oversized log may itself be a symptom.

---

# 92. Deleted Files Can Still Consume Space

Linux allows a process to continue holding an open file even after its directory entry has been deleted.

Therefore:

```bash
rm huge.log
```

does not always immediately recover disk space if a process still has the file open.

A useful investigation command, when available, is:

```bash
sudo lsof +L1
```

This can help identify open files whose link count indicates deletion.

Do not blindly kill processes simply to release files. Understand what owns them first.

---

# 93. Common Incident 5: Network Problem

Suppose:

```bash
curl http://app-server:8080
```

returns:

```text
Connection refused
```

Ask:

```text
Does the hostname resolve?

Can the host be reached?

Is the service running?

Is anything listening on 8080?

Is it listening on the correct address?
```

---

# 94. Network Troubleshooting Sequence

A simple beginner workflow:

```text
Name resolution
     |
     v
Route/reachability
     |
     v
Port
     |
     v
Listening process
     |
     v
Application
```

Commands may include:

```bash
getent hosts app-server
```

```bash
ip route
```

```bash
ping app-server
```

when ICMP testing is meaningful and permitted.

Then:

```bash
ss -lntp
```

and:

```bash
curl -v http://app-server:8080
```

---

# 95. Connection Refused vs Timeout

These are different symptoms.

```text
Connection refused
```

often means the destination was reached but nothing accepted the connection on that address and port, or something actively rejected it.

```text
Connection timed out
```

often points toward:

* Network path issues.
* Firewall filtering.
* Security rules.
* Unreachable destination.
* Packets being dropped.

These are clues, not absolute guarantees.

---

# 96. Logs During Network Troubleshooting

Suppose a client cannot connect to your application.

Follow the application's logs:

```bash
journalctl -u myapp -f
```

Then reproduce the request.

If nothing appears, the request may not be reaching the application.

If you see:

```text
authentication failed
```

then network connectivity may already be working and the problem is at a higher layer.

This is an important troubleshooting distinction.

---

# 97. Troubleshooting by Layers

Think of a request as moving through layers:

```text
Client
  |
  v
DNS
  |
  v
Network route
  |
  v
Server
  |
  v
Listening port
  |
  v
Service
  |
  v
Application
  |
  v
Dependency
```

Test progressively.

Do not jump directly to the database when DNS is broken.

---

# 98. Evidence From Previous Linux Parts

Troubleshooting combines everything you have learned.

Filesystem:

```bash
pwd
ls
find
df
du
```

Permissions:

```bash
ls -l
id
chmod
chown
```

Processes:

```bash
ps
top
pgrep
```

Services:

```bash
systemctl
journalctl
```

Networking:

```bash
ip
ss
curl
dig
```

Resources:

```bash
uptime
free
vmstat
iostat
```

Logs:

```bash
journalctl
grep
tail
less
dmesg
```

The commands are connected.

---

# 99. Troubleshooting Is a Decision Tree

Suppose:

```text
Website unavailable
```

Start:

```text
Can I reach the host?
       |
       +-- No
       |    |
       |    v
       | Network/DNS/routing investigation
       |
       +-- Yes
            |
            v
      Is port listening?
            |
       +----+----+
       |         |
      No        Yes
       |         |
       v         v
 Service      Application
 status       investigation
```

You do not need to memorize every possible branch.

Learn to let evidence choose the next step.

---

# 100. Common Mistake: Reading Thousands of Log Lines

A beginner runs:

```bash
journalctl
```

and starts scrolling.

This is inefficient.

Instead identify:

```text
Which service?
Which server?
Which time?
Which user?
Which PID?
Which error?
```

Then filter.

Example:

```bash
journalctl -u nginx --since "14:00" --until "14:15"
```

Focused investigation beats random scrolling.

---

# 101. Common Mistake: Grepping Only `error`

Suppose the actual message is:

```text
Permission denied
```

Your command:

```bash
grep -i error application.log
```

finds nothing.

That does not prove the application has no problem.

Search based on the symptom and component.

---

# 102. Common Mistake: Ignoring Warnings Before Failure

Logs may show:

```text
10:00 warning
10:05 warning
10:10 warning
10:15 warning
10:20 fatal failure
```

If you inspect only the fatal event, you may miss the buildup.

Look slightly before the incident.

Often the most valuable evidence occurs before the visible failure.

---

# 103. Common Mistake: Looking Only After Restart

Suppose a service fails at:

```text
14:00
```

Someone restarts it at:

```text
14:05
```

You begin investigation at:

```text
14:10
```

Do not inspect only:

```text
14:05 to 14:10
```

Search before the restart:

```bash
journalctl -u myapp --since "13:45" --until "14:10"
```

The cause may appear before the service died.

---

# 104. Common Mistake: Assuming the Last Error Is the Cause

Suppose:

```text
14:01 database authentication failure
14:01 application retries
14:02 worker exits
14:02 socket closed
14:02 connection reset
```

The last line is:

```text
connection reset
```

but that may merely be a downstream consequence.

Read the sequence.

---

# 105. Common Mistake: Changing Multiple Things

Suppose you:

```text
Change permissions
Restart service
Modify firewall
Edit configuration
Reboot server
```

Then it works.

Which change fixed it?

You do not know.

Change one controlled variable at a time whenever practical.

---

# 106. Common Mistake: Ignoring Exit Codes

Commands return exit statuses.

After a command:

```bash
echo $?
```

A common convention is:

```text
0
=
success
```

while non-zero generally indicates some form of failure.

Example:

```bash
grep "ERROR" application.log
echo $?
```

For `grep`, exit codes have command-specific meanings, so always understand the tool rather than assuming every non-zero status means the same thing.

---

# 107. Common Mistake: Using `sudo` to Hide Permission Problems

Suppose:

```bash
cat application.log
```

returns:

```text
Permission denied
```

Then:

```bash
sudo cat application.log
```

works.

You have proved root can read the file.

You have not proved the original user's access should be changed.

Ask:

```text
Should this user actually have access?

What are the security implications?

Which group should read this log?
```

Least privilege still matters during troubleshooting.

---

# 108. Common Mistake: Deleting Logs During an Incident

Logs are evidence.

Do not casually run commands such as:

```bash
rm -rf /var/log/*
```

This can:

* Destroy troubleshooting evidence.
* Break applications.
* Remove security records.
* Violate operational policies.
* Make root-cause analysis harder.

If disk pressure requires emergency action, handle logs carefully and according to system policy.

---

# 109. Common Mistake: Sharing Sensitive Logs

Logs may contain:

* Usernames.
* IP addresses.
* Tokens.
* Session identifiers.
* Internal hostnames.
* URLs.
* Email addresses.
* Application data.
* Occasionally secrets that should never have been logged.

Before posting logs publicly or sharing them externally, review and sanitize sensitive information.

---

# 110. Log-Based Troubleshooting Lab

Use a disposable Linux VM.

The goal is to investigate a real service using logs rather than guessing.

We will use a web server where possible.

You may use Nginx if installed.

If your lab does not have Nginx, use another systemd-managed service and adapt the exercises.

---

# 111. Lab Step 1: Explore `/var/log`

Run:

```bash
ls -lah /var/log
```

Identify:

* System logs.
* Authentication logs.
* Package-manager logs.
* Application directories.
* Journal directory if present.

Write down which logs exist on your distribution.

---

# 112. Lab Step 2: Identify Your Distribution

Run:

```bash
cat /etc/os-release
```

Look for:

```text
NAME=
ID=
VERSION=
```

Then compare your log layout with what you learned.

Do not assume Ubuntu and RHEL-family systems use identical log files.

---

# 113. Lab Step 3: Inspect the Journal

Run:

```bash
journalctl -n 30
```

Identify:

* Timestamp.
* Hostname.
* Process or service.
* Message.

Exit if a pager opens.

---

# 114. Lab Step 4: Inspect Current Boot

Run:

```bash
journalctl -b -n 50
```

Now list available boots:

```bash
journalctl --list-boots
```

If previous boot data exists, inspect:

```bash
journalctl -b -1 -n 50
```

---

# 115. Lab Step 5: Inspect Kernel Messages

Run:

```bash
journalctl -k -n 50
```

Then:

```bash
dmesg | tail -n 50
```

If permission is denied, try the appropriate privileged command on your own lab machine:

```bash
sudo dmesg | tail -n 50
```

Compare the information.

---

# 116. Lab Step 6: Choose a Service

List running services:

```bash
systemctl --type=service --state=running
```

Choose one service you recognize.

For example:

```text
ssh
```

or:

```text
sshd
```

or:

```text
nginx
```

The exact unit name depends on your distribution.

---

# 117. Lab Step 7: Inspect Service Status

Example:

```bash
systemctl status ssh
```

or:

```bash
systemctl status sshd
```

Identify:

```text
Loaded state
Active state
Main PID
Recent messages
```

Then inspect its journal.

Example:

```bash
journalctl -u ssh -n 50
```

Use your actual unit name.

---

# 118. Lab Step 8: Filter by Time

Run something similar to:

```bash
journalctl -u ssh --since "30 minutes ago"
```

Then try:

```bash
journalctl -u ssh --since "today"
```

Observe how filtering reduces irrelevant data.

---

# 119. Lab Step 9: Follow Logs Live

Open:

```bash
journalctl -f
```

In another terminal, perform a harmless action that creates a system event.

Observe new journal entries.

Stop:

```text
Ctrl+C
```

---

# 120. Lab Step 10: Follow a Specific Service

Run:

```bash
journalctl -u ssh -f
```

or another lab service.

Perform an appropriate action involving the service.

Observe whether new entries appear.

This teaches you how to correlate an action with generated logs.

---

# 121. Lab Step 11: Search Authentication Logs

On Ubuntu or Debian, if present:

```bash
sudo less /var/log/auth.log
```

On RHEL-family systems, if present:

```bash
sudo less /var/log/secure
```

Search for:

```text
sudo
```

or:

```text
sshd
```

If those files do not exist, use the journal instead.

---

# 122. Lab Step 12: Search the Journal for SSH

Example:

```bash
journalctl -u ssh --since "today"
```

or:

```bash
journalctl -u sshd --since "today"
```

Search for relevant authentication events.

Do not intentionally expose your SSH service to the internet merely to create failed-login examples.

---

# 123. Lab Step 13: Search With `grep`

Choose a safe log file.

Run:

```bash
grep -i "error" /path/to/log
```

Then:

```bash
grep -Ein "error|failed|warning" /path/to/log
```

Explain:

```text
-E
-i
-n
```

from what you learned earlier.

---

# 124. Lab Step 14: Inspect Log Sizes

Run:

```bash
sudo du -sh /var/log
```

Then:

```bash
sudo du -h --max-depth=1 /var/log 2>/dev/null | sort -h
```

On systems where `du` uses different option syntax, adapt accordingly.

Identify which log directories consume the most space.

Do not delete anything.

---

# 125. Lab Step 15: Explore Log Rotation

Run:

```bash
cat /etc/logrotate.conf
```

Then:

```bash
ls -l /etc/logrotate.d/
```

Open one configuration:

```bash
less /etc/logrotate.d/nginx
```

if present.

Identify directives such as:

```text
daily
weekly
rotate
compress
missingok
notifempty
postrotate
```

---

# 126. Lab Step 16: Inspect Rotated Logs

Look in:

```bash
ls -lh /var/log
```

or an application log directory.

You may find:

```text
syslog
syslog.1
syslog.2.gz
```

or similar files.

Do not assume old information disappeared just because it is not in the current log.

---

# 127. Lab Step 17: Search a Compressed Log

For gzip-compressed logs, a useful command is:

```bash
zgrep -i "error" logfile.gz
```

For interactive reading:

```bash
zless logfile.gz
```

This lets you investigate historical rotated logs without manually decompressing them first.

---

# 128. Lab Step 18: Create a Safe Test Log

Create a lab directory:

```bash
mkdir -p ~/linux-log-lab
```

Create a file:

```bash
touch ~/linux-log-lab/application.log
```

Add entries:

```bash
echo "$(date) INFO Application started" >> ~/linux-log-lab/application.log
```

```bash
echo "$(date) WARNING Disk usage approaching threshold" >> ~/linux-log-lab/application.log
```

```bash
echo "$(date) ERROR Unable to connect to database" >> ~/linux-log-lab/application.log
```

---

# 129. Lab Step 19: Search Your Test Log

Run:

```bash
cat ~/linux-log-lab/application.log
```

Then:

```bash
grep "ERROR" ~/linux-log-lab/application.log
```

Then:

```bash
grep -Ei "warning|error" ~/linux-log-lab/application.log
```

Now you have created and searched your own simple log.

---

# 130. Lab Step 20: Follow Your Test Log

Terminal 1:

```bash
tail -f ~/linux-log-lab/application.log
```

Terminal 2:

```bash
echo "$(date) ERROR Authentication failed" >> ~/linux-log-lab/application.log
```

Return to Terminal 1.

You should see the new entry appear.

Stop:

```text
Ctrl+C
```

---

# 131. Lab Step 21: Filter the Live Log

Run:

```bash
tail -f ~/linux-log-lab/application.log | grep --line-buffered "ERROR"
```

Then append:

```bash
echo "$(date) INFO Health check passed" >> ~/linux-log-lab/application.log
```

Then:

```bash
echo "$(date) ERROR Database unavailable" >> ~/linux-log-lab/application.log
```

Only matching lines should be shown by the filter.

`--line-buffered` is useful here with GNU `grep` so matching output appears promptly in the live pipeline.

---

# 132. Lab Step 22: Build an Incident Timeline

Add:

```bash
echo "$(date) INFO Deployment started" >> ~/linux-log-lab/application.log
sleep 2
echo "$(date) INFO Configuration loaded" >> ~/linux-log-lab/application.log
sleep 2
echo "$(date) ERROR Database authentication failed" >> ~/linux-log-lab/application.log
sleep 2
echo "$(date) CRITICAL Application startup aborted" >> ~/linux-log-lab/application.log
```

Read:

```bash
cat ~/linux-log-lab/application.log
```

Now reconstruct:

```text
Deployment
→ Configuration loaded
→ Authentication failure
→ Startup aborted
```

This is basic timeline analysis.

---

# 133. Lab Step 23: Identify Symptom and Root Cause

From the previous example:

Symptom:

```text
Application startup aborted
```

Immediate error:

```text
Database authentication failed
```

Potential root cause:

```text
Incorrect database credentials or authentication configuration.
```

Notice that:

```text
Application stopped
```

and:

```text
Wrong credentials
```

are not the same level of explanation.

---

# 134. Lab Step 24: Build a Hypothesis

Write:

```text
Hypothesis:

The application failed to start because the database credentials loaded during deployment are invalid.
```

Then write what evidence you would need:

```text
Check application configuration source.

Check whether credentials changed.

Test database authentication safely.

Compare with previous known-working configuration.

Check database-side authentication logs if available.
```

This is how engineers move from logs to investigation.

---

# 135. Lab Step 25: Practice a Permission Incident

Create:

```bash
mkdir -p ~/linux-log-lab/private
touch ~/linux-log-lab/private/secret.log
chmod 600 ~/linux-log-lab/private/secret.log
```

Inspect:

```bash
ls -l ~/linux-log-lab/private/secret.log
```

Explain who can read and write it.

Do not use `chmod 777` merely to make access easier.

---

# 136. Lab Step 26: Practice a Service Investigation

Choose a harmless service on your disposable VM.

Run:

```bash
systemctl status SERVICE
```

Then:

```bash
journalctl -u SERVICE -n 50
```

Then:

```bash
journalctl -u SERVICE --since "30 minutes ago"
```

Then:

```bash
journalctl -u SERVICE -f
```

Stop the live view with:

```text
Ctrl+C
```

You have now practiced the basic service-log workflow.

---

# 137. Lab Step 27: Build a System Evidence Snapshot

Run:

```bash
echo "=== DATE ==="
date

echo "=== HOST ==="
hostname

echo "=== UPTIME AND LOAD ==="
uptime

echo "=== CPU COUNT ==="
nproc

echo "=== MEMORY ==="
free -h

echo "=== DISK ==="
df -h

echo "=== INODES ==="
df -i

echo "=== TOP CPU PROCESSES ==="
ps aux --sort=-%cpu | head

echo "=== TOP MEMORY PROCESSES ==="
ps aux --sort=-%mem | head

echo "=== FAILED SERVICES ==="
systemctl --failed --no-pager
```

Understand every command.

This gives you a basic point-in-time troubleshooting snapshot.

---

# 138. Lab Step 28: Add Kernel Evidence

Run:

```bash
journalctl -k -n 50 --no-pager
```

Look for obvious:

```text
errors
warnings
OOM events
storage problems
network-device problems
```

Do not assume every warning is related to your incident.

Correlation matters.

---

# 139. Lab Step 29: Build a Slow-Server Investigation

Pretend someone reports:

```text
The server is slow.
```

Collect:

```bash
date
uptime
nproc
free -h
vmstat 1
df -h
df -i
ps aux --sort=-%cpu | head
ps aux --sort=-%mem | head
systemctl --failed
journalctl -p warning --since "30 minutes ago"
```

Stop `vmstat` after several samples.

Then answer:

```text
What is actually abnormal?

What is normal?

What evidence supports that conclusion?

What should I inspect next?
```

---

# 140. Lab Step 30: Build a Service-Failure Investigation

Pretend:

```text
myapp is down.
```

Your investigation should resemble:

```text
1. Confirm symptom.
2. Check service status.
3. Check recent service logs.
4. Check time of failure.
5. Check configuration.
6. Check dependencies.
7. Check ports.
8. Check resources.
9. Check recent changes.
10. Form hypothesis.
11. Test.
12. Apply controlled fix.
13. Verify service.
14. Verify application.
15. Verify user path.
```

The important skill is not merely knowing the commands.

It is knowing why you run each command.

---

# 141. Linux Incident Investigation Template

For future troubleshooting, use this structure.

```text
INCIDENT

Reported symptom:
What is the user or monitoring system reporting?

Start time:
When did it begin?

Scope:
Who or what is affected?

Recent changes:
What changed before the incident?

Current state:
What is happening now?

Evidence:
What do commands, metrics, and logs show?

Hypothesis:
What explanation best fits the evidence?

Test:
How can the hypothesis be tested safely?

Root cause:
What actually caused the incident?

Remediation:
What was changed?

Verification:
How was recovery confirmed?

Prevention:
What should prevent or detect recurrence?
```

---

# 142. Example Incident Report

```text
Reported symptom:
Users received HTTP 502 responses.

Start time:
14:03 UTC.

Scope:
All requests to one application instance.

Recent change:
Application deployment at 14:00 UTC.

Evidence:
Nginx remained active.
Application service was failed.
Application journal showed database authentication failures.
Database host was reachable.
Application credentials changed during deployment.

Hypothesis:
The deployment introduced invalid database credentials.

Test:
Credentials were compared with the approved secret source and safely tested.

Root cause:
Incorrect database password was deployed.

Remediation:
Correct secret was restored and application service restarted.

Verification:
Service remained active.
Local health endpoint returned 200.
External requests succeeded.
Application logs showed successful database connections.
Error rate returned to normal.

Prevention:
Add deployment-time secret validation and post-deployment health checks.
```

This is much stronger than:

```text
App went down. Restarted it. Fixed.
```

---

# 143. Troubleshooting Mental Model

When you encounter an error:

```text
Do not ask first:

"What command fixes this?"
```

Ask:

```text
"What does this symptom tell me?"
```

Then:

```text
What component produced it?
What evidence exists?
What happened before it?
What is the scope?
What hypotheses fit?
How can I eliminate possibilities?
```

This is how troubleshooting becomes a skill rather than command memorization.

---

# 144. What You Should Now Understand

Before continuing to Part 17, you should confidently understand:

* Why Linux systems generate logs.
* What a log is.
* Why logs are evidence.
* Why a log message is not automatically the root cause.
* What system logs are.
* What application logs are.
* What authentication and security logs are.
* What kernel logs are.
* What `/var/log` is used for.
* Why different distributions have different log files.
* Why applications can have different logging locations.
* The difference between access logs and error logs.
* Why authentication logs matter.
* How to inspect `/var/log`.
* How to read logs with `less`.
* How to search within `less`.
* How to use `tail`.
* How to use `tail -f`.
* Why live log following is useful.
* How to search logs with `grep`.
* Why searching only for `error` is insufficient.
* Why timestamps matter.
* How timestamps help create an incident timeline.
* Why timezones matter.
* What the systemd journal is.
* What `systemd-journald` does at a beginner level.
* What `journalctl` does.
* How to display recent journal entries.
* How to follow the journal live.
* How to filter by systemd service.
* How `systemctl status` and `journalctl` work together.
* How to filter journal entries by time.
* How to combine service and time filters.
* How to inspect current-boot logs.
* How to list available boots.
* How to inspect previous-boot logs.
* Why previous-boot logs may be essential.
* Why previous journal history may not always exist.
* What journal priorities are.
* How to filter by priority.
* Why priority filtering alone is insufficient.
* How to inspect kernel logs through `journalctl`.
* What `dmesg` does.
* The basic difference between `dmesg` and `journalctl -k`.
* How to search for OOM events.
* How to combine `journalctl` and `grep`.
* Why time-bounded searches are better.
* How structured journal fields can improve filtering.
* Why logs and metrics are different.
* What monitoring means.
* Why point-in-time information can miss earlier incidents.
* Why historical monitoring matters.
* How logs and metrics complement one another.
* What log rotation is.
* Why logs need rotation.
* What `logrotate` is.
* Where common `logrotate` configuration lives.
* What `daily` means.
* What `rotate` means.
* What `compress` means.
* What `missingok` means.
* What `notifempty` means.
* Why applications may need to reopen logs after rotation.
* Why `copytruncate` requires care.
* Why journal retention is different from traditional file rotation.
* What a symptom is.
* What a root cause is.
* Why symptoms can be misleading.
* How to establish incident scope.
* How to create a timeline.
* What a troubleshooting hypothesis is.
* How to test a hypothesis.
* Why random changes are dangerous.
* Why fixes must be verified.
* How to investigate permission problems.
* How to investigate service failures.
* How to investigate resource problems.
* How to investigate disk-full incidents.
* Why inode exhaustion matters.
* Why oversized logs can fill a filesystem.
* Why deleting an open log may not immediately release disk space.
* How network errors provide clues.
* The difference between connection refused and timeout.
* How logs help during network troubleshooting.
* Why troubleshooting should proceed through layers.
* Why previous Linux commands work together during incidents.
* Why reading thousands of unfiltered logs is inefficient.
* Why warnings before an incident matter.
* Why you should inspect events before a restart.
* Why the final error message may not be the root cause.
* Why changing multiple things at once makes diagnosis difficult.
* Why logs should not be casually deleted.
* Why sensitive log information must be protected.
* How to document an incident clearly.

---

# 145. Hands-On Practice

Complete these tasks on a disposable Linux environment.

1. Identify your Linux distribution.
2. List `/var/log`.
3. Identify available system logs.
4. Identify authentication logs if present.
5. Identify application log directories.
6. Check whether `/var/log/journal` exists.
7. Open a log using `less`.
8. Search inside `less`.
9. Display the last 20 lines of a log.
10. Display the last 100 lines.
11. Follow a safe log using `tail -f`.
12. Stop following it.
13. Search a log using `grep`.
14. Perform a case-insensitive search.
15. Search for multiple patterns.
16. Display matching line numbers.
17. Identify timestamps in a log.
18. Check the server's current time.
19. Check timezone information.
20. Run `journalctl`.
21. Display the latest 50 journal entries.
22. Follow the journal live.
23. Stop the live journal.
24. Select a systemd service.
25. Check its status.
26. Display its journal.
27. Display its latest 50 entries.
28. Filter its logs to the last 30 minutes.
29. Follow its logs live.
30. Inspect current-boot logs.
31. List available boots.
32. Inspect a previous boot if available.
33. Explain why previous-boot data may be unavailable.
34. Filter journal messages by error priority.
35. Filter by warning priority.
36. Inspect kernel journal messages.
37. Run `dmesg`.
38. Compare `dmesg` with `journalctl -k`.
39. Search for OOM messages.
40. Search for storage-related kernel errors.
41. Combine `journalctl` with `grep`.
42. Search logs within a defined time window.
43. Inspect journal metadata using verbose output.
44. Identify a PID field.
45. Identify a systemd unit field.
46. Explain why structured journal metadata is useful.
47. Inspect `/etc/logrotate.conf`.
48. List `/etc/logrotate.d`.
49. Open one rotation configuration.
50. Identify its rotation frequency.
51. Identify how many rotations it retains.
52. Identify whether it compresses logs.
53. Find a rotated log.
54. Find a compressed rotated log if one exists.
55. Search a compressed log using `zgrep`.
56. Read a compressed log using `zless`.
57. Create `~/linux-log-lab`.
58. Create `application.log`.
59. Add an INFO message.
60. Add a WARNING message.
61. Add an ERROR message.
62. Search only ERROR entries.
63. Search WARNING and ERROR entries together.
64. Follow the test log live.
65. Add another message from a second terminal.
66. Observe it appear.
67. Create a simple incident timeline.
68. Identify the symptom.
69. Identify the likely underlying cause.
70. Write a testable hypothesis.
71. Explain how you would test it.
72. Inspect current CPU/load evidence.
73. Inspect memory.
74. Inspect disk capacity.
75. Inspect inode usage.
76. Inspect top CPU processes.
77. Inspect top memory processes.
78. Check failed services.
79. Inspect recent warning/error journal entries.
80. Build a basic system evidence snapshot.
81. Investigate one harmless service.
82. Identify its unit name.
83. Identify its PID.
84. Find its recent logs.
85. Identify when it last started.
86. Identify whether it generated warnings.
87. Create a hypothetical permission incident.
88. Explain how you would investigate it.
89. Create a hypothetical service-start failure.
90. Explain how you would investigate it.
91. Create a hypothetical disk-full incident.
92. Explain how you would investigate it.
93. Create a hypothetical connection-refused incident.
94. Explain how you would investigate it.
95. Explain the difference between a symptom and root cause.
96. Explain why a timeline is useful.
97. Explain why you should avoid random fixes.
98. Explain why a successful restart does not prove the root cause is fixed.
99. Write a complete incident investigation template.
100. Explain why evidence-based troubleshooting is more reliable than command guessing.

---

# 146. Beginner Knowledge Check

1. Why do Linux systems generate logs?
2. What is a log?
3. Are logs automatically proof of root cause?
4. What are system logs?
5. What are application logs?
6. What are authentication logs?
7. What are kernel logs?
8. What is `/var/log`?
9. Do all Linux distributions have identical log files?
10. Where might Ubuntu authentication events traditionally appear?
11. Where might RHEL-family authentication events traditionally appear?
12. What is an access log?
13. What is an error log?
14. Why are timestamps important?
15. Why should you check the server's timezone?
16. What does `less` help you do?
17. What does `tail` do?
18. What does `tail -f` do?
19. How do you stop `tail -f`?
20. What does `grep` do?
21. What does `grep -i` do?
22. Why should you not search only for the word `error`?
23. What is the systemd journal?
24. What program collects journal events?
25. What command queries the journal?
26. What does `journalctl -n 50` do?
27. What does `journalctl -f` do?
28. What does `journalctl -u nginx` do?
29. What does `journalctl -b` do?
30. What does `journalctl --list-boots` do?
31. What does `journalctl -b -1` commonly do?
32. Why might previous-boot logs be unavailable?
33. How can you filter journal logs by time?
34. How can you filter journal logs by service and time?
35. What does `journalctl -p err` do?
36. Why should you not rely only on priority filtering?
37. What does `journalctl -k` show?
38. What does `dmesg` show?
39. What is the basic difference between `dmesg` and `journalctl -k`?
40. How could you search kernel logs for OOM events?
41. Why combine `journalctl` with `grep`?
42. Why are structured journal fields useful?
43. What is monitoring?
44. What is the basic difference between logs and metrics?
45. Why can current system metrics miss an earlier incident?
46. Why should logs and metrics be correlated?
47. What is log rotation?
48. Why is log rotation necessary?
49. What is `logrotate`?
50. Where is its main configuration commonly stored?
51. What is `/etc/logrotate.d`?
52. What does `daily` mean?
53. What does `rotate 7` mean?
54. What does `compress` do?
55. What does `missingok` do?
56. What does `notifempty` do?
57. Why can rotation be complicated for running applications?
58. What is `copytruncate`?
59. Why should `copytruncate` be used with understanding?
60. Does `logrotate` necessarily manage the systemd journal?
61. What is a symptom?
62. What is a root cause?
63. Why is `website unavailable` not a root cause?
64. What does establishing scope mean?
65. Why does scope matter?
66. What is an incident timeline?
67. What is a troubleshooting hypothesis?
68. What makes a hypothesis useful?
69. Why should you test hypotheses?
70. Why should you avoid random changes?
71. Why should you change one thing at a time when practical?
72. Why must fixes be verified?
73. Does `systemctl restart` succeeding prove the application works?
74. How would you begin investigating `Permission denied`?
75. Why is `chmod 777` usually a poor troubleshooting response?
76. How would you begin investigating a failed service?
77. What does `systemctl status` tell you?
78. Why should you inspect `journalctl -u SERVICE`?
79. How would you check which process owns a listening port?
80. How would you begin investigating resource pressure?
81. How can kernel logs help during memory incidents?
82. What does `No space left on device` potentially indicate?
83. Why should you run both `df -h` and `df -i`?
84. Why can logs cause a disk-full incident?
85. Why might deleting a large open log not immediately free space?
86. What can `lsof +L1` help identify?
87. What does `Connection refused` generally suggest?
88. What does a timeout generally suggest?
89. Why are these clues rather than absolute diagnoses?
90. Why should network troubleshooting proceed through layers?
91. Why should you inspect logs from before a restart?
92. Why might the last error not be the root cause?
93. Why should you avoid deleting logs during an incident?
94. Why can logs contain sensitive information?
95. Why should sensitive logs be sanitized before sharing?
96. What should an incident report contain?
97. What should happen after remediation?
98. What does prevention mean in troubleshooting?
99. Why is knowing commands alone insufficient?
100. What is the central principle of evidence-based Linux troubleshooting?

---

# 147. Beginner Knowledge Check Answers

1. To record events that help administrators understand system and application activity.

2. A record of an event produced by a system, service, application, or component.

3. No. They provide evidence that must be interpreted.

4. Logs describing operating-system and service-related events.

5. Logs generated by applications describing their requests, behavior, warnings, errors, and other events.

6. Logs related to authentication, login, SSH, `sudo`, and similar security activity.

7. Messages generated by the Linux kernel.

8. A common directory containing traditional Linux log files and log directories.

9. No.

10. Commonly `/var/log/auth.log`, depending on configuration.

11. Commonly `/var/log/secure`, depending on configuration.

12. A log recording requests handled by a service such as a web server.

13. A log recording warnings, failures, and other error-related events.

14. They let you correlate events with the time an incident occurred.

15. Different systems may record or display time differently, which can lead you to investigate the wrong period.

16. Read and search large text files interactively.

17. Displays the end of a file.

18. Continues displaying new lines as they are appended.

19. `Ctrl+C`.

20. Searches text for matching patterns.

21. Performs case-insensitive matching.

22. Failures may use words such as denied, failed, timeout, refused, killed, or application-specific messages.

23. systemd's centralized structured logging facility.

24. `systemd-journald`.

25. `journalctl`.

26. Displays the latest 50 journal entries.

27. Follows new journal entries live.

28. Filters journal entries for the Nginx systemd unit.

29. Shows messages associated with a boot, by default the current boot.

30. Lists boots known to the journal.

31. Shows messages from the previous boot when that history exists.

32. The journal may not be configured for persistent storage, or older data may already have been removed.

33. With options such as `--since` and `--until`.

34. Combine `-u` with `--since` and optionally `--until`.

35. Shows error-priority messages and more severe priorities.

36. Important failures may use other priorities or may exist outside the journal.

37. Kernel messages available through the journal.

38. Messages from the kernel ring buffer.

39. `dmesg` reads kernel ring-buffer messages, while `journalctl -k` queries kernel messages stored or available through the journal.

40. For example, `journalctl -k | grep -Ei "oom|out of memory|killed process"`.

41. To apply additional text filtering to journal output.

42. They allow filtering using metadata such as PID, executable, or systemd unit.

43. Observing the health and behavior of systems and applications over time or at a point in time.

44. Logs primarily record events, while metrics primarily represent numerical measurements.

45. The condition may have ended before you connected.

46. Metrics can reveal when behavior changed while logs can explain events occurring around the change.

47. The management of growing logs by rotating, retaining, compressing, and eventually removing older files.

48. Otherwise logs can grow indefinitely and consume storage.

49. A common Linux log-rotation utility.

50. `/etc/logrotate.conf`.

51. A directory commonly containing application-specific rotation rules.

52. The log is eligible for daily rotation according to the rotation process.

53. Retain the configured number of rotated generations.

54. Compresses rotated logs.

55. Avoids treating a missing log as an error for that rule.

56. Prevents empty logs from being rotated.

57. A process may continue writing through an already-open file descriptor and may need to reopen its log.

58. A rotation method that copies the current log and then truncates the original.

59. There can be a window where some log data is lost, and it may not be the best method for the application.

60. No. The journal has its own retention mechanisms.

61. The visible behavior indicating something is wrong.

62. The underlying reason the incident occurred.

63. It describes what users observe, not why the website became unavailable.

64. Determining which users, services, servers, components, locations, or time periods are affected.

65. It eliminates irrelevant possibilities and guides the investigation.

66. An ordered record of important events before, during, and after an incident.

67. A testable explanation for the observed evidence.

68. It should explain the evidence and be possible to test.

69. To determine whether the proposed explanation is supported by evidence.

70. They can introduce new problems and make it impossible to know what actually fixed the incident.

71. It makes cause and effect easier to determine.

72. A command completing does not prove the original user-visible problem has disappeared.

73. No.

74. Identify the user, inspect file and directory ownership and permissions, and determine what access should legitimately exist.

75. It grants excessive access and may create a security problem without addressing the intended ownership model.

76. Check `systemctl status SERVICE`, then inspect its logs.

77. Whether the service is loaded, active, failed, its PID, exit information, and recent messages.

78. It provides service-specific evidence around the failure.

79. For example, with `ss -lntp`, using appropriate privileges when needed.

80. Inspect load, CPU, memory, swap, processes, disk capacity, and relevant historical evidence.

81. They can reveal OOM events and other kernel-level memory problems.

82. Filesystem byte capacity exhaustion or inode exhaustion, among other storage-related conditions.

83. One checks filesystem space while the other checks inode availability.

84. Uncontrolled log growth can consume the filesystem.

85. A running process may still hold the deleted file open.

86. Open files that have been unlinked and may still consume space.

87. The destination was commonly reachable but nothing accepted the connection on that address and port, or the connection was actively rejected.

88. Packets may be dropped or the destination/path may be unreachable.

89. Different network and application configurations can produce similar symptoms.

90. It lets you determine where communication stops instead of guessing.

91. The evidence explaining the failure may have occurred before the restart.

92. Later errors can be consequences of an earlier failure.

93. Logs are important operational and forensic evidence, and deleting them can also affect applications or compliance requirements.

94. They may contain usernames, IP addresses, internal systems, tokens, identifiers, and application data.

95. To prevent accidental exposure of sensitive information.

96. Symptom, time, scope, recent changes, evidence, hypothesis, root cause, remediation, verification, and prevention.

97. Verify the system and the original user-visible functionality have recovered.

98. Changes that reduce the probability of recurrence or improve detection if it happens again.

99. Troubleshooting requires reasoning about systems, evidence, dependencies, and cause and effect.

100. Observe the symptom, establish scope, collect evidence, form and test hypotheses, identify the cause, make a controlled fix, and verify the result.

# 148. Final Linux Troubleshooting Mental Model

At this point, you should no longer think of Linux troubleshooting as:

```text
Problem
   |
   v
Search internet
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

Your model should be:

```text
                  INCIDENT
                     |
                     v
                  SYMPTOM
                     |
                     v
                   SCOPE
                     |
                     v
                  TIMELINE
                     |
                     v
              COLLECT EVIDENCE
                     |
       +-------------+-------------+
       |             |             |
       v             v             v
     Logs          Metrics       State
       |             |             |
       +-------------+-------------+
                     |
                     v
                 HYPOTHESIS
                     |
                     v
                    TEST
                     |
              +------+------+
              |             |
              v             v
          Supported      Rejected
              |             |
              v             |
          ROOT CAUSE        |
              |             |
              v             |
         REMEDIATION        |
              |             |
              v             |
          VERIFICATION      |
              |             |
              v             |
          PREVENTION        |
                            |
                            +----> New hypothesis
```

For a service problem:

```text
Service failed
     |
     v
systemctl status
     |
     v
journalctl -u SERVICE
     |
     v
When did it fail?
     |
     v
What happened immediately before?
     |
     v
Configuration?
Permissions?
Port?
Dependency?
Resources?
     |
     v
Test hypothesis
```

For a performance problem:

```text
Server slow
    |
    v
uptime
    |
    v
CPU + load
    |
    v
free -h
    |
    v
vmstat / top
    |
    v
Processes
    |
    v
Disk / network
    |
    v
Logs around incident
```

For a disk problem:

```text
No space left
     |
     v
df -h
     |
     +------ Space exhausted?
     |
     v
df -i
     |
     +------ Inodes exhausted?
     |
     v
du
     |
     v
What grew?
     |
     v
Why did it grow?
```

For a network problem:

```text
Connection failed
      |
      v
DNS resolution
      |
      v
Network path
      |
      v
Destination host
      |
      v
Listening port
      |
      v
Service
      |
      v
Application logs
      |
      v
Dependencies
```

For a permission problem:

```text
Permission denied
      |
      v
Who am I?
      |
      v
id
      |
      v
Who owns the resource?
      |
      v
ls -l
      |
      v
What permissions exist?
      |
      v
Can directories be traversed?
      |
      v
What access SHOULD exist?
      |
      v
Apply least-privilege correction
      |
      v
Verify
```

The central habit is simple:

```text
Do not ask:

"What command can make this error disappear?"

Ask:

"What evidence explains why this happened?"
```

That shift is one of the foundations of becoming competent at Linux administration, DevOps, SRE, Cloud, and Platform Engineering.
