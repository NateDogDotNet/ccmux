# ccmux

**C**laude **C**ode + t**mux**. Keep Claude Code alive across disconnects.
`ccmux` runs Claude inside persistent tmux sessions with auto-numbered
naming, so you can close your browser, switch computers, or survive a
code-server restart — and pick up exactly where you left off.

Originally built for code-server / VS Code Remote users who want multiple
Claude instances side-by-side per project, but works anywhere you have bash
and tmux.

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

- **bash** 4.0+ (macOS users: `brew install bash` — the system bash is 3.2)
- **tmux** 2.0+
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
Yes. Any terminal works: native terminal on Linux/macOS, tmux in ssh, etc.
The code-server use case is what motivated it, but there's nothing
code-server-specific in the code.

## License

MIT. See [LICENSE](LICENSE).
