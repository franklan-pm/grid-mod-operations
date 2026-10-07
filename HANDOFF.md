# Developer Handoff: Grid Operations Learning Tool

**File:** `grid-operations-diagram-v2.html` (single file, ~105 KB)
**Companion:** `README.md` (short overview and change log)
**Status:** Working v2 with three tabs (Explore, Scenarios, Quiz) and a light/dark toggle. No known functional bugs.

---

## 1. What this project is

An interactive, single-page learning tool that explains how a modern electric distribution utility's operational and IT systems fit together. It has three tabs:

| Tab | Purpose |
|---|---|
| **Explore the model** | A layered diagram from wholesale markets down to the physical power path. Animated, color-coded data flows. Clicking a box shows what it does, what it receives and sends, how fast it updates, and dims everything it is not connected to. |
| **Real-life scenarios** | Five step-by-step stories (heat wave peak, storm outage, sunny spring midday, planned maintenance, grid emergency). Each step lights up only the boxes and flows involved, with a timestamp and narrative. |
| **Concepts quiz** | 50 multiple-choice questions in 8 topics, with explanations. After answering, "Show this in the model" reveals the diagram with the relevant parts lit. |

## 2. Who it is for

**End users:** people who work in or around electric utilities and know the basic equipment (reclosers, regulators, meters, substations) but want to understand how the systems connect: what OMS, FLISR, SCADA, DERMS and MDMS each do, what data moves between them, how fast, and why. It also suits onboarding for engineers, analysts, IT staff and grid modernization project teams.

The project owner is a utility employee learning grid modernization. Writing assumes a reader who is technically literate but new to the system-integration view, so jargon is explained at first use and examples use realistic, concrete situations.

**This document's audience:** a developer picking up maintenance or extensions, who may not have a utility background. Section 5 covers the domain logic you need in order to edit content safely.

---

## 3. Project history and decisions

### Timeline

1. **v1 (input).** The owner had another LLM generate an interactive diagram from their study notes. Four layers: Planning (DR, Analytics, VPP), ADMS (OMS, FLISR, SCADA, VVO, network model), Head-end (AMI head-end, MDMS, Grid Comms/FEP, DERMS), Field (meters, RTUs/IEDs, switches/reclosers, "Gridscope" sensors, DERs).
2. **Expert review.** v1 was reviewed for correctness, clarity for newcomers, and data flow accuracy. Findings are in Section 5.3.
3. **v2 rebuild.** Corrected flows, re-layered systems, added missing systems, the OT security boundary, and a physical grid strip.
4. **Scenarios tab.** Added because the full model was overwhelming. Scenarios show one story at a time.
5. **Quiz tab.** 50 questions aimed at "how it fits together", not equipment basics.
6. **Light/dark toggle.** v2 initially followed the OS color scheme, so it opened in light mode on the owner's Windows machine. The owner expected the original dark look, so the default became dark with a remembered toggle.
7. **Cleanup.** The on-page "What changed" section moved into `README.md`.

### Key decisions

| Decision | Rationale |
|---|---|
| Keep everything in **one self-contained HTML file** | The owner opens it locally from OneDrive and shares it. No server, build step or install. |
| **Data-driven rendering** (NODES, FLOWS, SCEN, QUIZ arrays) | Content (the domain knowledge) can be edited without touching layout code. |
| **Keep v1's visual identity** (navy "control room" look, monospace text) | The owner liked it; the revision was about correctness, not restyling. |
| **Dark mode by default**, with a toggle saved in localStorage | Owner preference (see timeline step 6). |
| **Scenarios as a separate tab**, sharing the same SVG | Reduces overload. Reusing one diagram keeps the mental map consistent across tabs. |
| Quiz **hides the diagram until answered** | Showing highlights before answering would give the answer away. |
| Correct answer stored **first** in data, **shuffled** for display | Easy to author; deterministic per-question shuffle prevents an "always B" pattern. |
| Some relationships use **`rel` (logical link) instead of a drawn line** | Avoids line crossings and clutter where the link matters but drawing it would hurt readability (e.g., EMS ↔ SCADA, MDMS → CIS). |
| Generic "Line Sensors + FCIs" box, with **Gridscope as an example** | "Gridscope" came from the owner's notes and appears to be a specific product. Generic labels teach better; the example keeps the owner's context. |
| North American context | 60 Hz, ANSI C84.1, FERC, NERC, IEEE 1366. The owner works at a US utility. |

