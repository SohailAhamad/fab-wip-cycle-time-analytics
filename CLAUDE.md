# CLAUDE.md — Fab WIP & Cycle Time Analytics (.pbip)

Standing brief for any agent session working on this project. Read fully
before editing. Rules here override ad-hoc instructions unless the user
explicitly says otherwise.

## What this project is

A 4-page Power BI portfolio report on synthetic semiconductor fab MES
data (lot movement through a 20-step flow). It demonstrates
production-grade modeling: star schema, snapshot-based WIP logic, RLS,
version-controlled TMDL. Data is synthetic but must be treated as
production MES data. Correctness bugs are worse than missing features —
the audience is domain-literate buyers.

## Project layout

- Model (TMDL): `Fab WIP & Cycle Time Analytics.SemanticModel/definition/`
  - `tables/*.tmdl`, `relationships.tmdl`, `roles/Area_Litho.tmdl`
- Report (PBIR): `Fab WIP & Cycle Time Analytics.Report/definition/`
  - `report.json`, `pages/<pageFolder>/visuals/<visualId>/visual.json`
  - Equipment page folder: `equipqualpage00003`
    (visuals: e1toolhours, e2scrapstep, e3scraprate, e5vendor)
  - Governance page folder: `modelgovpage000004`
    (visuals: g1starschema, g2wipdef, g3rls)
  - Identify other visuals by their bound fields, never by guessing names.
- Theme: `Report/StaticResources/SharedResources/FabDark.json`,
  registered in report.json as customTheme.

## Data dictionary

Source CSVs live in `./fabdata/`. Tables are already imported — edit
existing partitions; never create duplicate tables.

- `fact_lot_movement` (grain: lot-step, 40,000 rows): lot_id,
  product_id, step_seq, step_name, area, equipment_id, track_in,
  track_out (datetime), qty_in, qty_out, queue_hrs, process_hrs,
  hold_flag (0/1), hold_reason, hold_hrs, rework_flag (0/1), plus
  derived: step_total_hrs, scrap_units, track_out_date (date key).
- `dim_lot` (2,000 lots): lot_id, product_id, priority
  (Normal/Hot/Super Hot), fab_start, fab_out (datetime),
  fab_out_date (date, derived), total_cycle_days, cycle_days_bin
  (0.5-day bins), wafers_start, wafers_out, wafers_scrapped.
- `dim_product`: product_id, product_name, family
  (Memory=P100,P200; Logic=P300; Analog=P400).
- `dim_equipment` (41 tools): equipment_id, area, vendor, install_year.
- `dim_step` (20 steps): step_seq, step_name, area, target_process_hrs,
  target_queue_hrs.
- `dim_date`: Date (key), Year, Month (sorted by MonthSort), MonthNo,
  MonthSort, Week. Marked as date table.
- `SnapshotDate`: parameter table driving WIP measures via slicer;
  default 2026-04-01. Never filter the fact table by it.

## Model rules (non-negotiable)

1. Star schema only. Active relationships, all many-to-one,
   single-direction, fact → dim: dim_lot (lot_id), dim_product
   (product_id), dim_equipment (equipment_id), dim_step (step_seq),
   dim_date (track_out_date → Date).
2. Inactive relationships: dim_lot.fab_out_date → dim_date.Date
   (used via USERELATIONSHIP for fab-out trends); dim_lot.product_id →
   dim_product.product_id.
3. NO bidirectional relationships, ever. If a filter doesn't propagate,
   use the TREATAS bridge pattern (below), not cross-filtering.
4. No calculated columns where a measure works. Derived columns belong
   in Power Query, not DAX, unless binning/sorting requires otherwise.
5. All measures live in the `_Measures` table with displayFolder set.
6. Formats: counts `#,##0`; hours/days `#,##0.0`; percents `0.0%`.

## Core logic definitions

**WIP definition (implement exactly, everywhere):** a lot is in WIP as
of SnapshotDate when `fab_start <= SnapshotDate AND fab_out >
SnapshotDate`. Its current step is the row where `track_in <=
SnapshotDate <= track_out`, else the most recently completed step
(max track_out <= SnapshotDate). All lots in this dataset complete the
flow — WIP exists only relative to the snapshot; any "current WIP"
measure that ignores the snapshot is wrong by construction.

