# 04 · Package Installation, Services and Scheduled Tasks

> [!NOTE]
> **Chapter Objectives**
> - Install, update, search for and remove packages (Debian and RedHat)
> - Control services with `systemctl`
> - Write your own `systemd` service
> - View logs with `journalctl`
> - Schedule recurring tasks with `cron`

## Table of Contents

1. [Package Management](#package-management)
2. [Service Management](#service-management)
3. [Creating a Custom Service](#creating-a-custom-service)
4. [Journald](#journald)
5. [CRON](#cron)
6. [Summary](#summary)

---

## Package Management

Two major distribution families, two package managers:

| Family | Distributions | Manager |
|--------|---------------|---------|
| **Debian** | Debian, Ubuntu, Mint, Raspberry Pi OS | `apt` |
| **RedHat** | RHEL, CentOS, Fedora, Rocky, Alma | `yum` (old) / `dnf` (current) |
| **Alpine** | Alpine (usually with docker)| `apk` |
| **ArchLinux** | ArchLinux, Garudam, ChatyOs, Manjaro (never on server!!)| `pacman` |

### Pre-Installation / Index Update

Before installing a package, update the **local** package database to retrieve the latest available versions.

On Debian:

```bash
sudo apt update
```

On RedHat, this operation is integrated into installation.

> [!IMPORTANT]
> `apt update` updates the **list** of available packages; it installs nothing.
> `apt upgrade` actually installs new versions. The two always go together.

### Install

On Debian-based:

```bash
sudo apt install package_name1 package_name2 ...
```

On RedHat-based:

```bash
sudo yum install package ...
```

```bash
sudo dnf install package1 ...
```

### Update

On Debian (must be preceded by `apt update`):

```bash
sudo apt upgrade
```

```bash
sudo apt upgrade package_name
```

On RedHat:

```bash
sudo yum update
```

```bash
sudo dnf update
```

### Remove

On Debian:

```bash
sudo apt remove package_name
```

On RedHat:

```bash
sudo yum remove package_name
```

```bash
sudo dnf remove package_name
```

> [!TIP]
> `apt remove` leaves configuration files in place; `apt purge` removes them too.
> `sudo apt autoremove` cleans up unused dependencies.

### Search

Equivalent to searching for an application in an app store.

On Debian:

```bash
apt search package_name
```

On RedHat:

```bash
yum search package_name
```

```bash
dnf search package_name
```

### Correspondence Table

| Action | Debian (`apt`) | RedHat (`dnf`) |
|--------|----------------|----------------|
| Update index | `apt update` | *(automatic)* |
| Install | `apt install pkg` | `dnf install pkg` |
| Update | `apt upgrade` | `dnf update` |
| Remove | `apt remove pkg` | `dnf remove pkg` |
| Search | `apt search pkg` | `dnf search pkg` |
| Info on a package | `apt show pkg` | `dnf info pkg` |
| List installed | `apt list --installed` | `dnf list installed` |

---

## Service Management

Services are programs running in the background on the machine (the server).

A few examples:

- HTTP/HTTPS server
- SSH server
- Database server
- …

### The systemctl Commands

| Command | Effect |
|----------|-------|
| `sudo systemctl enable name` | **Enable**: start the service at machine boot |
| `sudo systemctl disable name` | Disable automatic startup |
| `sudo systemctl start name` | Start the service **now** |
| `sudo systemctl stop name` | Stop the service |
| `sudo systemctl restart name` | Restart the service |
| `sudo systemctl reload name` | Reload its configuration **without** stopping the service |
| `sudo systemctl status name` | Inspect the current state of the service |

```bash
sudo systemctl enable service_name
```

```bash
sudo systemctl start service_name
```

```bash
sudo systemctl status service_name
```

> [!IMPORTANT]
> `enable` ≠ `start`. `enable` schedules startup **at next boot**, `start` starts **right now**.
> To do both at once: `sudo systemctl enable --now service_name`.

### List Services

```bash
systemctl list-units --type=service
```

```bash
systemctl list-units --type=service --state=running
```

```bash
systemctl list-units --type=service --state=active
```

List all failed services:

```bash
systemctl list-units --type=service --state=failed
```

---

## Creating a Custom Service

You need to create a file for this service (usually named `service_name.service`) in `/etc/systemd/system/SERVICENAME.service`:

```ini
[Unit]
Description=Example service
After=multi-user.target

[Service]
Type=simple
Restart=always
RestartSec=1
User=SERVICE_USERNAME
ExecStart=SERVICE_COMMAND

[Install]
WantedBy=multi-user.target
```

- Replace `SERVICE_USERNAME` with the name of the user the service will be tied to.
- Replace `SERVICE_COMMAND` with the command run by the service.

> [!WARNING]
> **Always use absolute paths** for commands and files used: `systemd` doesn't have the same `PATH` as your shell.
> Also verify that the service user has the right to execute this command and access the relevant files.

Find the absolute path of a command:

```bash
type -a COMMAND
```

### Available Targets

| systemd target | SystemV Runlevel | Alias | Description |
|----------------|------------------|-------|-------------|
| `default.target` | | | Always a symbolic link to `multi-user.target` or `graphical.target`. This is what systemd uses to start the system. Never alias it to `halt`, `poweroff` or `reboot`. |
| `graphical.target` | 5 | `runlevel5.target` | `multi-user.target` with a graphical interface |
| | 4 | `runlevel4.target` | Unused. Identical to runlevel 3 under SystemV. Can be created and customized to start local services without modifying `multi-user.target`. |
| `multi-user.target` | 3 | `runlevel3.target` | All services started, command line only |
| | 2 | `runlevel2.target` | Multi-user, without NFS, all other non-graphical services started |
| `rescue.target` | 1 | `runlevel1.target` | Basic system: filesystems mounted, minimal services and emergency shell on console |
| `emergency.target` | S | | Single-user mode — no services, no filesystems mounted. The most basic level, with only an emergency shell. |
| `halt.target` | | | Halt the system without cutting power |
| `reboot.target` | 6 | `runlevel6.target` | Reboot |
| `poweroff.target` | 0 | `runlevel0.target` | Halt the system and cut power |

### Complete Example

```ini
[Unit]
Description=Example service
After=multi-user.target

[Service]
Type=simple
Restart=always
RestartSec=1
User=debian
ExecStart=/usr/bin/node /home/debian/test.js

[Install]
WantedBy=multi-user.target
```

Then reload systemd and enable and start the service:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now my_service
```

> [!IMPORTANT]
> The `daemon-reload` is essential after **any** creation or modification of a `.service` file: without it, systemd continues to use the old version in memory.

---

## Journald

`journald` is Linux's logging tool under systemd. Its usage is relatively simple:

```bash
journalctl -u service_name
```

Example:

```bash
journalctl -u apache2
```

| Option | Effect |
|--------|--------|
| `-u name` | Filter on a **systemd unit** (the most common case) |
| `-t name` | Filter on a *syslog identifier* (the name the program gives to its logs) |
| `-f` | Follow logs in real-time |
| `-n 50` | Display the last 50 lines |
| `--since "1 hour ago"` | Filter by date |
| `-p err` | Keep only messages of a given priority level |

---

## CRON

Cron allows you to run tasks automatically at specific times.

May require installation:

```bash
sudo apt install cron
```

Cron is itself a service:

```bash
# Debian
systemctl status cron

# RedHat
systemctl status crond
```

### Manage Your Crontab

| Command | Effect |
|----------|--------|
| `crontab -e` | Edit your scheduled tasks |
| `crontab -l` | List your scheduled tasks |
| `crontab -r` | **Delete** your entire crontab |

> [!CAUTION]
> `crontab -r` deletes your entire crontab without confirmation — and `-r` is right next to `-e` on the keyboard. Keep a backup of your crontabs: `crontab -l > ~/crontab.bak`.

### Crontab Syntax

```text
# Example of job definition:
# .---------------- minute (0 - 59)
# |  .------------- hour (0 - 23)
# |  |  .---------- day of month (1 - 31)
# |  |  |  .------- month (1 - 12) OR jan,feb,mar,apr ...
# |  |  |  |  .---- day of week (0 - 6) (Sunday=0 or 7) OR sun,mon,tue,wed,thu,fri,sat
# |  |  |  |  |
# *  *  *  *  * command to be executed
```

For example, to run the `ls` command every minute:

```text
* * * * * /usr/bin/ls
```

Every hour (at XXh00):

```text
0 * * * * /usr/bin/ls
```

### Common Expressions

| CRON Expression | Meaning |
|-----------------|---------|
| `* * * * *`   | Every minute |
| `0 * * * *`   | Every hour (XXh00) |
| `30 2 * * *`  | Every day at 2:30 AM |
| `0 8 * * 1-5` | Monday to Friday at 8:00 AM |
| `*/5 * * * *` | Every 5 minutes |
| `0 */2 * * *` | Every 2 hours |
| `0 0 1 1 *`   | Every January 1st |
| `@reboot`     | At every boot |
| `@yearly`     | ⇒ `0 0 1 1 *` |
| `@monthly`    | Every month |
| `@weekly`     | Every week |
| `@daily`      | Every day |
| `@hourly`     | Every hour |

### System Crontab

There is also a system crontab in `/etc/crontab` (should not be modified in principle: user crontabs are sufficient).

This one has an **additional field**: the user who will run the task.

```bash
cat /etc/crontab
```

```text
# /etc/crontab: system-wide crontab
# Unlike any other crontab you don't have to run the `crontab'
# command to install the new version when you edit this file
# and files in /etc/cron.d. These files also have username fields,
# that none of the other crontabs do.

SHELL=/bin/sh
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin

# Example of job definition:
# .---------------- minute (0 - 59)
# |  .------------- hour (0 - 23)
# |  |  .---------- day of month (1 - 31)
# |  |  |  .------- month (1 - 12) OR jan,feb,mar,apr ...
# |  |  |  |  .---- day of week (0 - 6) (Sunday=0 or 7) OR sun,mon,tue,wed,thu,fri,sat
# |  |  |  |  |
# *  *  *  *  * user-name command to be executed
17 *	* * *	root	cd / && run-parts --report /etc/cron.hourly
25 6	* * *	root	test -x /usr/sbin/anacron || { cd / && run-parts --report /etc/cron.daily; }
47 6	* * 7	root	test -x /usr/sbin/anacron || { cd / && run-parts --report /etc/cron.weekly; }
52 6	1 * *	root	test -x /usr/sbin/anacron || { cd / && run-parts --report /etc/cron.monthly; }
#
```

### Managing Cron Task Output

By default, on some systems (notably RedHat), cron sends task output **by mail** (if an MTA is available). It is therefore useful to explicitly redirect the output of commands run via cron:

```text
* * * * * /bin/ls > /dev/null 2>&1
```

In this example, I redirect output to `/dev/null` and redirect errors to standard output (so, also to `/dev/null`).

In most cases, we redirect to a log file instead:

```text
* * * * * /bin/ls >> /tmp/customlog
```

```text
* * * * * /bin/ls >> /home/debian/cronlogs/customlog
```

### View Cron Logs

```bash
sudo journalctl -t CRON
```

Or via syslog (**requires `rsyslog`**):

```bash
sudo apt install rsyslog
```

```bash
cat /var/log/syslog | grep CRON
```

```text
2025-12-12T09:31:01.451982+00:00 balrog-c2-srv CRON[2708800]: (debian) CMD (/usr/bin/echo "Hello" >> /home/debian/hello.txt)
2025-12-12T09:32:01.459068+00:00 balrog-c2-srv CRON[2708812]: (debian) CMD (/usr/bin/echo "Hello" >> /home/debian/hello.txt)
2025-12-12T09:33:01.465445+00:00 balrog-c2-srv CRON[2708824]: (debian) CMD (/usr/bin/echo "Hello" >> /home/debian/hello.txt)
```

> [!IMPORTANT]
> These logs show that the command **was run**, but not what it produced. You must therefore manage the output logs of your scripts yourself.

### Best Practices

> [!TIP]
> - Always use **absolute paths** (cron has minimal `PATH` and doesn't execute your `.bashrc`).
> - Always **test the script manually** before putting it in a cron.
> - **Redirect outputs** to facilitate debugging.
> - **Document** your crons (a comment above each line).
> - **Check regularly** the execution logs of your crons.

---

## Summary

| Command | Role |
|----------|------|
| `apt update` / `apt upgrade` | Update index / packages |
| `apt install` / `apt remove` | Install / remove a package |
| `systemctl start` / `stop` / `restart` | Control a service |
| `systemctl enable --now` | Enable at boot **and** start |
| `systemctl status` | Status of a service |
| `systemctl daemon-reload` | Reload after modifying a `.service` |
| `journalctl -u name -f` | Follow a service's logs |
| `crontab -e` / `-l` | Edit / list your scheduled tasks |

---

⬅️ [Previous: 03 · Essential Commands](03_commandes_essentielles.md) · 🏠 [Table of Contents](README.md) · [Next: 05 · Security ➡️](05_securite.md)
