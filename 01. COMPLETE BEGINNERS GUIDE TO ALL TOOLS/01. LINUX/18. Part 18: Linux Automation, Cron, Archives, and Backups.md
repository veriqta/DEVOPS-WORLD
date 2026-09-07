# Part 18: Linux Automation, Cron, Archives, and Backups

In Part 17, you learned how to turn Linux commands into Bash scripts. You learned variables, conditions, loops, functions, exit codes, basic error handling, and debugging.

Now we will answer the next question:

How do you make Linux run those tasks automatically?

A script becomes much more useful when the system can run it without waiting for you to log in and start it manually.

In this part, you will learn scheduled automation with cron, archive and compression tools such as `tar` and `gzip`, backup fundamentals, backup verification, basic retention, and how to build and schedule a complete automated backup script.

By the end, you should understand this workflow:

```text
Files and directories
        |
        v
Backup script
        |
        v
Create archive
        |
        v
Compress archive
        |
        v
Verify backup
        |
        v
Store backup
        |
        v
Apply retention
        ^
        |
       Cron
Runs automatically
```

---

# 1. Why System Administration Is Automated

Imagine that every night you need to:

1. Back up an application's configuration.
2. Back up important application files.
3. Compress the backup.
4. Save it with the current date.
5. Verify that the backup exists.
6. Record whether the operation succeeded.
7. Remove backups older than your retention period.

You could log in every night and perform these tasks manually.

But that creates problems.

You may:

* Forget.
* Run the task late.
* Miss a step.
* Type the wrong command.
* Use the wrong directory.
* Forget to verify the backup.
* Forget to clean up old backups.

Automation makes repeatable administration possible.

Instead of:

```text
Administrator
     |
     v
Remember task
     |
     v
Log in
     |
     v
Run commands
```

you can build:

```text
Scheduler
    |
    v
Run script
    |
    v
Perform task
    |
    v
Record result
```

This is one of the foundations of Linux system administration.

---

# 2. What Should Be Automated?

Common Linux automation tasks include:

* Backups.
* Log cleanup.
* Temporary-file cleanup.
* Health checks.
* Disk-space checks.
* Report generation.
* Database maintenance.
* Certificate checks.
* File synchronization.
* Application maintenance.
* Monitoring scripts.
* Security scans.
* Scheduled downloads.
* Data processing.
* Log rotation.
* Scheduled application jobs.

But automation should not mean:

```text
Run dangerous commands automatically without validation.
```

Automation should mean:

```text
Understand task
      |
      v
Test manually
      |
      v
Script safely
      |
      v
Validate script
      |
      v
Schedule
      |
      v
Monitor result
```

Automating a bad procedure simply makes the bad procedure happen more consistently.

---

# 3. Manual Execution vs Scheduled Execution

Suppose you created:

```text
backup.sh
```

You can manually execute:

```bash
./backup.sh
```

That is automation of the procedure, but not automation of the schedule.

You still have to initiate it.

A scheduler allows Linux to execute it at a particular time.

For example:

```text
Every day at 2:00 AM
        |
        v
Run /home/user/scripts/backup.sh
```

One traditional Linux scheduler used for this is cron.

---

# 4. What Is Cron?

Cron is a time-based job scheduling system traditionally used on Unix and Linux systems.

It allows commands and scripts to run automatically according to schedules.

For example:

```text
Every day at 02:00
        |
        v
Run backup script
```

Or:

```text
Every Monday at 08:00
        |
        v
Generate report
```

Or:

```text
Every 15 minutes
        |
        v
Run monitoring check
```

The scheduled command is commonly called a cron job.

---

# 5. What Is the Cron Daemon?

A daemon is a program that runs in the background and provides a service.

You learned about daemons and services in Part 9.

The cron daemon is responsible for processing cron schedules and launching jobs at the appropriate times.

Depending on the Linux distribution, the service may be named:

```text
cron
```

or:

```text
crond
```

For example, on Ubuntu or Debian systems you may check:

```bash
systemctl status cron
```

On many Red Hat-family systems:

```bash
systemctl status crond
```

Do not assume every distribution uses the same service name.

---

# 6. Check Whether Cron Is Installed

First:

```bash
command -v crontab
```

Possible output:

```text
/usr/bin/crontab
```

This tells you that the `crontab` command can be found.

You can then inspect the service.

Ubuntu/Debian:

```bash
systemctl status cron
```

RHEL-family systems may use:

```bash
systemctl status crond
```

Look for:

```text
active (running)
```

If the service does not exist, cron may not be installed, or the system may use another scheduling mechanism.

---

# 7. What Is `crontab`?

`crontab` means cron table.

It contains scheduled jobs associated with a user.

A simplified example:

```text
0 2 * * * /home/ann/scripts/backup.sh
```

This tells cron to run the specified script every day at 2:00 AM.

Do not worry about the five fields yet.

We will build the syntax from zero.

---

# 8. User Crontabs

Linux can maintain cron jobs for different users.

For example:

```text
ann
 |
 +--> ann's scheduled jobs

root
 |
 +--> root's scheduled jobs
```

This matters because a cron job runs with the permissions of the user whose crontab launches it.

If your normal user cannot read:

```text
/root
```

a cron job running as that user does not magically gain root access.

The principle of least privilege still applies.

---

# 9. Viewing Your Cron Jobs

Run:

```bash
crontab -l
```

The:

```text
-l
```

means list.

If you have no crontab, you may receive a message indicating that no crontab exists for your user.

That is not necessarily an error.

It simply means you have not created scheduled jobs for that user yet.

---

# 10. Editing Your Cron Jobs

Run:

```bash
crontab -e
```

The:

```text
-e
```

means edit.

The first time, your system may ask which editor you want to use.

For beginners, Nano is usually easier than Vim.

After saving, cron installs the updated crontab for that user.

---

# 11. Do Not Edit Random Cron Files

Beginners sometimes find files under:

```text
/etc/cron.d
/etc/crontab
/etc/cron.daily
/etc/cron.hourly
```

and start editing them.

Those are system-level scheduling facilities and may use conventions that differ from a user's `crontab -e`.

For this beginner lesson, start with:

```bash
crontab -e
```

and understand user crontabs first.

---

# 12. Cron Syntax

A standard user cron entry contains five scheduling fields followed by a command:

```text
* * * * * command
| | | | |
| | | | +---- Day of week
| | | +------ Month
| | +-------- Day of month
| +---------- Hour
+------------ Minute
```

The five fields are:

```text
Minute
Hour
Day of month
Month
Day of week
```

Then comes:

```text
Command
```

---

# 13. Minute Field

The first field represents the minute.

Typical range:

```text
0-59
```

Example:

```text
30 * * * * command
```

means:

```text
At minute 30 of every hour.
```

So it could run at:

```text
01:30
02:30
03:30
04:30
...
```

---

# 14. Hour Field

The second field represents the hour using a 24-hour clock.

Typical range:

```text
0-23
```

Example:

```text
0 2 * * * command
```

means:

```text
At minute 0
At hour 2
Every day
```

Therefore:

```text
02:00 every day
```

---

# 15. Day-of-Month Field

The third field represents the day of the month.

Typical range:

```text
1-31
```

Example:

```text
0 2 1 * * command
```

means:

```text
At 02:00
On day 1 of the month
Every month
```

---

# 16. Month Field

The fourth field represents the month.

Typical numeric range:

```text
1-12
```

Example:

```text
0 2 1 1 * command
```

means:

```text
At 02:00
On January 1
```

Some cron implementations also support month names, but numeric fields are useful for learning the basic structure.

---

# 17. Day-of-Week Field

The fifth field represents the weekday.

Common numbering uses:

```text
0 = Sunday
1 = Monday
2 = Tuesday
3 = Wednesday
4 = Thursday
5 = Friday
6 = Saturday
```

Some implementations also accept `7` for Sunday.

Example:

```text
0 8 * * 1 command
```

means:

```text
Monday at 08:00
```

---

# 18. What Does `*` Mean?

An asterisk means:

```text
Every allowed value for this field
```

For example:

```text
0 2 * * *
```

means:

```text
Minute:       0
Hour:         2
Day:          every
Month:        every
Weekday:      every
```

Therefore:

```text
Every day at 02:00
```

---

# 19. Read Cron From Left to Right

When you see:

```text
30 4 * * *
```

read:

```text
Minute      = 30
Hour        = 4
Day         = every
Month       = every
Weekday     = every
```

Result:

```text
Every day at 04:30
```

Do not memorize cron expressions as random strings.

Break them into fields.

---

# 20. Every Hour

```text
0 * * * * command
```

means:

```text
At minute 0 of every hour.
```

Examples:

```text
01:00
02:00
03:00
04:00
```

---

# 21. Every 15 Minutes

Cron supports step values.

Example:

```text
*/15 * * * * command
```

This means:

```text
Every 15 minutes
```

typically:

```text
00
15
30
45
```

of every hour.

---

# 22. Every Five Minutes

```text
*/5 * * * * command
```

means:

```text
Every five minutes.
```

This is useful for testing simple cron jobs because you do not have to wait an entire day.

Do not use unnecessarily frequent schedules for expensive tasks.

---

# 23. Every Day at Midnight

```text
0 0 * * * command
```

means:

```text
Every day at 00:00
```

Midnight is hour:

```text
0
```

not:

```text
24
```

---

# 24. Every Day at 2:30 AM

```text
30 2 * * * command
```

Read:

```text
Minute = 30
Hour = 2
```

Result:

```text
02:30 every day
```

---

# 25. Every Monday at 8:00 AM

```text
0 8 * * 1 command
```

The fifth field:

```text
1
```

represents Monday.

---

# 26. Weekdays at 6:00 PM

You can specify a range:

```text
0 18 * * 1-5 command
```

This means:

```text
Monday through Friday at 18:00
```

Remember that cron normally uses a 24-hour clock.

---

# 27. Multiple Specific Hours

Lists can be separated with commas.

Example:

```text
0 8,12,18 * * * command
```

This runs at:

```text
08:00
12:00
18:00
```

every day.

---

# 28. A Cron Job for a Script

Suppose your script is:

```text
/home/ann/scripts/health-check.sh
```

A cron entry might be:

```text
0 8 * * * /home/ann/scripts/health-check.sh
```

That requests execution every day at 08:00.

But there are several problems beginners commonly encounter here.

The script may work manually and fail under cron.

Why?

Because cron does not necessarily run with the same environment as your interactive terminal.

---

# 29. The Cron Environment

When you log into a terminal, your shell may load files such as:

```text
.bashrc
.profile
```

These can configure:

* `PATH`.
* Aliases.
* Variables.
* Functions.
* Tool locations.

Cron jobs run in a more limited, non-interactive environment.

Therefore:

```text
Works in my terminal
```

does not automatically mean:

