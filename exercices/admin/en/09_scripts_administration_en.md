# 📜 A9 · Administration Scripts

*Chapter [06 · Shell scripting](../../06_shellscript/) — all six subsections*

> [!NOTE]
> **Objectives**: automate what you do twice. Audit accounts, backup with rotation,
> monitor the machine, create accounts in bulk — with scripts that fail cleanly, explain themselves
> and leave nothing behind.

> [!IMPORTANT]
> Rules valid for all scripts in this chapter:
> - shebang, executable, in `/usr/local/bin` or `/srv/ticketflow/scripts`;
> - `set -euo pipefail` at the head;
> - all variables quoted;
> - an administration script **never destroys anything** without a `--dry-run` available and confirmation;
> - it passes [ShellCheck](https://www.shellcheck.net/) without warnings.

---

## 1 · The Basics *(chapter [06.1](../../06_shellscript/01_Base/base.md))*

### Exercise 9.1 — `sysinfo.sh`

The script you'll run when arriving on an unknown machine.

1. Display, cleanly presented: machine name, distribution, kernel, *uptime*, average load,
   free memory, `/` occupancy, number of logged-in users, number of failed services.
2. Each value comes from a command whose output you capture in a variable.
3. Add a header with the date and server name, and a separator.
4. Have it write its report **to screen and** to `/var/log/ticketflow/sysinfo-<date>.txt`.
5. Store alert thresholds (disk, load) in variables at the top of the file, so they can be
   modified without reading the code.

### Exercise 9.2 — Arrays

1. Create a `SERVICES_CRITIQUES` array containing `ssh`, `nginx`, `postgresql` and `ticketflow-api`.
2. Display the number of elements, then each element.
3. Fill an array with the list of human users on the machine
   *(chapter [03](../../03_commandes_essentielles.md))* and display it.
4. Add a service to the array, remove one, re-display.

---

## 2 · Conditions *(chapter [06.2](../../06_shellscript/02_operateur_logique/exemple.md))*

### Exercise 9.3 — `check-server.sh`

1. The script must refuse to run if it is **not** run as root. How do you test this?
2. Verify that `/srv/ticketflow` exists and is a folder.
3. Verify that the configuration file exists and is **readable**.
4. Verify that the `pg_dump` command is available.
5. Verify that `/` occupancy is below the defined threshold; otherwise, display an alert.
6. Verify that a mandatory environment variable is defined **and not empty**.
7. Write a test that compares two numbers (free space and threshold) and a test that compares two strings
   (the environment). Why are the operators different?
8. Write, in one line, "create the backup folder if it doesn't exist", then
   "run the backup, and if it fails, write an alert and exit with error".

---

## 3 · Loops *(chapter [06.3](../../06_shellscript/03_boucles/exemple.md))*

### Exercise 9.4 — Iterating

1. Loop over `SERVICES_CRITIQUES` and display, for each, `✔` or `✘` depending on whether it's active.
2. Loop over users' homes and display, for each, its size
   *(chapter [03](../../03_commandes_essentielles.md))*.
3. Loop over files in `/etc/sudoers.d` and display the name of each with its permissions.
4. Loop over a list of IP addresses and test if each responds to ping, with a final summary.

### Exercise 9.5 — Read a File, Wait

1. Read `/etc/passwd` **line by line** and display, for human accounts only, the name and shell.
2. Read a `utilisateurs.csv` file (`name;group;comment`) and create the corresponding accounts.
   Skip empty lines and comments (`continue`), stop if a line is malformed (`break`).
3. Write a loop that waits for a service to be **active**, with a **maximum delay** and a
   failure message beyond that. Why is unbounded waiting unacceptable in an operations script?
4. Rewrite this wait with `until`.

### Exercise 9.6 — `backup.sh` and Rotation

1. Write a script that archives `/srv/ticketflow` and `/etc` to
   `/srv/ticketflow/backups/backup-YYYYMMDD-HHMMSS.tar.gz`, **preserving permissions and ACL**.
2. Add a PostgreSQL database dump, **never letting the password appear in the command line**
   *(hint: a sourced environment file, or `~/.pgpass`)*.
3. Keep only the **last 7** backups, delete the others.
4. Write the size and duration of each backup to a log file.
5. Produce a checksum for each archive, and a verification command.

---

## 4 · Error Handling *(chapter [06.4](../../06_shellscript/04_gestion_des_erreurs/gestion_des_erreurs.md))*

### Exercise 9.7 — Fail Cleanly

1. Add `set -euo pipefail` to `backup.sh` and observe what breaks — fix what needs fixing.
2. Temporarily disable automatic exit for **one** command whose failure is acceptable.
3. Write `erreur()`: message prefixed `[ERROR]` on **error output**, non-zero exit code.
4. Make the script exit with a **different** code depending on the cause: 1 (arguments),
   2 (missing prerequisites), 3 (backup failed). Why is this useful when the script is called by
   cron or a systemd *timer*?
5. An interrupted backup must **never** leave an incomplete archive: write to a temporary file
   renamed only on success.
6. With `trap`, guarantee the deletion of the temporary folder at script end, on error,
   and on `CTRL+C`. Verify all three cases.
7. Make sure an error sends a line to the system log *(hint: `logger`)*.
   Where do you find it afterwards?

---

## 5 · Arguments *(chapter [06.5](../../06_shellscript/05_Arguments/args.md))*

### Exercise 9.8 — `backup.sh` as a Real Command

```text
Usage: backup.sh -d <destination> [-k <number>] [-t full|db|conf] [-n] [-v] [-h]
  -d DEST   destination folder                            (required)
  -k N      number of backups to keep                      (default: 7)
  -t TYPE   what to backup: full | db | conf              (default: full)
  -n        dry-run: display what would be done
  -v        verbose mode
  -h        help
```

1. Implement parsing with `getopts`.
2. `-h` displays help and exits with code `0`; absence of `-d` is an error.
3. An unknown `-t` value is refused with an explicit message.
4. `-k` must be a **positive integer**: validate it.
5. In `-n` mode, no writes or deletions occur — each action is announced, prefixed `[dry-run]`.
6. Test: `-n -t db`, `-k abc`, `-t plop`, no options, `-h`.

### Exercise 9.9 — `useradd-batch.sh`

1. A script that receives a CSV file as argument and creates the accounts it describes.
2. It verifies **before** creating: the account doesn't already exist, the group exists, the name is valid.
3. It generates a random password per account, forces change on first login,
   and writes credentials to a file readable **by root alone**.
4. It produces a final report: created, skipped, in error.
5. It has a `--dry-run`.

---

## 6 · Functions *(chapter [06.6](../../06_shellscript/06_fonctions/fonctions.md))*

### Exercise 9.10 — Break Down and Share

1. Break down `backup.sh` into functions: `usage`, `log`, `erreur`, `verifier_prerequis`, `sauver_fichiers`,
   `sauver_base`, `rotation`, `nettoyage`.
2. The script body is now just a sequence of calls.
3. `log()` timestamps its messages and writes to both screen and log file.
4. `verifier_prerequis()` returns a **code** that the script uses.
5. Write a function that **returns a value**: `espace_libre()` returns the occupancy percentage
   of the partition passed as argument.
6. Use `local` everywhere; show, with an example, what breaks when you forget it.

### Exercise 9.11 — `lib-admin.sh`

1. Create `/usr/local/lib/lib-admin.sh` with `log`, `erreur`, `confirmer`, `require_root`, `require_cmd`,
   `service_actif`, `alerte` (send a line to system log).
2. The file must **execute nothing** when sourced.
3. Source it from `backup.sh`, `check-server.sh` and `useradd-batch.sh`, and remove all duplicated code.
4. Make the import work no matter which folder the script is run from.
5. Add color — automatically disabled when output is not a terminal.
   Why is this essential for a script called by cron?

### Exercise 9.12 — Production Deployment

1. Install `backup.sh` in `/usr/local/bin` with the correct owner and permissions
   *(chapter [02](../../02_file_permissions.md))*.
2. Schedule it daily — in cron **or** in a systemd *timer* *(chapter [04](../../04_installation_et_services.md))*.
3. Verify the next day that the task ran: where do you look?
4. Provoke a failure (nonexistent destination) and verify that you are **notified** without having to
   go check.
5. Write the **restoration** procedure in `journal.md`, and actually test it.
   A backup you've never restored is not a backup.

---

## ✅ Verification

- `/usr/local/bin` contains `sysinfo.sh`, `check-server.sh`, `backup.sh`, `useradd-batch.sh`, and
  `/usr/local/lib/lib-admin.sh` holds all the shared code.
- `shellcheck` returns nothing for any of them.
- `backup.sh -h` explains itself; `backup.sh` with no argument fails with a non-zero code.
- `backup.sh -n` doesn't modify anything at all.
- A `CTRL+C` in the middle of a backup leaves neither partial archive nor temporary folder.
- The backup runs on its own, fails loudly, and you have **restored** at least once.
