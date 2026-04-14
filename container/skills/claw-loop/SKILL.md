---
name: claw-loop
description: "Set up The Claw Loop — an autonomous BMAD development workflow. Orchestrates Claude Code CLI to run create-story, dev-story, and code-review cycles against a mounted project. Use when the user says /claw-loop, \"start the claw loop\", or \"set up autonomous dev\"."
---

I want you to set up **The Claw Loop** — an autonomous development workflow where you (my NanoClaw agent) orchestrate Claude Code CLI running inside your container to build software using the BMAD method.

This prompt has two parts:
- **PART 1: CONCEPTS** — Read this to understand how the system works. No actions required.
- **PART 2: EXECUTE SETUP** — Follow these steps in order to activate the loop.

After setup, a scheduled task fires every 3 minutes with a **lightweight ping** that tells you to continue working. The full procedure is saved to a reference file (`/workspace/group/claw-loop-procedure.md`) during setup — you read it when needed (first fire, after context reset, or if you lose track), not every cycle. This keeps payloads small and prevents context window bloat from repeated prompt injection.

---

# PART 1: CONCEPTS — Read and Internalize

*This section teaches you how the Claw Loop works. Read it carefully. No actions to take yet.*

---

## 1.1 What Is the Claw Loop?

You are an **orchestrator**. Claude Code CLI (`claude -p`) is the **worker**. You run Claude Code as a subprocess inside your container to execute BMAD slash commands that build software. Each invocation is a fresh context — no session management needed.

**Your job:** A scheduled task wakes you up every 3 minutes. Each time, you check the state file, determine which BMAD step to run next, execute it via `claude -p`, parse the output, update state, and report to your human.

**Key facts:**
- **Claude Code CLI is installed in your container** at the global npm path. You run it via `claude -p` (print/pipe mode) which executes non-interactively.
- **The target project is mounted** at `/workspace/extra/developer-experience`. This contains the BMAD project with `.claude/commands/`, `_bmad-output/`, and all source code.
- **Each `claude -p` invocation is a fresh context.** No need for `/clear` — every run starts clean. This is a major simplification over tmux-based orchestration.
- **The state file is your brain.** You wake up fresh every cycle with no memory of the last one. The JSON state file tells you where you are. Always read it, always update it.
- **Reporting is not optional.** Your human should never wonder what's happening. Every cycle, they get a short update.

## 1.2 Core Principles

1. **State file is the source of truth** — Not your memory. The state file decides what step you're on.
2. **One step per cycle** — Each task fire runs one BMAD step to completion, then exits.
3. **Fresh context every time** — Each `claude -p` call starts with a clean context window. No session carryover.
4. **Parse output carefully** — The `claude -p` stdout is your only signal for what happened. Parse it for completion markers, errors, and file paths.
5. **Fail gracefully** — Non-zero exit codes, timeouts, and API errors are expected events, not emergencies.
6. **Report everything** — Your human trusts you because they can see what you're doing.
7. **Never advance a broken story** — If code review found HIGH/CRITICAL issues, re-run code-review (up to 3 passes). Dev-story runs exactly once.

## 1.3 How Claude Code CLI Works in Your Container

The `claude` CLI is installed globally in the container image. You invoke it in pipe/print mode:

```bash
cd /workspace/extra/developer-experience && claude -p --model <model> --dangerously-skip-permissions "<prompt or /slash-command>"
```

**Key flags:**
- `-p` / `--print` — Non-interactive mode. Sends prompt, prints result, exits.
- `--model <name>` — Select model (e.g., `opus`, `sonnet`). Replaces the tmux `/model` command.
- `--dangerously-skip-permissions` — Required for autonomous operation (no human to approve tool calls).

**Slash commands** (`.claude/commands/`) are auto-discovered from the working directory. Running `claude -p "/bmad-bmm-dev-story story.md"` expands the BMAD command template and executes it.

**Output:** stdout contains Claude's full response (tool call results, generated code, status messages). stderr contains debug logs. Capture stdout for parsing.

**Exit codes:** 0 = success, non-zero = error (API failure, timeout, crash).

### Auto-Continue Preamble

Since `claude -p` is non-interactive, BMAD template checkpoints (like `[a] [c] [p] [y]`) can't pause for input. Every slash command invocation is wrapped with an **auto-continue preamble** that tells Claude Code how to handle routine checkpoints:

```
AUTONOMOUS MODE — You are running non-interactively. Follow these rules for any checkpoint or prompt:
- At BMAD template checkpoints ([a] [c] [p] [y]): choose [c] Continue
- At (y/n) or (y/n/edit) prompts: choose y
- At numbered option lists [1], [2], [3]: choose [1] (fix automatically / default)
- At "Continue to next step?" prompts: choose yes
- If you encounter a HALT condition: include the word HALT in your output and stop
- IMPORTANT: If you encounter a genuine decision point where the right choice is ambiguous
  (architecture trade-offs, unclear requirements, multiple valid approaches, security implications),
  DO NOT guess. Instead, output the marker DECISION_NEEDED followed by the question and options
  on the next line. Format: DECISION_NEEDED: <question> | OPTIONS: <option1>, <option2>, ...
Now execute: <slash-command>
```

This preamble replicates what ClawdBot's tmux observer did — auto-responding to routine checkpoints while flagging genuine decisions for human input.

### Decision Points — Pause and Ask

When `claude -p` output contains `DECISION_NEEDED`, the Claw Loop:
1. Extracts the question and options from the output
2. Pauses the loop (`status: "awaiting-human-decision"`)
3. Sends the question to the human via `mcp__nanoclaw__send_message`
4. Saves the question and partial context in the state file
5. On the next fire, checks if the human has responded
6. If yes: re-runs the step with the human's answer prepended to the prompt
7. If no: waits (keeps reporting `[WAITING]` each cycle until the human answers)

This gives you control over genuine architecture/design decisions while keeping the loop fully autonomous for routine work.

## 1.4 Intelligent Model Strategy

Different steps benefit from different models. The Claw Loop uses a **pre-computed model strategy file** (`claw-loop-model-strategy.yaml`) that assigns the right model tier per epic and story based on complexity analysis.

**Two model tiers (never use the lowest-tier model for development):**

| Tier | Current Model | Purpose |
|------|--------------|---------|
| `highest` | Best reasoning model available (currently Opus) | Complex stories: auth, payments, concurrency, algorithms, RLS policies |
| `standard` | Fast execution model available (currently Sonnet) | Straightforward stories: UI components, CRUD, scaffolding, config |

**Step-level defaults:**

| Step | Tier | Why |
|------|------|-----|
| create-story | `highest` — always | Story creation requires deep architecture analysis |
| code-review | `highest` — always | Adversarial review requires deep reasoning |
| dev-story | **varies by epic/story** | Determined by `claw-loop-model-strategy.yaml` |

