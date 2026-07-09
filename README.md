# Cozy Animated Player

A single-page browser video player with MKV support: multi-audio-track switching (including
Dolby/DTS/TrueHD via in-browser transcoding), subtitle extraction (SRT/ASS embedded in MKV, or
your own .srt/.vtt), drag-and-drop playlist, and a YouTube-style UI.

## Structure

```
index.html                     — the player itself (open this)
vendor/ffmpeg/                 — self-hosted ffmpeg.wasm assets (see below)
  ffmpeg.js                     — @ffmpeg/ffmpeg 0.12.15 (UMD build)
  814.ffmpeg.js                  — its internal worker chunk (loaded automatically by ffmpeg.js)
  ffmpeg-core.js                  — @ffmpeg/core 0.12.10 (UMD build, single-threaded)
  ffmpeg-core.wasm.part0…part3     — the decoder binary, split into <9MB chunks (see below)
  manifest.json                    — chunk order + sizes, used to reassemble the binary at runtime
  ffmpeg-util.js                  — @ffmpeg/util 0.12.2 (UMD build)
```

## Does this need to be on GitHub?

**No.** What it actually needs is to be served over `http://` or `https://` by *some* web
server — GitHub Pages is one option, but not the only one, and not required just to test it.

Opening `index.html` by double-clicking it loads it as a `file://` URL, and browsers
deliberately cripple `file://` pages: they block the `fetch()` calls this player uses to check
for local files, and they restrict or refuse the Web Worker that ffmpeg.wasm depends on
entirely. That's the actual reason MKV audio conversion doesn't work when opened directly —
it's not a GitHub-specific requirement, it's a "needs a real HTTP origin" requirement.

**To test locally with zero uploading**, serve the folder with any of these (run from inside
the project folder, then open the printed `http://localhost:...` address):

```bash
# Python (usually already installed)
python3 -m http.server 8000

# Node
npx serve .

# VS Code
# Install the "Live Server" extension, right-click index.html → "Open with Live Server"
```

Any of these make it behave exactly as it will once actually deployed — same-origin asset
loading, working Web Workers, everything. GitHub Pages, Netlify, Vercel, or your own server all
work equally well for real deployment; there's nothing GitHub-specific in the code.

## Why the wasm file is split into 4 parts

ffmpeg.wasm needs to load a ~31 MB WebAssembly decoder. GitHub's **drag-and-drop web upload**
caps individual files at 25 MB — the single `ffmpeg-core.wasm` (30.7 MB) is just over that.

(Note: this 25 MB cap is specific to the web upload button. Pushing via `git` command line or
GitHub Desktop has a much higher real limit — 100 MB per file — so if you're comfortable with
git directly, you could skip the chunking entirely and use one `ffmpeg-core.wasm` file instead.
The chunked version here is so the web-upload button works too, for anyone who'd rather not
touch git.)

Each `.wasm.part*` file is ≤8 MB. At runtime, `index.html` fetches `manifest.json` to learn the
correct order and sizes, fetches every part, concatenates them byte-for-byte into memory, and
hands the browser a single reassembled blob — functionally identical to loading one big file.
This only happens once per session (the very first time an MKV needs audio conversion), and the
five small files that make up the rest of `vendor/ffmpeg/` (all under 120 KB) were never
affected by the size limit in the first place.

If the `vendor/ffmpeg` folder is ever missing entirely (e.g. someone copies just `index.html`
elsewhere), the player automatically falls back to loading everything from jsdelivr/unpkg
instead — slower and dependent on those CDNs being reachable, but still functional.

**Important:** this deliberately uses the single-threaded `@ffmpeg/core` package, not
`@ffmpeg/core-mt`. The multi-threaded variant requires `SharedArrayBuffer`, which only exists
in browsers when the page is served with `Cross-Origin-Opener-Policy` /
`Cross-Origin-Embedder-Policy` response headers — headers GitHub Pages (and most static hosts)
have no way to set. The single-threaded core doesn't need any of that and works on plain static
hosting.

## Updating ffmpeg.wasm

These are pinned, verified-working versions. If you ever want to bump them: download the new
UMD builds from npm (`@ffmpeg/ffmpeg`, `@ffmpeg/core`, `@ffmpeg/util` — **not** `core-mt`),
replace the small files directly, and re-split the new `ffmpeg-core.wasm` into ≤8MB parts with
matching filenames (`ffmpeg-core.wasm.part0`, `.part1`, …) plus a regenerated `manifest.json`
(`{"totalSize": <bytes>, "chunkCount": <n>, "parts": [{"name": "...", "size": <bytes>}, ...]}`).
