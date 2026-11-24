# ⚙️ A11 · User Environment Configuration

*Chapters [10 · Shell Configuration](../../10_shell_config.md) and [11 · Advanced Configuration](../../11_advanced_config.md)*

> [!NOTE]
> **Objectives**: stop configuring each *home* by hand. Understand which file is read and when,
> enforce configuration across the entire machine, prepare the skeleton for new accounts, and deploy
> your own administrator configuration across a server fleet.

---

## Exercise 11.1 — Who Reads What, and When

1. What is the default shell on this server? Which shells are installed?
2. Establish the **complete** table: for a login shell, a non-login interactive shell, an `ssh server`,
   an `ssh server 'command'` and a script, **which files are read, in what order**?
   Verify it experimentally with temporary `echo`s in each file, then remove them.
3. Why is a variable defined in `~/.bashrc` not seen by a `cron` task
   *(chapter [04](../../04_installation_et_services.md))*? And by an `ssh server 'command'`?
4. Where do you place: an alias? a `PATH` modification? a variable needed by a service?
   Justify each answer.
5. How do you apply a modification without closing your session — two methods.

---

## Exercise 11.2 — System Configuration

1. What is the difference between `/etc/profile`, `/etc/profile.d/*.sh`, `/etc/bash.bashrc` and `~/.bashrc`?
2. Write `/etc/profile.d/ticketflow.sh` which, **for all users**:
   - adds `/srv/ticketflow/scripts` to the `PATH`;
   - defines `TICKETFLOW_HOME`;
   - sets a `umask` to `027` *(chapter [02](../../02_file_permissions.md))*.
3. Verify that a new user inherits it, and that an existing user does too.
4. Write `/etc/profile.d/prompt-root.sh` which gives a **red and explicit prompt** to root sessions.
   Why is this more than just cosmetics?
5. Add a system banner (`/etc/motd` or `/etc/issue.net`) reminding that the machine is in production
   and monitored. What is the difference between the two files, and which one displays before authentication?
6. Should system configuration prevent a user from overriding it in their *home*?
   Discuss, distinguishing convenience from security constraints.

---

## Exercise 11.3 — The Skeleton for New Accounts

1. What does `/etc/skel` contain? Do these files apply to already-created accounts?
2. Complete it: a `.bashrc` with home aliases, a minimal `.vimrc`, a `.tmux.conf`, a
   `README-serveur.txt`.
3. Create a test account and verify it inherits everything, with correct permissions and owner.
4. Write a script that applies a new version of `.bashrc` **to all existing accounts**,
   saving the old one and without overwriting customization
   *(chapter [06](../../06_shellscript/) — this is a complete scripting exercise)*.
5. Why is a brutal `cp /etc/skel/.bashrc /home/*/` a bad idea? List three problems.

---

## Exercise 11.4 — Administrator Aliases and Functions

Write them in **your** configuration, then decide which ones deserve to go in `/etc/profile.d`.

1. `ll`, `..`, `...`, and securing `rm`, `cp`, `mv`.
2. `ports` — listening ports with associated process.
3. `failed` — failed services.
4. `logsvc <service>` — tail a service's logs.
5. `psg <pattern>` — search for a process.
6. `bigfiles [path]` — the 10 largest files in a path.
7. `backup <file>` — timestamped copy of a file before modification. Why must this be
   a function and not an alias?
8. How do you run the **real** `rm` command bypassing your secure alias?

---

## Exercise 11.5 — `PATH`, History and Convenience

1. Display the `PATH`, one folder per line. Is `root`'s `PATH` the same as yours? Why?
2. Add `/srv/ticketflow/scripts` to the `PATH`, **only if it's not already there**, permanently.
3. Should you add it at the beginning or at the end? What risk in each case?
4. Configure the history: 10,000 entries, no duplicates, **with timestamps**, shared between multiple
   terminals. Why is timestamped history useful after an incident?
5. Make sure `root`'s history is also timestamped and preserved. Is this a good
   security practice, or a risk? Discuss.
6. Find in history the last command containing `systemctl` — with `grep`, then with `CTRL+r`.

---

## Exercise 11.6 — Your Configuration Across the Fleet

1. Create a `dotfiles-admin` repository containing your `.bashrc`, `.vimrc`, `.tmux.conf`, `.ssh/config`
   *(without keys or secrets!)* and your scripts.
2. Replace the files in your *home* with **symbolic links** to the repository.
3. Write `install.sh`: it creates links, **saves** any existing file before replacing it,
   is idempotent, and has a `--dry-run`.
4. Extract from the repository anything machine-specific or confidential: load it from a
   `~/.bashrc.local` if it exists, ignored by Git.
5. Deploy your configuration to a second VM in one command. Time it.
6. Compare three approaches — a repository + script, GNU Stow, a configuration management tool
   (Ansible) — and say which you'd choose for 2 servers, for 20, for 200.
7. *(Bonus)* Install `fzf` and `tmux` with your configuration, and write a function that chooses a
   systemd service with `fzf` to restart it.

> [!WARNING]
> On a production server, each tool added is one more dependency to maintain and one more attack surface. 
> A server's configuration is not a workstation's: justify each addition.

---

## ✅ Verification

- A new account created today automatically inherits the `PATH`, `umask`, aliases and files
  from `/etc/skel`, without manual intervention.
- A root session is visually impossible to confuse with a normal session.
- `install.sh` rebuilds your administrator environment on a new VM, without overwriting anything
  without a backup.
- No secrets or private keys ended up in the dotfiles repository.
- You can tell, for any configuration line, **which file** it belongs in
  and **why**.
