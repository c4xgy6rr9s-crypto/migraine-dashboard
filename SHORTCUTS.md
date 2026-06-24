# iOS Shortcuts spec

These shortcuts are built in the iOS Shortcuts app (not in this repo). This file documents what they do so the dashboard's expected data format stays in sync. If you rebuild them, match the JSON output exactly.

## Shared file

- Location: iCloud Drive → `Shortcuts/migraines.txt`
- Format: a single flat JSON array of episode objects (see `CLAUDE.md` for the shape).
- Extension is `.txt` because Shortcuts' "Save File" appends `.txt` to text content; the contents are JSON. The dashboard reads it regardless.
- Only one episode is "open" (`"end":null`) at a time.

## Format: an append-only event log

Each shortcut **appends one plain line** per event. No JSON brackets, no quotes, no find/replace inside existing text. The dashboard pairs each `start` with the next `end` and builds the JSON on import (`parseEventLog` in `index.html`).

Line shape — pipe-separated fields, first two are timestamp + event type, the rest are `key=value`:

```
2026-07-09T19:30 | start | cycle=18 | wx=Clear | sym=Visual aura, Light sensitivity
2026-07-09T20:50 | end | trg=Stress | tx=Hot bath, Lying down | eff=2
```

Keys: `cycle` (bare int), `wx` (string), `sym`/`trg`/`tx` (comma-separated lists), `eff` (bare int 1–5). All optional; omit a key and the field is just empty. The parser trims everything and is case-insensitive on keys and the `start`/`end` type, so spacing doesn't matter. A `start` with no following `end` shows as **ongoing**.

Why this is easier than the old JSON-text approach: there's nothing to match exactly. Smart Punctuation no longer breaks anything (no quotes to curl), and you never edit existing file content — you only add a line.

### The one append pattern (used by both shortcuts)

Shortcuts has no "append line" action, so:
1. **Text** action = the new line (built with Replace/Combine, see below).
2. **Get File** `migraines.txt` in iCloud Drive, *Error if Not Found OFF*.
3. **If** [file] *has any value*: **Combine Text** with [file], [new line] using **New Lines** as separator → so the new line lands on its own row. **Otherwise**: just the new line.
4. **Save File** `migraines.txt`, Overwrite ON, Ask Where to Save OFF.

Lists (`sym`/`trg`/`tx`) come from **Choose from List (Select Multiple)** → **Combine Text** with `, ` (comma-space) as the separator. An empty pick yields an empty value, which is fine (`sym=`).

## Start Migraine

Home Screen button:
1. Current date → format `yyyy-MM-dd'T'HH:mm` → StartISO.
2. Pull cycle day from Health (days since last period start) → CycleDay. (Sleep and alcohol Health pulls were tried and dropped.)
3. Get Current Weather → condition → WeatherCond.
4. (Optional) Choose from List for symptoms → Combine Text with `, ` → SymJoined.
5. Build the line text:
   `<StartISO> | start | cycle=<CycleDay> | wx=<WeatherCond> | sym=<SymJoined>`
6. Append it (the pattern above).
7. Adjust Date +60 min → Add Reminder "Check aura" with that alert. Reminder notes contain `shortcuts://run-shortcut?name=End%20Migraine` to launch End.

## End Migraine

Launched from the reminder link (or pinned):
1. Current date → format → EndISO.
2. Choose from List (Select Multiple) → Combine Text with `, ` for each:
   symptoms → SymJoined, triggers → TrgJoined, **treatments → TxJoined**.
3. Choose from Menu 1–5 → Effectiveness (bare number).
4. Build the line text:
   `<EndISO> | end | sym=<SymJoined> | trg=<TrgJoined> | tx=<TxJoined> | eff=<Effectiveness>`
5. Append it (same pattern).

`end` fields merge into the open episode: list values (`sym`/`trg`/`tx`) are **unioned** with whatever Start recorded (deduped), and `cycle`/`wx`/`eff` overwrite if present. So treatments only need to be asked at End — that's where the new treatment questions live.

## Planned: free-text "Other…" entry

Pattern: present fixed Choose-from-List items, then an Ask for Input text prompt for extras; join the typed extras to the picked items with the same `, ` separator before building the line. Guard the case where nothing was picked (avoid a leading `, `). No dashboard change needed — it renders arbitrary strings.
