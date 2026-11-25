# 🧰 A4 · La caja de herramientas

*Capítulo [03 · Órdenes esenciales](../../03_commandes_essentielles.md)*

> [!NOTE]
> **Objetivos**: explotar registros y archivos del sistema con `grep`, `sed`, `awk`, `cut`/`sort`/`uniq`,
> encontrar cualquier cosa con `find` y `xargs`, diagnosticar disco lleno con `du` y `df`, archivar
> con `tar`. Es la herramienta que usa todos los días, todos los días.

---

## 🧰 Conjunto de datos

Si su servidor es demasiado nuevo para tener registros interesantes, ponga este conjunto de trabajo:

```bash
mkdir -p /srv/labo/{logs,conf,archives,web} && cd /srv/labo

cat > logs/auth.log <<'EOF'
Mar  1 08:02:11 srv-tickets sshd[1420]: Accepted publickey for alice from 10.0.0.14 port 51422 ssh2
Mar  1 08:14:02 srv-tickets sshd[1503]: Failed password for root from 203.0.113.9 port 39112 ssh2
Mar  1 08:14:05 srv-tickets sshd[1503]: Failed password for root from 203.0.113.9 port 39112 ssh2
Mar  1 08:14:09 srv-tickets sshd[1503]: Failed password for invalid user admin from 203.0.113.9 port 39118 ssh2
Mar  1 08:21:44 srv-tickets sudo:    alice : TTY=pts/0 ; PWD=/home/alice ; USER=root ; COMMAND=/usr/bin/apt update
Mar  1 09:02:31 srv-tickets sshd[1688]: Failed password for bob from 198.51.100.4 port 44120 ssh2
Mar  1 09:02:35 srv-tickets sshd[1688]: Accepted password for bob from 198.51.100.4 port 44120 ssh2
Mar  1 09:40:00 srv-tickets sudo:      bob : user NOT in sudoers ; TTY=pts/1 ; PWD=/home/bob ; USER=root ; COMMAND=/bin/systemctl restart nginx
Mar  1 10:11:52 srv-tickets sshd[1902]: Failed password for root from 203.0.113.9 port 40122 ssh2
EOF

cat > logs/nginx-access.log <<'EOF'
10.0.0.14 - - [01/Mar/2024:09:01:02] "GET /api/tickets HTTP/1.1" 200 1420
203.0.113.9 - - [01/Mar/2024:09:01:44] "GET /admin.php HTTP/1.1" 404 162
203.0.113.9 - - [01/Mar/2024:09:01:45] "GET /wp-login.php HTTP/1.1" 404 162
10.0.0.14 - - [01/Mar/2024:09:03:10] "POST /api/tickets HTTP/1.1" 201 88
198.51.100.4 - - [01/Mar/2024:09:05:00] "GET /api/tickets HTTP/1.1" 500 512
10.0.0.14 - - [01/Mar/2024:09:07:31] "GET /api/health HTTP/1.1" 200 12
203.0.113.9 - - [01/Mar/2024:09:09:00] "GET /.env HTTP/1.1" 404 162
198.51.100.4 - - [01/Mar/2024:09:12:20] "GET /api/tickets HTTP/1.1" 500 512
EOF

cp /etc/ssh/sshd_config conf/sshd_config.copie
head -c 40M /dev/zero > archives/dump-2023.bin
head -c 12M /dev/zero > logs/vieux.log
touch -d '2 years ago' archives/dump-2023.bin
touch web/index.html web/.env conf/vhost.conf logs/old.tmp
```

---

## Ejercicio 4.1 — `grep`: buscar en registros

En `logs/auth.log`:

1. Todos los intentos de autenticación **fallidos**, con su número de línea.
2. El **número** de estos intentos.
3. Las conexiones **exitosas**, cualquiera que sea la caja.
4. Todas las líneas **excepto** las concernientes a `alice`.
5. Las líneas que contienen `Failed` **o** `NOT in sudoers`, en una sola orden *(regex extendida)*.
6. Una búsqueda **recursiva** de `203.0.113.9` en todo `/srv/labo`, mostrando solo los nombres de archivos.
7. En `/etc/ssh/sshd_config`: todas las directivas activas, es decir, las líneas que no son
   **ni vacías ni comentadas**. *(La orden que usará toda su carrera.)*

