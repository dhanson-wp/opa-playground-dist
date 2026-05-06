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
┌──────────────────────────────────────────────────────────────────┐
│  trunk (production code, build/ gitignored)                       │
│       │                                                            │
│       │  ./playground/build-and-push.sh                            │
│       ▼                                                            │
│  release-playground branch                                        │
│       ├─ trunk's source                                            │
│       └─ playground/dist/                                          │
│             ├─ opa-platform.zip  (plugin form, with build/)        │
│             └─ opa-theme.zip                                       │
│                                                                    │
│  Playground reads blueprint.json                                  │
│       └─ installPlugin + installTheme from the dist/ ZIPs          │
└──────────────────────────────────────────────────────────────────┘
```

Two pieces:

1. `playground/blueprint.json` — the recipe. Tells Playground which WP/PHP
   to use, which plugin/theme to install, how to log the user in, and where
   to land.
2. `playground/build-and-push.sh` — packages the mu-plugin (with the SPA
   `build/` directory baked in) as a regular WordPress plugin ZIP, packages
   the Opa theme as a theme ZIP, force-pushes both to the long-lived
   `release-playground` branch.

The `release-playground` branch only exists for distribution. Treat it as
ephemeral. Never base feature branches off it.

## Updating the tester link

After any change to `mu-plugins/opa-platform/` or `themes/opa/` that you
want testers to see, run from the repo root:

```bash
./playground/build-and-push.sh
```

The script:

1. Builds the SPA (`npm install && npm run build` inside the mu-plugin).
2. Packages the mu-plugin as `playground/dist/opa-platform.zip` in plugin
   form (loader file + nested directory tree, no path edits needed).
3. Packages the theme as `playground/dist/opa-theme.zip`.
4. Switches to `release-playground` branch, hard-resets to current `trunk`,
   commits the two new ZIPs, force-pushes.
5. Returns you to the branch you started on.

The Playground link itself never changes. Testers refreshing the link see
the latest build.

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
