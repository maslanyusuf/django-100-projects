# 001 Notes App — Step 5: Admin

## Step 4 check, answered

- `page = qs[:5]` does **not** hit the DB. Slicing an unevaluated QuerySet returns a new lazy QuerySet with `LIMIT 5` baked in.
- `for n in page: ...` hits the DB: one `SELECT ... LIMIT 5`.
- `qs.count()` hits the DB: one `SELECT COUNT(*)`. `qs` was never evaluated, so there's no result cache to reuse, and `count()` is a separate query from the one behind `page`.

## Concept

Django's admin is a CRUD UI generated from model metadata. You describe it with a `ModelAdmin` class and register that with `admin.site`.

- When `django.contrib.admin` is in `INSTALLED_APPS`, its `AppConfig.ready()` runs **autodiscover**, which imports the `admin` module of every installed app. That import executes your `@admin.register(...)` decorators. That's why nothing imports `notes/admin.py` explicitly.
- Admin reads `Model.Meta` (ordering, verbose names), field types and `__str__`, so a good `__str__` improves every admin screen.
- If a model defines `get_absolute_url()`, admin adds a "View on site" button, and `redirect(note)` in views will use it (Step 7). It should build the URL with `reverse()`, never a hardcoded path.

## Build this

1. In `notes/models.py`, add to `Note`:
   - `__str__` returning the title.
   - `get_absolute_url()` using `reverse("notes:detail", args=[self.pk])` (import from `django.urls`).
2. In `notes/admin.py`, register `Note` with a `ModelAdmin` decorated by `@admin.register(Note)`:
   - `list_display`: at least `title`, `is_pinned`, `is_archived`, `updated_at`.
   - `list_filter`: the two booleans.
   - `search_fields`: title and body.
   - `date_hierarchy = "created_at"`.
3. Run `python manage.py createsuperuser`.
4. `runserver`, open `/admin/`, log in, and create 5+ notes covering pinned, archived and normal, with different dates.
5. Exercise each option: search box, both filters, the date drill-down, and the default ordering (pinned first, then most recently updated).
6. Commit: `feat(001): admin registration and model helpers`.

Caveat: the `notes:detail` URL doesn't exist until Step 6, so `reverse()` raises `NoReverseMatch` if called. Don't click "View on site" or call `get_absolute_url()` yet. Defining the method is safe; nothing calls it until then.

## Paste me

- Your updated `notes/models.py` (the new methods only) and `notes/admin.py`.
- One line on anything in the admin that behaved differently than you expected.

Your next reply includes a **models-phase checkpoint** against the Testing and Security checklists.

## Check question

You never `import notes.admin` anywhere, yet the `Note` model shows up in the admin. What triggers that import, and when?
