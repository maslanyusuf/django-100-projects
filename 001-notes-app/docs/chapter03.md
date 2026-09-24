# 001 Notes App — Step 3: Model + migration

## Step 2 check, answered

- **`SECRET_KEY` missing:** `env("SECRET_KEY")` has no default, so startup raises `ImproperlyConfigured` immediately. Loud failure, nothing runs.
- **`DEBUG` missing:** it falls back to `False`, so the app runs in production mode. No debug pages leaking tracebacks and settings, and `ALLOWED_HOSTS` is enforced (empty list means every request gets a 400).
- Defaulting `DEBUG` to `True` would fail open: a forgotten env var in a deploy exposes tracebacks and settings publicly. Safe defaults fail closed.
- Note that `read_env` doesn't fail on a missing `.env` file. The failure comes from the missing variable.

## Concept

A **migration** is a Python file in `notes/migrations/` that records one schema change as a list of operations (`CreateModel`, `AddField`, ...).

- `makemigrations` diffs your current models against the state rebuilt by replaying all existing migrations, then writes the delta as a new file.
- `migrate` applies unapplied files to the DB and records each in the `django_migrations` table.
- Migrations are versioned history. You commit them and never edit an applied one; a model change means a new migration.
- `sqlmigrate` prints the SQL a migration will run without executing it. Read it until it stops being surprising.

Two model details in this project:

- `Meta` holds table-level options: `ordering`, `indexes`, `constraints`. Constraints named in `Meta` are enforced by the **database**, not just by Python validation. That's why the spec tests for `IntegrityError` rather than `ValidationError`.
- `auto_now` fires inside `Model.save()` only. `QuerySet.update()` bypasses it. It matters again in Step 11.

## Build this

Write `notes.Note` exactly per the Domain Model in the spec:

1. Fields: `title`, `body`, `is_pinned`, `is_archived`, `created_at`, `updated_at`, with the types, defaults and flags in the table. Confirm the PK is a `BigAutoField`. Check what `DEFAULT_AUTO_FIELD` and `NotesConfig.default_auto_field` are set to; don't assume.
2. `Meta.ordering = ["-is_pinned", "-updated_at"]`.
3. `Meta.indexes`: one composite `Index` on `["is_archived", "-is_pinned", "-updated_at"]`, named `note_list_idx`.
4. `Meta.constraints`: one `CheckConstraint` rejecting `title == ""`. Constraints require a `name`. On Django 5.1+ the keyword is `condition=`; `check=` is deprecated. Check your installed version with `python -m django --version`. You'll need a `Q` object, and negation (`~Q(...)`) is the idiomatic way to say "not equal".
5. Run `makemigrations`. Open the generated `0001_initial.py` and read it.
6. Run `python manage.py sqlmigrate notes 0001`. Find the table, the index and the `CHECK` clause in the SQL.
7. Run `migrate`. Then open `python manage.py dbshell` (or the shell) and confirm a row exists in `django_migrations` for `notes.0001_initial`.
8. Run `makemigrations --check`. It should exit clean.
9. Commit the model and the migration file together: `feat(001): note model and initial migration`.

Don't add `__str__`, `get_absolute_url` or the custom QuerySet yet. Those are Steps 4 and 5.

## Paste me

- `notes/models.py`.
- The `sqlmigrate` output.

## Check question

Later you change `title` to `max_length=300`. Do you edit `0001_initial.py`? If not, what do you run, and what does Django compare to detect the change?
