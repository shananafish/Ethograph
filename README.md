# ETHOGRAPH — Behavioral Network Recorder

A single‑file web app for **field behavioral data collection** with rhesus macaque social groups at ONPRC/OHSU. Tap animals on a live social‑network graph to record who did what to whom, with behaviors, qualifiers, affect, and duration — then sync to Google Sheets.

**▶ Live app:** https://shananafish.github.io/Ethograph/ethograph-behavior-logger.html

Built by Shannon (primate behavior/welfare specialist, BSU) in collaboration with Claude.

---

## Highlights

- 📋 **Roster** loaded from a Google Sheet (CSV) or pasted TSV/CSV
- 🕸️ **Force‑directed social network** — pan/zoom, drag nodes, dam–offspring nesting
- 👆 **Tap‑to‑record**: tap an initiator, tap a recipient, pick behaviors → save, via a **state‑aware action tray**
- 👥 **Multiple initiators/recipients** per record (coalitions, one‑to‑many), with **★ primary** designation and after‑the‑fact **member editing**
- ❓ **Generic "unknown" nodes** by demographic category, with a per‑record count
- 🔗 **Third‑party responses** — tap an edge while an initiator is selected to record a **redirect / intervene / seek‑aid / undirected** response, drawn as two edges routed through the referenced edge
- 🎨 **Type‑colored edges** — direct = blue, intervene = red, seek‑aid = green, redirect = yellow
- ⏸️ **Pause / resume a session** to check the dashboard mid‑observation
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
2. **Load the roster** — **Roster → Fetch from Google Sheets** (enter the roster password once; it's remembered on the device) — or paste a sheet.
3. **Select a group** — filter by group type, pick a group, then optionally narrow to a subset (tap demographic chips to bulk‑select, or check individual animals in the roster table). Leave everything unchecked for the whole group.
4. *(Optional)* **Group Dashboard** — jot group / daily‑observation / births / per‑animal notes for the session.
5. **Begin Session** — the network opens on "tap a node to start a new event."
6. **Record interactions** — the top **action tray** changes with what you've selected:
   - Tap an animal → it becomes the initiator; tap a second animal → recipient, and the behavior sheet opens automatically (**the 2‑tap fast path**).
   - The tray also offers **+Init / +Recip** (build a multi‑animal record — finish with **Done →**), **? Unknown** (add a generic placeholder), **Solo** (undirected), and **★ set primary** for the last‑tapped animal.
   - Choose behaviors/qualifiers/affect/duration and **Save**.
7. **Sync** to Google Sheets and/or **export CSV** from the home screen.

### Third‑party responses

With an initiator selected, **tap an existing edge** (its diamond) to record a response to that interaction:

- If the responder **wasn't** in that edge → it's assumed to be an **intervention**, and you're prompted straight for target(s).
- If the responder **was** in that edge → pick the type: **redirect · seek aid · undirected (solo)**.
- The response is tagged with its type and **linked** to the record it responds to, and is drawn as **two edges routed through the referenced edge's node**. **Redirections** also record who the animal redirected **away from**.

### Recording shortcuts

- **Tap the participants at the top of the behavior sheet** (they show a ✎) to jump back into node selection for that record — add/remove animals or set primaries, then **← Back to sheet**; your chosen behaviors are preserved.
- **Tap a record's diamond** (when nothing is selected) → **Edit record** (behaviors/duration/notes) · **⇄ Change members** · **View details**.
- **⏸ Pause** (recording header) drops back to home without ending the session; **▶ Resume session** on home returns you to recording.
- **Drag** nodes to rearrange; **pinch/scroll** to zoom; **⊙** resets the view.
- **Tap a node in the Baseline Network** view to see that animal's full roster info.

### Edge colors

Edges are colored by **interaction type**, not behavior domain (a record can carry aggression, submission, and affiliation at once):

| Type | Color |
| --- | --- |
| Direct | 🔵 blue |
| Intervene | 🔴 red |
| Seek aid | 🟢 green |
| Redirect | 🟡 yellow |

---

## Display preferences (⚙)

Saved per observer on the device. Options:

- **Layout** — Force · By matriline · By demographic · By full ID (grid) · By abbreviated ID (grid)
- **Color by** — Matriline · Random · Sex · Demographic · Uniform. Sex/Demographic use a preset palette scheme (Spectrum / Warm / Cool / Earth), and you can optionally assign a specific palette family per category (e.g. Female → Red→Purple, Male → Green→Teal).
- **Size by** — Demographic · Age · Uniform
- **Abbreviated ID** — Last 2 · Last 3 · Auto (shortest digits unique within the group)
- **Label rows** — choose which fields appear and in what order (Name / Dye / Abbr ID / Full ID). Up to 3 rows display; a 4th component acts as a **fallback** shown when an earlier field is blank. Each of rows 1–3 can be styled **bold** and/or **larger**.
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

Records are pushed to a Google Sheet via a bound **Apps Script web app** (header‑keyed rows, deduplicated and upserted by `Record_ID`). The **roster is also read through the Apps Script** (`doGet?pw=…`), gated by a password the user enters once per device — so the roster Sheet can be kept private rather than published. The Apps Script endpoint is configured as a constant near the top of the HTML file.

The output sheet should have header cells matching the record fields, including these columns added for the dashboard, third‑party, primary, and multi/unknown features:

```
TP_Type, Redirected_From, Primary_IDs,
Group_Notes, DOS_Notes, Births, Individual_Notes, Session_Notes
```

Notes:
- With **multiple participants**, `An1_ID` / `An1_Name` / `An2_ID` / `An2_Name` become comma‑joined lists.
- A **generic/unknown** participant appears as `UNK:<category>×N` (ID) and `<category> (unknown)×N` (name).
- **Third‑party responses** populate `TP_Type` (`redirect` / `intervene` / `seek_aid` / `undirected`) and `LinkedToRecord` (the label of the record responded to). `Redirected_From` holds the ID(s) a redirection moved away from.
- `Primary_IDs` lists any participants marked **★ primary**.
- Add matching header cells (`TP_Type`, `Redirected_From`, `Primary_IDs`) to the output sheet so the header‑keyed script writes them.

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