**How model resolution works at runtime:**
1. Read `claw-loop-model-strategy.yaml`
2. For create-story or code-review: always use `model_tier_mapping.highest`
3. For dev-story: check `story_overrides[current_story_key]` then `epic_overrides[current_epic]` then `step_defaults.dev_story`
4. Map the tier name to actual model via `model_tier_mapping`
5. Pass `--model <resolved-model-name>` to `claude -p`

**NEVER hardcode model names.** Always resolve from the strategy file. When Anthropic releases new models, you update two lines in the mapping section — every epic/story decision still works.

### The claw-loop-model-strategy.yaml File

This file lives at `_bmad-output/implementation-artifacts/claw-loop-model-strategy.yaml` inside the project directory. It's generated once during pre-sprint setup (Step B2) and updated optionally after epic completions.

```yaml
# ============================================================
# CLAW LOOP MODEL STRATEGY
# ============================================================
# Generated by: The Claw Loop v2.4 — Autonomous Dev Orchestrator
#
# TIER DEFINITIONS:
# highest: Best available reasoning model. Use for complex logic.
# standard: Fast execution model. Use for established patterns.
#
# IMPORTANT: Never use lowest-tier models (e.g., Haiku) for development.
# FUTURE-PROOFING: Update ONLY model_tier_mapping when new models release.

model_tier_mapping:
  highest: sonnet    # Update this line when new models release
  standard: sonnet   # Update this line when new models release

step_defaults:
  create_story: highest    # Always — deep reasoning for architecture analysis
  code_review: highest     # Always — adversarial review needs depth
  dev_story: standard      # Default for implementation, overridden below

epic_overrides:
  epic-1:
    dev_model: standard
    rationale: "Project scaffolding, design system, basic forms"
  epic-2:
    dev_model: highest
    rationale: "AI-assisted creation, randomization algorithm"
  # ... (populated by pre-sprint analysis)

story_overrides:
  1-3-owner-registration-and-account-creation:
    dev_model: highest
    rationale: "RLS policy setup, auth flow — security-critical"
  # ... (populated by pre-sprint analysis)
```

### Complexity Criteria — The Classification Rubric

During pre-sprint analysis (Step B2), you give CC these **explicit, measurable criteria** to classify each epic. CC does NOT decide what's complex — it applies these rules.

**HIGHEST tier required — if the epic/story involves ANY of these:**
- Database schema design, RLS policies, or row-level security
- Authentication, authorization, JWT handling, or session management
- External API integration (Stripe, Twilio, SendGrid, Groq, or any third-party)
- Webhook handling or signature verification
- Concurrency, race conditions, or atomic operations (e.g., `UPDATE...RETURNING`)
- State machines or multi-state lifecycle management
- Algorithms beyond simple CRUD (randomization, scoring, escalation chains, scheduling)
- Real-time data (subscriptions, live updates, WebSocket, polling)
- Financial calculations, billing, proration, or payment processing
- Background jobs, cron tasks, queue processing, or worker functions
- Data migration, retention policies, cleanup jobs, or archival logic
- Multi-tenant data isolation or cross-tenant security boundaries
- Complex form validation with interdependent fields
- Image processing, compression pipelines, or media handling

**STANDARD tier sufficient — ALL of the following must be true:**
- Primarily UI components following the project's established design system
- Standard CRUD operations using existing service layer patterns
- Configuration, scaffolding, or boilerplate setup
- No new RLS policies or auth changes introduced
- No external API calls or webhook handlers
- No concurrency concerns or atomic operations
- No new algorithms or complex business logic

**The rule is conservative:** When in doubt, classify as `highest`.

## 1.5 BMAD V6 Implementation Cycle

The Claw Loop orchestrates BMAD V6's **quality-gated loop** with conditional branching and a **double-review pattern**:

```
create-story → dev-story (runs ONCE) → code-review (pass 1)
                                            |
                                            ├── CLEAN (0 HIGH/CRITICAL) → advance to next story
                                            └── HIGH/CRITICAL found → fix in-place → code-review (pass 2)
                                                                                        |
                                                                                        ├── CLEAN → advance
                                                                                        └── HIGH/CRITICAL found → fix → code-review (pass 3, FINAL)
                                                                                                                          |
                                                                                                                          └── advance (max 3 passes)
```

**Key rules:**
- **Dev-story runs exactly ONCE per story. It never re-runs.** Code-review always fixes issues in-place (Option 1).
- **Code-review re-runs only if it finds HIGH or CRITICAL issues.** It fixes them, then a fresh review verifies the fixes.
- **Maximum 3 code-review passes.** After the 3rd pass, the story advances regardless — no infinite loops.
- **Why re-review matters:** When code-review fixes HIGH/CRITICAL issues, those fixes are unreviewed code. A fresh re-review catches problems the first reviewer introduced.

### Story Lifecycle Through Sprint Status

BMAD V6 tracks every story in `sprint-status.yaml`:
```
backlog → ready-for-dev → in-progress → review → done
```
- **create-story** moves: `backlog` → `ready-for-dev`
- **dev-story** moves: `ready-for-dev` → `in-progress` → `review`
- **code-review** moves: `review` → `done` (approved) or back to `in-progress` (issues remain)

## 1.6 Failure Detection

Since each BMAD step runs as a `claude -p` subprocess, failure detection is straightforward:

- **Exit code non-zero:** CLI crashed, API error, or timeout. Retry once, then escalate.
- **Output contains "HALT":** Stop condition — alert human immediately.
- **Output contains "FAIL" or test failures:** Dev-story produced failing tests — do NOT advance.
- **Output contains HIGH/CRITICAL issues:** Code-review found problems — re-review after fixes.
- **No meaningful output:** CLI may have hit rate limits or context overflow — retry with fresh invocation.
- **Repeated failures (3+ on same story):** Pause loop, alert human.

## 1.7 Scheduled Task Health Monitoring & Context Efficiency

The scheduled task fires every 3 minutes, fixed. It never gets deleted, recreated, or adjusted. This eliminates the #1 cause of silent task death: failed interval adjustments.

**Lightweight payload pattern:** Each fire sends a small payload (~800 chars) that tells you to continue working and where to find the full procedure. The full procedure lives in `/workspace/group/claw-loop-procedure.md` — you read it when you need it (first fire, after context reset, lost context), not every cycle.

**NanoClaw scheduling:** The loop uses NanoClaw's built-in `schedule_task` MCP tool with `schedule_type: "interval"` and `schedule_value: "180000"` (3 minutes in milliseconds). Each fire runs in the group's container context with access to `/workspace/group/` files.

