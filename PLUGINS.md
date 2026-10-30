# Launchpad plugins

A plugin is a single `.lpp` file (Launchpad plugin). It is Python source with a different extension, and `.lpp` is the only extension Launchpad accepts: `.py` files in the plugins folder are ignored, and Install plugin… refuses them. It can add **actions** to the Plugins menu (they return text that Launchpad shows in a window), add a **one-line note** under the selected game's details, and add whole **layouts** to the View menu.

## Using plugins

|What|How|
|-|-|
|Install|Plugins ▸ Install plugin… (the file is scanned first; only clean files are copied in)|
|Write your own|Plugins ▸ New plugin from template|
|Write your own layouts|Plugins ▸ New layout plugin from template|
|Switch on / off, block, remove|Plugins ▸ Manage plugins…|
|Pick a plugin's layout|View menu (listed only while its plugin is switched on)|
|Folder|`~/.launchpad/plugins/` (Plugins ▸ Open plugins folder)|

A plugin you install or drop into the folder is **off** until you enable it. Enabling is tied to the exact file contents, so if the file is edited afterwards it is switched off again until you re-approve it.

## Writing a plugin

```python
PLUGIN = {
    "id": "my_plugin",            # unique; lowercase letters, digits, underscores
    "name": "My Plugin",
    "version": "1.0",             # optional
    "author": "me",               # optional
    "description": "What it does.",   # optional
    "actions": [                  # optional: entries for the Plugins menu
        {"id": "count", "label": "Count my games"},
    ],
    "layouts": [                  # optional: entries for the View menu (max 5)
        {"id": "shelves", "label": "Shelves"},
    ],
}

def run_action(action_id, games):
    """Called from the Plugins menu. Return plain text (max 20,000 characters)."""
    return f"{len(games)} games"

def game_note(game):
    """Optional. One short line (max 300 characters) shown under the selected game."""
    return ""

def build_layout(layout_id, games, ctx):
    """Return a dict describing a library layout (see "Layouts" below)."""
    return {"items": [{"game": i} for i in range(len(games))]}
```

`PLUGIN` must be made of plain literals (no code). You need at least one of: `actions` + `run_action`, `game_note`, or `layouts` + `build_layout`. Declaring `layouts` without defining `build_layout` is an error; defining `build_layout` without declaring `layouts` only produces a warning (it would never be used). See `plugins/playtime_stats.lpp` for a fuller example.

### What a plugin receives

Each game is a dict with exactly these keys, and nothing else:

|key|meaning|
|-|-|
|`title`|game title (max 120 chars)|
|`source`|`"steam"`, `"epic"` or `"manual"`|
|`installed`|`True` / `False`|
|`playtime_seconds`|total play time|
|`play_count`|times launched|
|`last_played`|unix time, or `None`|
|`added`|unix time the game was added|

`run_action` gets the current account's whole library; `game_note` gets only the selected game; `build_layout` gets the games currently shown (the search box and filter are already applied), plus a small `ctx` dict: `{"width": window width in pixels, "dark": True/False}`. Plugins never receive file paths, launch arguments, Steam App IDs, Steam IDs / API keys / persona names, achievements, usernames, passwords, tokens or anything else from the launcher's accounts.

### What a plugin may use

* **Imports (only these):** `math statistics random re json datetime time calendar collections collections.abc itertools functools operator string textwrap heapq bisect decimal fractions enum dataclasses typing difflib unicodedata colorsys copy numbers`
* Normal Python: classes, comprehensions, f-strings, and so on.
* Not available: `open`, `eval`, `exec`, `compile`, `getattr`/`setattr`, `globals`/`vars`, `__dunder__` attributes, anything that touches files, the network, processes or the system.
* `print()` goes nowhere. Return text (or, for `build_layout`, a layout dict) instead.
* Limits: 10 seconds per action, 5 seconds per layout, 3 seconds per note (the plugin is stopped if it takes longer), 20,000 / 300 characters of output for actions / notes, and 300,000 characters of JSON for a layout.
* Use f-strings or `%` formatting; `str.format()` is accepted only on a literal string without attribute access (`"{:.1f}".format(x)` is fine, `"{0.name}"` is not).

## Layouts

A layout plugin does not draw anything. `build_layout` returns **plain data** saying which games to show, in what order, grouped how, and what text and colour goes on each one. Launchpad checks that data field by field and draws it itself, so a plugin gets no access to the user interface. Clicking, double-click to play, the right-click menu, the details bar and the arrow keys all work on a layout like on the built-in views.

`build_layout(layout_id, games, ctx)` runs again whenever the shown games change (searching, filtering, play time updating), so keep it quick. `layout_id` is whichever of your declared layouts was picked.

You refer to a game by its **position in `games`** (0, 1, 2 …), never by an id. A game can appear only once; repeats are dropped, and games you don't mention are not shown.

### The layout dict

