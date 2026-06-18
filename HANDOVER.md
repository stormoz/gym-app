# MAG Levels Hub — Handover Document

**Deliverable:** `mag-levels-hub.html` — a single, self-contained HTML file (≈160 KB). No build step, no dependencies to install. Open it in any modern browser, or host it anywhere static.

**What it is:** A child‑friendly but professional web app for Men's Artistic Gymnastics (MAG) in Australia / NSW, oriented around the Australian Levels Program (ALP / GymNSW). It helps gymnasts, parents and new judges understand levels, routines, scoring, and lets a gymnast build their own routine per apparatus.

---

## 1. How it's built

- **One file.** All HTML, CSS and JavaScript live in `mag-levels-hub.html`. The logo is an embedded base64 PNG in the header.
- **Vanilla everything.** No framework, no external JS. Only external resources are Google Fonts (Fredoka for display, Nunito for body).
- **Palette / theme** via CSS variables: `--blue #1f5fd6`, `--blue-dark #143b8f`, `--red #e02434`, navy logo `#062657`, plus tints/lines. White/blue/red overall.
- **Persistence:** `localStorage` under key **`mag_levels_hub_v1`**, wrapped in try/catch so it silently falls back to in‑memory if storage is blocked (e.g. in a sandboxed iframe).

### Editing / validation workflow used throughout
The whole app is one file, so edits were made with small Python scripts (string/regex replacement) and validated every time with:
1. Extract the `<script>` block to `check.js`, run `node --check check.js` (syntax).
2. Check tag balance and `{}`/`()`/backtick balance.
3. For the skill data, a headless Node "harness" stubs `document`/`box` and calls `renderDDOptions('')` to confirm the dropdown renders (bands, options, no runtime errors).
4. Copy to `/mnt/user-data/outputs/` and present.

Keep this loop — it caught real bugs (e.g. an early skill insertion that targeted the wrong array).

---

## 2. The five sections

The page is numbered 1–5:

1. **Choose a level** — pick Levels 1–10, banded Foundation (1–2), Junior Compulsory (3–6), Senior Optional (7–10). Drives the level overview: band, scoring type, the official **LAT pass mark** (L3–6 = 30.00 AA, L7 = 38, L8 = 39, L9 = 40, L10 = 41), plus clearly‑labelled **illustrative** "state competitive / podium" targets (NOT real GymNSW data).
2. **Routines & scoring by apparatus** — per‑apparatus (6 apparatus) representative elements, common deductions, and a plain‑language scoring box for the selected level.
3. **Track your scores & see your gap** — six score inputs → live all‑around total, a gauge with pass/competitive/podium markers, per‑apparatus scales with +/− gaps, a "focus here" weakest‑apparatus card and a "next milestone" card. "Clear scores" button.
4. **Build your own routine** — the routine builder (the most heavily iterated feature). See §3.
5. **How MAG scoring works** — accordion explainers. See §4.

---

## 3. Section 4 — the Routine Builder (most important to understand)

### UI
- Six **apparatus tabs**, each showing a live **skill count** ("Floor · N skills"). Each apparatus keeps its **own** routine (`state.routines[app]`).
- A **custom dropdown** (not a native `<select>`): a styled trigger opens a floating panel with a **search box** and the skill list. Clicking a skill adds it. Closes on outside‑click / Esc.
- The dropdown is **grouped by level band**, and **within each band sub‑grouped by skill type** (e.g. "Backward acro", "Circles & flairs", "Dismount"). Empty bands/types are skipped. Type **"Buck"** sorts first, **"Dismount"** last; for Vault the type sub‑header is suppressed (only one type).
- Each routine item: drag handle (⠿) + order number, name, colour‑coded difficulty badge, type tag, execution focus note, remove (×), and **up/down reorder** buttons. Reorder via **buttons or HTML5 drag‑and‑drop**.
- Live **estimated D‑score** summary: top‑8 element difficulty + 0.5 per distinct element‑group type (cap 4); Vault shows the best vault's value. Clearly labelled an estimate.

### Level bands (the grouping)
Codes → labels: `l13`→"Levels 1–3", `l4`→"Level 4", `l56`→"Levels 5–6", `l78`→"Levels 7–8", `l910`→"Levels 9–10", `lelite`→"Olympian". Defined in the `bands` array inside `renderDDOptions`.

