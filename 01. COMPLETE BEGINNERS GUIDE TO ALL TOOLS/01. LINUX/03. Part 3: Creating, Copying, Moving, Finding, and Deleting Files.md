# Part 3: Creating, Copying, Moving, Finding, and Deleting Files

In Part 2, you learned how the Linux filesystem is organized and how to navigate it using paths.

You learned how to answer questions such as:

* Where am I?
* What files and directories are here?
* How do I move somewhere else?
* Is this path absolute or relative?
* What do `/`, `~`, `.`, and `..` mean?

Now you are ready to start changing the filesystem.

This is an important transition.

Until now, most of your work involved inspecting and navigating existing resources.

In this part, you will learn how to:

* Create files
* Create directories
* Build directory structures
* View file contents
* Copy files and directories
* Move files
* Rename files
* Delete files and directories
* Identify file types
* Work with hidden files
* Use wildcards
* Search for files
* Understand basic file metadata
* Understand hard and symbolic links
* Troubleshoot common file-management problems

These are not just beginner commands.

Engineers use them constantly.

A DevOps engineer might use them to:

* Organize deployment files
* Copy configuration files before editing them
* Move application logs
* Locate configuration files
* Inspect CI/CD output
* Find large groups of files
* Create symbolic links
* Remove obsolete deployment artifacts

A simple workflow might look like:

```text
Create
  |
  v
Inspect
  |
  v
Copy
  |
  v
Modify
  |
  v
Move
  |
  v
Verify
  |
  v
Delete when no longer needed
```

The most important habit throughout this part is verification.

Do not simply run a command.

Run the command, then confirm that Linux did what you intended.

---

# 1. Before You Start: Create a Safe Practice Area

File-management commands can modify and delete data.

Do not practice destructive commands inside important system directories such as:

```text
/etc
/usr
/var
/root
```

Instead, work inside your home directory.

First check your location:

```bash
pwd
```

Return home if necessary:

```bash
cd ~
```

Verify:

```bash
pwd
```

Now create a directory for this part:

```bash
mkdir linux-part3
```

Enter it:

```bash
cd linux-part3
```

Verify:

```bash
pwd
```

Your path may look similar to:

```text
/home/ann/linux-part3
```

Everything in this article can now be practiced inside this directory.

---

# 2. Creating Files With `touch`

One simple way to create an empty file is:

```bash
touch filename
```

Example:

```bash
touch application.log
```

Verify:

```bash
ls
```

You should see:

```text
application.log
```

For more information:

```bash
ls -l application.log
```

You may see something similar to:

```text
-rw-r--r-- 1 ann ann 0 Aug 26 10:30 application.log
```

Notice the size:

```text
0
```

The file is empty.

---

# 3. What Does `touch` Actually Do?

Beginners often learn:

"`touch` creates files."

That is useful, but incomplete.

The main purpose of `touch` is to update file timestamps.

If the named file does not exist, `touch` normally creates an empty file.

If it already exists, `touch` updates certain timestamps instead of creating another file with the same name.

Try:

```bash
touch application.log
```

Then:

```bash
ls -l application.log
```

Wait briefly and run:

```bash
touch application.log
```

Then:

```bash
ls -l application.log
```

You may notice the displayed modification time changed.

This is because the file already existed.

Linux did not create:

```text
application.log
application.log
```

with identical names in the same directory.

It updated the existing file.

---

# 4. Creating Multiple Files

You can give `touch` multiple arguments.

Example:

```bash
touch app.log error.log access.log
```

Verify:

```bash
ls
```

You should see all three files.

This:

```bash
touch app.log error.log access.log
```

contains:

* Command: `touch`
* Argument 1: `app.log`
* Argument 2: `error.log`
* Argument 3: `access.log`

---

# 5. Creating Files at Specific Paths

You do not need to be inside the destination directory.

Suppose your current directory contains a directory called:

```text
logs
```

You could create:

```bash
touch logs/application.log
```

Or use an absolute path:

```bash
touch /home/ann/linux-part3/logs/application.log
```

The parent directory must already exist unless another tool creates it first.

If `logs` does not exist, you may receive:

```text
No such file or directory
```

`touch` creates files.

It does not automatically create missing parent directories.

---

# 6. Creating Directories With `mkdir`

`mkdir` means:

```text
make directory
```

General syntax:

```bash
mkdir directory-name
```

Example:

```bash
mkdir logs
```

Verify:

```bash
ls
```

You should see:

```text
logs
```

Check with long format:

```bash
ls -l
```

The entry should begin with:

```text
d
```

For example:

```text
drwxr-xr-x
```

The initial `d` means directory.

---

# 7. Creating Multiple Directories

You can create several directories at once:

```bash
mkdir scripts configs backups
```

Verify:

```bash
ls
```

You may now have:

```text
backups
configs
logs
scripts
```

This is more efficient than running `mkdir` four separate times.

---

# 8. Creating Nested Directories

Suppose you want this structure:

```text
project
|
+-- environments
    |
    +-- production
        |
        +-- nginx
```

You might try:

```bash
mkdir project/environments/production/nginx
```

If `project`, `environments`, and `production` do not already exist, Linux may respond:

```text
No such file or directory
```

By default, `mkdir` cannot create the final directory through nonexistent parent directories.

This is where:

```bash
mkdir -p
```

becomes useful.

---

# 9. `mkdir -p`

The `-p` option creates missing parent directories as needed.

Run:

```bash
mkdir -p project/environments/production/nginx
```

Now inspect:

```bash
ls project
```

Then:

```bash
ls project/environments
```

Then:

```bash
ls project/environments/production
```

You should eventually see:

```text
nginx
```

The resulting structure is:

```text
project
|
+-- environments
    |
    +-- production
        |
        +-- nginx
```

This is extremely useful when creating project structures.

---

# 10. Why `mkdir -p` Is Common in Automation

Automation often needs to ensure a directory structure exists before creating files.

For example:

```bash
mkdir -p app/logs/archive
```

If the directories do not exist, they are created.

If they already exist, `mkdir -p` generally does not fail merely because the directory exists.

This makes it useful in scripts and deployment processes.

---

# 11. Creating a Realistic Project Structure

Create:

```bash
mkdir -p devops-project/{configs,logs,scripts,backups}
```

Brace expansion is a shell feature that can help generate several related names.

Your shell expands this conceptually into:

```text
devops-project/configs
devops-project/logs
devops-project/scripts
devops-project/backups
```

If brace expansion is supported by your shell, inspect:

```bash
ls devops-project
```

You should see:

```text
backups
configs
logs
scripts
```

Now create some files:

```bash
touch devops-project/configs/app.conf
touch devops-project/logs/application.log
touch devops-project/scripts/deploy.sh
```

You are beginning to build a realistic filesystem structure.

---

# 12. Viewing Files

Creating files is only part of file management.

You also need to inspect their contents.

Linux provides several commands for different viewing situations.

Important commands include:

```text
cat
less
more
head
tail
```

They overlap, but they are useful for different purposes.

---

# 13. Adding Some Practice Content

Our files are currently empty.

Use this command to create a small practice log:

```bash
printf "Application started\nDatabase connected\nUser login successful\nWarning: memory usage high\nError: connection timeout\nApplication stopped\n" > devops-project/logs/application.log
```

Do not worry about mastering `printf` or `>` yet.

You will study redirection properly in Part 7.

For this exercise:

* `printf` produces text.
* `\n` represents a new line.
* `>` writes the output into the file.

Now the file contains several lines.

---

# 14. Viewing Files With `cat`

`cat` is commonly used to display file contents.

Run:

```bash
cat devops-project/logs/application.log
```

You should see:

```text
Application started
Database connected
User login successful
Warning: memory usage high
Error: connection timeout
Application stopped
```

For small files, `cat` is convenient.

---

# 15. What Does `cat` Mean?

`cat` comes from concatenate.

It can do more than simply display one file.

For example:

```bash
cat file1 file2
```

can display the contents of both files one after another.

You will later learn more ways to combine files and output.

For now, use `cat` primarily for quickly viewing small text files.

---

# 16. When `cat` Is Not Ideal

Imagine a log file containing 100,000 lines.

Running:

```bash
cat huge-application.log
```

could flood your terminal with output.

For large files, tools such as:

```text
less
head
tail
```

are often more useful.

Choosing the right tool matters.

---

# 17. Viewing Files With `less`

Run:

```bash
less devops-project/logs/application.log
```

`less` opens the file in an interactive viewer.

For a larger file, you can move through it without printing the entire file into your terminal history.

Common controls include:

```text
Space    Move forward
b        Move backward
/word    Search for word
n        Next search match
q        Quit
```

For example, while inside `less`, type:

```text
/Error
```

and press Enter.

`less` searches for `Error`.

Press:

```text
q
```

to leave the viewer.

---

# 18. Why `less` Is Important for Logs

Suppose you are investigating:

```text
/var/log/application.log
```

and it contains thousands of lines.

Instead of dumping everything with `cat`, you can use:

```bash
less /var/log/application.log
```

Then search for:

```text
ERROR
```

or:

```text
failed
```

This is a normal troubleshooting workflow.

---

# 19. Viewing Files With `more`

`more` is another pager used to view text one screen at a time.

Example:

```bash
more devops-project/logs/application.log
```

`more` is older and generally less flexible than `less`.

You will still encounter it on Linux and Unix-like systems.

A useful memory aid is:

```text
more
Older, simpler pager

less
More flexible interactive pager
```

For most interactive viewing, you will often prefer `less`.

---

# 20. Viewing the Beginning of a File With `head`

`head` displays the beginning of a file.

Run:

```bash
head devops-project/logs/application.log
```

By default, `head` normally displays the first 10 lines.

Our file contains fewer than 10 lines, so you may see the whole file.

You can request a specific number of lines:

```bash
head -n 3 devops-project/logs/application.log
```

Expected:

```text
Application started
Database connected
User login successful
```

This is useful when you only need the beginning of a large file.

---

# 21. Viewing the End of a File With `tail`

`tail` displays the end of a file.

Run:

```bash
tail devops-project/logs/application.log
```

By default, it normally shows the last 10 lines.

To show only the last three:

```bash
tail -n 3 devops-project/logs/application.log
```

Expected:

```text
Warning: memory usage high
Error: connection timeout
Application stopped
```

This is especially useful for logs because the newest log entries are often written near the end.

---

# 22. Following a File With `tail -f`

One of the most useful Linux troubleshooting commands is:

```bash
tail -f filename
```

The:

```text
-f
```

means follow.

Run:

```bash
tail -f devops-project/logs/application.log
```

The command remains running and waits for new lines to be added.

When another process writes new information to the file, `tail -f` displays it.

This is useful for monitoring logs in real time.

A common production workflow is:

```text
Application running
       |
       v
Application writes logs
       |
       v
Log file
       |
       v
tail -f
       |
       v
Engineer watches events
```

To stop `tail -f`, press:

```text
Ctrl+C
```

---

# 23. Choosing the Right File Viewer

Use this mental model:

| Command   | Best beginner use                  |
| --------- | ---------------------------------- |
| `cat`     | Quickly display a small file       |
| `less`    | Browse a larger file interactively |
| `more`    | Basic paged viewing                |
| `head`    | View the beginning                 |
| `tail`    | View the end                       |
| `tail -f` | Watch new lines appear             |

You do not need to choose only one.

Different troubleshooting situations require different tools.

---

# 24. Copying Files With `cp`

`cp` means copy.

Basic syntax:

```bash
cp SOURCE DESTINATION
```

Suppose you have:

```text
devops-project/configs/app.conf
```

Create some content:

```bash
printf "environment=production\nport=8080\n" > devops-project/configs/app.conf
```

Now create a backup:

```bash
cp devops-project/configs/app.conf devops-project/backups/app.conf
```

Verify:

```bash
ls devops-project/backups
```

You should see:

```text
app.conf
```

Check both:

```bash
cat devops-project/configs/app.conf
```

```bash
cat devops-project/backups/app.conf
```

The original still exists.

That is what makes copying different from moving.

---

# 25. Understanding Source and Destination

This command:

```bash
cp app.conf backup.conf
```

means:

```text
cp      command

app.conf
        source

backup.conf
        destination
```

Linux reads it approximately as:

```text
Copy app.conf
to
backup.conf
```

The order matters.

A common beginner mistake is reversing source and destination.

Before pressing Enter, mentally read:

```text
cp SOURCE DESTINATION
```

---

# 26. Copying and Renaming at the Same Time

Suppose:

```text
app.conf
```

exists.

Run:

```bash
cp app.conf app.conf.backup
```

Now both files exist:

```text
app.conf
app.conf.backup
```

This is a common configuration-management habit.

Before modifying an important configuration file, an administrator might create a backup copy.

For example:

```bash
cp nginx.conf nginx.conf.backup
```

This does not guarantee a complete backup strategy, but it can provide a quick local copy before an edit.

---

# 27. Copying a File Into a Directory

Suppose you have:

```text
app.conf
```

and a directory:

```text
backups/
```

Run:

```bash
cp app.conf backups/
```

The resulting structure becomes:

```text
.
|
+-- app.conf
|
+-- backups
    |
    +-- app.conf
```

The original remains in its original location.

---

# 28. Be Careful With Overwrites

Suppose the destination already contains:

```text
app.conf
```

Running:

```bash
cp app.conf backups/app.conf
```

may overwrite the destination file.

This can destroy the previous contents.

When working with important data, inspect before copying:

```bash
ls -l app.conf backups/app.conf
```

You can also use interactive mode:

```bash
cp -i app.conf backups/app.conf
```

Depending on your environment, this can prompt before overwriting.

Do not depend blindly on aliases or prompts.

Know your source and destination before copying.

---

# 29. Copying Directories

Suppose you have:

```text
configs/
```

containing:

```text
app.conf
database.conf
```

Trying:

```bash
cp configs backups
```

will not normally copy the directory contents as intended.

You need recursive copying.

---

# 30. `cp -r`

The:

```text
-r
```

option means recursive.

A directory may contain:

* Files
* Directories
* Directories inside directories
* More files

Recursive copying tells `cp` to descend through the directory structure.

Example:

```bash
cp -r configs configs-backup
```

Result:

```text
configs
|
+-- app.conf
+-- database.conf

configs-backup
|
+-- app.conf
+-- database.conf
```

Verify:

```bash
ls configs-backup
```

---

# 31. Why Recursive Operations Need Care

Recursive operations affect an entire directory tree.

For a directory like:

```text
project
|
+-- configs
+-- logs
+-- scripts
|   |
|   +-- deployment
|       |
|       +-- deploy.sh
|
+-- backups
```

a recursive operation may affect everything beneath `project`.

That is powerful.

It also increases the impact of mistakes.

Always inspect the target before recursive operations.

---

# 32. Moving Files With `mv`

`mv` means move.

General syntax:

```bash
mv SOURCE DESTINATION
```

Suppose:

```text
application.log
```

is in your current directory.

You want it under:

```text
logs/
```

Run:

```bash
mv application.log logs/
```

Verify:

```bash
ls
```

Then:

```bash
ls logs
```

The file should no longer be in its original location.

It should now exist inside `logs`.

---

# 33. Copy vs Move

This distinction is essential.

## Copy

```bash
cp source destination
```

Result:

```text
Original remains
+
New copy exists
```

## Move

```bash
mv source destination
```

Result:

```text
Original location no longer contains the file
+
File exists at new location
```

Think:

```text
cp = duplicate

mv = relocate
```

---

# 34. Renaming Files With `mv`

Linux does not require a separate basic `rename` command for simple renaming.

`mv` can rename a file.

Suppose:

```text
app.log
```

exists.

Run:

```bash
mv app.log application.log
```

Verify:

```bash
ls
```

`app.log` is gone.

`application.log` exists.

The file was moved from one name to another name within the same directory.

Conceptually:

```text
app.log
   |
   | mv
   v
application.log
```

---

# 35. Renaming Directories

The same principle applies to directories.

Example:

```bash
mv configs configuration
```

This can rename:

```text
configs/
```

to:

```text
configuration/
```

Verify:

```bash
ls
```

Again, `mv` handles both moving and renaming.

---

# 36. Moving and Renaming Simultaneously

Suppose:

```text
app.conf
```

is in your current directory.

You want to move it into:

```text
configs/
```

and rename it:

```text
production.conf
```

Run:

```bash
mv app.conf configs/production.conf
```

One command performed both operations.

---

# 37. `mv` Can Also Overwrite

Like `cp`, `mv` can replace an existing destination under certain conditions.

This makes careless moves dangerous.

Before moving important files, inspect:

```bash
ls -l SOURCE
```

and:

```bash
ls -l DESTINATION
```

You can use:

```bash
mv -i source destination
```

to request interactive confirmation before some overwrites.

Still, your main protection should be understanding the operation before running it.

---

# 38. Removing Files With `rm`

`rm` means remove.

Example:

```bash
touch temporary.txt
```

Verify:

```bash
ls
```

Now delete it:

```bash
rm temporary.txt
```

Verify again:

```bash
ls
```

The file should be gone.

---

# 39. `rm` Is Not a Recycle Bin

This is extremely important.

On a typical command-line Linux system:

```bash
rm file
```

does not move the file into a convenient desktop recycle bin.

You should treat removal as potentially permanent.

That means before using `rm`, ask:

* Am I in the correct directory?
* Is this the correct filename?
* Is this an important file?
* Do I have a backup?
* Does my command contain a wildcard?
* Am I using recursive deletion?

The more powerful the deletion command, the more carefully you should verify it.

---

# 40. Interactive Removal

For additional confirmation, you can use:

```bash
rm -i filename
```

Linux may ask whether you want to remove the file.

This can be useful while learning.

However, do not develop a habit of assuming every environment will protect you with prompts.

Always inspect first.

---

# 41. Removing Empty Directories With `rmdir`

`rmdir` removes empty directories.

Create:

```bash
mkdir old-directory
```

Then:

```bash
rmdir old-directory
```

Verify:

```bash
ls
```

The directory should be gone.

---

# 42. Why `rmdir` May Fail

Suppose:

```text
old-directory/
|
+-- file.txt
```

Then:

```bash
rmdir old-directory
```

may fail with a message similar to:

```text
Directory not empty
```

This is intentional.

`rmdir` is designed for empty directories.

That makes it relatively safe compared with recursive deletion.

---

# 43. Recursive Deletion With `rm -r`

To remove a directory and everything beneath it, Linux provides recursive deletion:

```bash
rm -r directory
```

Suppose:

```text
old-project
|
+-- configs
|   |
|   +-- app.conf
|
+-- logs
    |
    +-- app.log
```

Running:

```bash
rm -r old-project
```

can remove:

* `old-project`
* `configs`
* `app.conf`
* `logs`
* `app.log`

Everything beneath the directory is affected.

This is why recursive deletion requires care.

---

# 44. Safer Recursive Practice

Create a disposable test directory:

```bash
mkdir -p delete-practice/one/two
```

Create files:

```bash
touch delete-practice/file1.txt
touch delete-practice/one/file2.txt
touch delete-practice/one/two/file3.txt
```

Inspect:

```bash
find delete-practice
```

You should see the structure.

Now remove it:

```bash
rm -r delete-practice
```

Verify:

```bash
ls
```

Only practice recursive deletion on directories you intentionally created for testing.

---

# 45. Why `rm -rf` Requires Extreme Care

You will frequently encounter:

```bash
rm -rf
```

The options generally mean:

```text
-r    recursive
-f    force
```

Combined:

```text
-rf
```

can recursively remove directory contents while suppressing many prompts and complaints.

That combination is powerful and dangerous.

A typo in:

```bash
rm -rf some-directory
```

can potentially remove much more than intended, especially when:

* Running as root
* Using variables
* Using wildcards
* Using absolute paths
* Working in important directories
* Running deletion from scripts

Do not use `rm -rf` simply because an online tutorial tells you to.

First understand:

* Your current directory
* The exact target
* Whether variables expand correctly
* Whether wildcards match more files than expected

For beginner practice, prefer safer alternatives where possible.

---

# 46. A Dangerous Mindset to Avoid

Do not think:

