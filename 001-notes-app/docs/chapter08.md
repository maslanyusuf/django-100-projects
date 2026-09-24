# 001 Notes App — Step 8: Templates + messages

## Step 7 check, answered

PRG avoids duplicate submission. If the POST response is a rendered page, the browser's current entry in history _is_ that POST. Refresh (or Back/Forward) re-sends it, and you get the "confirm form resubmission" prompt and a duplicate note. After a redirect, the browser's current entry is a plain GET of the detail page, so refresh is harmless.

## Concept

**Template inheritance.** `base.html` defines the page shell and `{% block content %}{% endblock %}`. Child templates start with `{% extends "base.html" %}` and fill blocks. Nothing outside a block in a child template renders.

**`{% include %}`** pulls a fragment into another template. The fragment shares the includer's context (including loop variables). You'll split partials now because HTMX (Steps 9–11) returns them on their own.

**Autoescaping.** Every `{{ variable }}` is HTML-escaped by default. That's your XSS defense. `|safe` and `mark_safe` disable it, so never use them on user content. Filters like `|linebreaks` respect autoescape: they escape the input first, then add `<p>`/`<br>`.

**`{% for %}...{% empty %}...{% endfor %}`** gives you the empty state without an `{% if %}`.

**CSRF.** Django rejects POSTs without a valid token (403). Every `<form method="post">` needs `{% csrf_token %}`.

**Context processors.** A context processor is a function that adds variables to the context of every template rendered with a request (`render()` does this). The `messages` variable in templates comes from `django.contrib.messages.context_processors.messages`, already enabled in `TEMPLATES`.

**Messages framework.** `messages.success(request, "...")` in a view queues a flash message in per-request storage (cookie, falling back to session). It's read on a later request and removed once iterated, which is what makes "show once after redirect" work.

## Build this

Replace the Step 7 placeholders. File layout comes from the spec:

| File                                        | Contents                                                                                                                                                                                                      |
| ------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `templates/base.html`                       | Doctype, `<title>` block, `{% load static %}` + `static/css/app.css`, include of `_messages.html`, `{% block content %}`.                                                                                     |
| `templates/_messages.html`                  | Loop over `messages`; render each with its `tags` as a class.                                                                                                                                                 |
| `templates/notes/note_list.html`            | Extends base. Page heading, "New note" link, an Active/Archived toggle, a GET search form (`name="q"`, keeps the current `archived` value), and a container `id="note-list"` that includes `_note_list.html`. |
| `templates/notes/partials/_note_list.html`  | Loops `page_obj`, includes `_note_card.html` per note, `{% empty %}` state, then includes `_pagination.html`.                                                                                                 |
| `templates/notes/partials/_note_card.html`  | An `<article>` element: title linking to detail, pinned/archived state, and Pin / Archive / Delete as small `<form method="post">` buttons with CSRF tokens.                                                  |
| `templates/notes/partials/_pagination.html` | Prev/next and "page X of Y" from `page_obj`; links must **preserve `q` and `archived`**.                                                                                                                      |
| `templates/notes/note_detail.html`          | Title, body via `\|linebreaks`, timestamps, Edit link, Back link, Pin/Archive/Delete forms.                                                                                                                   |
| `templates/notes/note_form.html`            | Shared create/edit form: `{% csrf_token %}`, field errors visible, submit + cancel.                                                                                                                           |

Constraints that matter later, so get them right now:

- The card root must be an `<article>`, and the list container must be `#note-list`. Steps 10–11 target both.
- The search form is a plain GET form that works without JS. HTMX only enhances it in Step 10.
- Use `{% url "notes:..." %}` for every link and form action. No hardcoded paths.
- Add a few lines of CSS in `static/css/app.css` so it's usable. Styling is not graded.

Verify by hand:

1. Create a note: the success message appears once; refresh and it's gone.
2. Create a note titled `<script>alert(1)</script>`. It renders as literal text, and View Source shows `&lt;script&gt;`.
3. Empty database (or empty search): the `{% empty %}` state appears.
4. Seed 25+ notes (`bulk_create` in the shell). Page 2 works, and `q` and `archived` survive the pagination links.
5. Temporarily delete `{% csrf_token %}` from the create form: submitting returns 403. Restore it.
6. Commit: `feat(001): templates, partials and flash messages`.

## Paste me

- `base.html`, `_messages.html`, `_note_card.html` and `_note_list.html`.
- Anything from the manual checks that surprised you.

The views phase ends with this step. The **views-phase checkpoint** against the Testing and Security checklists opens the Step 9 file.

## Check question

`messages.success(...)` runs in a view that then redirects. Why can't you just pass the text through a template context variable, and what makes the message disappear after one display?
