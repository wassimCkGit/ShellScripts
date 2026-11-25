# 🐧 Formación Linux — Administración de sistemas y scripting de shell

> Soporte de formación **BStorm**: los fundamentos de la administración de Linux, luego la escritura de scripts de shell, desde el primer `pwd` hasta la configuración avanzada del entorno de trabajo.

---

## 📚 Programa

| # | Capítulo | Contenido |
|---|----------|---------|
| 01 | [Los fundamentos de la terminal](01_base_es.md) | Navegación, archivos, redirecciones, editores, procesos |
| 02 | [Los permisos](02_file_permissions_es.md) | POSIX, `chmod`, `chown`, permisos especiales |
| 02.1 | [Usuarios y grupos](02.1_user_management_es.md) | `useradd`, grupos, contraseñas, `su` / `sudo` |
| 03 | [Comandos esenciales](03_commandes_essentielles_es.md) | `tree`, `grep`, `sed`, `awk`, `cut` / `sort` / `uniq`, `find`, `xargs`, `du`, `df`, `tar` |
| 04 | [Paquetes, servicios y cron](04_instalacion_y_servicios_es.md) | `apt` / `dnf`, `systemctl`, servicios personalizados, `journalctl`, `cron` |
| 05 | [Seguridad](05_securite_es.md) | ACL, `sudoers`, cortafuegos, SSH, AppArmor / SELinux |
| 06 | [**Scripting de shell**](#-06--scripting-de-shell) | *ver el detalle más abajo* |
| 07 | [Docker — los desarrolladores](07_docker_es.md) | Imágenes, contenedores, volúmenes, `Dockerfile`, `docker compose` |
| 08 | [Docker — el administrador](08_docker_administration_es.md) | Demonio, redes, registros, recursos, refuerzo, copias de seguridad |
| 09 | [tmux](09_tmux_es.md) | Sesiones, ventanas, paneles, trabajo remoto |
| 10 | [Configuración del shell](10_shell_config_es.md) | `.bashrc`, `.zshrc`, `PATH`, prompt, alias, dotfiles |
| 11 | [Configuración avanzada](11_advanced_config_es.md) | Oh My Zsh, Powerlevel10k, Starship, Oh My Tmux, herramientas modernas |

### 🐚 06 · Scripting de shell

| # | Capítulo | Contenido |
|---|----------|---------|
| 06.1 | [Los fundamentos](06_shellscript/01_Base/base.md) | Shebang, variables, matrices, `read` |
| 06.2 | [Condiciones y operadores lógicos](06_shellscript/02_operateur_logique/ejemplo.md) | `if` / `elif` / `else`, pruebas, `&&`, `\|\|`, `!` |
| 06.3 | [Los bucles](06_shellscript/03_boucles/ejemplo.md) | `for`, `while`, `until`, lectura de archivo |
| 06.4 | [Gestión de errores](06_shellscript/04_gestion_des_erreurs/gestion_des_erreurs.md) | `$?`, `set -euo pipefail`, `trap` |
| 06.5 | [Los argumentos](06_shellscript/05_Arguments/args.md) | `$1`, `$@`, `getopts` |
| 06.6 | [Las funciones](06_shellscript/06_fonctions/fonctions.md) | Declaración, argumentos, `local`, `source` |

---

## ✏️ Ejercicios

| # | Ejercicio | Cubre |
|---|----------|-----------|
| 3 | [Ejercicio 3](ejercicios/Ejercicio3.md) | Estructura, archivos, redirecciones — cap. 01 |
| 4 | [Ejercicio 4](ejercicios/Ejercicio4.md) | Usuarios y grupos — cap. 02.1 |
| 5 | [Ejercicio 5](ejercicios/Ejercicio5.md) | `grep`, `find`, `tar` — cap. 03 |
| 6 | [Ejercicio 6](ejercicios/Ejercicio6.md) | Creación de un servicio systemd — cap. 04 |
| 7 | [Ejercicio 7](ejercicios/Ejercicio7.md) | Tareas `cron` — cap. 04 |
| 8 | [Ejercicio 8](ejercicios/Ejercicio8.md) | `sudoers` y delegación de derechos — cap. 05 |
| 9 | [Ejercicio 9](ejercicios/Ejercicio9.md) | SSH y cortafuegos — cap. 05 |
| 10 | [Ejercicio 10](ejercicios/Ejercicio10/) | Scripts: condiciones y bucles — cap. 06 |
| 11 | [Ejercicio 11](ejercicios/Ejercicios11/) | Scripts: argumentos y creación de usuarios — cap. 06 |
| 12 | [Ejercicio 12](ejercicios/Ejercicios12/) | Scripts: funciones y biblioteca compartida — cap. 06 |

### 🧭 Los dos itinerarios temáticos

Dos conjuntos de ejercicios separados, **sin soluciones**, cada uno construido sobre un hilo conductor único y cubriendo solo los capítulos relevantes para su audiencia:

| Itinerario | Para quién | Contenido |
|----------|----------|---------|
| 👩‍💻 [**Desarrollador**](ejercicios/dev/) | Un desarrollador que trabaja bajo Linux diariamente | Terminal, búsqueda de código y registros, administración mínima útil para un dev, scripting, Docker, tmux, entorno de trabajo — alrededor de la aplicación `ticketflow` |
| 🛠️ [**Administrador de sistemas**](ejercicios/admin/) | Un administrador que retoma un servidor | Control inicial, permisos, cuentas, caja de herramientas, servicios y cron, ACL y `sudo`, SSH y cortafuegos, AppArmor / SELinux, scripts de administración, Docker en explotación, entorno de usuarios — alrededor del servidor `srv-tickets` |

Los dos se responden mutuamente: el itinerario de desarrollador construye la aplicación, el itinerario de administrador explota el servidor que la aloja.

---

## 🗂️ Organización del repositorio

```text
.
├── 01_base.md … 11_advanced_config.md   # los capítulos del curso
├── 06_shellscript/                      # el módulo de scripting, una carpeta por tema
│   └── NN_tema/
│       ├── tema.md                      # el curso
│       └── ejemplo*.sh                  # los scripts de ejemplo, ejecutables
├── 07_docker/                           # los proyectos Docker de ejemplo, construibles
│   └── NN_ejemplo/
│       ├── Dockerfile                   # comentado paso a paso
│       └── …                            # las fuentes de la aplicación
├── ejercicios/                          # enunciados y soluciones
│   ├── dev/                             # el itinerario de desarrollador, sin soluciones
│   └── admin/                           # el itinerario de administrador de sistemas, sin soluciones
├── ejemplo/                             # archivos de trabajo para las manipulaciones
└── docker-compose.yml                   # base de datos PostgreSQL de demostración
```

---

## 🚀 Configuración del entorno

Todas las manipulaciones requieren una máquina Linux en la que se puedan romper cosas sin consecuencias. Tres opciones:

| Opción | Para quién |
|--------|----------|
| Una **VM** (VirtualBox, UTM, Hyper-V) con Debian 12 | La más cercana a un servidor real — **recomendado** |
| Un **contenedor Docker** de Debian | El más rápido de configurar |
| **WSL 2** (Windows) | Cómodo, pero `systemd` y los servicios son particulares |

Un sandbox desechable en un comando:

```bash
docker run -it --rm --name lab debian:12 bash
```

```bash
# En el contenedor, para tener las herramientas del curso
apt update && apt install -y sudo vim nano tree htop cron tmux
```

> [!NOTE]
> Un contenedor no ejecuta `systemd` por defecto: las manipulaciones `systemctl` del [capítulo 04](04_instalacion_y_servicios_es.md) requieren una VM real.
>
> El `docker-compose.yml` del repositorio **no** es este entorno de trabajo: solo sirve para lanzar una base de datos PostgreSQL utilizada en demostración — se explica en el [capítulo 07](07_docker_es.md#docker-compose).

> [!WARNING]
> **Nunca hagas los ejercicios de administración en tu máquina principal.** Los capítulos 02, 02.1, 04 y 05 manipulan usuarios, servicios, cortafuegos y configuración SSH — cosas que pueden cortarte el acceso a tu propio sistema.

---

## 🧭 Convenciones del soporte

| Elemento | Significado |
|---------|---------------|
| <code>```bash</code> | Un comando para escribir en la terminal |
| <code>```text</code> | Una salida de comando o un extracto de archivo |
| `> [!TIP]` | Un consejo o una buena práctica |
| `> [!IMPORTANT]` | Un punto a no perder para entender lo que sigue |
| `> [!WARNING]` | Una trampa frecuente |
| `> [!CAUTION]` | Una operación destructiva o peligrosa |

Cada capítulo comienza con sus **objetivos** y termina con un **resumen** de los comandos vistos.

---

## 🔗 Recursos

- [`man` en línea](https://man7.org/linux/man-pages/) — el manual de referencia
- [ExplainShell](https://explainshell.com/) — desglosa cualquier línea de comandos
- [ShellCheck](https://www.shellcheck.net/) — análisis estático de scripts de shell
- [tldr pages](https://tldr.sh/) — ejemplos en lugar de manuales
- [Google Shell Style Guide](https://google.github.io/styleguide/shellguide.html) — convenciones de escritura

---

**➡️ Comenzar la formación: [01 · Los fundamentos de la terminal](01_base_es.md)**
