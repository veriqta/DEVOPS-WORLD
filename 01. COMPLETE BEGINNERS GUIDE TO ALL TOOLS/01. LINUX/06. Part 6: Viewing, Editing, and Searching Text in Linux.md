# Part 6: Viewing, Editing, and Searching Text in Linux

In Part 5, you learned how Linux controls access to files and directories using users, groups, ownership, permissions, `chmod`, `chown`, `chgrp`, `umask`, and related access-control mechanisms.

Now you will learn how to work with the contents of those files.

This is one of the most important Linux skills for DevOps, Cloud, SRE, Platform Engineering, and system administration.

A large amount of Linux work involves text.

You will constantly need to:

* Read configuration files.
* Inspect logs.
* Edit settings.
* Create scripts.
* Search thousands of lines for an error.
* Extract one field from command output.
* Sort and count information.
* Transform text.
* Follow logs while an application is running.
* Combine several commands into a troubleshooting workflow.

By the end of this part, you should be able to open an unfamiliar Linux server and confidently inspect, edit, search, filter, and process text without depending on a graphical editor.

---

# 1. Why Text Files Matter in Linux

Linux relies heavily on plain-text files.

A plain-text file contains characters that humans and programs can read without requiring a specialized binary format.

For example:

```text
server_port=8080
environment=production
log_level=INFO
```

Linux uses text extensively because text files are:

* Easy to inspect.
* Easy to edit.
* Easy to search.
* Easy to compare.
* Easy to automate.
* Easy for programs to parse.
* Easy to store in Git.
* Convenient for remote server administration.

You will encounter text files almost everywhere.

---

# 2. Three Important Types of Linux Text Files

As a beginner, you will frequently work with three broad categories:

```text
Configuration files
Log files
Scripts
```

Each serves a different purpose.

---

# 3. Configuration Files

Configuration files tell applications and services how they should behave.

Examples include:

```text
/etc/ssh/sshd_config
/etc/hosts
/etc/fstab
/etc/nginx/nginx.conf
```

Not every file exists on every Linux system. Installed software and distribution choices determine what is available.

A configuration might contain:

```text
Port 22
PermitRootLogin no
PasswordAuthentication no
```

or:

```text
server {
    listen 80;
    server_name example.com;
}
```

Configuration files answer questions such as:

```text
Which port should the application use?

Where should logs be stored?

Which users are allowed?

Which network interface should be used?

Which features are enabled?
```

Changing configuration can change application behavior.

That is why configuration editing must be done carefully.

---

# 4. Configuration Files Can Break Services

Imagine you edit:

```text
/etc/nginx/nginx.conf
```

and accidentally remove a closing brace.

The service may refuse to reload.

The important lesson is:

```text
Edit
  |
  v
Validate
  |
  v
Apply or reload
  |
  v
Verify
```

not:

```text
Edit
  |
  v
Assume everything works
```

Later parts will teach service management in detail.

For now, remember that editing a file can have consequences for software using that file.

---

# 5. Log Files

Logs record events.

An application may write information such as:

```text
2026-08-26 10:14:22 INFO Application started
2026-08-26 10:14:27 INFO Connected to database
2026-08-26 10:15:02 WARN Response time exceeded 2 seconds
2026-08-26 10:15:18 ERROR Database connection failed
```

Logs help answer:

```text
What happened?

When did it happen?

Which component failed?

What error was reported?

Did the application restart?

Was authentication rejected?

Did the request reach the server?
```

Logs are evidence.

During troubleshooting, they are often one of the first places engineers investigate.

---

# 6. Scripts

A script contains instructions that another program interprets.

A simple Bash script might contain:

```bash
#!/bin/bash

echo "Starting deployment check"

date
hostname
whoami
```

Instead of manually typing the same sequence repeatedly, you can store commands in a script.

Scripts are used for:

* Automation.
* Deployment.
* Backups.
* Health checks.
* Server configuration.
* Maintenance.
* CI/CD.
* Monitoring.
* Troubleshooting.

Later in this Linux series, you will learn shell scripting properly.

For now, scripts are another important reason you need strong text-editing skills.

---

# 7. Our Text-Processing Mental Model

Most of this part can be understood using one simple model:

```text
Text
 |
 v
View
 |
 v
Search
 |
 v
Filter
 |
 v
Transform
 |
 v
Extract
 |
 v
Summarize
 |
 v
Use the result
```

Different Linux commands specialize in different parts.

For example:

```text
cat     -> display
less    -> inspect
grep    -> search/filter
sort    -> sort
uniq    -> identify adjacent duplicates
cut     -> extract fields
tr      -> translate characters
sed     -> stream editing/transformation
awk     -> field-oriented processing
wc      -> count
```

Do not treat these as unrelated commands.

Together they form a text-processing toolkit.

---

# 8. Create a Safe Practice Environment

Use your home directory for the exercises in this part.

Create:

```bash
mkdir -p ~/text-lab/{configs,logs,scripts,data}
```

Move into it:

```bash
cd ~/text-lab
```

Verify:

```bash
pwd
```

You should see something similar to:

```text
/home/yourusername/text-lab
```

Check the directories:

```bash
ls
```

Expected:

```text
configs  data  logs  scripts
```

We will use this lab throughout the article.

---

# 9. Create a Practice Log

Run:

```bash
printf '%s\n' \
'2026-08-26 10:00:01 INFO application started' \
'2026-08-26 10:00:05 INFO database connection established' \
'2026-08-26 10:01:17 WARN response time high' \
'2026-08-26 10:02:03 ERROR database connection lost' \
'2026-08-26 10:02:04 ERROR request failed' \
'2026-08-26 10:02:30 INFO database connection restored' \
> logs/application.log
```

You do not need to understand the entire command yet.

We will explain `printf` and redirection shortly.

---

# 10. Viewing Text With `cat`

`cat` originally refers to concatenating files.

Its simplest beginner use is displaying file contents.

Syntax:

```bash
cat FILE
```

Try:

```bash
cat logs/application.log
```

You should see the complete file.

---

# 11. What `cat` Actually Does

Conceptually:

```text
File
 |
 v
cat
 |
 v
Standard output
 |
 v
Terminal
```

`cat` reads the file and writes its contents to standard output.

For a short file, this is convenient.

---

# 12. When `cat` Is Useful

`cat` works well when:

* The file is short.
* You want the entire contents immediately.
* You are combining files.
* You want file contents to become input for another operation.

Example:

```bash
cat configs/app.conf
```

But imagine a log containing 200,000 lines.

Running:

```bash
cat huge.log
```

can flood your terminal.

For large files, use a pager such as `less`.

---

# 13. Displaying Multiple Files With `cat`

Create:

```bash
echo "frontend-server" > data/server1.txt
echo "database-server" > data/server2.txt
```

Now:

```bash
cat data/server1.txt data/server2.txt
```

Output:

```text
frontend-server
database-server
```

This demonstrates the original idea behind `cat`.

It can concatenate multiple inputs into one output stream.

---

# 14. Viewing Files With `less`

`less` is one of the most useful commands for inspecting large text files.

Syntax:

```bash
less FILE
```

Try:

```bash
less logs/application.log
```

Unlike `cat`, `less` opens an interactive pager.

It allows you to move through the file rather than dumping everything into the terminal.

---

# 15. Basic `less` Navigation

Inside `less`, useful keys include:

```text
Down Arrow    Move down
Up Arrow      Move up
Space         Move forward one screen
b             Move backward one screen
g             Go to beginning
G             Go to end
q             Quit
```

You can also search.

Type:

```text
/ERROR
```

and press Enter.

`less` searches forward for `ERROR`.

Press:

```text
n
```

to move to the next match.

Use:

```text
N
```

for the previous match.

Then:

```text
q
```

to exit.

---

# 16. Why `less` Matters in Production

Imagine:

```text
/var/log/application.log
```

contains 500,000 lines.

You probably do not want:

```bash
cat /var/log/application.log
```

Instead:

```bash
less /var/log/application.log
```

lets you inspect it safely and interactively.

You can search for:

```text
/ERROR
```

or:

```text
/timeout
```

without loading the whole file onto your terminal screen at once.

---

# 17. `more`

`more` is another pager.

Usage:

```bash
more logs/application.log
```

Historically, `more` was commonly used to move forward through text one screen at a time.

`less` generally provides richer navigation.

A common Linux joke is:

```text
less is more
```

For modern Linux work, learn `less` well.

Still recognize `more`, because you may encounter it on systems, in scripts, or in older documentation.

---

# 18. Viewing the Beginning With `head`

`head` displays the beginning of a file.

Syntax:

```bash
head FILE
```

Example:

```bash
head logs/application.log
```

By default, GNU `head` normally displays the first 10 lines.

Our file contains fewer lines, so you will see all of them.

---

# 19. Choosing the Number of Lines With `head`

Use:

```bash
head -n 3 logs/application.log
```

Output:

```text
2026-08-26 10:00:01 INFO application started
2026-08-26 10:00:05 INFO database connection established
2026-08-26 10:01:17 WARN response time high
```

Breakdown:

```text
head
Display beginning

-n
Specify number of lines

3
Number requested

logs/application.log
Target file
```

A shorter common syntax is:

```bash
head -3 logs/application.log
```

For beginners, `-n 3` makes the meaning clearer.

---

# 20. When `head` Is Useful

Suppose a CSV file contains:

```text
500,000 rows
```

You want to inspect its structure.

Instead of opening the entire file:

```bash
head -n 5 data.csv
```

You can quickly see:

* Column headings.
* Delimiters.
* Example records.
* Whether the file looks correct.

---

# 21. Viewing the End With `tail`

`tail` displays the end of a file.

Syntax:

```bash
tail FILE
```