```text
Works from cron
```

This distinction is extremely important.

---

# 30. Cron `PATH` Problems

Suppose your script contains:

```bash
mytool backup
```

It works manually.

Cron reports:

```text
mytool: command not found
```

Possible cause:

The directory containing `mytool` is in your interactive shell's `PATH`, but not cron's environment.

Investigate manually:

```bash
command -v mytool
```

Suppose:

```text
/usr/local/bin/mytool
```

Then your script can use:

```bash
/usr/local/bin/mytool backup
```

Or you can define a known `PATH` carefully in the script or crontab.

---

# 31. Using Absolute Command Paths

You can find a command with:

```bash
command -v tar
```

Example:

```text
/usr/bin/tar
```

Instead of relying on:

```bash
tar ...
```

a cron-oriented script can explicitly use:

```bash
/usr/bin/tar ...
```

when appropriate.

Absolute paths reduce dependency on environment-specific command lookup.

---

# 32. Cron Working-Directory Problems

Another common problem:

Your script contains:

```bash
cp config.conf backups/
```

When you manually run it from:

```text
/home/ann/project
```

it works.

Cron may start the job from a different working directory.

Then:

```text
config.conf
```

and:

```text
backups/
```

may refer to completely different locations or not exist.

This is why automated scripts should avoid relying accidentally on the current directory.

---

# 33. Use Absolute Paths for Files

Instead of:

```bash
source="project"
backup="backups"
```

prefer something explicit when the location is fixed:

```bash
source="/home/ann/project"
backup="/home/ann/backups"
```

Now the script knows exactly where to look regardless of its starting directory.

---

# 34. Explicitly Change Directory When Necessary

Sometimes the task genuinely needs to run from a particular directory.

Use:

```bash
cd /home/ann/project || exit 1
```

Why:

```text
|| exit 1
```

?

Because if:

```bash
cd /home/ann/project
```

fails, the script should not continue while assuming it is inside the project.

That could cause later commands to operate in the wrong directory.

---

# 35. Cron Output

When a cron job runs:

```bash
/home/ann/scripts/backup.sh
```

the script may produce standard output and standard error.

But there may be no terminal attached for you to watch.

You should decide what happens to that output.

You learned redirection in Part 7.

For example:

```text
0 2 * * * /home/ann/scripts/backup.sh > /home/ann/logs/backup.log 2>&1
```

This sends both standard output and standard error to:

```text
/home/ann/logs/backup.log
```

---

# 36. `>` vs `>>` for Cron Logs

This:

```text
> backup.log
```

overwrites the file each time.

This:

```text
>> backup.log
```

appends to the existing file.

For recurring logs, you might use:

```text
>> /home/ann/logs/backup.log 2>&1
```

But remember that an endlessly growing log file creates another problem.

Production systems should also consider log rotation and retention.

You learned the basics of `logrotate` in Part 16.

---

# 37. A Better Scheduled Command

Example:

```text
0 2 * * * /home/ann/scripts/backup.sh >> /home/ann/logs/backup.log 2>&1
```

This means:

```text
At 02:00 every day
        |
        v
Run backup.sh
        |
        +--> stdout
        |
        +--> stderr
               |
               v
       Append to backup.log
```

---

# 38. Create the Log Directory First

Cron does not automatically create:

```text
/home/ann/logs
```

If your cron entry redirects there and the directory does not exist, the job can fail before your script even starts.

Create it:

```bash
mkdir -p /home/ann/logs
```

Verify:

```bash
ls -ld /home/ann/logs
```

This demonstrates an important automation principle:

```text
Validate every dependency.
```

---

# 39. Testing Cron Safely

Do not begin by scheduling your important backup for:

```text
02:00 tomorrow
```

and then assume it will work.

First create a simple test.

For example:

```bash
mkdir -p "$HOME/cron-lab"
```

Then add to:

```bash
crontab -e
```

a temporary test such as:

```text
* * * * * /usr/bin/date >> /home/ann/cron-lab/cron-test.log 2>&1
```

This requests execution every minute.

After a few minutes:

```bash
cat /home/ann/cron-lab/cron-test.log
```

You should see timestamps.

After testing, remove the every-minute job.

Do not leave unnecessary test jobs running.

---

# 40. Verify the Installed Crontab

After editing:

```bash
crontab -l
```

Confirm that your entry is actually present.

Do not assume that because you opened the editor, the crontab was successfully saved.

---

# 41. Cron Troubleshooting Method

If a cron job does not run:

```text
Cron job failed
     |
     v
Does cron service run?
     |
     v
Is job actually installed?
     |
     v
Is schedule correct?
     |
     v
Can user execute script?
     |
     v
Are paths absolute?
     |
     v
Is PATH sufficient?
     |
     v
Does working directory matter?
     |
     v
Where is stdout/stderr going?
     |
     v
Check cron/service logs
     |
     v
Reproduce with similar environment
```

Troubleshoot systematically.

---

# 42. Cron Service Logs

On systemd-based systems, you may inspect the cron service.

Ubuntu/Debian example:

```bash
journalctl -u cron
```

Some systems:

```bash
journalctl -u crond
```

You can also inspect traditional system log locations depending on the distribution.

Do not assume cron logs are stored in exactly the same place on every Linux distribution.

---

# 43. Cron Time and Time Zones

Cron schedules are interpreted according to the system's cron/time configuration, commonly based on the system's local time.

Check:

```bash
date
```

and on systemd systems:

```bash
timedatectl
```

If you think:

```text
Run at 02:00
```

but the server is configured for another time zone, the job may run at a different wall-clock time than you expected.

Always know which clock your server uses.

---

# 44. Cron and Daylight Saving Time

Time changes can affect scheduled jobs.

Depending on system configuration and cron implementation, daylight-saving transitions can create surprising behavior around skipped or repeated local times.

For critical automation:

* Understand the server time zone.
* Understand scheduler behavior.
* Avoid making assumptions around time changes.
* Monitor whether jobs actually completed.

Scheduling something is not the same as proving that it ran successfully.

---

# 45. What Is an Archive?

Now we need another concept before building backups.

An archive combines multiple files and directories into one file.

Imagine:

```text
project/
├── config/
├── scripts/
├── logs/
└── README.md
```

An archive can package them together:

```text
project.tar
```

Think:

```text
Many files
    |
    v
Archive tool
    |
    v
One archive file
```

An archive does not necessarily mean the data has been compressed.

That distinction matters.

---

# 46. Archive vs Compression

These concepts are related but different.

Archive:

```text
Many files
    |
    v
One file
```

Compression:

```text
Data
 |
 v
Compression algorithm
 |
 v
Usually smaller representation
```

You can:

* Archive without compression.
* Compress an archive.
* Compress some individual files without first creating a multi-file archive.

`tar` is primarily an archiving tool.

`gzip` is a compression tool.

They are frequently used together.

---

# 47. What Is `tar`?

`tar` historically stands for tape archive.

It is widely used to package files and directories.

Check:

```bash
tar --version
```

Basic structure:

```text
tar [options] archive files
```

You will commonly see options such as:

```text
-c
-x
-t
-f
-v
-z
```

We will explain each.

---

# 48. Creating a Tar Archive

Create a practice directory:

```bash
mkdir -p ~/archive-lab/project
```

Create files:

```bash
touch ~/archive-lab/project/app.conf
touch ~/archive-lab/project/README.txt
touch ~/archive-lab/project/deploy.sh
```

Move to:

```bash
cd ~/archive-lab
```

Create an archive:

```bash
tar -cf project.tar project/
```

---

# 49. Understanding `tar -cf`

The command:

```bash
tar -cf project.tar project/
```

contains:

```text
tar
```

the program.

```text
-c
```

means:

```text
Create an archive.
```

```text
-f
```

means:

```text
The next relevant argument identifies the archive file.
```

```text
project.tar
```

is the archive being created.

```text
project/
```

is the directory being archived.

---

# 50. Verify the Archive Exists

Run:

```bash
ls -lh project.tar
```

You should see the archive.

But:

```text
File exists
```

does not necessarily prove:

```text
Backup contains what I expected.
```

We need to inspect it.

---

# 51. Listing Archive Contents

Run:

```bash
tar -tf project.tar
```

The:

```text
-t
```

means list archive contents.

You should see something similar to:

```text
project/
project/app.conf
project/README.txt
project/deploy.sh
```

This is a basic form of verification.

---

# 52. Verbose Tar Output

Add:

```text
-v
```

for verbose output.

Example:

```bash
tar -tvf project.tar
```

This displays more information about archived entries.

Similarly, during creation:

```bash
tar -cvf project.tar project/
```

prints the names being processed.

Verbose output is useful while learning, but it can create large logs when archiving thousands of files.

---

# 53. Extracting an Archive

Create a destination:

```bash
mkdir extracted
```

Then:

```bash
tar -xf project.tar -C extracted
```

The:

```text
-x
```

means extract.

The:

```text
-C
```

means change to the specified directory for the operation.

The archive contents should appear under:

```text
extracted/
```

Verify:

```bash
find extracted -maxdepth 3 -type f
```

---

# 54. Be Careful When Extracting Archives

Do not blindly extract unknown archives into important directories.

An archive can contain:

* Existing filenames.
* Nested directories.
* Unexpected paths.
* Potentially unsafe content.

A safer learning workflow is:

```text
Unknown archive
      |
      v
Inspect contents
      |
      v
Create temporary directory
      |
      v
Extract there
      |
      v
Inspect result
```

Use:

```bash
tar -tf archive.tar
```

before extracting when appropriate.

---

# 55. What Is gzip?

`gzip` is a compression utility.

Suppose:

```text
project.tar
```

is an uncompressed archive.

Run:

```bash
gzip project.tar
```

You will typically end up with:

```text
project.tar.gz
```

and the original `project.tar` is normally replaced by the compressed version unless options change that behavior.

---

# 56. `.tar.gz`

A file named:

```text
project.tar.gz
```

usually tells you:

```text
.tar
=
tar archive

.gz
=
gzip compressed
```

Conceptually:

```text
Files
  |
  v
tar
  |
  v
project.tar
  |
  v
gzip
  |
  v
project.tar.gz
```

---

# 57. Create and Compress With One tar Command

Instead of creating `.tar` and then running `gzip`, use:

```bash
tar -czf project.tar.gz project/
```

The:

```text
-z
```

means use gzip compression.

So:

```text
-c = create
-z = gzip
-f = archive filename
```

---

# 58. Extracting `.tar.gz`

Run:

```bash
tar -xzf project.tar.gz
```

Here:

```text
-x = extract
-z = gzip
-f = archive file
```

You can also extract into another directory:

```bash
mkdir restore-test
```

Then:

```bash
tar -xzf project.tar.gz -C restore-test
```

---

# 59. Listing a Compressed Archive

