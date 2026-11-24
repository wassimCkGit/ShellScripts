# 👥 A3 · Users, Groups and Accounts

*Chapter [02.1 · User and Group Management](../../02.1_user_management.md)*

> [!NOTE]
> **Objectives**: create and remove accounts properly, understand the files that describe them, separate human and service accounts, manage passwords and expirations, and stop working as `root`.

> [!CAUTION]
> Snapshot the VM **before** you start. We're locking accounts and touching `sudo`: keep a privileged session open permanently that you don't close.

---

## Exercise 3.1 — The Current State

1. List all accounts on the machine. How many are there?
2. Distinguish **human** accounts from **system** accounts: what criterion do you use? Produce a list of human accounts only.
3. What does each of the seven fields in a `/etc/passwd` line contain?
4. Why isn't the password there anymore, and where is it? Who can read this file, and with what rights?
5. List all groups, then those your user belongs to.
6. Which accounts have a valid login shell? Which have `/usr/sbin/nologin`, and why?

---

## Exercise 3.2 — Creating Team Accounts

The team: `alice` (admin), `bob` and `carine` (developers), plus two service accounts.

1. Create the `ops` group and the `dev` group.
2. Create `alice`, `bob` and `carine` **with** their *home*, shell `/bin/bash` and a comment (full name).
3. Add `alice` to `ops`, `bob` and `carine` to `dev` — **without** removing them from their primary group.
4. Add `alice` to the group that grants `sudo` rights on Debian. Verify with her.
5. Create the `deploy` service account: **no interactive *home* needed**, shell `/usr/sbin/nologin`, group `ops`. Why shouldn't a service account have a login shell?
6. Create the `backup` account, without a usable password, intended to be used **only** via SSH key.
7. Display, for each created account, its UID, GID and secondary groups.

---

## Exercise 3.3 — Passwords and Expiration

1. Set a password for `bob`.
2. Force `bob` to change his password **at his next login**.
3. Display `bob`'s password expiration policy (last change, expiration, warning).
4. Enforce: password valid 90 days, warning 7 days before, minimum 1 day between changes.
5. Set an **account expiration date** for `carine` to the last day of next month (an intern, we won't forget to close the account).
6. **Lock** `bob`'s account, try to log in, then unlock it. What's the difference between locking the password and disabling the shell?
7. Where is the **default** password policy for new accounts configured?

---

## Exercise 3.4 — Modifying and Deleting

1. Rename `carine` to `carine.dupont`, also renaming her *home*.
2. Change `bob`'s default shell to `/bin/sh`, then back to `/bin/bash`.
3. Move `bob`'s *home* to `/srv/home/bob` preserving his files.
4. Remove `bob` from the `dev` group without touching his other groups.
5. Delete the `carine.dupont` account **with** her *home* and mail spool.
6. After deletion, files owned by her subsist elsewhere on disk. How do you find them? What happens if you create a new account that gets the same UID?

---

## Exercise 3.5 — New Account Skeleton

1. What does `/etc/skel` contain on your machine?
2. Add a `README-serveur.txt` file reminding internal rules, and a `.bashrc` with a custom alias.
3. Create a new test account and verify it inherits these files.
4. Do **already existing** accounts inherit them? How do you apply the change to them?
5. Where are the default *home*, default shell and UID range for new accounts configured?

---

## Exercise 3.6 — Changing Identity Properly

1. Execute **one** command as root, without opening a root session.
2. Open a complete root session, with a clean environment. What's the difference from `su root` without a dash?
3. Open a session as `deploy` even though he has no login shell. In what exploitation case do we need this?
4. Verify "who you really are" in a `sudo -i` session: `whoami`, `logname`, `id`. Which traces the original user?
5. Find in `/var/log/auth.log` all privilege escalations from today.
6. Why do we leave `root` without a usable password on a well-managed server, and what replaces it? *(Continued in [A6](06_acl_et_sudo.md).)*

---

## ✅ Verification

- `alice`, `bob`, `deploy` and `backup` exist with correct shell, groups and expiration policy.
- No service account has a login shell.
- You can produce, in one line, the list of human accounts on the machine.
- You can answer "who did that, and when?" by reading `/var/log/auth.log`.
- Your documentation lists each created account: who it belongs to, why it exists, and when it must be closed.
