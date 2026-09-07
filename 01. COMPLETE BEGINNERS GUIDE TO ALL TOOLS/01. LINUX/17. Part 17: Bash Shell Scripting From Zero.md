# Part 17: Bash Shell Scripting From Zero

In the previous parts, you learned how to work with files, permissions, users, processes, services, packages, networking, storage, system resources, and logs.

Until now, you have mostly performed those tasks manually.

You might run:

```bash
df -h
free -h
uptime
systemctl status nginx
journalctl -u nginx
```

one command at a time.

But what if you need to perform the same checks every morning?

What if you manage 20 servers?

What if a deployment requires 15 commands in the correct order?

This is where shell scripting becomes useful.

A script lets you take commands and logic that you would normally execute manually and place them into a reusable file.

By the end of Part 17, you should be able to create Bash scripts from scratch, use variables and user input, make decisions with conditions, repeat tasks with loops, organize scripts with functions, understand exit codes, perform basic error handling, debug scripts, and build a practical Linux system health-check script.

---

# 1. What Is a Shell?

You have already been using a shell throughout this Linux series.

When you type:

```bash
pwd
```

or:

```bash
ls -la
```

or:

```bash
systemctl status nginx
```

something has to interpret what you typed.

That program is the shell.

A simplified flow looks like this:

```text
You
 |
 v
Terminal
 |
 v
Shell
 |
 v
Linux programs and system calls
 |
 v
Kernel
 |
 v
Hardware
```

The terminal provides the interface where you type.

The shell interprets your commands.

Linux then runs the requested programs.

---

# 2. What Is Bash?

Bash stands for:

```text
Bourne Again Shell
```

It is one of the most widely used Unix-style shells.

You can check your configured login shell with:

```bash
echo "$SHELL"
```

You might see:

```text
/bin/bash
```

You can also check the current Bash version when running Bash:

```bash
bash --version
```

Example:

```text
GNU bash, version 5.x.x
```

The exact version depends on your system.

---

# 3. Shell vs Bash

These terms should not be treated as identical.

A shell is a category of command interpreter.

Bash is one particular shell.

Other shells include:

```text
sh
zsh
ksh
fish
dash
```

Think:

```text
Shell
 |
 +-- Bash
 +-- Zsh
 +-- Ksh
 +-- Fish
 +-- Dash
```

A script written specifically for Bash may use features that another shell does not understand.

That becomes important when we discuss the shebang.

---

# 4. What Is Shell Scripting?

Shell scripting means writing shell commands and programming logic into a file so they can be executed together.

Instead of manually typing:

```bash
date
hostname
uptime
free -h
df -h
```

every time you inspect a server, you could put them into a script.

For example:

```bash
#!/bin/bash

date
hostname
uptime
free -h
df -h
```

Then run the script whenever you need the same information.

---

# 5. Manual Work vs Automation

Imagine checking five servers every morning.

On each server you run:

```bash
hostname
uptime
free -h
df -h
systemctl --failed
```

That is manageable once.

But repeated manual work creates several problems:

* You may forget a command.
* You may run commands in the wrong order.
* You may mistype something.
* The process takes time.
* Different administrators may perform different checks.
* Repeating the process across many servers becomes inefficient.

A script creates a repeatable procedure.

---

# 6. Why Linux Administrators Automate Tasks

Linux administrators use scripts for tasks such as:

* System health checks.
* Disk-space checks.
* Log analysis.
* File cleanup.
* Backup preparation.
* User administration.
* Service checks.
* Application startup.
* Deployment tasks.
* Configuration validation.
* Report generation.
* Monitoring helpers.
* Scheduled maintenance.
* File processing.

DevOps engineers may use Bash inside:

* CI/CD pipelines.
* Docker images.
* Cloud initialization.
* Deployment workflows.
* Infrastructure automation.
* GitHub Actions.
* Jenkins pipelines.
* Kubernetes administration scripts.

Bash is particularly useful for connecting Linux commands into repeatable workflows.

---

# 7. When Bash Is a Good Choice

Bash works well when the task mainly involves:

* Running Linux commands.
* Working with files.
* Checking services.
* Processing text.
* Managing processes.
* Calling command-line tools.
* Combining existing utilities.

For example:

```text
Check disk
    |
    v
Check memory
    |
    v
Check failed services
    |
    v
Search recent errors
    |
    v
Create report
```

Bash is well suited to this.

---

# 8. When Bash May Not Be the Best Choice

Bash can become difficult to maintain for large applications involving:

* Complex data structures.
* Large amounts of business logic.
* Extensive APIs.
* Complex JSON manipulation.
* Advanced concurrency.
* Large reusable software systems.

A language such as Python may be easier for those situations.

A useful beginner principle is:

```text
Linux command automation
        |
        v
Bash is often a strong choice.

Large software application
        |
        v
Consider a general-purpose programming language.
```

Do not try to turn every automation problem into a massive Bash script.

---

# 9. Creating Your First Bash Script

Create a practice directory:

```bash
mkdir -p ~/bash-lab
```

Enter it:

```bash
cd ~/bash-lab
```

Confirm:

```bash
pwd
```

You should see something similar to:

```text
/home/yourusername/bash-lab
```

Create your first script:

```bash
nano hello.sh
```

Enter:

```bash
#!/bin/bash

echo "Hello from Bash"
```

Save and exit.

---

# 10. What Is a `.sh` File?

The file:

```text
hello.sh
```

uses the `.sh` extension.

This communicates to humans that the file probably contains shell code.

However, Linux does not require `.sh` for an executable shell script.

A script could simply be named:

```text
health-check
```

instead of:

```text
health-check.sh
```

The extension is a naming convention.

It does not determine which interpreter executes the file.

---

# 11. What Is the Shebang?

The first line was:

```bash
#!/bin/bash
```

This is called a shebang.

It starts with:

```text
#!
```

followed by the interpreter path:

```text
/bin/bash
```

Together:

```text
#!/bin/bash
```

tells the operating system which interpreter should execute the file when the script itself is launched as an executable.

---

# 12. Why the Shebang Matters

Suppose your script contains Bash-specific syntax.

When you execute:

```bash
./script.sh
```

the system needs to know which interpreter should process it.

The shebang provides that information.

Conceptually:

```text
./script.sh
     |
     v
Read first line
     |
     v
#!/bin/bash
     |
     v
Start /bin/bash
     |
     v
Bash interprets script
```

---

# 13. `/bin/bash` vs `/usr/bin/env bash`

You may also encounter:

```bash
#!/usr/bin/env bash
```

This asks `env` to locate `bash` through the environment's `PATH`.

Both styles are common.

For this beginner series, we will primarily use:

```bash
#!/bin/bash
```

because it makes the interpreter path explicit and is common on Linux systems where Bash is installed there.

You should still understand that scripts in different environments may use other approaches.

---

# 14. Check Where Bash Is Installed

Run:

```bash
command -v bash
```

You may receive:

```text
/usr/bin/bash
```

or another path depending on the system.

You can also inspect:

```bash
ls -l /bin/bash
```

On some modern systems, `/bin` may itself point into `/usr/bin`.

---

# 15. Running a Script With Bash

Your script currently contains:

```bash
#!/bin/bash

echo "Hello from Bash"
```

Run:

```bash
bash hello.sh
```

Expected output:

```text
Hello from Bash
```

Here you explicitly started Bash and gave the file to Bash as an argument.

Conceptually:

```text
bash hello.sh
 |      |
 |      +-- Script file
 |
 +-- Interpreter
```

---

# 16. Running a Script Directly

Try:

```bash
./hello.sh
```

You may receive:

```text
Permission denied
```

Why?

Because the file may not have execute permission.

Check:

```bash
ls -l hello.sh
```

You might see:

```text
-rw-r--r-- 1 ann ann 36 Aug 27 10:00 hello.sh
```

Notice there is no:

```text
x
```

permission.

---

# 17. Making a Script Executable

Run:

```bash
chmod +x hello.sh
```

Now:

```bash
ls -l hello.sh
```

might show:

```text
-rwxr-xr-x 1 ann ann 36 Aug 27 10:00 hello.sh
```

Run:

```bash
./hello.sh
```

Expected:

```text
Hello from Bash
```

---

# 18. `bash script.sh` vs `./script.sh`

These are related but different.

When you run:

```bash
bash script.sh
```

you explicitly launch Bash and tell Bash to read `script.sh`.

The script itself does not need execute permission for Bash to read it, although you still need appropriate read/access permissions.

When you run:

```bash
./script.sh
```

you are asking the operating system to execute the file directly.

For that:

* The file needs execute permission.
* The shebang should identify the interpreter.

---

# 19. Why `./` Is Required

Suppose:

```bash
hello.sh
```

returns:

```text
command not found
```

but:

```bash
./hello.sh
```

works.

Why?

Because the shell normally searches directories listed in:

```bash
$PATH
```

for commands.

Your current directory is commonly not included in `PATH`.

The:

```text
./
```

means:

```text
Run the file named hello.sh from the current directory.
```

---

# 20. Comments

Comments explain code to humans.

In Bash:

```bash
# This is a comment
```

Bash ignores it.

Example:

```bash
#!/bin/bash

# Display the current hostname
hostname

# Display current disk usage
df -h
```

Comments should explain useful intent.

Avoid comments that merely repeat obvious code.

Weak:

```bash
# Run date
date
```

