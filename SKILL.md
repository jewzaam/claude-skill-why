---
name: why
description: Deep-dive 5-Whys investigation into the current problem or a specified topic. Triggers when the user explicitly invokes `/why` (alone or with a refining prompt). Use this whenever the user types `/why` to drill from a surface symptom or open question down to a root cause, coordinating batched AskUserQuestion prompts for operator-side context with parallel investigations of global standards, local project docs, and codebase evidence. Produces a `Why-<slug>.md` artifact in the project root capturing the chain, evidence, root cause, and recommended next steps.
disable-model-invocation: true
---

# Why

A structured "5 Whys" investigation. Drills from a surface question down to root cause by alternating **operator-side context gathering** (what the human already knows, has tried, expects) with **evidence gathering** from three angles (global standards, local docs, code). Produces a durable artifact so the reasoning survives the session.

## When to invoke

Only when the user explicitly types `/why` — alone (use the current problem from the conversation) or with a refining prompt after it (`/why the build flake on Windows`). Do not auto-trigger on stray uses of the word "why" in a question.

## Core idea

Surface questions are usually shallow restatements of the symptom. Real understanding comes from asking "why?" repeatedly — typical chains are 3–5 levels, hard cap 7 — treating each prior answer as the new question. At each level, ground the answer in evidence — not speculation — by dispatching specialised investigators in parallel:

- `why-standards` — what the user's global coding/process standards say about this
- `why-local-docs` — what this project's docs, CLAUDE.md, README, ADRs say
- `why-code` — what the code itself reveals (definitions, callers, history, recent changes)

...and combining their findings with **operator-side context** (what the user knows, has already ruled out, or believes) gathered via batched `AskUserQuestion` calls. Evidence + operator context → an answer for this why-level → restated as the next why.

The user's global CLAUDE.md mandates that all clarifications go through `AskUserQuestion`. Honor that — never ask the user questions in plain chat during a `/why` run.

## Permissions and tool discipline

Permission prompts annoy the user. At the **start of the `/why` run, fetch the allowlist once** by calling `mcp__allowlist__get_allowed_permissions`, and reuse that result for the whole run — don't re-fetch per command or per dispatch. Then:

1. **Prefer dedicated tools** — `Read`, `Grep`, `Glob`, `Agent` — over `Bash`. They don't trigger prompts and they're faster.
2. **Run only allowlisted Bash.** If a needed command isn't allowlisted, do not just run it and absorb the prompt — find an allowlisted equivalent (e.g., `git log -5` instead of `git log | head -5`; flags instead of pipes through non-allowlisted utilities).
3. **Never chain Bash with `;`.** Use `&&` or separate calls. Semicolons defeat hook inspection.
4. **No ad-hoc investigation scripts.** Do not invent inline `python -c "..."` or one-shot scripts to slice data. They are not allowlisted, they prompt, and they pile up. Per the user's standards, exploratory code goes in durable scripts under `scripts/explore/` — and **only after asking the user** via `AskUserQuestion` whether that's worth doing for this investigation. Treat ad-hoc scripts as a last resort that requires explicit consent.
5. **Pass the allowlist context to every dispatched agent.** Include the cached allowlist (or the relevant subset) in each subagent prompt so it inherits the same discipline. If you forget, the agent will return a "blocked: missing allowlist context" note — re-dispatch with it included.
6. **If a subagent reports a blocker** ("needed command X is not allowlisted"), surface that to the user via `AskUserQuestion` — propose the command and an allowlisted alternative if one exists — before re-dispatching.

## Workflow

### 1. Capture scope

- If `/why` has an argument, that is the seed question or context.
- Else the seed is the current problem from the conversation. State your reading of it back in one sentence and proceed; don't ask the user to re-explain what you already heard.
- Derive a kebab-case slug from the seed question: 3–6 words, lowercase ASCII, strip punctuation, collapse whitespace to `-`. Cap at ~50 chars.
- Determine the artifact directory: prefer the git repo root (one allowlisted call to `git rev-parse --show-toplevel`); if that fails or the cwd isn't a repo, write to cwd. Note the chosen location in the artifact's metadata section.
- If `Why-<slug>.md` already exists at that location, append a numeric suffix (`Why-<slug>-2.md`, `-3.md`, …) rather than overwriting — prior investigations are durable evidence and should not be silently replaced.

