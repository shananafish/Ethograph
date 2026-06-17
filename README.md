# ETHOGRAPH — Behavioral Network Recorder

A single‑file web app for **field behavioral data collection** with rhesus macaque social groups at ONPRC/OHSU. Tap animals on a live social‑network graph to record who did what to whom, with behaviors, qualifiers, affect, and duration — then sync to Google Sheets.

**▶ Live app:** https://shananafish.github.io/Ethograph/ethograph-behavior-logger.html

Built by Shannon (primate behavior/welfare specialist, BSU) in collaboration with Claude.

---

## Highlights

- 📋 **Roster** loaded from a Google Sheet (CSV) or pasted TSV/CSV
- 🕸️ **Force‑directed social network** — pan/zoom, drag nodes, dam–offspring nesting
- 👆 **Tap‑to‑record**: tap an initiator, tap a recipient, pick behaviors → save
- 👥 **Multiple initiators/recipients** per record (coalitions, one‑to‑many)
- ❓ **Generic "unknown" nodes** by demographic category, with a per‑record count
- ⚡ **Interventions & redirections** straight from a record's diamond
- 🎨 **Display preferences** (layout, color, size, labels, fonts) saved **per observer**
- 🗂️ **Group Dashboard** for per‑session notes (group, daily observation, births, per‑animal)
- ☁️ **Google Sheets sync** + **CSV export**
- 📱 **Installable PWA**, works offline, saves to the device automatically

---

## How it works

Data is organized as **Sessions → Events → Records**:

- **Session** — one observation outing (observer, group, conditions, start/end, notes)
- **Event** — a discrete observation period within a session
- **Record** — a single behavioral interaction (initiators, recipients, behaviors, qualifiers, affect, duration)

### Typical flow

1. **Set observer** — tap **👤** on the home screen and choose/enter your initials. Your display preferences are remembered for next time.
2. **Load the roster** — **Roster → Fetch from Google Sheets** (or paste a sheet).
3. **Select a group** — filter by group type, pick a group, then optionally narrow to a subset (tap demographic chips to bulk‑select, or check individual animals in the roster table). Leave everything unchecked for the whole group.
4. *(Optional)* **Group Dashboard** — jot group / daily‑observation / births / per‑animal notes for the session.
5. **Begin Session**, then **+ Record** to open the network.
6. **Record interactions** — tap an animal as the initiator (**An1**), tap another as the recipient (**An2**), choose behaviors/qualifiers/affect/duration, and **Save**. Use **⊕ Multi** for several animals per side, **? Unknown** to add a generic placeholder, or **Solo** for an undirected record.
7. **Sync** to Google Sheets and/or **export CSV** from the home screen.

### Recording shortcuts

- **Tap a record's diamond** → Edit · Intervention (the selected An1 steps in) · Redirection (the recipient redirects to a new target you tap) · View details.
- **Drag** nodes to rearrange; **pinch/scroll** to zoom; **⊙** resets the view.
- **Tap a node in the Baseline Network** view to see that animal's full roster info.

---

## Display preferences (⚙)

Saved per observer on the device. Options:

- **Layout** — Force · By matriline · By demographic · By full ID (grid) · By abbreviated ID (grid)
- **Color by** — Matriline · Random · Sex · Demographic · Uniform. Sex/Demographic use a preset palette scheme (Spectrum / Warm / Cool / Earth), and you can optionally assign a specific palette family per category (e.g. Female → Red→Purple, Male → Green→Teal).
- **Size by** — Demographic · Age · Uniform
- **Abbreviated ID** — Last 2 · Last 3 · Auto (shortest digits unique within the group)
- **Label rows** — choose which fields appear and in what order (Name / Dye / Abbr ID / Full ID); style row 1 bold/larger
- **Node scale & font size**

---

## Roster sheet format

The roster is a Google Sheet (published as CSV) or pasted text with a header row. Recognized columns:

| Column | Used for |
| --- | --- |
| `id` | Animal ID (required) |
| `name` | Display name |
| `dye` | Dye mark |
| `sex` | `m` / `f` (node shape) |
| `demog_cat` | Demographic category (size, grouping, generic nodes) |
| `current_group` | Group membership (group selection) |
| `matriline` | Matriline (color, clustering) |
| `dam` | Dam's ID (dam–offspring edges) |
| `age` | Age in years (size‑by‑age) |
| `birth` | Birth date (fallback for age) |

Any **additional columns** are kept and shown on the animal info card and the individual‑selection table — the app adapts automatically if you add or rename columns.

---

## Google Sheets sync

Records are pushed to a Google Sheet via a bound **Apps Script web app** (header‑keyed rows, deduplicated and upserted by `Record_ID`). The roster CSV URL and the Apps Script endpoint are configured as constants near the top of the HTML file (`SHEET_CSV_URL` and the sync URL).

The output sheet should have header cells matching the record fields, including these session‑level columns added for the dashboard and multi/unknown features:

```
Group_Notes, DOS_Notes, Births, Individual_Notes, Session_Notes
```

Notes:
- With **multiple participants**, `An1_ID` / `An1_Name` / `An2_ID` / `An2_Name` become comma‑joined lists.
- A **generic/unknown** participant appears as `UNK:<category>×N` (ID) and `<category> (unknown)×N` (name).

CSV export from the home screen includes all of the above without any setup.

---

## Install as an app (PWA)

Open the live URL in a mobile browser and choose **Add to Home Screen** (iOS Safari) or **Install** (Android Chrome). It runs full‑screen and works offline; data is stored on the device (`localStorage`) and survives restarts. Use **↑ Sync** / **CSV** to get data off the device.

---

## Deployment

This is a **single static file** — no build step, no dependencies except Google Fonts. To deploy via GitHub Pages, commit `ethograph-behavior-logger.html` to the repo; GitHub Pages serves it at the live URL above. To ship changes, edit the HTML and push.

---

## Privacy & data

- All data lives **on the device** until you sync or export. Preferences are per‑observer, per‑device (no account/login).
- Cross‑device syncing of settings is not supported (would require a backend).

---

## Tech notes

- Single self‑contained HTML file (inline CSS/JS), `localStorage` persistence, PWA manifest + service worker.
- SVG network with a custom force layout; live records use `Set`s and serialize to arrays when saved.
- See `ETHOGRAPH_PROJECT.md` for the detailed feature log and implementation notes.