Use:

```bash
tar -tzf project.tar.gz
```

This lets you inspect the contents without performing a full restoration.

---

# 60. Compression Does Not Guarantee a Smaller File

Compression often reduces size significantly for text and repetitive data.

But files such as:

* JPEG images.
* MP4 videos.
* ZIP archives.
* Already compressed formats.

may not shrink much.

Some may even gain a small amount of overhead.

Do not assume:

```text
gzip always makes everything dramatically smaller.
```

---

# 61. What Is ZIP?

ZIP is another archive and compression format.

You may encounter it frequently when exchanging files across operating systems.

Check:

```bash
command -v zip
command -v unzip
```

If installed, create:

```bash
zip -r project.zip project/
```

The:

```text
-r
```

means recursively include directory contents.

---

# 62. Inspecting ZIP Files

Use:

```bash
unzip -l project.zip
```

The:

```text
-l
```

lists archive contents.

This allows inspection before extraction.

---

# 63. Extracting ZIP Files

Create:

```bash
mkdir zip-restore
```

Then:

```bash
unzip project.zip -d zip-restore
```

The:

```text
-d
```

selects the destination directory.

Verify:

```bash
find zip-restore -maxdepth 3 -type f
```

---

# 64. tar.gz vs ZIP

Both are useful.

`tar.gz` is extremely common in Linux and Unix environments.

ZIP is common for cross-platform file exchange.

Conceptually:

| Format    | Common Linux use                                           |
| --------- | ---------------------------------------------------------- |
| `.tar`    | Archive files without gzip compression                     |
| `.gz`     | gzip-compressed data                                       |
| `.tar.gz` | tar archive compressed with gzip                           |
| `.zip`    | Combined archive/compression format, widely cross-platform |

The correct choice depends on the task.

---

# 65. What Is a Backup?

A backup is a separate copy of data maintained so that the data can be recovered after loss, corruption, deletion, failure, or another incident.

That last part matters.

Simply creating:

```text
project-copy/
```

does not automatically mean you have a good backup strategy.

A useful backup strategy considers:

```text
What are we protecting?

From what failure?

How often?

Where is the backup stored?

How long is it retained?

How do we know it succeeded?

Can we restore it?
```

---

# 66. Why Backups Matter

Data can be lost because of:

* Accidental deletion.
* Disk failure.
* Filesystem corruption.
* Application bugs.
* Failed deployments.
* Misconfiguration.
* Administrator mistakes.
* Security incidents.
* Hardware failure.
* Cloud-resource deletion.
* Software upgrades.
* Automation mistakes.

Backups reduce the impact of some of these failures.

But only when the backup itself is usable.

---

# 67. Copy Is Not Automatically a Good Backup

Suppose:

```text
Original:
/data/application

Backup:
/data/application-backup
```

Both are on the same disk.

If that disk fails, you may lose both.

So:

```text
Second copy
```

is better than no copy, but location matters.

A stronger strategy separates failure domains.

---

# 68. What Should Be Backed Up?

It depends on the system.

Possible examples include:

* Application data.
* User-generated files.
* Databases.
* Important configuration.
* Certificates when appropriate.
* Application configuration.
* Infrastructure configuration.
* Scripts.
* Deployment configuration.
* Business-critical files.
* System-specific configuration that cannot easily be recreated.

But you should not blindly back up everything.

---

# 69. What May Not Need Traditional Backup?

Some data may be reproducible.

For example:

* Operating-system packages that can be reinstalled.
* Application binaries that can be rebuilt.
* Container images stored safely in a registry.
* Infrastructure recreated from version-controlled code.
* Temporary files.
* Cache files.

The important question is:

```text
If this disappears, how do I recover it?
```

If the answer is:

```text
I cannot recreate it
```

it is a strong backup candidate.

---

# 70. Configuration Is Important

Consider:

```text
/etc/nginx/
```

Your web application may be easy to redeploy.

But if its server configuration exists only on the server and disappears, recovery becomes harder.

This is one reason modern engineering also uses:

* Version control.
* Infrastructure as Code.
* Configuration management.

Backups and reproducible infrastructure solve related but different problems.

---

# 71. Databases Need Special Treatment

Do not assume that creating:

```bash
tar -czf database-files.tar.gz /var/lib/database
```

while a database is running automatically creates a valid application-consistent backup.

Databases may have:

* In-memory state.
* Transaction logs.
* Active writes.
* Consistency requirements.
* Database-specific backup mechanisms.

For real database backups, use the database's supported backup or snapshot procedures.

This lesson focuses on file and directory backups.

---

# 72. Backup Location

Suppose your source is:

```text
/home/ann/project
```

You could store a lab backup in:

```text
/home/ann/backups
```

That is useful for learning.

But production backup design should consider storing copies on separate systems or storage services.

Why?

Because this:

```text
Server disk fails
     |
     +--> Original lost
     |
     +--> Local backup lost
```

is not sufficient protection.

---

# 73. Backup Naming

This is poor:

```text
backup.tar.gz
```

if every new run overwrites the previous backup.

A timestamp gives each backup a distinct identity.

Example:

```text
project-2026-08-27_023000.tar.gz
```

Now you can see when it was created.

---

# 74. Creating a Timestamp

Run:

```bash
date '+%Y-%m-%d_%H%M%S'
```

Possible output:

```text
2026-08-27_023000
```

Components:

```text
%Y = four-digit year
%m = month
%d = day
%H = hour
%M = minute
%S = second
```

---

# 75. Store a Timestamp in a Variable

```bash
timestamp=$(date '+%Y-%m-%d_%H%M%S')
```

Then:

```bash
echo "$timestamp"
```

Use it:

```bash
backup_name="project-${timestamp}.tar.gz"
```

Example:

```text
project-2026-08-27_023000.tar.gz
```

---

# 76. Creating a Backup Directory

Use:

```bash
mkdir -p "$HOME/backups"
```

The:

```text
-p
```

allows parent directories to be created as needed and avoids an error when the directory already exists.

Verify:

```bash
ls -ld "$HOME/backups"
```

---

# 77. A Simple Manual Backup

Suppose:

```text
Source:
/home/ann/project

Destination:
/home/ann/backups
```

Create:

```bash
timestamp=$(date '+%Y-%m-%d_%H%M%S')
```

Then:

```bash
tar -czf "/home/ann/backups/project-${timestamp}.tar.gz" \
    -C "/home/ann" project
```

Why use `-C`?

Because it avoids storing a long absolute path structure inside the archive and gives you better control over archive paths.

---

# 78. Verify the Backup File Exists

After creation:

```bash
ls -lh /home/ann/backups/
```

But again:

```text
Backup file exists
```

is only the first verification step.

We should inspect the archive.

---

# 79. Verify Archive Contents

Run:

```bash
tar -tzf "/home/ann/backups/project-${timestamp}.tar.gz"
```

You should see expected files.

This confirms that `tar` can at least read the archive structure and that expected entries appear.

---

# 80. Test Restoration

A much stronger test is to restore the backup.

Create:

```bash
mkdir -p /tmp/backup-restore-test
```

Extract:

```bash
tar -xzf "/home/ann/backups/project-${timestamp}.tar.gz" \
    -C /tmp/backup-restore-test
```

Inspect:

```bash
find /tmp/backup-restore-test -maxdepth 3 -type f
```

Now compare the restored content with what you expected.

---

# 81. A Backup Is Only Useful If It Can Be Restored

This principle is extremely important:

```text
Backup created
      !=
Recovery guaranteed
```

A better lifecycle is:

```text
Create
  |
  v
Verify
  |
  v
Protect
  |
  v
Retain
  |
  v
Test restore
```

A backup strategy without restore testing is incomplete.

---

# 82. Permissions and Backups

Your backup user needs permission to read the source data.

Suppose:

```bash
tar -czf backup.tar.gz /some/protected/directory
```

returns:

```text
Permission denied
```

Do not immediately run the entire process as root.

Ask:

* Which files require access?
* Should this user be allowed to read them?
* Can a dedicated backup account be used?
* What permissions are appropriate?
* Will the backup itself contain sensitive data?

Backups often contain some of the most sensitive information on a system.

Protect them accordingly.

---

# 83. Backup File Permissions

Suppose a backup contains:

* Configuration files.
* Private application data.
* User information.
* Credentials accidentally stored in configuration.
* Certificates or keys.

You may not want:

```text
-rw-r--r--
```

for every backup.

Depending on the use case, a restrictive permission such as:

```bash
chmod 600 backup-file.tar.gz
```

may be appropriate.

Recall from Part 5:

```text
600
=
owner: read + write
group: none
others: none
```

Choose permissions based on actual access requirements.

---

# 84. Retention

If you create one backup every day forever:

```text
Day 1 backup
Day 2 backup
Day 3 backup
...
Day 5000 backup
```

storage usage continues growing.

Retention defines how long backups are kept.

Example policy:

```text
Keep daily backups for 7 days.
```

More advanced strategies may keep:

* Daily backups.
* Weekly backups.
* Monthly backups.
* Annual archives.

For this beginner lesson, we will use a simple age-based retention rule.

---

# 85. Finding Old Backups

Suppose backups live in:

```text
/home/ann/backups
```

To inspect files older than a specified age, `find` can be useful.

Example:

```bash
find /home/ann/backups \
    -type f \
    -name 'project-*.tar.gz' \
    -mtime +7
```

Important: `find -mtime` uses completed 24-hour periods and its boundary behavior can surprise beginners. Before deleting anything, test the exact expression against sample files and confirm that it selects the ages you intend.

Most importantly, this command only displays matches.

It does not delete them.

Always inspect first.

---

# 86. Why You Should Inspect Before Deleting

Never begin retention testing with:

```bash
find ... -delete
```

Start with:

```bash
find ...
```

Inspect the results.

Ask:

```text
Are these exactly the files I intend to remove?
```

Only after validating the selection should deletion be considered.

This follows:

```text
Observe
   |
   v
Verify
   |
   v
Act
```

not:

```text
Delete
   |
   v
Discover mistake
```

---

# 87. Restrict Retention to Expected Files

This is dangerous thinking:

```bash
find /home/ann -type f -mtime +7 -delete
```

It is far too broad.

A safer design restricts:

* Exact backup directory.
* File type.
* Naming pattern.
* Age.
* Possibly depth.

For example, after careful testing:

```bash
find "$backup_dir" \
    -maxdepth 1 \
    -type f \
    -name 'project-*.tar.gz' \
    -mtime +7
```

Before adding deletion, verify the output.

---

# 88. Backup Script Requirements

Our automated script should:

1. Know the source directory.
2. Know the backup directory.
3. Know the retention policy.
4. Validate required commands.
5. Validate the source.
6. Create the destination.
7. Generate a timestamp.
8. Create a compressed archive.
9. Detect archive failure.
10. Verify the archive exists.
11. Verify the archive is readable.
12. Protect its permissions.
13. Apply retention carefully.
14. Log meaningful information.
15. Return an appropriate exit code.