### 2. Operator-side context (one batched round)

Before any investigation, batch 2–4 questions via `AskUserQuestion` to learn what the human already knows. Useful angles:

- What does "answered" or "root cause" look like for this question? (so you know when to stop drilling)
- What have they already tried, ruled out, or strongly suspect?
- Constraints they care about (timeline, blast radius, reversibility, who else is affected)
- Scope boundaries — what is explicitly **out** of scope

Keep it tight. One batch up front beats many small ones during the drill.

### 3. The why-loop

For each why-level (1 through N, typically 3–5):

1. **State the question** for this level. Why-1 is the seed; Why-K (K>1) is "why is the answer to Why-(K-1) the case?"
2. **Dispatch investigators in parallel** using the `Agent` tool with `subagent_type` chosen to match the role (see the dispatch table below). The agent's system prompt is already loaded by virtue of `subagent_type` — do not re-pass it. Do pass: the why-level question, the chain so far, the load-bearing operator context, and the allowlist context from `mcp__allowlist__get_allowed_permissions`. Phrase the prompt so the agent knows what answer would settle this level — do not just say "investigate."
3. **Optionally batch a targeted AskUserQuestion round** *only if* a sharp operator-side gap is blocking synthesis. If the evidence is sufficient, skip — do not pester the user every level.
4. **Synthesize** a single best-supported answer for this level. Cite which agent surfaced which evidence. Mark anything speculative as such; if conflicting evidence appears, say so.
5. **Decide to drill or stop.** Stop when:
   - The answer hits an external constraint that is not actionable from inside the project (regulation, hardware limit, third-party behavior).
   - The answer is a deliberate, documented decision with rationale (in standards, an ADR, or a referenced incident). Drilling further re-litigates a settled choice.
   - The user's stated success condition from step 2 has been met.
   - You have drilled five levels and the answers have stopped changing meaningfully.
   - Drilling further would not change the recommendation.

Do not pad the chain to hit five. A three-level chain that lands on a real root cause beats a seven-level chain that drifts.

### 4. Write the artifact

Write `Why-<slug>.md` to the directory chosen in step 1 using the template in `templates/why-output.md`. Then summarize for the user in chat: the seed question, the root cause in one sentence, and the top recommended next step. Link the artifact path. If the user skipped or cancelled the operator-round `AskUserQuestion`, proceed with the scope as stated and note the missing context in the "Open questions" section rather than blocking.

## Agent dispatch

Invoke the three companion agents by `subagent_type` via the `Agent` tool:

| Role | `subagent_type` |
|---|---|
| Global standards investigator | `why-standards` |
| Local project docs investigator | `why-local-docs` |
| Code + git history investigator | `why-code` |

For each dispatch, pass the why-level question, the accumulated chain so far, and any load-bearing operator context. Phrase the prompt so the agent knows what answer would settle this level — do not just say "investigate."

Launch the relevant subset in a single message so they run in parallel. You don't always need all three:

- Pure-code "why does this function behave this way" — code agent is primary; standards/local-docs often add little.
- Process or convention question ("why do we do X this way") — standards + local-docs are primary; code may add little.
- Architectural / decision question — local-docs (ADRs) first, then code for evidence the docs are still accurate.

## Output format

Follow `templates/why-output.md` for the artifact structure (scope, operator context, Why chain, root cause, numbered next steps, open questions, metadata). Use numbered lists for any items the user might reference back ("do 2 and 4"). Match the user's global communication preferences: direct, no editorialising, no sugarcoating, gender-neutral.

## Why this skill is shaped this way

Investigations fail most often by stopping at the first plausible-sounding answer or by speculating instead of citing evidence. The 5-Whys spine forces continuation past the first plausible answer; the three-agent dispatch forces grounding in something other than the model's prior. Batched `AskUserQuestion` is the single channel for operator input because the user has a hook that flags missed questions in plain chat — using `AskUserQuestion` is both correct and reliable. The artifact exists because the reasoning is the deliverable, not just the recommendation: future-you (or a teammate, or a JIRA reader) needs the evidence trail.

If a why-level can be answered without dispatching agents (e.g., the user already named the cause in the operator round), say so and skip the dispatch for that level rather than performing investigation theatre.
