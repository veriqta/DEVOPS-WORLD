# Part 11: Linux Environment Variables and Shell Configuration

In Part 10, you learned how Linux installs and manages software through packages, repositories, APT, DNF, dpkg, and RPM.

Now we need to understand something that affects almost every command and application you run on Linux: the shell environment.

You have already used commands such as:

```bash
cd ~
echo "$PATH"
command -v nginx
```

But why does `~` know where your home directory is? How does Linux find `nginx` without you typing its full path? How does a program know your username, preferred shell, language, or application settings?

Much of this behavior comes from variables, environment variables, shell expansion, and shell configuration files.

By the end of Part 11, you should be able to:

* Understand what a process environment is.
* Read environment variables.
* Understand `$HOME`, `$USER`, `$SHELL`, and `$PATH`.
* Understand exactly how command lookup works.
* Create shell variables.
* Export environment variables.
* Understand temporary versus persistent configuration.
* Safely modify `PATH`.
* Create aliases.
* Understand `.bashrc` and `.profile`.
* Recognize login and interactive shells.
* Troubleshoot `command not found`.
* Diagnose incorrect or missing environment variables.
* Build and test your own shell configuration.

---

# 1. What Is an Environment?

In everyday language, an environment is the surrounding set of conditions in which something operates.

Linux uses the term in a more specific way.

When a program runs, it becomes a process. That process can receive a collection of named values called environment variables.

Conceptually:

```text
Shell process
|
+-- HOME=/home/ann
+-- USER=ann
+-- SHELL=/bin/bash
+-- PATH=/usr/local/bin:/usr/bin:/bin
|
v
Program started by shell
|
v
Program receives environment variables
```

The environment gives programs information they may need without requiring every value to be written directly into the command.

---

# 2. What Is an Environment Variable?

An environment variable is a named value available in the environment of a process.

Examples include:

```text
HOME=/home/ann
USER=ann
SHELL=/bin/bash
PATH=/usr/local/bin:/usr/bin:/bin
```

Each has:

```text
NAME=value
```

For example:

```text
HOME=/home/ann
```

contains:

```text
Variable name: HOME
Value: /home/ann
```

Environment variable names are conventionally written in uppercase, although Linux does not require every variable to follow that convention.

---

# 3. Why Programs Use Environment Variables

Imagine an application needs to know where your home directory is.

One bad design would be to hard-code:

```text
/home/ann
```

That fails for another user:

```text
/home/david
```

Instead, software can use environment information.

Conceptually:

```text
User Ann
HOME=/home/ann

User David
HOME=/home/david
```

The same software can operate in both environments.

Environment variables are commonly used for:

* User information.
* Paths.
* Language and locale settings.
* Application configuration.
* Development settings.
* Runtime behavior.
* Tool configuration.
* Temporary configuration.
* Proxy settings.
* Application modes.

You will encounter environment variables constantly in DevOps.

---

# 4. Environment Variables in DevOps

Environment variables appear throughout modern infrastructure.

For example, an application might receive:

```text
APP_ENV=production
LOG_LEVEL=info
DATABASE_HOST=db.internal
PORT=8080
```

A CI/CD pipeline might use:

```text
BUILD_NUMBER
CI
WORKSPACE
```

A container might receive:

```text
APP_PORT=8080
```

A cloud application might use environment variables to determine runtime configuration.

The exact variables depend on the software.

This means environment variables are not just a Bash topic.

They appear in:

* Linux.
* Docker.
* Kubernetes.
* CI/CD.
* Cloud platforms.
* Python.
* Node.js.
* Java applications.
* Terraform workflows.
* Build systems.
* Automation scripts.

Understanding them now will make later technologies much easier to understand.

---

# 5. Important Security Warning About Environment Variables

Environment variables are often used to pass configuration to applications.

You may encounter examples involving:

```text
API_KEY
DATABASE_PASSWORD
TOKEN
SECRET_KEY
```

Do not conclude that environment variables automatically make secrets secure.

Environment variables can potentially be exposed through:

* Debug output.
* Application logs.
* Process inspection under appropriate permissions.
* CI/CD logs.
* Shell history, depending on how values are assigned.
* Diagnostic dumps.
* Misconfigured applications.

Later security and platform topics will introduce proper secret-management approaches.

For now, remember:

```text
Environment variable
does not automatically mean
secure secret storage.
```

---

# 6. Viewing the Environment With `env`

Run:

```bash
env
```

You will probably see many lines:

```text
HOME=/home/ann
USER=ann
SHELL=/bin/bash
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
...
```

Your exact output will differ.

The `env` command can display environment variables available to the process.

Do not worry if you see variables you do not recognize.

---

# 7. Reading `env` Output

Suppose you see:

```text
USER=ann
HOME=/home/ann
```

Each line follows approximately:

```text
VARIABLE=value
```

The left side is the variable name.

The right side is its value.

Try:

```bash
env | less
```

This sends the output through a pipe to `less`.

You learned pipes in Part 7.

Now you are combining previous Linux skills.

---

# 8. Searching Environment Variables

Instead of reading everything:

```bash
env | grep HOME
```

You may receive:

```text
HOME=/home/ann
```

Try:

```bash
env | grep PATH
```

Be careful.

This may match more than the exact `PATH` variable because other variable names or values may contain the word `PATH`.

A more precise pattern is:

```bash
env | grep '^PATH='
```

Here:

```text
^
```

means the beginning of the line in this regular-expression context.

So you are effectively asking:

```text
Find a line beginning with PATH=
```

---

# 9. `printenv`

Another useful command is:

```bash
printenv
```

It prints environment information.

To retrieve one variable:

```bash
printenv HOME
```

Possible output:

```text
/home/ann
```

Try:

```bash
printenv USER
```

and:

```bash
printenv PATH
```

Unlike:

```bash
echo "$PATH"
```

you provide the variable name to `printenv` without `$`.

Compare:

```bash
printenv HOME
```

with:

```bash
echo "$HOME"
```

Both can display the value, but they work differently.

---

# 10. Why `$` Appears Before Variable Names

Suppose:

```bash
HOME=/home/ann
```

When you type:

```bash
echo HOME
```

the shell sees ordinary text:

```text
HOME
```

So it prints:

```text
HOME
```

But when you type:

```bash
echo "$HOME"
```

the shell recognizes:

```text
$HOME
```

as a request to expand the variable.

Conceptually:

```text
Command entered

echo "$HOME"
       |
       v
Shell looks up HOME
       |
       v
HOME=/home/ann
       |
       v
Shell substitutes value
       |
       v
echo "/home/ann"
       |
       v
Output
/home/ann
```

This process is called variable expansion, which is one form of shell expansion.

---

# 11. `$HOME`

`HOME` normally identifies the current user's home directory.

Run:

```bash
echo "$HOME"
```

You may see:

```text
/home/ann
```

For the root account, it is commonly:

```text
/root
```

You can also run:

```bash
printenv HOME
```

---

# 12. `$HOME` and `~`

Earlier you learned:

```bash
cd ~
```

takes you to your home directory.

You can also use:

```bash
cd "$HOME"
```

In a normal Bash session, these commonly take you to the same place.

Verify:

```bash
cd ~
pwd
```

Then:

```bash
cd "$HOME"
pwd
```

However, `~` expansion and `$HOME` variable expansion are separate shell mechanisms.

Do not think of `~` as simply text replaced with `$HOME` in every possible context.

For beginner usage, remember that both commonly help you refer to your home directory.

---

# 13. `$USER`

Run:

```bash
echo "$USER"
```

You may see:

```text
ann
```

Compare:

```bash
whoami
```

with:

```bash
echo "$USER"
```

They often appear to give the same answer, but they represent different concepts.

`whoami` determines the effective username associated with the current process credentials.

`$USER` is an environment variable.

Environment variables can be changed.

For example:

```bash
USER=banana
```

Then:

```bash
echo "$USER"
```

could display:

```text
banana
```

But:

```bash
whoami
```

would still report your actual effective user.

This is an important lesson:

```text
Environment variables describe values.

They are not necessarily authoritative proof of system identity.
```

---

# 14. `$SHELL`

Run:

```bash
echo "$SHELL"
```

You might see:

```text
/bin/bash
```

This variable commonly indicates the user's configured login shell.

Do not automatically interpret `$SHELL` as:

```text
The exact shell process currently executing this command.
```

For example, your configured login shell might be Bash while you temporarily start another shell.

So:

```bash
echo "$SHELL"
```

is useful, but understand what it represents.

---

# 15. What Is a Shell Variable?

Before going deeper into environment variables, we need an important distinction.

A shell variable is a variable maintained by the shell.

Create one:

```bash
COURSE="Linux Beginner Series"
```

Display it:

```bash
echo "$COURSE"
```

Output:

```text
Linux Beginner Series
```

This variable exists in your current shell.

But that does not automatically mean programs started by the shell receive it as an environment variable.

---

# 16. Creating Variables

The syntax is:

```bash
NAME=value
```

Example:

```bash
CITY=Pachuca
```

Then:

```bash
echo "$CITY"
```

Output:

```text
Pachuca
```

For a value containing spaces:

```bash
COURSE="Complete Linux Beginner Series"
```

Then:

```bash
echo "$COURSE"
```

Output:

```text
Complete Linux Beginner Series
```

---

# 17. Do Not Put Spaces Around `=`

Correct:

```bash
NAME="Ann"
```

Incorrect:

```bash
NAME = "Ann"
```

The shell does not interpret the second form as a variable assignment.

In shell syntax, spaces separate words and commands.

This is a common beginner error.

---

# 18. Variable Names

Common variable names look like:

```text
NAME
APP_ENV
DATABASE_HOST
LOG_LEVEL
PROJECT_DIR
```

A shell variable name generally begins with a letter or underscore and then contains letters, numbers, or underscores.

Examples:

```bash
PROJECT_NAME="webapp"
APP_PORT=8080
LOG_DIR="/var/log/myapp"
```

Names are case-sensitive.

These are different:

```text
PATH
Path
path
```

That matters greatly when working with `$PATH`.

---

# 19. Shell Variables vs Environment Variables

This distinction is fundamental.

Suppose:

```bash
PROJECT="veriqta"
```

Your current Bash shell knows `PROJECT`.

Check:

```bash
echo "$PROJECT"
```

You get:

```text
veriqta
```

But run:

```bash
printenv PROJECT
```

You may get no output.

Why?

Because:

```text
PROJECT
```

is currently a shell variable, but it has not been exported into the environment inherited by child processes.

---

# 20. Parent and Child Processes

Remember Part 8.

Processes can start other processes.

Your shell is a process.

When Bash starts a command, that command normally runs as another process.

Conceptually:

```text
Bash
PID 2000
|
+-- grep
|   PID 2010
|
+-- python3
|   PID 2020
|
+-- nginx command
    PID 2030
```

Environment inheritance follows this parent-child relationship.

---

# 21. Environment Inheritance

When a process starts a child process, the child receives a copy of the parent's exported environment at that moment.

Conceptually:

```text
Bash
APP_ENV=production
       |
       | starts child
       v
Python
APP_ENV=production
```

But changes made by the child do not normally modify the parent shell's environment.

Conceptually:

```text
Parent Bash
APP_ENV=production
       |
       v
Child process changes its own APP_ENV
       |
       X
       |
Parent Bash remains unchanged
```

This direction of inheritance is very important.

---

# 22. `export`

The `export` shell builtin marks a shell variable for inclusion in the environment passed to subsequently started child processes.

Example:

```bash
PROJECT="veriqta"
```

Now:

```bash
export PROJECT
```

Check:

```bash
printenv PROJECT
```

