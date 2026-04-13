---
name: pyramid-orchestration
description: >
  Use when executing a multi-task plan with 4 or more delegated work units and
  keeping the main agent's context small matters.
---

# Pyramid Orchestration

Hierarchical orchestration for complex tasks using a file-backed mailbox. Keep
the pyramid mental model, but make handoffs durable: Main is the only agent
that dispatches other agents, and detailed work flows through files instead of
long chat replies.

```
Flat (context bloat):              Pyramid (context efficient):

Main ← full ← Job1                Main ← summary ← Worker A
     ← full ← Job2                     request file → Child A-01
     ← full ← Job3                     result file ← Child A-01
     ← full ← Job4                     summary ← Continuation Worker A
     ← full ← Job5                Main ← summary ← Worker B
     ← full ← Job6                     ...
= ~29k tokens in Main             = ~5.6k tokens in Main
```

## When to Activate

| Task Count | Mode | Reason |
|------------|------|--------|
| 1-3 delegated work units | Flat (skip this skill) | Overhead of Workers not worth it |
| 4-8 delegated work units | Pyramid with 2-3 Workers | Meaningful context savings |
| 9+ delegated work units | Pyramid with 3-5+ Workers | Essential to avoid context wall |

If you have a plan file, count the tasks. If the total expected delegated work
units (implementation, review, verification, or worker-requested child jobs)
exceeds 3, use pyramid mode.

## The Two Rules

1. **Main Agent stays light.** It interacts with the user, reads file paths,
   dispatches Workers and child agents, and collects summaries. It does NOT
   read code, write code, run tests, or perform reviews.
2. **Detailed output belongs in files.** All detailed output (code, diffs,
   review comments, test results) goes to files and git commits. Worker and
   child responses stay short.

## Runtime Model

1. **Main is the sole dispatcher.** Only Main creates Worker agents and child
   agents.
2. **Workers never spawn agents directly.** If a Worker needs deeper
   delegation, it writes a request file.
3. **Children write files, not essays.** Child agents put detailed output in
   result files and return a tiny status line.
4. **Workers stop at durable handoff points.** No live waiting loops. A Worker
   writes requests, returns `awaiting-results`, and Main later dispatches a
   continuation Worker.

## Run Directory

Create a run directory under the repo root:

```
.pyramid/runs/<run-id>/
├── manifest.md
├── requests/
├── results/
└── summaries/
```

Use it like this:

- `manifest.md`: plan path, Worker groups, current phase, and next action
- `requests/<request-id>.md`: a Worker's request for delegated child work
- `results/<request-id>.md`: detailed child output
- `summaries/<worker-id>.md`: Worker-written summary artifacts for later lookup

## Request File Contract

Workers request deeper delegation by writing a Markdown file with frontmatter:

```markdown
---
request_id: worker-a-01
worker_id: worker-a
status: requested
kind: implement
title: Build parser edge-case tests
result_path: .pyramid/runs/<run-id>/results/worker-a-01.md
---

Context files:
- path/to/file

Exact assignment:
- Write the failing tests for ...
- Run ...
- Record commands and results in the result file
```

Required fields:

- `request_id`
- `worker_id`
- `status`: `requested`, `dispatched`, `completed`, `failed`, or `cancelled`
- `kind`: `implement`, `review`, `verify`, `research`, or another brief label
- `title`
- `result_path`

Main updates the status as the request moves through the queue. Child agents
write the result file and Main marks the request complete or failed.

## Orchestration Protocol

### Step 1: Assess the Plan

Read the plan file. Count tasks. Determine grouping.

If task count ≤ 3: use flat delegation, exit this skill.

If task count > 3: continue with pyramid mode.

### Step 2: Group Tasks

Assign tasks to Workers using these heuristics, in priority order:

1. **Dependency coupling**: tasks that depend on each other go to the same
   Worker
2. **Module locality**: tasks touching the same files or directory go to the
   same Worker
3. **Phase alignment**: if the plan has phases, each phase or sub-phase maps to
   a Worker
4. **Size limit**: 2-4 tasks per Worker

### Step 3: Create the Run Directory

Before dispatching anyone:

1. Create `.pyramid/runs/<run-id>/`
2. Write `manifest.md` with:
   - plan path
   - grouped Worker assignments
   - current phase
   - open request ids
   - completed summary files
3. Use the same run directory for implementation, review, and verification

### Step 4: Dispatch Workers

For each group, dispatch a Worker Agent with this prompt structure:

```
You are a Worker Agent. Your assignment:

Plan file: {plan_path}
Run directory: .pyramid/runs/{run_id}
Worker id: {worker_id}
Your tasks: {task_numbers_and_descriptions}

Context files you should read:
- {relevant_file_1}
- {relevant_file_2}

Instructions:
1. Read the plan file and your assigned tasks
2. Execute what you can directly
3. If you need more delegated work, write request files in
   `.pyramid/runs/{run_id}/requests/`
4. Stop at a durable handoff point if you need Main to fulfill those requests
5. Commit your work with descriptive messages after each logical unit
6. Write any detailed output (review results, test reports) to files

When done, respond with ONLY:
- Status: completed | awaiting-results | partial | blocked
- Tasks completed: [list]
- Requests: [request ids, or "none"]
- Commits: [hashes]
- Issues: [any blockers, or "none"]

Keep your response under 100 words. Everything else is in the repo.
```

**Dispatch rules:**

