# ISR Loop Visualizer

An interactive, single-file explainer for the classic embedded control loop:

> **continuous voltage → ADC samples it into bits → core reformats / computes → PWM clocks it back out → GaN power stage drives a rail**

Built to make one point land for a non-specialist: *the interrupt period isn't a knob you turn for "speed" — it decides whether your output tracks reality or just photographs it.* Slide the ISR period from **20 µs** down to **1 µs** and watch the sample-and-hold staircase tighten onto the curve, the output ripple collapse, and the verdict flip from **STROBED** to **REAL-TIME**.

![screenshot](docs/screenshot.png)

## What it shows

- **Pipeline band** — the five stages with live readouts: the analog source, the ADC strobe, the core's binary word, the PWM duty, and the GaN rail filling a little battery. Bit packets flow ADC → core → PWM at the real tick rate.
- **Oscilloscope** — the true analog voltage (cyan), the ADC sample-and-hold staircase + strobe lines (amber), and the delivered rail (green), with the tracking error shaded red between them.
- **PWM drive** — the actual gate waveform, one PWM period per interrupt, with the resulting switching frequency.
- **Metrics + verdict** — loop rate, `T_isr / τ_load` ratio, samples per signal cycle, output ripple, tracking RMS error, control bandwidth, and a plain-English verdict.

## Why 1 µs ≫ 5 µs

The load (LC filter + battery) has a **fixed physical time constant** `τ_load ≈ 3 µs`. Everything keys off the ratio `r = T_isr / τ_load`:

| T_isr | r = T_isr/τ | Output ripple | Verdict |
|------:|:-----------:|:-------------:|:--------|
| 20 µs | 6.7× | ~12.6 Vpp | STROBED |
| 5 µs  | 1.7× | ~2.5 Vpp  | STROBED (marginal) |
| 2 µs  | 0.67× | ~0.4 Vpp | MARGINAL |
| 1 µs  | 0.33× | ~0.1 Vpp | REAL-TIME |

At **5 µs** the loop period is *longer* than the load can hold steady, so the rail drifts open-loop between updates — ripple scales with `(T_isr/τ)²`. At **1 µs** you're correcting ~3× per load time-constant, so the rail is glued to the command. That's the whole "buy GaN, switch faster, shrink the passives, close the loop cycle-by-cycle" argument in one picture.

> The numbers are a teaching model, not a SPICE deck. `(T_isr/τ)²` ripple scaling is the right *shape* (buck ripple ∝ 1/f_sw²); absolute values are tuned for legibility.

## Recording a GIF for slides

Hit **● Record 3s GIF**. It captures the diagram canvas (15 fps, 3 s) and downloads `isr-loop-<period>us.gif` — drop it straight into PowerPoint. `gif.js` is vendored locally (`vendor/gif.js`) and the worker is inlined, so recording works even when you just double-click `index.html` — no server, no CDN.

## Running it

Just open `index.html`. No build, no dependencies, no network needed. Keep the `vendor/` folder next to it.

For a quick local server if you prefer one:
```bash
python3 -m http.server 8000   # then visit http://localhost:8000
```

## License

MIT — see `LICENSE`.

TTA
