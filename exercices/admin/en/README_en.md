# 🛠️ System Administrator Track — Exercises

> A series of **solution-free** exercises, administration-focused: take over a server, manage its
> permissions and accounts, work with its logs, manage its packages, services and scheduled tasks,
> harden it, build it with scripts, run Docker properly and standardize the environment
> for all its users.
>
> The track covers chapters **01 to 06**, **08**, **09**, **10** and **11**. Chapter
> **07 · Docker for Developers** is covered here only from an operations angle (chapter 08):
> writing `Dockerfile`s is covered in the [developer track](../dev/) — the counterpart to this one,
> on the application side.

---

## 🧵 The Red Thread — `srv-tickets`

You are taking over administration of **`srv-tickets`**, the Debian 12 VM exposed on the Internet that hosts
the `ticketflow` application (an API + PostgreSQL database) written by the development team.

What you're left with:

| Element | State When You Arrive |
|---------|----------------------|
| Accounts | everyone connects as `root`, shared password |
| SSH | port 22, password authentication, `PermitRootLogin yes` |
| Firewall | non-existent |
| Backups | "there's a `.tar.gz` somewhere in `/root`" |
| Supervision | none: you learn about failures from users |
| Docker | installed "hastily", logs fill the disk |

**Your mission, through the exercises: make it a tenable server.** Each chapter picks up the server
where the previous one left it, but each file is playable independently.

---

## 📋 The Exercises

| # | File | Covers |
|---|------|--------|
| A1 | [Getting Started with the Server](01_prise_en_main.md) | ch. [01](../../01_base.md) and [09](../../09_tmux.md) — file system layout, logs, processes, redirections, tmux |
| A2 | [Permissions and Ownership](02_permissions.md) | ch. [02](../../02_file_permissions.md) — octal, symbolic, SUID / SGID / sticky, `chown`, `umask` |
| A3 | [Users, Groups and Accounts](03_utilisateurs_et_groupes.md) | ch. [02.1](../../02.1_user_management.md) — account lifecycle, `/etc/passwd`, `/etc/shadow`, service accounts |
| A4 | [The Toolkit](04_boite_a_outils.md) | ch. [03](../../03_commandes_essentielles.md) — `grep`, `sed`, `awk`, `find`, `xargs`, `du` / `df`, `tar` |
| A5 | [Packages, Services and Cron](05_paquets_services_cron.md) | ch. [04](../../04_installation_et_services.md) — `apt`, `systemctl`, custom unit, `journalctl`, `cron` |
| A6 | [ACL and `sudo` Delegation](06_acl_et_sudo.md) | ch. [05](../../05_securite.md) — ACL, mask, inheritance, `sudoers`, `sudoedit`, audit |
| A7 | [SSH and Firewall](07_ssh_et_pare_feu.md) | ch. [05](../../05_securite.md) — keys, `sshd` hardening, `Match`, tunnels, `ufw` / `nftables`, `fail2ban` |
| A8 | [AppArmor / SELinux](08_mac_apparmor_selinux.md) | ch. [05](../../05_securite.md) — DAC vs MAC, modes, profiles, contexts, booleans, diagnosis |
| A9 | [Administration Scripts](09_scripts_administration.md) | ch. [06](../../06_shellscript/) — audit, backup with rotation, monitoring, `getopts`, `trap`, library |
| A10 | [Docker for Administrators](10_docker_administration.md) | ch. [08](../../08_docker_administration.md) — daemon, volumes, networks, logs, resources, hardening, backups |
| A11 | [User Environment Configuration](11_environnement_utilisateurs.md) | ch. [10](../../10_shell_config.md) and [11](../../11_advanced_config.md) — `/etc/profile.d`, `/etc/skel`, multi-server dotfiles |
| A12 | [Final Project: Secure and Operate `srv-tickets`](12_projet_final.md) | everything above |

> [!IMPORTANT]
> These exercises **intentionally have no solutions**. An administrator doesn't work from memory:
> they work with `man`, the distribution's documentation and a test VM. Each exercise ends
> with **verification criteria** — if you meet them, your solution works.

---

## 🚀 Environment

> [!CAUTION]
> **Don't do any of this on a machine you need.** This track cuts SSH access,
> closes ports, locks accounts and modifies `sudoers`: three-quarters of the exercises can lock you out.
> That's precisely the point — but on a disposable VM with a snapshot.

| Element | Recommendation |
|---------|----------------|
| Machine | A **VM** Debian 12 (VirtualBox, UTM, Proxmox, Hyper-V) — `systemd`, SSH and firewall must be real |
| Snapshot | Take a **snapshot** before each chapter. You'll use it. |
| Access | Two terminals open permanently: one to work, **a second already connected** that you never close |

> [!WARNING]
> **The rule that runs through the entire track**: before reloading `sshd`, `sudoers` or the firewall, keep a
> privileged session **already open** and test the new configuration **from a third connection**.
> As long as the test hasn't passed, don't close the rescue session.

---

## ✅ How to Work

1. Read the entire statement, **then** check the command in `man` before running it as root.
2. Systematically save a configuration file before modifying it — and keep the `.bak`
   until the service restarts without error.
3. Verify syntax before applying: `visudo -c`, `sshd -t`, `nft -c -f`, `dockerd --validate`.
4. Take notes on what you do. Every other chapter will ask you to report the server's state:
   keep a `journal.md` file as you go — this is the professional reflex.
5. Version your configuration files (`/etc` in a Git repository, or at minimum your own fragments).