Now we will build it step by step.

---

# 89. Create the Backup Project

Run:

```bash
mkdir -p "$HOME/linux-automation-lab/scripts"
mkdir -p "$HOME/linux-automation-lab/source"
mkdir -p "$HOME/linux-automation-lab/backups"
mkdir -p "$HOME/linux-automation-lab/logs"
```

Create some source data:

```bash
mkdir -p "$HOME/linux-automation-lab/source/config"
mkdir -p "$HOME/linux-automation-lab/source/data"
```

Create files:

```bash
echo "server_port=8080" \
    > "$HOME/linux-automation-lab/source/config/app.conf"

echo "Application data" \
    > "$HOME/linux-automation-lab/source/data/application.txt"
```

Inspect:

```bash
find "$HOME/linux-automation-lab/source" -type f
```

---

# 90. Create the Backup Script

Run:

```bash
nano "$HOME/linux-automation-lab/scripts/backup.sh"
```

Start with:

```bash
#!/bin/bash
```

Then configuration:

```bash
source_dir="$HOME/linux-automation-lab/source"
backup_dir="$HOME/linux-automation-lab/backups"
retention_days=7
```

---

# 91. Why Configuration Goes Near the Top

These values may change.

Instead of scattering:

```text
/home/ann/linux-automation-lab/backups
```

throughout the script, define it once.

Then functions use:

```bash
"$backup_dir"
```

This improves maintainability.

---

# 92. Add Logging Functions

Add:

```bash
log_info() {
    echo "$(date '+%Y-%m-%d %H:%M:%S') INFO: $*"
}

log_error() {
    echo "$(date '+%Y-%m-%d %H:%M:%S') ERROR: $*" >&2
}
```

Now:

```bash
log_info "Backup started"
```

might produce:

```text
2026-08-27 02:30:00 INFO: Backup started
```

---

# 93. Validate Required Commands

Create:

```bash
check_dependencies() {
    local missing=0

    for command_name in tar find date chmod mkdir; do
        if ! command -v "$command_name" >/dev/null 2>&1; then
            log_error "Required command not found: $command_name"
            missing=1
        fi
    done

    if [ "$missing" -ne 0 ]; then
        return 1
    fi

    return 0
}
```

This prevents the script from blindly continuing when required utilities are unavailable.

---

# 94. Validate the Source Directory

Create:

```bash
check_source() {
    if [ ! -d "$source_dir" ]; then
        log_error "Source directory does not exist: $source_dir"
        return 1
    fi

    return 0
}
```

Why?

Because if the source is wrong, creating an empty or incorrect backup and reporting success would be dangerous.

---

# 95. Prepare the Backup Directory

Create:

```bash
prepare_backup_directory() {
    if ! mkdir -p "$backup_dir"; then
        log_error "Could not create backup directory: $backup_dir"
        return 1
    fi

    return 0
}
```

This handles the possibility that `mkdir` itself fails.

Possible causes include:

* Permission denied.
* Read-only filesystem.
* Invalid path.
* Storage problems.

---

# 96. Generate the Backup Filename

Create:

```bash
create_backup_name() {
    local timestamp

    timestamp=$(date '+%Y-%m-%d_%H%M%S') || {
        log_error "Could not generate timestamp"
        return 1
    }

    backup_file="$backup_dir/source-${timestamp}.tar.gz"

    return 0
}
```

Notice that:

```text
backup_file
```

is not declared `local`.

Why?

Because other functions need to know which backup file was created.

---

# 97. Create the Archive

We want the archive to contain:

```text
source/
```

rather than a long path such as:

```text
home/ann/linux-automation-lab/source/
```

So use:

```bash
tar -czf "$backup_file" \
    -C "$HOME/linux-automation-lab" source
```

Turn that into a function:

```bash
create_backup() {
    log_info "Creating backup: $backup_file"

    if ! tar -czf "$backup_file" \
        -C "$HOME/linux-automation-lab" source; then
        log_error "Backup creation failed"
        return 1
    fi

    return 0
}
```

---

# 98. Why Quote the Backup Filename?

We use:

```bash
"$backup_file"
```

instead of:

```bash
$backup_file
```

because paths can contain spaces or shell-sensitive characters.

Quoting variable expansions is one of the most important Bash habits you learned in Part 17.

---

# 99. Verify the File Exists

Create:

```bash
verify_backup_exists() {
    if [ ! -f "$backup_file" ]; then
        log_error "Backup file was not created: $backup_file"
        return 1
    fi

    return 0
}
```

But this is still weak verification.

The file could exist but be unusable.

---

# 100. Verify the Archive Can Be Read

Use:

```bash
tar -tzf "$backup_file" >/dev/null
```

If `tar` cannot read the archive, it should return a non-zero status.

Create:

```bash
verify_archive() {
    log_info "Verifying archive"

    if ! tar -tzf "$backup_file" >/dev/null; then
        log_error "Archive verification failed: $backup_file"
        return 1
    fi

    log_info "Archive verification passed"
    return 0
}
```

This is stronger than checking existence alone.

---

# 101. Archive Verification Has Limits

Running:

```bash
tar -tzf backup.tar.gz
```

proves useful things, but not everything.

It helps establish that:

* The archive is readable by `tar`.
* The compressed structure is not obviously broken.
* Entries can be listed.

It does not prove:

* Every required application file was selected.
* The backup reflects an application-consistent state.
* Every restored file will satisfy your recovery requirements.
* Your complete disaster-recovery process works.

Real backup validation requires restore testing.

---

# 102. Protect the Backup

Create:

```bash
protect_backup() {
    if ! chmod 600 "$backup_file"; then
        log_error "Could not set backup permissions"
        return 1
    fi

    return 0
}
```

Now only the owner should have read/write permissions.

Verify manually:

```bash
ls -l "$HOME/linux-automation-lab/backups"
```

---

# 103. Apply Retention Carefully

First, understand the selection:

```bash
find "$backup_dir" \
    -maxdepth 1 \
    -type f \
    -name 'source-*.tar.gz' \
    -mtime +7
```

Once you have tested and confirmed the expression in your lab, the script can remove matching files.

For clarity, we can process them one by one.

---

# 104. Retention Function

```bash
apply_retention() {
    log_info "Checking for backups older than retention period"

    while IFS= read -r -d '' old_backup; do
        log_info "Removing old backup: $old_backup"

        if ! rm -- "$old_backup"; then
            log_error "Could not remove old backup: $old_backup"
            return 1
        fi
    done < <(
        find "$backup_dir" \
            -maxdepth 1 \
            -type f \
            -name 'source-*.tar.gz' \
            -mtime +"$retention_days" \
            -print0
    )

    return 0
}
```

This introduces several useful defensive techniques.

---

# 105. Why `-print0`?

Filenames can contain spaces.

A simple line-based approach can sometimes become fragile when unusual filenames are involved.

`find -print0` separates filenames using a null character.

Then:

```bash
read -r -d ''
```

reads each null-separated pathname.

This is safer for arbitrary filenames.

---

# 106. Why `rm -- "$old_backup"`?

The:

```text
--
```

marks the end of command options.

Imagine a filename beginning with:

```text
-
```

Without `--`, some commands could interpret it as an option.

Using:

```bash
rm -- "$old_backup"
```

helps ensure the value is treated as a pathname.

---

# 107. Retention Still Requires Care

Even defensive syntax does not make deletion automatically safe.

Before enabling retention in a real environment, verify:

* `backup_dir`.
* Filename pattern.
* Age calculation.
* Permissions.
* Retention requirement.
* Whether another system manages retention.
* Whether legal or business requirements require longer storage.

Never copy a retention command into production without understanding what it selects.

---

# 108. Main Function

Now create:

```bash
main() {
    log_info "Backup process started"

    check_dependencies || return 1
    check_source || return 1
    prepare_backup_directory || return 1
    create_backup_name || return 1
    create_backup || return 1
    verify_backup_exists || return 1
    verify_archive || return 1
    protect_backup || return 1
    apply_retention || return 1

    log_info "Backup completed successfully: $backup_file"

    return 0
}
```

At the bottom:

```bash
main
exit $?
```

---

# 109. Complete Automated Backup Script

The complete beginner project now looks like:

```bash
#!/bin/bash

# Automated Linux file backup
# Creates a timestamped compressed archive,
# verifies it, restricts permissions,
# and applies basic retention.

base_dir="$HOME/linux-automation-lab"
source_dir="$base_dir/source"
backup_dir="$base_dir/backups"
retention_days=7
backup_file=""

log_info() {
    echo "$(date '+%Y-%m-%d %H:%M:%S') INFO: $*"
}

log_error() {
    echo "$(date '+%Y-%m-%d %H:%M:%S') ERROR: $*" >&2
}

check_dependencies() {
    local missing=0

    for command_name in tar find date chmod mkdir rm; do
        if ! command -v "$command_name" >/dev/null 2>&1; then
            log_error "Required command not found: $command_name"
            missing=1
        fi
    done

    if [ "$missing" -ne 0 ]; then
        return 1
    fi

    return 0
}

check_source() {
    if [ ! -d "$source_dir" ]; then
        log_error "Source directory does not exist: $source_dir"
        return 1
    fi

    return 0
}

prepare_backup_directory() {
    if ! mkdir -p "$backup_dir"; then
        log_error "Could not create backup directory: $backup_dir"
        return 1
    fi

    return 0
}

create_backup_name() {
    local timestamp

    timestamp=$(date '+%Y-%m-%d_%H%M%S') || {
        log_error "Could not generate timestamp"
        return 1
    }

    backup_file="$backup_dir/source-${timestamp}.tar.gz"

    return 0
}

create_backup() {
    log_info "Creating backup: $backup_file"

    if ! tar -czf "$backup_file" \
        -C "$base_dir" source; then
        log_error "Backup creation failed"
        return 1
    fi

    return 0
}

verify_backup_exists() {
    if [ ! -f "$backup_file" ]; then
        log_error "Backup file does not exist: $backup_file"
        return 1
    fi

    return 0
}

verify_archive() {
    log_info "Verifying backup archive"

    if ! tar -tzf "$backup_file" >/dev/null; then
        log_error "Archive verification failed: $backup_file"
        return 1
    fi

    log_info "Archive verification passed"
    return 0
}

protect_backup() {
    if ! chmod 600 "$backup_file"; then
        log_error "Could not set permissions on: $backup_file"
        return 1
    fi

    return 0
}

apply_retention() {
    log_info "Checking backup retention"

    while IFS= read -r -d '' old_backup; do
        log_info "Removing old backup: $old_backup"

        if ! rm -- "$old_backup"; then
            log_error "Could not remove old backup: $old_backup"
            return 1
        fi
    done < <(
        find "$backup_dir" \
            -maxdepth 1 \
            -type f \
            -name 'source-*.tar.gz' \
            -mtime +"$retention_days" \
            -print0
    )

    return 0
}

main() {
    log_info "Backup process started"

    check_dependencies || return 1
    check_source || return 1
    prepare_backup_directory || return 1
    create_backup_name || return 1
    create_backup || return 1
    verify_backup_exists || return 1
    verify_archive || return 1
    protect_backup || return 1
    apply_retention || return 1

    log_info "Backup completed successfully: $backup_file"

    return 0
}

main
exit $?
```

