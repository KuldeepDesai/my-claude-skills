---
name: "slack-morning-todo"
description: "Build a prioritized morning to-do list from Slack: scans the user's starred channels, @mentions, DMs, threads they're in, and notes they wrote to themselves, then publishes it as a live artifact page. Use whenever the user asks for their Slack to-do list, \"what do I need to do from Slack\", a morning Slack digest, missed mentions or action items, or \"what did I forget overnight\" — and for the scheduled daily run. Requires the Slack connector; if it isn't installed, stop and say so."
---

# Slack Morning To-Do

Turn the user's overnight Slack activity into one prioritized, trustworthy to-do list. The user works across timezones and often reads messages late at night, then forgets by morning. The list must be complete enough to replace re-reading Slack, and precise enough that each item says exactly what is being asked of them.

## Step 0: Check the Slack connector

Look for Slack MCP tools (search messages, read channel history, list channels, read threads, user lookup). Use ToolSearch to load them if deferred. If no Slack tools exist, STOP. Do not fall back to a browser, scraping, or guessing. Tell the user: "The Slack connector isn't installed. Install it from https://claude.ai/directory/slack, then rerun." Produce no list.

## Step 1: Establish identity and time window

1. Identify the user's own Slack user ID (via the connector's current-user/profile tool). All "mine" filters depend on this.
2. Determine the window start. Prefer "since last run": check existing artifact titled "Slack To-Do" (list_artifacts) for a stored `last_run` timestamp (kept in an HTML comment `<!-- last_run: ISO8601 -->`). If none exists, use the last 24 hours (48 hours if today is Monday, to cover the weekend). Always include a ~2-hour overlap before window start so nothing falls between runs, and dedupe afterward.
3. Note the user's timezone and render all times in it.

## Step 2: Collect sources

Gather in this order; each is a distinct way work reaches the user:

1. **@mentions**: search for messages mentioning the user (`<@USERID>`, and @here/@channel only when the message clearly asks someone to act and the user is a plausible owner). Include user-group mentions for groups the user belongs to.
2. **Direct messages and group DMs** received in the window.
3. **Starred channels**: list the user's starred channels/conversations and read history in the window. Starred channels signal importance, so look for requests, decisions needing input, and questions even without a tag. Skip pure chatter, FYIs, and bot noise unless the bot message demands action (e.g., approval requests, failed jobs assigned to them).
4. **Threads the user participated in**: re-read threads they've replied in that have new messages in the window; a new reply often contains a follow-up question or request.
5. **Self-messages**: read the user's self-DM ("notes to self") in the window, plus anything older that is still unresolved if it clearly reads as a pending to-do. Treat these as high-signal: the user wrote them to remember. Keep their wording, lightly cleaned.

Always read the full thread for any message you surface; the actual ask is often in a reply, and the ask may already be resolved.

## Step 3: Decide what is actually a task

For every candidate, answer: *Is the user expected to do something, and is it still open?*

- Include: direct requests, questions awaiting their answer, review/approval asks, deadlines, commitments the user made ("I'll send this by Friday"), self-notes, and decisions blocked on them.
- Exclude or mark done: items where the user already replied with the deliverable or a thread reaction/reply shows it's resolved; messages addressed to someone else in the same thread; pure FYIs; thanks/acknowledgements.
- When ambiguous (the user is tagged but the ask is unclear), include it in a "Check / unclear" bucket with a one-line reason rather than dropping it. A missed task costs more than one extra line.
- Merge duplicates: the same ask across channel, thread, and DM becomes one task with multiple source links.

For each task capture: a verb-first action ("Send revised pricing deck to Anna"), who asked, source channel/DM, deadline if stated or implied (convert relative dates like "EOD Friday" using the sender's timezone when known, and show it in the user's timezone), the message timestamp, and a Slack permalink.

## Step 4: Prioritize

Group into:

- **Urgent**: due today or overdue, explicit urgency words, or a senior person/client waiting, or something blocking others.
- **Today**: should be done today without being critical.
- **This week**: has a later deadline or no deadline but is a real ask.
- **Self-notes**: the user's own reminders, listed together, ordered by age.
- **Check / unclear**: ambiguous items.

Within a group, order by deadline, then by how long the item has been waiting. Overdue items older than the window still belong on the list if unresolved; flag them "carried over".

## Step 5: Publish as a live artifact

Create (first run) or update (later runs) a single artifact titled **Slack To-Do** with `mcp__cowork__create_artifact` / `mcp__cowork__update_artifact`. Reuse the existing one instead of creating duplicates. Load the tools via ToolSearch if deferred.

Page content (self-contained HTML, static data from this run, no connector calls needed in-page):

- Header: date, window scanned, counts per group.
- Sections in the order above, each task as a checkbox row: action, who/where, deadline chip, "waiting since" age, and a link to the Slack message. Checkbox state may use in-page JS only; do not rely on localStorage for correctness.
- A short "Scanned" footer listing channels read and sources checked, so the user can trust coverage and spot gaps.
- Hidden comment `<!-- last_run: ISO8601 -->` for the next window.

Keep it scannable: no paragraphs, one line per task plus a small metadata line. If there are zero tasks, say so explicitly and list what was scanned.

## Step 6: Reply briefly

In chat, give a 2-4 line summary: counts per group, the top 2-3 urgent items, and that the page was updated. If any channel or DM couldn't be read (permissions, rate limits), say which.

## Principles

- Precision over volume: every item must trace to a real message with a link. Never invent tasks or deadlines; if the deadline isn't stated, leave it blank.
- Privacy: only the user's own accessible content; don't post anything to Slack, and don't send messages or reactions.
- When uncertain, surface it in "Check / unclear" instead of silently dropping it.

