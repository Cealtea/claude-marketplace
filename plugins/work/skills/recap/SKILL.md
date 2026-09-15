---
name: recap
description: Compile the user's recently completed Linear tickets and closed Obsidian daily-note todos into a short recap and DM it to them on Slack. Manual/on-demand only — there is no recurring schedule for this. Use this whenever the user asks for a work recap in their own words — "post my recap to Slack", "send my done-ticket summary", "DM me what I finished this week", "what did I close since Friday/Tuesday" — or runs `/work:recap` directly.
---

# Linear + Obsidian → Slack recap

The user wants a periodic recap of what they've shipped, covering two sources of "done" work
(Linear tickets, Obsidian daily-note todos) merged into one Slack DM to themselves. They trigger
this manually — there's no recurring schedule for it — but each run still runs unattended in the
sense that matters here: build the message and send it, don't draft and wait for approval.

Before running, check whether the same window has already been sent recently (e.g. you or the
user ran this a few minutes ago with nothing new since). If nothing has changed since the last
send, say so and ask before sending a duplicate, rather than DMing the same recap twice.

## Configuration file

All user-specific values live in a local config file, **never in this skill**:

```text
~/.config/work-recap/config.json
```

Shape:

```json
{
  "slack_user_id": "U0XXXXXXXXX",
  "linear_user_id": "00000000-0000-0000-0000-000000000000",
  "obsidian_standup_dir": "/Users/me/Documents/Obsidian/Vault/Standup",
  "timezone": "America/Los_Angeles",
  "anchor_days": ["Tuesday", "Friday"],
  "anchor_time": "10:00"
}
```

| Key | Required | Meaning |
| --- | --- | --- |
| `slack_user_id` | yes | The user's Slack member ID. Used as `channel_id` for the DM. |
| `linear_user_id` | yes | The user's Linear user UUID. Used to filter "issues I created". |
| `obsidian_standup_dir` | no | Absolute path to the folder holding Obsidian daily notes. Omit or set to `null` to skip the Obsidian half. |
| `timezone` | yes | IANA time zone used for the window and weekday labels. |
| `anchor_days` | yes | The two weekdays the user normally runs the recap on. Default `["Tuesday", "Friday"]`. |
| `anchor_time` | yes | Local time-of-day the window starts at on an anchor day. Default `"10:00"`. |

### First run: setup

At the start of every run, read the config file:

```bash
cat "$HOME/.config/work-recap/config.json" 2>/dev/null
```

If it is missing, unparsable, or lacks a required key, run setup **before** anything else. Ask the
user for each value below, one short question at a time. Never guess a value from memory; only
prefill something you have actually looked up in this session.

1. **Slack user ID.** Ask for it directly. If a Slack user-search tool is available, offer to look
   it up by the user's name or email and confirm the result with them. Slack member IDs start
   with `U` or `W`.
2. **Linear user ID.** If a Linear "get user" / "viewer" tool is available, call it for the current
   user (`"me"`) and confirm the returned UUID with the user. Otherwise ask them to paste it
   (Linear → Settings → Account → the ID in the profile URL, or ask them to run the lookup later).
3. **Obsidian standup folder.** Ask for the absolute path to the folder containing daily notes, or
   "skip" to disable the Obsidian half. If given a path, verify it exists with `ls` before saving.
4. **Time zone.** Offer the machine's zone as the default:
   `readlink /etc/localtime | sed 's#.*/zoneinfo/##'` (fall back to `date +%Z` if that fails).
5. **Anchor days and time.** Offer the defaults (Tuesday and Friday at 10:00) and accept them
   unless the user wants something else.

Then write the file (mode 600) and tell the user where it was saved and that they can edit or
delete it to reconfigure:

```bash
mkdir -p "$HOME/.config/work-recap"
cat > "$HOME/.config/work-recap/config.json" <<'EOF'
{ ...values collected above... }
EOF
chmod 600 "$HOME/.config/work-recap/config.json"
```

If the user says "reconfigure", "change my recap settings", or similar, rerun setup with the
current values offered as defaults.

Do not echo the full config back into a Slack message or any shared location.

## Tool names

The exact MCP tool names are namespaced per connection and can differ between sessions. Resolve
each of these by capability rather than assuming a literal name; use ToolSearch with a keyword
query (e.g. `"list_issues linear"`, `"slack send message"`, `"device bash remote"`) if a tool
isn't already loaded.

- **Linear `list_issues`** (and optionally a Linear `get_user` tool for setup).
- **Slack `slack_send_message`** (and optionally a Slack user-search tool for setup).
- **Shell access to the user's machine**, needed for the Obsidian step. When this skill runs in
  an interactive session on the machine that holds the vault, plain **Bash** already has this.
  Only reach for a remote `device_bash`-style tool if plain Bash can't see
  `obsidian_standup_dir` at all, meaning the skill is running somewhere else (e.g. a cloud-hosted
  session). See Step 2.

## Step 0 — compute the time window

The user runs this on their two `anchor_days`, anchored at `anchor_time` in `timezone`. Each run
covers the gap since the *other* anchor, not a fixed number of days. With the defaults
(Tuesday/Friday, 10:00):

- Run on **Tuesday** → window is **last Friday 10:00 → now** (the weekend plus Mon, ~4 days).
- Run on **Friday** → window is **this Tuesday 10:00 → now** (~3 days).

Window end is always "now" (the actual moment this runs), not a rounded-off anchor time.

To compute the window start: get today's weekday in `timezone`, then pick the *other* anchor day
and walk back to its most recent occurrence.

```bash
TZ="$TIMEZONE" date +"%Y-%m-%d %H:%M %Z (ISO weekday %u, 1=Mon..7=Sun)"
```

