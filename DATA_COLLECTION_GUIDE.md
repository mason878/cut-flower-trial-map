# Cut Flower Trial Garden — Data Collection Guide

**Green Legacy · Orient, OH · Summer 2026 season**
Last updated: 2026-08-25 · App version at time of writing: **2026-08-25.1**

> **Maintenance rule:** every change to the app gets logged in §2 with the date,
> the version, and *why* it changed. The "why" matters more than the "what" —
> next season nobody will remember what problem a change was solving.

---

## 1. What the app is

| | |
|---|---|
| **Live URL** | https://mason878.github.io/cut-flower-trial-map/ |
| **Source** | GitHub repo `mason878/cut-flower-trial-map`, single file `index.html` |
| **Hosting** | GitHub Pages, serving `main` branch |
| **Backend** | Google Apps Script web app (`/macros/s/AKfycbyk…/exec`) writing to a Google Sheet; photos go to a Drive folder |
| **Scope** | 108 plots — Beds 1–6 North and 1–6 South |

Five tabs: Garden Overview, Bed Detail & Spacing, Planting Guide Table,
Log Data, Results.

The whole app is one HTML file with the CSS and JS inline. There is no build
step and no dependencies — what is in the repo is what runs.

---

## 2. Version history

### 2026-08-25.1 — three changes

**a) Photo-only submissions (new)**

Photos previously could only be attached to a harvest or planting record, so
there was no way to log a picture of a plot without also filing yield numbers.
Added a third form mode: 📷 Photos.

- Requires a plot and at least one photo. No stem fields.
- Stored with row type `photo`.
- The Results table excludes type `photo` from yield, TTH, harvest window and
  loss math, so a photo can never move a harvest figure. Verified against plot
  1: its Photos column shows the photo entry while its Harvests count stays at
  the number of real harvest rows.
- A plot with nothing but photo entries still gets a Results row, otherwise its
  pictures would be invisible.
- "Planting Record" shortened to "Planting" so three toggle buttons fit a phone.

**b) Forced update on open (new)**

Phones with the app on the home screen were holding cached copies and missing
deploys — a deploy would land and the crew would keep running the old version.

- `APP_VERSION` constant, displayed as a chip in the header (`v 2026-08-25.1`).
  **Ask for this first when someone reports a problem.**
- On open, on tab focus, and on reconnect, the app fetches its own file from the
  server with `cache: no-store`, reads the server's `APP_VERSION`, and reloads to
  `?v=<version>` if it differs.
- Will **not** reload mid-submission, mid-queue-flush, or with data in the form.
  In those cases it shows a bar: "A newer version is available. Finish or clear
  this entry, then load it."
- Never reloads twice for the same version. Throttled to once per 2 minutes.
  Silent when offline.

**c) Submission reliability (fix)**

Crews were getting "Submission failed" in the field, and blind retries were
writing duplicate rows.

Measured against the live endpoint before the fix:

| | Payload | Time |
|---|---|---|
| Submit, no photos | <1 KB | 2.3 s |
| Submit, 2 photos | 750 KB | **18.2 s** |
| Read all data (GET) | 189 KB / 976 rows | 1.8–6.3 s |

Four photos projected to 35–45 s on good wifi. On LTE in the garden, or if the
phone slept mid-upload, the request died — and the row had often already landed,
so the retry made a second copy. 16 duplicate rows were found in the sheet,
including harvests with byte-identical stem measurements.

- Photos compressed to 1200px / q0.6, down from 1600px / q0.75 — **57% smaller**
  (662 KB → 287 KB per photo on a test image; 4 photos 2.6 MB → 1.1 MB).
- Real upload deadline via `AbortController`: 30 s base + 20 s per photo, capped
  at 2 minutes.
- **On failure the app checks the sheet before retrying.** It fingerprints the
  record against the feed and only resubmits if it genuinely did not land. This
  is what stops new duplicates.
- Failed submissions are held in `localStorage` and send themselves on
  reconnect, on tab focus, and on next open. An amber banner on the Log Data tab
  shows the count with a "Retry now" button.
- If photos would exceed the ~3 MB storage ceiling, the numbers are kept and the
  photos dropped — a photo can be retaken, eight stem measurements on a plot
  already cut cannot.
- Clearer error when Apps Script returns an HTML error page instead of JSON.

---

## 3. How to deploy a change

1. Edit `index.html` in the repo (GitHub web editor is fine).
2. **Bump `APP_VERSION`.** It is the first thing in the config block. If you
   forget, phones compare identical versions and keep the old copy — the update
   check silently does nothing. This is the single easiest way to break a deploy.
