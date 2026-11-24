# 02 · File Permissions (POSIX)

> [!NOTE]
> **Chapter objectives**
> - Read and decode a permission line displayed by `ls -l`
> - Change permissions using octal and symbolic notation
> - Understand what `r`, `w`, and `x` mean on a **directory**
> - Know the special permissions (SUID, SGID, sticky bit)
> - Change the owner and group of a file

## Table of Contents

1. [Reading POSIX permissions](#reading-posix-permissions)
2. [The special case of directories](#the-special-case-of-directories)
3. [Changing permissions with chmod](#changing-permissions-with-chmod)
4. [Applying in bulk](#applying-in-bulk)
5. [Special permissions](#special-permissions)
6. [File types](#file-types)
7. [Changing the owner and group](#changing-the-owner-and-group)
8. [Summary](#summary)

---

## Reading POSIX permissions

```text
-rwxr-xr-- 1 username usergroup 219K May 15  2020 script.sh
```

Let's break down the first column:

```text
 -    rwx      r-x      r--
 │     │        │        │
type  user    group    others
```

| Letter | Meaning | On a file |
|--------|---------|-----------|
| `r` | **read** | Read the contents |
| `w` | **write** | Modify the contents |
| `x` | **execute** | Run the file (script, binary) |
| `-` | — | Permission not granted |

The three groups of three characters, in order:

| Group | Applies to | In the example |
|-------|------------|----------------|
| 1st (`rwx`) | The file's **owner** | `username` |
| 2nd (`r-x`) | The owning **group** | `usergroup` |
| 3rd (`r--`) | All **other** users | everyone |

So, in this case:

- the user `username` can **read / write / execute** the file;
- members of `usergroup` can **read and execute** it;
- everyone else can only **read** the file.

---

## The special case of directories

```text
drwxr-xr-- 1 username usergroup 219K May 15  2020 directory
```

The leading `d` means this is a **d**irectory. On a directory, the three permissions mean something quite different from what they mean on a file:

| Permission | On a directory |
|------------|----------------|
| `r` | List the contents (`ls`) |
| `w` | Add, rename, and delete files / subdirectories **inside** the directory |
| `x` | Traverse the directory, i.e. enter it (`cd`) and access what it contains |

> [!IMPORTANT]
> Without the `x` permission on a directory, the `r` permission is almost useless: you can see the file names, but you cannot open the files.
> Conversely, `w` on a directory lets you **delete** a file even if you have no permissions at all on that file — the directory decides.

---

## Changing permissions with chmod

### Octal notation

Permissions are encoded in octal, one digit from 0 to 7 per group:

| Value | Permission |
|-------|------------|
| `4` | read (`r`) |
| `2` | write (`w`) |
| `1` | execute (`x`) |

You add these digits together to build a permission:

| Digit | Calculation | Result |
|-------|-------------|--------|
| `7` | 4+2+1 | `rwx` |
| `6` | 4+2   | `rw-` |
| `5` | 4+1   | `r-x` |
| `4` | 4     | `r--` |
| `3` | 2+1   | `-wx` |
| `2` | 2     | `-w-` |
| `1` | 1     | `--x` |
| `0` | —     | `---` |

### Example 1 — lock a script down to its owner only

```bash
chmod 700 script.sh
```

Equivalent in symbolic notation:

```bash
chmod u=rwx,g=,o= script.sh
```


| | Permissions |
|---|---|
| **Before** | `-rwxr-xr-- 1 username usergroup 219K May 15  2020 script.sh` |
| **After** | `-rwx------ 1 username usergroup 219K May 15  2020 script.sh` |

Specific cases:
Add read for the owner
```bash
chmod u+r script.sh
```

Remove write from the group
```bash
chmod g-w script.sh
```

### Example 2 — a file shared with its group

```bash
chmod 664 script.sh
```

Equivalent in symbolic notation:

```bash
chmod u=rw,g=rw,o=r script.sh
```

| | Permissions |
|---|---|
| **Before** | `-rwx------ 1 username usergroup 219K May 15  2020 script.sh` |
| **After** | `-rw-rw-r-- 1 username usergroup 219K May 15  2020 script.sh` |

> [!TIP]
> Symbolic notation also accepts `+` and `-` to **add** or **remove** a permission without touching the rest:
>
> | Command | Effect |
> |---------|--------|
> | `chmod +x script.sh`   | Makes the file executable for everyone |
> | `chmod u+x script.sh`  | Executable for the owner only |
> | `chmod o-r file`       | Removes read from others |
> | `chmod a=r file`       | Read-only for everyone (`a` = *all*) |

---

## Applying in bulk

Change several files at once:

```bash
chmod 700 *.sh
```

Change a directory and apply the same permissions to everything inside it (`-R` = recursive):

```bash
chmod -R 700 directory/
```

> [!WARNING]
> `chmod -R 700` also applies the `x` permission to the **files** inside the directory, which is rarely what you want.
> Best practice is to handle the two cases separately:
>
> ```bash
> find directory/ -type d -exec chmod 700 {} +   # directories: need x to be traversed
> find directory/ -type f -exec chmod 600 {} +   # files: no x
> ```

Below, I set 700 on all the children contained in directory/ but not on the directory itself.
```bash
chmod -R 700 directory/*
```

---

## Special permissions

### SUID and SGID

```text
-rwsr-xr-x  →  the owner's x is replaced by an s
```

The **SUID** bit (on the user) or **SGID** bit (on the group) makes the file run **with the permissions of the file's owner / owning group**, rather than those of the user who launches it.

Add the bit:

```bash
chmod u+s file.sh
```

```bash
chmod g+s file.sh
```

Remove it:

```bash
chmod u-s file.sh
```

```bash
chmod g-s file.sh
```

### SGID on a directory

On a **directory**, `g+s` has a different effect, very useful for a shared workspace: every file created inside it automatically inherits the **directory's group** instead of its creator's group.

```bash
chmod g+s /srv/shared_project
```

### Sticky bit

```bash
chmod +t /tmp
```

On a world-writable directory, the *sticky bit* (`drwxrwxrwt`) ensures that **only a file's owner can delete it**. This is what protects `/tmp`.

---

## File types

The very first character of the line indicates the item's type:

```bash
ls -l /
```

```text
lrwxrwxrwx   1 root root    7 Dec  2  2024 bin -> usr/bin
```

Here `/bin` is a **symbolic link** pointing to `/usr/bin`.

| Character | Type |
|-----------|------|
| `-` | Regular file |
| `d` | Directory |
| `l` | Symbolic link |
| `c` | Character device (e.g. `/dev/null`) |
| `b` | Block device (e.g. `/dev/sda`) |
| `s` | Socket |
| `p` | Named pipe |

---

## Changing the owner and group

### Change the owner

```bash
chown newusername file
```

| | Result |
|---|---|
| **Before** | `-rw-rw-r-- 1 username    usergroup 219K May 15  2020 file` |
| **After** | `-rw-rw-r-- 1 newusername usergroup 219K May 15  2020 file` |

```bash
chown newusername:newgroup file
```

```bash
chown -R newusername:newgroup directory/
```

### Change the group

```bash
chgrp newgroup file
```

| | Result |
|---|---|
| **Before** | `-rw-rw-r-- 1 username usergroup 219K May 15  2020 file` |
| **After** | `-rw-rw-r-- 1 username newgroup  219K May 15  2020 file` |

> [!TIP]
> `chown` can do both at once, and it also accepts `-R`:
>
> ```bash
> chown newusername:newgroup file
> chown -R www-data:www-data /var/www
> ```
>
> Changing a file's owner requires root privileges.

---

## Summary

| Command | Purpose |
|---------|---------|
| `ls -l` | Display permissions |
| `chmod 750 f` | Set permissions (octal) |
| `chmod u+x f` | Add / remove a permission (symbolic) |
| `chmod -R` | Apply recursively |
| `chmod u+s` / `g+s` / `+t` | SUID / SGID / sticky bit |
| `chown user f` | Change the owner |
| `chgrp group f` | Change the group |
| `chown user:group f` | Change both |

**The most common values:**

| Octal | Symbolic | Typical use |
|-------|----------|-------------|
| `600` | `rw-------` | Private file (SSH key, secret) |
| `644` | `rw-r--r--` | Config file readable by everyone |
| `700` | `rwx------` | Private script or directory |
| `750` | `rwxr-x---` | Script shared with its group |
| `755` | `rwxr-xr-x` | Public binary or directory |

---

⬅️ [Previous: 01 · Basics](01_base.md) · 🏠 [Contents](README.md) · [Next: 02.1 · Users ➡️](02.1_user_management.md)