Better:

```bash
# Record when this health check was executed
date
```

---

# 21. The Shebang Is Special

Although:

```text
#
```

normally begins a comment, the first line:

```bash
#!/bin/bash
```

has a special purpose when the script is executed directly.

Do not treat it as an ordinary explanatory comment.

---

# 22. Variables

A variable stores a value under a name.

Example:

```bash
server_name="web01"
```

Now:

```text
server_name
```

is the variable name.

```text
web01
```

is its value.

Display it:

```bash
echo "$server_name"
```

Output:

```text
web01
```

---

# 23. Variable Assignment Syntax

Correct:

```bash
environment="production"
```

Incorrect:

```bash
environment = "production"
```

Bash does not allow spaces around `=` during ordinary variable assignment.

The second version is interpreted as something entirely different, usually an attempt to execute a command named `environment`.

---

# 24. Referencing Variables

Create:

```bash
name="Linux"
```

Then:

```bash
echo "$name"
```

Output:

```text
Linux
```

The:

```text
$
```

asks Bash to expand the variable and substitute its value.

---

# 25. Why Quote Variables?

Prefer:

```bash
echo "$name"
```

rather than:

```bash
echo $name
```

The difference becomes important when values contain:

* Spaces.
* Wildcards.
* Empty strings.
* Other special characters.

Example:

```bash
directory="Production Logs"
```

Correct:

```bash
ls "$directory"
```

Without quotes:

```bash
ls $directory
```

Bash may treat it as two separate arguments:

```text
Production
Logs
```

instead of one path.

A strong Bash habit is:

```text
Quote variable expansions unless you specifically need another behavior.
```

---

# 26. Single Quotes vs Double Quotes

Consider:

```bash
name="Ann"
```

Now:

```bash
echo "$name"
```

prints:

```text
Ann
```

But:

```bash
echo '$name'
```

prints:

```text
$name
```

Double quotes allow variable expansion.

Single quotes preserve the text literally.

---

# 27. Combining Variables With Text

Suppose:

```bash
service="nginx"
```

You can write:

```bash
echo "Checking $service"
```

Output:

```text
Checking nginx
```

Sometimes braces improve clarity:

```bash
echo "Checking ${service}_status"
```

Without braces:

```bash
echo "$service_status"
```

Bash would look for a variable named:

```text
service_status
```

rather than `service` followed by `_status`.

---

# 28. Command Substitution

Sometimes you want to store the output of a command.

Use:

```bash
$(command)
```

Example:

```bash
current_date=$(date)
```

Then:

```bash
echo "$current_date"
```

Another example:

```bash
server=$(hostname)
```

Then:

```bash
echo "Checking server: $server"
```

---

# 29. What Happens During Command Substitution?

For:

```bash
server=$(hostname)
```

Bash effectively does:

```text
Run hostname
      |
      v
Capture stdout
      |
      v
Store result in server
```

Then:

```bash
echo "$server"
```

uses the captured value.

---

# 30. User Input With `read`

A script can ask the user for input.

Example:

```bash
#!/bin/bash

echo "Enter your name:"
read name

echo "Hello, $name"
```

Run:

```bash
bash greeting.sh
```

You might see:

```text
Enter your name:
Ann
Hello, Ann
```

---

# 31. Using `read -p`

You can combine the prompt and input:

```bash
read -p "Enter your name: " name
```

Then:

```bash
echo "Hello, $name"
```

This is more compact.

---

# 32. Do Not Echo Secrets Carelessly

Suppose you ask for a password.

This is poor:

```bash
read -p "Password: " password
echo "Your password is $password"
```

Sensitive values should not normally be displayed.

Bash supports silent input:

```bash
read -s -p "Password: " password
echo
```

The:

```text
-s
```

prevents the typed characters from being echoed to the terminal.

But remember, safely handling secrets involves much more than hiding terminal input.

Avoid unnecessarily storing secrets in scripts, command histories, logs, or plain-text files.

---

# 33. Positional Parameters

A script can receive arguments from the command line.

Suppose:

```bash
./check-service.sh nginx
```

Inside the script:

```text
$1
```

contains:

```text
nginx
```

If you run:

```bash
./script.sh production web01
```

then:

```text
$1 = production
$2 = web01
```

---

# 34. Positional Parameter Example

Create:

```bash
nano server-info.sh
```

Add:

```bash
#!/bin/bash

echo "Environment: $1"
echo "Server: $2"
```

Make executable:

```bash
chmod +x server-info.sh
```

Run:

```bash
./server-info.sh production web01
```

Output:

```text
Environment: production
Server: web01
```

---

# 35. What Is `$0`?

Inside a script:

```text
$0
```

normally represents how the script itself was invoked.

Example:

```bash
#!/bin/bash

echo "Script: $0"
echo "First argument: $1"
```

Run:

```bash
./demo.sh nginx
```

Possible output:

```text
Script: ./demo.sh
First argument: nginx
```

---

# 36. What Is `$@`?

`$@` represents all positional arguments.

Create:

```bash
#!/bin/bash

echo "Arguments: $@"
```

Run:

```bash
./demo.sh nginx production web01
```

Output:

```text
Arguments: nginx production web01
```

But in real scripts, the safest common form is:

```bash
"$@"
```

because it preserves each argument separately.

---

# 37. Why `"$@"` Matters

Suppose you run:

```bash
./demo.sh "Production Server" nginx
```

There are two arguments:

```text
Argument 1 = Production Server
Argument 2 = nginx
```

Using:

```bash
"$@"
```

preserves those argument boundaries.

This becomes especially important in loops and when passing arguments to other commands.

---

# 38. Number of Arguments

Bash provides:

```text
$#
```

which represents the number of positional arguments.

Example:

```bash
#!/bin/bash

echo "Number of arguments: $#"
```

Run:

```bash
./demo.sh nginx production web01
```

Output:

```text
Number of arguments: 3
```

---

# 39. Validating Arguments

Suppose your script requires a service name.

You should not blindly assume `$1` exists.

Example:

```bash
#!/bin/bash

if [ "$#" -lt 1 ]; then
    echo "Usage: $0 SERVICE"
    exit 1
fi

service="$1"

echo "Checking service: $service"
```

Now the script explains how it should be used.

---

# 40. Exit Status

Linux commands normally return an exit status after execution.

You can inspect the previous command's status using:

```bash
echo $?
```

By convention:

```text
0
=
success
```

A non-zero value usually represents some type of failure or alternative condition.

The exact meaning depends on the command.

---

# 41. Exit Status Example

Run:

```bash
ls /tmp
```

Then:

```bash
echo $?
```

If successful:

```text
0
```

Now:

```bash
ls /directory-that-does-not-exist
```

Then immediately:

```bash
echo $?
```

You should receive a non-zero value.

---

# 42. Why `$?` Must Be Checked Immediately

Suppose:

```bash
ls /does-not-exist
echo "Finished"
echo $?
```

The final:

```bash
echo $?
```

reports the exit status of:

```bash
echo "Finished"
```

not the earlier `ls`.

So if you need the result:

```bash
ls /does-not-exist
status=$?

echo "ls returned: $status"
```

Capture it immediately.

---

# 43. Conditional Statements

Scripts often need to make decisions.

Example:

```text
If disk usage is high
    |
    v
Print warning

Otherwise
    |
    v
Print healthy
```

Bash provides:

```text
if
elif
else
fi
```

---

# 44. Basic `if` Structure

```bash
if condition; then
    commands
fi
```

Example:

```bash
if [ -f "/etc/passwd" ]; then
    echo "/etc/passwd exists"
fi
```

Notice:

```text
if
...
then
...
fi
```

`fi` closes the `if` statement.

---

# 45. `if` With `else`

```bash
if [ -f "/etc/passwd" ]; then
    echo "File exists"
else
    echo "File does not exist"
fi
```

Flow:

```text
Does file exist?
     |
 +---+---+
 |       |
Yes      No
 |       |
 v       v
Print   Print
exists  missing
```

---

# 46. `elif`

Use `elif` when there are multiple conditions.

Example:

```bash
usage=75

if [ "$usage" -ge 90 ]; then
    echo "Critical"
elif [ "$usage" -ge 80 ]; then
    echo "Warning"
else
    echo "Healthy"
fi
```

For:

```text
75
```

the output is:

```text
Healthy
```

---

# 47. What Is `[ ... ]`?

You will frequently see:

```bash
[ condition ]
```

The `[` is actually a command, commonly provided as a shell builtin as well as an external utility on many systems.

It evaluates an expression and returns an exit status.

For example:

```bash
[ 5 -gt 3 ]
```

Then:

```bash
echo $?
```

returns:

```text
0
```

because the comparison is true.

Spaces are important:

```bash
[ 5 -gt 3 ]
```

not:

```bash
[5 -gt 3]
```

---

# 48. Numeric Comparisons

Common numeric comparison operators include:

```text
-eq    equal
-ne    not equal
-gt    greater than
-ge    greater than or equal
-lt    less than
-le    less than or equal
```

Example:

```bash
cpu=90

if [ "$cpu" -gt 80 ]; then
    echo "CPU usage is high"
fi
```

---

# 49. Numeric Comparison Example

```bash
count=5

if [ "$count" -eq 5 ]; then
    echo "Count is five"
fi
```

Do not use:

```bash
[ "$count" > 5 ]
```

for this basic numeric comparison.

Use:

```bash
[ "$count" -gt 5 ]
```

---

# 50. Arithmetic With `(( ... ))`

Bash also provides arithmetic contexts.

Example:

```bash
count=10

if (( count > 5 )); then
    echo "Count is greater than five"
fi
```

For Bash arithmetic, this is often easier to read than:

```bash
if [ "$count" -gt 5 ]; then
```

Both are useful to recognize.

---

# 51. String Comparisons

Basic string comparisons include:

```text
=     equal
!=    not equal
-z    string is empty
-n    string is not empty
```

Example:

```bash
environment="production"

if [ "$environment" = "production" ]; then
    echo "Production environment"
fi
```

---

# 52. Checking for an Empty String

```bash
name=""

if [ -z "$name" ]; then
    echo "Name is empty"
fi
```

For non-empty:

```bash
if [ -n "$name" ]; then
    echo "Name has a value"
fi
```

Quoting variables is especially important here.

---

# 53. `[[ ... ]]` in Bash

Because this is a Bash scripting section, you will also frequently encounter:

```bash
[[ condition ]]
```

Example:

```bash
environment="production"

if [[ "$environment" == "production" ]]; then
    echo "Production environment"
fi
```

`[[ ... ]]` is Bash syntax with safer and more powerful behavior for many string tests than the traditional `[ ... ]`.

For Bash-specific scripts, you will often see it used.

You should still understand `[ ... ]` because it appears everywhere in shell scripts.

---

# 54. File Tests

Scripts frequently need to inspect files.

Useful tests include:

```text
-e    exists
-f    regular file
-d    directory
-r    readable
-w    writable
-x    executable
-s    exists and has non-zero size
```

Example:

```bash
config="/etc/myapp/config.conf"

if [ -f "$config" ]; then
    echo "Configuration file exists"
else
    echo "Configuration file missing"
fi
```

---

# 55. Directory Test

```bash
directory="/var/log"

if [ -d "$directory" ]; then
    echo "$directory is a directory"
fi
```

---

# 56. Executable Test

```bash
script="./deploy.sh"

if [ -x "$script" ]; then
    echo "Script is executable"
else
    echo "Script is not executable"
fi
```

---

# 57. Testing Whether a Command Exists

A useful administrative check is:

```bash
if command -v nginx >/dev/null 2>&1; then
    echo "nginx is installed"
else
    echo "nginx was not found"
fi
```

You learned redirection earlier.

Here:

```text
>/dev/null
```

discards standard output.

```text
2>&1
```

sends standard error to the same destination.

The script cares about the exit status, not the displayed output.

---

# 58. Testing Command Success Directly

You do not always need to manually inspect `$?`.

Instead of:

```bash
systemctl is-active nginx
status=$?

if [ "$status" -eq 0 ]; then
    echo "Running"
fi
```

you can often write:

```bash
if systemctl is-active --quiet nginx; then
    echo "nginx is running"
else
    echo "nginx is not running"
fi
```

This is an important Bash pattern.

`if` evaluates the command's exit status.

---

# 59. `for` Loops

Loops repeat commands.

Suppose you want to check several services:

```text
nginx
ssh
cron
```

Instead of repeating code, use a loop.

---

# 60. Basic `for` Loop

```bash
for service in nginx ssh cron; do
    echo "Checking $service"
done
```

Conceptually:

```text
Take nginx
    |
    v
Run body

Take ssh
    |
    v
Run body

Take cron
    |
    v
Run body
```

---

# 61. Looping Through Arguments

Remember:

```text
"$@"
```

represents all script arguments while preserving their boundaries.

You can write:

```bash
for service in "$@"; do
    echo "Checking: $service"
done
```

Run:

```bash
./check.sh nginx ssh cron
```

The loop processes each argument separately.

---

# 62. Looping Through Files

Example:

```bash
for file in *.log; do
    echo "Found log: $file"
done
```

Be careful with filename assumptions and unmatched patterns in real scripts.

Filenames can contain spaces and other unusual characters.

Production-quality file processing sometimes requires more careful techniques than a simple glob loop.

---

# 63. `while` Loops

A `while` loop continues while a condition remains true.

Example:

```bash
count=1

while [ "$count" -le 5 ]; do
    echo "Count: $count"
    count=$((count + 1))
done
```

Output:

```text
Count: 1
Count: 2
Count: 3
Count: 4
Count: 5
```

---

# 64. Arithmetic Expansion

This:

```bash
count=$((count + 1))
```

uses arithmetic expansion.

The expression:

```text
$(( ... ))
```

allows Bash to perform integer arithmetic.

Examples:

```bash
total=$((5 + 3))
```

```bash
remaining=$((100 - 75))
```

```bash
double=$((count * 2))
```

---

# 65. Infinite Loops

Be careful with conditions that never become false.

Example:

```bash
while true; do
    echo "Running"
done
```

This continues indefinitely until interrupted or otherwise terminated.

Stop an interactive instance with:

```text
Ctrl+C
```

Infinite loops can be intentional, but beginners should understand what they are creating before using them.

---

# 66. `break`

`break` exits a loop.

Example:

```bash
count=1

while true; do
    echo "$count"

    if [ "$count" -eq 5 ]; then
        break
    fi

    count=$((count + 1))
done
```

When `count` reaches 5, the loop stops.

---

# 67. `continue`

`continue` skips the remainder of the current iteration and moves to the next one.

Example:

```bash
for number in 1 2 3 4 5; do
    if [ "$number" -eq 3 ]; then
        continue
    fi

    echo "$number"
done
```

Output:

```text
1
2
4
5
```

---

# 68. Functions

Functions let you group reusable code.

Without functions:

```bash
echo "Checking disk"
df -h

echo "Checking memory"
free -h
```

As scripts grow, functions make responsibilities clearer.

---

# 69. Creating a Function

```bash
check_disk() {
    echo "Checking disk usage"
    df -h
}
```

Defining the function does not automatically run it.

Call it:

```bash
check_disk
```

Complete example:

```bash
#!/bin/bash

check_disk() {
    echo "Checking disk usage"
    df -h
}

check_disk
```

---

# 70. Function Arguments

Functions can receive positional arguments too.

Example:

```bash
check_service() {
    service="$1"
    echo "Checking $service"
}
```

Call:

```bash
check_service nginx
```

Inside the function:

```text
$1
```

is:

```text
nginx
```

---

# 71. Local Variables

Variables are global within a Bash script by default unless you limit their scope.

Inside functions, use:

```bash
local service="$1"
```

Example:

```bash
check_service() {
    local service="$1"
    echo "Checking $service"
}
```

This reduces accidental interference with variables elsewhere in the script.

---

# 72. Functions and Return Status

A Bash function returns an exit status.

Example:

```bash
check_directory() {
    local directory="$1"

    if [ -d "$directory" ]; then
        return 0
    else
        return 1
    fi
}
```

Then:

```bash
if check_directory "/var/log"; then
    echo "Directory exists"
else
    echo "Directory missing"
fi
```

---

# 73. `return` vs `exit`

This distinction is important.

Inside a function:

```bash
return 1
```

returns control from the function.

But:

```bash
exit 1
```

terminates the entire script.

Think:

```text
return
=
leave function

exit
=
leave script
```

---

# 74. Exit Codes

Your script should communicate whether it succeeded.

Example:

```bash
#!/bin/bash

echo "Health check completed"

exit 0
```

The:

```text
0
```

means success by convention.

For failure:

```bash
exit 1
```

The exact non-zero codes are up to your script unless you are following a defined convention.

---

# 75. Why Exit Codes Matter in DevOps

Suppose a CI/CD pipeline runs:

```bash
./deploy.sh
```

If the script fails internally but still exits:

```text
0
```

the pipeline may think deployment succeeded.

Conceptually:

```text
Script
  |
  v
Exit code
  |
  +-- 0 ------> Pipeline sees success
  |
  +-- nonzero -> Pipeline sees failure
```

Exit codes are part of how automation systems communicate.

---

# 76. Example of a Bad Script

```bash
#!/bin/bash

cp application.conf /etc/myapp/application.conf

echo "Deployment successful"

exit 0
```

What happens if `cp` fails?

The script may still print:

```text
Deployment successful
```

and explicitly return success.

That is misleading.

---

# 77. Better Error Checking

```bash
#!/bin/bash

if cp application.conf /etc/myapp/application.conf; then
    echo "Configuration copied successfully"
else
    echo "ERROR: Failed to copy configuration" >&2
    exit 1
fi
```

Now failure is handled explicitly.

Notice:

```bash
>&2
```

sends the error message to standard error.

---

# 78. Standard Output vs Standard Error in Scripts

Use standard output for normal information:

```bash
echo "Health check started"
```

Use standard error for errors:

```bash
echo "ERROR: Configuration file missing" >&2
```

This matters because callers can redirect them separately.

Example:

```bash
./script.sh >output.log 2>error.log
```

---

# 79. Command Chaining in Scripts

You learned:

```text
&&
```

and:

```text
||
```

earlier.

Example:

```bash
mkdir backup && echo "Backup directory created"
```

The second command runs only if the first succeeds.

Example:

```bash
mkdir backup || echo "Could not create backup"
```

The second command runs if the first fails.

These can be useful for short logic, but do not build unreadable chains containing many operations.

For complicated decisions, use `if`.

---

# 80. Basic Error Handling

A good script should consider:

```text
What can fail?

How will I detect failure?

What should happen next?

Should the script stop?

Should it continue?

What should the user see?

What exit code should be returned?
```

Example:

```bash
config="/etc/myapp/config.conf"

if [ ! -f "$config" ]; then
    echo "ERROR: $config does not exist" >&2
    exit 1
fi
```

---

# 81. Negating a Test

The:

```text
!
```

can mean logical NOT.

Example:

```bash
if [ ! -f "$config" ]; then
```

means:

```text
If $config is NOT a regular file
```

Another example:

```bash
if ! command -v curl >/dev/null 2>&1; then
    echo "ERROR: curl is required" >&2
    exit 1
fi
```

---

# 82. Validate Before Acting

Suppose a script needs a directory.

Weak:

```bash
rm "$backup_dir"/*
```

Safer thinking:

```text
Is backup_dir set?

Does it contain the expected path?

Does the directory exist?

Should files actually be removed?

Could expansion produce an unexpected target?
```

Automation magnifies mistakes.

A typo typed manually may affect one command.

A script may repeat that mistake across:

* Thousands of files.
* Many servers.
* Production environments.

Validation is critical.

---

# 83. Dangerous Variables

Consider:

```bash
rm -rf "$target_dir"
```

If:

```text
target_dir
```

contains an unexpected value, the consequences can be severe.

Before destructive operations, scripts should validate assumptions aggressively.

For example:

```bash
if [ -z "$target_dir" ]; then
    echo "ERROR: target_dir is empty" >&2
    exit 1
fi
```

But even that alone may not be enough.

For destructive automation, validate the exact expected path and use safer designs wherever possible.

---

# 84. `set -e`

You may encounter:

```bash
set -e
```

This asks Bash to exit in many situations when a command returns a non-zero status.

Example:

```bash
#!/bin/bash

set -e

mkdir /some/path
cp source.txt /some/path/
echo "Completed"
```

If a relevant command fails, Bash may terminate before later commands execute.

This sounds like automatic error handling.

But there is an important warning.

---

# 85. `set -e` Is Not Magic

Do not assume:

```bash
set -e
```

means:

```text
Every possible failure will always terminate my script exactly as I expect.
```

Its behavior depends on syntactic context.

Commands involved in constructs such as:

* Conditions.
* Logical command lists.
* Pipelines.
* Other shell contexts.

may behave differently than a beginner expects.

Therefore:

```text
set -e
```

does not replace understanding exit statuses and designing error handling deliberately.

---

# 86. Example of `set -e` Context

Consider:

```bash
if grep "ERROR" application.log; then
    echo "Errors found"
else
    echo "No ERROR matches"
fi
```

`grep` uses its exit status as part of the condition.

A non-zero status here has meaning within the conditional logic.

This demonstrates why:

```text
non-zero
```

does not always mean:

```text
Catastrophic failure, terminate everything.
```

You must understand the command's semantics.

---

# 87. Pipelines and Failures

Consider:

```bash
some_command | grep "ERROR"
```

A pipeline contains multiple commands.

Which failures should matter?

Bash has an option:

```bash
set -o pipefail
```

With `pipefail`, a failure earlier in a pipeline can affect the pipeline's overall status rather than relying only on the final command's status.

You may see scripts begin with:

```bash
set -euo pipefail
```

This is popular, but beginners should understand each option before copying it.

---

# 88. What `set -u` Does

Although not required for our first scripts, you should recognize:

```bash
set -u
```

It treats many references to unset variables as errors.

This can catch mistakes such as misspelled variable names.

But like other strict-mode options, it should be used with understanding.

---

# 89. Do Not Copy "Strict Mode" Blindly

You may see:

```bash
set -euo pipefail
```

at the top of scripts online.

Do not think:

```text
This automatically makes every Bash script safe.
```

It does not.

It changes Bash behavior.

A good engineer understands:

* What each option does.
* Where failures are expected.
* How functions behave.
* How pipelines behave.
* Whether unset variables are legitimate.
* What cleanup is required when the script exits.

---

# 90. Debugging With `bash -x`

Suppose your script behaves unexpectedly.

Run:

```bash
bash -x script.sh
```

Bash displays commands as they are expanded and executed.

Example script:

```bash
name="Ann"
echo "Hello $name"
```

Debug output may resemble:

```text
+ name=Ann
+ echo 'Hello Ann'
Hello Ann
```

This lets you see what Bash is actually doing.

---

# 91. Why `bash -x` Is Useful

Suppose:

```bash
directory="$HOME/logs"

if [ -d "$directory" ]; then
    echo "Found directory"
fi
```

does not behave as expected.

Running:

```bash
bash -x script.sh
```

can reveal:

* Variable values.
* Expanded commands.
* Branches taken.
* Loop iterations.
* Function calls.

This is often much better than staring at the script and guessing.

---

# 92. Be Careful Debugging Secrets

There is a serious risk.

If your script contains:

```bash
password="secret"
```

or processes tokens and credentials, tracing may expose those values in terminal output or logs.

Do not casually use:

```bash
bash -x
```

against production scripts handling secrets.

Debug output itself may contain sensitive information.

---

# 93. Syntax Checking

Before executing a script, Bash can check syntax:

```bash
bash -n script.sh
```

The:

```text
-n
```

asks Bash to read commands without executing them.

This can catch syntax problems such as malformed control structures.

It does not prove the script's logic is correct.

---

# 94. Syntax Error Example

Suppose:

```bash
if [ -f "/etc/passwd" ]; then
    echo "Found"
```

You forgot:

```text
fi
```

Run:

```bash
bash -n script.sh
```

Bash should report a syntax error.

Fix:

```bash
if [ -f "/etc/passwd" ]; then
    echo "Found"
fi
```

---

# 95. Common Beginner Error: Missing Spaces

Incorrect:

```bash
if ["$name" = "Ann"]; then
```

Correct:

```bash
if [ "$name" = "Ann" ]; then
```

The spaces around the expression matter.

Remember that:

```text
[
```

behaves like a command.

Its arguments must be separated properly.

---

# 96. Common Beginner Error: Spaces Around Assignment

Incorrect:

```bash
name = "Ann"
```

Correct:

```bash
name="Ann"
```

This is one of the most common Bash mistakes.

---

# 97. Common Beginner Error: Forgetting `$`

Suppose:

```bash
name="Ann"
```

This:

```bash
echo name
```

prints:

```text
name
```

This:

```bash
echo "$name"
```

prints:

```text
Ann
```

Assignment uses:

```bash
name="Ann"
```

Reading the value commonly uses:

```bash
"$name"
```

---

# 98. Common Beginner Error: Not Quoting Variables

Suppose:

```bash
directory="Application Logs"
```

This:

```bash
cd $directory
```

may be interpreted as separate arguments.

Use:

```bash
cd "$directory"
```

---

# 99. Common Beginner Error: Wrong Interpreter

A script contains Bash syntax but begins:

```bash
#!/bin/sh
```

On some systems:

```text
/bin/sh
```

does not point to Bash.

The script may fail.

If you intentionally write Bash syntax, declare Bash appropriately.

---

# 100. Common Beginner Error: Permission Denied

You run:

```bash
./script.sh
```

and receive:

```text
Permission denied
```

Investigate:

```bash
ls -l script.sh
```

If execute permission is missing:

```bash
chmod +x script.sh
```

Then try again.

Do not blindly change permissions to `777`.

---

# 101. Common Beginner Error: Command Not Found

You run:

```bash
script.sh
```

and receive:

```text
command not found
```

But the file exists.

Check:

```bash
ls -l
```

Then run:

```bash
./script.sh
```

Remember, the current directory is not normally searched automatically unless it is in `PATH`.

---

# 102. Common Beginner Error: Bad Interpreter

You may encounter:

```text
bad interpreter: No such file or directory
```

Possible causes include:

* Incorrect shebang path.
* Interpreter not installed.
* File created with incompatible line endings.
* Typing error in the shebang.

Inspect:

```bash
head -n 1 script.sh
```

Check Bash:

```bash
command -v bash
```

---

# 103. Windows Line Endings

A script created or edited on Windows may contain CRLF line endings.

Linux commonly expects LF.

This can produce strange errors such as an interpreter path that appears to contain:

```text
^M
```

Inspect with tools such as:

```bash
file script.sh
```

or:

```bash
cat -A script.sh
```

If necessary, convert the file using an appropriate tool such as `dos2unix` if installed.

---

# 104. Writing Readable Scripts

A script should be understandable months later.

Prefer descriptive names.

Weak:

```bash
x=90
y=$(df ...)
```

Better:

```bash
disk_threshold=90
disk_usage=$(...)
```

Variable names should communicate purpose.

---

# 105. Use Consistent Formatting

Prefer:

```bash
if [ "$disk_usage" -ge "$disk_threshold" ]; then
    echo "WARNING: Disk usage is high"
else
    echo "Disk usage is healthy"
fi
```

Avoid compressed code such as:

```bash
if [ "$disk_usage" -ge "$disk_threshold" ];then echo "high";else echo "ok";fi
```

Both may work, but the first is easier to review and troubleshoot.

---

# 106. Organize Scripts Into Sections

A larger script might have:

```text
1. Shebang

2. Configuration variables

3. Functions

4. Input validation

5. Main execution logic

6. Final exit
```

Example:

```bash
#!/bin/bash

# Configuration
disk_threshold=80

# Functions
check_disk() {
    ...
}

check_memory() {
    ...
}

# Main
check_disk
check_memory

exit 0
```

---

# 107. Do Not Hide Errors

Avoid:

```bash
some_command 2>/dev/null
```

everywhere simply because error messages look messy.

Sometimes suppressing expected noise is appropriate.

But errors often contain the exact evidence you need.

Ask:

```text
Why am I suppressing this?

Is the error expected?

Could this hide a real failure?
```

---

# 108. Logging From Scripts

A script can produce its own log messages.

Example:

```bash
echo "$(date '+%Y-%m-%d %H:%M:%S') INFO Health check started"
```

Possible output:

```text
2026-08-27 10:30:00 INFO Health check started
```

You can create helper functions:

```bash
log_info() {
    echo "$(date '+%Y-%m-%d %H:%M:%S') INFO $*"
}
```

Then:

```bash
log_info "Checking disk usage"
```

---

# 109. Sending Errors to Standard Error

Create:

```bash
log_error() {
    echo "$(date '+%Y-%m-%d %H:%M:%S') ERROR $*" >&2
}
```

Then:

```bash
log_error "Unable to read configuration"
```

This separates normal messages from error messages.

---

# 110. Script Security Basics

Scripts can be dangerous because they automate privileges.

Consider:

```bash
sudo ./maintenance.sh
```

Every command inside that script may now execute with elevated privileges depending on how it is designed and invoked.

Before running scripts with elevated privileges:

* Read the script.
* Understand what it changes.
* Check ownership.
* Check permissions.
* Validate paths.
* Protect secrets.
* Avoid unnecessary root execution.
* Test in a disposable environment.

---

# 111. Never Store Plain-Text Secrets Without Need

Avoid:

```bash
password="SuperSecret123"
```

inside scripts committed to Git.

Also be careful with:

```bash
export API_TOKEN="..."
```

because environment variables can have security implications and may be exposed in some workflows.

Use proper secret-management mechanisms appropriate to the environment.

---

# 112. Be Careful With User Input

Suppose:

```bash
read -p "Enter directory: " directory
rm -rf "$directory"
```

The quotes are good, but the design remains dangerous.

A user could enter the wrong directory.

Before destructive operations:

* Restrict acceptable inputs.
* Validate expected paths.
* Confirm assumptions.
* Prefer non-destructive behavior.
* Use least privilege.
* Test carefully.

---

# 113. Project: Build a Linux System Health-Check Script

Now we will combine the concepts.

Our script will check:

* Hostname.
* Date.
* Uptime.
* Load average.
* CPU count.
* Memory.
* Disk usage.
* Inode usage.
* Failed services.
* Top CPU-consuming processes.
* Top memory-consuming processes.
* Recent high-priority journal messages.

We will build it progressively instead of dumping the final script immediately.

---

# 114. Project Step 1: Create the Project Directory

Run:

```bash
mkdir -p ~/bash-lab/system-health
```

Enter:

```bash
cd ~/bash-lab/system-health
```

Create:

```bash
nano health-check.sh
```

---

# 115. Project Step 2: Add the Shebang

Add:

```bash
#!/bin/bash
```

Then:

```bash
echo "Linux System Health Check"
```

Save.

Make executable:

```bash
chmod +x health-check.sh
```

Run:

```bash
./health-check.sh
```

Expected:

```text
Linux System Health Check
```

---

# 116. Project Step 3: Add Basic System Information

Modify:

```bash
#!/bin/bash

echo "Linux System Health Check"
echo "========================="
echo

echo "Hostname:"
hostname

echo
echo "Date:"
date

echo
echo "Uptime:"
uptime
```

Run:

```bash
./health-check.sh
```

Inspect the output.

---

# 117. Project Step 4: Use Variables

Instead of repeatedly calling values later, capture useful information:

```bash
hostname_value=$(hostname)
current_date=$(date)
cpu_count=$(nproc)
```

Then:

```bash
echo "Hostname: $hostname_value"
echo "Date: $current_date"
echo "CPU cores: $cpu_count"
```

---

# 118. Project Step 5: Add Functions

Create:

```bash
show_header() {
    echo
    echo "========================================"
    echo "$1"
    echo "========================================"
}
```

Now:

```bash
show_header "SYSTEM INFORMATION"
```

This avoids repeating formatting code.

---

# 119. Project Step 6: System Information Function

```bash
check_system_info() {
    show_header "SYSTEM INFORMATION"

    echo "Hostname: $(hostname)"
    echo "Date: $(date)"
    echo "Uptime:"
    uptime
    echo "CPU cores: $(nproc)"
}
```

Call:

```bash
check_system_info
```

---

# 120. Project Step 7: Memory Function

Create:

```bash
check_memory() {
    show_header "MEMORY"
    free -h
}
```

This gives a human-readable memory summary.

Remember from Part 15 that you should pay attention to available memory rather than assuming all cache usage means memory is unavailable.

---

# 121. Project Step 8: Disk Function

Create:

```bash
check_disk() {
    show_header "DISK USAGE"
    df -h
}
```

Now the script reports filesystem capacity.

---

# 122. Project Step 9: Inode Function

Create:

```bash
check_inodes() {
    show_header "INODE USAGE"
    df -i
}
```

This matters because a filesystem can run out of inodes even when byte capacity remains.

---

# 123. Project Step 10: Failed Services

Create:

```bash
check_failed_services() {
    show_header "FAILED SERVICES"

    if command -v systemctl >/dev/null 2>&1; then
        systemctl --failed --no-pager
    else
        echo "systemctl is not available on this system"
    fi
}
```

We first verify that:

```text
systemctl
```

exists.

---

# 124. Project Step 11: Top CPU Processes

Create:

```bash
check_cpu_processes() {
    show_header "TOP CPU PROCESSES"
    ps aux --sort=-%cpu | head -n 6
}
```

Why six?

The first line contains column headings.

The next five lines show five processes.

---

# 125. Project Step 12: Top Memory Processes

```bash
check_memory_processes() {
    show_header "TOP MEMORY PROCESSES"
    ps aux --sort=-%mem | head -n 6
}
```

Now the report provides quick process-level evidence.

---

# 126. Project Step 13: Recent Journal Problems

Create:

```bash
check_recent_logs() {
    show_header "RECENT WARNING AND ERROR LOGS"

    if command -v journalctl >/dev/null 2>&1; then
        journalctl -p warning --since "30 minutes ago" \
            --no-pager -n 20
    else
        echo "journalctl is not available on this system"
    fi
}
```

The backslash:

```text
\
```

continues a command onto the next line.

This can improve readability.

---

# 127. Project Step 14: Add a Disk Threshold

Displaying all disk usage is useful.

But we also want the script to identify filesystems above a threshold.

Set:

```bash
disk_threshold=80
```

This means:

```text
Warn when usage is 80% or higher.
```

---

# 128. Extracting Disk Percentages

`df` produces output such as:

```text
Filesystem      Size  Used Avail Use% Mounted on
/dev/xvda1       30G   18G   12G  60% /
```

We need:

```text
60%
```

and then:

```text
60
```

This requires text processing.

A simple approach is:

```bash
df -P
```

combined with `awk`.

For example:

```bash
df -P | awk 'NR>1 {print $5, $6}'
```

Possible output:

```text
60% /
35% /boot
```

---

# 129. Removing `%`

Inside `awk`, or using shell parameter expansion later, we can remove `%`.

For a shell variable:

```bash
usage="60%"
```

this:

```bash
usage="${usage%\%}"
```

removes the trailing percent sign.

Now:

```text
usage
```

contains:

```text
60
```

---

# 130. Disk Threshold Function

For a beginner-friendly Bash version:

```bash
check_disk_thresholds() {
    show_header "DISK THRESHOLD CHECK"

    df -P | awk 'NR > 1 {print $5, $6}' |
    while read -r usage mountpoint; do
        usage="${usage%\%}"

        if [ "$usage" -ge "$disk_threshold" ]; then
            echo "WARNING: $mountpoint is ${usage}% full"
        else
            echo "OK: $mountpoint is ${usage}% full"
        fi
    done
}
```

Let's understand every major part.

---

# 131. Understanding `df -P`

```bash
df -P
```

uses POSIX-style output formatting that is often easier to process predictably than human-readable output.

For calculations, we do not use:

```bash
df -h
```

because values such as:

```text
1.2G
800M
```

require additional parsing.

We only need the percentage.

---

# 132. Understanding the `awk`

```bash
awk 'NR > 1 {print $5, $6}'
```

means:

```text
NR > 1
```

skip the header line.

Then:

```text
print $5, $6
```

prints fields 5 and 6.

In typical `df -P` output, these correspond to:

```text
usage percentage
mount point
```

Always understand the actual output format before writing parsing logic around it.

---

# 133. Understanding `read -r`

This:

```bash
while read -r usage mountpoint
```

reads each incoming line into variables.

Example input:

```text
60% /
```

becomes:

```text
usage=60%
mountpoint=/
```

The:

```text
-r
```

prevents backslashes from being interpreted specially by `read`.

It is a good default for reading literal text.

---

# 134. Threshold Comparison

After removing `%`:

```bash
if [ "$usage" -ge "$disk_threshold" ]; then
```

compares two integers.

If:

```text
usage = 90
threshold = 80
```

then:

```text
90 >= 80
```