By default, GNU `tail` normally shows the last 10 lines.

Try:

```bash
tail logs/application.log
```

For logs, the end is particularly useful because the newest events are commonly appended there.

---

# 22. Choosing Lines With `tail`

Use:

```bash
tail -n 3 logs/application.log
```

Expected:

```text
2026-08-26 10:02:03 ERROR database connection lost
2026-08-26 10:02:04 ERROR request failed
2026-08-26 10:02:30 INFO database connection restored
```

This answers:

```text
What happened most recently?
```

---

# 23. Following Logs With `tail -f`

One of the most important log commands is:

```bash
tail -f FILE
```

The:

```text
-f
```

means follow.

Run:

```bash
tail -f logs/application.log
```

The command shows the end of the file and waits.

If new lines are appended, they appear in your terminal.

---

# 24. Understanding `tail -f`

Conceptually:

```text
Application
    |
    | writes new log entry
    v
application.log
    |
    v
tail -f
    |
    v
Your terminal
```

This lets you watch logs in near real time.

---

# 25. Testing `tail -f`

Open two terminal windows.

In Terminal 1:

```bash
cd ~/text-lab
tail -f logs/application.log
```

Leave it running.

In Terminal 2:

```bash
echo "2026-08-26 10:05:00 ERROR API timeout" >> ~/text-lab/logs/application.log
```

Terminal 1 should immediately show:

```text
2026-08-26 10:05:00 ERROR API timeout
```

Stop `tail -f` with:

```text
Ctrl+C
```

`Ctrl+C` sends an interrupt signal to the foreground command.

We will study Linux processes and signals properly later.

---

# 26. Why `tail -f` Matters

Imagine you are troubleshooting a web application.

You restart or test the application and want to observe what happens.

You might follow its log:

```bash
tail -f /var/log/myapp/application.log
```

Then reproduce the problem.

As the request occurs, new log entries appear.

This creates a useful workflow:

```text
Follow logs
   |
   v
Reproduce problem
   |
   v
Observe new events
   |
   v
Identify error
```

---

# 27. A Limitation of Basic `tail -f`

Log files can be rotated.

For example:

```text
application.log
```

might become:

```text
application.log.1
```

while a new `application.log` is created.

GNU `tail` also provides:

```bash
tail -F FILE
```

which can be more resilient when a followed file is replaced during log rotation.

You do not need to master log rotation yet, but recognize that production log handling can be more complicated than one permanently growing file.

---

# 28. Creating Text With `echo`

`echo` writes its arguments to standard output.

Example:

```bash
echo "Linux text processing"
```

Output:

```text
Linux text processing
```

By itself, it does not necessarily create a file.

It simply prints text.

---

# 29. Redirecting `echo` Into a File

You can redirect output:

```bash
echo "environment=development" > configs/app.conf
```

Now:

```bash
cat configs/app.conf
```

Output:

```text
environment=development
```

The:

```text
>
```

redirects standard output into a file.

---

# 30. Warning: `>` Can Overwrite a File

Suppose:

```bash
cat configs/app.conf
```

shows:

```text
environment=development
```

Run:

```bash
echo "port=8080" > configs/app.conf
```

Now:

```bash
cat configs/app.conf
```

shows:

```text
port=8080
```

The previous content disappeared.

Why?

Because:

```text
>
```

normally truncates the target file before writing new output.

This is extremely important.

Do not casually use `>` with an important configuration file.

---

# 31. Appending With `>>`

To add content without replacing existing content, use:

```text
>>
```

Example:

```bash
echo "environment=development" > configs/app.conf
echo "port=8080" >> configs/app.conf
echo "log_level=INFO" >> configs/app.conf
```

Now:

```bash
cat configs/app.conf
```

shows:

```text
environment=development
port=8080
log_level=INFO
```

Think:

```text
>   replace/create output file

>>  append/create output file
```

Both create the file if it does not already exist in typical shell usage.

---

# 32. `echo` and Shell Expansion

Consider:

```bash
echo "$HOME"
```

You may see:

```text
/home/ann
```

The shell expands:

```text
$HOME
```

before `echo` receives the argument.

Compare:

```bash
echo '$HOME'
```

Output:

```text
$HOME
```

Single quotes normally prevent shell variable expansion.

This matters when generating configuration files and scripts.

Shell quoting deserves its own detailed treatment later.

---

# 33. Creating Text With `printf`

`printf` gives more predictable formatting control than `echo`.

Example:

```bash
printf 'Linux\n'
```

Output:

```text
Linux
```

Here:

```text
\n
```

represents a newline.

---

# 34. Multiple Lines With `printf`

Run:

```bash
printf 'environment=production\nport=8080\nlog_level=INFO\n'
```

Output:

```text
environment=production
port=8080
log_level=INFO
```

Redirect it:

```bash
printf 'environment=production\nport=8080\nlog_level=INFO\n' > configs/service.conf
```

Verify:

```bash
cat configs/service.conf
```

---

# 35. Why Learn Both `echo` and `printf`?

`echo` is convenient for simple output:

```bash
echo "Deployment complete"
```

`printf` is better when precise formatting matters:

```bash
printf 'Server: %s\nStatus: %s\n' "web01" "healthy"
```

Output:

```text
Server: web01
Status: healthy
```

Here:

```text
%s
```

is a placeholder for a string.

The supplied values replace the placeholders in order.

---

# 36. Text Editors

Viewing a file is different from editing it.

Commands such as:

```text
cat
less
head
tail
```

are primarily for reading.

To interactively modify text, Linux commonly provides editors such as:

```text
nano
vim
vi
```

We will begin with Nano because its workflow is easier for beginners.

Then you will learn enough Vim to survive when it is the available editor.

---

# 37. Nano From Zero

Nano is a terminal-based text editor.

Check whether it exists:

```bash
nano --version
```

If installed, you will see version information.

If:

```text
command not found
```

appears, Nano may not be installed.

Do not install software blindly on a production machine just to follow this lab.

Use an available practice environment.

---

# 38. Opening a File With Nano

Run:

```bash
nano configs/app.conf
```

Nano opens the file inside your terminal.

You can move using the arrow keys.

Unlike Vim, Nano starts in a mode where you can immediately type and edit text.

---

# 39. Understanding the Nano Screen

At the bottom you may see shortcuts resembling:

```text
^G Help
^O Write Out
^W Where Is
^K Cut
^U Paste
^X Exit
```

The:

```text
^
```

means the `Ctrl` key.

Therefore:

```text
^X
```

means:

```text
Ctrl+X
```

and:

```text
^O
```

means:

```text
Ctrl+O
```

---

# 40. Editing With Nano

Open:

```bash
nano configs/app.conf
```

Change the contents to:

```text
environment=production
port=8080
log_level=INFO
region=us-east-1
```

You can:

* Use arrow keys to move.
* Type normally.
* Use Backspace/Delete.
* Press Enter for new lines.

---

# 41. Saving in Nano

Press:

```text
Ctrl+O
```

Nano calls this:

```text
Write Out
```

It will ask for the filename.

If you opened:

```text
configs/app.conf
```

the current filename should already appear.

Press:

```text
Enter
```

to confirm.

Your changes are now written to disk.

---

# 42. Exiting Nano

Press:

```text
Ctrl+X
```

If there are no unsaved changes, Nano exits.

If there are unsaved changes, Nano asks whether you want to save them.

Read the prompt carefully rather than pressing random keys.

---

# 43. Searching Inside Nano

Open:

```bash
nano configs/app.conf
```

Press:

```text
Ctrl+W
```

This invokes:

```text
Where Is
```

Type:

```text
log_level
```

Press Enter.

Nano moves to the matching text.

This is useful when configuration files become larger.

---

# 44. Nano Beginner Workflow

Remember:

```text
Open
nano file

Edit
type normally

Search
Ctrl+W

Save
Ctrl+O
Enter

Exit
Ctrl+X
```

For many beginner administration tasks, this is enough.

---

# 45. Nano Troubleshooting: Permission Denied

Suppose:

```bash
nano /etc/example.conf
```

opens a file, but saving fails because your user does not have write permission.

Do not change the file to `777`.

First inspect:

```bash
ls -l /etc/example.conf
```

Ask:

```text
Who owns it?

Who should be allowed to modify it?

Does this change require administrative privileges?
```

A system configuration file may intentionally require root privileges.

If the change is authorized, an administrator may use an appropriate privileged workflow.

Permissions are security, not an inconvenience to disable.

---

# 46. Back Up Important Configuration Before Editing

Before editing a critical configuration, a common safe practice is:

```bash
sudo cp /etc/example.conf /etc/example.conf.bak
```

But do not automatically copy every system configuration this way without understanding ownership, permissions, storage, and operational procedures.

The principle is:

```text
Inspect
  |
  v
Back up when appropriate
  |
  v
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

Production environments may use Git, configuration management, infrastructure-as-code, deployment systems, or other formal change-management mechanisms instead.

---

# 47. Vim Introduction

Vim is a powerful terminal text editor.

It is based on the older `vi` editor.

Vim means:

```text
Vi IMproved
```

Check:

```bash
vim --version
```

Some systems may provide `vi` but not full Vim.

You do not need to become a Vim expert in this part.

But you should understand its basic modes because many Linux environments use Vim or Vi.

---

# 48. Why Beginners Get Trapped in Vim

A beginner runs:

```bash
vim app.conf
```

Then starts typing.

Instead of text appearing normally, unexpected things happen.

Why?

Vim is a modal editor.

The same keyboard key can perform different actions depending on the current mode.

Understanding modes solves much of the confusion.

---

# 49. Three Vim Modes to Know

For beginner purposes, focus on:

```text
Normal mode
Insert mode
Command-line mode
```

People sometimes casually call the last one "command mode", but Vim documentation distinguishes Normal mode from command-line mode.

A useful mental model is:

```text
Normal mode
   |
   | i
   v
