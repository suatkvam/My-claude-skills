---
name: decision
description: Record a new decision in the decision log, update the PRD to match, have an independent reviewer subagent check the change, and report
argument-hint: "<the decision or change> [--prd <path>] [--decisions <path>] [--lang <language>]"
disable-model-invocation: true
---

The PRD is a living document and the decision log explains why it changed. Apply this change:

$ARGUMENTS

## Arguments

The text above is the change to record. It may also contain these options; remove them from
the change text before you use it:

- `--prd <path>`: the PRD file. Default `docs/PRD.md`.
- `--decisions <path>`: the decision log. Default `docs/DECISIONS.md`.
- `--lang <language>`: the language of your final report. Default: the language the user
  writes in.

If the change text is empty, ask what to record and stop.

## Find the documents

1. Use a path given as an option as it is.
2. Otherwise use the default path if the file exists.
3. Otherwise look for the project's own location: check `CLAUDE.md`, `AGENTS.md` and the
   README for a named PRD or decision log, then search for files such as `PRD*.md`,
   `*requirements*.md`, `DECISIONS*.md` and `decision-log*.md`. If exactly one candidate fits,
   use it and say so in the report. If several fit, ask which one.
4. If there is no decision log, create one at the default path (see "Starting a new log").
5. If there is no PRD, do not invent one. Record the decision, skip steps 3 to 5 of the
   process, and say in the report that no PRD was found.

## Decision log format

If the log already exists, follow its format exactly: heading level, field names, number
width, order. Use the format below only for a new log.

```markdown
## D-07 — Short title of the decision

- **Date:** 2025-01-31
- **Status:** Accepted
- **Decision:** What was decided, in one to three sentences that can be read alone.
- **Why:** The reason, and the alternatives that were rejected.
- **Affects:** The PRD sections this changes.
```

Numbering rules:

- The identifier is `D-` and a number. The next number is the highest number in the log plus
  one. Keep the width the log already uses (`D-07`, then `D-08`).
- Numbers are never reused, renumbered or skipped. Entries are in ascending order, newest last.
- An entry is never deleted or rewritten. To reverse a decision, add a new entry and change
  the old entry's status to `Superseded by D-xx`.
- A change that only makes an existing decision more precise is a clarification, not a new
  decision: add a dated `- **Clarified <date>:** ...` line to that entry and use no new number.
- Other documents refer to a decision by its identifier, for example `(D-07)`.

### Starting a new log

Create the file with this header, then add the first entry as `D-01`:

```markdown
# Decisions

Append-only log of project decisions. Newest last. Numbers are never reused or renumbered.
The PRD states what is true now; this log states why.
```

## Process

0. Work in the main working tree on the default branch. If the current directory is a linked
   git worktree or another branch is checked out, stop and tell the user, unless the change
   text says to work there.
   Then keep the text from before the change, so that the reviewer can compare: copy the PRD
   and the decision log, whichever of them exist, as they are now to a temporary directory
   outside the project. A file that does not exist yet has no earlier text; remember that
   instead. If no review runs, delete the copies before you finish.
1. Read the whole decision log and the PRD sections that the change affects.
2. Record the change in the decision log as the next number, or as a clarification of an
   existing decision if that fits better.
3. Update every PRD section the decision affects. Depending on the PRD these can be:
   requirements, domain model, architecture, events, security, scope and MVP lists, success
   criteria, risks, build order, sources. Cite the decision identifier where the PRD changes.
   Write in the language the documents already use.
   Record only PRD-level content: user-visible behavior, security policy and contracts between
   modules. Leave implementation details to the tasks that implement them.
4. Review: use the Agent tool to run the `dev-workflow:reviewer` subagent. Give it the PRD
   path, the decision log path, the identifier of the new or changed decision and the list of
   PRD sections you changed. Also give it the paths of the copies from step 0 as the text
   before the change, and for a file that this run created, say that it has no earlier text.
   The reviewer cannot run commands, so it sees the earlier text only if you give it. Never
   review the change yourself. Fix the findings that are correct; do not act on the ones you
   reject, and say why in the report. Delete the copies when the review is done.
5. Update the PRD header with today's date and the latest decision identifier. If the PRD has
   no such header line, add `Last updated: <date> — latest decision: D-xx` under the title.
6. Before you finish, read the decision log again and check that the text of the new entry is
   really in it and that the numbers continue without a gap or a duplicate.
7. Report in the output language:
   - what changed in the decision log and in the PRD;
   - the reviewer's findings, separate from the fixes you made;
   - every question that needs the user's decision;
   - the reviewer's "Task-level" items in a separate list. They are not questions for the
     user; they belong in the descriptions of the tasks they affect.

Do not write application code unless the change text explicitly asks for it. Do not commit.
