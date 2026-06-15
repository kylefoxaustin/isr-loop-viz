# CLAUDE.md — ISR Loop Visualizer

Handoff notes for a Claude Code session. Single-file interactive explainer of an
ADC → ISR core → PWM → GaN control loop, built collaboratively with Claude in a
claude.ai session (June 2026).

## What this is

A self-contained `index.html` (canvas + vanilla JS, no framework, no build) that
animates a continuous voltage being sampled by an ADC, reformatted in a core,
emitted as PWM, and filtered by a GaN power stage into an output rail. The whole
point is pedagogical: changing the **interrupt period** visibly changes whether
the output tracks reality or strobes it.

## Architecture

Everything is one canvas (`#scope`, 1040×660 internal, DPR-scaled). Drawn in four
horizontal bands so a single canvas grab = a complete, self-explanatory figure
(important — the GIF recorder captures only the canvas):

1. **Pipeline** (`drawPipeline`) — 5 stage boxes + connectors + flowing bit packets.
2. **Oscilloscope** (`drawScope`) — analog truth (cyan), ADC sample-and-hold +
   strobe lines (amber), delivered rail (bold smooth line + translucent ripple
   **band**), error shading (faint red). The rail + band are colored by command
   **freshness**: green where a fresh ISR update just landed, red where it coasts
   on held/stale data. `drawGlitch` overlays the disturbance verdict (see below).
3. **PWM strip** (`drawPWM`) — gate waveform. Two carrier modes (`state.pwmMode`):
   `'isr'` = one period per interrupt (all green); `'fixed'` = 1 MHz carrier where
   replayed/stale periods are red and the fresh period after each ISR tick is green.
4. **Metrics** (`drawMeta`) — chips + verdict banner + slow-mo factor.

### Simulation model (the honest part)

- `vsig(t)` — analog source: sine (`Tsig=64µs`, `center=24V`, `amp=9V`) plus an
  optional decaying disturbance (`kickT`, `kickAmp`).
- ADC samples at every `nextTick` (`isrUs` apart), quantizes to `bits` (8 or 12).
- **One-period transport delay** is modeled explicitly: the sample latched at tick
  *k* (`uPending`) is only *applied* (`uApplied`) at tick *k+1*. This is why loop
  rate == latency.