Insert mode
   |
   | Esc
   v
Normal mode
   |
   | :
   v
Command-line mode
```

---

# 50. Normal Mode

When Vim starts, it normally begins in Normal mode.

Normal mode is used for:

* Navigation.
* Deleting.
* Copying.
* Pasting.
* Searching.
* Issuing editing commands.
* Entering other modes.

Typing letters in Normal mode does not behave like a normal text editor.

This is the main reason beginners become confused.

---

# 51. Insert Mode

Insert mode lets you type text into the file.

From Normal mode, press:

```text
i
```

You should see something such as:

```text
-- INSERT --
```

Now type normally.

To leave Insert mode:

```text
Esc
```

You return to Normal mode.

---

# 52. Command-Line Mode

From Normal mode, press:

```text
:
```

This opens Vim's command-line prompt at the bottom.

You can then issue commands such as:

```text
:w
```

or:

```text
:q
```

or:

```text
:wq
```

---

# 53. Saving in Vim

From Insert mode:

```text
Esc
```

Then type:

```text
:w
```

and press Enter.

`w` means:

```text
write
```

The file is saved.

---

# 54. Quitting Vim

From Normal mode:

```text
:q
```

Press Enter.

This quits if there are no unsaved changes.

---

# 55. Save and Quit Vim

Use:

```text
:wq
```

Meaning:

```text
write
+
quit
```

Press Enter.

Another common command is:

```text
:x
```

which can also write changes when needed and exit.

For now, `:wq` is easy to remember.

---

# 56. Quit Without Saving

Suppose you accidentally changed something.

Press:

```text
Esc
```

Then:

```text
:q!
```

Press Enter.

The:

```text
!
```

forces the quit and discards unsaved changes.

Be careful.

If you actually wanted those changes, they will be lost.

---

# 57. Essential Vim Survival Sequence

Remember:

```text
Start editing:
i

Stop editing:
Esc

Save:
:w

Quit:
:q

Save and quit:
:wq

Quit without saving:
:q!
```

This small set of commands is enough to prevent the classic:

```text
How do I exit Vim?
```

problem.

---

# 58. Searching in Vim

From Normal mode, type:

```text
/ERROR
```

Press Enter.

Vim searches forward.

Press:

```text
n
```

for the next occurrence.

Press:

```text
N
```

for the previous occurrence.

This search behavior resembles what you learned in `less`.

---

# 59. Nano vs Vim

For a beginner:

```text
Nano
 |
 +-- Simple
 +-- Type immediately
 +-- Short learning curve

Vim
 |
 +-- Modal
 +-- More powerful editing model
 +-- Common on servers
 +-- Larger learning curve
```

You do not need to choose one forever.

Use Nano when it helps you work safely.

Learn enough Vim to operate on systems where it is available or preferred.

---

# 60. Searching Files With `grep`

Now we move from interactive search to command-line search.

`grep` is one of the most important Linux commands.

It searches input for lines matching a pattern.

Syntax:

```bash
grep PATTERN FILE
```

Try:

```bash
grep "ERROR" logs/application.log
```

Output might include:

```text
2026-08-26 10:02:03 ERROR database connection lost
2026-08-26 10:02:04 ERROR request failed
2026-08-26 10:05:00 ERROR API timeout
```

---

# 61. What `grep` Is Doing

Conceptually:

```text
application.log
      |
      v
grep "ERROR"
      |
      +-- matching line
      +-- matching line
      +-- matching line
      |
      v
Terminal
```

`grep` is not just displaying the file.

It is selecting lines that match your pattern.

---

# 62. Searching for a Word

Create:

```bash
printf '%s\n' \
'server=web01' \
'environment=production' \
'port=8080' \
'log_level=INFO' \
'database=db01' \
> configs/web.conf
```

Search:

```bash
grep "production" configs/web.conf
```

Output:

```text
environment=production
```

---

# 63. `grep` Is Case Sensitive by Default

Suppose:

```text
ERROR
Error
error
```

all appear in a file.

Run:

```bash
grep "error" file
```

By default, `grep` distinguishes case.

It will not automatically treat:

```text
ERROR
```

and:

```text
error
```

as identical.

---

# 64. Case-Insensitive Search With `grep -i`

Use:

```bash
grep -i "error" logs/application.log
```

The:

```text
-i
```

means ignore case.

This can match:

```text
ERROR
Error
error
```

depending on the file contents.

---

# 65. Showing Line Numbers With `grep -n`

Use:

```bash
grep -n "ERROR" logs/application.log
```

Possible output:

```text
4:2026-08-26 10:02:03 ERROR database connection lost
5:2026-08-26 10:02:04 ERROR request failed
7:2026-08-26 10:05:00 ERROR API timeout
```

The numbers at the beginning are line numbers.

This is useful when you need to tell someone:

```text
The error appears around line 417.
```

---

# 66. Combining `grep` Options

You can combine options:

```bash
grep -in "error" logs/application.log
```

This means:

```text
-i
Ignore case

-n
Show line numbers
```

You may also see:

```bash
grep -i -n "error" logs/application.log
```

Both express the same intention.

---

# 67. Recursive `grep`

Suppose you have many configuration files:

```text
configs/
|
+-- app.conf
+-- service.conf
+-- web.conf
```

You want to find every occurrence of:

```text
production
```

Use:

```bash
grep -r "production" configs/
```

The:

```text
-r
```

means recursively search directories.

Possible output:

```text
configs/app.conf:environment=production
configs/service.conf:environment=production
configs/web.conf:environment=production
```

---

# 68. Recursive Search With Line Numbers

A practical form is:

```bash
grep -rn "production" configs/
```

Now you may see:

```text
configs/app.conf:1:environment=production
configs/service.conf:1:environment=production
configs/web.conf:2:environment=production
```

You now know:

```text
Filename
Line number
Matching line
```

---

# 69. Case-Insensitive Recursive Search

Combine:

```bash
grep -rin "error" logs/
```

This means:

```text
-r  recursive
-i  ignore case
-n  show line numbers
```

This style is common when investigating unfamiliar configuration or log directories.

---

# 70. Inverting Matches With `grep -v`

Sometimes you want everything except matching lines.

Use:

```bash
grep -v "INFO" logs/application.log
```

The:

```text
-v
```

means invert the match.

Instead of displaying lines containing `INFO`, it displays lines that do not match `INFO`.

This can help remove noise.

---

# 71. Practical Log Filtering With `grep -v`

Suppose a log contains thousands of routine health-check entries:

```text
INFO health check passed
INFO health check passed
INFO health check passed
ERROR database timeout
INFO health check passed
```

You could use:

```bash
grep -v "health check passed" application.log
```

to hide those routine lines.

Be careful.

Filtering can hide important context if your pattern is too broad.

---

# 72. Counting Matches With `grep -c`

Another useful option is:

```bash
grep -c "ERROR" logs/application.log
```

This reports the number of matching lines.

Important distinction:

It counts matching lines, not necessarily the total number of times the word appears.

If one line contains `ERROR` three times, `grep -c` still counts that line once.

---

# 73. Showing Context Around Matches

Logs often require surrounding lines.

GNU `grep` commonly supports:

```bash
grep -B 2 "ERROR" logs/application.log
```

`-B 2` means two lines before.

```bash
grep -A 2 "ERROR" logs/application.log
```

`-A 2` means two lines after.

```bash
grep -C 2 "ERROR" logs/application.log
```

`-C 2` means two lines of context on both sides.

This is valuable because the cause of an error may appear immediately before the error message.

---

# 74. Basic Regular Expressions

`grep` becomes much more powerful when you understand patterns.

A regular expression, often shortened to regex, describes a pattern of text.

Instead of searching only for:

```text
ERROR
```

you can express ideas such as:

```text
Lines beginning with ERROR

Lines ending with failed

Lines containing either of several patterns

Lines containing certain character sequences
```

Regex can become very advanced.

At beginner level, learn a few core ideas.

---

# 75. Literal Text vs Patterns

This:

```bash
grep "ERROR" file
```

looks for the pattern:

```text
ERROR
```

The letters are literal here.

But some characters have special meanings in regular expressions.

Examples include:

```text
^
$
.
*
[]
```

Their exact behavior depends on the regular-expression dialect and escaping.

We will keep the first examples simple.

---

# 76. `^` Means Beginning of Line

Create:

```bash
printf '%s\n' \
'ERROR database failed' \
'INFO application started' \
'WARN high memory usage' \
'ERROR API timeout' \
> logs/status.log
```

Run:

```bash
grep '^ERROR' logs/status.log
```

Output:

```text
ERROR database failed
ERROR API timeout
```

The:

```text
^
```

anchors the pattern to the beginning of the line.

---

# 77. `$` Means End of Line

Create:

```bash
printf '%s\n' \
'deployment passed' \
'database failed' \
'authentication failed' \
'backup passed' \
> data/results.txt
```

Run:

```bash
grep 'failed$' data/results.txt
```

Output:

```text
database failed
authentication failed
```

The:

```text
$
```

means the pattern must occur at the end of the line.

---

# 78. The Dot `.`

In regular expressions:

```text
.
```

normally matches any single character.

Suppose:

```text
cat
cut
cot
```

A pattern such as:

```text
c.t
```

can match all three.

Be careful.

A literal dot in something like:

```text
10.0.0.1
```

is not the same as regex `.` unless escaped or otherwise treated literally.

---

# 79. The `*` Regex Operator

This is a common source of beginner confusion.

In shell wildcards:

```text
*
```

often means:

```text
any sequence of filename characters
```

In basic regular expressions, `*` means roughly:

```text
zero or more occurrences of the preceding expression
```

These are not the same thing.

Do not confuse shell globbing with regular expressions.

---

# 80. Character Classes

Square brackets allow character matching.

For example:

```text
[0-9]
```

matches one digit.

```text
[a-z]
```

can match a lowercase letter in common locales, although locale rules can affect ranges.

Example:

```bash
grep '[0-9]' file
```

selects lines containing at least one matching digit.

---

# 81. Extended Regular Expressions

GNU `grep` supports:

```bash
grep -E
```

for extended regular expressions.

This is useful for constructs such as alternatives:

```bash
grep -E 'ERROR|WARN' logs/application.log
```

This selects lines containing either:

```text
ERROR
```

or:

```text
WARN
```

You may also encounter the older command:

```bash
egrep
```

but:

```bash
grep -E
```

is clearer and preferred in modern usage.

---

# 82. Regex Safety for Beginners

Do not try to write complicated regular expressions immediately.

Use this progression:

```text
Literal word
   |
   v