- If today is the first anchor day: window start = the most recent occurrence of the second
  anchor day.
- If today is the second anchor day: window start = the most recent occurrence of the first
  anchor day.
- If run ad hoc on some other day: fall back to whichever anchor day most recently passed, i.e.
  `days_back = min((weekday - a1) % 7, (weekday - a2) % 7)` over the two anchors' ISO weekday
  numbers. This keeps ad hoc runs sane without needing a third case.

Take `anchor_time` on the resulting date as the window start.

You'll need both the precise window-start timestamp (for Linear's `completedAt`, which has
time-of-day) and the window-start **date** alone (for Obsidian, which only records dates — see
Step 2).

## Step 1 — Linear

Use the Linear MCP's `list_issues` tool twice, then merge:

**a) Issues assigned to the user:**
`assignee: "me"`, `state: "completed"`, `updatedAt: "-P7D"`, `limit: 100`,
`fields: ["title", "url", "completedAt", "status", "team"]`.

Filter by state *type* `"completed"` rather than a state name: workspaces often rename their
"done" state (e.g. to "Production"). The returned `id` field is the human-readable identifier
(e.g. `ABC-123`), not an opaque UUID.

**b) Issues the user created but that aren't assigned to them:**
`state: "completed"`, `updatedAt: "-P7D"`, `limit: 250`,
`fields: ["title", "url", "completedAt", "createdById"]`.
Keep only results where `createdById` equals `linear_user_id` from the config. If the response
indicates more pages are available, follow the cursor for at most 2 additional pages — this is
a backstop query, not the primary source, so don't chase it indefinitely.

**c) Merge and filter:**
Combine (a) and (b), dedupe by identifier, then drop anything whose `completedAt` falls outside
the Step 0 window (before window-start or after now).

## Step 2 — Obsidian

Skip this step entirely if `obsidian_standup_dir` is missing or `null` in the config.

Daily notes live under `obsidian_standup_dir` (typically nested `YYYY/MM/`). They use the
Obsidian Tasks plugin format, e.g.:

```text
- [x] only deploy to review app with tag on ai-service ➕ 2026-08-27 ✅ 2026-08-27
- [x] book flight for offsite ➕ 2026-09-10 📅 2026-09-11 ✅ 2026-09-14
```

The date after ✅ is the completion date. Ignore every other emoji-date marker on the line
(➕ created, 📅 scheduled, and any others the Tasks plugin adds) — they're metadata, not part of
the task text.

**Getting a shell on the user's machine:** try plain Bash first — `ls "$OBSIDIAN_STANDUP_DIR"`.
If that path exists, you're already running on the right machine and every command below just
works. Only if that path is missing (you're running in some other environment with no filesystem
access to the vault) should you look for a remote `device_bash`-style tool instead — see Tool
names above. If neither gets you a shell with the vault, treat it as unreachable (see below).

Grep recursively under `obsidian_standup_dir` (not sibling backlog folders — those aren't daily
completions) for lines starting with `- [x]`, then keep only those whose ✅ date is on or after
the window's start **date** (not timestamp — Obsidian only records a date, so the entire start
day counts as in-window, even though the Linear window technically starts at `anchor_time` that
day). A plain date-range grep on the ✅ value is easy to get subtly wrong across month
boundaries — parsing dates properly is more reliable, e.g.:

```bash
python3 - "$OBSIDIAN_STANDUP_DIR" 2026-09-11 <<'PY'
import re, subprocess, sys
from datetime import date

standup_dir, start = sys.argv[1], sys.argv[2]   # substitute the real window-start date
window_start = date(*map(int, start.split('-')))

result = subprocess.run(
    ['grep', '-rn', r'^- \[x\]', standup_dir],
    capture_output=True, text=True
)
for line in result.stdout.strip().split('\n'):
    if not line:
        continue
    path, lineno, text = line.split(':', 2)
    m = re.search(r'✅\s*(\d{4}-\d{2}-\d{2})', text)
    if not m:
        continue
    done_date = date(*map(int, m.group(1).split('-')))
    if done_date >= window_start:
        clean = re.sub(r'^- \[x\]\s*', '', text)
        clean = re.sub(r'[➕📅✅🛫⏳]\s*\d{4}-\d{2}-\d{2}', '', clean).strip()
        print(f'{done_date.strftime("%a")} | {clean}')
PY
```

If the vault is genuinely unreachable (no local filesystem access and no working
`device_bash`-equivalent), don't fail the whole recap — note that in one short line in the final
message and continue with the Linear half only.

## Step 3 — build the message

Standard markdown (the Slack tool converts it). In this order:

**1. Linear tickets**, one per line, bold weekday abbreviation only (no date, no time) — the
weekday comes from `completedAt` converted to `timezone`:

```text
**Mon** — [ABC-123](url) Title
```

**2. Obsidian todos**, one per line, same bold-weekday style, weekday taken from the task's ✅
date:

```text
**Wed** — task text
```

**3. A `**Summary**` section** — bullet points. Group related tickets/todos into one bullet each
rather than restating every line verbatim; the point is to capture the handful of distinct
threads of work, not to transcribe the list above a second time.

If a section has no items, say so in one short line instead of an empty header. If both Linear
and Obsidian are empty, skip the itemized sections and Summary entirely — just send one line
saying nothing was completed in the window. Keep the whole message concise; this is a recap
someone skims, not a report they read closely.

## Step 4 — send

Call the Slack MCP's `slack_send_message` with `channel_id` set to `slack_user_id` from the
config and the composed message. Send it directly — no draft tool, no confirmation step (the
user invoked this themselves, so the message going to their own DM doesn't need a second
sign-off). The one exception is the duplicate check from the top of this file: if this exact
window was already sent recently with no new items, confirm before sending again instead of
DMing the same recap twice.
