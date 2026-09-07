# Part 7: Linux Pipes, Redirection, and Command Chaining

In Part 6, you learned how to view, edit, search, filter, and transform text using commands such as `cat`, `less`, `head`, `tail`, `grep`, `wc`, `sort`, `uniq`, `cut`, `tr`, `sed`, and `awk`.

Now you will learn how Linux connects commands together.

This is where individual commands begin to become powerful workflows.

Instead of running one command, reading its output, copying something manually, and running another command, the shell can connect those operations for you.

For example:

```bash
grep "ERROR" application.log | cut -d ' ' -f 3 | sort | uniq -c | sort -nr
```

A beginner may look at this and see one complicated command.

An experienced Linux user sees a sequence:

```text
Read relevant log lines
        |
        v
Extract one field
        |
        v
Sort identical values together
        |
        v
Count them
        |
        v
Sort by frequency
```

This part will teach you how to build that reasoning from the beginning.

By the end, you should understand:

* Standard input.
* Standard output.
* Standard error.
* File descriptors.
* Output and input redirection.
* Error redirection.
* `/dev/null`.
* Pipes.
* `tee`.
* `xargs`.
* Command chaining.
* Conditional command execution.
* Shell quoting.
* Escaping.
* Command substitution.
* Basic shell expansion.
* How to build pipelines gradually.
* How to troubleshoot broken pipelines.
* How engineers use pipelines to investigate application logs.

---

# 1. The Core Idea: Commands Consume Input and Produce Output

Many Linux commands follow a simple model:

```text
Input
  |
  v
Command
  |
  v
Output
```

Consider:

```bash
grep "ERROR" application.log
```

`grep`:

1. Reads data.
2. Searches for `ERROR`.
3. Produces matching lines.

That output normally appears in your terminal.

But it does not have to.

You can:

* Send it to another command.
* Save it in a file.
* Append it to a file.
* Store it in a variable.
* Send it somewhere else.

Understanding this data flow is fundamental to shell usage.

---

# 2. Standard Streams

Linux processes commonly work with three standard streams:

```text
Standard input
Standard output
Standard error
```

They are usually abbreviated:

```text
stdin
stdout
stderr
```

Think of them as channels through which information flows between a program and its environment.

---

# 3. Standard Input

Standard input, or `stdin`, is the default input stream.

A program can read data from it.

For an interactive command, standard input often comes from your keyboard.

For example:

```bash
cat
```

Run it without a filename.

Then type:

```text
Linux is running
```

Press Enter.

You may see:

```text
Linux is running
Linux is running
```

Why?

`cat` is reading what you type from standard input and writing it to standard output.

Stop it with:

```text
Ctrl+C
```

Conceptually:

```text
Keyboard
   |
   v
stdin
   |
   v
 cat
   |
   v
stdout
   |
   v
Terminal
```

---

# 4. Standard Output

Standard output, or `stdout`, is the normal output stream of a program.

Run:

```bash
echo "Linux"
```

Output:

```text
Linux
```

Conceptually:

```text
echo
 |
 v
stdout
 |
 v
Terminal
```

The terminal is normally where standard output goes when you have not redirected it elsewhere.

---

# 5. Standard Error

Standard error, or `stderr`, is a separate output stream intended for errors and diagnostic messages.

Try:

```bash
ls /directory-that-does-not-exist
```

You may see:

```text
ls: cannot access '/directory-that-does-not-exist': No such file or directory
```

This message is normally written to standard error.

Why separate errors from normal output?

Because sometimes you want to process successful output while treating errors differently.

For example:

```text
Command
   |
   +------> stdout ------> useful data
   |
   +------> stderr ------> error messages
```

That separation becomes extremely useful in scripts and automation.

---

# 6. Why stdout and stderr Need to Be Separate

Imagine a command searches thousands of files.

It successfully finds:

```text
/etc/app/app.conf
/etc/app/database.conf
```

but also encounters:

```text
Permission denied
Permission denied
```

If normal results and errors were impossible to distinguish, automated processing would become much harder.

Linux gives us separate streams so we can decide:

```text
Save successful results here.

Save errors somewhere else.

Show errors on screen.

Discard errors.

Combine both.

Send only successful results into another command.
```

---

# 7. File Descriptors

Linux represents open input/output resources using numbers called file descriptors.

The three standard streams have conventional file descriptor numbers:

```text
0 = standard input
1 = standard output
2 = standard error
```

Remember:

```text
0 -> stdin
1 -> stdout
2 -> stderr
```

A useful diagram:

```text
           Process
              |
     +--------+--------+
     |        |        |
     v        v        v
     0        1        2
   stdin    stdout   stderr
```

These numbers explain syntax such as:

```bash
2> errors.log
```

The `2` means:

```text
Redirect file descriptor 2.
```

File descriptor 2 is standard error.

---

# 8. Do You Need to Memorize File Descriptors?

Yes, these three are worth remembering:

```text
0 = input
1 = normal output
2 = errors
```

You will see them constantly in:

* Bash commands.
* Shell scripts.
* Cron jobs.
* CI/CD pipelines.
* Server administration.
* Troubleshooting.
* Automation.

---

# 9. Create a Safe Practice Environment

Create:

```bash
mkdir -p ~/pipeline-lab/{logs,output,data,errors}
```

Move into it:

```bash
cd ~/pipeline-lab
```

Verify:

```bash
pwd
```

Then:

```bash
ls
```

You should see:

```text
data  errors  logs  output
```

We will use this directory throughout the article.

---

# 10. Redirecting Standard Output With `>`

You already saw `>` briefly in Part 6.

Now we will understand what it actually does.

Run:

```bash
echo "production" > output/environment.txt
```

Nothing may appear on your terminal.

Why?

Normally:

```text
echo
 |
 v
stdout
 |
 v
Terminal
```

But redirection changes the destination:

```text
echo
 |
 v
stdout
 |
 v
output/environment.txt
```

Check:

```bash
cat output/environment.txt
```

Output:

```text
production
```

---

# 11. `>` Redirects Standard Output

The full conceptual meaning is:

```text
Take standard output and send it into this file.
```

Because stdout is file descriptor `1`, you could explicitly write:

```bash
echo "production" 1> output/environment.txt
```

This works because:

```text
1 = stdout
```

Usually, the `1` is omitted.

Therefore:

```bash
command > file
```

is effectively:

```bash
command 1> file
```

---

# 12. `>` Overwrites Existing Contents

This is extremely important.

Run:

```bash
echo "production" > output/environment.txt
```

Then:

```bash
echo "staging" > output/environment.txt
```

Now:

```bash
cat output/environment.txt
```

Output:

```text
staging
```

`production` disappeared.

Why?

`>` normally truncates an existing target file before writing the new output.

Think:

```text
>
Replace file contents with redirected output
```

---

# 13. Why `>` Can Be Dangerous

Imagine:

```bash
some-command > /etc/important.conf
```

If `/etc/important.conf` already contains critical configuration and the command is allowed to write there, its previous contents may be destroyed.

Before using `>` ask:

```text
What is the destination?

Does it already exist?

Do I intend to replace it?

Should I append instead?

Do I need a backup?
```

Do not blindly copy redirection commands on production systems.

---

# 14. Appending Standard Output With `>>`

`>>` appends output instead of replacing existing contents.

Run:

```bash
echo "server=web01" > output/servers.txt
```

Then:

```bash
echo "server=web02" >> output/servers.txt
```

Then:

```bash
echo "server=web03" >> output/servers.txt
```

Check:

```bash
cat output/servers.txt
```

Output:

```text
server=web01
server=web02
server=web03
```

Think:

```text
>   overwrite

>>  append
```

---

# 15. What Happens if the File Does Not Exist?

Both:

```bash
>
```

and:

```bash
>>
```

normally create the target file if it does not already exist, assuming you have permission to create it in that directory.

The difference matters when the file already exists.

---

# 16. Input Redirection With `<`

The:

```text
<
```

operator redirects a file into standard input.

Create:

```bash
printf '%s\n' \
'production' \
'staging' \
'development' \
> data/environments.txt
```

Normally you could run:

```bash
cat data/environments.txt
```

But you can also use:

```bash
cat < data/environments.txt
```

Conceptually:

```text
data/environments.txt
        |
        v
      stdin
        |
        v
       cat
        |
        v
      stdout
        |
        v
     Terminal
```

---

# 17. Why Use `<`?

Some commands naturally accept filenames, so input redirection may look unnecessary.

For example:

```bash
wc -l data/environments.txt
```

works.

But:

```bash
wc -l < data/environments.txt
```

also works.

There is an interesting difference.

The first may output:

```text
3 data/environments.txt
```

The second may output:

```text
3
```

Why?

With input redirection, `wc` receives data through standard input. It does not need to know the filename.

---

# 18. Input Redirection Mental Model

Remember:

```text
>
command output goes to file

<
file contents go to command input
```

Direction matters.

```text
command > file

file < command
```

is not the mental model.

Think about the arrows as data flow:

```text
command > file

file -> command
      represented as
command < file
```

---

# 19. Redirecting Standard Error With `2>`

Now use the file descriptor knowledge.

Run:

```bash
ls /does-not-exist
```

The error appears in your terminal.

Now:

```bash
ls /does-not-exist 2> errors/ls-errors.log
```

The error may disappear from your screen.

Check:

```bash
cat errors/ls-errors.log
```

You should see the error.

Why?

```text
2 = stderr
```

Therefore:

```text
2>
```

means:

```text
Redirect standard error into a file.
```

---

# 20. `2>` Overwrites the Error File

Just as:

```text
>
```

overwrites stdout destination contents,

```text
2>
```

normally overwrites the stderr destination.

Run:

```bash
ls /missing-one 2> errors/test.log
```

Then:

```bash
ls /missing-two 2> errors/test.log
```

Check:

```bash
cat errors/test.log
```

You will normally see only the most recent error.

---

# 21. Appending Errors With `2>>`

Use:

```bash
ls /missing-one 2> errors/test.log
```

Then:

```bash
ls /missing-two 2>> errors/test.log
```

Then:

```bash
ls /missing-three 2>> errors/test.log
```

Check:

```bash
cat errors/test.log
```

Now all errors should be present.

Remember:

```text
2>   overwrite error file

2>>  append to error file
```

---

# 22. Separating stdout and stderr

Create:

```bash
mkdir -p data/search
touch data/search/app.conf
touch data/search/database.conf
```

Now run a command that produces both successful and failed output:

```bash
ls data/search /missing-directory
```

You may see:

```text
ls: cannot access '/missing-directory': No such file or directory

data/search:
app.conf
database.conf
```

One part is normal output.

One part is error output.

They are separate streams even though your terminal displays both.

---

# 23. Redirect stdout and stderr to Different Files

Run:

```bash
ls data/search /missing-directory \
> output/ls-output.txt \
2> errors/ls-errors.txt
```

Now:

```bash
cat output/ls-output.txt
```

contains successful output.

And:

```bash
cat errors/ls-errors.txt
```

contains the error.

Conceptually:

```text
                ls
                 |
        +--------+--------+
        |                 |
        v                 v
      stdout            stderr
        |                 |
        v                 v
ls-output.txt       ls-errors.txt
```

---

# 24. Redirecting stdout and stderr Together

Sometimes you want both streams in the same file.

A common Bash form is:

```bash
command > combined.log 2>&1
```

Example:

```bash
ls data/search /missing-directory > output/combined.log 2>&1
```

Check:

```bash
cat output/combined.log
```

Both normal output and errors should be present.

---

# 25. Understanding `2>&1`

This syntax looks strange until you break it apart.

```text
2
stderr

>
redirect

&1
to the current destination of file descriptor 1
```

So:

```bash
2>&1
```

means:

```text
Send stderr to wherever stdout is currently going.
```

With:

```bash
command > combined.log 2>&1
```

the shell first sends stdout to:

```text
combined.log
```

Then sends stderr to the same place stdout is going.

---

# 26. Redirection Order Matters

Compare:

```bash
command > file 2>&1
```

with:

```bash
command 2>&1 > file
```

These are not necessarily equivalent.

Why?

Shell redirections are processed from left to right.

In:

```bash
command > file 2>&1
```

first:

```text
stdout -> file
```

then:

```text
stderr -> current stdout destination
       -> file
```

Both go to the file.

But:

```bash
command 2>&1 > file
```

first sends stderr to wherever stdout is currently pointing, commonly the terminal.

Then stdout is redirected to the file.

The earlier stderr redirection does not automatically follow the later stdout change.

This is an important shell concept.

---

# 27. Bash Shortcut: `&>`

Bash also supports:

```bash
command &> file
```

to redirect both stdout and stderr.

For example:

```bash
ls data/search /missing-directory &> output/all-output.txt
```

However, you should understand:

```bash
> file 2>&1
```

because it is widely encountered and teaches how file descriptors work.

---

# 28. `/dev/null`

Linux has a special device:

```text
/dev/null
```

Data written to `/dev/null` is discarded.

You can think of it as a data sink.

```text
Output
  |
  v
/dev/null
  |
  v
discarded
```

---

# 29. Discard Standard Output

Run:

```bash
echo "You will not see this" > /dev/null
```

Nothing appears.

The output was discarded.

---

# 30. Discard Standard Error

Run:

```bash
ls /does-not-exist 2> /dev/null
```

The error is suppressed.

This can be useful when an error is expected and genuinely irrelevant.

But use it carefully.

---

# 31. Why `/dev/null` Can Be Dangerous for Troubleshooting

Suppose an application fails and your script contains:

```bash
some-command 2> /dev/null
```

You may have thrown away the exact error message needed to diagnose the problem.

Do not use `/dev/null` merely to make errors disappear.

Ask:

```text
Is this error genuinely expected?

Do I need to log it instead?

Will hiding it make troubleshooting harder?
```

Suppressing evidence is not the same as fixing a problem.

---

# 32. Discard Both stdout and stderr

You may encounter:

```bash
command > /dev/null 2>&1
```

This sends both normal output and errors to `/dev/null`.

In Bash:

```bash
command &> /dev/null
```

can also do this.

Again, use this intentionally.

---

# 33. Pipes With `|`

The pipe operator is:

```text
|
```

A pipe connects the standard output of one command to the standard input of another.

Example:

```bash
cat data/environments.txt | sort
```

Conceptually:

```text
data/environments.txt
       |
       v
      cat
       |
       | stdout
       v
       |
       | pipe
       v
      sort
       |
       v
    terminal
```

The output of `cat` does not need to appear on screen first.

It becomes input for `sort`.

---

# 34. Why Pipes Matter

Without pipes, you might do this:

```bash
command1 > temporary.txt
command2 < temporary.txt
rm temporary.txt
```

A pipe lets you write:

```bash
command1 | command2
```

This is:

* Faster to write.
* Easier to automate.
* Often easier to reason about.
* Less dependent on temporary files.
* Powerful for data processing.

---

# 35. Avoid Unnecessary `cat`

You may see:

```bash
cat application.log | grep "ERROR"
```

It works.

But `grep` can read the file directly:

```bash
grep "ERROR" application.log
```

The second is simpler.

However, pipes become essential when the previous command generates data dynamically:

```bash
some-command | grep "ERROR"
```

The important lesson is not "never use `cat` in a pipe."

It is:

```text
Understand whether each command contributes something useful.
```

---

# 36. A Simple Pipeline

Create:

```bash
printf '%s\n' \
'production' \
'staging' \
'production' \
'development' \
'production' \
'staging' \
> data/environments.txt
```

Run:

```bash
sort data/environments.txt | uniq
```

Output:

```text
development
production
staging
```

First:

```text
sort
```

groups identical lines.

Then:

```text
uniq
```

removes adjacent duplicates.

---

# 37. Add Counting

Run:

```bash
sort data/environments.txt | uniq -c
```

Possible output:

```text
1 development
3 production
2 staging
```

Now another command has been added to the flow.

---

# 38. Sort the Counts

Run:

```bash
sort data/environments.txt | uniq -c | sort -nr
```

Possible output:

```text
3 production
2 staging
1 development
```

Understand each stage:

```text
Raw environments
       |
       v
sort
Group identical values
       |
       v
uniq -c
Count each group
       |
       v
sort -nr
Sort counts numerically, highest first
```

---

# 39. Pipes and stderr

A normal pipe:

```bash
command1 | command2
```

connects `command1`'s stdout to `command2`'s stdin.

Its stderr is not automatically part of that pipe.

This matters.

If:

```bash
command1
```

produces errors, they may still appear directly in your terminal.

That distinction becomes important in troubleshooting.

---

# 40. `tee`

Sometimes you want to:

1. See output on screen.
2. Save the same output in a file.
3. Possibly continue sending it through a pipeline.

`tee` is designed for this.

Example:

```bash
echo "deployment successful" | tee output/deployment.log
```

You see:

```text
deployment successful
```

and the same text is written to:

```text
output/deployment.log
```

---

# 41. Why Is It Called `tee`?

Think of a T-shaped pipe:

```text
             +----> file
             |
Input ------ tee
             |
             +----> stdout
```

The input is split into multiple directions.

---

# 42. `tee` in a Pipeline

Create:

```bash
printf '%s\n' \
'ERROR database timeout' \
'INFO request completed' \
'ERROR API unavailable' \
> logs/application.log
```

Run:

```bash
grep "ERROR" logs/application.log \
| tee output/errors.txt \
| wc -l
```

Possible terminal output:

```text
2
```

Meanwhile:

```bash
cat output/errors.txt
```

contains:

```text
ERROR database timeout
ERROR API unavailable
```

Flow:

```text
application.log
      |
      v
grep ERROR
      |
      v
     tee ------> errors.txt
      |
      v
    wc -l
      |
      v
      2
```

---

# 43. Appending With `tee -a`

Normal `tee` overwrites the destination file.

Use:

```bash
tee -a FILE
```

to append.

Example:

```bash
echo "new deployment" | tee -a output/deployment.log
```

The:

```text
-a
```

means append.

---

# 44. `tee` and Privileged Files

You may eventually encounter a situation like:

```bash
sudo echo "setting=value" > /etc/example.conf
```

and receive permission problems.

