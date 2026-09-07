# Part 15: Linux CPU, Memory, Swap, and Performance Fundamentals

In Part 14, you learned how Linux handles disks, partitions, filesystems, mounting, disk capacity, and storage troubleshooting.

Now we move from storage to system performance.

When a Linux server becomes slow, one of the biggest beginner mistakes is immediately assuming:

```text
The CPU is high.
```

A slow server can be caused by many things:

```text
CPU pressure
Memory pressure
Swap activity
Disk I/O
Network delays
A single bad process
Too many processes
Application problems
Database problems
External dependencies
Resource limits
```

The goal of this part is not simply to memorize `top`.

You need to learn how to investigate a Linux system methodically.

By the end of Part 15, you should be able to look at a slow Linux server and begin answering:

```text
Is the CPU actually busy?

How many CPU cores do I have?

Is the load high relative to that capacity?

Is the server running low on usable memory?

Is Linux simply using RAM for cache?

Is swap being used?

Is swap actively under pressure?

Which processes consume the resources?

Could disk I/O be slowing everything down?

Did the kernel kill a process because memory ran out?

Where is the bottleneck?
```

The basic performance model is:

```text
Application
     |
     v
Processes
     |
     +--------+---------+---------+
     |        |         |         |
     v        v         v         v
    CPU      RAM       Disk     Network
              |
              v
             Swap
```

A performance investigation is about finding which resource, component, or dependency is limiting the workload.

---

# 1. What Are System Resources?

A program cannot run using software alone.

It requires resources from the computer.

Important system resources include:

* CPU.
* Memory.
* Storage.
* Disk I/O capacity.
* Network capacity.
* Processes and threads.
* File descriptors.
* Kernel resources.

For this part, our main focus is:

```text
CPU
Memory
Swap
Disk I/O
```

These resources work together.

For example, imagine a web application:

```text
Incoming request
      |
      v
Web application
      |
      +---- CPU executes instructions
      |
      +---- RAM stores active data
      |
      +---- Disk reads/writes files
      |
      +---- Network communicates
```

A shortage or delay in any one area can affect the application's performance.

---

# 2. What Does Performance Mean?

Performance describes how effectively a system completes its work.

Important performance characteristics include:

* Response time.
* Throughput.
* Resource utilization.
* Latency.
* Capacity.
* Queueing.
* Reliability under load.

Suppose an API normally responds in:

```text
100 milliseconds
```

but suddenly responds in:

```text
5 seconds
```

Something changed.

Your job is not to immediately restart the server.

Your job is to determine why.

---

# 3. What Is a Bottleneck?

A bottleneck is the resource or component currently limiting system performance.

Imagine water flowing through pipes:

```text
Wide pipe
   |
   v
Wide pipe
   |
   v
Narrow pipe
   |
   v
Wide pipe
```

The narrow section limits overall flow.

Computers behave similarly.

For example:

```text
CPU:      healthy
Memory:   healthy
Network:  healthy
Disk:     overloaded
```

The disk can become the bottleneck.

Adding more CPU would not solve that problem.

---

# 4. Performance Troubleshooting Is About Evidence

Do not troubleshoot like this:

```text
Server slow
   |
   v
Restart server
```

Instead:

```text
Server slow
   |
   v
Define the symptom
   |
   v
Check overall health
   |
   v
Check CPU
   |
   v
Check memory
   |
   v
Check swap
   |
   v
Check processes
   |
   v
Check disk I/O
   |
   v
Check application/service evidence
   |
   v
Identify bottleneck
   |
   v
Fix root cause
   |
   v
Verify
```

The order may change depending on the incident, but the principle remains:

```text
Measure before changing.
```

---

# 5. CPU Fundamentals

CPU stands for:

```text
Central Processing Unit
```

The CPU executes instructions.

When a program runs:

```text
Program instructions
       |
       v
Operating system schedules work
       |
       v
CPU executes instructions
```

Tasks that consume CPU include:

* Compressing files.
* Compiling software.
* Encrypting data.
* Processing application requests.
* Performing calculations.
* Running scripts.
* Parsing data.
* Rendering content.
* Running database queries.

---

# 6. CPU Cores

Modern CPUs contain processing cores.

A core can execute work independently.

A simplified system might have:

```text
CPU
 |
 +-- Core 0
 +-- Core 1
 +-- Core 2
 +-- Core 3
```

This gives the operating system multiple execution resources.

A four-core system can generally perform more CPU work concurrently than an otherwise equivalent one-core system.

---

# 7. Logical CPUs

Linux often reports logical CPUs.

Technologies such as simultaneous multithreading can allow one physical core to expose multiple logical CPUs.

You might therefore see:

```text
Physical CPU
   |
   +-- Core 1
   |     +-- Logical CPU 0
   |     +-- Logical CPU 1
   |
   +-- Core 2
         +-- Logical CPU 2
         +-- Logical CPU 3
```

For beginner performance analysis, the number of logical CPUs visible to Linux is particularly useful when interpreting load and CPU capacity.

---

# 8. Find the Number of CPUs

A simple command is:

```bash
nproc
```

Example output:

```text
4
```

This means the command sees four processing units available to the current process.

Another command is:

```bash
lscpu
```

This provides much more information.

Example fields might include:

```text
CPU(s):                  4
Core(s) per socket:      2
Socket(s):               1
Thread(s) per core:      2
Architecture:            x86_64
```

---

# 9. Understanding `lscpu`

Run:

```bash
lscpu
```

Important information can include:

* CPU architecture.
* Number of CPUs.
* Number of cores.
* Threads per core.
* Sockets.
* CPU model.
* Virtualization information.

Do not try to memorize every field.

At beginner level, ask:

```text
How much CPU capacity is visible?

What architecture is this machine?

Is this physical or virtualized?
```

---

# 10. What Is CPU Utilization?

CPU utilization describes how much CPU processing capacity is being used over a period of time.

You may see:

```text
10%
```

or:

```text
85%
```

or:

```text
100%
```

But percentages require context.

A single process using 100% CPU does not always mean an entire multi-core server has no CPU capacity remaining.

---

# 11. Why CPU Percentages Can Confuse Beginners

Suppose your server has:

```text
8 logical CPUs
```

A process may show approximately:

```text
100% CPU
```

in some tools because it is fully using approximately one logical CPU.

Other tools and monitoring systems may normalize percentages differently.

Therefore, never interpret a percentage without understanding:

* Which tool produced it.
* Whether it is per CPU.
* Whether it is system-wide.
* How many CPUs the machine has.

---

# 12. Is High CPU Always Bad?

No.

High CPU can mean the system is doing useful work.

Imagine:

```text
Video encoding job
CPU: 95%
```

If the objective is to encode a file as quickly as possible, high CPU utilization may be expected.

Another example:

```text
Build server
Compiling application
CPU: 90%
```

Again, this may be normal.

The real questions are:

```text
Is this expected?

Is performance suffering?

Is work waiting for CPU?

Which process is responsible?

How long has this condition existed?
```

---

# 13. CPU-Bound Work

A workload is CPU-bound when CPU processing capacity is the primary limitation.

Examples might include:

* Compression.
* Encryption.
* Compilation.
* Mathematical computation.
* Some data processing.
* Certain application loops.

Conceptually:

```text
Application has work
       |
       v
Needs CPU
       |
       v
CPU already busy
       |
       v
Work waits
       |
       v
Response becomes slower
```

---

# 14. Memory Fundamentals

Memory usually refers to RAM.

RAM stands for:

```text
Random Access Memory
```

RAM provides fast temporary storage that programs use while running.

Consider:

```text
Application starts
      |
      v
Program code and data
      |
      v
Loaded into memory
      |
      v
CPU works with the data
```

RAM is much faster than ordinary persistent disk storage.

---

# 15. What Uses RAM?

Linux memory may be used by:

* Applications.
* The kernel.
* Process data.
* Shared libraries.
* Filesystem cache.
* Buffers and kernel structures.
* Temporary data.

Suppose you run:

```text
Nginx
Database
Monitoring agent
Java application
SSH
Linux kernel
```

All require memory.

---

# 16. Used Memory Does Not Automatically Mean a Problem

This is one of the most important Linux memory lessons.

A beginner might run:

```bash
free -h
```

and see a small value under `free`.

They panic:

```text
Linux is almost out of RAM!
```

Not necessarily.

Linux intentionally uses otherwise idle memory for useful purposes such as caching filesystem data.

Unused RAM does very little for performance.

Linux can use available RAM to make future operations faster.

---

# 17. Memory Cache

Imagine an application reads a file from storage.

Without useful caching:

```text
Application
     |
     v
Disk
     |
     v
Data returned
```

Storage access can be relatively slow.

Linux can retain useful file data in memory.

Later:

```text
Application
     |
     v
Memory cache
     |
     v
Data returned faster
```

This is beneficial.

Therefore:

```text
RAM used for cache
```

is not automatically:

```text
RAM wasted
```

or:

```text
memory crisis
```

---

# 18. Linux Can Reclaim Cache

Some cached memory can be reclaimed when applications need memory.

Conceptually:

```text
Available RAM
    |
    v
Linux uses some for cache
    |
    v
Application needs more RAM
    |
    v
Reclaimable cache can be reduced
    |
    v
Memory becomes available
```

This is why looking only at the `free` column can mislead you.

---

# 19. The `free` Command

Run:

```bash
free
```

You might see:

```text
               total        used        free      shared  buff/cache   available
Mem:         8060000     2500000      500000      200000     5060000     5000000
Swap:        2097148           0     2097148
```

Exact values and columns can vary by system.

The raw numbers may be inconvenient.

Use:

```bash
free -h
```

---

# 20. `free -h`

The:

```text
-h
```

means:

```text
human-readable
```

Run:

```bash
free -h
```

Example:

```text
               total        used        free      shared  buff/cache   available
Mem:           7.7Gi       2.4Gi       500Mi       190Mi       4.8Gi       4.7Gi
Swap:          2.0Gi          0B       2.0Gi
```

This is easier to interpret.

---

# 21. Understanding `total`

