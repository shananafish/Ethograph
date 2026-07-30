# ETHOGRAPH — Behavioral Network Recorder
*Project brief for Claude Code*

## What it is
A single-file HTML app (`ethograph-behavior-logger.html`) for field behavioral data collection with rhesus macaque social groups at ONPRC/OHSU. Built by Shannon (primate behavior/welfare specialist at BSU) in collaboration with Claude.

**Live URL:** https://shananafish.github.io/Ethograph/ethograph-behavior-logger.html

---

## Architecture
Single self-contained HTML file — all CSS, JS, and HTML inline. No build system, no dependencies except Google Fonts. Runs in browser, installable as PWA.

---

## Core data hierarchy
`Sessions → Events → Records`
- **Session:** observer, group, conditions, timestamps
- **Event:** a discrete observation period within a session
- **Record:** a single behavioral interaction (An1, An2, behaviors, qualifiers, affect, duration)

---

## Key features built
- Roster system: fetch from Google Sheets CSV or paste TSV/CSV
- Group/demog filter with chip selection
- Force-directed social network visualization (SVG, pan/zoom, node drag)
- Dam-offspring proximity in layout (offspring seeded/sprung below dam)
- Node labels: name line 1, dye·id line 2 (or full ID if no name/dye)
- Matriline colors, sex-based node shapes (circle=female, rect=male), demog-based sizes
- Behavior sheet: Agg+Sub tab (4-col grid), Affiliation tab, Interaction qualifiers tab, Affect tab, Notes tab
- Qualifiers per animal (An1/An2) within behavior tabs
- 3rd party behaviors (An1/An2) within Agg+Sub tab
- Simultaneity grouping (drag records together)
- Session screen: full event/record list, prev/next navigation, editable records
- Recording view: prev/next to browse saved events, record panel below network
- Google Sheets sync via Apps Script (header-keyed rows, Record_ID deduplication, upsert)
- LocalStorage persistence (survives app restart)
- PWA installable (manifest + service worker)
- Export CSV
- Sync All button, Clear data button with confirmation

---

## External endpoints

**Google Sheets Apps Script URL:**
```
https://script.google.com/macros/s/AKfycbw9Z7yQt67KBdOsJbNjS3QnR5aBdISnTQzQ8ImWCzC9fTUmIuC18Ndsgrec1IkK5T0j1A/exec
```

**Roster Google Sheet:**
```
https://docs.google.com/spreadsheets/d/1-Zw3zA7_QJmv_OzWRB0xD5DzcHGQJobpcwRwoq8tCdc/export?format=csv&gid=0
```

---

## Known bugs — all fixed 2026-06-16 (verified in browser)
1. ~~Simultaneity drag broken~~ → replaced with tap-to-group (⊕ Group → ⊕ Here); works mouse + touch; groups kept contiguous; panel robust to non-contiguous groups after ungroup
2. ~~Record reordering only works downward~~ → unified `moveRecord(from,to)`; works both directions, reaches top/bottom; touch maps `data-idx` (DOM≠array order with groups)
3. ~~Record editing flaky on mobile~~ → deterministic tap-to-edit (`rowTouchStart/Move/End` + `rowClick`) with movement slop; coexists with scroll, drag handle, action buttons
4. ~~Node movement feels stuck~~ → fixed leaked `window` mousemove listener (was the only one missing the AbortController signal); each view re-entry had been multiplying renders per drag move
5. ~~Auto zoom cuts off bottom~~ → `fitToContent` (shrink-to-fit, never >1:1) re-runs when the canvas size changes (record panel grow/hide, resize); leaves manual pan/zoom alone
6. ~~New session flaky after review~~ → `startSession`/`endSession` now fully reset recording state (was leaking stale `recNavIdx`/`activeEvent`/`records`)
7. ~~Can't delete solo events~~ → live in-progress event now shown in session view with Open + Delete (`deleteLiveEvent`)

---

## Roster access — password-gated (2026-06-18)
Roster is no longer fetched from the public CSV. Instead the app calls the Apps Script web app `doGet?pw=<password>&_=<ts>` (`mode:'cors'`, reads the response), and the user enters a **roster password once per device** (field in the Roster view, saved to `localStorage['ethograph_roster_pw']` only after a successful fetch; prefilled thereafter; init auto-fetch uses it). Wrong password → server returns `unauthorized` → app shows "Wrong roster password" and keeps the saved good one.

