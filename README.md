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
hypr-layout log                     # print the last run's trace (`--previous` for the one before)
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
  3                 move everything to ws3
```

The rows mirror the bar's workspace indicator: 1 to 5 always — the bar draws them
dimmed while they are empty — plus any other workspace that exists, up to 10. So the
list you pick from is the list already on screen. To send a layout to a workspace that
does not exist yet, switch to it once (which creates it) and it appears here;
`--workspace N` remains for scripts. With more than one layout saved, the layout is
asked for first.

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

So a replay can clear them out first, or fold them in, and the focused window can be dealt with on a key:

```bash
hypr-layout restore --spill 5      # strays go to ws5, then the layout is rebuilt
hypr-layout restore --tuck         # strays become tabs in the top-left tile instead
hypr-layout spill                  # send the focused window to ws5
hypr-layout tuck                   # fold the focused window into the layout
```

The default spill workspace is 5, and `--workspace N` moves it.

**Tucking** keeps the window where you are: it becomes a tab in the top-left tile — a tab costs no space, so the tile keeps its rectangle and the arrangement is untouched — and if that tile is not a group (nothing to join), it floats over that corner instead, which also keeps it out of the tiling tree. Tab numbers are no longer compared literally, because a tucked tab shifts every index after it, and that is not the layout being wrong.

Nothing is closed either way: a stray is moved or tucked, never killed.

## Launch-or-focus

`restore` never opens a second copy of something already open. If a window of that class exists, it is focused and moved instead — otherwise Firefox, Zed and OBS would simply open new windows every time you ran it. Candidates are scored on title, workspace and group membership, because every terminal shares a class and grabbing the wrong one drags an unrelated window across your layout. Use `--force` when you really do want new copies.

Two windows can share a class and be genuinely hard to tell apart — two Firefox windows drift apart in title as you navigate, and a title is not an identity. The pairing is therefore decided for all entries at once: every (entry, window) pair is scored, the most confident pairs are claimed first, and a tie goes to the window already nearest where that entry wants it. Matching one entry at a time is what let two browsers trade places on restore.

A Chromium window changes class from one session to the next, because Chromium takes its app id from how its *first* process was started: through `/usr/bin/chromium` (the wrapper that sets `CHROME_WRAPPER`) the window is `chromium`, started as the bare `/usr/lib/chromium/chromium` it is `chromium-browser`. Same browser either way, so the two names are one class as far as matching goes. Before that, a snapshot taken in one session never recognised the browser in the next: every replay opened another copy — and, because the copy also did not match, left it behind as a stray. That is the failure the run log exists to make visible.

Chromium **web apps** (the `--app=` windows Omarchy opens for Discord, WhatsApp and friends, named `chrome-<url>__<profile>`) cannot be relaunched at all: the window belongs to the shared browser process, so the command line saved for it is the plain browser's, and running that opens a blank tab rather than the app. Such an entry is matched to its window when one is open and reported when it is not — it is never launched into a blank window that then splits the layout. `--force` overrides this like every other launch decision.

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

## When a replay goes wrong

A replay usually starts from a keybinding, where stdout has nowhere to go — so a
replay that half-worked used to leave nothing behind to look at. Every run now
writes its whole trace to `~/.local/state/hypr-layout/last-run.log`, and the run
before it to `previous-run.log`:

```bash
hypr-layout log              # what the last replay did
hypr-layout log --previous   # the one before it
```

It records each dispatch *and what the compositor answered*, which window was
matched to which entry and by what evidence, every launch with its pid, every
window event, which groups and arrangement passes ran, and what was skipped and
why — plus a traceback if it ever crashes. The compositor's usual way to fail is
to answer `ok` and do something else, so the reply alone is not proof; what came
before it is.

A replay that cannot do part of its job says so instead of going quiet: a group
whose window never opened, a workspace whose arrangement was skipped for want of
a tile, a window that opened during the replay and could not be matched to
anything.

A group that cannot be built is taken back apart rather than left half-formed.
That is not tidiness: a group of one is not a group, but it still draws a tab
bar, and the arrangement pass moves a group as a single unit — so a workspace
holding one is no longer the shape the save describes. The arrangement for that
workspace is therefore skipped as well, and the tiling is left as it was found
rather than rebuilt into something the save never had. (On 2026-09-13 one failed
merge left exactly that state and the arrangement turned a working desktop into
0/9.) For the same reason a merge never *keeps* a window it pulled into a
neighbouring group by accident: the window is released again, so a layout can
only gain a tab its own save asked for.

A replay announces itself while it runs: a small window follows the run log
(floating and never focused, so it stays out of the very tiling it describes),
and every phase also goes to the desktop as a one-line OSD. `--no-progress`
turns both off.

Two things it waits for before it touches a group. A window exists long before
its application is done with it, so a replay waits until the windows it launched
have stopped resizing. And a window that is a fraction of the size of the tile
the save describes — or floating where the save says tiled — is not that window:
it is a dialog the app put up instead. A replay will not group it, because tab
groups hold the real window and the attempts churn everything they move. That is
not theory: on 2026-09-13 a replay relaunched eight apps, OBS opened with its
"unclean shutdown detected" prompt, and three merge attempts against that 500x142
dialog left the whole workspace scattered. Dismiss the dialog and restore again;
the second run completes.

A group that still fails is retried once after a pause, and if it fails again the
arrangement for that workspace is skipped and says so — as above, a workspace
whose groups are not the saved ones is not the saved shape. The arrangement also
refuses to run when a workspace holds a different *number* of tiles than the save
describes, whatever the reason (a stray, a window that opened mid-replay,
something of ours left on screen): rebuilding a tree around an extra leaf splits
tiles that should not be split, which is how one stray leaf halved all four tiles
of a saved layout that same evening.

Only one replay runs at a time. A second one started on top of the first parks
tiles the first is still inserting and dissolves the groups it has just built —
which is what a double-press of the keybinding looks like from the inside, and
it leaves a layout with nothing grouped and no clue why. The second one is
refused, and says who is still working.

## Checking it

`tests/replay-check` is an end-to-end check for a live session: it builds a 2×2
fixture with a group of three tabs on its own workspace, saves it, kills the
windows, relaunches them scrambled (two of them only when the replay starts
them), replays, and fails if a window, a rectangle, a tab order or a thing that
should *not* be grouped comes back wrong. It also checks that the trace lands on
disk and that a second concurrent replay is refused. It uses its own window
class and cleans up after itself, but it does move windows, so run it when you
are not mid-something.

It is hermetic on purpose: it picks two workspaces that have nothing on them,
switches the neighbour-group automation off while it runs, and puts your active
workspace and that setting back on the way out. An earlier version ran its
fixture on the user's own workspace, where one fixture window was merged into a
live group — a test must never share a window tree it does not own.

```bash
python3 tests/replay-check
```

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
o.launch_on_start("bash -c 'sleep 8; ~/.local/bin/hypr-layout restore --no-verify'")
```