```text
The command failed, so I will add -f.
```

`-f` is not a generic "fix the command" option.

If an operation fails, investigate why.

For example:

```text
Permission denied
```

does not automatically mean:

```text
Use sudo and force.
```

The permission failure may be protecting important data.

Troubleshooting should come before increasing the power of a command.

---

# 47. File Extensions in Linux

Windows users may be accustomed to extensions such as:

```text
.txt
.exe
.docx
.jpg
```

Linux supports filenames with extensions, but it does not depend on extensions in exactly the same way.

Examples:

```text
application.log
deploy.sh
nginx.conf
backup.tar.gz
```

are useful names because the extensions communicate intent to humans and tools.

But Linux can also have files named:

```text
passwd
hosts
fstab
bash
```

without familiar extensions.

---

# 48. Extension Does Not Guarantee File Type

Suppose someone renames:

```text
image.jpg
```

to:

```text
image.txt
```

Changing the name does not magically convert the underlying data from an image into text.

Similarly, a shell script does not become valid merely because its name ends in:

```text
.sh
```

File extensions are useful conventions.

They do not always determine the actual content.

---

# 49. Identifying File Types With `file`

Linux provides:

```bash
file filename
```

to inspect what kind of data a file appears to contain.

Example:

```bash
file devops-project/configs/app.conf
```

You might see something similar to:

```text
ASCII text
```

depending on the file contents and system.

Try:

```bash
file /bin/ls
```

You may see output indicating that it is an executable binary.

Try:

```bash
file /etc/passwd
```

You may see that it is text.

This is useful when a filename or extension is misleading.

---

# 50. Hidden Files

You learned in Part 2 that Linux conventionally treats names beginning with:

```text
.
```

as hidden.

Create one:

```bash
touch .environment
```

Run:

```bash
ls
```

You may not see it.

Now:

```bash
ls -a
```

You should see:

```text
.environment
```

The file still exists.

It was simply omitted from the default `ls` output.

---

# 51. Hidden Directories

The same rule applies to directories.

Create:

```bash
mkdir .settings
```

Plain:

```bash
ls
```

may not show it.

Use:

```bash
ls -a
```

You should see:

```text
.settings
```

Many important user-specific directories are hidden.

Examples you may encounter include:

```text
.ssh
.git
.config
.cache
```

Do not assume hidden means unimportant.

---

# 52. Wildcards

Wildcards allow shell patterns to match multiple filenames.

This is powerful.

It is also one of the easiest ways to accidentally affect more files than intended.

The major beginner patterns include:

```text
*
?
[...]
```

---

# 53. The `*` Wildcard

An asterisk:

```text
*
```

matches zero or more characters in a filename component.

Suppose you have:

```text
app.log
error.log
access.log
notes.txt
config.txt
```

Run:

```bash
ls *.log
```

You may see:

```text
access.log
app.log
error.log
```

The pattern:

```text
*.log
```

means approximately:

```text
anything ending in .log
```

---

# 54. More `*` Examples

Suppose you have:

```text
app-production.conf
app-staging.conf
database.conf
```

Run:

```bash
ls app*
```

This may match:

```text
app-production.conf
app-staging.conf
```

Run:

```bash
ls *.conf
```

This may match:

```text
app-production.conf
app-staging.conf
database.conf
```

---

# 55. Wildcards With Destructive Commands

This requires special attention.

Suppose you run:

```bash
rm *.log
```

The shell first determines which filenames match:

```text
*.log
```

Then `rm` receives those filenames.

If five log files match, all five may be removed.

Before using a wildcard with `rm`, first inspect the matches using:

```bash
ls *.log
```

Only after confirming the list should you consider deleting them.

Use this safety pattern:

```text
Preview
   |
   v
Verify
   |
   v
Act
```

---

# 56. The `?` Wildcard

A question mark:

```text
?
```

matches exactly one character.

Suppose you have:

```text
log1.txt
log2.txt
log3.txt
log10.txt
```

Run:

```bash
ls log?.txt
```

This may match:

```text
log1.txt
log2.txt
log3.txt
```

but not:

```text
log10.txt
```

because `10` contains two characters.

---

# 57. Character Matching With `[]`

Square brackets can match one character from a set or range.

Suppose:

```text
server1.log
server2.log
server3.log
server4.log
```

Run:

```bash
ls server[12].log
```

This can match:

```text
server1.log
server2.log
```

A range can also be used:

```bash
ls server[1-3].log
```

This can match:

```text
server1.log
server2.log
server3.log
```

Another example:

```bash
ls file[a-c].txt
```

may match:

```text
filea.txt
fileb.txt
filec.txt
```

These patterns are expanded by the shell before many commands receive them.

---

# 58. Wildcard Troubleshooting

Suppose:

```bash
ls *.log
```

returns something similar to:

```text
No such file or directory
```

One possible explanation is simple:

There are no matching `.log` files in your current directory.

Check:

```bash
pwd
```

Then:

```bash
ls
```

Ask:

* Am I in the correct directory?
* Do the filenames actually end with `.log`?
* Is capitalization different?

Remember:

```text
app.LOG
```

does not automatically match:

```text
*.log
```

because Linux filenames are case-sensitive.

---

# 59. Finding Files With `find`

`find` searches directory trees.

Basic form:

```bash
find STARTING-PATH CONDITIONS
```

For example:

```bash
find . -name "application.log"
```

Break it down:

```text
find
```

Command.

```text
.
```

Start searching from the current directory.

```text
-name
```

Search by name.

```text
"application.log"
```

Target filename.

---

# 60. Searching Recursively

Suppose:

```text
project
|
+-- logs
|   |
|   +-- application.log
|
+-- backups
    |
    +-- application.log
```

Run:

```bash
find project -name "application.log"
```

Possible output:

```text
project/logs/application.log
project/backups/application.log
```

`find` walks through directories recursively.

This makes it much more powerful than simply running:

```bash
ls
```

in one directory.

---

# 61. Finding by Pattern

Use:

```bash
find . -name "*.log"
```

The quotes are important.

They prevent the shell from expanding the wildcard too early.

You are asking `find` itself to match names ending in `.log`.

You may receive:

```text
./logs/app.log
./logs/error.log
./archive/old.log
```

---

# 62. Case-Insensitive Search With `find`

Linux filenames are case-sensitive, and:

```bash
find . -name "*.log"
```

may not match:

```text
APP.LOG
```

GNU `find` commonly supports:

```bash
find . -iname "*.log"
```

where:

```text
-iname
```

performs case-insensitive name matching.

This can be useful when you do not know the capitalization.

---

# 63. Finding Directories

You can restrict results by type.

To search for directories:

```bash
find . -type d -name "logs"
```

Here:

```text
-type d
```

means directory.

To search for regular files:

```bash
find . -type f -name "*.conf"
```

Here:

```text
-type f
```

means regular file.

---

# 64. Be Careful Combining `find` With Deletion

You may encounter advanced commands online that automatically delete files returned by `find`.

Do not begin with that approach.

First search:

```bash
find . -name "*.tmp"
```

Inspect the output.

Only after understanding exactly what matches should you consider any destructive action.

Search and deletion should not be combined casually.

---

# 65. Finding Files With `locate`

Another file-searching tool is:

```bash
locate
```

Example:

```bash
locate nginx.conf
```

Unlike `find`, `locate` usually searches a prebuilt database of filenames rather than walking the filesystem live.

This can make it very fast.

However, there are important limitations.

---

# 66. `find` vs `locate`

## `find`

Searches the filesystem starting from a path.

Example:

```bash
find /etc -name "hosts"
```

Advantages:

* Current filesystem search
* Powerful filtering
* Can search by many attributes

## `locate`

Searches an indexed database.

Example:

```bash
locate hosts
```

Advantages:

* Very fast
* Convenient for filename searches

Limitation:

The database may not contain files created very recently until it is updated.

Also, `locate` may not be installed by default.

---

# 67. Updating the `locate` Database

On systems where `locate` is installed, its database is commonly updated using:

```bash
sudo updatedb
```

Do not run this simply because you see it here if you do not understand your environment.

It may require administrative privileges.

The important beginner concept is:

```text
find
Search filesystem now

locate
Search filename database
```

---

# 68. Finding Executables With `which`

Suppose you run:

```bash
which ls
```

You may see:

```text
/usr/bin/ls
```

This tells you which executable would be found through your current command-search path for `ls`.

Try:

```bash
which bash
```

Possible output:

```text
/usr/bin/bash
```

or another path depending on your system.

`which` is useful when you want to know which executable your shell is likely to run from your `PATH`.

You will learn `PATH` properly in Part 11.

---

# 69. A More Shell-Aware Alternative

You will also encounter:

```bash
command -v ls
```

This is often useful because shell commands are not always simple executable files.

For example, commands can also be:

* Shell built-ins
* Aliases
* Functions

You do not need to master this distinction yet.

For now, understand that:

```bash
which command-name
```

and:

```bash
command -v command-name
```

help investigate how a command is resolved.

---

# 70. Finding Information With `whereis`

`whereis` can locate certain files associated with a command, such as:

* Binary
* Source
* Manual-page locations

Example:

```bash
whereis ls
```

You may see something similar to:

```text
ls: /usr/bin/ls /usr/share/man/man1/ls.1.gz
```

The exact result depends on your system.

---

# 71. `find`, `locate`, `which`, and `whereis`

These commands solve different problems.

| Command   | Main beginner question                                      |
| --------- | ----------------------------------------------------------- |
| `find`    | Where is a file or directory under this path?               |
| `locate`  | Where might this filename exist according to the index?     |
| `which`   | Which executable is found for this command?                 |
| `whereis` | Where are the binary/manual-related files for this command? |

Do not treat them as interchangeable.

---

# 72. File Metadata

A file contains more than its visible content.

Linux also tracks metadata.

Metadata means information about the file.

Examples include:

* Name
* Type
* Size
* Owner
* Group
* Permissions
* Timestamps
* Inode-related information

You have already seen some metadata with:

```bash
ls -l
```

For example:

```text
-rw-r--r-- 1 ann ann 2048 Aug 26 10:30 application.log
```

This line describes the file rather than displaying its contents.

---

