# 10 · Configuración del shell (`.bashrc`, `.zshrc`, …)

> [!NOTE]
> **Objetivos del capítulo**
> - Saber qué shell se utiliza y cómo cambiarlo
> - Entender la diferencia entre shell **de inicio de sesión** y shell **interactivo**
> - Saber **qué archivo** modificar, y por qué
> - Crear alias, funciones y variables de entorno persistentes
> - Personalizar el `PATH`, el prompt y el historial
> - Organizar y versionar los archivos de configuración (*dotfiles*)

## Tabla de contenidos

1. [¿Qué shell estoy usando?](#qué-shell-estoy-usando)
2. [Inicio de sesión, interactivo: entender los cuatro casos](#inicio-de-sesión-interactivo-entender-los-cuatro-casos)
3. [¿Qué archivo se carga, y cuándo?](#qué-archivo-se-carga-y-cuándo)
4. [¿Dónde debo poner mi configuración?](#dónde-debo-poner-mi-configuración)
5. [Aplicar las modificaciones](#aplicar-las-modificaciones)
6. [Los alias](#los-alias)
7. [Las funciones](#las-funciones)
8. [Las variables de entorno](#las-variables-de-entorno)
9. [El PATH](#el-path)
10. [Personalizar el prompt](#personalizar-el-prompt)
11. [Dominar el historial](#dominar-el-historial)
12. [Opciones del shell](#opciones-del-shell)
13. [Configuración del sistema y /etc/skel](#configuración-del-sistema-y-etcskel)
14. [Organizar y versionar los dotfiles](#organizar-y-versionar-los-dotfiles)
15. [Trampas clásicas](#trampas-clásicas)
16. [Resumen](#resumen)

---

## ¿Qué shell estoy usando?

El shell es el programa que interpreta lo que escribes en la terminal. Varios coexisten en la misma máquina.

| Shell | Ruta | Observación |
|-------|--------|----------|
| **sh** | `/bin/sh` | El shell POSIX mínimo. En Debian, en realidad es `dash`. |
| **bash** | `/bin/bash` | *Bourne Again Shell* — el estándar de facto en Linux |
| **zsh** | `/usr/bin/zsh` | Compatible con bash, más rico (autocompletado, globbing, temas) |
| **fish** | `/usr/bin/fish` | Muy amigable, pero **no** compatible con POSIX |

Conocer el shell de inicio de sesión:

```bash
echo $SHELL
```

Conocer el shell **realmente en ejecución**:

```bash
ps -p $$
```

Listar los shells instalados:

```bash
cat /etc/shells
```

### Cambiar de shell

```bash
chsh -s /usr/bin/zsh
```

Para otro usuario (como root):

```bash
sudo chsh -s /usr/bin/zsh usuario
```

> [!IMPORTANT]
> El cambio solo toma efecto en la **próxima sesión**. Verifica primero que el shell funcione ejecutándolo simplemente (`zsh`): un shell de inicio de sesión roto puede inutilizar la cuenta.
> El shell de un usuario se almacena en el último campo de su línea en `/etc/passwd`.

---

## Inicio de sesión, interactivo: entender los cuatro casos

Este es **el** punto que explica el 90% de los "por qué mi alias no funciona". Un shell se caracteriza por dos propiedades independientes.

| Tipo de shell | Definición | Ejemplo concreto |
|---------------|------------|-----------------|
| **Inicio de sesión + interactivo** | Abierto después de autenticarse | Conexión SSH, `su - usuario`, consola TTY |
| **Sin inicio de sesión + interactivo** | Abierto desde una sesión ya autenticada | Una nueva pestaña de terminal gráfica, `bash`, un panel de tmux |
| **Sin inicio de sesión + no interactivo** | Ejecuta un script, sin humano presente | `./mi_script.sh`, una tarea cron |
| **Inicio de sesión + no interactivo** | Raro | `ssh servidor 'comando'`, algunos scripts de despliegue |

Verificar en qué caso estás:

```bash
# ¿Shell interactivo? (la salida contiene una "i")
echo $-

# ¿Shell de inicio de sesión?
shopt -q login_shell && echo "inicio" || echo "sin-inicio"
```

---

## ¿Qué archivo se carga, y cuándo?

### Bash

```text
┌─ SHELL DE INICIO ─────────────────┐   ┌─ SHELL INTERACTIVO (sin inicio) ──┐
│ 1. /etc/profile                   │   │ 1. /etc/bash.bashrc               │
│ 2. /etc/profile.d/*.sh            │   │ 2. ~/.bashrc                      │
│ 3. el PRIMERO encontrado entre:   │   └────────────────────────────────────┘
│      ~/.bash_profile              │
│      ~/.bash_login                │   ┌─ SCRIPT (no interactivo) ─────────┐
│      ~/.profile                   │   │ Ningún archivo cargado            │
│                                   │   │ (a menos que $BASH_ENV esté def.) │
│ ...y al desconectarse:            │   └────────────────────────────────────┘
│      ~/.bash_logout               │
└─────────────────────────────────────┘
```

| Archivo | Alcance | Cargado para |
|---------|--------|-------------|
| `/etc/profile` | Todos los usuarios | Shells de inicio de sesión |
| `/etc/profile.d/*.sh` | Todos los usuarios | Shells de inicio — **el lugar propicio** para añadir config del sistema |
| `/etc/bash.bashrc` | Todos los usuarios | Shells interactivos |
| `~/.bash_profile` | Un usuario | Shells de inicio de sesión |
| `~/.profile` | Un usuario | Shells de inicio, **si** `.bash_profile` no existe |
| `~/.bashrc` | Un usuario | Shells interactivos sin inicio de sesión |
| `~/.bash_logout` | Un usuario | Al final de un shell de inicio de sesión |

> [!CAUTION]
> **Bash NO lee `~/.bashrc` en un shell de inicio de sesión.**
> Por eso la mayoría de `~/.bash_profile` contienen este puente:
>
> ```bash
> # Cargar ~/.bashrc si es interactivo
> if [ -n "$BASH_VERSION" ] && [ -f "$HOME/.bashrc" ]; then
>     . "$HOME/.bashrc"
> fi
> ```
>
> Sin este bloque, tus alias funcionan en un terminal local pero desaparecen en SSH.

### Zsh

Zsh es más regular: el nombre del archivo indica cuándo se lee.

| Orden | Archivo | Cargado para |
|-------|---------|-------------|
| 1 | `/etc/zshenv` luego `~/.zshenv` | **Todos** los shells, incluyendo scripts |
| 2 | `/etc/zprofile` luego `~/.zprofile` | Shells de inicio de sesión |
| 3 | `/etc/zshrc` luego `~/.zshrc` | Shells interactivos |
| 4 | `/etc/zlogin` luego `~/.zlogin` | Shells de inicio (después de `.zshrc`) |
| 5 | `~/.zlogout` | Al desconectarse |

> [!TIP]
> **Correspondencia mental bash → zsh:**
> `~/.bashrc` → `~/.zshrc` · `~/.bash_profile` → `~/.zprofile` · *(sin equivalente en bash)* → `~/.zshenv`

---

## ¿Dónde debo poner mi configuración?

Esta es la pregunta práctica. La tabla siguiente responde en casi todos los casos.

| Lo que quiero configurar | Bash | Zsh |
|---------------------------|------|-----|
| Un **alias** | `~/.bashrc` | `~/.zshrc` |
| Una **función** | `~/.bashrc` | `~/.zshrc` |
| El **prompt** (`PS1`) | `~/.bashrc` | `~/.zshrc` |
| **Colores**, autocompletado | `~/.bashrc` | `~/.zshrc` |
| Una **variable de entorno** (`EDITOR`, `LANG`) | `~/.bash_profile` o `~/.profile` | `~/.zprofile` |
| El **`PATH`** | `~/.bash_profile` o `~/.profile` | `~/.zprofile` (o `~/.zshenv`) |
| Algo que los **scripts** deben ver | `/etc/profile.d/` | `~/.zshenv` |
| Config **para todos los usuarios** | `/etc/profile.d/mi_archivo.sh` | igual |

**Regla simple de recordar:**

> Lo que es **heredado** por procesos hijo (variables exportadas, `PATH`) → archivo de **profile**.
> Lo que concierne **la comodidad de escritura** (alias, prompt, autocompletado) → archivo **rc**.

---

## Aplicar las modificaciones

Después de editar un archivo de configuración, **no** se recarga automáticamente. Tres opciones:

```bash
# 1. Recargar en el shell actual (lo más común)
source ~/.bashrc
```

```bash
# 2. Sintaxis equivalente, más portable
. ~/.bashrc
```

```bash
# 3. Reemplazar el shell actual por uno nuevo
exec bash -l
```

> [!IMPORTANT]
> `source archivo` ejecuta el archivo **en el shell actual** — eso permite que alias y variables persistan.
> `./archivo` lo ejecuta en un **subproceso**: el shell actual no conserva nada. Es exactamente la distinción vista en el [capítulo 06.6](06_shellscript/06_fonctions/fonctions.md#importar-funciones-de-otro-script).

> [!TIP]
> Haz una copia de seguridad antes de modificar un archivo de config:
> ```bash
> cp ~/.bashrc ~/.bashrc.bak
> ```

---

## Los alias

Un alias es un atajo de comando.

```bash
# Sintaxis: alias nombre='comando'
alias ll='ls -alh'
alias la='ls -A'
alias ..='cd ..'
alias ...='cd ../..'

# Asegurar comandos destructivos (pide confirmación)
alias rm='rm -i'
alias cp='cp -i'
alias mv='mv -i'

# Atajos de administración
alias ports='ss -tulpn'
alias myip='ip -4 addr show scope global | grep inet'
alias update='sudo apt update && sudo apt upgrade'
alias services='systemctl list-units --type=service --state=running'
```

Gestionar alias:

```bash
alias                  # listar todos los alias definidos
alias ll               # ver la definición de un alias
unalias ll             # eliminar un alias (para esta sesión)
\ls                    # ejecutar el comando real ignorando el alias
command ls             # lo mismo
```

> [!WARNING]
> **Los alias solo funcionan en shells interactivos.** **No** están disponibles en tus scripts, ni en cron.
> Nunca confíes en un alias dentro de un script: escribe el comando completo.
>
> Cuidado también con el efecto adverso de `alias rm='rm -i'`: adquieres la costumbre de una confirmación que no existirá en un servidor donde no esté definido el alias.

---

## Las funciones

Cuando un alias no es suficiente (necesitas argumentos, lógica), escribes una función en tu `.bashrc` / `.zshrc`.

```bash
# Crear una carpeta e ir a ella inmediatamente
mkcd() {
  mkdir -p "$1" && cd "$1"
}

# Extraer cualquier archivo
extract() {
  case "$1" in
    *.tar.gz|*.tgz)  tar -xzvf "$1" ;;
    *.tar.bz2)       tar -xjvf "$1" ;;
    *.tar)           tar -xvf  "$1" ;;
    *.zip)           unzip     "$1" ;;
    *.gz)            gunzip    "$1" ;;
    *) echo "Formato no soportado: $1" >&2; return 1 ;;
  esac
}

# Respaldar un archivo antes de modificarlo
bak() {
  cp -- "$1" "$1.$(date +%Y%m%d-%H%M%S).bak"
}
```

> [!TIP]
> Diferencia práctica: un **alias** sustituye texto al inicio del comando, una **función** puede recibir y manipular argumentos. En cuanto necesites `$1`, es una función.

---

## Las variables de entorno

```bash
# Variable de shell: visible solo en el shell actual
MI_VAR="valor"

# Variable de entorno: heredada por todos los procesos hijo
export MI_VAR="valor"

# Las dos en una línea
export EDITOR=vim
```

Variables más útiles para definir en tu profile:

```bash
export EDITOR=vim               # editor por defecto (git, crontab -e, visudo)
export VISUAL=vim
export PAGER=less
export LANG=es_ES.UTF-8         # idioma y codificación
export LESS='-R'                # preservar colores en less
```

Inspeccionar el entorno:

```bash
printenv                # todas las variables de entorno
printenv PATH           # una sola
env                     # equivalente
set                     # variables de entorno Y variables de shell
unset MI_VAR            # eliminar una variable
```

Algunas variables estándar:

| Variable | Contenido |
|----------|---------|
| `$HOME` | Carpeta personal del usuario |
| `$USER` | Nombre del usuario actual |
| `$PWD` | Carpeta actual |
| `$SHELL` | Shell de inicio de sesión |
| `$PATH` | Dónde buscar ejecutables |
| `$?` | Código de retorno del último comando |
| `$$` | PID del shell actual |

> [!CAUTION]
> **Nunca** almacenes secretos (contraseña, token de API) en un `.bashrc` versionado. Las variables de entorno se heredan por todos los procesos hijo y frecuentemente aparecen en logs de debug.
> Usa un archivo separado, no versionado y con `chmod 600`:
>
> ```bash
> [ -f ~/.secrets ] && source ~/.secrets
> ```

---

## El PATH

El `PATH` es la lista de carpetas donde el shell busca ejecutables, separadas por `:`.

```bash
echo $PATH
```

```text
/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
```

Añadir una carpeta — típicamente la de tus propios scripts:

```bash
# A colocar en ~/.bash_profile, ~/.profile o ~/.zprofile
export PATH="$HOME/bin:$PATH"
```

| Forma | Efecto |
|-------|-------|
| `export PATH="$HOME/bin:$PATH"` | Mi carpeta es **prioritaria** sobre el sistema |
| `export PATH="$PATH:$HOME/bin"` | El sistema es prioritario sobre mi carpeta |

> [!WARNING]
> **Nunca olvides reinjectar `$PATH`.** Escribir `export PATH="$HOME/bin"` reemplaza todo el `PATH`: ningún comando del sistema se encuentra.
>
> **Nunca añadas `.` (la carpeta actual) al `PATH`**: un archivo malicioso llamado `ls` depositado en una carpeta compartida sería ejecutado en tu lugar.

Saber qué ejecutable será realmente lanzado:

```bash
which ls
type -a ls
command -v ls
```

---

## Personalizar el prompt

El prompt se define con la variable `PS1` en bash, `PROMPT` en zsh.

### Bash

```bash
# Prompt por defecto en Debian
PS1='\u@\h:\w\$ '
```

| Secuencia | Muestra |
|----------|---------|
| `\u` | El nombre de usuario |
| `\h` | El nombre de la máquina (corto) — `\H` para el FQDN |
| `\w` | La ruta completa — `\W` solo la carpeta |
| `\$` | `#` si eres root, `$` si no |
| `\t` | La hora (HH:MM:SS) |
| `\n` | Un salto de línea |

Con colores:

```bash
PS1='\[\e[32m\]\u@\h\[\e[0m\]:\[\e[34m\]\w\[\e[0m\]\$ '
```

> [!IMPORTANT]
> Los códigos de color deben estar rodeados por `\[` y `\]`. Estos marcadores le indican a bash que estos caracteres **no** ocupan espacio en pantalla. Sin ellos, el shell calcula mal la longitud del prompt y la línea se muestra de lado cuando editas un comando largo.

Un prompt útil en administración — rojo cuando eres root:

```bash
if [ "$(id -u)" -eq 0 ]; then
  PS1='\[\e[1;31m\]\u@\h\[\e[0m\]:\w# '     # rojo = atención, soy root
else
  PS1='\[\e[1;32m\]\u@\h\[\e[0m\]:\w\$ '    # verde = usuario normal
fi
```

### Zsh

```bash
PROMPT='%n@%m %~ %# '
```

| Secuencia | Muestra |
|----------|---------|
| `%n` | Usuario |
| `%m` | Máquina |
| `%~` | Ruta (con `~` para home) |
| `%#` | `#` si eres root, `%` si no |
| `%F{green}…%f` | Color |

> [!TIP]
> Configurar manualmente un prompt rico (rama git, duración de ejecución, estado del último comando) es tedioso. Hay herramientas que lo hacen por ti: ver el [capítulo 11 · Configuración avanzada](11_advanced_config_es.md).

---

## Dominar el historial

El historial es la herramienta más rentable del shell — siempre que lo configures bien.

### Bash

```bash
# A colocar en ~/.bashrc

HISTSIZE=10000                    # comandos guardados en memoria
HISTFILESIZE=20000                # comandos guardados en archivo
HISTCONTROL=ignoreboth:erasedups  # ignora duplicados y líneas que comienzan con espacio
HISTIGNORE="ls:cd:pwd:exit:clear:history"
HISTTIMEFORMAT="%F %T "           # marcar con fecha/hora cada comando

shopt -s histappend               # añadir al historial en lugar de sobrescribir
```

### Zsh

```bash
# A colocar en ~/.zshrc

HISTFILE=~/.zsh_history
HISTSIZE=10000
SAVEHIST=20000

setopt HIST_IGNORE_ALL_DUPS       # sin duplicados
setopt HIST_IGNORE_SPACE          # ignora líneas que comienzan con espacio
setopt SHARE_HISTORY              # historial compartido entre terminales abiertos
setopt EXTENDED_HISTORY           # marca con fecha/hora
```

### Usar el historial

| Atajo / comando | Efecto |
|----------------------|-------|
| `CTRL + R` | Búsqueda incremental en el historial |
| `↑` / `↓` | Comando anterior / siguiente |
| `history` | Mostrar historial numerado |
| `!42` | Repetir el comando nº 42 |
| `!!` | Repetir el último comando — `sudo !!` |
| `!$` | Último argumento del comando anterior |
| `history -c` | Vaciar el historial |

> [!TIP]
> `HISTCONTROL=ignorespace` (incluido en `ignoreboth`) es muy práctico: un comando precedido de un **espacio** no se registra en el historial. Ideal para comandos que contienen contraseñas.

---

## Opciones del shell

### Bash — `shopt`

```bash
shopt -s autocd          # escribir "Documentos" equivale a "cd Documentos"
shopt -s cdspell         # corrige errores tipográficos en los cd
shopt -s checkwinsize    # recalcula tamaño de terminal después de cada comando
shopt -s globstar        # activa ** para búsqueda recursiva: ls **/*.md
shopt -s nocaseglob      # globbing insensible a mayúsculas
```

Listar todas las opciones: `shopt`

### Zsh — `setopt`

```bash
setopt AUTO_CD
setopt CORRECT           # propone corrección en caso de error tipográfico
setopt EXTENDED_GLOB
setopt NO_BEEP
```

### El autocompletado

En bash, el autocompletado avanzado (opciones de comandos, nombres de servicios, ramas git…) viene de un paquete a instalar:

```bash
sudo apt install bash-completion
```

Luego, en `~/.bashrc`:

```bash
if [ -f /usr/share/bash-completion/bash_completion ]; then
  . /usr/share/bash-completion/bash_completion
fi
```

En zsh:

```bash
autoload -Uz compinit && compinit
```

---

## Configuración del sistema y `/etc/skel`

### Aplicar configuración a todos los usuarios

**No** modifiques `/etc/profile` directamente: coloca un archivo en `/etc/profile.d/`.

```bash
sudo tee /etc/profile.d/empresa.sh > /dev/null <<'EOF'
export EDITOR=vim
export TZ="America/Mexico_City"
alias ll='ls -alh'
EOF
```

Todo archivo `.sh` en esta carpeta se carga para cada shell de inicio. Es limpio, reversible, y sobrevive actualizaciones del sistema.

### `/etc/skel` — el esqueleto de nuevas cuentas

Cuando creas un usuario con `useradd -m` (o `adduser`), el contenido de `/etc/skel` se **copia** en su *directorio home*.

```bash
ls -la /etc/skel
```

```text
.bash_logout
.bashrc
.profile
```

Aquí es donde colocas la configuración por defecto que deben recibir nuevas cuentas:

```bash
sudo cp mi_bashrc_tipo /etc/skel/.bashrc
```

> [!NOTE]
> `/etc/skel` solo afecta cuentas **creadas después** de la modificación. Usuarios existentes guardan sus archivos.

---

## Organizar y versionar los dotfiles

Un `.bashrc` de 400 líneas rápidamente se vuelve inmanejable. La mejor práctica es dividirlo:

```bash
# Al final de ~/.bashrc
for f in "$HOME"/.bashrc.d/*.sh; do
  [ -r "$f" ] && . "$f"
done
unset f
```

```text
~/.bashrc.d/
├── 10-aliases.sh
├── 20-functions.sh
├── 30-prompt.sh
└── 40-docker.sh
```

Y lo más importante: **versionan los dotfiles**.

```bash
mkdir ~/dotfiles && cd ~/dotfiles
git init
mv ~/.bashrc ~/dotfiles/bashrc
ln -s ~/dotfiles/bashrc ~/.bashrc
git add . && git commit -m "Dotfiles iniciales"
```

Los archivos en tu home ahora son solo enlaces simbólicos al repositorio: una única fuente de verdad, reutilizable en todas tus máquinas. Herramientas como **GNU stow** o **chezmoi** automatizan esta configuración.

---

## Trampas clásicas

> [!CAUTION]
> **1. Un comando que imprime texto en `.bashrc` rompe `scp` y `sftp`.**
> Estos tools abren un shell no interactivo e interpretan cualquier salida inesperada como protocolo inválido. Siempre protege este tipo de líneas:
>
> ```bash
> # Al inicio de .bashrc
> case $- in
>   *i*) ;;       # interactivo: continúa
>     *) return;; # no interactivo: sale inmediatamente
> esac
> ```

> [!CAUTION]
> **2. Un error de sintaxis en `.bashrc` rompe todos tus terminales nuevos.**
> Prueba antes de desconectarte, desde una **segunda terminal**:
>
> ```bash
> bash -n ~/.bashrc     # verifica sintaxis sin ejecutar
> source ~/.bashrc      # prueba realmente
> ```
>
> Si ya estás atrapado: `ssh usuario@servidor 'bash --norc -i'`, o `bash --norc` en consola.

> [!CAUTION]
> **3. `export PATH="/mi/carpeta"` sin `:$PATH`** — ningún comando funciona.
> Solución de emergencia en el shell actual:
>
> ```bash
> export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
> ```

> [!CAUTION]
> **4. Creer que los alias están disponibles en scripts y crons.** No lo están. Cron no ejecuta `.bashrc` ni `.profile`: usa rutas absolutas (ver [capítulo 04](04_instalacion_y_servicios_es.md#cron)).

---

## Resumen

| Archivo | Shell | Cuándo |
|---------|-------|-------|
| `~/.bashrc` | bash | Cada shell interactivo → alias, funciones, prompt |
| `~/.bash_profile` / `~/.profile` | bash | Inicio → variables de entorno, `PATH` |
| `~/.bash_logout` | bash | Desconexión |
| `~/.zshrc` | zsh | Cada shell interactivo |
| `~/.zprofile` | zsh | Inicio de sesión |
| `~/.zshenv` | zsh | **Todos** los shells, scripts incluidos |
| `/etc/profile.d/*.sh` | todos | Inicio, para todos los usuarios |
| `/etc/skel/` | — | Copiado en home de nuevas cuentas |

| Comando | Rol |
|----------|------|
| `echo $SHELL` / `ps -p $$` | ¿Qué shell? |
| `chsh -s /usr/bin/zsh` | Cambiar shell |
| `source ~/.bashrc` | Recargar configuración |
| `alias` / `unalias` | Gestionar alias |
| `export VAR=valor` | Definir variable de entorno |
| `printenv` | Listar entorno |
| `shopt -s` / `setopt` | Activar opción del shell |
| `bash -n ~/.bashrc` | Verificar sintaxis antes de romper sesión |

---

⬅️ [Anterior: 09 · tmux](09_tmux_es.md) · 🏠 [Tabla de contenidos](README_es.md) · [Siguiente: 11 · Configuración avanzada ➡️](11_advanced_config_es.md)