You should see:

```text
veriqta
```

You can also create and export in one command:

```bash
export PROJECT="veriqta"
```

---

# 23. Demonstrating Shell Variable vs Exported Variable

Try this lab.

Create:

```bash
LAB_NAME="linux-environment"
```

Check:

```bash
echo "$LAB_NAME"
```

Now start a child Bash that tries to display it:

```bash
bash -c 'echo "$LAB_NAME"'
```

You may get a blank line.

Why?

`LAB_NAME` was not exported.

Now:

```bash
export LAB_NAME
```

Run:

```bash
bash -c 'echo "$LAB_NAME"'
```

Now you should see:

```text
linux-environment
```

You have demonstrated environment inheritance.

---

# 24. Why the Single Quotes Matter in That Test

We used:

```bash
bash -c 'echo "$LAB_NAME"'
```

rather than:

```bash
bash -c "echo \"$LAB_NAME\""
```

The single quotes prevent the current shell from expanding `$LAB_NAME` before the child Bash starts.

That allows the child Bash to perform the expansion itself.

This connects directly with quoting concepts from Part 7.

---

# 25. Temporary Variables

Suppose you run:

```bash
export APP_ENV="development"
```

The variable exists in the current shell environment.

Check:

```bash
echo "$APP_ENV"
```

Now exit the shell:

```bash
exit
```

Open a new shell session.

Try:

```bash
echo "$APP_ENV"
```

Unless something else defines it, the variable is gone.

Why?

Because you changed the environment of a running shell process.

You did not permanently configure future shells.

---

# 26. Temporary Does Not Mean "For a Few Minutes"

A temporary variable does not expire after a specific number of minutes.

It generally remains available for the lifetime and relevant scope of the shell/process environment in which it exists.

For example:

```bash
export APP_ENV="development"
```

could remain available for hours if that shell remains open.

"Temporary" means it has not been configured so future shell sessions recreate it automatically.

---

# 27. Setting a Variable for One Command

You can provide an environment variable to a single command:

```bash
APP_ENV=testing ./myapp
```

Conceptually:

```text
Current shell
APP_ENV may be unset
       |
       v
Start ./myapp with:
APP_ENV=testing
       |
       v
myapp receives the value
       |
       v
Command ends
       |
       v
Current shell is not permanently changed
```

This pattern is common in Linux development and automation.

---

# 28. Test a One-Command Environment Variable

Run:

```bash
APP_ENV=testing printenv APP_ENV
```

Output:

```text
testing
```

Then:

```bash
printenv APP_ENV
```

If `APP_ENV` was not already exported in your shell, you may receive no output.

This demonstrates command-scoped environment assignment.

---

# 29. Removing Variables With `unset`

Create:

```bash
export TEST_VARIABLE="hello"
```

Check:

```bash
echo "$TEST_VARIABLE"
```

Then:

```bash
unset TEST_VARIABLE
```

Check:

```bash
echo "$TEST_VARIABLE"
```

You should receive a blank line.

`unset` removes the variable from the current shell.

It does not automatically remove a line that may define the variable inside `.bashrc`, `.profile`, or another configuration file.

If a startup file recreates it, it can return in a new shell.

---

# 30. Quoting Variables

Variable quoting is one of the most important shell habits you can learn.

Suppose:

```bash
PROJECT_DIR="/home/ann/Linux Projects"
```

Then:

```bash
echo "$PROJECT_DIR"
```

correctly treats the expanded value as one shell word for commands where that matters.

Use double quotes around variable expansions unless you specifically need shell word splitting or glob expansion.

A good beginner habit is:

```bash
"$VARIABLE"
```

rather than:

```bash
$VARIABLE
```

---

# 31. Double Quotes and Variable Expansion

Double quotes allow variable expansion.

Example:

```bash
NAME="Ann"
```

Then:

```bash
echo "Hello $NAME"
```

Output:

```text
Hello Ann
```

The shell expands `$NAME` inside double quotes.

---

# 32. Single Quotes and Variable Expansion

Single quotes prevent normal variable expansion.

Run:

```bash
echo 'Hello $NAME'
```

Output:

```text
Hello $NAME
```

Compare:

```bash
echo "Hello $NAME"
```

Output:

```text
Hello Ann
```

Remember:

```text
Single quotes
Treat contents literally.

Double quotes
Allow variable and certain other expansions.
```

---

# 33. Why Braces Are Useful

Suppose:

```bash
NAME="linux"
```

You want:

```text
linuxlab
```

This can be ambiguous:

```bash
echo "$NAMElab"
```

The shell looks for a variable named:

```text
NAMElab
```

Instead use:

```bash
echo "${NAME}lab"
```

Output:

```text
linuxlab
```

Braces clearly identify where the variable name ends.

---

# 34. Shell Expansion

Shell expansion means the shell transforms parts of your command before executing it.

You have already encountered several forms.

Variable expansion:

```bash
echo "$HOME"
```

Tilde expansion:

```bash
cd ~
```

Command substitution:

```bash
echo "$(hostname)"
```

Wildcard or pathname expansion:

```bash
ls *.log
```

Conceptually:

```text
You type command
      |
      v
Shell parses command
      |
      v
Shell performs applicable expansions
      |
      v
Shell determines command and arguments
      |
      v
Command executes
```

This is why understanding the shell matters.

The command you type is not always exactly what the external program receives.

---

# 35. Expansion Example

Suppose:

```bash
LOG_DIR="/var/log"
```

You type:

```bash
ls "$LOG_DIR"
```

Before `ls` executes, Bash expands:

```text
$LOG_DIR
```

to:

```text
/var/log
```

The effective command arguments are equivalent to:

```bash
ls /var/log
```

The `ls` program itself does not need to understand `$LOG_DIR`.

The shell performed the expansion first.

---

# 36. Environment Variables Are Case-Sensitive

Create:

```bash
PROJECT="Linux"
```

Then:

```bash
echo "$PROJECT"
```

Output:

```text
Linux
```

But:

```bash
echo "$project"
```

may output nothing.

Linux and Bash are case-sensitive.

This matters especially with:

```text
PATH
HOME
USER
SHELL
```

Do not write:

```text
Path
Home
User
Shell
```

and expect them to refer to the same variables.

---

# 37. Understanding `$PATH`

`PATH` is one of the most important environment variables in Linux.

Run:

```bash
echo "$PATH"
```

You may see:

```text
/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
```

`PATH` contains a list of directories.

Directories are normally separated by colons:

```text
:
```

Formatted vertically:

```text
/usr/local/sbin
/usr/local/bin
/usr/sbin
/usr/bin
/sbin
/bin
```

The shell uses these directories when searching for executable commands.

---

# 38. Why `$PATH` Exists

Suppose the `ls` executable is located at:

```text
/usr/bin/ls
```

You could type:

```bash
/usr/bin/ls
```

every time.

But normally you type:

```bash
ls
```

How does the shell know where to find it?

It performs command lookup.

`PATH` is a major part of that process for external commands.

---

# 39. Executable Lookup

When you type:

```bash
nginx
```

the shell must determine what `nginx` means.

For an external executable, it may search directories listed in `$PATH`.

Conceptually:

```text
PATH=
/usr/local/bin
/usr/bin
/bin
```

Search:

```text
/usr/local/bin/nginx
        |
        X not found

/usr/bin/nginx
        |
        v found
```

Then that executable can be run.

The actual shell command-resolution process also considers things such as aliases, functions, builtins, and cached command locations.

We will keep the model beginner-friendly while learning how to inspect those distinctions.

---

# 40. View `$PATH` One Directory Per Line

Run:

```bash
echo "$PATH" | tr ':' '\n'
```

You learned `tr` in Part 6.

Here:

```text
:
```

is being translated into newline characters.

This makes a long `PATH` easier to inspect.

Example:

```text
/usr/local/sbin
/usr/local/bin
/usr/sbin
/usr/bin
/sbin
/bin
```

---

# 41. PATH Order Matters

Suppose:

```text
PATH=/home/ann/bin:/usr/local/bin:/usr/bin
```

and two different executables exist:

```text
/home/ann/bin/tool
/usr/bin/tool
```

When command lookup searches `PATH`, the earlier matching executable normally wins.

Conceptually:

```text
Search /home/ann/bin
        |
        v
tool found
        |
        v
Stop PATH search
```

This means `PATH` order can affect which software version executes.

---

# 42. Why PATH Order Can Become a Security Problem

Imagine a malicious executable called:

```text
ls
```

is placed in an untrusted directory.

If that directory appears before trusted system directories in `PATH`, typing:

```bash
ls
```

could execute the malicious program.

This is one reason you should not carelessly add:

```text
.
```

the current directory, especially near the beginning of `PATH`.

A dangerous pattern can be:

```bash
export PATH=".:$PATH"
```

because command behavior can then depend on which directory you happen to be standing in.

For beginners, avoid adding the current directory to `PATH`.

---

# 43. Absolute Path vs PATH Lookup

Compare:

```bash
/usr/bin/ls
```

with:

```bash
ls
```

The first explicitly specifies the executable path.

The second asks the shell to resolve the command name.

If you suspect a `PATH` problem, trying the full executable path can help isolate the issue.

Example:

```bash
/usr/bin/python3 --version
```

If that works while:

```bash
python3 --version
```

does not, investigate command resolution and `PATH`.

---

# 44. Why `./script.sh` Is Often Required

Suppose your current directory contains:

```text
deploy.sh
```

You type:

```bash
deploy.sh
```

and receive:

```text
command not found
```

But:

```bash
./deploy.sh
```

works.

Why?

Because the current directory is normally not searched automatically unless it appears in `PATH`.

`./deploy.sh` explicitly says:

```text
Run deploy.sh from the current directory.
```

This is intentional and has security benefits.

---

# 45. `which`

You have already used:

```bash
which nginx
```

Possible output:

```text
/usr/sbin/nginx
```

`which` is commonly used to locate a command according to `PATH`.

Try:

```bash
which ls
```

and:

```bash
which python3
```

However, `which` is not always the best tool for understanding everything the shell might execute.

Shell commands can also be:

* Aliases.
* Functions.
* Builtins.

That is why `command -v` is particularly useful.

---

# 46. `command -v`

Run:

```bash
command -v ls
```

You may see:

```text
/usr/bin/ls
```

Try:

```bash
command -v cd
```

You may see:

```text
cd
```

Why not a filesystem path?

Because `cd` is normally a shell builtin.

Try:

```bash
command -v echo
```

The result depends on your shell environment.

`command -v` helps determine how the shell resolves a command.

---

# 47. External Commands vs Shell Builtins

Some commands are executable files.

Example:

```bash
command -v nginx
```

might return:

```text
/usr/sbin/nginx
```

Others are implemented inside the shell itself.

For example:

```bash
command -v cd
```

may return:

```text
cd
```

`cd` needs to affect the current shell's working directory, which is one reason it is implemented as a shell builtin.

An external child process could change its own directory, but that would not change the parent shell's working directory.

---

# 48. Use `type` to Understand Commands

Although not in our original command list, `type` is extremely useful here.

Run:

```bash
type cd
```

You may see:

```text
cd is a shell builtin
```

Run:

```bash
type ls
```

You may see a path, or perhaps an alias depending on your configuration.

Run:

```bash
type -a ls
```

This can show multiple possible definitions or locations.

This becomes very useful when troubleshooting command resolution.

---

# 49. What Is an Alias?

An alias gives a shell command or command sequence another name.

Example:

```bash
alias ll='ls -lh'
```

Now:

```bash
ll
```

causes the shell to expand the alias to:

```bash
ls -lh
```