---

# 110. Check the Script Syntax

Before execution:

```bash
bash -n "$HOME/linux-automation-lab/scripts/backup.sh"
```

If no syntax error appears, continue.

Remember:

```text
No syntax errors
```

does not mean:

```text
The backup logic is definitely correct.
```

---

# 111. Make the Script Executable

Run:

```bash
chmod +x "$HOME/linux-automation-lab/scripts/backup.sh"
```

Verify:

```bash
ls -l "$HOME/linux-automation-lab/scripts/backup.sh"
```

Then execute manually:

```bash
"$HOME/linux-automation-lab/scripts/backup.sh"
```

---

# 112. Check the Exit Status

Immediately:

```bash
echo $?
```

Successful execution should return:

```text
0
```

If it returns a non-zero value, investigate before scheduling it.

Never schedule an untested script simply because the syntax check passed.

---

# 113. Inspect the Backup Directory

Run:

```bash
ls -lh "$HOME/linux-automation-lab/backups"
```

You should see something similar to:

```text
source-2026-08-27_023000.tar.gz
```

Inspect permissions:

```bash
ls -l "$HOME/linux-automation-lab/backups"
```

The archive should have restrictive permissions if `chmod 600` succeeded.

---

# 114. Inspect the Archive

Store the newest backup path carefully:

```bash
latest_backup=$(find "$HOME/linux-automation-lab/backups" \
    -maxdepth 1 \
    -type f \
    -name 'source-*.tar.gz' \
    -printf '%T@ %p\n' 2>/dev/null |
    sort -n |
    tail -n 1 |
    cut -d' ' -f2-)
```

On GNU/Linux this can identify the newest matching backup.

For a simpler beginner lab, you can also copy the exact filename from:

```bash
ls -lh "$HOME/linux-automation-lab/backups"
```

and inspect it directly:

```bash
tar -tzf "/exact/path/to/backup.tar.gz"
```

Confirm that expected files are present.

---

# 115. Perform a Restore Test

Create:

```bash
rm -rf /tmp/linux-backup-restore-test
mkdir -p /tmp/linux-backup-restore-test
```

Warning: `rm -rf` is destructive. Here it targets a specific disposable lab directory. Verify the path before running it.

Extract:

```bash
tar -xzf "/exact/path/to/backup.tar.gz" \
    -C /tmp/linux-backup-restore-test
```

Inspect:

```bash
find /tmp/linux-backup-restore-test -type f
```

Read a restored file:

```bash
cat /tmp/linux-backup-restore-test/source/config/app.conf
```

Expected:

```text
server_port=8080
```

Now you have tested actual recovery, not just archive creation.

---

# 116. Schedule the Backup

Only after manual testing should you schedule it.

Edit:

```bash
crontab -e
```

For a daily 02:00 backup:

```text
0 2 * * * /home/ann/linux-automation-lab/scripts/backup.sh >> /home/ann/linux-automation-lab/logs/backup.log 2>&1
```

Replace:

```text
/home/ann
```

with the actual home directory on your system.

Check it with:

```bash
echo "$HOME"
```

Do not blindly copy another person's home path.

---

# 117. Why `$HOME` May Be Better Resolved Before Cron

Inside a cron command, environment expansion can differ from what you expect.

For beginner cron entries, explicit absolute paths make troubleshooting easier.

Find your home:

```bash
echo "$HOME"
```

Suppose:

```text
/home/ubuntu
```

Then use:

```text
0 2 * * * /home/ubuntu/linux-automation-lab/scripts/backup.sh >> /home/ubuntu/linux-automation-lab/logs/backup.log 2>&1
```

---

# 118. Test the Scheduled Version First

Waiting until 02:00 makes troubleshooting slow.

Temporarily schedule:

```text
* * * * * /home/ann/linux-automation-lab/scripts/backup.sh >> /home/ann/linux-automation-lab/logs/backup.log 2>&1
```

Wait a few minutes.

Then:

```bash
ls -lh /home/ann/linux-automation-lab/backups
```

and:

```bash
cat /home/ann/linux-automation-lab/logs/backup.log
```

If successful, change the schedule to the intended frequency.

Do not leave the every-minute backup enabled.

Otherwise you could generate:

```text
60 backups per hour
1,440 backups per day
```

before retention behavior catches up.

---

# 119. Cron Troubleshooting: Script Works Manually but Not Automatically

This is one of the most important beginner scenarios.

Problem:

```text
./backup.sh
```

works.

Cron does not.

Do not immediately rewrite the entire script.

Investigate differences between:

```text
Interactive shell
```

and:

```text
Cron environment
```

Check:

* `PATH`.
* Current working directory.
* Environment variables.
* File permissions.
* Script permissions.
* Home directory assumptions.
* Relative paths.
* Service availability.
* Credentials.
* Output redirection.

---

# 120. Cron Troubleshooting: Wrong PATH

Problem:

```text
command not found
```

inside the cron log.

Investigate:

```bash
command -v command_name
```

Then either:

* Use an absolute command path.
* Define an appropriate `PATH`.
* Fix the script's environment assumptions.

Do not assume cron loads your `.bashrc`.

---

# 121. Cron Troubleshooting: Relative Path Failure

Script:

```bash
tar -czf backups/data.tar.gz source/
```

Manual execution works.

Cron fails.

Possible cause:

Cron starts from another directory.

Fix:

```bash
base_dir="/home/ann/linux-automation-lab"
```

Then use:

```bash
"$base_dir/backups"
```

and:

```bash
"$base_dir/source"
```

Explicit paths make automation much more predictable.

---

# 122. Cron Troubleshooting: Permission Denied

Possible symptoms:

```text
Permission denied
```

Check:

```bash
ls -l /path/to/script
```

Check parent directory access:

```bash
namei -l /path/to/script
```

if `namei` is available.

Check source permissions:

```bash
ls -ld /path/to/source
```

Check destination:

```bash
ls -ld /path/to/backups
```

Ask:

```text
Which user owns this cron job?

Can that user read the source?

Can that user execute the script?

Can that user write to the destination?
```

---

# 123. Cron Troubleshooting: No Log File

Suppose:

```text
>> /home/ann/logs/backup.log
```

produces no log.

Check whether:

```text
/home/ann/logs
```

exists.

Redirection is prepared by the shell before the script executes.

If the destination directory does not exist, the script may never start.

---

# 124. Cron Troubleshooting: Wrong Schedule

Suppose you intended:

```text
02:30
```

but wrote:

```text
2 30 * * *
```

This is wrong because the fields are:

```text
Minute Hour
```

not:

```text
Hour Minute
```

Correct:

```text
30 2 * * *
```

Always decode cron expressions field by field.

---

# 125. Cron Troubleshooting: Wrong User

You may create a job with:

```bash
crontab -e
```

as one user, then look for it under another user's environment.

Check:

```bash
whoami
```

Then:

```bash
crontab -l
```

Cron jobs and permissions are user-specific.

---

# 126. Cron Troubleshooting: Script Has No Execute Permission

Check:

```bash
ls -l /home/ann/linux-automation-lab/scripts/backup.sh
```

If necessary:

```bash
chmod +x /home/ann/linux-automation-lab/scripts/backup.sh
```

Alternatively, a cron entry could explicitly invoke Bash:

```text
0 2 * * * /bin/bash /home/ann/linux-automation-lab/scripts/backup.sh ...
```

But do not use this to ignore a malformed script. Understand how you intend the script to be executed.

---

# 127. Cron Troubleshooting: Environment Variable Missing

Suppose your interactive shell contains:

```bash
export APP_HOME="/opt/myapp"
```

and your script expects:

```bash
"$APP_HOME"
```

Cron may not have that variable.

The script could unexpectedly receive an empty value.

Solutions depend on the application:

* Define required configuration explicitly.
* Load a controlled configuration file.
* Set required variables in the scheduled environment.
* Validate that required variables are non-empty.

Do not silently continue with empty critical variables.

---

# 128. Validate Important Variables

For example:

```bash
if [ -z "$source_dir" ]; then
    log_error "source_dir is empty"
    exit 1
fi
```

For destructive operations, validation should be even stronger.

Automation should fail safely when assumptions are not met.

---

# 129. Backup Troubleshooting: Archive Is Empty

Problem:

The backup file exists, but expected content is missing.

Possible causes:

* Wrong source path.
* Wrong working directory.
* Incorrect `tar -C`.
* Source directory was empty.
* Permissions prevented reading files.
* Files were excluded unintentionally.

Investigate:

```bash
tar -tzf backup.tar.gz
```

Then:

```bash
find /expected/source -type f
```

Compare expected input with archive contents.

---

# 130. Backup Troubleshooting: Permission Denied During tar

Problem:

```text
tar: ... Permission denied
```

Investigate the exact path.

Run:

```bash
ls -l /problem/path
```

and:

```bash
ls -ld /problem/directory
```

Remember:

Directory traversal requires appropriate execute permission.

Do not automatically solve this with:

```bash
chmod -R 777 ...
```

That is usually an unsafe response.

Fix the actual ownership or permission requirement.

---

# 131. Backup Troubleshooting: No Space Left on Device

Problem:

```text
No space left on device
```

Investigate:

```bash
df -h
```

Then:

```bash
df -i
```

Why both?

Because the system may have:

* No storage blocks available.
* No inodes available.

Then inspect:

```bash
du -sh /path/*
```

where appropriate.

Do not delete files randomly.

Identify what consumed the space.

---

# 132. Backup Troubleshooting: Backup Grows Every Day

Possible causes:

* Retention is not running.
* Retention pattern does not match filenames.
* `mtime` assumption is wrong.
* Cron is creating backups more frequently than intended.
* Backup destination is accidentally included in the source.

That final problem is especially important.

---

# 133. Never Accidentally Back Up the Backup Directory Into Itself

Imagine:

```text
Source:
/home/ann

Backup:
/home/ann/backups
```

If you archive the entire source including the backup directory, each new backup may contain previous backups.

You could create:

```text
Backup 1
Backup 2 contains Backup 1
Backup 3 contains Backup 1 + Backup 2
...
```

Storage usage can grow dramatically.

