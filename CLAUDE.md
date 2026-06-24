# CLAUDE.md — Migraine Tracker

Context for working on this project. Read this first.

## What this is

A personal migraine-tracking system with two halves:

1. **Capture** — iOS Shortcuts (built and maintained by the user in the Shortcuts app, not in this repo). They **append plain event-log lines** to `migraines.txt` in iCloud Drive (one `start` line, one `end` line per episode). The dashboard converts that log into the episode JSON on import. See `SHORTCUTS.md` for the line format.
2. **Dashboard** — `index.html`, a single self-contained file (no build step, no dependencies, no framework). Opens the JSON, shows stats/trends, exports PDF, imports/exports backups, and caches the last load in `localStorage`.

The dashboard is the only code in this repo. The Shortcuts live on the user's phone; this repo documents their expected output format so the two stay in sync.

## Hard constraints — do not break these

- **Single file, no dependencies.** `index.html` must stay self-contained: inline CSS, inline vanilla JS, no npm, no bundler, no external script/style tags, no CDN. It has to run by double-clicking the file or via GitHub Pages with zero tooling. **Intentional exception:** `apple-touch-icon.png` (180x180 lightning bolt) sits alongside `index.html` as the iOS "Add to Home Screen" icon — referenced via `<link rel="apple-touch-icon">` in the head. iOS data-URI icons are unreliable, so a real file was the deliberate choice. It's a static asset, not tooling/a dependency — don't "clean it up" back into the HTML.
- **No browser storage assumptions beyond localStorage.** localStorage is used for the data cache. It is wrapped in try/catch everywhere because it throws when the file is opened inside the Claude.ai artifact sandbox. Keep every storage access guarded.
- **The data formats are a contract with the Shortcuts.** Two surfaces: (a) the **event-log line format** the Shortcuts append (`parseEventLog`, keys `cycle/wx/sym/trg/tx/eff`), and (b) the **episode JSON shape** below that the log converts into and that backups use. Changing either means the user hand-edits Shortcuts on their phone. Don't rename casually. Additions that default gracefully when missing are fine. The dashboard auto-detects on import: text starting with `[`/`{` is parsed as JSON, anything else as the event log.
- **Privacy.** This is health data. It must stay local to the user's devices. Do not add analytics, telemetry, external fetches, or anything that sends data off-device. No third-party scripts.

## Migraine event-log schema (current)

The migraine file is an append-only event log. An episode spans **`auraStart` … `migraineEnd`**, with optional `auraEnd` between. Lists use **`~`** as the separator (deliberately, so item names can contain spaces/commas). Example:

```
2026-06-24T21:02 | auraStart | cycle=8 | wxHigh=34°C | wxLow=22°C | wxCond=Sunny | wxPress=30.044 inHg | wxHum=57 | wxWind=5.037 mph | wxUV=0 | wxAQ=4 |
2026-06-24T21:03 | auraEnd
2026-06-24T22:30 | migraineEnd | prodrome=Mood changes~Yawning | aura=Blind spot | migraine=Sensitivity to light and sound~Nausea | sev=1 | triggers=Lack of sleep | treatment=Lying down in dark~Caffeine | eff=3 |
```

- **`auraStart`** opens the episode (migraines **always** begin here — confirmed) and carries point-in-time context: `cycle` (bare int), `sex` (`1`/`0` = had sex in the last 2 days), and the weather block `wxHigh/wxLow/wxCond/wxPress/wxHum/wxWind/wxUV/wxAQ` (values may carry units like `34°C`, `30.044 inHg`; `parseFloat` strips them).
- **`auraEnd`** (optional) marks end of aura.
- **`migraineEnd`** closes the episode and carries the detail: `prodrome` / `aura` / `migraine` symptom groups (lists), `sev` (severity **1–5**), `triggers` (list), `treatment` (list), `eff` (relief **1–5**).
- Timestamps are local ISO `yyyy-MM-dd'T'HH:mm`. Onset = `auraStart` time; duration = `migraineEnd − auraStart`.
- **Robustness rules in `parseMigraineLog`:** the first event opens an episode (also accepts `migraineStart`/`start` for legacy data); `migraineEnd`/`end` closes; missing fields default to empty. The older `start/end` + `sym/trg/tx`/comma-list shape still parses. Exported JSON backups re-import via `normalizeJson`.
- **`sex` has no baseline** — logged only at onset, so it stays **descriptive** (History table + a clearly-caveated insight at strength ~0). To promote it to a real factor, log it on normal days too.
- **Weather now has a baseline.** A separate daily `weather` log (`<ts> | weather | wxHigh=.. | wxLow=.. | wxCond=.. | wxPress=.. | wxHum=.. | wxWind=.. | wxUV=.. | wxAQ=..`, parsed by `parseWeatherLog` into `WEATHER`) lets `computeInsights` z-score each migraine-onset metric (pressure, high temp, humidity, UV, AQ) against typical days and surface the top divergences (e.g. "lower pressure", "hotter days"). Condition strings aren't yet base-rate-ranked — possible future addition.

## Context data (sleep + alcohol)

