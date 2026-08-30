---
name: attack
description: >
  Deploy a QA agent swarm to analyze the codebase and produce a prioritized findings report,
  implementation spec, and test plan. Use when the user wants to run QA analysis, find bugs
  across multiple dimensions (security, performance, correctness, architecture, etc.), or
  deploy a swarm of specialized QA agents. Triggers on: code review, QA audit, bug sweep,
  quality analysis, find issues, check for bugs, swarm analysis.
---

Orchestrate a QA Swarm analysis. Treat the text supplied with the skill invocation, or the user's current request when no explicit argument was supplied, as the analysis prompt (after removing any `--model` option, see below).

## Model selection

The swarm supports three model tiers: `opus`, `sonnet`, and `haiku`. Every role has a default (listed with each dispatch below and in its `../../agents/*.md` frontmatter), and the user can override the whole run with one option:

- `--model opus` | `--model sonnet` | `--model haiku` anywhere in the invocation text (also accept `model=opus` and plain-language forms such as "use opus" or "with haiku").
- Strip the option from the text before using the remainder as the analysis prompt.
- Store the result as `swarm_model`. When no option is given, `swarm_model` is `auto` and every role uses its default.

When `swarm_model` is set, pass it as the model for EVERY agent dispatched by this skill (core, optional, fix planner) and forward it to the implement handoff in Step 6. When it is `auto`, dispatch each role with its default. Model labels are recommendations; if the host cannot provide the requested tier, use the closest available model and print one line saying which model was substituted -- never block the run.

### Host model mapping

Tier names (`opus`, `sonnet`, `haiku`) are host-neutral. Resolve each role's tier (default or `swarm_model`) and then translate it to the host's model name at dispatch time:

| Tier | Claude Code | Codex |
|------|-------------|-------|
| opus | opus | Terra |
| sonnet | sonnet | Luna |
| haiku | haiku | Luna |

On Codex, also accept `--model terra` (= opus) and `--model luna` (= sonnet) as aliases. Keep `swarm_model` and all printed tier labels in tier terms; when running on Codex, print the mapped name next to the tier the first time it is used (for example `opus -> Terra`). Note that `sonnet` and `haiku` both resolve to Luna on Codex, so `--model haiku` saves nothing there.

Examples:
```
Claude Code: /qa-swarm:attack --model opus "audit the payment flow"
Codex:       $qa-swarm:attack --model opus "audit the payment flow"   (dispatches on Terra)
```

## Portable role loading

Resolve bundled files relative to this `SKILL.md`, not the current working directory. The role directory is `../../agents/` from this file. Before dispatching any specialist, read that role's Markdown file and include its body in the subagent prompt. Claude Code may expose the same files as namespaced agent types, but never depend on that registration: Codex discovers the skills and does not automatically register `agents/` files.

Use the host's native subagent facility. Parallelize independent roles in one batch when supported. Model labels below are recommendations; use the closest available model without blocking the run. Use an available structured-input facility for user gates, or ask in plain chat when none exists.

Follow this pipeline exactly. Do not skip steps.

## Timing

Track elapsed time for each phase. At the start of each step, capture a Unix timestamp with an available shell command. Store these timestamps so you can compute durations at the end.

## Step 1: SETUP + PRE-READ

Record the pipeline start time and store it as `t_start`.

### 1a. Build file tree and categorize

1. List all source files (exclude node_modules, target, dist, build, .git, vendor, __pycache__)
2. Categorize every source file into tags based on file path and name:
   - **auth**: authentication, authorization, login, session, token, JWT, OAuth
   - **api**: route definitions, controllers, handlers, REST/GraphQL endpoints
   - **db**: database models, queries, migrations, ORM, repositories
   - **io**: file I/O, network calls, HTTP clients, external service calls, cache clients
   - **state**: shared state, global variables, singletons, concurrent data structures
   - **config**: configuration files, env vars, feature flags, deployment configs
   - **logic**: business logic, algorithms, state machines, data transformations
   - **frontend**: UI components, templates, client-side code
   - **test**: test files (note but do not assign to agents)
   - **entry**: entry points, main files, startup/shutdown code

   A file can have multiple tags. When in doubt, include the tag.

3. Store the full file tree (paths only).

### 1b. Auto-detect project type

Detect from file extensions, names, and directory structure:
- Primary language(s), framework(s), project type (library, CLI, web service, etc.)
- Notable characteristics: has CI, has Docker, has API spec, etc.

### 1c. Select optional agents

Based on detected project type:
- **qa-config-env**: projects with environment-dependent deployment (Docker, k8s, .env files)
- **qa-type-safety**: dynamically typed languages or projects without strict type checking
- **qa-logging**: services that run in production (web services, daemons)
- **qa-backwards-compat**: libraries, public APIs, or projects with external consumers
- **qa-supply-chain**: projects with third-party dependencies (package.json, Cargo.toml, go.mod, etc.)
- **qa-state-mgmt**: frontend apps or stateful services