3. Commit to `main`. GitHub Pages rebuilds in about a minute.
4. Verify: open the live URL and check the header chip shows the new version.
   If it shows the old one, force it with `?v=<new version>` — your own browser
   cache is the usual culprit.

Note that `raw.githubusercontent.com` serves a cached copy for a minute or two
after a commit, so a raw fetch is not a reliable way to confirm a deploy landed.
Check the commit history or the live page instead.

---

## 4. Data model

Every submission writes one row per plot. Three row types:

| Type | Written by | Counts toward |
|---|---|---|
| `harvest` | Harvest form | Harvests, first/last cut, TTH, window, avg stem, totals, losses |
| `planting` | Planting form | Plant date (latest planting row per plot wins) |
| `photo` | Photos form | Photos column only — nothing else |

Fields: `type`, `loc`, `variety`, `date`, `stems[]`, `avg`, `short`, `damaged`,
`total`, `notes`, `submitter`, `photos[]`, `submissionId`.

The Apps Script is type-agnostic — it stores whatever `type` it is handed. That
is why photo-only submissions needed no backend change. It also means a typo in
a type string would silently create a row that no calculation ever sees.

**Derived in the app, not stored:**

- **TTH** = days from plant date to first harvest
- **Harvest window** = days between first and last harvest
- **Losses %** = (short + damaged) ÷ total stems
- **Avg stem** = all individual measurements pooled across every harvest

Plant dates are pre-loaded (Wk 20 → May 21, Wk 22 → June 3) and overridden by
any Planting Record submission.

---

## 5. What we collect in the field

- **Stem length (inches)** — measure 8 random stems minimum, up to 10. The app
  averages them.
- **Short stems** — count below marketable length (12").
- **Damaged / cull stems** — unusable due to damaged blooms etc.
- **Yield** — total stems harvested for the variety, culls included.
- **Harvest date** — TTH and harvest window are calculated from this.
- **Photos** — up to 4 per submission, either attached to a harvest or on their
  own via the Photos tab.
- **Notes and collector name** — both optional, both worth filling in.

---

## 6. Troubleshooting

**"Submission failed"** — get the exact text after the colon:

| Message | Meaning |
|---|---|
| `Server returned an error page` | The Apps Script threw. Check its Executions log. |
| `Failed to fetch` / `Load failed` | Connection died mid-upload. Should now be caught and queued. |
| `Server error` | Script ran but rejected the record. Executions log. |
| Upload timed out | Exceeded the deadline. Fewer photos, or move to better signal. |

**Amber banner with a record count** — normal. Those records are safe on the
phone and will send themselves. Nothing to re-enter.

**A change was deployed but the crew doesn't see it** — check the version chip
in the header. If it is old, either `APP_VERSION` was not bumped, or that phone
predates the update checker and needs `?v=<version>` once.

**Numbers look wrong for a plot** — check for duplicate rows before assuming a
data-entry error. Duplicates from the pre-2026-08-25 retry behaviour are still
in the sheet.

---

## 7. Open items

- [ ] **Delete 3 test rows** — submitter `CLAUDE-TEST`: two harvest rows (plot 1,
      2026-08-24) and one photo probe (plot 1, 2026-08-25, notes "PROBE — DELETE
      ME"), plus 3 junk images in the Drive folder. Plot 1's Harvests and Losses
      figures are skewed until these are gone.
- [ ] **Clean up 16 duplicate rows** — use `findDuplicateRows()` in
      `apps-script-additions.gs`. It reports first and deletes nothing until you
      set `DELETE_THEM = true`. Review the list before letting it delete.
- [ ] **Optional Apps Script additions** (`apps-script-additions.gs`) — a
      `submissionId` column for server-side duplicate rejection, and a
      `photo-attach` endpoint so the record saves in ~2 s and photos upload
      separately. Then flip `PHOTO_ATTACH_ENABLED` to `true` in `index.html`.
      Written against the schema the feed returns, not the actual `Code.gs` —
      check `SHEET_NAME` and the Drive folder id first.
- [ ] **Field test** the reliability fix: submit from the far end of the garden
      on cell data with 4 photos, locking the phone mid-upload. Watch whether the
      amber banner ever appears. Frequent appearances mean the two-stage upload
      above is worth doing.

---

## 8. Where things live

| Item | Location |
|---|---|
| App source | `mason878/cut-flower-trial-map` → `index.html` |
| Apps Script | Google Apps Script project bound to the submissions Sheet |
| Submissions data | Google Sheet behind the Apps Script web app |
| Photos | Google Drive folder, links written into the `photos` column |
| Apps Script additions | `apps-script-additions.gs` (not yet applied) |
