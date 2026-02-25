# tmux-here

Two small tmux utilities. No config, no dependencies beyond tmux and bash.

**tmux-here** — create or attach to a session named after your current directory:

```
cd ~/projects/my-app
tmux-here
# → tmux session: "projects__my-app"
```

**tmux-continue** — pick an existing session from a numbered menu and attach:

```
tmux-continue
# → lists all sessions, you pick one
```

## Why this exists

Every tmux user eventually writes a 4-line script that does `tmux new -As $(pwd)`.
Then they discover edge cases:

- tmux crashes and the terminal is stuck in raw mode with no echo
- A SIGHUP from a dying tmux server kills the shell that launched it
- It breaks inside an existing tmux session
- Session names with dots or slashes get rejected
- It doesn't work on WSL because of socket directory permissions

This script is that 4-liner, rewritten after actually hitting all of those problems.
It's ~120 lines because error handling is 90% of the work.

## What it does

1. Derives a session name from `$PWD` (sanitized to `[a-zA-Z0-9_-]`)
2. Creates the session if it doesn't exist, attaches if it does (`tmux new-session -As`)
3. Protects the hosting shell from tmux crashes (signal traps on HUP/PIPE)
4. Restores terminal state if tmux dies mid-operation (`stty sane`, cursor, alt-screen)
5. Reports errors honestly instead of swallowing them

## Install

```bash
# Copy the scripts somewhere in your PATH
for script in tmux-here tmux-continue; do
    curl -fsSL "https://raw.githubusercontent.com/dluc/tmux-here/main/$script" \
        -o ~/.local/bin/"$script"
    chmod +x ~/.local/bin/"$script"
done
```

Or clone and symlink:

```bash
git clone https://github.com/dluc/tmux-here.git
ln -s "$(pwd)/tmux-here/tmux-here" ~/.local/bin/tmux-here
ln -s "$(pwd)/tmux-here/tmux-continue" ~/.local/bin/tmux-continue
```

### Requirements

- **bash** 3.2+ (macOS default, any Linux, WSL)
- **tmux** 1.8+ (for the `-A` flag; any version from the last decade)

## Usage

```bash
tmux-here            # create/attach session named after $PWD
tmux-here -f         # allow nesting inside an existing tmux session
tmux-here --help     # usage info
```

### Typical workflow

```bash
# Terminal 1: working on the API
cd ~/projects/api
tmux-here

# Terminal 2: resume exactly where you left off
cd ~/projects/api
tmux-here
# → attaches to the existing "projects__api" session
```

### Session naming

| Directory | Session name |
|---|---|
| `/home/user/projects/my-app` | `home__user__projects__my-app` |
| `/tmp/test` | `tmp__test` |
| `/` | `root` |

Non-alphanumeric characters (except `_` and `-`) become `_`.
Names are truncated to 128 characters.

## Safety features

**Signal isolation.** When a tmux server crashes or is OOM-killed, it sends
SIGHUP/SIGPIPE to attached clients. Without protection, these signals propagate
to the shell that launched the script — killing your terminal. tmux-here traps
these signals so a crash exits cleanly instead of cascading.

**Terminal restoration.** A tmux crash can leave the terminal in raw mode (no echo,
no line editing), stuck in the alternate screen buffer, or with a hidden cursor.
tmux-here runs `stty sane` and resets the terminal state on any non-zero exit.
These are no-ops on a healthy terminal.

**Nested session detection.** Running tmux-here inside tmux would create a confusing
nested session. The script detects `$TMUX` and refuses, with `-f` to override.

**Source guard.** If someone does `source tmux-here` instead of executing it, every
`exit` would kill their shell. The script detects this and refuses with `return 1`.

**No stderr suppression.** tmux's own error messages (bad config, wrong `$TERM`,
socket errors) are passed through to the user, not swallowed.

## tmux-continue

An interactive session picker. Lists every running tmux session in a numbered
menu and attaches to the one you choose.

```bash
tmux-continue            # show menu, pick a session
tmux-continue -f         # allow picking from inside tmux (nested attach)
tmux-continue --help     # usage info
```

### What it shows

