# 🎓 A12 · Final Project — Secure and Operate `srv-tickets`

*Entire track: chapters [01](../../01_base.md) to [06](../../06_shellscript/), [08](../../08_docker_administration.md),
[09](../../09_tmux.md), [10](../../10_shell_config.md) and [11](../../11_advanced_config.md)*

> [!NOTE]
> **Objective**: take over an abandoned machine and make it **tenable** — that is,
> documented, hardened, backed up, monitored and reproducible. The deliverable is not a command:
> it's a server you can account for.

> [!CAUTION]
> Start from a **fresh VM** with the existing setup described in the [track README](README.md#-the-red-thread--srv-tickets):
> everyone as root, SSH by password, no firewall, no backup. Snapshot at each step.

---

## 🎯 Requirements

You have one (fictional) week and a production server that **you are not allowed to bring down**
for more than a few seconds. At the end, an auditor arrives: they must be able to verify each point below
by reading your documentation and running your commands.

---

## Step 0 — Initial Audit *(chapters [01](../../01_base.md) and [03](../../03_commandes_essentielles.md))*

Produce `/root/audit-initial.md`, generated as much as possible by commands:

1. System: distribution, kernel, *uptime*, resources, partitions and usage rates.
2. Accounts: humans, systems, those with a shell, those in `sudo`, those without a password.
3. Active services, failed services, services **listening on external interfaces**.
4. Open ports, seen from the machine **and** from outside.
5. Scheduled tasks (crontabs of all users, `/etc/cron.*`, *timers*).
6. SUID files, *world-writable* files, files owned by non-existent UIDs.
7. What Docker is running, with what permissions and what ports.
8. The five risks you deem most urgent, ranked, with one justifying sentence each.

## Step 1 — Accounts *(chapters [02](../../02_file_permissions.md) and [02.1](../../02.1_user_management.md))*

1. Create a personal account per person, with the right group (`ops`, `dev`), the right expiration
   policy and the right shell.
2. Create technical accounts `deploy` and `backup`, without login shell.
3. Set correct permissions and ownership on `/srv/ticketflow`, `/var/log/ticketflow` and your scripts —
   folders and files should not have the same.
4. Set SGID and sticky bit where appropriate, and justify each in the log.
5. Remove from `root` the ability to connect directly, once — and only once —
   a personal account works with `sudo`.

## Step 2 — Delegation *(chapter [05](../../05_securite.md), sudo section)*

1. One `sudoers.d` fragment per need, named and commented: `ops`, `dev`, `deploy`, `backup`.
2. `dev` can only restart and check application services. `deploy` can run only one script,
   without a password, and doesn't own this script.
3. The ACL of `/srv/ticketflow` respects the table from exercise [6.4](06_acl_et_sudo.md).
4. `sudo -l` run with each account proves the requirements are met — copy outputs
   to the log.
5. The rescue procedure "broken `sudoers`" is written **before** you need it.

## Step 3 — Access *(chapter [05](../../05_securite.md), SSH and firewall sections)*

1. Key authentication only, `root` forbidden, access restricted to authorized groups,
   idle sessions closed, verbose logging, banner.
2. Firewall: everything closed on input except what is justified, each rule **commented**, PostgreSQL
   only from internal network, configuration persistent on reboot.
3. `fail2ban` active on SSH, with your administration network whitelisted.
4. A scan from outside, **before / after**, in the log.
5. Proof you haven't locked yourself out: a complete VM reboot, followed by a successful
   connection.

## Step 4 — Operations *(chapter [04](../../04_installation_et_services.md))*

1. The application runs under a dedicated account, in a hardened systemd unit, auto-restarted.
2. Logs are in the journal, persistent, size-limited, and `logrotate` manages application logs.
3. Security updates are applied, and you have a written position on automatic updates (for or against, and why).
4. Everything scheduled is documented: what, when, why, where outputs go.

## Step 5 — Tools *(chapter [06](../../06_shellscript/))*

Three scripts, backed by `lib-admin.sh`, installed in `/usr/local/bin`, with `--help` and `--dry-run`:

| Script | Role |
|--------|------|
| `srv-audit` | replays the audit from step 0 and reports what has changed since last time |
| `srv-backup` | backs up files + database + Docker volumes, with rotation and checksum |
| `srv-check` | health check: services, disk, load, certificates, containers; meaningful exit code |

1. Each passes ShellCheck without warnings.
2. Each fails cleanly, with a distinct exit code per cause, and a trace in system log.
3. `srv-backup` interrupted leaves no partial archive.
4. `srv-check` is scheduled every 5 minutes and **alerts you** without you having to check.
5. A **real restoration** was performed from a backup produced by `srv-backup`.

## Step 6 — Docker *(chapter [08](../../08_docker_administration.md))*

1. Log rotation configured globally, no container running as root or with `--privileged`.
2. Memory, CPU and PID limits on each service; explicit restart policy.
3. The stack restarts by itself after a machine reboot (systemd unit).
4. Volumes are backed up by `srv-backup`, and a restore has been tested.
5. Weekly cleanup is scheduled, and you know exactly what it deletes.

## Step 7 — Environment and Reproducibility *(chapters [10](../../10_shell_config.md) and [11](../../11_advanced_config.md))*

1. `/etc/profile.d` contains the `PATH`, `umask` and distinctive root prompt; `/etc/skel` is completed.
2. Your administrator dotfiles deploy in one command on a new machine.
3. Your modified configuration files (`/etc/ssh/sshd_config`, `sudoers.d`, units, `daemon.json`,
   `logrotate.d`) are **versioned** somewhere, with history of your modifications.
4. A colleague can rebuild this server from your documentation alone. Have it verified:
   give your `journal.md` to someone else and watch them try, without helping.

---

## 📄 Deliverable

A folder `/root/dossier-serveur/` (versioned) containing:

1. `audit-initial.md` — the state when you arrived.
2. `journal.md` — what you changed, when, and **why**.
3. `exploitation.md` — procedures: start, stop, deploy, backup, **restore**,
   what to do if disk is full, what to do if the application doesn't respond.
4. `acces.md` — who has access to what, with what right, and how to remove access when someone leaves.
5. `scripts/` — your scripts and their `README.md`.
6. `audit-final.md` — the same audit as step 0, for comparison. This is your proof.

---

## ✅ Success Criteria

- [ ] No connection possible as `root` or by password; each action is traceable to a person.
- [ ] An external scan shows only ports you can justify one by one.
- [ ] `systemctl --failed` is empty, and the application restarts by itself after a complete VM reboot.
- [ ] A backup has been **actually restored**, and the procedure is written.
- [ ] `srv-check` detects a failure you provoke (stop a service, fill the disk) and
      alerts you without you having to check.
- [ ] `--dry-run` exists and works on all your destructive commands.
- [ ] No secret appears in a file readable by all, nor in history, nor in a command line, nor in a Git repository.
- [ ] You can take two weeks vacation: someone else runs the server with your folder.

> [!TIP]
> This last point is the only one that really matters. A server only one human knows how to administer
> is not administered: it's held hostage.
