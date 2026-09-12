# hypr-layout

![two tiles drift away, one command brings the session back](docs/demo.gif)

Save your Hyprland session and get it back — windows, workspaces, tab groups **and the arrangement they were in**.

```bash
hypr-layout save               # snapshot what is on screen now
hypr-layout restore            # put everything back
hypr-layout restore --pick     # pick a layout and a workspace from a menu
hypr-layout save --name work   # keep more than one layout, side by side
hypr-layout layouts            # what is saved, and how many earlier saves exist
hypr-layout show               # print the saved snapshot
```

One Python file, standard library only, no build step, no plugin ABI to chase. It talks to Hyprland's IPC sockets — the command socket for dispatches and the event socket to learn the moment a window opens.

## Why this exists

Restoring a session sounds like "launch the same apps again". In practice four things quietly break it, and each one cost real debugging time:

1. **Placement by window rule fails for apps that hand off.** `hl.dsp.exec_cmd("[workspace 2] firefox")` does nothing when Firefox passes the window to its already-running instance — the process that creates the window never saw the rule. hypr-layout places windows **by address**, from the `openwindow` event, which cannot miss.
2. **Group dispatchers act on "the active window".** A focus that silently fails — for example on another workspace — means the next group command hits an *unrelated* window. A restore can therefore scramble a layout instead of rebuilding it. hypr-layout verifies the focus (reads `activewindow` back) before every such dispatch.
3. **`keyword` doesn't work under the Lua config parser** (`keyword can't work with non-legacy parsers. Use eval.`), and `hl.dsp.keyword` doesn't exist. Runtime options are set with `eval hl.config({ group = { auto_group = false } })`.
4. **A dwindle tree cannot be set, but it can be built.** With a window focused, `hl.dsp.layout("preselect r")` makes the next window split that window's rect on that side. hypr-layout recovers the tree from the rectangles in the snapshot and re-inserts the tiles in tree order — which is how the window that belongs bottom-right goes back to bottom-right.

## Requirements

- **Hyprland with the Lua dispatch API** — the `hl.dsp.*` dispatchers, as shipped by Omarchy
  (tested on 0.56.2). On a classic-config Hyprland build those commands do not parse and
  nothing will move. Check before installing:

  ```bash
  hyprctl dispatch 'hl.dsp.window.move({ workspace = "2" })'   # "ok" means you are set
  ```

- Python 3 — nothing to install, no third-party packages.

Note that one dispatcher in particular is not dependable: `hl.dsp.workspace` answered `ok`
and switched workspaces early in this tool's development, then later became a *table* and
error'd out, in the same session. hypr-layout therefore never switches workspaces —
`hl.dsp.focus` by window address crosses workspaces on its own — and every action it takes
is verified by reading the compositor back rather than trusting an `ok`.

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
hypr-layout save                    # snapshot -> ~/.config/hypr/layout.json  (do this first)
hypr-layout restore                 # replay it
hypr-layout restore --dry-run       # show what would happen, change nothing
hypr-layout show                    # print the snapshot
```

`restore` options:

| flag | effect |
|---|---|
| `--pick` | choose the workspace (and the layout, when more than one is saved) from a menu |
| `--history` | choose one of the last few saves of this layout instead |
| `--spill WS` | move windows on this layout's workspaces that are not part of it to `WS` first |
| `--name NAME` | restore a layout saved with `save --name` |
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

## More than one layout

The plain `save`/`restore` pair is wired to a single file, `~/.config/hypr/layout.json`.
`--name` keeps as many others as you like in `~/.config/hypr/layouts/`:

```bash
hypr-layout save --name work       # -> ~/.config/hypr/layouts/work.json
hypr-layout restore --name work
hypr-layout layouts                # name, windows, when it was saved
```

`--pick` asks instead of being told, and the asking is done by the menu your desktop
already has — on Omarchy that is `omarchy-menu-select`, the same picker its system
menus use (falling back to wofi, rofi, fuzzel, dmenu):

```
Where should it go?
  keep as saved     everything back on its own workspace
  1                 move everything to ws1 (has windows)
  2                 move everything to ws2
  …
```

Workspaces 1–10 are offered whether or not they exist yet — Hyprland creates a
workspace the moment a window lands on it, so a number that is still empty is a
valid answer. With more than one layout saved, the layout is asked for first.

## Going back a save

Every `save` rotates the file it is about to replace into `~/.config/hypr/layout-history/`,
keeping the last 5 per layout — so a save taken at a bad moment does not cost you
the good one:

```bash
hypr-layout restore --history      # 12 Sep 10:24:09 · 7 windows
```

Menu: **Layouts… → Go back to an earlier save…**

## Windows that are not in the layout

Anything else on a layout's workspace is a tile the saved tree knows nothing about, and it splits the arrangement — a fifth tile in a 2×2 halves everything. That is what happens when something opens a window while you are working: it lands in the layout and pushes the rest aside.

So a replay can clear them out first, and the focused window can be sent away on a key:

```bash
hypr-layout restore --spill 5      # strays go to ws5, then the layout is rebuilt
hypr-layout spill                  # send the focused window to ws5
hypr-layout spill --workspace 9    # ...or wherever you keep them
```

