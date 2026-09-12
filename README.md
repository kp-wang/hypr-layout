# hypr-layout

Save your Hyprland session and get it back — windows, workspaces, tab groups **and the arrangement they were in**.

Three commands:

```bash
hypr-layout save       # snapshot what is on screen now
hypr-layout restore    # put everything back
hypr-layout show       # print the saved snapshot
```

One Python file, standard library only, no build step, no plugin ABI to chase. It talks to Hyprland's IPC sockets — the command socket for dispatches and the event socket to learn the moment a window opens.

## Why this exists

Restoring a session sounds like "launch the same apps again". In practice four things quietly break it, and each one cost real debugging time:

1. **Placement by window rule fails for apps that hand off.** `hl.dsp.exec_cmd("[workspace 2] firefox")` does nothing when Firefox passes the window to its already-running instance — the process that creates the window never saw the rule. hypr-layout places windows **by address**, from the `openwindow` event, which cannot miss.
2. **Group dispatchers act on "the active window".** A focus that silently fails — for example on another workspace — means the next group command hits an *unrelated* window. A restore can therefore scramble a layout instead of rebuilding it. hypr-layout verifies the focus (reads `activewindow` back) before every such dispatch.
3. **`keyword` doesn't work under the Lua config parser** (`keyword can't work with non-legacy parsers. Use eval.`), and `hl.dsp.keyword` doesn't exist. Runtime options are set with `eval hl.config({ group = { auto_group = false } })`.
4. **A dwindle tree cannot be set, but it can be built.** With a window focused, `hl.dsp.layout("preselect r")` makes the next window split that window's rect on that side. hypr-layout recovers the tree from the rectangles in the snapshot and re-inserts the tiles in tree order — which is how the window that belongs bottom-right goes back to bottom-right.

## Requirements

- Hyprland, tested against **0.56.2** (Omarchy's Lua-config build).
- Python 3.9+ — nothing to install, no third-party packages.

## Install

```bash
mkdir -p ~/.local/bin
curl -fsSL https://raw.githubusercontent.com/leftydevkit/hypr-layout/main/hypr-layout \
  -o ~/.local/bin/hypr-layout
chmod +x ~/.local/bin/hypr-layout
```

Make sure `~/.local/bin` is on your `PATH`.

## Usage

```bash
hypr-layout save                    # snapshot -> ~/.config/hypr/layout.json
hypr-layout restore                 # replay it
hypr-layout restore --dry-run       # show what would happen, change nothing
hypr-layout show                    # print the snapshot
```

`restore` options:

| flag | effect |
|---|---|
| `--workspace N` | replay every window onto workspace `N` (name or number) |
| `--workspace-map 'firefox=2,obs=3'` | per-class workspace override; beats `--workspace` |
| `--except CLASS` | skip a class (repeatable) |
| `--force` | always launch, even when a window of that class is already open |
| `--no-arrange` | keep workspaces and groups, skip rebuilding the tiling arrangement |
| `--no-groups` | do not rebuild window groups |
| `--no-verify` | skip the final snapshot comparison |
| `--delay S` | stagger launches by `S` seconds (default `0`: all at once) |
| `-f FILE` | use a different layout file |
| `--floating` (on `save`) | also capture floating windows, with their size and position |

### Launch-or-focus

`restore` never opens a second copy of something already open. If a window of that class exists, it is focused and moved instead — otherwise Firefox, Zed and OBS would simply open new windows every time you ran it. Candidates are scored on title, workspace and group membership, because every terminal shares a class and grabbing the wrong one drags an unrelated window across your layout. Use `--force` when you really do want new copies.

### What a restore reports

```
replaying 4 windows
   1. ws3      com.obsproject.Studio (focused the window already open)
   2. ws3      foot               (focused the window already open)
   3. ws3      firefox            (focused the window already open)
   4. ws3      dev.zed.Zed        (focused the window already open)
  OK   group of 2: foot, com.obsproject.Studio (already correct)
  OK   group of 1: dev.zed.Zed (already correct)
  ok    1. ws3      com.obsproject.Studio [tab 2]
  ok    2. ws3      foot               [tab 1*]
  ok    3. ws3      firefox
  ok    4. ws3      dev.zed.Zed        [tab 1*]
4/4 windows placed correctly  (0.6s)
snapshot matches the live session
```

Every entry is checked against the compositor afterwards — workspace, tiling, tab number, which tab is showing, and the arrangement. A window in the wrong place is reported with the rectangle it got and the one it should have had, so "it looks fine" is never the evidence.

## Omarchy

Bindings (`~/.config/hypr/bindings.lua`):

```lua
o.bind("SUPER + ALT + W", "Restore work layout", "/home/USER/.local/bin/hypr-layout restore")
o.bind("SUPER + SHIFT + ALT + W", "Save work layout", "/home/USER/.local/bin/hypr-layout save")
```

Replay at login (`~/.config/hypr/autostart.lua`):

```lua
o.launch_on_start("bash -c 'sleep 8; echo \"=== $(date -Is) ===\"; ~/.local/bin/hypr-layout restore --no-verify' >>\"$HOME/.cache/hypr-layout-login.log\" 2>&1")
```

The sleep lets the session settle, and the log is there because a login-time replay has nowhere to print.

## How it works

- **One socket, not `hyprctl`.** Every dispatch goes over the command socket — about 0.1 ms against roughly 15 ms for spawning `hyprctl` per call.
- **Apps launch in parallel**, and each window is placed the instant it is announced on the event socket, instead of polling and sleeping between launches.
- **Placement is explicit and by address** — never inferred from a window rule.
- **Everything that acts on the active window is preceded by a verified focus**, because that is the one mistake that damages a live layout instead of failing cleanly.
- **`group:auto_group` is disabled for the whole replay** (including the arrangement pass) and restored afterwards, so windows opened mid-replay cannot join a group they do not belong to.
- **Arrangement is rebuilt** by parking the tiles on a hidden workspace (`special:hypr-layout`), deriving the dwindle tree from the saved rectangles, and re-inserting them in tree order with `layout preselect` + move. Anything that fails to land is moved back — a window is never left parked.

## Limitations

- The arrangement builder assumes the tiled windows in a snapshot form a clean BSP. A snapshot taken with overlapping or partially off-screen windows falls back to members-only restore (`--no-arrange` behaviour).
- Split ratios are re-applied for splits that are not near 50/50; near-half splits keep the compositor's default.
- Floating windows are only captured with `save --floating`.
- Verified on one Hyprland version (0.56.2). The IPC commands used are long-standing, but newer builds may rename dispatchers — if a dispatch fails the tool reports it rather than failing silently.

## License

MIT — see [LICENSE](LICENSE).
