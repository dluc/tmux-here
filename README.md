# tmux-here

Four small terminal multiplexer utilities — two for tmux, two for zellij.
No config, no dependencies beyond bash and your multiplexer of choice.

**tmux-here** / **zell-here** — create or attach to a session named after your
current directory:

```
cd /srv/my-app
tmux-here          # → tmux session: "srv_my-app"
zell-here          # → zellij session: "srv_my-app"
```

**tmux-continue** / **zell-continue** — pick an existing session from a numbered
menu and attach:

```
tmux-continue      # → lists all tmux sessions, you pick one
zell-continue      # → lists all zellij sessions, you pick one
```

## Why this exists

Every tmux user eventually writes a 4-line script that does `tmux new -As $(pwd)`.
Every zellij user writes the equivalent `zellij attach --create $(pwd)`.
Then they discover edge cases:

- The multiplexer crashes and the terminal is stuck in raw mode with no echo
- A SIGHUP from a dying server kills the shell that launched it
- It breaks inside an existing session (nesting)
- Session names with dots or slashes get rejected
- It doesn't work on WSL because of socket directory permissions
- Zellij hangs on session names longer than 36 characters

These scripts are that 4-liner, rewritten after actually hitting all of those
problems. Each is ~150 lines because error handling is 90% of the work.

## What it does

### tmux-here / zell-here

1. Derives a session name from `$PWD` (sanitized to `[a-zA-Z0-9_-]`)
2. Creates the session if it doesn't exist, attaches if it does
3. Protects the hosting shell from crashes (signal traps on HUP/PIPE)
4. Restores terminal state if the multiplexer dies (`stty sane`, cursor, alt-screen)
5. Reports errors honestly instead of swallowing them

### tmux-continue / zell-continue

1. Lists all running sessions in a numbered, column-aligned menu
2. Prompts for a selection with input validation
3. Attaches to the chosen session
4. Handles edge cases: no sessions, non-interactive stdin, nested sessions

## Install

```bash
# Copy the scripts somewhere in your PATH

# tmux utilities:
for script in tmux-here tmux-continue; do
    curl -fsSL "https://raw.githubusercontent.com/dluc/tmux-here/main/$script" \
        -o ~/.local/bin/"$script"
    chmod +x ~/.local/bin/"$script"
done

# zellij utilities:
for script in zell-here zell-continue; do
    curl -fsSL "https://raw.githubusercontent.com/dluc/tmux-here/main/$script" \
        -o ~/.local/bin/"$script"
    chmod +x ~/.local/bin/"$script"
done
```

Or clone and symlink:

```bash
git clone https://github.com/dluc/tmux-here.git
for script in tmux-here tmux-continue zell-here zell-continue; do
    ln -s "$(pwd)/tmux-here/$script" ~/.local/bin/"$script"
done
```

### Requirements

- **bash** 3.2+ (macOS default, any Linux, WSL)
- **tmux** 1.8+ for tmux-here / tmux-continue (for the `-A` flag; any version from the last decade)
- **zellij** 0.40+ for zell-here / zell-continue (for `list-sessions --short`)

## Usage

### tmux-here / zell-here

```bash
tmux-here            # create/attach tmux session named after $PWD
tmux-here -f         # allow nesting inside an existing tmux session
tmux-here --help     # usage info

zell-here            # create/attach zellij session named after $PWD
zell-here -f         # allow nesting inside an existing zellij session
zell-here --help     # usage info
```

### tmux-continue / zell-continue

```bash
tmux-continue        # show menu, pick a tmux session
tmux-continue -f     # allow picking from inside tmux (nested attach)
tmux-continue --help # usage info

zell-continue        # show menu, pick a zellij session
zell-continue -f     # allow picking from inside zellij (nested attach)
zell-continue --help # usage info
```

### Typical workflow

```bash
# Terminal 1: working on the API
cd ~/projects/api
tmux-here            # or: zell-here

# Terminal 2: resume exactly where you left off
cd ~/projects/api
tmux-here            # → attaches to the existing session
```

### Session naming

| Directory | Session name |
|---|---|
| `/home/user/projects/my-app` | `home_user_projects_my-app` |
| `/tmp/test` | `tmp_test` |
| `/` | `root` |

Non-alphanumeric characters (except `_` and `-`) become `_`.

