Esos cuatro comandos preparan la "máquina falsa" que da por hecho el ejercicio: los grupos, los usuarios y las carpetas de `srv-tickets`. Casi todo está explicado en [02.1_user_management.md](02.1_user_management.md) y [01_base.md](01_base.md). Lo único que el repo no explica es `mkdir -p` y la carpeta `/srv`.

## 1. Crear los grupos

```bash
sudo groupadd ops; sudo groupadd dev
```

- **`groupadd nombre`** crea un grupo nuevo. Se explica en la sección "Gestion des groupes" de [02.1_user_management.md:177](02.1_user_management.md#L177).
- **`sudo`** hace falta porque crear grupos es tarea de administración. El capítulo aconseja usar `sudo commande` antes que abrir una sesión root ([02.1_user_management.md:227](02.1_user_management.md#L227)).
- **`;`** separa dos comandos en la misma línea y los ejecuta uno detrás de otro.

¿Por qué estos dos grupos? Porque aparecen en el ejercicio:
- **`ops`** es el grupo de `backup.sh` (`-rwxr-x--- root ops`) y luego de `/srv/ticketflow` (ejercicio 2.6).
- **`dev`** es el grupo de `/srv/projet` (`drwxrws--- root dev`), que se usa para el SGID del 2.5.

Para comprobarlo: `cat /etc/group` o `getent group ops dev`.

## 2. Crear los usuarios

```bash
sudo useradd -m alice; sudo useradd -m bob; sudo useradd -m deploy
```

- **`useradd nombre`** crea el usuario ([02.1_user_management.md:29](02.1_user_management.md#L29)).
- **`-m`** fuerza la creación de su *home directory* (`/home/alice`, etc.). Según la tabla del capítulo, sin `-m` el usuario puede quedarse sin carpeta personal. La necesitas para el ejercicio 2.6.3 (reparar los permisos del home de un desarrollador).
- El capítulo explica que `adduser` es la versión interactiva (pide contraseña, etc.) y que **`useradd` es mejor para scripts** porque no hace preguntas. Por eso lo uso aquí.

¿Para qué sirve cada usuario?
- **`alice`** y **`bob`**: el ejercicio 2.5 pide verificar el SGID y el sticky bit **"con dos cuentas diferentes"**, y en el 2.3 te sirven como "usuario de prueba".
- **`deploy`**: el ejercicio 2.6.1 pide dar `/srv/ticketflow` al usuario `deploy`.

> [!NOTE]
> Estos usuarios no tienen contraseña. No importa, porque cambiarás a ellos con `sudo`: `sudo -u alice comando` o `sudo su - alice`. Ambas formas salen en [02.1_user_management.md:199](02.1_user_management.md#L199).

## 3. Añadir alice y bob al grupo `dev`

```bash
sudo usermod -aG dev alice; sudo usermod -aG dev bob
```

- **`usermod -aG grupo usuario`** añade el usuario a un grupo secundario ([02.1_user_management.md:83](02.1_user_management.md#L83)).
- **`-G`** indica los grupos secundarios.
- **`-a`** (*append*) es **obligatorio**. El capítulo lo avisa: sin `-a`, `usermod -G` **sustituye** todos los grupos del usuario y lo saca de los demás, incluido `sudo`.

¿Por qué los dos en `dev`? Para el ejercicio 2.5: los dos deben poder escribir en `/srv/projet`. Así verás que, gracias al SGID, sus archivos quedan en el grupo `dev` y no en el grupo personal de cada uno. Y gracias al sticky bit, alice no podrá borrar los archivos de bob.

Para comprobarlo: `groups alice` o `id bob` ([02.1_user_management.md:61](02.1_user_management.md#L61)).

## 4. Crear las carpetas

```bash
sudo mkdir -p /srv/ticketflow/backups /srv/projet /var/log/ticketflow
```

- **`mkdir`** crea carpetas ([01_base.md:135](01_base.md#L135)). Se le pueden pasar varias rutas a la vez.
- **`-p`** no está en la documentación del repo. Hace dos cosas:
  1. Crea las carpetas padre que falten. Aquí crea `/srv/ticketflow` y después `backups` dentro.
  2. No da error si la carpeta ya existe.
- **`sudo`** hace falta porque `/srv` y `/var` son del sistema y solo root puede escribir ahí ([01_base.md:113](01_base.md#L113)).

¿Por qué estas rutas? Todas salen en el ejercicio:

| Carpeta | Para qué | Ejercicio |
|---------|----------|-----------|
| `/srv/ticketflow/backups` | Permisos de un directorio de backups | 2.2 |
| `/srv/ticketflow` | `chmod` recursivo, `find`, `chown deploy:ops` | 2.4, 2.6 |
| `/srv/projet` | SGID y sticky bit | 2.5 |
| `/var/log/ticketflow` | Permisos de logs, `chgrp adm` | 2.2, 2.6 |

- **`/var`** aparece en la tabla del sistema de archivos de [01_base.md](01_base.md) como "datos que crecen con el tiempo", y `/var/log` es donde se guardan los logs.
- **`/srv`** no está en esa tabla. Es la carpeta estándar de Linux para los **datos que sirve la máquina** (aplicaciones, webs, proyectos). Por eso el ejercicio pone ahí `ticketflow` y `projet`.

## Comprobar que todo quedó bien

```bash
getent group ops dev          # grupos creados, con sus miembros
id alice; id bob; id deploy   # usuarios y sus grupos
ls -ld /srv/ticketflow /srv/ticketflow/backups /srv/projet /var/log/ticketflow
```

Con el último comando verás que las carpetas son de `root root` con permisos `drwxr-xr-x` (755). Ese es el punto de partida: en el ejercicio tendrás que cambiarlos con `chmod`, `chown` y `chgrp`.
---

## 1. Solo los usuarios reales (UID ≥ 1000)

```bash
awk -F: '$3 >= 1000 && $3 < 60000 {print $1}' /etc/passwd
# → stgadmin
```

**Desglose**

| Pieza | Qué es | Por qué |
|---|---|---|
| `awk` | El programa que procesa texto por columnas | Cada línea de `/etc/passwd` es un registro con campos |
| `-F:` | Separador de campos = `:` | `/etc/passwd` separa sus campos con `:`, no con espacios |
| `'...'` | Comillas simples alrededor del programa | Para que bash no interprete `$3` como una variable suya: así llega intacto a `awk` |
| `$3` | El tercer campo de la línea | En `/etc/passwd`, el campo 3 es el **UID** |
| `$3 >= 1000` | Condición: UID mayor o igual que 1000 | Los usuarios normales empiezan en 1000 (`UID_MIN` en `/etc/login.defs`) |
| `&&` | Y lógico: las dos condiciones deben cumplirse | |
| `$3 < 60000` | Condición: UID menor que 60000 | Para excluir a `nobody` (UID 65534), que es de sistema aunque tenga un número alto |
| `{print $1}` | Acción: mostrar el campo 1 | El campo 1 es el **nombre de usuario** |
| `/etc/passwd` | El archivo que se lee | Es donde están todos los usuarios |

**Cómo lo procesa con tu línea:**
```
stgadmin:x:1000:1000:,,,:/home/stgadmin:/bin/bash
   $1    $2  $3   $4  $5       $6          $7

$3 = 1000 → ¿1000 >= 1000? sí · ¿1000 < 60000? sí → se cumple → print $1 → stgadmin
```

```
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
$3 = 33 → ¿33 >= 1000? no → no se muestra
```

---

## 2. Usuarios que pueden iniciar sesión

```bash
awk -F: '$7 ~ /(bash|zsh|sh)$/ {print $1, $7}' /etc/passwd
# → root /bin/bash
#   stgadmin /bin/bash
```

**Desglose**

| Pieza | Qué es | Por qué |
|---|---|---|
| `-F:` | Separador `:` | Igual que antes |
| `$7` | El séptimo campo | En `/etc/passwd`, el campo 7 es el **shell** |
| `~` | Operador "coincide con la expresión regular" | Para comprobar un patrón, no un valor exacto (`==` exigiría el texto exacto) |
| `/.../` | Delimitadores de la expresión regular en `awk` | Como en `sed` o `grep` |
| `(bash\|zsh\|sh)` | Paréntesis con `\|` = "una de estas opciones" | Los shells reales más habituales |
| `$` (dentro de la regex) | "Al final del texto" | Para que coincida con `/bin/bash` pero no con algo que solo contenga "sh" en medio |
| `{print $1, $7}` | Mostrar el nombre y el shell | La coma inserta un espacio entre los dos |

**Por qué no sale `sync`:** su shell es `/bin/sync`, que termina en `sync`, no en `sh`. Tampoco salen los de `/usr/sbin/nologin` ni `/bin/false`.

**El `$` tiene dos significados distintos aquí**, según dónde esté:
- `$7` → "el campo 7" (sintaxis de `awk`)
- `...sh)$` → "final de línea" (sintaxis de las expresiones regulares)

---

## 3. Solo los grupos creados por usuarios (GID ≥ 1000)

```bash
awk -F: '$3 >= 1000 && $3 < 60000 {print $1}' /etc/group
# → stgadmin
#   ops
#   dev
```

**Desglose**

Es **el mismo comando que el n.º 1**, cambiando el archivo:

| Pieza | Diferencia con el n.º 1 |
|---|---|
| `/etc/group` | Ahora se lee el archivo de grupos |
| `$3` | En `/etc/group`, el campo 3 es el **GID** (no el UID) |
| `$1` | El nombre del **grupo** |

**Formato de `/etc/group`:**
```
dev:x:1002:
 $1 $2  $3  $4 (vacío: sin miembros)
```

**Por qué no sale `docker`**, aunque lo instalaste tú: Docker crea su grupo como **grupo de sistema** (GID 989), por debajo de 1000.

---

## 4. Solo los grupos que tienen miembros

```bash
awk -F: '$4 != "" {print $1 ": " $4}' /etc/group
# → adm: syslog,stgadmin
#   cdrom: stgadmin
#   ...
```

**Desglose**

| Pieza | Qué es | Por qué |
|---|---|---|
| `$4` | El cuarto campo | En `/etc/group`, es la **lista de miembros** |
| `!=` | "Distinto de" | |
| `""` | Texto vacío | `$4 != ""` = "el campo de miembros no está vacío" |
| `print $1 ": " $4` | Mostrar el nombre, el texto `": "` y los miembros | Sin comas: en `awk`, poner cosas una al lado de otra las **pega** sin espacio. El `": "` es texto literal entre comillas dobles |

**Diferencia entre `print $1, $4` y `print $1 ": " $4`:**
```
print $1, $4         → adm syslog,stgadmin    (la coma pone un espacio)
print $1 ": " $4     → adm: syslog,stgadmin   (pega los tres trozos)
```

**Recuerda:** el campo 4 solo muestra los miembros **secundarios**. Por eso el grupo `stgadmin` no aparece, aunque tú eres su miembro principal.

---

## 5. Solo tus grupos

```bash
groups
# → stgadmin adm cdrom sudo dip plugdev users docker
```

No es compuesto: `groups` sin argumentos muestra los grupos del usuario actual. El primero es tu grupo principal.

Un detalle que se ve en tu salida: **`ops` y `dev` están vacíos** (`ops:x:1001:` y `dev:x:1002:`). Si los creaste para un ejercicio, te falta añadir los usuarios:
```bash
sudo usermod -aG dev usuario
```

| Pieza | Por qué |
|---|---|
| `sudo` | Modificar usuarios requiere ser administrador |
| `usermod` | Modifica un usuario que ya existe |
| `-a` | *Append*: **añade** el grupo sin quitar los que ya tiene. Sin `-a`, perderías los demás grupos, incluido `sudo` |
| `-G dev` | El grupo secundario que se añade |
| `usuario` | A quién se añade |

---

## 6. Los alias en `~/.bashrc`

Este es el más complejo, porque mezcla tres cosas: un *here document*, alias y caracteres escapados.

```bash
cat >> ~/.bashrc << 'EOF'

# Usuarios y grupos sin el ruido del sistema
alias usuarios="awk -F: '\$3 >= 1000 && \$3 < 60000 {print \$1}' /etc/passwd"
alias grupos="awk -F: '\$3 >= 1000 && \$3 < 60000 {print \$1 \": \" \$4}' /etc/group"
alias gruposconmiembros="awk -F: '\$4 != \"\" {print \$1 \": \" \$4}' /etc/group"
EOF

source ~/.bashrc
```

### Parte A: escribir en el archivo (`cat >> ... << 'EOF'`)

| Pieza | Qué es | Por qué |
|---|---|---|
| `cat` | Muestra lo que recibe | Aquí recibe el texto que hay entre las dos marcas `EOF` |
| `>>` | Redirige la salida **añadiéndola al final** del archivo | Con un solo `>` **sobrescribirías** el `.bashrc` y perderías tu configuración |
| `~/.bashrc` | El archivo de configuración de bash | Se ejecuta cada vez que abres una terminal: los alias quedan permanentes |
| `<<` | *Here document*: "el texto de entrada va a continuación" | Para escribir varias líneas de una vez |
| `'EOF'` | La marca de fin, **entre comillas simples** | Las comillas hacen que bash **no interprete nada** del texto (ni los `$` ni las `\`): se escribe tal cual |
| `EOF` (al final, solo en su línea) | Cierra el texto | Puede ser cualquier palabra, pero `EOF` (*End Of File*) es la convención |

### Parte B: el alias

```bash
alias usuarios="awk -F: '\$3 >= 1000 && \$3 < 60000 {print \$1}' /etc/passwd"
```

| Pieza | Qué es | Por qué |
|---|---|---|
| `alias usuarios=` | Crea un comando nuevo llamado `usuarios` | Al escribir `usuarios`, bash lo sustituye por lo que va después del `=` |
| `"..."` | Comillas **dobles** exteriores | Porque el comando de dentro ya tiene comillas simples, y no se pueden meter comillas simples dentro de otras simples |
| `\$3` | `$` **escapado** con `\` | Dentro de comillas dobles, bash intentaría sustituir `$3` por **su** variable `$3` (vacía) **al crear el alias**. La `\` le dice "deja el `$` tal cual" |
| `\"` (en `\": \"` y `\"\"`) | Comillas dobles escapadas | Para meter comillas dobles **dentro** de las comillas dobles exteriores sin cerrarlas |

**Qué pasaría sin escapar los `$`:**
```bash
alias usuarios="awk -F: '$3 >= 1000 {print $1}' /etc/passwd"
# Al crear el alias, bash cambia $3 y $1 por vacío:
# → awk -F: ' >= 1000 {print }' /etc/passwd    ✘ error de sintaxis
```

### Parte C: activarlo

| Pieza | Por qué |
|---|---|
| `source ~/.bashrc` | Vuelve a leer el `.bashrc` en la terminal actual. Sin esto, los alias solo funcionarían en las terminales nuevas |

### Comprobarlo

```bash
alias usuarios      # muestra cómo ha quedado guardado el alias
tail -5 ~/.bashrc   # muestra las últimas 5 líneas del .bashrc, para ver que se añadieron
```

A partir de ahí:
```bash
usuarios            # → stgadmin
grupos              # → stgadmin: , ops: , dev:
gruposconmiembros   # → adm: syslog,stgadmin, cdrom: stgadmin...
```