Two extra files join to episodes by date; both are retrospective so they need no capture-time shortcut:
- **Alcohol** — DrinkControl CSV (`;`-delimited). Use `AccountedForDate` (date part) and the pre-computed **`TotalUnits(UK)`** column (already × NumberOfDrinks, = grams/8); sum per day. Dashboard attaches **units in the prior 7 days** to each episode.
- **Sleep** — `sleep.txt` event log, one line per night: `<wakeDate> | sleep | hrs=.. | awake=.. | bed=.. | wake=..`. Dashboard attaches the **previous night** (wake date == onset day) and a 7-day average. Apple's Sleep Score isn't reachable from Shortcuts, so the score is computed from raw sleep instead.

The dashboard imports all file types through one multi-file picker; `routeText` sniffs each by content (CSV header → alcohol; a `sleep` line → sleep; migraine event types → migraine log; a `weather` line → daily weather baseline). Migraine is checked before weather because `auraStart` lines also carry `wx` fields. Each type caches independently in `localStorage` (key `migraineLog.v2`), so loading one file doesn't wipe the others.

## Dashboard behaviour (current)

- Stats cards: episode count; frequency (raw count under 30 days span, extrapolated /mo only at >=30 days — deliberately avoids absurd early extrapolation); avg duration; avg treatment relief (/5).
- Sections: episodes-over-time scatter (dot height = duration), Symptoms bars, Triggers bars, Cycle phase bars, Weather bars, Treatments-used bars, History table.
- Cycle phase mapping: 1–5 menstrual, 6–13 follicular, 14 ovulation, 15–28 luteal, else "day N".
- **Multi-file import**: one picker accepts the migraine log, the DrinkControl CSV, and `sleep.txt` (any/all at once, or over time). `routeText` sniffs each by content and updates only that data type.
- **"Likely factors" is the headline.** `computeInsights()` returns ranked, plain-English findings: cycle phase (base-rate corrected vs ~28-day expectation), short sleep the night before vs the user's median, raised alcohol in the prior week vs typical weekly units, and self-reported triggers. Each only fires when it has enough data; a confidence banner warns while episodes < 5. Keep it honest — small n, transparent math, no fake p-values.
- Secondary sections: episodes-over-time scatter, Symptoms/Triggers/Cycle/Treatments bars, and a wide History table (now incl. severity, phase, prev-night sleep, prior-7-day units, weather).
- Export downloads a timestamped `.json` backup of the episodes (re-imports via `normalizeJson`).
- "Save as PDF" uses the browser print dialog (`@media print` block).
- localStorage cache (`migraineLog.v2`) stores all three data types and auto-restores on reopen. Snapshot, not live — re-import to refresh.

## Design / style conventions

- Editorial/print aesthetic: serif body (Iowan/Palatino/Georgia), monospace labels, terracotta + aura-purple + muted-green palette, thin rules, generous whitespace. Match this if adding UI; don't introduce a different visual language.
- Colour roles are CSS variables in `:root`. Reuse them.
- Bars: symptoms=aura purple, triggers=terracotta, cycle=green, weather=blue, treatments=teal.

## Open threads / likely next requests

- **Free-text entry in the Shortcut lists** — user is adding "Other…" + Ask-for-Input flows in Shortcuts (their side, not repo). Extras join the picked items with the same `, ` separator. No dashboard change needed; it already renders arbitrary strings.
- **Event-log format (done, 2026-06-24)** — Shortcuts now append plain `start`/`end` lines instead of hand-built JSON; the dashboard converts on import (`parseEventLog`). The new End treatment questions write to the `tx=` field. JSON import is still supported for backups.
- **GitHub Pages hosting** — this repo is structured for it (`index.html` at root). Just enable Pages on the default branch.
- **Auto-pulling data instead of manual import** — discussed and deferred. The blocker: a static page can't read iCloud Drive, and automatic+private+static can't all hold at once without a credential the page can't safely store. Don't implement a public-repo data sync without explicitly confirming the user accepts public health data. Current decision: manual import + localStorage cache.
- **Rate-normalised triggers** — base-rate correction is now applied to cycle phase in `computeInsights` (observed vs ~28-day expectation). Weather has **no baseline** (only logged at onset), so weather can't yet be a ranked factor — it's descriptive only. If she later logs daily weather, add weather to the base-rate-corrected factors.
- **Richer migraine schema + context joins (done, 2026-06-24)** — migraine log moved to `auraStart/auraEnd/migraineEnd` with phased symptoms, severity, and a weather block (`~` list separator). Dashboard rewritten around `parseMigraineLog` + alcohol/sleep joins + `computeInsights`. Verified via headless Edge against real data (alcohol7=20.87, sleepPrev=5.5h).
- **Sleep capture** — "Log Sleep" shortcut sums asleep stages from HealthKit "Sleep" samples (Apple's Sleep Score isn't exposed to Shortcuts) and appends to `sleep.txt`. Single-night cutoff is done in-loop via numeric `yyyyMMddHHmm` comparison (Health date filters take only static dates).
- **File System Access API** — could give desktop-Chrome one-click re-read, but doesn't work on iOS/Safari where the user mostly is. Low priority.

## Testing

No framework. Sanity-check by opening `index.html` directly in a browser (not inside an artifact sandbox — localStorage is blocked there) and loading a sample file. A couple of sample episodes are enough to verify rendering, the frequency logic, and the cache round-trip.
