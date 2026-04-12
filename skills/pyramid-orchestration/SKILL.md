---
name: pyramid-orchestration
description: >
  Use when executing a multi-task plan that would require 4 or more subagents.
  Organizes work into a pyramid: Main Agent dispatches Worker Agents, each Worker
  dispatches its own subagents. Workers return only brief summaries to Main,
  keeping the orchestrator's context lightweight. Use this instead of dispatching
  all subagents directly from a single agent when task count is high.
---

# Pyramid Orchestration

Hierarchical agent delegation for complex tasks. Instead of one agent managing all subagents directly (flat), add a Worker layer that absorbs the detailed results and returns only summaries.

```
Flat (context bloat):          Pyramid (context efficient):

Main ← full ← Sub1            Main ← summary ← Worker A ← full ← Sub1
     ← full ← Sub2                                       ← full ← Sub2
     ← full ← Sub3                  ← summary ← Worker B ← full ← Sub3
     ← full ← Sub4                                       ← full ← Sub4
     ← full ← Sub5                  ← summary ← Worker C ← full ← Sub5
     ← full ← Sub6                                       ← full ← Sub6
= ~29k tokens in Main         = ~5.6k tokens in Main
```

## When to Activate

| Task Count | Mode | Reason |
|------------|------|--------|
| 1-3 subagents | Flat (skip this skill) | Overhead of Workers not worth it |
| 4-8 subagents | Pyramid with 2-3 Workers | Meaningful context savings |
| 9+ subagents | Pyramid with 3-5+ Workers | Essential to avoid context wall |

If you have a plan file, count the tasks. If the total expected subagent dispatches (implementers + reviewers + testers) exceeds 3, use pyramid mode.

## The Two Rules

1. **Main Agent stays light.** It interacts with the user, reads file paths, dispatches Workers, collects summaries. It does NOT read code, write code, run tests, or perform reviews.
2. **Workers return summaries, not details.** All detailed output (code, diffs, review comments, test results) goes to files and git commits. The Worker's response to Main is under 100 words.

## Orchestration Protocol

### Step 1: Assess the Plan

Read the plan file. Count tasks. Determine grouping.

If task count ≤ 3: use flat delegation, exit this skill.

If task count > 3: continue with pyramid mode.

### Step 2: Group Tasks

Assign tasks to Workers using these heuristics (in priority order):

1. **Dependency coupling**: tasks that depend on each other go to the same Worker (avoids cross-Worker coordination)
2. **Module locality**: tasks touching the same files/directory go to the same Worker
3. **Phase alignment**: if the plan has phases, each phase or sub-phase maps to a Worker
4. **Size limit**: 2-4 tasks per Worker

### Step 3: Dispatch Workers

For each group, dispatch a Worker Agent with this prompt structure:

```
You are a Worker Agent. Your assignment:

Plan file: {plan_path}
Your tasks: {task_numbers_and_descriptions}

Context files you should read:
- {relevant_file_1}
- {relevant_file_2}

Instructions:
1. Read the plan file and your assigned tasks
2. Execute each task — use any available skills, dispatch subagents as needed
3. Commit your work with descriptive messages after each logical unit
4. Write any detailed output (review results, test reports) to files

When done, respond with ONLY:
- Status: completed | partial | blocked
- Tasks completed: [list]
- Commits: [hashes]
- Issues: [any blockers, or "none"]

Keep your response under 100 words. Everything else is in the repo.
```

**Dispatch rules:**
- Workers with no dependencies between them: dispatch in parallel
- Workers that depend on a previous Worker's output: dispatch sequentially
- Never dispatch more than 5 Workers in parallel (diminishing returns, resource contention)

### Step 4: Collect and Assess

After each Worker returns:
1. Record the summary (status, commits, issues)
2. If status is "completed": move to the next Worker or phase
3. If status is "partial": read git log to assess what was done, dispatch a new Worker for remaining tasks
4. If status is "blocked": evaluate the blocker — dispatch a fix Worker or ask the user

### Step 5: Review Cycle

After all implementation Workers complete:

Dispatch a review Worker:
```
You are a Review Worker. Your assignment:

Review all changes since commit {baseline_commit}.
Plan file: {plan_path}

Instructions:
1. Read the plan to understand what was intended
2. Review the implementation against the plan
3. Check code quality, test coverage, and correctness
4. Write a review document to {review_output_path}
5. Commit the review document

Respond with ONLY:
- Issues found: [count]
- Severity: [critical/moderate/minor]
- Review file: [path]

Keep your response under 50 words.
```

If issues are found, dispatch a fix Worker:
```
You are a Fix Worker. Your assignment:

Review document: {review_file_path}
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

Repeat the review-fix cycle until the review Worker reports zero critical issues, or escalate to the user.

### Step 6: Verification

Dispatch a verification Worker:
```
You are a Verification Worker. Your assignment:

Plan file: {plan_path}
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

### Step 7: Report to User

Summarize the full execution to the user:
- How many tasks were completed
- Key commit hashes or branch name
- Any remaining issues
- Verification result

## Failure Recovery

If Main's context gets compacted mid-execution:

1. Read the plan file (you know the path)
2. Run `git log --oneline -30` to see what Workers already committed
3. Compare commits against the plan to identify remaining tasks
4. Resume from where the last Worker left off

No special progress files needed — the plan file + git history is sufficient.

## What Main Agent's Context Looks Like

At the end of a 12-task execution with 4 Workers:

```
Main context contains:
  User conversation              ~3,000 tokens
  This skill                     ~2,000 tokens
  Plan file path + task count       ~100 tokens
  Worker A summary                  ~150 tokens
  Worker B summary                  ~150 tokens
  Worker C summary                  ~150 tokens
  Worker D summary                  ~150 tokens
  Review Worker summary             ~100 tokens
  Verification Worker summary       ~100 tokens
  ────────────────────────────
  Total                          ~5,900 tokens
```

Compare to flat mode: ~29,000+ tokens. That is the value of the pyramid.

## Anti-Patterns

| Pattern | Why It Breaks the Pyramid |
|---------|--------------------------|
| Main reads code files "to check" | Pulls detailed content into Main's context |
| Main asks Worker "explain what you did" | Worker returns a long explanation instead of summary |
| Worker returns full diff or test output | Detailed content belongs in files, not the response |
| Main dispatches all tasks to one Worker | That Worker becomes the new bottleneck (same as flat mode) |
| Skipping review because "Workers already tested" | Workers test their own work; independent review catches different bugs |

## Integration

This skill is standalone. It works with whatever skills and tools are available in the environment:
- If TDD skills are available, Workers can use them
- If code review skills are available, the review Worker can use them
- If no specialized skills are available, Workers use their default capabilities

The pyramid is an orchestration pattern, not a replacement for execution skills.
