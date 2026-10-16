---
name: wame-time-logging
description: "Use before writing any work-time entry to a task tracker (Teamwork today; Jira, Zoho or the WAME in-house tool later), or when the user asks 'zaloguj čas', 'loguj od 13:00', 'pravidlá logovania času', 'ako logujeme čas', 'time logging rules', 'log my time'. Holds the WAME time-logging rules, independent of any tracker: every entry is whole 5-minute blocks on the 5-minute grid, never a 1–4 minute fragment; tasks shorter than one block are merged into one shared entry on the task with the most work, and its description names every task; a leftover of even 1 minute is logged as 5 minutes; entries never overlap and never end after the current 5-minute block; a start time given by the user is split across the tasks by their estimates. Tracker plugins (teamwork-task, teamwork-task-test, teamwork-tasks-from-session) load it and add only their API details."
---

# WAME time-logging rules

Rules version **`wame-time-logging-v1`**. This skill is the one source of the
time-logging rules for every plugin that writes time, whatever the tracker. A
tracker plugin implements these rules and adds only what is specific to its
tracker: endpoints, field names, time zones, authentication. Change a rule here,
never in a consumer — two plugins with two rule sets put two different numbers
on the same timesheet.

## Terms

| Term | Meaning |
|---|---|
| **Block** | 5 minutes. A consumer may read it from its config (`time_rounding_minutes`); the default and the WAME standard is 5. |
| **Grid** | Clock times that are a whole number of blocks after midnight: 10:00, 10:05, 10:10. |
| **Entry** | One time record on one task: date, start, minutes, description. |
| **Measured minutes** | Wall-clock minutes of one finished unit of work (the implementation of a task, the QA of a task, a session), rounded **up** to whole minutes. The consumer says where the unit starts and ends. Time spent waiting for the user to approve a plan is not measured. |
| **Cursor** | The time where the next entry starts. |
| **Limit** | The end of the block that contains *now*: `ceil(now, block)`. When *now* is on the grid, the limit is *now*. |
| **Run** | One invocation of a consumer that handles one or more tasks. |
| **Bundle** | The finished work of the run that has no entry yet: tasks, their measured minutes and one line on what was done. |

## Hard rules

1. **Every entry is whole blocks** — 5, 10, 15 … minutes. Never 1–4 minutes,
   never 7, never 13. This holds whatever an older plugin config says
   (`round_threshold_minutes`, `min_log_minutes` and other sub-block settings
   are obsolete and ignored).
2. **Every entry starts on the grid.**
3. **Entries never overlap** — not each other, and not any entry of the same
   person that already exists that day. Another session can write while this
   one works, so re-read the person's last entry of the day right before every
   write.
4. **No entry ends after the limit.**
5. **No finished work disappears silently.** Every finished task ends up in an
   entry (its own or a shared one) or is named in the run summary with the
   reason.
6. **Only finished work is logged.** The consumer decides what finished means
   (a commit was made, a QA report was written). Analysed-only tasks, tasks the
   user aborted and failed commits get nothing.

## 1. Cursor

- At the start of the run, after the user approved the plan: cursor = the end of
  the person's last entry today, moved **up** to the grid. With no entry today,
  cursor = `floor(now, block)`.
- If that end is after the limit (somebody logged ahead of the clock), the
  cursor still starts there — rule 3 wins. Nothing fits until real time catches
  up, so the work waits in the bundle (section 3).
- After a successful write: cursor += the entry's minutes. After a failed write
  the cursor does not move; report the failure and do not retry blindly.
- When the user gives a start time, section 5 replaces this section.

## 2. Bundle — work shorter than one block

After each finished task:

1. Add the task to the bundle (task, measured minutes, one line on what was done).
2. If the bundle total is **below one block** → hold. No entry yet. The task
   waits; a consumer that moves a task to a "done" column after its entry moves
   it when the bundle is written.
3. If the bundle total is **one block or more** → write the bundle as one entry
   now (section 3).

At the **end of the run**, a bundle that is not empty is written as one entry of
**one block (5 minutes)** — even when it holds 1 minute of work.

So a run never writes a 1–4-minute entry, and never writes 5 minutes for each of
several quick tasks that together fit in one block.

## 3. Writing an entry

- **Minutes.** Bundle total of one block or more → the nearest whole block,
  half up, in integer arithmetic: `((total + block/2) / block) * block`
  (5 → 5, 7 → 5, 8 → 10, 12 → 10, 13 → 15). End-of-run bundle below one block
  → one block.
- **Room.** `room = floor((limit − cursor) / block) * block`.
  - room ≥ minutes → write the minutes.
  - 0 < room < minutes → write `room` and report the cut
    (`20 → 15 min — the cursor reached the current block`).
  - room = 0 → write nothing now. The bundle stays open with all its tasks; try
    again after the next task or at the end of the run.
- **Owner.** The entry goes on the task of the bundle with the most measured
  minutes; on a tie, on the task finished last.
- **Start** = cursor. **Date** = the cursor's local date.
- **Description** — section 4.
- After the write the bundle is empty, and every task that was in it counts as
  logged.

**End of the run with room = 0.** Add the bundle's tasks to the description of
the last entry this run wrote (edit the description only, the minutes stay) and
say so in the summary. If this run wrote no entry, write nothing and name the
tasks in the summary with the reason (`the timesheet is full until 10:15`).

## 4. Description

- Language: the consumer's report language (WAME default: Slovak). Business
  tone — what changed for the user, not how it was built. One or two sentences
  per task.