---

## 4. Technical architecture

### 4.1 File structure (top to bottom)

1. **Theme bootstrap script** in `<head>`. Reads `localStorage['grid-theme']` and sets `data-theme` on `<html>` *before* CSS paints, to avoid a flash of the wrong theme. Default `dark`.
2. **`<style>`.** Color tokens as CSS custom properties: light set on `:root`, dark set on `:root[data-theme="dark"]` (plus a `prefers-color-scheme` fallback for when no `data-theme` is set). All component styles, including quiz and scenario UI.
3. **Markup.** Header (title, theme toggle, legend), tab bar, per-tab intro text, and `#stage`, which holds `#quizbox`, `#stepbox`, `#pills`, the SVG container `#scroller > svg#dia`, and the `#detail` panel.
4. **`<script>`**, in this order:
   - Data: `NODES`, `FLOWS`, `BANDS`
   - SVG render (string building, then a single `innerHTML` assignment)
   - Data: `SCEN`, `QCATS`, `QUIZ`
   - Interaction: state, `apply()`, detail panel, scenario UI, tabs, diagram clicks, filters
   - Quiz UI
   - Theme toggle
   - `setMode('explore')` boot

**Dependencies:** none, apart from Google Fonts (Inter, JetBrains Mono), which fall back to system fonts offline. No frameworks, no build.

### 4.2 Coordinate system and layout

- SVG `viewBox="0 0 1456 1200"`, scaled to width with `min-width: 1180px`. Narrow screens scroll horizontally inside `#scroller`.
- Left labels: x 16–165. Bands: x 170–1450.
- **Enterprise IT column:** x 182–432, y 160–812. It overlays bands 2–4 on the left because back-office systems are not on the timing axis.
- **Main area:** x 456–1428.
- **Band y-ranges:**

  | Band | y range |
  |---|---|
  | Wholesale | 40–140 |
  | Forecasting | 160–300 |
  | ADMS | 330–660 |
  | Head-ends | 680–812 |
  | Field | 832–1008 |
  | Physical | 1026–1190 |

- **Corridors** between bands carry routed lines. Corridor y-values in use: 150, 166, 312, 320, 383, 818, 826.
- **OT zone** is a hand-drawn polygon (`M448,338 H1434 V652 H1105 V804 H779 V652 H448 Z`) that wraps the ADMS band, DERMS and the FEP. The **DMZ** is a dashed rect around the historian.

Coordinates are hand-placed. Moving a box means updating the `d` paths of every flow that touches it. Two crossings were accepted as the least-bad option: Analytics → DRMS crosses ISO ↔ VPP, and SCADA → Historian crosses AMI ↔ OMS.

### 4.3 Data schemas

```js
// NODES: one per box
{ id, x, y, w, h,
  title, sub,            // heading + subtitle
  lines: [],             // body text lines (pre-wrapped; ~24 chars for 169px boxes, ~20 for field boxes)
  chip,                  // optional footer line
  strip: true,           // optional: render as a thin dashed strip (operator, network model)
  custom: true,          // only 'grid': rendered by a dedicated function
  text,                  // strip body text
  rel: [ids],            // logical links that are not drawn; used for highlighting and "Connected to"
  det: { does, in, out, rate } }  // detail panel content

// FLOWS: one per drawn connection
{ k: 'tel'|'ctl'|'plan'|'biz'|'bus',  // telemetry, control, forecasts/limits, business records, comms wiring
  d: 'SVG path',                      // hand-routed; may contain multiple M subpaths (merges, buses)
  e: [ids],                           // endpoints; used for highlighting and matching
  both: true,                         // arrowheads at both ends
  l: [x, y, text, anchor, colorKind] }// optional label

// SCEN: scenarios
{ id, title, blurb, overview, key,
  steps: [{ when, title, text, nodes: [ids], flows: ['a|b' or 'a|b|kind'] }] }

// QUIZ
{ c: categoryKey, q, o: [correct, wrong, wrong, wrong], x: explanation, n: [ids], f: [flowKeys] }
```