### 1d. Pre-read all source files

**This is the key performance optimization.** Read ALL non-test source files and store their contents grouped by tag. This eliminates agent file-reading overhead -- agents receive code inline and analyze immediately with zero tool calls.

1. For each non-test source file, read it with available filesystem tools (cap at 750 lines per file -- if longer, read first 400 + middle 100 lines around the midpoint + last 150 lines with `[... {N} lines omitted ...]` markers between sections)
2. Group the file contents by tag. A file with multiple tags appears in multiple groups.
3. Format each file as:
   ```
   === {file_path} ===
   {file contents}
   ```

Parallelize independent reads to speed up this phase.

### 1e. Print summary and confirm

```
QA Swarm -- Codebase Summary
==============================
Source files: {file_count}
Estimated lines: ~{line_count}

Detected: {language(s)} {framework(s)} {project_type}

Agents to deploy:
  Core (6):  Security & Error, Performance & Resources, Correctness, Architecture, Data Flow, Async Patterns
  Optional:  {list of selected optional agents, or "none"}

Model: {swarm_model, or "auto (opus: security, architecture, compat; sonnet: other core + fix planner; haiku: other optional)"}
{on Codex, add: "Codex mapping: opus -> Terra, sonnet/haiku -> Luna"}

Estimated cost (API tokens, auto model mix):
  Small project  (< 50 files):   ~$0.30-0.80
  Medium project (50-200 files): ~$0.80-2.50
  Large project  (200+ files):   ~$2.50-6.00
  {if swarm_model is opus: "Opus selected -- expect roughly 3x the figures above."}
  {if swarm_model is haiku: "Haiku selected -- expect roughly 1/5 of the figures above."}
  {if swarm_model is haiku on Codex: "Haiku maps to Luna on Codex (same as sonnet) -- no additional savings."}

Proceed? (Y/n, adjust optional agents: "+logging -supply-chain", or change model: "model=opus")
```

Wait for user confirmation. If "n", stop. Parse any agent adjustments and any `model=` change (update `swarm_model` accordingly).

Record timestamp: `t_setup_done`.

## Step 2: SWARM

Print:
```
[Phase 1/3] Deploying {N} QA agents in parallel...
  Core: Security & Error, Performance & Resources, Correctness, Architecture, Data Flow, Async Patterns
  Optional: {list or "none"}
```

Launch ALL agents (core + optional) IN PARALLEL in a single message. Each agent gets its scoped file CONTENTS embedded directly -- no file paths to read.

For each agent, use this prompt template:

```
You are being deployed as part of a QA swarm analysis.

MISSION: {user's original prompt}

IMPORTANT: All source code is provided inline below. Do NOT use the Read tool -- analyze the code directly from this prompt. This is a performance optimization to eliminate file-reading overhead.

FULL FILE TREE (for reference -- paths only):
{the file tree from Step 1}

YOUR SCOPED SOURCE CODE:
{the actual file contents for this agent's tagged files, formatted as === path === \n content}

{Read ../../agents/qa-{name}.md relative to this skill and include its full content here as the agent's instructions}

Analyze the code provided above according to your specialty. Return your findings as structured JSON.
```

**Core agents and their scoped file contents:**

1. **qa-security-error** (default model: opus) -- receives contents of: auth + api + db + config + io + entry + logic files
2. **qa-performance-resources** (default model: sonnet) -- receives contents of: db + io + api + logic + state + entry + config files
3. **qa-correctness** (default model: sonnet) -- receives contents of: db + api + logic + io + config files
4. **qa-architecture** (default model: opus) -- receives contents of: entry + api + logic + config + db + io files
5. **qa-data-flow** (default model: sonnet) -- receives contents of: auth + api + db + io + logic + entry files
6. **qa-async-patterns** (default model: sonnet) -- receives contents of: api + io + logic + state + db + entry files

**Optional agents and their scoped file contents (if selected):**

- **qa-config-env** (default model: haiku): config + entry + io files
- **qa-type-safety** (default model: haiku): logic + api + db files
- **qa-logging** (default model: haiku): io + api + entry files
- **qa-backwards-compat** (default model: opus): api + db + config files
- **qa-supply-chain** (default model: haiku): config files + dependency/package files
- **qa-state-mgmt** (default model: haiku): state + frontend + logic files

The default models above apply only when `swarm_model` is `auto`. When the user chose `--model opus|sonnet|haiku`, dispatch every agent in this step with `swarm_model` instead. In both cases translate the tier through the host model mapping (Codex: opus -> Terra, sonnet/haiku -> Luna) before launching.

Launch all selected roles in one parallel batch when the host supports it. Each prompt must include the corresponding bundled role definition loaded from `../../agents/`.