|key|meaning|
|-|-|
|`style`|`"grid"` (cover cards, the default) or `"rows"` (one line per game, like a table)|
|`size`|grid only: `"small"`, `"medium"` (default) or `"large"` cards|
|`covers`|grid only: `False` draws a colour tile instead of cover art|
|`headers`|rows only: up to 4 column titles|
|`empty`|text shown when the layout places no games (max 200 chars)|
|`sections`|list of sections (max 60), each `{"title": ..., "subtitle": ..., "items": [...]}`; title and subtitle are optional (max 60 / 120 chars)|
|`items`|shortcut: put `items` at the top level instead of `sections` for one untitled section|

### Each item

|key|meaning|
|-|-|
|`game`|**required.** Index into `games`, a whole number from 0 to `len(games) - 1`|
|`text`|replaces the title line (max 80 chars)|
|`sub`|second line of text (max 80 chars); default is the play time|
|`badge`|short label on the card (max 16 chars); replaces the STEAM / EPIC badge|
|`color`|`"green"`, `"red"`, `"blue"`, `"orange"`, `"purple"`, `"gray"`, `"teal"`, `"pink"` or `"#rrggbb"`: tints the badge and adds an accent bar|
|`cols`|rows only: up to 4 short texts (max 40 chars each) for the right-hand columns|

Launchpad always adds its own "RUNNING" and "NOT INSTALLED" badges. Control characters in any text are replaced with spaces.

### Example

```python
PLUGIN = {
    "id": "by_playtime",
    "name": "By play time",
    "layouts": [{"id": "table", "label": "Play time table"}],
}

def build_layout(layout_id, games, ctx):
    order = sorted(range(len(games)), key=lambda i: -games[i]["playtime_seconds"])
    items = []
    for i in order:
        g = games[i]
        items.append({"game": i, "sub": g["source"],
                      "cols": [f"{g['playtime_seconds'] / 3600:.1f} h", f"{g['play_count']} launches"],
                      "color": "green" if g["installed"] else "gray"})
    return {"style": "rows", "headers": ["Played", "Launched"], "sections": [{"items": items}]}
```

Plugins ▸ New layout plugin from template writes a fuller one, with a grid of "shelves" grouped by play time as well as a table.

### When a layout goes wrong

* If `build_layout` raises, times out, or returns something invalid (wrong type, unknown `style`, out-of-range `game`, unknown colour, too big, not plain JSON-style data), the library shows the error message instead of games, and you can pick another view from the View menu. The message names the problem so you can fix it.
* If the plugin is switched off, removed or blocked while its layout is showing, Launchpad goes back to the Game cards.
* If it tries something the sandbox forbids, it is killed and blacklisted like any other plugin.

## How plugins are kept safe

1. **Static scan** (never runs the code). Before a plugin can be installed or enabled, its code is parsed and checked. **Dangerous** findings: imports of `os` / `sys` / `subprocess` / `socket` / `ctypes` / `pathlib` / `urllib` and similar, `eval` / `exec` / `open` / `getattr`, `__dunder__` tricks, climbing into interpreter frames, reaching into other modules' internals, string literals that point at Launchpad data, Steam data or personal files, and large encoded blobs. **Errors** (not malicious, but unusable): syntax errors, imports that aren't on the allowed list, bad metadata.
2. **Blacklist.** A plugin with dangerous findings is never installed or run; its SHA-256 and a "code shape" fingerprint (so re-formatting or re-commenting doesn't help) go into `~/.launchpad/plugin_blacklist.json`. Files you block by hand go there too, and you can unblock those. Scan-based and runtime blocks can't be unblocked (fix the code and it becomes a different file). A list of known-bad hashes can also ship inside the app (`PLUGIN_BUNDLED_BLACKLIST`).
3. **Sandbox process.** Each plugin runs in its own child process with an empty environment, an empty temporary folder as its home, stripped-down built-ins and an import allow-list. A Python audit hook blocks opening files, sockets, starting processes, `ctypes`, `sqlite3`, `os.*`, `gc`/frame tricks and any import outside the allow-list. The child process never even loads `subprocess`, `sqlite3`, `urllib` or Tk.
4. **Runtime enforcement.** If a plugin that got past the scan still *tries* something the sandbox forbids, it is killed, blacklisted and you are told.
5. **Minimal data in.** See "What a plugin receives" above.
6. **Data out, never code.** A layout comes back as plain JSON-style data (dicts, lists, text, numbers). It is size-limited, then validated against a fixed set of fields, so the only thing a layout can do is choose which of the games it was given to show and what short text and colour to put on them.

### Honest limits

This is strong defence in depth, but it is not a virtual machine. Python can't be sandboxed perfectly from inside Python, so only install plugins from people you trust, and treat the scanner as a safety net rather than a guarantee. In particular: memory limits are enforced on Linux/macOS only (Windows relies on the time limit); a plugin can't read file *contents*, but the sandbox doesn't hide that files exist, because it can't hide that from a process.
