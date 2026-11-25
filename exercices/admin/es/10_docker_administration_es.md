# 🐳 A10 · Docker para el administrador

*Capítulo [08 · Docker para el administrador del sistema](../../08_docker_administration.md)*

> [!NOTE]
> **Objetivos**: explotar un host Docker sin que se convierta en una zona sin ley — configurar el demonio,
> dominar volúmenes, redes y registros, limitar recursos, endurecer contenedores, respaldar, limpiar
> e iniciar una stack en el arranque.

> [!IMPORTANT]
> No construimos imágenes aquí: la escritura de `Dockerfile` se trata en el
> [capítulo 07](../../07_docker.md) y en el [recorrido desarrollador](../dev/05_docker.md).
> Aquí explotamos lo que los desarrolladores entregan.

---

## Ejercicio 10.1 — El host

1. ¿Dónde almacena Docker sus datos en esta máquina? ¿Cuánto espacio ocupa?
2. ¿Qué controlador de almacenamiento, qué controlador de registros, qué controlador de red por defecto?
3. ¿Qué versión del demonio, qué versión del cliente? ¿Se ejecuta el demonio como `root`?
4. ¿Qué servicio systemd lo pilota? ¿Dónde están sus registros?
5. ¿Quién, en esta máquina, puede hablar con el demonio Docker? ¿Por qué pertenecer al grupo `docker`
   equivale **de hecho** a ser root en el host? Muéstrelo en una orden — luego piense en qué
   implica para la regla `sudo` de sus desarrolladores *(capítulo [A6](06_acl_et_sudo_es.md))*.

---

## Ejercicio 10.2 — Configurar el demonio

1. Cree `/etc/docker/daemon.json` con: rotación de registros (10 Mo, 3 archivos), `live-restore`,
   y las direcciones de sus redes por defecto.
2. **Verifique la sintaxis antes de reiniciar** el demonio. ¿Qué sucede si el archivo no es válido?
3. Reinicie el demonio y verifique que los nuevos valores se toman correctamente.
4. Mueva el `data-root` a `/srv/docker` (una partición dedicada): ¿cuál es el procedimiento completo,
   en qué orden, y qué sucede con los contenedores existentes?
5. ¿Qué hace `live-restore`, y en qué caso es precioso en producción?

---

## Ejercicio 10.3 — Volúmenes y persistencia

1. Liste los volúmenes de la máquina, con su tamaño.
2. Lance un PostgreSQL en un volumen **nombrado**, cree una tabla, destruya el contenedor, lance uno nuevo
   en el mismo volumen: ¿están los datos?
3. Inspeccione el volumen: ¿dónde están realmente los archivos en el host? ¿Quién es el propietario?
4. Monte un archivo de configuración del host **en solo lectura** en un contenedor. Verifique que una
   escritura desde adentro se rechaza.
5. ¿Cuál es la diferencia entre un volumen nombrado y un *bind mount* desde el punto de vista de los permisos,
   los respaldos y SELinux/AppArmor?
6. Encuentre los volúmenes **huérfanos** (no vinculados a ningún contenedor) y su peso total.

---

## Ejercicio 10.4 — Red

1. Liste las redes Docker y su controlador.
2. Cree una red aplicativa `ticketflow-net` y adjunte dos contenedores. Verifique que se alcanzan
   **por su nombre**.
3. ¿Puede un contenedor en la red `bridge` por defecto alcanzar un contenedor de la red aplicativa?
   ¿Por qué esta diferencia es una característica de seguridad?
4. Publique un puerto, luego verifique qué está **realmente expuesto** en el host, y en qué interfaces.
5. Publique un puerto **solo en la boucle local**. ¿En qué caso es el reflejo correcto?
6. La trampa del capítulo: su `ufw` prohíbe el puerto 8080, pero el contenedor es alcanzable desde
   el exterior. Explique por qué, y dé **dos** formas de arreglarlo.
7. Desde un contenedor, resuelva el nombre de otro servicio y pruebe la conexión — sin instalar herramientas
   en la imagen aplicativa *(pista: un contenedor desechable adjunto a la misma red)*.

---

## Ejercicio 10.5 — Los registros

1. Muestre los registros de un contenedor, luego las últimas 50 líneas, luego en seguimiento, luego desde hace 10 minutos.
2. ¿Dónde se almacenan físicamente estos registros en el host? ¿Qué tamaño tienen?
3. Encuentre el contenedor cuyos registros ocupan más espacio.
4. Configure la rotación **globalmente** en `daemon.json`, luego **por contenedor** en un archivo Compose.
   ¿Cuál se aplica a los contenedores ya creados?
5. Envíe los registros de un servicio a `journald` y encuéntrelos con `journalctl`
   *(capítulo [04](../../04_installation_et_services.md))*.
6. Un contenedor llenó `/var/log` — o más bien `/var/lib/docker`. ¿Cuál es el procedimiento inmediato,
   y la corrección durable?

---

## Ejercicio 10.6 — Recursos

