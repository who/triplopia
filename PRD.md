# PRD: Triplopia Simulator (MVP)

**Status:** Draft v0.1
**Date:** 2026-10-04
**Owner:** Andy Bird
**Reference prototype:** `triplopia.html` (single-file WebGL2 page, built alongside this document)
**Public site:** https://who.github.io/triplopia/ (GitHub Pages, source at https://github.com/who/triplopia)
**Claude artifact (sign-in required):** https://claude.ai/artifact/HwKw4qNMyKBmtfWbEpA8rE

---

## 1. Summary

A browser-based tool that lets a person upload any image and see a physically grounded simulation of monocular triple vision (triplopia) caused by an irregular corneal surface. The simulation is not three semi-transparent layers. It models the actual optics: three fixed focal zones on the cornea, each producing its own image at a fixed offset and with its own focus error, summed as light on the retina, with a wandering focus state that makes the copies "take turns" being the sharp, dominant one.

The primary user is the owner, who has this condition in the left eye and wants to show others what it is like. Secondary users are family, friends, and clinicians.

---

## 2. Background and problem

### 2.1 The condition as described by the owner

- Monocular triplopia in the left eye. Cause: **uneven corneal surface**. Not keratoconus, cataract, lens subluxation, or post-surgical.
- **Three copies arranged in a triangle**: one upper, two lower (lower left and lower right). The owner's sketch exaggerates the spacing for clarity. In practice the copies sit almost on top of one another, only slightly offset, so edges and strokes read as doubled or tripled rather than as three separate objects.
- **The copies do not move.** Their relative positions are fixed. They do not drift apart or converge.
- **Dominance cycles.** The eye and brain alternate which copy is the "predominant" one. The lower-right copy wins most often, the upper copy sometimes, the lower-left rarely.
- **Mush state.** Sometimes what reaches perception is a combination of all three at once, which is confusing and distorting rather than three clean images.
- **The switch is subtle.** The handover from one dominant copy to the next is usually gradual enough that it happens without the owner noticing it happen.
- **Non-dominant copies are dim, not blurry.** The two secondary copies stay close to as sharp as the dominant one. What distinguishes the dominant copy is mainly brightness.
- **No copy is ever fully dominant.** The other two never disappear. The lead is a matter of degree, not an on/off state.

### 2.2 Why opacity layers are wrong

Stacking three copies at partial opacity produces three equally crisp ghosts that never change. It cannot reproduce:

- copies that differ in sharpness from one another and over time,
- the dominance shift, which is driven by which copy is in focus,
- the "mush" state, where no copy is sharp,
- the fact that small bright lights and thin high-contrast strokes triple far more visibly than soft gradients,
- dependence on pupil size (bright room vs dim room).

### 2.3 Working optical model

An irregular cornea behaves like a lens with several locally different curvatures. Light from one point in the world passes through three distinct zones of the cornea and lands in three regions on the retina.

- **Fixed geometry.** Each zone has a fixed prismatic offset, so each copy lands at a fixed position. This matches the owner's report that the copies never move.
- **Different focal powers.** Each zone has a slightly different power. The eye's lens (accommodation) can only bring one zone into focus at a time. The in-focus zone yields a sharp copy; the others are defocused.
- **Focus drives dominance, dominance shows as brightness.** The in-focus zone is only slightly sharper than the others, but the visual system treats it as "real" and suppresses the rest, which read as dimmer copies rather than blurrier ones. As accommodation drifts and re-targets, the dominant copy changes. This is the "taking turns." The handover is a continuous, overlapping crossfade with no identifiable moment of switching.
- **Mush = focus between zones.** When accommodation sits between two zone powers, no copy is sharp. All three are moderately blurred and roughly equal in weight, and they sum into an image that does not resolve. This is the "combination of all three" state.
- **Pupil dependence.** A small pupil (bright light) samples less of the cornea, so fewer zones contribute and the effect weakens. A large pupil (dim light) exposes all zones and the effect strengthens.
- **Light adds.** The retina sums light from all zones. Compositing must be additive in linear light, with total energy conserved, never alpha-blended in sRGB.

---

## 3. Goals and non-goals

### 3.1 Goals (MVP)

1. Upload an image and see a convincing, real-time simulation of the owner's triplopia.
2. Reproduce the four signature behaviors: fixed triangle geometry, continuous milky crossfade of dominance with the lower-right copy favored, secondary copies that are dim rather than blurry, and the intermittent mush state where all three are roughly equal.
3. Expose every model parameter as a live control so the owner can tune the simulation until it matches their experience, then save that as the default preset.
4. Allow instant comparison with the unmodified image.
5. Run entirely in the browser with no backend, no upload to a server, and no install.

### 3.2 Non-goals (MVP)

