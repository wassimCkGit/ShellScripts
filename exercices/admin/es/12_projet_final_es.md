# 🎓 A12 · Proyecto final — endurecer y explotar `srv-tickets`

*Todo el recorrido: capítulos [01](../../01_base.md) a [06](../../06_shellscript/), [08](../../08_docker_administration.md),
[09](../../09_tmux.md), [10](../../10_shell_config.md) y [11](../../11_advanced_config.md)*

> [!NOTE]
> **Objetivo**: retomar una máquina abandonada y hacerla **tenable** — es decir,
> documentada, endurecida, respaldada, supervisada y reproducible. El entregable no es una orden:
> es un servidor del que puede dar cuenta.

> [!CAUTION]
> Reparta de una **VM nueva** con lo existente descrito en el [README del recorrido](README_es.md#-el-hilo-rojo--srv-tickets):
> todo el mundo como root, SSH por contraseña, sin cortafuegos, sin respaldo. Snapshot a cada etapa.

---

## 🎯 El pliego de condiciones

Tiene una semana (ficticia) y un servidor en producción que **no puede apagar**
más de algunos segundos. Al final, un auditor pasa: debe poder verificar cada punto siguiente
leyendo su documentación e lanzando sus órdenes.

---

## Etapa 0 — Evaluación de la situación *(capítulos [01](../../01_base.md) y [03](../../03_commandes_essentielles.md))*

Produzca `/root/audit-inicial.md`, generado tanto como sea posible por órdenes:

1. Sistema: distribución, kernel, *uptime*, recursos, particiones y tasa de relleno.
2. Cuentas: humanas, sistemas, que tienen shell, que están en `sudo`, que sin contraseña.
3. Servicios activos, servicios en fallo, servicios **escuchando en el exterior**.
4. Puertos abiertos, vistos desde la máquina **y** desde el exterior.
5. Tareas planificadas existentes (crontabs de todos los usuarios, `/etc/cron.*`, *timers*).
6. Archivos SUID, archivos *world-writable*, archivos pertenecientes a UID inexistentes.
7. Lo que Docker tiene en ejecución, con qué permisos y qué puertos.
8. Los cinco riesgos que juzga más urgentes, clasificados, con una oración de justificación cada uno.

## Etapa 1 — Las cuentas *(capítulos [02](../../02_file_permissions.md) y [02.1](../../02.1_user_management.md))*

1. Cree una cuenta nominativa por persona, con el grupo correcto (`ops`, `dev`), la política
   de vencimiento correcta y el shell correcto.
2. Cree las cuentas técnicas `deploy` y `backup`, sin shell de conexión.
3. Establezca los permisos y la propiedad correctos en `/srv/ticketflow`, `/var/log/ticketflow` y sus scripts —
   los directorios y los archivos no deben tener los mismos.
4. Establezca SGID y sticky bit donde se impongan, y justifique cada colocación en el registro.
5. Retire a `root` la posibilidad de conectarse directamente, una vez — y solo una vez —
   que una cuenta nominativa funcione en `sudo`.

## Etapa 2 — La delegación *(capítulo [05](../../05_securite.md), parte sudo)*

1. Un fragmento `sudoers.d` por necesidad, nombrado y comentado: `ops`, `dev`, `deploy`, `backup`.
2. `dev` solo puede reiniciar y consultar el estado de los servicios aplicativos. `deploy` solo puede lanzar
   un script, sin contraseña, y este script no le pertenece.
3. Las ACL de `/srv/ticketflow` respetan el cuadro del ejercicio [6.4](06_acl_et_sudo_es.md).
4. `sudo -l` ejecutado con cada cuenta prueba que el pliego se respeta — copie las salidas
   en el registro.
5. El procedimiento de emergencia "`sudoers` roto" está escrito **antes** de necesitarlo.

## Etapa 3 — El acceso *(capítulo [05](../../05_securite.md), partes SSH y cortafuegos)*

1. Autenticación por clave únicamente, `root` prohibido, acceso restringido a grupos autorizados,
   sesiones inactivas cortadas, registro verbose, aviso.
2. Cortafuegos: todo cerrado en entrada excepto lo justificado, cada regla **comentada**, PostgreSQL
   únicamente desde la red interna, configuración persistente al reinicio.
3. `fail2ban` activo en SSH, con su red de administración en lista blanca.
4. Un escaneo desde el exterior, **antes / después**, en el registro.
5. La prueba de que no se encerró: un reinicio completo de la VM, seguido de una conexión exitosa.

## Etapa 4 — La explotación *(capítulo [04](../../04_installation_et_services.md))*

1. La aplicación se ejecuta bajo una cuenta dedicada, en una unidad systemd endurecida, reiniciada automáticamente.
2. Los registros están en el diario, persistentes, limitados en tamaño, y `logrotate` gestiona los registros aplicativos.
3. Las actualizaciones de seguridad se aplican, y tiene una posición escrita sobre actualizaciones
   automáticas (a favor o en contra, y por qué).
4. Todo lo planificado está documentado: qué, cuándo, por qué, dónde van las salidas.

## Etapa 5 — La herramienta *(capítulo [06](../../06_shellscript/))*

Tres scripts, respaldados en `lib-admin.sh`, instalados en `/usr/local/bin`, con `--help` y `--dry-run`:

| Script | Rol |
|--------|-----|
| `srv-audit` | repite la auditoría de la etapa 0 y señala qué ha cambiado desde la última vez |
| `srv-backup` | respalda archivos + base + volúmenes Docker, con rotación y suma de control |
| `srv-check` | control de salud: servicios, disco, carga, certificados, contenedores; código de salida parlante |

1. Cada uno pasa ShellCheck sin advertencia.
2. Cada uno falla correctamente, con un código de salida distinto por causa, y traza en el registro del sistema.
3. `srv-backup` interrumpido no deja ningún archivo incompleto.
4. `srv-check` está planificado cada 5 minutos y **le avisa** sin que tenga que mirar.
5. Se efectuó una **restauración real** desde un respaldo producido por `srv-backup`.

## Etapa 6 — Docker *(capítulo [08](../../08_docker_administration.md))*

1. Rotación de registros configurada globalmente, ningún contenedor como root ni en `--privileged`.
2. Límites de memoria, CPU y PID en cada servicio; política de reinicio explícita.
3. La stack se reinicia sola tras un reinicio de la máquina (unidad systemd).
4. Los volúmenes son respaldados por `srv-backup`, y se probó una restauración.
5. Se planificó una limpieza semanal, y sabe exactamente qué elimina.

## Etapa 7 — El entorno y la reproducibilidad *(capítulos [10](../../10_shell_config.md) y [11](../../11_advanced_config.md))*

1. `/etc/profile.d` lleva el `PATH`, el `umask` y el prompt root distintivo; `/etc/skel` está completado.
2. Sus dotfiles de administrador se despliegan en una orden en una máquina nueva.
3. Sus archivos de configuración modificados (`/etc/ssh/sshd_config`, `sudoers.d`, unidades, `daemon.json`,
   `logrotate.d`) están **versionados** en algún lugar, con historial de sus modificaciones.
4. Un colega puede reconstruir este servidor a partir de su documentación únicamente. Hágaselo verificar:
   dé su `journal.md` a otro y vea intentarlo, sin ayudarle.

---

## 📄 El entregable

Una carpeta `/root/dossier-serveur/` (versionada) que contenga:

1. `audit-inicial.md` — el estado a su llegada.
2. `journal.md` — lo que cambió, cuándo, y **por qué**.
3. `exploitation.md` — los procedimientos: iniciar, detener, desplegar, respaldar, **restaurar**,
   qué hacer si el disco está lleno, qué hacer si la aplicación no responde.
4. `acces.md` — quién tiene acceso a qué, con qué derecho, y cómo se retira el acceso cuando alguien se va.
5. `scripts/` — sus scripts y su `README.md`.
6. `audit-final.md` — la misma auditoría que en la etapa 0, para comparación. Es su demostración.

---

## ✅ Criterios de éxito

- [ ] No es posible conectarse como `root` ni por contraseña; cada acción es trazable a una persona.
- [ ] Un escaneo externo solo muestra los puertos que puede justificar uno a uno.
- [ ] `systemctl --failed` está vacío, y la aplicación se reinicia sola tras un reinicio completo de la VM.
- [ ] Un respaldo fue **restaurado** de verdad, y el procedimiento está escrito.
- [ ] `srv-check` detecta una pana que provoca (detiene un servicio, llena el disco) y
      le avisa sin que tenga que mirar.
- [ ] `--dry-run` existe y funciona en todas sus órdenes destructivas.
- [ ] Ningún secreto aparece en un archivo legible por todos, ni en un historial, ni en una línea
      de orden, ni en un repositorio Git.
- [ ] Puede irse quince días de vacaciones: alguien más sostiene el servidor con su carpeta.

> [!TIP]
> Este último punto es el único que realmente importa. Un servidor que solo un humano sabe administrar
> no está administrado: está en manos de alguien.