**TREATAS bridge pattern:** dim_lot receives no filter context from
dim_product (it sits on the one-side of the fact relationship). Every
measure that filters/iterates dim_lot must include:
`TREATAS ( VALUES ( dim_product[product_id] ), dim_lot[product_id] )`
as a CALCULATE / CALCULATETABLE filter argument. Currently applied in:
WIP Lots, WIP Wafers, WIP Lots (Current Step Area), Active Holds.

**Cycle-time stats** (Avg/Median/P95 Cycle Days) include completed lots
only: `fab_out <= [Snapshot Date]`.

**Rework** is meaningful only in Litho — rework denominators must be
Litho rows, not all fact rows. Card uses `Rework Rate % (Litho)`.

**Holds:** `Holds Count` is all-time; `Active Holds` is snapshot-scoped
(hold rows in-flight at snapshot, restricted to WIP lots). KPI rows
beside WIP figures must use Active Holds.

## RLS

Role `Area_Litho`: table filter `dim_equipment[area] = "Litho"`.
Propagates one-direction into the fact only. Do not "fix" the
propagation limits with bidirectionality — the limitation is
intentional and documented on the governance page.

## Presentation rules

- Theme FabDark.json: background #0F1117, visual background #1A1D26,
  border #2A2E3A, dataColors [#2DD4BF, #60A5FA, #F59E0B, #F87171,
  #A78BFA, #94A3B8], alert/negative #F87171 only, Segoe UI, visual
  titles 11pt #E5E7EB, axis labels 9pt #9CA3AF, KPI callout 28pt, no
  gridlines, rounded corners 6px, dark tooltips.
- Visual title convention: `Metric | qualifier` (e.g.
  "Tool Process Hours | top 15 tools").
- Max 6 visuals per page. Canvas 1280x720, 8px gaps, no overlaps, no
  scrollbars on bar charts — resize or Top-N instead.
- One accent color unless color encodes meaning; red = problem states
  only. No pies (one donut allowed), no gauges, no 3D.
- Charts sorted by value descending unless the axis is time or an
  ordered category (months chronological via MonthSort).

## Workflow rules

1. Power BI Desktop must be CLOSED before any file edit. Desktop
   overwrites agent edits on save.
2. `git commit` before every editing session. If Desktop rejects the
   project after edits, the user reverts (`git checkout .`) — do not
   enter a debug loop on schema-invalid JSON.
3. PBIR visual.json edits: schema validity over completeness. If a
   property's schema is uncertain, SKIP the sub-item and report it —
   never guess.
4. Never edit files under `.Report` unless the task explicitly targets
   the report layer.
5. Model changes (TMDL) are preferred over report-layer workarounds.
6. End every session by reporting: files changed, sub-items completed,
   sub-items skipped with reasons, and which validation expectations
   below the user should check after reopening Desktop.

## Validation expectations (check after every change)

At default snapshot 2026-04-01:
- WIP Lots ≈ 58 (Little's Law sanity: ~2,000 lots / 150 days × ~4.9
  days ≈ 65). Zero or 2,000 means snapshot logic broke.
- WIP Wafers ≈ 1,440; by family ≈ 48% Memory / 33% Analog / 19% Logic
  (never three identical slices — that means TREATAS is missing).
- Avg/Median/P95 cycle days ≈ 4.9 / 4.7 / 7.2.
- Active Holds: small number (single digits); Holds Count all-time
  ≈ 1,979.
- Rework Rate % (Litho) ≈ 3%; the all-rows version ≈ 0.5% is the
  wrong measure for display.
- Monthly trend: ~3 points (Jan–Mar 2026), chronological order,
  no (Blank) category.
- No scrollbars on any bar chart; all 10 areas visible in the fab
  time split.

## Context for judgment calls

This is a freelance portfolio flagship. When choosing between a clever
solution and a legible one, choose legible — the artifact must survive
scrutiny from a Power BI expert and a fab engineer simultaneously.
Anything ambiguous: stop and ask the user rather than optimizing
unprompted.
