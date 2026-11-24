# 03 · Essential Administrator Commands

> [!NOTE]
> **Chapter objectives**
> - Visualize a directory tree with `tree`
> - Search for text in files with `grep`
> - Edit text without opening an editor using `sed`
> - Process columns and do calculations with `awk`
> - Cut, sort, and count output with `cut`, `sort`, `uniq`, `wc`, and `tr`
> - Find files by name, date, or size with `find`
> - Analyze disk usage with `du` and `df`
> - Create and extract archives with `tar`

## Table of Contents

1. [List a directory tree — `tree`](#list-a-directory-tree--tree)
2. [Extract information — `grep`](#extract-information--grep)
3. [Replace text — `sed`](#replace-text--sed)
4. [Process columns — `awk`](#process-columns--awk)
5. [Cut, sort, count — `cut`, `sort`, `uniq`, `wc`, `tr`](#cut-sort-count--cut-sort-uniq-wc-tr)
6. [Find files — `find`](#find-files--find)
7. [Disk space analysis — `du` and `df`](#disk-space-analysis--du-and-df)
8. [Archiving — `tar`](#archiving--tar)
9. [Other commands worth knowing](#other-commands-worth-knowing)
10. [Summary](#summary)

---

## List a directory tree — `tree`

```bash
tree
```

```text
.
├── 01_base.md
├── 02.1_user_management.md
├── 02_file_permissions.md
├── 03_commandes_essentielles.md
├── 04_installation_et_services.md
├── 05_securite.md
├── 06_shellscript
│   ├── 01_Base
│   ├── 02_operateur_logique
│   ├── 03_boucles
│   ├── 04_gestion_des_erreurs
│   ├── 05_Arguments
│   └── 06_fonctions
├── 07_docker.md
├── 08_docker_administration.md
├── 09_tmux.md
├── 10_shell_config.md
├── 11_advanced_config.md
├── exemple
│   ├── errors.txt
│   ├── exemple.txt
│   ├── exemple2.txt
│   ├── exercice.txt
│   ├── exerciceEditeur.txt
│   ├── fichier.txt
│   ├── out.txt
│   └── test.js
├── exercices
│   ├── Exercice3.md
│   └── Exercice4.md
└── README.md
```

Useful options:

| Option | Effect |
|--------|--------|
| `tree -L 2` | Limit the depth to 2 levels |
| `tree -d`   | Show directories only |
| `tree -a`   | Include hidden files |
| `tree -h`   | Show file sizes |

---

## Extract information — `grep`

`grep` searches for a pattern in a file or in the output of another command.

```bash
grep "sudo" 02.1_user_management.md
```

```text
sudo commande
sudo -u username commande
sudo su
```

### Show line numbers

```bash
grep -n "sudo" 02.1_user_management.md
```

```text
114:sudo commande
122:sudo -u username commande
136:sudo su
```

### Recursive search through a directory tree

```bash
grep -R "sudo" ../ShellScripts
```

```text
../ShellScripts/exercices/Exercice4.md:sudo useradd -d /home/newuser -m newuser
../ShellScripts/02.1_user_management.md:sudo commande
../ShellScripts/02.1_user_management.md:sudo -u username commande
../ShellScripts/02.1_user_management.md:sudo su
../ShellScripts/03_commandes_essentielles.md:grep "sudo" 02.1_user_management.md
../ShellScripts/03_commandes_essentielles.md:sudo commande
../ShellScripts/03_commandes_essentielles.md:sudo -u username commande
../ShellScripts/03_commandes_essentielles.md:sudo su
```

> [!TIP]
> `../ShellScripts` can be shortened to `.` if you are already in that directory.

### Ignore case

```bash
grep -i "user" 02.1_user_management.md
```

### Filter the output of another command

```bash
ps aux | grep apache
```

### Options to remember

| Option | Effect |
|--------|--------|
| `-n` | Show the line number |
| `-i` | Case-insensitive |
| `-R` / `-r` | Recursive search in subdirectories |
| `-v` | Invert: show the lines that do **not** contain the pattern |
| `-c` | Count the matching lines |
| `-l` | Show only the **names** of the files containing the pattern |
| `-E` | Use extended regular expressions |
| `-A 3` / `-B 3` | Show 3 lines **after** / **before** each match |

---

## Replace text — `sed`

`grep` **searches**, `sed` **modifies**. It is a stream editor: it reads line by line, applies a transformation, and writes the result to standard output.

### Substitute a word

```bash
sed 's/localhost/127.0.0.1/' config.txt
```

```text
s   => substitute (replace)
/   => separator between the search and the replacement
```

> [!IMPORTANT]
> By default, `sed` only replaces the **first occurrence on each line** and **writes nothing to the file**: it only displays the result.

To replace **all** occurrences on a line, add the `g` (*global*) flag:

```bash
sed 's/localhost/127.0.0.1/g' config.txt
```

### Actually modify the file

```bash
sed -i 's/localhost/127.0.0.1/g' config.txt
```

`-i` = *in place*: the change is written to the file.

> [!CAUTION]
> `sed -i` overwrites the file **without confirmation**. On a system configuration file, always keep a copy:
>
> ```bash
> sed -i.bak 's/^Port 22/Port 2222/' /etc/ssh/sshd_config
> ```
>
> `-i.bak` saves the original to `sshd_config.bak` before modifying it.
> The habit to build: first run the command **without** `-i` to check the result on screen.

### Change the separator

When the pattern contains `/` (typically a path), use another separator to avoid escaping:

```bash
# Unreadable
sed 's/\/var\/www/\/srv\/www/g' file

# Readable: the separator becomes |
sed 's|/var/www|/srv/www|g' file
```

### Delete lines

```bash
# Delete lines containing "DEBUG"
sed '/DEBUG/d' app.log
```

```bash
# Delete empty lines
sed '/^$/d' file.txt
```

```bash
# Delete the first line (a CSV header, for example)
sed '1d' data.csv
```

### Print a range of lines

`-n` turns off automatic printing, and `p` (*print*) shows only what you ask for:

```bash
# Lines 10 to 20
sed -n '10,20p' /var/log/syslog
```

```bash
# From line 100 to the end
sed -n '100,$p' /var/log/syslog
```

### Comment / uncomment a directive

A system administration classic:

```bash
# Comment out the line starting with PermitRootLogin
sed -i 's/^PermitRootLogin/#PermitRootLogin/' /etc/ssh/sshd_config
```

```bash
# Uncomment the line containing "max_connections"
sed -i '/max_connections/s/^#//' /etc/postgresql/15/main/postgresql.conf
```

### Chain several transformations

```bash
sed -e 's/foo/bar/g' -e '/^$/d' file.txt
```

… or with a `;`:

```bash
sed 's/foo/bar/g; /^$/d' file.txt
```

### Options to remember

| Element | Effect |
|---------|--------|
| `s/pattern/replacement/` | Replace the first occurrence on each line |
| `s/pattern/replacement/g` | Replace **all** occurrences |
| `s/pattern/replacement/gi` | … ignoring case |
| `-i` / `-i.bak` | Edit the file in place (with a backup) |
| `-n` … `p` | Print only the selected lines |
| `-E` | Extended regular expressions (`+`, `?`, `(a\|b)`) |
| `/pattern/d` | Delete lines matching the pattern |
| `1d` / `1,5d` / `$d` | Delete line 1 / lines 1 to 5 / the last line |
| `^` / `$` | Start / end of line |
| `&` | Refers back to the matched text (`s/error/[&]/` → `[error]`) |

---

## Process columns — `awk`

Many Linux outputs are **tables**: `ps aux`, `df -h`, `/etc/passwd`, Apache logs… `awk` automatically splits each line into columns and lets you extract, filter, and calculate.

### Print a column

```bash
ps aux | awk '{print $1}'
```

Columns are named `$1`, `$2`, `$3`… and `$0` refers to the **whole line**.

```bash
# The user and the command of each process
ps aux | awk '{print $1, $11}'
```

> [!NOTE]
> By default, `awk` splits on **spaces and tabs**, ignoring repeats. That is what makes it more reliable than `cut` on aligned output like `ps aux` or `df -h`.

### Change the separator

```bash
awk -F: '{print $1}' /etc/passwd
```

```text
root
daemon
bin
sys
...
```

### Filter lines

Like `grep`, but you keep the power of columns:

```bash
# Lines containing "error", printing the 1st and 4th columns
awk '/error/ {print $1, $4}' /var/log/apache2/error.log
```

And with a **condition** on a column:

```bash
# Real users (UID >= 1000), 3rd field of /etc/passwd
awk -F: '$3 >= 1000 {print $1, $3}' /etc/passwd
```

```bash
# Processes using more than 5% CPU
ps aux | awk '$3 > 5.0 {print $2, $3, $11}'
```

### Calculate

```bash
# Sum of the memory (%MEM) used by all processes
ps aux | awk '{sum += $4} END {printf "%.1f %%\n", sum}'
```

```bash
# Number of lines and average length of a field
awk -F: '{n++; total += length($1)} END {print n " lines, average length " total/n}' /etc/passwd
```

`END { … }` runs **once at the end**, after all lines have been read. Its counterpart `BEGIN { … }` runs before the first line (handy for printing a header).

### Full example: the most active IPs in a log

```bash
awk '{print $1}' /var/log/apache2/access.log | sort | uniq -c | sort -rn | head
```

```text
   1420 192.168.1.24
    317 10.0.0.8
     94 203.0.113.7
```

```text
awk '{print $1}'  => extract the 1st column (the IP address)
sort              => group identical lines together
uniq -c           => count each IP
sort -rn          => sort by count, descending
head              => keep the top 10
```

### Elements to remember

| Element | Effect |
|---------|--------|
| `$1`, `$2`, … | The 1st, 2nd column… |
| `$0` | The whole line |
| `$NF` | The **last** column (`NF` = *number of fields*) |
| `NR` | The current line number |
| `-F:` / `-F';'` | Column separator |
| `/pattern/ { … }` | Act only on lines matching the pattern |
| `$3 > 100 { … }` | Condition on a column's value |
| `BEGIN { … }` / `END { … }` | Block run before / after processing |
| `printf "%.2f\n", x` | Formatted output (2 decimal places) |

> [!TIP]
> `grep` to search, `sed` to replace, `awk` for columns and calculations. As soon as an "`awk` script" grows beyond a few lines, you are better off writing a real shell script (chapter 06) or Python.

---

## Cut, sort, count — `cut`, `sort`, `uniq`, `wc`, `tr`

These small commands are almost never used alone: they are combined with pipes.

### `cut` — extract simple columns

```bash
cut -d: -f1 /etc/passwd
```

```text
-d: => the delimiter is ":"
-f1 => I want field no. 1
```

```bash
# Several fields
cut -d: -f1,3,7 /etc/passwd
```

```bash
# The first 10 characters of each line
cut -c1-10 file.txt
```

> [!WARNING]
> `cut` treats **every** delimiter as a separator: on aligned output with several spaces (`ps aux`, `df -h`), it produces empty columns. In that case, use `awk`.

### `sort` — sort

```bash
sort file.txt
```

| Option | Effect |
|--------|--------|
| `-r` | Reverse order |
| `-n` | **Numeric** sort (otherwise `10` comes before `9`) |
| `-h` | Sort human-readable sizes (`1K`, `2M`, `3G`) |
| `-u` | Remove duplicates |
| `-k2` | Sort on the 2nd column |
| `-t:` | Change the column separator |

```bash
# Users sorted by UID
sort -t: -k3 -n /etc/passwd
```

### `uniq` — deduplicate and count

```bash
sort file.txt | uniq -c
```

| Option | Effect |
|--------|--------|
| `-c` | Count occurrences |
| `-d` | Show only duplicated lines |
| `-u` | Show only unique lines |

> [!IMPORTANT]
> `uniq` only compares **consecutive** lines. So you almost always need a `sort` before it.

### `wc` — count

```bash
wc -l /var/log/syslog
```

| Option | Effect |
|--------|--------|
| `-l` | Number of **lines** |
| `-w` | Number of **words** |
| `-c` | Number of **bytes** |

```bash
# How many users are on the machine?
wc -l < /etc/passwd
```

### `tr` — translate characters

```bash
# All uppercase
echo "hello" | tr 'a-z' 'A-Z'
```

```bash
# Delete characters
echo "1 234 567" | tr -d ' '
```

```bash
# Squeeze repeated spaces into one (useful before a cut)
ps aux | tr -s ' ' | cut -d' ' -f1,11
```

| Option | Effect |
|--------|--------|
| *(none)* | Replace one set of characters with another |
| `-d` | Delete the given characters |
| `-s` | *Squeeze*: reduce repeats to a single occurrence |

---

## Find files — `find`

### By name

```bash
find / -name "passwd" 2> /dev/null
```

> [!NOTE]
> `2> /dev/null` redirects errors (typically the "Permission denied" messages on system directories) into the void, keeping only the useful results.

### By modification date

Find all files modified less than two days ago:

```bash
find directory -mtime -2
```

### By size

Here, files larger than 10 MiB:

```bash
find /var/log -type f -size +10M
```

### Act on the files found

```bash
# Delete the files in /tmp ending in .tmp
find /tmp -type f -name "*.tmp" -delete
```

```bash
# List the details of each file found
# -exec  => I run a command
# ls -lh => the command that is run
# {}     => the item found
# \;     => end of the instruction
find . -type f -name "*.md" -exec ls -lh {} \;
```

Output:

```text
-rw-r--r-- 1 rmdir  staff     0B Dec 10 15:32 ./09_tmux.md
-rw-r--r-- 1 rmdir  staff   593B Dec 11 13:09 ./exercices/Exercice4.md
-rw-r--r-- 1 rmdir  staff   1.1K Dec 10 15:47 ./exercices/Exercice3.md
-rw-r--r-- 1 rmdir  staff    81B Dec 11 11:42 ./05_securite.md
-rw-r--r-- 1 rmdir  staff     0B Dec 10 15:33 ./11_advanced_config.md
-rw-r--r-- 1 rmdir  staff   3.6K Dec 10 14:35 ./01_base.md
-rw-r--r-- 1 rmdir  staff   2.7K Dec 11 09:53 ./02_file_permissions.md
-rw-r--r-- 1 rmdir  staff    86B Dec 11 11:40 ./04_installation_et_services.md
-rw-r--r-- 1 rmdir  staff    12B Dec 10 10:39 ./README.md
-rw-r--r-- 1 rmdir  staff   2.3K Dec 11 11:24 ./02.1_user_management.md
-rw-r--r-- 1 rmdir  staff     0B Dec 10 15:32 ./10_shell_config.md
-rw-r--r-- 1 rmdir  staff   2.8K Dec 11 13:28 ./03_commandes_essentielles.md
```

> [!TIP]
> `-exec command {} +` (with a `+` instead of `\;`) groups all the results into **a single call** to the command instead of one call per file. It is much faster on large volumes.

### Pass the results to another command — `xargs`

`xargs` turns a list received on standard input into **arguments** for a command. It is the alternative to `-exec`, and the only option when the list comes from somewhere other than `find`.

```bash
find /var/log -name "*.gz" | xargs rm
```

Why `xargs`? Many commands (`rm`, `grep`, `tar`…) expect their targets as arguments, not on standard input: `find … | rm` does not work, `find … | xargs rm` does.

```bash
# Search for a pattern only in .conf files
find /etc -name "*.conf" | xargs grep -l "listen"
```

Put the argument somewhere other than at the end, with `-I`:

```bash
# {} is replaced by each result
find . -name "*.md" | xargs -I {} cp {} /tmp/backup/
```

> [!CAUTION]
> A file name containing a **space** breaks `xargs`: `my report.md` becomes two arguments. The robust version uses the `-print0` / `-0` pair:
>
> ```bash
> find /var/log -name "*.gz" -print0 | xargs -0 rm
> ```

| Option | Effect |
|--------|--------|
| `-0` | Inputs are separated by `\0` (with `find -print0`) |
| `-I {}` | Replace `{}` with the argument, wherever it is in the command |
| `-n 1` | Only one argument per command call |
| `-P 4` | Run 4 calls in **parallel** |
| `-t` | Print the command before running it (to check) |

### Criteria to remember

| Criterion | Effect |
|-----------|--------|
| `-name "*.log"` | By name (case-sensitive; `-iname` is not) |
| `-type f` / `-type d` / `-type l` | File / directory / symbolic link |
| `-size +10M` / `-size -1k` | Larger than / smaller than |
| `-mtime -2` | Modified less than 2 days ago (`-mmin -30`: 30 minutes) |
| `-user debian` | Owned by a user |
| `-perm -4000` | With a given permission bit (here SUID) |
| `-maxdepth 2` | Limit the search depth |
| `-delete` / `-exec … \;` / `-exec … +` | Act on the results |

---

## Disk space analysis — `du` and `df`

### `du` — what directories **consume**

```bash
du -h --max-depth=1 /var
```

```text
4.0K	/var/spool
40K	/var/tmp
1.2M	/var/backups
82M	/var/cache
4.0K	/var/local
5.5G	/var/log
4.0K	/var/mail
20K	/var/www
4.0K	/var/opt
153M	/var/lib
5.7G	/var
```

Then drill down into the guilty directory:

```bash
du -h --max-depth=1 /var/log
```

```text
252K	/var/log/apt
4.1G	/var/log/journal
29M	/var/log/apache2
8.0K	/var/log/runit
4.0K	/var/log/private
792K	/var/log/unattended-upgrades
5.5G	/var/log
```

> [!TIP]
> To quickly find the biggest consumers: `du -h --max-depth=1 /var | sort -hr | head`

### Find the total size of a set of files with `find`

```bash
find /var/backups -name "*.backup" -exec du -ch {} + | grep total
```

### `df` — what is **left** on the partitions

```bash
df -h
```

```text
Filesystem      Size  Used Avail Use% Mounted on
udev            1.9G     0  1.9G   0% /dev
tmpfs           384M  500K  383M   1% /run
/dev/sda1        79G  6.6G   69G   9% /
tmpfs           1.9G     0  1.9G   0% /dev/shm
tmpfs           5.0M     0  5.0M   0% /run/lock
/dev/sda15      124M   12M  113M  10% /boot/efi
tmpfs           384M     0  384M   0% /run/user/1000
```

| Command | Question it answers |
|---------|---------------------|
| `df -h` | Are my partitions full? |
| `du -h` | What is taking up space? |

---

## Archiving — `tar`

### Create an archive

```bash
tar -cvf backup.tar /directory
```

```text
c => create
v => verbose (detailed output)
f => specify the file to save the archive to. Must be in last position
```

### Create a compressed archive

```bash
tar -czvf backup.tar.gz /directory
```

```text
c => create
z => compress with gzip
v => verbose (detailed output)
f => specify the file to save the archive to
```

Equivalent in two steps:

```bash
tar -cvf backup.tar /directory
gzip backup.tar
```

…which produces `backup.tar.gz`.

### Extract an archive

```bash
tar -xvf backup.tar
```

Compressed archive:

```bash
tar -xzvf backup.tar.gz
```

Extract into a target directory:

```bash
tar -xzvf backup.tar.gz -C /target_directory
```

> [!TIP]
> Before extracting an unknown archive, list its contents to make sure it will not dump 200 files into the current directory:
>
> ```bash
> tar -tvf backup.tar.gz
> ```

### Create a tarball from a `find`

```bash
find . -type f -name "*.md" -print0 | tar --null -czvf mdbackups.tar.gz --files-from=-
```

```text
find:
  -type f        => look for files
  -name "*.md"   => whose name ends in .md
  -print0        => separate each result with a \0

tar:
  --null         => entries are separated by \0 (output of the previous command)
  -czvf archive  => create a compressed archive
  --files-from=- => the list of files comes from standard input (-),
                    i.e. from the output of find
```

> [!NOTE]
> The `-print0` / `--null` pair correctly handles file names containing **spaces** or newlines. Without it, a file named `my report.md` would be treated as two entries.

### `tar` options to remember

| Option | Effect |
|--------|--------|
| `-c` | Create an archive |
| `-x` | Extract an archive |
| `-t` | List the contents without extracting |
| `-f` | Archive file name |
| `-v` | Verbose |
| `-z` / `-j` / `-J` | Compress with gzip / bzip2 / xz |
| `-C /path` | Change to this directory before acting |

---

## Other commands worth knowing

They don't deserve a chapter of their own, but they come in handy every day.

### `tee` — write to a file **and** to the screen

```bash
apt update | tee /tmp/update.log
```

Useful when a command needs `sudo` to write a file — the usual `>` redirection fails, because it is the (non-root) *shell* that opens the file:

```bash
echo "127.0.0.1 mysite.local" | sudo tee -a /etc/hosts
```

`-a` = *append*, like `>>`.

### `diff` — compare two files

The reflex after changing a configuration:

```bash
diff /etc/ssh/sshd_config.bak /etc/ssh/sshd_config
```

```bash
# More readable version, with context
diff -u old.conf new.conf
```

`diff -r directory1 directory2` compares two entire directory trees.

### `stat` and `file` — a file's metadata

```bash
stat report.pdf
```

```text
  File: report.pdf
  Size: 24576      Blocks: 48   IO Block: 4096   regular file
Access: (0644/-rw-r--r--)  Uid: ( 1000/  rmdir)   Gid: ( 1000/  rmdir)
Modify: 2025-12-11 13:28:41.000000000 +0100
```

```bash
# What type is this file really? (the extension can lie)
file archive.bin
```

```text
archive.bin: gzip compressed data, from Unix
```

### `watch` — repeat a command

```bash
# Redisplay df -h every 2 seconds
watch df -h
```

```bash
# Every 5 seconds, highlighting what changes
watch -n 5 -d "ls -l /var/spool/mail"
```

`CTRL + C` to quit.

### `which` and `type` — where is this command?

```bash
which python3
```

```text
/usr/bin/python3
```

`type my_command` goes further: it tells you whether it is a binary, an **alias**, or a shell function.

### In short

| Command | Purpose |
|---------|---------|
| `tee` | Duplicate output: file + screen |
| `diff` | Compare two files or two directories |
| `stat` | Detailed metadata (size, dates, permissions) |
| `file` | A file's real type |
| `watch` | Repeat a command at a regular interval |
| `which` / `type` | Locate a command, detect an alias |
| `ln -s` | Create a symbolic link (see chapter 11) |
| `rsync` | Copy / synchronize, including remotely (see chapter 05) |

---

## Summary

| Command | Purpose |
|---------|---------|
| `tree` | Visualize a directory tree |
| `grep` | **Search** for text |
| `sed` | **Replace** or delete text |
| `awk` | Extract **columns**, filter, calculate |
| `cut` | Extract columns with a simple delimiter |
| `sort` / `uniq` / `wc` | Sort / deduplicate / count |
| `tr` | Translate or delete characters |
| `find` | Search for files |
| `xargs` | Pass a list as arguments to a command |
| `du` | Measure used space |
| `df` | Measure available space |
| `tar` | Archive and compress |
| `tee` / `diff` / `watch` | Duplicate output / compare / monitor |

> [!TIP]
> The trio to remember for analyzing logs: `grep` filters the lines, `awk` extracts the columns, `sort | uniq -c | sort -rn` counts and ranks.
>
> ```bash
> grep "404" access.log | awk '{print $7}' | sort | uniq -c | sort -rn | head
> ```
>
> … i.e. the 10 URLs most often not found.

---

⬅️ [Previous: 02.1 · Users](02.1_user_management.md) · 🏠 [Contents](README.md) · [Next: 04 · Packages and services ➡️](04_installation_et_services.md)
