# 🖥️ A1 · Puesta en marcha del servidor

*Capítulos [01 · Los fundamentos del terminal](../../01_base.md) y [09 · tmux](../../09_tmux.md)*

> [!NOTE]
> **Objetivos**: saber dónde se ha aterrizado. Reconocer la estructura de un sistema Linux, encontrar los
> archivos de configuración y los registros, leer qué está en ejecución, redirigir y archivar sus hallazgos —
> y trabajar en una sesión que sobreviva a una desconexión SSH.

---

## Ejercicio 1.1 — Evaluación de la situación

Conéctese a `srv-tickets` y responda, **con la orden correspondiente**, a las preguntas que le harán en
la reunión:

1. ¿Qué distribución y qué versión exacta?
2. ¿Qué versión del kernel? ¿Qué arquitectura?
3. ¿Cuál es el nombre de la máquina? ¿Cuánto tiempo lleva en funcionamiento?
4. ¿Cuántos procesadores, cuánta memoria RAM, cuánta queda libre?
5. ¿Qué particiones están montadas, y cuál está más llena?
6. ¿Cuál es la dirección IP del servidor y en qué interfaz?
7. ¿Quién está conectado a la máquina en este momento? ¿Quién se ha conectado recientemente?
8. Registre todas estas respuestas en `/root/journal.md` — es el comienzo de su documentación.

---

## Ejercicio 1.2 — La estructura de un sistema

Sin modificar nada, responda en una oración para cada uno:

1. ¿Qué se encuentra en `/etc`, `/var`, `/opt`, `/srv`, `/usr/local/bin` y `/tmp`?
2. ¿Cuál es la diferencia entre `/bin` y `/usr/local/bin`? ¿Dónde coloca **sus** scripts?
3. ¿Dónde están los registros del sistema? ¿Cuál es el archivo más grande en `/var/log`?
4. ¿Dónde se almacenan las unidades systemd proporcionadas por los paquetes? ¿Y **las suyas**?
5. ¿Ocupan espacio en disco `/proc` y `/sys`? ¿Por qué?
6. Liste el contenido de `/etc` en formato detallado, ordenado por **fecha de modificación** — ¿qué se
   ha modificado más recientemente en este servidor?

---

## Ejercicio 1.3 — Leer registros

1. Muestre las últimas 50 líneas de `/var/log/syslog`.
2. Siga `/var/log/auth.log` **en tiempo real** y, desde otra terminal, abra una sesión: ¿qué ve?
3. Muestre `/var/log/auth.log` página por página, pudiendo desplazarse hacia arriba.
4. ¿Cuántas líneas contiene `/var/log/auth.log`? ¿Cuál es su tamaño en disco?
5. Muestre las primeras 100 líneas del archivo, luego las líneas 200 a 220.
   *(Dos órdenes del capítulo 01 combinadas por una tubería bastan.)*

---

## Ejercicio 1.4 — Crear y rellenar archivos de configuración

1. Cree la estructura de trabajo del administrador:
```text
   /srv/ticketflow/
   ├── backups/
   ├── scripts/
   └── conf/
   /var/log/ticketflow/
```
   en **una sola orden** para la primera.
2. Con un **heredoc**, escriba `/srv/ticketflow/conf/serveur.conf`:
```ini
   [serveur]
   nom  = srv-tickets
   role = production
   admin = alice
```
3. Añada la línea `derniere_revue = <la fecha de hoy>` al final del archivo, **sin sobrescribirlo**,
   insertando la fecha producida por una orden.
4. Copie este archivo con extensión `.bak` — acostúmbrese a hacerlo, le será útil en todos los capítulos
   siguientes.
5. Cree un enlace simbólico `/root/conf` hacia `/srv/ticketflow/conf`, y verifíquelo.

---

## Ejercicio 1.5 — Redirecciones: mantener registro

1. Lance un inventario de los paquetes instalados y guárdelo en `/root/inventaire-paquets.txt`.
2. Lance una orden que falle (por ejemplo en una carpeta inexistente) y envíe **solo el error**
   a `/root/erreurs.log`.
3. Vuelva a lanzarla enviando **ambos flujos** al mismo archivo, **sin sobrescribir** su contenido.
4. Lance una orden larga descartando completamente su salida.
5. Escriba la salida de `df -h` en pantalla **y** en un archivo, en una sola orden.
6. Explique, en una oración, por qué `commande > fichier 2>&1` y `commande 2>&1 > fichier` no hacen
   lo mismo.

---

## Ejercicio 1.6 — Los procesos

1. Muestre todos los procesos de la máquina, luego solo los del usuario `root`.
2. Muestre los 5 procesos que más consumen **memoria**, luego los que más consumen **CPU**.
3. Encuentre el PID del servidor SSH, luego muestre el árbol de procesos.
4. Lance un proceso en segundo plano, encuéntrelo, deténgalo correctamente.
5. Un proceso no responde: ¿cuál es la diferencia entre la señal por defecto y `kill -9`?
   ¿En qué orden se intentan, y por qué?
6. ¿Qué proceso está escuchando en el puerto 22? ¿Y en el puerto 5432? *(Pista: `ss -tulpn`.)*
7. Un usuario se queja de que "el servidor va lento". Dé las **cuatro** órdenes que lanza en
   el primer minuto, y qué observa en cada una.

---

## Ejercicio 1.7 — Trabajar en una sesión que sobrevive *(capítulo [09](../../09_tmux.md))*

1. Instale tmux y abra una sesión nombrada `admin`.
2. Desactívese, verifique que sigue en ejecución, vuelva a conectarse.
3. Divida la pantalla en tres paneles: un shell, un `journalctl -f`, un `htop`.
4. Pase el panel de registros a pantalla completa, luego vuelva a la disposición normal.
5. Cree una segunda ventana nombrada `backup`, cambie entre las dos, renombre la primera `travail`.
6. Lance una actualización larga en un panel, **corte bruscamente su conexión SSH**, vuelva a conectarse
   y encuentre la actualización. ¿Qué hubiera pasado sin tmux?
7. Suba en el historial de un panel y busque en él una palabra. ¿Cómo se sale del modo copia?
8. Escriba en su `journal.md` la lista de accesos directos que usará todos los días.

> [!TIP]
> En un servidor remoto, `tmux` (o `screen`) no es un confort: es lo que evita que un corte de red
> interrumpa un `apt upgrade` o una migración de base de datos en mitad del proceso.

---

## ✅ Verificación

- `/root/journal.md` contiene la evaluación completa de la situación del ejercicio 1.1.
- `/srv/ticketflow/` y `/var/log/ticketflow/` existen, con `serveur.conf` y su copia de seguridad.
- Sabe encontrar sin dudarlo: la configuración de un servicio, sus registros, su unidad systemd, su PID
  y el puerto que escucha.
- Puede perder su conexión SSH sin perder su trabajo en curso.
