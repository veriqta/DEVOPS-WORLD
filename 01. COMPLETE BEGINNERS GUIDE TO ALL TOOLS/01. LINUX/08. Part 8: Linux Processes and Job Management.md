# Part 8: Linux Processes and Job Management

In Part 7, you learned how Linux commands communicate through standard input, standard output, standard error, pipes, redirection, and command chaining.

Now we move to another core Linux concept: processes.

Every command you run, web server you start, script you execute, database you operate, and monitoring agent running on a Linux server ultimately involves processes.

Understanding processes is essential for Linux administration, DevOps, Cloud, SRE, and Platform Engineering.

By the end of this part, you should be able to:

* Explain the difference between a program and a process.
* Understand how Linux starts processes.
* Identify processes using PIDs.
* Understand parent and child processes.
* Recognize foreground and background processes.
* Understand common process states.
* Inspect processes using `ps`, `top`, `htop`, and `pgrep`.
* Interpret basic CPU and memory usage.
* Start commands in the background.
* Manage shell jobs using `jobs`, `bg`, and `fg`.
* Understand `Ctrl+C` and `Ctrl+Z`.
* Understand Linux signals.
* Stop processes safely with `kill`.
* Explain `SIGTERM` and `SIGKILL`.
* Understand why `kill -9` should usually be a last resort.
* Keep long-running commands alive with `nohup`.
* Understand zombie processes.
* Investigate high CPU usage.
* Investigate high memory usage.
* Troubleshoot stuck processes.
* Follow a repeatable process-troubleshooting workflow.

---

# 1. What Is a Program?

A program is executable code stored somewhere on a system.

Examples include:

```text
/usr/bin/bash
/usr/bin/ls
/usr/bin/python3
/usr/bin/grep
```

Applications such as NGINX, SSH servers, databases, monitoring agents, and container runtimes also consist of executable programs and supporting files.

A program sitting on disk is not necessarily doing anything.

Think of it as instructions waiting to be executed.

```text
Program on disk
      |
      | executed
      v
Running process
```

---

# 2. What Is a Process?

A process is a running instance of a program.

Suppose Bash exists at:

```bash
/usr/bin/bash
```

That is a program.

When Bash runs, Linux creates a process for it.

Likewise:

```bash
sleep 60
```

starts a `sleep` process.

A process has more than program code. Linux must track information such as:

* Process ID.
* Parent process.
* User.
* Memory.
* CPU activity.
* Open files.
* Environment.
* Current state.
* Working directory.
* Permissions.
* Signals.
* Other kernel-managed resources.

A simplified view is:

```text
Program
   |
   v
Process
   |
   +--> PID
   +--> user
   +--> memory
   +--> CPU activity
   +--> open files
   +--> environment
   +--> state
   +--> parent
```

---

# 3. Program vs Process

The distinction is important.

A program is executable code.

A process is an executing instance of that code.

You can run the same program multiple times.

For example, open several terminals and run:

```bash
sleep 300
```

in each one.

The same `sleep` program is being used, but Linux creates separate processes.

Conceptually:

```text
             sleep program
                  |
        +---------+---------+
        |         |         |
        v         v         v
     Process    Process    Process
     PID 4201   PID 4217   PID 4259
```

Each process can be managed independently.

---

# 4. Why Processes Matter

Imagine users report that an application is slow.

Possible causes include:

```text
Application process consuming too much CPU

Application process consuming too much memory

Process stuck or blocked

Too many processes

Required process stopped

Process repeatedly crashing

Child processes not exiting

System overloaded by another process
```

Before restarting a server, you need to understand what is happening.

Process knowledge lets you investigate rather than guess.

---

# 5. How Processes Start

At a beginner level, you can think of process creation like this:

```text
Existing process
      |
      | starts another program
      v
New process
```

For example, you have a shell:

```text
Bash shell
```

You type:

```bash
sleep 60
```

Your shell causes the command to be executed as a process.

Conceptually:

```text
Terminal
   |
   v
Bash
   |
   +----> sleep
```

Bash is related to the new process as its parent.

The `sleep` process is the child.

The exact kernel mechanisms behind process creation can involve system calls such as `fork()`, `clone()`, and `execve()`. You do not need to master those system calls yet.

The beginner concept is:

```text
A running process can cause other processes to be created.
```

---

# 6. PID

Every process has a process ID, normally called a PID.

PID means:

```text
Process ID
```

It is a number Linux uses to identify a process.

Example:

```text
PID 4321
```

You might have:

```text
PID     COMMAND
4321    sleep
4388    bash
4510    python3
```

A PID lets you tell Linux:

```text
Show me information about process 4321.
```

or:

```text
Send a signal to process 4321.
```

---

# 7. Why PIDs Matter

Imagine three Python programs are running:

```text
PID     COMMAND
3101    python3
3150    python3
3208    python3
```

Saying:

```text
Stop Python.
```

is ambiguous.

Which process?

The PID provides precise identification.

You could target:

```text
PID 3150
```

without intentionally targeting the other two.

---

# 8. Finding Your Shell's PID

Bash provides:

```bash
echo $$
```

In Bash, `$$` expands to the PID of the current shell.

Example:

```text
6821
```

Now try:

```bash
ps -p $$
```

You may see something similar to:

```text
PID TTY          TIME CMD
6821 pts/0    00:00:00 bash
```

Your exact output will differ.

---

# 9. PID 1

Linux has a particularly important process:

```text
PID 1
```

On many modern Linux distributions, PID 1 is:

```text
systemd
```

You can inspect it:

```bash
ps -p 1
```

Or:

```bash
ps -p 1 -f
```

On many systems you may see:

```text
UID          PID    PPID  C STIME TTY          TIME CMD
root           1       0  0 ...   ?        00:00:... /sbin/init
```

or a command involving `systemd`.

Containers and specialized environments may use something different as PID 1.

We will study `systemd` and service management separately.

---

# 10. PPID

Processes also have a PPID.

PPID means:

```text
Parent Process ID
```

It identifies the process that is considered the parent of another process.

Example:

```text
PID     PPID    COMMAND
5000    4900    bash
5100    5000    sleep
```

Here:

```text
bash PID = 5000
sleep PID = 5100
sleep PPID = 5000
```

Therefore Bash is the parent of `sleep`.

---

# 11. Parent and Child Processes

Processes often form relationships.

Example:

```text
Terminal
   |
   v
Bash
PID 5000
   |
   +------> sleep
   |        PID 5100
   |
   +------> python3
            PID 5200
```

Both `sleep` and `python3` were started from Bash.

They may therefore be child processes of that shell.

---

# 12. Seeing Parent and Child Information

Run:

```bash
sleep 300 &
```

Your shell may print something such as:

```text
[1] 7312
```

Now:

```bash
ps -o pid,ppid,stat,cmd -p 7312
```

Example:

```text
PID    PPID STAT CMD
7312   6821 S    sleep 300
```

You can compare its PPID with:

```bash
echo $$
```

If the shell PID is `6821`, the relationship is visible:

```text
bash
PID 6821
   |
   v
sleep
PID 7312
PPID 6821
```

---

# 13. Process Trees

Processes can form trees.

For example:

```text
systemd
|
+-- sshd
|   |
|   +-- sshd
|       |
|       +-- bash
|           |
|           +-- python3
|
+-- nginx
|   |
|   +-- nginx worker
|   +-- nginx worker
|
+-- cron
```

Understanding the tree can help answer:

```text
Who started this process?

What process owns these workers?

Why do these child processes keep returning?

Should I stop the parent instead?
```

If available, `pstree` is useful:

```bash
pstree
```

Or:

```bash
pstree -p
```

The `-p` option includes PIDs.

If `pstree` is unavailable, it may require a package such as `psmisc`, depending on the distribution.

---

# 14. Foreground Processes

A foreground process is attached to your terminal in a way that normally occupies the shell until the command finishes, stops, or is interrupted.

Run:

```bash
sleep 60
```

Your shell prompt does not immediately return.

Conceptually:

```text
Shell prompt
    |
    v
sleep 60
    |
    | running in foreground
    |
    v
Shell waits
```

When `sleep` finishes, your prompt returns.

---

# 15. Foreground Does Not Mean "Visible Application"

Foreground in shell job control does not simply mean:

```text
Something I can see on screen.
```

It describes which job currently has foreground control of the terminal.

A command can produce no output and still be a foreground process.

For example:

```bash
sleep 60
```

prints almost nothing, but it occupies the foreground.

---

# 16. Background Processes

A background process allows the shell prompt to return while the command continues.

Run:

```bash
sleep 300 &
```

The:

```text
&
```

asks the shell to start the command as a background job.

You may see:

```text
[1] 7501
```

Then your prompt returns.

---

# 17. Understanding `[1] 7501`

The output:

```text
[1] 7501
```

contains two different identifiers.

```text
[1]
```

is the shell's job number.

```text
7501
```

is a process ID.

Do not confuse job IDs with PIDs.

You may refer to a shell job as:

```text
%1
```

while process tools normally use a PID such as:

```text
7501
```

---

# 18. Process vs Job

A process is a kernel-managed running entity.

A job is a shell job-control concept.

A job may involve one process or multiple processes, particularly when pipelines are involved.

For example:

```bash
sleep 300 | cat &
```

can represent one shell job involving multiple processes.

So:

```text
Process
Kernel concept

Job
Shell job-control concept
```

---

# 19. Process States

A process is not simply:

```text
running
```

or:

```text
stopped
```

Linux tracks different process states.

Common states you may encounter include:

```text
R = running or runnable
S = interruptible sleep
D = uninterruptible sleep
T = stopped or traced
Z = zombie
```

You may see additional letters and modifiers depending on the tool.

---

# 20. Running or Runnable State: `R`

`R` generally means the process is running on a CPU or is runnable and waiting for CPU time.