tmux-here truncates names to 128 characters (tmux limit is 256; shorter avoids clutter).
zell-here truncates to 35 characters due to Unix socket path limits and a
[known zellij bug](https://github.com/zellij-org/zellij/issues/4627) with longer names.

## Safety features

All four scripts share the same safety philosophy:

**Signal isolation.** When a multiplexer server crashes or is OOM-killed, it sends
SIGHUP/SIGPIPE to attached clients. Without protection, these signals propagate
to the shell that launched the script — killing your terminal. The scripts trap
these signals so a crash exits cleanly instead of cascading.

**Terminal restoration.** A crash can leave the terminal in raw mode (no echo,
no line editing), stuck in the alternate screen buffer, or with a hidden cursor.
The scripts run `stty sane` and reset the terminal state on any non-zero exit.
These are no-ops on a healthy terminal.

**Nested session detection.** Running inside an existing session would create a
confusing nested session. The scripts detect `$TMUX` / `$ZELLIJ` and refuse,
with `-f` to override.

**Source guard.** If someone does `source tmux-here` instead of executing it, every
`exit` would kill their shell. The scripts detect this and refuse with `return 1`.

**No stderr suppression.** Error messages from tmux/zellij (bad config, wrong `$TERM`,
socket errors) are passed through to the user, not swallowed.

### What it shows

tmux-continue:

```
3 tmux sessions found:

 1) api-server        2 windows (created Mon Feb 24 09:14:01 2026)
 2) dotfiles          1 windows (created Mon Feb 24 11:30:45 2026)
 3) projects_my-app   3 windows (created Tue Feb 25 08:02:12 2026)

Select session [1-3] (q to quit):
```

zell-continue:

```
3 zellij sessions found:

 1) api-server        [Created 2h 5m ago]
 2) dotfiles          [Created 5h 30m ago] (ATTACHED)
 3) projects_my-app   [Created 1d 2h ago]

Select session [1-3] (q to quit):
```

Session names are left-aligned and the detail column adjusts to the longest
name, so the output stays readable regardless of naming conventions.

### continue-specific safety

- **EOF handling.** Ctrl-D exits cleanly instead of looping.
- **Non-interactive guard.** Refuses when stdin is not a terminal (piped input),
  with a hint to use `tmux attach -t` / `zellij attach` directly.
- **Input validation.** Rejects non-numeric input and out-of-range numbers.
  Leading zeros are normalized to prevent octal interpretation in bash arithmetic.

## Limitations

- **Session name collisions.** Two directories that sanitize to the same string
  will share a session. For tmux-here this means paths differing only past 128
  characters (extremely unlikely). For zell-here the limit is 35 characters,
  so deeply nested paths are more likely to collide.

- **Symlink paths.** Two terminals reaching the same physical directory via different
  symlinks get different session names. This is arguably correct (you chose the path)
  but worth knowing.

- **No project picker.** The `here` scripts are deliberately a "session for the
  directory I'm already in" tool, not a session picker. The `continue` scripts
  cover the "which session do I reconnect to?" case. For fuzzy project finding,
  see Alternatives.

- **No layout management.** The `here` scripts create a single-window session.
  They don't define pane layouts, run startup commands, or restore previous
  window arrangements.

- **No session persistence across reboot.** These scripts create sessions; they
  don't save or restore them. If your multiplexer server restarts, sessions are gone.

## Alternatives

The "session per directory" pattern is well-established in the tmux ecosystem.
Zellij is newer, so the wrapper ecosystem is smaller — its built-in session
management (`attach --create`, layout files) covers more ground out of the box.

### Lightweight scripts (same category as tmux-here)

| Tool | What it adds | Tradeoff |
|---|---|---|
| [**tat**](https://thoughtbot.com/blog/tmux-only-for-long-running-processes) (thoughtbot) | ~15 lines. Uses `basename $PWD` for the name. | Simpler naming, no crash safety, no terminal recovery. |
| [**tmux-session**](https://bytes.zone/posts/tmux-session/) (Brian Hicks) | Walks up to the git root to name sessions. | Smarter naming for git repos. No crash handling. |
| **tmux-here** | Full-path naming, crash isolation, terminal recovery, WSL support. | More code (~150 lines) for edge cases you may never hit. |

All three converge on the same core: `tmux new-session -As <name>`. The differences
are in naming strategy and how much failure handling you want.

### Fuzzy-finder tools (different category)

| Tool | What it does |
|---|---|
| [**tmux-sessionizer**](https://github.com/ThePrimeagen/tmux-sessionizer) (ThePrimeagen) | Scans project directories, pipes to fzf, creates/switches sessions. A **project picker**, not a "name after pwd" script. |
| [**t**](https://github.com/joshmedeski/t-smart-tmux-session-manager) (Josh Medeski) | Like tmux-sessionizer but integrates zoxide for frecency-based sorting. |

These solve a different problem: "which project do I want to work on?" vs
the `here` scripts' "give me a session for where I already am."

### Full session managers (different category)

| Tool | What it does |
|---|---|
| [**tmuxinator**](https://github.com/tmuxinator/tmuxinator) | YAML-defined layouts: windows, panes, startup commands. Requires Ruby. |
| [**tmuxp**](https://tmuxp.git-pull.com/) | Same concept in Python. Supports `.tmuxp.yaml` files in project roots. |
| [**smug**](https://github.com/ivaaaan/smug) | Same concept as a single Go binary (no runtime deps). |

These are for when you want the same 3-pane layout every time you open a project.
The `here` scripts don't manage layouts. For zellij, the equivalent is built-in:
[layout files](https://zellij.dev/documentation/layouts) in `.kdl` format.

### Crash recovery (orthogonal, use alongside tmux)

| Tool | What it does |
|---|---|
| [**tmux-resurrect**](https://github.com/tmux-plugins/tmux-resurrect) | Saves/restores tmux sessions (layouts, directories, programs) across server restarts. Manual save/restore. |
| [**tmux-continuum**](https://github.com/tmux-plugins/tmux-continuum) | Auto-saves resurrect snapshots every 15 minutes. Auto-restores on tmux start. |

These are **complementary** to tmux-here, not replacements. tmux-here creates
sessions; resurrect/continuum persist them across reboots. Zellij has built-in
session serialization that partially covers this use case.

## License

MIT
