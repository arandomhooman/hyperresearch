---
name: hyperresearch-12-critics
description: >
  Step 12 of the hyperresearch V8 pipeline (CUSTOMIZED — debating team).
  Creates a critic agent-team and spawns 6 adversarial critics plus a
  verdict-synthesizer that interviews dissenting critics via SendMessage.
  Replaces the original 4-critic-parallel design with peer-to-peer
  debate among 6 critics (dialectic, depth, width, instruction, steelman,
  source-skeptic) so the patcher (step 14) receives a converged verdict
  rather than four uncoordinated JSONs. Critics never modify the draft
  directly. Invoked via Skill tool from the entry skill (full tier only).
---

# Step 12 — Adversarial critique (debating team of 6 + verdict synthesizer)

**Tier gate:** SKIP entirely for `light` tier — proceed directly to step 15 (polish). For `full` tier: spawn the team.

**Goal:** a converged, prioritized verdict the patcher can act on, produced by 6 critics that DEBATE each other instead of running blind.

**Why a team and not parallel subagents:** the original design ran 4 critics independently. They couldn't see each other's findings, so duplicates were common and contradictions never resolved. A team with SendMessage lets the verdict-synthesizer interview critics whose findings clash, converging on a smaller set of high-priority findings.

**Requires:** `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` (already set in user settings).

---

## Recover state

