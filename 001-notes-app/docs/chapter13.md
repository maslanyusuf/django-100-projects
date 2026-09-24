# 001 Notes App — Step 13: Tests

## Step 12 check, answered

The toolbar exposes internals: every SQL statement, settings, request headers, template paths and stack traces. That's information disclosure, so it must not exist outside development. `INTERNAL_IPS` is a second gate: even with `DEBUG=True`, the panel renders only when the request's client IP is in the list. An accidental `DEBUG=True` on a public host then doesn't show the panel to remote visitors. It's defense in depth, not a fix: `DEBUG=True` in production still leaks tracebacks and settings.

## Concept

**pytest-django** runs Django tests under pytest.

- `DJANGO_SETTINGS_MODULE` is configured in `pyproject.toml` (`[tool.pytest.ini_options]`), since pytest doesn't go through `manage.py`.
- DB access is **blocked by default**. `@pytest.mark.django_db` opts a test in, creates a test database, and wraps the test in a transaction rolled back afterwards. This keeps non-DB tests fast and catches accidental queries.
- Fixtures you'll use: `client` (a `django.test.Client`) and `django_assert_num_queries(n)` (a context manager that fails if the block doesn't run exactly `n` queries).
- The test client **skips CSRF checks by default**. To test the 403 case you need an explicit `django.test.Client(enforce_csrf_checks=True)`.
- pytest-django sets `DEBUG=False` for the run, and Django's test setup allows the `testserver` host, so `ALLOWED_HOSTS` doesn't get in the way.

**factory_boy** builds model instances for tests. `NoteFactory(DjangoModelFactory)` declares defaults once (`class Meta: model = Note`, fields via `factory.Faker` or `factory.Sequence`). Then `NoteFactory()` creates a saved row, `NoteFactory.build()` an unsaved one, and `NoteFactory.create_batch(n)` many. Override any field per call: `NoteFactory(title="x", is_pinned=True)`.

`SECRET_KEY` has no default, so tests need your `.env` present locally. CI (Step 005) will inject env vars instead.

## Build this

1. Add `pytest`, `pytest-django`, `pytest-cov` and `factory-boy` to `requirements-dev.txt` and install.
2. In `pyproject.toml`, add `[tool.pytest.ini_options]` with `DJANGO_SETTINGS_MODULE = "config.settings"`, test file patterns, and `addopts` for `--cov=notes --cov-report=term-missing`.
3. `notes/tests/factories.py`: `NoteFactory`.
4. Write the tests below. Each row maps to a line in the spec's Testing Checklist.

| File             | Cases                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           |
| ---------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `test_models.py` | `__str__`; `get_absolute_url`; `active` / `archived`; `search` (title match, body match, case-insensitive, empty `q` returns all); default ordering (pinned first, then newest `updated_at`); `title=""` raises `IntegrityError` at the DB level.                                                                                                                                                                                                                                                                               |
| `test_forms.py`  | Title required; body optional; title over 200 chars invalid.                                                                                                                                                                                                                                                                                                                                                                                                                                                                    |
| `test_views.py`  | List: 200, pagination boundaries (20 vs 21 notes), `?archived=1`, `?q=`. List with `HTTP_HX_REQUEST="true"` returns a partial (no `<html>`), without it the full page, and `Vary` includes `HX-Request`. Create/edit valid gives 302 plus a message; invalid gives 200 with form errors. Pin toggles and bumps `updated_at`; archive and delete return 200 with an empty body on htmx. GET on mutating URLs returns 405. POST without a CSRF token returns 403. `/healthz/` returns `{"status": "ok"}`. Unknown pk returns 404. |

5. **Query-count tests.** Assert the list view runs exactly 2 queries with `django_assert_num_queries`, with a few notes and again with more than one page of notes (page size must not change the count). Then a perf test: seed 10k notes with `bulk_create` and assert the same 2 queries.
6. Run `pytest`. Read the `term-missing` output and add tests until `notes/` coverage is **≥90%**.
7. Commit in small pieces, for example `test(001): note model and queryset` and `test(001): views and htmx branches`.

Gotchas to plan for:

- **Ordering test.** `updated_at` is `auto_now`, so a value you pass at creation is overwritten on `save()`. You need a way to write it that bypasses `save()`. Recall Step 3.
- **`IntegrityError` test.** After a database error inside a `django_db` test, the transaction is unusable. If you need more queries after the failing statement, wrap that statement in `transaction.atomic()`.
- **Messages.** `django.contrib.messages` has a helper to read queued messages from a response's request; find it in the docs rather than parsing HTML.
- **Form errors and context.** The test client exposes the template context on `response.context`, so assert on `form.errors` and `page_obj` there instead of searching the HTML.
- **`Vary`.** Assert on the header, not the body.
- **Toolbar namespace.** If you hit `'djdt' is not a registered namespace`, this is the pytest-django `DEBUG=False` issue from Step 12. Gate the toolbar so it isn't configured under test.

## Paste me

- `pyproject.toml`, `factories.py`, and your ordering test and CSRF test.
- The final coverage report.
- Any test that fought back.

The tests phase ends with this step. The **final checkpoint** against the full Testing and Security checklists and the Definition of Done opens the Step 14 file.

## Check question

In the ordering test, why can't you set `updated_at` by passing it to the factory, and what do you use instead?
