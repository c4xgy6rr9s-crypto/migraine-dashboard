# iOS Shortcuts spec

The capture side is built in the iOS Shortcuts app (not in this repo). The Shortcuts append plain-text **event logs** to iCloud Drive; the dashboard reads them and joins them by date. **The log line formats below are the contract** — if you rebuild a shortcut, match the line format exactly and the dashboard keeps working.

> Status note: the line formats here are verified against real exports. The per-shortcut *build steps* for the migraine and weather shortcuts are described at the approach level — the exact internal wiring (especially how the weather block and `sex` are obtained) lives on the phone. Correct anything that doesn't match how you actually built them.

## Files (all in iCloud Drive)

| File | Written by | One line per |
|------|-----------|--------------|
| `migraines.txt` | Start/End migraine shortcuts | episode event (`auraStart` / `auraEnd` / `migraineEnd`) |
| `weather.txt` | daily weather automation | day |
| `sleep.txt` | Log Sleep automation | night |
| `drinkcontrol.csv` | DrinkControl app export (not a shortcut) | drink |

## The shared append recipe

Shortcuts has no native "append a line" action, so every logging shortcut ends with:

1. **Text** — the line to add (built per shortcut below).
2. **Get File** the target `.txt` in iCloud Drive, *Error If Not Found OFF*.
3. **Text** — the file variable, a literal newline (press Return), then the new-line variable:
   ```
   [FileContents]
   [NewLine]
   ```
   This produces `file + newline + line`. On the first-ever run the file is empty, so you get one leading blank line — **the dashboard ignores blank lines**, so no conditional is needed.
4. **Save File** to the same path, *Overwrite ON*, *Ask Where to Save OFF*.

**Lists use `~` as the separator** (e.g. `migraine=Sensitivity to light and sound~Nausea`). Build them with **Choose from List (Select Multiple)** → **Combine Text** with `~`. A separator that isn't a comma or space means symptom/treatment names can contain anything. An empty pick yields an empty value (`triggers=`), which is fine. Because there are no quotes anywhere, Smart Punctuation is harmless.

---

## Migraine log — `migraines.txt`

An episode spans **`auraStart` … `migraineEnd`** (migraines always begin with `auraStart`), with an optional `auraEnd` between. Verified format:

```
2026-06-24T21:02 | auraStart | cycle=8 | sex=1 | wxHigh=34°C | wxLow=22°C | wxCond=Sunny | wxPress=30.044 inHg | wxHum=57 | wxWind=5.037 mph | wxUV=0 | wxAQ=4
2026-06-24T21:03 | auraEnd
2026-06-24T22:30 | migraineEnd | prodrome=Mood changes~Yawning | aura=Blind spot | migraine=Sensitivity to light and sound~Nausea | sev=1 | triggers=Lack of sleep | treatment=Lying down in dark~Caffeine | eff=3
```

Fields, by event:

| Event | Fields |
|-------|--------|
| `auraStart` | `cycle` (bare int, day of cycle), `sex` (`1`/`0` = had sex in last 2 days), weather block: `wxHigh` `wxLow` `wxCond` `wxPress` `wxHum` `wxWind` `wxUV` `wxAQ` (units like `°C`/`inHg`/`mph` may stay in the value) |
| `auraEnd` | *(none — just the timestamp)* |
| `migraineEnd` | `prodrome` `aura` `migraine` (symptom lists), `sev` (1–5), `triggers` (list), `treatment` (list), `eff` (relief 1–5) |

The dashboard pairs the events into one episode, takes onset = `auraStart` time, and merges `migraineEnd` detail in.

### Start Migraine (auraStart) — approach
1. Current date → format `yyyy-MM-dd'T'HH:mm`.
2. `cycle` from Health (days since last period start).
3. `sex` — your 1/0 flag (e.g. a Yes/No menu → `1`/`0`).
4. Weather block — Get Current Weather → Get Details for each metric (high, low, condition, pressure, humidity, wind, UV, air quality).
5. (Optional) any symptoms you pick at onset.
6. Build the `auraStart` line and **append** (recipe above).
7. Optionally schedule the reminder that launches End.

