# 09 · tmux (Terminal Multiplexer)

Es una herramienta muy interesante cuando se hace administración de sistemas de forma remota: permite gestionar **sesiones** en el servidor y **recuperarlas** en caso de desconexión.

> [!NOTE]
> **Objetivos del capítulo**
> - Crear, salir y rejoin a una sesión tmux
> - Dividir una terminal en paneles y gestionar varias ventanas
> - Navegar y buscar en el historial de la terminal

> [!TIP]
> **El concepto en una frase:** tu trabajo se ejecuta en el **servidor**, no en tu terminal. Si tu conexión SSH se corta, la sesión tmux sigue ejecutándose y la encuentras intacta cuando te reconectas.
>
> Jerarquía: una **sesión** contiene **ventanas**, que contienen **paneles** (*panes*).

## Tabla de contenidos

1. [Gestión de sesiones](#gestión-de-sesiones)
2. [Dividir la terminal (paneles)](#dividir-la-terminal-paneles)
3. [Las ventanas](#las-ventanas)
4. [Navegación y búsqueda](#navegación-y-búsqueda)
5. [Oh my tmux](#oh-my-tmux)
6. [Referencia rápida](#referencia-rápida)

---

## El prefijo

Casi todos los atajos de tmux comienzan con la combinación `CTRL + b`: se llama el **prefijo**. Se presiona `CTRL + b`, se **suelta**, luego se presiona la tecla de la acción.

En todo este capítulo, esta secuencia se abrevia como:

```text
CTRL + b  =>  tecla
```

---

## Gestión de sesiones

### 1. Instalación

```bash
sudo apt install tmux
```

### 2. Iniciar una sesión

```bash
tmux
```

```bash
tmux new -s nombre-sesion
```

> [!TIP]
> Siempre nombra tus sesiones (`-s`). Encontrar `nombre-sesion` es mucho más simple que adivinar a qué corresponde la sesión `0`.

### 3. Salir de una sesión manteniéndola activa (*detach*)

```text
CTRL + b  =>  d
```

### 4. Listar las sesiones activas

```bash
tmux ls
```

o, desde tmux:

```text
CTRL + b  =>  s
```

### 5. Rejoin a una sesión (*attach*)

```bash
tmux a -t nombre-sesion
```

```bash
tmux attach -t nombre-sesion
```

### 6. Destruir una sesión

```bash
tmux kill-session -t nombre-sesion
```

```bash
tmux kill -t nombre-sesion
```

```bash
tmux k -t nombre-sesion
```

### 7. Renombrar una sesión

```text
CTRL + b  =>  $
```

### 8. Cambiar el directorio de trabajo por defecto

Cuando se abre una nueva terminal en una sesión, se posiciona en un CWD (*current working directory*) por defecto.

```text
CTRL + b  =>  :
```

luego se escribe el comando:

```text
attach -c /nuevo/directorio
```

La idea es posicionarse automáticamente en el directorio correcto al abrir nuevas terminales en una sesión tmux — por ejemplo el directorio de trabajo de un proyecto.

---

## Dividir la terminal (paneles)

Puedo dividir la terminal en varias partes:

| Atajo | Acción |
|-----------|--------|
| `CTRL + b  =>  %` | Split **vertical** (lado a lado) |
| `CTRL + b  =>  "` | Split **horizontal** (uno encima del otro) |

> [!NOTE]
> Los nombres son confusos: `%` divide la pantalla con una barra vertical, `"` con una barra horizontal.

### Desplazarse entre paneles

| Atajo | Acción |
|-----------|--------|
| `CTRL + b  =>  ←↑↓→` | Ir al panel en esa dirección |
| `CTRL + b  =>  o` | Circular en el sentido de las agujas del reloj |
| `CTRL + b  =>  {` | Circular en el sentido inverso |
| `CTRL + b  =>  ;` | Cambiar al último panel utilizado |
| `CTRL + b  =>  z` | Ampliar / reducir el panel actual (pantalla completa) |
| `CTRL + b  =>  x` | Cerrar el panel actual |

---

## Las ventanas

| Atajo | Acción |
|-----------|--------|
| `CTRL + b  =>  c` | **C**rear una ventana |
| `CTRL + b  =>  ,` | Renombrar la ventana actual |
| `CTRL + b  =>  !` | Extraer el panel actual en su propia ventana |
| `CTRL + b  =>  &` | Cerrar la ventana activa |
| `CTRL + b  =>  n` | Ventana siguie**n**te |
| `CTRL + b  =>  p` | Ventana **p**recente |
| `CTRL + b  =>  w` | Listar las ventanas |
| `CTRL + b  =>  0-9` | Ir directamente a la ventana nº X |

---

## Navegación y búsqueda

### Activar el desplazamiento (modo copia)

```text
CTRL + b  =>  [
```

Salir del modo desplazamiento:

```text
q
```

### Buscar en el historial de la terminal

Primero hay que estar en modo desplazamiento:

```text
CTRL + b  =>  [
```

Luego:

```text
CTRL + r
```

y escribir lo que se busca.

> [!TIP]
> En modo copia, también se puede seleccionar texto (`ESPACIO` para comenzar, `INTRO` para copiar) y pegarlo con `CTRL + b  =>  ]`.

---

## Oh my tmux

Con [oh-my-tmux](https://github.com/gpakosz/.tmux), se puede usar el ratón:

```text
CTRL + b  =>  m
```

`oh-my-tmux` mejora tmux y añade soporte de ratón, una barra de estado legible y muchas configuraciones por defecto sensatas. Su configuración se detalla en el [capítulo 11 · Configuración avanzada](11_advanced_config_es.md#oh-my-tmux).

---

## Referencia rápida

### En línea de comandos

| Comando | Acción |
|----------|--------|
| `tmux` | Nueva sesión anónima |
| `tmux new -s nombre` | Nueva sesión nombrada |
| `tmux ls` | Listar sesiones |
| `tmux a -t nombre` | Rejoin a una sesión |
| `tmux kill-session -t nombre` | Destruir una sesión |

### En tmux (prefijo `CTRL + b`)

| Tecla | Acción | | Tecla | Acción |
|--------|--------|---|--------|--------|
| `d` | Desconectar | | `c` | Nueva ventana |
| `s` | Listar sesiones | | `,` | Renombrar la ventana |
| `$` | Renombrar la sesión | | `n` / `p` | Ventana siguiente / anterior |
| `%` | Split vertical | | `w` | Listar ventanas |
| `"` | Split horizontal | | `&` | Cerrar la ventana |
| `←↑↓→` | Cambiar de panel | | `[` | Modo scroll / copia |
| `z` | Ampliar un panel | | `?` | **Mostrar todos los atajos** |

---

⬅️ [Anterior: 08 · Docker para el administrador](08_docker_administration_es.md) · 🏠 [Tabla de contenidos](README_es.md) · [Siguiente: 10 · Configuración del shell ➡️](10_shell_config_es.md)