Aliases are convenient for frequently used interactive commands.

---

# 50. Viewing Aliases

Run:

```bash
alias
```

You may see existing aliases.

For a specific alias:

```bash
alias ll
```

if it exists.

Common systems may define aliases such as:

```bash
alias ll='ls -alF'
```

Exact defaults vary by distribution and shell configuration.

---

# 51. Creating an Alias

Create:

```bash
alias cproj='cd ~/projects'
```

Now:

```bash
cproj
```

attempts to change to:

```text
~/projects
```

If the directory does not exist, you receive an error.

Create it first if needed:

```bash
mkdir -p ~/projects
```

Then:

```bash
cproj
pwd
```

---

# 52. Removing an Alias

Use:

```bash
unalias cproj
```

Then:

```bash
command -v cproj
```

should no longer resolve that alias unless something else defines the same command name.

Like a temporary variable, an alias created interactively normally disappears when that shell session ends.

---

# 53. Aliases Can Hide Real Commands

Suppose:

```bash
alias ls='ls --color=auto'
```

When you type:

```bash
ls
```

you are not necessarily invoking the external executable with no arguments.

The shell expands the alias first.

Use:

```bash
type ls
```

to investigate.

You can also bypass an alias in Bash with:

```bash
command ls
```

or, commonly:

```bash
\ls
```

depending on what you are testing.

This is useful when an alias changes command behavior unexpectedly.

---

# 54. Temporary Configuration vs Persistent Configuration

So far, we have created things like:

```bash
export PROJECT="veriqta"
```

and:

```bash
alias ll='ls -lh'
```

These affect the current shell.

When that shell ends, the configuration may disappear.

To recreate settings automatically in future shell sessions, shell startup files are used.

For Bash, important files include:

```text
~/.bashrc
~/.profile
```

You may also encounter:

```text
~/.bash_profile
~/.bash_login
```

and system-wide configuration files.

---

# 55. Hidden Shell Configuration Files

Remember hidden files from Part 2 and Part 3.

Files beginning with:

```text
.
```

are hidden from normal `ls` output.

So:

```text
.bashrc
.profile
```

are hidden files.

View them:

```bash
ls -la ~
```

You may see:

```text
.bashrc
.profile
```

depending on your environment.

---

# 56. What Is `.bashrc`?

`.bashrc` is a Bash configuration file commonly read for interactive non-login Bash shells.

It is typically located at:

```text
~/.bashrc
```

It often contains:

* Aliases.
* Interactive shell settings.
* Prompt configuration.
* Shell functions.
* Environment-related configuration.
* Completion setup.

The exact contents vary by distribution.

---

# 57. Inspect `.bashrc`

Run:

```bash
less ~/.bashrc
```

Do not immediately edit it.

Read it first.

Look for:

```text
alias
PATH
PS1
```

You may see comments beginning with:

```text
#
```

A shell configuration file is executable shell syntax.

A syntax mistake can affect future shell sessions.

---

# 58. Back Up `.bashrc` Before Editing

Before making significant changes:

```bash
cp ~/.bashrc ~/.bashrc.backup
```

Verify:

```bash
ls -l ~/.bashrc ~/.bashrc.backup
```

This gives you a simple recovery copy.

For important configuration files, backups are a good habit.

---

# 59. Adding an Alias to `.bashrc`

Open:

```bash
nano ~/.bashrc
```

Add:

```bash
alias llab='ls -lah'
```

Save the file.

The current shell may not automatically reread the file just because it changed.

You can start a new appropriate Bash shell, or reload it in the current shell.

---

# 60. `source`

Run:

```bash
source ~/.bashrc
```

This tells the current shell to read and execute commands from that file.

Bash also supports:

```bash
. ~/.bashrc
```

The dot command is another form of sourcing.

Now:

```bash
llab
```

should work if the alias was defined correctly.

---

# 61. Why `source` Is Different From Running a Script

This distinction matters.

Suppose you execute a script as a child process.

Changes to that child process's environment normally do not modify the parent shell.

But:

```bash
source file
```

executes the file in the current shell context.

Therefore commands such as:

```bash
export PROJECT="veriqta"
```

inside the sourced file can affect your current shell.

Conceptually:

```text
Normal script execution

Current Bash
    |
    v
Child script process
    |
changes child environment
    |
    X
Current Bash unchanged
```

Compared with:

```text
source file

Current Bash
    |
    v
Read commands into same shell
    |
    v
Current shell changes
```

---

# 62. What Is `.profile`?

`.profile` is a user startup file commonly used for login-session environment configuration.

It may contain:

* Environment variables.
* `PATH` modifications.
* Commands that should run when a login environment is established.

On some systems, `.profile` may also arrange for `.bashrc` to be loaded for Bash under appropriate conditions.

The exact startup behavior depends on:

* Shell.
* Distribution.
* Whether the shell is login or non-login.
* Whether it is interactive.
* How the session was started.

Do not assume every Linux shell reads every startup file.

---

# 63. Login Shell vs Interactive Shell

These terms describe different properties.

They are not opposites.

A shell can be:

```text
Interactive + login

Interactive + non-login

Non-interactive + login

Non-interactive + non-login
```

For beginner purposes, focus on two questions:

```text
Is the shell a login shell?

Is the shell interactive?
```

---

# 64. What Is an Interactive Shell?

An interactive shell is a shell intended to interact directly with a user.

For example, you open a terminal and see:

```text
ann@server:~$
```

You type:

```bash
pwd
```

The shell responds.

Then you type another command.

That is an interactive experience.

Interactive configuration often includes things such as:

* Prompt appearance.
* Aliases.
* Command completion.
* Interactive convenience settings.

---

# 65. What Is a Non-Interactive Shell?

A non-interactive shell runs commands without a user continuously typing commands at a prompt.

For example:

```bash
bash script.sh
```

starts Bash to execute the script.

Automation systems also frequently execute non-interactive shells.

This matters because configuration that works in your interactive terminal may not automatically exist in:

* Cron.
* CI/CD jobs.
* systemd services.
* Remote automation.
* Scripts.

This is a major source of real-world environment problems.

---

# 66. What Is a Login Shell?

A login shell represents a shell started as part of a login-style session.

Historically, this literally corresponded closely to logging into a system.

Modern terminals, SSH sessions, desktop sessions, containers, and terminal applications can make the details less obvious.

For beginner purposes, understand that Bash uses different startup rules for login and non-login shells.

---

# 67. Bash Startup Files at Beginner Level

A simplified model is:

```text
Login Bash
|
+-- system-wide login configuration
|
+-- user login files
    |
    +-- ~/.bash_profile
    +-- ~/.bash_login
    +-- ~/.profile
```

Bash checks these user login files according to its startup rules, not simply all of them one after another.

Interactive non-login Bash commonly reads:

```text
~/.bashrc
```

Many Linux configurations connect these files so login sessions also receive useful `.bashrc` settings.

This is why startup behavior can vary across machines.

---

# 68. Do Not Memorize an Oversimplified Rule

You may hear:

```text
Put environment variables in .profile.
Put aliases in .bashrc.
```

That is a useful starting guideline, but real behavior depends on how sessions are started and how startup files source one another.

Instead of blindly memorizing:

```text
Which file should I use?
```

learn to ask:

```text
Which shell is running?

Is it interactive?

Is it a login shell?

Which startup files does this shell actually read?
```

That reasoning is more useful.

---

# 69. Check Whether Bash Is Interactive

Bash exposes shell option flags in:

```bash
echo "$-"
```

If the output contains:

```text
i
```

the Bash shell is interactive.

For example:

```text
himBHs
```

contains `i`.

Therefore it is interactive.

---

# 70. Check Whether Bash Is a Login Shell

In Bash, run:

```bash
shopt -q login_shell
```

Then:

```bash
echo $?
```

Remember exit statuses.

A result of:

```text
0
```

means the `login_shell` option test succeeded.

A result of:

```text
1
```

means it is not a login shell.

You can also use:

```bash
shopt login_shell
```

which displays the state more directly.

---

# 71. What Shell Am I Actually Running?

As discussed earlier:

```bash
echo "$SHELL"
```

does not necessarily prove the currently executing shell.

In Bash, you can inspect:

```bash
echo "$BASH_VERSION"
```

If you are running Bash, this normally contains the Bash version.

You can also inspect the current process:

```bash
ps -p $$ -o pid,ppid,comm,args
```

Here:

```text
$$
```

expands to the current shell's process ID in Bash.

This combines shell expansion with process management from Part 8.

---

# 72. Persistent Environment Variables

Suppose you want:

```text
PROJECT_HOME=/home/ann/projects
```

to be recreated in future shell sessions.

You might place an appropriate export command in a startup file used by those sessions:

```bash
export PROJECT_HOME="$HOME/projects"
```

The correct file depends on when and where you need the variable.

After modifying the relevant file, either start a new session or source it when sourcing is appropriate.

---

# 73. Persistence Is Really Re-Creation

This is an important conceptual point.

When you add:

```bash
export PROJECT_HOME="$HOME/projects"
```

to `.bashrc`, Linux is not keeping the same variable alive forever.

Instead:

```text
Old shell ends
     |
     v
Variable disappears with process
     |
     v
New shell starts
     |
     v
Startup file executes
     |
     v
Variable is created again
```

Persistent shell configuration means future sessions recreate the desired state.

---

# 74. Adding a Directory to `$PATH`

Suppose you create personal scripts under:

```text
/home/ann/bin
```

or more generally:

```text
$HOME/bin
```

Create the directory:

```bash
mkdir -p "$HOME/bin"
```

Suppose you want commands inside it to be executable without typing the full path.

You can add it to `PATH`.

A common pattern is:

```bash
export PATH="$HOME/bin:$PATH"
```

---

# 75. Understanding `export PATH="$HOME/bin:$PATH"`

Break it down.

Current `PATH`:

```text
/usr/local/bin:/usr/bin:/bin
```

`$HOME/bin`:

```text
/home/ann/bin
```

This expression:

```bash
"$HOME/bin:$PATH"
```

expands approximately to:

```text
/home/ann/bin:/usr/local/bin:/usr/bin:/bin
```

Then:

```bash
export PATH=...
```

sets and exports the new value.

You have added your personal directory without deleting the existing `PATH`.

---

# 76. Prepending vs Appending to `$PATH`

Prepending:

```bash
export PATH="$HOME/bin:$PATH"
```

produces:

```text
$HOME/bin
before
existing PATH directories
```

Appending:

```bash
export PATH="$PATH:$HOME/bin"
```

puts your directory at the end.

This matters because search order matters.

If two directories contain a command with the same name, the earlier relevant `PATH` match normally wins.

---

# 77. A Dangerous PATH Mistake

Suppose you run:

```bash
export PATH="$HOME/bin"
```

You have replaced the existing `PATH`.

Now commands in:

```text
/usr/bin
/bin
/usr/sbin
```

may no longer be found through normal `PATH` lookup.

You may type:

```bash
ls
```

and receive:

```text
command not found
```

even though `ls` still exists.

You did not necessarily delete `ls`.

You broke command lookup.

---

# 78. Recovering From a Broken PATH

Suppose you accidentally run:

```bash
export PATH="$HOME/bin"
```

Try the absolute path:

```bash
/usr/bin/ls
```

If that works, the executable exists.

Your `PATH` is the likely problem.

For a typical system, you might temporarily restore known system directories, but do not blindly copy a `PATH` from another distribution.

For example, on many systems a temporary repair might resemble:

```bash
export PATH="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
```

Then:

```bash
command -v ls
```

But the correct `PATH` depends on the system.

If the damage came from a startup file, fix that file as well.

