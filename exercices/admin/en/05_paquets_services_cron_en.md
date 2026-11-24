# 📦 A5 · Packages, Services and Scheduled Tasks

*Chapter [04 · Installing Packages, Services and Scheduled Tasks](../../04_installation_et_services.md)*

> [!NOTE]
> **Objectives**: master the package lifecycle, control and diagnose services, write a clean systemd unit for an in-house application, exploit `journalctl`, and schedule what needs to be scheduled — with both `cron` and systemd *timers*.

> [!IMPORTANT]
> A **real VM** is essential: a container does not run `systemd`.

---

## Exercise 5.1 — Packages

1. Update the package index, then display the list of packages that **can** be updated,
   without installing anything.
2. Apply security updates. What is the difference between `upgrade`, `full-upgrade` and
   `dist-upgrade`?
3. Install `nginx`, `postgresql-client`, `ufw`, `fail2ban` and `acl` in one command.
4. Search for the package that provides the `ss` command, then display its detailed information.
5. List the files installed by the `nginx` package. Which package does `/usr/bin/systemctl` belong to?
6. Remove `nginx` while keeping its configuration, then remove it **with** its configuration.
   What is the difference, and which one do you use when you want to start fresh?
7. Clean up unused dependencies and download cache.
8. A package was installed manually: how do you find the list of packages **explicitly
   installed** by a human, as opposed to dependencies?
9. Prevent automatic updates of a specific package *(hint: `apt-mark hold`)*. When do you do that,
   and what is the risk?

---

## Exercise 5.2 — Controlling Services

1. List all **active** services, then all those that are **failed**.
2. Display the status of `ssh`: is it running? is it enabled on boot? how long has it been running? what is its PID?
3. Stop, start, restart `nginx`. Reload its configuration **without** cutting existing connections.
4. What is the exact difference between `restart` and `reload`? And between `enable` and `start`?
5. Enable `nginx` on boot **and** start it in a single command.
6. Mask a service to prevent any activation, even by dependency. Then unmask it.
   When do you use `mask` instead of `disable`?
7. Display the list of dependencies of `nginx.service`, then what depends on it.
8. How long did the last machine boot take, and which services are the slowest?
   *(Hint: `systemd-analyze`.)*

---

## Exercise 5.3 — A Unit for the In-House Application

The `ticketflow` API must run continuously, under a dedicated account, and restart if it crashes.

```bash
sudo tee /usr/local/bin/ticketflow-api >/dev/null <<'EOF'
#!/bin/bash
echo "starting ticketflow API (pid $$)"
trap 'echo "stop requested"; exit 0' TERM INT
while true; do
  echo "$(date '+%F %T') api ok"
  sleep 15
done
EOF
```

1. Set the correct owner and permissions on this script for a system executable.
2. Write `/etc/systemd/system/ticketflow-api.service` with:
   - a clear description;
   - startup **after the network** and after `postgresql.service`;
   - `Type=simple`, execution under user `deploy` and group `ops`;
   - automatic restart on failure, with a 5-second delay;
   - working directory `/srv/ticketflow`;
   - an environment variable `NODE_ENV=production`;
   - installation in `multi-user.target`.
3. Reload the systemd configuration, enable the service on boot **and** start it in one command.
4. Check its status and read its logs.
5. Brutally kill the process: does systemd restart it? After how long?
6. Add three hardening directives (`NoNewPrivileges`, `PrivateTmp`, `ProtectSystem`) and verify
   that the service still restarts. What does each one do?
7. Make it read configuration from an environment file `/etc/default/ticketflow`
   instead of hardcoding it in the unit.
8. How do you override a directive of a unit **provided by a package** without modifying its original file
   (which would be overwritten on the next update)? *(Hint: `systemctl edit`.)*

---

## Exercise 5.4 — `journalctl`

1. Logs from the `ticketflow-api` service, then its last 50 lines, then in real-time follow mode.
2. Logs from "15 minutes ago", then those from today, then those from the **previous boot**.
3. Only messages at `error` level and above, for the entire system.
4. Kernel logs only.
5. Logs from a specific user, then from a specific PID.
6. How much space does the journal take up on disk? Limit it to 500 MB **permanently**.
7. Is the journal persistent after a reboot on this machine? How do you make it persistent?
8. Purge entries older than 30 days.
9. Export the service logs from today into a file, to attach to an incident report.

---

## Exercise 5.5 — `cron`

1. Display your crontab, then that of user `deploy`.
2. Write, in root's crontab, a task that runs `/usr/local/bin/backup.sh`
   **every day at 2:30 AM**, redirecting both standard output **and** errors to a log file.
3. A task that, **every 10 minutes**, writes the remaining disk space to a file.
4. A task that, **Monday through Friday, every hour from 8 AM to 6 PM**, runs a health check.
5. A task that, **on the first of each month**, archives the logs from the past month.
6. A task that runs **at every machine reboot**.
7. Write the same daily backup as a fragment `/etc/cron.d/ticketflow-backup` —
   what additional column does this format require?
8. What are `/etc/cron.daily` and similar for? When do you prefer them to a crontab line?
9. Your task "doesn't work" even though the command works in your terminal.
   Name the **three** most common causes and how you verify them.
10. Forbid the use of `cron` to all users except `root` and `deploy`
    *(hint: `/etc/cron.allow`)*.

---

## Exercise 5.6 — Systemd *timers*

1. List the active *timers* on the machine and the next scheduled time for each.
2. Write the pair `ticketflow-backup.service` + `ticketflow-backup.timer` that replaces the cron task
   from exercise 5.5.2: type `oneshot`, daily trigger at 2:30 AM, with catch-up if the machine
   was powered off *(hint: `Persistent=true`)*.
3. Enable the *timer*, verify its next execution, manually trigger the service to test.
4. Name three advantages of a systemd *timer* over a crontab line.

---

## Exercise 5.7 — `logrotate`

1. Look at how `/etc/logrotate.d/` is organized.
2. Write `/etc/logrotate.d/ticketflow`: daily rotation of logs from `/var/log/ticketflow`,
   7 rotations kept, compression, no error if the file is missing, and creation of the new
   file with correct permissions and owner.
3. Test your configuration **in dry-run mode**, without actually rotating *(hint: `-d`)*.
4. Force an immediate rotation to verify the result.
5. Why can an application that keeps its log file open continue writing to the old
   file after rotation, and what directive fixes the problem?

---

## ✅ Verification

- `systemctl status ticketflow-api` shows an active, hardened service, running under `deploy`, automatically
  restarted after a `kill -9`.
- `systemctl --failed` returns nothing.
- The daily backup exists **either** as cron **or** as a *timer*, and you can justify your choice.
- `journalctl -u ticketflow-api --since today` shows traces of your tests.
- `logrotate -d /etc/logrotate.d/ticketflow` reports no errors.
- Your `journal.md` lists everything that is scheduled on the machine, with the time and reason.
