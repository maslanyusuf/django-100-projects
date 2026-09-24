# 001 Notes App — Step 2: Env-driven settings

## Step 1 check, answered

`settings.py`, specifically `INSTALLED_APPS`. Without the entry, `makemigrations` never imports `notes.models`, reports "No changes detected", and admin, templates and app discovery skip the app too.

## Concept

`settings.py` is plain Python executed once at startup. `manage.py`, `wsgi.py` and `asgi.py` find it through the `DJANGO_SETTINGS_MODULE` env var, which they default to `config.settings`.

The default file hardcodes `SECRET_KEY`, `DEBUG = True`, an empty `ALLOWED_HOSTS` and a SQLite path. All of that is per-environment configuration. Committing it means either secrets in git or divergent copies of the file per environment.

The fix is the twelve-factor pattern: settings read the environment, and a `.env` file (never committed) supplies values locally. `django-environ` provides that:

- `environ.Env` gives typed readers: `env("X")`, `env.list(...)`, `env.bool(...)`.
- `env.db(...)` parses a `DATABASE_URL` into Django's `DATABASES` dict.
- A variable with no default and no value raises `ImproperlyConfigured`. That's intentional: a missing `SECRET_KEY` should stop startup, not fall back silently.

`BASE_DIR` in modern Django is a `pathlib.Path`, so `BASE_DIR / "templates"` works directly.

## Build this

1. Create `.env.example` (committed) with placeholder values for `SECRET_KEY`, `DEBUG`, `ALLOWED_HOSTS` and `DATABASE_URL`. Copy it to `.env` and fill in a real secret. Generate one with `django.core.management.utils.get_random_secret_key()`.
2. Create `.gitignore` covering at least `.env`, `db.sqlite3`, `.venv/` and `__pycache__/`.
3. In `config/settings.py`, instantiate `environ.Env` and load `BASE_DIR / ".env"`.
4. Replace the hardcoded values:
   - `SECRET_KEY`: required, no default.
   - `DEBUG`: bool, default `False`.
   - `ALLOWED_HOSTS`: list, default `[]`.
   - `DATABASES`: from `DATABASE_URL`, default `sqlite:///db.sqlite3`.
5. Set `TEMPLATES[0]["DIRS"] = [BASE_DIR / "templates"]` and `STATICFILES_DIRS = [BASE_DIR / "static"]`. Create both empty directories (add a `.gitkeep` so git tracks them).
6. Verify:
   - `python manage.py check` passes.
   - Rename `.env` temporarily; `runserver` should fail with a clear `ImproperlyConfigured` message. Restore it.
7. Commit: `chore(001): env-driven settings`. Confirm `git status` never shows `.env`.

Watch for: `environ.Env.read_env` takes a path, and `DEBUG=(bool, False)` in the `Env(...)` constructor is how you declare the cast and default in one place.

## Paste me

- Your updated settings sections (env setup, `SECRET_KEY`/`DEBUG`/`ALLOWED_HOSTS`, `DATABASES`, `TEMPLATES` dirs, `STATICFILES_DIRS`).
- Your `.env.example` and `.gitignore`.

## Check question

`DEBUG` defaults to `False` and `SECRET_KEY` has no default. With no `.env` present, what happens on startup for each, and why is that the safer failure mode than defaulting `DEBUG` to `True`?
