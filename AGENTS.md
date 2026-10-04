# spruceOS

## Local-only paths (never commit)

- `.scratch/` = issue tracker + local tests. It is gitignored.
  Before every commit, `git status` must not list it.
- Test files live ONLY in `.scratch/tests/` (local-only, gitignored). Never add `test_*` under
  `App/` or anywhere tracked. Tests must resolve repo code via paths relative to the
  repo root (see `.scratch/tests/test_progress_bar_*.py`), never by sitting next to the code.

## Agent skills

### Issue tracker

Issues live as local markdown files under `.scratch/`. See `docs/agents/issue-tracker.md`.

### Triage labels

Default five-role vocabulary (needs-triage, needs-info, ready-for-agent, ready-for-human, wontfix). See `docs/agents/triage-labels.md`.

### Domain docs

Single-context (one `GLOSSARY.md` + `docs/adr/` at repo root). See `docs/agents/domain.md`.
