# 📦 A5 · Paquetes, servicios y tareas planificadas

*Capítulo [04 · Instalación de paquetes, servicios y tareas planificadas](../../04_installation_et_services.md)*

> [!NOTE]
> **Objetivos**: dominar el ciclo de vida de los paquetes, pilotar y diagnosticar servicios, escribir una
> unidad systemd correcta para la aplicación interna, explotar `journalctl`, y planificar lo que debe
> planificarse — con `cron` como con los *timers* systemd.

> [!IMPORTANT]
> Se requiere una **máquina virtual real**: un contenedor no ejecuta `systemd`.

---

## Ejercicio 5.1 — Paquetes

1. Actualice el índice de paquetes, luego muestre la lista de paquetes que **pueden** actualizarse,
   sin instalar nada.
2. Aplique las actualizaciones de seguridad. ¿Cuál es la diferencia entre `upgrade`, `full-upgrade` y
   `dist-upgrade`?
3. Instale `nginx`, `postgresql-client`, `ufw`, `fail2ban` y `acl` en una orden.
4. Busque el paquete que proporciona la orden `ss`, luego muestre su información detallada.
5. Liste los archivos instalados por el paquete `nginx`. ¿A qué paquete pertenece `/usr/bin/systemctl`?
6. Elimine `nginx` conservando su configuración, luego elimínelo **con** su configuración.
   ¿Cuál es la diferencia, y cuál se usa cuando se quiere empezar de cero?
7. Haga limpieza de dependencias que se volvieron innecesarias y de la caché de descargas.
8. Un paquete fue instalado manualmente: ¿cómo encontrar la lista de paquetes **instalados
   explícitamente** por un humano, por oposición a las dependencias?
9. Impida la actualización automática de un paquete específico *(pista: `apt-mark hold`)*. ¿En qué caso
   se hace esto, y cuál es el riesgo?

---

## Ejercicio 5.2 — Pilotar servicios

1. Liste todos los servicios **activos**, luego todos los que están **en fallo**.
2. Muestre el estado de `ssh`: ¿está en ejecución? ¿está activado al arranque? ¿desde cuándo? ¿cuál es su PID?
3. Detenga, inicie, reinicie `nginx`. Recargue su configuración **sin** cortar las conexiones en curso.
4. ¿Cuál es la diferencia exacta entre `restart` y `reload`? ¿Y entre `enable` y `start`?
5. Active `nginx` al arranque **e** inícielo en una sola orden.
6. Enmascare un servicio para evitar toda activación, incluso por dependencia. Luego desenmáscarelo.
   ¿En qué caso se usa `mask` en lugar de `disable`?
7. Muestre la lista de dependencias de `nginx.service`, luego qué depende de él.
8. ¿Cuánto tiempo tomó el último arranque de la máquina, y qué servicios fueron los más lentos?
   *(Pista: `systemd-analyze`.)*

---

## Ejercicio 5.3 — Una unidad para la aplicación interna

La API `ticketflow` debe ejecutarse permanentemente, bajo una cuenta dedicada, y reiniciarse si se bloquea.

```bash
sudo tee /usr/local/bin/ticketflow-api >/dev/null <<'EOF'
#!/bin/bash
echo "iniciando API ticketflow (pid $$)"
trap 'echo "parada solicitada"; exit 0' TERM INT
while true; do
  echo "$(date '+%F %T') api ok"
  sleep 15
done
EOF
```

1. Establezca el propietario y los permisos correctos en este script para un ejecutable del sistema.
2. Escriba `/etc/systemd/system/ticketflow-api.service` con:
   - una descripción clara;
   - un arranque **después de la red** y después de `postgresql.service`;
   - `Type=simple`, ejecución bajo el usuario `deploy` y el grupo `ops`;
   - un reinicio automático en caso de fallo, con un retraso de 5 segundos;
   - el directorio de trabajo `/srv/ticketflow`;
   - una variable de entorno `NODE_ENV=production`;
   - una instalación en `multi-user.target`.
3. Recargue la configuración de systemd, active el servicio en el arranque **e** inícielo en una orden.
4. Verifique su estado y lea sus registros.
5. Mate bruscamente el proceso: ¿lo reinicia systemd? ¿Después de cuánto tiempo?
6. Añada tres directivas de endurecimiento (`NoNewPrivileges`, `PrivateTmp`, `ProtectSystem`) y verifique
   que el servicio sigue reiniciándose. ¿Qué hace cada una?