The `total` memory value represents the amount of RAM Linux reports as available for use, with some details depending on kernel accounting.

Example:

```text
total
7.7 GiB
```

This is approximately the system's usable physical memory.

---

# 22. Understanding `used`

The `used` column represents memory considered actively used according to `free`'s accounting.

Do not treat it as:

```text
memory permanently unavailable
```

Linux memory accounting includes several categories and can change as the kernel reclaims memory.

---

# 23. Understanding `free`

The `free` column is memory that is currently completely unused.

Example:

```text
free
500 MiB
```

A low `free` value alone is not enough to conclude that memory pressure exists.

---

# 24. Understanding `buff/cache`

Linux uses memory for buffers and caches.

This helps improve performance.

Example:

```text
buff/cache
4.8 GiB
```

A beginner may think:

```text
4.8 GiB has disappeared.
```

It has not.

The operating system is using RAM productively, and much of this memory can be reclaimable when needed.

---

# 25. Understanding `available`

For basic Linux memory troubleshooting, `available` is often much more useful than `free`.

`available` estimates how much memory can be made available for starting new applications without needing to swap heavily.

Suppose:

```text
free:       300 MiB
available:  5 GiB
```

This is very different from:

```text
free:       300 MiB
available:  200 MiB
```

The second situation deserves much more attention.

---

# 26. Beginner Memory Rule

Do not ask only:

```text
How much memory is free?
```

Ask:

```text
How much memory is available?

Is available memory declining?

Is swap being used?

Is swap actively increasing?

Are processes consuming unexpected memory?

Is the system experiencing OOM events?
```

---

# 27. What Is Swap?

Swap is disk-backed space that Linux can use as part of virtual memory management.

Swap can exist as:

* A swap partition.
* A swap file.

You can inspect it using:

```bash
free -h
```

You may see:

```text
Swap:  2.0Gi  300Mi  1.7Gi
```

---

# 28. RAM vs Swap

Simplified:

```text
RAM
Fast
Physical memory
Primary working memory
```

versus:

```text
Swap
Disk-backed
Much slower than RAM
Can hold memory pages moved out of RAM
```

Do not think of swap as equivalent to adding more physical RAM.

It is not.

---

# 29. Why Linux Uses Swap

Linux may move less actively used memory pages to swap.

This can free RAM for more useful purposes.

Simplified example:

```text
RAM
 |
 +-- Active application data
 +-- Rarely used application pages
 +-- Useful filesystem cache
```

Linux may decide:

```text
Rarely used pages
      |
      v
     Swap
```

allowing RAM to be used more effectively.

---

# 30. Is Swap Usage Automatically Bad?

No.

This is another common beginner mistake.

Suppose:

```text
Swap total: 4 GiB
Swap used: 300 MiB
```

That does not automatically mean:

```text
The server is running out of memory.
```

Some pages may have been swapped earlier and remain there even after memory pressure has passed.

More important questions include:

```text
Is the system currently swapping heavily?

Is available RAM very low?

Are applications slowing down?

Is swap usage growing?

Are page-in/page-out rates high?

Is the system thrashing?
```

---

# 31. What Is Swap Pressure?

Swap pressure occurs when the system relies significantly on swapping because physical memory is constrained.

If actively needed pages repeatedly move between RAM and swap:

```text
RAM
 ^  |
 |  v
Swap
```

performance can suffer badly because disk-backed swap is far slower than RAM.

---

# 32. What Is Thrashing?

At a simplified level, thrashing occurs when the system spends excessive effort moving memory pages between RAM and storage rather than doing useful application work.

Conceptually:

```text
Application needs page A
        |
        v
Bring A from swap
        |
        v
Need space
        |
        v
Move B to swap
        |
        v
Application needs B
        |
        v
Bring B back
        |
        v
More swapping
```

The server can feel extremely slow.

---

# 33. Inspect Swap

Use:

```bash
free -h
```

You can also use:

```bash
swapon --show
```

Example:

```text
NAME      TYPE SIZE USED PRIO
/swapfile file   2G 256M   -2
```

This identifies configured active swap.

---

# 34. `/proc/swaps`

Linux also exposes swap information through:

```bash
cat /proc/swaps
```

You might see:

```text
Filename      Type      Size      Used    Priority
/swapfile     file      2097148   262144  -2
```

This is another example of `/proc` exposing kernel information.

---

# 35. Swappiness Introduction

Linux has a setting commonly called:

```text
vm.swappiness
```

You can inspect it:

```bash
sysctl vm.swappiness
```

or:

```bash
cat /proc/sys/vm/swappiness
```

It influences the kernel's tendency to swap anonymous memory relative to reclaiming other memory.

Do not interpret it as:

```text
percentage of RAM that must be used before swapping
```

That is an oversimplification.

And do not change it merely because a tutorial recommends a particular number.

Workload and system design matter.

---

# 36. `uptime`

Run:

```bash
uptime
```

Example:

```text
10:42:31 up 14 days, 3:22, 2 users, load average: 0.35, 0.42, 0.38
```

This gives you several pieces of information:

```text
Current time
How long system has been running
Number of logged-in users
Load averages
```

---

# 37. What Is System Uptime?

From:

```text
up 14 days, 3:22
```

you can see how long the machine has been running since its last boot.

This can provide context.

For example:

```text
Server rebooted 4 minutes ago
```

may explain why:

* Caches are cold.
* Services recently restarted.
* Processes have low uptime.
* A recent incident may have triggered a reboot.

---

# 38. What Is Load Average?

Linux load average is often misunderstood.

It is not simply:

```text
CPU percentage
```

Load average represents the average number of tasks that are runnable or in certain uninterruptible wait states over time.

In practical beginner terms, it helps show how much work is competing for CPU execution or waiting in relevant states, including some forms of I/O wait.

---

# 39. The Three Load Numbers

From:

```text
load average: 0.35, 0.42, 0.38
```

the values represent approximately:

```text
0.35 = 1-minute load average
0.42 = 5-minute load average
0.38 = 15-minute load average
```

These different windows help you understand the trend.

---

# 40. Why Three Time Windows Matter

Imagine:

```text
1 minute:   8.00
5 minutes:  3.00
15 minutes: 1.00
```

This suggests load has increased recently.

Conceptually:

```text
Past --------------------------> Now

15 min        5 min        1 min
1.00          3.00         8.00

                 Rising
                   /
                  /
                 /
```

Now consider:

```text
1 minute:   1.00
5 minutes:  4.00
15 minutes: 8.00
```

This suggests load was higher earlier and has been falling.

---

# 41. Load Must Be Compared With CPU Capacity

A load of:

```text
4.00
```

means very different things on different machines.

Compare:

```text
Machine A
1 logical CPU
Load: 4
```

with:

```text
Machine B
16 logical CPUs
Load: 4
```

Machine A may have substantial contention.

Machine B may have plenty of CPU execution capacity.

---

# 42. Simple Load Interpretation

As a beginner approximation, compare runnable workload with the number of logical CPUs.

For a one-CPU machine:

```text
Load around 1
```

can mean approximately one task's worth of demand is present.

For four CPUs:

```text
Load around 4
```

can represent roughly full CPU scheduling capacity if the load consists primarily of runnable CPU work.

But remember:

```text
Load is not identical to CPU utilization.
```

Tasks waiting in certain uninterruptible states can contribute to load.

---

# 43. Example: 1 CPU

System:

```bash
nproc
```

returns:

```text
1
```

Load:

```text
4.50
```

This indicates demand greatly exceeds one CPU's execution capacity, or tasks are accumulating in load-contributing wait states.

Investigation is warranted.

---

# 44. Example: 8 CPUs

System:

```bash
nproc
```

returns:

```text
8
```

Load:

```text
4.50
```

This does not automatically indicate CPU saturation.

There may be plenty of CPU capacity.

Check actual CPU utilization and process state before concluding anything.

---

# 45. Never Diagnose From Load Alone

High load can result from:

```text
CPU contention
```

but also:

```text
Tasks waiting in uninterruptible states
```

which can occur during I/O-related problems.

Therefore:

```text
High load
```

does not automatically mean:

```text
Need more CPU.
```

---

# 46. `top`

`top` is one of the most important Linux performance tools.

Run:

```bash
top
```

It provides a continuously updating view of:

* System load.
* CPU usage.
* Memory usage.
* Swap.
* Processes.
* Per-process CPU usage.
* Per-process memory usage.

Unlike commands that print once and exit, `top` remains interactive.

Press:

```text
q
```

to quit.

---

# 47. Understanding the `top` Header

A typical header might resemble:

```text
top - 12:15:32 up 3 days,  2 users,  load average: 1.20, 0.90, 0.70
Tasks: 210 total, 2 running, 208 sleeping, 0 stopped, 0 zombie
%Cpu(s): 20.0 us, 5.0 sy, 0.0 ni, 70.0 id, 5.0 wa
MiB Mem :  7970 total, 500 free, 3000 used, 4470 buff/cache
MiB Swap: 2048 total, 1800 free, 248 used
```

Then it lists processes.

---

# 48. CPU Fields in `top`

You may see:

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

At beginner level, the most important include:

```text
us
sy
id
wa
st
```

Let's understand them.

---

# 49. `us`, User CPU Time

`us` means time spent running user-space processes.

Example:

```text
us = 75%
```

This could indicate applications are performing substantial CPU work.

Possible examples:

* Java application.
* Python process.
* Database.
* Compression command.
* Build process.

---

# 50. `sy`, System CPU Time

`sy` represents CPU time spent executing kernel work.

Example:

```text
sy = 20%
```

Some system CPU usage is completely normal.

Very high system CPU may warrant investigation into:

* System calls.
* Networking.
* Storage activity.
* Kernel-related workloads.
* Process behavior.

Do not diagnose solely from one snapshot.

---

# 51. `id`, Idle CPU

`id` means CPU idle time.

Example:

```text
id = 80%
```

means the CPU is mostly idle during that measurement period.

If:

```text
id = 1%
```

the CPUs are very busy.

But you still need to determine what kind of activity is occurring.

---

# 52. `wa`, I/O Wait

