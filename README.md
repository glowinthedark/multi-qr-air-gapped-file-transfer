# Airgap QR Transfer

Send a file from one device to another with **no Wi-Fi, no Bluetooth, no network connectivity of any kind** — using nothing but a screen and a camera (and, optionally, a speaker and a microphone).

Two single-file, dependency-free HTML pages:

- **[🔗 🌐 `sender.html`](https://glowinthedark.github.io/multi-qr-air-gapped-file-transfer/sender.html)** — runs in a browser on the device that has the file. Encodes it into a stream of QR codes and shows them vsync-paced.
- **[🔗 🌐 `receiver.html`](https://glowinthedark.github.io/multi-qr-air-gapped-file-transfer/receiver.html)**  — runs in a browser on the device with a camera. Watches the screen, reconstructs the file, verifies it, plays a success chime, and downloads it automatically.

Both files are fully self-contained: all required libraries are inlined directly into the HTML. Once loaded, neither page makes any network request, ever. They work identically on desktop and mobile (Android/iOS), in any modern browser.

**Protocol v2** (current) is a significant rework of v1 for speed and reliability:

- **Systematic first pass** — every block is sent once, verbatim, before any coding; a clean capture completes with zero overhead.
- **Seed-derived fountain packets** — block indices are re-derived on the receiver from `(sessionId, packetId, k)`, so no index list travels over the wire (and the v1 bug where high-degree packets silently overflowed the QR capacity is structurally impossible).
- **Metadata piggybacked in every data packet** — the receiver can start (and finish) decoding from the very first frame it sees; the header frame only carries the filename.
- **Frame intervals down to 50 ms**, vsync-paced, with the next frame pre-built during the current one. (A 2×2 multi-QR grid and an extra-dense preset were field-tested and removed: real-world cameras could not resolve them reliably.)
- **Frame-change beacon + identical-frame skip** — the receiver cheaply detects stale frames instead of re-decoding them.
- **Native `BarcodeDetector`** (hardware QR decoding, multi-code per frame) is used automatically once it has been verified binary-safe against jsQR; all decoding runs in a Web Worker off the main thread, with full-resolution ROI tracking.
- **Audio ACK back-channel (optional)** — the receiver's speaker sends short FSK messages that the sender's microphone decodes: live progress, targeted resend requests for the last few missing blocks, and a DONE signal that auto-stops the sender.
- **Screen wake locks** on both sides, camera zoom/focus controls, native `CompressionStream` compression when available.

---

## Quick start

### 1. Open the sender

On the device that has the file, just double-click `sender.html` (or open it via `File → Open` in your browser). No server needed for the sender.

1. Choose a file.
2. Pick a quality preset (**Balanced** is the default — see [Choosing a preset](#choosing-a-preset)) and a frame interval.
3. Click **Start Broadcasting**. Leave the tab in the foreground. **Fullscreen** makes the codes bigger and easier to scan.
4. Audio ACKs are **on by default**: the first Start requests microphone permission — grant it and the sender will show the receiver's live progress, resend exactly the blocks it misses, and stop by itself when the receiver confirms completion. Denying (or the prompt not appearing) just means one-way mode; use **Enable ACK Mic** to retry. ⚠️ Like the receiver's camera, the microphone only works on `https://` or `http://localhost` — if you opened `sender.html` as a plain `file://` page, serve it from the same local server as the receiver to use audio ACKs.

### 2. Open the receiver

The receiver needs camera access, and browsers only grant camera access on secure origins (`https://` or `http://localhost`) — **not** on a plain `file://` page. So you need a tiny local web server on the receiving device:

```bash
# from the folder containing receiver.html
python3 -m http.server 8000
```

Then open **`http://localhost:8000/receiver.html`** in your browser.

> No Python? Any static file server works: `npx serve .`, `php -S localhost:8000`, VS Code's "Live Server" extension, etc. The only requirement is that the URL is `localhost` or HTTPS. **On Android specifically**, see [Serving the receiver page from an Android device](#serving-the-receiver-page-from-an-android-device) below for app-based alternatives that don't require a terminal.

1. Click **Start Scanning** and grant camera permission.
2. Point the camera at the sender's screen, filling as much of the frame as comfortably possible, steady and well-lit. Use the zoom slider if your camera exposes one.
3. Watch the progress bar climb. Transfers usually complete in one pass; losses are repaired automatically by the fountain stream (and by audio-ACK-targeted resends if the sender's mic is enabled).
4. At 100% the file is CRC-verified, a **success chime plays, and the download starts automatically**. The **Download File** button stays available if you need the file again.
5. The sender stops by itself if its ACK mic is on; otherwise click **Stop** on the sender once the receiver is done.

### 3. Sending another file

Click **New Transfer** on the sender (lets you choose a new file). Every click of **Start Broadcasting** mints a fresh session id — even a rebroadcast of the same file — and the receiver detects the new session on its own after a few frames and resets automatically, so you normally don't need to touch the receiver at all (**Reset Session** exists for forcing a manual reset). You don't need to reload either page.

---

## Choosing a preset

| Preset                   | QR version                                  | Data per code | Notes                                                                                                                                        |
| ------------------------ | ------------------------------------------- | ------------- | -------------------------------------------------------------------------------------------------------------------------------------------- |
| **Balanced** *(default)* | 22 (105×105 modules)                        | 974 B         | Good speed/reliability trade-off for most phone cameras at arm's length.                                                                     |
| **Robust**               | 15 (77×77 modules), higher error-correction | 383 B         | Slowest, but will scan reliably even with a poor camera, bad lighting, or some hand shake. Use this if Balanced is failing to make progress. |

Theoretical rate at Balanced is ~9.7 KB/s at 10 frames/s, more at shorter intervals. Real-world throughput depends on the camera; if the receiver's "rejected" count climbs or progress stalls, slow the interval down before reaching for the Robust preset. (An extra-dense version-30 preset and a 2×2 multi-QR grid existed briefly and were removed — in field tests typical cameras never decoded them.)

---

## Reading the receiver's stats panel

| Stat                                        | Meaning                                                                                                                                                                 |
| ------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Decoder                                     | Which decode path is active: jsQR in a worker, jsQR on the main thread (fallback), or the native BarcodeDetector once verified.                                         |
| Blocks solved                               | How many of the file's internal chunks have been recovered so far.                                                                                                      |
| Pending equations                           | Partially-useful packets received that don't fully resolve a block yet (normal, will clear up as more packets arrive).                                                  |
| Camera frames processed / identical skipped | Skipped frames were detected as unchanged since the last decode (the sender never repeats a frame, so unchanged = stale) — high skip counts are healthy, they save CPU. |
| Packets decoded OK / rejected / duplicate   | Rejected = checksum failed (harmless, just wasted); duplicate = same packet seen again within ~1.5 s (camera oversampling; harmless).                                   |
| Last packet degree                          | Internal detail — how many file-chunks were XORed together in the most recently decoded packet.                                                                         |
| Audio ACKs                                  | Whether the speaker back-channel is enabled / sending DONE.                                                                                                             |

---

## Requirements & limitations

- **Browser**: any reasonably modern Chrome, Safari, Firefox, or Edge — desktop or mobile. No extensions or installs. (The native-decoder and worker paths are progressive enhancements; everything falls back gracefully.)
- **Receiver needs a camera** and a secure origin (`localhost` or HTTPS) to access it.
- **Sender needs a screen** large/bright enough for the receiving camera to read comfortably.
- Both tabs should stay in the **foreground** — mobile browsers throttle or suspend background tabs. Both pages take a screen wake lock while active so the display won't sleep mid-transfer.
- The visual channel is **one-way**; the audio ACK channel is an optional enhancement, not a requirement — with it disabled the protocol degrades to pure broadcast exactly like v1.
- Practical file size range: anything from a few bytes up to tens of megabytes works in principle; very large files just mean a longer broadcast. There's no hard cap baked in beyond the protocol's 65,535-block ceiling (see technical section).

---

## Serving the receiver page from an Android device

If the receiving device is an Android phone and you'd rather not use a terminal/`python3 -m http.server`, you can use a dedicated app instead. All of these let you point a folder containing `receiver.html` at a local HTTP server and then just open `http://localhost:PORT/receiver.html` (or `http://<phone-ip>:PORT/receiver.html` from the same device) in the browser.

### Simple, dedicated HTTP file-server apps (open source)

**On F-Droid** (open source, installable without a Google account):

| App                                                                    | What it does                                                                                                                                                                                  | Source                                                                                                                      |
| ---------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------- |
| **Simple HTTP Server** (`com.phlox.simpleserver`) — also on Play Store | Pick a folder, serve it as static HTTP content; shows IP/port, some versions QR-code the URL                                                                                                  | [F-Droid](https://f-droid.org/packages/com.phlox.simpleserver/)                                                             |
| **Share via HTTP** (`com.MarcosDiez.shareviahttp`)                     | Not a standalone server app — adds a "Share via HTTP" entry to Android's share sheet for any file/folder; spins up a temporary server just for that item, with an optional QR code of the URL | [F-Droid](https://f-droid.org/packages/com.MarcosDiez.shareviahttp/) · [GitHub](https://github.com/marcosdiez/shareviahttp) |
| **lWS – lightweight Web Server**                                       | Under 100 KB, GPL-3.0, static file serving, configurable doc root + port, QR code of the URL                                                                                                  | [F-Droid](https://f-droid.org/en/packages/net.basov.lws.fdroid/)                                                            |
| **ServeIt**                                                            | Flutter-based, serves a folder (default `/sdcard/Downloads`) on a configurable port, scannable QR                                                                                             | [F-Droid](https://f-droid.org/en/packages/com.example.flutter_http_server/)                                                 |
| **HTTP FS file server**                                                | More featured — also supports WebDAV/HTTPS, password protection, QR code                                                                                                                      | [GitHub](https://github.com/Tiarait/HTTP-FS-file-server)                                                                    |

**Directly from GitHub** (build or sideload the APK; not necessarily on F-Droid):

- [`android-http-server`](https://github.com/piotrpolak/android-http-server) (piotrpolak) — zero-dependency Java servlet container, GPLv3
- [`android-http-server`](https://github.com/zahidaz/android-http-server) (zahidaz) — modern Ktor/Kotlin-coroutines based, actively maintained

### Well-known general-purpose apps with a built-in "expose via HTTP" feature

These are apps many people already have installed for other reasons, which happen to include a local web-server mode — handy since it avoids installing anything new just for this:

- **X-plore File Manager** — long-standing, widely used file manager with a built-in Wi-Fi/HTTP server mode for browsing/transferring files from a desktop (or another phone's) browser.
- **Solid Explorer** — popular dual-pane file manager with a "Web Server" share option that works the same way.
- **MiXplorer** — another major file manager with an HTTP server feature for LAN transfers.
- **Termux** — not a file-server app per se, but a genuine terminal (installs Python, Node, etc.), so it can run `python3 -m http.server` exactly like a desktop. Open source, widely used by power users; available from [F-Droid](https://f-droid.org/packages/com.termux/)/GitHub rather than Play Store.
- **KDE Connect** — mainly a phone↔desktop integration tool (notifications, clipboard sync, etc.), but its device file-browsing/SFTP feature can also be used to reach files on the phone from a desktop browser or file manager, if you already use it for other phone-PC syncing.

**Practical recommendation:** for the *receiver* side of this tool, **Simple HTTP Server** or the built-in server in **X-plore**/**Solid Explorer** are the fastest path to getting `receiver.html` reachable at `http://localhost:PORT/...` without touching a terminal. **Share via HTTP** is also handy on the *sender* side if you want to hand someone `sender.html` itself (or the file the receiver downloaded) over the system share sheet with zero setup.

---

<details>
<summary><strong>📖 Full technical explanation (click to expand)</strong></summary>

## Overview of the pipeline (protocol v2)

```
SENDER                                                    RECEIVER
──────                                                    ────────
file bytes
   │
   ▼
deflate-raw (native CompressionStream,
pako fallback)
   │
   ▼
split into k fixed-size blocks
   │
   ▼
systematic pass: packetId 0..k-1 =                camera (1080p/60 requested,
block i verbatim; then endless fountain           requestVideoFrameCallback)
packets whose indices BOTH sides derive                   │
from (sessionId, packetId, k) via a                       ▼
shared deterministic PRNG                         decode Web Worker:
   │                                              identical-frame skip (16×16
   ▼                                              luminance signature) → jsQR
1 or 4 QR codes per display frame,                with full-res ROI tracking, or
vsync-paced (rAF), pre-built during the           native BarcodeDetector once
previous frame's window, plus an                  verified binary-safe
alternating beacon bar                                    │
   │                                                      ▼
   └────────────── light ────────────────►       parse packet, CRC32 check,
                                                 re-derive indices from packetId
   ┌───────────── sound ─────────────────┐               │
   │ (optional FSK ACKs: progress,       │               ▼
   │  missing-block NACKs, DONE)         │       incremental GF(2) Gaussian
   ▼                                     │       elimination (Uint32 XOR lanes)
sender mic demodulates ACKs:             │               │
targeted resends + auto-stop             └──── speaker ──┤
                                                          ▼
                                                 reassemble, CRC32-verify,
                                                 inflate, verify length,
                                                 chime + auto-download
```

## Why a fountain code instead of "just send chunks in order"

A naive approach — chunk the file, QR-encode each chunk, cycle through them — requires the receiver to catch **every single chunk** at least once. Camera-based scanning is lossy: motion blur, autofocus hunting, screen glare, and timing jitter mean some fraction of displayed frames will simply never be read. With sequential chunking, a single permanently-missed frame means the file can never be completed without a retransmission request.

**Fountain codes** (this project uses a **Luby Transform / LT code**) solve this elegantly:

- Each coded packet is a **random XOR combination** of a randomly-chosen subset of the file's blocks.
- The sender keeps generating *new* packets forever with no notion of which ones the receiver has seen.
- The receiver needs **any** sufficiently large set of packets — not specific ones — to reconstruct all blocks via Gaussian elimination over GF(2).
- Loss just means "wait a bit longer," never "transfer is stuck forever."

v2 additionally makes the stream **systematic**: `packetId 0..k-1` are defined as "block `i` verbatim" (degree 1). On a good link the whole file arrives in the first pass with zero coding overhead and zero solver work; the fountain packets that follow exist purely to repair whatever the camera missed.

### Seed-derived indices and cross-engine determinism

v2 does **not** transmit the per-packet index list. Both sides derive the degree and the block indices from `(sessionId, packetId, k)` using an identical PRNG (an xorshift32 core seeded via integer hashing — `Math.imul`, shifts, xors: exact integer ops everywhere).

The classic objection (and the reason v1 sent explicit indices) is that the degree distribution's construction uses floating-point math, and `Math.log` is **not** guaranteed bit-identical across JavaScript engines — a single one-ulp disagreement between, say, desktop Chrome and mobile Safari could make the receiver assume the wrong indices for a packet and silently corrupt the linear system. v2 removes the hazard instead of the feature:

- The distribution is built using **only IEEE-754 correctly-rounded operations** (`+ - * /`, `Math.sqrt`, `Math.floor`), which the spec *does* guarantee bit-identical everywhere.
- The natural log needed by the Robust Soliton parameters is computed by `detLog()` — an atanh power series with a *fixed* iteration count. Whether or not the series result equals the true logarithm to the last bit is irrelevant; what matters is that every engine computes the *same* sequence of correctly-rounded operations and therefore the *same* double.
- The two files carry byte-identical copies of `detLog`, `buildDegreeCDF`, `pktRandom`, and `indicesForPacket` (verifiable with `diff`), and the whole-file CRC32 check at completion remains as the end-to-end backstop.

This recovers `1 + 2×degree` bytes per packet (up to 121 B — 12% of a Balanced frame in v1), makes every data packet a fixed size (v1 packets with degree > 8 exceeded the QR capacity and silently failed to render), and removes the need for any degree cap.

### The degree distribution

70% of the probability mass is a standard **Robust Soliton** (`c = 0.1`, `δ = 0.05`); the remaining 30% is placed on a single "dense" degree `min(⌊k/2⌋, 512)` (floored at 8). Rationale: the receiver runs *full* Gaussian elimination, not just peeling. In the endgame — a handful of blocks missing after the systematic pass — sparse low-degree packets mostly hit already-solved blocks (coupon-collector waste), whereas a dense row is almost always linearly independent. Simulation across `k` = 10…8000 at 15–25% frame loss shows ~10% reception overhead with this mixture, versus 50%+ for a plain soliton feeding a GE decoder in the same regime.

For each packet, `u = rng()/2³²` selects a degree by binary search over the CDF; the indices are then drawn without replacement using the same per-packet PRNG stream (with a deterministic sequential fallback if rejection sampling stalls). `packetId < k` bypasses all of this (systematic).

## Wire format

Two packet types, distinguished by a leading type byte. All multi-byte integers are big-endian (`DataView` default). Type `0x01` (the v1 data packet) is retired; the receiver logs a version-mismatch warning if it sees one.

### Header packet (type `0x00`)

Sent as the first two frames and every 40th frame thereafter. Since v2 piggybacks all numeric metadata in every data packet, the header's only unique cargo is the **filename** — a transfer completes fine without ever catching one (the file is then named `received_file.bin`).

| Field          | Size     | Description                                                 |
| -------------- | -------- | ----------------------------------------------------------- |
| type           | 1 B      | `0x00`                                                      |
| sessionId      | 4 B      | Random per-file session identifier                          |
| fileNameLen    | 1 B      | Length of the filename in bytes (UTF-8, truncated to 255 B) |
| fileName       | variable | UTF-8 filename                                              |
| originalSize   | 4 B      | Size of the file before compression                         |
| compressedSize | 4 B      | Size after deflate                                          |
| blockSize      | 2 B      | Fixed size of every block (last block zero-padded)          |
| k              | 2 B      | Number of blocks                                            |
| crcCompressed  | 4 B      | CRC32 of the entire compressed payload                      |
| headerCrc      | 4 B      | CRC32 of all preceding header bytes                         |

### Data packet (type `0x02`) — fixed size, exactly fills the QR

| Field          | Size          | Description                                                                         |
| -------------- | ------------- | ----------------------------------------------------------------------------------- |
| type           | 1 B           | `0x02`                                                                              |
| sessionId      | 4 B           | Session identifier                                                                  |
| packetId       | 4 B           | `< k`: systematic (block = packetId). `≥ k`: fountain; indices derived from this id |
| k              | 2 B           | Number of blocks (piggybacked metadata)                                             |
| blockSize      | 2 B           | Block size (piggybacked)                                                            |
| compressedSize | 4 B           | (piggybacked)                                                                       |
| originalSize   | 4 B           | (piggybacked)                                                                       |
| crcCompressed  | 4 B           | (piggybacked)                                                                       |
| payload        | `blockSize` B | The block itself, or the XOR of the derived index set                               |
| packetCrc      | 4 B           | CRC32 over all preceding bytes                                                      |

Overhead is a constant **29 bytes**, so `blockSize = QR byte capacity − 29` (974 / 383 B for Balanced / Robust).

## Audio ACK back-channel (optional)

Receiver speaker → sender microphone. 16-FSK: sixteen data tones at 1500–4200 Hz (180 Hz apart), separator tone 1150 Hz, preamble alternating 4700/4400 Hz, 90 ms slots. Each message is `preamble(4 slots) · [SEP, nibble]×2·len · END(4700 Hz)` with a CRC-8 (poly `0x07`) appended:

- **DONE** `[0xD1, sessionIdLow]` — repeated 3 times after completion (enough for the sender to catch one without becoming a nuisance). The sender verifies the session byte, stops broadcasting, and plays the chime.
- **PROGRESS/NACK** `[0xA2, sessionIdLow, solved u16le, (missingIndex u16le)×0..3]` — chirped every ~8 s throughout the transfer (so the sender shows live receiver progress), tightening to every ~5 s with up to 3 missing block indices once ≥ 85% of blocks are solved. The sender pushes the missing indices to the front of its schedule as systematic resends — collapsing the "last few blocks" tail to a few frames.

Data rate is tiny (~20 bit/s) and that's fine: the QR channel carries the data; sound only carries *control*. The success chime uses notes at 523–1047 Hz, entirely below the FSK band, so it can never be mistaken for a symbol. Everything works with the channel disabled — it's an accelerator, not a dependency.

## QR encoding & display details

- [`qrcode-generator`](https://github.com/kazuhikoarase/qrcode-generator) (MIT) in **byte mode**, with `stringToBytes` overridden to a raw 1-byte-per-char-code mapping (payloads are binary, not UTF-8).
- Fixed QR version per preset so every frame has identical dimensions — keeps camera focus/exposure stable.
- Each code is generated as a **1-pixel-per-module `ImageData`** (4-module quiet zone included) and blitted with `imageSmoothingEnabled = false` at an exact integer scale — crisp, pixel-aligned modules with no anti-aliased edges, and far cheaper than per-module `fillRect`.
- Frame pacing uses `requestAnimationFrame` (vsync-aligned), with the *next* frame's codes built incrementally during the *current* frame's display window, so a swap is just a blit. Missed deadlines are counted in the stats ("Missed swap deadlines").
- Below the code, a **beacon bar** (two half-bars swapping black/white every frame) makes frame changes trivially observable.
- ECC level **L** for Balanced (inter-frame redundancy comes from the fountain code, so paying for intra-frame redundancy is usually a bad trade), **M** for Robust.

## Receiver capture & decode details

- Camera is requested at 1920×1080@60 with `facingMode: environment`; continuous autofocus is applied when supported and a zoom slider appears when the track exposes zoom.
- `video.requestVideoFrameCallback` (rAF fallback) fires once per *camera* frame; exactly one frame is in flight at a time (natural backpressure).
- All pixel work happens in a **Web Worker** (the vendored jsQR source is stored in an inert `text/plain` script tag and prepended into the worker blob; if workers/OffscreenCanvas are unavailable, it's eval'd on the main thread as a fallback).
- **ROI tracking**: full-frame search (alternating whole-frame and quadrant sweeps, so a small code anywhere in the frame is found at high effective resolution) until a code is found, then the code's region is cropped from the **full-resolution** frame — no global downscale throwing away module detail. The box follows drift and expires after repeated misses.
- **Identical-frame skip**: a 16×16 luminance signature per region; if it matches the last attempt, the decode is skipped (the sender never shows the same packet twice in a row, so an unchanged region is always stale). This is where most of the "duplicate decode" CPU waste of v1 went.
- **Native `BarcodeDetector`**: if present, it runs in *probation* alongside jsQR and must return byte-identical results 4 times before taking over (its `rawValue` is a string; on binary-safe platforms that's a 1:1 Latin-1 mapping, on others it mangles bytes ≥ 0x80 — probation detects which world we're in using real traffic, no test assets needed). Once active, it decodes full frames in hardware, multiple codes at once. Any rare packet that happens to decode as valid UTF-8 and gets mangled simply fails its CRC and is repaired by the fountain stream.

## Decoding: incremental Gaussian elimination over GF(2)

Unchanged in spirit from v1 — an online row-echelon solver where rows are XOR-equations:

- `solved: Map<blockIndex, row>` — fully known blocks.
- `pivots: Map<pivotIndex, {indices: Set, row}>` — partially-reduced equations keyed by their smallest unknown index.

New rows are first reduced against `solved`, then against colliding pivots (symmetric difference on the index set, XOR on the payload); a row that drops to degree 1 solves its block and **cascades** through all pending pivots. One algorithm covers both the fast peeling path and full elimination, and completes as soon as `k` linearly-independent equations have arrived.

v2 performance changes: payload buffers are 4-byte padded and XORed as `Uint32Array` lanes (4× fewer operations on ~1 KB blocks), and packet dedup is a small time-windowed map (~1.5 s) rather than an ever-growing set — camera oversampling duplicates are caught, while audio-ACK-triggered resends of old packet ids are correctly accepted (and redundant rows reduce to nothing for near-zero cost anyway).

### Completion and integrity verification

Once `solved.size === k`: concatenate, truncate to `compressedSize`, CRC32-verify against `crcCompressed`, `inflateRaw`, verify `originalSize`. Only then: chime, auto-download, and DONE ACKs. A failed check keeps the decoder listening rather than delivering a corrupt file.

## Session handling

Each prepared file gets a random 32-bit `sessionId`. A *different* session ID must appear in **three** CRC-valid packets before the receiver abandons an in-progress transfer for it — a genuine new transfer satisfies this within a fraction of a second, while a fluke misread cannot.

## Compression

Raw DEFLATE (`CompressionStream('deflate-raw')` when the browser has it — off the main thread and much faster on big files — with `pako.deflateRaw` as the fallback; the receiver inflates with pako). Fewer bytes directly means fewer blocks/frames/seconds. Already-compressed data will see `compressedSize ≈ originalSize`, which is expected.

## Libraries used (all vendored/inlined, MIT/Apache-2.0)

| Library                                                                 | Purpose                                    | License    |
| ----------------------------------------------------------------------- | ------------------------------------------ | ---------- |
| [`qrcode-generator`](https://github.com/kazuhikoarase/qrcode-generator) | QR encoding (sender)                       | MIT        |
| [`jsQR`](https://github.com/cozmo/jsQR)                                 | QR decoding from camera frames (receiver)  | Apache-2.0 |
| [`pako`](https://github.com/nodeca/pako)                                | DEFLATE compression/decompression fallback | MIT/Zlib   |

All are included verbatim inside the HTML files — there are no `<script src="https://...">` references anywhere, by design.

## Known limits

- Max `k` (block count) is 65,535 due to the 16-bit block-count field. At the Robust block size (383 B) that's a ceiling of roughly 24 MB compressed; at Balanced (974 B) it's ~62 MB.
- The audio ACK channel assumes the two devices are within normal speaking distance in a reasonably quiet room; in a loud environment just leave it off — the protocol degrades to pure one-way broadcast.
- Sender and receiver must be the **same protocol version** (v2 files, as shipped together). A v2 receiver logs a clear warning if it sees v1 packets.

</details>
