# 001 Notes App — Step 14: Tooling + commits

## Step 13 check, answered

`updated_at` is `auto_now`: its `pre_save` sets the field to "now" on every `save()`, including the one inside `NoteFactory()`. Any value you pass is overwritten. `QuerySet.update()` issues a direct SQL `UPDATE` and never calls `save()` or `pre_save`, so `auto_now` doesn't fire. Use `Note.objects.filter(pk=note.pk).update(updated_at=...)` to set a controlled timestamp.

## Final checkpoint (self-check)

This is the last Hints step, so this covers the whole spec. You haven't pasted code since Step 1, so none of it is reviewed. Tick every box by hand. Anything unticked is a bug or a gap.

**Testing Checklist**

Unit

- [ ] `Note.__str__`, `get_absolute_url`
- [ ] `NoteQuerySet.active/archived/search` (title match, body match, case-insensitive, empty `q` returns all)
- [ ] Default ordering: pinned first, then `-updated_at`
- [ ] `CheckConstraint` blocks `title=""` (`IntegrityError`)
- [ ] `NoteForm`: title required, body optional, max length 200

Integration

- [ ] List: 200, pagination boundaries, `?archived=1`, `?q=`
- [ ] htmx returns a partial, non-htmx the full page, `Vary` includes `HX-Request`
- [ ] Create/edit: valid gives 302 plus message; invalid gives 200 with errors
- [ ] Pin toggles and bumps `updated_at`; archive/delete return 200 empty on htmx
- [ ] Mutating endpoints reject GET (405)
- [ ] POST without CSRF token gives 403 (`Client(enforce_csrf_checks=True)`)
- [ ] `/healthz/` returns `{"status": "ok"}`
- [ ] 404 on unknown pk

E2E

- [ ] Manual smoke: create, search, pin, archive, delete

Load / perf

- [ ] 10k notes seeded with `bulk_create`; list view still 2 queries
- [ ] `EXPLAIN` on the list query with `DATABASE_URL` pointed at Postgres; confirm `note_list_idx` is used

**Security Checklist**

- [ ] `SECRET_KEY` and `DEBUG` from env; `DEBUG` defaults to `False`
- [ ] `ALLOWED_HOSTS` set explicitly outside dev
- [ ] `{% csrf_token %}` on every POST form; `X-CSRFToken` on htmx requests
- [ ] No `|safe` / `mark_safe` on user content
- [ ] Mutations are POST-only
- [ ] `ModelForm.Meta.fields` is an explicit list
- [ ] `.env` and `db.sqlite3` in `.gitignore`
- [ ] `check --deploy` reviewed (warnings about HSTS, SSL redirect and secure cookies are expected locally)
- [ ] No auth in this project: bind to localhost only, never expose publicly

**Definition of Done** (the rest is covered by this step)

- [ ] `make install && make migrate && make run` works from a clean clone with only `.env` copied from `.env.example`
- [ ] `makemigrations --check` clean; `0001_initial.py` committed
- [ ] `make test` green, ≥90% coverage on `notes/`
- [ ] `make lint` clean; pre-commit installed
- [ ] Conventional Commits history on a feature branch merged to `main`
- [ ] Root `README.md` tracker updated (001 → 🟢). If the root README doesn't exist yet, say so and we generate it as its own deliverable.

## Concept

Three tools, three jobs:

- **ruff** lints: unused imports, undefined names, common bug patterns. It reports and can autofix some.
- **black** formats: it rewrites code to one canonical style, so style is never a review topic.
- **isort** sorts imports. With `profile = "black"` it produces output black won't reformat back.

Config for all three lives in `pyproject.toml`. Keep line length identical across them or they'll fight.

**pre-commit** runs those checks on **staged files** at `git commit` time. `pre-commit install` writes a Git hook into `.git/hooks/`. Each hook is pinned to a `rev` in `.pre-commit-config.yaml`, so every clone runs identical tool versions. If a hook fails or rewrites a file, the commit is aborted, and you re-stage and commit again.

A **Makefile** gives the repo one vocabulary (`make test`) that every later project reuses, and CI can call the same targets. Recipes must be indented with a **tab**, and non-file targets go under `.PHONY`.

**Conventional Commits**: `type(scope): summary`, for example `feat(001): live search with htmx`. Types you'll use: `feat`, `fix`, `chore`, `test`, `docs`, `refactor`. The convention keeps history scannable and can drive changelogs later.

## Build this

1. Add `ruff`, `black`, `isort` and `pre-commit` to `requirements-dev.txt`. Configure all three in `pyproject.toml`: matching line length, isort `profile = "black"`, and a decision on whether ruff should skip `migrations/` (generated code; common practice is to exclude it).
2. Create `.pre-commit-config.yaml` with hooks for ruff, black and isort. Get correct `rev` pins by running `pre-commit autoupdate` rather than typing versions from memory. Run `pre-commit install`, then `pre-commit run --all-files`, and commit whatever it reformats as its own `chore(001)` commit.
3. Write the `Makefile` with these targets:

| Target         | Job                                                                                         |
| -------------- | ------------------------------------------------------------------------------------------- |
| `install`      | Install dev requirements. Also install the pre-commit hook (the DoD requires it installed). |
| `migrate`      | `python manage.py migrate`                                                                  |
| `run`          | `python manage.py runserver`                                                                |
| `test`         | `pytest`                                                                                    |
| `lint`         | **Check only, no rewriting**: ruff, black in check mode, isort in check mode.               |
| `fmt`          | **Rewrites files**: black and isort (and ruff autofix if you want).                         |
| `deploy-check` | `python manage.py check --deploy`                                                           |

Do not add Docker targets; they start at 005. 4. Make sure `.env.example` contains every variable the app reads, with values that let the app start as-is. Test it: clone the repo into a temp dir, copy `.env.example` to `.env`, run `make install && make migrate && make run`. 5. Run `make deploy-check` with `DEBUG=False` and read each warning. Decide which are irrelevant locally and which you'd fix before exposing this anywhere. 6. **Branching.** I should have specified this in Step 1: the workflow is one feature branch, `feat/001-notes-app`, merged to `main`. Check `git branch`. If you've been committing on `main`, create the branch now (`git switch -c feat/001-notes-app`), do this step there, and merge with `git merge --no-ff` so the merge is recorded. Start the branch at Step 1 on every later project. 7. Review `git log --oneline`. Every message should follow Conventional Commits. Don't rewrite pushed history to fix old ones; just get the rest right. 8. Commit the tooling: `chore(001): ruff, black, isort, pre-commit and makefile`.

## Paste me

- `Makefile`, the tool sections of `pyproject.toml`, and `.pre-commit-config.yaml`.
- `git log --oneline` and `git branch`.
- The output of `make lint` and `make deploy-check`.
- Any unticked box from the final checkpoint above.

## Check question

Why are `fmt` (rewrites files) and `lint` (only reports) separate targets, and which one belongs in CI?
