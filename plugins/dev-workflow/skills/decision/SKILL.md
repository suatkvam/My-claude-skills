---
name: decision
description: Record a new decision in the decision log, update the PRD to match, have an independent reviewer subagent check the change, and report
argument-hint: "<the decision or change> [--prd <path>] [--decisions <path>] [--lang <language>] [--here]"
disable-model-invocation: true
---

The PRD is a living document and the decision log explains why it changed. Apply this change:

$ARGUMENTS

## Arguments

The text above is the change to record. It may end with the options below. Recognise an option
only at the end, after the change text. A word such as `--prd` inside the change text is part
of the decision: keep it. Remove the options at the end from the change text before you use it.

- `--prd <path>`: the PRD file. Default `docs/PRD.md`.
- `--decisions <path>`: the decision log. Default `docs/DECISIONS.md`.
- `--lang <language>`: the language of your final report. Default: the language the user
  writes in.
- `--here`: work in the current branch or worktree (see step 0 of the process).

A value that contains a space must be in quotes: `--prd "docs/My PRD.md"`,
`--lang "Brazilian Portuguese"`. A value without quotes is one word. If you cannot tell where
a value ends or where the change text ends, ask and stop.

If the change text is empty, ask what to record and stop.

## Find the documents

Paths are relative to the current working directory. In every search, ignore the case of file
names and skip dependency and build directories (`.git`, `node_modules`, `vendor`, `dist`,
`build`, `target`, virtual environments).

1. Use a path given as an option as it is. If `--prd` names a file that does not exist, say so
   and stop. If `--decisions` names a file that does not exist, ask whether to start a new log
   at that path and stop until the user answers.
2. Otherwise, if `CLAUDE.md`, `AGENTS.md` or the README names a PRD or a decision log, use
   that file.
3. Otherwise use the default path if the file exists. Check it without regard to case:
   `docs/decisions.md` is the decision log.
4. Otherwise search. PRD: `PRD*.md`, `*requirements*.md`. Decision log: `DECISIONS*.md`,
   `decision-log*.md`, and a directory named `adr`, `adrs` or `decisions` that holds one file
   per decision. If exactly one candidate fits, use it. If several fit, ask which one.
5. If no decision log was found, do not create one silently: tell the user where you looked
   and ask whether to start a new log at the default path (see "Starting a new log"). If the
   project has no `docs/` directory but has `doc/` or `documentation/`, propose that directory
   instead of a new `docs/`.
6. If there is no PRD, do not invent one. Record the decision, skip steps 3 to 5 of the
   process, and say in the report that no PRD was found.

Say in the report which PRD and which decision log you used.

## Decision log format

If the log already exists, it is the authority on its own format. Follow it exactly: heading
level or table columns, field names, identifier prefix, number width, and order (newest first
or newest last). A directory with one file per decision is also a log: the new entry is a new
file named and written like the others.

Take the next number from the log: the highest number among the entry identifiers plus one.
A log that has its header but no entries yet (for example one written by the `prd` skill when
the sources decided nothing) starts at `D-01` in the format of "Starting a new log".
Entry identifiers are the entry headings, the first column of a table, or the file names; a
number that is only mentioned inside the text of an entry does not count. Find the highest
identifier with Grep over the whole log, not from what you read: a read of a long file can be
cut off before the newest entries. If the entries have no sequential number (for example they
are identified by date), ask how to identify the new entry and stop. Gaps and duplicates that
are already in the log are history: leave them as they are.

These rules hold for every log:

- Numbers are never reused or renumbered.
- An entry is never deleted or rewritten. To reverse a decision, add a new entry and mark the
  old entry as superseded by the new one, in the way the log's format allows.
- A change that only makes an existing decision more precise is a clarification, not a new
  decision: add a dated clarification to that entry, in the way the log's format allows, and
  use no new number.
- Other documents refer to a decision by its identifier, for example `(D-07)`.

### Starting a new log

The format and the rules in this section are for a new log only. Create the file with this
header, then add the first entry as `D-01`. Write the header, the field names and the status
values in the language of the PRD (in English if there is no PRD); the text below shows the
English form:

```markdown
# Decisions

Append-only log of project decisions. Newest last. Numbers are never reused or renumbered.
The PRD states what is true now; this log states why.
```

```markdown
## D-07 — Short title of the decision

- **Date:** 2025-01-31
- **Status:** Accepted
- **Decision:** What was decided, in one to three sentences that can be read alone.
- **Why:** The reason, and the alternatives that were rejected.
- **Affects:** The PRD sections this changes.
```

- The identifier is `D-` and a two-digit number (`D-07`, then `D-08`). Numbers are not skipped.
- Entries are in ascending order, newest last.
- A reversed decision gets the status `Superseded by D-xx`.
- A clarification is a line `- **Clarified <date>:** ...` in the entry.

## Process

0. Check where you are. Skip this check if `--here` was given, if the project is not a git
   repository, if the repository has only one local branch, or if the default branch cannot be
   determined (`git symbolic-ref --short refs/remotes/origin/HEAD` prints nothing). Otherwise,
   if the current directory is a linked git worktree (`git rev-parse --git-dir` and
   `git rev-parse --git-common-dir` differ) or the checked-out branch is not the default
   branch, stop and tell the user that `--here` allows the run there.

   Then keep the text from before the change, so that the reviewer can compare:
   - Remove what an interrupted earlier run left behind:
     `find "${TMPDIR:-/tmp}" -maxdepth 1 -name 'dev-workflow-decision.*' -mmin +60 -exec rm -rf {} +`
   - Make a new directory: `mktemp -d "${TMPDIR:-/tmp}/dev-workflow-decision.XXXXXX"`. Without
     a shell that has `mktemp`, make a directory with a random name in the system's temporary
     directory.
   - Copy the PRD and the decision log, whichever of them exist, into it as they are now. A
     file that does not exist yet has no earlier text; remember that instead.
   - Tell the user the path of the directory, so that it can be deleted by hand if the run is
     interrupted.

   From here on, delete the directory before you stop for any reason, also when no review runs.
1. Read the whole decision log, in parts if it is long, and the PRD sections that the change
   affects.
2. Record the change in the decision log as the next number, or as a clarification of an
   existing decision if that fits better.
3. Update every PRD section the decision affects. Depending on the PRD these can be:
   requirements, domain model, architecture, events, security, scope and MVP lists, success
   criteria, risks, build order, sources. Cite the decision identifier where the PRD changes.
   Write in the language the documents already use.
   Record only PRD-level content: user-visible behavior, security policy and contracts between
   modules. Leave implementation details to the tasks that implement them.
4. Review. First make the diff: run `diff -u <copy> <current file>` for the PRD and for the
   decision log. Then use the Agent tool to run the `dev-workflow:reviewer` subagent. Put in
   its prompt: the PRD path, the decision log path, the identifier of the new or changed
   decision, the list of PRD sections you changed, and the complete `diff -u` output itself.
   For a file that this run created, say that it is new and has no earlier text. The reviewer
   cannot run commands and may not be able to read outside the project, so it sees the earlier
   text only through the diff in its prompt. Never review the change yourself.
   Fix the findings that are correct; do not act on the ones you reject, and say why in the
   report. If the reviewer cannot run (no Agent tool, an error, a denied permission), do not
   replace it: say in the report that the change was not reviewed.
   Delete the temporary directory when the review is done or has failed.
5. Update the PRD header with today's date and the latest decision identifier. Use the date
   field the PRD already has, for example `updated:` in its frontmatter or a "Last updated"
   line, and do not add a second one. Only if the PRD has no date field at all, add
   `Last updated: <date> — latest decision: D-xx` under the title. For a clarification, update
   only the date: the latest decision identifier stays as it is.
6. Before you finish, check with Grep that the text of the new entry is really in the decision
   log, that its number is the previous highest number plus one, and that no other entry has
   the same identifier.
7. Report in the output language:
   - which PRD and decision log were used, and what changed in them;
   - the reviewer's findings, separate from the fixes you made, or that no review ran;
   - every question that needs the user's decision;
   - the reviewer's "Task-level" items in a separate list. They are not questions for the
     user; they belong in the descriptions of the tasks they affect.

Do not write application code unless the change text explicitly asks for it. Do not commit.