Read these inputs:
- `research/scaffold.md` — vault_tag
- `research/prompt-decomposition.json` — pipeline_tier, atomic items
- `research/notes/final_report_<vault_tag>.md` — synthesized draft from step 11
- `research/query-<vault_tag>.md` — canonical research query
- `research/quote-audit.json` — quote verifier output from step 11b (MUST exist)
- `research/quantitative-audit.json` — numeric-claim auditor output from step 11b (MUST exist)
- `research/counterfactual-map.json` — weak-point map from step 11b (MUST exist)
- `research/bibliography-audit.json` — citation-reality audit from step 11b (MUST exist)
- `research/round` — current round number (1 or 2; if file doesn't exist, this is round 1)

If any of the four 11b audit files is missing, halt and re-invoke step 11b first.

---

## Procedure

### Step 12.1 — Determine round

Read `research/round` (a file containing just `1` or `2`). If missing, write `1` and treat as round 1.

- **Round 1**: critique the synthesizer's draft. Standard procedure below.
- **Round 2**: critique the patched draft. Same procedure; outputs go to
  `research/critic-findings-<name>-round2.json` and
  `research/critic-verdict-round2.json`.

For brevity below, the file paths use `<round-suffix>` which is empty
for round 1 or `-round2` for round 2.

### Step 12.2 — Create the team

Call TeamCreate:

```
TeamCreate({
  team_name: "critics-<vault_tag><round-suffix>",
  description: "Debating critic team for hyperresearch step 12 (round N) — 6 critics + verdict synthesizer review draft and converge on a prioritized verdict",
  agent_type: "verdict-synthesizer"
})
```

The team file lands at `~/.claude/teams/critics-<vault_tag><round-suffix>/config.json`. The corresponding task list is at `~/.claude/tasks/critics-<vault_tag><round-suffix>/`.

### Step 12.3 — Seed the team task list

Create one task per critic role plus the verdict synthesis task. For each, write the description so the assignee can self-orient by reading it.

```
TaskCreate(subject: "Dialectic critique", description: "Read research/notes/final_report_<vault_tag>.md and research/query-<vault_tag>.md. Apply the hyperresearch-dialectic-critic procedure. Write findings to research/critic-findings-dialectic<round-suffix>.json. Mark task completed when JSON is written.")
TaskCreate(subject: "Depth critique", description: "...hyperresearch-depth-critic...research/critic-findings-depth<round-suffix>.json...")
TaskCreate(subject: "Width critique", description: "...hyperresearch-width-critic...research/critic-findings-width<round-suffix>.json...")
TaskCreate(subject: "Instruction critique", description: "...hyperresearch-instruction-critic...research/critic-findings-instruction<round-suffix>.json. Also read research/prompt-decomposition.json.")
TaskCreate(subject: "Steelman critique", description: "...hyperresearch-steelman-critic...research/critic-findings-steelman<round-suffix>.json...")
TaskCreate(subject: "Source skeptic critique", description: "...hyperresearch-source-skeptic-critic...research/critic-findings-source-skeptic<round-suffix>.json...")
TaskCreate(subject: "Verdict synthesis", description: "Wait for all six critic JSONs to land. Read them, the draft, and research/quote-audit.json. Interview critics where findings contradict via SendMessage. Write research/critic-verdict<round-suffix>.json. Mark task completed when verdict is written.")
```

### Step 12.4 — Spawn 7 teammates in parallel

ONE message, SEVEN Agent blocks. Every Agent call includes `team_name` and a unique `name`. All carry the canonical research query and the file paths.

For each critic (replace `<critic-name>` and `<task-id>` accordingly):

```
Agent({
  subagent_type: "hyperresearch-<critic-name>-critic",
  team_name: "critics-<vault_tag><round-suffix>",
  name: "<critic-name>-critic",
  description: "<critic-name> critic on the draft",
  prompt: |
    RESEARCH QUERY (verbatim, gospel):
    > {{paste research/query-<vault_tag>.md body}}

    QUERY FILE: research/query-<vault_tag>.md

    YOU ARE A TEAM MEMBER named "<critic-name>-critic" in team "critics-<vault_tag><round-suffix>".
    Other team members: dialectic-critic, depth-critic, width-critic,
    instruction-critic, steelman-critic, source-skeptic-critic, verdict-synthesizer.
    The verdict-synthesizer may SendMessage you to ask clarifying questions about
    your findings after they're written. Respond in plain text via SendMessage.

    YOUR TASK ID: <task-id> (claim it via TaskUpdate owner: "<critic-name>-critic" first).

    YOUR INPUTS:
    - draft_path: research/notes/final_report_<vault_tag>.md
    - output_path: research/critic-findings-<critic-name><round-suffix>.json
    - vault_tag: <vault_tag>
    - query_file_path: research/query-<vault_tag>.md
    - decomposition_path: research/prompt-decomposition.json   (instruction-critic only)

    Mark your task completed via TaskUpdate when your findings JSON is written.
})
```

And for the verdict synthesizer:

```
Agent({
  subagent_type: "hyperresearch-verdict-synthesizer",
  team_name: "critics-<vault_tag><round-suffix>",
  name: "verdict-synthesizer",
  description: "Verdict synthesis after critics converge",
  prompt: |
    RESEARCH QUERY (verbatim, gospel):
    > {{paste research/query-<vault_tag>.md body}}

    YOU ARE A TEAM MEMBER named "verdict-synthesizer" in team
    "critics-<vault_tag><round-suffix>". Six critics are running in parallel.
    Wait for all six critic findings JSONs to exist before reading them — go
    idle while you wait; the system will wake you when team activity continues.

    YOUR TASK ID: <verdict-task-id>.

    YOUR INPUTS:
    - draft_path: research/notes/final_report_<vault_tag>.md
    - findings_paths: [
        research/critic-findings-dialectic<round-suffix>.json,
        research/critic-findings-depth<round-suffix>.json,
        research/critic-findings-width<round-suffix>.json,
        research/critic-findings-instruction<round-suffix>.json,
        research/critic-findings-steelman<round-suffix>.json,
        research/critic-findings-source-skeptic<round-suffix>.json
      ]
    - integrity_audits: [
        research/quote-audit.json,
        research/quantitative-audit.json,
        research/counterfactual-map.json,
        research/bibliography-audit.json
      ]
    - team_name: critics-<vault_tag><round-suffix>
    - output_path: research/critic-verdict<round-suffix>.json

    Use SendMessage to interview critics where their findings contradict.
    Resolve or escalate. Mark task completed when verdict JSON is written.

    Priority ordering for the verdict: critical findings from
    quantitative-audit (unit errors, fabricated numbers), quote-audit
    (fabricated quotes), and bibliography-audit (fabricated citations)
    ALWAYS rank above critic findings — these are objective integrity
    defects, not interpretive judgments. The counterfactual-map informs
    how to weight critic findings (findings at named weak points are
    more important than findings elsewhere).
})
```

### Step 12.5 — Wait, monitor, shut down

- Teammates go idle after each turn. That is normal. Do NOT comment on idleness unless work is genuinely stuck.
- The verdict-synthesizer will SendMessage critics during debate. You will see brief idle notifications summarizing those peer DMs. Informational only — no action needed.
- When the verdict task is marked completed, all 7 outputs exist on disk.
- Send `{type: "shutdown_request"}` via SendMessage to each teammate, in order. Each will finish its current turn and shut down.
- Call TeamDelete to clean up the team directory.

### Step 12.6 — Verify outputs

Confirm all expected files exist:

```
research/critic-findings-dialectic<round-suffix>.json
research/critic-findings-depth<round-suffix>.json
research/critic-findings-width<round-suffix>.json
research/critic-findings-instruction<round-suffix>.json
research/critic-findings-steelman<round-suffix>.json
research/critic-findings-source-skeptic<round-suffix>.json
research/critic-verdict<round-suffix>.json
```

If the verdict contains `contradictions_escalated` entries, surface them to the orchestrator now — they may require a structural decision before step 14 can patch.

---

## Exit criterion

- All 7 JSON files exist for the current round
- Each is valid JSON
- The verdict's `contradictions_escalated` is either empty or has been handled by the orchestrator

---

## Next step

Return to the entry skill (`hyperresearch`). Invoke step 13:

```
Skill(skill: "hyperresearch-13-gap-fetch")
```

After step 14 (patcher) completes, the entry skill checks the round counter. If round 1, increments to round 2 and re-invokes step 12 on the patched draft. If round 2, proceeds to step 15 (polish).
