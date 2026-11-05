# Fleet health score and its explanation (Live Sessions)

## Introduction

The **Live Sessions** view (one machine, or Fleet Mode across every machine running `collector.js`) has a **Fleet Overview** panel with a **health score** from 0 to 100 and a label:

| Score | Label |
| --- | --- |
| 80 or more | Healthy |
| 55–79 | Needs attention |
| Below 55 | Action required |

The score starts at 100, and points come off for four things:

| Factor | Deduction | Type |
| --- | --- | --- |
| Developer machines not reporting | Up to 15 in total: offline ÷ total × 15 | **risk** |
| Unused AI processes still listening on a port ("orphan ports") | 5 each | **risk** |
| Sessions at 90% or more of their context window | 18 each | **info** |
| Sessions at 70–89% of their context window | 8 each | **info** |

When the score is **below 80**, a plain-language explanation is always shown under the badge. It has one overall sentence ("N items for IT to fix…" or "Nothing urgent below…"), then one line per factor saying what it means and who should act. **Risk** items come first, whatever their points. The point numbers aren't shown.

This document replaces the openspec change `openspec/changes/fleet-health-explainability/` in our format. That change fixed two problems:

1. "Context Critical" used to say sessions "will lose work". In fact Claude Code **auto-compacts** history and carries on. The summary counted "critical" at 85% while the session bar used 90%, so the two could disagree.
2. The score had no explanation at all.

Example: 1 of 4 machines is offline, and one session is at 93% context. Deductions: offline 4 (¼ × 15, rounded), auto-compacting 18, so the score is **78, "Needs attention"**. The explanation says "1 item for IT to fix". It lists **"1 developer machine not reporting"** first (ask them to restart `collector.js`), then "1 session auto-summarising history: normal behaviour, not a problem".

This document describes the panel **today**. It's written from the code, not as a plan for new work.

## What we have today

| Piece | Where | What it does |
| --- | --- | --- |
| `ctxBarHtml(pct)` | `public/app.js` | Session context bar: red at 90% or more ("Context full — Claude Code auto-compacts here, work continues"), amber at 70% or more |
| `fleetStats(fleet)` | `public/app.js` | Fleet Mode figures, using the same 90/70 thresholds |
| Single-machine stats | `public/app.js` | The same figures for the local machine |
| `healthScoreBreakdown(stats)` | `public/app.js` | The factor list `{ tier, headline, sentence, deltaPoints }`, sorted risk first, then by points |
| `renderLiveStatsPanel(stats)` | `public/app.js` | Score = 100 − sum of the factors' points, the label, the KPI tiles and the explanation |
| Styles | `public/styles.css` (`.lss-health-why*`) | Headline and sentence layout, amber risk and blue info icons |

## What it does

- Tells a non-technical viewer what's wrong with the fleet and who should fix it, without hovering.
- Keeps the score and its explanation from disagreeing, because both come from the same list.
- Makes sure expected behaviour (auto-compaction) is never described as data loss.

## Where the data comes from

| Figure | Source |
| --- | --- |
| Machines online and total | Fleet snapshots (`/api/fleet/sessions`), from `collector.js` on each machine |
| Sessions and their context % | The same snapshots, or the local session scan |
| Orphan ports | Listening ports of agent tools with no live owner process (`pid: "ORPHAN"`) |
| Active and total sessions, tokens, tracked spend | The same snapshots; spend is priced on the server |

No server change was needed: everything is computed in the browser from data the view already loads.

## How it works

1. Count the context buckets: **critical** at 90% or more, **warning** at 70–89%, over every session in the snapshot.
2. `healthScoreBreakdown` builds one item for each factor that's above zero:

| Factor | Headline | Sentence (in short) |
| --- | --- | --- |
| Offline machines | "N developer machines not reporting" | Usage isn't lost, just untracked; restart the monitoring agent |
| Orphan ports | "N unused AI processes still running" | Costs nothing by itself; IT can safely stop them |
| Context ≥ 90% | "N sessions auto-summarising history" | Normal; only look if answers start repeating or forgetting |
| Context 70–89% | "N sessions approaching that same point" | No action needed yet |

3. Sort: **risk** before **info**, then larger deductions first.
4. Score = max(0, round(100 − total deductions)). Label from the table above.
5. With a score below 80 and at least one item, show the explanation: one overall sentence, then each item's headline and sentence.
6. The KPI tile reads "Context Critical: history auto-compacted, work continues" when any session is at 90% or more. Otherwise it shows "Context Pressure", either as a count or "all sessions healthy".

## API

None. It uses the existing `/api/fleet/sessions` and Live Sessions data.

## Dashboard / UI

The **Live Sessions → Fleet Overview** panel, in both single-machine and Fleet Mode: health badge, KPI tiles (Machines, Active Sessions, Tokens In Flight, Context Critical/Pressure…), and the explanation block when the score is below 80.

## Changes by file

These are the files the feature lives in today. Nothing changes in this document.

| File | What it does |
| --- | --- |
| `public/app.js` | `ctxBarHtml`, `fleetStats`, single-machine stats, `healthScoreBreakdown`, `renderLiveStatsPanel` |
| `public/styles.css` | `.lss-health-why*` |
| `openspec/changes/fleet-health-explainability/*` | Original change, replaced by this document |

## Tests

None automated. The openspec tasks record manual checks:

- 86–89% counts as warning, not critical, and the bar agrees;
- 90% or more shows the auto-compact wording;
- the factors add up to the deduction, and the explanation shows without hovering;
- nothing is shown at 80 or above;
- both views work.

`healthScoreBreakdown` lives inside `public/app.js` and isn't exported, so it can't be unit-tested as it is.

## Known limitations

Found while reading the code. These are listed as they are; fixing them would be separate work.

1. **The score and the explanation can contradict each other.** The heaviest deduction (18 points per session) is for auto-compaction, which the explanation calls "normal behaviour, not a problem". Three long sessions take the score to 46, "Action required", while the text says "Nothing urgent below". The plan kept the old weights on purpose, but that's what produces the mixed message.
2. **Old sessions still count.** Context buckets cover every session in the snapshot (the last 48 hours), not only active ones, so a session that finished yesterday at 95% still costs 18 points.
3. **Fixed thresholds and weights** (90/70%, 18/8/5/15, 80/55) are in the browser code. They don't depend on the model's context size, and a workspace can't change them.
4. **Offline machines can cost at most 15 points**, however many there are. Half the fleet going dark, which is a real loss of visibility, weighs less than one auto-compacting session.
5. **Nothing is tested.** The logic is in the 6,500-line SPA file, with only manual checks recorded.
6. **The score has no history.** It's worked out fresh on each view, so nobody can see whether the fleet is getting healthier or worse.