Case-insensitive word
   |
   v
Beginning/end anchors
   |
   v
Character classes
   |
   v
Simple alternatives
   |
   v
More advanced regex later
```

A simple correct search is better than a clever pattern you do not understand.

---

# 83. Counting With `wc`

`wc` means word count.

It can count:

* Lines.
* Words.
* Bytes.
* Characters, depending on options.

Run:

```bash
wc logs/application.log
```

You may see three numbers followed by the filename.

Typically these represent:

```text
lines words bytes
```

---

# 84. Counting Lines With `wc -l`

Use:

```bash
wc -l logs/application.log
```

The:

```text
-l
```

counts lines.

This is useful for questions such as:

```text
How many records are in this file?

How many matching log lines did my pipeline produce?
```

---

# 85. Counting Words With `wc -w`

Use:

```bash
wc -w logs/application.log
```

The:

```text
-w
```

counts words according to `wc`'s word-separation rules.

---

# 86. Counting Bytes With `wc -c`

Use:

```bash
wc -c logs/application.log
```

This counts bytes.

Do not automatically equate bytes with human-visible characters. Multi-byte encodings such as UTF-8 can make the distinction important.

GNU `wc` also commonly supports:

```bash
wc -m
```

for character counts.

---

# 87. Combining `grep` and `wc`

You want to count lines containing:

```text
ERROR
```

You could use:

```bash
grep "ERROR" logs/application.log | wc -l
```

The:

```text
|
```

is a pipe.

It sends the standard output of the first command into the standard input of the next command.

Conceptually:

```text
application.log
      |
      v
grep "ERROR"
      |
      v
matching lines
      |
      v
wc -l
      |
      v
number
```

You will study shell pipelines in greater depth later.

For now, understand the basic flow.

---

# 88. `grep -c` vs `grep | wc -l`

Both can be useful.

```bash
grep -c "ERROR" file
```

directly counts matching lines.

```bash
grep "ERROR" file | wc -l
```

counts the lines emitted by `grep`.

For a straightforward single-file match count, `grep -c` is simpler.

The pipeline becomes useful when additional processing sits between search and count.

---

# 89. Sorting Text With `sort`

Create:

```bash
printf '%s\n' \
'production' \
'development' \
'staging' \
'development' \
'production' \
'production' \
> data/environments.txt
```

View:

```bash
cat data/environments.txt
```

Now:

```bash
sort data/environments.txt
```

Output:

```text
development
development
production
production
production
staging
```

`sort` sends sorted output to your terminal.

It does not modify the original file by default.

---

# 90. Verify That `sort` Did Not Modify the File

Run:

```bash
cat data/environments.txt
```

The original ordering should still exist.

This is an important pattern.

Many Linux text-processing tools:

```text
Read input
Process it
Write result to standard output
```

They do not automatically rewrite the original file.

---

# 91. Reverse Sorting

Use:

```bash
sort -r data/environments.txt
```

The:

```text
-r
```

reverses the sort order.

---

# 92. Numeric Sorting

Create:

```bash
printf '%s\n' 100 9 25 3 1000 > data/numbers.txt
```

Run:

```bash
sort data/numbers.txt
```

You may get lexical ordering:

```text
100
1000
25
3
9
```

Why?

Plain `sort` does not necessarily interpret values as numbers.

Use:

```bash
sort -n data/numbers.txt
```

Output:

```text
3
9
25
100
1000
```

The:

```text
-n
```

means numeric comparison.

---

# 93. Removing Adjacent Duplicates With `uniq`

Use:

```bash
uniq data/environments.txt
```

You might be surprised that some duplicates remain.

Why?

`uniq` works on adjacent repeated lines.

It does not globally search the entire file for duplicates.

---

# 94. `sort | uniq`

A common workflow is:

```bash
sort data/environments.txt | uniq
```

First:

```text
sort
```

puts identical lines together.

Then:

```text
uniq
```

collapses adjacent duplicates.

Output:

```text
development
production
staging
```

---

# 95. Counting Duplicates With `uniq -c`

Run:

```bash
sort data/environments.txt | uniq -c
```

Output may resemble:

```text
2 development
3 production
1 staging
```

This tells you how many adjacent occurrences `uniq` saw after sorting.

This pattern is extremely useful.

---

# 96. Real-World `sort` and `uniq`

Imagine a file containing HTTP status codes:

```text
200
200
500
404
200
500
503
```

You want a frequency count.

Use:

```bash
sort status-codes.txt | uniq -c
```

You might get:

```text
3 200
1 404
2 500
1 503
```

Now you have converted raw records into a small summary.

---

# 97. Extracting Fields With `cut`

`cut` extracts selected portions of each input line.

Create:

```bash
printf '%s\n' \
'ann:engineering:production' \
'alice:security:staging' \
'bob:platform:production' \
> data/team.txt
```

The delimiter is:

```text
:
```

Each line contains fields:

```text
username:team:environment
```

---

# 98. Using `cut -d` and `-f`

Run:

```bash
cut -d ':' -f 1 data/team.txt
```

Output:

```text
ann
alice
bob
```

Breakdown:

```text
cut
Extract fields

-d ':'
Use : as delimiter

-f 1
Return field 1

data/team.txt
Input file
```

---

# 99. Extracting Another Field

Run:

```bash
cut -d ':' -f 2 data/team.txt
```

Output:

```text
engineering
security
platform
```

Field 3:

```bash
cut -d ':' -f 3 data/team.txt
```

Output:

```text
production
staging
production
```

---

# 100. Practical `cut` Example With `/etc/passwd`

Recall from Part 4 that `/etc/passwd` uses colon-separated fields.

You can inspect usernames:

```bash
cut -d ':' -f 1 /etc/passwd
```

This extracts the first field from every line.

This is a useful example of connecting Linux concepts.

You are no longer learning `cut` in isolation.

You are using it against a real Linux system file you already understand.

---

# 101. Translating Characters With `tr`

`tr` translates or deletes characters from standard input.

Unlike many commands we have used, `tr` normally reads from standard input rather than taking a filename as its main operand.

Example:

```bash
echo "production" | tr 'a-z' 'A-Z'
```

Output:

```text
PRODUCTION
```

Conceptually:

```text
echo
 |
 v
production
 |
 v
tr
 |
 v
PRODUCTION
```

---

# 102. Replacing Delimiters With `tr`

Run:

```bash
echo "web01:production:healthy" | tr ':' ','
```

Output:

```text
web01,production,healthy
```

Every:

```text
:
```

becomes:

```text
,
```

---

# 103. Deleting Characters With `tr -d`

Use:

```bash
echo "server-01" | tr -d '-'
```

Output:

```text
server01
```

The:

```text
-d
```

means delete characters from the specified set.

---

# 104. Why `tr` Is Useful

You may need to:

* Convert lowercase to uppercase.
* Replace delimiters.
* Remove certain characters.
* Normalize simple text streams.

It is small, but very useful inside pipelines.

---

# 105. Introduction to `sed`

`sed` means stream editor.

It reads text, applies editing instructions, and writes the result.

A beginner use is substitution.

Suppose:

```bash
cat configs/app.conf
```

contains:

```text
environment=production
port=8080
log_level=INFO
region=us-east-1
```

Run:

```bash
sed 's/production/staging/' configs/app.conf
```

Output:

```text
environment=staging
port=8080
log_level=INFO
region=us-east-1
```

---

# 106. Understanding the `sed` Substitution

The expression:

```text
s/production/staging/
```

means:

```text
s
substitute

production
pattern to replace

staging
replacement
```

The slash separates the parts.

Conceptually:

```text
s / old / new /
```

---

# 107. `sed` Does Not Necessarily Modify the Original File

After:

```bash
sed 's/production/staging/' configs/app.conf
```

check:

```bash
cat configs/app.conf
```

The file should still contain:

```text
environment=production
```

The normal `sed` command wrote transformed text to standard output.

It did not rewrite the file.

This makes it useful for previewing transformations.

---

# 108. Global Replacement With `sed`

Suppose:

```text
error error error
```

With:

```bash
sed 's/error/ERROR/' file
```

only the first matching occurrence on each line is normally replaced.

To replace all matching occurrences on each line:

```bash
sed 's/error/ERROR/g' file
```

The:

```text
g
```

means global within each processed line.

---

# 109. In-Place `sed` Editing

You may encounter:

```bash
sed -i 's/production/staging/' configs/app.conf
```

The:

```text
-i
```

requests in-place editing.

This changes the file.

Be careful.

For beginners, prefer:

```bash
sed 's/production/staging/' configs/app.conf
```

first.

Inspect the output.

Only perform an in-place edit when you understand the transformation and have an appropriate recovery plan.

Also note that `sed -i` syntax differs across some Unix implementations, such as GNU/Linux and BSD/macOS.

---

# 110. Safe `sed` Workflow

A good beginner workflow is:

```text
Original file
   |
   v
