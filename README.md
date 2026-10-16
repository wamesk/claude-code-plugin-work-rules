# work-rules

WAME work rules that do not depend on any task tracker, by [WAME](https://wame.sk).
Part of the `wame` marketplace.

## Why

The rules for logging time and for estimating work used to live inside the Teamwork
plugins — the estimate methodology as a block copied verbatim into five plugins, the
time-logging rules spread over three. We are moving away from Teamwork (Jira, Zoho or a
WAME in-house tool), and the rules must move with us unchanged. This plugin holds them
once, without a word about any tracker's API. A tracker plugin loads the skill and adds
only its own endpoints, field names and time zones.

| Skill | Rules | What it decides |
|---|---|---|
| `wame-time-logging` | `wame-time-logging-v1` | How work time becomes timesheet entries |
| `wame-estimate` | `wame-estimate-v3` | How many minutes a task, subtask or project phase gets |

## Time logging in short

- Every entry is **whole 5-minute blocks** and starts on the 5-minute grid. No 1–4 minute
  fragments, no 7 or 13 minutes.
- Work of one block or more rounds to the **nearest** block (7 → 5, 8 → 10, 12 → 10).
- **Tasks shorter than one block are merged.** They wait in a bundle until the bundle
  reaches one block; then one shared entry goes on the task with the most work, and its
  description names every task and what was done on it.
- A leftover at the end of a run — **even 1 minute** — is logged as one block (5 minutes).
- Entries never overlap (each other or anything the person already logged that day) and
  never end after the current 5-minute block.
- When the user gives a start time (`loguj od 13:00`), nothing is logged per task; at the
  end the whole span is split across the tasks **by their estimates**.

The full rules, with worked examples that any implementation must reproduce, are in
[`skills/wame-time-logging/SKILL.md`](skills/wame-time-logging/SKILL.md).

## Estimates in short

One number directly — no speedup factor, no buffer. Claude Code writes the code; an
engineer decides and reviews. Multiples of 15 minutes, a split proposed above 240, a hard
cap at 480, anchors from measured sessions. Full text:
[`skills/wame-estimate/SKILL.md`](skills/wame-estimate/SKILL.md).

## Who uses it

`teamwork-task`, `teamwork-task-test`, `teamwork-task-analyze`, `teamwork-tasks-from-desk`,
`teamwork-tasks-from-dnr`, `teamwork-tasks-from-session` and `dnr-business` declare it in
`dependencies`, so installing any of them installs this plugin too.

## Installation

```
/plugin marketplace add wamesk/claude-code
/plugin install work-rules@wame
```

You can also ask for the rules directly: *„aké sú pravidlá logovania času"*,
*„odhadni tento task"*.

## Changing a rule

Change it here, bump the version, then bump the consumers only if their own text changes.
Never copy a rule into a consumer — a per-plugin variant is how two plugins start
writing two different numbers for the same work.

## License

MIT