Example:

```text
STAT
R
```

A process does not have to be executing on a CPU every nanosecond to be considered runnable.

Linux schedules many processes across available CPU cores.

---

# 21. Sleeping State: `S`

`S` commonly means interruptible sleep.

The process is waiting for something.

That might be:

* Input.
* A timer.
* A network event.
* Another resource.
* Another event from the kernel.

For example:

```bash
sleep 300
```

does not need to burn CPU continuously for 300 seconds.

Most of the time it waits.

You will commonly see many healthy Linux processes in sleeping states.

Sleeping does not mean broken.

---

# 22. Uninterruptible Sleep: `D`

`D` generally represents uninterruptible sleep.

A process in this state is commonly waiting inside the kernel for an operation such as I/O to complete.

For example, problems involving:

* Storage.
* Network filesystems.
* Block devices.
* Certain kernel-level I/O operations.

can sometimes contribute to processes remaining in `D` state.

A process stuck in `D` deserves investigation.

Simply sending stronger kill signals may not solve the underlying problem while the kernel operation remains blocked.

---

# 23. Stopped State: `T`

`T` indicates a stopped or traced process.

You can create a stopped job using:

```text
Ctrl+Z
```

while a foreground process is running.

For example:

```bash
sleep 300
```

Then press:

```text
Ctrl+Z
```

You may see:

```text
[1]+  Stopped    sleep 300
```

The process has not completed.

It has been suspended.

---

# 24. Zombie State: `Z`

`Z` means zombie.

A zombie process has already finished execution, but its parent has not yet collected its termination status.

This can sound strange.

The process is already dead in the sense that it is no longer executing normal program code.

But a small process-table entry remains.

Conceptually:

```text
Child process runs
       |
       v
Child exits
       |
       v
Kernel keeps exit status
       |
       v
Parent should collect status
       |
       +---- collected ----> entry removed
       |
       +---- not collected -> zombie
```

We will return to zombies later.

---

# 25. Viewing Processes With `ps`

`ps` means process status.

Run:

```bash
ps
```

You may see something like:

```text
PID TTY          TIME CMD
6821 pts/0    00:00:00 bash
7812 pts/0    00:00:00 ps
```

By itself, `ps` usually displays only a limited selection of processes associated with your current context.

For system troubleshooting, you often need broader options.

---

# 26. Understanding Basic `ps` Output

Common columns include:

```text
PID
TTY
TIME
CMD
```

### PID

Process ID.

### TTY

The terminal associated with the process, when applicable.

### TIME

CPU time consumed by the process, not simply how long it has existed.

### CMD

The command.

---

# 27. `ps aux`

A widely used command is:

```bash
ps aux
```

This uses BSD-style options, which is why you commonly see it without a leading dash.

The output may include columns such as:

```text
USER PID %CPU %MEM VSZ RSS TTY STAT START TIME COMMAND
```

This is one of the most useful process snapshots for beginners.

---

# 28. Understanding `ps aux`

Example:

```text
USER      PID %CPU %MEM    VSZ   RSS TTY STAT START TIME COMMAND
root        1  0.0  0.2  ...    ... ?   Ss   ...   ... /sbin/init
ann      8120 25.0  1.4  ...    ... ?   R    ...   ... python3 app.py
```

Important fields:

### USER

The user that owns the process.

### PID

Process ID.

### %CPU

CPU utilization reported for the process.

### %MEM

Percentage of physical memory associated with the process according to the tool's calculation.

### VSZ

Virtual memory size.

### RSS

Resident Set Size, roughly the portion of process memory currently resident in physical RAM.

### TTY

Associated terminal.

### STAT

Process state plus possible modifiers.

### START

When the process started.

### TIME

Accumulated CPU time.

### COMMAND

Command and arguments.

---

# 29. VSZ vs RSS

Beginners often misunderstand these.

Suppose:

```text
VSZ = 900000
RSS = 120000
```

Do not immediately conclude:

```text
This process is physically consuming 900 MB of RAM.
```

VSZ represents the process's virtual address-space size and can include mappings that are not all resident in physical memory.

RSS represents resident memory pages and is often more useful for a quick look at actual RAM residency.

Even RSS requires context because memory can be shared between processes.

For beginner troubleshooting:

```text
%MEM + RSS
```

are useful starting points.

Do not treat every memory column as interchangeable.

---

# 30. `ps -ef`

Another common process command is:

```bash
ps -ef
```

This uses traditional UNIX-style options.

You may see:

```text
UID PID PPID C STIME TTY TIME CMD
```

This view is particularly useful when you care about:

```text
PID
PPID
user
command
```

---

# 31. `ps aux` vs `ps -ef`

Both are widely used.

Use:

```bash
ps aux
```

when you want a convenient view including:

```text
CPU
memory
state
command
```

Use:

```bash
ps -ef
```

when a traditional full-format listing with PPID is useful.

Neither is universally "better."

They present process information differently.

---

# 32. Custom `ps` Output

You do not always need every column.

For example:

```bash
ps -eo pid,ppid,user,stat,%cpu,%mem,cmd
```

Here:

```text
-e
all processes

-o
choose output fields
```

This can produce a clean troubleshooting view.

You can also inspect one process:

```bash
ps -p 8120 -o pid,ppid,user,stat,%cpu,%mem,etime,cmd
```

Replace `8120` with a real PID.

`etime` shows elapsed time since the process started.

---

# 33. `ps` Is a Snapshot

This is important.

`ps` shows a snapshot of process information at the time you run it.

It does not continuously refresh.

Think:

```text
ps
|
v
Take picture of process state
|
v
Display
|
v
Exit
```

For live monitoring, use tools such as:

```text
top
htop
```

---

# 34. Finding Processes With `grep`

A common beginner technique is:

```bash
ps aux | grep nginx
```

This may work, but you may also see the `grep` command itself.

For example:

```text
root    2001 ... nginx
ann     2500 ... grep nginx
```

Why?

Because `grep nginx` itself contains the text `nginx`.

A better tool for many process-name lookups is `pgrep`.

---

# 35. `pgrep`

`pgrep` searches running processes and prints matching PIDs.

Example:

```bash
pgrep bash
```

Possible output:

```text
6821
7120
```

To show names too:

```bash
pgrep -l bash
```

Possible output:

```text
6821 bash
7120 bash
```

---

# 36. `pgrep -a`

A useful option on systems that support it is:

```bash
pgrep -a python3
```

This can display:

```text
PID command-and-arguments
```

For example:

```text
8120 python3 app.py
8301 python3 worker.py
```

This helps distinguish multiple processes using the same executable.

---

# 37. Match Carefully With `pgrep`

Suppose:

```bash
pgrep python
```

returns several processes.

Do not immediately kill all of them.

First identify:

```text
Which one belongs to my application?

Who owns it?

What arguments is it using?

Who started it?
```

Useful checks include:

```bash
pgrep -a python
```

and:

```bash
ps -p PID -f
```

Never treat a process name alone as sufficient evidence on a shared or production server.

---

# 38. `top`

`top` provides a continuously updating view of processes and system activity.

Run:

```bash
top
```

Unlike `ps`, `top` keeps refreshing.

A typical screen contains:

```text
System uptime and load
Task counts
CPU information
Memory information
Swap information
Process list
```

Press:

```text
q
```

to quit.

---

# 39. Understanding the Top of `top`

The exact format differs by implementation and system, but you may see something similar to:

```text
top - 10:15:02 up 4 days, 2:30, 2 users, load average: 0.42, 0.38, 0.31
Tasks: 180 total, 1 running, 179 sleeping, 0 stopped, 0 zombie
```

This provides a quick system-level picture.

You can ask:

```text
How long has the system been running?

How many tasks exist?

How many are running?

Are there stopped processes?

Are there zombies?
```

---

# 40. CPU Information in `top`

You may see CPU categories such as:

```text
us
sy
ni
id
wa
hi
si
st
```

At beginner level, focus first on:

```text
us = time running user-space work
sy = time running kernel/system work
id = idle CPU time
wa = time associated with waiting for I/O
```

Other fields become more important as your troubleshooting skills grow.

If:

```text
id
```

is very low for a sustained period, the CPUs may be heavily utilized.

If:

```text
wa
```

is unusually high, investigate I/O rather than assuming the CPU itself is the only problem.

---

# 41. Process CPU in `top`

The process list contains:

```text
%CPU
```

You can often sort by CPU interactively by pressing:

```text
P
```

depending on the `top` implementation.

Look for processes consistently near the top.

But do not immediately kill the highest CPU process.

High CPU can be legitimate.

Examples:

```text
Compilation
Video processing
Compression
Database queries
Batch processing
Application load
Security scanning
```

Your task is to determine whether the usage is expected.

---

# 42. Memory Information in `top`

`top` also shows memory statistics and process memory usage.

You may be able to sort by memory using:

```text
M
```

Look at:

```text
%MEM
RES
```

But again, context matters.

A database may intentionally use significant memory for caching.

A Java process may reserve or manage a large heap.

High memory is a symptom to investigate, not automatic proof of a problem.

---

# 43. `htop`

`htop` is an interactive process viewer that many users find easier to navigate than `top`.

Run:

```bash
htop
```

If you receive:

```text
command not found
```

it may not be installed.

On Ubuntu or Debian-based systems, installation may look like:

```bash
sudo apt update
sudo apt install htop
```

On Fedora:

```bash
sudo dnf install htop
```

On RHEL-compatible systems, availability depends on enabled repositories.

Do not blindly install packages on managed or production systems without authorization.

---

# 44. Why `htop` Is Useful

Depending on the version and configuration, `htop` provides:

* Interactive process lists.
* CPU meters.
* Memory meters.
* Process tree views.
* Sorting.
* Searching.
* Filtering.
* Signal sending.

It can make process relationships easier to understand.

