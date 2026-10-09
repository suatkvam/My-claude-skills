# claude-dev-workflow

A Claude Code plugin marketplace with one plugin, `dev-workflow`, for projects that are driven
by documents: a living PRD and an append-only decision log.

| Component | Type | What it does |
| :- | :- | :- |
| `/dev-workflow:prd` | skill | Writes the first PRD and decision log from your source documents. Only what a source states as decided becomes a decision; everything else goes to open questions. Optional web research by subagents. The reviewer checks the result. |
| `/dev-workflow:decision` | skill | Records a decision as the next `D-xx` entry, updates every PRD section it affects, has a separate reviewer subagent check the change, and reports. |
| `/dev-workflow:project-map` | skill | Reads the code and writes a map of the project: architecture, component catalogue, end-to-end flows, and what is real versus mock. |
| `dev-workflow:reviewer` | subagent | Reviews the PRD against the decision log: the sections one decision changed, or a whole new PRD and log against their sources. `decision` and `prd` run it; you can also ask for it by name. |
| `dev-workflow:researcher` | subagent | Researches one focus area on the web and returns sourced findings. `prd --research` runs it. |

The subagents run on the model of your session. For the best results, use a strong model
such as Opus. If the environment variable `CLAUDE_CODE_SUBAGENT_MODEL` is set, they run on
that model instead.

The skills run only when you type them. Claude does not start them on its own.

## Install

In a Claude Code session:

```text
/plugin marketplace add suatkvam/My-claude-skills
/plugin install dev-workflow@claude-dev-workflow
```

Or from your shell:

```bash
claude plugin marketplace add suatkvam/My-claude-skills
claude plugin install dev-workflow@claude-dev-workflow
```

To try it without installing, clone the repository and run
`claude --plugin-dir ./plugins/dev-workflow`.

## `/dev-workflow:prd`

```
/dev-workflow:prd <source files or directories> [--research] [--prd <path>] [--decisions <path>] [--lang <language>] [--here]
```

Use it once, to start the documents of a project. Sources are files you already have: design
notes, an exported chat, an `idea.md`. After this, record every change with
`/dev-workflow:decision`.

| Option | Default | Meaning |
| :-- | :-- | :-- |
| `--research` | off | Run three `researcher` subagents (users and market, prior art and competitors, technical feasibility) on the web |
| `--prd <path>` | `docs/PRD.md` | Where to write the PRD |
| `--decisions <path>` | `docs/DECISIONS.md` | Where to write the decision log |
| `--lang <language>` | the language you write in | Language of the final report |
| `--here` | off | Work in the current branch or worktree instead of stopping |

Examples:

```
/dev-workflow:prd docs/source/design.md

/dev-workflow:prd docs/idea.md --research --lang Turkish
```

What happens:

1. The skill finds the targets the way `decision` finds documents (a path named in
   `CLAUDE.md`, `AGENTS.md` or the README, the default paths, then a search). It stops if the
   PRD already has content or the decision log has entries: it never overwrites or merges. Use
   `decision` for changes. Sources must be inside the project; PDFs are read, other binary
   files are not.
2. Claude reads every source and sorts each statement: decided, superseded, proposal, open, or
   context. **Only what a source states as decided becomes a decision.** In a chat export,
   only your own acceptance decides; what an assistant wrote is a proposal. For each source
   that does not say what it is, Claude asks once: approved specification or draft? When
   unsure, a statement is a proposal.
3. With `--research`, researcher subagents search the web, one per focus that has open
   questions. This sends a general description of your project and its open questions to web
   search, without names, identifiers, secrets or quotes from your documents. The researchers
   have only web tools and cannot read your files. Their findings go into a "Research" section
   with sources, marked unverified; their recommendations go to "Open questions".
4. Claude writes the decision log (`D-01`, ... in the order decisions were made; superseded
   decisions get their own entry) and the PRD, citing a decision or a source for every claim.
   Sections the sources do not cover are left out, not padded. If the sources decide nothing,
   the log has only its header and the PRD says `latest decision: none`.
5. The `reviewer` subagent checks both documents against the sources: no decision without a
   source, every decision reflected, every open item listed. Claude fixes the valid findings.
6. You get a report: what was written, what was skipped, the review, and the open questions.

