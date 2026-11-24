# 🧰 A4 · The Toolkit

*Chapter [03 · Essential Commands](../../03_commandes_essentielles.md)*

> [!NOTE]
> **Objectives**: exploit logs and system files with `grep`, `sed`, `awk`, `cut`/`sort`/`uniq`, find anything with `find` and `xargs`, diagnose a full disk with `du` and `df`, archive with `tar`. This is the toolkit you use all day, every day.

---

## 🧰 Datasets

If your server is too new to have interesting logs, use this test dataset:

```bash
mkdir -p /srv/labo/{logs,conf,archives,web} && cd /srv/labo

cat > logs/auth.log <<'EOF'
Mar  1 08:02:11 srv-tickets sshd[1420]: Accepted publickey for alice from 10.0.0.14 port 51422 ssh2
Mar  1 08:14:02 srv-tickets sshd[1503]: Failed password for root from 203.0.113.9 port 39112 ssh2
Mar  1 08:14:05 srv-tickets sshd[1503]: Failed password for root from 203.0.113.9 port 39112 ssh2
EOF

cp /etc/ssh/sshd_config conf/sshd_config.copie
head -c 40M /dev/zero > archives/dump-2023.bin
head -c 12M /dev/zero > logs/vieux.log
touch -d '2 years ago' archives/dump-2023.bin
touch web/index.html web/.env conf/vhost.conf logs/old.tmp
```

---

## Exercise 4.1 — `grep`: Search Logs

On `logs/auth.log`:

1. All **failed** authentication attempts, with line numbers.
2. The **count** of these attempts.
3. **Successful** logins, case-insensitive.
4. All lines **except** those about `alice`.
5. Lines containing `Failed` **or** `NOT in sudoers`, in one command.
6. **Recursive** search for `203.0.113.9` in all `/srv/labo`, showing only filenames.
7. On `/etc/ssh/sshd_config`: all active directives (lines that are neither empty nor commented).

---

## Exercise 4.2 — `sed`: Modify Configurations

On `conf/sshd_config.copie` — **never on the real file until the command is validated**:

1. Display the file with `#PermitRootLogin` uncommented and set to `no`, **without** modifying it.
2. Make the actual modification, keeping a backup.
3. Change the port from `22` to `2222`, touching only the `Port` directive.
4. Comment all lines starting with `PasswordAuthentication`.
5. Remove empty lines **and** commented lines at once, write to `conf/sshd_config.clean`.
6. Display only lines 10 to 25 of the original file.
7. Why is changing the separator (`s|…|…|`) essential when manipulating paths?

> [!CAUTION]
> `sed -i` overwrites without confirmation or undo. On a file in `/etc`, always run the command without `-i` first, and keep a `.bak` until the service restarts without error.

---

## Exercise 4.3 — `awk`: Analyze

1. On `/etc/passwd`: name and UID of all accounts with **UID ≥ 1000**.
2. On `/etc/passwd`: accounts with shell **other than** `/usr/sbin/nologin` or `/bin/false`.
3. On `logs/auth.log`: the **top IPs** originating failed authentications, most persistent first.
4. On `logs/auth.log`: list of users targeted by these failures.
5. On `logs/nginx-access.log`: number of requests **per HTTP code**, in an `END` block.
6. On `logs/nginx-access.log`: IPs that caused at least one `500` error.
7. On `logs/nginx-access.log`: **total volume** of bytes served, displayed in MB.
8. On `ps aux` output: total memory consumed by one user's processes.

---

## Exercise 4.4 — `cut`, `sort`, `uniq`, `wc`, `tr`

1. List of usernames from `/etc/passwd`, with `cut`.
2. Count of accounts on the machine, in one command.
3. Sorted, deduplicated list of IPs in `nginx-access.log`, with occurrence counts.
4. Shells used on the machine, deduplicated and counted — how many accounts per shell?
5. From `df -h` output, extract mountpoint and usage percentage, aligned, compressing spaces with `tr -s` then `cut`.

---

## Exercise 4.5 — `find`: Locate

1. All `.conf` files under `/etc`, without permission errors on screen.
2. All files larger than **10 MB** under `/srv` and `/var`.
3. Files **unmodified for over a year** in `/srv/labo/archives`.
4. Files modified **less than 60 minutes ago** under `/etc`.
5. Files owned by a given user, outside their *home*.
6. Files **world-writable** under `/srv`: why is this an audit point?
7. All `.tmp` files in `/srv/labo`, listed then deleted.
8. Details (`ls -lh`) of each `.log` found, via `-exec`.

---

## Exercise 4.6 — `xargs`: Chain

1. Search `PermitRootLogin` **only in `.conf` files** of `/etc/ssh/`, combining `find` and `xargs grep`.
2. Copy all `.conf` from `/srv/labo/conf` to `/srv/labo/archives/conf/` with `xargs -I {}`.
3. Count total lines of all logs in `/srv/labo/logs`.
4. A file named `my final report.log` breaks naive `find … | xargs rm`: create it, see it fail, then write the **robust** version.
5. Write the command that compresses all `.log` files older than 7 days — first with `-exec`, then with `xargs`. What performance difference, and why?

---

## Exercise 4.7 — The Disk is Full

It's Friday evening. `/` is at 98%.

1. Display all partition usage, human-readable.
2. Display weight of each top-level folder of `/`, without going deeper.
3. Descend: find the **10 largest folders** in `/var`.
4. Find the **10 largest files** on the machine.
5. `df` says disk is full but `du` sums don't match. Name **two** classic causes.
6. How many **inodes** are free?
7. Write the line that compresses all logs older than 30 days in `/var/log/ticketflow` — and explain why `logrotate` does it better.

---

## Exercise 4.8 — `tar`: Backup

1. Archive `/etc` to `/srv/ticketflow/backups/etc-<date>.tar.gz`.
2. List archive contents without extracting.
3. Extract **one single file** to a restore folder.
4. Create an archive of `/srv/labo` **excluding** `archives/` and `.tmp` files.
5. Create an archive containing exactly the files returned by a `find`.
6. What option preserves owners, rights and ACL? Why is this vital for system backup?
7. Verify integrity of an existing archive and produce its checksum.

---

## Exercise 4.9 — Diagnostic Utilities

1. Write an audit's output to both screen **and** a file, one command.
2. Compare `conf/sshd_config.copie` and the original `sshd_config`, in readable format.
3. Display the real type of `archives/dump-2023.bin` — does the extension tell the truth?
4. Display all metadata of `/etc/shadow`: rights, owner, dates, inode.
5. Is `systemctl` a binary, an alias, a function? Where is it located?
6. What command would redisplay `df -h` every 5 seconds highlighting changes?

---

## Exercise 4.10 — The Incident Report

One command line per question.

1. The **top 5 IPs** that failed to authenticate, with attempt counts.
2. List of users who used `sudo` today, deduplicated.
3. IPs requesting suspicious paths (`.env`, `wp-login`, `admin.php`) in nginx logs.
4. Count of `5xx` errors per IP, sorted.
5. An archive `/srv/ticketflow/backups/incident-<date>.tar.gz` containing all log files mentioning the offending IP.
6. A file `/root/incident-report.txt` gathering results from the five points above, each preceded by a title.

---

## ✅ Verification

- You produce the top attacking IPs from an `auth.log` without hesitation on command order.
- You can answer "why is the disk full?" in fewer than five commands.
- `/srv/ticketflow/backups/` contains a restorable archive of `/etc` **with its rights**.
- `/root/incident-report.txt` exists and reads cleanly.
- You can explain each layer of each of your pipes.