---

## Ejercicio 4.2 — `sed`: modificar configuraciones

En `conf/sshd_config.copie` — **nunca en el archivo verdadero mientras la orden no esté validada**:

1. Muestre el archivo con `#PermitRootLogin` descomentado y puesto a `no`, **sin** modificar el archivo.
2. Haga realmente la modificación, guardando una copia de seguridad.
3. Cambie el puerto de `22` a `2222`, tocando solo la directiva `Port`.
4. Comente todas las líneas que comienzan con `PasswordAuthentication`.
5. Elimine de un golpe las líneas vacías **y** las líneas comentadas, escriba el resultado en
   `conf/sshd_config.propre`.
6. Muestre solo las líneas 10 a 25 del archivo original.
7. ¿Por qué el cambio de separador (`s|…|…|`) es imprescindible en cuanto se manipulan rutas?

> [!CAUTION]
> `sed -i` sobrescribe sin confirmación y sin deshacer. En un archivo de `/etc`, **siempre** lance la orden
> sin `-i` primero, y mantenga un `.bak` hasta que el servicio se reinicie sin error.

---

## Ejercicio 4.3 — `awk`: analizar

1. En `/etc/passwd`: el nombre y el UID de todas las cuentas cuyo **UID ≥ 1000**.
2. En `/etc/passwd`: las cuentas que tengan un shell **distinto** de `/usr/sbin/nologin` o `/bin/false`.
3. En `logs/auth.log`: el **top de IPs** originarias de fallos de autenticación, la más insistente primero.
4. En `logs/auth.log`: la lista de usuarios objetivo de estos fallos.
5. En `logs/nginx-access.log`: el número de solicitudes **por código HTTP**, en un bloque `END`.
6. En `logs/nginx-access.log`: las IPs que provocaron al menos un error `500`.
7. En `logs/nginx-access.log`: el **volumen total** de bytes servidos (última columna), mostrado en Mo.
8. En la salida de `ps aux`: la memoria total consumida por los procesos de un usuario dado.

---

## Ejercicio 4.4 — `cut`, `sort`, `uniq`, `wc`, `tr`

1. La lista de nombres de usuarios de `/etc/passwd`, con `cut`.
2. El número de cuentas de la máquina, en una orden.
3. La lista ordenada y sin duplicados de IPs presentes en `nginx-access.log`, con su número de ocurrencias.
4. Los shells usados en la máquina, sin duplicados y contados — ¿cuántas cuentas por shell?
5. En la salida de `df -h`, extraiga el punto de montaje y el porcentaje de ocupación, alineados,
   comprimiendo espacios con `tr -s` luego `cut`. ¿Por qué `awk` lo hace más limpiamente?

---

## Ejercicio 4.5 — `find`: encontrar

1. Todos los archivos `.conf` bajo `/etc`, sin errores de permiso en pantalla
   *(pista: redirigir la salida de error)*.
2. Todos los archivos de más de **10 Mo** bajo `/srv` y `/var`.
3. Los archivos **no modificados hace más de un año** en `/srv/labo/archives`.
4. Los archivos modificados hace **menos de 60 minutos** bajo `/etc` — útil tras una intervención.
5. Los archivos pertenecientes a un usuario dado, fuera de su *home*.
6. Los archivos **world-writable** (permisos de escritura para todos) bajo `/srv`: ¿por qué es un
   punto de auditoría?
7. Los archivos `.tmp` de `/srv/labo`, listados luego eliminados.
8. El detalle (`ls -lh`) de cada `.log` encontrado, vía `-exec`.

---

## Ejercicio 4.6 — `xargs`: encadenar

1. Busque `PermitRootLogin` **solo en archivos `.conf`** de `/etc/ssh/`, combinando
   `find` y `xargs grep`.
