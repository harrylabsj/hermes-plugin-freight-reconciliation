---
name: freight-reconcile-and-explain
version: 0.3.0
author: Haina
description: Start deterministic reconciliation runs, poll jobs and explain evidence-backed differences. Use when running the re-calculation or answering "why is this amount different".
---

# Runs and difference explanation

## Prerequisites
Mappings and rates are CONFIRMED on the admin page; required manual trip
associations are confirmed. Running unconfirmed → the rules layer returns
RULE_UNCONFIRMED / RATE_NOT_FOUND; relay that honestly.

## Flow

1. **Start a run**: `freight_start_run(case_id)` → returns job_id immediately
   (import and re-calculation are long tasks; synchronous waiting is not allowed
   under the 30-second response constraint, §18.2).
2. **Poll**: `freight_get_job(job_id)`; do not start new runs while
   QUEUED/RUNNING. Terminal states: SUCCEEDED / PARTIAL / FAILED. PARTIAL means
   a bounded scope completed with failures remaining — present it as such.
   On FAILED, relay error.code / message / recovery_action verbatim. A job left
   QUEUED/RUNNING after a connector restart is marked FAILED/RUN_INTERRUPTED on
   next startup — follow its recovery_action and simply start the run again;
   inputs are frozen by digest so the re-run is deterministic.
3. **Summary**: `freight_get_summary(run_id)`:
   - net billed = comparable + non-comparable + explicitly excluded (conservation);
   - positive / negative / net difference; amount coverage
     (denominator = absolute value of all bill lines);
   - when amount_complete=false you must warn "cannot formally freeze".
4. **Difference list**: `freight_list_issues(run_id, code?, cursor?, limit?)`
   paged; when the cursor is invalidated by changed query parameters, re-query —
   never present a truncated page as complete.
5. **Explain**: `freight_get_evidence(run_id, group_id)` — four panels: original
   bill location, trip/evidence, rule and formula trace, current decision. State
   precisely "which trip, which rule version, which proofs" each number rests on.
6. **Pending association**: `freight_list_match_candidates(case_id)` lists
   AMBIGUOUS/UNMATCHED lines; P3 candidates are suggestions only — show the
   candidates and their basis; after the user decides,
   `freight_prepare_match(case_id, bill_line_id, trip_id)` creates the
   confirmation candidate (confirmed on the admin page). Never substitute
   "highest score" for human confirmation.

## Boundaries
- Do not rewrite amount formulas; never stack per-group price differences with
  suspected-duplicate amounts (a group's difference counts once).
- NOT_BILLED_IN_PROVIDED_SCOPE is a reminder, not savings; unproven amounts are
  listed separately, never mixed into differences.
- Present with run_id, version, coverage denominator and open-item counts.
