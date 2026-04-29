# tmux notes

Living reference for tmux. Append as you learn.

## Mental model

- **Server**: one background process per user. Hosts everything below. Survives ssh disconnects.
- **Session**: a named workspace owned by the server (e.g. `dev`, `agentA`). Has its own set of windows. You *attach* to a session; the work keeps running when you detach.
- **Window**: like a browser tab inside a session. Has a name and a number. The bar at the bottom of tmux shows `1:lpu 2:agentA* 3:agentB ...` — those are windows of the current session.
- **Pane**: a split inside one window. A window can be split into multiple panes (vertical / horizontal). Each pane runs its own shell.

Hierarchy: `server → sessions → windows → panes → shell`.

## The prefix key

All tmux shortcuts start with the **prefix**, then a key. Default prefix: `Ctrl-b`.

Press them in sequence (release prefix, then press the next key) — not as a chord.

## Windows (the tabs)

| Keys | Action |
| --- | --- |
| `Ctrl-b 1` … `Ctrl-b 9` | Jump to window N |
| `Ctrl-b n` | Next window |
| `Ctrl-b p` | Previous window |
| `Ctrl-b w` | Interactive picker (arrows + Enter) |
| `Ctrl-b c` | Create new window |
| `Ctrl-b ,` | Rename current window |
| `Ctrl-b &` | Kill current window (asks to confirm) |
| `Ctrl-b l` | Toggle to last-used window |

## Panes (splits within one window)

| Keys | Action |
| --- | --- |
| `Ctrl-b %` | Split left/right |
| `Ctrl-b "` | Split top/bottom |
| `Ctrl-b ←/→/↑/↓` | Move between panes |
| `Ctrl-b o` | Cycle through panes |
| `Ctrl-b z` | Zoom current pane to fullscreen (toggle) |
| `Ctrl-b x` | Kill current pane |
| `Ctrl-b Space` | Cycle through layouts |
| `Ctrl-b {` / `Ctrl-b }` | Swap pane with previous / next |

## Sessions

| Keys | Action |
| --- | --- |
| `Ctrl-b d` | Detach. Tmux keeps running on the host. Reattach with `tmux attach` or `tmux a -t <name>`. |
| `Ctrl-b s` | List/switch sessions interactively |
| `Ctrl-b $` | Rename current session |
| `Ctrl-b (` / `Ctrl-b )` | Previous / next session |

## Scrollback / copy mode

| Keys | Action |
| --- | --- |
| `Ctrl-b [` | Enter copy mode (scroll, search) |
| (in copy mode) `↑/↓ PageUp/PageDown` | Move |
| (in copy mode) `/` then text | Search forward |
| (in copy mode) `?` then text | Search backward |
| (in copy mode) `Space` | Start selection |
| (in copy mode) `Enter` | Copy selection to tmux buffer |
| (in copy mode) `q` | Exit copy mode |
| `Ctrl-b ]` | Paste tmux buffer |

## Other

| Keys | Action |
| --- | --- |
| `Ctrl-b ?` | Show full keybinding cheat sheet (`q` to exit) |
| `Ctrl-b :` | Command prompt (e.g. `:kill-session`, `:new-window -n foo`) |
| `Ctrl-b t` | Show a big clock in the current pane (silly but built-in) |

## Useful CLI commands (run from a regular shell)

```sh
tmux ls                       # list all sessions on the server
tmux new -s NAME              # create + attach a new session named NAME
tmux a                        # attach to the most recent session
tmux a -t NAME                # attach to a specific session
tmux kill-session -t NAME     # kill one session
tmux kill-server              # nuke the whole tmux server (all sessions die)
```

`-A` is handy for "attach if exists, create if not":

```sh
tmux new-session -As NAME     # idempotent attach-or-create
```

## Habits worth forming

- **Detach, don't close the window.** `Ctrl-b d` leaves tmux running on the remote. Closing the terminal window kills the ssh connection but tmux survives — reattach with `gdevvm` or `tmux a -t dev`.
- **`Ctrl-b z` is your friend** when one pane needs the full screen for a moment, then again to un-zoom.
- **One tmux session per project / context.** Don't pile everything into one session — windows within a session should be related work.

## Gotchas

- **Nested tmux**: if you `tmux attach` inside an already-running tmux, the inner session shares the same prefix. To send a key to the *inner* tmux, press the prefix twice: `Ctrl-b Ctrl-b <key>`.
- **TERM mismatch**: if tmux refuses to start with `missing or unsuitable terminal`, the remote machine is missing the terminfo entry for whatever your local terminal advertises. Either install the terminfo (e.g. `infocmp -x | ssh remote -- tic -x -`) or run with a fallback `TERM=xterm-256color`.