- **One task in the entry** — the consumer's usual description.
- **Two or more tasks in the entry** — a shared entry. The first line says so
  and gives the real work time, then one line per task, the owner first, the
  rest in the order they were done:

  ```
  Spoločný záznam za 3 úlohy (spolu 6 min práce):
  #45522728 Priradenie úlohy viacerým ľuďom — otestované 4 akceptačné kritériá
  #45522670 Hashované heslo IMAP — overená akcia Test pripojenia
  #45522683 Neznámy kód poschodia — overené hlásenie v reporte importu
  ```

- **Merged into an earlier entry at the end of the run** (section 3) — append
  `Patrí sem aj: #<id> <názov> — <čo sa spravilo>` lines to that entry's
  description.

## 5. The user gives a start time

When the user says when the work started (`loguj od 13:00`, `čas od 13:00`):

1. Write nothing per task. Record each task's measured minutes and write
   everything after the last task is finished.
2. **Span** = from the given start, moved up to the grid, to the limit.
   **Blocks** = span / block. If an entry of the person already lies inside the
   span, stop and ask — the span is not free.
3. Split the blocks across the finished tasks **in proportion to their
   estimates** (a task with no estimate: its measured minutes) by largest
   remainder: every task gets `floor(share)`, the blocks left over go one each to
   the largest fractional parts (tie: the larger estimate, then the earlier task).
4. A task that gets 0 blocks has no entry. Name it in the entry with the most
   blocks, in the shared-entry format of section 4.
5. The entries follow each other from the start, in task order.
6. Show the split table — task, estimate, blocks, start–end — and write only
   after the user's OK.

Example: start 13:00, now 14:02 → limit 14:05 → 13 blocks. Estimates A 60,
B 30, C 15 → shares 7.43 / 3.71 / 1.86 → floors 7 / 3 / 1 = 11, the 2 blocks
left go to C (.86) and B (.71) → A 13:00–13:35, B 13:35–13:55, C 13:55–14:05.

## 6. One entry for a whole session

A consumer that logs a finished session (session → task): measured minutes =
from the session start, or the end of the person's last entry if that is later,
to now. Minutes and room as in section 3 — at least one block for any work of
1 minute or more. A split across subtasks follows the split of section 5.

## 7. Correcting and confirming

- Correct a misplaced entry by **editing** it (date, start, minutes,
  description). Never delete and re-create — the history of the tracker keeps
  both.
- When the user chose to confirm every duration (for example `time_mode=ask`),
  propose the value these rules give. A number the user types is final; if it
  is not whole blocks, say so once in the summary.

## 8. Run summary

At the end of the run show:

- where the cursor started and why (`last entry today ended 10:03 → 10:05`,
  `first entry today`, `user start 13:00`);
- one row per entry: start–end · minutes · owner task · other tasks in it;
- the cut entries (`before → after`, why), the tasks merged into an earlier
  entry by an edit, and the tasks with no entry and why.

## 9. Worked examples

Block 5 min, cursor 10:00, enough room unless the row says otherwise.

| Run (task: measured minutes) | Entries |
|---|---|
| A 1 | end of run: A 10:00–10:05 (5) |
| A 2, B 2, C 2 | after C (6): C 10:00–10:05 (5), shared with A and B |
| A 4, B 4 | after B (8): B 10:00–10:10 (10), shared with A |
| A 3, B 20 | after B (23): B 10:00–10:25 (25), shared with A |
| A 12 | A 10:00–10:10 (10) |
| A 13 | A 10:00–10:15 (15) |
| A 7, B 1 | A 10:00–10:05 (5); end of run: B 10:05–10:10 (5) |
| now 10:12, A 20 | limit 10:15, room 15: A 10:00–10:15 (15), cut 20 → 15 |
| cursor 10:15 (written by another session), now 10:12, A 4 | room 0: A waits; at the end room is still 0 and the run wrote nothing → no entry, A named in the summary |

A consumer's implementation is correct when it gives these rows.

## 10. What a tracker plugin adds

These belong in the tracker plugin, not here:

- How to read the person's own entries of a day. The filter must be the person
  who logged the time, not everybody on the task.
- The time zone of reads and of writes.
- How to create an entry (task, date, start, minutes, description) and how to
  edit one.
- Where run files go: a per-run directory, never a fixed `/tmp` path — two
  sessions run the same plugin at once.

Known adapter — **Teamwork** (`teamwork-task`, `teamwork-task-test`,
`teamwork-tasks-from-session`): read with `GET /projects/api/v3/time.json`
filtered by `assignedToUserIds` (`userId` silently returns the whole company);
`timeLogged` comes back in UTC, the `time` field of a write is the user's local
time; create with `POST /projects/api/v3/tasks/{id}/time.json`, edit with
`PATCH /projects/api/v3/time/{id}.json`.

## Why these rules

- **2026-10-16.** QA entries of 2–3 minutes showed up on the timesheet:
  `teamwork-task-test` reused the sub-block rule that `teamwork-task` 1.3.0
  introduced (work below 4 minutes logged as raw minutes, a cut to the exact
  free minutes). The rule now is whole blocks only; several quick tasks share
  one block with a description that names them all; a leftover of even
  1 minute is one block.
- **2026-10-05.** With a start time given, per-task logging gave the first task
  74 minutes — it absorbed the gap from the start to its own start. Given a
  start time, the whole span is split once at the end, by estimates.
- **Nearest block, not always up.** An 11-minute fix is 10 minutes, not 15. The
  rounding stays honest in both directions; only work shorter than one block
  rounds up, because a block is the smallest entry.
- **The current block counts.** With whole blocks only, a cap at *now* would
  drop every quick run that ends inside a block. Ending an entry at the end of
  the current block puts it at most 4 minutes ahead of the clock.