Design source and destination paths carefully.

---

# 134. Backup Troubleshooting: Corrupt Archive

Problem:

```bash
tar -tzf backup.tar.gz
```

returns an error.

Possible causes:

* Backup creation was interrupted.
* Disk became full.
* File was truncated.
* Storage corruption.
* Incomplete transfer.
* Compression failed.

Do not delete the original source because a backup file merely exists.

Verification must happen first.

---

# 135. Backup Troubleshooting: Restore Fails

A backup that cannot be restored is a serious problem.

Investigate:

1. Can the archive be listed?
2. Can it be extracted into a temporary directory?
3. Are permissions sufficient?
4. Is there enough disk space?
5. Does the archive contain expected paths?
6. Were application-specific backup procedures required?
7. Is the backup complete?

Restore testing should be part of backup operations, not an activity first attempted during a disaster.

---

# 136. Backups and Encryption

Compression is not encryption.

A file:

```text
backup.tar.gz
```

is not automatically protected from someone who obtains the file.

`gzip` compresses.

It does not provide confidentiality.

Production backup strategies may require encryption:

```text
Data
 |
 v
Backup
 |
 v
Encryption
 |
 v
Protected backup storage
```

The exact encryption method depends on the environment and is beyond this beginner section, but remember:

```text
Compressed != encrypted
```

---

# 137. Backups and the 3-2-1 Principle

A widely used backup guideline is the 3-2-1 idea:

```text
3 copies of important data
2 different types of storage/media
1 copy off-site
```

The exact implementation varies with modern cloud and infrastructure environments, but the underlying principle is useful.

Do not allow one failure to destroy every copy.

---

# 138. Local Backup vs Remote Backup

Local:

```text
Server
 |
 +-- Application
 |
 +-- Backup
```

Problem:

```text
Server/storage failure
       |
       v
Potential loss of both
```

Remote:

```text
Server
 |
 +-- Application
 |
 +------> Separate backup location
```

This reduces some shared failure risks.

Production backup architecture should consider independent failure domains.

---

# 139. Backup Frequency

How frequently should backups run?

There is no universal answer.

It depends on:

```text
How much data can the organization afford to lose?
```

If backups run once every 24 hours, a failure could potentially expose a much larger recovery gap than if appropriate backups or replication occur more frequently.

This leads to an important concept.

---

# 140. Recovery Point Objective

Recovery Point Objective, or RPO, describes the acceptable amount of data loss measured in time.

Example:

```text
RPO = 24 hours
```

might mean the organization accepts the possibility of recovering to data as old as approximately 24 hours, depending on the backup design.

Another system might require:

```text
RPO = 15 minutes
```

The backup strategy should match business requirements.

---

# 141. Recovery Time Objective

Recovery Time Objective, or RTO, concerns how quickly service should be restored after disruption.

For example:

```text
RTO = 4 hours
```

means recovery procedures should be designed around restoring acceptable service within that target.

RPO asks:

```text
How much recent data can we afford to lose?
```

RTO asks:

```text
How long can recovery take?
```

These become increasingly important as you move toward intermediate infrastructure engineering.

---

# 142. Automation Is More Than Scheduling

At first, automation may look like:

```text
cron + script
```

But reliable automation requires:

```text
Correct task
     |
     v
Safe script
     |
     v
Input validation
     |
     v
Error handling
     |
     v
Scheduling
     |
     v
Logging
     |
     v
Verification
     |
     v
Monitoring
```

If nobody knows that a scheduled backup has failed for 30 days, the existence of the cron entry provides little comfort.

---

# 143. Idempotency Introduction

You will encounter the word `idempotent` frequently in DevOps.

At a beginner level, think:

```text
Can I run this automation repeatedly without causing unintended damage?
```

For example:

```bash
mkdir -p "$backup_dir"
```

is useful because repeated execution does not fail merely because the directory already exists.

Compare:

```bash
mkdir "$backup_dir"
```

which may fail if the directory already exists.

Tools such as Ansible, Terraform, Kubernetes, and configuration-management systems build much more deeply on concepts of desired state and repeatable operations.

---

# 144. Automation Should Be Observable

Imagine:

```text
Cron ran backup
```

but nobody knows whether it succeeded.

Better:

```text
Cron
 |
 v
Backup script
 |
 +--> Logs
 |
 +--> Exit status
 |
 +--> Backup artifact
 |
 +--> Verification
```

Production automation may additionally send:

* Monitoring metrics.
* Alerts.
* Notifications.
* Audit events.

The principle is simple:

```text
Automated tasks should leave evidence.
```

---

# 145. Automation Should Fail Clearly

Bad:

```bash
tar -czf backup.tar.gz source/
echo "Backup complete"
```

If `tar` fails, the script may still print:

```text
Backup complete
```

Better:

```bash
if tar -czf backup.tar.gz source/; then
    echo "Backup complete"
else
    echo "Backup failed" >&2
    exit 1
fi
```

The caller should be able to distinguish success from failure.

---

# 146. Automation Should Fail Safely

Consider a cleanup script.

Bad design:

```text
Variable missing
     |
     v
Continue anyway
     |
     v
Delete unexpected files
```

Better:

```text
Variable missing
     |
     v
Validation fails
     |
     v
Stop
     |
     v
Log error
```

For destructive automation, failing safely is usually better than guessing.

---

# 147. Automation Security

Scheduled jobs deserve security attention because they may run unattended.

Check:

* Who owns the script?
* Who can modify it?
* Which user executes it?
* Does it need root?
* Can another user modify files it executes?
* Does it contain secrets?
* Where are logs written?
* Can logs expose secrets?
* Are backups protected?
* Can input paths be manipulated?

Imagine root runs:

```text
/root/scripts/backup.sh
```

but an untrusted user can modify that script.

That creates a serious privilege problem.

Protect automation code.

---

# 148. Check Script Ownership and Permissions

Run:

```bash
ls -l /path/to/backup.sh
```

Ask:

```text
Who owns it?

Who can write to it?

Who executes it?
```

A script executed with elevated privileges should not normally be writable by untrusted users.

---

# 149. Do Not Put Passwords Directly in Crontab

Avoid entries such as:

```text
* * * * * command --password SuperSecretPassword
```

Why?

Secrets may become visible through:

* Configuration files.
* Process inspection.
* Logs.
* Backups.
* Administrative access.

Use the authentication and secret-management mechanisms appropriate to the application.

---

# 150. Cron vs systemd Timers

Modern Linux systems using systemd also provide:

```text
systemd timers
```

for scheduling.

They can offer features such as:

* Integration with systemd services.
* Dependency management.
* Centralized journal logging.
* Persistent timer behavior.
* More detailed scheduling controls.

However, cron remains important and widely encountered.

For this beginner curriculum, mastering cron first gives you a strong foundation.

You can later learn systemd timers as an intermediate administration topic.

---

# 151. Backup and Automation Project

Your final project is to create an automated backup system for a fictional application.

Scenario:

You administer a Linux server running:

```text
/opt/veriqta-app/
```

The application has:

```text
/opt/veriqta-app/
├── config/
├── scripts/
└── data/
```

Your job is to:

1. Back it up.
2. Create timestamped archives.
3. Compress them.
4. Verify them.
5. Protect their permissions.
6. Retain backups for a defined period.
7. Log each execution.
8. Schedule the process.
9. Test failure conditions.
10. Perform a restoration.

For a non-root beginner lab, use a directory inside your home folder instead of `/opt`.

---

# 152. Project Step 1: Create the Application

Use:

```bash
mkdir -p "$HOME/veriqta-app/config"
mkdir -p "$HOME/veriqta-app/scripts"
mkdir -p "$HOME/veriqta-app/data"
```

Create:

```bash
echo "port=8080" > "$HOME/veriqta-app/config/app.conf"
```

Create:

```bash
echo "production=true" >> "$HOME/veriqta-app/config/app.conf"
```

Create:

```bash
echo "Customer application data" \
    > "$HOME/veriqta-app/data/application.data"
```

Create:

```bash
echo '#!/bin/bash' > "$HOME/veriqta-app/scripts/start.sh"
echo 'echo "Starting application"' \
    >> "$HOME/veriqta-app/scripts/start.sh"
```

Make executable:

```bash
chmod +x "$HOME/veriqta-app/scripts/start.sh"
```

---

# 153. Project Step 2: Inspect the Application

Run:

```bash
find "$HOME/veriqta-app" -type f
```

Then:

```bash
ls -l "$HOME/veriqta-app/config"
ls -l "$HOME/veriqta-app/scripts"
ls -l "$HOME/veriqta-app/data"
```

You should understand exactly what will be backed up before automating it.

---

# 154. Project Step 3: Create Backup Infrastructure

Run:

```bash
mkdir -p "$HOME/veriqta-backup/scripts"
mkdir -p "$HOME/veriqta-backup/backups"
mkdir -p "$HOME/veriqta-backup/logs"
mkdir -p "$HOME/veriqta-backup/restore-tests"
```

Your structure becomes:

```text
~/veriqta-app/
├── config/
├── scripts/
└── data/

~/veriqta-backup/
├── scripts/
├── backups/
├── logs/
└── restore-tests/
```

Notice that the backup directory is outside the application source.

This prevents accidentally archiving previous backups as application data.

---

# 155. Project Step 4: Create the Script

Create:

```bash
nano "$HOME/veriqta-backup/scripts/backup-veriqta.sh"
```

Adapt the backup script from this lesson.

Use:

```bash
source_dir="$HOME/veriqta-app"
backup_dir="$HOME/veriqta-backup/backups"
```

Choose a retention period.

For example:

```bash
retention_days=7
```

---

# 156. Project Step 5: Test Manually

Before cron:

```bash
bash -n "$HOME/veriqta-backup/scripts/backup-veriqta.sh"
```

Then:

```bash
chmod +x "$HOME/veriqta-backup/scripts/backup-veriqta.sh"
```

Run:

```bash
"$HOME/veriqta-backup/scripts/backup-veriqta.sh"
```

Check:

```bash
echo $?
```

Do not continue until the manual execution behaves correctly.

---

# 157. Project Step 6: Verify the Backup

Run:

```bash
ls -lh "$HOME/veriqta-backup/backups"
```

Then inspect the exact archive:

```bash
tar -tzf "/path/to/generated/archive.tar.gz"
```

Confirm that you see:

```text
veriqta-app/
veriqta-app/config/
veriqta-app/config/app.conf
veriqta-app/scripts/
veriqta-app/scripts/start.sh
veriqta-app/data/
veriqta-app/data/application.data
```

Your exact paths depend on how you use `tar -C`.

---

# 158. Project Step 7: Restore

Choose a clean test directory:

```bash
mkdir -p "$HOME/veriqta-backup/restore-tests/test1"
```

Extract:

```bash
tar -xzf "/path/to/generated/archive.tar.gz" \
    -C "$HOME/veriqta-backup/restore-tests/test1"
```

Inspect:

```bash
find "$HOME/veriqta-backup/restore-tests/test1" -type f
```

Read:

```bash
cat "$HOME/veriqta-backup/restore-tests/test1/veriqta-app/config/app.conf"
```

Compare with:

```bash
cat "$HOME/veriqta-app/config/app.conf"
```

---

# 159. Project Step 8: Test a Failure

Temporarily change:

```bash
source_dir="$HOME/veriqta-app"
```

to:

```bash
source_dir="$HOME/directory-that-does-not-exist"
```

Run:

```bash
"$HOME/veriqta-backup/scripts/backup-veriqta.sh"
```

The script should:

* Detect the missing source.
* Print an error.
* Stop.
* Return non-zero.
* Not claim successful completion.

Check:

```bash
echo $?
```

Then restore the correct source path.

---

# 160. Project Step 9: Test Permission Behavior

Create a test file with restrictive permissions only in your lab if you understand the ownership implications.

Observe whether the backup user can read it.

If `tar` reports permission errors:

1. Identify the exact file.
2. Inspect ownership.
3. Inspect permissions.
4. Determine what access should exist.
5. Fix the specific problem.
6. Retry.
7. Verify the archive.

Do not use `chmod -R 777` as a troubleshooting shortcut.

---

# 161. Project Step 10: Add Logging

Create:

```bash
mkdir -p "$HOME/veriqta-backup/logs"
```

Run manually with:

```bash
"$HOME/veriqta-backup/scripts/backup-veriqta.sh" \
    >> "$HOME/veriqta-backup/logs/backup.log" 2>&1
```

Inspect:

```bash
cat "$HOME/veriqta-backup/logs/backup.log"
```

Confirm that both success and failure messages are useful.

---

# 162. Project Step 11: Test Through Cron

Find your actual home directory:

```bash
echo "$HOME"
```

Use the absolute result in your crontab.

Temporarily create an every-minute job:

```text
* * * * * /absolute/home/path/veriqta-backup/scripts/backup-veriqta.sh >> /absolute/home/path/veriqta-backup/logs/backup.log 2>&1
```

Then:

```bash
crontab -l
```

Wait for execution.

Inspect:

```bash
ls -lh "$HOME/veriqta-backup/backups"
```

and:

```bash
tail -n 50 "$HOME/veriqta-backup/logs/backup.log"
```

---

# 163. Project Step 12: Change to the Real Schedule

After successful cron testing, replace the temporary schedule.

For example:

```text
0 2 * * * /absolute/home/path/veriqta-backup/scripts/backup-veriqta.sh >> /absolute/home/path/veriqta-backup/logs/backup.log 2>&1
```

This schedules the backup daily at 02:00 according to the server's applicable cron time configuration.

Verify:

```bash
crontab -l
```

---

# 164. Project Step 13: Verify the Entire Workflow

Your final architecture should look like:

```text
Application files
~/veriqta-app/
       |
       v
backup-veriqta.sh
       |
       +---- Validate source
       |
       +---- Validate tools
       |
       +---- Generate timestamp
       |
       +---- tar archive
       |
       +---- gzip compression
       |
       +---- Verify archive
       |
       +---- Restrict permissions
       |
       +---- Apply retention
       |
       +---- Return exit status
       |
       v
~/veriqta-backup/backups/
       ^
       |
      Cron
       |
       v
Scheduled execution
       |
       +---- stdout
       |
       +---- stderr
              |
              v
        backup.log
```

You should understand every arrow in this diagram.

---

# 165. What You Should Now Understand

Before moving to Part 19, you should understand:

* Why system administration is automated.
* The difference between scripting a task and scheduling it.
* What scheduled tasks are.
* What cron is.
* What a cron job is.
* What the cron daemon does.
* Why the service may be named `cron` or `crond`.
* How to check whether `crontab` exists.
* How to inspect the cron service.
* What a crontab is.
* How user crontabs work.
* Why cron jobs inherit the permissions of their user.
* How `crontab -l` works.
* How `crontab -e` works.
* Why system cron files differ from user crontabs.
* The five cron scheduling fields.
* Minute.
* Hour.
* Day of month.
* Month.
* Day of week.
* What `*` means.
* How step values such as `*/5` work.
* How ranges such as `1-5` work.
* How lists such as `8,12,18` work.
* How to schedule hourly jobs.
* How to schedule daily jobs.
* How to schedule weekday jobs.
* How to schedule jobs at specific times.
* Why cron's environment differs from your interactive shell.
* Why `PATH` causes cron failures.
* Why relative paths cause cron failures.
* Why absolute paths are useful.
* Why cron output should be redirected or otherwise handled.
* The difference between `>` and `>>`.
* Why scheduled jobs need logs.
* Why cron log files need retention too.
* How to test cron safely.
* Why temporary every-minute jobs should be removed.
* How to verify an installed crontab.
* How to investigate cron service logs.
* Why server time zones matter.
* What an archive is.
* What compression is.
* Why archiving and compression are different concepts.
* What `tar` does.
* What `tar -c` means.
* What `tar -x` means.
* What `tar -t` means.
* What `tar -f` means.
* What `tar -v` means.
* What `tar -z` means.
* How to create a `.tar` archive.
* How to inspect it.
* How to extract it.
* How to create `.tar.gz`.
* How to extract `.tar.gz`.
* What gzip does.
* Why compression does not always dramatically reduce file size.
* What ZIP is.
* How `zip -r` works.
* How `unzip -l` works.
* How `unzip` extracts files.
* The difference between `.tar`, `.gz`, `.tar.gz`, and `.zip`.
* What a backup is.
* Why a copy is not automatically a strong backup.
* How to decide what needs backup.
* Why reproducible data may be handled differently.
* Why databases require application-aware backup strategies.
* Why backup location matters.
* Why timestamped filenames are useful.
* How to generate timestamps.
* How to create backup directories.
* How to create a compressed file backup.
* How to verify an archive.
* Why existence alone does not prove backup validity.
* Why restore testing matters.
* Why backup permissions matter.
* Why backups can contain sensitive information.
* What retention means.
* Why unlimited retention consumes storage.
* How `find` can identify old backup files.
* Why `find -mtime` needs careful testing around age boundaries.
* Why you should inspect matches before deleting them.
* Why retention rules should be narrowly scoped.
* How to create an automated Bash backup script.
* How to validate dependencies.
* How to validate source directories.
* How to handle archive failures.
* How to verify archives.
* How to protect backup permissions.
* How to implement basic retention.
* Why `-print0` can improve pathname handling.
* Why `rm --` is useful.
* How to schedule a backup.
* How to troubleshoot a script that works manually but fails in cron.
* How to troubleshoot permissions.
* How to troubleshoot missing commands.
* How to troubleshoot wrong paths.
* How to troubleshoot wrong schedules.
* How to troubleshoot disk-full errors.
* Why a backup directory should not accidentally be included in itself.
* Why compressed data is not encrypted data.
* The basic 3-2-1 backup principle.
* Why separate failure domains matter.
* What RPO means.
* What RTO means.
* Why automation should be observable.
* Why automation should fail clearly.
* Why automation should fail safely.
* Why scheduled scripts need secure ownership and permissions.
* Why passwords should not be embedded in cron commands.
* What idempotency means at a beginner level.
* That systemd timers are another scheduling mechanism you may encounter.

---

# 166. Hands-On Practice

Complete these tasks yourself.

1. Check whether `crontab` exists.
2. Check whether your system uses `cron` or `crond`.
3. Check the scheduler service status.
4. Run `crontab -l`.
5. Open `crontab -e`.
6. Explain each of the five cron fields.
7. Write a schedule for every hour.
8. Write a schedule for every 15 minutes.
9. Write a schedule for every day at midnight.
10. Write a schedule for every day at 02:30.
11. Write a schedule for Monday at 08:00.
12. Write a schedule for weekdays at 18:00.
13. Write a schedule for 08:00, 12:00, and 18:00.
14. Create a cron test directory.
15. Schedule `date` temporarily.
16. Redirect its output to a file.
17. Confirm the file changes.
18. Remove the temporary job.
19. Check your server's current time.
20. Check its time zone.
21. Create an archive practice directory.
22. Create three files.
23. Create a `.tar` archive.
24. List its contents.
25. Extract it.
26. Verify the extracted files.
27. Compress a tar archive with gzip.
28. Create `.tar.gz` directly using `tar`.
29. List the compressed archive.
30. Extract it into a separate directory.
31. Check whether `zip` is installed.
32. Create a ZIP archive.
33. List its contents.
34. Extract it.
35. Explain archive vs compression.
36. Create a backup source directory.
37. Create realistic configuration files.
38. Create a backup destination.
39. Generate a timestamp.
40. Use the timestamp in a filename.
41. Create a compressed backup.
42. Verify that it exists.
43. List its archive contents.
44. Extract it into a restore directory.
45. Compare restored files with source files.
46. Check backup permissions.
47. Set restrictive permissions.
48. Create several test backup files.
49. Use `find` to inspect old files.
50. Do not delete anything yet.
51. Verify your retention expression.
52. Create a backup Bash script.
53. Add a shebang.
54. Add configuration variables.
55. Add logging.
56. Add dependency validation.
57. Add source validation.
58. Add destination creation.
59. Add timestamp generation.
60. Add archive creation.
61. Add archive verification.
62. Add restrictive permissions.
63. Add retention.
64. Add a `main` function.
65. Return useful exit codes.
66. Check syntax with `bash -n`.
67. Run manually.
68. Check `$?`.
69. Inspect the backup.
70. Restore it.
71. Intentionally provide a missing source path.
72. Verify that the script fails.
73. Check its exit status.
74. Repair the source path.
75. Run again.
76. Redirect script output to a log.
77. Inspect the log.
78. Temporarily schedule the script every minute.
79. Confirm cron actually runs it.
80. Inspect cron-generated backups.
81. Inspect cron-generated logs.
82. Remove the every-minute schedule.
83. Create the intended daily schedule.
84. Verify with `crontab -l`.
85. Intentionally create a relative-path problem in a disposable copy.
86. Observe the failure.
87. Repair it using absolute paths.
88. Temporarily remove execute permission.
89. Observe direct-execution failure.
90. Restore execute permission.
91. Investigate a simulated `PATH` problem.
92. Find a command using `command -v`.
93. Use its absolute path.
94. Check `df -h`.
95. Check `df -i`.
96. Explain why both matter to backups.
97. Explain why local-only backups have limitations.
98. Explain RPO.
99. Explain RTO.
100. Explain how you would prove that your backup system actually works.

---

# 167. Beginner Knowledge Check

