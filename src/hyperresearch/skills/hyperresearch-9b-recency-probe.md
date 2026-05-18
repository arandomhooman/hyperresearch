---
name: hyperresearch-9b-recency-probe
description: >
  Step 9b of the hyperresearch V8 pipeline (CUSTOM ADDITION). Runs after
  step 9 (evidence digest) and before step 10 (triple draft). Spawns the
  hyperresearch-recency-probe subagent to check whether developments in
  the last 90 days would invalidate or update the corpus's committed
  positions. Outputs recency-flags.md that the draft orchestrators in
  step 10 read before writing. Full tier only. Invoked via Skill tool
  from the entry skill.
---

# Step 9b — Recency probe

**Tier gate:** SKIP for `light` tier (light goes straight from step 2 to step 10). For `full` tier: run the probe.

**Goal:** catch recent developments that would invalidate the corpus before the draft is written. A draft built on 6-month-old sources may ship with claims that are already wrong; this is the last chance to qualify or pivot before pen hits paper.

**Why this runs before step 10:** corrections applied here cost a small fetch wave. Corrections applied after step 11 require a patch round and risk patching the wrong thing. Earliest possible insertion is the cheapest.

---

## Recover state

Read these inputs:
- `research/scaffold.md` — vault_tag
- `research/prompt-decomposition.json` — pipeline_tier
- `research/comparisons.md` — committed positions
- `research/temp/evidence-digest.md` — load-bearing claims
- `research/query-<vault_tag>.md` — canonical research query
- Today's date (resolve from system)

---

## Procedure

### Step 9b.1 — Spawn the recency probe

ONE Agent call:

```
Agent({
  subagent_type: "hyperresearch-recency-probe",
  description: "Recency check on committed positions",
  prompt: |
    RESEARCH QUERY (verbatim, gospel):
    > {{paste research/query-<vault_tag>.md body}}

    PIPELINE POSITION: You are step 9b of the hyperresearch V8 pipeline.
    Step 9 (evidence digest) just finished. After you return, step 10
    (triple draft) reads research/recency-flags.md as a primary input —
    draft orchestrators will hedge or pivot based on what you find.

    YOUR INPUTS:
    - vault_tag: <vault_tag>
    - comparisons_path: research/comparisons.md
    - evidence_digest_path: research/temp/evidence-digest.md
    - output_path: research/recency-flags.md
    - today: <YYYY-MM-DD>

    Budget: 5-10 URLs fetched via hyperresearch-fetcher Task delegation.
    Date discipline: only sources published in the last 90 days count.
    Prioritize INVALIDATING findings over updates over confirmations.
})
```

### Step 9b.2 — Triage the output

After the probe returns, read `research/recency-flags.md`. Look at the counts in the report-back:

- **`invalidating_count == 0`** → recency safe. Step 10 draft orchestrators will see an empty "Invalidating findings" section and proceed normally.

- **`invalidating_count > 0`** → at least one committed position needs revision before drafting:
  - If the probe explicitly recommends DROPPING a committed position, surface this to the orchestrator. The orchestrator decides whether to (a) drop the position and re-do the relevant cross-locus reconcile, (b) heavily qualify it in step 10, or (c) accept the risk and proceed.
  - If only updating findings (no full invalidations), step 10 can proceed — the draft orchestrators will use the updates to refine numbers and timelines.

### Step 9b.3 — Conditional re-reconcile

If the orchestrator decided to drop a committed position in 9b.2, re-invoke step 6 (cross-locus reconcile) to update `research/comparisons.md` before proceeding to step 10. Otherwise proceed directly.

---

## Exit criterion

- `research/recency-flags.md` exists
- The orchestrator has triaged any `invalidating_count > 0` outcome (either accepted, qualified, or re-reconciled)

---

## Next step

Return to the entry skill (`hyperresearch`). Invoke step 10:

```
Skill(skill: "hyperresearch-10-triple-draft")
```

Reminder for the draft orchestrators: step 10 must include `research/recency-flags.md` in the inputs passed to each draft sub-orchestrator.