However, you should still learn `ps` and `top`.

`htop` may not be installed on minimal servers.

---

# 45. CPU Usage

CPU usage tells you how much processor time is being consumed.

You can inspect CPU-heavy processes with:

```bash
ps aux --sort=-%cpu | head
```

The:

```text
--sort=-%cpu
```

sorts with the highest reported CPU usage first.

The:

```text
-
```

before `%cpu` means descending order.

---

# 46. A Better CPU Investigation View

Try:

```bash
ps -eo pid,ppid,user,stat,%cpu,%mem,etime,cmd --sort=-%cpu | head -n 15
```

This gives:

```text
PID
PPID
user
state
CPU
memory
elapsed time
command
```

This is more useful than only knowing:

```text
python3 uses CPU.
```

You want context.

---

# 47. CPU Percentage Can Be Confusing

On multicore systems, CPU percentages may not always behave exactly as beginners expect.

Depending on the tool and its conventions, a multithreaded process may show more than 100 percent CPU because it can consume significant time across multiple CPU cores.

For example, on a system with multiple logical CPUs:

```text
100% CPU
```

does not necessarily mean:

```text
The entire machine has exactly zero CPU capacity left.
```

Always interpret process CPU alongside system-wide CPU information and CPU count.

You can check processor information with:

```bash
nproc
```

and:

```bash
lscpu
```

---

# 48. Memory Usage

To see memory-heavy processes:

```bash
ps aux --sort=-%mem | head
```

Or:

```bash
ps -eo pid,ppid,user,%cpu,%mem,rss,vsz,stat,cmd --sort=-%mem | head -n 15
```

Remember:

```text
RSS
Resident memory

VSZ
Virtual address-space size
```

---

# 49. Do Not Diagnose Memory From One Process Column Alone

If memory is suspected, combine process information with system-level memory information.

A useful command is:

```bash
free -h
```

You may see:

```text
               total    used    free    shared    buff/cache    available
Mem:            ...
Swap:           ...
```

Do not panic simply because the `free` column is small.

Linux deliberately uses available RAM for caches.

The:

```text
available
```

value is often more useful for a quick understanding of how much memory can be made available for new applications without swapping heavily.

We will study memory in greater depth later.

---

# 50. Starting Background Processes With `&`

Run:

```bash
sleep 600 &
```

Your shell may display:

```text
[1] 9012
```

Your prompt immediately returns.

Check:

```bash
jobs
```

You may see:

```text
[1]+  Running    sleep 600 &
```

The process is still running.

---

# 51. `jobs`

`jobs` shows jobs managed by your current shell.

This distinction is important.

Run:

```bash
jobs
```

It does not show every process on the machine.

It shows jobs known to that shell's job-control system.

Compare:

```text
ps
System process information

jobs
Current shell's jobs
```

---

# 52. Multiple Background Jobs

Run:

```bash
sleep 500 &
sleep 600 &
sleep 700 &
```

Then:

```bash
jobs
```

You may see:

```text
[1]   Running    sleep 500 &
[2]-  Running    sleep 600 &
[3]+  Running    sleep 700 &
```

The exact symbols may vary.

Job numbers are:

```text
%1
%2
%3
```

---

# 53. `Ctrl+C`

When a foreground process is running, pressing:

```text
Ctrl+C
```

usually causes the terminal driver to send an interrupt signal, normally `SIGINT`, to the foreground process group.

For beginner purposes:

```text
Ctrl+C
asks the foreground job to interrupt/terminate.
```

Example:

```bash
sleep 600
```

Press:

```text
Ctrl+C
```

The command normally ends and the prompt returns.

---

# 54. `Ctrl+C` Is Not `kill -9`

This distinction matters.

`Ctrl+C` commonly sends:

```text
SIGINT
```

It is not the same as:

```text
SIGKILL
```

Programs can handle or ignore `SIGINT` in some circumstances.

`SIGKILL` cannot be caught or ignored by normal user-space signal handling.

---

# 55. `Ctrl+Z`

Run:

```bash
sleep 600
```

Then press:

```text
Ctrl+Z
```

You may see:

```text
[1]+  Stopped    sleep 600
```

The process did not finish.

It was suspended.

Check:

```bash
jobs
```

You should see it as stopped.

---

# 56. `Ctrl+C` vs `Ctrl+Z`

Remember:

```text
Ctrl+C
Send interrupt, usually used to stop the foreground job.

Ctrl+Z
Suspend the foreground job.
```

A common beginner mistake is pressing `Ctrl+Z` when they actually want to terminate something.

That leaves a stopped job behind.

---

# 57. `bg`

Suppose you ran:

```bash
sleep 600
```

and pressed:

```text
Ctrl+Z
```

Now:

```bash
jobs
```

shows:

```text
[1]+ Stopped sleep 600
```

Run:

```bash
bg %1
```

The shell resumes job 1 in the background.

Check:

```bash
jobs
```

It should now be running.

---

# 58. `fg`

To bring a background or stopped job into the foreground:

```bash
fg %1
```

Now the job occupies the terminal foreground again.

You can then press:

```text
Ctrl+C
```

if you want to interrupt it.

---

# 59. Job-Control Workflow

A useful practice sequence is:

```bash
sleep 600
```

Then:

```text
Ctrl+Z
```

Then:

```bash
jobs
```

Then:

```bash
bg %1
```

Then:

```bash
jobs
```

Then:

```bash
fg %1
```

Then:

```text
Ctrl+C
```

Conceptually:

```text
Foreground
    |
 Ctrl+Z
    |
    v
Stopped
    |
   bg
    |
    v
Background
    |
   fg
    |
    v
Foreground
    |
 Ctrl+C
    |
    v
Interrupted
```

---

# 60. What Is a Signal?

A signal is a mechanism used to notify a process that an event has occurred or to request some action.

Signals can be used for things such as:

* Requesting termination.
* Interrupting.
* Stopping.
* Continuing.
* Notifying.
* Triggering application-specific behavior.

You can list signals using:

```bash
kill -l
```

The exact list depends on the system.

---

# 61. Common Signals

Some signals you will encounter frequently include:

```text
SIGHUP
SIGINT
SIGQUIT
SIGKILL
SIGTERM
SIGSTOP
SIGCONT
```

You do not need to memorize every signal immediately.

For now, understand:

```text
SIGINT
Interactive interrupt.

SIGTERM
Request graceful termination.

SIGKILL
Forceful termination by the kernel.

SIGSTOP
Stop a process.

SIGCONT
Continue a stopped process.
```

---

# 62. Signal Numbers

Signals also have numbers.

On Linux systems, commonly:

```text
SIGINT  = 2
SIGKILL = 9
SIGTERM = 15
```

This is why you see:

```bash
kill -9 PID
```

But using names is often clearer:

```bash
kill -SIGTERM PID
```

or:

```bash
kill -TERM PID
```

Names communicate intent better than unexplained numbers.

---

# 63. `kill`

Despite its name, `kill` does not always mean:

```text
Immediately destroy this process.
```

`kill` sends a signal to a process.

If you run:

```bash
kill 9012
```

the default signal is normally:

```text
SIGTERM
```

So:

```bash
kill 9012
```

is typically equivalent to:

```bash
kill -TERM 9012
```

---

# 64. SIGTERM

`SIGTERM` means a request to terminate.

The important word is:

```text
request
```

A program can catch `SIGTERM` and perform cleanup.

Depending on the application, cleanup may include:

* Finishing work.
* Closing files.
* Closing network connections.
* Flushing buffers.
* Releasing locks.
* Removing temporary resources.
* Saving state.
* Shutting down workers.

Conceptually:

```text
SIGTERM
   |
   v
Application receives request
   |
   +--> cleanup
   +--> close resources
   +--> finish shutdown
   |
   v
Exit
```

---

# 65. Testing SIGTERM Safely

Start:

```bash
sleep 600 &
```

Find the PID from the output or:

```bash
pgrep -a sleep
```

Suppose the PID is:

```text
9300
```

Verify:

```bash
ps -p 9300 -f
```

Then:

```bash
kill -TERM 9300
```

Verify:

```bash
ps -p 9300
```

If the process terminated, it should no longer appear as a normal running process.

---

# 66. SIGKILL

`SIGKILL` is much more forceful.

You may send it with:

```bash
kill -KILL PID
```

or:

```bash
kill -9 PID
```

A process cannot catch, block, or ignore `SIGKILL`.

The kernel terminates it.

That makes `SIGKILL` powerful, but it also prevents normal user-space cleanup.

---

# 67. Why `kill -9` Should Not Be Your First Response

Suppose an application is processing data.

You immediately run:

```bash
kill -9 PID
```

The process gets no opportunity to perform normal signal-driven cleanup.

Depending on the application, this can leave behind issues such as:

* Incomplete application-level work.
* Temporary files.
* External state requiring recovery.
* Locks or resources that other components must clean up.
* Interrupted requests.

Well-designed filesystems and databases have recovery mechanisms, so this does not mean `SIGKILL` automatically corrupts everything.

The point is:

```text
SIGKILL removes the process's chance to shut down cleanly.
```

Prefer graceful application-specific shutdown mechanisms or `SIGTERM` when appropriate.

---

# 68. A Better Termination Sequence

When a process appears stuck:

```text
1. Identify the correct process.
2. Understand what it is doing.
3. Check whether it is safe to stop.
4. Use the application's normal shutdown mechanism if one exists.
5. Otherwise consider SIGTERM.
6. Wait and verify.
7. Investigate why it did not stop.
8. Use SIGKILL only when necessary.
9. Verify termination.
10. Check application/system health afterward.
```

Do not turn:

```bash
kill -9
```

into a reflex.

---

# 69. Sending SIGTERM

Use:

```bash
kill PID
```

or:

```bash
kill -TERM PID
```

Example:

```bash
kill -TERM 9300
```

Then verify:

```bash
ps -p 9300
```

---

# 70. Sending SIGKILL

If you have determined that forceful termination is necessary:

```bash
kill -KILL PID
```

or:

```bash
kill -9 PID
```

Then verify:

```bash
ps -p PID
```

But if the process remains visible, investigate its state.

It may be a zombie, or it may be blocked in a kernel state such as uninterruptible sleep.

---

# 71. Why `kill -9` May Appear Not to Work

This surprises beginners.

`SIGKILL` cannot be ignored by a normal process.

But a process in:

```text
D
```

state may remain present while blocked in an uninterruptible kernel operation.

The signal can remain pending until the kernel operation reaches a point where termination can proceed.

Therefore:

```text
kill -9 did not remove it instantly
```

does not mean:

```text
Linux forgot what SIGKILL means.
```

You may need to investigate the underlying I/O problem.

---

# 72. `pkill`

`pkill` sends signals to processes based on matching criteria, often process names.

Example:

```bash
pkill sleep
```

This may target multiple matching `sleep` processes.

That is exactly why you must be careful.

Before using:

```bash
pkill something
```

inspect matches:

```bash
pgrep -a something
```

Then determine whether all matches should actually receive the signal.

---

# 73. Safer `pkill` Thinking

Do not do:

```text
The process name looks familiar.
Therefore kill every matching process.
```

Instead:

```text
Search
  |
  v
Inspect matches
  |
  v
Check owner
  |
  v
Check command arguments
  |
  v
Understand role
  |
  v
Choose correct target
  |
  v
Send appropriate signal
  |
  v
Verify
```

---

# 74. Process Ownership and Signals

A regular user cannot arbitrarily signal every process on the machine.

Linux permissions apply.

Usually, you can signal processes you own.

Processes belonging to another user may produce:

```text
Operation not permitted
```

Administrative privileges may be required for certain processes.

Do not respond to this by automatically using `sudo`.

First ask:

```text
Should I be controlling this process?

Who owns it?

Is it a production service?

What will happen if I stop it?
```

---

# 75. Long-Running Commands

Some commands take a long time.

Examples:

```text
Large backups
Data imports
Large file transfers
Build jobs
Compression
Maintenance scripts
Batch processing
Log processing
```

If you start such a command in an SSH session, you need to think about what happens when your terminal or SSH session closes.

Simply adding:

```text
&
```

does not always provide the session independence you may expect.

---

# 76. Why `&` Is Not a Complete Long-Running Solution

Suppose:

```bash
long-command &
```

The process runs in the background relative to your shell.

But it may still have relationships to:

* Your shell.
* Your terminal.
* Its standard streams.
* Session hangup behavior.

When your terminal disconnects, a background process may receive `SIGHUP` or encounter I/O problems depending on how it was launched and what it is doing.

This is where tools such as `nohup`, `tmux`, `screen`, or proper service managers become useful.

---

# 77. `nohup`

`nohup` means:

```text
no hangup
```

It runs a command while arranging for it to ignore `SIGHUP`.

Example:

```bash
nohup sleep 600 &
```

You may see:

```text
nohup: ignoring input and appending output to 'nohup.out'
```

Exact behavior and wording can vary.

---

# 78. `nohup` and Output

A common pattern is:

```bash
nohup command > application.log 2>&1 &
```

Break it down:

```text
nohup
run while ignoring hangup signal

command
the program being executed

> application.log
send stdout to application.log

2>&1
send stderr to stdout's destination

&
run as a background job
```

This combines concepts from Part 7 with process management.

---

# 79. Safe `nohup` Practice

Run:

```bash
nohup sh -c 'for i in 1 2 3 4 5; do date; sleep 2; done' \
> ~/nohup-practice.log 2>&1 &
```

Your shell prints a PID.

Check:

```bash
jobs
```

Then:

```bash
cat ~/nohup-practice.log
```

Wait a few seconds:

```bash
cat ~/nohup-practice.log
```

You should see additional timestamps appear as the process continues.

---

# 80. Monitoring a Long-Running Command

Suppose:

```bash
nohup ./backup.sh > backup.log 2>&1 &
```

Do not simply assume it is working.

Verify the process:

```bash
pgrep -a backup
```

or use the exact PID.

Check:

```bash
ps -p PID -f
```

Monitor output:

```bash
tail -f backup.log
```

This gives you two types of evidence:

```text
Process evidence
Is it still running?

Application evidence
Is it making progress?
```

Both matter.

---

# 81. `nohup` Is Not a Service Manager

`nohup` is useful, but it is not a replacement for proper service management.

For a production service, you often want features such as:

* Automatic startup.
* Restart policies.
* Dependency management.
* Structured service status.
* Logging.
* Controlled shutdown.
* Resource management.

That is where systems such as `systemd` become important.

You will learn service management later.

---

# 82. `tmux` and `screen`

Another approach to long-running interactive work is a terminal multiplexer such as:

```text
tmux
screen
```

These allow sessions to continue while you detach and reconnect later.

They are especially useful for:

* Interactive administrative sessions.
* Long troubleshooting sessions.
* Remote server work.

They are different from `nohup`.

`nohup` protects a command from hangup behavior.

`tmux` and `screen` provide persistent terminal sessions.

---

# 83. Zombie Processes

Return to:

```text
Z
```

A zombie process has already exited.

It is not actively running normal code.

The kernel retains a small record containing information such as its exit status because the parent process is expected to collect it.

Conceptually:

```text
Parent
  |
  +--> Child
         |
         v
       exits
         |
         v
Kernel retains termination record
         |
         v
Parent should reap child
```

If the parent does not do so promptly:

```text
Zombie
```

---

# 84. Why Zombies Exist

When a child exits, its parent may need to know:

```text
Did the child succeed?

What was its exit status?

How did it terminate?
```

The kernel cannot immediately discard all termination information before the parent has an opportunity to retrieve it.

That is why a small process-table entry can remain.

---

# 85. Finding Zombies

You can inspect process states:

```bash
ps -eo pid,ppid,stat,cmd
```

Look for:

```text
Z
```

in `STAT`.

A targeted example:

```bash
ps -eo pid,ppid,stat,cmd | awk '$3 ~ /^Z/'
```

You may also see zombie counts in `top`.

---

# 86. Can You Kill a Zombie?

This is a common interview and beginner question.

A zombie is already terminated.

Sending:

```bash
kill -9 ZOMBIE_PID
```

does not solve the core problem because there is no normally executing child process left to terminate.

The problem is that its parent has not collected the child's exit status.

Investigation should focus on:

```text
Who is the parent?

Why is the parent not reaping children?

Is the parent application malfunctioning?
```

---

# 87. Investigating a Zombie's Parent

Suppose:

```text
PID   PPID   STAT   CMD
9100  8000   Z      [worker] <defunct>
```

The parent PID is:

```text
8000
```

Inspect it:

```bash
ps -p 8000 -f
```

Do not immediately kill the parent.

First understand what it is.

If the parent is an important application, stopping it may affect users.

---

# 88. Are One or Two Zombies an Emergency?

Not necessarily.

A small number of short-lived zombies may disappear when the parent reaps them.

The larger concern is persistent or increasing zombie accumulation.

That may indicate:

* Application bugs.
* Broken child-process handling.
* Parent process problems.

Think:

```text
One observation
Needs context

Hundreds or thousands accumulating
Potential operational problem
```

---

# 89. High CPU Investigation

Now we move from process commands to troubleshooting.

Scenario:

```text
Users report the application is slow.
Monitoring shows CPU utilization is high.
```

Do not begin with:

```bash
kill -9 something
```

Begin with evidence.

---

# 90. High CPU Troubleshooting Workflow

Use:

```text
1. Confirm the symptom.
2. Check system-wide CPU.
3. Identify top CPU consumers.
4. Identify the exact process.
5. Determine the owner and command.
6. Check how long it has been running.
7. Inspect parent/child relationships.
8. Check application logs.
9. Determine whether CPU use is expected.
10. Check whether the process is making progress.
11. Choose a safe corrective action.
12. Verify recovery.
13. Investigate the root cause.
```

---

# 91. Step 1: Confirm CPU Pressure

Use:

```bash
top
```

Look at system-wide CPU activity.

Ask:

```text
Is CPU actually saturated?

Is the problem persistent?

Is CPU mostly user-space work?

Is system/kernel CPU high?

Is I/O wait high instead?

How many CPUs exist?
```

Check:

```bash
nproc
```

A server with:

```text
2 CPUs
```

has very different capacity from one with:

```text
64 CPUs
```

---

# 92. Step 2: Identify CPU-Heavy Processes

Use:

```bash
ps -eo pid,ppid,user,stat,%cpu,%mem,etime,cmd \
--sort=-%cpu | head -n 15
```

Or use:

```bash
top
```

Ask:

```text
Which PID is consuming CPU?

Which user owns it?

What is the full command?

Is it one process or many workers?
```

---

# 93. Step 3: Inspect the Process

Suppose:

```text
PID = 12500
```

Run:

```bash
ps -p 12500 -o pid,ppid,user,stat,%cpu,%mem,etime,cmd
```

Then:

```bash
ps -p 12500 -f
```

If needed:

```bash
pstree -p 12500
```

or inspect its parent:

```bash
ps -p PPID -f
```

Now you know more than:

```text
Python is using CPU.
```

You may discover:

```text
python3 /opt/app/report-generator.py
```

That is actionable context.

---

# 94. Step 4: Check Logs

Suppose the process belongs to an application.

Look for its logs.

For example:

```bash
tail -n 100 /path/to/application.log
```

Or:

```bash
tail -f /path/to/application.log
```

Search:

```bash
grep -i "error" /path/to/application.log
```

You are looking for evidence such as:

```text
Repeated retries
Infinite loops
Request floods
Database failures
Dependency timeouts
Repeated exceptions
Large batch processing
```

Do not assume high CPU itself explains the root cause.

---

# 95. Step 5: Determine Whether the Work Is Expected

Imagine:

```text
PID 12500
CPU 350%
COMMAND java ...
```

Possible interpretations:

```text
Normal scheduled batch job

Unexpected runaway application

Legitimate traffic spike

Bad deployment

Retry storm

Expensive query workload

CPU-intensive maintenance
```

The same metric can mean different things.

Troubleshooting requires context.

---

# 96. Safe High CPU Practice Process

You can create a CPU-intensive process for practice.

A simple example is:

```bash
yes > /dev/null &
```

`yes` repeatedly generates output, and `/dev/null` discards it.

Your shell should return a PID.

Immediately identify it:

```bash
pgrep -a yes
```

Check:

```bash
ps -eo pid,ppid,user,stat,%cpu,%mem,cmd --sort=-%cpu | head
```

You should see `yes` using noticeable CPU.

Stop only the process you started:

```bash
kill -TERM PID
```

Verify:

```bash
ps -p PID
```

Do not leave CPU stress processes running unnecessarily.

---

# 97. High Memory Investigation

Scenario:

```text
An application becomes slow.
The server is under memory pressure.
```

Again:

```text
Observe before acting.
```

---

# 98. High Memory Troubleshooting Workflow

Use:

```text
1. Confirm system memory pressure.
2. Check available memory.
3. Check swap.
4. Identify memory-heavy processes.
5. Identify the exact application.
6. Inspect RSS and %MEM.
7. Check process age.
8. Look for growth over time.
9. Check application logs.
10. Understand whether caching or memory use is expected.
11. Check for OOM evidence if processes disappeared.
12. Choose corrective action.
13. Verify recovery.
14. Investigate why memory grew.
```

---

# 99. Step 1: Check System Memory

Run:

```bash
free -h
```

Then:

```bash
top
```

Ask:

```text
How much memory is available?

Is swap being used?

Is memory pressure persistent?

Is one process dominant?

Are several processes contributing?
```

---

# 100. Step 2: Find Memory-Heavy Processes

Use:

```bash
ps -eo pid,ppid,user,%cpu,%mem,rss,vsz,stat,etime,cmd \
--sort=-%mem | head -n 15
```

Then inspect the suspicious PID:

```bash
ps -p PID -o pid,ppid,user,%cpu,%mem,rss,vsz,stat,etime,cmd
```

---

# 101. Step 3: Do Not Assume "High Memory" Means Leak

A process using a lot of memory may be expected.

Examples:

```text
Database cache

Java heap

Search engine

In-memory cache

Large data-processing job

Build process
```

A memory leak is usually more about uncontrolled or unexpected growth over time than one isolated large number.

You need repeated observations and application context.

---

# 102. Out-of-Memory Events

When Linux cannot satisfy memory requirements under severe pressure, the kernel may invoke the Out-Of-Memory killer.

You may find evidence in kernel logs.

Depending on the distribution and permissions, useful commands can include:

```bash
dmesg | grep -i -E 'oom|out of memory|killed process'
```

or:

```bash
journalctl -k | grep -i -E 'oom|out of memory|killed process'
```

You will learn `journalctl` in greater depth in the logging and service-management sections.

The key lesson is:

```text
If a process mysteriously disappeared during memory pressure, investigate whether the kernel killed it.
```

---

# 103. Stuck Process Troubleshooting

Scenario:

```text
A process is still present, but the application appears frozen.
```

Do not immediately conclude:

```text
The process must be killed.
```

First determine what "stuck" means.

Possibilities include:

```text
Waiting for disk I/O

Waiting for network I/O

Waiting for a lock

Waiting for another process

Sleeping normally

Deadlocked inside the application

Blocked on external dependency

CPU loop

Uninterruptible kernel wait
```

---

# 104. Step 1: Identify the Exact Process

Use:

```bash
pgrep -a application-name
```

or:

```bash
ps aux | grep application-name
```

Then inspect:

```bash
ps -p PID -o pid,ppid,user,stat,%cpu,%mem,etime,cmd
```

Do not target a process until you know exactly what it is.

---

# 105. Step 2: Check Process State

Look at:

```text
STAT
```

Possible clues:

```text
R
actively running/runnable

S
sleeping/waiting

D
uninterruptible sleep

T
stopped

Z
zombie
```

The state changes your troubleshooting path.

---

# 106. If the Process Is `T`

A `T` state may mean the process was stopped.

Check shell jobs if relevant:

```bash
jobs
```

If it belongs to your current shell and was accidentally suspended:

```bash
bg %JOB
```

or:

```bash
fg %JOB
```

can resume it.

A process that appears "frozen" may simply have been suspended with `Ctrl+Z`.

---

# 107. If the Process Is `Z`

Do not:

```bash
kill -9 ZOMBIE_PID
```

and expect that to solve the issue.

Inspect:

```bash
ps -p ZOMBIE_PID -o pid,ppid,stat,cmd
```

Then inspect its parent:

```bash
ps -p PPID -f
```

The parent-child handling is the real area of investigation.

---

# 108. If the Process Is `D`

A persistent `D` state may point toward I/O or kernel-level waiting.

Investigate broader system health.

Depending on the environment, questions include:

```text
Is a disk failing?

Is a mounted filesystem unavailable?

Is network storage unreachable?

Is the process waiting on a blocked device?

Are many processes in D state?
```

`kill -9` may not make such a process disappear immediately.

Find the underlying blocked operation.

---

# 109. If the Process Is `R` With High CPU

If it remains runnable and consumes high CPU:

```text
Check whether workload is expected.

Check logs.

Check whether requests increased.

Check whether code is retrying.

Check whether multiple workers are affected.

Check recent deployments.

Check whether it is making progress.
```

Do not diagnose solely from the letter `R`.

---

# 110. If the Process Is `S`

Sleeping is often normal.

For example:

```bash
sleep 1000
```

will normally spend most of its time sleeping.

Server processes also frequently wait for:

```text
Connections

Requests

Timers

Events
```

So:

```text
S
```

does not automatically mean stuck.

---

# 111. Check How Long the Process Has Been Running

Use:

```bash
ps -p PID -o pid,lstart,etime,stat,cmd
```

`lstart` shows a start timestamp.

`etime` shows elapsed time.

This helps answer:

```text
Did this process start after the latest deployment?

Has it been running for weeks?

Did it restart recently?

Is this supposed to be a short job that has been running for hours?
```

---

# 112. Check the Parent

Use:

```bash
ps -p PID -o pid,ppid,cmd
```

Suppose:

```text
PID   PPID
8200  8100
```

Inspect:

```bash
ps -p 8100 -f
```

The parent can provide crucial context.

You may discover the suspicious process is a worker created by:

```text
Web server

Application master

Shell script

Job runner

Service manager
```

Stopping one child may accomplish nothing if the parent immediately recreates it.

---

# 113. Process Respawning

Suppose you kill:

```text
PID 8200
```

Then a few seconds later another similar process appears:

```text
PID 8350
```

Do not assume:

```text
kill failed.
```

A parent process or service manager may have restarted it.

Investigate:

```text
Who started it?

Is systemd managing it?

Is a supervisor restarting it?

Is Kubernetes recreating it?

Is a parent worker manager respawning it?
```

This becomes especially important in production environments.

---

# 114. Check Whether the PID Changed

Do not rely only on process names.

Suppose:

```bash
pgrep -a myapp
```

before termination:

```text
8200 /opt/myapp
```

After:

```text
8350 /opt/myapp
```

The original process did terminate.

A new process replaced it.

This is different from:

```text
PID 8200 ignored my signal.
```

---

# 115. PIDs Can Be Reused

Linux can eventually reuse process IDs after processes exit.

Therefore, in sensitive operations, do not wait a long time after identifying a PID and blindly assume it still refers to the same process.

Re-check:

```bash
ps -p PID -f
```

before sending destructive signals.

This habit matters on busy systems.

---

# 116. Permission Denied When Sending a Signal

Suppose:

```bash
kill PID
```

returns:

```text
Operation not permitted
```

Investigate:

```bash
whoami
ps -p PID -o user,pid,ppid,cmd
```

The process may belong to another user.

Do not automatically escalate.

Confirm that you are authorized to control it.

---

# 117. "No Such Process"

You may see:

```text
kill: (...) - No such process
```

Possible reasons:

```text
The process already exited.

You typed the wrong PID.

The PID changed.

You are using stale information.
```

Verify:

```bash
ps -p PID
```

and search again:

```bash
pgrep -a process-name
```

---

# 118. Why `ps aux | grep` Sometimes Confuses Beginners

Run:

```bash
ps aux | grep sleep
```

You may see:

```text
ann ... sleep 600
ann ... grep sleep
```

The second result is your search command.

Use:

```bash
pgrep -a sleep
```

when appropriate.

Or a common shell pattern is:

```bash
ps aux | grep '[s]leep'
```

because the `grep` command line no longer contains the exact contiguous search string `sleep`.

But `pgrep` is usually clearer for beginners.

---

# 119. Process Troubleshooting Reasoning

Use this model:

```text
Observe
   |
   v
Confirm symptom
   |
   v
Identify process
   |
   v
Inspect PID and PPID
   |
   v
Check owner
   |
   v
Check state
   |
   v
Check CPU and memory
   |
   v
Check runtime
   |
   v
Inspect logs
   |
   v
Understand parent/children
   |
   v
Form hypothesis
   |
   v
Test safely
   |
   v
Take corrective action
   |
   v
Verify
   |
   v
Find root cause
```

---

# 120. Why "Restart It" Is Not Troubleshooting

Restarting a process can restore service.

But restarting alone may destroy evidence.

Suppose an application hangs every day at 14:00.