Otherwise the problem will return when a new shell starts.

---

# 79. Build PATH Safely

Instead of replacing:

```bash
export PATH="$HOME/bin"
```

preserve the old value:

```bash
export PATH="$HOME/bin:$PATH"
```

Then verify:

```bash
echo "$PATH"
```

and:

```bash
command -v ls
```

and:

```bash
command -v your-command
```

Never modify `PATH` without checking the result.

---

# 80. Creating Your Own Executable

Create:

```bash
mkdir -p "$HOME/bin"
```

Create a script:

```bash
nano "$HOME/bin/server-info"
```

Add:

```bash
#!/usr/bin/env bash

echo "User: $(whoami)"
echo "Host: $(hostname)"
echo "Directory: $(pwd)"
echo "Date: $(date)"
```

Save and exit.

Make it executable:

```bash
chmod 755 "$HOME/bin/server-info"
```

Try:

```bash
"$HOME/bin/server-info"
```

It should run.

---

# 81. Why the Script Does Not Yet Run by Name

Try:

```bash
server-info
```

If `$HOME/bin` is not in `PATH`, you may receive:

```text
server-info: command not found
```

The file exists.

The executable permission exists.

But the shell does not know to search that directory.

This distinction is essential:

```text
File exists
+
Executable
+
Directory searchable through PATH
=
Command can normally be found by name
```

---

# 82. Add the Directory Temporarily

Run:

```bash
export PATH="$HOME/bin:$PATH"
```

Then:

```bash
command -v server-info
```

You should see something similar to:

```text
/home/ann/bin/server-info
```

Now:

```bash
server-info
```

should work.

You have just seen `PATH` command lookup directly.

---

# 83. Make the PATH Change Persistent

For an interactive Bash lab, add the line to an appropriate startup file such as `.bashrc`:

```bash
export PATH="$HOME/bin:$PATH"
```

Then:

```bash
source ~/.bashrc
```

Verify:

```bash
echo "$PATH"
```

and:

```bash
command -v server-info
```

Open a new terminal or Bash session and verify again.

---

# 84. Avoid Duplicating PATH Entries

If you repeatedly add:

```bash
export PATH="$HOME/bin:$PATH"
```

from configuration that gets sourced multiple times, you may end up with:

```text
/home/ann/bin:/home/ann/bin:/home/ann/bin:/usr/bin:/bin
```

Usually this is not catastrophic, but it creates messy configuration.

Some startup files conditionally add directories only when appropriate.

For beginners, inspect your `PATH` after configuration changes:

```bash
echo "$PATH" | tr ':' '\n'
```

---

# 85. Empty PATH Components

A subtle security issue can occur if `PATH` contains an empty component, for example:

```text
/usr/bin::/bin
```

or a leading/trailing colon depending on shell behavior.

An empty component can be interpreted as the current directory during command lookup.

For beginners, avoid constructing `PATH` values with accidental empty entries.

Be careful with commands such as:

```bash
export PATH="$NEW_DIR:$PATH"
```

if `NEW_DIR` might be empty and you have not considered the result.

---

# 86. `command not found`

You will see this error frequently:

```text
command not found
```

It does not have one cause.

Possible causes include:

* Command is misspelled.
* Software is not installed.
* Executable is not in `PATH`.
* `PATH` is incorrect.
* Script is in the current directory but `.` is not in `PATH`.
* Executable has another name.
* Alias or function behavior is confusing you.
* Shell configuration has not been reloaded.
* You are using a different shell or environment.
* Permissions prevent normal execution in a way that may produce a different error.
* The command exists only in another user's environment.

Do not immediately reinstall software.

---

# 87. `command not found` Troubleshooting Workflow

Use:

```text
command not found
      |
      v
Check spelling
      |
      v
command -v COMMAND
      |
      v
Is software installed?
      |
      v
Do I know the executable path?
      |
      v
Does absolute path work?
      |
      v
Inspect PATH
      |
      v
Is required directory present?
      |
      v
Check executable permission
      |
      v
Check shell configuration
      |
      v
Check aliases/functions if relevant
```

---

# 88. Scenario: Typing Mistake

You type:

```bash
ngnix
```

instead of:

```bash
nginx
```

You receive:

```text
command not found
```

Before editing `PATH`, check spelling.

Troubleshooting should begin with simple causes.

---

# 89. Scenario: Package Not Installed

Run:

```bash
command -v nginx
```

No result.

Check the package database.

Debian:

```bash
dpkg -s nginx
```

RPM:

```bash
rpm -q nginx
```

If the package is not installed, `PATH` may be perfectly fine.

The executable simply may not exist.

---

# 90. Scenario: Executable Exists Outside PATH

Suppose:

```text
/opt/veriqta/bin/veriqta
```

exists.

Run:

```bash
ls -l /opt/veriqta/bin/veriqta
```

Then:

```bash
/opt/veriqta/bin/veriqta
```

works.

But:

```bash
veriqta
```

returns:

```text
command not found
```

Inspect:

```bash
echo "$PATH" | tr ':' '\n'
```

If:

```text
/opt/veriqta/bin
```

is missing, you have found the likely cause.

---

# 91. Scenario: Current Directory Script

You have:

```text
./deploy.sh
```

Check:

```bash
ls -l deploy.sh
```

If executable:

```bash
./deploy.sh
```

may work.

But:

```bash
deploy.sh
```

may not.

Reason:

```text
Current directory is not normally searched unless included in PATH.
```

Do not "fix" this by casually adding `.` to the beginning of `PATH`.

Use:

```bash
./deploy.sh
```

or install the script into an appropriate command directory.

---

# 92. Scenario: File Exists but Is Not Executable

Check:

```bash
ls -l "$HOME/bin/server-info"
```

Suppose:

```text
-rw-r--r--
```

There is no execute bit.

Fix:

```bash
chmod 755 "$HOME/bin/server-info"
```

Verify:

```bash
ls -l "$HOME/bin/server-info"
```

Then:

```bash
server-info
```

if its directory is in `PATH`.

This combines Part 5 with Part 11.

---

# 93. Scenario: PATH Works in Terminal but Not in Script

Suppose:

```bash
mytool
```

works in your terminal.

But a script says:

```text
mytool: command not found
```

Possible cause:

Your interactive shell loads a startup file that adds the tool's directory to `PATH`.

The script's environment may not load the same file.

Conceptually:

```text
Interactive terminal
      |
      +--> reads interactive config
      |
      +--> PATH contains /opt/mytool/bin
      |
      v
mytool works


Automation/script environment
      |
      +--> different startup behavior
      |
      +--> PATH lacks /opt/mytool/bin
      |
      v
mytool not found
```

This is a very common production problem.

---

# 94. Do Not Assume `.bashrc` Fixes Every Environment

A beginner might respond:

```text
I'll put everything in .bashrc.
```

But:

* systemd services do not simply behave like your interactive Bash session.
* Cron may have a different environment.
* CI/CD runners may construct their own environment.
* Containers may define variables differently.
* Non-interactive shells may not read `.bashrc` in the way you expect.
* Another user's shell has different startup files.

Always understand where the process gets its environment.

---

# 95. Environment Variables and `sudo`

Suppose:

```bash
export APP_ENV="development"
```

Then:

```bash
printenv APP_ENV
```

works.

You may expect:

```bash
sudo printenv APP_ENV
```

to show the same thing.

It may not.

`sudo` applies security policies to the environment and may preserve, remove, or replace variables depending on configuration.

Do not assume:

```text
My shell environment
=
sudo command environment
```

This is intentional for security.

---

# 96. `$HOME` and `sudo`

Another common beginner mistake is assuming every command involving `sudo` automatically changes `$HOME` in exactly the way they expect.

Shell expansion can occur before `sudo` executes.

For example:

```bash
sudo echo "$HOME"
```

Your current shell expands:

```text
$HOME
```

before `sudo` starts `echo`.

So the output may still reflect your current shell's expansion.

Always think:

```text
Which shell performs the expansion?

Which process receives the result?
```

This reasoning becomes very important in administration.

---

# 97. Environment Variables and Shell Scripts

Create:

```bash
nano ~/environment-demo.sh
```

Add:

```bash
#!/usr/bin/env bash

echo "User: $USER"
echo "Home: $HOME"
echo "Shell: $SHELL"
echo "Path: $PATH"
echo "Project: $PROJECT"
```

Save.

Make executable:

```bash
chmod 755 ~/environment-demo.sh
```

Set:

```bash
PROJECT="linux-lab"
```

Run:

```bash
~/environment-demo.sh
```

Observe whether `PROJECT` appears.

Then:

```bash
export PROJECT
```

Run the script again.

You should understand why the result changed.

---

# 98. Environment Variables and Child Processes

This deserves repetition because it is foundational.

Suppose:

```bash
export COLOR="blue"
```

Then:

```bash
bash
```

starts a child Bash.

Inside it:

```bash
echo "$COLOR"
```

shows:

```text
blue
```

Now change:

```bash
COLOR="red"
```

Exit:

```bash
exit
```

Back in the parent shell:

```bash
echo "$COLOR"
```

should still show:

```text
blue
```

The child's change did not travel backward into the parent.

---

# 99. Environment Variables Are Copied, Not Shared Live

Think:

```text
Parent
COLOR=blue
    |
    | creates child
    v
Child receives copy
COLOR=blue
    |
    v
Child changes:
COLOR=red
```

You do not now have one shared variable.

You have separate process environments.

Therefore:

```text
Parent remains blue.
Child becomes red.
```

This process model will help later with containers and application processes.

---

# 100. System-Wide Environment Configuration

So far we have focused mostly on one user's shell.

Linux also has system-wide shell and environment configuration mechanisms.

Depending on distribution and context, you may encounter:

```text
/etc/environment
/etc/profile
/etc/profile.d/
/etc/bash.bashrc
/etc/bashrc
```

Not every distribution uses these files identically.

Do not edit all of them just because they exist.

Understand their purpose first.

---

# 101. `/etc/profile`

`/etc/profile` is commonly associated with system-wide login shell configuration for Bourne-style shells.

It may:

* Establish system-wide environment settings.
* Set defaults.
* Run scripts from other configuration locations.

Inspect:

```bash
less /etc/profile
```

Do not edit it during your first exploration.

---

# 102. `/etc/profile.d/`

Many distributions use:

```text
/etc/profile.d/
```

for modular shell configuration scripts.

Inspect:

```bash
ls -l /etc/profile.d/
```

You may see multiple `.sh` files.

This allows packages and administrators to manage separate pieces of shell configuration instead of putting everything into one huge `/etc/profile`.

---

# 103. `/etc/environment`

You may encounter:

```text
/etc/environment
```

for system-wide environment configuration.

It is not simply a normal Bash script in every context.

Do not assume that arbitrary Bash syntax, command substitution, aliases, or shell logic placed there will work like `.bashrc`.

Its interpretation depends on the components reading it.

This is a good general rule:

```text
A configuration file containing NAME=value
does not automatically mean
it behaves like a Bash script.
```

---

# 104. System-Wide vs User-Specific Configuration

User-specific configuration affects one user's environment.

Examples:

```text
~/.bashrc
~/.profile
```

System-wide configuration can affect many users.

Examples:

```text
/etc/profile
/etc/profile.d/
```

Conceptually:

```text
System-wide configuration
       |
       +--> Ann
       +--> David
       +--> Application user
       +--> Administrator
```

Because the impact is broader, system-wide changes require more caution.

---

# 105. Do Not Use System-Wide Configuration Without a Reason

If only your user needs:

```text
$HOME/bin
```

in `PATH`, changing global configuration may be unnecessary.

Prefer the smallest appropriate scope.

