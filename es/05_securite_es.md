# 05 · Seguridad

> [!NOTE]
> **Objetivos del capítulo**
> - Ir más allá de los permisos POSIX con las **ACL**
> - Delegar derechos de administración de forma precisa mediante **sudoers**
> - Filtrar el tráfico de red con un **cortafuegos** (`ufw`, `firewalld`, `iptables`, `nftables`)
> - Configurar, reforzar y diagnosticar un acceso **SSH** por claves
> - Confinar los servicios con el control de acceso obligatorio (**AppArmor / SELinux**)

Este capítulo cubre cuatro capas que se superponen en un servidor Linux. No se reemplazan entre sí, se apilan:

```text
┌──────────────────────────────────────────────────────────────┐
│  MAC        AppArmor / SELinux   → lo que un proceso puede    │
│                                    hacer, incluso como root   │
├──────────────────────────────────────────────────────────────┤
│  Privilegios sudo / sudoers       → quién puede convertirse   │
│                                     en quién                  │
├──────────────────────────────────────────────────────────────┤
│  Archivos   POSIX + ACL          → quién puede leer qué       │
├──────────────────────────────────────────────────────────────┤
│  Red        ufw / nftables       → quién puede alcanzar la    │
│             SSH                    máquina, y cómo            │
└──────────────────────────────────────────────────────────────┘
```

## Contenido

