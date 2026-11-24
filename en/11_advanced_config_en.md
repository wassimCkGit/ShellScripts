# 11 · Advanced Configuration (`oh-my-*`, Themes and Tools)

> [!NOTE]
> **Chapter Objectives**
> - Install and configure **Oh My Zsh**, its themes and plugins
> - Know the alternatives: **Oh My Bash**, **Starship**
> - Configure **Oh My Tmux**
> - Install necessary fonts (**Nerd Fonts**)
> - Replace historical tools with modern equivalents
> - Deploy your configuration cleanly across multiple machines

> [!IMPORTANT]
> This chapter assumes [chapter 10](10_shell_config.md) is mastered: everything that follows merely **generates** or **fills** the `.zshrc`, `.bashrc` and `.tmux.conf` files you now know how to read.

## Table of Contents

1. [Before Starting: Where to Install All This?](#before-starting-where-to-install-all-this)
2. [Oh My Zsh](#oh-my-zsh)
3. [Essential Plugins](#essential-plugins)
4. [Themes and Powerlevel10k](#themes-and-powerlevel10k)
5. [Starship — The Universal Alternative](#starship--the-universal-alternative)
6. [Oh My Bash](#oh-my-bash)
7. [Oh My Tmux](#oh-my-tmux)
8. [Nerd Fonts](#nerd-fonts)
9. [Modern CLI Tools](#modern-cli-tools)
10. [fzf — Fuzzy Search](#fzf--fuzzy-search)
11. [Deploy Your Configuration Across Multiple Servers](#deploy-your-configuration-across-multiple-servers)
12. [Summary](#summary)

---

## Before Starting: Where to Install All This?

> [!CAUTION]
> **On a production server, do not install a shell framework.**
>
> | Machine | Recommendation |
> |---------|---|
> | **Workstation / dev machine** | Yes — the comfort and speed gains are real |
> | **Production server** | No — at best a minimal, versioned, reviewed `.bashrc` |
>
> Reasons: a framework adds tens of milliseconds to **every** shell startup (including those opened by your scripts), introduces unmanaged dependencies, and a broken `.zshrc` on a remote server can cut off your access.

> [!WARNING]
> **On `curl | bash` installations.**
> Most tools in this chapter install with a command like `curl -fsSL https://… | sh`. This amounts to executing arbitrary code downloaded from the internet, with your rights.
> The minimum reflex: download the script first, read it, then execute it.
>
> ```bash
> curl -fsSL https://example.com/install.sh -o install.sh
> less install.sh          # read what you're about to launch
> sh install.sh
> ```

---

## Oh My Zsh

[Oh My Zsh](https://ohmyz.sh/) is a configuration framework for zsh: it provides a structured `.zshrc`, ~300 plugins and ~150 themes.

### Prerequisites

```bash
sudo apt install zsh git curl
```

```bash
zsh --version
```

### Installation

```bash
sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
```

The installer:

1. clones the repository into `~/.oh-my-zsh`;
2. backs up your existing `.zshrc` to `~/.zshrc.pre-oh-my-zsh`;
3. generates a new `~/.zshrc`;
4. offers to make zsh your default shell (`chsh`).

### Anatomy of the Generated `.zshrc`

```bash
# Installation path
export ZSH="$HOME/.oh-my-zsh"

# The theme used
ZSH_THEME="robbyrussell"

# Enabled plugins (note: separated by SPACES, not commas)
plugins=(git docker sudo)

# Load the framework — everything above must be defined BEFORE this line
source $ZSH/oh-my-zsh.sh

# ── Your personal configuration goes here ──
alias ll='ls -alh'
export EDITOR=vim
```

> [!IMPORTANT]
> The `ZSH_THEME` and `plugins` variables must be defined **before** `source $ZSH/oh-my-zsh.sh`. Placed after, they have no effect — this is the most common error.

Reload after modification:

```bash
source ~/.zshrc
```

### Update and Uninstall

```bash
omz update           # update Oh My Zsh
```

```bash
uninstall_oh_my_zsh  # uninstall and restore the old .zshrc
```

### Some Useful Built-in Plugins

They're already present in `~/.oh-my-zsh/plugins/`, just add them to the list.

| Plugin | Benefit |
|--------|---------|
| `git` | Dozens of aliases (`gst`, `gco`, `gp`…) and branch in the prompt |
| `sudo` | **Double `ESC`**: prefix the current command with `sudo` |
| `docker` / `docker-compose` | Completion of containers, images, services |
| `systemd` | Aliases `sc-start`, `sc-status`, `sc-restart`… |
| `history` | Aliases `h`, `hs` for searching history |
| `command-not-found` | Suggests the package to install when command is missing |
| `extract` | The `x file.tar.gz` command — extracts any format |
| `colored-man-pages` | Colored manual pages |

```bash
plugins=(git sudo docker systemd extract colored-man-pages command-not-found)
```

> [!TIP]
> Each plugin loads at shell startup. A list of 30 plugins makes terminal opening noticeably slow — stick with those you actually use.

---

## Essential Plugins

Two plugins are **not** included and must be cloned. They are by far the most useful.

### zsh-autosuggestions

Suggests the end of the command in gray based on your history — `→` to accept.

```bash
git clone https://github.com/zsh-users/zsh-autosuggestions \
  "${ZSH_CUSTOM:-$HOME/.oh-my-zsh/custom}/plugins/zsh-autosuggestions"
```

### zsh-syntax-highlighting

Colors the command while typing: green if it exists, red otherwise. You see your typos **before** pressing enter.

```bash
git clone https://github.com/zsh-users/zsh-syntax-highlighting.git \
  "${ZSH_CUSTOM:-$HOME/.oh-my-zsh/custom}/plugins/zsh-syntax-highlighting"
```

### Activation

```bash
plugins=(git sudo zsh-autosuggestions zsh-syntax-highlighting)
```

> [!WARNING]
> `zsh-syntax-highlighting` must be **last** in the list: it redefines input widgets and must come after all others.

```bash
source ~/.zshrc
```

---

## Themes and Powerlevel10k

### Change Theme

```bash
ZSH_THEME="agnoster"
```

Available themes are in `~/.oh-my-zsh/themes/`. Some classics:

| Theme | Style |
|-------|-------|
| `robbyrussell` | The default — sober and fast |
| `agnoster` | *Powerline* style (requires a Nerd Font) |
| `af-magic` | Full path, git branch on the right |
| `random` | A different theme each time — handy for choosing |

### Powerlevel10k

[Powerlevel10k](https://github.com/romkatv/powerlevel10k) is the reference theme: very fast, highly configurable, with a configuration assistant.

```bash
git clone --depth=1 https://github.com/romkatv/powerlevel10k.git \
  "${ZSH_CUSTOM:-$HOME/.oh-my-zsh/custom}/themes/powerlevel10k"
```

In `~/.zshrc`:

```bash
ZSH_THEME="powerlevel10k/powerlevel10k"
```

Then:

```bash
source ~/.zshrc
```

A wizard launches on first open and writes your answers to `~/.p10k.zsh`. To re-run it:

```bash
p10k configure
```

> [!NOTE]
> Powerlevel10k requires a compatible font to display its icons and separators — see [Nerd Fonts](#nerd-fonts). Without it, the prompt displays squares or question marks.

---

## Starship — The Universal Alternative

[Starship](https://starship.rs/) is not a framework but **only a prompt**. Its advantage: it's written in Rust (nearly instant startup) and works identically under **bash, zsh, fish and PowerShell**.

```bash
curl -sS https://starship.rs/install.sh | sh
```

Activation — a single line at the **end** of your config file:

```bash
# ~/.bashrc
eval "$(starship init bash)"
```

```bash
# ~/.zshrc
eval "$(starship init zsh)"
```

Configuration in `~/.config/starship.toml`:

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
| Scope | zsh only | All shells |
| Provides | Prompt + plugins + aliases | Prompt only |
| Configuration | `.zshrc` (shell) | `starship.toml` (declarative) |
| Good choice if… | You live in zsh | You alternate bash/zsh, or want to stay lightweight |

---

## Oh My Bash

[Oh My Bash](https://github.com/ohmybash/oh-my-bash) is the Oh My Zsh equivalent for those staying with bash.

```bash
bash -c "$(curl -fsSL https://raw.githubusercontent.com/ohmybash/oh-my-bash/master/tools/install.sh)"
```

The structure of the generated `.bashrc` is the same:

```bash
export OSH="$HOME/.oh-my-bash"

OSH_THEME="font"

completions=(git ssh)
aliases=(general)
plugins=(git bashmarks)

source "$OSH/oh-my-bash.sh"
```

> [!NOTE]
> Oh My Bash is significantly less rich than Oh My Zsh — bash doesn't offer the same extension possibilities. If shell comfort is a goal, switching to zsh is the most worthwhile choice; otherwise, `bash-completion` + `starship` already covers the essentials.

---

## Oh My Tmux

[Oh My Tmux](https://github.com/gpakosz/.tmux) is a ready-to-use tmux configuration: readable status bar, mouse support, consistent shortcuts. It complements [chapter 09](09_tmux.md).

### Installation

```bash
cd ~
git clone https://github.com/gpakosz/.tmux.git
ln -s -f .tmux/.tmux.conf
cp .tmux/.tmux.conf.local .
```

> [!IMPORTANT]
> **Never modify `~/.tmux.conf`**: it's a symbolic link to the repository, your changes would be lost at the next update.
> All customization goes in **`~/.tmux.conf.local`**, which is your file.

### Common Customizations

In `~/.tmux.conf.local`:

```bash
# Enable mouse (resize, select, scroll)
set -g mouse on

# Number windows from 1 rather than 0
set -g base-index 1
setw -g pane-base-index 1

# Longer history
set -g history-limit 20000

# Status bar theme
tmux_conf_theme_status_left=' ❐ #S '
tmux_conf_theme_status_right='#{prefix}#{pairing} %R , %d %b | #{username}@#{hostname} '

# Copy to system clipboard
tmux_conf_copy_to_os_clipboard=true
```

Reload configuration without leaving tmux:

```text
CTRL + b  =>  r
```

or

```bash
tmux source-file ~/.tmux.conf
```

### Reminder: The Mouse

Once `set -g mouse on` is active, you can scroll, click to change panes and resize with the mouse. The toggle shortcut provided by oh-my-tmux:

```text
CTRL + b  =>  m
```

---

## Nerd Fonts

Modern themes (Powerlevel10k, agnoster, starship) display icons that don't exist in standard fonts. [Nerd Fonts](https://www.nerdfonts.com/) are classic fonts **enriched** with these thousands of glyphs.

> [!IMPORTANT]
> The font installs **on your local machine** — the one running your terminal emulator — **not** on the remote server. The server sends characters; your terminal draws them.

### Installation on Linux

```bash
mkdir -p ~/.local/share/fonts
cd ~/.local/share/fonts

curl -fLo "MesloLGS NF Regular.ttf" \
  https://github.com/romkatv/powerlevel10k-media/raw/master/MesloLGS%20NF%20Regular.ttf

fc-cache -fv
```

Then select the font in your terminal's preferences (GNOME Terminal, Konsole, Windows Terminal, iTerm2…).

Most popular fonts: **MesloLGS NF**, **FiraCode Nerd Font**, **JetBrainsMono Nerd Font**, **Hack Nerd Font**.

---

## Modern CLI Tools

Much of the comfort doesn't come from the shell, but from replacing historical tools.

| Classic | Alternative | Benefit |
|---------|-------------|---------|
| `cat` | **bat** | Syntax coloring, line numbers, git integration |
| `ls` | **eza** (ex-`exa`) | Colors, icons, tree view, git status |
| `find` | **fd** | Simple syntax, fast, respects `.gitignore` |
| `grep` | **ripgrep** (`rg`) | Much faster, recursive by default |
| `du` | **ncdu** | Interactive disk usage explorer |
| `top` | **btop** / **htop** | Readable interface, graphs |
| `cd` | **zoxide** | Navigate by frequency of use: `z project` |
| `man` | **tldr** | Concrete examples rather than complete manual |
| — | **jq** | Manipulate JSON on the command line |

### Installation (Debian / Ubuntu)

```bash
sudo apt install bat fd-find ripgrep ncdu btop jq tldr
```

> [!NOTE]
> On Debian and Ubuntu, `bat` installs as **`batcat`** and `fd` as **`fdfind`** (name conflicts with other packages). Hence aliases in `.bashrc`:
>
> ```bash
> alias bat='batcat'
> alias fd='fdfind'
> ```

`eza` and `zoxide` aren't always in the repositories:

```bash
# eza
sudo apt install eza    # or: cargo install eza

# zoxide
curl -sS https://raw.githubusercontent.com/ajeetdsouza/zoxide/main/install.sh | sh
```

Then, in `~/.bashrc` / `~/.zshrc`:

```bash
alias ls='eza --icons --group-directories-first'
alias ll='eza -alh --icons --git'
alias tree='eza --tree'

eval "$(zoxide init bash)"   # or zsh
```

---

## fzf — Fuzzy Search

[fzf](https://github.com/junegunn/fzf) deserves its own section: it's probably the tool that most changes how you work in a terminal.

```bash
sudo apt install fzf
```

Once integrated into the shell, it replaces three things:

| Shortcut | Effect |
|----------|--------|
| `CTRL + R` | **Fuzzy** search in history (replaces native search) |
| `CTRL + T` | Insert a found file path interactively |
| `ALT + C` | Navigate to a chosen directory interactively |

Integration:

```bash
# ~/.bashrc
source /usr/share/doc/fzf/examples/key-bindings.bash
source /usr/share/doc/fzf/examples/completion.bash
```

```bash
# ~/.zshrc — or simply the Oh My Zsh plugin:
plugins=(git fzf)
```

It combines with anything that produces a list:

```bash
# Choose a systemd service to restart
sudo systemctl restart "$(systemctl list-units --type=service --plain --no-legend | awk '{print $1}' | fzf)"

# Kill a chosen process… finally, with the keyboard
kill -9 "$(ps aux | fzf | awk '{print $2}')"
```

---

## Deploy Your Configuration Across Multiple Servers

Doing all this by hand on 20 machines isn't viable. Three approaches, by order of seriousness:

### 1. A Dotfiles Repository + Installation Script

```bash
git clone https://github.com/me/dotfiles.git ~/dotfiles
cd ~/dotfiles && ./install.sh
```

The `install.sh` creates symbolic links (see [chapter 10](10_shell_config.md#organize-and-version-your-dotfiles)) and installs missing packages.

### 2. GNU Stow

```bash
sudo apt install stow
cd ~/dotfiles
stow bash tmux zsh     # automatically creates links in ~
```

### 3. Configuration Management Tool

Ansible, Salt or Puppet, once you have a real server fleet. This also guarantees the configuration stays **identical and reproducible** — a `curl | bash` run manually six months ago never is.

> [!TIP]
> On a server where you don't want to install anything, you can still bring your config for the duration of a session:
>
> ```bash
> ssh server -t "bash --rcfile <(curl -fsSL https://my.site/bashrc)"
> ```

---

## Summary

| Tool | Role | Config File |
|------|------|------------|
| **Oh My Zsh** | zsh framework (plugins, themes, aliases) | `~/.zshrc` |
| **Powerlevel10k** | zsh prompt theme | `~/.p10k.zsh` |
| **Starship** | Universal multi-shell prompt | `~/.config/starship.toml` |
| **Oh My Bash** | bash framework | `~/.bashrc` |
| **Oh My Tmux** | tmux configuration | `~/.tmux.conf.local` |
| **Nerd Fonts** | Fonts with icons | Local terminal preferences |
| **fzf** | Interactive fuzzy search | `~/.bashrc` / `~/.zshrc` |
| **bat, eza, fd, rg, zoxide** | Modern replacements for basic tools | Aliases in `rc` file |

**A comfortable configuration in five steps:**

1. `zsh` + Oh My Zsh
2. `zsh-autosuggestions` and `zsh-syntax-highlighting` plugins
3. A Nerd Font + Powerlevel10k (or Starship)
4. `fzf` and `zoxide`
5. All of this in a git dotfiles repository

> [!CAUTION]
> **And the important reminder**: all of this applies to your workstation. On a server, shell value lies in its predictability, not its aesthetics. You must remain efficient on bare `bash`, with only `vi` available — exactly what you'll find in an incident.

---

⬅️ [Previous: 10 · Shell Configuration](10_shell_config.md) · 🏠 [Table of Contents](README.md)
