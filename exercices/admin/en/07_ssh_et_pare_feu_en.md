# 🛡️ A7 · SSH and Firewall

*Chapter [05 · Security](../../05_securite.md) — **firewall**, **SSH** and **fail2ban** sections*

> [!NOTE]
> **Objectives**: never connect by password again, harden `sshd` without cutting off access,
> know how to bounce and tunnel, close everything that doesn't need to be open, and silence robots.

> [!CAUTION]
> **The chapter where you lock yourself out.** Procedure for each modification:
> 1. snapshot the VM;
> 2. an SSH session **already open** that you close under no circumstances;
> 3. syntax verification (`sshd -t`, `nft -c -f`);
> 4. `reload` (not `restart`);
> 5. test from a **third** connection — and only then, close the rescue session.

---

## Exercise 7.1 — The Current State

1. Is the SSH server running? On which port? What version?
2. Which ports are listening on the machine, and by which processes?
   Which ones are listening on **all** interfaces rather than on the loopback?
3. Display the **effective** `sshd` configuration as the daemon understands it — not the file, the
   resolved configuration.
4. Is a firewall active? Which one? What are its current rules?
5. From your workstation, scan the open ports of the server. Record the result: this will be your
   "before / after".

---

## Exercise 7.2 — The Keys

1. Generate on your workstation an `ed25519` key pair with an identifying comment and a **passphrase**.
2. Why `ed25519` rather than RSA 2048? In what case do we keep RSA?
3. Deploy the public key on the `alice` account on the server — first with the dedicated tool, then
   **by hand** (to understand what it does).
4. What are the **exact** expected permissions on `~/.ssh` and on `~/.ssh/authorized_keys`?
   What happens if they're too broad? Test it.
5. Connect by key, then verify in the server logs that authentication went through the key.
6. Add your key to the SSH agent so you no longer have to type the passphrase at each connection.
7. Display the fingerprint of your public key. What is it used for in an inventory?

---

## Exercise 7.3 — Harden `sshd`

One point at a time, with syntax verification and testing between each:

1. Completely forbid `root` login.
2. Disable **password** authentication (and keyboard-interactive authentication).
3. Restrict login to only members of the `ops` group, plus the `backup` account.
4. Reduce attack surface: no X11 forwarding, no `AllowAgentForwarding`, no
   `PermitEmptyPasswords`, low `MaxAuthTries`.
5. Disconnect idle sessions after 10 minutes.
6. Set logging to `VERBOSE` — what does this add to the logs, and why is it useful?
7. Display a legal banner before login.
8. Change the listening port to 2222. What does this really provide, and what doesn't it?
   What must you remember to modify **at the same time** (firewall, fail2ban, `~/.ssh/config`,
   SELinux if applicable)?

### Exercise 7.4 — The `Match` Blocks

1. The `backup` account should only be able to run **rsync**, without an interactive shell.
2. `ops` members connected **from the internal network** have tunnel rights; others don't.
3. An `sftp-client` account is confined to its folder by a `chroot`, without shell access.
4. Verify each rule by actually connecting with the relevant account.
5. Restrict, **on the `authorized_keys` side**, the deployment key: only one authorized command, from a
   given IP, without tunnels or terminal allocation. Why is this restriction complementary to
   the `Match` one?

### Exercise 7.5 — The Client, Tunnels and Transfers

1. Write a `~/.ssh/config` on your workstation: an alias `srv-tickets` with the host, port, user
   and key; common values for all hosts; connection reuse.
2. Then connect with a simple `ssh srv-tickets`.
3. Reach a second machine **via** the server (jump).
4. Open a **local** tunnel making the server's PostgreSQL database (which only listens on `127.0.0.1`)
   reachable from your workstation on port 5433. Verify with a client.
5. Open a SOCKS proxy and route a browser through the server.
6. Transfer a file with `scp`, then a folder with `rsync`. Why use `rsync` as soon as the operation
   repeats?
7. `known_hosts`: what happens if the server's fingerprint changes? How do you react **correctly**
   (and why deleting the line without thinking is a bad answer)?

---

## Exercise 7.6 — The Firewall

Choose `ufw` **or** `nftables` and do the entire exercise with the same tool.

1. Display the current rules and the default policy.
2. Default policy: **deny all inbound**, allow all outbound.
3. Allow SSH **before anything else**. Why is this order non-negotiable?
4. Allow HTTP and HTTPS from anywhere.
5. Allow PostgreSQL **only from the internal subnet**.
6. Allow the administration port only from your IP, with a **comment** on the rule.
7. Activate the firewall, then verify from your workstation what is reachable and what is no longer.
8. Limit the connection rate of SSH to slow down robots.
9. Log rejected packets, with a rate limit. Where do these lines end up?
10. Delete a rule that became unnecessary, list the rules **numbered**, and make the configuration
    persistent on reboot. Verify by actually rebooting the VM.
11. The service is running but not reachable from outside. Give the **four** hypotheses to test,
    in order, and the command for each.

---

## Exercise 7.7 — `fail2ban`

1. Install and start `fail2ban`, then display the list of active jails.
2. Why do we configure `jail.local` and not `jail.conf`?
3. Configure the SSH jail: 5 attempts, 10-minute window, 1-hour ban,
   taking into account **your** SSH port.
4. Add your administration network to the whitelist — the classic oversight.
5. Intentionally trigger a ban from another machine, observe it, then unban the IP.
6. Display the jail statistics: how many attempts, how many bans?
7. Verify how `fail2ban` connects with your firewall: where are the ban rules
   actually written?

---

## ✅ Verification

- An `ssh root@srv-tickets` fails; an `ssh -o PreferredAuthentications=password alice@…` also fails.
- You connect by key, on the new port, by typing `ssh srv-tickets`.
- A port scan from outside shows only what you explicitly opened —
  compare with the inventory from exercise 7.1.
- The firewall survives a complete VM reboot.
- Five failed connection attempts from a test IP ban it, and your admin network cannot
  be banned.
- `journal.md` documents: SSH port, authorized accounts, open ports and for whom, rescue procedure.
