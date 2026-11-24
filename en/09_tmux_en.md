# 09 · tmux (Terminal Multiplexer)

It's a very useful tool when doing remote system administration: it allows you to manage **sessions** on the server and **recover** them in case of disconnection.

> [!NOTE]
> **Chapter Objectives**
> - Create, exit and join a tmux session
> - Split a terminal into panes and manage multiple windows
> - Navigate and search in terminal history

> [!TIP]
> **The concept in one sentence:** your work runs on the **server**, not in your terminal. If your SSH connection drops, the tmux session continues running and you find it intact when you reconnect.
>
> Hierarchy: a **session** contains **windows**, which contain **panes**.

## Table of Contents

1. [Session Management](#session-management)
2. [Splitting the Terminal (Panes)](#splitting-the-terminal-panes)
3. [Windows](#windows)
4. [Navigation and Search](#navigation-and-search)
5. [Oh my tmux](#oh-my-tmux)
6. [Cheat Sheet](#cheat-sheet)

---

## The Prefix

Almost all tmux shortcuts start with the combination `CTRL + b`: we call it the **prefix**. You press `CTRL + b`, **release**, then press the action key.

Throughout this chapter, this sequence is noted:

```text
CTRL + b  =>  key
```

---

## Session Management

### 1. Installation

```bash
sudo apt install tmux
```

### 2. Start a Session

```bash
tmux
```

```bash
tmux new -s session-name
```

> [!TIP]
> Always name your sessions (`-s`). Finding `session-name` is much easier than guessing what session `0` corresponds to.

### 3. Exit a Session While Keeping It Active (*detach*)

```text
CTRL + b  =>  d
```

### 4. List Active Sessions

```bash
tmux ls
```

or, from inside tmux:

```text
CTRL + b  =>  s
```

### 5. Join a Session (*attach*)

```bash
tmux a -t session-name
```

```bash
tmux attach -t session-name
```

### 6. Destroy a Session

```bash
tmux kill-session -t session-name
```

```bash
tmux kill -t session-name
```

```bash
tmux k -t session-name
```

### 7. Rename a Session

```text
CTRL + b  =>  $
```

### 8. Change the Default Working Directory

When you open a new terminal in a session, it positions itself in a default CWD (*current working directory*).

```text
CTRL + b  =>  :
```

then enter the command:

```text
attach -c /new/directory
```

The idea is to automatically position yourself in the right folder when opening new terminals in a tmux session — for example a project's working folder.

---

## Splitting the Terminal (Panes)

You can divide the terminal into multiple parts:

| Shortcut | Action |
|----------|--------|
| `CTRL + b  =>  %` | Split **vertical** (side by side) |
| `CTRL + b  =>  "` | Split **horizontal** (one above the other) |

> [!NOTE]
> The names are confusing: `%` separates the screen with a vertical bar, `"` with a horizontal bar.

### Move Between Panes

| Shortcut | Action |
|----------|--------|
| `CTRL + b  =>  ←↑↓→` | Go to the pane in that direction |
| `CTRL + b  =>  o` | Cycle clockwise |
| `CTRL + b  =>  {` | Cycle counter-clockwise |
| `CTRL + b  =>  ;` | Toggle to the last used pane |
| `CTRL + b  =>  z` | Zoom / unzoom the current pane (fullscreen) |
| `CTRL + b  =>  x` | Close the current pane |

---

## Windows

| Shortcut | Action |
|----------|--------|
| `CTRL + b  =>  c` | **C**reate a window |
| `CTRL + b  =>  ,` | Rename the current window |
| `CTRL + b  =>  !` | Extract the current pane into its own window |
| `CTRL + b  =>  &` | Close the active window |
| `CTRL + b  =>  n` | Next wi**n**dow |
| `CTRL + b  =>  p` | **P**revious window |
| `CTRL + b  =>  w` | List windows |
| `CTRL + b  =>  0-9` | Go directly to window number X |

---

## Navigation and Search

### Enable Scrolling (Copy Mode)

```text
CTRL + b  =>  [
```

Exit copy mode:

```text
q
```

### Search in Terminal History

You must first be in copy mode:

```text
CTRL + b  =>  [
```

Then:

```text
CTRL + r
```

and type what you're looking for.

> [!TIP]
> In copy mode, you can also select text (`SPACE` to start, `ENTER` to copy) and paste it with `CTRL + b  =>  ]`.

---

## Oh my tmux

With [oh-my-tmux](https://github.com/gpakosz/.tmux), you can notably use the mouse:

```text
CTRL + b  =>  m
```

`oh-my-tmux` enhances tmux and adds mouse support, a readable status bar, and many sensible defaults. Its configuration is detailed in [chapter 11 · Advanced Configuration](11_advanced_config.md#oh-my-tmux).

---

## Cheat Sheet

### Command Line

| Command | Action |
|---------|--------|
| `tmux` | New anonymous session |
| `tmux new -s name` | New named session |
| `tmux ls` | List sessions |
| `tmux a -t name` | Join a session |
| `tmux kill-session -t name` | Destroy a session |

### Inside tmux (prefix `CTRL + b`)

| Key | Action | | Key | Action |
|-----|--------|---|-----|--------|
| `d` | Detach | | `c` | New window |
| `s` | List sessions | | `,` | Rename window |
| `$` | Rename session | | `n` / `p` | Next / Previous window |
| `%` | Split vertical | | `w` | List windows |
| `"` | Split horizontal | | `&` | Close window |
| `←↑↓→` | Change pane | | `[` | Scroll / copy mode |
| `z` | Zoom pane | | `?` | **Show all shortcuts** |

---

⬅️ [Previous: 08 · Docker for the Administrator](08_docker_administration.md) · 🏠 [Table of Contents](README.md) · [Next: 10 · Shell Configuration ➡️](10_shell_config.md)
