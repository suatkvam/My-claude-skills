# claude-dev-workflow

A Claude Code plugin marketplace with one plugin, `dev-workflow`, for projects that are driven
by documents: a living PRD and an append-only decision log.

| Component | Type | What it does |
| :- | :- | :- |
| `/dev-workflow:decision` | skill | Records a decision as the next `D-xx` entry, updates every PRD section it affects, has a separate reviewer subagent check the change, and reports. |
| `/dev-workflow:project-map` | skill | Reads the code and writes a map of the project: architecture, component catalogue, end-to-end flows, and what is real versus mock. |
| `dev-workflow:reviewer` | subagent | Reviews changed PRD sections against the decision log. `decision` runs it; you can also ask for it by name. |

Both skills run only when you type them. Claude does not start them on its own.

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

## `/dev-workflow:decision`

```text
/dev-workflow:decision <the decision or change> [--prd <path>] [--decisions <path>] [--lang <language>]
```

| Option | Default | Meaning |
| :- | :- | :- |
| `--prd <path>` | `docs/PRD.md` | The PRD file |
| `--decisions <path>` | `docs/DECISIONS.md` | The decision log |
| `--lang <language>` | the language you write in | Language of the final report |

Examples:

```text
/dev-workflow:decision Sessions expire after 30 days of inactivity instead of never.

/dev-workflow:decision Drop CSV export from the MVP; keep JSON only. --lang Turkish

/dev-workflow:decision Use Postgres row-level security for tenant isolation --prd specs/product.md --decisions specs/decisions.md
```

What happens:

1. Claude reads the decision log and the affected PRD sections.
2. It adds the next `D-xx` entry, or a dated clarification on an existing entry.
3. It updates every PRD section the decision affects and cites the identifier there.
4. The `dev-workflow:reviewer` subagent reviews the change. Claude fixes the valid findings.
5. Claude updates the PRD header (date and latest decision) and checks that the numbering has
   no gap.
6. You get a report: what changed, the reviewer's findings, questions that need your decision,
   and task-level items listed separately.

The skill edits documents only. It does not write application code and does not commit. It
stops if you are in a linked git worktree or not on the default branch, unless you tell it to
work there.

## `/dev-workflow:project-map`

```text
/dev-workflow:project-map [output path ending in .md] [output language]
```

Defaults: `docs/PROJECT-MAP.md`, in the language you write in.

Examples:

```text
/dev-workflow:project-map

/dev-workflow:project-map docs/architecture/MAP.md English
```

The map trusts code over documents and backs each claim with `file:line`. It labels every
component REAL, PARTIAL, SKELETON, MOCK or DEAD, and lists every mock, stub and TODO in one
tracking table. If the output file already exists, the skill updates it from what changed
since the last run and adds a "Changes since last map" section. It modifies no other file.

## Expected document structure

```text
docs/
  PRD.md            living product requirements; states what is true now
  DECISIONS.md      append-only decision log; states why
  PROJECT-MAP.md    written by project-map
```

If your project keeps these files elsewhere, pass the paths as options. Without options,
`decision` checks `CLAUDE.md`, `AGENTS.md` and the README for the location, then searches for
files such as `PRD*.md` and `DECISIONS*.md`.

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
- If the log already exists, `decision` follows its format instead of the one above.
- If there is no log, `decision` creates `docs/DECISIONS.md` and starts at `D-01`.
- If there is no PRD, `decision` records the decision only and tells you. It does not invent
  a PRD.

The PRD carries a header line that `decision` keeps current:

```markdown
Last updated: 2025-01-31 — latest decision: D-07
```

## Türkçe

`dev-workflow`, yaşayan bir PRD ve karar günlüğü ile yürütülen projeler için bir Claude Code
eklentisidir.

Kurulum:

```text
/plugin marketplace add suatkvam/My-claude-skills
/plugin install dev-workflow@claude-dev-workflow
```

- `/dev-workflow:decision <karar> [--prd <yol>] [--decisions <yol>] [--lang <dil>]`: kararı
  `docs/DECISIONS.md` dosyasına bir sonraki `D-xx` numarasıyla yazar, PRD'nin etkilenen
  bölümlerini günceller, değişikliği ayrı bir `reviewer` alt ajanına inceletir ve rapor verir.
  Karar günlüğü yoksa `D-01` ile başlatır.
- `/dev-workflow:project-map [çıktı yolu] [dil]`: kodu okur ve projenin haritasını
  `docs/PROJECT-MAP.md` dosyasına yazar: mimari, bileşen listesi, uçtan uca akışlar, gerçek
  ve sahte (mock) parçalar.

Rapor ve çıktı dili varsayılan olarak yazdığınız dildir. Örnek:

```text
/dev-workflow:decision Oturumlar 30 gün hareketsizlikten sonra kapanır. --lang Turkish
/dev-workflow:project-map docs/HARITA.md Turkish
```

Skill'ler yalnızca siz yazdığınızda çalışır; Claude kendiliğinden başlatmaz.

## License

MIT. See [LICENSE](LICENSE).
