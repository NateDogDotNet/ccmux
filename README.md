# ccmux

**C**laude **C**ode + t**mux**. Keep Claude Code alive across disconnects.
`ccmux` runs Claude inside persistent tmux sessions with auto-numbered
naming, so you can close your browser, switch computers, or survive a
code-server restart — and pick up exactly where you left off.

Originally built for code-server / VS Code Remote users who want multiple
Claude instances side-by-side per project, but works anywhere you have bash
and tmux.

## Demo

![ccmux demo](demo.gif)

30-second walkthrough of the switching-machines flow. The raw asciinema
recording is also at [`demo.cast`](demo.cast) if you want to replay it
in an asciinema player:

```bash
asciinema play demo.cast
```

## Why

The Claude Code CLI runs as long as its terminal stays open. Close the browser
tab, switch laptops, or let code-server drop a terminal, and the process is
stranded (or worse, killed).

`ccmux` fixes this by running Claude inside a tmux session named after the
current project directory. The session persists independently of any
terminal. You can detach, reconnect from anywhere, and pick up the exact
same Claude with its full context.

## Use case: switching machines

You're working on your desktop, one Claude running in code-server. You lock
the screen and leave for the airport. On the plane, you open your laptop:

```bash
cd ~/myproject
ccmux -T            # detach the desktop's hold on all project sessions
ccmux               # in terminal 1 — grabs the first session back
ccmux               # in terminal 2 — grabs the next
# ...and so on for each Claude you had running
```

Every Claude you had open is back, unchanged, in side-by-side terminals.
The desktop, when you unlock it, just shows "[detached]" — no data lost,
Claude kept running the whole time.

## Requirements

