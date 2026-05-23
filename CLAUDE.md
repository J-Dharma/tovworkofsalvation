# TOV Work of Salvation — CLAUDE.md

Project brief for Claude Code sessions. Read this before touching anything.

---

## What this is

A single-file HTML web app (`index.html`) for LDS ward council meetings. The bishop and org leaders use it to track individuals and families they're ministering to, coordinate action items, and run a focused weekly meeting. It replaced a Google Sheet the council was using.

**Live URL:** hosted on GitHub Pages  
**Stack:** single self-contained HTML file (~140KB). Vanilla JS, no build step, no framework, no npm. Opens directly in a browser.

---

## File structure

Everything lives in one file: `index.html`

Internal layout (top to bottom):
1. `<style>` — all CSS, CSS variables, responsive/print rules
2. HTML — topbar, modals, board split, archive view, mobile tabs
3. `<script>` — all JS in one block

**Do not split into multiple files.** The single-file constraint is intentional — it deploys to GitHub Pages as `index.html` with zero build step and can be opened directly from Downloads.

---

## Architecture

### State

```js
let state = {
  meetingDate: 'YYYY-MM-DD',  // always current/next Sunday, auto-corrected on load
  savedAt: null,               // timestamp, used for cloud sync conflict resolution
  people: [...],               // all individuals & families (active + archived)
  meetingArchive: [...],       // past meeting snapshots saved at End Meeting
  council: {...},              // ward council member names, keyed by role id
}
```

State is persisted to `localStorage` under key `ct-v5` and synced to Google Sheets via Apps Script.

### Person object

```js
{
  id, familyName, firstNames,
  submittedBy, dateSubmitted,
  orgId,           // matches ORGS[].id
  focusId,         // matches FOCUSES[].id
  bishopricSupport, // name string (e.g. "John Smith")
  gender: 'u'|'f'|'m'|'family',
  covenantPath: [stepId],   // completed steps
  currentStep: stepId|null, // active/next step (green tile)
  status: 'active'|'archived',
  archivedAt, archiveNote,
  queuedFor: date|null,       // hand-added to upcoming meeting
  lastDiscussed: date|null,   // auto-stamped on any write during meeting
  actions: [{
    id, text, owner, open,
    createdAt, createdMeeting,
    notes: [{ id, at, author, text }],  // mid-action notes (without closing)
    closedAt, closedBy, closedNote
  }],
  history: [{
    id, at, author, text,
    note?,        // completion note for closed actions
    actionId?,    // if this entry is about a closed action
    actionNotes?, // notes that were on the action when closed
  }]
}
```

### Major JS objects

| Object | Purpose |
|--------|---------|
| `UI` | All rendering, modal open/close, user interactions |
| `App` | All data mutations (addAction, closeAction, addNote, archivePerson, endMeeting, etc.) |
| `Sync` | Google Sheets push/pull, debounced 1200ms, hash-based dirty check |
| `Admin` | Ward council member names modal |
| `MobNav` | Mobile panel switching, badge updates |

### Key functions

- `saveState()` — writes state to localStorage + schedules Sync.push()
- `personById(id)` — looks up a person in state.people
- `getQueue()` — people in the current meeting agenda (open actions OR queuedFor === meetingDate)
- `getFollowUps()` — queue filtered to those with open actions ("Follow Up" section)
- `getHandAdded()` — queue filtered to those without open actions ("Discussion" section)
- `getRoster()` — all active people
- `displayName(p)` — "Sofia Garcia" format (firstNames + familyName)
- `councilOptions()` — returns configured council members for typeahead dropdowns
- `loadCouncil()` / `saveCouncil()` — reads/writes council config; council is stored in `state.council` so it syncs

---

## Reference data

### ORGS
`bishopric`, `relief-society`, `elders-quorum`, `young-men`, `young-women`, `primary`, `sunday-school`, `ward-mission`, `temple-fh`

### FOCUSES (Work of Salvation)
`living` — Living the Gospel  
`caring` — Caring for Those in Need  
`inviting` — Inviting All to Receive the Gospel  
`uniting` — Uniting Families for Eternity

### COUNCIL_ROLES (ids)
`bishop`, `counselor1`, `counselor2`, `exsec`, `clerk`, `eq`, `rs`, `yw`, `primary`, `ss`, `mission`, `temple`, `elders`

### Covenant Path steps
Full path (Brother/Family): `baptism`, `sacrament`, `aaronic`, `melchizedek`, `endowment`, `sealing`  
Sister path: `baptism`, `sacrament`, `endowment`, `sealing`  
Three-state click: unchecked → completed (white) → current (green) → unchecked

---

## Design language

