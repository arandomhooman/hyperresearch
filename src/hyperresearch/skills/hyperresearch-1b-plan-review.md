---
name: hyperresearch-1b-plan-review
description: >
  Step 1b of the hyperresearch V8 pipeline (CUSTOM ADDITION). Surfaces the
  step-1 decomposition (atomic items, tier, required section headings,
  coverage matrix, expected cost/time) to the user via AskUserQuestion
  before step 2 burns the breadth sweep. Catches misclassified queries and
  wrong-direction decompositions cheaply (~$1 to redo step 1) rather than
  expensively (~$200+ deep into the pipeline). Runs for ALL tiers. Invoked
  via Skill tool from the entry skill.
---

# Step 1b — Plan review gate

**Tier gate:** Runs for ALL tiers — step 1's tier classification is itself one of the things the user is reviewing.

**Goal:** give the user a chance to course-correct the pipeline plan BEFORE step 2 starts crawling the web. Step 2 is the first step that spends real money (parallel fetcher waves). Catching a misclassified tier, wrong atomic items, or missing required section here costs nothing; catching them at step 11 costs a full pipeline run.

**Why this step exists:** the single highest-cost failure mode in the pipeline is "we researched the wrong question." The decomposition is where that branch happens. A 30-second human review of the decomposition prevents the multi-hour failure case.

---

## Recover state

Read these inputs:
- `research/scaffold.md` — vault_tag, modality, wrapper requirements
- `research/prompt-decomposition.json` — atomic items, sub_questions, entities, required_section_headings, response_format, citation_style, pipeline_tier
- `research/temp/coverage-matrix.md` — verbatim query phrases mapped to atomic items
- `research/query-<vault_tag>.md` — canonical research query (GOSPEL — verify it matches what the user actually typed)

---

## Procedure

### Step 1b.1 — Compose the plan summary for the user

Build a single concise message that surfaces the decomposition. Format:

```markdown
## Plan review — about to start research

**Query:** <one-line summary of the canonical query, OR "{first 200 chars}..." if it's long>

**Tier:** <light | full>
- Estimated cost: <$5–15 for light, $200–400 for full with customizations>
- Estimated time: <~30–40 min for light, ~3–5 hours for full>

**Response format:** <short | structured | argumentative>
**Citation style:** <wikilink | inline | none>

**Required section headings** (the draft will be structured this way):
- <heading 1>
- <heading 2>
- ...

**Atomic items the pipeline will cover** (<N> total):
1. <atomic item 1 — sub-question or entity, short form>
2. <atomic item 2>
...

**Coverage gaps the search plan will need to fill:**
<read research/temp/coverage-matrix.md and list 3-5 most important coverage rows>
```

Keep the summary tight — under 50 lines. The user is reviewing, not reading a full report. The full decomposition is on disk if they want to inspect it.

### Step 1b.2 — Ask the user via AskUserQuestion

Use the AskUserQuestion tool with this exact shape:

```
AskUserQuestion({
  questions: [{
    question: "Proceed with this research plan?",
    header: "Plan review",
    multiSelect: false,
    options: [
      {
        label: "Proceed as planned",
        description: "Run the full pipeline as decomposed. Best if the atomic items, tier, and required headings look right."
      },
      {
        label: "Adjust the plan",
        description: "Tell me what to change — wrong tier, missing atomic item, wrong required heading, scope too broad/narrow, etc. I'll edit the decomposition and re-confirm."
      },
      {
        label: "Cancel",
        description: "Stop the pipeline. No fetching happens. Nothing is spent past step 1."
      }
    ]
  }]
})
```

### Step 1b.3 — Handle the user's choice

**If "Proceed as planned":** continue to Next step (invoke step 2).

**If "Adjust the plan":** the user provides notes in free-form text (via the "Other" / notes channel of AskUserQuestion). Read their notes and apply the requested changes:

- **Tier change** (light ↔ full): edit `pipeline_tier` in `research/prompt-decomposition.json`. If they bumped to full, also re-derive any tier-gated downstream behavior.
- **Atomic item add/remove**: edit the `atomic_items` array in `prompt-decomposition.json`.
- **Required heading change**: edit `required_section_headings` in `prompt-decomposition.json`.
- **Scope adjustments**: edit the relevant fields and add a "scope_notes" entry to the scaffold so downstream steps see the user's intent.

After applying changes, **re-invoke step 1b** (this skill) to show the user the updated plan and confirm. Iterate until the user picks "Proceed as planned" or "Cancel".

**If "Cancel":** write a brief log to `research/temp/cancelled.md` noting the cancellation timestamp and the user's reason (if provided). Do NOT invoke step 2. Do NOT spend further. Report to the user: "Pipeline cancelled. Nothing fetched. The decomposition and scaffold remain at `research/` if you want to inspect them or restart with `/hyperresearch`."

---

## Rules

- **This is a gate, not a step that does work.** No subagents are spawned here. The orchestrator reads, summarizes, asks, applies changes, repeats.
- **Honor the user's choice.** "Cancel" means cancel. Do not auto-rerun the pipeline. Do not interpret cancel as "ask again with different framing."
- **Keep the summary scannable.** Long bulleted lists buried in prose defeat the purpose. The whole summary should fit on one screen.
- **Tier classification is the single most expensive-to-be-wrong field.** If light vs full is misclassified, the cost gap is ~50×. When summarizing, lead with the tier and its cost/time so the user can catch this before anything else.

---

## Exit criterion

- The user has chosen "Proceed as planned" OR has chosen "Cancel" (terminal state)
- If "Adjust", iteration continues — the exit criterion is only met when the user picks one of the two terminal choices

---

## Next step

If the user chose "Proceed as planned":

Return to the entry skill (`hyperresearch`). Invoke step 2:

```
Skill(skill: "hyperresearch-2-width-sweep")
```

If the user chose "Cancel": terminate. Do not invoke step 2.
