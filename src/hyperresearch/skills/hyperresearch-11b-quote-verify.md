---
name: hyperresearch-11b-quote-verify
description: >
  Step 11b of the hyperresearch V8 pipeline (CUSTOM — expanded). Runs after
  step 11 (synthesize) and before step 12 (critics). Spawns FOUR pre-critic
  integrity agents in PARALLEL against the synthesized report: quote-verifier
  (verbatim quote check vs. evidence-digest), quantitative-auditor (numeric
  claim check vs. cited sources), counterfactual-probe (self-weak-point map),
  and bibliography-checker (sampled citation reality check). Each writes its
  own JSON audit. Critics in step 12 consume all four. The skill name is kept
  as "quote-verify" for backward-compat with prior pipeline versions, but its
  scope is now broader. Full tier only.
---

# Step 11b — Pre-critic integrity passes (4 parallel agents)

**Tier gate:** SKIP entirely for `light` tier — proceed directly to step 15 (polish). For `full` tier: run all four passes in parallel.

**Goal:** before the critics build findings against the synthesized draft, scrub the draft for objective integrity defects: fabricated quotes, wrong numbers, self-fragile thesis joints, and hallucinated citations. The critics then attack a draft that's already passed quote/number/citation reality checks and has a structured weak-point map to work from.

**Why parallel:** all four agents read the immutable final_report.md and write disjoint JSON outputs. They don't depend on each other. Running them in parallel cuts elapsed time by ~4× vs. sequential.

---

## Recover state

