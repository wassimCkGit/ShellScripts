# 🖥️ A1 · Server Orientation

*Chapters [01 · Terminal Basics](../../01_base.md) and [09 · tmux](../../09_tmux.md)*

> [!NOTE]
> **Objectives**: know where you've landed. Recognize a Linux system's directory structure, find configuration files and logs, read what's running, redirect and archive your findings — and work in a session that survives an SSH interruption.

---

## Exercise 1.1 — The Lay of the Land

Connect to `srv-tickets` and answer, **with commands to back you up**, the questions you'll be asked in the meeting:

1. What distribution and what exact version?
2. What kernel version? What architecture?
3. What is the machine's hostname? How long has it been running?
4. How many processors, how much RAM, how much is left free?
5. What partitions are mounted, and which one is the fullest?
6. What is the server's IP address, and on which interface?
7. Who is logged in to the machine right now? Who has logged in recently?
8. Record all these answers in `/root/journal.md` — this is the beginning of your documentation.

---

## Exercise 1.2 — System Directory Structure

Without modifying anything, answer in one sentence for each:

1. What do you find in `/etc`, `/var`, `/opt`, `/srv`, `/usr/local/bin` and `/tmp`?
2. What's the difference between `/bin` and `/usr/local/bin`? Where do you put **your** scripts?
3. Where are the system logs? What is the largest file in `/var/log`?
4. Where are the systemd units provided by packages stored? And **yours**?
5. Do `/proc` and `/sys` take up space on disk? Why?
6. List the contents of `/etc` in detailed format, sorted by **modification date** — what was last touched on this server?

---

## Exercise 1.3 — Reading Logs

1. Display the last 50 lines of `/var/log/syslog`.
2. Follow `/var/log/auth.log` **in real time** and, from another terminal, open a session: what do you see?
3. Display `/var/log/auth.log` page by page, being able to scroll back.
4. How many lines does `/var/log/auth.log` contain? What is its size on disk?
5. Display the first 100 lines of the file, then lines 200 to 220.

---

## Exercise 1.4 — Creating and Filling Configuration Files

1. Create the administrator's working directory structure in **one command**.
2. With a **heredoc**, write a configuration file.
3. Add a line with today's date to the end of the file, **without overwriting it**.
4. Copy this file as `.bak` — develop the habit, it will serve you in all following chapters.
5. Create a symbolic link and verify it.

---

## Exercise 1.5 — Redirections: Keeping a Record

1. Launch an inventory of installed packages and write it to a file.
2. Launch a command that fails and send **only the error** to a log file.
3. Relaunch it sending **both streams** to the same file, **without overwriting** its contents.
4. Launch a long command throwing away its entire output completely.
5. Write output to both the screen **and** a file, in one command.
6. Explain the difference between output redirection orders.

---

## Exercise 1.6 — Processes

1. Display all processes on the machine, then only those of the `root` user.
2. Display the 5 processes consuming the most **memory**, then the most **CPU**.
3. Find the PID of the SSH server, then display the process tree.
4. Launch a process in the background, find it, stop it cleanly.
5. What is the difference between the default signal and `kill -9`?
6. What process is listening on port 22? And on port 5432?
7. Give the **four** commands you launch within a minute to diagnose a "slow server".

---

## Exercise 1.7 — Working in a Session That Survives *(chapter [09](../../09_tmux.md))*

1. Install tmux and open a session named `admin`.
2. Detach, verify it's still running, rejoin it.
3. Split the screen into three panes: one shell, one `journalctl -f`, one `htop`.
4. Put the log pane in fullscreen, then return to normal layout.
5. Create a second window named `backup`, switch between the two, rename the first `work`.
6. Launch a long update in one pane, **brutally cut your SSH connection**, reconnect and find the update.
7. Scroll up in a pane's history and search for a word.
8. Write in your documentation the list of shortcuts you'll use every day.

> [!TIP]
> On a remote server, `tmux` (or `screen`) is not a comfort: it's what prevents a network interruption from breaking an `apt upgrade` or database migration in the middle.

---

## ✅ Verification

- Documentation contains the complete state assessment.
- Working directories exist with configuration and backup files.
- You can find without hesitation: a service's configuration, its logs, its systemd unit, its PID and the port it listens on.
- You can lose your SSH connection without losing your current work.
