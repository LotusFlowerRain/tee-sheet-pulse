# Tee Sheet Pulse — Technical Walkthrough

Pace-of-play management for a semi-private / university golf course.
Built for CEE 250 / MS&E 265 (Feature Flow), Stanford, Spring 2026.

---

## 1. The problem, stated precisely

"Rounds are too slow" is not actionable. Turning it into something a pro shop
can act on required three separate models:

1. **A target.** What *should* a round take? Not one number — it depends on
   holes played and whether the group walks or rides.
2. **A measurement.** Where is each group actually, right now?
3. **A cost function.** A slow group's delay is not local. It propagates to
   every group behind it. Pace-of-play is a **queueing problem**, not a
   per-group problem.

The product follows from (3): the thing worth alerting on is not "this group
is slow," it is "this group is about to make five other groups slow."

---

## 2. Scope boundary — what is real and what is simulated

Stated up front, because it determines how to read everything below.

| Layer | Status |
|---|---|
| Pace model, risk scoring, cascade projection, alert logic | **Real** — the actual algorithms, unit-checked |
| Pro shop UI, all three views, check-in/check-out/dispatch flows | **Real** — complete and interactive |
| Player behaviour database (112 players) | **Real schema, synthetic rows** |
| Group positions | **Simulated** — projected from tee time × target pace |
| GPS hardware | **Does not exist yet.** Designed in Fusion 360, not manufactured |
| Backend, persistence, auth | **None.** Single static file, no network I/O |

This is deliberate. The alerting logic is the hard part and the part that
determines whether the hardware is worth building. It was designed *first*,
against a deterministic simulator, so that the position source could be
swapped in later without the logic changing. See §6.

---

## 3. Architecture

One self-contained HTML file. No build step, no dependencies, no external
requests — it runs from `file://` or any static host.

That is a deliberate constraint, not laziness: the deliverable needed to be
something a pro shop manager could open by double-clicking, and something a
reviewer could read end-to-end in one pass.

```
┌─ DATA (module-level constants) ──────────────────────────────┐
│  PACE           target minutes, by holes × walking/riding    │
│  GROUPS         today's tee sheet + status                   │
│  TRACKERS       13 GPS clips: battery, assignment, source    │
│  PLAYERS        112 players, historical pace behaviour       │
│  COURSE_HOLES   18 holes: coords, par, yardage               │
└──────────────────────────────┬───────────────────────────────┘
                               │
┌─ DERIVED (pure functions, no DOM, no side effects) ──────────┐
│  projHole(g)      where they SHOULD be    (tee time × target)│
│  actHole(g)       where they ARE          (tee time × actual)│
│  minsBehind(g)    (projHole − actHole) × target per-hole     │
│  paceStatus(g)    → on | watch | behind | done | idle        │
│                                                              │
│  trackerSource(g) → live | stale | est     ← trust tier      │
│  position(g)      → { hole, src }          ← provenance      │
│                                                              │
│  gapMin(a,b)      minutes of separation between two groups   │
│  gapThr(a,b)      alert threshold — VARIES BY CONFIDENCE     │
│  riskScore(g)     pre-round risk from group attributes       │
│  pdbRisk(p)       historical risk from a player's record     │
└──────────────────────────────┬───────────────────────────────┘
                               │
┌─ RENDER (pure read of state → innerHTML) ────────────────────┐
│  renderAll()  ─┬─ renderStats      renderTrackers            │
│                ├─ renderAlerts     renderPaceLegend          │
│                ├─ renderTeeSheet   renderUpcoming            │
│                └─ renderDlog                                 │
│  renderMap()      SVG course board + live positions          │
│  renderHistory()  player database + pre-round action queue   │
└──────────────────────────────┬───────────────────────────────┘
                               │
┌─ CLOCK ──────────────────────────────────────────────────────┐
│  setInterval 30s → advance NOW → age trackers → re-render    │
└──────────────────────────────────────────────────────────────┘
```

**Data flow is one-directional and stateless.** No component holds derived
state. Every render recomputes from the data constants, so the board cannot
drift out of sync with itself. The 30-second tick advances the clock and
re-renders; there is no diffing and no cache to invalidate.

---

## 4. The pace model

```js
const PACE = {
  '18-cart':    { total: 255, per: 14.2 },
  '18-walking': { total: 285, per: 15.8 },
  '9-cart':     { total: 120, per: 13.3 },
  '9-walking':  { total: 135, per: 15.0 },
};
```

Two projections run against every group:

- `projHole()` — hole they should be on, from elapsed time ÷ target per-hole
- `actHole()`  — hole they are on, from elapsed time ÷ their observed per-hole

`minsBehind()` is the gap between those two, expressed in minutes.

**Design decision — status is relative, not absolute.** `paceStatus()`
classifies on `minsBehind / elapsed`, not on raw minutes:

```js
const pct = minsBehind(g, now) / elapsed;
return pct >= 0.20 ? 'behind' : pct >= 0.10 ? 'watch' : 'on';
```

