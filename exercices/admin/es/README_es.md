# 🛠️ Recorrido administrador del sistema — ejercicios

> Una serie de ejercicios **sin solución**, orientada hacia administración: tomar en mano un servidor,
> mantener sus permisos y cuentas, explotar sus registros, gestionar sus paquetes, servicios y tareas
> planificadas, endurecerlo, equiparlo con scripts, hacer correr Docker correctamente e industrializar
> el entorno de todos sus usuarios.
>
> El recorrido cubre los capítulos **01 a 06**, **08**, **09**, **10** y **11**. El capítulo
> **07 · Docker para los desarrolladores** se aborda aquí solo bajo el aspecto de explotación (capítulo 08):
> la escritura de `Dockerfile` se trata en el [recorrido desarrollador](../dev/) — el complemento de este,
> del lado de la aplicación.

---

## 🧵 El hilo rojo — `srv-tickets`

Reasume la administración de **`srv-tickets`**, la VM Debian 12 expuesta en Internet que aloja
la aplicación `ticketflow` (una API + una base PostgreSQL) escrita por el equipo de desarrollo.

Lo existente que se le deja:

| Elemento | Estado a su llegada |
|----------|----------------------|
| Cuentas | todo el mundo se conecta como `root`, contraseña compartida |
| SSH | puerto 22, autenticación por contraseña, `PermitRootLogin yes` |
| Cortafuegos | inexistente |
| Respaldos | "hay un `.tar.gz` en algún lugar de `/root`" |
| Supervisión | ninguna: se enteran de las panas por los usuarios |
| Docker | instalado "al azar", los registros llenan el disco |

**Su misión, a lo largo de los ejercicios: convertirlo en un servidor tenable.** Cada capítulo retoma
el servidor donde el anterior lo dejó, pero cada archivo sigue siendo jugable independientemente.

---

## 📋 Los ejercicios

| # | Archivo | Versa sobre |
|---|---------|-----------|
| A1 | [Puesta en marcha del servidor](01_prise_en_main_es.md) | cap. [01](../../01_base.md) y [09](../../09_tmux.md) — arborescencia del sistema, registros, procesos, redirecciones, tmux |
| A2 | [Permisos y propiedad](02_permissions_es.md) | cap. [02](../../02_file_permissions.md) — octal, simbólico, SUID / SGID / sticky, `chown`, `umask` |
| A3 | [Usuarios, grupos y cuentas](03_utilisateurs_et_groupes_es.md) | cap. [02.1](../../02.1_user_management.md) — ciclo de vida de cuentas, `/etc/passwd`, `/etc/shadow`, cuentas de servicio |
| A4 | [La caja de herramientas](04_boite_a_outils_es.md) | cap. [03](../../03_commandes_essentielles.md) — `grep`, `sed`, `awk`, `find`, `xargs`, `du` / `df`, `tar` |
| A5 | [Paquetes, servicios y cron](05_paquets_services_cron_es.md) | cap. [04](../../04_installation_et_services.md) — `apt`, `systemctl`, unidad personalizada, `journalctl`, `cron` |
| A6 | [ACL y delegación `sudo`](06_acl_et_sudo_es.md) | cap. [05](../../05_securite.md) — ACL, máscara, herencia, `sudoers`, `sudoedit`, auditoría |
| A7 | [SSH y cortafuegos](07_ssh_et_pare_feu_es.md) | cap. [05](../../05_securite.md) — claves, endurecimiento `sshd`, `Match`, túneles, `ufw` / `nftables`, `fail2ban` |
| A8 | [AppArmor / SELinux](08_mac_apparmor_selinux_es.md) | cap. [05](../../05_securite.md) — DAC vs MAC, modos, perfiles, contextos, booleanos, diagnóstico |
| A9 | [Scripts de administración](09_scripts_administration_es.md) | cap. [06](../../06_shellscript/) — auditoría, respaldo con rotación, supervisión, `getopts`, `trap`, biblioteca |
| A10 | [Docker para el administrador](10_docker_administration_es.md) | cap. [08](../../08_docker_administration.md) — demonio, volúmenes, redes, registros, recursos, endurecimiento, respaldos |
| A11 | [El entorno de los usuarios](11_environnement_utilisateurs_es.md) | cap. [10](../../10_shell_config.md) y [11](../../11_advanced_config.md) — `/etc/profile.d`, `/etc/skel`, dotfiles multi-servidores |
| A12 | [Proyecto final: endurecer y explotar `srv-tickets`](12_projet_final_es.md) | todo lo anterior |

> [!IMPORTANT]
> Estos ejercicios **no tienen intencionalmente solución**. Un administrador no trabaja de memoria:
> trabaja con `man`, la documentación de la distribución y una VM de prueba. Cada ejercicio termina
> con **criterios de verificación** — si los cumple, su solución es sólida.

---

## 🚀 Entorno

> [!CAUTION]
> **Nada de esto se hace en una máquina de la que necesita.** Este recorrido corta accesos SSH,
> cierra puertos, bloquea cuentas y modifica `sudoers`: tres cuartas partes de los ejercicios pueden
> encerrarlo. Es precisamente el interés — pero en una VM desechable de la que tiene un snapshot.

| Elemento | Recomendación |
|----------|----------------|
| Máquina | Una **VM** Debian 12 (VirtualBox, UTM, Proxmox, Hyper-V) — `systemd`, SSH y el cortafuegos deben ser reales |
| Snapshot | Tome un **snapshot** antes de cada capítulo. Lo usará. |
| Acceso | Dos terminales abiertos permanentemente: uno para trabajar, **uno segundo ya conectado** que nunca cierre |

> [!WARNING]
> **La regla que atraviesa todo el recorrido**: antes de recargar `sshd`, `sudoers` o el cortafuegos,
> mantenga una sesión privilegiada **ya abierta** y pruebe la nueva configuración **desde una tercera conexión**.
> Mientras la prueba no pase, no cierre la sesión de emergencia.

---

## ✅ Cómo trabajar

1. Lea el enunciado por completo, **luego** verifique la orden en `man` antes de ejecutarla como root.
2. Respalde sistemáticamente un archivo de configuración antes de modificarlo — y mantenga el `.bak`
   hasta que el servicio se reinicie sin error.
3. Verifique la sintaxis antes de aplicar: `visudo -c`, `sshd -t`, `nft -c -f`, `dockerd --validate`.
4. Anote lo que hace. Un capítulo de cada dos le pedirá que rinda cuenta del estado del servidor:
   mantenga un archivo `journal.md` a medida que avanza, es el reflejo del oficio.
5. Versione sus archivos de configuración (`/etc` en un repositorio Git, o al mínimo sus fragmentos).