1. [ACL (Access Control Lists)](#acl-access-control-lists)
   - [Requisitos previos y verificaciones](#requisitos-previos-y-verificaciones)
   - [Leer una ACL en detalle](#leer-una-acl-en-detalle)
   - [Modificar las ACL](#modificar-las-acl)
   - [La máscara, en detalle](#la-máscara-en-detalle)
   - [ACL por defecto y herencia](#acl-por-defecto-y-herencia)
   - [Caso práctico: una carpeta de equipo](#caso-práctico-una-carpeta-de-equipo)
   - [Guardar, copiar y migrar ACL](#guardar-copiar-y-migrar-acl)
   - [Diagnóstico ACL](#diagnóstico-acl)
2. [Sudo / sudoers](#sudo--sudoers)
   - [Uso cotidiano](#uso-cotidiano)
   - [Editar sudoers sin bloquearse](#editar-sudoers-sin-bloquearse)
   - [Anatomía del archivo](#anatomía-del-archivo)
   - [La sintaxis de una regla](#la-sintaxis-de-una-regla)
   - [Las etiquetas](#las-etiquetas)
   - [Los Defaults](#los-defaults)
   - [Los alias](#los-alias)
   - [sudoedit — editar un archivo protegido, correctamente](#sudoedit--editar-un-archivo-protegido-correctamente)
   - [Casos prácticos completos](#casos-prácticos-completos)
   - [Las trampas de seguridad](#las-trampas-de-seguridad)
   - [Auditoría y registro](#auditoría-y-registro)
   - [Diagnóstico sudo](#diagnóstico-sudo)
3. [Cortafuegos](#cortafuegos)
   - [Los conceptos comunes](#los-conceptos-comunes)
   - [ufw (Debian, Ubuntu)](#ufw-debian-ubuntu)
   - [firewalld (RHEL, Fedora)](#firewalld-rhel-fedora)
   - [iptables](#iptables)
   - [nftables](#nftables)
   - [Probar y diagnosticar](#probar-y-diagnosticar-un-cortafuegos)
4. [SSH](#ssh)
   - [Instalación y verificación](#instalación-y-verificación)
   - [Los dos archivos de configuración](#los-dos-archivos-de-configuración)
   - [Gestión de claves](#gestión-de-claves)
   - [Desplegar su clave pública](#desplegar-su-clave-pública)
   - [Reforzar el servidor: sshd_config](#reforzar-el-servidor-sshd_config)
   - [Los bloques Match](#los-bloques-match)
   - [Restringir una clave en authorized_keys](#restringir-una-clave-en-authorized_keys)
   - [Configurar el cliente: ~/.ssh/config](#configurar-el-cliente-sshconfig)
   - [known_hosts y huellas dactilares](#known_hosts-y-huellas-dactilares)
   - [El agente SSH](#el-agente-ssh)
   - [Túneles y rebotes](#túneles-y-rebotes)
   - [Transferir archivos](#transferir-archivos)
   - [fail2ban](#fail2ban)
   - [Diagnóstico SSH](#diagnóstico-ssh)
5. [AppArmor / SELinux](#apparmor--selinux)
   - [AppArmor (Debian, Ubuntu, SUSE)](#apparmor-debian-ubuntu-suse)
   - [SELinux (RHEL, Fedora, CentOS, Rocky)](#selinux-rhel-fedora-centos-rocky)
   - [Método de diagnóstico](#método-de-diagnóstico)
6. [Resumen](#resumen)

---

## ACL (Access Control Lists)

Los permisos POSIX solo conocen tres destinos:

```text
 Propietario   Grupo   Otros
 rwx           rwx     rwx
```

Esto es suficiente mientras las necesidades sean simples. Pero tan pronto como se quiere *« Alice puede escribir, Bob solo puede leer, el equipo dev puede leer, y nadie más ve nada »*, sería necesario crear un grupo por cada combinación de derechos. Ese es exactamente el problema que resuelven las **ACL**: conceder derechos a **un usuario específico** o a **un grupo específico**, además del propietario y del grupo propietario.

### Requisitos previos y verificaciones

**1. Las herramientas** — no siempre están instaladas:

```bash
sudo apt install acl          # Debian / Ubuntu
sudo dnf install acl          # RHEL / Fedora
```

**2. El sistema de archivos debe soportar ACL.** Esto es el caso por defecto de ext4 y XFS en distribuciones modernas, pero se debe verificar:

```bash
# Para ext2/3/4
sudo tune2fs -l /dev/sda1 | grep "Default mount options"
```

```text
Default mount options:    user_xattr acl
```

```bash
# Verificar las opciones de montaje efectivas
mount | grep " / "
findmnt -o TARGET,SOURCE,FSTYPE,OPTIONS /
```

Si falta la opción `acl`, hay que añadirla en `/etc/fstab`:

```text
# /etc/fstab
UUID=xxxx-xxxx  /srv  ext4  defaults,acl  0  2
```

```bash
sudo mount -o remount /srv
```

**3. Prueba rápida** — si el sistema de archivos no soporta ACL, `setfacl` falla explícitamente:

```bash
setfacl -m u:nobody:r /tmp/test.txt
```

```text
setfacl: /tmp/test.txt: Operation not supported
```

### Leer una ACL en detalle

```bash
getfacl hello.txt
```

```text
# file: hello.txt              ← el archivo en cuestión
# owner: debian                ← propietario POSIX
# group: debian                ← grupo propietario POSIX
user::rw-                      ← derechos del propietario (= la "u" de ls -l)
user:alice:rw-                 ← ENTRADA NOMBRADA: derechos específicos de alice
group::r--                     ← derechos del grupo propietario (= la "g" de ls -l)
group:dev:r-x                  ← ENTRADA NOMBRADA: derechos específicos del grupo dev
mask::rwx                      ← techo aplicado a las entradas nombradas y al grupo
other::r--                     ← derechos de otros (= la "o" de ls -l)
```

| Tipo de línea | Significado |
|---------------|------|
| `user::` | El propietario — **nunca** limitado por la máscara |
| `user:nombre:` | Un usuario nombrado — sujeto a la máscara |
| `group::` | El grupo propietario — sujeto a la máscara |
| `group:nombre:` | Un grupo nombrado — sujeto a la máscara |
| `mask::` | El techo de los derechos efectivos |
| `other::` | Todos los demás — **nunca** limitado por la máscara |
| `default:…` | Una ACL heredada por los elementos creados en la carpeta |

### Detectar que un archivo tiene ACL

```bash
ls -la hello.txt
```

```text
-rw-r-xr--+ 1 debian debian   378 Dec 12 10:29 hello.txt
```

> [!TIP]
> El pequeño **`+`** al final de los permisos señala derechos adicionales a los POSIX clásicos. Es el **único** indicio visible en un `ls` — de ahí la importancia de verificar con `getfacl` cuando un archivo se comporta « de manera extraña ».
>
> Trampa de lectura adicional: cuando hay una ACL presente, la columna de derechos del grupo en `ls -l` ya no muestra los derechos del grupo propietario, sino **la máscara**. Un `ls` puede anunciar entonces `rwx` donde el grupo solo tiene realmente `r--`.

Encontrar todos los archivos con ACL en un árbol de directorios:

```bash
getfacl -R -s /srv 2>/dev/null | grep "^# file:"
```

### Modificar las ACL

La sintaxis general es:

```text
setfacl [-R] -m  <tipo>:<nombre>:<derechos>  <destino>
         │    │      │      │      │
         │    │      │      │      └─ rwx, rw, r, o - para nada
         │    │      │      └──────── vacío para propietario / grupo propietario
         │    │      └─────────────── u (usuario), g (grupo), m (máscara), o (otros)
         │    └────────────────────── -m modifica, -x retira, -b borra todo
         └─────────────────────────── recursivo
```

| Comando | Efecto |
|----------|-------|
| `setfacl -m u:alice:rw f` | Dar `rw` al usuario alice |
| `setfacl -m g:dev:rx f` | Dar `rx` al grupo dev |
| `setfacl -m o::--- f` | Retirar todo derecho a otros |
| `setfacl -m m:r f` | Fijar la máscara a `r` |
| `setfacl -x u:alice f` | Retirar la entrada de alice |
| `setfacl -b f` | Eliminar **todas** las ACL (volver a POSIX puro) |
| `setfacl -k d/` | Eliminar solo las ACL **por defecto** |
| `setfacl -R -m …` | Aplicar recursivamente |
| `setfacl -d -m u:alice:rw d/` | Establecer una ACL **por defecto** (herencia) |

#### 1. Dar derechos a un usuario

```bash
setfacl -m u:test:rw hello.txt
```

```bash
getfacl hello.txt
```

```text
# file: hello.txt
# owner: debian
# group: debian
user::rw-
user:test:rw-
group::r--
mask::rw-
other::r--
```

#### 2. Dar derechos a un grupo

```bash
setfacl -m g:test:rx hello.txt
```

```text
# file: hello.txt
# owner: debian
# group: debian
user::rw-
group::r--
group:test:r-x
mask::r-x
other::r--
```

#### 3. Múltiples entradas en un comando

Las entradas se separan por comas:

```bash
setfacl -m u:alice:rw,u:bob:r,g:dev:rx rapport.txt
```

#### 4. Aplicar recursivamente

```bash
setfacl -R -m u:test:rX dossier/
```

> [!TIP]
> Note la **`X` mayúscula**: concede el derecho de ejecución **solo a las carpetas** (y a los archivos que ya son ejecutables). Esto es exactamente lo que se quiere en modo recursivo — sin él, `-R -m u:test:rx` haría ejecutables todos los archivos de datos del árbol.

#### 5. Retirar derechos

```bash
setfacl -x u:test hello.txt      # retirar una entrada específica
setfacl -b hello.txt             # borrar todo, volver a POSIX puro
```

### La máscara, en detalle

Este es el punto que confunde a todo el mundo, y la primera causa de ACL que « no funcionan ».

La `máscara` es un **techo** aplicado a todas las entradas nombradas (`user:nombre`, `group:nombre`) **y** al grupo propietario. **No** se aplica ni al propietario (`user::`) ni a otros (`other::`).

```text
Derecho efectivo  =  derecho concedido  Y  máscara
```

```bash
setfacl -m u:test:rwx test.sh
setfacl -m m:r test.sh
getfacl test.sh
```

```text
# file: test.sh
# owner: debian
# group: debian
user::rwx
user:test:rwx                   #effective:r--     ← la máscara decidió
group::r-x                      #effective:r--
mask::r--
other::---
```

`getfacl` es explícito: muestra `#effective:` junto a cada entrada limitada por la máscara. El usuario `test` aunque tenga `rwx` inscrito, solo puede **leer**.

> [!CAUTION]
> **`chmod` recalcula la máscara.** Este es el piège clásico, y es silencioso:
>
> ```bash
> setfacl -m u:alice:rwx proyecto/     # alice tiene rwx
> chmod g-w proyecto/                  # creemos solo tocar el grupo...
> getfacl proyecto/                    # ...pero la máscara pasó a r-x
> ```
>
> `chmod` sobre los derechos del **grupo** modifica en realidad la **máscara** cuando hay ACL presentes. Los derechos de alice caen con él. Después de cualquier `chmod` en un archivo con ACL, verifique la máscara:
>
> ```bash
> setfacl -m m:rwx proyecto/           # restaurar el techo
> ```

Por defecto, `setfacl` recalcula automáticamente la máscara para que incluya todas las entradas. Para evitarlo:

```bash
setfacl -n -m u:alice:rwx archivo    # -n: no recalcular la máscara
```

### ACL por defecto y herencia

Las ACL por defecto (`default:`, o la opción `-d`) se aplican **solo a las carpetas**. No dan ningún derecho sobre la carpeta misma: definen las ACL que recibirán automáticamente los elementos **creados en su interior**.

```bash
setfacl -d -m u:test:rw compartido/
setfacl -d -m g:dev:rwx compartido/
```

O con la sintaxis explícita:

```bash
setfacl -m d:u:test:rw compartido/
setfacl -m d:g:dev:rwx compartido/
```

```bash
getfacl compartido/
```

```text
# file: compartido
# owner: debian
# group: dev
user::rwx
group::r-x
other::---
default:user::rwx
default:user:test:rw-
default:group::r-x
default:group:dev:rwx
default:mask::rwx
default:other::---
```

Verifiquemos la herencia:

```bash
touch compartido/nuevo.txt
getfacl compartido/nuevo.txt
```

```text
# file: compartido/nuevo.txt
# owner: debian
# group: dev
user::rw-
user:test:rw-
group::r-x                      #effective:r--
group:dev:rwx                   #effective:rw-
mask::rw-
other::---
```

> [!IMPORTANT]
> Tres puntos a conocer sobre la herencia:
> 1. Se aplica **solo a los nuevos** archivos. Los archivos ya presentes no se modifican — hay que hacer un `setfacl -R` para ellos.
> 2. Una **subcarpeta** creada hereda tanto las ACL de acceso **como** las ACL por defecto: la herencia se propaga en profundidad.
> 3. Un archivo nunca obtiene el derecho `x` por herencia si no se crea ejecutable; la máscara se ajusta en consecuencia (`rwx` por defecto se convierte en `rw-` efectivo arriba).

Eliminar las ACL por defecto de una carpeta:

```bash
setfacl -k compartido/
```

### Caso práctico: una carpeta de equipo

Este es *el* caso de uso de ACL. El objetivo:

> Una carpeta `/srv/proyecto` donde el equipo **dev** trabaja en conjunto. Todo archivo creado por cualquiera debe ser inmediatamente modificable por todo el equipo. El equipo **qa** puede leer. Nadie más ve nada.

```bash
# 1. Los grupos
sudo groupadd dev
sudo groupadd qa
sudo usermod -aG dev alice
sudo usermod -aG dev bob
sudo usermod -aG qa carol

# 2. La carpeta, propiedad del grupo dev
sudo mkdir -p /srv/proyecto
sudo chgrp dev /srv/proyecto
sudo chmod 2770 /srv/proyecto
```

El `2` de `2770` es el **SGID**: todo archivo creado en la carpeta pertenecerá al grupo `dev` y no al grupo primario de su creador. Es el complemento indispensable de las ACL por defecto (ver el [capítulo 02](02_file_permissions_es.md#sgid-en-una-carpeta)).

```bash
# 3. Las ACL de acceso: lo que cada uno puede hacer en la carpeta misma
sudo setfacl -m g:dev:rwx /srv/proyecto
sudo setfacl -m g:qa:rx  /srv/proyecto

# 4. Las ACL por defecto: de lo que heredarán los nuevos archivos
sudo setfacl -d -m g:dev:rwx /srv/proyecto
sudo setfacl -d -m g:qa:rx   /srv/proyecto
sudo setfacl -d -m o::---    /srv/proyecto
```

Verificación:

```bash
getfacl /srv/proyecto
```

```text
# file: srv/proyecto
# owner: root
# group: dev
# flags: -s-                    ← el SGID está bien puesto
user::rwx
group::rwx
group:dev:rwx
group:qa:r-x
mask::rwx
other::---
default:user::rwx
default:group::rwx
default:group:dev:rwx
default:group:qa:r-x
default:mask::rwx
default:other::---
```

Prueba en tamaño real:

```bash
sudo -u alice touch /srv/proyecto/archivo_alice.txt
getfacl /srv/proyecto/archivo_alice.txt
```

```text
# file: srv/proyecto/archivo_alice.txt
# owner: alice
# group: dev                     ← gracias al SGID
user::rw-
group::rwx                       #effective:rw-
group:dev:rwx                    #effective:rw-   ← bob puede modificar
group:qa:r-x                     #effective:r--   ← carol puede leer
mask::rw-
other::---                        ← nadie más
```

### Guardar, copiar y migrar ACL

**Guardar las ACL de un árbol de directorios** — hay que hacerlo antes de cualquier manipulación arriesgada:

```bash
getfacl -R /srv/proyecto > /root/acl-proyecto.bak
```

**Restaurar**:

```bash
setfacl --restore=/root/acl-proyecto.bak
```

**Copiar las ACL de un archivo a otro**:

```bash
getfacl source.txt | setfacl --set-file=- destination.txt
```

**Preservar las ACL durante copias y copias de seguridad** — las herramientas no lo hacen *por defecto*:

| Herramienta | Opción requerida |
|-------|----------------|
| `cp` | `cp -p` o `cp --preserve=all` |
| `rsync` | `rsync -A` (y `-X` para atributos extendidos) |
| `tar` | `tar --acls` |
| `mv` | Nada que hacer — las ACL siguen al archivo en el mismo sistema de archivos |

```bash
rsync -avAX /srv/proyecto/ /mnt/backup/proyecto/
tar --acls -czvf proyecto.tar.gz /srv/proyecto
```

> [!WARNING]
> Una copia de seguridad hecha sin `-A` / `--acls` restaura archivos solo con permisos POSIX. El día de la restauración, la estructura de derechos cuidadosamente construida ha desaparecido — y nadie se da cuenta hasta que alguien se queja de un acceso denegado.

### Diagnóstico ACL

Una ACL que no produce el efecto esperado casi siempre viene de una de estas cinco causas:

| Síntoma | Causa probable | Verificación |
|----------|----------------|--------------|
| `Operation not supported` | El sistema de archivos no está montado con `acl` | `findmnt -o OPTIONS /srv` |
| El usuario no puede escribir a pesar de `rw` | La **máscara** limita el derecho | `getfacl f \| grep effective` |
| El derecho desapareció después de un `chmod` | `chmod` recalculó la máscara | `setfacl -m m:rwx f` |
| El usuario no puede **alcanzar** el archivo | Falta el derecho `x` en una **carpeta padre** | `namei -l /srv/proyecto/archivo.txt` |
| Los nuevos archivos no heredan | ACL por defecto ausente, o archivos creados antes | `getfacl d/ \| grep default` |

El comando más útil para el cuarto caso:

```bash
namei -l /srv/proyecto/docs/report.txt
```

```text
f: /srv/proyecto/docs/report.txt
 drwxr-xr-x root  root  /
 drwxr-xr-x root  root  srv
 drwxrws---+ root dev   proyecto
 drwx------  root root  docs        ← el bloqueo está aquí
 -rw-rw----+ alice dev  report.txt
```

`namei -l` despliega la ruta carpeta por carpeta y muestra inmediatamente dónde se rompe el acceso. Un derecho perfecto en el archivo no sirve de nada si una carpeta padre prohíbe el tránsito.

### Resumen ACL

| Comando | Efecto |
|----------|-------|
| `getfacl f` | Leer las ACL |
| `getfacl -R d/ > f.bak` | Guardar un árbol |
| `setfacl --restore=f.bak` | Restaurar |
| `setfacl -m u:user:rw f` | Añadir/modificar una entrada de usuario |
| `setfacl -m g:group:rx f` | Añadir/modificar una entrada de grupo |
| `setfacl -R -m u:user:rX d/` | Recursivo, `X` = ejecución solo en carpetas |
| `setfacl -d -m g:dev:rwx d/` | ACL por defecto (herencia) |
| `setfacl -m m:rwx f` | Fijar la máscara |
| `setfacl -x u:user f` | Retirar una entrada |
| `setfacl -b f` / `-k d/` | Borrar todo / borrar las ACL por defecto |
| `namei -l /ruta/archivo` | Encontrar dónde una ruta se bloquea |

---

## Sudo / sudoers

`sudo` (*super user do*) permite ejecutar un comando como otro usuario (`root` por defecto), después de autenticarse con **su propio** contraseña.

Este último punto es lo que lo hace la herramienta central de la administración: a diferencia de `su`, nadie necesita conocer la contraseña de root, cada acción se registra de forma nominativa, y se pueden retirar los derechos de una persona sin cambiar la contraseña de todos.

### Uso cotidiano

```bash
sudo cat /var/log/syslog
```

| Comando | Efecto |
|----------|-------|
| `sudo comando` | Ejecutar como root |
| `sudo -u alice comando` | Ejecutar como otro usuario |
| `sudo -g dev comando` | Ejecutar con otro grupo |
| `sudo -i` | Abrir un shell de login de root (carga el entorno de root) |
| `sudo -s` | Abrir un shell de root preservando el entorno actual |
| `sudo -E comando` | Preservar las variables de entorno |
| `sudo -l` | **Listar lo que tengo derecho a hacer** |
| `sudo -k` | Olvidar la autenticación en caché (re-pedir contraseña) |
| `sudo -v` | Prolongar el caché de autenticación |
| `sudo !!` | Repetir el comando anterior con sudo |

El comando más útil para entender los derechos:

```bash
sudo -l
```

```text
Matching Defaults entries for alice on servidor:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User alice may run the following commands on servidor:
    (root) /usr/bin/systemctl restart apache2
    (root) NOPASSWD: /usr/local/bin/deploy.sh
```

```bash
sudo -l -U bob        # como administrador: listar derechos de otro usuario
```

> [!NOTE]
> Si `/var/log/syslog` no existe, es que `rsyslog` no está instalado:
> ```bash
> sudo apt install rsyslog
> ```

Por defecto, hay que pertenecer al grupo `sudo` (Debian/Ubuntu) o `wheel` (RHEL/Fedora) para poder usar `sudo`.

```bash
sudo usermod -aG sudo alice        # Debian
sudo usermod -aG wheel alice       # RHEL
```

> [!IMPORTANT]
> La adición a un grupo solo toma efecto en la **próxima sesión**. Alice debe desconectarse y reconectarse — o ejecutar `newgrp sudo` — antes de que `sudo` funcione para ella.

### Editar sudoers sin bloquearse

> [!CAUTION]
> **Nunca edite `/etc/sudoers` directamente con un editor.** Un error de sintaxis en este archivo hace que `sudo` sea totalmente inutilizable, y la bloquea de todo acceso administrativo en una máquina donde root no tiene contraseña (caso por defecto de Ubuntu).
>
> ```bash
> sudo visudo
> ```
>
> `visudo` bloquea el archivo contra ediciones concurrentes y **valida la sintaxis antes de guardar**. En caso de error, propone re-editar en lugar de escribir un archivo roto.

Elegir el editor:

```bash
sudo EDITOR=nano visudo
sudo update-alternatives --config editor    # cambiar el valor por defecto en Debian
```

Verificar un archivo sin editarlo:

```bash
sudo visudo -c
```

```text
/etc/sudoers: parsed OK
/etc/sudoers.d/despliegue: parsed OK
```

**La buena práctica: no toque `/etc/sudoers`.** Coloque sus reglas en un archivo dedicado bajo `/etc/sudoers.d/`, lo que las hace legibles, reversibles e implementables por herramientas de gestión de configuración:

```bash
sudo visudo -f /etc/sudoers.d/equipo_web
```

```text
# /etc/sudoers.d/equipo_web
# Gestión por Ansible — no editar a mano
%webops ALL=(root) /usr/bin/systemctl restart nginx, /usr/bin/systemctl reload nginx
```

```bash
sudo chmod 0440 /etc/sudoers.d/equipo_web
```

> [!WARNING]
> Dos reglas sobre `/etc/sudoers.d/`:
> - El archivo debe estar en **0440** y pertenecer a root, sino sudo lo ignora silenciosamente.
> - Los archivos cuyo nombre contiene un `.` o termina en `~` son **ignorados**. Nombre `equipo_web`, no `equipo_web.conf`.

### Anatomía del archivo

```text
# /etc/sudoers

# ── Defaults: el comportamiento global de sudo ──────────────────────
Defaults        env_reset
Defaults        mail_badpass
Defaults        secure_path="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"

# ── Alias: accesos directos nombrados ─────────────────────────────
User_Alias      ADMINS = alice, bob
Cmnd_Alias      SERVICIOS = /usr/bin/systemctl

# ── Reglas de usuario ───────────────────────────────────────────
root            ALL=(ALL:ALL) ALL

# ── Reglas de grupo (prefijadas por %) ────────────────────────────
%sudo           ALL=(ALL:ALL) ALL
%admin          ALL=(ALL) ALL

# ── Inclusión del directorio de fragmentos — SIEMPRE AL FINAL ───────
@includedir /etc/sudoers.d
```

> [!IMPORTANT]
> **El orden importa: la última regla que coincide prevalece.** Si un usuario está cubierto por múltiples reglas, la más baja en el archivo se aplica. Por eso `@includedir /etc/sudoers.d` está en la última línea: sus fragmentos pueden así anular las reglas generales.
>
> (En versiones más antiguas, la directiva se escribe `#includedir /etc/sudoers.d` — y el `#` **no** es un comentario aquí.)

### La sintaxis de una regla

```text
alice    servidor1 = (root:web) NOPASSWD: /usr/bin/systemctl restart nginx
  │         │         │   │        │                    │
  │         │         │   │        │                    └─ COMANDOS autorizados
  │         │         │   │        └─────────────────────── ETIQUETAS (opcional)
  │         │         │   └──────────────────────────────── usuario destino (opcional)
  │         │         └──────────────────────────────────── usuario destino
  │         └────────────────────────────────────────────── HOSTS donde se aplica la regla
  └──────────────────────────────────────────────────────── QUIÉN (% = un grupo)
```

La regla canónica, la del grupo `sudo`:

```text
%sudo   ALL=(ALL:ALL)   ALL
```

Se lee: *« los miembros del grupo `sudo`, desde cualquier host, pueden convertirse en cualquier usuario y cualquier grupo, para ejecutar cualquier comando »*.

#### Restringir la identidad que se puede asumir

```text
test  ALL=(debian)    ALL
```

`test` puede hacer todo, pero solo bajo la identidad de `debian` — no como root:

```bash
sudo -u debian comando     # ✅ autorizado
sudo comando               # ❌ denegado (destino = root)
```

#### Restringir los comandos

```text
test ALL=(root)   /usr/bin/systemctl
```

`test` solo puede usar `systemctl` como root — pero **con cualquier argumento**, incluyendo `systemctl stop firewalld`.

```text
test ALL=(root)   /usr/bin/systemctl restart apache2
```

Aquí, el comando está congelado **con sus argumentos**: solo puede reiniciar apache2.

```text
test ALL=(root)   /usr/bin/systemctl restart apache2, /usr/bin/systemctl status apache2
```

Múltiples comandos se separan por comas.

#### Prohibir un comando

```text
test ALL=(devops) ALL, !/bin/rm
```

`test` puede asumir la identidad de `devops` para todos los comandos, excepto `rm`.

> [!CAUTION]
> **Las listas negras no funcionan.** `!/bin/rm` se rodea en tres segundos:
>
> ```bash
> sudo -u devops cp /bin/rm /tmp/r && sudo -u devops /tmp/r archivo
> sudo -u devops vim -c ':!rm archivo'
> sudo -u devops find . -exec rm {} \;
> ```
>
> Cualquier comando capaz de lanzar un shell, copiar un binario o ejecutar un programa arbitrario anula la restricción. **Use siempre una lista blanca** de comandos explícitos, y verifique que ninguno permita escape.

### Las etiquetas

Las etiquetas modifican el comportamiento de los comandos que siguen, hasta el final de la regla.

| Etiqueta | Efecto |
|-----|-------|
| `NOPASSWD:` | No se pide contraseña |
| `PASSWD:` | Re-pedir contraseña (anula un `NOPASSWD:` anterior) |
| `NOEXEC:` | Impedir que el comando lance otros programas |
| `SETENV:` | Autorizar a pasar variables de entorno |
| `LOG_OUTPUT:` | Registrar la salida del comando |

```text
# El despliegue es automatizado: sin contraseña para este script específico,
# pero el resto de los derechos permanece protegido
deploy ALL=(root) NOPASSWD: /usr/local/bin/deploy.sh
deploy ALL=(root) PASSWD:   /usr/bin/systemctl restart nginx
```

> [!CAUTION]
> **`NOPASSWD` se debe usar con parsimonia.** Cada entrada es una puerta abierta si la cuenta está comprometida: ya no es necesario conocer la contraseña para elevar privilegios.
> Resérvelo a comandos específicos, no desviables, y típicamente a cuentas de servicio no interactivas. Un `NOPASSWD: ALL` es equivalente a dar la contraseña de root.

La etiqueta `NOEXEC` merece ser conocida — neutraliza las evasiones de shell en comandos que las permiten:

```text
alice ALL=(root) NOEXEC: /usr/bin/less /var/log/secure
```

Sin él, `less` permitiría `!sh` y daría un shell de root.

### Los Defaults

Las líneas `Defaults` pilotan el comportamiento de sudo mismo.

```text
# Comportamiento básico
Defaults    env_reset                       # partir de un entorno limpio
Defaults    secure_path="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
Defaults    mail_badpass                    # enviar correo al admin en caso de contraseña incorrecta

# Duración del caché de autenticación, en minutos
Defaults    timestamp_timeout=15            # 0 = pedir cada vez, -1 = nunca
Defaults    passwd_tries=3                  # número de intentos

# Registro
Defaults    logfile="/var/log/sudo.log"
Defaults    log_input, log_output           # registro completo de sesiones
Defaults    iolog_dir="/var/log/sudo-io"

# Comodidad
Defaults    pwfeedback                      # mostrar asteriscos al escribir
Defaults    editor="/usr/bin/vim"           # editor utilizado por sudoedit
Defaults    !visiblepw                      # rechazar si la contraseña se mostraría en texto plano
```

Los `Defaults` pueden ser dirigidos:

```text
Defaults:alice          timestamp_timeout=60      # para la usuaria alice
Defaults>root           !set_logname              # cuando el destino es root
Defaults!/usr/bin/vim   NOEXEC                    # para este comando
Defaults@servidor-web   log_output                # en este host
```

> [!IMPORTANT]
> `env_reset` y `secure_path` son protecciones **esenciales**: evitan que un usuario desvíe un comando sudo manipulando `PATH` o `LD_PRELOAD`. No los desactive para « hacer funcionar » un script — pase la ruta absoluta del comando en su lugar.

### Los alias

Los alias hacen el archivo legible y mantenible cuando el número de reglas aumenta.

**1. Los usuarios**

```text
User_Alias  ADMINS  = debian, bob
User_Alias  JUNIORS = carol, dave, %practicantes
```

**2. Los hosts**

```text
Host_Alias  SERVIDOR_WEB = nombre_servidor1, nombre_servidor2
Host_Alias  SERVIDOR_BD  = 192.168.1.20, 192.168.1.21
```

**3. Los comandos**

```text
Cmnd_Alias  MANTENIMIENTO = /usr/bin/apt, /usr/bin/systemctl
Cmnd_Alias  LECTURA_LOG = /usr/bin/journalctl, /usr/bin/less /var/log/*
Cmnd_Alias  RED      = /usr/sbin/ip, /usr/bin/ss, /usr/sbin/tcpdump
```

**4. Las identidades destino**

```text
Runas_Alias  SERVICIOS = www-data, postgres, redis
```

Ensamblados:

```text
ADMINS   SERVIDOR_WEB = (root)     MANTENIMIENTO
JUNIORS  SERVIDOR_WEB = (root)     LECTURA_LOG
ADMINS   SERVIDOR_BD  = (SERVICIOS) ALL
```

### sudoedit — editar un archivo protegido, correctamente

Esta es la funcionalidad más subutilizada de sudo, y sin embargo la respuesta correcta a una necesidad muy común: *modificar un archivo que pertenece a root, con **mi** editor y **mi** configuración, sin dar derechos de root a ese editor.*

```bash
sudoedit /etc/nginx/nginx.conf
```

```bash
sudo -e /etc/nginx/nginx.conf      # sintaxis equivalente
```

**Lo que realmente sucede:**

```text
1. sudo copia /etc/nginx/nginx.conf a un archivo temporal
2. Le da su propiedad
3. Lanza SU editor, con SU usuario y SU configuración
   → sin derechos de root en el editor, ni en sus complementos
4. Al cerrar, si el archivo cambió, sudo copia el temporal
   a su ubicación de origen, con los derechos y el propietario de origen
5. Toda la operación se registra
```

**Por qué es importante**, comparado con `sudo vim /etc/nginx/nginx.conf`:

| | `sudo vim archivo` | `sudoedit archivo` |
|---|---|---|
| Usuario del editor | **root** | usted |
| Configuración cargada | la de **root** (`/root/.vimrc`) — no la suya | **la suya** (`~/.vimrc`, `~/.config/nvim/`) |
| Complementos ejecutados | como root | con sus derechos |
| Archivo temporal de intercambio (`.swp`) | escrito como root, a veces en lugar incorrecto | en su espacio |
| Derecho sudo necesario | `ALL` o el editor completo | `sudoedit /ruta/precisa` |

> [!IMPORTANT]
> **El caso de la configuración personalizada.** Si utiliza Neovim con una configuración elaborada (LSP, complementos, tema), `sudo nvim` no la carga: root tiene su propio `$HOME`, por lo tanto su propia configuración — muy a menudo inexistente. Se encuentra con un vi desnudo en el peor momento.
>
> La tentación es entonces hacer `sudo -E nvim` para conservar `$HOME`, o peor, hacer `/root/.config` un enlace simbólico al suyo. **Esto es exactamente lo que no debe hacer**: hace entonces ejecutar **como root** decenas de miles de líneas de complementos de terceros, descargados desde GitHub y actualizados automáticamente. Un solo complemento comprometido, y la máquina lo está también.
>
> `sudoedit` resuelve el problema correctamente: su configuración completa, sus complementos, su LSP — todo funciona bajo su usuario, y solo la recopia final del archivo es privilegiada.

**Configurar el editor utilizado** — `sudoedit` mira, en orden, `$SUDO_EDITOR`, `$VISUAL`, `$EDITOR`, luego el `Defaults editor` del sudoers:

```bash
export SUDO_EDITOR=nvim        # en ~/.bashrc o ~/.zshrc
```

Del lado del sudoers, hay que autorizar explícitamente las variables de entorno del usuario, sino `env_reset` las borra:

```text
Defaults  env_keep += "SUDO_EDITOR VISUAL EDITOR"
```

**Delegar la edición de archivos específicos** — aquí es donde `sudoedit` cobra todo su sentido en sudoers. Se da el derecho a modificar un archivo, no el derecho a lanzar un editor como root:

```text
# El equipo web puede modificar los vhosts, nada más
%webops ALL=(root) sudoedit /etc/nginx/sites-available/*

# Un desarrollador puede ajustar la configuración de su aplicación
alice ALL=(root) sudoedit /etc/miapp/config.yml
```

```bash
sudo -l -U alice
```

```text
User alice may run the following commands on servidor:
    (root) sudoedit /etc/miapp/config.yml
```

> [!WARNING]
> En una regla sudoers, escriba `sudoedit` **sin ruta** — es una pseudo-comando interna a sudo, no un binario separado. `/usr/bin/sudoedit` en una regla no funciona como se espera.
>
> Cuidado también con los comodines: `sudoedit /etc/nginx/*` autoriza `sudoedit /etc/nginx/../shadow` en versiones antiguas de sudo. Manténgase en rutas precisas, y actualice sudo a la versión actual.

### Casos prácticos completos

#### 1. Un operador que solo gestiona servicios

```text
# /etc/sudoers.d/ops_servicios
Cmnd_Alias SVC_WEB = /usr/bin/systemctl start nginx,   \
                     /usr/bin/systemctl stop nginx,    \
                     /usr/bin/systemctl restart nginx, \
                     /usr/bin/systemctl reload nginx,  \
                     /usr/bin/systemctl status nginx

Cmnd_Alias SVC_LOG = /usr/bin/journalctl -u nginx *

%ops ALL=(root) SVC_WEB, SVC_LOG
```

#### 2. Una cuenta de despliegue automatizado

```text
# /etc/sudoers.d/despliegue
# Cuenta no interactiva utilizada por CI — clave SSH restringida del lado de authorized_keys
despliegue ALL=(root) NOPASSWD: /usr/local/bin/deploy.sh
despliegue ALL=(root) NOPASSWD: /usr/bin/systemctl restart miapp

Defaults:despliegue !requiretty
Defaults:despliegue log_output
```

> [!CAUTION]
> El script `/usr/local/bin/deploy.sh` debe pertenecer imperiosamente a **root** y solo ser modificable por root (`chown root:root`, `chmod 755`). Si es modificable por el usuario `despliegue`, puede escribir cualquier cosa en él y obtener un shell de root — la regla sudo se convierte en un `NOPASSWD: ALL` disfrazado.
>
> ```bash
> sudo chown root:root /usr/local/bin/deploy.sh
> sudo chmod 755 /usr/local/bin/deploy.sh
> ```

#### 3. Un equipo que gestiona sus propias configuraciones

```text
# /etc/sudoers.d/equipo_app
%devs ALL=(root) sudoedit /etc/miapp/*.yml
%devs ALL=(root) /usr/bin/systemctl restart miapp
%devs ALL=(miapp) /bin/bash          # shell bajo la identidad del servicio, no root
```

#### 4. Una cuenta de copia de seguridad

```text
# /etc/sudoers.d/copia_seguridad
copia_seguridad ALL=(root) NOPASSWD: /usr/bin/rsync --server --sender *
copia_seguridad ALL=(root) NOPASSWD: /usr/bin/tar --acls -czf * /srv/datos
```

### Las trampas de seguridad

> [!CAUTION]
> **1. Comodines demasiado amplios.** Un `*` en un comando cubre mucho más de lo que se imagina:
>
> ```text
> alice ALL=(root) /usr/bin/chown * /var/www
> ```
>
> Esta regla autoriza `sudo chown alice /etc /var/www` — el comodín absorbe argumentos adicionales.

> [!CAUTION]
> **2. Scripts modificables.** Cualquier comando autorizado en sudo debe pertenecer a root y ser modificable solo por root — tanto el binario como todas las carpetas de su ruta.

> [!CAUTION]
> **3. Rutas relativas.** Una regla debe usar siempre una ruta absoluta. `alice ALL=(root) systemctl` (sin ruta) no funciona y no aporta garantía alguna.

Auditoría rápida de los derechos concedidos en una máquina:

```bash
# Todas las reglas efectivas, todos archivos confundidos
sudo grep -rvE '^\s*($|#)' /etc/sudoers /etc/sudoers.d/

# Los NOPASSWD, a revisar en prioridad
sudo grep -rn "NOPASSWD" /etc/sudoers /etc/sudoers.d/

# ¿Quién está en el grupo sudo?
getent group sudo wheel
```

### Auditoría y registro

Cada uso de sudo se registra, exitoso o no.

```bash
sudo cat /var/log/auth.log            # Debian / Ubuntu
sudo cat /var/log/secure              # RHEL / Fedora
sudo journalctl -t sudo
sudo journalctl _COMM=sudo --since today
```

```text
Dec 12 09:31:01 servidor sudo:    alice : TTY=pts/0 ; PWD=/home/alice ;
  USER=root ; COMMAND=/usr/bin/systemctl restart nginx
Dec 12 09:33:22 servidor sudo:      bob : usuario NOT in sudoers ; TTY=pts/1 ;
  PWD=/home/bob ; USER=root ; COMMAND=/usr/bin/apt install htop
```

**El registro de sesión** es la funcionalidad de auditoría avanzada: sudo puede registrar **todo lo que se escribió y se mostró** durante un comando, y reproducirlo.

```text
Defaults  log_input, log_output
Defaults  iolog_dir="/var/log/sudo-io"
```

```bash
sudo sudoreplay -l                    # listar sesiones registradas
```

```text
Dec 12 10:15:03 2025 : alice : TTY=/dev/pts/0 ; CWD=/home/alice ;
  USER=root ; TSID=000001 ; COMMAND=/bin/bash
```

```bash
sudo sudoreplay 000001                # reproducir la sesión, en tiempo real
sudo sudoreplay -s 10 000001          # reproducir 10× más rápido
```

> [!TIP]
> Es extremadamente útil en formación y en análisis post-incident: se ve exactamente qué se hizo, en orden, incluyendo errores de escritura. Se debe desplegar al menos en cuentas con un shell de root.

### Diagnóstico sudo

| Mensaje / síntoma | Causa | Corrección |
|--------------------|-------|------------|
| `alice is not in the sudoers file` | No miembro de `sudo`/`wheel`, sin regla | `usermod -aG sudo alice` luego reconectarse |
| `Sorry, user alice is not allowed to execute …` | El comando no coincide con ninguna regla | `sudo -l` para ver lo autorizado; verificar ruta absoluta y argumentos |
| `sudo: no tty present and no askpass program` | Ejecución no interactiva (cron, CI) sin `NOPASSWD` | Añadir `NOPASSWD:` para este comando específico |
| `sudo: /etc/sudoers.d/xxx is mode 0644, should be 0440` | Permisos incorrectos — el archivo es **ignorado** | `chmod 0440` y `chown root:root` |
| El comando funciona a mano pero no vía sudo | `secure_path` reemplaza su `PATH` | Usar la ruta absoluta del comando |
| `sudo: unable to resolve host servidor` | El nombre del host no está en `/etc/hosts` | Añadir `127.0.1.1 servidor` en `/etc/hosts` |

**Salir de un sudoers roto** — por orden de preferencia:

```bash
# 1. Una sesión de root aún está abierta en algún lugar: reparar inmediatamente
visudo -c && visudo

# 2. pkexec, si está instalado (PolicyKit, independiente de sudo)
pkexec visudo

# 3. La cuenta de root tiene una contraseña
su -

# 4. Último recurso: reiniciar en modo rescate desde GRUB
#    (añadir "init=/bin/bash" a los parámetros del núcleo, luego remontar / en rw)
mount -o remount,rw /
visudo
```

> [!TIP]
> **El reflejo que evita todo esto:** antes de modificar un sudoers en una máquina distante, abra una **segunda sesión SSH como root** y déjela abierta. Si rompe algo, aún tiene acceso para reparar. Es el mismo reflejo que para SSH y el cortafuegos — se vuelve más adelante en las secciones siguientes.

### Resumen sudo

| Comando | Rol |
|----------|------|
| `sudo -l` | Listar los propios derechos |
| `sudo -l -U bob` | Listar derechos de otro |
| `sudo -u alice cmd` | Ejecutar como otra identidad |
| `sudo -i` / `sudo -s` | Shell de root (login / no-login) |
| `sudoedit archivo` | Editar un archivo protegido con **su propio** editor |
| `visudo` | Editar `/etc/sudoers` de forma segura |
| `visudo -c` | Validar la sintaxis |
| `visudo -f /etc/sudoers.d/x` | Crear un fragmento de reglas |
| `sudoreplay -l` / `sudoreplay ID` | Listar / reproducir una sesión registrada |
| `journalctl -t sudo` | Consultar logs |

---

## Cortafuegos

Un cortafuegos filtra el tráfico de red definiendo reglas que determinan qué puede pasar o no.

Bajo Linux, cuatro herramientas coexisten — pero no están al mismo nivel:

| Herramienta | Rol |
|-------|------|
| **netfilter** | El motor de filtrado, **en el núcleo**. Ninguna de las herramientas siguientes lo reemplaza: lo pilotan. |
| **iptables** | La interfaz histórica de netfilter |
| **nftables** | La nueva interfaz oficial, que reemplaza iptables |
| **ufw** | Una capa simplificada sobre iptables/nftables, en sistemas Debian |
| **firewalld** | El equivalente de ufw en sistemas RedHat, con la noción de zonas |

> [!TIP]
> En la práctica: se usa `ufw` (Debian) o `firewalld` (RHEL) en el día a día, y se desciende al nivel `nftables` solo para reglas que la capa superior no sabe expresar. Sin embargo, hay que saber leer una regla `iptables`: es lo que producen la mayoría de documentaciones, scripts antiguos y Docker.

### Los conceptos comunes

Sea cual sea la herramienta, cuatro nociones reaparecen.

**1. El sentido del tráfico** — tres cadenas principales:

```text
                    ┌──────────────┐
   Internet ──────► │    INPUT     │ ──► procesos locales
                    ├──────────────┤
   procesos ─────► │   OUTPUT     │ ──► Internet
                    ├──────────────┤
   Internet ──────► │   FORWARD    │ ──► otra máquina (enrutamiento, Docker, VPN)
                    └──────────────┘
```

En un servidor clásico, **es `INPUT` lo que cuenta**: es aquí donde se decide quién puede alcanzar la máquina.

**2. La política por defecto** — lo que sucede a un paquete que no coincide con ninguna regla:

| Política | Significado |
|-----------|---------------|
| `ACCEPT` / `allow` | Todo lo que no está prohibido está autorizado — **lista negra**, a proscrire |
| `DROP` / `deny` | El paquete se descarta silenciosamente — **lista blanca**, el buen enfoque |
| `REJECT` | El paquete se rechaza con un mensaje de error ICMP |

> [!IMPORTANT]
> ¿`DROP` o `REJECT`? `DROP` no responde nada: el cliente espera hasta timeout, lo que ralentiza los escáneres pero también sus propios diagnósticos. `REJECT` responde inmediatamente « conexión rechazada », lo que es más cortés y más legible.
> Convención común: `DROP` hacia Internet, `REJECT` en la red interna.

**3. El orden de las reglas.** Las reglas se evalúan **de arriba a abajo**, y la **primera** que coincide decide. Una regla de autorización colocada después de un `DROP` general nunca será alcanzada.

**4. El seguimiento de conexión (*stateful*).** El núcleo mantiene un registro de las conexiones establecidas. Esto es lo que permite autorizar solo las conexiones **entrantes iniciales** mientras se dejan que las respuestas al tráfico que inició regresen.

```text
ESTABLISHED  → paquete perteneciente a una conexión ya aceptada
RELATED      → conexión relacionada con otra (ej. el canal de datos FTP)
NEW          → primer intento de conexión — es el que filtramos
```

---

### ufw (Debian, Ubuntu)

#### Ver el estado

```bash
sudo ufw status verbose
```

```text
Status: active
Logging: on (low)
Default: deny (incoming), allow (outgoing), disabled (routed)
New profiles: skip

To                         Action      From
--                         ------      ----
22/tcp                     ALLOW IN    Anywhere
80,443/tcp (WWW Full)      ALLOW IN    Anywhere
3306/tcp                   ALLOW IN    192.168.1.0/24
22/tcp (v6)                ALLOW IN    Anywhere (v6)
80,443/tcp (WWW Full (v6)) ALLOW IN    Anywhere (v6)
```

Cada regla aparece dos veces: una para IPv4 y otra para IPv6.

Para eliminar reglas, necesita su número:

```bash
sudo ufw status numbered
```

```text
     To                         Action      From
     --                         ------      ----
[ 1] 22/tcp                     ALLOW IN    Anywhere
[ 2] 80,443/tcp (WWW Full)      ALLOW IN    Anywhere
[ 3] 3306/tcp                   ALLOW IN    192.168.1.0/24
```

#### La buena secuencia de implementación

> [!CAUTION]
> **En un servidor distante, el orden de estos comandos no es negociable.** Activar ufw antes de haber autorizado SSH le corta el acceso a la máquina, instantánea y definitivamente (excepto acceso por consola).

```bash
# 1. Política por defecto: cerrar todo en entrada, permitir todo en salida
sudo ufw default deny incoming
sudo ufw default allow outgoing

# 2. SSH EN PRIMER LUGAR — antes que nada
sudo ufw allow ssh

# 3. Los servicios realmente necesarios
sudo ufw allow http
sudo ufw allow https

# 4. Solo ahora, activar
sudo ufw enable
```

```text
Command may disrupt existing ssh connections. Proceed with operation (y|n)?
```

#### Crear reglas

```bash
# Por nombre de servicio (leído desde /etc/services)
sudo ufw allow ssh
sudo ufw allow http

# Por número de puerto
sudo ufw allow 22
sudo ufw allow 22/tcp
sudo ufw allow 8080/tcp

# Un rango de puertos
sudo ufw allow 6000:6010/tcp

# Desde una IP específica
sudo ufw allow from 203.0.113.42

# Desde una subred, hacia un puerto específico — la forma más útil
sudo ufw allow from 192.168.1.0/24 to any port 3306 proto tcp

# En una interfaz dada
sudo ufw allow in on eth1 to any port 5432

# Con un comentario — hacer sistemáticamente
sudo ufw allow from 192.168.1.0/24 to any port 3306 comment 'MySQL LAN solamente'
```

#### Bloquear y eliminar

```bash
sudo ufw deny 3306                          # bloquear (silencioso)
sudo ufw reject 3306                        # rechazar (con mensaje de error)
sudo ufw deny from 203.0.113.66             # proscribir una IP

sudo ufw delete allow 22                    # por descripción
sudo ufw status numbered && sudo ufw delete 3   # por número
```

> [!WARNING]
> Eliminar por número **renumerota** las reglas siguientes. Para eliminar varias, comience por la mayor, o relea `ufw status numbered` entre cada eliminación.

#### Insertar una regla en una posición específica

El orden importa: `ufw insert` coloca la regla antes de otras.

```bash
sudo ufw insert 1 deny from 203.0.113.66
```

#### Limitar conexiones (anti-bruteforce)

```bash
sudo ufw limit ssh
```

Bloquea una IP que intenta más de 6 conexiones en 30 segundos. Es una red mínima, útil pero menos precisa que [fail2ban](#fail2ban).

#### Los perfiles de aplicaciones

```bash
sudo ufw app list
```

```text
Available applications:
  Apache Full
  Nginx Full
  Nginx HTTP
  OpenSSH
```

```bash
sudo ufw app info 'Nginx Full'
sudo ufw allow 'Nginx Full'
```

Los perfiles son simplemente archivos en `/etc/ufw/applications.d/`:

```text
[Nginx Full]
title=Web Server (Nginx, HTTP + HTTPS)
description=Small, but very powerful and efficient web server
ports=80,443/tcp
```

#### Registro

```bash
sudo ufw logging on
sudo ufw logging medium        # off | low | medium | high | full
```

```bash
sudo tail -f /var/log/ufw.log
```

```text
[UFW BLOCK] IN=eth0 OUT= SRC=203.0.113.66 DST=192.168.1.10
  PROTO=TCP SPT=54321 DPT=3306 WINDOW=1024 SYN
```

| Campo | Significado |
|-------|------|
| `UFW BLOCK` | La regla aplicada |
| `SRC` / `DST` | IP origen / destino |
| `SPT` / `DPT` | Puerto origen / **puerto destino** — el que interesa |
| `PROTO` | Protocolo |

#### Reinicializar

```bash
sudo ufw reset          # desactiva y elimina todas las reglas
```

Los archivos de configuración se encuentran en `/etc/ufw/` y `/etc/default/ufw`.

---

### firewalld (RHEL, Fedora)

firewalld introduce una noción ausente en ufw: las **zonas**. Cada interfaz de red se asigna a una zona, y cada zona tiene su propio conjunto de reglas. Esto se pensó para máquinas con múltiples interfaces (un servidor con un puerto público y un puerto LAN).

```bash
sudo firewall-cmd --get-zones
```

| Zona | Nivel de confianza |
|------|---------------------|
| `drop` | Todo se descarta, sin respuesta |
| `block` | Todo se rechaza con un mensaje ICMP |
| `public` | **Zona por defecto** — red no fiable, algunos servicios autorizados |
| `external` | Red externa, con mascarading (NAT) |
| `dmz` | Máquinas expuestas, acceso interno limitado |
| `work` / `home` | Redes relativamente fiables |
| `internal` | Red interna |
| `trusted` | **Todo** se acepta |

#### Estado e inspección

```bash
sudo firewall-cmd --state
sudo firewall-cmd --get-active-zones
sudo firewall-cmd --get-default-zone
sudo firewall-cmd --list-all
```

```text
public (active)
  target: default
  icmp-block-inversion: no
  interfaces: eth0
  sources:
  services: dhcpv6-client ssh http https
  ports: 8080/tcp
  protocols:
  forward: yes
  masquerade: no
  forward-ports:
  source-ports:
  icmp-blocks:
  rich rules:
```

#### Runtime versus permanente

> [!IMPORTANT]
> Esta es **la** particularidad de firewalld, y la fuente de la mayoría de errores: mantiene dos configuraciones distintas.
>
> | | Efecto | Sobrevive al reboot |
> |---|---|---|
> | Sin `--permanent` | Inmediato | ❌ No |
> | Con `--permanent` | **Sin** efecto inmediato | ✅ Sí |
>
> La secuencia correcta es por tanto sistemáticamente:
>
> ```bash
> sudo firewall-cmd --permanent --add-service=http
> sudo firewall-cmd --reload
> ```
>
> Truco de prueba seguro: aplicar primero en runtime solo, verificar que todo funciona, **luego** hacerlo permanente. Si se corta el acceso, un simple reboot restaura la situación.

#### Gestionar servicios y puertos

```bash
# Por nombre de servicio (definido en /usr/lib/firewalld/services/)
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=https
sudo firewall-cmd --permanent --remove-service=cockpit

# Por puerto
sudo firewall-cmd --permanent --add-port=8080/tcp
sudo firewall-cmd --permanent --add-port=6000-6010/udp
sudo firewall-cmd --permanent --remove-port=3306/tcp

# Listar servicios disponibles
sudo firewall-cmd --get-services

sudo firewall-cmd --reload
```

#### Trabajar con zonas

```bash
# Asignar una interfaz a una zona
sudo firewall-cmd --permanent --zone=internal --change-interface=eth1

# Asignar una fuente (subred) a una zona de confianza
sudo firewall-cmd --permanent --zone=internal --add-source=192.168.1.0/24

# Abrir un servicio solo en esta zona
sudo firewall-cmd --permanent --zone=internal --add-service=mysql

sudo firewall-cmd --reload
```

Resultado: MySQL es accesible solo desde la LAN, no desde la interfaz pública — sin haber escrito ninguna regla de IP.

#### Las rich rules

Para lo que los servicios y las zonas no cubren:

```bash
# Autorizar una IP específica en un puerto específico
sudo firewall-cmd --permanent --add-rich-rule='
  rule family="ipv4" source address="203.0.113.42" port port="5432" protocol="tcp" accept'

# Proscribir una IP
sudo firewall-cmd --permanent --add-rich-rule='
  rule family="ipv4" source address="203.0.113.66" reject'

# Limitar la velocidad de conexión y registrar
sudo firewall-cmd --permanent --add-rich-rule='
  rule service name="ssh" log prefix="SSH " level="info" limit value="3/m" accept'

sudo firewall-cmd --reload
sudo firewall-cmd --list-rich-rules
```

#### El modo pánico

```bash
sudo firewall-cmd --panic-on       # corta TODO el tráfico, inmediatamente
sudo firewall-cmd --panic-off
sudo firewall-cmd --query-panic
```

> [!CAUTION]
> `--panic-on` corta también su sesión SSH. Resérvelo a intervenciones desde la consola física o consola virtual del proveedor cloud.

---

### iptables

iptables organiza las reglas en **tablas**, ellas mismas compuestas de **cadenas**.

| Tabla | Uso |
|-------|-------|
| `filter` | **Tabla por defecto** — aceptar o rechazar (cadenas INPUT, OUTPUT, FORWARD) |
| `nat` | Traducción de direcciones: redirección de puertos, mascarading |
| `mangle` | Modificación de encabezados de paquetes |
| `raw` | Exclusión del seguimiento de conexión |

#### Leer las reglas

```bash
sudo iptables -L -v -n --line-numbers
```

| Opción | Efecto |
|--------|-------|
| `-L` | Listar |
| `-v` | Verboso (contadores de paquetes y octetos) |
| `-n` | No resolver nombres — mucho más rápido |
| `--line-numbers` | Numerar, para poder eliminar por índice |

```text
Chain INPUT (policy DROP 12 packets, 720 bytes)
num  pkts bytes target  prot opt in   out  source      destination
1    1523  128K ACCEPT  all  --  lo   *    0.0.0.0/0   0.0.0.0/0
2   84021   42M ACCEPT  all  --  *    *    0.0.0.0/0   0.0.0.0/0   ctstate RELATED,ESTABLISHED
3      42  2520 ACCEPT  tcp  --  *    *    0.0.0.0/0   0.0.0.0/0   tcp dpt:22
```

> [!TIP]
> Las columnas `pkts` y `bytes` son preciosas en diagnóstico: una regla cuyo contador se queda en cero nunca es alcanzada. Esto es a menudo señal de que una regla más arriba intercepta el tráfico.
> `sudo iptables -Z` reinicia los contadores a cero antes de una prueba.

#### Construir un conjunto minimal de reglas

Aquí hay un conjunto completo y comentado, para un servidor web con SSH:

```bash
#!/bin/bash
set -euo pipefail

# 1. Vaciar reglas existentes
iptables -F
iptables -X

# 2. Política por defecto: rechazar todo en entrada
iptables -P INPUT   DROP
iptables -P FORWARD DROP
iptables -P OUTPUT  ACCEPT

# 3. Autorizar la interfaz de loopback (indispensable — muchos servicios
#    se comunican entre sí vía 127.0.0.1)
iptables -A INPUT -i lo -j ACCEPT

# 4. Autorizar respuestas al tráfico que hemos iniciado (stateful)
iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT

# 5. Descartar paquetes inválidos
iptables -A INPUT -m conntrack --ctstate INVALID -j DROP

# 6. SSH — primero entre los servicios
iptables -A INPUT -p tcp --dport 22 -m conntrack --ctstate NEW -j ACCEPT

# 7. HTTP / HTTPS
iptables -A INPUT -p tcp --dport 80  -m conntrack --ctstate NEW -j ACCEPT
iptables -A INPUT -p tcp --dport 443 -m conntrack --ctstate NEW -j ACCEPT

# 8. PostgreSQL, solo desde la LAN
iptables -A INPUT -p tcp -s 192.168.1.0/24 --dport 5432 -j ACCEPT

# 9. Autorizar ping (útil para supervisión)
iptables -A INPUT -p icmp --icmp-type echo-request -j ACCEPT

# 10. Registrar lo que se descarta, con límite de velocidad
iptables -A INPUT -m limit --limit 5/min -j LOG --log-prefix "IPT-DROP: " --log-level 4
```

> [!IMPORTANT]
> Los pasos 3 y 4 son los que se olvidan, y sus síntomas son desconcertantes:
> - sin `-i lo -j ACCEPT`, servicios locales dejan de funcionar sin razón aparente;
> - sin la regla `ESTABLISHED,RELATED`, la máquina no puede descargar nada: los paquetes se van, pero las respuestas se bloquean en la entrada.

#### Añadir, insertar, eliminar

```bash
iptables -A INPUT -p tcp --dport 80 -j ACCEPT     # -A: Añadir al FINAL
iptables -I INPUT 1 -s 203.0.113.66 -j DROP       # -I: Insertar en posición 1
iptables -D INPUT 3                               # -D: eliminar regla n° 3
iptables -R INPUT 2 -p tcp --dport 8080 -j ACCEPT # -R: reemplazar
```

> [!WARNING]
> `-A` añade **después** de todas las reglas existentes. En una cadena que termina con un `DROP` explícito, la nueva regla nunca será evaluada. En caso de duda, use `-I` con una posición, y verifique con `-L --line-numbers`.

#### Persistencia

> [!CAUTION]
> **Las reglas iptables se pierden al reiniciar.** Esta es la propiedad que sorprende más — y que salva también: en caso de un error de manipulación que le corta el acceso, un reinicio restaura la situación.

```bash
sudo apt install iptables-persistent          # Debian/Ubuntu
sudo iptables-save > /etc/iptables/rules.v4
sudo ip6tables-save > /etc/iptables/rules.v6
```

```bash
sudo iptables-restore < /etc/iptables/rules.v4
```

---

### nftables

`nftables` reemplaza iptables, ip6tables, arptables y ebtables por una herramienta única y una sintaxis coherente. Es el motor por defecto de Debian 10+ y RHEL 8+.

#### Inspeccionar

```bash
sudo nft list ruleset
```

#### La estructura

```text
table  <familia>  <nombre>        familia: ip, ip6, inet (ambos), arp, bridge
  └─ chain <nombre> { type … hook … priority … ; policy … ; }
       └─ reglas
```

#### Un ruleset completo y comentado

```bash
#!/usr/sbin/nft -f
# /etc/nftables.conf

flush ruleset

table inet filtro {
    chain entrada {
        # hook entrada: filtramos lo que entra; policy drop: lista blanca
        type filter hook input priority 0; policy drop;

        # Loopback
        iif lo accept

        # Conexiones ya establecidas
        ct state established,related accept
        ct state invalid drop

        # ICMP (ping, descubrimiento de MTU)
        ip protocol icmp accept
        ip6 nexthdr icmpv6 accept

        # SSH, con limitación anti-bruteforce
        tcp dport 22 ct state new limit rate 6/minute accept

        # Web
        tcp dport { 80, 443 } accept

        # PostgreSQL solo desde la LAN
        ip saddr 192.168.1.0/24 tcp dport 5432 accept

        # Registrar el resto antes de descartarlo
        limit rate 5/minute log prefix "NFT-DROP: " level info
    }

    chain reenvío {
        type filter hook forward priority 0; policy drop;
    }

    chain salida {
        type filter hook output priority 0; policy accept;
    }
}
```

> [!TIP]
> La sintaxis es claramente más legible que iptables: los **conjuntos** (`{ 80, 443 }`) evitan repetir reglas, y la familia `inet` cubre IPv4 **y** IPv6 en un solo conjunto de reglas — donde iptables obligaba a escribir todo dos veces.

#### Cargar, probar y hacer permanente

```bash
# Verificar la sintaxis sin aplicar
sudo nft -c -f /etc/nftables.conf

# Aplicar
sudo nft -f /etc/nftables.conf

# Hacer permanente (el servicio recarga /etc/nftables.conf al iniciar)
sudo systemctl enable --now nftables
```

Comandos puntuales:

```bash
sudo nft add rule inet filter input tcp dport 8080 accept
sudo nft list ruleset -a                  # -a muestra los handles
sudo nft delete rule inet filter input handle 12
sudo nft flush ruleset                    # borrar todo
```

---

### Probar y diagnosticar un cortafuegos

#### 1. ¿El servicio escucha realmente?

Antes de acusar al cortafuegos, verifique que algo escucha en el puerto:

```bash
sudo ss -tulpn
```

```text
Netid State  Local Address:Port   Process
tcp   LISTEN 0.0.0.0:22           users:(("sshd",pid=812,fd=3))
tcp   LISTEN 127.0.0.1:5432       users:(("postgres",pid=1043,fd=5))
tcp   LISTEN 0.0.0.0:80           users:(("nginx",pid=1201,fd=6))
```

> [!IMPORTANT]
> Fíjese en la **dirección de escucha**, no solo en el puerto. Arriba, PostgreSQL escucha en `127.0.0.1:5432`: es inaccesible desde el exterior, y **ninguna regla de cortafuegos cambiará nada**. Hay que modificar la configuración del servicio (`listen_addresses` para PostgreSQL).
>
> `0.0.0.0` = todas las interfaces · `127.0.0.1` = solo la máquina local · `::` = todas, en IPv6.

#### 3. Leer los registros

```bash
sudo tail -f /var/log/ufw.log                    # ufw
sudo journalctl -f -k | grep -E "DROP|REJECT"    # iptables / nftables
sudo firewall-cmd --get-log-denied               # firewalld
sudo firewall-cmd --set-log-denied=all
```

#### 4. La lista de verificación

| Síntoma | Verificar |
|---------|-----------|
| Conexión rechazada de inmediato | Ningún servicio escucha (`ss -tulpn`), o regla `REJECT` |
| Conexión que se cuelga hasta el timeout | Regla `DROP`, o cortafuegos intermedio (nube, router) |
| Funciona en local, no a distancia | El servicio escucha solo en `127.0.0.1` |
| Funciona en IPv4, no en IPv6 | Reglas IPv6 olvidadas (`ip6tables`, o familia `inet` en nftables) |
| La regla existe pero no tiene efecto | Una regla anterior intercepta — verificar el orden y los contadores |
| Todo se rompe tras un reinicio | Reglas no persistidas, o `--permanent` olvidado |
| Docker ignora mi cortafuegos | Docker escribe sus propias reglas en la cadena `DOCKER-USER` |

> [!WARNING]
> **Docker esquiva ufw.** Un contenedor lanzado con `-p 3306:3306` abre el puerto en todas las interfaces insertando sus reglas **antes** que las de ufw. Su `ufw deny 3306` no se aplica.
> Dos soluciones: publicar en el bucle local (`-p 127.0.0.1:3306:3306`), o escribir sus reglas en la cadena `DOCKER-USER`, que se evalúa antes que las de Docker.

#### La red de seguridad anti-bloqueo

> [!TIP]
> Antes de cualquier manipulación del cortafuegos en una máquina remota, programe un reinicio automático. Si se corta el acceso, le reabre la puerta cinco minutos después.
>
> ```bash
> # Debian / ufw
> echo "ufw --force reset && ufw --force enable" | sudo at now + 5 minutes
>
> # iptables
> echo "iptables -P INPUT ACCEPT && iptables -F" | sudo at now + 5 minutes
> ```
>
> Si todo va bien, se cancela el trabajo programado:
>
> ```bash
> sudo atq              # listar
> sudo atrm 1           # cancelar
> ```
>
> Y por supuesto, la regla universal de este capítulo: **mantenga una segunda sesión SSH abierta** durante toda la intervención.

### Resumen del cortafuegos

| Acción | ufw | firewalld | nftables |
|--------|-----|-----------|----------|
| Estado | `ufw status verbose` | `firewall-cmd --list-all` | `nft list ruleset` |
| Activar | `ufw enable` | `systemctl enable --now firewalld` | `systemctl enable --now nftables` |
| Abrir un puerto | `ufw allow 80/tcp` | `firewall-cmd --permanent --add-port=80/tcp` | `tcp dport 80 accept` |
| Abrir un servicio | `ufw allow http` | `firewall-cmd --permanent --add-service=http` | — |
| Restringir a una LAN | `ufw allow from 192.168.1.0/24 to any port 3306` | `--zone=internal --add-source=…` | `ip saddr 192.168.1.0/24 … accept` |
| Eliminar | `ufw delete 3` | `--permanent --remove-port=80/tcp` | `nft delete rule … handle N` |
| Aplicar | inmediato | `firewall-cmd --reload` | `nft -f /etc/nftables.conf` |
| Persistencia | automática | `--permanent` | `/etc/nftables.conf` |

---

## SSH

SSH (*Secure Shell*) es un protocolo de conexión a distancia a un servidor (Linux como Windows), que funciona en línea de comandos. Es la herramienta central de la administración a distancia: todo lo demás — transferencia de archivos, túneles, despliegue, copia de seguridad — pasa por él.

### Instalación y verificación

```bash
sudo apt install openssh-server        # el servidor (para aceptar conexiones)
sudo apt install openssh-client        # el cliente (para conectarse a otros lugares)
```

```bash
systemctl status ssh                   # Debian/Ubuntu : el servicio se llama "ssh"
systemctl status sshd                  # RHEL/Fedora   : se llama "sshd"
```

```bash
ss -tulpn | grep sshd                  # ¿en qué puerto y qué interfaz escucha?
```

### Los dos archivos de configuración

> [!IMPORTANT]
> No confunda los dos archivos — es el error más común con SSH, y hace perder horas:
>
> | Archivo | Concierne | Utilizado para |
> |---------|----------|--------------|
> | `/etc/ssh/ssh_config` | El **cliente** | Las opciones utilizadas cuando *me conecto* a otras máquinas |
> | `~/.ssh/config` | El **cliente**, para un usuario | Ídem, prioritario sobre el archivo del sistema |
> | `/etc/ssh/sshd_config` | El **servidor** (demonio `sshd`) | Las reglas impuestas a los que *se conectan a mi máquina* |
>
> La `d` de `sshd_config` es la de *daemon*. Una directiva `PasswordAuthentication no` colocada en `ssh_config` no asegura **nada**: solo impide que su cliente proponga una contraseña a los servidores distantes.

### Gestión de claves

#### Generar un par de claves

```bash
ssh-keygen -t ed25519 -C "alice@portable-2026"
```

```text
Generating public/private ed25519 key pair.
Enter file in which to save the key (/home/alice/.ssh/id_ed25519):
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in /home/alice/.ssh/id_ed25519
Your public key has been saved in /home/alice/.ssh/id_ed25519.pub
The key fingerprint is:
SHA256:2Xk9…Qw8 alice@portable-2026
```

| Opción | Rol |
|--------|------|
| `-t ed25519` | Tipo de clave |
| `-C "comentario"` | Comentario — ponga **quién** y **qué máquina**, esto es lo que permite después identificar una clave en un `authorized_keys` |
| `-f ~/.ssh/id_deploy` | Ruta del archivo (para tener varias claves) |
| `-a 100` | Número de rondas de derivación de la contraseña |

El comando genera **dos** archivos:

| Archivo | Naturaleza | Regla |
|---------|--------|-------|
| `id_ed25519` | Clave **privada** | Permanece en su máquina, **nunca** se comparte |
| `id_ed25519.pub` | Clave **pública** | Se deposita en los servidores |

#### ¿Qué tipo de clave elegir?

| Tipo | Veredicto |
|------|---------|
| **ed25519** | ✅ **La opción por defecto** — corta, rápida, segura, soportada en todas partes desde OpenSSH 6.5 (2014) |
| `rsa` (4096 bits) | Aceptable si necesita comunicarse con equipo antiguo: `ssh-keygen -t rsa -b 4096` |
| `ecdsa` | A evitar — sin beneficio sobre ed25519 |
| `dsa` | ❌ Obsoleta y desactivada por defecto |

#### La contraseña

> [!IMPORTANT]
> **Ponga una contraseña en sus claves personales.** Una clave privada sin contraseña es una contraseña en texto plano en un archivo: quienquiera que copie este archivo — robo de portátil, copia de seguridad mal protegida, malware — obtiene todos sus accesos.
>
> La objeción habitual (« voy a tener que escribirla todo el tiempo ») se resuelve con el [agente SSH](#el-agente-ssh): se introduce una vez por sesión.
>
> Las claves **de servicio** (CI/CD, copias de seguridad automatizadas) son la excepción: no pueden tener contraseña, ya que nadie está allí para escribirla. Se compensan con restricciones en `authorized_keys` — ver [más abajo](#restringir-una-clave-en-authorized_keys).

Añadir, cambiar o quitar una contraseña en una clave existente:

```bash
ssh-keygen -p -f ~/.ssh/id_ed25519
```

#### Inspeccionar una clave

```bash
ssh-keygen -l -f ~/.ssh/id_ed25519.pub          # huella dactilar
ssh-keygen -lv -f ~/.ssh/id_ed25519.pub         # huella dactilar + arte ASCII
```

```text
256 SHA256:2Xk9…Qw8 alice@portable-2026 (ED25519)
```

Ejemplo de clave pública (formato: `tipo` `clave` `comentario`):

```text
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIL8kQ2vXn5R7T9pM3wYqZ1cD4eF6gH8jK0lN2oP4rS6t alice@portable-2026
```

> [!TIP]
> **La rotación de claves.** Se aconseja cambiar sus claves SSH cada seis meses aproximadamente.
> Durante una rotación, despliegue la nueva clave **antes** de retirar la antigua, y verifique que la nueva funciona en cada servidor antes de limpiar. El comentario `-C` con la fecha hace el inventario mucho más simple.

### Desplegar su clave pública

La clave pública se deposita en el servidor, en `~/.ssh/authorized_keys` del usuario destino. Este archivo puede contener varias claves, una por línea.

**El método automático:**

```bash
ssh-copy-id username@server
ssh-copy-id -i ~/.ssh/id_deploy.pub deploy@server      # una clave precisa
```

**El método manual**, cuando `ssh-copy-id` no está disponible:

```bash
cat ~/.ssh/id_ed25519.pub | ssh username@server \
  "mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
```

#### Los permisos, causa n° 1 de fallos

> [!CAUTION]
> `sshd` **rechaza silenciosamente** cualquier clave cuyos permisos sean demasiado amplios — esto es una protección, pero el mensaje de error del lado del cliente (`Permission denied (publickey)`) no dice nada al respecto.

| Elemento | Permisos | Propietario |
|---------|-------------|--------------|
| `~` (el home) | Sin `w` para grupo/otros — `755` o menos | el usuario |
| `~/.ssh/` | `700` | el usuario |
| `~/.ssh/authorized_keys` | `600` | el usuario |
| `~/.ssh/id_ed25519` (privada) | `600` | el usuario |
| `~/.ssh/id_ed25519.pub` | `644` | el usuario |
| `~/.ssh/config` | `600` | el usuario |

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys ~/.ssh/id_ed25519 ~/.ssh/config
chmod 644 ~/.ssh/id_ed25519.pub
chown -R $USER:$USER ~/.ssh
```

### Reforzar el servidor: `sshd_config`

Aquí hay una configuración de producción comentada. Cada línea se coloca en `/etc/ssh/sshd_config`.

```text
# ── Escucha ────────────────────────────────────────────────────────
Port 22
AddressFamily inet                    # Solo IPv4, si IPv6 no se usa
ListenAddress 0.0.0.0

# ── Autenticación ──────────────────────────────────────────────
PermitRootLogin no                    # nunca conexión root directa
PubkeyAuthentication yes
PasswordAuthentication no             # solo claves
PermitEmptyPasswords no
ChallengeResponseAuthentication no
KbdInteractiveAuthentication no
UsePAM yes

MaxAuthTries 3                        # 3 intentos luego desconexión
MaxSessions 5
LoginGraceTime 30                     # 30 s para autenticarse

# ── Quién tiene derecho a conectarse ────────────────────────────────
AllowGroups sshusers                  # lista blanca por grupo

# ── Reducción de la superficie de ataque ─────────────────────────────
X11Forwarding no
AllowAgentForwarding no
AllowTcpForwarding no                 # dejar en "yes" si usa túneles
PermitTunnel no
PermitUserEnvironment no

# ── Sesiones inactivas ────────────────────────────────────────────
ClientAliveInterval 300               # sonda cada 5 min
ClientAliveCountMax 2                 # desconecta tras 2 sondeos sin respuesta

# ── Registro ────────────────────────────────────────────────────
LogLevel VERBOSE                      # registra las huellas de claves utilizadas

# ── Aviso legal (obligatorio en algunos contextos) ─────────────────────────
Banner /etc/issue.net
```

Las directivas más importantes, resumidas:

| Directiva | Valor | Por qué |
|-----------|--------|----------|
| `PermitRootLogin` | `no` | Fuerza el paso por una cuenta nominativa luego `sudo` — trazabilidad |
| `PasswordAuthentication` | `no` | Elimina de un golpe el ataque de fuerza bruta de contraseñas |
| `AllowGroups` / `AllowUsers` | lista blanca | Una nueva cuenta del sistema no se hace accesible por accidente |
| `MaxAuthTries` | `3` | Limita intentos por conexión |
| `LoginGraceTime` | `30` | Limita conexiones medio abiertas |
| `LogLevel` | `VERBOSE` | Registra **qué clave** sirvió — indispensable en auditoría |

#### El procedimiento de modificación, sin bloquearse

> [!CAUTION]
> **La regla absoluta: mantenga una segunda sesión SSH abierta durante toda la manipulación.** Mientras esta sesión esté activa, no se ve afectada por la recarga de configuración: sigue siendo su puerta de escape.

```bash
# 1. Guardar
sudo cp /etc/ssh/sshd_config /etc/ssh/sshd_config.bak

# 2. Modificar
sudo vim /etc/ssh/sshd_config

# 3. VERIFICAR LA SINTAXIS — antes de cualquier recarga
sudo sshd -t
```

Ninguna salida significa que todo es correcto. Sino:

```text
/etc/ssh/sshd_config line 42: Bad configuration option: PermitRootLgin
```

```bash
# 4. Ver la configuración efectiva, como la entiende sshd
sudo sshd -T | grep -iE "permitrootlogin|passwordauth|allowgroups"

# 5. Aplicar (reload, no restart: se preservan sesiones en curso)
sudo systemctl reload ssh

# 6. DESDE UN TERCER TERMINAL, probar una nueva conexión
ssh -v username@servidor
```

No cierre la sesión de escape hasta haber confirmado que la nueva conexión funciona.

> [!TIP]
> `sshd -T` es muy útil: muestra la configuración **resuelta**, valores por defecto incluidos. Es la única forma de saber con seguridad qué hace el servidor, sin tener que leer archivos incluidos y bloques `Match`.
>
> En distribuciones recientes, `sshd_config` termina con `Include /etc/ssh/sshd_config.d/*.conf`. Como el **primer** valor encontrado prevalece en SSH, los archivos de este carpeta anulan el resto: verifique allí en caso de comportamiento inesperado.

### Los bloques Match

`Match` aplica directivas a un subconjunto de conexiones. Esto permite políticas diferenciadas.

```text
# Configuración global: bloqueada
PasswordAuthentication no
AllowTcpForwarding no
PermitRootLogin no

# La cuenta de copia de seguridad solo tiene derecho a rsync, sin shell interactivo
Match User backup
    ForceCommand /usr/bin/rrsync -ro /srv/data
    AllowTcpForwarding no
    X11Forwarding no

# Los administradores, desde la red interna, pueden usar túneles
Match Group admins Address 192.168.1.0/24
    AllowTcpForwarding yes
    AllowAgentForwarding yes

# Una cuenta SFTP confinada a su carpeta (chroot)
Match Group sftponly
    ChrootDirectory /srv/sftp/%u
    ForceCommand internal-sftp
    AllowTcpForwarding no
    PermitTunnel no
```

| Criterio | Ejemplo |
|---------|---------|
| `Match User alice,bob` | Por usuario |
| `Match Group admins` | Por grupo |
| `Match Address 192.168.1.0/24` | Por IP origen |
| `Match LocalPort 2222` | Por puerto de escucha |

> [!IMPORTANT]
> Un bloque `Match` se extiende **hasta el próximo `Match`** o el fin del archivo. Todas las directivas globales deben por tanto escribirse **antes** del primer `Match` — sino se ven capturadas por un bloque y se aplican solo a él.
>
> Para el chroot SFTP, `ChrootDirectory` requiere que la carpeta y **todos sus padres** pertenezcan a root y no sean modificables por el grupo u otros.

### Restringir una clave en `authorized_keys`

Cada línea de `authorized_keys` puede tener un prefijo con opciones. Este es el lugar correcto para encuadrar las claves de automatización, que no tienen contraseña.

```text
# Una clave de despliegue: solo puede lanzar un comando, desde una IP dada
command="/usr/local/bin/deploy.sh",from="203.0.113.10",no-agent-forwarding,no-port-forwarding,no-pty,no-X11-forwarding ssh-ed25519 AAAAC3Nz… ci@gitlab

# Una clave de copia de seguridad: rsync en solo lectura
command="/usr/bin/rrsync -ro /srv/data",no-pty,no-port-forwarding ssh-ed25519 AAAAC3Nz… backup@nas

# Una clave de administración normal, restringida a la red interna
from="192.168.1.0/24" ssh-ed25519 AAAAC3Nz… alice@portable
```

| Opción | Efecto |
|--------|-------|
| `command="…"` | **Obliga** la ejecución de este comando, sea lo que pida el cliente |
| `from="192.168.1.0/24,10.0.0.5"` | Acepta esta clave solo desde estas direcciones |
| `no-pty` | Prohíbe la asignación de terminal — sin shell interactivo |
| `no-port-forwarding` | Prohíbe los túneles |
| `no-agent-forwarding` | Prohíbe la transferencia de agente |
| `no-X11-forwarding` | Prohíbe la transferencia gráfica |
| `expiry-time="20261231"` | Fecha de expiración de la clave |
| `restrict` | **Prohibir todo**, luego reabrir explícitamente — la forma recomendada |

```text
restrict,command="/usr/local/bin/backup.sh" ssh-ed25519 AAAAC3Nz… backup@nas
```

> [!TIP]
> `restrict` apareció con OpenSSH 7.2 y es mejor que enumerar los `no-*`: desactiva todo, incluidas las funcionalidades añadidas en versiones futuras. Se reabre luego caso por caso con `pty`, `port-forwarding`, etc.

**Auditar las claves desplegadas en un servidor:**

```bash
sudo find /home /root -name authorized_keys -exec echo "── {}" \; -exec cat {} \;
```

```bash
# Con huellas dactilares, para comparar con un inventario
sudo ssh-keygen -l -f /home/alice/.ssh/authorized_keys
```

### Configurar el cliente: `~/.ssh/config`

Este es el archivo que transforma el uso diario de SSH. Evita recordar IP, puertos y nombres de usuario.

```text
# ~/.ssh/config

# ── Valores comunes a todos los hosts ─────────────────────────
Host *
    ServerAliveInterval 60            # mantener la conexión viva
    ServerAliveCountMax 3
    AddKeysToAgent yes                # añadir la clave al agente en 1er uso
    HashKnownHosts yes
    Compression yes

# ── Un servidor simple ─────────────────────────────────────────
Host web
    HostName 203.0.113.10
    User debian
    Port 22
    IdentityFile ~/.ssh/id_ed25519

# ── Un servidor en un puerto no estándar ───────────────────────
Host prod-db
    HostName db.exemple.be
    User admin
    Port 2222
    IdentityFile ~/.ssh/id_prod
    IdentitiesOnly yes                # probar SOLO esta clave

# ── Un servidor accesible solo vía un rebote ────────────────────
Host interne
    HostName 10.0.0.42
    User debian
    ProxyJump web                     # pasa por el host "web" definido arriba

# ── Un motivo: todos los servidores de un dominio ─────────────────
Host *.exemple.be
    User admin
    IdentityFile ~/.ssh/id_pro

# ── Reutilizar una conexión existente (acelera mucho) ────────
Host lento
    HostName lento.exemple.be
    ControlMaster auto
    ControlPath ~/.ssh/sockets/%r@%h-%p
    ControlPersist 10m
```

La conexión se convierte en:

```bash
ssh web
ssh interne          # atraviesa automáticamente el rebote
scp fichero.txt web:/tmp/
```

| Directiva | Rol |
|-----------|------|
| `HostName` | La dirección real |
| `User` | La cuenta distante |
| `Port` | El puerto |
| `IdentityFile` | La clave a presentar |
| `IdentitiesOnly yes` | No probar todas las claves del agente (evita `MaxAuthTries exceeded`) |
| `ProxyJump` | Rebote por una máquina intermedia |
| `ServerAliveInterval` | Evita cortes de sesiones inactivas |
| `ControlMaster` / `ControlPersist` | Multiplexing: las conexiones siguientes reutilizan la primera, instantáneamente |

> [!IMPORTANT]
> A diferencia de la mayoría de archivos de configuración Linux, **SSH retiene el primer valor encontrado**, no el último. Los bloques más específicos deben por tanto colocarse **en la parte superior** del archivo, y el `Host *` **en la parte inferior**.

Para crear la carpeta de sockets de multiplexing:

```bash
mkdir -p ~/.ssh/sockets && chmod 700 ~/.ssh/sockets
```

### `known_hosts` y huellas dactilares

En la primera conexión:

```text
The authenticity of host '203.0.113.10 (203.0.113.10)' can't be established.
ED25519 key fingerprint is SHA256:2Xk9pQr7…Qw8.
Are you sure you want to continue connecting (yes/no/[fingerprint])?
```

> [!IMPORTANT]
> Esta pregunta no es una formalidad: es **la** protección contra el ataque del hombre en el medio. Responder `yes` sin verificar equivale a aceptar cualquier servidor que se presentara a esta dirección.
>
> La buena práctica: recuperar la huella dactilares por un canal independiente (consola del proveedor nube, documentación interna, salida de instalación) y compararla. En el servidor:
>
> ```bash
> sudo ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub
> ```

La huella dactilares aceptada se registra en `~/.ssh/known_hosts`. Si después cambia:

```text
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
@    WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!     @
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
```

Dos explicaciones posibles: el servidor se reinstáló (caso frecuente e inofensivo), o alguien se interpone (caso grave). **No retire la línea hasta haber determinado cuál.**

```bash
ssh-keygen -R 203.0.113.10                  # retirar la entrada de un host
ssh-keygen -F 203.0.113.10                  # buscar una entrada
ssh-keyscan -t ed25519 203.0.113.10         # recuperar la clave de host
```

### El agente SSH

El agente mantiene sus claves desbloqueadas en memoria: la contraseña se solicita solo una vez por sesión.

```bash
eval "$(ssh-agent -s)"          # iniciar el agente (a menudo ya lanzado por el escritorio)
ssh-add ~/.ssh/id_ed25519       # añadir una clave
ssh-add -l                      # listar claves cargadas
ssh-add -t 3600 ~/.ssh/id_prod  # cargar durante 1 hora solo
ssh-add -D                      # descargar todas las claves
```

Para que las claves se añadan automáticamente, en `~/.ssh/config`:

```text
Host *
    AddKeysToAgent yes
```

#### La transferencia de agente

```bash
ssh -A servidor                  # transferir el agente al servidor
```

Esto permite, desde el servidor, rebotar hacia una tercera máquina con sus claves locales — sin jamás copiar allí una clave privada.

> [!CAUTION]
> **La transferencia de agente es arriesgada.** Mientras su sesión esté abierta, el administrador (o cualquier atacante que haya obtenido root) en la máquina intermedia puede usar el socket de su agente para autenticarse **en todas partes donde sus claves dan acceso**.
>
> No la active nunca globalmente (`ForwardAgent yes` bajo `Host *`). Prefiera **`ProxyJump`**, que alcanza el mismo resultado sin jamás exponer el agente en la máquina de rebote:
>
> ```bash
> ssh -J rebote destino
> ```

### Túneles y rebotes

```bash
# Rebote: alcanzar una máquina vía otra
ssh -J rebote@bastion admin@10.0.0.42

# Túnel LOCAL: exponer un servicio distante en mi máquina
# Aquí, la base distante (accesible solo localmente en el servidor)
# se hace alcanzable en localhost:5433
ssh -L 5433:localhost:5432 admin@servidor

# Túnel DISTANTE: exponer un servicio local en el servidor
ssh -R 8080:localhost:3000 admin@servidor

# Proxy SOCKS: hacer pasar un navegador por el servidor
ssh -D 1080 admin@servidor
```

```text
Túnel local  (-L)    mi máquina:5433  ──►  servidor  ──►  db:5432
Túnel distante (-R)   servidor:8080     ──►  mi máquina:3000
Proxy SOCKS   (-D)    mi máquina:1080  ──►  servidor  ──►  Internet
```

| Opción útil | Efecto |
|--------------|-------|
| `-N` | No ejecutar comando — solo túnel |
| `-f` | Pasar a segundo plano |
| `-T` | No asignar terminal |

```bash
ssh -fNT -L 5433:localhost:5432 admin@servidor      # túnel en segundo plano
```

El caso de uso típico: acceder a una base de datos que solo escucha en `127.0.0.1` del servidor, desde su cliente gráfico local — sin jamás exponerla en la red.

### Transferir archivos

```bash
# scp — simple, adaptado a transferencias puntuales
scp fichero.txt admin@servidor:/srv/data/
scp -r carpeta/ admin@servidor:/srv/
scp admin@servidor:/var/log/app.log ./

# rsync — incremental, retoma donde se detuvo: la opción correcta cuando se repite
rsync -avz --progress carpeta/ admin@servidor:/srv/carpeta/
rsync -avzAX --delete /srv/data/ admin@servidor:/backup/data/

# sftp — sesión interactiva
sftp admin@servidor
```

`rsync` se beneficia de su `~/.ssh/config`: `rsync -av carpeta/ web:/srv/` funciona con el alias `web`.

> [!TIP]
> Opciones `rsync` a conocer: `-a` (archive: permisos, fechas, enlaces), `-v` (verboso), `-z` (compresión), `-A` (**ACL**), `-X` (atributos extendidos), `--delete` (espejo exacto — **destructivo**, probar primero con `--dry-run`).

### fail2ban

`fail2ban` lee los logs, detecta intentos de autenticación repetidos y banea temporalmente las IP culpables a nivel de cortafuegos.

```bash
sudo apt install fail2ban
```

La configuración **nunca** se modifica en `jail.conf` (sobrescrita en actualizaciones) sino en `jail.local`:

```text
# /etc/fail2ban/jail.local

[DEFAULT]
bantime  = 1h
findtime = 10m
maxretry = 5
backend  = systemd
ignoreip = 127.0.0.1/8 192.168.1.0/24      # nunca banearse a sí mismo

[sshd]
enabled  = true
port     = ssh
maxretry = 3
bantime  = 24h
```

```bash
sudo systemctl enable --now fail2ban
sudo fail2ban-client status
sudo fail2ban-client status sshd
```

```text
Status for the jail: sshd
|- Filter
|  |- Currently failed: 2
|  |- Total failed:     1847
|  `- Journal matches:  _SYSTEMD_UNIT=sshd.service
`- Actions
   |- Currently banned: 5
   |- Total banned:     213
   `- Banned IP list:   203.0.113.66 198.51.100.4 …
```

```bash
sudo fail2ban-client set sshd unbanip 203.0.113.66     # desbanear
```

> [!IMPORTANT]
> Complete `ignoreip` con sus propias redes **antes** de activar el servicio. Banearse a sí mismo de su propio servidor por fail2ban tras tres errores de escritura es un gran clásico.
>
> Tenga en cuenta: si `PasswordAuthentication` ya está en `no`, lo esencial del ataque de fuerza bruta ya está neutralizado. fail2ban sigue siendo útil para reducir el ruido en los logs y la carga.

### Diagnóstico SSH

#### Del lado del cliente: el modo verboso

```bash
ssh -v username@servidor          # verboso
ssh -vvv username@servidor        # muy verboso
```

Las líneas a buscar en la salida:

```text
debug1: Offering public key: /home/alice/.ssh/id_ed25519 ED25519 SHA256:2Xk9…
debug1: Authentications that can continue: publickey,password
debug1: Server accepts key: /home/alice/.ssh/id_ed25519
```

#### Del lado del servidor: los logs

```bash
sudo journalctl -u ssh -f              # Debian
sudo journalctl -u sshd -f             # RHEL
sudo tail -f /var/log/auth.log
```

```text
sshd[1234]: Accepted publickey for alice from 203.0.113.5 port 51234 ssh2:
  ED25519 SHA256:2Xk9pQr7…Qw8
sshd[1235]: Failed password for invalid user admin from 198.51.100.4 port 40122 ssh2
sshd[1236]: Authentication refused: bad ownership or modes for file /home/bob/.ssh/authorized_keys
```

Para un diagnóstico realmente detallado, se puede lanzar un segundo `sshd` en modo debug en otro puerto, sin tocar el servicio en producción:

```bash
sudo /usr/sbin/sshd -d -p 2223          # muestra todo, una conexión luego se detiene
ssh -p 2223 username@servidor            # desde otro terminal
```

#### La lista de verificación `Permission denied (publickey)`

Este es el error más frecuente. En orden:

| # | Verificación | Comando |
|---|--------------|----------|
| 1 | ¿Se propone la clave? | `ssh -v` → buscar `Offering public key` |
| 2 | ¿Se usa la clave correcta? | `ssh -i ~/.ssh/id_ed25519 user@servidor` |
| 3 | ¿Está la clave pública en el servidor? | `grep "$(cut -d' ' -f2 ~/.ssh/id_ed25519.pub)" ~/.ssh/authorized_keys` |
| 4 | ¿Son correctos los permisos? | `ls -ld ~ ~/.ssh ~/.ssh/authorized_keys` |
| 5 | ¿Es el home escribible para el grupo? | `chmod g-w ~` |
| 6 | ¿Está autorizado el usuario? | `sudo sshd -T \| grep -i allow` |
| 7 | ¿Interfiere un bloque `Match`? | `sudo sshd -T -C user=alice,host=…,addr=…` |
| 8 | ¿Ha etiquetado mal SELinux `.ssh`? | `restorecon -Rv ~/.ssh` |
| 9 | ¿Está la cuenta bloqueada o expirada? | `sudo passwd -S alice` · `sudo chage -l alice` |

> [!TIP]
> El punto 8 es un gran clásico en RHEL: un `~/.ssh` creado por un script o copiado desde otro lugar lleva un contexto SELinux incorrecto, y `sshd` no puede leerlo — mientras que los permisos POSIX son perfectos. `restorecon -Rv ~/.ssh` resuelve el problema en un segundo. Ver la sección [SELinux](#selinux-rhel-fedora-centos-rocky).

#### Otros síntomas

| Mensaje | Causa probable |
|---------|----------------|
| `Connection refused` | `sshd` no escucha en este puerto, o cortafuegos en `REJECT` |
| `Connection timed out` | Cortafuegos en `DROP`, IP incorrecta, o grupo de seguridad nube |
| `Too many authentication failures` | El agente propone demasiadas claves → `IdentitiesOnly yes` |
| `Host key verification failed` | Clave de host cambió → verificar, luego `ssh-keygen -R` |
| `no matching host key type found` | Servidor antiguo, algoritmos obsoletos → `-o HostKeyAlgorithms=+ssh-rsa` |
| `Broken pipe` tras inactividad | Falta keepalive → `ServerAliveInterval 60` |

### Nombres de dominio personalizados — el archivo hosts

Se pueden crear nombres de dominio locales en el archivo hosts:

| OS | Ruta |
|----|--------|
| Linux / macOS | `/etc/hosts` |
| Windows | `C:\Windows\System32\drivers\etc\hosts` |

```text
192.168.1.42      bf.devops25
```

```bash
ssh username@bf.devops25
```

> [!CAUTION]
> **El archivo hosts tiene prioridad sobre DNS.** Si coloca:
> ```text
> 127.0.0.1      www.google.com
> ```
> cada intento de conexión a Google será redirigido a su propia máquina.
> Tenga cuidado por tanto con los nombres de dominio que asigna a sus servidores.
>
> Para un uso común, `~/.ssh/config` es preferible a `/etc/hosts`: no modifica la resolución de nombres de todo el sistema y no requiere derechos root.

### Resumen SSH

| Comando | Rol |
|----------|------|
| `ssh-keygen -t ed25519 -C "quien@máquina"` | Generar un par de claves |
| `ssh-copy-id user@servidor` | Desplegar su clave pública |
| `ssh-keygen -l -f cle.pub` | Huella dactilares de una clave |
| `ssh-keygen -R hote` | Retirar un host de `known_hosts` |
| `ssh-add -l` / `ssh-add -t 3600` | Gestionar el agente |
| `sshd -t` | **Validar `sshd_config` antes de recargar** |
| `sshd -T` | Mostrar la configuración efectiva |
| `ssh -v` | Diagnosticar una conexión |
| `ssh -J rebote destino` | Rebote (preferible a `-A`) |
| `ssh -fNT -L 5433:localhost:5432 srv` | Túnel local en segundo plano |
| `rsync -avzAX src/ srv:/dst/` | Sincronizar preservando las ACL |
| `fail2ban-client status sshd` | Ver los baneos |

**Los seis ajustes que importan en `sshd_config`:**

```text
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
AllowGroups sshusers
MaxAuthTries 3
LogLevel VERBOSE
```

---

## AppArmor / SELinux

### DAC contra MAC: ¿por qué una capa más?

Todo lo que hemos visto hasta ahora — permisos POSIX, ACL, sudoers — pertenece al control de acceso **discrecional** (DAC, *Discretionary Access Control*): es el **propietario** de un archivo quien decide quién puede acceder a él. Y por encima de todo eso, root puede todo.

AppArmor y SELinux añaden un control de acceso **obligatorio** (MAC, *Mandatory Access Control*): una política definida a nivel de sistema, aplicada por el **núcleo**, que ni el propietario de un archivo ni root pueden eludir sobre la marcha.

**El escenario concreto:**

> Se descubre una falla en su servidor web. Un atacante obtiene la ejecución de código arbitrario en el proceso `nginx`.
>
> - **Sin MAC**: el proceso corre como el usuario `www-data`. El atacante puede leer todo lo que `www-data` puede leer — por tanto `/etc/passwd`, archivos de otros sitios, copias de seguridad legibles por todos, y puede abrir conexiones de red salientes.
> - **Con MAC**: el núcleo solo autoriza el proceso `nginx` a leer `/var/www/`, escribir en `/var/log/nginx/` y escuchar en los puertos 80 y 443. Cualquier otro intento es rechazado **incluso si los permisos POSIX lo permitían**. La compromisión queda confinada.

Es una defensa en profundidad: no reemplaza los permisos, pone un techo a los daños cuando todo lo demás ha fallado.

| | AppArmor | SELinux |
|---|---|---|
| **Distribuciones** | Debian, Ubuntu, SUSE | RHEL, Fedora, CentOS, Rocky, Alma |
| **Enfoque** | Perfiles por **ruta** de binario | **Etiquetas** (*labels*) en cada archivo y proceso |
| **Unidad de política** | Un archivo de perfil por ejecutable | Una política global + contextos |
| **Si el archivo se mueve** | El perfil ya no se aplica | La etiqueta sigue el archivo |
| **Complejidad** | Legible, rápido de aprender | Mucho más fino, pero más abrupto |
| **Archivos de config** | `/etc/apparmor.d/` | `/etc/selinux/`, etiquetas en el sistema de archivos |

### Saber qué corre en la máquina

```bash
# ¿AppArmor presente y activo?
sudo aa-status
cat /sys/module/apparmor/parameters/enabled       # Y o N

# ¿SELinux presente y activo?
sestatus
getenforce
```

---

## AppArmor (Debian, Ubuntu, SUSE)

### 1. Verificar el estado

```bash
sudo aa-status
```

```text
apparmor module is loaded.
32 profiles are loaded.
29 profiles are in enforce mode.
   /usr/bin/man
   /usr/sbin/nginx
   /usr/sbin/tcpdump
   ...
3 profiles are in complain mode.
   /usr/local/bin/backup.sh
   ...
2 processes have profiles defined.
2 processes are in enforce mode.
   /usr/sbin/nginx (1234)
   /usr/sbin/nginx (1235)
0 processes are in complain mode.
0 processes are unconfined but have a profile defined.
```

### 2. Los tres modos

| Modo | Comportamiento | Comando |
|------|----------------|---------|
| **enforce** | Bloquea todo lo que no está explícitamente permitido, y lo registra | `sudo aa-enforce /usr/sbin/nginx` |
| **complain** | No impide nada, pero **registra** todo lo que habría sido bloqueado | `sudo aa-complain /usr/sbin/nginx` |
| **disabled** | El perfil se ignora | `sudo aa-disable /usr/sbin/nginx` |

> [!TIP]
> El modo **complain** es la herramienta central del trabajo con AppArmor: se pone un perfil en complain, se ejecuta la aplicación con normalidad durante unos días, se recopila todo lo que ha intentado hacer realmente y luego se pasa a enforce. Nunca se adivina un perfil, se observa.

Estos comandos provienen del paquete:

```bash
sudo apt install apparmor-utils apparmor-profiles
```

### 3. Anatomía de un perfil

Los perfiles viven en `/etc/apparmor.d/`. El nombre del archivo reproduce la ruta del binario, con las `/` sustituidas por `.`:

| Binario | Archivo de perfil |
|---------|-------------------|
| `/usr/sbin/nginx` | `/etc/apparmor.d/usr.sbin.nginx` |
| `/usr/local/bin/backup.sh` | `/etc/apparmor.d/usr.local.bin.backup.sh` |

Aquí tienes un perfil completo y comentado, que confina un script de copia de seguridad casero:

```text
# /etc/apparmor.d/usr.local.bin.backup.sh

abi <abi/3.0>,
include <tunables/global>

# La ruta del programa confinado, seguida del bloque de reglas
/usr/local/bin/backup.sh {

  # ── Abstracciones: bloques de reglas reutilizables proporcionados por el sistema
  include <abstractions/base>          # lo mínimo vital (libc, /dev/null, locales…)
  include <abstractions/bash>          # lo que necesita un script bash

  # ── Capabilities: los privilegios finos de root que el programa necesita
  capability dac_read_search,          # leer archivos a pesar de los permisos
  capability chown,

  # ── El propio script y sus intérpretes
  /usr/local/bin/backup.sh r,
  /bin/bash                ix,         # ejecutado DENTRO de este mismo perfil
  /usr/bin/tar             mrix,
  /usr/bin/gzip            mrix,
  /usr/bin/find            mrix,

  # ── Los datos: solo lectura en el origen
  /srv/data/           r,
  /srv/data/**         r,

  # ── El destino: lectura y escritura
  /var/backups/        rw,
  /var/backups/**      rw,

  # ── El registro
  /var/log/backup.log  w,

  # ── Prohibiciones explícitas (prioritarias sobre cualquier autorización)
  deny /etc/shadow        rwklx,
  deny /home/*/.ssh/**    rwklx,
  deny /root/**           rwklx,
}
```

**Las letras de permiso**, que constituyen lo esencial de la sintaxis:

| Letra | Significado |
|-------|-------------|
| `r` | Lectura |
| `w` | Escritura |
| `a` | Solo adición al final del archivo (*append*) |
| `k` | Bloqueo del archivo |
| `l` | Creación de enlaces |
| `m` | Proyección en memoria ejecutable (*mmap* — necesaria para binarios y bibliotecas) |
| `ix` | Ejecutar **heredando** el perfil actual |
| `Px` | Ejecutar bajo **su propio** perfil (que debe existir), con limpieza del entorno |
| `Cx` | Ejecutar bajo un subperfil definido en este archivo |
| `Ux` | Ejecutar **sin confinamiento** — a evitar, es un agujero en la política |

**Los comodines de ruta:**

| Patrón | Coincide con |
|--------|--------------|
| `/srv/data/*` | Los archivos **directamente** en `/srv/data/` |
| `/srv/data/**` | Todo, **de forma recursiva**, bajo `/srv/data/` |
| `/home/*/.config/` | El `.config` de cualquier usuario |
| `/var/log/app-[0-9].log` | Una clase de caracteres |

> [!IMPORTANT]
> **Cada regla termina con una coma.** Es el error de sintaxis n.º 1 en los perfiles de AppArmor, y `apparmor_parser` lo señala con un número de línea poco explícito.
>
> En las versiones anteriores a AppArmor 3, la sintaxis de inclusión es `#include <abstractions/base>` (con el `#`, que aquí **no** es un comentario). La forma moderna `include <…>` es equivalente.

Las abstracciones disponibles se encuentran en `/etc/apparmor.d/abstractions/` — hay para casi todo:

```bash
ls /etc/apparmor.d/abstractions/
```

```text
apache2-common  base  bash  consoles  dbus  fonts  nameservice
nis  openssl  perl  php  python  ssl_certs  user-tmp  wutmp  ...
```

### 4. Leer una denegación en los logs

Es el reflejo que hay que adquirir: cuando una aplicación confinada se comporta de forma extraña, la respuesta está en los logs del kernel.

```bash
sudo dmesg | grep -i apparmor
```

```bash
sudo journalctl -k | grep -i "apparmor=\"DENIED\""
```

O, si `auditd` está instalado:

```bash
sudo grep apparmor /var/log/audit/audit.log
```

Una denegación se lee así:

```text
audit: type=1400 audit(1765108800.123:456): apparmor="DENIED" operation="open"
  profile="/usr/sbin/nginx" name="/srv/www/index.html" pid=1234 comm="nginx"
  requested_mask="r" denied_mask="r" fsuid=33 ouid=0
```

| Campo | Lo que te dice |
|-------|----------------|
| `apparmor="DENIED"` | El acceso fue **bloqueado** (`ALLOWED` en modo complain) |
| `operation="open"` | Lo que el programa intentaba hacer |
| `profile="/usr/sbin/nginx"` | **Qué perfil** bloqueó |
| `name="/srv/www/index.html"` | **Qué recurso** se pretendía |
| `requested_mask="r"` | El derecho solicitado — aquí la lectura |

La regla que falta se deduce directamente de las dos últimas filas: `/srv/www/** r,`.

### 5. Crear un perfil, método completo

Es el flujo de trabajo estándar, en cinco pasos.

```bash
# 1. Generar un esqueleto de perfil y lanzar la aplicación bajo vigilancia
sudo aa-genprof /usr/local/bin/backup.sh
```

`aa-genprof` se queda a la espera. En **una segunda terminal**, se ejecuta la aplicación en todas sus condiciones de uso:

```bash
# 2. Ejercitar la aplicación: todos los casos de uso, incluidos los casos de error
/usr/local/bin/backup.sh
/usr/local/bin/backup.sh --full
```

```bash
# 3. Volver a la primera terminal y pulsar "S" (Scan).
#    aa-genprof propone cada acceso observado: Allow / Deny / Glob / Abort.
#    Se responde una vez por acceso, luego "F" (Finish) para guardar.
```

```bash
# 4. Dejar funcionar en complain unos días, luego recoger lo que falta
sudo aa-logprof
```

```bash
# 5. Pasar a enforce cuando ya no aparezca ninguna denegación
sudo aa-enforce /usr/local/bin/backup.sh
```

> [!TIP]
> `aa-logprof` es el comando que conviene relanzar periódicamente: relee los logs, presenta cada nuevo acceso observado y actualiza el perfil. Es él quien hace el grueso del trabajo — rara vez se escribe un perfil enteramente a mano.

### 6. Modificar un perfil suministrado por un paquete

Editar directamente `/etc/apparmor.d/usr.sbin.nginx` funciona, pero tus modificaciones entrarán en conflicto en la próxima actualización del paquete. El mecanismo previsto para ello es el directorio `local/`:

```bash
sudo nano /etc/apparmor.d/local/usr.sbin.nginx
```

```text
# Permitir a nginx servir un docroot personalizado
/srv/monsitio/** r,
/var/log/monsitio/*.log w,
```

El perfil principal ya contiene, al final del bloque, la línea `include <local/usr.sbin.nginx>` que carga este archivo.

### 7. Recargar

```bash
# Recargar un perfil concreto
sudo apparmor_parser -r /etc/apparmor.d/usr.sbin.nginx

# Verificar la sintaxis sin cargar
sudo apparmor_parser -Q /etc/apparmor.d/usr.sbin.nginx

# Recargar todo
sudo systemctl reload apparmor
```

> [!CAUTION]
> Un perfil de AppArmor solo se aplica a un proceso **en el momento de su ejecución**. Recargar el perfil de un servicio ya iniciado no reconfina el proceso en curso: hay que reiniciar el servicio.
>
> ```bash
> sudo apparmor_parser -r /etc/apparmor.d/usr.sbin.nginx
> sudo systemctl restart nginx
> ```

---

## SELinux (RHEL, Fedora, CentOS, Rocky)

### 1. Los modos y el estado

```bash
sestatus
```

```text
SELinux status:                 enabled
SELinuxfs mount:                /sys/fs/selinux
Loaded policy name:             targeted
Current mode:                   enforcing
Mode from config file:          enforcing
Policy MLS status:              enabled
Policy deny_unknown status:     allowed
Max kernel policy version:      33
```

| Modo | Comportamiento |
|------|----------------|
| **Enforcing** | Aplica la política y bloquea |
| **Permissive** | No impide nada, pero registra todo lo que habría sido bloqueado |
| **Disabled** | SELinux está completamente desactivado |

```bash
getenforce                  # modo actual
sudo setenforce 0           # pasar a permisivo — TEMPORAL, se pierde al reiniciar
sudo setenforce 1           # volver a enforcing
```

El modo permanente está en `/etc/selinux/config`:

```text
# /etc/selinux/config
SELINUX=enforcing
# SELINUX puede valer: enforcing | permissive | disabled
SELINUXTYPE=targeted
```

> [!CAUTION]
> **Nunca pongas `SELINUX=disabled` para «hacer funcionar» una aplicación.** Es la respuesta equivocada a un problema real, y tiene un coste oculto: mientras SELinux está desactivado, los archivos nuevos no reciben etiqueta. El día que lo reactivas, nada arranca, y hay que reetiquetar todo el sistema:
>
> ```bash
> sudo touch /.autorelabel && sudo reboot     # puede tardar largos minutos
> ```
>
> Si tienes que depurar, usa **`permissive`**: la política sigue registrando todo lo que habría bloqueado, sin romper la producción, y la vuelta a `enforcing` es inmediata.

### 2. El contexto de seguridad

SELinux etiqueta **todo**: archivos, procesos, puertos, usuarios. Una etiqueta tiene cuatro campos:

```text
system_u : object_r : httpd_sys_content_t : s0
   │          │              │              │
 user      role           TYPE           nivel MLS
```

> [!IMPORTANT]
> En la política `targeted` (la de todas las distribuciones por defecto), **solo cuenta el `type`** en la práctica. Los demás campos solo entran en juego en configuraciones multinivel (MLS), muy raras. Concéntrate en el campo que termina en `_t`.

Los contextos se leen con la opción `-Z`, presente en la mayoría de los comandos:

```bash
ls -Z /var/www/html
```

```text
unconfined_u:object_r:httpd_sys_content_t:s0 index.html
```

```bash
ps -eZ | grep nginx
```

```text
system_u:system_r:httpd_t:s0    1234 ?  00:00:00 nginx
```

```bash
id -Z                      # mi propio contexto
netstat -Z / ss -Z         # contexto de los sockets
```

### 3. El principio del Type Enforcement

La política es una lista de reglas de la forma: *un proceso de tipo A puede realizar tal acción sobre un objeto de tipo B*.

```text
Proceso nginx    →  httpd_t
Archivo servido  →  httpd_sys_content_t
Regla            →  allow httpd_t httpd_sys_content_t : file { read getattr open };
```

Consecuencia directa, y causa del 90 % de los problemas encontrados: **un archivo creado en el lugar equivocado o movido con `mv` lleva la etiqueta equivocada**, y el servicio no puede leerlo — ni siquiera con `chmod 777`.

```bash
# La trampa clásica: cp respeta el contexto de destino, mv conserva el de origen
cp archivo.html /var/www/html/      # ✅ toma httpd_sys_content_t
mv ~/archivo.html /var/www/html/    # ❌ conserva user_home_t → 403 Forbidden
```

### 4. Cambiar un contexto

**Temporalmente** — se pierde en el próximo reetiquetado:

```bash
sudo chcon -t httpd_sys_content_t /var/www/html/index.html
sudo chcon -R -t httpd_sys_content_t /var/www/html/
```

**Definitivamente** — se registra la regla en la política y luego se aplica:

```bash
# 1. Declarar la regla para esta ruta (y todo lo que contiene)
sudo semanage fcontext -a -t httpd_sys_content_t "/srv/monsitio(/.*)?"

# 2. Aplicar la regla a los archivos ya presentes
sudo restorecon -Rv /srv/monsitio
```

| Comando | Alcance |
|---------|---------|
| `chcon` | Cambia la etiqueta **ahora**, sin memoria — un `restorecon` la sobrescribe |
| `semanage fcontext -a` | Registra la regla **en la política** — es la fuente de verdad |
| `restorecon` | Reaplica a los archivos lo que dice la política |

> [!TIP]
> **El reflejo que hay que recordar: `semanage fcontext` y luego `restorecon`.** `chcon` es práctico para probar una hipótesis en treinta segundos, pero nunca debe quedarse en un procedimiento de instalación.
>
> `semanage` viene del paquete `policycoreutils-python-utils` (`sudo dnf install policycoreutils-python-utils`).

Consultar las reglas registradas:

```bash
sudo semanage fcontext -l | grep monsitio
```

### 5. Los booleanos

Muchos comportamientos habituales ya están previstos por la política y se controlan con simples interruptores. Es la solución más limpia para un gran número de bloqueos.

```bash
getsebool -a | grep httpd
```

```text
httpd_can_network_connect --> off
httpd_can_network_connect_db --> off
httpd_can_sendmail --> off
httpd_enable_homedirs --> off
httpd_use_nfs --> off
```

```bash
# -P hace el cambio permanente (persistente tras reiniciar)
sudo setsebool -P httpd_can_network_connect on
```

Algunos booleanos que se encuentran muy a menudo:

| Booleano | Activar cuando… |
|----------|-----------------|
| `httpd_can_network_connect` | El servidor web debe llamar a una API o a un backend (reverse proxy) |
| `httpd_can_network_connect_db` | El servidor web se conecta a una base de datos remota |
| `httpd_enable_homedirs` | El servidor web sirve archivos desde `/home` |
| `ftpd_full_access` | El servidor FTP debe acceder a todo el sistema de archivos |
| `nis_enabled` | Autenticación mediante NIS/LDAP en algunos casos |

### 6. Los puertos

SELinux también etiqueta los puertos. Hacer que un servicio escuche en un puerto no estándar exige, por tanto, declararlo.

```bash
# ¿Qué puertos están permitidos para el tipo ssh_port_t?
sudo semanage port -l | grep ssh
```

```text
ssh_port_t                     tcp      22
```

```bash
# Permitir a sshd escuchar en el puerto 2222
sudo semanage port -a -t ssh_port_t -p tcp 2222
```

```bash
# Ídem para un servidor web en el puerto 8080
sudo semanage port -a -t http_port_t -p tcp 8080
```

> [!IMPORTANT]
> Cambiar el puerto de un servicio exige **tres** acciones coherentes, y es fácil olvidar una:
> 1. modificar la configuración del servicio (`Port 2222` en `sshd_config`);
> 2. abrir el puerto en el cortafuegos (`firewall-cmd --permanent --add-port=2222/tcp`);
> 3. declarar el puerto a SELinux (`semanage port -a`).

### 7. Diagnosticar una denegación

Las denegaciones de SELinux se llaman **AVC denials** (*Access Vector Cache*).

```bash
sudo ausearch -m AVC -ts recent
```

```text
type=AVC msg=audit(1765108800.123:456): avc:  denied  { read } for  pid=1234
  comm="nginx" name="index.html" dev="sda1" ino=12345
  scontext=system_u:system_r:httpd_t:s0
  tcontext=unconfined_u:object_r:user_home_t:s0
  tclass=file permissive=0
```

El mensaje se lee como una frase:

| Campo | Lectura |
|-------|---------|
| `denied { read }` | La acción denegada |
| `comm="nginx"` | El programa afectado |
| `scontext=…httpd_t` | **Origen**: el tipo del proceso |
| `tcontext=…user_home_t` | **Destino**: el tipo del objeto pretendido — ahí está la anomalía |
| `tclass=file` | La clase de objeto |
| `permissive=0` | 0 = realmente bloqueado; 1 = solo registrado |

Aquí está todo dicho: un proceso `httpd_t` intentó leer un archivo que se quedó etiquetado como `user_home_t`. La corrección es un `restorecon`.

**Una ayuda más legible** la proporciona `setroubleshoot`:

```bash
sudo dnf install setroubleshoot-server
sudo sealert -a /var/log/audit/audit.log
```

`sealert` traduce cada denegación a lenguaje corriente y propone el o los comandos correctivos — a menudo el booleano adecuado o el `semanage fcontext` adecuado.

**Como último recurso**, cuando ningún booleano ni ninguna etiqueta existente es adecuado, se genera un módulo de política a medida:

```bash
# Examinar lo que permitiría la regla generada — A LEER antes de aplicar
sudo ausearch -m AVC -ts recent | audit2allow -m monapp

# Generar e instalar el módulo
sudo ausearch -m AVC -ts recent | audit2allow -M monapp
sudo semodule -i monapp.pp
```

> [!WARNING]
> `audit2allow` autoriza **exactamente** lo que fue denegado, sin juzgar. Si una denegación proviene de un intento real de intrusión o de una aplicación mal configurada, acabas de inscribir el fallo en la política.
> Lee siempre la salida de `audit2allow -m` antes de instalar el módulo, y no recurras a él sino después de haber descartado los booleanos y el reetiquetado.

---

## Método de diagnóstico

Ante un servicio que se niega a acceder a un recurso aunque los permisos POSIX parezcan correctos:

| Paso | AppArmor | SELinux |
|------|----------|---------|
| **1. Confirmar que es realmente el MAC** | `sudo aa-complain /usr/sbin/nginx` y volver a probar | `sudo setenforce 0` y volver a probar |
| **2. Leer la denegación** | `sudo dmesg \| grep DENIED` | `sudo ausearch -m AVC -ts recent` |
| **3. Corregir correctamente** | Añadir la regla en `/etc/apparmor.d/local/…` | Booleano (`setsebool -P`), etiqueta (`semanage fcontext` + `restorecon`) o puerto (`semanage port -a`) |
| **4. Volver a cerrar** | `sudo aa-enforce /usr/sbin/nginx` | `sudo setenforce 1` |

> [!CAUTION]
> El paso 4 no es opcional. Un `setenforce 0` « solo para ver » que nunca se revierte es el modo de fallo más común de SELinux en producción: la máquina funciona meses sin ninguna protección y nadie se da cuenta.
>
> Verificación a integrar en sus controles regulares:
> ```bash
> getenforce                      # debe responder Enforcing
> sudo aa-status | head -3        # debe anunciar perfiles en modo enforce
> ```

> [!TIP]
> Cuando un servicio se niega obstinadamente a acceder a un archivo aunque los permisos POSIX sean correctos, piense en AppArmor/SELinux **antes** de buscar en otro lado. El modo *permissive* / *complain* permite confirmar el diagnóstico en treinta segundos, sin desactivar la protección.

---

## Resumen

| Dominio | Comandos clave |
|---------|----------------|
| **ACL** | `getfacl`, `setfacl -m`, `setfacl -d -m`, `setfacl -x`, `setfacl -b`, `namei -l` |
| **Sudo** | `sudo -l`, `sudoedit`, `visudo`, `visudo -c`, `/etc/sudoers.d/`, `sudoreplay` |
| **Cortafuegos** | `ufw allow/deny/limit`, `firewall-cmd --permanent`, `iptables -L -v -n`, `nft list ruleset`, `ss -tulpn` |
| **SSH** | `ssh-keygen -t ed25519`, `ssh-copy-id`, `sshd -t`, `sshd -T`, `ssh -v`, `~/.ssh/config` |
| **AppArmor** | `aa-status`, `aa-complain`, `aa-enforce`, `aa-genprof`, `aa-logprof`, `apparmor_parser -r` |
| **SELinux** | `sestatus`, `getenforce`, `ls -Z`, `semanage fcontext`, `restorecon`, `setsebool -P`, `ausearch`, `sealert` |

### Los reflejos básicos en un servidor expuesto

1. **SSH**: sin conexión root directa, sin autenticación por contraseña, claves `ed25519` con contraseña.
2. **Cortafuegos**: activo, en lista blanca — todo cerrado excepto lo estrictamente necesario.
3. **Sudo**: derechos explícitos y nominativos, lista blanca de comandos, nunca `NOPASSWD: ALL`, `sudoedit` antes que `sudo vim`.
4. **Archivos**: las ACL para los casos finos, nunca `chmod 777` — y un `namei -l` antes de concluir que un permiso está roto.
5. **MAC**: AppArmor o SELinux dejado en *enforcing*, nunca desactivado para esquivar un problema.
6. **Actualizaciones** de seguridad aplicadas regularmente.
7. **Logs** consultados — `auth.log`, `journalctl -t sudo`, `fail2ban-client status` — y no solo escritos.

### El reflejo que atraviesa todo el capítulo

> [!TIP]
> **Mantenga siempre una segunda sesión abierta.**
>
> Ya sea que modifique `sshd_config`, el cortafuegos o el sudoers, la manipulación puede cortarle el acceso a la máquina. Una sesión ya autenticada, dejada abierta en paralelo, no se ve afectada por ninguno de estos cambios: es su puerta de escape, y cuesta diez segundos de abrir.
>
> Complétela, en operaciones de red, con una remisión a cero programada:
>
> ```bash
> echo "ufw --force reset" | sudo at now + 5 minutes
> ```
>
> Y verifique **antes** de recargar, cada vez que una herramienta lo permita:
>
> ```bash
> sudo sshd -t              # SSH
> sudo visudo -c            # sudoers
> sudo nft -c -f /etc/nftables.conf   # nftables
> sudo apparmor_parser -Q /etc/apparmor.d/monperfil
> ```

---

⬅️ [Anterior: 04 · Instalación y servicios](04_instalacion_y_servicios_es.md) · 🏠 [Tabla de contenidos](README_es.md) · [Siguiente: 06 · Shell script ➡️](06_shell_script_es.md)
