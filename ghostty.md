# Ghostty notes

Living reference for Ghostty (the terminal emulator). Append as you learn.

## Mental model

Ghostty has four nested concepts:

- **Window** — a top-level OS window. macOS treats each as its own window.
- **Tab** — like browser tabs, lives inside one window. Each tab has its own *surface*.
- **Split** — a pane inside a tab. Ghostty calls these "splits" (other emulators say "panes").
- **Surface** — Ghostty's internal name for "one terminal grid running one shell." Every tab and every split is a surface. When config docs talk about a "surface," that's what they mean.

Each surface runs its own shell process — closing the surface kills the shell.

## Config file

- Path: `~/.config/ghostty/config`
- Format: `key = value`, one per line, `#` for comments. Many options can repeat (e.g. multiple `keybind =` lines).
- Reload without restarting: `cmd+shift+,`
- Some options (window appearance, etc.) need a full restart even after reload.

## Useful CLI commands

Run from any shell:

```sh
ghostty +show-config --default --docs   # every option, with docs (very useful)
ghostty +list-themes                    # all bundled themes, live preview as you scroll
ghostty +list-keybinds                  # current keybindings (defaults + your overrides)
ghostty +list-fonts                     # available fonts on this Mac
ghostty +list-actions                   # all action names you can bind to a key
```

`ghostty +show-config --default --docs` is the canonical reference — when you wonder "is there a setting for X?", grep that output.

## Default keybindings (macOS)

Most are stock macOS app conventions, plus terminal-specific ones.

### Tabs

| Keys | Action |
| --- | --- |
| `cmd+t` | New tab |
| `cmd+w` | Close current surface (tab / split / window) |
| `cmd+shift+[` | Previous tab |
| `cmd+shift+]` | Next tab |
| `cmd+1` … `cmd+9` | Jump to tab N |

### Windows

| Keys | Action |
| --- | --- |
| `cmd+n` | New window |
| `cmd+\`` | Cycle to next Ghostty window (custom binding — see below) |
| `cmd+shift+\`` | Cycle backwards |

**Heads-up on `cmd+\``**: Ghostty has *no default* binding for it, and macOS does **not** auto-provide cmd+\` cycling — that only happens for apps that register the action via their Window menu, which Ghostty doesn't. So you have to bind it yourself. Already in this config:

```
keybind = cmd+grave_accent=goto_window:next
keybind = cmd+shift+grave_accent=goto_window:previous
```

If you'd rather have `cmd+\`` cycle *tabs* (browser-style) instead of windows, change `goto_window` to `next_tab` / `previous_tab`.

### Splits (your custom bindings)

| Keys | Action |
| --- | --- |
| `cmd+d` | Split right (vertical divider) |
| `cmd+shift+d` | Split down (horizontal divider) |
| `cmd+alt+←/→/↑/↓` | Move focus to split in that direction |
| `cmd+ctrl+←/→/↑/↓` | Resize split by 20 |
| `cmd+shift+enter` | Toggle zoom of focused split (full tab) |

### Other

| Keys | Action |
| --- | --- |
| `cmd+c` / `cmd+v` | Copy / paste |
| `cmd+f` | Find in scrollback |
| `cmd+k` | Clear screen but keep scrollback (mimics iTerm `cmd+R`) |
| `cmd++` / `cmd+-` / `cmd+0` | Font bigger / smaller / reset |
| `cmd+shift+,` | Reload config |
| `cmd+shift+\`` (global) | Toggle Quick Terminal — see below |

## Quick Terminal

A floating terminal that slides in from a screen edge with one keypress. Works *system-wide*, even when Ghostty isn't focused (requires Accessibility permission the first time).

- Hotkey: `cmd+shift+\``
- Slide direction: `quick-terminal-position` (top / bottom / left / right / center)
- Animation speed: `quick-terminal-animation-duration` (seconds, 0 = instant)

Use cases: quick one-liner without context switch, tail logs, scratchpad. iTerm2 called this the "hotkey window."

## Shell integration

Auto-injected for zsh and bash on macOS. No action needed in `.zshrc`. Gives you:

- **CWD inheritance**: new tabs/splits open in the same directory as the current one.
- **Prompt marking**: Ghostty knows where prompts are, so `cmd+↑/↓` jumps between prompts.
- **`cmd+click` on file paths** in terminal output to open them.
- **`cmd+click` on URLs** to open in browser.

Disable globally: `shell-integration = none`. Disable specific features via `shell-integration-features = no-cursor,no-sudo,no-title`.

## SSH and terminfo

Ghostty advertises itself as `TERM=xterm-ghostty`. Remote machines without that terminfo entry will fail with `missing or unsuitable terminal: xterm-ghostty`.

Two fixes:

1. **Install Ghostty's terminfo on the remote** (one-off):
   ```sh
   infocmp -x | ssh REMOTE -- tic -x -
   ```
   After this, ssh + tmux + vim all work with full Ghostty capabilities (true color, etc).

2. **Override TERM in the remote command** (per-invocation):
   ```sh
   ssh REMOTE -t "TERM=xterm-256color tmux attach -t dev"
   ```
   Loses Ghostty-specific terminfo features but works on any remote.

The `gdevvm` alias uses option 2.

## Themes

`ghostty +list-themes` opens a scrollable preview — your terminal recolors live as you arrow through. Press Enter to see the name to put in `theme = <name>`.

Auto-switch with macOS dark/light mode:

```
theme = light:Builtin Solarized Light,dark:TokyoNight
```

## Fonts

Without `font-family` set, Ghostty uses bundled JetBrains Mono Nerd Font (icons work in eza, starship, lazygit, etc).

Gotcha: `font-family =` lines **append as fallbacks**, they don't replace. To swap the primary, you must reset first:

```
font-family = ""
font-family = "Berkeley Mono"
```

Adding without the empty line just adds it as a fallback for missing glyphs.

Disable ligatures (e.g. `==` rendering as ⩵):

```
font-feature = -calt
font-feature = -liga
```

## Useful options worth knowing

| Option | What it does |
| --- | --- |
| `confirm-close-surface = false` | No "are you sure?" prompt on close |
| `copy-on-select = clipboard` | Selection auto-copies, like iTerm2 |
| `scrollback-limit = 100000000` | Bytes per surface — 100 MB ≈ unlimited |
| `macos-option-as-alt = true` | Option key acts as Alt/Meta — needed for `opt+←/→` word jumping in zsh |
| `macos-titlebar-style = tabs` | Compact tab bar in titlebar |
| `background-opacity = 0.92` | Transparency (0.0–1.0) |
| `background-blur-radius = 20` | macOS background blur behind transparency |
| `cursor-style = block` / `bar` / `underline` | Cursor shape |
| `cursor-style-blink = false` | Steady cursor |

## Gotchas

- **`open -na Ghostty.app --args -e <cmd>`** spawns a *new* Ghostty instance, not a new window of the existing one. Ghostty has no AppleScript / IPC on macOS, so you can't ask the running instance to open a tab from the outside. (This is why `gdevvm` opens one window with a tmux session inside, instead of six separate windows.)
- **Quick Terminal needs Accessibility permission**. First time you press the hotkey, macOS prompts. If it silently doesn't work later, check System Settings → Privacy & Security → Accessibility.
- **Reloading config (`cmd+shift+,`) doesn't reload everything** — window appearance and a few startup-only options need a full restart.

## References

- Docs: https://ghostty.org/docs
- Config reference: `ghostty +show-config --default --docs`
- Action reference: `ghostty +list-actions`
