# 👥 A3 · Usuarios, grupos y cuentas

*Capítulo [02.1 · Gestión de usuarios y grupos](../../02.1_user_management.md)*

> [!NOTE]
> **Objetivos**: crear y eliminar cuentas correctamente, comprender los archivos que las describen, separar
> cuentas humanas y cuentas de servicio, gestionar contraseñas y vencimientos, y dejar de trabajar en `root`.

> [!CAUTION]
> Snapshot de la VM **antes** de comenzar. Se bloquean cuentas y se modifica `sudo`:
> mantenga permanentemente una sesión privilegiada abierta que no cierre.

---

## Ejercicio 3.1 — Lo existente

1. Liste todas las cuentas de la máquina. ¿Cuántas hay?
2. Distinga las cuentas **humanas** de las cuentas **sistema**: ¿qué criterio usa?
   Produzca la lista de solo las cuentas humanas *(capítulo [03](../../03_commandes_essentielles.md):
   `awk` sobre `/etc/passwd`)*.
3. ¿Qué contiene cada uno de los siete campos de una línea de `/etc/passwd`?
4. ¿Por qué la contraseña ya no está allí, y dónde está? ¿Quién puede leer este archivo, y con qué permisos?
5. Liste todos los grupos, luego los de su usuario.
6. ¿Qué cuentas tienen un shell de conexión válido? ¿Cuáles tienen `/usr/sbin/nologin`, y por qué?

---

## Ejercicio 3.2 — Crear las cuentas del equipo

El equipo: `alice` (administradora), `bob` y `carine` (desarrolladores), más dos cuentas técnicas.

1. Cree el grupo `ops` y el grupo `dev`.
2. Cree `alice`, `bob` y `carine` **con** su *home*, el shell `/bin/bash` y un comentario
   (nombre completo) completado.
3. Ponga `alice` en `ops`, `bob` y `carine` en `dev` — **sin** sacarlos de su grupo principal.
4. Añada `alice` al grupo que da derechos `sudo` en Debian. Verifique con ella.
5. Cree la cuenta técnica `deploy`: **no hay *home* interactivo necesario**, shell `/usr/sbin/nologin`,
   grupo `ops`. ¿Por qué una cuenta de servicio no debe tener shell de conexión?
6. Cree la cuenta `backup`, sin contraseña utilizable, destinada a ser usada **solo** por clave SSH.
7. Muestre, para cada cuenta creada, su UID, su GID y sus grupos secundarios.

---

## Ejercicio 3.3 — Contraseñas y vencimiento

1. Establezca una contraseña para `bob`.
2. Force a `bob` a cambiar su contraseña **en su próxima conexión**.
3. Muestre la política de vencimiento de la cuenta de `bob` (última modificación, vencimiento, advertencia).
4. Imponga: contraseña válida 90 días, advertencia 7 días antes, mínimo 1 día entre dos cambios.
5. Establezca una **fecha de vencimiento de la cuenta** para `carine` el último día del próximo mes
   (una pasante, no olvidará cerrar la cuenta).
6. **Bloquee** la cuenta de `bob`, intente conectarse, luego desbloqúela.
   ¿Cuál es la diferencia entre bloquear la contraseña y deshabilitar el shell?
7. ¿Dónde se configura la política de contraseñas **por defecto** para las nuevas cuentas?

---

## Ejercicio 3.4 — Modificar y eliminar

1. Renombre `carine` a `carine.dupont`, renombrando también su *home*.
2. Cambie el shell predeterminado de `bob` a `/bin/sh`, luego vuelva a `/bin/bash`.
3. Mueva el *home* de `bob` a `/srv/home/bob` conservando sus archivos.
4. Retire `bob` del grupo `dev` sin tocar sus otros grupos.
5. Elimine la cuenta `carine.dupont` **con** su *home* y su bandeja de correo.
6. Después de la eliminación, persisten archivos de ella en otros lugares del disco.
   ¿Cómo los encuentra *(capítulo [03](../../03_commandes_essentielles.md))*? ¿Qué sucede si
   crea una nueva cuenta que recupera el mismo UID?

---

## Ejercicio 3.5 — El esqueleto de las nuevas cuentas

1. ¿Qué contiene `/etc/skel` en su máquina?
2. Añádale un archivo `README-serveur.txt` recordando las reglas internas, y un `.bashrc` ampliado con un
   alias personalizado.
3. Cree una nueva cuenta de prueba y verifique que hereda bien estos archivos.
4. ¿Heredan las cuentas **ya existentes**? ¿Cómo aplicarles la modificación?
   *(Ver el capítulo [10](../../10_shell_config.md) y el ejercicio [A11](11_environnement_utilisateurs_es.md).)*
5. ¿Dónde se configura el *home* predeterminado, el shell predeterminado y el rango de UID de las nuevas cuentas?

---

## Ejercicio 3.6 — Cambiar de identidad correctamente

1. Ejecute **una sola** orden como root, sin abrir una sesión root.
2. Abra una sesión root completa, con un entorno limpio. ¿Cuál es la diferencia con `su root` sin guión?
3. Abra una sesión como `deploy` aunque no tenga shell de conexión.
   *(Pista: una opción de `su` permite forzar el shell.)* ¿En qué caso de explotación se necesita?
4. Verifique "quién es realmente" en una sesión `sudo -i`: `whoami`, `logname`, `id`.
   ¿Cuál de estas órdenes rastrea al usuario original?
5. Encuentre en `/var/log/auth.log` todas las elevaciones de privilegios del día
   *(capítulo [03](../../03_commandes_essentielles.md))*.
6. ¿Por qué se deja `root` sin contraseña utilizable en un servidor bien mantenido, y por qué se reemplaza?
   *(Lo siguiente está en [A6](06_acl_et_sudo_es.md).)*

---

## ✅ Verificación

- `alice`, `bob`, `deploy` y `backup` existen con el shell correcto, los grupos correctos y la política
  de vencimiento correcta.
- Ninguna cuenta de servicio tiene shell de conexión.
- Sabe producir, en una línea, la lista de cuentas humanas de la máquina.
- Sabe responder a "¿quién hizo eso, y cuándo?" leyendo `/var/log/auth.log`.
- Su `journal.md` documenta cada cuenta creada: a quién pertenece, por qué existe,
  y cuándo deberá cerrarse.