If you immediately restart it every time, you may never discover:

```text
Memory leak

Deadlock

Database timeout

Storage issue

Bad scheduled job

Dependency failure

Traffic spike

Recent deployment problem
```

Sometimes restoring service quickly is necessary.

But when possible, capture evidence first.

At minimum:

```bash
date
ps -p PID -o pid,ppid,user,stat,%cpu,%mem,etime,cmd
top
free -h
```

and relevant logs can preserve valuable context.

---

# 121. Process Troubleshooting Lab

We will now build a safe process-management lab.

Create:

```bash
mkdir -p ~/process-lab/{logs,output,scripts}
cd ~/process-lab
```

Verify:

```bash
pwd
```

---

# 122. Lab 1: Identify Your Shell

Run:

```bash
echo $$
```

Then:

```bash
ps -p $$
```

Then:

```bash
ps -p $$ -o pid,ppid,user,stat,etime,cmd
```

Identify:

* Your shell PID.
* Its PPID.
* User.
* State.
* Command.

---

# 123. Lab 2: Start a Foreground Process

Run:

```bash
sleep 30
```

Observe:

```text
Does the shell prompt return immediately?
```

Wait for completion or use:

```text
Ctrl+C
```

Explain why this was a foreground process.

---

# 124. Lab 3: Start a Background Process

Run:

```bash
sleep 300 &
```

Record the output.

You may see:

```text
[1] 10500
```

Identify:

```text
Job ID = 1
PID = 10500
```

Then:

```bash
jobs
```

---

# 125. Lab 4: Inspect Parent and Child

Use the PID from Lab 3:

```bash
ps -p 10500 -o pid,ppid,user,stat,cmd
```

Replace `10500` with your real PID.

Compare:

```bash
echo $$
```

Is your shell the parent?

---

# 126. Lab 5: Multiple Background Jobs

Run:

```bash
sleep 400 &
sleep 500 &
sleep 600 &
```

Then:

```bash
jobs
```

Identify:

* Job numbers.
* Running state.
* PIDs.

Now compare with:

```bash
pgrep -a sleep
```

Explain the difference between a job number and PID.

---

# 127. Lab 6: Suspend a Foreground Process

Run:

```bash
sleep 600
```

Press:

```text
Ctrl+Z
```

Then:

```bash
jobs
```

The job should be stopped.

Use:

```bash
ps -o pid,ppid,stat,cmd -C sleep
```

if supported, or inspect its known PID.

Look for:

```text
T
```

---

# 128. Lab 7: Resume in Background

Using the stopped job:

```bash
bg %JOB_NUMBER
```

For example:

```bash
bg %1
```

Check:

```bash
jobs
```

The job should now be running in the background.

---

# 129. Lab 8: Bring It Back to Foreground

Run:

```bash
fg %JOB_NUMBER
```

Then press:

```text
Ctrl+C
```

Check:

```bash
jobs
```

Explain the complete state transition.

---

# 130. Lab 9: Find a Process With `pgrep`

Start:

```bash
sleep 700 &
```

Then:

```bash
pgrep -l sleep
```

Then:

```bash
pgrep -a sleep
```

Explain why `-a` may be more useful when several similar processes exist.

---

# 131. Lab 10: Inspect With `ps aux`

Run:

```bash
ps aux | head
```

Identify:

```text
USER
PID
%CPU
%MEM
STAT
COMMAND
```

Then locate one of your `sleep` processes.

Do not worry if its CPU is nearly zero.

That is expected.

---

# 132. Lab 11: Inspect With `ps -ef`

Run:

```bash
ps -ef | head
```

Identify:

```text
UID
PID
PPID
CMD
```

Compare it with:

```bash
ps aux
```

Explain one situation where PPID is useful.

---

# 133. Lab 12: Use `top`

Run:

```bash
top
```

Find:

* Task count.
* CPU section.
* Memory section.
* Process list.
* `%CPU`.
* `%MEM`.
* `PID`.
* `COMMAND`.

Press:

```text
q
```

to exit.

---

# 134. Lab 13: Generate High CPU Safely

Start:

```bash
yes > /dev/null &
```

Record the PID.

Now:

```bash
ps -eo pid,ppid,user,stat,%cpu,%mem,cmd \
--sort=-%cpu | head
```

Then:

```bash
top
```

Find your `yes` process.

Exit `top`.

Stop the exact process:

```bash
kill -TERM PID
```

Verify:

```bash
ps -p PID
```

Do not leave it running.

---

# 135. Lab 14: Graceful Termination

Start:

```bash
sleep 900 &
```

Record its PID.

Verify:

```bash
ps -p PID -f
```

Then:

```bash
kill -TERM PID
```

Verify again:

```bash
ps -p PID
```

Explain why `SIGTERM` is generally preferable to immediately using `SIGKILL`.

---

# 136. Lab 15: Inspect Available Signals

Run:

```bash
kill -l
```

Find:

```text
SIGINT
SIGTERM
SIGKILL
SIGSTOP
SIGCONT
```

Remember at minimum:

```text
2  SIGINT
9  SIGKILL
15 SIGTERM
```

on Linux.

---

# 137. Lab 16: Stop and Continue With Signals

Start:

```bash
sleep 900 &
```

Record the PID.

Send:

```bash
kill -STOP PID
```

Inspect:

```bash
ps -p PID -o pid,stat,cmd
```

You should see a stopped state.

Continue it:

```bash
kill -CONT PID
```

Inspect again.

Then terminate:

```bash
kill -TERM PID
```

---

# 138. Lab 17: `pkill` Carefully

Start several practice processes:

```bash
sleep 901 &
sleep 902 &
sleep 903 &
```

Inspect:

```bash
pgrep -a sleep
```

Do not run `pkill sleep` if you have unrelated `sleep` processes you want to preserve.

The purpose of this exercise is to understand why name-based termination can affect multiple processes.

Clean up your own known PIDs individually:

```bash
kill -TERM PID1 PID2 PID3
```

---

# 139. Lab 18: `nohup`

Run:

```bash
nohup sh -c 'for i in 1 2 3 4 5; do echo "$(date) process-lab"; sleep 2; done' \
> ~/process-lab/logs/nohup.log 2>&1 &
```

Record the PID.

Monitor:

```bash
tail -f ~/process-lab/logs/nohup.log
```

After several entries, press:

```text
Ctrl+C
```

This stops `tail`, not necessarily the background command.

Check:

```bash
cat ~/process-lab/logs/nohup.log
```

---

# 140. Lab 19: Process Snapshot Report

Create a process report:

```bash
{
    echo "Process Report"
    echo "Generated: $(date)"
    echo "Hostname: $(hostname)"
    echo
    echo "Top CPU Processes"
    ps -eo pid,ppid,user,stat,%cpu,%mem,etime,cmd \
    --sort=-%cpu | head -n 10
    echo
    echo "Top Memory Processes"
    ps -eo pid,ppid,user,stat,%cpu,%mem,rss,etime,cmd \
    --sort=-%mem | head -n 10
} > ~/process-lab/output/process-report.txt
```

Inspect:

```bash
less ~/process-lab/output/process-report.txt
```

This combines concepts from Parts 6, 7, and 8.

---

# 141. Lab 20: High CPU Troubleshooting Scenario

Start:

```bash
yes > /dev/null &
```

Pretend you did not already know which process was responsible.

Investigate:

```bash
top
```

Then:

```bash
ps -eo pid,ppid,user,stat,%cpu,%mem,etime,cmd \
--sort=-%cpu | head -n 10
```

Identify the process.

Inspect it:

```bash
ps -p PID -o pid,ppid,user,stat,%cpu,%mem,etime,cmd
```

Check its parent:

```bash
ps -p PPID -f
```

Then stop only your test process:

```bash
kill -TERM PID
```

Verify:

```bash
ps -p PID
```

Finally recheck:

```bash
top
```

The correct workflow was:

```text
Observe
-> identify
-> inspect
-> understand
-> act
-> verify
```

not:

```text
CPU high
-> kill random process
```

---

# 142. Lab 21: Stopped Process Troubleshooting

Start:

```bash
sleep 1000
```

Press:

```text
Ctrl+Z
```

Pretend you did not remember doing that.

Investigate:

```bash
jobs
```

Then inspect the process state.

You should find:

```text
T
```

Instead of killing it, resume it:

```bash
bg %JOB_NUMBER
```

Then:

```bash
jobs
```

This demonstrates why checking process state matters.

---

# 143. Lab 22: Process Disappeared

Start:

```bash
sleep 5 &
```

Record its PID.

Wait more than five seconds.

Then:

```bash
ps -p PID
```

It should be gone.

Now:

```bash
kill PID
```

may report:

```text
No such process
```

Explain why this does not represent a Linux failure.

The process already completed.

---

# 144. Lab 23: Background Job Completion

Run:

```bash
sleep 3 &
```

Wait.

Then press Enter or run:

```bash
jobs
```

You may see completion information.

This demonstrates that:

```text
background
```

does not mean:

```text
permanent.
```

The process still exits when its work completes.

---

# 145. Lab 24: CPU and Memory Comparison

Run:

```bash
ps -eo pid,user,%cpu,%mem,rss,vsz,stat,cmd \
--sort=-%cpu | head -n 10
```

Then:

```bash
ps -eo pid,user,%cpu,%mem,rss,vsz,stat,cmd \
--sort=-%mem | head -n 10
```

Compare.

Ask:

```text
Is the highest CPU process also the highest memory process?
```

Often it is not.

CPU pressure and memory pressure are different problems.

---

# 146. Lab 25: Full Process Troubleshooting Simulation

Scenario:

Users report:

```text
"The Linux server is slow."
```

You are not allowed to restart the server immediately.

Perform the investigation in this order.

### Step 1

Record:

```bash
date
hostname
uptime
```