is true.

The script prints a warning.

---

# 135. Add Overall Status

We want the script to communicate whether serious conditions were detected.

We can begin with:

```bash
overall_status=0
```

Then functions can mark problems.

However, there is an important Bash issue.

A loop fed through a pipeline may run in a subshell depending on Bash behavior and settings.

For example:

```bash
df -P | while read ...
```

changes made inside that loop may not persist afterward.

This is exactly the kind of subtle behavior that makes larger Bash scripts more challenging.

---

# 136. Avoiding the Pipeline Subshell Problem

Instead, we can use process substitution:

```bash
while read -r usage mountpoint; do
    ...
done < <(df -P | awk 'NR > 1 {print $5, $6}')
```

The:

```text
<(command)
```

is process substitution.

It allows the loop to read command output without placing the loop itself on the right side of a normal pipeline.

This is a Bash-specific feature.

---

# 137. Improved Disk Threshold Function

```bash
check_disk_thresholds() {
    show_header "DISK THRESHOLD CHECK"

    while read -r usage mountpoint; do
        usage="${usage%\%}"

        if [ "$usage" -ge "$disk_threshold" ]; then
            echo "WARNING: $mountpoint is ${usage}% full"
            overall_status=1
        else
            echo "OK: $mountpoint is ${usage}% full"
        fi
    done < <(df -P | awk 'NR > 1 {print $5, $6}')
}
```

Now:

```text
overall_status
```

can remain changed after the loop.

---

# 138. Check Required Commands

A script should not assume every utility exists.

Create:

```bash
require_command() {
    local command_name="$1"

    if ! command -v "$command_name" >/dev/null 2>&1; then
        echo "ERROR: Required command '$command_name' was not found" >&2
        return 1
    fi
}
```

Then:

```bash
require_command df
require_command free
require_command ps
```

---

# 139. Checking Multiple Dependencies

We can create:

```bash
check_dependencies() {
    local missing=0

    for command_name in df free ps awk head hostname date uptime nproc; do
        if ! command -v "$command_name" >/dev/null 2>&1; then
            echo "ERROR: Missing command: $command_name" >&2
            missing=1
        fi
    done

    if [ "$missing" -ne 0 ]; then
        return 1
    fi

    return 0
}
```

Now the script validates its basic environment.

---

# 140. Main Execution Section

At the bottom of the script:

```bash
main() {
    check_dependencies || exit 1

    check_system_info
    check_memory
    check_disk
    check_inodes
    check_disk_thresholds
    check_failed_services
    check_cpu_processes
    check_memory_processes
    check_recent_logs

    echo
    echo "Health check completed"

    return "$overall_status"
}

main
exit $?
```

This gives the script a clear entry point.

---

# 141. Why Use `main`?

Bash does not require a `main` function.

But using one can improve structure.

You can organize the file as:

```text
Configuration
     |
     v
Functions
     |
     v
main()
     |
     v
main
```

This makes larger scripts easier to read.

---

# 142. Complete Health-Check Script

Now combine the pieces:

```bash
#!/bin/bash

# Linux System Health Check
# Performs basic system checks and reports potential disk issues.

disk_threshold=80
overall_status=0

show_header() {
    echo
    echo "========================================"
    echo "$1"
    echo "========================================"
}

check_dependencies() {
    local missing=0

    for command_name in df free ps awk head hostname date uptime nproc; do
        if ! command -v "$command_name" >/dev/null 2>&1; then
            echo "ERROR: Missing command: $command_name" >&2
            missing=1
        fi
    done

    if [ "$missing" -ne 0 ]; then
        return 1
    fi

    return 0
}

check_system_info() {
    show_header "SYSTEM INFORMATION"

    echo "Hostname: $(hostname)"
    echo "Date: $(date)"
    echo "CPU cores: $(nproc)"
    echo
    echo "Uptime:"
    uptime
}

check_memory() {
    show_header "MEMORY"
    free -h
}

check_disk() {
    show_header "DISK USAGE"
    df -h
}

check_inodes() {
    show_header "INODE USAGE"
    df -i
}

check_disk_thresholds() {
    show_header "DISK THRESHOLD CHECK"

    while read -r usage mountpoint; do
        usage="${usage%\%}"

        if [ "$usage" -ge "$disk_threshold" ]; then
            echo "WARNING: $mountpoint is ${usage}% full"
            overall_status=1
        else
            echo "OK: $mountpoint is ${usage}% full"
        fi
    done < <(df -P | awk 'NR > 1 {print $5, $6}')
}

check_failed_services() {
    show_header "FAILED SERVICES"

    if command -v systemctl >/dev/null 2>&1; then
        systemctl --failed --no-pager
    else
        echo "systemctl is not available"
    fi
}

check_cpu_processes() {
    show_header "TOP CPU PROCESSES"
    ps aux --sort=-%cpu | head -n 6
}

check_memory_processes() {
    show_header "TOP MEMORY PROCESSES"
    ps aux --sort=-%mem | head -n 6
}

check_recent_logs() {
    show_header "RECENT WARNING AND ERROR LOGS"

    if command -v journalctl >/dev/null 2>&1; then
        journalctl -p warning --since "30 minutes ago" \
            --no-pager -n 20
    else
        echo "journalctl is not available"
    fi
}

main() {
    if ! check_dependencies; then
        echo "ERROR: Dependency check failed" >&2
        return 1
    fi

    check_system_info
    check_memory
    check_disk
    check_inodes
    check_disk_thresholds
    check_failed_services
    check_cpu_processes
    check_memory_processes
    check_recent_logs

    show_header "FINAL STATUS"

    if [ "$overall_status" -eq 0 ]; then
        echo "No disk threshold warnings detected."
    else
        echo "One or more disk usage warnings were detected."
    fi

    return "$overall_status"
}

main
exit $?
```

---

# 143. Validate the Script Before Running It

First:

```bash
bash -n health-check.sh
```

If there is no syntax error, Bash normally produces no output.

That means:

```text
Syntax check passed
```

It does not mean:

```text
Every command and every piece of logic is correct.
```

---

# 144. Make It Executable

Run:

```bash
chmod +x health-check.sh
```

Verify:

```bash
ls -l health-check.sh
```

You should see execute permissions.

Then:

```bash
./health-check.sh
```

---

# 145. Inspect the Exit Code

Immediately after:

```bash
echo $?
```

If no disk threshold warning was triggered:

```text
0
```

If the script detected a threshold condition:

```text
1
```

This makes the script useful to another automation system.

---

# 146. Debug the Script

Run:

```bash
bash -x health-check.sh
```

Watch:

* Variable assignments.
* Function calls.
* Command substitution.
* Conditions.
* Loop iterations.

Do this in your lab environment.

Remember that `bash -x` can expose sensitive values in other scripts.

---

# 147. Modify the Threshold

Change:

```bash
disk_threshold=80
```

temporarily to:

```bash
disk_threshold=1
```

Run:

```bash
./health-check.sh
```

Most non-empty filesystems should now exceed the threshold.

You should see warnings.

Then:

```bash
echo $?
```

You should see the script's non-zero status if the warning logic was triggered.

This is a safe way to test a branch without filling your disk.

Restore the threshold afterward.

---

# 148. Why We Test Failure Paths

Beginners often test only:

```text
Everything works.
```

But production scripts need testing for:

```text
File missing
Command missing
Permission denied
Service stopped
Bad argument
Disk threshold exceeded
Unexpected input
Command failure
```

A script is not properly tested simply because the happy path works.

---

# 149. Safely Breaking the Script

Instead of deleting a real command, temporarily add a fake dependency:

```bash
for command_name in df free ps fakecommand123; do
```

Run the script.

You should receive something like:

```text
ERROR: Missing command: fakecommand123
ERROR: Dependency check failed
```

Then:

```bash
echo $?
```

should be non-zero.

Restore the original dependency list afterward.

This teaches you to test failure handling deliberately.

---

# 150. Project Troubleshooting: Permission Denied

Problem:

```text
./health-check.sh: Permission denied
```

Investigate:

```bash
ls -l health-check.sh
```

Possible cause:

Execute permission is missing.

Fix:

```bash
chmod +x health-check.sh
```

Verify:

```bash
./health-check.sh
```

---

# 151. Project Troubleshooting: Command Not Found

Problem:

```text
health-check.sh: command not found
```

Check:

```bash
pwd
ls -l health-check.sh
```

If the file is in the current directory:

```bash
./health-check.sh
```

Remember that the current directory is normally not automatically searched through `PATH`.

---

# 152. Project Troubleshooting: Bad Interpreter

Problem:

```text
bad interpreter
```

Inspect:

```bash
head -n 1 health-check.sh
```

Expected:

```text
#!/bin/bash
```

Check:

```bash
command -v bash
```

Also inspect the file type:

```bash
file health-check.sh
```

Look for line-ending problems if the file came from another operating system.

---

# 153. Project Troubleshooting: Integer Expression Error

Suppose you see something involving:

```text
integer expression expected
```

Investigate the variable being compared.

Use:

```bash
bash -x health-check.sh
```

Check whether:

```text
usage
```

actually contains an integer.

Maybe it contains:

```text
80%
```

instead of:

```text
80
```

or unexpected output changed the parsing.

The lesson is:

```text
Do not assume parsed command output always has the shape you expect.
```

---

# 154. Project Troubleshooting: Permission Errors From Journal

