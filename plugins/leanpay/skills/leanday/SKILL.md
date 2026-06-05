---
name: leanday
description: Daily Slack briefing with a Lean Score — scans your configured Slack channels and Google Calendar, surfaces unanswered mentions and hot keyword messages, and tells you how lean your day looks.
when_to_use: When the user runs /leanday or asks for a morning Slack summary and day vibe check.
---

You are running the /leanday skill. Scan the user's Slack channels and Google Calendar, calculate a Lean Score, and present a structured daily briefing.

**CRITICAL RULES — strictly enforced:**
- **All user-facing questions in this skill MUST be asked via the `AskUserQuestion` tool — never as free-text "type X" prompts.** The user supplies any custom value (channel list, working hours, keywords, mentions) through the tool's built-in **"Other"** free-text entry. Pure status messages that are NOT questions (e.g. "config saved", "MCP failed") stay as normal one-line output.
- The silence rules below apply from the moment you **start gathering data (Step 4)** onward. Setup questions in Steps 1–2 are asked normally via `AskUserQuestion` before that.
- Once data gathering begins: do NOT narrate ("Checking...", "Found...", "Scanning..."), do NOT show intermediate results/counts/findings, do NOT explain your process.
- The moment before gathering data, output exactly one line: `⏳ Gathering your data...` — then go silent until the final briefing.
- From `⏳ Gathering your data...` onward the only outputs are that line and the final briefing.

## Step 1: Hot keywords

**Call A — mode (`AskUserQuestion`, single-select).** Ask one question, header `Keywords`. Put the default list in the question text:

> Here are the hot keywords I'll scan for: `urgent, urgentno, hitno, bitno, važno, bug, problem, help, asap, deadline, blocker, blocked, critical, broken, crash, down, outage, production, produkcija, reminder, review`. What would you like to do?

Options:
- **Keep as is** — use the default list unchanged.
- **Add more** — keep the defaults and append my own.
- **Replace** — use only my own keywords.

- If **Keep as is** → continue with the default list.
- If **Add more** or **Replace** → make **Call B**.

**Call B — collect keywords (`AskUserQuestion`, single question) — only if Add more / Replace.** Header `Keywords`. Question: "Type your keywords as a comma-separated list in the **Other** field." Provide a few illustrative option chips the user may pick instead of typing (e.g. `release, deploy, rollback` · `incident, sev1, pager` · `regression, hotfix`). Read the selected chip OR the Other text as the list.
- For **Add more** → append to the default list for this session.
- For **Replace** → use only the supplied list for this session.

Then continue.

## Step 2: Load or create channel config

Check if `~/.claude/leanday/config.yaml` exists.

**If it does NOT exist** — ask all three setup questions in a **single `AskUserQuestion` call (3 questions)**:

1. Header `Channels` — "Which Slack channels should I scan? Type them comma-separated in **Other** (e.g. `#engineering, #general, #backend`)." Provide 2–3 illustrative chips (e.g. `#engineering, #general` · `#backend, #frontend`); the real answer normally comes via Other.
2. Header `Hours` — "What are your working hours?" Options: `09:00–17:00`, `09:00–18:00`, `10:00–18:00`, plus Other for a custom range.
3. Header `Mentions` — "I already track @you, @here, @channel and @everyone. Add team/group mentions? To add, type them comma-separated in **Other**." Options: **No extra mentions** (defaults are enough) and one illustrative chip (e.g. `@frontend-team, @backend-team`).

Read each answer (selected option or Other text). Save to `~/.claude/leanday/config.yaml` in this format:

```yaml
channels:
  - "#channel-one"
  - "#channel-two"
working_hours:
  start: "09:00"
  end: "18:00"
custom_mentions:   # e.g. ["@frontend-team", "@backend-team"] — empty list if "No extra mentions"
```

Then output (plain line, not a question): "Got it! I'll remember this. Run `/leanday --reconfigure` any time to change your settings."

**If it exists**, try to parse it. If the file is malformed or unreadable, output (plain line): "Your config file at `~/.claude/leanday/config.yaml` could not be read. Run `/leanday --reconfigure` to set it up again." and stop.

Load the config and check if all required fields are present: `channels`, `working_hours`, `custom_mentions`. If any are missing, ask **only the missing ones** in a single `AskUserQuestion` call (the matching subset of the three questions above), using the same option design. Save the answers back to the config file. Then continue.