**Wait for ALL agents to complete.** If any agent fails, log it and continue:
```
Agent {name} failed: {error}
Continuing with {N}/{total} agent results.
```

Record timestamp: `t_swarm_done`.

## Step 3: INLINE AGGREGATION

Print:
```
[Phase 2/3] Aggregating and ranking findings...
```

**Do NOT launch an agent for this step.** Perform aggregation inline to save time.

### 3a. Merge and deduplicate

Combine all agent findings. Identify duplicates:
- Same file + same line range (within 5 lines) = duplicate
- Same file + same function + similar title = duplicate

For duplicates, keep the one with higher confidence and build a `flagged_by` array. Print:
```
Dedup: {original_count} findings -> {deduped_count} ({removed} duplicates merged)
```

### 3b. Validate severity

Review each finding's severity:
- **P0 Critical** must be actively exploitable, cause data loss, or crash production. If it's really a code smell, downgrade.
- **P1 High** must cause real problems under normal usage. Not theoretical.
- **P2 Medium** is for latent risks and code smells that will compound.
- **P3 Low** is for improvement opportunities.

Confidence gates:
- P0 requires confidence >= "likely" OR corroboration by 3+ agents
- P1 requires confidence >= "likely" OR corroboration by 2+ agents
- P2-P3 can have "suspected" confidence if evidence is quoted

If P0 but only "suspected" with no corroboration, downgrade to P1 or P2.

### 3c. Validate confidence

- **"confirmed"** requires a traceable path and quotable code snippet that is unambiguously wrong
- **"likely"** requires strong evidence with acknowledged runtime uncertainty
- **"suspected"** is for patterns that look wrong but could be intentional

Downgrade if evidence doesn't support the tag. No specific file path + code snippet = cannot be "confirmed."

### 3d. Apply corroboration scoring

- Normalize file paths before matching
- Match by: same file + same function, OR same file + line numbers within 5 lines
- Count distinct agents per issue
- 3+ agents: boost confidence one level if evidence supports it
- 2 agents: note corroboration, no auto-boost
- 1 agent: stands on its own

### 3e. Format the report

Compile the final report in this format:

```markdown
# QA Swarm Report
**Date:** {DATE}
**Prompt:** "{ORIGINAL_PROMPT}"
**Agents deployed:** {COUNT} ({CORE_COUNT} core + {OPTIONAL_COUNT} optional)

## Summary
- P0 Critical: {N} findings
- P1 High: {N} findings
- P2 Medium: {N} findings
- P3 Low: {N} findings
- Total: {N} findings ({N} confirmed, {N} likely, {N} suspected)

## P0 - Critical

### [P0-001] {title}
**Confidence:** {confidence} | **Corroborated by:** {N} agents ({agent_list})
**Location:** {file}:{line} in `{function}`
**Description:** {description}
**Evidence:**
\`\`\`
{evidence}
\`\`\`
**Suggested fix:** {suggested_fix}
**Related files:** {related_files}

## P1 - High
(same format)

## P2 - Medium
(same format)

## P3 - Low
(same format)
```

Rules:
- Number findings sequentially: P0-001, P0-002, P1-001, etc.
- Every finding MUST have file path, line number, and evidence
- Remove findings that lack concrete evidence
- Merge findings that describe the same issue, crediting all agents
- The report must be self-contained
- Do NOT add your own findings -- only organize what agents found
- Be conservative. A report full of P0s loses credibility.

Record timestamp: `t_agg_done`.

Print a table of ALL findings sorted by severity then confidence:

```
Findings Summary
==================

| ID     | Severity | Confidence | Title                          | Location                        | Corroborated By   |
|--------|----------|------------|--------------------------------|---------------------------------|--------------------|
| P0-001 | P0       | confirmed  | SQL injection in login handler | src/auth.ts:42 `handleLogin()`  | 3 agents (SEC,ERR) |
| ...    | ...      | ...        | ...                            | ...                             | ...                |
```

Print every finding -- do not truncate.

## Step 4: FIX PLANNER

Print:
```
[Phase 3/3] Generating implementation spec and test plan...
```

Launch ONE agent:

**qa-fix-planner** (default model: sonnet; use `swarm_model` when set):
- Receives the final ranked report (full markdown from Step 3)
- Has access to the codebase (to read existing test patterns and verify P0 evidence)
- Produces BOTH the implementation spec AND the test plan

Read `../../agents/qa-fix-planner.md`, resolved relative to this skill, and include its full content in the prompt.

Record timestamp: `t_output_done`.

When the agent returns, split its output on the `===SPEC_START===` / `===SPEC_END===` / `===TESTS_START===` / `===TESTS_END===` delimiters to extract the two documents.

## Step 5: SAVE + HANDOFF

Print:
```
Saving reports...
```