### End Migraine (migraineEnd) — approach
1. Current date → format.
2. **Choose from List (Select Multiple)** → **Combine Text `~`** for: `prodrome`, `aura`, `migraine`, `triggers`, `treatment`.
3. **Choose from Menu** → `sev` (1–5) and `eff` (1–5).
4. Build the `migraineEnd` line and **append**.

`auraEnd` is just a timestamped line appended when the aura ends.

---

## Weather log — `weather.txt`

One line per day (a daily automation), same weather block as `auraStart`. Verified format:

```
2026-06-24T22:26 | weather | wxHigh=34°C | wxLow=22°C | wxCond=Mostly Clear | wxPress=30.049 inHg | wxHum=61 | wxWind=3.8 mph | wxUV=0 | wxAQ=4
```

This is the **baseline** the dashboard compares migraine-onset weather against (it z-scores pressure, temp, humidity, UV and air quality at onset vs. typical days). Build: a time-of-day **Automation** → Get Current Weather → Get Details → build the `weather` line → **append** to `weather.txt`. The dashboard keys by date and keeps the last reading of each day.

---

## Sleep log — `sleep.txt`

One line per night, written by the **Log Sleep** automation (date = wake date):

```
2026-06-23 | sleep | hrs=7.2 | awake=0.6 | bed=2026-06-22T23:10 | wake=2026-06-23T06:58
```

`hrs` = hours asleep (sum of asleep stages), `awake` = awake time during the night. `bed`/`wake` are optional (only needed for a future sleep-consistency metric). Apple's Sleep Score isn't exposed to Shortcuts, so the dashboard computes its own quality signal from these raw numbers.

### Log Sleep — steps (this one is fully specified)
1. **Date** (Current) → `Now`; **Adjust Date** −1 Day → `Yesterday`; **Format Date** `yyyyMMdd` → `YNum`; **Text** `[YNum]1800` → `CutNum` (6pm-yesterday cutoff as a comparable number).
2. **Find Health Samples** → **Sleep**, filter *Start Date is in the last 2 Days*, sort *Start Date, Oldest First* → `SleepSamples`. (The Health date filter only takes a static "last N days", so the precise single-night cutoff is done in the loop.)
3. **Number** `0` → Set `AsleepMin`; **Number** `0` → Set `AwakeMin`.
4. **Repeat with Each** `SleepSamples`:
   - **Get Details of Health Sample** → **Value** → `Stage`; → **Start Date** → `St`; → **End Date** → `En`.
   - **Format Date** `St` as `yyyyMMddHHmm` → `StNum`.
   - **If `StNum` is greater than `CutNum`** (i.e. last night):
     - **Get Time Between** `St` and `En` (Minutes) → `Seg`.
     - **If `Stage` contains `Awake`** → `AwakeMin + Seg`; **Otherwise If contains `Bed`** → skip; **Otherwise** → `AsleepMin + Seg`.
5. `hrs` = `AsleepMin / 60` (1 dp); `awake` = `AwakeMin / 60` (1 dp).
6. Build the `sleep` line and **append** to `sleep.txt`.
7. **Automation** → Time of Day (a fixed morning time; that fixed time is what makes the "6pm yesterday" cutoff isolate one night).

> Sleep-stage `Value` comes back as a bare stage name (`REM`, `Core`, `Deep`, `Awake`, `In Bed`), so the `contains Awake` / `contains Bed` / else logic counts the right segments.

---

## Alcohol — `drinkcontrol.csv`

Not a shortcut: export from the DrinkControl app (`;`-delimited CSV). The dashboard uses `AccountedForDate` and the app's pre-computed `TotalUnits(UK)` column. Re-export and re-import periodically; the dashboard attaches each migraine's prior-7-day unit total.