A regular user may not have access to every journal entry.

If:

```bash
journalctl
```

shows limited information, understand that journal visibility depends on system permissions and configuration.

Do not solve every visibility issue by permanently running the entire script as root.

Ask which individual checks actually require elevated privileges.

Least privilege still applies to automation.

---

# 155. Project Troubleshooting: Different Distribution

Commands and service names vary.

For example:

```text
ssh
```

may be:

```text
sshd
```

depending on the distribution.

Some systems may not use systemd at all.

A robust script should understand its supported environment rather than pretending every Linux system is identical.

---

# 156. How to Think About Script Failures

Use the troubleshooting method from Part 16:

```text
Observe
   |
   v
Read exact error
   |
   v
Identify failing line
   |
   v
Inspect variables
   |
   v
Inspect command exit status
   |
   v
Form hypothesis
   |
   v
Test
   |
   v
Fix
   |
   v
Run again
   |
   v
Verify
```

Do not randomly rewrite the script.

---

# 157. Bash Scripting Mental Model

When Bash reads a script, many things may happen:

```text
Read script
    |
    v
Parse syntax
    |
    v
Expand variables
    |
    v
Perform substitutions
    |
    v
Evaluate conditions
    |
    v
Run commands
    |
    v
Receive exit statuses
    |
    v
Choose next action
```

A Bash script is not merely a text file containing commands.

It contains logic controlling how and when commands run.

---

# 158. From Commands to Automation

At the beginning of your Linux learning, you might have done:

```bash
df -h
```

Then you learned:

```bash
df -h | grep ...
```

Then:

```bash
usage=$(...)
```

Then:

```bash
if [ "$usage" -ge 80 ]; then
    ...
fi
```

Then:

```bash
for ...
```

Then:

```bash
check_disk() {
    ...
}
```

Finally:

```text
Gather data
    |
    v
Evaluate it
    |
    v
Make decision
    |
    v
Report status
    |
    v
Return exit code
```

That progression is the foundation of useful shell automation.

---

# 159. What You Should Now Understand

Before continuing, you should understand:

* What a shell is.
* What Bash is.
* The difference between a shell and Bash.
* What shell scripting means.
* Why Linux administrators automate tasks.
* When Bash is useful.
* When another programming language may be more appropriate.
* How to create a Bash script.
* What `.sh` means.
* Why `.sh` is not technically required.
* What a shebang is.
* What `#!/bin/bash` means.
* Why the interpreter matters.
* How to locate Bash.
* How to run a script with `bash script.sh`.
* How to execute a script using `./script.sh`.
* Why execute permission matters.
* How to use `chmod +x`.
* Why `./` may be necessary.
* What comments are.
* How variables work.
* Why spaces around `=` break ordinary assignments.
* How `$variable` expands a value.
* Why variables should usually be quoted.
* The difference between single and double quotes.
* How `${variable}` can remove ambiguity.
* What command substitution is.
* How `$(command)` works.
* How to collect user input with `read`.
* How `read -p` works.
* Why secret input needs special care.
* What positional parameters are.
* What `$0` represents.
* What `$1` and `$2` represent.
* What `$@` represents.
* Why `"$@"` is important.
* What `$#` represents.
* How to validate required arguments.
* What an exit status is.
* What `$?` represents.
* Why `$?` must be captured immediately when needed.
* Why zero conventionally means success.
* Why non-zero statuses need command-specific interpretation.
* How `if` works.
* How `elif` works.
* How `else` works.
* Why `fi` is required.
* What `[ ... ]` does.
* Why spaces around test expressions matter.
* Numeric comparisons.
* String comparisons.
* Empty-string tests.
* Bash arithmetic conditions.
* What `[[ ... ]]` is.
* File tests.
* How to test whether a file exists.
* How to test whether a directory exists.
* How to test whether a file is executable.
* How to test whether a command exists.
* How to use a command directly as an `if` condition.
* How `for` loops work.
* How to loop through script arguments.
* How `while` loops work.
* What arithmetic expansion is.
* What `break` does.
* What `continue` does.
* What functions are.
* How to define and call functions.
* How functions receive arguments.
* Why `local` variables are useful.
* How functions return statuses.
* The difference between `return` and `exit`.
* Why script exit codes matter to CI/CD systems.
* Why successful-looking output does not prove a command succeeded.
* How to send errors to standard error.
* How `&&` and `||` can control execution.
* Why complex logic should usually use readable conditions.
* What basic error handling means.
* How to negate conditions.
* Why scripts should validate before making changes.
* Why destructive variables require extreme care.
* What `set -e` does at a basic level.
* Why `set -e` has limitations.
* What `pipefail` changes.
* What `set -u` does.
* Why `set -euo pipefail` should not be copied blindly.
* How to debug using `bash -x`.
* Why tracing can expose secrets.
* How to check syntax with `bash -n`.
* Common Bash syntax mistakes.
* How line endings can break scripts.
* Why readable variable names matter.
* Why functions improve maintainability.
* Why errors should not be hidden without reason.
* How scripts can generate useful log messages.
* Why root privileges should be minimized.
* Why scripts should not contain unnecessary secrets.
* Why user input must be validated.
* How to build a basic health-check script.
* How to test successful and failure paths.
* How to debug script failures methodically.

---

# 160. Hands-On Practice

Complete these exercises yourself.

1. Create `~/bash-practice`.
2. Create `hello.sh`.
3. Add a Bash shebang.
4. Print `Hello Linux`.
5. Run it using `bash hello.sh`.
6. Check its permissions.
7. Make it executable.
8. Run it using `./hello.sh`.
9. Add three comments.
10. Create a variable called `server`.
11. Assign `web01`.
12. Print it.
13. Create a variable containing spaces.
14. Print it safely using quotes.
15. Compare single and double quotes.
16. Store `hostname` output in a variable.
17. Store `date` output in a variable.
18. Print both.
19. Ask the user for their name.
20. Print a greeting.
21. Ask for an environment name.
22. Print the selected environment.
23. Create a script that accepts one argument.
24. Print `$1`.
25. Add a second argument.
26. Print `$2`.
27. Print `$@`.
28. Print `$#`.
29. Print `$0`.
30. Reject execution when no argument is supplied.
31. Print a usage message.
32. Run a successful command.
33. Inspect `$?`.
34. Run a failing command.
35. Inspect `$?`.
36. Capture an exit status in a variable.
37. Write an `if` statement.
38. Add `else`.
39. Add `elif`.
40. Compare two integers.
41. Compare two strings.
42. Test for an empty string.
43. Test whether `/etc/passwd` exists.
44. Test whether `/var/log` is a directory.
45. Test whether your script is executable.
46. Test whether `bash` exists.
47. Test whether `curl` exists.
48. Write a `for` loop.
49. Loop through five words.
50. Loop through positional arguments.
51. Write a `while` loop.
52. Count from 1 through 10.
53. Use arithmetic expansion.
54. Use `break`.
55. Use `continue`.
56. Create a function.
57. Call the function.
58. Pass an argument to it.
59. Use a local variable.
60. Return success from a function.
61. Return failure from a function.
62. Test the function using `if`.
63. Create a script that exits `0`.
64. Verify its exit code.
65. Create one that exits `1`.
66. Verify its exit code.
67. Send an error message to stderr.
68. Redirect stdout and stderr separately.
69. Use `&&`.
70. Use `||`.
71. Validate a file before reading it.
72. Validate a directory before entering it.
73. Create a script using `set -e` in your lab.
74. Observe what happens when a simple command fails.
75. Research the context before assuming `set -e` catches everything.
76. Try `set -o pipefail`.
77. Run a script with `bash -x`.
78. Observe variable expansion.
79. Run `bash -n` against a valid script.
80. Intentionally remove `fi`.
81. Run `bash -n` again.
82. Repair the syntax.
83. Create a script with an invalid assignment containing spaces around `=`.
84. Observe the error.
85. Repair it.
86. Create a variable containing spaces.
87. Demonstrate why quoting matters.
88. Create the system health-check project.
89. Add system information.
90. Add memory information.
91. Add disk information.
92. Add inode information.
93. Add failed-service information.
94. Add top CPU processes.
95. Add top memory processes.
96. Add recent journal warnings.
97. Add a disk threshold.
98. Force the threshold branch safely by temporarily lowering the threshold.
99. Verify the final exit code.
100. Debug the complete script with `bash -x`.

---

# 161. Beginner Knowledge Check

