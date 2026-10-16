# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [1.0.0] - 2026-10-16

### Added

- **`wame-time-logging` skill — rules `wame-time-logging-v1`.** One tracker-independent
  source for every plugin that writes time. Whole 5-minute blocks on the 5-minute grid
  only; work of one block or more rounds to the nearest block; tasks shorter than one
  block are merged into one shared entry on the task with the most work, with a
  description that names every task; a leftover of even 1 minute is one block; entries
  never overlap and never end after the current 5-minute block; a user-given start time
  is split across the tasks by their estimates (largest remainder). Worked examples that
  every implementation must reproduce, and the list of what a tracker plugin adds (the
  known Teamwork details included).
- **`wame-estimate` skill — methodology `wame-estimate-v3`.** The block that
  `teamwork-task-analyze`, `teamwork-tasks-from-dnr`, `teamwork-tasks-from-desk`,
  `teamwork-tasks-from-session` and `dnr-business` used to copy verbatim, moved here
  unchanged in substance; those plugins now load it.