This follows the same principle you learned with permissions:

```text
Only change what needs to change.
```

Broad configuration changes create broader failure possibilities.

---

# 106. Shell Configuration Syntax Errors

Suppose you add this to `.bashrc`:

```bash
export PROJECT_HOME="$HOME/projects
```

Notice the missing closing quote.

The next shell may report syntax errors or fail to process later configuration correctly.

Before closing your current working shell, you can check Bash syntax:

```bash
bash -n ~/.bashrc
```

If the command produces no output and returns success, Bash did not detect a syntax error.

Check:

```bash
echo $?
```

A zero status indicates successful syntax checking.

---

# 107. Why Keep the Current Terminal Open?

Suppose you modify `.bashrc` incorrectly.

If you immediately close your only working terminal and then discover new shells behave badly, recovery can become more inconvenient.

A good habit:

1. Back up the file.
2. Make a small change.
3. Run a syntax check.
4. Open another terminal.
5. Test the new configuration.
6. Keep the original working session open until verification succeeds.

This is a simple production-minded habit.

---

# 108. PATH Troubleshooting

When `PATH` behaves unexpectedly, inspect it directly:

```bash
echo "$PATH"
```

Then:

```bash
echo "$PATH" | tr ':' '\n'
```

Ask:

* Is the required directory present?
* Is the path spelled correctly?
* Does the directory exist?
* Does the executable exist there?
* Is the executable actually executable?
* Is another executable with the same name found earlier?
* Did a startup file overwrite `PATH`?
* Did I open a new shell?
* Did I source the correct file?
* Am I using the shell I think I am?

---

# 109. Check Whether a PATH Directory Exists

Suppose you expect:

```text
/opt/veriqta/bin
```

Check:

```bash
ls -ld /opt/veriqta/bin
```

Then:

```bash
ls -l /opt/veriqta/bin
```

If the directory does not exist, adding it to `PATH` will not magically create it.

`PATH` only tells the shell where to search.

It does not install software or create directories.

---

# 110. Check the Executable

Suppose:

```text
/opt/veriqta/bin/veriqta
```

should exist.

Run:

```bash
ls -l /opt/veriqta/bin/veriqta
```

Then:

```bash
file /opt/veriqta/bin/veriqta
```

Then:

```bash
/opt/veriqta/bin/veriqta
```

Ask:

```text
Does it exist?

Is it executable?

Is it a script or binary?

Does absolute-path execution work?
```

Only then move deeper into `PATH` troubleshooting.

---

# 111. Multiple Versions of the Same Command

Suppose you have:

```text
/usr/bin/python3
/usr/local/bin/python3
```

Which runs when you type:

```bash
python3
```

Use:

```bash
command -v python3
```

Then:

```bash
type -a python3
```

The `PATH` order helps determine which external executable is selected.

This is common when software is:

* Installed by the distribution.
* Installed manually.
* Installed by a version manager.
* Installed into a user's directory.

---

# 112. Command Hashing

Bash may remember command locations to speed up repeated command execution.

You can inspect remembered locations with:

```bash
hash
```

If you move or replace executables and command resolution behaves unexpectedly, you may encounter stale command-location information.

Bash can clear its remembered command locations with:

```bash
hash -r
```

Do not use this as the first solution to every `PATH` problem.

First inspect:

```bash
command -v COMMAND
type -a COMMAND
echo "$PATH"
```

---

# 113. Environment Variable Troubleshooting

Suppose an application says:

```text
DATABASE_HOST is not set
```

Do not immediately modify `.bashrc`.

Check:

```bash
printenv DATABASE_HOST
```

Then:

```bash
echo "$DATABASE_HOST"
```

If it exists as a shell variable but not in `printenv`, perhaps it was never exported.

Check:

```bash
export -p | grep DATABASE_HOST
```

This helps distinguish:

```text
Shell variable
```

from:

```text
Exported variable.
```

---

# 114. Problem: Variable Exists in Shell but Application Cannot See It

You run:

```bash
DATABASE_HOST="db.internal"
```

Then:

```bash
echo "$DATABASE_HOST"
```

works.

But your application cannot see it.

Possible cause:

You created a shell variable but did not export it.

Fix:

```bash
export DATABASE_HOST
```

or:

```bash
export DATABASE_HOST="db.internal"
```

Then restart the application process so it inherits the new environment.

Existing child processes do not automatically receive later changes made to the parent shell.

---

# 115. Problem: Exported Variable Still Does Not Reach Running Application

Suppose an application is already running.

Then you execute:

```bash
export LOG_LEVEL="debug"
```

Do not assume the already running application suddenly receives the new value.

Environment inheritance occurs when processes are created.

Conceptually:

```text
Application started
LOG_LEVEL=info
       |
       v
Later shell changes:
LOG_LEVEL=debug
       |
       X
       |
Existing application remains unchanged
```

You may need to restart or otherwise reconfigure the application according to its design.

---

# 116. Problem: Variable Works Until You Log Out

You run:

```bash
export PROJECT_HOME="$HOME/projects"
```

Everything works.

Tomorrow, after logging in:

```bash
echo "$PROJECT_HOME"
```

returns nothing.

Cause:

The variable was temporary.

Fix:

Place the appropriate definition in a startup or application configuration mechanism that is actually loaded in the required context.

Then verify in a new session.

---

# 117. Problem: Variable Is in `.bashrc` but Still Missing

Possible causes:

* Current shell has not reloaded `.bashrc`.
* Shell is not Bash.
* Shell is non-interactive.
* Startup behavior does not read `.bashrc`.
* Syntax error occurred before the variable definition.
* Another configuration later overwrites it.
* Variable was not exported when a child process needs it.
* You edited another user's `.bashrc`.
* You edited the wrong home directory.

Investigate systematically.

---

# 118. Verify the Correct Home Directory

Run:

```bash
echo "$HOME"
```

Then:

```bash
ls -la "$HOME"
```

If you used:

```bash
sudo nano ~/.bashrc
```

remember that `~` expansion can be performed by your current shell before `sudo` runs.

Do not casually assume you are editing root's file just because the editor itself runs with `sudo`.

Use explicit paths when administrative context matters.

---

# 119. Problem: Variable Has Unexpected Spaces

Suppose:

```bash
PROJECT_NAME="Linux Lab"
```

This is valid.

But if you later use:

```bash
mkdir $PROJECT_NAME
```

unquoted expansion can cause word splitting, potentially resulting in two arguments:

```text
Linux
Lab
```

Instead:

```bash
mkdir "$PROJECT_NAME"
```

creates one directory named:

```text
Linux Lab
```

This is one reason quoting variable expansions is important.

---

# 120. Problem: Variable Contains an Old Value

Check where it is defined.

For example:

```bash
grep -n 'PROJECT_HOME' ~/.bashrc ~/.profile 2>/dev/null
```

You may discover:

```text
~/.profile:
export PROJECT_HOME="$HOME/project"

~/.bashrc:
export PROJECT_HOME="$HOME/projects"
```

Different startup paths may result in different values.

Avoid defining the same variable in many places unless you understand the precedence and loading behavior.

---

# 121. Problem: PATH Is Duplicated

Run:

```bash
echo "$PATH" | tr ':' '\n'
```

Suppose:

```text
/home/ann/bin
/home/ann/bin
/home/ann/bin
/usr/bin
/bin
```

Investigate startup files:

```bash
grep -n 'PATH=' ~/.bashrc ~/.profile 2>/dev/null
```

Look for repeated additions.

Do not simply delete random lines.

Determine why the file is being sourced repeatedly or why the same directory is added in multiple places.

---

# 122. Problem: Wrong Version Runs

You install a new tool under:

```text
/usr/local/bin/tool
```

but:

```bash
tool --version
```

shows an unexpected version.

Run:

```bash
command -v tool
```

Then:

```bash
type -a tool
```

Then:

```bash
echo "$PATH" | tr ':' '\n'
```

You may discover another executable appears earlier in command resolution.

The issue is not necessarily installation.

It may be command precedence.

---

# 123. Problem: Alias Changes Behavior

Suppose:

```bash
rm
```

behaves differently than expected.

Check:

```bash
type rm
```

If you see:

```text
rm is aliased to ...
```

you have found additional shell behavior.

Use:

```bash
alias rm
```

to inspect the alias.

Do not delete shell configuration until you understand where the alias was defined.

---

# 124. Problem: Works for My User, Fails for Another User

Suppose:

```bash
server-info
```

works for you.

Another user receives:

```text
command not found
```

Check:

```text
Your PATH
Their PATH
```

Your personal:

```text
$HOME/bin
```

is not automatically in another user's `PATH`.

Also consider:

* File permissions.
* Directory permissions.
* Ownership.
* Different startup files.
* Different shells.

Environment configuration is often user-specific.

---

# 125. Problem: Works Over SSH but Not Through systemd

You SSH into a server and run:

```bash
myapp
```

successfully.

But a systemd service cannot find:

```text
myapp
```

Do not assume systemd loads your `.bashrc`.

A systemd service has its own execution environment and service configuration.

The correct fix may involve:

* Explicit executable paths.
* Service environment configuration.
* Correct service user.
* Proper `Environment=` or environment file configuration where appropriate.

We will cover systemd service configuration more deeply in later practical material.

The important lesson is:

```text
Interactive shell environment
is not automatically
service environment.
```

---

# 126. Problem: Works Locally but Fails in CI/CD

Your terminal has:

```text
PATH=/home/ann/bin:...
```

Your CI runner may have:

```text
PATH=/usr/local/bin:/usr/bin:/bin
```

Therefore:

```bash
mytool
```

works locally but fails in the pipeline.

Troubleshoot the pipeline environment itself.

Do not say:

```text
But it works on my machine.
```

Instead ask:

```text
What environment does the CI process actually receive?
```

---

# 127. Environment Debugging Commands

These commands are useful:

```bash
env
```

```bash
printenv
```

```bash
printenv VARIABLE
```

```bash
echo "$VARIABLE"
```

```bash
export -p
```

```bash
echo "$PATH"
```

```bash
echo "$PATH" | tr ':' '\n'
```

```bash
command -v COMMAND
```

```bash
type COMMAND
```

```bash
type -a COMMAND
```

```bash
which COMMAND
```

```bash
echo "$SHELL"
```

```bash
echo "$BASH_VERSION"
```

```bash
echo "$-"
```

```bash
shopt login_shell
```

```bash
ps -p $$ -o pid,ppid,comm,args
```

These answer different questions.

---

# 128. A Better Environment Troubleshooting Process

When an environment-related problem occurs:

```text
Observe symptom
      |
      v
Identify process affected
      |
      v
Check variable in that environment
      |
      v
Is it defined?
      |
      v
Is it exported if child needs it?
      |
      v
Is value correct?
      |
      v
Where was it defined?
      |
      v
Which startup/configuration files were loaded?
      |
      v
Was variable later overwritten?
      |
      v
Restart/recreate process if necessary
      |
      v
Verify
```

This is much better than immediately adding random `export` commands to `.bashrc`.

---

# 129. Security: Do Not Put Secrets Everywhere

Avoid casually putting sensitive values directly into files such as:

```text
~/.bashrc
~/.profile
```

especially on shared or managed systems.

For example:

```bash
export DATABASE_PASSWORD="SuperSecretPassword"
```

may seem convenient, but it creates security and operational concerns.

Use the secret-management mechanism appropriate to the application and platform.

Later, when you learn CI/CD, Docker, Kubernetes, and cloud platforms, this becomes especially important.

---

# 130. Security: Do Not Echo Secrets

Suppose:

```text
API_TOKEN
```