7. Haga leer la configuración desde un archivo de entorno `/etc/default/ticketflow`
   en lugar de en la unidad.
8. ¿Cómo sobrecargar una directiva de una unidad **proporcionada por un paquete** sin modificar su archivo
   original (que sería sobrescrito en la próxima actualización)? *(Pista: `systemctl edit`.)*

---

## Ejercicio 5.4 — `journalctl`

1. Los registros del servicio `ticketflow-api`, luego sus últimas 50 líneas, luego en seguimiento en tiempo real.
2. Los registros desde "hace 15 minutos", luego los de hoy, luego los del **arranque anterior**.
3. Solo los mensajes de nivel `error` y superiores, para todo el sistema.
4. Los registros del kernel únicamente.
5. Los registros de un usuario específico, luego de un PID específico.
6. ¿Cuánto espacio ocupa el diario en disco? Límitelo a 500 Mo de forma **permanente**.
7. ¿Es el diario persistente después de un reinicio en esta máquina? ¿Cómo hacerlo persistente?
8. Purgue las entradas de más de 30 días.
9. Exporte los registros del servicio del día a un archivo, para adjuntarlos a un informe de incidente.

---

## Ejercicio 5.5 — `cron`

1. Muestre su crontab, luego la del usuario `deploy`.
2. Escriba, en la crontab de `root`, una tarea que lance `/usr/local/bin/backup.sh`
   **todos los días a las 2:30**, redirigiendo salida estándar **y** errores a un archivo de registro.
3. Una tarea que, **cada 10 minutos**, escriba el espacio libre en disco en un archivo.
4. Una tarea que, **de lunes a viernes, cada hora de 8 a 18**, lance un control de salud.
5. Una tarea que, **el primero de cada mes**, archive los registros del mes anterior.
6. Una tarea que se ejecute **en cada arranque** de la máquina.
7. Escriba el mismo respaldo diario como fragmento `/etc/cron.d/ticketflow-backup` —
   ¿qué columna adicional requiere este formato?
8. ¿Para qué sirven `/etc/cron.daily` y similares? ¿Cuándo se prefieren a una línea de crontab?
9. Su tarea "no funciona" aunque la orden funciona en su terminal.
   Cite las **tres** causas más frecuentes y cómo las verifica.
10. Prohiba el uso de `cron` a todos los usuarios excepto `root` y `deploy`
    *(pista: `/etc/cron.allow`)*.

---

## Ejercicio 5.6 — Los *timers* systemd

1. Liste los *timers* activos de la máquina y la próxima fecha de cada uno.
2. Escriba el par `ticketflow-backup.service` + `ticketflow-backup.timer` que reemplaza la tarea cron
   del ejercicio 5.5.2: tipo `oneshot`, disparo diario a las 2:30, con recuperación si la máquina
   estaba apagada *(pista: `Persistent=true`)*.
3. Active el *timer*, verifique su próxima ejecución, dispare el servicio manualmente para probar.
4. Cite tres ventajas de un *timer* systemd sobre una línea de crontab.

---

## Ejercicio 5.7 — `logrotate`

1. Observe cómo se organiza `/etc/logrotate.d/`.
2. Escriba `/etc/logrotate.d/ticketflow`: rotación diaria de registros de `/var/log/ticketflow`,
   7 rotaciones conservadas, compresión, sin error si el archivo está ausente, y creación del nuevo
   archivo con los permisos y propietario correctos.
3. Pruebe su configuración **en seco**, sin hacer nada *(pista: `-d`)*.
4. Force una rotación inmediata para verificar el resultado.
5. ¿Por qué una aplicación que mantiene su archivo de registro abierto puede seguir escribiendo en el archivo
   antiguo después de la rotación, y qué directiva arregla el problema?

---

## ✅ Verificación

- `systemctl status ticketflow-api` muestra un servicio activo, endurecido, ejecutándose bajo `deploy`, reiniciado
  automáticamente después de un `kill -9`.
- `systemctl --failed` no devuelve nada.
- El respaldo diario existe **bien** en cron, **bien** en *timer*, y puede justificar su elección.
- `journalctl -u ticketflow-api --since today` muestra huellas de sus pruebas.
- `logrotate -d /etc/logrotate.d/ticketflow` no reporta errores.
- Su `journal.md` lista todo lo que está planificado en la máquina, con hora y razón.