### 4.4 Core logic

- **`state`**: `{ sel, filter, mode: 'explore'|'scen'|'quiz', scen, step }`. Quiz state lives separately in `quiz`: `{ topic, order[], pos, answers{}, show }`.
- **`connectedTo(id)`**: union of flow endpoints, the node's own `rel`, and any node whose `rel` includes it (links are symmetric).
- **`matches(flow, key)`**: `"a|b"` matches any flow whose `e` contains both a and b; `"a|b|ctl"` also requires that kind. A bus flow with five endpoints therefore matches `"fep|recl"` and lights up as a whole.
- **`litSpec()`**: returns the active highlight set, or null. In scenario mode it is the current step; in quiz mode it is the current question once "Show this in the model" is pressed. Otherwise null, which means Explore-style highlighting.
- **`apply()`**: the single function that sets `dim`, `lit` and `sel` classes on flows and nodes. Every UI change ends by calling it.
- **Scenarios** prepend an auto-generated **Overview** step whose nodes and flows are the union of all steps. `revealLit()` scrolls the window and the horizontal scroller so the lit area is visible below the sticky step panel.
- **Quiz shuffle**: `perm(qi)` is a seeded linear congruential generator (LCG) based on question index, so option order is stable between visits. Current distribution of correct letters across 50 questions: A 16, B 9, C 13, D 12.
- **Tabs** use `data-show="explore|scen|quiz"` on elements. The CSS rule `[hidden]{display:none!important}` is required because some components set `display:flex`.

### 4.5 Persistence

| localStorage key | Contents |
|---|---|
| `grid-theme` | `'light'` or `'dark'` |
| `grid-quiz-v1` | `{ topic, order, pos, answers }` |

**Important:** quiz answers are keyed by **array index** into `QUIZ`. Add new questions at the **end** of the array. If you reorder or delete questions, bump the key to `grid-quiz-v2`, or saved progress will point at the wrong questions. Reordering also changes each question's shuffle.

All storage access is wrapped in try/catch. The page works when storage is unavailable; it just won't remember anything.

### 4.6 Accessibility and input

- Nodes are focusable (`tabindex=0`, `role=button`); Enter or Space selects.
- Scenarios: ← and → change steps.
- Quiz: A–D or 1–4 to answer, ← and → to navigate.
- The detail and step panels use `aria-live="polite"`.
- `prefers-reduced-motion` stops the dash animations and makes scrolling instant.

### 4.7 Hosting notes

The file also runs as a published claude.ai artifact. Published pages only allow scripts from a few CDNs and fonts from Google Fonts, and block all other network requests. The current file needs nothing else, so keep it that way.

### 4.8 Testing done

- `node --check` on the extracted script.
- Headless Chromium (Playwright) screenshots in dark and light mode, scenario steps and quiz states. No page errors.
- A validation script that checks every scenario and quiz `nodes` ID exists in `NODES` and every flow key matches at least one `FLOWS` entry. Re-run something like this after content edits:

```js
SCEN.forEach(sc => sc.steps.forEach(s => {
  s.nodes.forEach(n => byId[n] || console.warn(sc.id, 'bad node', n));
  s.flows.forEach(k => FLOWS.some(f => matches(f, k)) || console.warn(sc.id, 'bad flow', k));
}));
// same pattern for QUIZ[i].n and QUIZ[i].f
```

---

## 5. Domain logic: how the grid model works

### 5.1 The mental model

The diagram is organized by **response time**, from slow at the top to instant at the bottom, with **enterprise back-office systems** set off to the side:

| Layer | Contains | Acts on |
|---|---|---|
| Wholesale + transmission | Transmission EMS, ISO/RTO | 5 min to day-ahead |
| Forecasting, programs, aggregation | Grid Analytics, VPP, DRMS | Minutes to days |
| ADMS + control room | Operator, OMS, FLISR, Power Flow/SE, VVO, network model, SCADA; DERMS alongside | Seconds to minutes |
| Head-ends + communications | AMI head-end, Grid Comms/FEP, DER + DR head-end | Seconds (SCADA) to hours (meter batches) |
| Field devices | Meters, substation, reclosers/switches, regulators/caps, line sensors, fuses, DERs, flexible loads | Milliseconds (protection) |
| Physical grid | Transmission → substation → feeder → lateral → service transformer → customer | Continuous, 60 Hz |
| Enterprise IT (side column) | CIS + call center, work/crew management, GIS, historian (DMZ), MDMS | Not on the timing axis |

