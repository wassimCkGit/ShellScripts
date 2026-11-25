# 11 · Configuración avanzada (`oh-my-*`, temas y herramientas)

> [!NOTE]
> **Objetivos del capítulo**
> - Instalar y configurar **Oh My Zsh**, sus temas y plugins
> - Conocer las alternativas: **Oh My Bash**, **Starship**
> - Configurar **Oh My Tmux**
> - Instalar las fuentes necesarias (**Nerd Fonts**)
> - Reemplazar herramientas históricas por equivalentes modernos
> - Desplegar la configuración limpiamente en varias máquinas

> [!IMPORTANT]
> Este capítulo asume que el [capítulo 10](10_shell_config_es.md) está dominado: todo lo que sigue solo **genera** o **llena** los archivos `.zshrc`, `.bashrc` y `.tmux.conf` que ya sabes leer.

## Tabla de contenidos

1. [Antes de empezar: ¿dónde instalar todo esto?](#antes-de-empezar-dónde-instalar-todo-esto)
2. [Oh My Zsh](#oh-my-zsh)
3. [Los plugins indispensables](#los-plugins-indispensables)
4. [Los temas y Powerlevel10k](#los-temas-y-powerlevel10k)
5. [Starship — la alternativa universal](#starship--la-alternativa-universal)
6. [Oh My Bash](#oh-my-bash)
7. [Oh My Tmux](#oh-my-tmux)
8. [Las Nerd Fonts](#las-nerd-fonts)
9. [Herramientas CLI modernas](#herramientas-cli-modernas)
10. [fzf — la búsqueda difusa](#fzf--la-búsqueda-difusa)
11. [Desplegar la configuración en varios servidores](#desplegar-la-configuración-en-varios-servidores)
12. [Resumen](#resumen)

---

## Antes de empezar: ¿dónde instalar todo esto?

> [!CAUTION]
> **En un servidor de producción, no instales frameworks de shell.**
>
> | Máquina | Recomendación |
> |---------|----------------|
> | **Puesto de trabajo / máquina de dev** | Sí — la comodidad y la velocidad ganadas son reales |
> | **Servidor de producción** | No — como mucho un `.bashrc` mínimo, versionado y revisado |
>
> Las razones: un framework añade decenas de milisegundos al arranque de **cada** shell (incluso los abiertos por tus scripts), introduce dependencias no gestionadas por el gestor de paquetes, y un `.zshrc` roto en un servidor remoto puede cortarte el acceso.

> [!WARNING]
> **Sobre las instalaciones `curl | bash`.**
> La mayoría de herramientas de este capítulo se instalan con un comando como `curl -fsSL https://… | sh`. Esto equivale a ejecutar código arbitrario descargado de Internet, con tus derechos.
> El reflejo mínimo: descarga el script primero, léelo, luego ejecútalo.
>
> ```bash
> curl -fsSL https://ejemplo.com/install.sh -o install.sh
> less install.sh          # lee lo que estás a punto de lanzar
> sh install.sh
> ```

---

## Oh My Zsh

[Oh My Zsh](https://ohmyz.sh/) es un framework de configuración para zsh: proporciona un `.zshrc` estructurado, ~300 plugins y ~150 temas.

### Requisitos previos

```bash
sudo apt install zsh git curl
```

```bash
zsh --version
```

### Instalación

```bash
sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
```

El instalador:

1. clona el repositorio en `~/.oh-my-zsh`;
2. guarda tu `.zshrc` existente en `~/.zshrc.pre-oh-my-zsh`;
3. genera un nuevo `~/.zshrc`;
4. propone hacer de zsh tu shell por defecto (`chsh`).

### Anatomía del `.zshrc` generado

```bash
# Ruta de la instalación
export ZSH="$HOME/.oh-my-zsh"

# El tema utilizado
ZSH_THEME="robbyrussell"

# Los plugins activados (atención: separados por ESPACIOS, no comas)
plugins=(git docker sudo)

# Carga el framework — todo lo anterior debe estar definido ANTES de esta línea
source $ZSH/oh-my-zsh.sh

# ── Tu configuración personal va aquí ──
alias ll='ls -alh'
export EDITOR=vim
```

> [!IMPORTANT]
> Las variables `ZSH_THEME` y `plugins` deben definirse **antes** de `source $ZSH/oh-my-zsh.sh`. Colocadas después, no tienen efecto — es el error más frecuente.

Recarga después de modificar:

```bash
source ~/.zshrc
```

### Actualización y desinstalación

```bash
omz update           # actualizar Oh My Zsh
```

```bash
uninstall_oh_my_zsh  # desinstalar y restaurar el antiguo .zshrc
```

### Algunos plugins integrados útiles

Ya están presentes en `~/.oh-my-zsh/plugins/`, solo hay que añadirlos a la lista.

| Plugin | Aportación |
|--------|--------|
| `git` | Decenas de alias (`gst`, `gco`, `gp`…) y la rama en el prompt |
| `sudo` | **Doble `ESC`**: prefija el comando actual con `sudo` |
| `docker` / `docker-compose` | Autocompletado de contenedores, imágenes, servicios |
| `systemd` | Alias `sc-start`, `sc-status`, `sc-restart`… |
| `history` | Alias `h`, `hs` para buscar en el historial |
| `command-not-found` | Sugiere el paquete a instalar cuando el comando no existe |
| `extract` | El comando `x archivo.tar.gz` — extrae cualquier formato |
| `colored-man-pages` | Páginas de manual en color |

```bash
plugins=(git sudo docker systemd extract colored-man-pages command-not-found)
```

> [!TIP]
> Cada plugin se carga al arrancar el shell. Una lista de 30 plugins hace que abrir una terminal sea notablemente lento — quédate con los que realmente usas.

---

## Los plugins indispensables

Dos plugins **no** están incluidos y deben clonarse. Con diferencia, son los más útiles.

### zsh-autosuggestions

Propone en gris el final del comando según tu historial — `→` para aceptar.

```bash
git clone https://github.com/zsh-users/zsh-autosuggestions \
  "${ZSH_CUSTOM:-$HOME/.oh-my-zsh/custom}/plugins/zsh-autosuggestions"
```

### zsh-syntax-highlighting

Colorea el comando mientras escribes: verde si existe, rojo si no. Ves tus errores tipográficos **antes** de validar.

```bash
git clone https://github.com/zsh-users/zsh-syntax-highlighting.git \
  "${ZSH_CUSTOM:-$HOME/.oh-my-zsh/custom}/plugins/zsh-syntax-highlighting"
```

### Activación

```bash
plugins=(git sudo zsh-autosuggestions zsh-syntax-highlighting)
```

> [!WARNING]
> `zsh-syntax-highlighting` debe ser **el último** de la lista: redefine los widgets de entrada y debe pasar después de todos los demás.

```bash
source ~/.zshrc
```

---

## Los temas y Powerlevel10k

### Cambiar de tema

```bash
ZSH_THEME="agnoster"
```

Los temas disponibles están en `~/.oh-my-zsh/themes/`. Algunos clásicos:

| Tema | Estilo |
|-------|-------|
| `robbyrussell` | El defecto — sobrio y rápido |
| `agnoster` | Estilo *powerline* (requiere una Nerd Font) |
| `af-magic` | Ruta completa, rama git a la derecha |
| `random` | Un tema diferente cada vez que se abre — útil para elegir |

### Powerlevel10k

[Powerlevel10k](https://github.com/romkatv/powerlevel10k) es el tema de referencia: muy rápido, muy configurable, con un asistente de configuración.

```bash
git clone --depth=1 https://github.com/romkatv/powerlevel10k.git \
  "${ZSH_CUSTOM:-$HOME/.oh-my-zsh/custom}/themes/powerlevel10k"
```

En `~/.zshrc`:

```bash
ZSH_THEME="powerlevel10k/powerlevel10k"
```

Luego:

```bash
source ~/.zshrc
```

Un asistente se ejecuta al abrir por primera vez y escribe tus respuestas en `~/.p10k.zsh`. Para ejecutarlo de nuevo:

```bash
p10k configure
```

> [!NOTE]
> Powerlevel10k requiere una fuente compatible para mostrar sus iconos y separadores — ver [Las Nerd Fonts](#las-nerd-fonts). Sin ella, el prompt muestra cuadrados o signos de interrogación.

---

## Starship — la alternativa universal

[Starship](https://starship.rs/) no es un framework sino **solo un prompt**. Su interés: está escrito en Rust (arranque casi instantáneo) y funciona idénticamente bajo **bash, zsh, fish y PowerShell**.

```bash
curl -sS https://starship.rs/install.sh | sh
```

Activación — una sola línea al **final** del archivo de config:

```bash
# ~/.bashrc
eval "$(starship init bash)"
```

```bash
# ~/.zshrc
eval "$(starship init zsh)"
```

Configuración en `~/.config/starship.toml`:

```toml
add_newline = true

[character]
success_symbol = "[➜](bold green)"
error_symbol = "[➜](bold red)"

[directory]
truncation_length = 3

[git_branch]
symbol = " "
```

| | Oh My Zsh + p10k | Starship |
|---|---|---|
| Alcance | solo zsh | Todos los shells |
| Proporciona | Prompt + plugins + alias | Solo el prompt |
| Configuración | `.zshrc` (shell) | `starship.toml` (declarativo) |
| Buena opción si… | Vives en zsh | Alternas bash/zsh, o quieres mantenerte ligero |

---

## Oh My Bash

[Oh My Bash](https://github.com/ohmybash/oh-my-bash) es el equivalente de Oh My Zsh para quienes se quedan con bash.

```bash
bash -c "$(curl -fsSL https://raw.githubusercontent.com/ohmybash/oh-my-bash/master/tools/install.sh)"
```

La estructura del `.bashrc` generado es la misma:

```bash
export OSH="$HOME/.oh-my-bash"

OSH_THEME="font"

completions=(git ssh)
aliases=(general)
plugins=(git bashmarks)

source "$OSH/oh-my-bash.sh"
```

> [!NOTE]
> Oh My Bash es notablemente menos rico que Oh My Zsh — bash no ofrece las mismas posibilidades de extensión. Si la comodidad del shell es un objetivo, cambiar a zsh es la opción más rentable; si no, `bash-completion` + `starship` ya cubre lo esencial.

---

## Oh My Tmux

[Oh My Tmux](https://github.com/gpakosz/.tmux) es una configuración de tmux lista para usar: barra de estado legible, soporte de ratón, atajos coherentes. Complementa el [capítulo 09](09_tmux_es.md).

### Instalación

```bash
cd ~
git clone https://github.com/gpakosz/.tmux.git
ln -s -f .tmux/.tmux.conf
cp .tmux/.tmux.conf.local .
```

> [!IMPORTANT]
> **Nunca modifiques `~/.tmux.conf`**: es un enlace simbólico al repositorio, tus cambios se perderían en la siguiente actualización.
> Toda personalización va en **`~/.tmux.conf.local`**, que es tu archivo.

### Personalizaciones comunes

En `~/.tmux.conf.local`:

```bash
# Activar el ratón (redimensionar, seleccionar, desplazarse)
set -g mouse on

# Numerar ventanas comenzando en 1 en lugar de 0
set -g base-index 1
setw -g pane-base-index 1

# Historial más largo
set -g history-limit 20000

# Tema de la barra de estado
tmux_conf_theme_status_left=' ❐ #S '
tmux_conf_theme_status_right='#{prefix}#{pairing} %R , %d %b | #{username}@#{hostname} '

# Copiar al portapapeles del sistema
tmux_conf_copy_to_os_clipboard=true
```

Recarga la configuración sin salir de tmux:

```text
CTRL + b  =>  r
```

o

```bash
tmux source-file ~/.tmux.conf
```

### Recordatorio: el ratón

Una vez que `set -g mouse on` está activo, puedes desplazarte, hacer clic para cambiar panel y redimensionar con el ratón. El atajo de alternancia proporcionado por oh-my-tmux:

```text
CTRL + b  =>  m
```

---

## Las Nerd Fonts

Los temas modernos (Powerlevel10k, agnoster, starship) muestran iconos que no existen en fuentes estándar. Las [Nerd Fonts](https://www.nerdfonts.com/) son fuentes clásicas **enriquecidas** con miles de glifos.

> [!IMPORTANT]
> La fuente se instala **en tu máquina local** — donde se ejecuta el emulador de terminal — y **no** en el servidor remoto. El servidor envía caracteres; es tu terminal quien los dibuja.

### Instalación en Linux

```bash
mkdir -p ~/.local/share/fonts
cd ~/.local/share/fonts

curl -fLo "MesloLGS NF Regular.ttf" \
  https://github.com/romkatv/powerlevel10k-media/raw/master/MesloLGS%20NF%20Regular.ttf

fc-cache -fv
```

Luego selecciona la fuente en las preferencias de tu terminal (GNOME Terminal, Konsole, Windows Terminal, iTerm2…).

Fuentes más utilizadas: **MesloLGS NF**, **FiraCode Nerd Font**, **JetBrainsMono Nerd Font**, **Hack Nerd Font**.

---

## Herramientas CLI modernas

Una buena parte de la comodidad no proviene del shell, sino del reemplazo de herramientas históricas.

| Clásica | Alternativa | Aportación |
|-----------|-------------|--------|
| `cat` | **bat** | Coloreado sintáctico, números de línea, integración git |
| `ls` | **eza** (ex-`exa`) | Colores, iconos, árbol, estado git |
| `find` | **fd** | Sintaxis simple, rápida, respeta `.gitignore` |
| `grep` | **ripgrep** (`rg`) | Mucho más rápida, recursiva por defecto |
| `du` | **ncdu** | Explorador interactivo de uso de disco |
| `top` | **btop** / **htop** | Interfaz legible, gráficos |
| `cd` | **zoxide** | Desplazarse por frecuencia de uso: `z proyecto` |
| `man` | **tldr** | Ejemplos concretos en lugar de un manual completo |
| — | **jq** | Manipular JSON en línea de comandos |

### Instalación (Debian / Ubuntu)

```bash
sudo apt install bat fd-find ripgrep ncdu btop jq tldr
```

> [!NOTE]
> En Debian y Ubuntu, `bat` se instala como **`batcat`** y `fd` como **`fdfind`** (conflictos de nombres con otros paquetes). De donde los alias en el `.bashrc`:
>
> ```bash
> alias bat='batcat'
> alias fd='fdfind'
> ```

`eza` y `zoxide` no siempre están en los repositorios:

```bash
# eza
sudo apt install eza    # o: cargo install eza

# zoxide
curl -sS https://raw.githubusercontent.com/ajeetdsouza/zoxide/main/install.sh | sh
```

Luego, en `~/.bashrc` / `~/.zshrc`:

```bash
alias ls='eza --icons --group-directories-first'
alias ll='eza -alh --icons --git'
alias tree='eza --tree'

eval "$(zoxide init bash)"   # o zsh
```

---

## fzf — la búsqueda difusa

[fzf](https://github.com/junegunn/fzf) merece su propia sección: probablemente es la herramienta que más cambia la forma de trabajar en una terminal.

```bash
sudo apt install fzf
```

Una vez integrada en el shell, reemplaza tres cosas:

| Atajo | Efecto |
|-----------|-------|
| `CTRL + R` | Búsqueda **difusa** en el historial (reemplaza la búsqueda nativa) |
| `CTRL + T` | Insertar una ruta de archivo encontrada interactivamente |
| `ALT + C` | Desplazarse a una carpeta elegida interactivamente |

Integración:

```bash
# ~/.bashrc
source /usr/share/doc/fzf/examples/key-bindings.bash
source /usr/share/doc/fzf/examples/completion.bash
```

```bash
# ~/.zshrc — o simplemente el plugin de Oh My Zsh:
plugins=(git fzf)
```

Se combina con todo lo que produce una lista:

```bash
# Elegir un servicio systemd para reiniciar
sudo systemctl restart "$(systemctl list-units --type=service --plain --no-legend | awk '{print $1}' | fzf)"

# Matar un proceso elegido interactivamente
kill -9 "$(ps aux | fzf | awk '{print $2}')"
```

---

## Desplegar la configuración en varios servidores

Reproducir todo esto manualmente en 20 máquinas no es viable. Tres enfoques, por orden de seriedad:

### 1. Un repositorio de dotfiles + un script de instalación

```bash
git clone https://github.com/yo/dotfiles.git ~/dotfiles
cd ~/dotfiles && ./install.sh
```

El `install.sh` crea los enlaces simbólicos (ver [capítulo 10](10_shell_config_es.md#organizar-y-versionar-tus-dotfiles)) e instala los paquetes faltantes.

### 2. GNU Stow

```bash
sudo apt install stow
cd ~/dotfiles
stow bash tmux zsh     # crea automáticamente los enlaces en ~
```

### 3. Una herramienta de gestión de configuración

Ansible, Salt o Puppet, en cuanto se trata de un verdadero parque de servidores. También es lo que garantiza que la configuración permanezca **idéntica y reproducible** — un `curl | bash` ejecutado manualmente hace seis meses nunca lo es.

> [!TIP]
> En un servidor donde no quieres instalar nada, aún es posible llevar tu config durante una sesión:
>
> ```bash
> ssh servidor -t "bash --rcfile <(curl -fsSL https://mi.sitio/bashrc)"
> ```

---

## Resumen

| Herramienta | Rol | Archivo de config |
|-------|------|-------------------|
| **Oh My Zsh** | Framework zsh (plugins, temas, alias) | `~/.zshrc` |
| **Powerlevel10k** | Tema de prompt zsh | `~/.p10k.zsh` |
| **Starship** | Prompt universal multi-shell | `~/.config/starship.toml` |
| **Oh My Bash** | Framework bash | `~/.bashrc` |
| **Oh My Tmux** | Configuración tmux | `~/.tmux.conf.local` |
| **Nerd Fonts** | Fuentes con iconos | Preferencias del terminal local |
| **fzf** | Búsqueda difusa interactiva | `~/.bashrc` / `~/.zshrc` |
| **bat, eza, fd, rg, zoxide** | Reemplazantes modernos de herramientas de base | alias en el archivo `rc` |

**Una configuración cómoda en cinco pasos:**

1. `zsh` + Oh My Zsh
2. Plugins `zsh-autosuggestions` y `zsh-syntax-highlighting`
3. Una Nerd Font + Powerlevel10k (o Starship)
4. `fzf` y `zoxide`
5. Todo en un repositorio git de dotfiles

> [!CAUTION]
> **Y el recordatorio que cuenta**: todo esto vale para tu máquina de trabajo. En un servidor, el valor de un shell radica en su previsibilidad, no en su estética. Debes seguir siendo eficiente en un `bash` desnudo, con solo `vi` disponible — es exactamente lo que encontrarás el día de un incidente.

---

⬅️ [Anterior: 10 · Configuración del shell](10_shell_config_es.md) · 🏠 [Tabla de contenidos](README_es.md)