The sleep lets the session settle before the replay starts moving windows. There is no
redirect: a login-time replay has nowhere to print, which is exactly why the run log
exists — `hypr-layout log` reads back what it did.

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
- **The run keeps a trace.** Every dispatch and its reply, every match, launch, event, group and arrangement decision goes to `~/.local/state/hypr-layout/last-run.log`, because a replay that runs from a keybinding has no terminal to leave its output in — and the interesting failure is the one that never gets to say anything on the way out. `hypr-layout log` prints it.
- **Nothing gives up quietly.** A group whose window never opened, a workspace whose arrangement was skipped, a window that appeared and matched nothing: each is named. A replay that silently stops halfway is indistinguishable from one that is still working.
- **One replay at a time**, so a second press of the keybinding cannot stampede the first one's half-built layout.
- **Launches are waited for only as long as it can still help.** A launcher that has already exited and produced no window — a hand-off to the running browser — is given a few seconds, not the full timeout, so a replay that cannot place everything still finishes promptly.
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
- A Chromium web app cannot be brought back by a replay (see above); open it and replay again. Its tile cannot be rebuilt either, so a workspace with a missing tile keeps the tiling it had rather than an approximation — that is reported, not guessed at.
- Earlier saves are kept per layout, to a fixed depth of 5; there is no promotion of one to "the good one".
- A replay waits for a launched app for as long as its launcher is still running, then a few seconds more; an app slower than that to show a window is reported as missing. Nothing is left waiting on a window that a process hand-off will never produce.
- Verified on one Hyprland version (0.56.2). The IPC commands used are long-standing, but newer builds may rename dispatchers — if a dispatch fails the tool reports it rather than failing silently.

## License

MIT — see [LICENSE](LICENSE).
