# 001 Notes App — Step 1: Scaffold

## Concept

Django separates the **project** from **apps**.

- The **project** is the deployable config container: settings, root URLconf, WSGI/ASGI entry points.
- An **app** is a Python package that owns models, views and templates for one domain.
- An app only exists to Django once it's in `INSTALLED_APPS`. Django uses that list to discover models, migrations, templates and admin registrations.
- Apps are meant to be reusable and single-purpose, so `notes` is an app and `config` is the project.

The trailing dot in `startproject config .` matters. Without it, Django nests an extra directory and you end up with `001-notes-app/config/manage.py`, which breaks the layout every later project copies.

## Build this

1. Inside `001-notes-app/`, create and activate a venv.
2. Install `django`, `django-environ` and `django-htmx`. Pin them in `requirements.txt`.
3. Run `django-admin startproject config .` so `manage.py` sits at the project root.
4. Run `python manage.py startapp notes`.
5. Add the app to `INSTALLED_APPS` using the dotted `AppConfig` path from the spec (`notes.apps.NotesConfig`).
6. Run `python manage.py runserver` and confirm the default page loads.
7. Commit: `chore(001): scaffold project`.

Don't touch env config yet. That's Step 2.

## Paste me

- Your `INSTALLED_APPS` block.
- The output of `ls` at the project root.

## Check question

`startapp` creates `notes/`, but Django ignores it until you edit one file. Which file, and what would break if you skipped that? (Think: what does `makemigrations` look at?)