`wa` generally represents CPU time spent idle while waiting for I/O completion.

High I/O wait can be a clue that storage or another I/O path is contributing to poor performance.

Example:

```text
CPU idle: relatively low
I/O wait: high
Server: slow
```

This may point toward an I/O bottleneck.

But do not diagnose a disk failure from `wa` alone.

Gather additional evidence.

---

# 53. `st`, Steal Time

On virtual machines, you may encounter:

```text
st
```

which means steal time.

This represents time the virtual CPU wanted to run but the hypervisor was servicing other work instead.

Significant steal time can indicate contention at the virtualization layer.

This is particularly relevant to virtual machines.

---

# 54. `top` Process List

Below the summary, you may see:

```text
PID USER      PR NI   VIRT   RES   SHR S %CPU %MEM   TIME+ COMMAND
```

Important beginner fields include:

```text
PID
USER
S
%CPU
%MEM
COMMAND
```

---

# 55. `PID`

`PID` means:

```text
Process ID
```

You learned this in Part 8.

Example:

```text
PID 2451
```

uniquely identifies that process at that moment.

You can inspect it further:

```bash
ps -p 2451 -f
```

---

# 56. `%CPU`

This shows CPU usage attributed to the process according to `top`'s accounting.

If a process appears at the top with very high CPU usage, it deserves investigation.

But ask:

```text
What is the process?

Is the usage expected?

How long has it been high?

Is it affecting service performance?

Is it using one CPU or many?
```

Do not immediately kill it.

---

# 57. `%MEM`

This shows the process's memory usage as a percentage of physical memory according to the tool's accounting.

A process with:

```text
%MEM 40
```

deserves attention.

But again:

```text
Large memory use
```

does not automatically mean:

```text
memory leak.
```

A database may intentionally use large amounts of memory.

Context matters.

---

# 58. `VIRT`, `RES`, and `SHR`

You may see:

```text
VIRT
RES
SHR
```

`VIRT` refers to the process's virtual address space.

`RES` represents resident memory, the portion currently resident in physical RAM.

`SHR` represents memory that may be shared with other processes.

For beginner troubleshooting, `RES` is usually more useful than simply looking at `VIRT`.

A huge `VIRT` number does not necessarily mean the process physically consumes that much RAM.

---

# 59. Process State in `top`

The `S` column may contain values such as:

```text
R
S
D
T
Z
```

Common meanings include:

```text
R = running or runnable
S = interruptible sleeping
D = uninterruptible sleep
T = stopped/traced
Z = zombie
```

A large number of processes in `D` state can be a clue during I/O-related performance problems.

---

# 60. Sorting `top` by CPU

In many `top` implementations, press:

```text
P
```

to sort by CPU usage.

This helps answer:

```text
Which processes are consuming the most CPU?
```

---

# 61. Sorting `top` by Memory

Press:

```text
M
```

in common `top` implementations to sort by memory usage.

This helps answer:

```text
Which processes are consuming the most memory?
```

Interactive key behavior can vary slightly by implementation, so use `top` help when needed.

---

# 62. `htop`

`htop` provides a more interactive process viewer.

Run:

```bash
htop
```

If installed, it can provide:

* CPU meters.
* Memory meters.
* Swap meters.
* Process tree views.
* Easier sorting.
* Interactive process management.

Not every system has `htop` installed by default.

---

# 63. `top` vs `htop`

At a beginner level:

| `top`                          | `htop`                        |
| ------------------------------ | ----------------------------- |
| Commonly installed             | May require installation      |
| Text-based                     | More interactive              |
| Very useful on minimal servers | Easier for visual exploration |
| Important administration skill | Helpful companion tool        |

Learn `top` even if you prefer `htop`.

On a production server, `top` may be available when `htop` is not.

---

# 64. Finding CPU-Heavy Processes With `ps`

You do not always need an interactive tool.

Run:

```bash
ps aux
```

You can combine it with sorting:

```bash
ps aux --sort=-%cpu | head
```

This means:

```text
ps aux
      |
      v
List processes
      |
      v
--sort=-%cpu
      |
      v
Highest CPU first
      |
      v
head
      |
      v
Show first few lines
```

---

# 65. Finding Memory-Heavy Processes

Run:

```bash
ps aux --sort=-%mem | head
```

This gives you a quick view of processes with high percentage memory usage.

Another useful approach is sorting by resident memory where supported:

```bash
ps aux --sort=-rss | head
```

`RSS` relates to resident memory.

---

# 66. High CPU Incident

Imagine users report:

```text
The API is very slow.
```

You SSH to the server.

Do not begin with:

```bash
sudo reboot
```

Start with evidence.

Run:

```bash
uptime
```

Suppose:

```text
load average: 12.20, 10.40, 4.10
```

Then:

```bash
nproc
```

returns:

```text
4
```

The load is high relative to four logical CPUs and has risen recently.

Now investigate further.

---

# 67. Check CPU With `top`

Run:

```bash
top
```

Suppose:

```text
%Cpu(s): 95 us, 3 sy, 2 id
```

CPU is almost completely busy.

The process list shows:

```text
PID    %CPU   COMMAND
4812   370    java
```

On a multi-core system, a multithreaded process may exceed 100% in `top`, depending on its display mode.

Now you have evidence pointing toward that Java process.

---

# 68. Do Not Immediately Kill High-CPU Processes

Ask:

```text
What is PID 4812?

Which service owns it?

Is the CPU expected?

What changed recently?

Is traffic unusually high?

Was there a deployment?

Is the process stuck?

Is this batch processing?

Are errors occurring?
```

Inspect:

```bash
ps -fp 4812
```

Potentially:

```bash
systemctl status your-service
```

and:

```bash
journalctl -u your-service
```

if it is a systemd service.

---

# 69. Possible High CPU Causes

High CPU can result from:

* Legitimate traffic increase.
* Infinite loop.
* Application bug.
* Expensive database work.
* Compression.
* Encryption.
* Compilation.
* Too many workers.
* Runaway script.
* Malware or unauthorized workload.
* Misconfigured scheduled job.
* Excessive retries.
* Logging or parsing workload.

The metric tells you:

```text
CPU is busy.
```

It does not tell you:

```text
why.
```

---

# 70. High CPU Troubleshooting Flow

Use:

```text
Users report slowness
        |
        v
Check uptime/load
        |
        v
Check CPU count
        |
        v
Check overall CPU utilization
        |
        v
Identify CPU-consuming processes
        |
        v
Identify service/application
        |
        v
Check logs and recent changes
        |
        v
Determine expected vs abnormal
        |
        v
Fix root cause
        |
        v
Verify CPU and application recovery
```

---

# 71. High Memory Incident

Suppose users report:

```text
Application is becoming unstable.
```

Run:

```bash
free -h
```

You see:

```text
Mem:
total       8Gi
available   300Mi

Swap:
total       2Gi
used        1.8Gi
```

This deserves investigation.

But do not immediately conclude:

```text
memory leak.
```

Find the consumers.

---

# 72. Find Memory Consumers

Run:

```bash
ps aux --sort=-%mem | head
```

or use:

```bash
top
```

and sort by memory.

Suppose:

```text
java
```

is using several gigabytes of resident memory.

Ask:

```text
Is this expected?

What is its configured memory limit?

Has usage grown over time?

Was traffic higher?

Did the application change?

Are multiple copies running?

Is there an OOM event?
```

---

# 73. Memory Leak Introduction

A memory leak occurs when a program continues retaining memory it no longer needs.

Simplified:

```text
Application starts
Memory: 500 MB
      |
      v
After 1 hour
Memory: 1 GB
      |
      v
After 4 hours
Memory: 3 GB
      |
      v
After 12 hours
Memory: 7 GB
```

If the growth continues without stabilizing and corresponds to retained unnecessary memory, the application may have a leak.

But a single high-memory snapshot cannot prove this.

You need a trend.

---

# 74. Why Time-Series Data Matters

Suppose memory is:

```text
10:00   2 GB
10:05   5 GB
10:10   7 GB
10:15   7 GB
10:20   7 GB
```

The application may simply have warmed up and stabilized.

Compare:

```text
10:00   2 GB
11:00   3 GB
12:00   4 GB
13:00   5 GB
14:00   6 GB
15:00   7 GB
```

This sustained growth deserves deeper investigation.

One snapshot cannot show the trend.

---

# 75. What Is Out of Memory?

Eventually a Linux system may be unable to satisfy memory allocation requirements.

This can lead to an:

```text
Out-of-memory
```

condition.

Linux has mechanisms to handle severe memory exhaustion.

One important mechanism is the:

```text
OOM killer
```

---

# 76. What Is the OOM Killer?

OOM means:

```text
Out Of Memory
```

Under severe memory pressure, the kernel may select one or more processes for termination to recover memory.

Simplified:

```text
Applications consume memory
        |
        v
Available memory becomes critically low
        |
        v
Reclaim/swap cannot satisfy demand
        |
        v
Kernel encounters OOM condition
        |
        v
Process selected
        |
        v
Process killed
        |
        v
Memory recovered
```

This can look like an application randomly disappearing.

It was not necessarily random.

---

# 77. Signs of an OOM Event

A service may suddenly stop.

You restart it.

It works again.

A beginner may conclude:

```text
The application crashed.
```

But the kernel may have killed it because of memory pressure.

Check kernel and system logs.

Useful commands can include:

```bash
journalctl -k
```

and:

```bash
dmesg
```

Search for OOM-related messages:

```bash
journalctl -k | grep -i -E 'oom|out of memory|killed process'
```

or:

```bash
dmesg | grep -i -E 'oom|out of memory|killed process'
```

Access to kernel messages may require elevated privileges.

---

# 78. Example OOM Evidence

You might encounter a kernel message similar to:

```text
Out of memory: Killed process 4812 (java)
```

Now you have important evidence.

The next question is not simply:

```text
How do I restart Java?
```

It is:

```text
Why did the system run out of memory?
```

---

# 79. Possible OOM Causes

Potential causes include:

* Memory leak.
* Insufficient RAM for workload.
* Too many application workers.
* Unexpected traffic.
* Multiple duplicate processes.
* Oversized application memory configuration.
* Swap unavailable or insufficient for the intended design.
* Container or service memory limits.
* Batch job consuming excessive memory.
* Application bug.

You need to identify the cause.

---

# 80. Do Not Treat Swap as a Complete OOM Fix

If a server runs out of memory, you might think:

```text
Just add lots of swap.
```

Swap can provide useful flexibility, but it is not a substitute for correctly sizing memory or fixing a memory leak.

If an application continuously allocates memory:

```text
RAM fills
   |
   v
Swap fills
   |
   v
System becomes slower
   |
   v
Memory continues growing
   |
   v
Failure still occurs
```

The underlying problem remains.

---

# 81. Observing Swap Activity

`free -h` tells you how much swap is currently used.

It does not by itself tell you the rate at which pages are moving in and out.

For activity, a useful tool is:

```bash
vmstat
```

For example:

```bash
vmstat 1
```

This prints updates approximately every second until stopped.

Press:

```text
Ctrl+C
```

to stop it.

---

# 82. Understanding `vmstat`

`vmstat` provides information about:

* Processes.
* Memory.
* Paging.
* Block I/O.
* CPU.

A simplified header may include:

```text
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
```

There is a lot here.

Do not try to memorize everything at once.

---

# 83. Important `vmstat` Fields

For beginner troubleshooting, useful fields include:

```text
r
b
si
so
us
sy
id
wa
st
```

Where broadly:

```text
r  = runnable processes
b  = processes blocked in certain waits
si = swap in
so = swap out
us = user CPU
sy = system CPU
id = idle CPU
wa = I/O wait
st = steal
```

---

# 84. Swap-In and Swap-Out

In `vmstat`:

```text
si
```

means swap-in activity.

```text
so
```

means swap-out activity.

Persistent significant activity can be evidence of memory pressure.

Do not panic over a single non-zero observation.

Look for sustained behavior together with:

```text
low available memory
+
application slowdown
+
swap activity
```

---

# 85. The First `vmstat` Line

When running:

```bash
vmstat 1
```

the first data line may represent statistics since boot rather than the same one-second interval as subsequent lines.

For short-term analysis, watch several subsequent samples.

This is an important detail when interpreting `vmstat`.

---

# 86. Disk I/O Fundamentals

I/O means:

```text
Input / Output
```

Disk I/O refers to reading data from and writing data to storage.

Examples:

```text
Reading database pages
Writing application logs
Loading files
Writing backups
Reading container layers
Writing temporary files
```

---

# 87. Disk I/O Is Not the Same as Disk Capacity

You learned disk capacity in Part 14.

These are different:

```text
Disk capacity
=
How much data can be stored
```

```text
Disk I/O performance
=
How quickly storage operations can be serviced
```

A disk can be:

```text
20% full
```

and still be an I/O bottleneck.

Likewise, a filesystem can be:

```text
95% full
```

without currently experiencing high I/O latency.

They are separate dimensions.

---

# 88. IOPS

You may encounter:

```text
IOPS
```

which means:

```text
Input/Output Operations Per Second
```

It measures the number of I/O operations a storage system can perform or is performing per second, depending on context.

This matters especially for workloads involving many storage operations.

---

# 89. Throughput

Storage throughput describes how much data is transferred over time.

For example:

```text
MB/s
```

A workload reading large sequential files may care greatly about throughput.

A workload performing many tiny random operations may care more about IOPS and latency.

---

# 90. Storage Latency

Latency describes how long an operation takes.

Simplified:

```text
Application requests disk read
          |
          v
Storage receives request
          |
       waiting
          |
          v
Data returned
```

The longer the waiting time, the higher the latency.

High storage latency can make applications slow even if CPU usage is low.

---

# 91. Why Disk I/O Can Increase Load

Remember:

```text
Load average
```

can include tasks in certain uninterruptible wait states.

If many processes wait on storage:

```text
Process A --\
Process B ---\
Process C ----> Storage
Process D ---/
Process E --/
```

load may rise even though the CPUs are not doing useful computation at 100%.

This is why:

```text
High load
```

does not always mean:

```text
High CPU demand.
```

---

# 92. `iostat`

A common storage performance tool is:

```bash
iostat
```

It is often provided by the `sysstat` package.

It may not be installed by default.

A useful command is:

```bash
iostat -xz 1
```

Options and output vary by implementation, but on common Linux systems this provides extended device statistics at intervals.

---

# 93. Important `iostat` Concepts

Depending on version, fields may include information related to:

* Reads per second.
* Writes per second.
* Read throughput.
* Write throughput.
* Request latency.
* Queueing.
* Device utilization.

Common field names can include:

```text
r/s
w/s
rkB/s
wkB/s
await
%util
```

Exact names can vary.

---

# 94. `r/s` and `w/s`

Broadly:

```text
r/s
```

means reads per second.

```text
w/s
```

means writes per second.

These help show the volume of I/O operations.

---

# 95. `await`

In common Linux `iostat` output:

```text
await
```

represents average time for I/O requests to be serviced, including relevant queue and service time.

Higher values can indicate increased storage latency.

But there is no universal number where:

```text
above X = bad
```

because acceptable latency depends on:

* Storage technology.
* Workload.
* Application requirements.
* Cloud volume type.
* Request size.
* Access pattern.

Compare against expected behavior and baseline.

---

# 96. `%util`

On many traditional block-device setups, `%util` indicates how much time the device had I/O requests in progress.

A value near 100% can be significant for some devices and workloads.

However, modern storage systems, RAID, SSDs, NVMe, virtualization, and cloud storage can make simplistic `%util = saturation` interpretations unreliable.

Use it as one signal, not the entire diagnosis.

---

# 97. A CPU Problem vs an I/O Problem

Consider two servers.

Server A:

```text
CPU user: 95%
CPU idle: 2%
I/O wait: low
```

One application process is consuming most CPU.

This points toward a CPU-heavy workload.

Server B:

```text
CPU user: 10%
CPU idle: moderate
I/O wait: high
Load: high
Many processes waiting
```

This suggests investigation of I/O.

The same symptom:

```text
Server is slow
```

can have completely different causes.

---

# 98. What Is Resource Saturation?

A resource becomes saturated when demand reaches or exceeds the amount of work the resource can service efficiently.

Think:

```text
Requests arriving
|||||||||||||||||||
          |
          v
       Resource
       ||||||||
          |
          v
Requests completed
```

If demand exceeds service capacity:

```text
Waiting queue grows
```

Latency often increases.

---

# 99. Utilization vs Saturation

These concepts are related but different.

Utilization asks:

```text
How busy is the resource?
```

Saturation asks:

```text
Is work waiting because the resource cannot service it fast enough?
```

A resource can be highly utilized without causing a problem if it is keeping up with demand.

Saturation is often more directly related to performance degradation.

---

# 100. Errors Matter Too

When analyzing a resource, also look for errors.

For example:

```text
CPU
Utilization
Saturation
Errors/context
```

```text
Memory
Usage
Pressure
OOM events
```

```text
Disk
Utilization
Queueing/latency
I/O errors
```

```text
Network
Utilization
Drops/retransmissions
Errors
```

Performance troubleshooting should not focus on percentages alone.

---

# 101. The Slow-Server Problem

Suppose someone messages you:

```text
Production is slow.
```

That statement is not yet useful enough.

Ask:

```text
What exactly is slow?

Website?

API?

SSH?

Database?

File operations?

Everything?

When did it begin?

Is it constant or intermittent?

Which users are affected?

Was anything changed?
```

You need to define the symptom.

---

# 102. Step 1: Establish the Scope

Determine whether the problem affects:

```text
One application
One service
One server
One availability zone
One customer
All users
All servers
```

This changes your investigation.

If one application is slow while the server is healthy, the problem may not be system-wide.

---

# 103. Step 2: Check Uptime and Load

Run:

```bash
uptime
```

Example:

```text
up 120 days, load average: 18.5, 14.2, 8.1
```

Then determine CPU capacity:

```bash
nproc
```

Suppose:

```text
4
```

Load is elevated relative to CPU count.

But you still do not know why.

---

# 104. Step 3: Inspect Overall System Activity

Run:

```bash
top
```

Look at:

```text
CPU
Memory
Swap
Load
Process states
Top CPU consumers
Top memory consumers
```

Do not focus on only one number.

---

# 105. Step 4: Check Memory

Run:

```bash
free -h
```

Ask:

```text
How much RAM is available?

How much swap is used?

Is available memory critically low?
```

Then use:

```bash
vmstat 1
```

if you need to observe active swapping and general system activity.

---

# 106. Step 5: Identify Processes

For CPU:

```bash
ps aux --sort=-%cpu | head
```

For memory:

```bash
ps aux --sort=-%mem | head
```

Then investigate suspicious PIDs:

```bash
ps -fp PID
```

Example:

```bash
ps -fp 4812
```

Determine:

```text
Who owns it?

What command started it?

Which service does it belong to?

Is it expected?
```

---

# 107. Step 6: Check Disk Capacity

Even in a performance incident, check storage capacity:

```bash
df -h
```

Why?

Because a nearly full or full filesystem can cause applications to:

* Fail writes.
* Fail log operations.
* Fail temporary-file creation.
* Behave unpredictably.
* Stop services.

Also:

```bash
df -i
```

when inode exhaustion is plausible.

---

# 108. Step 7: Check I/O

Use:

```bash
vmstat 1
```

and, when available:

```bash
iostat -xz 1
```

Look for evidence of:

* I/O wait.
* Blocked tasks.
* High latency.
* Heavy reads/writes.
* Queueing.
* Storage errors.

Then correlate with the application's behavior.

---

# 109. Step 8: Check Logs

If a service is affected:

```bash
systemctl status service-name
```

Then:

```bash
journalctl -u service-name
```

For recent messages:

```bash
journalctl -u service-name --since "10 minutes ago"
```

You can also inspect relevant application log files.

Do not ignore application evidence while staring at system metrics.

---

# 110. Step 9: Check Kernel Evidence

For memory and hardware-related problems:

```bash
journalctl -k
```

or:

```bash
dmesg
```

Look for:

* OOM events.
* Storage errors.
* Filesystem errors.
* Network device issues.
* Kernel warnings.

---

# 111. Step 10: Check Recent Changes

Ask:

```text
Was there a deployment?

Was configuration changed?

Was traffic increased?

Was a cron job started?

Was a backup running?

Was a new monitoring agent installed?

Was the VM resized?

Was a database migration running?

Was logging changed?

Was a new container deployed?
```

Performance incidents often have a trigger.

---

# 112. Performance Troubleshooting Methodology

Use this framework:

```text
1. Observe
2. Define
3. Measure
4. Compare
5. Correlate
6. Form hypothesis
7. Test
8. Remediate
9. Verify
10. Prevent
```

Let's understand each step.

---

# 113. Observe

Start with the reported symptom.

Example:

```text
API latency increased.
```

Do not convert it immediately into:

```text
CPU problem.
```

The symptom is:

```text
latency
```

The cause is still unknown.

---

# 114. Define

Make the problem specific.

Instead of:

```text
Server is slow.
```

define:

```text
API response time increased from 200 ms to 4 seconds beginning around 14:20.
```

That gives you something measurable.

---

# 115. Measure

Gather evidence:

```bash
uptime
nproc
free -h
top
vmstat 1
df -h
```

and when appropriate:

```bash
iostat -xz 1
```

Also gather application metrics and logs.

---

# 116. Compare

Compare the current state against:

* Normal baseline.
* Earlier measurements.
* Similar servers.
* Expected workload.
* Known capacity.

For example:

```text
Normal load: 1 to 3
Current load: 25
```

is more meaningful than seeing:

```text
Load: 25
```

without context.

---

# 117. Correlate

Look for relationships.

Example:

```text
14:20 deployment
        |
        v
14:21 CPU increases
        |
        v
14:22 API latency increases
        |
        v
New process consumes 350% CPU
```

This does not automatically prove causation, but it creates a strong hypothesis.

---

# 118. Form a Hypothesis

Example:

```text
The new application release appears to contain a CPU-intensive code path that is saturating available CPU under production traffic.
```

This is much stronger than:

```text
CPU is high.
```

The first statement can be tested.

---

# 119. Test the Hypothesis

Possible tests might include:

* Compare with another instance.
* Inspect process CPU.
* Review application logs.
* Check release timing.
* Roll back according to deployment procedure.
* Reproduce safely.
* Profile the application.

Do not perform risky production experiments without appropriate controls.

---

# 120. Remediate

The remediation depends on the cause.

Examples:

```text
CPU bug
→ fix or roll back application
```

```text
Legitimate traffic growth
→ scale capacity
```

```text
Memory leak
→ fix application and manage immediate memory pressure
```

```text
Disk bottleneck
→ address workload/storage architecture
```

```text
Runaway scheduled job
→ stop/correct job safely
```

---

# 121. Verify

After remediation, confirm:

```text
Did CPU recover?

Did available memory recover?

Did swap activity decrease?

Did load fall?

Did disk latency improve?

Did application response time recover?

Did errors stop?
```

Never assume:

```text
Command succeeded
```

means:

```text
Incident solved.
```

---

# 122. Prevent Recurrence

After recovery, ask:

```text
Why was this not detected earlier?

Do we need an alert?

Do we need resource limits?

Do we need capacity changes?

Do we need better logging?

Do we need application changes?

Do we need autoscaling?

Do we need better load testing?

Do we need a runbook?
```

Troubleshooting should improve the system.

---

# 123. Common Beginner Mistake: Restarting First

Suppose CPU is high.

You restart the application.

CPU returns to normal.

You conclude:

```text
Fixed.
```

But you may have destroyed useful evidence.

The problem might return in three hours.

Before restarting when circumstances allow, collect:

```text
Process information
CPU state
Memory state
Logs
Load
Relevant metrics
Recent changes
```

Sometimes immediate restoration is necessary, but evidence collection remains important.

---

# 124. Common Beginner Mistake: Killing With `kill -9`

You learned this in Part 8.

A high CPU process appears.

Beginner response:

```bash
kill -9 PID
```

That sends `SIGKILL`.

The process cannot perform normal cleanup.

Prefer understanding the process and, when termination is appropriate, using graceful methods first.

High resource consumption alone is not permission to kill a production process.

---

# 125. Common Beginner Mistake: Looking Only at CPU

A server is slow.

`top` shows:

```text
CPU idle: 70%
```

Beginner says:

```text
The server is fine.
```

But perhaps:

```text
I/O wait is high
```

or:

```text
Application is waiting for database
```

or:

```text
Memory is under pressure
```

or:

```text
External API is slow.
```

CPU is only one part of the system.

---

# 126. Common Beginner Mistake: Looking Only at `free`

Example:

```text
free memory: 200 MiB
```

Beginner says:

```text
We need more RAM.
```

But:

```text
available memory: 6 GiB
```

The system may be healthy.

Understand Linux memory accounting.

---

# 127. Common Beginner Mistake: Treating Any Swap Use as Failure

Example:

```text
Swap used: 250 MiB
```

Beginner:

```text
Memory problem!
```

Not necessarily.

Check:

```text
Available memory
Swap-in/out activity
Application latency
Memory trend
OOM evidence
```

A static amount of used swap is not enough to diagnose memory pressure.

---

# 128. Common Beginner Mistake: Comparing Load to 1

Someone sees:

```text
load average: 4
```

and says:

```text
The server is overloaded because load is above 1.
```

But the server has:

```text
32 CPUs
```

Load must be interpreted relative to capacity and workload state.

---

# 129. Common Beginner Mistake: Assuming High Load Means CPU

High load plus low CPU utilization can point toward waiting tasks.

Check:

```bash
vmstat 1
```

and:

```bash
iostat -xz 1
```

when available.

Also inspect process states.

High load is evidence.

It is not the root cause.

---

# 130. Common Beginner Mistake: Installing Tools During an Incident Without Thinking

You may discover:

```bash
htop
```

or:

```bash
iostat
```

is not installed.

Do not blindly modify production servers because a tutorial says:

```bash
sudo apt install ...
```

Consider:

* Change-control requirements.
* Repository availability.
* Production policy.
* Existing tools.
* Security restrictions.

Often you can investigate substantially with tools already installed.

---

# 131. Real-World Scenario 1: CPU Saturation

Users report:

```text
Checkout API is slow.
```

You run:

```bash
uptime
```

Output:

```text
load average: 14.2, 11.8, 5.1
```

Then:

```bash
nproc
```

Output:

```text
4
```

Then:

```bash
top
```

You observe:

```text
CPU idle: 1%
I/O wait: low
```

One application consumes most CPU.

This points toward CPU saturation.

---

# 132. Investigating the CPU Scenario

Check:

```bash
ps aux --sort=-%cpu | head
```

You identify:

```text
checkout-service
```

Then inspect logs.

You discover repeated messages related to processing the same failed transaction.

A deployment occurred 20 minutes earlier.

Potential hypothesis:

```text
New application behavior is repeatedly retrying failed work without sufficient backoff, causing CPU saturation.
```

Now you have a technical direction.

---

# 133. Real-World Scenario 2: Memory Pressure

Users report:

```text
Application becomes slower throughout the day.
```

You run:

```bash
free -h
```

and see:

```text
available: 250 MiB
swap used: 3.5 GiB
```

Then:

```bash
vmstat 1
```

shows sustained swap activity.

Then:

```bash
ps aux --sort=-rss | head
```

shows one application consuming increasing memory.

Historical monitoring shows:

```text
09:00  2 GB
11:00  4 GB
13:00  6 GB
15:00  9 GB
```

This is evidence of abnormal memory growth.

---

# 134. Real-World Scenario 3: OOM

Users report:

```text
Application suddenly disappeared at 03:14.
```

You run:

```bash
journalctl -k | grep -i -E 'oom|out of memory|killed process'
```

You find an OOM event involving the application.

Now check:

* Memory consumption.
* Workload.
* Application configuration.
* Other processes.
* Container/service limits.
* Recent changes.
* Swap configuration.

The restart is not the root-cause analysis.

---

# 135. Real-World Scenario 4: High Load but CPU Not Busy

Users report:

```text
Server is extremely slow.
```

`uptime` shows:

```text
load average: 30, 28, 25
```

The server has:

```text
8 CPUs
```

You expect CPU saturation.

But `top` shows relatively low user CPU and significant I/O wait.

Several processes are in:

```text
D
```

state.

`iostat` shows elevated storage latency.

Now the investigation moves toward storage.

Adding CPU would probably not solve the bottleneck.

---

# 136. Real-World Scenario 5: Low Free Memory but Healthy Server

Run:

```bash
free -h
```

You see:

```text
total:       32 GiB
free:        600 MiB
available:   18 GiB
```

A beginner may panic.

But there is still substantial available memory because Linux can reclaim suitable cached memory.

No significant swap activity exists.

Applications are healthy.

This is not evidence of a memory crisis.

---

# 137. Real-World Scenario 6: Swap Used but No Current Pressure

You see:

```text
Swap total: 4 GiB
Swap used: 800 MiB
```

But:

```text
Available RAM: 10 GiB
```

and:

```bash
vmstat 1
```

shows:

```text
si = 0
so = 0
```

across repeated samples.

This suggests there is no significant current swapping activity.

Some older inactive pages may simply remain in swap.

Do not diagnose from swap usage alone.

---

# 138. Real-World Scenario 7: Backup Job Creates I/O Pressure

Every night at 02:00:

```text
API latency rises.
```

CPU remains moderate.

Memory remains healthy.

But disk latency rises sharply.

You discover:

```text
02:00
backup job starts
```

The backup reads large amounts of application data from the same storage used by the application.

Possible bottleneck:

```text
Application I/O
       |
       +----------+
                  |
                  v
               Storage
                  ^
                  |
       +----------+
       |
Backup I/O
```

The two workloads compete for storage resources.

---

# 139. Real-World Scenario 8: Too Many Workers

An application is configured with:

```text
200 workers
```

on a small VM.

Each worker consumes:

```text
100 MB
```

Approximate memory demand:

```text
200 × 100 MB
=
20,000 MB
```

That is roughly 20 GB before considering the operating system and other services.

If the server has:

```text
8 GB RAM
```

you have a serious capacity mismatch.

More concurrency is not automatically better.

---

# 140. Real-World Scenario 9: CPU Is High Because Work Increased

Suppose traffic doubles.

CPU rises from:

```text
40%
```

to:

```text
80%
```

but:

* Latency remains acceptable.
* Error rates remain normal.
* Load remains manageable.
* Capacity remains sufficient.

This may simply represent legitimate resource utilization.

High utilization should be interpreted in context.

---

# 141. Real-World Scenario 10: Runaway Script

Someone runs:

```bash
while true; do
    some-expensive-command
done
```

The loop has no pause and no exit condition.

CPU rises sharply.

Use:

```bash
ps aux --sort=-%cpu | head
```

to identify the process.

Inspect:

```bash
ps -fp PID
```

Determine the owner and command.

Then terminate appropriately if authorized and safe.

The root cause is the runaway loop, not Linux itself.

---

# 142. System Resource Investigation Lab

This lab teaches you how CPU, memory, swap, load, and I/O appear on a real Linux system.

Use a disposable Linux VM or lab server.

Do not intentionally exhaust CPU, RAM, or storage on production systems.

---

# 143. Lab Step 1: Establish System Capacity

Run:

```bash
nproc
```

Record:

```text
Logical CPUs:
```

Then:

```bash
lscpu
```

Identify:

```text
Architecture:
CPU(s):
Thread(s) per core:
Core(s) per socket:
Socket(s):
```

Do not worry if your virtual machine exposes simplified topology.

---

# 144. Lab Step 2: Check Uptime

Run:

```bash
uptime
```

Record:

```text
System uptime:
1-minute load:
5-minute load:
15-minute load:
```

Now compare the load with:

```bash
nproc
```

Do not simply label it high or low.

Explain why.

---

# 145. Lab Step 3: Inspect Memory

Run:

```bash
free -h
```

Record:

```text
Total RAM:
Used RAM:
Free RAM:
Buff/cache:
Available RAM:
Swap total:
Swap used:
Swap free:
```

Then answer:

```text
Why is available memory more useful than looking only at free memory?
```

---

# 146. Lab Step 4: Inspect Swap

Run:

```bash
swapon --show
```

If no output appears, your lab may not currently have active swap.

Also run:

```bash
cat /proc/swaps
```

Compare the results.

Do not create swap simply because none exists. The lab's objective here is inspection.

---

# 147. Lab Step 5: Inspect Overall Performance

Run:

```bash
top
```

Identify:

```text
Load average
CPU user time
CPU system time
CPU idle
I/O wait
Memory
Swap
Running processes
Sleeping processes
Zombie processes
```

Then find the highest CPU process.

Then find the highest memory process.

Exit with:

```text
q
```

---

# 148. Lab Step 6: Use `ps` for CPU

Run:

```bash
ps aux --sort=-%cpu | head
```

Identify:

```text
PID
USER
%CPU
%MEM
COMMAND
```

for the highest CPU-consuming process.

Then:

```bash
ps -fp PID
```

using the real PID.

---

# 149. Lab Step 7: Use `ps` for Memory

Run:

```bash
ps aux --sort=-%mem | head
```

Then:

```bash
ps aux --sort=-rss | head
```

Compare the results.

Ask:

```text
Which processes consume the most resident memory?

Are they expected services?
```

---

# 150. Lab Step 8: Observe `vmstat`

Run:

```bash
vmstat 1
```

Watch at least several samples.

Focus on:

```text
r
b
si
so
us
sy
id
wa
st
```

Stop with:

```text
Ctrl+C
```

Record what you observed.

---

# 151. Lab Step 9: Create Controlled CPU Load

Only on a disposable lab system.

A simple CPU-consuming command is:

```bash
yes > /dev/null &
```

What happens?

`yes` continuously generates output.

```text
yes
 |
 v
continuous output
 |
 v
/dev/null
```

Because `/dev/null` discards the output, the process can run continuously and consume CPU.

The:

```text
&
```

runs it in the background.

---

# 152. Lab Step 10: Identify the CPU Load

Run:

```bash
top
```

Find:

```text
yes
```

Observe its CPU usage.

Then quit `top`.

Use:

```bash
pgrep yes
```

to find the PID.

You may also use:

```bash
ps aux | grep '[y]es'
```

---

# 153. Lab Step 11: Stop the CPU Load Gracefully

Use:

```bash
pkill yes
```

or terminate the specific lab PID with:

```bash
kill PID
```

Do not begin with:

```bash
kill -9 PID
```

Verify:

```bash
pgrep yes
```

No matching lab process should remain.

---

# 154. Lab Step 12: Observe CPU Recovery

Run:

```bash
top
```

or:

```bash
vmstat 1
```

Compare CPU idle time with what you observed while `yes` was running.

You have now intentionally created and removed a CPU-heavy workload.

---

# 155. Lab Step 13: Compare Load

Immediately after stopping the workload:

```bash
uptime
```

Observe load average.

You may notice that load does not instantly return to its previous value.

Why?

Because load average represents smoothed averages over:

```text
1 minute
5 minutes
15 minutes
```

It reflects recent history, not only the current instant.

---

# 156. Lab Step 14: Create Multiple CPU Workers

If your disposable VM has multiple CPUs, you can create a small number of controlled workers.

For example:

```bash
yes > /dev/null &
yes > /dev/null &
```

Then:

```bash
top
```

Observe:

* CPU usage.
* Load.
* Number of running processes.

Do not create dozens of workers.

The goal is understanding, not crashing the VM.

---

# 157. Lab Step 15: Stop All Lab Workers

Run:

```bash
pkill yes
```

Verify:

```bash
pgrep yes
```

Then:

```bash
uptime
```

and:

```bash
top
```

Observe recovery.

---

# 158. Lab Step 16: Controlled Memory Observation

Do not intentionally exhaust all RAM.

Instead, inspect a process you already know.

For example:

```bash
ps aux --sort=-rss | head
```

Choose a non-sensitive process and note:

```text
PID
RSS
%MEM
```

Then compare with `top`.

The goal is learning how tools report memory, not forcing an OOM event.

---

# 159. Lab Step 17: Check for OOM History

Run:

```bash
sudo journalctl -k | grep -i -E 'oom|out of memory|killed process'
```

If nothing appears, that is fine.

It means you may not have relevant OOM events in the available journal.

Do not intentionally create an OOM incident just to generate output.

---

# 160. Lab Step 18: Check Disk Capacity

Run:

```bash
df -h
```

Then:

```bash
df -i
```

Confirm that neither:

```text
byte capacity
```

nor:

```text
inode capacity
```

is exhausted.

This reinforces Part 14.

---

# 161. Lab Step 19: Observe I/O With `vmstat`

Run:

```bash
vmstat 1
```

Observe:

```text
bi
bo
wa
```

Broadly:

```text
bi
=
blocks received from block devices

bo
=
blocks sent to block devices

wa
=
I/O wait
```

Exact units and interpretation depend on the tool/system.

Watch the pattern rather than memorizing one sample.

---

# 162. Lab Step 20: Use `iostat` If Available

Check:

```bash
command -v iostat
```

If a path appears, run:

```bash
iostat -xz 1
```

Watch several samples.

Focus on:

```text
read activity
write activity
latency
device utilization
```

Stop with:

```text
Ctrl+C
```

If `iostat` is unavailable, do not install it unless appropriate for your lab environment.

---

# 163. Lab Step 21: Observe Normal Baseline

Run these when the lab machine is mostly idle:

```bash
uptime
```

```bash
free -h
```

```bash
vmstat 1
```

```bash
top
```

Record normal values.

This teaches an important performance principle:

```text
You need to know what normal looks like.
```

Without a baseline, it is harder to know what abnormal means.

---

# 164. Lab Step 22: Build a Resource Snapshot

Create a simple troubleshooting snapshot:

```bash
echo "=== UPTIME ==="
uptime

echo "=== CPU COUNT ==="
nproc

echo "=== MEMORY ==="
free -h

echo "=== DISK ==="
df -h

echo "=== INODES ==="
df -i

echo "=== TOP CPU ==="
ps aux --sort=-%cpu | head

echo "=== TOP MEMORY ==="
ps aux --sort=-%mem | head
```

Run it.

Understand every command before turning such checks into scripts.

---

# 165. Lab Step 23: Interpret the Snapshot

Do not simply save the output.

Answer:

```text
How many CPUs exist?

What is the load?

Is the load high relative to CPU count?

How much memory is available?

Is swap configured?

Is swap heavily used?

Which process uses most CPU?

Which process uses most memory?

Are filesystems close to full?

Are inodes close to full?
```

This turns commands into reasoning.

---

# 166. Lab Step 24: Troubleshooting Scenario

Pretend you receive:

```text
The server is slow.
```

Without changing anything, perform:

```bash
uptime
nproc
free -h
top
vmstat 1
df -h
df -i
ps aux --sort=-%cpu | head
ps aux --sort=-%mem | head
```

Then write a short diagnosis.

Example:

```text
The server currently has four logical CPUs.

Load is 0.8, 0.7, 0.6, which does not indicate obvious CPU contention relative to four CPUs.

CPU is mostly idle.

Available memory is healthy.

There is no sustained swap activity.

Disk and inode capacity are healthy.

No process shows abnormal CPU or memory consumption.

Current system-level evidence does not show obvious CPU, memory, swap, or capacity pressure.

The next investigation should move toward the affected application, network, database, external dependencies, and historical metrics.
```

That is far better than:

```text
Server looks fine.
```

---

# 167. Lab Step 25: Create a High-CPU Scenario

On the disposable system:

```bash
yes > /dev/null &
```

Wait briefly.

Then collect:

```bash
uptime
```

```bash
top
```

```bash
ps aux --sort=-%cpu | head
```

```bash
vmstat 1
```