Why can this happen?

Because the shell performs `>` redirection before `sudo` runs `echo`.

`sudo` applies to `echo`, not automatically to the shell process opening the destination file.

A pattern you may encounter is:

```bash
echo "setting=value" | sudo tee /etc/example.conf
```

Here `tee` itself runs with the requested privilege.

Be very careful.

This can overwrite a system configuration file.

For appending:

```bash
echo "setting=value" | sudo tee -a /etc/example.conf
```

Do not practice this against real system configuration unless the change is intentional and authorized.

---

# 45. `xargs`

`xargs` takes data from standard input and uses it to construct command arguments.

This distinction matters because not every command naturally reads the information you want from stdin.

Basic idea:

```text
Input lines
   |
   v
xargs
   |
   v
Build command arguments
   |
   v
Run command
```

---

# 46. Simple `xargs` Example

Run:

```bash
printf '%s\n' one two three | xargs echo
```

Output:

```text
one two three
```

`xargs` collected the input and supplied it as arguments to `echo`.

Conceptually:

```text
stdin:
one
two
three

        |
        v

xargs echo

        |
        v

echo one two three
```

---

# 47. Why `xargs` Exists

Some commands expect:

```text
command ARGUMENT
```

rather than consuming arbitrary stdin as the target argument.

Suppose another command produces filenames.

You may need to transform:

```text
file1
file2
file3
```

into something conceptually like:

```bash
command file1 file2 file3
```

`xargs` helps bridge that gap.

---

# 48. `xargs` With One Item at a Time

Run:

```bash
printf '%s\n' web01 web02 db01 | xargs -n 1 echo "Checking"
```

Possible output:

```text
Checking web01
Checking web02
Checking db01
```

The:

```text
-n 1
```

tells `xargs` to use one input item per command invocation.

---

# 49. Using a Placeholder With `xargs -I`

Example:

```bash
printf '%s\n' web01 web02 db01 \
| xargs -I {} echo "Checking server {}"
```

Output:

```text
Checking server web01
Checking server web02
Checking server db01
```

Here:

```text
{}
```

is the replacement placeholder.

For every input line, `xargs` substitutes that value into the command.

---

# 50. `xargs` Requires Care With Filenames

This is important.

Filenames can contain:

* Spaces.
* Tabs.
* Newlines.
* Quotes.
* Other special characters.

A naive command such as:

```bash
find . -name '*.log' | xargs rm
```

can behave incorrectly with unusual filenames.

It is also destructive.

Do not practice deletion this way.

When working with `find` and `xargs`, you may encounter the safer null-delimited pattern:

```bash
find . -name '*.log' -print0 | xargs -0 command
```

Here:

```text
-print0
```

uses a null character to separate results.

And:

```text
xargs -0
```

expects null-separated input.

You do not need to master this yet.

Remember:

```text
xargs can turn input into command arguments, but quoting and filename boundaries matter.
```

---

# 51. Combining `grep`, `sort`, `uniq`, `cut`, `awk`, and `wc`

These commands become much more useful when connected.

We will build several pipelines gradually.

Create:

```bash
printf '%s\n' \
'2026-08-26 10:00:01 INFO web01 request=1001 status=200' \
'2026-08-26 10:00:02 INFO web02 request=1002 status=200' \
'2026-08-26 10:00:03 ERROR web01 request=1003 status=500' \
'2026-08-26 10:00:04 WARN web03 request=1004 status=429' \
'2026-08-26 10:00:05 ERROR web02 request=1005 status=502' \
'2026-08-26 10:00:06 INFO web01 request=1006 status=200' \
'2026-08-26 10:00:07 ERROR web01 request=1007 status=500' \
'2026-08-26 10:00:08 INFO web03 request=1008 status=200' \
'2026-08-26 10:00:09 WARN web02 request=1009 status=429' \
'2026-08-26 10:00:10 ERROR web03 request=1010 status=503' \
> logs/api.log
```

---

# 52. First Question: How Many Lines?

Use:

```bash
wc -l logs/api.log
```

Expected:

```text
10 logs/api.log
```

This establishes the size of our sample.

---

# 53. How Many Error Lines?

Use:

```bash
grep "ERROR" logs/api.log | wc -l
```

Expected:

```text
4
```

Flow:

```text
api.log
  |
  v
grep ERROR
  |
  v
4 matching lines
  |
  v
wc -l
  |
  v
4
```

You could also use:

```bash
grep -c "ERROR" logs/api.log
```

But the pipeline helps demonstrate stream composition.

---

# 54. Which Servers Produced Errors?

First inspect error lines:

```bash
grep "ERROR" logs/api.log
```

The fields are:

```text
date time level server request status
```

Therefore the server is field 4.

Extract it:

```bash
grep "ERROR" logs/api.log | awk '{print $4}'
```

Output:

```text
web01
web02
web01
web03
```

---

# 55. Count Errors by Server

Continue:

```bash
grep "ERROR" logs/api.log \
| awk '{print $4}' \
| sort \
| uniq -c
```

Output:

```text
2 web01
1 web02
1 web03
```

Now sort highest first:

```bash
grep "ERROR" logs/api.log \
| awk '{print $4}' \
| sort \
| uniq -c \
| sort -nr
```

Now the server with the most error lines appears first.

---

# 56. Extract HTTP Status Codes

Inspect:

```text
status=500
```

The status is field 6.

Use:

```bash
awk '{print $6}' logs/api.log
```

Output includes:

```text
status=200
status=200
status=500
...
```

Now remove `status=`:

```bash
awk '{print $6}' logs/api.log | cut -d '=' -f 2
```

Output:

```text
200
200
500
429
502
200
500
200
429
503
```

---

# 57. Count Status Codes

Use:

```bash
awk '{print $6}' logs/api.log \
| cut -d '=' -f 2 \
| sort \
| uniq -c \
| sort -nr
```

Possible output:

```text
4 200
2 500
2 429
1 503
1 502
```

You have transformed raw application logs into a useful operational summary.

---

# 58. Why Pipeline Order Matters

Compare:

```bash
sort file | uniq -c
```

with:

```bash
uniq -c file | sort
```

They are not necessarily equivalent.

`uniq` only counts adjacent identical lines.

Therefore:

```text
sort
  |
  v
uniq
```

is often necessary when identical records are scattered through a file.

Pipeline order represents logic.

Changing the order can change the answer.

---

# 59. Command Chaining With `;`

A semicolon lets you put multiple commands on one shell line.

Example:

```bash
pwd; whoami; date
```

The shell runs:

```text
pwd
then
whoami
then
date
```

The commands run sequentially.

Importantly, later commands generally run even if an earlier command fails.

---

# 60. Demonstrating `;`

Run:

```bash
echo "first"; ls /missing-directory; echo "third"
```

You should see:

1. `first`.
2. An error from `ls`.
3. `third`.

The failure did not prevent the third command from running.

Think:

```text
command1 ; command2
```

means approximately:

```text
Run command1.
Then run command2 regardless of command1's success status.
```

---

# 61. Exit Status

To understand `&&` and `||`, you need one new concept.

When a command finishes, it returns an exit status.

Conventionally:

```text
0 = success

non-zero = some kind of failure or special condition
```

You can inspect the most recent command's exit status using:

```bash
echo $?
```

Example:

```bash
true
echo $?
```

Usually:

```text
0
```

Now:

```bash
false
echo $?
```

Usually:

```text
1
```

---

# 62. Why Exit Status Matters

The shell can use exit status to make decisions.

Conceptually:

```text
Run command
    |
    v
Exit status
    |
 +--+--+
 |     |
 0   non-zero
 |     |
 v     v
success failure
```

This is the foundation for conditional command execution.

---

# 63. Conditional Execution With `&&`

The:

```text
&&
```

operator runs the next command only if the previous command succeeds.

Example:

```bash
mkdir test-directory && cd test-directory
```

Meaning:

```text
Create directory.

If that succeeds,
enter it.
```

This is safer than:

```bash
mkdir test-directory; cd test-directory
```

when the second action only makes sense if the first succeeded.

---

# 64. Demonstrating `&&`

Run:

```bash
echo "first" && echo "second"
```

Both should run.

Now:

```bash
ls /missing-directory && echo "Directory exists"
```

The `ls` command fails.

Therefore:

```text
Directory exists
```

should not be printed.

---

# 65. Real-World Use of `&&`

Suppose you need to create a project directory and only continue if creation succeeds:

```bash
mkdir ~/project && cd ~/project
```

Or:

```bash
some-validation-command && some-deployment-command
```

Conceptually:

```text
Validate
   |
   | success
   v
Deploy
```

This can prevent a later action from running after an earlier failure.

However, production deployment logic should generally be more explicit and robust than blindly chaining commands.

---

# 66. Conditional Execution With `||`

The:

```text
||
```

operator runs the next command if the previous command fails.

Example:

```bash
ls /missing-directory || echo "Directory not found"
```

Since `ls` fails, the second command runs.

Output may include:

```text
ls: cannot access '/missing-directory': No such file or directory
Directory not found
```

---

# 67. `&&` vs `||`

Remember:

```text
command1 && command2

Run command2 if command1 succeeds.
```

