# Deploying the Iso Room Editor (Ruffle + GitHub Pages)

The Ruffle build had three independent gotchas. Each one alone makes the page show blank. This document captures what fixed it so you don't have to rediscover them.

---

## TL;DR — the three fixes

1. **Compile with Apache Flex SDK 4.16.1, not AIR SDK 30.** AIR SDK 30's `asc2` emits ABC bytecode Ruffle's AVM2 verifier can't parse (`Error #1107`). Flex 4.16.1's older `mxmlc` emits classic ABC that Ruffle accepts.
2. **Delete the `obj/` directory before every build.** The Flex compiler caches ABC blocks in `obj/` and will happily reuse a corrupt cached block from a previous failed build — leaving you with a SWF whose surface bytes change but whose broken inner ABC silently survives. Symptom: you change source, recompile, the SWF size shifts, but Ruffle still throws the same `Error #1107`. Always `rm -rf obj/` before building for Ruffle.
3. **Use SWF-relative paths inside the AS3 code AND set Ruffle's `base` option in the HTML.** Ruffle resolves `URLLoader` relative URLs against the *host page*, not the SWF — opposite of stock Flash Player. Set `base: "bin/"` on the player config and keep your AS3 URLs relative (`assets/wall.png`, `manifest.json`, `../fods/...`). That way the same SWF works on `localhost` AND on a GitHub Pages subpath like `peruql.github.io/iso-room-editor/`, with no absolute paths to retarget per environment.

---

## The build command

`build_ruffle.bat` in the editor project's root:

```batch
@echo off
setlocal
cd /d "%~dp0"

set FLEX_SDK=%USERPROFILE%\Desktop\swfdecrypt\flex_sdk
set MXMLC=%FLEX_SDK%\bin\mxmlc.bat
set PLAYERGLOBAL_HOME=%FLEX_SDK%\frameworks\libs\player

REM Clean the ABC cache — non-negotiable for Ruffle builds.
if exist obj rmdir /s /q obj

call "%MXMLC%" ^
    +playerglobal.version=11.1 ^
    -target-player=11.1 ^
    -swf-version=14 ^
    -optimize=false ^
    -static-link-runtime-shared-libraries=true ^
    -source-path=src ^
    -output=bin\NewProject.swf ^
    src\Main.as
endlocal
```

Why FP 11.1 / SWF 14? Higher player versions (11.3, 32.0) make Flex 4.16.1 emit ABC instructions Ruffle's verifier rejects. 11.1 is the sweet spot — old enough to be safe, new enough to compile our code.

Tradeoff: `MouseEvent.RIGHT_MOUSE_DOWN` requires FP 11.2, so right-click map panning doesn't dispatch in Ruffle. The editor already supports `Space + drag` as a fallback (handled by `IsoInputHandler.onMouseDown`), so this isn't a real loss.

---

## Path conventions inside the AS3 code

All `URLLoader` / `Loader.load()` calls use paths relative to the SWF's directory:

```actionscript
// In EditorUI.as
loadFloorAsset("floor_42", "assets/s_1424.png");
lib.loadAsset("wall", "assets/wall.png", ...);
loader.load(new URLRequest("assets/wall_visibility.json"));

// In AssetManifest.as
public static const MANIFEST_URL:String = "manifest.json";
public static const FODS_ROOT:String = "../fods";
```

When the SWF runs:
- Under the **standalone Flash projector / AIR**: paths resolve against the SWF's directory (`bin/`).
- Under **Ruffle**: paths resolve against `base` from the player config.

Setting `base: "bin/"` in the harness makes both runtimes behave identically.

---

## The HTML harness

The page-level config in `index.html`:

```javascript
await player.load({
  url: "bin/NewProject.swf?v=2",   // ?v=N busts browser cache on each redeploy
  base: "bin/",                    // critical — see above
  allowScriptAccess: true,
  autoplay: "on",
  unmuteOverlay: "hidden",
  backgroundColor: "#11161b",
  letterbox: "on",
  warnOnUnsupportedContent: false,
  contextMenu: "off",              // disable Ruffle's right-click quality menu
  playerVersion: 32,
});
```

`contextMenu: "off"` plus `stage.showDefaultContextMenu = false` in `Main.as` together suppress every right-click popup so the canvas is unencumbered.