contains a secret.

Avoid troubleshooting with:

```bash
echo "$API_TOKEN"
```

on shared screens, recorded terminals, CI logs, or environments where output is captured.

Instead, you can sometimes check whether it exists without printing the value.

For Bash:

```bash
if [ -n "${API_TOKEN:-}" ]; then
    echo "API_TOKEN is set"
else
    echo "API_TOKEN is not set"
fi
```

This confirms presence without intentionally printing the secret.

---

# 131. Security: PATH and Privileged Commands

`PATH` deserves additional care when using elevated privileges.

If an administrative script relies on:

```bash
some-command
```

and its `PATH` is unsafe, the wrong executable could potentially run.

Production scripts often use:

* Controlled `PATH` values.
* Explicit executable paths where appropriate.
* Restricted permissions on executable directories.

Do not place writable untrusted directories ahead of trusted system directories in privileged execution environments.

---

# 132. Shell Configuration Lab

Now we will build a complete beginner environment and shell-configuration lab.

Use a normal non-production Linux user.

Do not perform the lab as root unless a specific step requires administrative access.

---

# 133. Lab Step 1: Identify Your User

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
echo "$USER"
```

Compare the results.

Ask:

```text
Which values come from actual process credentials?

Which value comes from the environment?
```

---

# 134. Lab Step 2: Inspect Core Variables

Run:

```bash
printenv HOME
```

```bash
printenv USER
```

```bash
printenv SHELL
```

```bash
printenv PATH
```

Then:

```bash
echo "$HOME"
echo "$USER"
echo "$SHELL"
echo "$PATH"
```

Understand why both approaches can display environment values.

---

# 135. Lab Step 3: Inspect the Full Environment

Run:

```bash
env | less
```

Then:

```bash
env | grep '^HOME='
```

```bash
env | grep '^USER='
```

```bash
env | grep '^SHELL='
```

```bash
env | grep '^PATH='
```

Do not attempt to memorize every variable.

Your goal is to recognize the structure:

```text
NAME=value
```

---

# 136. Lab Step 4: Create a Shell Variable

Run:

```bash
LAB_NAME="Linux Environment Lab"
```

Then:

```bash
echo "$LAB_NAME"
```

Now:

```bash
printenv LAB_NAME
```

Observe the difference.

Explain:

```text
Why can echo see it while printenv may not?
```

Because it currently exists as a shell variable but has not been exported.

---

# 137. Lab Step 5: Export the Variable

Run:

```bash
export LAB_NAME
```

Then:

```bash
printenv LAB_NAME
```

Now:

```bash
bash -c 'echo "$LAB_NAME"'
```

The child Bash should receive it.

You have demonstrated:

```text
Shell variable
      |
      v
export
      |
      v
Environment variable
      |
      v
Inherited by child
```

---

# 138. Lab Step 6: Test Parent and Child Behavior

Set:

```bash
export COLOR="blue"
```

Start a child shell:

```bash
bash
```

Inside:

```bash
echo "$COLOR"
```

Then:

```bash
COLOR="red"
echo "$COLOR"
```

Exit:

```bash
exit
```

Back in the parent:

```bash
echo "$COLOR"
```

You should understand why the parent remains:

```text
blue
```

---

# 139. Lab Step 7: Test a Command-Scoped Variable

Run:

```bash
APP_ENV=testing printenv APP_ENV
```

Then:

```bash
printenv APP_ENV
```

If `APP_ENV` was not already exported, the second command should not display the testing value.

Explain why.

---

# 140. Lab Step 8: Practice Quoting

Run:

```bash
PROJECT_NAME="Linux Practice Project"
```

Then:

```bash
echo "$PROJECT_NAME"
```

Now create:

```bash
mkdir -p "$HOME/environment-lab"
cd "$HOME/environment-lab"
```

Run:

```bash
mkdir "$PROJECT_NAME"
```

Then:

```bash
ls -l
```

You should see one directory:

```text
Linux Practice Project
```

This demonstrates safe quoting.

---

# 141. Lab Step 9: Inspect PATH

Run:

```bash
echo "$PATH"
```

Then:

```bash
echo "$PATH" | tr ':' '\n'
```

Write down the directories in order.

Ask:

```text
Which directory is searched first?

Which is searched second?

Why does order matter?
```

---

# 142. Lab Step 10: Locate Commands

Run:

```bash
command -v ls
```

```bash
command -v bash
```

```bash
command -v cd
```

```bash
command -v python3
```

Then:

```bash
type ls
type cd
type bash
```

Observe the difference between external executables and shell builtins.

---

# 143. Lab Step 11: Create a Personal Command Directory

Run:

```bash
mkdir -p "$HOME/bin"
```

Verify:

```bash
ls -ld "$HOME/bin"
```

Now create:

```bash
nano "$HOME/bin/veriqta-info"
```

Add:

```bash
#!/usr/bin/env bash

echo "Linux Environment Lab"
echo "User: $(whoami)"
echo "Hostname: $(hostname)"
echo "Home: $HOME"
echo "Current directory: $(pwd)"
echo "Shell variable: $SHELL"
echo "Date: $(date)"
```

Save and exit.

---

# 144. Lab Step 12: Make the Script Executable

Run:

```bash
chmod 755 "$HOME/bin/veriqta-info"
```

Verify:

```bash
ls -l "$HOME/bin/veriqta-info"
```

You should see execute permissions.

Run using the absolute path:

```bash
"$HOME/bin/veriqta-info"
```

Verify the output.

---

# 145. Lab Step 13: Test Command Lookup

Check:

```bash
command -v veriqta-info
```

If `$HOME/bin` is not already in your `PATH`, it may return nothing.

Try:

```bash
veriqta-info
```

You may receive:

```text
command not found
```

Now you have intentionally created a safe PATH problem.

---

# 146. Lab Step 14: Fix PATH Temporarily

Run:

```bash
export PATH="$HOME/bin:$PATH"
```

Verify:

```bash
echo "$PATH" | tr ':' '\n'
```

Then:

```bash
command -v veriqta-info
```

Expected:

```text
/home/your-user/bin/veriqta-info
```

Now:

```bash
veriqta-info
```

should run.

---

# 147. Lab Step 15: Back Up `.bashrc`

Run:

```bash
cp ~/.bashrc ~/.bashrc.part11.backup
```

Verify:

```bash
ls -l ~/.bashrc ~/.bashrc.part11.backup
```

If `.bashrc` does not exist in your environment, investigate your shell before blindly creating configuration.

---

# 148. Lab Step 16: Add an Alias

Edit:

```bash
nano ~/.bashrc
```

Add:

```bash
alias labdir='cd "$HOME/environment-lab"'
```

Save.

Check syntax:

```bash
bash -n ~/.bashrc
```

Then:

```bash
source ~/.bashrc
```

Test:

```bash
labdir
pwd
```

You should arrive at:

```text
$HOME/environment-lab
```

---

# 149. Lab Step 17: Make PATH Persistent

If appropriate for your Bash environment, add:

```bash
export PATH="$HOME/bin:$PATH"
```

to your selected startup configuration.

Check syntax if editing `.bashrc`:

```bash
bash -n ~/.bashrc
```

Reload:

```bash
source ~/.bashrc
```

Then:

```bash
command -v veriqta-info
```

Open a new terminal or Bash session and test again.

The goal is to verify actual persistence, not merely assume it.

---

# 150. Lab Step 18: Create a Persistent Lab Variable

Add an appropriate variable definition to the startup file used for your intended session:

```bash
export VERIQTA_LAB="linux-part-11"
```

Reload or start a new relevant session.

Check:

```bash
printenv VERIQTA_LAB
```

Then:

```bash
bash -c 'printenv VERIQTA_LAB'
```

Verify that child processes receive it.

---

# 151. Lab Step 19: Inspect Shell Type

Run:

```bash
echo "$SHELL"
```

Then:

```bash
echo "$BASH_VERSION"
```

Then:

```bash
echo "$-"
```

Check whether:

```text
i
```

appears.

Then:

```bash
shopt login_shell
```

Finally:

```bash
ps -p $$ -o pid,ppid,comm,args
```

Write down:

* Configured login shell.
* Current shell process.
* Whether Bash is interactive.
* Whether Bash considers itself a login shell.

---

# 152. Lab Step 20: Inspect Startup Files

Run:

```bash
ls -la ~ | grep -E '\.bashrc|\.profile|\.bash_profile|\.bash_login'
```

Then inspect whichever files exist:

```bash
less ~/.bashrc
```

```bash
less ~/.profile
```

Do not assume every listed file exists.

Observe whether one file sources another.

---

# 153. Lab Step 21: Inspect System-Wide Configuration

Read only:

```bash
less /etc/profile
```

Then:

```bash
ls -l /etc/profile.d/
```

If present:

```bash
cat /etc/environment
```

Do not modify these files during the beginner lab.

Your goal is to recognize:

```text
User configuration
vs
system-wide configuration.
```

---

# 154. Lab Step 22: Create an Intentional Variable Failure

Run:

```bash
DATABASE_HOST="db.internal"
```

Check:

```bash
echo "$DATABASE_HOST"
```

Then:

```bash
bash -c 'echo "$DATABASE_HOST"'
```

The child may not see it.

Diagnose:

```text
Variable exists in parent shell.
Variable was not exported.
```

Fix:

```bash
export DATABASE_HOST
```

Verify:

```bash
bash -c 'echo "$DATABASE_HOST"'
```

---

# 155. Lab Step 23: Create an Intentional Command Failure

Create:

```bash
mkdir -p "$HOME/tools"
```

Then:

```bash
cp "$HOME/bin/veriqta-info" "$HOME/tools/tool-info"
```

Check:

```bash
ls -l "$HOME/tools/tool-info"
```

Try:

```bash
tool-info
```

If `$HOME/tools` is not in `PATH`, it fails.

Now:

```bash
"$HOME/tools/tool-info"
```

works.

Diagnose the problem before fixing it.

Then temporarily add:

```bash
export PATH="$HOME/tools:$PATH"
```

Verify:

```bash
command -v tool-info
```

Then:

```bash
tool-info
```

---

# 156. Lab Step 24: Investigate Command Precedence

Create two directories:

```bash
mkdir -p "$HOME/version1" "$HOME/version2"
```

Create:

```bash
nano "$HOME/version1/mytool"
```

Add:

```bash
#!/usr/bin/env bash
echo "Version 1"
```

Create:

```bash
nano "$HOME/version2/mytool"
```

Add:

```bash
#!/usr/bin/env bash
echo "Version 2"
```

Make both executable:

```bash
chmod 755 "$HOME/version1/mytool" "$HOME/version2/mytool"
```

Now:

```bash
export PATH="$HOME/version1:$HOME/version2:$PATH"
```

Run:

```bash
mytool
```

Expected:

```text
Version 1
```

Now:

```bash
export PATH="$HOME/version2:$HOME/version1:$PATH"
hash -r
```

Run:

```bash
mytool
```

Expected:

```text
Version 2
```

You have demonstrated why `PATH` order matters.

---

# 157. Lab Step 25: Use `type -a`

With both `mytool` executables available:

```bash
type -a mytool
```

You should see multiple possible locations.

Then:

```bash
command -v mytool
```

Observe which one would be selected first.

This is a powerful troubleshooting technique when multiple software versions exist.

---

# 158. Lab Step 26: Clean Up Temporary Variables

Run:

```bash
unset LAB_NAME
unset COLOR
unset DATABASE_HOST
```

Verify:

```bash
printenv LAB_NAME
printenv COLOR
printenv DATABASE_HOST
```

If you placed any of these variables in startup files, remove the corresponding configuration line there as well if you do not want them recreated.

---

# 159. Lab Step 27: Verify Everything

At the end of the lab, answer these questions using commands rather than guesses:

```text
Who am I?