**Two-layer watchdog:**
- **Layer 1 (inside task):** Every fire writes a heartbeat timestamp to the state file — even smart-skip fires.
- **Layer 2 (watchdog task):** A separate 15-minute scheduled task checks the timestamp. If 10+ minutes stale, main loop died — alert human + recreate it.

## 1.8 BMAD V6 Output Recognition

When parsing `claude -p` output, look for these key patterns:

| Output Pattern | Meaning | Next Action |
|----------------|---------|-------------|
| `Story Status: ready-for-dev` | create-story completed | Advance to dev-story |
| `Story status updated to review` | dev-story completed | Check tests, advance to code-review |
| `Story Status: done` + no HIGH/CRITICAL | code-review passed | Advance to next story |
| HIGH/CRITICAL issues found | code-review found problems | Re-run code-review (up to 3 passes) |
| `HALT` | Stop condition | Alert human, do NOT continue |
| `FAIL` or test failure output | Tests failing | Do NOT advance, retry or escalate |
| Non-zero exit code | CLI error | Retry once, then escalate |

## 1.9 Response Format

Every cycle report to the human MUST follow this exact format. No freestyling, no narrative paragraphs, no cheerleading.

**Calculate elapsed time:** `elapsed = NOW - metrics.currentStoryStartedAt` (round to nearest minute). This tracks total time on the current story across all phases (create-story, dev-story, code-review).

**Templates by status:**

**[WORKING]** — Step is running or was just launched:
```
[WORKING] Story X.X | step | elapsed min | Model
SAW: <one line — what the step is doing or what output showed>
ACTION: None — <brief reason>
PROGRESS: Story N/Total | Epic N | Review pass: N/3
```

**[TRANSITION]** — Agent advanced the pipeline:
```
[TRANSITION] Story X.X | from-step → to-step | elapsed min
SAW: <one line — what triggered the transition>
ACTION: <what was done>
PROGRESS: Story N/Total | Epic N | Review pass: N/3
STATS: <elapsed> min | <N> cycles | <N> review passes | <model tier>
```

**[DONE]** — Story fully complete:
```
[DONE] Story X.X — Story Name
SAW: <what confirmed completion>
ACTION: Advancing to story Y.Y
PROGRESS: Story N/Total complete | Epic N (M remaining)
STATS: <elapsed> min | <N> cycles | <N> review passes | <model tier>
```

**[ERROR]** — Step failed:
```
[ERROR] Story X.X | step | attempt N
SAW: <what went wrong — exit code, error message>
ACTION: <recovery action taken>
PROGRESS: Story N/Total | Epic N | Review pass: N/3
```

**[WAITING]** — Awaiting human decision:
```
[WAITING] Story X.X | step | waiting N min
SAW: Decision needed — <first 60 chars of question>
ACTION: Awaiting human response (asked N min ago)
PROGRESS: Story N/Total | Epic N | Review pass: N/3
```

**[NEEDS-HUMAN]** / **[HALTED]** — Human intervention required:
```
[NEEDS-HUMAN] Story X.X | step
SAW: <what went wrong>
ACTION: Loop paused — awaiting human input
PROGRESS: Story N/Total | Epic N | Review pass: N/3
```

**Format rules:**
- Line 1 is ALWAYS `[STATUS] Story X.X | key-info` — glanceable in 1 second
- SAW is ALWAYS one line, max ~80 characters — forces compression
- ACTION is ALWAYS one line
- PROGRESS is ALWAYS `Story N/Total | Epic N | Review pass: N/3` — consistent positioning
- STATS line ONLY on [DONE] and [TRANSITION] — not needed every 3 minutes
- No emojis except on problems
- No "Next cycle:" predictions — speculative, often wrong
- No token counts, no fire/cycle counts in reports — these go in the activity log only
- No cheerleading, commentary, or narrative text — data only
- **Send reports via `mcp__nanoclaw__send_message`** — this delivers immediately to the current group while you're still working

---

# PART 2: EXECUTE SETUP — Follow These Steps in Order

*This section is procedural. Execute each step sequentially.*

---

## Step 0: Research the BMAD Method

Go research the BMAD Method to understand how it works. Search for "BMAD Method Claude Code" and "bmad-method" on GitHub/npm. Key things to learn:
- BMAD uses slash commands stored in `.claude/commands/` in the project
- The core cycle is: **create-story** → **dev-story** (once) → **code-review** (up to 3 passes if HIGH/CRITICAL found)
- Common BMAD slash commands:
  - `/bmad-bmm-create-story` — Generates a comprehensive story file with ACs, tasks, subtasks, and dev notes
  - `/bmad-bmm-dev-story` — Implements the story: writes code, migrations, components, hooks, and tests
  - `/bmad-bmm-code-review` — Adversarially reviews the implementation, finds issues, can fix them in-place
  - `/bmad-bmm-sprint-status` — Shows current sprint progress
- Each command is self-contained — it loads all the context it needs from project files
- Sprint-status.yaml is the authoritative queue — it tracks every story's state

**Important:** Don't just memorize commands — understand what each step DOES so you can recognize completion and know what comes next.

## Step 1: Ask Setup Questions

Ask these questions **one at a time, wait for each answer**:

**1. Confirm your story queue from sprint-status.yaml**
- Read `/workspace/extra/developer-experience/_bmad-output/implementation-artifacts/sprint-status.yaml`
- Parse all stories and their current statuses
- Present summary: "Found X stories across Y epics. Z are backlog, W are in-progress, V are done."
- Ask: "Should I start from the first backlog story, or do you want to specify a starting point?"

**2. What's your escalation preference?**
- **Autonomous:** Loop handles failures automatically up to 3 retries, only pauses for repeated failures
- **Conservative:** Loop pauses and alerts for any failure beyond first retry
- **Aggressive:** Loop handles everything including skipping stuck stories after 5 failures
- Default to autonomous if they don't have a preference

**3. Verify Claude Code CLI works in the container**
- Run: `claude --version` via Bash to confirm the CLI is available
- Run a quick test: `cd /workspace/extra/developer-experience && claude -p --dangerously-skip-permissions "What project is this? List the top-level directories."` to verify:
  - The CLI works
  - The project mount is accessible and writable
  - API credentials are flowing through OneCLI
- Show the human the test output

## Step 2: Parse Sprint-Status and Build the Queue

```
1. Read _bmad-output/implementation-artifacts/sprint-status.yaml from the project directory
2. Extract all story keys (pattern: N-N-name, NOT epic-N or epic-N-retrospective)
3. Group by epic number
4. Order: stories within each epic in sequence, epics in order
5. Identify the first story that is NOT "done" — this is your starting point
6. Store the full ordered queue in the state file
```

## Step 3: Run Pre-Sprint Model Strategy Analysis

This step generates `claw-loop-model-strategy.yaml`. You use `claude -p` to analyze the project.

