# 03 · Comandos esenciales del administrador

> [!NOTE]
> **Objetivos del capítulo**
> - Visualizar una estructura de árbol con `tree`
> - Buscar texto en archivos con `grep`
> - Modificar texto sin abrir un editor con `sed`
> - Procesar columnas y calcular con `awk`
> - Dividir, ordenar y contar una salida con `cut`, `sort`, `uniq`, `wc` y `tr`
> - Encontrar archivos según su nombre, fecha o tamaño con `find`
> - Analizar el uso del disco con `du` y `df`
> - Crear y extraer archivos con `tar`

## Tabla de contenidos

1. [Listar una estructura de árbol — `tree`](#listar-una-estructura-de-árbol--tree)
2. [Extraer información — `grep`](#extraer-información--grep)
3. [Reemplazar texto — `sed`](#reemplazar-texto--sed)
4. [Procesar columnas — `awk`](#procesar-columnas--awk)
5. [Dividir, ordenar, contar — `cut`, `sort`, `uniq`, `wc`, `tr`](#dividir-ordenar-contar--cut-sort-uniq-wc-tr)
6. [Encontrar archivos — `find`](#encontrar-archivos--find)
7. [Análisis del espacio en disco — `du` y `df`](#análisis-del-espacio-en-disco--du-y-df)
8. [Archivación — `tar`](#archivación--tar)
9. [Otros comandos a conocer](#otros-comandos-a-conocer)
10. [Resumen](#resumen)

---

## Listar una estructura de árbol — `tree`

```bash
tree
```

```text
.
├── 01_base.md
├── 02.1_user_management.md
├── 02_file_permissions.md
├── 03_commandes_essentielles.md
├── 04_installation_et_services.md
├── 05_securite.md
├── 06_shellscript
│   ├── 01_Base
│   ├── 02_operateur_logique
│   ├── 03_boucles
│   ├── 04_gestion_des_erreurs
│   ├── 05_Arguments
│   └── 06_fonctions
├── 07_docker.md
├── 08_docker_administration.md
├── 09_tmux.md
├── 10_shell_config.md
├── 11_advanced_config.md
├── ejemplo
│   ├── errors.txt
│   ├── ejemplo.txt
│   ├── ejemplo2.txt
│   ├── exercice.txt
│   ├── exerciceEditor.txt
│   ├── archivo.txt
│   ├── out.txt
│   └── test.js
├── ejercicios
│   ├── Ejercicio3.md
│   └── Ejercicio4.md
└── README.md
```

Opciones útiles:

| Opción | Efecto |
|--------|-------|
| `tree -L 2` | Limitar la profundidad a 2 niveles |
| `tree -d`   | Mostrar solo carpetas |
| `tree -a`   | Incluir archivos ocultos |
| `tree -h`   | Mostrar el tamaño de los archivos |

---

## Extraer información — `grep`

`grep` busca un patrón en un archivo o en la salida de otro comando.

```bash
grep "sudo" 02.1_user_management.md
```

```text
sudo comando
sudo -u usuario comando
sudo su
```

### Recuperar el número de líneas

```bash
grep -n "sudo" 02.1_user_management.md
```

```text
114:sudo comando
122:sudo -u usuario comando
136:sudo su
```

### Búsqueda recursiva en una estructura de árbol

```bash
grep -R "sudo" ../ShellScripts
```

```text
../ShellScripts/ejercicios/Ejercicio4.md:sudo useradd -d /home/newuser -m newuser
../ShellScripts/02.1_user_management.md:sudo comando
../ShellScripts/02.1_user_management.md:sudo -u usuario comando
../ShellScripts/02.1_user_management.md:sudo su
../ShellScripts/03_commandes_essentielles.md:grep "sudo" 02.1_user_management.md
../ShellScripts/03_commandes_essentielles.md:sudo comando
../ShellScripts/03_commandes_essentielles.md:sudo -u usuario comando
../ShellScripts/03_commandes_essentielles.md:sudo su
```

> [!TIP]
> `../ShellScripts` puede acortarse a `.` si ya estás en la carpeta.

### Ignorar mayúsculas y minúsculas

```bash
grep -i "usuario" 02.1_user_management.md
```

### Filtrar la salida de otro comando

```bash
ps aux | grep apache
```

### Las opciones a recordar

| Opción | Efecto |
|--------|-------|
| `-n` | Mostrar el número de línea |
| `-i` | Insensible a mayúsculas |
| `-R` / `-r` | Búsqueda recursiva en subcarpetas |
| `-v` | Invertir: mostrar líneas que **no** contienen el patrón |
| `-c` | Contar el número de líneas coincidentes |
| `-l` | Mostrar solo el **nombre** de archivos que contienen el patrón |
| `-E` | Usar expresiones regulares extendidas |
| `-A 3` / `-B 3` | Mostrar 3 líneas **después** / **antes** de cada resultado |

---

## Reemplazar texto — `sed`

`grep` **busca**, `sed` **modifica**. Es un editor de flujo: lee línea por línea, aplica una transformación y escribe el resultado en la salida estándar.

### Sustituir una palabra

```bash
sed 's/localhost/127.0.0.1/' config.txt
```

```text
s   => substitute (reemplazar)
/   => separador entre la búsqueda y el reemplazo
```

> [!IMPORTANT]
> Por defecto, `sed` solo reemplaza la **primera ocurrencia de cada línea** y **no escribe nada en el archivo**: solo muestra el resultado.

Para reemplazar **todas** las ocurrencias de una línea, se añade el marcador `g` (*global*):

```bash
sed 's/localhost/127.0.0.1/g' config.txt
```

### Modificar realmente el archivo

```bash
sed -i 's/localhost/127.0.0.1/g' config.txt
```

`-i` = *in place*, la modificación se escribe en el archivo.

> [!CAUTION]
> `sed -i` sobrescribe el archivo **sin confirmación**. En un archivo de configuración del sistema, siempre mantén una copia:
>
> ```bash
> sed -i.bak 's/^Port 22/Port 2222/' /etc/ssh/sshd_config
> ```
>
> `-i.bak` guarda el original en `sshd_config.bak` antes de modificar.
> El reflejo: ejecuta primero el comando **sin** `-i` para verificar el resultado en pantalla.

### Cambiar separador

Cuando el patrón contiene `/` (una ruta, típicamente), se usa otro separador para evitar escapes:

```bash
# Ilegible
sed 's/\/var\/www/\/srv\/www/g' archivo

# Legible: el separador se convierte en |
sed 's|/var/www|/srv/www|g' archivo
```

### Eliminar líneas

```bash
# Eliminar líneas que contienen "DEBUG"
sed '/DEBUG/d' app.log
```

```bash
# Eliminar líneas vacías
sed '/^$/d' archivo.txt
```

```bash
# Eliminar la primera línea (el encabezado de un CSV por ejemplo)
sed '1d' data.csv
```

### Mostrar un rango de líneas

`-n` corta la visualización automática, y `p` (*print*) solo muestra lo que pedimos:

```bash
# Las líneas 10 a 20
sed -n '10,20p' /var/log/syslog
```

```bash
# De la línea 100 hasta el final
sed -n '100,$p' /var/log/syslog
```

### Comentar / descomentar una directiva

Un clásico de la administración del sistema:

```bash
# Comentar la línea que comienza con PermitRootLogin
sed -i 's/^PermitRootLogin/#PermitRootLogin/' /etc/ssh/sshd_config
```

```bash
# Descomentar la línea que contiene "max_connections"
sed -i '/max_connections/s/^#//' /etc/postgresql/15/main/postgresql.conf
```

### Encadenar varias transformaciones

```bash
sed -e 's/foo/bar/g' -e '/^$/d' archivo.txt
```

… o con un `;`:

```bash
sed 's/foo/bar/g; /^$/d' archivo.txt
```

### Las opciones a recordar

| Elemento | Efecto |
|---------|-------|
| `s/patrón/reemplazo/` | Reemplazar la primera ocurrencia de cada línea |
| `s/patrón/reemplazo/g` | Reemplazar **todas** las ocurrencias |
| `s/patrón/reemplazo/gi` | … ignorando mayúsculas |
| `-i` / `-i.bak` | Modificar el archivo en su lugar (con copia de seguridad) |
| `-n` … `p` | Solo mostrar las líneas seleccionadas |
| `-E` | Expresiones regulares extendidas (`+`, `?`, `(a\|b)`) |
| `/patrón/d` | Eliminar líneas que coincidan con el patrón |
| `1d` / `1,5d` / `$d` | Eliminar la línea 1 / líneas 1 a 5 / la última |
| `^` / `$` | Inicio / fin de línea |
| `&` | Recuerda el texto encontrado (`s/error/[&]/` → `[error]`) |

---

## Procesar columnas — `awk`

Muchas salidas de Linux son **tablas**: `ps aux`, `df -h`, `/etc/passwd`, registros de Apache… `awk` divide automáticamente cada línea en columnas y permite extraer, filtrar y calcular.

### Mostrar una columna

```bash
ps aux | awk '{print $1}'
```

Las columnas se llaman `$1`, `$2`, `$3`… y `$0` designa la **línea completa**.

```bash
# El usuario y el comando de cada proceso
ps aux | awk '{print $1, $11}'
```

> [!NOTE]
> Por defecto, `awk` divide en **espacios y tabulaciones**, ignorando repeticiones. Esto lo hace más confiable que `cut` en una salida alineada como `ps aux` o `df -h`.

### Cambiar el separador

```bash
awk -F: '{print $1}' /etc/passwd
```

```text
root
daemon
bin
sys
...
```

### Filtrar líneas

Como `grep`, pero manteniendo el poder de las columnas:

```bash
# Las líneas que contienen "error", de las que mostramos la 1ª y 4ª columna
awk '/error/ {print $1, $4}' /var/log/apache2/error.log
```

Y con una **condición** en una columna:

```bash
# Los usuarios reales (UID >= 1000), 3er campo de /etc/passwd
awk -F: '$3 >= 1000 {print $1, $3}' /etc/passwd
```

```bash
# Los procesos que consumen más del 5% de CPU
ps aux | awk '$3 > 5.0 {print $2, $3, $11}'
```

### Calcular

```bash
# Suma de la memoria (%MEM) consumida por todos los procesos
ps aux | awk '{sum += $4} END {printf "%.1f %%\n", sum}'
```

```bash
# Número de líneas y tamaño promedio de un campo
awk -F: '{n++; total += length($1)} END {print n " líneas, longitud media " total/n}' /etc/passwd
```

`END { … }` se ejecuta **una vez al final**, cuando se han leído todas las líneas. Su equivalente `BEGIN { … }` se ejecuta antes de la primera línea (práctico para mostrar un encabezado).

### Ejemplo completo: las IPs más activas de un registro

```bash
awk '{print $1}' /var/log/apache2/access.log | sort | uniq -c | sort -rn | head
```

```text
   1420 192.168.1.24
    317 10.0.0.8
     94 203.0.113.7
```

```text
awk '{print $1}'  => extraer la 1ª columna (la dirección IP)
sort              => agrupar líneas idénticas
uniq -c           => contar cada IP
sort -rn          => ordenar por número descendente
head              => mantener el top 10
```

### Los elementos a recordar

| Elemento | Efecto |
|---------|-------|
| `$1`, `$2`, … | La 1ª, 2ª columna… |
| `$0` | La línea completa |
| `$NF` | La **última** columna (`NF` = *number of fields*) |
| `NR` | El número de la línea actual |
| `-F:` / `-F';'` | Separador de columnas |
| `/patrón/ { … }` | Actuar solo en líneas que coincidan con el patrón |
| `$3 > 100 { … }` | Condición en el valor de una columna |
| `BEGIN { … }` / `END { … }` | Bloque ejecutado antes / después del procesamiento |
| `printf "%.2f\n", x` | Visualización formateada (2 decimales) |

> [!TIP]
> `grep` para buscar, `sed` para reemplazar, `awk` para columnas y cálculos. En cuanto un "script `awk`" supera unas pocas líneas, es mejor escribir un script shell verdadero (capítulo 06) o Python.

---

## Dividir, ordenar, contar — `cut`, `sort`, `uniq`, `wc`, `tr`

Estos pequeños comandos casi nunca se usan solos: se ensamblan con pipes.

### `cut` — extraer columnas simples

```bash
cut -d: -f1 /etc/passwd
```

```text
-d: => el delimitador es ":"
-f1 => quiero el campo nº 1
```

```bash
# Varios campos
cut -d: -f1,3,7 /etc/passwd
```

```bash
# Los primeros 10 caracteres de cada línea
cut -c1-10 archivo.txt
```

> [!WARNING]
> `cut` trata **cada** delimitador como una separación: en una salida alineada con varios espacios (`ps aux`, `df -h`), da columnas vacías. En ese caso, usa `awk`.

### `sort` — ordenar

```bash
sort archivo.txt
```

| Opción | Efecto |
|--------|-------|
| `-r` | Orden inverso |
| `-n` | Ordenar **numérico** (sino `10` va antes que `9`) |
| `-h` | Ordenar tamaños legibles (`1K`, `2M`, `3G`) |
| `-u` | Eliminar duplicados |
| `-k2` | Ordenar por la 2ª columna |
| `-t:` | Cambiar el separador de columnas |

```bash
# Los usuarios ordenados por UID
sort -t: -k3 -n /etc/passwd
```

### `uniq` — desduplicar y contar

```bash
sort archivo.txt | uniq -c
```

| Opción | Efecto |
|--------|-------|
| `-c` | Contar ocurrencias |
| `-d` | Mostrar solo líneas duplicadas |
| `-u` | Mostrar solo líneas únicas |

> [!IMPORTANT]
> `uniq` solo compara líneas **consecutivas**. Por lo tanto, casi siempre necesita un `sort` antes.

### `wc` — contar

```bash
wc -l /var/log/syslog
```

| Opción | Efecto |
|--------|-------|
| `-l` | Número de **líneas** |
| `-w` | Número de **palabras** |
| `-c` | Número de **bytes** |

```bash
# ¿Cuántos usuarios hay en la máquina?
wc -l < /etc/passwd
```

### `tr` — transformar caracteres

```bash
# Todo en mayúsculas
echo "hola" | tr 'a-z' 'A-Z'
```

```bash
# Eliminar caracteres
echo "1 234 567" | tr -d ' '
```

```bash
# Comprimir espacios repetidos en uno solo (útil antes de cut)
ps aux | tr -s ' ' | cut -d' ' -f1,11
```

| Opción | Efecto |
|--------|-------|
| *(ninguna)* | Reemplazar un conjunto de caracteres por otro |
| `-d` | Eliminar los caracteres indicados |
| `-s` | *Squeeze*: reducir repeticiones a una sola ocurrencia |

---

## Encontrar archivos — `find`

### Por nombre

```bash
find / -name "passwd" 2> /dev/null
```

> [!NOTE]
> El `2> /dev/null` redirige los errores (típicamente "Permission denied" en carpetas del sistema) hacia la nada, para mantener solo los resultados útiles.

### Por fecha de modificación

Encontrar todos los archivos modificados hace menos de dos días:

```bash
find carpeta -mtime -2
```

### Por tamaño

Aquí, archivos de más de 10 MiB:

```bash
find /var/log -type f -size +10M
```

### Realizar acciones en archivos encontrados

```bash
# Elimina archivos de /tmp que terminan en .tmp
find /tmp -type f -name "*.tmp" -delete
```

```bash
# Lista los detalles de cada archivo encontrado
# -exec => ejecuto un comando
# ls -lh => el comando ejecutado
# {}     => el elemento encontrado
# \;     => fin de la instrucción
find . -type f -name "*.md" -exec ls -lh {} \;
```

Salida:

```text
-rw-r--r-- 1 rmdir  staff     0B Dec 10 15:32 ./09_tmux.md
-rw-r--r-- 1 rmdir  staff   593B Dec 11 13:09 ./ejercicios/Ejercicio4.md
-rw-r--r-- 1 rmdir  staff   1.1K Dec 10 15:47 ./ejercicios/Ejercicio3.md
-rw-r--r-- 1 rmdir  staff    81B Dec 11 11:42 ./05_securite.md
-rw-r--r-- 1 rmdir  staff     0B Dec 10 15:33 ./11_advanced_config.md
-rw-r--r-- 1 rmdir  staff   3.6K Dec 10 14:35 ./01_base.md
-rw-r--r-- 1 rmdir  staff   2.7K Dec 11 09:53 ./02_file_permissions.md
-rw-r--r-- 1 rmdir  staff    86B Dec 11 11:40 ./04_installation_et_services.md
-rw-r--r-- 1 rmdir  staff    12B Dec 10 10:39 ./README.md
-rw-r--r-- 1 rmdir  staff   2.3K Dec 11 11:24 ./02.1_user_management.md
-rw-r--r-- 1 rmdir  staff     0B Dec 10 15:32 ./10_shell_config.md
-rw-r--r-- 1 rmdir  staff   2.8K Dec 11 13:28 ./03_commandes_essentielles.md
```

> [!TIP]
> `-exec comando {} +` (con un `+` en lugar de `\;`) agrupa todos los resultados en **una sola ejecución** del comando en lugar de una por archivo. Es mucho más rápido con grandes volúmenes.

### Pasar los resultados a otro comando — `xargs`

`xargs` transforma una lista recibida en la entrada estándar en **argumentos** de un comando. Es la alternativa a `-exec`, y la única forma cuando la lista viene de otra parte que no sea `find`.

```bash
find /var/log -name "*.gz" | xargs rm
```

¿Por qué `xargs`? Muchos comandos (`rm`, `grep`, `tar`…) esperan sus objetivos como argumentos, no en la entrada estándar: `find … | rm` no funciona, `find … | xargs rm` sí.

```bash
# Buscar un patrón solo en archivos .conf
find /etc -name "*.conf" | xargs grep -l "listen"
```

Colocar el argumento en otra parte que no sea el final, con `-I`:

```bash
# {} es reemplazado por cada resultado
find . -name "*.md" | xargs -I {} cp {} /tmp/copia_seguridad/
```

> [!CAUTION]
> Un nombre de archivo que contiene un **espacio** rompe `xargs`: `mi reporte.md` se convierte en dos argumentos. La versión robusta usa el par `-print0` / `-0`:
>
> ```bash
> find /var/log -name "*.gz" -print0 | xargs -0 rm
> ```

| Opción | Efecto |
|--------|-------|
| `-0` | Las entradas están separadas por `\0` (con `find -print0`) |
| `-I {}` | Reemplazar `{}` por el argumento, donde sea en el comando |
| `-n 1` | Un solo argumento por ejecución del comando |
| `-P 4` | Ejecutar 4 llamadas en **paralelo** |
| `-t` | Mostrar el comando antes de ejecutarlo (para verificar) |

### Los criterios a recordar

| Criterio | Efecto |
|---------|-------|
| `-name "*.log"` | Por nombre (sensible a mayúsculas; `-iname` no lo es) |
| `-type f` / `-type d` / `-type l` | Archivo / carpeta / enlace simbólico |
| `-size +10M` / `-size -1k` | Más grande que / más pequeño que |
| `-mtime -2` | Modificado hace menos de 2 días (`-mmin -30`: 30 minutos) |
| `-user debian` | Perteneciente a un usuario |
| `-perm -4000` | Con un bit de permiso dado (aquí SUID) |
| `-maxdepth 2` | Limitar la profundidad de búsqueda |
| `-delete` / `-exec … \;` / `-exec … +` | Actuar en los resultados |

---

## Análisis del espacio en disco — `du` y `df`

### `du` — lo que **consumen** las carpetas

```bash
du -h --max-depth=1 /var
```

```text
4.0K	/var/spool
40K	/var/tmp
1.2M	/var/backups
82M	/var/cache
4.0K	/var/local
5.5G	/var/log
4.0K	/var/mail
20K	/var/www
4.0K	/var/opt
153M	/var/lib
5.7G	/var
```

Luego descendemos a la carpeta culpable:

```bash
du -h --max-depth=1 /var/log
```

```text
252K	/var/log/apt
4.1G	/var/log/journal
29M	/var/log/apache2
8.0K	/var/log/runit
4.0K	/var/log/private
792K	/var/log/unattended-upgrades
5.5G	/var/log
```

> [!TIP]
> Para encontrar rápidamente los más grandes: `du -h --max-depth=1 /var | sort -hr | head`

### Encontrar el peso total de un conjunto de archivos con `find`

```bash
find /var/backups -name "*.backup" -exec du -ch {} + | grep total
```

### `df` — lo que **queda** en las particiones

```bash
df -h
```

```text
Filesystem      Size  Used Avail Use% Mounted on
udev            1.9G     0  1.9G   0% /dev
tmpfs           384M  500K  383M   1% /run
/dev/sda1        79G  6.6G   69G   9% /
tmpfs           1.9G     0  1.9G   0% /dev/shm
tmpfs           5.0M     0  5.0M   0% /run/lock
/dev/sda15      124M   12M  113M  10% /boot/efi
tmpfs           384M     0  384M   0% /run/user/1000
```

| Comando | Pregunta a la que responde |
|----------|--------------------------------|
| `df -h`  | ¿Mis particiones están llenas? |
| `du -h`  | ¿Qué está ocupando espacio? |

---

## Archivación — `tar`

### Crear un archivo

```bash
tar -cvf copia.tar /carpeta
```

```text
c => crear
v => verbose (salida detallada)
f => especificar en qué archivo guardar el archivo. Debe estar en última posición
```

### Crear un archivo comprimido

```bash
tar -czvf copia.tar.gz /carpeta
```

```text
c => crear
z => comprimir con gzip
v => verbose (salida detallada)
f => especificar en qué archivo guardar el archivo
```

Equivalente en dos pasos:

```bash
tar -cvf copia.tar /carpeta
gzip copia.tar
```

…que produce `copia.tar.gz`.

### Extraer un archivo

```bash
tar -xvf copia.tar
```

Archivo comprimido:

```bash
tar -xzvf copia.tar.gz
```

Extraer en una carpeta objetivo:

```bash
tar -xzvf copia.tar.gz -C /carpeta_objetivo
```

> [!TIP]
> Antes de extraer un archivo desconocido, lista su contenido para verificar que no vaya a verter 200 archivos en la carpeta actual:
>
> ```bash
> tar -tvf copia.tar.gz
> ```

### Crear un tarball desde un `find`

```bash
find . -type f -name "*.md" -print0 | tar --null -czvf mdbackups.tar.gz --files-from=-
```

```text
find :
  -type f        => buscar archivos
  -name "*.md"   => cuyo nombre termina en .md
  -print0        => separar cada resultado con un \0

tar :
  --null         => las entradas están separadas por \0 (salida del comando anterior)
  -czvf archivo  => crear un archivo comprimido
  --files-from=- => la lista de archivos proviene de la entrada estándar (-),
                    por lo tanto de la salida de find
```

> [!NOTE]
> El par `-print0` / `--null` sirve para manejar correctamente nombres de archivos que contienen **espacios** o saltos de línea. Sin él, un archivo `mi reporte.md` se trataría como dos entradas.

### Las opciones `tar` a recordar

| Opción | Efecto |
|--------|-------|
| `-c` | Crear un archivo |
| `-x` | Extraer un archivo |
| `-t` | Listar el contenido sin extraer |
| `-f` | Nombre del archivo de archivo |
| `-v` | Verbose |
| `-z` / `-j` / `-J` | Comprimir con gzip / bzip2 / xz |
| `-C /ruta` | Colocarse en esta carpeta antes de actuar |

---

## Otros comandos a conocer

No merecen un capítulo, pero son útiles todos los días.

### `tee` — escribir en un archivo **y** en la pantalla

```bash
apt update | tee /tmp/update.log
```

Útil cuando un comando necesita `sudo` para escribir un archivo — la redirección `>` clásica falla, porque es el *shell* (no root) el que abre el archivo:

```bash
echo "127.0.0.1 misitio.local" | sudo tee -a /etc/hosts
```

`-a` = *append*, como `>>`.

### `diff` — comparar dos archivos

El reflejo después de modificar una configuración:

```bash
diff /etc/ssh/sshd_config.bak /etc/ssh/sshd_config
```

```bash
# Versión más legible, con contexto
diff -u archivo_antiguo.conf archivo_nuevo.conf
```

`diff -r carpeta1 carpeta2` compara dos estructuras de árbol enteras.

### `stat` y `file` — los metadatos de un archivo

```bash
stat reporte.pdf
```

```text
  File: reporte.pdf
  Size: 24576      Blocks: 48   IO Block: 4096   regular file
Access: (0644/-rw-r--r--)  Uid: ( 1000/  rmdir)   Gid: ( 1000/  rmdir)
Modify: 2025-12-11 13:28:41.000000000 +0100
```

```bash
# ¿Qué tipo es realmente este archivo? (la extensión puede engañar)
file archivo.bin
```

```text
archivo.bin: gzip compressed data, from Unix
```

### `watch` — repetir un comando

```bash
# Redibujar df -h cada 2 segundos
watch df -h
```

```bash
# Cada 5 segundos, resaltando qué cambia
watch -n 5 -d "ls -l /var/spool/mail"
```

`CTRL + C` para salir.

### `which` y `type` — ¿dónde está este comando?

```bash
which python3
```

```text
/usr/bin/python3
```

`type mi_comando` va más lejos: indica si se trata de un binario, un **alias** o una función del shell.

### En resumen

| Comando | Rol |
|----------|------|
| `tee` | Duplicar una salida: archivo + pantalla |
| `diff` | Comparar dos archivos o dos carpetas |
| `stat` | Metadatos detallados (tamaño, fechas, permisos) |
| `file` | Tipo real de un archivo |
| `watch` | Repetir un comando a intervalo regular |
| `which` / `type` | Localizar un comando, detectar un alias |
| `ln -s` | Crear un enlace simbólico (ver capítulo 11) |
| `rsync` | Copiar / sincronizar, incluso remotamente (ver capítulo 05) |

---

## Resumen

| Comando | Rol |
|----------|------|
| `tree` | Visualizar una estructura de árbol |
| `grep` | **Buscar** texto |
| `sed` | **Reemplazar** o eliminar texto |
| `awk` | Extraer **columnas**, filtrar, calcular |
| `cut` | Extraer columnas con delimitador simple |
| `sort` / `uniq` / `wc` | Ordenar / desduplicar / contar |
| `tr` | Transformar o eliminar caracteres |
| `find` | Buscar archivos |
| `xargs` | Pasar una lista como argumentos a un comando |
| `du` | Medir el espacio utilizado |
| `df` | Medir el espacio disponible |
| `tar` | Archivar y comprimir |
| `tee` / `diff` / `watch` | Duplicar una salida / comparar / monitorear |

> [!TIP]
> El trío a recordar para analizar registros: `grep` filtra líneas, `awk` extrae columnas, `sort | uniq -c | sort -rn` cuenta y clasifica.
>
> ```bash
> grep "404" access.log | awk '{print $7}' | sort | uniq -c | sort -rn | head
> ```
>
> … es decir: las 10 URL que se encuentran con más frecuencia.

---

⬅️ [Anterior: 02.1 · Usuarios](02.1_user_management_es.md) · 🏠 [Tabla de contenidos](README_es.md) · [Siguiente: 04 · Paquetes y servicios ➡️](04_instalacion_y_servicios_es.md)