- **bash** 4.0+
- **tmux** 2.0+
- Linux (no macOS / WSL support at this time)
- **claude** on `$PATH` ([install instructions](https://docs.claude.com/claude-code))

## Install

Pick one:

**As a script on your PATH (recommended):**

```bash
git clone https://github.com/NateDogDotNet/ccmux ~/.local/share/ccmux
ln -s ~/.local/share/ccmux/ccmux ~/.local/bin/ccmux
# Ensure ~/.local/bin is in your PATH
```

**Single-file:**

```bash
curl -fsSL https://raw.githubusercontent.com/NateDogDotNet/ccmux/main/ccmux \
  -o ~/.local/bin/ccmux
chmod +x ~/.local/bin/ccmux
```

## Configuration

`ccmux` ships with zero default arguments to `claude`. Configure defaults by
exporting `CCMUX_DEFAULT_ARGS` in your shell rc:

```bash
# ~/.bashrc or ~/.zshrc

# Always run claude with --dangerously-skip-permissions and -c (continue):
export CCMUX_DEFAULT_ARGS="--dangerously-skip-permissions -c"

# Or just continue, no permissions bypass:
export CCMUX_DEFAULT_ARGS="-c"
```

You can override per-invocation with `-p` (plain, no defaults), `-s` (drop
`--dangerously-skip-permissions`), or `-f` (drop `-c`).

Optional session-name override:

```bash
export CCMUX_SESSION_PREFIX="myproject"   # default is basename of $PWD
```

By default, ccmux applies two session-scoped tweaks to every new session so
Claude renders correctly in tmux. Neither touches your `~/.tmux.conf` or other
sessions:

```bash
# Mouse wheel scrolling (disable with CCMUX_MOUSE=off)
export CCMUX_MOUSE=off

# TERM inside the session (empty string skips forcing TERM)
export CCMUX_TERM=tmux-256color
```

Two more opt-in tweaks are available for the features that tmux does not expose
per-session — enabling them modifies your tmux **server** state (persists until
the server restarts). Both default off; opt in knowingly:

```bash
# 24-bit true color — appends ',*:RGB' to terminal-features (idempotent).
export CCMUX_RGB=on

# OSC 52 clipboard passthrough — enables 'set-clipboard on' so mouse-selected
# text reaches your OS clipboard via your terminal emulator.
export CCMUX_CLIPBOARD=on
```

If you'd rather keep ccmux fully non-invasive, leave these off and put the
equivalent lines in your `~/.tmux.conf` instead — see
[Troubleshooting](#troubleshooting) for the snippets.

## Usage

Run `ccmux --help` for the full reference. The quick version:

| Command | What it does |
|---|---|
| `ccmux` | Attach to this project's first unattached session, or create one. |
| `ccmux -n` | Spawn a new session in this project (don't reattach). |
| `ccmux -t` | Take over lowest-numbered session (kicks other clients off). |
| `ccmux -t ProjectName-2` | Take over a specific session by name. |
| `ccmux -T` | Detach every client from every `<project>-N` session. |
| `ccmux -p` | Launch plain `claude`, ignoring `CCMUX_DEFAULT_ARGS`. |
| `ccmux -l` | List all tmux sessions (attached/unattached). |
| `ccmux <name>` | Attach to or create a session with that exact name. |

Short flags stack: `ccmux -nf` = `ccmux --new --fresh`.

### Multiple Claudes side-by-side

Open multiple terminals in your editor (code-server, VS Code, etc.) and run
`ccmux` in each. Each one gets its own tmux session (`myproject-1`,
`myproject-2`, ...). Click between terminals to focus different Claudes.

### Detaching

- `Ctrl-b d` — detach the current tmux session (Claude keeps running).
- Closing the terminal tab/pane — same effect.
- `tmux kill-session -t myproject-1` — permanently end a session.

## FAQ

**Why not just use tmux directly?**
You absolutely can. `ccmux` adds three things on top: (1) per-project session
naming so you don't manage identifiers, (2) auto-numbering for multiple
instances in the same project, (3) a `--takeover` flow for switching between
computers.

**Does my Claude session survive a reboot?**
No. tmux sessions live in memory. Host reboot kills them. Within a session,
browser closes, network drops, and code-server restarts are survivable.

**Why does `--dangerously-skip-permissions` show up in examples?**
It's a common personal default for users who want Claude to operate without
per-command confirmations. It is **not** the shipped default — you opt in via
`CCMUX_DEFAULT_ARGS` if you want it.

**Can I use this with zsh / fish?**
The script itself runs under bash (shebang pins it). You don't need your
interactive shell to be bash — just have bash installed.

**Does this work outside code-server?**
Yes. Any Linux terminal works — native, tmux in ssh, etc. The code-server
use case is what motivated it, but there's nothing code-server-specific
in the code.

## Troubleshooting

### Scrolling

Mouse wheel scrolling works out of the box — ccmux applies `set-option mouse on`
to every session it creates (session-scoped, so your `~/.tmux.conf` and other
tmux sessions are untouched). If the wheel still doesn't scroll, your terminal
may be intercepting it; try holding **Shift** while scrolling to bypass the
terminal and hit tmux's scrollback directly.

Prefer keyboard-only? Disable the default:

```bash
export CCMUX_MOUSE=off
```

**Keyboard copy-mode** is always available regardless of the mouse setting:
press `Ctrl-b [` to enter, navigate with arrow keys / PageUp / PageDown / Vim
keys, then `q` or `Enter` to exit.

### Scrollback buffer is too short

tmux defaults to 2000 lines of history per pane. If you lose output to the top,
bump the limit in your `~/.tmux.conf`:

```
set -g history-limit 100000
```

(This is a pane-creation-time option, so ccmux can't set it per-session after
the fact — it has to go in your tmux config.)

### The view snaps to the bottom while Claude is "Thinking"

Claude Code UI behavior, not tmux. Recent versions support `Ctrl-6` as a freeze
toggle: press once to pause output while you scroll, press again to unfreeze
and catch up.

### Colors look wrong (flat diffs, broken color picker, wrong shades)

ccmux already injects `TERM=tmux-256color` into each session it creates, which
fixes most 256-color rendering issues. If your terminal supports 24-bit true
color (iTerm2, Kitty, Wezterm, Alacritty, VS Code, Ghostty, modern GNOME
Terminal, Windows Terminal, etc.) and you still see flat colors, pick one:

```bash
# Let ccmux apply it (server-scope, idempotent, persists until tmux restart):
export CCMUX_RGB=on
```

```
# Or put it in ~/.tmux.conf yourself and leave CCMUX_RGB off:
set -sa terminal-features ",*:RGB"
```

### Copying Claude's output to the OS clipboard

With `CCMUX_MOUSE=on` (the default), mouse-selected text lands in tmux's
internal buffer — you can paste it back into tmux with `Ctrl-b ]` but your
system clipboard doesn't get it. To route selections to your OS clipboard via
OSC 52, pick one:

```bash
# Let ccmux apply it (server-scope, persists until tmux restart):
export CCMUX_CLIPBOARD=on
```

```
# Or put it in ~/.tmux.conf yourself and leave CCMUX_CLIPBOARD off:
set -g set-clipboard on
```

Most modern terminal emulators honor OSC 52 out of the box; some (e.g. recent
GNOME Terminal) require a preference to be enabled.

### `git push` / `gh` fails after switching machines with `-t`

Classic footgun. When you `ccmux -t` from a new machine, the pane's shell
still has the old host's `$SSH_AUTH_SOCK` — the socket path is dead, so
anything that needs your SSH agent silently fails. Fixes:

```bash
# Inside the reattached pane, refresh the env from tmux:
eval "$(tmux show-environment -s SSH_AUTH_SOCK)"
```

Or open a fresh pane (`Ctrl-b c`) — new panes inherit the updated session env.

The same applies to `$DISPLAY` or any other host-specific env var you rely on.
tmux's `update-environment` option controls which variables get refreshed on
attach; the defaults cover the common ones.

## License

MIT. See [LICENSE](LICENSE).