1. **Run the analysis via Claude Code CLI:**
   ```bash
   cd /workspace/extra/developer-experience && claude -p --model opus --dangerously-skip-permissions "Read the following files:
   - _bmad-output/planning-artifacts/epics.md (or all epic*.md files)
   - _bmad-output/planning-artifacts/architecture.md

   For each epic, determine whether the dev-story step requires the 'highest' or 'standard' model tier. Apply these criteria EXACTLY:

   HIGHEST required if the epic involves ANY of:
   - Database schema design, RLS policies, or row-level security
   - Authentication, authorization, JWT, or session management
   - External API integration (Stripe, Twilio, SendGrid, Groq, etc.)
   - Webhook handling or signature verification
   - Concurrency, race conditions, or atomic operations
   - State machines or multi-state lifecycle management
   - Algorithms beyond CRUD (randomization, scoring, escalation, scheduling)
   - Real-time data (subscriptions, live updates, WebSocket, polling)
   - Financial calculations, billing, proration, or payments
   - Background jobs, cron tasks, queue processing, or workers
   - Data migration, retention policies, cleanup, or archival
   - Multi-tenant data isolation or cross-tenant security
   - Complex form validation with interdependent fields
   - Image processing, compression pipelines, or media handling

   STANDARD sufficient only if ALL of:
   - Primarily UI components following established design system
   - Standard CRUD with existing service layer patterns
   - No new RLS policies or auth changes
   - No external API calls or webhook handlers
   - No concurrency concerns
   - No new algorithms or complex business logic

   When in doubt, classify as 'highest.'

   Also identify any INDIVIDUAL STORIES that should override their epic's classification.

   Output your analysis as a YAML block with this exact structure:
   epic_overrides:
     epic-1:
       dev_model: highest/standard
       rationale: 'one line why'
   story_overrides:
     story-key-here:
       dev_model: highest/standard
       rationale: 'one line why'"
   ```

2. **Parse the YAML output** from Claude's response. Extract `epic_overrides` and `story_overrides`.

3. **Write `claw-loop-model-strategy.yaml`** to `/workspace/extra/developer-experience/_bmad-output/implementation-artifacts/` with the model_tier_mapping, step_defaults, and the analysis results.

4. **Show the human for approval.** List each epic with its classification. Ask: "Does this model strategy look right? Say 'approve' to continue or tell me what to change."

## Step 4: Create the State File

Create `/workspace/group/bmad-dev-state.json` in your group folder:

```json
{
  "currentStory": "1-1-project-initialization-and-development-environment",
  "currentStoryNumber": "1.1",
  "currentStoryFilePath": null,
  "currentEpic": 1,
  "currentStep": "create-story",
  "status": "running",
  "lastUpdated": "ISO-TIMESTAMP",
  "lastActionAt": "ISO-TIMESTAMP",
  "currentStepType": "create-story",
  "notes": "Claw Loop initialized from sprint-status.yaml.",
  "storyQueue": ["PARSED-FROM-SPRINT-STATUS"],
  "completedStories": [],
  "currentEpicStories": ["STORIES-IN-CURRENT-EPIC"],
  "stepOrder": ["create-story", "dev-story", "code-review"],
  "cronInterval": "3min-fixed-with-smart-skip",
  "escalationMode": "autonomous",
  "failureCount": 0,
  "reviewPassNumber": 1,
  "totalStoriesCompleted": 0,
  "totalEpicsCompleted": 0,
  "sessionStartedAt": "ISO-TIMESTAMP",
  "lastReviewOutcome": null,
  "lastExitCode": null,
  "pendingDecision": {
    "awaiting": false,
    "question": null,
    "options": null,
    "stepWhenAsked": null,
    "askedAt": null,
    "partialOutput": null,
    "humanResponse": null
  },
  "quarantinedStories": [],
  "gitVerification": {
    "pendingVerification": false,
    "verificationAttempts": 0,
    "lastVerifiedStory": null,
    "lastVerifiedCommit": null
  },
  "modelStrategy": {
    "currentModel": null,
    "currentModelTier": null,
    "modelSource": null,
    "fallbackActive": false,
    "modelStrategyFile": "_bmad-output/implementation-artifacts/claw-loop-model-strategy.yaml",
    "lastModelSwitchAt": null
  },
  "cronHealth": {
    "lastCronFire": "ISO-TIMESTAMP",
    "consecutiveFires": 0,
    "lastSkipReason": null,
    "cronStatus": "healthy"
  },
  "metrics": {
    "currentStoryStartedAt": "ISO-TIMESTAMP",
    "currentStoryCronFires": 0,
    "sprintStartedAt": "ISO-TIMESTAMP",
    "totalCronFires": 0,
    "totalCronSkips": 0,
    "activityLogFile": "_bmad-output/implementation-artifacts/claw-loop-activity.log",
    "completedStoryMetrics": []
  }
}
```

## Step 5: Create the Activity Log

Create `_bmad-output/implementation-artifacts/claw-loop-activity.log` in the project directory (`/workspace/extra/developer-experience/`):

```
# ============================================================
# CLAW LOOP ACTIVITY LOG
# ============================================================
# Generated by: The Claw Loop v2.4 — Autonomous Dev Orchestrator
# Project: {project_name}
# Started: {ISO-TIMESTAMP}
# Sprint: {total_stories} stories across {total_epics} epics
#
# This file is an append-only audit trail of all Claw Loop
# activity. It is NOT a BMAD artifact — it is created and
# maintained by the Claw Loop orchestration system.
#
# Format: TIMESTAMP | EVENT | DETAILS...
# ============================================================
```

**Event types:**
```
CRON_FIRE   | story:X.X | step:STEP | model:MODEL(TIER) | action:ACTION
CRON_SKIP   | story:X.X | step:STEP | reason:REASON
TRANSITION  | story:X.X | from:STEP | to:STEP | model:OLD→NEW
STEP_RUN    | story:X.X | step:STEP | model:MODEL(TIER) | exit_code:N
STEP_OUTPUT | story:X.X | step:STEP | summary:FIRST-200-CHARS
REVIEW_DONE | story:X.X | outcome:OUTCOME | pass:NofN | action:ACTION
STORY_DONE  | story:X.X | duration:NNmin | cron_fires:N | review_passes:N | dev_model:TIER
EPIC_DONE   | epic:N | stories:N | total_duration:NNmin | avg_review_loops:N.N
FAILURE     | story:X.X | step:STEP | exit_code:N | failureCount:N | action:ACTION
MODEL_SWITCH | from:MODEL | to:MODEL | tier:TIER | source:SOURCE
MODEL_FALLBACK | reason:REASON | action:all-highest
GIT_VERIFIED | story:X.X | commit:HASH | files:N
GIT_UNCOMMITTED | story:X.X | action:requested-commit
GIT_ISSUE   | story:X.X | action:alerted-human-proceeding
DECISION_NEEDED | story:X.X | step:STEP | question:FIRST-100-CHARS
DECISION_ANSWERED | story:X.X | step:STEP | answer:ANSWER | wait_time:Nmin
QUARANTINE  | story:X.X | reason:REASON
HUMAN_CMD   | command:CMD | reason:REASON
CRON_HEALTH | status:STATUS | consecutiveFires:N
CRON_RECOV  | downtime:Nmin | action:watchdog-recreated-task
```

