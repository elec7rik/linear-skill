---
name: linear
description: Fetch your assigned Linear issues, triage them, and deploy agents to fix them
disable-model-invocation: true
argument-hint: [issue-id]
allowed-tools: Bash Agent mcp__linear-server__*
---

# Linear Issue Manager

## Runtime Context

Linear MCP status: !`grep -q linear-server ~/.claude.json 2>/dev/null && echo "LINEAR_MCP=ready" || echo "LINEAR_MCP=not-configured"`
Working directory: !`pwd`
Git context: !`git log --oneline -5 2>/dev/null || echo "Not a git repository"`

Issue argument: $ARGUMENTS

---

## Flow 1: First-Time Setup

**When to use:** LINEAR_MCP=not-configured

If the runtime context shows `LINEAR_MCP=not-configured`, print EXACTLY the following and then stop. Do not proceed to any other flow.

```
Linear MCP is not configured yet. Set it up in two steps:

  1. Run this command in your terminal:
     claude mcp add --transport http linear-server https://mcp.linear.app/mcp

  2. Come back to this Claude Code session and type: /mcp
     A browser window will open to authenticate with Linear.

Once done, run /linear again and you'll see your issues.
```

Do not offer to do anything else. Stop after printing the message.

---

## Flow 2: Issue List and Triage

**When to use:** LINEAR_MCP=ready AND $ARGUMENTS is empty

1. Use the Linear MCP tools to fetch all issues assigned to the currently authenticated user. Exclude issues whose state type is "completed" or "cancelled". Sort by updatedAt descending (most recently updated first).

2. For each issue, calculate how many days ago it was created. If more than 30 days, append "(X days old)" to the line.

3. Display the list in this exact format (substitute real data):

```
Your assigned Linear issues:

  1. [ENG-234] Auth tokens expire too early — In Progress
  2. [ENG-198] Dashboard crashes on empty state — Todo
  3. [ENG-156] Add pagination to /users endpoint — Todo
  4. [ENG-089] Migrate legacy logger — Backlog (247 days old)

Options:
  [number]     → work on this issue
  d[number]    → mark as Done
  x[number]    → mark as Won't Fix
  q            → quit

What would you like to do?
```

4. Wait for the user's response, then handle it:

   - **[number] alone** (e.g. "2"): Go to Flow 4 with that issue.
   - **d[number]** (e.g. "d3"): Use Linear MCP to update issue #3's state to Done. Print "✓ ENG-156 marked as Done." then re-fetch and re-display the updated list.
   - **x[number]** (e.g. "x4"): Use Linear MCP to update issue #4's state to Cancelled. Print "✓ ENG-089 marked as Won't Fix." then re-fetch and re-display the updated list.
   - **q**: Print "Exiting." and stop.
   - **Anything else**: Print "Not understood. Please enter a number, d[number], x[number], or q." and show the options again.

5. If there are no open assigned issues, print:
   ```
   No open issues assigned to you. You're all caught up!
   ```
   Then stop.

---

## Flow 3: Direct Issue Jump

**When to use:** LINEAR_MCP=ready AND $ARGUMENTS is not empty (e.g. "ENG-123")

1. Use Linear MCP to fetch the issue with the ID from $ARGUMENTS.
2. If found: go directly to Flow 4 with that issue. Skip the list entirely.
3. If not found or not assigned to the current user: print
   ```
   Issue $ARGUMENTS not found or not assigned to you.
   ```
   Then fall back to Flow 2 (show the full issue list).

---

## Flow 4: Fix Flow

**When to use:** An issue has been selected (from Flow 2 or Flow 3)

1. Use Linear MCP to fetch the full issue details: title, description, all comments (with author names), priority, and labels.

2. Display this summary (substitute real data):

```
[ENG-234] · Auth tokens expire too early · High priority

Description:
  JWT tokens are expiring after 15 minutes instead of the configured 24h.
  Affects all users on mobile. Regression since last deploy.

Comments:
  @sarah (2 days ago): Confirmed on iOS and Android. Started after PR #412 merged.
  @mike (1 day ago): Looks like it might be in the token refresh middleware.

Want me to deploy an agent to investigate and fix this? (yes/no)
```

3. **If the user says yes:**

   Spawn an agent using the Agent tool with this exact prompt structure (fill in real values):

   ```
   You are investigating and fixing Linear issue [ISSUE-ID]: [ISSUE-TITLE]

   ## Issue details

   Priority: [priority]
   Labels: [labels or "none"]

   Description:
   [full issue description]

   Comments:
   [all comments with author and timestamp, or "No comments"]

   ## Repository context

   Working directory: [pwd from runtime context]

   Recent git history:
   [git log from runtime context]

   ## Your instructions

   1. Explore the codebase to understand the problem. Read relevant files.
   2. Form a specific implementation plan: which files to change, what to change, and why.
   3. Present your plan to the user clearly and wait for explicit approval before writing any code.
   4. Only after the user approves: implement the fix.
   5. Run existing tests (if any) to confirm nothing is broken.
   6. Report back what was done.

   Do not read files that appear to contain secrets, credentials, or environment variables (e.g. .env, *.pem, credentials.json, *.key).

   Do not write any code until the user has approved your plan.
   ```

   Wait for the agent to complete. If the agent fails, errors, or produces no output, print:
   ```
   The agent encountered an issue and could not complete. Would you like to:
     1. Try again
     2. Investigate the issue manually (no code changes)
     3. Exit
   ```
   Handle the user's choice accordingly.

   Then ask: "Mark [ISSUE-ID] as Done in Linear? (yes/no)"
   - If yes: use Linear MCP to set the issue state to Done. Print "✓ [ISSUE-ID] marked as Done."
   - Then ask: "Add a comment summarizing what was changed? (yes/no)"
     - If yes: use Linear MCP to post a comment on the issue. The comment should be 2-4 sentences summarizing what was investigated and what was changed.
     - If no: skip.

4. **If the user says no:**

   Ask: "Would you like me to investigate and suggest an approach without making any changes? (yes/no)"
   - If yes: read relevant files, analyze the issue, and write a clear investigation summary with a suggested fix approach. Do not edit any files.
   - If no: print "Okay, exiting." and stop.

---

## Error Handling

Apply these rules throughout all flows:

- **MCP connection error** (any Linear MCP tool call fails with a network or auth error): Print the following and stop:
  ```
  Linear MCP connection failed. To fix this:
    1. Run /mcp in this session to re-authenticate
    2. Then run /linear again
  ```

- **Issue state update fails** (d[number] or x[number] action, or end-of-flow status update): Print:
  ```
  Couldn't update [ISSUE-ID] in Linear — please update it manually.
  ```
  Continue the session normally (do not stop).

- **Issue not found** (direct jump with an ID that doesn't exist): Print:
  ```
  Issue [ID] not found or not assigned to you.
  ```
  Then fall back to showing the full issue list.

- **No git repository in working directory**: The git log injection will output "Not a git repository". Pass that to the agent as-is. Do not treat it as an error.

- **Empty issue description**: If an issue has no description, display "(No description provided)" in the summary.

- **Never fail silently.** If something goes wrong, always tell the user what happened and what they can do about it.