---

## Repo layout for GitHub Pages

The **public** repo (`iso-room-editor-public/`) contains *only* the runtime artifacts — not the AS3 source, not the build scripts, not anything with credentials in it:

```
index.html         Ruffle harness (paths relative to this file)
bin/
  NewProject.swf   compiled editor (rebuilt locally, copied in)
  manifest.json    auto-generated asset catalog
  assets/          floor + wall sprites
fods/              isometric asset library (~3500 PNGs, ~63MB)
ruffle/            self-hosted Ruffle nightly (JS + WASM)
README.md
DEPLOY.md         (this file)
```

The **private** source repo lives separately on disk and is never pushed. Anything with secrets (Cloudflare R2 keys, proxy passwords, login scripts) stays out of the public artifact entirely.

---

## GitHub Pages setup (one-time)

```bash
# From the public dist directory
git init -b main
git config user.name "your-handle"
git config user.email "your-handle@users.noreply.github.com"
git add -A
git commit -m "Initial public release"

# Create the repo + push in one command (requires gh CLI authenticated)
gh repo create iso-room-editor --public --source=. --remote=origin --push

# Enable Pages on main branch root
# (MSYS_NO_PATHCONV stops Git Bash from mangling the /repos/... URL)
MSYS_NO_PATHCONV=1 gh api -X POST /repos/<owner>/iso-room-editor/pages \
    -f "source[branch]=main" \
    -f "source[path]=/"
```

The site goes live at `https://<owner>.github.io/iso-room-editor/` in 30–60 seconds.

---

## Redeploying after code changes

```bash
# 1. In the editor source project
rm -rf obj/              # MUST clean the cache
./build_ruffle.bat       # produces bin/NewProject.swf

# 2. Copy artifacts into the public dist
cp bin/NewProject.swf  ../iso-room-editor-public/bin/
cp bin/manifest.json   ../iso-room-editor-public/bin/   # if assets changed
# (regenerate manifest first if you added new fods/ files:)
#   python tools/build_manifest.py

# 3. Bump cache buster + push
cd ../iso-room-editor-public
# bump ?v=N in index.html (any change to the URL forces clients to re-fetch)
git add -A
git commit -m "Rebuild"
git push
```

Pages auto-rebuilds in ~30 seconds. Old SWFs cached in browsers still serve until the `?v=N` bumps.

---

## Symptoms-and-causes cheatsheet

| Symptom | Cause | Fix |
|---|---|---|
| Page blank, console shows `Error #1107: ABC data is corrupt` | Stale ABC in `obj/` cache, or compiling with AIR SDK | `rm -rf obj/` + use Flex SDK 4.16.1 |
| Page blank, console shows `Error #1065: Variable Main is not defined` | Same as above — ABC verifier bailed before symbol-class registration | Same |
| SWF loads but no floors/walls render | Asset URLs resolved to wrong base | Set `base: "bin/"` in Ruffle config; kep AS3 paths SWF-relative |
| `Error: unable to open '{playerglobalHome}/11.X'` at compile time | `PLAYERGLOBAL_HOME` env not set | Set in build script: `set PLAYERGLOBAL_HOME=%FLEX_SDK%\frameworks\libs\player` |
| Right-click pops Ruffle's quality menu | Default context menu wasn't disabled | Add `contextMenu: "off"` to Ruffle config and `stage.showDefaultContextMenu = false` in `Main.as` |
| FlashDevelop's own build (AIR target) breaks with `compiler.debug must only be set once` | `additional` flags in `.as3proj` conflict with FD's auto-injected debug flag | Set `<option additional="" />` in `New Project.as3proj` and `<option optimize="True" />` |

---

## Why this combination works

The big win is decoupling the AS3 source from the deployment environment. Paths inside the SWF say "load this *relative to my directory*" — and both runtimes (standalone Flash and Ruffle) implement that, just via different mechanisms (loader URL vs. `base` option). The harness configures the runtime once; the SWF is portable.

The smaller wins (Flex over AIR, clean `obj/`, FP 11.1 over 11.3) are all about producing ABC bytecode an emulator built in 2026 will accept. Adobe stopped iterating on the player; Ruffle is catching up. Older, simpler ABC is the safer target until Ruffle's coverage of newer instructions stabilizes.