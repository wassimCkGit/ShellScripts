# 🛡️ A7 · SSH y cortafuegos

*Capítulo [05 · Seguridad](../../05_securite.md) — partes **cortafuegos**, **SSH** y **fail2ban***

> [!NOTE]
> **Objetivos**: nunca más conectarse por contraseña, endurecer `sshd` sin cortarse el acceso,
> saber rebotar y hacer túneles, cerrar todo lo que no tiene que estar abierto, y hacer callar a los robots.

> [!CAUTION]
> **El capítulo donde uno se encierra. ** El procedimiento, en cada modificación:
> 1. snapshot de la VM;
> 2. una sesión SSH **ya abierta** que no cierre bajo ningún pretexto;
> 3. verificación de sintaxis (`sshd -t`, `nft -c -f`);
> 4. `reload` (no `restart`);
> 5. prueba desde una **tercera** conexión — y solo entonces cierre la sesión de emergencia.

---

## Ejercicio 7.1 — Evaluación de la situación

1. ¿Está en ejecución el servidor SSH? ¿En qué puerto? ¿Qué versión?
2. ¿Qué puertos están en escucha en la máquina, y por qué procesos?
   ¿Cuáles escuchan en **todas** las interfaces en lugar de en la boucle local?
3. Muestre la configuración `sshd` **efectiva** como la entiende el demonio — no el archivo, la
   configuración resuelta.
4. ¿Está activo un cortafuegos? ¿Cuál? ¿Cuáles son sus reglas actuales?
5. Desde su puesto, escanee los puertos abiertos del servidor. Registre el resultado: será su
   "antes / después".

---

## Ejercicio 7.2 — Las claves

1. Genere en su puesto un par de claves `ed25519` con un comentario identificador y una **frase de contraseña**.
2. ¿Por qué `ed25519` en lugar de `rsa` 2048? ¿En qué caso se mantiene RSA?
3. Despliegue la clave pública en la cuenta `alice` del servidor — primero con la herramienta dedicada,
   luego **a mano** (para entender qué hace).
4. ¿Cuáles son los permisos **exactos** esperados en `~/.ssh` y en `~/.ssh/authorized_keys`?
   ¿Qué sucede si son demasiado amplios? Pruebe.
5. Conéctese por clave, luego verifique en los registros del servidor que la autenticación pasó
   por la clave.
6. Añada su clave al agente SSH para no tener que escribir la frase de contraseña cada vez.
7. Muestre la huella de su clave pública. ¿Para qué sirve en un inventario?

---

## Ejercicio 7.3 — Endurecer `sshd`

Un punto a la vez, con verificación de sintaxis y prueba entre cada:

1. Prohibir completamente la conexión de `root`.
2. Desactivar la autenticación por **contraseña** (y la autenticación interactiva por teclado).
3. Restringir la conexión a solo miembros del grupo `ops`, más la cuenta `backup`.
4. Reducir la superficie de ataque: sin transferencia X11, sin `AllowAgentForwarding`, sin
   `PermitEmptyPasswords`, bajo `MaxAuthTries`.
5. Desconectar sesiones inactivas después de 10 minutos.
6. Pasar el registro a `VERBOSE` — ¿qué se añade en los registros, y por qué es útil?
7. Mostrar un aviso legal antes de la conexión.
8. Cambiar el puerto de escucha a 2222. ¿Qué trae realmente esto, y qué no? ¿Qué hay que acordarse de modificar
   **al mismo tiempo** (cortafuegos, fail2ban, `~/.ssh/config`, SELinux si aplica)?

### Ejercicio 7.4 — Los bloques `Match`

1. La cuenta `backup` solo debe poder lanzar **`rsync`**, sin shell interactivo.
2. Los miembros de `ops` conectados **desde la red interna** tienen derechos a túneles; los otros no.
3. Una cuenta `sftp-client` está confinada a su carpeta por `chroot`, sin acceso shell.
4. Verifique cada regla conectándose realmente con la cuenta correspondiente.
5. Restrinja, **en `authorized_keys`**, la clave de despliegue: una sola orden autorizada, desde una
   IP dada, sin túnel ni asignación de terminal. ¿Por qué esta restricción es complementaria de
   la del `Match`?

