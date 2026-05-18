---
name: hyperresearch-6b-checkpoint
description: >
  Step 6b of the hyperresearch V8 pipeline (CUSTOM ADDITION). Mid-pipeline
  AskUserQuestion gate between step 6 (cross-locus reconcile) and step 7
  (source tensions). Surfaces the surviving loci, the depth-investigator
  committed positions, and the cross-locus tensions so the user can
  course-correct BEFORE drafting begins. Catches cases where the initial
  plan looked right but the depth investigation drifted. Mid-pipeline
  cost insurance — by this point you've spent maybe $80 of a $300 run.
  Full tier only. Invoked via Skill tool from step 6's "Next step".
---

# Step 6b — Mid-pipeline checkpoint

**Tier gate:** SKIP for `light` tier (light skips steps 3–9 entirely). For `full` tier: run the gate.

**Goal:** give the user a second course-correction opportunity halfway through the pipeline, after the depth investigations have committed positions and the cross-locus tensions are visible. This is structurally identical to step 1b but mid-pipeline: catches drift that started after the plan was approved.

**Why this step exists:** the plan-review gate (1b) catches "wrong question" errors. This gate catches "right question, wrong loci picked / wrong positions taken" errors. By step 6, you can see whether the depth investigators converged on what you actually wanted, or whether they drifted into tangents.

---

## Recover state

Read these inputs:
- `research/scaffold.md` — vault_tag, modality
- `research/prompt-decomposition.json` — original atomic items + tier
- `research/loci.json` — the loci that survived dedup
- `research/comparisons.md` — cross-locus tensions and the depth investigators' committed positions
- `research/temp/contradiction-graph.json` — if it exists, the corpus's evidence fights
- `research/query-<vault_tag>.md` — canonical research query

---

## Procedure

### Step 6b.1 — Compose the checkpoint summary

Build a single concise message that surfaces what's happened so far. Format:

```markdown
## Mid-pipeline checkpoint — about to draft

**Query:** <one-line summary of the canonical query>

**Spent so far (approx.):** ~$50–100 on breadth sweep + depth investigation
**Yet to spend:** ~$150–300 on drafting + verification + critics + patches

**Loci that survived dedup** (<N> total):
1. <locus name> — <one_line> — flavor: <convergent | dialectical>
2. <locus name> — ...
...

**Depth-investigator committed positions** (one per locus):
1. <locus name>: "<committed_position one-liner>" — confidence: <high/med/low>
2. ...

**Cross-locus tensions** (top 3-5 from comparisons.md):
- <tension name>: <one-line description of the disagreement>
- ...

**Key contradictions in the corpus** (if any):
- <contradiction>: side A vs side B

**Sanity check questions for the user:**
- Are these the right loci, or did the analysts pick tangents?
- Do the committed positions align with what you actually wanted to know?
- Are there missing angles the depth investigation should have covered?
```

Keep the summary tight — under 80 lines. Surface the substance, not the full files.

### Step 6b.2 — Ask the user via AskUserQuestion

```
AskUserQuestion({
  questions: [{
    question: "Proceed to drafting with these loci, positions, and tensions?",
    header: "Mid-pipeline check",
    multiSelect: false,
    options: [
      {
        label: "Proceed to drafting",
        description: "Loci look right, positions are reasonable, tensions are well-framed. Run the drafting + verification + critic phases as planned."
      },
      {
        label: "Adjust positions or loci",
        description: "Tell me what to revise — drop a weak locus, push back on a committed position, add a missing angle. I'll incorporate via additional depth investigation or by editing the artifacts."
      },
      {
        label: "Cancel",
        description: "Stop the pipeline. The vault keeps everything fetched so far — useful as a research substrate. No drafting happens."
      }
    ]
  }]
})
```

### Step 6b.3 — Handle the user's choice

**If "Proceed to drafting":** continue to Next step (invoke step 7).

**If "Adjust positions or loci":** the user provides notes. Apply changes:
- **Drop a locus**: edit `research/loci.json` to remove the locus. Note in
  `research/temp/orchestrator-notes.md` that this locus was deprecated by user
  decision, so the patcher/critics won't expect it later.
- **Push back on a position**: edit `research/comparisons.md` to qualify or
  reverse the committed position. The synthesizer in step 11 will see your
  edit and respect it.
- **Add a missing angle**: spawn a fresh `hyperresearch-depth-investigator`
  subagent on a new ad-hoc locus the user specified. Wait for it to complete,
  then re-run step 6 (cross-locus reconcile) before re-invoking this skill.

After applying changes, re-invoke step 6b to show the user the updated state
and confirm. Iterate until terminal choice.

**If "Cancel":** write `research/temp/cancelled-at-6b.md` with the timestamp
and the user's reason (if any). Do NOT invoke step 7. Report: "Pipeline
cancelled at mid-checkpoint. Vault contains all fetched sources from steps
1-6 — usable as research substrate via `hyperresearch search ...`. Restart
with `/hyperresearch` if you want to retry."

---

## Rules

- **This is a gate, not a step that does work.** No subagents are spawned
  here except potentially one ad-hoc depth-investigator in the "adjust"
  path.
- **Honor the user's choice.** "Cancel" means cancel. Don't re-ask.
- **Sanity-check questions are mandatory in the summary.** They prime the
  user to engage critically rather than rubber-stamping.
- **The vault is durable across cancellation.** Always tell the user that
  cancellation preserves the fetched corpus — they can re-research later
  without re-fetching.

---

## Exit criterion

- The user has chosen "Proceed to drafting" OR has chosen "Cancel" (terminal state).
- If "Adjust", iteration continues until the user picks a terminal choice.

---

## Next step

If the user chose "Proceed to drafting":

Return to the entry skill (`hyperresearch`). Invoke step 7:

```
Skill(skill: "hyperresearch-7-source-tensions")
```

If the user chose "Cancel": terminate. Do not invoke step 7.