The documents follow the language convention in `CLAUDE.md`, `AGENTS.md`, the README or the
sources, otherwise the language of the sources. Like `decision`, the skill stops in a linked
worktree or off the default branch unless you pass `--here`. It writes no code, does not
change the sources and does not commit.

## `/dev-workflow:decision`

```text
/dev-workflow:decision <the decision or change> [--prd <path>] [--decisions <path>] [--lang <language>] [--here]
```

| Option | Default | Meaning |
| :- | :- | :- |
| `--prd <path>` | `docs/PRD.md` | The PRD file |
| `--decisions <path>` | `docs/DECISIONS.md` | The decision log |
| `--lang <language>` | the language you write in | Language of the final report |
| `--here` | off | Work in the current branch or worktree instead of stopping |

Put the options at the end, after the decision text. A value with a space goes in quotes:
`--prd "docs/My PRD.md"`. An option name inside the decision text stays part of the text.

Examples:

```text
/dev-workflow:decision Sessions expire after 30 days of inactivity instead of never.

/dev-workflow:decision Drop CSV export from the MVP; keep JSON only. --lang Turkish

/dev-workflow:decision Use Postgres row-level security for tenant isolation --prd specs/product.md --decisions specs/decisions.md
```

What happens:

1. Before editing, the skill copies the PRD and the decision log to a temporary directory
   (`dev-workflow-decision.*` in your system's temporary directory) and tells you the path.
   The reviewer gets the `diff -u` between that copy and the result, so the review works
   without git and on the first run. The directory is deleted at the end; a later run removes
   what an interrupted run left.
2. Claude reads the decision log and the affected PRD sections.
3. It adds the next `D-xx` entry, or a dated clarification on an existing entry.
4. It updates every PRD section the decision affects and cites the identifier there.
5. The `dev-workflow:reviewer` subagent reviews the change. Claude fixes the valid findings.
   If the reviewer cannot run, the report says that the change was not reviewed.
6. Claude updates the PRD header (date and latest decision) and checks that the new number
   follows the highest one and is not used twice.
7. You get a report: what changed, the reviewer's findings, questions that need your decision,
   and task-level items listed separately.

The skill edits documents only. It does not write application code and does not commit. It
stops if you are in a linked git worktree or not on the default branch; pass `--here` to work
there. The check is skipped when the project is not a git repository, has only one local
branch, or has no known default branch (no `origin/HEAD`).

## `/dev-workflow:project-map`

```text
/dev-workflow:project-map [output path ending in .md] [output language] [--run-checks]
```

Defaults: `docs/PROJECT-MAP.md`, in the language you write in. A path with a space goes in
quotes. The skill asks about any argument it does not recognise.

By default the skill runs nothing from your project: no tests, no check or build command. It
names the test command and marks the test status "not run". With `--run-checks` it runs the
project's own test/check command, under these limits: it installs no dependencies, runs
nothing that needs network access, external services or secrets, uses no watch mode, stops a
command after 10 minutes, and compares `git status` before and after (a difference is
reported, not cleaned up).

Examples:

```text
/dev-workflow:project-map

/dev-workflow:project-map docs/architecture/MAP.md English

/dev-workflow:project-map --run-checks
```

The map trusts code over documents and backs each claim with `file:line`. It labels every
component REAL, PARTIAL, SKELETON, MOCK or DEAD, with the test status in a separate column,
and lists every mock, stub and TODO in one tracking table. Dependencies, vendored code and
generated code are skipped. If the output file is a map from an earlier run, the skill
updates it from what changed since then and adds a "Changes since last map" section. If the
file exists but is not a map (it has no "Generated ... from" line), the skill asks before it
overwrites it. It modifies no other file and does not regenerate a code graph.

## Expected document structure

```text
docs/
  PRD.md            living product requirements; states what is true now
  DECISIONS.md      append-only decision log; states why
  PROJECT-MAP.md    written by project-map
```

If your project keeps these files elsewhere, pass the paths as options. Without options,
`decision` uses, in this order: a file that `CLAUDE.md`, `AGENTS.md` or the README names;
the default path; a search for files such as `PRD*.md` and `DECISIONS*.md` and for an `adr/`
or `decisions/` directory. File names match without regard to case (`docs/decisions.md` is
found), and dependency and build directories are skipped. Paths are relative to the directory
you run Claude Code in. If several candidates fit, it asks.

A decision log entry:

```markdown
## D-07 — Short title of the decision

- **Date:** 2025-01-31
- **Status:** Accepted
- **Decision:** What was decided, in one to three sentences that can be read alone.
- **Why:** The reason, and the alternatives that were rejected.
- **Affects:** The PRD sections this changes.
```

- The next number is the highest number plus one. Numbers are never reused, renumbered or
  skipped.
- Entries are never deleted. A reversed decision gets a new entry, and the old one gets the
  status `Superseded by D-xx`.
- If the log already exists, `decision` follows its format instead of the one above:
  identifier prefix, number width, order, table or headings. The `D-xx` format is for a new
  log only. If the entries have no sequential number, it asks.
- If there is no log, `decision` asks before it creates `docs/DECISIONS.md` (or the file in
  an existing `doc/` or `documentation/` directory) and starts at `D-01`, written in the
  language of the PRD.
- If there is no PRD, `decision` records the decision only and tells you. It does not invent
  a PRD.

The PRD carries a header line that `decision` keeps current:

```markdown
Last updated: 2025-01-31 — latest decision: D-07
```

If the PRD already has a date field, such as `updated:` in its frontmatter, `decision` uses
that field and adds no second line.

## Calling the reviewer by hand

Ask for the `dev-workflow:reviewer` subagent by name. To review a whole PRD and log against
their sources, say "new documents" and give the source paths. Without input it finds the PRD and the
decision log in the same order as `decision`, takes the newest entry of the log and the PRD
sections in its "Affects" field, and says at the top of its output what it assumed. Without a
diff it cannot check for lost behavior and says so. If it finds no PRD, it does no review.

## Türkçe

`dev-workflow`, yaşayan bir PRD ve karar günlüğü ile yürütülen projeler için bir Claude Code
eklentisidir.

Kurulum:

```text
/plugin marketplace add suatkvam/My-claude-skills
/plugin install dev-workflow@claude-dev-workflow
```

- `/dev-workflow:prd <kaynaklar> [--research] [--prd <yol>] [--decisions <yol>] [--lang <dil>] [--here]`:
  projenin ilk PRD'sini ve karar günlüğünü kaynak belgelerden (tasarım notları, sohbet
  çıktısı, `idea.md`) yazar. Yalnızca bir kaynağın karar olarak belirttiği şey karar olur;
  sohbet çıktısında yalnızca sizin onayınız karardır. Geri kalanı açık sorulara gider.
  `--research` ile araştırmacı alt ajanlar projenin genel tarifini ve açık soruları web'de
  arar; bulgular kaynaklı ve doğrulanmamış olarak işaretlenir. PRD zaten doluysa durur;
  sonraki değişiklikler `decision` ile yapılır.
