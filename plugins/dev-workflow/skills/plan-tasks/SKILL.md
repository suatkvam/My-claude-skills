---
name: plan-tasks
description: Split one build step, milestone or set of requirements of the PRD into session-sized tasks in the project's task list, with references, dependencies, expected files and testable acceptance criteria, numbered without collisions across branches and worktrees
argument-hint: "<build step, milestone or requirement IDs> [--prd <path>] [--decisions <path>] [--tasks <path>] [--lang <language>] [--here]"
disable-model-invocation: true
---

Plan tasks for:

$ARGUMENTS

## Arguments

The text above names what to plan: a build step or milestone of the PRD (for example
"step 3" or "Milestone 2"), one or more requirement identifiers (`FR-12 FR-13`), or a short
description of a part of the PRD. After that it may hold notes for the planner, such as the
"Task-level" items that the `decision` or `prd` report listed: put each one into the task it
concerns.

The text may end with the options below. Recognise an option only at the end, after the
text. A word such as `--prd` inside the text is part of the text: keep it.

- `--prd <path>`: the PRD.
- `--decisions <path>`: the decision log.
- `--tasks <path>`: the task list.
- `--lang <language>`: the language of your final report. Default: the language the user
  writes in.
- `--here`: work in the current branch or worktree (step 0).

A value that contains a space must be in quotes: `--tasks "docs/My Tasks.md"`. A value without
quotes is one word. If the text is empty, matches nothing in the PRD, or matches several parts
and you cannot tell which one is meant, ask and stop.

"Step N" or "Milestone N" refers to the PRD section titled Build order, Milestones, Roadmap or
the like (or its translation). If the PRD has no such section, ask what the step means.

## Find the documents

Paths are relative to the current working directory. In every search, ignore the case of file
names and skip dependency and build directories (`.git`, `node_modules`, `vendor`, `dist`,
`build`, `target`, virtual environments).

- PRD and decision log: a path given as an option; otherwise a file that `CLAUDE.md`,
  `AGENTS.md` or the README names; otherwise `docs/PRD.md` and `docs/DECISIONS.md`; otherwise
  a search for `PRD*.md`, `*requirements*.md`, `DECISIONS*.md`, `decision-log*.md`, or a
  directory named `adr`, `adrs` or `decisions`. If several candidates fit, ask. If there is no
  PRD, stop: tasks are planned from a PRD; suggest `/dev-workflow:prd`. A missing decision log
  is not a reason to stop; say in the report that there was none.
- Task list: a path given as an option; otherwise a file that `CLAUDE.md`, `AGENTS.md` or the
  README names; otherwise `docs/TASKS.md`; otherwise a search for `TASKS*.md`, `TODO*.md`,
  `BACKLOG*.md`. If none exists, ask whether to create one at `docs/TASKS.md` (or in an
  existing `doc/` or `documentation/` directory if the project has no `docs/`) and stop until
  the user answers.
- Project rules: `CLAUDE.md` and `AGENTS.md` at the root and in subdirectories, and any
  document they point to about tasks, ownership, module boundaries, language or decision
  levels.

## Task format

If the task list has a template section (a block marked as the template for new tasks), that
template is the authority. Otherwise, if it has tasks, the newest tasks are the authority.
Follow the format exactly: identifier prefix and number width, headings or table columns,
field names, status values, owner and agent values, and where new entries go. Fill every field
the format has.

The content of "Process" step 5 is required. If the format has no field for part of it, put
that part into the closest field or into the task's body text. Never add new fields to an
existing format.

Only for a new task list, use this format:

```markdown
# Tasks

Tasks are planned from the PRD with /dev-workflow:plan-tasks. Identifiers are never reused.
Status: todo, in-progress, blocked, done.
```

```markdown
## T-001 — Short title

- **Status:** todo
- **Owner:** ai
- **PRD:** the sections and requirement IDs this implements
- **Decisions:** the decision IDs it must follow
- **Depends on:** task IDs, or none
- **Files:** the files or modules it is expected to create or change
- **Acceptance:** testable criteria, each one checkable by a test or a command
- **Out of scope:** what this task must not do
```

`Owner` is `ai` or `human` unless the project defines other values. New tasks go at the end.