```
3 tmux sessions found:

 1) api-server           2 windows (created Mon Feb 24 09:14:01 2026)
 2) dotfiles             1 windows (created Mon Feb 24 11:30:45 2026)
 3) projects__my-app     3 windows (created Tue Feb 25 08:02:12 2026)

Select session [1-3] (q to quit):
```

Session names are left-aligned and the detail column adjusts to the longest
name, so the output stays readable regardless of naming conventions.

### Safety

Same philosophy as tmux-here:

- **Nested session detection.** Refuses when `$TMUX` is set, with `-f` to override.
- **EOF handling.** Ctrl-D exits cleanly instead of looping.
- **Non-interactive guard.** Refuses when stdin is not a terminal (piped input),
  with a hint to use `tmux attach -t` directly.
- **Input validation.** Rejects non-numeric input and out-of-range numbers.
  Leading zeros are normalized to prevent octal interpretation in bash arithmetic.

## Limitations

- **Session name collisions.** Two directories that differ only after 128 characters
  (post-sanitization) will map to the same session. Unlikely in practice but
  theoretically possible with very long paths.

- **Symlink paths.** Two terminals reaching the same physical directory via different
  symlinks get different session names. This is arguably correct (you chose the path)
  but worth knowing.

- **No project picker.** tmux-here is deliberately a "session for the directory
  I'm already in" tool, not a session picker. tmux-continue covers the "which
  session do I reconnect to?" case. For fuzzy project finding, see Alternatives.

- **No layout management.** tmux-here creates a single-window session. It doesn't
  define pane layouts, run startup commands, or restore previous window arrangements.

- **No session persistence across reboot.** tmux-here creates sessions; it doesn't
  save or restore them. If your tmux server restarts, sessions are gone.

## Alternatives

The "tmux session per directory" pattern is well-established. Here's how tmux-here
fits in the landscape:

### Lightweight scripts (same category)

| Tool | What it adds | Tradeoff |
|---|---|---|
| [**tat**](https://thoughtbot.com/blog/tmux-only-for-long-running-processes) (thoughtbot) | ~15 lines. Uses `basename $PWD` for the name. | Simpler naming, no crash safety, no terminal recovery. |
| [**tmux-session**](https://bytes.zone/posts/tmux-session/) (Brian Hicks) | Walks up to the git root to name sessions. | Smarter naming for git repos. No crash handling. |
| **tmux-here** | Full-path naming, crash isolation, terminal recovery, WSL support. | More code (~120 lines) for edge cases you may never hit. |

All three converge on the same core: `tmux new-session -As <name>`. The differences
are in naming strategy and how much failure handling you want.

### Fuzzy-finder tools (different category)

| Tool | What it does |
|---|---|
| [**tmux-sessionizer**](https://github.com/ThePrimeagen/tmux-sessionizer) (ThePrimeagen) | Scans project directories, pipes to fzf, creates/switches sessions. A **project picker**, not a "name after pwd" script. |
| [**t**](https://github.com/joshmedeski/t-smart-tmux-session-manager) (Josh Medeski) | Like tmux-sessionizer but integrates zoxide for frecency-based sorting. |

These solve a different problem: "which project do I want to work on?" vs
tmux-here's "give me a session for where I already am."

### Full session managers (different category)

| Tool | What it does |
|---|---|
| [**tmuxinator**](https://github.com/tmuxinator/tmuxinator) | YAML-defined layouts: windows, panes, startup commands. Requires Ruby. |
| [**tmuxp**](https://tmuxp.git-pull.com/) | Same concept in Python. Supports `.tmuxp.yaml` files in project roots. |
| [**smug**](https://github.com/ivaaaan/smug) | Same concept as a single Go binary (no runtime deps). |

These are for when you want the same 3-pane layout every time you open a project.
tmux-here doesn't manage layouts.

### Crash recovery (orthogonal, use alongside)

| Tool | What it does |
|---|---|
| [**tmux-resurrect**](https://github.com/tmux-plugins/tmux-resurrect) | Saves/restores tmux sessions (layouts, directories, programs) across server restarts. Manual save/restore. |
| [**tmux-continuum**](https://github.com/tmux-plugins/tmux-continuum) | Auto-saves resurrect snapshots every 15 minutes. Auto-restores on tmux start. |

These are **complementary** to tmux-here, not replacements. tmux-here creates
sessions; resurrect/continuum persist them across reboots. You can use both.

## License

MIT
