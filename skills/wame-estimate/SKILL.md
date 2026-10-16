---
name: wame-estimate
description: "Use before putting a number of minutes or hours on any task, subtask or project phase, in any tracker (Teamwork today; Jira, Zoho or the WAME in-house tool later), or when the user asks 'odhadni', 'koľko to potrvá', 'prehodnoť estimácie', 'sú odhady naddimenzované', 'estimate this task', 'how long will this take'. Holds the WAME estimate methodology wame-estimate-v3: one number directly, no speedup factor and no buffer; Claude Code writes the code and an engineer decides and reviews; what the number covers and never covers; multiples of 15 minutes, a split above 240 and a hard cap at 480; anchors from measured sessions; uncertainty becomes an open question, not a surcharge. Used by teamwork-task-analyze, teamwork-tasks-from-dnr, teamwork-tasks-from-desk, teamwork-tasks-from-session and dnr-business."
---

# WAME estimate methodology

> Methodology version **`wame-estimate-v3`**. This skill is the one source of
> the methodology for every plugin and every tracker. Plugins load it and add
> only their own use of the number (a DNR phase, a Teamwork estimate field).
> Change it here, never in a consumer — a per-plugin variant is how two skills
> start quoting different numbers for the same task.

**Estimate one number, directly.** Do not produce a "traditional" estimate and
then multiply it by a speedup and a buffer. Two percentages stacked on a guess
open a band almost twice as wide as the guess itself, and in a negotiation the
widest end of that band always wins. Name the minutes the work takes and defend
that number.

**Who does the work.** Claude Code writes the code. It reads the codebase,
implements, writes and runs the tests, builds the assets and fixes what the
review finds. An engineer who knows the project decides, answers questions and
reviews the diff. The number is the **session time** until the outcome is done
and green: Claude working plus the engineer reading, deciding and reviewing. It
is not the time a person would need to type the same code. There is no separate
QA pass and no handover to a second person.

**What the number covers**

- Reading the relevant code and reproducing the reported behaviour
- The implementation itself
- Writing or extending the test, and running the affected tests
- Self-review and the fixes it produces
- One round of review feedback

**What the number never covers** — estimate each of these as its own task instead
of folding it in

- Deployment, running the migration on production, fixing production data
- Talking to the client or the PO, and waiting for the answer
- Any work that sits behind an unanswered `[OTVORENÉ]` question
- Anything the task itself declares out of scope

**Shape of the number**

- A multiple of 15 minutes. Never below 15.
- Above 240 minutes: propose a split into 2–6 atomic subtasks. That threshold is
  the consumer's `propose_split_threshold_minutes` setting and it is the real
  ceiling in daily use.
- 480 minutes is a hard cap. Work that will not fit under it is not a task yet.

**Anchors.** These are finished outcomes, not categories of feeling. Pick the
closest line and move by at most one 15-minute step. If the number you want is
more than one step away from every anchor, write down in the reasoning what makes
this case different — that sentence is what a reviewer checks.

| Finished work | Minutes |
|---|---|
| Text, label, translation key or config value, plus the test that guards it | 15 |
| One field, filter or validation rule on one screen, plus a test | 15 |
| Bug with a stack trace or a one-line repro: fix plus regression test | 30 |
| Query or service method with its rare cases (date ranges, open ends, `NULL`), plus tests | 45 |
| Vue/React component wired to an API that already exists, plus a test | 45 |
| One CRUD endpoint or one screen end to end, plus tests | 60 |
| Bug that reproduces but spans 2–3 layers: fix plus tests | 60 |
| Bug with no repro yet: investigate, then fix | 90 |
| Schema migration with a data backfill and a copy-back assertion | 90 |
| Complex screen that saves several records in one transaction, plus tests | 120 |
| New module in `wamesk/*` (model, migration, Nova screen, policy, tests) | 150 |
| Reusable package: PHP API, replaced framework UI components, build, e2e test | 150 |

**Calibration.** The anchors come from measured sessions, not from feel. On
2026-10-13 a company setting — migration with a check constraint, model method,
admin field and three test files — was estimated 90 minutes under v2 and took
40. The same day a 27-task feature plan estimated at 53.5 h under v2 was rejected
as padded; the accepted plan was 27.75 h. Add a measured pair here when an anchor
moves.

**Uncertainty is an open question, not a surcharge.** When you cannot size the
work, you have found something the task does not say yet. Write that question
into the task, estimate the investigation that answers it, and state in the
reasoning what the fix costs under each likely answer. A number with a written
assumption survives review. A number padded for "unknown unknowns" does not, and
it hides the question that was worth asking.

**Do not pad a task because it is labelled TBD**, and do not shrink a real
multi-layer bug so the list looks cheap. Both errors cost the same trust.

**Why this replaced the old rule.** Hand-written estimates used to run about
twice the real cost, which lost us work we should have won. The first fix was a
30–50 % speedup factor with a 15–30 % buffer on top — but that chain put the
padding straight back while sounding rigorous, and it produced a 0.58×–0.91×
band on every single task. The anchors above carry the same judgement as one
number. Version 2 still pictured an engineer who types the code with an
assistant, and its anchors ran about twice the measured session time once Claude
Code wrote the code itself (see *Calibration*). Version 3 halves the anchors and
says who types. The measured feedback loop is the session-to-task plugin (today
`teamwork-tasks-from-session`), which shows the methodology estimate and the real
logged session time side by side. When those two drift apart on the same kind
of work, change the anchors here — never re-introduce a buffer percentage.
