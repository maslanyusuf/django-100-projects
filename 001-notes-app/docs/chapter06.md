# 001 Notes App — Step 6: URLconf

## Step 5 check, answered

`django.contrib.admin`'s `AppConfig.ready()` calls `autodiscover()`. That runs once at startup during `django.setup()` (triggered by `runserver`, `manage.py` commands, WSGI/ASGI loading). It imports the `admin` module of every installed app, which executes your `@admin.register(...)` decorators.

## Models-phase checkpoint (self-check)

You haven't pasted Steps 3–5, so I haven't reviewed them. Verify each item yourself before moving on. Anything you can't tick is a bug to fix now, not later.

**Testing Checklist items you can already verify in `manage.py shell`:**

- [ ] `str(note)` returns the title.
- [ ] `Note.objects.active()`, `.archived()` and `.search()` behave as specified (title match, body match, case-insensitive, empty `q` returns everything).
- [ ] Default ordering is pinned first, then newest `updated_at` (`Note.objects.all().query` shows the `ORDER BY`).
- [ ] `Note.objects.create(title="")` raises `IntegrityError` (the DB constraint fires, not Python validation).
- [ ] `get_absolute_url()` works after this step.

**Security Checklist items that apply so far:**

- [ ] `SECRET_KEY` and `DEBUG` come from env; `DEBUG` defaults to `False`.
- [ ] `.env` and `db.sqlite3` are in `.gitignore`, and `git status` never shows them.
- [ ] `.env.example` is committed and contains no real secret.
- [ ] `makemigrations --check` is clean.

Automated versions of these become tests in Step 13.

## Concept

A **URLconf** is a module exposing a `urlpatterns` list of `path()` entries. Django takes the request path, walks the root URLconf (`ROOT_URLCONF` in settings) top to bottom, and dispatches to the **first** matching pattern's view.

- `path("notes/<int:pk>/", view, name="detail")`: `<int:pk>` is a **path converter**. It matches digits only and passes `pk` to the view as an `int`.
- `include()` mounts another URLconf under a prefix, so each app owns its routes.
- `app_name = "notes"` in the app's `urls.py` creates a **namespace**, so a route named `detail` is addressed as `notes:detail`. Without it, two apps that both name a route `detail` collide.
- Resolve URLs by name: `reverse("notes:detail", args=[pk])` in Python and `{% url "notes:detail" note.pk %}` in templates. Never hardcode paths; renaming a route then only touches one file.
- Order matters when patterns overlap, since first match wins.

## Build this

1. Create `notes/urls.py` with `app_name = "notes"` and these routes. Names and paths come straight from the spec's API table:

| Path                       | Name      |
| -------------------------- | --------- |
| `/`                        | `list`    |
| `/notes/new/`              | `create`  |
| `/notes/<int:pk>/`         | `detail`  |
| `/notes/<int:pk>/edit/`    | `edit`    |
| `/notes/<int:pk>/delete/`  | `delete`  |
| `/notes/<int:pk>/pin/`     | `pin`     |
| `/notes/<int:pk>/archive/` | `archive` |

2. In `config/urls.py`, keep `admin/` and add `path("", include("notes.urls"))`.
3. The views don't exist until Step 7, so add throwaway one-line view functions in `notes/views.py` (`note_list`, `note_create`, `note_detail`, `note_edit`, `note_delete`, `note_pin`, `note_archive`) that just return an `HttpResponse`. Step 7 replaces them. This lets `check` pass and `reverse()` resolve now.
4. Run `python manage.py check`.
5. In the shell, verify:
   - `reverse("notes:detail", args=[1])` gives `/notes/1/`.
   - `resolve("/notes/1/")` (from `django.urls`) returns the right view name and kwargs.
   - `Note.objects.first().get_absolute_url()` works.
6. Hit `/`, `/notes/new/` and `/notes/1/` in the browser and confirm each stub responds.
7. Commit: `feat(001): notes urlconf with namespace`.

`/healthz/` is a project-level route with no namespace; it comes in Step 12, so leave it out for now.

## Paste me

- `notes/urls.py` and `config/urls.py`.
- The output of your `reverse`/`resolve` checks.

## Check question

Two apps later both define a route named `detail`. Without `app_name`, what happens to `{% url "detail" 1 %}`, and how does the namespace fix it?
