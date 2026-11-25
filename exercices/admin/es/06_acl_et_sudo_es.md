# 🎛️ A6 · ACL y delegación `sudo`

*Capítulo [05 · Seguridad](../../05_securite.md) — partes **ACL** y **sudo / sudoers***

> [!NOTE]
> **Objetivos**: ir más allá de las limitaciones de los permisos POSIX con ACL (máscara, herencia, copia de seguridad),
> y delegar derechos de administración precisos sin distribuir `ALL=(ALL) ALL` a todo el equipo.

> [!CAUTION]
> `sudoers` es el archivo que puede encerrarlo más rápidamente. **Snapshot antes**, una sesión root
> abierta permanentemente en un segundo terminal, y nunca editar sin `visudo`.

---

## Parte 1 — ACL

### Ejercicio 6.1 — Requisitos previos

1. Verifique que el paquete `acl` está instalado.
2. Verifique que el sistema de archivos que lleva `/srv` está montado con soporte ACL.
   ¿Cómo hacerlo permanente si no es así?
3. Cree `/srv/projet` y ubique, en un `ls -l`, el signo que indica que un archivo tiene ACL.

### Ejercicio 6.2 — Leer y establecer

1. Muestre la ACL completa de `/srv/projet`. ¿Qué significa cada línea cuando no se establece ninguna ACL extendida?
2. Dé a `bob` el derecho de **leer y escribir** en `/srv/projet`, sin ponerlo en el grupo propietario.
3. Dé al grupo `dev` derechos de **solo lectura** en este mismo directorio.
4. Relea la ACL: ¿cuál es el valor de la `máscara`? ¿Qué sucede si la establece a `r--`?
5. Retire la entrada de `bob`, luego elimine **todas** las ACL extendidas del directorio.

### Ejercicio 6.3 — Herencia

1. Establezca una ACL **por defecto** en `/srv/projet`: todo archivo nuevo debe ser legible y modificable
   por el grupo `dev`.
2. Cree un archivo con la cuenta `alice` y otro con `bob`: verifique que ambos tengan bien
   la ACL heredada.
3. Añada el **SGID** en el directorio para que el grupo propietario se conserve, y explique la
   diferencia entre lo que hace SGID y lo que hace la ACL por defecto.
4. ¿Hereda un archivo **copiado** en el directorio la ACL? ¿Y un archivo **movido**?
   Pruebe ambos casos y explique.

### Ejercicio 6.4 — El caso práctico completo

Pliego de condiciones para `/srv/projet`:

| Quién | Derecho esperado |
|-------|------------------|
| `alice` (ops) | todo, incluida la administración del directorio |
| grupo `dev` | lectura / escritura en archivos, creación de archivos |
| `deploy` | solo lectura |
| todos los demás | nada, ni siquiera listar |

1. Escriba la secuencia de órdenes que lo implementa, ACL de acceso **y** ACL por defecto.
2. Verifique cada línea de la tabla conectándose con las cuentas correspondientes.
3. **Respalde** las ACL del directorio en un archivo, elimínelas todas, luego **restaure**
   desde la copia de seguridad.
4. Copie la arborescencia a `/srv/projet-copie` **conservando las ACL**: ¿qué opción de `cp`
   (u otra orden) se necesita?
5. ¿Por qué las ACL deben aparecer en su procedimiento de respaldo? ¿Qué sucede si restaura un `tar`
   hecho sin la opción adecuada?

---

## Parte 2 — `sudo`

### Ejercicio 6.5 — La situación actual

1. Muestre lo que **usted** tiene derecho a hacer con `sudo`.
2. ¿Quién es miembro del grupo `sudo` en esta máquina?
3. ¿Dónde se definen las reglas? ¿Cuál es el orden de lectura, y por qué la línea de inclusión de fragmentos
   debe ser **al final**?
4. Produzca la lista de **todas** las reglas efectivas, en todos los archivos, luego la de las reglas
   `NOPASSWD` — estos son sus puntos de auditoría prioritarios.

### Ejercicio 6.6 — Editar sin encerrarse

1. ¿Por qué nunca se edita `/etc/sudoers` con `nano` directamente?
2. Cree el fragmento `/etc/sudoers.d/ops` **con la herramienta dedicada**, y establezca sus permisos correctos
   (¿cuáles son, y qué sucede si son demasiado amplios?).
3. Verifique la sintaxis del conjunto de la configuración **sin** aplicarla.
4. Introduzca voluntariamente un error de sintaxis en un fragmento, constate lo que dice la verificación,
   y corrija.

### Ejercicio 6.7 — Delegar con precisión

Escriba las reglas correspondientes a cada una de estas necesidades, cada una en su propio fragmento nombrado:

1. El grupo `ops` puede hacer todo, con contraseña.
2. El grupo `dev` puede **solo** reiniciar y consultar el estado de `nginx` y `ticketflow-api`,
   y leer sus registros — nada más.
3. La cuenta `deploy` puede lanzar **un solo** script de despliegue, **sin contraseña** (es llamado
   por CI, sin terminal).
4. La cuenta `backup` puede lanzar `rsync` y `tar` en lectura, sin contraseña.
5. `bob` puede editar los archivos de configuración de su aplicación — **con `sudoedit`**, no con un
   editor lanzado como root. Explique la diferencia de riesgo.
6. Defina un alias de órdenes `SERVICES` y un alias de usuarios `ADMINS`, y reescriba dos de las
   reglas anteriores usándolos.
7. Establezca los `Defaults`: caché de autenticación de 5 minutos, registro de órdenes,
   mensaje de advertencia personalizado en el primer `sudo`.

### Ejercicio 6.8 — Las trampas

Para cada uno de estos ejemplos, explique **por qué la regla es peligrosa** y proponga una versión correcta:

1. `bob ALL=(ALL) NOPASSWD: /usr/bin/vim`
2. `%dev ALL=(ALL) NOPASSWD: /bin/chmod`
3. `deploy ALL=(ALL) NOPASSWD: /usr/local/bin/deploy.sh` — siendo el script modificable por `deploy`.
4. `carine ALL=(ALL) NOPASSWD: /usr/bin/find`
5. `%dev ALL=(ALL) ALL` "en espera, afinaremos después".

### Ejercicio 6.9 — Auditoría y registro

1. Encuentre en los registros todas las órdenes lanzadas con `sudo` hoy, y por quién
   *(capítulo [03](../../03_commandes_essentielles.md))*.
2. Encuentre los **intentos rechazados** (usuario no en `sudoers`, u orden no autorizada).
3. Active el registro de entrada / salida de una sesión `sudo` para una cuenta sensible,
   y relea una sesión registrada.
4. Escriba el procedimiento de emergencia: **acaba de romper `sudoers` y no tiene más sesión root
   abierta**. Liste, en orden, lo que intenta.

---

## ✅ Verificación

- `getfacl /srv/projet` refleja exactamente el cuadro del ejercicio 6.4, y un archivo nuevo creado por
  cualquier miembro de `dev` hereda los derechos correctos.
- `visudo -c` está limpio, y cada derecho delegado vive en su propio fragmento de `/etc/sudoers.d/`.
- `sudo -l` ejecutado con `bob`, `deploy` y `backup` muestra **exactamente** lo que el pliego prevé.
- Ninguna regla `NOPASSWD` toca un editor, intérprete o script modificable por quien la llama.
- Su procedimiento de emergencia está escrito en `journal.md` **antes** de necesitarlo.