```text
command1 || command2

Run command2 if command1 fails.
```

A useful mental model:

```text
             command1
                |
        +-------+-------+
        |               |
     success          failure
        |               |
       &&              ||
        |               |
        v               v
 next command      next command
```

---

# 68. Combining `&&` and `||`

You may encounter:

```bash
command && echo "Success" || echo "Failed"
```

For a simple command, this may appear to act like:

```text
If command succeeds:
    print Success

Otherwise:
    print Failed
```

But shell chaining has subtleties because each command has its own exit status.

Do not treat `&& ... || ...` as a perfect replacement for a proper `if` statement in all cases.

You will learn shell scripting and conditionals later.

For now, use simple chains where their behavior is clear.

---

# 69. `;` vs `&&` vs `||`

Remember:

```text
;

Run the next command regardless.
```

```text
&&

Run the next command if the previous command succeeds.
```

```text
||

Run the next command if the previous command fails.
```

Example:

```bash
command1; command2
```

Always proceeds to `command2`.

```bash
command1 && command2
```

Proceeds only after success.

```bash
command1 || command2
```

Proceeds only after failure.

---

# 70. Single Quotes

Shell quoting is critical.

Single quotes are:

```text
' '
```

Text inside single quotes is generally treated literally by Bash.

Example:

```bash
echo '$HOME'
```

Output:

```text
$HOME
```

The shell does not expand `$HOME`.

---

# 71. Double Quotes

Double quotes are:

```text
" "
```

They preserve the text as one argument while still allowing certain expansions.

Example:

```bash
echo "$HOME"
```

You may see:

```text
/home/yourusername
```

The shell expanded the variable.

Compare:

```bash
echo '$HOME'
```

Output:

```text
$HOME
```

---

# 72. Why Quoting Matters

Suppose:

```bash
name="Linux Engineer"
```

Run:

```bash
printf '<%s>\n' $name
```

The shell may split the expanded value into separate words:

```text
<Linux>
<Engineer>
```

Now:

```bash
printf '<%s>\n' "$name"
```

Output:

```text
<Linux Engineer>
```

The double quotes preserve the expanded value as one argument.

This is one reason proper quoting is essential in shell scripting.

---

# 73. Single vs Double Quotes

A beginner rule:

```text
'text'
Treat contents literally.

"text"
Keep the contents together, but allow expansions such as $VARIABLE and $(command).
```

Examples:

```bash
echo '$USER'
```

prints:

```text
$USER
```

while:

```bash
echo "$USER"
```

prints your username.

---

# 74. Escaping Characters

A backslash:

```text
\
```

can remove the special meaning of the next character in many shell contexts.

Example:

```bash
echo \$HOME
```

Output:

```text
$HOME
```

Without the backslash:

```bash
echo $HOME
```

the shell expands the variable.

---

# 75. Escaping Spaces

Suppose you have:

```text
My Project
```

as a directory name.

You could write:

```bash
cd My\ Project
```

The backslash escapes the space.

Or, usually more readably:

```bash
cd "My Project"
```

Quoting is often easier when paths contain spaces.

---

# 76. Escaping Double Quotes

Suppose you want output:

```text
The server is "healthy"
```

You can use:

```bash
echo "The server is \"healthy\""
```

The backslashes prevent the inner double quotes from terminating the outer quoted string.

Alternatively:

```bash
echo 'The server is "healthy"'
```

works because double quotes are ordinary characters inside single quotes.

---

# 77. Why Shell Quoting Is a Security Issue

Incorrect quoting can cause:

* Unexpected word splitting.
* Wildcard expansion.
* Incorrect filenames.
* Broken scripts.
* Unexpected arguments.
* Security problems when untrusted data is involved.

Do not think of quotes as cosmetic punctuation.

They change how the shell interprets commands.

---

# 78. Command Substitution

Command substitution allows the output of one command to become part of another command.

Modern shell syntax:

```bash
$(command)
```

Example:

```bash
echo "Today is $(date)"
```

The shell:

1. Runs `date`.
2. Captures its standard output.
3. Substitutes that output into the larger command.
4. Runs `echo`.

---

# 79. Understanding `$(command)`

Consider:

```bash
echo "User: $(whoami)"
```

Suppose:

```bash
whoami
```

returns:

```text
ann
```

The shell effectively constructs:

```bash
echo "User: ann"
```

Then executes it.

Output:

```text
User: ann
```

---

# 80. Store Command Output in a Variable

You can write:

```bash
current_user=$(whoami)
```

Then:

```bash
echo "$current_user"
```

Output:

```text
ann
```

Conceptually:

```text
whoami
  |
  v
stdout
  |
  v
command substitution
  |
  v
current_user
```

---

# 81. Useful Command Substitution Examples

Hostname:

```bash
server=$(hostname)
```

Date:

```bash
today=$(date +%F)
```

Current directory:

```bash
location=$(pwd)
```

Count errors:

```bash
error_count=$(grep -c "ERROR" logs/api.log)
```

Then:

```bash
echo "Errors found: $error_count"
```

---

# 82. Why `$(...)` Is Preferred

You may encounter older syntax using backticks:

```bash
result=`command`
```

Modern scripts generally prefer:

```bash
result=$(command)
```

because it is easier to read and nest.

For example:

```bash
echo "$(basename "$(pwd)")"
```

Nested backticks are much harder to understand.

---

# 83. Shell Expansion Basics

Before Linux executes a command, the shell performs several kinds of interpretation and expansion.

You have already encountered some of them.

Examples include:

* Variable expansion.
* Command substitution.
* Filename expansion.
* Tilde expansion.
* Word splitting in certain contexts.
* Quote processing.

You do not need to master Bash's complete expansion order yet.

But you must understand that the shell often transforms what you type before the target program receives it.

---

# 84. Variable Expansion

Suppose:

```bash
environment="production"
```

Then:

```bash
echo "$environment"
```

The shell expands:

```text
$environment
```

to:

```text
production
```

before `echo` receives it.

---

# 85. Tilde Expansion

You already know:

```text
~
```

represents your home directory in common shell contexts.

Example:

```bash
cd ~
```

The shell expands it to something such as:

```text
/home/ann
```

This is another shell expansion.

---

# 86. Filename Expansion

Suppose:

```text
logs/
├── api.log
├── database.log
└── nginx.log
```

Run:

```bash
ls logs/*.log
```

The shell expands:

```text
logs/*.log
```

into matching filenames before `ls` runs.

Conceptually:

```text
You type:

ls logs/*.log

Shell expands:

ls logs/api.log logs/database.log logs/nginx.log

Then ls runs.
```

The `*` is being handled by the shell.

This is shell globbing, not regex.

---

# 87. Why Quotes Affect Expansion

Suppose:

```bash
echo *
```

The shell may expand `*` into filenames in your current directory.

But:

```bash
echo '*'
```

prints:

```text
*
```

because single quotes prevent that expansion.

This illustrates why quoting changes command behavior.

---

# 88. Brace Expansion Introduction

Bash also supports brace expansion.

You used something similar when creating directories:

```bash
mkdir -p ~/pipeline-lab/{logs,output,data,errors}
```

The shell expands this into multiple paths:

```text
~/pipeline-lab/logs
~/pipeline-lab/output
~/pipeline-lab/data
~/pipeline-lab/errors
```

Another example:

```bash
echo server{1,2,3}
```

Output:

```text
server1 server2 server3
```

This is generated by the shell before `echo` runs.

---

# 89. Shell Expansion Can Cause Surprises

Suppose you accidentally run:

```bash
echo *
```

inside a directory containing hundreds of files.

The shell expands `*` into all matching filenames.

Similarly:

```bash
rm *
```

would pass those matching names to `rm`.

That is destructive.

Do not experiment with destructive wildcard commands against important directories.

Understand expansion before using it with commands that delete, move, change permissions, or modify resources.

---

# 90. Building Useful Command Pipelines

A good pipeline should answer a specific question.

Do not start by thinking:

```text
How many commands can I combine?
```

Start with:

```text
What question am I trying to answer?
```

Example:

```text
Which server generated the most ERROR log entries?
```

Now build the pipeline around that question.

---

# 91. Pipeline-Building Method

Use this sequence:

```text
1. Define the question
        |
        v
2. Inspect the input
        |
        v
3. Select relevant records
        |
        v
4. Extract relevant fields
        |
        v
5. Normalize if necessary
        |
        v
6. Sort/group
        |
        v
7. Count or summarize
        |
        v
8. Sort final result
        |
        v
9. Verify against raw data
```

This is much safer than writing a long pipeline from memory.

---

# 92. Practical Question: Which Server Has the Most Errors?

Start:

```bash
grep "ERROR" logs/api.log
```

Inspect the results.

Next:

```bash
grep "ERROR" logs/api.log | awk '{print $4}'
```

Inspect again.

Next:

```bash
grep "ERROR" logs/api.log | awk '{print $4}' | sort
```

Then:

```bash
grep "ERROR" logs/api.log \
| awk '{print $4}' \
| sort \
| uniq -c
```

Finally:

```bash
grep "ERROR" logs/api.log \
| awk '{print $4}' \
| sort \
| uniq -c \
| sort -nr
```

