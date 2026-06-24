# Migraine Tracker

A private migraine log and analysis dashboard. Capture episodes (plus sleep, alcohol and weather) on an iPhone with Shortcuts; review likely factors, trends and history in a single self-contained HTML file. **No data ever leaves your device.**

## Pieces

- **`index.html`** — the dashboard. One file, no dependencies, no build. Open it directly or host it on GitHub Pages. It loads your data locally in the browser and never makes a network request.
- **iOS Shortcuts** (not in this repo — they live on the phone) append plain text logs to iCloud Drive:
  - **migraines** — `auraStart` / `auraEnd` / `migraineEnd` lines per episode (symptoms, severity, triggers, treatment, cycle day, sex, and weather at onset).
  - **sleep** — one line per night (hours asleep, awake time), summed from Apple Health.
  - **weather** — one line per day (the baseline migraine onsets are compared against).
  - **alcohol** — exported from the DrinkControl app as a CSV.
  See `SHORTCUTS.md` for the capture specs and `CLAUDE.md` for the exact log formats.

## How it works

1. The Shortcuts log episodes, sleep and weather to text files as they happen.
2. Open the dashboard and **Load / Import** your files (migraine log, `sleep.txt`, `weather.txt`, and the DrinkControl CSV — all at once, or as you get them). It auto-detects each file by content and remembers them between visits.
3. The dashboard joins everything by date and shows **Likely factors** — a ranked, plain-English read on what seems to precede your migraines (cycle phase, short sleep, raised alcohol, weather, your noted triggers), followed by trends and a full history table.

## Likely factors — how to read them

The analysis is deliberately transparent and conservative:

- **Base-rate corrected** where a baseline exists — cycle phase is compared to how much of your cycle each phase actually occupies, and weather at onset is z-scored against your daily weather log. Raw counts alone would always crown common conditions.
- **Honest about small samples.** Until you've logged roughly 8–10 episodes a banner marks the factors as tentative, and any factor only appears once it has enough data behind it.
- **`sex` and self-reported triggers** are shown but flagged as descriptive — `sex` has no non-migraine baseline yet, so it can't be ranked as a trend.

## Using the dashboard

- **Load / Import** — pick any combination of your data files.
- **Export Backup** — download a timestamped `.json` of your episodes.
- **Save as PDF** — opens the print dialog (handy for taking to a doctor).
- Your last import is cached in the browser until you clear it.

## Hosting on GitHub Pages

`index.html` is at the repo root, so: repo **Settings → Pages → deploy from a branch → `main` / root**. The dashboard is then served at your Pages URL. **Your data still loads manually from your device every time** — it is never committed to the repo and never sent over the network, which is what keeps the health data private even though the page itself is public. The included `.gitignore` keeps real data files out of the repo; only the synthetic `sample-*` files are tracked.

## Privacy

This is health data, and the design keeps it local: the dashboard makes no network requests, has no analytics, and reads your files in-browser. Publishing the dashboard to GitHub Pages publishes only the code and fake sample data — never your logs.