Rationale: 10 minutes down after 40 minutes of play is a real problem;
10 minutes down after 4 hours is noise. An absolute threshold would fire
constantly on the last few holes, where accumulated drift is normal and
nothing can be done about it anyway. A proportional threshold stays
meaningful across the whole round.

Tradeoff accepted: early in a round the denominator is small, so the metric is
twitchy for the first ~20 minutes. Mitigated by `elapsed <= 0 → idle`.

---

## 5. Position provenance — the core design decision

The hardest question in the project was not "how do we track groups," it was:

> **What do you display when you don't know where a group is?**

A pro shop board that goes blank is useless. A board that silently guesses is
worse than useless — it will get a marshal dispatched to the wrong hole, and
after that happens twice nobody trusts the board again.

Three options were considered:

| Option | Problem |
|---|---|
| Show nothing without a GPS fix | Board is empty most of the time; unusable |
| Show a projection, unlabelled | Indistinguishable from measurement; destroys trust on first bad dispatch |
| **Show a projection, labelled with its provenance** | **Chosen** |

So `position()` never returns a bare number. It returns a value *and* where
that value came from:

```js
function position(g){
  const t = trackerForGrp(g.id), src = trackerSource(g);
  if (src === 'live')  return { hole: actHole(g), src: 'live'  };  // GPS fix
  if (src === 'stale') return { hole: t.lastHole, src: 'stale',     // last known
                                staleMin: t.staleMin };
  return                      { hole: projHole(g), src: 'est'   };  // projection
}
```

Three trust tiers: **live** (current GPS fix), **stale** (tracker silent >5 min,
falling back to last known hole), **est** (no tracker — pure projection from
tee time). The tier is surfaced everywhere the position is: tee sheet, course
board, alerts, tooltips.

### The consequence worth pointing at

Provenance does not just decorate the UI — it **changes system behaviour**:

```js
// tighter threshold when both groups have live GPS
const gapThr = (a,b) =>
  (trackerSource(a) === 'live' && trackerSource(b) === 'live') ? 5 : 6;
```

The gap-closing alert fires at 5 minutes of separation when both positions are
GPS-confirmed, and 6 minutes when either is estimated. **The system acts more
decisively when it trusts its data more, and hedges when it doesn't.**

Alert copy changes on the same signal: confirmed positions read
*"Real position confirmed"*; estimated ones read *"Send marshal to verify"* —
so the marshal knows whether to trust the hole number before walking out.

Tradeoff: a one-minute threshold delta is a guess, not a calibrated number.
With real deployment data it should be fit from observed false-alarm rates.
It is currently an assumption, and marked as one.

### A related decision: golfers never report position

Position comes from course-owned hardware only. Golfers are never asked to run
an app or share location. Two reasons: you cannot make round-completion depend
on guests installing software, and it removes location privacy from the design
space entirely.

---

## 6. Why the position source is swappable

`position()` is the **only** function in the codebase that knows where a
position comes from. Everything downstream — alerts, the tee sheet, the map,
the legend, cascade projection — consumes `{ hole, src }` and never touches the
tracker table.

Replacing the simulator with real GPS is therefore a change to one function:
geofence a lat/lon fix against hole polygons, return the hole and mark it
`live`. No alerting logic changes. That seam is the reason it was safe to build
the logic before the hardware.

---

## 7. Cascade projection

The alert that matters most is not "a group is slow" but "here is what that
costs":

```
Davis Corp — 28 min behind pace at Hole 7                      [Stale]
At this pace: Johnson, Park family, Williams, Chen duo,
McKee / Russell each risk ~22 min delay.
```

Computed by finding the slowest group, taking every group teeing off after it,
and projecting the propagated delay. This converts a local symptom into the
number that justifies sending a marshal.

---

## 8. Preventive, not reactive

Two layers of prediction, both aimed at acting *before* a round goes wrong.

**Pre-round, from group attributes** — `riskScore()` scores first-timers,
rental clubs, four-player groups, late arrivals, and walking groups. Surfaced
at check-in with a suggested handover script.

**Pre-round, from history** — the player database records, per player: average
minutes/hole against target, consistency, last-five-round trend, behaviour
tags, marshal dispatches, late arrivals, and no-shows.

Risk here is deliberately **behavioural, not just pace**:

```js
pts += p.d >= 4.5 ? 5 : p.d >= 3 ? 3 : p.d >= 1.5 ? 2 : p.d >= 0.5 ? 1 : 0;
pts += p.mar >= 3 ? 2 : p.mar >= 1 ? 1 : 0;                   // marshal callouts
pts += p.late >= 2 ? 1 : 0;                                   // late arrivals
pts += p.ns   >= 1 ? 1 : 0;                                   // no-shows
pts += (p.c && p.c < 55) ? 1 : 0;                             // erratic
// >= 5 High · >= 3 Elevated · >= 1 Watch
```

Two properties fall out of this, both deliberate:

- A player at only **+1.7 min/hole can read High** if they have three marshal
  callouts and two no-shows. Pace alone would miss them — and they are exactly
  the player a marshal wants warning about.