**Server-side setup required for this to work (do BEFORE pushing the HTML):**
1. In the Sheet, remove public sharing / unpublish the CSV (make it private — only the script owner can read).
2. Apps Script → Project Settings → Script Properties: add `ETHOGRAPH_ROSTER_PW` = the chosen password.
3. Add a `doGet(e)` that checks `e.parameter.pw === ETHOGRAPH_ROSTER_PW`, reads the roster sheet via `SpreadsheetApp.openById(...)`, and returns it as **TSV** (`rows.map(r=>r.join('\t')).join('\n')` — TSV avoids comma issues; `parseRoster` auto-detects the tab delimiter). Deploy access stays **"Anyone"** (Google-login would break the cross-origin fetch); our own pw check is the gate.
4. Until the `doGet` exists, roster fetch returns nothing useful — the **paste TSV/CSV** fallback still works.

**Write/sync uses the same credential (2026-06-18):** `postSessionToSheet` now POSTs `{pw, rows}` (the same `ethograph_roster_pw`) instead of a bare rows array, and refuses to sync if no password is set. `doPost` must read `body.rows` and check `body.pw === ETHOGRAPH_ROSTER_PW` (same Script Property as the roster read). Sync stays `mode:'no-cors'` (fire-and-forget; the app can't read the reply, so a rejected write shows the optimistic "synced" — acceptable since the same pw was already validated on read).

```js
function doPost(e){
  var SECRET = PropertiesService.getScriptProperties().getProperty('ETHOGRAPH_ROSTER_PW');
  var body; try { body = JSON.parse(e.postData.contents); } catch(_) { return ContentService.createTextOutput('bad'); }
  if (!SECRET || !body || body.pw !== SECRET) return ContentService.createTextOutput('unauthorized');
  var rows = body.rows || [];
  // …existing header-keyed upsert over `rows`…
}
```

Note: the password is typed (not in the page source), so it's a genuine gate for both read and write — but the deployed app itself is still public.

## Field-test bug fixes — 2026-06-18 (iPad/iPhone + records)
1. **Node drag broken on iPad/iPhone** → root cause: the drag re-ran `renderSVG()` each move, destroying the touch-start target so iOS stopped delivering `touchmove`. Now each node is wrapped in `<g data-nid>` and dragged **in place** via a group `transform` (`applyNodeDrag`/`endNodeDrag`); full re-render only on release. Works on touch + mouse.
2. **Reset (⊙) didn't restore view; nodes cut off after zoom/records** → `resetPanZoom` now calls `fitToContent` (frames all nodes) instead of a 1:1 transform.
3. **Bottom cut off on some iPads (can't see all records)** → `#app` now uses `height:100dvh` (vh fallback) so Safari's toolbar/home-indicator can't push the record panel off-screen.
4. **Generic-node picker couldn't be closed to record** → added a **Done** button; tapping any node also closes it (`tapNode`→`closeGenPicker`).
5. **Node overlap (many animals; generics overlap existing)** → force layout now spreads into a **virtual area that grows with N** (clamped there, not the canvas) with phase-2 passes scaled to group size; phase-1 iterations scale down for big groups (responsive). New generics get collision-aware placement (`placeGeneric`). 219-animal group → ~15 minor touches vs. edge-piling before.
6. **Couldn't edit a record when browsing a previous event** → saved-event edits now refresh the recording view (was only refreshing the session view); also fixed a bug where a saved-record edit spuriously mutated the live `records` array.
7. **Couldn't reorder/group records after ending an event** → the record panel's edit/reorder/group/ungroup/delete now act on the **currently-displayed** array (`curRecs()`: live event or browsed saved event via prev/next) and persist (`commitPanelChange`). Saved-event rows get the same full controls as the live event.

## Recording UX batch — 2026-07-29 (verified in browser)
Seven changes to the recording flow (verified via file:// preview + injected roster; sync payloads inspected):

1. **An1/An2 relabelled → "Init 1 / Recip 1"** across the recording UI (hints, sheet header, node picker, behavior summaries). **Data column names unchanged** (`An1_ID`, `An2_ID`, … stay as-is for sync/CSV/analysis stability).
2. **Multi lives in the per-node menu.** `⊕ Multi` now appears only *after* a node is selected (alongside Solo), seeded with the already-tapped animal. The builder is two toggle chips **`+ Init: …` / `+ Recip: …`** (tap to switch which side your taps add to) plus **`Record →`**. Empty recipients = solo. Fast path (tap Init 1 → tap Recip 1 → sheet) is unchanged.
3. **Change a past record's members.** Diamond menu → **⇄ Change members** enters the multi builder seeded from that record's initiators/recipients; **`Save members ✓`** writes the new membership back to the same record and **leaves behaviors/qualifiers/affect untouched**. Live records in the current event. Fns: `startEditMembers`/`saveMembers`/`cancelEditMembers`, `membersEdit` state.
4. **Session start shows the network** (not the session summary): lands on the recording view with "tap a node to start a new event" so you can begin an event immediately.
5. **Label prefs:** up to **3 label rows** render; a 4th configured component acts as a **fallback** (shown when an earlier row's value is NA). Added **Bold/Larger style toggles for rows 2 and 3** (`labelBoldRow2/BigRow2/BoldRow3/BigRow3`), matching the existing row-1 controls.
6. **Pause / Resume session.** A `⏸` button in the recording header returns home without ending the session; a **▶ RESUME SESSION** button on home re-enters recording. Lets you check the dashboard mid-session. Fns: `pauseSession`/`resumeSession`.
7. **Third-party response flow.** With initiator(s) selected, the diamond menu offers **↳ Third-party response** → a **classification menu (Redirect / Intervene / Seek aid / Undirected-solo)** → pick recipient(s) or solo → normal behavior sheet. The record is **tagged with the type and linked to the record responded to** (`tpType` + `linkedRef`). Undirected goes straight to a solo sheet. Kept the existing diamond Intervention/Redirection alongside. New column **`TP_Type`** in the Sheet sync + CSV; record panel shows a type tag + `→ rec #N` link. Fns: `startTPResponse`/`chooseTPType`/`tpRecord`/`tpSolo`/`openTPSheet`, `pendingTP`/`sheetTpType` state, `TP_TYPE_LABELS`.

## Recording redesign batch — 2026-07-30 (verified in browser)
Evolves the 2026-07-29 flow into a single state-aware model. **Supersedes** parts of the previous batch: the `⊕ Multi` toggle, the node picker on involved-node taps, the diamond "Third-party response" submenu, and the standalone diamond Intervention/Redirection entries are all replaced by the tray + edge-tap flow below.

1. **State-aware action tray** (replaces the old hint bar). Rendered inline in `#hint-text`; the legacy static buttons are hidden. States:
   - *Empty:* "Tap animal(s) to begin a record" + `? Unknown ▾`.
   - *One initiator (fast path):* name + `+Init` / `+Recip` / `? Unknown` / `Solo` / `ℹ Info` / `✕`. **Plain node taps keep the 2-tap fast path** (tap initiator → tap recipient → sheet auto-opens). Touching a tray button latches build mode.
   - *Build mode:* `+ Init: …` / `+ Recip: …` toggle chips (with ★ on primaries) + `☆ set primary: <last-tapped>` + `? Unknown` + `Done →` + `✕`.
   - Key fns: `updateHint` (the tray), `enterBuild`, `buildDone`/`buildCancel`, `clearSelection`. First node tap always selects as initiator now (`selectParticipant`; the old `showNodePicker` branch was removed — edit a record by tapping its **edge** instead).
2. **Third-party via edge-tap.** With initiator(s) selected, tapping an **edge** (`tapDiamond`, recipient-side) starts a response to that record. `beginTPFromEdge` branches on whether the responder is already in the edge:
   - **Not a participant → assumed `intervene`**, prompts straight for targets.
   - **A participant →** type menu in the tray: **Redirect / Seek aid / Undirected(solo)** (no Intervene). `tpAwait` (picking type) → `pickTPType` → `pendingTP` (collecting targets) → `openTPSheet`.
   - Record stores `tpType` + `linkedRef`. **Redirects** also store `redirectFrom` = the referenced edge's other participant(s). Tapping an edge with **nothing** selected still opens the trimmed edit menu (Edit record · ⇄ Change members · View details).
3. **Edge color by interaction type** (not behavior domain — a record can carry agg+sub+aff at once, so `primaryDomain`/`domainColor` were removed). **direct = blue `#3b82f6`, intervene = red `#e74c3c`, seek-aid = green `#2ecc71`, redirect = yellow `#f1c40f`** (`DIRECT_EDGE_COLOR`, `TP_TYPE_COLORS`, `edgeTypeColor`/`edgeTypeMarker`). Type-colored arrowhead markers `m-direct`/`m-intervene`/`m-seekaid`/`m-redirect`.
4. **Third-party edges routed through the referenced edge's node.** For `intervene`/`seek_aid`/`redirect` (`TP_ROUTED`), the two edges connect the animals **via the referenced record's diamond** (`edgeHub[i]=recMids[linkedRef]`); the record's own diamond sits at the midpoint of hub→recipient-centroid (`diaPos`). The selected reference edge is ringed white during `tpAwait`/`pendingTP`.
5. **★ Primary designation.** In build mode, the last-tapped selected node gets a `☆ set primary` toggle; primaries show a gold ★ on the node and before names. Stored as `primaryIds` (either side). Fns: `toggleBuildPrimary`, `buildFocus`/`multiPrimary` state.
6. **Tap the sheet participants → re-select members.** The behavior-sheet participant header (`#sheet-pair`, shows a ✎) is clickable → `editSheetMembers` drops back to the network in build mode seeded from the sheet; `Back to sheet` (`returnToSheet`) writes the new membership and **preserves the in-progress behaviors** (`sel`/`selQua`/`selAffect`). Live recording only (`currentView==='recording' && !_sessionEdit`). `renderSheetPair` centralizes the header.
7. **Data columns added:** `Redirected_From`, `Primary_IDs` (sync + CSV). `TP_Type` from the prior batch retained.

**Bug fixes in this batch:** CSV export was writing no value for `TP_Type` (every later column shifted by one) — fixed + aligned. CSV export crashed on any record because `getQ('agg')` spread the per-side qualifier object — now handles per-side + flat. `editRecord` used to wipe a third-party record's `tpType`/`linkedRef`/`redirectFrom` on save — now restored. `openTPSheet` guards against an empty responder set.

## Next features (prioritized)
1. ~~Landing page after group selection~~ → **DONE 2026-06-16.** Group Dashboard view (opens from home via "Dashboard" button). Per-session notes: Group, DOS (Daily Observation Sheet), Births, per-animal Individual notes. Staged as a draft (survives reload) when no session is active; edits the live session's notes directly when one is. "Begin Session" hands off to the existing setup form. Synced/exported as session-level columns (see below).
2. ~~Preferences panel~~ → **DONE 2026-06-16, redesigned same day.** Display Preferences overlay (⚙ on home/network/recording headers), persisted to `ethograph_prefs`. Current options:
   - **Layout:** Force / By matriline / By demog / **By full ID** / **By abbr ID** (grid: numeric low→high, left→right then top→bottom; abbr uses the abbreviation rule). Clustering via `clusterInfo`; grid via a branch in `computePositions`.
   - **Color by:** Matriline / **Random** (distinct stable hue per animal) / **Sex** / **Demog** / Uniform. Sex & Demog use a chosen **preset palette scheme** (Spectrum / Warm / Cool / Earth, `COLOR_SCHEMES` + `prefs.palette`): each category gets a distinct hue and individuals within a category are **shaded** by lightness (`schemeColor`). **Optionally, a palette family can be assigned per category** via a per-category dropdown (e.g. Female → Red→Purple, Male → Green→Teal) — that category's individuals then **spread across the palette's hue range** (`PALETTES`, `categoryColor`, stored in `prefs.catPalettes` keyed `sex:f`/`demog:adult f`; "Auto" = use the scheme). Stable ordering from `recomputeDisplayMeta` → `colorMeta`. The Palette section appears only when color is sex/demog.
   - **Size by:** Demog / **Age** / Uniform. Age reads roster `age` (numeric years) or `dob`/`birthdate` (date → years) via `parseAge`; stored on `INDS`, scaled across the group's `ageMin..ageMax`; falls back to demog tier when age is missing.
   - **Node scale:** Small / Medium / Large.
   - **Abbreviated ID:** Last 2 / Last 3 / **Auto-unique** (shortest trailing digits unique within the group, min 2) — `abbrId()`.
   - **Label rows:** ordered list, one component per row (Name / Dye / Abbr ID / Full ID), reorder ↑/↓, add/remove. `prefs.labelRows`.
   - **Row style (rows 1–3):** Bold and/or Larger toggles per row (`labelBold/BigRow1..3`). Plus overall Font size S/M/L. Up to 3 rows render; a 4th configured component is a **fallback** for NA values (2026-07-29).
3. ~~Diamond tap popup redesign~~ → **DONE 2026-06-16.** Tapping a diamond opens one unified popup: **Edit record**, **Intervention** (the currently selected An1 steps in → opens the sheet linked to this record with `interv_general` pre-tagged; An2 defaults to the original aggressor, swappable), **Redirection** (the recipient/An2 redirects to a new target you tap → linked record pre-tagged `redirected` + `redirect_to`), and **View details** (read-only summary toggle). Replaces the old split tooltip/picker. Key fns: `tapDiamond`/`startIntervention`/`startRedirection`/`openRedirectionSheet`; redirection uses a `pendingRedirect` tap-capture mode.
4. Edge lines go to node edge not center
5. ~~Multiple initiators/recipients~~ → **DONE 2026-06-16.** A record can have several initiators AND several recipients. **Shared behaviors per side** (the existing `beh.*.an1`/`an2` model is unchanged), **single record with ID lists** (`an1Ids`/`an2Ids`; `an1Id`/`an2Id` kept = first, for backward-compat and the diamond flows). UX: a **"⊕ Multi"** toggle in the recording hint bar → tap to add/remove initiators → "Recipients →" → tap recipients → "Record →". Renders **hub-and-spoke** (each initiator → one diamond → each recipient). Sheet/CSV `An1_ID`/`An1_Name`/`An2_ID`/`An2_Name` become comma-joined. Editing a multi record's behaviors is via the sheet; **membership is now editable** via the diamond's **⇄ Change members** (2026-07-29, batch #3). Helpers: `a1List`/`a2List`/`centroidOf`/`sideNames`.
6. ~~Individual animal info from roster~~ → **DONE 2026-06-16.** Info card (`showAnimalInfo`) renders **all roster columns dynamically** (adapts when the sheet's columns change), resolves `dam` to a name, and shows a "this session: N records" activity line. Reachable from: tapping a node in the **Baseline Network** view, tapping an animal's name in the **Group Dashboard** list, and an **ℹ Animal info** option in the recording node picker. (Fixed a layout bug: `#info-content`/`#prefs-content` were using `.sheet-foot` which is `display:flex` — now plain `.sheet-scroll`, so both stack vertically.)
7. ~~Filter by group type in roster~~ → **DONE 2026-06-16.** The Select Group screen has a "Group type" chip row (All + types derived from the leading letters of `current_group`: CBG/HTG/JBG/SBG/STG). Selecting a type narrows the group list. `groupType(g)` + `activeGroupType` in `buildFilterUI`.
8. Matriline/rank/demog clustering options
9. Microsoft Graph / OneDrive integration (needs Azure AD app registration from OHSU IT)
11. ~~Observer profiles (local, no sign-in)~~ → **DONE 2026-06-16.** Display preferences are now stored **per observer** on the device (`ethograph_profiles = {observer: prefs}`, `ethograph_observer` = current). A **👤 observer** button on the home header opens a profile overlay to switch/add observers (initials). First observer set carries the existing setup into their profile; a brand-new observer starts from `DEFAULT_PREFS`; switching restores that observer's prefs. No backend — cross-device sync deferred (would need the blocked Azure/OneDrive auth). The session-setup Observer field defaults to the current observer. Fns: `loadPrefs`/`savePrefs`/`setObserver`/`loadProfiles`/`buildProfilePanel`.
12. ~~Select an individual-animal subset per session~~ → **DONE 2026-06-16.** The Select Group screen shows the whole group roster as a **spreadsheet-style table** (one row per animal; columns in sheet order: ✓ select + ID, Sex, Name, Dye, Birth, Age [1 decimal], Dam, Matriline, Dam In Group, Demographic, Matrank — `current_group` omitted; sticky header; scrolls both axes). **Demographic chips above the table are quick-selectors** — tapping one checks/unchecks all members of that category, and a chip shows active when all its members are checked. Then any row can be toggled to fine-tune. `selectedInds` is the single source of truth; **empty = whole group**. Fns: `buildDemogUI` (quick-select), `buildIndividualUI` (table), `applyRosterFilter`. `filterMode` + `selectedInds`; `buildIndividualUI`; `applyRosterFilter` builds `INDS` from the chosen individuals when in individual mode. Home/setup labels show "N selected".
10. ~~Generic "unknown" nodes by demographic category~~ → **DONE 2026-06-16.** A **"? Unknown"** button in the recording hint bar opens a category picker (all roster demog categories present in the group + a plain **Unknown**); selecting one adds a reusable generic node to the network (dashed outline, dimmer fill, category-only label). Generic ids are `'?'+category` (treated as unknown ids). Persist **per group** in `ethograph_generics`, restored in `applyRosterFilter`. They participate in records like any node; tapping one pops a **counter** (`openGenCounter`) to set how many of that category are involved in *this* record. Counts stored in `rec.genCounts` (in-progress: `pendingGenCounts`). Sheet/CSV tokens: `An1_ID`=`UNK:adult f×3`, `An1_Name`=`adult f (unknown)×3` (`syncPartId`/`syncPartName`). Panel/sheet show `category ×N` (`partName`/`sideNames(ids,rec)`/`sheetPartLabel`). Remove via the node's info card. Helpers: `isGeneric`/`genericLabel`/`makeGenericNode`/`addGeneric`/`removeGeneric`/`selectParticipant`.

---

## Data columns in Google Sheet — `EthographData` tab
```
Record_ID, Session, Observer, Group, Demog, DataType, Conditions, ScanNum,
Session_Start, Session_End, Session_Notes, Event, Record_Label, Timestamp, GroupID,
An1_ID, An1_Name, An2_ID, An2_Name, Solo, Duration, LinkedToRecord, TP_Type, Redirected_From, Primary_IDs,
An1_Aggression, An1_Submission, An1_Affiliation,
An2_Aggression, An2_Submission, An2_Affiliation,
An1_ThirdParty, An2_ThirdParty,
An1_Qualifiers_Agg, An2_Qualifiers_Agg,
An1_Qualifiers_Sub, An2_Qualifiers_Sub,
An1_Qualifiers_Aff, An2_Qualifiers_Aff,
Qualifiers_Interaction, Affect_An1, Affect_An2, Notes
```

## Notes columns — `EthographNotes` tab (2026-07-30)
```
Note_ID, Timestamp, Observer, Group, Scope, Subtype, Animal_ID, Animal_Name, Text, Session, Session_ID
```
Standalone subtyped notes from the Group Dashboard (`notesLog`), synced to a **separate tab** (`postNotesToSheet` → `{pw, notes}`; Apps Script upserts by `Note_ID`, auto-creating the tab). CSV export writes a second `…_notes.csv`. **Removed** the old per-record `Group_Notes` / `DOS_Notes` / `Births` / `Individual_Notes` columns.
**Note (multi-animal):** `An1_ID`, `An1_Name`, `An2_ID`, `An2_Name` may now be **comma-joined lists** when a record has multiple initiators/recipients (Feature #5). Behaviors/qualifiers/affect columns are shared per side.

**Note (unknown nodes, #10):** a generic participant appears in `An1_ID`/`An2_ID` as `UNK:<category>×N` (e.g. `UNK:adult f×3`) and in `An1_Name`/`An2_Name` as `<category> (unknown)×N`. `×N` is omitted when N=1.

**Note (third-party + primary, updated 2026-07-30):** `TP_Type` holds the third-party classification (`redirect` / `intervene` / `seek_aid` / `undirected`), blank for ordinary records. `LinkedToRecord` carries the `Record_Label` of the record being responded to. `Redirected_From` holds the ID(s) a redirection moved away from (the referenced edge's other participants). `Primary_IDs` lists participants marked ★ primary. **Add `TP_Type`, `Redirected_From`, and `Primary_IDs` header cells to the Google Sheet** so the header-keyed Apps Script writes them (CSV export includes all three with no setup).

**Note (dashboard notes redesign, 2026-07-30):** the Group Dashboard is now a **notes logger** (not per-session note fields). Group notes carry a subtype (DOS / Monitoring / Group Release / Formation / Introduction); individual notes carry a subtype (DOS / Wound / Infant / Behavior / Appearance / Social / Release / Clinical / Other) + **one or more animals** (`indNoteSel`) chosen from the **same spreadsheet table + demographic quick-select chips as the session animal picker** (`buildNoteAnimalTable`/`buildNoteDemogChips`/`buildNoteAnimalList`, mirroring `buildIndividualUI`/`buildDemogUI`). Stored as `animalIds[]` (accessor `noteAnimalIds` falls back to legacy `animalId`); `Animal_ID`/`Animal_Name` are comma-joined. Each note has an **editable date/time** (`datetime-local`, defaults to now via `nowLocalInput()`; drives `ts`/`timestamp`). Notes are logged immediately for the group and optionally tagged with the active session (`sessionId`). Stored in `notesLog` (persisted in the `LS_KEY` payload as `notes`), exported to the `EthographNotes` tab / `…_notes.csv`. Fns: `addNoteEntry`/`addGroupNote`/`addIndivNote`/`deleteNote`/`renderNoteLogs`/`buildNoteRows`. `Session_Notes` (session-level free text) stays on the data tab.

---

## Roster CSV expected columns
```
id, name, dye, sex, demog_cat, current_group, matriline, dam
```
(plus others which are ignored by INDS but kept in ROSTER). **Observed in the live sheet (2026-06-16):** also `dam_in_group` and **`matrank`** (matriline rank). So rank data *does* exist — a future option for Preferences color/size/clustering (feature #8), contrary to the earlier "no rank column" note. The animal-info card (#6) shows all of these automatically. **For "Size by: Age" (#2 redesign):** add an `age` column (numeric years) — or a `birth`/`dob`/`birthdate`/`birth_date` column (any Date-parseable format), from which age in years is computed. (Live sheet now has both `age` and `birth`.) Missing values fall back to demographic-tier sizing.

---

## Tech notes for Claude Code

### Always do after edits
Syntax-check JS after every change:
```bash
# Extract script block and check
node --check ethograph-behavior-logger.html  # won't work directly
# Instead extract <script> content to temp file and check that
```

### Sets vs Arrays
Live records use **Sets** for behavior/qualifier fields. Saved records serialize to **Arrays**. All display/summary functions must handle both via `toSet()` helper pattern:
```js
const toSet = v => v instanceof Set ? v : new Set(Array.isArray(v) ? v : []);
```

### Pan/zoom state — critical
`panZoom['evt-svg']` and `panZoom['net-svg']` objects must be **mutated in place**, never replaced:
```js
// CORRECT
const pz = panZoom[svgId]; pz.x = 0; pz.y = 0; pz.scale = 1;
// WRONG — breaks listener closures
panZoom[svgId] = {x:0, y:0, scale:1};
```

### Node positions
Computed synchronously via `computePositions()`, cached in `positions{}` object. Only reset on group change. Check `INDS.every(ind => positions[ind.id])` before re-simulating.

### Navigation state
- `recNavIdx`: `-1` = live active event, `0..n-1` = browsing saved event in recording view
- `reviewSession`: `null` = viewing active session, set = viewing past session in session screen
- Always clear `reviewSession = null` at start of `startSession()` to prevent stale state

### Record structure (live, uses Sets)
```js
{
  id, timestamp, groupId, solo, linkedRef,
  an1Id, an2Id,
  beh: {
    agg: {an1: Set, an2: Set},
    sub: {an1: Set, an2: Set},
    aff: {an1: Set, an2: Set},
    tp:  {an1: Set, an2: Set},
  },
  qualifiers: {
    agg: {an1: Set, an2: Set},
    sub: {an1: Set, an2: Set},
    aff: {an1: Set, an2: Set},
    ix: Set,
  },
  affect: {an1: Set, an2: Set},
  duration, notes
}
```

### Record structure (saved, uses Arrays)
Same as above but all Sets become Arrays.

### Key functions
- `computeLabelsFor(recs)` — label records with numbers/letters accounting for simultaneous groups
- `behSummaryFull(rec)` — returns `{dots, lines}` for display, handles Set/Array
- `primaryDomain(rec)` — returns `'agg'|'sub'|'aff'|'tp'`
- `serializeRec(r)` — converts live record (Sets) to serializable form (Arrays)
- `saveToLocalStorage()` — call after any data change
- `updateRecPanel()` — rebuilds record list below network
- `updateRecNav()` — updates prev/next bar in recording view
- `updateSessionView()` — rebuilds session screen event/record list
- `renderSVG(svgId)` — full SVG re-render, use sparingly during drag
- `applyPanZoom(svgId)` — applies CSS transform from panZoom state
- `initPanZoom(containerId, svgId)` — attaches pan/zoom/drag listeners, uses AbortController to clean up old ones

### Views
`home → roster → filter → setup → session ↔ recording (network)`

`showView(name)` activates a view and triggers side effects (layout, panel updates etc.)
