---
name: hyperresearch-11b-quote-verify
description: >
  Step 11b of the hyperresearch V8 pipeline (CUSTOM ADDITION). Runs after step
  11 (synthesize) and before step 12 (critics). Spawns the
  hyperresearch-quote-verifier subagent to verify every verbatim quote in
  the synthesized report against evidence-digest.md (and the source notes
  directly when needed). Catches fabricated quotes, paraphrased-in-quote-marks,
  mis-attribution, and uncited compression. The critics in step 12 then see
  a draft whose quotes are already audited. Full tier only. Invoked via
  Skill tool from the entry skill.
---

# Step 11b — Quote verification

**Tier gate:** SKIP entirely for `light` tier — proceed directly to step 15 (polish). For `full` tier: run the verifier.

**Goal:** catch the highest-embarrassment failure mode (fabricated or paraphrased-in-quote-marks quotes) BEFORE the critics build findings on top of bad quotes.

**Why this runs before step 12:** if a quote is fabricated and the critics build findings around the (fake) quoted passage, the patcher patches the wrong thing. Quote verification first means critics see a draft whose quoted passages are already authentic.

---

## Recover state

Read these inputs:
- `research/scaffold.md` — vault_tag
- `research/prompt-decomposition.json` — pipeline_tier
- `research/notes/final_report_<vault_tag>.md` — synthesized draft from step 11
- `research/temp/evidence-digest.md` — canonical verbatim quotes from step 9
- `research/query-<vault_tag>.md` — canonical research query

---

## Procedure

### Step 11b.1 — Spawn the quote verifier

ONE Agent call:

```
Agent({
  subagent_type: "hyperresearch-quote-verifier",
  description: "Verify quotes in synthesized report",
  prompt: |
    RESEARCH QUERY (verbatim, gospel):
    > {{paste research/query-<vault_tag>.md body}}

    PIPELINE POSITION: You are step 11b of the hyperresearch V8 pipeline.
    Step 11 (synthesizer) produced the final report. After you return,
    step 12 (critics) reads your audit alongside the draft. Your job is
    to catch fabricated and mis-attributed quotes BEFORE the critics
    build findings around them.

    YOUR INPUTS:
    - draft_path: research/notes/final_report_<vault_tag>.md
    - evidence_digest_path: research/temp/evidence-digest.md
    - vault_tag: <vault_tag>
    - output_path: research/quote-audit.json

    Read every quoted passage in the draft. Cross-check against
    evidence-digest first; for quotes not in the digest, search the
    source note directly. Output a JSON per your agent's schema.
})
```

### Step 11b.2 — Inspect the audit summary

After the verifier returns, read `research/quote-audit.json`. Note the counts in `audit_summary`:

- `fabricated > 0` → CRITICAL. The synthesizer invented quotes. Decision tree:
  - If <3 fabrications and the draft can survive their removal: proceed to step 12 with the audit; the patcher will remove them.
  - If >=3 fabrications OR the fabrications support load-bearing claims: STOP the pipeline. Re-spawn the synthesizer with explicit anti-fabrication instructions, OR escalate to the user. Do NOT proceed to step 12 with structural fabrication unresolved.
- `paraphrased_in_quotes > 0` → CRITICAL. Quote marks are lying. The patcher must remove the quote marks (converting to paraphrase) OR replace with the verbatim source text.
- `mis_attributed > 0` → typically `major`. Patcher fixes attribution.
- `compressed > 0` → `minor` or `major`. Mostly fixed by inserting ellipses.

### Step 11b.3 — Surface fabrications to the orchestrator

If `fabricated > 0`, write a brief log entry naming each fabricated quote to `research/temp/orchestrator-notes.md`. This makes them visible if a future debugging pass needs to investigate why a synthesizer hallucinated.

---

## Exit criterion

- `research/quote-audit.json` exists and is valid JSON
- If any `fabricated` findings exist, they are either (a) <3 and proceeding to patch, or (b) the orchestrator has decided how to handle them

---

## Next step

Return to the entry skill (`hyperresearch`). Invoke step 12:

```
Skill(skill: "hyperresearch-12-critics")
```
