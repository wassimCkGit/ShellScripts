# 🐳 A10 · Docker for the System Administrator

*Chapter [08 · Docker for System Administrators](../../08_docker_administration.md)*

> [!NOTE]
> **Objectives**: operate a Docker host without it becoming a lawless zone — configure the daemon,
> master volumes, networks and logs, limit resources, harden containers, backup, clean up
> and launch a stack at boot.

> [!IMPORTANT]
> We don't build images here: writing `Dockerfile`s is covered in
> [chapter 07](../../07_docker.md) and in the [developer track](../dev/05_docker.md).
> Here, we operate what developers deliver.

---

## Exercise 10.1 — The Host

1. Where does Docker store its data on this machine? How much space does it take?
2. What storage driver, what logs driver, what network driver by default?
3. What daemon version, what client version? Does the daemon run as `root`?
4. Which systemd service runs it? Where are its logs?
5. Who on this machine can talk to the Docker daemon? Why does belonging to the `docker` group
   **actually** equate to being root on the host? Demonstrate it in one command — then reflect on what
   this means for your developers' `sudo` rule *(chapter [A6](06_acl_et_sudo.md))*.

---

## Exercise 10.2 — Configure the Daemon

1. Create `/etc/docker/daemon.json` with: log rotation (10 MB, 3 files), `live-restore`,
   and your default network addresses.
2. **Verify the syntax before restarting** the daemon. What happens if the file is invalid?
3. Restart the daemon and verify that the new values are properly applied.
4. Move the `data-root` to `/srv/docker` (a dedicated partition): what is the complete procedure,
   in what order, and what happens to existing containers?
5. What does `live-restore` do, and in what case is it precious in production?

---

## Exercise 10.3 — Volumes and Persistence

1. List the machine's volumes, with their size.
2. Launch a PostgreSQL on a **named** volume, create a table in it, destroy the container, start a new one
   on the same volume: is the data there?
3. Inspect the volume: where are the files actually located on the host? Who owns them?
4. Mount a configuration file from the host **read-only** in a container. Verify that a
   write is refused from inside.
5. What is the difference between a named volume and a *bind mount* from the perspective of permissions, backups
   and SELinux/AppArmor?
6. Find **orphan** volumes (no longer attached to any container) and their total size.

---

## Exercise 10.4 — Network

1. List Docker networks and their driver.
2. Create an application network `ticketflow-net` and attach two containers to it. Verify that they reach each other
   **by their name**.
3. Can a container on the default `bridge` network reach a container on the application network?
   Why is this difference a security feature?
4. Publish a port, then verify what is **actually exposed** on the host, and on which interfaces.
5. Publish a port **only on the loopback interface**. In what case is this the right reflex?
6. The pitfall of this chapter: your `ufw` forbids port 8080, and yet the container is reachable from
   outside. Explain why, and give **two** ways to fix it.
7. From a container, resolve the name of another service and test the connection — without installing tools
   in the application image *(hint: a disposable container attached to the same network)*.

---

## Exercise 10.5 — Logs

1. Display a container's logs, then the last 50 lines, then in follow mode, then from the last 10 minutes.
2. Where are these logs physically stored on the host? How large are they?
3. Find the container whose logs take up the most space.
4. Configure rotation **globally** in `daemon.json`, then **per container** in a Compose file.
   Which applies to containers already created?
5. Send a service's logs to `journald` and find them with `journalctl`
   *(chapter [04](../../04_installation_et_services.md))*.
6. A container filled up `/var/log` — or rather `/var/lib/docker`. What is the immediate course of action,
   and the lasting fix?

---

## Exercise 10.6 — Resources

1. Display real-time consumption of all containers.
2. Launch a container limited to **512 MB** of memory and **0.5 CPU**. Verify the limit from the host.
3. Trigger a memory overage: what does Docker do? Where do you see it?
4. Limit the number of processes (`--pids-limit`) and explain what it protects against.
5. Apply these limits in a Compose file rather than on the command line.
6. Configure the restart policy `unless-stopped` on a service, reboot the machine,
   verify. What is the difference from `always`?
7. Add a `HEALTHCHECK` on the command line to a container and find its health status.
   What does Docker do with an `unhealthy` container — and what does it **not** do?

---

## Exercise 10.7 — Harden

1. Verify under which user the main process of each of your containers runs.
   How many run as root?
2. Launch a container with a non-privileged user imposed from outside the image.
3. Launch a container with **root filesystem read-only**, with a `tmpfs` for what must be
   written.
4. Apply standard hardening: `--cap-drop=ALL` then only the necessary capabilities,
   `--security-opt=no-new-privileges`.
5. What are the consequences of `--privileged`? Give one case where it's legitimate and three where it's laziness.
6. A container that mounts `/var/run/docker.sock`: explain the risk precisely.
7. Audit your containers: write a command line that lists, for each, its user, its
   restart policy and its published ports *(hint: `docker inspect --format`)*.
8. *(Bonus)* Test **rootless** mode: what do you gain, what do you lose?

---

## Exercise 10.8 — Backup and Restoration

1. Back up an entire **volume** to a `.tar.gz` archive on the host, using a disposable container.
2. Restore this archive to a new volume and verify the data.
3. Make an **application dump** of the database (`pg_dump`) from the container, without exposing the password.
   Why is an application dump better than a volume copy for a database?
4. Export an image to a file, delete the image, re-import it.
5. What is the difference between `docker save`/`load` and `docker export`/`import`?
6. Integrate volume backups into your `backup.sh` from chapter [A9](09_scripts_administration.md).

---

## Exercise 10.9 — Cleanup and Disk Space

1. Display what Docker consumes, broken down by images, containers, volumes and build cache.
2. Delete stopped containers, then untagged images.
3. What is the exact difference between `docker system prune` and `docker system prune -a`? Which one is
   dangerous on a production host, and why?
4. Write a scheduled task that cleans up each week — explicitly excluding what must
   never be deleted. Where do you install it *(chapter [04](../../04_installation_et_services.md))*?
5. Test your cleanup command **in dry-run** before scheduling it: how do you do it?

---

## Exercise 10.10 — A Stack in Production

1. Retrieve the `05_stack_compose` stack from the repository *(see [`07_docker/05_stack_compose/`](../../07_docker/05_stack_compose/))*
   and deploy it to `/srv/ticketflow`.
2. Write the systemd unit that launches this stack **at machine startup** and shuts it down cleanly
   at shutdown. Test with an actual reboot.
3. Layer the production file: what differences from the development file, and how do you verify
   **what is actually applied**?
4. Deploy a new image version without noticeable downtime: what is the sequence?
5. Set up minimal monitoring: a script that reports any stopped or `unhealthy` container,
   scheduled every 5 minutes, writing to the system log
   *(chapter [A9](09_scripts_administration.md))*.
6. Classic failures — for each, both the diagnostic command **and** the fix:
   - the container restarts in a loop;
   - it cannot reach the database;
   - the port is already in use on the host;
   - the volume is mounted but empty;
   - "it worked before the image update".

---

## ✅ Verification

- `docker info` shows the configured log rotation, and no container has a log file of several GB.
- No application container runs as root or with `--privileged`, and you can prove it in one command.
- The stack restarts all by itself after a complete VM reboot.
- A volume has been backed up **and actually restored**.
- Weekly cleanup is scheduled and you know exactly what it deletes.
- `journal.md` documents: where the data is, what is backed up, how to restore.