Run sed without -i
   |
   v
Inspect output
   |
   v
Confirm transformation
   |
   v
Back up if appropriate
   |
   v
Apply intentional change
   |
   v
Verify file
```

Never turn a search-and-replace experiment directly into a production configuration change without verifying it.

---

# 111. Introduction to `awk`

`awk` is a text-processing language.

It is particularly useful for structured text where each line contains fields.

Consider:

```text
web01 production healthy
web02 production unhealthy
db01 production healthy
dev01 development healthy
```

Create it:

```bash
printf '%s\n' \
'web01 production healthy' \
'web02 production unhealthy' \
'db01 production healthy' \
'dev01 development healthy' \
> data/servers.txt
```

---

# 112. Printing a Field With `awk`

Run:

```bash
awk '{print $1}' data/servers.txt
```

Output:

```text
web01
web02
db01
dev01
```

In `awk`:

```text
$1
```

means:

```text
first field
```

---

# 113. Printing the Second Field

Run:

```bash
awk '{print $2}' data/servers.txt
```

Output:

```text
production
production
production
development
```

Third field:

```bash
awk '{print $3}' data/servers.txt
```

Output:

```text
healthy
unhealthy
healthy
healthy
```

---

# 114. Understanding `$0` in `awk`

In `awk`:

```text
$0
```

means the entire current record, normally the whole line.

Try:

```bash
awk '{print $0}' data/servers.txt
```

This prints each complete line.

Think:

```text
$0 = whole line
$1 = first field
$2 = second field
$3 = third field
```

---

# 115. Filtering With `awk`

You can combine conditions with actions.

Run:

```bash
awk '$2 == "production" {print $1}' data/servers.txt
```

Output:

```text
web01
web02
db01
```

Breakdown:

```text
$2 == "production"
If field 2 equals production

{print $1}
Print field 1
```

This is already more than simple extraction.

You are making a decision based on one field and outputting another.

---

# 116. Custom Delimiters in `awk`

Our earlier team file used:

```text
ann:engineering:production
```

To tell `awk` that `:` separates fields:

```bash
awk -F ':' '{print $1}' data/team.txt
```

The:

```text
-F ':'
```

sets the input field separator.

Output:

```text
ann
alice
bob
```

---

# 117. `cut` vs `awk`

For straightforward delimiter-based extraction:

```bash
cut -d ':' -f 1 file
```

is simple.

For logic and richer processing:

```bash
awk -F ':' '$3 == "production" {print $1}' file
```

is more powerful.

Output:

```text
ann
bob
```

Do not ask:

```text
Which command is universally better?
```

Ask:

```text
What transformation am I trying to perform?
```

---

# 118. `grep` vs `sed` vs `awk`

A beginner mental model:

```text
grep
Find/select matching lines

sed
Transform/edit text streams

awk
Process records and fields
```

There is overlap.

For example, `awk` can search, and `sed` can select lines.

But this simplified distinction helps you choose a starting tool.

---

# 119. Searching Configuration Files

Imagine you inherit a server and need to find where port `8080` is configured.

You know the application configurations are under:

```text
~/text-lab/configs
```

Use:

```bash
grep -rn "8080" ~/text-lab/configs
```

This searches recursively and shows line numbers.

You might see:

```text
/home/user/text-lab/configs/app.conf:2:port=8080
/home/user/text-lab/configs/service.conf:2:port=8080
/home/user/text-lab/configs/web.conf:3:port=8080
```

Now you know where the value appears.

---

# 120. Searching System Configuration

Suppose you are investigating SSH configuration.

A command such as:

```bash
grep -n "PasswordAuthentication" /etc/ssh/sshd_config
```

may find a setting.

But be careful.

Configuration files can contain:

* Comments.
* Included files.
* Defaults.
* Multiple declarations.
* Distribution-specific behavior.

Finding a line does not automatically prove the effective configuration.

Searching is evidence gathering, not always the final answer.

---

# 121. Ignoring Comment Lines

Many configuration files use:

```text
#
```

for comments.

Suppose:

```text
#port=8080
port=9090
#log_level=DEBUG
log_level=INFO
```

To remove lines beginning with `#`:

```bash
grep -v '^#' file
```

But this may still include blank lines.

You can extend filtering later as your regex knowledge improves.

Again, do not assume every configuration format uses `#` for comments.

---

# 122. Searching Logs

Imagine an application log contains thousands of lines.

Start with:

```bash
grep "ERROR" application.log
```

Then perhaps:

```bash
grep -n "ERROR" application.log
```

Then:

```bash
grep -C 3 "ERROR" application.log
```

This progression answers:

```text
Are there errors?

Where are they?

What happened around them?
```

---

# 123. Searching Multiple Error Levels

Use:

```bash
grep -E 'ERROR|WARN' logs/application.log
```

This finds either pattern.

Add line numbers:

```bash
grep -nE 'ERROR|WARN' logs/application.log
```

Add case-insensitivity if necessary:

```bash
grep -niE 'error|warn' logs/application.log
```

Do not stack options merely because you can.

Know what each one does.

---

# 124. Searching Compressed or Rotated Logs

In real environments, logs may appear as:

```text
application.log
application.log.1
application.log.2.gz
```

A normal `grep` does not directly treat compressed `.gz` data as plain text.

On systems with gzip tools, you may encounter:

```bash
zgrep "ERROR" application.log.2.gz
```

You do not need to master rotated logs in this part.

The important lesson is that historical logs may exist outside the current `.log` file.

---

# 125. Building Text Pipelines

This is where Linux starts becoming powerful.

Suppose:

```text
data/team.txt
```

contains:

```text
ann:engineering:production
alice:security:staging
bob:platform:production
```

You want only environments.

Use:

```bash
cut -d ':' -f 3 data/team.txt
```

Output:

```text
production
staging
production
```

Now sort:

```bash
cut -d ':' -f 3 data/team.txt | sort
```

Output:

```text
production
production
staging
```

Now count:

```bash
cut -d ':' -f 3 data/team.txt | sort | uniq -c
```

Output:

```text
2 production
1 staging
```

---

# 126. Understand the Pipeline

Do not memorize:

```bash
cut -d ':' -f 3 data/team.txt | sort | uniq -c
```

Understand the flow:

```text
team.txt
   |
   v
cut
Extract environment
   |
   v
production
staging
production
   |
   v
sort
   |
   v
production
production
staging
   |
   v
uniq -c
   |
   v
2 production
1 staging
```

This is the Unix philosophy in action.

Small tools can be combined to solve larger problems.

---

# 127. Real-World Example: Count Log Levels

Create a larger log:

```bash
printf '%s\n' \
'2026-08-26 12:00:01 INFO request started' \
'2026-08-26 12:00:02 INFO request completed' \
'2026-08-26 12:01:01 WARN response slow' \
'2026-08-26 12:01:30 ERROR database timeout' \
'2026-08-26 12:02:00 INFO request started' \
'2026-08-26 12:02:02 ERROR request failed' \
'2026-08-26 12:03:00 WARN memory high' \
'2026-08-26 12:04:00 INFO request completed' \
> logs/production.log
```

Each line has fields:

```text
Date Time Level Message...
```

Therefore the log level is field 3.

Use:

```bash
awk '{print $3}' logs/production.log
```

Output:

```text
INFO
INFO
WARN
ERROR
INFO
ERROR
WARN
INFO
```

---

# 128. Count Each Log Level

Pipe it:

```bash
awk '{print $3}' logs/production.log | sort | uniq -c
```

Output:

```text
2 ERROR
4 INFO
2 WARN
```

You transformed raw logs into an operational summary.

This is the kind of thinking Linux engineers use constantly.

---

# 129. Sort the Counts Numerically

You can continue:

```bash
awk '{print $3}' logs/production.log | sort | uniq -c | sort -nr
```

The final:

```text
sort -nr
```

means:

```text
-n
numeric sort

-r
reverse
```

So the highest count appears first.

This is a simple example of gradually building a pipeline.

---

# 130. Do Not Build Complex Pipelines All at Once

Suppose you see:

```bash
awk '{print $3}' file | sort | uniq -c | sort -nr
```

A beginner may copy it without understanding anything.

Instead build it progressively.

First:

```bash
awk '{print $3}' file
```

Inspect.

Then:

```bash
awk '{print $3}' file | sort
```

Inspect.

Then:

```bash
awk '{print $3}' file | sort | uniq -c
```

Inspect.

Finally:

```bash
awk '{print $3}' file | sort | uniq -c | sort -nr
```

This technique makes pipelines much easier to troubleshoot.

---

# 131. Troubleshooting Text Commands

When a text-processing command gives unexpected output, ask:

```text
Does the input actually contain what I think?

Am I using the correct file?

Am I in the expected directory?

Is the search case-sensitive?

Is my delimiter correct?

Are fields separated consistently?

Is shell quoting changing my pattern?

Is regex interpreting special characters?

Does the file use unusual line endings?

Do I have permission to read the file?

Is the file actually plain text?
```

These questions prevent random command changes.

---

# 132. Problem: `grep` Finds Nothing

You run:

```bash
grep "error" application.log
```

Nothing appears.

Possible causes:

1. There are no matches.
2. The file uses `ERROR`, not `error`.
3. You are searching the wrong file.
4. The error is in a rotated log.
5. Your pattern is wrong.
6. The file is binary or formatted differently.
7. Permissions or path assumptions are wrong.

Start by inspecting:

```bash
head application.log
```

