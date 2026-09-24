# 001 Notes App — Step 12: Dev tooling

## Step 11 check, answered

htmx treats a **204 No Content** as "nothing to do": the request succeeds but no swap happens, so the card stays. An empty **200** is a normal response with an empty body. htmx swaps the target with that empty string, and with `hx-swap="outerHTML"` the target element is replaced by nothing, so it leaves the DOM.

## HTMX-phase checkpoint (self-check)

You haven't pasted Steps 9–11 code, so nothing here is reviewed. Tick each item by hand.

**Testing Checklist items now verifiable:**

- [ ] `curl -H "HX-Request: true" localhost:8000/` returns a fragment (no `<html>`); without the header, the full page; `Vary` includes `HX-Request`.
- [ ] Pin toggles and bumps `updated_at` (confirm in admin).
- [ ] Archive and delete return 200 with an empty body on htmx and remove the card.
- [ ] Non-htmx fallbacks redirect correctly (JS disabled).
- [ ] Mutating endpoints still return 405 on GET.
- [ ] Search, refresh, Back and Forward all show the right results.

**Security Checklist items that apply now:**

- [ ] `X-CSRFToken` header present on htmx requests; htmx POST works, and a `curl -X POST` without a token gets 403.
- [ ] htmx is pinned to an exact version (SRI hash or vendored).
- [ ] No `|safe` in any partial; the `<script>` title still renders escaped in cards, list and detail.

## Concept

**Django Debug Toolbar** is a dev-only app plus middleware that injects a panel into HTML responses. The panels you care about here are SQL (every query, duplicates, timings, stack traces), Headers, Templates and Settings.

- It exposes internals (settings, SQL, code paths), so it must exist **only** when `DEBUG` is on.
- It also only renders for requests whose client IP is in `INTERNAL_IPS`. So even a `DEBUG=True` mistake in production doesn't show it to remote users.
- It has three touch points: `INSTALLED_APPS`, `MIDDLEWARE` (as early as possible, but after anything that encodes the response body such as `GZipMiddleware`) and a URL include for its own endpoints.
- It injects into full HTML documents containing `</body>`. htmx fragments won't show it, which is expected.

**Health endpoint.** `/healthz/` is a cheap, unauthenticated route that answers "is the process up and routing?" Load balancers and container orchestrators poll it. It should touch nothing slow, so no DB query for now.

**Logging.** The `LOGGING` setting takes the standard library's `dictConfig` schema: **formatters** shape a line, **handlers** send records somewhere (console, file), **loggers** route named records to handlers at a level. Django merges your dict with its defaults; set `disable_existing_loggers` to `False` so Django's own loggers keep working. In code you'd use `logging.getLogger(__name__)`, so records carry the module path and inherit config from parent loggers such as `notes`.

## Build this

1. Create `requirements-dev.txt` starting with `-r requirements.txt`, then add `django-debug-toolbar`. Install it.
2. **Toolbar, gated on `DEBUG`:**
   - In `settings.py`, when `DEBUG` is true, append the app and the middleware, and set `INTERNAL_IPS = ["127.0.0.1"]`.
   - In `config/urls.py`, include its URLs (`__debug__/`) only when `settings.DEBUG` is true. Check the docs for the exact include; don't guess the module path.
3. **Health view.** Return `JsonResponse({"status": "ok"})`. Define it in `notes/views.py` (the spec tree has no other home for it) and route it in `config/urls.py` as `/healthz/`, name `healthz`, un-namespaced.
4. **Logging.** Add a `LOGGING` dict: version 1, `disable_existing_loggers` false, one console handler, and a root logger whose level comes from a `LOG_LEVEL` env var (default `INFO`) read through `django-environ`. Add `LOG_LEVEL` to `.env.example`.
5. **Verify:**
   - With `DEBUG=True`, the toolbar appears on `/` at `127.0.0.1:8000`. Open the SQL panel: with rows present the list view runs **exactly 2 queries** (`COUNT` + `SELECT`). If it's more, use the panel's stack traces to find the extra one and fix it.
   - With `DEBUG=False` (set `ALLOWED_HOSTS=localhost,127.0.0.1`; note `runserver` won't serve static files without `--insecure`), the toolbar is gone and `/__debug__/` is a 404.
   - `curl localhost:8000/healthz/` returns `{"status": "ok"}` with a JSON content type.
   - In `manage.py shell`, `logging.getLogger("notes").info("hello")` prints at `LOG_LEVEL=INFO` and is silent at `WARNING`.
6. Commit: `chore(001): debug toolbar, healthz and logging`.

Heads-up for Step 13: pytest-django forces `DEBUG=False` after settings load. If the toolbar is wired in with `DEBUG=True` in your `.env`, tests can fail with a `'djdt' is not a registered namespace` error. You'll fix that when it appears.

## Paste me

- The toolbar/`INTERNAL_IPS` parts of `settings.py`, plus your `LOGGING` dict.
- `config/urls.py` and the healthz view.
- The query count the SQL panel showed (and the culprit if it wasn't 2).

## Check question

Why is the toolbar gated on `DEBUG`, and what does `INTERNAL_IPS` add on top of that?