### Ejercicio 7.5 — El cliente, túneles y transferencias

1. Escriba un `~/.ssh/config` en su puesto: un alias `srv-tickets` con el host, puerto, usuario
   y clave; valores comunes a todos los hosts; reutilización de conexión.
2. Luego conéctese simplemente con `ssh srv-tickets`.
3. Alcance una segunda máquina **vía** el servidor (rebote).
4. Abra un túnel **local** haciendo accesible la base PostgreSQL del servidor (que solo escucha en `127.0.0.1`)
   desde su puesto en el puerto 5433. Verifique con un cliente.
5. Abra un proxy SOCKS e haga pasar un navegador por el servidor.
6. Transfiera un archivo con `scp`, luego una carpeta con `rsync`. ¿Por qué `rsync` en cuanto la operación
   se repite?
7. `known_hosts`: ¿qué sucede si la huella del servidor cambia? ¿Cómo se reacciona **correctamente**
   (y por qué eliminar la línea sin pensar es una mala respuesta)?

---

## Ejercicio 7.6 — El cortafuegos

Elija `ufw` **o** `nftables` y haga todo el ejercicio con la misma herramienta.

1. Muestre las reglas actuales y la política por defecto.
2. Política por defecto: **rechazar todo en entrada**, autorizar todo en salida.
3. Autorice SSH **ante todo**. ¿Por qué este orden es innegociable?
4. Autorice HTTP y HTTPS desde cualquier lugar.
5. Autorice PostgreSQL **solo desde la subred interna**.
6. Autorice el puerto de administración solo desde su IP, con un **comentario** en la regla.
7. Active el cortafuegos, luego verifique desde su puesto qué es accesible y qué ya no.
8. Limite el flujo de conexiones SSH para ralentizar robots.
9. Registre los paquetes rechazados, con límite de flujo. ¿Dónde aterrizan estas líneas?
10. Elimine una regla que ya no sirve, liste las reglas **numeradas**, y haga la configuración
    persistente al reinicio. Verifíquelo reiniciando realmente la VM.
11. El servicio está en ejecución pero no es accesible desde el exterior. Dé las **cuatro** hipótesis a probar,
    en orden, y la orden para cada una.

---

## Ejercicio 7.7 — `fail2ban`

1. Instale e inicie `fail2ban`, luego muestre la lista de cárceles activas.
2. ¿Por qué se configura `jail.local` y no `jail.conf`?
3. Configure la cárcel SSH: 5 intentos, ventana de 10 minutos, bloqueo de 1 hora,
   teniendo en cuenta **su puerto** SSH.
4. Añada su red de administración a la lista blanca — el olvido clásico.
5. Provoque voluntariamente un bloqueo desde otra máquina, constate, luego desbloquee la IP.
6. Muestre las estadísticas de la cárcel: ¿cuántos intentos, cuántos bloqueos?
7. Verifique cómo `fail2ban` se articula con su cortafuegos: ¿dónde se escriben realmente las reglas
   de bloqueo?

---

## ✅ Verificación

- Un `ssh root@srv-tickets` falla; un `ssh -o PreferredAuthentications=password alice@…` también.
- Se conecta por clave, en el nuevo puerto, escribiendo `ssh srv-tickets`.
- Un escaneo de puertos desde el exterior solo muestra lo que explicita abrió —
  compárelo con el relevamiento del ejercicio 7.1.
- El cortafuegos sobrevive a un reinicio completo de la VM.
- Cinco fallos de conexión desde una IP de prueba la hacen bloquear, y su red de admin no puede
  ser bloqueada.
- `journal.md` documenta: puerto SSH, cuentas autorizadas, puertos abiertos y para quién, procedimiento de emergencia.