# 73. Inspecting More Metadata With `stat`

A useful command is:

```bash
stat filename
```

Example:

```bash
stat devops-project/logs/application.log
```

You may see information including:

* File size
* Blocks
* Inode
* Permissions
* Owner
* Group
* Access time
* Modification time
* Status-change time

Exact formatting depends on the system.

---

# 74. Understanding File Timestamps

Three important timestamps commonly discussed in Linux are:

* Access time
* Modification time
* Change time

These are often called:

```text
atime
mtime
ctime
```

---

# 75. Access Time

Access time, or:

```text
atime
```

relates to when file data was accessed.

Filesystem mount options and optimizations can affect exactly when access time is updated.

For beginners, think:

```text
atime
When file data was accessed
```

---

# 76. Modification Time

Modification time, or:

```text
mtime
```

relates to when the file's contents were last modified.

If you change:

```text
port=8080
```

to:

```text
port=9090
```

the file content changed.

Its modification time changes.

This is often the timestamp displayed by:

```bash
ls -l
```

---

# 77. Change Time

Change time, or:

```text
ctime
```

relates to when inode metadata or file status last changed.

For example, changing:

* Permissions
* Ownership

can change `ctime`.

A common beginner mistake is to think:

```text
ctime = creation time
```

On traditional Unix/Linux terminology, that is incorrect.

`ctime` means status-change time, not simply creation time.

Some filesystems may separately expose a birth or creation timestamp, but you should not confuse that with `ctime`.

---

# 78. Why Timestamps Matter

Timestamps become useful during troubleshooting.

Imagine:

```text
The application broke at 14:05.
```

You inspect configuration files and discover:

```text
nginx.conf modified at 14:03
```

That does not prove the configuration caused the incident.

But it gives you evidence worth investigating.

Timestamps help answer:

```text
What changed?
When did it change?
Does that timing correlate with the failure?
```

---

# 79. Introduction to Inodes

Linux filesystems commonly track files using internal structures called inodes.

An inode stores metadata about a filesystem object.

You can see inode numbers with:

```bash
ls -li
```

Example:

```text
123456 -rw-r--r-- 1 ann ann 100 Aug 26 10:30 app.conf
```

The first number may be the inode number.

At this stage, you do not need deep inode knowledge.

You need this concept because hard links make more sense when you understand that a filename and the underlying filesystem object are not exactly the same thing.

---

# 80. Filename vs Underlying File

A simplified mental model is:

```text
Filename
   |
   v
Inode
   |
   v
File data
```

The filename is a directory entry referring to an inode.

The inode describes the underlying filesystem object and points toward its data.

This becomes important when discussing links.

---

# 81. What Is a Link?

Linux supports links that allow filesystem objects to be referenced in different ways.

Two important types are:

* Hard links
* Symbolic links

They behave differently.

---

# 82. Hard Links Introduction

A hard link creates another filename referring to the same underlying inode.

Suppose:

```text
original.txt
```

refers to inode:

```text
5001
```

Create a hard link:

```bash
ln original.txt second-name.txt
```

Now conceptually:

```text
original.txt ----+
                 |
                 +---- inode 5001 ---- file data
                 |
second-name.txt -+
```

Both names refer to the same underlying file.

---

# 83. Creating a Hard Link With `ln`

Create a file:

```bash
printf "production configuration\n" > original.txt
```

Create the hard link:

```bash
ln original.txt second-name.txt
```

Inspect:

```bash
ls -li original.txt second-name.txt
```

You should notice that both names normally show the same inode number.

Display either:

```bash
cat original.txt
```

```bash
cat second-name.txt
```

Both show the same underlying data.

---

# 84. What Happens if One Hard-Link Name Is Removed?

Suppose:

```text
original.txt
second-name.txt
```

both refer to the same inode.

If you run:

```bash
rm original.txt
```

the underlying data does not necessarily disappear because:

```text
second-name.txt
```

still references it.

You can run:

```bash
cat second-name.txt
```

and the data should remain accessible.

This demonstrates why a hard link is not simply a separate copied file.

---

# 85. Hard-Link Limitations

Hard links have important limitations.

Generally:

* Hard links cannot cross filesystem boundaries.
* Ordinary users cannot normally create hard links to directories.
* They are less visually obvious than symbolic links.

As a beginner, the key idea is:

```text
Hard link
Another name for the same underlying inode
```

---

# 86. Symbolic Links

A symbolic link, also called:

```text
symlink
```

or:

```text
soft link
```

is different.

A symbolic link stores a reference to another path.

Conceptually:

```text
shortcut-name
     |
     v
/path/to/real-file
```

This is closer to the idea of a shortcut.

---

# 87. Creating Symbolic Links With `ln -s`

Create:

```bash
printf "server configuration\n" > server.conf
```

Now create a symbolic link:

```bash
ln -s server.conf current.conf
```

Inspect:

```bash
ls -l
```

You may see something similar to:

```text
current.conf -> server.conf
```

The arrow indicates the target.

Read:

```bash
cat current.conf
```

You should see:

```text
server configuration
```

The shell follows the symbolic link to the target.

---

# 88. Symbolic Links to Directories

Symbolic links can also point to directories.

Suppose:

```text
/opt/application/releases/version1
```

exists.

An administrator might conceptually create:

```text
current -> /opt/application/releases/version1
```

Applications or deployment scripts can then refer to:

```text
current
```

instead of hardcoding the release directory everywhere.

This pattern is useful in deployment systems.

---

# 89. Hard Link vs Symbolic Link

A simplified comparison:

| Hard link                                        | Symbolic link                             |
| ------------------------------------------------ | ----------------------------------------- |
| Another name for the same inode                  | References another path                   |
| Same underlying filesystem object                | Separate link object                      |
| Normally cannot cross filesystems                | Can reference paths on other filesystems  |
| Usually used for files                           | Can easily reference files or directories |
| Target data may remain after one name is deleted | Can break if target disappears            |

The distinction will become clearer with practice.

---

# 90. Broken Symbolic Links

Suppose:

```text
current.conf -> server.conf
```

Now delete the target:

```bash
rm server.conf
```

The symbolic link:

```text
current.conf
```

may still exist.

But its target is gone.

Try:

```bash
cat current.conf
```

You may receive:

```text
No such file or directory
```

The symbolic link is now broken.

Conceptually:

```text
current.conf
     |
     v
server.conf
     X
Target missing
```

---

# 91. Why Symbolic Links Break

A symbolic link points to a path.

If the target:

* Is deleted
* Is moved
* Is renamed
* Becomes unavailable

the symbolic link may no longer resolve correctly.

This is a common troubleshooting issue.

Use:

```bash
ls -l link-name
```

to inspect where the link points.

---

# 92. Relative vs Absolute Symbolic-Link Targets

Symbolic links can contain relative or absolute target paths.

For example:

```bash
ln -s server.conf current.conf
```

uses a relative target.

While:

```bash
ln -s /etc/example/server.conf current.conf
```

uses an absolute target.

Relative symbolic links are interpreted relative to the directory containing the symbolic link, not simply based on whichever directory your shell happens to be in later.

This distinction becomes important when links are moved or directory structures change.

---

# 93. Common File-Management Error: `No such file or directory`

Suppose:

```bash
cp application.log backups/
```

returns:

```text
No such file or directory
```

Possible causes include:

* `application.log` does not exist.
* `backups` does not exist.
* You are in the wrong directory.
* You misspelled something.
* Capitalization is wrong.
* A parent directory is missing.

Troubleshoot systematically.

First:

```bash
pwd
```

Then:

```bash
ls
```

Then inspect the source:

```bash
ls -l application.log
```

Then inspect the destination:

```bash
ls -ld backups
```

Do not guess.

Gather evidence.

---

# 94. Common Error: `Permission denied`

You may try to create, move, copy, or delete something and receive:

```text
Permission denied
```

This means Linux's access-control rules are preventing the operation.

Do not immediately use:

```bash
sudo
```

First investigate:

* Who am I?
* Who owns the file?
* Who owns the directory?
* What are the permissions?
* Should I actually be modifying this location?

Start with:

```bash
whoami
```

Then:

```bash
ls -l filename
```

For a directory:

```bash
ls -ld directory
```

Permissions are covered thoroughly in Part 5.

For now, recognize that `Permission denied` is useful information.

It tells you Linux blocked an operation.

---

# 95. Common Error: Destination Already Exists

Suppose:

```bash
mkdir logs
```

returns:

```text
File exists
```

Check:

```bash
ls -ld logs
```

Perhaps the directory already exists.

If you are building nested structures, this is one reason:

```bash
mkdir -p
```

is convenient.

Do not delete an existing directory merely because `mkdir` says it already exists.

First inspect what is there.

---

# 96. Common Error: `Directory not empty`

You run:

```bash
rmdir old-project
```

and receive:

```text
Directory not empty
```

That means the directory contains something.

Inspect:

```bash
ls -la old-project
```

Then decide what should happen.

Do not automatically switch to:

```bash
rm -rf old-project
```

without inspecting the contents.

The error may have prevented accidental data loss.

---

# 97. Common Error: Copying a Directory Without `-r`

You run:

```bash
cp project project-backup
```

and receive a message indicating that the directory was not copied.

The issue is that directories contain nested content.

Use recursive copying when that is genuinely your intention:

```bash
cp -r project project-backup
```

Then verify:

```bash
find project-backup
```

---

# 98. Common Error: Confusing Source and Destination

Suppose you intend:

```text
Copy app.conf into backups.
```

Correct:

```bash
cp app.conf backups/
```

But you accidentally run:

```bash
cp backups/app.conf app.conf
```

Now you may overwrite your current configuration with the backup.

This is why you should mentally read:

```text
cp SOURCE DESTINATION
```

before executing important copy operations.

---

# 99. Common Error: Wildcard Matches Too Much

Suppose a directory contains:

```text
application.log
error.log
audit.log
important.log
```

You intend to remove one file but run:

```bash
rm *.log
```

The pattern matches all four.

Preview first:

```bash
ls *.log
```

Then use the most specific target possible.

A command that is shorter is not automatically safer.

---

# 100. Common Error: Broken Symbolic Link

You run:

```bash
cat current.conf
```

and receive:

```text
No such file or directory
```

But:

```bash
ls
```

shows:

```text
current.conf
```

That seems confusing.

Inspect:

```bash
ls -l current.conf
```

You might discover:

```text
current.conf -> server.conf
```

but:

```text
server.conf
```

no longer exists.

The visible entry exists.

Its target does not.

That is a broken symbolic link.

---

# 101. Safe File-Management Practices

File-management commands can be simple but destructive.

Develop safe habits early.

## 1. Check Your Location

Before a destructive operation:

```bash
pwd
```

## 2. Inspect the Target

Use:

```bash
ls -la
```

or:

```bash
ls -ld directory
```

## 3. Preview Wildcards

Before:

```bash
rm *.log
```

run:

```bash
ls *.log
```

## 4. Back Up Important Configuration

Before changing important files, consider creating a backup:

```bash
cp app.conf app.conf.backup
```

## 5. Avoid Root Unless Required

More privileges increase the potential impact of mistakes.

## 6. Do Not Add `-f` Without Understanding the Failure

Investigate first.

## 7. Verify After Every Important Operation

After copying:

```bash
ls -l destination
```

After moving:

```bash
ls -l destination
```

After deleting:

```bash
ls
```

After creating:

```bash
ls -ld directory
```

## 8. Prefer Specific Paths

This:

```bash
rm old-application.log
```

is more specific than:

```bash
rm *.log
```

Specific commands reduce accidental scope.

---

# 102. A File-Management Troubleshooting Framework

When a file operation fails, use this process:

```text
Observe the error
       |
       v
Check current directory
       |
       v
Verify source exists
       |
       v
Verify destination exists
       |
       v
Check spelling and case
       |
       v
Inspect file type
       |
       v
Inspect permissions
       |
       v
Check wildcard expansion
       |
       v
Fix the actual cause
       |
       v
Verify
```

Useful commands include:

```bash
pwd
```

```bash
ls -la
```

```bash
ls -ld directory
```

```bash
file filename
```

```bash
stat filename
```

```bash
find . -name "filename"
```

Troubleshooting is evidence gathering.

Do not randomly change things until the error disappears.

---

# 103. Practical Project Directory Lab

Now combine the entire part into one hands-on project.

## Scenario

You are preparing a simple application workspace on a Linux server.

The application needs:

* Configuration
* Logs
* Scripts
* Backups
* Archived logs

You will create the structure, populate it, inspect it, copy configuration, move logs, create links, find files, and safely clean temporary resources.

---

# 104. Lab Step 1: Create the Workspace

Return home:

```bash
cd ~
```

Create:

```bash
mkdir -p linux-file-lab
```

Enter it:

```bash
cd linux-file-lab
```

Verify:

```bash
pwd
```

---

# 105. Lab Step 2: Create the Directory Structure

Create:

```bash
mkdir -p application/{configs,logs/archive,scripts,backups}
```

Inspect:

```bash
find application -type d
```

Your structure should resemble:

```text
application
|
+-- configs
|
+-- logs
|   |
|   +-- archive
|
+-- scripts
|
+-- backups
```

---

# 106. Lab Step 3: Create Application Files

Run:

```bash
touch application/configs/app.conf
touch application/configs/database.conf
touch application/logs/application.log
touch application/logs/error.log
touch application/scripts/deploy.sh
```

Verify:

```bash
find application -type f
```

You should see all the files.

---

# 107. Lab Step 4: Add Configuration Content

Run:

```bash
printf "environment=production\nport=8080\n" > application/configs/app.conf
```

Then:

```bash
printf "host=db.internal\nport=5432\n" > application/configs/database.conf
```

Inspect:

```bash
cat application/configs/app.conf
```

Then:

```bash
cat application/configs/database.conf
```

---

# 108. Lab Step 5: Add Log Data

Run:

```bash
printf "Application starting\nApplication ready\nUser login successful\nWarning: high memory\nError: database timeout\n" > application/logs/application.log
```

Run:

```bash
printf "Database timeout\nConnection retry failed\n" > application/logs/error.log
```

Inspect the application log:

```bash
cat application/logs/application.log
```

View the first two lines:

```bash
head -n 2 application/logs/application.log
```

View the last two:

```bash
tail -n 2 application/logs/application.log
```

Open interactively:

```bash
less application/logs/application.log
```

Press:

```text
q
```

to exit.

---

# 109. Lab Step 6: Back Up Configuration

Copy:

```bash
cp application/configs/app.conf application/backups/app.conf.backup
```

Verify:

```bash
ls -l application/backups
```

Compare:

```bash
cat application/configs/app.conf
```

```bash
cat application/backups/app.conf.backup
```

The original and backup should contain the same data.

---

# 110. Lab Step 7: Back Up the Entire Configuration Directory

Run:

```bash
cp -r application/configs application/backups/configs-backup
```

Inspect:

```bash
find application/backups
```

You should see the copied configuration directory and its files.

---

# 111. Lab Step 8: Archive a Log by Moving It

Create an old log:

```bash
touch application/logs/application-old.log
```

Move it:

```bash
mv application/logs/application-old.log application/logs/archive/
```

Verify the source:

```bash
ls application/logs
```

Verify the destination:

```bash
ls application/logs/archive
```

The file should exist only in the archive location.

---

# 112. Lab Step 9: Rename the Archived Log

Run:

```bash
mv application/logs/archive/application-old.log application/logs/archive/application-previous.log
```

Verify:

```bash
ls application/logs/archive
```

The file has been renamed.

---

# 113. Lab Step 10: Create a Hidden Environment File

Run:

```bash
touch application/.environment
```

Now:

```bash
ls application
```

You may not see it.

Run:

```bash
ls -la application
```

You should see:

```text
.environment
```

This confirms how hidden files work.

---

# 114. Lab Step 11: Identify File Types

Run:

```bash
file application/configs/app.conf
```

Then:

```bash
file application/scripts/deploy.sh
```

Then:

```bash
file /bin/ls
```

Compare the results.

Do not assume a filename alone tells you the real file type.

---

# 115. Lab Step 12: Practice Wildcards

Find configuration files in the configuration directory:

```bash
ls application/configs/*.conf
```

Find logs:

```bash
ls application/logs/*.log
```

Create:

```bash
touch application/logs/server1.log
touch application/logs/server2.log
touch application/logs/server3.log
touch application/logs/server10.log
```

Try:

```bash
ls application/logs/server?.log
```

Observe which names match.

Then:

```bash
ls application/logs/server[1-3].log
```

Observe the result.

---

# 116. Lab Step 13: Find Files

Search for:

```text
app.conf
```

using:

```bash
find application -name "app.conf"
```

Find all configuration files:

```bash
find application -type f -name "*.conf"
```

Find all logs:

```bash
find application -type f -name "*.log"
```

Find directories called:

```text
archive
```

using:

```bash
find application -type d -name "archive"
```

---

# 117. Lab Step 14: Inspect Metadata

Run:

```bash
ls -l application/configs/app.conf
```

Then:

```bash
stat application/configs/app.conf
```

Look for:

* Size
* Owner
* Group
* Permissions
* Modification time
* Change time
* Access-related information

You do not need to memorize every field yet.

The goal is to recognize that a file has metadata beyond its content.

---

# 118. Lab Step 15: Create a Hard Link

Create:

```bash
ln application/configs/app.conf app-hardlink.conf
```

Inspect inode numbers:

```bash
ls -li application/configs/app.conf app-hardlink.conf
```

The inode numbers should normally match.

Display both:

```bash
cat application/configs/app.conf
```

```bash
cat app-hardlink.conf
```

Both names reference the same underlying file.

---

# 119. Lab Step 16: Create a Symbolic Link

Create:

```bash
ln -s application/configs/app.conf current-app.conf
```

Inspect:

```bash
ls -l current-app.conf
```

You should see an arrow pointing toward the target.

Read:

```bash
cat current-app.conf
```

The content should come from the target file.

---

# 120. Lab Step 17: Practice a Broken Symbolic Link Safely

Create:

```bash
touch temporary-target.txt
```

Create:

```bash
ln -s temporary-target.txt temporary-link.txt
```

Verify:

```bash
ls -l temporary-link.txt
```

Now delete only the target:

```bash
rm temporary-target.txt
```

Inspect:

```bash
ls -l temporary-link.txt
```

Try:

```bash
cat temporary-link.txt
```

The link exists, but its target does not.

You have intentionally created a broken symbolic link.

Remove the broken link:

```bash
rm temporary-link.txt
```

---

# 121. Lab Step 18: Practice Safe Deletion

Create:

```bash
mkdir -p deletion-test/nested
```

Create:

```bash
touch deletion-test/file1.txt
touch deletion-test/nested/file2.txt
```

Inspect:

```bash
find deletion-test
```

Try:

```bash
rmdir deletion-test
```

It should fail because the directory is not empty.

This failure protects you from accidentally deleting nested content.

Now, because this directory was deliberately created for deletion practice, remove it recursively:

```bash
rm -r deletion-test
```

Verify:

```bash
ls
```

---

# 122. Lab Step 19: Practice Wildcard Safety

Create:

```bash
touch temp1.log temp2.log temp3.log important.txt
```

Before deleting anything, preview:

```bash
ls *.log
```

You should see:

```text
temp1.log
temp2.log
temp3.log
```

Now you know exactly what:

```text
*.log
```

matches.

For this controlled exercise, remove:

```bash
rm *.log
```

Verify:

```bash
ls
```

`important.txt` should remain.

The lesson is not:

```text
Wildcards are safe.
```

The lesson is:

```text
Preview wildcard expansion before destructive operations.
```

---

# 123. Lab Step 20: Final Inspection

Run:

```bash
pwd
```

Then:

```bash
find application
```

Then:

```bash
ls -la
```

Then:

```bash
find application -type f -name "*.conf"
```

Then:

```bash
find application -type f -name "*.log"
```

You have now created and managed a realistic directory structure.

---

# 124. Troubleshooting Challenge 1

You run:

```bash
cp app.conf backups/
```

and receive:

```text
cp: cannot stat 'app.conf': No such file or directory
```

What should you do?

