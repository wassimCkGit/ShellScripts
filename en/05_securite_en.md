# 05 · Security

> [!NOTE]
> **Chapter Objectives**
> - Go beyond POSIX permissions with **ACL**
> - Delegate administration rights precisely via **sudoers**
> - Filter network traffic with a **firewall** (`ufw`, `firewalld`, `iptables`, `nftables`)
> - Configure, harden and diagnose **SSH** access by keys
> - Confine services with mandatory access control (**AppArmor / SELinux**)

This chapter covers the four layers that stack on a Linux server. They don't replace each other, they stack:

```text
┌──────────────────────────────────────────────────────────────┐
│  MAC        AppArmor / SELinux   → what a process can        │
│                                    do, even as root          │
├──────────────────────────────────────────────────────────────┤
│  Privileges sudo / sudoers       → who can become who        │
├──────────────────────────────────────────────────────────────┤
│  Files      POSIX + ACL          → who can read what        │
├──────────────────────────────────────────────────────────────┤
│  Network    ufw / nftables       → who can reach the        │
│             SSH                    machine, and how         │
└──────────────────────────────────────────────────────────────┘
```

## Table of Contents

1. [ACL (Access Control Lists)](#acl-access-control-lists)
   - [Prerequisites and verification](#prerequisites-and-verification)
   - [Reading an ACL in detail](#reading-an-acl-in-detail)
   - [Modifying ACL](#modifying-acl)
   - [The mask in detail](#the-mask-in-detail)
   - [Default ACL and inheritance](#default-acl-and-inheritance)
   - [Practical case: a team folder](#practical-case-a-team-folder)
   - [Backing up, copying and migrating ACL](#backing-up-copying-and-migrating-acl)
   - [ACL diagnosis](#acl-diagnosis)
2. [Sudo / sudoers](#sudo--sudoers)
   - [Daily usage](#daily-usage)
   - [Editing sudoers without locking yourself out](#editing-sudoers-without-locking-yourself-out)
   - [File anatomy](#file-anatomy)
   - [Rule syntax](#rule-syntax)
   - [Tags](#tags)
   - [Defaults](#defaults)
   - [Aliases](#aliases)
   - [sudoedit — editing a protected file, properly](#sudoedit--editing-a-protected-file-properly)
   - [Complete practical cases](#complete-practical-cases)
   - [Security pitfalls](#security-pitfalls)
   - [Audit and logging](#audit-and-logging)
   - [Sudo diagnosis](#sudo-diagnosis)
3. [Firewall](#firewall)
   - [Common concepts](#common-concepts)
   - [ufw (Debian, Ubuntu)](#ufw-debian-ubuntu)
   - [firewalld (RHEL, Fedora)](#firewalld-rhel-fedora)
   - [iptables](#iptables)
   - [nftables](#nftables)
   - [Testing and diagnosing](#testing-and-diagnosing-a-firewall)
4. [SSH](#ssh)
   - [Installation and verification](#installation-and-verification)
   - [The two configuration files](#the-two-configuration-files)
   - [Key management](#key-management)
   - [Deploying your public key](#deploying-your-public-key)
   - [Hardening the server: sshd_config](#hardening-the-server-sshd_config)
   - [Match blocks](#match-blocks)
   - [Restricting a key in authorized_keys](#restricting-a-key-in-authorized_keys)
   - [Configuring the client: ~/.ssh/config](#configuring-the-client-sshconfig)
   - [known_hosts and fingerprints](#known_hosts-and-fingerprints)
   - [The SSH agent](#the-ssh-agent)
   - [Tunnels and jumps](#tunnels-and-jumps)
   - [Transferring files](#transferring-files)
   - [fail2ban](#fail2ban)
   - [SSH diagnosis](#ssh-diagnosis)
5. [AppArmor / SELinux](#apparmor--selinux)
   - [AppArmor (Debian, Ubuntu, SUSE)](#apparmor-debian-ubuntu-suse)
   - [SELinux (RHEL, Fedora, CentOS, Rocky)](#selinux-rhel-fedora-centos-rocky)
   - [Diagnostic method](#diagnostic-method)
6. [Summary](#summary)

---

## ACL (Access Control Lists)

POSIX permissions only know three targets:

```text
 Owner   Group   Other
 rwx     rwx     rwx
```

This is sufficient as long as needs are simple. But as soon as you want *"Alice can write, Bob can only read, the dev team can read, and no one else sees anything"*, you would need to create a group for each combination of rights. This is exactly the problem that **ACL** solves: grant rights to **a specific user** or **a specific group**, in addition to the owner and owning group.

### Prerequisites and verification

**1. The tools** — they are not always installed:

```bash
sudo apt install acl          # Debian / Ubuntu
sudo dnf install acl          # RHEL / Fedora
```

**2. The filesystem must support ACL.** This is the case by default for ext4 and XFS on modern distributions, but should be verified:

```bash
# For ext2/3/4
sudo tune2fs -l /dev/sda1 | grep "Default mount options"
```

```text
Default mount options:    user_xattr acl
```

```bash
# Check effective mount options
mount | grep " / "
findmnt -o TARGET,SOURCE,FSTYPE,OPTIONS /
```

If the `acl` option is missing, add it to `/etc/fstab`:

```text
# /etc/fstab
UUID=xxxx-xxxx  /srv  ext4  defaults,acl  0  2
```

```bash
sudo mount -o remount /srv
```

**3. Quick test** — if the filesystem doesn't support ACL, `setfacl` fails explicitly:

```bash
setfacl -m u:nobody:r /tmp/test.txt
```

```text
setfacl: /tmp/test.txt: Operation not supported
```

### Reading an ACL in detail

```bash
getfacl hello.txt
```

```text
# file: hello.txt              ← the file in question
# owner: debian                ← POSIX owner
# group: debian                ← POSIX owning group
user::rw-                      ← owner's rights (= the "u" in ls -l)
user:alice:rw-                 ← NAMED ENTRY: alice's specific rights
group::r--                     ← owning group's rights (= the "g" in ls -l)
group:dev:r-x                  ← NAMED ENTRY: dev group's specific rights
mask::rwx                      ← ceiling applied to named entries and group
other::r--                     ← others' rights (= the "o" in ls -l)
```

| Line type | Meaning |
|-----------|---------|
| `user::` | The owner — **never** limited by the mask |
| `user:name:` | A named user — subject to the mask |
| `group::` | The owning group — subject to the mask |
| `group:name:` | A named group — subject to the mask |
| `mask::` | The ceiling of effective rights |
| `other::` | Everyone else — **never** limited by the mask |
| `default:…` | An ACL inherited by elements created in the folder |

### Spotting a file with ACL

```bash
ls -la hello.txt
```

```text
-rw-r-xr--+ 1 debian debian   378 Dec 12 10:29 hello.txt
```

> [!TIP]
> The small **`+`** at the end of permissions signals additional rights beyond classic POSIX. It's the **only** visible clue in `ls` — hence the importance of checking with `getfacl` when a file behaves "strangely".
>
> Additional reading pitfall: when an ACL is present, the group rights column in `ls -l` no longer displays the owning group's rights, but **the mask**. So `ls` can announce `rwx` where the group actually only has `r--`.

Find all files bearing ACL in a directory tree:

```bash
getfacl -R -s /srv 2>/dev/null | grep "^# file:"
```

### Modifying ACL

The general syntax is:

```text
setfacl [-R] -m  <type>:<name>:<rights>  <target>
         │    │      │      │      │
         │    │      │      │      └─ rwx, rw, r, or - for nothing
         │    │      │      └──────── empty for owner / owning group
         │    │      └─────────────── u (user), g (group), m (mask), o (other)
         │    └────────────────────── -m modifies, -x removes, -b erases all
         └─────────────────────────── recursive
```

| Command | Effect |
|---------|--------|
| `setfacl -m u:alice:rw f` | Give `rw` to user alice |
| `setfacl -m g:dev:rx f` | Give `rx` to group dev |
| `setfacl -m o::--- f` | Remove all rights from others |
| `setfacl -m m:r f` | Set mask to `r` |
| `setfacl -x u:alice f` | Remove alice's entry |
| `setfacl -b f` | Delete **all** ACL (return to pure POSIX) |
| `setfacl -k d/` | Delete only **default** ACL |
| `setfacl -R -m …` | Apply recursively |
| `setfacl -d -m u:alice:rw d/` | Set a **default** ACL (inheritance) |

#### 1. Giving rights to a user

```bash
setfacl -m u:test:rw hello.txt
```

```bash
getfacl hello.txt
```

```text
# file: hello.txt
# owner: debian
# group: debian
user::rw-
user:test:rw-
group::r--
mask::rw-
other::r--
```

#### 2. Giving rights to a group

```bash
setfacl -m g:test:rx hello.txt
```

```text
# file: hello.txt
# owner: debian
# group: debian
user::rw-
group::r--
group:test:r-x
mask::r-x
other::r--
```

#### 3. Multiple entries in one command

Entries are separated by commas:

```bash
setfacl -m u:alice:rw,u:bob:r,g:dev:rx report.txt
```

#### 4. Applying recursively

```bash
setfacl -R -m u:test:rX folder/
```

> [!TIP]
> Note the **uppercase `X`**: it grants execute permission **only to directories** (and files that are already executable). This is exactly what you want recursively — without it, `-R -m u:test:rx` would make all data files in the tree executable.

#### 5. Removing rights

```bash
setfacl -x u:test hello.txt      # remove a specific entry
setfacl -b hello.txt             # erase everything, return to pure POSIX
```

### The mask in detail

This is the point that confuses everyone, and the main cause of ACL "that don't work".

The `mask` is a **ceiling** applied to all named entries (`user:name`, `group:name`) **and** the owning group. It applies **neither** to the owner (`user::`) **nor** to others (`other::`).

```text
Effective right  =  granted right  AND  mask
```

```bash
setfacl -m u:test:rwx test.sh
setfacl -m m:r test.sh
getfacl test.sh
```

```text
# file: test.sh
# owner: debian
# group: debian
user::rwx
user:test:rwx                   #effective:r--     ← the mask decided
group::r-x                      #effective:r--
mask::r--
other::---
```

`getfacl` is explicit: it displays `#effective:` next to each entry limited by the mask. User `test` may have `rwx` written, but can only **read**.

> [!CAUTION]
> **`chmod` recalculates the mask.** It's the classic pitfall, and it's silent:
>
> ```bash
> setfacl -m u:alice:rwx project/     # alice has rwx
> chmod g-w project/                  # we think we're only touching the group...
> getfacl project/                    # ...but the mask changed to r-x
> ```
>
> `chmod` on **group** rights actually modifies the **mask** when ACL are present. Alice's rights fall with it. After any `chmod` on a file bearing ACL, check the mask:
>
> ```bash
> setfacl -m m:rwx project/           # restore the ceiling
> ```

By default, `setfacl` automatically recalculates the mask to encompass all entries. To prevent this:

```bash
setfacl -n -m u:alice:rwx file    # -n: don't recalculate the mask
```

### Default ACL and inheritance

Default ACL (`default:`, or the `-d` option) apply **only to directories**. They grant no rights on the directory itself: they define the ACL that elements **created inside** will automatically receive.

```bash
setfacl -d -m u:test:rw shared/
setfacl -d -m g:dev:rwx shared/
```

Or with explicit syntax:

```bash
setfacl -m d:u:test:rw shared/
setfacl -m d:g:dev:rwx shared/
```

```bash
getfacl shared/
```

```text
# file: shared
# owner: debian
# group: dev
user::rwx
group::r-x
other::---
default:user::rwx
default:user:test:rw-
default:group::r-x
default:group:dev:rwx
default:mask::rwx
default:other::---
```

Let's verify inheritance:

```bash
touch shared/newfile.txt
getfacl shared/newfile.txt
```

```text
# file: shared/newfile.txt
# owner: debian
# group: dev
user::rw-
user:test:rw-
group::r-x                      #effective:r--
group:dev:rwx                   #effective:rw-
mask::rw-
other::---
```

> [!IMPORTANT]
> Three things to know about inheritance:
> 1. It applies **only to new** files. Existing files are not modified — you need a `setfacl -R` for them.
> 2. A **subfolder** created inherits **both** access ACL **and** default ACL: inheritance propagates in depth.
> 3. A file never gets the `x` right through inheritance if it's not created executable; the mask adjusts accordingly (`rwx` by default becomes `rw-` effective above).

Remove default ACL from a directory:

```bash
setfacl -k shared/
```

### Practical case: a team folder

This is *the* use case for ACL. The objective:

> A folder `/srv/project` where the **dev** team works together. Any file created by anyone must be immediately modifiable by the whole team. The **qa** team can read. No one else sees anything.

```bash
# 1. The groups
sudo groupadd dev
sudo groupadd qa
sudo usermod -aG dev alice
sudo usermod -aG dev bob
sudo usermod -aG qa carol

# 2. The folder, owned by the dev group
sudo mkdir -p /srv/project
sudo chgrp dev /srv/project
sudo chmod 2770 /srv/project
```

The `2` in `2770` is the **SGID**: any file created in the folder will belong to the `dev` group and not the creator's primary group. It's the essential complement to default ACL (see [chapter 02](02_file_permissions.md#sgid-on-a-folder)).

```bash
# 3. Access ACL: what everyone can do on the folder itself
sudo setfacl -m g:dev:rwx /srv/project
sudo setfacl -m g:qa:rx  /srv/project

# 4. Default ACL: what new files will inherit
sudo setfacl -d -m g:dev:rwx /srv/project
sudo setfacl -d -m g:qa:rx   /srv/project
sudo setfacl -d -m o::---    /srv/project
```

Verification:

```bash
getfacl /srv/project
```

```text
# file: srv/project
# owner: root
# group: dev
# flags: -s-                    ← SGID is set
user::rwx
group::rwx
group:dev:rwx
group:qa:r-x
mask::rwx
other::---
default:user::rwx
default:group::rwx
default:group:dev:rwx
default:group:qa:r-x
default:mask::rwx
default:other::---
```

Real-world test:

```bash
sudo -u alice touch /srv/project/file_alice.txt
getfacl /srv/project/file_alice.txt
```

```text
# file: srv/project/file_alice.txt
# owner: alice
# group: dev                     ← thanks to SGID
user::rw-
group::rwx                       #effective:rw-
group:dev:rwx                    #effective:rw-   ← bob can modify
group:qa:r-x                     #effective:r--   ← carol can read
mask::rw-
other::---                       ← no one else
```

### Backing up, copying and migrating ACL

**Back up ACL from a directory tree** — do this before any risky manipulation:

```bash
getfacl -R /srv/project > /root/acl-project.bak
```

**Restore**:

```bash
setfacl --restore=/root/acl-project.bak
```

**Copy ACL from one file to another**:

```bash
getfacl source.txt | setfacl --set-file=- destination.txt
```

**Preserve ACL when copying and backing up** — tools don't do it *by default*:

| Tool | Required option |
|------|-----------------|
| `cp` | `cp -p` or `cp --preserve=all` |
| `rsync` | `rsync -A` (and `-X` for extended attributes) |
| `tar` | `tar --acls` |
| `mv` | Nothing needed — ACL follow the file on the same filesystem |

```bash
rsync -avAX /srv/project/ /mnt/backup/project/
tar --acls -czvf project.tar.gz /srv/project
```

> [!WARNING]
> A backup made without `-A` / `--acls` restores files with POSIX permissions only. On restore day, the carefully built access structure has disappeared — and no one notices until someone complains about access denied.

### ACL diagnosis

An ACL that doesn't produce the expected effect almost always comes from one of these five causes:

| Symptom | Likely cause | Verification |
|---------|--------------|--------------|
| `Operation not supported` | Filesystem not mounted with `acl` | `findmnt -o OPTIONS /srv` |
| User can't write despite `rw` | The **mask** caps the right | `getfacl f \| grep effective` |
| Right disappeared after `chmod` | `chmod` recalculated the mask | `setfacl -m m:rwx f` |
| User can't **reach** the file | Missing `x` right on a **parent folder** | `namei -l /srv/project/file.txt` |
| New files don't inherit | Default ACL missing, or files created before | `getfacl d/ \| grep default` |

The most useful command for the fourth case:

```bash
namei -l /srv/project/docs/report.txt
```

```text
f: /srv/project/docs/report.txt
 drwxr-xr-x root  root  /
 drwxr-xr-x root  root  srv
 drwxrws---+ root dev   project
 drwx------  root root  docs        ← the blockage is here
 -rw-rw----+ alice dev  report.txt
```

`namei -l` unrolls the path folder by folder and immediately shows where access breaks. Perfect rights on the file are useless if a parent folder forbids traversal.

### ACL Summary

| Command | Effect |
|---------|--------|
| `getfacl f` | Read ACL |
| `getfacl -R d/ > f.bak` | Back up a directory tree |
| `setfacl --restore=f.bak` | Restore |
| `setfacl -m u:user:rw f` | Add/modify a user entry |
| `setfacl -m g:group:rx f` | Add/modify a group entry |
| `setfacl -R -m u:user:rX d/` | Recursive, `X` = execute on directories only |
| `setfacl -d -m g:dev:rwx d/` | Default ACL (inheritance) |
| `setfacl -m m:rwx f` | Set the mask |
| `setfacl -x u:user f` | Remove an entry |
| `setfacl -b f` / `-k d/` | Erase everything / erase default ACL |
| `namei -l /path/file` | Find where a path breaks |

---

## Sudo / sudoers

`sudo` (*super user do*) allows executing a command as another user (`root` by default), after authentication with **your own** password.

This last point is what makes it the central administration tool: unlike `su`, no one needs to know the root password, each action is traced individually, and you can revoke someone's rights without changing everyone's password.

### Daily usage

```bash
sudo cat /var/log/syslog
```

| Command | Effect |
|---------|--------|
| `sudo command` | Execute as root |
| `sudo -u alice command` | Execute as another user |
| `sudo -g dev command` | Execute with another group |
| `sudo -i` | Open a root login shell (loads root's environment) |
| `sudo -s` | Open a root shell preserving current environment |
| `sudo -E command` | Preserve environment variables |
| `sudo -l` | **List what I'm allowed to do** |
| `sudo -k` | Forget cached authentication (re-ask password) |
| `sudo -v` | Extend authentication cache |
| `sudo !!` | Replay previous command with sudo |

The most useful command to understand your rights:

```bash
sudo -l
```

```text
Matching Defaults entries for alice on server:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User alice may run the following commands on server:
    (root) /usr/bin/systemctl restart apache2
    (root) NOPASSWD: /usr/local/bin/deploy.sh
```

```bash
sudo -l -U bob        # as admin: list another user's rights
```

> [!NOTE]
> If `/var/log/syslog` doesn't exist, it's because `rsyslog` is not installed:
> ```bash
> sudo apt install rsyslog
> ```

By default, you must belong to the `sudo` group (Debian/Ubuntu) or `wheel` group (RHEL/Fedora) to use `sudo`.

```bash
sudo usermod -aG sudo alice        # Debian
sudo usermod -aG wheel alice       # RHEL
```

> [!IMPORTANT]
> Adding to a group only takes effect at the **next session**. Alice must log out and back in — or run `newgrp sudo` — before `sudo` works for her.

### Editing sudoers without locking yourself out

> [!CAUTION]
> **Never edit `/etc/sudoers` directly with an editor.** A syntax error in this file makes `sudo` completely unusable, and locks you out of all admin access on a machine where root has no password (Ubuntu's default).
>
> ```bash
> sudo visudo
> ```
>
> `visudo` locks the file against concurrent edits and **validates syntax before saving**. On error, it offers to re-edit rather than write a broken file.

Choose your editor:

```bash
sudo EDITOR=nano visudo
sudo update-alternatives --config editor    # change the default on Debian
```

Verify a file without editing it:

```bash
sudo visudo -c
```

```text
/etc/sudoers: parsed OK
/etc/sudoers.d/deployment: parsed OK
```

**Best practice: don't touch `/etc/sudoers`.** Deposit your rules in a dedicated file under `/etc/sudoers.d/`, which makes them readable, reversible and deployable by configuration management:

```bash
sudo visudo -f /etc/sudoers.d/team_web
```

```text
# /etc/sudoers.d/team_web
# Managed by Ansible — do not edit by hand
%webops ALL=(root) /usr/bin/systemctl restart nginx, /usr/bin/systemctl reload nginx
```

```bash
sudo chmod 0440 /etc/sudoers.d/team_web
```

> [!WARNING]
> Two rules for `/etc/sudoers.d/`:
> - The file must be **0440** and owned by root, otherwise sudo silently ignores it.
> - Files whose name contains a `.` or ends with `~` are **ignored**. Name it `team_web`, not `team_web.conf`.

### File anatomy

```text
# /etc/sudoers

# ── Defaults: global sudo behavior ──────────────────────────
Defaults        env_reset
Defaults        mail_badpass
Defaults        secure_path="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"

# ── Aliases: named shortcuts ───────────────────────────────
User_Alias      ADMINS = alice, bob
Cmnd_Alias      SERVICES = /usr/bin/systemctl

# ── User rules ─────────────────────────────────────────────
root            ALL=(ALL:ALL) ALL

# ── Group rules (prefixed by %) ────────────────────────────
%sudo           ALL=(ALL:ALL) ALL
%admin          ALL=(ALL) ALL

# ── Include fragment directory — ALWAYS LAST ───────────────
@includedir /etc/sudoers.d
```

> [!IMPORTANT]
> **Order matters: the last matching rule wins.** If a user is covered by multiple rules, the lowest one in the file applies. That's why `@includedir /etc/sudoers.d` is on the last line: your fragments can override general rules.
>
> (On older versions, the directive is written `#includedir /etc/sudoers.d` — and the `#` is **not** a comment here.)

### Rule syntax

```text
alice    server1 = (root:web) NOPASSWD: /usr/bin/systemctl restart nginx
  │         │         │   │        │                    │
  │         │         │   │        │                    └─ COMMANDS allowed
  │         │         │   │        └─────────────────────── TAGS (optional)
  │         │         │   └──────────────────────────────── target group (optional)
  │         │         └──────────────────────────────────── target user
  │         └────────────────────────────────────────────── HOSTS where rule applies
  └──────────────────────────────────────────────────────── WHO (% = a group)
```

The canonical rule, for the `sudo` group:

```text
%sudo   ALL=(ALL:ALL)   ALL
```

It reads: *"members of the `sudo` group, from anywhere, can become any user and any group, to execute any command"*.

#### Restrict the impersonable identity

```text
test  ALL=(debian)    ALL
```

`test` can do everything, but only under the identity of `debian` — not root:

```bash
sudo -u debian command     # ✅ allowed
sudo command               # ❌ refused (target = root)
```

#### Restrict commands

```text
test ALL=(root)   /usr/bin/systemctl
```

`test` can only use `systemctl` as root — but **with any argument**, including `systemctl stop firewalld`.

```text
test ALL=(root)   /usr/bin/systemctl restart apache2
```

Here, the command is frozen **with its arguments**: they can only restart apache2.

```text
test ALL=(root)   /usr/bin/systemctl restart apache2, /usr/bin/systemctl status apache2
```

Multiple commands are separated by commas.

#### Forbid a command

```text
test ALL=(devops) ALL, !/bin/rm
```

`test` can impersonate `devops` for all commands except `rm`.

> [!CAUTION]
> **Blacklists don't work.** `!/bin/rm` is bypassed in three seconds:
>
> ```bash
> sudo -u devops cp /bin/rm /tmp/r && sudo -u devops /tmp/r file
> sudo -u devops vim -c ':!rm file'
> sudo -u devops find . -exec rm {} \;
> ```
>
> Any command capable of launching a shell, copying a binary, or executing an arbitrary program nullifies the restriction. **Always use a whitelist** of explicit commands, and verify that none allow escape.

### Tags

Tags modify behavior for the commands that follow, until the end of the rule.

| Tag | Effect |
|-----|--------|
| `NOPASSWD:` | No password asked |
| `PASSWD:` | Re-ask password (cancels a previous `NOPASSWD:`) |
| `NOEXEC:` | Prevent the command from launching other programs |
| `SETENV:` | Allow passing environment variables |
| `LOG_OUTPUT:` | Log the command's output |

```text
# Deployment is automated: no password for this specific script,
# but the rest of rights remain protected
deploy ALL=(root) NOPASSWD: /usr/local/bin/deploy.sh
deploy ALL=(root) PASSWD:   /usr/bin/systemctl restart nginx
```

> [!CAUTION]
> **Use `NOPASSWD` sparingly.** Each entry is an open door if the account is compromised: no need to know the password to escalate.
> Reserve it for specific, non-detourable commands, typically on non-interactive service accounts. `NOPASSWD: ALL` is equivalent to giving the root password.

The `NOEXEC` tag deserves to be known — it neutralizes shell escapes on commands that allow them:

```text
alice ALL=(root) NOEXEC: /usr/bin/less /var/log/secure
```

Without it, `less` would allow `!sh` and give a root shell.

### Defaults

`Defaults` lines pilot sudo's own behavior.

```text
# Basic behavior
Defaults    env_reset                       # start from a clean environment
Defaults    secure_path="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
Defaults    mail_badpass                    # mail admin on bad password

# Authentication cache duration, in minutes
Defaults    timestamp_timeout=15            # 0 = ask every time, -1 = never
Defaults    passwd_tries=3                  # number of attempts

# Logging
Defaults    logfile="/var/log/sudo.log"
Defaults    log_input, log_output           # full session logging
Defaults    iolog_dir="/var/log/sudo-io"

# Comfort
Defaults    pwfeedback                      # show asterisks on input
Defaults    editor="/usr/bin/vim"           # editor used by sudoedit
Defaults    !visiblepw                      # refuse if password would show in plaintext
```

`Defaults` can be targeted:

```text
Defaults:alice          timestamp_timeout=60      # for user alice
Defaults>root           !set_logname              # when target is root
Defaults!/usr/bin/vim   NOEXEC                    # for this command
Defaults@server-web    log_output                 # on this host
```

> [!IMPORTANT]
> `env_reset` and `secure_path` are **essential** protections: they prevent a user from hijacking a sudo command by manipulating `PATH` or `LD_PRELOAD`. Don't disable them to "make a script work" — pass the absolute path instead.

### Aliases

Aliases make the file readable and maintainable as the number of rules grows.

**1. Users**

```text
User_Alias  ADMINS  = debian, bob
User_Alias  JUNIORS = carol, dave, %interns
```

**2. Hosts**

```text
Host_Alias  SERVER_WEB = server1name, server2name
Host_Alias  SERVER_DB  = 192.168.1.20, 192.168.1.21
```

**3. Commands**

```text
Cmnd_Alias  MAINTENANCE = /usr/bin/apt, /usr/bin/systemctl
Cmnd_Alias  READ_LOG = /usr/bin/journalctl, /usr/bin/less /var/log/*
Cmnd_Alias  NETWORK      = /usr/sbin/ip, /usr/bin/ss, /usr/sbin/tcpdump
```

**4. Target identities**

```text
Runas_Alias  SERVICES = www-data, postgres, redis
```

Assembled:

```text
ADMINS   SERVER_WEB = (root)     MAINTENANCE
JUNIORS  SERVER_WEB = (root)     READ_LOG
ADMINS   SERVER_DB  = (SERVICES) ALL
```

### sudoedit — editing a protected file, properly

It's the most underutilized feature of sudo, and yet the right answer to a very common need: *modify a file belonging to root, with **my** editor and **my** configuration, without giving root rights to the editor.*

```bash
sudoedit /etc/nginx/nginx.conf
```

```bash
sudo -e /etc/nginx/nginx.conf      # equivalent syntax
```

**What actually happens:**

```text
1. sudo copies /etc/nginx/nginx.conf to a temporary file
2. It gives you ownership of it
3. It launches YOUR editor, as YOUR user with YOUR configuration
   → no root rights on the editor, nor on its plugins
4. On close, if the file changed, sudo copies the temp file back
   to its original location, with original rights and owner
5. The whole operation is logged
```

**Why this matters**, compared to `sudo vim /etc/nginx/nginx.conf`:

| | `sudo vim file` | `sudoedit file` |
|---|---|---|
| Editor user | **root** | you |
| Configuration loaded | **root**'s (`/root/.vimrc`) — not yours | **yours** (`~/.vimrc`, `~/.config/nvim/`) |
| Plugins executed | as root | with your rights |
| Temp swap file (`.swp`) | written as root, sometimes in the wrong place | in your space |
| Sudo right needed | `ALL` or the entire editor | `sudoedit /precise/path` |

> [!IMPORTANT]
> **The case of personalized configuration.** If you use Neovim with an elaborate configuration (LSP, plugins, theme), `sudo nvim` doesn't load it: root has its own `$HOME`, thus its own configuration — usually nonexistent. You end up in bare vi at the worst moment.
>
> The temptation is then to do `sudo -E nvim` to preserve `$HOME`, or worse, make `/root/.config` a symlink to yours. **This is exactly what you must not do**: you're then executing **as root** tens of thousands of lines of third-party plugins, downloaded from GitHub and auto-updated. One compromised plugin, and the machine is too.
>
> `sudoedit` solves it cleanly: your full configuration, your plugins, your LSP — all run as your user, and only the final file copy is privileged.

**Configure the editor used** — `sudoedit` looks, in order, at `$SUDO_EDITOR`, `$VISUAL`, `$EDITOR`, then `Defaults editor` from sudoers:

```bash
export SUDO_EDITOR=nvim        # in ~/.bashrc or ~/.zshrc
```

On the sudoers side, you must explicitly allow user environment variables, or `env_reset` erases them:

```text
Defaults  env_keep += "SUDO_EDITOR VISUAL EDITOR"
```

**Delegate editing of specific files** — this is where `sudoedit` truly shines in sudoers. You grant the right to modify a file, not the right to launch an editor as root:

```text
# The web team can modify vhosts, nothing else
%webops ALL=(root) sudoedit /etc/nginx/sites-available/*

# A developer can adjust their app's configuration
alice ALL=(root) sudoedit /etc/myapp/config.yml
```

```bash
sudo -l -U alice
```

```text
User alice may run the following commands on server:
    (root) sudoedit /etc/myapp/config.yml
```

> [!WARNING]
> In a sudoers rule, write `sudoedit` **without a path** — it's an internal sudo pseudo-command, not a standalone binary. `/usr/bin/sudoedit` in a rule doesn't work as expected.
>
> Also watch wildcards: `sudoedit /etc/nginx/*` authorizes `sudoedit /etc/nginx/../shadow` on old sudo versions. Stick to precise paths, and keep sudo updated.

### Complete practical cases

#### 1. An operator who only manages services

```text
# /etc/sudoers.d/ops_services
Cmnd_Alias SVC_WEB = /usr/bin/systemctl start nginx,   \
                     /usr/bin/systemctl stop nginx,    \
                     /usr/bin/systemctl restart nginx, \
                     /usr/bin/systemctl reload nginx,  \
                     /usr/bin/systemctl status nginx

Cmnd_Alias SVC_LOG = /usr/bin/journalctl -u nginx *

%ops ALL=(root) SVC_WEB, SVC_LOG
```

#### 2. An automated deployment account

```text
# /etc/sudoers.d/deploy
# Non-interactive account used by CI — SSH key restricted on authorized_keys
deploy ALL=(root) NOPASSWD: /usr/local/bin/deploy.sh
deploy ALL=(root) NOPASSWD: /usr/bin/systemctl restart myapp

Defaults:deploy !requiretty
Defaults:deploy log_output
```

> [!CAUTION]
> The script `/usr/local/bin/deploy.sh` must absolutely belong to **root** and be modifiable only by root (`chown root:root`, `chmod 755`). If modifiable by the `deploy` user, they can write anything to it and get a root shell — the sudo rule becomes a disguised `NOPASSWD: ALL`.
>
> ```bash
> sudo chown root:root /usr/local/bin/deploy.sh
> sudo chmod 755 /usr/local/bin/deploy.sh
> ```

#### 3. A team managing its own configurations

```text
# /etc/sudoers.d/team_app
%devs ALL=(root) sudoedit /etc/myapp/*.yml
%devs ALL=(root) /usr/bin/systemctl restart myapp
%devs ALL=(myapp) /bin/bash          # shell as the service user, not root
```

#### 4. A backup account

```text
# /etc/sudoers.d/backup
backup ALL=(root) NOPASSWD: /usr/bin/rsync --server --sender *
backup ALL=(root) NOPASSWD: /usr/bin/tar --acls -czf * /srv/data
```

### Security pitfalls

> [!CAUTION]
> **1. Overly broad wildcards.** A `*` in a command covers much more than imagined:
>
> ```text
> alice ALL=(root) /usr/bin/chown * /var/www
> ```
>
> This rule authorizes `sudo chown alice /etc /var/www` — the wildcard absorbs extra arguments.

> [!CAUTION]
> **2. Modifiable scripts.** Any command authorized via sudo must belong to root and be modifiable only by root — the binary and all its path directories.

> [!CAUTION]
> **3. Relative paths.** A rule must always use an absolute path. `alice ALL=(root) systemctl` (without a path) doesn't work and provides no guarantee.

Quick audit of rights granted on a machine:

```bash
# All effective rules, all files combined
sudo grep -rvE '^\s*($|#)' /etc/sudoers /etc/sudoers.d/

# The NOPASSWDs, to review in priority
sudo grep -rn "NOPASSWD" /etc/sudoers /etc/sudoers.d/

# Who's in the sudo group?
getent group sudo wheel
```

### Audit and logging

Every use of sudo is logged, successful or not.

```bash
sudo cat /var/log/auth.log            # Debian / Ubuntu
sudo cat /var/log/secure              # RHEL / Fedora
sudo journalctl -t sudo
sudo journalctl _COMM=sudo --since today
```

```text
Dec 12 09:31:01 server sudo:    alice : TTY=pts/0 ; PWD=/home/alice ;
  USER=root ; COMMAND=/usr/bin/systemctl restart nginx
Dec 12 09:33:22 server sudo:      bob : user NOT in sudoers ; TTY=pts/1 ;
  PWD=/home/bob ; USER=root ; COMMAND=/usr/bin/apt install htop
```

**Session logging** is the advanced audit feature: sudo can record **everything typed and displayed** during a command, and replay it.

```text
Defaults  log_input, log_output
Defaults  iolog_dir="/var/log/sudo-io"
```

```bash
sudo sudoreplay -l                    # list recorded sessions
```

```text
Dec 12 10:15:03 2025 : alice : TTY=/dev/pts/0 ; CWD=/home/alice ;
  USER=root ; TSID=000001 ; COMMAND=/bin/bash
```

```bash
sudo sudoreplay 000001                # replay the session, real-time
sudo sudoreplay -s 10 000001          # replay 10× faster
```

> [!TIP]
> This is extremely useful in training and incident post-mortem: you see exactly what was done, in order, including typos. Deploy at least for accounts with a root shell.

### Sudo diagnosis

| Message / symptom | Cause | Fix |
|--------------------|-------|------|
| `alice is not in the sudoers file` | Not member of `sudo`/`wheel`, no rule | `usermod -aG sudo alice` then log back in |
| `Sorry, user alice is not allowed to execute …` | Command matches no rule | `sudo -l` to see what's allowed; check absolute path and arguments |
| `sudo: no tty present and no askpass program` | Non-interactive execution (cron, CI) without `NOPASSWD` | Add `NOPASSWD:` for this specific command |
| `sudo: /etc/sudoers.d/xxx is mode 0644, should be 0440` | Wrong permissions — file is **ignored** | `chmod 0440` and `chown root:root` |
| Command works manually but not via sudo | `secure_path` replaces your `PATH` | Use absolute path of the command |
| `sudo: unable to resolve host server` | Hostname not in `/etc/hosts` | Add `127.0.1.1 server` to `/etc/hosts` |

**Getting out of a broken sudoers** — by order of preference:

```bash
# 1. A root session is still open somewhere: fix immediately
visudo -c && visudo

# 2. pkexec, if installed (PolicyKit, independent of sudo)
pkexec visudo

# 3. Root account has a password
su -

# 4. Last resort: boot into rescue mode from GRUB
#    (add "init=/bin/bash" to kernel parameters, then remount / in rw)
mount -o remount,rw /
visudo
```

> [!TIP]
> **The reflex that avoids all this:** before modifying sudoers on a remote machine, open a **second SSH session as root** and leave it open. If you break something, you still have access to fix it. It's the same reflex as for SSH and firewall — we come back to this in later sections.

### Sudo summary

| Command | Role |
|---------|------|
| `sudo -l` | List my own rights |
| `sudo -l -U bob` | List another user's rights |
| `sudo -u alice cmd` | Execute under another identity |
| `sudo -i` / `sudo -s` | Root shell (login / non-login) |
| `sudoedit file` | Edit a protected file with **my own** editor |
| `visudo` | Edit `/etc/sudoers` safely |
| `visudo -c` | Validate syntax |
| `visudo -f /etc/sudoers.d/x` | Create a rule fragment |
| `sudoreplay -l` / `sudoreplay ID` | List / replay a recorded session |
| `journalctl -t sudo` | View logs |

---

[Document continues with Firewall, SSH, and AppArmor/SELinux sections...]
