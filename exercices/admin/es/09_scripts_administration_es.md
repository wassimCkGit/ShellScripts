# 📜 A9 · Scripts de administración

*Capítulo [06 · Shell scripting](../../06_shellscript/) — los seis subcapítulos*

> [!NOTE]
> **Objetivos**: automatizar lo que hace dos veces. Auditar cuentas, respaldar con rotación,
> supervisar la máquina, crear cuentas en serie — con scripts que fallan correctamente, se explican
> y no dejan nada atrás.

> [!IMPORTANT]
> Reglas válidas para todos los scripts del capítulo:
> - shebang, ejecutable, en `/usr/local/bin` o `/srv/ticketflow/scripts`;
> - `set -euo pipefail` al inicio;
> - toda variable entrecomillada;
> - un script de administración **no destruye nada** sin un `--dry-run` disponible y una confirmación;
> - pasa [ShellCheck](https://www.shellcheck.net/) sin advertencias.

---

## 1 · Los fundamentos *(capítulo [06.1](../../06_shellscript/01_Base/base.md))*

### Ejercicio 9.1 — `sysinfo.sh`

El script que lanzará al llegar a una máquina desconocida.

1. Muestre, presentado correctamente: nombre de la máquina, distribución, kernel, *uptime*, carga promedio,
   memoria libre, ocupación de `/`, número de usuarios conectados, número de servicios en fallo.
2. Cada valor viene de una orden cuya salida captura en una variable.
3. Añada un encabezado con la fecha y el nombre del servidor, y un separador.
4. Haga que escriba su informe **en pantalla y** en `/var/log/ticketflow/sysinfo-<fecha>.txt`.
5. Almacene los umbrales de alerta (disco, carga) en variables al inicio del archivo, para que se
   modifiquen sin releer el código.

### Ejercicio 9.2 — Matrices

1. Cree una matriz `SERVICIOS_CRITICOS` que contenga `ssh`, `nginx`, `postgresql` y `ticketflow-api`.
2. Muestre el número de elementos, luego cada elemento.
3. Rellene una matriz con la lista de usuarios humanos de la máquina
   *(capítulo [03](../../03_commandes_essentielles.md))* y muéstrela.
4. Añada un servicio a la matriz, retire uno, vuelva a mostrar.

---

## 2 · Condiciones *(capítulo [06.2](../../06_shellscript/02_operateur_logique/exemple.md))*

### Ejercicio 9.3 — `check-server.sh`

1. El script debe rechazar ejecutarse si **no** está lanzado como root. ¿Cómo se prueba?
2. Verifique que `/srv/ticketflow` existe y es una carpeta.
3. Verifique que el archivo de configuración existe y es **legible**.
4. Verifique que la orden `pg_dump` está disponible.
5. Verifique que la ocupación de `/` está por debajo del umbral definido; si no, muestre una alerta.
6. Verifique que una variable de entorno obligatoria está definida **y no vacía**.
7. Escriba una prueba que compare dos números (espacio libre y umbral) y una prueba que compare dos cadenas
   (entorno). ¿Por qué los operadores no son los mismos?
8. Escriba, en una sola línea, "crea la carpeta de respaldo si no existe", luego
   "lanza el respaldo, y si falla, escribe una alerta y sale en error".

---

## 3 · Bucles *(capítulo [06.3](../../06_shellscript/03_boucles/exemple.md))*

### Ejercicio 9.4 — Recorrer

1. Bucle sobre `SERVICIOS_CRITICOS` y muestre, para cada uno, `✔` o `✘` según esté activo.
2. Bucle sobre los *homes* de los usuarios y muestre, para cada uno, su peso
   *(capítulo [03](../../03_commandes_essentielles.md))*.
3. Bucle sobre los archivos de `/etc/sudoers.d` y muestre el nombre de cada uno con sus permisos.
4. Bucle sobre una lista de direcciones IP y pruebe si cada una responde al ping, con un resumen final.

### Ejercicio 9.5 — Leer un archivo, esperar

1. Lea `/etc/passwd` **línea por línea** y muestre, para solo las cuentas humanas, el nombre y el shell.
2. Lea un archivo `usuarios.csv` (`nombre;grupo;comentario`) y cree las cuentas correspondientes.
   Salte líneas vacías y comentarios (`continue`), deténgase si una línea está mal formada (`break`).
3. Escriba un bucle que espere que un servicio esté realmente `active`, con un **límite máximo** y un
   mensaje de fallo después. ¿Por qué una espera sin límite es inaceptable en un script de operación?
4. Reescriba esta espera con `until`.

### Ejercicio 9.6 — `backup.sh` y la rotación

1. Escriba un script que archive `/srv/ticketflow` y `/etc` en
   `/srv/ticketflow/backups/backup-AAAAMMJJ-HHMMSS.tar.gz`, **preservando permisos y ACL**.
2. Añada un volcado de la base PostgreSQL, **nunca haciendo aparecer la contraseña en la línea
   de orden** *(pista: un archivo de entorno descargado, o `~/.pgpass`)*.
3. Conserve solo los **últimos 7** respaldos, elimine los otros.
4. Escriba el tamaño y la duración de cada respaldo en un archivo de registro.
5. Produzca una suma de control por archivo, y una orden de verificación.

---

## 4 · Gestión de errores *(capítulo [06.4](../../06_shellscript/04_gestion_des_erreurs/gestion_des_erreurs.md))*

### Ejercicio 9.7 — Fallar correctamente

1. Añada `set -euo pipefail` a `backup.sh` y constate qué se rompe — corrija lo que deba.
2. Desactive temporalmente la parada automática para **una sola** orden cuyo fallo es aceptable.
3. Escriba `error()`: mensaje prefijado `[ERROR]` en la **salida de error**, código de salida no nulo.
4. Haga que el script salga con un código **diferente** según la causa: 1 (argumentos),
   2 (requisito faltante), 3 (respaldo fallido). ¿Por qué es útil cuando cron o *timer* systemd lo llama?
5. Un respaldo interrumpido **nunca debe** dejar un archivo incompleto: escriba en un archivo
   temporal rebautizado solo en caso de éxito.
6. Con `trap`, garantice la eliminación de la carpeta temporal al final del script, en caso de error,
   y en `CTRL+C`. Verifique los tres casos.
7. Haga que un error envíe una línea al registro del sistema *(pista: `logger`)*.
   ¿Dónde lo encuentra después?

---

## 5 · Argumentos *(capítulo [06.5](../../06_shellscript/05_Arguments/args.md))*

### Ejercicio 9.8 — `backup.sh` como verdadera orden

```text
Uso: backup.sh -d <destino> [-k <número>] [-t completo|base|conf] [-n] [-v] [-a]
  -d DEST   carpeta destino                           (obligatorio)
  -k N      número de respaldos conservados           (defecto: 7)
  -t TIPO   qué se respalda: completo | base | conf   (defecto: completo)
  -n        dry-run: muestra qué se haría
  -v        modo verboso
  -a        ayuda
```

1. Implemente el análisis con `getopts`.
2. `-a` muestra la ayuda y sale con código `0`; la ausencia de `-d` es un error.
3. Un valor de `-t` desconocido se rechaza con un mensaje claro.
4. `-k` debe ser un **entero positivo**: valídelo.
5. En modo `-n`, no hay escritura ni eliminación — cada acción se anuncia, prefijada `[dry-run]`.
6. Pruebe: `-n -t base`, `-k abc`, `-t plop`, sin opción, `-a`.

### Ejercicio 9.9 — `useradd-batch.sh`

1. Un script que recibe un archivo CSV como argumento y crea las cuentas que describe.
2. Verifica **antes** de crear: la cuenta no existe ya, el grupo existe, el nombre es válido.
3. Genera una contraseña aleatoria por cuenta, fuerza su cambio en la primera conexión,
   y escribe los identificadores en un archivo legible **solo por root**.
4. Produce un reporte final: creados, ignorados, en error.
5. Tiene un `--dry-run`.

---

## 6 · Funciones *(capítulo [06.6](../../06_shellscript/06_fonctions/fonctions.md))*

### Ejercicio 9.10 — Dividir y compartir

1. Divida `backup.sh` en funciones: `usage`, `log`, `error`, `verifier_prerequis`, `sauver_fichiers`,
   `sauver_base`, `rotation`, `nettoyage`.
2. El cuerpo del script es solo una serie de llamadas.
3. `log()` horodata sus mensajes y escribe tanto a pantalla como al archivo de registro.
4. `verifier_prerequis()` devuelve un **código** que el script explota.
5. Escriba una función que **devuelva un valor**: `espace_libre()` devuelve el porcentaje de ocupación
   de la partición pasada como argumento.
6. Use `local` en todas partes; muestre, con un ejemplo, qué se rompe si lo olvida.

### Ejercicio 9.11 — `lib-admin.sh`

1. Cree `/usr/local/lib/lib-admin.sh` con `log`, `error`, `confirmar`, `require_root`, `require_cmd`,
   `service_actif`, `alerta` (envío de una línea al registro del sistema).
2. El archivo **no debe ejecutar nada** cuando se descarga.
3. Descargue desde `backup.sh`, `check-server.sh` y `useradd-batch.sh`, y elimine todo el código duplicado.
4. Haga que el import funcione cualquiera que sea la carpeta desde la que se lanza el script.
5. Añada color — desactivado automáticamente cuando la salida no es un terminal.
   ¿Por qué es imprescindible para un script llamado por cron?

### Ejercicio 9.12 — Puesta en producción

1. Instale `backup.sh` en `/usr/local/bin` con el propietario y los permisos correctos
   *(capítulo [02](../../02_file_permissions.md))*.
2. Planifíquelo diariamente — en cron **o** en *timer* systemd *(capítulo [04](../../04_installation_et_services.md))*.
3. Verifique al día siguiente que la tarea pasó: ¿dónde mira?
4. Provoque un fallo (destino inexistente) y verifique que se **le avisa** sin tener que mirar.
5. Escriba el procedimiento de **restauración** en `journal.md`, y pruébelo realmente.
   Un respaldo que nunca ha restaurado no es un respaldo.

---

## ✅ Verificación

- `/usr/local/bin` contiene `sysinfo.sh`, `check-server.sh`, `backup.sh`, `useradd-batch.sh`, y
  `/usr/local/lib/lib-admin.sh` contiene todo el código común.
- `shellcheck` no devuelve nada en ninguno de ellos.
- `backup.sh -a` se explica por sí solo; `backup.sh` sin argumento falla con código no nulo.
- `backup.sh -n` no modifica nada en absoluto.
- Un `CTRL+C` a mitad del respaldo no deja ni archivo incompleto ni carpeta temporal.
- El respaldo se ejecuta solo, falla ruidosamente, y ha **restaurado** al menos una vez.
