# 🧱 A8 · AppArmor / SELinux

*Chapter [05 · Security](../../05_securite.md) — **AppArmor / SELinux** section*

> [!NOTE]
> **Objectives**: understand what mandatory access control (MAC) adds on top of POSIX permissions,
> know in what mode the machine is running, read a denial in the logs, adjust a profile or context —
> and especially **don't disable the protection at the first difficulty**.

> [!IMPORTANT]
> Debian / Ubuntu use **AppArmor**, RHEL / Fedora / Rocky use **SELinux**. Do the section that
> corresponds to your VM, then **read** the other: you will encounter both in your career.

---

## Exercise 8.1 — The Principle

1. One sentence each: what is **DAC** (POSIX permissions from chapter 02) and what is **MAC**?
2. A process runs as `root`. DAC allows it to read everything. What else can still prevent it?
3. Give a concrete example of a compromise that a MAC blocks but that a correct `chmod` would not have blocked.
4. On your machine: is AppArmor present? active? And SELinux? What commands tell you?

---

## AppArmor Section *(Debian, Ubuntu, SUSE)*

### Exercise 8.2 — State and Modes

1. Display AppArmor's state: how many profiles loaded, how many in `enforce`, how many in `complain`?
2. What is the difference between `enforce`, `complain` and `disabled`? In what order do you use them
   when building a profile?
3. List the loaded profiles and identify the one for your web server or database client.
4. Set an existing profile to `complain`, verify the change, then set it back to `enforce`.

### Exercise 8.3 — Reading a Denial

1. Provoke a denial: have a confined program read a file that its profile doesn't give it access to
   (for example, have nginx serve a folder outside its profile).
2. Find the denial in the logs. What keyword are you searching for?
   *(Chapter [03](../../03_commandes_essentielles.md) for filtering.)*
3. Decode the line: which program, which operation, which path, which profile?
4. A user tells you "it works as root but not from the service" even though POSIX permissions are
   correct. What is your first hypothesis, and how do you verify it in thirty seconds?

### Exercise 8.4 — Write and Adjust a Profile

1. Write `/usr/local/bin/backup.sh`, a script that reads `/srv/ticketflow` and writes to
   `/srv/ticketflow/backups`.
2. Create a profile for this script: read-only on sources, write on the destination,
   execution of the binaries it needs, and **nothing else**.
3. Load it in `complain` mode, run the script on **all** its use cases (including error cases),
   then collect what was missing.
4. Switch to `enforce` and verify that the script still works.
5. Verify that it **cannot** read `/etc/shadow` or write anywhere but its destination folder.
6. Modify the profile of a package (nginx, for example) to authorize a custom docroot —
   **without** editing the file provided by the package. Where do we write this override?
7. Verify the syntax of a profile without loading it, then reload it, then reload the entire set.

---

## SELinux Section *(RHEL, Fedora, Rocky)*

### Exercise 8.5 — State and Modes

1. Display the current mode and the mode configured at startup. Where is the latter set?
2. Temporarily switch to `permissive`, verify, return to `enforcing`.
   Why do we **never** do a permanent `SELINUX=disabled` to "fix" a problem?
3. What is the difference between `permissive` and `disabled` from a diagnostic point of view?

### Exercise 8.6 — Contexts

1. Display the security context of `/var/www/html`, of an `httpd` process and of your own shell.
2. What do the four fields of a context mean?
3. Move a file from your *home* to `/var/www/html` with `mv`, then with `cp`.
   Compare the contexts obtained and explain the classic pitfall.
4. Fix the context of the moved file — first punctually, then **durably** by declaring
   the rule for the path and reapplying it.
5. Have your web server serve a docroot in `/srv/web`: which type must you declare, and how?

### Exercise 8.7 — Booleans, Ports and Diagnostics

1. List the active booleans concerning your web server. What does `httpd_can_network_connect` do?
2. Activate one of them **permanently**, and explain what the persistence option changes.
3. Have `sshd` listen on port 2222: which port type must you declare, and with what command?
   *(The link with exercise [7.3](07_ssh_et_pare_feu.md) is direct.)*
4. Provoke a denial, find it in the audit logs, and have the cause explained.
5. Generate the policy module that would authorize this denial — **and read it before installing it**.
   Why do we never install it without reading it?

---

## Exercise 8.8 — The Diagnostic Method

A service refuses to access a file. Write, in `journal.md`, your **decision tree** starting
from the question "is this a permissions problem?":

1. What do you verify first (DAC: owner, permissions, ACL, traversable path)?
2. How do you know, in one command, if the MAC is involved?
3. How do you confirm it **without** disabling protection?
4. How do you fix it properly — and why is the "fix" of "I'll put everything at 777" or
   "I'll disable SELinux" an admission of failure?
5. What do you write in the log once the problem is resolved?

---

## ✅ Verification

- You can tell, on any machine, if a MAC is running and in what mode.
- Your backup script runs under a restricted profile / context, **and** you have proven that it
  cannot read `/etc/shadow`.
- You have read and decoded at least one denial in the logs, and corrected the cause rather than disabled protection.
- Your decision tree from exercise 8.8 fits on one page and is readable by someone else.