Do not add `sudo`.

Start with:

```bash
pwd
```

Then:

```bash
ls -la
```

Search if necessary:

```bash
find . -name "app.conf"
```

Possible root cause:

The file exists, but not in your current directory.

---

# 125. Troubleshooting Challenge 2

You run:

```bash
rmdir logs
```

and receive:

```text
Directory not empty
```

What should you do?

Inspect:

```bash
ls -la logs
```

Determine what is inside.

Only then decide whether the contents should:

* Remain
* Be moved
* Be archived
* Be deleted

Do not automatically force deletion.

---

# 126. Troubleshooting Challenge 3

You run:

```bash
cat current.conf
```

and receive:

```text
No such file or directory
```

Yet:

```bash
ls
```

shows:

```text
current.conf
```

Inspect:

```bash
ls -l current.conf
```

If you see:

```text
current.conf -> production.conf
```

check:

```bash
ls -l production.conf
```

If the target is missing, you have a broken symbolic link.

---

# 127. Troubleshooting Challenge 4

You run:

```bash
ls *.LOG
```

but the files are:

```text
app.log
error.log
access.log
```

Why did the pattern fail?

Linux filenames are case-sensitive.

```text
*.LOG
```

and:

```text
*.log
```

are different patterns.

---

# 128. What You Should Now Understand

Before moving to Part 4, you should understand:

* How to create files with `touch`
* What `touch` actually does
* How to create directories with `mkdir`
* How to create nested directories
* Why `mkdir -p` is useful
* How to view small files with `cat`
* How to browse larger files with `less`
* What `more` does
* How to view the beginning of a file with `head`
* How to view the end with `tail`
* How `tail -f` follows a changing file
* How to copy files with `cp`
* Source vs destination
* How to copy directories with `cp -r`
* How to move files with `mv`
* How `mv` also renames files and directories
* How to remove files with `rm`
* How `rmdir` differs from `rm -r`
* Why recursive deletion is powerful
* Why `rm -rf` requires extreme care
* How Linux treats file extensions
* Why extensions do not guarantee file type
* How to use `file`
* How hidden files and directories work
* How `*` works
* How `?` works
* How character matching with `[]` works
* Why wildcards are dangerous with destructive commands
* How `find` searches directory trees
* How `locate` differs from `find`
* What `which` does
* What `whereis` does
* What file metadata is
* What `stat` displays
* The basic meaning of atime, mtime, and ctime
* What an inode is at a beginner level
* What a hard link is
* What a symbolic link is
* How `ln` creates hard links
* How `ln -s` creates symbolic links
* Why symbolic links can become broken
* How to troubleshoot common file-management failures
* How to verify file operations safely

---

# 129. Commands You Should Now Recognize

You should be comfortable recognizing and beginning to use:

```bash
touch
mkdir
mkdir -p
cat
less
more
head
tail
tail -f
cp
cp -r
mv
rm
rm -i
rmdir
rm -r
file
find
locate
which
whereis
command -v
stat
ln
ln -s
```

And shell patterns such as:

```text
*
?
[1-3]
[a-c]
```

You should not simply memorize these.

You should know what problem each one solves.

---

# 130. Beginner Knowledge Check

Try answering these before checking the answers.

1. What does `touch` do if a file does not exist?

2. What can `touch` do if the file already exists?

3. What does `mkdir` mean?

4. Why would you use `mkdir -p`?

5. Which command is convenient for displaying a small text file?

6. Which command is useful for interactively browsing a large file?

7. What does `head` display?

8. What does `tail` display?

9. Why is `tail -f` useful for application logs?

10. What is the basic syntax of `cp`?

11. Does `cp` normally remove the original?

12. Why is `cp -r` needed for directories?

13. What does `mv` do?

14. How can `mv` rename a file?

15. What does `rm` do?

16. Why should you treat `rm` as potentially permanent?

17. What does `rmdir` normally remove?

18. Why might `rmdir` refuse to remove a directory?

19. What does recursive deletion mean?

20. Why is `rm -rf` dangerous?

21. Do Linux files require extensions?

22. Does renaming a file from `.jpg` to `.txt` convert its contents to text?

23. What does `file` help determine?

24. What makes a filename conventionally hidden?

25. What does `*` match?

26. What does `?` match?

27. What might `[1-3]` match?

28. Why should you preview wildcards before using `rm`?

29. What does `find` do?

30. What is one important difference between `find` and `locate`?

31. What does `which` help identify?

32. What does `whereis` provide?

33. What is file metadata?

34. What is `mtime`?

35. Is `ctime` simply file creation time?

36. What is an inode at a beginner level?

37. What is a hard link?

38. What is a symbolic link?

39. What command creates a hard link?

40. What option makes `ln` create a symbolic link?

41. Why can a symbolic link become broken?

42. What should you check first when a file command says `No such file or directory`?

43. Should `Permission denied` automatically cause you to rerun the command with `sudo`?

44. What is the difference between copying and moving?

45. Why should important file operations always be verified?

---

# 131. Knowledge Check Answers

1. It normally creates an empty file.

2. It can update timestamps on the existing file.

3. `mkdir` means make directory.

4. It creates missing parent directories when building nested paths and is useful when directories may already exist.

5.

```bash
cat
```

6.

```bash
less
```

7. The beginning of a file.

8. The end of a file.

9. It remains attached to the file and displays newly appended lines, making it useful for real-time log monitoring.

10.

```text
cp SOURCE DESTINATION
```

11. No. The original normally remains.

12. Because directories may contain nested files and directories that must be processed recursively.

13. It moves a file or directory from one path to another.

14. Move the old name to a new name:

```bash
mv old-name new-name
```

15. It removes files.

16. Command-line removal does not normally provide a convenient recycle-bin recovery workflow.

17. Empty directories.

18. Because it contains files or directories.

19. It means processing a directory and its contents, including nested directories.

20. It combines recursive deletion with forceful behavior and can remove large directory trees with little protection against mistakes.

21. No.

22. No.

23. It helps identify the actual type of data contained in a file.

24. Its name begins with a dot.

25. Zero or more characters within a filename component.

26. Exactly one character.

27. One character in the range 1 through 3.

28. Because a wildcard may match more files than you intended.

29. It searches through a directory tree using conditions such as filename and type.

30. `find` searches the filesystem from a specified path. `locate` normally searches a prebuilt filename database.

31. The executable found through the current command-search path.

32. Locations associated with a command, such as its binary and manual pages.

33. Information describing a file, such as size, ownership, permissions, and timestamps.

34. Modification time, generally relating to the last change to file contents.

35. No. Traditional Linux `ctime` refers to status-change time.

36. A filesystem data structure that stores metadata about an object and helps reference its underlying data.

37. Another filename referring to the same underlying inode.

38. A separate filesystem object containing a reference to another path.

39.

```bash
ln source link-name
```

40.

```text
-s
```

Example:

```bash
ln -s target link-name
```

41. Because the path it references can be deleted, moved, renamed, or otherwise become unavailable.

42. Start by checking your current location and whether the source and destination paths actually exist.

Useful commands include:

```bash
pwd
ls -la
```

43. No. Investigate why access is denied before increasing privileges.

44. Copying leaves the original and creates another copy. Moving relocates the original filesystem entry to another path or name.

45. Because a successful-looking command does not replace verification. You should confirm that the intended file exists in the intended location and that unintended files were not affected.

---



| #  | Question                                                                         | Answer                                                                                                                                                                                     |
| -- | -------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| 1  | What does touch do if a file does not exist?                                     | It normally creates an empty file.                                                                                                                                                         |
| 2  | What can touch do if the file already exists?                                    | It can update timestamps on the existing file.                                                                                                                                             |
| 3  | What does mkdir mean?                                                            | mkdir means make directory.                                                                                                                                                                |
| 4  | Why would you use mkdir -p?                                                      | It creates missing parent directories when building nested paths and is useful when directories may already exist.                                                                         |
| 5  | Which command is convenient for displaying a small text file?                    | cat                                                                                                                                                                                        |
| 6  | Which command is useful for interactively browsing a large file?                 | less                                                                                                                                                                                       |
| 7  | What does head display?                                                          | The beginning of a file.                                                                                                                                                                   |
| 8  | What does tail display?                                                          | The end of a file.                                                                                                                                                                         |
| 9  | Why is tail -f useful for application logs?                                      | It remains attached to the file and displays newly appended lines, making it useful for real-time log monitoring.                                                                          |
| 10 | What is the basic syntax of cp?                                                  | cp SOURCE DESTINATION                                                                                                                                                                      |
| 11 | Does cp normally remove the original?                                            | No. The original normally remains.                                                                                                                                                         |
| 12 | Why is cp -r needed for directories?                                             | Because directories may contain nested files and directories that must be processed recursively.                                                                                           |
| 13 | What does mv do?                                                                 | It moves a file or directory from one path to another.                                                                                                                                     |
| 14 | How can mv rename a file?                                                        | Move the old name to a new name: mv old-name new-name                                                                                                                                      |
| 15 | What does rm do?                                                                 | It removes files.                                                                                                                                                                          |
| 16 | Why should you treat rm as potentially permanent?                                | Command-line removal does not normally provide a convenient recycle-bin recovery workflow.                                                                                                 |
| 17 | What does rmdir normally remove?                                                 | Empty directories.                                                                                                                                                                         |
| 18 | Why might rmdir refuse to remove a directory?                                    | Because it contains files or directories.                                                                                                                                                  |
| 19 | What does recursive deletion mean?                                               | It means processing a directory and its contents, including nested directories.                                                                                                            |
| 20 | Why is rm -rf dangerous?                                                         | It combines recursive deletion with forceful behavior and can remove large directory trees with little protection against mistakes.                                                        |
| 21 | Do Linux files require extensions?                                               | No.                                                                                                                                                                                        |
| 22 | Does renaming a file from .jpg to .txt convert its contents to text?             | No.                                                                                                                                                                                        |
| 23 | What does file help determine?                                                   | It helps identify the actual type of data contained in a file.                                                                                                                             |
| 24 | What makes a filename conventionally hidden?                                     | Its name begins with a dot.                                                                                                                                                                |
| 25 | What does \* match?                                                              | Zero or more characters within a filename component.                                                                                                                                       |
| 26 | What does ? match?                                                               | Exactly one character.                                                                                                                                                                     |
| 27 | What might \[1-3] match?                                                         | One character in the range 1 through 3.                                                                                                                                                    |
| 28 | Why should you preview wildcards before using rm?                                | Because a wildcard may match more files than you intended.                                                                                                                                 |
| 29 | What does find do?                                                               | It searches through a directory tree using conditions such as filename and type.                                                                                                           |
| 30 | What is one important difference between find and locate?                        | find searches the filesystem from a specified path. locate normally searches a prebuilt filename database.                                                                                 |
| 31 | What does which help identify?                                                   | The executable found through the current command-search path.                                                                                                                              |
| 32 | What does whereis provide?                                                       | Locations associated with a command, such as its binary and manual pages.                                                                                                                  |
| 33 | What is file metadata?                                                           | Information describing a file, such as size, ownership, permissions, and timestamps.                                                                                                       |
| 34 | What is mtime?                                                                   | Modification time, generally relating to the last change to file contents.                                                                                                                 |
| 35 | Is ctime simply file creation time?                                              | No. Traditional Linux ctime refers to status-change time.                                                                                                                                  |
| 36 | What is an inode at a beginner level?                                            | A filesystem data structure that stores metadata about an object and helps reference its underlying data.                                                                                  |
| 37 | What is a hard link?                                                             | Another filename referring to the same underlying inode.                                                                                                                                   |
| 38 | What is a symbolic link?                                                         | A separate filesystem object containing a reference to another path.                                                                                                                       |
| 39 | What command creates a hard link?                                                | ln source link-name                                                                                                                                                                        |
| 40 | What option makes ln create a symbolic link?                                     | -s (Example: ln -s target link-name)                                                                                                                                                       |
| 41 | Why can a symbolic link become broken?                                           | Because the path it references can be deleted, moved, renamed, or otherwise become unavailable.                                                                                            |
| 42 | What should you check first when a file command says No such file or directory?  | Start by checking your current location and whether the source and destination paths actually exist. Useful commands include: pwd, ls -la                                                  |
| 43 | Should Permission denied automatically cause you to rerun the command with sudo? | No. Investigate why access is denied before increasing privileges.                                                                                                                         |
| 44 | What is the difference between copying and moving?                               | Copying leaves the original and creates another copy. Moving relocates the original filesystem entry to another path or name.                                                              |
| 45 | Why should important file operations always be verified?                         | Because a successful-looking command does not replace verification. You should confirm that the intended file exists in the intended location and that unintended files were not affected. |
---