What is my home directory?

What is my configured login shell?

Am I currently running Bash?

Is this Bash interactive?

Is it a login shell?

What is my PATH?

Which directory is searched first?

Where does ls resolve?

Is cd an external executable?

Where does veriqta-info resolve?

Can child processes see VERIQTA_LAB?

Which startup file defines my alias?

Which startup file modifies my PATH?
```

If you can answer those questions and explain why, you understand the foundation of Linux shell environments.

---

# 160. Real-World Scenario: Application Works Manually but Fails as a Service

You SSH into a server.

Run:

```bash
myapp
```

It works.

systemd starts the same application and it fails with:

```text
command not found
```

A beginner might think:

```text
systemd is broken.
```

A better investigation:

```text
Manual shell works
        |
        v
Which executable did shell use?
        |
        v
command -v myapp
        |
        v
/home/ann/bin/myapp
        |
        v
Does systemd service have same PATH?
        |
        v
No
        |
        v
Root cause:
Different execution environments
```

A robust service configuration should not depend blindly on one user's interactive `.bashrc`.

---

# 161. Real-World Scenario: Deployment Variable Missing

A deployment script expects:

```text
DEPLOY_ENV=production
```

Your terminal shows:

```bash
echo "$DEPLOY_ENV"
```

Output:

```text
production
```

But the deployment process says:

```text
DEPLOY_ENV is not set
```

Investigate:

1. Is it only a shell variable?
2. Was it exported?
3. Is the deployment running as a child of this shell?
4. Is another user executing it?
5. Is CI/CD executing it?
6. Is `sudo` changing the environment?
7. Is systemd launching it?
8. Is the process already running with an older environment?

The correct question is not merely:

```text
Is the variable set on the server?
```

The better question is:

```text
Is the variable available in the environment of the process that needs it?
```

---

# 162. Real-World Scenario: Wrong Terraform Version

Imagine:

```bash
terraform version
```

shows an older version than expected.

Do not immediately reinstall Terraform.

Run:

```bash
command -v terraform
```

Then:

```bash
type -a terraform
```

Then:

```bash
echo "$PATH" | tr ':' '\n'
```

You may discover:

```text
/usr/local/bin/terraform
/home/ann/bin/terraform
```

The older executable appears earlier in command resolution.

Root cause:

```text
PATH precedence
```

not necessarily failed installation.

---

# 163. Real-World Scenario: CI/CD Command Not Found

A deployment works from your SSH session:

```bash
kubectl version --client
```

but Jenkins reports:

```text
kubectl: command not found
```

Possible explanation:

```text
Your SSH shell
PATH includes /usr/local/bin

