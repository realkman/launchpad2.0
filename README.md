# Launchpad

A simple custom game launcher with local accounts, built with Python and Tkinter.

## Features

* Local accounts (username + password, PBKDF2-hashed) with optional "Keep me signed in"
* Per-user game library stored in SQLite
* Add games manually or **Import folder** to scan for launchable files
* Steam sync: detects your Steam install, reads installed games, and can optionally include owned games via the Steam Web API
* Game cards with cover art (Steam artwork is cached locally; Pillow enables JPG/WebP covers)
* Play count, playtime and last-played tracking
* Search, filters (All / Installed / Not installed / Steam / Non-Steam), card or list view
* Light, dark, or match-system theme
* NEW Plugins, allows users to expand launchpad's features without harming the users computer.

## Run from source

Requires Python 3.8+ with Tkinter (bundled with the Windows/macOS installers; on Debian/Ubuntu: `sudo apt install python3-tk`).

```bash
pip install -r requirements.txt   # optional: Pillow for real cover art
python launchpad.py
```

## Download the Launcher

Check releases.



The exe is in `dist/Launchpad.exe`.

## Where data lives

|What|Location|
|-|-|
|Accounts + libraries|`\~/.launchpad/launchpad.db`|
|Settings (theme, view, Steam options)|`\~/.launchpad/settings.json`|
|Cached cover art|`\~/.launchpad/covers/`|
|"Keep me signed in" token|`\~/.launchpad/session.token`|

Set the `LAUNCHPAD\_HOME` environment variable to store everything somewhere else.

> The Steam Web API key is stored in plain text in `settings.json`. Never share that file.

## Repo layout

```
launchpad.py            # the app (single file)
Launchpad.spec          # PyInstaller build config
requirements\*.txt
Python 3.8+ with Tkinter



License

MIT, see [LICENSE](LICENSE).