Nothing is closed: a stray is moved to another workspace, where it carries on running.

## Launch-or-focus

`restore` never opens a second copy of something already open. If a window of that class exists, it is focused and moved instead — otherwise Firefox, Zed and OBS would simply open new windows every time you ran it. Candidates are scored on title, workspace and group membership, because every terminal shares a class and grabbing the wrong one drags an unrelated window across your layout. Use `--force` when you really do want new copies.

Two windows can share a class and be genuinely hard to tell apart — two Firefox windows drift apart in title as you navigate, and a title is not an identity. The pairing is therefore decided for all entries at once: every (entry, window) pair is scored, the most confident pairs are claimed first, and a tie goes to the window already nearest where that entry wants it. Matching one entry at a time is what let two browsers trade places on restore.

## What a restore reports

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
o.bind("SUPER + ALT + W", "Restore a saved layout…", "/home/USER/.local/bin/hypr-layout restore --pick --spill 5")
o.bind("SUPER + SHIFT + ALT + W", "Save work layout", "/home/USER/.local/bin/hypr-layout save")
o.bind("SUPER + SHIFT + ALT + S", "Send window to ws5", "/home/USER/.local/bin/hypr-layout spill")
```

`--pick` uses the Omarchy menu, so the same commands can live in it
(`~/.config/omarchy/extensions/omarchy-menu.jsonc`, which hot-reloads):

```jsonc
"layouts": {"icon":"󱂬","label":"Layouts","description":"Save and restore window layouts"},
"layouts.restore": {"label":"Restore layout…","action":"hypr-layout restore --pick"},
"layouts.undo": {"label":"Go back to an earlier save…","action":"hypr-layout restore --pick --history"},
"layouts.save": {"label":"Save layout as…","action":"hypr-layout save --pick"},
```

A row needs its leading glyph: without one the menu draws the label clipped.

Replay at login (`~/.config/hypr/autostart.lua`):

```lua
o.launch_on_start("bash -c 'sleep 8; echo \"=== $(date -Is) ===\"; ~/.local/bin/hypr-layout restore --no-verify' >>\"$HOME/.cache/hypr-layout-login.log\" 2>&1")
```

The sleep lets the session settle, and the log is there because a login-time replay has nowhere to print.

## Where it fits

Omarchy's compositor gives you tiling, groups and workspaces; what no session
manager does out of the box is put a *specific arrangement* back after a reboot
or a reshuffle. This is a small tool for that one job.

## How it works

- **One socket, not `hyprctl`.** Every dispatch goes over the command socket — about 0.1 ms against roughly 15 ms for spawning `hyprctl` per call.
- **Apps launch in parallel**, and each window is placed the instant it is announced on the event socket, instead of polling and sleeping between launches.
- **Placement is explicit and by address** — never inferred from a window rule.
- **A saved command is repaired before it is run.** Apps that rewrite their own command line — Electron and Chromium do — collapse the NUL separators in `/proc/<pid>/cmdline`, so the command arrives as one string holding the whole line. It is split back into arguments; without that, executing it fails with `ENOENT` and the window silently never appears.
- **A failed launch says so out loud.** stdout is not a terminal when the replay comes from a keybinding or a menu, so warnings also go to the desktop notification daemon.
- **Everything that acts on the active window is preceded by a verified focus**, because that is the one mistake that damages a live layout instead of failing cleanly.
- **`group:auto_group` is disabled for the whole replay** (including the arrangement pass) and restored afterwards, so windows opened mid-replay cannot join a group they do not belong to.
- **Animations are switched off for the replay** and restored afterwards. A replay parks and re-inserts every tile; with animations on you watch the layout strobe its way back, with them off it looks like the layout simply appears.
- **Arrangement is rebuilt** by parking the tiles on a hidden workspace (`special:hypr-layout`), deriving the dwindle tree from the saved rectangles, and re-inserting them in tree order with `layout preselect` + move. Anything that fails to land is moved back — a window is never left parked.

## Limitations

- The arrangement builder assumes the tiled windows in a snapshot form a clean BSP. A snapshot taken with overlapping or partially off-screen windows falls back to members-only restore (`--no-arrange` behaviour).
- Split ratios are re-applied for splits that are not near 50/50; near-half splits keep the compositor's default.
- Floating windows are only captured with `save --floating`.
- `--no-groups` also gives up the arrangement for those windows: a tile is moved as a unit through its group, so a tile whose windows were left ungrouped cannot be rebuilt as one.
- Windows of one class are told apart by title, workspace and group. When the titles have drifted there is nothing left to tell them apart, so the tie-break keeps the window nearest its target instead of moving it — a restore does not reshuffle what it cannot identify.
- Earlier saves are kept per layout, to a fixed depth of 5; there is no promotion of one to "the good one".
- Verified on one Hyprland version (0.56.2). The IPC commands used are long-standing, but newer builds may rename dispatchers — if a dispatch fails the tool reports it rather than failing silently.

## License

MIT — see [LICENSE](LICENSE).