- Output `vout` integrates toward `uApplied` with **fixed** load time constant
  `cfg.tauLoad = 3µs` (the LC filter + battery — it's physical, it does NOT scale
  with the ISR knob; that's the whole lesson).
- Switching ripple ∝ `(T_isr/τ)²` (right shape for buck ripple ∝ 1/f_sw²; magnitude
  tuned for legibility, clamped via `rippleVpp()`). It is **not** baked into the
  rail line — `voutHist` stores `{t, v:vout, rip:rippleVpp()*0.5}` and the scope
  draws the smooth `v` as a line inside a `±rip` band. (This is what made the 20µs
  view legible — the old code added a triangle ripple straight onto the line and it
  whipped up and down like noise.)
- **Freshness** (`drawScope`/`drawPWM`): a carrier period / rail segment is "fresh"
  when `(t mod isrUs) < Tcarrier`, where `Tcarrier = 1µs` in fixed mode else `isrUs`.
  Fresh → green, stale → red. So per-ISR mode is all-green; fixed-1MHz at 20µs is
  ~95% red.
- **Disturbance verdict** (`drawGlitch`): on a kick, the next ADC sample is
  `ts = ceil(kickT/isrUs)*isrUs`; `blind = ts-kickT` and `captured = exp(-blind/9)`
  (9µs = the disturbance decay τ in `vsig`). ≥0.6 CAUGHT (green), ≥0.3 PARTIAL
  (amber), else MISSED (red). Phase-dependent — a slow loop *occasionally* gets
  lucky, which is honest.
- Verdict from `r = isrUs/tauLoad`: ≤0.45 REAL-TIME, ≤1.5 MARGINAL, else STROBED.

Virtual time advances in `advance(dtv)` with a 0.2µs sub-step integrator decoupled
from rAF jitter. `state.speed` is µs of virtual time per real second; the on-canvas
"≈ N× slow-motion" label is `1e6/speed` (real µs are invisible — we say so).

### GIF recording

`gif.js` is vendored at `vendor/gif.js` (loaded via `<script src>`, fine from
file://). The **worker** is base64-inlined into `index.html` as `GIF_WORKER_B64`
and turned into a Blob URL at record time (`makeWorkerUrl`) — this is the workaround
that makes worker-based encoding run from `file://` (plain worker files are blocked
there). Record = 15 fps × 3 s, scaled 0.66, auto-downloads `isr-loop-<period>us.gif`.

If you ever re-vendor gif.js, re-inject the worker:
```bash
python3 - <<'PY'
import base64
b64=base64.b64encode(open('vendor/gif.worker.js','rb').read()).decode()
h=open('index.html').read().replace('__GIF_WORKER_B64__', b64)  # only on a fresh placeholder
open('index.html','w').write(h)
PY
```
(The placeholder is already consumed in the committed file — to re-inject, restore
the `"__GIF_WORKER_B64__"` token first, or replace the existing base64 string.)

## Gotchas / things already fixed

- **Packet smear at fast ticks** — at 1–2µs many packets are in flight; drawing a
  hex label per packet turned into an unreadable smear. Fixed: flowing dots for all
  in-flight packets, hex label on the *newest* one only (`drawPackets`).
- **Divide-by-zero placeholder** — an early packet-progress line divided by
  `isrUs*0`. Removed; packet motion is driven by real wall-clock in `drawPackets`.
- **Google Fonts** (JetBrains Mono / Major Mono Display) load over the network; the
  canvas uses a `monospace` fallback so it renders fine offline. Don't make canvas
  text depend on web-font load timing.
- The `0V` y-axis label sits right on the scope/PWM panel seam — cosmetic, low
  priority.

## Design system

Blueprint/CAD aesthetic, shared with the `robot-power-budget` project:
- bg `#060b11` / panel `#0c1622`, cyan `#35d0ff`, amber `#ffb23e`,
  green `#46e08a`, red `#ff5d6c`; faint cyan grid.
- Display face Major Mono Display (h1 only), body/data JetBrains Mono.
- Corner-bracket panels, segmented-button controls, focus-visible amber outlines.

## Common tasks

- **Re-tune the 1µs vs 5µs contrast** → `cfg.tauLoad` and the `rippleVpp()` /
  `verdict()` thresholds. Bigger `tauLoad` softens the contrast; smaller sharpens it.
- **Add an ISR period option** → add a `<button data-isr="N">` in `#isr`.
- **Add an ADC width** → add a `<button data-bits="N">` in `#bits`. The core word
  splits in half adaptively (`<12 bits` uses 12px font, else 10px) and the packet
  hex label box auto-sizes, so wide words don't overflow.
- **Re-tune the glitch verdict** → thresholds + the `exp(-blind/9)` decay in
  `drawGlitch` (9µs must match the disturbance τ in `vsig`).
- **Change ADC range / signal** → `cfg.center/amp/Tsig/vmin/vmax`.
- **Robotics scenario** (instead of power conversion) → swap the GaN-stage glyph
  and rename the rail to a joint position/torque; the loop math is identical.

## Owner voice

Kyle signs off **TTA** (Trust the Awesomeness). Match it in commits/comments/docs:
casual, technical, direct. No corporate hedging, no marketing voice.

## History

- Initial build: pipeline + scope + PWM + metrics, ISR/bits/speed controls,
  disturbance kick, play/pause, GIF recorder.
- Fix: packet smear → dots + single newest label.
- Fix: removed divide-by-zero packet placeholder.
- Validated physics headlessly across 20/10/5/2/1 µs; screenshotted all three
  regimes; confirmed GIF encode path runs from file://.
- Added **fixed 1 MHz PWM carrier** mode (red = replayed/stale, green = fresh) +
  an **18-bit** ADC option (adaptive core-word split, auto-sized packet hex).
- Reworked the rail render: **ripple as a translucent band** around a smooth line
  (killed the 20µs zigzag); rail + band colored by **freshness** (green/red).
- Folded freshness into the kick: **blind window + CAUGHT/PARTIAL/MISSED** verdict
  (`drawGlitch`).
- Validated all of the above headlessly (Chrome `--virtual-time-budget`; glitch
  injected via a temp `setTimeout` in throwaway copies since `state` is closure-scoped).

---
TTA
