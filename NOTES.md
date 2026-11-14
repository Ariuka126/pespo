# Field Shift — build notes

`dist/index.html` is the whole app: one file, no build step, no backend, no keys.
Open it by double-click, or drop it on Netlify / Vercel / GitHub Pages.
`dist/sw.js` is optional; it only does anything on real hosting (https).

## 1. Stack, and why

Plain HTML + CSS + JS in a single file, with **Leaflet 1.9.4 inlined** (not from a CDN).

- Opens from `file://`, so it works on a judge's laptop with no internet and no install.
- 309 KB total, which matters on a cheap Android phone on a rural connection. React + Vite +
  Tailwind would have added a build step and a bigger bundle for no gain at this size.
- Source lives in `src/` and is assembled by `node build.mjs`. Edit `src/*`, run the build,
  ship `dist/`.
- Leaflet markers are `L.circleMarker`, so no marker images are needed — the brief's warning
  about broken default icons does not apply.

## 2. AI integration: option (A), the copy-paste bridge

As instructed, (A) is the default and the only mechanism built. No API key exists anywhere in
the code. Step 4 composes the full prompt from the real figures and the farmer's answers,
shows it in full, copies it, links out to a free AI tool, and parses the answer back into the
table. The parser accepts JSON (fenced or bare), falls back to a markdown table, and if both
fail it shows the raw text and says so instead of pretending.

`generateAnalysis(payload)` is the single swap point. It returns
`{ mode, prompt, text }`; a later (B) or (C) only has to resolve `text` from a proxy.
All AI output is rendered as text nodes, never as HTML, and is capped per field.

## 3. Which numbers are real, and which are not

**Real and sourced — do not edit without re-sourcing:**

- The three climate reference points (Zuunkharaa, Darkhan, Bulgan), in `src/data.js`. Darkhan and
  Bulgan are embedded exactly as supplied. Zuunkharaa was moved on 2026-11-15 to the valley at
  48.85, 106.46 (a different POWER cell from 48.88, 106.82) and re-fetched; see the comment in
  `src/data.js` for exactly how its values were derived.
- Any other point: fetched live from NASA POWER, 2001–2025, and computed in the browser.
  `-999` fill values are dropped; a summer missing more than 7 days is excluded.
- The soil table: Tserengunsen et al. (2022), MJGG 43, DOI 10.5564/mjgg.v59i43.2513. I checked
  these against the published paper — pH, CaCO₃, organic matter, P₂O₅ and K₂O all match at all
  four depths. Labelled throughout as regional reference values, never as "your field".
- SoilGrids: queried live, with a null check as well as `res.ok`. If values come back empty —
  which is what ISRIC's paused service does today — the app says "not available" and shows
  nothing. No SoilGrids numbers are embedded.
- Town coordinates: GeoNames. National border: Natural Earth 1:10m.

**There are no placeholder numbers in this build.** Nothing is invented anywhere. Where data
is missing the app shows an honest empty state.

**Not a number, but needs your eye:** the seasonal "what to do now" banner is a general
calendar I wrote, not derived from data. It is tagged *Estimated*. Have an agronomist check it.

## 4. Status of open items (2026-11-15)

1. **Zuunkharaa coordinates — done.** Moved to 48.85, 106.46. Re-fetched from POWER
   (2001–2025) and run through `computeClimate()`: summer rain 211.2 mm (dry year 156.6),
   July 19.5 °C, spring wind 2.94 m/s, cell elevation 1038 m. The same run reproduces the old
   point's and Darkhan's and Bulgan's rain figures exactly, but its frost-free count is one day
   higher than the team script on all three, so Zuunkharaa is stored as 114 − 1 = 113.
   `daysAbove0` is null for Zuunkharaa: the team script's definition gives ~55 days fewer than
   `computeClimate()` on every point and could not be reproduced. Worth settling before judging.
2. **P₂O₅ and K₂O units — still open.**
3. **Mongolian wording — done.** хяруу / хяруугүй хугацаа for frost and frost-free season,
   хаврын улаан буудай for spring wheat, applied everywhere including the AI prompt.
4. **Scoring formulas — still not rebuilt** (no coefficients supplied). The farmer's ranked
   priority now steers the AI's rotation ranking instead.
5. **Live-API check — done** from a browser on https: POWER, SoilGrids (now returning real
   values) and Nominatim all answered for Khutag-Undur, a non-preset point.

## 5. Added 2026-11-15

- Frost-pocket note under the frost-free figures; soil "not sampled here" text is a warning box.
- Step 3 what-if toggles: July +1.5 °C, June–August rain −20 %. Pure arithmetic on the shown
  record, labelled as a what-if, carried into the AI prompt and the printed report.
- Prompt requires ≥ 3 rotations: the current one as baseline (`is_current`) plus ≥ 2
  alternatives, each with `why_fits`, `benefits`, `gaps_risks`, ranked by priority 1.
- Results render as comparison cards: a swipe row on phones, side by side from 760 px.
  The parser also reads markdown tables in either orientation and the old JSON keys.
- Service worker cache bumped to `fieldshift-2026-11-15-r2`.

## What was tested

- Climate maths against synthetic daily data with known answers: summer-rain mean and 25th
  percentile, frost-free season, the day count above 0 °C, July mean, spring wind, and `-999`
  handling. All exact.
- Border test: Ulaanbaatar, Darkhan, Altanbulag, Ölgii and Zamyn-Üüd inside; Irkutsk, Erenhot
  and Hohhot rejected; Kyakhta flagged as border-adjacent.
- Full click-through of all five steps at 360 px, in Mongolian and English, light and dark.
- WCAG AA contrast on every text node in both themes, across all five steps and both
  secondary views: no failures.
- Every touch target measured: none under 44 × 44 px.
- No horizontal page scroll at 360 px.
- The printed report fits on exactly one A4 page.
- Mocked-network tests of the live POWER path, the SoilGrids all-null trap, and a normal
  SoilGrids response (pH 72 → 7.2; nitrogen ÷100; bulk density ÷100).

## Two bugs that testing caught

- Every "I don't know / skip" option was being positioned 999 px off-screen, because the
  option's `.skip` class collided with the skip-to-content link's rule. The farmer could not
  skip a question at all. The link is now `.skiplink`.
- `replaceChildren(null)` was printing the literal word "null" in the location bar.

## Space Apps submission facts

NASA POWER, Agroclimatology community, parameters `PRECTOTCORR`, `T2M_MIN`, `T2M`, `WS2M`,
2001–2025, endpoint `power.larc.nasa.gov/api/temporal/daily/point`.
AI tool use: this app asks a general-purpose LLM to write the advice table, and the code was
written with Claude. Disclose both in the SI field.
Remember the limits: 30 s video **or** 7 slides, every link public with no login, nobody under
18 on camera.
