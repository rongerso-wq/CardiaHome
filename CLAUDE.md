# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

**CardiaHome** — a single-page mobile **dashboard concept** for an app that helps **Congestive Heart Failure (CHF) patients** run their daily self-check (weigh → blood pressure → symptom check), take their medications, and reach their care team. It serves all CHF patients but is **designed elderly-first**: every decision optimizes for the oldest, least tech-comfortable, possibly low-vision/shaky-handed user. It is a **design/stakeholder concept**, not a working product — all data is hardcoded placeholder, and there is no backend, telephony, or real Bluetooth.

Created by Dr. Roni Gershonovitch & Dr. Gal Lavie.

## Shape

One self-contained file: `index.html` — all CSS in one `<style>` block, all markup, and one vanilla-JS IIFE `<script>` at the end of `<body>`. **No build step, no dependencies, no network calls.** All icons, the zone ring/glyph, the sparklines, and the detail-screen trend charts are inline/JS-generated SVG. Fonts are the **system stack only** (the credit is system-sans italic — the earlier script/Comic-Sans fallback was removed in the audit pass) so the artifact stays fully offline and portable.

## Other files in this folder

The project is no longer just the app — it now also ships presentation/reference files:
- **`index.html`** — the app (everything else in this doc is about this file).
- **`pitch.html`** — one-page business pitch (headline, 3-sentence pitch, market stats, a phone screenshot, both founders' photos). For management / investors.
- **`tech.html`** — one-page "technology behind it" (architecture / clinical engine / design / quality, as a 2×2 grid).
- **`STATUS.md`**, **`PITCH.md`** — markdown versions of the project status and the pitch.
- **Assets:** `preview.png` (the app screenshot used in the pitch), `founder-roni.jpg`, `founder-gal.jpg` (headshots).
- **`Remote Monitoring Solutions … .pdf`** — user-supplied market/clinical reference. Don't edit.

**Portability gotcha (important):** `pitch.html` and `tech.html` have their images **embedded as base64 data URIs**, not linked — so a single file renders when emailed or opened on a phone. A relative `src="preview.png"` breaks the moment the HTML travels without its assets (this was a real bug). If you regenerate an image, re-embed it — PowerShell: read bytes → `[Convert]::ToBase64String` → string-replace the `src="<file>"` with the `data:` URI (Python is **not** installed on this machine).

**Regenerating `preview.png`** — capture the app's **own** phone frame; do *not* wrap it in a second CSS frame (that caused width-overflow / "cut" bugs). Copy `index.html` to a temp file, inject CSS to hide `#splash`/`#onboarding` and pin `.screen { height: 740px; overflow: hidden }`, drive it to the target state, and screenshot headless Chrome with `--default-background-color=00000000` (transparent) at ~`438×820 --force-device-scale-factor=2`.

## Run & verify

Open `index.html` directly in Chrome (`file://` works — double-click it). On launch you land on the welcome gate; press **Get Started** to reach the dashboard.

Because this is a **visual** artifact, verify changes by rendering a screenshot with headless Chrome rather than guessing:

```bash
"/c/Program Files/Google/Chrome/Application/chrome.exe" --headless --disable-gpu \
  --hide-scrollbars --force-device-scale-factor=1 --virtual-time-budget=1500 \
  --window-size=440,2100 \
  --screenshot="C:/Users/litbe/AppData/Local/Temp/cardia.png" \
  "file:///C:/Users/litbe/Roni Claude/CardiaHome/index.html"
```

Then read the PNG. Notes: `--virtual-time-budget` lets CSS animations settle before capture; the screenshot height = `--window-size` height, so use a **tall** window (~2100) to capture the whole scrolling page. The dashboard renders inside a CSS device frame, so the real content area is **~390px wide** — keep that width budget in mind when sizing text. To verify an interactive state, copy the file to a temp `.html`, append a `<script>` that drives it on `load` — first clear the two launch overlays with `document.getElementById('splash-start').click()` then `document.getElementById('ob-skip').click()`, then drive the dashboard (e.g. `document.querySelector('.card[data-detail="weight"]').click()`) — and screenshot that.

## Layout model

The UI is a **CSS phone-frame mockup** (`.device` → `.screen`) centered on a studio backdrop. **The page is one tall scrolling document**: `.screen` has no fixed height — it grows to fit its content (`min-height` only) and the whole window scrolls. `body` is `align-items: flex-start` so the top isn't clipped when the device is taller than the viewport. (Consequence: the `.statusbar` scrolls with the page, it is *not* pinned.)

Three layers, by `z-index`:
- **Dashboard** (base) — the scrolling `.content` column inside `.screen`.
- **Slide-in panels** (`#panel-weight`, `#panel-bp`, `#panel-messages`, z 40) — `position: fixed`, viewport-sized, slide in from the right by toggling `.open` (translateX). The messages panel is a flex-column chat (`.chat-inner`: scrolling `.chat` thread + pinned `.composer`).
- **Welcome gate** (`#splash`, z 50) — `position: fixed` full-viewport teal launch screen, dismissed by adding `.dismissed`.
- **Onboarding** (`#onboarding`, z 45 — between splash and panels) — first-run 3-step personalization (name → usual/`dry` weight → nurse), dismissed by `.dismissed`.

All three overlays (splash, onboarding, panels) use the same **class-toggle pattern**; they're `position: fixed` (not absolute) specifically so they fill the viewport instead of stretching the tall document.

### Dashboard content (top → bottom)
1. **Brand lockup** (`.brand`) — heart mark + "CardiaHome".
2. **Topbar** (`.topbar`) — avatar, greeting, live date (filled by JS), notification bell.
3. **Red-flag alert** (`#alert`, `hidden` by default) — amber/red "contact your nurse" banner + a red call button; shown and styled by `updateZone()` (see the zone engine in Behavior — there is no `updateAlert`).
4. **Heart-status hero / Zone indicator** (`.hero`) — the clinical **Green/Yellow/Red zone** card (replaced the old "92 score"). A circular ring with a center glyph (check for green, exclamation for yellow/red — two `<g>` groups toggled by CSS), the zone title, and a plain-language reason. Zone color via `.hero.zone-yellow` / `.hero.zone-red` classes set by `updateZone()`.
5. **Emergency button** (`.emergency`) — the most prominent element by design: full-width gradient-red "Call Community Nurse", ≥64px tall. **Red is reserved exclusively for this** (amber = "contact today", red = "call now"); never use red for status. Below it, the calmer **`.message-cta`** ("Message your care team", with a `#mc-unread` badge) opens the chat panel — the async counterpart to the synchronous call.
6. **Today's Check** (`.cards`, **4 tasks** → "X of 4 done") — Scale (180 lbs), Blood Pressure (125/80), and **Oxygen / SpO₂** (97%) — each a metric card with a tinted icon tile (`.tile.scale/.bp/.spo2`), its own status badge, a tap-to-log `.ring`, a 7-day sparkline, and a detail panel (`#panel-weight`/`#panel-bp`/`#panel-spo2`, each with a stepper); plus "How do you feel today?" symptom rows (good/bad **SVG faces**, not emoji) — **4 rows: Breathing, Leg Swelling, Fatigue, and Chest pain / can't breathe** (the last row is `[data-urgent]` and maps to Red zone if bad, not Yellow). A `.synced` chip frames the vitals as **auto-synced from Apple Watch** (the product vision: most data flows from the watch).
6b. **Today's Tip** (`.tip`) — a zone-aware personalized-feedback card (`#tip-text`, set by `updateZone()`; green rotates through `TIPS_GREEN`, yellow/red give targeted salt/rest advice).
7. **Medications** (`.medcard`) — leads with a weekly **adherence** strip (`.adherence`: `#adh-pct`, `#adh-streak`, a 7-day `.adh-ring` grid with `#adh-today` live), then the CHF regimen grouped by time of day, each row tap-to-take.
7b. **Learn about your heart** (`.learn`) — a single-open education accordion (`[data-learn]` items: daily weighing, low-salt, warning signs, medications).
8. **Connected Devices** (`.devices`) — Apple Watch / Smart Scale / BP Cuff, Bluetooth-linked.
8b. **Safety disclaimer** (`.disclaimer`) — "concept demo · not a medical device · call 911 in an emergency · **if you feel worse, seek care even if this app shows green**" + the weight-zone source line (regulatory/safety framing; see the SaMD note below).
9. **Credit** (`.credit`) — footer in system-sans italic with a teal→blue gradient text-clip (no script/cursive font stack — that was removed in the audit pass to avoid a Comic Sans fallback).

## Behavior (all in the single IIFE)

- **Daily-check progress** — the two metric cards (`.card[data-task]`) and the feel-card (`[data-feelcard]`) drive a "X of 4 done" counter + dots. `setDone(card,on)` + `refreshProgress()`. The feel-card counts done once all **four** symptom rows have a selection.
- **Quick-log vs. open detail** — the metric cards carry `data-detail="weight|bp"`. Tapping the **card body** opens its detail panel; clicking the **`.ring`** quick-logs (stopPropagation → `setDone`). Inside a panel, `.panel-cta` (`data-log`) marks the matching card done and closes.
- **Clinical zone engine** (`computeZone()` + `updateZone()`) — the core clinical logic, **multi-vital**. Models: `weight { dry, today, min:100, max:350, hist[7] }`, `bp { sys, dia, min_sys, max_sys, min_dia, max_dia }`, `spo2 { val, min:75, max:100 }` (BP + SpO₂ are "auto-synced from the watch"). Rules: **weight** ≥5 lb/day → Red; ≥3 lb/day, ≥3 lb/2-day, ≥5 lb/week, or ≥5 lb above `dry` → Yellow; **BP** ≥180/110 → Red, ≥160/100 → Yellow; **SpO₂** <90 → Red, <94 → Yellow; **urgent symptom** (`[data-urgent]` feel-row with `.sel-bad`, i.e. "Chest pain / can't breathe") → Red; any other flagged symptom → Yellow. `badSymptoms()` queries non-urgent rows; `urgentSymptoms()` queries `[data-urgent]` rows. `computeZone()` returns `{ zone, reasons, sub:{weight,bp,spo2} }`; overall = `worstOf(...)` (thresholds follow ACC/AHA HF guidance — cited in-app). `updateZone()` recolors the hero (`aria-live="polite"` on `.hero` announces changes to screen readers), sets `#alert` (**alert tails no longer claim auto-alert** — copy tells patient to call nurse/911 themselves), badges each card by its **own** sub-zone (`setBadge`/`setStat` — both use DOM API, no `innerHTML`), **persists today's readings to `localStorage`** (`ch_today_weight`/`ch_today_bpsys`/`ch_today_bpdia`/`ch_today_spo2`), and **redraws all three trend charts live** (last point = the current reading). On load the model block reads those `ch_today_*` keys back (recovery) — **all reads are `Number.isFinite()`-guarded** so corrupted storage can't silently NaN the zone math. Called from `refreshProgress()` and every stepper.
- **Steppers** (detail screens, drive the zone live) — Weight `#wt-minus`/`#wt-plus` → `setWeight()` (clamp 100–350); **BP has two steppers** — Top/Systolic `#bp-sys-minus`/`-plus` + Bottom/Diastolic `#bp-dia-minus`/`-plus`, each **±2**, both routed through one `setBP()` that clamps sys & dia (`min_sys`/`max_sys`/`min_dia`/`max_dia`); Oxygen `#sp-minus`/`-plus` → `setSpo2()` (±1). All coerce with `Number()` and call `updateZone()`. **Every +/- button is bound via `pressHold(btn, action)`** (not a plain click): pointerdown fires one step, then after ~0.4s auto-repeats every 90ms until pointerup/leave; Enter/Space still single-step. (`.step-btn` has `touch-action:none` so a hold doesn't scroll/zoom.) The weight panel also has `.scen` chips (`SCEN`: steady/two/week/spike) that load a 7-day `weight.hist` to **isolate one zone rule** each. (The BP panel no longer shows a reading-history list — just the systolic trend chart.)
- **Detail trend charts** — `drawChart(svg, values, color)` builds the chart with the **SVG DOM API** (`createElementNS`/`setAttribute` via the `svgEl()` helper — *not* `innerHTML`) and validates inputs (`color` must match `/^#[0-9a-f]{3,8}$/`, values coerced to finite numbers). Gridlines + area-gradient + highlighted last point; auto-scales with headroom. Drawn for `#chart-weight`, `#chart-bp`, `#chart-spo2`, and **re-drawn inside `updateZone()` on every change** so each chart's last point tracks the live reading (bp/spo2 use a fixed 7-point history + the current value; weight uses `hist.concat(today)`). **Keep the no-`innerHTML` discipline if you extend it** — this is the one spot that would become an XSS sink if wired to real data.
- **Medications & adherence** — each `[data-med]` row's `.med-check` toggles `.taken`; `refreshMeds()` updates `#med-count` ("X of N taken", independent of the 3-task daily counter) **and** the weekly adherence readout — `#adh-pct` (prior 6 days = 23/24 doses + today's live progress), the `#adh-today` ring state (none/partial/full), and `#adh-streak`.
- **Emergency / splash** — the emergency button swaps to a green "Connecting to nurse…" state for ~3s; the splash "Get Started" adds `.dismissed`.
- **Onboarding / personalization** — `#onboarding` runs `obShow(step)` over 3 steps; `applyValues()` writes the first name into `.greeting` + `.avatar`, the nurse name into `.chat-name` + `.chat-avatar`, sets `weight.dry`, and calls `updateZone()` so the zone baseline is patient-specific. `applyOnboarding()` also persists to `localStorage` (`ch_onboarded`/`ch_name`/`ch_nurse`/`ch_dry`, all `try/catch` — file:// may block it); on load a saved flag restores values and `.dismissed`es the overlay. Tapping `.avatar` re-opens it. **Defaults (John / 178 lb / Sarah) match the static markup, so Skip is a no-op.**
- **Care-team messaging** — `#open-messages` (and the topbar bell) call `openMessages()` → opens `#panel-messages` and clears the unread badges. `addMsg(side, text)` appends a bubble via the **DOM API + `textContent`** (never `innerHTML`); `sendUser()` (from the `.qr` quick-reply chips, the send button, or Enter in `#msg-input`) adds the patient bubble then `nurseReply()` shows a typing indicator and a **keyword-routed** canned reply (weight / breathing / question / generic). Simulated — no real backend.

## Design system

All visual tokens are CSS custom properties in `:root` (palette, hero gradient `--hero-1/2/3`, type scale `--t-*`, radii, shadows, `--tap-min: 64px`). **Re-skin the whole app by editing `:root`.** Aesthetic is "Premium Clinical": warm off-white, near-black ink (high contrast), one calm green for all positive status (`--calm` = `#0F6B47`, darkened for AA text contrast on `--calm-bg`), one alarm red, soft 24px+ corners, layered depth. Border-radius follows a 3-step scale: containers 26px (`--radius-card`), buttons 18px, small icon chips 14px.

**Elderly-first invariants — preserve these in any change:** large readable type, high contrast (AA), the emergency button staying the dominant element above the fold, and large tap targets — primary logging controls (`.ring` 46px, `.med-check` 48px) and buttons should stay ≥46px; reserve `--tap-min` (64px) for the headline CTAs.

## Accessibility & audit

This file passed a design (Agent Gourges) + security (Agent Smith) review; fixes are applied. Decisions worth preserving:
- **Contrast** — status green darkened (`--calm` `#0F6B47`), alert message darkened (`#6A3F08`), hero secondary text opacity raised. Don't lighten these back.
- **Tap targets** — `.ring`→46px, `.med-check`→48px (were 38/42). Keep primary logging controls ≥46px.
- **Motion** — a `@media (prefers-reduced-motion: reduce)` block disables all animations/transitions (the heartbeat logo, pulse pips, alert slide-in run infinitely otherwise). Keep it.
- **Symptom faces** are SVG, not emoji (rendering consistency).
- **Charts** are built via the SVG DOM API, never `innerHTML`. **`setBadge` also uses DOM API** (`createElement`/`createTextNode`) — no `innerHTML` anywhere in the file.
- **Selected symptom faces** show a `✓` badge (`::after` black circle) in addition to the color change — selection is not conveyed by color alone (WCAG SC 1.4.1).
- Security: verified **fully offline** — no external URLs, fonts, `fetch`, `eval`. All patient/medication/clinician data is **fictional sample data** (noted in an HTML comment above `<main>`).
- **CSP**: all three files (`index.html`, `pitch.html`, `tech.html`) carry a `Content-Security-Policy` meta tag. `index.html` allows `unsafe-inline` for scripts/styles (required by the inline IIFE + style block) and `img-src data:` only; `connect-src 'none'` enforces offline. Companion pages have no `script-src` at all.
- **Zone thresholds (ACC/AHA)**: weight ≥5 lb/day → Red; ≥3 lb/day, ≥3 lb/2-day, ≥5 lb/week, or ≥5 lb above dry → Yellow. All four weight rules are now independent `if` blocks (not `else if`), so multiple simultaneous triggers all surface in the reasons string. BP ≥180/110 → Red, ≥160/100 → Yellow. SpO₂ <90 → Red, <94 → Yellow.
- **localStorage expiry**: vital readings (`ch_today_weight`, `ch_today_bpsys`, `ch_today_bpdia`, `ch_today_spo2`) are only restored if the companion `ch_today_date` key matches today's ISO date — stale readings from prior sessions are discarded on load, not silently fed into the zone engine.
- **Input guards**: `#ob-name` and `#ob-nurse` have `maxlength="40"` + `autocomplete="off"`; `sanitizeName()` strips non-letter characters before use. `#msg-input` has `maxlength="1000"` + `autocomplete="off"`. Nurse reply uses `firstName()` dynamically, not a hardcoded name.
- **Regulatory framing:** the in-app `.disclaimer` marks this as a concept demo, not a medical device. Research note for any productization — the moment the zone engine *advises action on readings* it likely becomes **Software as a Medical Device** (FDA Class II / EU MDR Class IIa: HIPAA/GDPR, ISO 13485, clinical evaluation, cybersecurity). Keep the disclaimer until that path is taken.

## Gotchas

- **Flex children must not shrink.** `.content` is a flex column; `.content > * { flex: 0 0 auto; }` pins each section to its natural height. Without it, flexbox compresses and *clips* tall sections. If a section looks truncated, suspect this first.
- **Overlays must be `position: fixed`, not `absolute`.** Because `.screen` is as tall as the whole document, an absolute overlay would stretch to the full document height instead of covering one viewport.
- **One-line titles.** At ~390px content width, oversized type wraps easily; `.card[data-task] .card-title` is `nowrap` and sizes are tuned to fit. Bumping `--t-title` or the emergency label up re-introduces wrapping.
- **Hero ring rotation is scoped to the direct child** (`.hero-ring > svg`, not `.hero-ring svg`). The progress ring is rotated −90°; the descendant selector also caught the nested zone-glyph SVG and rotated it (a check rendered as ">"). Keep the `>`.
- **Building SVG children:** add zone glyphs etc. as markup `<g>` groups toggled by CSS, or via `createElementNS` — setting `innerHTML` on an SVG element risks wrong-namespace children that don't render. (Same rule as the charts.)
