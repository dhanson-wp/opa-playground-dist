# Opa — Playground

Fast, disposable Opa instance running in the browser. Send the link, get
feedback in 30 seconds. No accounts, no installs, no infra.

## Tester link

```
https://playground.wordpress.net/?blueprint-url=https://raw.githubusercontent.com/dhanson-wp/opa-playground-dist/main/blueprint.json
```

That URL boots a Playground, installs the Opa platform plugin and theme,
logs in as `admin`, and lands the user at the Opa dashboard. Because the
Playground starts with zero posts, the welcome overlay shows on first load.

Each visit is a fresh disposable WordPress in WebAssembly + SQLite. No
data persists across reloads. The whole thing runs in the user's browser.

## How it works

```
┌──────────────────────────────────────────────────────────────────────┐
│  dhanson-wp/opa-blog (PRIVATE)                                        │
│       │  source code, build/ gitignored                                │
│       │                                                                │
│       │  ./playground/build-and-push.sh                                │
│       ▼                                                                │
│  dhanson-wp/opa-playground-dist (PUBLIC, satellite repo)              │
│       ├─ blueprint.json                                                │
│       └─ dist/                                                         │
│             ├─ opa-platform.zip  (plugin form, with built SPA)         │
│             └─ opa-theme.zip                                           │
│                                                                        │
│  Playground reads blueprint.json from the public satellite repo       │
│       └─ installPlugin + installTheme from the dist/ ZIPs              │
└──────────────────────────────────────────────────────────────────────┘
```

Why a satellite repo? The main `opa-blog` repo is private. WordPress
Playground Blueprint URLs must be publicly fetchable, so a separate public
repo holds only the distribution artifacts. The main repo stays private.

Two pieces in the main repo:

1. `playground/blueprint.json` — the recipe. Tells Playground which
   WP/PHP to use, which plugin/theme to install, how to log the user in,
   and where to land. This file is copied verbatim to the satellite repo
   on each rebuild.
2. `playground/build-and-push.sh` — packages the mu-plugin (with the SPA
   `build/` directory baked in) as a regular WordPress plugin ZIP,
   packages the Opa theme as a theme ZIP, syncs both plus the Blueprint
   to the satellite repo.

## Updating the tester link

After any change to `mu-plugins/opa-platform/`, `themes/opa/`, or
`playground/blueprint.json` that you want testers to see, commit the
change in the main repo, then run from the repo root:

```bash
./playground/build-and-push.sh
```

The script:

1. Builds the SPA (`npm install && npm run build` inside the mu-plugin).
2. Packages the mu-plugin as `playground/dist/opa-platform.zip` in plugin
   form (loader file + nested directory tree, no path edits needed).
3. Packages the theme as `playground/dist/opa-theme.zip`.
4. Clones the satellite repo to `~/opa-playground-dist` (first run only).
5. Resets the satellite repo to its origin/main, copies in the new
   Blueprint + ZIPs, commits, pushes.

The Playground link itself never changes. Testers refreshing the link
see the latest build within seconds of the push completing.

## Why a regular plugin, not a mu-plugin?

mu-plugins live at `wp-content/mu-plugins/` and auto-load. WordPress
Playground's `installPlugin` step drops files into `wp-content/plugins/`.
The Opa loader is structured so it works in either location — same
`require_once` chain, same hooks. For the Playground tester loop, regular
plugin form is the path of least resistance.

In production on Pressable, the same code runs as a mu-plugin (no
activation needed, network-wide). Two distribution forms, one codebase.

## Troubleshooting

If Playground shows a blank screen at the SPA URL: the build artifacts
weren't included in the ZIP. Rebuild with `./playground/build-and-push.sh`
and verify `playground/dist/opa-platform.zip` is non-trivial in size
(should be 1MB+ — the SPA bundle is the bulk).

If the welcome overlay doesn't appear: there are already posts in the
SQLite db. That shouldn't happen on a fresh Playground load. If it does,
hard-refresh the Playground page or open in an incognito window.

If the public toolbar doesn't appear on the front-end: the user needs to
be logged in (the toolbar is logged-in only by design). Click "View site"
from the dashboard while still logged in.

## Local testing without pushing

To test the Blueprint locally before pushing release-playground:

```bash
cd ~/opa-blog
./playground/build-and-push.sh   # generates playground/dist/*.zip
npx @wp-playground/cli@latest run-blueprint \
  --blueprint=./playground/blueprint.json \
  --blueprint-may-read-adjacent-files
```

But the live Playground link can only fetch from public URLs, so for
sharing with testers you have to push to `release-playground`.
