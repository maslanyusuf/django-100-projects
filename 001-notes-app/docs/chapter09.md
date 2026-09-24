# 001 Notes App — Step 9: Add HTMX

## Step 8 check, answered

A redirect is a **new request**. Template context lives only for the single response it renders, so a context variable set in the POST request is gone by the time the browser GETs the redirect target. `messages` persists the text in storage (cookie, falling back to session) across the two requests. When a template iterates `messages`, each message is marked used, and `MessageMiddleware` drops used messages when it processes the response. That's why it shows exactly once.

## Views-phase checkpoint (self-check)

You haven't pasted Steps 7–8 code, so nothing here is reviewed. Tick each item by hand. Anything you can't tick is a bug to fix before HTMX layers on top of it.

**Testing Checklist items now verifiable in the browser:**

- [ ] List: 200; `?page=999` and `?page=abc` don't crash; `?archived=1` and `?q=` filter correctly.
- [ ] Create/edit: valid gives 302 plus a flash message; invalid gives 200 with visible errors.
- [ ] Pin toggles and bumps `updated_at` (the list reorders); archive and delete work through the plain-form fallback.
- [ ] GET on `/notes/<pk>/delete/`, `/pin/` and `/archive/` returns 405.
- [ ] POST without a CSRF token returns 403.
- [ ] Unknown pk returns 404.

**Security Checklist items that apply now:**

- [ ] `{% csrf_token %}` on every POST form (create, edit, pin, archive, delete).
- [ ] No `|safe` or `mark_safe` anywhere; the `<script>` title test renders escaped.
- [ ] Every mutation is `@require_POST`.
- [ ] `NoteForm.Meta.fields` is an explicit list.

Automated versions become tests in Step 13.

## Concept

**HTMX** lets HTML elements issue HTTP requests via `hx-*` attributes and swap the returned HTML fragment into the page. The server owns all state and returns HTML, not JSON. Your job in Django is to return the right fragment.

**Middleware** is an ordered list of hooks (`MIDDLEWARE` in settings) wrapping every request and response. Each runs top-down on the request and bottom-up on the response, so order matters when one depends on another's work.

`django-htmx` ships `HtmxMiddleware`, which reads the `HX-Request` header htmx sends on every request and sets `request.htmx`. It's truthy for htmx requests and falsy otherwise, and it exposes helpers such as `request.htmx.target` and `request.htmx.trigger`. That gives you the branching used from Step 10 on: same URL, partial for htmx, full page otherwise.

**CSRF and htmx.** A `<form>` submission includes the hidden `csrfmiddlewaretoken` field. An `hx-post` on a standalone element (a bare button) sends no form fields, so Django's CSRF middleware would reject it with 403. The fix is a global request header: htmx's `hx-headers` on `<body>` adds `X-CSRFToken` to every htmx request, and Django accepts that header by default.

## Build this

1. Install `django-htmx` if you haven't; confirm it's in `requirements.txt`.
2. Add `"django_htmx"` to `INSTALLED_APPS` and `"django_htmx.middleware.HtmxMiddleware"` to `MIDDLEWARE`. It only reads headers and must run before your views, so the end of the list is fine.
3. Load htmx 2.x in `base.html`. Choose one, and pin an **exact** version (no `latest`, no floating major):
   - CDN `<script>` tag with the exact version, ideally with an `integrity` (SRI) hash from the htmx docs, or
   - vendor the file into `static/js/` and reference it with `{% static %}`.
4. On `<body>` in `base.html`, add `hx-headers` carrying `{"X-CSRFToken": "{{ csrf_token }}"}` as JSON. Mind the quoting: JSON needs double quotes inside the attribute, so wrap the attribute value in single quotes.
5. Do not add any other `hx-*` attributes yet. Interactions start in Step 10.
6. Verify:
   - Browser console: `htmx.version` returns your pinned version, and the Network tab shows the script loading.
   - Temporarily add `print(bool(request.htmx))` to `note_list`, run the server, and compare `curl localhost:8000/` with `curl -H "HX-Request: true" localhost:8000/`. The first prints `False`, the second `True`. Remove the print.
   - View Source shows the real token in the `hx-headers` value.
7. Commit: `feat(001): add htmx and django-htmx`.

## Paste me

- Your `INSTALLED_APPS` and `MIDDLEWARE` lists.
- The `<head>` and `<body>` tags from `base.html`.
- The two curl outputs.

## Check question

Why does an `hx-post` on a standalone button need the `X-CSRFToken` header when a normal `<form method="post">` with `{% csrf_token %}` doesn't, and what would you see in the browser without it?