- **Navy topbar:** `#0c3a6b` with `#1473b5` accent border
- **Font:** Inter (Google Fonts)
- **Background:** `#f5f7fa`
- **Root font size:** `clamp(12px, 1.1vw, 16px)` — scales across laptop/desktop/TV
- CSS variables for all colors: `--navy`, `--accent`, `--bg`, `--surface`, `--ink`, `--ink-2`, `--ink-3`, `--ink-4`, `--line`, `--line-2`
- Avoid hard-coded `px` for layout dimensions — prefer `em`/`rem` so scaling works

---

## localStorage keys

| Key | Contents | Syncs to sheet |
|-----|----------|---------------|
| `ct-v5` | Main state (people, meetingDate, meetingArchive, council) | ✅ Yes |
| `ct-v5-sync` | Apps Script URL | ❌ Device-local |
| `ct-v5-hash` | Last sync hash | ❌ Device-local |
| `ct-v5-council` | Council names backup | Backed up from state |
| `ct-user` | Note-taker name ("Taking notes…") | ❌ Device-local |

---

## Google Sheets sync

**Protocol:** v8 (chunked PropertiesService, 4000 chars/chunk, write-verify)  
**Apps Script URL:** hardcoded as default in `Sync.init()` — new browsers auto-connect  
**Push:** `POST {action:'save', data: JSON.stringify(state)}`  
**Pull:** `GET ?action=load` → returns `{ok:true, data: '...json string...'}`  
**Conflict resolution:** `savedAt` timestamp — cloud wins if newer  
**Debounce:** 1200ms after any state change

The full Apps Script source is embedded in the sync panel (`APPS_SCRIPT_SOURCE` constant) for easy copy-paste setup.

---

## Meeting flow

1. App loads → `meetingDate` auto-corrects to current/next Sunday
2. **Follow Up section** — auto-populated from anyone with open actions
3. **Discussion section** — hand-queued by org leaders using → arrow (or + FAB on mobile)
4. During meeting: bishop taps a card → detail modal → adds notes, checks off actions, adds new actions
5. **End Meeting** → captures snapshot to `meetingArchive`, advances date to next Sunday, clears `queuedFor` for people without open actions
6. Print report opens in new tab: per-person cards (notes + open actions + completed today) + owner summary

---

## Modals

All modals use `display:flex` / `display:none` — **not** `.open` class. No scrims except the detail modal. All have `position:fixed; top:50%; left:50%; transform:translate(-50%,-50%)` and a `position:absolute; top:12px; right:12px` close button.

| Modal id | Purpose |
|----------|---------|
| `modal-detail` | Person detail (actions, notes, stewardship, covenant path) |
| `modal-add` | Add new individual |
| `modal-end` | End meeting confirmation + print |
| `modal-admin` | Ward council member names |
| `modal-sync` | Google Sheets sync config |

---

## Mobile

Breakpoint: `≤700px`

- Bottom tab bar (`.mob-tabs`): **Individuals** | **Added This Week** (with badge)
- FAB (`+`) on Individuals tab for quick adds
- Topbar shows sync status indicator (`#mob-sync-pill`) — ☁ idle / ⟳ syncing / ✓ saved / ⚠ offline
- Council and End Meeting buttons hidden on mobile (`tb-btn` hidden via media query)
- Detail modal goes full-screen; covenant path (`#d-covenant-wrap`) and discussion history (`.detail-history-section`) hidden on mobile
- `MobNav.show('roster'|'queue')` toggles panel visibility via `.mob-hidden` class

---

## Known patterns / gotchas

- **Never use `window.print()`** — always use `_printCurrentMeeting()` or `_printMeeting()` which open a clean report in a new tab
- **Syntax checking:** after any JS edit, run through acorn or the browser console — the object literal syntax is dense and missing commas between methods are a common error
- **`displayName(p)`** — always use this for rendering names, never concatenate `familyName` + `firstNames` directly
- **`lastDiscussed`** is stamped by `App.markDiscussed(pid)` on any write (note, action, action close) — stewardship field edits do NOT stamp it
- **`queuedFor`** — when the last open action on a person is closed mid-meeting, it gets stamped to `state.meetingDate` so they stay in the queue until End Meeting
- **Council config** lives in `state.council` (syncs) AND `localStorage['ct-v5-council']` (backup). `loadCouncil()` prefers `state.council`; `saveCouncil()` writes both
- **Meeting date** is always auto-corrected to `currentSundayISO()` on `UI.init()` — never stale
- **`bishopricSupport`** stores the name string (e.g. `"John Smith"`) not the role label. The dropdown shows `"John Smith — Bishop"` but the stored value is just the name. The print report strips role prefixes with `.replace(/^[^—]+—\s*/, '')`