Get today's date and save the three output files:
1. Write the report (from Step 3) to `docs/qa-swarm/{DATE}-report.md`
2. Write the spec (from Step 4) to `docs/qa-swarm/{DATE}-spec.md`
3. Write the test plan (from Step 4) to `docs/qa-swarm/{DATE}-tests.md`

Create the `docs/qa-swarm/` directory if it does not exist.

Record timestamp: `t_save_done`.

Compute phase durations (format as Xm Ys):
- Setup + Pre-read: `t_setup_done - t_start`
- User Confirm: (skip from total)
- Agent Swarm: `t_swarm_done - t_setup_done`
- Aggregation: `t_agg_done - t_swarm_done`
- Fix Planner: `t_output_done - t_agg_done`
- Save Files: `t_save_done - t_output_done`
- Total: `t_save_done - t_start` minus user confirm wait

Count agents dispatched, grouped by tier (on Codex, append the mapped name, e.g. `Opus (Terra)`, `Sonnet (Luna)`, `Haiku (Luna)`):
- Core agents: always 6 (default: security-error and architecture on Opus, the other 4 on Sonnet)
- Optional agents: count selected (default: backwards-compat on Opus, the rest on Haiku)
- Fix Planner: always 1 (default Sonnet)
- When `swarm_model` is set, all `7 + optional_count` agents count under that single model.

```
QA Swarm Analysis Complete
============================
Findings: {total} ({P0} P0, {P1} P1, {P2} P2, {P3} P3)
Confidence: {confirmed} confirmed, {likely} likely, {suspected} suspected
Model: {swarm_model, or "auto"}

Phase Timing:
  Setup + Pre-read  {Xm Ys}
  Agent Swarm       {Xm Ys}   ({N} agents in parallel: 6 core + {optional_count} optional)
  Aggregation       {Xm Ys}   (inline -- no agent)
  Fix Planner       {Xm Ys}   (1 agent)
  Save Files        {Xm Ys}
  ────────────────────────
  Total             {Xm Ys}   (excludes user confirmation wait)

Agent Usage:
  Opus   : {opus_count} agents  ({breakdown, e.g. "2 core + 1 optional"})
  Sonnet : {sonnet_count} agents  ({breakdown, e.g. "4 core + 1 fix planner"})
  Haiku  : {haiku_count} agents  ({breakdown, e.g. "{N} optional"})
  Total  : {7 + optional_count} agents dispatched

Report:    docs/qa-swarm/{DATE}-report.md
Spec:      docs/qa-swarm/{DATE}-spec.md
Test Plan: docs/qa-swarm/{DATE}-tests.md
```

## Step 6: AUTO-HANDOFF TO IMPLEMENT (fresh-context subagent)

Immediately after printing the summary, auto-invoke the implement phase in a **fresh-context subagent**. The subagent starts with zero context from attack -- this is the functional equivalent of `/clear` before running implement, without requiring manual user action.

Ask the user once:
```
Proceed to implementation now? [Y/n]
(Selecting Y hands off to a fresh-context subagent running the QA Swarm implement skill.
 Selecting n stops here -- you can resume later by running:
   Claude Code: /qa-swarm:implement docs/qa-swarm/{DATE}-report.md docs/qa-swarm/{DATE}-spec.md docs/qa-swarm/{DATE}-tests.md{ " --model {swarm_model}" if set }
   Codex: $qa-swarm:implement docs/qa-swarm/{DATE}-report.md docs/qa-swarm/{DATE}-spec.md docs/qa-swarm/{DATE}-tests.md{ " --model {swarm_model}" if set })
```

If the user declines (n), STOP.

If the user confirms (Y or empty), spawn a fresh-context general subagent with the following self-contained prompt. Resolve the sibling implementation skill as `../implement/SKILL.md` relative to this file and pass its absolute path; the subagent has no access to this session's context, so the prompt MUST stand alone. Append `--model {swarm_model}` to the invocation input only when the user chose a model (omit it for `auto` so implement uses its own per-role defaults):

```
You are executing the QA Swarm implement skill in a fresh session.

Read the complete skill instructions at {implement_skill_abs_path}. Treat these as
the active instructions and execute them with this invocation input:
  {report_abs_path} {spec_abs_path} {tests_abs_path}{ " --model {swarm_model}" if set }

All three files already exist on disk. Read them fresh. Follow the skill
exactly -- including phase selection (present the table and wait for user input
using an available structured-input facility or plain chat), TDD setup (up to 3 parallel test-writer agents), phase
execution, and final results report.

When the skill completes, return a concise summary: phases run, issues fixed,
issues unresolved, test pass/fail counts, and the path to the results file.
Do not re-describe work the user already saw -- just the outcome.
```

Use absolute paths for the three files so the subagent's path resolution does not depend on any shared working-directory state.

When the subagent returns, print its summary verbatim and STOP.