1. What is a shell?
2. What is Bash?
3. Is Bash the only Linux shell?
4. What is shell scripting?
5. Why do administrators automate tasks?
6. Give three tasks suitable for Bash automation.
7. When might Python be more appropriate than Bash?
8. What does `.sh` indicate?
9. Is `.sh` required for a script to execute?
10. What is a shebang?
11. What does `#!/bin/bash` mean?
12. Why does the shebang matter when running `./script.sh`?
13. What does `bash script.sh` do?
14. What does `./script.sh` do?
15. Why can `bash script.sh` work when `./script.sh` returns permission denied?
16. How do you add execute permission?
17. Why do you often need `./`?
18. How do Bash comments begin?
19. What is a variable?
20. What is wrong with `name = "Ann"`?
21. How do you reference a variable?
22. Why should variable expansions usually be quoted?
23. What is the difference between single and double quotes?
24. What does `${name}` mean?
25. What is command substitution?
26. What does `server=$(hostname)` do?
27. What does `read` do?
28. What does `read -p` do?
29. Why should passwords not be echoed?
30. What is `$1`?
31. What is `$2`?
32. What is `$0`?
33. What is `$@`?
34. Why is `"$@"` important?
35. What is `$#`?
36. What is an exit status?
37. What does `$?` contain?
38. What does exit status `0` normally mean?
39. Does every non-zero exit status mean the same thing?
40. Why must `$?` be inspected immediately?
41. What does `if` do?
42. What does `elif` do?
43. What does `else` do?
44. What closes a Bash `if` statement?
45. What does `-eq` mean?
46. What does `-ne` mean?
47. What does `-gt` mean?
48. What does `-ge` mean?
49. What does `-lt` mean?
50. What does `-le` mean?
51. How do you compare strings?
52. What does `-z` test?
53. What does `-n` test?
54. What does `-f` test?
55. What does `-d` test?
56. What does `-e` test?
57. What does `-x` test?
58. What does `command -v` help determine?
59. Can `if` test a command's exit status directly?
60. What is a `for` loop?
61. What is a `while` loop?
62. What does `break` do?
63. What does `continue` do?
64. What does `$(( ... ))` do?
65. What is a function?
66. Why use functions?
67. What does `local` do inside a function?
68. What is the difference between `return` and `exit`?
69. Why do exit codes matter in CI/CD?
70. Why is printing "success" insufficient?
71. How do you send an error message to stderr?
72. What does `&&` do?
73. What does `||` do?
74. Why should scripts validate inputs?
75. Why are destructive commands especially dangerous inside scripts?
76. What does `set -e` attempt to do?
77. Why is `set -e` not complete error handling?
78. What does `set -o pipefail` affect?
79. What does `set -u` do?
80. Why should `set -euo pipefail` not be copied blindly?
81. What does `bash -x` do?
82. Why can `bash -x` be dangerous around secrets?
83. What does `bash -n` do?
84. Can `bash -n` detect logical errors?
85. Why are spaces important around `[ ... ]`?
86. Why are spaces not allowed around `=` in normal assignments?
87. What can cause a bad-interpreter error?
88. How can Windows line endings affect Bash scripts?
89. Why should variable names be descriptive?
90. Why should scripts use consistent formatting?
91. Why should you avoid hiding every error with `2>/dev/null`?
92. Why should scripts avoid unnecessary root privileges?
93. Why should secrets not be committed inside scripts?
94. Why should user input be treated carefully?
95. What information does the health-check project collect?
96. Why does it check both disk space and inodes?
97. Why does it return an exit status?
98. Why should failure paths be deliberately tested?
99. How should you troubleshoot a Bash script?
100. What is the main difference between manually running Linux commands and automating them with Bash?

---

# 162. Beginner Knowledge Check Answers

1. A command interpreter that receives commands and coordinates their execution.

2. Bash is the Bourne Again Shell, a widely used Unix-style shell.

3. No. Other shells include Zsh, Ksh, Fish, Dash, and others.

4. Writing shell commands and programming logic into reusable executable or interpreter-readable files.

5. To make repetitive work faster, consistent, repeatable, and less dependent on manual typing.

6. Examples include health checks, service checks, file processing, log analysis, and backup preparation.

7. When the task becomes a larger software system with complex data structures, extensive logic, APIs, or other requirements that Bash handles poorly.

8. It conventionally indicates a shell script.

9. No.

10. The special first line beginning with `#!` that identifies the script interpreter.

11. Use `/bin/bash` as the interpreter when the script is executed directly.

12. The operating system needs to know which interpreter should process the file.

13. Explicitly starts Bash and asks it to interpret `script.sh`.

14. Requests execution of the script located in the current directory.

15. `bash script.sh` reads the file through Bash, while direct execution requires execute permission.

16. `chmod +x script.sh`.

17. The current directory is commonly not included in `PATH`.

18. `#`.

19. A named value used by the script.

20. Normal Bash variable assignments cannot contain spaces around `=`.

21. Commonly with `$name` or `${name}`.

22. To preserve the value as intended when it contains spaces, empty values, wildcard characters, and other shell-sensitive content.

23. Double quotes permit expansions such as variables. Single quotes preserve their contents literally.

24. It expands the variable named `name`, with braces clearly delimiting the variable name.

25. Capturing the output of a command for substitution into another command or value.

26. Runs `hostname` and stores its standard output in `server`.

27. Reads input.

28. Displays a prompt and reads input.

29. It can expose sensitive information.

30. The first positional argument.

31. The second positional argument.

32. How the script itself was invoked.

33. All positional arguments.

34. It preserves separate argument boundaries.

35. The number of positional arguments.

36. A numerical status returned after a command, function, or script executes.

37. The exit status of the most recently completed relevant command.

38. Success by convention.

39. No. Commands can assign specific meanings to different non-zero statuses.

40. Another command will replace the value represented by `$?`.

41. Executes logic based on a command or condition's success or failure.

42. Adds another condition.

43. Provides the fallback branch.

44. `fi`.

45. Numerically equal.

46. Numerically not equal.

47. Numerically greater than.

48. Numerically greater than or equal.

49. Numerically less than.

50. Numerically less than or equal.

51. For example, `[ "$a" = "$b" ]` or Bash's `[[ "$a" == "$b" ]]`.

52. Whether a string has zero length.

53. Whether a string has non-zero length.

54. Whether a path is a regular file.

55. Whether a path is a directory.

56. Whether a path exists.

57. Whether a path is executable.

58. Whether a command can be located and which command would be used.

59. Yes.

60. A structure that repeats commands over a sequence of values.

61. A structure that repeats while its condition remains successful or true.

62. Leaves a loop.

63. Skips the rest of the current loop iteration.

64. Performs arithmetic expansion.

65. A named reusable block of commands.

66. To organize code, reduce repetition, and make scripts easier to maintain.

67. Creates a variable scoped to the function.

68. `return` leaves a function with a status. `exit` terminates the script.

69. Automation systems use exit statuses to determine success or failure.

70. A preceding command may have failed even though the script prints a success message.

71. For example, `echo "ERROR" >&2`.

72. Runs the next command when the previous command succeeds.

73. Runs the next command when the previous command fails.

74. Inputs and assumptions can be wrong, missing, or dangerous.

75. Automation can repeat or amplify destructive mistakes.

76. It causes Bash to exit in many contexts when a command fails.

77. Its behavior has contextual exceptions and does not replace deliberate error handling.

78. How pipeline exit status reflects failures among commands in the pipeline.

79. It treats many uses of unset variables as errors.

80. The options change shell behavior and can create unexpected results when the author does not understand them.

81. Runs Bash with execution tracing.

82. Expanded commands may reveal passwords, tokens, or other sensitive values.

83. Checks Bash syntax without executing commands.

84. No. Valid syntax can still contain incorrect logic.

85. `[` expects properly separated arguments.

86. Bash assignment syntax requires `name=value`.

87. Wrong shebang path, missing interpreter, line-ending problems, or a malformed first line.

88. CRLF endings can introduce carriage-return characters that Linux tools or the shebang interpreter path do not expect.

89. They make the script easier to understand and maintain.

90. Readable formatting makes logic and mistakes easier to identify.

91. You may hide important evidence needed for troubleshooting.

92. Scripts can perform many actions automatically, so excessive privileges increase the impact of mistakes or compromise.

93. Source repositories and other users may expose them.

94. Incorrect or malicious input can cause unintended behavior, especially when used in paths or commands.

95. Host information, uptime, CPU count, memory, disk, inodes, failed services, processes, and recent journal events.

96. A filesystem can run out of either storage capacity or inodes.

97. So humans and other automation can determine whether the check detected a problem.

98. A script that works only when everything is healthy has not been adequately tested.

99. Read the exact error, identify the failing line, inspect variables and exit statuses, form a hypothesis, use tools such as `bash -n` and `bash -x`, make a controlled correction, and verify.

100. Manual execution depends on a person repeatedly entering commands. A Bash script encodes those commands and the logic connecting them so the workflow can run consistently and repeatedly.

# 163. Final Bash Mental Model

Do not think of Bash scripting as:

```text
Put many commands in a file
```

Think of it as:

```text
INPUT
  |
  v
VALIDATION
  |
  v
VARIABLES
  |
  v
COMMANDS
  |
  v
EXIT STATUSES
  |
  v
DECISIONS
  |
  +------ if / elif / else
  |
  v
REPETITION
  |
  +------ for / while
  |
  v
FUNCTIONS
  |
  v
ERROR HANDLING
  |
  v
OUTPUT
  |
  v
EXIT CODE
```

And when something fails:

```text
Script fails
    |
    v
Read exact error
    |
    v
bash -n
    |
    +-- Syntax problem?
    |
    v
bash -x
    |
    v
Inspect expanded commands
    |
    v
Check variable values
    |
    v
Check command exit statuses
    |
    v
Form hypothesis
    |
    v
Test correction
    |
    v
Verify normal path
    |
    v
Verify failure path
```

The goal is not to memorize Bash syntax.

The goal is to reach the point where you can look at a manual Linux workflow and think:

```text
I understand each step.

I understand what can fail.

I understand how to detect those failures.

I can turn this into a readable, safe, repeatable script.
```

That is the foundation of Bash automation for Linux administration, DevOps, Cloud, SRE, and Platform Engineering.