Now you have an answer.

---

# 93. Verify Your Pipeline

Do not automatically trust a pipeline because it produced output.

Suppose it says:

```text
2 web01
1 web03
1 web02
```

Return to:

```bash
grep "ERROR" logs/api.log
```

Manually inspect the small result set.

Confirm that `web01` really appears twice.

This is especially important while learning.

A syntactically valid pipeline can still answer the wrong question.

---

# 94. Practical Application-Log Analysis

Now we will build a more realistic dataset.

Create:

```bash
cat > logs/production-app.log <<'EOF'
2026-08-26T14:00:01 INFO service=api host=web01 status=200 duration=120ms
2026-08-26T14:00:02 INFO service=api host=web02 status=200 duration=98ms
2026-08-26T14:00:03 WARN service=api host=web01 status=200 duration=950ms
2026-08-26T14:00:04 ERROR service=api host=web03 status=500 duration=1300ms
2026-08-26T14:00:05 ERROR service=payments host=web02 status=502 duration=2100ms
2026-08-26T14:00:06 INFO service=api host=web01 status=200 duration=110ms
2026-08-26T14:00:07 ERROR service=payments host=web02 status=500 duration=1800ms
2026-08-26T14:00:08 INFO service=api host=web03 status=200 duration=105ms
2026-08-26T14:00:09 WARN service=payments host=web02 status=429 duration=870ms
2026-08-26T14:00:10 ERROR service=api host=web03 status=503 duration=2300ms
2026-08-26T14:00:11 INFO service=api host=web01 status=200 duration=100ms
2026-08-26T14:00:12 ERROR service=payments host=web02 status=502 duration=1900ms
EOF
```

The:

```text
<<'EOF'
```

syntax is called a here-document.

It allows multiple lines of input to be supplied to a command.

We will study shell scripting syntax in more depth later.

For this lab, it gives us a convenient log file.

---

# 95. Inspect Before Processing

First:

```bash
wc -l logs/production-app.log
```

Then:

```bash
head -n 3 logs/production-app.log
```

Then:

```bash
tail -n 3 logs/production-app.log
```

Then:

```bash
less logs/production-app.log
```

Never begin complex analysis without understanding the input structure.

---

# 96. Understand the Log Fields

A line looks like:

```text
2026-08-26T14:00:05 ERROR service=payments host=web02 status=502 duration=2100ms
```

Fields:

```text
$1 = timestamp
$2 = log level
$3 = service
$4 = host
$5 = status
$6 = duration
```

That understanding determines our later `awk` commands.

---

# 97. Count Errors

Use:

```bash
grep -c "ERROR" logs/production-app.log
```

Or:

```bash
grep "ERROR" logs/production-app.log | wc -l
```

Expected:

```text
5
```

---

# 98. Save Errors While Counting Them

Use:

```bash
grep "ERROR" logs/production-app.log \
| tee output/production-errors.log \
| wc -l
```

Now:

* The error records are stored.
* The terminal shows the count.
* You can inspect the saved subset later.

Verify:

```bash
cat output/production-errors.log
```

---

# 99. Which Services Are Failing?

Start:

```bash
grep "ERROR" logs/production-app.log
```

Extract field 3:

```bash
grep "ERROR" logs/production-app.log | awk '{print $3}'
```

Output resembles:

```text
service=api
service=payments
service=payments
service=api
service=payments
```

Remove the prefix:

```bash
grep "ERROR" logs/production-app.log \
| awk '{print $3}' \
| cut -d '=' -f 2
```

Now:

```text
api
payments
payments
api
payments
```

---

# 100. Count Errors by Service

Continue:

```bash
grep "ERROR" logs/production-app.log \
| awk '{print $3}' \
| cut -d '=' -f 2 \
| sort \
| uniq -c \
| sort -nr
```

Expected:

```text
3 payments
2 api
```

This suggests the payments service generated more error records in this sample.

It does not automatically tell you why.

---

# 101. Which Hosts Generate Errors?

Use:

```bash
grep "ERROR" logs/production-app.log \
| awk '{print $4}' \
| cut -d '=' -f 2 \
| sort \
| uniq -c \
| sort -nr
```

Possible output:

```text
3 web02
2 web03
```

Now you have another clue.

---

# 102. Which Error Status Codes Occur?

Use:

```bash
grep "ERROR" logs/production-app.log \
| awk '{print $5}' \
| cut -d '=' -f 2 \
| sort \
| uniq -c \
| sort -nr
```

Possible output:

```text
2 502
2 500
1 503
```

Now the error distribution is clearer.

---

# 103. Save a Summary With Redirection

Suppose you want the error-by-service summary saved:

```bash
grep "ERROR" logs/production-app.log \
| awk '{print $3}' \
| cut -d '=' -f 2 \
| sort \
| uniq -c \
| sort -nr \
> output/errors-by-service.txt
```

Verify:

```bash
cat output/errors-by-service.txt
```

---

# 104. Save and Display With `tee`

Instead:

```bash
grep "ERROR" logs/production-app.log \
| awk '{print $3}' \
| cut -d '=' -f 2 \
| sort \
| uniq -c \
| sort -nr \
| tee output/errors-by-service.txt
```

Now you see the result and save it simultaneously.

---

# 105. Capture the Error Count With Command Substitution

Run:

```bash
error_count=$(grep -c "ERROR" logs/production-app.log)
```

Then:

```bash
echo "Total application errors: $error_count"
```

Output:

```text
Total application errors: 5
```

You have now combined:

* `grep`.
* Command substitution.
* Variables.
* Quoting.
* `echo`.

---

# 106. Add Hostname and Date

Run:

```bash
echo "Report generated on $(hostname) at $(date)"
```

The exact output depends on your machine.

You can build a small report:

```bash
echo "Server: $(hostname)"
echo "Date: $(date)"
echo "Errors: $(grep -c "ERROR" logs/production-app.log)"
```

Notice the nested quoting in the last example can become difficult to read.

A clearer version is:

```bash
error_count=$(grep -c 'ERROR' logs/production-app.log)
echo "Errors: $error_count"
```

Readable commands are easier to maintain and troubleshoot.

---

# 107. Redirect a Report

You could create:

```bash
{
    echo "Application Log Report"
    echo "Server: $(hostname)"
    echo "Date: $(date)"
    echo "Errors: $(grep -c 'ERROR' logs/production-app.log)"
} > output/report.txt
```

The braces group commands so their combined stdout can be redirected together.

You do not need to master command grouping yet.

The important idea is that redirection can apply to a stream produced by multiple operations.

Check:

```bash
cat output/report.txt
```

---

# 108. Command Chaining in an Operational Workflow

Suppose you want to create an analysis directory, enter it, and only then create a file.

Use:

```bash
mkdir -p ~/analysis && cd ~/analysis && touch report.txt
```

Flow:

```text
mkdir
  |
success?
  |
 yes
  v
 cd
  |
success?
  |
 yes
  v
touch
```

If `cd` fails, `touch` does not run.

That is different from:

```bash
mkdir -p ~/analysis; cd ~/analysis; touch report.txt
```

which attempts each command sequentially regardless of earlier failure.

---

# 109. Failure Handling With `||`

Example:

```bash
grep "CRITICAL" logs/production-app.log \
|| echo "No CRITICAL lines found"
```

But there is an important detail.

`grep` returns:

```text
0 if matches were found

1 if no lines matched

other non-zero values for certain errors
```

So `||` can run when no matches exist, not only when something is technically broken.

Exit statuses are command-specific.

Do not automatically interpret every non-zero status as:

```text
The program crashed.
```

---

# 110. Exit Status Is Contextual

For `grep`:

```text
0 = match found

1 = no match

2 = error, in common GNU grep behavior
```

For another command, non-zero statuses can have different meanings.

Therefore:

```bash
command || echo "failed"
```

may oversimplify what actually happened.

When writing robust automation, understand the command's documented exit-status behavior.

---

# 111. Pipeline Exit Status Introduction

Suppose:

```bash
command1 | command2 | command3
```

Which exit status represents the pipeline?

In Bash's default behavior, the pipeline status is generally the exit status of the last command.

This can hide failures earlier in the pipeline.

For example:

```text
command1 fails
    |
    v
command2 still runs
    |
    v
command3 succeeds
```

The overall pipeline may appear successful based on the final command.

---

# 112. `pipefail` Introduction

Bash provides:

```bash
set -o pipefail
```

With `pipefail` enabled, a pipeline can report failure when a command within the pipeline fails, rather than only relying on the final command's status.

This becomes particularly important in:

* Shell scripts.
* CI/CD pipelines.
* Deployment automation.

You do not need to enable it globally just to complete this beginner lab.

Remember that:

```text
A pipeline producing output does not prove every stage succeeded.
```

---

# 113. Pipeline Troubleshooting Method

When a pipeline gives the wrong result, do not randomly modify it.

Break it apart.

Suppose:

```bash
grep "ERROR" logs/production-app.log \
| awk '{print $4}' \
| cut -d '=' -f 2 \
| sort \
| uniq -c \
| sort -nr
```

gives unexpected results.

Start with:

```bash
grep "ERROR" logs/production-app.log
```

Correct?

Then:

```bash
grep "ERROR" logs/production-app.log | awk '{print $4}'
```

Correct?

Then:

```bash
grep "ERROR" logs/production-app.log \
| awk '{print $4}' \
| cut -d '=' -f 2
```

Correct?

Continue one stage at a time.

This isolates the stage where your assumption becomes wrong.

---

# 114. Common Pipeline Failure: Wrong Field Number

Suppose you use:

```bash
awk '{print $5}'
```

expecting a hostname.

But your log format changed.

Maybe field 5 is now:

```text
status=500
```

The command works perfectly.

Your assumption is wrong.

Always inspect:

```bash
head logs/file
```

before relying on field numbers.

---

# 115. Common Pipeline Failure: Wrong Delimiter

Suppose:

```text
service=api
```

is passed to:

```bash
cut -d ':' -f 2
```

You expected:

```text
api
```

but the delimiter is:

```text
=
```

not:

```text
:
```

Correct:

```bash
cut -d '=' -f 2
```

The tool is not broken.

The input format and your command disagree.

---

# 116. Common Pipeline Failure: Case Sensitivity

You run:

```bash
grep "error" file
```

but the log contains:

```text
ERROR
```

Result:

```text
nothing
```

Try:

```bash
grep -i "error" file
```

Again, inspect the actual input rather than guessing.

---

# 117. Common Pipeline Failure: stderr Is Not Being Piped

Suppose:

```bash
some-command | grep "ERROR"
```

but the error message you want is written to stderr.

`grep` may never receive it.

Remember:

```text
Normal pipe:
stdout -> next command
```

not automatically:

```text
stdout + stderr -> next command
```

In Bash, you may encounter:

```bash
some-command 2>&1 | grep "ERROR"
```

This merges stderr into stdout before the pipe.

Only do this when combining the streams makes sense for your task.

---

# 118. Common Pipeline Failure: Overwriting the Wrong File

You intended:

```bash
command >> results.log
```

but typed:

```bash
command > results.log
```

Previous contents disappear.

Before redirecting to important files:

```bash
ls -l results.log
```

and, when appropriate:

```bash
cp results.log results.log.bak
```

Understand the consequences before writing.

---

# 119. Common Pipeline Failure: Permission Denied

You run:

```bash
command > /protected/path/file
```

and receive:

```text
Permission denied
```

Do not immediately use:

```bash
chmod 777
```

Investigate:

```bash
whoami
id
ls -ld /protected/path
ls -l /protected/path/file
```

Ask whether you are supposed to have permission to write there.

---

# 120. Common Pipeline Failure: Quoting

Suppose:

```bash
pattern="database timeout"
```

Then:

```bash
grep $pattern file
```

can be interpreted as multiple arguments after word splitting.

Prefer:

```bash
grep "$pattern" file
```

Quoting variables is an important shell habit.

---

# 121. Common Pipeline Failure: Wildcard Expansion

Suppose you intend to pass literal:

```text
*.log
```

to a program.

But you type:

```bash
echo *.log
```

The shell may expand it into filenames.

To preserve it literally:

```bash
echo '*.log'
```

The shell's preprocessing can change what the program actually receives.

---

# 122. Common Pipeline Failure: `uniq` Counts Incorrectly

You run:

```bash
awk '{print $4}' file | uniq -c
```

but identical values are scattered.

You may see repeated groups.

Use:

```bash
awk '{print $4}' file | sort | uniq -c
```

if grouping identical values globally is the goal.

---

# 123. Common Pipeline Failure: Numeric vs Lexical Sorting

Suppose counts are:

```text
2
10
100
9
```

Plain:

```bash
sort
```

may not give the numeric ordering you expect.

Use:

```bash
sort -n
```

For descending numeric order:

```bash
sort -nr
```

---

# 124. Common Pipeline Failure: Hiding Errors

Suppose:

```bash
some-command 2> /dev/null | grep "something"
```

returns nothing.

Maybe there genuinely was no result.

Or maybe the command failed and you discarded the evidence.

During troubleshooting, remove unnecessary `/dev/null` redirection.

Let yourself see errors.

---

# 125. Practical Pipeline Troubleshooting Lab

Use:

```text
~/pipeline-lab
```

The following lab combines everything learned in this part.

---

# 126. Lab 1: Standard Output

Run:

```bash
echo "Linux pipeline lab"
```

Identify:

```text
What is stdout?

Where did it go?
```

Now:

```bash
echo "Linux pipeline lab" > output/lab.txt
```

Verify:

```bash
cat output/lab.txt
```

Explain why nothing appeared on screen during the redirected command.

---

# 127. Lab 2: Overwrite vs Append

Run:

```bash
echo "first" > output/test.txt
echo "second" > output/test.txt
```

Inspect:

```bash
cat output/test.txt
```

Now:

```bash
echo "third" >> output/test.txt
```

Inspect again.

Explain the difference between:

```text
>
```

and:

```text
>>
```

---

# 128. Lab 3: Input Redirection

Create:

```bash
printf '%s\n' alpha beta gamma > data/words.txt
```

Run:

```bash
wc -l data/words.txt
```

Then:

```bash
wc -l < data/words.txt
```

Compare the output.

Explain why the second form does not need to display the filename.

---

# 129. Lab 4: Standard Error

Run:

```bash
ls /missing-lab-directory
```

Observe the error.

Now:

```bash
ls /missing-lab-directory 2> errors/missing.log
```

Inspect:

```bash
cat errors/missing.log
```

Explain:

```text
Why did 2> capture the message?
```

---

# 130. Lab 5: Append Errors

Run:

```bash
ls /missing-one 2> errors/multiple.log
ls /missing-two 2>> errors/multiple.log
ls /missing-three 2>> errors/multiple.log
```

Inspect:

```bash
cat errors/multiple.log
```

Explain:

```text
2>

vs

2>>
```

---

# 131. Lab 6: Separate stdout and stderr

Run:

```bash
ls data /missing-directory \
> output/success.txt \
2> errors/failure.txt
```

Inspect:

```bash
cat output/success.txt
```

Then:

```bash
cat errors/failure.txt
```

Identify which stream produced each file.

---

# 132. Lab 7: Combine stdout and stderr

Run:

```bash
ls data /missing-directory \
> output/combined.txt \
2>&1
```

Inspect:

```bash
cat output/combined.txt
```

Explain:

```text
2>&1
```

in your own words.

---

# 133. Lab 8: `/dev/null`

Run:

```bash
echo "discard this" > /dev/null
```

Then:

```bash
ls /missing-directory 2> /dev/null
```

Explain what happened.

Then answer:

```text
Why might discarding stderr be a bad idea during troubleshooting?
```

---

# 134. Lab 9: Basic Pipe

Run:

```bash
printf '%s\n' banana apple orange apple banana apple \
| sort
```

Then:

```bash
printf '%s\n' banana apple orange apple banana apple \
| sort \
| uniq -c
```

Then:

```bash
printf '%s\n' banana apple orange apple banana apple \
| sort \
| uniq -c \
| sort -nr
```

Explain each stage.

---

# 135. Lab 10: `tee`

Run:

```bash
grep "ERROR" logs/production-app.log \
| tee output/error-copy.log \
| wc -l
```

Check:

```bash
cat output/error-copy.log
```

Explain how `tee` allowed one stream to serve two purposes.

---

# 136. Lab 11: `xargs`

Run:

```bash
printf '%s\n' web01 web02 web03 \
| xargs -I {} echo "Checking {}"
```

Explain:

* What entered through stdin.
* What `xargs` did.
* What `{}` represented.
* Which command eventually ran.

---

# 137. Lab 12: `;`

Run:

```bash
echo "start"; ls /missing-directory; echo "finish"
```

Did `finish` run?

Why?

---

# 138. Lab 13: `&&`

Run:

```bash
echo "start" && echo "success"
```

Then:

```bash
ls /missing-directory && echo "This should only appear after success"
```

Explain why the second `echo` did or did not run.

---

# 139. Lab 14: `||`

Run:

```bash
ls /missing-directory || echo "The previous command did not succeed"
```

Explain why the second command ran.

---

# 140. Lab 15: Exit Status

Run:

```bash
ls data
echo $?
```

Then:

```bash
ls /missing-directory
echo $?
```

Compare the values.

Do not assume all non-zero values always mean the same thing across every command.

---

# 141. Lab 16: Quotes

Run:

```bash
echo '$HOME'
```

Then:

```bash
echo "$HOME"
```

Then:

```bash
echo \$HOME
```

Explain why all three behave differently.

---

# 142. Lab 17: Spaces and Quoting

Run:

```bash
mkdir -p "data/My Project"
```

Then:

```bash
ls "data/My Project"
```

Try:

```bash
cd "data/My Project"
```

Return:

```bash
cd ~/pipeline-lab
```

Explain why the quotes matter.

---

# 143. Lab 18: Command Substitution

Run:

```bash
echo "Current user: $(whoami)"
```

Then:

```bash
echo "Hostname: $(hostname)"
```

Then:

```bash
echo "Current directory: $(pwd)"
```

Explain what happens before `echo` executes.

