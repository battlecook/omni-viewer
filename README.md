# Omni Viewer

[![Sponsor](https://img.shields.io/badge/Sponsor-%E2%9D%A4-red)](https://github.com/sponsors/battlecook)

**One viewer engine, every platform.** Omni Viewer opens audio, video, images, documents,
spreadsheets, ML model files, and engineering/automotive data formats — without leaving the
tool you already work in, and without uploading your files anywhere.

This repository is the **hub**: it explains how the pieces fit together and where to go for
each platform. The code lives in the repositories linked below.

---

## Architecture

Every platform build sits on top of the same shared engine, [`omni-viewer-core`](https://github.com/battlecook/omni-viewer-core).
The core is host-agnostic — parsers take bytes and return typed document models, viewers mount
into a DOM element, and anything host-specific (file access, printing, asset URLs) is injected
through small interfaces.

```
                          ┌───────────────────────────┐
                          │     omni-viewer-core      │
                          │  parsers + viewers (TS)   │
                          │   published on npm        │
                          └─────────────┬─────────────┘
                                        │
       ┌────────────────┬───────────────┼───────────────┬────────────────┐
       │                │               │               │                │
 ┌─────┴─────┐   ┌──────┴──────┐  ┌─────┴─────┐  ┌──────┴──────┐  ┌──────┴──────┐
 │  VS Code  │   │   Chrome    │  │  Obsidian │  │     Web     │  │  JetBrains  │
 │  / Cursor │   │  extension  │  │  plugin   │  │     app     │  │ IDE plugin  │
 └───────────┘   └─────────────┘  └───────────┘  └─────────────┘  └─────────────┘
                                                                   (separate JVM
                                                                    implementation)
```

The JetBrains plugin is a separate JVM/Java implementation rather than a consumer of the
TypeScript core, so its format coverage and feature set differ from the others.

---

## Platforms at a glance

| Platform | Repository | Get it | Issues |
|---|---|---|---|
| **Shared core (library)** | [omni-viewer-core](https://github.com/battlecook/omni-viewer-core) | [npm](https://www.npmjs.com/package/omni-viewer-core) | [Issues](https://github.com/battlecook/omni-viewer-core/issues) |
| **VS Code / Cursor** | [vscode-omni-viewer](https://github.com/battlecook/vscode-omni-viewer) | [VS Code Marketplace](https://marketplace.visualstudio.com/items?itemName=battlecook.omni-viewer) · [Open VSX](https://open-vsx.org/extension/battlecook/omni-viewer) | [Issues](https://github.com/battlecook/vscode-omni-viewer/issues) |
| **Chrome** | [omni-viewer-chrome](https://github.com/battlecook/omni-viewer-chrome) | [Chrome Web Store](https://chromewebstore.google.com/detail/omni-viewer/mbhllahknhjahklfinmbnmdlpeommegd) | [Issues](https://github.com/battlecook/omni-viewer-chrome/issues) |
| **Obsidian** | [omni-viewer-obsidian](https://github.com/battlecook/omni-viewer-obsidian) | Manual install (see repo) | [Issues](https://github.com/battlecook/omni-viewer-obsidian/issues) |
| **Web** | private | [omni-viewer-web.web.app](https://omni-viewer-web.web.app/) | **[Issues here](https://github.com/battlecook/omni-viewer/issues)** ↓ |
| **JetBrains IDEs** | private | [JetBrains Marketplace](https://plugins.jetbrains.com/plugin/28550-omni-viewer) | **[Issues here](https://github.com/battlecook/omni-viewer/issues)** ↓ |

> **Where do I file a bug?**
> Use the platform's own issue tracker when it has one. The **Web** and **JetBrains** source
> repositories are private, so their issues come here — see [Reporting issues](#reporting-issues).

---

## 1. omni-viewer-core — shared engine

[github.com/battlecook/omni-viewer-core](https://github.com/battlecook/omni-viewer-core) · MIT · TypeScript

The parsing and rendering core shared by the VS Code, Chrome, Obsidian, and web builds.
Published to npm, so you can embed the same viewers in your own app.

```sh
npm install omni-viewer-core
```

> **Status: 0.x pre-release.** APIs may change between minor versions.

Heavy format libraries are **optional peer dependencies** — install only what your formats
need (`pdfjs-dist` for PDF, `jszip` + `docx-preview` for Word, `mermaid` for Mermaid, …), and
your bundler never has to resolve the rest.

Parsers are pure functions over bytes with dependencies injected:

```ts
import { parseExcel } from 'omni-viewer-core/parsers/excel';
import * as XLSX from 'xlsx';

const { result } = parseExcel(bytes, { xlsx: XLSX });
if (result.status !== 'failed') {
    console.log(result.document.sheetNames);
}
```

Viewers follow the same shape — `mount*Viewer(input, container, ctx, deps)` per format, plus
`self-loading` variants that dynamically import their own dependencies.

**Format coverage**

| Category | Formats |
|---|---|
| Documents | PDF, Word (DOCX, legacy DOC), HWP/HWPX, PowerPoint (PPTX, legacy PPT), Markdown, LaTeX (structure + math preview, not typesetting) |
| Data & spreadsheets | Excel, CSV/TSV, JSON, JSONL/NDJSON, YAML, TOML, Parquet, Avro, HDF5, MATLAB MAT, Safetensors, GGUF, ONNX, Protocol Buffers, ReqIF, SQLite |
| Media & graphics | Audio (waveform/spectrogram), video, images, Photoshop PSD |
| Engineering & automotive | CAN DBC, AUTOSAR ARXML, ASAM A2L, Vector ASC/BLF, ASAM MDF 4 (MF4), PCAP/PCAPNG, ROS bag, STEP (STP) |
| Diagrams & GIS | Mermaid, PlantUML, ESRI Shapefile |
| Archives | ZIP-family listing and safe entry preview |

<details>
<summary><strong>Note on <code>xlsx</code> (Excel / embedded workbooks)</strong></summary>

The Excel paths require **`xlsx` >= 0.20.2**. The npm registry's `xlsx` stops at 0.18.5, which
has known vulnerabilities (prototype pollution, ReDoS). SheetJS distributes patched builds from
its own CDN:

```sh
npm install https://cdn.sheetjs.com/xlsx-0.20.3/xlsx-0.20.3.tgz
```
</details>

---

## 2. vscode-omni-viewer — VS Code & Cursor

[github.com/battlecook/vscode-omni-viewer](https://github.com/battlecook/vscode-omni-viewer) · MIT · TypeScript

The most complete build. Opens supported files as custom editors directly in VS Code and Cursor.

- **Install**: [VS Code Marketplace](https://marketplace.visualstudio.com/items?itemName=battlecook.omni-viewer) · [Open VSX Registry](https://open-vsx.org/extension/battlecook/omni-viewer)
- **Issues**: [vscode-omni-viewer/issues](https://github.com/battlecook/vscode-omni-viewer/issues)

Highlights beyond plain viewing:

- **Audio** — WaveSurfer.js waveform + spectrogram, region selection and loop, playback speed,
  volume, and full stream info (sample rate, channels, bit depth). Formats: MP3, WAV, PCM (raw
  s16le mono 16 kHz), AIFF/AIF/AIFC, AMR/AWB, OGG, FLAC, AC3, AAC, M4A.
- **Image** — 10–500% zoom, rotate, flip, fit-to-screen, brightness/contrast/saturation/grayscale
  filters with presets, and save-filtered-image back into the workspace.
- **PSD** — layer/group tree with per-layer visibility, single-layer modal inspection, composite
  vs. layer-by-layer redraw, transparency checkerboard (via [ag-psd](https://github.com/Agamnentzar/ag-psd)).
- **Video** — loop regions, 0.25–4x speed, 10s skip, keyboard shortcuts.
- **PDF** — annotate, stamps, signatures, page reorder, merge, save / save-as.
- **CSV** — in-place cell editing written back to the file.
- **ML models** — ONNX graph topology with node inspection and searchable node/tensor/IO tables;
  GGUF metadata, tensor index, and quantization summary read without touching tensor payloads.
- **Content-signature rerouting** — files whose bytes don't match their extension still open in
  the right viewer.

> Codec-dependent playback still relies on the VS Code webview browser engine.

---

## 3. omni-viewer-chrome — Chrome extension

[github.com/battlecook/omni-viewer-chrome](https://github.com/battlecook/omni-viewer-chrome) · MIT · TypeScript

Manifest V3 extension that views images, documents, media, archives, and data/engineering files
right in the browser — **no uploads**. Registers as a Chrome **file handler**, so supported files
open in Omni Viewer instead of downloading.

- **Install**: [Chrome Web Store](https://chromewebstore.google.com/detail/omni-viewer/mbhllahknhjahklfinmbnmdlpeommegd)
- **Issues**: [omni-viewer-chrome/issues](https://github.com/battlecook/omni-viewer-chrome/issues)
- **Permissions**: `storage` only, plus host access limited to the Omni Viewer share service and
  Google identity/storage endpoints used by it. Parsing and rendering happen locally.

---

## 4. omni-viewer-obsidian — Obsidian plugin

[github.com/battlecook/omni-viewer-obsidian](https://github.com/battlecook/omni-viewer-obsidian) · MIT · TypeScript

Obsidian port of the VS Code extension — view and, for several formats, edit files directly
inside your vault. The same bundle runs on **desktop, Android, and iOS**.

- **Install**: manual for now — `npm install && npm run build`, then copy `main.js`,
  `manifest.json`, and `styles.css` into `<vault>/.obsidian/plugins/omni-viewer/` and enable
  **Omni Viewer** under Settings → Community plugins. Full steps in the repo README.
- **Issues**: [omni-viewer-obsidian/issues](https://github.com/battlecook/omni-viewer-obsidian/issues)

**Activation caveat**: Obsidian core already claims some extensions (md, pdf, images, audio,
video, …) and the plugin cannot take those over. For them, use the file context menu →
**Open with … Viewer**. Everything else (csv, zip, parquet, dbc, hwp, xlsx, …) opens with
Omni Viewer by default.

Commands (`Cmd`/`Ctrl`-P): `Refresh viewer`, `Share current file`, `Open shared link`.

**Mobile limits** — RAR/7z/DMG/system-tar extraction, ffmpeg transcoding, LibreOffice PDF
fallback, and legacy DOC rendering are desktop-only native helpers. GGUF is desktop-first
(mobile must read the whole file into memory, so only files under 512 MB open).

---

## 5. omni-viewer-web — browser app

**Live app: [omni-viewer-web.web.app](https://omni-viewer-web.web.app/)** · Nuxt 4 + Vue 3 + TypeScript

Zero-install viewer that runs **100% client-side — files are never uploaded to a server.**
Drop a file in the browser and read it.

> The source repository is **private**, so please file web issues in
> **[this repository](https://github.com/battlecook/omni-viewer/issues)** — see
> [Reporting issues](#reporting-issues).

| Viewer | Formats | Key features |
|---|---|---|
| Audio | MP3, WAV, OGG, FLAC, AAC, M4A, WebM | Waveform, spectrogram, region loop, 0.5–2x speed, per-channel stereo view |
| Image | JPG, PNG, GIF, BMP, WebP, SVG | 10–500% zoom, rotate, flip, fit to screen |
| Video | MP4, WebM, OGG | Playback controls, 0.5–2x speed, fullscreen |
| CSV / TSV | CSV, TSV | Table view, full-text search, column sort, pagination |
| Excel | XLSX, XLS | Multi-sheet tabs, search, pagination |
| Word | DOCX, DOC | DOCX → HTML via mammoth.js; legacy DOC staged parser (experimental) |
| PowerPoint | PPTX, PPT | Slide-by-slide text extraction; legacy PPT limited (convert to `.pptx`) |
| PDF | PDF | Page-by-page canvas rendering, zoom |
| HWP | HWP, HWPX | Document view and editing via rhwp |
| JSONL | JSONL, NDJSON, JSON Lines | Line list with pretty-printed JSON popover |
| Parquet | Parquet | Table view, search, sort, pagination |
| PSD | PSD | Composite preview, dimensions, layer list ([ag-psd](https://github.com/Agamnentzar/ag-psd)) |
| Code | JSON, YAML, Python, JS, TS, HTML, CSS, MD, … | Syntax highlighting (highlight.js) |

---

## 6. intellij-omni-viewer — JetBrains IDEs

**[JetBrains Marketplace — Omni Viewer](https://plugins.jetbrains.com/plugin/28550-omni-viewer)** · Java

Plugin for IntelliJ IDEA, WebStorm, PyCharm, GoLand, and other JetBrains IDEs. This is a
separate JVM implementation, so its feature set differs from the TypeScript-based builds.

> The source repository is **private**, so please file JetBrains issues in
> **[this repository](https://github.com/battlecook/omni-viewer/issues)** — see
> [Reporting issues](#reporting-issues).

**Free features**

- **Audio** — play/pause/stop, progress bar with time display for MP3, WAV, OGG, FLAC, M4A, AAC, WMA.
- **Parquet** — table view with schema, real-time search across columns, column sort, JSON export,
  clipboard copy; handles very large datasets.
- **Diagrams** — PlantUML (`.puml`, `.plantuml`, `.pu`, `.iuml`) and Mermaid (`.mmd`, `.mermaid`)
  preview with source editing in a split editor and SVG/HTML copy.
- **PowerPoint** — rendered slides, thumbnail navigation, text search. Animations, transitions,
  embedded media playback, and some SmartArt/advanced effects are not reproduced.
- **PDF** — files up to 3 pages are free.

**Premium features** (paid license via JetBrains Marketplace)

- **Media Player** — MP4, MOV, WebM playback with seek, time slider, volume, fullscreen.
  AVI, MKV, WMV, and FLV are not supported (JavaFX Media limitation).
- **Word** — DOC/DOCX text extraction with search, highlight, and find next/previous. Text only:
  tables, images, and formatting are not rendered and links are plain text.
- **PDF beyond 3 pages** — larger PDFs show a 3-page preview; the rest requires a license.
  Rendered pages, thumbnail navigation, password prompt, and metadata.

---

## File sharing

The VS Code, Chrome, and Obsidian builds can upload a copy of the selected file to the Omni
Viewer share service and hand you a link; opening a shared link downloads the file back locally.

- **Limits**: max **10 MB** per file, link expires after **5 minutes**.
- **Privacy Policy**: <https://omni-viewer-web.web.app/privacy/>

Ordinary viewing never uploads anything — sharing is the only path that sends bytes off your
machine, and only when you explicitly trigger it.

---

## Reporting issues

Most platforms have their own tracker — use it when it exists:

| For | File here |
|---|---|
| Shared core / npm package | [omni-viewer-core/issues](https://github.com/battlecook/omni-viewer-core/issues) |
| VS Code / Cursor | [vscode-omni-viewer/issues](https://github.com/battlecook/vscode-omni-viewer/issues) |
| Chrome extension | [omni-viewer-chrome/issues](https://github.com/battlecook/omni-viewer-chrome/issues) |
| Obsidian plugin | [omni-viewer-obsidian/issues](https://github.com/battlecook/omni-viewer-obsidian/issues) |
| **Web app** | [omni-viewer/issues](https://github.com/battlecook/omni-viewer/issues/new) (this repo) |
| **JetBrains IDE plugin** | [omni-viewer/issues](https://github.com/battlecook/omni-viewer/issues/new) (this repo) |

The `omni-viewer-web` and `intellij-omni-viewer` repositories are private, so issues cannot be
opened on them directly. When filing here, **name the platform in the title or body** (e.g.
`[JetBrains]` or `[Web]`) and include:

- Platform and version (IDE/browser/Obsidian version, plugin or extension version)
- OS
- File format and, if possible, a small sample file that reproduces the problem
- What you expected vs. what happened

---

## License

MIT for the public repositories: `omni-viewer-core`, `vscode-omni-viewer`,
`omni-viewer-chrome`, `omni-viewer-obsidian`, and `intellij-omni-viewer`. See each repository
for its own `LICENSE` and third-party notices.

## Support

If Omni Viewer is useful to you, sponsorship helps keep it maintained:

[![Sponsor](https://img.shields.io/badge/Sponsor-%E2%9D%A4-red)](https://github.com/sponsors/battlecook)
[![Ko-fi](https://img.shields.io/badge/Ko--fi-Support-ff5e5b)](https://ko-fi.com/eyedealisty)
