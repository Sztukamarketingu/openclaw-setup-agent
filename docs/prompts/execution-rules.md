# Execution Rules — Failure Handling & Runtime Limits

These rules apply to the agent during all deployment and automation phases (Phase 3, browser automation, n8n integration tasks).

The agent must follow these rules at all times during task execution. They are not optional.

---

## Rules

**1. Failure limit: 3 attempts maximum**
- If a task step fails 3 times, STOP immediately
- Do not retry automatically beyond 3 attempts
- After the third failure:
  - Report the error clearly (exact message, step that failed)
  - Explain likely causes
  - Suggest possible fixes
  - Ask for user input before continuing

**2. Retry logic**
- Only retry if the failure is potentially recoverable (network timeout, temporary error)
- Do not retry identical actions blindly — adjust the approach if possible (different parameter, different method)
- Apply a short delay between retries (start with 2s, then 5s, then 10s)

**3. Time limit: 10 minutes per task**
- Maximum runtime for any single task: 10 minutes
- If a task exceeds this limit:
  - Stop execution immediately
  - Report what was completed before the timeout
  - Explain why it may be taking longer than expected
  - Ask the user whether to continue, abort, or try a different approach

**4. Infinite loop protection**
- Detect repeated or cyclical behavior (same action, same result, no progress)
- If the same action repeats 3+ times without progress → STOP
- Flag this as a potential infinite loop
- Report the repeating pattern to the user

**5. Resource awareness**
- Avoid long-running polling loops — use event-driven approaches when available
- Prefer efficient, targeted commands over broad scans
- Do not leave browser sessions or SSH connections open after task completion

**6. Override rules**
- Only exceed these limits if the user explicitly instructs to do so in this session
- Always confirm before running tasks expected to take longer than 5 minutes

---

## Required output format for any failed task

When a task stops due to failure, report:

```
Task: [name of the task]
Attempts: [number]
Time spent: [duration]
Status: FAILED / STOPPED

Error: [exact error message]
Step that failed: [which step in the sequence]

Likely causes:
- [cause 1]
- [cause 2]

Suggested fixes:
- [fix 1]
- [fix 2]

Next step: Waiting for your input before continuing.
```

---

## Application during deployment (Phase 3)

During VPS deployment, apply these rules to each step:

| Step | Retry? | Max attempts | Timeout |
|------|--------|--------------|---------|
| Hostinger API call | Yes (network) | 3 | 30s |
| SSH command | Yes (connection) | 3 | 60s |
| SCP file upload | Yes (connection) | 3 | 120s |
| Gateway restart | Yes | 2 | 30s |
| Channel status probe | Yes | 3 | 30s |

If any step hits its limit: stop, report, ask for input.

---

## Application during browser automation

During browser sessions, apply these rules per action:

- Navigation timeout: 30 seconds
- Element interaction (click, type): 3 attempts with snapshot regeneration between each
- Form submission: 2 attempts
- If page does not load or element not found after 3 attempts: stop and report

Never loop through a failed page action more than 3 times without stopping to report.