Principles the content follows throughout:

1. **Protection acts before software.** Reclosers, relays and fuses clear faults locally; FLISR only acts after a lockout.
2. **Head-ends communicate, applications decide.** Gateways move data; OMS, FLISR, MDMS and others make the decisions.
3. **SCADA is the only real-time path to grid devices.** DR and small DERs use internet and vendor-cloud paths instead.
4. **Power flow / state estimation is the calculation core.** FLISR, VVO and DERMS all depend on it.
5. **OT and IT are separated.** Control systems sit in a firewalled OT zone; the historian in the DMZ is how operational data reaches IT tools.
6. **AMI is great for outages and history, not real-time control.** Intervals arrive in batches; only events arrive in seconds.

### 5.2 Flow color semantics

| Color | Kind | Meaning |
|---|---|---|
| Cyan | `tel` | Telemetry and status |
| Amber | `ctl` | Control commands |
| Purple | `plan` | Forecasts, schedules, limits |
| Pink | `biz` | Business and customer records |
| Grey | `bus` | Communications wiring (no direction) |
| Green | — | Electric power, physical strip only |

Two-way exchanges use arrowheads at both ends, with the dominant kind's color and a label explaining both directions.

### 5.3 What was wrong in v1 and how it was fixed

| v1 issue | Fix in v2 | Why it matters |
|---|---|---|
| Demand response arrow landed on SCADA | DRMS → DER + DR head-end (vendor clouds, OpenADR) → flexible loads | DR never travels over SCADA. This is a common newcomer misconception. |
| Analytics exchanged "forecasts, limits" directly with SCADA | Analytics reads from the historian and MDMS; sends forecasts to Power Flow/SE, VPP and DR | SCADA doesn't consume forecasts, and IT analytics shouldn't connect directly into OT. |
| Analytics → VPP/DR described in text but not drawn | Drawn | Text and arrows contradicted each other. |
| MDMS and DERMS in the "head-end" band | MDMS moved to enterprise IT; DERMS beside ADMS | They are applications, not gateways. v1's own text said DERMS sits alongside ADMS. |
| DERMS had no source for grid limits | Power Flow/SE → DERMS | Checking dispatch against limits is DERMS's core job. |
| OMS pings shown one-way (up) | Two-way AMI ↔ OMS | Pings go down; replies and last gasps come up. |
| No FLISR ↔ OMS link | FLISR → OMS switching updates | The outage model must reflect restoration switching. |
| DER protocols listed as "IEEE 2030.5, SunSpec Modbus" | IEEE 2030.5, DNP3 for large sites, vendor clouds; IEEE 1547-2018 noted | Modbus is mostly an on-site protocol; DNP3 is common utility-to-DER. |
| "15-min reads" implied near-real-time | Batches every 4–24 h; events in seconds | Correct expectations about what AMI can do. |
| "Planning" band labeled hours to days | Renamed "Forecasting, programs + aggregation", minutes to days | "Distribution planning" in utilities means multi-year capacity studies. |
| Diagram implied full automation | Control room operator strip added | Most switching involves people; FLISR often runs supervised. |
| Missing systems | Added Power Flow/SE, CIS + call center, work/crew management, GIS, historian, transmission EMS, ISO/RTO | These are needed to explain real outage and DER workflows. |
| Missing equipment | Added substation (breakers, relays, LTC), regulators + caps, fuses, flexible loads | Fuses in particular explain why AMI and customer calls matter for outages. |
| No physical grid | Added a power-path strip, including solar backfeed | Gives terms like feeder, lateral and isolate a physical anchor. |
| No security boundary | Added the OT zone polygon and DMZ | Shows where data crosses from control systems into IT. |

### 5.4 Where the knowledge comes from

The content reflects **general North American utility practice and public standards**, drawn from Claude's training knowledge of the industry. It was **not** verified against the owner's utility's specific systems, vendors or procedures. Standards referenced:

| Topic | Source |
|---|---|
| DER interconnection capabilities (volt-VAR, ride-through, anti-islanding) | IEEE 1547-2018 |
| DER communications | IEEE 2030.5 (also used in California's Rule 21 / CSIP) |
| SCADA protocols | DNP3 (IEEE 1815); IEC 61850 in substations |
| Control-center data exchange | ICCP (IEC 60870-6 / TASE.2) |
| Demand response signaling | OpenADR |
| Service voltage limits | ANSI C84.1, Range A: 114–126 V on a 120 V basis |
| Reliability indices (SAIDI, SAIFI; sustained = more than 5 min) | IEEE 1366 |
| DER aggregations in wholesale markets | FERC Order 2222 |
| Under-frequency load shedding (first stage commonly 59.3 Hz) | NERC PRC-006 and regional programs |
| Critical infrastructure protection (mainly bulk electric system assets) | NERC CIP |

**Simplifications and caveats** (keep these in mind when editing):

- Customer counts, megawatt amounts and timings in the scenarios are **illustrative examples**, not real event data.
- Utilities vary widely. Some run DR through DERMS, some integrate OMS and DMS differently, some treat the AMI head-end as OT, and some sensors report through the FEP rather than a vendor cloud.
- The FEP bus lights up all of its devices together when highlighted.
- Relationships shown via `rel` rather than a line: EMS ↔ SCADA/operator/substation, MDMS → CIS billing, fuse ↔ OMS (by inference).
- **Recommendation:** have a subject matter expert from the owner's utility review the content before it is used for formal training.

---

## 6. How to make common changes

| Task | Steps |
|---|---|
| Edit a box's text | Change `lines`, `sub`, `chip` or `det` in `NODES`. Keep lines within the character widths in 4.3. |
| Add a box | Add to `NODES` with coordinates in a free slot; add `FLOWS` with hand-routed `d` paths; check for overlaps in a screenshot. |
| Add a connection | Add to `FLOWS`. Route along corridors; use `both: true` for two-way links. If routing is impractical, use `rel` instead. |
| Add a scenario | Append to `SCEN`. Steps reference node IDs and flow keys; run the validation snippet in 4.8. |
| Add quiz questions | Append to the **end** of `QUIZ` (see 4.5). Correct answer first. Add `n` and `f` for "Show this in the model". |
| Add a quiz topic | Add to `QCATS` and use the key in `c`. Topic buttons are generated automatically. |
| Change colors | Edit the tokens in both the light (`:root`) and dark (`:root[data-theme="dark"]` and the media-query block) sets. |

## 7. Known limitations and ideas

**Limitations**
- Hand-placed coordinates make layout changes laborious; there is no auto-layout.
- The page is designed for desktop. On phones the diagram scrolls horizontally, and the sticky step panel is capped at 46% of the screen height.
- Quiz progress is saved per browser, with no sync between devices.
- The light theme is complete, but it was tuned less carefully than the dark theme.

**Ideas raised or worth considering**
- More scenarios: predictive maintenance (asset health → work orders), wildfire risk shutoffs, EV charging clusters overloading a service transformer, cyber incident response at the IT/OT boundary.
- Utility-specific customization: swap generic names for the owner's actual systems and vendors.
- A glossary tab, or hover definitions for acronyms.
- Export or print view for training sessions.
- Version stamp and changelog embedded in the page footer.

## 8. Files

| File | Purpose |
|---|---|
| `grid-operations-diagram-v2.html` | The tool. Everything is in this file. |
| `README.md` | Short overview (what, who, how) and the "What changed from the first version" list. |
| `HANDOFF.md` | This document. |
| `index.html` | Redirect to the tool so the GitHub Pages root URL (https://franklan-pm.github.io/grid-mod-operations/) opens it. Update it if the main file is renamed. |
| `docs/screenshot.png` | README image of the Explore tab (dark mode, 2× scale). Retake after visual changes with headless Chrome: `chrome --headless=new --hide-scrollbars --force-device-scale-factor=2 --window-size=1500,1665 --virtual-time-budget=5000 --screenshot=docs/screenshot.png <file URL of the HTML>`. |