---

# 144. Lab 19: Variable Plus Command Substitution

Run:

```bash
errors=$(grep -c "ERROR" logs/production-app.log)
```

Then:

```bash
echo "Total errors: $errors"
```

Explain:

```text
What was stored in errors?

Which command generated it?

Why was $(...) needed?
```

---

# 145. Lab 20: Shell Expansion

Run:

```bash
touch data/app.log data/api.log data/database.log
```

Then:

```bash
echo data/*.log
```

Observe the filenames.

Then:

```bash
echo 'data/*.log'
```

Compare.

Explain why the quoted version remains literal.

---

# 146. Lab 21: Brace Expansion

Run:

```bash
mkdir -p data/server{1,2,3}
```

Check:

```bash
ls data
```

Explain what the shell expanded before `mkdir` ran.

---

# 147. Lab 22: Error Analysis Pipeline

Using:

```text
logs/production-app.log
```

answer:

```text
Which service generated the most ERROR lines?
```

Build the pipeline one stage at a time.

Solution:

```bash
grep "ERROR" logs/production-app.log \
| awk '{print $3}' \
| cut -d '=' -f 2 \
| sort \
| uniq -c \
| sort -nr
```

Do not accept the solution until you can explain every stage.

---

# 148. Lab 23: Host Analysis

Question:

```text
Which host generated the most ERROR lines?
```

Solution:

```bash
grep "ERROR" logs/production-app.log \
| awk '{print $4}' \
| cut -d '=' -f 2 \
| sort \
| uniq -c \
| sort -nr
```

Verify the result manually against:

```bash
grep "ERROR" logs/production-app.log
```

---

# 149. Lab 24: HTTP Error Status Analysis

Question:

```text
Which HTTP error status appears most frequently?
```

Use:

```bash
grep "ERROR" logs/production-app.log \
| awk '{print $5}' \
| cut -d '=' -f 2 \
| sort \
| uniq -c \
| sort -nr
```

Explain why:

```text
grep
```

comes before:

```text
awk
```

in this particular solution.

---

# 150. Lab 25: Save and Display Analysis

Take the previous pipeline and save the result while displaying it:

```bash
grep "ERROR" logs/production-app.log \
| awk '{print $5}' \
| cut -d '=' -f 2 \
| sort \
| uniq -c \
| sort -nr \
| tee output/error-status-summary.txt
```

Verify:

```bash
cat output/error-status-summary.txt
```

---

# 151. Lab 26: Intentional Pipeline Error

Run the correct command:

```bash
grep "ERROR" logs/production-app.log \
| awk '{print $4}'
```

Now intentionally use:

```bash
grep "ERROR" logs/production-app.log \
| awk '{print $7}'
```

What happens?

Why?

Inspect the raw log again.

The purpose is to teach:

```text
A command can execute successfully while your logic is incorrect.
```

---

# 152. Lab 27: Intentional Delimiter Error

Run:

```bash
grep "ERROR" logs/production-app.log \
| awk '{print $4}' \
| cut -d ':' -f 2
```

Compare with:

```bash
grep "ERROR" logs/production-app.log \
| awk '{print $4}' \
| cut -d '=' -f 2
```

Explain why one matches the actual data format.

---

# 153. Lab 28: stderr and Pipes

Run:

```bash
ls data /missing-directory | grep "log"
```

Observe what happens to the error.

Now try:

```bash
ls data /missing-directory 2>&1 | grep -E 'log|cannot access'
```

Explain why `grep` can now process the error text.

---

# 154. Lab 29: Build a Small Report

Create:

```bash
error_count=$(grep -c 'ERROR' logs/production-app.log)
warn_count=$(grep -c 'WARN' logs/production-app.log)
total_count=$(wc -l < logs/production-app.log)
```

Then:

```bash
{
    echo "Application Log Summary"
    echo "Server: $(hostname)"
    echo "Generated: $(date)"
    echo "Total records: $total_count"
    echo "ERROR records: $error_count"
    echo "WARN records: $warn_count"
} | tee output/application-summary.txt
```

Inspect:

```bash
cat output/application-summary.txt
```

---

# 155. Lab 30: Full Troubleshooting Scenario

Scenario:

A developer reports:

```text
"The application is failing, but I don't know whether the problem is tied to one service, one host, or one HTTP status."
```

Using:

```text
logs/production-app.log
```

answer:

1. How many total records exist?
2. How many ERROR records exist?
3. How many WARN records exist?
4. Which services produced errors?
5. Which service produced the most?
6. Which hosts produced errors?
7. Which host produced the most?
8. Which HTTP error statuses occurred?
9. Which status occurred most often?
10. Save the ERROR records separately.
11. Save an error-status summary.
12. Display the summary while saving it.
13. Store the total error count in a variable.
14. Print a one-line summary containing the hostname and error count.
15. Verify every summary against the original error records.

Your workflow should resemble:

```text
Observe
   |
   v
Inspect raw input
   |
   v
Define question
   |
   v
Filter relevant records
   |
   v
Extract relevant fields
   |
   v
Normalize
   |
   v
Group
   |
   v
Count
   |
   v
Sort
   |
   v
Save/report
   |
   v
Verify against raw evidence
```

---

# 156. Practical Troubleshooting Principles

Pipelines are powerful, but they can also hide mistakes.

Use these principles.

### Principle 1: Inspect the input first

Before:

```bash
awk '{print $9}' file
```

look at:

```bash
head file
```

Know what field 9 represents.

### Principle 2: Build one stage at a time

Do not immediately write six commands.

Add one transformation, inspect it, then continue.

### Principle 3: Understand stdout and stderr

If data seems missing, determine which stream produced it.

### Principle 4: Do not hide useful errors

Remove unnecessary:

```bash
2> /dev/null
```

while troubleshooting.

### Principle 5: Quote variables

Prefer:

```bash
grep "$pattern" file
```

rather than:

```bash
grep $pattern file
```

### Principle 6: Verify final results

A pipeline can run successfully and still answer the wrong question.

---

# 157. Command Reference

## Standard streams

```text
stdin   = standard input
stdout  = standard output
stderr  = standard error
```

## File descriptors

```text
0 = stdin
1 = stdout
2 = stderr
```

## Redirection

```bash
command > file
command >> file
command < file
command 2> errors.log
command 2>> errors.log
command > output.log 2> errors.log
command > combined.log 2>&1
```

## Discarding output

```bash
command > /dev/null
command 2> /dev/null
command > /dev/null 2>&1
```

## Pipes

```bash
command1 | command2
```

## `tee`

```bash
command | tee file
command | tee -a file
```

## `xargs`

```bash
command1 | xargs command2
command1 | xargs -n 1 command2
command1 | xargs -I {} command2 {}
```

## Chaining

```bash
command1; command2

command1 && command2

command1 || command2
```

## Command substitution

```bash
$(command)
```

Example:

```bash
user=$(whoami)
```

## Quoting

```bash
'text'
"text"
\character
```

---

# 158. What You Should Now Understand

Before moving to Part 8, you should confidently understand:

* What standard streams are.
* What standard input means.
* What standard output means.
* What standard error means.
* Why stdout and stderr are separate.
* What a file descriptor is at a beginner level.
* Why `0` represents stdin.
* Why `1` represents stdout.
* Why `2` represents stderr.
* How `>` redirects stdout.
* Why `>` can overwrite existing data.
* How `>>` appends stdout.
* How `<` redirects a file into stdin.
* How `2>` redirects stderr.
* How `2>>` appends stderr.
* How stdout and stderr can be sent to separate files.
* How `2>&1` can send stderr to stdout's current destination.
* Why redirection order matters.
* What `/dev/null` is.
* Why discarding errors can hurt troubleshooting.
* What a pipe does.
* Why pipes are central to Linux workflows.
* Why a normal pipe connects stdout, not automatically stderr.
* What `tee` does.
* How `tee -a` differs from normal `tee`.
* Why `tee` can be useful with privileged writes.
* What `xargs` does.
* Why `xargs` converts stdin into command arguments.
* Why filenames and delimiters require care with `xargs`.
* How `grep`, `awk`, `cut`, `sort`, `uniq`, and `wc` can work together.
* Why pipeline order matters.
* How `;` chains commands.
* Why `;` does not depend on previous success.
* What an exit status is.
* Why exit status `0` conventionally means success.
* What `&&` does.
* What `||` does.
* Why non-zero exit statuses must be interpreted according to the command.
* What single quotes do.
* What double quotes do.
* Why quoting variables matters.
* What escaping means.
* How to escape a space or special character.
* What command substitution is.
* How `$(command)` works.
* How command output can be stored in a variable.
* What basic shell expansion means.
* What variable expansion is.
* What tilde expansion is.
* What filename expansion is.
* Why shell wildcard expansion differs from regex.
* What brace expansion does.
* Why destructive commands combined with expansion require caution.
* How to build a pipeline one stage at a time.
* How to verify a pipeline against raw data.
* Why a pipeline can produce technically valid but logically incorrect results.
* Why stderr sometimes seems to bypass a pipe.
* Why `pipefail` matters in more robust shell automation.
* How to perform basic application-log analysis using pipelines.

