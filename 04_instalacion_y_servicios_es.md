# 04 · Instalación de paquetes, servicios y tareas planificadas

> [!NOTE]
> **Objetivos del capítulo**
> - Instalar, actualizar, buscar y eliminar paquetes (Debian y RedHat)
> - Controlar servicios con `systemctl`
> - Crear tu propio servicio `systemd`
> - Consultar logs con `journalctl`
> - Planificar tareas recurrentes con `cron`

## Tabla de contenidos

1. [Gestión de paquetes](#gestión-de-paquetes)
2. [Gestión de servicios](#gestión-de-servicios)
3. [Creación de un servicio personalizado](#creación-de-un-servicio-personalizado)
4. [Journald](#journald)
5. [CRON](#cron)
6. [Resumen](#resumen)

---

## Gestión de paquetes

Dos grandes familias de distribuciones, dos gestores de paquetes:

| Familia | Distribuciones | Gestor |
|---------|----------------|--------|
| **Debian** | Debian, Ubuntu, Mint, Raspberry Pi OS | `apt` |
| **RedHat** | RHEL, CentOS, Fedora, Rocky, Alma | `yum` (antiguo) / `dnf` (actual) |
| **Alpine** | Alpine | `apk` |

### Pre-instalación / actualización del índice

Antes de instalar un paquete, debes actualizar la base de datos **local** de paquetes, para obtener las últimas versiones disponibles.

En Debian:

```bash
sudo apt update
```

En RedHat, esta operación está integrada en la instalación.

> [!IMPORTANT]
> `apt update` actualiza la **lista** de paquetes disponibles; no instala nada.
> `apt upgrade` instala efectivamente las nuevas versiones. Los dos siempre van juntos.

### Instalar

En Debian:

```bash
sudo apt install nombre_paquete1 nombre_paquete2 ...
```

En RedHat:

```bash
sudo yum install nombre_paquete ...
```

```bash
sudo dnf install paquete1 ...
```

### Actualizar

En Debian (debe ir precedido obligatoriamente de un `apt update`):

```bash
sudo apt upgrade
```

```bash
sudo apt upgrade nombre_paquete
```

En RedHat:

```bash
sudo yum update
```

```bash
sudo dnf update
```

### Eliminar

En Debian:

```bash
sudo apt remove nombre_paquete
```

En RedHat:

```bash
sudo yum remove nombre_paquete
```

```bash
sudo dnf remove nombre_paquete
```

> [!TIP]
> `apt remove` deja los archivos de configuración en su lugar; `apt purge` los elimina también.
> `sudo apt autoremove` limpia las dependencias que se volvieron innecesarias.

### Buscar

Operación equivalente a buscar una aplicación en una tienda.

En Debian:

```bash
apt search nombre_paquete
```

En RedHat:

```bash
yum search nombre_paquete
```

```bash
dnf search nombre_paquete
```

### Tabla de correspondencia

| Acción | Debian (`apt`) | RedHat (`dnf`) |
|--------|----------------|----------------|
| Actualizar el índice | `apt update` | *(automático)* |
| Instalar | `apt install pkg` | `dnf install pkg` |
| Actualizar | `apt upgrade` | `dnf update` |
| Eliminar | `apt remove pkg` | `dnf remove pkg` |
| Buscar | `apt search pkg` | `dnf search pkg` |
| Información de un paquete | `apt show pkg` | `dnf info pkg` |
| Listar instalados | `apt list --installed` | `dnf list installed` |

---

## Gestión de servicios

Los servicios son programas que se ejecutan en segundo plano en la máquina (el servidor).

Algunos ejemplos:

- Servidor HTTP/HTTPS
- Servidor SSH
- Servidor de base de datos
- …

### Los comandos systemctl

| Comando | Efecto |
|---------|--------|
| `sudo systemctl enable nombre` | **Activar**: iniciar el servicio al arrancar la máquina |
| `sudo systemctl disable nombre` | Desactivar el inicio automático |
| `sudo systemctl start nombre` | Iniciar el servicio **ahora** |
| `sudo systemctl stop nombre` | Detener el servicio |
| `sudo systemctl restart nombre` | Reiniciar el servicio |
| `sudo systemctl reload nombre` | Recargar su configuración **sin** detener el servicio |
| `sudo systemctl status nombre` | Inspeccionar el estado actual del servicio |

```bash
sudo systemctl enable nombre_servicio
```

```bash
sudo systemctl start nombre_servicio
```

```bash
sudo systemctl status nombre_servicio
```

> [!IMPORTANT]
> `enable` ≠ `start`. `enable` programa el inicio **en el próximo arranque**, `start` inicia **ahora**.
> Para hacer ambas cosas de una vez: `sudo systemctl enable --now nombre_servicio`.

### Listar servicios

```bash
systemctl list-units --type=service
```

```bash
systemctl list-units --type=service --state=running
```

```bash
systemctl list-units --type=service --state=active
```

Listar todos los servicios en error:

```bash
systemctl list-units --type=service --state=failed
```

---

## Creación de un servicio personalizado

Debes crear un archivo para este servicio (generalmente nombrado `nombre_del_servicio.service`), en `/etc/systemd/system/NOMBRE_SERVICIO.service`:

```ini
[Unit]
Description=Ejemplo de servicio
After=multi-user.target

[Service]
Type=simple
Restart=always
RestartSec=1
User=NOMBRE_USUARIO_SERVICIO
ExecStart=COMANDO_SERVICIO

[Install]
WantedBy=multi-user.target
```

- Reemplaza `NOMBRE_USUARIO_SERVICIO` con el nombre del usuario al que estará vinculado el servicio.
- Reemplaza `COMANDO_SERVICIO` con el comando ejecutado por el servicio.

> [!WARNING]
> **Siempre usa rutas absolutas** de los comandos y archivos: `systemd` no tiene el mismo `PATH` que tu shell.
> Verifica también que el usuario del servicio tenga derecho de ejecutar este comando y de acceder a los archivos concernidos.

Encontrar la ruta absoluta de un comando:

```bash
type -a COMANDO
```

### Los targets disponibles

| target de systemd | Runlevel SystemV | Alias | Descripción |
|-------------------|------------------|-------|-------------|
| `default.target` | | | Siempre un enlace simbólico hacia `multi-user.target` o `graphical.target`. Es lo que systemd usa para arrancar el sistema. Nunca lo enlaces hacia `halt`, `poweroff` o `reboot`. |
| `graphical.target` | 5 | `runlevel5.target` | `multi-user.target` con una interfaz gráfica |
| | 4 | `runlevel4.target` | No utilizado. Idéntico al runlevel 3 en SystemV. Puede ser creado y personalizado para iniciar servicios locales sin modificar `multi-user.target`. |
| `multi-user.target` | 3 | `runlevel3.target` | Todos los servicios iniciados, solo línea de comandos |
| | 2 | `runlevel2.target` | Multi-usuario, sin NFS, todos los otros servicios no gráficos iniciados |
| `rescue.target` | 1 | `runlevel1.target` | Sistema básico: sistemas de archivos montados, servicios mínimos y shell de rescate en la consola |
| `emergency.target` | S | | Modo mono-usuario — ningún servicio, ningún sistema de archivos montado. El nivel más básico, con solo un shell de emergencia. |
| `halt.target` | | | Detiene el sistema sin cortar la alimentación |
| `reboot.target` | 6 | `runlevel6.target` | Reinicio |
| `poweroff.target` | 0 | `runlevel0.target` | Detiene el sistema y corta la alimentación |

### Ejemplo completo

```ini
[Unit]
Description=Ejemplo de servicio
After=multi-user.target

[Service]
Type=simple
Restart=always
RestartSec=1
User=debian
ExecStart=/usr/bin/node /home/debian/test.js

[Install]
WantedBy=multi-user.target
```

Después queda recargar systemd, luego activar e iniciar el servicio:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now mi_servicio
```

> [!IMPORTANT]
> El `daemon-reload` es indispensable después de **cualquier** creación o modificación de un archivo `.service`: sin él, systemd continúa usando la versión anterior en memoria.

---

## Journald

`journald` es la herramienta de registro de Linux bajo systemd. Su uso es relativamente simple:

```bash
journalctl -u nombre_servicio
```

Ejemplo:

```bash
journalctl -u apache2
```

| Opción | Efecto |
|--------|--------|
| `-u nombre` | Filtrar en una **unidad** systemd (el caso más común) |
| `-t nombre` | Filtrar en un *identificador syslog* (el nombre que el programa da a sus logs) |
| `-f` | Seguir los logs en directo |
| `-n 50` | Mostrar las últimas 50 líneas |
| `--since "1 hour ago"` | Filtrar por fecha |
| `-p err` | Mantener solo los mensajes de un nivel de prioridad dado |

---

## CRON

Cron permite lanzar tareas automáticamente a momentos precisos.

Puede necesitar instalación:

```bash
sudo apt install cron
```

Cron es en sí mismo un servicio:

```bash
# Debian
systemctl status cron

# RedHat
systemctl status crond
```

### Administrar su crontab

| Comando | Efecto |
|---------|--------|
| `crontab -e` | Editar sus tareas planificadas |
| `crontab -l` | Listar sus tareas planificadas |
| `crontab -r` | **Eliminar** todo su crontab |

> [!CAUTION]
> `crontab -r` elimina la totalidad del crontab sin pedir confirmación — y `-r` está justo al lado de `-e` en el teclado. Guarda una copia de tus crontabs: `crontab -l > ~/crontab.bak`.

### Sintaxis de un crontab

```text
# Ejemplo de definición de tarea:
# .---------------- minuto (0 - 59)
# |  .------------- hora (0 - 23)
# |  |  .---------- día del mes (1 - 31)
# |  |  |  .------- mes (1 - 12) O ene,feb,mar,abr ...
# |  |  |  |  .---- día de la semana (0 - 6) (domingo=0 o 7) O dom,lun,mar,mié,jue,vie,sab
# |  |  |  |  |
# *  *  *  *  * comando a ejecutar
```

Por ejemplo, para ejecutar el comando `ls` cada minuto:

```text
* * * * * /usr/bin/ls
```

Cada hora (a las XXh00):

```text
0 * * * * /usr/bin/ls
```

### Expresiones comunes

| Expresión CRON | Significado |
|-----------------|-------------|
| `* * * * *`   | Cada minuto |
| `0 * * * *`   | Cada hora (XXh00) |
| `30 2 * * *`  | Cada día a las 2h30 |
| `0 8 * * 1-5` | De lunes a viernes a las 8h00 |
| `*/5 * * * *` | Cada 5 minutos |
| `0 */2 * * *` | Cada 2 horas |
| `0 0 1 1 *`   | Cada 1 de enero |
| `@reboot`     | En cada arranque |
| `@yearly`     | ⇒ `0 0 1 1 *` |
| `@monthly`    | Cada mes |
| `@weekly`     | Cada semana |
| `@daily`      | Cada día |
| `@hourly`     | Cada hora |

### El crontab del sistema

Existe también un crontab del sistema, en `/etc/crontab` (que no se debe modificar en principio: los crontabs por usuario son suficientes).

Este tiene un **campo adicional**: el usuario que ejecutará la tarea.

```bash
cat /etc/crontab
```

```text
# /etc/crontab: crontab de todo el sistema
# A diferencia de cualquier otro crontab no tienes que ejecutar el comando `crontab'
# para instalar la nueva versión cuando editas este archivo
# y los archivos en /etc/cron.d. Estos archivos también tienen campos de nombre de usuario,
# que ninguno de los otros crontabs tienen.

SHELL=/bin/sh
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin

# Ejemplo de definición de tarea:
# .---------------- minuto (0 - 59)
# |  .------------- hora (0 - 23)
# |  |  .---------- día del mes (1 - 31)
# |  |  |  .------- mes (1 - 12) O ene,feb,mar,abr ...
# |  |  |  |  .---- día de la semana (0 - 6) (domingo=0 o 7) O dom,lun,mar,mié,jue,vie,sab
# |  |  |  |  |
# *  *  *  *  * nombre-usuario comando a ejecutar
17 *	* * *	root	cd / && run-parts --report /etc/cron.hourly
25 6	* * *	root	test -x /usr/sbin/anacron || { cd / && run-parts --report /etc/cron.daily; }
47 6	* * 7	root	test -x /usr/sbin/anacron || { cd / && run-parts --report /etc/cron.weekly; }
52 6	1 * *	root	test -x /usr/sbin/anacron || { cd / && run-parts --report /etc/cron.monthly; }
#
```

### Administrar las salidas de las tareas cron

Por defecto, en ciertos sistemas (RedHat especialmente), cron envía la salida de las tareas **por correo** (si un MTA está disponible). Por lo tanto, es interesante redirigir explícitamente la salida de los comandos lanzados vía cron:

```text
* * * * * /bin/ls > /dev/null 2>&1
```

En este ejemplo, redirijo la salida hacia `/dev/null` y redirijo los errores hacia la salida estándar (por lo tanto, ella también, hacia `/dev/null`).

En la mayoría de los casos, se redirige más bien hacia un archivo de log:

```text
* * * * * /bin/ls >> /tmp/customlog
```

```text
* * * * * /bin/ls >> /home/debian/cronlogs/customlog
```

### Ver los logs de cron

```bash
sudo journalctl -t CRON
```

O vía syslog (**requiere `rsyslog`**):

```bash
sudo apt install rsyslog
```

```bash
cat /var/log/syslog | grep CRON
```

```text
2025-12-12T09:31:01.451982+00:00 balrog-c2-srv CRON[2708800]: (debian) CMD (/usr/bin/echo "Hello" >> /home/debian/hello.txt)
2025-12-12T09:32:01.459068+00:00 balrog-c2-srv CRON[2708812]: (debian) CMD (/usr/bin/echo "Hello" >> /home/debian/hello.txt)
2025-12-12T09:33:01.465445+00:00 balrog-c2-srv CRON[2708824]: (debian) CMD (/usr/bin/echo "Hello" >> /home/debian/hello.txt)
```

> [!IMPORTANT]
> Estos logs muestran que el comando **fue lanzado**, pero no lo que produjo. Por lo tanto, debes administrar tú mismo los logs de salida de tus scripts.

### Buenas prácticas

> [!TIP]
> - Siempre usar **rutas absolutas** (cron tiene un `PATH` mínimo y no ejecuta tu `.bashrc`).
> - Siempre **probar el script manualmente** antes de ponerlo en un cron.
> - **Redirigir las salidas** para facilitar el debugging.
> - **Documentar** tus crons (un comentario arriba de cada línea).
> - **Revisar regularmente** los logs de ejecución de tus crons.

---

## Resumen

| Comando | Rol |
|---------|-----|
| `apt update` / `apt upgrade` | Actualizar el índice / los paquetes |
| `apt install` / `apt remove` | Instalar / eliminar un paquete |
| `systemctl start` / `stop` / `restart` | Controlar un servicio |
| `systemctl enable --now` | Activar al arranque **y** iniciar |
| `systemctl status` | Estado de un servicio |
| `systemctl daemon-reload` | Recargar después de modificar un `.service` |
| `journalctl -u nombre -f` | Seguir los logs de un servicio |
| `crontab -e` / `-l` | Editar / listar tus tareas planificadas |

---

⬅️ [Anterior: 03 · Comandos esenciales](03_commandes_essentielles_en.md) · 🏠 [Tabla de contenidos](README.md) · [Siguiente: 05 · Seguridad ➡️](05_securite.md)