### Step 2

Check:

```bash
top
```

### Step 3

Inspect CPU-heavy processes:

```bash
ps -eo pid,ppid,user,stat,%cpu,%mem,etime,cmd \
--sort=-%cpu | head -n 15
```

### Step 4

Inspect memory-heavy processes:

```bash
ps -eo pid,ppid,user,stat,%cpu,%mem,rss,etime,cmd \
--sort=-%mem | head -n 15
```

### Step 5

Check memory:

```bash
free -h
```

### Step 6

Inspect suspicious PID:

```bash
ps -p PID -o pid,ppid,user,stat,%cpu,%mem,rss,vsz,lstart,etime,cmd
```

### Step 7

Inspect its parent:

```bash
ps -p PPID -f
```

### Step 8

Search for similar processes:

```bash
pgrep -a PROCESS_NAME
```

### Step 9

Check relevant application logs.

### Step 10

Determine whether the behavior is expected.

### Step 11

If termination is justified, use the least disruptive appropriate mechanism.

### Step 12

Verify the process state afterward.

### Step 13

Verify CPU and memory again.

### Step 14

Verify application health.

### Step 15

Document the evidence and root cause.

---

# 147. Common Process Troubleshooting Mistakes

Avoid these habits.

### Mistake 1: Killing the first high-CPU process

High CPU may be legitimate.

### Mistake 2: Starting with `kill -9`

You remove the process's opportunity for normal signal-driven cleanup.

### Mistake 3: Confusing a job ID with a PID

```text
%1
```

and:

```text
10500
```

are not the same type of identifier.

### Mistake 4: Assuming `S` means stuck

Sleeping processes are extremely common.

### Mistake 5: Trying to kill zombies

The zombie has already exited.

Investigate its parent.

### Mistake 6: Assuming a process ignored `kill -9`

It may be in uninterruptible sleep, or you may be looking at a replacement process.

### Mistake 7: Ignoring PPID

The parent can explain why workers exist or respawn.

### Mistake 8: Treating `&` as full session persistence

Backgrounding and surviving terminal/session closure are different concepts.

### Mistake 9: Looking only at `%MEM`

Memory diagnosis requires system context.

### Mistake 10: Restarting before collecting evidence

You may remove the evidence needed to understand the root cause.

---

# 148. Practical Production Scenario

Imagine this environment:

```text
Users
  |
  v
Load Balancer
  |
  v
Linux Application Server
  |
  +--> Web Process
  |
  +--> Worker Process
  |
  +--> Monitoring Agent
  |
  +--> Logging Agent
```

Monitoring alerts:

```text
CPU: 97%
```

A beginner might do:

```bash
top
```

see:

```text
python3  390%
```

and immediately run:

```bash
kill -9 PID
```

A better engineer asks:

```text
Which Python process?

What application does it belong to?

What arguments started it?

Who owns it?

Who is its parent?

Is 390% possible because multiple CPU cores are involved?

Was there a traffic spike?

Was there a deployment?

Are logs showing retries?

Is the process doing legitimate batch work?

Will a service manager restart it?

Will killing it interrupt requests?

Can it shut down gracefully?
```

Then:

```text
Observe
-> collect evidence
-> form hypothesis
-> choose action
-> verify
```

That reasoning is more important than memorizing `kill`.

---

# 149. Command Reference

## Basic process listing

```bash
ps
```

## Full process views

```bash
ps aux
ps -ef
```

## Custom process view

```bash
ps -eo pid,ppid,user,stat,%cpu,%mem,etime,cmd
```

## Inspect one process

```bash
ps -p PID -f
```

```bash
ps -p PID -o pid,ppid,user,stat,%cpu,%mem,etime,cmd
```

## Process tree

```bash
pstree
pstree -p
```

## Find process

```bash
pgrep process-name
pgrep -l process-name
pgrep -a process-name
```

## Live monitoring

```bash
top
htop
```

## CPU-heavy processes

```bash
ps -eo pid,ppid,user,stat,%cpu,%mem,etime,cmd \
--sort=-%cpu | head
```

## Memory-heavy processes

```bash
ps -eo pid,ppid,user,%cpu,%mem,rss,vsz,stat,cmd \
--sort=-%mem | head
```

## Background process

```bash
command &
```

## View shell jobs

```bash
jobs
```

## Resume job in background

```bash
bg %1
```

## Bring job to foreground

```bash
fg %1
```

## Default termination signal

```bash
kill PID
```

## Explicit SIGTERM

```bash
kill -TERM PID
```

## SIGKILL

```bash
kill -KILL PID
```

or:

```bash
kill -9 PID
```

## Stop

```bash
kill -STOP PID
```

## Continue

```bash
kill -CONT PID
```

## List signals

```bash
kill -l
```

## Find processes before `pkill`

```bash
pgrep -a process-name
```

## Name-based signal

```bash
pkill process-name
```

## Long-running command

```bash
nohup command > command.log 2>&1 &
```

## System memory

```bash
free -h
```

## CPU count

```bash
nproc
```

---

# 150. What You Should Now Understand

Before moving to Part 9, you should confidently understand:

* The difference between a program and a process.
* Why one program can have multiple running processes.
* How processes start at a beginner level.
* What a PID is.
* Why PIDs are important.
* What PID 1 represents.
* What a PPID is.
* Parent and child processes.
* Why process trees matter.
* Foreground processes.
* Background processes.
* What `&` does.
* The difference between processes and shell jobs.
* The difference between job IDs and PIDs.
* Common process states.
* `R`, `S`, `D`, `T`, and `Z`.
* Why sleeping processes are not automatically unhealthy.
* Why persistent `D` state deserves investigation.
* What a zombie is.
* Why a zombie cannot simply be fixed with `kill -9`.
* How to use `ps`.
* How to use `ps aux`.
* How to use `ps -ef`.
* How to create custom `ps` views.
* What USER, PID, PPID, STAT, `%CPU`, `%MEM`, RSS, VSZ, and COMMAND mean.
* Why `ps` is a snapshot.
* How `top` differs from `ps`.
* What `htop` is.
* How to use `pgrep`.
* Why `pgrep -a` is useful.
* Why process-name matching requires caution.
* Basic CPU usage interpretation.
* Why process CPU can exceed 100 percent on multicore systems.
* Basic memory usage interpretation.
* Why VSZ and RSS are different.
* Why low "free" memory alone does not prove a memory problem.
* How to start background processes.
* How `jobs` works.
* What `Ctrl+C` normally does.
* What `Ctrl+Z` does.
* How `bg` works.
* How `fg` works.
* What Linux signals are.
* What `SIGINT` is.
* What `SIGTERM` is.
* What `SIGKILL` is.
* What `SIGSTOP` and `SIGCONT` do.
* Why `kill` actually sends signals.
* Why `kill PID` normally sends `SIGTERM`.
* Why `kill -9` should not be your first response.
* Why `SIGKILL` may not immediately remove a process in `D` state.
* How `pkill` works.
* Why `pgrep` should often be used before `pkill`.
* Why process ownership affects signal permissions.
* What `nohup` does.
* Why `&` and `nohup` solve different problems.
* Why `nohup` is not a production service manager.
* What `tmux` and `screen` are conceptually.
* How zombie processes occur.
* How to investigate a zombie's parent.
* How to investigate high CPU.
* How to investigate high memory.
* Why high memory does not automatically mean memory leak.
* What an OOM event is at a beginner level.
* How to investigate a stuck process.
* Why process state changes your troubleshooting strategy.
* Why parent processes matter when children respawn.
* Why a newly appearing PID can mean the process was restarted.
* Why PIDs can eventually be reused.
* Why evidence should be collected before restarting when practical.
* Why process troubleshooting is about reasoning, not memorizing commands.

---

# 151. Hands-On Practice

Without simply copying the completed labs, perform these tasks:

1. Find your shell PID.

2. Find your shell's PPID.

3. Display your shell's user, state, and elapsed time.

4. Start a foreground `sleep` process.

5. Interrupt it using `Ctrl+C`.

6. Start another foreground process.

7. Suspend it using `Ctrl+Z`.

8. View it with `jobs`.

9. Resume it using `bg`.

10. Bring it back using `fg`.

11. Terminate it safely.

12. Start three background processes.

13. Identify their job IDs.

14. Identify their PIDs.

15. Explain why those IDs are different.

16. Inspect one process using `ps`.

17. Inspect its PPID.

18. Identify its parent.

19. Use `ps aux`.

20. Use `ps -ef`.

21. Explain a difference between the two views.

22. Use a custom `ps -eo` format.

23. Use `pgrep`.

24. Use `pgrep -a`.

25. Use `top`.

26. Identify CPU and memory fields.

27. Install or use `htop` only if appropriate for your environment.

28. Create a safe CPU-intensive process.

29. Find it without relying on the PID printed when you started it.

30. Confirm its CPU usage.

31. Inspect its parent.

32. Stop it using `SIGTERM`.

33. Verify that it disappeared.

34. Start a `sleep` process.

35. Send `SIGSTOP`.

36. Verify its state.

37. Send `SIGCONT`.

38. Verify it resumed.

39. Send `SIGTERM`.

40. List available signals.

41. Explain `SIGTERM`.

42. Explain `SIGKILL`.

43. Explain why `kill -9` should not be the default.

44. Find your highest CPU processes.

45. Find your highest memory processes.

46. Compare the two lists.

47. Run `free -h`.

48. Explain why `available` memory matters.

49. Run `nproc`.

50. Explain why CPU count matters when interpreting `%CPU`.

51. Run a command with `nohup`.

52. Redirect its stdout.

53. Redirect its stderr.

54. Run it in the background.

55. Monitor its log with `tail -f`.

56. Stop only `tail` with `Ctrl+C`.

57. Verify whether the background process still exists.

