# OpenClaw Browser Automation

OpenClaw includes a built-in browser (Chromium via CDP) that lets the agent automate web panels, forms, and interfaces that have no public API.

Official docs: https://docs.openclaw.ai
Browser tool reference: see OpenClaw tool documentation for `browser_*` actions.

---

## When to use browser automation

**Use the browser when:**
- The user needs to automate a private web panel (booking system, CRM, ticketing, intranet)
- No stable public API or n8n integration exists for that system
- The task requires filling forms, clicking, or reading data from a web interface

**Do not use the browser when:**
- A direct API or existing integration exists — always prefer API over browser automation
- The task only analyzes data already available as text/JSON/CSV in memory or files

---

## Browser modes

Choose the mode based on where OpenClaw runs and what the target system requires.

| Mode | When to use |
|------|-------------|
| `openclaw` | Default. Isolated Chromium instance. Use for server-side automation (VPS/Docker) and when login happens fresh each time. |
| `chrome` | Connect to the user's existing Chrome session via extension. Use when 2FA, SSO, or existing login cookies are required. |
| `remote` | Connect to an external CDP endpoint (Browserless, headless Chrome service). Use for scaling parallel browser instances. |

---

## Core principles

### Always use snapshots

Before clicking or typing, request a page snapshot. The snapshot:
- Numbers all interactive elements (buttons, links, inputs)
- Provides text labels and roles for each element
- Is the only reliable way to identify elements — never guess CSS selectors

```
Snapshot element example:
  7: button "Add Event" [role=button]
  8: input "Event name" [role=textbox]
  9: input "Date" [role=textbox, type=date]
```

### Reference elements by number

Always use the element number from the snapshot, not a guessed selector:

```
Correct:  click element 7 ("Add Event" button)
Wrong:    click element with class ".btn-primary"
```

### Regenerate snapshot after page changes

If the page layout changes after a click, pop-up, error, or navigation — generate a new snapshot before continuing.

---

## Standard action sequence

For any browser automation task:

1. **Launch browser** in the appropriate mode
2. **Navigate** to the target URL
3. **Take snapshot** of the loaded page
4. **Log in** if required (find login fields in snapshot → fill → submit)
5. **Take snapshot** after login
6. **Find target action** in snapshot (add, edit, search)
7. **Click target** using element number
8. **Take snapshot** of the resulting form/page
9. **Fill fields** by mapping user data to numbered form elements
10. **Submit** using the save/confirm button number
11. **Take screenshot** of the result page
12. **Return structured data** (IDs, status, links) for use by other tools or n8n

---

## Typical use case: adding a record in a web panel

When a user wants to add a concert, booking, order, or any record to a panel without API:

```
1. Collect required parameters from user:
   - name / title
   - date and time
   - location or venue
   - relevant IDs or codes
   - any other fields shown in the form

2. Launch browser in openclaw or remote mode (server) 
   or chrome mode (if existing session needed)

3. Navigate to the panel URL
   → Take snapshot

4. Log in if needed
   → Take snapshot after login

5. Find and click "Add / New / Create" button (by number in snapshot)
   → Take snapshot of the form

6. Map user data to form fields (by number and label in snapshot)
   → Fill all required fields

7. Click Save / Submit button (by number in snapshot)
   → Take snapshot of confirmation

8. Read result: ID, status, confirmation message
   → Return as structured output for n8n or other systems
```

---

## Updating an existing record

```
1. Navigate to list view
   → Take snapshot

2. Identify the target row by name, date, or ID combination
   → Click the Edit button for that row (by number in snapshot)
   → Take snapshot of edit form

3. Update the required fields

4. Save
   → Take screenshot of updated view
   → Return updated status and key fields
```

---

## Multi-step tasks and multi-tab sessions

- When comparing data across multiple systems, use multiple browser tabs in one session
- Maintain session state — do not restart the browser between related steps
- For batch operations (adding many records), iterate through the list in one session
- Watch for rate limits or anti-bot detection — slow down if the panel starts blocking

---

## Output format for n8n integration

When browser automation is triggered by n8n via `/hooks/agent`, return structured output:

```json
{
  "status": "success",
  "record_id": "EVT-12345",
  "url": "https://panel.example.com/events/12345",
  "screenshot": "available",
  "fields_set": {
    "name": "Summer Concert",
    "date": "2026-07-15",
    "venue": "Main Hall"
  }
}
```

This allows n8n to continue the workflow using the created record's ID or link.

---

## Config: enabling browser tools in OpenClaw

Browser automation requires enabling the browser tool in the agent's configuration:

```json5
agents: {
  defaults: {
    tools: {
      allow: ["browser_navigate", "browser_snapshot", "browser_click",
              "browser_type", "browser_screenshot"],
    },
  },
},
```

Or enable all tools (less secure — not recommended for production):

```json5
agents: {
  defaults: {
    tools: {
      allow: ["*"],
    },
  },
},
```

**Security note:** Browser tools give the agent significant access to web interfaces. Limit `tools.allow` to the specific browser actions needed. Do not enable browser tools for agents that should only answer questions or draft text.

---

## Error handling in browser sessions

If the page shows an unexpected dialog, alert, or error:
1. Take a new snapshot
2. Read the message content
3. Decide: retry, refresh, close dialog, or stop and report
4. After 3 failed attempts on the same step — stop and ask the user

Do not loop indefinitely on browser errors. Report clearly what was completed and what failed.
