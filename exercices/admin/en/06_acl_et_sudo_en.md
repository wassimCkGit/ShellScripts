# 🎛️ A6 · ACL and `sudo` Delegation

*Chapter [05 · Security](../../05_securite.md) — **ACL** and **sudo / sudoers** sections*

> [!NOTE]
> **Objectives**: go beyond the limits of POSIX permissions with ACL (mask, inheritance, backup), and
> delegate precise administrative rights without distributing `ALL=(ALL) ALL` to the entire team.

> [!CAUTION]
> `sudoers` is the file that can lock you out the fastest. **Snapshot before**, keep a root session
> open permanently in a second terminal, and never edit without `visudo`.

---

## Part 1 — ACL

### Exercise 6.1 — Prerequisites

1. Verify that the `acl` package is installed.
2. Verify that the file system hosting `/srv` is mounted with ACL support.
   How would you make it permanent if it's not the case?
3. Create `/srv/projet` and identify, in an `ls -l`, the sign that indicates a file has ACL.

### Exercise 6.2 — Read and Set

1. Display the complete ACL of `/srv/projet`. What does each line mean when no extended ACL is set?
2. Give `bob` the right to **read and write** in `/srv/projet`, without adding him to the owner group.
3. Give the `dev` group **read-only** rights to the same folder.
4. Re-read the ACL: what is the value of the `mask`? What happens if you set it to `r--`?
5. Remove `bob`'s entry, then delete **all** extended ACL entries from the folder.

### Exercise 6.3 — Inheritance

1. Set a **default** ACL on `/srv/projet`: every new file should be readable and modifiable
   by the `dev` group.
2. Create a file with the `alice` account and another with `bob`: verify that both have
   the inherited ACL.
3. Add the **SGID** to the folder so the owner group is preserved, and explain the
   difference between what SGID does and what default ACL does.
4. Does a file **copied** to the folder inherit the ACL? What about a file **moved**?
   Test both cases and explain.

### Exercise 6.4 — The Complete Practical Case

Requirements for `/srv/projet`:

| Who | Expected Right |
|-----|-----------------|
| `alice` (ops) | everything, including administering the folder |
| `dev` group | read/write on files, file creation |
| `deploy` | read-only |
| everyone else | nothing, not even list |

1. Write the sequence of commands that implements this, with both **access** ACL and **default** ACL.
2. Verify each line of the table by logging in with the relevant accounts.
3. **Save** the folder's ACL to a file, delete them all, then **restore** them
   from the backup.
4. Copy the directory tree to `/srv/projet-copie` **while preserving ACL**: which `cp` option (or
   which other command) is needed?
5. Why should ACL appear in your backup procedure? What happens if you
   restore a `tar` made without the proper option?

---

## Part 2 — `sudo`

### Exercise 6.5 — The Current State

1. Display what **you** have the right to do with `sudo`.
2. Who is a member of the `sudo` group on this machine?
3. Where are the rules defined? What is the read order, and why must the line including fragments
   be **at the end**?
4. Produce the list of **all** effective rules, all files combined, then the list of
   `NOPASSWD` rules — these are your priority audit points.

### Exercise 6.6 — Edit Without Locking Yourself Out

1. Why should you never edit `/etc/sudoers` with `nano` directly?
2. Create the fragment `/etc/sudoers.d/ops` **with the dedicated tool**, and set the correct
   permissions (what are they, and what happens if they're too broad?).
3. Verify the syntax of the entire configuration **without** applying it.
4. Intentionally introduce a syntax error in a fragment, observe what the verification says,
   and correct it.

### Exercise 6.7 — Delegate Precisely

Write the rules corresponding to each of these needs, each in its own named fragment:

1. The `ops` group can do everything, with password.
2. The `dev` group can **only** restart and check the status of `nginx` and `ticketflow-api`,
   and read their logs — nothing else.
3. The `deploy` account can run **only one** deployment script, **without a password** (it is called
   by CI, without a terminal).
4. The `backup` account can run `rsync` and `tar` in read mode, without a password.
5. `bob` can edit his application's configuration files — **with `sudoedit`**, not with an
   editor run as root. Explain the security difference.
6. Define a command alias `SERVICES` and a user alias `ADMINS`, and rewrite two of the
   above rules using them.
7. Set the `Defaults`: authentication cache of 5 minutes, command logging,
   custom warning message on first `sudo`.

### Exercise 6.8 — The Pitfalls

For each of these examples, explain **why the rule is dangerous** and propose a corrected version:

1. `bob ALL=(ALL) NOPASSWD: /usr/bin/vim`
2. `%dev ALL=(ALL) NOPASSWD: /bin/chmod`
3. `deploy ALL=(ALL) NOPASSWD: /usr/local/bin/deploy.sh` — the script being modifiable by `deploy`.
4. `carine ALL=(ALL) NOPASSWD: /usr/bin/find`
5. `%dev ALL=(ALL) ALL` « for now, we'll refine it later ».

### Exercise 6.9 — Audit and Logging

1. Find in the logs all commands run with `sudo` today, and by whom
   *(chapter [03](../../03_commandes_essentielles.md))*.
2. Find the **denied attempts** (user not in `sudoers`, or command not authorized).
3. Enable input/output logging of a `sudo` session for a sensitive account,
   and review a recorded session.
4. Write the rescue procedure: **you just broke `sudoers` and no longer have a root session
   open**. List, in order, what you try.

---

## ✅ Verification

- `getfacl /srv/projet` reflects exactly the table from exercise 6.4, and a new file created by
  any member of `dev` inherits the correct rights.
- `visudo -c` is clean, and each delegated right lives in its own fragment of `/etc/sudoers.d/`.
- `sudo -l` executed with `bob`, `deploy` and `backup` shows **strictly** what the requirements specify.
- No `NOPASSWD` rule applies to an editor, an interpreter, or a script modifiable by its caller.
- Your rescue procedure is written in `journal.md` **before** you need it.