Jenkins process
PATH does not
```

Check the environment used by Jenkins.

Do not solve this by giving Jenkins access to your personal shell configuration without understanding the security and operational implications.

---

# 164. Real-World Scenario: Wrong `$HOME`

An application stores configuration under:

```text
$HOME/.config/application/
```

You run it as your user and it works.

Then you run it under another user.

The application cannot find its configuration.

Why?

Different user:

```text
HOME=/home/ann
```

versus:

```text
HOME=/home/deploy
```

The application may therefore search different paths.

Environment differences often explain:

```text
Works for me, fails for service account.
```

---

# 165. Real-World Scenario: Environment Change Did Not Affect Application

You run:

```bash
export LOG_LEVEL="debug"
```

But the running application continues producing normal logs.

Why?

The application was already running before you changed the shell environment.

Your new export affects future child processes of the shell.

It does not reach backward into an unrelated existing process.

You may need to restart the application with the intended environment.

---

# 166. Real-World Scenario: `.bashrc` Breaks New Shells

You accidentally add:

```bash
export PATH="/custom/tools"
```

to `.bashrc`.

You open another terminal.

Now:

```bash
ls
```

returns:

```text
command not found
```

Reason:

You replaced the entire `PATH`.

Use an absolute path if necessary:

```bash
/usr/bin/nano ~/.bashrc
```

Correct the line to preserve the existing path, for example:

```bash
export PATH="/custom/tools:$PATH"
```

provided `/custom/tools` is genuinely required.

Then validate:

```bash
bash -n ~/.bashrc
```

Open another shell and test.

---

# 167. What You Should Now Understand

Before continuing to Part 12, you should confidently understand:

* What a process environment is.
* What environment variables are.
* Why applications use environment variables.
* How environment variables are represented as names and values.
* How to inspect the environment with `env`.
* How to use `printenv`.
* What `$HOME` represents.
* What `$USER` represents.
* Why `$USER` is not authoritative proof of identity.
* What `$SHELL` commonly represents.
* Why `$SHELL` may not identify the exact currently executing shell.
* What `$PATH` is.
* Why `PATH` exists.
* How directories are separated inside `PATH`.
* How to display `PATH` one directory per line.
* How external command lookup uses `PATH`.
* Why `PATH` order matters.
* Why an absolute executable path bypasses normal `PATH` lookup.
* Why `./script.sh` may work while `script.sh` does not.
* What `which` does.
* What `command -v` does.
* Why `command -v` is useful.
* What shell builtins are.
* Why `cd` is a shell builtin.
* How `type` helps investigate command resolution.
* What shell variables are.
* How shell variables differ from exported environment variables.
* How to create variables.
* Why spaces around `=` break assignments.
* Why variable names are case-sensitive.
* What `export` does.
* How child processes inherit exported variables.
* Why child environment changes do not modify the parent shell.
* How to create a variable for one command.
* What `unset` does.
* Why environment variables are normally temporary unless recreated.
* What persistent shell configuration really means.
* Why variables should usually be quoted during expansion.
* How double quotes differ from single quotes.
* Why `${VARIABLE}` syntax is useful.
* What shell expansion means.
* What variable expansion is.
* What tilde expansion is.
* What command substitution is.
* How wildcard expansion fits into shell processing.
* What aliases are.
* How to create aliases.
* How to inspect aliases.
* How to remove aliases.
* How aliases can change command behavior.
* What `.bashrc` is.
* What `.profile` is.
* Why startup files are hidden files.
* How to back up `.bashrc`.
* What `source` does.
* Why sourcing differs from running a script normally.
* What an interactive shell is.
* What a non-interactive shell is.
* What a login shell is.
* Why login and interactive are separate properties.
* How to determine whether Bash is interactive.
* How to determine whether Bash is a login shell.
* How to inspect the current shell process.
* Why Bash startup files differ by shell mode.
* Why you should not assume every shell reads `.bashrc`.
* How to make environment settings persistent in an appropriate context.
* How to add a directory to `PATH`.
* The difference between prepending and appending `PATH`.
* Why replacing `PATH` accidentally can break basic commands.
* How to recover from a broken `PATH`.
* Why adding `.` to `PATH` can create security problems.
* Why empty `PATH` entries can be dangerous.
* How to create a personal executable directory.
* How to make a script executable.
* How to make a personal script discoverable through `PATH`.
* Why duplicate `PATH` entries can occur.
* How command hashing can affect troubleshooting.
* What `/etc/profile` is used for.
* What `/etc/profile.d/` contains.
* What `/etc/environment` is.
* Why `/etc/environment` should not automatically be treated like a Bash script.
* The difference between user-specific and system-wide configuration.
* Why system-wide changes should be used carefully.
* How to syntax-check `.bashrc`.
* Why you should keep a working terminal open while testing shell changes.
* How to troubleshoot `command not found`.
* How to troubleshoot missing environment variables.
* Why a shell variable may not be visible to a child application.
* Why an already running application does not automatically receive later environment changes.
* Why terminal, systemd, cron, CI/CD, and `sudo` environments may differ.
* Why environment variables should not automatically be treated as secure secret storage.
* Why `PATH` security matters for privileged commands.

---

# 168. Hands-On Practice

Complete these tasks without copying the solutions until you have tried them yourself.

1. Display all environment variables.
2. Display only `HOME`.
3. Display only `USER`.
4. Display only `SHELL`.
5. Display only `PATH`.
6. Display `PATH` one directory per line.
7. Determine your real effective username.
8. Compare `whoami` with `$USER`.
9. Change a normal shell variable and prove that it does not change your actual user identity.
10. Create `COURSE="Linux"`.
11. Display it.
12. Determine whether `printenv` can see it.
13. Export it.
14. Prove that a child Bash can now see it.
15. Remove it with `unset`.
16. Create a variable containing spaces.
17. Use it safely with quotes.
18. Demonstrate the difference between single and double quotes.
19. Use `${VARIABLE}` to join a variable with additional text.
20. Create a variable for only one command.
21. Prove that the value does not remain afterward.
22. Determine where `ls` resolves.
23. Determine whether `cd` is a builtin.
24. Use `type -a` on a command.
25. Create `$HOME/bin`.
26. Create an executable script inside it.
27. Run the script using its full path.
28. Attempt to run it by name before adding the directory to `PATH`.
29. Explain why it fails.
30. Add `$HOME/bin` to `PATH` temporarily.
31. Verify the new `PATH`.
32. Locate the script with `command -v`.
33. Run it by name.
34. Make the change persistent.
35. Open a new shell.
36. Confirm the command still works.
37. Create an alias.
38. Make the alias persistent.
39. Remove the temporary alias with `unalias`.
40. Explain why it can return in a new shell if still defined in `.bashrc`.
41. Back up `.bashrc`.
42. Inspect `.bashrc`.
43. Inspect `.profile`.
44. Check `.bashrc` syntax.
45. Source `.bashrc`.
46. Determine whether your current Bash is interactive.
47. Determine whether it is a login shell.
48. Inspect the current shell PID and parent PID.
49. Inspect `/etc/profile`.
50. Inspect `/etc/profile.d/`.
51. Inspect `/etc/environment` if present.
52. Explain user-specific versus system-wide shell configuration.
53. Create two executables with the same name in different directories.
54. Put both directories in `PATH`.
55. Change their order.
56. Prove that command resolution changes.
57. Use `type -a` to find both.
58. Intentionally create an unexported variable.
59. Run a child process that cannot see it.
60. Diagnose the problem.
61. Export it.
62. Verify the fix.
63. Explain why changing the variable in the child does not change the parent.
64. Explain why an existing application does not receive new environment variables automatically.
65. Explain why `.bashrc` may not affect a systemd service.
66. Explain why a command might work through SSH but fail through CI/CD.
67. Explain why `$HOME` may differ between users.
68. Explain why `sudo` may change environment behavior.
69. Explain why an environment variable should not automatically be considered a secure secret.
70. Write your own environment troubleshooting checklist.

---

# 169. Beginner Knowledge Check

1. What is a process environment?
2. What is an environment variable?
3. Why do programs use environment variables?
4. What format is commonly used to represent an environment variable?
5. What does `env` show?
6. What does `printenv` do?
7. What does `$HOME` usually contain?
8. What does `$USER` usually contain?
9. Why can `$USER` differ conceptually from `whoami`?
10. What does `$SHELL` commonly represent?
11. Does `$SHELL` always prove which shell process is currently running?
12. What is `$PATH`?
13. Why does `PATH` exist?
14. What separates directories in `PATH`?
15. Why does `ls` normally work without `/usr/bin/ls`?
16. Why does `PATH` order matter?
17. What happens if two directories contain executables with the same name?
18. What is an absolute executable path?
19. Why might `./script.sh` work while `script.sh` does not?
20. Why is the current directory normally not automatically searched?
21. What does `which` help you determine?
22. What does `command -v` help determine?
23. What is a shell builtin?
24. Why is `cd` normally a builtin?
25. What is a shell variable?
26. How do you create one?
27. Why should there not be spaces around `=`?
28. What is an environment variable compared with an ordinary shell variable?
29. What does `export` do?
30. How do child processes receive environment variables?
31. Can a child process normally modify its parent's environment?
32. What happens to temporary variables when the shell ends?
33. How can a variable be supplied to one command?
34. What does `unset` do?
35. Why should variable expansions usually be quoted?
36. What happens to `$NAME` inside double quotes?
37. What happens to `$NAME` inside single quotes?
38. Why would you use `${NAME}`?
39. What is shell expansion?
40. What is variable expansion?
41. What is command substitution?
42. What is an alias?
43. How do you create an alias?
44. How do you list aliases?
45. How do you remove an alias?
46. Can aliases affect command troubleshooting?
47. What is `.bashrc`?
48. What is `.profile`?
49. Why are those files hidden?
50. What does `source ~/.bashrc` do?
51. Why is sourcing different from executing a normal child script?
52. What is an interactive shell?
53. What is a non-interactive shell?
54. What is a login shell?
55. Can an interactive shell also be a login shell?
56. How can you check whether Bash is interactive?
57. How can you check whether Bash is a login shell?
58. Why should you not assume every Bash session reads the same startup files?
59. What does persistent shell configuration actually mean?
60. How do you add `$HOME/bin` to the beginning of `PATH`?
61. How do you add it to the end?
62. What is the difference between prepending and appending?
63. Why is `export PATH="$HOME/bin"` dangerous?
64. Does adding a directory to `PATH` create that directory?
65. Does adding a directory to `PATH` install software?
66. Why can adding `.` to `PATH` be dangerous?
67. Why can empty `PATH` components be dangerous?
68. What should you check after changing `PATH`?
69. What does `type -a` help reveal?
70. What does `hash -r` do in Bash?
71. What is `/etc/profile`?
72. What is `/etc/profile.d/`?
73. What is `/etc/environment`?
74. Should `/etc/environment` automatically be treated as a Bash script?
75. What is the difference between user and system-wide configuration?
76. Why should system-wide changes be made carefully?
77. How can you check `.bashrc` syntax?
78. Why should you keep an existing terminal open when testing major shell configuration changes?
79. What are common causes of `command not found`?
80. How can you determine whether an executable exists outside `PATH`?
81. What should you check when an application cannot see a variable?
82. Why can `echo "$VARIABLE"` work while a child application cannot see the variable?
83. Why does exporting fix that situation?
84. Will exporting a variable update an application that is already running?
85. Why might a variable work in your terminal but not through systemd?
86. Why might a command work locally but not in CI/CD?
87. Why might the same command work for one user and fail for another?
88. How can `PATH` cause the wrong software version to run?
89. Why can aliases make troubleshooting confusing?
90. Why can `$HOME` change application behavior?
91. Does `sudo` necessarily preserve your entire environment?
92. Why is `sudo echo "$HOME"` potentially misleading when studying environment behavior?
93. Are environment variables automatically secure places for secrets?
94. Why should you avoid printing secret environment variables?
95. Why is `PATH` important to security?
96. What is the difference between a missing command and a missing package?
97. What is the difference between a missing executable and an executable outside `PATH`?
98. What is the difference between changing a variable and making it persistent?
99. What is the difference between your interactive environment and a service environment?
100. What troubleshooting process should you follow when an environment-related application failure occurs?

---

# 170. Beginner Knowledge Check Answers

1. The collection of exported named values available to a process.

2. A named value passed as part of a process environment.

3. To receive configuration and contextual information without hard-coding every value.

4. `NAME=value`.

5. Environment variables available to the process running `env`.

6. It prints environment values, either broadly or for specified variable names.

7. The user's home directory.

8. A username value associated with the environment.

9. `$USER` is an environment value and can be changed, while `whoami` reports the effective username from process credentials.

10. The user's configured login shell.

11. No.

12. An environment variable containing directories used during external command lookup.

13. So commands can be invoked by name without always typing full executable paths.

14. Colons.

15. The shell resolves the command, commonly finding an executable through directories listed in `PATH`.

16. Earlier matching executable locations take precedence during `PATH` search.

17. The relevant command found first according to command resolution and `PATH` order is normally used.

18. A path beginning from `/` that directly identifies an executable.

19. `./script.sh` explicitly identifies a file in the current directory.

20. Primarily to avoid unsafe and unpredictable command execution from arbitrary working directories.

21. The location of a command found through its lookup behavior, commonly based on `PATH`.

22. How the shell resolves a command name.

23. A command implemented by the shell itself rather than requiring a separate executable process.

24. It must change the current shell's working directory.

25. A named value maintained by the shell.

26. For example:

```bash
NAME="Ann"
```

27. Spaces separate shell words, so the expression would no longer be parsed as the intended assignment.

28. An exported variable is included in the environment passed to child processes.

29. It marks a shell variable for inclusion in the environment of subsequently started child processes.

30. They inherit a copy of the exported parent environment when created.

31. Normally, no.

32. They disappear with that process unless recreated by future configuration.

33. For example:

```bash
APP_ENV=testing ./app
```

34. It removes a variable from the current shell.

35. To prevent unwanted word splitting and pathname expansion after expansion.

36. Bash expands the variable.

37. It remains literal text.

38. To clearly mark the boundaries of the variable name.

39. The shell's transformation of command text before execution.

40. Replacing a variable reference with its value.

41. Replacing `$(command)` with command output.

42. A shell-defined shortcut or replacement for another command string.

43. For example:

```bash
alias ll='ls -lah'
```

44.

```bash
alias
```

45.

```bash
unalias ll
```

46. Yes. An alias can change what happens when you type a command name.

47. A Bash startup/configuration file commonly used for interactive non-login Bash shells.

48. A user startup file commonly associated with login-session configuration.

49. Their names begin with `.`.

50. It executes the contents of `.bashrc` in the current shell.

51. Sourcing affects the current shell, while normal script execution usually occurs in another process.

52. A shell that accepts commands directly from a user.

53. A shell running commands without ongoing interactive prompting.

54. A shell started with login-shell semantics.

55. Yes.

56.

```bash
echo "$-"
```

and check for `i`.

57.

```bash
shopt login_shell
```

58. Bash uses different startup rules depending on login and interactive status and how it was invoked.

59. Future processes execute configuration that recreates the desired values.

60.

```bash
export PATH="$HOME/bin:$PATH"
```

61.

```bash
export PATH="$PATH:$HOME/bin"
```

62. Prepending searches the new directory earlier. Appending searches it later.

63. It replaces the existing `PATH`, potentially making system commands unavailable by name.

64. No.

65. No.

66. It can allow executables from the current directory to take precedence unexpectedly.

67. They can cause the current directory to participate in command lookup in shell environments where empty components have that meaning.

68. Inspect the new `PATH` and verify important commands and the intended new command resolve correctly.

69. Multiple possible definitions or executable locations for a command.

70. It clears Bash's remembered command-location hash table.

71. A system-wide startup file commonly associated with login shell configuration.

72. A directory commonly containing modular system-wide profile scripts.

73. A file used by some system components for environment configuration.

74. No.

75. User configuration applies to a particular user. System-wide configuration can affect multiple users.

76. A mistake can affect many accounts or applications.

77.

```bash
bash -n ~/.bashrc
```

78. It gives you a working session from which you can repair a broken configuration.

79. Misspelling, software not installed, incorrect `PATH`, executable outside `PATH`, different shell environment, or incorrect assumptions about command names.

80. Locate or inspect the expected file directly and try its absolute path.

81. Whether it exists, has the correct value, is exported, and is available in the actual process environment.

82. The shell itself can expand its own shell variable, while a child process only inherits exported variables.

83. It includes the variable in environments passed to subsequently created child processes.

84. No.

85. systemd does not automatically use the same environment as your interactive shell.

86. CI/CD runners can have different users, `PATH` values, variables, startup behavior, and installed software.

87. Users can have different home directories, permissions, startup files, shells, and `PATH` values.

88. An older executable can appear earlier in command resolution than the version you intended to use.

89. The typed command may expand to something different from what you expected.

90. Applications often derive configuration paths and file locations from the user's home directory.

91. No.

92. Your current shell may expand `$HOME` before `sudo` executes the command.

93. No.

94. Terminal output, logs, recordings, and CI systems may expose the value.

95. It controls where executables are searched, so unsafe directories or precedence can cause unintended programs to run.

96. A command can be missing even though a related package is installed, and a package can be absent even though a manually installed command exists.

97. A missing executable does not exist at the expected location. An executable outside `PATH` exists but cannot be found by normal command-name lookup.

98. Changing a variable modifies the current environment. Persistence requires future environments to recreate the setting through appropriate configuration.

99. They are different process contexts and may have completely different variables, users, startup behavior, and `PATH` values.

100. Identify the affected process, inspect its actual environment, verify the variable exists and is exported where necessary, check its value and source, understand the startup/configuration path, correct the smallest root cause, restart or recreate the affected process when necessary, and verify the result.

---

# 171. Final Mental Model

The most important idea from this part is that programs do not run in isolation.

They run inside process environments.

```text
Linux user
    |
    v
Shell starts
    |
    +--> Startup configuration
    |
    +--> Shell variables
    |
    +--> Exported environment variables
    |
    +--> PATH
    |
    +--> Aliases
    |
    v
You enter a command
    |
    v
Shell parses and expands it
    |
    v
Command resolution
    |
    +--> Alias?
    +--> Function?
    +--> Builtin?
    +--> Executable through PATH?
    |
    v
Process starts
    |
    v
Receives exported environment
```

For variables:

```text
Shell variable

PROJECT=veriqta
      |
      v
Current shell can use it
      |
      X
Child does not automatically receive it
```

After:

```bash
export PROJECT
```

the model becomes:

```text
Parent shell
PROJECT=veriqta
      |
      v
Creates child
      |
      v
Child receives PROJECT=veriqta
```

But:

```text
Child changes PROJECT
      |
      X
      |
Parent is not changed
```

For `PATH`:

```text
You type:

mytool
   |
   v
Shell resolves command
   |
   v
Search PATH directories in order
   |
   +--> /home/ann/bin
   |
   +--> /usr/local/bin
   |
   +--> /usr/bin
   |
   +--> /bin
   |
   v
First appropriate executable found
   |
   v
Execute
```

And when something works in your terminal but fails somewhere else, think:

```text
My terminal
     |
     +--> My user
     +--> My HOME
     +--> My PATH
     +--> My exported variables
     +--> My startup files
     +--> My aliases
     |
     v
Works
```

versus:

```text
systemd / cron / CI / another user
     |
     +--> Different user?
     +--> Different HOME?
     +--> Different PATH?
     +--> Different variables?
     +--> Different shell?
     +--> Different startup behavior?
     |
     v
Fails
```

The correct troubleshooting question is therefore not:

```text
Does this variable exist somewhere on the server?
```

It is:

```text
What environment does the process that needs this variable actually have?
```

And when a command cannot be found, do not immediately install or reinstall software.

Think:

```text
command not found
      |
      v
Is the command spelled correctly?
      |
      v
Does the executable exist?
      |
      v
Is its directory in PATH?
      |
      v
Which executable does command -v find?
      |
      v
Are multiple versions present?
      |
      v
Is an alias/function involved?
      |
      v
Is this the same user and shell environment?
      |
      v
Which startup configuration created this environment?
```

Once you understand that model, environment variables stop being mysterious strings beginning with `$`. They become part of a predictable process model that you can inspect, configure, and troubleshoot.
