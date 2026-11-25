# 🔐 A2 · Permisos y propiedad

*Capítulo [02 · Permisos de archivos](../../02_file_permissions.md)*

> [!NOTE]
> **Objetivos**: leer una línea de `ls -l` sin pensar, establecer permisos correctos en octal y simbólico,
> comprender qué significan `r`, `w` y `x` en un directorio, dominar SUID / SGID / sticky bit,
> y reparar una propiedad incorrecta.

---

## Ejercicio 2.1 — Leer

Aquí hay un extracto de `ls -l` tomado de `srv-tickets`:

```text
-rwxr-x---  1 root   ops     2048  /usr/local/bin/backup.sh
-rw-r--r--  1 root   root     742  /etc/ticketflow/app.conf
drwxrws---  4 root   dev     4096  /srv/projet/
-rw-------  1 alice  alice   1675  /home/alice/.ssh/id_ed25519
-rwsr-xr-x  1 root   root   68208  /usr/bin/passwd
drwxrwxrwt  8 root   root    4096  /tmp
lrwxrwxrwx  1 root   root      21  /root/conf -> /srv/ticketflow/conf
```

Para **cada línea**: ¿cuál es el tipo de objeto? ¿Qué pueden hacer el propietario, el grupo, los otros?
¿cuál es la notación **octal** correspondiente? ¿Y para las tres últimas, **qué bit especial** se establece,
y para qué sirve aquí concretamente?

---

## Ejercicio 2.2 — Establecer permisos

Dé la orden `chmod` **octal** para cada una de estas necesidades, luego la versión **simbólica** equivalente:

| Ruta | Necesidad |
|------|-----------|
| `/root/.ssh/id_ed25519` | clave privada: solo el propietario, lectura / escritura |
| `/etc/ticketflow/app.conf` | legible por todos, modificable solo por root |
| `/usr/local/bin/backup.sh` | ejecutable por root y por el grupo `ops`, invisible para otros |
| `/srv/ticketflow/backups` | atravesable y listable por el grupo, prohibido para otros |
| `/var/log/ticketflow` | el servicio escribe en él, los administradores leen, nadie más entra |

Luego **al revés**: traduzca a `rwx` los permisos `640`, `750`, `775`, `1777` y `4755`.

---

## Ejercicio 2.3 — El caso de los directorios

1. En un archivo `r`, `w` y `x` son claros. En un **directorio**, ¿qué permite exactamente cada uno?
2. Cree `/srv/test-droits`, ponga un archivo en él, luego retire `x` del directorio para su usuario de
   prueba. ¿Puede seguir **listando** el directorio? ¿**leyendo** el archivo cuyo nombre conoce? Explique.
3. Repita la prueba retirando `r` pero dejando `x`. ¿Qué puede hacer esta vez?
4. Un usuario puede eliminar un archivo del que **no** es propietario y sobre el que **no tiene ningún**
   derecho de escritura. ¿Cómo es posible? ¿Cómo evitarlo?

---

## Ejercicio 2.4 — Aplicar en masa

1. Aplique `750` recursivamente a `/srv/ticketflow`.
2. Problema: esto también hace los **archivos** ejecutables, lo que no queremos. Escriba las dos órdenes
   que establecen `750` en los **directorios** y `640` en los **archivos**, usando `find`
   *(capítulo [03](../../03_commandes_essentielles.md))*.
3. Repita lo mismo con la notación simbólica `X` mayúscula. ¿Qué hace exactamente?
4. ¿Por qué `chmod -R 777` es casi siempre la respuesta equivocada? Dé dos consecuencias concretas.

---

## Ejercicio 2.5 — Los bits especiales

1. **SGID en un directorio**: haga que todo archivo creado en `/srv/projet` pertenezca
   automáticamente al grupo `dev`, independientemente de su creador. Verifique con dos cuentas diferentes.
2. **Sticky bit**: haga que en este directorio compartido, cada uno solo pueda eliminar **sus propios**
   archivos. Verifique intentando eliminar el archivo de otro.
3. **SUID**: ¿por qué `/usr/bin/passwd` lo necesita? ¿Qué pasaría sin él?
4. Busque en todo el sistema los archivos con el bit **SUID** *(pista: `find / -perm`)* y
   explique por qué esta lista es un **punto de auditoría** de seguridad.
5. Establezca SUID en una copia de `/bin/bash` en `/tmp`, constate el peligro, luego **elimínela
   inmediatamente**. ¿Qué hubiera hecho si hubiera encontrado este archivo en un servidor en producción?

---

## Ejercicio 2.6 — Propietario y grupo

1. Dé `/srv/ticketflow` al usuario `deploy` y al grupo `ops`, recursivamente, en una orden.
2. Cambie **solo el grupo** de `/var/log/ticketflow` a `adm`.
3. Un `git pull` lanzado con `sudo` dejó archivos pertenecientes a root en el *home* de un desarrollador.
   Escriba la orden que repara la propiedad de todo su *home*.
4. Copie un archivo con `cp` y luego con `cp -a`: ¿qué sucede con el propietario, grupo, permisos y fechas?
   ¿Qué opción se usa para una copia de seguridad?
5. ¿Puede un usuario sin privilegios dar uno de sus archivos a otra persona? ¿Por qué?

---

## Ejercicio 2.7 — `umask`

1. Muestre su `umask` actual. ¿Qué permisos obtiene un **archivo** recién creado? ¿Un **directorio**?
2. Verifíquelo creando ambos y leyendo sus permisos.
3. Cambie el `umask` a `027` y repita la prueba. ¿Qué cambia para "los otros"?
4. ¿Dónde debe escribir este valor para que se aplique **a todos los usuarios** en cada conexión?
   *(Ver también el capítulo [10](../../10_shell_config.md).)*
5. ¿Por qué un `umask` de `002` es frecuente en un servidor de trabajo colaborativo y peligroso en otros lugares?

---

## ✅ Verificación

- Traduce `rwxr-x---` ↔ `750` en ambas direcciones, sin tabla de correspondencia.
- `/srv/projet` tiene SGID **y** sticky bit, y lo ha probado con dos cuentas.
- `/srv/ticketflow` tiene directorios en `750` y archivos en `640` — no al revés.
- Sabe producir la lista de binarios SUID de la máquina y decir por qué se supervisa.
- Ninguna copia de shell SUID se queda en `/tmp`.
