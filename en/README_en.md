# 🐧 Linux Training — System Administration & Shell Scripting

> **BStorm** training material: the basics of Linux administration, then shell script writing, from the first `pwd` to advanced configuration of your work environment.

---

## 📚 Curriculum

| # | Chapter | Content |
|---|---------|---------|
| 01 | [Terminal Basics](01_base.md) | Navigation, files, redirections, editors, processes |
| 02 | [Permissions](02_file_permissions.md) | POSIX, `chmod`, `chown`, special permissions |
| 02.1 | [Users and Groups](02.1_user_management.md) | `useradd`, groups, passwords, `su` / `sudo` |
| 03 | [Essential Commands](03_commandes_essentielles.md) | `tree`, `grep`, `sed`, `awk`, `cut` / `sort` / `uniq`, `find`, `xargs`, `du`, `df`, `tar` |
| 04 | [Packages, Services and Cron](04_installation_et_services.md) | `apt` / `dnf`, `systemctl`, custom services, `journalctl`, `cron` |
| 05 | [Security](05_securite.md) | ACL, `sudoers`, firewall, SSH, AppArmor / SELinux |
| 06 | [**Shell Scripting**](#-06--shell-scripting) | *see details below* |
| 07 | [Docker — Developers](07_docker.md) | Images, containers, volumes, `Dockerfile`, `docker compose` |
| 08 | [Docker — System Administrators](08_docker_administration.md) | Daemon, networks, logs, resources, hardening, backups |
| 09 | [tmux](09_tmux.md) | Sessions, windows, panes, remote work |
| 10 | [Shell Configuration](10_shell_config.md) | `.bashrc`, `.zshrc`, `PATH`, prompt, aliases, dotfiles |
| 11 | [Advanced Configuration](11_advanced_config.md) | Oh My Zsh, Powerlevel10k, Starship, Oh My Tmux, modern tools |

### 🐚 06 · Shell Scripting

| # | Chapter | Content |
|---|---------|---------|
| 06.1 | [Basics](06_shellscript/01_Base/base.md) | Shebang, variables, arrays, `read` |
| 06.2 | [Conditions and Logical Operators](06_shellscript/02_operateur_logique/exemple.md) | `if` / `elif` / `else`, tests, `&&`, `\|\|`, `!` |
| 06.3 | [Loops](06_shellscript/03_boucles/exemple.md) | `for`, `while`, `until`, file reading |
| 06.4 | [Error Handling](06_shellscript/04_gestion_des_erreurs/gestion_des_erreurs.md) | `$?`, `set -euo pipefail`, `trap` |
| 06.5 | [Arguments](06_shellscript/05_Arguments/args.md) | `$1`, `$@`, `getopts` |
| 06.6 | [Functions](06_shellscript/06_fonctions/fonctions.md) | Declaration, arguments, `local`, `source` |

---

## ✏️ Exercises

| # | Exercise | Covers |
|---|----------|--------|
| 3 | [Exercise 3](exercices/Exercice3.md) | File tree, files, redirections — ch. 01 |
| 4 | [Exercise 4](exercices/Exercice4.md) | Users and groups — ch. 02.1 |
| 5 | [Exercise 5](exercices/Exercice5.md) | `grep`, `find`, `tar` — ch. 03 |
| 6 | [Exercise 6](exercices/Exercice6.md) | Creating a systemd service — ch. 04 |
| 7 | [Exercise 7](exercices/Exercice7.md) | `cron` tasks — ch. 04 |
| 8 | [Exercise 8](exercices/Exercice8.md) | `sudoers` and rights delegation — ch. 05 |
| 9 | [Exercise 9](exercices/Exercice9.md) | SSH and firewall — ch. 05 |
| 10 | [Exercise 10](exercices/Exercice10/) | Scripts: conditions and loops — ch. 06 |
| 11 | [Exercise 11](exercices/Exercices11/) | Scripts: arguments and user creation — ch. 06 |
| 12 | [Exercise 12](exercices/Exercices12/) | Scripts: functions and shared library — ch. 06 |

### 🧭 Two Thematic Paths

Two separate exercise sets, **without solutions**, each built on a unique storyline and covering only chapters relevant to its audience:

| Path | For Whom | Content |
|------|----------|---------|
| 👩‍💻 [**Developer**](exercices/dev/) | A dev who works on Linux daily | Terminal, code and log searching, essential admin for devs, scripting, Docker, tmux, work environment — around the `ticketflow` application |
| 🛠️ [**System Administrator**](exercices/admin/) | An admin taking over a server | Startup, permissions, accounts, toolbox, services and cron, ACL and `sudo`, SSH and firewall, AppArmor / SELinux, admin scripts, Docker in production, user environment — around the `srv-tickets` server |

They complement each other: the developer path builds the application, the administrator path runs the server hosting it.

---

## 🗂️ Repository Organization

```text
.
├── 01_base.md … 11_advanced_config.md   # course chapters
├── 06_shellscript/                      # scripting module, one folder per topic
│   └── NN_theme/
│       ├── theme.md                     # the course
│       └── exemple*.sh                  # example scripts, executable
├── 07_docker/                           # example Docker projects, buildable
│   └── NN_exemple/
│       ├── Dockerfile                   # commented step by step
│       └── …                            # application sources
├── exercices/                           # exercises and solutions
│   ├── dev/                             # developer path, without solutions
│   └── admin/                           # system administrator path, without solutions
├── exemple/                             # working files for exercises
└── docker-compose.yml                   # demonstration PostgreSQL database
```

---

## 🚀 Environment Setup

All exercises require a Linux machine where you can break things without consequence. Three options:

| Option | For Whom |
|--------|----------|
| A **VM** (VirtualBox, UTM, Hyper-V) with Debian 12 | Closest to a real server — **recommended** |
| A **Docker container** Debian | Quickest to set up |
| **WSL 2** (Windows) | Comfortable, but `systemd` and services work differently |

A disposable sandbox in one command:

```bash
docker run -it --rm --name lab debian:12 bash
```

```bash
# In the container, to have course tools
apt update && apt install -y sudo vim nano tree htop cron tmux
```

> [!NOTE]
> A container doesn't run `systemd` by default: the `systemctl` manipulations in [chapter 04](04_installation_et_services.md) require a real VM.
>
> The `docker-compose.yml` in the repository is **not** this work environment: it only runs a PostgreSQL database used for demonstration — it's explained in [chapter 07](07_docker.md#docker-compose).

> [!WARNING]
> **Never do administration exercises on your main machine.** Chapters 02, 02.1, 04 and 05 manipulate users, services, firewall and SSH configuration — things that can lock you out of your own system.

---

## 🧭 Documentation Conventions

| Element | Meaning |
|---------|---------|
| <code>```bash</code> | A command to type in the terminal |
| <code>```text</code> | Command output or file excerpt |
| `> [!TIP]` | A tip or best practice |
| `> [!IMPORTANT]` | A key point to understand what follows |
| `> [!WARNING]` | A common pitfall |
| `> [!CAUTION]` | A destructive or dangerous operation |

Each chapter starts with its **objectives** and ends with a **summary** of commands covered.

---

## 🔗 Resources

- [`man` online](https://man7.org/linux/man-pages/) — the reference manual
- [ExplainShell](https://explainshell.com/) — breaks down any command line
- [ShellCheck](https://www.shellcheck.net/) — static analysis of shell scripts
- [tldr pages](https://tldr.sh/) — examples rather than manuals
- [Google Shell Style Guide](https://google.github.io/styleguide/shellguide.html) — writing conventions

---

**➡️ Start the training: [01 · Terminal Basics](01_base.md)**