1. Why automate system administration?
2. What is the difference between a Bash script and scheduled automation?
3. What is cron?
4. What is a cron job?
5. What does the cron daemon do?
6. Why might the service be called `cron` on one distribution and `crond` on another?
7. What does `crontab -l` do?
8. What does `crontab -e` do?
9. Are all users' crontabs the same?
10. Which permissions does a user's cron job normally have?
11. How many scheduling fields come before the command in a standard user cron entry?
12. What is the first field?
13. What is the second?
14. What is the third?
15. What is the fourth?
16. What is the fifth?
17. What does `*` mean?
18. What does `*/5` mean in the minute field?
19. What does `0 2 * * *` mean?
20. What does `30 2 * * *` mean?
21. What does `0 8 * * 1` mean?
22. What does `1-5` mean in the weekday field?
23. Why can a script work manually but fail under cron?
24. Why is `PATH` important?
25. Why are absolute paths useful?
26. Why can relative paths break scheduled scripts?
27. What happens to cron stdout and stderr?
28. Why should scheduled jobs have useful logging?
29. Why can `>>` be preferable to `>` for a recurring log?
30. What new problem can an endlessly appended log create?
31. Why should cron be tested before scheduling a critical job?
32. Why is an every-minute schedule useful during testing?
33. Why should that test schedule be removed afterward?
34. Why does the server time zone matter?
35. What is an archive?
36. What is compression?
37. Are archiving and compression identical?
38. What does `tar` do?
39. What does `tar -c` mean?
40. What does `tar -x` mean?
41. What does `tar -t` mean?
42. What does `tar -f` mean?
43. What does `tar -z` mean?
44. What does `tar -czf` commonly create?
45. How do you list `.tar.gz` contents?
46. How do you extract a `.tar.gz` archive?
47. What does gzip do?
48. Does gzip encrypt data?
49. What is ZIP?
50. How do you list ZIP contents?
51. What is a backup?
52. Why is a second copy on the same disk limited protection?
53. What kinds of files commonly require backup?
54. Why might temporary cache files not need backup?
55. Why do databases require special consideration?
56. Why are timestamps useful in backup filenames?
57. How can `date` generate a timestamp?
58. Why should backup creation be verified?
59. Does archive existence prove the backup is valid?
60. Why should you test restoration?
61. Why can backups be security-sensitive?
62. What is retention?
63. Why is retention necessary?
64. How can `find` help with retention?
65. Why should you inspect `find` results before deletion?
66. Why should a retention expression be narrowly scoped?
67. Why can `find -mtime` age boundaries require careful testing?
68. Why validate required commands?
69. Why validate the source directory?
70. Why validate the destination?
71. Why should a failed `tar` command stop a backup workflow?
72. What does `tar -tzf` help verify?
73. Why might backup permissions be `600`?
74. What does `600` mean?
75. Why use `-print0` with arbitrary filenames?
76. Why can `rm --` be useful?
77. Why should backup retention be tested before deletion is enabled?
78. What happens if the backup directory is inside the source being backed up?
79. Why can backups grow unexpectedly?
80. How do you investigate `Permission denied` during backup?
81. Why is `chmod -R 777` usually a poor fix?
82. What should you check when you see `No space left on device`?
83. Why should you check both `df -h` and `df -i`?
84. What can cause an archive to become corrupt?
85. Why is a restore test stronger than simply listing an archive?
86. What does compressed mean?
87. What does encrypted mean?
88. What is the basic 3-2-1 backup principle?
89. What is a failure domain?
90. What is RPO?
91. What is RTO?
92. How are RPO and RTO different?
93. What does idempotency mean at a beginner level?
94. Why should automated tasks leave evidence?
95. Why should automation fail clearly?
96. Why should destructive automation fail safely?
97. Why does script ownership matter?
98. Why should cron jobs avoid unnecessary root privileges?
99. Why should secrets not be placed directly in cron commands?
100. What evidence would convince you that an automated backup system actually works?

---

# 168. Knowledge Check Answers

1. To perform repeatable administrative work consistently without depending on a person to manually initiate every execution.

2. A script automates the procedure. A scheduler such as cron automates when the procedure starts.

3. A traditional time-based Unix and Linux job scheduler.

4. A command or script configured to execute according to a cron schedule.

5. It processes schedules and launches jobs at the appropriate times.

6. Linux distributions package and name services differently.

7. Lists the current user's cron jobs.

8. Opens the current user's crontab for editing.

9. No. Users can have separate crontabs.

10. Normally the permissions of the user whose crontab runs the job.

11. Five.

12. Minute.

13. Hour.

14. Day of month.

15. Month.

16. Day of week.

17. Every allowed value for that field.

18. Every five minutes.

19. Every day at 02:00.

20. Every day at 02:30.

21. Every Monday at 08:00 under the common weekday numbering.

22. Monday through Friday under that numbering.

23. Cron normally provides a more limited non-interactive environment.

24. It determines where commands are searched for.

25. They reduce dependence on the current environment and working directory.

26. Cron may start the job from a directory different from the one you use manually.

27. They need somewhere to go, such as cron's configured mail handling or explicit redirection. For reliable beginner automation, handle them deliberately.

28. Logs provide evidence of execution and failures.

29. `>>` appends instead of overwriting previous information.

30. The log can grow indefinitely and consume storage.

31. A valid-looking schedule does not prove the script will work in cron's environment.

32. It gives rapid feedback while testing.

33. It can create excessive executions, files, load, and logs.

34. Cron executes according to the server's applicable clock and scheduler configuration, which may differ from your expected local time.

35. A file that packages one or more files and directories together.

36. Encoding data in a representation that often uses less storage.

37. No.

38. It creates, lists, and extracts archives, among other operations.

39. Create.

40. Extract.

41. List archive contents.

42. Specify the archive file.

43. Use gzip compression.

44. A gzip-compressed tar archive.

45. `tar -tzf archive.tar.gz`.

46. `tar -xzf archive.tar.gz`.

47. Compresses data using gzip.

48. No.

49. A widely used archive/compression format.

50. `unzip -l file.zip`.

51. A separate copy maintained for recovery.

52. One disk failure can destroy both original and copy.

53. Important application data, configuration, user-generated data, and other information that cannot easily be recreated.

54. It may be safely regenerated and therefore may not justify backup storage.

55. Active databases have consistency requirements and database-specific backup procedures.

56. They prevent every backup from simply overwriting the previous one and help identify creation time.

57. For example, `date '+%Y-%m-%d_%H%M%S'`.

58. Commands can fail, archives can be incomplete, and the wrong data can be selected.

59. No.

60. Recovery is the actual purpose of a backup.

61. They may contain sensitive application data, configurations, credentials, certificates, or user information.

62. A policy describing how long backup copies are kept.

63. Without it, backup storage may grow indefinitely.

64. It can select backup files based on directory, name, type, age, and other attributes.

65. To ensure the selection matches only files you intend to remove.

66. Broad deletion rules can remove unrelated files.

67. `-mtime` is based on completed 24-hour periods, so its exact boundary may differ from a beginner's intuitive reading of "older than N days."

68. To avoid running an incomplete workflow when essential utilities are unavailable.

69. To prevent creating a misleading backup when the expected source does not exist.

70. The script must know that it can create and write backup data there.

71. Continuing after archive failure could produce false success messages or perform retention without a valid new backup.

72. It checks whether `tar` can read and list the archive structure.

73. To restrict access to the owning user when that matches the security requirement.

74. Owner can read and write. Group and others receive no permissions.

75. It handles filenames safely even when they contain spaces, newlines, or other unusual characters.

76. It marks the end of command options so a pathname beginning with `-` is not interpreted as an option.

77. Retention performs destructive operations.

78. New backups can accidentally include older backups, causing rapid recursive growth.

79. Failed retention, overly frequent schedules, incorrect source selection, or backing up previous backups can all cause growth.

80. Identify the exact path, then inspect the executing user, ownership, file permissions, and directory permissions.

81. It grants excessive access instead of fixing the actual permission problem.

82. Disk capacity and inode availability.

83. Either storage blocks or inodes can be exhausted.

84. Interrupted creation, full storage, truncation, corruption, or incomplete transfer.

85. It demonstrates that data can actually be extracted, not merely that archive metadata can be read.

86. Data has been transformed to reduce or alter its storage representation.

87. Data has been transformed cryptographically to protect confidentiality from unauthorized readers.

88. Keep three copies, on two types of storage/media, with one copy off-site.

89. A component or location that can fail independently. Separating copies across failure domains reduces the chance that one event destroys everything.

90. Recovery Point Objective, the acceptable amount of data loss expressed in time.

91. Recovery Time Objective, the target amount of time for restoring service after disruption.

92. RPO concerns how much recent data may be lost. RTO concerns how long recovery may take.

93. A repeatable operation should be designed so repeated execution does not create unintended damage or inconsistent results.

94. Administrators need to know whether the task ran, what happened, and whether it succeeded.

95. Failures should be detectable rather than disguised as success.

96. A failed assumption should stop dangerous actions rather than cause the automation to guess.

97. A user who can modify a privileged scheduled script may be able to influence what privileged code executes.

98. Least privilege limits the potential impact of mistakes and compromise.

99. Secrets may be exposed through configuration, process information, logs, backups, or administrative access.

100. Successful scheduled executions, useful logs, correct exit statuses, verified archive contents, protected backup files, and, most importantly, successful restore tests.

# 169. Final Mental Model

Do not think of Linux automation as:

```text
Write script
+
Put it in cron
=
Done
```

Think:

```text
UNDERSTAND THE TASK
        |
        v
PERFORM IT MANUALLY
        |
        v
IDENTIFY FAILURE CONDITIONS
        |
        v
WRITE SCRIPT
        |
        v
VALIDATE INPUTS
        |
        v
HANDLE ERRORS
        |
        v
TEST SUCCESS PATH
        |
        v
TEST FAILURE PATH
        |
        v
SCHEDULE
        |
        v
CAPTURE OUTPUT
        |
        v
VERIFY EXECUTION
        |
        v
MONITOR
```

For backups:

```text
IDENTIFY IMPORTANT DATA
        |
        v
SELECT BACKUP METHOD
        |
        v
CREATE BACKUP
        |
        v
VERIFY ARCHIVE
        |
        v
PROTECT BACKUP
        |
        v
STORE SAFELY
        |
        v
APPLY RETENTION
        |
        v
TEST RESTORE
        |
        v
REPEAT AND MONITOR
```

The most important lesson in Part 18 is this:

```text
A cron entry is not proof that a task ran.

A backup file is not proof that the right data was backed up.

A compressed archive is not an encrypted archive.

A successful backup command is not proof that recovery will work.

Automation is only trustworthy when you can observe it, verify it, troubleshoot it, and recover from it.
```

That is the foundation you need before moving from beginner Linux administration into more serious production automation.
