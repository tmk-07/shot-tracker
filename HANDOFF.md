# Shot Tracker — Project Handoff

Last updated: 2026-07-11  
Workspace: `/Users/timmy/Documents/shot tracker`  
Repository: `https://github.com/tmk-07/shot-tracker`  
Production: `https://shot-tracker.timmykim07.workers.dev/` and `shooters.tkimify.com`

## 1. Current status

The project began as a single-page make/miss counter and has been redesigned into an iPhone-first spatial shot tracker. All current work is local and uncommitted. Nothing from this redesign has been pushed or redeployed.

Current Git state:

- Branch: `main`, tracking `origin/main`
- Modified: `index.html`, `README.md`
- New: `assets/`, this `HANDOFF.md`
- Do not discard or reset these changes.
- Do not deploy to Cloudflare until the user explicitly approves the current preview.

The app remains dependency-free: one HTML file with inline CSS/JavaScript plus two local PNG assets.

## 2. Files

| File | Purpose |
| --- | --- |
| `index.html` | Entire application: UI, session state, shot classification, charts, PNG export |
| `assets/half-court-map.png` | User-supplied half-court background used by the picker and both analysis maps |
| `assets/shot-tracker-logo.png` | Transparent RGBA app logo used in the header |
| `README.md` | Public project overview |
| `wrangler.jsonc` | Cloudflare static-assets configuration (`name: shot-tracker`, assets directory `.`) |

The logo is a project-bound derivative created from `/Users/timmy/Downloads/tkimify-site/assets/shot-tracker-logo.png`. Its background was converted to transparency. The project copy is 1254×1254 RGBA and has transparent corner pixels.

## 3. Product behavior

### Live session

- The interface is optimized for iPhone use.
- Tapping the half court selects an exact shot point.
- Subsequent Make/Miss taps remain at that point until the user taps elsewhere.
- The only preset button is **Free throw**.
- Free-throw mode also remains active until the court is tapped.
- Free throws count toward overall totals and the Free Throw breakdown card.
- Free throws have no coordinates and are intentionally excluded from both court maps.
- Undo removes the latest attempt.
- End Session requires confirmation and opens analysis.
- Session state persists in `localStorage`.

### Analysis

The analysis screen contains, in this order:

1. Overall percentage hero card
2. Four percentage cards: Layup, Midrange, 3-pointer, Free throw
3. Zone Efficiency chart
4. Every Attempt chart
5. Download Summary and New Session actions

Removed by user request:

- KDE frequency chart
- Volume-bubble chart
- Personal efficiency/hex chart
- Insight sentence such as “Layup led the session…”
- Layup/Mid/3PT preset buttons
- Large “Pick your spot. Take your shot.” heading
- “spot held” feedback copy

## 4. State and data model

Storage key: `shot-tracker-v3`

Changing this key reset the earlier local preview/test data. There is no migration from `v1` or `v2`.

State shape:

```js
{
  startedAt: Number,
  endedAt?: Number,
  ended: Boolean,
  selected: null | {
    x?: Number,
    y?: Number,
    type?: "freeThrow",
    source: "court" | "freeThrow"
  },
  shots: Array<
    | { x: Number, y: Number, made: Boolean, type: "layup" | "midrange" | "three", t: Number }
    | { made: Boolean, type: "freeThrow", t: Number }
  >
}
```

Important helpers in `index.html`:

- `classify(point)`: maps a court point to layup, midrange, or three; handles free throw specially.
- `mappedShots()`: excludes free throws and malformed coordinate records from chart datasets.
- `zoneFor(shot)`: maps coordinate shots into Bucks-style efficiency regions.
- `stats(shots)`: returns makes, total, and percentage.
- `renderCourt()`, `renderScatter()`, `renderZones()`, `renderAnalysis()`.

## 5. Court coordinate system

All SVGs use `viewBox="0 0 500 470"`.

The user-supplied 640×372 court PNG is rendered with:

```html
preserveAspectRatio="xMidYMid slice"
```

It is multiplied over a light-yellow rectangle (`#fbf5df`) so its white background becomes the requested warm chart background while black court lines remain dark.

Displayed court landmarks are measured from the rendered 500×470 chart image after the app’s `xMidYMid slice` transform, using a thresholded black-line mask from `assets/half-court-map.png`:

```js
const COURT = {
  viewBox: { w: 500, h: 470 },
  outer: { left: 43.5, right: 458, top: 50.5, bottom: 420 },
  paint: { left: 201.5, right: 300, top: 50.5, ftLineY: 200 },
  hoop: { x: 250.5, y: 87 },
  archCenter: { x: 250.542, y: 108.563 },
  archRadius: 169.811,
  three: { leftX: 79, rightX: 422.5, verticalTopY: 50.5, verticalEndY: 112.5 },
  zones: { cornerZoneBottomY: 143, sideSplitY: 171.5, centerThreeFlareDeg: 10, centerThreeFlareDx: 38.007 }
};
```

The extractor thresholds pixels with luminance below roughly 60, samples known rows/columns for straight court landmarks, and fits the three-point arc from detected arc pixels. The straight three-point vertical endpoint uses a tight detector so the beginning of the curve is not misclassified as vertical line. Keep `isOutsideArc()`, `zoneFor()`, and the zone SVG generation synchronized with `COURT` when changing geometry.

### Current zone layout

There are eleven efficiency regions:

- Left/right corner three
- Left/right post/baseline midrange
- Paint (one complete rectangle; no separate rim zone)
- Left/right elbow midrange
- Center midrange/top of key
- Left/right wing three
- Center/top three

Recent user-directed geometry:

