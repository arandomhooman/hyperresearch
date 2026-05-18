---
name: hyperresearch-14-patcher
description: >
  Step 14 of the hyperresearch V8 pipeline. Spawns the hyperresearch-patcher
  subagent (TOOL-LOCKED to Read + Edit) to apply critic findings as
  surgical Edit hunks against the synthesized final report. Zero
  regeneration. Pre-stubs the patch log because Edit cannot create files.
  Handles orchestrator-escalated structural restructures inline. Invoked
  via Skill tool from the entry skill (full tier).
---

# Step 14 — Patch pass

**Tier gate:** SKIP entirely for `light` tier (no critics = no findings to patch). For `full`: run as documented.

**Goal:** apply critic findings to the draft as surgical Edit hunks. Zero regeneration.

---

## Recover state

Read these inputs:
- `research/scaffold.md` — vault_tag
- `research/round` — current round (1 or 2; default 1 if missing). Compute `<round-suffix>`: empty for round 1, `-round2` for round 2.
- `research/notes/final_report_<vault_tag>.md` — the synthesized final report from step 11 (round 1) OR the round-1-patched draft (round 2)
- All `research/critic-findings-<name><round-suffix>.json` files for the current round (6 critic types: dialectic, depth, width, instruction, steelman, source-skeptic)
- `research/critic-verdict<round-suffix>.json` — prioritized verdict from the verdict-synthesizer
- `research/quote-audit.json` — quote verifier output from step 11b
- `research/quantitative-audit.json` — numeric-claim audit from step 11b
- `research/counterfactual-map.json` — weak-point map from step 11b
- `research/bibliography-audit.json` — citation-reality audit from step 11b
- `research/temp/evidence-digest.md` — patcher's primary citation source
- `research/query-<vault_tag>.md` — canonical research query

---

## Step 14.0 — Skip gate (optional)

Before spawning the patcher, check whether `research/skip-patcher.txt` exists. If it does, the invoker has requested that step 14 be bypassed. In that case, skip to "Skip path" below.

**Skip path:** Record a minimal log:

```bash
python -c "
import json, pathlib
files = ['research/critic-findings-dialectic.json','research/critic-findings-depth.json','research/critic-findings-width.json','research/critic-findings-instruction.json']
total = sum(len(json.loads(pathlib.Path(f).read_text()).get('findings',[])) for f in files if pathlib.Path(f).exists())
pathlib.Path('research/patch-log.json').write_text(json.dumps({'total_findings': total, 'applied': [], 'skipped': [{'reason': 'patcher-skipped-by-invoker'}], 'conflicts': [], 'orchestrator_escalated': []}))
"
```

Then proceed to step 15. Most runs should not use this gate.

---

## Step 14.1 — Pre-create the patch log stub

The patcher is tool-locked to `[Read, Edit]` — it cannot Write. Edit can only modify files that already exist. So you (the orchestrator) MUST write the canonical stub first, which the patcher will then Edit to populate. Round-suffix the log so round 2 doesn't overwrite round 1:

```bash
echo '{"total_findings": 0, "applied": [], "skipped": [], "conflicts": [], "orchestrator_escalated": []}' > research/patch-log<round-suffix>.json
```

The schema above is canonical. The patcher's only job on this file is to Edit the existing keys — `total_findings` becomes an integer, the four arrays get populated. **The patcher MUST NOT invent alternate schemas** — downstream tooling assumes the canonical shape.

If you skip this step the patcher will silently have nowhere to write its log, will inline the log in its response instead, and you may mis-capture or drop the data entirely.

---

## Step 14.2 — Spawn the patcher

Spawn ONCE.

**Spawn template:**
```
subagent_type: hyperresearch-patcher
prompt: |
  RESEARCH QUERY (verbatim, gospel):
  > {{paste research/query-<vault_tag>.md body}}

  QUERY FILE: research/query-<vault_tag>.md

  PIPELINE POSITION: You are step 14 (patcher) of the hyperresearch V8
  pipeline. Step 12 produced critic findings; step 13 filled vault gaps.
  After you return, step 15 (polish auditor) does the final hygiene pass.
  You are TOOL-LOCKED to [Read, Edit] — you cannot Write.

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
  - verdict_path: research/critic-verdict<round-suffix>.json
  - integrity_audits:
      quote_audit_path: research/quote-audit.json
      quantitative_audit_path: research/quantitative-audit.json
      counterfactual_map_path: research/counterfactual-map.json
      bibliography_audit_path: research/bibliography-audit.json
  - patch_log_path: research/patch-log<round-suffix>.json   (already stubbed)
  - evidence_digest_path: research/temp/evidence-digest.md
  - round: <1 or 2>

  CRITICAL: read research/critic-verdict<round-suffix>.json FIRST. Its
  `priority_queue` is the canonical ordering — apply findings in that
  order, not the order they appear in the individual finding files. The
  verdict synthesizer has already clustered duplicates, resolved
  contradictions, and ranked by severity. Trust it.

  CRITICAL: the four 11b integrity audits identify objective defects
  (fabricated text, wrong numbers, hallucinated citations, fragile
  thesis joints). Any `critical` finding from these MUST be patched
  before applying critic-finding patches. Order within criticals:
  1. quantitative-audit fabricated/unit_error (wrong numbers ship as
     misinformation)
  2. bibliography-audit fabricated_vault_note/fabricated_url/misused_citation
     (hallucinated sources destroy trust)
  3. quote-audit fabricated/paraphrased_in_quotes (lying quote marks)
  4. critic-verdict critical findings (interpretive defects)

  CRITICAL: final report extension is `.md`. Do NOT rename, do NOT
  convert to another format. The patcher operates by Edit on the
  existing markdown file.
```