58. Find any zombie processes on your practice machine.

59. If none exist, explain how you would identify one.

60. Explain why killing a zombie itself does not solve the parent problem.

61. Create a process report.

62. Include PID.

63. Include PPID.

64. Include user.

65. Include state.

66. Include CPU.

67. Include memory.

68. Include elapsed time.

69. Include command.

70. Save the report using redirection.

71. Simulate high CPU.

72. Diagnose it using `top`.

73. Diagnose it using `ps`.

74. Stop the exact offending practice process.

75. Verify CPU recovery.

76. Create an accidentally stopped process.

77. Diagnose the `T` state.

78. Resume it instead of killing it.

79. Explain what you would investigate for persistent `D` state.

80. Write your own process-troubleshooting workflow from memory.

---

# 152. Beginner Knowledge Check

1. What is a program?

2. What is a process?

3. Can one program create multiple processes?

4. What does PID mean?

5. Why does a process need a PID?

6. What is PID 1?

7. What does PPID mean?

8. What is a parent process?

9. What is a child process?

10. Why are process trees useful?

11. What is a foreground process?

12. What is a background process?

13. What does `&` do?

14. What is a shell job?

15. Is a job ID the same as a PID?

16. What does process state `R` mean?

17. What does `S` mean?

18. Does `S` automatically mean something is wrong?

19. What does `D` mean?

20. Why can persistent `D` state be important?

21. What does `T` mean?

22. What does `Z` mean?

23. What is a zombie process?

24. Why does a zombie remain in the process table?

25. Can `kill -9` fix the zombie itself?

26. What does `ps` show?

27. What does `ps aux` provide?

28. What does `ps -ef` provide?

29. What does `%CPU` mean at a beginner level?

30. What does `%MEM` mean?

31. What is RSS?

32. What is VSZ?

33. Why is VSZ not the same as physical RAM usage?

34. Is `ps` continuously updated?

35. What is `top`?

36. What is `htop`?

37. What does `pgrep` do?

38. What does `pgrep -a` add?

39. Why should you inspect process matches before terminating them?

40. What does `jobs` show?

41. Does `jobs` show every process on the server?

42. What does `Ctrl+C` usually send?

43. What does `Ctrl+Z` do?

44. What does `bg` do?

45. What does `fg` do?

46. What is a signal?

47. What is `SIGINT`?

48. What is `SIGTERM`?

49. What is `SIGKILL`?

50. What is `SIGSTOP`?

51. What is `SIGCONT`?

52. What does `kill PID` normally send?

53. Why is `SIGTERM` generally preferable to `SIGKILL`?

54. Why should `kill -9` usually be a last resort?

55. Can a normal process catch `SIGKILL`?

56. Why might a process in `D` state remain visible after `SIGKILL`?

57. What does `pkill` do?

58. Why can `pkill` be dangerous?

59. Why can a regular user receive "Operation not permitted" from `kill`?

60. What does `nohup` do?

61. What does `&` do that is different from `nohup`?

62. Is `nohup` a service manager?

63. What are `tmux` and `screen` used for conceptually?

64. Why might a background process disappear after an SSH session ends?

65. What should you check first when CPU is high?

66. Should you immediately kill the highest CPU process?

67. Why can a process show more than 100 percent CPU?

68. What command can show the number of available processing units?

69. What command gives a basic system memory summary?

70. Does low `free` memory alone prove a memory problem?

71. What is a memory leak at a basic level?

72. What is the OOM killer?

73. Where might you look for evidence of an OOM event?

74. What should you check when a process appears stuck?

75. Why does process state matter?

76. What should you investigate for a `T` process?

77. What should you investigate for a zombie?

78. What should you investigate for persistent `D` state?

79. Why might a process return after you kill it?

80. Why should you recheck the PID?

81. Can PIDs eventually be reused?

82. Why should you collect evidence before restarting a process?

83. What is the difference between restoring service and finding root cause?

84. Why should application logs be checked during process troubleshooting?

85. What is the correct general troubleshooting sequence?

---

# 153. Beginner Knowledge Check Answers

1. Executable instructions stored on the system.

2. A running instance of a program with kernel-managed state and resources.

3. Yes.

4. Process ID.

5. It uniquely identifies a currently existing process within the relevant PID namespace at that time.

6. The first process in a Linux process namespace, commonly `systemd` on modern host systems, though other environments can use something else.

7. Parent Process ID.

8. A process associated as the creator or parent of another process.

9. A process created under another process.

10. They reveal relationships between processes.

11. A job currently controlling the terminal foreground.

12. A job running without occupying the shell's foreground.

13. Requests that the shell run the command as a background job.

14. A unit managed by the shell's job-control mechanism.

15. No.

16. Running or runnable.

17. Interruptible sleep.

18. No. Sleeping is extremely common.

19. Uninterruptible sleep.

20. It can indicate blocked kernel-level I/O or another operation requiring deeper investigation.

21. Stopped or traced.

22. Zombie.

23. An exited process whose termination status has not yet been collected by its parent.

24. So the parent can retrieve termination information.

25. No. The child has already exited.

26. Process information at a point in time.

27. A broad process listing including CPU, memory, state, user, and command information.

28. A full-format process listing including PID and PPID.

29. An indication of CPU use associated with the process according to the tool's measurement.

30. A percentage-based memory measure reported for the process.

31. Resident Set Size, roughly memory pages resident in RAM for that process, though shared memory complicates interpretation.

32. Virtual memory size.

33. Virtual address space can contain mappings and allocations that are not all physically resident in RAM.

34. No. It is a snapshot.

35. A continuously updating process and system activity viewer.

36. An interactive process viewer with a more navigable interface on systems where it is installed.

37. Searches processes and returns matching PIDs.

38. It can show command-line information with the PID.

39. A name can match multiple unrelated or important processes.

40. Jobs belonging to the current shell.

41. No.

42. Normally `SIGINT`.

43. Suspends the foreground job, normally using a stop signal generated by the terminal.

44. Resumes a stopped shell job in the background.

45. Moves a job into the foreground.

46. A mechanism for notifying or controlling processes.

47. An interrupt signal commonly associated with `Ctrl+C`.

48. A request for graceful termination.

49. A forceful kernel-enforced termination signal.

50. A signal that stops process execution.

51. A signal that resumes a stopped process.

52. `SIGTERM` by default.

53. The application may handle it and perform cleanup before exiting.

54. `SIGKILL` prevents normal user-space cleanup.

55. No.

56. It may be blocked in an uninterruptible kernel operation and cannot complete termination until the kernel reaches an appropriate point.

57. Sends a signal to processes selected by matching criteria such as name.

58. It can match multiple processes.

59. The user may not have permission to signal that process.

60. Runs a command while arranging for it to ignore `SIGHUP`.

61. `&` backgrounds the job. `nohup` changes hangup-signal behavior.

62. No.

63. Persistent terminal-session management.

64. It may still depend on the shell, terminal, standard streams, or session hangup behavior.

65. Confirm the system-wide CPU symptom and identify the processes consuming CPU.

66. No. Determine whether the workload is expected and understand the process first.

67. A multithreaded process may consume CPU time across multiple cores, depending on the tool's reporting convention.

68.

```bash
nproc
```

69.

```bash
free -h
```

70. No. Linux uses RAM for caching and other useful purposes.

71. Unintended memory growth where an application continues retaining memory it no longer needs.

72. A kernel mechanism that can terminate processes under severe memory exhaustion.

73. Kernel logs, for example through `dmesg` or `journalctl -k`, depending on permissions and system configuration.

74. PID, owner, process state, CPU, memory, elapsed time, parent, logs, and what resource or dependency it may be waiting for.

75. Different states suggest different causes and actions.

76. Whether it was intentionally or accidentally suspended and whether it should be resumed.

77. Its parent process and why that parent is not reaping exited children.

78. Storage, filesystem, device, network-storage, and other kernel-level I/O conditions.

79. A parent, supervisor, service manager, container orchestrator, or other controller may restart it.

80. The original process may have exited, been replaced, or the PID information may be stale.

81. Yes.

82. Restarting can remove valuable diagnostic evidence.

83. Restoring service makes the system usable again. Root-cause analysis determines why the failure occurred and how to prevent recurrence.

84. Logs may reveal retries, failures, dependency problems, crashes, or other behavior explaining the process symptoms.

85.

```text
Observe
-> confirm
-> identify
-> inspect
-> gather evidence
-> form a hypothesis
-> test safely
-> correct the cause
-> verify
-> prevent recurrence
```

# 154. Final Mental Model

A Linux system is constantly managing processes.

Think of it like this:

```text
                     Linux Kernel
                          |
          +---------------+---------------+
          |               |               |
          v               v               v
       Process          Process         Process
       PID 1200         PID 1300        PID 1400
          |               |               |
          |               |               |
          v               v               v
       CPU/RAM         CPU/RAM         CPU/RAM
          |
          v
     Process state
          |
   +------+------+------+------+------+
   |      |      |      |      |
   R      S      D      T      Z
```

Your job as an engineer is not simply to know:

```bash
ps
top
kill
```

Your job is to answer:

```text
What process is this?

Why does it exist?

Who started it?

Who owns it?

What is its PID?

What is its PPID?

What state is it in?

How much CPU is it using?

How much memory is it using?

How long has it been running?

What do its logs say?

Is its behavior expected?

Is it actually stuck?

Will another process restart it?

What happens if I stop it?

Can I terminate it gracefully?

How will I verify recovery?

What caused the problem in the first place?
```

That is the foundation of Linux process troubleshooting.

A beginner sees:

```text
CPU 100%
```

and thinks:

```text
Kill something.
```

A Linux engineer thinks:

```text
CPU is the symptom.
First, find the evidence.
```

That reasoning is what prepares you for real DevOps, Cloud, SRE, and Platform Engineering work.