- Corner zones are about 50% taller than the original version and follow the outside of the three-point arc.
- Post and elbow side zones have nearly identical vertical spans.
- Top-of-key boundaries are straight and vertical from the free-throw line to the three-point arc.
- Center-three boundaries continue from those points but flare outward about 10°.
- The center-three region is therefore wider near half court.

Zone geometry is now generated from the shared `COURT` object in `index.html`, not independent hand-entered coordinates. `renderZones()` uses:

- `archPoint(angleDeg)`, `archAtY()`, and `archAtX()` helpers based on `COURT.archCenter` and `COURT.archRadius`
- one shared inside-three clip path and one shared outside-three clip path
- simple rectangles/polygons clipped against the measured three-point geometry
- shared `COURT` anchors for paint, side split, corner split, flare lines, and outer bounds

This keeps adjacent zones from having separately typed arc boundaries that drift apart.

## 6. Visualization rules

### Zone Efficiency

- Must remain above Every Attempt.
- Uses the supplied court image as the background.
- Green means above the session’s mapped-shot average.
- Red/pink means below average.
- Pale yellow is near average.
- Empty zones use a low-opacity warm neutral.
- Current color function: `colorScale(delta)`.
- Free throws must never affect zone percentages or the mapped-shot baseline.

### Every Attempt

- Makes are small filled green circles.
- Misses are small filled red circles, not crosses.
- Repeated shots at the exact same point are deterministically jittered using a golden-angle offset.
- Free throws are excluded.

## 7. PNG exports

Each chart has its own PNG action. SVG is serialized, drawn to Canvas, and exported as a PNG.

The summary PNG is 1080×1350 and currently contains:

- “SHOT PROFILE” heading
- Rounded light-green hero box matching the analysis screen
- Large green percentage
- Makes and attempts inside the hero box
- Zone Efficiency chart, not Every Attempt
- Date footer

Summary export code is the `#summaryBtn` click handler. It uses `CanvasRenderingContext2D.roundRect()`, so verify on the minimum target iOS version if browser support becomes a concern.

## 8. Visual design preferences

The user prefers:

- Light, mostly white UI
- Restrained pastel color accents
- Light yellow graph backgrounds
- Green for positive/good; red for negative/bad
- Subtle Make and Miss buttons, still recognizably green/red
- Light action buttons with colored text rather than dark solid buttons
- Interactive hover and keyboard-focus outlines
- Consistent styling across charts
- iPhone-first sizing and touch targets

Current action themes:

- Make: pale green
- Miss: pale red
- End Session: pale red with red text
- Download Summary: pale green with green text
- New Session: pale blue with blue text

## 9. Testing performed

Previously verified through the local in-app browser:

- Court selection and persistent location
- Free-throw mode persistence
- Switching from FT mode back to a court point
- FT attempts excluded from map dots
- Undo and end-session confirmation
- Four-card percentage breakdown
- Zone chart before attempt chart
- Filled red miss dots and deterministic jitter
- Saved ended sessions reopening into analysis
- Responsive iPhone and desktop widths without horizontal overflow

Latest checks after the logo/summary update:

- Inline JavaScript parses successfully with `new Function(...)`.
- `git diff --check` passes.
- Transparent logo is RGBA with alpha range 0–255 and transparent corner alpha 0.
- The newest browser helper rejected `file://` screenshot automation due its URL security policy, so the final logo/summary iteration still needs one human visual pass in the already-open preview.

Useful local checks:

```bash
python3 -m http.server 4173
# then open http://127.0.0.1:4173/

node -e "const fs=require('fs');const h=fs.readFileSync('index.html','utf8');new Function(h.match(/<script>([\\s\\S]*?)<\\/script>/)[1]);console.log('JS parses')"

git diff --check
```

## 10. Deployment

The user explicitly wants Cloudflare deployment only after approval.

- Do not deploy proactively.
- Before deployment, visually verify the current app and summary PNG on an iPhone-sized viewport.
- Confirm the intended production target/domain with the user.
- `wrangler.jsonc` is configured for a static asset deployment from the repository root.
- A likely command is `npx wrangler deploy`, but it has not been run for this redesign and may require Cloudflare authentication or installing Wrangler.
- After approval, commit/push and deployment should be treated as separate explicit actions unless the user authorizes both.

## 11. Known follow-ups and cautions

1. **Zone-geometry refactor complete:** The 2026-07-11 pass extracted measured court constants from `assets/half-court-map.png`, moved zone rendering/classification to shared `COURT` anchors, removed the freehand three-point arc overlay, and verified the chart in the local browser at iPhone width.
2. **Logo fidelity:** The transparent project logo was produced through an image-editing/chroma-key workflow. At its displayed 44×44 size it should be reviewed against the user’s original asset.
3. **Asset size:** `shot-tracker-logo.png` is roughly 716 KB for a 44 px display. It could be optimized later, but preserve transparency and visual fidelity.
4. **Storage migration:** `shot-tracker-v3` intentionally does not read previous storage keys.
5. **Accessibility:** The court picker is pointer-driven and does not yet have a full keyboard location-selection alternative.
6. **Session history:** Only one active/completed session is stored; there is no session archive or backend.
7. **Native widget:** A true lock-screen iPhone widget would require a separate SwiftUI/WidgetKit companion app and shared storage. It is not part of this repository.

## 12. Recommended next-agent sequence

1. Read this file, `README.md`, and all of `index.html` before editing.
2. Inspect `git diff` and preserve the dirty worktree.
3. Open the local preview at iPhone dimensions.
4. Verify the logo and summary PNG first; those are the newest, least visually tested changes.
5. If changing zones, update both `zoneFor()` and the SVG paths together.
6. Keep Free Throw records out of `mappedShots()` and both charts.
7. Run JavaScript parse and `git diff --check` after every change.
8. Stop before Cloudflare deployment and ask for approval.