or:

```bash
less application.log
```

Then perhaps:

```bash
grep -i "error" application.log
```

Do not immediately assume `grep` is broken.

---

# 133. Problem: `cat` Says No Such File

You run:

```bash
cat application.log
```

and receive:

```text
No such file or directory
```

Investigate:

```bash
pwd
```

Then:

```bash
ls
```

Perhaps the file is actually:

```text
logs/application.log
```

Use:

```bash
cat logs/application.log
```

This connects directly to the filesystem-navigation skills from Part 2.

---

# 134. Problem: Permission Denied

You run:

```bash
cat application.log
```

and receive:

```text
Permission denied
```

Do not run:

```bash
chmod 777 application.log
```

Inspect:

```bash
whoami
id
ls -l application.log
```

If the file is inside a protected path, inspect the parent directories too.

Use the permission troubleshooting process from Part 5.

---

# 135. Problem: Nano Cannot Save

You edit a file and Nano reports that it cannot write it.

Ask:

```text
Do I own the file?

Do I have write permission?

Do I have required directory access?

Is the filesystem read-only?

Is the disk full?

Am I editing a system-owned configuration?
```

Start with:

```bash
ls -l file
```

and:

```bash
df -h
```

when storage may be relevant.

Do not solve every write failure with permission changes.

---

# 136. Problem: Vim Will Not Let Me Type Normally

You opened Vim and typed:

```text
hello
```

but strange actions occurred.

Likely cause:

You were in Normal mode.

Press:

```text
i
```

to enter Insert mode.

Type.

Then:

```text
Esc
```

to return to Normal mode.

---

# 137. Problem: Vim Says There Are Unsaved Changes

You try:

```text
:q
```

and Vim refuses to exit.

If you want to save:

```text
:wq
```

If you intentionally want to discard changes:

```text
:q!
```

Read the warning before choosing.

---

# 138. Problem: `uniq` Did Not Remove Every Duplicate

Input:

```text
production
staging
production
```

Run:

```bash
uniq file
```

Both `production` lines remain because they are not adjacent.

Use:

```bash
sort file | uniq
```

if global deduplication by sorted line value is what you need.

---

# 139. Problem: `sort` Orders Numbers Strangely

Input:

```text
2
10
100
3
```

Plain:

```bash
sort file
```

may sort lexically.

Use:

```bash
sort -n file
```

when numeric comparison is required.

---

# 140. Problem: `cut` Returns the Wrong Field

You run:

```bash
cut -d ':' -f 2 file
```

but output is wrong.

Inspect:

```bash
head file
```

Maybe the delimiter is:

```text
,
```

rather than:

```text
:
```

Or perhaps whitespace separates fields.

Understand the input format before choosing the parser.

---

# 141. Problem: `awk` Fields Are Unexpected

By default, `awk` handles whitespace-separated fields conveniently.

If your data is colon-separated:

```text
ann:engineering:production
```

then:

```bash
awk '{print $2}' file
```

does not interpret the colons as field separators.

Specify:

```bash
awk -F ':' '{print $2}' file
```

Always understand the delimiter.

---

# 142. Problem: `sed` Changed More Than Expected

Suppose your replacement pattern is too broad.

You intended to change one setting but matched many lines.

This is why previewing is valuable:

```bash
sed 's/old/new/' file
```

before using in-place editing.

For critical files:

```text
Search first
Preview
Back up if appropriate
Apply
Inspect diff or result
Validate application
```

---

# 143. Text-Processing Lab

Now combine everything.

Use:

```text
~/text-lab
```

Do not use important system files for destructive practice.

---

# 144. Lab Task 1: Create an Application Configuration

Move:

```bash
cd ~/text-lab
```

Create:

```bash
printf '%s\n' \
'app_name=veriqta-api' \
'environment=development' \
'port=8080' \
'log_level=DEBUG' \
'database_host=db01.internal' \
> configs/application.conf
```

View:

```bash
cat configs/application.conf
```

---

# 145. Lab Task 2: Inspect It in Different Ways

Run:

```bash
head -n 2 configs/application.conf
```

Then:

```bash
tail -n 2 configs/application.conf
```

Then:

```bash
less configs/application.conf
```

Exit `less` using:

```text
q
```

Explain to yourself when each command is more useful.

---

# 146. Lab Task 3: Edit With Nano

Open:

```bash
nano configs/application.conf
```

Change:

```text
environment=development
```

to:

```text
environment=staging
```

Change:

```text
log_level=DEBUG
```

to:

```text
log_level=INFO
```

Save:

```text
Ctrl+O
Enter
```

Exit:

```text
Ctrl+X
```

Verify:

```bash
cat configs/application.conf
```

---

# 147. Lab Task 4: Search Configuration

Run:

```bash
grep "environment" configs/application.conf
```

Then:

```bash
grep -n "log_level" configs/application.conf
```

Then:

```bash
grep -i "INFO" configs/application.conf
```

Understand what each option changes.

---

# 148. Lab Task 5: Create Production Logs

Create:

```bash
printf '%s\n' \
'2026-08-26 14:00:01 INFO service started' \
'2026-08-26 14:00:05 INFO database connected' \
'2026-08-26 14:01:00 INFO request completed status=200' \
'2026-08-26 14:01:20 WARN request slow status=200' \
'2026-08-26 14:02:01 ERROR database timeout status=500' \
'2026-08-26 14:02:02 ERROR request failed status=500' \
'2026-08-26 14:03:00 INFO database reconnected' \
'2026-08-26 14:03:30 INFO request completed status=200' \
'2026-08-26 14:04:00 WARN memory usage high' \
'2026-08-26 14:05:00 ERROR upstream timeout status=502' \
> logs/api.log
```

---

# 149. Lab Task 6: Investigate the Log

First:

```bash
wc -l logs/api.log
```

Then:

```bash
head -n 3 logs/api.log
```

Then:

```bash
tail -n 3 logs/api.log
```

Search errors:

```bash
grep "ERROR" logs/api.log
```

Count them:

```bash
grep -c "ERROR" logs/api.log
```

Show line numbers:

```bash
grep -n "ERROR" logs/api.log
```

---

# 150. Lab Task 7: Search Errors With Context

Run:

```bash
grep -C 1 "ERROR" logs/api.log
```

Observe what occurred immediately around each error.

Ask:

```text
Did an INFO or WARN event occur before it?

Did recovery happen afterward?

Are several errors related?
```

This is how logs begin to tell a story.

---

# 151. Lab Task 8: Filter Routine Information

Run:

```bash
grep -v "INFO" logs/api.log
```

You should mostly see:

```text
WARN
ERROR
```

Now:

```bash
grep -E 'WARN|ERROR' logs/api.log
```

Compare the results.

The commands approach the task differently.

---

# 152. Lab Task 9: Follow the Log

Terminal 1:

```bash
tail -f ~/text-lab/logs/api.log
```

Terminal 2:

```bash
echo "2026-08-26 14:06:00 ERROR authentication service unavailable status=503" >> ~/text-lab/logs/api.log
```

Observe Terminal 1.

Then add:

```bash
echo "2026-08-26 14:06:20 INFO authentication service recovered" >> ~/text-lab/logs/api.log
```

Observe again.

Stop Terminal 1 with:

```text
Ctrl+C
```

---

# 153. Lab Task 10: Extract Status Codes

Our status information appears in fields such as:

```text
status=500
```

Start by finding lines containing status:

```bash
grep "status=" logs/api.log
```

Then use `awk` to inspect the last field:

```bash
grep "status=" logs/api.log | awk '{print $NF}'
```

`$NF` means the last field in the current record.

Possible output:

```text
status=200
status=200
status=500
status=500
status=200
status=502
status=503
```

---

# 154. Lab Task 11: Remove `status=`

Pipe through `cut`:

```bash
grep "status=" logs/api.log | awk '{print $NF}' | cut -d '=' -f 2
```

Possible output:

```text
200
200
500
500
200
502
503
```

Understand each stage.

Do not memorize the whole line.

---

# 155. Lab Task 12: Count Status Codes

Continue:

```bash
grep "status=" logs/api.log \
| awk '{print $NF}' \
| cut -d '=' -f 2 \
| sort \
| uniq -c
```

You may see something like:

```text
3 200
2 500
1 502
1 503
```

This is a basic log-analysis pipeline.

---

# 156. Lab Task 13: Sort by Frequency

Add:

```bash
sort -nr
```

Complete pipeline:

```bash
grep "status=" logs/api.log \
| awk '{print $NF}' \
| cut -d '=' -f 2 \
| sort \
| uniq -c \
| sort -nr
```

Now the most frequent status should appear first.

You have combined:

```text
grep
awk
cut
sort
uniq
sort
```

into one workflow.

---

# 157. Lab Task 14: Transform Configuration With `sed`

Preview:

```bash
sed 's/environment=staging/environment=production/' configs/application.conf
```

Inspect the output.

Then check the actual file:

```bash
grep "environment" configs/application.conf
```

It should still say:

```text
environment=staging
```

because you did not use in-place editing.

This demonstrates the difference between transformed output and modifying the source file.

---

# 158. Lab Task 15: Practice `tr`

Run:

```bash
echo "production:veriqta-api:healthy" | tr ':' ','
```

Expected:

```text
production,veriqta-api,healthy
```

Then:

```bash
echo "production" | tr 'a-z' 'A-Z'
```

Expected:

```text
PRODUCTION
```

---

# 159. Lab Task 16: Practice `sort` and `uniq`

Create:

```bash
printf '%s\n' \
'nginx' \
'docker' \
'kubernetes' \
'nginx' \
'linux' \
'docker' \
'nginx' \
> data/tools.txt
```

Run:

```bash
sort data/tools.txt
```

Then:

```bash
sort data/tools.txt | uniq
```

Then:

```bash
sort data/tools.txt | uniq -c
```

Then:

```bash
sort data/tools.txt | uniq -c | sort -nr
```

Explain what each additional command contributes.

---

# 160. Lab Task 17: Practice Vim Safely

Create a disposable file:

```bash
echo "environment=development" > configs/vim-practice.conf
```

Open:

```bash
vim configs/vim-practice.conf
```

Then:

1. Press `i`.
2. Edit `development` to `production`.
3. Press `Esc`.
4. Type `:wq`.
5. Press Enter.

Verify:

```bash
cat configs/vim-practice.conf
```

If you become stuck:

```text
Esc
:q!
Enter
```

will quit without saving your unsaved practice changes.

---

# 161. Lab Task 18: Recursive Configuration Search

Create:

```bash
mkdir -p configs/nginx configs/application
```

Run:

```bash
echo "listen=8080" > configs/nginx/server.conf
echo "backend_port=8080" > configs/application/backend.conf
echo "metrics_port=9090" > configs/application/monitoring.conf
```

Now search:

```bash
grep -rn "8080" configs/
```

You should see both matching files and their line numbers.

This simulates searching an unfamiliar configuration tree.

---

# 162. Lab Task 19: Regex Practice

Create:

```bash
printf '%s\n' \
'INFO service healthy' \
'ERROR database timeout' \
'WARN memory high' \
'INFO request complete' \
'ERROR API failed' \
> logs/regex.log
```

Find lines beginning with `ERROR`:

```bash
grep '^ERROR' logs/regex.log
```

Find lines ending in `failed`:

```bash
grep 'failed$' logs/regex.log
```

Find ERROR or WARN:

```bash
grep -E 'ERROR|WARN' logs/regex.log
```

---

# 163. Lab Task 20: Build Your Own Investigation

Scenario:

A developer tells you:

```text
"The API is failing. I think it started after 14:02."
```

Using:

```text
logs/api.log
```

investigate without immediately reading the entire file.

Try to answer:

1. How many lines are in the log?
2. What are the newest entries?
3. How many errors occurred?
4. Which error levels exist?
5. Which HTTP status codes occurred?
6. Which status code appears most frequently?
7. What happened around the database timeout?
8. Did the database recover?
9. Was there another failure later?
10. What evidence supports your conclusion?

This is the beginning of real troubleshooting.

---

# 164. Text Troubleshooting Reasoning

When investigating a large file, use a process:

```text
Understand the question
        |
        v
Identify likely file
        |
        v
Inspect file size/type
        |
        v
Sample beginning/end
        |
        v
Search broad patterns
        |
        v
Narrow the search
        |
        v
Inspect surrounding context
        |
        v
Extract useful fields
        |
        v
Count/summarize
        |
        v
Form hypothesis
        |
        v
Find evidence
```

Do not begin with a giant complicated command.

---

# 165. Real-World Scenario: Nginx Error Investigation

Imagine:

```text
Users
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

Users report:

```text
502 Bad Gateway
```

You locate an Nginx error log.

A sensible investigation could begin:

```bash
tail -n 100 /var/log/nginx/error.log
```

Then perhaps:

```bash
grep -i "error" /var/log/nginx/error.log
```

Or search for:

```text
upstream
```

depending on what the log contains.

You might use:

```bash
grep -iC 3 "upstream" /var/log/nginx/error.log
```

The goal is not to memorize one magical Nginx command.

The goal is to know how to interrogate text evidence.

---

# 166. Real-World Scenario: Find a Configuration Value

You inherit an application.

You know the configuration is somewhere under:

```text
/etc/myapp/
```

but you do not know which file controls the database host.

You might search:

```bash
grep -rin "database" /etc/myapp/
```

Then narrow based on results.

Maybe you find:

```text
/etc/myapp/application.conf:42:database_host=db01.internal
```

Before changing it:

```bash
less /etc/myapp/application.conf
```

Read the surrounding configuration.

Then determine:

* Whether this is the active file.
* Whether the line is commented.
* Whether another file overrides it.
* Whether the service needs validation/reload.
* Whether the change is authorized.

Searching tells you where to investigate.

It does not automatically tell you what to change.

---

# 167. Real-World Scenario: Count Production Errors

Suppose the application log contains millions of entries.

You need to know how many lines contain `ERROR`.

A simple approach:

```bash
grep -c "ERROR" application.log
```

If you need different error types, perhaps:

```bash
grep "ERROR" application.log | awk '{print $5}' | sort | uniq -c | sort -nr
```

But only use `$5` if you have inspected the log and confirmed that field 5 actually contains the value you need.

Never copy field numbers blindly.

---

# 168. Real-World Scenario: CI/CD Logs

A CI/CD job might produce:

```text
INFO Checking out repository
INFO Installing dependencies
INFO Running tests
ERROR Test suite failed
ERROR exit code 1
INFO Uploading artifacts
```

A quick search:

```bash
grep -nE 'ERROR|failed|exit code' pipeline.log
```

may identify useful lines.

Then inspect context:

```bash
grep -nC 5 "Test suite failed" pipeline.log
```

This is more useful than scrolling through thousands of lines without a plan.

---

# 169. Real-World Scenario: Compare Expected Configuration

Suppose you expect:

```text
environment=production
```

Search:

```bash
grep -rn '^environment=' configs/
```

You might discover:

```text
configs/app.conf:environment=production
configs/worker.conf:environment=staging
```

Now you have found a possible inconsistency.

The text tools help you turn a vague statement:

```text
"Something is configured wrong."
```

into evidence:

```text
worker.conf is configured for staging.
```

---

# 170. Security Considerations When Working With Text

Text files may contain sensitive information.

Examples:

```text
password=...
api_key=...
token=...
private_key=...
database_password=...
```

Be careful when running commands that print secrets to your terminal.

For example:

```bash
cat secrets.conf
```

may expose sensitive values:

* On your screen.
* In terminal recordings.
* During screen sharing.
* In captured CI logs.
* In copied troubleshooting output.

Do not expose secrets unnecessarily.

---

# 171. Be Careful With Shell History

Commands themselves may contain secrets.

For example, avoid habits such as:

```bash
echo "password=SuperSecretPassword" > config
```

because the command may be stored in shell history depending on shell configuration.

Later we will cover environment variables, secrets, and shell behavior more carefully.

For now, remember:

```text
Just because a command works does not mean it is a secure way to handle secrets.
```

---

# 172. Be Careful With `sed -i`

This:

```bash
sed -i 's/old/new/' production.conf
```

changes a file directly.

If your pattern is wrong, you may damage configuration.

Safer beginner workflow:

```bash
grep "old" production.conf
```

Then:

```bash
sed 's/old/new/' production.conf
```

Inspect.

Back up or use version control when appropriate.

Then make the intended change.

Verify afterward.

---

# 173. Be Careful With Redirection

This:

```bash
command > important.conf
```

can replace the contents of:

```text
important.conf
```

This:

```bash
command >> important.conf
```

appends instead.

Before pressing Enter, ask:

```text
Am I replacing this file?

Am I appending?

