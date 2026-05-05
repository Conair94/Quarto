# Design Document — *The Mathematics of the Striking Moonrise* (rev. 2)

**Status:** draft, pre-implementation
**Author:** Connor Lockhart (with Claude)
**Scope:** Rewrite the post `posts/moonrise-phases/index.qmd` with mathematically rigorous content from first principles, fix two existing OJS visualizers, and add two new visualizers (stereographic celestial-hemisphere animation; 3D globe with terminator). Preserve the existing section layout and tone.
**Non-goals:** Sub-arcminute astronomical accuracy (we are not VSOP87/ELP-2000); modeling refraction, parallax, or Δ(UT1−TT) at survey precision; supporting arbitrary historical epochs outside ±50 yr of J2000.

---

## 1. Audience & accuracy target

- **Audience:** A reader with calculus and trigonometry but no prior celestial mechanics. The post should *introduce the field*, not just present results.
- **Accuracy target:** Rise/set times within **±2 min** vs. NOAA / USNO almanac for Washington, D.C. between 2024–2030. Lunar phase angle within **±0.5°**. This is good enough to be defensible while staying derivable in a blog post.
- **Reproducibility:** Every numeric table cell must be produced by a code block in the post. No "magic numbers."

The current post fails this: §4's table is decoupled from any code, §6's diagram conflates synodic phase with sidereal motion, and §8's solver uses an unnormalized residual.

---

## 2. The first-principles narrative

The rewrite organizes the math as a sequence of coordinate transforms, each motivated before it is written. Section numbering preserves the existing layout (§1–§8) but the *content* under each section is new.

### §1. The astronomical framework — three frames

We motivate three reference frames in order:

1. **Ecliptic (heliocentric → geocentric)** — defined by Earth's orbital plane. The Sun's apparent geocentric longitude $\lambda_\odot$ runs 0–360° per tropical year; latitude $\beta_\odot \equiv 0$ by definition.
2. **Equatorial** — defined by Earth's rotation axis, tilted by the obliquity $\varepsilon = 23.4393° - 0.0130°\,T$ (centuries from J2000). Coordinates: right ascension $\alpha$, declination $\delta$.
3. **Horizontal (topocentric)** — defined by the observer's zenith. Coordinates: azimuth $A$, altitude $a$. Depends on latitude $\varphi$ and local sidereal time $\theta$.

The chain of transforms is the spine of the post:

$$ (\lambda,\beta) \xrightarrow{R_x(-\varepsilon)} (\alpha,\delta) \xrightarrow{R_y(\theta - \alpha)\,R_x(\varphi-90°)} (A, a) $$

We present each as a rotation of a unit vector, *not* as a tangle of $\sin/\cos$ identities — this is the pedagogical improvement over the current post.

### §2. The Sun's apparent motion — Kepler's equation

From first principles: Earth orbits the Sun on an ellipse with eccentricity $e \approx 0.0167$. The Sun's *mean anomaly* $M$ advances linearly in time; its *true anomaly* $\nu$ does not. Kepler's equation

$$ M = E - e \sin E, \qquad \tan\tfrac{\nu}{2} = \sqrt{\tfrac{1+e}{1-e}}\,\tan\tfrac{E}{2} $$

gives the equation of center to leading order: $\nu - M \approx 2e \sin M + \tfrac{5}{4}e^2 \sin 2M$. Combined with the obliquity, this is the *equation of time* — already a derivation worth showing.

We use the Meeus-style low-precision series (Astronomical Algorithms ch. 25):

$$ \lambda_\odot = L_0 + C(M), \qquad L_0 = 280.4665° + 36000.7698°\,T $$

with $C$ the equation of center above. This is good to ~0.01° over 1900–2100.

### §3. The Moon — five fundamental arguments

Where the current post collapses lunar motion to "13.18°/day," we introduce the five Delaunay-style fundamental arguments that *every* lunar theory uses:

- $L'$ — Moon's mean longitude
- $D$ — mean elongation from the Sun
- $M$ — Sun's mean anomaly
- $M'$ — Moon's mean anomaly
- $F$ — Moon's argument of latitude (angular distance from ascending node)