2. Copie todos los `.conf` de `/srv/labo/conf` a `/srv/labo/archives/conf/` con `xargs -I {}`.
3. Cuente el número total de líneas de todos los registros de `/srv/labo/logs`.
4. Un archivo `mi informe final.log` rompe un `find … | xargs rm` ingenuo: créelo, compruébelo, luego escriba
   la versión **robusta**.
5. Escriba la orden que comprime todos los `.log` de más de 7 días — primero con `-exec`,
   luego con `xargs`. ¿Qué diferencia de rendimiento, y por qué?

---

## Ejercicio 4.7 — El disco está lleno

Es la llamada del viernes por la noche. `/` está al 98%.

1. Muestre la ocupación de todas las particiones, legible.
2. Muestre el peso de cada directorio de primer nivel de `/`, sin descender más.
3. Descienda: encuentre los **10 directorios más grandes** de `/var`.
4. Encuentre los **10 archivos más grandes** de la máquina.
5. `df` dice que el disco está lleno pero la suma de `du` no lo encuentra. Cite **dos** causas clásicas
   *(pista: un archivo eliminado pero aún abierto por un proceso; los inodos)*.
6. ¿Cuántos **inodos** libres quedan?
7. Escriba la línea que comprime todos los registros de más de 30 días de `/var/log/ticketflow` — y explique
   por qué `logrotate` lo hace mejor que su línea. *(A revisar en [A5](05_paquets_services_cron_es.md).)*

---

## Ejercicio 4.8 — `tar`: respaldar

1. Archive `/etc` en `/srv/ticketflow/backups/etc-<fecha>.tar.gz`.
2. Liste el contenido del archivo sin extraerlo.
3. Extraiga de él **un solo archivo** en una carpeta de restauración.
4. Cree un archivo de `/srv/labo` **excluyendo** `archives/` y los `.tmp`.
5. Cree un archivo que contenga exactamente los archivos devueltos por un `find` *(pista: `-T -`)*.
6. ¿Qué opción preserva propietarios, permisos y ACL? ¿Por qué es vital para un respaldo del sistema?
7. Verifique la integridad de un archivo existente y produzca su suma de control.

---

## Ejercicio 4.9 — Los utilitarios de diagnóstico

1. Escriba la salida de una auditoría en pantalla **y** en un archivo, en una orden.
2. Compare `conf/sshd_config.copie` y el `sshd_config` original, en versión legible.
3. Muestre el tipo verdadero de `archives/dump-2023.bin` — ¿la extensión dice la verdad?
4. Muestre todos los metadatos de `/etc/shadow`: permisos, propietario, fechas, inodo.
5. ¿Es `systemctl` un binario, un alias, una función? ¿Dónde se encuentra?
6. ¿Qué orden mostraría `df -h` cada 5 segundos resaltando los cambios?

---

## Ejercicio 4.10 — El informe de incidente

Una sola línea de orden por pregunta.

1. El **top 5 de IPs** que fallaron en autenticarse, con su número de intentos.
2. La lista de usuarios que usaron `sudo` hoy, sin duplicados.
3. Las IPs que solicitaron rutas sospechosas (`.env`, `wp-login`, `admin.php`) en el registro nginx.
4. El número de solicitudes en error `5xx` por IP, ordenado.
5. Un archivo `/srv/ticketflow/backups/incident-<fecha>.tar.gz` que contenga todos los archivos de `logs/`
   que mencionen la IP fautiva.
6. Un archivo `/root/rapport-incident.txt` que reúna los resultados de los cinco puntos anteriores,
   cada uno precedido de un título.

---

## ✅ Verificación

- Produce el top de IPs atacantes de un `auth.log` sin dudarlo sobre el orden de órdenes.
- Sabe responder a "¿por qué el disco está lleno?" en menos de cinco órdenes.
- `/srv/ticketflow/backups/` contiene un archivo de `/etc` restaurable **con sus permisos**.
- `/root/rapport-incident.txt` existe y se lee por sí solo.
- Sabe explicar cada etapa de cada uno de sus pipes.