Identify the evidence indicating CPU demand.

Then stop the process:

```bash
pkill yes
```

Verify recovery.

---

# 168. Lab Step 26: Explain What Happened

Your explanation should look something like:

```text
The yes process continuously generated output and discarded it into /dev/null.

Because the process performed work continuously without waiting for user input, it consumed substantial CPU time.

top and ps identified the process as a major CPU consumer.

vmstat showed increased user CPU utilization and reduced idle CPU.

After terminating the process, CPU idle capacity recovered.

Load average declined more gradually because it represents historical averages.
```

This is the reasoning you should practice.

---

# 169. Performance Investigation Checklist

When a Linux server becomes slow, your first-pass checklist can be:

```text
1. Define exactly what is slow.
2. Check uptime.
3. Check CPU count.
4. Check load averages.
5. Inspect overall CPU utilization.
6. Inspect memory availability.
7. Check swap usage.
8. Check active swap behavior.
9. Identify high-CPU processes.
10. Identify high-memory processes.
11. Check filesystem capacity.
12. Check inode capacity.
13. Check disk I/O when indicated.
14. Check process states.
15. Check application logs.
16. Check kernel logs.
17. Check recent deployments/configuration changes.
18. Check traffic/workload changes.
19. Form a hypothesis.
20. Test it safely.
21. Remediate the cause.
22. Verify recovery.
23. Determine how to prevent recurrence.
```

Do not mechanically run every command in every incident.

Use the evidence to guide the next check.

---

# 170. What You Should Now Understand

Before continuing to Part 16, you should confidently understand:

* What system resources are.
* Why applications require CPU, memory, storage, and network resources.
* What performance means.
* What a bottleneck is.
* Why troubleshooting requires evidence.
* What a CPU does.
* What CPU cores are.
* What logical CPUs are.
* How to use `nproc`.
* What `lscpu` shows.
* What CPU utilization means.
* Why CPU percentages require context.
* Why high CPU is not automatically bad.
* What CPU-bound work means.
* What RAM is.
* What uses RAM.
* Why low completely free RAM does not automatically indicate a problem.
* Why Linux uses memory for cache.
* Why cache can improve performance.
* Why some cached memory can be reclaimed.
* What `free` does.
* What `free -h` does.
* What total, used, free, buff/cache, and available broadly mean.
* Why available memory matters.
* What swap is.
* Why Linux uses swap.
* Why swap is slower than RAM.
* Why some swap usage is not automatically bad.
* What swap pressure means.
* What thrashing means at a beginner level.
* How to inspect swap.
* What `swapon --show` does.
* What swappiness is at a beginner level.
* Why swappiness should not be blindly changed.
* What `uptime` reports.
* What load average means.
* What the 1-minute load represents.
* What the 5-minute load represents.
* What the 15-minute load represents.
* Why the three load windows show trends.
* Why load must be compared with CPU capacity.
* Why load is not the same as CPU percentage.
* Why I/O waits can contribute to load.
* What `top` does.
* How to read the basic `top` header.
* What `us` means.
* What `sy` means.
* What `id` means.
* What `wa` means.
* What `st` means.
* How `top` shows processes.
* What `%CPU` means.
* What `%MEM` means.
* Why `RES` is useful.
* What common process states mean.
* What `htop` is.
* Why you should still learn `top`.
* How to find CPU-heavy processes using `ps`.
* How to find memory-heavy processes.
* How to investigate high CPU.
* Why you should not immediately kill a high-CPU process.
* Common causes of high CPU.
* How to investigate high memory.
* What a memory leak is.
* Why one memory snapshot cannot prove a leak.
* Why historical trends matter.
* What an OOM condition is.
* What the OOM killer does.
* How to look for OOM evidence.
* Why restarting an OOM-killed application does not solve the root cause.
* Why swap is not a complete solution for insufficient memory.
* What `vmstat` does.
* What `r`, `b`, `si`, `so`, `us`, `sy`, `id`, `wa`, and `st` broadly represent.
* Why repeated `vmstat` samples matter.
* What disk I/O means.
* Why disk I/O performance differs from disk capacity.
* What IOPS means.
* What throughput means.
* What storage latency means.
* Why storage latency can slow applications.
* Why I/O problems can produce high load.
* What `iostat` is.
* What `r/s` and `w/s` broadly mean.
* What `await` represents.
* Why `%util` must be interpreted carefully.
* How CPU-bound and I/O-bound problems differ.
* What resource saturation means.
* The difference between utilization and saturation.
* Why errors are part of performance analysis.
* How to investigate a slow Linux server systematically.
* Why recent changes matter.
* Why baselines matter.
* Why evidence should be collected before disruptive actions when possible.
* Why restarting first can hide the root cause.
* Why `kill -9` should not be your first response.
* Why low `free` memory alone is not a diagnosis.
* Why used swap alone is not a diagnosis.
* Why high load alone is not a diagnosis.
* How CPU, memory, swap, disk I/O, processes, and applications interact.

---

# 171. Hands-On Practice

Complete these tasks on a disposable Linux environment:

1. Run `nproc`.
2. Record the number of logical CPUs.
3. Run `lscpu`.
4. Identify the architecture.
5. Identify CPU count.
6. Identify cores and threads where available.
7. Run `uptime`.
8. Record the system uptime.
9. Record the 1-minute load.
10. Record the 5-minute load.
11. Record the 15-minute load.
12. Compare load with CPU count.
13. Explain why load is not CPU percentage.
14. Run `free`.
15. Run `free -h`.
16. Record total RAM.
17. Record used RAM.
18. Record free RAM.
19. Record buff/cache.
20. Record available RAM.
21. Explain why available differs from free.
22. Record total swap.
23. Record used swap.
24. Run `swapon --show`.
25. Run `cat /proc/swaps`.
26. Explain what swap is.
27. Explain why swap is slower than RAM.
28. Explain why swap usage is not automatically a problem.
29. Inspect `vm.swappiness`.
30. Explain why you should not blindly modify it.
31. Run `top`.
32. Identify user CPU.
33. Identify system CPU.
34. Identify idle CPU.
35. Identify I/O wait.
36. Identify steal time if present.
37. Identify the highest CPU process.
38. Identify the highest memory process.
39. Identify its PID.
40. Identify its user.
41. Identify its process state.
42. Run `ps aux --sort=-%cpu | head`.
43. Compare with `top`.
44. Run `ps aux --sort=-%mem | head`.
45. Run `ps aux --sort=-rss | head`.
46. Explain `VIRT`.
47. Explain `RES`.
48. Explain why a huge `VIRT` value does not necessarily equal physical RAM consumption.
49. Run `vmstat 1`.
50. Observe at least five subsequent samples.
51. Identify `r`.
52. Identify `b`.
53. Identify `si`.
54. Identify `so`.
55. Identify `us`.
56. Identify `sy`.
57. Identify `id`.
58. Identify `wa`.
59. Identify `st`.
60. Explain what sustained swap-in/out activity may indicate.
61. Run `df -h`.
62. Verify filesystem capacity.
63. Run `df -i`.
64. Verify inode capacity.
65. Check whether `iostat` exists.
66. Run `iostat -xz 1` if available.
67. Observe reads.
68. Observe writes.
69. Observe latency.
70. Observe device utilization.
71. Create one controlled `yes > /dev/null &` process.
72. Find its PID.
73. Observe its CPU consumption.
74. Observe changes in `vmstat`.
75. Observe changes in `top`.
76. Observe load average.
77. Terminate the test process gracefully.
78. Verify it stopped.
79. Observe CPU recovery.
80. Observe how load declines over time.
81. Explain why load does not instantly return to its earlier value.
82. Check the kernel journal for OOM events.
83. Explain what the OOM killer does.
84. Explain why an application may disappear during an OOM event.
85. Explain what a memory leak is.
86. Explain why one snapshot cannot prove a memory leak.
87. Explain what a CPU bottleneck looks like.
88. Explain what an I/O bottleneck may look like.
89. Explain what memory pressure may look like.
90. Explain what swap pressure may look like.
91. Explain why a nearly full disk is different from slow disk I/O.
92. Explain IOPS.
93. Explain throughput.
94. Explain latency.
95. Explain utilization.
96. Explain saturation.
97. Build a basic resource snapshot.
98. Interpret the snapshot.
99. Write a slow-server investigation checklist.
100. Explain why evidence should come before remediation.

---

# 172. Beginner Knowledge Check

