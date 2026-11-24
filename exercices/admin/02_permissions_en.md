# 🔐 A2 · Permissions and Ownership

*Chapter [02 · File Permissions](../../02_file_permissions_en.md)*

> [!NOTE]
> **Objectives**: read a line of `ls -l` without hesitation, set appropriate permissions in octal or symbolic
> notation, understand what `r`, `w`, and `x` mean on a directory, master SUID / SGID / the sticky bit,
> and repair broken ownership.

---

## Exercise 2.1 — Read

Here is an excerpt from `ls -l` taken on `srv-tickets`:

```text
-rwxr-x---  1 root   ops     2048  /usr/local/bin/backup.sh
-rw-r--r--  1 root   root     742  /etc/ticketflow/app.conf
drwxrws---  4 root   dev     4096  /srv/projet/
-rw-------  1 alice  alice   1675  /home/alice/.ssh/id_ed25519
-rwsr-xr-x  1 root   root   68208  /usr/bin/passwd
drwxrwxrwt  8 root   root    4096  /tmp
lrwxrwxrwx  1 root   root      21  /root/conf -> /srv/ticketflow/conf
```

For **each line**: what type of object is it? What can the owner, group, and others do?
What is the corresponding **octal** notation? And for the last three, **which special bit** is set,
and what is its concrete purpose here?

---

## Exercise 2.2 — Set Permissions

Give the octal `chmod` command for each of these requirements, then the equivalent **symbolic** version:

| Path | Requirement |
|------|-------------|
| `/root/.ssh/id_ed25519` | private key: owner only, read / write |
| `/etc/ticketflow/app.conf` | readable by everyone, modifiable by root only |
| `/usr/local/bin/backup.sh` | executable by root and by the `ops` group, invisible to others |
| `/srv/ticketflow/backups` | traversable and searchable by the group, forbidden to others |
| `/var/log/ticketflow` | the service can write in it, admins can read it, nobody else can enter |

Then **work the other way**: translate the permissions `640`, `750`, `775`, `1777`, and `4755` into `rwx` notation.

-rw-r-----	
-rwxr-x---	
-rwxrwxr-x	
-rwxrwxrwt	
-rwsr-xr-x

---

## Exercise 2.3 — Directory Cases

1. On a file, `r`, `w`, and `x` are straightforward. On a **directory**, what does each one allow exactly?
`r` : ls
`w` : Add, rename, and delete files
`x` : ls + cd  
2. Create `/srv/test-droits`, put a file in it, and remove `x` from the directory for your test user.
   Can the user still **list** the directory?
   yes, the user can still list the directory, because r lets them read the names. Without it, the file’s own permissions are never even checked. **Read** a file whose name they know? Explain. No, they can’t read a file even when they know its name, because opening /srv/test-droits/file.txt means passing through the directory, and that needs x. 
3. Repeat the test by removing `r` but leaving `x`. What can the user do this time?
   I can "cd" the folder and "cat"  to go into the file and read it but i can not "ls" the folder.
4. How can a user delete a file they do **not** own and have **no** write permission for? A user can delete a file they don’t own and can’t write to if they have write and execute permission on the parent directory. Deleting removes an entry from the directory, so only the directory’s permissions matter. To prevent it, set the sticky bit on the directory (chmod +t dir / chmod 1777 dir, like /tmp): then only the file’s owner, the directory’s owner or root can delete or rename files in it.

---

## Exercise 2.4 — Apply in Bulk

1. Apply `750` recursively to `/srv/ticketflow`.
2. Problem: this also makes **files** executable, which is not desired. Write the two commands
   that set `750` on **directories** and `640` on **files**, using `find`
   *(chapter [03](../../03_commandes_essentielles_en.md))*.
3. Do the same with symbolic notation using the uppercase `X`. What does it do exactly?
4. Why is `chmod -R 777` almost always the wrong answer? Give two concrete consequences.

---

## Exercise 2.5 — Special Bits

1. **SGID on a directory**: make every file created in `/srv/projet` belong automatically to the `dev`
   group, regardless of who creates it. Verify this with two different accounts.
2. **Sticky bit**: make sure that, in this shared directory, each person can delete only **their own**
   files. Verify by trying to delete another person's file.
3. **SUID**: why does `/usr/bin/passwd` need it? What would happen without it?
4. Search the entire system for files carrying the **SUID** bit *(hint: `find / -perm`)* and
   explain why this list is a security audit point.
5. Set the SUID bit on a copy of `/bin/bash` in `/tmp`, observe the danger, and then **delete it
   immediately**. What would you do if you found this file on a production server?

---

## Exercise 2.6 — Owner and Group

1. Give `/srv/ticketflow` to the `deploy` user and the `ops` group, recursively, with a single command.
2. Change **only the group** of `/var/log/ticketflow` to `adm`.
3. A `git pull` run with `sudo` left files owned by root in a developer's home directory.
   Write the command that repairs the ownership of the entire home directory.
4. Copy a file with `cp`, then with `cp -a`: what happens to ownership, group, permissions, and timestamps?
   Which option is used for a backup?
5. Can an unprivileged user give one of their files to someone else? Why?

---

## Exercise 2.7 — `umask`

1. Display your current `umask`. What permissions does a newly created **file** receive? And a **directory**?
2. Verify this by creating both and reading their permissions.
3. Change the `umask` to `027` and repeat the test. What does this change for "others"?
4. Where should this value be written so it applies to **all users** at every login?
   *(See also chapter [10](../../10_shell_config_en.md).)*
5. Why is a `002` `umask` common on a collaborative work server, and dangerous elsewhere?

---

## ✅ Verification

- You translate `rwxr-x---` ↔ `750` in both directions without using a conversion table.
- `/srv/projet` has both SGID **and** the sticky bit, and you proved this with two accounts.
- `/srv/ticketflow` has directories set to `750` and files set to `640` — not the reverse.
- You can produce the list of SUID binaries on the machine and explain why it is monitored.
- No SUID shell copy remains in `/tmp`.
