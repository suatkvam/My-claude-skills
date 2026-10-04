---
name: project-map
description: Map the project's architecture, workflows and real-vs-mock status by reading the code, and write it to a file
argument-hint: "[output path ending in .md] [output language]"
disable-model-invocation: true
---

Produce a complete map of this project and write it to a file. The reader is the project owner:
agents wrote most of the code, and the owner needs to understand and track what exists and how it works.

Arguments: "$ARGUMENTS". It may contain an output path (ending in .md) and/or an output language
(e.g. "English", "Turkish"). Defaults: path docs/PROJECT-MAP.md, language the one the user writes in
(English if that is not clear).
Write the document in the output language. Keep code, file and symbol names as they are. Explain
each technical term in one sentence the first time it appears.
Do not modify any other file. Do not commit.

## How to work
- Trust code over documents. READMEs, PRDs and plans state intent; the code shows what actually
  runs. When they disagree, describe the code and list the contradiction separately.
- Back every claim with file:line. Mark anything you could not confirm as "unverified"; never guess.
- On a large project, first scan the directory tree, package manifests (pyproject, package.json,
  Cargo.toml, go.mod), docker-compose, Makefile, CI config, entry points (main, CLI, server, worker)
  and tests. Use subagents to read in parallel if useful, but merge and check their results yourself.
- If the project defines its own test/check command, run it and use the result. Do not run anything
  that needs network access, external services or secrets; note it instead.

### Use a code graph if one is available
- Check first, before you scan the tree: does this session have an MCP tool or a skill that serves
  a code graph (graphify, GitNexus or anything similar), or does the repository contain graph
  output (e.g. graphify-out/, a GitNexus index directory)? These names are examples. Recognise any
  tool or directory that holds modules, symbols and the edges between them.
- Check freshness: compare the time the graph was generated with the time of the last commit
  (`git log -1 --format=%cI`). If the graph is older and a local CLI can regenerate it without
  network access, regenerate it. If it cannot be regenerated, use the graph only as a hint and
  confirm in the code everything you take from it.
- Use the graph for: the inventory of modules and components, the dependencies between modules,
  the call chains in the flows, and code that nothing calls (DEAD). This replaces walking the
  files one by one.
- A graph shows structure, not bodies. Give the REAL/PARTIAL/SKELETON/MOCK labels and the file:line
  evidence from reading the function bodies. Read only the files the graph points to.
- If there is no graph, continue with the directory scan described above.
- Name the source in the "Generated" line at the top of the map, e.g. "graph: graphify <time>".
  If the graph was stale and used only as a hint, say so there.

## Accuracy rules
- Counts in the Summary (REAL/PARTIAL/...) must be computed from the rows of the component catalogue,
  not estimated. Recount on every update.
- Ownership, assignment, status and dependencies (who owns a task, which agent, what is done) must be
  read from the source file that records them (task list, issue tracker, CODEOWNERS); never infer them.
- For agent commands/skills/subagents, state where each one lives: project `.claude/`, user
  `~/.claude/`, or an installed plugin. Check all three before listing them.
- If a check or test fails, rerun the failing test alone (at least 3 times). Report it as FLAKY if
  it passes in isolation, as FAILING if it fails consistently, with the evidence.
- Commits not traceable to a task/issue/PR are listed under "Process", with a note if the project
  records them elsewhere (e.g. a status file); do not call them violations unless a rule says so.

## Update mode
- If the output file already exists, read it first and update it instead of starting over.
- Find what changed since it was generated, using the first source that works:
  1. git: the "Generated ... from <commit hash>" line at the top (or `git log -1 --format=%H -- <file>`),
     then `git diff --stat <hash>..HEAD` and `git log --oneline <hash>..HEAD`.
  2. A code graph (e.g. graphify-out/, a GitNexus index, or a graph skill/tool): use it only if
     the graph was regenerated after the map's "Generated" time; compare modules, dependencies and
     call edges with what the map describes. If the graph is older than the code, ignore it.
  3. File modification times: files newer than the map's "Generated" time (skip build output,
     caches, virtualenvs, node_modules).
  If none of these is reliable, regenerate fully.
- Re-read only the changed code and the components that depend on it; update only the affected
  sections and re-check REAL/MOCK/SKELETON labels for changed components.
- If most packages changed or the file is malformed, regenerate fully.
- Put a "Changes since last map" section right after the Summary: new/removed components, status
  changes (e.g. SKELETON -> REAL), new or removed mocks, flows that now work end to end, and which
  change-detection source was used.
- First line under the title: "Generated <date time> from <commit hash | graph <timestamp> |
  file times>".

## Sections
1. **Summary** — What the project does, for whom, and what stage it is at (5-8 sentences). A status
   table: how many components are REAL / PARTIAL / SKELETON / MOCK.
2. **Architecture** — Layers/packages and the dependency rules between them (who may import whom,
   and where the rule is enforced). One Mermaid diagram. Show external systems (database, model
   server, queue, third-party APIs) as separate nodes.
3. **Component catalogue** — One table row per module/service/component:
   | Component | Path | What it does | Why it exists (requirement/decision, with ID if any) | Used by | Status |
   Status labels: REAL (working implementation with tests), PARTIAL (some of it works; say what is
   missing), SKELETON (signature/interface only, body missing, NotImplementedError or xfail tests),
   MOCK (fake/stub/fabricated data; say what it stands in for and where the real one will live),
   DEAD (not called from anywhere).
4. **End-to-end flows** — For each main use case (e.g. a request, a background job, an import):
   step by step from entry to exit, which function calls which, what shape the data has and where it
   goes. Label every step REAL/MOCK/SKELETON. One Mermaid sequence diagram per flow. If a flow does
   not work end to end today, say exactly where it breaks.
5. **Data model and storage** — Tables/schemas/files, the migration chain, which component writes
   and reads which data, ownership and isolation rules.
6. **Real or mock? (tracking table)** — Every fake, stub, mock, hard-coded value, fabricated data,
   disabled feature flag and TODO/FIXME in the project:
   | What | Where | What it imitates | Used in tests or at runtime | Real implementation: task/file |
   Mark mocks used at runtime (outside tests) in bold: they are the riskiest.
6b. **Contracts and interfaces** — Every smart contract (Solidity, Move, Rust/ink, etc.) and every
   interface/port/protocol that modules depend on. For each: why it was written (which requirement or
   decision), its functions/entry points, state it holds, events it emits, who calls it, access
   control, and whether it is deployed/implemented or only declared. For smart contracts also note
   the network, deployment address if present, and test coverage.
7. **Configuration and running** — Environment variables and settings with defaults, how to
   install, run and test, which external services are required.
8. **Development process** — Task/decision/plan files if any, automation scripts, CI, code rules;
   how a change is made and merged.
9. **Risks and loose ends** — Contradictions between docs and code, untested critical paths,
   security-sensitive spots, half-finished work.
10. **Reading guide** — The 8-10 files to read, in order, to understand the project, with one
    sentence each on why.

When done, tell me where the file was written and the 5 most important findings, in the output language.