1. What is a system resource?
2. What does CPU stand for?
3. What does the CPU do?
4. What is a CPU core?
5. What is a logical CPU?
6. What does `nproc` show?
7. What does `lscpu` show?
8. What is CPU utilization?
9. Is high CPU automatically bad?
10. What is a CPU-bound workload?
11. What does RAM stand for?
12. What is RAM used for?
13. Why does Linux use RAM for cache?
14. Is cached RAM necessarily wasted memory?
15. Can Linux reclaim some cached memory?
16. What does `free` show?
17. What does `free -h` change?
18. What does `total` memory broadly represent?
19. What does `free` memory represent?
20. What does `available` memory represent?
21. Why is available memory often more useful than free memory?
22. What is swap?
23. Is swap as fast as RAM?
24. Why does Linux use swap?
25. Is any swap usage automatically bad?
26. What is swap pressure?
27. What is thrashing?
28. What does `swapon --show` do?
29. What is `vm.swappiness`?
30. Should you blindly set swappiness to a number from a tutorial?
31. What does `uptime` show?
32. What is load average?
33. What are the three load-average windows?
34. What does the first load value represent?
35. What does the second represent?
36. What does the third represent?
37. Why should load be compared with CPU count?
38. Is load average the same as CPU utilization?
39. Can I/O-related waiting contribute to load?
40. What does `top` do?
41. What does `us` mean in CPU statistics?
42. What does `sy` mean?
43. What does `id` mean?
44. What does `wa` mean?
45. What does `st` mean?
46. What does `%CPU` show for a process?
47. What does `%MEM` show?
48. What does `RES` broadly represent?
49. What does `VIRT` broadly represent?
50. Why should you not diagnose memory use from `VIRT` alone?
51. What does process state `R` mean?
52. What does `S` mean?
53. What does `D` broadly mean?
54. What does `Z` mean?
55. What is `htop`?
56. Why is learning `top` still important?
57. How can you list processes sorted by CPU?
58. How can you list processes sorted by memory?
59. What should you do before killing a high-CPU process?
60. Name three possible causes of high CPU.
61. What is memory pressure?
62. What is a memory leak?
63. Can one high-memory snapshot prove a memory leak?
64. Why are historical measurements useful?
65. What does OOM stand for?
66. What is the OOM killer?
67. Why might Linux kill a process during severe memory pressure?
68. Where can you look for OOM evidence?
69. Does restarting an OOM-killed application necessarily fix the problem?
70. Why is adding swap not always a complete solution?
71. What does `vmstat` show?
72. What does `r` broadly represent?
73. What does `b` broadly represent?
74. What does `si` mean?
75. What does `so` mean?
76. What does `wa` indicate?
77. Why should you observe several `vmstat` samples?
78. What is disk I/O?
79. Is disk capacity the same as disk I/O performance?
80. What does IOPS mean?
81. What is storage throughput?
82. What is storage latency?
83. Why can storage latency make a server slow?
84. What is `iostat` used for?
85. What does `r/s` broadly represent?
86. What does `w/s` broadly represent?
87. What does `await` broadly represent?
88. Should `%util` always be interpreted as a perfect saturation measurement?
89. What is a bottleneck?
90. What is resource utilization?
91. What is resource saturation?
92. Why is saturation important?
93. Why should errors be considered during performance investigations?
94. What should you do first when someone says a server is slow?
95. Why should you check recent changes?
96. Why are baselines useful?
97. Why can restarting immediately be harmful to troubleshooting?
98. Why should `kill -9` not be the first response to high CPU?
99. What should you verify after remediation?
100. What is the main objective of Linux performance troubleshooting?

---

# 173. Beginner Knowledge Check Answers

1. A computing resource used by the operating system and applications, such as CPU, memory, storage, or network capacity.

2. Central Processing Unit.

3. It executes instructions.

4. An independent processing unit within a CPU.

5. A CPU execution unit exposed to the operating system, which may represent a physical core or a hardware thread.

6. The number of processing units available to the current process.

7. CPU architecture and topology information.

8. A measure of CPU processing capacity being used over time.

9. No. High utilization may represent useful expected work.

10. A workload whose performance is primarily limited by CPU processing capacity.

11. Random Access Memory.

12. Fast temporary storage used by running programs and the operating system.

13. To make operations such as repeated file access faster and use otherwise idle memory productively.

14. No.

15. Yes.

16. Memory and swap usage information.

17. It displays sizes in human-readable units.

18. The amount of physical memory available to Linux according to its accounting.

19. Memory currently completely unused.

20. An estimate of memory that can be made available for new workloads without substantial swapping.

21. Linux can reclaim suitable cached memory, so completely unused RAM alone does not describe usable capacity.

22. Disk-backed space used as part of virtual memory management.

23. No.

24. It can move less-active memory pages out of RAM and provide flexibility during memory management.

25. No.

26. A condition where the system is relying significantly on swap because RAM is constrained.

27. Excessive movement of actively needed pages between RAM and storage, causing severe performance degradation.

28. It shows active swap devices or files.

29. A kernel parameter influencing swap-related memory-reclaim behavior.

30. No. Workload and system design matter.

31. Current time, system uptime, logged-in users, and load averages.

32. A measure of average runnable and certain uninterruptibly waiting tasks over time.

33. Approximately 1 minute, 5 minutes, and 15 minutes.

34. Recent approximately one-minute load.

35. Approximately five-minute load.

36. Approximately fifteen-minute load.

37. The same load value means different things on systems with different CPU capacities.

38. No.

39. Yes.

40. It provides an interactive real-time view of system and process activity.

41. User-space CPU time.

42. Kernel/system CPU time.

43. Idle CPU time.

44. I/O wait.

45. Steal time in virtualized environments.

46. CPU usage attributed to the process.

47. The process's physical-memory usage as a percentage according to the tool.

48. Memory currently resident in physical RAM.

49. The process's virtual address space.

50. Virtual address space is not the same as physical RAM currently consumed.

51. Running or runnable.

52. Interruptible sleeping.

53. Uninterruptible sleep, commonly associated with waiting for certain kernel operations such as I/O.

54. Zombie.

55. An interactive process and resource viewer.

56. `top` is widely available and commonly present on minimal Linux servers.

57. For example, `ps aux --sort=-%cpu | head`.

58. For example, `ps aux --sort=-%mem | head`.

59. Identify what it is, why it is consuming CPU, whether the behavior is expected, and the effect of terminating it.

60. Application bugs, legitimate workload growth, runaway scripts, compilation, encryption, excessive retries, or expensive computation.

61. A condition where memory demand is placing significant pressure on available physical memory and reclaim mechanisms.

62. A condition where an application retains memory it no longer needs, causing inappropriate memory growth.

63. No.

64. They reveal trends and distinguish sustained growth from normal temporary or stable usage.

65. Out Of Memory.

66. A Linux kernel mechanism that may terminate processes during severe memory exhaustion to recover memory.

67. To recover memory so the system can continue operating.

68. Kernel logs through tools such as `journalctl -k` or `dmesg`.

69. No.

70. It does not fix memory leaks, bad sizing, excessive workers, or continuously increasing memory demand.

71. Process, memory, paging, I/O, system, and CPU statistics.

72. Runnable processes.

73. Processes blocked in certain waits.

74. Swap-in activity.

75. Swap-out activity.

76. CPU time spent idle while waiting for I/O.

77. One observation may be transient, and the first line may represent statistics accumulated since boot.

78. Input and output operations involving storage.

79. No.

80. Input/Output Operations Per Second.

81. The amount of data transferred over time.

82. The time required for storage operations to be serviced.

83. Applications may spend time waiting for storage operations to finish.

84. Examining CPU and block-device I/O statistics.

85. Reads per second.

86. Writes per second.

87. Average time associated with servicing I/O requests in common `iostat` output.

88. No. Modern storage architectures require more careful interpretation.

89. The resource or component limiting system performance.

90. How busy a resource is.

91. A condition where demand reaches or exceeds the resource's ability to service work efficiently, causing work to queue or wait.

92. Saturation often directly contributes to increasing latency and degraded performance.

93. Errors may reveal resource or component failures that percentages alone cannot show.

94. Define exactly what is slow and determine the scope of the problem.

95. A deployment, configuration change, traffic increase, scheduled job, or infrastructure change may explain the beginning of the problem.

96. They show what normal behavior looks like and make abnormal behavior easier to recognize.

97. It can remove evidence, reset counters, terminate processes, clear transient state, and temporarily hide the root cause.

98. `SIGKILL` prevents normal process cleanup and high CPU alone does not establish that the process should be forcibly terminated.

99. Resource recovery, application recovery, latency, error rate, and whether the original symptom is actually resolved.

100. To identify the real limiting resource or underlying cause using evidence, correct it safely, verify recovery, and reduce the chance of recurrence.

# 174. Final Performance Mental Model

Everything in Part 15 should now fit together:

```text
                    USERS
                      |
                      v
                 APPLICATION
                      |
                      v
                   PROCESS
                      |
          +-----------+-----------+
          |           |           |
          v           v           v
         CPU         RAM        DISK I/O
          |           |           |
          |           v           |
          |          SWAP         |
          |                       |
          +-----------+-----------+
                      |
                      v
                 PERFORMANCE
```

When the server becomes slow, do not begin with a conclusion.

Begin with questions.

```text
Slow server
     |
     v
What exactly is slow?
     |
     v
What changed?
     |
     v
What resources are under pressure?
```

For CPU:

```text
How many CPUs?
     |
     v
nproc / lscpu
     |
     v
What is the load?
     |
     v
uptime
     |
     v
Is CPU actually busy?
     |
     v
top / vmstat
     |
     v
Which process?
     |
     v
ps / top
```

For memory:

```text
How much RAM exists?
     |
     v
free -h
     |
     v
How much is available?
     |
     v
Is swap being used?
     |
     v
Is swap actively moving?
     |
     v
vmstat
     |
     v
Which process uses memory?
     |
     v
top / ps
     |
     v
Any OOM evidence?
     |
     v
journalctl -k / dmesg
```

For disk I/O:

```text
Server slow
    |
    v
CPU not obviously saturated
    |
    v
High load / waiting processes?
    |
    v
Check I/O wait
    |
    v
vmstat / top
    |
    v
Inspect storage activity
    |
    v
iostat
    |
    v
Check latency, queueing, workload
```

And never forget storage capacity:

```text
df -h
```

and inode capacity:

```text
df -i
```

The central troubleshooting habit is:

```text
SYMPTOM
   |
   v
OBSERVE
   |
   v
MEASURE
   |
   v
COMPARE
   |
   v
CORRELATE
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
REMEDIATE
   |
   v
VERIFY
   |
   v
PREVENT
```

A beginner sees:

```text
Load average = 20
```

and says:

```text
CPU problem.
```

You should ask:

```text
How many CPUs?

What is CPU utilization?

What is I/O wait?

How many tasks are runnable?

Are processes blocked?

Which processes are involved?
```

A beginner sees:

```text
Free RAM = 200 MB
```

and says:

```text
We need more RAM.
```

You should ask:

```text
What is available memory?

How much is cache?

Is swap active?

Are si and so persistently elevated?

Which processes consume memory?

Are there OOM events?

What does the historical trend show?
```

A beginner sees:

```text
Swap used = 1 GB
```

and says:

```text
The server has a memory problem.
```

You should ask:

```text
Is swapping happening now?

Is available RAM low?

Is application performance affected?

Is swap usage growing?
```

A beginner sees:

```text
Server slow.
```

and restarts it.

You should think:

```text
What evidence can I collect first?
```

That difference, moving from reacting to symptoms toward measuring, forming hypotheses, testing them, and identifying root causes, is the foundation of Linux performance troubleshooting.