Is that what I actually intend?
```

One character can make a major difference.

---

# 174. Command Reference

## Viewing

```bash
cat FILE
less FILE
more FILE
head FILE
head -n NUMBER FILE
tail FILE
tail -n NUMBER FILE
tail -f FILE
```

## Creating output

```bash
echo "TEXT"
printf 'FORMAT\n'
```

## Editing

```bash
nano FILE
vim FILE
```

## Searching

```bash
grep "PATTERN" FILE
grep -i "PATTERN" FILE
grep -n "PATTERN" FILE
grep -r "PATTERN" DIRECTORY
grep -v "PATTERN" FILE
grep -E 'PATTERN1|PATTERN2' FILE
```

## Counting

```bash
wc FILE
wc -l FILE
wc -w FILE
wc -c FILE
```

## Sorting and duplicates

```bash
sort FILE
sort -n FILE
sort -r FILE
uniq FILE
sort FILE | uniq
sort FILE | uniq -c
```

## Extracting

```bash
cut -d ':' -f 1 FILE
awk '{print $1}' FILE
awk -F ':' '{print $1}' FILE
```

## Transforming

```bash
tr 'a-z' 'A-Z'
sed 's/OLD/NEW/' FILE
```

---

# 175. What You Should Now Understand

Before moving to Part 7, you should understand:

* Why text files are central to Linux.
* Why configuration files matter.
* Why logs are important for troubleshooting.
* Why scripts are text-based automation resources.
* How `cat` displays file contents.
* Why `cat` is not ideal for huge files.
* How `less` lets you navigate large files.
* How to search inside `less`.
* What `more` does.
* How `head` displays the beginning of a file.
* How `tail` displays the end.
* How `tail -f` follows appended log entries.
* Why `tail -F` may help with replaced or rotated logs.
* How `echo` creates standard output.
* How `>` redirects and overwrites.
* How `>>` appends.
* Why careless redirection can destroy file contents.
* How `printf` provides controlled formatting.
* How to open, edit, save, search, and exit Nano.
* Why Nano may fail to save protected files.
* Why backups or version control matter before configuration changes.
* What Vim is.
* Why Vim is modal.
* What Normal mode is.
* What Insert mode is.
* What command-line mode is.
* How to enter Insert mode.
* How to return to Normal mode.
* How to save in Vim.
* How to quit Vim.
* How to quit without saving.
* How to search inside Vim.
* What `grep` does.
* Why `grep` is case-sensitive by default.
* How `grep -i` ignores case.
* How `grep -n` shows line numbers.
* How recursive `grep` works.
* How `grep -v` inverts matching.
* How to show surrounding context.
* What a regular expression is.
* What `^` and `$` mean.
* What `.` means in regex.
* Why regex `*` differs from a shell wildcard.
* What basic character classes are.
* How `grep -E` supports extended patterns.
* What `wc` counts.
* How `wc -l` counts lines.
* How `sort` orders text.
* Why numeric sorting may require `-n`.
* Why `uniq` only removes adjacent duplicates.
* Why `sort | uniq` is common.
* How `uniq -c` counts duplicates.
* How `cut` extracts delimiter-separated fields.
* How `tr` translates or deletes characters.
* What `sed` is.
* How a basic `sed` substitution works.
* Why normal `sed` output does not necessarily change the source file.
* Why `sed -i` requires care.
* What `awk` is.
* How `$1`, `$2`, and `$0` work in `awk`.
* What `$NF` represents.
* How to configure an `awk` field separator.
* The basic differences between `grep`, `sed`, and `awk`.
* How pipes connect text-processing commands.
* Why complex pipelines should be built incrementally.
* How to search configuration directories.
* How to investigate logs.
* How to extract and summarize log data.
* How to troubleshoot unexpected text-processing results.
* Why secrets require care when viewing and manipulating text.

---

# 176. Hands-On Practice

Without copying the lab instructions above, create:

```text
~/linux-text-project/
|
+-- configs/
+-- logs/
+-- data/
+-- scripts/
```

Complete these tasks:

1. Create an application configuration with at least six settings.

2. View it with `cat`.

3. Open it with `less`.

4. Display only its first three lines.

5. Display only its last two lines.

6. Edit it with Nano.

7. Save and exit Nano.

8. Open a disposable copy in Vim.

9. Change one value.

10. Save and quit Vim.

11. Create a log containing at least 20 entries.

12. Include `INFO`, `WARN`, and `ERROR` levels.

13. Find every `ERROR`.

14. Find errors case-insensitively.

15. Show their line numbers.

16. Display context around each error.

17. Count error lines.

18. Display everything except `INFO`.

19. Find both `WARN` and `ERROR` with one search.

20. Follow the log with `tail -f`.

21. Append a new entry from another terminal.

22. Observe it appear.

23. Count the total number of log lines.

24. Extract the log-level field with `awk`.

25. Sort the levels.

26. Count each unique level.

27. Sort those counts from highest to lowest.

28. Create colon-separated data.

29. Extract individual fields with `cut`.

30. Extract them again with `awk -F`.

31. Convert lowercase text to uppercase with `tr`.

32. Replace one configuration value using `sed` without modifying the original.

33. Verify the original remained unchanged.

34. Search recursively through all configuration files.

35. Create duplicate data and explain why `uniq` alone may not remove all duplicates.

36. Fix the duplicate problem using `sort`.

37. Create numbers and compare `sort` with `sort -n`.

38. Use a regex to find lines beginning with `ERROR`.

39. Use another regex to find lines ending with `failed`.

40. Explain every stage of one multi-command pipeline in your own words.

---

# 177. Beginner Knowledge Check

1. Why are text files important in Linux?

2. What is a configuration file?

3. What is a log file?

4. What is a script?

5. What does `cat` do?

6. Why might `cat` be inconvenient for a huge file?

7. What does `less` do?

8. How do you quit `less`?

9. How do you search forward inside `less`?

10. What does `head` do?

11. What does `head -n 5 file` mean?

12. What does `tail` do?

13. Why is `tail` particularly useful for logs?

14. What does `tail -f` do?

15. How do you stop `tail -f`?

16. What does `echo` normally write to?

17. What does `>` do?

18. Why can `>` be dangerous?

19. What does `>>` do?

20. What is one reason to use `printf` instead of `echo`?

21. How do you open a file with Nano?

22. How do you save in Nano?

23. How do you exit Nano?

24. How do you search in Nano?

25. Why might Nano fail to save a system configuration?

26. What is Vim?

27. What is Normal mode?

28. What is Insert mode?

29. How do you enter Insert mode?

30. How do you leave Insert mode?

31. How do you save in Vim?

32. How do you save and quit?

33. How do you quit without saving?

34. What does `grep` do?

35. Is normal `grep` case-sensitive?

36. What does `grep -i` do?

37. What does `grep -n` do?

38. What does `grep -r` do?

39. What does `grep -v` do?

40. What does `grep -C 3` provide?

41. What is a regular expression?

42. What does `^` represent?

43. What does `$` represent?

44. What does `.` normally mean in regex?

45. Is regex `*` identical to the shell filename wildcard `*`?

46. What does `grep -E` provide?

47. What does `wc -l` count?

48. What does `wc -w` count?

49. What does `sort` do?

50. Why would you use `sort -n`?

51. How does `uniq` treat duplicates?

52. Why is `sort | uniq` common?

53. What does `uniq -c` do?

54. What does `cut` do?

55. What does `cut -d ':' -f 1` mean?

56. What does `tr` do?

57. What does `tr -d` do?

58. What is `sed`?

59. What does `s/old/new/` mean in `sed`?

60. Does ordinary `sed 's/old/new/' file` necessarily modify `file`?

61. Why should `sed -i` be used carefully?

62. What is `awk`?

63. What does `$1` mean in `awk`?

64. What does `$0` mean?

65. What does `$NF` mean?

66. What does `awk -F ':'` do?

67. What is a pipe?

68. Why should complicated pipelines be built one stage at a time?

69. What is the basic difference between `grep`, `sed`, and `awk`?

70. Why should you be careful when printing configuration files containing secrets?

---

# 178. Knowledge Check Answers

1. Linux uses text extensively for configuration, logs, scripts, automation, and many forms of system data.

2. A file that defines settings controlling the behavior of a program or service.

3. A record of events generated by a system, service, or application.

4. A text file containing instructions interpreted by a shell or another interpreter.

5. Reads files and writes their contents to standard output, among other concatenation uses.

6. It can flood the terminal with a large amount of output.

7. It provides interactive paged viewing of text.

8. Press `q`.

9. Type `/pattern` and press Enter.

10. Displays the beginning of input.

11. Display the first five lines.

12. Displays the end of input.

13. Recent log events are commonly appended to the end.

14. Continues following appended data.

15. Usually `Ctrl+C`.

16. Standard output.

17. Redirects output to a file, normally replacing or truncating its previous contents.

18. It can destroy existing file contents if used against the wrong file.

19. Appends output to a file.

20. It provides more controlled and predictable formatting.

21.

```bash
nano FILE
```

22. `Ctrl+O`, then confirm the filename.

23. `Ctrl+X`.

24. `Ctrl+W`.

25. Your user may intentionally lack write permission.

26. A modal terminal text editor based on Vi.

27. Vim's main mode for navigation and editing commands.

28. The mode used for entering text.

29. Press `i` from Normal mode.

30. Press `Esc`.

31.

```text
:w
```

32.

```text
:wq
```

33.

```text
:q!
```

34. Selects lines matching a pattern.

35. Yes, normally.

36. Ignores case distinctions during matching.

37. Displays line numbers for matches.

38. Searches directories recursively.

39. Selects nonmatching lines.

40. Three lines of context before and after matches.

41. A pattern language used to describe text to match.

42. Beginning of a line.

43. End of a line.

44. Any single character in common regex usage.

45. No. Shell globbing and regular expressions use `*` differently.

46. Extended regular-expression syntax.

47. Lines.

48. Words.

49. Orders lines according to its sorting rules.

50. To compare values numerically.

51. It collapses adjacent duplicate lines.

52. Sorting places identical lines next to one another before `uniq` processes them.

53. Counts adjacent duplicate occurrences.

54. Extracts selected characters or delimiter-separated fields.

55. Use `:` as the delimiter and output field 1.

56. Translates or deletes characters from standard input.

57. Deletes specified characters.

58. A stream editor for selecting and transforming text.

59. Substitute the first matching `old` with `new` on each processed line by default.

60. No. It normally sends the transformed result to standard output.

61. Because it changes the file directly.

62. A text-processing language particularly useful for records, fields, filtering, and reporting.

63. First field.

64. Entire current record or line.

65. The last field.

66. Sets `:` as the input field separator.

67. A shell mechanism that connects one command's standard output to another command's standard input.

68. You can inspect each intermediate result and identify where an assumption or transformation goes wrong.

69. A useful beginner model is `grep` for selecting/searching lines, `sed` for transformations, and `awk` for field-oriented processing and logic.

70. Commands may expose passwords, tokens, API keys, private keys, and other sensitive values on screens, recordings, logs, or shared terminals.

---

# 179. The Mental Model to Keep

When you encounter an unfamiliar Linux text file, think:

```text
What is this file?
      |
      v
Configuration, log, script, or data?
      |
      v
Do I need to read or modify it?
      |
      +----------------+
      |                |
      v                v
     Read             Modify
      |                |
      v                v
cat / less        nano / vim
head / tail            |
      |                v
      |             Save safely
      v                |
Search                 v
grep                 Validate
      |
      v
Need specific fields?
      |
      v
cut / awk
      |
      v
Need transformation?
      |
      v
tr / sed / awk
      |
      v
Need counts or summaries?
      |
      v
wc / sort / uniq
      |
      v
Combine carefully with pipes
      |
      v
Inspect every stage
      |
      v
Use evidence to make a decision
```

The goal is not to memorize a collection of commands.

The goal is to become comfortable looking at text and asking:

```text
What information do I need?

Where is it?

How can I inspect it safely?

How can I narrow the data?

How can I extract the useful part?

How can I verify that my interpretation is correct?
```

Once you develop that mindset, Linux configuration files and logs stop looking like walls of text. They become searchable evidence that you can inspect, transform, and reason about systematically.