- **Mid-range pace needs corroboration.** +3.0 to +4.4 min/hole scores 3, which
  is Elevated, not High. Being moderately slow is not on its own enough to flag
  someone as a problem; some corroborating signal has to agree. Beyond +4.5
  (roughly 80 minutes over an 18-hole round) pace stands on its own and scores
  High unaided.

The second property was **not** in the original model — see §9.

Output is the **pre-round action queue**: players booked within 7 days who
need a preventive touch, each with a specific instruction rather than a score.

---

## 9. Engineering notes

**The test suite caught a scoring flaw on its first run.** Writing assertions
for `pdbRisk()`, I asserted that a player at +3.5 min/hole would classify as
High. It returned Elevated. The test was right and my assumption was wrong:
pace contributed at most 3 points while High required 5, so **no player could
ever reach High on pace alone**, regardless of how slow they were. A golfer
averaging +5.0 min/hole with an otherwise clean record — 90 minutes over across
a round — was being classified as merely Elevated.

It had gone unnoticed because in the seeded dataset very slow players almost
always *also* had marshal callouts, so the corroborating points masked the gap.
Fixed by making the pace contribution graduated, with a top tier at +4.5
min/hole that scores 5 on its own. This is the clearest argument I have for
writing the assertions earlier: the bug was invisible in the UI and obvious the
moment the logic was stated as an expectation.


**Determinism under a repainting clock.** The board re-renders every 30
seconds. The course illustration places trees and bunkers pseudo-randomly —
with `Math.random()` the entire course visibly reshuffled twice a minute.
Fixed with a seeded generator keyed on hole index, so the drawing is identical
across every render:

```js
const rnd = (i,k) => { const x = Math.sin(i*12.9898 + k*78.233) * 43758.5453;
                       return x - Math.floor(x); };
```

**A latent wrong-data bug from stale coupling.** `buildConfirmStep()` — the
check-in confirmation step — resolved its group via `golferGroupId`, a variable
belonging to a different view. It defaulted to group 4, so the confirmation
step could describe a group other than the one being checked in. Root cause was
shared mutable view state across two surfaces. Found while removing the
golfer-facing page; the variable turned out to be unused in that function and
was deleted.

**Coordinate mapping under letterboxing.** The course SVG scales with
`preserveAspectRatio`, so the drawn box is not the element box. Naively mapping
SVG units to screen pixels by element width put hole tooltips progressively off
target at non-native aspect ratios. Corrected by deriving the actual drawn box
from the scale factor and centring offsets.

**Single source of truth.** The hole table was duplicated between `renderMap()`
and the tooltip handler — two arrays that had to be edited in lockstep.
Collapsed into one `COURSE_HOLES` constant.

---

## 10. Known gaps and what I would do next

Stated plainly, because they are the honest state of the project.

- **No tests until late.** The pace functions are pure and trivially testable;
  they should have had assertions from the start. Added as an in-page harness
  (`?selftest=1`) rather than a toolchain, to keep the single-file property.
- **No persistence.** The dispatch log does not survive a refresh.
- **No auth.** Fine for a prototype, not for a pro shop.
- **The hardware BOM needs a respec.** The design specified an **ESP32-C6**,
  which has WiFi 6, BLE, and 802.15.4 — but **no LoRa and no cellular radio**.
  WiFi does not exist in the middle of a fairway, so the current part cannot
  deliver the connectivity the product assumes. Correct options are ESP32 +
  SX1262, or STM32WL (integrated LoRa), or nRF9160 for LTE-M.
- **The BOM cost estimate was wrong.** $9.50 was an early figure that did not
  survive contact with a real parts list; GPS + MCU + e-ink + LiPo + enclosure
  is realistically **$18–30** at prototype volume. Removed from the UI.
- **Next milestone would not be hardware.** It would be phone-as-tracker:
  `navigator.geolocation` POSTing to a backend, geofenced against hole
  polygons. That exercises the genuinely hard software problem — mapping a
  lat/lon fix to a hole number, disambiguating parallel fairways with sequence
  constraints — with real satellite data and no fabrication. Hardware only
  makes sense once that works.

---

## 11. How this was built

Written with AI assistance (Claude Code), which produced the majority of the
implementation code. My own contribution was the parts that determine whether
the software is correct and worth building:

- **Domain modelling** — the pace matrix, what counts as a risk flag, the
  decision that pace-of-play is a queueing problem and cascade cost is the
  metric that matters.
- **The provenance design** (§5) — that positions must carry their source, and
  that confidence should feed back into alert thresholds.
- **Product scope** — dropping the golfer-facing page entirely to sharpen the
  product around the pro shop; removing unit pricing from an operational
  screen; deciding golfers must never self-report position.
- **Review and rejection** — auditing generated code against the model, which
  is how the stale-coupling bug, the duplicated hole table, and the
  non-deterministic repaint were found.
- **Correcting my own errors** — the ESP32-C6 radio gap and the BOM cost were
  both my mistakes, caught on review rather than by a reviewer.

The judgment calls above are the substance of the project. The code is the
easy part; deciding what the code should do, and being able to say why, is not.