- `/dev-workflow:decision <karar> [--prd <yol>] [--decisions <yol>] [--lang <dil>] [--here]`:
  kararı karar günlüğüne (varsayılan `docs/DECISIONS.md`) bir sonraki numarayla yazar, PRD'nin
  etkilenen bölümlerini günceller, değişikliği ayrı bir `reviewer` alt ajanına inceletir ve
  rapor verir. Var olan günlüğün biçimini izler. Günlük yoksa sorar, sonra `D-01` ile başlatır.
  Varsayılan dalda değilseniz ya da bağlı bir worktree'deyseniz durur; `--here` ile orada
  çalışır. Seçenekler karar metninin sonunda yazılır.
- `/dev-workflow:project-map [çıktı yolu] [dil] [--run-checks]`: kodu okur ve projenin
  haritasını `docs/PROJECT-MAP.md` dosyasına yazar: mimari, bileşen listesi, uçtan uca
  akışlar, gerçek ve sahte (mock) parçalar. Varsayılan olarak projenin testlerini ve
  komutlarını çalıştırmaz; `--run-checks` ile sınırlı biçimde çalıştırır.

Rapor ve çıktı dili varsayılan olarak yazdığınız dildir. Örnek:

```text
/dev-workflow:decision Oturumlar 30 gün hareketsizlikten sonra kapanır. --lang Turkish
/dev-workflow:project-map docs/HARITA.md Turkish
```

Skill'ler yalnızca siz yazdığınızda çalışır; Claude kendiliğinden başlatmaz.

## License

MIT. See [LICENSE](LICENSE).