1. Muestre el consumo en tiempo real de todos los contenedores.
2. Lance un contenedor limitado a **512 Mo** de memoria y **0,5 CPU**. Verifique el límite desde el host.
3. Provoque un exceso de memoria: ¿qué hace Docker? ¿Dónde lo ve?
4. Limite el número de procesos (`--pids-limit`) y explique contra qué protege.
5. Aplique estos límites en un archivo Compose en lugar de en línea de orden.
6. Configure la política de reinicio `unless-stopped` en un servicio, reinicie la máquina,
   verifique. ¿Cuál es la diferencia con `always`?
7. Añada un `HEALTHCHECK` en línea de orden a un contenedor y encuentre su estado de salud.
   ¿Qué hace Docker con un contenedor `unhealthy` — y qué **no hace**?

---

## Ejercicio 10.7 — Endurecer

1. Verifique bajo qué usuario se ejecuta el proceso principal de cada uno de sus contenedores.
   ¿Cuántos se ejecutan como root?
2. Lance un contenedor con un usuario no privilegiado impuesto desde fuera de la imagen.
3. Lance un contenedor con **sistema de archivos raíz en solo lectura**, con un `tmpfs` para lo que debe
   escribirse.
4. Aplique el endurecimiento estándar: `--cap-drop=ALL` luego las únicas capacidades necesarias,
   `--security-opt=no-new-privileges`.
5. ¿Qué consecuencias tiene `--privileged`? Dé un caso donde es legítimo y tres donde es pereza.
6. Un contenedor que monta `/var/run/docker.sock`: explique precisamente el riesgo.
7. Audite sus contenedores: escriba una línea de orden que liste, para cada uno, su usuario, su
   política de reinicio y sus puertos publicados *(pista: `docker inspect --format`)*.
8. *(Bonus)* Pruebe el modo **rootless**: ¿qué se gana, qué se pierde?

---

## Ejercicio 10.8 — Respaldo y restauración

1. Respalde un **volumen** completo en un archivo `.tar.gz` en el host, pasando por un contenedor desechable.
2. Restaure este archivo en un volumen nuevo y verifique los datos.
3. Haga un **volcado aplicativo** de la base (`pg_dump`) desde el contenedor, sin exponer la contraseña.
   ¿Por qué un volcado aplicativo vale más que una copia del volumen para una base de datos?
4. Exporte una imagen en un archivo, elimine la imagen, reimporte.
5. ¿Cuál es la diferencia entre `docker save`/`load` y `docker export`/`import`?
6. Integre el respaldo de volúmenes a su `backup.sh` del capítulo [A9](09_scripts_administration_es.md).

---

## Ejercicio 10.9 — Limpieza y espacio en disco

1. Muestre qué consume Docker, distribuido por imágenes, contenedores, volúmenes y caché de construcción.
2. Elimine los contenedores detenidos, luego las imágenes sin etiqueta.
3. ¿Cuál es la diferencia exacta entre `docker system prune` y `docker system prune -a`? ¿Cuál es
   peligrosa en un host de producción, y por qué?
4. Escriba una tarea planificada que haga limpieza cada semana — excluyendo explícitamente lo que no debe
   nunca eliminarse. ¿Dónde la instala *(capítulo [04](../../04_installation_et_services.md))*?
5. Pruebe su orden de limpieza **en seco** antes de planificarla: ¿cómo se hace?

---

## Ejercicio 10.10 — Una stack en producción

1. Recupere la stack `05_stack_compose` del repositorio *(ver [`07_docker/05_stack_compose/`](../../07_docker/05_stack_compose/))*
   y despliéguela en `/srv/ticketflow`.
2. Escriba la unidad systemd que inicia esta stack **en el arranque de la máquina** y la detiene correctamente
   en el apagado. Pruebe con un reinicio real.
3. Superponga el archivo de producción: ¿qué diferencias con el archivo de desarrollo, y cómo
   verifica **qué se aplica realmente**?
4. Despliegue una nueva versión de imagen sin interrupción perceptible: ¿cuál es la secuencia?
5. Implante una supervisión mínima: un script que señale todo contenedor detenido o `unhealthy`,
   planificado cada 5 minutos, que escriba en el registro del sistema
   *(capítulo [A9](09_scripts_administration_es.md))*.
6. Los fallos clásicos — para cada uno, la orden de diagnóstico **y** la corrección:
   - el contenedor se reinicia en bucle;
   - no alcanza la base;
   - el puerto ya se usa en el host;
   - el volumen está montado pero vacío;
   - "funcionaba antes de la actualización de la imagen".

---

## ✅ Verificación

- `docker info` muestra la rotación de registros configurada, y ningún contenedor tiene un archivo de registro de varios Go.
- Ningún contenedor aplicativo se ejecuta como root ni en `--privileged`, y puede probarlo en una orden.
- La stack se reinicia toda sola después de un reinicio completo de la VM.
- Un volumen fue respaldado **y restaurado** de verdad.
- La limpieza semanal está planificada y sabe exactamente qué elimina.
- `journal.md` documenta: dónde están los datos, qué se respalda, cómo se restaura.