- Workers with no dependencies between them: dispatch in parallel
- Workers that depend on a previous Worker's output: dispatch sequentially
- Never dispatch more than 5 Workers in parallel

### Step 5: Service the Request Queue

When a Worker returns `awaiting-results`:

1. Read `.pyramid/runs/<run-id>/requests/` for files with `status: requested`
2. Update each selected request to `status: dispatched`
3. Dispatch a child agent for each request
4. Give the child:
   - the request file path
   - the plan file path
   - the target `result_path`
5. Require the child to:
   - write detailed output to the result file
   - avoid long chat replies
   - report only `request_id`, `status`, and `result_path`
6. Mark the request `completed` or `failed` after the child finishes

### Step 6: Continue or Close Workers

After a Worker or child returns:

1. Record the summary in `manifest.md`
2. If a Worker status is `completed`: move to the next Worker or phase
3. If a Worker status is `awaiting-results`: dispatch a continuation Worker
   after the requested result files exist
4. If a Worker status is `partial`: inspect git history, update the manifest,
   and dispatch a follow-up Worker for the remaining tasks
5. If a Worker status is `blocked`: evaluate the blocker, dispatch a fix path,
   or ask the user

Continuation Workers get:

- the same `plan_path`
- the same `worker_id`
- the same `run_id`
- the paths to completed result files they should absorb
- the original task slice plus any remaining tasks

### Step 7: Review Cycle

After all implementation Workers complete, dispatch a review Worker:

```
You are a Review Worker. Your assignment:

Review all changes since commit {baseline_commit}.
Plan file: {plan_path}
Run directory: .pyramid/runs/{run_id}

Instructions:
1. Read the plan to understand what was intended
2. Review the implementation against the plan
3. Check code quality, test coverage, and correctness
4. Write a review document to {review_output_path}
5. Commit the review document

Respond with ONLY:
- Issues found: [count]
- Severity: [critical | moderate | minor]
- Review file: [path]

Keep your response under 50 words.
```

If issues are found, dispatch a fix Worker:

```
You are a Fix Worker. Your assignment:

Review document: {review_file_path}
Run directory: .pyramid/runs/{run_id}
Fix all issues listed in the review.

Instructions:
1. Read the review document
2. Fix each issue
3. Commit fixes with descriptive messages

Respond with ONLY:
- Issues fixed: [count]
- Commits: [hashes]
- Remaining: [any unfixed issues, or "none"]

Keep your response under 50 words.
```

Review and fix Workers may also use the mailbox protocol if they need deeper
delegated work. Main still services every request.

### Step 8: Verification

Dispatch a verification Worker:

```
You are a Verification Worker. Your assignment:

Plan file: {plan_path}
Run directory: .pyramid/runs/{run_id}
Verify that all tasks in the plan are correctly implemented.

Instructions:
1. Read the plan
2. For each task, verify the implementation exists and works
3. Run relevant tests
4. Write verification results to {verification_output_path}

Respond with ONLY:
- Result: pass | fail
- Tasks verified: [count]
- Failures: [list, or "none"]

Keep your response under 50 words.
```

Verification Workers may also write request files and stop at
`awaiting-results` if Main must fulfill delegated checks.

### Step 9: Report to User

Summarize the full execution to the user:

- how many tasks were completed
- key commit hashes or branch name
- any remaining issues
- verification result

## Failure Recovery

If Main's context gets compacted mid-execution:

1. Read the plan file and `.pyramid/runs/<run-id>/manifest.md`
2. Inspect `requests/` to find outstanding or failed request ids
3. Inspect `results/` and `summaries/` to see what was already produced
4. Run `git log --oneline -30` to correlate commits with Worker summaries
5. Resume from the last incomplete Worker or request id

The run directory is the durable execution log. Git history remains the
secondary source of truth.

## What Main Agent's Context Looks Like

At the end of a 12-task execution with 4 Workers:

```
Main context contains:
  User conversation              ~3,000 tokens
  This skill                     ~2,000 tokens
  Plan file path + task count       ~100 tokens
  Worker A summary                  ~150 tokens
  Request ids + result paths        ~200 tokens
  Worker B summary                  ~150 tokens
  Worker C summary                  ~150 tokens
  Worker D summary                  ~150 tokens
  Review Worker summary             ~100 tokens
  Verification Worker summary       ~100 tokens
  ────────────────────────────
  Total                          ~6,000 tokens
```

Compare to flat mode: ~29,000+ tokens. That is the value of the pyramid.

## Anti-Patterns

| Pattern | Why It Breaks the Pyramid |
|---------|--------------------------|
| Main reads code files "to check" | Pulls detailed content into Main's context |
| Main asks Worker "explain what you did" | Worker returns a long explanation instead of summary |
| Worker returns full diff or test output | Detailed content belongs in files, not the response |
| Worker says it will spawn its own subagent | This protocol assumes Main is the only dispatcher |
| Workers wait in a live loop for result files | Durable handoffs are safer than blocking loops |
| Request files omit `status` or `result_path` | Main cannot safely route or recover the work |
| Main dispatches all tasks to one Worker | That Worker becomes the new bottleneck |
| Skipping review because "Workers already tested" | Workers test their own work; independent review catches different bugs |

## Integration

This skill is standalone. It works with whatever skills and tools are available
in the environment:

- If TDD skills are available, Workers can use them
- If code review skills are available, review Workers can use them
- If no specialized skills are available, Workers use their default
  capabilities

The pyramid is an orchestration pattern, not a replacement for execution
skills.
