# Ballistic Audio Synthesizer v1.5

<!-- TODO: replace with a 5–15s screen capture of the UI + a link to the live build -->
<!-- e.g. [**▶ Live build**](https://<your-url>) · [**▶ Demo video**](https://youtu.be/<id>) -->

A standalone, self-contained single-file HTML/CSS/JavaScript developer tool for designing and generating procedural, physically modeled firearm sound effects via the Web Audio API. Built specifically for brutalist, ASCII-rendered retro-sci-fi games, this tool bridges the gap between authentic milsim ballistics and industrial-organic sound design.

No samples are loaded, looped, or layered - every shot is computed on the fly from the ballistic parameters you set, in real time, and can be bounced to a 16-bit/48 kHz WAV through the exact same DSP code you hear live.

---

## Overview & Purpose

Instead of relying on heavy, static audio sample files, this workbench uses real-time digital signal processing (DSP) to simulate the actual physical forces of a weapon firing: thermodynamic gas expansion, mechanical sear drops, structural chassis resonance, and ballistic attachments.

It lets you tweak those forces in real time - 46 parameters, 42 of them on sliders, all exposed over a terminal-style CLI - and export clean JSON profiles for integration into TypeScript/WebGL game engines.

---

## How the sound is emulated

Most procedural gun sounds are built the way you'd build a kick drum: an oscillator swept down in pitch, an amplitude envelope, plus a noise burst for "texture". In a dense mix at low level that passes. Up close it collapses, because it is missing every physical signature of an actual discharge:

| Physical reality | What a synth shortcut does instead |
|---|---|
| A gunshot has **no pitch**. Low-frequency content is gas expanding against air and metal - broadband and stochastic. | A single fundamental the ear locks onto and immediately flags as artificial. |
| The pressure wave and the mechanical crack are **phase-coherent** - same event, same instant. | Independent layers that cancel and reinforce at arbitrary points: thin, wobbly shots. |
| A microphone diaphragm and an eardrum are driven into **asymmetric** overload. | Symmetric soft-clipping → even harmonics that read as fuzz, not overload. |
| Gas cools and pressure dissipates by **rate**, not to a fixed duration. | A stretched fixed-length envelope that never scales correctly. |

The workbench answers each of those directly:

1. **Propellant shockwave - the low end.** Shaped pink/brown noise → swept low-pass (300 Hz → cavitation floor) → 1–3 ms attack from *exactly zero* → exponential decay expressed as the rate β → tanh saturation. There is no oscillator anywhere in the chain, so there is no fundamental for the ear to catch - it reads as pressure, not pitch.
2. **Phase lock.** The shockwave, the Friedlander blast lobe, the bore-gas burst and the supersonic crack all start from one shared sample-accurate origin, on both the realtime and offline paths. Layers that start together don't cancel each other.
3. **Friedlander blast wave + bore gas.** The muzzle event is the standard analytic overpressure model - sharp positive phase, negative phase behind it - alongside brownian gas expansion, with a bore-resonance term tied to the barrel length.
4. **Asymmetric diaphragm overload.** The main waveshaper distorts the positive and negative pressure phases differently (0.55 / 0.35 split), with independent drive and bias - the lopsided harmonic grit symmetric clipping cannot produce.
5. **Attachments are filter chains, not reverb sends.** A suppressor damps the blast, not the action - the mechanical cycle stays full level, which is how a real weapon behaves and a common tell when skipped.
6. **One engine for live and export.** The realtime graph and the `OfflineAudioContext` bounce call the same builder functions. What you hear is what you render.
7. **A −0.8 dB safety limiter** on the master, so aggressive settings can't clip the output.

---

## Core Architecture & Synthesis Bays

The audio engine separates every gunshot report into distinct, physically motivated layers:

* **`BAY_01` - Primary Bore / Shock Front**
  * Parameter matrix plus mechanical transients (0–5 ms): high-passed noise and synthetic clicks simulating sear drops, firing-pin strikes, and bolt-carrier group cycling.
  * Shockwave & gas expansion (5–30 ms): the swept-filter propellant shockwave (no oscillator), the analytic Friedlander blast lobe, and brownian bore-gas expansion.

* **`BAY_02` - Biquad Notch / Diaphragm / Modifiers**
  * Live transfer-response curve; HPF → notch → body filter stage.
  * Asymmetric soft-clipping diaphragm with independent drive and bias (air overload / element saturation).
  * Supersonic crack - independent velocity, sharpness and gain, disableable for subsonic loads.
  * **Suppressors:** steep low-pass (800 Hz default) with blast gain reduction, mechanical click isolated.
  * **Muzzle brakes / compensators:** peaking filter (brake) and high-shelf (comp) with a vent-energy boost for high-frequency lateral/vertical gas venting.
  * **Environmental tail:** algorithmic early-reflection tap bank plus damped late tail (bunker / corridor / open / urban), configurable ground-slap delay, tail-choke ducking and ear-reflex ducking.

* **`BAY_03` - Cartridge Energy, Caliber & Chassis Mass**
  * **Caliber scale:** dynamically scales the sub-bass cavitation floor (**58 Hz → 36 Hz**) and exponential decay based on projectile energy, from 9 mm subsonic to .50 BMG. A loadout selector carries published service-load data per platform (bullet mass, muzzle velocity, muzzle energy, powder charge in grains): case capacity drives the cavity and decay, propellant charge drives `subEnergy`, and ½mv² muzzle energy drives peak overpressure - √-compressed, so a 9 mm lands near 1.25× and a .50 near 2.40×.
  * **Chassis resonance:** modulates mechanical bolt-clacks and a bandpass ring-modulator tail by frame mass - polymer frames ring high and short (~2.4 kHz, ~195 ms), milled brutalist steel rings low and lingers (~900 Hz, ~387 ms).
  * **Barrel length → exit pressure:** short barrels drive the soft-clipper harder, simulating raw, unburnt-powder flash. Anchored to exactly 1.0× at the 14.5″ reference; `exitPressure` scales the coupling and can be zeroed to decouple it.

The center column holds the oscilloscope (pressure / spectrum), a 64-band log FFT waterfall and the terminal CLI; the left column holds the ordnance archive (preset explorer) and the PNG → ASCII ordnance profile panel.

---

## Technical Specifications

* **Technology stack:** pure vanilla HTML, CSS and JavaScript - one file, no framework, no build step.
* **Audio core:** native Web Audio API - `AudioContext` + `OfflineAudioContext`, `BiquadFilterNode`, `WaveShaperNode`, `DelayNode`, `DynamicsCompressorNode`.
* **Export:** 16-bit PCM WAV at 48 kHz, plus a JSON parameter manifest (including derived BAY_03 values, so a patch is reproducible from the manifest alone).
* **ASCII pipeline:** PNG → monochrome ASCII plate on an `OffscreenCanvas`, target under 40 ms per conversion, explicit errors (`E_EMPTY` / `E_FORMAT` / `E_TOO_SMALL`), baked per preset and included in the export.
* **Dependencies:** **zero external libraries.** Completely self-contained within a single file for instant local execution or hosting.

---

## Quick Start & Usage

1. Clone or download the single-file tool into your local workspace.
2. Open `index.html` directly in any modern browser (or serve it - recommended, see below).
3. Use the **ASCII terminal UI** sliders to tweak weapon mechanics, cartridge mass, and attachments in real time.
4. Press **Spacebar** or click **FIRE** to preview the procedural audio output and inspect the real-time oscilloscope and FFT waterfall.
5. Click **EXPORT JSON** for the parameter payload, or **RENDER WAV** for a 48 kHz bounce. Keys **1–6** quick-load the six factory profiles; type `help` in the center console for the CLI.

---

## Presets

Six factory ordnance profiles, plus a `localStorage` archive for your own:

| ID | Weapon | Fire mode |
|---|---|---|
| `glock19` | Glock-19 9mm | semi |
| `m4a1` | M4A1 5.56 NATO | burst |
| `barrett50` | Barrett .50 BMG | semi |
| `rem870` | Rem 870 12ga slug | semi |
| `mp5sd` | MP5SD suppressed | auto |
| `brutalist_rail` | Hypervelocity coil | semi |

Factory profiles ship **no default ASCII artwork** - an ordnance profile is something you bake from your own image.

---

## Terminal CLI

The center console is a real command line, not a text box:

```
fire [count]                          discharge the weapon
preset <id> | load <id> | list        load or list presets
preset save <folder>/<name>           save current config to the archive
set <param> <value>                   set any parameter, clamped to its range
set caliber [0-100]                   BAY_03 cartridge macro
mode [semi|burst|auto]                fire mode
rate [100-1200]                       cyclic rate, RPM
attachment [none|suppressor|brake|comp]
env [bunker|corridor|open|urban]
crack [on|off] | tailchoke [on|off]
ascii [preview|view|bake|clear]
export                                open the JSON export manifest
wav                                   render and download a 48 kHz WAV
help
```

---

## Game Engine Integration Example

Exported JSON profiles can be loaded directly into your TypeScript/WebGL game loop - the parameters live under the manifest's `parameters` key:

```typescript
interface WeaponAudioProfile {
  subFreq: number;        // cavity floor, Hz - 58 = 9mm subsonic … 36 = .50 BMG
  subEnergy: number;      // propellant charge multiplier (0.55 – 1.75)
  subDamping: number;     // decay rate β, s⁻¹ - higher dumps pressure faster
  chassisMass: number;    // frame mass index, 0 – 1
  barrelLength: number;   // inches - drives the exit-pressure soft-clip drive
  attachment: 'none' | 'suppressor' | 'muzzleBrake' | 'comp';
}

export class RuntimeWeaponAudio {
  constructor(private ctx: AudioContext) {}

  public play(profile: WeaponAudioProfile) {
    // Map JSON parameters to a live Web Audio node graph:
    // shaped noise → swept lowpass → tanh shaper → asymmetric diaphragm
  }
}
```

The manifest also carries derived values (effective soft-clip drive, ring sweep, decay-to−60 dB, and the matched ammunition loadout) under `parameters.bay03CartridgeEnergyCaliberChassis`, so nothing has to be re-derived at load time.

---

## Running it

No build step, no dependencies, no npm.

```bash
# just open it
start index.html          # Windows
open index.html           # macOS

# or serve it (recommended - OffscreenCanvas prefers a real origin)
npx serve .               # or: python3 -m http.server 8000
```

Then open `http://localhost:8000`.

**Requirements.** A browser with Web Audio API and `OfflineAudioContext` - Chrome, Edge, Firefox or Safari, current versions. Audio starts suspended until the first click, per autoplay policy.

---

## Repository layout

```
index.html    the entire application - markup, styles, DSP engine, CLI, ASCII pipeline
README.md     this file
```

---

## Credits

Designed and built by **[@ch2rlie000](https://github.com/ch2rlie000)** though ALL advanced mathematics & non-linear equations governing weapon physics were implemented with the aid of **Qwen 3.8 Flash** (VSCode + OpenCode), I am NOT a mathmagician & still very much so learning, but this was intended to be made with sophistication, deep understanding & as a huge learning curve for me.

---

## License

MIT - see [LICENSE](LICENSE).