- Deriving parameters from a corneal topography or wavefront scan (desired later, see Section 9).
- Simulating any other condition (diplopia, astigmatism alone, cataract glare, floaters).
- Binocular simulation or how the good eye and bad eye combine.
- Video or webcam input.
- Accounts, sharing links, cloud storage, analytics.
- Mobile-first design. It should not break on a phone, but desktop is the target.

---

## 4. Users and primary scenarios

| User | Scenario |
|---|---|
| Owner | Loads a photo of something familiar, tunes sliders until it matches, shows the live view to someone. |
| Family or friend | Is handed the screen, watches the live view, holds the "normal" button to compare, scrubs manual focus to feel the dominance shift. |
| Clinician | Views a tuned preset on a text sample and a point-light sample to understand the patient's subjective experience. |

---

## 5. Functional requirements

### 5.1 Image input

- **FR-1** Upload an image via file picker, drag and drop onto the view, or paste from clipboard.
- **FR-2** Accept common formats (PNG, JPEG, WebP, GIF first frame).
- **FR-3** Downscale images larger than 1600 px on the long side for performance, preserving aspect ratio.
- **FR-4** Ship three built-in demo images that need no upload: a text page (reading), a night scene with small bright lights, and a daytime scene with signage. The text demo loads by default.
- **FR-5** Images never leave the browser.

### 5.2 Rendering

- **FR-6** Render on the GPU (WebGL2 fragment shader). Target 60 fps at 1080p on an integrated GPU at the default quality setting.
- **FR-7** Each of the three copies is produced by convolving the source image with a geometric defocus point spread function (uniform disk) whose radius is proportional to the distance between the current focus state and that copy's focal power, scaled by pupil size, plus a small baseline blur.
- **FR-8** The three copies are summed in linear light with per-copy weights normalized to total 1, then converted back to sRGB. No alpha blending.
- **FR-9** Each copy's offset is a fixed vector expressed as a percentage of image height, multiplied by a global separation scale. Offsets do not vary over time.
- **FR-10** Disk blur sampling uses a golden-angle spiral with per-pixel rotation noise and mip-level selection to avoid banding at large radii. Quality setting selects 24, 48, or 96 taps.
- **FR-11** Pixels sampled outside the image contribute black, so copies near edges fade rather than smear the border.

### 5.3 Dominance dynamics (the "taking turns")

The handover between dominant copies must feel milky and continuous, never like a slideshow. There are therefore **no discrete switch events** in the model.

- **FR-12** Each copy carries its own slowly wandering dominance signal, implemented as smoothed Ornstein-Uhlenbeck noise with a user-set timescale ("dominance drift timescale").
- **FR-13** The three copies' weights come from a soft competition (softmax) over those signals plus a per-copy static bias derived from the "how often focus lands here" settings. Because the signals drift continuously, the winner changes by gradual crossfade, and two copies can share the lead for stretches of time.
- **FR-14** A separate, slower noise signal occasionally collapses the competition's contrast toward zero, scaled by the "indecision" setting. In that state all three copies carry similar weight, which produces the mush.
- **FR-15** The "dominance contrast" setting scales how strongly the leading copy stands out. At zero all copies are always equal.
- **FR-15a** A "minimum presence" floor guarantees every copy always keeps at least that share of the total light (default 18% each), so no copy ever vanishes however strong the leader. At the defaults the leader typically carries about 45 to 60% of the light and each secondary 18 to 30%.
- **FR-16** The focus state used for blur is the weight-averaged focal power of the three zones, plus a small sinusoidal microfluctuation. Blur therefore follows dominance smoothly rather than driving it.
- **FR-17** Optional blinks (off by default): every 4 to 9 seconds the view goes black for about 120 ms and the dominance signals are reshuffled.
- **FR-18** Manual focus mode freezes the dynamics and lets the user scrub a focus slider; copies closer in power to the chosen focus receive more weight, so an observer can feel each copy take the lead in turn.

### 5.4 Perceptual weighting

- **FR-19** Each copy's final weight is its brightness setting, times a pupil ramp (non-central copies fade out as pupil shrinks below about 5 mm and vanish near 2.5 mm), times its dominance weight from Section 5.3. Weights are normalized to total 1 so overall brightness is conserved. All three copies remain nearly equally sharp; brightness, not blur, is the visual signature of dominance.
- **FR-20** One copy is designated the central zone and is always exposed regardless of pupil. Default: lower right.

### 5.5 Controls

All controls are live sliders or selects in a side panel, with current values shown.

