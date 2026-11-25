# 01 · Los fundamentos de la terminal Linux

> [!NOTE]
> **Objetivos del capítulo**
> - Navegar por una estructura de directorios desde la terminal
> - Entender la organización del sistema de archivos Linux
> - Crear, copiar, mover y eliminar archivos y carpetas
> - Redirigir las salidas de un comando y encadenarlos entre sí
> - Leer y editar un archivo con `nano` y `vim`
> - Inspeccionar y detener los procesos en ejecución

## Tabla de contenidos

1. [Navegación en la terminal](#navegación-en-la-terminal)
2. [El sistema de archivos Linux](#el-sistema-de-archivos-linux)
3. [Manipulación de archivos y carpetas](#manipulación-de-archivos-y-carpetas)
4. [Redirección de salidas](#redirección-de-salidas)
5. [Mostrar el contenido de un archivo](#mostrar-el-contenido-de-un-archivo)
6. [Editar un archivo](#editar-un-archivo)
7. [Redirección con EOF (heredoc)](#redirección-con-eof-heredoc)
8. [Encadenar comandos con el pipe](#encadenar-comandos-con-el-pipe)
9. [Gestión de procesos](#gestión-de-procesos)
10. [Resumen](#resumen)

---

## Navegación en la terminal

Mostrar la ubicación actual en la terminal (**p**rint **w**orking **d**irectory):

```bash
pwd
```

Desplazarse a una carpeta:

```bash
cd NombreDeLaCarpeta
```

Desplazarse a la carpeta padre:

```bash
cd ..
```

Desplazarse a una carpeta ubicada en la carpeta padre:

```bash
cd ../NombreDeLaCarpeta
```

> [!TIP]
> Algunos atajos de rutas que aparecen constantemente:
>
> | Atajo | Significado |
> |-----------|---------------|
> | `.`  | La carpeta actual |
> | `..` | La carpeta padre |
> | `~`  | El *home* del usuario actual (`/home/username`) |
> | `-`  | La carpeta anterior (`cd -` hace el viaje de ida y vuelta) |
> | `/`  | La raíz del sistema |

---

## El sistema de archivos Linux

En Linux, **todo parte de una única raíz notada `/`**. No hay unidades como `C:` o `D:` como en Windows: los discos, las unidades USB e incluso ciertos periféricos están "conectados" (*montados*) en algún lugar de este único árbol.

Bajo esta raíz, siempre se encuentran más o menos las mismas carpetas, cada una con un rol bien definido (es el estándar *FHS — Filesystem Hierarchy Standard*).

```
/
├── bin    → comandos básicos (ls, cp, cat…)
├── sbin   → comandos de administración (reservados a root)
├── boot   → kernel de Linux y archivos de inicio
├── etc    → archivos de configuración del sistema
├── home   → carpetas personales de los usuarios
├── root   → carpeta personal del administrador (root)
├── lib    → bibliotecas compartidas por los programas
├── usr    → programas y datos instalados (la mayor parte del sistema)
├── var    → datos que varían: logs, colas, cachés…
├── tmp    → archivos temporales (eliminados en el reinicio)
├── opt    → software de terceros instalado "aparte"
├── mnt    → punto de montaje manual (discos, recursos compartidos en red)
├── media  → montaje automático de medios extraíbles (USB, CD…)
├── dev    → periféricos vistos como archivos (discos, terminales…)
├── proc   → sistema de archivos virtual: estado de procesos y kernel
└── sys    → sistema de archivos virtual: hardware y parámetros del kernel
```

| Carpeta | Rol | Ejemplo concreto |
|---------|------|-----------------|
| `/bin`  | Comandos esenciales, disponibles para todos | `/bin/ls`, `/bin/cp` |
| `/sbin` | Comandos de administración del sistema | `/sbin/reboot`, `/sbin/fdisk` |
| `/boot` | Kernel e inicio (bootloader) | `/boot/vmlinuz`, GRUB |
| `/etc`  | **Toda la configuración** del sistema | `/etc/passwd`, `/etc/ssh/sshd_config` |
| `/home` | Archivos personales de los usuarios | `/home/alice`, `/home/bob` |
| `/root` | *Home* del usuario root | `/root` |
| `/usr`  | Programas instalados y sus recursos | `/usr/bin`, `/usr/share` |
| `/var`  | Datos que crecen con el tiempo | `/var/log`, `/var/www` |
| `/tmp`  | Archivos temporales, desechables | archivos de un instalador |
| `/dev`  | Periféricos de hardware vistos como archivos | `/dev/sda`, `/dev/null` |
| `/proc` | Vista **virtual** de procesos y kernel | `/proc/cpuinfo`, `/proc/1234` |
| `/sys`  | Vista **virtual** del hardware y kernel | `/sys/class/net` |

> [!NOTE]
> `/proc` y `/sys` no existen en el disco: son generados **sobre la marcha** por el kernel. Leer `/proc/cpuinfo` equivale a pedirle al kernel la información del procesador directamente.

> [!TIP]
> Dos carpetas que hay que saber de memoria en el día a día: **`/etc`** (donde se configura) y **`/var/log`** (donde se mira qué pasó cuando algo no funciona).

> [!CAUTION]
> Las carpetas `/`, `/etc`, `/bin`, `/boot`, `/usr`… pertenecen al sistema. Solo se tocan con `sudo` y sabiendo lo que se hace: una eliminación desafortunada puede inutilizar la máquina.

---

## Manipulación de archivos y carpetas

Crear un archivo:

```bash
touch nombreDelArchivo
```

Eliminar un archivo:

```bash
rm nombreDelArchivo
```

Crear una nueva carpeta:

```bash
mkdir nombreDeLaCarpeta
```

Eliminar una carpeta **vacía**:

```bash
rmdir nombreDeLaCarpeta
```

Eliminar una carpeta **y todo su contenido**:

```bash
rm -Rf nombreDeLaCarpeta
```

> [!CAUTION]
> `rm -Rf` elimina **sin confirmación y sin papelera**: no hay vuelta atrás.
> Siempre verifica la ruta antes de confirmar, especialmente si comienza con `/` o contiene una variable — `rm -Rf "$CARPETA/"` con `CARPETA` vacía apunta a la raíz.

Mover un archivo o una carpeta:

```bash
mv ./origen nueva/ubicacion/
```

Este comando mueve el archivo `origen` (ubicado en la carpeta actual) a `./nueva/ubicacion/origen`.

Renombrar un archivo o una carpeta — es el mismo comando, se "mueve" a un nuevo nombre:

```bash
mv nombreAnterior nombreNuevo
```

Copiar un archivo:

```bash
cp origen copia
```

Crea un nuevo archivo llamado `copia` con el mismo contenido que `origen`.

Copiar una carpeta (`-R` = recursivo, es decir, con su contenido):

```bash
cp -R origen/ nuevaCarpeta/
```

---

## Redirección de salidas

Redirigir la salida de un comando a un archivo:

```bash
echo "Hello" > ejemplo.txt
```

o

```bash
echo "Hello" >> ejemplo.txt
```

La diferencia entre `>` y `>>`:

| Operador | Comportamiento |
|-----------|--------------|
| `>`  | **Sobrescribe** el contenido del archivo (y lo crea si no existe) |
| `>>` | **Añade** al final del archivo |

Esto funciona con cualquier comando:

```bash
ls >> archivos.txt
```

También puedo redirigir **solo los errores**:

```bash
ls 2> errores.txt
```

> [!NOTE]
> Cada programa tiene tres flujos estándar. El número colocado antes de `>` designa el flujo a redirigir:
>
> | Flujo | Número | Rol |
> |------|--------|------|
> | `stdin`  | `0` | Entrada estándar (el teclado por defecto) |
> | `stdout` | `1` | Salida estándar (`>` es un atajo para `1>`) |
> | `stderr` | `2` | Salida de error |
>
> Redirigir todo al mismo lugar: `comando > salida.log 2>&1`
> Tirar todo a la basura: `comando > /dev/null 2>&1`

---

## Mostrar el contenido de un archivo

Mostrar todo el archivo:

```bash
cat nombreDelArchivo.txt
```

Mostrar las 10 primeras líneas:

```bash
head nombreDelArchivo
```

Mostrar las 10 últimas líneas:

```bash
tail nombreDelArchivo
```

Mostrar las primeras/últimas `n` líneas (aquí 5):

```bash
head -n 5 nombreDelArchivo
```

```bash
tail -n 5 nombreDelArchivo
```

> [!TIP]
> `tail -f archivo.log` sigue el archivo **en directo**: las nuevas líneas se muestran a medida que se escriben. Esencial para monitorear logs.
> Para recorrer un archivo largo: `less archivo` (flechas para desplazarse, `/` para buscar, `q` para salir).

---

## Editar un archivo

Dos editores están disponibles nativamente en la mayoría de distribuciones Linux.

### nano — el más simple

```bash
nano nombreDelArchivo
```

Atajos útiles:

| Atajo | Acción |
|-----------|--------|
| `CTRL + O` | Guardar |
| `CTRL + X` | Salir |
| `CTRL + K` | Cortar una línea |
| `CTRL + U` | Pegar la línea cortada |
| `CTRL + W` | Buscar texto |
| `CTRL + \` | Reemplazar texto |

### vim — el más poderoso

```bash
vim nombreDelArchivo
```

Tres modos importantes:

| Modo | Cómo entrar | Para qué sirve |
|------|------------------|----------------|
| **Normal** | `ESC` (modo por defecto) | Desplazarse, eliminar, copiar |
| **Inserción** | `i` | Escribir texto |
| **Comando** | `:` | Guardar, salir, buscar/reemplazar |

**Añadir texto, paso a paso:**

1. Abrir `vim`
2. Presionar `i`
3. Escribir el texto
4. `ESC` para volver al modo normal
5. `:wq` (o `:x`) para guardar y salir

| Comando | Acción |
|----------|--------|
| `:w`  | Guardar |
| `:q`  | Salir |
| `:q!` | Salir **sin** guardar |
| `:wq` / `:x` | Guardar y salir |

> [!IMPORTANT]
> En Debian y otras distribuciones, `vim` no siempre está instalado, pero `vi` (su antecesor) casi siempre lo está.

---

## Redirección con EOF (heredoc)

El *heredoc* permite escribir varias líneas de una vez en un archivo — muy útil en un script para generar un archivo de configuración.

```bash
cat <<EOF >> salida.txt
el contenido que quiero en mi archivo
puedo hacer saltos de línea
etc
...
y cierro el archivo con:
EOF
```

> [!TIP]
> Con `<<EOF`, las variables se interpretan (`$USER` se reemplaza por su valor).
> Con `<<'EOF'` (entre comillas), el texto se escribe **tal cual**, sin interpretar.

---

## Encadenar comandos con el pipe

El `|` (*pipe*) redirige la **salida** de un comando hacia la **entrada** de otro.

Extraer todas las líneas que contienen `sha-256`:

```bash
cat /etc/postgresql/15/main/pg_hba.conf | grep sha-256
```

Ordenar una salida:

```bash
ls / | sort
```

Contar el número de líneas que contienen `sha-256`:

```bash
cat /etc/postgresql/15/main/pg_hba.conf | grep sha-256 | wc -l
```

Se pueden encadenar tantos comandos como se quiera: cada `|` pasa el resultado al siguiente eslabón.

---

## Gestión de procesos

Mostrar los procesos en ejecución:

```bash
ps aux
```

Versión interactiva:

```bash
top
```

O mucho más legible (hay que instalar):

```bash
htop
```

### Buscar un proceso específico

```bash
ps aux | grep node
```

Una vez que tengo el **PID** (*process ID*) del proceso, puedo terminarlo:

```bash
kill -9 PID
```

> [!WARNING]
> `kill -9` (SIGKILL) termina el proceso **bruscamente**: no tiene la oportunidad de cerrar correctamente sus archivos o conexiones.
> Siempre intenta primero `kill PID` (SIGTERM, terminación limpia), y guarda `-9` para los procesos que no responden.


```
       SIGABRT      P1990      Core    Abort signal from abort(3)
       SIGALRM      P1990      Term    Timer signal from alarm(2)
       SIGBUS       P2001      Core    Bus error (bad memory access)
       SIGCHLD      P2001      Ign     Child stopped, terminated, or continued
       SIGCLD         -        Ign     A synonym for SIGCHLD
       SIGCONT      P1990      Cont    Continue if stopped
       SIGEMT         -        Term    Emulator trap
       SIGFPE       P1990      Core    Erroneous arithmetic operation
       SIGHUP       P1990      Term    Hangup detected on controlling terminal
                                       or death of controlling process
       SIGILL       P1990      Core    Illegal Instruction
       SIGINFO        -                A synonym for SIGPWR
       SIGINT       P1990      Term    Interrupt from keyboard
       SIGIO          -        Term    I/O now possible (4.2BSD)
       SIGIOT         -        Core    IOT trap.  A synonym for SIGABRT
       SIGKILL      P1990      Term    Kill signal
       SIGLOST        -        Term    File lock lost (unused)
       SIGPIPE      P1990      Term    Broken pipe: write to pipe with no
                                       readers; see pipe(7)
       SIGPOLL      P2001      Term    Pollable event (Sys V);
                                       synonym for SIGIO
       SIGPROF      P2001      Term    Profiling timer expired
       SIGPWR         -        Term    Power failure (System V)
       SIGQUIT      P1990      Core    Quit from keyboard
       SIGSEGV      P1990      Core    Invalid memory reference
       SIGSTKFLT      -        Term    Stack fault on coprocessor (unused)
       SIGSTOP      P1990      Stop    Stop process
       SIGTSTP      P1990      Stop    Stop typed at terminal
       SIGSYS       P2001      Core    Bad system call (SVr4);
                                       see also seccomp(2)
       SIGTERM      P1990      Term    Termination signal
       SIGTRAP      P2001      Core    Trace/breakpoint trap
       SIGTTIN      P1990      Stop    Terminal input for background process
       SIGTTOU      P1990      Stop    Terminal output for background process
       SIGUNUSED      -        Core    Synonymous with SIGSYS
       SIGURG       P2001      Ign     Urgent condition on socket (4.2BSD)
       SIGUSR1      P1990      Term    User-defined signal 1
       SIGUSR2      P1990      Term    User-defined signal 2
       SIGVTALRM    P2001      Term    Virtual alarm clock (4.2BSD)
       SIGXCPU      P2001      Core    CPU time limit exceeded (4.2BSD);
                                       see setrlimit(2)
       SIGXFSZ      P2001      Core    File size limit exceeded (4.2BSD);
                                       see setrlimit(2)
       SIGWINCH     P2024      Ign     Window resize signal (4.3BSD, Sun)
```
---

## Resumen

| Comando | Rol |
|----------|------|
| `pwd` | Dónde estoy |
| `cd` | Desplazarse |
| `ls` | Listar el contenido de una carpeta |
| `touch` / `mkdir` | Crear un archivo / una carpeta |
| `rm` / `rmdir` | Eliminar un archivo / una carpeta vacía |
| `mv` | Mover o renombrar |
| `cp` | Copiar |
| `cat` / `head` / `tail` | Leer un archivo (todo / inicio / fin) |
| `nano` / `vim` | Editar un archivo |
| `>` / `>>` / `2>` | Redirigir la salida / añadir al final / redirigir errores |
| `\|` | Encadenar dos comandos |
| `ps` / `top` / `htop` | Ver los procesos |
| `kill` | Detener un proceso |

> [!TIP]
> En caso de duda sobre un comando: `man comando` (manual completo) o `comando --help` (ayuda rápida).

---

🏠 [Tabla de contenidos](README_es.md) · [Siguiente: 02 · Permisos ➡️](02_file_permissions_es.md)