### SKILLS data schema
`const SKILLS = { FX:[…], PH:[…], SR:[…], VT:[…], PB:[…], HB:[…] };`
Each entry:
```
{ id, n, d, v, gn, lv, ex }
```
- `id` — unique string, prefixed by apparatus (`fx_`, `ph_`, `sr_`, `vt_`, `pb_`, `hb_`). **Never reuse/rename ids** — saved routines reference them; `loadState` filters out unknown ids.
- `n` — full display name. For **Floor acro**, names are written as the **whole pathway** (e.g. "Round-off, back handspring, back layout"; "Front handspring, front tuck"). The connecting skill is called **"back handspring"** (never "flic‑flac").
- `d` — difficulty **letter** on the FIG men's scale **A–J** (`A`=0.1 … `H`=0.8, `I`=0.9, `J`=1.0). Foundation/training skills use `Fdn`. Vault uses `''` (empty).
- `v` — numeric value. Foundation = `0`. Vault uses the real **vault value** (e.g. 2.0–6.2). Others = the A–J value.
- `gn` — the **type** (element group), e.g. `Forward acro`, `Backward acro`, `Strength / Hold`, `Circles & flairs`, `Travels`, `Single-leg & scissors`, `Kip / uprise to support`, `Swing to handstand`, `Strength holds`, `Upper-arm elements`, `Support & long swing`, `Under-bar elements`, `Long swings`, `In-bar elements`, `Flight / release`, `Dismount`, `Buck`. Vault = `''` (renders as "Vault" / no sub‑header).
- `lv` — the level band code (above).
- `ex` — short execution focus note ("what to clean / watch for").

**Important:** no single quotes inside any string field (names/ex use UTF‑8 chars like `½`, `–`, `ö`, `&` freely, but no `'`).

### Difficulty → colour
`dvColor(v)`: `v<=0` green (Foundation); then light‑blue → blue → dark‑blue (A–C) → amber (D) → orange (E) → red (F+). `vtColor(v)` colours vault by value. Badge shows the letter (`s.d`) for non‑vault, the value for vault. (`Fdn` shows in green; `I`/`J` show in red.)

### Key functions (all inside the one `<script>`)
- `buildRBTabs()` — renders the six tabs with counts; wires tab clicks (`state.rbApp = …; saveState(); renderRB()`).
- `renderRB()` — sets active tab, trigger label, closes dropdown, calls `renderRoutine()`.
- `renderDDOptions(filter)` — builds the band→type grouped option list; search filter on name/type; option click pushes id → `saveState()` → `renderRoutine()`.
- `ddToggle(open)` / search / outside‑click / Esc handlers — custom dropdown open/close.
- `renderRoutine()` — renders the current apparatus's routine items (badge, type tag, exec note, reorder, remove), drag‑and‑drop handlers, and the D‑score summary.
- `moveItem(k,from,to)` — up/down reorder.
- `skillById(k,id)` — lookup.

### Current skill totals (281 total)
| Apparatus | Total | 1–3 | 4 | 5–6 | 7–8 | 9–10 | Olympian |
|---|---|---|---|---|---|---|---|
| Floor | 61 | 8 | 6 | 6 | 14 | 15 | 12 |
| Pommel Horse | 51 | 8 | 9 | 7 | 13 | 10 | 4 |
| Rings | 45 | 6 | 6 | 2 | 11 | 12 | 8 |
| Vault | 35 | 2 | 2 | 8 | 7 | 7 | 9 |
| Parallel Bars | 40 | 6 | 6 | 3 | 11 | 11 | 3 |
| High Bar | 49 | 6 | 7 | 7 | 9 | 12 | 8 |

The **Olympian** band (`lelite`) auto‑holds the genuinely elite skills (rule used when assigning: non‑vault `v >= 0.6`, i.e. F/G/H/I/J; vault `v >= 5.2`).

### How to add a skill (recipe)
1. Append an entry to the right apparatus array in `SKILLS`, anchored to the `SKILLS` object (search `'const SKILLS'` first — **`FX:[` also appears in the level‑elements `ELEM` data, which comes earlier**; that bug bit us once).
2. Give it a unique `id`, a real `gn` type that matches an existing type for that apparatus, a `lv` band, an `A–J` letter + matching `v`, and a short `ex`.
3. Validate (node --check, render harness), copy to outputs.

---

## 4. Section 5 — Scoring explainer (ALP rules captured)

Two corrections the domain expert (Kon) supplied are baked in:

- **Compulsory levels 1–6** start from a **10.0** start value, **plus a bonus**: **+0.5 at Level 4** and **+1.0 at Levels 5–6** (raising the start value to 10.5 / 11.0) — on **every apparatus except Vault**, which always starts from 10.0.
- **Optional (own) routines are age‑dependent:** an **over‑age** gymnast may build their own routine from **Level 6 and above**; an **under‑age** gymnast only from **Level 9 and above** (set routines below that).

Other accordions cover the six apparatus / all‑around and the LAT pass mark.

> Known inconsistency (flagged, not yet changed because the user scoped fixes to section 5): the **level picker** still bands 7–10 as "Senior · Optional", the **Level 7 blurb** says gymnasts build their own routines, and the **Section‑2 scoring box** shows "optional · D+E" for any level > 6 and doesn't yet reflect the L4 / L5–6 bonus. Aligning these is an open task.

---

## 5. Ratings accuracy (important context)

Difficulty values were reviewed against the **2025–2028 FIG Men's Code of Points** (verified by web search). The men's code now extends to **I (0.9)** and **J (1.0)**.

**Verified / corrected (Floor most thoroughly):**
- Piked triple back (**Nagornyy**) = **J**; tucked triple back (**Liukin**) = **I**.
- Jarman (double layout 3½ tw) = **J**; Shirai III (double layout 3/1) = **I**; double layout 5/2 = **H**; double layout 2/1 = **G**; Penev / double layout 1/1 = **F**; double layout = **E**.
- Double pike backward = **D**; double‑double (double back 2/1) = **E**; full‑in (double back 1/1) = **D**.
- Single straight salto with 1/1 twist = **B** (so back/front full = B, front double full = C, Randi 2½ = D).
- Hypolito (Arabian double layout 1/1) = **G**; Zapata II (double front straight 3/2) = **G**; double front tuck 1/1 = **F**; Tamayo = **E**; Korosteljev = **E**.
- Rings: triple tuck dismount = **G**; double layout dismount (Alvarez) = **E**. P‑Bars: Morisue = E, Healy = D (already matched).

**Still best‑estimate (NOT individually verified):** most Pommel, High Bar and Vault named values, and any newer eponymous names that were modelled on real elements but not confirmed against competition records. These are internally consistent on the A–J scale but should be confirmed against the official ALP/FIG code before being treated as authoritative.

---

## 6. Logo

The header logo is the user's "MAG GYMNASTICS" emblem (navy gymnast on rings, red straps), processed earlier (white background knocked out, legs rebuilt/together, slightly more muscular). Embedded as a base64 PNG inside `<img class="logo-img">` in the `.mark` header area (height ~52px). A standalone corrected asset also exists: `mag_logo_fixed.png`.

---

## 7. Persistence details

`saveState()` writes `{level, app, scores, rbApp, routines}` to `localStorage['mag_levels_hub_v1']`; called on every mutation (level/apparatus select, score input, clear, add/remove/reorder/clear routine, tab switch). `loadState()` restores on init and **validates** everything (drops invalid level/apparatus/skill ids), so an old/corrupt save can't break the app. Persists per browser/device (not synced).

---

## 8. Open tasks / standing offers

1. **Align Sections 1 & 2 with the Section‑5 rules** (mark 7–8 as set routines, start optional at L9 for under‑age / L6 over‑age, show the non‑vault L4/L5–6 bonus in the tracker scale and level data).
2. **Verify remaining ratings** against the Code of Points for Pommel, High Bar, Vault (and confirm every eponymous name is a real competed element). The user asked for "professional, competition‑performed" skills — the elite tier is modelled on real elements but a per‑apparatus CoP verification pass is still outstanding.
3. **Decide on the developmental tier:** Levels 1–6 include intentional training skills (rolls, buck circles, mat drills, glide kips) that are *not* elite competition elements. Open question from the last exchange: keep them (current state) or strip to competition‑only.
4. Replace illustrative competitive/podium targets with **real GymNSW results** if available.
5. Optional niceties: export/import or full "reset app" for saved data; keyboard navigation in the dropdown; tag each skill with its exact ALP level.

---

## 9. File map

- `mag-levels-hub.html` — the app (the deliverable; also at `/mnt/user-data/outputs/`).
- `mag_logo_fixed.png` — standalone corrected full logo asset.
- `HANDOVER.md` — this document.

**Quick start for a new maintainer:** open `mag-levels-hub.html` in a browser to see the app; to change skills, edit the `SKILLS` object in the inline `<script>` following the schema in §3; validate with `node --check` on the extracted script before shipping.