| Group | Control | Range | Default |
|---|---|---|---|
| Dynamics | Dominance drift timescale | 1 to 20 s | 6 s |
| Dynamics | Indecision (contrast collapse) | 0 to 100% | 30% |
| Dynamics | Microfluctuation | 0 to 0.5 | 0.03 |
| Dynamics | Dominance contrast | 0 to 100% | 60% |
| Dynamics | Minimum presence of every copy | 0 to 33% | 18% |
| Dynamics | Manual focus toggle + position | -1.2 to 1.2 | off, 0.6 |
| Dynamics | Blinks toggle | on/off | off |
| Optics | Pupil diameter | 2 to 8 mm | 4.5 mm |
| Optics | Separation scale | 0 to 4× | 1× |
| Optics | Defocus blur of non-dominant copies | 0 to 6 | 0 (off) |
| Optics | Baseline blur | 0 to 1% | 0 (off) |
| Optics | Central zone | A / B / C | C (lower right) |
| Optics | Render quality | 24 / 48 / 96 taps | 48 |
| Per copy A (upper) | x, y offset | -5 to 5%, 0.05 steps | +0.20%, -0.70% |
| Per copy A | Focal power | -1.2 to 1.2 | -0.6 |
| Per copy A | Brightness | 0 to 2× | 1× |
| Per copy A | Dominance frequency | 0 to 100% | 30% |
| Per copy B (lower left) | x, y offset | | -0.60%, +0.40% |
| Per copy B | Focal power | | 0.0 |
| Per copy B | Brightness | | 1× |
| Per copy B | Dominance frequency | | 10% |
| Per copy C (lower right) | x, y offset | | +0.40%, +0.40% |
| Per copy C | Focal power | | +0.6 |
| Per copy C | Brightness | | 1× |
| Per copy C | Dominance frequency | | 60% |

Default offsets keep the triangle shape of the owner's sketch but at a much smaller scale: the copies sit almost on top of one another, about 1% of image height apart, just enough to double and triple edges. Default dominance frequencies encode "lower right usually, upper sometimes, lower left rarely." Both blur defaults are zero: all three copies are fully sharp and differ only in brightness. The blur path stays in the renderer for tuning and for the future topography-driven mode.

### 5.6 Comparison and output

- **FR-21** A hold-to-compare button and the Space key show the unmodified image while held.
- **FR-22** A live indicator shows the three copies as dots: size is current blur, brightness is current weight, a ring marks the copy focus is heading toward. A status line reports focus value, which copy is sharpest, and whether focus is stalled between copies.
- **FR-23** Reset all parameters to defaults.

---

## 6. Non-functional requirements

- **NFR-1** Single static HTML file. No build step, no dependencies, no network calls. Opens from disk or any static host.
- **NFR-2** WebGL2 required. Show a clear message if unavailable.
- **NFR-3** Steady frame pacing. Dynamics are frame-rate independent (time-step based).
- **NFR-4** Panel and view remain usable down to about 820 px wide, stacking vertically below that.
- **NFR-5** No image data or parameters are transmitted anywhere.

---

## 7. Success criteria

1. The owner, looking at the simulation with their good eye, says it matches what their left eye sees on at least the text demo and the night-lights demo.
2. An observer without the condition can correctly describe the three signature behaviors after 30 seconds of watching: fixed triangle, one copy sharp at a time, occasional everything-blurry state.
3. The owner can reach a satisfactory match with slider tuning only, with no code changes.
4. 60 fps at default quality on the owner's machine with a 1600 px image.

---

## 8. Technical design (as built in the prototype)

- **Platform:** Vanilla JS + WebGL2. Full-screen triangle, one fragment shader.
- **Texture:** Source image uploaded as RGBA8 with mipmaps (needed for large-radius blur sampling).
- **Shader uniforms per frame:** three 2D offsets, three blur radii, three normalized weights, tap count, render mode (simulate / original / blink).
- **Blur:** Vogel disk sampling (`r = sqrt((i+0.5)/N) · R`, `θ = i · 2.39996 + noise`), per-pixel interleaved gradient noise rotation, LOD = log2(2R/√N) in pixels.
- **Color:** gamma 2.2 approximation to and from linear light.
- **Dynamics:** per-copy smoothed Ornstein-Uhlenbeck noise, softmax competition with static bias, slow contrast-collapse noise (Section 5.3), stepped in the animation loop with frame-rate independent time steps.
- **Serving:** `python -m http.server` or any static host. Opening the file directly also works.

Known simplifications to revisit:

- Disk PSF ignores diffraction and the true wavefront shape of each zone. Real zones would give asymmetric, possibly comet-shaped blur.
- Pupil only ramps copy weights and scales blur. It does not change blur shape or which part of each zone is exposed.
- No chromatic difference between copies (owner has not reported color fringing).
- Dominance is modeled as a soft weight competition driven by random drift. The real driver is some mix of accommodation and cortical suppression; the statistics of the drift (timescale, how often the lead is shared) are tuned by eye, not measured.

---

## 9. Future work (post-MVP)

