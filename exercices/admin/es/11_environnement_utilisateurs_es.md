# ⚙️ A11 · El entorno de los usuarios

*Capítulos [10 · Configuración del shell](../../10_shell_config.md) y [11 · Configuración avanzada](../../11_advanced_config.md)*

> [!NOTE]
> **Objetivos**: dejar de configurar cada *home* a mano. Comprender qué archivo se lee y cuándo,
> imponer una configuración a toda la máquina, preparar el esqueleto de nuevas cuentas, e implementar
> su propia configuración de administrador en un parque de servidores.

---

## Ejercicio 11.1 — Quién lee qué, y cuándo

1. ¿Cuál es el shell por defecto en este servidor? ¿Qué shells están instalados?
2. Establezca el cuadro **completo**: para un shell de login, un shell interactivo no-login, un `ssh servidor`,
   un `ssh servidor 'orden'` y un script, **qué archivos se leen, en qué orden**?
   Verifíquelo experimentalmente con `echo` temporales en cada archivo, luego elimínelos.
3. ¿Por qué una variable definida en `~/.bashrc` no se ve por una tarea `cron`
   *(capítulo [04](../../04_installation_et_services.md))*? ¿Y por un `ssh servidor 'orden'`?
4. ¿Dónde coloca: un alias? una modificación del `PATH`? una variable necesaria para un servicio?
   Justifique cada respuesta.
5. ¿Cómo aplicar una modificación sin cerrar su sesión — dos métodos.

---

## Ejercicio 11.2 — La configuración del sistema

1. ¿Cuál es la diferencia entre `/etc/profile`, `/etc/profile.d/*.sh`, `/etc/bash.bashrc` y `~/.bashrc`?
2. Escriba `/etc/profile.d/ticketflow.sh` que, **para todos los usuarios**:
   - añada `/srv/ticketflow/scripts` al `PATH`;
   - defina `TICKETFLOW_HOME`;
   - fije un `umask` a `027` *(capítulo [02](../../02_file_permissions.md))*.
3. Verifique que un usuario nuevo lo hereda, y que un usuario existente también.
4. Escriba `/etc/profile.d/prompt-root.sh` que dé un **prompt rojo y explícito** a las sesiones root.
   ¿Por qué es más que un capricho?
5. Añada un aviso del sistema (`/etc/motd` o `/etc/issue.net`) recordando que la máquina está en producción
   y registrada. ¿Cuál es la diferencia entre ambos archivos, y cuál se muestra antes de la autenticación?
6. ¿Debe una configuración del sistema evitar que un usuario la sobrecargue en su *home*?
   Discuta, distinguiendo comodidad y restricción de seguridad.

---

## Ejercicio 11.3 — El esqueleto de las nuevas cuentas

1. ¿Qué contiene `/etc/skel`? ¿Se aplican estos archivos a las cuentas ya creadas?
2. Complételo: un `.bashrc` con los alias internos, un `.vimrc` mínimo, un `.tmux.conf`, un
   `README-serveur.txt`.
3. Cree una cuenta de prueba y verifique que hereda todo, con los permisos y propietario correctos.
4. Escriba un script que aplique una nueva versión de `.bashrc` **a todas las cuentas existentes**,
   respaldando la anterior y sin sobrescribir una personalización
   *(capítulo [06](../../06_shellscript/) — es un ejercicio de scripting completo)*.
5. ¿Por qué un `cp /etc/skel/.bashrc /home/*/` brutal es mala idea? Cite tres problemas.

---

## Ejercicio 11.4 — Alias y funciones del administrador

Escribalas en **su** configuración, luego decida cuáles merecen ir en `/etc/profile.d`.

1. `ll`, `..`, `...`, y la securización de `rm`, `cp`, `mv`.
2. `ports` — los puertos en escucha con el proceso asociado.
3. `failed` — los servicios en fallo.
4. `logsvc <servicio>` — sigue los registros de un servicio.
5. `psg <motivo>` — busca un proceso.
6. `bigfiles [ruta]` — los 10 archivos más grandes de una ruta.
7. `backup <archivo>` — copia horodatada de un archivo antes de modificationlo. ¿Por qué esta **debe** ser
   una función y no un alias?
8. ¿Cómo lanzar la **verdadera** orden `rm` evitando su alias securizado?

---

## Ejercicio 11.5 — `PATH`, historial y comodidad

1. Muestre el `PATH`, un directorio por línea. ¿El `PATH` de `root` es igual al suyo? ¿Por qué?
2. Añada `/srv/ticketflow/scripts` al `PATH`, **solo si no está ya**, de forma permanente.
3. ¿Debe añadirse al principio o al final? ¿Qué riesgo en cada caso?
4. Configure el historial: 10 000 entradas, sin duplicados, **horodataje**, compartición entre múltiples
   terminales. ¿Por qué el horodataje del historial es útil tras un incidente?
5. Haga que el historial de `root` también esté horodatado y conservado. ¿Es esto buena práctica de
   seguridad, o un riesgo? Discuta.
6. Encuentre en el historial la última orden que contiene `systemctl` — con `grep`, luego con `CTRL+r`.

---

## Ejercicio 11.6 — Su configuración, en todo el parque

1. Cree un repositorio `dotfiles-admin` que contenga su `.bashrc`, `.vimrc`, `.tmux.conf`, `.ssh/config`
   *(sin claves ni secretos!)* y sus scripts.
2. Reemplace los archivos de su *home* por **enlaces simbólicos** al repositorio.
3. Escriba `install.sh`: crea los enlaces, **respalda** todo archivo existente antes de reemplazarlo,
   es idempotente, y tiene un `--dry-run`.
4. Saque del repositorio todo lo específico de una máquina o confidencial: cárguelo desde un
   `~/.bashrc.local` si existe, ignorado por Git.
5. Despliegue su configuración en una segunda VM en una orden. Cronométrelo.
6. Compare tres enfoques — un repositorio + script, GNU Stow, una herramienta de gestión de configuración
   (Ansible) — y diga cuál elegiría para 2 servidores, para 20, para 200.
7. *(Bonus)* Instale `fzf` y `tmux` con su configuración, y escriba una función que elija un
   servicio systemd con `fzf` para reiniciarlo.

> [!WARNING]
> En un servidor de producción, cada herramienta añadida es una dependencia más a mantener y una superficie
> de ataque más. La configuración de un servidor no es la de un puesto de trabajo:
> justifique cada adición.

---

## ✅ Verificación

- Una cuenta nueva creada hoy hereda automáticamente el `PATH`, el `umask`, los alias y los archivos
  de `/etc/skel`, sin intervención manual.
- Una sesión root es visualmente imposible de confundir con una sesión normal.
- `install.sh` reconstruye su entorno de administrador en una VM nueva, sin sobrescribir nada sin respaldo.
- Ningún secreto ni ninguna clave privada partió en el repositorio de dotfiles.
- Sabe decir, para cualquier línea de configuración, **en qué archivo** debe vivir
  y **por qué**.