# 132. Final Practice Challenge

Before moving to Part 4, try performing these tasks without looking at the previous examples:

1. Create a directory called `linux-practice`.

2. Inside it, create:

```text
configs
logs
scripts
backups
```

3. Inside `logs`, create:

```text
archive
```

4. Create three `.log` files.

5. Create two `.conf` files.

6. Display one configuration file with `cat`.

7. Open it with `less`.

8. Display its first two lines.

9. Display its last two lines.

10. Copy one configuration file into `backups`.

11. Copy the entire `configs` directory.

12. Move one log into `logs/archive`.

13. Rename the moved log.

14. Create a hidden file.

15. Display the hidden file using `ls`.

16. Use a wildcard to list all `.log` files.

17. Use `find` to locate every `.conf` file.

18. Use `file` to inspect a configuration file.

19. Use `stat` to inspect its metadata.

20. Create a hard link.

21. Confirm that the hard link and original have the same inode.

22. Create a symbolic link.

23. Inspect the symbolic link with `ls -l`.

24. Intentionally create a broken symbolic link.

25. Diagnose why it is broken.

26. Create a disposable nonempty directory.

27. Try removing it with `rmdir`.

28. Explain why the operation fails.

29. Remove the disposable directory recursively.

30. Verify the final filesystem structure.

If you can perform these tasks and explain what each command is doing, you have moved beyond simply recognizing Linux file-management commands.

You are beginning to understand how Linux filesystem operations work.
---

### All commands and command forms in Part 3

