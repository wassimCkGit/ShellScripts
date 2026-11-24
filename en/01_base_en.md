# 01 · Linux Terminal Basics

> [!NOTE]
> **Chapter Objectives**
> - Navigate a directory tree from the terminal
> - Understand Linux filesystem organization
> - Create, copy, move and delete files and folders
> - Redirect a command's output and chain them together
> - Read and edit a file with `nano` and `vim`
> - Inspect and stop running processes

## Table of Contents

1. [Navigation in the terminal](#navigation-in-the-terminal)
2. [The Linux filesystem](#the-linux-filesystem)
3. [File and folder manipulation](#file-and-folder-manipulation)
4. [Output redirection](#output-redirection)
5. [Displaying file contents](#displaying-file-contents)
6. [Editing a file](#editing-a-file)
7. [Redirection with EOF (heredoc)](#redirection-with-eof-heredoc)
8. [Chaining commands with pipes](#chaining-commands-with-pipes)
9. [Process management](#process-management)
10. [Summary](#summary)

---

## Navigation in the terminal

Display the current location in the terminal (**p**rint **w**orking **d**irectory):

```bash
pwd
```

Move into a folder:

```bash
cd FolderName
```

Move to the parent folder:

```bash
cd ..
```

Move to a folder in the parent folder:

```bash
cd ../FolderName
```

> [!TIP]
> A few path shortcuts that come up all the time:
>
> | Shortcut | Meaning |
> |----------|---------|
> | `.`  | The current folder |
> | `..` | The parent folder |
> | `~`  | The current user's *home* (`/home/username`) |
> | `-`  | The previous folder (`cd -` toggles between two) |
> | `/`  | The system root |

---

## The Linux filesystem

On Linux, **everything starts from a single root noted `/`**. There are no `C:` or `D:` drives like Windows: disks, USB sticks and even some devices are "plugged in" (*mounted*) somewhere in this single tree.

Under this root, you always find roughly the same folders, each with a specific role (the *FHS — Filesystem Hierarchy Standard*).

```
/
├── bin    → basic commands (ls, cp, cat…)
├── sbin   → administration commands (reserved for root)
├── boot   → Linux kernel and boot files
├── etc    → system configuration files
├── home   → users' personal folders
├── root   → root's home folder
├── lib    → shared libraries for programs
├── usr    → installed programs and data (most of the system)
├── var    → data that changes: logs, queues, caches…
├── tmp    → temporary files (deleted on reboot)
├── opt    → third-party software installed "separately"
├── mnt    → manual mount point (disks, network shares)
├── media  → automatic mounting of removable media (USB, CD…)
├── dev    → devices seen as files (disks, terminals…)
├── proc   → virtual filesystem: process and kernel state
└── sys    → virtual filesystem: hardware and kernel parameters
```

| Folder | Role | Practical example |
|--------|------|-------------------|
| `/bin`  | Essential commands, available to all | `/bin/ls`, `/bin/cp` |
| `/sbin` | System administration commands | `/sbin/reboot`, `/sbin/fdisk` |
| `/boot` | Kernel and boot files (bootloader) | `/boot/vmlinuz`, GRUB |
| `/etc`  | **All system** configuration | `/etc/passwd`, `/etc/ssh/sshd_config` |
| `/home` | Users' personal files | `/home/alice`, `/home/bob` |
| `/root` | The root user's *home* | `/root` |
| `/usr`  | Installed programs and resources | `/usr/bin`, `/usr/share` |
| `/var`  | Data that grows over time | `/var/log`, `/var/www` |
| `/tmp`  | Temporary, disposable files | installer files |
| `/dev`  | Hardware devices as files | `/dev/sda`, `/dev/null` |
| `/proc` | **Virtual** view of processes and kernel | `/proc/cpuinfo`, `/proc/1234` |
| `/sys`  | **Virtual** view of hardware and kernel | `/sys/class/net` |

> [!NOTE]
> `/proc` and `/sys` don't exist on disk: they're generated **on the fly** by the kernel. Reading `/proc/cpuinfo` amounts to asking the kernel for processor info directly.

> [!TIP]
> Two folders to know by heart daily: **`/etc`** (where you configure) and **`/var/log`** (where you look when things don't work).

> [!CAUTION]
> Folders `/`, `/etc`, `/bin`, `/boot`, `/usr`… belong to the system. Touch them only with `sudo` and knowing what you're doing: a careless deletion can make the machine unusable.

---

## File and folder manipulation

Create a file:

```bash
touch fileName
```

Delete a file:

```bash
rm fileName
```

Create a new folder:

```bash
mkdir folderName
```

Delete an **empty** folder:

```bash
rmdir folderName
```

Delete a folder **and all its contents**:

```bash
rm -Rf folderName
```

> [!CAUTION]
> `rm -Rf` deletes **without confirmation and without trash**: there's no undo. Always check the path before pressing enter, especially if it starts with `/` or contains a variable — `rm -Rf "$FOLDER/"` with empty `FOLDER` targets the root.

Move a file or folder:

```bash
mv ./source new/location/
```

This moves the file `source` (located in the current folder) to `./new/location/source`.

Rename a file or folder — same command, we "move" to a new name:

```bash
mv oldName newName
```

Copy a file:

```bash
cp source copy
```

Creates a new file named `copy` with the same content as `source`.

Copy a folder (`-R` = recursive, so with its content):

```bash
cp -R source/ newFolder/
```

---

## Output redirection

Redirect a command's output to a file:

```bash
echo "Hello" > example.txt
```

or

```bash
echo "Hello" >> example.txt
```

The difference between `>` and `>>`:

| Operator | Behavior |
|----------|----------|
| `>`  | **Overwrites** the file's content (and creates it if it doesn't exist) |
| `>>` | **Appends** to the end of the file |

This works with any command:

```bash
ls >> files.txt
```

I can also redirect **only errors**:

```bash
ls 2> errors.txt
```

> [!NOTE]
> Every program has three standard streams. The number before `>` designates which stream to redirect:
>
> | Stream | Number | Role |
> |--------|--------|------|
> | `stdin`  | `0` | Standard input (keyboard by default) |
> | `stdout` | `1` | Standard output (`>` is shorthand for `1>`) |
> | `stderr` | `2` | Error output |
>
> Redirect everything to one place: `command > output.log 2>&1`
> Discard everything: `command > /dev/null 2>&1`

---

## Displaying file contents

Display the entire file:

```bash
cat filename.txt
```

Display the first 10 lines:

```bash
head filename
```

Display the last 10 lines:

```bash
tail filename
```

Display `n` first/last lines (here 5):

```bash
head -n 5 filename
```

```bash
tail -n 5 filename
```

> [!TIP]
> `tail -f logfile.log` follows the file **in real time**: new lines display as they're written. Essential for monitoring logs.
> To browse a long file: `less file` (arrows to move, `/` to search, `q` to quit).

---

## Editing a file

Two editors are available natively on most Linux distributions.

### nano — the simplest

```bash
nano filename
```

Useful shortcuts:

| Shortcut | Action |
|----------|--------|
| `CTRL + O` | Save |
| `CTRL + X` | Quit |
| `CTRL + K` | Cut a line |
| `CTRL + U` | Paste the cut line |
| `CTRL + W` | Search text |
| `CTRL + \` | Replace text |

### vim — the most powerful

```bash
vim filename
```

Three important modes:

| Mode | How to enter | Purpose |
|------|--------------|---------|
| **Normal** | `ESC` (default mode) | Navigate, delete, copy |
| **Insert** | `i` | Type text |
| **Command** | `:` | Save, quit, search/replace |

**Add text, step by step:**

1. Open `vim`
2. Press `i`
3. Type the text
4. `ESC` to return to normal mode
5. `:wq` (or `:x`) to save and quit

| Command | Action |
|---------|--------|
| `:w`  | Save |
| `:q`  | Quit |
| `:q!` | Quit **without** saving |
| `:wq` / `:x` | Save and quit |

> [!IMPORTANT]
> On Debian and other distributions, `vim` isn't always installed, but `vi` (its ancestor) almost always is.

---

## Redirection with EOF (heredoc)

The *heredoc* lets you write multiple lines at once into a file — very handy in a script for generating a config file.

```bash
cat <<EOF >> output.txt
the content I want in my file
I can add line breaks
etc
...
and I close the file with:
EOF
```

> [!TIP]
> With `<<EOF`, variables are interpreted (`$USER` is replaced by its value).
> With `<<'EOF'` (in quotes), text is written **as-is**, without interpretation.

---

## Chaining commands with pipes

The `|` (*pipe*) redirects the **output** of one command to the **input** of another.

Extract all lines containing `sha-256`:

```bash
cat /etc/postgresql/15/main/pg_hba.conf | grep sha-256
```

Sort an output:

```bash
ls / | sort
```

Count lines containing `sha-256`:

```bash
cat /etc/postgresql/15/main/pg_hba.conf | grep sha-256 | wc -l
```

You can chain as many commands as you want: each `|` passes the result to the next link.

---

## Process management

Display running processes:

```bash
ps aux
```

Interactive version:

```bash
top
```

Or much more readable (to install):

```bash
htop
```

### Find a specific process

```bash
ps aux | grep node
```

Once I have the **PID** (*process ID*) of the process, I can kill it:

```bash
kill -9 PID
```

> [!WARNING]
> `kill -9` (SIGKILL) kills the process **brutally**: it doesn't get a chance to close files or connections properly.
> Always try `kill PID` (SIGTERM, clean shutdown) first, and keep `-9` for unresponsive processes.

---

## Summary

| Command | Role |
|---------|------|
| `pwd` | Where am I? |
| `cd` | Move around |
| `ls` | List folder contents |
| `touch` / `mkdir` | Create a file / a folder |
| `rm` / `rmdir` | Delete a file / an empty folder |
| `mv` | Move or rename |
| `cp` | Copy |
| `cat` / `head` / `tail` | Read a file (all / start / end) |
| `nano` / `vim` | Edit a file |
| `>` / `>>` / `2>` | Redirect output / append to end / redirect errors |
| `\|` | Chain two commands |
| `ps` / `top` / `htop` | View processes |
| `kill` | Stop a process |

> [!TIP]
> When in doubt about a command: `man command` (full manual) or `command --help` (quick help).

---

🏠 [Contents](README.md) · [Next: 02 · Permissions ➡️](02_file_permissions.md)
