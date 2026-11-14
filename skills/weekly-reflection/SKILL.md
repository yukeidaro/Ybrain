---
name: weekly-reflection
version: 1.0.0
description: |
  Weekly interview that keeps memory and skills current from real use. Reads
  the past week's agent sessions (Claude Code transcripts) and meeting/personal
  notes, finds what the brain should learn (repeated preferences, corrections,
  contradictions, friction with a skill), asks the user a few evidence-backed
  questions one at a time, and applies only the answers they approve: facts via
  remember, skill edits via skillify. Use on a weekly cron or when the user says
  "weekly reflection", "review my week", or "what should you learn from this week".
triggers:
  - "weekly reflection"
  - "review my week"
  - "what should you learn from this week"
  - "update your memory from my sessions"
  - "interview me about this week"
tools:
  - query
  - search
  - get_page
  - put_page
  - remember
mutating: true
writes_pages: true
writes_to:
  - daily/
---

# Weekly Reflection — Learn From Real Sessions, Confirmed by the User

> **Convention:** See `skills/conventions/brain-first.md`. Questions follow the
> `skills/ask-user/SKILL.md` gate: one question per message, stop and wait.

Memory and skills drift away from how the user actually works. Once a week this
skill reads what really happened, turns it into a short interview, and writes
back only what the user confirms. The user's answers are the source of truth;
sessions are only the evidence that prompts a question.

## Contract

This skill guarantees:
- **Evidence first.** Every question cites where it came from (page slug plus a
  short quote or paraphrase from that week). No evidence, no question.
- **The user decides.** Nothing is remembered, corrected or edited without an
  explicit yes to that specific item. Silence or "skip" changes nothing.
- **Bounded.** At most 5 questions per run, one per message, highest value first.
  An item the user skipped is not re-asked for 4 weeks; one they declined is not
  re-asked at all.
- **Right tool for each change.** New facts go through `remember` with
  `--content-origin user_said`; wrong facts through `correction-pipeline`;
  identity answers through `soul-audit`; skill changes through `skillify`.
- **A record every week.** One `daily/reflections/YYYY-Www` page lists what was asked,
  the answers, what changed and what was skipped, so the next run builds on it.

## Phases

1. **Read last week's record.** `gbrain get daily/reflections/<last-week>` if it exists.
   Note skipped items (wait 4 weeks), declined items (never re-ask) and any
   change that was promised but not applied.

2. **Bring new sessions in.** Import Claude Code sessions since the last run:
   `gbrain transcripts ingest --format claude-code --since last`. If the user
   archives sessions another way, the `conversation-archive` skill owns that.

3. **Gather the week.**
   - `gbrain query "decisions, preferences and corrections this week" --since 7d`
   - `gbrain query "what was discussed and decided" --types meeting --since 7d`
   - `gbrain query "ideas, todos and open questions" --types note --since 7d`
   Read the hits that matter with `gbrain get <slug>` (or `--around-chunk-id`).

4. **Find candidates.** Look for, in this order of value:
   - **Corrections** — the user told the agent it was wrong, or redid its work.
   - **Repeated instructions** — the user said the same thing in 2+ sessions
     (a preference or rule the brain should already know).
   - **Contradictions** — this week's statement conflicts with a stored fact
     (`gbrain recall "<topic>" --json` to check).
   - **Skill friction** — a skill was skipped, misrouted, or its steps were
     overridden by the user; or a manual routine recurred that no skill covers.
   Drop anything already confirmed, declined, or that a fresh session would get
   right without help (CONTRIBUTING.md, "Changing a skill", step 2).

5. **Interview.** Rank candidates and ask the top ones, one per message, in the
   user's language. Each question has the evidence line, then 2-4 options with a
   skip. Stop the turn after each question.

6. **Apply what was approved.**
   - New fact or preference: `gbrain remember "<fact in the user's words>" --entity <slug> --content-origin user_said --provenance "weekly reflection YYYY-MM-DD"`
     (add `--replaces <fact_id>` when it supersedes a stored fact).
   - Stored fact was wrong: hand off to `correction-pipeline`.
   - Answer about who the user is or how the agent should behave: `soul-audit`.
   - Skill change: run `skillify` on that skill. Edit the existing line, do not
     append; the edit must not score below the previous version.

7. **Write the record.** `gbrain put daily/reflections/YYYY-Www` with the format below.

## Question Format

```
🪞 Weekly reflection (2/4)

Evidence: in 3 sessions this week (conversations/sessions/2026-11-10-…, …)
you asked for answers in a table before prose.

Should I remember this as a standing preference?
1. **Yes, always** — save it as a preference
2. **Only for reports** — save it with that scope
3. **No** — it was situational
4. **Skip** — ask me another week
```

## Output Format

The `daily/reflections/YYYY-Www` page:

```markdown
# Weekly reflection — YYYY-Www

**Period:** YYYY-MM-DD to YYYY-MM-DD
**Sources read:** N sessions, M meetings, K notes

## Asked
| # | Question | Evidence | Answer |
|---|----------|----------|--------|

## Changed
- Memory: {fact} (fact_id …)
- Skill: {skill} — {one-line change}

## Skipped (re-ask after YYYY-MM-DD)
## Declined (do not re-ask)
## Not asked (over the limit, carried to next week)
```

Reply to the user with a 3-line summary: how many questions, what changed, what
carries over. If the week had nothing worth asking, say so in one line.

## Weekly schedule

Install through the `cron-scheduler` skill (it handles quiet hours):

- **Schedule:** once a week at a time the user picks (e.g. Friday 17:00 local).
- **Job prompt:** `Read skills/weekly-reflection/SKILL.md and run it for the past 7 days. Ask me at most 5 questions, one at a time.`
- **Idempotent:** if `daily/reflections/<this-week>` already exists, continue it instead
  of starting over.

## When it fails

Follow the [agent operator protocol](../conventions/agent-operator-protocol.md) for any gbrain error `code`, exit code, `[AGENT]` block or notice block. Specific to this skill:

- `gbrain transcripts ingest` finds no Claude Code sessions: say which material was read (meetings and notes only) and continue; do not report "nothing happened this week".
- A query prints a degraded notice or `GBRAIN_DB_ACCESS`: stop and route to `db-repair`; an unreadable brain is not an empty week.
- `remember` returns `write_pending`: poll `gbrain write-request <request_id>` before telling the user it was saved.
- The user does not answer: record the open questions as "Not asked" and end; never apply a default.

## Anti-Patterns

- Asking a question without evidence from this week, or inventing the evidence.
- Saving an inference as `user_said`: only the user's own answer gets that origin.
- Stacking several questions in one message, or asking more than 5.
- Editing a skill or memory before the user said yes to that item.
- Writing the user's private details (names, quotes, clients) into a skill body;
  skills stay deployment-neutral, the specifics live in brain pages.
- Re-asking a declined item, or a skipped one before 4 weeks.
- Quoting secrets from transcripts; if a session page still holds a credential,
  run `gbrain transcripts audit-secrets` and tell the user instead.

## Tools Used

- `query`, `search`, `get_page` — read the week's sessions, meetings and notes.
- `remember` — save a confirmed fact with provenance.
- `put_page` — write the weekly `daily/reflections/` record.