Language: tasks added to an existing list use the language of that list. A new list uses the
language convention in the project rules (for example "content in Turkish, headings in
English"), otherwise the language of the PRD. This covers the header, the field names and the
task text.

## Numbering

The next identifier is the highest identifier in use anywhere plus one, because tasks are often
planned on one branch while others are open. Only entry identifiers count: the task headings,
or the first column of a table. An identifier that is only mentioned (in "Depends on", in the
text) or that is an example inside the template does not count.

Look in these places:

1. The task list in the current directory, including uncommitted changes. Use Grep.
2. Every worktree in `git worktree list --porcelain`, including uncommitted changes. Skip
   worktrees marked `prunable` or whose directory is missing. Use Grep on their task list.
3. Every local and remote branch from
   `git for-each-ref --format='%(refname:short)' refs/heads refs/remotes`, skipping
   `*/HEAD`. On each one, find the task list with
   `git ls-tree -r --name-only <ref>` (match the file name without regard to case; the file
   may have moved), then search it in Bash with
   `git show <ref>:<path-from-repository-root> | grep -E '<pattern>'`. Paths in `git show`
   are from the repository root, not from the current directory.

Say in the report which places were searched, every ref whose task list is at a different
path, that remote branches are only as recent as the last fetch (do not fetch without asking),
and if the clone is shallow or single-branch (`git rev-parse --is-shallow-repository`), that
some branches may be missing. If the project is not a git repository, use the current file
only. Never reuse or renumber an identifier, also when a task was deleted.

## Process

0. Check where you are. Skip this check if `--here` was given, if the project is not a git
   repository, if the repository has only one local branch, or if the default branch cannot be
   determined (`git symbolic-ref --short refs/remotes/origin/HEAD` prints nothing). Otherwise,
   if the current directory is a linked git worktree (`git rev-parse --git-dir` and
   `git rev-parse --git-common-dir` differ), if HEAD is detached, or if the checked-out branch
   is not the default branch, stop and tell the user that `--here` allows the run there. The
   task list is usually planned on the default branch, where every branch can see it.
1. Find the documents. Read the part of the PRD that the text names, every requirement and
   section it references, the decisions those cite, and the project rules. When a decision is
   marked `Superseded by D-xx`, follow it to the entry that replaced it and cite that one.
   Read the whole task list, in parts if it is long. Note the PRD's "Last updated ... latest
   decision" line or date field.
2. Look for existing work: tasks on the current list and on every branch and worktree from
   "Numbering" whose title or PRD references match what you are about to plan. If you find
   likely duplicates, list them and ask before you write.
3. Split the work into tasks that one agent can finish in one working session: roughly one
   module, interface, endpoint or screen each, with its tests. A task that needs more than a
   few files of new code, or two unrelated changes, is too big: split it. A task that cannot be
   tested on its own is badly cut: merge or recut it.
4. Order and assign. Contracts and interfaces that other tasks depend on come first, then the
   parts that implement or use them; number the tasks in that order and record every
   dependency. If the project rules say who owns what, follow them. If they separate kinds of
   work (for example a skeleton with tests by one owner and the implementation by another),
   make each kind its own task with its own owner. Otherwise use `ai`, and `human` for work
   that needs the owner's judgment rather than implementation: product wording and prompts,
   evaluation data and thresholds, security policy, and anything the PRD leaves to the owner.
5. For every task, give: the PRD references; the decisions it must follow; dependencies; the
   files or modules it is expected to touch; acceptance criteria that a test or a command can
   check ("returns 404 for an unknown id", not "works correctly"); what is out of scope; and
   the PRD version it was planned against (from step 1).
6. Do not decide product questions. If the PRD leaves a PRD-level question open (user-visible
   behavior, security policy, a contract between modules) and a task depends on it, do not
   pick an answer: plan the task as blocked on that question (with the `blocked` status, or, if
   the format has none, a line "Waits for: <question>" in its dependency field), or leave it
   out, and list the question in the report. Implementation details (internal data
   structures, limits, library choices the PRD does not fix) are not questions: say in the
   task that the implementer decides them.
7. Write the new tasks into the task list where its format puts new entries (at the end, at the
   top of a newest-first list, or in the matching section of a grouped list). Change no
   existing task.
8. Check that every new identifier is unique across all the places in "Numbering" and that
   the new identifiers are consecutive from the highest one found. A gap in this file is
   expected when the highest identifier is on another branch; say so in the report.
9. Report in the output language:
   - the documents used and the PRD version planned against;
   - the places searched for identifiers, and their limits;
   - the new tasks as a short table: identifier, title, owner, depends on;
   - the PRD-level questions that blocked clean planning, and which tasks wait for them;
   - anything unclear in the PRD that did not block, separately.

Do not write application code. Do not change the PRD or the decision log: a gap in them is a
question for the user, to be recorded with `/dev-workflow:decision`. Do not commit.
