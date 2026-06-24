# Airgap QR Transfer

Send a file from one device to another with **no Wi-Fi, no Bluetooth, no network connectivity of any kind** — using nothing but a screen and a camera.

Two single-file, dependency-free HTML pages:

- **`sender.html`** — runs in a browser on the device that has the file. Encodes it into a stream of QR codes and displays them on a loop.
- **`receiver.html`** — runs in a browser on the device with a camera. Watches the screen, reconstructs the file, and lets you download it.

Both files are fully self-contained: all required libraries are inlined directly into the HTML. Once loaded, neither page makes any network request, ever. They work identically on desktop and mobile (Android/iOS), in any modern browser.

---

## Quick start

### 1. Open the sender

On the device that has the file, just double-click `sender.html` (or open it via `File → Open` in your browser). No server needed for the sender.

1. Choose a file.
2. Pick a quality preset (**Balanced** is the safe default — see [Choosing a preset](#choosing-a-preset)).
3. Click **Start Broadcasting**.
4. A QR code will start flashing on screen. Leave the tab in the foreground.

### 2. Open the receiver

The receiver needs camera access, and browsers only grant camera access on secure origins (`https://` or `http://localhost`) — **not** on a plain `file://` page. So you need a tiny local web server on the receiving device:

```bash
# from the folder containing receiver.html
python3 -m http.server 8000
```

Then open **`http://localhost:8000/receiver.html`** in your browser.

> No Python? Any static file server works: `npx serve .`, `php -S localhost:8000`, VS Code's "Live Server" extension, etc. The only requirement is that the URL is `localhost` or HTTPS. **On Android specifically**, see [Serving the receiver page from an Android device](#serving-the-receiver-page-from-an-android-device) below for app-based alternatives that don't require a terminal.

1. Click **Start Scanning** and grant camera permission.
2. Point the camera at the sender's screen, filling as much of the frame as comfortably possible, steady and well-lit.
3. Watch the progress bar climb. There is no pairing, handshake, or acknowledgement — the sender just broadcasts forever, and the receiver passively accumulates enough data to reconstruct the file.
4. When it hits 100%, a **Download File** button appears. Click it, and you're done.
5. On the sender, click **Stop** once you've confirmed the receiver finished.

### 3. Sending another file

Click **New Transfer** on the sender (picks a new session, lets you choose a new file) and **Reset Session** on the receiver (clears its decoder state). You don't need to reload either page.

---

## Choosing a preset

| Preset | QR version | Data per frame | Notes |
|---|---|---|---|
| **Dense** | 30 (137×137 modules) | ~1.7 KB | Fastest, but needs a steady hand, good focus, and good lighting. Best for laptop screen → phone camera at close range. |
| **Balanced** *(default)* | 22 (105×105 modules) | ~1.0 KB | Good speed/reliability trade-off for most phone cameras at arm's length. |
| **Robust** | 15 (77×77 modules), higher error-correction | ~0.4 KB | Slowest, but will scan reliably even with a poor camera, bad lighting, or some hand shake. Use this if Balanced is failing to make progress. |

If progress stalls or the receiver shows a lot of rejected frames, switch to a more robust preset on the sender (it can be changed before or during a transfer — changing it mid-transfer effectively starts a new session).

Typical real-world throughput is roughly **3–15 KB/s** depending on preset, distance, lighting, and camera quality — a few minutes for a small document, longer for multi-megabyte files. This is intentionally a last-resort, no-infrastructure channel, not a fast one.

---

## Reading the receiver's stats panel

| Stat | Meaning |
|---|---|
| Blocks solved | How many of the file's internal chunks have been recovered so far. |
| Pending equations | Partially-useful frames received that don't fully resolve a block yet (normal, will clear up as more frames arrive). |
| Frames scanned / decoded OK / rejected / duplicate | Rejected = camera caught a frame badly enough that its checksum failed (harmless, just wasted); duplicate = same frame seen twice (harmless). A high rejection rate is a sign to slow down, get closer, improve lighting, or switch to a more robust preset. |
| Last packet degree | Internal detail — how many file-chunks were XORed together in the most recently decoded frame. |

---

## Requirements & limitations

- **Browser**: any reasonably modern Chrome, Safari, Firefox, or Edge — desktop or mobile. No extensions or installs.
- **Receiver needs a camera** and a secure origin (`localhost` or HTTPS) to access it.
- **Sender needs a screen** large/bright enough for the receiving camera to read comfortably.
- Both tabs should stay in the **foreground** — mobile browsers throttle or suspend background tabs, which will pause the sender's loop or starve the receiver's scan loop.
- This is a **one-way visual channel** — there's no return path, no acknowledgements, and that's by design (true air-gap compatible: works even if the receiver has no way to signal back to the sender at all).
- Practical file size range: anything from a few bytes up to tens of megabytes works in principle; very large files just mean a longer broadcast. There's no hard cap baked in beyond the protocol's 65,535-block ceiling (see technical section).

---

## Serving the receiver page from an Android device

If the receiving device is an Android phone and you'd rather not use a terminal/`python3 -m http.server`, you can use a dedicated app instead. All of these let you point a folder containing `receiver.html` at a local HTTP server and then just open `http://localhost:PORT/receiver.html` (or `http://<phone-ip>:PORT/receiver.html` from the same device) in the browser.

### Simple, dedicated HTTP file-server apps (open source)

**On F-Droid** (open source, installable without a Google account):

| App | What it does | Source |
|---|---|---|
| **Simple HTTP Server** (`com.phlox.simpleserver`) — also on Play Store | Pick a folder, serve it as static HTTP content; shows IP/port, some versions QR-code the URL | [GooglePlay](https://play.google.com/store/apps/details?id=com.phlox.simpleserver) [DevSite](https://shttps.phlox.dev/releases/) |
| **Share via HTTP** (`com.MarcosDiez.shareviahttp`) | Not a standalone server app — adds a "Share via HTTP" entry to Android's share sheet for any file/folder; spins up a temporary server just for that item, with an optional QR code of the URL | [F-Droid](https://f-droid.org/packages/com.MarcosDiez.shareviahttp/) · [GitHub](https://github.com/marcosdiez/shareviahttp) |
| **lWS – lightweight Web Server** | Under 100 KB, GPL-3.0, static file serving, configurable doc root + port, QR code of the URL | [F-Droid](https://f-droid.org/en/packages/net.basov.lws.fdroid/) |
| **ServeIt** | Flutter-based, serves a folder (default `/sdcard/Downloads`) on a configurable port, scannable QR | [F-Droid](https://f-droid.org/en/packages/com.example.flutter_http_server/) |
| **HTTP FS file server** | More featured — also supports WebDAV/HTTPS, password protection, QR code | [GitHub](https://github.com/Tiarait/HTTP-FS-file-server) |

<!--
**Directly from GitHub** (build or sideload the APK; not necessarily on F-Droid):

- [`android-http-server`](https://github.com/piotrpolak/android-http-server) (piotrpolak) — zero-dependency Java servlet container, GPLv3
- [`android-http-server`](https://github.com/zahidaz/android-http-server) (zahidaz) — modern Ktor/Kotlin-coroutines based, actively maintained
-->
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

## Overview of the pipeline

```
SENDER                                                    RECEIVER
──────                                                    ────────
file bytes
   │
   ▼
deflateRaw (pako)              ──── compressed bytes ───►
   │
   ▼
split into k fixed-size blocks
   │
   ▼
Robust Soliton fountain encoder
   │  (infinite stream of XOR-combined,
   │   degree-tagged, explicit-index packets)
   ▼
QR-encode each packet (byte mode)
   │
   ▼
render to <canvas>, loop forever  ──camera──►  jsQR decode
                                                    │
                                                    ▼
                                          parse packet, CRC32 check
                                                    │
                                                    ▼
                                          incremental GF(2) Gaussian
                                          elimination (online solver)
                                                    │
                                          (repeat until k blocks solved)
                                                    ▼
                                          reassemble compressed buffer
                                                    │
                                          CRC32-verify whole payload
                                                    ▼
                                          inflateRaw (pako)
                                                    │
                                          verify length, trigger download
```

## Why a fountain code instead of "just send chunks in order"

A naive approach — chunk the file, QR-encode each chunk, cycle through them — requires the receiver to catch **every single chunk** at least once. Camera-based scanning is lossy: motion blur, autofocus hunting, screen glare, and timing jitter mean some fraction of displayed frames will simply never be read. With sequential chunking, a single permanently-missed frame means the file can never be completed without some kind of retransmission request — which requires a return channel. There is no return channel here, on purpose (this works even if the receiving device has no way whatsoever to signal back).

**Fountain codes** (this project uses a **Luby Transform / LT code** with a **Robust Soliton** degree distribution) solve this elegantly:

- Each transmitted packet is not "block #7" but a **random linear combination** (XOR, since we're in GF(2)) of a small, randomly-chosen subset of the file's blocks.
- The sender just keeps generating and displaying *new* random packets forever, in a loop, with no notion of which ones the receiver has or hasn't seen.
- The receiver needs to successfully capture **any** sufficiently large set of these packets — not a specific one — to mathematically reconstruct the original blocks via linear algebra (Gaussian elimination over GF(2), described below).
- The number of packets needed to fully decode `k` blocks is `k` plus a small overhead (empirically ~10–60% depending on `k` and luck), regardless of *which* packets were lost. Loss just means "wait a bit longer," never "transfer is stuck forever."

This is exactly the property needed for a one-way, no-feedback, lossy optical channel.

### Degree distribution and why indices are sent explicitly

LT codes need a *degree distribution* — a probability distribution over "how many blocks should this packet XOR together." The classic choice, used here, is the **Robust Soliton distribution**: mostly low degrees (lots of degree-1 and degree-2 packets, which resolve instantly or near-instantly) with a deliberate "spike" of higher-degree packets to guarantee, with high probability, that the decoding process never stalls before finishing.

A textbook LT code implementation often derives both the *degree* and the *specific source-block indices* for a packet from a shared deterministic pseudo-random seed (e.g. the packet's sequence number), so the receiver can regenerate the same indices without the sender ever transmitting them — saving bandwidth.

**This implementation deliberately does not do that.** The seed-based approach requires the sender's and receiver's PRNG and degree-distribution math to agree *bit-for-bit*, which in turn depends on `Math.log`/`Math.sqrt` producing identical floating-point results across potentially two different JavaScript engines (e.g. desktop Chrome vs. mobile Safari). The ECMAScript spec does **not** guarantee bit-identical transcendental math across implementations. A single floating-point disagreement would cause the receiver to assume the wrong set of indices for a packet — silently corrupting the linear system with no way to detect it from a CRC alone (the CRC only validates that the *XORed payload bytes* weren't corrupted by the camera; it can't detect "the receiver decided the wrong indices belong to this XOR").

So instead, **each packet explicitly carries its own degree and the exact list of source-block indices it XORs together** (as part of the packet bytes, protected by the same CRC32 as the payload). This costs a small amount of extra bandwidth (a few bytes per packet) but makes the protocol's correctness independent of any floating-point determinism assumption — sender and receiver only need to agree on byte layout, which is checked and enforced by CRC32, not on reproducing each other's arithmetic. Given the explicit goal of being "failsafe," this tradeoff was made deliberately in favor of provable correctness over a marginal bandwidth saving.

### The Robust Soliton CDF (sender-side only)

```
rho[1]    = 1/k
rho[i]    = 1/(i·(i-1))                          for i = 2..k
S         = c · ln(k/δ) · √k                     (spike width, c=0.1, δ=0.05)
tau[i]    = S/(k·i)                              for i = 1..(k/S − 1)
tau[k/S] += S · ln(S/δ) / k                       (the spike itself)
mu[i]     = (rho[i] + tau[i]) / Σ(rho+tau)
CDF[i]    = Σ_{j≤i} mu[j]
```

For each packet, the sender draws `u = Math.random()`, finds the smallest `i` such that `CDF[i] ≥ u` via binary search, and that `i` is the packet's degree. Then it picks `i` distinct block indices uniformly at random (rejection sampling: keep drawing random indices into a `Set` until it has `i` distinct ones).

Because this entire calculation happens **only on the sender**, and the result is transmitted explicitly rather than re-derived, there is no cross-engine determinism requirement anywhere in the protocol.

## Wire format

Two packet types, distinguished by a leading type byte. All multi-byte integers are big-endian (`DataView` default).

### Header packet (type `0x00`)

Re-broadcast periodically (every 12 frames by default) so the receiver can pick it up quickly regardless of when it starts watching.

| Field | Size | Description |
|---|---|---|
| type | 1 B | `0x00` |
| sessionId | 4 B | Random per-file session identifier |
| fileNameLen | 1 B | Length of the filename in bytes (UTF-8, truncated to 255 B) |
| fileName | variable | UTF-8 filename |
| originalSize | 4 B | Size of the file before compression |
| compressedSize | 4 B | Size after `deflateRaw` |
| blockSize | 2 B | Fixed size of every block (last block zero-padded) |
| k | 2 B | Number of blocks |
| crcCompressed | 4 B | CRC32 of the entire compressed payload |
| headerCrc | 4 B | CRC32 of all preceding header bytes |

### Data packet (type `0x01`)

| Field | Size | Description |
|---|---|---|
| type | 1 B | `0x01` |
| sessionId | 4 B | Must match the header's session |
| packetId | 4 B | Monotonically increasing counter (used for dedup, not for index derivation) |
| degree | 1 B | Number of source blocks XORed into this packet (1–60) |
| indices | `2 × degree` B | The source block indices (uint16 each) |
| payload | `blockSize` B | XOR of the listed source blocks |
| payloadCrc | 4 B | CRC32 of the payload bytes |

Per-frame overhead is therefore `14 + 2·degree` bytes on top of the raw block payload — small relative to the ~400–1700 byte payloads used in practice.

## QR encoding details

- Uses [`qrcode-generator`](https://github.com/kazuhikoarase/qrcode-generator) (MIT) in **byte mode**, with its default UTF-8 `stringToBytes` function overridden to a raw 1-byte-per-char-code mapping — necessary because the packets are arbitrary binary data, not text, and the library's default would mangle any byte ≥ 0x80 by re-encoding it as multi-byte UTF-8.
- A fixed QR version/error-correction-level pair is used per preset (rather than auto-sizing), so every frame in a session has identical dimensions — this keeps the receiver's camera focus/exposure stable instead of constantly re-adjusting to varying QR densities.
- Rendering is done by hand onto a `<canvas>`, module-by-module via `fillRect`, with the canvas's pixel dimensions chosen as an **exact integer multiple** of the QR's module count (`moduleCount + 2×quietZone`). This guarantees every module renders as a crisp, pixel-aligned square with no sub-pixel blending — a real and common failure mode when QR codes are rendered via scaled `<img>` tags or non-integer canvas scaling, which introduces anti-aliased gray edges that confuse camera-based decoders.
- A 4-module quiet zone border is included, as required by the QR spec for reliable finder-pattern detection.

QR error-correction level **L** (the lowest, ~7% recoverable damage) is used deliberately even though higher levels exist. Since the fountain code already provides redundancy *across* frames, paying for *within-frame* redundancy via a higher ECC level (M/Q/H) would only reduce usable payload per frame for limited benefit — the "Robust" preset is the exception, using ECC **M**, intended specifically for situations where individual frames are likely to be partially unreadable (bad lighting/focus) rather than relying purely on inter-frame redundancy.

## Decoding: incremental Gaussian elimination over GF(2)

This is the core algorithm that makes the receiver tolerant of frames arriving in **any order**, with **any pattern of loss**, with no time limit. It's mathematically equivalent to maintaining a partial row-echelon form of a linear system, where "rows" are XOR-equations and "addition" is XOR.

### Data structures

- `solved: Map<blockIndex, Uint8Array>` — blocks that are fully known.
- `pivots: Map<pivotIndex, {indices: Set<number>, data: Uint8Array}>` — partially-reduced equations, keyed by the smallest still-unknown block index they reference.

### Algorithm (`addRow`)

When a new packet arrives with index-set `I` and XORed payload `D`:

1. **Reduce against known blocks.** For every index in `I` that's already in `solved`, XOR its known value into `D` and remove that index from `I`. (If a block is already known, it no longer contributes unknown information to this equation.)
2. If `I` is now empty, this packet was fully redundant — discard it (in principle its data should now be all-zero if everything is consistent; this is not currently asserted, but a non-zero residue here would indicate a corrupted packet that slipped past the CRC, which is astronomically unlikely given a 32-bit checksum).
3. Otherwise, **reduce against existing pivot rows**: let `p` = the smallest index still in `I`.
   - If a pivot row already exists for `p`: XOR that pivot row's index-set and data into `I`/`D` (symmetric difference for the index set, XOR for the data), and go back to step 2/3 with the updated `I`/`D`.
   - If no pivot exists for `p`: this row *becomes* the new pivot for `p`.
     - If `I` now contains only `p` itself (degree 1), block `p` is fully solved: move it into `solved` and trigger a **cascade**.

### Cascade

Solving a new block can make previously-stuck pivot rows solvable. The cascade walks every existing pivot row, removes the newly-solved index wherever it appears (XORing in its now-known value), and recursively solves and cascades any row that drops to degree 1 as a result. This is a standard **peeling decoder** step, here folded into the same structure as the Gaussian elimination rather than implemented as a separate pass — meaning there's no separate "try peeling, then fall back to full elimination" logic; one algorithm handles both, and is guaranteed to find every block as soon as the receiver has accumulated `k` linearly-independent equations (i.e., full rank), regardless of whether that happens via the fast "instant degree-1" path or the slower full-elimination path.

This is genuinely just **Gaussian elimination over GF(2)**, maintained incrementally one row at a time instead of batched — which is what allows it to process frames as they arrive, live, with no fixed batch size or retry logic.

### Completion and integrity verification

Once `solved.size === k`:

1. Concatenate all `k` solved blocks in order, then truncate to the header's `compressedSize` (undoing the zero-padding applied to the final block on the sender side).
2. Compute CRC32 over that trimmed buffer and compare against the header's `crcCompressed`. If it doesn't match, the reconstruction is **not** accepted — the decoder keeps listening for more frames rather than offering a corrupted file. (In practice this should essentially never trigger, since every individual packet was already CRC-checked before being fed into the solver — this is a final end-to-end belt-and-suspenders check.)
3. `pako.inflateRaw` to decompress, and verify the resulting length matches the header's `originalSize` as a sanity check.
4. Only then is the download button enabled.

## Session handling

Each prepared file gets a random 32-bit `sessionId`. Because there's no handshake, the receiver must guard against two things:

- **Stale data from a previous transfer.** Solved blocks and pivot rows are namespaced by session implicitly (any reset clears the whole solver state).
- **A single misread frame falsely appearing to start a new session.** A camera misread could, in principle, decode to a syntactically valid packet with a corrupted `sessionId` field that happens to pass its own CRC by chance (1-in-4-billion-ish for a random 32-bit collision against an unrelated CRC, but not literally impossible). To avoid one such fluke nuking an in-progress transfer, a *different* session ID has to appear **three times** before the receiver commits to switching to it. A genuine new transfer (sender clicked "New Transfer") will naturally repeat its new session ID every frame, satisfying this within a fraction of a second; a one-off fluke will not repeat and is ignored.

## Compression

`pako.deflateRaw` (raw DEFLATE, no zlib/gzip header) is used to shrink the payload before fountain-encoding, since fewer bytes directly means fewer blocks/frames/seconds. Raw deflate (rather than full gzip) is used purely to shave a few header/footer bytes that would otherwise be redundant with this protocol's own framing and checksums. Highly-compressible files (text, structured data) benefit enormously; already-compressed or random/encrypted data will see `compressedSize ≈ originalSize` and that's expected.

## Libraries used (all vendored/inlined, MIT/Apache-2.0)

| Library | Purpose | License |
|---|---|---|
| [`qrcode-generator`](https://github.com/kazuhikoarase/qrcode-generator) | QR encoding (sender) | MIT |
| [`jsQR`](https://github.com/cozmo/jsQR) | QR decoding from camera frames (receiver) | Apache-2.0 |
| [`pako`](https://github.com/nodeca/pako) | DEFLATE compression/decompression | MIT/Zlib |

All three are included verbatim inside the HTML files — there are no `<script src="https://...">` references anywhere, by design, so the tool keeps working with zero connectivity after the initial page load.

## Known limits

- Max `k` (block count) is 65,535 due to the 16-bit block-index field. At the smallest practical block size (~400 B under the Robust preset) that's a ceiling of roughly 25 MB; with larger blocks (Dense preset, ~1.7 KB) it's closer to 100+ MB. Far larger than what's practical to transfer at a few KB/s over a camera anyway.
- Max degree per packet is capped at 60 to bound per-packet index overhead; this is independent of `k` and works fine across the full supported range.
- There is intentionally no automatic retry/ack protocol — by design, this works over a channel with **zero return path**. The only "control" signal is the human deciding when to click Stop.

</details>
