# leanpay plugin

Leanpay's public Claude Code plugin — AI infrastructure skills and extensions.

Installed via the [`leanpay-ai`](../../README.md) marketplace:

```
/plugin marketplace add leanpay/leanpay-ai
/plugin install leanpay@leanpay-ai
```

---

## Skills

### `/leanpay:leanday` — Lean into your day

> *Lean into your day.* Scans your Slack channels and Google Calendar, calculates your **Lean Score**, and surfaces what actually needs your attention — before you open a single tab.

#### What it looks like

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  Thursday, June 5
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  🌤️ Good afternoon, John

  ✨ Today's vibe:  Lean Gone 🟠

  ── Slack (since 09:00) ──────────────────
  🔔 Unanswered mentions    ████░░░░░░  2   (direct @you)
  📢 Unanswered broadcasts  ██░░░░░░░░  1   (@here/@channel)
  🚨 Unanswered keywords    ██░░░░░░░░  1

  ── Today ────────────────────────────────
  📅 Meetings               █████░░░░░  4  (40% of day)
  📋 Day type:              Meeting-Heavy Day 📅

  ── What's Ahead ─────────────────────────
  ⏰ Next up:       Team sync in 18 min
  🎯 Focus window:  16:00–18:00  (2h free)

  📌 Needs your attention:

  🔴 [HIGH]   #engineering (14:32): "@you can you take a look at this PR?"
  🟠 [MED]    📅 Unanswered invite: "Q2 Planning" (from: Ana)
  🟠 [MED]    #general (11:05): "bug on staging — anyone available?"
  🟡 [LOW]    #general (10:15): "@here please fill in the lunch order by EOD"
```

#### Requirements

- [Claude Code](https://claude.ai/code)
- Slack connected via MCP
- Google Calendar connected via MCP

#### Setup

On first run, the skill asks (via interactive prompts) for:
1. Which Slack channels to scan
2. Your working hours (e.g. `09:00–18:00`)
3. Hot keywords — keep the defaults, add your own, or replace entirely
4. Optional team or group mentions (e.g. `@frontend-team`) on top of `@you`, `@here`, `@channel`

This is saved to `~/.claude/leanday/config.yaml` and reused on every run.

To update your settings:
```
/leanpay:leanday --reconfigure
```

#### Usage

```
/leanpay:leanday
```

Run it any time — morning, after lunch, after a meeting. The Slack window always covers from your working hours start to right now, so you only see what's relevant.

#### How Slack scanning works

The skill scans your configured channels from the start of your working day to the current time. It looks for:
- Messages that directly mention you (@you)
- Broadcast mentions (@here, @channel, @everyone)
- Custom team or group mentions you configured (e.g. @frontend-team)
- Messages containing hot keywords

For each match, it checks whether you've already replied. If you have — it's skipped. Only unanswered messages show up.

#### Lean Score

Your score starts at 100 and goes down based on what's pending:

| Signal | Deduction |
|---|---|
| Each unanswered direct mention (@you) | -8 pts |
| Each unanswered broadcast (@here / @channel) | -3 pts |
| Each unanswered hot keyword message | -5 pts |
| Each meeting today | -5 pts |
| More than 4 meetings today | -10 pts |
| Back-to-back meetings | -5 pts |

#### Vibe labels

Every score tier has a pool of labels — picked randomly each run so it never gets stale.

| Score | Vibes |
|---|---|
| 90–100 | Lean Dream 🧘 · Peak Lean ✨ · Lean Machine 💚 · Full Lean 🌿 · Lean Bliss 🌤️ |
| 70–89 | Lean & Clean 🌿 · Lean Streak 💪 · Leaning In 🙂 · Mostly Lean 👌 · Lean Enough 🟢 |
| 50–69 | Losing the Lean 🟡 · Lean-ish 🤔 · Leaning Out 😬 · Mildly Chunky 🟡 · Lean Under Pressure ⚠️ |
| 30–49 | Lean Gone 🟠 · Where Did Lean Go? 😅 · Anti-Lean 🔴 · Lean? What Lean? 😤 · Heavy Day Ahead 🟠 |
| 0–29 | Abandon Lean 🔥 · Anything But Lean 🚨 · Full Sprint No Brakes 🔥 · Lean Left the Building 💀 · SOS: Zero Lean 🆘 |