## Step 6: Save the Cron Procedure File

Save the **entire CRON PROCEDURE** section below (everything between the ``` fences at the end of this document) to `/workspace/group/claw-loop-procedure.md` in your group folder. This is the detailed step-by-step procedure you'll reference on each task fire. The task payload itself will be lightweight — it just points you here.

Verify the file was saved correctly and is readable.

## Step 7: Create the Scheduled Loop Task

Use the `schedule_task` MCP tool to create the main dev loop task:

```
schedule_task(
  prompt: "<lightweight payload below>",
  schedule_type: "interval",
  schedule_value: "180000"
)
```

**Lightweight task payload (use this as the `prompt` parameter):**

```
BMAD V6 DEV LOOP — Task fire.

Full procedure: /workspace/group/claw-loop-procedure.md
State file: /workspace/group/bmad-dev-state.json
Project directory: /workspace/extra/developer-experience
Model strategy: /workspace/extra/developer-experience/_bmad-output/implementation-artifacts/claw-loop-model-strategy.yaml
Activity log: /workspace/extra/developer-experience/_bmad-output/implementation-artifacts/claw-loop-activity.log

Continue the dev loop from where you left off. Execute steps 0-4 in order as documented in the procedure file.

If this is your first fire, after a context reset, or if you've lost track of the procedure:
Read /workspace/group/claw-loop-procedure.md end-to-end before acting.
Otherwise, continue from your current state — the state file tells you where you are.

Key rules (always active):
NEVER advance with failing tests or unresolved HIGH/CRITICAL review issues
ALWAYS update state file after any action — it is your only memory
ALWAYS report every task fire to the human via mcp__nanoclaw__send_message
NEVER hardcode model names — resolve from claw-loop-model-strategy.yaml
If rate limited, stop and report — do not retry
```

## Step 8: Create the Daily Summary Task

Use `schedule_task` to create the daily summary:

```
schedule_task(
  prompt: "<summary prompt below>",
  schedule_type: "cron",
  schedule_value: "0 8 * * *"
)
```

Adjust the cron expression to the human's preferred time. The `prompt` parameter:

```
CLAW LOOP DAILY SUMMARY — Read these files and generate a sprint progress report:

1. Read /workspace/group/bmad-dev-state.json
2. Read /workspace/extra/developer-experience/_bmad-output/implementation-artifacts/claw-loop-activity.log
3. Count for the last 24 hours:
   - Stories completed (list each with name and duration from STORY_DONE entries)
   - Stories currently in-progress
   - Any stories that required 3+ review passes (flag these specifically)
   - Failure events (count from FAILURE entries)
   - Any human interventions (HUMAN_CMD entries)
4. Calculate cumulative sprint progress:
   - Total stories done / total stories in queue
   - Epics completed / total epics
   - Current epic and story
5. Send via mcp__nanoclaw__send_message:

"CLAW LOOP DAILY SUMMARY — [DATE]
Stories completed today: X (Y total done / Z remaining)
Epics: N complete, M in progress
Current: Story X.X — [name]

Stories done today:
- X.X [name] (XX min, N review passes)
- X.X [name] (XX min, N review passes)

Flagged (3+ review passes): [list or 'None']
Failures: X total
Human interventions: X

Task health: X fires, Y skips"
```

## Step 9: Add Watchdog Task and Group Memory Entry

**Create a watchdog scheduled task** using `schedule_task`:

```
schedule_task(
  prompt: "<watchdog prompt below>",
  schedule_type: "interval",
  schedule_value: "900000"
)
```

The `prompt` parameter:

```
CLAW LOOP WATCHDOG — Check if the bmad-dev-loop is still alive.

1. Read /workspace/group/bmad-dev-state.json
2. If the file does NOT exist OR state.status is not "running": exit silently — loop is not active.
3. Read cronHealth.lastCronFire timestamp.
4. Calculate how long ago that was (in minutes).
5. If more than 10 minutes have passed:
   - The bmad-dev-loop task has died silently.
   - Send alert via mcp__nanoclaw__send_message: "[CRON DEAD] Claw Loop hasn't fired in [X] minutes. Alerting human."
   - Update state: cronHealth.cronStatus = "stale"
   - Append to activity log: "CRON_RECOV | downtime:Xmin | action:watchdog-alerted"
6. If less than 10 minutes: cron is healthy, exit silently — no message needed.
```

**Add to your group's CLAUDE.md** (`/workspace/group/CLAUDE.md`):

```markdown
## Active Automations

**Claw Loop v2.4** is active for project developer-experience at /workspace/extra/developer-experience.
- State file: `/workspace/group/bmad-dev-state.json`
- Procedure: `/workspace/group/claw-loop-procedure.md`
- Activity log: `/workspace/extra/developer-experience/_bmad-output/implementation-artifacts/claw-loop-activity.log`
- If status is "running", the bmad-dev-loop task should be firing every 3 minutes.
- Watchdog task checks every 15 minutes and alerts if stale.
```

## Step 10: Verify Everything Works

- Read back the state file and confirm it's valid JSON
- Read back the procedure file and confirm it was saved completely
- Confirm the activity log was created with the branded header
- Confirm the group CLAUDE.md entry was added
- List tasks via `list_tasks` to confirm all three are scheduled
- Send a status update via `mcp__nanoclaw__send_message`
- Tell the human: "Claw Loop v2.4 is live. Fixed 3-min task with smart-skip. Starting: Story [X.X] [name] (model: [tier] → [model]). Queue: [N] stories across [M] epics. Model strategy: [X] epics highest, [Y] epics standard. Watchdog active. Say 'pause loop' anytime to stop."

---

# CRON PROCEDURE — Reference File Content

**This is the text that gets saved to `/workspace/group/claw-loop-procedure.md` during Step 6.**
**The NanoClaw agent reads this file on its first task fire, after a context reset, or whenever it needs to re-orient. It is NOT injected into the task payload every fire — the lightweight ping (Step 7) tells the agent where to find this file.**

```
<cron-rules>
IMMUTABLE RULES — These override everything below. Violating any of these is a critical failure.

1. NEVER advance a story with failing tests — re-run dev-story until tests pass
2. NEVER advance a story that code-review found HIGH/CRITICAL issues on — re-run code-review (max 3 passes)
3. NEVER skip a story automatically — after 3 failures, pause and alert the human
4. NEVER hardcode model names — always resolve from claw-loop-model-strategy.yaml
5. ALWAYS update the state file after any action — it is your only memory
6. ALWAYS report every task fire to the human via mcp__nanoclaw__send_message — silence means the loop is dead
7. ALWAYS respect HALTs — stop and alert, never respond to a HALT
8. Dev-story runs exactly ONCE per story. Code-review fixes issues in-place.
</cron-rules>

BMAD V6 DEV LOOP — Execute these steps IN ORDER, every task fire:

PROJECT_DIR=/workspace/extra/developer-experience
STATE_FILE=/workspace/group/bmad-dev-state.json
STRATEGY_FILE=/workspace/extra/developer-experience/_bmad-output/implementation-artifacts/claw-loop-model-strategy.yaml
ACTIVITY_LOG=/workspace/extra/developer-experience/_bmad-output/implementation-artifacts/claw-loop-activity.log

<cron-step id="0" name="heartbeat-and-smart-skip">
STEP 0: HEARTBEAT + SMART-SKIP (do this FIRST, every fire, no exceptions)

  0a. Write heartbeat timestamp to state file:
    state.cronHealth.lastCronFire = NOW
    state.cronHealth.consecutiveFires += 1
    state.cronHealth.cronStatus = "healthy"

  0b. Smart-skip check:
    Read state.lastActionAt and state.currentStepType
    Calculate elapsed = NOW - lastActionAt

    IF currentStepType == "dev-story" AND elapsed < 8 min:
      THEN skip this cycle (dev-story runs longer, give it time)
      ACTION: Log "CRON_SKIP | reason:smart-skip(dev-story-elapsed<8min)"
      EXIT task

    IF currentStepType == "code-review" AND elapsed < 6 min:
      THEN skip this cycle
      ACTION: Log "CRON_SKIP | reason:smart-skip(code-review-elapsed<6min)"
      EXIT task

    IF currentStepType == "create-story" AND elapsed < 4 min:
      THEN skip this cycle
      ACTION: Log "CRON_SKIP | reason:smart-skip(create-story-elapsed<4min)"
      EXIT task

    ELSE PROCEED with full logic below
</cron-step>

<cron-step id="1" name="read-state-and-validate">
STEP 1: READ STATE + VALIDATE MODEL STRATEGY

  1a. Read /workspace/group/bmad-dev-state.json
    Extract: currentStory, currentStoryNumber, currentStep, status, reviewPassNumber,
    failureCount, currentStoryFilePath, modelStrategy, gitVerification

  1b. Check status:
    IF status == "halted" or "human-review-needed": EXIT (do not proceed)
    IF status == "paused": EXIT silently
    IF status == "awaiting-human-decision":
      Check if the human has sent a message since pendingDecision.askedAt
      (look for recent messages in the conversation or IPC input)
      IF human responded:
        STATE: pendingDecision.humanResponse = <their answer>
        STATE: status = "running"
        PROCEED to Step 2 (will re-run the step with human's answer)
      ELSE:
        ACTION: Report [WAITING] to human (remind them the question is pending)
        EXIT
    IF status != "running": EXIT

  1c. Read + VALIDATE claw-loop-model-strategy.yaml

    VALIDATION — the file MUST contain ALL of these keys:
    - model_tier_mapping.highest (non-empty string)
    - model_tier_mapping.standard (non-empty string)
    - step_defaults.create_story, step_defaults.code_review, step_defaults.dev_story
    - epic_overrides (object, can be empty)

    IF VALIDATION FAILS:
      STATE: modelStrategy.fallbackActive = true
      ACTION: Use SAFE DEFAULTS: highest = sonnet, standard = sonnet, ALL steps use "highest"
      ACTION: Alert human: "[MODEL-FALLBACK] claw-loop-model-strategy.yaml is invalid. Using all-highest."
      ACTION: Log "MODEL_FALLBACK | reason:<parse-error|missing-keys|file-not-found>"

    IF VALIDATION PASSES:
      STATE: modelStrategy.fallbackActive = false
</cron-step>

<cron-step id="2" name="resolve-and-run">
STEP 2: RESOLVE MODEL + RUN NEXT STEP

  --- MODEL RESOLUTION ---
  1. Read strategy file (loaded in Step 1c)
  2. IF step is create-story or code-review:
       tier = "highest" (always)
     IF step is dev-story:
       Check story_overrides[currentStory] — use if found
       ELSE check epic_overrides["epic-" + currentEpic] — use if found
       ELSE use step_defaults.dev_story
  3. Map tier to model name: model_tier_mapping[tier]
  4. STATE: modelStrategy.currentModel = resolved name, modelStrategy.currentModelTier = tier
  --- END MODEL RESOLUTION ---

  --- STORY FILE RESOLUTION ---
  The agent MUST know the story file path so CC never has to search for it.

  1. Check state.currentStoryFilePath — if set and non-null, use it
  2. IF null (e.g., after create-story needs to run):
     For create-story: no file path needed, story number is sufficient
     For dev-story/code-review: search the project directory:
       _bmad-output/implementation-artifacts/*<currentStory>*.md
     OR read sprint-status.yaml to find the story key and match to a file
  3. Store the resolved path in state.currentStoryFilePath
  --- END STORY FILE RESOLUTION ---

  --- BUILD AND RUN THE COMMAND ---
  Base command: cd /workspace/extra/developer-experience && claude -p --model <resolved-model> --dangerously-skip-permissions

  AUTO-CONTINUE PREAMBLE (prepend to EVERY slash command):
  ```
  AUTONOMOUS MODE — You are running non-interactively. Follow these rules for any checkpoint or prompt:
  - At BMAD template checkpoints ([a] [c] [p] [y]): choose [c] Continue
  - At (y/n) or (y/n/edit) prompts: choose y
  - At numbered option lists [1], [2], [3]: choose [1] (fix automatically / default)
  - At "Continue to next step?" prompts: choose yes
  - If you encounter a HALT condition: include the word HALT in your output and stop
  - IMPORTANT: If you encounter a genuine decision point where the right choice is ambiguous
    (architecture trade-offs, unclear requirements, multiple valid approaches, security implications),
    DO NOT guess. Instead, output the marker DECISION_NEEDED followed by the question and options
    on the next line. Format: DECISION_NEEDED: <question> | OPTIONS: <option1>, <option2>, ...
  Now execute:
  ```

  IF pendingDecision.humanResponse is set (re-running after human answered a decision):
    EXTRA CONTEXT prepended after preamble, before slash command:
    "The human was asked: '<pendingDecision.question>' and answered: '<pendingDecision.humanResponse>'. Apply this decision."
    STATE: pendingDecision = reset all fields to defaults

  IF currentStep == "create-story":
    PROMPT: "<preamble> /bmad-bmm-create-story <currentStoryNumber>"
    NOTE: create-story reads sprint-status.yaml; the story number helps find the right one

  IF currentStep == "dev-story":
    Resolve story file path first (see above)
    PROMPT: "<preamble> /bmad-bmm-dev-story <currentStoryFilePath>"
    NOTE: The file path is passed as an argument — CC uses it directly

  IF currentStep == "code-review":
    Resolve story file path first (see above)
    PROMPT: "<preamble> /bmad-bmm-code-review <currentStoryFilePath>"
    NOTE: The file path is passed as an argument — CC reviews it directly

  Run the full command via Bash:
    <base> "<prompt>" 2>&1 | tee /workspace/group/step-output.txt

  For long-running steps (dev-story), use background execution if the Bash tool
  timeout is insufficient. The output file at /workspace/group/step-output.txt
  can be read after completion.

  Capture: exit code, stdout (full output), stderr (debug info)
  STATE: lastActionAt = NOW, lastExitCode = <exit code>
  ACTION: Log "STEP_RUN | story:X.X | step:STEP | model:MODEL(TIER) | exit_code:N"
  --- END BUILD AND RUN ---
</cron-step>

<cron-step id="3" name="parse-and-transition">
STEP 3: PARSE OUTPUT + DECIDE NEXT STATE

  Read the command output (stdout from Step 2).

  <condition id="3a" trigger="HALT detected">
  IF output contains "HALT":
    STATE: status = "halted", notes = HALT message text
    ACTION: Alert human: "CC HALTED on story X.X: [halt reason]"
    ACTION: Log "FAILURE | action:halt-detected"
    Do NOT continue. Wait for human.
  </condition>

  <condition id="3a2" trigger="DECISION_NEEDED detected">
  IF output contains "DECISION_NEEDED":
    Extract the question and options from the line following the marker
    Expected format: DECISION_NEEDED: <question> | OPTIONS: <option1>, <option2>, ...

    STATE: status = "awaiting-human-decision"
    STATE: pendingDecision.awaiting = true
    STATE: pendingDecision.question = extracted question
    STATE: pendingDecision.options = extracted options
    STATE: pendingDecision.stepWhenAsked = currentStep
    STATE: pendingDecision.askedAt = NOW
    STATE: pendingDecision.partialOutput = first 500 chars of output (for context)
    ACTION: Alert human via mcp__nanoclaw__send_message:
      "[DECISION] Story X.X | <step> needs your input:
      <question>
      Options: <options>
      Reply with your choice to continue the loop."
    ACTION: Log "DECISION_NEEDED | story:X.X | step:STEP | question:<first 100 chars>"
    Do NOT continue. Wait for human response.
  </condition>

  <condition id="3b" trigger="non-zero exit code">
  IF exit code != 0:
    STATE: failureCount += 1
    IF failureCount >= 3:
      STATE: status = "human-review-needed"
      ACTION: Alert human: "Story X.X has failed 3 times on step <step>. Pausing for human review."
      ACTION: Log "FAILURE | exit_code:N | failureCount:3 | action:paused-for-human"
    ELSE:
      ACTION: Alert human: "Step <step> failed on story X.X (exit code N). Will retry next cycle."
      ACTION: Log "FAILURE | exit_code:N | failureCount:N | action:will-retry"
      NOTE: The step will re-run on the next task fire (state.currentStep unchanged)
  </condition>

  <condition id="3c" trigger="create-story completed">
  IF currentStep == "create-story" AND output contains "Story Status: ready-for-dev" or similar completion marker:
    CAPTURE story file path from output (create-story prints the path when saving)
    STATE: currentStoryFilePath = extracted path
    STATE: currentStep = "dev-story", failureCount = 0
    ACTION: Log "TRANSITION | from:create-story | to:dev-story | storyFile:<path>"
    NOTE: The actual dev-story run happens on the NEXT task fire
  </condition>

  <condition id="3d" trigger="dev-story completed">
  IF currentStep == "dev-story" AND output contains "Story status updated to review" or completion marker:
    FIRST: Check output for test results

    IF output contains "FAIL" or "failed" or test failure indicators:
      Tests failed — DO NOT advance to code-review
      STATE: failureCount += 1
      IF failureCount >= 2:
        STATE: status = "human-review-needed"
        ACTION: Alert human: "Tests failing repeatedly on story X.X. Pausing for human review."
      ELSE:
        ACTION: Alert human: "Tests failing on story X.X — will retry dev-story."
        NOTE: Dev-story will re-run on next fire (we set failureCount but keep currentStep)
        Actually: Dev-story runs exactly ONCE. If tests fail, we still advance to code-review
        and let code-review fix the issues. Update:
      ACTION: Log "TRANSITION | from:dev-story | to:code-review | note:test-failures-detected"
      STATE: currentStep = "code-review", failureCount = 0

    IF tests passed or no test failure indicators:
      STATE: currentStep = "code-review", failureCount = 0
      ACTION: Log "TRANSITION | from:dev-story | to:code-review"
  </condition>

  <condition id="3e" trigger="code-review completed">
  IF currentStep == "code-review":
    Parse output for review outcome:
    - Extract: Story Status ("done" or "in-progress")
    - Look for HIGH or CRITICAL issues found
    - Look for severity counts

    Dev-story NEVER re-runs. Code-review fixes issues in-place.
    The ONLY loop is code-review → code-review (max 3 passes total).

    IF output shows HIGH or CRITICAL issues were found:
      Issues were fixed in-place. Re-review needed to verify fixes.

      IF reviewPassNumber >= 3:
        Max review passes reached. Story is DONE — move on.
        ACTION: Alert human: "Story X.X advancing after 3 review passes. Some issues may remain."
        ACTION: Log "REVIEW_DONE | outcome:max-passes-reached | pass:3of3 | action:advancing"
        GOTO: GIT VERIFICATION below

      ELSE:
        STATE: reviewPassNumber += 1, lastReviewOutcome = "high-found-rereviewing"
        ACTION: Log "REVIEW_DONE | outcome:high-found | pass:Nof3 | action:re-review"
        NOTE: Code-review will re-run on next fire (currentStep stays "code-review")

    IF output contains "Story Status: done" AND NO HIGH or CRITICAL issues:
      Story APPROVED — clean review.
      ACTION: Log "REVIEW_DONE | outcome:clean | pass:Nof3 | action:advancing"
      GOTO: GIT VERIFICATION below

    IF output contains "Story Status: in-progress" or action items created:
      Treat same as HIGH issues found — re-review needed.
      IF reviewPassNumber >= 3: advance anyway (see above)
      ELSE: STATE: reviewPassNumber += 1
  </condition>

  --- GIT VERIFICATION (mandatory after every STORY_DONE) ---
  After code-review approves the story:

  1. Run: cd /workspace/extra/developer-experience && git status && git log --oneline -5
  2. Parse the output:
     IF commit found referencing the story:
       Log "GIT_VERIFIED | story:X.X | commit:<hash>"
       Proceed to ADVANCE
     IF uncommitted changes exist:
       Run: cd /workspace/extra/developer-experience && git add -A && git commit -m "feat(story-X.X): <story name> — implementation complete"
       STATE: gitVerification.verificationAttempts += 1
       IF commit succeeds: Log "GIT_VERIFIED" and proceed
       IF commit fails AND verificationAttempts >= 2:
         Alert human: "[GIT-ISSUE] Story X.X completed but changes aren't committed."
         Log "GIT_ISSUE | story:X.X | action:alerted-human-proceeding"
         Proceed to ADVANCE anyway
  --- END GIT VERIFICATION ---

  --- ADVANCE TO NEXT STORY ---
  STATE: Move currentStory to completedStories
  STATE: Advance currentStory to next in storyQueue
  STATE: currentStep = "create-story"
  STATE: Reset failureCount = 0, reviewPassNumber = 1
  STATE: totalStoriesCompleted += 1
  STATE: gitVerification = reset all fields
  STATE: currentStoryFilePath = null (new story, create-story will generate it)
  STATE: metrics.currentStoryStartedAt = NOW

  EPIC BOUNDARY CHECK:
  IF all stories in currentEpic show "done" in sprint-status.yaml:
    STATE: totalEpicsCompleted += 1
    ACTION: Alert human: "Epic N complete! All X stories done."
    ACTION: Log "EPIC_DONE | epic:N | stories:X | total_duration:Nmin"
    Optionally suggest: "Consider running /bmad-bmm-retrospective for Epic N"

  ACTION: Log "STORY_DONE | story:X.X | duration:NNmin | cron_fires:N | review_passes:N | dev_model:TIER"
  ACTION: Log "TRANSITION | from:code-review | to:create-story"
  NOTE: The actual create-story run happens on the NEXT task fire
  --- END ADVANCE ---
</cron-step>

<cron-step id="4" name="log-report-verify">
STEP 4: LOG, REPORT, AND VERIFY (do this EVERY cycle, including skips)

  4a. Append activity log entry to the activity log file:
    Format: TIMESTAMP | EVENT_TYPE | story:X.X | step:STEP | model:MODEL(TIER) | action:ACTION
    Log every task fire (CRON_FIRE or CRON_SKIP)
    On STORY_DONE: push completed story metrics to state.metrics.completedStoryMetrics

  4b. Update state.metrics:
    state.metrics.currentStoryCronFires += 1
    state.metrics.totalCronFires += 1

  4c. Send report to human via mcp__nanoclaw__send_message:

    USE THIS EXACT FORMAT — no freestyling, no narrative, no cheerleading.

    Calculate elapsed: NOW - metrics.currentStoryStartedAt (round to nearest minute)

    **[WORKING]** (step just launched, waiting for next fire to get results):
    ```
    [WORKING] Story X.X | <step> | <elapsed> min | <Model>
    SAW: <1 line — step launched or output summary>
    ACTION: None — awaiting completion
    PROGRESS: Story N/Total | Epic N | Review pass: N/3
    ```

    **[TRANSITION]** (step completed, advancing pipeline):
    ```
    [TRANSITION] Story X.X | <from-step> → <to-step> | <elapsed> min
    SAW: <1 line — what triggered the transition>
    ACTION: <what was done>
    PROGRESS: Story N/Total | Epic N | Review pass: N/3
    STATS: <elapsed> min | <N> cycles | <N> review passes | <model tier>
    ```

    **[DONE]** (story complete — only on STORY_DONE):
    ```
    [DONE] Story X.X — <Story Name>
    SAW: <what confirmed completion>
    ACTION: Advancing to story Y.Y
    PROGRESS: Story <N>/<Total> complete | Epic <N> (<M> remaining)
    STATS: <elapsed> min | <N> cycles | <N> review passes | <model tier>
    ```

    **[ERROR]** (step failed):
    ```
    [ERROR] Story X.X | <step> | attempt <N>
    SAW: <exit code, error summary>
    ACTION: <will retry / paused for human>
    PROGRESS: Story N/Total | Epic N | Review pass: N/3
    ```

    **[WAITING]** (awaiting human decision):
    ```
    [WAITING] Story X.X | <step> | waiting <N> min
    SAW: Decision needed — <first 60 chars of question>
    ACTION: Awaiting human response (asked <N> min ago)
    PROGRESS: Story N/Total | Epic N | Review pass: N/3
    ```

    **[NEEDS-HUMAN]** / **[HALTED]**:
    ```
    [NEEDS-HUMAN] Story X.X | <step>
    SAW: <what went wrong>
    ACTION: Loop paused — awaiting human input
    PROGRESS: Story N/Total | Epic N | Review pass: N/3
    ```

    Format rules:
    - Line 1: ALWAYS [STATUS] + story + key info
    - SAW: ALWAYS one line, max ~80 chars
    - ACTION: ALWAYS one line
    - PROGRESS: ALWAYS same format
    - STATS: ONLY on [DONE] and [TRANSITION]
    - No emojis except on problems
    - No predictions, no token counts, no commentary

  4d. CYCLE VERIFICATION — Before exiting, confirm ALL of these:
    state.lastUpdated written with current timestamp
    Activity log entry appended (CRON_FIRE or CRON_SKIP)
    Report sent to human via mcp__nanoclaw__send_message
    All state changes from this cycle written to state file
</cron-step>
```

---

## How to Control the Loop

Tell your NanoClaw agent any of these in chat:

- **"Pause the loop"** — Sets status to "paused", task keeps firing but exits immediately
- **"Resume the loop"** — Sets status back to "running"
- **"Skip this story"** — Marks story as skipped in state, advances to next
- **"Stop the loop"** — Sets status to "stopped", pauses the scheduled task
- **"Loop status"** — Shows current story, step, test results, review loop count, queue remaining
- **"Switch to conservative"** — Changes escalation mode, pauses on any failure
- **"Switch to autonomous"** — Returns to default autonomous failure handling
- **"Run retrospective"** — Pauses the loop and runs `/bmad-bmm-retrospective` for the current epic
- **"Force advance"** — Overrides quality gate and advances to next story (use with caution)
- **"Quarantine story X.X"** — Removes a problematic story from the queue:
  1. Mark story as "quarantined" in state (distinct from "skipped" or "done")
  2. Log: `QUARANTINE | story:X.X | reason:<human's reason>`
  3. Continue with next story
  4. Quarantined stories appear in daily summary
  5. Human can later say **"Unquarantine story X.X"** to restore it

---