| Command                          | What it does                                                                                                                                                      | How to use it                                                   | Example from the note                                                            |
| -------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------- | -------------------------------------------------------------------------------- |
| `pwd`                            | Shows your current working directory.                                                                                                                             | Run it before file operations to confirm where you are.         | `pwd`                                                                            |
| `cd ~`                           | Returns to your home directory.                                                                                                                                   | Use `cd` with `~`.                                              | `cd ~`                                                                           |
| `cd directory`                   | Moves into a directory.                                                                                                                                           | Put the directory name or path after `cd`.                      | `cd linux-part3`                                                                 |
| `ls`                             | Lists files and directories in the current directory.                                                                                                             | Run it by itself, or give it a path.                            | `ls`                                                                             |
| `ls path`                        | Lists contents of another directory without entering it.                                                                                                          | Put the path after `ls`.                                        | `ls devops-project`                                                              |
| `ls -l`                          | Shows a detailed long listing.                                                                                                                                    | Add `-l`.                                                       | `ls -l`                                                                          |
| `ls -l file`                     | Shows detailed information about a particular file.                                                                                                               | Put the filename after `ls -l`.                                 | `ls -l application.log`                                                          |
| `ls -ld directory`               | Shows information about the directory itself rather than listing its contents.                                                                                    | Use `-ld` followed by the directory.                            | `ls -ld backups`                                                                 |
| `ls -a`                          | Shows all entries, including hidden files and directories.                                                                                                        | Add `-a`.                                                       | `ls -a`                                                                          |
| `ls -la`                         | Shows hidden entries together with detailed information.                                                                                                          | Combine `-l` and `-a`.                                          | `ls -la`                                                                         |
| `ls -li`                         | Shows a long listing including inode numbers.                                                                                                                     | Add `-i` to long listing.                                       | `ls -li`                                                                         |
| `ls -li file1 file2`             | Compares inode information for specified files.                                                                                                                   | Give multiple filenames after `ls -li`.                         | `ls -li original.txt second-name.txt`                                            |
| `touch filename`                 | Creates an empty file if it does not exist. If it already exists, updates its timestamps.                                                                         | Put the filename after `touch`.                                 | `touch application.log`                                                          |
| `touch file1 file2 file3`        | Creates or touches multiple files at once.                                                                                                                        | Give multiple filenames as arguments.                           | `touch app.log error.log access.log`                                             |
| `touch path/file`                | Creates a file at a specified path, provided the parent directory exists.                                                                                         | Give `touch` the relative or absolute path.                     | `touch logs/application.log`                                                     |
| `touch .filename`                | Creates a hidden file because its name begins with `.`.                                                                                                           | Start the filename with a dot.                                  | `touch .environment`                                                             |
| `mkdir directory`                | Creates a directory.                                                                                                                                              | Put the new directory name after `mkdir`.                       | `mkdir logs`                                                                     |
| `mkdir dir1 dir2 dir3`           | Creates several directories at once.                                                                                                                              | Give multiple directory names.                                  | `mkdir scripts configs backups`                                                  |
| `mkdir path`                     | Attempts to create a directory at the specified path. Parent directories must already exist.                                                                      | Supply the directory path.                                      | `mkdir project/environments/production/nginx`                                    |
| `mkdir -p path`                  | Creates a directory and any missing parent directories.                                                                                                           | Add `-p` before the path.                                       | `mkdir -p project/environments/production/nginx`                                 |
| `mkdir -p dir/{a,b,c}`           | Uses brace expansion with `mkdir -p` to create several related directories.                                                                                       | Put comma-separated names inside braces.                        | `mkdir -p devops-project/{configs,logs,scripts,backups}`                         |
| `mkdir .directory`               | Creates a hidden directory.                                                                                                                                       | Begin the directory name with `.`.                              | `mkdir .settings`                                                                |
| `printf "text\n" > file`         | Produces text and writes it into a file. `\n` creates new lines and `>` writes the output to the file.                                                            | Put the text inside quotes and the destination after `>`.       | `printf "environment=production\nport=8080\n" > devops-project/configs/app.conf` |
| `cat file`                       | Displays the contents of a file. Best suited to small files.                                                                                                      | Put the filename after `cat`.                                   | `cat devops-project/logs/application.log`                                        |
| `cat file1 file2`                | Displays multiple files one after another.                                                                                                                        | Give multiple filenames.                                        | `cat file1 file2`                                                                |
| `less file`                      | Opens a file in an interactive viewer. Useful for larger files and logs.                                                                                          | Put the filename after `less`.                                  | `less devops-project/logs/application.log`                                       |
| `/word` in `less`                | Searches for a word while inside `less`.                                                                                                                          | Type `/` followed by the search term, then Enter.               | `/Error`                                                                         |
| `n` in `less`                    | Goes to the next search match.                                                                                                                                    | Press `n` while using `less`.                                   | `n`                                                                              |
| `Space` in `less`                | Moves forward through the file.                                                                                                                                   | Press Space.                                                    | `Space`                                                                          |
| `b` in `less`                    | Moves backward through the file.                                                                                                                                  | Press `b`.                                                      | `b`                                                                              |
| `q` in `less`                    | Exits the `less` viewer.                                                                                                                                          | Press `q`.                                                      | `q`                                                                              |
| `more file`                      | Displays a file one screen at a time using a basic pager.                                                                                                         | Put the filename after `more`.                                  | `more devops-project/logs/application.log`                                       |
| `head file`                      | Displays the beginning of a file, normally the first 10 lines.                                                                                                    | Put the filename after `head`.                                  | `head devops-project/logs/application.log`                                       |
| `head -n NUMBER file`            | Displays a specified number of lines from the beginning.                                                                                                          | Replace `NUMBER` with the number of lines.                      | `head -n 3 devops-project/logs/application.log`                                  |
| `tail file`                      | Displays the end of a file, normally the last 10 lines.                                                                                                           | Put the filename after `tail`.                                  | `tail devops-project/logs/application.log`                                       |
| `tail -n NUMBER file`            | Displays a specified number of lines from the end.                                                                                                                | Replace `NUMBER` with the number required.                      | `tail -n 3 devops-project/logs/application.log`                                  |
| `tail -f file`                   | Follows a file and displays new lines as they are added. Useful for monitoring logs.                                                                              | Add `-f` before the filename.                                   | `tail -f devops-project/logs/application.log`                                    |
| `Ctrl+C`                         | Stops a running foreground command such as `tail -f`.                                                                                                             | Hold Ctrl and press C.                                          | `Ctrl+C`                                                                         |
| `cp SOURCE DESTINATION`          | Copies a file. The original remains.                                                                                                                              | Give the source first and destination second.                   | `cp app.conf backup.conf`                                                        |
| `cp file directory/`             | Copies a file into a directory while retaining its filename.                                                                                                      | Give the file first and destination directory second.           | `cp app.conf backups/`                                                           |
| `cp file new-name`               | Copies a file while giving the copy another name.                                                                                                                 | Make the destination a new filename.                            | `cp app.conf app.conf.backup`                                                    |
| `cp -i SOURCE DESTINATION`       | Copies interactively and can prompt before overwriting an existing destination.                                                                                   | Add `-i`.                                                       | `cp -i app.conf backups/app.conf`                                                |
| `cp -r SOURCE DESTINATION`       | Recursively copies a directory and its contents.                                                                                                                  | Add `-r` when copying a directory tree.                         | `cp -r configs configs-backup`                                                   |
| `mv SOURCE DESTINATION`          | Moves a file or directory.                                                                                                                                        | Give the source first and destination second.                   | `mv application.log logs/`                                                       |
| `mv old-name new-name`           | Renames a file or directory.                                                                                                                                      | Use the old name as source and new name as destination.         | `mv app.log application.log`                                                     |
| `mv file directory/new-name`     | Moves and renames a file in one operation.                                                                                                                        | Make the destination contain both the new path and filename.    | `mv app.conf configs/production.conf`                                            |
| `mv -i SOURCE DESTINATION`       | Moves interactively and can request confirmation before an overwrite.                                                                                             | Add `-i`.                                                       | `mv -i source destination`                                                       |
| `rm file`                        | Removes a file. The note warns that this should be treated as potentially permanent.                                                                              | Put the target filename after `rm`.                             | `rm temporary.txt`                                                               |
| `rm -i file`                     | Requests interactive confirmation when removing a file.                                                                                                           | Add `-i`.                                                       | `rm -i filename`                                                                 |
| `rmdir directory`                | Removes an empty directory.                                                                                                                                       | Put the empty directory name after `rmdir`.                     | `rmdir old-directory`                                                            |
| `rm -r directory`                | Recursively removes a directory and everything beneath it.                                                                                                        | Add `-r`.                                                       | `rm -r old-project`                                                              |
| `rm -rf directory`               | Recursively and forcibly removes a directory tree. The note specifically identifies this as powerful and dangerous.                                               | `-r` means recursive and `-f` means force.                      | `rm -rf old-project`                                                             |
| `rm *.log`                       | Removes all filenames matching the `.log` wildcard pattern. The note says to preview matches first.                                                               | Use the wildcard only after verifying what it matches.          | `rm *.log`                                                                       |
| `file filename`                  | Inspects the actual type of data a file appears to contain.                                                                                                       | Put the filename or path after `file`.                          | `file devops-project/configs/app.conf`                                           |
| `file /bin/ls`                   | Inspects the file type of the `ls` binary.                                                                                                                        | Give the path to `file`.                                        | `file /bin/ls`                                                                   |
| `file /etc/passwd`               | Inspects what type of data `/etc/passwd` contains.                                                                                                                | Give the path to `file`.                                        | `file /etc/passwd`                                                               |
| `ls *.log`                       | Lists filenames ending in `.log`. `*` matches zero or more characters.                                                                                            | Put `*` before `.log`.                                          | `ls *.log`                                                                       |
| `ls app*`                        | Lists names beginning with `app`.                                                                                                                                 | Put `*` after the starting characters.                          | `ls app*`                                                                        |
| `ls *.conf`                      | Lists names ending in `.conf`.                                                                                                                                    | Use `*.conf`.                                                   | `ls *.conf`                                                                      |
| `ls log?.txt`                    | Matches names where `?` represents exactly one character.                                                                                                         | Put `?` at the position where exactly one character may occur.  | `ls log?.txt`                                                                    |
| `ls server[12].log`              | Matches one character from the specified set.                                                                                                                     | Put acceptable characters inside `[]`.                          | `ls server[12].log`                                                              |
| `ls server[1-3].log`             | Matches one character within the specified range.                                                                                                                 | Use a range inside `[]`.                                        | `ls server[1-3].log`                                                             |
| `ls file[a-c].txt`               | Matches one letter from `a` through `c`.                                                                                                                          | Put the letter range inside brackets.                           | `ls file[a-c].txt`                                                               |
| `find STARTING-PATH CONDITIONS`  | Searches a directory tree recursively according to specified conditions.                                                                                          | Give a starting location followed by search conditions.         | `find STARTING-PATH CONDITIONS`                                                  |
| `find directory`                 | Recursively displays entries beneath the specified directory.                                                                                                     | Put the starting directory after `find`.                        | `find delete-practice`                                                           |
| `find . -name "filename"`        | Searches from the current directory for an exact filename.                                                                                                        | `.` starts at the current directory and `-name` searches names. | `find . -name "application.log"`                                                 |
| `find path -name "filename"`     | Recursively searches a specified directory tree by name.                                                                                                          | Replace the starting path and filename.                         | `find project -name "application.log"`                                           |
| `find . -name "*.log"`           | Searches recursively for names matching `.log`. Quotes prevent the shell from expanding `*` before `find` receives it.                                            | Put the wildcard pattern in quotes.                             | `find . -name "*.log"`                                                           |
| `find . -iname "*.log"`          | Performs a case-insensitive name search.                                                                                                                          | Use `-iname` instead of `-name`.                                | `find . -iname "*.log"`                                                          |
| `find . -type d -name "logs"`    | Searches specifically for directories named `logs`.                                                                                                               | `-type d` restricts matches to directories.                     | `find . -type d -name "logs"`                                                    |
| `find . -type f -name "*.conf"`  | Searches specifically for regular files matching `.conf`.                                                                                                         | `-type f` restricts matches to regular files.                   | `find . -type f -name "*.conf"`                                                  |
| `find . -name "*.tmp"`           | Searches recursively for `.tmp` files. The note uses this to teach previewing before destructive operations.                                                      | Search and inspect the results first.                           | `find . -name "*.tmp"`                                                           |
| `locate filename`                | Searches an indexed filename database rather than walking the filesystem live.                                                                                    | Put the filename or search term after `locate`.                 | `locate nginx.conf`                                                              |
| `locate hosts`                   | Searches the locate database for names containing `hosts`.                                                                                                        | Give `hosts` as the search term.                                | `locate hosts`                                                                   |
| `sudo updatedb`                  | Updates the database commonly used by `locate`. The note warns that this may require administrative privileges and should not be run blindly.                     | Run `updatedb` through `sudo` when appropriate.                 | `sudo updatedb`                                                                  |
| `which command`                  | Shows which executable is found through the current command search path.                                                                                          | Put the command name after `which`.                             | `which ls`                                                                       |
| `which bash`                     | Shows which `bash` executable is found.                                                                                                                           | Give `bash` to `which`.                                         | `which bash`                                                                     |
| `command -v command-name`        | Investigates how a shell command is resolved. It can be more shell-aware than `which`.                                                                            | Put the command after `command -v`.                             | `command -v ls`                                                                  |
| `whereis command`                | Finds certain files associated with a command, such as its binary and manual-page locations.                                                                      | Put the command name after `whereis`.                           | `whereis ls`                                                                     |
| `stat filename`                  | Displays detailed file metadata, including size, inode, permissions, ownership and timestamps.                                                                    | Put the filename after `stat`.                                  | `stat devops-project/logs/application.log`                                       |
| `ln SOURCE LINK-NAME`            | Creates a hard link. Both filenames refer to the same underlying inode.                                                                                           | Give the existing file first and new hard-link name second.     | `ln original.txt second-name.txt`                                                |
| `ln -s TARGET LINK-NAME`         | Creates a symbolic link that references another path.                                                                                                             | Add `-s`, then give the target and link name.                   | `ln -s server.conf current.conf`                                                 |
| `ln -s /absolute/path link-name` | Creates a symbolic link using an absolute target path.                                                                                                            | Give the absolute target first and link name second.            | `ln -s /etc/example/server.conf current.conf`                                    |
| `ls -l link-name`                | Inspects a symbolic link and shows where it points.                                                                                                               | Use `ls -l` on the link.                                        | `ls -l current.conf`                                                             |
| `whoami`                         | Displays the current user. Used in the note when troubleshooting permission problems.                                                                             | Run it by itself.                                               | `whoami`                                                                         |
| `sudo`                           | Runs an appropriate command with elevated privileges. In this note it is mainly discussed as something not to use automatically when you see `Permission denied`. | The note does not teach general `sudo` usage here.              | `sudo`                                                                           |

### The complete Part 3 command set

The note itself ends up teaching or using these main commands:

```text
pwd
cd
ls
touch
mkdir
printf
cat
less
more
head
tail
cp
mv
rm
rmdir
file
find
locate
updatedb
which
command -v
whereis
stat
ln
whoami
sudo
```

And the important command options or forms taught in the same note are:

```text
ls -l
ls -ld
ls -a
ls -la
ls -li

mkdir -p

head -n
tail -n
tail -f

cp -i
cp -r

mv -i

rm -i
rm -r
rm -rf

find -name
find -iname
find -type d
find -type f

ln -s
```

The wildcard patterns taught are:

```text
*
?
[...]
```

And the note demonstrates them with:

```text
ls *.log
ls app*
ls *.conf
ls log?.txt
ls server[12].log
ls server[1-3].log
ls file[a-c].txt
```






---

# Next Part

Part 4 will cover:

Linux Users, Groups, Root, and sudo.

You will learn how Linux identifies users, how UID and GID values work, where account information is stored, how users and groups are created and modified, what the root account really means, how `su` and `sudo` differ, and why least privilege is one of the most important principles in Linux administration.