Linear-in-$T$ expressions from Meeus ch. 47. We then take **only the four largest periodic terms** in longitude (evection, variation, yearly, reduction) and the largest in latitude. This loses ~0.1° vs. full ELP but keeps the math legible:

$$ \lambda_M \approx L' + 6.289°\sin M' - 1.274°\sin(2D - M') + 0.658°\sin 2D + \cdots $$
$$ \beta_M \approx 5.128°\sin F + \cdots $$

The synodic phase is then $D = \lambda_M - \lambda_\odot$, and **illumination** is $k = \tfrac{1}{2}(1 - \cos D')$ where $D'$ is the *phase angle* corrected for the Sun-Earth-Moon parallax (negligible here).

This finally lets us answer "what is the moon's phase when it rises 1h after sunset" with a real number, not a curve fit.

### §4. From declination to rise time — the hour-angle equation

Standard derivation from the spherical triangle (zenith, pole, body):

$$ \cos H_0 = \frac{\sin h_0 - \sin\varphi \sin\delta}{\cos\varphi \cos\delta} $$

where $h_0$ is the geometric altitude of the body's center at rise. We **explicitly** include:

- Standard refraction at horizon: $-34'$
- Solar semidiameter: $-16'$ → $h_0 = -50'$ for sun
- Lunar semidiameter (variable, ~$15'-17'$) and parallax ($\approx +57'$, *raises* apparent rise) → $h_0 \approx +7'$ for moon

The current post ignores these entirely. They shift moonrise by **~5–10 minutes** — comparable in magnitude to the orbital-inclination effect being studied, so omitting them invalidates the seasonal table.

### §5. Sidereal time and the rise-set residual

We define Greenwich Mean Sidereal Time:

$$ \theta_G = 18.6973746^h + 24.06570982441908^h \cdot d + \cdots $$

(Meeus 12.4, $d$ = days from J2000.0 UT). Local sidereal time $\theta = \theta_G + L$ for east longitude $L$.

Rise time of a body with RA $\alpha$ and rise hour angle $-H_0$ satisfies $\theta = \alpha - H_0$, solved for UT. Because $\alpha$ and $\delta$ themselves change during the day (especially for the Moon, ~0.5°/hour in RA), we **iterate**: predict rise UT, recompute $\alpha,\delta$ at that UT, repeat. Two iterations suffice to <30 s.

The current post does no iteration — another source of its error.

### §6. The seasonal table, reproduced

Rebuild the §4 table programmatically. Columns:

| Date (UT) of full moon | Sunset (local) | Moonrise (local) | Δt (min) | Required age for 1h-after-sunset rise (days) | Illumination at that age |

Recompute for **2026–2027** at $\varphi=38.9°N$, $L=-77.04°$. Replace the current table verbatim. Expect Δt to range roughly $-30$ to $+25$ min — the current post's values are in the right neighborhood but not exactly reproducible.

### §7. The harvest moon, properly

The current §5 hand-waves the harvest-moon effect. Replace with a derivation: the daily lag in moonrise is

$$ \Delta t_\text{daily} \approx \frac{1}{1 - \dot\alpha_M / \dot\theta} \cdot \frac{2\pi}{\dot\theta} $$

Near the autumnal equinox, the moon's path on the ecliptic crosses the equator at a *shallow* angle (because the ecliptic itself is most "horizontal" relative to the eastern horizon at sunset for northern observers in autumn evenings). This compresses the daily increase in $\alpha_M$ projected onto the horizon, shrinking the lag to ~25 min instead of the ~50 min average. We show the geometry.

### §8. The interactive solver

Reuse §3's full lunar series. Solve for the lunar mean longitude $L'$ such that moonrise occurs exactly 60 min after sunset on a given calendar date, using Newton's method on a residual that is **explicitly normalized to $(-\pi, \pi]$** (the current code's bug). Report phase angle, illumination, age past full.

---

## 3. Visualizer redesign

Four diagrams total. Two are repaired; two are new. All are OJS, consistent with the existing post; all share a single computation module so the math is written once.

### 3.1 Shared computation module

A single `{ojs}` block at the top of §1 exports:

```
sunPosition(jd)          → {lambda, alpha, delta, ra_h}
moonPosition(jd)         → {lambda, beta, alpha, delta, distance_km}
riseSet(body, jd, lat, lon, h0)  → {rise_jd, transit_jd, set_jd}
horizonProject(alpha, delta, lat, lst) → {alt, az}
illumination(jd)         → k ∈ [0,1]
```

`jd` is Julian Date (UT). All four visualizers consume this module. This eliminates the current post's three duplicated copies of `getMoonEq`/`getSunEq`.

### 3.2 Visualizer A — Geometric overhead diagram (repair existing §6 diagram)

**Preserve:** SVG layout, color palette, Earth/Sun/Moon glyphs, slider sidebar (day-of-year, latitude, ascending-node).

**Fix:**
- Replace fake `lam_m = lam_s + π + (day % 29.53)*2π/29.53` with `moonPosition(jd).lambda`. Day-of-year slider maps to JD via a calendar-year selector (default 2026).
- Add a **synodic-phase slider** (separate from day-of-year) so the user can decouple "where the moon is in its orbit" from "what date it is" — useful for the harvest-moon teaching point.
- Show the moon's ecliptic latitude as a vertical offset from the orbital ring (currently zero — the whole point of the post is lost).
- Render Earth's terminator as a great-circle in the orthographic projection rather than a half-disk arc — currently the shadow is anchored to the Sun's *image* angle, not its real direction.

### 3.3 Visualizer B — 24-hour horizon timeline (repair existing §6 timeline)

**Preserve:** Horizontal time axis, sun-up yellow band, moon-up gray band, label.

**Fix:**
- Use `riseSet(...)` for both bodies; show actual sunrise/sunset/moonrise/moonset for the selected date.
- Show *two* moon bands when the moon is up across local midnight (current code does this but with the wrong source data).
- Add tick marks at the 1-hour-after-sunset target so the alignment with moonrise is visually obvious.
- Add a small inline readout: "Moon rises X min after sunset; illumination Y%."

### 3.4 Visualizer C — Stereographic celestial hemisphere (new)

A "fish-eye" plot of the sky as seen from the observer, animated through one year.

**Projection:** Stereographic from the nadir, so the zenith is at center, the horizon is the bounding circle, altitude $a$ maps to radius $r = R \tan(\tfrac{90°-a}{2})$, azimuth maps to angle. North up, east right (mirror-image of the sky as printed in star charts — but matching what you'd see looking up).

**What's drawn:**
- Cardinal points (N/E/S/W) on the horizon ring.
- The celestial equator and the ecliptic as great-circle arcs (computed once per frame from $\theta$).
- The Sun and Moon as colored disks; the Moon shaded by phase angle relative to the Sun's direction.
- A faint trail showing the body's path over the past 24 hours.

**Controls:**
- Date slider (day-of-year).
- Time-of-day slider (0–24 h local solar time).
- "Animate" toggle: at 30 fps, advance the time slider 4 min/frame, completing 24 h in 12 s; alternatively, "year mode" advances at 1 day/frame at noon, completing the year in 12 s.
- Latitude slider (the existing one).

**Implementation cost:** ~150 lines of OJS, two `requestAnimationFrame` loops. No external libraries beyond d3 (already loaded).

### 3.5 Visualizer D — 3D globe with terminator (new)

A rotating 3D Earth showing the day/night line and the sub-solar / sub-lunar points.

**Rendering choice:** **Three.js** is overkill and adds a 600 KB script. We use **d3-geo** with an orthographic projection — this is what e.g. the NYT election globe uses, ~10 KB additional. A graticule, coastlines (from `world-atlas/land-110m.json`, ~30 KB), and a great-circle terminator are all native to d3-geo.

**What's drawn:**
- Earth in orthographic projection, rotatable by drag.
- Day/night terminator: the great circle 90° from the sub-solar point. Night side shaded.
- Sub-solar point (☉ marker) and sub-lunar point (☾ marker), both moving in real time as the date/time advances.
- The observer's location (Washington, D.C. by default, or the latitude slider's value at $L=-77°$) marked with a pin. A small inset shows whether the Sun and Moon are above the observer's local horizon at that instant.

**Controls:** Same date/time/animate controls as Visualizer C, shared.

**Why both C and D?** They answer different questions. C shows the *observer's experience* (where in my sky is the moon right now?). D shows the *global geometry* (which half of Earth sees the moon? where is sub-solar?). The pedagogical pairing is the point.

---

## 4. File layout & build

Single source file: `posts/moonrise-phases/index.qmd`. No new files except this design doc and:

- `posts/moonrise-phases/_compute.ojs` — shared computation module, included via `{ojs} import` in §1.
- `posts/moonrise-phases/_data/land-110m.json` — coastlines for Visualizer D (vendored, not fetched at runtime).

Build is unchanged: `quarto render`. Output goes to `docs/posts/moonrise-phases/index.html`. The user's existing freeze cache (`execute: freeze: auto` in `_quarto.yml`) means we'll need to clear `_freeze/` for this post.

---

## 5. Validation plan

Before the rewrite is declared done:

1. **Numerical regression test** (Python, throwaway): For 12 dates spanning 2026, compute sunrise/sunset/moonrise/moonset with our formulas and compare to USNO almanac values. All four within ±2 min, all phase angles within ±0.5°. If not, isolate which term is missing.
2. **Visual sanity check** for Visualizer C: At local solar noon on the equinox at $\varphi=38.9°$, the Sun should be at altitude $90° - 38.9° = 51.1°$ due south. The plot should show this.
3. **Visual sanity check** for Visualizer D: At UT noon on June 21, the sub-solar point should be at $(23.44°N, 0°E\pm$ equation-of-time$)$. The pin and terminator should align.
4. **Cross-check vs. existing table**: regenerate the §4 table programmatically and diff against the current post. Differences > 5 min flag a problem in *either* the new code or the current post — investigate before publishing.

---

## 6. Risk register

| Risk | Likelihood | Mitigation |
|---|---|---|
| Lunar series truncated too aggressively → >5 min error in moonrise | medium | Fall back to ~15 longitude terms (still legible in a code block) if step 1 of validation fails |
| OJS reactivity loops with shared time slider across 4 viz | medium | Single `viewof` for time, all viz read it; no viz writes back |
| 3D globe perf on mobile | low | d3-geo orthographic at 60° latitude graticule is fine on phones; we don't render coastlines at full resolution |
| Quarto freeze cache serves stale HTML | high | Document `rm -rf _freeze/posts/moonrise-phases` step in the implementation TODO |
| New diagrams overwhelm the post visually | medium | Keep §6 layout (existing two diagrams stay where they are); put C and D in a new §7.5 "Two views of the same sky" before the conclusion |

---

## 7. Implementation phases (proposed)

1. **Phase 1 — math module + validation harness.** Write `_compute.ojs` and a Python mirror; run validation §5.1. No post changes yet. *Stop here for review.*
2. **Phase 2 — repair §1–§5 prose** for first-principles rigor; regenerate §4 table from code.
3. **Phase 3 — repair Visualizers A and B** (§6) using the new module. Preserve layout per user request.
4. **Phase 4 — build Visualizers C and D** in a new §7 ("Two views of the same sky"). Renumber existing §7 conclusion to §8 and existing §8 solver to §9.
5. **Phase 5 — final pass:** clear `_freeze/`, `quarto render`, manual QA in browser, commit.

Each phase is independently reviewable and reversible.

---

## 8. Open questions for the user

Before starting Phase 1:

1. **Location:** lock to Washington, D.C., or make latitude/longitude both user-configurable across the post? (Currently the post says D.C. but the slider varies latitude only.)
2. **Year:** 2026–2027 as in the current table, or auto-roll to "current calendar year"?
3. **Accuracy floor:** is ±2 min adequate, or do you want USNO-grade (±10 s) accuracy? The latter pushes us to ELP-2000/82 truncated and roughly triples the lunar code length.
4. **Visualizer D scope:** is a rotatable orthographic globe enough, or do you want a true WebGL sphere with texture? The latter is ~10x the implementation effort and adds a heavy dependency.
5. **Tone:** the current post is concise and conversational. The first-principles rewrite will be ~2–3x longer. Is that acceptable, or do you want the derivations in collapsible callouts to keep the surface short?