---

# 159. Hands-On Practice

Without copying the completed examples, create your own:

```text
~/linux-pipeline-project/
|
+-- logs/
+-- data/
+-- reports/
+-- errors/
```

Complete these tasks:

1. Create a 30-line application log.

2. Include `INFO`, `WARN`, and `ERROR`.

3. Include at least three service names.

4. Include at least three server names.

5. Include HTTP status codes.

6. Redirect normal command output into a file.

7. Append additional output.

8. Generate an intentional error and redirect stderr separately.

9. Append another error to the same error file.

10. Run a command producing stdout and stderr.

11. Save each stream separately.

12. Save both streams together.

13. Redirect something harmless to `/dev/null`.

14. Explain why you should not routinely discard troubleshooting errors.

15. Use `<` to provide a file as stdin.

16. Use a pipe to count matching log records.

17. Use `grep` to select errors.

18. Use `awk` to extract service names.

19. Use `cut` to remove a `key=` prefix.

20. Use `sort` to group values.

21. Use `uniq -c` to count them.

22. Use `sort -nr` to rank them.

23. Use `tee` to save and display a result.

24. Use `tee` inside a multi-stage pipeline.

25. Use `xargs` with safe, non-destructive commands.

26. Demonstrate `;`.

27. Demonstrate `&&`.

28. Demonstrate `||`.

29. Inspect an exit status with `$?`.

30. Demonstrate the difference between single and double quotes.

31. Escape a special character.

32. Create and access a directory containing a space.

33. Use `$(whoami)` inside another command.

34. Store `hostname` output in a variable.

35. Store the number of ERROR records in a variable.

36. Print a report using those variables.

37. Demonstrate filename expansion using `*.log`.

38. Prevent filename expansion using quotes.

39. Demonstrate brace expansion.

40. Build a pipeline answering which service has the most errors.

41. Build another pipeline answering which host has the most errors.

42. Build another pipeline counting HTTP error statuses.

43. Save the results into report files.

44. Intentionally break one field number in an `awk` command.

45. Diagnose the problem stage by stage.

46. Intentionally use the wrong `cut` delimiter.

47. Diagnose and correct it.

48. Demonstrate that stderr does not normally flow through `|`.

49. Redirect stderr into stdout and pipe the combined stream.

50. Verify all final summaries against the original log.

---

# 160. Beginner Knowledge Check

1. What are the three standard streams?

2. What is stdin?

3. What is stdout?

4. What is stderr?

5. Why does Linux separate normal output from errors?

6. What file descriptor represents stdin?

7. What file descriptor represents stdout?

8. What file descriptor represents stderr?

9. What does `>` do?

10. What risk comes with `>`?

11. What does `>>` do?

12. What does `<` do?

13. What does `2>` mean?

14. What does `2>>` mean?

15. How can stdout and stderr be saved separately?

16. What does `2>&1` mean?

17. Why does redirection order matter?

18. What is `/dev/null`?

19. Why might `2> /dev/null` be harmful during troubleshooting?

20. What does `|` do?

21. Which stream normally enters a pipe?

22. Does stderr automatically enter a normal pipe?

23. Why are pipes useful?

24. What does `tee` do?

25. What does `tee -a` do?

26. Why might `tee` be useful when writing through `sudo`?

27. What does `xargs` do?

28. Why must `xargs` be used carefully with filenames?

29. Why is `sort | uniq -c` common?

30. What does `sort -nr` do?

31. What does `;` mean between commands?

32. What is an exit status?

33. What does exit status `0` conventionally mean?

34. What does `&&` do?

35. What does `||` do?

36. Does every non-zero status mean exactly the same thing?

37. What do single quotes generally do?

38. What do double quotes generally do?

39. Why is `"$variable"` usually safer than `$variable`?

40. What does a backslash do in many shell contexts?

41. How can you reference a path containing spaces?

42. What is command substitution?

43. What does `$(whoami)` do?

44. How do you store command output in a variable?

45. What is variable expansion?

46. What is tilde expansion?

47. What is filename expansion?

48. Is shell `*.log` a regular expression?

49. What is brace expansion?

50. Why can shell expansion become dangerous with destructive commands?

51. Why should a pipeline be built incrementally?

52. Why should the raw data be inspected before using `awk '{print $7}'`?

53. What happens if `cut` uses the wrong delimiter?

54. Why might a pipeline miss error messages?

55. How can stderr be combined with stdout before a pipe in Bash?

56. Why can `uniq -c` give misleading grouping without `sort`?

57. What is the difference between a command executing successfully and your analysis being logically correct?

58. What does `set -o pipefail` address?

59. Why should final pipeline results be verified?

60. What question should you ask before building any pipeline?

---

# 161. Knowledge Check Answers

1. Standard input, standard output, and standard error.

2. The default input stream available to a process.

3. The normal output stream.

4. The stream commonly used for errors and diagnostics.

5. So normal data and diagnostic information can be processed independently.

6. `0`.

7. `1`.

8. `2`.

9. Redirects stdout to a file, normally replacing its existing contents.

10. It can overwrite valuable data.

11. Appends stdout to a file.

12. Provides a file as standard input.

13. Redirect file descriptor 2, stderr, to a destination.

14. Append stderr to a destination.

15. For example:

```bash
command > output.log 2> errors.log
```

16. Redirect stderr to the current destination of stdout.

17. The shell processes redirections from left to right, so the destination referenced by a later or earlier redirection can differ.

18. A special device that discards data written to it.

19. It can hide useful diagnostic evidence.

20. Connects one command's stdout to another command's stdin.

21. stdout.

22. No.

23. They let small commands be combined into larger data-processing workflows without manually storing every intermediate result.

24. Copies stdin to standard output and one or more files.

25. Appends rather than replacing the destination file.

26. Because `sudo tee` can perform the actual privileged file write, whereas `sudo echo ... > file` does not automatically privilege the shell's redirection operation.

27. Converts input into arguments for another command.

28. Whitespace and unusual filename characters can cause incorrect argument boundaries unless handled safely.

29. Sorting groups identical lines, then `uniq -c` counts adjacent groups.

30. Sort numerically in reverse order.

31. Run the next command sequentially regardless of whether the previous command succeeded.

32. A numeric status returned when a command finishes.

33. Success.

34. Run the next command only if the previous command succeeds.

35. Run the next command if the previous command does not succeed.

36. No. Exit-status meaning depends on the program.

37. Preserve enclosed text literally in common Bash usage.

38. Keep text together while still allowing expansions such as variables and command substitution.

39. Quoting helps prevent unwanted word splitting and filename expansion.

40. It can remove the special meaning of the following character.

41. Quote it:

```bash
cd "My Project"
```

or escape the space:

```bash
cd My\ Project
```

42. Running a command and substituting its stdout into another shell expression.

43. Runs `whoami` and substitutes its output.

44.

```bash
variable=$(command)
```

45. The shell replaces a variable reference with its value.

46. The shell expands `~` to the appropriate home directory in supported contexts.

47. The shell expands patterns such as `*.log` into matching filenames.

48. No. It is shell globbing.

49. A shell expansion that generates multiple strings from braces, such as `server{1,2,3}`.

50. Expansion may cause a destructive command to operate on many more files than expected.

51. So each transformation can be inspected and errors can be isolated.

52. Because field 7 may not contain what you assume.

53. It may extract the wrong information or fail to split the line as intended.

54. The messages may be written to stderr rather than stdout.

55.

```bash
command 2>&1 | other-command
```

56. `uniq` groups only adjacent identical lines.

57. A command may return success while processing the wrong field, wrong file, wrong delimiter, or wrong assumptions.

58. It helps pipelines reflect failures within earlier pipeline stages instead of relying only on the final command's status.

59. A plausible-looking result can still be wrong.

60. Ask:

```text
What question am I trying to answer?
```

---

# 162. The Mental Model to Keep

When working with Linux commands, think in terms of data flow:

```text
                INPUT
                  |
                  v
              stdin (0)
                  |
                  v
               COMMAND
                  |
          +-------+-------+
          |               |
          v               v
      stdout (1)       stderr (2)
          |               |
          |               +----> terminal
          |               +----> error file
          |               +----> /dev/null
          |
          +----> terminal
          |
          +----> file
          |
          +----> tee
          |
          +----> pipe
                   |
                   v
              next command
                   |
                   v
              next command
                   |
                   v
                 result
```

When a pipeline fails, do not see it as one enormous command.

See individual stages:

```text
Raw data
   |
   v
Filter
   |
   v
Inspect
   |
   v
Extract
   |
   v
Inspect
   |
   v
Normalize
   |
   v
Inspect
   |
   v
Sort
   |
   v
Count
   |
   v
Rank
   |
   v
Verify
```

And remember the most important distinction from this part:

```text
Pipes move data between commands.

Redirection changes where streams come from or go.

Command chaining decides which commands run and when.

Quoting controls how the shell interprets text.

Expansion transforms parts of your command before the program runs.
```

Once you understand those five ideas, long Linux command lines stop looking like random symbols. You can read them from left to right, identify where the data comes from, understand what each command does to it, determine where the result goes, and troubleshoot the workflow one stage at a time.
