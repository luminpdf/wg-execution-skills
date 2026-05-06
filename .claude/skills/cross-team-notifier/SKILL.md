---
name: cross-team-notifier
description: >
  Generate a cross-team notification message for an experiment that is about to go live.
  Sends a 48-hour advance FYI notice to stakeholders following the "inform, not consult" principle.
  Trigger when the user says "notify teams about {experiment}", "cross-team notification",
  "release notice for {experiment}", "send FYI for {experiment}", or when an experiment
  moves to "Ready to Launch" status with a release date set.
---

# Cross-Team Notifier

Generate a structured cross-team notification message 48 hours before an experiment goes live. The notification is an FYI — silence means no objection. It is not an approval gate.

## Notion references

- **Database URL:** `https://www.notion.so/ee3129652044476084f200d34d22c1aa`
- **Data source:** `collection://d198507c-8621-47aa-a6df-c726597b2575`
- **Team Contacts Mapping:** `https://www.notion.so/32061884a9478197bb16f42bdd48f09b#33461884a947804fbc0dc00b7f353de2`

## Where this fits in the lifecycle

```
In Development → Ready to Launch → [48h NOTIFICATION SENT] → Live
                           ↑ trigger point
```

Status trigger: experiment is in **"Ready to Launch"** with a release date set.

## Workflow

### Step 1: Identify the experiment

If the user specifies an experiment by name or URL, fetch it directly using `notion-fetch`.

If not specified, query the database for experiments with status **"Ready to Launch"** using `notion-search` or `notion-fetch` on `collection://d198507c-8621-47aa-a6df-c726597b2575`. Present the list and ask the user which experiment to notify about.

### Step 2: Gate check — "Ready to Notify" checklist

Before generating the notification, verify the three items from the Linear ticket's "Ready to Notify" checklist are complete:

| Item | Source | Block or warn |
|------|--------|---------------|
| [TL] Affected Teams set in Notion | `Affected Teams` property on experiment | **BLOCK** |
| [Dev] Growthbook link added in Notion | `Growthbook Link` property on experiment | Warn (omit line from message) |
| [QC] Loom record added in Notion | Loom URL in Linear ticket description/attachments | Warn (omit line from message) |

**Affected Teams (hard gate):**
Read the `Affected Teams` multi-select property from the experiment.
- **If empty (not set):** STOP. Warn the user:
  > "Affected Teams" is not set on this experiment. This is required before sending a cross-team notification. Please set it on the Notion experiment page (options: None, PDF, AG, Sign, Experience, Core) and try again.
- **If set to "None":** Proceed, but skip @-mentions in the notification (broadcast only).
- **If set to one or more teams:** Proceed with targeted @-mentions.

**Growthbook link and Loom record (soft checks):**
If either is missing, note it to the user (e.g., "Growthbook link not found — omitting from notification") and continue. These are included in the message when present but are not blockers.

### Step 3: Extract notification data from the brief

Fetch the full experiment page using `notion-fetch`. Extract the following fields:

| Field | Source in brief |
|-------|----------------|
| `experiment_name` | Page title (`Name` property) |
| `release_date` | Computed: notification_date + 2 business days (earliest). QC confirms exact date in thread. |
| `affected_teams` | `Affected Teams` property (multi-select) |
| `change_summary` | Section 1 / Hypothesis body — 1–2 plain-language sentences describing what is changing (e.g. "Adding social proof badges and trust signals to the Payment Page.") |
| `notion_brief_url` | Page URL |
| `linear_ticket_url` | `Linear Ticket` URL property |
| `linear_ticket_id` | Issue ID extracted from the Linear ticket URL (e.g. `LWG-38`) |
| `growthbook_url` | `Growthbook Link` property on the experiment page, or from the Linear ticket description — set to `None` if not found (omit line from message) |
| `loom_url` | Loom link from the Linear ticket — check attachments and description for any URL matching `loom.com/share/`. Set to `None` if not found (omit line from message) |
| `notification_link` | `Notification Link` property — if already set, warn that notification was already sent |

The release date is not extracted from the brief — it is computed in Step 4. All other fields should come from the brief — if any critical field is missing, note it as "TBD" and warn the user.

#### Resolve team contacts → Slack user IDs

