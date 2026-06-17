# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

**CardiaHome** — a single-page mobile **dashboard concept** for an app that helps **Congestive Heart Failure (CHF) patients** run their daily self-check (weigh → blood pressure → symptom check), take their medications, and reach their care team. Designed elderly-first: every decision optimizes for the oldest, least tech-comfortable, possibly low-vision/shaky-handed user.

Part of the **VitalHub** suite (hub: `https://rongerso-wq.github.io/VitalHub/`). The companion app is **Respiro** (COPD, at `https://respiro-ochre.vercel.app`). CardiaHome lives at `https://cardiahome.vercel.app`.

Created by Dr. Roni Gershonovitch & Dr. Gal Lavie.

## Shape

One self-contained file: `index.html` — all CSS in one `<style>` block, all markup, and one vanilla-JS IIFE `<script>` at the end of `<body>`. **No build step, no dependencies, no network calls.** All icons and trend charts are inline/JS-generated SVG. Fonts are the **system stack only**.

## Other files

- **`index.html`** — the app.
- **`pitch.html`** — one-page business pitch (headline, market stats, phone screenshot, founders' photos).
- **`tech.html`** — "technology behind it" architecture/clinical/design reference.
- **`STATUS.md`**, **`PITCH.md`** — markdown versions of status and pitch.
- **Assets:** `preview.png`, `founder-roni.jpg`, `founder-gal.jpg`.
- **`Remote Monitoring Solutions … .pdf`** — user-supplied reference. Don't edit.

**Portability:** `pitch.html` and `tech.html` embed images as base64 data URIs. If you regenerate an image, re-embed it via PowerShell (`[Convert]::ToBase64String`). Python is not installed on this machine.

## Run & verify

Open `index.html` directly in Chrome (`file://` works). The app now opens **directly to the dashboard** — no splash, no onboarding wizard. The onboarding div has `class="onboard dismissed"` pre-set in HTML so it's invisible before JS runs.

**No CSP meta tag** — a prior security audit added one (`default-src 'none'`) that blocked inline JS/Babel. It was removed. CSP lives in Vercel headers (`vercel.json`) only.

Headless screenshot:
```bash
"/c/Program Files/Google/Chrome/Application/chrome.exe" --headless --disable-gpu \
  --hide-scrollbars --force-device-scale-factor=1 --virtual-time-budget=1500 \
  --window-size=440,2100 \
  --screenshot="C:/Users/litbe/AppData/Local/Temp/cardia.png" \
  "file:///C:/Users/litbe/Roni Claude/CardiaHome/index.html"
```

## Layout model

On mobile: full-viewport card. On desktop (≥600px): centered card (max-width 480px, box-shadow), **no phone-frame mockup** — the black bezel/status-bar was removed. The `.statusbar` div is `display:none`.

Three overlay layers by z-index:
- **Dashboard** (base) — scrolling `.content` column inside `.screen`.
- **Slide-in panels** (`#panel-weight`, `#panel-bp`, `#panel-spo2`, `#panel-messages`, z 40) — `position:fixed`, slide in from right via `.open` (translateX).
- **Onboarding** (`#onboarding`, z 45) — starts with `.dismissed` class in HTML. Tap `.avatar` to re-open for editing profile. Auto-applies defaults (John / 178 lb / Nurse Sarah) on first visit.
- **Urgent modal** (`#urgent-modal`, z 60) — appears when user taps "bad" on the chest-pain/can't-breathe symptom row.

## Dashboard content (top → bottom)

1. **← VitalHub link** — fixed position top-left, always visible, links back to the hub.
2. **Brand lockup** (`#brand-tap`) — heart mark + "CardiaHome". **Tap 5× fast** to toggle demo/presentation mode (reveals scenario chips in the weight panel). A small "Demo Mode" button appears in the bottom-right corner when active.
3. **Topbar** — avatar (tap to re-open onboarding editor), greeting, live date, notification bell.
4. **Red-flag alert** (`#alert`, `hidden` by default) — amber/red "contact your nurse" banner; shown by `updateZone()`.
5. **Heart-status hero** (`.hero`) — Green/Yellow/Red zone card. Ring + center glyph (check vs. exclamation). Zone color via `.hero.zone-yellow` / `.hero.zone-red`.
6. **7-day zone strip** (`#week-dots`) — row of 7 colored dots (Su→Today) showing each day's stored zone. Empty circles for days with no data. Updated by `updateWeekStrip()` on every `updateZone()` call.
7. **Emergency button** — full-width "Call Community Nurse", ≥64px.
8. **Message care team** (`#open-messages`) — opens chat panel.
9. **Today's tip** (`#tip-text`) — zone-aware personalized feedback.
10. **Today's Check** (4 tasks: Scale, BP, Oxygen, Feel) — metric cards + detail panels.
11. **Medications** — weekly adherence strip + 4 CHF med rows (Furosemide, Lisinopril, Carvedilol, Spironolactone). Taken state **persists per day** to localStorage.
12. **Learn** — 4-item education accordion.
13. **Connected Devices** — "Coming Soon" section. Shows Bluetooth BP Cuff, Smart Scale, Apple Watch as planned integrations (greyed, labelled "Planned"). Not wired to any Bluetooth API. Badge reads "Coming Soon".
14. **Safety disclaimer** + **Created-by credit** (Dr. Roni Gershonovitch & Dr. Gal Lavie).

## localStorage key inventory

All keys prefixed `ch_`:

| Key | Value | Notes |
|-----|-------|-------|
| `ch_onboarded` | `'1'` | Onboarding complete flag |
| `ch_name` | string | Patient first name |
| `ch_nurse` | string | Nurse name |
| `ch_dry` | number string | Dry weight (lbs) baseline |
| `ch_today_date` | `YYYY-MM-DD` | Date of last vital entry |
| `ch_today_weight` | number string | Today's weight (lbs) |
| `ch_today_bpsys` | number string | Today's systolic BP |
| `ch_today_bpdia` | number string | Today's diastolic BP |
| `ch_today_spo2` | number string | Today's SpO₂ |
| `ch_wt_YYYY-MM-DD` | number string | **Per-day weight** for 7-day history |
| `ch_zone_YYYY-MM-DD` | `green\|yellow\|red` | **Per-day zone** for week strip |
| `ch_meds_YYYY-MM-DD` | JSON `{"0":bool,…}` | **Per-day med taken state** (0–3 = Furosemide→Spironolactone) |

**Recovery logic:** On load, `ch_today_*` vitals are only restored if `ch_today_date` equals today. `loadWeightHistory()` builds `weight.hist[6]` (oldest→newest, indices 6→1 days ago) from `ch_wt_*` keys. `loadMedState()` restores today's med checkboxes from `ch_meds_YYYY-MM-DD`. `updateWeekStrip()` reads 7 days of `ch_zone_*` keys.

## Behavior (IIFE)

- **Zone engine** (`computeZone()` + `updateZone()`) — multi-vital clinical rules (ACC/AHA HF thresholds): weight ≥5 lb/day → Red; ≥3 lb/day, ≥3 lb/2-day, ≥5 lb/week, or ≥5 lb above `dry` → Yellow; BP ≥180/110 → Red, ≥160/100 → Yellow; SpO₂ <90 → Red, <94 → Yellow; urgent symptom (`[data-urgent]` + `.sel-bad`) → Red; any flagged symptom → Yellow. `updateZone()` also saves `ch_wt_YYYY-MM-DD` and `ch_zone_YYYY-MM-DD` to localStorage, redraws charts, calls `updateWeekStrip()` and `updateWeightRows()`.

- **Weight history** — `loadWeightHistory()` runs on init and fills `weight.hist` from `ch_wt_*` keys (falls back to `weight.dry` for missing days). `updateWeightRows()` populates the "Today / Yesterday / …" rows inside `#panel-weight` with real stored values.

- **Medication persistence** — `loadMedState()` runs on init, restores `.taken` class on med rows from `ch_meds_YYYY-MM-DD`. `saveMedState()` fires on every med-check tap. Resets automatically each new day (new date key = no saved state).

- **Week strip** — `updateWeekStrip()` draws 7 `.wd` divs in `#week-dots`. Dot colors: `.wd-green`, `.wd-yellow`, `.wd-red`, `.wd-empty` (no data). CSS handles color.

- **Demo / presentation mode** — tap `#brand-tap` (the CardiaHome logo) 5× within 1.5 s to toggle `body.demo-mode`. In demo mode: `.scenarios` chips become visible in `#panel-weight`, `#demo-toggle` button appears bottom-right. Tap the button again to exit demo mode. Useful for investor presentations to show Green/Yellow/Red zone scenarios live.

- **Steppers** — Weight `#wt-minus/plus` → `setWeight()` (clamp 100–350); BP `#bp-sys-minus/plus` ±2, `#bp-dia-minus/plus` ±2 → `setBP()`; SpO₂ `#sp-minus/plus` ±1 → `setSpo2()`. All via `pressHold()` (tap = one step, hold = auto-repeat 90ms).

- **Trend charts** — `drawChart(svg, values, color)` via SVG DOM API (`createElementNS` — never `innerHTML`). Color must match `/^#[0-9a-fA-F]{3,8}$/`. Re-drawn live in `updateZone()`.

- **Medications** — `refreshMeds()` updates `#med-count`, `#adh-pct`, `#adh-today` ring, `#adh-streak`. Prior 6 days assumed 23/24 doses taken; today's count is live.

- **Onboarding** — 3 steps (name → dry weight → nurse). Starts pre-dismissed in HTML. On first visit (no `ch_onboarded`), `applyOnboarding()` runs with defaults. `applyValues()` sets `.greeting`, `.avatar`, `.chat-name`, `.chat-avatar`, `weight.dry`, then calls `updateZone()`. Tap `.avatar` to re-open.

- **Care-team messaging** — keyword-routed canned nurse replies (weight / breathing / question / generic). `addMsg()` uses `textContent` (never `innerHTML`).

- **Urgent symptom modal** (`#urgent-modal`) — shown when user taps "bad" on the chest-pain row. Offers "Call 911" link and a continue option.

## Design system

All tokens in `:root`: `--calm` `#0F6B47` (status green), `--alarm` `#D12B2B` (emergency red), `--hero-1/2/3` (teal gradient), warm off-white bg, near-black ink. Re-skin by editing `:root`. Elderly-first invariants: large type, high contrast (AA), emergency button dominant above the fold, tap targets ≥46px (`.ring` 46px, `.med-check` 48px, headline CTAs 64px).

## Accessibility

- `@media (prefers-reduced-motion)` disables all animations.
- Symptom faces are SVG (not emoji).
- Charts built via SVG DOM API — no `innerHTML` anywhere.
- Selected symptom faces show a `✓` badge (`::after`) — selection not conveyed by color alone (WCAG SC 1.4.1).
- `#alert` has `role="alert" aria-live="polite"`.
- Urgent modal has `role="alertdialog"`.
- All patient/med/clinician data is fictional (noted in HTML comment above `<main>`).

## Known JS gotchas

- **Curly/smart quotes** (`'` `'`) are NOT valid JS string delimiters. The file was previously broken by 5 smart-quote strings; all fixed (switched to double quotes). Never paste text from Word/Pages/ChatGPT into string literals without checking.
- **TIPS_GREEN and tipTextEl** are defined after the onboarding block in the IIFE. `updateZone()` guards `if (tipTextEl)` so calling it before those vars are assigned is safe.
- **`weight.hist`** is `[oldest, …, yesterday]` (6 entries). `weight.hist.concat(weight.today)` = 7-point chart. `hist[hist.length-1]` = yesterday.
- **Flex children must not shrink.** `.content > * { flex: 0 0 auto }` prevents compressed/clipped sections.
- **Overlays must be `position:fixed`.** `.screen` is full document height; absolute positioning would stretch instead of covering the viewport.

## Regulatory note

Once the zone engine advises action on real readings this likely becomes **Software as a Medical Device** (FDA Class II / EU MDR Class IIa). Keep the `.disclaimer` until that path is taken.