1. **Topography-driven mode.** Import a corneal topography or wavefront (Zernike) export, compute the actual point spread function, and replace the three-disk model. Highest fidelity path.
2. **Per-copy PSF shape.** Elliptical or comet blur per zone, with rotation.
3. **Chromatic offsets** if any copy is observed to carry a color fringe.
4. **Presets.** Save and load named parameter sets as JSON; share a preset via URL hash.
5. **Side-by-side mode** (normal vs simulated) in addition to hold-to-compare.
6. **Webcam / live video input.**
7. **Recording** a short clip (WebM) of the live simulation for sharing.
8. **Gaze-triggered re-focus.** Clicking a point in the image re-targets focus, mimicking looking at something new.

---

## 10. Open questions

| # | Question | Why it matters | Current assumption |
|---|---|---|---|
| 1 | Is a corneal topography scan available? | Would replace guessed geometry with measured optics. | Not available; hand-tuned. |
| 2 | How long does each copy typically hold the lead? | Sets the drift timescale default. Transition is confirmed as a continuous, milky crossfade. | 6 s timescale; owner tunes. |
| 3 | Does the cycle run on its own, or mainly after blinks or gaze shifts? | Determines whether blinks should be the main trigger. | Both; autonomous cycling with blink resets. |
| 4 | Are the three copies equally sharp when each is "winning," or is one zone always softer? | Per-copy baseline blur may differ. | Equal baseline; all copies nearly equally sharp, per owner. |
| 5 | Any color fringing on any copy? | Would add chromatic offset. | None. |
| 6 | Does the effect clearly weaken in bright light? | Validates the pupil model. | Yes, per optics. |
| 7 | "Distortive": overlap confusion only, or do straight lines bend? | Bending would need a warp stage, not just a PSF. | Overlap only. |
| 8 | Exact separation relative to object size at typical viewing distances. | Sets default offsets. | Resolved: copies almost coincident, about 1% of image height apart, triangle shape kept. |

---

## 11. Decision log

- **2026-10-04** Cause confirmed as irregular corneal surface. Rules out lens-based models; a static multi-zone corneal PSF model is appropriate.
- **2026-10-04** Copies confirmed static in position. Geometry is fixed; only focus state and perceptual weighting vary over time.
- **2026-10-04** Triangle arrangement confirmed by sketch (one upper, two lower). Defaults set accordingly.
- **2026-10-04** Dominance order confirmed: lower right most often, upper sometimes, lower left rarely. Default dominance weights 60/30/10; lower right designated the central (always exposed) zone.
- **2026-10-04** Rejected opacity-layer approach in favor of GPU PSF convolution with additive linear-light compositing.
- **2026-10-04** Chose WebGL2 over WebGPU for the MVP for broad compatibility; single-file, no backend.
- **2026-10-04** Owner feedback on first prototype: the switch between dominant copies is usually extremely subtle, and non-dominant copies are dim rather than blurry. Defaults changed to a slow glide (switch speed 0.8), mild blur (0.7), strong dominance brightness boost (80%), blinks off. Model framing updated: focus selects the dominant copy, but the visible difference is brightness, not blur.
- **2026-10-04** Owner feedback on second prototype: the rotation looked like a slideshow; it needs to be milky, blending. Replaced the dwell-and-switch state machine with continuous per-copy dominance drift and softmax competition. There are no switch events anymore; the lead passes by overlapping crossfade. Switch-speed control removed; dwell replaced by drift timescale (default 6 s).
- **2026-10-04** Owner feedback on third prototype: no copy is ever truly fully dominant, yet the defaults sometimes made the alternatives invisible. Added a per-copy minimum presence floor (18%) and lowered default dominance contrast to 60%.
- **2026-10-04** Owner set copy separation scale default to 0.28×. The sketch exaggerated spacing relative to a full photo; the tuned value is about 1.4% of image height between the upper and lower copies.
- **2026-10-04** Owner clarified the copies are almost on top of one another, just slightly offset. Per-copy default offsets reduced to roughly 1% of image height (triangle shape kept), separation scale returned to 1×, offset sliders narrowed to ±5% with 0.05% steps for fine tuning.
- **2026-10-04** Owner set both blur defaults to zero. Dominance is purely a brightness difference between three equally sharp copies; defocus blur remains available as a control but is off by default.
- **2026-10-04** Defaults approved by owner. "Save still" removed as unnecessary. Published as a private Claude artifact for sharing: https://claude.ai/artifact/HwKw4qNMyKBmtfWbEpA8rE
- **2026-10-04** Owner wanted no sign-in for viewers. Deployed to GitHub Pages from the public repo who/triplopia; live at https://who.github.io/triplopia/. Owner reported NaN in the artifact viewer; the animation loop now uses its own clock and sanitizes all state each frame.