1. Fetch the Team Contacts Mapping table from the Notion workflow page (`https://www.notion.so/32061884a9478197bb16f42bdd48f09b#33461884a947804fbc0dc00b7f353de2`).
2. For each team in `affected_teams`, look up the contact email(s) from the mapping:

   | Team | Contact emails |
   |------|---------------|
   | PDF | thuyntt@luminpdf.com, tientm@luminpdf.com, andb@luminpdf.com |
   | AG | anntt@luminpdf.com, trungdd@luminpdf.com |
   | Sign | anhpq@luminpdf.com, hieudm@luminpdf.com |
   | Experience | anntt@luminpdf.com, tientm@luminpdf.com |
   | Core | anbx@luminpdf.com |

3. For each unique email, use Slack MCP `slack_search_users` to resolve the email → Slack user ID.
4. Collect the resolved Slack user IDs for @-mentioning in the message.
5. If any email fails to resolve, note it and fall back to displaying the email address.

### Step 4: Generate the notification message

Build the `mentions` string: if `affected_teams` contains actual team names (not "None"), format as space-separated @-mentions followed by team names in parentheses — e.g. `<@U123> <@U456> (PDF, Sign)`. If "None", omit the 👋 line entirely.

Build the `growthbook_line`: if `growthbook_url` is available, include `• <{growthbook_url}|Growthbook experiment>`. If not found, omit this line.

Build the `loom_line`: if `loom_url` is available, include `• <{loom_url}|View Demo>` as the last bullet under *What's changing*. If not found, omit this line.

Fill the template below with extracted data. Use Slack mrkdwn formatting (`*bold*`). Use `━━━━━━━━━━━━━━━━━━━━` as section dividers (Slack rejects `---` markdown in messages).

```
*{experiment_name}*

*Heads-up:* {mentions}

*What's changing:*
• {change_summary}
{loom_line}
*Release:* minimum 1h from now — QC will confirm in thread.

*Resources:*
• <{linear_ticket_url}|Linear ticket {linear_ticket_id}>
{growthbook_line}• <{notion_brief_url}|Notion brief>
```

### Step 4.5: Copy standards check

Before finalising the `change_summary` and any copy in the notification, apply Lumin style guide rules:
- Sentence case in all copy (no title case)
- No hyphens after numerals (7 day trial, not 7-day trial)
- No hyphens after prefixes (retyping, not re-typing)
- No Oxford commas
- Numerals always (1 step, not one step)
- eSignature (noun) / eSign (verb) — never "e-signature"
- en-dashes (–) not em-dashes (—)
- No spaces around slashes

Full guide: https://www.notion.so/Lumin-grammar-style-guide-f35f3b8fcedb47e8aaec20c14d143ed3

### Step 5: Present and confirm

Present the filled notification to the user (Jean) for review. Ask:

```
Here's the release notice for {experiment_name}:

{filled_template}

---

Ready to send to #vn-web-growth-heads-up?
Anything you'd like to adjust before sending?
```

Wait for confirmation or edits before posting.

### Step 6: Post and log

After user confirms:

1. Post the message to `#vn-web-growth-heads-up` using Slack MCP `slack_send_message`.
2. Capture the Slack message permalink/URL.
3. Write the Slack message URL back to the `Notification Link` property on the Notion experiment page using `notion-update-page`.

This prevents duplicate notifications — if `Notification Link` is already set, the cron (Phase B) and manual runs will detect it.

## Design rationale

| Decision | Why |
|----------|-----|
| **Lean template — name, what, when, links** | Hien flagged the old template as too noisy; recipients couldn't tell what to focus on |
| **Plain-language change summary** | 1–2 sentences from the hypothesis body — enough context without clicking through |
| **Growthbook link included** | Non-WG engineers need this to see variant configs without asking |
| **@-mentions from Affected Teams** | Targeted notification — only relevant people get pinged |
| **Notification Link write-back** | Dedup mechanism — prevents duplicate sends on re-runs or cron |
| **Affected Teams gate** | Required before notification — forces TL to think about cross-team impact during dev |

## Delivery

**Manual trigger with Slack posting.** The PO runs the skill, reviews the generated message, confirms, and the skill posts directly to `#vn-web-growth-heads-up` via Slack MCP. Automated cron-based triggering (Phase B) will be enabled after manual testing validates the flow.

## Notes

- All fields map to existing experiment brief properties and body sections — no new data entry required beyond `Affected Teams`.
- The PO (Jean) and TL (Hien Tran) are hardcoded as owners since this is a single-team skill.
- If multiple experiments are "Ready to Launch", offer to generate batch notifications.
- Team contacts mapping is maintained on the Notion workflow page — update there if team contacts change.
- Slack user ID resolution happens at notification time via email lookup — no separate mapping file needed.
