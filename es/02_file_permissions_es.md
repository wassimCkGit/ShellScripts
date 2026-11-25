# 02 · Los permisos de archivos (POSIX)

> [!NOTE]
> **Objetivos del capítulo**
> - Leer y decodificar una línea de permisos mostrada por `ls -l`
> - Modificar permisos en notación octal y notación simbólica
> - Entender qué significan `r`, `w` y `x` en una **carpeta**
> - Conocer los permisos especiales (SUID, SGID, sticky bit)
> - Cambiar el propietario y el grupo de un archivo

## Tabla de contenidos

1. [Leer los permisos POSIX](#leer-los-permisos-posix)
2. [El caso especial de las carpetas](#el-caso-especial-de-las-carpetas)
3. [Modificar los permisos con chmod](#modificar-los-permisos-con-chmod)
4. [Aplicar en masa](#aplicar-en-masa)
5. [Permisos especiales](#permisos-especiales)
6. [Tipos de archivos](#tipos-de-archivos)
7. [Cambiar el propietario y el grupo](#cambiar-el-propietario-y-el-grupo)
8. [Resumen](#resumen)

---

## Leer los permisos POSIX

```text
-rwxr-xr-- 1 usuario grupo 219K May 15  2020 script.sh
```

Descompongamos la primera columna:

```text
 -    rwx      r-x      r--
 │     │        │        │
tipo  usuario  grupo    otros
```

| Letra | Significado | En un archivo |
|--------|---------------|----------------|
| `r` | **read** (lectura) | Leer el contenido |
| `w` | **write** (escritura) | Modificar el contenido |
| `x` | **execute** (ejecución) | Ejecutar el archivo (script, binario) |
| `-` | — | Derecho ausente |

Los tres grupos de tres caracteres, en orden:

| Grupo | Concerne | En el ejemplo |
|--------|----------|----------------|
| 1er (`rwx`) | El **propietario** del archivo | `usuario` |
| 2o (`r-x`)  | El **grupo** propietario | `grupo` |
| 3er (`r--`)  | Todos los **otros** usuarios | todo el mundo |

Entonces, en este caso:

- el usuario `usuario` puede **leer / escribir / ejecutar** el archivo;
- los miembros de `grupo` pueden **leer y ejecutar**;
- los otros solo pueden **leer** el archivo.

---

## El caso especial de las carpetas

```text
drwxr-xr-- 1 usuario grupo 219K May 15  2020 carpeta
```

La `d` al inicio indica que se trata de un **d**irectory (carpeta). En una carpeta, los tres derechos tienen un significado completamente diferente al de un archivo:

| Derecho | En una carpeta |
|-------|----------------|
| `r` | Listar el contenido (`ls`) |
| `w` | Añadir, renombrar y eliminar archivos / subcarpetas **dentro** de la carpeta |
| `x` | Atravesar la carpeta, es decir, entrar en ella (`cd`) y acceder a su contenido |

> [!IMPORTANT]
> Sin el derecho `x` en una carpeta, el derecho `r` es casi inútil: se pueden ver los nombres de los archivos, pero no abrirlos.
> Por el contrario, `w` en una carpeta permite **eliminar** un archivo incluso sin ningún derecho sobre ese archivo — es la carpeta la que decide.

---

## Modificar los permisos con chmod

### Notación octal

Los permisos se codifican en octal, un dígito del 0 al 7 por grupo:

| Valor | Derecho |
|--------|-------|
| `4` | lectura (`r`) |
| `2` | escritura (`w`) |
| `1` | ejecución (`x`) |

Se suman estos dígitos para componer un derecho:

| Dígito | Cálculo | Resultado |
|---------|--------|----------|
| `7` | 4+2+1 | `rwx` |
| `6` | 4+2   | `rw-` |
| `5` | 4+1   | `r-x` |
| `4` | 4     | `r--` |
| `3` | 2+1   | `-wx` |
| `2` | 2     | `-w-` |
| `1` | 1     | `--x` |
| `0` | —     | `---` |

### Ejemplo 1 — bloquear un script solo para su propietario

```bash
chmod 700 script.sh
```

Equivalente en notación simbólica:

```bash
chmod u=rwx,g=,o= script.sh
```

| | Permisos |
|---|---|
| **Antes** | `-rwxr-xr-- 1 usuario grupo 219K May 15  2020 script.sh` |
| **Después** | `-rwx------ 1 usuario grupo 219K May 15  2020 script.sh` |

Caso especial:
Añade lectura al propietario
```bash
chmod u+r script.sh
```

Retira escritura al grupo
```bash
chmod g-w script.sh
```

### Ejemplo 2 — un archivo compartido con su grupo

```bash
chmod 664 script.sh
```

Equivalente en notación simbólica:

```bash
chmod u=rw,g=rw,o=r script.sh
```

| | Permisos |
|---|---|
| **Antes** | `-rwx------ 1 usuario grupo 219K May 15  2020 script.sh` |
| **Después** | `-rw-rw-r-- 1 usuario grupo 219K May 15  2020 script.sh` |

> [!TIP]
> La notación simbólica también acepta `+` y `-` para **añadir** o **retirar** un derecho sin tocar el resto:
>
> | Comando | Efecto |
> |----------|-------|
> | `chmod +x script.sh`   | Hace el archivo ejecutable para todos |
> | `chmod u+x script.sh`  | Ejecutable solo para el propietario |
> | `chmod o-r archivo`    | Retira la lectura a los otros |
> | `chmod a=r archivo`    | Solo lectura para todos (`a` = *all*) |

---

## Aplicar en masa

Modificar varios archivos de una vez:

```bash
chmod 700 *.sh
```

Modificar una carpeta y aplicar los mismos permisos a todo su contenido (`-R` = recursivo):

```bash
chmod -R 700 carpeta/
```

> [!WARNING]
> `chmod -R 700` también aplica el derecho `x` a los **archivos** contenidos en la carpeta, lo que rara vez es deseado.
> La mejor práctica es separar los dos casos:
>
> ```bash
> find carpeta/ -type d -exec chmod 700 {} +   # carpetas: necesitan x para ser atravesadas
> find carpeta/ -type f -exec chmod 600 {} +   # archivos: sin x
> ```

A continuación, pongo los permisos 700 a todos los hijos contenidos en carpeta/ pero no a la carpeta.
```bash
chmod -R 700 carpeta/*
```

---

## Permisos especiales

### SUID y SGID

```text
-rwsr-xr-x  →  la x del propietario es reemplazada por una s
```

El bit **SUID** (en el usuario) o **SGID** (en el grupo) permite ejecutar el archivo **con los derechos del propietario / del grupo propietario** del archivo, y no con los del usuario que lo ejecuta.

Añadir el bit:

```bash
chmod u+s archivo.sh
```

```bash
chmod g+s archivo.sh
```

Retirarlo:

```bash
chmod u-s archivo.sh
```

```bash
chmod g-s archivo.sh
```

### SGID en una carpeta

En una **carpeta**, `g+s` tiene otro efecto, muy útil para un espacio de trabajo compartido: todo archivo creado dentro hereda automáticamente el **grupo de la carpeta** en lugar del grupo de su creador.

```bash
chmod g+s /srv/proyecto_compartido
```

### Sticky bit

```bash
chmod +t /tmp
```

En una carpeta con permisos de escritura para todos, el *sticky bit* (`drwxrwxrwt`) hace que **solo el propietario de un archivo pueda eliminarlo**. Esto es lo que protege `/tmp`.

---

## Tipos de archivos

El primer carácter de la línea indica el tipo del elemento:

```bash
ls -l /
```

```text
lrwxrwxrwx   1 root root    7 Dec  2  2024 bin -> usr/bin
```

Aquí `/bin` es un **enlace simbólico** que apunta a `/usr/bin`.

| Carácter | Tipo |
|-----------|------|
| `-` | Archivo ordinario |
| `d` | Carpeta (*directory*) |
| `l` | Enlace simbólico |
| `c` | Dispositivo en modo carácter (ej. `/dev/null`) |
| `b` | Dispositivo en modo bloque (ej. `/dev/sda`) |
| `s` | Socket |
| `p` | Tubería nombrada (*named pipe*) |

---

## Cambiar el propietario y el grupo

### Cambiar el propietario

```bash
chown nuevoUsuario archivo
```

| | Resultado |
|---|---|
| **Antes** | `-rw-rw-r-- 1 usuario    grupo 219K May 15  2020 archivo` |
| **Después** | `-rw-rw-r-- 1 nuevoUsuario grupo 219K May 15  2020 archivo` |

```bash
chown nuevoUsuario:nuevoGrupo archivo
```

```bash
chown -R nuevoUsuario:nuevoGrupo carpeta/
```

### Cambiar el grupo

```bash
chgrp nuevoGrupo archivo
```

| | Resultado |
|---|---|
| **Antes** | `-rw-rw-r-- 1 usuario grupo 219K May 15  2020 archivo` |
| **Después** | `-rw-rw-r-- 1 usuario nuevoGrupo  219K May 15  2020 archivo` |

> [!TIP]
> `chown` puede hacer los dos de una vez, y también acepta `-R`:
>
> ```bash
> chown nuevoUsuario:nuevoGrupo archivo
> chown -R www-data:www-data /var/www
> ```
>
> Cambiar el propietario de un archivo requiere derechos root.

---

## Resumen

| Comando | Rol |
|----------|------|
| `ls -l` | Mostrar los permisos |
| `chmod 750 f` | Definir los permisos (octal) |
| `chmod u+x f` | Añadir / retirar un derecho (simbólico) |
| `chmod -R` | Aplicar recursivamente |
| `chmod u+s` / `g+s` / `+t` | SUID / SGID / sticky bit |
| `chown user f` | Cambiar el propietario |
| `chgrp group f` | Cambiar el grupo |
| `chown user:group f` | Cambiar los dos |

**Valores que aparecen más frecuentemente:**

| Octal | Simbólico | Uso típico |
|-------|------------|---------------|
| `600` | `rw-------` | Archivo privado (clave SSH, secreto) |
| `644` | `rw-r--r--` | Archivo de config legible por todos |
| `700` | `rwx------` | Script o carpeta privada |
| `750` | `rwxr-x---` | Script compartido con su grupo |
| `755` | `rwxr-xr-x` | Binario o carpeta pública |

---

⬅️ [Anterior: 01 · Los fundamentos](01_base_es.md) · 🏠 [Tabla de contenidos](README_es.md) · [Siguiente: 02.1 · Usuarios ➡️](02.1_user_management_es.md)