Read these inputs:
- `research/scaffold.md` — vault_tag
- `research/prompt-decomposition.json` — pipeline_tier
- `research/notes/final_report_<vault_tag>.md` — synthesized draft (must end in `.md`; if it doesn't, halt and flag — see Final-report invariant at bottom)
- `research/temp/evidence-digest.md` — canonical verbatim quotes from step 9
- `research/comparisons.md` — cross-locus tensions
- `research/query-<vault_tag>.md` — canonical research query

---

## Procedure

### Step 11b.1 — Verify final report is a markdown file

The pipeline invariant is: the final report lives at `research/notes/final_report_<vault_tag>.md` — a `.md` file, no other format. Before spawning the audits, confirm:

```bash
test -f "research/notes/final_report_<vault_tag>.md" && \
  echo "OK: final report is .md" || \
  { echo "FATAL: expected research/notes/final_report_<vault_tag>.md, not found"; exit 1; }
```

If the file exists but doesn't end in `.md`, OR if the synthesizer wrote a different format (HTML, JSON, etc.), HALT the pipeline and surface the error. The downstream patcher and polish-auditor assume markdown; they cannot operate on other formats.

### Step 11b.2 — Spawn all 4 audit agents in parallel

In ONE message, spawn four Agent calls. Every call MUST include `model: "opus"` (the agent definitions already specify Opus, but the orchestrator may want to be explicit).

**Agent 1: quote-verifier**
```
Agent({
  subagent_type: "hyperresearch-quote-verifier",
  description: "Verify quotes in synthesized report",
  prompt: |
    RESEARCH QUERY (verbatim, gospel):
    > {{paste research/query-<vault_tag>.md body}}

    PIPELINE POSITION: You are part of the step 11b parallel integrity
    pass. Three sibling audits (quantitative, counterfactual, bibliography)
    are running in parallel.

    YOUR INPUTS:
    - draft_path: research/notes/final_report_<vault_tag>.md
    - evidence_digest_path: research/temp/evidence-digest.md
    - vault_tag: <vault_tag>
    - output_path: research/quote-audit.json
})
```

**Agent 2: quantitative-auditor**
```
Agent({
  subagent_type: "hyperresearch-quantitative-auditor",
  description: "Audit numeric claims in synthesized report",
  prompt: |
    RESEARCH QUERY (verbatim, gospel):
    > {{paste research/query-<vault_tag>.md body}}

    PIPELINE POSITION: You are part of the step 11b parallel integrity
    pass. Three sibling audits are running in parallel.

    YOUR INPUTS:
    - draft_path: research/notes/final_report_<vault_tag>.md
    - evidence_digest_path: research/temp/evidence-digest.md
    - vault_tag: <vault_tag>
    - output_path: research/quantitative-audit.json
})
```

**Agent 3: counterfactual-probe**
```
Agent({
  subagent_type: "hyperresearch-counterfactual-probe",
  description: "Probe weak points in draft thesis",
  prompt: |
    RESEARCH QUERY (verbatim, gospel):
    > {{paste research/query-<vault_tag>.md body}}

    PIPELINE POSITION: You are part of the step 11b parallel integrity
    pass. Your weak-point map will be consumed by the critic team in
    step 12 to focus their attacks where the thesis is fragile.

    YOUR INPUTS:
    - draft_path: research/notes/final_report_<vault_tag>.md
    - comparisons_path: research/comparisons.md
    - vault_tag: <vault_tag>
    - output_path: research/counterfactual-map.json
})
```

**Agent 4: bibliography-checker**
```
Agent({
  subagent_type: "hyperresearch-bibliography-checker",
  description: "Sample-verify citations in synthesized report",
  prompt: |
    RESEARCH QUERY (verbatim, gospel):
    > {{paste research/query-<vault_tag>.md body}}

    PIPELINE POSITION: You are part of the step 11b parallel integrity
    pass. Three sibling audits are running in parallel.

    YOUR INPUTS:
    - draft_path: research/notes/final_report_<vault_tag>.md
    - vault_tag: <vault_tag>
    - output_path: research/bibliography-audit.json
})
```

### Step 11b.3 — Wait for all 4 to complete, then triage

Wait for all four audits to return. Read each output JSON. Triage:

- **quote-audit.json**: `fabricated > 0` → critical. `>=3 fabrications` or load-bearing fabrications → consider stopping. Otherwise proceed; patcher fixes.
- **quantitative-audit.json**: `fabricated > 0` or `unit_error > 0` → critical. Same triage as quote-audit.
- **counterfactual-map.json**: read `highest_risk_failure_mode` — surface to orchestrator-notes for context entering step 12.
- **bibliography-audit.json**: `fabricated_vault_note > 0`, `fabricated_url > 0`, or `misused_citation > 0` → critical. If any are present, the synthesizer hallucinated citations — a serious failure mode. Decision tree:
  - <3 critical findings and the cited claims are recoverable: proceed; patcher fixes.
  - >=3 critical findings or fabrications support load-bearing claims: STOP the pipeline. Re-spawn the synthesizer with explicit anti-fabrication instructions, OR escalate to the user.

### Step 11b.4 — Log summary

Append to `research/temp/orchestrator-notes.md`:

```markdown
## Step 11b — integrity audit summary (<timestamp>)

- quote-audit: <pass>/<total>, fabricated: <n>, paraphrased_in_quotes: <n>
- quantitative-audit: <pass>/<total>, fabricated: <n>, unit_error: <n>
- counterfactual-map: <n> weak points, highest risk: <one-liner>
- bibliography-audit: <pass>/<sampled> of <total> citations, fabricated: <n>, misused: <n>

Critics in step 12 will read all four JSON files plus this summary.
```

---

## Exit criterion

- All four JSON files exist on disk:
  - `research/quote-audit.json`
  - `research/quantitative-audit.json`
  - `research/counterfactual-map.json`
  - `research/bibliography-audit.json`
- Each is valid JSON
- Any critical findings have been triaged (either proceeding to patch, or pipeline halted)

---

## Final-report invariant (always check)

The final report MUST be a markdown file at `research/notes/final_report_<vault_tag>.md`. Every downstream step (12, 13, 14, 15, 16) assumes this. If at any point in this skill you discover the file is missing or has the wrong extension, HALT and surface the error to the orchestrator rather than continuing. Edit/Read tool operations against a non-existent or non-markdown final report will fail downstream.

---

## Next step

Return to the entry skill (`hyperresearch`). Invoke step 12:

```
Skill(skill: "hyperresearch-12-critics")
```
