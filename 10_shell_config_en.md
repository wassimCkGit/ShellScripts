# 10 · Shell Configuration (`.bashrc`, `.zshrc`, …)

> [!NOTE]
> **Chapter objectives**
> - Know which shell you are using and how to change it
> - Understand the difference between a **login** shell and an **interactive** shell
> - Know **which file** to edit, and why
> - Create persistent aliases, functions, and environment variables
> - Customize your `PATH`, prompt, and history
> - Organize and version your configuration files (*dotfiles*)

## Table of Contents

1. [Which shell am I using?](#which-shell-am-i-using)
2. [Login, interactive: understanding the four cases](#login-interactive-understanding-the-four-cases)
3. [Which file is loaded, and when?](#which-file-is-loaded-and-when)
4. [Where should I put my configuration?](#where-should-i-put-my-configuration)
5. [Applying your changes](#applying-your-changes)
6. [Aliases](#aliases)
7. [Functions](#functions)
8. [Environment variables](#environment-variables)
9. [The PATH](#the-path)
10. [Customizing the prompt](#customizing-the-prompt)
11. [Mastering the history](#mastering-the-history)
12. [Shell options](#shell-options)
13. [System configuration and /etc/skel](#system-configuration-and-etcskel)
14. [Organizing and versioning your dotfiles](#organizing-and-versioning-your-dotfiles)
15. [Classic pitfalls](#classic-pitfalls)
16. [Summary](#summary)

---

## Which shell am I using?

The shell is the program that interprets what you type in the terminal. Several of them coexist on the same machine.

| Shell | Path | Note |
|-------|------|------|
| **sh** | `/bin/sh` | The minimal POSIX shell. On Debian, it is actually `dash`. |
| **bash** | `/bin/bash` | *Bourne Again Shell* — the de facto standard on Linux |
| **zsh** | `/usr/bin/zsh` | Bash-compatible, richer (completion, globbing, themes) |
| **fish** | `/usr/bin/fish` | Very user-friendly, but **not** POSIX-compatible |

Find your login shell:

```bash
echo $SHELL
```

Find the shell **actually running**:

```bash
ps -p $$
```

List the installed shells:

```bash
cat /etc/shells
```

### Changing shell

```bash
chsh -s /usr/bin/zsh
```

For another user (as root):

```bash
sudo chsh -s /usr/bin/zsh username
```

> [!IMPORTANT]
> The change only takes effect at the **next login**. First check that the shell works by simply launching it (`zsh`): a broken login shell can make the account unusable.
> A user's shell is stored in the last field of their line in `/etc/passwd`.

---

## Login, interactive: understanding the four cases

This is **the** point that explains 90% of "why doesn't my alias work?". A shell is characterized by two independent properties.

| Shell type | Definition | Concrete example |
|------------|------------|------------------|
| **Login + interactive** | Opened after authentication | SSH connection, `su - user`, TTY console |
| **Non-login + interactive** | Opened from an already authenticated session | A new graphical terminal tab, `bash`, a tmux pane |
| **Non-login + non-interactive** | Runs a script, with no human in front of it | `./my_script.sh`, a cron job |
| **Login + non-interactive** | Rare | `ssh server 'command'`, some deployment scripts |

Check which case you are in:

```bash
# Interactive shell? (the output contains an "i")
echo $-

# Login shell?
shopt -q login_shell && echo "login" || echo "non-login"
```

---

## Which file is loaded, and when?

### Bash

```text
┌─ LOGIN SHELL ───────────────────────┐   ┌─ INTERACTIVE SHELL (non-login) ─┐
│ 1. /etc/profile                     │   │ 1. /etc/bash.bashrc             │
│ 2. /etc/profile.d/*.sh              │   │ 2. ~/.bashrc                    │
│ 3. the FIRST one found among:       │   └─────────────────────────────────┘
│      ~/.bash_profile                │
│      ~/.bash_login                  │   ┌─ SCRIPT (non-interactive) ──────┐
│      ~/.profile                     │   │ No file loaded                  │
│                                     │   │ (unless $BASH_ENV is set)       │
│ …and at logout:                     │   └─────────────────────────────────┘
│      ~/.bash_logout                 │
└─────────────────────────────────────┘
```

| File | Scope | Loaded for |
|------|-------|------------|
| `/etc/profile` | All users | Login shells |
| `/etc/profile.d/*.sh` | All users | Login shells — **the clean place** to add system-wide config |
| `/etc/bash.bashrc` | All users | Interactive shells |
| `~/.bash_profile` | One user | Login shells |
| `~/.profile` | One user | Login shells, **if** `.bash_profile` does not exist |
| `~/.bashrc` | One user | Non-login interactive shells |
| `~/.bash_logout` | One user | At the end of a login shell |

> [!CAUTION]
> **Bash does NOT read `~/.bashrc` in a login shell.**
> That is why most `~/.bash_profile` files contain this bridge:
>
> ```bash
> # Load ~/.bashrc if interactive
> if [ -n "$BASH_VERSION" ] && [ -f "$HOME/.bashrc" ]; then
>     . "$HOME/.bashrc"
> fi
> ```
>
> Without this block, your aliases work in a local terminal but disappear over SSH.

### Zsh

Zsh is more consistent: the file name tells you when it is read.

| Order | File | Loaded for |
|-------|------|------------|
| 1 | `/etc/zshenv` then `~/.zshenv` | **All** shells, including scripts |
| 2 | `/etc/zprofile` then `~/.zprofile` | Login shells |
| 3 | `/etc/zshrc` then `~/.zshrc` | Interactive shells |
| 4 | `/etc/zlogin` then `~/.zlogin` | Login shells (after `.zshrc`) |
| 5 | `~/.zlogout` | At logout |

> [!TIP]
> **Mental mapping bash → zsh:**
> `~/.bashrc` → `~/.zshrc` · `~/.bash_profile` → `~/.zprofile` · *(no bash equivalent)* → `~/.zshenv`

---

## Where should I put my configuration?

This is the practical question. The table below answers it in almost every case.

| What I want to configure | Bash | Zsh |
|--------------------------|------|-----|
| An **alias** | `~/.bashrc` | `~/.zshrc` |
| A **function** | `~/.bashrc` | `~/.zshrc` |
| The **prompt** (`PS1`) | `~/.bashrc` | `~/.zshrc` |
| **Colors**, completion | `~/.bashrc` | `~/.zshrc` |
| An **environment variable** (`EDITOR`, `LANG`) | `~/.bash_profile` or `~/.profile` | `~/.zprofile` |
| The **`PATH`** | `~/.bash_profile` or `~/.profile` | `~/.zprofile` (or `~/.zshenv`) |
| Something **scripts** must see | `/etc/profile.d/` | `~/.zshenv` |
| A config **for all users** | `/etc/profile.d/my_file.sh` | same |

**Simple rule to remember:**

> What is **inherited** by child processes (exported variables, `PATH`) → **profile** file.
> What concerns **typing comfort** (aliases, prompt, completion) → **rc** file.

---

## Applying your changes

After editing a configuration file, it is **not** reloaded automatically. Three options:

```bash
# 1. Reload in the current shell (the most common)
source ~/.bashrc
```

```bash
# 2. Equivalent syntax, more portable
. ~/.bashrc
```

```bash
# 3. Replace the current shell with a fresh one
exec bash -l
```

> [!IMPORTANT]
> `source file` runs the file **in the current shell** — that is what allows aliases and variables to stick.
> `./file` runs it in a **subprocess**: the current shell keeps nothing from it. This is exactly the distinction seen in [chapter 06.6](06_shellscript/06_fonctions/fonctions.md#importer-des-fonctions-dun-autre-script).

> [!TIP]
> Make a backup before editing a config file:
> ```bash
> cp ~/.bashrc ~/.bashrc.bak
> ```

---

## Aliases

An alias is a command shortcut.

```bash
# Syntax: alias name='command'
alias ll='ls -alh'
alias la='ls -A'
alias ..='cd ..'
alias ...='cd ../..'

# Make destructive commands safer (ask for confirmation)
alias rm='rm -i'
alias cp='cp -i'
alias mv='mv -i'

# Administration shortcuts
alias ports='ss -tulpn'
alias myip='ip -4 addr show scope global | grep inet'
alias upd='sudo apt update && sudo apt upgrade'
alias services='systemctl list-units --type=service --state=running'
```

Managing your aliases:

```bash
alias                  # list all defined aliases
alias ll               # show an alias's definition
unalias ll             # remove an alias (for the current session)
\ls                    # run the real command, ignoring the alias
command ls             # same
```

> [!WARNING]
> **Aliases only work in interactive shells.** They are **not** available in your scripts, nor in a cron job.
> So never rely on an alias in a script: write the full command.
>
> Also beware of the side effect of `alias rm='rm -i'`: you get used to a confirmation that will not exist on a server where the alias is not defined.

---

## Functions

When an alias is no longer enough (you need arguments, logic), write a function in your `.bashrc` / `.zshrc`.

```bash
# Create a directory and move into it immediately
mkcd() {
  mkdir -p "$1" && cd "$1"
}

# Extract any archive
extract() {
  case "$1" in
    *.tar.gz|*.tgz)  tar -xzvf "$1" ;;
    *.tar.bz2)       tar -xjvf "$1" ;;
    *.tar)           tar -xvf  "$1" ;;
    *.zip)           unzip     "$1" ;;
    *.gz)            gunzip    "$1" ;;
    *) echo "Unsupported format: $1" >&2; return 1 ;;
  esac
}

# Back up a file before editing it
bak() {
  cp -- "$1" "$1.$(date +%Y%m%d-%H%M%S).bak"
}
```

> [!TIP]
> Practical difference: an **alias** substitutes text at the start of a command, a **function** can receive and manipulate arguments. As soon as you need `$1`, it's a function.

---

## Environment variables

```bash
# Shell variable: visible only in the current shell
MY_VAR="value"

# Environment variable: inherited by all child processes
export MY_VAR="value"

# Both on one line
export EDITOR=vim
```

The most useful variables to set in your profile:

```bash
export EDITOR=vim               # default editor (git, crontab -e, visudo)
export VISUAL=vim
export PAGER=less
export LANG=en_US.UTF-8         # language and encoding
export LESS='-R'                # preserve colors in less
```

Inspecting the environment:

```bash
printenv                # all environment variables
printenv PATH           # just one
env                     # equivalent
set                     # environment variables AND shell variables
unset MY_VAR            # remove a variable
```

Some standard variables:

| Variable | Contents |
|----------|----------|
| `$HOME` | The user's home directory |
| `$USER` | The current user's name |
| `$PWD` | The current directory |
| `$SHELL` | The login shell |
| `$PATH` | Where to look for executables |
| `$?` | Exit code of the last command |
| `$$` | PID of the current shell |

> [!CAUTION]
> **Never** store a secret (password, API token) in a versioned `.bashrc`. Environment variables are inherited by all child processes and often end up in debug logs.
> Use a separate, unversioned file with `chmod 600`:
>
> ```bash
> [ -f ~/.secrets ] && source ~/.secrets
> ```

---

## The PATH

The `PATH` is the list of directories in which the shell looks for executables, separated by `:`.

```bash
echo $PATH
```

```text
/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
```

Add a directory — typically the one holding your own scripts:

```bash
# Put this in ~/.bash_profile, ~/.profile or ~/.zprofile
export PATH="$HOME/bin:$PATH"
```

| Form | Effect |
|------|--------|
| `export PATH="$HOME/bin:$PATH"` | My directory takes **priority** over the system |
| `export PATH="$PATH:$HOME/bin"` | The system takes priority over my directory |

> [!WARNING]
> **Never forget to include `$PATH` again.** Writing `export PATH="$HOME/bin"` replaces the whole `PATH`: no system command can be found anymore.
>
> **Never add `.` (the current directory) to the `PATH`**: a malicious file named `ls` dropped in a shared directory would be run instead of the real one.

Find out which executable will actually run:

```bash
which ls
type -a ls
command -v ls
```

---

## Customizing the prompt

The prompt is defined by the `PS1` variable in bash, `PROMPT` in zsh.

### Bash

```bash
# Debian's default prompt
PS1='\u@\h:\w\$ '
```

| Sequence | Displays |
|----------|----------|
| `\u` | The user name |
| `\h` | The machine name (short) — `\H` for the FQDN |
| `\w` | The full path — `\W` for the directory only |
| `\$` | `#` if root, `$` otherwise |
| `\t` | The time (HH:MM:SS) |
| `\n` | A newline |

With colors:

```bash
PS1='\[\e[32m\]\u@\h\[\e[0m\]:\[\e[34m\]\w\[\e[0m\]\$ '
```

> [!IMPORTANT]
> Color codes must be wrapped in `\[` and `\]`. These markers tell bash that these characters take up **no** space on screen. Without them, the shell miscalculates the prompt length and the line displays incorrectly as soon as you edit a long command.

A useful prompt for administration — red when you are root:

```bash
if [ "$(id -u)" -eq 0 ]; then
  PS1='\[\e[1;31m\]\u@\h\[\e[0m\]:\w# '     # red = careful, I am root
else
  PS1='\[\e[1;32m\]\u@\h\[\e[0m\]:\w\$ '    # green = normal user
fi
```

### Zsh

```bash
PROMPT='%n@%m %~ %# '
```

| Sequence | Displays |
|----------|----------|
| `%n` | User |
| `%m` | Machine |
| `%~` | Path (with `~` for home) |
| `%#` | `#` if root, `%` otherwise |
| `%F{green}…%f` | Color |

> [!TIP]
> Configuring a rich prompt by hand (git branch, execution time, status of the last command) is tedious. Tools do it for you: see [chapter 11 · Advanced configuration](11_advanced_config.md).

---

## Mastering the history

History is the shell's most rewarding tool — provided you configure it.

### Bash

```bash
# Put this in ~/.bashrc

HISTSIZE=10000                    # commands kept in memory
HISTFILESIZE=20000                # commands kept in the file
HISTCONTROL=ignoreboth:erasedups  # ignore duplicates and lines starting with a space
HISTIGNORE="ls:cd:pwd:exit:clear:history"
HISTTIMEFORMAT="%F %T "           # timestamp each command

shopt -s histappend               # append to the history instead of overwriting it
```

### Zsh

```bash
# Put this in ~/.zshrc

HISTFILE=~/.zsh_history
HISTSIZE=10000
SAVEHIST=20000

setopt HIST_IGNORE_ALL_DUPS       # no duplicates
setopt HIST_IGNORE_SPACE          # ignore lines starting with a space
setopt SHARE_HISTORY              # history shared between open terminals
setopt EXTENDED_HISTORY           # timestamps
```

### Using the history

| Shortcut / command | Effect |
|--------------------|--------|
| `CTRL + R` | Incremental search through the history |
| `↑` / `↓` | Previous / next command |
| `history` | Show the numbered history |
| `!42` | Re-run command no. 42 |
| `!!` | Re-run the last command — `sudo !!` |
| `!$` | Last argument of the previous command |
| `history -c` | Clear the history |

> [!TIP]
> `HISTCONTROL=ignorespace` (included in `ignoreboth`) is very handy: a command preceded by a **space** is not saved in the history. Ideal for a command containing a password.

---

## Shell options

### Bash — `shopt`

```bash
shopt -s autocd          # typing "Documents" is the same as "cd Documents"
shopt -s cdspell         # fixes typos in cd commands
shopt -s checkwinsize    # recalculates the terminal size after each command
shopt -s globstar        # enables ** for recursive matching: ls **/*.md
shopt -s nocaseglob      # case-insensitive globbing
```

List all options: `shopt`

### Zsh — `setopt`

```bash
setopt AUTO_CD
setopt CORRECT           # suggests a correction when you make a typo
setopt EXTENDED_GLOB
setopt NO_BEEP
```

### Completion

In bash, advanced completion (command options, service names, git branches…) comes from a package you need to install:

```bash
sudo apt install bash-completion
```

Then, in `~/.bashrc`:

```bash
if [ -f /usr/share/bash-completion/bash_completion ]; then
  . /usr/share/bash-completion/bash_completion
fi
```

In zsh:

```bash
autoload -Uz compinit && compinit
```

---

## System configuration and `/etc/skel`

### Applying a configuration to all users

You do **not** edit `/etc/profile` directly: you drop a file into `/etc/profile.d/`.

```bash
sudo tee /etc/profile.d/company.sh > /dev/null <<'EOF'
export EDITOR=vim
export TZ="Europe/Brussels"
alias ll='ls -alh'
EOF
```

Every `.sh` file in this directory is loaded for each login shell. It is clean, reversible, and survives system updates.

### `/etc/skel` — the skeleton for new accounts

When you create a user with `useradd -m` (or `adduser`), the contents of `/etc/skel` are **copied** into their home directory.

```bash
ls -la /etc/skel
```

```text
.bash_logout
.bashrc
.profile
```

So this is where you put the default configuration that new accounts should receive:

```bash
sudo cp my_standard_bashrc /etc/skel/.bashrc
```

> [!NOTE]
> `/etc/skel` only affects accounts **created after** the change. Existing users keep their files.

---

## Organizing and versioning your dotfiles

A 400-line `.bashrc` quickly becomes unmanageable. Best practice is to split it up:

```bash
# At the end of ~/.bashrc
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

And above all: **version your dotfiles**.

```bash
mkdir ~/dotfiles && cd ~/dotfiles
git init
mv ~/.bashrc ~/dotfiles/bashrc
ln -s ~/dotfiles/bashrc ~/.bashrc
git add . && git commit -m "Initial dotfiles"
```

The files in your home directory are now just symbolic links to the repository: a single source of truth, reusable on all your machines. Tools like **GNU stow** or **chezmoi** automate this setup.

---

## Classic pitfalls

> [!CAUTION]
> **1. A command that prints text in `.bashrc` breaks `scp` and `sftp`.**
> These tools open a non-interactive shell and interpret any unexpected output as an invalid protocol. Always guard this kind of line:
>
> ```bash
> # At the very top of .bashrc
> case $- in
>   *i*) ;;       # interactive: carry on
>     *) return;; # non-interactive: exit immediately
> esac
> ```

> [!CAUTION]
> **2. A syntax error in `.bashrc` breaks all your new terminals.**
> Test before logging out, from a **second terminal**:
>
> ```bash
> bash -n ~/.bashrc     # checks the syntax without running it
> source ~/.bashrc      # actually tests it
> ```
>
> If you are already locked out: `ssh user@server 'bash --norc -i'`, or `bash --norc` on the console.

> [!CAUTION]
> **3. `export PATH="/my/directory"` without `:$PATH`** — no command works anymore.
> Emergency fix in the current shell:
>
> ```bash
> export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
> ```

> [!CAUTION]
> **4. Believing aliases are available in scripts and cron jobs.** They are not. Cron runs neither `.bashrc` nor `.profile`: use absolute paths (see [chapter 04](04_installation_et_services.md#cron)).

---

## Summary

| File | Shell | When |
|------|-------|------|
| `~/.bashrc` | bash | Every interactive shell → aliases, functions, prompt |
| `~/.bash_profile` / `~/.profile` | bash | Login → environment variables, `PATH` |
| `~/.bash_logout` | bash | Logout |
| `~/.zshrc` | zsh | Every interactive shell |
| `~/.zprofile` | zsh | Login |
| `~/.zshenv` | zsh | **All** shells, scripts included |
| `/etc/profile.d/*.sh` | all | Login, for all users |
| `/etc/skel/` | — | Copied into new accounts' home directories |

| Command | Purpose |
|---------|---------|
| `echo $SHELL` / `ps -p $$` | Which shell? |
| `chsh -s /usr/bin/zsh` | Change shell |
| `source ~/.bashrc` | Reload the configuration |
| `alias` / `unalias` | Manage aliases |
| `export VAR=value` | Set an environment variable |
| `printenv` | List the environment |
| `shopt -s` / `setopt` | Enable a shell option |
| `bash -n ~/.bashrc` | Check the syntax before breaking your session |

---

⬅️ [Previous: 09 · tmux](09_tmux.md) · 🏠 [Contents](README.md) · [Next: 11 · Advanced configuration ➡️](11_advanced_config.md)
