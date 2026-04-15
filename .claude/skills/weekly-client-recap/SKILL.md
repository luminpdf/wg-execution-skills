---
name: weekly-client-recap
description: >
  Generate a strategic weekly recap Slack message for internal stakeholders (CEO, CTO, Head of Revenue).
  Covers past-week experiment velocity, learning log metrics, and upcoming releases from Linear.
  Trigger when the user says "weekly recap", "client recap", "generate weekly update",
  "weekly client update", "stakeholder recap", "weekly report", or any variation of
  wanting a high-level weekly summary for leadership.
user-invocable: true
---

# Weekly Client Recap

Generates a strategic weekly recap message formatted in Slack mrkdwn, ready to copy and paste. Covers past-week experiment velocity, learning log metrics, and upcoming releases pulled from Linear.

## Data sources

| Source | Data source ID / reference |
|--------|---------------------------|
| Weekly Velocity Tracker | `collection://6a5b765d-95ce-4822-815e-d3a7c63b0083` |
| Notion Experiments DB | `collection://d198507c-8621-47aa-a6df-c726597b2575` |
| Linear (team: Web Growth) | `mcp__linear__list_issues` / `mcp__linear__get_issue` |

---

## Step 1 — Determine the week range

Calculate the current ISO week (Monday–Sunday) using today's date from context.

- **Week start:** Monday of the current week
- **Week end:** Sunday of the current week
- Format dates as `DD MMM` for display (e.g., `31 Mar – 06 Apr 2026`)

---

## Step 2 — Read velocity metrics from Weekly Velocity Tracker

Use `mcp__notion__notion-fetch` on the Weekly Velocity Tracker DB (`collection://6a5b765d-95ce-4822-815e-d3a7c63b0083`).

Find the row whose **Week Start** date falls within the current ISO week (Monday–Sunday from Step 1).

If a matching row is found, read all columns directly — no on-the-fly computation needed:
- **Experiments Released** (number + ticket links)
- **Features Released** (number + ticket links)
- **Ready to Launch** (number + ticket links)
- **Finalized** (number + ticket links)
- **Learning Logs Ready** (number)
- **Learning Logs On-Time** (number)
- **Lessons Learnt** (free text from DL)

Compute the on-time percentage: `Learning Logs On-Time / Learning Logs Ready * 100` (show `0%` if Learning Logs Ready is 0).

If **no row exists** for the current week, output `[Weekly Velocity Tracker row not found for this week — ask TL/DL to create it]` for the velocity section and continue with Steps 3–5 as normal.

---

## Step 3 — Query Linear for upcoming releases

Call `mcp__linear__list_issues` filtered to:
- Team: `Web Growth`
- Statuses: `Ready to Launch`, `In Review`, `QA`

For each ticket returned, call `mcp__linear__get_issue` to fetch the full title and description.

Collect:
- Ticket ID (e.g., `LWG-42`) and its Linear URL
- Title
- Description (for summarization in Step 4)

Sort by status priority: `Ready to Launch` first, then `In Review`, then `QA`.

---

## Step 4 — Generate 1-sentence user-value summaries

For each Linear ticket from Step 3, synthesize a **single concise sentence** from the title + description:
- Frame as the **customer benefit or business outcome**, not a feature spec
- Bad: "Adds a banner to the upgrade modal"
- Good: "Surfaces the value of upgrading at the moment users hit a PDF limit, reducing friction in the decision to convert"

If a ticket has no description, use the title to infer the most likely user benefit.

---

## Step 5 — Compose the Slack message

Use `mcp__plugin_slack_slack__slack_send_message_draft` to save the message as a draft in Jean's DM (`U0140HS47C6`) for review before sending. Do not send directly. Return the draft link so Jean can review and send it herself.

Use this structure exactly:

```
*Web Growth — Week of [Mon DD MMM – Sun DD MMM YYYY]*

[1 short human-sounding intro sentence. Keep it punchy and specific — reference the week's defining move or momentum signal. E.g., "Here's our week 1 Q2 snapshot — infrastructure cleared, 5 experiments queuing up." or "Quick recap: 3 experiments shipped, first signals incoming."]

[2–3 sentence strategic narrative. Cover: (1) experiments shipped and their business purpose, (2) any infrastructure or non-experiment work — always frame infra work as a strategic enabler tied to an OKR/KR (e.g., "prerequisite for O2-KR1: 5 experiments/week"), (3) any parallel strategic deliverables in progress. Frame around outcomes, not task completion.]

*This week's velocity*
• Experiments released: [N]
• Features released: [N]
• Experiments finalized: [N]
• Learning logs ready: [N] ([X]% on-time within 3 days)
  [If learning logs = 0 AND it is early in the quarter or pipeline is in build-out phase, add:]
  _([reason — e.g., "Q2 pipeline week 1 — first experiments still running; decision signals and logs expected in ~2 weeks as tests reach significance"])_

*Key learnings*
• <URL|Experiment Name> — [1-sentence takeaway if inferrable from context, else omit]
• ...
[Or: [data unavailable — add "Learning Log URL" property to Experiments DB]]

*Lessons from the team*
[Nam's text from "Lessons Learnt" column in the Weekly Velocity Tracker. Omit this entire section if the field is empty.]

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

*Coming this week — [N] experiments targeting [OKR name]*

[1–2 sentence bridge explicitly naming which OKR and KRs the upcoming releases collectively target — e.g., "This week's portfolio makes a simultaneous push across all three O1 revenue KRs: reducing friction in the trial conversion path, recovering churners before they exit, and deepening Desktop App engagement at the highest-intent moment."]

[Group experiments by OKR KR. For every ticket, use the Linear URL fetched in Step 3 as a hyperlink — never use plain ticket IDs:]
*[OKR code] · [KR name]*
• <LINEAR_URL|LWG-XX> — [1-sentence user-value summary]

*[OKR code] · [KR name]*
• <LINEAR_URL|LWG-XX> — [1-sentence user-value summary]
• <LINEAR_URL|LWG-XX> — [1-sentence user-value summary]
...

[If there are non-experiment features (e.g., UX improvements, infra), group separately — also hyperlinked:]
_Plus [N] supporting features [brief description]: <LINEAR_URL|LWG-XX>, <LINEAR_URL|LWG-XX>_

[Or if no tickets: _No experiments in Ready to Launch / In Review / QA_]
```

### Tone rules

- Use strategic, outcome-oriented language: "the team validated", "we're accelerating", "this week we're testing whether"
- Do NOT use feature-spec language: "added a button", "updated the modal", "fixed the flow"
- Lead with business outcomes and customer impact
- Each upcoming release sentence must convey the *why* (user benefit or hypothesis being tested), not the *what* (implementation)
- Keep the narrative section tight — 2–3 sentences maximum
- Always map "Coming this week" experiments to their Q2 OKR + KR — never list them as a flat unstructured list
- Infra and non-experiment work must be framed as strategic enablers (e.g., "unblocks velocity", "prerequisite for O2-KR1") — never as plain maintenance

---

## Error handling

| Condition | Behavior |
|-----------|----------|
| No Weekly Velocity Tracker row for current week | Show `[Weekly Velocity Tracker row not found for this week — ask TL/DL to create it]` for velocity section; continue with Steps 3–5 |
| No experiments released this week | Show `0` — no error |
| No experiments finalized this week | Show `0` — no error |
| No Linear tickets in target statuses | Show `_No experiments in Ready to Launch / In Review / QA_` |
| Linear ticket has no description | Infer user benefit from title alone |
