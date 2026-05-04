# Ceremonies as Sessions

Most Agile ceremonies are already session-shaped. They have defined scope, a natural exit artifact, and a time box. They have just accumulated overhead that makes them feel bigger than they are — largely because the information they exist to surface was never written down anywhere else.

AI-native engineering changes that. Repo memory, design docs, FF-merge, and tags make most ceremony inputs ambient. The ceremonies do not disappear — they compress.

## Code Review

Code review is one of the biggest unsolved friction points in adopting AI-native engineering. The current model treats it as an async interrupt — a notification queue that pulls engineers out of their own sessions to context-switch into someone else's code. That is the worst possible review experience, and it does not get better just because the code was written faster.

**Review is a session, not a queue.**
A dedicated 30–60 minute review session — scoped, focused, uninterrupted — produces better signal than three fragmented 10-minute drive-bys. Budget it that way. Block the time. Treat a mis-framed or under-prepared review request the same way you treat a mis-framed session: send it back.

**FF-merge + session hygiene changes the review surface.**
If the author ends every session with green tests, a coherent diff, a design note, and a tag — the reviewer's job is fundamentally different. You are not reconstructing intent from a squashed blob. You are reviewing a session's worth of deliberate, documented work. The diff tells a story because the session was shaped to tell one.

**Repo memory as pre-read.**
If the author updated repo memory at session close, the reviewer can orient in minutes. The "what is this repo even doing" tax — which dominates most review time on unfamiliar code — largely disappears.

**The open question: who reviews, and when?**
Low-risk, well-tested, well-documented sessions may self-review via the artifact they produce. Higher-stakes arc completions warrant a dedicated review session from a second engineer. This needs experimentation, not a policy.

**What to track:**
- Review lag: time from session close to review complete
- Review session length vs. review quality (bugs caught, design feedback given)
- Re-work rate: sessions that required a follow-up session due to review feedback

## Sprint Retro → Reflection Session

The cleanest mapping. Fixed scope (what happened this arc), clear artifact (a committed change to the team's working agreement), natural time box (60–90 min).

The session version:
- Everyone pre-reads the git history, session log, and arc doc *before* the meeting
- The retro itself is pure synthesis and decision — no reconstruction, no surprises
- It ends with a written, committed process change — not a list of feelings on a board

Exit artifact: a pull request to the team's working agreement or process doc. If there is no artifact, the session did not close.

## Daily Standup → Synchronized Code Review Window

The standup's real job is "are we blocked, and does anyone need help?" A shared daily CR window does that — and produces value at the same time.

How it works:
- A fixed daily window (e.g. 9:00–9:45) where everyone is doing code review
- Blockers surface naturally — you are all in the same cognitive space at the same time
- Questions get answered on demand; bigger issues pull in the wider team immediately
- No status theater. The git log is the status.

Design question worth experimenting with: true synchronous block (everyone reviewing together, on a call) vs. shared time window (everyone reviewing independently, channel open for questions). The latter scales better across time zones.

Exit artifact: reviewed and merged (or returned) sessions from the prior day.

## Sprint Planning → Arc Framing Session

Planning is bloated today because estimating in points requires negotiation. Replace points with sessions and planning becomes:
- Here is the arc goal
- Here are the candidate sessions, in order
- Here is what done looks like
- Here is what is explicitly out of scope

One engineer + one PM, 60–90 minutes, produces a framed arc ready to execute. The rest of the team does not need to be in the room — they read the arc doc.

Exit artifact: a committed arc framing doc with goal, session list, exit criteria, and explicit out-of-scope decisions.

## Sprint Demo → Tag-Driven Release Notes

This one nearly eliminates itself. If every session ends with a tag and release notes, the "demo" is reading the changelog. The information is already ambient.

What survives:
- A monthly or arc-boundary stakeholder demo still makes sense for visibility
- The weekly "show what you built" meeting becomes redundant when artifacts speak for themselves

Exit artifact: the release notes are the artifact. The demo, if it happens, is a presentation of what is already written — not a substitute for writing it.

## Office Hours — Connective Tissue Between Sessions

Office hours are not a ceremony. They are the intentional space where cross-session questions get resolved without breaking anyone's flow.

**Dev team office hours.**
A standing weekly session — open, time-boxed (60 min), no agenda required. The critical property: attendance is *pull*, not *push*. If your sessions are going cleanly and repo memory is current, you may not need it that week. But it exists as a safety valve for:
- Cross-session blockers that need a second brain
- Architecture questions that span more than one engineer's arc
- "I've been stuck for two sessions" — the signal that something needs to be talked through, not just coded through

Optional attendance is what gives it value. A week where nobody shows up is a good week, not a failed meeting.

**Stakeholder office hours.**
A shorter standing window (30 min, weekly or bi-weekly) that gives business stakeholders a predictable on-ramp without ceremony overhead. The problem it solves: right now stakeholders either wait for a demo or interrupt engineers ad hoc. Both are bad. A known open window decouples "can I ask a question" from "is there a scheduled review."

**The session properties hold:**
- Fixed time box
- Pull-based attendance
- Clear purpose
- Light exit artifact: a brief note of what was discussed and any decisions made

Office hours attendance is itself a health signal. Heavy attendance means sessions are not well-framed or blockers are not surfacing early enough. Light attendance means the system is working.

## The Underlying Pattern

Ceremonies exist to compensate for information that is not written down.

- Standups exist because nobody knows what anyone else is doing
- Planning is long because intent was never documented
- Retros are fuzzy because the arc was not framed clearly at the start
- Demos are required because there are no release notes

AI-native engineering + session hygiene makes most of that information ambient *before* the ceremony starts. The result is not fewer ceremonies — it is ceremonies that actually fit in a session, because the pre-work is already done.

**Hypothesis worth testing:** a team running full session hygiene should be able to cut total ceremony time by 50–70% within one quarter, with no loss of coordination or visibility — and measurable gains in both.