**If the user runs `/leanday --reconfigure`:**

**Call A (`AskUserQuestion`, single-select)**, header `Reconfigure`. Put the current settings in the question text:

> Current settings — Channels: [list from config]; Working hours: [from config]. What would you like to do?

Options:
- **Add more channels** — keep existing settings and extend.
- **Replace everything** — start the config from scratch.

- If **Replace everything** → run the same single 3-question call as first-run (above) and overwrite the config.
- If **Add more channels** → make **Call B**: a single `AskUserQuestion` call with 3 questions:
  1. Header `Channels` — "Which channels to add? Type them comma-separated in **Other**." (appended to the existing list)
  2. Header `Hours` — "Working hours?" Options: **Keep current ([start]–[end])**, `09:00–17:00`, `09:00–18:00`, `10:00–18:00`, plus Other. "Keep current" = no change.
  3. Header `Mentions` — "Team/group mentions?" Options: **Keep current**, **Replace (type in Other)**; plus Other to add. Apply accordingly.

  Save the merged result back to the config.

## Step 3: Authenticate

Connect to Slack MCP and Google Calendar MCP. If either fails, notify the user which one failed and continue with what's available.

## Step 4: Gather data

First, run this bash command to get the exact current date and time, plus yesterday's date:

```bash
date '+%Y-%m-%d %H:%M' && (date -d 'yesterday' '+%Y-%m-%d' 2>/dev/null || date -v-1d '+%Y-%m-%d')
```

Store the first line as `NOW` and the second as `YESTERDAY`. Use `NOW` as the explicit time parameter in every Slack and Google Calendar MCP call. Do not let any tool determine "current time" on its own — always pass `NOW` explicitly as the time reference. Do not assume, estimate, or use any other source for the current time.

**Note on Slack search date filters:** Slack's `after:YYYY-MM-DD` modifier is exclusive — `after:TODAY` excludes today's messages. Always use `after:YESTERDAY` (i.e. `after:[TODAY-1]`) in any `slack_search_public_and_private` query strings to correctly include today's messages. The `oldest`/`latest` Unix timestamp parameters on `slack_read_channel` are not affected by this and can use the exact timestamps from `NOW`.

**From Slack — window: `working_hours.start` → NOW (exact time from bash above):**

Before reading, validate each channel name: it must start with `#` and contain no spaces. If a channel name is invalid, skip it and add to the failed channels list without calling the MCP.

For each configured channel:
1. Read all messages strictly between `working_hours.start` and the exact current time obtained above. If the read fails for a channel (invalid name, permissions, network error), skip it and add it to a list of failed channels to report at the end.
2. Find messages that:
   - Directly mention the user (@username), OR
   - Contain a broadcast mention (@here, @channel, @everyone), OR
   - Contain any custom mention from `custom_mentions` in config (if any), OR
   - Contain any hot keyword from Step 1
3. For each found message, check its thread and subsequent channel messages to see if the user has already replied
4. Keep only messages the user has NOT replied to — these need attention
5. If the user has replied, skip that message entirely

**From Google Calendar:**

Whole day overview (window: `working_hours.start` → `working_hours.end`):
- All meetings today — title, start time, duration
- Total meeting count and % of working day occupied

Upcoming only (window: exact current time → `working_hours.end`):
- Meetings that start strictly AFTER the current time — these are "What's Ahead"
- Back-to-back blocks among upcoming meetings only
- Longest uninterrupted free block from NOW until `working_hours.end`
- Unanswered invites (status: needs action / not responded)

Do NOT show past meetings (start time < current time) in the "What's Ahead" section.

## Step 5: Calculate Lean Score

Start at 100. Deduct based on whole-day data:
- Each unanswered direct mention (@username): -8 pts
- Each unanswered broadcast (@here / @channel / @everyone): -3 pts
- Each unanswered hot keyword message: -5 pts
- Each meeting today (whole day): -5 pts
- More than 4 meetings today: additional -10 pts
- Any back-to-back meeting block: -5 pts

Minimum 0, maximum 100.

## Step 6: Pick a random vibe label

Based on score, randomly pick ONE label from the matching tier:

- **90–100:** Lean Dream 🧘 / Peak Lean ✨ / Lean Machine 💚 / Full Lean 🌿 / Lean Bliss 🌤️
- **70–89:** Lean & Clean 🌿 / Lean Streak 💪 / Leaning In 🙂 / Mostly Lean 👌 / Lean Enough 🟢
- **50–69:** Losing the Lean 🟡 / Lean-ish 🤔 / Leaning Out 😬 / Mildly Chunky 🟡 / Lean Under Pressure ⚠️
- **30–49:** Lean Gone 🟠 / Where Did Lean Go? 😅 / Anti-Lean 🔴 / Lean? What Lean? 😤 / Heavy Day Ahead 🟠
- **0–29:** Abandon Lean 🔥 / Anything But Lean 🚨 / Full Sprint No Brakes 🔥 / Lean Left the Building 💀 / SOS: Zero Lean 🆘

## Step 7: Determine Day Type

Based on whole-day meeting load and Slack noise:

- **Focus Day 🎯** — fewer than 3 meetings, no unanswered Slack items (score 70+)
- **Balanced Day ⚖️** — moderate meetings and some Slack activity (score 50–69)
- **Meeting-Heavy Day 📅** — more than 4 meetings or 50%+ of day in meetings
- **Fire Drill Day 🚨** — unanswered hot keywords AND heavy meetings
- **Slack Storm Day 🌪️** — many unanswered Slack items but light calendar

## Step 8: Display the briefing

Output ONLY this, nothing else before or after:

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  [Day], [Date]
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  [GREETING], [name]

  ✨ Today's vibe:  [RANDOM VIBE LABEL]

  ── Slack (since [working_hours.start]) ──────────────────
  🔔 Unanswered mentions    [bar]  [count]   (direct @you)
  📢 Unanswered broadcasts  [bar]  [count]   (@here/@channel)
  🚨 Unanswered keywords    [bar]  [count]

  ── Today ────────────────────────────────
  📅 Meetings               [bar]  [count]  ([X]% of day)
  📋 Day type:              [DAY TYPE LABEL]

  ── What's Ahead ─────────────────────────
  ⏰ Next up:       [meeting title] in [X] min
  🎯 Focus window:  [HH:MM–HH:MM]  ([X]h free)

  📌 Needs your attention:

  🔴 [HIGH]   #[channel] ([time]): "[message excerpt]"
  🟠 [MED]    📅 Unanswered invite: "[meeting title]" (from: [organizer])
  🟡 [LOW]    #[channel] ([time]): "[message excerpt]"

  ── Lean Score ───────────────────────────
  [score] / 100  [each deduction on its own line, e.g.:]
  └─ -8   1 unanswered mention
  └─ -3   1 unanswered broadcast
  └─ -5   1 hot keyword message
  └─ -25  5 meetings today
  └─ -10  more than 4 meetings
  └─ -5   back-to-back meetings
```

**Greeting** — based on current time:
- Before 12:00 → `☀️ Good morning`
- 12:00–17:00 → `🌤️ Good afternoon`
- After 17:00 → `🌙 Good evening`

**Progress bars:** filled/empty blocks relative to the highest count. Max 10 blocks.

**Priority rules:**
- HIGH — unanswered direct mentions (@username)
- MED — unanswered @here / @channel / @everyone that contain a hot keyword; unanswered calendar invites
- LOW — unanswered @here / @channel / @everyone without hot keywords; other unanswered hot keyword messages

Sort within each priority by recency (newest first). Show max 7 items total.

If the attention list is empty (no unanswered mentions, keywords, or calendar invites), replace the attention list with:
> 🧘 Your Slack is clear and your calendar is under control. Go build something great.

If any channels failed to load, append at the very end:
> ⚠️ Could not read: [channel names]. Check that they exist and you have access.

**Lean Score breakdown** — always shown at the bottom. Only list deductions that actually applied (skip lines with 0 deduction). Use plain human-readable labels, e.g.:
- `-8   1 unanswered mention`
- `-3   2 unanswered broadcasts`
- `-5   1 hot keyword message`
- `-20  4 meetings today`
- `-10  more than 4 meetings`
- `-5   back-to-back meetings`

If no deductions at all (score 100): show `100 / 100  ✓ Nothing to deduct.`