The patcher's job:
- Apply each finding's patch as an Edit on the draft file
- Populate the pre-stubbed patch log via Edit on `research/patch-log.json`
- Each Edit hunk stays surgical: change as little as possible while addressing the issue
- Reject findings that don't serve the research_query (the patcher checks every finding against the canonical query)
- Escalate findings that require structural restructure (rather than applying them as oversized patches)

---

## Step 14.3 — Read the patch log

Check the patch log when the patcher returns (round-suffix-aware):

- **Did the patcher apply all `critical` findings?** If any critical was SKIPPED, that's a pipeline blocker — resolve it yourself before step 15 (or before round 2). Options:
  - (a) reject the finding as invalid after re-reading the draft
  - (b) escalate to the user
  - (c) hand-craft an Edit to address it (you have Write/Edit access; the lock applies only to the patcher subagent)

- **Did any findings CONFLICT?** Look at the conflict log. If two critics disagreed and the patcher picked one, consider whether the discarded one was actually more important. Note that the verdict-synthesizer should have pre-resolved most contradictions; any remaining conflict in the patch log likely indicates the verdict-synthesizer escalated rather than resolved.

- **Did the patcher log a "patch too large" skip?** That means a critic proposed regeneration in patch clothing. If the finding was critical, re-spawn the critic with a tighter suggestion, or address it yourself with multiple small hunks.

- **Is the patch log still the empty stub?** If yes, the patcher failed to log — its Task result will contain the real log inline. Read the Task result, parse out the JSON, and write it to `research/patch-log<round-suffix>.json` yourself via Bash so downstream lint rules see it.

---

## Step 14.4 — Handle orchestrator-escalated findings (structural restructures)

The patcher populates `orchestrator_escalated` with findings where `requires_orchestrator_restructure: true` — most commonly, structural-mirror-check findings from the instruction critic (wrong H2 order / missing required heading / extra H2). The patcher's tool-lock cannot safely move / rename H2 sections, so YOU handle them here, before step 15:

For each entry:
1. Read the `issue` field to understand which H2 in the draft needs to move, be added, or be renamed.
2. Apply the restructure via hand-written Edit calls on `research/notes/final_report_<vault_tag>.md`. You have Write and Edit access — the tool lock applies only to the patcher and polish auditor subagents.
3. Preserve the body content within each H2 section — you are moving / renaming / inserting headings, not regenerating prose. If a new heading is added and its body needs fresh content, write a short evidence-grounded paragraph for it.
4. Log changes in `research/orchestrator-restructure-log.md` (plain markdown, one bullet per change) so downstream lint rules can see this step happened.
5. Never regenerate a whole section or the whole draft. The "patch not regenerate" invariant still binds you — broader tools but not broader license.

---

## Constraints

- **Do not apply revisions yourself in step 14.2.** You MUST spawn the patcher subagent. Do NOT call Edit directly on `research/notes/final_report_<vault_tag>.md` — the patcher has the tool-lock invariants (surgical-edit discipline, conflict resolution, integrate-don't-caveat rule) baked into its prompt. Bypassing it defeats the entire adversarial-review architecture. If the patcher returns empty, re-spawn it once — don't fall back to doing the work yourself unless step 14.4 escalations require it.

- **Do not re-spawn the patcher on the same findings** unless you've modified the findings. The patcher's second run on identical input is a waste.

---

## Exit criterion

- `research/patch-log<round-suffix>.json` exists with `total_findings` set and at least one of `applied` / `skipped` / `conflicts` populated
- All critical findings either applied or resolved by orchestrator
- All `orchestrator_escalated` findings handled (with `research/orchestrator-restructure-log<round-suffix>.md` if any structural restructures were applied)
- `research/notes/final_report_<vault_tag>.md` has been edited (or no edits needed if findings were trivial)

---

## Step 14.5 — Round routing

Read the round counter at `research/round` (default: `1`).

**If round == 1:**

1. Write `2` to `research/round` (overwrites the `1`).
2. Log a brief round-1 summary to `research/temp/round1-summary.md` — what the major patches changed, what's expected to differ in the round-2 critique.
3. Return to the entry skill and invoke step 12 (round 2):

   ```
   Skill(skill: "hyperresearch-12-critics")
   ```

   Step 12 will detect `research/round == 2`, suffix its output filenames with `-round2`, and run the full debating-critic team on the round-1-patched draft. Steps 13 and 14 follow, also round-2-suffixed. After round 2's patch completes, this step routes to step 15.

**If round == 2:**

1. Confirm round 2 patches are clean.
2. Log a final two-round summary to `research/temp/two-round-summary.md` — concise diff of what round 1 caught vs. what round 2 caught (the second-round catches are typically defects introduced by round-1 patches, or weaknesses the original draft was too rough to expose).
3. Reset the round counter for next pipeline run: write `1` to `research/round` (or delete it).
4. Return to the entry skill and invoke step 15:

   ```
   Skill(skill: "hyperresearch-15-polish")
   ```

---

## Next step

See Step 14.5 above. The next-step routing depends on the current round.
