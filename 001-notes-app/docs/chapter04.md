# 001 Notes App — Step 4: Custom QuerySet

## Step 3 check, answered

No, never edit an applied migration. Run `makemigrations`; it writes `0002_...` with an `AlterField`.

Django detects the change by comparing your current models to the project state rebuilt by **replaying the migration files**, not by inspecting the live DB. Editing `0001` would do nothing on any database that already recorded it in `django_migrations`, so environments would silently diverge.

## Concept

A **QuerySet** is a lazy, chainable description of a query. Nothing hits the DB until it's evaluated by iterating, slicing with a step, `len()`, `bool()`, `list()`, or a terminal method like `.count()`, `.get()` or `.exists()`. Every chained call (`.filter()`, `.exclude()`) returns a **new** QuerySet.

Because of that, business filters belong on a custom QuerySet subclass, not in views:

- Views stay thin and read like intent: `Note.objects.active().search(q)`.
- The filters are unit-testable without an HTTP request.
- They chain in any order, and every later project can reuse the pattern.

`QuerySet.as_manager()` builds a `Manager` exposing your QuerySet's methods, so `Note.objects.active()` works as well as `Note.objects.filter(...).active()`. Assigning it to `objects` replaces the default manager.

`Q` objects wrap a condition so it can be combined with `|` (OR), `&` (AND) and `~` (NOT). Plain keyword arguments to `.filter()` can only AND.

## Build this

1. In `notes/models.py`, define `NoteQuerySet(models.QuerySet)` above `Note` with three methods:
   - `active()`: not archived.
   - `archived()`: archived.
   - `search(q: str)`: case-insensitive match on title **or** body. If `q` is empty, it must return everything unfiltered (the list view will pass `""` when the search box is blank).
2. Attach it to `Note` with `objects = NoteQuerySet.as_manager()`.
3. Open `python manage.py shell`. Create three or four notes with `Note.objects.create(...)`, including one archived and one matching only in the body.
4. Print the SQL for `Note.objects.active().search("x").query` and check the `WHERE` clause matches your intent (`OR` grouped correctly, archived flag present).
5. Prove laziness: wrap the code below in `CaptureQueriesContext` (from `django.test.utils`, used with `django.db.connection`) and confirm that building a QuerySet issues zero queries while iterating it issues one.
6. Run `makemigrations --check`. It should report no changes, because managers don't alter the schema.
7. Commit: `feat(001): note queryset with active/archived/search`.

Gotcha: on SQLite, `icontains` is only case-insensitive for ASCII (a documented backend limit). Fine here; it becomes moot on Postgres.

## Paste me

- Your `NoteQuerySet` and the `objects = ...` line.
- The printed SQL from step 4.

## Check question

Given `qs = Note.objects.active().search("x")`, which of these hit the DB: `page = qs[:5]`, `for n in page: ...`, `qs.count()`? Say which and why.
